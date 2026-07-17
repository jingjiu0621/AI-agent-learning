# 7.3.5 Reflection Storage — 反思结果存储与复用

## 简单介绍

反思结果如果只是用完即弃，Agent 就只能在当前对话中改进，无法在跨会话、跨任务中持续进化。反思存储（Reflection Storage）是将反思得到的经验持久化，实现"一次反思，永远受益"的关键基础设施。

## 基本原理

反思存储的核心架构：

```
反思结果 → 结构化存储 → 检索匹配 → 复用注入
   ↑                              |
   └────────── 反馈循环 ←─────────┘
```

1. **结构化**：将自由文本的反思结果转为结构化的经验记录
2. **存储**：存入向量数据库或关系型数据库
3. **检索**：新任务到来时，检索相关历史经验
4. **复用**：将匹配的经验注入到 Agent 的 System Prompt 或上下文
5. **反馈**：评估复用效果，强化/弱化该经验的权重

## 背景

Reflexion 论文中的反思结果是存储在 episodic memory buffer 中、仅在当前 episode 内使用的。Generative Agents 论文（Park et al., 2023）展示了将反思结果持久化到记忆流（Memory Stream）的效果——Agent 在 2 天后还能回忆起 3 天前的反思经验。这启发了反思存储的工程化设计。

## 之前针对这个问题的做法与结果

1. **对话内存储**：反思结果仅保留在当前对话的上下文窗口中。结果：简单，但跨对话无法复用。
2. **固定文件存储**：将反思结果追加到一个文本文件中。结果：可跨对话，但检索困难——100 条经验后很难找到需要的那条。
3. **向量存储检索**：将反思结果向量化存入向量数据库，按语义相似度检索。结果：检索效率高，但存在"检索到不相关经验"的噪音问题。

## 核心矛盾

**经验的通用性 vs 特异性**：过于通用的经验（"仔细检查参数"）适用范围广但价值低，过于特异的经验（"当使用 Redshift 且查询涉及分区表时需要检查 distkey"）价值高但适用范围窄。好的反思存储需要在两者之间找到平衡。

## 当前主流优化方向

1. **经验分层**：将经验分为三层——通用原则（适用于所有任务）、领域规则（适用于某类任务）、特定教训（适用于特定工具或场景），检索时按优先级排序。
2. **经验-任务的匹配度评分**：检索到的经验并非全盘接受，而是由 LLM 判断"这个经验在当前的 context 下是否适用"。
3. **经验衰减与强化**：每次经验被成功复用且产生正面效果，其"信任分数"增加；如果经验导致更差的结果，信任分数降低。信任分数低的经验最终被淘汰。
4. **经验自动泛化**：当积累到多条类似的特异性经验时，LLM 自动将其泛化为通用原则。

## 实现的最大挑战

```python
class ExperienceStore:
    def __init__(self, vector_db=None):
        self.db = vector_db or InMemoryVectorDB()
        self.experiences = []
    
    async def store_reflection(self, reflection: ReflectionResult):
        """存储反思结果"""
        # 生成经验记录
        experience = {
            "id": str(uuid.uuid4()),
            "task_type": reflection.task_type,
            "tools_involved": reflection.tools_involved,
            "error_pattern": reflection.root_cause,
            "improvement": reflection.improvement_strategies,
            "reusable_experience": reflection.reusable_experience,
            "abstract_principle": self._abstract_principle(reflection),
            "success_count": 0,
            "fail_count": 0,
            "trust_score": 0.5,
            "created_at": time.time()
        }
        
        # 向量化存储
        embedding = await self._embed(experience["abstract_principle"])
        self.db.insert(experience, embedding)
        
        return experience["id"]
    
    async def retrieve_relevant(self, task: str, context: dict, 
                                 top_k: int = 3) -> list[Experience]:
        """检索与当前任务相关的历史经验"""
        query = f"{task} {context.get('tools', [])} {context.get('domain', '')}"
        query_embedding = await self._embed(query)
        
        candidates = self.db.similarity_search(query_embedding, k=top_k * 2)
        
        # 重排序 + 信任分数加权
        scored = []
        for exp in candidates:
            relevance = await self._judge_relevance(task, exp)
            score = relevance * exp["trust_score"]
            scored.append((score, exp))
        
        scored.sort(reverse=True)
        return [exp for _, exp in scored[:top_k] if _ > 0.3]
    
    async def apply_experience(self, task: str, agent_prompt: str) -> str:
        """将相关经验注入到 Agent Prompt 中"""
        experiences = await self.retrieve_relevant(task, {})
        if not experiences:
            return agent_prompt
        
        experience_section = "\n\n## Previous Experience Notes\n"
        for exp in experiences:
            experience_section += f"- {exp['abstract_principle']}\n"
            if exp['trust_score'] > 0.7:
                experience_section += f"  Details: {exp['reusable_experience']}\n"
        
        return agent_prompt + experience_section
    
    def update_trust(self, experience_id: str, was_helpful: bool):
        """更新经验的信任分数（强化学习）"""
        exp = self.db.get(experience_id)
        if was_helpful:
            exp["success_count"] += 1
            exp["trust_score"] = min(1.0, exp["trust_score"] + 0.1)
        else:
            exp["fail_count"] += 1
            exp["trust_score"] = max(0.0, exp["trust_score"] - 0.15)
        self.db.update(experience_id, exp)
```

最大挑战是**经验复用时的上下文污染**：注入过多历史经验到 Agent 的 Prompt 中，可能会干扰 Agent 对当前任务的专注。需要严格控制注入的经验数量（通常 1-3 条），且经验以"提示"而非"指令"的形式出现。

## 能力边界和结果边界

- **能力**：可以让 Agent 在跨对话中积累经验，形成持续的学习曲线
- **边界**：经验的质量取决于初始反思的质量——低质量反思→低质量经验→低质量复用
- **结果**：经过 50-100 次经验积累，Agent 在常见任务上的错误率通常能降低 30-50%

## 核心优势

反思存储将 Agent 从"单次任务执行者"升级为"持续学习系统"。每个任务都是学习机会，每次反思都是知识积累，每个经验都让后续任务受益。

## 工程优化方向

- 定期对存储的经验做"压缩和泛化"——合并相似经验，淘汰过时或低信任度的经验
- 经验版本控制：如果某个经验因为工具升级等原因变得不再适用，可以标记为 deprecated
- 共享经验库：多 Agent 实例共享同一个经验库，一个 Agent 的教训所有 Agent 受益
- 经验导出能力：将高质量经验导出为可嵌入的 Prompt 模板或规则

## 适合场景

- 需要长期运行的 Agent 系统（客服、运维、个人助理）
- 同一 Agent 需要处理大量相似任务的场景
- 多 Agent 团队的集体经验管理
