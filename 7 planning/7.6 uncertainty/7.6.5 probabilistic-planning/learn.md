# 7.6.5 Probabilistic Planning — 概率规划

## 简单介绍

概率规划（Probabilistic Planning）是在考虑不确定性的情况下制定最优行动策略的规划方法。Agent 不对"哪个行动最好"做确定性假设，而是为每个可能的行动分配成功概率和预期效用，选择期望效用最大化的路径。

## 基本原理

概率规划的核心概念：

1. **状态概率**：每个可能状态的概率分布
2. **行动效果**：每个行动在不同状态下产生不同结果的概率
3. **期望效用**：每个行动可能结果的效用加权和
4. **最优策略**：选择最大化期望效用的行动序列

## 背景

概率规划在机器人学（POMDP 规划）和运筹学（随机优化）中有成熟的理论基础。在 LLM Agent 中，概率规划的意义在于：Agent 面对多个可能的行动路径时，不是盲目选择"看起来最好的"那一条，而是量化评估每条路径的风险和收益。

## 之前针对这个问题的做法与结果

1. **确定性规划**：假设所有行动都会产生预期结果。结果：简单但脆弱。
2. **条件规划**：在分支点预设 if-then。结果：更稳健但需要预判所有可能分支。
3. **概率规划**：用概率和期望效用做决策。结果：最稳健但计算成本最高。

## 工程实现

```python
class ProbabilisticPlanner:
    async def plan_with_uncertainty(self, task: str) -> ProbabilisticPlan:
        # 生成多个可能的规划路径
        paths = await self._generate_paths(task, n=5)
        
        # 评估每条路径的成功概率和预期成本
        scored_paths = []
        for path in paths:
            prob_success = await self._estimate_success_prob(path)
            expected_cost = await self._estimate_cost(path)
            expected_utility = prob_success * self._utility(path)
            scored_paths.append({
                "path": path,
                "prob_success": prob_success,
                "expected_cost": expected_cost,
                "expected_utility": expected_utility
            })
        
        # 选择期望效用最大的路径
        best = max(scored_paths, key=lambda x: x["expected_utility"])
        return best
```

最大挑战是**概率估计的准确性**——要让 LLM 准确估计"这个步骤的成功概率"非常困难。LLM 倾向于高估成功概率（乐观偏差）。缓解方案：让 LLM 同时给出"乐观"和"悲观"估计，取中间值；或使用历史统计数据修正 LLM 的概率估计。

## 适合场景

- 高风险决策（每一步选择都有重大影响）
- 资源受限场景（需要在不确定中做出最优分配）
- 作为高级 Agent 的决策引擎核心
