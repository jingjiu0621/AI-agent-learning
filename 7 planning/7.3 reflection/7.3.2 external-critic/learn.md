# 7.3.2 External Critic — 外部评审（Critic Agent）

## 简单介绍

外部评审（External Critic）是指引入一个独立的 Agent 作为"评审者"，对主 Agent 的执行过程和结果进行评价。相比自我反思，外部 Critic 提供了"第二双眼睛"——不受主 Agent 的认知偏差影响，能发现主 Agent 忽略的问题。

## 基本原理

Critic Agent 的工作模式：

1. **观察但不执行**：Critic 只阅读主 Agent 的执行轨迹，不参与实际工具调用
2. **独立评估**：基于评估标准（可以是预定义的 Checklist 或 LLM 的自由判断），对主 Agent 的表现给出评价
3. **建设性反馈**：指出问题 + 给出具体改进建议
4. **可选分级**：从简单评分到详细评审

```
主 Agent 执行:
  Thought: 我需要查询用户信息
  Action: query_database("SELECT * FROM users")
  Observation: 返回 1000 条记录...
  Thought: 数据太多，我需要筛选
  Action: query_database("SELECT * FROM users WHERE ...")
  Observation: ...

Critic 评审:
  "问题1: 使用了 SELECT *，在生产数据库中这是危险的。
   应该只查询需要的字段。
   问题2: 第一次查询返回 1000 条未分页数据，
   应该添加 LIMIT 子句。
   评分: 6/10 - 功能正确但安全性和效率需要改进"
```

## 背景

Critic Agent 的概念源自"Agent 辩论"（Agent Debate）和"多 Agent 评审"的研究方向。Du et al. (2023) 的"Improving Factuality and Reasoning in Language Models through Multiagent Debate" 和后续工作证明了多个 Agent 互相评审可以比单个 Agent 产生更准确的结果。

## 之前针对这个问题的做法与结果

1. **无评审**：Agent 的输出即最终输出。结果：简单直接，但错误无法被捕获。
2. **规则评审**：用硬编码规则检查输出（如 JSON 格式检查、长度检查）。结果：可靠但只能捕获格式错误，无法评审内容质量。
3. **LLM 自评**：同一个 Agent 自我评审。结果：有作用，但受认知偏差影响，Agent 倾向于认同自己的决策。

## 核心矛盾

**评审成本 vs 评审收益**：每次外部评审需要一次额外的 LLM 调用，增加了成本和延迟。对于简单任务，评审的成本可能超过它带来的改进收益；对于复杂高价值任务，评审的收益显著高于成本。

## 当前主流优化方向

1. **差异化评审**：简单任务不评审、中等任务抽样评审、高价值任务全量评审。评审深度与任务的重要性和复杂度匹配。
2. **专用 Critic 模型**：用较小的模型（如 GPT-4o-mini 或微调的小模型）作为 Critic，降低评审成本，同时保持可接受的评审质量。
3. **多 Critic 投票**：多个 Critic 从不同维度评审（一个关注正确性、一个关注效率、一个关注安全），取综合意见。
4. **迭代评审**：主 Agent 和 Critic 多轮交互——Critic 指出问题→主 Agent 修改→Critic 再次评审→直到通过。

## 实现的最大挑战

```python
class CriticAgent:
    def __init__(self, llm, criteria: list[str] = None):
        self.llm = llm
        self.criteria = criteria or [
            "正确性: 结果是否准确？",
            "完整性: 是否遗漏了重要信息？",
            "效率: 工具调用是否最优？",
            "安全性: 是否存在安全隐患？",
        ]
    
    async def review(self, task: str, trajectory: list[ReActStep], 
                     final_output: str) -> ReviewReport:
        prompt = f"""
        你是一个独立的 Agent 行为评审专家。
        请评审以下 Agent 的执行过程。
        
        原始任务: {task}
        
        执行轨迹:
        {self._format_trajectory(trajectory)}
        
        最终输出:
        {final_output}
        
        评审维度:
        {chr(10).join(f'- {c}' for c in self.criteria)}
        
        请按以下格式输出:
        {{
            "overall_score": float,  // 0-100
            "dimension_scores": {{"维度名": float}},
            "critical_issues": [{{"severity": "high"|"medium"|"low", "description": str, "suggestion": str}}],
            "positive_feedback": [str],
            "summary": str
        }}
        
        注意: 你只评审 Agent 的行为，不要自己执行工具或提供答案。
        """
        
        result = await self.llm.predict(prompt)
        report = ReviewReport(**json.loads(result))
        
        # 根据评审结果自动决策
        if report.overall_score < 60:
            report.decision = "REJECTED"
        elif report.overall_score < 80:
            report.decision = "NEEDS_IMPROVEMENT"
        else:
            report.decision = "APPROVED"
        
        return report
    
    async def iterative_review(self, task: str, agent, 
                               max_rounds: int = 3) -> tuple[str, ReviewReport]:
        """迭代评审—改进循环"""
        current_output = None
        for round in range(max_rounds):
            # 主 Agent 执行
            current_output = await agent.run(task)
            
            # Critic 评审
            report = await self.review(task, agent.trajectory, current_output)
            
            if report.decision == "APPROVED":
                return current_output, report
            
            # 将评审反馈注入主 Agent，让它改进
            agent.inject_feedback(report.summary)
        
        return current_output, report
```

最大挑战是**评审标准的一致性**：不同 Critic（即使是同一个 LLM 的不同调用）可能给出不一致的评审结果。同一个执行轨迹，这次批评"太保守"，下次批评"太激进"。缓解方案：提供清晰的评分 Rubric 作为评审的锚点，而不是让 LLM 自由发挥；对关键评审，用多个 Critic 投票取中位数。

## 能力边界和结果边界

- **能力**：能发现主 Agent 约 40-60% 的明显错误和次优决策
- **边界**：Critic 本身的 LLM 能力不能超过主 Agent 的 LLM——如果 GPT-4o 作为主 Agent，GPT-4o-mini 作为 Critic 可能无法发现 GPT-4o 的高级错误
- **结果**：经过 Critic 审查的 Agent 输出，在复杂任务上的准确率可提升 15-30%

## 核心优势

外部 Critic 提供了自我反思无法实现的 "外部视角"。它不受主 Agent 的执行惯性影响，能跳出主 Agent 的思维框架，发现"当局者迷"的问题。在安全敏感场景中，Critic 是不可或缺的安全层。

## 工程优化方向

- Critic 的角色专业化：不同领域使用不同的 Critic（安全 Critic、代码质量 Critic、事实性 Critic）
- 评审缓存：如果相同或高度相似的任务已被评审过，直接复用之前的评审结果
- Critic 行为监控：监控 Critic 的评审质量——如果 Critic 频繁错误拒绝好的结果，需要调整 Critic 的 Prompt

## 适合场景

- 高风险决策场景（医疗建议、金融操作、法律分析）
- 需要质量保障的生产级 Agent 系统
- 教学场景：Critic 作为导师，帮助初学者 Agent 改进
