# 7.6.2 Confidence Scoring — 置信度评分

## 简单介绍

置信度评分（Confidence Scoring）是量化 Agent 对其决策或输出确定程度的机制。良好的置信度评分让 Agent 知道"何时自己有把握、何时需要求助"。

## 基本原理

置信度评分的来源：

1. **Token 概率**：LLM 输出 token 的 logprob 均值（越低越不确定）
2. **自洽性**：多次采样输出的结果一致性（越一致越确定）
3. **工具结果质量**：工具返回的数据是否完整、及时、合理
4. **历史记录**：类似任务在历史上的成功率

综合置信度 = f(token_prob, self_consistency, tool_quality, history)

## 背景

传统机器学习模型通常有 native 的置信度估计（softmax 概率）。LLM 的置信度估计更加复杂——因为 LLM 的"高概率 token"不一定意味着"正确答案"，token 概率主要反映语言的流畅性而非事实的正确性。

## 之前针对这个问题的做法与结果

1. **仅用 token 概率**：用生成 token 的平均 logprob 作为置信度。结果：与"答案质量"的相关性较弱，因为流畅的错误答案也可以有高概率。
2. **Self-Consistency**：多次采样，选最一致的答案。结果：显著提升置信度评估的准确性，但成本是 N 倍推理。
3. **LLM 自我评分**：让 LLM 给自己的答案打分。结果：LLM 倾向于高估自己的答案（确认偏差）。

## 核心矛盾

**置信度-准确度的校准**：理想的置信度应该是"校准"（Calibrated）的——置信度 80% 意味着实际正确率 80%。但 LLM 的置信度往往过校准（overconfident），置信度 90% 时实际正确率可能只有 70%。

## 工程实现

```python
class ConfidenceScorer:
    async def score(self, answer: str, context: dict) -> float:
        scores = []
        
        # 1. Token 概率
        if "token_logprobs" in context:
            prob_score = self._token_prob_score(context["token_logprobs"])
            scores.append(("token", prob_score, 0.3))
        
        # 2. Self-Consistency（如果有多条轨迹）
        if "multiple_trajectories" in context:
            consist_score = self._consistency_score(context["multiple_trajectories"])
            scores.append(("consistency", consist_score, 0.4))
        
        # 3. 验证器评分
        if "verification_result" in context:
            ver_score = context["verification_result"]["score"] / 100
            scores.append(("verification", ver_score, 0.3))
        
        # 加权综合
        total = sum(weight * score for _, score, weight in scores)
        return min(1.0, max(0.0, total))
```

最大挑战是**校准**——让置信度分数与真实准确率对齐。实践中，建议定期收集"置信度 vs 实际正确率"的数据，绘制校准曲线，然后根据曲线做后校准（Post-Calibration）。

## 适合场景

- 决定是否需要人工审核的场景
- 作为回退策略的触发信号
- 评估 Agent 输出的可信任度
