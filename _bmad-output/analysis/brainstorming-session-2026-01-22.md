---
stepsCompleted: [1]
inputDocuments: ['总体想法.md']
session_topic: '通用 AI 上下文工程系统设计完善与风险识别'
session_goals: '将构想从想法稿升级为可执行的系统设计蓝图'
selected_approach: 'progressive-flow'
techniques_used: ['First Principles Thinking', 'Morphological Analysis', 'Six Thinking Hats', 'Constraint Mapping']
ideas_generated: []
context_file: '总体想法.md'
---

# Brainstorming Session Results

**Facilitator:** 耶稣
**Date:** 2026-01-22

## Session Overview

**Topic:** 通用 AI 上下文工程系统（工作台）——设计完善与风险识别
**Goals:** 验证核心概念、收敛开放问题、发现潜在盲点、优化系统架构

### Context Guidance

输入文档`/Users/jingshun/Desktop/AI 上下文工程可视化项目/总体想法.md`描述了一个通用 AI 上下文工程系统的完整构想，包括：
- 核心抽象：上下文单元（变量 + 卡片）
- Agent 分层职责（系统级/变量级/审查级）
- 变更传播与溯源系统
- 微内核+插件化底层架构
- 6个待讨论的开放问题

### Session Setup

用户希望通过系统化的头脑风暴将这个初步构想梳理成熟，使项目变得更加健壮和可落地。

---

## Phase 1: 发散探索（First Principles Thinking）

### 摘要
本次头脑风暴以第一性原理审视原始设计，形成了 **11 个核心设计决策**：
1) 统一“变量/卡片”为 Context Unit；
2) 引入 active/archived 生命周期与“晋升”机制；
3) 区分 Reference/Activation/Containment/Derivation 四类关系边；
4) Tag + Policy 分离（分类与行为正交）；
5) Authority Source vs Maintainer 分离；
6) Trigger = Event + Condition + 安全阀；
7) ChangeSet/ImpactSet/Reason 结构化确认；
8) Event Sourcing 作为溯源底座；
9) 配置包版本化 + Migration；
10) UI 从“流程编排”转向“有机系统蓝图”；
11) Agent 推理过程纳入溯源（底层记录一切，用户层选择性开放）。

---

## 背景：我们在挑战什么？
输入文档《总体想法.md》描述了一个通用 AI 上下文工程系统的初步构想。核心理念是：**LLM 本质无状态，要让 AI 在复杂长期任务中表现得“有状态、连贯、可控”，关键在于上下文工程。**
原始设计包含：变量/卡片、溯源系统、变量结果/卡片结果、四种确认模式等。我们用第一性原理挑战它们：是否存在更简洁且可扩展的抽象？

---

## 问题 1：变量和卡片真的是两个不同的东西吗？

### 问题背景
原始设计用“变量=原子”、“卡片=组合”的二元抽象。

### 挑战
> 「角色卡」能被「势力卡」引用为“变量”吗？若可以，二者的区分是否只是视角而非本质？

### 讨论
用户确认可以，并指出“变量们其实都是上下文包”。由此可见二者本质一致，只是粒度不同。

### 结论
**统一抽象：Context Unit（上下文单元）**
- 底层只有一种实体，可作为叶子或容器；
- “变量/卡片”留在 UI 作为视图/标签，而非底层结构。

**价值**：存储、权限、溯源、传播、查询、UI 复用同一机制，显著降复杂度。

---

## 问题 2：「变量」与「变量结果」的边界是什么？

### 问题背景
区分“变量=活数据”与“变量结果=固定产出”在实践易模糊。

### 挑战
> AI 写第三章正文引入新角色。应当：A) 仅留在结果；B) 自动提取为变量；C) 由用户决定是否提升？

### 讨论
- 否决 A：信息被锁在结果中，难以复用；
- 否决 B：自动膨胀，变量爆炸；
- 选择 C：保留人类最终决策权，按需纳入系统。

### 结论
**生命周期：active / archived + 晋升机制**
- `active`：参与依赖图、会触发传播、可修改；
- `archived`：只读存档、不触发传播、不入依赖网；
- **默认原则：AI 产出中新出现的实体/事实默认进入 `archived`，仅在确认后才可晋升为 `active`。**
- 支持 `archived → active`“晋升”，必须经确认并记录溯源。

**价值**：避免膨胀（默认不进系统网络）、保留决策权、不丢信息、过程可追溯。

---

## 问题 3：引用即依赖，但所有关系都是同一种“边”吗？

### 问题背景
原始设计把所有关系都视作“引用边”，易导致传播混乱与可视化负担。

### 挑战
> “包含关系”和“被规则引用”是同一种吗？“触发维护”和“派生来源”又是同一种吗？

### 讨论
不同关系语义与传播行为不同，需要区分与可视化过滤。

### 结论
**四种边类型**
| 边类型 | 语义 | 传播行为 | 示例 |
|---|---|---|---|
| Reference | 规则/Prompt 中引用 | ✅ 触发下游重新评估 | `{{角色A}}` 被写作规则引用 |
| Activation | A 变化激活 B 维护流程 | ✅ 触发 Maintainer 工作 | 角色死亡 → 激活相关状态更新 |
| Containment | 结构包含（父子） | ⚠️ 仅向上冒泡状态变化（脏标记），不向下游传播；是否进一步触发维护需显式规则 | 角色卡包含“姓名/性格” |
| Derivation | 派生/溯源 | ❌ 仅记录来源，不参与传播 | 第三章正文派生自角色卡+大纲 |

**价值**：UI 可按边类型过滤，传播规则清晰，契合“神经系统”隐喻。

---

## 问题 4：类型体系（CONST/SLOW/DYN）在统一抽象下怎么办？

### 问题背景
原始设计延续了传统变量类型分类：CONST（常量）、SLOW（缓慢变化）、DYN（动态）、FACT（事实）。这种分类体系假设变量有固定的“本质类型”。

### 挑战
> 如果一切都是 Context Unit，固定的四分类还有意义吗？一个变量能不能既是“人物相关”又是“高风险需审查”？

### 讨论
- 固定分类的问题：一个变量只能属于一种类型，但实际场景中往往需要多维度描述；
- 用户反馈：应该通过“标题、备注和 #标签”来分类，而非强制类型；
- 进一步区分：分类（是什么）和行为（怎么对待）是两件事，不应耦合。

### 结论
**Tag + Policy 分离**
- Tag 管“是什么”（如 `#人物` `#世界观` `#关键事实` `#不可变`）；
- Policy 管“怎么对待”（确认模式、可写 Agent、可用模型等）。

**Policy 结构示例：**
```yaml
Policy:
  confirmation_strategy: confirm_required  # immediate / confirm_required / batch_queue / auto_trusted
  allowed_maintainers: [Agent-A, Agent-B]
  allowed_models: [claude-3, gpt-4]
  requires_review: true
  max_auto_propagation_depth: 3
```

**价值**：分类与行为正交，最大灵活与可迁移性。

---

## 问题 5：`owner` 概念如何在通用系统中定义？

### 问题背景
原始设计《总体想法.md》第 11 节将 `owner` 概念列为开放问题：「在通用系统里是否应该下沉到模板？或者区分“权威源”与“维护权”」。

### 挑战
> 一个 Context Unit 的 owner 是指“谁有权修改它”还是“最终以谁的版本为准”？如果 AI Agent 提出修改建议，但人类是最终权威，如何表达这种关系？

### 讨论
- “owner”一词含义模糊：可能指创建者、维护者、或权威来源；
- 实际场景：一个角色设定可以是“人类为最终权威”+“AI Agent 负责监控和建议修改”；
- 需要分离两个概念：谁说了算（冲突时）vs 谁负责日常维护。

### 结论
**Authority Source vs Maintainer 分离**
- Authority Source：冲突时以谁为准（人类/外部文档/系统/Agent）；
- Maintainer：谁负责提更新建议（Agent/人类/规则）。

**价值**：消除歧义，权限/职责更精确。

---

## 问题 6：触发机制够不够？条件/批量/链式爆炸如何处理？

### 问题背景
原始设计列出四种触发类型：用户输入、关联变量变更、定时触发、外部调用。这些是否足够覆盖实际场景？

### 挑战
> 1. 条件触发怎么办？比如“当章节数 > 10 时自动触发某流程”。
> 2. 批量触发怎么办？用户选中一组变量统一更新。
> 3. 链式爆炸怎么办？A→B→C→D 形成长链时，如何防止无限传播？

### 讨论
- 四种触发本质是**事件源**，不是触发机制本身；
- 条件触发 = 事件源 + 条件过滤，不需要新增“第五种触发”；
- 批量触发 = 多个事件的聚合处理，应在确认中心支持；
- 链式爆炸需要系统级安全阀，而非依赖用户配置。

### 结论
**Trigger = Event + Condition + 安全阀**
- 事件源：user_input / unit_changed / time_tick / external_call / 自定义；
- 条件：可选的过滤规则，满足时才触发；
- 安全阀：最大传播深度、频率限制/去抖、`impactSet` 超阈强制人工确认。

**价值**：可扩展（不用加新触发类型）、不易过时、安全可控。

---

## 问题 7：确认机制如何结构化？

### 问题背景
原始设计提出四种确认模式：严格、宽松、批量接受、YOLO。这是正确方向，但“确认”的数据结构是什么？

### 挑战
> 用户在确认时需要知道什么？只是“接受/拒绝”吗？还是需要看到完整的变更上下文？

### 讨论
- 如果只是“是/否”开关，用户无法做出明智决策；
- 用户需要知道：要改什么（ChangeSet）、会影响谁（ImpactSet）、为什么要改（Reason）；
- 四种模式本质是“谁可以自动提交 ChangeSet”和“什么情况必须人工介入”的策略配置。

### 结论
**ChangeSet + ImpactSet + Reason**
- ChangeSet：要改哪些单元、改成什么（可拆分接受）；
- ImpactSet：会影响哪些下游（按层级/边类型计算）；
- Reason：为何而改（触发事件 trigger、证据引用 evidence refs、可选 agent rationale）。
- 执行策略：immediate / confirm_required（默认）/ batch_queue / auto_trusted。

**价值**：让确认基于完整上下文，不是“是/否”开关；策略可按 Tag/Policy 精细配置。

---

## 问题 8：溯源系统应当是模块还是地基？

### 问题背景
原始设计将溯源称为“系统灵魂”，并列为十大模块之一。但这种定位是否足够？

### 挑战
> 如果溯源只是“模块之一”，它就可能被绕过或事后补充。如何让“强制溯源”成为架构必然而非约定？

### 讨论
- 溯源不应是“事后记录”，而应是“系统事实的定义方式”；
- Event Sourcing 架构：当前状态 = 所有事件的折叠结果；
- 这样溯源不是可选功能，而是数据存储的本质——无法绕过。

### 结论
**Event Sourcing 作为溯源底座**
- Event Store：不可变事件记录（谁在何时因何对哪些单元做了什么）；
- Projection：当前状态视图（由事件计算得出）；
- Snapshot：性能优化的状态快照；
- 统一 Query API：UI 和 Agent 共用同一套查询接口。

**价值**：强制溯源（架构必然）、时间旅行、Undo/Redo、Diff 对比、统一查询模型。

---

## 问题 9：模板/配置包会过时，如何安全升级？

### 问题背景
核心哲学是“系统不过时，模板可过时”。但如果官方小说模板升级了 schema，用户项目怎么办？

### 挑战
> 一年后你升级官方模板，用户的旧项目能否平滑迁移？如果不能，核心愿景就会被破坏。

### 讨论
- 没有 migration 机制，模板升级会导致用户项目崩溃；
- 这不是“以后再考虑”的问题，必须从第一天设计；
- 配置包需要版本号 + 迁移规则。

### 结论
**版本化 + Migration**
- 配置包声明 `schemaVersion`；
- 提供从旧版本的迁移脚本/规则（哪怕初期很简单）；
- 系统在加载配置包时检测版本，自动或提示运行迁移。

**价值**：兑现“系统不过时，模板可过时”的核心承诺；用户项目可安全升级。

---

## 问题 10：与 Dify 的本质区别是什么？（范式转换）

### 范式对比
| 维度 | Dify 范式 | 本项目范式 |
|---|---|---|
| 构建思维 | 流程图：A→B→C | 有机系统：设计器官与相互作用 |
| 核心问题 | 下一步做什么 | 系统需要哪些器官，如何协同 |
| 执行模型 | 顺序/并行 | 事件驱动 + 相互响应 |
| 变更模型 | 手动触发下一步 | 变更沿“神经系统”传播 |

### 结论（UI 必须体现）
- 主视图=“系统蓝图”，卡片=“器官”，连线=“神经/血管”，溯源=“神经系统扫描”；
- 构建起点是“需要哪些器官”，而非“第一步做什么”。

---

## Phase 1 总结：11 个核心设计决策

| # | 决策 | 核心理由 |
|---|---|---|
| 1 | 统一为 Context Unit | 降复杂、机制复用 |
| 2 | active/archived + 晋升 | 避免膨胀、保决策权、可追溯 |
| 3 | 四类边（Reference/Activation/Containment/Derivation） | 传播清晰、可视化友好 |
| 4 | Tag + Policy | 分类/行为正交、可迁移 |
| 5 | Authority vs Maintainer | 权限与职责精确化 |
| 6 | Event + Condition + 安全阀 | 可扩展与安全控制 |
| 7 | ChangeSet/ImpactSet/Reason | 基于完整信息的确认 |
| 8 | Event Sourcing | 溯源为地基、时间旅行 |
| 9 | 包版本化 + Migration | 模板安全升级 |
| 10 | UI 范式：有机系统蓝图 | 引导正确心智模型 |
| 11 | Agent 推理溯源 | 底层记录一切、用户层选择性开放 |

---

## 待议事项（后续 Phase 讨论）

**✅ 已解决：术语冲突**
- 原“Trigger 边”已改名为 **Activation Edge**（表示“A 变化激活 B 维护流程”的关系）
- **Trigger Rule**：Event + Condition + 安全阀（规则配置对象）
- 两者现已明确区分

**✅ 已补充：Containment 边的向下广播规则**

**ContainmentEdge 的可选配置：**
```
├── bubbleUp: true（默认）——子级变化向上冒泡脏标记
├── broadcastDown: false（默认）——父级变化不自动向下广播
└── broadcastRules: [可选] 定义哪些父级属性变化时广播给哪些子级
```

**配置示例：**
```yaml
# 势力卡的 Containment Policy
containment:
  broadcastRules:
    - when: "阵营立场" changed
      notify: all children with tag #角色
      action: suggest_review  # 建议审查，而非强制更新
```

**设计原则：**
- 默认行为简单（不广播），降低认知负担
- 高级用户可配置复杂规则
- 广播动作默认为 `suggest_review`（建议审查），而非 `force_update`（强制更新），保留人类决策权

---

## 问题 11：Agent 推理过程如何纳入溯源？

### 问题背景
每个变量/卡片可以绑定 AI Agent 负责。当 Agent-A 更新变量 X 后，关联变量的 Agent-B 会收到通知。Agent-B 可能需要查询“Agent-A 为什么更新 X”来做出更准确的判断。

### 挑战
> 溯源系统是否需要记录 Agent 的完整推理过程？Agent 对溯源系统的查询本身是否也要记录？

### 讨论
- 底层设计理念：**底层记录一切，用户层选择性开放**
- Agent 的完整推理过程（Chain of Thought）应该记录
- Agent 的溯源查询行为本身也应该记录
- 这符合核心理念：高度模块化底层 + 高度用户自定义 = 立于不败之地

### 结论
**Event 结构扩展：**
```yaml
Event:
  # 基础信息
  timestamp, actor, action, subject, changes
  
  # Agent 推理上下文（新增）
  reasoning:
    trigger_event: 触发此次更新的事件
    inputs_read: Agent 读取了哪些 Context Unit
    traceability_queries: Agent 做了哪些溯源查询
    chain_of_thought: Agent 完整推理过程
    conclusion: 最终结论
  
  # 传播信息（新增）
  notifications_sent: 通知了哪些下游 Agent
```

**Agent 溯源查询 API：**
| 查询 | 权限级别 |
|------|----------|
| `getLatestChange(X)` | 基础 |
| `getChangeTrigger(X)` | 标准 |
| `getAgentRationale(X)` | 高级 |
| `getInputsConsidered(X)` | 高级 |
| `traceBack(X, depth)` | 完整 |

**关键原则：**
- 底层 Event Store 记录一切（推理过程、查询行为、通知记录）
- 用户可配置每个 Agent 的查询权限级别
- 用户可在 Agent 规则（Prompt）中指定何时调用溯源查询

**价值：**Agent 间协作有据可查、可追溯、可调试；用户可按需控制复杂度。
