# 7.3.1 Self-Reflection — 自我反思：回顾行为与改进

## 简单介绍

自我反思（Self-Reflection）是指 Agent 在没有外部反馈的情况下，自行回顾自己的行为轨迹，识别错误和可改进之处，并生成改进策略的能力。这是 Agent 自主进化最基础的形式。

## 基本原理

自我反思的工作流程：

1. **回顾**：Agent 审视自己刚刚完成的执行轨迹（Thought-Action-Observation 序列）
2. **分析**：识别出哪些步骤有效、哪些步骤无效，以及无效的原因
3. **抽象**：从具体错误中提炼出一般性经验（"下次遇到这种情况，我应该先确认 X 再执行 Y"）
4. **改进**：将经验转化为具体的行为调整，用于下一轮执行

```
执行轨迹:
  Thought: 搜索最新的销售数据
  Action: query_sales(period="2024")
  Observation: Error: 表不存在
  
  Thought: 表名可能叫 sales_data
  Action: query_sales_data(period="2024")  
  Observation: 返回了数据...

自我反思:
  "我直接假设了表名是 sales，但实际表名是 sales_data。
   下次查询数据库前，应该先 list_tables 确认表名。
   改进策略: 在查询任何数据库前，先用 schema 工具了解表结构。"
```

## 背景

自我反思的概念源自人类认知心理学中的"元认知"（metacognition）——对自己认知过程的认知。在 AI 领域，Reflexion 论文（Shinn et al., 2023）首次将自我反思形式化为 Agent 的学习机制，通过在 ReAct 循环后增加反思步骤，实现了"语言强化学习"——Agent 通过语言描述（而非梯度更新）来改进自身行为。

## 之前针对这个问题的做法与结果

1. **无反思**：Agent 执行完任务就直接结束，好或不好都是最终答案。结果：同样的错误会反复出现，没有学习效应。
2. **人工反馈**：由人类审查 Agent 的执行轨迹并给出改进建议。结果：效果最好但无法规模化，限制了 Agent 的自主性。
3. **简单评分**：只记录成功/失败，不分析原因。结果：数据可用但信息量不足，无法指导具体改进。

## 核心矛盾

**反思的客观性**：Agent 的自我反思受限于它自身的认知——如果 Agent 不够聪明到能正确执行任务，它可能也不够聪明到能正确分析自己为什么失败。这被称为"反思的盲区"——你无法发现你不知道的东西。

## 当前主流优化方向

1. **结构化反思框架**：引导 Agent 按固定结构反思（事实回顾 → 问题识别 → 根因分析 → 改进策略），而非自由文本反思。
2. **对比反思**：将成功轨迹和失败轨迹一起提供给 Agent，让它在对比中发现差异。对比比单看失败更能产生洞察。
3. **多视角反思**：让 Agent 从不同角色视角进行反思（"作为数据分析师怎么看？作为安全工程师怎么看？"），获得更全面的改进建议。
4. **渐进式反思**：先做快速表面的反思（耗时少），如果问题仍然存在，再做深度反思（耗时多）。

## 实现的最大挑战

```python
class SelfReflection:
    def reflect(self, trajectory: list[ReActStep], task: str, 
                outcome: TaskOutcome) -> ReflectionResult:
        trajectory_text = self._format_trajectory(trajectory)
        
        prompt = f"""
        任务: {task}
        任务结果: {"成功" if outcome.success else "失败"}
        {f"错误信息: {outcome.error}" if not outcome.success else ""}
        
        执行轨迹:
        {trajectory_text}
        
        请按以下结构进行自我反思:
        
        1. 事实回顾: 客观描述发生了什么
        2. 问题识别: 哪些步骤有问题？为什么？
        3. 根因分析: 问题的根本原因是什么？
           (知识不足？工具选择错误？参数错误？推理错误？)
        4. 改进策略: 下次应该怎么做？
        5. 可复用经验: 这个经验是否适用于更广泛的场景？
        
        输出 JSON 格式:
        {{
            "factual_review": str,
            "issues": [str],
            "root_cause": str,
            "improvement_strategies": [str],
            "reusable_experience": str
        }}
        """
        
        result = self.llm.predict(prompt)
        reflection = json.loads(result)
        
        return ReflectionResult(
            reflection=reflection,
            improved_prompt=self._generate_improved_prompt(reflection)
        )
    
    def _generate_improved_prompt(self, reflection: dict) -> str:
        """将反思结果转化为行为约束，注入到下次的 System Prompt 中"""
        experience = reflection.get("reusable_experience", "")
        strategies = reflection.get("improvement_strategies", [])
        
        improved = f"[From previous experience]\n{experience}\n"
        for s in strategies:
            improved += f"- {s}\n"
        
        return improved
```

最大挑战是**反思信号的稀疏性**：在简单任务中，Agent 可能偶然成功，反思得到的"经验"可能只是运气好而非真正学到了什么。需要区分"真正的改进"和"过拟合到单一成功案例"。缓解方案是：只有多次一致的经验才被采纳为长期策略，单次经验只作为短期提示（在当前对话有效，不持久化）。

## 能力边界和结果边界

- **能力**：可以从 60-70% 的失败案例中提取有意义的改进建议
- **边界**：无法发现 LLM 自身知识边界之外的问题（如 Agent 不知道有个更好的工具可用，反思也不会发现）
- **结果**：经过 3-5 轮反思-执行的循环，Agent 在同类任务上的成功率通常能提升 20-40%

## 核心优势

自我反思让 Agent 不再是一个"用完即弃"的一次性工具，而是一个"越用越好"的学习型系统。它不需要人类标注数据、不需要模型微调，仅通过语言反馈就实现了行为改进。

## 工程优化方向

- 反思结果的置信度评分——低置信度的反思结果只做短期参考，高置信度的固化到 Prompt 中
- 反思的触发频率自适应——初期频繁反思，积累足够经验后降低频率
- 反思结果的遗忘机制——如果某个"经验"在实践中导致更差的结果，自动撤销

## 适合场景

- Agent 需要在同一领域反复执行相似任务的场景
- 错误成本较高的场景（宁可慢一点也要更可靠）
- 无法获得人工反馈的自主运行系统
