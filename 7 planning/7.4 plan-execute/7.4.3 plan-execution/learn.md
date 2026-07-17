# 7.4.3 Plan Execution — 计划执行：按序 / 并行调度

## 简单介绍

计划执行（Plan Execution）是将验证过的计划转化为实际行动的阶段。执行引擎负责调度步骤、传递中间结果、处理错误，确保计划被忠实而高效地执行。

## 基本原理

计划执行引擎的核心职责：

1. **调度**：根据步骤依赖图决定执行顺序（串行还是并行）
2. **上下文传递**：将前置步骤的输出作为后置步骤的输入
3. **状态跟踪**：记录每个步骤的状态（待执行/执行中/成功/失败）
4. **结果收集**：汇总所有步骤的结果，形成最终输出

## 背景

在经典的 Plan-and-Execute 中，执行阶段是一个被动的"播放器"——按照计划播放步骤序列。现代实现中，执行引擎承担了更多智能职责，包括在步骤失败时触发重新规划、在上下文不足时触发上下文压缩等。

## 之前针对这个问题的做法与结果

1. **完全串行执行**：严格按计划顺序，一步完成后执行下一步。结果：简单可靠，但效率低，忽略了不依赖的步骤可以并行。
2. **依赖驱动执行**：根据依赖图，无依赖的步骤并行执行。结果：效率显著提升，但对执行引擎要求更高。
3. **ReAct 混合执行**：执行步骤时，每个步骤内部使用 ReAct 循环（允许步骤级灵活调整）。结果：兼具计划的结构性和执行的灵活性。

## 核心矛盾

**计划忠实度 vs 执行灵活性**：严格执行计划保证了方向的正确性，但可能错过了执行过程中出现的优化机会；灵活执行能抓住机会，但可能偏离原定计划。

## 当前主流优化方向

1. **分层执行**：顶层的计划阶段严格执行，每个步骤内部的实现使用 ReAct 灵活执行。这是"计划要严，执行要活"的最佳实践。
2. **状态机驱动的执行器**：每个步骤定义一个状态机（pending→ready→running→success/failed），执行引擎按状态转换驱动。
3. **检查点机制**：在计划的自然断点处（如阶段切换时）插入检查点，验证当前状态是否符合预期，再决定继续执行还是调整。
4. **进度报告**：执行引擎实时生成进度报告（已完成/总步骤数、当前状态、预估剩余时间），增强可观测性。

## 实现的最大挑战

```python
class PlanExecutor:
    def __init__(self, tools: dict):
        self.tools = tools
        self.context: dict[str, Any] = {}
        self.step_states: dict[int, StepState] = {}
    
    async def execute(self, plan: Plan) -> ExecutionResult:
        # 初始化步骤状态
        for step in plan.steps:
            self.step_states[step.step_id] = StepState.PENDING
        
        # 按依赖关系执行
        completed = set()
        while len(completed) < len(plan.steps):
            # 找出所有可执行的步骤（依赖都已完成的）
            ready = self._get_ready_steps(plan, completed)
            if not ready:
                # 死锁检测
                if not completed:
                    raise ExecutionError("存在无法满足的依赖关系")
                break
            
            # 执行就绪步骤（并行）
            results = await asyncio.gather(*[
                self._execute_step(step) for step in ready
            ])
            
            # 处理结果
            for step, result in zip(ready, results):
                if result.success:
                    self.context[step.expected_output] = result.data
                    completed.add(step.step_id)
                    self.step_states[step.step_id] = StepState.SUCCESS
                else:
                    self.step_states[step.step_id] = StepState.FAILED
                    # 处理失败（触发修复或重新规划）
                    if not await self._handle_step_failure(step, result, plan):
                        return ExecutionResult(
                            success=False,
                            failed_step=step.step_id,
                            partial_results=self.context
                        )
        
        return ExecutionResult(
            success=len(completed) == len(plan.steps),
            context=self.context
        )
    
    def _get_ready_steps(self, plan: Plan, completed: set) -> list[Step]:
        """找出所有依赖已完成的步骤"""
        ready = []
        for step in plan.steps:
            if self.step_states[step.step_id] != StepState.PENDING:
                continue
            if all(dep in completed for dep in step.depends_on):
                ready.append(step)
        return ready
    
    async def _execute_step(self, step: Step) -> StepResult:
        """执行单个步骤"""
        self.step_states[step.step_id] = StepState.RUNNING
        
        # 准备输入
        inputs = {inp: self.context.get(inp) for inp in step.expected_input}
        
        if step.tool is None:
            # "思考"步骤——无需调用工具
            return StepResult(success=True, data=f"[思考] {step.description}")
        
        tool = self.tools.get(step.tool)
        if not tool:
            return StepResult(success=False, error=f"工具 '{step.tool}' 不可用")
        
        try:
            data = await tool.execute(**inputs)
            return StepResult(success=True, data=data)
        except Exception as e:
            return StepResult(success=False, error=str(e))
```

最大挑战是**步骤失败时的状态一致性**：如果一个并行执行的步骤失败，但同层的其他步骤已经成功，执行引擎需要确保已成功步骤的结果不会污染失败后的重试或修复过程。方案一：快照隔离——每层执行前保存上下文快照，失败时回滚到快照点。方案二：标记失效——失败步骤的输出被标记为"unreliable"，下游步骤不能使用。

## 能力边界和结果边界

- **能力**：能可靠执行 10-50 步的计划，支持步骤间依赖解析和并行调度
- **边界**：对于需要实时交互的计划（如"每 5 分钟检查一次数据"），执行引擎需要额外的时间触发机制；不能处理步骤数量超过上下文窗口容纳范围的长计划
- **效果**：良好的计划执行引擎能让计划以 90%+ 的准确率执行完成，且提供完整的执行日志

## 核心优势

执行引擎将"计划"和"执行"的关注点分离——规划器专注于"做什么"，执行器专注于"怎么做"。这种分离让两个组件可以独立优化。

## 工程优化方向

- 执行中间结果的可视化：实时显示 DAG 执行状态
- 步骤超时保护：每个步骤设置最大执行时间，超时后自动标记为失败
- 执行上下文的心跳检测：如果工具调用无响应（hang），主动中断并尝试替代方案

## 适合场景

- 步骤数量和依赖关系已知的确定型任务
- 执行阶段需要高可靠性的生产环境
- 作为 Plan-and-Execute 模式的核心执行引擎
