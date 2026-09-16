# 推荐效果分析 Agent：Dify 多 Agent 工作流设计方案

> 本文只设计 **Dify 工作流层**：入口、规则路由、Main Supervisor、Professional Sub-Agent、并行执行、结果汇合、Prompt 与会话承接。
>
> Elasticsearch / DB 查询、指标计算、统计检验、异常检测、贡献拆解、参数合法性和大结果持久化继续由现有 Plugin Tool Contract 负责。工作流只消费 Tool Schema 与 Tool Observation，不在本方案内重复定义 Tool 内部实现。

---

## 1. 整体设计

### 1.1 目标

工作流采用 **Fast Signal Extractor + Fast Rule Router + Main Supervisor + Professional Sub-Agent** 的分层方式。入口先由 Code 节点只提取可确定识别的显式信号，再由 Fast Rule Router 基于这些信号进行保守路由；明确、单域、可确定的请求直接进入对应 Sub-Agent，复杂、多目标、存在指代或需要语义理解的请求进入 Main Supervisor，由 Main Supervisor 拆成独立 Agent Objective，再由 Dify Workflow 并行执行 Professional Sub-Agent。

```text
用户请求
  ↓
Start
  ↓
Fast Signal Extractor（Code）
  ↓
Sensitive Metric Gate
  ↓
Fast Rule Router
  ├─ 明确 effect ──────────────┐
  ├─ 明确 experiment ──────────┤
  ├─ 明确 investigation ───────┤
  │                            │
  └─ 复杂 / 多目标 / 无法确定 → Main Supervisor
                               │
                     统一 routing_result
                               ↓
                     Professional Agents
                  ┌────────────┼────────────┐
                  ↓            ↓            ↓
              Effect       Experiment   Investigation
               Agent          Agent          Agent
                  └────────────┼────────────┘
                               ↓
                        Result Collector
                               ↓
                   ┌───────────┴───────────┐
                   ↓                       ↓
              单 Agent                多 Agent
             直接输出              Synthesis LLM
                   └───────────┬───────────┘
                               ↓
                             Answer
```

核心职责固定为：

```text
Fast Signal Extractor：决定“原文中有哪些可以由代码确定识别的显式信号”
Fast Rule Router：决定“这些显式信号是否足以直接交给某个专业 Agent”
Main Supervisor：决定“复杂请求应拆成哪些独立 Agent Objective”
Sub-Agent：决定“为了完成自己的 Objective 调什么 Tool、下一步做什么”
Tool：决定“数据怎么查、指标怎么算”
Dify Workflow：决定“Agent 节点如何并行、汇合和输出”
```

### 1.2 设计原则

| 原则 | 规则 |
| --- | --- |
| 工作流只负责编排 | Fast Signal Extractor、Fast Rule Router、Main Supervisor、Agent Dispatcher、Result Collector、Synthesis 构成控制面；业务查询和计算保留在 Tool。 |
| Signal Extractor 只做确定识别 | Code 只通过固定词典、正则和确定映射提取显式 site、指标 token、ID、强意图信号等 `fast_hints`；不做开放式语义理解，不补默认值，不生成正式 Tool 参数。 |
| Fast 保守 | Fast Rule Router 只消费 `fast_hints` 与原始请求做高置信单域路由；命中多个业务域、存在多目标、需要理解依赖、存在指代或不能确定归属时进入 Main Supervisor。 |
| Main Supervisor 只拆 Agent Objective | Main Supervisor 不选择 Tool、不生成 Tool 参数、不规划 Tool DAG、不读取业务 Observation。 |
| Agent Objective 必须独立 | 只有可以独立完成、最终再合并的目标才能拆到不同 Agent 并行执行。存在数据依赖时由一个 Agent 内部完成整条链路。 |
| 原因类目标收敛到 Investigation | “为什么、原因、导致、是否被某因素影响”等需要 Observation 驱动取证的问题由 Investigation Agent 完成现象确认和后续调查，不先拆 Effect / Experiment 再跨 Agent 依赖。 |
| Agent 自主 Tool Calling | Professional Agent 根据 Objective、已确认参数、Tool Schema 和 Observation 自主选择 Tool；工作流不建立 Tool 级 Router。 |
| Agent 之间并行 | Main Supervisor 生成多个独立 Agent Objective 时，Dify Workflow 并行启动相应 Agent 节点。 |
| 同域请求合并 | 同一轮中属于同一 Professional Agent 的多个独立要求合并为一个 Objective，首版每个 Agent 最多启动一次。 |
| 公共业务背景单一维护 | EC10 / EC20、页面、指标、敏感项、Evidence 边界维护为 `shared_business_background`，按 Prompt 清单注入，不在各角色重复改写。 |
| Prompt 与 Tool Schema 分工 | Prompt 定义角色、决策边界和证据规则；字段、枚举、默认值和参数上限以 Tool Schema 为准。 |
| 当前输入优先 | 会话历史只用于用户明确承接上一轮时解析省略信息；当前请求中的新条件覆盖历史条件。 |
| 缺口最小追问 | 只追问会改变业务语义或无法合法执行的必要事实；有 Tool 明确公开默认值时不追问。 |
| 结果不升级 | Contribution、Anomaly、Level Shift、时间重合和局部诊断信号按原证据力度表达，不自动升级为根因。 |
| 失败和无数据分离 | `no_data`、`partial`、`failed`、`unsupported`、`null` 与数值 0 分别解释。 |
| 结果汇总不产生新事实 | Multi-Agent Synthesis 只组织 Sub-Agent 已取得的事实、结论和限制，不调用 Tool、不补计算、不新增原因判断。 |

### 1.3 参考项目与采用方式

本方案只采用能够直接映射到 Dify 工作流的实现方式。

| 参考 | 已核对的实现方式 | 本方案采用内容 |
| --- | --- | --- |
| Dify 官方 `langgenius/dify` | Workflow Agent 节点支持 `roster_agent` / `inline_agent`；Agent v2 由 Dify Runtime 管理 Tool Calling；Workflow 原生承担分支与并行执行 | Professional Agent 建成可复用 Roster Agent；主工作流只负责路由、并行和汇合 |
| `svcvit/Awesome-Dify-Workflow` | Advanced Chatflow 中使用 Question Classifier、Agent、Tool、变量汇合等原生节点组合业务流程 | 采用“入口规则 / 语义节点 → Agent → Answer”的工作流组织方式，不建立独立调度服务 |
| `BannyLon/DifyAIA` | Agent 型应用可直接由 Agent 理解用户请求并生成 Tool 参数；Workflow 中的 Code 节点用于确定性转换和结构整理，不要求所有 Agent 前统一增加一次 LLM Parameter Extractor | 入口只用 Code 提取 Fast 所需显式信号；复杂语义和真正 Tool 参数交给 Main / Professional Agent |
| `datawhalechina/self-dify` | 复杂流程使用 Dify 原生分支、变量、Iteration 等控制节点 | 并行和汇合交给 Workflow Runtime，动态业务取证仍留在 Agent 内部 |
| `AdamPlatin123/Open-Deep-Research-workflow-on-Dify` | 预先确定的批处理步骤由 Workflow 编排；模型负责语义任务，Workflow 负责执行结构 | 只借鉴“模型决策与 Workflow 执行结构分离”，不把原因调查预先画成固定 Tool DAG |
| Mentor Supervisor / Specialist Agent | Supervisor 负责分工，专业 Agent 负责各自业务域和 Tool 调用 | 采用 Main Supervisor + Effect / Experiment / Investigation 三类 Professional Agent 的职责边界 |

### 1.4 Prompt 设计标准与原则

本节规定 Prompt 的组织、装配和维护方式。节点是否调用、分支条件、输入来源和失败路径仍由对应工作流节点定义；Prompt 不替代工作流协议。

#### 1.4.1 组织与维护标准

| 原则 | Prompt 组织标准 | 边界 |
| --- | --- | --- |
| 单一职责 | 先写角色与任务，再写可信输入、处理规则、输出协议、自检 | Main Supervisor 不承担 Tool 调用；Professional Agent 不承担跨 Agent 路由；Synthesis 不承担调查 |
| 公共和专属分离 | 公共业务背景只维护一份；各 Agent 只追加角色专属规则 | 不在 Effect / Experiment / Investigation 各复制一份站点、页面和指标口径 |
| 指令和资料分离 | System Prompt 放稳定规则；User Prompt 只放本轮动态输入 JSON | 用户输入、历史摘要、Agent Objective、Tool Observation 不写进稳定规则正文 |
| 输出协议自包含 | 输出字段、枚举、状态关系和示例放在同一 Prompt 的 `<output_contract>` | 不要求模型通过章节号自行查字段 |
| 示例不创造事实 | 示例只展示结构和行为，数值、日期、ID 明确为假设值 | 不从示例补当前站点、时间、实验角色或指标值 |
| Tool Schema 不复制 | Prompt 只说明 Tool 使用边界 | Tool 字段、枚举、默认值、TopN 限制以运行时 Tool Schema 为准 |
| 单次调用指令完整 | 最终装配给模型的 System Prompt 必须包含角色所需全部稳定规则 | 不依赖模型读取本 MD 其他章节 |
| 规则就近 | 路由规则放 Main Supervisor；Evidence / Completion 放 Professional Agent；整题合并规则放 Synthesis | 相似文字不跨角色删除必要约束 |
| 自检不新增执行 | `<final_self_check>` 只检查即将输出的结构和范围 | 自检阶段不再调用 Tool、不重新规划 |

推荐 Prompt 阅读顺序固定为：

```text
<role>
→ {{ shared_business_background }}（需要时）
→ <trusted_inputs>
→ 角色处理规则
→ <output_contract>
→ <final_self_check>
```

#### 1.4.2 Prompt 注入清单

| 节点 | System Prompt | User Prompt | 公共业务背景 | Tool Schema |
| --- | --- | --- | --- | --- |
| Fast Signal Extractor | 无 | 无 | 无 | 无 |
| Fast Rule Router | 无 | 无 | 无 | 无 |
| Main Supervisor | `main_supervisor_prompt` | `main_supervisor_input` | 是 | 否 |
| Effect Agent | `professional_agent_common_prompt + effect_agent_profile` | `professional_agent_input` | 是 | Dify 自动提供已授权 Tool Schema |
| Experiment Agent | `professional_agent_common_prompt + experiment_agent_profile` | `professional_agent_input` | 是 | Dify 自动提供已授权 Tool Schema |
| Investigation Agent | `professional_agent_common_prompt + investigation_agent_profile` | `professional_agent_input` | 是 | Dify 自动提供已授权 Tool Schema |
| Multi-Agent Synthesis | `multi_agent_synthesis_prompt` | `multi_agent_synthesis_input` | 是 | 否 |
| Sensitive Gate / IfElse / Result Collector / Answer | 无 | 无 | 无 | 无 |

#### 1.4.3 Prompt 动态输入边界

动态输入只通过 User Prompt JSON 传入，不拼接进 System Prompt 的规则段。

| 动态输入 | Main Supervisor | Professional Agent | Synthesis |
| --- | --- | --- | --- |
| `raw_query` | 是 | 是 | 是 |
| `fast_hints` | 是 | 是 | 否 |
| `request_datetime` | 是 | 是 | 是 |
| 明确承接的会话上下文 | 是 | 按路由结果传入 | 否 |
| `objective` | 否 | 是 | 通过 AgentResult 已包含 |
| `known_parameters` | 否 | 是 | 否 |
| Tool Observation | 否 | Agent Runtime 内部 | 只读取 AgentResult，不读取原始 Tool JSON |
| Sub-Agent Results | 否 | 否 | 是 |

### 1.5 `shared_business_background` 完整 Prompt

```text
<business_background>

1. EC10表示港台推荐站点，EC20表示大陆推荐站点。两个站点的数据、页面映射和业务维度独立；各项分析及其结果必须保留明确的站点范围，不跨站点混合统计，也不在缺少依据时默认选择站点。

2. 页面名称和编号具有站点范围。对应关系按站点分别登记如下，箭头左侧是完整页面名称，右侧是整数编号。某名称未在当前站点列出时，不得套用其他站点编号。

EC10（港台推荐）：
- 全部 → 0
- 商品详情页 → 1
- 商品清单页 → 2
- 搜索页 → 3
- 所有商品页 → 4
- 购物车页 → 5
- mini购物车页 → 6
- blog页 → 7

EC20（大陆推荐）：
- 全部 → 0
- 商品详情页 → 1
- 商品清单页 → 2
- mini购物车页 → 3
- 搜索页 → 4
- 购物车页 → 5
- 订单完成页 → 7
- 404页 → 8
- 推荐落地 → 10

mini购物车页和购物车页是不同页面，必须保留完整名称；不得猜测未登记名称的编号，不得根据页面名称反推站点。店铺ID本身不能证明所属站点。

3. 推荐效果包括曝光、点击、转化、订单、推荐引导GMV及公开比率指标。CTR是推荐点击量除以推荐曝光量；CVR是推荐转化量除以推荐点击量；CTCVR是推荐转化量除以推荐曝光量。未限定的GMV表示rec_gmv推荐引导GMV，不扩大为店铺GMV或推荐GMV占比。

4. store_gmv、store_gmv_per_user和rec_gmv_ratio是禁止公开的敏感指标。不得为其生成查询、排序、比较、归因、计算或展示任务；不得通过公开结果估算或反推，也不得替换为其他公开指标。请求命中任意一项时整轮停止，同轮公开部分也不执行。

5. 页面、店铺、推荐模式、策略、召回和实验分组是不同业务维度，各个值必须保留其角色。control和treatment是用户在本次比较中明确赋予的角色，不能根据编号或结果表现猜测。策略或召回比较不自动等于A/B实验。

6. 过滤限定数据范围，分组决定结果粒度，比较表达对象、角色或周期之间的对照。出现某个维度或值不自动表示按它分组或比较；用户的过滤、分组和比较要求必须分别保留。

7. 整体范围与其中的具体值不能重复计入。整体比例不等于各组比例的简单平均，取决于同一统计口径下汇总的分子和分母。不同范围的结果不能仅因指标同名就合并或相互替代。

8. 无数据、覆盖不足、不可计算、能力不支持、查询失败和数值0必须分别表述。null含义依据结果中的状态及原因解释，不能一律解释为0；指标被排除时不得声称已返回该指标。

9. 推荐请求量/PV、系统QPS/耗时/错误、店铺局部诊断捕获具有不同统计口径，不能混算。店铺诊断来自局部Top候选捕获；未捕获不代表正常，capture_rate不是真实故障率。

10. 数值来源、异常、贡献、时间重合和结构变化不能自动证明原因。尚未查询、查询失败、无数据、证据不足和已经排除是不同状态；原因结论必须有相应证据支持。

11. 推荐效果指标按日T+1产出。“最近、近期、近N天”等相对时间除非明确包含今天，否则以昨天为最新完整日期；用户明确指定今天时不得改成昨天。本规则不适用于实时运行流量。无数据不构成扩大原查询窗口或改写原时间范围的依据。

</business_background>
```

---

## 2. 阶段一：请求预处理

### 2.1 Start 节点

**节点类型：** Start

#### 输入协议

| 字段 | 来源 | 必填 | 用途 |
| --- | --- | --- | --- |
| `sys.query` | Dify | 是 | 当前用户原始请求 |
| `sys.datetime` | Dify | 是 | 相对时间解释 |
| `sys.conversation_id` | Dify | 是 | 当前会话标识 |
| Chat History / Agent Memory | Dify 原生 | 否 | 后续模型节点处理明确承接式表达 |

#### 处理

Start 原样接收本轮请求，不解析业务、不拆 Agent Objective、不生成 Tool 参数。

#### 规则

| 情况 | 处理 |
| --- | --- |
| 新问题 | 原样进入 Fast Signal Extractor |
| 当前输入与历史冲突 | 后续模型以当前输入为准 |
| 用户只写“那昨天呢”“这个实验呢”等承接表达 | 保留原文和会话上下文，不在 Start 补参数 |

#### 输出协议

```json
{
  "raw_query": "{{ sys.query }}",
  "request_datetime": "{{ sys.datetime }}"
}
```

#### 样例

输入：

```text
EC10 购物车最近 CTR 为什么下降？
```

输出：

```json
{
  "raw_query": "EC10 购物车最近 CTR 为什么下降？",
  "request_datetime": "2026-09-17T02:00:00+08:00"
}
```

### 2.2 Fast Signal Extractor

**节点类型：** Code

该节点只为 Fast Rule Router 准备确定性 `fast_hints`。它不是通用参数提取器，不调用 LLM，也不替 Main / Professional Agent 理解完整用户语义。

#### 输入协议

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `raw_query` | string | 当前用户原始请求 |
| `request_datetime` | string | 原样透传；仅确定日期表达需要时使用 |

#### 处理

Code 通过固定词典、正则、明确格式和无歧义映射识别显式信号，并统一大小写、数组类型和空值。无法确定的内容保持未识别，不尝试补全。

```text
raw_query
  ↓
固定词典 / 正则匹配
  ↓
值标准化 + 去重
  ↓
fast_hints
```

#### 规则

| 信号 | 可识别内容 | 处理规则 |
| --- | --- | --- |
| `explicit_site` | 明确出现 `EC10` / `EC20` | 标准化为大写；未出现则不填 |
| `explicit_metrics` | CTR、CVR、CTCVR、GMV、曝光、点击、转化等登记指标名称 | 只做登记词到 canonical token 的确定映射；去重 |
| `explicit_page_names` | 完整登记页面名称，如“购物车页”“mini购物车页” | 保留页面名称；不在此转换 `scene` |
| `explicit_merchant_ids` | 明确带“店铺/merchant”角色的 ID | 只在角色明确时提取；裸数字不猜角色 |
| `explicit_ab_ids` | 明确带“实验/ab/流量组”等角色的 ID | 只在角色明确时提取 |
| `explicit_control_ab_id` | 明确写出“对照/control”对应的 ab_id | 不按 ID 大小或结果猜角色 |
| `explicit_treatment_ab_ids` | 明确写出“实验组/treatment”对应的 ab_id | 可为数组 |
| `reason_signal` | 明确原因问法，如“为什么”“原因”“什么导致”“是不是…造成” | 只匹配强原因表达；存在否定/引用等歧义时不直接置真 |
| `experiment_signal` | A/B、对照组/实验组、control/treatment、明确 ab_id 语义 | 只产生域信号，不生成实验 Tool 参数 |
| `effect_signal` | 明确效果指标、趋势、周期比较、排名、推荐流量等登记表达 | 只产生域信号 |
| `reference_signal` | “这个、刚才、那个实验、继续看”等明显承接词 | 标记需要上下文语义理解，Fast 不直达 |
| `multi_clause_signal` | 明确的多动作连接形式，且两侧都命中不同强域信号 | 只作为 Fast 放弃直达的信号，不负责拆句 |
| 时间表达 | 仅保留明显原始片段或确定日期 | 不把“最近”“之前”“同期”等模糊表达擅自解析成正式时间范围 |

Code 只允许做确定性规范化：

```text
大小写统一
canonical token 映射
数组去重
空字符串 → 未识别
明确 ID 类型标准化
固定词典命中
```

Code 不允许做：

```text
merchant_id → 猜 site
page_name → 在 site 未确认时转 scene
两个 ab_id → 猜 control / treatment
“最近” → 自动补 7 天
根据用户句意拆 Agent Objective
选择 Tool 或生成 Tool arguments
```

#### 输出协议

```json
{
  "raw_query": "...",
  "request_datetime": "...",
  "fast_hints": {
    "explicit_site": "EC10",
    "explicit_metrics": ["ctr"],
    "explicit_page_names": ["购物车页"],
    "explicit_merchant_ids": [],
    "explicit_ab_ids": [],
    "explicit_control_ab_id": null,
    "explicit_treatment_ab_ids": [],
    "reason_signal": true,
    "experiment_signal": false,
    "effect_signal": true,
    "reference_signal": false,
    "multi_clause_signal": false,
    "raw_time_expressions": ["最近"]
  }
}
```

未识别字段使用固定空值：标量为 `null`，数组为 `[]`，布尔信号为 `false`。该输出只供工作流规则和下游 Agent 参考，不代表正式业务参数已经确认完整。

#### 样例

输入：

```text
EC10 购物车最近 CTR 为什么下降？
```

输出：

```json
{
  "fast_hints": {
    "explicit_site": "EC10",
    "explicit_metrics": ["ctr"],
    "explicit_page_names": ["购物车页"],
    "explicit_merchant_ids": [],
    "explicit_ab_ids": [],
    "explicit_control_ab_id": null,
    "explicit_treatment_ab_ids": [],
    "reason_signal": true,
    "experiment_signal": false,
    "effect_signal": true,
    "reference_signal": false,
    "multi_clause_signal": false,
    "raw_time_expressions": ["最近"]
  }
}
```

输入：

```text
看看 12345 最近是不是有问题。
```

Code 不知道 `12345` 是 merchant_id、ab_id 还是其他 ID，因此不赋角色；后续 Fast 不能直达，交 Main Supervisor 处理。

### 2.3 Sensitive Metric Gate

**节点类型：** If/Else / Code

#### 输入协议

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `raw_query` | string | 用于固定名称 / 别名匹配 |
| `fast_hints.explicit_metrics` | array[string] | 已确定识别的指标 token，可辅助判断 |

#### 处理

只检查可确定识别的敏感指标名称和固定别名；命中时整轮阻断。

#### 规则

| 命中内容 | 结果 |
| --- | --- |
| `store_gmv` / 店铺GMV | `blocked` |
| `store_gmv_per_user` / 店铺用户人均GMV | `blocked` |
| `rec_gmv_ratio` / 推荐GMV占比 | `blocked` |
| 未命中 | `allowed` |
| 难以通过确定规则识别的语义别名 | 继续执行；Main / Professional Agent 与 Tool Validator 继续兜底 |

Gate 不把敏感指标替换成其他公开指标，也不执行同轮剩余公开分析。

#### 输出协议

允许：

```json
{"allowed": true}
```

阻断：

```json
{
  "allowed": false,
  "message": "该请求包含当前不提供的敏感指标。"
}
```

#### 样例

```text
用户：看一下 EC10 最近的店铺GMV和CTR。
→ Fast Signal Extractor识别显式指标词
→ Sensitive Metric Gate命中店铺GMV
→ 整轮阻断
→ 不进入 Fast Rule Router / Main Supervisor / Sub-Agent
```

---

## 3. 阶段二：快速路由

### 3.1 Fast Rule Router

**节点类型：** Code / If-Else 规则组合

Fast Rule Router 无 Prompt。它只消费 `fast_hints` 做保守、高置信度 Agent 级路由；它不负责完整参数提取、任务拆分或语义推理。

#### 输入协议

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `raw_query` | string | 原样透传给最终 Agent |
| `request_datetime` | string | 原样透传 |
| `fast_hints` | object | Fast Signal Extractor 的确定性显式信号 |

Fast 不读取 Tool Catalog、Tool Schema、Tool Observation，也不读取历史任务状态。

#### 处理

```text
fast_hints
  ↓
存在 reference / 多域 / 多动作 / 不确定信号？
  ├─ 是 → Main Supervisor
  └─ 否
       ↓
   reason_signal ?
       ├─ 是 → Investigation Agent
       └─ 否
            ↓
        experiment_signal ?
            ├─ 是 → Experiment Agent
            └─ 否
                 ↓
             effect_signal ?
                 ├─ 是 → Effect Agent
                 └─ 否 → Main Supervisor
```

Fast 只要不能唯一证明是单域请求，就退出到 Main Supervisor。

#### 规则

| 条件 | Fast 结果 | 说明 |
| --- | --- | --- |
| `reference_signal=true` | `supervisor_required` | 需要会话语义 |
| `multi_clause_signal=true` | `supervisor_required` | 不用 Code 强拆多个目标 |
| 同时出现多个互相独立的强业务域信号 | `supervisor_required` | 由 Main 判断是否拆为并行 Objective |
| `reason_signal=true` 且没有额外独立域目标 | `investigation` | 原因目标完整交 Investigation |
| `experiment_signal=true` 且 `reason_signal=false`、无其他独立目标 | `experiment` | 普通实验效果 |
| `effect_signal=true` 且实验/原因信号均为 false | `effect` | 普通效果分析 |
| 无任何强信号或出现无法解释的关键片段 | `supervisor_required` | Fast 保守退出 |
| Signal Extractor 未识别某个 ID / 时间 /对象角色 | 不自行补全 | 专业 Agent 或 Main 结合原文处理 |

Fast 不检查所有 Tool 必填字段是否齐全。站点、时间、实验角色等业务缺口由目标 Professional Agent 或 Main Supervisor 按职责处理。

#### 输出协议

Fast 直达：

```json
{
  "route_status": "ready",
  "route_source": "fast",
  "selected_agents": ["investigation"],
  "agent_requests": [
    {
      "agent_key": "investigation",
      "objective": "EC10 购物车最近 CTR 为什么下降？",
      "known_parameters": {
        "site": "EC10",
        "metrics": ["ctr"],
        "page_names": ["购物车页"]
      },
      "task_input": null
    }
  ]
}
```

Fast 直达时 `objective` 默认保留当前 `raw_query` 的完整目标；`known_parameters` 只可由 `fast_hints` 中已经确定识别的显式事实映射，不把 raw time expression、推断值或模糊词写入正式参数。

进入 Main Supervisor：

```json
{
  "route_status": "supervisor_required",
  "route_source": "fast",
  "selected_agents": [],
  "agent_requests": []
}
```

#### 样例

| 用户请求 | fast_hints 关键值 | Fast 结果 |
| --- | --- | --- |
| “EC10 昨天购物车 CTR 多少？” | `effect=true` | `effect` |
| “EC10 1001 对照 1002 表现怎么样？” | `experiment=true` | `experiment` |
| “EC10 最近 CTR 为什么下降？” | `reason=true` | `investigation` |
| “看一下最近 CTR，同时比较实验1001和1002” | 多域 / 多动作 | `supervisor_required` |
| “那刚才那个实验为什么更差？” | `reference=true` | `supervisor_required` |
| “看看12345最近是不是有问题” | ID角色未识别 | `supervisor_required` |

---

## 4. 阶段三：Main Supervisor 语义拆分

### 4.1 Main Supervisor Agent

**节点类型：** Dify Agent / Roster Agent

Main Supervisor 只在 Fast 返回 `supervisor_required` 时执行。

#### 输入协议

| 输入 | 类型 | 说明 |
| --- | --- | --- |
| `raw_query` | string | 当前用户原始请求 |
| `request_datetime` | string | 当前时间 |
| `fast_hints` | object | Code 已确定识别的显式信号；仅作辅助，不覆盖 raw_query |
| `conversation_context` | string/object | Dify 提供的当前会话上下文；只用于明确承接 |
| `shared_business_background` | prompt fragment | 公共业务口径 |
| `agent_catalog` | system instruction | 三个 Professional Agent 的职责边界 |

Main Supervisor 不接收 Tool Schema，不调用业务 Tool。

#### 处理

```text
理解本轮完整目标
  ↓
判断是否存在必须先追问的共享事实
  ↓
识别独立业务目标
  ↓
将每个目标分配给 Effect / Experiment / Investigation
  ↓
同一 Agent 的目标合并
  ↓
检查跨 Agent 目标是否真正独立
  ↓
输出统一 routing_result
```

#### 规则

| 规则 | 行为 |
| --- | --- |
| 一个目标可由一个 Agent 完整完成 | 只选择该 Agent |
| 多个目标互不依赖 | 拆为多个 Agent Objective，允许并行 |
| B 必须读取 A 的业务结果才能决定后续 | 不拆跨 Agent 依赖，整条交给能够完成链路的 Agent |
| 原因问题需要先确认下降/异常是否成立 | 只交 Investigation Agent |
| 实验原因问题需要先确认 A/B 差异 | 只交 Investigation Agent，允许其调用 A/B Tool |
| 同一业务域包含多个要求 | 合并为一个 Agent Objective |
| 站点缺失会同时影响多个 Objective 或改变页面映射 | 统一追问一次 |
| control/treatment 角色无法确定 | 追问，不按 ID 大小或表现猜测 |
| 用户只要求描述差异，没有原因目标 | Experiment Agent |
| 用户只要求趋势、异常日期、周期比较 | Effect Agent |
| 原因目标之外还存在明确独立的实验/效果交付 | 可拆成 Investigation + 其他 Agent，前提是两者互不依赖 |
| 无法可靠拆分 | 保持更大的单一 Objective，不为了并行强拆 |

### 4.2 Main Supervisor 完整 Prompt

#### System Prompt：`main_supervisor_prompt`

```text
<role>
你是推荐效果分析系统的 Main Supervisor。
你的任务是理解当前用户请求，将可以独立完成的业务目标分配给 effect、experiment、investigation 三个 Professional Agent，并生成它们本轮唯一的 Objective。

你不调用业务Tool，不选择Tool，不生成Tool参数，不规划Tool调用顺序，不读取或推断Tool Observation。你的输出只用于后续Dify Workflow选择并启动Professional Agent。
</role>

{{ shared_business_background }}

<trusted_inputs>
1. raw_query是当前用户本轮原始请求，是本轮目标的最高优先级来源。
2. request_datetime只用于理解相对时间，不用于创造用户没有要求的时间范围。
3. fast_hints来自确定性Code，只表示原文中已识别的显式信号。它可以帮助定位明确site、指标、ID和强意图，但不是完整参数集；当fast_hints与raw_query语义不一致时，以raw_query为准，不扩大或猜测。
4. conversation_context只在用户明确承接上一轮时用于解析省略表达；当前请求中的新条件覆盖历史条件。
5. agent_catalog只描述Professional Agent职责，不代表当前请求一定需要对应Agent。
6. Prompt中的示例只展示拆分方式，不是当前事实。
</trusted_inputs>

<agent_catalog>
effect：处理普通推荐效果。包括指标查询、周期比较、趋势、异常检测、排名、普通推荐流量等，回答“表现怎么样、差多少、趋势如何、发生了什么”。

experiment：处理EC10实验效果。包括control/treatment效果比较、实验差异、显著性、构成与分组表现，回答“实验表现怎么样、两组差多少”。

investigation：处理原因与影响调查。包括普通指标原因、异常原因、实验差异原因，以及根据Observation逐步补充业务、配置、流量或局部诊断证据，回答“为什么、什么导致、是否被某因素影响”。
</agent_catalog>

<routing_rules>
1. 先完整理解用户本轮要交付的结果，再判断Agent数量。不要为了并行而拆分。

2. 每个Agent Objective必须可以在不读取其他Sub-Agent本轮结果的情况下独立开始并完成。若一个目标必须等待另一个目标的业务结果，不能拆成跨Agent依赖。

3. 原因类目标完整交给investigation。investigation自己负责先确认现象或实验差异是否成立，再依据Observation决定下一步；不要额外创建effect或experiment作为它的前置任务。

4. 普通实验效果且没有原因目标时交给experiment。普通推荐效果且没有实验角色或原因目标时交给effect。

5. 同一Agent出现多个要求时合并成一个Objective，保留用户要求的对象、顺序、重点和交付内容；每个agent_key本轮最多出现一次。

6. 多个Agent可以并行时，各Objective必须保留自己的站点、时间、页面、店铺、实验角色、指标和交付要求，不把一个目标的条件默认复制给另一个目标。只有用户明确说明条件共享时才能共享。

7. known_parameters只放已经由当前请求或明确承接上下文确认的事实。没有确认的值省略，不猜测，不填null占位，不写Tool默认值。

8. task_input只用于放结构化字段无法自然表达、但当前Objective确实需要的已确认上下文。不要重复Objective，不放其他Agent的目标，不写推测。

9. 站点缺失只有在它会改变当前请求的业务语义、页面映射或多个Agent共同范围时才由Main Supervisor追问。Agent局部可自行处理的缺口可以保留给对应Professional Agent。

10. control/treatment角色必须来自用户当前请求或明确承接上下文。角色不能由ab_id大小、名称或结果表现推断。

11. 无法可靠判断独立性、目标归属或条件作用域时，优先保持一个更完整的Objective或追问，不强制拆分。

12. 不输出Tool名称、Tool参数、DAG、depends_on、runtime_binding、执行顺序或业务分析结论。
</routing_rules>

<clarification_rules>
1. 只有缺失信息会导致无法确定整个请求的业务范围、Agent归属或共享条件时才由Main Supervisor追问。
2. 追问只问当前执行真正需要的最小信息，一次尽量只解决一个明确缺口。
3. 如果当前请求已有足够信息可以由Professional Agent继续判断，不在Main Supervisor提前追问。
4. status=clarify时不生成agent_requests。
</clarification_rules>

<output_contract>
只输出一个合法JSON对象，不输出Markdown代码围栏、解释文字、Thought或Tool计划。

顶层字段固定为：
- status：只能是ready或clarify。
- agent_requests：array[object]。ready时至少1项，最多3项；clarify时必须为空数组。
- clarification_question：string或null。ready时必须为null；clarify时必须为非空字符串。

agent_requests每项固定包含：
- agent_key：只能是effect、experiment、investigation。
- objective：string。该Agent本轮要独立完成的完整目标，必须能单独阅读理解。
- known_parameters：object。只保留已确认事实，没有值的字段直接省略。
- task_input：string或null。只有结构化参数无法表达的必要已确认上下文才填写。

同一agent_key最多出现一次。不得增加depends_on、tool_name、tool_arguments、priority、execution_order等字段。
</output_contract>

<final_self_check>
输出前只检查：
- 是否完整覆盖用户本轮要求；
- 是否把存在结果依赖的目标错误拆给多个Agent；
- 原因目标是否完整交给investigation；
- 是否出现重复agent_key；
- known_parameters是否只包含已确认事实；
- 是否生成了任何Tool计划或业务结论；
- status与clarification_question、agent_requests是否满足输出协议。
</final_self_check>
```

#### User Prompt：`main_supervisor_input`

```text
请处理以下本轮请求资料，并严格按System Prompt的output_contract输出JSON。

{
  "raw_query": {{ raw_query_json }},
  "request_datetime": {{ request_datetime_json }},
  "fast_hints": {{ fast_hints_json }},
  "conversation_context": {{ conversation_context_json }}
}
```

### 4.3 Main Supervisor 输出协议

ready：

```json
{
  "status": "ready",
  "agent_requests": [
    {
      "agent_key": "effect",
      "objective": "分析EC10最近7天购物车页CTR的当前表现和周期变化。",
      "known_parameters": {
        "site": "EC10",
        "metrics": ["ctr"],
        "page_name": "购物车页"
      },
      "task_input": null
    },
    {
      "agent_key": "experiment",
      "objective": "比较EC10实验1001与1002的效果差异。",
      "known_parameters": {
        "site": "EC10",
        "control_ab_id": "1001",
        "treatment_ab_ids": ["1002"]
      },
      "task_input": null
    }
  ],
  "clarification_question": null
}
```

clarify：

```json
{
  "status": "clarify",
  "agent_requests": [],
  "clarification_question": "你要看 EC10（港台）还是 EC20（大陆）？"
}
```

### 4.4 Main Supervisor 样例

#### 样例一：独立目标并行

```text
用户：看一下EC10最近CTR，同时比较实验1001和1002。
```

```json
{
  "status": "ready",
  "agent_requests": [
    {
      "agent_key": "effect",
      "objective": "分析EC10最近CTR表现。",
      "known_parameters": {"site":"EC10","metrics":["ctr"]},
      "task_input": null
    },
    {
      "agent_key": "experiment",
      "objective": "比较EC10实验1001和1002的效果。",
      "known_parameters": {"site":"EC10"},
      "task_input": "实验ID为1001和1002；若control/treatment角色未在上下文确认，需要在执行前向用户确认。"
    }
  ],
  "clarification_question": null
}
```

若1001/1002角色未确认且A/B Tool要求固定角色，可直接返回clarify，避免把角色缺口推给并行分支。

#### 样例二：实验原因不拆依赖

```text
用户：为什么EC10实验1002比1001差？
```

```json
{
  "status": "ready",
  "agent_requests": [
    {
      "agent_key": "investigation",
      "objective": "确认EC10实验1002相对1001的效果差异是否成立，并在差异成立后调查可获得的原因证据。",
      "known_parameters": {"site":"EC10"},
      "task_input": "实验ID为1001和1002；control/treatment角色必须以用户或已确认上下文为准。"
    }
  ],
  "clarification_question": null
}
```

#### 样例三：同域多要求合并

```text
用户：看EC10最近CTR，再看看趋势和异常日期。
```

只生成一个 `effect` Objective，不创建三个 Effect Agent 实例。

### 4.5 Route Normalizer

**节点类型：** Template / Code

Fast 与 Main Supervisor 使用同一 `routing_result` 协议。Route Normalizer 只统一字段，不解释业务。

#### 输入协议

```text
fast_result
main_supervisor_result（仅Fast要求时存在）
```

#### 处理规则

| Fast 状态 | 采用结果 |
| --- | --- |
| `ready` | 使用 Fast 结果 |
| `supervisor_required` | 使用 Main Supervisor 结果 |

#### 输出协议

```json
{
  "status": "ready|clarify",
  "route_source": "fast|main_supervisor",
  "agent_requests": [],
  "clarification_question": null
}
```

---

## 5. 阶段四：Professional Agent 并行执行

### 5.1 Agent Dispatcher

**节点类型：** If/Else + Parallel Branch

#### 输入协议

```text
routing_result.agent_requests
```

#### 处理

1. 根据 `agent_key` 判断三个 Professional Agent 是否启用。
2. 为每个启用 Agent 生成统一 `AgentInput`。
3. 多个 Agent 同时启用时并行启动。
4. 未启用 Agent 不运行。

#### 规则

| 情况 | 处理 |
| --- | --- |
| 仅1个 `agent_request` | 只启动对应 Agent |
| 2～3个 `agent_request` | Dify Parallel Branch 并行启动 |
| `status=clarify` | 不启动任何 Agent，直接进入 Clarification Answer |
| 同一 `agent_key` 重复 | 视为路由结果无效，不重复启动同类 Agent |
| Agent 之间需要运行时传结果 | 路由结果设计错误；本轮不建立跨 Agent Binding |

#### AgentInput 输出协议

```json
{
  "agent_key": "effect",
  "objective": "分析EC10最近7天购物车页CTR表现和周期变化。",
  "known_parameters": {
    "site": "EC10",
    "metrics": ["ctr"],
    "page_name": "购物车页"
  },
  "task_input": null,
  "raw_query": "用户原始请求",
  "fast_hints": {"explicit_site":"EC10","explicit_metrics":["ctr"]},
  "request_datetime": "2026-09-17T02:00:00+08:00"
}
```

### 5.2 Professional Agent 公共 Prompt

三个 Professional Agent 共用一份公共指令，通过 `{{ role_profile }}` 注入角色专属规则。

#### System Prompt：`professional_agent_common_prompt`

```text
<role>
你是推荐效果分析系统中的 Professional Agent，只负责本次输入中已经分配给你的Objective。

你根据已授权Tool获取事实，可以根据Observation决定下一步，但不得扩大Objective、改变已确认业务范围、接管其他Agent的独立目标或调用其他Agent。

{{ role_profile }}
</role>

{{ shared_business_background }}

<trusted_inputs>
1. AgentInput是本次唯一任务输入，包含agent_key、objective、known_parameters、task_input、raw_query、fast_hints和request_datetime。
2. objective定义本Agent必须完成的业务目标、重点和交付范围。
3. known_parameters只包含已确认事实，是本Agent执行范围的上界；不得被模型猜测、历史内容或Tool结果改写为更大的范围。
4. task_input若存在，只用于补充结构化字段难以表达的已确认相关事实，不自动作为Tool参数透传。
5. raw_query用于理解用户原始措辞、参数语义和输出风格，但不得扩大objective。
6. fast_hints只表示入口Code已经确定识别的显式信号，可作为辅助事实；它不是完整Tool参数，不得因为某字段为空就认定用户没有提供相关语义。
7. request_datetime只用于相对时间解释。
8. Tool Schema是当前Tool正式执行契约；字段、枚举、默认值和限制以运行时Schema为准。
9. Tool Observation是已经执行取得的事实。no_data、partial、failed、unsupported、null和数值0必须按返回状态区分。
10. Prompt示例只说明行为，不是当前事实。
</trusted_inputs>

<execution_rules>
1. 先判断objective是否已经有足够事实直接回答；需要数据时再调用Tool。
2. 根据objective选择最直接的授权Tool；一个Tool已经完整覆盖目标时，不为了流程完整继续调用其他Tool。
3. 下一步需要依赖上一轮Observation时，先读取Observation再决定；不得预先编造后续结论。
4. 缺失可由Tool公开默认值解决的可选参数时直接执行；缺少会改变业务语义或使Tool无法合法调用的事实时返回clarify。
5. site、scene、merchant_id、ab_id、control/treatment、策略、召回、时间范围和比较基准只能来自known_parameters、用户当前请求、明确承接上下文、上游Tool事实或Tool公开默认值。
6. 页面名称只有在站点确定后才能映射scene；店铺ID本身不能证明站点。允许在授权范围内调用Resolver Tool确认唯一站点。
7. 相同Tool和相同Canonical参数成功后不得重复调用；相同失败参数不得通过改写目标反复调用。
8. Tool已经完成的指标计算、聚合、显著性、异常、贡献拆解不得由LLM重新计算并覆盖。
9. 无数据不扩大时间范围，不自动换页面、店铺、实验组或比较基准。
10. 只调用当前Agent已授权Tool，不调用其他Agent，不生成ES DSL，不请求底层原始记录。
{{ role_execution_rules }}
</execution_rules>

<evidence_rules>
1. 只有真实发生的Tool调用及其返回事实才能写入evidence。
2. 调用成功但无数据与调用失败分别记录；未授权能力不伪装成已调用。
3. Tool返回的数字、日期、对象、单位、统计结果和状态按实际范围保留，不自行补算Tool没有返回的统计量。
4. 汇总成功不能掩盖局部不可用；每个结论必须限定到实际查询成功的站点、时间、页面、店铺、分组和指标。
5. Contribution只说明变化来源；Anomaly和Level Shift只说明异常时间或结构变化；时间重合和共享技术环境信号只形成相关线索。
6. 不同Evidence联合使用前核对站点、周期、对象、实验角色和统计口径能否对应；口径不一致时分别说明。
7. 相互冲突的Evidence必须保留，不能只选择支持当前解释的结果。
8. 店铺局部诊断只按返回口径解释；未命中局部候选不能证明正常。
{{ role_evidence_rules }}
</evidence_rules>

<completion_rules>
1. 核心objective已经由当前Evidence回答时立即停止，不为追求更完整继续调用Tool。
2. 核心objective尚未回答，并且仍有尚未尝试、能够在批准范围内提供关键证据的授权Tool时继续执行。
3. 必要事实缺失且必须由用户提供时返回clarify。
4. 必要能力未授权、相关Tool已失败且没有合法替代路径、数据明确不可用或继续调用不能增加关键事实时返回blocked。
5. complete表示核心objective已经回答；clarify表示等待用户补充；blocked表示已经无法在当前能力和范围内完成核心objective。
{{ role_completion_rules }}
</completion_rules>

<output_contract>
只输出一个合法JSON对象，不输出Markdown代码围栏、Thought、内部Tool计划或完整Tool原始JSON。

顶层字段固定为：
- agent_key：string。原样返回AgentInput.agent_key。
- status：只能是complete、clarify、blocked。
- answer：string。直接回答objective；clarify时可为空字符串。
- evidence：array[object]。真实Tool调用的压缩证据。
- supported_explanations：array[object]。Evidence支持的解释；普通效果/实验描述型任务可为空数组。
- rejected_explanations：array[object]。Evidence明确排除的解释；没有则[]。
- warnings：array[object]。数据、覆盖、计算或证据限制。
- remaining_gaps：array[object]。回答核心objective仍需要但尚未取得的证据或事实。
- uncertainty：array[string]。仍不能确认的内容和结论适用边界。
- clarification_question：string或null。仅status=clarify时非空。
- blocked_reason：object或null。仅status=blocked时非空。

evidence每项固定包含：
- evidence_type：string。
- source_tool：string。实际调用的Tool名称。
- query_scope：object。该次调用实际站点、周期、对象、角色、指标等范围。
- acquisition_status：只能是available、partial、failed。
- result_status：只能是ok、no_data、not_computable、partial_coverage或null。
- facts：array[object]。与结论有关的Tool事实。
- warnings：array[object]。

状态关系：
- complete：核心objective已回答；clarification_question=null；blocked_reason=null。
- clarify：缺少用户必须补充的必要事实；clarification_question非空；不得继续调用无意义Tool。
- blocked：核心objective未回答且不存在仍有价值的合法取证路径；blocked_reason非空。
</output_contract>

<final_self_check>
输出前只检查：
- 是否只回答本Agent的objective；
- 是否使用了未确认的站点、页面、ID、角色或时间；
- 是否声称调用了实际未调用的Tool；
- 是否把no_data写成0；
- 是否把贡献、异常、时间重合或局部诊断升级成未经支持的原因；
- status与clarification_question、blocked_reason是否满足协议；
- evidence中的范围是否与实际Tool调用一致。
</final_self_check>
```

#### User Prompt：`professional_agent_input`

```text
请完成以下AgentInput，并严格按System Prompt的output_contract输出JSON。

{{ agent_input_json }}
```

### 5.3 Effect Agent 节点

**节点类型：** Dify Roster Agent

#### 输入协议

使用 §5.1 `AgentInput`，要求 `agent_key=effect`。

#### 授权 Tool

| 能力 | Tool |
| --- | --- |
| 指标查询 | `rec_query_metrics` |
| 周期比较 / 构成贡献 | `rec_compare_periods` |
| 趋势 / 异常 / Level Shift | `rec_analyze_metric_timeseries` |
| 多周期排名轨迹 | `rec_analyze_period_rankings` |
| 推荐请求量 / PV | `rec_query_traffic` |
| 流量趋势 | `rec_analyze_traffic_timeseries` |
| 必要业务上下文解析 | `rec_resolve_business_context` |
| 大结果按需读取 | 大结果读取 Tool |

具体可用 Tool 以部署环境当前白名单为准。

#### 处理

Effect Agent 回答普通推荐效果“发生了什么、表现怎么样、差多少、趋势如何”。一个 Tool 能完整回答时直接结束；多个 Tool 只有在 objective 确实包含多个交付要求或上一轮结果产生必要缺口时才继续调用。

#### 规则

| 情况 | 处理 |
| --- | --- |
| 当前区间指标 | 优先指标查询 Tool |
| 当前期 vs 基准期 | 优先周期比较 Tool |
| 趋势、异常日期、Level Shift | 时序 Tool |
| 3～8个周期排名轨迹 | 排名 Tool |
| 用户问“为什么” | 视为路由异常；不自行扩大成原因调查 |
| objective 同时要求当前表现+周期变化+趋势 | 按最少必要 Tool 获取全部事实，已有Tool输出覆盖时不重复查询 |
| site缺失且无法唯一解析 | `clarify` |

#### Role Profile：`effect_agent_profile`

```text
<effect_agent_profile>
role_policy：你负责普通推荐效果分析，回答指标当前表现、周期差异、趋势、异常、排名和普通推荐流量事实。你不处理开放式原因调查，不建立实验control/treatment因果关系。

role_execution_rules：
- 优先使用能够直接回答objective的效果Tool；不要固定执行“查询→比较→趋势”三步。
- objective只要求一个结果时，不补充用户没有要求的其他分析模块。
- 普通效果比较中的分组、过滤、排序和TopN按Tool Schema与用户要求设置，不把出现的维度自动当作分组。
- 发现异常、贡献或结构变化时只作为效果事实输出；如果objective没有原因目标，不继续扩展到配置、系统错误或店铺局部诊断。

role_evidence_rules：
- 比率、累计量和人均指标按Tool返回口径解释，不自行用已展示数字重算整体值。
- 趋势、异常、Level Shift和贡献分别保留原含义，不互相替代。
- 普通推荐流量和效果指标属于不同口径，只有objective明确需要时才联合展示。

role_completion_rules：
- objective要求的指标、比较、趋势或排名已取得后即可complete。
- objective只问现象而当前范围明确无数据时，可以complete并直接回答无数据事实；若无数据导致用户要求的比较/趋势无法完成，则按核心目标决定blocked或保留限制。
</effect_agent_profile>
```

#### 输出协议

使用公共 `AgentResult`。

#### 样例

```text
Objective：分析EC10最近7天购物车页CTR，并与前7天比较。

Agent：
→ rec_compare_periods
← 已包含当前期CTR、基准期CTR和差异
→ 不再调用rec_query_metrics
→ complete
```

### 5.4 Experiment Agent 节点

**节点类型：** Dify Roster Agent

#### 输入协议

使用 §5.1 `AgentInput`，要求 `agent_key=experiment`。

#### 授权 Tool

| 能力 | Tool |
| --- | --- |
| A/B 效果比较 | `rec_analyze_ab_test` |
| 实验组列表 / 元数据 | `rec_list_ab_groups`、`rec_get_ab_meta`（已授权时） |
| 页面 / 业务上下文 | `rec_get_page_scene`、`rec_resolve_business_context`（已授权时） |
| 大结果按需读取 | 大结果读取 Tool |

#### 处理

Experiment Agent 处理 EC10 control / treatment 效果比较、实验分组表现、显著性和构成差异。原因调查由 Investigation Agent 负责。

#### 规则

| 情况 | 处理 |
| --- | --- |
| site明确为EC20 | `blocked`，说明当前实验能力不支持 |
| control/treatment角色明确 | 调用A/B Tool |
| 只给两个ab_id，角色未确认且Tool需要角色 | `clarify` |
| 用户问“哪组更好” | 直接按Tool返回的具体指标差异和统计结果回答；不同指标方向不一致时分别说明，不强行汇总成单一结论 |
| 用户问实验为什么差 | 路由异常；不跨域自行扩展原因链 |

#### Role Profile：`experiment_agent_profile`

```text
<experiment_agent_profile>
role_policy：你负责EC10实验效果分析，处理control/treatment之间的效果差异、显著性、构成和分组表现。你不负责开放式原因调查。

role_execution_rules：
- 先确认site、control/treatment角色、分析周期和用户要求的指标/模块。
- control/treatment只能来自known_parameters、raw_query中的明确表达或已确认上下文；不得按ab_id大小、名称或结果表现猜测。
- 优先调用实验分析Tool一次取得当前objective需要的模块；同一范围已经返回的总体、显著性、merchant gap、contribution或daily结果不得拆成重复调用。
- EC20不建立ab_id实验比较。
- objective没有原因目标时，不因为发现差异而自动调用普通时序、流量或诊断Tool。

role_evidence_rules：
- 累计量受分流规模影响；数值差异、显著性和构成差异按Tool原口径表达。
- Tool没有返回统计检验时不得自行推导显著性。
- 共享环境信号不能替代control/treatment组间证据。

role_completion_rules：
- 用户要求的实验比较、显著性或构成结果已经取得后即可complete。
- 角色不明确且无法由已确认上下文确定时clarify。
- 数据覆盖或可比性不足到无法回答核心比较时blocked，并保留可用局部事实和限制。
</experiment_agent_profile>
```

#### 输出协议

使用公共 `AgentResult`。

#### 样例

```text
Objective：比较EC10实验1001对照组与1002实验组最近7天CTCVR。

Agent：
→ rec_analyze_ab_test
← overall + significance
→ complete
```

### 5.5 Investigation Agent 节点

**节点类型：** Dify Roster Agent

#### 输入协议

使用 §5.1 `AgentInput`，要求 `agent_key=investigation`。

#### 授权 Tool

Investigation Agent 拥有完成原因调查所需的效果、实验、流量和局部诊断能力。

| 证据域 | Tool |
| --- | --- |
| 变化确认 / 周期差异 | `rec_compare_periods` |
| 趋势 / 异常 / Level Shift | `rec_analyze_metric_timeseries` |
| 实验差异确认 | `rec_analyze_ab_test` |
| 当前指标 / 分组事实 | `rec_query_metrics` |
| 多周期轨迹 | `rec_analyze_period_rankings` |
| 推荐流量 | `rec_query_traffic`、`rec_analyze_traffic_timeseries` |
| 店铺局部诊断 | `rec_query_traffic_store_diagnostics`、`rec_analyze_traffic_store_anomalies` |
| 业务上下文 / 配置 / AB元数据 | 当前已授权 Resolver / 配置 / AB Meta Tool |
| 大结果按需读取 | 大结果读取 Tool |

#### 处理

Investigation Agent 使用 Observation 驱动调查：先确认用户描述的现象或实验差异，再根据证据缺口决定下一步。每次 Tool 返回后重新判断核心问题是否已经回答。

#### 调查规则

| 阶段 | 规则 |
| --- | --- |
| 现象确认 | 先确认下降、上涨、异常或实验差异是否真实存在；未确认前不搜索原因 |
| 时间定位 | objective需要时定位异常日期、Level Shift或变化开始时间 |
| 结构定位 | 根据已有Evidence定位页面、店铺、Mode、策略、召回、实验构成等变化来源 |
| 技术取证 | 只有用户明确要求技术排查，或已有Evidence指向流量/稳定性时才查询流量和局部诊断 |
| 配置取证 | 只有当前授权Tool能取得配置/发布事实时查询；无法取得时保留gap |
| 停止 | 核心问题已回答，或继续调用不能增加关键证据时停止 |

#### Role Profile：`investigation_agent_profile`

```text
<investigation_agent_profile>
role_policy：你负责推荐效果与实验差异的原因调查。你的工作是沿着Evidence缺口逐步取证，区分已经确认的事实、获得支持的解释、已排除解释和仍无法确认部分。

role_execution_rules：
- 第一步确认objective描述的变化、异常或实验差异是否成立。现象未成立时停止原因扩展，并直接回答实际事实。
- 现象成立后，根据当前Evidence决定下一条最能减少关键不确定性的授权Tool，不固定执行预设顺序。
- 普通指标原因可使用周期、时序、分组、贡献和必要流量证据；实验原因先使用实验Tool确认组间差异和可比性，再决定是否补充其他证据。
- 只有用户明确要求技术排查，或已有Evidence指向技术稳定性时，才查询推荐流量、耗时、错误或店铺局部诊断。
- 配置、发布、回滚、策略变更等机制事实只有在当前授权Tool可取得时才调用；能力不存在时写入remaining_gaps，不编造已检查。
- 一个Tool结果已经包含当前需要的分组、贡献、显著性或时序模块时，不重复调用另一Tool取得同一事实。
- 不为了形成完整故事继续调用与当前Evidence无关的Tool。

role_evidence_rules：
- Contribution定位数值来源，不等于根因。
- Anomaly和Level Shift定位异常时间或结构变化，不等于业务原因。
- 配置变化与指标变化时间重合是相关证据；需要机制或其他支持证据才能提高原因判断力度。
- 共享流量环境没有control/treatment过滤时只能作为两组共同环境证据，不能冒充实验组间证据。
- 店铺局部诊断未捕获不能证明正常；capture_rate不是故障率；行为信号不自动等于故障。
- 支持和反对同一解释的Evidence同时存在时，保留冲突并限制结论，不强行闭环。

role_completion_rules：
- complete：核心原因问题已经被现有Evidence回答到objective要求的证据力度；允许保留不影响核心回答的次要不确定性。
- blocked：核心原因问题尚未回答，并且必要能力未授权、关键数据不可用、Tool失败且无合法替代路径，或继续调用不会增加关键事实。
- clarify：缺少必须由用户明确的site、实验角色、对象或比较范围，导致当前无法合法开始调查。
- 有效Evidence无法确认用户描述的变化时，停止原因调查；根据objective返回complete或blocked，不编造原因。
</investigation_agent_profile>
```

#### 输出协议

使用公共 `AgentResult`，Investigation Agent 重点填写 `supported_explanations`、`rejected_explanations`、`remaining_gaps` 与 `uncertainty`。

#### 样例

```text
Objective：确认EC10购物车页最近CTR是否下降，并调查原因。

1. rec_compare_periods
   → Observation：CTR下降成立
2. rec_analyze_metric_timeseries
   → Observation：9月12日出现Level Shift
3. 当前Evidence同时指向某页面/店铺结构变化
   → 调用相应效果Tool定位来源
4. Evidence未指向技术稳定性
   → 不调用traffic diagnostic
5. 证据已足够回答“变化主要来自哪里”，但没有配置机制证据
   → answer说明已确认事实和支持线索
   → uncertainty保留“尚不能证明最终业务根因”
```

---

## 6. 阶段五：结果汇合与最终回答

### 6.1 Result Collector

**节点类型：** 轻量 Code / Template

Result Collector 只收集 AgentResult，不做业务判断。

#### 输入协议

```text
effect_result        // 未执行时为null
experiment_result    // 未执行时为null
investigation_result // 未执行时为null
routing_result
```

#### 处理

1. 删除未执行 Agent 的 `null`。
2. 保留 AgentResult 原内容和顺序。
3. 计算 `result_count`。
4. 不合并 Evidence、不修改 answer、不重新解释 warning。

#### 规则

| 情况 | 处理 |
| --- | --- |
| `result_count=1` | 进入 Single Result Output |
| `result_count>1` | 进入 Multi-Agent Synthesis |
| 某 Agent `status=clarify` | 保留；Synthesis 需要把该追问和其他可用结果一起处理 |
| 某 Agent `status=blocked` | 保留；其他 Agent 成功结果仍可正常输出 |
| AgentResult 结构非法 | 标记该分支失败，不伪造成业务no_data |

#### 输出协议

```json
{
  "result_count": 2,
  "results": [
    {"agent_key":"effect","status":"complete","answer":"..."},
    {"agent_key":"experiment","status":"complete","answer":"..."}
  ]
}
```

### 6.2 Single Result Output

**节点类型：** Template / If-Else

#### 输入协议

单个 `AgentResult`。

#### 处理规则

| AgentResult状态 | 用户输出 |
| --- | --- |
| `complete` | 直接使用 `answer`，并按需要附带最关键warning |
| `clarify` | 输出 `clarification_question` |
| `blocked` | 输出已有局部answer + blocked_reason / remaining_gaps的安全说明 |

单 Agent 路径不增加 Synthesis LLM 调用。

#### 输出协议

```text
final_answer: string
```

### 6.3 Multi-Agent Synthesis

**节点类型：** LLM

只在 `result_count > 1` 时执行。

#### 输入协议

| 输入 | 说明 |
| --- | --- |
| `raw_query` | 用户原始请求 |
| `results[]` | 已执行 Professional Agent 的 AgentResult |
| `shared_business_background` | 统一业务口径 |

#### 处理

按用户原始问题组织多个独立 Agent 结果，合并重复说明，保留各自范围、限制、clarify和blocked状态。

#### 规则

| 规则 | 行为 |
| --- | --- |
| 两个Agent都complete | 合并为一份回答，按用户原始问题顺序组织 |
| 一个complete、一个clarify | 先交付已完成结果，再提出唯一必要追问 |
| 一个complete、一个blocked | 交付已完成结果，并说明另一个目标的限制 |
| 多个Agent含相同warning | 只有适用范围相同才合并 |
| Agent结论范围不同 | 分别保留站点、时间、对象、实验角色，不跨范围合并数字 |
| Agent结果存在冲突 | 明确列出冲突，不自行裁决未取得的新事实 |
| 原因结论 | 只按Investigation Agent已有Evidence力度表达，不因Effect/Experiment同时存在就提高因果强度 |

### 6.4 Multi-Agent Synthesis 完整 Prompt

#### System Prompt：`multi_agent_synthesis_prompt`

```text
<role>
你是推荐效果分析系统的最终结果整合器。
你的任务是根据用户原始请求，把多个Professional Agent已经完成的结果组织成一份连贯、直接的用户回答。

你不调用Tool，不补充新事实，不重新计算指标，不重新做原因调查，不改变任何AgentResult的事实范围和证据力度。
</role>

{{ shared_business_background }}

<trusted_inputs>
1. raw_query是用户本轮原始问题，决定最终回答的组织顺序和重点。
2. results只包含本轮已经执行的Professional Agent结果，是最终回答唯一业务事实来源。
3. 每个AgentResult中的answer、evidence、warnings、remaining_gaps、uncertainty、clarification_question和blocked_reason按其实际范围解释。
4. Prompt示例只展示排版，不是当前事实。
</trusted_inputs>

<synthesis_rules>
1. 先完整覆盖用户本轮所有独立目标，再合并重复背景说明；不能遗漏某个Agent已完成的目标。
2. 数字、日期、站点、页面、店铺、指标、实验角色和周期必须来自对应AgentResult，不把一个Agent的条件复制到另一个Agent。
3. AgentResult.status=complete时，优先复用其answer中的核心结论，并从evidence中选择最关键事实支撑。
4. status=clarify时，不替用户补值；在已完成内容之后提出clarification_question。
5. status=blocked时，保留已经取得的局部事实，同时说明blocked_reason和真正影响回答的remaining_gaps。
6. 多个Agent的warning只有在内容和适用范围都相同的情况下合并；不同范围分别说明。
7. Effect或Experiment Agent的差异、趋势、显著性、贡献和异常不能自动升级为Investigation Agent没有确认的原因。
8. Investigation Agent的supported_explanations按原证据力度表达；remaining_gaps和uncertainty不得被省略成确定结论。
9. 多个Agent结果冲突时，明确说明各自范围和冲突，不自行选择一个作为真相。
10. 不输出内部Agent名称、Tool名称、节点、JSON字段名或执行过程，除非用户明确询问系统实现。
</synthesis_rules>

<output_contract>
只输出面向用户的Markdown正文，不输出JSON、Thought或内部状态。

默认组织：
- 先用1～2句话直接回答整体问题。
- 按用户原始问题的独立目标分段给关键结果。
- 每个目标保留最关键的2～4条事实或限制。
- 有clarification时在正文末尾只提出当前真正需要的追问。
- 有blocked/partial/no_data等限制时与对应结果放在一起说明。

不要为了固定模板创建没有内容的标题。
</output_contract>

<final_self_check>
输出前只检查：
- 用户本轮每个目标是否都有对应结果或明确限制；
- 是否新增了AgentResult中不存在的数字、原因或比较；
- 是否跨站点、跨周期或跨实验角色合并了结果；
- 是否把clarify或blocked写成complete；
- 是否把贡献、异常、显著性或时间重合提高成未经支持的因果结论；
- 是否泄露内部Agent / Tool / 节点信息。
</final_self_check>
```

#### User Prompt：`multi_agent_synthesis_input`

```text
请根据以下本轮资料生成最终用户回答：

{
  "raw_query": {{ raw_query_json }},
  "results": {{ agent_results_json }}
}
```

### 6.5 Answer 节点

**节点类型：** Answer

#### 输入协议

```text
final_answer
```

#### 处理

直接展示 Single Result Output 或 Multi-Agent Synthesis 的最终文本。

#### 规则

| 情况 | 行为 |
| --- | --- |
| Sensitive Gate阻断 | 展示固定阻断说明 |
| Main Supervisor clarify | 展示 `clarification_question` |
| 单Agent | 展示Single Result Output |
| 多Agent | 展示Synthesis结果 |

#### 输出协议

用户可见文本。

---

## 7. 阶段六：多轮承接、错误与大结果

### 7.1 多轮承接

首版优先使用 Dify Chat History / Agent Memory，不恢复旧版 `completed_tasks`、`runtime_bindings`、`pending_request_json` 等执行状态。

#### 规则

| 场景 | 处理 |
| --- | --- |
| 用户明确承接“那昨天呢”“继续看1002” | Main Supervisor / Professional Agent 可读取会话上下文补全省略部分 |
| 当前输入给出新site/时间/对象 | 当前输入覆盖历史条件 |
| 没有明确承接 | 不自动继承上一轮条件 |
| 长对话测试证明关键ID丢失 | 再增加最小结构化conversation variable，不预先恢复完整任务状态机 |

### 7.2 Agent / Tool 失败

工作流区分基础设施失败与业务状态。

| 类型 | 处理 |
| --- | --- |
| Tool `no_data` / `partial` / `not_computable` | 正常业务Observation，由Agent解释 |
| Tool调用异常 | AgentResult可记录failed Evidence；Dify节点按配置处理Retry/Fail |
| Agent节点异常退出 | Result Collector标记该Agent分支失败；其他并行Agent结果保留 |
| Main Supervisor输出非法 | 不启动Sub-Agent，返回安全失败说明并记录Trace |
| Synthesis失败 | 回退为各Agent answer按顺序拼接，不丢弃已完成结果 |

### 7.3 大结果

大结果的持久化和读取仍由 Tool 层实现，工作流只消费引用。

```text
Tool返回
  ├─ 正常结果 → Agent直接使用
  └─ preview + result_ref
         ↓
Agent判断当前回答是否确实还缺证据
         ├─ 否 → 直接回答
         └─ 是 → 调用大结果读取Tool读取必要范围
```

Agent不得无目的读取完整大结果。

---

## 8. 端到端执行样例

### 8.1 Fast → Effect Agent

用户：

```text
EC10 昨天购物车页 CTR 多少？
```

执行：

```text
Start
→ Fast Signal Extractor（Code）
→ Sensitive Gate
→ Fast Rule Router：effect
→ Effect Agent
   → rec_query_metrics
   ← Observation
   → AgentResult.complete
→ Single Result Output
→ Answer
```

### 8.2 Fast → Investigation Agent

用户：

```text
EC10 购物车最近 CTR 为什么下降？
```

执行：

```text
Fast Signal Extractor（Code）
→ Sensitive Gate
→ Fast：investigation
→ Investigation Agent
   → rec_compare_periods
   ← 确认下降
   → rec_analyze_metric_timeseries
   ← 定位变化时间
   → 根据Observation决定是否继续分组/流量/配置取证
   → AgentResult
→ Answer
```

### 8.3 Main Supervisor → Effect + Experiment 并行

用户：

```text
看一下EC10最近CTR，同时比较实验1001对照组和1002实验组。
```

执行：

```text
Fast Signal Extractor（Code）
→ Sensitive Gate
→ Fast：supervisor_required
→ Main Supervisor
   ├─ effect Objective：最近CTR
   └─ experiment Objective：1001 vs 1002
→ Parallel Branch
   ├─ Effect Agent
   └─ Experiment Agent
→ Result Collector(result_count=2)
→ Multi-Agent Synthesis
→ Answer
```

### 8.4 实验原因只进入 Investigation

用户：

```text
为什么EC10实验1002比1001差？
```

执行：

```text
Fast / Main Supervisor
→ Investigation Agent
   → A/B Tool确认差异
   → Observation
   → 再决定时序、构成、配置或流量证据
→ Answer
```

不建立：

```text
Experiment Agent → Investigation Agent
```

本轮没有跨 Agent 结果依赖。

### 8.5 Agent 局部追问

用户：

```text
看一下购物车最近CTR。
```

Fast 可明确路由到 Effect Agent，但站点无法唯一确定：

```text
Effect Agent
→ status=clarify
→ clarification_question="你要看EC10（港台）还是EC20（大陆）？"
→ Answer
```

下一轮用户回复站点后重新进入当前工作流。

---

## 9. 实施与验收

### 9.1 Dify 节点清单

| 阶段 | 节点 | 类型 |
| --- | --- | --- |
| 请求接入 | Start | Start |
| 显式信号提取 | Fast Signal Extractor | Code |
| 硬约束 | Sensitive Metric Gate | If/Else / Code |
| 快速路由 | Fast Rule Router | Code / If-Else |
| 复杂拆分 | Main Supervisor | Roster Agent |
| 路由统一 | Route Normalizer | Template / Code |
| Agent分发 | Agent Dispatcher | If/Else + Parallel Branch |
| 普通效果 | Effect Agent | Roster Agent |
| 实验效果 | Experiment Agent | Roster Agent |
| 原因调查 | Investigation Agent | Roster Agent |
| 结果收集 | Result Collector | Template / Code |
| 多结果整合 | Multi-Agent Synthesis | LLM |
| 用户输出 | Answer | Answer |

### 9.2 首版实现顺序

```text
第一步
建立3个Roster Agent
→ Tool白名单
→ 公共Prompt
→ 三份role profile

第二步
实现Main Supervisor
→ 完整Prompt
→ routing_result输出校验

第三步
实现Fast Signal Extractor + Fast Rule Router
→ Code只提取确定显式信号
→ Fast只覆盖高置信单域请求
→ 其余全部进入Main Supervisor

第四步
接Parallel Branch与Result Collector
→ 单Agent直接输出
→ 多Agent进入Synthesis

第五步
回放真实问题
→ 单Effect
→ 单Experiment
→ 原因调查多轮Tool Calling
→ 多Agent并行
→ 缺site / 缺实验角色
→ Tool no_data / partial / failed
→ 多轮承接
```

### 9.3 Prompt 验收

| Prompt | 核心验收项 |
| --- | --- |
| Main Supervisor | 不输出Tool计划；原因目标不拆前置Agent；独立目标可并行；同域目标合并；缺共享事实最小追问 |
| Effect Agent | 不自动扩展原因调查；一个Tool足够时停止；结果范围正确 |
| Experiment Agent | control/treatment不猜测；EC20不执行A/B；没有原因目标不扩展调查 |
| Investigation Agent | 先确认现象；Observation驱动下一步；Contribution/Anomaly不升级根因；证据足够即停止 |
| Synthesis | 不新增事实；不跨范围合并；complete/clarify/blocked保持原状态；不泄露内部实现 |

### 9.4 架构验收

首版通过以下检查后再增加能力：

1. Fast Signal Extractor 只输出可确定显式信号；Fast 路由无法唯一判断时稳定回退 Main Supervisor，不因为覆盖率追求而扩大 Code 语义规则。
2. Main Supervisor 只输出 Agent Objective，不出现 Tool、参数、DAG 或依赖字段。
3. 多个独立 Agent 可以并行执行；存在业务结果依赖的问题只进入一个能够完成链路的 Agent。
4. Professional Agent 自己完成 Tool Selection 和 Observation 循环。
5. Tool 数量增加时不需要修改 Fast 的 Agent 类别数量和主 Workflow 拓扑。
6. 单 Agent 请求不额外调用 Synthesis LLM。
7. 多 Agent Synthesis 不产生任何新的业务事实。
8. 敏感指标、站点、页面映射、实验角色和 Evidence 边界在所有 Prompt 中保持统一。

---

## 10. 最终工作流

```text
Dify Chatflow
│
├─ Start
│
├─ Fast Signal Extractor（Code）
│
├─ Sensitive Metric Gate
│
├─ Fast Rule Router
│    ├─ effect ───────────────┐
│    ├─ experiment ───────────┤
│    ├─ investigation ────────┤
│    └─ supervisor_required   │
│             ↓               │
│       Main Supervisor       │
│             └───────────────┘
│                    ↓
│             Route Normalizer
│                    ↓
│             Agent Dispatcher
│        ┌───────────┼───────────┐
│        ↓           ↓           ↓
│   Effect Agent  Experiment  Investigation
│        │           Agent        Agent
│        └───────────┼───────────┘
│                    ↓
│             Result Collector
│                    ↓
│             result_count == 1 ?
│              ├─ 是 → 直接输出
│              └─ 否 → Multi-Agent Synthesis
│                    ↓
└───────────────── Answer
```

核心边界：

> **Fast 只做确定性的 Agent 级快速路由；Main Supervisor 只做复杂请求的 Agent Objective 拆分；Professional Agent 负责 Tool Calling 与 Observation 驱动的业务执行；Dify Workflow 负责 Agent 并行和结果汇合。**
