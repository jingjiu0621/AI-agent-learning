# 7.6.6 Risk-Aware Planning — 风险感知规划

## 简单介绍

风险感知规划（Risk-Aware Planning）是在规划过程中主动识别、评估和缓解风险的规划方法。与概率规划关注"哪个选择最优"不同，风险感知规划关注"哪些风险不可接受"。

## 基本原理

风险感知规划的四步流程：

1. **风险识别**：列出所有可能出问题的地方
2. **风险评估**：每个风险的发生概率 × 潜在影响 = 风险等级
3. **风险缓解**：对高风险项制定应对措施
4. **风险监控**：执行过程中持续监控风险指标

## 背景

风险管理在项目管理（PMBOK）和安全工程中有成熟的 ISO 标准。将这些方法论引入 Agent 规划，让 Agent 具备了"先想清楚可能出什么问题，再制定应对方案"的前瞻性能力。

## 之前针对这个问题的做法与结果

1. **无风险感知**：规划时假设一切顺利。结果：出问题时措手不及。
2. **事后风险管理**：出现问题后分析风险。结果：亡羊补牢，但已经造成了损失。
3. **事前风险评估**：规划阶段就识别和缓解风险。结果：最佳实践，能显著降低故障率。

## 工程实现

```python
class RiskAwarePlanner:
    async def plan_with_risk_management(self, task: str) -> RiskAwarePlan:
        # 1. 风险识别
        risks = await self._identify_risks(task)
        
        # 2. 风险评估
        for risk in risks:
            risk.severity = risk.probability * risk.impact
            risk.level = "critical" if risk.severity > 0.7 else \
                         "high" if risk.severity > 0.4 else \
                         "medium" if risk.severity > 0.2 else "low"
        
        # 3. 风险缓解
        for risk in risks:
            if risk.level in ["critical", "high"]:
                risk.mitigation = await self._generate_mitigation(risk)
        
        # 4. 风险指标
        indicators = self._define_risk_indicators(risks)
        
        return RiskAwarePlan(
            base_plan=await self._create_base_plan(task),
            risk_assessment=risks,
            risk_indicators=indicators
        )
    
    async def monitor_risks(self, plan: RiskAwarePlan, 
                             execution_context: dict) -> list[Alert]:
        """执行中监控风险指标"""
        alerts = []
        for indicator in plan.risk_indicators:
            current_value = await self._check_indicator(indicator)
            if current_value > indicator.threshold:
                alerts.append(Alert(
                    risk=indicator.risk,
                    message=f"风险指标 {indicator.name} 超阈值: {current_value}"
                ))
        return alerts
```

最大挑战是**风险识别的完备性**——LLM 可能漏掉某些"不那么明显"但实际可能发生的风险。缓解方案：让 LLM 从多个角度识别风险（技术的、操作的、外部的、安全角度的）。

## 适合场景

- 高风险领域的 Agent（金融、医疗、自动驾驶）
- 需要审计和合规的生产系统
- 任何"失败会造成显著损失"的场景
