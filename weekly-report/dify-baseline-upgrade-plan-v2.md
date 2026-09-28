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

1.14.2 与 1.17.1 共用同一套 PostgreSQL / Redis 基础设施，通过独立数据库、Redis DB、Prefix、队列和频道完成版本隔离。

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

### 3.3 阶段一完成后的数据迁移

阶段一功能和数据库结构完成后，将 1.14.2 在同一时间点的 PostgreSQL 业务快照一次性导入当前 1.17.1 开发库 `ai_studio_1171_dev`。目标库保留已经建立的 1.17.1 Schema 和当前 `alembic_version`，不重新执行旧库 Migration，也不覆盖目标库的版本记录。

本次只迁 PostgreSQL 业务数据。旧 1.14.2 环境继续运行，快照之后的新数据不自动同步；S3 对象、Redis、向量库、Plugin Daemon 数据库和私钥文件不由迁移脚本搬运，只在数据导入后核对数据库引用。

#### 3.3.1 执行步骤

| 步骤 | 操作 | 主要处理 | 输出 |
| --- | --- | --- | --- |
| 1 | `inspect` | 读取源库和 `ai_studio_1171_dev` 的表、字段、主外键、唯一约束、索引和 `alembic_version`；生成字段映射并检查类型兼容 | Schema、版本、表数量和 Mapping 清单 |
| 2 | `export` | 在源库开启 `REPEATABLE READ` 一致性事务，逐表导出业务快照 | 每表 JSONL、行数、SHA-256 校验值和快照元数据 |
| 3 | `backup-target` | 在写入前对 `ai_studio_1171_dev` 执行完整 `pg_dump` | `target-before.dump` |
| 4 | `apply` | 再次检查目标 Schema 未变化；按外键依赖倒序清理核准的目标测试数据，再按正序写入转换后的旧数据 | 单事务导入结果及序列恢复 |
| 5 | `verify` | 对比导入前快照与目标库行数，并输出差异报告 | `counts-after.json`、`differences.json`、`report.json` |
| 6 | 业务核对 | 检查账号/OA、默认 Workspace、owner/current、应用、Workflow、知识库、历史文件引用和旧密钥可读性 | 阶段二继续使用的 1.17.1 开发数据 |

执行命令固定为：

```bash
cd api

export SOURCE_DB_HOST=...
export SOURCE_DB_PORT=5432
export SOURCE_DB_USERNAME=...
export SOURCE_DB_PASSWORD=...
export SOURCE_DB_DATABASE=...

export TARGET_DB_HOST=...
export TARGET_DB_PORT=...
export TARGET_DB_USERNAME=...
export TARGET_DB_PASSWORD=...
export TARGET_DB_DATABASE=ai_studio_1171_dev

export MIGRATION_WORK_DIR=/data/dify-migration/$(date +%Y%m%d_%H%M%S)

uv run python scripts/migrate_snapshot.py inspect
uv run python scripts/migrate_snapshot.py export
uv run python scripts/migrate_snapshot.py backup-target
uv run python scripts/migrate_snapshot.py apply
uv run python scripts/migrate_snapshot.py verify
```

连接密码只通过环境变量提供，不写入脚本参数、日志和迁移产物；`MIGRATION_WORK_DIR` 放在 Git 工作区之外，快照、备份和报告文件不提交仓库。

#### 3.3.2 Inspect：先确定哪些数据可以迁

`inspect` 不写数据，先完成三件事：

1. 确认源库和目标库不是同一个数据库，且目标数据库固定为 `ai_studio_1171_dev`。
2. 比较两边表和字段，源字段在新版没有对应字段、字段类型不兼容、或目标新增必填字段没有默认值时直接停止。
3. 生成明确的迁移清单。只有显式列入排除范围的表或字段才跳过，不通过名称猜测“执行相关数据”。

关键保护：

```python
TARGET_DATABASE = "ai_studio_1171_dev"

EXCLUDED_TABLES = {
    "alembic_version",
}

EXCLUDED_COLUMNS: dict[str, set[str]] = {}
```

字段映射采用“能明确转换才迁”的规则。例如账号补齐新版 `normalized_email`，模型类型按 1.17.1 名称转换：

```python
MODEL_TYPE_RENAMES = {
    "text-generation": "llm",
    "embeddings": "text-embedding",
    "reranking": "rerank",
}

def normalize_email(value):
    return value.strip().lower() if value is not None else None
```

旧 `alembic_version` 不导入，目标库继续使用阶段一当前 Migration Head。

#### 3.3.3 Export：生成一致性业务快照

源库全程只读，在同一个 `REPEATABLE READ` 事务中逐表读取，避免导出过程中各表对应到不同时间点。

每张表输出一个 JSONL 文件，同时记录：

- 表行数；
- SHA-256；
- 源数据库名；
- 源库 `alembic_version`。

导出的日期、UUID、bytes 等类型使用显式编码，导入时按原类型恢复，不统一转成普通字符串。

#### 3.3.4 Backup：写入前先备份目标库

在修改 `ai_studio_1171_dev` 前执行一次完整备份：

```bash
pg_dump \
  --format=custom \
  --no-owner \
  --no-privileges \
  --host "$TARGET_DB_HOST" \
  --port "$TARGET_DB_PORT" \
  --username "$TARGET_DB_USERNAME" \
  --dbname "ai_studio_1171_dev" \
  --file "$MIGRATION_WORK_DIR/target-before.dump"
```

`apply` 只有在 `target-before.dump` 已存在时才允许执行。

#### 3.3.5 Apply：单事务替换业务数据

正式写入前重新读取目标 Schema；如果目标结构与 `inspect` 时不一致，停止导入并重新执行 `inspect`。

写入顺序由外键关系自动计算：

```text
父表 → 子表       导入顺序
子表 → 父表       清理顺序
```

核心写入必须处于同一个数据库事务中：

```python
with target.begin() as conn:
    for table in reversed(order):
        conn.execute(text(f'DELETE FROM "{table}"'))

    for table in order:
        for row in snapshot_rows(table):
            mapped = map_row(table, row, target_schema[table])
            conn.execute(metadata.tables[table].insert().values(**mapped))

    reset_sequences(conn, order)
```

其中：

- 只清理 Mapping 中明确标记为 `copy` 的目标业务表；
- 不执行无范围清库或 `CASCADE`；
- 保留原业务 ID 和外键关系；
- 新版新增字段使用明确的转换值、数据库默认值或可空值；
- 任意表写入失败时整个事务回滚；
- `alembic_version`、S3、Redis、向量库、插件库和私钥文件都不在这个事务中修改。

#### 3.3.6 数据范围

| 数据范围 | 迁移处理 |
| --- | --- |
| 账号与 OA 身份 | 保留 account_id、状态、邮箱和 OA open_id；按新版规则生成 `normalized_email`，不自动合并冲突账号 |
| Workspace 与成员 | 保留 Tenant ID、owner、成员角色、默认空间、归档状态和 current 关系 |
| 邀请与创建记录 | 保留邀请 Token 摘要、状态、有效期、角色和 Workspace 创建幂等记录 |
| 应用与 Workflow | 保留应用、Workflow 定义及版本信息，按字段 Mapping 写入新版结构 |
| 知识库与文件记录 | 导入知识库、文档、分段和文件数据库记录，不移动真实对象 |
| 会话与运行历史 | 保留会话、消息、运行和节点历史，不重新执行历史任务 |
| 模型及工具凭据 | 保留加密内容和 Tenant 密钥引用，按新版模型类型和引用结构转换 |
| 安装记录 | 保留原安装状态，不重新执行管理员初始化 |
| 新版新增表 | 无旧数据来源的表保留阶段一现有结构和必要初始化值 |
| 公司调度数据 | 按明确表和字段排除，阶段三评审后再决定是否迁移 |

不迁移以下内容：

- 旧 Redis Cache、登录 Session、Celery 队列和 Result；
- S3 文件对象；
- 向量库集合；
- Plugin Daemon 数据库和插件文件；
- 公司调度专用数据；
- 旧 `alembic_version`。

#### 3.3.7 Verify：迁移结果校验

脚本先做逐表行数对比：

```python
differences = {
    table: {
        "source": source_count,
        "target": target_count,
    }
    for table in migrated_tables
    if source_count != target_count
}

if differences:
    raise RuntimeError("行数校验存在差异，停止后续使用")
```

行数一致后再做业务级检查：

| 检查项 | 校验内容 |
| --- | --- |
| 账号 / OA | account_id、邮箱、状态、open_id 绑定和 `normalized_email` |
| Workspace | 默认空间唯一、owner 正确、成员角色正确、current 唯一且不指向归档空间 |
| 邀请 | Token 摘要、状态、有效期和角色保持一致 |
| 应用 / Workflow | 原应用和 Workflow 能正常打开，定义和历史记录仍关联原 ID |
| 知识库 | Dataset、Document、Segment 关联完整，原向量引用仍能对应 |
| 文件 | 数据库中的历史 Key 能访问现有 S3 对象 |
| 私钥 | 历史 Tenant 继续使用原私钥解密已有加密数据 |

迁移完成后的数据链路为：

```text
1.14.2 PostgreSQL
      │
      ├─ inspect
      ├─ 一致性快照 export
      ▼
迁移数据包
      │
      ├─ backup ai_studio_1171_dev
      ├─ Mapping / 数据转换
      ▼
ai_studio_1171_dev
      │
      ├─ 单事务 apply
      └─ verify + 业务核对
      ▼
阶段二 / 阶段三继续开发
```

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

## 五、阶段三：执行调度能力评审

原有公司调度管理围绕统一准入、优先级、Job、Scheduler、双 Worker、任务中心、Console 调试、Human Input、Schedule、监控审计等能力进行了较多扩展，整体设计偏重，其中部分能力并不是当前业务必须保留。与此同时，1.17.1 的应用执行、Runtime、Worker、任务状态、Human Input 和 Trigger / Schedule 链路已经发生较大变化，旧实现不能直接平移。

因此阶段三不默认全量迁移，而是先重新评估每项能力的实际业务价值和迁移成本，再决定 **完整迁移、基于 1.17.1 重写、部分保留，或直接使用官方能力**。

### 5.1 高工作量迁移项

以下工作量表示：**如果决定继续保留对应公司能力，在 1.17.1 上完成适配或重写所需的主要开发成本**。各项存在公共执行链路，预计时间不能直接逐行相加。

| 能力方向 | 高工作量项 | 主要改造内容 | 工作量 | 预计时间 |
| --- | --- | --- | --- | --- |
| 托管执行引擎 | Managed Worker 与 1.17.1 Runtime 对接 | 重写 Worker 执行入口，接入 1.17.1 官方执行服务，同时保留 Job、Lease、容量和 Standard / Critical 调度 | 高 | **3～5 天** |
| Workflow / Chatflow | 正式执行链路迁移 | 重做公司 Job 与 workflow_run_id / task_id 映射，接入新版执行任务、事件、失败终态和结果回写 | 高 | **3～4 天** |
| Chat / Completion / Agent | 多应用类型托管执行 | 分应用适配新版 Generator / Service、Session、Message / Conversation、Streaming / Blocking 和终态提取 | 高 | **4～6 天** |
| 结果交付 | Streaming / Blocking / Result | 重新建立 Job 与官方执行标识映射，适配排队、运行、失败、停止、断线重连和最终结果查询 | 高 | **2～3 天** |
| 任务控制 | Stop / Cancel / Retry | queued 与 running 分别对接公司 Job 和官方停止；Retry 使用新 generation，避免旧结果覆盖 | 高 | **2～3 天** |
| Console 调试 | Draft、Single Node、Iteration、Loop | 重新接新版调试执行入口，并恢复排队、停止、结果查询、SSE 和前端状态隔离 | 高 | **4～6 天** |
| Human Input | Pause / Resume / Retry | 将公司 generation / fence 与 1.17.1 WorkflowPause、ResumptionContext、resume_app_execution 重新串联 | 高 | **3～5 天** |
| API / WebApp 入口 | 托管路由接入 | 在新版 Controller / Service 上增加托管判断，并重新适配 Streaming / Blocking / Stop 协议 | 中到高 | **2～3 天** |
| Schedule | 正式与草稿定时触发 | 基于 1.17.1 Trigger / Schedule 重新接入公司准入，避免旧轮询与官方链路重复执行 | 中到高 | **2～4 天** |

按公共执行链路合并计算，如果上述调度能力大部分继续保留，整体开发量约 **15～22 个工作日**；若 Agent、Human Input 或官方 Runtime 接入还需要额外抽象，预留 **20～25 个工作日**。

### 5.2 评审结果

评审后每项能力只保留一种处理方式：

| 处理方式 | 适用情况 |
| --- | --- |
| 完整迁移 | 公司能力仍有明确业务价值，且原设计与 1.17.1 变化较小 |
| 基于 1.17.1 重写 | 能力需要保留，但旧实现与新版 Runtime / Worker / Session / Trigger 等接口已经不兼容 |
| 部分保留 | 只保留准入、优先级、容量、资源池等仍有价值的治理能力，具体执行继续使用官方链路 |
| 使用官方能力 | 1.17.1 已覆盖需求，原公司实现没有继续维护的必要 |

阶段三最终固定：**保留功能、删除功能、需要重写的连接层、最终运行组件、数据库表与 Migration、Redis 队列与频道，以及 DevOps 应用和启动角色**。

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


---

## 七、实施时间表

阶段一已经进入收尾，代码迁移、独立 1.17.1 环境、基础组件部署、中间件隔离和业务能力迁移基本完成，当前剩余工作集中在线上完整功能验证。后续阶段不提前固定具体日期，以上一阶段完成结果作为进入条件。

状态说明：**✅ 已完成　🟡 进行中　⬜ 待开展**

| 时间 / 进入条件 | 阶段 | 工作内容 | 状态 |
| --- | --- | --- | --- |
| 09.21 - 09.23 | 阶段一：基线与部署 | 建立独立 1.17.1 分支、流水线和 `test-upgrade-1.17.1` 部署；完成 API / Web / Worker / Beat / Plugin Daemon / Sandbox 基础运行 | ✅ |
| 09.23 - 09.25 | 阶段一：基础设施 | 完成 PostgreSQL、Redis Cache、Celery、Pub/Sub、Event Bus、Socket.IO Sentinel 隔离，以及平台构建与启动适配 | ✅ |
| 09.24 - 09.28 | 阶段一：业务能力迁移 | 完成 SkyOA、S3/KMS、文件与私钥、默认 Workspace、Workspace 管理、邀请、管理员初始化、SSE 与通用兼容修改 | ✅ |
| 09.27 - 09.28 | 阶段一：数据迁移准备 | 完成 `ai_studio_1171_dev` 的快照导入流程、字段映射、目标库备份、数据转换和校验方案 | ✅ |
| 09.28 起 | 阶段一：线上完整验证 | 在升级部署上完整验证登录、账号与 Workspace、邀请、管理员初始化、S3/KMS、文件、私钥、Workflow / SSE、Socket.IO 等实际链路 | 🟡 |
| 阶段一线上验证完成后 | 阶段二：新功能升级 | 确定并接入实际需要的 Workflow / Human Input / Agent / WebApp / Tracing / Knowledge / 多模态等 1.17.1 新能力 | ⬜ |
| 阶段二功能范围确定后 | 阶段二：组件与部署 | 根据已选功能增加 Agent Backend、Local Sandbox、Agent SSRF Proxy、API WebSocket 等必要运行组件和流水线 | ⬜ |
| 阶段二完成后 | 阶段三：执行调度评审 | 重新评估原公司调度能力，确定完整迁移、基于 1.17.1 重写、部分保留或直接使用官方能力 | ⬜ |
| 调度评审完成后 | 阶段三：调度能力处理 | 按第五章评审结论实施最终保留的调度、Worker、任务管理和治理能力 | ⬜ |
| 三个阶段完成后 | 最终收束 | 固定最终组件和配置，执行正式数据迁移，收束 release / 流水线和原 DevOps 组件，并下线平行升级环境 | ⬜ |

阶段一当前完成条件只剩：**线上完整功能链路验证通过**。完成后，后续开发统一基于已经形成的 1.17.1 环境继续推进。

