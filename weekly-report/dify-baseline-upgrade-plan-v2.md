# Dify 1.17.1 基线升级方案

## 一、升级目标

当前平台基于 Dify 1.14.2，并扩展了 SkyOA、工作空间治理、S3/KMS、文件与租户私钥、托管执行调度等公司能力。本次升级以 Dify 1.17.1 为新基线，分三个阶段完成现有能力迁移、新版本能力升级和执行调度能力处理，最终将成熟的 1.17.1 版本收束到原有 DevOps 组件。

| 目标 | 内容 |
| --- | --- |
| 基线升级 | 平台基础版本由 Dify 1.14.2 升级到 Dify 1.17.1 |
| 现有能力迁移 | 将 SkyOA、工作空间、S3/KMS、文件与私钥等现有公司能力适配到 1.17.1 |
| 新能力升级 | 在阶段一形成的 1.17.1 版本上增加确定要启用的新功能及对应运行组件 |
| 调度能力处理 | 对现有执行调度、任务中心和专用 Worker 确定迁移、重写或取消方案 |
| 数据迁移 | 阶段一完成后先将数据迁到独立 1.17.1 数据空间，三阶段完成后再执行最终正式数据迁移 |
| 最终收束 | 用最终 1.17.1 制品替换原有组件部署，并下线临时升级环境 |

---

## 二、升级架构

### 2.1 当前架构与 Dify 1.17.1 官方架构

![当前架构 vs Dify 1.17.1 官方架构](assets/dify-baseline-upgrade/current-vs-dify-1.17.1-official-v11.svg)

### 2.2 本次迁移架构

![Dify 1.17.1 升级实施路径](assets/dify-baseline-upgrade/dify-1.17.1-migration-scope-v14.svg)

---

## 三、阶段一：现有能力迁移

**功能总览**

| 能力方向 | 当前能力 | 阶段一处理 | 1.17.1 落点 |
| --- | --- | --- | --- |
| 平台构建与部署 | API / Web 公司构建方式、环境配置、启动参数 | 按 1.17.1 依赖和启动方式调整公司构建与发布入口 | API / Web 官方制品与运行角色 |
| 数据库与 Redis | 当前测试库、缓存、Celery 使用旧版数据空间 | 建立独立 1.17.1 数据库、Redis DB、Prefix 和队列空间 | PostgreSQL / Redis |
| Redis Event Bus | Sentinel 环境下跨进程消息 | Sentinel 模式复用远程 Redis 客户端，并隔离频道 | Redis Event Bus |
| Socket.IO | 1.17.1 新增工作流实时协作 RedisManager | 增加 Sentinel URL、认证、DB 和频道隔离 | Socket.IO |
| SkyOA 登录 | Provider、回调、账号匹配和绑定 | 适配新版 Account、Identity、Repository 和 Session | Account / Identity |
| 管理员初始化 | SkyOA 首次安装、初始管理员、失败清理 | 接入新版 Setup 流程和新版事务模型 | Setup / Account / Workspace |
| 邀请注册 | 数据库邀请记录、SkyOA 接受邀请 | 适配新版 Account Activation 和成员接口 | Account / Workspace |
| 默认工作空间 | 唯一默认空间、新用户加入、current workspace | 适配新版 Tenant、Session 和成员关系 | Workspace / Tenant |
| 工作空间管理 | 创建、指定 owner、查询、切换、归档和权限 | 复用新版成员、Session 和 RBAC 基础 | Workspace |
| SSE 请求 | 自定义 Header 与认证 Header 合并 | 迁入独立于调度的通用 Header 修复 | Web Request / SSE |
| S3 / KMS | KMS 凭据、刷新、S3 Client | 将公司 KMS Provider 接入 1.17.1 Storage | Storage Provider |
| 文件与租户私钥 | 对象前缀、应用目录、历史文件、RSA 私钥 | 适配新版 Storage / FileService 并兼容历史引用 | Storage / Tenant |

### 3.1 功能迁移

#### 3.1.1 SkyOA 登录

**来源：** `origin/feature/20260701` 的 SkyOA 登录能力，以及 `origin/feature/20260825_S3` 中账号邮箱匹配和资料补充。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 登录入口 | SkyOA 与官方 Provider 并存 | 官方 OAuth 入口和 Service 已调整 | 保留官方 GitHub / Google 等流程，只为 SkyOA 增加独立 Provider 分支 |
| 回调协议 | SkyOA 使用 POST `code`、`state`、`currentUrl` | 官方 Provider 主要使用自身 OAuth 回调协议 | 保留 SkyOA 现有 POST 回调，不强行改造成其他 Provider 协议 |
| state / nonce | 短期 Cookie 保存登录上下文 | 新版 OAuth 状态管理变化 | 保留 SkyOA 的 state / nonce 校验，生命周期与官方登录流程隔离 |
| 账号查找 | 优先按 OA 绑定查找，未绑定时按邮箱匹配旧账号 | 新版使用 Repository、`normalized_email` 和显式 Session | 按新版 Repository / Session 重写调用，保留“绑定优先、邮箱兜底、重复邮箱拒绝”规则 |
| 身份绑定 | 登录后将 openId 关联到原账号 | Identity 关联接口和事务方式变化 | 使用 1.17.1 Identity 能力完成绑定，不改变原 `account_id` |
| 账号信息补充 | OA 返回姓名、邮箱、头像等资料 | 新版账号字段和更新入口变化 | 只补充公司需要的账号资料，不覆盖新版账号治理逻辑 |
| 前端入口 | 登录页与账号设置页展示 SkyOA | 新版页面结构变化 | 在 1.17.1 页面中恢复 SkyOA 登录按钮和绑定状态 |

#### 3.1.2 管理员初始化

**来源：** `origin/feature/20260701` 的 SkyOA 首次初始化流程，以及 `origin/feature/20260825_S3` 的失败清理修复。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 初始化入口 | 启用 SkyOA 时通过 OA 身份创建初始管理员 | 官方 Setup 初始化流程变化 | 在 1.17.1 Setup 流程增加 SkyOA 初始化分支，未启用 SkyOA 时保持官方流程 |
| 初始化检查 | 仅允许未初始化环境执行 | 新版已有安装状态与初始化检查 | 直接复用 1.17.1 官方初始化状态判断 |
| 管理员账号 | 初始化时创建公司管理员账号 | 新版 Account 创建与 Session 方式变化 | 使用新版账号创建接口和 Session，保留公司管理员身份 |
| 初始 Workspace | 初始化管理员同时创建 Workspace | Workspace 创建流程和事务边界变化 | 调用统一 Workspace 创建能力，不在 Setup 中重复实现一套 |
| 初始化登录 | 创建完成后直接进入系统 | 登录返回和 Session 结构变化 | 按 1.17.1 登录态重新适配初始化后的跳转 |
| 失败清理 | 旧实现中存在清理范围过大的风险 | 新版事务边界变化 | 失败时只回滚和清理由本次初始化创建的账号、Workspace 和 Identity 关系 |

#### 3.1.3 邀请注册

**来源：** `origin/feature/20260701` 的数据库邀请与 SkyOA 接受邀请流程。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 邀请记录 | 邀请持久化到数据库，维护 pending / accepted / cancelled / expired 等状态 | 官方邀请与激活流程变化 | 保留公司邀请模型、状态和有效期 |
| Token | 保存邀请 Token 摘要并校验有效性 | 官方邀请 Token 逻辑不同 | 保留现有 Token 语义，兼容已有邀请链接 |
| 邀请登录 | 邀请场景只允许 SkyOA | 官方激活页支持其他登录方式 | 公司邀请页面继续只展示 SkyOA，不影响普通登录入口 |
| 邮箱校验 | OA 邮箱必须与邀请邮箱一致 | 新版 OAuth / 激活流程拆分 | 在 SkyOA 接受邀请流程继续校验邮箱一致性 |
| 账号创建 | OA 用户不存在时创建账号，已存在时复用 | Account Activation 与 Session 变化 | 接入 1.17.1 账号创建 / 激活能力 |
| 加入 Workspace | 接受邀请后按邀请角色加入指定 Workspace | 成员接口与事务方式变化 | 使用新版成员接口写入 TenantAccountJoin，并保持原邀请角色 |
| 重复接受 | 已接受邀请不能重复加入 | 新版接口幂等方式变化 | 保留邀请状态检查与幂等处理 |

#### 3.1.4 默认工作空间

**来源：** `origin/feature/20260701` 的默认 Workspace、自动加入和 current workspace 治理。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 默认标记 | Tenant 增加 `is_default` | 官方无公司默认空间语义 | 保留 `is_default` 字段与唯一默认 Workspace 约束 |
| 新账号归属 | SkyOA 新账号默认加入公司 Workspace | 新版注册默认空间处理变化 | 禁止创建无意义个人空间，加入公司默认 Workspace |
| 无空间账号 | 登录时发现没有 Workspace 会自动加入默认空间 | 登录和成员接口变化 | 在新版登录完成后调用统一默认 Workspace 服务 |
| current workspace | 保证账号只有一个有效 current Workspace | 新版 Session 与 Workspace 上下文变化 | 保留“已有有效 current → 普通有效空间 → 默认空间”的选择顺序 |
| 归档兼容 | current 不能落在 archived Workspace | 新版归档状态处理变化 | current 修复时过滤 archived Workspace |
| 管理员关系 | 默认 Workspace owner 参与公司管理员判断 | 官方 owner 只表示 Workspace 角色 | 保留公司管理员判断，但与阶段三执行调度权限解耦 |
| 数据库升级 | 历史 Tenant 需要补充默认标记 | 官方无对应字段 | 在公司 migration 中补字段、索引和默认空间初始化 |

#### 3.1.5 工作空间管理

**来源：** `origin/feature/20260701` 的 Workspace 创建、列表、切换和归档，以及 `origin/feature/20260825_S3` 的指定 owner 和筛选能力。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 创建 Workspace | 系统管理员通过公司入口创建 Workspace | 官方没有同一套公司管理入口 | 保留管理入口，底层改用 1.17.1 Tenant / Session 能力 |
| 指定 owner | 创建时可指定已有账号或邮箱作为 owner | 账号查询与 Session 变化 | 使用新版账号查询，保留大小写处理、重复账号校验和 owner 唯一性 |
| 列表查询 | 支持分页、状态和 owner 信息 | 新版 Model / Response 类型变化 | 按新版 ORM / DTO 重新组装返回结构 |
| 条件筛选 | 支持创建时间、owner 等筛选 | Controller 和 Query Service 变化 | 在新版查询层保留公司需要的筛选条件 |
| Workspace 切换 | 切换前检查成员关系与归档状态 | 新版已有 `switch_tenant` 相关校验 | 复用官方基础校验，再维护 company current workspace 规则 |
| Workspace 归档 | 归档后不删除历史数据，并重新选择 current | 官方无公司管理入口 | 保留归档状态与 current 收敛规则 |
| 权限 | 系统管理员、Workspace owner、普通成员分层 | 1.17.1 RBAC 基础变化 | 复用新版 RBAC，仅补充公司 Workspace 管理边界 |
| 页面 | 旧版有公司 Workspace 管理页面 | 1.17.1 页面结构变化 | 在新版 Console 信息架构中重新接入入口，不整页覆盖旧实现 |

#### 3.1.6 通用 SSE 请求

**来源：** `origin/feature/20260825_S3` 混合提交中独立于执行调度的 SSE Header 修复。

| 项目 | 当前能力 / 问题 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 自定义 Header | 请求可追加业务自定义 Header | 新版请求封装发生变化 | 保留非冲突自定义 Header |
| 系统 Header | 认证、CSRF、分享身份等 Header 不能被覆盖 | 新版请求入口重新组织 | 合并时系统 Header 优先，禁止自定义 Header 覆盖安全字段 |
| GET / POST | 两种 SSE 请求都需正确透传 Header | 新版调用入口不同 | 分别验证 GET / POST 请求 |
| 调度相关逻辑 | 原提交同时包含旧公司调度相关测试或逻辑 | 阶段一不迁执行调度 | 只提取通用 Header 修复，不迁混合提交中的调度部分 |

#### 3.1.7 Redis Event Bus 与 Socket.IO

**来源：** `origin/feature/20260825_S3` 的 Event Bus Sentinel 能力，以及 1.17.1 新增 Socket.IO RedisManager 后产生的 Sentinel 适配。

| 项目 | 当前能力 / 来源 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| Event Bus | Sentinel 环境已有远程 Redis 主连接 | Event Bus 可根据 URL 单独建连接，空地址可能回退本机 | Sentinel 模式直接复用已初始化的远程 Redis Client；非 Sentinel 保留独立 URL |
| Socket.IO | 1.14.2 没有这条 Redis 跨进程连接 | 1.17.1 新增独立 `RedisManager` | 根据 Sentinel 节点、Service Name、DB 和认证构造 `redis+sentinel://` |
| Client 关系 | Event Bus 可复用主 Redis Client | Socket.IO 自己管理 RedisManager | 两者使用同一 Sentinel 基础设施，但保持独立 Client |
| Redis DB | 公司环境使用远程 Sentinel | Socket.IO 默认地址处理不适配公司 Sentinel | 显式带入目标 DB 和 Sentinel 配置 |
| Channel | Redis Pub/Sub 不按逻辑 DB 隔离 | 新旧版本同时运行时可能互相收到消息 | 使用环境前缀隔离 Event Bus / Socket.IO Channel |
| 非 Sentinel | 普通 Redis 地址可直接连接 | 1.17.1 原逻辑可用 | 保留官方非 Sentinel 地址处理 |

#### 3.1.8 S3 / KMS

**来源：** `origin/feature/20260825_S3` 中最终有效的 KMS Provider、S3 Client 与凭据刷新能力。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| KMS Provider | 通过公司 KMS 获取并解密 S3 临时凭据 | 官方没有公司 KMS 协议 | 在 1.17.1 Storage 配置中增加公司 KMS Provider |
| KMS 协议 | 使用最终 `data/signature` 请求协议 | 旧分支历史上存在中间方案 | 只迁最终有效协议，不恢复中间实现 |
| 凭据来源 | IAM、KMS、静态 AK/SK 多种方式 | 官方 S3 Client 已支持 IAM / 静态凭据 | 保留官方分流，在 KMS 配置完整时增加 KMS 分支 |
| 定时刷新 | 凭据到期前后台刷新 | 官方无公司刷新线程 | 保留每日 / 周期刷新能力 |
| 刷新失败 | 刷新失败时继续使用旧 Client 并稍后重试 | 官方无该公司逻辑 | 保留旧 Client，延迟重试，不立即中断运行 |
| 多进程 | fork 后需要重建刷新线程和锁 | Worker 仍可能多进程 | 保留 fork 后初始化逻辑 |
| S3 请求失败 | KMS 模式下凭据可能失效 | 官方只执行正常请求 | 认证类失败时刷新 Client 后受控重试一次 |
| 预签名 / 流式读取 | 旧公司版本缺少新版能力 | 1.17.1 已提供 | 不覆盖官方实现，只让其使用当前有效 S3 Client |

#### 3.1.9 文件路径

**来源：** `origin/feature/20260825_S3` 中对象前缀、应用目录和文件上传链路改造。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 对象前缀 | 对象存储统一增加 `Dify/` 前缀 | 官方直接使用业务 Key | 在对象存储公共层统一增加前缀，已有前缀时不重复添加 |
| 本地存储 | Local / Volume Storage 不需要对象前缀 | Storage Provider 结构变化 | 前缀只作用于对象存储实现 |
| Tenant 目录 | 文件按 Tenant 组织 | 新版 FileService 路径变化 | 保留 Tenant 目录层级 |
| App 目录 | 部分文件需要按 App 进一步隔离 | 新版 FileService 默认没有公司 `app_id` 参数 | 增加必要的可选 `app_id`，并校验应用归属 |
| Console 上传 | 已有鉴权后的 App 上下文 | Controller 参数变化 | 从已鉴权上下文传递真实 App ID |
| Service API / WebApp | 文件上传入口不同 | 新版入口重新组织 | 分别补齐 App ID 透传 |
| Workflow 文件 | 节点结果、草稿变量会写文件 | 新版 Workflow 文件链路变化 | 在需要落对象存储的路径继续传递真实 App ID |
| 历史 Key | 数据库中已经保存历史 `UploadFile.key` | 新目录规则与历史不同 | 读取兼容历史 Key，不在阶段一全量搬迁对象 |

#### 3.1.10 租户私钥

**来源：** `origin/feature/20260825_S3` 中 Tenant RSA 私钥路径与缓存能力。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 私钥路径 | 使用 `Dify/RSA_privateKey/{tenant_id}/private.pem` | 官方 Key Provider 结构变化 | 新 Workspace 继续记录公司私钥路径 |
| 数据库引用 | Tenant 侧保存私钥对象路径 | 新版读取接口变化 | 读取优先使用数据库记录，不依赖重新推导路径 |
| 私钥缓存 | 公司按 Tenant 使用独立 Redis Cache | 新版缓存 Key 实现变化 | 保留最终公司缓存语义，只作用于私钥缓存 |
| 新 Workspace | 创建空间时生成并保存私钥 | Workspace 创建服务变化 | 在统一 Workspace 创建流程中调用新版 Key 能力并记录真实路径 |
| 创建失败 | Workspace 创建失败可能留下孤儿私钥 | 新版事务和 Storage 操作分离 | 创建失败时只删除本次新建的私钥对象 |
| 历史私钥 | 旧对象实际位置可能与新规则不同 | migration 修改路径不会自动搬对象 | 先核对数据库引用和真实对象位置，再决定是否回填或迁移 |
| 兼容原则 | 已有 Tenant 私钥必须继续可用 | 1.17.1 不认识公司历史路径 | 不重新生成已有 Tenant 私钥，优先做路径兼容 |

### 3.2 独立 1.17.1 基线与组件部署

阶段一在独立的 1.17.1 升级线上开发和部署，当前 1.14.2 环境保持运行。代码、流水线、部署和数据空间分别隔离。

#### 3.2.1 分支与流水线

| 对象 | 当前 1.14.2 | 独立 1.17.1 升级线 | 阶段一处理 |
| --- | --- | --- | --- |
| 基线分支 | `release/1.14.2` | `release/1.17.1` | 1.17.1 官方基线独立保留 |
| 迁移开发分支 | 当前 1.14.2 开发线 | `feature/20260920_1.17.1_UPGRADE` | 现有能力迁移统一在独立 1.17.1 分支开发 |
| 测试发布分支 | `test` | `test_1.17.1_UPGRADE` | 两套测试发布分支互不覆盖 |
| 流水线 | 当前 1.14.2 test 流水线 | 独立 1.17.1 流水线 | 分别绑定对应分支、Commit 和制品 |
| 部署 | 当前 test 部署 | `test-upgrade-1.17.1` | 在现有应用下增加隔离部署，当前 test 保持不动 |

#### 3.2.2 基础运行组件

| 组件 | 阶段一处理 |
| --- | --- |
| API | 现有 `dify-api` 应用增加 1.17.1 upgrade 部署 |
| General Worker | 现有 `dify-worker` 应用增加 1.17.1 upgrade 部署，只运行官方 Worker |
| Beat | 现有 `dify-worker-beat` 应用增加 1.17.1 upgrade 部署 |
| Web | 现有 `dify-web` 应用增加 1.17.1 upgrade 部署 |
| Plugin Daemon | 沿用现有应用，部署 1.17.1 对应版本 |
| Sandbox | 沿用现有应用，部署 1.17.1 对应版本 |
| Agent Runtime 组件 | 阶段一不新增，放到阶段二按新功能决定 |
| 公司 Scheduler / Standard / Critical Worker | 阶段一不部署，放到阶段三处理 |

#### 3.2.3 数据与中间件隔离

| 资源 | 阶段一处理 |
| --- | --- |
| PostgreSQL | 1.17.1 使用独立数据库；当前本地为 `ai_studio_1171_dev` |
| Redis Cache | 与 1.14.2 隔离；当前本地使用 DB 2 和 `dify_1171_dev` 前缀 |
| Celery Broker / Result | 与旧 Worker 隔离；当前本地使用 DB 3 和相同环境前缀 |
| Redis Pub/Sub | 使用环境前缀隔离频道，不能只依赖 Redis DB |
| Event Bus | Sentinel 模式复用远程 Redis Client |
| Socket.IO | 独立 RedisManager 连接同一套 Sentinel，并使用独立 Channel 前缀 |
| S3 / KMS | 复用公司基础设施，按 1.17.1 路径和凭据逻辑隔离新写入 |

### 3.3 阶段一完成后的数据迁移

阶段一功能迁移完成后，将 1.14.2 数据迁入独立的 1.17.1 数据空间，阶段二和阶段三继续基于这套 1.17.1 环境开发。此时不覆盖原有 1.14.2 数据和部署。

| 数据类型 | 处理方式 |
| --- | --- |
| PostgreSQL | 从 1.14.2 数据副本迁入独立 1.17.1 数据库，并执行 1.17.1 官方 Migration 与阶段一保留的公司 Migration |
| 账号 / Workspace | 保留原 account_id、Tenant、成员关系、默认空间和归档状态 |
| 调度历史表 | 暂时保留，不因阶段一未启用调度代码而删除；最终由阶段三决定 |
| Redis Cache | 不迁旧缓存，1.17.1 使用独立空命名空间 |
| Celery Broker / Result | 不迁旧队列和执行结果，避免跨版本消费 |
| S3 文件 | 默认不全量搬迁，通过历史 Key 兼容读取；新写入使用 1.17.1 规则 |
| 租户私钥 | 保留原私钥对象与数据库引用，不重新生成 |

---

## 四、阶段二：1.17.1 新能力升级

阶段二直接在阶段一形成的 1.17.1 分支、流水线、部署和数据上继续开发，不再创建第二套 1.17.1 基线。重点是确定每项新能力是否需要新增运行组件，以及在公司 DevOps 中以“新应用”还是“现有应用新增部署”的方式承载。

### 4.1 新能力与组件增量

| 新能力 | 复用阶段一组件 | 需要新增的 1.17.1 运行组件 | DevOps 处理 | 当前结论 |
| --- | --- | --- | --- | --- |
| Workflow / Chatflow 新能力、Human Input 增强、WebApp 增强 | API / Web / Worker / Beat | 无固定新增组件 | 直接在现有 1.17.1 部署上增加配置或前后端能力 | 可直接基于阶段一升级 |
| 新 Agent App / Agent Skills / Agent Home Snapshot | API / Web / Plugin Daemon | `agent_backend` | 倾向新增独立应用 `dify-agent-backend`、独立流水线和部署 | 需要确定是否启用 |
| Agent 本地 Sandbox / Shell 工作区 | `agent_backend` | `local_sandbox` | 若选择 Local Runtime，新增 `dify-agent-local-sandbox` 应用或等价独立运行组件 | 待讨论 |
| Agent Sandbox 出网隔离 | `local_sandbox` / API | `agent_ssrf_proxy` | 新增独立代理组件，或复用公司已有等价网络代理能力 | 待讨论 |
| Workflow 实时协作 | API / Web / Redis | `api_websocket` 运行角色 | 官方使用同一 API 镜像的独立 WebSocket 进程；公司侧需要决定放在 `dify-api` 下新增部署还是单独应用 | 待讨论 |
| Unified / Knowledge Tracing | API / Worker | 取决于公司现有 Trace 后端 | 优先复用现有可观测基础设施，不默认新增 Dify 应用 | 待讨论 |
| 知识库 / 检索增强 | Worker / Plugin Daemon / 向量库 | 取决于具体数据源和 Vector Store | 只为实际启用的数据源增加依赖 | 按功能选择 |
| difyctl、多模态文件、Tool 新参数 | 现有组件 | 无长期运行组件 | 不新增应用 | 可按需求启用 |

### 4.2 组件部署待讨论项

| 讨论项 | 需要确定的内容 |
| --- | --- |
| Agent Runtime 后端 | 使用 Local Sandbox 还是其他 Runtime；决定是否需要 `local_sandbox` 和对应网络代理 |
| Agent Backend 部署 | `agent_backend` 是否作为新的 DevOps 应用；对应流水线、Secret、Redis 和内部 API 地址如何管理 |
| API WebSocket | 作为 `dify-api` 下的新部署，还是独立 `dify-api-websocket` 应用 |
| Agent SSRF Proxy | 使用官方独立代理组件，还是复用公司已有网络代理能力 |
| Redis | Agent Backend 使用的 Redis DB / Prefix 是否继续复用现有 Sentinel，并如何与 API / Celery 隔离 |
| 新组件流水线 | 每个新增应用是否需要独立代码构建，还是直接使用官方/统一镜像制品 |
| 最终组件数量 | 只有实际启用能力需要的组件才进入最终架构，未启用能力不提前申请应用 |

---

## 五、阶段三：执行调度能力处理

阶段三基于完成阶段二后的 1.17.1 版本处理现有公司执行调度能力。每项能力最终只保留一种结果：继续迁移、按 1.17.1 重写，或取消并使用官方能力。

### 5.1 功能范围

| 能力方向 | 当前能力 | 1.17.1 变化 | 处理方向 |
| --- | --- | --- | --- |
| 执行准入 | Policy、Admission、Priority、容量限制 | 应用入口和异步执行参数变化 | 迁移 / 外围重写 / 取消 |
| 调度中心 | Job、Scheduler、Lease、Outbox、Generation | 任务状态与官方执行标识、Worker 生命周期变化 | 迁移 / 重写 / 缩减 / 取消 |
| 专用 Worker | Standard / Critical Worker | 旧 Worker 直接调用旧 Runtime | 迁移 / 基于官方执行重写 / 取消 |
| 正式应用托管 | Workflow、Chatflow、Chat、Completion、Agent | 各应用执行入口、Session、消息和结果管理变化 | 继续托管 / 直接接官方 |
| Console / Human Input / Schedule | 草稿调试、暂停恢复、定时触发 | 1.17.1 已提供新的官方执行链路 | 继续治理 / 使用官方 |
| 任务中心 | Query、Cancel、Stop、Retry、Streaming / Blocking Result | 原任务中心依赖公司 Job | 保留 / 重做 / 取消 |
| 监控与审计 | Worker / Scheduler Health、OTel、执行审计 | 指标主体取决于最终调度架构 | 随最终方案保留或重做 |

### 5.2 对最终组件的影响

| 最终方向 | 组件结果 |
| --- | --- |
| 完全使用 1.17.1 官方执行 | 不再部署公司 Scheduler / Standard / Critical Worker；任务中心和调度管理同步缩减或下线 |
| 保留外围治理 | 保留轻量 Policy / Job / Admission 能力，官方 Worker 继续负责实际执行 |
| 继续托管执行 | 需要重新接入 1.17.1 Runtime，并保留或重写 Scheduler、Job、Standard / Critical Worker |
| 部分能力保留 | 按功能拆分最终组件，只保留仍有业务价值的调度服务和管理入口 |

阶段三结束后形成最终的代码范围、数据库范围、队列范围和 DevOps 组件清单，随后进入最终收束。

---

## 六、最终收束

最终收束不再长期保留 `test-upgrade-1.17.1` 这套平行环境，而是将三个阶段形成的最终 1.17.1 版本覆盖到原有 DevOps 应用和正式数据链路。

### 6.1 原有组件收束

| 对象 | 最终处理 |
| --- | --- |
| `dify-api` | 原应用切换为最终 1.17.1 API 制品和配置 |
| `dify-worker` | 原应用切换为最终 1.17.1 Worker 制品和队列配置 |
| `dify-worker-beat` | 原应用切换为最终 1.17.1 Beat |
| `dify-web` | 原应用切换为最终 1.17.1 Web |
| `dify-plugin-daemon` | 切换到最终 1.17.1 对应版本 |
| `dify-sandbox` | 切换到最终 1.17.1 对应版本 |
| 阶段二新增组件 | 只保留已确定启用能力需要的 Agent Backend / Local Sandbox / WebSocket / Proxy 等组件 |
| 阶段三调度组件 | 按阶段三最终方案保留、重写或下线 |
| 临时升级部署 | 原组件切换完成后下线 `test-upgrade-1.17.1` |

### 6.2 分支与流水线收束

| 项目 | 最终处理 |
| --- | --- |
| `release/1.17.1` | 收束为最终 1.17.1 公司基线 |
| `feature/20260920_1.17.1_UPGRADE` | 阶段一到阶段三的最终代码合入 release 后结束长期开发职责 |
| `test_1.17.1_UPGRADE` | 用于最终发布前测试，正式切换后按发布流程维护或清理 |
| 基础组件流水线 | 原有 API / Web / Worker 等流水线改为构建最终 1.17.1 基线 |
| 新增组件流水线 | 仅为阶段二、三最终保留的新应用建立正式流水线 |

### 6.3 最终数据迁移

阶段一后的独立 1.17.1 数据用于阶段二、三持续开发；最终收束时仍以正式 1.14.2 数据的最新快照作为迁移源，再执行一次最终 Migration。

| 数据类型 | 最终处理 |
| --- | --- |
| PostgreSQL | 备份正式 1.14.2 数据后，执行最终 1.17.1 官方 Migration + 三阶段最终保留的公司 Migration |
| 账号 / Workspace | 保留账号 ID、成员关系、默认空间和归档状态 |
| 调度数据 | 根据阶段三最终结论继续使用、只读保留或停止新增 |
| Redis Cache | 不搬迁旧缓存，最终 1.17.1 使用新的命名空间 |
| Celery Broker / Result | 不迁旧队列和 Result，切换前停止旧任务入口后从新队列启动 |
| S3 | 优先保持对象不动并兼容历史 Key；只有最终路径方案明确需要时才搬对象 |
| 租户私钥 | 保留既有对象和引用，不重新生成历史 Tenant 私钥 |

最终流程为：**独立 1.17.1 升级线完成三个阶段 → 固定最终组件清单 → 收束 release / 流水线 → 迁移正式数据 → 原有 DevOps 组件切换到最终 1.17.1 → 下线平行升级环境。**
