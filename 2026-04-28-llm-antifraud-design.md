# 概要设计文档：元景 LLM 反诈模型集成

**版本:** v1.7
**日期:** 2026-05-07
**所属项目:** antispaming 2850

***

## 1. 概述

本文档基于《元景 LLM 反诈模型集成 PRD v1.5》进行概要设计，覆盖 LLM 检测功能的方案说明、配置组合、判定规则、模块划分、内部通信协议、调用流程、数据库设计。

***

## 2. 方案说明

### 2.1 方案一句话描述

在现有的 BSM 黑白名单检测流程中，**并行插入一路 LLM 大模型检测**——支持两种部署模式：中转模式时 IPM 通过 IPC 发给 APM，APM 调用外部大模型 API；直连模式时 IPM 直接加载 libllmplugin 调用大模型 API。大模型返回内容分类（涉诈/涉黄/涉赌/广告/催收/正常/传销）后，按可配置策略执行放行、监控、拦截或加黑处置。

### 2.2 方案范围

| 包含                                          | 不包含            |
| ------------------------------------------- | -------------- |
| IPM ↔ APM 的 IPC 协议扩展（0x88/0x89 + 0x8A/0x8B） | 修改原有的 BSM 检测逻辑 |
| libllmplugin 公共库（流控+统计+日志+HTTP调用）           | 修改原有的黑白名单判断    |
| APM 侧插件加载、大模型 API 调用                        | 修改原有的拦截入库流程    |
| IPM 侧请求发送、超时控制、策略分发（加黑/拦截/监控/放行）            | 修改原有的统计上报流程    |
| 直连模式（call\_mode=0，IPM 本地加载 libllmplugin）    | <br />         |
| 中转模式（call\_mode=1，IPM→APM→大模型 API）          | <br />         |
| 配置的数据库存储和动态加载                               | <br />         |

### 2.3 方案的各参与方

```
短信中心 → IPM（判断黑白名单 → 并行 BSM+LLM 检测 → 综合判定）→ 短信中心
                                    │
                            LLM 路径 (中转模式):
                            IPM → [IPC 0x88] → APM → [libllmplugin.so] → 大模型 API
                                  ← [IPC 0x89] ←                              ↑
                                                                       内置流控+统计+周期日志

                            LLM 路径 (直连模式):
                            IPM → [libllmplugin.so] → 大模型 API
                                       ↑
                                内置流控+统计+周期日志
```

### 2.4 方案的关键决策点

整个方案有 **3 个配置开关** 和 **3 个运行时结果** 决定最终行为：

**3 个配置开关：**

| #  | 开关                       | 可选值      | 作用                  |
| -- | ------------------------ | -------- | ------------------- |
| S1 | llm\_enable              | ON / OFF | LLM 检测功能总闸          |
| S2 | llm\_allowed\_msg\_types | 类型 ID 列表 | 哪些短信类型需要送 LLM 检测    |
| S3 | llm\_result\_actions     | 7 个策略编码  | 每个分类结果（0\~6）对应的处置策略 |

**3 个运行时结果：**

| #  | 结果                   | 可能值                  |
| -- | -------------------- | -------------------- |
| R1 | BSM 检测结果             | 拦截 / 放行              |
| R2 | LLM 检测结果（大模型 API 返回） | 正常/涉诈/涉黄/涉赌/广告/催收/传销 |
| R3 | LLM 调用状态             | 成功 / 超时 / 失败 / 限流    |

***

## 3. 配置组合与行为矩阵

### 3.1 llm\_enable 总开关

| llm\_enable | LLM 调用 | LLM 拦截 | 适用场景                  |
| :---------: | :----: | :----: | --------------------- |
|     OFF     |   不调用  |   不拦截  | LLM 服务异常时紧急关闭，完全走原有逻辑 |
|      ON     |  正常调用  |   按配置  | 正常生产运营                |

> llm\_enable=OFF 时，系统行为与未集成 LLM 前完全一致，0 性能损耗。

### 3.2 llm\_allowed\_msg\_types 检测范围

此配置控制「哪些短信类型会送 LLM 检测」。短信类型 ID 列表以逗号分隔存储，如 `1,6` 表示只对类型 1 和类型 6 的短信执行 LLM 检测。

| 短信类型是否在 allowed\_msg\_types 中 | LLM 是否检测 |
| :---------------------------: | :------: |
|               是               |    检测    |
|               否               |    不检测   |

### 3.3 llm\_result\_actions 分类处置策略

此配置控制「每个 LLM 分类结果（0\~6）对应的处置策略」。存储为逗号分隔 7 个策略编码，按分类枚举值 0\~6 顺序排列。

**策略编码：**

|  编码 |  策略 | 说明                               |
| :-: | :-: | -------------------------------- |
|  0  |  放行 | 不做处理，短信正常通过                      |
|  1  |  监控 | 仅记录监控入库，不下发拦截（复用 BSM 监控能力）       |
|  2  |  拦截 | 拦截该条短信                           |
|  3  |  加黑 | 将主叫号码加入黑名单 + 拦截该条短信（复用 IPM 加黑能力） |

**默认配置：** `"0,3,2,2,1,1,0"` 对应：

| 枚举值 | 分类标签 |  默认策略  |
| :-: | ---- | :----: |
|  0  | 正常   |   放行   |
|  1  | 涉诈   | **加黑** |
|  2  | 涉黄   |   拦截   |
|  3  | 涉赌   |   拦截   |
|  4  | 广告   |   监控   |
|  5  | 催收   |   监控   |
|  6  | 传销   |   放行   |

### 3.4 配置组合汇总表

| llm\_enable | 短信类型匹配 | LLM 调用 |           LLM 处置          | 最终行为              |
| :---------: | :----: | :----: | :-----------------------: | :---------------- |
|     OFF     |    -   |   不调用  |            无影响            | 仅根据 BSM 结果        |
|      ON     |   匹配   |   调用   | 按 llm\_result\_actions 执行 | BSM结果 + LLM策略综合判定 |
|      ON     |   不匹配  |   不调用  |            无影响            | 仅根据 BSM 结果        |

***

## 4. 判定决策表

### 4.1 最终结果判定

`最终结果 = f(BSM结果, LLM调用状态, LLM分类, llm_result_actions)`

**编号规则：**

- BSM 结果：A=放行, B=拦截
- LLM 调用状态：1=成功, 2=超时, 3=失败, 4=限流
- LLM 分类：0\~6 → 查 llm\_result\_actions 得到策略：P=放行(0), M=监控(1), I=拦截(2), B=加黑(3)

**完整判定表：**

|  #  | BSM |   LLM状态  | LLM分类 |   LLM策略   |  最终结果  |    stop\_reason    | 补充行为 |
| :-: | :-: | :------: | :---: | :-------: | :----: | :----------------: | ---- |
|  1  |  A  |   成功(1)  |   0   |   P(放行)   |   放行   |      NO\_STOP      | -    |
|  2  |  A  |   成功(1)  |   1   | B(**加黑**) | **拦截** | CAUSE\_LLM\_DETECT | 号码加黑 |
|  3  |  A  |   成功(1)  |   2   | I(**拦截**) | **拦截** | CAUSE\_LLM\_DETECT | -    |
|  4  |  A  |   成功(1)  |   3   | I(**拦截**) | **拦截** | CAUSE\_LLM\_DETECT | -    |
|  5  |  A  |   成功(1)  |   4   | M(**监控**) | **放行** |      NO\_STOP      | 监控入库 |
|  6  |  A  |   成功(1)  |   5   | M(**监控**) | **放行** |      NO\_STOP      | 监控入库 |
|  7  |  A  |   成功(1)  |   6   |   P(放行)   |   放行   |      NO\_STOP      | -    |
|  8  |  A  | 超时/失败/限流 |   -   |     -     |   放行   |      NO\_STOP      | -    |
|  9  |  B  |   成功(1)  |   0   |   P(放行)   | **拦截** |        BSM原因       | -    |
|  10 |  B  |   成功(1)  |   1   | B(**加黑**) | **拦截** |       BSM原因¹       | 号码加黑 |
|  11 |  B  |   成功(1)  |   2   | I(**拦截**) | **拦截** |       BSM原因¹       | -    |
|  12 |  B  |   成功(1)  |   4   | M(**监控**) | **拦截** |        BSM原因       | 监控入库 |
|  13 |  B  | 超时/失败/限流 |   -   |     -     | **拦截** |        BSM原因       | -    |

> ¹ 当 BSM 已判定拦截时，stop\_reason 保留 BSM 的原有原因（如 CAUSE\_BLACKLIST），即使 LLM 也是拦截/加黑，不覆盖。

### 4.2 判定规则总结

```
最终结果 = 拦截 当且仅当:
   BSM结果=拦截
   或 LLM策略 ∈ {拦截, 加黑}

号码加黑 当且仅当:
   LLM策略 = 加黑 (3)

监控入库 当且仅当:
   LLM策略 = 监控 (1)

stop_reason:
   BSM拦截 → BSM原有原因
   LLM加黑/拦截 + BSM放行 → CAUSE_LLM_DETECT
```

### 4.3 CAUSE\_LLM\_DETECT 触发条件

```mermaid
flowchart TD
    A["LLM策略 ∈ {拦截, 加黑}<br/>BSM 结果 = 放行"] --> B["stop_reason = CAUSE_LLM_DETECT"]
    C["LLM策略 ∈ {拦截, 加黑}<br/>BSM 结果 = 拦截"] --> D["stop_reason = BSM 原有原因"]
    E["LLM策略 = 加黑"] --> F["复用 IPM 加黑接口<br/>将主叫号码加入黑名单"]
    G["LLM策略 = 监控"] --> H["复用 BSM 监控接口<br/>记录监控入库"]
```

***

## 5. 整体架构

### 5.1 进程架构

#### 中转模式（call\_mode=1，默认）

```mermaid
flowchart TB
    subgraph IPM["IPM 进程"]
        NDP["NeDataProcess<br/>(BSM检测完成后调用)"]
        APCP["CLLMAPMCallProxy<br/>· send_detect_request()<br/>· 打包 0x88 协议<br/>· 注册等待队列<br/>· on_response() 匹配"]
        NDP --> APCP
    end

    subgraph IPC["复用现有 IPC 通道 (TCP)"]
        direction LR
        REQ["0x88 LLMDetectReq_package"]
        RESP["0x89 LLMDetectResp_package"]
    end

    subgraph APM["APM 进程"]
        APMH["APM Handler<br/>(识别 0x88 路由)"]
        LPL["LLMPluginLoader<br/>· dlopen 加载<br/>· llm_plugin_detect()"]
        LLMP["libllmplugin.so<br/>· 令牌桶流控<br/>· 5分钟统计日志<br/>· OAuth2 token管理<br/>· HTTP调用大模型API"]
        APMH --> LPL
        LPL --> LLMP
    end

    LLMAPI["大模型 API<br/>(外部服务)"]
    LLMP -->|"HTTP POST<br/>multipart/form-data"| LLMAPI

    APCP -.->|"send_detect_request(msg_id, sms_content)"| REQ
    RESP -.->|"on_response(result_type, confidence)"| APCP
    REQ -->|"TCP/IP"| APMH
```

#### 直连模式（call\_mode=0）

```mermaid
flowchart TB
    subgraph IPM["IPM 进程"]
        NDP["NeDataProcess<br/>(BSM检测完成后调用)"]
        LPL["CLLMPluginLoader<br/>· dlopen 加载<br/>· llm_plugin_detect()"]
        LLMP["libllmplugin.so<br/>· 令牌桶流控<br/>· 5分钟统计日志<br/>· OAuth2 token管理<br/>· HTTP调用大模型API"]
        NDP -.->|"detect(sms_content, msg_id)<br/>→ result_type, confidence"| LPL
        LPL --> LLMP
    end

    LLMAPI["大模型 API<br/>(外部服务)"]
    LLMP -->|"HTTP POST<br/>multipart/form-data"| LLMAPI
```

### 5.2 短信处理主流程（含 LLM 并行检测）

```mermaid
flowchart TD
    START(["从队列取出待处理短信"])

    START --> PARSE[协议解析<br/>国家码处理<br/>号码属性识别]
    PARSE --> STAT[流量统计<br/>SP话单记录<br/>特殊号码处理]
    STAT --> BL[IPM黑白名单判断]

    BL --> TYPE{"名单类型?"}

    TYPE -->|黑名单| BLOCK["直接拦截入库<br/>（不走 LLM 检测）"]

    TYPE -->|白名单| PASSTHRU["放行<br/>后续BSM/APM处理<br/>（不走 LLM 检测）"]

    TYPE -->|非黑白名单| CHECK_LLM{"llm_enable=ON?<br/>且 短信类型在 allowed_msg_types?"}:::new

    CHECK_LLM -->|是| BRANCH{"call_mode?"}:::new

    BRANCH -->|直连 0| SYNC_DETECT["CLLMPluginLoader::detect()<br/>→ libllmplugin 本地加载<br/>→ 同步调大模型 API<br/>→ 内置流控+统计<br/>(同步，BSM 后等待结果)"]:::new

    BRANCH -->|中转 1| SEND_LLM["CLLMAPMCallProxy::send_detect_request()<br/>→ 打包 0x88 → IPC 发给 APM<br/>→ 注册 msg_id 到等待队列<br/>(异步，不阻塞 BSM 主线程)"]:::new

    CHECK_LLM -->|否| BSM_ONLY["仅执行 BSM 名单检测<br/>（不走 LLM 检测）"]

    SYNC_DETECT -.-> BSM_MAIN["主线程继续:<br/>发送 BSM 名单检测"]:::new

    SEND_LLM -.-> BSM_MAIN

    BSM_MAIN --> CHECK_RESULT{"call_mode?<br/>直连: detect 返回结果<br/>中转: on_response() 有结果?"}:::new

    BSM_ONLY --> BSM_JUDGE{"BSM 结果?"}
    BSM_JUDGE -->|拦截| FINAL_BLOCK["最终结果: 拦截"]
    BSM_JUDGE -->|放行| FINAL_PASS["最终结果: 放行"]

CHECK_RESULT -->|是 结果已到达| GET_ACTION["CLLMClient::get_action()<br/>→ result_actions[result.result_type]<br/>→ 0=放行,1=监控,2=拦截,3=加黑"]:::new

    CHECK_RESULT -->|否 超时/失败| LLM_FAIL["LLM=放行<br/>仅依赖 BSM 结果"]:::new
    LLM_FAIL --> BSM_JUDGE2{"BSM 结果?"}:::new
    BSM_JUDGE2 -->|拦截| FINAL_BLOCK
    BSM_JUDGE2 -->|放行| FINAL_PASS

    GET_ACTION --> DISPATCH{"action?"}:::new

    DISPATCH -->|加黑 3| LLM_BLACKLIST["将主叫号码加入黑名单<br/>复用 IPM 加黑接口"]:::new
    DISPATCH -->|拦截 2| LLM_BLOCK["LLM 判定拦截"]:::new
    DISPATCH -->|监控 1| LLM_MONITOR["is_dxcz_monitor = 1<br/>(监控入库标志)"]:::new
    DISPATCH -->|放行 0| BSM_JUDGE3{"BSM 结果?"}:::new

    LLM_BLACKLIST --> LLM_BLOCK

    LLM_MONITOR --> BSM_JUDGE3

    BSM_JUDGE3 -->|拦截| FINAL_BLOCK
    BSM_JUDGE3 -->|放行| FINAL_PASS

    LLM_BLOCK --> STOP_CAUSE{"BSM 结果?"}:::new
    STOP_CAUSE -->|BSM 已拦截| S1["stop_reason = BSM 原有原因"]:::new
    STOP_CAUSE -->|BSM 放行| S2["stop_reason = CAUSE_LLM_DETECT"]:::new
    S1 --> FINAL_BLOCK
    S2 --> FINAL_BLOCK

    BLOCK --> AFTER[统计流量<br/>抽样上报]
    PASSTHRU --> AFTER
    FINAL_BLOCK --> AFTER
    FINAL_PASS --> AFTER

    AFTER --> NEED_BSM{"需要后续分析?"}
    NEED_BSM -->|否| END(["释放资源<br/>下一轮循环"])
    NEED_BSM -->|是| BUILD["构建发往BSM/APM数据包<br/>填充结果字段<br/>计算阻止原因"]
    BUILD --> RESPOND["适配拦截模式<br/>应答网元处理"]
    RESPOND --> DB_BLOCK["拦截入库处理"]
    DB_BLOCK --> END

    classDef new fill:#ffcccc,stroke:#ff0000,stroke-width:2px;
```

### 5.3 APM 内部 LLM 请求处理流程

```mermaid
flowchart TD
    RECV["APM Handler 收到 function_id=0x88"]

    RECV --> PARSE_HDR["解析包头, 验证 message_id = 0x88"]
    PARSE_HDR --> PARSE_BODY["解析包体<br/>提取 sender_num, receiptor_num<br/>提取 sms_content 变长字段"]
    PARSE_BODY --> LOAD_PLUGIN["LLMPluginLoader::load_plugin()<br/>dlopen libllmplugin.so"]

    LOAD_PLUGIN --> INIT["llm_plugin_init(api_url, auth_url, ..., max_qps)<br/>初始化 curl, 令牌桶, 注册日志回调"]
    INIT --> DETECT["llm_plugin_detect(sms_content, sms_id,<br/>access_token, result_type, confidence)<br/>→ 库内令牌桶限流 → OAuth2 token → HTTP 调用"]

    DETECT -->|成功| SUCCESS_RESP["返回成功响应<br/>is_success = 0<br/>result_type = 分类结果<br/>confidence = 置信度"]

    DETECT -->|失败| FAIL_RESP["返回失败响应<br/>is_success = 1<br/>error_msg = 错误描述"]

    DETECT -->|限流| RATE_LIMIT_RESP["返回限流降级响应<br/>is_success = 1<br/>result_type = 0"]

    SUCCESS_RESP --> SEND_BACK["组装 LLMDetectResp_package<br/>function_id = 0x89"]
    FAIL_RESP --> SEND_BACK
    RATE_LIMIT_RESP --> SEND_BACK

    SEND_BACK --> IPC_SEND["通过 IPC 发送响应给 IPM"]
```

### 5.4 设计原则

| 原则    | 说明                                                         |
| ----- | ---------------------------------------------------------- |
| 双模式部署 | 支持中转模式（IPM→APM→大模型API）和直连模式（IPM→大模型API）                    |
| 插件解耦  | 所有大模型逻辑收进 libllmplugin 动态库，IPM/APM 通过 dlopen 加载            |
| 流量控制  | libllmplugin 库内建令牌桶算法，每个进程独立限速，无需中心协调                      |
| 零侵入   | 不修改原有 BSM/APM 检测逻辑                                         |
| 线程安全  | token 缓存使用 mutex 保护，限流器线程安全                                |
| 失败容错  | LLM 任何失败默认放行，不阻塞核心短信通路                                     |
| 异步非阻塞 | send\_detect\_request() 立即返回，BSM 完成后通过 on\_response() 检查结果 |

***

## 6. 异常场景矩阵

### 6.1 LLM 检测各异常场景的处理

| 异常场景           | 触发条件                      |       LLM 结果      | 对短信影响     | 日志      |            告警            |
| :------------- | :------------------------ | :---------------: | :-------- | :------ | :----------------------: |
| IPC 发送失败       | APM 连接断开                  |         放行        | 无影响，走 BSM | ERROR   |             -            |
| APM 响应超时       | 超过 llm\_wait\_timeout\_ms |         放行        | 无影响，走 BSM | WARNING |             -            |
| APM 限流降级       | QPS 超过 llm\_max\_qps      | 放行（is\_success=2） | 无影响，走 BSM | INFO    |             -            |
| 大模型 API 超时     | 超过 llm\_timeout\_ms       | 放行（is\_success=1） | 无影响，走 BSM | WARNING | 连续5次触发 LLM\_API\_TIMEOUT |
| 大模型 API 返回非200 | HTTP 状态码错误                | 放行（is\_success=1） | 无影响，走 BSM | ERROR   |  连续5次触发 LLM\_API\_ERROR  |
| 大模型 API 返回错误码  | code != 0                 | 放行（is\_success=1） | 无影响，走 BSM | ERROR   |    归入 LLM\_API\_ERROR    |
| 认证失败（token）    | client\_id/secret 错误      | 放行（is\_success=1） | 无影响，走 BSM | ERROR   | 连续3次触发 LLM\_AUTH\_FAILED |
| 配置未加载          | 数据库字段为空                   |     LLM 检测不执行     | 无影响，走 BSM | INFO    |             -            |
| IPC 连接断开       | 网络分区                      |       等待队列清空      | 无影响，走 BSM | WARNING |             -            |

### 6.2 核心原则

**LLM 检测是增值能力，任何 LLM 失败都不影响核心短信通路，默认放行保证短信可达。**

***

## 7. 内部通信协议设计

### 7.1 现有协议框架

**复用现有通信链路：** IPM ↔ APM 内部通信

现有协议头定义（`CommunicationProtocol.h`，`MESSAGE_HEAD_SIZE = 16`）：

```cpp
struct message_header {
    ACE_INT32  message_id;   // 功能码/命令字（网络字节序）
    ACE_INT32  version;      // 协议版本
    ACE_INT32  length;       // 包体长度（网络字节序）
    ACE_INT32  sequence;     // 序列号（用于请求-响应匹配）
};
```

**现有 function\_id 参考值：**

| function\_id | 协议名称                   | 说明     |
| ------------ | ---------------------- | ------ |
| 0x01         | LOGIN\_REQ             | 登录请求   |
| 0x02         | LOGOUT\_REQ            | 注销请求   |
| 0x03         | LOGIN\_RESP            | 登录应答   |
| 0x10         | CONFIG\_CHANGE\_NOTIFY | 配置变更通知 |
| 0x80         | MESSAGE\_PROCESS\_REQ  | 消息处理请求 |
| 0x81         | MESSAGE\_PROCESS\_RESP | 消息处理应答 |

### 7.2 LLM 协议扩展

**扩展方式：** 新增 function\_id（0x88/0x89），复用现有消息头和通信链路

| 协议                     | function\_id | 方向        | 说明       |
| ---------------------- | ------------ | --------- | -------- |
| LLMDetectReq\_package  | 0x88         | IPM → APM | LLM 检测请求 |
| LLMDetectResp\_package | 0x89         | APM → IPM | LLM 检测响应 |

**LLM 检测请求协议（IPM → APM）：**

```cpp
#ifdef _LINUX
#pragma pack(push,1)
#endif

struct llm_detect_req {
    ACE_INT64  msg_id;                     // 短信消息ID（用于匹配响应）
    char       sender_num[21];             // 主叫号码（不足补0）
    char       receiptor_num[21];          // 被叫号码（不足补0）
    ACE_INT64  send_time;                  // 发送时间
    ACE_INT16  sms_content_len;            // 短信内容长度，最大512
    // char sms_content[sms_content_len];   // 变长字段，紧跟在结构体后面
};

struct LLMDetectReq_package {
    message_header         pkg_header;      // function_id = 0x88
    llm_detect_req        pkg_body;
    // char sms_content[sms_content_len];  // 变长字段
};

#ifdef _LINUX
#pragma pack(pop)
#endif
```

**LLM 检测响应协议（APM → IPM）：**

```cpp
#ifdef _LINUX
#pragma pack(push,1)
#endif

struct llm_detect_resp {
    ACE_INT64  msg_id;                    // 短信消息ID（与请求对应）
    ACE_INT8   is_success;                // 检测是否成功：0=成功，1=失败，2=限流
    ACE_INT8   result_type;               // 分类结果：0~6
    float       confidence;               // 置信度
    ACE_INT16  error_msg_len;            // 错误信息长度，最大127
    // char error_msg[error_msg_len];      // 变长字段，检测失败时填写
};

struct LLMDetectResp_package {
    message_header        pkg_header;      // function_id = 0x89
    llm_detect_resp       pkg_body;
    // char error_msg[error_msg_len];      // 变长字段
};

#ifdef _LINUX
#pragma pack(pop)
#endif
```

### 7.3 响应 is\_success 字段说明

| is\_success | 含义                   | 后续处理                                     |
| :---------: | -------------------- | ---------------------------------------- |
|      0      | 检测成功，result\_type 有效 | 根据 result\_actions\[result\_type] 执行处置策略 |
|      1      | 检测失败（API 错误/超时/配置错误） | LLM=放行，仅依赖 BSM                           |
|      2      | 限流降级（APM 令牌不足）       | LLM=放行，仅依赖 BSM                           |

### 7.4 分类结果枚举

| 枚举值 | 常量名                     | 分类标签 | 说明           |
| :-: | ----------------------- | ---- | ------------ |
|  0  | LLM\_RESULT\_NORMAL     | 正常   | 正常短信，不拦截     |
|  1  | LLM\_RESULT\_FRAUD      | 涉诈   | 高危，需拦截       |
|  2  | LLM\_RESULT\_PORN       | 涉黄   | 违规内容，建议拦截    |
|  3  | LLM\_RESULT\_GAMBLING   | 涉赌   | 违规内容，建议拦截    |
|  4  | LLM\_RESULT\_AD         | 广告   | 商业广告，可根据策略决定 |
|  5  | LLM\_RESULT\_COLLECTION | 催收   | 催收短信，可根据策略决定 |
|  6  | LLM\_RESULT\_PYRAMID    | 传销   | 传销内容         |

### 7.5 协议打包/解包时序

```mermaid
sequenceDiagram
    participant NDP as NeDataProcess
    participant APCP as CLLMAPMCallProxy
    participant IPC as IPC通道
    participant APM as APM Handler

    NDP->>APCP: send_detect_request(msg_id, sender, receiptor, send_time, sms_content, content_len)
    activate APCP

    Note over APCP: 分配 buffer = sizeof(LLMDetectReq_package) + content_len
    Note over APCP: message_header:<br>· message_id = htonl(0x88)<br>· version = htonl(1)<br>· length = htonl(sizeof(llm_detect_req)+content_len)<br>· sequence = htonl(0)
    Note over APCP: llm_detect_req:<br>· msg_id = CPublicTools::htonl64(msg_id)<br>· sender_num = strncpy(...)<br>· receiptor_num = strncpy(...)<br>· send_time = htonl64(send_time)<br>· sms_content_len = htons(content_len)
    Note over APCP: 拷贝 sms_content 变长字段到 buffer 尾部
    APCP->>APCP: send_to_apm(buffer, pkg_size)

    APCP->>IPC: ACE TCP 发送到 APM
    Note over APCP: 注册 LLMWaitContext<br>m_pending[msg_id] = {send_time, timeout_ms, result_ready=false}

    APCP-->>NDP: return true（异步，不阻塞）
    deactivate APCP

    IPC->>APM: function_id=0x88

    Note over APM: LLMPluginLoader → llm_plugin_detect()<br/>    Note over APM: 库内置流控+统计+周期日志
    Note over APM: 组装 0x89 响应包

    IPC->>APCP: function_id=0x89 响应到达
    APCP->>APCP: on_response(msg_id, out_result)
    Note over APCP: find(msg_id) → 不存在则已超时或非法，丢弃
    Note over APCP: elapsed_ms = (now - send_time).msec()
    Note over APCP: elapsed_ms > timeout_ms → 超时，丢弃
    Note over APCP: elapsed_ms <= timeout_ms → 拷贝结果，移除上下文

    APCP-->>NDP: return true/false
```

***

## 8. 模块设计

### 8.1 模块总览

| 模块                    |   所在进程  | 文件名                                    | 职责                                                      |
| --------------------- | :-----: | -------------------------------------- | ------------------------------------------------------- |
| CLLMAPMCallProxy      |   IPM   | `01_IPM/src/llm/LLMAPMCallProxy.h/cpp` | IPC 发送请求、接收响应、协议打包解包、msg\_id 匹配、超时控制                    |
| CLLMClient            |   IPM   | `01_IPM/src/llm/LLMClient.h/cpp`       | 直连 HTTP 调用、access\_token 缓存管理、get\_action() 策略查询        |
| LLMProtocol           | IPM+APM | `01_IPM/src/llm/LLMProtocol.h`         | 0x88/0x89 协议结构体定义、分类结果常量                                |
| APMHandler            |   APM   | 扩展现有模块                                 | 识别 0x88 路由到 LLM 处理                                      |
| CLLMPluginLoader（IPM） |   IPM   | `01_IPM/src/llm/LLMPluginLoader.h/cpp` | dlopen 加载 libllmplugin.so（直连模式）                         |
| LLMPluginLoader（APM）  |   APM   | `03_APM/src/llm/LLMPluginLoader.h/cpp` | dlopen 加载 libllmplugin.so（中转模式）                         |
| libllmplugin          | IPM/APM | `libllmplugin/`                        | 公共库：令牌桶流控 + OAuth2 token 管理 + HTTP 调用大模型 API + 5 分钟统计日志 |

### 8.2 IPM 侧模块关系

```mermaid
classDiagram
    class CLLMAPMCallProxy {
        +instance() CLLMAPMCallProxy*
        +startup() bool
        +finish() void
        +send_detect_request(msg_id, sender, receiver, send_time, sms_content, content_len) bool
        +on_response(msg_id, out_result) bool
        +is_result_available(msg_id) bool
        +cleanup() void
        -send_to_apm(data, len) bool
        -m_pending: map~ACE_INT64, LLMWaitContext~
        -m_mutex: boost::mutex
    }

    class CLLMPluginLoader {
        +instance() CLLMPluginLoader*
        +load_plugin(config) bool
        +update_plugin_config(config) bool
        +detect(sms_content, sms_id, access_token, result_type, confidence) int
        +unload_plugin() void
        -check_alarm(succeeded, sms_id) void
        -m_handle: void*
        -m_consecutive_failures: int
    }

    class LLMResult {
        +result_type: int
        +confidence: float
        +is_success: bool
        +error_message: string
    }

    class CLLMClient {
        +instance() CLLMClient*
        +startup() bool
        +finish() void
        +detect_spam(sender, receiver, sms_content, out_result) bool
        +get_action(result) int
        +should_block(result) bool
        -get_access_token(out_token) bool
        -send_request(token, sender, receiver, content, out_result) bool
        -m_cached_token: string
        -m_token_expire_time: time_t
        -m_mutex: boost::mutex
    }

    CLLMAPMCallProxy --> LLMResult : 返回检测结果
    CLLMPluginLoader --> LLMResult : 返回检测结果
    CLLMClient --> LLMResult : 返回检测结果
    NeDataProcess --> CLLMAPMCallProxy : send_detect_request() + on_response()（中转模式）
    NeDataProcess --> CLLMPluginLoader : detect()（直连模式）
    NeDataProcess --> CLLMClient : get_action()
```

### 8.3 流控设计（libllmplugin 内置）

**设计原则：** 流控逻辑收进 libllmplugin 公共库内，每个进程独立的令牌桶算法，无需中心协调。

**限流策略：**

| 配置项           | 类型  | 默认值 | 说明                     |
| ------------- | --- | --- | ---------------------- |
| llm\_max\_qps | int | 500 | 每秒最大 LLM 检测请求数，0 表示不限流 |

**令牌桶实现要点：**

`llm_plugin_init()`/`llm_plugin_update_config()` 接收 `max_qps` 参数，初始化/重置令牌桶

每次 `llm_plugin_detect()` 调用时锁内做限流检查：

基于 `steady_clock` 计算时间差，按比例补充令牌

不足 1 个令牌时立即返回 `LLM_RATE_LIMITED (-2)`

- 令牌充足时锁内快照配置，释放锁，执行 HTTP 调用
- HTTP 完成后重新加锁更新统计计数
- 返回 `LLM_RATE_LIMITED` 时，调用方（APM 的 CltSocketSpm / IPM 的 NeDataProcess）以 `is_success=1` 返回，表示检测不可用、放行

**统计计数器（库内维护）：**

| 计数器                     | 类型            | 说明            |
| ----------------------- | ------------- | ------------- |
| s\_total\_count         | uint64\_t     | 总检测请求数        |
| s\_rate\_limited\_count | uint64\_t     | 被限流拒绝的请求数     |
| s\_failed\_count        | uint64\_t     | HTTP 调用失败的请求数 |
| s\_result\_counts\[7]   | uint64\_t\[7] | 各分类结果计数（0\~6） |

**周期日志输出：**

- 每隔 5 分钟通过回调输出一次统计信息，格式：
  `[LLM Perf] total=X, rate_limited=Y, failed=Z, results=[a,b,c,d,e,f,g]`

### 8.4 插件接口设计

```cpp
// LLMPlugin.h - 纯C接口，保证跨语言调用

// 返回码
#define LLM_OK               0   // 成功
#define LLM_ERR             -1   // 通用错误（网络/解析等）
#define LLM_RATE_LIMITED    -2   // 流控拒绝

// 日志级别
#define LLM_LOG_ERROR       1
#define LLM_LOG_WARN        2
#define LLM_LOG_INFO        3
#define LLM_LOG_DEBUG       4

// 日志回调类型 — 库通过此回调输出日志
typedef void (*llm_log_fn)(int level, const char* msg);

#ifdef __cplusplus
extern "C" {
#endif

// 初始化：9 参数，含 max_qps 流控
int llm_plugin_init(const char* api_url, const char* auth_url,
                     const char* app_id, const char* client_id,
                     const char* client_secret, const char* access_token,
                     int timeout_ms, int retry_count,
                     int max_qps);

// 热更新配置（同 9 参数）
int llm_plugin_update_config(const char* api_url, const char* auth_url,
                              const char* app_id, const char* client_id,
                              const char* client_secret, const char* access_token,
                              int timeout_ms, int retry_count,
                              int max_qps);

// 执行检测，返回 LLM_OK / LLM_ERR / LLM_RATE_LIMITED
int llm_plugin_detect(const char* sms_content,
                       const char* sms_id,
                       const char* access_token,
                       int* result_type, float* confidence);

// 注册日志回调（NULL 则取消注册，回退到 stderr）
void llm_plugin_set_log_fn(llm_log_fn callback);

// 释放所有资源
void llm_plugin_cleanup();

#ifdef __cplusplus
}
#endif
```

**接口说明：**

| 函数                          | 职责                               | 实现要点                                   |
| --------------------------- | -------------------------------- | -------------------------------------- |
| llm\_plugin\_init           | 初始化 curl 全局、令牌桶、统计计数器            | 只执行一次，注册日志回调                           |
| llm\_plugin\_update\_config | 热更新配置，重置令牌桶                      | 动态调整 max\_qps                          |
| llm\_plugin\_detect         | 锁内限流检查 → 配置快照 → HTTP 调用 → 锁内更新统计 | 返回 LLM\_OK/LLM\_ERR/LLM\_RATE\_LIMITED |
| llm\_plugin\_set\_log\_fn   | 注册/注销日志回调                        | 调用方（APM/IPM）注册其 LOGGER\_FACTORY 包装器    |
| llm\_plugin\_cleanup        | 释放 curl 资源、重置所有计数器               | 进程退出或插件卸载时调用                           |

***

## 9. 调用流程

### 9.1 NeDataProcess 中 LLM 检测完整时序

```mermaid
sequenceDiagram
    participant NDP as NeDataProcess
    participant APCP as CLLMAPMCallProxy
    participant LC as CLLMClient
    participant IPC as IPC通道
    participant APM as APM侧

    Note over NDP: BSM 黑白名单判断完成，未命中黑/白名单

    NDP->>NDP: 检查 llm_enable && 短信类型在 allowed_msg_types

    NDP->>APCP: send_detect_request(msg_id, sender, receiptor, send_time, sms_content, content_len)
    activate APCP
    Note over APCP: 打包 0x88 → send_to_apm() → 注册等待队列
    APCP->>IPC: 发送 0x88 协议包
    APCP-->>NDP: return true
    deactivate APCP

    Note over NDP: BSM 主线程继续执行（规则检测）

    IPC--)APM: APM 处理 0x88 → 返回 0x89

    Note over NDP: BSM 检测完成

    NDP->>APCP: on_response(msg_id, out_result)
    activate APCP
    Note over APCP: 查找 msg_id → 检查超时 → 拷贝结果
    APCP-->>NDP: return true/false
    deactivate APCP

    alt LLM 结果及时到达且未超时
        NDP->>LC: get_action(llm_result)
        activate LC
        Note over LC: result_actions[result.result_type]<br>→ 0=放行, 1=监控, 2=拦截, 3=加黑
        LC-->>NDP: return int
        deactivate LC
        Note over NDP: 策略=放行: 无操作<br>策略=监控: is_dxcz_monitor=1<br>策略=拦截/加黑 + BSM放行 → CAUSE_LLM_DETECT<br>策略=拦截/加黑 + BSM拦截 → BSM原因
    else LLM 超时或失败
        Note over NDP: LLM=放行，仅依赖 BSM
    end
```

### 9.2 CLLMClient::get\_action() 策略分发逻辑

```mermaid
flowchart TD
    IN["get_action(result)"] --> CHECK_SUCCESS{"result.is_success?"}

    CHECK_SUCCESS -->|false| RETURN0["return LLM_ACTION_PASS (0)<br/>(检测失败，放行)"]

    CHECK_SUCCESS -->|true| LOOKUP["result_actions[result.result_type]"]

    LOOKUP --> DISPATCH{"action?"}

    DISPATCH -->|PASS (0)| NOP["放行: 无操作"]
    DISPATCH -->|MONITOR (1)| MON["监控: is_dxcz_monitor = 1"]
    DISPATCH -->|BLOCK (2)| BLK["拦截: auth=BLACK, llm_triggered_block=true"]
    DISPATCH -->|BLACKLIST (3)| BLL["加黑: auth=BLACK, llm_triggered_block=true"]
```

### 9.3 CLLMClient 直连 HTTP 调用（备用路径）

```mermaid
sequenceDiagram
    participant LC as CLLMClient
    participant Curl as libcurl
    participant Auth as OAuth2认证服务
    participant API as 大模型API

    Note over LC: detect_spam() 被调用

    LC->>LC: get_access_token(token)
    activate LC
    Note over LC: 检查 m_cached_token 是否有效

    alt 缓存有效且未过期
        LC-->>LC: 直接返回缓存 token
    else 缓存过期或为空
        LC->>Curl: POST auth_url/app_id/token
        Curl->>Auth: HTTP POST {grant_type, client_id, client_secret}
        Auth-->>Curl: {access_token, expires_in}
        Curl-->>LC: 解析 → 更新缓存
    end
    deactivate LC

    Note over LC: 使用 token 发起检测

    LC->>Curl: POST api_url (multipart/form-data)
    Note over Curl: Header: Authorization: Bearer {token}<br>Body: prompt={sms_content}, sms_id={id}

    Curl->>API: HTTP 请求
    API-->>Curl: {code, message, sms_id, label, confidence}

    alt code == 0
        Note over LC: result_type = label→枚举<br>confidence = confidence<br>is_success = true
    else code != 0
        Note over LC: is_success = false<br>error_message = message
    end
```

### 9.4 超时机制

**超时配置：**

| 配置项                    | 类型  | 默认值  | 说明                                     |
| ---------------------- | --- | ---- | -------------------------------------- |
| llm\_wait\_timeout\_ms | int | 5000 | IPM 侧 CLLMAPMCallProxy 等待 APM 响应超时（毫秒） |

**超时处理流程（代码** **`LLMAPMCallProxy.cpp:103-129`）：**

```mermaid
flowchart TD
    ARRIVE["APM 响应 0x89 到达<br/>on_response(msg_id, out_result)"] --> LOCK[加锁 m_mutex]

    LOCK --> FIND{"m_pending.find(msg_id)?"}

    FIND -->|"不存在（已清理或非法 msg_id）"| DROP["return false"]

    FIND -->|存在| ELAPSED["计算 elapsed_ms = (now - send_time).msec()"]

    ELAPSED -->|"elapsed_ms > timeout_ms<br/>（超时）"| LOG_WARN["WARNING 日志<br/>'APM response timeout, discard'"]
    LOG_WARN --> ERASE["m_pending.erase(it)<br/>return false"]

    ELAPSED -->|"elapsed_ms <= timeout_ms<br/>（正常）"| COPY["拷贝 LLMResult<br/>移除等待上下文<br/>return true"]
```

***

## 10. 数据库设计

### 10.1 SYS\_COMMON\_VALUE 表结构

LLM 所有配置存储在 `SYS_COMMON_VALUE` 表中，适配 IPM 集群部署模式，集群所有节点自动获得一致配置。

**完整字段清单：**

| 字段名                      | 类型            |       默认值       | 说明                                       |
| ------------------------ | ------------- | :-------------: | ---------------------------------------- |
| llm\_enable              | NUMBER(1)     |        0        | 功能总开关（0=关闭, 1=开启）                        |
| llm\_call\_mode          | NUMBER(1)     |        1        | 0=直连, 1=APM中转                            |
| llm\_max\_qps            | NUMBER(6)     |       500       | 每秒最大请求数，0=不限流                            |
| llm\_retry\_count        | NUMBER(2)     |        2        | 失败重试次数                                   |
| llm\_timeout\_ms         | NUMBER(6)     |      30000      | APM 侧 LLM 检测超时（毫秒）                       |
| llm\_wait\_timeout\_ms   | NUMBER(6)     |       5000      | IPM 侧等待 APM 响应超时（毫秒）                     |
| llm\_api\_url            | VARCHAR2(256) |        -        | 大模型检测 API 地址                             |
| llm\_auth\_url           | VARCHAR2(256) |        -        | OAuth2 认证服务地址                            |
| llm\_app\_id             | VARCHAR2(64)  |        -        | 元景平台分配的应用 ID                             |
| llm\_client\_id          | VARCHAR2(64)  |        -        | OAuth2 客户端 ID                            |
| llm\_client\_secret      | VARCHAR2(256) |        -        | 客户端密钥（AES 加密存储）                          |
| llm\_access\_token       | VARCHAR2(512) |        -        | 静态访问令牌（备用）                               |
| llm\_allowed\_msg\_types | VARCHAR2(128) |        -        | 允许检测的短信类型，逗号分隔                           |
| llm\_result\_actions     | VARCHAR2(128) | `0,3,2,2,1,1,0` | 7个分类0\~6的处置策略编码，逗号分隔。0=放行,1=监控,2=拦截,3=加黑 |

```sql
ALTER TABLE SYS_COMMON_VALUE ADD (llm_call_mode NUMBER(1) DEFAULT 1);
ALTER TABLE SYS_COMMON_VALUE ADD (llm_max_qps NUMBER(6) DEFAULT 500);
ALTER TABLE SYS_COMMON_VALUE ADD (llm_retry_count NUMBER(2) DEFAULT 2);
ALTER TABLE SYS_COMMON_VALUE ADD (llm_wait_timeout_ms NUMBER(6) DEFAULT 5000);
ALTER TABLE SYS_COMMON_VALUE ADD (llm_result_actions VARCHAR2(128) DEFAULT '0,3,2,2,1,1,0');
```

### 10.2 DB 加载 SQL

```sql
select COUNTRY_CODE, SPTYPE_LEN_MIN, SPTYPE_LEN_MAX, SPTYPE_LEN_STATUS,
       llm_enable, llm_api_url, llm_auth_url, llm_app_id, llm_client_id,
       llm_client_secret, llm_access_token, llm_timeout_ms, llm_retry_count,
       llm_allowed_msg_types, llm_result_actions, llm_call_mode, llm_max_qps,
       llm_wait_timeout_ms,
       kafka_ifx_enable, kafka_brokers, kafka_topic, kafka_retries,
       kafka_timeout_ms, kafka_queue_size
from SYS_COMMON_VALUE
```

### 10.3 告警配置

LLM 告警复用现有告警体系，新增以下告警项：

| 告警ID              | 告警描述         | 触发条件                |  级别 | 处理建议               |
| ----------------- | ------------ | ------------------- | :-: | ------------------ |
| LLM\_AUTH\_FAILED | LLM认证连续失败    | 连续3次获取token失败       |  重要 | 检查credentials配置和网络 |
| LLM\_API\_TIMEOUT | LLM检测API连续超时 | 连续5次检测超时            |  警告 | 检查网络和API服务状态       |
| LLM\_API\_ERROR   | LLM检测API连续错误 | 连续5次非200或error code |  警告 | 检查API地址和服务状态       |

***

## 11. 配置项

### 11.1 完整配置清单

| 配置项                      | 类型     |       默认值       | 存储位置 | 说明                                       |
| ------------------------ | ------ | :-------------: | ---- | ---------------------------------------- |
| llm\_enable              | bool   |      false      | DB   | 功能总开关                                    |
| llm\_call\_mode          | int    |        1        | DB   | 0=直连, 1=APM中转                            |
| llm\_max\_qps            | int    |       500       | DB   | 每秒最大请求数                                  |
| llm\_retry\_count        | int    |        2        | DB   | 失败重试次数                                   |
| llm\_api\_url            | string |        -        | DB   | 大模型检测 API 地址                             |
| llm\_auth\_url           | string |        -        | DB   | OAuth2 认证地址                              |
| llm\_app\_id             | string |        -        | DB   | 应用 ID                                    |
| llm\_client\_id          | string |        -        | DB   | 客户端 ID                                   |
| llm\_client\_secret      | string |        -        | DB   | 客户端密钥（加密存储）                              |
| llm\_access\_token       | string |        -        | DB   | 静态令牌（备用）                                 |
| llm\_timeout\_ms         | int    |      30000      | DB   | APM侧超时（毫秒）                               |
| llm\_wait\_timeout\_ms   | int    |       5000      | DB   | IPM侧等待超时（毫秒）                             |
| llm\_allowed\_msg\_types | string |        -        | DB   | 检测的短信类型，逗号分隔                             |
| llm\_result\_actions     | string | `0,3,2,2,1,1,0` | DB   | 7个分类0\~6的处置策略编码，逗号分隔。0=放行,1=监控,2=拦截,3=加黑 |

### 11.2 动态生效项

| 配置项                    | 生效方式 | 实现位置                                                                |
| ---------------------- | ---- | ------------------------------------------------------------------- |
| llm\_enable            | 实时   | CLLMAPMCallProxy / CLLMPluginLoader（按 call\_mode）                   |
| llm\_result\_actions   | 实时   | NeDataProcess 按策略执行（加黑/拦截/监控/放行）                                    |
| llm\_call\_mode        | 实时   | NeDataProcess 路径分支选择                                                |
| llm\_max\_qps          | 实时   | APM: LLMPluginLoader::update\_plugin\_config() → libllmplugin 重置令牌桶 |
| llm\_wait\_timeout\_ms | 实时   | CLLMAPMCallProxy 等待队列                                               |
| llm\_result\_actions   | 实时   | NeDataProcess 策略分发 + CLLMClient::get\_action()                      |

***

### 11.3 界面设计：大联通大模型接口参数配置

**菜单位置:** 接口管理 → 大联通大模型接口参数配置

**页面布局（分栏卡片式，5 个配置区）：**

```
┌─ 基础开关 ────────────────────────────────────────┐
│  [✓] 启用大模型检测                                │
└────────────────────────────────────────────────────┘

┌─ 架构与流控 ──────────────────────────────────────┐
│  调用模式: (●) APM中转 (○) 直连                    │
│  max_qps: [ 500  ] QPS    重试: [ 2 ] 次           │
│  API超时: [30000] ms     等待超时: [5000] ms       │
└────────────────────────────────────────────────────┘

┌─ 接口地址 ────────────────────────────────────────┐
│  API地址:    [_________________]                   │
│  认证地址:   [_________________]  OAuth2           │
│  应用ID:     [_________________]                   │
│  客户端ID:   [_________________]                   │
│  客户端密钥: [••••••••••________] [显示]           │
│  静态令牌:   [_________________]  备用降级          │
└────────────────────────────────────────────────────┘

┌─ 检测范围 ────────────────────────────────────────┐
│  [✓] 点对点短信  [✓] SP短信  [ ] 行业短信  ...    │
└────────────────────────────────────────────────────┘

┌─ 处置策略 ────────────────────────────────────────┐
│  分类标签   枚举值  处置策略                       │
│  正常       0      [放行 ▼]                       │
│  涉诈       1      [加黑 ▼]                       │
│  涉黄       2      [拦截 ▼]                       │
│  涉赌       3      [拦截 ▼]                       │
│  广告       4      [监控 ▼]                       │
│  催收       5      [监控 ▼]                       │
│  传销       6      [放行 ▼]                       │
│  [全部设为监控] [全部设为放行] [恢复默认]          │
└────────────────────────────────────────────────────┘

                                        [保存配置]
```

#### 控件要素与约束

| #  | 分组    | 标签       | DB字段                    | 控件   |  必填 |   默认  | 约束                      |
| -- | ----- | -------- | ----------------------- | ---- | :-: | :---: | ----------------------- |
| 1  | 基础开关  | 启用大模型检测  | `llm_enable`            | 复选框  |  -  | false | 关闭时整表单灰显                |
| 2  | 架构与流控 | 调用模式     | `llm_call_mode`         | 单选组  |  是  | 1(中转) | 0=直连,1=中转；切换弹确认框        |
| 3  | 架构与流控 | max\_qps | `llm_max_qps`           | 数字框  |  是  |  500  | 1\~10000, 0=不限流         |
| 4  | 架构与流控 | 重试次数     | `llm_retry_count`       | 数字框  |  是  |   2   | 0\~5                    |
| 5  | 架构与流控 | API超时    | `llm_timeout_ms`        | 数字框  |  是  | 30000 | 1000\~60000 ms          |
| 6  | 架构与流控 | 等待超时     | `llm_wait_timeout_ms`   | 数字框  |  是  |  5000 | 1000\~30000 ms；直连模式灰显   |
| 7  | 接口地址  | API地址    | `llm_api_url`           | 文本框  |  是  |   -   | http(s):// 开头, max 256  |
| 8  | 接口地址  | 认证地址     | `llm_auth_url`          | 文本框  |  否  |   -   | 空则使用静态令牌, max 256       |
| 9  | 接口地址  | 应用ID     | `llm_app_id`            | 文本框  |  是  |   -   | max 64                  |
| 10 | 接口地址  | 客户端ID    | `llm_client_id`         | 文本框  |  是  |   -   | max 64                  |
| 11 | 接口地址  | 客户端密钥    | `llm_client_secret`     | 密码框  |  是  |   -   | 加密存储, 有\[显示]切换, max 256 |
| 12 | 接口地址  | 静态令牌     | `llm_access_token`      | 文本框  |  否  |   -   | 认证地址为空时作为降级, max 512    |
| 13 | 检测范围  | 短信类型     | `llm_allowed_msg_types` | 复选框组 |  是  |   -   | 至少一项, 后端逗号分隔            |
| 14 | 处置策略  | 正常→策略    | `llm_result_actions[0]` | 下拉框  |  是  | 放行(0) | 选项: 放行/监控/拦截/加黑         |
| 15 | 处置策略  | 涉诈→策略    | `llm_result_actions[1]` | 下拉框  |  是  | 加黑(3) | 选项: 放行/监控/拦截/加黑         |
| 16 | 处置策略  | 涉黄→策略    | `llm_result_actions[2]` | 下拉框  |  是  | 拦截(2) | 选项: 放行/监控/拦截/加黑         |
| 17 | 处置策略  | 涉赌→策略    | `llm_result_actions[3]` | 下拉框  |  是  | 拦截(2) | 选项: 放行/监控/拦截/加黑         |
| 18 | 处置策略  | 广告→策略    | `llm_result_actions[4]` | 下拉框  |  是  | 监控(1) | 选项: 放行/监控/拦截/加黑         |
| 19 | 处置策略  | 催收→策略    | `llm_result_actions[5]` | 下拉框  |  是  | 监控(1) | 选项: 放行/监控/拦截/加黑         |
| 20 | 处置策略  | 传销→策略    | `llm_result_actions[6]` | 下拉框  |  是  | 放行(0) | 选项: 放行/监控/拦截/加黑         |

#### 短信类型复选框枚举

| 枚举值 | 标签    |   默认   |
| :-: | ----- | :----: |
|  1  | 点对点短信 |    ✓   |
|  2  | SP短信  |    ✓   |
|  3  | 行业短信  | <br /> |
|  4  | 国际短信  | <br /> |
|  5  | 彩信    |    ✓   |
|  6  | 闪信    | <br /> |

#### 处置策略选项表

|  策略 |  编码 | 含义              | 后端行为            |
| :-: | :-: | --------------- | --------------- |
|  放行 |  0  | 正常通过，不做处理       | 无操作             |
|  监控 |  1  | 仅记录监控入库，不下发拦截   | 复用 BSM 监控入库     |
|  拦截 |  2  | 拦截该条短信          | 现有 LLM block 逻辑 |
|  加黑 |  3  | 主叫号码加黑 + 拦截该条短信 | 复用 IPM 加黑接口     |

#### 联动规则

| 触发器                             | 联动行为                                                                      |
| ------------------------------- | ------------------------------------------------------------------------- |
| `llm_enable`=false              | 整个表单禁用，仅保留此开关可操作                                                          |
| `call_mode`=直连(0)               | 「等待超时」灰显，提示"直连模式无需此配置"                                                    |
| `llm_auth_url`为空                | 「静态令牌」旁提示"已作为认证凭据"                                                        |
| 保存时 `llm_enable`=true           | 校验 api\_url/app\_id/client\_id/client\_secret 必填，allowed\_msg\_types 至少一项 |
| 保存时 `llm_enable`=false          | 仅保存开关状态，其余字段跳过校验                                                          |
| 保存前                             | 数字字段校验范围，密码字段为空时保留数据库原值                                                   |
| **保存成功且** **`call_mode`=中转(1)** | **弹窗提示"配置已保存，请重启 APM 及所有 IPM 节点使配置生效"**                                   |
| **保存成功且** **`call_mode`=直连(0)** | **弹窗提示"配置已保存，请重启所有 IPM 节点使配置生效"**                                         |

#### 保存行为说明

- 前端按联动规则做约束校验，未通过时红框标出错误字段
- 校验通过后提交到后端更新 `SYS_COMMON_VALUE` 表并触发热加载
- 保存成功后按 `call_mode` 弹出对应的重启提示（见联动规则表末两行）
- 密码字段 `llm_client_secret` 页面显示遮盖值，提交时若为空则表示不修改（保留数据库原值）

***

### 11.4 内存配置结构（SystemConfigCollection.h）

```cpp
struct LLMConfig {
    bool        enable;                    // 功能总开关
    int         call_mode;                 // 0=直连, 1=APM中转
    int         max_qps;                   // 每秒最大请求数
    int         retry_count;               // 失败重试次数
    string      api_url;                   // 大模型检测API地址
    string      auth_url;                  // OAuth2认证地址
    string      app_id;                    // 应用ID
    string      client_id;                 // 客户端ID
    string      client_secret;             // 客户端密钥
    string      access_token;              // 静态令牌
    int         timeout_ms;                // APM侧超时
    int         wait_timeout_ms;           // IPM侧等待超时
    vector<int> allowed_msg_types;         // 允许检测的消息类型
    vector<int> result_actions;            // 7个分类0~6的处置策略编码

    LLMConfig()
        : enable(false), call_mode(1), max_qps(500), retry_count(2),
          timeout_ms(30000), wait_timeout_ms(5000) {
        // 默认: 0=放行,1=加黑,2=拦截,3=拦截,4=监控,5=监控,6=放行
        int default_actions[] = {0, 3, 2, 2, 1, 1, 0};
        result_actions.assign(default_actions, default_actions + 7);
    }
};
```

***

## 12. 文件清单

### 12.1 新增文件

| 文件路径                                 | 说明                              |
| ------------------------------------ | ------------------------------- |
| `01_IPM/src/llm/LLMAPMCallProxy.h`   | IPC 中转代理头文件                     |
| `01_IPM/src/llm/LLMAPMCallProxy.cpp` | IPC 中转代理实现                      |
| `01_IPM/src/llm/LLMProtocol.h`       | 0x88/0x89 + 0x8A/0x8B 协议结构体定义   |
| `03_APM/src/llm/LLMProtocol.h`       | 同上（APM 侧副本，与 IPM 侧一致）           |
| `01_IPM/src/llm/LLMPluginLoader.h`   | IPM 插件加载器（直连模式）                 |
| `01_IPM/src/llm/LLMPluginLoader.cpp` | IPM 插件加载器实现 + 日志桥接 + 告警         |
| `03_APM/src/llm/LLMPluginLoader.h`   | APM 插件加载器（中转模式）                 |
| `03_APM/src/llm/LLMPluginLoader.cpp` | APM 插件加载器实现 + 日志桥接 + 告警         |
| `libllmplugin/include/LLMPlugin.h`   | 公共库插件接口（含返回码、日志级别、回调类型）         |
| `libllmplugin/src/LLMPlugin.cpp`     | 公共库插件实现（含令牌桶流控 + 统计 + 5 分钟周期日志） |

### 12.2 修改文件

| 文件路径                                           | 修改内容                                                                                                     |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `01_IPM/src/ModulePublicDefine.h`              | 新增 `CAUSE_LLM_DETECT = -37` 阻止原因枚举值                                                                      |
| `01_IPM/src/config/SystemConfigCollection.h`   | LLMConfig 新增 call\_mode/max\_qps/wait\_timeout\_ms/result\_actions，移除 block\_enable/block\_result\_types |
| `01_IPM/src/config/SystemConfigDBLoader.cpp`   | SQL 新增列 + 配置加载逻辑（result\_actions 逗号分隔解析）                                                                 |
| `01_IPM/src/config/SystemConfigFileLoader.cpp` | ini 文件新增配置项读取（result\_actions 替换 block\_result\_types）                                                   |
| `01_IPM/src/comm/ApmHandle.cpp`                | LLM\_CONFIG\_UPDATE\_REQ 处理：解析 result\_actions 字符串替换旧 block 字段                                           |
| `01_IPM/src/process/NeDataProcess.cpp`         | LLM 并行检测 + call\_mode 分支（直连/中转）+ 策略分发（加黑/拦截/监控/放行）+ CAUSE\_LLM\_DETECT                                   |
| `01_IPM/src/process/BusinessControl.cpp`       | 启动初始化/析构时加载/卸载插件（直连模式）                                                                                   |
| `01_IPM/src/llm/LLMClient.h/cpp`               | 新增 `get_action()` 方法读取 result\_actions，`should_block()` 底层改用 result\_actions                             |
| `01_IPM/src/llm/LLMProtocol.h`                 | result\_actions\[64] 替换 block\_enable + block\_result\_type\_count + block\_result\_types                |
| `01_IPM/src/protocol/ProtocolFactory.h`        | 包含 LLMProtocol.h                                                                                         |
| `03_APM/src/SystemConfigCollection.h`          | 同 IPM 侧，LLMConfig 新增 result\_actions、移除 block\_enable/block\_result\_types                               |
| `03_APM/src/SystemConfigDBLoader.cpp`          | SQL 加载 result\_actions 替换 block\_result\_types                                                           |
| `03_APM/src/ProtocolFactory.h`                 | 包含 LLMProtocol.h                                                                                         |
| `03_APM/src/llm/LLMPluginLoader.h/cpp`         | 更新函数指针为 9 参数（+max\_qps）、新增日志回调注册                                                                         |
| `03_APM/src/CltSocketSpm.h/cpp`                | 新增 `send_llm_config_update()` 打包 result\_actions 发送给 IPM；处理 LLM\_RATE\_LIMITED(-2)                       |
| `03_APM/src/CltProxy.h/cpp`                    | 广播 LLM\_CONFIG\_UPDATE\_REQ 配置到所有 IPM 节点                                                                 |
| `03_APM/src/BusinessControl.cpp`               | 删除 LLMFlowController、初始化时传递 max\_qps 给插件                                                                 |
| `03_APM/src/llm/LLMFlowController.h/cpp`       | **删除** — 流控已移至 libllmplugin 内置                                                                           |
| `03_APM/src/llm/LLMProtocol.h`                 | result\_actions\[64] 替换旧 block 字段（同 IPM 侧）                                                               |
| `libllmplugin/include/LLMPlugin.h`             | 新增返回码/日志级别/日志回调 type/9 参数接口                                                                              |
| `libllmplugin/src/LLMPlugin.cpp`               | 新增令牌桶流控、统计计数器、5 分钟周期日志                                                                                   |
| `libllmplugin/CMakeLists.txt`                  | 新增 ANTISPAM\_INC\_ROOT 可选 SDK 路径                                                                         |

***

## 13. 部署说明

| 部署项             | 说明                                   |
| --------------- | ------------------------------------ |
| libllmplugin.so | 中转模式：部署在 APM 服务器；直连模式：部署在所有 IPM 服务器  |
| 大模型API地址        | 中转模式：APM 可访问外网即可；直连模式：各 IPM 节点需能访问外网 |
| llm\_max\_qps   | 页面可配置，默认 500 QPS，实时生效（每进程独立限速）       |
| IPM/APM         | 配置变更后需重启 IPM 和 APM 节点使配置生效           |
| DB 扩展           | 执行 ALTER TABLE 新增 5 个字段（见 10.1）      |

***

## 文档结束
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTk2MzA5MzM2OV19
-->