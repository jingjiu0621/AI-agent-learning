# 8.5.4 Pipeline Pattern -- 流水线模式

## 1. 简单介绍

Pipeline Pattern（流水线模式）是将多个 Agent 按照确定的顺序排列成一条链路，每个 Agent 专精于某一阶段的处理，接收前一阶段的输出、完成自身转换后传递给下一阶段。整个系统就像一条多智能体装配线，数据从一端流入，经过一系列专业化处理后从另一端流出。

这种模式来源于工业生产中的流水线概念和 Unix 操作系统的管道（pipe）机制，是结构化多步 Agent 工作流最直观的实现方式之一。

> 文本处理流水线示例：输入文本 -> Agent A（语言检测）-> Agent B（翻译）-> Agent C（摘要）-> Agent D（格式化为 JSON）-> 结构化输出结果

---

## 2. 基本原理

### 核心机制

Pipeline 模式建立在一个简单的抽象之上：每个阶段都是一个独立的处理单元，接收定义良好的输入，产生定义良好的输出，并将结果传递给下一个阶段。数据在流水线中单向流动，每个阶段只关心自己的转换职责。

```
                          Pipeline 模式架构
    =========================================================

    输入
      │
      ▼
    ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
    │ Stage 1  │ ──► │ Stage 2  │ ──► │ Stage 3  │ ──► │ Stage 4  │
    │          │     │          │     │          │     │          │
    │ Agent A  │     │ Agent B  │     │ Agent C  │     │ Agent D  │
    │ 分词     │     │ 实体识别  │     │ 关系抽取  │     │ 图构建    │
    └──────────┘     └──────────┘     └──────────┘     └──────────┘
         │                │                │                │
         ▼                ▼                ▼                ▼
    [Token 列表]    [标注实体]      [关系三元组]       [知识图谱]
                                                         │
                                                         ▼
                                                       输出
```

### 典型数据流

每个阶段内部包含三个子步骤：

```
    ┌─────────────────────────────────────────┐
    │           单个 Stage 内部结构              │
    │                                          │
    │   输入 ──► ┌─────────────────┐ ──► 输出  │
    │            │  1. Input Port  │          │
    │            │    接收上游数据   │          │
    │            │         │        │          │
    │            │         ▼        │          │
    │            │  2. Process     │          │
    │            │   LLM 推理/函数  │          │
    │            │         │        │          │
    │            │         ▼        │          │
    │            │  3. Output Port │          │
    │            │    向下游发送    │          │
    │            └─────────────────┘          │
    └─────────────────────────────────────────┘
```

### 数据流方向

```
    输入数据流方向 ───────────────────────────────────────────────►
                                                                   
    Stage 1    Stage 2    Stage 3    Stage 4    Stage 5            
    ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐          
    │ Input├──►│ Wait ├──►│ Proc ├──►│ Enri ├──►│ Out  ├──► Result
    │ Check│   │ Valid│   │  LLM │   │  ch  │   │ Form │          
    │      │   │      │   │      │   │      │   │  at  │          
    └──────┘   └──────┘   └──────┘   └──────┘   └──────┘          
                                                                   
    每一阶段仅依赖前一阶段的输出，不跨阶段交互
```

### 缓冲区机制

在实践中，流水线阶段之间通常使用队列（Queue）作为缓冲区，实现生产者和消费者的解耦：

```
    Stage N                  缓冲区 Queue                 Stage N+1
    ┌──────────┐           ┌──────────────┐           ┌──────────┐
    │          │  enqueue  │              │  dequeue  │          │
    │ Producer │ ────────► │ [item][item] │ ────────► │ Consumer │
    │          │           │ [item][item] │           │          │
    └──────────┘           └──────────────┘           └──────────┘
                               │        │
                           bounded/unbounded
                           可选容量限制
```

缓冲区解决了以下问题：
- **速率不匹配**：快慢不一的阶段之间通过缓冲区平滑流量
- **故障隔离**：下游故障时，上游仍可继续生产到缓冲区满为止
- **批处理窗口**：缓冲区为批处理提供了积累空间

---

## 3. 背景与发展

### 起源

Pipeline 的思想在计算机科学中由来已久：

| 时期 | 来源 | 核心思想 |
|------|------|----------|
| 1960s | Unix 管道（`|`） | `cmd1 \| cmd2 \| cmd3` -- 每个命令的输出是下一个命令的输入 |
| 1970s | 流水线 CPU 架构 | 指令分为取指、译码、执行、写回，每级独立并行 |
| 1980s | MapReduce | `map` 阶段 -> `shuffle` 阶段 -> `reduce` 阶段 |
| 2000s | ETL 流水线 | Extract -> Transform -> Load，数据仓库的基石 |
| 2010s | 流处理框架 | Apache Kafka, Flink, Storm 的数据管道 |
| 2020s | LLM Agent 流水线 | 多 Agent 分工，每个 Agent 负责一个 NLP 子任务 |

### 在 Agent 系统中的演进

**第一阶段 -- 静态线性流水线：**
最早的 Agent 流水线是固定的顺序链路，每个阶段使用一个独立的 LLM 调用。典型应用如：输入 -> 意图识别 Agent -> 实体提取 Agent -> 查询 Agent -> 格式化输出 Agent。

**第二阶段 -- 条件分支流水线：**
引入路由机制，根据中间结果选择不同分支路径，形成 DAG 结构。例如：输入 -> 分类 Agent -> [如果 A 走分支 1，如果 B 走分支 2] -> 各分支专用处理 Agent。

**第三阶段 -- 动态流水线：**
流水线结构可以根据任务特性在运行时动态构建，结合 Orchestrator Agent 动态编排阶段顺序和参数。

---

## 4. 之前做法与局限

### 单体 Agent 方式（Pipeline 出现之前）

在 Pipeline 模式被引入到多 Agent 系统之前，常见的做法是使用一个全能型 Agent 完成整个工作流：

```
    ┌─────────────────────────────────────────────────────┐
    │               Monolithic Agent                      │
    │                                                     │
    │  输入                                                │
    │    │                                                 │
    │    ▼                                                 │
    │  ┌─────────────────────────────────────────────┐    │
    │  │      单次 LLM Call / 单 Agent 处理            │    │
    │  │                                             │    │
    │  │  1. 阅读输入                                 │    │
    │  │  2. 识别意图                                 │    │
    │  │  3. 提取实体                                 │    │
    │  │  4. 执行查询                                 │    │
    │  │  5. 格式化输出                               │    │
    │  │  6. 返回结果                                 │    │
    │  └─────────────────────────────────────────────┘    │
    │                                                     │
    │  输出                                                │
    └─────────────────────────────────────────────────────┘
```

### 单体方式的局限

| 问题 | 描述 |
|------|------|
| **上下文过载** | 一个 Agent 的 prompt 需要包含所有阶段的指令，长度爆炸，超出上下文窗口 |
| **专业化困难** | 无法为不同子任务使用不同的 prompt 策略、模型或工具配置 |
| **中间状态不可见** | 无法观察或干预中间结果，出错时难以定位问题阶段 |
| **调试困难** | 一次 LLM Call 内干了多件事，出错时不知道是哪个步骤导致的 |
| **复用性差** | 相同的子任务在不同工作流中需要重复编写 prompt |
| **成本浪费** | 简单任务和复杂任务使用相同的模型和资源粒度 |

### 案例对比

> 文档处理场景：输入一篇 PDF 论文要求生成结构化摘要
>
> **单体方式**：一次 LLM 调用中，prompt 包含"先提取文本、再识别关键术语、再生成摘要、再翻译成中文"。对于长文档，上下文窗口很容易撑爆；摘要质量下降时无法定位是提取阶段还是生成阶段的问题。
>
> **Pipeline 方式**：Stage 1 文本提取 Agent -> Stage 2 关键术语识别 Agent -> Stage 3 英文摘要 Agent -> Stage 4 中译 Agent。每步可单独调试、单独优化，可以混合使用不同模型（如提取用 GPT-4o，翻译用更便宜的小模型）。

---

## 5. 核心矛盾

### 5.1 顺序瓶颈 vs 专业化收益

```
    矛盾核心：
    流水线将任务分解为专业化阶段，但阶段之间的顺序依赖限制了并行度。
    
    瓶颈阶段示意：
    
    输入 ──► Stage A (100ms) ──► Stage B (500ms) ──► Stage C (100ms) ──► 输出
                                     ▲
                                     │
                             吞吐率上限 = 2 条/秒
                             整体延迟  = 700ms
                             
    Stage B 是瓶颈，即使 A 和 C 很快，系统吞吐也无法超过 B 的速度。
```

| 矛盾方面 | 专业化收益 | 流水线代价 |
|----------|-----------|-----------|
| 任务分解 | 每个 Agent 职责清晰，prompt 精简 | 阶段间引入序列化/反序列化开销 |
| 模型选择 | 可为不同阶段选用最合适的模型 | 多模型调用增加总延迟 |
| 可调试性 | 每个阶段可独立测试和监控 | 需要额外的数据传递验证机制 |
| 可扩展性 | 瓶颈阶段可独立扩容 | 扩容后的速率匹配更复杂 |

### 5.2 流水线深度 vs 延迟

```
    Pipeline 深度对延迟的影响：

    深度 2:  ┌───┐   ┌───┐
              │ A │──►│ B │──► 延迟 = 100 + 100 = 200ms
              └───┘   └───┘

    深度 4:  ┌───┐   ┌───┐   ┌───┐   ┌───┐
              │ A │──►│ B │──►│ C │──►│ D │──► 延迟 = 100×4 = 400ms
              └───┘   └───┘   └───┘   └───┘

    深度 8:  延迟 = 100×8 = 800ms（如果都是 100ms 阶段）

    结论：流水线每增加一个阶段，端到端延迟就增加一个阶段的处理时间。
          深度和延迟是线性关系，但专业化和模块化也是深度的函数。
```

### 5.3 阶段耦合 vs 独立演化

```
    理想：               现实：
    ┌──────┐             ┌──────┐
    │Stage │  松耦合       │Stage │  隐式依赖
    │  N   ├───────────►  │ N+1  │  ◄──────────
    └──────┘  明确接口    └──────┘              │
         │                                     │
         │  契约：                            Stage N+1 期望
         │  {text, entities,                   的 key 名、格式、
         │   metadata}                         schema 必须与
                                              Stage N 输出一致
```

- **强耦合表现**：下游阶段隐式依赖上游输出的字段名、格式、甚至具体措辞
- **演化代价**：修改一个阶段的输出格式可能导致下游所有阶段都需要调整
- **平衡手段**：定义严格的阶段间契约（Schema），使用版本化接口

---

## 6. 主流优化方向

### 6.1 并行流水线（每个阶段多 Worker）

最常见的优化：为瓶颈阶段部署多个并行 Worker 实例，提高吞吐量。

```
    输入队列
        │
        ▼
    ┌──────────┐
    │ Stage A  │  (1 Worker)
    │ 预处理    │
    └────┬─────┘
         │
         ▼
    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │ Stage B  │  │ Stage B  │  │ Stage B  │  (3 Workers)
    │ 实体识别  │  │ 实体识别  │  │ 实体识别  │
    └────┬─────┘  └────┬─────┘  └────┬─────┘
         │             │             │
         └──────┬──────┴──────┬──────┘
                │             │
                ▼             ▼
           ┌──────────┐  ┌──────────┐
           │ Stage C  │  │ Stage C  │  (2 Workers)
           │ 格式化    │  │ 格式化    │
           └────┬─────┘  └────┬─────┘
                │             │
                └──────┬──────┘
                       ▼
                   输出队列

    效果：瓶颈 B 从 1 个实例扩展到 3 个，吞吐率提高 3 倍
```

### 6.2 自适应批处理（Adaptive Batching）

将多个输入打包成一批交给 LLM 处理，提高吞吐和性价比。

```
    到达速率低时：逐条处理
    输入1 ──► [Batch] ──► LLM ──► 输出1
    
    到达速率高时：合并批处理
    输入1 ──┐
    输入2 ──┤              ┌─────┐
    输入3 ──┼──► [Batch]──►│ LLM │──► 输出[1,2,3]
    输入4 ──┤              └─────┘
    输入5 ──┘

    批处理窗口策略：
    - 时间窗口：每 500ms 或积累 10 条就触发
    - 数量窗口：积累到 N 条就触发
    - 自适应：根据当前队列深度动态调整窗口大小
```

### 6.3 背压机制（Backpressure）

当下游处理不过来时，信号反向传播，让上游减速或停止发送，防止系统崩溃。

```
    背压传播方向 ◄── ◄── ◄── ◄──
                                    
    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
    │ Stage  │    │ Stage  │    │ Stage  │    │ Stage  │
    │   1    │◄───│   2    │◄───│   3    │◄───│   4    │
    │ Token  │    │  LLM   │    │  LLM   │    │ Write  │
    │ izer   │    │ Infer  │    │ Enrich │    │  DB    │
    └────────┘    └────────┘    └────────┘    └────┬───┘
                                                    │
                                                    ▼
                                                ┌────────┐
                                                │  DB 写  │
                                                │ 满/慢    │
                                                └────────┘

    实现方式：
    ┌──────────────────────────────────────────────┐
    │ 1. 有限缓冲区 + 满时阻塞                       │
    │    输出队列满时，producer 的 enqueue() 阻塞      │
    │                                               │
    │ 2. 信号量 / 令牌桶                             │
    │    下游定期释放令牌，上游持有令牌才能发送           │
    │                                               │
    │ 3. 动态速率限制                                │
    │    根据下游处理能力动态调整上游的发送速率           │
    └──────────────────────────────────────────────┘
```

### 6.4 流水线分支（DAG 而非线性）

根据条件将数据路由到不同分支，形成有向无环图（DAG）。

```
    输入
      │
      ▼
    ┌──────────┐
    │ Route    │  -- 分类/路由 Agent
    │ Agent    │
    └────┬─────┘
         │
    ┌────┴────┬──────────┐
    ▼         ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│ Branch │ │ Branch │ │ Branch │
│   A    │ │   B    │ │   C    │
│ 翻译   │ │ 摘要    │ │ 问答   │
└────┬───┘ └────┬───┘ └────┬───┘
     │          │          │
     └────┬─────┴─────┬────┘
          │           │
          ▼           ▼
     ┌────────┐  ┌────────┐
     │ Merge  │  │ Merge  │
     │ Agent  │  │ Agent  │
     │ 英文   │  │ 中文   │
     └────────┘  └────────┘

    更复杂的 DAG 结构：

    ┌────────┐
    │ 输入    │
    └───┬────┘
        │
        ▼
    ┌────────┐      ┌────────┐
    │ 分词 A ├─────►│ 实体   ├──────┐
    └────────┘      │ 识别 B │      │
                    └────────┘      │
                                    ▼
    ┌────────┐      ┌────────┐  ┌────────┐
    │ 句法 C ├─────►│ 语义 D ├─►│ 融合 E │
    └────────┘      └────────┘  └────────┘
```

### 6.5 推测执行（Speculative Execution）

当无法确定路由时，同时执行多个可能的分支，丢弃不需要的结果。

```
    输入 ──► 不确定性判断
                   │
            ┌──────┴──────┐
            │              │
            ▼              ▼
       ┌────────┐    ┌────────┐
       │ Path A │    │ Path B │  (同时执行)
       │ 猜测 1  │    │ 猜测 2  │
       └────┬───┘    └────┬───┘
            │              │
            └──────┬──────┘
                   ▼
              验证判断
              │       │
          匹配 A    匹配 B
              │       │
              ▼       ▼
          采用 A    采用 B
          丢弃 B    丢弃 A

    适用场景：延迟敏感但路径不确定的任务
    代价：额外的计算资源消耗
```

### 6.6 动态阶段扩缩（Dynamic Stage Scaling）

基于实时监控指标自动调整各阶段的 Worker 数量。

```
    监控系统
    ┌──────────────────────┐
    │ Stage B Latency: 2s  │
    │ Queue Depth:   150   │  ──── 触发扩容
    │ CPU Usage:     92%   │
    └──────────┬───────────┘
               │
               ▼
    扩缩控制器
    ┌──────────────────────┐
    │ 当前: Stage B x 1    │
    │ 目标: Stage B x 3    │  ──── 扩容指令
    │ 扩容后预期延迟: ~700ms │
    └──────────────────────┘

    缩容条件：
    - 队列持续为空
    - 阶段利用率低于阈值（如 < 30%）
    - 资源预算约束
```

---

## 7. 实现挑战

### 7.1 阶段故障传播

```
    场景：Stage C 的 LLM 调用超时或返回格式错误

    ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
    │ Stage  │   │ Stage  │   │ Stage  │   │ Stage  │
    │   A    │──►│   B    │──►│   C    │──►│   D    │
    │ 成功   │   │ 成功   │   │ 失败!  │   │ 等待   │
    └────────┘   └────────┘   └───┬────┘   └────────┘
                                   │
                           ┌───────┴───────┐
                           │               │
                           ▼               ▼
                     ┌──────────┐   ┌──────────┐
                     │ 重试策略  │   │ 死信队列  │
                     │ x3 重试   │   │ 人工处理  │
                     │ 指数退避  │   │ 或丢弃   │
                     └──────────┘   └──────────┘

    故障处理策略：
    1. 重试（Retry）    -- 瞬时故障，指数退避重试
    2. 跳过（Skip）     -- 非关键阶段，传递空结果
    3. 降级（Degrade）  -- 使用备用模型或简化逻辑
    4. 终止（Abort）    -- 关键路径故障，终止整个流水线
    5. 隔离（Isolate）  -- 将故障项移到死信队列，继续处理其他项
```

### 7.2 数据序列化开销

阶段之间需要将 Agent 的输出序列化为可传递的格式（JSON、Protocol Buffers 等），对于复杂对象序列化/反序列化本身有性能开销。

```
    传输格式设计陷阱：

    反例 -- 冗余传输：
    Stage A 输出: {"result": "xxx", "raw_output": "..."}   (100KB)
    Stage B 输出: {"result": "yyy", "raw_output": "...", "stage_a_result": "xxx"}  (200KB)
    Stage C 输出: {"result": "zzz", "raw_output": "...", "stage_a_result": "xxx", "stage_b_result": "yyy"}  (300KB)

    正例 -- 最小传递：
    Stage A 输出: {"data": {...}, "meta": {"stage": "A", "version": 1}}
    Stage B 输入: {"data": {...}}  (只读取需要的数据)
    Stage B 输出: {"data": {...}, "meta": {"stage": "B", "version": 1}}
```

### 7.3 速率匹配

```
    快慢阶段之间的速率匹配问题：

    时间线：
    Stage A (100ms/条)  ────┬────┬────┬────┬────┬────┬────►
                            │    │    │    │    │    │
    Stage B (500ms/条)      └────────┘────────┘────────►
                                       ▲
                                       缓冲区持续增长

    如果不加控制：
    运行 10s 后，A 生产了 100 条，B 只消费了 20 条
    缓冲区积累 80 条 -> 内存压力 -> OOM -> 系统崩溃

    解决方案：
    - 有界缓冲区（Bounded Queue）
    - 下游背压信号
    - 自适应速率控制
```

### 7.4 死锁预防

在流水线中使用有界缓冲区和并发 Worker 时可能触发死锁。

```
    死锁场景：

    有界缓冲区大小 = 2
    Worker 数 = 2

    Worker 1: Stage A -> B
    Worker 2: Stage A -> B

    时间点 T:
    Worker 1 持有 Stage A 的输出（缓冲区满，等待 B 消费）
    Worker 2 持有 Stage A 的输出（缓冲区满，等待 B 消费）
    B 的 Worker 都在等待缓冲区有空位才能写入...
    → 循环等待！系统卡死。

    预防方法：
    1. 全局无界或足够大的中间缓冲区
    2. 使用异步 I/O 避免持有锁等待
    3. 引入超时机制，超时后回滚重试
    4. 分开生产者和消费者的线程池
```

### 7.5 流水线冲刷（Pipeline Flush）

当流水线中的某个项出错或需要终止时，已分布在多个阶段中的部分处理结果需要被正确清理。

```
    冲刷前：
    ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
    │ A(完成) │──►│ B(完成) │──►│ C(处理中)│──►│ D(排队) │
    └────────┘   └────────┘   └────────┘   └────────┘
                                            （待处理）
    错误发生在项 #123 的 Stage C

    冲刷策略：
    ┌─────────────────────────────────────────────────────┐
    │ 立即冲刷（Immediate Flush）                           │
    │ 丢弃所有在途项，可能导致已处理部分丢失                   │
    │                                                      │
    │ 优雅冲刷（Graceful Flush）                            │
    │ 当前项完成当前阶段后丢弃，保留其他未受影响项的上下文       │
    │                                                      │
    │ 事务性冲刷（Transactional Flush）                      │
    │ 只有全部阶段成功才算完成，任何阶段失败则整体回滚           │
    └─────────────────────────────────────────────────────┘
```

---

## 8. 能力与边界

### 适用场景（Pipeline 擅长的领域）

| 场景 | 示例 | 原因 |
|------|------|------|
| 明确定义的多步转换 | ETL 流水线、文档处理 | 每个步骤职责清晰，接口固定 |
| 数据处理工作流 | 数据清洗 -> 特征提取 -> 模型推理 | 单向数据流，天然适合流水线 |
| 内容生成流水线 | 大纲 -> 草稿 -> 润色 -> 格式化 | 逐步精炼，每步聚焦不同层面 |
| 流式/批处理系统 | 实时日志处理、消息管道 | 流水线的缓冲和并发机制天然匹配 |
| 审核链 | 内容检查 -> 合规审查 -> 最终审批 | 前后依赖，顺序明确的审批流程 |
| 代价分层处理 | 先用小模型过滤，再用大模型精处理 | 不同阶段使用不同资源，节约成本 |

### 不适用场景（Pipeline 的边界）

| 场景 | 不适用的原因 |
|------|-------------|
| 需要来回交互的任务 | 流水线是单向的，不支持循环和反向通信 |
| 递归精炼（Iterative Refinement） | 流水线阶段之间不支持反馈循环 |
| 动态重新规划 | 流水线结构在运行时固定，难以动态调整阶段 |
| 需要全局上下文的任务 | 每个阶段只看到前一个阶段的输出，缺乏全局视图 |
| 高度不确定的路径 | 如果每个阶段都可能改变后续路径，DAG 会过于复杂 |

### 决策矩阵

```
    选择 Pipeline 模式的决策树：

    任务需要多步处理？
        ├── 否 → 单 Agent 即可
        └── 是 → 各步之间有顺序依赖？
                  ├── 否 → 考虑 Manager-Worker 或 Mesh 模式
                  └── 是 → 各步的接口是固定的？
                            ├── 否 → 需要更灵活的模式（Hybrid）
                            └── 是 → Pipeline 模式很合适
                                      │
                                      需要考虑：
                                      - 流水线深度（不要超过 5-8 级）
                                      - 是否有瓶颈阶段（需要并行化）
                                      - 是否需要条件分支（转为 DAG）
```

---

## 9. 与其他模式对比

### 9.1 Pipeline vs Manager-Worker

```
    Manager-Worker:                         Pipeline:
    ┌─────────┐                             ┌────┐   ┌────┐   ┌────┐
    │ Manager │──► Worker A                 │ A  │──►│ B  │──►│ C  │
    │         │──► Worker B                 └────┘   └────┘   └────┘
    │         │──► Worker C
    └─────────┘
    扇出（Fan-out）                        链式（Chain）
    并发执行                               顺序执行
```

| 维度 | Pipeline | Manager-Worker |
|------|----------|----------------|
| 数据流 | 链式顺序 | 中心化扇出 |
| 并发模型 | 阶段间流水线并发 | 任务级并行 |
| 扩展性 | 按阶段扩缩 Worker | 增加 Worker 即可 |
| 职责划分 | 按处理阶段 | 按任务分发 |
| 全局协调 | 不需要中心协调 | 需要 Manager 协调 |

### 9.2 Pipeline vs Mesh

```
    Pipeline（线性）:
    A ──► B ──► C ──► D

    Mesh（全连接）:
         ┌───┐
    ┌────┤ A ├────┐
    │    └───┘    │
    ▼             ▼
  ┌───┐         ┌───┐
  │ B │◄────────│ C │
  └───┘         └───┘
    │             │
    └──────┬──────┘
           ▼
         ┌───┐
         │ D │
         └───┘
```

| 维度 | Pipeline | Mesh |
|------|----------|------|
| 通信拓扑 | 相邻节点 | 任意节点 |
| 复杂度 | O(n) 连接 | O(n^2) 连接 |
| 消息路由 | 简单转发 | 需要路由表 |
| 故障影响 | 下游全部中断 | 可绕路 |
| 适用场景 | 固定顺序流程 | 自由交互 |

### 9.3 Pipeline + Manager 混合

Pipeline 可以与 Manager-Worker 结合形成混合架构：Manager 将任务分发到不同的流水线分支。

```
    ┌──────────┐
    │ Manager  │
    └────┬─────┘
         │
    ┌────┴────┬───────────┬───────────┐
    ▼         ▼           ▼           ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│Pipeline│ │Pipeline│ │Pipeline│ │Pipeline│
│  A1    │ │  A2    │ │  B1    │ │  B2    │
│ A→B→C  │ │ A→B→D  │ │ E→F→G  │ │ E→F→H  │
└────────┘ └────────┘ └────────┘ └────────┘
    │         │           │           │
    └────┬────┴───────────┴──────┬────┘
         │                       │
         ▼                       ▼
    ┌────────┐             ┌────────┐
    │ Merge  │             │ Merge  │
    │ Type 1 │             │ Type 2 │
    └────────┘             └────────┘
```

---

## 10. 核心优势

### 10.1 概念模型简单

流水线的语义清晰直观，数据从一段流向另一端，每个阶段做一件事。这种模型易于理解、沟通和维护。

### 10.2 易于监控和调试

```
    Pipeline 中的每个阶段都是可观测的边界：

    输入 ──► ┌──────────┐ ──► ┌──────────┐ ──► ┌──────────┐ ──► 输出
              │ Stage A  │     │ Stage B  │     │ Stage C  │
              │          │     │          │     │          │
              │ 输入/输出 │     │ 输入/输出 │     │ 输入/输出 │
              │ 可记录    │     │ 可记录    │     │ 可记录    │
              │ 可检验    │     │ 可检验    │     │ 可检验    │
              └──────────┘     └──────────┘     └──────────┘

    监控能力：
    - 每阶段的输入和输出可持久化（便于审计）
    - 每阶段的延迟、吞吐率可独立度量
    - 错误可精确定位到是哪个阶段的哪一步出了问题
    - 中间结果可人工审查和干预
```

### 10.3 天然适合流式处理

Pipeline 的缓冲区和异步特性使其天然支持流式场景：数据项可以逐个或批量流过流水线，而无需等待所有数据就绪。

### 10.4 阶段可组合

```
    阶段的可组合性：

    [A] → [B] → [C]         流水线 1
    [A] → [D] → [E]         流水线 2（复用 A）
    [X] → [B] → [Y]         流水线 3（复用 B）

    阶段库（Stage Library）：
    ┌────────────────────────────────────────┐
    │ TextExtractor    EntityRecognizer      │
    │ TextTranslator   Summarizer           │
    │ JsonFormatter    MarkdownConverter    │
    │ SQLGenerator     CodeReviewer         │
    │ ...                                   │
    └────────────────────────────────────────┘

    这些阶段可以像乐高积木一样组合成不同的流水线。
```

### 10.5 资源优化

不同阶段可以使用不同配置的模型和计算资源：

| 阶段 | 模型 | 成本 | 延迟要求 |
|------|------|------|----------|
| 语言检测 | 小型模型（如 TinyBERT） | 低 | 不重要 |
| 实体提取 | 中型模型 | 中 | 重要 |
| 关系推理 | 大型模型（如 GPT-4o） | 高 | 关键 |

---

## 11. 工程优化

### 11.1 阶段池化（Stage Pooling）

为瓶颈阶段部署多个 Worker 实例，每个 Worker 运行相同的阶段逻辑，通过负载均衡器分发任务。

```
    输入队列 ──► 负载均衡 ──┬──► Stage B Worker 1
                            ├──► Stage B Worker 2
                            ├──► Stage B Worker 3
                            └──► Stage B Worker N
                                      │
                                      ▼
                                输出队列 ──► Stage C

    扩缩策略：
    ┌─────────────────────────────────────────────┐
    │ 指标             阈值           动作          │
    │─────────────────────────────────────────────│
    │ queue_depth      > 100         增加 Worker   │
    │ queue_depth      == 0          减少 Worker   │
    │ p99_latency      > 2000ms      增加 Worker   │
    │ worker_util      < 30%         减少 Worker   │
    └─────────────────────────────────────────────┘
```

### 11.2 阶段间断路器（Circuit Breaker）

当下游阶段连续失败时，断路器打开，快速拒绝请求而非等待超时，防止级联故障。

```
    正常状态：断路器关闭
    Stage B ──► 断路器 ──► Stage C
         │         │           │
         │     正常转发         │
         └────────┴────────────┘

    故障状态：断路器打开
    Stage B ──► 断路器 ──► Stage C
         │         │           │
         │     快速失败         │
         │    返回错误          │
         └─────────────────────┘

    半开状态：尝试恢复
    Stage B ──► 断路器 ──► Stage C
         │         │           │
         │  放行一个请求        │
         │  ┌── 成功？──┐      │
         │  │          │       │
         │ 关闭        │ 再次打开│
         └─────────────┘───────┘
```

### 11.3 检查点持久化（Checkpoint Persistence）

在每个阶段完成后保存中间结果，故障时可从最近的检查点恢复。

```
    检查点存储（持久化存储：DB / 对象存储 / 文件系统）

    ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
    │ Stage  │   │ Stage  │   │ Stage  │   │ Stage  │
    │   A    │──►│   B    │──►│   C    │──►│   D    │
    └───┬────┘   └───┬────┘   └───┬────┘   └───┬────┘
        │            │            │            │
        ▼            ▼            ▼            ▼
    ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
    │chkpt_A │   │chkpt_B │   │chkpt_C │   │chkpt_D │
    │{...}   │   │{...}   │   │{...}   │   │{...}   │
    └────────┘   └────────┘   └────────┘   └────────┘

    恢复流程：
    1. 检测到故障（Stage C 崩溃）
    2. 读取最近的检查点 chkpt_B
    3. 从 Stage C 重放，使用 B 的输出作为输入
    4. 继续后续步骤

    检查点策略：
    - 全部检查：每阶段都保存（安全但开销大）
    - 关键检查：只在关键阶段后保存（风险权衡）
    - 增量检查：只保存和上一个检查点不同的数据
```

### 11.4 流水线指标

| 指标 | 类型 | 说明 |
|------|------|------|
| `stage_process_time` | Histogram | 每阶段的处理时间 |
| `stage_queue_depth` | Gauge | 每阶段输入队列深度 |
| `stage_throughput` | Counter | 每阶段每秒处理条数 |
| `stage_error_rate` | Counter | 每阶段错误次数 / 总处理数 |
| `stage_retry_count` | Counter | 每阶段重试次数 |
| `pipeline_end_to_end_latency` | Histogram | 完整流水线的总延迟 |
| `pipeline_items_in_flight` | Gauge | 流水线中正在处理的项目数 |
| `backpressure_events` | Counter | 背压触发次数 |
| `circuit_breaker_state` | Gauge | 断路器状态（0=关闭, 1=半开, 2=打开） |

### 11.5 数据契约（Schema Contract）

定义阶段间传递数据的 Schema，保证前后兼容：

```
    ┌─────────────────────────────────────────────────────┐
    │  Stage A 输出 Schema v1.0                          │
    │  {                                                   │
    │    "text":       string,        // 处理后的文本       │
    │    "metadata":  {                                    │
    │      "source":    string,                            │
    │      "timestamp": int                                │
    │    },                                                │
    │    "entities":   [                                   │
    │      {                                               │
    │        "name":    string,                            │
    │        "type":    "PERSON"|"ORG"|"LOC",              │
    │        "offset":  [int, int]                         │
    │      }                                               │
    │    ]                                                 │
    │  }                                                   │
    │                                                      │
    │  向后兼容原则：                                       │
    │  - 可以添加新字段（下游忽略不认识的字段）                │
    │  - 不能删除或重命名现有字段                             │
    │  - 可以扩展现有枚举值（下游应优雅处理未知值）             │
    └─────────────────────────────────────────────────────┘
```

---

## 12. 代码示例

以下示例展示一个 提取 -> 转换 -> 加载（ETL）流水线的完整 Python 伪代码实现。

### 12.1 PipelineStage 抽象基类

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Generic, Optional, TypeVar
import time
import json
import logging

logger = logging.getLogger(__name__)

# 类型变量：输入类型和输出类型
T = TypeVar("T")
U = TypeVar("U")


class StageStatus(Enum):
    """阶段处理状态"""
    SUCCESS = "success"
    FAILURE = "failure"
    SKIPPED = "skipped"
    RETRYING = "retrying"


@dataclass
class StageContext:
    """流水线上下文，携带全局元数据和追踪信息"""
    pipeline_id: str
    item_id: str
    created_at: float = field(default_factory=time.time)
    metadata: dict = field(default_factory=dict)
    checkpoint_dir: Optional[str] = None


@dataclass
class StageResult(Generic[U]):
    """阶段处理结果"""
    status: StageStatus
    output: Optional[U] = None
    error: Optional[str] = None
    latency_ms: float = 0.0
    retry_count: int = 0


class PipelineStage(ABC, Generic[T, U]):
    """
    流水线阶段的抽象基类。
    
    每个阶段接收类型 T 的输入，产生类型 U 的输出。
    子类需要实现 process() 方法。
    """
    
    def __init__(self, name: str, max_retries: int = 3, timeout_ms: int = 30000):
        self.name = name
        self.max_retries = max_retries
        self.timeout_ms = timeout_ms
        self._metrics = {
            "processed": 0,
            "succeeded": 0,
            "failed": 0,
            "total_latency_ms": 0.0,
        }
    
    def execute(self, input_data: T, context: StageContext) -> StageResult[U]:
        """
        执行阶段处理，包含重试逻辑和指标收集。
        """
        start_time = time.time()
        retry_count = 0
        last_error = None
        
        while retry_count <= self.max_retries:
            try:
                if retry_count > 0:
                    logger.info(f"[{self.name}] Retry {retry_count}/{self.max_retries}")
                
                output = self.process(input_data, context)
                latency = (time.time() - start_time) * 1000
                
                self._update_metrics(success=True, latency=latency)
                
                return StageResult(
                    status=StageStatus.SUCCESS,
                    output=output,
                    latency_ms=latency,
                    retry_count=retry_count,
                )
                
            except Exception as e:
                last_error = e
                retry_count += 1
                logger.warning(f"[{self.name}] Error: {e}")
                
                if retry_count <= self.max_retries:
                    # 指数退避
                    wait_time = 2 ** retry_count
                    logger.info(f"[{self.name}] Waiting {wait_time}s before retry...")
                    time.sleep(wait_time)
        
        latency = (time.time() - start_time) * 1000
        self._update_metrics(success=False, latency=latency)
        
        return StageResult(
            status=StageStatus.FAILURE,
            error=str(last_error),
            latency_ms=latency,
            retry_count=retry_count,
        )
    
    @abstractmethod
    def process(self, input_data: T, context: StageContext) -> U:
        """子类实现具体的处理逻辑"""
        pass
    
    def _update_metrics(self, success: bool, latency: float):
        self._metrics["processed"] += 1
        if success:
            self._metrics["succeeded"] += 1
        else:
            self._metrics["failed"] += 1
        self._metrics["total_latency_ms"] += latency
    
    def get_metrics(self) -> dict:
        """获取阶段统计指标"""
        total = self._metrics["processed"]
        return {
            "name": self.name,
            "processed": total,
            "succeeded": self._metrics["succeeded"],
            "failed": self._metrics["failed"],
            "avg_latency_ms": (
                self._metrics["total_latency_ms"] / total if total > 0 else 0
            ),
            "success_rate": (
                (self._metrics["succeeded"] / total * 100) if total > 0 else 0
            ),
        }
```

### 12.2 具体阶段实现

```python
# ======== Stage 1: Extract - 文本提取 ========

@dataclass
class ExtractOutput:
    """提取阶段输出"""
    raw_text: str
    source: str
    length: int
    language: str


class TextExtractStage(PipelineStage[str, ExtractOutput]):
    """
    从原始输入中提取文本内容。
    示例：从 URL、文件路径或原始文本中提取。
    """
    
    def __init__(self):
        super().__init__(name="TextExtractor")
    
    def process(self, input_data: str, context: StageContext) -> ExtractOutput:
        # 模拟 LLM 调用或规则引擎
        logger.info(f"[TextExtractStage] Extracting text from source...")
        
        # 实际项目中这里可能调用 LLM 或使用文档解析库
        extracted_text = input_data  # 简化：直接传递
        
        return ExtractOutput(
            raw_text=extracted_text,
            source=context.metadata.get("source", "direct_input"),
            length=len(extracted_text),
            language=self._detect_language(extracted_text),
        )
    
    def _detect_language(self, text: str) -> str:
        # 简化的语言检测
        if len(text) > 0:
            return "zh" if any("一" <= c <= "鿿" for c in text[:100]) else "en"
        return "unknown"


# ======== Stage 2: Transform - 实体提取与转换 ========

@dataclass
class Entity:
    name: str
    entity_type: str
    confidence: float


@dataclass
class TransformOutput:
    """转换阶段输出"""
    entities: list[Entity]
    summary: str
    enriched_text: str


class TransformStage(PipelineStage[ExtractOutput, TransformOutput]):
    """
    从提取的文本中识别实体并生成摘要。
    使用 LLM 进行分析。
    """
    
    def __init__(self):
        super().__init__(name="EntityTransformer", max_retries=2)
    
    def process(self, input_data: ExtractOutput, context: StageContext) -> TransformOutput:
        logger.info(f"[TransformStage] Processing {input_data.length} chars...")
        
        # 模拟 LLM 调用进行实体识别和摘要生成
        # 实际项目中：llm_response = call_llm(input_data.raw_text, prompt="...")
        
        entities = [
            Entity(name="Sample Corp", entity_type="ORG", confidence=0.95),
            Entity(name="John Doe", entity_type="PERSON", confidence=0.88),
        ]
        
        summary = f"文档共 {input_data.length} 字，识别到 {len(entities)} 个实体。"
        
        return TransformOutput(
            entities=entities,
            summary=summary,
            enriched_text=input_data.raw_text,
        )
    
    def validate_input(self, input_data: ExtractOutput) -> bool:
        """输入验证：拒绝空文本"""
        return len(input_data.raw_text.strip()) > 0


# ======== Stage 3: Load - 格式化和输出 ========

@dataclass
class LoadOutput:
    """加载阶段输出"""
    formatted_result: dict
    output_path: str


class LoadStage(PipelineStage[TransformOutput, LoadOutput]):
    """
    将转换后的数据格式化为目标结构并输出。
    """
    
    def __init__(self, output_path: str = "./output"):
        super().__init__(name="ResultFormatter")
        self.output_path = output_path
    
    def process(self, input_data: TransformOutput, context: StageContext) -> LoadOutput:
        logger.info(f"[LoadStage] Formatting results...")
        
        formatted = {
            "pipeline_id": context.pipeline_id,
            "item_id": context.item_id,
            "summary": input_data.summary,
            "entities": [
                {"name": e.name, "type": e.entity_type, "confidence": e.confidence}
                for e in input_data.entities
            ],
            "enriched_length": len(input_data.enriched_text),
            "processed_at": time.time(),
        }
        
        # 实际项目这里可能写入数据库或文件
        logger.info(f"[LoadStage] Output ready: {json.dumps(formatted, indent=2)}")
        
        return LoadOutput(
            formatted_result=formatted,
            output_path=f"{self.output_path}/{context.pipeline_id}_{context.item_id}.json",
        )
```

### 12.3 Pipeline 编排器

```python
@dataclass
class PipelineConfig:
    """流水线配置"""
    name: str
    checkpoint_enabled: bool = True
    max_concurrent_items: int = 10
    buffer_size: int = 100
    on_failure: str = "stop"  # "stop" | "skip" | "dead_letter"


class PipelineOrchestrator:
    """
    Pipeline 编排器。
    负责串联各阶段、管理缓冲区、处理错误、收集指标。
    """
    
    def __init__(self, config: PipelineConfig):
        self.config = config
        self.stages: list[PipelineStage] = []
        self._stage_buffers: dict[str, list] = {}
        self._metrics = {
            "items_submitted": 0,
            "items_completed": 0,
            "items_failed": 0,
            "stage_metrics": {},
        }
    
    def add_stage(self, stage: PipelineStage) -> "PipelineOrchestrator":
        """添加阶段到流水线"""
        self.stages.append(stage)
        self._stage_buffers[stage.name] = []
        self._metrics["stage_metrics"][stage.name] = {
            "input_count": 0,
            "output_count": 0,
        }
        return self
    
    def run(self, item_id: str, input_data: Any, metadata: dict = None) -> Optional[Any]:
        """
        运行一个 item 通过整个流水线。
        返回最终输出，或失败时返回 None。
        """
        context = StageContext(
            pipeline_id=self.config.name,
            item_id=item_id,
            metadata=metadata or {},
        )
        
        self._metrics["items_submitted"] += 1
        current_input = input_data
        
        for stage in self.stages:
            # 检查断路器（简化实现）
            stage_metrics = self._metrics["stage_metrics"][stage.name]
            
            # 执行阶段
            self._metrics["stage_metrics"][stage.name]["input_count"] += 1
            result = stage.execute(current_input, context)
            
            if result.status == StageStatus.SUCCESS:
                current_input = result.output
                self._metrics["stage_metrics"][stage.name]["output_count"] += 1
                
                # 可选的检查点保存
                if self.config.checkpoint_enabled:
                    self._save_checkpoint(context, stage.name, result.output)
                    
            elif result.status == StageStatus.FAILURE:
                logger.error(
                    f"[Pipeline] Stage '{stage.name}' failed for item '{item_id}': "
                    f"{result.error}"
                )
                self._metrics["items_failed"] += 1
                
                if self.config.on_failure == "stop":
                    return None
                elif self.config.on_failure == "skip":
                    continue
                elif self.config.on_failure == "dead_letter":
                    self._send_to_dead_letter(item_id, stage.name, result.error)
                    return None
        
        self._metrics["items_completed"] += 1
        return current_input
    
    def run_batch(self, items: list[tuple[str, Any]]) -> list[tuple[str, Optional[Any]]]:
        """
        批量运行多个 item 通过流水线。
        """
        results = []
        for item_id, input_data in items:
            output = self.run(item_id, input_data)
            results.append((item_id, output))
        return results
    
    def _save_checkpoint(self, context: StageContext, stage_name: str, data: Any):
        """保存检查点（简化）"""
        if context.checkpoint_dir:
            checkpoint_path = f"{context.checkpoint_dir}/{context.item_id}_{stage_name}.json"
            # 序列化并保存
            logger.info(f"[Pipeline] Checkpoint saved: {checkpoint_path}")
    
    def _send_to_dead_letter(self, item_id: str, stage_name: str, error: str):
        """发送失败 item 到死信队列"""
        logger.warning(f"[Pipeline] Dead letter: item='{item_id}', stage='{stage_name}', error='{error}'")
        # 实际项目写入死信队列或数据库
    
    def get_pipeline_metrics(self) -> dict:
        """获取完整流水线指标"""
        stage_metrics = {
            name: stage.get_metrics()
            for name, stage in zip(
                [s.name for s in self.stages],
                self.stages
            )
        }
        
        total_processed = self._metrics["items_submitted"]
        return {
            "pipeline": self.config.name,
            "items_submitted": total_processed,
            "items_completed": self._metrics["items_completed"],
            "items_failed": self._metrics["items_failed"],
            "success_rate": (
                (self._metrics["items_completed"] / total_processed * 100)
                if total_processed > 0 else 0
            ),
            "stages": stage_metrics,
            "num_stages": len(self.stages),
        }


# ======== 带背压的有界缓冲区 ========

from threading import Semaphore, Lock
from collections import deque


class BoundedBuffer:
    """
    有界缓冲区，用于阶段之间解耦和背压控制。
    """
    
    def __init__(self, capacity: int = 100):
        self.capacity = capacity
        self._queue = deque()
        self._mutex = Lock()
        self._not_empty = Semaphore(0)
        self._not_full = Semaphore(capacity)
        self._closed = False
    
    def put(self, item: Any) -> bool:
        """
        放入一个 item。如果缓冲区满则阻塞（背压）。
        返回 True 成功放入，False 表示已关闭。
        """
        self._not_full.acquire()  # 如果满则阻塞 → 背压
        
        with self._mutex:
            if self._closed:
                self._not_full.release()
                return False
            self._queue.append(item)
        
        self._not_empty.release()
        return True
    
    def get(self) -> Optional[Any]:
        """
        取出一个 item。如果缓冲区空则阻塞。
        返回 None 表示已关闭且无数据。
        """
        self._not_empty.acquire()
        
        with self._mutex:
            if self._closed and not self._queue:
                self._not_empty.release()
                return None
            item = self._queue.popleft()
        
        self._not_full.release()
        return item
    
    def close(self):
        """关闭缓冲区，唤醒所有等待的线程"""
        with self._mutex:
            self._closed = True
        self._not_empty.release()  # 唤醒 getter
        self._not_full.release()   # 唤醒 putter
    
    @property
    def size(self) -> int:
        with self._mutex:
            return len(self._queue)
    
    @property
    def is_full(self) -> bool:
        with self._mutex:
            return len(self._queue) >= self.capacity
    
    @property
    def is_empty(self) -> bool:
        with self._mutex:
            return len(self._queue) == 0


# ======== 并行流水线 Worker ========

from concurrent.futures import ThreadPoolExecutor, as_completed
import threading


class ParallelStageWorker:
    """
    并行执行一个阶段的多个 Worker 实例。
    通过有界缓冲区与上下游解耦。
    """
    
    def __init__(
        self,
        stage: PipelineStage,
        num_workers: int = 2,
        input_buffer: BoundedBuffer = None,
        output_buffer: BoundedBuffer = None,
    ):
        self.stage = stage
        self.num_workers = num_workers
        self.input_buffer = input_buffer
        self.output_buffer = output_buffer
        self._running = False
        self._worker_threads = []
    
    def start(self):
        """启动所有 Worker 线程"""
        self._running = True
        self._worker_threads = [
            threading.Thread(target=self._worker_loop, daemon=True)
            for _ in range(self.num_workers)
        ]
        for t in self._worker_threads:
            t.start()
        logger.info(
            f"[ParallelStageWorker] Started {self.num_workers} workers "
            f"for stage '{self.stage.name}'"
        )
    
    def _worker_loop(self):
        """Worker 主循环：从输入缓冲区读取 -> 处理 -> 写入输出缓冲区"""
        while self._running:
            try:
                item = self.input_buffer.get()
                if item is None:
                    break  # 缓冲区关闭
                
                input_data, context = item
                result = self.stage.execute(input_data, context)
                
                if result.status == StageStatus.SUCCESS:
                    self.output_buffer.put((result.output, context))
                else:
                    # 失败处理
                    logger.error(
                        f"[ParallelStageWorker] Item '{context.item_id}' "
                        f"failed at '{self.stage.name}': {result.error}"
                    )
                    # 发送到死信或丢弃
                    
            except Exception as e:
                logger.error(f"[ParallelStageWorker] Worker error: {e}")
    
    def stop(self):
        """停止所有 Worker"""
        self._running = False
        for t in self._worker_threads:
            t.join(timeout=5)


# ======== 流水线使用示例 ========

def main():
    """完整的 ETL Pipeline 运行示例"""
    
    # 1. 创建阶段
    extract_stage = TextExtractStage()
    transform_stage = TransformStage()
    load_stage = LoadStage(output_path="./output")
    
    # 2. 配置流水线
    config = PipelineConfig(
        name="etl-pipeline-v1",
        checkpoint_enabled=True,
        max_concurrent_items=5,
        buffer_size=50,
        on_failure="stop",
    )
    
    # 3. 构建流水线
    pipeline = PipelineOrchestrator(config)
    pipeline.add_stage(extract_stage)
    pipeline.add_stage(transform_stage)
    pipeline.add_stage(load_stage)
    
    # 4. 运行单个 item
    sample_text = """
    Sample Corporation announced today that CEO John Doe will be stepping down.
    The company reported strong Q4 results with revenue exceeding expectations.
    """
    
    result = pipeline.run(
        item_id="item-001",
        input_data=sample_text,
        metadata={"source": "news_feed", "priority": "high"},
    )
    
    if result:
        print(f"Pipeline completed successfully!")
        print(f"Output path: {result.output_path}")
        print(f"Entities found: {len(result.formatted_result['entities'])}")
    
    # 5. 批量运行
    batch_items = [
        ("item-002", "Second document content here..."),
        ("item-003", "Third document with different content..."),
    ]
    
    batch_results = pipeline.run_batch(batch_items)
    
    # 6. 输出指标
    metrics = pipeline.get_pipeline_metrics()
    print(f"\nPipeline Metrics:")
    print(f"  Success rate: {metrics['success_rate']:.1f}%")
    print(f"  Items completed: {metrics['items_completed']}")
    for stage_name, stage_met in metrics["stages"].items():
        print(f"  Stage '{stage_name}':")
        print(f"    Processed: {stage_met['processed']}")
        print(f"    Avg latency: {stage_met['avg_latency_ms']:.1f}ms")
        print(f"    Success rate: {stage_met['success_rate']:.1f}%")


# ======== 流水线分支示例（DAG） ========

class RouterStage(PipelineStage[str, tuple[str, str]]):
    """
    路由阶段：根据输入分类路由到不同分支。
    输出 (route_key, modified_input) 供下游路由选择。
    """
    
    def process(self, input_data: str, context: StageContext) -> tuple[str, str]:
        # 判断输入类型：翻译、摘要、问答
        if "translate" in input_data.lower():
            route = "translation"
        elif "summarize" in input_data.lower():
            route = "summary"
        else:
            route = "qa"
        
        logger.info(f"[RouterStage] Routing to: {route}")
        return (route, input_data)


class BranchPipeline:
    """
    带分支的流水线：Router -> 分支 A | 分支 B | 分支 C -> Merge
    """
    
    def __init__(self):
        self.router = RouterStage()
        self.branches = {}  # route_key -> PipelineOrchestrator
        self.merge_stage = None
    
    def add_branch(self, route_key: str, pipeline: PipelineOrchestrator):
        self.branches[route_key] = pipeline
        return self
    
    def run(self, item_id: str, input_data: str) -> Optional[Any]:
        # 路由
        context = StageContext(pipeline_id="branch-pipeline", item_id=item_id)
        route_result = self.router.execute(input_data, context)
        
        if route_result.status != StageStatus.SUCCESS:
            logger.error(f"[BranchPipeline] Routing failed: {route_result.error}")
            return None
        
        route_key, data = route_result.output
        
        # 选择分支
        branch = self.branches.get(route_key)
        if branch is None:
            logger.warning(f"[BranchPipeline] No branch for route '{route_key}'")
            return None
        
        logger.info(f"[BranchPipeline] Executing branch '{route_key}'")
        return branch.run(item_id, data)


# ======== 故障恢复示例 ========

class RecoveryPipeline:
    """
    带检查点恢复的流水线。
    当流水线崩溃后，可以从最近的检查点恢复。
    """
    
    def __init__(self, stages: list[PipelineStage], checkpoint_store: dict = None):
        self.stages = stages
        self.checkpoint_store = checkpoint_store or {}
    
    def run_with_recovery(self, item_id: str, input_data: Any) -> Optional[Any]:
        # 尝试从检查点恢复
        last_checkpoint = self.checkpoint_store.get(item_id)
        
        if last_checkpoint:
            start_stage = last_checkpoint["stage_index"] + 1
            current_input = last_checkpoint["output"]
            logger.info(f"[RecoveryPipeline] Resuming from stage {start_stage}")
        else:
            start_stage = 0
            current_input = input_data
        
        # 从恢复点继续执行
        for i in range(start_stage, len(self.stages)):
            stage = self.stages[i]
            context = StageContext(
                pipeline_id="recovery-pipeline",
                item_id=item_id,
            )
            
            result = stage.execute(current_input, context)
            
            if result.status == StageStatus.SUCCESS:
                current_input = result.output
                # 保存检查点
                self.checkpoint_store[item_id] = {
                    "stage_index": i,
                    "output": result.output,
                }
            else:
                logger.error(
                    f"[RecoveryPipeline] Failed at stage {i} ('{stage.name}'): "
                    f"{result.error}"
                )
                return None
        
        # 清理完成的检查点
        self.checkpoint_store.pop(item_id, None)
        return current_input
```

### 12.4 流水线架构全景图

```
    完整的 Pipeline 系统架构：

    ┌──────────────────────────────────────────────────────────────────────┐
    │                          Pipeline Orchestrator                       │
    │  ┌──────────────────────────────────────────────────────────────┐   │
    │  │                      Pipeline 定义                            │   │
    │  │  Stage1 ──► BoundedBuf ──► Stage2 ──► BoundedBuf ──► Stage3  │   │
    │  └──────────────────────────────────────────────────────────────┘   │
    │                              │                                       │
    │         ┌────────────────────┼────────────────────┐                  │
    │         ▼                    ▼                    ▼                  │
    │  ┌─────────────┐   ┌────────────────┐   ┌────────────────┐         │
    │  │   输入适配器  │   │   监控仪表盘    │   │   持久化存储    │         │
    │  │  REST/gRPC   │   │  延迟/吞吐/错误  │   │  检查点/日志    │         │
    │  │  队列消费     │   │   实时警报      │   │  死信队列       │         │
    │  └─────────────┘   └────────────────┘   └────────────────┘         │
    │                              │                                       │
    │         ┌────────────────────┼────────────────────┐                  │
    │         ▼                    ▼                    ▼                  │
    │  ┌─────────────┐   ┌────────────────┐   ┌────────────────┐         │
    │  │   断路器管理  │   │   扩缩容控制器   │   │   错误处理      │         │
    │  │  故障隔离    │   │   Worker 池    │   │   重试/降级     │         │
    │  │  熔断恢复    │   │   自动扩缩     │   │   死信路由     │         │
    │  └─────────────┘   └────────────────┘   └────────────────┘         │
    └──────────────────────────────────────────────────────────────────────┘
```

---

## 总结

Pipeline Pattern 是多 Agent 系统中最基础、最直观的模式之一。它将复杂任务分解为一系列顺序执行的专业化阶段，每个阶段负责一个明确的子任务。

**使用 Pipeline 的最佳时机**：
- 任务可以清晰地分解为顺序依赖的步骤
- 每个步骤的输入输出接口是固定的
- 需要独立监控、调试和优化每个步骤
- 需要为不同步骤分配不同的计算资源或模型

**避免使用 Pipeline 的时机**：
- 任务需要 Agent 之间频繁的双向通信
- 处理流程在运行时频繁变化
- 单个阶段的失败会导致整个链路的不可恢复中断

Pipeline 模式是构建可观测、可调试、可组合的多 Agent 系统的基石，也是理解更复杂模式（DAG、Hybrid）的基础。
