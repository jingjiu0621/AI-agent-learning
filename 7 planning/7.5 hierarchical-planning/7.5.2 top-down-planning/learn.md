# 7.5.2 Top-Down Planning — 自顶向下逐层细化

## 简单介绍

自顶向下规划（Top-Down Planning）是最自然的分层规划方法——从最抽象的目标开始，逐步细化到具体的执行步骤。每一步细化都增加了更多的细节和确定性。

## 基本原理

自顶向下规划的过程可以类比"缩放"——从卫星图（战略层）逐步放大到街景图（执行层）：

```
Level 0: "做一个市场调研报告"（目标）
Level 1: "收集数据 → 分析数据 → 撰写报告"（3 个阶段）
Level 2: "搜索行业报告 + 爬取竞品信息 + ..."（阶段拆解）
Level 3: "打开浏览器 → 输入搜索词 → ..."（具体步骤）
```

每层细化都遵循"保持完整性"原则——下一层步骤的集合必须能完整实现上一层的目标。

## 背景

自顶向下规划是经典 AI 规划（特别是 STRIPS 和 HTN）的主流方法。它反映了人类处理复杂问题最自然的策略——先定大方向，再逐步细化。在 LLM 时代，这种方法被证明非常适合 LLM 的推理模式：LLM 在高层抽象上的推理比直接生成细粒度步骤更可靠。

## 之前针对这个问题的做法与结果

1. **一步到位的规划**：直接从目标生成具体步骤。结果：LLM 生成的步骤要么遗漏关键环节，要么顺序不合理。
2. **两阶段细化**：目标→粗粒度步骤→细粒度步骤。结果：比一步到位好，但第二阶段的细化质量受限于第一阶段的分割质量。
3. **递归细化**：持续细化直到每步都可执行。结果：最彻底，但需要设定递归终止条件。

## 核心矛盾

**细化的一致性**：每层细化时，下层的步骤是否真的完整覆盖了上层目标？如果遗漏了，错误会在执行阶段暴露，但那时已经投入了大量细化成本。

## 工程实现要点

```python
class TopDownPlanner:
    async def refine(self, goal: str, level: int = 0) -> list[Step]:
        if self._is_executable(goal):
            return [Step(description=goal, is_atomic=True)]
        
        # 将当前目标分解为子步骤
        sub_goals = await self._decompose(goal, level)
        
        # 完整性验证
        completeness = await self._verify_completeness(goal, sub_goals)
        if not completeness["is_complete"]:
            sub_goals = await self._fix_decomposition(goal, sub_goals, completeness)
        
        # 递归细化每个子步骤
        all_steps = []
        for sg in sub_goals:
            sub_steps = await self.refine(sg, level + 1)
            all_steps.extend(sub_steps)
        
        return all_steps
```

核心挑战是"完整性验证"——如何确保分解没有遗漏。实践中可以在分解 prompt 中要求 LLM 明确回答"这些子步骤是否完整覆盖了原始目标？"，并在发现遗漏时自动补全。

## 适合场景

- 目标明确、路径可以自上而下推理的任务
- 人类参与制定高层目标、Agent 负责细化执行的场景
- 需要在不同抽象层次向 stakeholders 展示规划的层次
