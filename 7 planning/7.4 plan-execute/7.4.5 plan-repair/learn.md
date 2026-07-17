# 7.4.5 Plan Repair — 失败重规划

## 简单介绍

计划修复（Plan Repair）是比计划调整更激进的机制——当执行中的失败无法通过局部调整解决时，需要放弃部分或全部剩余计划，重新规划一条到达目标的路径。

## 基本原理

计划修复的触发条件通常是：

1. **关键步骤失败**：某个对后续所有步骤都有影响的步骤执行失败，且无法通过重试解决
2. **资源不可用**：计划依赖的资源（工具、API、数据源）在执行时不可用
3. **前提条件不满足**：执行发现任务的前提条件与计划制定时的假设不同
4. **计划失效**：执行中发现的实际情况与原计划的假设严重不符

修复策略的三种级别：
```
L1 局部修复: 替换/跳过失败的步骤
  例: 搜索 API A 不可用 → 改用搜索 API B

L2 路径修复: 从失败点重新规划到目标
  例: 数据源 A 找不到数据 → 重新规划从数据源 B 获取数据的路径

L3 全局修复: 基于当前状态完全重新规划
  例: 任务理解有误 → 基于正确理解重新规划
```

## 背景

在经典 AI 规划中，计划修复有成熟的理论（如基于依赖的回溯修复）。在 LLM Agent 中，修复更依赖 LLM 对"已发生了什么"和"还需要什么"的语义理解，而非形式化的因果推理。

## 之前针对这个问题的做法与结果

1. **完全重试**：从头开始重新执行整个计划。结果：简单但不高效——如果第 8 步失败，前 7 步的重做完全浪费。
2. **完全重新规划**：基于当前状态重新生成全新计划。结果：最优但成本高。
3. **手动修复**：由人类分析失败原因并修改计划。结果：准确但速度慢。

## 核心矛盾

**修复的完全性 vs 修复成本**：全局重新规划最彻底但最贵，局部修复便宜但可能留下隐患——修复后的计划可能在后续执行中暴露出更多问题。

## 当前主流优化方向

1. **失败原因分类驱动的修复**：先对失败原因精确分类（超时/权限/数据缺失/逻辑错误），再根据原因选择最合适的修复策略。
2. **渐进式修复**：先尝试最轻量的修复（L1），如果仍然失败则升级到 L2，再到 L3。
3. **利用已完成的工作**：修复时充分复用已成功步骤的中间结果，避免重复劳动。
4. **修复预算**：设定修复的最大尝试次数或最大 token 预算，超预算则整体失败并报告"无法完成"。

## 实现的最大挑战

```python
class PlanRepairer:
    async def repair(self, plan: Plan, failed_step: StepResult,
                     execution_context: dict) -> RepairResult:
        # Step 1: 失败原因分析
        cause = await self._analyze_failure(failed_step, execution_context)
        
        # Step 2: 选择合适的修复策略
        strategy = self._choose_strategy(cause)
        
        if strategy == "RETRY":
            return await self._retry(failed_step)
        
        elif strategy == "SUBSTITUTE":
            return await self._find_substitute(failed_step, execution_context)
        
        elif strategy == "REPLAN":
            return await self._replan_from_failure(
                plan, failed_step, execution_context
            )
        
        elif strategy == "ABORT":
            return RepairResult(
                success=False,
                message=f"无法修复: {cause.explanation}"
            )
    
    async def _analyze_failure(self, step: StepResult, 
                                context: dict) -> FailureCause:
        prompt = f"""
        分析以下步骤失败的原因:
        
        步骤: {step.step_name}
        工具: {step.tool_name}
        参数: {step.parameters}
        错误: {step.error}
        
        当前上下文状态: {list(context.keys())}
        
        可能的失败原因:
        1. 临时错误（可重试）: API 超时、网络波动
        2. 参数错误: 参数格式/值有问题，修正后可重试
        3. 工具不可用: 该工具当前无法使用，需要替代方案
        4. 先决条件不满足: 缺少执行该步骤所需的前置条件
        5. 数据问题: 输入数据为空/格式错误/不完整
        6. 逻辑错误: 该步骤本就不应该执行（规划错误）
        
        输出 JSON:
        {{
            "category": str,
            "explanation": str,
            "is_retryable": bool,
            "suggested_strategy": "RETRY" | "SUBSTITUTE" | "REPLAN" | "ABORT"
        }}
        """
        result = await self.llm.predict(prompt)
        return FailureCause(**json.loads(result))
    
    async def _replan_from_failure(self, plan: Plan, failed_step: StepResult,
                                    context: dict) -> RepairResult:
        """从失败点开始重新规划到终点"""
        # 确定当前状态：还有哪些目标未完成
        remaining_goal = self._extract_remaining_goal(plan, failed_step.step_id)
        
        # 确定可用资源：已成功步骤的输出
        available = {
            k: v for k, v in context.items() 
            if not k.startswith("_failed_")
        }
        
        prompt = f"""
        原始任务目标: {plan.overall_strategy}
        已完成的步骤: {context}
        剩余目标: {remaining_goal}
        
        由于步骤 "{failed_step.step_name}" 失败（{failed_step.error}），
        请基于当前已完成的工作，规划一条到达剩余目标的新路径。
        
        注意: 不要重复已完成的步骤，充分利用已有的中间结果 {available}。
        """
        
        new_steps = await self.llm.predict(prompt)
        
        return RepairResult(
            success=True,
            strategy="REPLAN",
            new_plan=new_steps,
            message=f"已从步骤 {failed_step.step_id} 处重新规划"
        )
```

最大挑战是**修复后的计划与已完成部分的衔接**：新计划需要正确引用已完成的中间结果，如果新计划假设某个中间结果存在但实际上不存在（或格式不一样），会导致新的失败。缓解方案：在修复后的计划中明确标注"这个输入来自已完成的步骤 X 的输出"，并做类型/格式检查。

## 能力边界和结果边界

- **能力**：能修复 60-80% 的计划执行失败，其中约半数可以通过 L1/L2 轻量修复解决
- **边界**：如果失败的根本原因是 LLM 能力限制（如 LLM 无法理解某个领域的任务），任何修复策略都无法解决
- **效果**：相比"失败后完全重试"，计划修复能节省 40-60% 的 token 消耗和 30-50% 的执行时间

## 核心优势

计划修复是 Plan-and-Execute 模式的"安全网"——它让 Agent 即使遇到执行失败也能优雅地恢复，大大提升了系统的鲁棒性和自主性。

## 工程优化方向

- 修复模式的知识积累：将常见的失败模式和对应的修复策略存入数据库，形成"修复经验库"
- 修复策略的自动选择：通过强化学习优化修复策略选择——哪种修复策略在哪种失败场景下最有效
- 修复的预防性提示：如果 LLM 检测到某个步骤的失败概率较高（如调用的 API 经常超时），可以在执行前就生成备选方案

## 适合场景

- 生产环境中不能轻易失败的 Agent 系统
- 工具调用依赖外部 API，不确定性高的场景
- 长时间运行的任务（修复比从头重做成本低得多）
