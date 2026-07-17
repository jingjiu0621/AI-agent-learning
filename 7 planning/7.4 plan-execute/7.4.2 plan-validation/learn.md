# 7.4.2 Plan Validation — 计划验证：可行性 / 一致性 / 完整性

## 简单介绍

计划验证（Plan Validation）是在执行计划之前检查计划的可行性、一致性和完整性。就像软件开发中的代码评审一样，计划验证能提前发现计划中的缺陷，避免在执行阶段浪费时间和 token。

## 基本原理

计划验证从三个维度检查：

1. **可行性**：每个步骤是否可用当前工具执行？参数是否完整？
2. **一致性**：步骤间是否有循环依赖？依赖链是否合理？输入-输出匹配？
3. **完整性**：所有必要的步骤是否都被覆盖？是否有遗漏？

## 背景

在经典 AI 规划中，计划验证有形式化的方法（如通过模型检查验证计划是否满足时态逻辑约束）。在 LLM Agent 中，验证更多依赖语义判断——"这个计划从常识上看合理吗？"而非严格的形式化证明。

## 之前针对这个问题的做法与结果

1. **不验证**：生成计划后直接执行。结果：快，但计划中的错误在执行时才会暴露，此时已经浪费了执行成本。
2. **人工验证**：人类审查计划后再放行。结果：最可靠但最慢，不适合自主 Agent。
3. **LLM 自验证**：让 LLM 自己检查自己生成的计划。结果：有一定效果，但 LLM 倾向于认为自己的计划没问题（确认偏差）。

## 核心矛盾

**验证的彻底性 vs 成本**：全面验证需要多次 LLM 调用，验证成本可能接近甚至超过执行成本。需要在验证深度和验证价值之间做权衡。

## 当前主流优化方向

```python
class PlanValidator:
    def __init__(self, llm, tools: dict):
        self.llm = llm
        self.tools = tools
    
    async def validate(self, plan: Plan, task: str) -> ValidationResult:
        issues = []
        
        # 1. 工具可用性验证
        tool_issues = self._validate_tools(plan)
        issues.extend(tool_issues)
        
        # 2. 依赖图验证
        dep_issues = self._validate_dependencies(plan)
        issues.extend(dep_issues)
        
        # 3. LLM 语义验证
        semantic_issues = await self._validate_semantics(plan, task)
        issues.extend(semantic_issues)
        
        # 4. 完整性检查
        completeness = await self._check_completeness(plan, task)
        
        severity = self._assess_severity(issues)
        is_valid = severity == "PASS" or severity == "WARNING"
        
        return ValidationResult(
            is_valid=is_valid,
            severity=severity,
            issues=issues,
            completeness_score=completeness.score,
            suggestions=completeness.suggestions
        )
    
    def _validate_tools(self, plan: Plan) -> list[Issue]:
        """工具可用性验证"""
        issues = []
        for step in plan.steps:
            if step.tool and step.tool not in self.tools:
                issues.append(Issue(
                    severity="ERROR",
                    step_id=step.step_id,
                    description=f"工具 '{step.tool}' 不存在",
                    suggestion=f"替换为 {' ,'.join(self.tools.keys())} 之一"
                ))
        return issues
    
    def _validate_dependencies(self, plan: Plan) -> list[Issue]:
        """依赖图验证——检查循环依赖和缺失依赖"""
        issues = []
        step_ids = {s.step_id for s in plan.steps}
        
        for step in plan.steps:
            for dep_id in step.depends_on:
                if dep_id not in step_ids:
                    issues.append(Issue(
                        severity="ERROR",
                        step_id=step.step_id,
                        description=f"依赖的步骤 {dep_id} 不存在",
                        suggestion="检查步骤 ID 是否正确"
                    ))
        
        # 检查循环依赖（简单的 DFS 实现）
        if self._has_cycle(plan):
            issues.append(Issue(
                severity="CRITICAL",
                step_id=-1,
                description="计划中存在循环依赖",
                suggestion="重新规划步骤顺序"
            ))
        
        return issues
    
    async def _validate_semantics(self, plan: Plan, task: str) -> list[Issue]:
        """LLM 语义验证——计划是否合理"""
        prompt = f"""
        验证以下计划是否合理：
        
        原始任务: {task}
        计划:
        {json.dumps(plan, indent=2)}
        
        检查:
        1. 步骤顺序是否合乎逻辑？
        2. 是否有不合理的工具使用？
        3. 是否有安全隐患？
        4. 是否有冗余步骤？
        
        输出 JSON 格式的问题列表。
        如果没问题，输出空数组。
        """
        result = await self.llm.predict(prompt)
        return [Issue(**i) for i in json.loads(result)]
```

最大挑战是**语义验证的标准模糊**："计划是否合理"是一个主观判断，不同验证者可能给出不同结论。解决方案是提供明确的验证标准（Checklist），而不是笼统地让 LLM 自由判断。

## 能力边界和结果边界

- **能力**：能检测出 70-80% 的计划缺陷（工具不存在、依赖循环、明显遗漏）
- **边界**：无法检测"计划的方向完全错误"——如果计划在方法论上就有问题但内部一致，验证可能通过
- **效果**：经过验证的计划，执行阶段出错率降低 40-60%，平均节省 2-3 次不必要的重试

## 核心优势

"先在思想中验证，再在现实中执行"——计划验证将错误捕获从执行阶段提前到设计阶段，大幅降低了试错成本。

## 工程优化方向

- 验证规则的可配置性：不同应用场景可以配置不同的验证规则和严格程度
- 增量验证：计划发生变化时，只验证受影响的部分而不是全量重新验证
- 验证缓存：相同的计划结构（任务类型相同、工具集相同）可以直接复用之前的验证结果

## 适合场景

- 执行成本高的任务（涉及付费 API、长耗时操作）
- 安全关键任务（如涉及数据库写操作、文件删除）
- 需要高度可靠性的生产级 Agent
