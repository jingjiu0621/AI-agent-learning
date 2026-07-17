# 10.1 Logging — Agent 结构化日志

日志是 Agent 可观测性的第一道防线。与追踪（tracing）关注"调用链路"、监控（monitoring）关注"聚合指标"不同，日志关注的是 **"当时发生了什么"**——以事件为单位，按时间序完整记录 Agent 的思考（Thought）、行动（Action）和观察（Observation）。

## 什么是 Agent 日志？

传统日志记录的是代码执行路径：哪个函数被调用、参数是什么、返回值是什么。Agent 日志在此基础上增加了 AI 特有的维度：

```
传统日志（REST API）:
  [INFO] GET /api/users/123 → 200 OK (45ms)

Agent 日志:
  [THOUGHT] 用户想查天气，我需要先获取用户位置
  [ACTION] 调用 get_location(user_id=123) → {city: "Beijing"}
  [THOUGHT] 好，现在用北京查天气
  [ACTION] 调用 get_weather(city="Beijing") → {temp: 22, condition: "sunny"}
  [OBSERVATION] 天气 API 返回正常，温度 22°C
  [INFO] Agent 回复："北京目前晴，22°C"
```

## 日志 vs 追踪 vs 监控

| 维度 | 日志 (Logging) | 追踪 (Tracing) | 监控 (Monitoring) |
|------|---------------|----------------|-------------------|
| 粒度 | 事件级 | 请求/调用级 | 聚合级 |
| 数据结构 | 非结构化/半结构化 | Span ID + Parent ID | 时序指标 |
| 查询方式 | 全文搜索 + 过滤 | 按 Trace ID 关联 | 按时间窗口聚合 |
| 主要用途 | 调试、审计、分析 | 链路分析、延迟分解 | 告警、容量规划 |
| 存储量 | 最大 | 中等 | 最小 |
| 保留时间 | 短（天~周） | 中（周~月） | 长（月~年） |

## 核心设计原则

1. **结构化 > 非结构化**——JSON 格式的日志可以被程序自动解析、聚合、告警，纯文本日志只能靠人眼搜索
2. **每个日志行都要可独立理解**——包含足够的上下文（trace_id, session_id, timestamp），使单行日志不依赖上下文也能定位问题
3. **异步写入，不阻塞主流程**——日志 IO 不应增加 Agent 的响应延迟
4. **敏感信息过滤是必须的**——Agent 的思考过程可能包含用户隐私或业务敏感数据，日志系统需要内置脱敏机制
5. **日志级别动态可调**——生产环境默认 INFO，调试时动态切换到 DEBUG，不需要重新部署

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|---------|------|
| 10.1.1 structured-logging | 如何设计结构化日志格式？ | ★★★☆☆ |
| 10.1.2 thought-logging | 如何记录 Agent 的思考过程？ | ★★★☆☆ |
| 10.1.3 action-logging | 如何记录工具调用的完整信息？ | ★★★☆☆ |
| 10.1.4 observation-logging | 如何记录观察结果和异常？ | ★★★☆☆ |
| 10.1.5 log-levels | 如何设计合理的日志级别体系？ | ★★☆☆☆ |
| 10.1.6 log-aggregation | 如何构建日志聚合管道？ | ★★★☆☆ |

## 向下关联

- → 10.2 tracing：日志中的 trace_id/session_id 是关联到追踪系统的桥梁
- → 10.3 monitoring：日志中的聚合数据（Token 数、延迟、错误率）是监控指标的输入源
- → 10.4 debugging：详细的日志是调试的基础，没有日志的调试相当于蒙眼修机器
