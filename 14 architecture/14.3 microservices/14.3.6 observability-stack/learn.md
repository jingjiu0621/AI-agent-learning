# 可观测性技术栈 (Observability Stack) for AI Agent

> 本文档面向 AI Agent 微服务架构的可观测性设计，覆盖传统三大支柱在 Agent 场景下的演进与落地。

---

## 1. 基本原理：可观测性的三大支柱

可观测性（Observability）区别于传统监控（Monitoring），核心在于**通过外部输出来推断系统内部状态**，而无需发布新代码来探测问题。

```
  ┌──────────────────────────────────────────────────┐
  │              可观测性三大支柱                       │
  │                                                   │
  │  ┌──────────┐   ┌──────────┐   ┌──────────┐      │
  │  │  Logging │   │ Metrics  │   │ Tracing  │      │
  │  │  (日志)   │   │  (指标)   │   │  (链路)   │      │
  │  └────┬─────┘   └────┬─────┘   └────┬─────┘      │
  │       │              │              │             │
  │       ▼              ▼              ▼             │
  │  ┌──────────────────────────────────────┐        │
  │  │        Why?  What?  Where?           │        │
  │  │  日志告诉你发生了什么                   │        │
  │  │  指标告诉你趋势与异常                   │        │
  │  │  链路告诉你请求路径                     │        │
  │  └──────────────────────────────────────┘        │
  └──────────────────────────────────────────────────┘
```

### 1.1 Logging（日志）
- 不可变、带时间戳的事件记录
- 结构化格式（JSON）方便机器解析
- Agent 场景下需要包含 session_id、step_id、trace_id

### 1.2 Metrics（指标）
- 可聚合的数值型数据（计数器、直方图、仪表盘）
- 低存储成本、适合长期趋势分析
- Agent 场景关注步数分布、工具延迟、Token 消耗

### 1.3 Tracing（链路追踪）
- 端到端请求的完整路径还原
- Span = 单个操作单元，Trace = Span 的有向无环图
- Agent 场景下追踪 ReAct 循环的每次迭代

---

## 2. 背景与演进：Agent 可观测性的特殊性

传统微服务可观测性关注的是**请求-响应**维度：延迟、错误率、吞吐量。Agent 系统在此基础上增加了全新的观测维度。

### 2.1 传统可观测性 vs Agent 可观测性

```
传统服务:          Agent 系统:

┌──────┐           ┌──────────────┐
│ 客户端 │──req──→│  Agent Orc   │
└──────┘          └──────┬───────┘
       ←─resp──           │
                          │  ┌──────────────┐
                          │──→   LLM Call   │ ← 需要观测推理过程
                          │  └──────┬───────┘
                          │         │
                          │  ┌──────────────┐
                          │──→   Tool A    │ ← 工具调用链
                          │  └──────────────┘
                          │         │
                          │  ┌──────────────┐
                          │──→   Tool B    │
                          │  └──────────────┘
                          │         │
                          │  ┌──────────────┐
                          │──→   LLM Call   │ ← 再次推理
                          │  └──────────────┘
                          │
                          ▼
                      ┌──────────┐
                      │  响应合成  │
                      └──────────┘
```

### 2.2 Agent 可观测性的三大核心差异

| 维度 | 传统微服务 | Agent 系统 |
|------|-----------|------------|
| 观测对象 | 请求/响应 | 推理轨迹 (Reasoning Trace) |
| 成功标准 | 2xx/4xx, 延迟 < P99 | 语义正确性, 任务完成率 |
| 调用模式 | 固定拓扑 | 动态图 (LLM 决定下一步) |
| 数据特征 | 低基数 | 高基数 (每次 prompt 不同) |
| 问题定位 | 哪个服务挂了 | 哪一步推理错了 |

### 2.3 需要观测的新维度

1. **推理轨迹 (Reasoning Trace)**: LLM 的 Thought 过程, 中间的观察与决策
2. **工具调用链 (Tool Call Chain)**: 跨越多个微服务的工具调用依赖关系
3. **语义正确性**: 不仅仅是响应时间, 回答是否合理、安全
4. **Token 消耗**: 每次推理步骤的输入/输出 Token 数, 直接影响成本
5. **循环检测**: Agent 是否陷入了死循环 (Same state revisited)

---

## 3. Logging 架构

### 3.1 结构化日志与 Trace Context 传播

Agent 系统要求日志必须携带**全链路上下文**, 才能在百万行日志中快速定位一次 Agent 会话。

```python
# Python: 结构化日志配置示例
import structlog
import logging
from opentelemetry import trace

# 配置 structlog 生成 JSON 格式日志
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.StackInfoRenderer(),
        structlog.dev.ConsoleRenderer() if __debug__
        else structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# 在 Agent 会话开始时注入上下文
from structlog.contextvars import bind_contextvars
bind_contextvars(
    session_id="session_abc123",
    user_id="user_456",
    agent_type="research-agent",
)
```

### 3.2 Agent 特定日志事件

Agent 系统的日志事件应该包含三类核心事件, 遵循 **ReAct (Reasoning + Acting)** 模式的日志规范：

| 事件类型 | 日志级别 | 关键字段 |
|---------|---------|---------|
| Thought | INFO | thought_text, step_number, session_id |
| ToolCall | INFO | tool_name, input_params, duration_ms, step_number |
| ToolResult | INFO | tool_name, success, output_truncated, token_usage |
| LLMRequest | DEBUG | model_name, prompt_tokens, temperature |
| LLMResponse | DEBUG | completion_tokens, finish_reason, latency_ms |
| Observation | INFO | observation_summary, source_tool, step_number |
| AgentError | ERROR | error_type, recoverable, step_number |
| SessionEnd | INFO | total_steps, total_tokens, task_completed, duration_sec |

```python
# Agent 日志事件示例
import structlog
logger = structlog.get_logger()

class ReActLogger:
    """ReAct 循环的结构化日志记录器"""

    def log_thought(self, step: int, thought: str, session_id: str):
        logger.info("agent.thought",
                     step=step,
                     thought_summary=thought[:200],  # 截断避免日志过载
                     session_id=session_id,
                     event_type="Thought")

    def log_tool_call(self, step: int, tool: str, params: dict,
                      session_id: str, trace_id: str):
        logger.info("agent.tool_call",
                     step=step,
                     tool_name=tool,
                     params_snapshot=str(params)[:500],
                     session_id=session_id,
                     trace_id=trace_id,
                     event_type="ToolCall")

    def log_tool_result(self, step: int, tool: str, success: bool,
                        duration_ms: float, session_id: str):
        logger.info("agent.tool_result",
                     step=step,
                     tool_name=tool,
                     success=success,
                     duration_ms=round(duration_ms, 2),
                     session_id=session_id,
                     event_type="ToolResult")

    def log_llm_call(self, step: int, model: str,
                     prompt_tokens: int, session_id: str):
        logger.debug("agent.llm_request",
                      step=step,
                      model=model,
                      prompt_tokens=prompt_tokens,
                      event_type="LLMRequest")

    def log_error(self, step: int, error: Exception,
                  recoverable: bool, session_id: str):
        logger.error("agent.error",
                      step=step,
                      error_type=type(error).__name__,
                      error_message=str(error),
                      recoverable=recoverable,
                      session_id=session_id,
                      event_type="AgentError")
```

### 3.3 日志聚合架构

```
                    Agent 微服务集群
  ┌─────────────────────────────────────────────┐
  │  Agent-1     Agent-2     Agent-N            │
  │  structlog → structlog → structlog          │
  └──────────────┬──────────────────────────────┘
                 │  JSON over stdout / TCP
                 ▼
         ┌───────────────┐
         │  Promtail /    │   日志采集器 (DaemonSet)
         │  Fluentd       │
         └───────┬───────┘
                 │
         ┌───────▼───────┐
         │   Loki / ELK   │   日志存储与索引
         │   (对象存储)    │
         └───────┬───────┘
                 │  LogQL / KQL
         ┌───────▼───────┐
         │    Grafana    │   可视化与告警
         │   Explore      │
         └───────────────┘
```

Loki 的优势在于与 Prometheus 共用同一标签体系, 适合 K8s 环境的 Agent 部署。Elasticsearch 更适合全文搜索场景（如检索 LLM 响应内容进行质量分析）。

---

## 4. Metrics 体系

Agent 系统的指标分为三个层次, 构成完整的可观测金字塔：

```
            ┌────────────────────────┐
            │   Business 业务指标     │  ← 面向产品/业务
            │  任务完成率 / 成功率    │
            └──────────┬─────────────┘
                       │
            ┌──────────▼─────────────┐
            │   Agent Agent 行为指标   │  ← 面向 Agent 开发者
            │  步数分布 / Token 消耗   │
            └──────────┬─────────────┘
                       │
            ┌──────────▼─────────────┐
            │   Service 服务指标      │  ← 面向 SRE
            │  RED: Rate/Error/Dur   │
            └────────────────────────┘
```

### 4.1 服务层指标 (Service RED)

```python
from prometheus_client import Counter, Histogram, Gauge
import time

# ── Rate (请求速率) ──
agent_requests_total = Counter(
    'agent_requests_total',
    'Total Agent requests',
    ['agent_type', 'service_version']
)

# ── Errors (错误率) ──
agent_errors_total = Counter(
    'agent_errors_total',
    'Total Agent errors by type',
    ['agent_type', 'error_category']  # llm_error, tool_error, timeout, unsafe
)

# ── Duration (延迟分布) ──
agent_request_duration_seconds = Histogram(
    'agent_request_duration_seconds',
    'Agent request latency in seconds',
    ['agent_type'],
    buckets=(0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0, 60.0, 120.0, 300.0)
    # Agent 的延迟范围比传统服务大得多
)
```

### 4.2 Agent 行为指标

```python
# ── 步数分布 ──
agent_steps_per_task = Histogram(
    'agent_steps_per_task',
    'Number of ReAct steps per task',
    ['agent_type'],
    buckets=(1, 2, 3, 5, 8, 13, 20, 30, 50)  # Fibonacci buckets
)

# ── 单步延迟 ──
agent_step_duration_seconds = Histogram(
    'agent_step_duration_seconds',
    'Duration of a single ReAct step',
    ['step_type'],  # thought, tool_call, observation
    buckets=(0.05, 0.1, 0.2, 0.5, 1.0, 2.0, 5.0, 10.0)
)

# ── 工具调用延迟 ──
agent_tool_latency_seconds = Histogram(
    'agent_tool_latency_seconds',
    'Tool execution latency',
    ['tool_name'],
    buckets=(0.01, 0.05, 0.1, 0.2, 0.5, 1.0, 2.0, 5.0, 10.0)
)

# ── Token 消耗 ──
agent_llm_token_usage = Counter(
    'agent_llm_token_usage_total',
    'Total LLM token consumption',
    ['model_name', 'token_type']  # prompt, completion
)

# ── 当前活跃会话数 ──
agent_active_sessions = Gauge(
    'agent_active_sessions',
    'Number of currently active agent sessions',
    ['agent_type']
)

# ── 工具调用成功率 ──
agent_tool_success_ratio = Gauge(
    'agent_tool_success_ratio',
    'Tool call success rate (0-1)',
    ['tool_name']
)
```

### 4.3 业务级指标

```python
# ── 任务完成率 ──
agent_task_completed = Counter(
    'agent_task_completed_total',
    'Task completion count by status',
    ['agent_type', 'status']  # success, failed, cancelled, timeout
)

# ── 语义质量评分 (需要 LLM-as-Judge 后处理) ──
agent_semantic_quality = Gauge(
    'agent_semantic_quality_score',
    'Semantic quality score (0-100) from LLM evaluation',
    ['agent_type']
)
```

### 4.4 关键 SLI/SLO 示例

```yaml
# Agent SLO 定义示例
slos:
  - name: "agent_completion_rate"
    description: "Agent 任务完成率 >= 95%"
    sli: "agent_task_completed{status='success'} / agent_task_completed_total"
    target: 0.95
    window: 28d

  - name: "agent_p99_latency"
    description: "Agent 响应 P99 <= 30s"
    sli: "agent_request_duration_seconds{p99}"
    target: 30.0
    window: 7d

  - name: "tool_call_success_rate"
    description: "工具调用成功率 >= 98%"
    sli: "sum(rate(agent_tool_success_total)) / sum(rate(agent_tool_calls_total))"
    target: 0.98
    window: 7d
```

---

## 5. Tracing 实现

### 5.1 OpenTelemetry 分布式追踪

Agent 系统的追踪远比传统 HTTP 请求追踪复杂, 因为 Agent 调用图是**动态生成**的。

```
Trace: Agent Session "session_abc"
│
├── Span: orchestrator.handle_request
│   ├── Span: agent.run (ReAct Loop Root)
│   │   ├── Span: agent.step_1.think       ← LLM 推理
│   │   ├── Span: agent.step_1.tool_call    ← 工具调用
│   │   │   ├── Span: tool.search.execute   ← 跨服务调用
│   │   │   └── Span: tool.search.parse
│   │   ├── Span: agent.step_1.observe      ← 处理结果
│   │   ├── Span: agent.step_2.think        ← 再次推理
│   │   ├── Span: agent.step_2.tool_call
│   │   │   └── Span: tool.code_runner.exec
│   │   └── Span: agent.step_3.think        ← 最终推理
│   └── Span: agent.response_synthesis
│
└── Span: audit.log_persistence
```

### 5.2 Span 设计实现

```python
from opentelemetry import trace
from opentelemetry.trace import SpanKind, Status, StatusCode
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider, SpanProcessor
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
import time

# ── 初始化 Tracer ──
resource = Resource.create({
    "service.name": "agent-orchestrator",
    "service.version": "1.0.0",
    "deployment.environment": "production",
})

provider = TracerProvider(resource=resource)
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://tempo:4317"))
)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)


class ReActSpanContext:
    """ReAct 循环的 Span 管理器"""

    def __init__(self, session_id: str, agent_type: str):
        self.session_id = session_id
        self.agent_type = agent_type
        self.root_span = None
        self.step_spans = []

    def start_session(self, task: str):
        """创建根 Trace Span"""
        self.root_span = tracer.start_span(
            name="agent.session",
            kind=SpanKind.SERVER,
            attributes={
                "session.id": self.session_id,
                "agent.type": self.agent_type,
                "task.description": task[:500],
            },
        )
        self.root_span.__enter__()
        return self.root_span

    def end_session(self, success: bool, total_steps: int, total_tokens: int):
        """结束根 Span 并设置状态"""
        self.root_span.set_attribute("agent.total_steps", total_steps)
        self.root_span.set_attribute("agent.total_tokens", total_tokens)
        self.root_span.set_status(StatusCode.OK if success else StatusCode.ERROR)
        self.root_span.__exit__(None, None, None)

    def start_step_span(self, step_type: str, step_num: int) -> object:
        """创建步骤级 Span (thought / tool_call / observe)"""
        span = tracer.start_span(
            name=f"agent.step.{step_type}",
            kind=SpanKind.INTERNAL,
            attributes={
                "step.number": step_num,
                "step.type": step_type,
                "session.id": self.session_id,
            },
        )
        span.__enter__()
        self.step_spans.append(span)
        return span

    def end_step_span(self, span, attributes: dict = None):
        """结束步骤 Span 并补充属性"""
        if attributes:
            span.set_attributes(attributes)
        span.set_status(StatusCode.OK)
        span.__exit__(None, None, None)
```

### 5.3 Trace Context 通过 gRPC 传播

Agent 调用下游工具服务时, 需要通过 gRPC metadata 传播 trace context：

```python
import grpc
from opentelemetry.propagate import inject, extract
from opentelemetry.trace.propagation.tracecontext import (
    TraceContextTextMapPropagator
)

class InstrumentedToolClient:
    """带追踪的工具调用客户端"""

    def call_tool(self, tool_name: str, payload: dict) -> dict:
        """通过 gRPC 调用工具服务, 携带 Trace Context"""
        # 创建子 span
        with tracer.start_as_current_span(
            f"tool.call.{tool_name}",
            kind=SpanKind.CLIENT,
            attributes={
                "tool.name": tool_name,
                "payload.size": len(str(payload)),
            },
        ) as span:
            # 将 trace context 注入到 gRPC metadata
            metadata = {}
            inject(metadata)  # 注入 traceparent / tracestate

            # 构造 gRPC 调用
            try:
                start = time.time()
                response = self._grpc_call(
                    tool_name, payload,
                    metadata=metadata  # 携带上下文
                )
                duration = time.time() - start

                span.set_attribute("rpc.duration_ms", round(duration * 1000, 2))
                span.set_status(StatusCode.OK)
                return response

            except Exception as e:
                span.set_status(StatusCode.ERROR, str(e))
                span.record_exception(e)
                raise
```

---

## 6. Agent 特有的可观测性

### 6.1 ReAct 循环 Span 可视化

ReAct 循环的可视化是 Agent 调试的核心工具。通过 Grafana Tempo 的 TraceQL 可以实现循环级别的可视化查询。

```
TraceQL 查询示例:
  { resource.service.name = "agent-orchestrator" }
  |  span.agent.type = "research-agent"
  |  span.session.id = "session_abc"
  |  { span.step.type = "thought" } && { span.step.type = "tool_call" }

可视化输出 (Grafana Tempo 瀑布图):

Service: agent-orchestrator  Session: session_abc  Duration: 12.3s
──────────────────────────────────────────────────────────────
agent.session                   [─────────────────12.3s──────────]
├─ agent.step.1.thought         [──0.8s─]    ← LLM 推理
├─ agent.step.1.tool_call       [    ──2.1s──]  ← 调用 Search API
│  └─ tool.search.execute       [    ──1.9s─]  ← 跨服务
├─ agent.step.1.observe         [       ─0.1s]
├─ agent.step.2.thought         [        ─1.2s─]
├─ agent.step.2.tool_call       [           ──3.5s──]  ← 调用 Code Runner
│  └─ tool.code_runner.exec     [           ──3.3s─]
├─ agent.step.2.observe         [              ─0.1s]
├─ agent.step.3.thought         [               ─0.9s─]
└─ agent.response_synthesis     [                  ─1.8s─]
```

### 6.2 工具调用依赖图

当 Agent 在一个任务中调用多个工具, 且工具之间有数据依赖时, 需要生成**动态依赖图**。

```
                    ┌──────────────┐
                    │  User Query  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Step 1      │
                    │  Thought     │──→ "我需要先搜索知识库"
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Step 2      │
                    │  Tool Call   │──→ knowledge_base.search
                    │  输出: doc_id │    └── 返回文档列表
                    └──────┬───────┘
                           │ doc_id
                    ┌──────▼───────┐
                    │  Step 3      │
                    │  Tool Call   │──→ document_reader.read(doc_id)
                    │  输出: content│    └── 依赖 Step 2 的输出
                    └──────┬───────┘
                           │ content
                    ┌──────▼───────┐
                    │  Step 4      │
                    │  Tool Call   │──→ code_interpreter.run
                    │  输出: result │    └── 依赖 Step 3 的输出
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Step 5      │
                    │  Thought     │──→ "基于计算结果总结"
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Final       │
                    │  Response    │
                    └──────────────┘
```

### 6.3 LLM Prompt/Response 审计

LLM 的输入输出审计是 Agent 可观测性的关键差异化能力, 用于调试、安全审查和合规。

```python
import hashlib
from typing import Optional
from datetime import datetime

class LLMAuditLogger:
    """LLM Prompt/Response 审计日志 (敏感数据处理)"""

    def __init__(self, storage_backend: str = "s3"):
        self.storage = storage_backend
        # 注意: Prompt 可能包含 PII, 需要脱敏策略

    def record_llm_interaction(
        self,
        session_id: str,
        step_num: int,
        model: str,
        prompt: str,
        response: str,
        prompt_tokens: int,
        completion_tokens: int,
        latency_ms: float,
    ):
        """记录 LLM 交互, 使用 content-addressed 存储"""
        prompt_hash = hashlib.sha256(prompt.encode()).hexdigest()
        response_hash = hashlib.sha256(response.encode()).hexdigest()

        # 元数据写入可查询的索引
        audit_record = {
            "session_id": session_id,
            "step_num": step_num,
            "model": model,
            "timestamp": datetime.utcnow().isoformat(),
            "prompt_hash": prompt_hash,
            "response_hash": response_hash,
            "prompt_tokens": prompt_tokens,
            "completion_tokens": completion_tokens,
            "latency_ms": latency_ms,
            "prompt_truncated": len(prompt) > 10000,
            "response_truncated": len(response) > 10000,
            # 完整 prompt/response 存储在对象存储中
            "prompt_uri": f"s3://agent-audit/prompts/{prompt_hash}.json",
            "response_uri": f"s3://agent-audit/responses/{response_hash}.json",
        }

        # 写入时序数据库 (用于查询元数据)
        self._write_to_index(audit_record)

        # 完整内容写入对象存储 (低成本)
        self._write_to_object_store(
            f"prompts/{prompt_hash}.json", {
                "content": prompt,
                "session_id": session_id,
                "step_num": step_num,
            }
        )
        self._write_to_object_store(
            f"responses/{response_hash}.json", {
                "content": response,
                "session_id": session_id,
                "step_num": step_num,
            }
        )

    def _write_to_index(self, record: dict):
        """写入可查询的索引 (Elasticsearch 或 Loki)"""
        # logger.info("llm.audit", **record)
        pass

    def _write_to_object_store(self, key: str, data: dict):
        """写入低成本对象存储"""
        # s3_client.put_object(Bucket="agent-audit", Key=key, Body=json.dumps(data))
        pass
```

---

## 7. 完整代码示例

### 7.1 Agent 服务的完整 OpenTelemetry 植入

```python
"""agent_observability.py — Agent 服务的完整可观测性植入"""

import time
import json
import structlog
from typing import Any, Dict, Optional

from opentelemetry import trace
from opentelemetry.trace import SpanKind, StatusCode
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
from opentelemetry.metrics import (
    MeterProvider, Counter, Histogram, Gauge,
)
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.metrics import MeterProvider as SDKMeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader

from prometheus_client import start_http_server

# ─── 初始化 ─────────────────────────────────────────────────

def init_observability(service_name: str = "agent-orchestrator",
                       otlp_endpoint: str = "http://localhost:4317"):
    """初始化完整的可观测性栈"""

    resource = Resource.create({
        "service.name": service_name,
        "service.version": "1.0.0",
    })

    # ── Tracing ──
    tracer_provider = TracerProvider(resource=resource)
    tracer_provider.add_span_processor(
        BatchSpanProcessor(
            OTLPSpanExporter(endpoint=otlp_endpoint)
        )
    )
    trace.set_tracer_provider(tracer_provider)

    # ── Metrics (OTLP + Prometheus 双通道) ──
    metric_reader = PeriodicExportingMetricReader(
        OTLPMetricExporter(endpoint=otlp_endpoint),
        export_interval_millis=15000,
    )
    meter_provider = SDKMeterProvider(
        resource=resource,
        metric_readers=[metric_reader],
    )

    # 启动 Prometheus HTTP 端点 (用于本地抓取)
    start_http_server(9090)

    tracer = trace.get_tracer(service_name)
    meter = meter_provider.get_meter(service_name)

    return tracer, meter


# ─── 可观测的 Agent 基类 ────────────────────────────────────

class ObservableAgent:
    """带完整可观测性的 Agent 基类"""

    def __init__(self, agent_type: str, tracer, meter):
        self.agent_type = agent_type
        self.tracer = tracer
        self.meter = meter

        # 定义 Metrics
        self.m_steps = meter.create_histogram(
            name="agent.steps_per_task",
            description="Number of ReAct steps per task",
            unit="1",
        )
        self.m_step_duration = meter.create_histogram(
            name="agent.step_duration_ms",
            description="Duration per ReAct step",
            unit="ms",
        )
        self.m_tool_latency = meter.create_histogram(
            name="agent.tool_latency_ms",
            description="Tool execution latency",
            unit="ms",
        )
        self.m_token_usage = meter.create_counter(
            name="agent.token_usage_total",
            description="Total token consumption",
            unit="1",
        )
        self.m_errors = meter.create_counter(
            name="agent.errors_total",
            description="Total errors by category",
        )

    def run(self, task: str, session_id: str) -> Dict[str, Any]:
        """运行 Agent 任务, 带 Tracing 埋点"""

        with self.tracer.start_as_current_span(
            "agent.run",
            kind=SpanKind.SERVER,
            attributes={
                "session.id": session_id,
                "agent.type": self.agent_type,
                "task.preview": task[:200],
            },
        ) as root_span:
            start_time = time.time()
            steps = 0
            total_tokens = 0

            try:
                # ReAct 循环
                for step in self._react_loop(task, session_id):
                    steps += 1
                    total_tokens += step.get("tokens_used", 0)

                    # 记录步数指标
                    self.m_step_duration.record(
                        step.get("duration_ms", 0),
                        {"step_type": step.get("type", "unknown")},
                    )

                # 记录最终指标
                elapsed = (time.time() - start_time) * 1000
                self.m_steps.record(steps, {"agent_type": self.agent_type})
                self.m_token_usage.add(total_tokens, {"agent_type": self.agent_type})

                root_span.set_attribute("agent.total_steps", steps)
                root_span.set_attribute("agent.total_tokens", total_tokens)
                root_span.set_attribute("agent.duration_ms", elapsed)
                root_span.set_status(StatusCode.OK)

                return {"status": "success", "steps": steps, "tokens": total_tokens}

            except Exception as e:
                self.m_errors.add(1, {
                    "agent_type": self.agent_type,
                    "error_type": type(e).__name__,
                })
                root_span.set_status(StatusCode.ERROR, str(e))
                root_span.record_exception(e)
                return {"status": "error", "error": str(e)}

    def _react_loop(self, task: str, session_id: str):
        """ReAct 循环 (模拟)"""
        # 实际项目中这里会循环调用 LLM + 工具
        for i in range(3):
            with self.tracer.start_as_current_span(
                f"agent.step.{i}",
                attributes={
                    "step.number": i,
                    "session.id": session_id,
                },
            ) as step_span:
                step_start = time.time()

                # Thought 阶段
                thought = self._think(task, i)

                if i < 2:  # 中间步骤: 调用工具
                    tool_result = self._call_tool("search", {"q": task})
                    yield {
                        "type": "tool_call",
                        "tokens_used": 100,
                        "duration_ms": (time.time() - step_start) * 1000,
                    }
                else:  # 最终步骤: 输出
                    yield {
                        "type": "thought",
                        "tokens_used": 50,
                        "duration_ms": (time.time() - step_start) * 1000,
                    }

    def _think(self, task: str, step: int) -> str:
        """调用 LLM 推理 (模拟)"""
        return f"Thinking about {task} step {step}"

    def _call_tool(self, tool: str, params: dict) -> dict:
        """调用工具 (模拟)"""
        with self.tracer.start_as_current_span(
            f"tool.{tool}",
            kind=SpanKind.CLIENT,
            attributes={"tool.name": tool},
        ):
            time.sleep(0.1)  # 模拟工具延迟
            return {"result": "mock"}


# ─── 启动 ───────────────────────────────────────────────────

if __name__ == "__main__":
    tracer, meter = init_observability()
    agent = ObservableAgent("research-agent", tracer, meter)
    result = agent.run("分析最近的 AI 论文趋势", session_id="demo_001")
    print(json.dumps(result, indent=2))
```

### 7.2 自定义 Span Processor 用于 ReAct 循环分析

```python
class ReActSpanProcessor(SpanProcessor):
    """自定义 Span Processor: 检测 ReAct 循环异常模式"""

    def __init__(self):
        self._step_counts = {}  # session_id → step_count

    def on_start(self, span, parent_context=None):
        """Span 开始时检测可能的循环"""
        if span.attributes.get("step.type") == "thought":
            session_id = span.attributes.get("session.id")
            if not session_id:
                return

            # 追踪每个 session 的步数
            count = self._step_counts.get(session_id, 0) + 1
            self._step_counts[session_id] = count

            # 检测死循环: 超过阈值
            if count > 20:
                span.add_event(
                    "potential_infinite_loop_detected",
                    {
                        "session_id": session_id,
                        "step_count": count,
                        "threshold": 20,
                    },
                )
                # 可以触发告警或熔断
                span.set_attribute("alert.loop_detected", True)

    def on_end(self, span):
        """Span 结束时清理上下文"""
        # 如果 session 结束, 清理计数器
        if "session.id" in span.attributes and span.name == "agent.session":
            session_id = span.attributes["session.id"]
            self._step_counts.pop(session_id, None)

    def shutdown(self):
        pass

    def force_flush(self, timeout_millis: int = 30000):
        pass
```

### 7.3 Grafana Dashboard JSON 定义 (Agent 监控)

```json
{
  "title": "Agent Observability Dashboard",
  "panels": [
    {
      "title": "Agent 请求速率 & 错误率",
      "type": "timeseries",
      "datasource": "Prometheus",
      "targets": [
        {
          "expr": "sum(rate(agent_requests_total[5m])) by (agent_type)",
          "legendFormat": "{{agent_type}} - 请求"
        },
        {
          "expr": "sum(rate(agent_errors_total[5m])) by (agent_type)",
          "legendFormat": "{{agent_type}} - 错误"
        }
      ]
    },
    {
      "title": "ReAct 步数分布 (P50/P95/P99)",
      "type": "stat",
      "datasource": "Prometheus",
      "targets": [
        {
          "expr": "histogram_quantile(0.50, sum(rate(agent_steps_per_task_bucket[5m])) by (le))",
          "legendFormat": "P50"
        },
        {
          "expr": "histogram_quantile(0.95, sum(rate(agent_steps_per_task_bucket[5m])) by (le))",
          "legendFormat": "P95"
        },
        {
          "expr": "histogram_quantile(0.99, sum(rate(agent_steps_per_task_bucket[5m])) by (le))",
          "legendFormat": "P99"
        }
      ]
    },
    {
      "title": "Token 消耗 (按模型)",
      "type": "barchart",
      "datasource": "Prometheus",
      "targets": [
        {
          "expr": "sum(rate(agent_llm_token_usage_total[5m])) by (model_name, token_type)",
          "legendFormat": "{{model_name}} - {{token_type}}"
        }
      ]
    },
    {
      "title": "工具延迟热点图",
      "type": "heatmap",
      "datasource": "Prometheus",
      "targets": [
        {
          "expr": "sum(rate(agent_tool_latency_seconds_bucket[5m])) by (le, tool_name)",
          "legendFormat": "{{tool_name}}"
        }
      ]
    },
    {
      "title": "活跃 Agent 会话数",
      "type": "stat",
      "datasource": "Prometheus",
      "targets": [
        {
          "expr": "sum(agent_active_sessions) by (agent_type)",
          "legendFormat": "活跃会话"
        }
      ],
      "thresholds": [
        {"value": 100, "color": "green"},
        {"value": 500, "color": "yellow"},
        {"value": 1000, "color": "red"}
      ]
    },
    {
      "title": "Trace 详情 (Tempo)",
      "type": "trace",
      "datasource": "Tempo",
      "targets": [
        {
          "query": "{resource.service.name=\"agent-orchestrator\"}",
          "queryType": "traceql"
        }
      ]
    }
  ]
}
```

---

## 8. 工具栈推荐

### 8.1 开源方案 (LGTM Stack)

```
                          ┌─────────────────┐
                          │   Grafana UI    │
                          │   Explore/Alert │
                          └──┬───┬───┬──────┘
                             │   │   │
               ┌─────────────┘   │   └─────────────┐
               ▼                 ▼                 ▼
        ┌──────────┐    ┌────────────┐   ┌──────────────┐
        │          │    │            │   │              │
        │  Loki    │    │ Prometheus │   │   Tempo      │
        │  (日志)   │    │  (指标)     │   │  (链路追踪)    │
        │          │    │            │   │              │
        └────┬─────┘    └─────┬──────┘   └──────┬───────┘
             │                │                  │
             └────────────────┼──────────────────┘
                              │
                     ┌────────▼────────┐
                     │  OpenTelemetry  │
                     │  Collector      │
                     │  (数据管道)      │
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │  Agent 微服务    │
                     │  (OTel SDK)     │
                     └─────────────────┘
```

| 组件 | 作用 | Agent 场景优势 |
|------|------|---------------|
| OpenTelemetry SDK | 代码植入, 生成 span/metric/log | 统一 API, 多语言支持 |
| OpenTelemetry Collector | 数据接收、处理、导出 | 可以过滤高基数 span, 采样控制 |
| Grafana Tempo | 分布式追踪存储 | TraceQL 支持 Agent 循环查询 |
| Loki | 日志聚合 | 与 Prometheus 标签体系一致 |
| Prometheus | 指标存储与告警 | 成熟稳定, Agent 指标聚合 |
| Grafana | 可视化与告警 | 统一面板, Trace/Log/Metric 关联 |

### 8.2 商业化/托管方案

| 方案 | 适用场景 | 核心能力 |
|------|---------|---------|
| LangSmith | LangChain 生态 | LLM Call 追踪, Prompt 版本管理, 在线评估 |
| Semantic Kernel | .NET / Azure 生态 | 内置的 Agent 诊断, Azure Monitor 集成 |
| Datadog APM | 全栈观测 | AI Ops 自动异常检测, Agent trace 分析 |
| New Relic AI Monitoring | LLM 应用 | LLM 响应质量评分, Token 成本分析 |
| Arize AI | ML 可观测性 | Embedding 漂移检测, Prompt 安全扫描 |

### 8.3 选型决策树

```
Agent 系统规模?
├─ 实验阶段 (每日 < 1K 会话)
│  └─ LangSmith (快速上手, 零配置)
├─ 生产阶段 (每日 1K-100K 会话)
│  ├─ 已有 Grafana 基础设施?
│  │  ├─ 是 → OpenTelemetry + LGTM
│  │  └─ 否 → Datadog (托管, 省运维)
└─ 大规模 (每日 > 100K 会话)
   └─ OpenTelemetry + 自建 LGTM
      └─ 必须自定义采样策略控制成本
```

---

## 9. 最大挑战

### 9.1 Agent Trace 的高基数问题

**问题**: 每次 LLM prompt 不同 → span 属性高基数 → 存储成本爆炸。

```
传统服务 Span 示例:
  HTTP GET /api/users/123  →  路径固定, 基数低

Agent Span 示例:
  agent.step.1.thought  →  prompt = "分析...新...趋势..."
  agent.step.2.thought  →  prompt = "基于上一步结果..."
  # 每次 prompt 内容不同, 不可枚举
```

**解决方案**:

```yaml
# OpenTelemetry Collector 配置: 采样策略
processors:
  tail_sampling:
    policies:
      # 1. 全部保存: 错误 Trace
      - name: error-sampling
        type: status_code
        config:
          status_code: ERROR
          sampling_percentage: 100

      # 2. 按比例采样: 成功 Trace
      - name: success-sampling
        type: probabilistic
        config:
          sampling_percentage: 10  # 只保存 10%

      # 3. 慢 Trace 全部保存
      - name: slow-sampling
        type: latency
        config:
          threshold_ms: 30000   # 超过 30 秒的 Trace
          sampling_percentage: 100

  attributes:
    actions:
      # 移除高基数属性, 只保留摘要
      - key: "agent.thought_content"
        action: delete  # 不存储到 Trace DB
      - key: "agent.tool_params"
        action: delete
```

### 9.2 存储成本

| 数据类型 | 存储需求 | 典型 Retention | 1K 会话/天的成本估算 |
|---------|---------|---------------|-------------------|
| Trace (全量) | ~50 KB/会话 | 7 天 | ~350 MB |
| Trace (10% 采样) | ~5 KB/会话 | 30 天 | ~150 MB |
| Metrics | ~1 KB/会话 | 90 天 | ~90 MB |
| Logs (结构化) | ~10 KB/会话 | 30 天 | ~300 MB |
| LLM Audit (对象存储) | ~100 KB/会话 | 90 天 | ~9 GB |

### 9.3 Prompt 中的敏感数据

```python
class PromptSanitizer:
    """Prompt 脱敏处理器: 在写入可观测性系统前移除敏感信息"""

    SENSITIVE_PATTERNS = [
        (r'api_key["\']?\s*[:=]\s*["\'][^"\']+["\']', '[REDACTED_API_KEY]'),
        (r'password["\']?\s*[:=]\s*["\'][^"\']+["\']', '[REDACTED_PASSWORD]'),
        (r'token["\']?\s*[:=]\s*["\'][^"\']+["\']', '[REDACTED_TOKEN]'),
        (r'\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b', '[REDACTED_IP]'),  # IP
        (r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b', '[REDACTED_EMAIL]'),
    ]

    @classmethod
    def sanitize(cls, text: str) -> str:
        """对文本执行脱敏"""
        result = text
        for pattern, replacement in cls.SENSITIVE_PATTERNS:
            result = re.sub(pattern, replacement, result)
        return result

    @classmethod
    def sanitize_span(cls, span):
        """遍历 span 属性并脱敏"""
        for key in list(span.attributes.keys()):
            if any(kw in key for kw in ['prompt', 'input', 'query', 'params']):
                value = span.attributes[key]
                if isinstance(value, str):
                    span.set_attribute(
                        key + ".sanitized",
                        cls.sanitize(value)
                    )
                    span.set_attribute(key, '[REDACTED]')
```

---

## 10. 能力边界

### 10.1 可观测性开销对 Agent 延迟的影响

```
Agent 延迟分解 (典型值):

基础 Agent 处理:              ────────── 10s ──────────
                                   |
LLM API 调用:                 ───── 8s ─────
                                   |
工具执行:                     ── 1.5s ──
                                   |
可观测性开销:                 ─0.5s─ ← 不可忽略!
    ├─ SDK 内嵌采集:          ~50ms
    ├─ Span 导出 (批处理):     ~100ms
    ├─ 指标聚合上报:           ~50ms
    ├─ 日志序列化:            ~30ms
    └─ 审计日志写入:          ~270ms (对象存储)
```

### 10.2 可观测性自身的性能优化策略

```python
class LazyObservability:
    """延迟/异步可观测性: 减少对 Agent 主路径的影响"""

    def __init__(self):
        self._buffer = []
        self._batch_size = 10
        self._flush_interval = 5.0  # seconds

    def record_step(self, step_data: dict):
        """将观测数据缓冲到内存, 不阻塞主流程"""
        self._buffer.append(step_data)

        if len(self._buffer) >= self._batch_size:
            # 异步刷新 (不等待完成)
            import threading
            threading.Thread(
                target=self._flush,
                daemon=True,
            ).start()

    def _flush(self):
        """批量导出观测数据"""
        batch = self._buffer[:self._batch_size]
        self._buffer = self._buffer[self._batch_size:]

        # 批量写入
        self._export_spans(batch)
        self._export_metrics(batch)
        self._export_logs(batch)

    def _export_spans(self, batch: list):
        """批量导出 Span"""
        # 使用 gRPC streaming 批量发送
        pass

    def _export_metrics(self, batch: list):
        """批量导出指标"""
        pass

    def _export_logs(self, batch: list):
        """批量导出日志"""
        pass
```

### 10.3 已知的限制

| 限制 | 说明 | 缓解措施 |
|------|------|---------|
| Trace 基数爆炸 | 每个 Agent Trace 包含 10-50+ Span | Head/Tail 采样, 属性裁剪 |
| LLM 延迟掩盖问题 | LLM API 本身的 2-10s 延迟掩盖了基础设施问题 | 独立监控 LLM API 延迟, 与 Agent 延迟分开 |
| 语义正确性难以自动化 | 需要 LLM-as-Judge, 增加了额外成本和延迟 | 只对 1-5% 的采样会话做评估 |
| 分布式追踪上下文丢失 | Agent 通过消息队列异步调用时, Trace Context 丢失 | 使用消息头传播 W3C Trace Context |
| 存储成本不可控 | 每个 Agent 会话产生大量结构化数据 | 分层存储 (热/温/冷), 自动 Retention Policy |
| 调试延迟 | 从异常发生到可观测性系统反映异常存在 15s-60s 延迟 | 关键告警使用 Prometheus AlertManager, 不依赖 Trace |

### 10.4 何时关闭可观测性

在以下场景中, 可观测性开销可能不可接受:

1. **高吞吐实时 Agent**: 每秒处理 100+ 个简单请求, 可观测性 SDK 会成比例增加 CPU 开销
2. **资源受限部署**: Edge 设备、低配容器, 可观测性 SDK 可能占用 10-20% 的 CPU
3. **Agent 内部子循环**: 微秒级的小工具调用, 植入 OTEL Span 的开销占比过高

**策略**: 使用动态采样, 根据系统负载自动调整采样率。

```yaml
# 自适应采样配置
adaptive_sampling:
  enabled: true
  # 低负载 (< 100 req/s): 100% 采样
  load_lt_100_rps: 1.0
  # 正常负载 (100-500 req/s): 50% 采样
  load_100_500_rps: 0.5
  # 高负载 (500-1000 req/s): 10% 采样
  load_500_1000_rps: 0.1
  # 过载 (> 1000 req/s): 仅采样错误
  load_gt_1000_rps:
    mode: "errors_only"
```

---

## 附录: 关键概念速查

| 术语 | 定义 | Agent 场景含义 |
|------|------|---------------|
| Trace | 一次端到端请求的完整路径 | 一次 Agent 会话的全部 ReAct 循环 |
| Span | Trace 中的一个操作单元 | 一次 Thought / Tool Call / Observation |
| Span Attributes | Span 的键值对元数据 | step_number, tool_name, prompt_tokens |
| Span Events | Span 内的时间点事件 | LLM 响应到达, 工具调用开始 |
| RED Metrics | Rate / Errors / Duration | Agent 请求速率、错误率、P99 延迟 |
| Cardinality | 唯一标签组合的数量 | Agent prompt 内容的高基数问题是核心挑战 |
| Sampling | 只存储部分 Trace 数据 | 必须做, 否则存储成本不可控 |
| Baggage | 跨服务传播的上下文键值对 | Session ID、User ID 在工具链中传递 |
| ReAct Loop | Agent 的推理-行动循环 | 需要特殊的 Span 设计来可视化循环迭代 |
