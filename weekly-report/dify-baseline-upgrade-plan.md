# Dify 1.17.1 基线升级方案

## 一、升级目标

当前平台已经在 Dify 上完成账号与工作空间治理、托管执行引擎、Console 调试、KMS/S3 存储安全和健康监控等企业能力。本次升级以 **Dify 1.17.1** 为新基线，将这些能力重新接入新版代码结构。

| 目标 | 内容 |
| --- | --- |
| 基线升级 | 平台统一升级到 Dify 1.17.1 |
| 能力保留 | SkyOA、工作空间、托管执行、调试、KMS/S3、监控等现有能力继续可用 |
| 数据兼容 | 保留账号、工作空间、执行记录、文件和租户私钥等历史数据 |
| 平稳切换 | 升级前完成数据备份，测试通过后切换新版运行组件，并保留回滚能力 |

---

## 二、升级范围

### 2.1 本次升级新增功能

以下按 **Dify 1.14.2 → 1.17.1** 的官方 Release Notes 与 1.17.1 代码能力整理，只保留新增的产品功能，不包含接口重构、权限细化、Bug Fix 和一般性体验优化。

| 功能方向 | 新功能 | 功能描述 |
| --- | --- | --- |
| Workflow | 自然语言生成 Workflow / Chatflow | 通过自然语言描述目标，自动生成 Workflow 或 Chatflow，并支持继续修改和完善 |
| Workflow | 推理过程展示 | Workflow、Chatflow 和 CLI 可以单独展示模型思考过程，并与最终答案区分 |
| Workflow | Human Input 富表单 | 人工输入节点支持下拉选择、文件和多文件上传，不再局限于文本输入 |
| Workflow | 长耗时模型任务 | Workflow 可以等待图片、视频等长时间生成任务完成，而不是因响应时间较长直接超时 |
| Workflow | Workflow 运行记录导出 | 支持导出 Workflow 历史运行记录，便于排查和留档 |
| Workflow | Tool 多选参数 | Tool 节点参数支持多选下拉，可一次选择多个预定义值 |
| Workflow | Workflow 节点定位 | 可以从运行日志或错误信息直接定位并高亮对应节点 |
| Workflow | Loop / Iteration 内 Human Input | Human Input 可以放入循环和迭代节点中，在循环过程中暂停并等待人工处理 |
| Workflow | LLM Environment | 可统一维护一套模型和参数配置，并由多个 Workflow 节点复用 |
| WebApp | WebApp 自定义展示 | Chatbot、Agent、Chatflow 可自定义输入提示，并在应用页面展示应用描述 |
| Agent | 新版 Agent App | 新增独立 Agent 应用形态，可单独创建、配置、运行和发布 |
| Agent | Agent Sandbox | Agent 自带独立运行环境，可执行代码和 Shell 命令 |
| Agent | Agent Skills | Agent 可以使用可复用的 Skills，将工具、代码和能力封装后重复使用 |
| Agent | Workspace Agent 管理 | 支持在工作空间中统一管理和复用 Agent，并可在 Workflow Agent 节点中引用 |
| Agent | Agent DSL 导出 | Agent 应用可以直接导出 DSL 配置用于备份和版本管理 |
| Agent | Agent Home Snapshot | 发布 Agent 时可以保存当前运行环境，后续运行从同一环境状态恢复 |
| Agent | E2B Sandbox | Agent 除本地 Sandbox 外，还可以使用 E2B 云端 Sandbox |
| Agent | 长会话自动压缩 | Agent 对话过长时自动压缩历史上下文，优先保留近期内容 |
| 可观测 | Unified Tracing | 统一记录应用、Workflow、节点、Loop、Iteration、Agent、Tool 等执行链路 |
| 可观测 | Knowledge Tracing | 增加知识库索引和检索过程的 Trace，可查看 RAG 数据处理链路 |
| CLI | difyctl | 可以通过命令行查看和运行 Dify 应用与 Workflow，用于脚本和自动化调用 |
| 知识库 | Excel 图片解析 | 导入 Excel 时可以提取表格内嵌图片，避免图片内容在知识库导入时丢失 |
| 知识库 | ODT 文档解析 | 知识库支持解析 OpenDocument Text（ODT）文档 |
| 知识库 | TiDB 混合检索 | TiDB Vector 支持全文检索和向量检索组合使用 |
| 多模态 | 文件直接传递给多模态模型 | LLM 和 Agent 可以直接将图片文件传递给支持视觉能力的模型 |
| 工具 | 日期参数类型 | Tool 插件参数新增 date / date-picker 类型 |
| 安全 | Cloudflare Turnstile | 登录和邮箱验证码登录可以启用 CAPTCHA 验证 |
| 密钥管理 | 外部 KMS Provider | 新增可插拔 KMS 能力，并提供 Azure Key Vault 接入与密钥轮换 |
| 数据治理 | 会话自动清理 | 可以按配置自动清理历史会话数据，控制长期数据增长 |
| Marketplace | 插件作者主页 | Marketplace 支持插件作者公开主页，集中展示作者发布的插件 |

### 2.2 原基线开发功能升级取舍

本表按小功能点拆分。**保留**表示本次继续迁移；**部分保留**表示业务能力保留、通用能力优先复用新版；**本期暂缓**表示本次先使用 1.17.1 官方能力，不再接入公司托管链路；**舍弃**表示功能本身不再恢复。

| 能力方向 | 功能点 | 功能描述 | 升级建议 | 判断依据 |
| --- | --- | --- | --- | --- |
| 账号与工作空间 | SkyOA 登录 | 使用 SkyOA 完成统一身份登录和账号关联 | 保留，必要适配 | 公司统一登录入口仍然需要，新版没有替代公司 SkyOA 业务规则 |
| 账号与工作空间 | 超级管理员初始化 | 初始化系统管理员并建立初始管理权限 | 保留，必要适配 | 属于公司平台治理规则，升级后仍然需要 |
| 账号与工作空间 | 邀请注册 | 通过邀请完成账号激活和工作空间加入 | 保留，必要适配 | 现有成员加入方式仍然有效，没有必要取消 |
| 账号与工作空间 | 默认工作空间 | 无有效工作空间的用户自动进入指定默认工作空间 | 保留，必要适配 | 属于公司租户治理规则，直接影响用户登录后的归属 |
| 账号与工作空间 | 工作空间管理 | 创建、归档、Owner 管理、当前工作空间选择等 | 保留，必要适配 | 仍是平台管理的核心能力 |
| 账号与工作空间 | 工作空间权限 | 区分系统管理员、Owner 和普通成员的操作权限 | 保留，必要适配 | 公司权限边界仍然需要继续生效 |
| 托管执行引擎 | 执行策略 Policy | 配置 Critical / Standard 等执行等级和优先级 | 保留 | 公司统一调度的核心规则，可继续复用 |
| 托管执行引擎 | Admission 准入控制 | 根据队列、租户和容量决定任务是否进入执行系统 | 保留 | 用于统一限制任务流量和容量，是托管执行的核心能力 |
| 托管执行引擎 | 优先级队列 | 按任务等级、优先级和进入顺序统一排队 | 保留 | 公司调度目标没有变化，可直接保留规则 |
| 托管执行引擎 | Scheduler | 从等待任务中选择可执行任务并分配到 Worker | 保留，适配新版执行入口 | 调度逻辑可复用，主要变化在与新版 Runtime 的连接 |
| 托管执行引擎 | Standard Worker | 执行普通优先级托管任务 | 保留，适配新版执行入口 | 双池隔离仍有价值，只需调整执行器调用方式 |
| 托管执行引擎 | Critical Worker | 执行高优先级托管任务 | 保留，适配新版执行入口 | 高优先级任务隔离仍然需要 |
| 托管执行引擎 | Worker 容量控制 | 通过心跳、并发和健康状态控制可用执行容量 | 保留 | 公司容量治理逻辑仍可复用 |
| 托管执行引擎 | Job 状态管理 | 记录 queued、running、completed、failed、cancelled 等任务状态 | 保留 | 托管任务查询、管理和结果交付均依赖该能力 |
| 托管执行引擎 | Streaming 结果交付 | 托管任务执行过程中持续向调用方返回结果 | 保留，适配新版结果流 | 正式应用仍需要流式输出，仅连接方式需要调整 |
| 托管执行引擎 | Blocking 结果交付 | 同步等待任务完成并返回最终结果 | 保留，适配新版结果流 | 正式 API 调用仍需要同步返回模式 |
| 托管执行引擎 | Cancel / Stop | 对排队任务取消、对运行任务停止 | 保留，适配新版停止机制 | 属于任务治理基础能力，新版已有停止能力可直接衔接 |
| 正式应用执行 | Workflow 托管执行 | Workflow 正式运行统一进入公司排队和容量治理 | 保留 | 正式执行链路价值明确，也是托管引擎最核心的落地点 |
| 正式应用执行 | Chatflow 托管执行 | Chatflow 正式运行统一进入公司托管调度 | 保留 | 与 Workflow 一样属于正式业务执行链路 |
| 正式应用执行 | Chat 托管执行 | Chat 应用统一进入公司托管调度 | 保留 | 正式执行仍可复用官方生成器，迁移成本相对可控 |
| 正式应用执行 | Completion 托管执行 | Completion 应用统一进入公司托管调度 | 保留 | 正式执行链路较独立，可继续接入 |
| 正式应用执行 | AGENT_CHAT 托管执行 | 原有 Agent Chat 应用进入公司托管调度 | 保留旧能力 | 现有应用仍可能依赖该模式；新版独立 Agent 作为另一套能力处理 |
| Console 调试 | Workflow / Chatflow 草稿托管调试 | 草稿运行也进入公司排队、状态管理和停止链路 | 本期暂缓 | 与新版草稿快照和调试入口耦合较深，优先使用 1.17.1 官方调试 |
| Console 调试 | 单节点托管调试 | 单节点测试进入公司排队并支持任务查询、停止 | 本期暂缓 | 依赖旧单节点内部执行接口，新版接口变化较大 |
| Console 调试 | Iteration 托管调试 | 单独调试迭代节点并接入公司任务管理 | 本期暂缓 | 涉及多轮事件和调试状态协议，迁移收益低于正式执行 |
| Console 调试 | Loop 托管调试 | 单独调试循环节点并接入公司任务管理 | 本期暂缓 | 与新版循环运行和事件协议耦合较深 |
| Human Input | 公司托管暂停恢复 | Human Input 暂停后通过公司 Job 代次、排队和恢复链路继续执行 | 本期暂缓 | 公司旧暂停数据格式与新版恢复上下文差异较大，本期直接使用官方暂停恢复链路 |
| 定时触发 | 正式 Schedule 托管执行 | 定时触发的正式 Workflow 统一进入公司调度 | 保留 | 属于正式应用执行入口，可与其他正式 Workflow 一起保留 |
| 定时触发 | 草稿 Schedule 调试 | 草稿定时触发测试进入公司托管调度 | 本期暂缓 | 属于调试能力，优先减少本期迁移范围 |
| 存储与数据安全 | KMS 凭据获取 | 通过公司 KMS 获取访问 S3 所需凭据 | 保留 | 现有存储认证依赖公司 KMS，无法直接取消 |
| 存储与数据安全 | KMS 凭据刷新 | 自动刷新临时凭据并处理失效重试 | 保留 | 保证长期运行时 S3 访问持续可用 |
| 存储与数据安全 | S3 存储接入 | 文件上传、下载、预签名等统一使用公司对象存储 | 保留必要接入 | 业务仍依赖现有对象存储，但通用文件能力优先复用新版 |
| 存储与数据安全 | 历史文件路径兼容 | 旧版本已经写入的文件 Key 继续能够读取 | 必须保留 | 直接关系到历史文件是否还能访问 |
| 存储与数据安全 | 租户私钥兼容 | 保留已有租户加密私钥及其对象存储引用 | 必须保留 | 直接关系到已有加密数据是否还能解密 |
| 存储与数据安全 | Redis 事件通信 | API、Scheduler、Worker 之间传递执行事件 | 保留 | 托管执行跨进程通信仍然需要 |
| 管理与可观测 | 任务管理 | 查询、筛选、取消和查看托管执行任务 | 保留 | 与托管执行引擎直接配套 |
| 管理与可观测 | 调度策略管理 | 管理优先级和执行策略 | 保留 | Policy 保留后，对应管理入口也需要保留 |
| 管理与可观测 | Worker / Scheduler 健康 | 查看 Worker 数量、容量、Scheduler 状态等 | 保留 | 属于公司托管执行专属健康信息，新版通用 Health 无法完全替代 |
| 管理与可观测 | OTel 执行监控 | 采集任务、Worker、Scheduler 等运行指标 | 部分保留 | 公司调度专属指标继续保留，通用 Trace 优先采用新版 Unified Tracing |
| 管理与可观测 | 执行审计 | 记录策略修改、任务操作、执行主体等审计信息 | 保留 | 管理类操作仍需要追踪 |
| 管理与可观测 | 执行数据自动清理 | 按保留周期自动删除托管执行历史数据 | 舍弃 | 该功能在原基线后续版本中已经被撤销，本次不重新恢复 |
| 部署与运行 | API / Web 构建 | 在 DevOps 中构建并发布 API、Web 制品 | 保留 | 已有发布链路可继续使用 |
| 部署与运行 | Worker / Scheduler 独立运行角色 | General Worker、Scheduler、Standard/Critical Worker 独立启动和扩缩容 | 保留，随迁移范围调整 | 托管执行保留后仍需要这些独立运行角色 |


---

## 三、总体方案

### 3.1 目标架构

![Dify 1.17.1 基线升级目标架构](assets/dify-baseline-upgrade/dify-baseline-upgrade-architecture.svg)

升级后仍然以 Dify 1.17.1 为主体：

- **Dify 原生能力**负责应用 Runtime、Console/WebApp、账号基础服务、Workflow Runtime 和文件基础能力。
- **企业增强能力**继续负责 SkyOA、工作空间治理、托管调度、KMS/S3、安全和平台管理。
- 旧版与新版职责冲突时，保留企业业务规则，底层实现按 1.17.1 当前接口重接，不恢复一套旧兼容 Runtime。

### 3.2 迁移原则

| 原则 | 处理方式 |
| --- | --- |
| 新版已有能力优先 | 新版已经重构的账号、Workflow、Session、文件能力直接复用 |
| 保留企业业务规则 | 调度优先级、默认工作空间、SkyOA、KMS 等公司规则继续保留 |
| 不整分支覆盖 | 按功能迁移，不直接将旧 feature 分支整体 merge 到 1.17.1 |
| 历史数据不重建 | 不重新生成账号、工作空间、私钥或批量改写 S3 对象 |
| 先公共底座后应用入口 | 账号、数据模型、调度底座先完成，再接 Workflow/Chat/调试 |

---

## 四、具体迁移方案

### 4.1 账号与工作空间

#### SkyOA 统一登录

| 迁移内容 | 具体代码调整 |
| --- | --- |
| 保留 SkyOA 授权入口和回调协议 | 在新版 api/controllers/console/auth/oauth.py 中补回 SkyOA provider，不整文件覆盖官方 OAuth Controller |
| 保留 open_id / userId 身份映射 | 将旧 Account.get_by_openid()、旧绑定方法改为 1.17.1 的 account_oauth_repository / identity 关联能力 |
| 适配新版 Session | 登录、查账号、注册、身份绑定全部显式传递新版 Session，重新划分 commit / rollback 边界 |
| 保留自动注册规则 | 仅已验证 SkyOA 身份允许自动建号，同时继续执行新版冻结、禁用、席位和账号限制 |
| 保留 state / nonce 校验 | 在 api/libs/oauth.py 和回调入口保留 nonce、Cookie、state 比对和日志脱敏 |
| 恢复前端登录入口 | 在新版登录页和账号绑定页面增加 SkyOA 按钮及绑定状态，不恢复旧页面整体实现 |

#### 超级管理员、邀请和工作空间

| 迁移内容 | 具体代码调整 |
| --- | --- |
| 超级管理员初始化 | 将 SkyOA bootstrap 身份接入新版 SetupService.initialize；保留一次性初始化和唯一管理员规则 |
| 初始化失败保护 | 修改旧 RegisterService.setup 的大范围清理逻辑，只回滚本次新建账号、租户和身份关联 |
| 邀请注册 | 保留 workspace_invitations、token 摘要、pending / accepted / cancelled / expired 状态；接受邀请接入新版 AccountActivationService |
| 默认工作空间 | 保留 Tenant.is_default 和唯一默认空间规则；无工作空间账号登录后加入默认空间，不创建个人空间 |
| current workspace | 新增/迁移 current tenant resolver，使用新版 Session 修复唯一 current，并跳过已归档空间 |
| 工作空间管理 | 迁移创建、指定 owner、归档、筛选和幂等创建逻辑；接口接到新版 workspace controller / query service |
| 权限和缓存 | 系统管理员、当前 workspace owner 和普通成员继续分层；切换/归档后同步失效 profile、workspace 和 RBAC 缓存 |

---

### 4.2 托管执行引擎

#### 数据模型与执行策略

| 迁移内容 | 具体代码调整 |
| --- | --- |
| 执行模型 | 迁入 api/models/app_execution.py 中 Job、Policy、Generation、Lease、Outbox、Worker 配置等模型 |
| 数据库 migration | 保留企业历史 revision，不修改已经执行过的 migration；将公司链与 1.17.1 官方链在最终版本合流 |
| Policy | 迁入 critical / standard、0~99 优先级、版本 CAS 和审计 |
| 调度权限 | 系统管理员走 /system 管理入口；workspace owner 只能管理当前空间允许范围内的调度策略 |

#### Admission / Queue / Scheduler

| 迁移内容 | 具体代码调整 |
| --- | --- |
| 统一准入 | 迁入 app_execution_admission_service，保留全局/租户队列上限、容量判断和准入锁 |
| 输入快照 | 按 1.17.1 输入对象重写 snapshot codec，不把旧 ORM 对象直接保存到跨进程任务 |
| 幂等入队 | Job 与 Outbox 同事务创建，保留 enqueue sequence、idempotency key 和原子状态转换 |
| Scheduler | 迁入独立 Scheduler、leader lease、优先级排序、queued timeout 和过期任务重排 |
| 终态收敛 | 保留取消、超时、执行异常和 recovery_required 的收敛逻辑，旧 Worker 的迟到结果不能覆盖新 generation |

#### Worker / Capacity / Result

| 迁移内容 | 具体代码调整 |
| --- | --- |
| 双池 Worker | 保留 critical / standard 两类 Worker 与独立队列，继续校验 Worker 身份 |
| Worker 生命周期 | 在新版 Celery bootstep / task 生命周期上迁入 claim、lease、续租、完成通知，不替换官方 Consumer |
| 容量控制 | 保留心跳 v2、目标并发、实际 pool 和健康 Worker 数量；在线扩缩容改为新版 QoS / consumer gate |
| Session 生命周期 | Worker 中按新版显式 Session 管理数据库连接，不复制旧 db.session.close() 方案 |
| Streaming | 复用 1.17.1 prepared subscription / 原生 SSE，增加 queued / running / terminal 的 Job 投影 |
| Blocking | 保留有界等待和完成通知，超时后仍可通过 Job/result 查询最终结果 |
| 任务管理 | 迁入任务列表、详情、筛选、取消、停止、恢复、SSE 刷新和系统管理员操作 |

---

### 4.3 应用执行与 Console 调试

#### 正式应用执行

| 功能 | 具体迁移方式 |
| --- | --- |
| Workflow | Service API / WebApp 入口先走受管 Admission；Worker 按 1.17.1 WorkflowAppGenerator / Workflow Runtime 执行，不恢复旧 Workflow Adapter |
| Chatflow | 保留 conversation / message / task 身份，按新版 AdvancedChatAppGenerator 重新实现 Worker 调用和结果投影 |
| Chat | 迁入 CHAT 的快照、队列和 stop；Worker 使用新版 ChatAppGenerator，保留官方消息落库、moderation、trace |
| Completion | 保留 Streaming / Blocking / result / stop；Worker 使用新版 CompletionAppGenerator，不套 Workflow 结果结构 |
| Agent | 先迁旧 AGENT_CHAT 受管执行；1.17.1 新 AppMode.AGENT 保留官方路径，待确认是否统一纳入托管执行 |

#### Console 调试

| 功能 | 具体迁移方式 |
| --- | --- |
| Workflow / Chatflow 草稿 | 迁入草稿 graph、features、inputs 冻结；排队后即使用户继续修改草稿，本轮仍执行已冻结版本 |
| 单节点调试 | 将旧单节点 Adapter 改接 1.17.1 run_draft_workflow_node / WorkflowEntry.single_step_run，补异步 accepted、查询和停止 |
| 迭代调试 | 受管准入后调用新版 single_iteration_generate，保留多轮事件、节点状态和停止 |
| 循环调试 | 调用新版 single_loop_generate，保留预分配运行身份、多轮事件、停止和 SSE 重连 |
| 前端状态 | 运行状态按 app / workflow / node / job 隔离，防止上一次调试结果覆盖新一轮或其他节点 |

#### Human Input 与定时触发

| 功能 | 具体迁移方式 |
| --- | --- |
| Human Input | 保留 Job generation / fence；表单、上传和 WorkflowPause 使用 1.17.1 原生实现 |
| 暂停恢复 | 表单提交后只创建新的受管 generation，由 Worker 恢复同一 task / conversation / message，不并行触发两套 resume |
| 旧暂停数据 | 升级前确认是否存在未完成暂停任务；存在时按旧 schema 与新版恢复上下文做兼容处理，不直接删除 |
| 定时触发 | 保留正式 schedule 统一准入和 trigger source；LEGACY / ENFORCED 模式只允许一个轮询者 |
| 草稿定时调试 | 冻结本次 trigger 输入，只创建调试 Job，不推进真实 schedule 的 next_run_at |

---

### 4.4 KMS、S3、文件与 Redis

| 功能 | 具体迁移方式 |
| --- | --- |
| KMS Provider | 将公司 KMS_URL、DOMAIN、SERVICE_NAME、签名协议和凭据刷新迁入新版 storage config |
| S3 Client | 在 1.17.1 aws_s3_storage.py 上接入 KMS provider，不整文件覆盖，保留新版预签名和流式读取能力 |
| 凭据刷新 | 保留定时刷新和失败重试；IAM、静态 AK/SK 和 KMS 三种来源明确分流 |
| 新文件 Key | 新上传统一使用企业约定的 Dify/... 路径规则 |
| 历史文件 | 保留数据库中的旧 UploadFile.key；读取时兼容历史 Key，不做全量对象搬迁 |
| 租户私钥 | 保留 encrypt_private_key 及旧私钥对象引用；新空间记录真实路径，不重新生成旧租户密钥 |
| S3 删除 | 单独验证知识库删除链路的实际 Key 和 DeleteObject 权限，解决“数据库删了但对象未删”的问题 |
| Redis 事件总线 | 明确 EVENT_BUS_REDIS_URL，保证 API / Scheduler / Worker 使用同一可达事件总线，不再默认落到各自 127.0.0.1 |

---

### 4.5 管理、监控与部署入口

| 功能 | 具体迁移方式 |
| --- | --- |
| Health | 迁入队列、Scheduler lease、Worker 数量、容量和依赖健康接口 |
| OTel | 将 Job、generation、Worker、Scheduler 指标接入 1.17.1 当前 OTel 初始化方式，避免重复注册 |
| Audit | 保留策略修改、Worker 配置、任务操作的 actor / tenant / reason / version |
| 运行入口 | 保留独立 API、Web、Scheduler、Worker 启动角色 |
| 打包 | 已迁入 API/Web 打包与产物清理；如果 Scheduler / Worker 复用 API 后端产物，则只增加启动角色，不复制后端包 |
| 环境配置 | 逐功能增加必要环境变量，真实 secret 由部署环境注入，不将旧 .env.test 整份覆盖新版配置 |

---

## 五、DevOps 部署方案

API、General Worker、Beat、Scheduler、Standard/Critical Worker 复用同一套 API 后端制品，通过不同应用和启动命令区分运行角色。

### 5.1 部署应用

| 应用 | 制品来源 | 是否新增 | 作用 |
| --- | --- | --- | --- |
| `dify-api` | API 制品 | 否 | Console / Service API、登录、管理入口 |
| `dify-worker` | API 制品 | 否 | Dify 官方 Celery 普通异步任务 |
| `dify-worker-beat` | API 制品 | 否 | 官方周期 Celery Task |
| `dify-scheduler` | API 制品 | 否 | App Execution Scheduler |
| `dify-worker-standard` | API 制品 | 是 | Standard 托管执行任务 |
| `dify-worker-critical` | API 制品 | 是 | Critical 托管执行任务 |
| `dify-web` | Web 制品 | 否 | Console / WebApp |
| `dify-sandbox` | 独立镜像 | 否 | Code Node 执行 |
| `dify-plugin-daemon` | 独立镜像 | 否 | Plugin 安装和运行 |

`dify-sandbox`、`dify-plugin-daemon` 本次不改启动方式，只核对兼容版本、配置和健康状态。新增的是 Standard/Critical 两个托管 Worker。

### 5.2 构建命令

API、Worker、Beat、Scheduler、Standard/Critical Worker 使用 API 制品：

```bash
TARGET_PROCESS=dify-api bash build.sh
```

Web 使用 Web 制品：

```bash
TARGET_PROCESS=dify-web bash build.sh
```

如果 DevOps 为 `dify-worker`、`dify-scheduler` 等应用分别传入自己的 `TARGET_PROCESS`，`build.sh` 只增加这些名称到“API 制品”分支，不能复制第二套后端打包逻辑。

API 制品必须包含：

```text
api 源码
pyproject.toml / uv.lock
dify-agent
.env.test
运行脚本
```

Web 制品必须包含：

```text
.next/standalone
.next/static
public
scripts/copy-and-start.mjs
运行用 package.json
```

### 5.3 各应用启动命令

#### dify-api

当前测试环境启动命令：

```bash
uv run flask run --host 0.0.0.0 --port=5001 --debug
```

该命令用于测试环境。数据库 migration 不再放进 API init 脚本，API 启动只负责启动服务。

#### dify-worker

General Worker 继续消费 Dify 官方普通队列：

```bash
cp .env.test .env && \
uv run celery -A app.celery worker \
  -P gevent \
  -c 1 \
  --loglevel INFO \
  -Q dataset,dataset_summary,priority_dataset,priority_pipeline,pipeline,mail,ops_trace,app_deletion,plugin,workflow_storage,conversation,workflow,schedule_poller,schedule_executor,triggered_workflow_dispatcher,trigger_refresh_executor,retention,workflow_based_app_execution
```

兼容阶段保留 `workflow_based_app_execution`；等所有应用正式入口、Console 调试、Human Input 和定时触发全部迁入托管链路后再移除。

#### dify-worker-beat

```bash
uv run celery -A app.celery beat
```

Beat 只负责官方周期任务，不承担 App Execution Scheduler 的优先级调度。

#### dify-scheduler

```bash
cp .env.test .env && \
uv run dotenv -f .env run -- bash -c '
exec env \
  APP_EXECUTION_PROCESS_ROLE=scheduler \
  SQLALCHEMY_POOL_SIZE="${APP_EXECUTION_SCHEDULER_DB_POOL_SIZE:-5}" \
  SQLALCHEMY_MAX_OVERFLOW="${APP_EXECUTION_SCHEDULER_DB_MAX_OVERFLOW:-0}" \
  flask app-execution-scheduler
'
```

该命令需要等 Scheduler 命令和相关代码完成迁移后再发布。

#### dify-worker-standard

目标平台命令统一走受管 Worker 启动脚本：

```bash
cp .env.test .env && \
APP_EXECUTION_WORKER_POOL=standard \
bash dev/start-app-execution-worker
```

只允许消费：

```text
app_execution_workflow_standard
```

#### dify-worker-critical

```bash
cp .env.test .env && \
APP_EXECUTION_WORKER_POOL=critical \
bash dev/start-app-execution-worker
```

只允许消费：

```text
app_execution_workflow_critical
```

Standard/Critical 的底层 Flask/Celery 入口随 3.11/3.28 代码一起迁入，DevOps 固定调用统一启动脚本，不在平台重复拼接内部参数。

#### dify-web

```bash
pnpm start
```

实际进入：

```text
node ./scripts/copy-and-start.mjs
```

`NEXT_PUBLIC_*` 在构建阶段写入产物，因此修改 `web/.env.test` 后必须重新构建，不能只重启旧 Web。

### 5.4 发布前配置

| 配置 | 要求 |
| --- | --- |
| API / Worker | 数据库、Redis、S3、KMS、Plugin Daemon 地址一致 |
| Redis Event Bus | 显式配置 `EVENT_BUS_REDIS_URL`，API/Scheduler/Worker 必须指向同一可达地址 |
| SkyOA | Client Secret 通过 DevOps Secret 注入，不写入源码和普通日志 |
| Web | `NEXT_PUBLIC_*` 在目标 Commit 构建前确认 |
| Standard Worker | `APP_EXECUTION_WORKER_POOL=standard`，只消费 standard 队列 |
| Critical Worker | `APP_EXECUTION_WORKER_POOL=critical`，只消费 critical 队列 |
| Scheduler | 只允许一个有效 leader；数据库池参数独立配置 |
| Sandbox / Plugin | 保持现有测试环境配置，确认 API Key / 内部认证与新版 API 一致 |

### 5.5 发布顺序

进入发布窗口后按以下顺序执行：

```text
1. 记录分支、Commit、构建号和配置版本
2. 停止 Scheduler、Beat 和所有 Worker 的任务消费
3. 完成 PostgreSQL 全库备份并验证备份文件
4. 执行一次性数据库 Migration Job
5. 发布 dify-api
6. 发布 dify-worker、dify-worker-beat
7. 发布 dify-scheduler
8. 发布 dify-worker-standard、dify-worker-critical
9. 发布 dify-web
10. 核对 dify-sandbox、dify-plugin-daemon
11. 执行完整验收
12. 恢复正常任务和流量
```

Migration 只能执行一次，不能让 API、Worker、Scheduler 的 init 脚本并发自动执行 migration。

### 5.6 队列切换

托管执行不能一次性把旧队列删掉，按以下顺序切换：

```text
兼容模式
  ↓
General Worker 保留 workflow_based_app_execution
  ↓
正式执行 / Console 调试 / Human Input / 定时触发全部迁移
  ↓
确认旧队列无积压、无新任务进入
  ↓
APP_EXECUTION_MODE=enforced
  ↓
Standard / Critical Worker 正常消费
  ↓
最后从 General Worker 移除 workflow_based_app_execution
```

禁止 General Worker 和专用 Worker 同时消费同一个受管执行队列。

---

## 六、数据备份与数据库升级

本次不做数据库副本运行。升级前直接对测试环境进行**完整备份**，备份成功并可读取后再执行 migration。

### 6.1 PostgreSQL 全库备份

使用 DevOps 数据库备份能力或执行等价的 `pg_dump`：

```bash
export BACKUP_FILE="dify_test_$(date +%Y%m%d_%H%M%S).dump"

PGPASSWORD="$DB_PASSWORD" pg_dump \
  -h "$DB_HOST" \
  -p "${DB_PORT:-5432}" \
  -U "$DB_USERNAME" \
  -d "$DB_DATABASE" \
  -Fc \
  -f "$BACKUP_FILE"
```

备份完成后必须验证文件可读取：

```bash
pg_restore -l "$BACKUP_FILE" >/dev/null
ls -lh "$BACKUP_FILE"
```

同时记录升级前 Alembic 版本：

```sql
SELECT version_num FROM alembic_version;
```

### 6.2 备份范围

这次升级涉及账号、工作空间、执行引擎和 Dify 原生数据，因此不再只备份 `tenants`、`tenant_account_joins`、`alembic_version` 三张表，直接备份整个 Dify PostgreSQL 数据库。

S3 本次不做全量复制，因为升级不搬迁已有对象；发布前只需要记录关键历史文件、UploadFile Key 和租户私钥 Key，用于升级后抽样验证。

Redis 不作为数据库回滚数据源，不通过恢复 Redis 队列完成回滚，避免重复消费旧任务。

### 6.3 Migration 执行

Migration 从长期服务中拆出来，作为一次性发布步骤执行。

以 Dify 1.17.1 的 migration 命令为准：

```bash
cp .env.test .env
uv run flask upgrade-db
```

如果 DevOps 当前还配置旧的：

```bash
uv run flask db upgrade
```

发布前统一确认目标分支实际 CLI，只保留一个 Migration Job；不能两个命令都执行。

Migration 完成后至少验证：

```sql
SELECT version_num FROM alembic_version;

SELECT id, name, status, is_default
FROM tenants
WHERE is_default = true;

SELECT tenant_id, account_id, role, current
FROM tenant_account_joins
WHERE tenant_id = '<默认工作空间ID>';
```

要求：

- Alembic revision 是目标代码当前 head；
- 默认工作空间只有一个且状态正常；
- 系统管理员在默认工作空间中仍为 owner；
- 不通过手工修改 `alembic_version` 跳过 migration。

### 6.4 回滚

回滚顺序固定为：

```text
停止新版 Scheduler / Beat / Worker
  ↓
保存 API / Worker / Web / Scheduler / Plugin / Sandbox 日志
  ↓
回滚到上一个稳定 Commit / 构建产物
  ↓
判断旧代码是否兼容已经升级的数据库
  ├─ 兼容：保留当前数据库
  └─ 不兼容：使用发布前 PostgreSQL 备份恢复
  ↓
重新验证 SkyOA、默认工作空间、owner、普通异步任务和应用执行
```

不把 `flask db downgrade` 作为默认回滚方案；是否能够 downgrade 需要逐条检查具体 migration。

---

## 七、测试与验收方案

测试分为“部署检查 → 数据库检查 → 功能回归 → 托管执行专项 → 外部依赖”五层，全部在当前测试环境完成，不再增加数据库副本运行。

### 7.1 部署检查

| 检查项 | 操作 | 通过标准 |
| --- | --- | --- |
| API | `curl -fsS http://127.0.0.1:5001/health` | HTTP 200 |
| Web | `curl -I http://127.0.0.1:3000` | 页面返回 200，静态资源无 404 |
| General Worker | `uv run celery -A app.celery inspect ping` | Worker 返回 pong |
| Beat | 检查启动日志和唯一实例 | 只有一个 Beat 在发布周期任务 |
| Scheduler | 检查 leader lease、派发日志和系统 Health | 只有一个有效 Scheduler leader |
| Standard Worker | 管理页/日志检查 pool 与队列 | 只消费 standard 队列 |
| Critical Worker | 管理页/日志检查 pool 与队列 | 只消费 critical 队列 |
| Sandbox | 运行一次 Code Node | 正常返回代码执行结果 |
| Plugin Daemon | 打开插件页并执行一个插件 | 安装/调用链路正常 |

### 7.2 数据库和工作空间

| 场景 | 操作 | 预期结果 |
| --- | --- | --- |
| Migration | 查询 `alembic_version` | 等于目标代码 revision |
| 默认工作空间 | 查询 `tenants.is_default=true` | 只有一条 |
| 系统管理员 | 查询 `tenant_account_joins` | 默认空间 role=owner |
| current workspace | 登录已有用户 | current 唯一且不会落到 archived workspace |
| 新用户 | 首次 SkyOA 登录 | 创建/关联账号并加入默认工作空间 |
| 已有用户 | SkyOA 登录 | 保留原 account_id 和成员关系 |

### 7.3 应用正式执行

每种应用至少执行一次 Streaming、Blocking、停止和结果查询：

| 类型 | 核心验证 |
| --- | --- |
| Workflow | 排队 → running → completed；Streaming / Blocking；stop |
| Chatflow | conversation/message 身份保持；多轮会话正常 |
| Chat | 消息落库、流式输出、停止和结果查询 |
| Completion | Streaming / Blocking 返回结构正确 |
| AGENT_CHAT | Tool/模型调用正常，受管队列状态正确 |

### 7.4 Console 调试

| 场景 | 核心验证 |
| --- | --- |
| Workflow / Chatflow 草稿 | 入队后继续编辑草稿，本次仍执行入队时快照 |
| 单节点 | accepted → job 查询 → 节点结果；可停止 |
| Iteration | 多轮事件、每轮节点状态和最终结果完整 |
| Loop | 多轮事件、SSE 重连后状态不丢失 |
| 前端状态 | 连续两次运行、多个节点并发时状态不串 |
| Human Input | 暂停 → 表单提交 → 新 generation 恢复 → 完成 |
| 定时触发 | 到点只产生一份正式 Job；草稿定时调试不推进真实 schedule |

### 7.5 托管执行专项

| 场景 | 通过标准 |
| --- | --- |
| Standard / Critical 路由 | 两类任务进入对应队列和 Worker |
| 优先级 | 同一等级内按 priority / enqueue sequence 调度 |
| Worker 下线 | 不再向不健康 Worker 放量 |
| Lease | Worker 丢失租约后旧结果不能覆盖新 generation |
| Queue Limit | 超过全局/租户上限时拒绝准入并返回明确状态 |
| Cancel / Stop | queued 可取消；running 可停止；终态只收敛一次 |
| Streaming | queued/running/terminal 事件顺序正确 |
| Blocking | 超时后仍可通过 Job/result 查询最终结果 |
| Scheduler HA | 多实例情况下只有 leader 派发 |

### 7.6 存储、插件和安全

| 场景 | 通过标准 |
| --- | --- |
| KMS | 能取凭据、刷新后继续访问 S3 |
| 文件上传 | 新文件 Key 使用新规则，数据库记录可读 |
| 历史文件 | 原 UploadFile.key 仍可下载和预签名 |
| 文件删除 | 知识库删除后数据库记录和 S3 对象同时删除 |
| 租户私钥 | 旧租户私钥仍可读取，不重新生成 |
| Redis Event Bus | API/Scheduler/Worker 事件可跨进程交付 |
| Plugin | Plugin Daemon 认证、安装和执行正常 |
| 日志 | 不出现 SkyOA Secret、KMS Secret、完整 Token |

### 7.7 发布通过条件

只有以下全部通过才算升级完成：

```text
所有应用来自同一目标 Commit
数据库 Migration 成功
SkyOA / 默认工作空间 / owner 正常
General Worker / Beat / Scheduler / 双 Worker 职责不串
Workflow / Chatflow / Chat / Completion / AGENT_CHAT 正式执行通过
草稿 / 单节点 / Iteration / Loop / Human Input / Schedule 通过
KMS / S3 / Sandbox / Plugin Daemon 通过
Health / Audit / OTel 无阻断问题
回滚入口和数据库备份可用
```

## 八、实施时间表

状态说明：**✅ 已完成　🟡 进行中　⬜ 待开展**

<table>
  <thead>
    <tr><th>时间</th><th>功能</th><th>任务项</th><th>完成</th></tr>
  </thead>
  <tbody>
    <tr><td rowspan="7">09.21 - 09.25</td><td rowspan="2">基线与构建</td><td>确定 Dify 1.17.1 基线并建立升级分支</td><td>✅</td></tr>
    <tr><td>迁入 API/Web 平台构建、产物清理和依赖路径处理</td><td>✅</td></tr>
    <tr><td rowspan="5">账号与工作空间</td><td>SkyOA OAuth / state / 身份映射迁移</td><td>⬜</td></tr>
    <tr><td>超级管理员初始化接入新版 SetupService</td><td>⬜</td></tr>
    <tr><td>邀请注册接入新版 AccountActivationService</td><td>⬜</td></tr>
    <tr><td>默认工作空间、current workspace 和 owner 保护迁移</td><td>⬜</td></tr>
    <tr><td>工作空间创建、归档、权限和缓存迁移</td><td>⬜</td></tr>

    <tr><td rowspan="7">09.28 - 10.02</td><td rowspan="7">托管执行底座</td><td>迁入执行模型及企业历史 migration</td><td>⬜</td></tr>
    <tr><td>完成官方链与企业 migration 链合流</td><td>⬜</td></tr>
    <tr><td>迁移 Policy 与 workspace 调度权限</td><td>⬜</td></tr>
    <tr><td>迁移 Admission、Queue、Input Snapshot</td><td>⬜</td></tr>
    <tr><td>迁移 Scheduler、Lease、Outbox 和终态收敛</td><td>⬜</td></tr>
    <tr><td>迁移 Worker、心跳 v2、执行租约和容量管理</td><td>⬜</td></tr>
    <tr><td>迁移 Job 管理、Streaming、Blocking 和结果查询</td><td>⬜</td></tr>

    <tr><td rowspan="8">10.05 - 10.09</td><td rowspan="5">正式应用执行</td><td>Workflow 正式执行接入新版 Workflow Runtime</td><td>⬜</td></tr>
    <tr><td>Chatflow 正式执行和会话身份迁移</td><td>⬜</td></tr>
    <tr><td>Chat 正式执行、结果和停止迁移</td><td>⬜</td></tr>
    <tr><td>Completion Streaming / Blocking / Stop 迁移</td><td>⬜</td></tr>
    <tr><td>AGENT_CHAT 受管执行迁移，新 AGENT 范围确认</td><td>⬜</td></tr>
    <tr><td rowspan="3">Console 调试</td><td>Workflow / Chatflow 草稿冻结、排队、查询和停止</td><td>⬜</td></tr>
    <tr><td>单节点、迭代、循环调试接入新版原生执行入口</td><td>⬜</td></tr>
    <tr><td>前端运行状态、SSE 重连和新旧轮次隔离</td><td>⬜</td></tr>

    <tr><td rowspan="7">10.12 - 10.16</td><td rowspan="2">暂停与调度</td><td>Human Input generation、暂停、恢复和重试迁移</td><td>⬜</td></tr>
    <tr><td>正式定时触发与草稿 schedule 调试迁移</td><td>⬜</td></tr>
    <tr><td rowspan="4">存储与安全</td><td>KMS Provider 和凭据刷新接入新版 S3</td><td>⬜</td></tr>
    <tr><td>新旧文件 Key 与租户私钥引用兼容</td><td>⬜</td></tr>
    <tr><td>知识库文件 S3 删除链路修复</td><td>⬜</td></tr>
    <tr><td>Redis Event Bus 地址和跨进程事件交付配置</td><td>⬜</td></tr>
    <tr><td>管理与监控</td><td>Health、OTel、Audit、Scheduler/Worker 运行入口迁移</td><td>⬜</td></tr>

    <tr><td rowspan="6">10.19 - 10.23</td><td rowspan="6">验收与上线</td><td>完成 PostgreSQL、关键 S3 Key、配置和 migration 状态备份</td><td>⬜</td></tr>
    <tr><td>执行数据库 migration 并检查历史数据</td><td>⬜</td></tr>
    <tr><td>完成 API / Web / Scheduler / Worker 启动验证</td><td>⬜</td></tr>
    <tr><td>完成账号、正式执行、Console 调试和 Human Input 回归</td><td>⬜</td></tr>
    <tr><td>完成 KMS/S3、任务管理、Health 和 Audit 回归</td><td>⬜</td></tr>
    <tr><td>完成正式切换和回滚验证</td><td>⬜</td></tr>
  </tbody>
</table>
