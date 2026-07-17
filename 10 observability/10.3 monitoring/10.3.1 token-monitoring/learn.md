# 10.3.1 Token Monitoring — Token 消耗监控

## 简单介绍

Token 监控是 Agent 可观测性中**直接与成本挂钩**的维度。在 Agent 系统中，Token 就是钱——每次 LLM 调用都在消耗 prompt tokens 和 completion tokens，这些直接对应 OpenAI/Anthropic 的 API 费用。没有 Token 监控，团队无法回答最基本的运营问题："Agent 一个月花了多少钱？"

## 基本原理

Token 监控的核心是三个数字：**输入 Token**（Prompt）、**输出 Token**（Completion）、**单价**（$/1M tokens）。Agent 的 Token 消耗不仅来自最终的 LLM 调用，还来自系统 Prompt、历史消息、工具定义等"隐藏消耗"。

```python
class TokenMonitor:
    """Token 消耗监控器"""

    def __init__(self):
        self.metrics = {
            "prompt_tokens": Counter(),
            "completion_tokens": Counter(),
            "total_tokens": Counter(),
            "estimated_cost": Counter(),
        }
        # 模型定价（$/1M tokens）
        self.PRICING = {
            "claude-sonnet-5": {"input": 3.0, "output": 15.0},
            "claude-haiku-4.5": {"input": 0.25, "output": 1.25},
            "gpt-4o": {"input": 2.50, "output": 10.0},
            "gpt-4o-mini": {"input": 0.15, "output": 0.60},
        }

    def record_llm_call(self, model: str, prompt_tokens: int,
                        completion_tokens: int, agent_id: str):
        pricing = self.PRICING.get(model, {"input": 0, "output": 0})
        cost = (prompt_tokens * pricing["input"] +
                completion_tokens * pricing["output"]) / 1_000_000

        self.metrics["prompt_tokens"].inc(prompt_tokens)
        self.metrics["completion_tokens"].inc(completion_tokens)
        self.metrics["total_tokens"].inc(prompt_tokens + completion_tokens)
        self.metrics["estimated_cost"].inc(cost)

        # Prometheus 指标
        TOKEN_CONSUMPTION.labels(
            model=model,
            agent_id=agent_id,
            type="prompt"
        ).inc(prompt_tokens)

        COST_USD.labels(model=model, agent_id=agent_id).inc(cost)
```

Agent Token 消耗的构成分析：

```
一个 5 步 Agent 任务的 Token 消耗分解（示例）：
┌─────────────────────────────────────────┬──────────┬──────────┐
│  组件                                    │ 输入 Token │ 输出 Token │
├─────────────────────────────────────────┼──────────┼──────────┤
│  系统 Prompt（Agent 身份 + 规则）         │  2,500   │    -     │
│  工具定义（JSON Schema 描述）             │  1,200   │    -     │
│  历史消息（前 4 轮对话）                  │  4,500   │    -     │
│  Step 1 Thought + Action                 │    -     │   450    │
│  Step 2 Thought + Action                 │    -     │   380    │
│  Step 3 Thought + Action                 │    -     │   520    │
│  Step 4 Thought + Action                 │    -     │   310    │
│  Step 5 Final Response                   │    -     │   200    │
├─────────────────────────────────────────┼──────────┼──────────┤
│  总计                                    │  8,200   │  1,860   │
│  成本 (Sonnet 5): 8.2K×$3 + 1.86K×$15   = $0.0525              │
└─────────────────────────────────────────┴──────────┴──────────┘
```

## 背景

在传统监控中，"成本"通常指服务器费用。但在 Agent 系统中，LLM API 费用往往超过服务器费用，成为最大的运营成本。而且 Agent 的 Token 消耗有独特的特性：

1. **不可预测性**——Agent 的 Token 消耗与任务难度、Agent "思考" 的复杂程度相关，无法像传统 API 那样按请求数预测
2. **隐藏消耗**——系统 Prompt、工具 Schema、历史消息等"非回答"部分可能占总消耗的 60% 以上
3. **多模型混合**——Agent 可能使用不同模型（如规划用 Opus、执行用 Haiku），每个模型的定价不同

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 不监控 Token | 只监控 API 调用次数 | 无法估算成本，"月底收到账单才知道花了多少钱" |
| 粗略估算 | 每次输出 token 数 × 固定价格 | 忽略了输入 token 和系统 Prompt 的消耗，估算偏差 50%+ |
| LLM 返回后手动记录 | 在代码中手动调用 token counting | 容易遗漏，忽略了工具的 token 定义消耗 |
| 仅监控总消费 | 只看总 token 数 | 无法按 Agent/模型/用户拆分，不知道钱花在了哪里 |

## 核心矛盾

| 矛盾维度 | 不监控 | 按总 Token 监控 | 精细拆分监控 |
|---------|-------|---------------|-------------|
| 实施成本 | 0 | 低 | 中 |
| 成本可见性 | 无 | 总成本 | 每 Agent/每用户/每任务成本 |
| 优化能力 | 无 | 可优化总消耗 | 可精确定位浪费 |
| 告警价值 | 无 | 可设总量告警 | 可设细粒度告警 |

**核心矛盾**：Token 监控的粒度与实施复杂度成正比。最简单的"记总数"无法回答"哪个 Agent 最费钱"，但"每个工具定义消耗的 token"这种极致粒度又导致实施成本过高。

## 当前主流优化方向

1. **实时 Token 计数器**——在 LLM API 调用层集成实时 Token 计数（使用 tiktoken 等库），每次调用后立即记录，支持近实时的成本 DashBoard
2. **多维度 Token 分析**——按 agent_id、model、task_type、user_id、session_id 等维度拆分 Token 消耗，支持 OLAP 分析
3. **消耗预测与预算告警**——基于历史消耗趋势预测本月 Token 消耗，当预测值超过预算阈值时提前告警
4. **Token 浪费检测**——识别"无用 Token"：Agent 重复调用、过长但未被使用的上下文、冗余的工具定义
5. **成本归因与分摊**——将 Token 消耗归因到具体的用户请求或业务线，实现成本的分摊和计费

## 实现的最大挑战

1. **精确计数的延迟**——Token 计数需要 tiktoken 等库的处理时间，在高吞吐场景下可能成为瓶颈
2. **多模型定价维护**——模型提供商的定价经常变化，用于成本估算的定价表需要定期更新
3. **系统 Prompt 的 Token 分摊**——共享的系统 Prompt（如 Agent 的 Role Definition）的 Token 消耗应该分摊到每次调用还是作为一个固定开销？
4. **缓存调用的 Token 计算**——如果使用了 Prompt Caching，实际消耗的 Token 少于返回的 Token 数，监控系统需要准确反映这种差异

## 能力边界

**能做什么：**
- 精确追踪 Agent 系统的 Token 消耗和成本
- 按 agent_id、model、task 等多维度拆分解读
- 设置预算告警，防止 Token 消耗失控
- 识别 Token 浪费模式，优化成本

**不能做什么：**
- 不能直接降低 Token 消耗——监控只暴露问题，优化需要其他手段（Prompt 压缩、缓存、模型路由）
- 不能精确预测未来消耗——Agent 的行为是动态的，预测仅供参考
- 不能追踪非 LLM 调用的成本——工具调用的 API 费用、基础设施费用不在 Token 监控范围内

## 最终工程优化

1. **分层成本模型**——将 Token 消耗分层：固定层（系统 Prompt、工具 Schema）按请求数均摊，动态层（历史消息、最终输出）按实际记录，准确计算"每个请求的真实成本"
2. **Prometheus Histogram 收集**——使用 Prometheus Histogram 收集 Token 消耗分布（0-1K、1K-4K、4K-8K、8K+ 等桶），支持高效的百分位查询
3. **按模型级别的成本分摊表**——维护自动更新的模型定价表（从提供商 API 获取），确保成本计算的准确性
4. **Token 消耗异常检测**——基于历史数据建立 Token 消耗基线，当单次请求的 Token 消耗超过基线 3σ 时自动告警
5. **实时 Cost-to-Completion 估计**——在 Agent 执行过程中实时估算"到目前为止成本"和"预计完成成本"，支持成本预算的实时控制
