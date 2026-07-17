# 7.4.6 Hierarchical Plan-Execute — 分层 Plan-and-Execute

## 简单介绍

分层 Plan-and-Execute（Hierarchical Plan-Execute）将规划-执行模式扩展到多层抽象——每一层都有自己的规划器和执行器，上层制定战略计划，下层制定战术计划，最下层执行具体步骤。这种模式适合超复杂、长步骤、需要多级抽象的任务。

## 基本原理

分层 Plan-and-Execute 的核心架构：

```
Level 0 (战略层):  总体目标 → 阶段划分
Level 1 (战术层):  每个阶段 → 详细步骤链
Level 2 (执行层):  每个步骤 → ReAct 循环
```

每一层的职责：
- **上层**（规划器）：制定该层的大粒度计划，分配给下层执行
- **下层**（执行器）：接收上层的计划步骤，将其分解为更细粒度的子步骤执行

这种分层天然解决了"规划粒度"的矛盾——上层不用关心细节，下层不用操心全局。

## 背景

分层规划的思想源于经典 AI 的 HTN（Hierarchical Task Network）规划和机器人学的"分层控制架构"。在 LLM Agent 中的应用借鉴了软件工程中的"分层架构"——每一层只关注自己层次的抽象，通过定义清晰的接口（计划-执行协议）来协作。

## 之前针对这个问题的做法与结果

1. **扁平 Plan-and-Execute**：单层规划器生成 10-20 步的详细计划。结果：规划器压力大，长计划的质量随长度下降。
2. **两阶段分解**：先粗分再细分。结果：比扁平好，但两阶段之间的衔接需要额外管理。
3. **动态分层**：根据任务复杂度自动决定层数。结果：灵活但实现复杂，且 Agent 对"需要几层"的判断不一定准确。

## 核心矛盾

**层数的确定**：层数太少退化为扁平规划（失去分层优势），层数太多引入不必要的层级开销和层间通信延迟。

## 当前主流优化方向

1. **三层固定架构**：战略→战术→执行。经过实践检验，三层通常是复杂任务的最优折中——足够深以管理复杂度，足够浅以避免过度分层。
2. **层间上下文传递**：上层只传递"目标+约束+关键中间结果"，不传递细粒度的执行日志。这保持了每层的关注点分离。
3. **跨层反馈**：当执行层发现当前计划无法执行时，不仅仅是本层调整，还需要向上层报告"这个目标在当前约束下无法实现"，触发上层调整目标或提供更多资源。
4. **可选的层级跳过**：对于简单任务，可以直接从战略层跳到执行层，跳过战术层。层级是逻辑概念，不是固定的代码结构。

## 实现的最大挑战

```python
class HierarchicalPlanExecute:
    """分层 Plan-and-Execute 实现"""
    
    def __init__(self, llm, tools: dict):
        self.llm = llm
        self.tools = tools
    
    async def run(self, task: str) -> str:
        # Phase 0: 任务理解
        task_analysis = await self._analyze_task(task)
        
        if task_analysis["complexity"] == "simple":
            # 简单任务→直接执行层
            return await self._execution_level(task, {})
        
        # Phase 1: 战略层——划分阶段
        strategic_plan = await self._strategic_level(task, task_analysis)
        
        # Phase 2: 逐阶段战术执行
        final_result = {}
        for phase in strategic_plan["phases"]:
            # 战术层——为每个阶段制定详细计划
            tactic = await self._tactic_level(phase, final_result)
            
            # 执行层——执行战术计划的每个步骤
            phase_result = await self._execution_level(tactic, final_result)
            final_result.update(phase_result)
            
            # 阶段间检查点
            status = await self._checkpoint(phase, phase_result, task)
            if not status["on_track"]:
                # 偏离轨道——可能需要在战略层面调整
                ...
        
        return self._synthesize(final_result, task)
    
    async def _strategic_level(self, task: str, analysis: dict) -> StrategicPlan:
        """战略层：将总体目标划分为 3-5 个阶段"""
        prompt = f"""
        为以下任务制定高层战略计划（3-5 个阶段）。
        每个阶段应该是一个相对独立的工作单元。
        
        任务: {task}
        分析: {analysis}
        
        输出 JSON:
        {{
            "phases": [
                {{
                    "phase_id": int,
                    "name": str,
                    "objective": str,
                    "expected_deliverable": str,
                    "dependencies": [int]  // 依赖哪些前置阶段
                }}
            ]
        }}
        
        注意：这是战略层，不需要具体步骤，只需要阶段划分和目标定义。
        """
        return StrategicPlan(**json.loads(await self.llm.predict(prompt)))
    
    async def _tactic_level(self, phase: Phase, 
                             context: dict) -> TacticalPlan:
        """战术层：为某个阶段制定 3-8 步的详细计划"""
        prompt = f"""
        为以下阶段制定战术计划：
        
        阶段目标: {phase.objective}
        预期产出: {phase.expected_deliverable}
        可用上下文: {context}
        可用工具: {list(self.tools.keys())}
        
        输出 3-8 个具体步骤，
        每个步骤指定工具和参数（JSON 格式）。
        """
        return TacticalPlan(**json.loads(await self.llm.predict(prompt)))
    
    async def _execution_level(self, plan: TacticalPlan, 
                                context: dict) -> dict:
        """执行层：用 ReAct 模式执行每个步骤"""
        # 这里可以使用标准的 ReAct 执行器
        executor = PlanExecutor(self.tools)
        result = await executor.execute(plan)
        return result.context if result.success else {}
```

最大挑战是**层间通信的带宽控制**：上层需要给下层足够的上下文才能让下层正确执行，但传递过多信息又破坏关注点分离。实践中，上层传递给下层的通常是"任务描述 + 关键约束 + 相关中间结果"，不超过 1000 token。

## 能力边界和结果边界

- **能力**：可以可靠处理需要 20-50 步的复杂任务，任务越复杂分层的优势越明显
- **边界**：不适合需要极致低延迟的实时场景（分层引入的通信开销会降低响应速度）；对于只需要 3-5 步的简单任务，分层是过度工程
- **效果**：在复杂任务上，分层架构比扁平 Plan-and-Execute 的成功率高 20-30%，执行效率高 30-50%

## 核心优势

分层架构让 Agent 可以处理"一眼看不到头"的超长任务。每层只关心自己层次的问题，避免了"在规划阶段就想清楚所有细节"的不可能任务。

## 工程优化方向

- 层数自适应：LLM 在任务分析阶段判断需要几层，而不是固定在 3 层
- 层间缓存：上层生成的计划结果可以缓存和复用，避免相同阶段重复规划
- 并行阶段执行：如果阶段之间没有依赖关系，可以在战略层分配后并行执行
- 层级监控仪表盘：展示每层的计划、执行状态、层间通信内容，对调试复杂 Agent 行为至关重要

## 适合场景

- 需要 20+ 步骤的超长链任务
- 任务有自然的层次结构（公司战略→部门计划→个人执行）
- 需要多 Agent 分工的大规模系统（每个层次可以由不同的 Agent 实例负责）
