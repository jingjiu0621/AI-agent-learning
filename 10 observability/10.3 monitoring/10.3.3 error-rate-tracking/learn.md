# 10.3.3 Error Rate Tracking — 错误率追踪

## 简单介绍

错误率追踪（Error Rate Tracking）监控 Agent 系统中各类错误的频率和趋势。Agent 系统的错误来源比传统系统更加多样——不仅有 LLM API 错误、工具调用错误，还有 Agent 特有的"逻辑错误"：规划错了、忽略了关键信息、陷入了循环。

## 基本原理

Agent 错误的分层分类体系：

```python
class AgentErrorType:
    """Agent 错误分类体系"""

    # Layer 1: LLM 错误
    LLM_TIMEOUT = "llm.timeout"           # LLM API 超时
    LLM_RATE_LIMIT = "llm.rate_limit"     # 限流
    LLM_INVALID_RESPONSE = "llm.invalid"  # 响应格式错误
    LLM_CONTEXT_LENGTH = "llm.context_exceeded"  # 上下文超长

    # Layer 2: 工具错误
    TOOL_TIMEOUT = "tool.timeout"         # 工具调用超时
    TOOL_AUTH_ERROR = "tool.auth_error"   # 认证失败
    TOOL_INVALID_PARAMS = "tool.invalid_params"  # 参数错误
    TOOL_SERVICE_DOWN = "tool.service_down"     # 服务不可用

    # Layer 3: Agent 逻辑错误
    AGENT_PLANNING_ERROR = "agent.planning_error"    # 规划错误
    AGENT_LOOP_DETECTED = "agent.loop_detected"      # 死循环检测
    AGENT_CONTEXT_EXCEEDED = "agent.context_exceeded" # Agent 上下文超长
    AGENT_INVALID_ACTION = "agent.invalid_action"    # 无效行动

    # Layer 4: 系统错误
    SYSTEM_MEMORY = "system.memory_exhausted"   # 内存耗尽
    SYSTEM_CRASH = "system.crash"                # 进程崩溃
```

```python
# 错误率聚合
class ErrorRateTracker:
    def __init__(self):
        self.counters = defaultdict(Counter)
        self.window_sizes = {"1m": 60, "5m": 300, "1h": 3600}

    def record_error(self, error_type: str, agent_id: str):
        now = time.time()
        self.counters[error_type][agent_id] += 1

    def get_error_rate(self, error_type: str, window: str = "5m") -> float:
        """返回每分钟错误数"""
        total = sum(self.counters[error_type].values())
        window_sec = self.window_sizes[window]
        return total / (window_sec / 60)
```

## 背景

Agent 系统的错误以链式反应为特征：一个工具调用出错 → Agent 下一轮的 Thought 基于错误数据 → 后续所有决策都错了 → 最终输出错误结果。这与传统系统不同——传统系统中一个请求失败了通常独立的，不会影响后续请求。

这使得 Agent 的错误率追踪比传统系统更关键：发现"错误率上升"时不仅是一个指标，而可能意味着"Agent 开始批量输出错误结果"。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 只追踪 HTTP 状态码 | 只记录 4xx/5xx 错误 | 完全漏掉 Agent 的逻辑错误（HTTP 200 但回答错误） |
| 只追踪 LLM API 错误 | 记录 LLM 调用的超时和限流 | 忽略了工具调用错误和 Agent 逻辑错误 |
| 统一 ERROR 日志 | 所有错误都记录为 ERROR 级别 | 无法区分错误类型，无法分类分析 |
| 人工查看日志找错误 | 不定期查看日志了解错误情况 | 滞后严重，无法及时发现错误率上升趋势 |

## 错误率计算

Agent 系统的几个关键错误率指标：

```python
# 1. LLM 错误率 = LLM 错误次数 / LLM 调用次数
llm_error_rate = llm_errors / llm_total_calls * 100

# 2. 工具错误率 = 工具错误次数 / 工具调用次数
tool_error_rate = tool_errors / tool_total_calls * 100

# 3. 任务失败率 = 失败任务数 / 总任务数
#     失败：Agent 超时、Agent 明确表示无法完成、输出无效
task_failure_rate = failed_tasks / total_tasks * 100

# 4. 用户反馈错误率 = 用户点踩/举报次数 / 总交互次数
user_feedback_error_rate = negative_feedback / total_interactions * 100
```

## 当前主流优化方向

1. **错误分类自动化**——使用 LLM 自动对 Agent 的错误进行分类和归因（LLM 的错 vs 工具的错 vs Agent 的规划错），减少人工分类的工作量
2. **错误传播链追踪**——自动识别"错误连锁反应"：A 工具返回错误 → Agent 基于错误数据决策 → 之后的步骤都错了。在监控中将这种链式错误标记为一个错误事件
3. **异常检测 vs 阈值告警**——使用统计异常检测（3-sigma、移动平均）而非固定阈值来检测错误率异常，减少误报
4. **归因分析**——自动关联错误率上升与版本变更（Agent Prompt 更新、工具版本升级、模型切换），快速定位根因
5. **错误严重性分级**——不仅记录错误类型，还量化错误的影响：A 类（用户可见的错误）、B 类（可自动恢复的错误）、C 类（内部错误用户无感知）

## 实现的最大挑战

1. **逻辑错误的判定**——Agent 成功调用了工具、输出了格式正确的回复，但内容完全是错的。这种"逻辑错误"无法通过自动化指标判断，需要 LLM-as-Judge 或用户反馈
2. **错误归因的模糊性**——Agent 输出错误可能由多个因素叠加导致（Prompt 不清晰 + 工具返回奇怪数据 + 模型能力不足），难以精确定因
3. **错误率的季节性模式**——Agent 在非高峰期的错误率模式可能与高峰期完全不同，固定阈值难以适应
4. **"成功" vs "正确" 的区别**——Agent 成功完成了一次工具调用（HTTP 200），但返回的数据是错误的或不完整的。监控系统可能误判为成功

## 能力边界

**能做什么：**
- 分类和量化 Agent 系统的各类错误
- 监控错误率趋势，自动检测异常上升
- 识别错误的主要来源（是 LLM 不稳定还是工具不稳定）
- 为错误预算管理提供数据

**不能做什么：**
- 不能自动修复错误——Error Tracking 是诊断手段，修复需要 retry/fallback/human-escalation
- 不能判断所有"错误"——逻辑错误、幻觉等需要语义评估（Module 11）而非简单计数
- 不能完全避免误报——错误率上升可能是流量增加（分子分母同时增加）而不是真正的问题

## 最终工程优化

1. **错误口径统一**——所有 Agent 错误使用统一的 Error 对象格式（type、detail、severity、timestamp、trace_id），便于统一分析和告警
2. **错误率基线的自动更新**——基于过去 30 天的数据自动计算"正常错误率基线"，按小时粒度更新，适应 Agent 使用的周期性变化
3. **错误溯源标签**——在 Prometheus 指标的 label 中携带 agent_id、tool_name、llm_model 等维度，支持快速的"下钻分析"（总错误率上升 → 哪个 Agent 升了 → 哪个工具导致的）
4. **错误事件关联**——当用户主动反馈"回答不对"时，自动关联该会话的所有错误指标（该 session 中的 LLM 调用是否都有异常、工具调用是否有失败），形成完整的根因分析
5. **错误模式知识库**——将识别出的错误模式持续积累到知识库中，Agent 在推理时自动检索知识库中的错误模式，主动避免已知陷阱
