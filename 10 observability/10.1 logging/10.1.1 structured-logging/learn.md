# 10.1.1 Structured Logging — 结构化日志

## 简单介绍

结构化日志（Structured Logging）是将日志记录从"写文本"升级为"写结构化数据"的工程实践。在 Agent 系统中，日志不再是程序员写给人看的文本消息，而是**机器可解析的事件流**——每个日志行都是一个包含预定义字段的 JSON 对象，可以被日志聚合系统自动收集、解析、索引和查询。

## 基本原理

结构化日志的核心思想是将日志分解为**固定字段**（时间戳、级别、模块、trace_id）和**动态字段**（事件特定的上下文数据），以键值对而非字符串拼接的方式记录。

```python
# ❌ 非结构化日志
import logging
logging.info(f"Agent {agent_id} called tool {tool_name} with params {params}")

# ✅ 结构化日志
import structlog
logger.info("tool_call",
    agent_id="agent-123",
    tool_name="get_weather",
    params={"city": "Beijing"},
    duration_ms=450,
    token_count=156,
    trace_id="trace-abc-123"
)
```

结构化日志的典型 JSON 输出：

```json
{
  "timestamp": "2026-07-17T10:30:00.123Z",
  "level": "INFO",
  "logger": "agent.loop",
  "event": "tool_call",
  "trace_id": "tr-7f3a1b2c",
  "session_id": "sess-001",
  "agent_id": "weather-agent-v2",
  "turn_number": 3,
  "data": {
    "tool": "get_weather",
    "arguments": {"city": "Beijing", "units": "metric"},
    "result_summary": "200 OK, temp=22",
    "duration_ms": 320
  }
}
```

## 背景

传统日志以 log4j 时代的 `printf` 风格为主流——开发者在代码中编写模板字符串，运行时填入变量。这种做法在单体应用中尚可接受，但在微服务和 Agent 系统中暴露出根本性问题：

1. **不可解析**——不同开发者写的日志格式不同，无法用统一的工具自动处理
2. **不可关联**——没有标准化的上下文标识符（trace_id），无法跨服务/跨模块关联同一请求的日志
3. **不可聚合**——纯文本日志无法被 Prometheus 等指标系统直接消费，需要额外的 parsing 层

ECF（Elastic Common Schema）和 OpenTelemetry 日志标准推动了结构化日志的规范化，定义了标准的字段命名和类型。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| print 调试 | 直接 print() 输出到控制台 | 无法检索、无法分级、生产环境不可用 |
| logging 模块 + 格式化字符串 | Python logging 的 %s 格式化 | 单机可用，但无法跨服务关联，无法自动解析 |
| 集中式日志收集（EFK） | Filebeat + Elasticsearch + Kibana | 能搜索了，但纯文本格式导致搜索效率低、结构化查询困难 |
| JSON 日志（早期） | 开发者手动拼接 JSON 字符串 | 容易出错、缺乏 schema 校验、字段命名不统一 |

## 核心矛盾

| 矛盾维度 | 非结构化日志 | 结构化日志 |
|---------|-------------|-----------|
| 可读性 | 人眼友好 | 需工具辅助 |
| 机器可解析性 | 差 | 好 |
| 查询能力 | grep 级别 | SQL / DSL 级别 |
| 指标提取 | 需正则解析 | 字段直接映射 |
| 跨系统关联 | 不可能 | trace_id/session_id 关联 |
| 写入性能 | 快（简单字符串） | 略慢（JSON 序列化） |

## 当前主流优化方向

1. **Schema 标准化**——采用 ECS（Elastic Common Schema）或 OTel 语义约定作为日志 schema，确保不同框架和工具的日志可以互相理解和关联
2. **零成本抽象**——使用 structlog、loguru 等库实现结构化日志，开发体验与普通 logging 几乎无异，但输出自动为 JSON
3. **日志与追踪融合**——日志与 Span 自动关联，日志行自动成为对应 Span 的属性，无需手动维护 trace_id
4. **异步批处理**——日志写入通过异步队列批量发送，避免 IO 阻塞 Agent 主循环
5. **Schema 即文档**——通过 JSON Schema 或 Protobuf 定义日志格式，自动生成文档和校验

## 能力边界

**能做什么：**
- 自动解析：所有字段对机器友好，日志聚合系统可以直接索引
- 跨系统关联：通过标准化的 trace_id 关联不同组件的日志
- 自动指标提取：从日志中直接提取延迟、错误数、Token 消耗等指标
- 精确查询：按 agent_id 查询、按 tool_name 过滤、按 duration_ms 排序

**不能做什么：**
- 不能解决"日志太多"的问题——结构化日志本身不含采样策略
- 不能自动分析日志内容——结构化格式方便了存储和检索，但日志的含义仍需人理解
- 不能保证日志的完整性——如果 Agent 进程崩溃，内存中的日志缓冲区可能丢失

## 与其他的区别

| 对比项 | 标准 logging | structlog | loguru |
|--------|-------------|-----------|--------|
| 自动 JSON 输出 | 需手动配置 Formatter | 默认支持 | 需扩展 |
| 上下文绑定 | 无 | Logger.bind() | logger.bind() |
| 性能 | 高 | 中（序列化开销） | 中 |
| 链式调用 | 不支持 | with logger.bind(): | 支持 |

工程推荐：对于 Agent 系统，优先使用 `structlog`——它在 Python 生态中提供了最好的结构化和上下文绑定能力，且与 standard logging 兼容可以逐步迁移。对于高性能场景（如 Gateway 层），考虑 Rust 的 `tracing` crate 或 Go 的 `zap`。

## 最终工程优化

1. **预热 JSON 序列化器**——在 Agent 启动时预分配 JSON 编码器，避免重复的对象创建
2. **批量写入 + 异步刷新**——日志先写入内存 RingBuffer，积累到阈值或时间间隔后批量写入磁盘/网络
3. **局部采样**——对高频日志（如每步 Token 统计）使用 1:N 采样，对低频关键日志（如错误）使用完整记录
4. **日志压缩**——长时间归档使用 gzip 压缩，存储压缩比通常能达到 10:1
5. **Hot Reload 日志级别**——通过配置文件或 API 动态调整各级别日志的详细程度，无需重启
