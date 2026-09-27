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
| SkyOA 登录 | Provider、回调、账号匹配和绑定 | 适配新版 Account、Identity、Repository 和 Session | Account / Identity |
| 管理员初始化 | SkyOA 首次安装、初始管理员、失败清理 | 接入新版 Setup 流程和事务模型 | Setup / Account / Workspace |
| 邀请注册 | 数据库邀请记录、SkyOA 接受邀请 | 适配新版 Account Activation 和成员接口 | Account / Workspace |
| 默认工作空间 | 唯一默认空间、新用户加入、current workspace | 适配新版 Tenant、Session 和成员关系 | Workspace / Tenant |
| 工作空间管理 | 创建、指定 owner、查询、切换、归档和权限 | 复用新版 Tenant、Session 和 RBAC | Workspace |
| 通用 SSE 请求 | 自定义 Header 与认证 Header 合并 | 迁入独立于调度的通用 Header 修复 | Web Request / SSE |
| S3 / KMS | KMS 凭据、刷新、S3 Client | 将公司 KMS Provider 接入 1.17.1 Storage | Storage Provider |
| 存储路径与租户私钥 | 对象前缀、应用目录、历史文件、RSA 私钥 | 适配新版 Storage / FileService，并兼容历史引用 | Storage / Tenant |

### 3.1 功能迁移

#### 3.1.1 SkyOA 登录

**来源：** `origin/feature/20260701`；账号邮箱匹配与资料补充来自 `origin/feature/20260825_S3`。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 登录入口 | SkyOA 与官方 Provider 并存 | 官方 OAuth 登录由新版 AccountOAuthService 等服务承接 | 保留官方 Provider 流程，只增加 SkyOA 独立分支 |
| POST 回调 | 前端提交 `code`、`state`、`currentUrl` | 官方 Provider 主要使用 GET 回调 | 保留 SkyOA POST 回调和原登录结果 |
| state / nonce | 使用短期 Cookie 校验本次登录 | 官方有独立 state 处理 | SkyOA 保留原 state / nonce 与 300 秒 Cookie 生命周期 |
| 账号查找 | 先查 OA 绑定，未命中再按邮箱匹配 | 新版身份查询转为 Repository，账号增加 `normalized_email` | 使用新版 Repository / Session，保留“绑定优先、邮箱兜底、重复邮箱拒绝”规则 |
| OA 身份绑定 | 保存账号后关联 openId | 新版改用 Repository.link 等接口 | 保留原 account_id 和保存顺序，改用新版绑定接口 |
| 待激活账号资料 | 只对待激活账号更新姓名、语言、时区 | 官方激活流程不包含公司资料补充 | 保留原更新条件，使用新版 Session 保存 |
| Provider 展示 | 登录页和账号页按已配置 Provider 展示 | 新版绑定查询由 AccountIntegrationService 提供 | 将 SkyOA 加入 Provider 名单，登录页按名单展示；在新版 `/account` 页面恢复绑定状态 |
| 配置 | Client ID、Secret、授权、Token、userinfo、scope 六项 | 1.17.1 无 SkyOA 配置 | 按新版配置结构迁入六项 SkyOA 参数 |
| 日志 | 已有身份摘要脱敏，但失败日志可能带敏感信息 | 官方没有 SkyOA 请求分支 | 保留摘要脱敏，并限制失败日志只记录异常类型和 HTTP 状态码 |

#### 3.1.2 管理员初始化

**来源：** `origin/feature/20260701`；失败清理修复来自 `origin/feature/20260825_S3`。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 初始化入口 | 启用 SkyOA 时通过 OA 身份创建初始管理员 | Setup 初始化流程变化 | 在新版 Setup 中增加 SkyOA 初始化分支；未启用时保持官方流程 |
| 初始化检查 | 仅未初始化环境允许执行 | 新版已有安装状态检查 | 复用 1.17.1 初始化状态判断 |
| 管理员账号 | 初始化时创建公司管理员账号 | Account 创建与 Session 方式变化 | 使用新版账号接口和 Session |
| 初始 Workspace | 初始化管理员同时创建 Workspace | Workspace 创建流程变化 | 调用统一 Workspace 创建能力，不在 Setup 中重复实现 |
| 初始化后登录 | 创建完成后直接进入系统 | 登录返回和 Session 结构变化 | 按 1.17.1 登录态重新适配 |
| 失败清理 | 初始化失败时清理本次创建内容 | 新版事务边界变化 | 先回滚，再只清理由本次初始化创建的账号、Workspace 和 Identity |

#### 3.1.3 邀请注册

**来源：** `origin/feature/20260701`。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 邀请记录 | 数据库存储邀请、状态、有效期和 Token | 官方邀请与激活流程变化 | 保留公司邀请模型和生命周期 |
| 邀请登录 | 邀请场景只允许 SkyOA | 官方激活页支持多种登录方式 | 邀请流程继续只展示 SkyOA |
| 邮箱校验 | OA 邮箱必须与邀请邮箱一致 | OAuth 与激活流程拆分 | 在 SkyOA 接受邀请流程继续校验邮箱一致性 |
| 账号处理 | 不存在则创建，存在则复用 | Account Activation 与 Session 变化 | 使用 1.17.1 账号创建 / 激活能力 |
| 加入 Workspace | 按邀请角色加入目标 Workspace | 成员接口与事务方式变化 | 使用新版成员接口保留原角色和 account_id |
| 重复接受 | 已接受邀请不能重复加入 | 新版幂等方式变化 | 继续由邀请状态控制重复接受 |

#### 3.1.4 默认工作空间

**来源：** `origin/feature/20260701`。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 默认标记 | Tenant 增加 `is_default` | 官方无公司默认 Workspace 语义 | 保留字段和唯一默认 Workspace 约束 |
| SkyOA 新账号 | 新账号加入默认 Workspace | 新版注册与个人空间创建方式变化 | SkyOA 新账号不创建无意义个人空间，直接加入默认 Workspace |
| 无空间账号 | 登录后自动加入默认 Workspace | 登录和成员接口变化 | 登录完成后调用统一默认 Workspace 服务 |
| current workspace | 维护唯一有效 current | 新版依赖 Session 与 Workspace 上下文 | 保留“有效 current → 其他有效空间 → 默认空间”的选择顺序 |
| 归档兼容 | current 不允许落到 archived Workspace | 新版归档状态处理变化 | current 修复时过滤 archived Workspace |
| 管理员判断 | 默认 Workspace owner 参与公司管理员判断 | 官方 owner 仅表示 Workspace 角色 | 保留公司管理员判断，但不带入阶段三调度权限 |
| 旧库字段 | 历史 Tenant 需要新增默认标记 | 官方无对应 migration | 公司 Migration 中补字段、索引和默认空间初始化 |

#### 3.1.5 工作空间管理

**来源：** `origin/feature/20260701`；指定 owner 与筛选增强来自 `origin/feature/20260825_S3`。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 创建 Workspace | 系统管理员通过公司入口创建 | 官方没有同一套公司管理入口 | 保留公司入口，底层使用新版 Tenant / Session |
| 指定 owner | 可按已有账号或邮箱指定 owner | 账号查询与 Session 变化 | 复用新版账号查询，保留大小写、重复账号和 owner 校验 |
| 查询与筛选 | 分页、创建时间、owner 等条件 | Model / Controller / Query Service 变化 | 在新版查询层保留公司需要的筛选条件 |
| 切换 Workspace | 校验成员关系和归档状态 | 新版已有基础切换校验 | 复用官方校验，再维护 current workspace 规则 |
| 归档 Workspace | 归档后保留数据并重新选择 current | 官方无公司管理入口 | 保留归档状态和 current 收敛规则 |
| 权限 | 系统管理员、owner、普通成员分层 | 1.17.1 RBAC 基础变化 | 复用新版 RBAC，仅补公司 Workspace 管理边界 |
| 管理页面 | 旧版有公司 Workspace 管理页面 | 新版 Console 页面结构变化 | 页面入口位置待确认，不整页覆盖旧实现 |

#### 3.1.6 通用 SSE 请求

**来源：** `origin/feature/20260825_S3` 的通用请求头修复。当前方案已确定，但尚未实施。

| 项目 | 当前能力 / 问题 | 1.17.1 情况 | 迁移方案 |
| --- | --- | --- | --- |
| Header 合并 | 自定义 Header 不能覆盖认证、CSRF、分享身份等系统 Header | `fetchOptions.headers` 仍可能覆盖先设置的系统 Header | 使用旧修复的合并规则，系统 Header 优先，保留不冲突的自定义 Header |
| GET / POST | 两类 SSE 请求都需要保留 Header | 新版请求封装位置变化 | 在新版请求层统一处理 GET / POST |
| 调度依赖 | 原测试场景中混有公司调度路径 | 阶段一不迁执行调度 | 只迁通用 Header 逻辑，测试改为普通 SSE 请求 |

#### 3.1.7 S3 / KMS

**来源：** `origin/feature/20260825_S3` 中最终有效的 KMS Provider、S3 Client 与凭据刷新能力。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| KMS Provider | 通过公司 KMS 获取并解密 S3 凭据 | 官方无公司 KMS 协议 | 在 1.17.1 Storage 中增加公司 KMS Provider |
| KMS 协议 | 最终使用 `data/signature` 协议 | 旧分支存在过中间协议 | 只迁最终有效协议，不恢复 cp/r、本地 Token 等中间方案 |
| 凭据来源 | IAM、KMS、静态凭据分流 | 官方已有 IAM / 静态凭据 | 保留官方分流，在 KMS 配置完整时增加 KMS 分支 |
| 定时刷新 | 按时区刷新临时凭据 | 官方无公司刷新线程 | 保留刷新机制和刷新时区配置 |
| 刷新失败 | 保留旧 Client，稍后重试 | 官方无该公司逻辑 | 失败时继续使用旧 Client，并延迟重试 |
| 多进程 | fork 后需要重建刷新状态 | Worker 仍可能多进程 | 保留 fork 后锁、线程和状态重建 |
| S3 失败重试 | 凭据类错误后刷新 Client 再重试一次 | 官方无该公司规则 | 保留一次受控重试 |
| exists | 只有对象不存在返回 false | 官方错误处理不同 | 保留公司错误区分，其他异常继续抛出 |
| 预签名 / 流式读取 | 旧公司基线缺少新版实现 | 1.17.1 已支持 | 保留官方实现，只切换到当前有效 S3 Client |

#### 3.1.8 存储路径与租户私钥

**来源：** `origin/feature/20260825_S3` 中对象前缀、应用目录、Tenant 私钥路径和缓存能力。

| 项目 | 当前能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 对象前缀 | 对象存储统一增加 `Dify/` | 官方公共 Storage 直接使用业务 Key | 在对象存储公共层统一加前缀；已有前缀不重复添加，本地 / Volume Storage 不加 |
| 预签名路径 | 旧前缀处理未覆盖新版签名入口 | 1.17.1 增加公共预签名能力 | 签名前应用同一前缀规则，保留官方有效期和内容类型参数 |
| Tenant / App 目录 | 上传 Key 包含 Tenant、可选 App 和文件 UUID | 新版 FileService 没有公司 `app_id` 参数 | 在新版 FileService 增加可选 `app_id` 并校验 UUID 和租户归属 |
| 上传入口 | Console、Service API、WebApp、Workflow 都需传 App 身份 | 新版调用链重新组织 | 从已鉴权上下文透传真实 App ID，不信任请求自报其他应用身份 |
| 历史文件 | 数据库保存旧 `UploadFile.key` | 新路径规则不同 | 保持历史 Key 可读，不在阶段一全量搬迁对象 |
| 私钥路径 | `Dify/RSA_privateKey/{tenant_id}/private.pem` | 官方默认路径和 Key Provider 变化 | 新 Workspace 继续记录公司私钥路径 |
| 私钥引用 | Tenant 保存私钥对象路径 | 新版读取接口变化 | 读取优先使用数据库记录，不依赖重新拼路径 |
| 私钥缓存 | 公司按 Tenant 使用固定缓存名 | 新版使用不同缓存 Key | 保留公司最终缓存语义，只作用于私钥缓存 |
| 创建与清理 | 创建 Workspace 时生成私钥，失败删除本次对象 | 新版事务与 Storage 操作分离 | 在统一 Workspace 创建流程记录私钥路径；失败时只清理本次新对象 |
| 历史私钥 | 旧对象实际位置仍需核对 | Migration 写路径不会搬对象 | **待确认：** 根据实际对象位置决定回填或迁移，不重新生成旧 Tenant 私钥 |
| 旧库 Migration 链 | 私钥 Migration 依赖旧公司历史 revision | 官方 1.17.1 不包含该链 | **待确认：** 根据旧库实际 `alembic_version` 和表结构设计合法接续；不删除历史父 revision，也不通过空脚本或 stamp 跳过 |

### 3.2 独立 1.17.1 基线与组件部署

#### 3.2.1 分支、流水线与 DevOps 部署

| 对象 | 当前 1.14.2 | 独立 1.17.1 升级线 | 阶段一处理 |
| --- | --- | --- | --- |
| 基线分支 | `release/1.14.2` | `release/1.17.1` | 1.17.1 官方基线独立保留 |
| 迁移开发分支 | 当前 1.14.2 开发线 | `feature/20260920_1.17.1_UPGRADE` | 阶段一迁移统一在独立 1.17.1 分支开发 |
| 测试发布分支 | `test` | `test_1.17.1_UPGRADE` | 两套测试发布分支互不覆盖 |
| 流水线 | 当前 1.14.2 test 流水线 | 1.17.1 独立流水线 | 分别绑定对应分支、Commit 和制品 |
| 部署 | 当前 test 部署 | `test-upgrade-1.17.1` | 在现有应用下新增隔离部署，当前 test 保持不动 |

#### 3.2.2 平台构建与启动

**来源：** `origin/feature/20260624_test_1` 的平台部署支持；1.17.1 冲突按新版行为处理。

| 项目 | 原公司实现 | 1.17.1 情况 | 阶段一处理 |
| --- | --- | --- | --- |
| Web 启动参数 | 使用 npm 启动参数 | 新版改为 pnpm | 使用新版 pnpm 参数 |
| Web 监听地址 | 未指定时读取 `SERVER_HOST` | 新版默认读取系统 `HOSTNAME`，容器内可能是容器名 | 保留 `SERVER_HOST`，避免容器名称成为监听地址 |
| Node 版本 | 允许较宽 Node 22 / 24 范围 | 1.17.1 要求更高的 Node 24 版本 | 完全采用 1.17.1 官方版本要求 |
| UUIDv7 Migration | 发现已有 `uuidv7()` 时会提前退出 | 1.17.1 会跳过已有函数并继续创建缺失的 `uuidv7_boundary()` | 使用 1.17.1 官方 Migration 行为 |
| build.sh | 公司平台通过脚本生成制品 | 1.17.1 依赖、目录和前端构建方式变化 | 保留公司制品入口，只适配新版构建依赖和产物 |
| API / Worker / Beat | 共用后端代码 | 1.17.1 继续按运行角色区分 | 共用同一 API 后端制品，通过不同启动角色运行 |
| Web | 独立前端制品 | 1.17.1 前端构建方式变化 | 使用 1.17.1 Web 制品独立发布 |

#### 3.2.3 基础运行组件

| 组件 | 阶段一处理 |
| --- | --- |
| API | 现有 `dify-api` 应用新增 1.17.1 upgrade 部署 |
| General Worker | 现有 `dify-worker` 应用新增 1.17.1 upgrade 部署，只运行 1.17.1 官方 Worker |
| Beat | 现有 `dify-worker-beat` 应用新增 1.17.1 upgrade 部署 |
| Web | 现有 `dify-web` 应用新增 1.17.1 upgrade 部署 |
| Plugin Daemon | 沿用现有应用，部署 1.17.1 对应版本 |
| Sandbox | 沿用现有应用，部署 1.17.1 对应版本 |
| Agent Runtime 组件 | 阶段一不新增，放到阶段二根据新功能决定 |
| 公司 Scheduler / Standard / Critical Worker | 阶段一不部署，放到阶段三处理 |

#### 3.2.4 PostgreSQL、Redis 与 Celery 隔离

1.14.2 与 1.17.1 并行期间可以共用 PostgreSQL / Redis 基础设施，但数据库、Redis DB、Key Prefix 和队列空间必须隔离。

| 资源 | 当前处理 |
| --- | --- |
| PostgreSQL | 1.17.1 使用独立数据库；当前 local 使用 `ai_studio_1171_dev` |
| local PostgreSQL 地址 | 当前已切换到验证可用的 `10.89.70.9:54321` |
| test PostgreSQL 地址 | 保留现有 test 地址，部署网络连通性仍需确认 |
| Redis Cache | 使用 DB 2，并配置 `dify_1171_dev` 前缀 |
| Celery Broker / Result | Sentinel 节点使用完整 URL，统一使用 DB 3 和 `dify_1171_dev` 前缀 |
| Redis Pub/Sub | 不按逻辑 DB 隔离，必须通过 Channel Prefix 区分 1.14.2 与 1.17.1 |
| Plugin Daemon | 继续使用自身独立数据库 / Redis，不与 Dify 主库混用 |

#### 3.2.5 Event Bus 与 Socket.IO Sentinel 适配

| 项目 | 1.14.2 / 公司能力 | 1.17.1 情况 | 阶段一处理 |
| --- | --- | --- | --- |
| Event Bus | Sentinel 环境已有远程 Redis 主 Client | 即使启用 Sentinel，仍可能根据空 Event Bus URL 回退本机 Redis | Sentinel 模式复用已初始化的远程 Redis Client；非 Sentinel 保留显式 Event Bus URL |
| Socket.IO | 旧基线没有该跨进程 Redis 连接 | 1.17.1 新增独立 `RedisManager` | 使用 Sentinel 节点、Service Name、DB 和认证构造 `redis+sentinel://` |
| Client 关系 | Event Bus 可直接复用主 Redis Client | Socket.IO 自己维护 RedisManager | 两者共用 Sentinel 基础设施，但 Socket.IO 不复用 Event Bus Client |
| Channel | 旧环境无前缀 | 1.17.1 发布 / 订阅会使用 `REDIS_KEY_PREFIX` | 1.17.1 使用 `dify_1171_dev` 前缀，与旧频道隔离 |
| 当前状态 | 无 Socket.IO 跨进程能力 | 新版实时协作依赖 Socket.IO | Event Bus Sentinel 复用及 Socket.IO Sentinel 连接已完成本地连接与隔离验证；登录后的工作流实时协作仍未确认 |

### 3.3 阶段一完成后的数据迁移

阶段一功能和独立运行环境完成后，从当前 1.14.2 数据生成迁移副本，在独立 1.17.1 数据空间完成数据升级。阶段二和阶段三继续基于这套 1.17.1 数据开发；原 1.14.2 数据和部署保持不动。

| 数据类型 | 处理方式 |
| --- | --- |
| PostgreSQL | 从 1.14.2 数据副本迁入独立 1.17.1 数据库，再执行 1.17.1 官方 Migration 与阶段一保留的公司 Migration |
| Alembic 历史 | 保留旧公司历史 revision，根据旧库实际 `alembic_version` 和表结构设计接续链 |
| 账号 / Workspace | 保留原 account_id、Tenant、成员关系、默认 Workspace、current 和归档状态 |
| 调度历史表 | 阶段一不启用调度运行代码，但不删除旧表、历史数据和 Migration 记录；最终由阶段三决定 |
| Redis Cache | 不迁旧缓存，1.17.1 使用独立空命名空间 |
| Celery Broker / Result | 不迁旧队列和旧 Result，新旧 Worker 不跨版本消费 |
| S3 文件 | 默认不全量搬迁，通过历史 Key 兼容读取；新写入按 1.17.1 路径规则 |
| 租户私钥 | 保留原私钥对象和数据库引用，不重新生成；历史路径先核对真实对象位置 |

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
