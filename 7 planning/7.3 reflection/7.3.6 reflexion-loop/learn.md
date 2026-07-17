# 7.3.6 Reflexion Loop — Reflexion 完整循环实现

## 简单介绍

Reflexion（反射）是 ReAct 的增强版——在标准 ReAct 循环结束后增加一个反思步骤，将反思结果作为下一轮尝试的输入。这种"执行→反思→改进→再执行"的闭环实现了语言层面的强化学习。

## 基本原理

Reflexion 的核心架构包含三个角色：

1. **Actor（执行者）**：标准的 ReAct Agent，负责执行任务
2. **Evaluator（评估者）**：判断任务是否成功完成，生成反馈信号
3. **Reflector（反思者）**：分析执行轨迹，生成改进策略

```
         任务目标
            ↓
    ┌───────────────────┐
    │   Actor (ReAct)    │──── 执行轨迹 ────→┐
    └───────────────────┘                    │
            ↑                                ↓
    ┌───────────────────┐           ┌───────────────────┐
    │  下次提升建议      │◄──────────│   Evaluator       │
    │                    │           │   (评估结果)       │
    └───────────────────┘           └───────────────────┘
            ↑                                ↓
    ┌───────────────────┐           ┌───────────────────┐
    │  改进策略注入      │◄──────────│   Reflector       │
    │                    │           │   (反思分析)       │
    └───────────────────┘           └───────────────────┘
```

## 背景

Reflexion 论文（Shinn et al., 2023）在决策任务（AlfWorld）和推理任务（HotpotQA）上展示了显著效果——在 AlfWorld 上，Reflexion 将成功率从 73% 提升到 97%。关键创新在于"语言强化学习"：不需要微调模型，仅通过语言反馈就让 Agent 持续改进。

## 之前针对这个问题的做法与结果

1. **单次 ReAct**：Agent 执行一次，无论成功失败都输出结果。结果：简单直接，但遇到复杂任务容易失败。
2. **手动重试**：如果失败，人类手动分析原因并修改 Prompt 重试。结果：有效但不可自动化。
3. **简单重试**：失败后直接重新运行，不分析原因。结果：如果 Agent 犯的是系统性错误，重试 n 次会犯同样的错误 n 次。

## 核心矛盾

**Reflexion 的改进幅度 vs 成本**：每轮 Reflexion 需要多次 LLM 调用（执行 + 评估 + 反思），成本是单次 ReAct 的 3-5 倍。不是所有任务都值得用 Reflexion——对简单任务，单次 ReAct 就足够。

## 实现的最大挑战

```python
class ReflexionAgent:
    def __init__(self, llm, max_trials: int = 3):
        self.llm = llm
        self.max_trials = max_trials
        self.actor = ReActAgent(llm)
        self.memory = []  # 反思记忆
    
    async def run(self, task: str) -> str:
        for trial in range(1, self.max_trials + 1):
            print(f"\n{'='*40}\nTrial {trial}/{self.max_trials}\n{'='*40}")
            
            # Phase 1: Actor 执行
            result = await self.actor.run(task)
            trajectory = self.actor.get_trajectory()
            
            # Phase 2: Evaluator 评估
            evaluation = await self._evaluate(task, result, trajectory)
            
            if evaluation["success"]:
                print(f"✓ 任务在第 {trial} 轮成功完成")
                return self._format_final(trial, result, self.memory)
            
            # Phase 3: Reflector 反思
            reflection = await self._reflect(task, trajectory, evaluation)
            self.memory.append(reflection)
            
            # Phase 4: 改进策略注入 Actor
            self._inject_feedback(reflection, trial)
        
        return self._format_final("failed", self.memory)
    
    async def _evaluate(self, task: str, result: str, 
                        trajectory: list) -> dict:
        """评估执行结果"""
        prompt = f"""
        任务: {task}
        执行结果: {result}
        执行轨迹: {trajectory}
        
        任务是否成功完成？考虑以下标准：
        1. 最终答案是否正确/完整？
        2. 是否有不必要的步骤？
        3. 是否有安全隐患？
        
        输出 JSON:
        {{
            "success": bool,
            "score": int,
            "reasoning": str,
            "key_issues": [str]
        }}
        """
        return json.loads(await self.llm.predict(prompt))
    
    async def _reflect(self, task: str, trajectory: list,
                       evaluation: dict) -> dict:
        """生成反思和改进策略"""
        prompt = f"""
        你是一个反射 Agent。分析以下执行轨迹和评估结果，
        生成具体的改进策略供下次尝试使用。
        
        任务: {task}
        执行轨迹: {trajectory}
        评估: {evaluation}
        
        历史反思（已尝试过的改进）:
        {self.memory}
        
        注意: 不要重复之前已经尝试过的改策略。
        
        输出 JSON:
        {{
            "what_went_wrong": str,
            "root_cause": str,
            "concrete_fixes": [str],
            "modified_plan": str  
        }}
        """
        return json.loads(await self.llm.predict(prompt))
    
    def _inject_feedback(self, reflection: dict, trial: int):
        """将反思结果注入到 Actor 的上下文中"""
        feedback = f"""
[Reflexion Feedback - Trial {trial}]
Previous attempt issues:
{reflection['what_went_wrong']}

Root cause:
{reflection['root_cause']}

Concrete improvements for this attempt:
{chr(10).join(f'- {fix}' for fix in reflection['concrete_fixes'])}

Modified plan:
{reflection['modified_plan']}

Please apply these improvements in your execution.
"""
        self.actor.inject_context(feedback)
```

最大挑战是**反思的边际效益递减**：第一轮反思通常能带来最大的改进，第二轮改进幅度减小，第三轮以后往往改进很小甚至原地踏步。需要检测何时停止反思循环——如果连续两轮反思产生的策略没有实质区别，就应该停止。

## 能力边界和结果边界

- **能力**：在决策任务上，3 轮 Reflexion 通常能将成功率提升 20-40%
- **边界**：Reflexion 不能解决 Agent 根本能力不足的问题（如 LLM 本身不具备某个领域的知识）；对已经高成功率的任务，Reflexion 的改进空间有限
- **成本分析**：一轮 Reflexion 约消耗 3-5 次 LLM 调用的 token。如果单次成功率为 60%，3 轮 Reflexion 后累积成功率为 ~94%，成本约是单次的 3-5 倍

## 核心优势

Reflexion 是目前最实用的 Agent 自我改进方法——不需要微调、不需要 RLHF、不需要人类标注数据，仅通过"语言反馈"就实现了可量化的性能提升。它是"Agent 自我进化"最直接的工程实现。

## 与其他模式的区别

| 维度 | ReAct | Reflexion | Plan-and-Execute |
|------|-------|-----------|------------------|
| 改进机制 | 无 | 事后反思 | 执行中调整 |
| 错误处理 | 当前步重试 | 下轮全局改进 | 局部调整计划 |
| 状态持久化 | 无 | 跨轮记忆 | 计划状态 |
| 适用场景 | 简单到中等任务 | 需多轮尝试的困难任务 | 有明确阶段的任务 |

## 工程优化方向

- 自适应轮数：根据任务复杂度和首次执行分数，动态决定 Reflexion 轮数
- 并行试错：如果资源允许，同时执行多轮 Reflexion（不同策略），选最优结果
- 反思结果的结构化输出：定义标准 schema，方便与储存系统集成
- 与记忆系统集成：跨任务的反思经验共享，避免每新任务从零开始反思

## 适合场景

- 复杂推理任务（多跳 QA、代码生成、数据分析）
- 决策类任务（AlfWorld、WebShop 等交互环境）
- 需要高可靠性的生产 Agent（多轮自我修正确保质量）
