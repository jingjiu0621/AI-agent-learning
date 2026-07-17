# 10.2 Tracing — 链路追踪

Tracing（链路追踪）是 Agent 可观测性的第二层。如果说日志（Logging）回答的是"发生了什么"，追踪回答的是 **"这次请求经历了什么"** ——从用户输入到最终输出的完整调用路径。

## 什么是 Agent Tracing？

Tracing 将一个完整的 Agent 执行过程（一次用户请求）建模为 **Trace**，每条 Trace 由多个 **Span** 组成，每个 Span 代表一个逻辑操作单元：

```
Trace: "用户查天气" (trace_id: tr-xxx)
    │
    ├── Span: Agent 主循环 (root span, 5000ms)
    │   ├── Span: Thought #1 (LLM 调用, 800ms)
    │   ├── Span: Action #1 (get_location, 300ms)
    │   ├── Span: Observation #1 (结果解析, 50ms)
    │   ├── Span: Thought #2 (LLM 调用, 600ms)
    │   ├── Span: Action #2 (get_weather, 400ms)
    │   └── Span: Observation #2 (结果解析, 50ms)
    │
    └── [Trace 结束，Agent 回复用户]
```

## Tracing vs Logging

| 维度 | Logging | Tracing |
|------|---------|---------|
| 焦点 | 事件："当时发生了什么" | 链路："从起点到终点的路径" |
| 数据结构 | 独立事件（可关联） | 嵌套 Span 树 |
| 关键字段 | timestamp, level, message | trace_id, span_id, parent_span_id |
| 查询方式 | 全文搜索 + 过滤 | 按 trace_id 展开树 |
| 典型问题 | "Agent 为什么调用了这个工具？" | "这次请求为什么花了 10 秒？" |

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|---------|------|
| 10.2.1 span-design | 如何设计 Span 的结构和粒度？ | ★★★☆☆ |
| 10.2.2 trace-context | 如何在分布式调用中传播上下文？ | ★★★★☆ |
| 10.2.3 langsmith-integration | 如何使用 LangSmith 追踪 Agent？ | ★★★☆☆ |
| 10.2.4 open-telemetry | 如何用 OTel 实现标准追踪？ | ★★★★☆ |
| 10.2.5 custom-tracer | 如何实现自定追踪器？ | ★★★☆☆ |
| 10.2.6 distributed-tracing | 分布式 Agent 如何链路追踪？ | ★★★★★ |

## 核心设计思想

1. **Trace 即请求的生命周期**——从用户输入到 Agent 输出，Trace 贯穿 Agent 的整个处理过程，包括所有 LLM 调用、工具调用和记忆访问
2. **Span 嵌套表达因果关系**——父 Span "包含"子 Span，子 Span "支持"父 Span 的结果；嵌套结构自然地表达了操作的层次关系
3. **延迟分解是 Tracing 的核心价值**——通过 Span 的耗时信息，精确回答"时间花在了哪里"——是 LLM 调用慢、工具响应慢、还是在思考上花了太多轮
4. **Tracing 集成需要框架层面支持**——好的 Tracing 不是事后加在代码外围，而是从 Agent 框架设计之初就内建的面切关注点

## 关键工程启示

- Span 的粒度需要在"太粗（没有价值）"和"太细（性能开销大）"之间权衡：通常将"一次 LLM 调用"和"一次工具调用"分别作为一个 Span
- trace_id 应在 Agent 入口处生成，通过上下文传播贯穿整个请求链路
- Tracing 系统应支持采样——生产环境默认 10% 采样，调试时 100%
- 对于 LLM 调用 Span，除了耗时还应记录 model、token_count、temperature 等元数据

## 向下关联

- ← 10.1 logging：日志中携带 trace_id，使日志条目可以关联到 Trace
- → 10.3 monitoring：Trace 中的延迟和错误数据汇聚后成为监控指标
- → 10.4 debugging：Trace 回放是调试的强大工具
