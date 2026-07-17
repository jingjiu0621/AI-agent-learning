# 10.2.5 Custom Tracer — 自定义追踪器实现

## 简单介绍

Custom Tracer（自定义追踪器）是在不依赖 LangSmith/OTel 等外部框架的情况下，自行实现 Agent 追踪能力的工程实践。适合的场景包括：轻量级 Agent 不希望引入外部依赖、需要完全控制追踪数据的格式和存储、在受限网络环境中运行无法连接外部追踪服务。

## 基本原理

一个最小化的追踪器只需要三个核心概念：Trace、Span 和 Context。

```python
import time
import uuid
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Span:
    trace_id: str
    span_id: str
    parent_id: Optional[str]
    name: str
    start_time: float          # time.time() 时间戳
    end_time: Optional[float] = None
    attributes: dict = field(default_factory=dict)
    status: str = "OK"         # OK | ERROR
    events: list = field(default_factory=list)

    def duration_ms(self) -> float:
        if self.end_time:
            return (self.end_time - self.start_time) * 1000
        return 0.0

@dataclass
class Trace:
    trace_id: str
    root_span: Span
    spans: list = field(default_factory=list)

class SimpleTracer:
    """最小化追踪器实现"""

    def __init__(self, agent_id: str):
        self.agent_id = agent_id
        self._spans: dict[str, Span] = {}
        self._stack: list[str] = []  # span_id 栈

    def start_span(self, name: str, attributes: dict = None) -> Span:
        trace_id = self._get_or_create_trace_id()
        span = Span(
            trace_id=trace_id,
            span_id=uuid.uuid4().hex[:12],
            parent_id=self._stack[-1] if self._stack else None,
            name=name,
            start_time=time.time(),
            attributes=attributes or {},
        )
        self._spans[span.span_id] = span
        self._stack.append(span.span_id)
        return span

    def end_span(self, span: Span, status: str = "OK"):
        span.end_time = time.time()
        span.status = status
        if self._stack and self._stack[-1] == span.span_id:
            self._stack.pop()

    def export(self) -> list[dict]:
        """导出为可序列化的字典列表"""
        return [
            {
                "trace_id": s.trace_id,
                "span_id": s.span_id,
                "parent_id": s.parent_id,
                "name": s.name,
                "duration_ms": s.duration_ms(),
                "status": s.status,
                "attributes": s.attributes,
            }
            for s in self._spans.values()
        ]

    def _get_or_create_trace_id(self) -> str:
        if not self._stack:
            return uuid.uuid4().hex[:16]
        return self._spans[self._stack[0]].trace_id
```

在 Agent 中的使用：

```python
class Agent:
    def __init__(self):
        self.tracer = SimpleTracer(agent_id="weather-agent")

    async def run(self, query: str):
        root = self.tracer.start_span("agent_run", {"query": query})

        thought_span = self.tracer.start_span("thought", {"model": "claude"})
        # ... LLM 调用
        self.tracer.end_span(thought_span)

        tool_span = self.tracer.start_span("tool_call", {"tool": "get_weather"})
        # ... 工具调用
        self.tracer.end_span(tool_span)

        self.tracer.end_span(root)

        # 导出的追踪数据可写入本地文件或发送到后端
        return self.tracer.export()
```

## 背景

大多数生产级的 Agent 最终都会走向自建追踪器而不是完全依赖外部服务。原因包括：
1. **数据主权**——敏感数据不应离开自己的网络
2. **定制需求**——Agent 的追踪数据结构很特殊，现有方案不能 100% 满足
3. **成本控制**——SaaS 追踪服务的费用在 Agent 高吞吐场景下可能很高
4. **灵活性**——Agent 框架迭代快，追踪方案需要能够快速适配

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 不追踪 | 只依赖日志 | 无法从整体视角分析 Agent 性能 |
| 用装饰器手写 | 用 @timing 装饰器记录函数耗时 | 只能记录耗时，无法关联调用链路 |
| 直接复制 OTel SDK | 自己实现了一个简化版 OTel | 实现复杂度适中，但与 OTel 不兼容 |
| 使用第三方简单方案 | aiologger、structlog 等 | 功能有限，不是真正的追踪系统 |

## 核心矛盾

| 矛盾维度 | 手写最小追踪 | OTel | LangSmith |
|---------|------------|------|-----------|
| 实现成本 | 低（几百行） | 中 | 无 |
| 定制性 | 完全可控 | 高（但需学 API） | 低 |
| 生态兼容 | 无 | 好 | LangChain 好 |
| 维护成本 | 中（需自己迭代） | 低（社区维护） | 低 |
| 功能完整性 | 基础 | 丰富 | 丰富 |

**核心矛盾**：自建追踪器在前期的开发成本低、灵活性高，但随着 Agent 系统的发展，需要不断"补课"——先是补采样策略、然后是上下文传播、最后是兼容标准协议。**建议采用"渐进式"策略**：从最小自建开始，逐步向 OTel 标准靠拢，最终在 Agent 框架层使用 OTel API 但通过自建的 Exporter 控制数据流向。

## 当前主流优化方向

1. **装饰器驱动的自动追踪**——利用 Python 装饰器或元编程，自动为 Agent 的关键方法（`step()`、`call_llm()`、`call_tool()`）添加 Span，减少手动埋点
2. **async-friendly 追踪上下文**——使用 `contextvars` 实现异步安全的追踪上下文传递，避免在 `asyncio.gather` 并发调用中上下文串线
3. **环形缓冲 + 异步导出**——Span 数据写入内存中的环形缓冲（RingBuffer），后台线程异步写入磁盘或网络，避免追踪 IO 阻塞 Agent 执行
4. **内存追踪 + 按需导出**——所有 Span 默认保留在内存中，只在发生错误或显式调用 export 时写入持久化存储，适合调试场景
5. **与结构化日志对齐**——Span 格式与结构化日志格式统一（共用 trace_id、相同的字段命名），使得日志和追踪可以无缝关联

## 实现的最大挑战

1. **并发安全性**——当 Agent 使用 asyncio.gather 并发调用多个工具时，多个 Span 同时在多个协程中操作，追踪器需要保证 ContextVar 不在协程间交叉
2. **内存泄漏风险**——如果 Agent 运行时间很长（数小时），内存中积累的 Span 可能导致 OOM，需要正确实现 Span 的过期和清理
3. **Span 树序列化**——从扁平的 Span 列表中重建树结构需要递归处理 parent_id，序列化/反序列化时必须保持引用关系
4. **Trace 完整性保证**——Agent 进程崩溃时，未 export 的 Span 全部丢失，需要实现周期性 flush 机制

## 能力边界

**能做什么：**
- 在不需要外部依赖的情况下获得 Agent 的完整调用链路
- 精确定位延迟瓶颈（哪一步、哪个工具耗时最长）
- 灵活控制追踪数据的格式和存储位置
- 在受限环境（内网、离线）中正常运行

**不能做什么：**
- 不能与外部追踪生态集成（除非兼容 OTel 协议）
- 不能开箱即用地提供可视化界面——需要自己搭
- 不能自动插装第三方库——只能追踪 Agent 框架自身的方法
- 不能实现复杂的采样策略——需要自建采样逻辑

## 与其他的区别

| 对比项 | 自建追踪器 | OTel 集成 |
|--------|----------|----------|
| 代码量 | ~300 行核心 + ~500 行工具 | ~30 行（初始化）+ 自动插装 |
| 学习曲线 | 低 | 中（需理解 OTel 概念） |
| 调试便利性 | 高（完全知道内部逻辑） | 中（黑盒 SDK） |
| 迁移成本 | 低（可以逐步替换为 OTel） | 低（标准化） |

## 最终工程优化

1. **跨度数据压缩**——Span 数据在内存中用 Protobuf 或 MessagePack 序列化，减少存储占用 60-80%
2. **分级的 Span 保留策略**——完整 Span 保留在内存 RingBuffer（最近 1000 个），汇总指标保留更长时间（最近 10000 个）——类似 Linux 内核的 trace 机制
3. **ContextVar 安全的 Span 上下文**——为每个协程维护独立的 Span 栈，使用 `contextvars.ContextVar[Stack]` 而非全局变量，确保并发安全
4. **渐进式兼容 OTel**——自建追踪器实现 OTel 的 SpanData 接口，当后期需要切换到标准 OTel 时，只需更换底层实现而不改业务代码
5. **自动 Span 修饰器**——实现 `@trace` 修饰器，自动记录函数名称、参数（可配置脱敏）、返回值和执行时间，减少重复代码
