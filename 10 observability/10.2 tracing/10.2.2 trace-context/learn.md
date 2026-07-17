# 10.2.2 Trace Context — 链路上下文传播

## 简单介绍

Trace Context（追踪上下文）是分布式追踪的"粘合剂"——它负责将 trace_id、span_id 等信息从一个服务传递到下一个服务，使得分散在不同进程/机器上的 Span 可以串联成一条完整的 Trace。在 Agent 系统中，上下文传播尤其重要：一次 Agent 执行可能跨越 Agent 框架 → LLM API → 工具 API → 数据库等多个系统。

## 基本原理

Trace Context 的传播通过**载体（Carrier）**进行：发送方将上下文信息注入到请求中，接收方从中提取并创建子 Span。

```
进程 A（Agent 框架）                    进程 B（工具服务）
┌──────────────────┐                ┌──────────────────┐
│ Span: root        │                │ Span: tool_exec  │
│ trace_id: tr-xxx  │     HTTP       │ trace_id: tr-xxx │
│ span_id: sp-001   │ ────────────→  │ parent_id: sp-001│
│ context: {         │   Headers     │                   │
│   trace_id,        │ ←──────────── │                   │
│   span_id,         │               └──────────────────┘
│   baggage          │
│ }                  │
└──────────────────┘
```

HTTP 传播的标准格式（W3C Trace Context）：

```python
# 发送方注入上下文
import requests
from opentelemetry import propagate

headers = {}
propagate.inject(headers)
# headers = {
#   "traceparent": "00-tr-a1b2c3d4e5f6-sp-0001-01",
#   "tracestate": "agent=tool_call,turn=3",
# }
response = requests.get("https://tool-api.example.com/weather", headers=headers)
```

```python
# 接收方提取上下文
from opentelemetry import propagate

def handle_request(request):
    ctx = propagate.extract(request.headers)
    # ctx 包含 trace_id 和 parent_span_id
    with tracer.start_as_current_span("tool_exec", context=ctx) as span:
        # 这个 span 自动成为调用者的子 span
        result = process_request(request)
        return result
```

Agent 系统中上下文传播的途径：

| 传播途径 | 载体 | 典型场景 | 可靠性 |
|---------|------|---------|--------|
| HTTP Headers | traceparent header | Agent → 工具 API 调用 | 高 |
| gRPC Metadata | Metadata 键值对 | Agent → 微服务调用 | 高 |
| Message Queue | 消息属性 | Agent → 异步任务 | 中（需无损传递） |
| LLM API | 无标准支持 | Agent → API 调用 | 低（需用额外机制） |
| 进程内 | ContextVar | 框架内部模块间 | 高 |

## 背景

Trace Context 的标准化经历了三个阶段：
1. **厂商自定义**——各 APM 厂商（Datadog、NewRelic、Zipkin）各自定义了自己的传播格式，互不兼容
2. **W3C Trace Context 标准**——2019 年 W3C 发布了 Trace Context 标准（`traceparent` + `tracestate`），成为业界统一规范
3. **Agent 系统的挑战**——Agent 调用 LLM API 时，LLM 提供商通常不支持传播用户自定义的上下文，导致 Agent → LLM 调用段的追踪断裂

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 不传播 | 每个服务各自生成 trace_id | 无法关联跨服务的调用 |
| 自定义 Header | 团队自己定义 X-Trace-Id Header | 缺乏标准，跨团队协作困难 |
| 仅 HTTP 传播 | 只通过 HTTP Header 传播 | 无法覆盖 gRPC、消息队列等场景 |
| application 层传递 | 在业务参数中显式传递 trace_id | 侵入性强，污染业务代码 |

## 核心矛盾

| 矛盾维度 | 不传播上下文 | 标准传播 | 完全传播 |
|---------|------------|---------|---------|
| 实现复杂度 | 0 | 低 | 高（需适配所有协议） |
| Trace 完整性 | 断裂 | 覆盖 HTTP 调用 | 覆盖所有调用 |
| 调试价值 | 低 | 高 | 最高 |
| 对业务代码影响 | 无 | 小（中间件/拦截器） | 中 |

**核心矛盾**：LLM API 调用是 Agent 的核心操作，但 LLM 提供商通常不支持 W3C Trace Context 传播，导致 Trace 中最大的耗时段（LLM 调用）无法被正确追踪。解决方案是在 Agent 框架层面手动关联：将 LLM 调用建模为 Span，在调用前后记录时间戳，即使无法将上下文传递给 LLM 服务端。

## 当前主流优化方向

1. **W3C Trace Context + Baggage**——traceparent 用于追踪传播，tracestate/tracestate 用于携带业务上下文（如 agent_id），两者的组合覆盖了"追踪"和"业务"两个需求
2. **LLM API 的上下文模拟**——由于无法将 trace_id 传递给 LLM 提供商，在 Agent 框架层面模拟"虚拟 Span"，在 LLM 调用开始前创建子 Span，结束后记录耗时
3. **自动注入中间件**——框架层自动为 HTTP 客户端、gRPC 客户端、数据库驱动注入上下文传播，业务代码零感知
4. **上下文兜底机制**——当上下文传播失败（header 丢失、格式错误）时，接收方自动生成新的 trace_id，并通过日志标记"上下文断裂"
5. **跨传播协议的桥接**——HTTP → 消息队列 → gRPC 的跨协议传播自动转换上下文格式

## 实现的最大挑战

1. **异步代码中的上下文传递**——Python asyncio / JavaScript Promise 中，上下文不会自动在回调间传递，需要使用 ContextVar / AsyncLocalStorage 手动管理
2. **LLM API 调用的追踪盲区**——当 Agent 调用 OpenAI / Anthropic API 时，LLM 服务端的处理过程（排队、推理、生成）对客户端完全不可见，无法追踪
3. **Baggage 大小控制**——tracestate 的大小有限制（通常 < 512 字节），但 Agent 可能需要传播大量业务属性（task_type, agent_role, session_id）
4. **上下文劫持风险**——如果用户传入的请求中包含 traceparent header，恶意的 trace_id 可能污染追踪系统

## 能力边界

**能做什么：**
- 跨 HTTP/gRPC/消息队列 保持追踪链路的连续性
- 通过 Baggage 传播业务属性（agent_id, session_id）
- 实现"从用户请求到工具调用的全链路追踪"
- 跨进程/机器的 Span 自动关联

**不能做什么：**
- 不能穿透 LLM 提供商——无法追踪到 LLM 服务端内部的处理过程
- 不能保证 100% 传播——消息队列、流处理等异步场景可能有上下文丢失
- 不能自动修复不兼容的系统——老系统不支持 Trace Context 传播，需要适配层

## 与其他的区别

| 对比项 | W3C Trace Context | OpenTelemetry Baggage |
|--------|------------------|----------------------|
| 用途 | 追踪传播 | 业务属性传播 |
| 内容 | trace_id, span_id, trace_flags | 键值对 |
| 大小限制 | 严格（traceparent 固定 55 字节） | 宽松（但受 Header 大小限制） |
| 传播范围 | 全局 | 全局 |
| 何时用 | 所有请求 | 需要传递业务上下文时 |

## 最终工程优化

1. **标准 W3C 实现**——使用 opentelemetry-propagator 的标准 W3C Trace Context 实现，避免自定义传播格式
2. **Baggage 白名单**——只允许预定义的 baggage key 传播，防止敏感信息（如用户 ID）通过追踪系统泄露
3. **混合传播策略**——对 LLM API 使用"虚拟 Span + 近似时间戳"，对工具 API 使用"标准 W3C 传播"，两者在可视化层无缝拼接
4. **上下文恢复**——Agent 崩溃重启后，通过持久化的上下文信息恢复 Trace（对于长时间运行的 Agent 尤其重要）
5. **异步上下文自动传播**——使用 Python 的 `contextvars` 或 Node.js 的 `AsyncLocalStorage`，确保 asyncio 中的上下文自动传递，无需在每个协程中手动传递
