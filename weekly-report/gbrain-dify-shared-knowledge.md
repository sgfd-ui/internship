# GBrain 共享知识库 × Dify MCP 接入方案

## 一、背景

### 1.1 问题内容

目前，推荐效果分析 Agent 的上下文主要保存在单次对话中。对话结束后，用户补充的业务背景、Agent 已完成的排查过程以及形成的分析结论，无法在其他新对话中被自动发现和复用。

由此带来两个主要问题：

1. **背景信息被重复提供。** 不同同事分析同一时间段或同一业务事件时，需要反复向 Agent 补充相同的背景信息。
2. **相似问题被重复排查。** 历史异常即使已经完成分析，后续在新对话中仍需重新查询数据、排除原因并整理结论，已有经验无法在团队内持续复用。

### 1.2 建设目标

本期基于已选定的 GBrain 建设团队共享知识库，并接入 Dify 推荐效果分析 Agent。实现业务背景和历史分析经验的统一沉淀、跨对话检索和团队复用。

### 1.3 预期效果

```mermaid
sequenceDiagram
    actor A as 同事甲（当前对话）
    participant Agent as 推荐效果分析 Agent
    participant Data as 当前分析数据
    participant Brain as GBrain 共享知识库
    actor B as 同事乙（新对话）

    A->>Agent: 购物车页 GMV 最近异常上涨，为什么？
    Agent->>Data: 查询当前效果与流量
    Data-->>Agent: GMV 上涨，流量明显增加
    A-->>Agent: 补充背景：双 11 大促
    Agent-->>A: 本次结论：活动流量驱动
    Agent->>Brain: 沉淀背景、证据、排查过程和结论

    Note over B,Agent: 一周后发起全新对话

    B->>Agent: 详情页 GMV 这几天也在涨，是什么原因？
    Agent->>Brain: 检索相似历史经验
    Brain-->>Agent: 返回双 11 历史 Case
    Agent->>Data: 查询本次实际数据并验证
    Data-->>Agent: 返回当前证据
    Agent-->>B: 输出本次结论和历史经验参考
```

---

## 二、共享知识方案调研与选型

| 项目 | 核心方式 | 与本场景的适配情况 | 结论 |
| --- | --- | --- | --- |
| **GBrain** | 以 Page 保存完整知识，通过 Chunk、Fact、Timeline、Link 等结构组织和检索；同时支持关键词与向量混合查询 | 能保留完整历史 Case，又能通过实验、页面、指标等精确标识和业务语义共同检索；面向团队共享 Brain，读写链路都比较直接 | **推荐，本期选型** |
| RAGFlow | 以文档和 Chunk 为核心，提供混合检索、Parent-Child、GraphRAG 等能力 | 传统 RAG 能力完整，适合大规模文档知识库；但本场景主要是短业务背景和历史 Case，共享部署组件较多 | 备选，不作为本期首选 |
| OpenViking | 将 Resource、Memory、Experience 等上下文按目录和 L0/L1/L2 分层组织 | 适合 Agent 长期上下文和分层读取，但查询和知识组织方式更偏上下文目录体系，与当前 Case 复用链路相比更复杂 | 备选 |
| SAG | 将内容抽取为 Event、Entity，并在查询时构建关系增强结果 | 适合需要事件关系扩展的知识场景，但本期核心需求是稳定找回完整历史 Case，关系加工会增加额外链路 | 备选 |
| LightRAG | 以实体、关系和原文 Chunk 形成图增强检索 | 对跨知识关联和图关系检索能力较强，但需要额外维护实体关系，写入和索引链路更重 | 后续图关系需求可考虑 |
| Mem0 | 将对话内容提炼为 Memory，并按用户、Agent、Run 等 Scope 管理 | 更适合个人或 Agent 长期记忆；自动提炼和合并会弱化完整 Case、精确标识和原始排查过程，不适合作为本期团队经验主库 | 不作为本期方案 |

---

## 三、整体架构

### 3.1 架构图

```mermaid
flowchart LR
    U[用户请求]

    subgraph D[Dify]
        A[推荐效果分析 Agent]
        MC[MCP Client\nDify 内置]
    end

    subgraph B[业务数据]
        DS[DataSrc / 数据 Tool]
    end

    subgraph G[GBrain 共享知识服务]
        MS[GBrain HTTP MCP Server\n/mcp]
        DB[(PostgreSQL + pgvector)]
        M[Embedding / LLM Provider]
    end

    U --> A
    A --> DS
    DS --> A

    A --> MC
    MC --> MS
    MS --> DB
    MS --> M
    MS --> MC
    MC --> A
```

Dify 不再额外开发一层 GBrain Tool。推荐效果分析 Agent 直接使用 Dify 原生 MCP Client 连接 GBrain 暴露的 HTTP MCP Server，GBrain 负责共享知识的存储、检索和写入。

### 3.2 模块职责

| 模块 | 主要职责 | 输出 |
| --- | --- | --- |
| 推荐效果分析 Agent | 理解用户问题，组织当前数据查询，并决定何时读取或写入历史经验 | 本次分析结论 |
| DataSrc / 数据 Tool | 查询当前时间、页面、实验和指标数据 | 当前事实与证据 |
| Dify MCP Client | 连接 GBrain MCP Server，发现并调用 GBrain 暴露的标准 MCP 能力 | MCP 调用结果 |
| GBrain HTTP MCP Server | 对外提供 `search`、`query`、`get_page`、`put_page` 等共享知识能力，并负责鉴权与调用审计 | 历史经验 / 写入结果 |
| PostgreSQL + pgvector | 保存 Page、Chunk、Fact、关系和向量索引 | 团队共享知识数据 |
| Embedding / LLM Provider | 为语义检索、抽取和后台知识加工提供模型能力 | 向量及加工结果 |

---

## 四、GBrain 知识组织

本期共享知识只保存两类内容：**业务背景**和**历史分析 Case**。完整内容统一以 Page 保存，GBrain 再基于 Page 组织用于检索的 Chunk、Fact 和关联信息。

```mermaid
flowchart LR
    A[业务背景 / 历史分析结果]
    --> B[整理为完整 Page]
    --> C[GBrain 写入]

    C --> D[(Page 正文)]
    C --> E[Chunk / Embedding]
    C --> F[Fact]
    C --> G[Link / Timeline]

    E --> H[语义与关键词检索]
    F --> H
    G --> H
    H --> I[命中 Page]
    I --> J[读取完整背景 / 证据 / 排查过程 / 结论]
```

一条历史 Case 主要包含：问题范围、业务背景、分析证据、已完成的排查过程、最终结论和来源信息。例如一次“购物车页 GMV 上涨”分析，会保存对应站点、页面和时间范围，大促背景，流量与转化证据，已排除的配置因素，以及最终原因结论。

**完整 Case 以 Page 为复用单位，Chunk、Fact、Link 和 Timeline 主要负责帮助检索和关联到对应 Page。**

---

## 五、MCP 接入与使用流程

GBrain 原生提供 MCP Server，Dify 也已支持直接连接 HTTP MCP Server，因此本方案不再开发 GBrain Tool / Plugin，而是直接使用标准 MCP 链路。

### 5.1 MCP 接入实现

GBrain 在服务端以长期进程启动 HTTP MCP：

```bash
gbrain serve --http --port 3131
```

启动后，GBrain 对外提供 `/mcp` 入口。Dify 在 **Tools → MCP → Add MCP Server** 中配置该地址，例如同机 Docker 网络下使用：

```text
http://gbrain:3131/mcp
```

如果后续 GBrain 独立到其他服务器，则只需要把地址替换为内部 HTTPS 地址：

```text
https://gbrain.internal.example.com/mcp
```

Dify 连接后会自动发现 GBrain 暴露的 MCP 能力，推荐效果分析 Agent 只挂载本期需要的读写能力，不需要手工重新定义每个 Tool Schema。

```mermaid
flowchart LR
    A[GBrain 启动 HTTP MCP Server]
    --> B[创建 Dify 专用访问凭据]
    --> C[Dify 添加 MCP Server URL]
    --> D[Dify 发现 GBrain MCP Tools]
    --> E[选择 Search / Query / Page 读写能力]
    --> F[挂载到推荐效果分析 Agent]
    --> G[完成联通测试]
```

GBrain 的远程 MCP 支持 OAuth 2.1 和 `read / write / admin` 权限划分。本期为 Dify 创建独立客户端，只授予 `read + write`，不授予 `admin`：

```bash
gbrain auth register-client dify-rec-agent \
  --grant-types client_credentials \
  --scopes "read write"
```

数据库连接、Embedding Key、LLM Key 均只保存在 GBrain 服务端；Dify 只保存访问 MCP 所需的凭据，不持有 GBrain 数据库和模型密钥。

### 5.2 历史经验读取

读取历史经验时，Agent 先通过 MCP 查询相关 Case，再结合当前业务 Tool 重新验证。历史 Case 只提供背景和排查方向，不直接替代当前数据。

```mermaid
flowchart LR
    A[用户问题]
    --> B[推荐效果分析 Agent]

    B --> C[MCP: search / query]
    C --> D[GBrain 返回候选 Page]
    D --> E[MCP: get_page]
    E --> F[完整历史背景 / 证据 / 排查 / 结论]

    B --> G[Data Tool 查询当前数据]
    G --> H[当前事实与证据]

    F --> I[结合历史经验验证]
    H --> I
    I --> J[输出本次结论]
```

默认路径是 `search / query → get_page`：先用关键词、语义和业务条件找到相关 Case，再读取完整 Page。对于已经知道 Page 标识的情况，可直接调用 `get_page`。

### 5.3 历史经验写入

写入不放在每一次普通查询后执行，而是在形成明确业务背景或完成一次有复用价值的分析后触发。

```mermaid
flowchart LR
    A[本次分析完成]
    --> B{是否值得沉淀}

    B -- 否 --> C[结束，不写入]
    B -- 是 --> D[整理问题范围 / 背景 / 证据 / 排查 / 结论 / 来源]
    D --> E[形成完整 Case Page]
    E --> F[MCP: put_page]
    F --> G[GBrain 保存 Page]
    G --> H[后续检索可复用]
```

本期写入规则保持简单：一次性指标查询、未验证猜测和内部执行日志不进入共享知识库；已确认业务背景、有证据的排查过程和最终结论可以通过 `put_page` 写入。

### 5.4 MCP 与后台任务边界

Dify 只通过 MCP 调用在线知识能力，例如 `search`、`query`、`get_page`、`put_page`。`sync`、`embed`、`extract`、`dream`、`enrich` 等需要本地引擎或文件系统的维护能力不由 Dify 远程调用，而是在 GBrain 服务所在机器上按需执行或调度。

因此运行时边界保持为：**Dify 负责使用知识，GBrain 服务负责管理知识。**

---

## 六、部署方案

### 6.1 部署原则

GBrain **逻辑上独立成服务，物理上前期可以与 Dify 共用同一台服务器**。这样既不把共享知识能力嵌入某个 Dify 应用，又不需要当前阶段额外增加机器；后续需要扩容时，只迁移 GBrain 服务并修改 MCP 地址即可。

生产数据使用 PostgreSQL + pgvector。可以与 Dify 共用 PostgreSQL 实例，但应为 GBrain 建立独立的数据库和数据库账号，避免与 Dify 自身表结构、权限和升级生命周期耦合。

### 6.2 初期部署拓扑

```mermaid
flowchart TB
    subgraph H[同一台服务器 / 同一内网]
        subgraph D[Dify Stack]
            A[推荐效果分析 Agent]
            MC[Dify MCP Client]
        end

        subgraph G[GBrain Stack]
            MS[GBrain HTTP MCP Server\n:3131 /mcp]
            W[GBrain 后台维护任务\n按需]
        end

        subgraph P[PostgreSQL 实例]
            DDB[(Dify Database)]
            GDB[(GBrain Database\npgvector)]
        end
    end

    MP[Embedding / LLM Provider]

    A --> MC
    MC -->|内部 HTTP MCP| MS
    MS --> GDB
    W --> GDB
    MS --> MP
    W --> MP

    D -.独立数据库.-> DDB
```

这里的“共机”只是共享服务器资源，不代表把 GBrain 代码放进 Dify 容器。建议分别运行独立 Container / Service：

```text
Server
├── Dify Stack
│   ├── api / worker / web / plugin-daemon ...
│   └── MCP Client
│
├── GBrain Stack
│   ├── gbrain serve --http --port 3131
│   └── maintenance worker / cron（按需）
│
└── PostgreSQL
    ├── dify database
    └── gbrain database + pgvector
```

### 6.3 服务部署步骤

**第一步：准备 GBrain 数据库。** 在现有 PostgreSQL 实例中创建独立 `gbrain` Database 和独立账号，启用 pgvector；Dify 与 GBrain 不共用业务表。

**第二步：部署 GBrain Service。** 使用 Bun 安装或 GBrain 编译二进制，在独立容器 / systemd Service 中配置 `DATABASE_URL`、Embedding Provider 和需要的 LLM Provider，然后执行 `gbrain doctor` 完成连接和 Schema 检查。

**第三步：启动远程 MCP。** 长期运行：

```bash
gbrain serve --http --port 3131
```

服务启动后由进程守护机制负责自动拉起。初期只在 Dify 与 GBrain 所在内部网络开放 `3131`，不直接暴露公网。

**第四步：创建 Dify 专用 MCP 身份。** 在 GBrain 侧创建 `read + write` 的 OAuth Client，凭据只配置给 Dify；数据库密码和模型 Key 不进入 Dify。

**第五步：Dify 接入。** 在 Dify 的 MCP 配置中添加 `http://gbrain:3131/mcp`，完成授权并确认能够发现 `search`、`query`、`get_page`、`put_page` 等能力，然后挂载到推荐效果分析 Agent。

**第六步：联调读写闭环。** 先验证“写入一个测试 Page → 新会话 Search 命中 → get_page 读取完整内容”，再接入实际推荐效果分析链路。

### 6.4 网络、权限与运行维护

运行时只有一条跨系统入口：`Dify MCP Client → GBrain /mcp`。PostgreSQL 不对 Dify Agent 暴露，GBrain 的模型 Key 也不下发到 Dify。

权限按最小范围配置：Dify 仅需要共享知识的 `read + write`；`admin`、数据库迁移和后台维护命令只允许 GBrain 运维侧使用。GBrain 的 MCP 调用统一记录审计日志，便于后续定位谁在何时读取或写入了知识。

PostgreSQL 是在线知识主存储，需要纳入现有备份策略。Markdown / Git 如需保留，可作为定期导出、人工审查和版本归档手段，不放在在线查询主链路中。

### 6.5 后续独立机器扩容

如果后续知识量、Embedding 任务或并发提高，再将 GBrain Stack 和 GBrain Database 迁到独立服务器。Dify 侧不需要修改 Agent 逻辑，只将 MCP Server 地址从：

```text
http://gbrain:3131/mcp
```

切换为：

```text
https://gbrain.internal.example.com/mcp
```

即可完成物理拆分。

---

## 七、实施计划

| 时间 | 工作内容 | 产出 |
| --- | --- | --- |
| 9.7—9.8 | 创建 GBrain 独立数据库，部署 GBrain Service，完成 Provider 与 `gbrain doctor` 检查 | GBrain 服务与存储可用 |
| 9.9 | 启动 HTTP MCP、创建 Dify 专用 OAuth Client，并在 Dify 完成 MCP Server 连接 | Dify 能发现 GBrain MCP 能力 |
| 9.10—9.11 | 接入 `search / query / get_page`，完成历史 Case 读取与当前 DataSrc 联合验证 | 历史经验可参与新对话分析 |
| 9.14—9.15 | 接入 `put_page`，实现 Case 整理、写入条件和写入闭环 | 有效分析结果可沉淀到 GBrain |
| 9.16—9.18 | 完成端到端联调、权限审计、备份和异常场景验证 | 形成可上线的共享知识闭环 |
