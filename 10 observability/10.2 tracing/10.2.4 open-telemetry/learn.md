# 10.2.4 OpenTelemetry — OpenTelemetry 追踪实现

## 简单介绍

OpenTelemetry（OTel）是 CNCF 的观测性标准，提供了一套统一的 API 和 SDK 用于生成、采集和导出遥测数据（Traces、Metrics、Logs）。在 Agent 系统中，OTel 是实现**厂商无关的链路追踪**的最佳选择——它可以将 Agent 的追踪数据导出到任何兼容后端（Jaeger、Zipkin、Datadog、Grafana Tempo、Honeycomb 等）。OTel 的 Trace API 规范（W3C Trace Context）也已成为事实上的行业标准。

## 基本原理

OTel 在 Agent 系统中的集成分为三个层次：

```
Agent 应用
    │
    ├── OTel SDK（Instrumentation）
    │   ├── 自动插装（Auto-instrumentation）:
    │   │   ├── HTTP 客户端/服务端自动追踪
    │   │   ├── gRPC 调用自动追踪
    │   │   └── 数据库驱动自动追踪
    │   │
    │   └── 手动插装（Manual Instrumentation）:
    │       ├── Agent 主循环 Span
    │       ├── LLM 调用 Span
    │       ├── 工具调用 Span
    │       └── 记忆操作 Span
    │
    ├── OTel Processor
    │   ├── BatchSpanProcessor（批量处理）
    │   ├── Attribute 富化
    │   └── 采样决策
    │
    └── OTel Exporter
        ├── OTLP gRPC Exporter → Collector
        └── OTLP HTTP Exporter → Collector
                │
                ▼
        OTel Collector
        ├── Receiver: OTLP
        ├── Processor: 过滤、采样、批处理
        └── Exporter: Jaeger / Tempo / Datadog / ...
```

核心代码示例：

```python
# 1. OTel 初始化
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="otel-collector:4317"))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

# 2. 在 Agent 主循环中手动插装
class AgentLoop:
    def __init__(self):
        self.tracer = trace.get_tracer("agent.core")

    async def run(self, user_input: str):
        # 创建根 Span
        with self.tracer.start_as_current_span("agent_run") as root_span:
            root_span.set_attribute("agent.id", self.agent_id)
            root_span.set_attribute("user.input.length", len(user_input))

            # 工具调用 Span
            with self.tracer.start_as_current_span("tool_call") as tool_span:
                tool_span.set_attribute("tool.name", "get_weather")
                tool_span.set_attribute("tool.params", '{"city": "Beijing"}')
                result = await self.call_tool("get_weather", city="Beijing")
                tool_span.set_attribute("tool.success", result.success)

            # LLM 调用 Span
            with self.tracer.start_as_current_span("llm_call") as llm_span:
                llm_span.set_attribute("llm.model", "claude-sonnet-5")
                llm_span.set_attribute("llm.tokens.prompt", 450)
                llm_span.set_attribute("llm.tokens.completion", 32)
                response = await self.call_llm(result)
```

## 背景

OTel 由 OpenTracing 和 OpenCensus 两个项目合并而成（2019），旨在解决观测性领域"标准太多，互不兼容"的问题。2021 年成为 CNCF 毕业项目，目前是云原生观测性的事实标准。

在 Agent 可观测性领域，OTel 的推广面临挑战：标准 OTel 的自动插装对 LLM API 调用没有直接支持——它无法自动理解"LLM 调用"的语义。因此 Agent 框架需要在 OTel 基础上手动添加 LLM 特有的属性和 Span 类型。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 自建追踪系统 | 团队自行实现 trace_id 传播和 Span 管理 | 重复造轮子、维护成本高、与其他系统集成困难 |
| 厂商锁定 | 使用 Datadog APM 或 New Relic 专有 API | 迁移困难、成本高、Agent 特定语义支持差 |
| 仅 LangSmith | 只使用 LangSmith 进行追踪 | LangChain 用户友好但非 LangChain 项目集成困难 |
| 不标准化 | 每个微服务使用不同的追踪方案 | 跨服务 Trace 断裂，整体可观测性为零 |

## 核心矛盾

| 矛盾维度 | 厂商锁定（DD/NR） | OTel 标准 | 自建 |
|---------|-----------------|-----------|------|
| 集成复杂度 | 低 | 中 | 高 |
| 数据主权 | ❌ | ✅ | ✅ |
| Agent 语义 | ❌ | 需自定义 | 可定制 |
| 生态工具 | 丰富 | 丰富 | 少 |
| 迁移成本 | 高 | 低 | 中 |

**核心矛盾**：OTel 提供了标准化的追踪能力，但对 Agent 特有的 LLM 调用、工具调用、记忆操作等语义缺乏原生支持。解决方案是在 OTel 基础上建立"Agent 语义约定"——定义标准的 Agent Span name 和 attribute key，使得不同 Agent 框架的追踪数据可以在同一后端中统一分析。

## 当前主流优化方向

1. **OTel Agent 语义约定**——社区正在推动 Agent/LLM 特有的 OTel 语义约定（Semantic Conventions），如 `llm.model`、`llm.tokens`、`tool.name` 等标准 attribute key
2. **GenAI OTel 扩展**——OpenTelemetry 社区推出的 GenAI 扩展，专门定义了 LLM 调用、向量检索等 AI 操作的 Span 语义规范
3. **OTel Collector 的 Agent 数据处理**——使用 OTel Collector 的 processor 对 Agent 追踪数据进行富化（添加 agent_id、session_id）、过滤（只保留关键 Span）和采样
4. **自动插装库生态**——LangChain、LlamaIndex 等框架提供了 OTel 兼容的自动插装，使得在不修改代码的情况下获得 Agent 追踪能力
5. **OTel + Prometheus 指标融合**——用 OTel 收集 Span 数据的同时提取指标（延迟直方图、错误率、Token 消耗）写入 Prometheus，实现追踪与监控的统一数据源

## 实现的最大挑战

1. **Agent 特有的 Span 语义**——目前 OTel 对 LLM 调用的语义约定还在演进中，Agent 的实现者需要在标准化完成前自行定义 attribute 命名规范
2. **高基数属性**——Agent 的 trace_id 是高基数（每秒数十万个唯一值），对 OTel Collector 和后端存储的扩展性要求高
3. **异步 Agent 的上下文传播**——Python asyncio 中 OTel 的隐式上下文传播（基于 ContextVar）可能在复杂的协程结构中丢失
4. **OTel SDK 的开销**——在 Agent 的每次 LLM/工具调用中创建 Span 有性能和内存开销，高吞吐场景下不可忽视

## 能力边界

**能做什么：**
- 标准化的分布式追踪，与 Jaeger/Tempo/Datadog 等后端兼容
- 自动插装 HTTP/gRPC/数据库调用
- 通过 OTel Collector 实现灵活的管道处理（采样、过滤、富化）
- 与已有的 OTel 基础设施无缝集成

**不能做什么：**
- 不能自动理解 LLM 调用的语义——需要手动添加 Agent 特定的 attribute
- 不能自动关联 Prompt/Response 内容——OTel Span 不鼓励存储大量数据（如完整的 Prompt 文本）
- 不能提供 LLM 特定的评估功能——OTel 是数据采集标准，不是分析平台

## 与其他的区别

| 对比项 | LangSmith | OTel | 自建 Tracer |
|--------|-----------|------|------------|
| 标准化程度 | 专有 | 行业标准 | 无 |
| 后端兼容 | 仅 LangSmith | 任意 OTel 兼容后端 | 自定义 |
| LLM 语义 | ✅ 原生 | ⚠️ 需扩展 | 取决于实现 |
| 自动插装 | LangChain 专用 | 通用（HTTP/gRPC/DB） | 无 |
| 数据量控制 | 有限 | 灵活（Collector processor） | 完全控制 |

## 最终工程优化

1. **OTel SDK 懒初始化**——Agent 启动时不初始化所有 OTel 组件，而是在第一次需要追踪时懒加载，减少对启动时间的影响
2. **采样策略配置化**——通过环境变量或配置中心动态调整 OTel 采样率（`OTEL_TRACES_SAMPLER=parentbased_traceidratio`），支持按 agent_id、task_type 等维度的差异化采样
3. **Agent attribute 标准化**——定义团队级的 Agent OTel attribute 命名规范，如 `agent.turn_number`、`agent.thought_hash`、`tool.duration_ms`，确保所有 Agent 模块使用统一的 key
4. **Collector pipeline 双层设计**——上层 Collector 负责全局处理（跨团队通用），下层 Sidecar Collector（每个 Agent Pod 一个）负责本地处理（缓存、降采样），减少网络传输
5. **Tail-based sampling**——在 OTel Collector 中实现基于结果的采样：所有 ERROR Span 100% 保留，成功 Span 按 1% 采样，确保"出错的 Trace 从不丢失"
