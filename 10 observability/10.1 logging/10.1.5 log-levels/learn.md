# 10.1.5 Log Levels — 日志级别设计

## 简单介绍

日志级别（Log Levels）为日志系统提供了**严重性分级**机制，使开发者可以根据场景过滤和定位信息。在 Agent 系统中，合理的日志级别设计尤为重要——Agent 的每一步（Thought、Action、Observation）都可能产生大量日志，如果没有清晰地分级，调试就像在"噪音中找信号"。

## 基本原理

标准的日志级别金字塔：

```
                    ┌──────┐
                    │FATAL│  ← 系统崩溃，必须立即处理
                   │ERROR│    ← 功能不可用，需要调查
                  │WARN │     ← 异常但可恢复，值得关注
                 │ INFO │     ← 关键事件：调用了什么、结果如何
                │DEBUG │      ← 详细信息：参数值、中间状态
               │ TRACE │      ← 最细粒度：执行路径、变量值
```

每个级别在上层级别的基础上增加了详细程度：

```python
# FATAL —— Agent 无法继续运行
logger.fatal("agent_crash", reason="LLM provider returned 503 for 5 retries")

# ERROR —— 某次执行失败但系统仍在运行
logger.error("tool_execution_failed", tool="send_email", error="auth_expired")

# WARN —— 异常但被自动恢复机制处理
logger.warn("rate_limit_approaching", used=95, limit=100)

# INFO —— 正常流程中的关键事件
logger.info("tool_call", tool="search_web", query="latest AI news")

# DEBUG —— 开发调试用详细信息
logger.debug("thought_content", raw_llm_response="...")

# TRACE —— 函数级执行跟踪
logger.trace("memory_retrieve", embedding_time_ms=12, candidates=50)
```

## 背景

日志级别的概念来自传统日志框架（log4j, Python logging），其最初设计是为了在开发环境和生产环境之间快速切换日志详细程度。但在 Agent 系统中，级别体系面临新的挑战：

1. Agent 的"一个请求"包含多个步骤（Thought → Action → Observation 循环），每个步骤都有不同的日志粒度需求
2. Agent 系统的错误来源比传统系统更多样——LLM 错误、工具错误、规划错误、记忆错误
3. Agent 的 Token 消耗本身就很高，冗余的日志输出会影响性能和成本

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 只有 INFO 和 ERROR | 所有正常日志都 INFO，异常都 ERROR | INFO 太多被忽略，ERROR 太少失去定位价值 |
| 全程 DEBUG | 开发和测试都用 DEBUG 级别 | 信息过载，关键信息被淹没在大量细节中 |
| 按模块固定级别 | 不同模块配置不同默认级别 | 配置僵化，跨模块调试时需要在多个文件间切换 |
| 无 TRACE 级别 | 只有 DEBUG 作为最细粒度 | 无法区分"不太重要的细节"和"非常重要的细节" |

## 核心矛盾

| 矛盾维度 | 级别太少 | 级别太多 |
|---------|---------|---------|
| 理解成本 | 低 | 高（需要记 6+ 级别含义） |
| 过滤精度 | 粗（要么全要、要么全不要） | 细（可以精确控制） |
| 误判成本 | 关键信息被淹没 | 级别用错导致信息错位 |

在 Agent 系统中，一个实际的问题是：**Agent 的 Thought 应该是什么级别？** INFO 太粗（每步 Thought 都 INFO，数量太多），DEBUG 又太细（默认不输出，难以监控）。

**推荐方案**：将 Agent 的 Thought 分为两级——"行动决策"的 Thought 用 INFO（"用户想要...，所以我需要调用 X"），"纯内省"的 Thought 用 DEBUG（"让我重新审视上一步的结果，确认是否理解正确"）。

## 当前主流优化方向

1. **动态级别切换**——通过 API 或配置文件动态调整日志级别，无需重启 Agent。特别是在线调试场景：发现 Agent 行为异常 → 动态切换到 DEBUG → 完成后恢复
2. **请求级级别覆盖**——对特定的 Agent 请求临时提高日志级别（如传入 `debug=true` 参数），不影响其他请求
3. **自适应日志级别**——系统根据当前状态自动调整级别：正常时 INFO，检测到异常趋势时自动降级到 DEBUG
4. **结构化级别**——不仅标记"有多严重"，还标记"什么类型"（`INFO[TOOL]`, `INFO[THOUGHT]`, `WARN[LATENCY]`），实现多维度过滤
5. **级别与采样联动**——DEBUG 日志默认 1:100 采样，但当 ERROR 发生时自动提升采样率为 1:1，确保错误前后的完整上下文

## 实现的最大挑战

1. **级别的语义一致性**——不同开发者和模块对级别的理解可能不一致（有人觉得"工具超时"是 WARN，有人觉得是 ERROR），需要建立团队级共识
2. **级别传播与覆盖**——在多层架构中（Gateway → Agent → Tool → API），外层日志级别的切换需要传播到内层模块
3. **Agent 的"级别漂移"**——随着 Agent 任务的推进，同一条日志的级别可能变化（初始 INFO，发现上下文后升级为 WARN）

## 能力边界

**能做什么：**
- 快速过滤出关注的日志类型（只看 ERROR，或只看 TOOL 相关）
- 在开发和运维之间建立一致的沟通语言（"看这个 INFO 日志"）
- 降低日志存储成本（低级别日志可以采样或短保留）

**不能做什么：**
- 不能替代合理的告警规则——日志级别是分类手段，不是检测手段
- 不能解决日志量过大的根本问题——级别只是过滤，总输出量取决于代码逻辑
- 不能保证级别的正确使用——需要 code review 和工具（linter）来强制执行

## 与其他的区别

Agent 的日志级别与通用日志级别相比，多了 Agent 特有的语义维度：

```python
# 通用日志
logger.info("user_login", user_id="123")

# Agent 专有日志级别拓展
logger.thought(level="INFO", content="用户需要查天气，我需要调取位置信息")
logger.action(level="INFO", tool="get_weather", params={...})
logger.observation(level="DEBUG", result="API returned 200")  # 结果内容在 DEBUG
logger.observation(level="INFO", result_summary="天气数据获取成功")  # 结果状态在 INFO
```

很多 Agent 框架（LangChain、CrewAI）在标准级别基础上增加了 **THOUGHT** 和 **ACTION** 特定的日志方法，将"级别"和"类型"两个维度解耦。

## 最终工程优化

1. **级别层次结构标准化**——采用 Python `logging` 的 `CRITICAL`/`ERROR`/`WARNING`/`INFO`/`DEBUG` 五级制，不推荐引入自定义级别（会导致团队间理解不一致）
2. **类型-级别矩阵**——将日志的"级别"和"类型"作为独立维度，支持 `level=INFO, type=TOOL` 和 `level=DEBUG, type=TOOL` 的精细过滤
3. **级别基础的采样和保留策略**——ERROR 日志保留 90 天、WARN 保留 30 天、INFO 保留 7 天、DEBUG 保留 2 天，平衡存储和可追溯性
4. **启动调试模式**——通过环境变量 `AGENT_DEBUG=true` 或请求参数 `?debug=1` 触发全链路的 DEBUG 级别，便于开发阶段和生产环境按需排错
