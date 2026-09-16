# 推荐效果分析 Agent：Dify 工作流重构方案

> 本文只设计 **Dify 工作流与 Prompt 编排**。推荐效果 Tool 的 ES 查询、指标计算、统计检验、异常检测、贡献拆解、Repository 和大结果存储继续以现有 Tool Contract 为准，不在本文重复实现。

## 1. 整体设计

### 1.1 目标

本次重构的目标不是把原方案的业务规则和 Prompt 全部删掉，而是重新划清 Dify 与代码的职责边界：

- **Dify 负责智能编排**：请求理解、结构化提取、条件分支、澄清、多轮上下文、Agent Tool Calling、结果解释与最终回答。
- **Tool 负责确定性能力**：ES / DB 查询、指标口径、周期聚合、A/B 计算、统计检验、异常检测、贡献拆解、数据质量与业务参数校验。
- **普通分析走固定工作流**：指标查询、综合效果、周期比较、趋势 / 异常、A/B 对比直接进入对应 Tool。
- **只有原因调查进入 Agent**：当下一步查什么需要依赖上一轮 Observation 时，由 Dify 原生 Agent 自主选择授权 Tool。
- 不再自研 Planner、Reviewer、Compiler、Scheduler、Task Runtime、Binding Runtime 或另一套 Agent 框架。

### 1.2 设计原则

| 原则 | 规则 |
| --- | --- |
| Dify 原生优先 | Dify 已有 LLM、Parameter Extractor、Question Classifier、If/Else、Tool、Agent、Variable Assigner、Variable Aggregator、Conversation Variable、Answer 等节点时，优先直接使用。 |
| 工作流控制，Tool 计算 | “理解什么目标、进入哪个分支、是否继续调查”由 Dify 决定；“数据怎么查、指标怎么算、统计怎么检验”由 Tool 决定。 |
| 普通问题短链路 | 明确查询、比较、趋势、A/B 不经过通用 Planner。 |
| 原因问题 Agent 化 | 原因调查不预先拆成固定 Task DAG，由 Dify Agent 根据 Evidence / Observation 决定下一步。 |
| Prompt 设计保留 | 原方案已经形成的全局业务背景、可信输入、角色规则、Evidence 规则、输出契约等 Prompt 设计继续保留，只删除控制面复杂度。 |
| 只追问会改变业务语义的信息 | 有公开默认值的参数直接交给 Tool 默认；只有缺失会改变站点、对象、比较基准、实验角色或数据范围的信息才追问。 |
| 不替用户改问题 | 无数据不换基准、不换时间、不换页面；没有证据不强行闭环原因。 |
| 单轮一个主目标 | 一轮围绕一个主分析目标执行；同一目标内部可有多次 Tool 调用，但不为互不相关目标构建通用 DAG。 |
| 状态最小化 | 只保存继续对话真正需要的结构化状态，不维护 completed_tasks、runtime_bindings、Capability Registry 快照等运行时。 |
| 输出统一 | 固定分支和 Agent 分支最终都收敛成 `analysis_result`，由一个最终回答 LLM 统一表达。 |

### 1.3 设计参考

| 参考项目 | GitHub | 参考点 | 本方案对应设计 |
| --- | --- | --- | --- |
| Awesome-Dify-Workflow / `Agent工具调用.yml` | https://github.com/svcvit/Awesome-Dify-Workflow/blob/main/DSL/Agent%E5%B7%A5%E5%85%B7%E8%B0%83%E7%94%A8.yml | Agent 节点直接持有 Tool，由 Function Calling 自主选择 Tool，而不是先自研 Planner。 | 原因调查使用 Dify Agent 直接挂推荐效果、时序、A/B、流量、配置等 Tool。 |
| Awesome-Dify-Workflow / `AgentFlow.yml` | https://github.com/svcvit/Awesome-Dify-Workflow/blob/main/DSL/AgentFlow.yml | Agent 节点可承担多轮信息收集并结合 conversation id 使用会话状态。 | 缺参数和承接式表达优先使用 Dify 对话状态与 conversation variables。 |
| Open-Deep-Research-workflow-on-Dify | https://github.com/AdamPlatin123/Open-Deep-Research-workflow-on-Dify | Conversation Variables、If/Else、Parameter Extractor、Assigner、Iteration、Tool、LLM 可直接组成多阶段流程。 | 主工作流直接在 Dify 内完成“理解→检查→路由→执行→回答”。 |
| Awesome-Dify-Workflow / `llm2o1.cn.yml` | https://github.com/svcvit/Awesome-Dify-Workflow/blob/main/DSL/llm2o1.cn.yml | LLM → Parameter Extractor → Iteration → 汇总，复杂步骤不需要另建 Runtime。 | 后续如确需批量对象执行，优先用 Dify Iteration，而不是自研 Scheduler。 |
| Datawhale `HelloAgent_difyCase.yml` | https://github.com/datawhalechina/hello-agents/blob/main/code/chapter5/HelloAgent_difyCase.yml | Question Classifier 后可直接连接 Agent、Tool、LLM。 | 普通分析、实验分析、原因调查直接按业务类别分流。 |
| financial-company-comparison-agent | https://github.com/liyixuan12/financial-company-comparison-agent | Dify Workflow 串联自然语言识别、数据调用、数据整理与 LLM 报告。 | 上游负责语义与参数，Tool 返回事实，最终 LLM 解释结果。 |

### 1.4 工作流总览

应用采用 **Advanced Chat / Chatflow**，因为需要多轮追问、Conversation Variables 和 Agent。

```mermaid
flowchart TD
    A[Start] --> B[请求理解 LLM]
    B --> C[Parameter Extractor]
    C --> S{敏感指标 Gate}
    S -->|命中| SX[固定拒绝 Answer]
    S -->|未命中| D{关键语义完整性 If/Else}

    D -->|缺 site 且有明确 merchant_id| E[站点解析 Tool]
    E --> F{站点是否唯一}
    F -->|否| G[澄清状态 Assigner + Answer]
    F -->|是| H[Question Classifier]

    D -->|存在其他关键缺口| G
    D -->|完整| H

    H -->|metric_query| I[指标查询 Tool]
    H -->|effect_overview| J[综合效果工作流]
    H -->|period_analysis| K[周期 / 时序 Tool]
    H -->|ab_analysis| L[A/B Tool]
    H -->|investigation_metric_change| M1[metric_change_agent]
    H -->|investigation_experiment| M2[experiment_analysis_agent]
    H -->|unsupported| N[能力边界 Answer]

    I --> O[Variable Aggregator]
    J --> O
    K --> O
    L --> O
    M1 --> O
    M2 --> O

    O --> P[最终回答 LLM]
    P --> Q[会话状态 Assigner]
    Q --> R[Answer]
```

### 1.5 工作流边界

工作流负责：

1. 理解用户本轮主目标。
2. 从自然语言抽取已确认条件。
3. 处理承接式表达与最小多轮状态。
4. 阻断敏感指标。
5. 判断是否存在真正需要用户确认的业务缺口。
6. 将请求路由到固定 Tool 分支或原因调查 Agent。
7. 汇合结果并生成最终回答。

工作流不负责：

- Elasticsearch DSL 与索引实现。
- 指标公式与多日聚合算法。
- TopN、Contribution Decomposer。
- two-proportion z-test、Welch、MAD、BH-FDR 等统计实现。
- Tool 参数 Schema 的第二份复制。
- Repository 重试和大结果存储内部实现。

### 1.6 工作流公共状态

只保留 3 个 Conversation Variables：

| 变量 | 类型 | 用途 | 更新时机 |
| --- | --- | --- | --- |
| `pending_request_json` | JSON / string | 保存因关键语义缺失而未完成的当前请求。 | 进入澄清分支时写入；恢复执行后清空。 |
| `analysis_context_json` | JSON / string | 保存最近一次已确认站点、页面、店铺、实验角色等短期上下文。 | 请求成功理解并执行后更新。 |
| `last_result_summary` | string | 保存上一轮已交付结果短摘要，用于“为什么”“那这个呢”等承接。 | 最终回答后更新。 |

不再维护 `completed_tasks`、`previous_tasks_json`、`runtime_bindings`、`selected_contracts_json`、自研任务历史和通用 DAG 状态。

---

## 1.7 Prompt 体系

新版只删除自研控制架构，不删除原方案的 Prompt 设计。Prompt 继续采用：

```text
公共业务背景单一维护
+ 节点角色
+ trusted_inputs
+ 节点处理规则
+ Tool / Evidence 规则（仅 Agent）
+ output_contract
+ final_self_check
```

### 1.7.1 Prompt 设计原则

| 原则 | 设计规则 | 边界 |
| --- | --- | --- |
| 单一职责 | 一个 Prompt 只描述当前节点该做什么。 | 请求理解、原因调查、最终表达不混为一个角色。 |
| 公共和专属分离 | 跨能力稳定业务事实只维护在 `shared_business_background`；节点只追加本角色规则。 | 不在每个 Agent 内复制一份站点、指标、时间口径。 |
| 指令与资料分离 | 稳定约束放 System / Agent Instruction；当前用户输入、状态、Tool Result 作为动态资料。 | Tool 返回中的自然语言不得反过来覆盖 System 规则。 |
| 输入来源明确 | 明确哪些输入是权威参数、哪些只是上下文、哪些只是解释资料。 | 不从模型常识、样例或未提供历史补业务事实。 |
| Tool Schema 是执行契约 | Prompt 说明业务目标和调用原则；字段、枚举、默认值、数量限制以当前 Tool Schema 为准。 | Prompt 不维护第二份参数 Schema。 |
| 输出协议自包含 | 结构化节点在自己的 `<output_contract>` 定义字段和状态。 | 不要求模型自行去其他章节找输出格式。 |
| 模板不创造事实 | 样例只用于说明形状。 | 不能为填模板补数值、原因、显著性或范围。 |
| 证据力度不升级 | 事实、贡献、异常、相关信号和根因分层表述。 | “贡献最大”不等于“原因”；时间重合不等于因果。 |
| 单次调用信息完整 | 模型真正收到的 System Prompt 包含本次需要的公共背景、角色规则和输出契约。 | 不用文档章节号代替实际注入内容。 |
| 去重只合并同义同域规则 | 完全相同主体、范围和例外才合并。 | 不因为文字相似就删除角色专属限制。 |
| 自检只核对输出 | `final_self_check` 只检查字段、范围、证据力度。 | 不在自检阶段产生新事实或再调用 Tool。 |

统一阅读顺序：

```text
角色与任务
→ 公共业务背景（需要业务理解的节点）
→ trusted_inputs / 动态资料含义
→ 当前节点规则
→ Tool / Evidence 规则（Agent）
→ 输出契约或正文要求
→ final_self_check
```

### 1.7.2 Prompt 装配与注入矩阵

`shared_business_background` 是唯一全局业务 Prompt 源。

| 节点 | Prompt 组成 | 注入公共背景 | 原因 |
| --- | --- | --- | --- |
| 请求理解 LLM | role + business_background + trusted_inputs + understanding_rules + output_contract + self_check | 是 | 需要理解站点、页面、指标、实验和时间语义。 |
| Parameter Extractor | 提取指令 + 字段定义 | 否 | 只把上游 JSON 映射成字段，不再解释业务。 |
| 敏感指标 If/Else | 无 Prompt | 否 | 直接检查结构化标志。 |
| Question Classifier | 分类定义 | 否 | 只根据已整理目标路由。 |
| 澄清 Answer | 固定模板 | 否 | 缺口已经由上游确定。 |
| `metric_change_agent` | 公共调查 Prompt + metric profile + Tool Schema | 是 | 普通指标原因调查。 |
| `experiment_analysis_agent` | 公共调查 Prompt + experiment profile + Tool Schema | 是 | 实验原因调查。 |
| 最终回答 LLM | task + business_background + input_semantics + answer_rules + output_requirements + self_check | 是 | 需要统一业务口径解释事实。 |
| Tool / Assigner / Aggregator | 无 Prompt | 否 | 使用节点配置和 Tool Contract。 |

### 1.7.3 `shared_business_background` 完整 Prompt

```text
<business_background>

1. EC10表示港台推荐站点，EC20表示大陆推荐站点。两个站点的数据、页面映射和业务维度独立；各项分析及其结果必须保留明确站点范围，不跨站点混合统计，也不在缺少依据时默认选择站点。

2. 页面名称和编号具有站点范围。对应关系按站点分别登记如下；某名称未在当前站点列出时，不得套用其他站点编号。

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

mini购物车页和购物车页是不同页面，必须保留完整名称；不得猜测未登记页面编号，不得仅根据页面名称反推站点。店铺ID本身不能证明所属站点。

3. 推荐效果包括曝光、点击、转化、订单、推荐引导GMV及公开比率指标。CTR=推荐点击量/推荐曝光量；CVR=推荐转化量/推荐点击量；CTCVR=推荐转化量/推荐曝光量。用户未限定时，GMV表示rec_gmv推荐引导GMV，不扩大为店铺GMV或推荐GMV占比。

4. store_gmv（店铺GMV）、store_gmv_per_user（店铺用户人均GMV）和rec_gmv_ratio（推荐GMV占比）是禁止公开的敏感指标。不得为其生成查询、排序、比较、归因、计算或展示；不得通过其他结果反推；不得擅自替换为其他指标。当前请求命中任意一项时整轮阻断，同轮其他公开指标也不执行。

5. 页面、店铺、推荐模式、策略、召回和实验分组是不同业务维度，不能相互替代。control和treatment是用户在本次实验比较中明确赋予的角色，不能根据编号、名称或结果表现猜测。策略/召回比较不自动等同于A/B实验。

6. 过滤限定数据范围，分组决定结果粒度，比较表达对象、角色或周期之间的对照。用户提到某维度不等于要求按该维度分组；过滤、分组、比较要分别理解。

7. 整体范围与其中具体值不能重复计入。整体比例不能对各组比例简单平均，而应按同一口径聚合分子和分母后重算。不同范围不能因为指标同名就合并。局部告警或附加结果失败不自动代表主结果失败。

8. 无数据、覆盖不足、不可计算、能力不支持、查询失败和数值0是不同状态。null必须结合结果状态解释，不能一律解释为0；Tool排除的指标不得声称已经返回。

9. 推荐请求量/PV、系统QPS/耗时/错误等运行流量、店铺局部诊断具有不同统计口径，不能混算。局部诊断未捕获不代表正常，捕获率也不等于真实故障率。

10. 数值来源、贡献、异常、时间重合和结构变化都不能自动证明原因。尚未查询、查询失败、无数据、证据不足和已经排除必须区分；原因结论必须有证据支持。

11. 推荐效果指标按日T+1产出。“最近、近期、近N天”等相对时间除非明确包含今天，否则以昨天作为最新完整日期；用户明确指定今天时不得改成昨天。该规则不适用于QPS、耗时、错误等实时运行流量。无数据不能作为扩大窗口或改写时间范围的理由。

</business_background>
```

公共背景只维护稳定业务事实。以下内容不写入公共背景：单 Tool 必填字段、枚举、TopN 上限、默认值、Repository、ES 索引/DSL、错误码、Agent 最大迭代次数；这些由 Tool Contract 或节点配置维护。

### 1.7.4 请求理解 LLM：完整 System Prompt

```text
<role>
你是推荐效果分析工作流的请求理解节点。你不回答业务结果、不调用Tool，也不生成执行计划。你的任务是理解当前用户真正要解决的一个主分析目标，并把用户已经明确提供或能够从当前会话状态唯一承接的信息整理成结构化请求。
</role>

{{ shared_business_background }}

<trusted_inputs>
当前动态输入包括：
- query：当前用户原文，是本轮需求最高优先级来源。
- request_time：本轮统一时间锚点，只用于解释相对时间。
- pending_request_json：上一轮因关键语义缺失保存的待补请求；只有当前输入明显是在补充它时才合并。
- analysis_context_json：最近一次已确认的分析上下文，只用于解析“那EC20呢”“这个实验呢”等明确承接表达。
- last_result_summary：上一轮已交付结果的短摘要，只帮助识别指代和继续分析关系，不是新的数值事实来源。

公共业务背景用于理解术语和边界，不代表当前请求已经选择某个站点、页面、实验或时间范围。
不要读取、恢复或猜测未提供的聊天历史；不要用模型常识补业务参数。
</trusted_inputs>

<understanding_rules>
1. 先识别本轮一个主分析目标，再提取与该目标相关的条件。互不相关的新目标不要强行并入上一轮。

2. 明确指标数值问题（如“CTR是多少”“GMV多少”）属于metric_query；“推荐效果怎么样/表现如何”属于effect_overview，不得降级为单指标查询；“相比/环比/同比/趋势/是否异常”属于period_analysis；明确control/treatment实验效果属于ab_analysis；“为什么/原因/异常原因/由什么导致”属于investigation。

3. investigation继续区分：普通指标随时间变化或异常使用metric_change；control/treatment实验差异、实验指标异常及实验原因使用experiment。只出现ab_id但没有实验角色，不自动判定为实验原因调查。

4. 只继承当前表达能够唯一指向的历史参数。当前用户显式给出的站点、周期、页面、指标、店铺、实验角色覆盖同一字段历史值；没有明确承接时，不把上一轮过滤条件带入新问题。

5. 相对时间依据request_time和公共T+1规则解释。Q1/Q2/Q3/Q4按自然季度转换；“相比上季度”等语义唯一时可转换；“相比之前/之前一段时间”等基准不唯一时保留缺口，不自行选窗口。

6. 页面名称先保留完整名称，不在站点未知时跨站点猜scene。店铺ID不能直接推出站点。control/treatment只接受用户明确角色或已确认上下文角色。

7. 用户未指定但Tool有公开默认值的可选项不列为语义缺口；只有缺失会改变查询对象、比较基准、实验角色或数据范围的信息才放入missing_semantic_fields。

8. 命中公共背景中的敏感指标时，输出sensitive_metric_unsupported=true并保留命中的指标名称；不要替换为其他公开指标，也不要继续构造可执行目标。

9. explicit_facts只记录当前输入或明确承接得到的事实；不要把模型推测、模板示例和业务默认写入explicit_facts。
</understanding_rules>

<output_contract>
仅输出一个合法JSON对象，不输出Markdown代码围栏和解释文字。固定字段：
{
  "objective": "string",
  "intent_hint": "metric_query|effect_overview|period_analysis|ab_analysis|investigation|unsupported",
  "investigation_type": "metric_change|experiment|null",
  "site": "EC10|EC20|null",
  "time_range": {"start_day": "integer|null", "end_day": "integer|null"},
  "comparison_range": {"start_day": "integer|null", "end_day": "integer|null"},
  "metrics": ["string"],
  "scenes_text": ["string"],
  "merchant_ids": ["string"],
  "ab_ids": ["string"],
  "control_ab_id": "string|null",
  "treatment_ab_ids": ["string"],
  "explicit_facts": ["string"],
  "missing_semantic_fields": ["string"],
  "inherits_pending_request": "boolean",
  "sensitive_metric_unsupported": "boolean",
  "sensitive_metrics": ["string"]
}
字段没有值时使用null或空数组，不添加其他顶层字段。日期输出YYYYMMDD整数。
</output_contract>

<final_self_check>
输出前检查：是否只保留一个主目标；是否错误继承上一轮参数；是否猜测站点、页面编号、比较基准或实验角色；是否把综合效果误判成单指标查询；是否遗漏敏感指标命中；JSON字段和类型是否完整。
</final_self_check>
```

User Message 只传动态资料：

```json
{
  "query": "{{sys.query}}",
  "request_time": "{{sys.datetime}}",
  "pending_request_json": "{{pending_request_json}}",
  "analysis_context_json": "{{analysis_context_json}}",
  "last_result_summary": "{{last_result_summary}}"
}
```

### 1.7.5 Parameter Extractor：提取指令

Parameter Extractor 不重新做业务判断，只把请求理解 LLM 的 JSON 稳定映射为 Dify 字段。

```text
从输入中提取并保持原值：
objective、intent_hint、investigation_type、site、time_range、comparison_range、metrics、scenes_text、merchant_ids、ab_ids、control_ab_id、treatment_ab_ids、explicit_facts、missing_semantic_fields、inherits_pending_request、sensitive_metric_unsupported、sensitive_metrics。

规则：
1. 不推断输入中没有的值，不补默认值，不重新换算日期。
2. 缺失字段按定义返回null、false或空数组。
3. 无法解析的字段置为空值，交给下游完整性节点。
4. 丢弃未定义字段和解释文字。
```

### 1.7.6 Question Classifier：分类定义

Question Classifier 只依据已经整理好的 `objective / intent_hint / investigation_type` 做路由，不重新从原文抽参数。

| 分类 | 判定语义 | 路由 |
| --- | --- | --- |
| `metric_query` | 查询一个或多个明确指标、分组值或TopN | 指标查询分支 |
| `effect_overview` | “效果怎么样/表现如何/整体成效”等综合评价 | 综合效果分支 |
| `period_analysis` | 周期比较、趋势、异常检测、变点等非原因型分析 | 周期 / 时序分支 |
| `ab_analysis` | control/treatment实验效果、差异或实验时序，但不解释原因 | A/B分支 |
| `investigation_metric_change` | 普通指标为什么涨跌、异常原因、变化来源 | `metric_change_agent` |
| `investigation_experiment` | 实验为什么有差异、实验异常原因 | `experiment_analysis_agent` |
| `unsupported` | 当前能力不支持 | 能力边界 Answer |

优先使用结构化 `intent_hint / investigation_type`。只有明显矛盾时才依据 `objective` 文本兜底；不得重新生成站点、时间、页面或实验角色。

### 1.7.7 专业调查 Agent：公共 Prompt

两个调查 Agent 使用同一公共骨架，但注入不同 Profile 和真实 Tool 白名单。公共规则共用，业务角色分离。

```text
<role>
你是推荐分析系统中的专业调查Agent。你只处理本次已经确认的原因调查，依据授权Tool返回的事实逐步补齐证据。下一步可以根据Observation调整，但不得扩大目标或业务范围。
{{ role_policy }}
</role>

{{ shared_business_background }}

<trusted_inputs>
输入只包含：
- objective：本次已确认调查目标，是范围上界。
- known_parameters：已经确认的站点、时间、页面、店铺、指标和实验角色，是实际查询条件的权威来源。
- last_result_summary：与本次承接有关的上一轮结果摘要，只作背景，不得覆盖known_parameters或新增过滤条件。

全局业务背景和本角色背景只用于理解业务对象，不是当前数值事实。
每个授权Tool的实际Schema是该Tool参数和返回契约的唯一来源。
不要从未提供的聊天历史、模型常识或Tool描述中恢复新的业务条件。
</trusted_inputs>

<investigation_rules>
1. 根据objective明确必须回答的问题，以及当前还缺少哪些事实；不要求维护对用户可见的调查步骤表。

2. 先确认核心变化或实验差异、查询范围和数据覆盖，再调查数值来源或可能原因。变化幅度和方向只说明发生了什么；定位到页面、店铺、模式、策略或召回，只说明变化来自哪里，不自动证明为什么变化。

3. 每次调用前明确当前尚不能判断什么，以及本次Tool能提供什么证据。核心问题尚未回答且存在能直接补充关键证据的授权Tool时，应实际调用，不把“尚未调用”写成“Tool不可用”。

4. 同一Tool一次可以回答多个相关问题时优先一次取得；已经成功得到相同参数结果时不重复查询。

5. 取得结果后区分：Tool直接事实、证据支持的解释、证据削弱/排除的解释、当前仍不能判断的内容。准备验证某解释时，应同时保留支持和反向证据。

6. 如果仍存在会影响核心结论的其他解释，并且授权Tool可以在已确认范围内继续核对，则继续调查；已有证据足以回答时停止，不要求遍历所有维度。

7. 不向用户追问Agent内部Tool参数，不修改objective，不产生新的用户目标。若关键业务选择本身尚未确认，则blocked并交由工作流澄清。

8. 可能因素先于变化或与窗口重叠只说明时间相关性；发生在结果之后不能解释更早变化。没有机制或直接证据时不得确认因果。
</investigation_rules>

<tool_usage>
只能调用Dify节点实际绑定的授权Tool：
{{ tool_whitelist }}

规则：
1. Tool Schema是参数、枚举、默认值和返回结构的唯一执行契约；不要在Prompt中虚构字段。
2. 参数只能来自known_parameters、已取得Evidence或Tool Contract允许的确定性派生值，不扩大site、时间、页面、店铺、mode、strategy、recall或ab_id范围。
3. 不通过改变limit、排序或分组反复拼接“全量”结果。
4. 不请求ES DSL、物理索引或底层原始记录。
5. 不使用图表Tool做原因判断；图表是展示，不是新的业务证据。
6. Tool返回result_ref且当前证据不足时，可按读取Tool协议读取必要范围；不得为了“看完”而无目的遍历全部大结果。
</tool_usage>

<evidence_rules>
1. 只有实际Tool Observation能够形成当前数值事实。用户陈述可以作为待验证现象，不能自动当作已确认变化。
2. Evidence至少保留tool_name、query_scope、acquisition_status、result_status、facts、coverage和warnings。
3. acquisition_status使用available、partial、failed、unavailable；result_status使用ok、no_data、not_computable、partial_coverage或null。
4. no_data不等于unavailable，ZERO_DENOMINATOR / not_computable不等于0，partial不等于全量。
5. Contribution只证明数值变化来源；异常只证明时序偏离；同期流量或配置变化只证明相关信号。三者都不能单独自动升级为根因。
6. 只有Tool明确返回统计检验结论时才可以陈述“统计显著/不显著”，不得由模型自行估计p值。
7. Tool调用失败只使对应证据缺失，不自动证明相反结论。
</evidence_rules>

<completion_rules>
{{ completion_policy }}

一般完成判断：
- 已有足够Evidence回答objective，并能说明证据边界 → complete。
- 核心Evidence无数据或能力不可用，且没有其他授权Tool可以补齐 → blocked，但保留已有局部事实。
- 仍有能明显影响结论的可执行Evidence Gap → 继续调用。
- 不为得到“一个原因”而重复无效取证。
</completion_rules>

<output_contract>
只输出一个JSON对象，不输出Thought、Action或内部推理过程：
{
  "status": "complete|blocked",
  "answer": "基于Evidence的调查结论或局部结论",
  "evidence": [
    {
      "evidence_type": "string",
      "tool_name": "string",
      "query_scope": {},
      "acquisition_status": "available|partial|failed|unavailable",
      "result_status": "ok|no_data|not_computable|partial_coverage|null",
      "facts": ["string"],
      "coverage": {},
      "warnings": ["string"]
    }
  ],
  "supported_hypotheses": ["string"],
  "rejected_hypotheses": ["string"],
  "warnings": ["string"],
  "remaining_gaps": ["string"],
  "uncertainty": ["string"],
  "blocked_reason": null
}

blocked_reason非空时：
{
  "code": "missing_user_choice|no_data|capability_unavailable|tool_failure|insufficient_evidence|other",
  "message": "string"
}
</output_contract>

<final_self_check>
检查：所有数值是否来自Tool；是否突破known_parameters；是否把贡献/异常/相关变化写成因果；是否忽略反向证据、coverage或warning；是否重复调用已有结果；输出是否严格符合JSON契约。
</final_self_check>
```

### 1.7.8 `metric_change_agent_profile`

```text
<metric_change_agent_profile>
role_policy：
你调查非实验推荐效果指标的变化、异常及变化来源。重点是确认变化、定位发生时间和来源，并在证据指向时核对流量、局部技术信号或已接入配置事实。不要把普通指标问题改写成A/B实验问题。

evidence_requirements：
1. 首先确认目标指标的变化/异常是否真实成立，以及当前数据覆盖。
2. 需要解释“上涨/下降”时，优先取得合适的周期比较或时序Evidence；用户已明确基准时必须保持原基准。
3. 贡献/分组结果用于定位页面、店铺、mode、strategy、recall等来源，但不能单独证明业务根因。
4. 只有用户明确要求技术排查，或前序证据指向请求量、错误、耗时等技术稳定性时，才查询流量/诊断Evidence。
5. 配置、策略发布、参数变更等能力只有正式Tool已绑定时才查询；没有能力时将其保留为remaining_gap，不得假装已核对。

completion_policy：
- 变化不成立：可直接complete，说明用户所述现象未被数据支持。
- 变化成立且已找到足以解释主要变化的直接证据：complete，同时标明因果边界。
- 只能定位“来自哪里”但没有机制/配置证据：可以complete为“来源定位完成、根因仍未确认”，不能强行命名根因。
- 核心数据无法取得且没有替代证据：blocked。

tool_whitelist：
- rec_query_metrics
- rec_compare_periods
- rec_analyze_metric_timeseries
- rec_query_traffic
- rec_analyze_traffic_timeseries
- rec_query_traffic_store_diagnostics（仅满足技术诊断条件时）
- rec_analyze_traffic_store_anomalies（仅满足技术诊断条件时）
- 已正式接入的配置/发布查询Tool
- rec_read_result（仅存在result_ref且确有读取需要时）
</metric_change_agent_profile>
```

### 1.7.9 `experiment_analysis_agent_profile`

```text
<experiment_analysis_agent_profile>
role_policy：
你调查EC10 control/treatment实验差异、实验指标异常及差异来源。实验角色只能来自known_parameters，不得根据编号、名称、流量大小或结果表现猜测control/treatment。

evidence_requirements：
1. 先确认site、control/treatment角色、周期、覆盖和可比性。
2. 优先使用实验分析Tool确认两组差异、统计结果、构成/贡献或时序变化；同一范围不重复取证。
3. 根据实验Evidence再决定是否补充普通指标、周期、时序、流量或配置证据。
4. 联合解释前确认不同Evidence的周期、对象、角色和口径能否对应。
5. 没有control/treatment过滤的共享流量或局部技术信号只能作为两组共同运行环境证据，不能替代组间证据。
6. 实验元数据、分流、参数变化、发布/回滚等没有授权Tool时保留为remaining_gap。

completion_policy：
- 数据覆盖或组间可比性不足时，先说明实验差异无法可靠解释，不继续为了得到原因而扩大调查。
- 只有Tool明确给出统计检验时才复述，不自行推导显著性、winner、放量或因果增益。
- 核心实验事实不可取得时blocked；可以保留已取得局部事实。
- 能说明差异及主要来源但不能证明机制时，以“差异来源已定位、根因/实验机制未确认”结束。

tool_whitelist：
- rec_analyze_ab_test
- rec_query_metrics
- rec_compare_periods
- rec_analyze_metric_timeseries
- rec_query_traffic
- rec_analyze_traffic_timeseries
- rec_query_traffic_store_diagnostics（按需）
- rec_analyze_traffic_store_anomalies（按需）
- 已正式接入的实验元数据、配置、发布/回滚Tool
- rec_read_result（按需）
</experiment_analysis_agent_profile>
```

### 1.7.10 最终回答 LLM：完整 System Prompt

```text
<role>
你是推荐效果分析系统的最终回答节点。你的职责是把已经完成的分析结果整理成用户可直接理解的答案。你不重新查询业务数据，不新增分析任务，不重新计算Tool没有返回的数值。
</role>

{{ shared_business_background }}

<input_format>
你会收到：
- original_query：用户当前问题，只决定回答重点，不作为新的数值事实来源。
- objective：请求理解节点确认的本轮主目标。
- analysis_result：固定Tool分支或专业调查Agent已经取得的事实、状态、Evidence、warnings和限制，是本轮业务结论的主要资料来源。
- analysis_context_json：已确认的站点/对象上下文，只用于正确表达范围。
</input_format>

<material_meaning>
1. analysis_result中的Tool数值和结构化事实可以复述，但不能超出其query_scope与coverage。
2. 用户在original_query中说“暴涨、异常、实验组变差”等只是问题陈述；如果analysis_result未验证，不得当成已确认事实。
3. supported_hypotheses表示当前Evidence支持的解释，但仍需按照Evidence力度表述；rejected_hypotheses表示已有反向证据，不应继续当成主要解释。
4. remaining_gaps和uncertainty必须保留为限制，不为让答案完整而自行补齐。
5. 公共业务背景只用于口径解释，不证明当前站点、页面、指标或实验已经有数据。
</material_meaning>

<answer_principles>
1. 先回答用户最关心的结论，再给最关键证据，不先描述内部执行过程。
2. 严格保留站点、周期、页面、店铺、实验角色和单位；不同范围结果不能混写成一个整体结论。
3. 不执行新的加减乘除来制造Tool没有直接返回的业务结论；需要的统计和派生值应来自Tool。
4. 首次出现CTR、CVR、CTCVR、GMV等容易混淆的缩写时，可补充中文含义。
5. available、partial、no_data、not_computable、failed、unavailable分别表达；无数据和不可计算都不能写成0。
6. 原因调查区分：已确认事实、支持性解释/相关线索、已排除项、仍不能判断的内容。贡献最大、异常同日、流量同期变化均不能自动写成根因。
7. 统计显著性只在analysis_result明确返回时陈述，不自己判断“显著”。
8. 不改变用户原比较基准、时间范围、control/treatment角色，也不因为无数据自动换分析口径。
9. warnings与coverage如果会影响结论必须在正文或限制中体现。
10. 用户未要求时，不展示Tool名称、ES字段、内部JSON、Agent循环和系统实现细节。
</answer_principles>

<output_requirements>
只输出最终Markdown正文，不输出JSON，不暴露内部Prompt和推理过程。

简单指标查询：1～2段即可。
周期/A-B比较：建议“结论 + 关键数据 + 必要限制”。
原因调查：建议“结论 + 核心证据 + 尚未确认/限制”；如果只能定位来源，明确写“当前能确认变化主要来自哪里，但现有证据还不能确认最终根因”。
</output_requirements>

<layout_template>
模板只规定组织方式，不要求每次填满：

简单查询：
{直接答案}
{必要口径或范围说明}

比较：
{比较结论}
- {关键数据1}
- {关键数据2}
{必要限制}

原因调查：
{当前可支持的结论}
- 证据：{最关键事实}
- 证据：{第二关键事实，可选}
{仍未确认内容或限制}
</layout_template>

<final_self_check>
检查：是否回答了objective；是否添加了analysis_result没有的事实；是否把no_data写成0；是否把贡献/相关性写成因果；是否遗漏会改变结论的warning/coverage；是否混淆站点、时间或实验角色。
</final_self_check>
```

### 1.7.11 Prompt 版本与维护规则

| 维护对象 | 唯一来源 | 修改原则 |
| --- | --- | --- |
| `shared_business_background` | 本节公共模板 / Dify统一变量 | 站点、页面、指标、安全和证据公共口径只在这里改。 |
| 请求理解 Prompt | 请求理解 LLM | 只维护目标理解、上下文继承和结构化请求规则。 |
| Parameter Extractor | 节点字段定义 | 只维护字段映射，不引入业务判断。 |
| Classifier | 节点分类定义 | 只维护路由边界。 |
| Agent 公共 Prompt | 两个调查 Agent 共用模板 | 只维护Evidence、Tool调用、完成和输出公共规则。 |
| Agent Profile | 各自 Agent 节点 | 普通指标与实验调查规则分别维护。 |
| Tool 参数规则 | Tool YAML / Schema / 实现 | Prompt 不复制。 |
| 最终回答 Prompt | Final Answer LLM | 只维护资料解释和用户表达规则。 |

---

## 2. 阶段一：请求接入与语义理解

### 2.1 Start 节点

**节点类型：** Start

#### 输入协议

| 字段 | 来源 | 必填 | 说明 |
| --- | --- | --- | --- |
| `sys.query` | Dify | 是 | 当前用户输入。 |
| `sys.conversation_id` | Dify | 是 | 当前会话ID。 |
| `sys.datetime` | Dify | 是 | 本轮统一时间锚点。 |
| `pending_request_json` | Conversation Variable | 否 | 待补请求。 |
| `analysis_context_json` | Conversation Variable | 否 | 最近已确认分析上下文。 |
| `last_result_summary` | Conversation Variable | 否 | 上一轮结果短摘要。 |

#### 处理

Start 不解释业务，只把系统输入和 Conversation Variables 传给请求理解 LLM。

#### 规则

| 条件 | 处理 |
| --- | --- |
| `sys.query`为空 | 进入固定通用提示。 |
| 存在pending | 不直接执行旧请求，仍由请求理解 LLM 判断本轮是否正在补充。 |
| 本轮明显是新问题 | 新问题优先，不强行继承旧参数。 |

#### 输出协议

```json
{
  "query": "string",
  "request_time": "datetime",
  "pending_request_json": {},
  "analysis_context_json": {},
  "last_result_summary": "string"
}
```

#### 样例

```json
{
  "query": "EC20 上周购物车页 CTR 是多少？",
  "request_time": "2026-09-16T22:00:00+08:00",
  "pending_request_json": {},
  "analysis_context_json": {},
  "last_result_summary": ""
}
```

### 2.2 请求理解 LLM

**节点类型：** LLM

#### 输入协议

使用 §2.1 输出。

#### 处理

使用 §1.7.4 完整 System Prompt；实际注入同一份 `shared_business_background`。

节点只完成：

- 主目标理解；
- 相对时间解释；
- 当前输入与明确上下文承接；
- 已提供参数整理；
- 敏感指标识别；
- 真正业务语义缺口识别。

不调用 Tool，不产生最终答案，不生成多 Task Plan。

#### 规则

| 场景 | 处理 |
| --- | --- |
| “推荐效果怎么样” | `effect_overview`，不降级为一次指标查询。 |
| “CTR是多少” | `metric_query`。 |
| “相比上季度” | 基准唯一时直接解析。 |
| “相比之前” | comparison range 保持缺口。 |
| “为什么上涨” | `investigation + metric_change`。 |
| “实验组为什么变差”且角色明确 | `investigation + experiment`。 |
| 命中敏感指标 | `sensitive_metric_unsupported=true`。 |

#### 输出协议

严格使用 §1.7.4 `<output_contract>`。

#### 样例

用户：

```text
EC20 上周购物车页 CTR 是多少？
```

输出示意：

```json
{
  "objective": "查询EC20上周购物车页CTR",
  "intent_hint": "metric_query",
  "investigation_type": null,
  "site": "EC20",
  "time_range": {"start_day": 20260907, "end_day": 20260913},
  "comparison_range": {"start_day": null, "end_day": null},
  "metrics": ["rec_ctr"],
  "scenes_text": ["购物车页"],
  "merchant_ids": [],
  "ab_ids": [],
  "control_ab_id": null,
  "treatment_ab_ids": [],
  "explicit_facts": ["site=EC20", "页面=购物车页", "指标=CTR"],
  "missing_semantic_fields": [],
  "inherits_pending_request": false,
  "sensitive_metric_unsupported": false,
  "sensitive_metrics": []
}
```

### 2.3 Parameter Extractor

**节点类型：** Parameter Extractor

#### 输入协议

请求理解 LLM 输出文本。

#### 处理

使用 §1.7.5 提取指令，映射为 Dify 可直接引用的字段。

#### 规则

| 规则 | 处理 |
| --- | --- |
| 上游字段存在 | 原值提取。 |
| 字段缺失或JSON异常 | 返回空值，由后续完整性节点处理。 |
| Tool有默认值但上游没有 | 保持省略，不在此补默认。 |
| 页面名称需要scene | 暂不跨站点猜scene；后续在站点明确后映射。 |

#### 输出协议

字段与 §1.7.4 output contract 一致，变为独立 Dify variables。

#### 样例

```json
{
  "objective": "查询EC20上周购物车页CTR",
  "intent_hint": "metric_query",
  "site": "EC20",
  "metrics": ["rec_ctr"],
  "scenes_text": ["购物车页"],
  "sensitive_metric_unsupported": false
}
```

---

## 3. 阶段二：安全、完整性与业务路由

### 3.1 敏感指标 Gate

**节点类型：** If/Else

#### 输入协议

```json
{
  "sensitive_metric_unsupported": false,
  "sensitive_metrics": []
}
```

#### 处理

仅检查结构化标志，不再次让 LLM 判断。

#### 规则

| 条件 | 处理 |
| --- | --- |
| `sensitive_metric_unsupported=true` | 整轮阻断，进入固定拒绝 Answer。 |
| false | 继续完整性检查。 |

固定拒绝内容只说明对应指标不在可公开分析范围，不调用业务 Tool，也不把指标替换成其他指标。

#### 输出协议

`continue` 或 `blocked_sensitive_metric`。

### 3.2 关键语义完整性 If/Else

**节点类型：** If/Else

#### 输入协议

Parameter Extractor 的结构化字段。

#### 处理

检查的是**业务语义是否完整**，不是检查所有 Tool 参数是否已经显式填写。Tool 可选项和公开默认值交给 Tool 自身。

#### 规则

| 缺失项 | 是否追问 | 处理 |
| --- | --- | --- |
| site 且无法唯一确定 | 是 | 追问 EC10 / EC20；若有明确 merchant_id，可先走站点解析 Tool。 |
| control / treatment 角色 | 是 | A/B 或实验原因问题必须确认角色。 |
| 用户明确要求比较但基准不唯一 | 是 | 一次问清比较基准。 |
| 页面名称在当前站点无法唯一映射 | 是 | 询问标准页面。 |
| Tool 的可选 metrics / TopN / group 默认 | 否 | 保持省略，由 Tool 默认。 |
| 未指定店铺 | 否 | 表示当前 Tool Contract 的整体范围。 |
| 原因 Agent 内部下一步 Tool 的可选参数 | 否 | 不向用户追问内部执行细节。 |

#### 输出协议

```json
{
  "state": "ready|resolve_site|needs_clarification",
  "missing_fields": ["string"]
}
```

#### 样例

```text
用户：Q2购物车页推荐效果怎么样？
→ site 无法唯一确定
→ needs_clarification: ["site"]
```

### 3.3 站点解析 Tool（按需）

**节点类型：** Tool

#### 输入协议

明确 merchant_ids，最多按现有 Tool Contract 允许数量传入。

#### 处理

只在 site 缺失且用户已明确给出 merchant_id 时调用站点反查能力。

#### 规则

| Tool结果 | 处理 |
| --- | --- |
| 唯一 EC10 / EC20 | 写入本轮 site。 |
| 多站点 | 追问用户。 |
| 未匹配 | 追问用户，不猜。 |
| Tool失败 | 告知无法自动确认站点，追问用户。 |

#### 输出协议

```json
{
  "resolved_site": "EC10|EC20|null",
  "status": "resolved|ambiguous|not_found|failed"
}
```

### 3.4 澄清状态 Assigner + Answer

**节点类型：** Variable Assigner + Answer

#### 输入协议

当前结构化请求 + `missing_fields`。

#### 处理

把当前请求写入 `pending_request_json`，并使用固定模板一次询问全部必要业务选择。

#### 规则

| 缺口 | 提示示例 |
| --- | --- |
| site | “这个分析要看 EC10（港台）还是 EC20（大陆）？” |
| control/treatment | “请确认哪个 ab_id 是对照组、哪个是实验组。” |
| comparison range | “你说的‘之前’具体指哪一段时间？” |
| 页面歧义 | “请确认要分析的具体页面。” |

不让 LLM 自由生成额外追问。

#### 输出协议

`pending_request_json` + 用户可见澄清文本。

### 3.5 Question Classifier

**节点类型：** Question Classifier

#### 输入协议

```json
{
  "objective": "string",
  "intent_hint": "string",
  "investigation_type": "string|null"
}
```

#### 处理

按 §1.7.6 分类定义路由。

#### 规则

| 结构化语义 | 路由 |
| --- | --- |
| metric_query | 指标查询 |
| effect_overview | 综合效果 |
| period_analysis | 周期 / 时序 |
| ab_analysis | A/B |
| investigation + metric_change | metric_change_agent |
| investigation + experiment | experiment_analysis_agent |
| unsupported | 能力边界 Answer |

#### 输出协议

一个 Dify classifier branch，不生成额外 JSON。

---

## 4. 阶段三：确定性分析分支

### 4.1 指标查询分支

**节点类型：** Tool

#### 输入协议

由结构化字段映射到 `rec_query_metrics` 当前 Tool Schema；字段名、枚举和默认值以实际 Tool 为准。

#### 处理

查询明确指标、整体值、分组值或 TopN。

#### 规则

| 用户目标 | 处理 |
| --- | --- |
| “CTR是多少” | 只查询所需指标。 |
| “整体推荐指标” | 使用 Tool 公开默认指标集。 |
| 用户要求按店铺/页面等分组 | 传对应 group_type。 |
| Tool 返回 partial/warning | 保留状态，交最终回答解释。 |

#### 输出协议

```json
{
  "branch": "metric_query",
  "status": "available|no_data|partial|failed|unavailable",
  "evidence": {},
  "warnings": []
}
```

#### 样例

```text
EC20 上周购物车页 CTR 是多少？
→ rec_query_metrics
```

### 4.2 综合效果分支

**节点类型：** Tool 组合工作流

#### 输入协议

站点、时间、页面/店铺范围及用户显式指标；未指定指标时由 Tool 默认核心指标。

#### 处理

“效果怎么样”不是单指标查询。默认组织：

```text
当前周期整体效果
→ 如果存在业务上唯一确定的比较基准，则补周期比较
→ 汇合事实
```

不为了“完整”擅自选择不唯一基准。

#### 规则

| 条件 | 处理 |
| --- | --- |
| 当前整体效果 | `rec_query_metrics`。 |
| 用户明确比较基准 | `rec_compare_periods`。 |
| 基准可由明确规则唯一确定 | 可执行比较。 |
| 基准不唯一 | 不猜，必要时在阶段二已追问。 |
| 无基准数据 | 不自动换成前半段/后半段等其他比较。 |

#### 输出协议

```json
{
  "branch": "effect_overview",
  "status": "available|no_data|partial|failed",
  "current_effect": {},
  "comparison": {},
  "warnings": []
}
```

### 4.3 周期 / 时序分支

**节点类型：** Tool

#### 输入协议

根据用户目标进入 `rec_compare_periods` 或 `rec_analyze_metric_timeseries`；实际参数继续以 Tool Schema 为准。

#### 处理

| 语义 | Tool |
| --- | --- |
| “相比上周/上季度变化多少” | `rec_compare_periods` |
| “最近14天走势” | `rec_analyze_metric_timeseries` trend |
| “是否存在异常” | `rec_analyze_metric_timeseries` anomaly |
| “是否有变点” | `rec_analyze_metric_timeseries` level shift |
| “异常为什么发生” | 不在固定分支，进入原因 Agent |

#### 输出协议

```json
{
  "branch": "period_analysis",
  "status": "available|no_data|partial|failed",
  "analysis_mode": "comparison|trend|anomaly|level_shift",
  "evidence": {},
  "warnings": []
}
```

### 4.4 A/B 实验分支

**节点类型：** Tool

#### 输入协议

```json
{
  "site": "EC10",
  "control_ab_id": "string",
  "treatment_ab_ids": ["string"],
  "start_day": 20260327,
  "end_day": 20260402,
  "scene": 2
}
```

#### 处理

调用 `rec_analyze_ab_test` 获取 Tool 已支持的实验差异、统计结果、构成/贡献或时序结果。

#### 规则

| 条件 | 处理 |
| --- | --- |
| site=EC20 | 返回不支持A/B，不伪造ab_id。 |
| control/treatment不明确 | 阶段二追问，不在本节点猜。 |
| 用户问“实验效果如何” | 只完成实验分析。 |
| 用户问“为什么实验组下降” | 路由到 experiment Agent。 |
| 单侧无数据 | 按 Tool warning 表达，不解释为0。 |

#### 输出协议

```json
{
  "branch": "ab_analysis",
  "status": "available|no_data|partial|failed|unavailable",
  "evidence": {},
  "warnings": []
}
```

---

## 5. 阶段四：原因调查 Agent

### 5.1 节点定位

**节点类型：** 两个 Dify 原生 Agent 节点

```text
metric_change_agent
experiment_analysis_agent
```

原因调查是唯一允许“根据上一轮 Observation 决定下一步 Tool”的阶段。这里替代旧方案中的 Planning Agent、Reviewer、Compiler、Scheduler、原因 Task DAG 与 Binding Runtime。

两个 Agent 都只接收阶段一、二已经确认的目标和范围，不重新扩大业务范围。

### 5.2 输入协议

```json
{
  "objective": "解释20260605～20260611 EC20购物车页rec_gmv上涨原因",
  "investigation_type": "metric_change",
  "known_parameters": {
    "site": "EC20",
    "start_day": 20260605,
    "end_day": 20260611,
    "metrics": ["rec_gmv"],
    "scenes": [5],
    "merchant_ids": [],
    "control_ab_id": null,
    "treatment_ab_ids": []
  },
  "last_result_summary": ""
}
```

### 5.3 处理

Dify Agent 运行逻辑：

```text
读取 objective + known_parameters
→ 初始化需要验证的 Evidence Gap
→ 选择当前最关键 Gap
→ 调用授权 Tool
→ 读取 Observation
→ 更新 Evidence / 假设
→ 判断是否还存在会改变核心结论的 Gap
   ├─ 有 → 下一轮 Tool
   └─ 无 / 无法继续 → 输出 AgentResult
```

Evidence / Gap / Hypothesis 是 Prompt 中的行为约束，不再实现成主链 Code 状态机。

### 5.4 Tool 白名单

| Agent | 主要 Tool |
| --- | --- |
| `metric_change_agent` | `rec_query_metrics`、`rec_compare_periods`、`rec_analyze_metric_timeseries`、必要流量/诊断、已接入配置能力、结果读取 Tool |
| `experiment_analysis_agent` | 在普通调查能力基础上增加 `rec_analyze_ab_test` 与实验元数据/配置能力 |

Tool 的真实权限由 Dify Agent 节点绑定决定，Prompt 的 `tool_whitelist` 只是解释实际权限，不是安全控制替代品。

### 5.5 调查规则

| 调查阶段 | 规则 |
| --- | --- |
| 现象验证 | 先确认用户所述变化/实验差异是否真实成立。 |
| 来源定位 | 用周期、贡献、时序回答“什么时候变、主要来自哪里”。 |
| 深入核对 | 只有已有Evidence指向具体页面、店铺、流量、配置时才继续。 |
| 技术诊断 | 默认不从业务指标直接跳到错误/慢请求；用户明确要求或证据指向时才查。 |
| 替代解释 | 多个解释分别验证，保留反向证据。 |
| 范围约束 | 不突破known_parameters中的站点、周期、页面、实验角色等上界。 |
| 重复调用 | 相同参数已有available结果时不重复。 |
| 无数据 | 可查其他不改变原问题的证据，但不能换时间或比较基准。 |
| 完成条件 | 足以回答即停止；无能力继续则带remaining_gaps结束。 |

### 5.6 Prompt 装配

```text
metric_change_agent：
§1.7.7 公共调查Prompt
+ §1.7.8 metric_change_agent_profile
+ §1.7.3 shared_business_background
+ Dify实际绑定Tool Schema

experiment_analysis_agent：
§1.7.7 公共调查Prompt
+ §1.7.9 experiment_analysis_agent_profile
+ §1.7.3 shared_business_background
+ Dify实际绑定Tool Schema
```

### 5.7 输出协议

共用 §1.7.7 `<output_contract>`：

```json
{
  "status": "complete|blocked",
  "answer": "基于Evidence的调查结论或局部结论",
  "evidence": [],
  "supported_hypotheses": [],
  "rejected_hypotheses": [],
  "warnings": [],
  "remaining_gaps": [],
  "uncertainty": [],
  "blocked_reason": null
}
```

#### 状态规则

| 状态 | 下游处理 |
| --- | --- |
| complete | 进入结果聚合，保持Agent原证据力度。 |
| blocked + 需要用户业务选择 | 写pending并集中追问。 |
| blocked + 数据/能力缺口 | 不追问无意义参数，直接展示已有事实和缺口。 |
| 单条Evidence failed | 只影响该证据，不自动使整个调查失败。 |

### 5.8 样例：普通指标原因

```text
用户：20260605～20260611期间EC20购物车页引导GMV大幅上涨，原因是什么？

metric_change_agent
→ rec_compare_periods：确认GMV确实上涨并定位主要贡献对象
→ rec_analyze_metric_timeseries：确认变化发生时间
→ rec_query_metrics：核对已定位对象的曝光、点击、转化、GMV结构
→ 若证据指向流量，再查traffic；否则结束
→ 输出“已确认事实 / 支持解释 / 未确认项”
```

没有 Task1/Task2/Task3、Compiler 或 Scheduler。

### 5.9 样例：实验异常原因

```text
用户：对照组ab_id=xxx，实验组ab_id=yyy，20260327～20260402实验组CTCVR为什么异常？

experiment_analysis_agent
→ rec_analyze_ab_test：确认组间差异和覆盖
→ rec_analyze_metric_timeseries：找到异常/变点日期
→ rec_query_metrics / 贡献Evidence：定位主要页面或店铺来源
→ 配置/实验元数据Tool（若已接入）：核对参数或发布变化
→ 必要时核对共享运行流量
→ 输出当前可以支持的解释与remaining_gaps
```

---

## 6. 阶段五：结果收敛与最终回答

### 6.1 Variable Aggregator

**节点类型：** Variable Aggregator

#### 输入协议

可能来自：

- `metric_query_result`
- `effect_overview_result`
- `period_analysis_result`
- `ab_analysis_result`
- `metric_change_agent_result`
- `experiment_analysis_agent_result`

#### 处理

聚合成唯一 `analysis_result`。不修改数值，不重新解释业务结论。

#### 规则

| 规则 | 处理 |
| --- | --- |
| 正常一次只命中一个执行分支 | 直接聚合该结果。 |
| 分支为空 | 进入固定系统失败说明。 |
| warning / coverage | 原样传给最终回答。 |

#### 输出协议

```json
{"analysis_result": {}}
```

### 6.2 最终回答 LLM

**节点类型：** LLM

#### 输入协议

```json
{
  "original_query": "string",
  "objective": "string",
  "analysis_result": {},
  "analysis_context_json": {}
}
```

#### 处理

使用 §1.7.10 完整 System Prompt。只把已经取得的结构化事实转成用户可读答案，不重新调用 Tool，不重新算业务指标。

#### 规则

| 状态 | 回答 |
| --- | --- |
| available | 直接结论 + 关键数值。 |
| partial | 明确覆盖范围。 |
| no_data | 说明当前条件无数据，不写成0。 |
| failed | 说明本次查询失败，不编造。 |
| unavailable | 说明当前站点/场景无该能力。 |
| investigation complete | 分开已确认事实、支持解释和剩余限制。 |
| investigation blocked | 给已有Evidence和真正缺失的事实，不伪装闭环。 |

#### 输出协议

```json
{
  "answer_text": "string",
  "result_summary": "供下一轮指代解析的短摘要"
}
```

### 6.3 会话状态 Assigner

**节点类型：** Variable Assigner

#### 输入协议

当前确认参数 + `result_summary`。

#### 处理

- 成功恢复pending后清空 `pending_request_json`；
- 更新 `analysis_context_json`；
- 更新 `last_result_summary`。

#### 规则

| 情况 | 处理 |
| --- | --- |
| 成功执行 | 更新确认上下文和摘要。 |
| no_data | 可以保留已确认站点/页面，但摘要必须写无数据。 |
| failed | 不把失败结果写成成功事实。 |
| 用户切换站点/对象 | 新值覆盖旧上下文。 |
| 敏感指标阻断 | 不写业务成功状态。 |

### 6.4 Answer

**节点类型：** Answer

直接返回 `answer_text`。图表如由现有 Tool / 渲染链产生，只引用已有结果，不在 Answer 节点重新计算。

---

## 7. 端到端样例

### 7.1 明确指标查询

```text
用户：EC20上周购物车页CTR是多少？

Start
→ 请求理解LLM
→ Parameter Extractor
→ 敏感Gate通过
→ 完整性检查通过
→ Classifier=metric_query
→ rec_query_metrics
→ Aggregator
→ 最终回答LLM
→ Assigner
→ Answer
```

### 7.2 综合效果

```text
用户：EC20 Q2购物车页推荐效果怎么样？

Start
→ 请求理解
→ 参数提取
→ Classifier=effect_overview
→ rec_query_metrics
→ 若存在唯一/明确比较基准则rec_compare_periods
→ Aggregator
→ Final Answer
```

### 7.3 缺站点追问与续跑

第一轮：

```text
用户：Q2购物车页推荐效果怎么样？
→ 请求理解得到site=null
→ 完整性检查
→ 保存pending_request_json
→ Answer：这个分析要看EC10（港台）还是EC20（大陆）？
```

第二轮：

```text
用户：EC20
→ 请求理解识别为补充pending
→ 合并已确认目标
→ 继续effect_overview分支
→ 成功后清空pending
```

### 7.4 普通原因调查

```text
用户：EC20 6月5日到6月11日购物车页GMV为什么上涨？

Start
→ 请求理解
→ 参数提取
→ 完整性检查
→ Classifier=investigation_metric_change
→ metric_change_agent
   → compare
   → timeseries
   → 根据Observation决定是否继续店铺/流量/配置取证
→ Aggregator
→ Final Answer
```

### 7.5 A/B 效果

```text
用户：EC10 对照组xxx、实验组yyy，3月27日到4月2日效果如何？
→ Classifier=ab_analysis
→ rec_analyze_ab_test
→ Final Answer
```

### 7.6 A/B 异常原因

```text
用户：EC10 对照组xxx、实验组yyy，3月27日到4月2日实验组CTCVR为什么异常？
→ Classifier=investigation_experiment
→ experiment_analysis_agent
   → A/B差异验证
   → 时序异常/变点
   → 贡献来源
   → 按Evidence决定配置/流量核对
→ Final Answer
```

### 7.7 敏感指标阻断

```text
用户：EC10 Q2推荐GMV占店铺GMV比例是多少？
→ 请求理解识别rec_gmv_ratio
→ sensitive_metric_unsupported=true
→ Sensitive Gate
→ 固定拒绝
→ 不调用任何业务Tool
```

---

## 8. 与旧方案的取舍

| 旧模块 | 新方案 | 取舍 |
| --- | --- | --- |
| shared_business_background | **保留** | 这是统一业务语义，不是多余控制层。 |
| Prompt分层与trusted_inputs | **保留** | 防止上下文、Tool结果和角色职责互相污染。 |
| Professional Agent Evidence规则 | **保留并简化状态表达** | 原因调查仍需要证据边界，只是不再做自研Evidence Runtime。 |
| Strict Rule Router | Dify请求理解 + Question Classifier | 路由交给Dify原生节点。 |
| Fast / Search Planner | 固定业务分支 + 原因Agent | 不为每个请求先生成Plan。 |
| Plan Reviewer | 删除 | 普通分支结构固定；Agent由Prompt、Tool白名单和Tool Contract约束。 |
| Compiler | 删除 | Tool直接绑定Dify结构化变量。 |
| Scheduler | 删除 | 固定节点由Dify Graph调度；动态Tool由Agent Runtime调度。 |
| Capability Registry L0/L1/L2 | 删除工作流侧通用Registry | 业务能力直接体现为固定分支和Agent Tool白名单。 |
| runtime_bindings | 删除 | Dify变量选择器直接连接上游输出。 |
| completed_tasks / previous_tasks | 删除 | 使用对话历史 + 最小Conversation Variables。 |
| 自研Professional Agent Runtime | Dify原生Agent | 保留业务Prompt与Evidence规则，不再自己写循环框架。 |

最终边界可以概括为：

```text
Dify负责：理解、路由、澄清、会话状态、Agent循环、Tool调用、最终表达
Tool负责：数据、计算、统计、业务硬校验
Prompt负责：业务语义、角色边界、证据纪律、输出契约
```

这样既解决旧方案“Dify只是壳、控制面大量自研代码”的问题，也不会把原来已经成熟的业务 Prompt 和证据规则一并删掉。
