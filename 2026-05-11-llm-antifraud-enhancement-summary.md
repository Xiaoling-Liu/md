# LLM 反欺诈能力接入 — 概要设计

## 1. 功能概述

### 1.1 背景

利用大模型（LLM）对短信进行反欺诈分类识别，作为现有 IPM 名单鉴权和 BSM（行为分析/内容分析）检测的补充。LLM 可识别涉诈、涉黄、涉赌、广告推销、催收、传销等多种违规类型，解决传统规则库无法覆盖新型诈骗模式的盲区。

### 1.2 整体架构

```
┌───────────┐   IPC 协议 (0x88/0x89/0x8A)     ┌───────────┐
│    IPM    │◄─────────────────────────────►  │    APM    │
│  (业务层)  │                                 │  (中转层)  │
│           │  Relay 模式: IPM→APM→LLM API    │           │
│           │  Direct 模式: IPM 直连 LLM API   │           │
└─────┬─────┘                                 └─────┬─────┘
      │ dlopen/libllmplugin.so                      │ dlopen/libllmplugin.so
      ▼                                              ▼
┌──────────────┐                            ┌──────────────┐
│  libllmplugin │                            │  libllmplugin │
│  线程池×4     │                            │  同步调用     │
│  HTTP + 解析  │                            │  HTTP + 解析  │
└──────┬───────┘                            └──────┬───────┘
       │                                            │
       └──────────────── LLM API ──────────────────┘
```

### 1.3 核心能力

| 能力           | 说明                                         |
| ------------ | ------------------------------------------ |
| **短信分类识别**   | 将短信内容发送至 LLM API，返回分类结果（7 种类型）             |
| **策略可配置**    | 每种分类结果可按业务需要配置不同动作（放行/监控/拦截/加黑）            |
| **并行检测**     | LLM 检测与 IPM 名单鉴权、BSM（行为分析/内容分析）并行执行，互不阻塞   |
| **异步非阻塞**    | 直连模式下 HTTP 请求在 libllmplugin 线程池执行，不阻塞业务线程  |
| **本地 IP 绑定** | 支持按网卡接口名绑定出访 IP，满足安全策略                     |
| **全量监控**     | LLM 检测结果独立入表（DATA\_LLM\_MONITOR），支持 WEB 查询 |

### 1.4 架构决策

1. **置信度不在应用层使用：** LLM API 返回的置信度在不同模型/版本间不可比，应用层不缓存、不写入监控表、不参与策略判断。`DATA_SM_STOP.llm_confidence` 字段保留（兼容），固定写入 `0.0`。
2. **未识别 label 按正常处理：** LLM API 返回的分类 label 若不在已知列表中，打印 WARNING 日志，按正常类型（type=0）处理，不拦截。
3. **异步多线程封装在动态库内部：** libllmplugin.so 内部维护线程池，对外提供同步/异步两套 C 接口，上层模块不感知多线程细节。

***

## 2. 系统架构

### 2.1 两种运行模式

| 模式             | 配置值          | HTTP 发起方                         | 网络拓扑       | 适用场景                  |
| -------------- | ------------ | -------------------------------- | ---------- | --------------------- |
| **Direct（直连）** | call\_mode=0 | IPM → libllmplugin 线程池 → LLM API | IPM 直连外网   | IPM 节点可访问 LLM API     |
| **Relay（中转）**  | call\_mode=1 | APM → libllmplugin → LLM API     | IPM→APM→外网 | IPM 不可直接外访，由 APM 统一转发 |

### 2.2 数据流

**Direct 模式：**

```
短信到达 → NeDataProcess::process_thread()
    │
    ├─ 消息类型检查 → 不在 allowed_msg_types 中则跳过 LLM
    │
    ├─ 1. detect_async(sms_content, sms_id, token, &req_id)
    │      └─ libllmplugin: 限流检查 → 入工作队列 → worker 线程 HTTP + 解析
    │
    ├─ 2. list_authentication_result()    ← IPM 名单鉴权（黑白名单）+ BSM 行为分析/内容分析 与 LLM HTTP 并行
    │
    ├─ 3. wait_result(req_id, timeout)    ← 轮询 s_results 表
    │      ├─ 就绪 → 读取 result_type
    │      └─ 超时 → 透传（不拦截）
    │
    └─ 4. get_action(result_type) → 策略处理
           ├─ PASS      → 无操作
           ├─ MONITOR   → 写入 DATA_LLM_MONITOR
           ├─ BLOCK     → 覆盖鉴权结果为阻止，DATA_SM_STOP 写入含 LLM 字段
           └─ BLACKLIST → BLOCK + 发送 IPC 0x8C 通知 APM 添加个人黑名单
```

**Relay 模式：**

```
短信到达 → NeDataProcess::process_thread()
    │
    ├─ 1. send_detect_request() → IPC 0x88 → APM
    │      └─ APM CltSocketSpm 接收 → libllmplugin detect_async → wait_result → 回复 IPC 0x89
    │
    ├─ 2. list_authentication_result()    ← IPM 名单鉴权 + BSM 行为分析/内容分析与 APM LLM 处理并行
    │
    ├─ 3. on_response(msg_id)            ← 检查 APM 是否已返回结果
    │      ├─ 有结果 → 读取 result_type
    │      └─ 超时 → 透传
    │
    └─ 4. get_action(result_type) → 策略处理（同 Direct）
```

### 2.3 消息类型过滤

LLM 检测仅针对配置中 `allowed_msg_types` 列表内的消息类型执行。不在列表中的消息类型直接跳过 LLM 检测，仅走 BSM 流程。

### 2.4 名单鉴权、BSM 与 LLM 联合决策

IPM 内部 `list_authentication_result()` 按优先级依次检查：

| 优先级   | 检查项           | 所在模块         | 说明             |
| ----- | ------------- | ------------ | -------------- |
| 1（最高） | 黑白名单          | IPM 内部       | 本地名单匹配，直接放行或阻止 |
| 2     | BSM 行为分析/内容分析 | BSM 模块       | 异步检测，返回是否阻止    |
| 3     | LLM 大模型分类     | libllmplugin | 与上述并行执行        |

**联合决策矩阵：**

| 黑白名单   | BSM 行为/内容分析 | LLM Action      | 最终结果                               |
| ------ | ----------- | --------------- | ---------------------------------- |
| 阻止     | —           | —               | 拦截，stop\_reason=黑白名单原因             |
| 放行/无匹配 | 阻止          | —               | 拦截，stop\_reason=BSM 原因             |
| 放行/无匹配 | 放行          | PASS/MONITOR    | 放行（MONITOR 时记录监控表）                 |
| 放行/无匹配 | 放行          | BLOCK/BLACKLIST | 拦截，stop\_reason=CAUSE\_LLM\_DETECT |

**关键规则：** 黑白名单 > BSM > LLM。低优先级检测可在高优先级放行后补充拦截。

***

## 3. 配置与策略

### 3.1 SYS\_LLM\_CONFIG 配置表

SYS\_LLM\_CONFIG 为本次新增配置表，单行约束（id=1），集中管理 LLM 反欺诈全部配置：

```sql
CREATE TABLE SYS_LLM_CONFIG (
    id                  NUMBER(1)     PRIMARY KEY CHECK (id=1),
    llm_enable          NUMBER(1)     DEFAULT 0,
    llm_call_mode       NUMBER(1)     DEFAULT 1,
    llm_api_url         VARCHAR2(256) NOT NULL,
    llm_auth_url        VARCHAR2(256) NOT NULL,
    llm_app_id          VARCHAR2(64)  NOT NULL,
    llm_client_id       VARCHAR2(64)  NOT NULL,
    llm_client_secret   VARCHAR2(256) NOT NULL,
    llm_timeout_ms      NUMBER(6)     DEFAULT 30000,
    llm_retry_count     NUMBER(2)     DEFAULT 2,
    llm_max_qps         NUMBER(6)     DEFAULT 500,
    llm_wait_timeout_ms NUMBER(6)     DEFAULT 5000,
    llm_allowed_msg_types VARCHAR2(128),
    llm_result_actions  VARCHAR2(128) DEFAULT '0,3,2,2,1,1,0',
    llm_local_bind_iface VARCHAR2(64),
    create_time         DATE          DEFAULT SYSDATE,
    update_time         DATE          DEFAULT SYSDATE,
    update_user         VARCHAR2(32)
);
```

| 字段                     | 类型            | 约束                        | 说明                    | 默认值             |
| ---------------------- | ------------- | ------------------------- | --------------------- | --------------- |
| id                     | NUMBER(1)     | PRIMARY KEY CHECK (id=1)  | 锁单行（仅允许 1 条记录）        | —               |
| llm\_enable            | NUMBER(1)     | CHECK (0/1)               | 总开关                   | 0               |
| llm\_call\_mode        | NUMBER(1)     | CHECK (0/1)               | 0=直连, 1=中转            | 1               |
| llm\_api\_url          | VARCHAR2(256) | NOT NULL                  | LLM API 地址            | —               |
| llm\_auth\_url         | VARCHAR2(256) | NOT NULL                  | OAuth2 认证服务地址         | —               |
| llm\_app\_id           | VARCHAR2(64)  | NOT NULL                  | 应用 ID                 | —               |
| llm\_client\_id        | VARCHAR2(64)  | NOT NULL                  | OAuth2 客户端 ID         | —               |
| llm\_client\_secret    | VARCHAR2(256) | NOT NULL                  | 客户端密钥（cryptAES 加密存储） | —               |
| llm\_timeout\_ms       | NUMBER(6)     | CHECK (1000~120000)       | HTTP 请求超时毫秒           | 30000           |
| llm\_retry\_count      | NUMBER(2)     | CHECK (0~10)              | 失败重试次数               | 2               |
| llm\_max\_qps          | NUMBER(6)     | CHECK (>=0)               | 每秒最大请求数              | 500             |
| llm\_wait\_timeout\_ms | NUMBER(6)     | CHECK (1000~60000)        | IPM 等待响应超时            | 5000            |
| llm\_allowed\_msg\_types | VARCHAR2(128) | —                         | 允许检测的消息类型列表（逗号分隔）    | —               |
| llm\_result\_actions   | VARCHAR2(128) | —                         | 7 种分类的策略编码           | 0,3,2,2,1,1,0 |
| llm\_local\_bind\_iface | VARCHAR2(64)  | —                         | 绑定网卡接口名（如 eth0）      | —               |
| create\_time           | DATE          | —                         | 创建时间                  | SYSDATE         |
| update\_time           | DATE          | —                         | 更新时间                  | SYSDATE         |
| update\_user           | VARCHAR2(32)  | —                         | 更新人                   | —               |

> Access Token 统一由 libllmplugin 通过 `auth_url` 自动获取和刷新，不存储于配置表。

### 3.2 分类结果与策略映射

LLM API 返回 7 种分类结果，每种结果可通过 `llm_result_actions` 配置对应动作：

| result\_type | 分类 | 默认策略          | 说明   |
| ------------ | -- | ------------- | ---- |
| 0            | 正常 | PASS（放行）      | 正常短信 |
| 1            | 涉诈 | BLACKLIST（加黑） | 欺诈类  |
| 2            | 涉黄 | BLOCK（拦截）     | 色情类  |
| 3            | 涉赌 | BLOCK（拦截）     | 赌博类  |
| 4            | 广告 | MONITOR（监控）   | 广告推销 |
| 5            | 催收 | MONITOR（监控）   | 催收类  |
| 6            | 传销 | PASS（放行）      | 传销类  |

**配置格式：** `llm_result_actions` 为逗号分隔的 7 个整数，分别对应 type 0-6 的动作编码。

| 动作编码 | 常量                     | 说明                       |
| ---- | ---------------------- | ------------------------ |
| 0    | LLM\_ACTION\_PASS      | 放行，无操作                   |
| 1    | LLM\_ACTION\_MONITOR   | 监控，写入 DATA\_LLM\_MONITOR |
| 2    | LLM\_ACTION\_BLOCK     | 拦截，写入 DATA\_SM\_STOP     |
| 3    | LLM\_ACTION\_BLACKLIST | 加黑 + 拦截                  |

### 3.3 配置加载与广播

```
APM 启动 → DB 加载 SYS_LLM_CONFIG → 初始化 libllmplugin
                                      → 广播 0x8A 至所有 IPM 节点

IPM 接收广播 → 更新 SYS_CONFIG → CLLMPluginLoader::update_plugin_config()

配置变更 → APM 重新加载 DB → update_plugin_config() + 广播 0x8A
```

***

## 4. 接口定义

### 4.1 libllmplugin API（C 接口）

提供在动态库 `libllmplugin.so` 中，通过 dlopen/dlsym 加载。

```c
// ── 初始化（9 参数） ──
int llm_plugin_init(
    const char* api_url,          // LLM API 地址
    const char* auth_url,         // OAuth2 认证地址
    const char* app_id,           // 应用 ID
    const char* client_id,        // OAuth2 客户端 ID
    const char* client_secret,    // 客户端密钥
    int timeout_ms,               // HTTP 超时（毫秒）
    int retry_count,              // 失败重试次数
    int max_qps,                  // 每秒最大请求数
    const char* local_bind_iface  // 绑定网卡接口名（可为空）
);

int llm_plugin_update_config(/* 同 init 9 参数 */);

// ── 同步检测（Relay 模式使用） ──
int llm_plugin_detect(
    const char* sms_content,
    const char* sms_id,
    int* result_type,      // 输出分类: 0=正常..6=传销
    float* confidence      // 输出: 固定 0.0
);

// ── 异步检测（Direct 模式使用） ──
#define LLM_OK      0
#define LLM_PENDING 1
#define LLM_ERR     -1

int llm_plugin_detect_async(const char* sms_content, const char* sms_id,
                            uint64_t* out_request_id);
int llm_plugin_poll_result(uint64_t request_id, int* result_type, float* confidence);
int llm_plugin_wait_result(uint64_t request_id, int timeout_ms,
                           int* result_type, float* confidence);

// ── 日志与清理 ──
void llm_plugin_set_log_fn(void (*log_fn)(int level, const char* msg));
void llm_plugin_cleanup();
```

**返回值：**

| 常量                 | 值  | 说明                     |
| ------------------ | -- | ---------------------- |
| LLM\_OK            | 0  | 成功                     |
| LLM\_PENDING       | 1  | poll 时结果未就绪 / wait 时超时 |
| LLM\_RATE\_LIMITED | -2 | 被限流                    |
| LLM\_ERR           | -1 | 通用错误                   |

### 4.2 IPM ↔ APM IPC 协议

消息结构定义在 `01_IPM/src/llm/LLMProtocol.h`（IPM 侧）和 `03_APM/src/llm/LLMProtocol.h`（APM 侧）。

**LLM 检测请求（function\_id=0x88）：**

```
┌──────────┬────────────┬──────────────────────────────┐
│ 字段     │ 类型       │ 说明                         │
├──────────┼────────────┼──────────────────────────────┤
│ msg_id   │ ACE_INT64  │ 消息唯一标识，用于匹配响应   │
│ sender   │ char[21]   │ 主叫号码                     │
│ receiptor│ char[21]   │ 被叫号码                     │
│ send_time│ ACE_INT64  │ 发送时间戳                   │
│ sms_content_len│Int16│ 短信内容长度                  │
│ sms_content│char[]   │ 短信内容（变长，紧接包头）    │
└──────────┴────────────┴──────────────────────────────┘
```

**LLM 检测响应（function\_id=0x89）：**

```
┌──────────┬────────────┬──────────────────────────────┐
│ 字段     │ 类型       │ 说明                         │
├──────────┼────────────┼──────────────────────────────┤
│ msg_id   │ ACE_INT64  │ 匹配请求的 msg_id            │
│ is_success│ACE_INT8   │ 0=成功, 1=失败               │
│ result_type│ACE_INT8  │ 0=正常..6=传销               │
│ confidence│float      │ 固定 0.0                     │
│ error_msg_len│Int16  │ 错误消息长度                  │
│ error_msg│char[]     │ 错误消息（变长）              │
└──────────┴────────────┴──────────────────────────────┘
```

**配置更新广播（function\_id=0x8A）：** 由 APM 主动下发至所有 IPM 节点，包含 SYS\_LLM\_CONFIG 的全部 LLM 相关字段。Token 统一由 libllmplugin 通过 `auth_url` 自动获取，不包含在广播报文中。

**个人黑名单添加（function\_id=0x8C）：** IPM 在 LLM action=BLACKLIST 时发送至 APM，APM 持久化后广播 `PERSONAL_BLKLIST_OP_REQ` 至所有 IPM。

### 4.3 各模块职责

| 模块               | 文件                             | 职责                                              |
| ---------------- | ------------------------------ | ----------------------------------------------- |
| **IPM**          | `NeDataProcess.cpp`            | 短信处理入口，LLM 检测调用与 action 处理                      |
| <br />           | `CLLMClient`                   | 单例，封装 `get_action()` 策略判断、`detect_spam()` 同步检测  |
| <br />           | `CLLMPluginLoader`             | 包装 libllmplugin.so 的 dlopen/dlsym + API 转发      |
| <br />           | `CLLMAPMCallProxy`             | Relay 模式：管理待响应的 msg\_id 表，发送 IPC 请求/处理响应        |
| <br />           | `LLMProtocol.h`                | 协议结构体定义                                         |
| <br />           | `InformationLLMMonitorStorage` | DATA\_LLM\_MONITOR 表 OCCI 写入                    |
| **APM**          | `CltSocketSpm.cpp`             | 接收 IPM 的 LLM 检测请求（0x88），调用 libllmplugin，回复 0x89 |
| <br />           | `BusinessControl.cpp`          | LLM 配置加载、插件初始化、配置变更广播                           |
| <br />           | `LLMPluginLoader`              | APM 侧 libllmplugin.so 包装层                       |
| **libllmplugin** | `LLMPlugin.cpp`                | curl HTTP、JSON 解析、access\_token 管理、线程池、限流       |
| <br />           | `LLMPlugin.h`                  | C 接口声明                                          |

***

## 5. libllmplugin 内部设计

### 5.1 功能范围

| 功能              | 说明                                        |
| --------------- | ----------------------------------------- |
| HTTP 通信         | 基于 libcurl，构造请求/解析响应                      |
| Access Token 管理 | 自动获取、缓存、刷新                                |
| 速率控制            | 基于 token bucket，按 max\_qps 限流             |
| 本地 IP 绑定        | CURLOPT\_INTERFACE 按接口名绑定                 |
| 异步线程池           | 4 个工作线程，std::thread + condition\_variable |
| 超时管理            | request\_id 粒度的超时回收                       |
| 日志回调            | 通过 `set_log_fn` 注册，输出至框架 LOGGER           |

### 5.2 异步工作流

```
detect_async(content, sms_id, token, &req_id)
    │
    ├─ 限流检查（s_mutex）
    │    ├─ 通过 → 继续
    │    └─ 超过 → 返回 LLM_RATE_LIMITED
    │
    ├─ request_id = atomic<uint64_t>.fetch_add(1)
    ├─ 预写 s_results[req_id].ret_code = LLM_PENDING
    ├─ 复制字符串入 s_request_queue（s_async_mutex）
    └─ notify_one 唤醒 worker
                 │
    ┌────────────┴──────────────────┐
    │  Worker 线程 (×4)             │
    │  wait(cv, queue有新任务)       │
    │  dequeue                      │
    │  config snapshot (s_mutex)    │
    │  get_token (s_mutex)          │
    │  curl HTTP POST               │
    │  JSON 解析                    │
    │  label → result_type          │
    │  s_results[id] = {ret, type}  │
    └────────────────────────────────┘

wait_result(req_id, timeout_ms)
    │ 每 10ms poll_result
    │ ├─ PENDING → 继续
    │ ├─ OK     → erase(id), 返回 result
    │ └─ 超时   → erase(id), 返回 LLM_PENDING
```

### 5.3 API 响应解析

```json
{
    "label": "涉诈",
    "confidence": 0.95,
    "sms_id": "13800138000_12345_67890"
}
```

- `label` → 映射为 result\_type（0-6），未识别 label 按 type=0 处理 + WARNING 日志
- `confidence` → 应用层不使用，始终返回 0.0f

***

## 6. 数据库设计

### 6.1 DATA\_SM\_STOP（拦截表）加 3 列

```sql
ALTER TABLE DATA_SM_STOP ADD (
    llm_result_type NUMBER(3),      -- 0=正常..6=传销
    llm_confidence  NUMBER(8,6),    -- 固定 0.0（兼容保留）
    llm_triggered   NUMBER(3)       -- 0=BSM 触发, 1=LLM 触发
);
```

### 6.2 DATA\_LLM\_MONITOR（监控表，新建）

```sql
CREATE TABLE DATA_LLM_MONITOR (
    list_id         NUMBER(18)    NOT NULL,
    sms_id          VARCHAR2(256)  NOT NULL,   -- {sender}_{ne_list_id}_{sm_id}
    sender          VARCHAR2(22)   NOT NULL,   -- 主叫号码
    receiptor       VARCHAR2(22)   NOT NULL,   -- 被叫号码
    sm_content      VARCHAR2(4000),            -- 短信内容
    send_time       DATE,                       -- 短信发送时间
    llm_result_type NUMBER(3),                 -- LLM 分类结果
    llm_confidence  NUMBER(8,6),               -- 固定 0.0
    detect_time     DATE          NOT NULL,     -- LLM 检测时间
    sync_id         NUMBER(18)                 -- 同步 ID
)
TABLESPACE ASTBLSP_2
PARTITION BY RANGE (detect_time) INTERVAL (NUMTODSINTERVAL(1, 'DAY'))
( PARTITION P0 VALUES LESS THAN (TO_DATE('2026-01-01', 'YYYY-MM-DD')) )
NOCACHE;

CREATE INDEX IDX_LLM_MONITOR_DETECT_TIME ON DATA_LLM_MONITOR(detect_time) LOCAL;
CREATE INDEX IDX_LLM_MONITOR_SENDER      ON DATA_LLM_MONITOR(sender)      LOCAL;
CREATE INDEX IDX_LLM_MONITOR_RESULT_TYPE ON DATA_LLM_MONITOR(llm_result_type) LOCAL;
```

### 6.3 SYS\_LLM\_CONFIG

参见 §3.1 完整表结构。Token 统一通过 `llm_auth_url` 自动获取，无需预置 token 字段。

### 6.4 写入逻辑

| 场景            | 写入表                | 触发条件                                  |
| ------------- | ------------------ | ------------------------------------- |
| LLM 触发拦截      | DATA\_SM\_STOP     | action=BLOCK/BLACKLIST，且黑白名单/BSM 均未拦截 |
| 黑白名单/BSM 触发拦截 | DATA\_SM\_STOP     | 名单阻止或 BSM 返回阻止（llm\_triggered=0）      |
| LLM 监控记录      | DATA\_LLM\_MONITOR | action=MONITOR                        |
| 个人黑名单持久化      | 个人黑名单表             | action=BLACKLIST（通过 APM）              |

***

## 7. 监控查询界面

### 7.1 功能说明

基于 DATA\_LLM\_MONITOR 表提供 WEB 查询页面，用于运营人员查看 LLM 检测结果。

### 7.2 查询条件

| 条件      | 控件      | 必填 | 匹配方式            |
| ------- | ------- | -- | --------------- |
| 检测时间    | 日期范围选择器 | 是  | detect\_time 范围 |
| 主叫号码    | 文本输入框   | 否  | LIKE 模糊匹配       |
| 被叫号码    | 文本输入框   | 否  | LIKE 模糊匹配       |
| LLM 分类  | 下拉多选    | 否  | 精确匹配            |
| sms\_id | 文本输入框   | 否  | 精确匹配            |
| 短信内容    | 文本输入框   | 否  | LIKE 模糊匹配       |

### 7.3 结果列表

| 列       | 说明                | 格式/处理               |
| ------- | ----------------- | ------------------- |
| 检测时间    | detect\_time      | yyyy-MM-dd HH:mm:ss |
| sms\_id | 完整显示              | —                   |
| 主叫号码    | sender            | 脱敏                  |
| 被叫号码    | receiptor         | 脱敏                  |
| 短信内容    | sm\_content       | 超 50 字符截断加 "..."    |
| 发送时间    | send\_time        | yyyy-MM-dd HH:mm:ss |
| 分类结果    | llm\_result\_type | 中文标签                |
| 置信度     | llm\_confidence   | 固定 0%               |

**分页：** 每页 20 行，支持翻页。

### 7.4 详情弹出

点击某行弹出详情窗口，展示所有字段完整内容（sm\_content 全文）。

### 7.5 查询 SQL

```sql
SELECT * FROM DATA_LLM_MONITOR
 WHERE detect_time >= :start_time
   AND detect_time <  :end_time + 1
   AND (:sender IS NULL OR sender LIKE '%' || :sender || '%')
   AND (:receiptor IS NULL OR receiptor LIKE '%' || :receiptor || '%')
   AND (:result_type IS NULL OR llm_result_type = :result_type)
 ORDER BY detect_time DESC;
```

***

## 8. 测试要点

### 8.1 基本功能测试

| TC | 场景            | 前置                   | 验证点                    |
| -- | ------------- | -------------------- | ---------------------- |
| 01 | Direct 模式正常检测 | call\_mode=0, API 正常 | 短信处理不阻塞，BSM 与 LLM 并行   |
| 02 | Relay 模式正常检测  | call\_mode=1, APM 在线 | IPC 请求/响应正常，结果正确       |
| 03 | 未识别 label     | label 不在已知列表         | WARNING 日志，按 type=0 处理 |
| 04 | API 异常        | 500/超时               | 检测失败，短信透传不拦截           |
| 05 | 限流触发          | 超 QPS                | 返回 RATE\_LIMITED，不崩溃   |
| 06 | 配置变更即时生效      | 修改 result\_actions   | 新请求使用新策略               |
| 07 | 插件重载          | reload\_plugin       | 旧线程池退出，新线程池正常处理        |

### 8.2 联合决策测试

| TC | 场景       | 黑白名单 | BSM | LLM Action | 预期结果                     |
| -- | -------- | ---- | --- | ---------- | ------------------------ |
| 08 | LLM 拦截   | 无匹配  | 放行  | BLOCK      | 拦截，stop\_reason=LLM      |
| 09 | 名单阻止优先   | 阻止   | —   | —          | 拦截，stop\_reason=名单原因     |
| 10 | BSM 阻止优先 | 无匹配  | 阻止  | —          | 拦截，stop\_reason=BSM 原因   |
| 11 | 都放行      | 无匹配  | 放行  | PASS       | 放行                       |
| 12 | LLM 监控   | 无匹配  | 放行  | MONITOR    | 放行，写入 DATA\_LLM\_MONITOR |
| 13 | LLM 加黑   | 无匹配  | 放行  | BLACKLIST  | 拦截 + 持久化黑名单              |

### 8.3 DATA\_SM\_STOP 写入验证

| TC | 场景        | llm\_triggered  | llm\_result\_type |
| -- | --------- | --------------- | ----------------- |
| 13 | LLM 触发    | 1               | 对应分类值             |
| 14 | 名单/BSM 触发 | 0               | 0                 |
| 15 | 升级前历史数据   | NULL（无 DEFAULT） | NULL              |

### 8.4 DATA\_LLM\_MONITOR 验证

| TC | 场景                | 验证点             |
| -- | ----------------- | --------------- |
| 16 | action=MONITOR 写入 | 表中存在记录，字段正确     |
| 17 | action=BLOCK 时不写入 | 表中无记录           |
| 18 | INSERT 失败不阻塞      | WARNING 日志，管道继续 |
| 19 | 分区间隔按天            | 跨天数据分布在正确分区     |

### 8.5 并发与压力

| TC | 场景                 | 验证点                     |
| -- | ------------------ | ----------------------- |
| 20 | 10 线程并发提交          | 线程池排队，无死锁/数据竞争          |
| 21 | 长时间高负载             | 无内存泄漏，s\_results 不过期积累  |
| 22 | libllmplugin 动态库兼容 | 旧版 .so 无 async 符号时降级不崩溃 |

### 8.6 监控页面验证

| TC | 场景      | 验证点          |
| -- | ------- | ------------ |
| 23 | 按时间范围查询 | 返回正确时间段数据    |
| 24 | 按主叫模糊查询 | LIKE 匹配正确    |
| 25 | 分类下拉筛选  | 过滤正确         |
| 26 | 分页      | 每页 20 条，翻页正常 |
| 27 | 详情弹出    | 完整字段显示       |

***

## 9. 升级与回滚

### 9.1 数据库升级


变更内容：

- DATA_SM_STOP 加 3 列（NULLABLE，无 DEFAULT，秒级完成）
- 新建 DATA_LLM_MONITOR 表（含索引、分区）
- SYS_LLM_CONFIG 加 `llm_local_bind_iface` 列

### 9.2 程序升级

1. 部署新版本 `libllmplugin.so`
2. 部署新版本 IPM 模块
3. 部署新版本 APM 模块（Relay 模式时）
4. 配置 `SYS_LLM_CONFIG.llm_local_bind_iface`（需要绑定网卡时）
5. 按顺序重启：APM → IPM

### 9.3 回滚
