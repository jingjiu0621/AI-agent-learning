# 8.5.7 Hybrid Patterns — 混合模式

## 1. 简单介绍

Hybrid Patterns（混合模式）是指将多个基础 Swarm Pattern（Manager-Worker、Pipeline、Market、Mesh、Parliament）组合成多层架构的复合组织模式。现实世界中的多智能体系统几乎从不使用单一纯模式——就像一家公司不会只有组织结构图而没有工作流程、没有市场策略一样，复杂的多 Agent 系统需要组合多种模式来应对不同层面的需求。

混合模式的核心思想是：**不同模式解决不同问题**。Manager-Worker 擅长任务分解与分配，Pipeline 擅长阶段化处理，Market 擅长资源优化配置，Mesh 擅长动态协作，Parliament 擅长集体决策。将这些模式按层次组合，可以构建出远超单一模式能力范围的系统。

```
     ┌─────────────────────────────────────────────────────────────┐
     │                   混合模式 (Hybrid Patterns)                  │
     │  将多种 Swarm Pattern 组合为分层架构,解决复杂现实问题          │
     └─────────────────────────────────────────────────────────────┘
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         ▼                          ▼                          ▼
   ┌─────────────┐          ┌─────────────┐          ┌─────────────┐
   │ 战略决策层    │          │ 任务组织层    │          │ 执行调度层    │
   │ Parliament   │          │ Manager-W.   │          │ Market / Mesh│
   │ 集体制定策略  │          │ 分解与分配    │          │ 资源调度/协作  │
   └─────────────┘          └─────────────┘          └─────────────┘
                                       │
                                       ▼
                              ┌─────────────────────┐
                              │ 阶段处理层            │
                              │ Pipeline             │
                              │ 流水线阶段化处理       │
                              └─────────────────────┘
```

## 2. 基本原理

### 2.1 分层架构

混合模式的基本原理是**分层组合（Layered Composition）**：系统的不同层次使用不同的组织模式，每个层次解决不同粒度的组织问题。层次之间的通信通过定义良好的接口进行。

```
                    ┌──────────────────────────────────────────┐
                    │           混合模式分层架构                  │
                    └──────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  模式层 A: 战略决策                                                │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  Parliament Pattern                                       │   │
  │  │  多个 Agent 通过投票/辩论决定: "做什么"、"优先级"、"策略"    │   │
  │  │  Interface: decision(topic, options) -> chosen_option     │   │
  │  └──────────────────────┬───────────────────────────────────┘   │
  │                         │ 策略决策输出                          │
  │                         ▼                                       │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  适配层: 策略 → 任务                                       │   │
  │  │  Translates high-level strategy into concrete tasks       │   │
  │  └──────────────────────┬───────────────────────────────────┘   │
  │                         │ 任务列表                              │
  │                         ▼                                       │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  Manager-Worker Pattern                                   │   │
  │  │  Manager 将策略层任务分解为子任务,分配给 Worker Agent       │   │
  │  │  Interface: decompose(task) -> [subtasks]                 │   │
  │  │             assign(subtask, worker) -> result             │   │
  │  └──────────────────────┬───────────────────────────────────┘   │
  │                         │ 子任务流                              │
  │                         ▼                                       │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  模式层 C: 阶段处理                                        │   │
  │  │  Pipeline Pattern                                         │   │
  │  │  Manager 输出的子任务按阶段流入 Pipeline                    │   │
  │  │  Interface: process(stage, input) -> output               │   │
  │  └──────────────────────┬───────────────────────────────────┘   │
  │                         │ 处理中需要资源                          │
  │                         ▼                                       │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  模式层 D: 资源调度                                        │   │
  │  │  Market Pattern                                           │   │
  │  │  Pipeline 阶段内的计算任务通过 Market 竞标分配计算资源       │   │
  │  │  Interface: bid(task) -> (worker, price)                  │   │
  │  └──────────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────┘
```

### 2.2 五种常见混合组合

| 混合模式 | 组合方式 | 适用场景 | 典型架构 |
|----------|----------|----------|----------|
| **Manager-Pipeline** | Manager 分解任务，Pipeline 分阶段执行 | 软件开发、内容生产流水线 | 上层分解 + 下层串行处理 |
| **Market-Mesh** | Mesh 负责发现和拓扑，Market 负责交易 | 云计算资源调度、众包平台 | 动态发现 + 竞价分配 |
| **Parliament-Manager** | Parliament 制定政策，Manager 执行 | 组织治理、系统策略管理 | 集体决策 + 集中执行 |
| **Pipeline-Mesh** | Pipeline 作为主流程，Mesh 处理异常/旁路 | 数据管道、事件处理系统 | 主线串行 + 旁路协作 |
| **Full Hybrid** | 三层以上组合全部模式 | 大型企业级多 Agent 系统 | 完整分层架构 |

### 2.3 组合原则

```
┌─────────────────────────────────────────────────────┐
│              混合模式组合的黄金原则                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. 每层解决一个维度的组织问题                          │
│     - 战略层: 做什么 (What)                          │
│     - 组织层: 谁来做 (Who)                           │
│     - 执行层: 怎么做 (How)                           │
│     - 调度层: 用什么做 (With What)                   │
│                                                     │
│  2. 层间接口必须比内部接口更稳定                        │
│     - 内部可以频繁重构，层间契约需要版本管理             │
│                                                     │
│  3. 每层内部隔离失败                                  │
│     - Pipeline 的某个 stage 故障不应影响 Parliament     │
│                                                     │
│  4. 避免模式冲突                                      │
│     - Market 的完全自由竞争 vs Manager 的集中控制       │
│     - 同一层不能同时使用矛盾的模式                      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## 3. 背景与发展

### 3.1 混合模式的兴起

混合模式的产生是自然演化而非理论设计的结果。当开发者开始用多 Agent 系统解决实际问题时，一个无可回避的事实浮现出来：**现实问题从来不会恰好符合单一模式**。

```
时间线: 单一模式 → 模式组合 → 分层架构

2023 初                    2023 末                      2024+
单一 Swarm Pattern         Pattern 组合                  分层混合架构
─────────────              ─────────────                ─────────────
                           
Manager-Worker             Manager-Worker               Parliament Layer
  │                        + Pipeline                      │
  │                          │                          Manager Layer
Pipeline                   Manager-Worker                   │
  │                        + Mesh                       Pipeline Layer
  │                          │                              │
Market                     Parliament                    Market Layer
  │                        + Market                        │
Mesh                           │                        Mesh Layer
                            Three-layer hybrid
```

### 3.2 思想渊源

混合模式的设计思想来源于多个成熟领域：

- **TOGAF 企业架构**：分层架构的思想——企业架构标准 TOGAF 的核心是"架构开发方法（ADM）"，强调不同层次的架构（业务、数据、应用、技术）使用不同的模式。混合模式将这一思想引入多 Agent 系统。

- **微服务架构模式**：微服务中的"API Gateway + Service Mesh + Backend Services"三层结构与混合模式高度相似——每层解决不同问题，通过定义良好的 API 通信。

- **组织设计理论**：亨利·明茨伯格的组织结构理论指出，真实组织的结构是"混合体"——既有职能层级（Manager-Worker），又有跨职能项目组（Mesh），还有内部市场化机制（Market）。

- **计算机体系结构**：计算机系统本身就是经典的混合模式——指令流水线（Pipeline）、多核并行（Mesh）、分级存储（Market-like cache allocation）。

## 4. 之前做法与局限

在混合模式成为显式设计思想之前，多 Agent 系统开发者通常采用"押注单一模式"的策略：选择一个模式并在其框架内解决所有问题。这种做法带来了一系列系统性局限。

### 4.1 纯 Pipeline 的局限

```
纯 Pipeline 模式的典型困境:

 输入 → Stage A → Stage B → Stage C → 输出
         │          │          │
         └──────────┴──────────┘
                  ❌
  问题: Stage B 发现需要 Stage A 没有准备好的数据
  方案: 只能让 Stage B 等待或从头重新执行
  如果用了 Mesh: Stage B 可以直接请求其他 Agent 补充数据
```

**Pipeline 的核心假设**是"任务可以线性分阶段处理"。但当某个阶段需要动态调整、需要回溯或需要并行扩展时，Pipeline 的线性结构就成为瓶颈。纯 Pipeline 无法处理：
- 需要动态分配资源的计算密集型阶段
- 需要多 Agent 协作的非线性子问题
- 异常处理需要回溯到前序阶段

### 4.2 纯 Manager-Worker 的局限

```
纯 Manager-Worker 的典型困境:

        Manager
      ↙  ↓  ↘
    W1   W2   W3
    │    │    │
    └────┴────┘
        ❌
  问题: W1 的输出需要作为 W2 的输入
  Manager: 必须等待 W1 完成再调度 W2
  如果用了 Pipeline: Worker 之间可以直接传递中间结果
```

**Manager-Worker 的核心假设**是"Manager 可以且应该管理所有任务分配"。但当 Worker 之间存在依赖关系时，Manager 就退化为一个笨重的 Pipeline 调度器——它本不该承担这个角色。

### 4.3 纯 Market 的局限

```
纯 Market 模式的典型困境:

  Task T1: 竞价 Agent A $10, Agent B $12 → A 获得
  Task T2: 竞价 Agent C $8,  Agent D $15 → C 获得
  Task T3: 需要 A + C 协作 (复合技能)
  
  问题: 没有机制处理复合任务和 Agent 间协作
  Market 只擅长独立任务的竞价分配
  如果用了 Mesh: Agent 之间可以自由组合协作
```

**Market 的核心假设**是"任务可以独立竞价"。但对需要 Agent 间协作的复合任务，Market 缺乏编排能力——它只能分配，不能组织。

### 4.4 纯 Mesh 的局限

```
纯 Mesh 模式的典型困境:

  Agent数量 → 通信连接数 = n(n-1)/2
  n=5  → 10 条连接  ✓  manageable
  n=10 → 45 条连接  ⚠  noisy
  n=50 → 1225 条连接 ❌  explosion
```

**Mesh 的核心假设**是"Agent 之间可以直接通信协作"。但在规模扩大时，全连接 Mesh 的通信复杂度是 O(n^2)，无法扩展。Mesh 也缺乏全局视角——每个 Agent 只知道邻居，不知道整体状态。

### 4.5 纯 Parliament 的局限

```
纯 Parliament 模式的典型困境:

  每个决策都需要 → 提案 → 辩论 → 投票 → 裁决
  即使是很简单的操作:
    "今天天气怎么样?"
    也要经过完整的议会流程
  
  问题: 决策开销太大,不适合高频操作
  如果用了 Manager: 日常操作由 Manager 直接决策,只有重大策略才用 Parliament
```

**Parliament 的核心假设**是"集体决策优于个人决策"。但集体决策的成本远高于个体决策——纯 Parliament 系统会因过度民主而瘫痪。

### 4.6 单一模式 vs 混合模式对比

| 维度 | 单一模式 | 混合模式 |
|------|----------|----------|
| 问题覆盖范围 | 仅限模式设计范围内的场景 | 几乎可以覆盖任意复杂场景 |
| 扩展性 | 受模式固有瓶颈限制 | 可以通过增加模式层扩展 |
| 实现复杂度 | 低到中 | 中到高 |
| 运维复杂度 | 低 | 中到高 |
| 性能优化空间 | 受限于单一通信/协作模式 | 每层可独立优化 |
| 故障隔离 | 模式内故障可能影响全局 | 层间隔离，故障受限 |
| 适应变化能力 | 弱——模式假设限制了变化 | 强——可通过切换层模式适应 |

## 5. 核心矛盾

混合模式虽然强大，但引入了一系列新型矛盾。这些矛盾不是 Bug，而是设计决策中必须 trade-off 的维度。

### 5.1 模式集成复杂度 vs 专业化收益

```
                   集成成本曲线
收益/成本
   │
   │    收益(专业化分工的好处)
   │   ┌─────────────────────
   │   │                    ╱
   │   │                   ╱
   │   │                  ╱
   │   │                 ╱   ← 交叉点: 每增加一个模式,
   │   │                ╱      边际收益递减,边际成本递增
   │   │               ╱
   │   │              ╱   成本(集成、调试、运维)
   │   │             ╱
   │   │            ╱
   │   │           ╱
   │   └───────────────────────────────
   │    1     2     3     4     5    模式数量
```

**矛盾本质**：每增加一个模式层，系统获得该模式的专业化收益，但必须付出集成成本——两个模式之间的接口设计、数据转换、错误处理、调试复杂度。当模式数量超过某个阈值后，集成成本超过专业化收益。

**应对策略**：
- 只添加"当前问题迫切需要"的模式，不要为了"可能有用"而添加
- 定义清晰的接口契约，降低集成成本
- 考虑模式之间的自然亲和力（如 Pipeline 和 Manager-Worker 天然互补）

### 5.2 层耦合 — 模式间的涟漪效应

```
                   层耦合的涟漪效应

  ┌─────────────────────────────────────────┐
  │  Parliament: "改变策略为 QPS > 100 时    │
  │              降低响应精度"               │
  └────────────────┬────────────────────────┘
                   │ 策略变更
                   ▼
  ┌─────────────────────────────────────────┐
  │  Manager: 需要重新编排 Worker 的优先级    │ ← 被影响
  └────────────────┬────────────────────────┘
                   │ 任务分配变更
                   ▼
  ┌─────────────────────────────────────────┐
  │  Pipeline: 某些 stage 需要调整超时设置    │ ← 被影响
  └────────────────┬────────────────────────┘
                   │ 阶段处理变更
                   ▼
  ┌─────────────────────────────────────────┐
  │  Market: 需要更改资源竞价的 QoS 参数      │ ← 被影响
  └─────────────────────────────────────────┘
```

**矛盾本质**：系统的层次不是完全正交的。上层策略变化必然影响下层的执行参数。如果层间耦合过紧，一次小小的策略调整可能导致全系统大规模重构。

**应对策略**：
- 层间接口要"稳定且粗粒度"——每次层间调用携带足够上下文，减少调用次数
- 引入"配置层"——上层决策输出为配置参数而非命令，下层自适应
- 采用事件驱动而非命令驱动的层间通信

### 5.3 模式冲突 — 不相容的假设

不同的模式建立在不同的假设之上。当这些假设在同一个系统中并存时，可能产生冲突。

| 模式组合 | 冲突点 | 具体表现 |
|----------|--------|----------|
| Manager-Worker + Market | 集中控制 vs 自由竞争 | Manager 指定 Worker 做某事 vs Market 让 Worker 自主竞价 |
| Parliament + Manager | 民主决策 vs 集中执行 | 议会否决了 Manager 正在执行的计划 |
| Pipeline + Mesh | 线性顺序 vs 动态互联 | Pipeline 要求 stage 3 等待 stage 2 完成，但 Mesh 让它们直接通信 |
| Market + Mesh | 交易成本 vs 协作成本 | Market 的每次交互都要竞价，但 Mesh 假设协作是免费的 |

**应对策略**：
- **层次分离**：将冲突的模式放在不同层次，避免直接交互
- **Anti-Corruption Layer（防腐层）**：在冲突模式之间插入转换层
- **时间隔离**：不同时间阶段使用不同模式（如规划阶段用 Parliament，执行阶段用 Pipeline）

### 5.4 Debugging 复杂度

混合模式的最大实际矛盾不是设计阶段的，而是运维和调试阶段的。当系统出问题时，问题可能来自：

```
问题表现: Pipeline Stage C 输出质量下降

可能原因:
  ├── 模式层 A (Parliament): 最近改变了策略,影响了质量预期
  ├── 模式层 B (Manager): Worker 分配不合理
  ├── 模式层 C (Pipeline): Stage 内部的 Agent 出了 Bug
  ├── 模式层 D (Market): 竞价机制导致低质 Worker 中标
  ├── 层间通信: 接口数据格式不兼容
  ├── 层间时序: 策略变更还未传播到执行层
  └── 外部因素: 数据源质量下降

每个可能原因属于不同的模式层,需要不同的调试方法和工具链。
```

## 6. 主流优化方向

### 6.1 模式接口契约（Pattern Interface Contract）

为每层模式定义标准化的接口契约，使层与层之间的交互可预测、可测试、可替换。

```
┌─────────────────────────────────────────────────────────────┐
│                  模式接口契约 (PIC) 规范                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Pattern Interface = {                                       │
│    identity: {                                               │
│      name: "pipeline-v2",                                   │
│      type: "Pipeline",                                      │
│      version: "2.1.0"                                       │
│    },                                                        │
│    provided: [   // 该模式对外提供的服务                      │
│      { name: "process", input: StageInput, output: StageOut }│
│    ],                                                        │
│    required: [   // 该模式需要的下层支撑                      │
│      { name: "schedule", contract: "Market-API-v1" }         │
│    ],                                                        │
│    assumptions: [  // 模式的假设（用于冲突检测）              │
│      "stages_execute_in_order",                             │
│      "no_side_effects_between_stages"                       │
│    ],                                                        │
│    constraints: {  // 约束条件                               │
│      max_concurrency: 10,                                    │
│      max_stage_duration_ms: 30000                           │
│    }                                                         │
│  }                                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 防腐层（Anti-Corruption Layer）

当两个模式的假设不兼容时，在它们之间插入一个适配层，负责翻译、隔离和冲突消解。

```
┌──────────────┐         ┌──────────────────┐         ┌──────────────┐
│              │         │   Anti-Corruption  │         │              │
│   Manager    │ ──────▶ │       Layer       │ ──────▶ │    Market    │
│              │         │                   │         │              │
│  "Agent B,   │         │  Manager 的命令式  │         │  "Task #42   │
│   执行任务X"  │         │   请求转化为       │         │   开放竞价"   │
│              │         │   Market 的市场    │         │              │
│  命令式       │         │   化请求           │         │  竞拍式       │
│  集中控制     │         │                   │         │  自由竞争     │
│              │         │  同时: Market 的   │         │              │
│              │         │   结果转化为        │         │              │
│              │         │   Manager 可理解   │         │              │
│              │ ◀────── │   的状态报告        │ ◀────── │              │
└──────────────┘         └──────────────────┘         └──────────────┘

防腐层的具体职责:
  1. 协议翻译: Manager 的"命令" → Market 的"标书"
  2. 假设修补: 补偿双方模式假设的差异
  3. 语义映射: 一方的事件对应另一方的动作
  4. 失败隔离: Market 的异常不会渗透到 Manager 层
```

### 6.3 层次化模式组合

通过 hierarchical composition 而不是 flat composition 来组合模式。外层模式"包含"内层模式。

```
                   外层模式: Parliament
                   ┌─────────────────────────────┐
                   │  策略决策                     │
                   │                             │
                   │  中层模式: Manager-Worker    │
                   │  ┌───────────────────────┐  │
                   │  │  Manager               │  │
                   │  │  ┌─────────────────┐   │  │
                   │  │  │ 内层: Pipeline   │   │  │
                   │  │  │ S1→S2→S3→S4     │   │  │
                   │  │  └─────────────────┘   │  │
                   │  │  ┌─────────────────┐   │  │
                   │  │  │ 内层: Mesh      │   │  │
                   │  │  │ A↔B↔C 协作      │   │  │
                   │  │  └─────────────────┘   │  │
                   │  └───────────────────────┘  │
                   │                             │
                   │  底层模式: Market            │
                   │  ┌───────────────────────┐  │
                   │  │   Agent A  $10/hr      │  │
                   │  │   Agent B  $12/hr      │  │
                   │  │   Agent C   $8/hr      │  │
                   │  └───────────────────────┘  │
                   └─────────────────────────────┘

关键: 每层只与相邻层交互,不跨层直接调用
```

### 6.4 阶段感知模式切换

根据任务所处的阶段动态切换模式。同一系统在不同时间使用不同的模式。

```
  Task Phase:  Analysis  →  Planning  →  Execution  →  Verification
                    │            │             │             │
  Active Pattern:   │            │             │             │
                    ▼            ▼             ▼             ▼
               ┌────────┐  ┌────────┐   ┌──────────┐  ┌──────────┐
               │  Mesh  │  │Parliament│  │Pipeline  │  │Parliament │
               │自由探索│  │集体决策  │  │流水线执行 │  │质量评审  │
               │头脑风暴│  │投票表决  │  │分阶段处理 │  │集体裁决  │
               └────────┘  └────────┘   └──────────┘  └──────────┘
                      \        |            /              /
                       \       |           /              /
                        ───────┼───────────              /
                               │                        /
                               └──── 共享 Manager ─────┘
                                    (负责模式切换)
```

### 6.5 模式选择决策树

在模式接口契约的支持下，可以建立决策树来指导"什么时候增加一个模式层"。

```
                   是否需要新的组织维度？
                           │
                     ┌─────┴─────┐
                     │           │
                     Yes         No ──▶ 维持现有模式
                     │
                     ▼
              当前模式是否能满足？
               ┌─────┴─────┐
               │           │
               Yes         No
               │           │
               ▼           ▼
         扩展当前模式   问题属于哪个维度？
         而非加新模式     │
                    ┌─────┼─────────┬─────────┐
                    │     │         │         │
                    ▼     ▼         ▼         ▼
                 战略    组织      执行      资源
                 │       │         │         │
                 ▼       ▼         ▼         ▼
             Parliament Manager  Pipeline  Market
                       │         │
                       │         │
                       └────┬────┘
                            │
                    需要动态协作?
                            │
                      ┌─────┴─────┐
                      │           │
                      Yes         No
                      │           │
                      ▼           ▼
                     Mesh    仅 Pipeline
```

## 7. 实现挑战

### 7.1 跨模式通信翻译

不同模式使用不同的"语言"：Manager-Worker 用命令式语言（"去做X"），Market 用竞价语言（"谁愿意以Y价格做X"），Mesh 用协商语言（"我们一起来做X"）。在混合系统中，需要在这些模式语言之间进行翻译。

```python
# 挑战示例: Manager 的"命令"需要翻译为 Market 的"招标"

# Manager 风格:
manager_says = "Agent_42, execute task T-1001 immediately with high priority."

# Market 风格需要:
market_needs = {
    "task_id": "T-1001",
    "type": "auction",
    "reserve_price": None,  # Manager 没给预算!
    "qualifications": [],   # Manager 没说需要什么技能!
    "deadline": None,       # Manager 没给截止时间!
    "qos_requirements": {}  # Manager 说 high priority, 但 Market 需要量化指标
}

# 翻译层需要补充这些缺失的信息,将命令式语义转化为市场语义
```

### 7.2 跨模式状态一致性

当多个模式层维护自己的状态时，保持跨层状态一致性是巨大的挑战。

```
时间线:                      T1         T2          T3
                        ┌─────────┬──────────┬──────────┐
Parliament 状态:        │ 策略 v1  │ 策略 v1   │ 策略 v2  │
Manager 状态:           │ 任务分配  │ 任务执行中 │ 任务执行中│
Pipeline 状态:          │ Stage 2  │ Stage 3   │ Stage 3  │
Market 状态:            │ 竞价中    │ 已分配     │ 已分配    │
                        └─────────┴──────────┴──────────┘

问题: T3 时 Parliament 改变了策略,但 Manager/Pipeline/Market
      仍然在执行基于策略 v1 分配的任务。

需要处理:
  - 进行中的任务是否应该被新策略影响?
  - 应该"立即切换"还是"优雅过渡"?
  - 新旧策略之间的状态如何迁移?
```

### 7.3 跨模式调试

传统调试工具针对单模式系统设计。混合模式系统中，问题可能跨越多个模式层，现有工具无法追踪跨模式的调用链。

```
传统调试:
  [Pipeline] Stage 3: 错误! 输入数据格式异常.
  开发者: Stage 2 的输出有问题吗? → 检查 Stage 2 → 没问题
  开发者: 那是 Market 分配错了 Agent? → 检查 Market → 没问题
  开发者: 是 Manager 的输入错了? → 检查 Manager → 没问题
  开发者: 是 Parliament 的策略导致输入变了? → 检查 Parliament → 确实是!

现代方案:
  分布式追踪 + 模式上下文标签
  ┌────────────────────────────────────────────┐
  │ Trace ID: abc-123                          │
  │ Span 1: Parliament.decide (policy_v2)      │
  │ Span 2: Manager.decompose (task_42)        │
  │ Span 3: Pipeline.process (stage_3)         │ ← 错误在这里
  │ Span 4: Market.allocate (agent_7)          │
  │                                            │
  │ Context: { pattern: "Pipeline",             │
  │            layer: "execution",             │
  │            parent_pattern: "Manager" }     │
  └────────────────────────────────────────────┘
```

### 7.4 跨层性能优化

每层的优化目标可能冲突。Market 追求资源利用率最大化（让 Agent 始终忙碌），但 Pipeline 追求低延迟（让 Agent 始终可用）。

```
优化冲突示例:

  Market 优化目标: 资源利用率 > 95%
    → Agent 总是被分配任务,几乎不空闲
    → Pipeline 的 stage 2 需要立即执行时没有可用 Agent!
    → 端到端延迟增加了 300%

  Pipeline 优化目标: P99 延迟 < 100ms
    → 需要预留 20% 的 Agent 作为热备
    → Market 的资源利用率降到 75%
    → 资源浪费!

  解决方案: 分层优化 + 全局约束
    - Market 层: 利用率 > 80% (局部目标,有约束)
    - Pipeline 层: P99 < 200ms (局部目标,有约束)
    - 全局: 利用率 > 80% AND P99 < 200ms (全局约束)
```

### 7.5 配置复杂度

N 个模式组合意味着 N 套配置系统，以及 N(N-1)/2 对交互配置。

```
单一模式配置: 1 个配置文件
  pipeline:
    stages: [A, B, C]
    timeout: 30s

混合模式配置: ~ 7+ 个配置文件
  parliament:
    rules: ...
    voting: ...
  manager:
    workers: ...
    decompose_strategy: ...
  pipeline:
    stages: ...
    timeout: ...
  market:
    pricing: ...
    bidding: ...
  layer_a_b_adapter:
    mappings: ...
    error_handling: ...
  layer_b_c_adapter:
    mappings: ...
    error_handling: ...
  global:
    tracing: ...
    logging: ...
```

## 8. 能力与边界

### 8.1 能力

混合模式的核心理念是"组合超越个体"：通过组合不同模式的能力，混合架构可以覆盖单模式无法企及的场景范围。

| 能力维度 | 说明 |
|----------|------|
| **任务复杂度** | 可以处理需要多层次分解的超复杂任务（如软件系统开发、大型研究项目） |
| **规模弹性** | 不同模式层可以独立扩缩——Parliament 层只需要少量决策 Agent，Pipeline 层可以大幅扩展 |
| **场景适应** | 不同阶段、不同任务类型可以切换到最合适的模式 |
| **容错能力** | 层间隔离防止故障传播，一个模式层的崩溃不意味整个系统崩溃 |
| **组织灵活性** | 可以根据问题特性"拼装"最适合的模式组合 |

### 8.2 边界

混合模式不是银弹，它在以下场景中可能不是最佳选择：

```
                复杂度收益曲线

系统价值
   │
   │             混合模式优势区
   │               ┌────┐
   │              ╱      ╲
   │             ╱        ╲
   │            ╱          ╲
   │           ╱            ╲
   │ 单一模式  ╱              ╲    过度工程区
   │   优势区 ╱                ╲ ──────
   │         ╱                  ╲
   │        ╱                    ╲
   └──────────────────────────────────────────
     低                   系统复杂度                    高
            ↑                          ↑
         简单任务                  复杂任务
         2-3 个 Agent             20+ 个 Agent
         确定性流程                高度不确定性
```

**不适合使用混合模式的场景**：

1. **简单确定性任务**：如果任务可以用 2-3 个 Agent 和一个简单的 Pipeline 完成，混合模式是过度工程。
2. **原型阶段**：早期原型应该用最简单的模式快速验证，等系统稳定后再引入混合架构。
3. **小规模团队**：混合模式的运维复杂度要求专门的团队维护。
4. **时间敏感型快速交付**：混合模式的设计、实现和调试周期远比单模式长。

### 8.3 危险的过度工程信号

```
⚠ 过度工程警告信号:

1. 你正在为"可能需要的"未来场景添加模式层
   → 正确做法: 只为"当前需要"添加

2. 你发现自己在写"模式配置"的配置的配置
   → 正确做法: 简化,移除不必要的抽象层

3. 团队中没有人能说清系统到底用了多少种模式
   → 正确做法: 文档化架构并定期审查

4. 每次 Bug 修复都需要修改 3 个以上的模式层
   → 正确做法: 检查层间接口是否过于脆弱

5. 模式之间的适配器代码比模式本身的代码还多
   → 正确做法: 重新考虑模式选择,可能选择了不兼容的模式
```

## 9. 常用混合架构

### 9.1 Manager-Pipeline 混合

**结构**：Manager 负责任务分解和 Worker 管理，Pipeline 负责子任务的阶段化处理。这是最常见、最实用的混合模式。

```
                        Manager-Pipeline 混合架构

                     ┌─────────────────────────────┐
                     │         Manager Agent        │
                     │  任务分解 + Worker 分配 +    │
                     │  进度跟踪 + 质量管理         │
                     │                              │
                     │  decompose(task) -> subtasks │
                     └────┬────┬────┬────┬─────────┘
                          │    │    │    │
                    ┌─────┘    │    │    └─────┐
                    ▼          ▼    ▼          ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Subtask  │ │ Subtask  │ │ Subtask  │
              │    A     │ │    B     │ │    C     │
              └────┬─────┘ └────┬─────┘ └────┬─────┘
                   │            │            │
                   ▼            ▼            ▼
              ┌─────────────────────────────────────────┐
              │           Pipeline Stage 1               │
              │   [Research] → [Draft] → [Review]       │
              │          并行处理所有 subtask            │
              └─────────────────┬───────────────────────┘
                                │ 合并中间结果
                                ▼
              ┌─────────────────────────────────────────┐
              │           Pipeline Stage 2               │
              │   [Integrate] → [Polish] → [Verify]     │
              └─────────────────┬───────────────────────┘
                                │ 最终交付
                                ▼
              ┌─────────────────────────────────────────┐
              │            输出: 完整结果                 │
              └─────────────────────────────────────────┘

  Manager 职责:
    - 将用户请求分解为可 Pipeline 处理的子任务
    - 监控 Pipeline 进度,处理异常
    - 在 Pipeline 完成后合并结果

  Pipeline 职责:
    - 专注处理"阶段化转换"——将输入逐步转化为输出
    - 每个阶段可以并行或串行
    - 不关心"为什么做这个",只关心"怎么做"

  典型流程:
    1. 用户: "写一篇关于 AI 安全的博客"
    2. Manager 分解: [研究, 大纲, 初稿, 配图, 审校]
    3. Pipeline Stage 1: 研究并行,产出研究报告
    4. Pipeline Stage 2: 根据研究报告写大纲
    5. Pipeline Stage 3: 根据大纲写初稿
    6. Manager 审查初稿质量
    7. Pipeline Stage 4: 配图
    8. Pipeline Stage 5: 审校
    9. Manager 汇总交付
```

### 9.2 Market-Mesh 混合

**结构**：Mesh 负责 Agent 之间的发现和拓扑维护，Market 负责具体任务的竞价分配。

```
                    Market-Mesh 混合架构

                    ┌─────────────────────────┐
                    │       Mesh Layer          │
                    │  Agent 发现与拓扑维护     │
                    │                           │
                    │    A ─── B                │
                    │    │ \   /│               │
                    │    │  \ / │               │
                    │    C ─── D                │
                    │                           │
                    │  维护: heartbeat, status,  │
                    │  capability index, routing │
                    └───────────┬───────────────┘
                                │ Agent 能力注册
                                │ 可用状态更新
                                ▼
                    ┌─────────────────────────┐
                    │       Market Layer        │
                    │  任务竞价与资源分配       │
                    │                           │
                    │  Task T1 → 竞价 → Agent A │
                    │  Task T2 → 竞价 → Agent C │
                    │  Task T3 → 竞价 → Agent B │
                    │                           │
                    │  维护: bid history,        │
                    │  pricing, SLA tracking    │
                    └─────────────────────────┘

  Mesh 职责:
    - 维护 Agent 的实时可用状态和技能信息
    - 提供"谁可以做什么"的发现服务
    - 监控 Agent 健康状态

  Market 职责:
    - 对具体任务发起竞价
    - 根据价格、质量、可用性选择最优 Agent
    - 跟踪任务执行和结算

  典型流程:
    1. 任务到达 Market 层
    2. Market 查询 Mesh 的 capability index: "谁可以处理这个任务?"
    3. Mesh 返回候选 Agent 列表及其当前状态
    4. Market 向候选 Agent 发起竞价
    5. Agent 返回报价（价格 + 交付时间）
    6. Market 选择最优方案并分配任务
    7. 任务完成后 Market 触发结算
    8. Agent 更新状态回 Mesh
```

### 9.3 Parliament-Manager 混合

**结构**：Parliament 负责集体决策和策略制定，Manager 负责策略的执行落地。

```
                   Parliament-Manager 混合架构

                    ┌─────────────────────────────┐
                    │      Parliament Layer         │
                    │                              │
                    │  提案 → 辩论 → 投票 → 裁决    │
                    │                              │
                    │  决策类型:                    │
                    │  - 任务优先级策略             │
                    │  - 质量标准定义               │
                    │  - 资源分配政策               │
                    │  - 异常处理方案               │
                    │                              │
                    │  Output: Policy Document      │
                    └───────────┬──────────────────┘
                                │ 政策输出
                                ▼
                    ┌─────────────────────────────┐
                    │      Adapter Layer            │
                    │  政策 → 执行参数              │
                    │  "提高质量标准" → threshold:0.95│
                    └───────────┬──────────────────┘
                                │ 参数配置
                                ▼
                    ┌─────────────────────────────┐
                    │      Manager Layer            │
                    │                              │
                    │  根据 Parliament 的策略配置    │
                    │  管理 Worker 的执行           │
                    │                              │
                    │  常规决策: Manager 自主        │
                    │  重大决策: 提交 Parliament     │
                    └─────────────────────────────┘

  Parliament 职责:
    - 制定和修订系统级别策略
    - 仲裁 Manager 无法解决的冲突
    - 批准重大变更

  Manager 职责:
    - 在 Parliament 制定的策略框架内自主执行
    - 将策略转化为 Worker Agent 的具体指令
    - 监控执行偏差并在需要时回馈到 Parliament
```

### 9.4 Pipeline-Mesh 混合

**结构**：Pipeline 作为主要的线性处理流程，Mesh 作为异常处理和旁路协作的网络。

```
                     Pipeline-Mesh 混合架构

          Pipeline 主线流程:

         输入 → Stage A → Stage B → Stage C → Stage D → 输出
                    │                        │
                    │  Stage B 遇到复杂问题    │ Stage D 需要额外
                    │  需要多 Agent 协作      │ 数据验证服务
                    ▼                        ▼
          ┌────────────────┐    ┌──────────────────────┐
          │  Mesh 旁路网络   │    │  Mesh 服务网络         │
          │                  │    │                       │
          │  B1 ←→ B2       │    │  D1(主) ←→ D2(验证)   │
          │   ↕     ↕       │    │     ↕                  │
          │  B3 ←→ B4       │    │  D3(备)               │
          │                  │    │                       │
          │  并行处理复杂问题  │    │  多 Agent 交叉验证     │
          │  结果合并回 StageB│    │  结果合并回 Stage D    │
          └────────────────┘    └──────────────────────┘
                    │                        │
                    └──────────┬─────────────┘
                               │
                               ▼
                         Pipeline 主线继续


  Pipeline 职责:
    - 处理常规、确定性的主流程
    - 将任务按阶段逐步转化

  Mesh 职责:
    - 处理 Pipeline 阶段中需要动态协作的复杂子问题
    - 旁路异常处理——当某个阶段出错时,Mesh 提供恢复路径
    - 质量检验——在关键阶段后通过 Mesh 进行交叉验证
```

## 10. 核心优势

### 10.1 问题-模式匹配度最大化

混合模式允许系统架构师为问题的不同部分选择最合适的模式，而不是强迫整个问题适应一个模式的框架。

```
问题: 构建一个软件系统开发的多 Agent 平台

问题维度            → 最匹配的模式         → 混合方案
─────────────────────────────────────────────────────
需求分析阶段         → Mesh (自由探索)      → Mesh
架构决策            → Parliament (集体决策)  → Parliament
任务分解            → Manager-Worker       → Manager   │
编码实现            → Pipeline (阶段化)     → Pipeline  ├── Mixed
代码审查            → Parliament (评审)     → Parliament│
资源调度            → Market (竞价)         → Market   │
Bug 修复(紧急)      → Manager (直接分配)    → Manager  │
```

### 10.2 正交化复杂度

不同维度的复杂度被分配到不同的模式层，避免了单一模式需要处理所有复杂度的问题。

```
单一模式需要处理:                              混合模式:
┌────────────────────────────┐               ┌──────────────────┐
│  策略 + 组织 + 执行 + 调度  │               │ Parliament: 策略  │
│  全部在同一个模式里实现      │               ├──────────────────┤
│                            │               │ Manager: 组织     │
│  "上帝类"反模式             │               ├──────────────────┤
│  改一个地方可能影响所有功能   │               │ Pipeline: 执行    │
│                            │               ├──────────────────┤
│                            │               │ Market: 调度      │
│                            │               └──────────────────┘
│                            │               每层只关心自己的复杂度
│                            │               修改策略不影响执行流程
└────────────────────────────┘
```

### 10.3 渐进式演进能力

混合模式可以从小规模开始，逐步增加模式层，而不需要一次性设计完整架构。

```
Phase 1: 最小可行系统
  Manager-Worker (2-3 Agent)
  └── 快速验证核心功能

Phase 2: 增加处理效率
  Manager-Worker + Pipeline
  └── 将串行任务组织为流水线

Phase 3: 增加决策质量
  Parliament + Manager-Worker + Pipeline
  └── 引入集体决策机制

Phase 4: 增加资源效率
  Parliament + Manager-Worker + Pipeline + Market
  └── Agent 资源市场化配置

Phase 5: 增加协作效率
  Full Hybrid (全部模式)
  └── 覆盖所有组织维度
```

### 10.4 故障隔离

因为模式之间通过定义良好的接口通信，一个模式层的故障可以被隔离，不会导致全系统崩溃。

```
              ✗ Pipeline Layer Crash

  Parliament Layer: 独立运行,不受影响 ✓
      依然可以做出策略决策

  Manager Layer: 感知 Pipeline 不可用 ✓
      可以暂停任务分配,等待恢复

  Market Layer: 独立运行,不受影响 ✓
      可以继续管理已分配的 Agent 资源

  → 恢复: Manager 重新启动 Pipeline 层
  → 恢复后: Parliament 的决策可以传递给新的 Pipeline 实例
```

## 11. 工程优化

### 11.1 模式边界契约（Pattern Boundary Contract）

为每个模式层定义明确的边界契约，契约包含该层的输入/输出规范、性能承诺和错误处理策略。

```python
from dataclasses import dataclass, field
from typing import Any, Callable, Optional
from enum import Enum
import time
import logging

logger = logging.getLogger(__name__)


class PatternType(Enum):
    MANAGER_WORKER = "manager_worker"
    PIPELINE = "pipeline"
    MARKET = "market"
    MESH = "mesh"
    PARLIAMENT = "parliament"


@dataclass
class PatternContract:
    """模式边界契约——定义模式层的输入/输出/承诺"""
    pattern_type: PatternType
    version: str
    max_latency_ms: int
    max_concurrent_calls: int
    error_handling: str  # "fail_fast" | "retry" | "circuit_breaker"
    input_schema: dict
    output_schema: dict

    def validate_input(self, data: dict) -> bool:
        """验证输入是否符合契约"""
        # 实际实现中使用 JSON Schema 验证
        required_fields = self.input_schema.get("required", [])
        for field in required_fields:
            if field not in data:
                logger.error(f"契约验证失败: 缺少必填字段 {field}")
                return False
        return True

    def validate_output(self, data: dict) -> bool:
        """验证输出是否符合契约"""
        required_fields = self.output_schema.get("required", [])
        for field in required_fields:
            if field not in data:
                logger.error(f"输出验证失败: 缺少必填字段 {field}")
                return False
        return True


# 契约定义示例
PIPELINE_CONTRACT = PatternContract(
    pattern_type=PatternType.PIPELINE,
    version="2.1.0",
    max_latency_ms=30000,
    max_concurrent_calls=10,
    error_handling="circuit_breaker",
    input_schema={
        "required": ["task_id", "stages", "input_data"],
        "type": "object"
    },
    output_schema={
        "required": ["task_id", "result", "stage_logs"],
        "type": "object"
    }
)

MANAGER_CONTRACT = PatternContract(
    pattern_type=PatternType.MANAGER_WORKER,
    version="1.2.0",
    max_latency_ms=5000,
    max_concurrent_calls=5,
    error_handling="retry",
    input_schema={
        "required": ["goal", "constraints"],
        "type": "object"
    },
    output_schema={
        "required": ["goal", "subtasks", "assignments"],
        "type": "object"
    }
)
```

### 11.2 跨模式追踪

在混合系统中，所有模式层共享一个追踪基础设施。每个跨层调用都携带追踪上下文。

```python
import uuid
from contextvars import ContextVar
from dataclasses import dataclass, field
from typing import Optional
import time

# 全局追踪上下文
trace_context: ContextVar[Optional["TraceSpan"]] = ContextVar("trace_context", default=None)


@dataclass
class TraceSpan:
    trace_id: str
    span_id: str
    parent_span_id: Optional[str]
    pattern: str  # 当前模式类型
    layer: str    # 层名
    operation: str
    start_time: float = field(default_factory=time.time)
    end_time: Optional[float] = None
    tags: dict = field(default_factory=dict)

    def finish(self):
        self.end_time = time.time()
        duration = (self.end_time - self.start_time) * 1000
        logger.info(
            f"[Trace {self.trace_id}] [{self.pattern}:{self.layer}] "
            f"{self.operation} completed in {duration:.1f}ms"
        )


class TraceContextManager:
    """跨模式追踪上下文管理器"""

    @staticmethod
    def create_trace() -> str:
        trace_id = uuid.uuid4().hex[:16]
        return trace_id

    @staticmethod
    def span(pattern: str, layer: str, operation: str):
        """创建一个追踪 span (用作 context manager)"""
        parent = trace_context.get()
        span = TraceSpan(
            trace_id=parent.trace_id if parent else TraceContextManager.create_trace(),
            span_id=uuid.uuid4().hex[:12],
            parent_span_id=parent.span_id if parent else None,
            pattern=pattern,
            layer=layer,
            operation=operation,
            tags={
                "pattern": pattern,
                "layer": layer,
                "parent_pattern": parent.pattern if parent else None
            }
        )
        return _TraceContext(span)


class _TraceContext:
    def __init__(self, span: TraceSpan):
        self.span = span

    def __enter__(self):
        trace_context.set(self.span)
        logger.info(
            f"[Trace {self.span.trace_id}] Enter [{self.span.pattern}:{self.span.layer}] "
            f"{self.span.operation}"
        )
        return self.span

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.span.finish()
        trace_context.set(None)


# 使用示例: 跨模式调用自动携带追踪上下文
def parliament_decide(topic: str) -> str:
    with TraceContextManager.span("Parliament", "strategy", f"decide:{topic}"):
        # 议会决策逻辑
        decision = f"policy_for_{topic}"
        return decision


def manager_execute(decision: str):
    with TraceContextManager.span("Manager", "organization", f"execute:{decision}"):
        subtasks = decompose(decision)
        for st in subtasks:
            pipeline_process(st)


def pipeline_process(subtask: dict):
    with TraceContextManager.span("Pipeline", "execution", f"process:{subtask['id']}"):
        # 流水线处理
        pass
```

### 11.3 模式间事件总线

使用统一的事件总线进行模式间通信，而不是直接调用。这降低了层间耦合。

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Callable, Dict, List
import asyncio


class PatternEvent(Enum):
    # Parliament 发出的事件
    POLICY_CHANGED = "policy.changed"
    STRATEGY_DECIDED = "strategy.decided"
    CONFLICT_RESOLVED = "conflict.resolved"

    # Manager 发出的事件
    TASK_DECOMPOSED = "task.decomposed"
    WORKER_ASSIGNED = "worker.assigned"
    PROGRESS_UPDATED = "progress.updated"

    # Pipeline 发出的事件
    STAGE_COMPLETED = "stage.completed"
    STAGE_FAILED = "stage.failed"
    PIPELINE_DONE = "pipeline.done"

    # Market 发出的事件
    AUCTION_STARTED = "auction.started"
    BID_RECEIVED = "bid.received"
    TASK_ALLOCATED = "task.allocated"


@dataclass
class EventMessage:
    event_type: PatternEvent
    source_pattern: str
    source_layer: str
    payload: dict
    trace_id: str
    timestamp: float = field(default_factory=time.time)


class PatternEventBus:
    """模式间事件总线——解耦模式层"""

    def __init__(self):
        self._handlers: Dict[PatternEvent, List[Callable]] = {event: [] for event in PatternEvent}

    def subscribe(self, event: PatternEvent, handler: Callable):
        self._handlers[event].append(handler)
        logger.info(f"订阅事件 {event.value}")

    async def publish(self, event: EventMessage):
        logger.info(
            f"[EventBus] {event.source_pattern} -> "
            f"{event.event_type.value}"
        )
        for handler in self._handlers[event.event_type]:
            try:
                await handler(event)
            except Exception as e:
                logger.error(f"事件处理失败 {event.event_type.value}: {e}")


# 使用示例: 解耦的跨模式通信
event_bus = PatternEventBus()

# Pipeline 层: 订阅 Manager 的任务分解事件
async def on_task_decomposed(event: EventMessage):
    subtasks = event.payload["subtasks"]
    logger.info(f"Pipeline 接收到 {len(subtasks)} 个子任务,开始流水线处理")
    for st in subtasks:
        await pipeline_process(st)

event_bus.subscribe(PatternEvent.TASK_DECOMPOSED, on_task_decomposed)

# Manager 层: 发出任务分解事件（不关心谁处理）
async def manager_workflow(goal: str):
    subtasks = [{"id": "s1", "action": "research"}, {"id": "s2", "action": "draft"}]
    await event_bus.publish(EventMessage(
        event_type=PatternEvent.TASK_DECOMPOSED,
        source_pattern="Manager",
        source_layer="organization",
        payload={"goal": goal, "subtasks": subtasks},
        trace_id=uuid.uuid4().hex[:16]
    ))
```

### 11.4 模式层健康检查

每层暴露标准化的健康检查和指标，用于全局监控。

```python
@dataclass
class PatternHealth:
    pattern: str
    layer: str
    status: str  # "healthy" | "degraded" | "down"
    active_tasks: int
    error_rate: float
    avg_latency_ms: float
    last_check: float = field(default_factory=time.time)


class PatternMonitor:
    """模式层健康监控"""

    def __init__(self):
        self.health_status: Dict[str, PatternHealth] = {}

    def register_pattern(self, pattern: str, layer: str):
        key = f"{pattern}:{layer}"
        self.health_status[key] = PatternHealth(
            pattern=pattern, layer=layer, status="healthy",
            active_tasks=0, error_rate=0.0, avg_latency_ms=0.0
        )

    def report_health(self, pattern: str, layer: str, health: PatternHealth):
        key = f"{pattern}:{layer}"
        self.health_status[key] = health

    def get_system_health(self) -> str:
        """全局健康状态"""
        statuses = [h.status for h in self.health_status.values()]
        if all(s == "healthy" for s in statuses):
            return "healthy"
        if any(s == "down" for s in statuses):
            return "degraded"
        return "healthy"  # 部分 degraded 仍算整体健康


# 全局监控实例
monitor = PatternMonitor()
monitor.register_pattern("Parliament", "strategy")
monitor.register_pattern("Manager", "organization")
monitor.register_pattern("Pipeline", "execution")
monitor.register_pattern("Market", "resource")
```

## 12. 代码示例

### 12.1 完整混合模式系统：AI 内容生产平台

以下示例展示一个 AI 内容生产平台，它组合了四种模式：
- **Parliament**: 编辑委员会决定内容策略和主题优先级
- **Manager-Worker**: 主编分解任务并分配给专业 Agent
- **Pipeline**: 内容经过研究→写作→审校→发布的流水线
- **Market**: 写作任务在 Agent 之间竞价分配，优化成本和质量

```python
"""
Hybrid Multi-Agent System: AI Content Production Platform

Pattern Architecture:
  Layer 1 (Strategy):  Parliament  — 编辑委员会决定内容策略
  Layer 2 (Org):       Manager     — 主编分解和分配任务
  Layer 3 (Execution): Pipeline    — 内容生产流水线
  Layer 4 (Resource):  Market      — 写作资源竞价分配

Cross-Pattern Communication: Event Bus + Trace Context
"""

import uuid
import time
import logging
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Callable
from enum import Enum
from abc import ABC, abstractmethod

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("HybridSystem")

# ════════════════════════════════════════════════════════════════
# 基础类型定义
# ════════════════════════════════════════════════════════════════

@dataclass
class ContentTask:
    id: str
    title: str
    content_type: str  # "blog", "tutorial", "docs", "news"
    target_audience: str
    requirements: Dict
    priority: int = 0
    status: str = "pending"
    created_at: float = field(default_factory=time.time)

@dataclass
class ContentPiece:
    task_id: str
    title: str
    body: str
    quality_score: float = 0.0
    word_count: int = 0
    author: str = ""
    review_notes: List[str] = field(default_factory=list)

@dataclass
class AgentProfile:
    id: str
    skills: List[str]       # ["research", "writing", "review"]
    specialties: List[str]  # ["AI", "web_dev", "security"]
    quality_history: List[float] = field(default_factory=list)
    current_load: int = 0
    base_price: float = 10.0

# ════════════════════════════════════════════════════════════════
# 模式接口契约
# ════════════════════════════════════════════════════════════════

class PatternContract:
    """模式接口契约——定义模式的输入输出规范"""

    def __init__(self, pattern_name: str, version: str):
        self.pattern_name = pattern_name
        self.version = version

    def validate(self, operation: str, data: dict) -> bool:
        """验证操作和数据是否符合契约"""
        logger.info(f"[Contract] {self.pattern_name} v{self.version} | "
                    f"validate {operation}")
        return True  # 简化实现

    def log_invocation(self, caller: str, operation: str, trace_id: str):
        logger.info(f"[Contract] {caller} -> {self.pattern_name}.{operation} "
                    f"[trace: {trace_id}]")


# 契约实例
PARLIAMENT_CONTRACT = PatternContract("Parliament", "1.0.0")
MANAGER_CONTRACT = PatternContract("Manager", "1.0.0")
PIPELINE_CONTRACT = PatternContract("Pipeline", "2.0.0")
MARKET_CONTRACT = PatternContract("Market", "1.5.0")


# ════════════════════════════════════════════════════════════════
# 模式层 1: Parliament — 编辑委员会
# ════════════════════════════════════════════════════════════════

class Parliament:
    """议会模式: 集体决策内容策略"""

    def __init__(self):
        self.members: List[str] = ["Editor_Chief", "SEO_Expert", "Domain_Lead"]
        self.policies: Dict = {}
        logger.info(f"[Parliament] 成立: 成员 {self.members}")

    @PatternContract.log_invocation
    def decide_strategy(self, market_trends: dict, trace_id: str = "") -> dict:
        PARLIAMENT_CONTRACT.validate("decide_strategy", market_trends)
        logger.info(f"[Parliament] 策略决策会议 [trace: {trace_id}]")

        # 模拟辩论和投票过程
        votes = {}
        for member in self.members:
            # 每个成员根据自己专长投票
            vote = {
                "member": member,
                "preferred_topic": market_trends.get("hot_topic", "AI"),
                "priority": 8 if member == "SEO_Expert" else 6,
                "reason": f"{member} 的领域分析"
            }
            votes[member] = vote
            logger.info(f"  {member} 投票: {vote['preferred_topic']} (优先级 {vote['priority']})")

        # 聚合决策
        decision = {
            "topic": "AI Safety in Enterprise",
            "content_type": "tutorial",
            "priority": 8,
            "deadline": "3d",
            "quality_threshold": 0.85,
            "votes": votes,
            "trace_id": trace_id
        }
        self.policies["current_strategy"] = decision
        logger.info(f"[Parliament] 决策: {decision['topic']} "
                    f"(优先级 {decision['priority']})")
        return decision

    @PatternContract.log_invocation
    def review_content(self, piece: ContentPiece, trace_id: str = "") -> ContentPiece:
        PARLIAMENT_CONTRACT.validate("review_content", {"piece_id": piece.task_id})
        logger.info(f"[Parliament] 评审内容: {piece.title} [trace: {trace_id}]")

        # 集体评审
        score = min(1.0, piece.quality_score * 1.1)  # 模拟提升
        piece.quality_score = score
        piece.review_notes.append(f"Parliament 评审通过, 评分 {score:.2f}")
        logger.info(f"  Review score: {score:.2f}")
        return piece


# ════════════════════════════════════════════════════════════════
# 模式层 2: Manager — 内容主编
# ════════════════════════════════════════════════════════════════

class Manager:
    """Manager-Worker 模式: 任务分解与分配"""

    def __init__(self):
        self.workers: Dict[str, AgentProfile] = {}
        self.active_tasks: Dict[str, ContentTask] = {}

    def register_worker(self, worker: AgentProfile):
        self.workers[worker.id] = worker
        logger.info(f"[Manager] 注册 Worker: {worker.id} "
                    f"(技能: {worker.skills})")

    def decompose_task(self, strategy: dict, trace_id: str = "") -> List[ContentTask]:
        """将策略决策分解为可执行的任务"""
        MANAGER_CONTRACT.validate("decompose", strategy)
        logger.info(f"[Manager] 分解策略: {strategy['topic']} "
                    f"[trace: {trace_id}]")

        tasks = [
            ContentTask(
                id=f"task_{uuid.uuid4().hex[:8]}",
                title=f"Research: {strategy['topic']}",
                content_type="research",
                target_audience="general",
                requirements={"depth": "comprehensive", "sources": 10},
                priority=strategy["priority"]
            ),
            ContentTask(
                id=f"task_{uuid.uuid4().hex[:8]}",
                title=f"Write: {strategy['topic']} Tutorial",
                content_type="tutorial",
                target_audience="developers",
                requirements={"format": "step-by-step", "examples": True},
                priority=strategy["priority"]
            ),
            ContentTask(
                id=f"task_{uuid.uuid4().hex[:8]}",
                title=f"Illustrate: {strategy['topic']} Diagrams",
                content_type="illustration",
                target_audience="developers",
                requirements={"style": "technical", "count": 3},
                priority=strategy["priority"] - 1
            ),
        ]

        for t in tasks:
            self.active_tasks[t.id] = t
            logger.info(f"  子任务: {t.title} [{t.id}]")

        return tasks


# ════════════════════════════════════════════════════════════════
# 模式层 3: Pipeline — 内容生产流水线
# ════════════════════════════════════════════════════════════════

class PipelineStage(ABC):
    """Pipeline 阶段抽象基类"""

    def __init__(self, name: str):
        self.name = name

    @abstractmethod
    def process(self, input_data: any, trace_id: str = "") -> any:
        pass


class ResearchStage(PipelineStage):
    """研究阶段——收集和分析资料"""

    def __init__(self):
        super().__init__("Research")

    def process(self, task: ContentTask, trace_id: str = "") -> dict:
        PIPELINE_CONTRACT.validate("process", {"task_id": task.id, "stage": self.name})
        logger.info(f"  [Pipeline:{self.name}] 研究: {task.title} [trace: {trace_id}]")
        return {
            "task_id": task.id,
            "findings": f"Research findings for {task.title}",
            "sources": ["source1", "source2"],
            "key_points": ["point1", "point2", "point3"],
            "stage": self.name
        }


class WritingStage(PipelineStage):
    """写作阶段——基于研究资料撰写内容"""

    def __init__(self):
        super().__init__("Writing")

    def process(self, research_output: dict, trace_id: str = "") -> ContentPiece:
        PIPELINE_CONTRACT.validate("process", {"research_id": research_output["task_id"]})
        logger.info(f"  [Pipeline:{self.name}] 写作: {research_output['task_id']} "
                    f"[trace: {trace_id}]")

        return ContentPiece(
            task_id=research_output["task_id"],
            title=f"Article on {research_output.get('key_points', ['topic'])[0]}",
            body=f"Full article content based on {len(research_output['sources'])} sources...",
            quality_score=0.75,
            word_count=1500,
            author="writer_agent"
        )


class ReviewStage(PipelineStage):
    """审校阶段——质量检查和改进"""

    def __init__(self):
        super().__init__("Review")

    def process(self, piece: ContentPiece, trace_id: str = "") -> ContentPiece:
        PIPELINE_CONTRACT.validate("process", {"piece_id": piece.task_id})
        logger.info(f"  [Pipeline:{self.name}] 审校: {piece.title} "
                    f"[trace: {trace_id}]")

        piece.quality_score = min(1.0, piece.quality_score + 0.15)
        piece.review_notes.append(f"[{self.name}] 审校通过")
        logger.info(f"  Quality after review: {piece.quality_score:.2f}")
        return piece


class ContentPipeline:
    """Pipeline 模式: 内容生产流水线"""

    def __init__(self):
        self.stages: List[PipelineStage] = [
            ResearchStage(),
            WritingStage(),
            ReviewStage()
        ]
        logger.info(f"[Pipeline] 初始化: {[s.name for s in self.stages]}")

    def execute(self, task: ContentTask, trace_id: str = "") -> ContentPiece:
        """执行完整的流水线处理"""
        logger.info(f"[Pipeline] 开始处理任务: {task.id} [trace: {trace_id}]")

        # 阶段 1: 研究
        research_out = self.stages[0].process(task, trace_id)

        # 阶段 2: 基于研究结果写作
        draft = self.stages[1].process(research_out, trace_id)

        # 阶段 3: 审校
        final = self.stages[2].process(draft, trace_id)

        logger.info(f"[Pipeline] 完成: {final.title} (质量: {final.quality_score:.2f})")
        return final


# ════════════════════════════════════════════════════════════════
# 模式层 4: Market — 写作资源竞价市场
# ════════════════════════════════════════════════════════════════

class MarketAuction:
    """Market 模式: Agent 资源竞价市场"""

    def __init__(self):
        self.bids: Dict[str, List] = {}
        self.winning_bids: Dict[str, str] = {}
        logger.info("[Market] 竞价市场初始化")

    def register_agent(self, agent: AgentProfile):
        MARKET_CONTRACT.validate("register_agent", {"agent_id": agent.id})
        logger.info(f"[Market] 注册 Agent: {agent.id} (底价: ${agent.base_price}/hr)")

    def auction_task(self, task: ContentTask, agents: List[AgentProfile],
                     trace_id: str = "") -> tuple:
        """为任务发起竞价,返回 (中标 Agent, 中标价格)"""
        MARKET_CONTRACT.validate("auction", {"task_id": task.id})
        logger.info(f"[Market] 竞价开始: {task.title} [trace: {trace_id}]")

        bids_received = []
        for agent in agents:
            # 模拟 Agent 报价: 价格基于其历史质量和当前负载
            quality_factor = (sum(agent.quality_history[-5:]) / 5
                              if agent.quality_history else 0.5)
            load_penalty = agent.current_load * 2
            bid_price = agent.base_price + load_penalty - (quality_factor * 5)
            bid_price = max(5.0, bid_price)  # 最低价格

            bid = {
                "agent_id": agent.id,
                "price": round(bid_price, 2),
                "quality": quality_factor,
                "estimated_hours": task.requirements.get("depth") == "comprehensive" and 4 or 2
            }
            bids_received.append(bid)
            logger.info(f"  Agent {agent.id}: 报价 ${bid['price']} | "
                        f"质量 {bid['quality']:.2f} | "
                        f"预计 {bid['estimated_hours']}h")

        # 选择最优方案: 综合价格和质量的评分
        for bid in bids_received:
            bid["score"] = (bid["quality"] * 10) - (bid["price"] / 10)

        winner = max(bids_received, key=lambda b: b["score"])
        self.winning_bids[task.id] = winner["agent_id"]
        logger.info(f"[Market] 中标: Agent {winner['agent_id']} "
                    f"(最终得分: {winner['score']:.2f})")

        # 返回中标 Agent 和价格
        winner_agent = next(a for a in agents if a.id == winner["agent_id"])
        return winner_agent, winner["price"]


# ════════════════════════════════════════════════════════════════
# 防腐层 (Anti-Corruption Layer): Manager ↔ Market 适配器
# ════════════════════════════════════════════════════════════════

class ManagerMarketAdapter:
    """防腐层: 将 Manager 的命令式任务分配转化为 Market 的竞价模型"""

    def __init__(self, manager: Manager, market: MarketAuction):
        self.manager = manager
        self.market = market

    def assign_task_through_market(self, task: ContentTask,
                                   trace_id: str = "") -> Optional[str]:
        """
        将 Manager 的任务通过 Market 竞价分配
        翻译职责:
          1. Manager 的"命令" → Market 的"标书"
          2. 补充竞价所需的元数据(预算、技能要求)
          3. Market 的"中标结果" → Manager 的"任务分配"
        """
        logger.info(f"[AntiCorruption] 翻译: Manager -> Market")
        logger.info(f"  Manager: '分配任务 {task.id} 给最合适的 Agent'")
        logger.info(f"  Market: '任务 {task.id} 开放竞价, 最低价中标'")

        # 获取所有可用的 Worker
        available_agents = list(self.manager.workers.values())

        # 通过 Market 竞价
        winner_agent, price = self.market.auction_task(task, available_agents, trace_id)

        # 更新 Manager 状态
        self.manager.active_tasks[task.id].status = "assigned"
        winner_agent.current_load += 1

        result = f"Task {task.id} -> Agent {winner_agent.id} @ ${price}"
        logger.info(f"[AntiCorruption] 翻译: Market -> Manager")
        logger.info(f"  Market: 'Agent {winner_agent.id} 中标, 价格 ${price}'")
        logger.info(f"  Manager: '好的, 任务分配给 {winner_agent.id}'")

        logger.info(f"[AntiCorruption] 分配结果: {result}")
        return winner_agent.id


# ════════════════════════════════════════════════════════════════
# 完整混合系统
# ════════════════════════════════════════════════════════════════

class HybridContentSystem:
    """
    混合模式内容生产系统

    架构:
    ┌─────────────────────────────────────┐
    │ Parliament   (策略决策/编辑委员会)    │
    ├─────────────────────────────────────┤
    │ Manager      (任务分解/主编)         │
    ├─────────────────────────────────────┤
    │ Pipeline     (阶段处理/内容流水线)   │
    ├─────────────────────────────────────┤
    │ Market       (资源调度/竞价市场)     │
    └─────────────────────────────────────┘
    """

    def __init__(self):
        # 初始化各模式层
        self.parliament = Parliament()
        self.manager = Manager()
        self.pipeline = ContentPipeline()
        self.market = MarketAuction()

        # 防腐层
        self.manager_market_adapter = ManagerMarketAdapter(self.manager, self.market)

        # 注册 Worker Agent
        self._setup_workers()
        # 注册 Agent 到市场
        for worker in self.manager.workers.values():
            self.market.register_agent(worker)

        logger.info("=" * 60)
        logger.info("混合模式内容生产系统启动")
        logger.info(f"模式组合: Parliament + Manager-Worker + Pipeline + Market")
        logger.info("=" * 60)

    def _setup_workers(self):
        workers = [
            AgentProfile("writer_ai", ["research", "writing"], ["AI", "ML"],
                         quality_history=[0.8, 0.85, 0.9, 0.82, 0.88]),
            AgentProfile("writer_web", ["research", "writing"], ["web_dev", "frontend"],
                         quality_history=[0.75, 0.8, 0.72, 0.85, 0.9]),
            AgentProfile("writer_sec", ["research", "writing"], ["security", "networking"],
                         quality_history=[0.9, 0.88, 0.92, 0.85, 0.95]),
            AgentProfile("illustrator", ["illustration", "design"], ["diagrams", "infographics"],
                         quality_history=[0.7, 0.75, 0.8, 0.78, 0.82]),
        ]
        for w in workers:
            self.manager.register_worker(w)

    def produce_content(self, request: dict) -> ContentPiece:
        """
        完整的内容生产流程——展示所有模式层的协同

        流程:
        1. Parliament: 决定内容策略 (战略层)
        2. Manager: 分解策略为子任务 (组织层)
        3. Manager-Market Adapter: 通过市场分配写作任务 (防腐层)
        4. Pipeline: 流水线执行 (执行层)
        5. Parliament: 最终质量评审 (战略层回环)
        """
        trace_id = uuid.uuid4().hex[:16]
        logger.info(f"\n{'='*60}")
        logger.info(f"新内容生产请求 [trace: {trace_id}]")
        logger.info(f"{'='*60}")

        # ── 步骤 1: Parliament 策略决策 ──
        logger.info("\n--- [Parliament Layer] 策略决策 ---")
        strategy = self.parliament.decide_strategy(
            {"hot_topic": request.get("topic", "AI")}, trace_id
        )

        # ── 步骤 2: Manager 任务分解 ──
        logger.info("\n--- [Manager Layer] 任务分解 ---")
        tasks = self.manager.decompose_task(strategy, trace_id)

        # ── 步骤 3: Manager ↔ Market 协作 (通过防腐层) ──
        logger.info("\n--- [Manager-Market via Anti-Corruption Layer] 资源分配 ---")
        for task in tasks[:2]:  # 前两个任务通过市场竞价
            self.manager_market_adapter.assign_task_through_market(task, trace_id)

        # ── 步骤 4: Pipeline 流水线处理 ──
        logger.info("\n--- [Pipeline Layer] 内容生产 ---")
        results = []
        for task in tasks:
            piece = self.pipeline.execute(task, trace_id)
            results.append(piece)

        # ── 步骤 5: Parliament 最终评审 (策略层回环) ──
        logger.info("\n--- [Parliament Layer] 最终评审 ---")
        final_piece = results[0] if results else None
        for piece in results:
            self.parliament.review_content(piece, trace_id)

        logger.info(f"\n{'='*60}")
        logger.info(f"生产完成: {final_piece.title if final_piece else 'N/A'}")
        logger.info(f"最终质量: {final_piece.quality_score:.2f}" if final_piece else "N/A")
        logger.info(f"使用了 {len(self.manager.workers)} 个 Agent, "
                    f"{len(self.pipeline.stages)} 个 Pipeline 阶段")
        logger.info(f"{'='*60}")

        return final_piece


# ════════════════════════════════════════════════════════════════
# 系统运行
# ════════════════════════════════════════════════════════════════

if __name__ == "__main__":
    system = HybridContentSystem()

    result = system.produce_content({
        "topic": "AI Safety in Enterprise",
        "format": "tutorial",
        "urgency": "high"
    })

    print(f"\n最终交付: {result.title}")
    print(f"质量评分: {result.quality_score:.2f}")
    print(f"审校记录: {result.review_notes}")
```

### 12.2 输出示例

```
============================================================
混合模式内容生产系统启动
模式组合: Parliament + Manager-Worker + Pipeline + Market
============================================================

============================================================
新内容生产请求 [trace: a1b2c3d4e5f6g7h8]
============================================================

--- [Parliament Layer] 策略决策 ---
[Parliament] 成员投票: Editor_Chief → "AI Safety" (优先级 8)
[Parliament] 成员投票: SEO_Expert → "AI Safety" (优先级 9)
[Parliament] 成员投票: Domain_Lead → "AI Safety" (优先级 7)
[Parliament] 决策: "AI Safety in Enterprise" (优先级 8)

--- [Manager Layer] 任务分解 ---
[Manager] 子任务 1: Research: AI Safety in Enterprise
[Manager] 子任务 2: Write: AI Safety in Enterprise Tutorial
[Manager] 子任务 3: Illustrate: AI Safety in Enterprise Diagrams

--- [Manager-Market via Anti-Corruption Layer] 资源分配 ---
[AntiCorruption] 翻译: Manager's "assign" → Market's "auction"
[Market] Agent writer_ai:  报价 $8.50 | 质量 0.85 | 预计 4h
[Market] Agent writer_web: 报价 $9.20 | 质量 0.80 | 预计 2h
[Market] Agent writer_sec: 报价 $7.80 | 质量 0.90 | 预计 4h
[Market] 中标: Agent writer_sec (得分: 8.22)
[AntiCorruption] 分配结果: task_abc123 -> Agent writer_sec @ $7.80

--- [Pipeline Layer] 内容生产 ---
[Pipeline] 阶段 Research → 发现 10 个关键来源
[Pipeline] 阶段 Writing → 完成 1500 字初稿 (质量 0.75)
[Pipeline] 阶段 Review → 质量提升至 0.90

--- [Parliament Layer] 最终评审 ---
[Parliament] 评审通过, 评分: 0.92

============================================================
生产完成: Article on AI Safety in Enterprise
最终质量: 0.92
使用了 4 个 Agent, 3 个 Pipeline 阶段
============================================================
```

### 12.3 关键设计思考

以上代码示例展示了混合模式的核心设计原则：

1. **每层独立演化**：Parliament 层可以改变投票规则，不影响 Pipeline 层的阶段处理逻辑。
2. **防腐层翻译语义**：Manager 和 Market 有不相容的假设（命令 vs 竞价），防腐层在中间做语义转换，让两者无需直接理解对方。
3. **追踪跨模式调用**：`trace_id` 贯穿所有模式层——从 Parliament 的决策到 Pipeline 的最终输出，任何步骤的问题都可以追溯到源头。
4. **契约保障接口稳定**：每个模式层都通过 `PatternContract` 声明自己的输入/输出协议，跨层调用必须经过契约验证。
5. **模式组合不是叠加而是分层**：每层解决一个不同维度的问题（策略、组织、执行、资源），而不是把多个模式的代码混在一起。

## 总结

混合模式（Hybrid Patterns）是多 Agent 系统从"玩具"走向"生产系统"的必经之路。单一 Swarm Pattern 适合教学和简单场景，但现实世界的多 Agent 系统几乎总是需要组合多种模式。

**核心原则**：每层解决一个维度的组织问题，通过定义良好的接口契约通信。

**常见陷阱**：过度工程——为"可能需要的"未来场景添加模式层，而混合模式的威力恰恰在于其"按需组合"的能力。对于大多数应用，从 2-3 个模式的简单组合开始，随需求演进逐步增加模式层是最佳策略。

**选择建议**：
- 2-3 个 Agent + 简单任务 → 单一模式（Manager-Worker 或 Pipeline）
- 5-10 个 Agent + 中等复杂任务 → 2 模式组合（Manager-Pipeline 最常见）
- 10-50 个 Agent + 复杂任务 → 3-4 模式组合（增加 Market 或 Parliament）
- 50+ 个 Agent + 高度复杂/动态任务 → Full Hybrid（全部五种模式）
