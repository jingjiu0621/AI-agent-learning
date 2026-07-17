# 10.2.1 Span Design — Span 结构与设计

## 简单介绍

Span 是分布式追踪的基本工作单元，代表一个具有开始时间、结束时间和元数据的逻辑操作。在 Agent 系统中，Span 是构建完整调用链路的基础——每一次 LLM 调用、工具调用、检索操作都对应一个 Span。良好的 Span 设计决定了整个追踪系统的质量和可用性。

## 基本原理

Span 的标准结构（遵循 OpenTelemetry 规范）：

```
Span {
    trace_id:       "tr-a1b2c3d4",        # 所属 Trace 的唯一 ID
    span_id:        "sp-0001",              # 本 Span 的唯一 ID
    parent_span_id: "sp-0000",              # 父 Span 的 ID（Root Span 此字段为空）
    name:           "llm_call_thought",     # Span 名称
    kind:           SpanKind.INTERNAL,      # Span 类型
    start_time:     "2026-07-17T10:30:00Z", # 开始时间
    end_time:       "2026-07-17T10:30:00.8Z", # 结束时间
    status:         StatusCode.OK,          # 状态
    attributes: {                            # 属性（键值对）
        "llm.model": "claude-sonnet-5",
        "llm.temperature": 0.7,
        "llm.prompt_tokens": 450,
        "llm.completion_tokens": 32,
    },
    events: [                                # 事件（带时间戳的注解）
        {
            "name": "llm.request_start",
            "timestamp": "2026-07-17T10:30:00Z",
        },
    ],
    links: [                                 # 链接（关联其他 Span）
        {"trace_id": "tr-xxx", "span_id": "sp-xxx"},
    ],
}
```

Agent 系统中常见的 Span 类型：

| Span 类型 | Kind | 典型属性 | 父 Span |
|----------|------|---------|--------|
| Agent Root | SERVER | agent_id, session_id, user_id | 无 |
| Thought | INTERNAL | model, temperature, token_count | Agent Root |
| Tool Call | CLIENT | tool_name, arguments, duration | Agent Root |
| Tool Internal | INTERNAL | tool_version, cache_hit | Tool Call |
| Memory Retrieve | INTERNAL | memory_type, top_k, score | Agent Root |
| LLM Call | CLIENT | model, tokens_in, tokens_out | Agent Root / Thought |

## 背景

Span 的概念起源于 Google Dapper 论文（2010），首次在分布式系统中引入了"请求级追踪"的抽象。此后 Twitter Zipkin、Jaeger、OpenTelemetry 等系统进一步标准化了 Span 的规范和传播方式。

在 Agent 系统中，Span 设计面临新的挑战：Agent 的执行不是简单的"调用链"，而是**迭代循环**——Thought → Action → Observation → Thought。这种循环结构无法用传统的 DAG Span 树完美表达，需要额外的设计考虑。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 无追踪 | 只有日志，没有 Span | 无法从整体视角理解一次请求的耗时分布 |
| 扁平 Span | 所有操作都是根 Span 的子 Span | 丢失了操作嵌套关系，无法区分"LLM 调用是 Thought 的一部分" |
| HTTP 请求级追踪 | 只追踪 HTTP 级别的调用 | 忽略了 Agent 内部的 LLM 调用（不是 HTTP 请求但却是主要耗时） |
| 单层 Agent Span | 每个 Agent 循环步骤一个 Span | 无法展开查看步骤内部的 LLM 调用和工具调用详情 |

## 核心矛盾

| 矛盾维度 | 粗粒度 Span | 细粒度 Span |
|---------|------------|------------|
| 追踪价值 | 低（只能看总耗时） | 高（可精确定位瓶颈） |
| 性能开销 | 低 | 高（创建/记录 Span 有时间成本） |
| 存储成本 | 低 | 高 |
| 理解成本 | 低 | 中 |
| 调试效果 | 差 | 好 |

**核心矛盾**：Span 的粒度决定了追踪系统的价值。太粗粒度的 Span 无法帮助定位问题（"Agent 执行了 10 秒"——然后呢？），太细粒度的 Span 会导致追踪系统成为性能瓶颈。

**建议粒度**：将 Agent 的每次 LLM 调用、每次工具调用、每次记忆检索作为独立的 Span。函数内部的简单操作（如字符串处理）不需要 Span。

## 当前主流优化方向

1. **AST 级别的自动 Span 注入**——通过 AST 分析或装饰器自动为 Agent 框架的关键方法添加 Span，减少手动埋点
2. **Span 压缩与聚合**——对相似的 Span（如连续的 Thought 步骤）自动聚合为摘要，减少存储但保留统计信息
3. **LLM 调用 Span 的语义增强**——在 Span 属性中不仅记录 Token/Model，还记录 prompt/response 的哈希和摘要
4. **Span 生命周期管理**——Agent 的一个 Trace 可能持续数分钟到数小时（如研究型 Agent），Span 需要支持长时间运行的场景和分段上报
5. **多维度 Span 标签**——在标准 OTel 属性基础上增加 Agent 特有维度（turn_number, agent_role, task_type）

## 实现的最大挑战

1. **循环结构建模**——Agent 的 ReAct 循环中，Thought → Action → Observation 不是简单的树结构，每个循环的 Observation 又是下一个 Thought 的输入。如何用 Span 树表达这种有状态迭代是一大挑战
2. **并发 Span 管理**——当 Agent 并行调用多个工具时，多个子 Span 同时存在，需要正确的上下文隔离和合并
3. **长 Trace 的可行性**——如果 Agent 执行 50 步才完成一个任务，Trace 将包含 150+ 个 Span（50 × Thought + Action + Observation），这远超传统追踪系统的设计预期

## 能力边界

**能做什么：**
- 精确量化 Agent 执行的时间分布：LLM 调用占多少、工具调用占多少、内部处理占多少
- 直观展示 Agent 的执行流程：每个步骤的顺序、嵌套、并行关系
- 通过 Span 标签快速过滤特定类型的操作（如只看工具调用延迟）

**不能做什么：**
- 不能自动区分"好的"延迟和"坏的"延迟——Span 记录了耗时，但"正常"vs"异常"需要额外规则
- 不能解决 Agent 的推理质量问题——Tracing 关注"执行了多久"而非"执行得对不对"
- 不能处理 Span 丢失——在分布式系统中，部分 Span 可能因网络或崩溃而丢失，Trace 可能不完整

## 与其他的区别

| 对比项 | 传统 RPC Tracing | Agent Tracing |
|--------|-----------------|--------------|
| 结构 | 线性调用链（A→B→C） | 循环迭代 + 树状 |
| 典型 Span | HTTP 请求、数据库查询 | Thought、Action、Observation |
| 耗时特征 | 毫秒级 | 秒级到分钟级 |
| 关键指标 | 延迟、错误率 | Token 消耗、推理轮数 |
| 采样策略 | 固定比例（1-10%） | 自适应（按任务复杂度） |

## 最终工程优化

1. **Span 采样策略设计**——简单任务（1-3 步）100% 采样，复杂任务（10+ 步）1% 采样，保证可观测性的同时控制成本
2. **Span 异步上报**——Span 完成时先写入本地 RingBuffer，后台线程批量上报，不阻塞 Agent 主流程
3. **Span 标签索引优化**——对 agent_id、tool_name、status 等高基数查询字段建立二级索引，避免全量扫描
4. **Span 生命周期钩子**——在 Agent 框架中提供 on_span_start / on_span_end 回调，便于集成自定义逻辑（如成本计算、评价触发）
5. **Span 兜底超时**——对于长时间运行的 Agent，设置 Span 自动关闭的超时时间（如 30 分钟无新 Span 自动关闭整个 Trace）
