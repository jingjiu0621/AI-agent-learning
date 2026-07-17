# 7.6.1 Uncertainty Sources — 不确定性来源

## 简单介绍

不确定性来源（Uncertainty Sources）是理解 Agent 在何处、为何产生不确定性的基础。不确定性的来源可以分为三类：模型不确定性、工具不确定性和环境不确定性。

## 基本原理

三类不确定性的特征：

| 来源 | 举例 | 特征 | 是否可控 |
|------|------|------|---------|
| 模型不确定性 | LLM 对某个事实不确定 | 输出结果的 token 概率低 | 部分可控（换模型、加 context） |
| 工具不确定性 | API 超时、返回错误 | 与外部系统可靠性相关 | 不可控（只能缓解） |
| 环境不确定性 | 任务描述模糊、用户意图不明 | 与问题定义相关 | 部分可控（主动澄清） |

```
Agent 决策的不确定性 = LLM_uncertainty + Tool_uncertainty + Environment_uncertainty
```

## 背景

不确定性管理在经典决策理论中有成熟框架（概率论、贝叶斯决策理论）。在 LLM Agent 中，不确定性来源更加多样化——不仅来自数据和环境，还来自 LLM 本身的概率性输出。

## 工程实现

```python
class UncertaintyTracker:
    def __init__(self):
        self.sources = {
            "llm": {"confidence": 1.0, "factors": []},
            "tool": {"confidence": 1.0, "factors": []},
            "environment": {"confidence": 1.0, "factors": []},
        }
    
    def record_llm_uncertainty(self, token_logprobs: list[float]):
        """基于 token 概率记录模型不确定性"""
        avg_prob = sum(token_logprobs) / len(token_logprobs)
        self.sources["llm"]["confidence"] = avg_prob
        if avg_prob < 0.7:
            self.sources["llm"]["factors"].append("low_token_probability")
    
    def record_tool_uncertainty(self, tool_name: str, status: str):
        """记录工具调用的不确定性"""
        if status in ["timeout", "error", "unexpected_result"]:
            self.sources["tool"]["confidence"] *= 0.7
            self.sources["tool"]["factors"].append(f"{tool_name}_{status}")
    
    def overall_confidence(self) -> float:
        """综合置信度（各来源的乘积）"""
        conf = 1.0
        for source in self.sources.values():
            conf *= source["confidence"]
        return conf
```

最大挑战是**不确定性来源的耦合**——工具调用失败可能是因为模型选错了参数（模型不确定性→工具不确定性），环境不确定可能是因为模型没有主动澄清（模型不确定性→环境不确定性）。解耦这些因素的因果关系需要更深的分析。

## 适合场景

- 生产级 Agent 的健康监控
- 决定何时触发回退或人工升级
- Agent 行为分析中定位瓶颈环节
