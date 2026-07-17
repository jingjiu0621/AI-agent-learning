# 7.4.4 Plan Adaptation — 执行中动态调整

## 简单介绍

计划调整（Plan Adaptation）是指 Agent 在执行计划的过程中，根据实际观察到的结果动态修改剩余计划的能力。这是 Plan-and-Execute 区分于僵化的 Plan-then-Do 的关键特性。

## 基本原理

计划调整发生在以下几种情况：

1. **信息更新**：执行过程中获得了计划制定时不知道的新信息
2. **中间结果偏差**：实际输出与预期输出不符，需要调整后续步骤
3. **外部变化**：环境状态发生改变（如 API 版本更新、数据发生变化）
4. **优化机会**：发现比原计划更高效的路径

## 背景

经典 AI 规划中的"条件规划"（Conditional Planning）已经考虑了执行中的分支——计划中包含 if-then 分支。但在 LLM Agent 场景中，分支条件往往不是预先可知的（你不知道执行到第 3 步时会发现什么），因此需要在运行时动态判定是否调整。

## 之前针对这个问题的做法与结果

1. **不调整**：严格执行原计划，无论执行中发现什么。结果：简单但脆弱，一步偏差可能导致后续所有步骤都无效。
2. **人工介入调整**：执行卡住时由人类修改计划。结果：可靠但慢。
3. **完全重新规划**：每次遇到意外都从头生成新计划。结果：彻底但浪费了已执行的步骤和中间结果。

## 核心矛盾

**调整的成本 vs 收益**：频繁调整计划可以保持计划与实际情况的匹配，但每次调整都需要 LLM 调用和重新验证。关键是区分"需要调整的意外"和"可以忽略的噪音"。

## 当前主流优化方向

1. **局部重新规划**：只调整受影响的部分——检测到步骤 X 的输出与预期不符时，只重新规划 X 之后且依赖 X 的步骤，保留不依赖 X 的部分不变。
2. **计划补丁**：不重写整个计划，而是生成"缝合"操作——在现有计划的基础上插入、替换或删除步骤。
3. **条件锚点**：在计划的关键决策点预设"如果 X 则 A，如果 Y 则 B"的分支，让执行中的选择不是"调整"而是"按预判的分支走"。
4. **自适应步进**：每执行一步或几步后，让 LLM 快速评估"当前计划是否仍然合理"，只在实际偏离超过阈值时才触发调整。

## 实现的最大挑战

```python
class PlanAdapter:
    async def adapt(self, original_plan: Plan, completed_steps: list[StepResult],
                    current_context: dict, deviation: Deviation) -> Plan:
        """根据执行偏差调整计划"""
        
        # 检测受影响的下游步骤
        affected = self._find_affected_steps(
            original_plan, 
            deviation.origin_step_id,
            current_context
        )
        
        if self._is_minor_deviation(deviation):
            # 微小偏差——只需调整参数，不需要改计划
            return self._patch_parameters(original_plan, affected, deviation)
        
        # 计算已完成的步骤中还有哪些产出可用
        available_outputs = {
            r.step_id: r.data for r in completed_steps if r.success
        }
        
        # 局部重新规划——只对受影响的部分
        partial_task = self._construct_partial_task(
            original_plan, affected, available_outputs, deviation
        )
        
        prompt = f"""
        原计划目标: {original_plan.overall_strategy}
        
        执行到步骤 {deviation.origin_step_id} 时发现:
        {deviation.description}
        
        预期: {deviation.expected}
        实际: {deviation.actual}
        
        已完成的步骤及产出:
        {available_outputs}
        
        需要调整的步骤: {[s.name for s in affected]}
        
        请调整这些步骤以适应当前情况。
        保持与原计划的整体方向一致。
        输出调整后的步骤列表（JSON 数组）。
        """
        
        adjusted_steps = await self.llm.predict(prompt)
        
        # 合并：未受影响的部分 + 调整后的部分
        return self._merge_plan(original_plan, affected, adjusted_steps)
    
    def _find_affected_steps(self, plan: Plan, origin_id: int,
                              context: dict) -> list[Step]:
        """找出受偏差影响的后续步骤"""
        affected = []
        # 通过依赖传递性找到所有直接或间接依赖 origin_id 的步骤
        for step in plan.steps:
            if step.step_id <= origin_id:
                continue
            # 如果步骤的输入链中包含 origin_id 的输出
            if self._depends_on(step, origin_id, plan):
                affected.append(step)
        return affected
```

最大挑战是**调整后的计划一致性保证**：局部调整可能产生与原计划其他部分不一致的步骤（如重复、冲突或遗漏）。需要在调整后做一次快速一致性检查。

## 能力边界和结果边界

- **能力**：能处理 60-70% 的执行中意外情况，通过局部调整恢复而不需要完全重新规划
- **边界**：如果预期的偏差太大（如发现原计划的方法论完全错了），局部调整不够，需要完全重新规划
- **效果**：好的调整机制能让计划完成率从 50% 提升到 85%+，同时只增加 20-30% 的额外 token 消耗

## 核心优势

计划调整让 Plan-and-Execute 从"僵硬"变为"灵活"——保留了计划的结构性优势，同时获得了 ReAct 式的适应性。这是生产系统中最实用的折中方案。

## 工程优化方向

- 偏差分类模型：训练/配置一个分类器快速判断偏差类型（微小/中等/重大），指导不同的调整策略
- 调整效果的追踪：记录每次调整后任务是否成功完成，学习哪些调整策略最有效
- "撤销"能力：如果调整后的新计划表现更差，支持回滚到原计划的对应分支

## 适合场景

- 环境变化频繁的动态任务
- 执行过程中可能发现新信息的研究类任务
- 需要在"按计划执行"和"灵活应变"之间平衡的生产系统
