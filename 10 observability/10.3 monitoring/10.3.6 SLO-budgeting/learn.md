# 10.3.6 SLO Budgeting — SLO 定义与错误预算管理

## 简单介绍

SLO（Service Level Objective，服务级别目标）是 Agent 系统服务质量的量化承诺。错误预算（Error Budget）是 SLO 的反面——允许出错的额度。SLO 和错误预算构成了 **"以数据驱动的运维决策"** 核心：当错误预算充足时可以放心发布新功能，当错误预算不足时必须停止发布、聚焦稳定性。

## 基本原理

SLO 的数学定义：

```
SLO = 1 - (错误时间 / 总时间)

示例：Agent 系统的月度 SLO 定义
  目标：99.9% 的请求在 15 秒内完成
  错误：端到端延迟 > 15 秒的请求
  月度总请求：100,000
  允许错误：100 个请求（99.9% 的 0.1%）
  错误预算：100 次 / 月
```

Agent 系统的关键 SLO 指标：

```python
class AgentSLO:
    """Agent 系统 SLO 定义"""

    # 可用性 SLO
    availability = SLO(
        name="agent_availability",
        target=0.999,           # 99.9% 可用
        window="30d",           # 月度评估
        measure="llm_api_error_rate < 1% AND tool_error_rate < 2%",
    )

    # 延迟 SLO
    latency = SLO(
        name="agent_latency",
        target=0.95,            # 95% 请求在目标时间内
        threshold_ms=15000,     # 15 秒
        window="7d",
        measure="end_to_end_latency_p95",
    )

    # 质量 SLO（需评估系统配合）
    quality = SLO(
        name="agent_response_quality",
        target=0.98,
        window="7d",
        measure="llm_judge_score > 0.8",  # LLM-as-Judge 评分 > 0.8
    )
```

错误预算的计算和使用：

```python
class ErrorBudget:
    def __init__(self, slo: SLO, period_seconds: int = 30 * 24 * 3600):
        self.total_budget = 1 - slo.target  # 如 99.9% → 0.001 = 0.1%
        self.consumed = 0
        self.period_start = time.time()

    def consume(self, is_error: bool):
        """每次请求后调用，标记是否消耗了错误预算"""
        if is_error:
            self.consumed += 1

    @property
    def remaining(self) -> float:
        """剩余错误预算（百分比）"""
        return max(0, self.total_budget - self.consumed / self.total_requests)

    @property
    def should_deploy(self) -> bool:
        """是否可以发布新版本？"""
        # 错误预算剩余 > 50% 才能发布
        return self.remaining > 0.5
```

## 背景

SLO 和错误预算的概念来自 Google SRE 实践（《Site Reliability Engineering》）。传统 SRE 将 SLO 应用于服务可用性（如"99.99% 可用"），但在 Agent 系统中需要扩展到更丰富的维度：

1. **可用性 SLO**——Agent 是否是"可服务"的（LLM API 可用、工具可用）
2. **性能 SLO**——Agent 的响应速度（延迟）
3. **质量 SLO**——Agent 的响应质量（正确性、有用性）
4. **成本 SLO**——Agent 的 Token 消耗效率

其中质量 SLO 是 Agent 系统独有的，也是实现难度最大的。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 无 SLO | 没有定义服务质量目标 | 无法量化系统好坏，所有优化都是凭感觉 |
| 单一可用性 SLO | 只看系统是否在运行 | Agent 在线但输出完全错误也没有告警 |
| 固定目标无弹性 | SLO 目标是 100% 完美 | 不切实际导致团队无视 SLO |
| SLO 不与部署流程关联 | SLO 只是数据看板 | 发布新版本时不考虑当前错误预算 |

## 核心矛盾

| 矛盾维度 | 无 SLO | 严格 SLO | 弹性 SLO |
|---------|-------|---------|---------|
| 目标明确性 | 无 | 好 | 好 |
| 灵活性 | N/A | 差（"99.9% 太严格导致无法发布"） | 好 |
| 运维纪律 | 无 | 好 | 好 |
| 团队可接受度 | N/A | 低 | 高 |

**核心矛盾**：SLO 越高，运维约束越大；SLO 越低，用户感知越差。**关键原则**：SLO 应该对外（用户承诺的）略高于对内（目标值），保留安全余量。

## SLI / SLO / SLA 的区别

| 概念 | 定义 | 示例 | 所有者 |
|------|------|------|-------|
| SLI (Indicator) | 实际测量的指标值 | 最近 5 分钟的 P95 延迟 = 3.2s | 工程团队 |
| SLO (Objective) | 目标的阈值 | 月度 P95 延迟 < 10s | 工程团队 |
| SLA (Agreement) | 对外承诺 + 违约处罚 | 月度可用性 99.9%，否则退款 10% | 业务/Sales |

## 当前主流优化方向

1. **多维度 SLO**——Agent 系统需要同时满足可用性、延迟、质量、成本的 SLO，任一维度超标都算违反 SLO
2. **滚动窗口 SLO**——使用滚动时间窗口（如"过去 30 天"）而非固定日历月计算 SLO，避免月底"突然发现错误预算不足"
3. **服务级 SLO 与 Agent 级 SLO 分层**——服务级 SLO（整个 Agent 平台的可用性）、Agent 级 SLO（特定 Agent 如客服 Agent 的延迟）、用户级 SLO（特定租户的体验）
4. **SLO 的自动协商**——基于 Agent 的动态负载自动调整 SLO 目标（高峰期适当放宽延迟 SLO，低峰期收紧）
5. **质量 SLO 的 LLM-as-Judge**——使用 LLM 评估 Agent 的响应质量（正确性、完整性、安全性），将评估分数纳入 SLO

## 实现的最大挑战

1. **质量 SLO 的客观测量**——Agent 的响应质量是主观的，使用 LLM-as-Judge 评估会引入"评判者的偏差"。不同 LLM 作为评判者可能给出不同评分
2. **SLO 与成本之间的权衡**——提高 SLO（如使用更强的模型、更多的推理步骤）会增加 Token 消耗，SLO 目标和成本控制需要平衡
3. **错误预算消耗的速度**——Agent 系统的错误预算可能在几分钟内耗尽（如 LLM 提供商的区域性故障），传统基于月的错误预算模型不适用
4. **多 Agent 的 SLO 聚合**——一个端到端请求涉及多个 Agent，每个 Agent 有自己的 SLO，如何聚合为"用户体验 SLO"？

## 能力边界

**能做什么：**
- 量化 Agent 系统的服务质量水平
- 为团队提供客观的"是否应该发布"的门禁标准
- 在错误预算不足时暂停新功能发布，聚焦稳定性
- 建立用户期望与工程能力之间的桥梁

**不能做什么：**
- 不能提高 Agent 的响应质量——SLO 只是衡量标准
- 不能完美预测 Agent 的失败——Agent 的行为是不确定的
- 不能解决 SLO 定义的分歧——"什么是 Agent 的 '错误响应'"需要团队达成共识
- 不能替代根本原因分析——SLO 告诉你目标的达成情况，但不告诉你为什么没达成

## 最终工程优化

1. **SLO 的自动化仪表盘**——在 Grafana 中建立 SLO 看板，实时展示每个 SLO 的达成情况和错误预算消耗率，用红色/黄色/绿色标记状态
2. **错误预算消耗的加速告警**——当错误预算消耗速度超过正常值的 2 倍时（即使总量还没超标），触发预警，提前介入
3. **SLO 的定期 Review**——每月 Review SLO 的定义是否仍然合理：目标值是否需要调整？指标定义是否需要优化？遗漏了哪些重要维度？
4. **多速度 SLO 分层**——将用户请求分为"关键路径"（核心功能）和"非关键路径"（增值功能），为关键路径设置更严格的 SLO，在错误预算不足时优先保证核心功能
5. **SLO 驱动容量规划**——当错误预算消耗的分析显示"延迟 SLO 超标是因为 LLM 调用排队"时，触发自动扩容决策，将容量规划从"凭经验"变为"数据驱动"
