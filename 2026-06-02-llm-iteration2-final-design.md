# LLM 反欺诈迭代二：拦截/加黑 + 可配置分类映射 方案设计

**版本:** v2.1  
**日期:** 2026-06-02（更新 2026-06-03）  
**所属项目:** antispaming 2850  
**前置依赖:** 迭代一（LLM 异步检测 + 监控入库）

### 修订历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v2.0 | 2026-06-02 | 初版：可配置标签映射 + 拦截/加黑 + 汇聚裁决 |
| v2.1 | 2026-06-03 | 移除所有硬编码类型数量限制：新增 `LLM_MAX_RESULT_TYPES=128` 常量（覆盖 ACE_INT8 全范围 0-127），`s_result_counts[8]` → 动态大小，IPC `result_actions[64]` → `[2048]`，统计日志动态生成。新增 LLM 类型只需添加关键字即可，类型编号自动递增 |

---

## 1. 概述

### 1.1 迭代一做了什么

迭代一实现了 LLM 大模型检测的基础能力：短信送到大模型识别分类，识别结果写入监控表 `DATA_LLM_MONITOR`，供运营查询分析。**迭代一中 LLM 只监控，不拦截、不加黑。**

### 1.2 迭代二要做什么

迭代二包含两大块需求：

**需求块 A — 拦截/加黑：** LLM 从"只监控"升级为"可以拦截和加黑"。配置了拦截策略的短信类型，LLM 直接触发拦截；配置了加黑策略的类型，LLM 把发送号码加入黑名单。BSM 和 LLM 两个检测系统的结果需要**汇聚裁决**。

**需求块 B — 可配置分类映射：** 当前 LLM API 返回的短信分类（正常/涉诈/涉黄/涉赌/广告/催收/传销）与类型编号（0-6）的映射是**硬编码在 libllmplugin 中的**。当大模型接口新增分类（如"涉暴"、"涉政"等），必须重编译 plugin 才能支持。迭代二将标签映射改为**数据库可配置**，运营可动态新增分类并配置其拦截/加黑策略。

### 1.3 一句话总结

> 在迭代一的基础上，LLM 检测结果从"仅监控"升级为"可以拦截/加黑"。LLM 短信分类标签与类型编号的映射从硬编码改为数据库配置，支持动态新增分类。当配置了拦截或加黑策略时，BSM 和 LLM 的结果需要汇聚裁决。

---

## 2. 需求汇总

| 编号 | 需求 | 说明 |
|------|------|------|
| R1 | 支持直连和中转两种模式 | 直连：IPM 直接调用 LLM API；中转：IPM→APM→LLM API |
| R2 | 无拦截/加黑配置时保持现有行为 | 不汇聚、不等待、fire-and-forget，与迭代一一致 |
| R3 | 有拦截/加黑配置时汇聚裁决 | BSM 和 LLM 结果都回来后做最终判断，支持早返 |
| R4 | 不送 LLM 的消息不等待 | 保持现有 BSM 单独裁决流程 |
| R5 | BSM 数据优先 + 融合 LLM 类型 | 双方都拦截时，拦截原因以 BSM 为准 |
| R6 | 最小改动 | 不影响现有 BSM 检测、IPM 黑白名单、APM 中转逻辑 |
| **R7** | **LLM 标签映射可配置** | 标签→类型编号映射从 SYS_LLM_CONFIG 表加载，不硬编码 |
| **R8** | **result_actions 动态大小** | result_actions 支持任意数量的类型，不再固定 7 个 |
| **R9** | **动态新增分类** | 运营通过修改 DB 配置即可新增 LLM 分类，无需重编译 |
| **R10** | **类型 7+ 是可配置的真实分类** | 当前类型 7 硬编码为"未分类→始终放行"，改为可配置的真实分类 |
| **R11** | **向后兼容** | 默认配置与当前硬编码行为完全一致，存量部署无感知 |
| **R12** | **SAM 界面可配置分类标签** | 运营通过 Web 管理界面配置标签→类型映射和对应的处置策略 |

---

## 3. 需求块 B 详细设计：可配置标签映射

### 3.1 问题描述

当前硬编码在 `LLMPlugin.cpp:884-897`：

```cpp
if (label == "正常")      *result_type = 0;
else if (label == "涉诈") *result_type = 1;
else if (label == "涉黄") *result_type = 2;
else if (label == "涉赌") *result_type = 3;
else if (label == "广告") *result_type = 4;
else if (label == "催收") *result_type = 5;
else if (label == "传销") *result_type = 6;
else                      *result_type = 7;  // 未分类 = 始终放行
```

同时 `result_actions` 向量固定 7 个元素（类型 0-6），类型 7 硬编码为 PASS。

**问题**：大模型新增分类（如"涉暴"、"涉政"），系统不支持，被迫映射为"未分类"，且无法为新分类配置拦截策略。

### 3.2 数据库变更

#### 新增列：`llm_label_mapping`

```sql
-- SYS_LLM_CONFIG 表新增列
ALTER TABLE SYS_LLM_CONFIG ADD llm_label_mapping VARCHAR2(1024) DEFAULT '';

COMMENT ON COLUMN SYS_LLM_CONFIG.llm_label_mapping IS
'LLM标签到类型映射, JSON格式: {"标签":"类型号",...}
 未匹配的标签由系统硬编码映射到保留类型7(LLM_RESULT_UNKNOWN),始终放通.
 类型编号7为系统保留位,不可在JSON中使用.
 空字符串或NULL表示使用内置默认映射(与当前硬编码一致)';
```

#### 更新 `llm_result_actions` 注释

```sql
-- 旧注释: '7个处置位图, 对应类型0-6...'
-- 新注释:
COMMENT ON COLUMN SYS_LLM_CONFIG.llm_result_actions IS
'N个处置位图, 逗号分隔, 索引对应标签映射中的类型编号:
 bit0=拦截,bit1=监控,bit2=加黑(0=放行).
 元素数量与标签映射中的类型总数一致';
```

### 3.3 标签映射格式

**存储格式**：Mini-JSON 字符串（无需引入 JSON 库，手写轻量解析器）

**默认值**（与当前硬编码完全一致）：
```json
{"正常":0,"涉诈":1,"涉黄":2,"涉赌":3,"广告":4,"催收":5,"传销":6}
```

**扩展示例**（大模型新增"涉暴"、"涉政"分类）：
```json
{"正常":0,"涉诈":1,"涉黄":2,"涉赌":3,"广告":4,"催收":5,"传销":6,"涉暴":8,"涉政":9}
```

**关键设计变更**：`*` 键不再出现在 JSON 中。未匹配到任何标签的返回值，由系统**硬编码映射到保留类型 `LLM_RESULT_UNKNOWN`**，始终放通。

### 3.3a 保留位：未识别分类（Unrecognized / Unknown）

为保障系统安全，设置一个**独立于可配置类型的保留位**，用于处理 LLM API 返回的所有无法匹配标签的分类结果。

| 属性 | 值 |
|------|-----|
| **常量名** | `LLM_RESULT_UNKNOWN` |
| **类型编号** | `7`（固定，与现有协议兼容） |
| **标签** | `"未分类"` / `"UNKNOWN"` |
| **行为** | **硬编码始终放通（PASS）** — 不拦截、不加黑、不监控 |
| **可配置性** | **不可配置** — 此类型不在 `result_actions` 中，不受任何配置影响 |
| **用途** | 安全兜底：LLM 接口返回新分类标签但尚未配置映射时，自动归入此类型 |

**在 `get_action()` 中的处理**（`LLMClient.cpp`）：

```cpp
int CLLMClient::get_action(const LLMResult& result) {
    if (result.is_success != 0) {
        return 0;  // 失败/限流 → PASS
    }

    // 保留位：未识别分类 → 硬编码放通，不做任何处理
    // 此类型不在 result_actions 配置范围内，确保 LLM 返回未知标签时
    // 系统安全兜底，不会误拦截正常短信
    if (result.result_type == LLM_RESULT_UNKNOWN) {
        return 0;  // 始终 PASS
    }

    const std::vector<int>& actions = SYS_CONFIG->get_llm_config().result_actions;
    if (result.result_type >= 0 && result.result_type < (int)actions.size()) {
        return actions[result.result_type];
    }

    // 超出 actions 范围 → 安全默认 PASS
    return 0;
}
```

**在 libllmplugin 中的处理**（`LLMPlugin.cpp::parse_detect_response()`）：

```cpp
// 查找标签映射
std::map<std::string, int>::const_iterator it = s_label_map.find(label);
if (it != s_label_map.end()) {
    *result_type = it->second;       // 已知标签 → 配置的类型编号
} else {
    *result_type = LLM_RESULT_UNKNOWN;  // 未知标签 → 保留位 7，始终放通
}
```

**对 SAM 界面的影响**：

- 标签映射表格**不包含**"未分类"行（它是保留位，不可配置）
- 不需要"未知标签映射到"下拉框（始终映射到保留位 7）
- 运营只需配置**已知的、有明确分类意义的标签**

**约定**：`llm_label_mapping` JSON 中的类型编号不能使用 7（保留位）。SAM 后端验证时检查是否有类型编号 = 7，如有则拒绝保存。

### 3.4 Plugin API 变化

#### 新增函数

```c
// llm_plugin_update_label_mapping — 更新标签到类型编号的映射
// @param label_mapping_json  JSON映射字符串, NULL或空字符串恢复内置默认
// 返回 LLM_OK(0) 成功, LLM_ERR(-1) 解析失败
int llm_plugin_update_label_mapping(const char* label_mapping_json);
```

#### 内部实现

- libllmplugin 内部用 `std::map<std::string, int>` 存储标签→类型映射
- `s_catch_all_type` 记录 `*` 键对应的 fallback 类型（默认 7）
- `parse_detect_response()` 中的硬编码 if/else 链替换为 map 查找
- 加载时 `dlsym` 获取新符号，若不存在（旧版本 plugin）则优雅跳过

### 3.5 配置加载流程

```
DB SYS_LLM_CONFIG.llm_label_mapping
    ↓ (SystemConfigDBLoader 读取)
LLMConfig.label_mapping (std::string)
    ↓ (BusinessControl / ApmHandle)
LLMPluginLoader::update_label_mapping(json)
    ↓ (dlsym → llm_plugin_update_label_mapping)
libllmplugin: 替换 s_label_map
    ↓ (parse_detect_response 使用新映射)
LLM API 返回标签 → 映射为类型编号 → 查找 result_actions
```

### 3.6 IPC 协议变化

`llm_config_update` 结构体变更：

```cpp
// CommunicationProtocol.h — LLM 常量区新增
const ACE_INT16 LLM_MAX_RESULT_TYPES = 128;  // ACE_INT8 range (0-127), type 7 reserved

struct llm_config_update {
    // ... 现有字段不变 ...
    ACE_INT32  blacklist_duration_min;
    ACE_INT32  label_mapping_len;  // NEW: 0=无映射(用默认), >0=后续JSON长度
    // char   label_mapping[label_mapping_len];  // 追加在固定结构体之后
};
```

同时 `result_actions` 字段从 `char[64]` 扩容为 `char[2048]`，以支持 `LLM_MAX_RESULT_TYPES`（128 个类型）的逗号分隔策略值传输。

- `label_mapping_len = 0` → IPM 端使用内置默认映射（向后兼容）
- `label_mapping_len > 0` → 从包尾读取对应长度的 JSON 字符串

### 3.7 向后兼容

| 场景 | 行为 |
|------|------|
| `llm_label_mapping` 为空 | Plugin 使用内置默认映射（与当前硬编码一致） |
| 旧 plugin 无 `llm_plugin_update_label_mapping` 符号 | dlsym 返回 NULL，跳过调用，plugin 用内置映射 |
| 旧 IPM 收新 IPC 包 | `label_mapping_len` = 0（旧 APM 不填），无影响 |
| `result_actions` 比映射需要的元素少 | 自动补 0（PASS），记录告警日志 |

### 3.8 类型 7 处理变化

| 项目 | 改前 | 改后 |
|------|------|------|
| 类型 7 含义 | 硬编码"未分类"，始终 PASS | **系统保留位 `LLM_RESULT_UNKNOWN`**：硬编码"未识别分类"，始终放通，不可配置 |
| `result_actions` 大小 | 固定 7 (类型 0-6) | 动态，`result_actions.size()` 等于 `llm_label_mapping` 中配置的最大类型编号+1 |
| 类型 7 在 result_actions 中 | 不在（硬编码 PASS） | **不在**（保留位，不参与 `result_actions` 索引） |
| `get_action(type=7)` | `if (type == 7) return 0;` | `if (type == LLM_RESULT_UNKNOWN) return 0;` （硬编码，不可绕过） |
| 未知标签处理 | 硬编码 → 类型 7 | 查询 `s_label_map` → 未匹配时 → `LLM_RESULT_UNKNOWN`（类型 7）→ 始终 PASS |
| 类型 8+ | 不存在 | 可配置的真实分类，通过 `result_actions[8]` 等控制 |

**安全性保障**：即使 `result_actions` 被错误配置（例如只配了 3 个值但标签映射引用了类型 8），`get_action()` 对越界的 `result_type` 也会返回 0（PASS），不会误拦截。保留位 7 提供双重兜底。

### 3.9 SAM Web 管理界面设计

SAM 是 Struts 2 + Hibernate 的 Java Web 应用（`05_SAM/`），所有配置管理遵循统一模式：**JSP → Action → Service → DAO → DB，然后通过 NIO Socket 通知 APM 重载配置**。

#### 当前 LLM 配置页面

**页面文件**:
- JSP: `05_SAM/antispam2850/WebRoot/pages/qg/interfacemanage/SysLlmCfg.jsp`
- JS: `05_SAM/antispam2850/WebRoot/assets/js/qg/interfacemanage/SysLlmCfg.js`
- Action: `05_SAM/antispam2850/src/.../action/SysLlmCfgAction.java`
- Service: `05_SAM/antispam2850/src/.../service/SysLlmCfgService.java`
- Model: `05_SAM/antispam2850/src/.../model/SysLlmCfg.java` + `SysLlmCfg.hbm.xml`
- Bean: `05_SAM/antispam2850/src/.../bean/SysLlmResultAction.java`

当前页面包含基本开关、性能参数、连接参数、消息类型多选、以及 **7 个固定行**的处置策略（类型 0-6，每行 3 个复选框：拦截/监控/加黑）。

#### 改造方案：处置策略表格动态化

**改后 UI 布局**：

```
┌─ LLM 分类标签配置 ──────────────────────────────────────────────┐
│                                                                  │
│  ※ 未识别分类（保留位）：LLM 返回未知标签时，始终放通，不可配置    │
│                                                                  │
│  标签名称        类型编号    拦截   监控   加黑    操作            │
│  ┌──────────┐  ┌────┐    ┌──┐   ┌──┐  ┌──┐  ┌────┐            │
│  │ 正常     │  │ 0  │    │  │   │  │  │  │  │    │            │
│  │ 涉诈     │  │ 1  │    │✓ │   │✓ │  │  │  │ 删除│            │
│  │ 涉黄     │  │ 2  │    │✓ │   │  │  │  │  │ 删除│            │
│  │ 涉赌     │  │ 3  │    │✓ │   │  │  │  │  │ 删除│            │
│  │ 广告     │  │ 4  │    │  │   │✓ │  │  │  │ 删除│            │
│  │ 催收     │  │ 5  │    │  │   │✓ │  │  │  │ 删除│            │
│  │ 传销     │  │ 6  │    │  │   │  │  │  │  │ 删除│            │
│  │ 涉暴     │  │ 8  │    │✓ │   │  │  │  │  │ 删除│            │
│  │ 涉政     │  │ 9  │    │✓ │   │  │  │  │  │ 删除│            │
│  └──────────┘  └────┘    └──┘   └──┘  └──┘  └────┘            │
│                                                                  │
│  [+ 新增分类标签]  类型编号自动分配（跳过保留位 7）                │
└──────────────────────────────────────────────────────────────────┘
```

**关键交互**：
- **动态行**：每行展示标签名（可编辑）、类型编号（只读，自动分配）、3 个处置策略复选框、删除按钮
- **新增按钮**：点击弹出输入框，填写标签名称，自动分配下一个可用类型编号（跳过保留位 7）
- **类型编号 7 不出现**：它是系统保留位（未识别分类 → 始终放通），不在此表格中显示
- **前 7 行有默认值**（类型 0-6），与当前硬编码一致，确保向后兼容
- **新增行从类型 8 开始**编号，运营可为新 LLM 分类创建条目

**SAM 后端验证**：
- 标签名称不能重复
- 类型编号不能重复
- **类型编号不能为 7**（保留位）
- 类型编号不能超过 127（`ACE_INT8` 上限）

#### 数据流（保存时）

```
前端 JS:
  1. 用户点击"保存"
  2. saveData() 遍历表格每一行
  3. 构建 JSON 字符串: {"正常":0,"涉诈":1,...,"涉暴":8,"涉政":9}
     （注意：保留位 7 不出现在 JSON 中，未知标签由系统硬编码映射到 LLM_RESULT_UNKNOWN）
  4. 同时构建 result_actions: "0,5,2,2,3,3,0,1,1" (类型0-6+8+9=9个值)
  5. 将 JSON 赋给隐藏字段 labelMapping
  6. 将 result_actions 赋给隐藏字段 resultActions
  7. AJAX POST 到 SysLlmCfg!modify.do

后端 Service (SysLlmCfgService.beforeSave):
  8. 接收 labelMapping JSON 字符串
  9. 验证 JSON 格式、标签名不重复、类型编号不冲突
  10. 写入 SYS_LLM_CONFIG.llm_label_mapping
  11. 写入 SYS_LLM_CONFIG.llm_result_actions
  12. socket() 发送 ConfigChangeNotifyReq(CHANGE_NOTIFY_SYS_LLM_CONFIG)
      通知 APM 重载

APM C++ 端:
  13. SystemConfigDBLoader 读取新的 llm_label_mapping 和 result_actions
  14. LLMPluginLoader::update_label_mapping() 传给 libllmplugin
  15. libllmplugin 替换 s_label_map
  16. 后续 detect 调用使用新映射
```

#### SAM 侧改动文件清单

| 文件 | 改动内容 |
|------|---------|
| `SysLlmCfg.jsp` | 处置策略区域从固定 7 行改为动态表格；新增标签名称输入列；新增"添加分类"按钮和"未知标签映射"下拉框 |
| `SysLlmCfg.js` | 新增 `addLabelRow()`、`removeLabelRow()` 函数；`saveData()` 中序列化动态表格为 JSON；验证标签名不重复 |
| `SysLlmCfgAction.java` | `init()` 中解析 `llm_label_mapping` JSON 传给 JSP 渲染动态行；`modify()` 接收前端传来的映射数据 |
| `SysLlmCfgService.java` | `beforeSave()` 中解析/验证标签映射 JSON；保存到新增的 `llm_label_mapping` 列 |
| `SysLlmCfg.java` (Model) | 新增 `labelMapping` 字段（String） |
| `SysLlmCfg.hbm.xml` | 新增 `llm_label_mapping` 属性映射（对应 DB 新列） |
| `SysLlmResultAction.java` | 新增 `labelName` 字段；构造改为动态（不再固定 7 个实例） |

#### 向后兼容处理

- `llm_label_mapping` 为空或 NULL → 前端渲染默认 7 行（正常/涉诈/涉黄/涉赌/广告/催收/传销 + 未分类），与当前页面样式一致
- 旧版 JSP（升级前）→ result_actions 仍按 7 个值处理，C++ 端使用内置默认映射
- 不需要数据迁移脚本更新现有行 —— 空 label_mapping 等同于使用内置默认

---

## 4. 需求块 A 详细设计：拦截/加黑 + 汇聚裁决

> 以下内容基于现有迭代二设计文档 `2026-05-20-llm-iteration2-block-blacklist-design.md`，整合到本方案中。

### 4.1 复用现有框架：SMSC GT Flood Cache 汇聚模式

IPM 中已有一套生产验证的**多维度异步响应汇聚框架** — `CSmscGTMonitorCacheData`（文件：`01_IPM/src/SmscGTMonitor/SmscGTCacheData.h/.cpp`）。

#### 现有框架的核心模式

```
请求到达 → 确定需要几个维度的分析(N个)
         → request_times = N, result_times = 0
         → 发送 N 个异步请求到 BSM
         |
         ├─ 任一维度返回 BLOCK → 立即应答 (早返) → 清理缓存
         ├─ 维度返回 PASS/UN_ANALYSIS → result_times |= bitmask
         │   ├─ if result_times == request_times → 汇聚裁决 → 应答 → 清理
         │   └─ else → 继续等待
         └─ 定时器(50ms) → 超时 → 按已有结果强制裁决
```

**关键数据结构** (`TSMSCGTCacheData`, `ModulePublicDefine.h:1791-1807`):
```cpp
typedef struct {
    unsigned short  request_times;   // 预期响应数
    unsigned short  result_times;    // 已完成维度 bitmask
    int             critial_result;  // 当前最严重结果
    ACE_UINT64      cache_time;      // 超时计时
    // ... 缓存数据
} TSMSCGTCacheData;
```

#### 映射到 BSM + LLM 汇聚

| SMSC GT Flood | BSM + LLM 汇聚 | 说明 |
|---------------|---------------|------|
| `request_times` = N (多维度) | `expected_count` = 2 (BSM + LLM) | 只需等两方 |
| `result_times` bitmask | `bsm_arrived` + `llm_arrived` 标志 | 简化版本 |
| `critial_result` | `decision` (UNDECIDED/BLOCK/PASS) | 汇聚裁决结果 |
| 任一方 BLOCK → 早返 | **相同** — BSM 或 LLM 任一 BLOCK → 立即应答 | 完全复用 |
| 全到达 → 汇聚裁决 | **相同** — 双方都到 → 最终裁决 | 完全复用 |
| 定时器超时 → 强制裁决 | **相同** — `wait_timeout_ms` 超时 → Fail Open | 完全复用 |

#### 复用方式

**方案：在 `CLLMAPMCallProxy` 中扩展 `MonitorContext` 为 `MergeContext`**

当前的 `MonitorContext`（只跟踪 LLM 结果）扩展为 `MergeContext`（同时跟踪 BSM 和 LLM）：

```cpp
// 当前结构（仅监控）
struct MonitorContext {
    char msg_id[128];
    ACE_Time_Value send_time;
    char sender[64], receiver[64], sm_id[128], sm_content[512];
};

// 迭代二扩展为 MergeContext（汇聚裁决）
struct MergeContext {
    char       msg_id[128];
    ACE_Time_Value create_time;       // 超时计时

    // BSM 侧
    bool       bsm_arrived;
    int        bsm_result;            // PERMIT / FORBID
    int        bsm_stop_cause;        // 拦截原因
    char       bsm_stop_reason[256];

    // LLM 侧
    bool       llm_arrived;
    int        llm_result_type;       // LLM 分类 (0-7+)
    float      llm_confidence;
    bool       llm_is_success;

    // 裁决
    int        decision;              // UNDECIDED / BLOCK / PASS
    char       sender[64], receiver[64], sm_id[128], sm_content[512];
};

std::map<std::string, MergeContext> m_merge_map;  // 替代当前 m_pending
```

**关键方法**（复用 SMSC GT Flood 模式）：

```cpp
// 当任一结果到达时调用，参考 process_result_resp()
void on_result_arrived(const std::string& msg_id, bool is_bsm, ...);

// 裁决逻辑
void try_merge_decision(MergeContext& ctx);

// 早返：任一 BLOCK → 立即应答
void early_block_response(MergeContext& ctx, int source);  // source: BSM or LLM

// 双方都到 → 最终裁决
void final_merge_decision(MergeContext& ctx);

// 超时清理（复用现有 cleanup_expired 定时器）
void cleanup_expired();
```

**与 BSM 流程的集成点**：

当前 BSM 处理链：`BsmHandle → BsmDataProcess::handle_respond_result() → respond_ne()`

迭代二改为：当 MergeContext 存在时 → BSM 结果先进入 MergeContext 汇聚，不直接 respond_ne：

```cpp
// BsmDataProcess::handle_respond_result() 中：
if (CLLMAPMCallProxy::instance()->has_merge_context(msg_id)) {
    // 走汇聚路径
    CLLMAPMCallProxy::instance()->on_bsm_result(msg_id, result, stop_cause, ...);
    return;  // 不直接应答，由汇聚逻辑统一处理
}
// 否则走原有直接应答路径（路径A）
respond_ne(...);
```

### 4.2 三种路径

| 路径 | 触发条件 | LLM 发送方式 | 是否等待 LLM | 最终裁决方式 |
|------|---------|------------|------------|------------|
| **路径A** | 不送 LLM，或仅配置监控 | fire-and-forget | 否 | BSM 单独决定 |
| **路径B** | 配置拦截/加黑 + 直连模式 | IPM 本地异步调用 | **是** | BSM + LLM 汇聚 |
| **路径C** | 配置拦截/加黑 + 中转模式 | IPM→APM→LLM | **是** | BSM + LLM 汇聚 |

### 4.2 MergeContext 数据结构

每条走汇聚路径的短信，在 IPM 内存中维护：

| 字段 | 来源 | 说明 |
|------|------|------|
| `msg_id` | IPM 生成 | 唯一标识，用于匹配 BSM 和 LLM 结果 |
| `bsm_status` | BSM 响应 | `PENDING` / `BLOCK` / `PASS` |
| `llm_status` | LLM 响应 | `PENDING` / `BLOCK` / `PASS` |
| `bsm_data` | BSM 响应 | stop_cause, stop_reason 等拦截原因 |
| `llm_data` | LLM 响应 | result_type, confidence |
| `decision` | 裁决结果 | `UNDECIDED` / `BLOCK` / `PASS` |
| `create_time` | 系统时间 | 用于超时清理 |

**LLM 结果如何映射为 BLOCK/PASS**（适配可配置映射后）：
- `is_success != 0`（失败/限流）→ PASS
- `result_type >= result_actions.size()` → PASS（安全默认）
- `result_actions[result_type]` 含 `LLM_ACTION_BLOCK` 位 → BLOCK
- 其他 → PASS

### 4.3 汇聚裁决规则

1. **早返规则**：任一方先返回 BLOCK，立即向短信中心应答拦截，不等另一方
2. **放行需共识**：先返回 PASS 不能直接放行，必须等双方都 PASS 才最终放行
3. **BSM 优先入库**：双方都 BLOCK 时，DATA_SM_STOP 的 stop_cause/stop_reason 以 BSM 为准，LLM 分类作为附加字段

### 4.4 异常场景矩阵

| 异常场景 | 对短信影响 | 处理方式 |
|---------|----------|---------|
| LLM API 超时 | 不阻塞 | BSM 单独裁决 |
| LLM 限流跳过 | 不阻塞 | 视为 PASS |
| LLM 返回错误 | 不阻塞 | 视为 PASS |
| BSM 超时 | 按 LLM 结果 | 超时后以 LLM 结果为准 |
| IPC 连接断开 | 不阻塞 | 降级为路径A（仅 BSM） |
| MergeContext 超时 | 放行 | Fail Open |

### 4.5 配置复用

迭代二**不新增配置项**（拦截/加黑部分），复用迭代一配置：

| 配置项 | 用途 |
|--------|------|
| `llm_enable` | 总开关 |
| `llm_call_mode` | 0=直连, 1=中转 |
| `llm_result_actions` | 每个类型的处置位图（迭代二 BLOCK/BLACKLIST 位生效） |
| `llm_wait_timeout_ms` | 汇聚等待超时（默认 2000ms） |

### 4.6 行为变化

| 配置 | 迭代一 | 迭代二 |
|------|--------|--------|
| `result_actions = 0`（放行） | 不监控不拦截 | 不变 |
| `result_actions = 2`（监控） | 监控入库 | 不变 |
| `result_actions = 1`（拦截） | **仅监控入库** | **真正拦截 + 监控入库** |
| `result_actions = 4`（加黑） | **仅监控入库** | **拦截+加黑+监控入库** |

---

## 5. 数据库变更汇总

### 5.1 SYS_LLM_CONFIG 表

| 变更 | 说明 |
|------|------|
| 新增列 `llm_label_mapping VARCHAR2(1024)` | 标签→类型映射 JSON |
| 更新列注释 `llm_result_actions` | 从"7个处置位图"改为"N个处置位图" |

### 5.2 已有字段（迭代一已添加，迭代二真正写入）

| 表 | 字段 | 用途 |
|-----|------|------|
| DATA_SM_STOP | llm_result_type | LLM 分类结果 |
| DATA_SM_STOP | llm_confidence | LLM 置信度 |
| DATA_SM_STOP | llm_triggered | 拦截触发方（0=BSM, 1=LLM） |
| DATA_SM_STOP | llm_message_id | LLM 检测消息 ID |
| DATA_BLKLISTS | llm_result_type | LLM 分类结果 |
| DATA_BLKLISTS | sms_id | 消息 ID |

### 5.3 新增 SYS_CODING_INFO 条目

| CODING_FLAG | CODING_NO | CONTENT |
|-------------|-----------|---------|
| 627 | -38 | 阻止-LLM反诈检测拦截 |
| 610 | 7 | AI模型识别（加黑类型） |
| 632 | 5 | AI模型识别（被叫黑名单处理类型） |

---

## 6. 涉及改动范围

### 6.1 libllmplugin 侧

| 文件 | 改动 |
|------|------|
| `LLMPlugin.h` | 新增 `llm_plugin_update_label_mapping()` 声明；新增 `#define LLM_MAX_RESULT_TYPES 128`（覆盖 ACE_INT8 全范围 0-127） |
| `LLMPlugin.cpp` | 新增 `s_label_map`、`s_catch_all_type`、`init_default_label_map()`、`llm_plugin_update_label_mapping()`；`parse_detect_response()` 标签查找改为 map 查询；**`s_result_counts[8]` → `[LLM_MAX_RESULT_TYPES]`**（所有栈/静态数组统一使用该常量）；**统计日志格式动态生成**（仅打印非零类型，类型 8+ 使用数字标签）；`record_detect_result()` 边界检查适配动态大小 |

### 6.2 APM 侧

| 文件 | 改动 |
|------|------|
| `SystemConfigCollection.h` | LLMConfig 新增 `label_mapping`；更新构造函数默认值 |
| `SystemConfigDBLoader.cpp` | SQL 新增 `llm_label_mapping` 列；移除 `result_actions` 大小 7 的限制 |
| `LLMPluginLoader.h/.cpp` | 新增 `m_plugin_update_label_mapping` 函数指针；`update_label_mapping()` 方法 |
| `CltSocketSpm.cpp` | `send_llm_config_update()` 追加 label_mapping 可变长负载 |
| `BusinessControl.cpp` | 配置加载/重载后调用 `update_label_mapping()` |

### 6.3 IPM 侧（含拦截/加黑汇聚逻辑）

| 文件 | 改动 |
|------|------|
| `config/SystemConfigCollection.h` | LLMConfig 新增 `label_mapping`；result_actions 动态大小 |
| `config/SystemConfigDBLoader.cpp` | SQL 新增列；移除大小限制 |
| `comm/ApmHandle.cpp` | 解析 label_mapping 可变长负载；**LLM 响应路由到 MergeContext 汇聚** |
| `llm/LLMPluginLoader.h/.cpp` | 直连模式：新增函数指针、`update_label_mapping()` |
| `llm/LLMClient.h/.cpp` | 移除类型 7 硬编码；`get_action()` 越界保护；**新增 `needs_merge()`** |
| **`llm/LLMAPMCallProxy.h/.cpp`** | **MonitorContext → MergeContext 扩展；新增 BSM+LLM 汇聚裁决逻辑（复用 SMSC GT Flood 模式）；新增 `on_bsm_result()`/`on_llm_result()`；新增异步入库队列；超时清理** |
| **`process/BsmDataProcess.cpp`** | **BSM 响应处理中判断是否有 MergeContext，有则路由到汇聚路径而非直接应答** |
| **`process/NeDataProcess.cpp`** | **BSM 发送前检查是否需要汇聚（`needs_merge()`），是则创建 MergeContext** |
| `BusinessControl.cpp` | 直连模式初始化 libllmplugin 并注册回调 |

**核心复用**：汇聚裁决框架直接复用 `CSmscGTMonitorCacheData` 的 `request_times`/`result_times`/早返/定时超时模式，代码结构一致，已在生产环境验证。

### 6.4 SAM Web 管理端

| 文件 | 改动 |
|------|------|
| `SysLlmCfg.jsp` | 处置策略区域改为动态表格；标签名称列；新增/删除按钮 |
| `SysLlmCfg.js` | 动态行增删逻辑；JSON 序列化提交 |
| `SysLlmCfgAction.java` | `init()` 解析 label_mapping JSON 传给 JSP |
| `SysLlmCfgService.java` | `beforeSave()` 验证/保存 label_mapping JSON |
| `SysLlmCfg.java` (Model) | 新增 `labelMapping` 字段 |
| `SysLlmCfg.hbm.xml` | 新增 `llm_label_mapping` 属性映射 |
| `SysLlmResultAction.java` | 新增 `labelName`；改为动态构造 |

### 6.5 IPC 协议

| 文件 | 改动 |
|------|------|
| `include/CommunicationProtocol.h` | 新增 `LLM_MAX_RESULT_TYPES = 128` 常量；`llm_config_update` 新增 `label_mapping_len` 字段；`result_actions[64]` → `[2048]` 支持动态类型数 |
| `21_PUB_DLL/05_include/CommunicationProtocol.h` | 同上（libllmplugin 编译期副本） |

### 6.6 数据库脚本（三脚本规则）

| 文件 | 改动 |
|------|------|
| `create_sa_table.sql` | SYS_LLM_CONFIG 新增 llm_label_mapping 列 |
| `init_sa_data_zh.sql` / `_en.sql` | 默认 INSERT 增加 llm_label_mapping |
| `UPDATE/V100R00C52GS02_V100R00C52GS01.SQL` | 新升级脚本 |
| `UPDATE/V100R00C52GS01_V100R00C52GS02_ROLLBACK.SQL` | 新回滚脚本 |

---

## 7. 与迭代一的兼容性

| 场景 | 兼容性 |
|------|--------|
| 现有配置 `result_actions` 仅含 MONITOR | **完全兼容**，行为不变（路径A） |
| 现有没有配置 BLOCK/BLACKLIST 的客户 | **不受影响** |
| 升级后配置了 BLOCK/BLACKLIST | **走汇聚路径**，LLM 参与拦截决策 |
| llm_label_mapping 为空 | **使用内置默认映射**，与当前行为一致 |
| 旧版 libllmplugin.so（无新符号） | dlsym 失败 → 跳过，使用内置映射 |
| 升级后关闭 `llm_enable` | **完全回到迭代一之前的行为** |

---

## 8. 实现顺序

1. **数据库脚本**（4 个文件）— 独立，可先执行
2. **libllmplugin**（LLMPlugin.h + LLMPlugin.cpp）— 标签映射改为可配置
3. **IPC 协议**（CommunicationProtocol.h）— 结构体变更
4. **APM 侧**（配置加载 + plugin 传参 + IPC 发送）
5. **IPM 侧**（配置加载 + IPC 解析 + 汇聚裁决 + 拦截/加黑）
6. **SAM Web 界面**（JSP + JS + Action + Service + Model）— 动态标签映射管理

---

## 9. 风险与缓解

| 风险 | 缓解 |
|------|------|
| 汇聚逻辑引入新 bug | **复用 SMSC GT Flood Cache 生产验证框架** — `request_times`/`result_times`/早返/定时超时模式已在多维度 BSM 分析中稳定运行 |
| 标签映射 JSON 解析失败 | 回退内置默认映射，记录 ERROR 日志 |
| result_actions 比映射需要的类型少 | 自动补 0（PASS），记录告警 |
| 旧 plugin 无 `llm_plugin_update_label_mapping` 符号 | dlsym 判空，优雅跳过 |
| IPC 包过大 | label_mapping 限制 1024 字节 |
| 汇聚等待增加延迟 | 早返机制：任一方 BLOCK 立即应答 |
| LLM 误拦截正常短信 | 默认策略偏保守；运营可调整 |
| MergeContext 内存泄漏 | 复用 `cleanup_expired()` 超时清理机制 |
