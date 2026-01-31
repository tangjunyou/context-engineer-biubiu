# AI 上下文工程可视化项目（均衡型）— 工程化技术方案与落地设计（v0.2）

> 日期：2026-01-31  
> 目标：在不积累技术债的前提下，把“上下文工程的可视化 + 可溯源 + 可传播 + 可确认”的核心体验，用 **Rust 本地优先 + 高度模块化/可插拔底座** 工程化落地。  
> 你已确认：**选择均衡型**；并且对若干关键点“暂时采用我的建议，直到出现更好选择”，需要在文档中做备注。

---

## 0. 你已经做出的选择与“暂定备注”

你在上一次回复中确认的决策，我把它们工程化成可执行的“决策清单”，并且把“暂定”点明确标出来（方便未来替换、不背技术债）。

### 0.1 PRD 标注需进一步讨论的技术点（对齐）

之前方案里明确整理过 PRD 中标注的 5 个“需要进一步讨论确认”的决策点：存储架构、协作方案、LLM Provider 架构、内容审核策略、Event Sourcing 实现。

> 本文就是在这 5 个点上给出工程落地版的最终可执行路径，并补全模块边界/接口契约/迁移策略。

### 0.2 均衡型最终选型（当前版本）

- **数据底座**：事件溯源（Event Log）为唯一事实源 + 图投影（Graph Projection）为查询与可视化加速层
    
    - 原因：你项目的“神经系统/有机体”隐喻与“过程可追溯”要求，本质上需要可回放的事件链与可重建的状态视图。
        
- **图数据库（默认）**：Ladybug（嵌入式图数据库，Cypher）
    
    - **暂定备注**：你要求“在没有出现更好选择之前都采取”，我将 Ladybug 标为“默认但可替换”，并设计 GraphStore 适配层，保证未来可切换到 FalkorDB / Neo4j / IndraDB。
        
    - 关键事实：Kuzu 已归档（archived），该生态分支里 Ladybug 被视作继任者之一（此处必须更新旧方案里对 Kuzu 的依赖）。
        
- **图数据库（备选）**：IndraDB（Rust 原生、可嵌入/可 server、可插拔 datastore）
    
    - **暂定备注**：作为 Ladybug 风险对冲；同时也是“纯 Rust 生态”路线的技术后备。([GitHub](https://github.com/indradb/indradb "https://github.com/indradb/indradb"))
        
- **协作（MAV）**：本地优先 + 文件级同步（iCloud/Google Drive/Dropbox）
    
    - **暂定备注**：你要求两种方式都需要（本地优先 + 后续多人协作），所以 MAV 用最小系统验证价值；V1.5 再上 P2P+CRDT。该分阶段路线之前已明确。【434:11†技术方案推荐报告.md†L5-L38】
        
- *P + CRDT，端到端加密可加 Noise）
    
    - Yrs：Yjs 的 Rust 兼容移植。([docs.rs](https://docs.rs/yrs/latest/yrs/ "https://docs.rs/yrs/latest/yrs/"))
        
    - libp2p：Rust 实现的模块化 P2P 框架。([docs.rs](https://docs.rs/libp2p/latest/libp2p/ "https://docs.rs/libp2p/latest/libp2p/"))
        
- **LLM Provider 架构**：Dify 式三层（ModelManager → Provider → Model）+ 支持 Agent/Unit 覆盖
    
    - 这是为了同时满足：普通用户“全局配置够用” + 高阶用户“Agent/Unit 精细控制、降级链可审计”。【434:10†技术方案推荐报告.md†L14-L55】
        
    - 参考实现：Dify / Cherry Studio（只借鉴结构与经验，不直接抄实现）。【434:10†审核**：MAV 零审核；V1.5+ 可选本地自检 Agent（模板分享走社区准则确认）  
        -台背合规锅导致架构扭曲”。【434:3†技术方案推荐报告.md†L5-L36】
        
- **MCP**：使用官方 Rust SDK（rmcp），构建“本项目既是 MCP Client 也是 MCP SDK 以 `rmcp`/`rmcp-macros` 形式提供，面向 Server/Client，支持多种 Transport（stdio、HTTP 等）。([GitHub](https://github.com/modelcontextprotocol/rust-sdk "https://github.com/modelcontextprotocol/rust-sdk"))
    

---

## 1. 工程原则（用来“杀死技术债”）

你明确要求“不允许积累技术债、底层高度模块化可插拔”。我将它具体化为可执行的工程约束：

1. **所有可替换的外部系统都必须隔离在 Adapter crate**
    
    - Graph DB / EventStore / BlobStore / SearchIndex / ModelProvider 都必须通过 trait 边界隔离。
        
    - 任何业务层不得直接写 Cypher/Redis 协议/SQL（必须走抽象接口），否则立刻产生 vendor lock-in 技术债。
        
2. **事件是唯一事实源；任何“状态”都可重建**
    
    - 图数据库是投影（Projection），可丢可重建，不承载“真相”。
        
    - 这可以把“换图数据库/换索引/换 UI”从灾难变成“重放即可”。
        
3. **数据通用性第一**
    
    - 对外交换格式 `.ctxpkg`、事件 Schema 版本、资源/附件规范要长期稳定；内部存储实现随时可换。
        
    - “通用性”比“内部实现优雅”更重要。
        
4. **渐进增强不是“先糊后补”**
    
    - 你要的是“高质量”，所以 MAV 只减功能、不减架构骨架：接口契约、格式版本、回放能力、测试基座从第一天就要到位。【434:7†技术方案推荐报告.md†L29-L32】
        

---

## 2. 总体架构（均衡型）

### 2.1 分层视图

```mermaid
flowchart TB
  UI[UI: Tauri + React] --> CoreAPI[Core A:contentReference[oaicite:11]{index=11}d[Command Bus]
  Cmd --> ES[EventStore (事实源)]
  ES --> PE[Projection Engine]
  PE --> GS[GraphStore (投影)]
  CoreAPI --> Query[Query API]
  Query --> GS
  CoreAPI --> Blob[BlobStore]
  CoreAPI --> Model[ModelRouter]
  Model --> Providers[Providers(OpenAI-compatible)]
  CoreAPI --> Plugins[Plugin Host (Wasm)]
  CoreAPI --> Sync[Sync Layer]
  Sync -->|MAV| FileSync[iCloud/Drive]
  Sync -->|V1.5+| P2P[libp2p + Yrs]
```

> 关键点：**写路径 = Command → EventStore → Projection**；读路径主要走 GraphStore。  
> Projection 可以随时重放修复一致性。

### 2.2 “确认中心 + 传播引擎”在架构中的位置

- **变更传播引擎**（Propagation Engine）不是 UI 逻辑，而是核心域服务：
    
    - 输入：某个 Context Unit 的变更事件
        
    - 输出：受影响的 Unit 列表、需要触发的 Maintainer/Review 任务、需要进入确认中心的候选变更集
        
- **确认中心**是一个状态机（State Machine）+ 事件流：
    
    - 所有确认动作都写入 EventStore，保证可审计与可回放。
        

---

## 3. 核心领域模型（Context Unit / 边类型 / 传播规则）

### 3.1 Context Unit（统一抽象）

> 一切都是 Context Unit：人物卡、世界观规则、章节正文、Prompt 片段、工具链配置、Agent 配置、模型套餐、审查规则……都用同一种实体抽象。

最小字段建议：

- `unit_id: ULID`
    
- `unit_type: String`（由插件或内置类型系统提供）
    
- `title: String`
    
- `body: RichText | Markdown | JSON`
    
- `tags: Vec<String>`
    
- `risk_flags: Vec<RiskFlag>`（“高风险需审查”等不做固定四分类，使用 tag/flag）
    
- `meta: JSON`（可扩展）
    

### 3.2 四种边类型（决定传播行为的关键）

你在思维演进里已经得出非常关键的结论：**引用即依赖，但边必须分类型，否则传播混乱**。工程上我直接把它固化为 enum + 传播策略矩阵：

|边类型|语义|传播行为（引擎默认）|
|---|---|---|
|`Reference`|规则/Prompt/正文中引用某 Unit|✅ 触发下游重新评估（dirty + 建议任务）|
|`Activation`|A 变化激活 B 的维护流程|✅ 触发 Maintainer 工作流|
|`Containment`|结构包含（父子）|⚠️ 默认只向上冒泡“脏标记”；是否向下传播需显式规则|
|`Derivation`|派生/溯源（记录来源）|❌ 不参与传播（只用于 provenance/time-travel）|

> 这套规则的价值：UI 可以按边类型过滤；传播可解释；“神经系统”隐喻成立。

### 3.3 GraphStore 必须支持的最小查询

为了不把业务绑死在 Cypher/某图引擎上，我们定义**业务必须的查询集合**（GraphStore trait 只暴露这些能力）：

- `neighbors(unit_id, edge_type, direction)`
    
- `downstream(unit_id, edge_types, max_depth)`（用于传播）
    
- `upstream_provenance(unit_id, max_depth)`（用于溯源）
    
- `subgraph(root, filters)`（用于可视化局部图）
    
- `checkpoint_get()/checkpoint_set()`（投影进度）
    

---

## 4. 事件溯源设计（EventStore 为唯一事实源）

### 4.1 事件类型（建议分层）

事件不要“一个大 JSON”，而要分域（便于演进、也便于插件扩展）。

- **Unit 事件**
    
    - `UnitCreated`
        
    - `UnitUpdated`（可携带 patch：JSON Patch 或 field-level diff）
        
    - `UnitDeleted`
        
- **Edge 事件**
    
    - `EdgeAdded`
        
    - `EdgeRemoved`
        
- **传播/确认事件**
    
    - `PropagationComputed`（可选：用于调试/回放）
        
    - `ConfirmationTaskCreated`
        
    - `ConfirmationAccepted` / `ConfirmationRejected` / `ConfirmationEdited`
        
- **模型调用与降级事件**
    
    - `ModelCallRequested`
        
    - `ModelCallCompleted`
        
    - `ModelDegraded`（你强调要可审计，这就是审计点）
        
- **插件事件**
    
    - `PluginInstalled` / `PluginEnabled` / `PluginDisabled`
        

### 4.2 事件 Schema 与版本策略（通用性保证）

- 事件 envelope（强制）：
    
    - `event_id`（ULID，单调趋势 + 可排序）
        
    - `ts`（写入时间）
        
    - `schema_version`
        
    - `event_type`
        
    - `payload`（MessagePack/CBOR/JSON）
        
    - `actor`（user/agent/plugin）
        
    - `correlation_id`（一次操作链路）
        
- 事件兼容规则：
    
    - 只允许“向后兼容”的字段新增
        
    - 破坏性变更必须升 `schema_version`，并提供 migration（以事件重放为基础做迁移）
        

### 4.3 EventStore 的两阶段路线（均衡型落地）

#### MAV：SQLite（WAL）作为事件容器（暂定）

> 这是你同意“暂时采用我的建议”的部分之一：先用成熟的 SQLite WAL 解决 crash safety 与跨平台问题，同时不把业务绑定在 SQL 上（只当 append-only log）。【434:4†技术方案推荐报告.md†L13-L17】

- Schema 极简：
    
    - `events(event_id INTEGER PRIMARY KEY AUTOINCREMENT, ulid TEXT UNIQUE, ts INTEGER, typ只做 append、range scan、按 id 重放
        
- **关键约束**：业务层禁止做“SQL 查询业务状态”，状态查询必须走 GraphStore（否则未来换 EventStore 会产生技术债）
    

**暂定备注（触发替换信号）**：  
当以下任一条件成立，即进入 V1.5 的自研 ctxlog：

- 单项目事件量达到你定义的阈值（例如百万级）导致 SQLite scan 成本明显上升
    
- 需要“单文件可携带 + 内嵌 WAL + footer 扫描恢复”这类更强的可移植能力
    

#### V1.5：自研 `ctxlog`（借鉴 Memvid MV2/WAL 思路，定制化实现）

你明确强调“自研不是从零开始，而是站在巨人肩膀上”。所以这里直接把 Memvid 当成“文件格式与测试策略参考”，而不是把 Memvid 当成你的事件存储本体：

- Memvid 的单文件思路：把数据/索引/元信息封装成一个可携带文件，并支持 time-travel/debug。([GitHub](https://github.com/memvid/memvid "https://github.com/memvid/memvid"))
    
- Memvid 的 WAL 提交协议（用于 crash safety 的思路参考）。([Memvid](https://docs.memvid.com/file-format/wal "https://docs.memvid.com/file-format/wal"))
    

> 工程策略：  
> 你不是要做 Memvid（AI memory layer），你要的是 **ctxlog**（可重放事件日志）。借鉴它的“单文件 + 可恢复 + 可回放”理念即可。

**ctxlog 设计目标（必须满足）**

- 单文件（便于 `.ctxpkg` 打包与移动）
    
- append-only（避免中途破坏）
    
- 每条记录有 checksum
    
- footer 或 manifest 支持快速恢复（断电/kill -9）
    
- 支持 segment/compaction（长期运行不膨胀）
    
- 可选加密（后续）
    

---

## 5. 图投影存储（GraphStore）：Ladybug 默认 + IndraDB 备选 + FalkorDB 远期

### 5.1 为什么默认选 Ladybug（而不是继续押 Kuzu）

- 现实变化：Kuzu 已归档；并且有 benchmark 项目明确写出“kuzu archived (since Oct 2025)… Ladybug is a successor”。([GitHub](https://github.com/FalkorDB/FalkorDB "https://github.com/FalkorDB/FalkorDB"))
    
- Ladybug 是嵌入式图数据库，支持 Cypher，面向对象存储/文件存储场景，适合“本地优先 + 单项目文件夹”。([GitHub](https://github.com/orgs/LadybugDB/repositories "https://github.com/orgs/LadybugDB/repositories"))
    

### 5.2 Ladybug 的工程集成方式与风险

**GitHub**

```text
https://github.com/LadybugDB/ladybug
```

**采用原因**

- 支持 Cypher（你会大量用到：依赖传播、多跳溯源、局部子图查询）([Ladybug](https://docs.ladybugdb.com/get-started/cypher-intro/ "https://docs.ladybugdb.com/get-started/cypher-intro/"))
    
- 有扩展机制（JSON、向量、全文等），对你未来把“记忆/检索”融入图谱有利。([Ladybug](https://docs.ladybugdb.com/extensions/ "https://docs.ladybugdb.com/extensions/"))
    
- benchmark 中多跳查询表现强（例如 q8/q9 类型），适合你“变更传播”这种天然多跳场景。
    

**主要风险**

1. **成熟度/生态风险（中）**：继任分支仍在快速迭代，API 可能变化。
    
    - 缓解：GraphStore trait 隔离 + 适配器单独 crate；投影可重建；并行保持 IndraDB 备选。
        
2. **Rust 集成风险（中）**：核心是 C++（通常需 FFI/绑定层）。
    
    - 缓解：把 FFI 完全关在 `ctx-graph-ladybug` crate；对外只暴露纯 Rust trait；CI 做跨平台绑定测试。
        
3. **功能缺口风险（低~中）**：比如你未来需要向量/全文/权限等具体能力时，可能存在限制。
    
    - 缓解：这些能力都应该是“可选加速层”，不改变事件事实源；缺了也能工作。
        

### 5.3 IndraDB 作为备选（纯 Rust 对冲）

**GitHub**

```text
https://github.com/indradb/indradb
```

**采用原因**

- Rust 写的图数据库，提供 server 与 library，可嵌入应用。([GitHub](https://github.com/indradb/indradb "https://github.com/indradb/indradb"))
    
- 支持可插拔 datastore（如 Postgres/sled 等），天然契合你“适配主流存储平台”的要求。([GitHub](https://github.com/indradb/indradb "https://github.com/indradb/indradb"))
    
- 设计思想受 Facebook TAO 启发（适合“固定查询集合 + 图关系服务”模式）。([GitHub](https://github.com/indradb/indradb "https://github.com/indradb/indradb"))
    

**风险**

- License（MPL-2.0）在某些商业分发场景要评估合规策略。([GitHub](https://github.com/indradb/indradb "https://github.com/indradb/indradb"))
    
- 社区活跃度与性能上限需要 PoC 验证（尤其是你那种多跳传播 + 高频写入）。
    

### 5.4 FalkorDB 作为 V2.0 的“服务端/企业级”路径（可选）

**GitHub**

```text
https://github.com/FalkorDB/FalkorDB
```

**采用原因（远期）**

- 当你需要“多人实时协作 + 企业合规 + 服务端部署”时，FalkorDB 这类服务型图数据库更适合做中心化后端。
    
- 但它不应替代本地事实源（否则违背你本地优先理念）；它应是可选的“远程投影/共享层”。
    

**风险**

- 引入服务端意味着运维/合规/成本都上涨（这正是你要谨慎的点），所以只放在 V2.0 选项中。
    

---

## 6. 插件系统（微内核）：Extism + Wasmtime（Wasm）

你强调“高度模块化、可插拔、不要技术债”，插件系统是必须的——但也必须安全。

### 6.1 选型：Extism（插件框架）+ Wasmtime（Wasm Runtime）

**GitHub**

```text
https://github.com/extism/extism
https://github.com/bytecodealliance/wasmtime
```

**采用原因**

- Extism 提供跨语言 Wasm 插件机制，适合把“单位类型/规则/检查器/导入导出器/Agent 工具”做成可装卸模块。([GitHub](https://github.com/extism/extism "https://github.com/extism/extism"))
    
- Wasmtime 是 Bytecode Alliance 的 Wasm runtime，是 Extism 等生态常用底座之一。([GitHub](https://github.com/bytecodealliance/wasmtime "https://github.com/bytecodealliance/wasmtime"))
    

### 6.2 插件能力边界（第一阶段就要定死）

为了避免“插件系统成技术债黑洞”，第一阶段只开放 **强约束、可审计** 的插件点：

- `UnitTypePlugin`：定义 `unit_type` 的 schema（JSON Schema）与默认 UI 渲染元数据
    
- `LintPlugin`：对 Unit/Edge 做规则检查，输出可解释的诊断
    
- `Importer/ExporterPlugin`：把外部数据转成事件流（或 `.ctxpkg`）
    
- `AgentToolPlugin`：以 MCP tool 或本地 tool 形式提供能力（必须声明权限）
    

> **明确不做**：任意 UI 注入/任意系统调用（这会把安全与兼容性搞炸）。

### 6.3 风险与缓解

- **风险（高）**：插件可执行任意逻辑导致安全问题
    
    - 缓解：Wasm 沙箱 + 权限声明（文件/网络/进程）+ 插件签名（后续）+ 默认禁网
        
- **风险（中）**：插件 API 演进导致兼容性灾难
    
    - 缓解：Plugin API 版本化 + 向后兼容期 + “官方插件测试矩阵”
        

---

## 7. 协作与同步：MAV 文件同步 → V1.5 P2P/CRDT

### 7.1 MAV（你要高质量但不想上来就搞分布式）

- 方式：`.ctxpkg` 或项目目录由用户自己用 iCloud/Drive 同步
    
- 冲突策略：不做“静默合并”，而是进确认中心（你项目最强的 UX 能力之一就是确认/审查）
    

这条路线的价值是：**先把“传播 + 确认 + 溯源”打磨到极致**，协作只是把同一套能力扩展到多人。

### 7.2 V1.5：Yrs + libp2p

**GitHub**

```text
https://github.com/y-crdt/y-crdt
https://github.com/libp2p/rust-libp2p
```

**采用原因**

- Yrs：Yjs 的 Rust 兼容移植，适合富文本/结构化协作数据。([docs.rs](https://docs.rs/yrs/latest/yrs/ "https://docs.rs/yrs/latest/yrs/"))
    
- libp2p：模块化 P2P 网络栈。([GitHub](https://github.com/libp2p/rust-libp2p "https://github.com/libp2p/rust-libp2p"))
    

**风险**

- NAT 穿透与可用性：P2P 不是写几行代码就完事
    
    - 缓解：预留 relay 服务器；并把“实时协作”放到 V1.5，先打磨单人核心体验【434:11†技术方案推荐报告.md†L35-L38】
        

---

## 8. MCP 集成：让你的项目成为“上下文 OS”

### 8.1 选型：官方 Rust MCP SDK（rmcp）

**GitHub**

```text
https://github.com/modelcontextprotocol/rust-sdk
```

**采用原因**

- 作为 MCP Server：把你的“上下文图 + 可追溯决策 + 传播/确认结果”暴露给外部助手（Claude Desktop / IDE Agent / 自建 Agent）。([GitHub](https://github.com/modelcontextprotocol/rust-sdk "https://github.com/modelcontextprotocol/rust-sdk"))
    
- 作为 MCP Client：接入外部工具（文件系统、网页抓取、代码执行、知识库、数据库等）而不把你工程变成一堆“集成屎山”。
    

**风险**

- MCP 生态仍在快速发展，协议与实现都可能迭代
    
    - 缓解：把 MCP 放在独立 crate（`ctx-mcp`）；核心域模型不依赖 MCP；MCP 只是适配层。
        

---

## 9. LLM Provider / ModelRouter（可审计、可降级、可复现）

### 9.1 三层结构（建议落地版）

- `ModelManager`：se_url、限额、策略）
    
- `Provider`：OpenAI-compatible 的抽象（OpenAI、Anthropic、国产等都用同接口）
    
- `Model`：具体模型 + 定价/上下文长度/能力标签
    
- `Binding`：Agent/Unit 对 Model 的覆盖与降级链
    

该思路与之前的推荐一致：Dify 式三层架构 + 支持 Agent 级覆盖。【434:10†技术方案推荐报告.md†L14-L55】

### 9.2 Rust HTTP 客户端实现建议

- 你可以直接使用 `async-openai` 作为 OpenAI-compatible 的起点（不是必须，但能省大量协议细节时间）。([GitHub](https://github.com/64bit/async-openai "https://github.com/64bit/async-openai"))
    
- 但“最终控制权”在你这边：Provider 适配层必须可替换（例如未来你要更强的重试/熔断/成本估算/审计字段）。
    

**GitHub**

```text
https://github.com/64bit/async-openai
```

### 9.3 决策日志（Decision Log）是强制事件

任何“模型降级/重试/选择不同 provider”的动作必须落事件（否则你项目的“审计”就成口号）：

- `ModelDegraded { from, to, reason, latency_ms, cost_estimate, correlation_id }`
    

---

## 10. UI 技术栈（桌面端优先）：Tauri + React + 图可视化

### 10.1 桌面壳：Tauri

**GitHub**

```text
https://github.com/tauri-apps/tauri
```

**原因**

- 你要本地优先、数据本地、同时 b 前端是均衡之选。([GitHub](https://github.com/tauri-apps/tauri "https://github.com/tauri-apps/tauri"))
    

### 10.2 图/流程可视化：React Flow + Cytoscape.js（分工明确）

**GitHub**

```text
https://github.com/xyflow/xyflow
https://github.com/cytoscape/cytoscape.js
```

**分工**

- React Flow：更适合“节点式工作流编辑/拖拽/连线”的交互型 UI。([GitHub](https://github.com/xyflow/xyflow "https://github.com/xyflow/xyflow"))
    
- Cytoscape.js：更适合“关系网络可视化 + 图算法/分析”（溯源路径高亮、最短路径、聚类等）。([GitHub](https://github.com/cytoscape/cytoscape.js/ "https://github.com/cytoscape/cytoscape.js/"))
    

### 10.3 UI 组件与状态管理（工程质量优先）

**GitHub**

```text
https://github.com/radix-ui/primitives
https://github.com/pmndrs/zustand
```

- Radix：强调可访问性与高质量组件底座。([GitHub](https://github.com/radix-ui/primitives "https://github.com/radix-ui/primitives"))
    
- Zustand：轻量状态管理，适合复杂交互面板与图编辑器状态。([GitHub](https://github.com/pmndrs/zustand "https://github.com/pmndrs/zustand"))
    

---

## 11. `.ctxpkg`：数据通用性的“最终锚点”（必须从第一天设计）

### 11.1 目标

- 任何项目都能导出为 `.ctxpkg`（一个文件），里面包含：
    
    - `manifest.json`（版本、依赖、hash）
        
    - `events/`（事件流：MessagePack/CBOR）
        
    - `snapshot/`（可选：加速启动）
        
    - `blobs/`（附件：图片、pdf、音频等）
        
    - `indexes/`（可选：投影或缓存，可丢）
        

### 11.2 为什么 `.ctxpkg` 能“杀死技术债”

- 你未来换 Graph DB、换 EventStore、换插件系统，都不影响用户资产：
    
    - 因为用户资产是事件流 + blob + manifest
        
- 你也能实现“适配主流存储平台”：
    
    - SQLite/文件/对象存储/Postgres 都只是 `.ctxpkg` 的不同落地形态
        

---

## 12. 可直接用 / 借鉴重写 / 必须自研：三类清单（按你要求）

### 12.1 可直接用（尽量不重复造轮子）

- rmcp（MCP Rust SDK）([GitHub](https://github.com/modelcontextprotocol/rust-sdk "https://github.com/modelcontextprotocol/rust-sdk"))
    
- y-crdt / yrs（CRDT）([docs.rs](https://docs.rs/yrs/latest/yrs/ "https://docs.rs/yrs/latest/yrs/"))
    
- rust-libp2p（P2P）([GitHub](https://github.com/libp2p/rust-libp2p "https://github.com/libp2p/rust-libp2p"))
    
- Tauri / React Flow / Cytoscape / Zustand / Radix（UI）([GitHub](https://github.com/tauri-apps/tauri "https://github.com/tauri-apps/tauri"))
    
- Extism / Wasmtime（Wasm 插件机制）([GitHub](https://github.com/extism/extism "https://github.com/extism/extism"))
    
- Ladybug（默认图投影）([GitHub](https://github.com/orgs/LadybugDB/repositories "https://github.com/orgs/LadybugDB/repositories"))
    

### 12.2 借鉴灵感并 Rust 化重写（“站在巨人肩膀上自研”）

- Memvid：借鉴单文件 + 可回放 + WAL/恢复思路（不是照搬产品目标）([GitHub](https://github.com/memvid/memvid "https://github.com/memvid/memvid"))
    
- Graphiti：借鉴“时序/双时序 + 增量更新”对你“思想变迁、上下文演化”的表达（实现用 Rust 自研）
    
- Dify / Cherry Studio：借鉴 Provider registry、路由、凭证隔离与策略层级（实现 Rust 自研）【434:10†技术方案推荐报告.md†L26-L29】
    

### 12.3 必须自研（你的项目护城河）

这些在之前的报告里已明确为“需要定制化设计”的模块：Context Unit 抽象、四种边类型系统、变更传播引擎、确认中心、溯源可视化。【434:6†技术方案推荐报告.md†L28-L35】

---

## 13. 工程落地：代码仓库/模块拆分建议（防止耦合）

建议 Rust workspace（每个 crate 都是未来替换点）：

```text
ctx-workspace/
  crates/
    ctx-core/                # 领域模型、事件定义、trait
    ctx-command/             # Command Bus + validation
    ctx-eventstore/          # EventStore trait + SQLite impl
    ctx-eventstore-ctxlog/   # 自研 ctxlog（V1.5）
    ctx-graph/               # GraphStore trait
    ctx-graph-ladybug/       # Ladybug adapter（FFI 隔离）
    ctx-graph-indradb/       # IndraDB adapter（备选）
    ctx-projection/          # Projection engine（幂等、checkpoint）
    ctx-propagation/         # 传播引擎（按边类型策略）
    ctx-confirmation/        # 确认中心（状态机）
    ctx-modelrouter/         # Provider/Model/Binding + DecisionLog
    ctx-mcp/                 # MCP server/client adapters
    ctx-plugins/             # Extism host :contentReference[oaicite:46]{index=46}              # ctxpkg 打包/解包、manifest、hash
    ctx-sync/                # MAV 文件同步钩子；V1.5 libp:contentReference[oaicite:47]{index=47}tauri/
    cli/
```

---

## 14. 质量与测试策略（自研也要“工业级”）

你要高质量、不能积累技术债，自研模块必须从第一天就有以下测试基座：

- **EventStore/ctxlog**
    
    - crash harness（随机 kill -9 后重启校验）
        
    - property-based tests（proptest：随机事件序列 → 重放一致）
        
    - fuzzing（cargo-fuzz：解析/恢复逻辑）
        
- **Projection**
    
    - 幂等性测试：重复 apply 同一批事件，图状态不变
        
    - checkpoint 测试：断点续投影不丢不重
        
- **Propagation**
    
    - 边类型矩阵测试：Reference/Activation/Containment/Derivation 的传播行为固定
        
- **Plugin**
    
    - 沙箱权限测试：默认无网络、无文件系统；权限必须显式声明
        

---

## 15. 里程碑（均衡型）

### MAV（0–3 个月）：核心价值验证但不牺牲骨架

- Context Unit CRUD + 四种边 + 局部图可视化
    
- EventStore（SQLite WAL，append-only）
    
- GraphStore（Ladybug 投影 + checkpoint 重放）
    
- 传播引擎：Reference/Activation 路径能算出影响面
    
- 确认中心：能把“变更建议”变成可审计的确认事件
    
- `.ctxpkg` 导入/导出（确保通用性从第一天成立）
    

### V1.5（3–6 个月）：把“自研底座”补齐

- ctxlog（单文件事件日志）替换 SQLite（满足你对“优雅与可携带”的追求）【434:0†技术方案推荐报告.md†L22-L33】
    
- P2P+CRDT 协作（Yrs + libp2p）
    
- 插件系统：lint/import/export/agent tool 插件可用
    

### V2.0（6–12 个月）：企业/团队增强（可选）

- 远程投影（FalkorDB/Neo4j）作为共享图层
    
- 合规/审计增强（可选自检 agent、策略模板）
    
- 多端同步体验优化（relay、权限、评论）
    

---

## 16. “暂定”事项清单（你要求必须备注）

以下选择当前采用，但我明确标为“暂定”，并给出替换触发信号：

1. **默认图数据库：Ladybug（暂定）**
    
    - 触发替换：生态活跃度下降/关键功能缺口/FFI 成本不可控
        
    - 备选：IndraDB（嵌入式 Rust）、远期 FalkorDB（服务端）
        
2. **MAV EventStore：SQLite（暂定）**
    
    - 触发替换：需要真正的单文件携带、WAL 内嵌、footer 恢复、事件量导致性能瓶颈
        
    - 替换目标：自研 ctxlog（借鉴 Memvid 思路）([GitHub](https://github.com/memvid/memvid "https://github.com/memvid/memvid"))
        
3. **协作：MAV 文件同步（暂定）**
    
    - 触发替换：你要“实时协作/多人工作台”的体验指标进入 P0
        
    - 替换目标：Yrs + libp2p([docs.rs](https://docs.rs/yrs/latest/yrs/ "https://docs.rs/yrs/latest/yrs/"))
        
4. **LLM Provider：Dify 式三层 + Agent 覆盖（暂定）**
    
    - 触发替换：出现更强的行业标准或你需要完全不同的配置范式
        
    - 目前它在工程可维护性、扩展性上最均衡。【434:10†技术方案推荐报告.md†L14-L55】
        

---

## 17. 参考与依赖（GitHub 地址汇总）

> 为了符合你“必须写清楚 GitHub 地址”的要求，我把核心依赖统一列在这里（链接放在 code block，避免 Markdown 渲染限制）。

```text
# Graph
https://github.com/Ladybu :contentReference[oaicite:50]{index=50}com/indradb/indradb
https://github.com/FalkorDB/FalkorDB
https://github.com/kuzudb/kuzu   (archived)

# Event/WAL inspiration
https://github.com/memvid/memvid

# MCP
https://github.com/modelcontextprotocol/rust-sdk

# CRDT / P2P
https://github.com/y-crdt/y-crdt
https://github.com/libp2p/rust-libp2p

# Plugins (Wasm)
https://github.com/extism/extism
https://github.com/bytecodealliance/wasmtime

# UI (desktop + graph)
https://github.com/tauri-apps/tauri
https://github.com/xyflow/xyflow
https://github.com/cytoscape/cytoscape.js
https://github.com/pmndrs/zustand
https://github.com/radix-ui/primitives

# OpenAI-compatible client (optional helper)
https://github.com/64bit/async-openai
```

---

## 结语：为什么这版是“更工程落地”的均衡型

- 它把你最核心的产品理念（上下文是有状态的、变更必须可传播、决策必须可确认、过中心**这条最稳的工程骨架上。
    
- 它把你最在意的底线（不积累技术债、底层高度模块化、数据通用性）落成了**adapter 边界 + `.ctxpkg` 锚点 + 事件 schema 版本策略**。
    
- 它把“自研”落成“借鉴 + 定制化自研”：Memvid/Graphiti/Dify 只借鉴结构与关键机制，你的护城河模块仍是自研且可审计。【434:6†技术方案推荐报告.md†L28-L35】
    

如果你愿意，我下一步可以在不改变本方案的前提下，继续把 **“MAV 的最小可交付范围（MVP 但不牺牲骨架）”** 细化成：

- 具体的数据库 schema / 事件 schema（JSON Schema/MessagePack）
    
- GraphStore 的最小 query 集合与 Cypher/适配实现策略
    
- 传播引擎的算法细节（按边类型的 traversal + dirty 标记 + 任务生成）  
    并且把每一项拆成可直接进 issue tracker 的任务列表。