# 8.5.1 Manager-Worker -- 管理者-工作者模式

## 1. 简单介绍

**Manager-Worker 模式**（管理者-工作者模式，亦称 Master-Slave、Master-Worker、Controller-Worker）是分布式系统与多智能体系统中最基础、应用最广泛的协调模式。其核心思想是：**一个中心节点（Manager/管理者）负责任务分解与分配，多个同质或异质的工作节点（Worker/工作者）并行执行子任务，管理者最终汇总并整合各工作者的执行结果。**

这种模式将"决策"与"执行"两个关注点明确分离，管理者关注全局策略与任务编排，工作者聚焦于具体子任务的高效执行。它天然适合那些可以拆分为独立子任务的工作负载，是"分而治之"（Divide and Conquer）原则在 Agent 系统中的直接体现。

```
                            +-------------------+
                            |                   |
                            |    Manager        |
                            |  (Task Decomposer |
                            |   & Aggregator)   |
                            |                   |
                            +--------+----------+
                                     |
                    decompose & assign (task queue)
                                     |
              +----------+-----------+-----------+----------+
              |          |           |           |          |
              v          v           v           v          v
        +---------+ +---------+ +---------+ +---------+ +---------+
        | Worker1 | | Worker2 | | Worker3 | | Worker4 | | Worker5 |
        | (Agent) | | (Agent) | | (Agent) | | (Agent) | | (Agent) |
        +---------+ +---------+ +---------+ +---------+ +---------+
              |          |           |           |          |
              +----------+-----------+-----------+----------+
                                     |
                           return results (aggregate)
                                     |
                            +--------v----------+
                            |                   |
                            |   Manager         |
                            |  (Merge & Format  |
                            |   Final Output)   |
                            |                   |
                            +-------------------+
```

在 AI Agent 语境下，Manager 是一个"元认知"智能体（meta-agent），它不直接操作领域逻辑，而是通过规划（planning）和任务分解（task decomposition）调动一组子智能体协同工作。Worker 可以是专门的领域 Agent（如代码生成 Agent、数据分析 Agent、搜索 Agent），也可以是通用 LLM 实例。

---

## 2. 基本原理

### 2.1 核心工作流

Manager-Worker 的执行流程可用四个阶段描述：

```
  PHASE 1: TASK DECOMPOSITION
  =============================
  用户输入 ("写一篇关于量子计算的市场分析报告")
        |
        v
  Manager 分析任务:
    ├── 子任务1: 调研量子计算核心技术进展 (搜索 + 摘要)
    ├── 子任务2: 收集主要厂商动态 (Google, IBM, Microsoft)
    ├── 子任务3: 分析市场规模与预测数据
    ├── 子任务4: 对比经典计算与量子计算优劣势
    └── 子任务5: 撰写最终报告 (汇总所有子任务结果)
        |
        v
  生成任务描述包 (Task Spec), 每个包包含:
    { task_id, description, context, expected_output_format, deadline }


  PHASE 2: TASK ASSIGNMENT & DISPATCH
  =====================================
  Manager 将子任务放入共享任务队列 (Task Queue)
        |
        v
  +---------------------------------------------------+
  |                   Task Queue                       |
  |  [Task1] [Task2] [Task3] [Task4] [Task5] ...      |
  +---------------------------------------------------+
        |
        v
  调度策略:
    ├── Round-Robin: 轮流分配给下一个可用 Worker
    ├── Least-Loaded: 分配给当前负载最低的 Worker
    ├── Specialized: 根据 Worker 专长分配 (Router模式)
    └── Broadcast: 所有 Worker 执行相同任务 (投票/冗余)


  PHASE 3: PARALLEL EXECUTION
  ============================
  Worker1 (搜索 Agent)     Worker2 (分析 Agent)      Worker3 (写作 Agent)
  ┌─────────────────┐     ┌─────────────────┐      ┌─────────────────┐
  │ receive Task1    │     │ receive Task2    │      │ receive Task5    │
  │ execute search   │     │ analyze data     │      │ draft content    │
  │ format results   │     │ generate insights│      │ apply style guide│
  │ return to Queue  │     │ return to Queue  │      │ return to Queue  │
  └─────────────────┘     └─────────────────┘      └─────────────────┘
        |                       |                         |
        v                       v                         v
  +---------------------------------------------------------------+
  |                     Result Queue                              |
  |  [Result1] [Result2] [Result5] ... (out-of-order possible)    |
  +---------------------------------------------------------------+


  PHASE 4: RESULT AGGREGATION
  =============================
  Manager 从 Result Queue 拉取结果
        |
        v
  聚合策略:
    ├── Ordered Merge:  按原始任务顺序合并结果
    ├── Vote / Majority: 冗余执行时投票决定最终输出
    ├── Best-of-N:      保留质量评分最高的结果
    └── Progressive:    结果边到边聚合（流式输出）
        |
        v
  Manager 输出最终结果给用户
```

### 2.2 任务描述协议

Manager 与 Worker 之间的通信遵循结构化协议。每个任务包至少包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `task_id` | UUID | 全局唯一任务标识 |
| `parent_id` | UUID? | 父任务 ID（支持嵌套分解） |
| `description` | String | 自然语言任务描述 |
| `context` | Dict | 共享上下文（参考文档、前置结果等） |
| `input_spec` | Dict | 结构化输入参数 |
| `output_schema` | JSON Schema | 期望的输出结构 |
| `dependencies` | List[UUID] | 前置依赖任务 ID 列表 |
| `priority` | Int | 优先级（数值越低越优先） |
| `timeout` | Int | 超时时间（秒） |
| `retry_policy` | Dict | 重试策略（max_retries, backoff） |

结果包字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `task_id` | UUID | 对应任务 ID |
| `worker_id` | String | 执行 Worker 标识 |
| `status` | Enum | `success` / `failure` / `partial` |
| `data` | Any | 执行结果数据 |
| `metadata` | Dict | 执行元数据（耗时、token 消耗、置信度） |
| `error` | Dict? | 错误信息（如有） |

### 2.3 同步与异步模式

Manager-Worker 有两种典型执行模式：

- **同步模式（Synchronous）**: Manager 分发任务后阻塞等待所有 Worker 返回，适用于子任务间有依赖关系的场景。Manager 维护一个依赖图（DAG），按拓扑序调度。
- **异步模式（Asynchronous）**: Manager 分发任务后立即返回，Worker 通过回调（callback）、消息队列或事件总线异步返回结果。适用于高延迟、长周期任务。

---

## 3. 背景与发展

### 3.1 分布式计算中的起源

Manager-Worker 模式的思想可以追溯到 20 世纪 60 年代的 **主从架构（Master-Slave Architecture）**：

- **1970s -- 数据库主从复制**: 一个 Master 节点处理写请求，多个 Slave 节点处理只读查询，实现读写分离和负载均衡。
- **1980s -- 并行计算**: 在 MPI（Message Passing Interface）和 PVM（Parallel Virtual Machine）中，"Master Process" 负责任务切分，"Worker Processes" 执行计算片断，这是 HPC（High Performance Computing）的核心模式。
- **1990s -- 分布式操作系统**: Amoeba、Plan 9 等分布式 OS 采用"处理器池"（processor pool）模型，中心管理器将进程调度到空闲 CPU 上执行。
- **2004 -- Google MapReduce**: Jeff Dean 和 Sanjay Ghemawat 提出的 MapReduce 编程模型将 Manager-Worker 推向工程实践的巅峰。MapReduce 的 `Master` 节点负责任务调度、故障恢复和数据分片，`Mapper` 和 `Reducer` 是两类 Worker 节点。

```
  MapReduce 中的 Manager-Worker 结构:

  +---------+      +===============+      +---------+
  |  Input  |----->|    Master     |<----->|  Output |
  |  Data   |      | (Manager)     |       |  Data   |
  +---------+      +=======+=======+       +---------+
                           |
             +-------------+-------------+
             |             |             |
         +---v---+     +---v---+     +---v---+
         |Map #1 |     |Map #2 |     |Map #N |   (Map Phase)
         +---+---+     +---+---+     +---+---+
             |             |             |
             +------+------+------+------+
                    |             |
                +---v---+     +---v---+
                |Reduce |     |Reduce |       (Reduce Phase)
                |  #1   |     |  #2   |
                +-------+     +-------+
```

- **2006 -- Apache Hadoop**: 开源实现了 MapReduce，将 Manager-Worker 模式带入工业界大规模应用。
- **2010s -- 微服务与容器编排**: Kubernetes 的 Controller Manager + Worker Node 架构、Celery 的任务队列架构、AWS Lambda 的函数即服务（FaaS），本质都是 Manager-Worker 的变体。

### 3.2 进入 AI Agent 时代

随着大语言模型（LLM）的兴起，Manager-Worker 模式在 Agent 系统中经历了新的演变：

| 时间 | 系统/框架 | Manager-Worker 实现 |
|---|---|---|
| 2023 | **AutoGPT** | 一个 Agent 循环中持续执行"思考-行动-观察"，本质上是一个单 Agent 模拟的 Manager（规划）+ Worker（执行） |
| 2023 | **BabyAGI** | 明确的 Task Manager + Task Queue + Execution Worker 架构，LLM 驱动任务分解和结果评估 |
| 2024 | **ChatDev** | 多个专用 Agent（CEO、CTO、Programmer、Reviewer）构成组织化 Manager-Worker 链，模拟软件公司组织结构 |
| 2024 | **MetaGPT** | 角色化 Agent 协作框架，Product Manager / Architect / Engineer 形成领域化的 Manager-Worker 层次 |
| 2024 | **Microsoft AutoGen** | 支持 `GroupChat` 和 `Sequential` 等多种对话模式，其中 `GroupChatManager` 协调多个 Assistant Agent |
| 2024 | **LangGraph** | 基于状态机的 Agent 编排框架，支持定义条件分支和循环，`Supervisor` Node 可以调度多个 Sub-agent |
| 2025 | **OpenAI Agents SDK** | 内置 `Handoff` 机制，一个主 Agent（Manager）可以将任务转交给专用 Agent（Worker）处理 |
| 2025 | **Anthropic Claude Agent SDK** | 支持 Agent 树状分解和多层协作，较新的模式探索 |

关键转折发生在 2024 年：研究者发现，**让 LLM 同时承担"全局规划"和"细节执行"会导致上下文溢出和注意力碎片化**。Manager-Worker 模式将这两个角色分离，让每个 Agent 在更窄的上下文窗口中专注工作，显著提高了复杂任务的完成质量。

---

## 4. 之前做法与局限

### 4.1 单 Agent 单体架构

在引入 Manager-Worker 模式之前，构建 Agent 应用的主流做法是**单 Agent 单体架构（Monolithic Agent）**：

```
  用户输入
      |
      v
  +---------------------------------------------+
  |           Monolithic Agent                   |
  |                                              |
  |  [System Prompt: "你是全能的AI助手..."]       |
  |                                              |
  |   Step 1: 分析需求 (在上下文中)              |
  |   Step 2: 检索信息 (调用工具)                |
  |   Step 3: 处理数据 (LLM 推理)                |
  |   Step 4: 生成输出 (LLM 生成)                |
  |   Step 5: 自我检查 (LLM 评估)                |
  |   ... (所有步骤在同一上下文窗口)              |
  |                                              |
  +---------------------------------------------+
      |
      v
  最终输出
```

### 4.2 这种架构的主要局限

**1. 上下文溢出（Context Overflow）**

单 Agent 需要在同一个上下文窗口中完成所有工作：系统提示、用户输入、检索到的文档、中间推理步骤、工具调用记录、错误信息、自我反思结果。对于需要多轮工具调用和大量外部数据的长任务，上下文窗口很快被填满，早期信息被"遗忘"。

典型表现：Agent 在任务中段突然忘记用户最初的需求，或重复已完成的工作。

**2. 注意力碎片化（Attention Fragmentation）**

LLM 在同一上下文中处理多种不同类型的任务（规划、搜索、分析、写作、验证）时，注意力资源被分散。研究表明，模型在"多任务回绕"（multi-task wrap）情况下更易产生事实性错误和逻辑跳跃。

**3. 单点故障（Single Point of Failure）**

整个任务的成败完全取决于一个 Agent 实例。如果该实例产生幻觉、陷入循环、或触发内容安全策略拒绝回答，整个任务无条件失败。没有容错机制。

**4. 并行度为零（No Parallelism）**

所有步骤顺序执行，即使步骤之间没有依赖关系。例如，"搜索公司 A 的背景"和"搜索公司 B 的背景"这两个完全独立的子任务必须串行完成。

```
  单 Agent 执行路径（串行）:
  时间 -->
  [搜索 A] -> [分析 A] -> [搜索 B] -> [分析 B] -> [搜索 C] -> [写报告]

  Manager-Worker 执行路径（并行）:
  时间 -->
  [分解任务]
  [分配]
    ├── [Worker1: 搜索 A]   ──┐
    ├── [Worker2: 搜索 B]   ──┤  (同时执行)
    └── [Worker3: 搜索 C]   ──┘
  [聚合结果]
  [Worker4: 写报告]
```

**5. 工具冲突（Tool Conflict）**

单 Agent 持有过多工具时，面临"工具选择困难"——模型需要在几十个工具描述中准确选出合适的工具，错误率随工具数量呈超线性增长。Manager-Worker 模式中，每个 Worker 只持有少量领域专精工具。

**6. 没有专业化能力（No Specialization）**

同一个 Agent 既要当"搜索专家"又要当"数据分析师"又要当"作家"。系统提示词中塞入多个角色描述会导致角色冲突（"你现在是一个严格的数据分析师...但同时你也要写出引人入胜的文章"）。

---

## 5. 核心矛盾

Manager-Worker 模式看似简单直接，但在实际系统中面临几个深层次的矛盾：

### 5.1 集中协调 vs Worker 自主性

```
  矛盾三角形:

              任务一致性 (Coherence)
                     /\
                    /  \
                   /    \
                  /      \
                 /        \
                /          \
               /            \
  集中控制 (Centralized) ---- Worker 自主 (Worker Autonomy)
```

- **过度集中**: Manager 做所有决策（"Worker3 用谷歌搜索，Worker5 用必应搜索"），Worker 变成纯粹的函数调用器。失去了 Agent 应有的智能判断能力，Manager 成为瓶颈。
- **过度自主**: Manager 只给一个模糊目标（"调研这个市场"），Worker 自行决定搜索策略、分析方法和输出格式。结果可能质量不统一、格式混乱，甚至偏离原始目标。

**权衡点**: Manager 应定义"做什么"和"输出的形式标准"，Worker 应自主决定"怎么做"。

### 5.2 Manager 瓶颈

当 Worker 数量增加时，Manager 面临处理瓶颈：

```
  Worker 数量 vs Manager 负载:

  负载
   ^
   |                     Manager 成为瓶颈区
   |                        /
   |                      /
   |                    /
   |                 /   ← Manager 单点处理能力上限
   |              /
   |           /
   |        /
   |     /   ← 低负载区，Manager 空闲等待 Worker
   |  /
   +-------------------------------------------------> Worker 数量
      1   2   3   4   5   6   7   8   9   10  11  12
```

Manager 的瓶颈操作包括：
- 任务分解（需要 LLM 推理，是计算密集型操作）
- 结果验证与聚合（需要读取所有 Worker 的输出，IO 密集型）
- Worker 状态跟踪（需要维护和更新每个 Worker 的状态）

### 5.3 Worker 空闲 vs 负载均衡

```
  不均衡分配:
  +--------------------------------------------------+
  | Worker1: ██████████████████████████  (繁忙)       |
  | Worker2: ████                         (空闲)       |
  | Worker3: ████████████████            (中等)       |
  | Worker4: ██                           (几乎空闲)   |
  +--------------------------------------------------+
  总等待时间 = max(Worker1) = 长

  均衡分配:
  +--------------------------------------------------+
  | Worker1: ████████████                   (均衡)     |
  | Worker2: ████████████                   (均衡)     |
  | Worker3: ████████████                   (均衡)     |
  | Worker4: ████████████                   (均衡)     |
  +--------------------------------------------------+
  总等待时间 = 明显缩短
```

关键问题：
- 如何提前知道每个子任务的耗时？
- 任务拆分粒度应多细？（粒度太粗 -> 负载不均；粒度太细 -> 调度开销过大）
- Worker 性能异构时（不同规格的 LLM 实例），如何加权分配？

### 5.4 通信开销 vs 并行收益

增加 Worker 并非总是提高效率。通信开销会随 Worker 数量增加而增长：

```
  吞吐量
   ^
   |        ___
   |      /    \
   |    /       \_____ 边际收益递减区域
   |  /               \______ 负收益区域（通信开销主导）
   |/
   +-------------------------------------------------> Worker 数量
      最优并行度 N*
```

通信开销包括：
- Manager 向 N 个 Worker 分发任务的序列化/反序列化成本
- Manager 从 N 个 Worker 接收结果的等待时间（尤其当 Worker 执行时间差异大时）
- 中间结果传输的 Token 成本

---

## 6. 主流优化方向

### 6.1 动态 Worker 池（Dynamic Worker Pool）

根据实时负载自动伸缩 Worker 数量，而不是固定 Worker 规模。

```
              +-------------------+
              |  Worker Pool      |
              |  Manager          |
              |                   |
              |  +------+         |
              |  |Scheduler|      |
              |  +---+---+       |
              |      |           |
              |      v           |
              |  +-----------+   |
              |  | Pool      |   |
              |  | Manager   |   |
              |  +-----------+   |
              +-------------------+
                    |     |
         spawn / destroy     health check
                    |     |
         +----------+-----+----------+
         |          |                |
    +----v---+ +---v----+     +-----v----+
    |Worker1 | |Worker2 | ... |Worker N  |
    +--------+ +--------+     +----------+

  扩缩容策略:
  ├── Reactive: 队列长度 > 阈值 -> 加 Worker; Worker 空闲 > 阈值 -> 减 Worker
  ├── Predictive: 基于历史负载时序预测未来需求，提前扩缩
  └── Hybrid: 基础负载 + 弹性缓冲池
```

### 6.2 自适应任务分配（Adaptive Task Distribution）

不依赖固定的 Round-Robin 或随机分配，而是基于 Worker 的历史表现动态调度：

| 策略 | 原理 | 适用场景 |
|---|---|---|
| **Least-Loaded** | 分配给当前任务队列最短的 Worker | Worker 同质，任务执行时间差异大 |
| **Shortest-Expected** | 根据历史数据预估各 Worker 执行时间，选择最早空闲的 | Worker 性能异构 |
| **Power-of-Two-Choices** | 随机选 2 个 Worker，分配给其中负载较轻的 | 大规模 Worker 池，理论最优比 |
| **Expertise-Based** | 跟踪每个 Worker 在不同任务类型上的成功率，优先分配给专长匹配的 | Worker 功能异质 |
| **Cost-Aware** | 考虑 Token 消耗/API 调用成本，选择性价比最高的 Worker | 成本敏感场景 |

### 6.3 层次化管理（Hierarchical Management）

当 Worker 数量超过百级时，单层 Manager-Worker 结构不再适用。引入中间层 Manager：

```
                                +-------------------+
                                |  Root Manager     |
                                |  (全局规划者)      |
                                +--------+----------+
                                         |
                     +-------------------+-------------------+
                     |                   |                   |
               +-----v------+     +-----v------+     +-----v------+
               | Sub-Mgr 1  |     | Sub-Mgr 2  |     | Sub-Mgr 3  |
               | (领域1)    |     | (领域2)    |     | (领域3)    |
               +--+--+--+---+     +--+--+--+---+     +--+--+--+---+
                  |  |  |           |  |  |           |  |  |
          +-------+  |  +----+  +---+  |  +----+  +---+  |  +----+
          v          v       v  v      v       v  v      v       v
        W1  W2  W3  W4  W5  W6  W7  W8  W9  W10 W11 W12 W13 W14

  Root Manager:
    - 负责顶层任务分解（"写一份多领域市场报告" -> "技术部分"+"市场部分"+"竞争分析"）
    - 不直接与底层 Worker 通信

  Sub-Manager:
    - 负责领域内任务进一步分解和分配
    - 管理本领域 Worker 的状态和结果
    - 向 Root Manager 返回领域级聚合结果

  Worker:
    - 执行原子级子任务
    - 只与所属 Sub-Manager 通信
```

### 6.4 Worker 专业化（Worker Specialization）

避免所有 Worker"大而全"，让每个 Worker 专注于特定能力：

```
  +-------------------------------------------------------------+
  |                     Manager (Router)                         |
  +--------+--------+--------+--------+--------+--------+--------+
           |        |        |        |        |        |
           v        v        v        v        v        v
  +--------++ +-----+---+ +--+------+ +------+-+ +------+-+ +------+
  | Search  | | Code   | | Data    | | Write  | | Review | | QA   |
  | Worker  | | Worker | | Analyst | | Worker | | Worker | |Worker|
  +---------+ +--------+ +---------+ +--------+ +--------+ +------+
  持工具有:    持工具有:   持工具有:    持工具有:   持工具有:   持工具有:
  - 网页搜索   - 代码解释器  - Python   - 文档模板  - 代码审查   - 测试框架
  - 向量检索   - git       - 可视化库  - 风格检查  - 静态分析  - 断言库
  - 知识库API  - IDE 工具   - SQL      - 多语言    - 安全扫描  - 覆盖率
```

好处：
- Worker 系统提示大幅简化，只有专长领域的指令
- 工具数量少，工具选择准确率高
- 输出质量更稳定、可预测

### 6.5 容错委托（Fault-Tolerant Delegation）

处理 Worker 失效的各种情况：

| 失效模式 | 检测方法 | 恢复策略 |
|---|---|---|
| Worker 超时无响应 | 心跳检测 + 超时计时器 | 将任务重新分配给另一个 Worker |
| Worker 输出格式错误 | Schema 校验 | 请求重试，附带格式修正指令 |
| Worker 产生幻觉/低质量 | 自我一致性检查 + 质量评分 | 触发冗余执行（2-3个 Worker 投票） |
| Worker 永远陷入循环 | 最大步骤限制 + LLM 调用次数上限 | 强制终止，部分结果回滚 |
| Manager 自身失效 | 主备切换（Leader Election） | 备用 Manager 接管状态 |

```
  失效恢复流程图:

  分发任务 T 给 Worker_A
        |
        v
  +-----------------------+
  | 启动任务 T 的超时计时器 |<--- 超时时间 = 预估时间 * (1 + 容忍系数)
  +-----------------------+
        |         |
        |         |
  正常完成         超时
        |         |
        v         v
  验证结果    标记 Worker_A 为 suspect
        |         |
   格式正确?     重新分配 T 给 Worker_B
    /    \           |
   是     否         v
    |      |     Worker_B 执行
    v      v         |
  聚合   通知 Worker_A    失败?
         "请修正格式"      |     |
          |              是    否
          v               |     v
     重试 N 次?          重试  聚合 (比较 A vs B)
        /   \               |
       是    否             去重后使用较优结果
       |     |
       v     v
     重新分配 标记失败
     给 Worker_C 记录日志
```

---

## 7. 实现挑战

### 7.1 任务粒度调优（Task Granularity Tuning）

```
  粗粒度                 细粒度
  <-----------+-----------+----------->
  高吞吐        最优粒度      高开销
  低灵活性      (Goldilocks)  低利用率

  粒度风险:
  ┌─────────────────────────────────────────────┐
  │  粒度过粗:                                   │
  │  - 任务不可再分，无法发挥并行优势              │
  │  - 单个 Worker 执行时间过长，成为长尾          │
  │  - 某个 Worker 失败则丢失大量中间结果          │
  ├─────────────────────────────────────────────┤
  │  粒度过细:                                   │
  │  - Manager 调度开销超过 Worker 执行时间        │
  │  - 大量小任务导致 Manager 成为瓶颈             │
  │  - 跨任务上下文传递成本高昂                    │
  │  - 跟踪数百个子任务的状态复杂度过高             │
  └─────────────────────────────────────────────┘
```

在实践中，任务粒度调优依赖经验法则：

1. **子任务执行时间应远大于调度开销**: 如果调度一个任务需要 ~500ms（包括 Manager LLM 推理时间），那么 Worker 执行时间至少应在 5-10 秒以上。
2. **子任务之间的上下文共享量应最小**: 如果两个子任务共享大量上下文数据，它们本应是一个子任务。
3. **子任务产出应该是自包含的**: 每个子任务的输出应当独立可验证，不需要依赖其他子任务的中间结果来理解。

### 7.2 Worker 失效检测

在基于 LLM 的 Agent 系统中，Worker "失效"不限于进程崩溃，还包括更多软性失效模式：

- **语义失效**: Worker 的输出在格式上正确，但在语义上完全偏离任务（例如，要求分析北美市场，Worker 给出了欧洲市场的数据）。
- **虚假完成**: Worker 声称任务完成但实际上没有执行任何有意义的操作（"对不起，我无法完成这个任务"）。
- **逻辑失效**: Worker 的推理过程中出现明显错误，导致最终结果不可靠。

检测策略：
- **验证 Worker（Validator Worker）**: 专门设置一个 Worker 对其他 Worker 的输出进行合理性校验。
- **一致性交叉检查**: 同一任务分配给多个 Worker，比较输出的一致性。
- **结构约束强制**: 通过 JSON Schema、正则表达式等硬约束确保输出格式正确。
- **概率置信度评分**: LLM 自己标注对输出的置信度，低于阈值的标记为可疑。

### 7.3 结果一致性

当多个 Worker 并行执行不同子任务时，如何确保最终结果在逻辑上一致、不自相矛盾？

```
  问题场景:
  ==========
  市场报告任务:
    Worker_A (搜索专家): "2025年市场规模为 $150B"
    Worker_B (趋势分析): "市场规模年增长率 12%"
    Worker_C (竞争分析): "市场份额: 公司X 占 40%, 公司Y 占 35%"
  
  矛盾:
    - 如果市场规模 $150B，公司X 占 40% = $60B
    - 但 Worker_B 的数据暗示 2024 年市场为 $150B/1.12 = $133.9B
    - 这些数据可能来自不同来源、不同统计口径
```

解决方案：

1. **共享知识库（Shared Context）**: Manager 在任务分配时提供统一的参考数据集（如"所有市场数据请引用 GrandViewResearch 2025 年 1 月发布的报告"）。
2. **问题后聚合校验（Post-Aggregation Validation）**: Manager 聚合时自动检查跨 Worker 结果的一致性，标记矛盾点要求相关 Worker 修正。
3. **先聚合后推理（Aggregate-Then-Reason）**: Manager 先收集原始数据，再统一调用一个"整合 Worker"进行最终的推理和撰写，确保逻辑一致性。
4. **版本化数据引用**: 每个结果附带数据来源和时效性标签，Manager 在聚合时对不同时间粒度的数据进行加权。

### 7.4 大规模通信开销

当 Worker 数量从几个扩展到数百个时，通信模式成为瓶颈：

| Worker 数 | 通信模式 | 瓶颈 | 建议架构 |
|---|---|---|---|
| 1-10 | 直接 RPC / 内存队列 | Manager CPU | 单层 Manager-Worker |
| 10-100 | 消息队列（RabbitMQ / Redis） | 网络 IO | 引入 Sub-Manager 层 |
| 100-1,000+ | 事件驱动（Kafka / NATS） | 序列化 | 多层层次化管理 |
| 10,000+ | 去中心化 Gossip 协议 | 一致性 | Peer-to-Peer Mesh |

---

## 8. 能力与边界

### 8.1 适用场景（When to Use）

Manager-Worker 模式在以下场景中表现优异：

| 场景 | 原因 | 示例 |
|---|---|---|
| **易并行任务**（Embarrassingly Parallel） | 子任务间无依赖或依赖很少，天然适合并行 | 批量文件处理、多语言翻译、多源数据采集 |
| **Fan-Out 查询** | 同一问题需要从多个角度/来源获取答案 | 市场调研、竞品分析、多数据库查询 |
| **分层决策** | 先做高层决策，再细化为具体执行步骤 | 软件架构设计、项目管理、战略规划 |
| **内容生成流水线** | 生成 -> 检查 -> 修改 的多阶段工作流 | 代码生成（写 -> 审查 -> 测试 -> 部署） |
| **Quality-of-Service 分级** | 不同用户/任务需要不同服务质量 | 免费用户分配 1 个 Worker，付费用户分配 5 个 |
| **多视角评估** | 同一输出需要从不同维度评估 | 论文评审、代码审查、风险评估 |

典型成功案例：

```
  案例1: 多语言内容本地化
  =======================
  Manager 接收一篇英文文章
    ├── Worker_zh: 翻译为中文
    ├── Worker_ja: 翻译为日文
    ├── Worker_ko: 翻译为韩文
    ├── Worker_fr: 翻译为法文
    └── Worker_de: 翻译为德文
  Manager 聚合所有译文并输出

  并行度: 5x
  结果: 如果每个翻译需要 30 秒，总耗时 30 秒而非 150 秒


  案例2: 多维度代码审查
  =====================
  Manager 接收一份 PR diff
    ├── Worker_security: 安全漏洞扫描
    ├── Worker_perf: 性能影响分析
    ├── Worker_style: 代码风格检查
    ├── Worker_test: 测试覆盖率评估
    └── Worker_docs: 文档完整性检查
  Manager 汇总所有审查意见并生成整合报告

  并行度: 5x
  结果: 每个审查需要 20 秒，总耗时 20 秒而非 100 秒
```

### 8.2 不适用场景（When NOT to Use）

| 场景 | 原因 | 替代模式 |
|---|---|---|
| **紧密耦合的迭代推理**（Iterative Reasoning） | 每一步都依赖上一步的结果，无法并行 | 单 Agent 链式推理或 Pipeline 模式 |
| **极小任务**（Trivial Tasks） | Manager 的分解 + 聚合成本超过直接执行 | 单 Agent 直接处理 |
| **高实时性要求**（Real-Time） | 并行通信和调度引入额外延迟 | 预计算 + 缓存，或 Pipeline 模式 |
| **任务状态依赖性强**（State-Dependent） | 子任务共享大量可变全局状态 | Parliament 模式（共享黑板） |
| **Worker 数量极少**（1-2 Workers） | Manager 的开销无法被并行收益抵消 | 单 Agent + 工具调用 |
| **任务不可分解**（Non-Decomposable） | 任务是原子性的，无法拆分为独立子任务 | 直接执行 |

具体反例：

```
  反例1: 交互式对话
  ==================
  用户: "帮我逐步推导微分方程的解"
  
  为什么 Manager-Worker 不合适:
  - 每一步推导依赖于前一步的结果
  - Workers 之间需要频繁交换中间状态
  - Manager 串行化这些交互反而比直接解更慢
  
  更好的做法: 单 Agent 链式推理 + 工具使用


  反例2: 简单问答
  ==================
  用户: "巴黎是哪个国家的首都？"
  
  为什么 Manager-Worker 不合适:
  - 任务分解成本（1次 LLM 调用）> 直接回答成本（1次 LLM 调用）
  - 引入不必要的延迟和系统复杂度
  
  更好的做法: 单 Agent 直接回答
```

### 8.3 规模边界

```
  规模 vs 效果曲线:

  效果
   ^
   |  最佳实践区
   |  +----------------+
   |  |                |
   |  |  Workers: 3-20 |
   |  | 层次: 1-2 层   |
   |  | 任务粒度: 中   |
   |  +----------------+
   |        /                \
   |      /                    \
   |    /                        \
   |  /                            \
   |/                                \
   +-------------------------------------> 规模
  低复杂度                       超大规模
  (1-2 Workers)                  (100+ Workers)
  不值得分解                    通信/协调成本主导
```

经验法则：
- **3-7 个 Worker**: 理想的并行度，管理和通信开销可控
- **8-20 个 Worker**: 需要 Sub-Manager 或分组策略
- **20+ 个 Worker**: 必须引入层次化结构和异步通信
- **100+ 个 Worker**: 建议考虑 Mesh 模式或市场模式替代

---

## 9. 与其他模式对比

### 9.1 对比全景

```
                               协调方式
                         集中式             分散式
                       +----------------+----------------+
                       |                |                |
                静态   | Manager-Worker |      Mesh      |
                结构   |   (本节)       |   (8.5.5)      |
                       |                |                |
                       +----------------+----------------+
                       |                |                |
                动态   |   Parliament   |    Market      |
                结构   |   (8.5.2)      |   (8.5.3)      |
                       |                |                |
                       +----------------+----------------+

                       +----------------+----------------+
                       |                |                |
                线性   |   Pipeline     |                |
                流程   |   (8.5.4)      |                |
                       |                |                |
                       +----------------+----------------+
```

### 9.2 详细对比

| 维度 | Manager-Worker | Pipeline | Parliament | Mesh |
|---|---|---|---|---|
| **协调方式** | 集中式（Manager） | 链式传递 | 共享黑板（Blackboard） | 点对点（Peer-to-Peer） |
| **通信拓扑** | Star | Chain | Fully Connected via Board | Graph (任意) |
| **并行度** | 高（Fan-Out） | 低（串行阶段） | 高（多 Agent 共享黑板） | 中（依赖协议） |
| **耦合度** | 松散（Worker 间无交互） | 紧（阶段间强依赖） | 松散（通过黑板解耦） | 紧（需协商协议） |
| **容错性** | 中（Manager 是 SPOF） | 低（任一点断裂整个链断） | 高（黑板持久化） | 高（去中心化） |
| **扩展性** | 中（Manager 瓶颈） | 低（线性延伸） | 中（黑板竞争） | 高（理论无限） |
| **实现复杂度** | 低 | 中 | 高 | 最高 |
| **调试难度** | 低（中心化日志） | 中（阶段边界明确） | 中（通过黑板追踪） | 高（分布式追踪） |
| **典型规模** | 3-20 Workers | 3-10 阶段 | 5-15 Agents | 10-1000+ Agents |
| **适用任务** | 独立子任务 | 顺序依赖任务 | 协作推理任务 | 大规模自治系统 |

### 9.3 何时选择 Manager-Worker

**选择 Manager-Worker 当：**
- 子任务之间高度独立，没有或很少需要互相通信
- 需要一个中心点来控制全局策略和质量标准
- 系统的可观测性和调试便利性是优先考虑
- 团队对实现简单性有要求（相比 Mesh 或 Parliament）

**改选 Pipeline 当：**
- 任务的各个阶段有明确的先后顺序
- 每个阶段只依赖前一阶段的输出
- 阶段的输入/输出有清晰的数据契约

**改选 Parliament 当：**
- 子任务需要共享和竞争访问同一个知识库
- Agent 之间需要"群聊式"交流和协商
- 任务需要多轮讨论才能收敛

**改选 Mesh 当：**
- 需要大规模部署（100+ Agents）
- 系统要求完全去中心化，没有单点故障
- Agent 之间的交互模式高度动态无法预定义

---

## 10. 核心优势

### 10.1 实现简单（Simplicity）

在所有多 Agent 协调模式中，Manager-Worker 的架构最直观、最容易实现。

```
  实现差异:
  ==========
  Manager-Worker:   1 个队列 + N 个 Worker 进程
  Parliament:       1 个黑板 + N 个 Agent + 协调协议 + 仲裁机制
  Mesh:             每个 Agent 需要发现、路由、协商、心跳协议

  代码量级估算 (Python 伪代码):
  Manager-Worker:   ~200 行核心逻辑
  Pipeline:         ~300 行核心逻辑
  Parliament:       ~500-800 行核心逻辑
  Mesh:             ~1000+ 行核心逻辑
```

### 10.2 职责清晰（Clear Responsibility）

```
  Manager 关注:           Worker 关注:
  ┌──────────────┐       ┌──────────────┐
  │ 全局任务规划   │       │ 子任务高效执行 │
  │ 任务分解策略   │       │ 领域知识应用   │
  │ 资源调度分配   │       │ 专长工具调用   │
  │ 质量监控与评估 │       │ 结果格式化输出 │
  │ 异常处理与恢复 │       │ 错误上报与反馈 │
  │ 结果整合与输出 │       │              │
  └──────────────┘       └──────────────┘
  系统提示词: ~1000 tokens  系统提示词: ~300 tokens
  (规划+管理)               (领域专长+执行)
```

这种关注点分离（Separation of Concerns）带来：
- **系统提示词更精简**: 每个 Agent 只需要关注自己角色相关的指令
- **更容易调试**: 问题定位到具体 Agent（"结果是 Workers 生成的格式不对"vs"Manager 的聚合逻辑有 bug"）
- **独立演进**: Manager 的策略逻辑和 Worker 的执行逻辑可以独立修改和升级

### 10.3 天然并行（Natural Parallelism）

Manager-Worker 的 Fan-Out 拓扑使并行成为默认行为而非事后优化：

- 任务分解完成后，所有独立子任务可同时执行
- 系统吞吐量近线性扩展（受限于 Worker 数量和任务粒度）
- 可以混合使用不同执行后端（本地进程 + 远程 API + 云函数）

### 10.4 易于监控（Easy to Monitor）

集中式的架构使得观测和监控天然容易：

```
  可观测性矩阵:
  ==============
  指标              Manager 层面                  Worker 层面
  ─────────────────────────────────────────────────────────────
  吞吐量        (完成子任务数)/时间               (单 Worker 完成数)/时间
  延迟          端到端任务耗时                   单任务执行耗时
  错误率        (失败子任务数)/总子任务数          (失败次数)/总执行次数
  资源利用率    Worker 池使用率                  Token 消耗 / API 调用数
  队列深度      等待分配的子任务数量              Worker 本地队列长度
  健康状态      Manager 存活                     Worker 心跳 + 最后活动时间

  日志集中: 所有 Agent 的日志可汇聚到 Manager，形成端到端的事件序列。
```

---

## 11. 工程优化

### 11.1 Worker 结果缓存（Worker Result Cache）

对于重复性高的子任务（如同源搜索、相同数据计算），缓存可以大幅减少 LLM 调用和 Token 消耗。

```
  缓存策略:
  ==========
  用户请求 1: "分析公司 A"  -> 分解 -> [搜索 A, 分析 A, 生成报告]
  用户请求 2: "对比 A 和 B" -> 分解 -> [搜索 A (CACHE HIT!), 搜索 B, ...]

  +-------------------+
  |    Cache Layer    |
  |                   |
  |  key: task_hash   |
  |  value: result    |
  |  +-------------+  |
  |  | search:A    |  |
  |  |   -> {...}  |  |
  |  | search:B    |  |
  |  |   -> {...}  |  |
  |  | analyze:C   |  |
  |  |   -> {...}  |  |
  |  +-------------+  |
  +-------------------+

  缓存命中策略:
  ├── Exact Match: 任务描述完全一致
  ├── Semantic Match: 任务描述语义相似 (embeddings + 相似度阈值)
  └── TTL-Based: 设置过期时间，保证数据新鲜度
```

注意事项：
- 缓存键需包含上下文依赖（同样的"搜索 A"任务如果上下文不同，缓存可能不适用）
- LLM 调用缓存需要谨慎处理"时间敏感"的任务（如实时新闻搜索）
- 缓存建议使用 LRU（Least Recently Used）或 LFU（Least Frequently Used）淘汰策略

### 11.2 结果去重（Result Deduplication）

当多个 Worker 执行相似任务（如冗余执行、不同搜索源）时，结果中可能存在重复内容。

```
  去重流程:
  ==========
  Worker_A: 返回结果 [R1, R2, R3]
  Worker_B: 返回结果 [R2, R4, R5]
       |
       v
  +-------------------+
  |   Dedup Engine    |
  |                   |
  |  1. Exact Dedup:  |
  |     hash(R2)==hash (R2) -> 保留一个            |
  |                   |
  |  2. Near-Dedup:   |
  |     MinHash/LSH 相似度 > threshold -> 合并      |
  |                   |
  |  3. Semantic Dedup:                             |
  |     Embedding 余弦相似度 > threshold -> 提取精华 |
  +-------------------+
       |
       v
  去重后: [R1, R2, R3, R4, R5]
```

### 11.3 自适应超时（Adaptive Timeouts）

不依赖固定的硬编码超时，而是基于 Worker 的历史执行数据动态调整。

```
  动态超时计算:
  =============
  对于任务类型 T:
    历史执行时间: [t1, t2, t3, ..., tn]
    均值 μ = avg(ti)
    标准差 σ = std(ti)

    超时时间 = μ + k * σ

    其中 k 为容忍系数:
    ├── 关键路径任务: k = 2.0 (更高容忍)
    ├── 常规任务:    k = 3.0
    └── 非关键任务:  k = 4.0 (容忍长尾)

  实现:
    timeout_tracker = {
        "search":  {"mean": 12.3, "std": 2.1, "sample_count": 150},
        "analyze": {"mean": 25.7, "std": 5.3, "sample_count": 80},
        "write":   {"mean": 18.1, "std": 3.8, "sample_count": 120},
    }

    def get_timeout(task_type: str, criticality: str) -> float:
        stats = timeout_tracker[task_type]
        k = {"critical": 2.0, "normal": 3.0, "background": 4.0}[criticality]
        return stats["mean"] + k * stats["std"]
```

### 11.4 健康检查（Health Check）

维持 Worker 池的健康状态，及时剔除异常 Worker。

```
  健康检查协议:
  =============

  主动检查 (Pull):
  ┌──────────┐         ┌──────────┐
  │ Manager  │ ───ping─→│ Worker   │
  │          │ ←──pong──│          │
  │          │          │          │
  │  每隔 T 秒检查一次   │          │
  │  连续失败 N 次则下线  │          │
  └──────────┘          └──────────┘

  被动检查 (Push):
  ┌──────────┐          ┌──────────┐
  │ Manager  │          │ Worker   │
  │          │ ←──heartbeat────────│
  │          │   {"worker_id":"W1",│
  │          │    "status":"busy", │
  │          │    "task_id":"T3",  │
  │          │    "progress":0.6,  │
  │          │    "timestamp":...} │
  │          │          │          │
  │  超过 2*interval 未收到标记为可疑 │
  └──────────┘          └──────────┘

  健康状态机:
  ┌─────────────────────────────────────────────────┐
  │                                                   │
  │   HEALTHY ──(N次失败)──→ SUSPECT ──(M次失败)──→ DEAD  │
  │      ↑                    │                      │
  │      └──────(恢复)────────┘                      │
  │                                                   │
  │  DEAD: 任务重新分配，Worker 从池中移除              │
  │  SUSPECT: 不再接收新任务，等待恢复或确认死亡          │
  └─────────────────────────────────────────────────┘
```

### 11.5 背压（Backpressure）

当系统负载超过处理能力时，通过背压机制保护系统不被压垮。

```
  背压策略层次:
  =============

  层 1: Task Queue 限长
  ┌──────────────────────────────────┐
  │ Task Queue                       │
  │ [█ █ █ █ █ █ █ █] (已满)        │
  │                                  │
  │ 队列满时:                         │
  │ - Manager 停止接受新任务           │
  │ - 返回 503 Service Unavailable   │
  └──────────────────────────────────┘

  层 2: Worker 级背压
  ┌─────────────────────────────────────┐
  │ Worker 内部:                        │
  │ 本地任务队列 [█ █ █ █] (阈值=5)     │
  │                                     │
  │ 队列达到阈值:                        │
  │ - 不再接受 Manager 的新任务           │
  │ - 发送 backpressure 信号给 Manager   │
  │ - Manager 将任务分配给其他 Worker     │
  └─────────────────────────────────────┘

  层 3: Token 预算
  ┌─────────────────────────────────────┐
  │ Token 使用监控:                     │
  │ - 每分钟 / 每小时 Token 预算         │
  │ - 预算用尽时暂停 Worker 执行         │
  │ - 预算恢复后自动继续                 │
  └─────────────────────────────────────┘

  层 4: 降级服务
  ┌─────────────────────────────────────┐
  │ 系统过载时:                         │
  │ - 减少冗余执行 (2x -> 1x)           │
  │ - 简化输出格式 (详细 -> 摘要)        │
  │ - 缩短上下文 (全量搜索 -> 前 3 条)   │
  │ - 跳过高成本任务 (语义分析 -> 直接返回)│
  └─────────────────────────────────────┘
```

### 11.6 负载均衡策略图示

```
  +-------------------------------------------------------------+
  |                   负载均衡策略对比                            |
  +-------------------------------------------------------------+

  1. Round-Robin (轮询)
     Worker1: [T1] [T4] [T7] ...
     Worker2: [T2] [T5] [T8] ...
     Worker3: [T3] [T6] [T9] ...
     优点: 实现简单，保证"公平"
     缺点: 不考虑 Worker 实际负载和性能差异

  2. Weighted Round-Robin (加权轮询)
     Worker1 (权重 2): [T1] [T2] [T5] [T6] ...
     Worker2 (权重 1): [T3] [T7] ...
     Worker3 (权重 1): [T4] [T8] ...
     优点: 适合异构 Worker (不同规格的 LLM)
     缺点: 权重需要人工配置或自适应学习

  3. Least-Connections (最少连接)
     Worker1: [T1]          (1项)
     Worker2: [T2] [T3]     (2项)  <- 不选
     Worker3: (空闲)        (0项)  <- 选这个
     优点: 自动负载均衡
     缺点: 需要 Worker 实时上报负载

  4. Shortest-Expected-Time (最短预估时间)
     预估: Worker1 可用 @ t=5s, Worker2 @ t=3s, Worker3 @ t=0s
     新任务分配给 Worker3
     优点: 最小化总体完成时间
     缺点: 需要准确的执行时间预估模型

  5. Power of Two Choices (两随机一择优)
     随机选 Worker1 和 Worker4:
       Worker1 有 3 个排队任务
       Worker4 有 1 个排队任务
     选择 Worker4
     优点: 大规模下理论最优
     缺点: 小规模下效果不明显

  6. Specialized Routing (专长路由)
     搜索任务 -> Worker(search_agent)
     编写任务 -> Worker(writer_agent)
     分析任务 -> Worker(analyst_agent)
     优点: 充分利用专长
     缺点: 需要任务分类和 Worker 注册机制

  +-------------------------------------------------------------+
```

---

## 12. 代码示例

以下是一个完整的 Manager-Worker 模式 Python 伪代码实现，包含任务队列、Worker 池、结果聚合和基本错误处理。

### 12.1 基础类型定义

```python
"""
manager_worker.py -- Manager-Worker Pattern Implementation
"""

from __future__ import annotations

import asyncio
import hashlib
import json
import logging
import time
import uuid
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Callable, Optional

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


# ─── 枚举定义 ──────────────────────────────────────────────────

class TaskStatus(Enum):
    PENDING = "pending"          # 等待分配
    ASSIGNED = "assigned"        # 已分配给 Worker
    RUNNING = "running"          # Worker 正在执行
    SUCCESS = "success"          # 执行成功
    FAILED = "failed"            # 执行失败
    TIMEOUT = "timeout"          # 执行超时
    CANCELLED = "cancelled"      # 已取消


class WorkerStatus(Enum):
    IDLE = "idle"                # 空闲，可接受任务
    BUSY = "busy"                # 正在执行任务
    SUSPECT = "suspect"          # 可疑（可能异常）
    OFFLINE = "offline"          # 离线


# ─── 数据类 ──────────────────────────────────────────────────

@dataclass
class TaskSpec:
    """任务描述"""
    task_id: str = field(default_factory=lambda: uuid.uuid4().hex)
    description: str = ""                # 自然语言任务描述
    context: dict = field(default_factory=dict)  # 共享上下文
    input_data: dict = field(default_factory=dict)  # 结构化的输入数据
    output_schema: Optional[dict] = None # 期望的输出格式 (JSON Schema)
    dependencies: list = field(default_factory=list)  # 依赖的任务 ID 列表
    priority: int = 0                    # 优先级 (数值越低越优先)
    timeout: float = 60.0                # 超时时间 (秒)
    max_retries: int = 2                 # 最大重试次数
    task_type: str = "general"           # 任务类型 (用于路由)


@dataclass
class TaskResult:
    """任务执行结果"""
    task_id: str
    worker_id: str
    status: TaskStatus
    data: Any = None
    error: Optional[str] = None
    metadata: dict = field(default_factory=dict)
    # metadata should include: start_time, end_time, token_count, confidence_score


@dataclass
class WorkerInfo:
    """Worker 状态信息"""
    worker_id: str
    status: WorkerStatus = WorkerStatus.IDLE
    capabilities: list = field(default_factory=list)      # 能力列表
    current_task: Optional[str] = None                    # 当前执行的任务 ID
    task_history: list = field(default_factory=list)      # 最近 N 个任务记录
    performance: dict = field(default_factory=lambda: {
        "avg_exec_time": 0.0,
        "success_rate": 1.0,
        "total_tasks": 0,
        "failed_tasks": 0,
    })
    last_heartbeat: float = field(default_factory=time.time)
```

### 12.2 Manager 核心实现

```python
# ─── Manager 核心 ──────────────────────────────────────────────

class TaskDecomposer(ABC):
    """任务分解器接口 -- 输入复杂任务，输出子任务列表"""
    @abstractmethod
    async def decompose(self, task_description: str, context: dict) -> list[TaskSpec]:
        ...


class ResultAggregator(ABC):
    """结果聚合器接口 -- 输入子任务结果列表，输出最终结果"""
    @abstractmethod
    async def aggregate(self, results: list[TaskResult], original_context: dict) -> Any:
        ...


class Manager:
    """
    Manager-Worker 模式中的 Manager。

    负责任务分解、分配、监控和结果聚合。
    采用异步事件循环驱动，支持并行 Worker 执行。
    """

    def __init__(
        self,
        decomposer: TaskDecomposer,
        aggregator: ResultAggregator,
        worker_pool_size: int = 5,
        task_queue_maxsize: int = 100,
        heartbeat_interval: float = 5.0,
    ):
        self.decomposer = decomposer
        self.aggregator = aggregator
        self.worker_pool_size = worker_pool_size

        # 队列
        self.task_queue: asyncio.Queue = asyncio.Queue(maxsize=task_queue_maxsize)
        self.result_queue: asyncio.Queue = asyncio.Queue()

        # Worker 管理
        self.workers: dict[str, WorkerInfo] = {}
        self.worker_pool: dict[str, Worker] = {}

        # 任务追踪
        self.pending_tasks: dict[str, TaskSpec] = {}
        self.running_tasks: dict[str, tuple[TaskSpec, str]] = {}  # task_id -> (spec, worker_id)
        self.completed_tasks: dict[str, TaskResult] = {}

        # 调度
        self.heartbeat_interval = heartbeat_interval
        self.scheduling_strategy: str = "least_loaded"  # round_robin | least_loaded | specialized
        self._round_robin_index = 0

        # 生命周期
        self._running = False

    async def start(self):
        """启动 Manager，初始化 Worker 池并开始事件循环"""
        self._running = True

        # 初始化 Worker 池
        logger.info(f"Initializing worker pool with {self.worker_pool_size} workers...")
        for i in range(self.worker_pool_size):
            worker_id = f"worker-{i:03d}"
            worker = Worker(worker_id=worker_id)
            self.worker_pool[worker_id] = worker
            self.workers[worker_id] = WorkerInfo(
                worker_id=worker_id,
                capabilities=worker.capabilities,
            )

        # 启动 Worker 任务
        for worker_id, worker in self.worker_pool.items():
            asyncio.create_task(self._run_worker_loop(worker_id, worker))

        # 启动心跳检测
        asyncio.create_task(self._heartbeat_loop())

        # 启动结果处理
        asyncio.create_task(self._result_processing_loop())

        logger.info(f"Manager started with {self.worker_pool_size} workers")
        return self

    async def stop(self):
        """优雅关闭 Manager 和所有 Worker"""
        logger.info("Manager shutting down...")
        self._running = False

        # 取消所有待处理任务
        for task_id in self.pending_tasks:
            self.pending_tasks[task_id]  # 标记为重试
        self.pending_tasks.clear()

        # 通知 Worker 停止
        stop_tasks = []
        for worker_id, worker in self.worker_pool.items():
            stop_tasks.append(worker.stop())
        await asyncio.gather(*stop_tasks, return_exceptions=True)

        logger.info("Manager stopped")

    async def execute(self, user_request: str, context: dict | None = None) -> Any:
        """
        执行用户请求的完整生命周期：
        1. 任务分解
        2. 入队调度
        3. 等待 Worker 执行
        4. 结果聚合
        5. 返回最终结果
        """
        context = context or {}
        start_time = time.time()

        logger.info(f"Processing request: {user_request[:80]}...")

        # Step 1: 任务分解
        subtasks = await self.decomposer.decompose(user_request, context)
        logger.info(f"Decomposed into {len(subtasks)} subtasks")

        # 步骤: 记录所有子任务
        total_subtasks = len(subtasks)
        for task in subtasks:
            self.pending_tasks[task.task_id] = task

        # Step 2: 按依赖关系分批入队
        # (简化示例：无依赖的全部入队，有依赖的等依赖完成后再入队)
        for task in subtasks:
            if not task.dependencies:
                await self.task_queue.put(task)
            # 有依赖的任务由 _check_dependencies 处理

        # Step 3: 等待所有子任务完成
        # 使用计数器追踪进度
        completed_count = 0
        results = []

        while completed_count < total_subtasks:
            result = await self.result_queue.get()
            results.append(result)
            completed_count += 1

            # 检查是否有依赖就绪的任务可以入队
            await self._enqueue_ready_tasks(subtasks)

            # 进度日志
            if completed_count % max(1, total_subtasks // 5) == 0:
                elapsed = time.time() - start_time
                logger.info(
                    f"Progress: {completed_count}/{total_subtasks} "
                    f"({elapsed:.1f}s elapsed)"
                )

        # Step 4: 结果聚合
        final_result = await self.aggregator.aggregate(results, context)

        elapsed = time.time() - start_time
        logger.info(f"Request completed in {elapsed:.2f}s")

        return final_result

    async def _enqueue_ready_tasks(self, all_tasks: list[TaskSpec]):
        """检查并入队依赖已满足的任务"""
        completed_ids = set(self.completed_tasks.keys())

        for task in all_tasks:
            if task.task_id in self.pending_tasks and task.task_id not in self.completed_tasks:
                # 检查所有依赖是否已完成
                if all(dep_id in completed_ids for dep_id in task.dependencies):
                    try:
                        await self.task_queue.put(task)
                        logger.debug(f"Enqueued task {task.task_id} (dependencies met)")
                    except asyncio.QueueFull:
                        logger.warning(f"Task queue full, cannot enqueue {task.task_id}")
                        # 这里需要更复杂的等待逻辑，简化处理

    async def _run_worker_loop(self, worker_id: str, worker: "Worker"):
        """Worker 主循环：从队列取任务并执行"""
        while self._running:
            task_spec = await self.task_queue.get()

            # 更新 Worker 状态
            worker_info = self.workers[worker_id]
            worker_info.status = WorkerStatus.BUSY
            worker_info.current_task = task_spec.task_id

            # 标记任务为运行中
            self.running_tasks[task_spec.task_id] = (task_spec, worker_id)
            if task_spec.task_id in self.pending_tasks:
                del self.pending_tasks[task_spec.task_id]

            try:
                # 给 Worker 分配任务（带超时）
                result = await asyncio.wait_for(
                    worker.execute(task_spec),
                    timeout=task_spec.timeout,
                )

                # 验证结果格式
                if task_spec.output_schema:
                    is_valid = self._validate_output(result.data, task_spec.output_schema)
                    if not is_valid:
                        result.status = TaskStatus.FAILED
                        result.error = "Output schema validation failed"

                # 记录到 Worker 历史
                worker_info.task_history.append({
                    "task_id": task_spec.task_id,
                    "status": result.status.value,
                    "exec_time": result.metadata.get("end_time", time.time())
                                 - result.metadata.get("start_time", time.time()),
                })
                # 保持历史记录上限
                if len(worker_info.task_history) > 100:
                    worker_info.task_history.pop(0)

                # 更新性能指标
                perf = worker_info.performance
                perf["total_tasks"] += 1
                if result.status == TaskStatus.FAILED:
                    perf["failed_tasks"] += 1
                perf["success_rate"] = (
                    1.0 - perf["failed_tasks"] / max(1, perf["total_tasks"])
                )

            except asyncio.TimeoutError:
                # 任务超时处理
                result = TaskResult(
                    task_id=task_spec.task_id,
                    worker_id=worker_id,
                    status=TaskStatus.TIMEOUT,
                    error=f"Task timed out after {task_spec.timeout}s",
                    metadata={"start_time": time.time(), "end_time": time.time()},
                )
                logger.warning(f"Task {task_spec.task_id} timed out on {worker_id}")
            except Exception as e:
                result = TaskResult(
                    task_id=task_spec.task_id,
                    worker_id=worker_id,
                    status=TaskStatus.FAILED,
                    error=str(e),
                    metadata={"start_time": time.time(), "end_time": time.time()},
                )
                logger.error(f"Task {task_spec.task_id} failed on {worker_id}: {e}")

            # 标记任务完成
            self.completed_tasks[task_spec.task_id] = result
            if task_spec.task_id in self.running_tasks:
                del self.running_tasks[task_spec.task_id]

            # Worker 恢复为空闲
            worker_info.status = WorkerStatus.IDLE
            worker_info.current_task = None

            # 放入结果队列，通知主循环
            await self.result_queue.put(result)

            # 标记任务队列项处理完成
            self.task_queue.task_done()

    async def _heartbeat_loop(self):
        """定期检查 Worker 健康状态"""
        while self._running:
            await asyncio.sleep(self.heartbeat_interval)
            now = time.time()
            for worker_id, info in self.workers.items():
                # 如果 Worker 超过 3 个心跳间隔没有心跳，标记为可疑
                if now - info.last_heartbeat > self.heartbeat_interval * 3:
                    if info.status not in (WorkerStatus.OFFLINE, WorkerStatus.SUSPECT):
                        logger.warning(f"Worker {worker_id} missed heartbeats, marking suspect")
                        info.status = WorkerStatus.SUSPECT

    async def _result_processing_loop(self):
        """处理结果队列中的结果（可以在聚合前进行额外处理）"""
        while self._running:
            # 简化版本：结果已在 worker 循环中放入了 result_queue
            # 这里可以添加额外的处理逻辑，如缓存、去重等
            await asyncio.sleep(0.1)

    def _select_worker(self, task_spec: TaskSpec) -> str:
        """根据调度策略选择最适合的 Worker"""
        available = [
            (wid, info) for wid, info in self.workers.items()
            if info.status == WorkerStatus.IDLE
        ]

        if not available:
            # 所有 Worker 都忙，随机选择一个（或等待——这里简化处理）
            available = list(self.workers.items())

        if self.scheduling_strategy == "round_robin":
            idx = self._round_robin_index % len(available)
            self._round_robin_index += 1
            return available[idx][0]

        elif self.scheduling_strategy == "least_loaded":
            # 选择当前排队任务最少的 Worker
            return min(
                available,
                key=lambda x: len([
                    t for t in self.running_tasks.values()
                    if t[1] == x[0]
                ])
            )[0]

        elif self.scheduling_strategy == "specialized":
            # 选择与任务类型匹配的 Worker
            task_type = task_spec.task_type
            for wid, info in available:
                if task_type in info.capabilities:
                    return wid
            # 回退：选择空闲最久的
            return min(
                available,
                key=lambda x: x[1].last_heartbeat
            )[0]

        else:
            return available[0][0]

    def _validate_output(self, data: Any, schema: dict) -> bool:
        """校验输出是否符合 Schema"""
        try:
            # 简化校验：检查必要字段是否存在
            if "required" in schema:
                for field_name in schema["required"]:
                    if field_name not in data:
                        return False
            return True
        except Exception:
            return False
```

### 12.3 Worker 实现

```python
# ─── Worker 实现 ──────────────────────────────────────────────

class Worker(ABC):
    """
    抽象的 Worker 基类。

    具体 Worker 应继承此类并实现 execute() 方法。
    """

    def __init__(self, worker_id: str):
        self.worker_id = worker_id
        self.capabilities: list = ["general"]  # 子类应覆盖
        self._running = True

    @abstractmethod
    async def execute(self, task: TaskSpec) -> TaskResult:
        """
        执行单个子任务。

        这个方法是 Worker 的核心，不同的 Worker 实现不同的执行逻辑。
        可以是 LLM 调用、API 请求、数据处理等。
        """
        ...

    async def stop(self):
        """停止 Worker"""
        self._running = False


class LLMWorker(Worker):
    """
    基于 LLM 的通用 Worker。

    接受自然语言任务描述，使用 LLM 生成结果。
    """

    def __init__(
        self,
        worker_id: str,
        system_prompt: str = "You are a helpful assistant.",
        model: str = "claude-sonnet-4-20250514",
        capabilities: list | None = None,
    ):
        super().__init__(worker_id)
        self.system_prompt = system_prompt
        self.model = model
        if capabilities:
            self.capabilities = capabilities

    async def execute(self, task: TaskSpec) -> TaskResult:
        """
        使用 LLM 执行任务。

        在真实实现中，这里会调用 LLM API (Anthropic, OpenAI 等)。
        这里展示伪代码逻辑。
        """
        start_time = time.time()

        try:
            # 构建 LLM 调用参数
            messages = [
                {"role": "system", "content": self.system_prompt},
                {"role": "user", "content": self._build_prompt(task)},
            ]

            # 伪代码：实际调用 LLM API
            # response = await anthropic.messages.create(
            #     model=self.model,
            #     system=self.system_prompt,
            #     messages=messages,
            #     max_tokens=4096,
            # )
            # result_text = response.content[0].text

            # 模拟 LLM 调用（实际实现中替换为真实 API）
            result_text = (
                f"Simulated result for task: {task.description[:50]}..."
            )
            await asyncio.sleep(0.1)  # 模拟延迟

            end_time = time.time()

            return TaskResult(
                task_id=task.task_id,
                worker_id=self.worker_id,
                status=TaskStatus.SUCCESS,
                data={
                    "text": result_text,
                    "task_type": task.task_type,
                },
                metadata={
                    "start_time": start_time,
                    "end_time": end_time,
                    "exec_time": end_time - start_time,
                    "model": self.model,
                    "token_count": 0,  # 实际调用时记录
                },
            )

        except Exception as e:
            end_time = time.time()
            return TaskResult(
                task_id=task.task_id,
                worker_id=self.worker_id,
                status=TaskStatus.FAILED,
                error=str(e),
                metadata={
                    "start_time": start_time,
                    "end_time": end_time,
                },
            )

    def _build_prompt(self, task: TaskSpec) -> str:
        """根据 TaskSpec 构建 LLM 输入的 Prompt"""
        parts = [f"## Task: {task.description}"]

        if task.context:
            parts.append(f"\n## Context:\n{json.dumps(task.context, indent=2)}")

        if task.input_data:
            parts.append(f"\n## Input Data:\n{json.dumps(task.input_data, indent=2)}")

        if task.output_schema:
            parts.append(
                f"\n## Expected Output Format:\n{json.dumps(task.output_schema, indent=2)}"
            )

        return "\n".join(parts)
```

### 12.4 具体实现示例

```python
# ─── 具体实现示例 ──────────────────────────────────────────────

class MarketResearchDecomposer(TaskDecomposer):
    """市场调研任务分解器"""

    async def decompose(self, task_description: str, context: dict) -> list[TaskSpec]:
        """
        将市场调研任务分解为子任务。
        实际应用中，这里的分解策略可以由 LLM 动态生成。
        这里展示硬编码的分解逻辑作为示例。
        """
        # 解析出目标主题
        topic = context.get("topic", task_description)

        return [
            TaskSpec(
                description=f"Search and summarize recent developments in {topic}",
                task_type="search",
                context={"topic": topic, "depth": "comprehensive"},
                output_schema={
                    "type": "object",
                    "required": ["key_developments", "sources"],
                },
                priority=0,
                timeout=30.0,
            ),
            TaskSpec(
                description=f"Identify key companies and researchers in {topic}",
                task_type="search",
                context={"topic": topic, "focus": "players"},
                output_schema={
                    "type": "object",
                    "required": ["companies", "researchers"],
                },
                priority=0,
                dependencies=[],  # 示例：无依赖
                timeout=30.0,
            ),
            TaskSpec(
                description=f"Analyze market trends and forecasts for {topic}",
                task_type="analyze",
                context={"topic": topic},
                output_schema={
                    "type": "object",
                    "required": ["market_size", "growth_rate", "key_trends"],
                },
                priority=1,
                timeout=45.0,
            ),
            TaskSpec(
                description=f"Write final market research report for {topic}",
                task_type="write",
                context={"topic": topic},
                # 依赖前三个任务的结果
                dependencies=[],  # 实际应在依赖管理器中动态设置
                output_schema={
                    "type": "object",
                    "required": ["executive_summary", "analysis", "conclusions"],
                },
                priority=2,
                timeout=60.0,
            ),
        ]


class MarketResearchAggregator(ResultAggregator):
    """市场调研结果聚合器"""

    async def aggregate(self, results: list[TaskResult], original_context: dict) -> dict:
        """聚合所有子任务的结果为最终的调研报告"""
        # 按任务类型组织结果
        by_type = {}
        for result in results:
            task_type = result.data.get("task_type", "unknown")
            if task_type not in by_type:
                by_type[task_type] = []
            by_type[task_type].append(result)

        # 检查是否有失败的任务
        failures = [r for r in results if r.status in (TaskStatus.FAILED, TaskStatus.TIMEOUT)]
        if failures:
            logger.warning(f"{len(failures)} subtasks failed or timed out")

        # 构建最终报告
        report = {
            "topic": original_context.get("topic", "Unknown"),
            "summary": "Market research report generated via Manager-Worker pattern",
            "sections": {},
            "warnings": [],
        }

        # 整合搜索类结果
        search_results = by_type.get("search", [])
        report["sections"]["research_findings"] = [
            r.data.get("text", "") for r in search_results
        ]

        # 整合分析类结果
        analysis_results = by_type.get("analyze", [])
        report["sections"]["analysis"] = [
            r.data.get("text", "") for r in analysis_results
        ]

        # 整合写作类结果
        write_results = by_type.get("write", [])
        report["sections"]["report"] = [
            r.data.get("text", "") for r in write_results
        ]

        # 报告失败任务
        if failures:
            report["warnings"].append(
                f"{len(failures)} subtasks encountered issues. "
                "Report may be incomplete."
            )

        return report


# ─── 使用示例 ──────────────────────────────────────────────────

async def main():
    """演示 Manager-Worker 模式的基本用法"""
    # 1. 创建 Manager
    manager = Manager(
        decomposer=MarketResearchDecomposer(),
        aggregator=MarketResearchAggregator(),
        worker_pool_size=5,
        task_queue_maxsize=50,
        heartbeat_interval=5.0,
    )

    # 2. 启动 Manager
    await manager.start()

    try:
        # 3. 准备 Worker 池: 创建不同专长的 Worker
        search_worker = LLMWorker(
            worker_id="worker-search",
            system_prompt=(
                "You are a search and research specialist. "
                "Find comprehensive and up-to-date information. "
                "Always cite your sources."
            ),
            capabilities=["search"],
        )
        analysis_worker = LLMWorker(
            worker_id="worker-analysis",
            system_prompt=(
                "You are a data analyst specialized in market research. "
                "Provide data-driven insights and identify patterns."
            ),
            capabilities=["analyze"],
        )
        writer_worker = LLMWorker(
            worker_id="worker-writer",
            system_prompt=(
                "You are a professional report writer. "
                "Create clear, well-structured, and persuasive documents."
            ),
            capabilities=["write"],
        )

        # 注册 Worker (实际系统中可以使用不同的 Worker 实现)
        manager.worker_pool["worker-search"] = search_worker
        manager.workers["worker-search"] = WorkerInfo(
            worker_id="worker-search",
            capabilities=["search"],
        )
        # ... 类似地注册其他 Worker

        # 4. 执行任务
        result = await manager.execute(
            user_request="Conduct market research on Quantum Computing",
            context={
                "topic": "Quantum Computing",
                "depth": "standard",
                "format": "detailed",
            },
        )

        # 5. 输出结果
        print(json.dumps(result, indent=2, ensure_ascii=False))

    finally:
        # 6. 优雅关闭
        await manager.stop()


# ─── 入口 ────────────────────────────────────────────────────

if __name__ == "__main__":
    asyncio.run(main())
```

### 12.5 任务生命周期追迹

执行过程中的核心状态流转：

```
  任务 T 的生命周期:

  CREATED (由 Manager 创建)
      |
      v
  PENDING (在 Task Queue 中排队)
      |
      |  (被 Worker 取出)
      v
  ASSIGNED (分配给 Worker_W)
      |
      v
  RUNNING (Worker_W 正在执行)
      |
      +-----> SUCCESS (正常完成)
      |          |
      |          v
      |       Manager 验证结果格式
      |          |
      |          +--> 通过: 放入 Result Queue
      |          +--> 不通过: -> FAILED
      |
      +-----> FAILED (执行异常)
      |          |
      |          v
      |       Manager 判断是否重试
      |          |
      |          +--> max_retries < 已重试次数: 重新 PENDING
      |          +--> max_retries >= 已重试次数: 最终失败
      |
      +-----> TIMEOUT (超时)
          |
          v
      任务重新 PENDING，分配给其他 Worker
```

---

## 总结

Manager-Worker 模式是现代多 Agent 系统的基石。它通过"一个大脑规划、多双手执行"的朴素思想，解决了单 Agent 系统的上下文溢出、串行瓶颈和职责混杂问题。

**核心收获：**

1. **分而治之始终有效** -- 将复杂任务拆分为可独立执行的子任务，是提升 AI 系统能力上限的最可靠策略。

2. **关注点分离降低复杂度** -- Manager 管"做什么"和"做的怎么样"，Worker 管"怎么做"。每个 Agent 的系统提示词更短、更专注、更稳定。

3. **简单性是一种竞争优势** -- 相比 Parliament 或 Mesh，Manager-Worker 的实现和调试成本最低，对于 80% 的多 Agent 场景已经足够。

4. **瓶颈可预测、可管理** -- Manager 作为中心节点既是优势（易于监控）也是风险（单点故障和扩展瓶颈）。通过层次化管理、异步通信和动态 Worker 池，这些问题在实践中是可控的。

5. **没有银弹** -- Manager-Worker 不适合紧密耦合的迭代任务和大规模去中心化场景。在这些场景中，Pipeline、Parliament 或 Mesh 模式是更好的选择。

**最佳实践简要清单：**

| 实践 | 建议 |
|---|---|
| Worker 数量 | 3-7 个为宜，超过 20 个考虑层次化 |
| 任务粒度 | 子任务执行时间 >> 调度开销 |
| 错误处理 | 始终设定超时、重试和降级策略 |
| Worker 专长 | 每个 Worker 聚焦一种任务类型 |
| 结果验证 | 使用 Schema 校验 + 语义检查双重保障 |
| 可观测性 | 记录每个子任务的全链路日志 |
| 调度策略 | Least-Loaded 通用性强，Specialized 适合专长 Worker |
| 缓存 | 对幂等子任务启用语义缓存 |

---

*参考和延伸阅读：*

- Gamma, E. et al. (1994). *Design Patterns: Elements of Reusable Object-Oriented Software*. (Master-Slave pattern discussion)
- Dean, J. & Ghemawat, S. (2004). *MapReduce: Simplified Data Processing on Large Clusters*. OSDI.
- Li, G. et al. (2024). *A Survey on Multi-Agent Systems based on Large Language Models*. arXiv:2407.08392.
- Park, J. S. et al. (2024). *Generative Agents: Interactive Simulacra of Human Behavior*. UIST.
- Hong, S. et al. (2024). *MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework*. ICLR.
- Wu, Q. et al. (2024). *AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation Framework*. Microsoft Research.
- LangChain (2024). *LangGraph: Multi-Agent Workflows*. Documentation.
