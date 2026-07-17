# 7.4.1 Plan Creation — 计划生成：LLM 规划器设计

## 简单介绍

计划生成（Plan Creation）是 Plan-and-Execute 模式的第一阶段——LLM 作为规划器，分析任务目标、评估可用工具、生成结构化的执行计划。规划器的设计直接影响后续执行的成功率。

## 基本原理

LLM 规划器的输入和输出：

- **输入**：任务描述 + 可用工具列表 + 约束条件（时间、资源、安全策略）
- **处理**：分析任务、拆解步骤、确定顺序和依赖
- **输出**：结构化的计划（步骤列表，每步包含：名称、描述、工具、参数、依赖、预期输出）

```json
{
  "plan": [
    {
      "step_id": 1,
      "name": "获取用户输入参数",
      "tool": "input_parser",
      "input": "raw_task",
      "output": "parsed_params",
      "depends_on": []
    },
    {
      "step_id": 2,
      "name": "查询历史数据",
      "tool": "query_database",
      "input": "parsed_params",
      "output": "historical_data",
      "depends_on": [1]
    }
  ]
}
```

## 背景

Plan-and-Execute 模式的规划器设计经历了几个阶段。早期的"Plan-then-Do"完全严格——一旦规划完成就不允许改变。这导致了严重的脆弱性：如果规划忽略了某个细节，整个执行都会失败。现代 Plan-and-Execute 采用柔性规划，计划是"指南"而非"圣旨"。

## 之前针对这个问题的做法与结果

1. **LLM 自由式规划**：让 LLM 自由生成步骤列表。结果：灵活但格式不一致，难以自动化执行。
2. **结构化规划**：用 JSON Schema 约束规划输出。结果：可自动化执行，但 LLM 生成的步骤可能不完整。
3. **模板填充规划**：对常见任务类型预定义规划模板。结果：可靠但缺乏灵活性，无法处理模板之外的场景。

## 核心矛盾

**计划的详细度 vs 生成成本**：越详细的计划越容易执行，但生成详细计划需要更多 token 和更长的 LLM 推理时间。太粗略的计划导致执行时频繁"计划跟不上变化"，太详细的计划在需求变更时浪费。

## 当前主流优化方向

1. **渐进式计划生成**：先生成高层计划（5-10 步），在执行每步前再生成该步的详细子计划。这结合了 Plan-and-Execute 和 ReAct 的优点。
2. **计划骨架+细节填充**：先用快速 LLM 调用生成计划骨架，再逐个步骤填充细节。这减少了 LLM 单次生成长计划的认知负担。
3. **工具感知规划**：规划器生成计划时已经知道每个工具的接口和限制，生成的步骤天然与工具能力对齐。
4. **可执行性自检**：规划器生成计划后，自我检查每个步骤是否可执行（工具是否存在、参数是否完整）。

## 实现的最大挑战

```python
class PlanCreator:
    async def create_plan(self, task: str, tools: list[Tool], 
                          context: dict) -> Plan:
        prompt = f"""
        你是一个任务规划专家。为以下任务生成可执行的步骤计划。
        
        任务: {task}
        
        可用工具:
        {self._format_tools(tools)}
        
        额外上下文:
        {context}
        
        生成计划时遵循:
        1. 每个步骤必须对应一个可用工具或"思考/分析"步骤
        2. 明确标注步骤间的依赖关系
        3. 每个步骤指定输入来源和输出产物
        4. 总步骤数控制在 5-12 步
        5. 如果任务复杂，建议分层规划而非单层长列表
        
        输出 JSON:
        {{
            "plan_name": str,
            "overall_strategy": str,
            "steps": [
                {{
                    "step_id": int,
                    "name": str,
                    "description": str,
                    "tool": str | null,
                    "expected_input": [str],
                    "expected_output": str,
                    "depends_on": [int],
                    "success_criteria": str,
                    "fallback": str | null
                }}
            ],
            "estimated_total_steps": int,
            "risk_factors": [str]
        }}
        """
        result = await self.llm.predict(prompt)
        return Plan(**json.loads(result))
    
    async def create_plan_with_hallucination_check(self, task: str, 
                                                     tools: list[Tool]) -> Plan:
        """带幻觉检测的计划生成"""
        plan = await self.create_plan(task, tools, {})
        
        # 检查计划中的工具幻觉：计划中使用的工具是否真的在可用列表中
        tool_names = {t.name for t in tools}
        for step in plan.steps:
            if step.tool and step.tool not in tool_names:
                # 工具不存在——让规划器修复
                prompt = f"""
                步骤 "{step.name}" 使用的工具 "{step.tool}" 不在可用工具列表中。
                可用工具: {tool_names}
                请为这个步骤选择一个实际可用的工具，或将其标注为"思考"步骤。
                """
                fix = await self.llm.predict(prompt)
                # 应用修复
                ...
        
        return plan
```

最大挑战是**规划器对工具能力的"幻觉"**——规划器经常生成计划步骤调用一个"不存在"或"功能被夸大"的工具。例如，规划器可能假设有一个"综合分析工具"可用，但实际系统中只有原始的数据查询工具。缓解方案：工具描述要诚实（不要夸大能力），且在规划后增加工具可用性验证步骤。

## 能力边界和结果边界

- **能力**：能为 80% 以上的常见任务生成合理的、可执行的计划（5-12 步）
- **边界**：对于需要高度专业领域知识的任务（如设计电路），规划器生成的计划可能缺少关键步骤；对于需要紧密人机协作的任务，计划中应该适当插入"用户确认"节点
- **输出结构**：好的计划应该让执行器不需要额外思考就能直接执行（"傻瓜式计划"）

## 核心优势

LLM 规划器的最大价值在于它能"一次思考，多次使用"——规划阶段投入的推理成本可以在整个执行阶段摊销。对于复杂任务，这是比 ReAct 更经济的策略。

## 工程优化方向

- 计划生成的反馈闭环：将执行阶段的成功率数据反馈到规划器，持续改进规划质量
- 计划模板缓存：对高频任务缓存计划模板，只需要填充参数而不是重新生成
- 规划器的"思考过程"记录：规划器在生成计划时产生的推理过程可以作为执行阶段的参考上下文

## 适合场景

- 步骤间有明确依赖关系的流程型任务
- 需要全局最优而非局部最优的优化型任务
- 执行阶段可能由较弱的 LLM 或自动化脚本运行的场景
