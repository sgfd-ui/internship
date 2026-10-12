# Dify 1.17.1 基线升级方案

## 一、升级目标

当前平台基于 Dify 1.14.2，并扩展了 SkyOA、工作空间治理、S3/KMS、文件与租户私钥、托管执行调度等公司能力。本次升级以 Dify 1.17.1 为新基线，分三个阶段完成现有能力迁移、新版本能力升级和执行引擎隔离，最终以本地迁移演练为前提升级线上原数据库，并仅收束 PostgreSQL 与 Redis 数据资源；既有 1.17.1 应用和 DevOps 部署不在最终收束范围内。

| 目标 | 内容 |
| --- | --- |
| 基线升级 | 平台基础版本由 Dify 1.14.2 升级到 Dify 1.17.1 |
| 现有能力迁移 | 将 SkyOA、工作空间、S3/KMS、文件与私钥等现有公司能力适配到 1.17.1 |
| 新能力升级 | 在阶段一形成的 1.17.1 版本上增加确定要启用的新功能及对应运行组件 |
| 执行引擎隔离 | 不迁原公司托管调度；以 1.17.1 官方 Workflow / Chatflow 异步执行为基础，增加应用执行池、Queue Router 和独立 Priority Worker |
| 数据库升级 | 从线上原测试库 `ai_studio` 获取最新备份做本地演练，完成兼容迁移后在线上原库就地升级到 1.17.1，不以本地开发库覆盖线上库 |
| 最终收束 | 仅整理 PostgreSQL、Redis 的数据连接与运行命名空间；不合并或替换原 DevOps 应用、流水线及 Worker 部署 |

---

## 二、升级架构

### 2.1 当前架构与 Dify 1.17.1 官方架构

![当前架构 vs Dify 1.17.1 官方架构](assets/dify-baseline-upgrade/current-vs-dify-1.17.1-official-v11.svg)

### 2.2 数据迁移与最终收束路径

![Dify 1.17.1 本地验证、线上原库升级与数据库 Redis 收束](assets/dify-baseline-upgrade/dify-1.17.1-local-to-online-db-redis-v17.svg)

---

## 三、阶段一：现有能力迁移

### 3.1 功能迁移

业务功能按依赖顺序迁移：**S3/KMS → 文件路径与密钥公共操作 → 默认工作空间与自动注册 → 工作空间管理 → 邀请注册 → 管理员初始化**。SkyOA 登录和通用 SSE 修复可独立迁移。各功能新增的表、字段和索引直接接在 1.17.1 当前 Migration Head 之后，旧公司 Migration 只作为结构和数据修复规则来源，不直接复制旧父版本或调度链。

| 能力 | 主要迁移内容 | 1.17.1 适配位置 |
| --- | --- | --- |
| SkyOA 登录 | Provider、POST 回调、state/nonce、账号匹配、OA 绑定、Provider 展示与日志脱敏 | Account / Identity / OAuth / AccountIntegrationService |
| S3 / KMS | 公司 KMS 协议、凭据选择与刷新、S3 Client、失败重试 | Storage Provider / AWS S3 Storage |
| 文件路径与租户私钥 | 对象前缀、Tenant/App 文件目录、app_id 透传、私钥路径与缓存 | Storage / FileService / RSA Provider |
| 默认工作空间与自动注册 | `is_default`、可信 SkyOA 建号、默认空间加入、current workspace 恢复 | AccountService / Tenant / CurrentTenantResolver |
| 工作空间管理 | 创建、指定 owner、幂等、额度、切换、归档、成员治理和管理页面 | Workspace Service / RBAC / WorkspaceCard |
| 邀请注册 | 数据库邀请、SkyOA 接受邀请、邮箱校验、成员加入 | Account / Workspace / Invitation |
| 管理员初始化 | SkyOA 首次安装、管理员和默认空间创建、失败清理 | Setup / Workspace Provisioning |
| 通用 SSE | 系统 Header 与自定义 Header 合并 | Web Request / SSE |
| Workflow 请求参数 | 会话 ID、父消息 ID 去空格，空白值转为 None 后再做 UUID 校验 | Workflow Console Request |

#### 3.1.1 SkyOA 登录

**来源：** `origin/feature/20260701`；账号邮箱匹配与资料补充来自 `origin/feature/20260825_S3`。

| 项目 | 1.14.2 公司能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| Provider 与登录入口 | SkyOA 与官方 Provider 并存 | 官方 OAuth 登录改由新版服务承接 | 保留官方 Provider，增加独立 SkyOA 分支 |
| 回调协议 | 前端 POST `code`、`state`、`currentUrl` | 官方 Provider 采用自身回调流程 | 保留 SkyOA POST 回调，官方回调不改 |
| state / nonce | 300 秒 Cookie 保存登录上下文 | 官方有独立 state 处理 | SkyOA 继续使用原 state / nonce 与 Cookie 生命周期 |
| Token / userinfo | 使用 SkyOA Token 和 userinfo 协议解析身份 | 官方无 SkyOA Provider | 保留公司请求、身份解析及离职/停用等校验 |
| 账号匹配 | 先查 OA 绑定，未命中再按邮箱匹配 | 新版使用 Repository、显式 Session 和 `normalized_email` | 保留“绑定优先、邮箱兜底、重复邮箱拒绝”，改用新版 Repository / Session |
| OA 绑定 | 登录后将 openId 关联原账号 | 身份关联接口变化 | 使用新版 Repository.link，保留原 account_id |
| Provider 展示 | 登录页和账号页按已配置 Provider 展示 | 新版绑定列表由 AccountIntegrationService 提供 | 将 SkyOA 加入 Provider 名单，在登录页和 `/account` 展示 |
| 配置 | Client ID、Secret、授权、Token、userinfo、scope | 官方无 SkyOA 配置 | 按 1.17.1 配置结构增加六项参数 |
| 日志 | 公司摘要脱敏 | HTTP 客户端日志可能打印 URL 中的敏感参数 | 在控制台和文件日志输出前脱敏 SkyOA Token URL 中的 Secret 和授权码 |

SkyOA 新用户建号和默认空间加入统一由 3.1.4 处理，不在登录逻辑中重复实现注册流程。

#### 3.1.2 S3 / KMS

**来源：** `origin/feature/20260825_S3` 的最终 KMS / S3 实现。

| 项目 | 1.14.2 公司能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| KMS Provider | 公司 KMS 获取临时 S3 凭据 | 官方无公司 KMS 协议 | 在新版 Storage 增加公司 KMS Credential Provider |
| KMS 协议 | 最终使用 `data/signature` | 历史分支存在多版中间协议 | 只迁最终 `data/signature` 协议 |
| 凭据选择 | IAM、KMS、静态 AK/SK 分流 | 官方支持 IAM 与静态凭据 | IAM 优先；KMS 配置完整时使用 KMS；全空时使用静态凭据 |
| 定时刷新 | 临时凭据按时区周期刷新 | 官方无公司刷新线程 | 保留刷新时区、周期检查和文件操作前到期检查 |
| 刷新失败 | 保留旧 Client 并延迟重试 | 官方无该逻辑 | 新凭据成功后再替换 Client，失败时继续使用旧 Client |
| 多进程 | fork 后重新初始化刷新状态 | Worker 仍可能多进程 | 保留进程级锁、线程和状态重建 |
| 请求失败 | 认证类错误刷新 Client 后重试一次 | 官方按普通 S3 错误处理 | KMS 模式下刷新凭据并受控重试一次 |
| exists | 只有对象不存在返回 false | 官方错误路径不同 | 保留不存在与其他异常的区分 |
| 新版能力 | 旧基线缺少部分预签名和流式能力 | 1.17.1 已提供 | 保留官方能力，只让其使用当前有效 S3 Client |
| 环境配置 | 公司测试环境使用 S3 / KMS | 1.17.1 配置项发生变化 | 按新版配置结构维护 endpoint、bucket、region、寻址方式、刷新时区及 KMS 参数 |

#### 3.1.3 文件路径与租户私钥

**来源：** `origin/feature/20260701` 与 `origin/feature/20260825_S3` 的文件路径、RSA 私钥和公共密钥操作。

| 项目 | 1.14.2 公司能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 对象前缀 | 对象存储统一使用 `Dify/` | 官方公共 Storage 直接使用业务 Key | save/load/download/exists/delete/scan/签名统一应用前缀；Local / Volume 不加 |
| 文件目录 | 按 Tenant 和可选 App 组织对象 | 新版 FileService 无公司 `app_id` 参数 | 文件保存为 `Dify/upload_files/{tenant_id}/{app_id?}/{uuid}` |
| App 归属校验 | 传入 app_id 时校验应用属于当前 Tenant | 新版无公司校验 | FileService 增加可选 app_id，并校验 UUID 与租户归属 |
| 上传入口 | Console、Service API、WebApp、Workflow 传 App 身份 | 新版上传链路重新组织 | 从已鉴权上下文透传真实 App ID；RemoteFileService 同步增加 app_id |
| Workflow 文件 | 节点结果和草稿结果按应用归档 | 新版仍会将结果落文件 | Repository 和 Draft Variable Service 传入所属 App ID |
| 私钥路径 | `Dify/RSA_privateKey/{tenant_id}/private.pem` | 官方默认私钥路径和 Provider 不同 | 新 Tenant 使用公司路径，并在 Tenant 记录实际私钥引用 |
| 私钥缓存 | 按 Tenant 缓存私钥 | 新版缓存键不同 | 使用公司最终缓存键规则，只影响私钥缓存 |
| 密钥操作 | 创建空间时分步生成、保存和失败清理 | 新版通过 Provider 统一生成 | 在新版 Provider 上保留生成、保存、删除的分步能力，供 Workspace Provisioning 使用 |
| 数据结构 | Tenant 保存私钥路径 | 1.17.1 无公司字段 | 在当前 1.17.1 Migration Head 后增加私钥字段及所需结构 |
| 历史数据 | 旧文件和私钥已有真实对象 | 新路径结构不同 | 阶段一只建立新结构；旧对象和旧私钥引用统一在 3.3 数据导入时处理 |

#### 3.1.4 默认工作空间与自动注册

**来源：** `origin/feature/20260701` 的默认空间、自动注册、current workspace 和默认 owner 保护。

| 项目 | 1.14.2 公司能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 默认空间 | Tenant 增加 `is_default` | 官方无公司默认空间语义 | 在新版 Tenant 增加 `is_default`，建立唯一默认空间约束 |
| SkyOA 自动注册 | 已验证 OA 用户可在普通注册关闭时建号 | 新版普通注册受系统开关限制 | 给注册/建号链路增加可信 OA 参数，仅 SkyOA 验证成功时允许绕过普通注册关闭 |
| 个人空间 | SkyOA 用户不创建无意义个人空间 | 新版普通注册可创建个人空间 | SkyOA 注册后直接加入默认 Workspace |
| 成员加入 | 无空间用户自动加入默认空间 | 新版成员操作使用显式 Session | 使用同一 Session 写入成员关系；已有成员保留角色，新成员使用入口指定角色 |
| current workspace | 有效 current 优先，否则正常空间，最后默认空间 | 新版只维护当前空间上下文 | 引入 CurrentTenantResolver，按公司顺序选择并保存唯一 current |
| 角色恢复 | 切换/恢复后使用对应空间成员角色 | 新版保存空间和角色 | 复用新版角色字段，恢复时同步更新最近使用时间 |
| 默认 owner 保护 | 默认空间唯一 owner 不允许删除、降级或转移 | 官方无公司保护 | 保留 owner 删除、角色变更、转移和归档保护方法 |
| 设置默认空间 | CLI 指定默认 Tenant | 官方无对应命令 | 保留 `set-default-tenant`，校验唯一 owner 和确认邮箱 |
| 数据结构 | 默认标记、唯一 current、查询索引 | 官方缺少公司约束 | 在新环境 Migration 中创建字段、约束和索引，不复制旧 revision 链 |

#### 3.1.5 工作空间管理

**来源：** `origin/feature/20260701` 及 `origin/feature/20260825_S3` 的公司 Workspace 管理能力。

| 项目 | 1.14.2 公司能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 管理入口 | 公司管理员创建和管理 Workspace | 新版主导航改为 WorkspaceCard | 在 `WorkspaceCard` 的 WorkspaceSwitcher 下接入“创建空间”和“管理空间”按钮 |
| 管理员可见性 | `is_system_admin` 控制入口 | 新版资料通过 userProfileQueryOptions 获取 | 在 AccountResponse 增加公司管理员字段并生成前端类型 |
| 创建 Workspace | 公司统一创建服务 | 新版无公司创建事务 | 保留 WorkspaceProvisioningService，统一处理来源、幂等、额度、owner、插件策略、密钥和事件 |
| 指定 owner | 选择已有账号或填写新邮箱 | 新版账号增加 `normalized_email` | 复用有效账号；新邮箱创建 PENDING 账号并补 normalized_email |
| 幂等 | 同一请求重试复用请求编号 | 官方无公司创建请求记录 | 保留 WorkspaceCreationRequest、摘要和唯一键 |
| 额度 | 创建前执行公司额度锁和判断 | 新版许可证读取接口变化 | 使用新版 License 读取方式，保留原锁和计数顺序 |
| 列表与筛选 | 分页、创建时间、owner 展示 | 新版无公司管理列表 | 在管理弹窗中保留分页、日期筛选、owner 信息和缓存刷新 |
| 切换 | 校验空间、账号和成员关系 | 新版切换方法使用显式 Session | 保留公司锁定和检查顺序，并更新新版 last_opened_at |
| 归档 | 逻辑归档，记录时间、操作者、原因 | 官方只有基础状态字段 | 保留归档服务、首次审计信息和 current workspace 恢复 |
| 成员治理 | 默认 owner 删除/改角色/转移保护 | 新版调用位置变化 | 在新版删除服务、成员接口和 owner 转移流程接入公司保护 |
| 前端组件 | 旧版使用 WorkplaceSelector | 新版组件体系变化 | 管理入口迁到 WorkspaceCard；弹窗使用新版 Dialog、SegmentedControl、Input 等组件 |

#### 3.1.6 邀请注册

**来源：** `origin/feature/20260701` 的数据库邀请和 SkyOA 接受邀请流程。

| 项目 | 1.14.2 公司能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| 邀请模型 | 数据库存储 Token 摘要、状态、有效期、角色和接受信息 | 官方无公司邀请表 | 在新环境增加 WorkspaceInvitation 表、唯一约束和索引 |
| 邀请入口 | 管理员生成邀请 | 新版保留复制邀请链接能力 | 保留链接分享；邮件发送依赖现有邮件服务配置 |
| 接受方式 | 受邀人通过 SkyOA 登录 | 官方激活流程不同 | 邀请页面只走 SkyOA，不使用密码注册直接激活 |
| 邮箱校验 | OA 邮箱必须等于邀请邮箱 | 新版 OAuth 与激活链路分离 | 在接受邀请时继续校验 SkyOA 邮箱 |
| 账号处理 | 不存在则创建，存在则复用 | 新版 Account / Session 变化 | 使用新版账号能力，保留原 account_id 和账号状态 |
| 成员加入 | 按邀请角色加入 Workspace | 新版成员接口变化 | 使用新版成员模型写入邀请角色 |
| 邀请状态 | pending / accepted / cancelled / expired | 官方无公司状态机 | 保留状态、有效期、重发和幂等规则 |
| 普通登录 | 旧混合逻辑包含密码邀请参数 | 新版登录服务已变化 | 邀请逻辑从普通密码登录移除，只保留 SkyOA 邀请流程 |

#### 3.1.7 管理员初始化

**来源：** `origin/feature/20260701` 的 SkyOA 首次安装与公司初始化流程。

| 项目 | 1.14.2 公司能力 | 1.17.1 变化 | 迁移方案 |
| --- | --- | --- | --- |
| Setup 入口 | SkyOA 身份初始化管理员 | 新版 Setup 服务和数据结构变化 | 在新版 Setup 增加 SkyOA 初始化分支 |
| 管理员账号 | 使用 SkyOA 资料创建初始管理员 | Account 创建与 Session 变化 | 使用新版账号接口，保留 OA 身份和公司管理员标记 |
| 默认 Workspace | 初始化时创建默认空间 | Workspace 创建逻辑已集中到公司服务 | 复用 3.1.5 WorkspaceProvisioningService 创建默认空间 |
| owner / 成员 | 初始化管理员成为默认空间 owner | 新版成员写入方式变化 | 使用公司统一创建事务保存 owner 成员 |
| 公私钥 | 创建空间同时生成并记录密钥 | 新版 Key Provider 变化 | 复用 3.1.3 已迁入的 RSA 分步接口 |
| 失败清理 | 初始化失败只清理本次新建内容 | 新版事务与外部 Storage 操作分离 | 回滚数据库，并只删除本次创建的私钥和初始化对象 |
| 安装状态 | 初始化完成后记录系统已安装 | 新版安装记录增加字段 | 使用 1.17.1 安装记录结构，不另建公司状态模型 |

#### 3.1.8 通用 SSE 请求

**来源：** `origin/feature/20260825_S3` 的通用请求头修复。

| 项目 | 1.14.2 公司能力 | 1.17.1 情况 | 迁移方案 |
| --- | --- | --- | --- |
| Header 合并 | 自定义 Header 与认证、CSRF、分享身份 Header 合并 | `fetchOptions.headers` 可能覆盖系统 Header | 系统 Header 优先，保留不冲突的自定义 Header |
| GET / POST | 两类 SSE 请求均使用相同合并规则 | 新版请求封装位置变化 | 在新版 Request 层统一处理 GET / POST |
| 调度代码 | 原提交混有公司调度测试路径 | 阶段一不迁公司调度 | 只提取通用 Header 修复，不迁调度接口和测试依赖 |

#### 3.1.9 Workflow 请求参数兼容

**来源：** `origin/feature/20260825_S3` 混合提交 `e2e06e69` 中独立于公司调度的请求参数处理。

| 项目 | 1.14.2 公司处理 | 1.17.1 情况 | 迁移方案 |
| --- | --- | --- | --- |
| 会话 ID | 去除两端空格，纯空白转为 None | 新版 `uuid_value()` 已接受空字符串，但纯空格行为不同 | 先 trim，空白转 None，其余值继续交给新版 UUID 校验 |
| 父消息 ID | 与会话 ID 使用同一规范化 | 新版未加入同一处理 | 使用相同 trim / None 规则 |
| 调度调试代码 | 同一提交包含公司 Console 调度修改 | 阶段一不迁公司调度 | 只迁请求模型处理，调度服务及对应测试排除 |

### 3.2 独立 1.17.1 基线与组件部署

#### 3.2.1 分支、流水线与 DevOps 部署

| 对象 | 1.14.2 环境 | 独立 1.17.1 环境 | 阶段一处理 |
| --- | --- | --- | --- |
| 基线分支 | `release/1.14.2` | `release/1.17.1` | 1.17.1 官方基线独立维护 |
| 迁移开发分支 | 1.14.2 开发线 | `feature/20260920_1.17.1_UPGRADE` | 阶段一能力统一在独立 1.17.1 分支迁移 |
| 测试发布分支 | `test` | `test_1.17.1_UPGRADE` | 两套测试分支互不覆盖 |
| 流水线 | 1.14.2 test 流水线 | 1.17.1 独立流水线 | 分别绑定对应分支、Commit 和制品 |
| 部署 | 当前 test 部署 | `test-upgrade-1.17.1` | 在现有应用下增加隔离部署，原 test 保持不动 |
| 配置 | 1.14.2 test 配置 | 1.17.1 upgrade 配置 | 数据库、Redis、Secret、服务地址和前端构建参数独立 |

#### 3.2.2 平台构建与启动

**来源：** `origin/feature/20260624_test_1` 的平台部署支持，以及 1.17.1 在公司基础镜像上的兼容要求。

| 项目 | 原公司实现 | 1.17.1 变化 | 阶段一处理 |
| --- | --- | --- | --- |
| Web 启动参数 | 使用 npm 参数 | 新版改为 pnpm | 使用新版 pnpm 参数 |
| Web 监听地址 | 未指定时读取 `SERVER_HOST` | 新版默认读取系统 `HOSTNAME` | 保留 `SERVER_HOST`，避免容器名成为监听地址 |
| Node 版本 | 支持较宽 Node 22 / 24 范围 | 1.17.1 使用新版 Node 要求 | 采用 1.17.1 官方 Node 版本约束 |
| UUIDv7 Migration | 已有 `uuidv7()` 时旧逻辑可能提前退出 | 新版继续创建缺失的 `uuidv7_boundary()` | 使用 1.17.1 官方 Migration 行为 |
| build.sh | 公司脚本生成部署制品 | 新版依赖、目录和构建产物变化 | 保留公司制品入口，适配 1.17.1 构建依赖和产物 |
| Git 兼容 | 旧版 `flask-restx` 从 PyPI 安装，运行环境不要求 Git | 1.17.1 从 Git 仓库获取指定提交，`uv sync --dev` 需要 Git | DevOps 初始化脚本在 `uv sync --dev` 前检查并安装 Git；正式环境由运维固化到 `.init/init_shell.sh` 或预装 Git 的后端基础镜像 |
| API / Worker / Beat | 共用后端代码 | 1.17.1 继续按运行角色区分 | 共用同一后端制品，通过不同启动角色运行 |
| Web | 独立前端制品 | 前端构建方式变化 | 使用 1.17.1 Web 制品独立发布 |

#### 3.2.3 基础运行组件

| 组件 | 阶段一部署 |
| --- | --- |
| API | 现有 `dify-api` 应用增加 1.17.1 upgrade 部署 |
| General Worker | 现有 `dify-worker` 应用增加 1.17.1 upgrade 部署，只运行官方 Worker |
| Beat | 现有 `dify-worker-beat` 应用增加 1.17.1 upgrade 部署 |
| Web | 现有 `dify-web` 应用增加 1.17.1 upgrade 部署 |
| Plugin Daemon | 沿用现有应用，部署 1.17.1 对应版本 |
| Sandbox | 沿用现有应用，部署 1.17.1 对应版本 |

#### 3.2.4 PostgreSQL、Redis 与 Celery 隔离

开发与验证阶段，1.14.2 与独立 1.17.1 升级部署共用 PostgreSQL / Redis 基础设施，但通过独立数据库、Redis DB、Prefix、队列和频道隔离。**这些隔离配置属于升级期间的环境配置，不代表最终要把旧库数据导入开发库；线上升级以原测试库 `ai_studio` 就地 Schema Migration 为准。**

| 资源 | 1.17.1 隔离方案 |
| --- | --- |
| PostgreSQL | 使用独立数据库 `ai_studio_1171_dev` |
| Redis Cache | 使用 Redis DB 2，Key Prefix 为 `dify_1171_dev` |
| Celery Broker / Result | 使用 Redis DB 3，Key Prefix 为 `dify_1171_dev` |
| Redis Pub/Sub | 使用 `dify_1171_dev` Channel Prefix，不依赖 Redis DB 隔离 |
| Plugin Daemon | 使用独立插件数据库与 Redis 空间，不与 Dify 主库和主缓存混用 |

#### 3.2.5 Event Bus、Socket.IO 与 WebSocket 地址适配

| 项目 | 1.17.1 变化 | 阶段一处理 |
| --- | --- | --- |
| Event Bus | 开启 Sentinel 时仍可能按空 Event Bus URL 建立本机连接 | Sentinel 模式复用已初始化的远程 Redis Client；非 Sentinel 使用显式 Event Bus URL |
| Socket.IO Redis | 新增独立 `RedisManager`，不复用 Event Bus Client | 使用 Sentinel 节点、Service Name、DB 和认证构造 `redis+sentinel://` |
| Client 边界 | Event Bus 与 Socket.IO 是两套 Redis Client | 共用同一 Sentinel 基础设施，但保持独立 Client |
| Channel | Redis Pub/Sub 不按逻辑 DB 隔离 | 发布和订阅统一使用 `dify_1171_dev` 前缀 |
| Socket 地址 | 未显式配置时官方可回退到 localhost | 优先使用显式 `SOCKET_URL`；未配置时从 API 地址推导 ws/wss，并去掉路径、查询串和 fragment；解析失败再回退 localhost |
| 新版协作逻辑 | 新版客户端根据最终 Socket 地址判断草稿回退 | 保留 1.17.1 客户端判断，只替换地址推导逻辑 |

### 3.3 数据库迁移：本地验证 → 线上原库升级

数据库升级对象是**原测试环境的 PostgreSQL 数据库 `ai_studio`**，从 Dify 1.14.2 升到 1.17.1。使用原库最新备份在本地独立数据库演练，再将同一套经过验证的官方及公司兼容 Migration 应用到线上原库。**不使用本地测试库或 `ai_studio_1171_dev` 的内容覆盖远程 `ai_studio`，也不再安排两次业务快照搬迁。**

根据 2026-10-07 整理的[数据库迁移方案](#331-本地验证)，历史备份的本地演练已经完成，**远程原库升级尚未执行**。现有 1.17.1 升级环境继续保持独立，直到原库升级准备完毕。

#### 3.3.1 本地验证

| 顺序 | 操作 | 产出 |
| --- | --- | --- |
| 1 | 获取线上 `ai_studio` **最新备份**，恢复到本地独立演练库，核对表、字段、历史 Migration 版本及账号权限 | 可重复使用的真实数据演练环境 |
| 2 | 整理正式的“**Dify 1.17.1 官方 Migration + 公司兼容 Migration**”，检查旧版分叉历史和重复字段，不通过改 `alembic_version` 跳过迁移 | 随应用代码发布的正式迁移链 |
| 3 | 按 **3.3.3 旧公司调度清理清单**清除旧调度结构，再运行新版 Schema / 必要数据转换 | 原数据库结构升级到 1.17.1 |
| 4 | 对账保留的业务数据、关系、权限与迁移版本；再次运行迁移确认无新变更，并在独立库实际演练备份恢复 | 可执行迁移包与演练结论 |

已有历史演练表明：备份恢复、清理旧调度、Schema 升级、数据对账、重复升级及残留引用检查均已通过；密码登录、原应用读取、新文件上传下载和无模型 Workflow 已完成本地验证。**仍需用最新备份复验，并把本地演练脚本整理为正式可执行版本**；不能只修改演练脚本的连接地址便直接操作线上库。

#### 3.3.2 线上更新

| 顺序 | 操作 | 要求 |
| --- | --- | --- |
| 1. 停服务 | 核对远程实例与库名 `ai_studio`；停止连接该库的新旧 API、Worker、Beat、旧 Scheduler 等写入进程 | 即使测试环境暂时无人使用，也要避免后台任务继续写入 |
| 2. 全库备份 | 对线上原库执行完整 `pg_dump`，恢复到另一独立库，核对数据**及应用账号权限** | 确认备份可实际恢复，而不仅是文件存在 |
| 3. 清理旧调度 | 只处理 **3.3.3** 中列明的 14 张公司调度表及指定列、索引、序列 | 保留官方 Trigger / Schedule 和业务数据；禁止扩大删除范围 |
| 4. 原库升级 | 由**单个迁移进程**在原 `ai_studio` 运行已复验的官方及公司兼容 Migration | 不能让多个应用实例启动时同时执行 Migration |
| 5. 核对并启动 | 先核对数据与新版 Schema，再启用 1.17.1 API、Web、Default Worker 和按需启用的 Priority Worker；验收后恢复**单实例 Beat** | 旧 Scheduler、旧 Managed Worker 不再启动 |

整个过程是**原库就地升级**，不是清空原库后导入开发库。线上若出现新备份与历史演练不一致、预期外删重或字段冲突，先排查并重新演练，不继续执行。

#### 3.3.3 旧公司调度清理范围

仅删除旧公司托管执行留下的专用结构，新执行隔离仍走 Dify 1.17.1 官方 Runtime。以下行数来自 **2026-10-01 备份的本地演练**，不代表当前线上数量，线上执行前需重新统计。

| 旧公司专用表 | 历史演练行数 |
| --- | ---: |
| `app_execution_console_debug_idempotency` | 11 |
| `app_execution_generations` | 1,178 |
| `app_execution_input_snapshots` | 1,178 |
| `app_execution_jobs` | 1,178 |
| `app_execution_outbox` | 1,227 |
| `app_execution_policies` | 1 |
| `app_execution_policy_audits` | 1 |
| `app_execution_priority_overrides` | 0 |
| `app_execution_resource_slots` | 2 |
| `app_execution_results` | 1,054 |
| `app_execution_scheduler_leases` | 1 |
| `app_execution_scheduling_audits` | 1 |
| `app_execution_worker_config_audits` | 0 |
| `app_execution_worker_configs` | 2 |
| **合计 14 张表** | **5,834** |

同时删除如下旧公司结构：

| 对象 | 处理 |
| --- | --- |
| `workflow_trigger_logs.schedule_id` | 删除旧调度关联列 |
| `workflow_trigger_logs.scheduled_for` | 删除旧计划时间列 |
| `ux_workflow_trigger_logs_schedule_occurrence` | 删除相应唯一索引 |
| `app_execution_enqueue_sequence_seq` | 删除旧调度入队序列 |

两列在历史演练中各有 741 个非空值，**不等于额外删除 1,482 行**。`workflow_trigger_logs` 官方表仍保留（历史演练 2,011 行），包括 `queue_name`。保留账号、Workspace、邀请、应用、官方 Workflow、官方 Trigger / Schedule、会话消息、凭据和私钥引用。**不清 Redis，不删除 S3 对象、私钥实体、插件数据**；遇到清单之外的依赖应停止处理，不能用宽泛 `CASCADE` 扩大删除。

#### 3.3.4 官方与公司兼容 Migration

| 类型 | 迁移处理 |
| --- | --- |
| 官方 Schema | 执行 1.14.2 → 1.17.1 官方结构变更，新增必要的表、字段、索引和函数 |
| 历史公司字段 | 如私钥路径字段已存在，先比对类型、约束和定义；一致则保留，缺失才补齐，定义冲突则停止 |
| Migration 衔接 | 正确连接旧公司历史与 1.17.1 新迁移链，迁移文件纳入新版代码；不得仅修改 `alembic_version` |
| 邮箱 | 补齐 `accounts.normalized_email`，不修改原 `email` |
| 维护人 | `apps.maintainer` / `datasets.maintainer` 按 `created_by` 回填 |
| 终端用户类型 | `end_users.type` 中 `service_api` 转为 `service-api` |
| 模型类型 | 根据官方新枚举映射（例如 `text-generation` → `llm`），不做无依据的数据转换 |
| 执行隔离 | 增加 `apps.execution_pool` 等本次确定保留的公司字段，历史 App 默认 `default` |

历史备份演练为：清理后 122 张表，升级后 146 张表，原表新增 24 列；数据总行数 **16,694 → 10,860 → 10,860**，减少的 5,834 行均来自明确列出的旧公司调度表。这些数字只用作**历史演练参考**，不能当作最新线上预期值。

#### 3.3.5 迁移核对与恢复

数据核对先于产生新记录的功能操作，至少核对：保留表的逐表逐字段差异、账号与 Workspace 关系、应用/Workflow/历史运行记录、文件引用及私钥解密、列与索引、序列、Migration Head；重复运行迁移应无待执行变更。之后核对 OA 登录、空间切换、历史文件访问、插件和必要模型调用，这些依赖远程环境的项目**不能被本地结果替代**。

失败时按 **停止新版服务 → 用线上升级前备份恢复 `ai_studio` → 核对数据与权限 → 恢复匹配的旧版本服务**执行。只回退镜像无法恢复已删除调度数据；恢复前保留失败现场和日志。恢复会失去备份时间点之后的新写入。

---

## 四、阶段二：1.17.1 新能力升级

阶段二直接基于阶段一形成的独立 1.17.1 版本继续升级，不再新建第二套基线。功能层只选择需要启用的 1.17.1 新能力；部署层只增加这些能力实际需要的运行组件。

### 4.1 功能升级

| 能力方向 | 1.17.1 新增或增强能力 | 组件影响 | 阶段二处理 |
| --- | --- | --- | --- |
| Workflow 与应用编排 | 自然语言生成 Workflow / Chatflow、运行记录导出、节点定位、LLM Environment 等 | 主要复用 API / Web / Worker | 在阶段一 1.17.1 基线上直接启用 |
| Human Input | 富表单、Loop / Iteration 内 Human Input | 复用官方 Workflow Runtime | 使用 1.17.1 官方能力 |
| Agent | 新 Agent App、Agent Skills、Agent DSL、Agent Home Snapshot | 需要 Agent Backend；Local Runtime 还需要 Sandbox 与网络代理 | 作为阶段二重点新增能力接入 |
| WebApp | 应用描述、输入提示等展示能力 | Web / API | 直接在现有 1.17.1 组件上升级 |
| 可观测 | Unified Tracing、Knowledge Tracing | API / Worker / Trace 后端 | 接入现有公司可观测链路 |
| CLI | difyctl | 无长期运行组件 | 作为运维工具使用，不新增应用 |
| 知识库与检索 | Excel 图片解析、ODT、TiDB 混合检索等 | Worker / Plugin / Vector Store | 按现有知识库后端和数据源启用 |
| 多模态与工具 | 文件直接传入多模态模型、日期参数类型 | API / Plugin | 随应用能力直接启用 |
| 安全与数据治理 | 外部 KMS Provider、会话清理等 | Storage / 定时任务 | 与阶段一已有 KMS / Storage 能力合并使用 |

### 4.2 组件与部署

阶段二沿用阶段一已经部署的 API、General Worker、Beat、Web、Plugin Daemon 和 Sandbox；只为 1.17.1 新能力补充额外运行角色。

| 对应能力 | 新增运行组件 | DevOps 部署方式 | 与阶段一的关系 |
| --- | --- | --- | --- |
| 新 Agent App / Skills / Agent Home | `agent_backend` | 新增独立应用 `dify-agent-backend`，独立部署和运行配置 | 调用阶段一 API、Plugin Daemon、Redis 等基础服务 |
| Agent Local Runtime / Shell Workspace | `local_sandbox` | 新增独立应用 `dify-agent-local-sandbox` | 只服务 Agent Runtime，不替换阶段一 `dify-sandbox` |
| Agent Sandbox 出网隔离 | `agent_ssrf_proxy` | 作为 Agent Runtime 独立网络代理组件部署 | 与 `local_sandbox` 配套使用 |
| Workflow 实时协作 | `api_websocket` | 复用 API 后端制品，在 `dify-api` 下增加独立 WebSocket 部署 | 使用阶段一 Socket.IO / Redis Sentinel 配置 |
| Unified / Knowledge Tracing | Trace 后端连接 | 不新增 Dify 核心应用，接入公司现有 Trace 基础设施 | API / Worker 增加对应配置 |
| 知识库与检索增强 | Plugin / Vector Store 依赖 | 复用 Plugin Daemon、Worker 和现有向量数据库；仅增加对应 Provider 配置 | 不额外复制核心服务 |
| difyctl / WebApp / 多模态与 Tool 增强 | 无 | 不新增长期运行组件 | 直接使用阶段一现有组件 |

新增应用继续使用独立 1.17.1 分支和流水线体系，和阶段一组件使用同一版本基线；新增组件的 Redis、Secret、内部服务地址和网络策略随对应应用单独配置。

---

## 五、阶段三：执行引擎隔离

本阶段采用重新设计的[执行引擎隔离方案](dify-execution-engine-isolation-plan.md)，不再迁移旧公司统一准入、Job / Lease、Managed Scheduler、Managed Worker、任务中心及调度专用监控。直接复用 Dify 1.17.1 官方执行机制，仅为 **Workflow / Chatflow 异步执行**增加应用级执行池、Celery Queue 路由和独立 Priority Worker。

### 5.1 方案架构

![Dify 执行引擎隔离架构](assets/dify-baseline-upgrade/dify-execution-isolation-architecture-v4.svg)

隔离范围是 **Queue 和 Worker 计算资源**，不是部署两套 Dify。API、PostgreSQL、Redis Broker、Plugin Daemon、Sandbox 和官方 Runtime 继续共享。

### 5.2 执行池与隔离范围

| 项目 | Default Pool | Priority Pool |
| --- | --- | --- |
| App 配置 | `apps.execution_pool = default`（默认） | `apps.execution_pool = priority` |
| 执行 Queue | `workflow_based_app_execution` | `workflow_based_app_execution_priority` |
| Worker | 现有 General Worker | 新增独立 Priority Worker |
| 计算资源 | 现有 CPU / Memory / replicas | 独立配置 CPU / Memory / replicas |
| 官方执行代码 | 共享 1.17.1 Workflow / Chatflow Runtime | 与默认池使用同一份代码 |
| Human Input 恢复 | Resume 再进入 Default Queue | Resume 再进入 Priority Queue |

本阶段只隔离经 Celery 异步投递的 Workflow / Chatflow 任务及其恢复任务。官方同步 Blocking 路径、Chat / Completion / Agent 的直接 Generator 路径保持原样，不纳入本阶段隔离。

### 5.3 代码改造

| 改动位置 | 具体处理 |
| --- | --- |
| App Model / Migration | 在 `apps` 增加 `execution_pool varchar(32)`，默认 `default`，取值 `default / priority`；历史应用保持默认池 |
| App 配置接口 | 支持查询和修改 `execution_pool` |
| ExecutionQueueRouter | 固定映射到 Default / Priority Queue；不做任务调度、优先级计算或状态管理 |
| `api/services/app_generate_service.py` | Workflow / Chatflow 异步投递改由 Router 选择 `queue`，使用官方 Task 的 `apply_async` |
| `api/services/human_input_service.py` | `resume_app_execution` 投递前按所属 App 重新选择执行池，恢复后仍进入对应 Worker |
| AppGenerator / GraphEngine / WorkflowRun / SSE | 复用 1.17.1 官方实现，不恢复旧公司 Generator 接管与结果协议 |

Queue Router 映射：

```python
DEFAULT_QUEUE = "workflow_based_app_execution"
PRIORITY_QUEUE = "workflow_based_app_execution_priority"

class ExecutionQueueRouter:
    @staticmethod
    def resolve(app) -> str:
        return PRIORITY_QUEUE if app.execution_pool == "priority" else DEFAULT_QUEUE
```

首次执行使用 `workflow_based_app_execution_task.apply_async(args=[payload_json], queue=queue)`；Human Input 恢复调用官方 `resume_app_execution.apply_async(..., queue=queue)`。两处共用 Router，任务内部执行链路不改。

### 5.4 Worker 与部署

![执行隔离部署架构](assets/dify-baseline-upgrade/dify-execution-isolation-deployment-v4.svg)

| 组件 | 改造 |
| --- | --- |
| Dify API / Web | 沿用现有部署，API 增加入队 Queue 路由 |
| Default Worker（`dify-worker`） | 继续监听现有其他队列和 `workflow_based_app_execution`，不监听 Priority Queue |
| Priority Worker | 同一 1.17.1 后端镜像新增独立 Deployment，仅监听 `workflow_based_app_execution_priority` |
| Redis / Celery | 复用阶段一已经隔离的 1.17.1 Broker，新增 Queue 名，不新建 Broker |
| PostgreSQL / Plugin Daemon / Sandbox | 继续共享，不为执行池另部署一套 |

Priority Worker 启动命令：

```bash
cp .env.test .env && \
uv run celery -A app.celery worker \
  -P gevent -c 1 --loglevel INFO \
  -Q workflow_based_app_execution_priority
```

Default / Priority Worker 分别配置 CPU、Memory 和 replicas；新增 Worker 只消费 Priority Queue，不监听其他普通业务队列。

### 5.5 原公司调度能力处理

| 旧能力 | 处理 |
| --- | --- |
| Policy / Admission / 优先级排队 | 不迁入运行链路；改为 App 选择 Default / Priority 执行池 |
| Job / Lease / Outbox / Scheduler / 容量调度 | 不迁、不启动旧专用服务 |
| 旧 Standard / Critical Managed Worker | 不复制旧 Worker 实现；统一用 Dify 1.17.1 官方 Worker 制品，部署两组实例 |
| 任务中心、公司 Streaming / Blocking / Result 协议 | 不迁；使用官方 WorkflowRun、Message、SSE 和任务控制能力 |
| Console Debug / Human Input / Schedule 的托管接入 | 不迁旧接管层；继续使用官方功能，仅为 Human Input Resume 增加队列路由 |
| 调度监控、审计及历史数据 | 不恢复旧调度专用页面；历史表和记录保留在旧库及备份，不作为新版运行依赖 |

## 六、最终收束：仅 PostgreSQL 与 Redis

三个功能阶段继续沿用现有 1.17.1 升级分支和部署。**最终收束只涉及 PostgreSQL 和 Redis**，不再包含 API / Web / Worker / Beat 等应用回迁、DevOps 应用替换、流水线合并或下线升级部署。线上数据库升级的具体步骤见 **3.3**，无需在收束阶段再进行第二次业务数据搬迁。

### 6.1 PostgreSQL：收束到升级后的原库

| 对象 | 最终处理 |
| --- | --- |
| 原测试库 `ai_studio` | 用最新备份完成本地演练后，停止相关写入服务，**直接在该原库**运行清理与 1.17.1 Schema Migration |
| `ai_studio_1171_dev` | 作为升级阶段隔离使用的开发/演练数据库；**不将其业务数据覆盖** `ai_studio` |
| 1.17.1 数据库连接 | 线上原库完成升级且数据核对通过后，将现有 1.17.1 应用数据库连接指向升级后的 `ai_studio` |
| 历史数据 | 保留账号、Workspace、应用、Workflow、消息、文件和凭据引用；只按明确清单删除旧公司调度结构 |
| 旧版本回退 | 保留升级前可恢复备份；回退时数据库与旧版本代码必须成套恢复 |

### 6.2 Redis：统一连接和隔离命名空间

| 项目 | 最终处理 |
| --- | --- |
| Cache / Session | 核对现有 1.17.1 Redis DB 与 Key Prefix；收束后继续使用明确的 1.17.1 命名空间，避免与旧版本 Key 冲突 |
| Celery Broker / Result | 统一 1.17.1 Broker 连接，保留 `workflow_based_app_execution` 和 `workflow_based_app_execution_priority` 两个执行队列及其他官方队列；不迁移旧任务和 Result |
| Pub/Sub / Event Bus / Socket.IO | 统一远程 Redis / Sentinel 连接与 Channel Prefix；Pub/Sub 不依赖 Redis 逻辑 DB 隔离 |
| 历史 Redis 数据 | **不清空 Redis**，不复制原 1.14.2 的缓存、Session、旧 Broker 待消费任务到新版本；切换前明确停止旧写入方 |

两者的最终关系是：**现有 1.17.1 部署 → 已升级的 `ai_studio` + 统一配置的 Redis**。S3、Plugin Daemon、Sandbox、向量库及 DevOps 部署不在本次最终收束范围内，沿用各自现有方案。

---
