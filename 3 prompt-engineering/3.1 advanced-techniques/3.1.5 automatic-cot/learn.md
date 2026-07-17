# Auto-CoT — 自动思维链提示

## 简单介绍

Auto-CoT（Automatic Chain-of-Thought）解决了 Few-shot CoT 最实际的痛点：**手动编写 CoT 示例既费时又费力，且难以覆盖多样化的问题类型。** Auto-CoT 的目标是让 LLM 自动生成高质量的思维链示例，从而完全自动化 CoT 的准备工作。

其核心流程是：从问题集中采样有代表性的问题 → 让 LLM 为这些问题生成推理链 → 用自动生成的（问题、推理链、答案）三元组作为 Few-shot CoT 的示例。整个过程无需人工介入。对于 Agent 开发来说，这意味着可以在运行时动态生成针对特定领域或任务的 CoT 示例。

## 基本原理

Auto-CoT 分为两个主要阶段：

**阶段一：问题采样与聚类**
1. 将全部问题（或大规模问题候选集）按照问题类型、难度、主题等进行聚类
2. 从每个聚类中选取若干代表性问题作为"示例候选"
3. 这样做的目的：用最少量的示例覆盖最多样化的问题类型

**阶段二：推理链自动生成**
1. 对每个选中的候选问题，使用 Zero-shot CoT（"Let's think step by step"）让 LLM 生成推理链
2. 对生成结果进行简单验证（如确保推理链包含最终答案）
3. 将验证通过的（问题、推理链、答案）作为 Few-shot CoT 的示例

```python
class AutoCoT:
    """自动思维链：自动生成 CoT 示例"""

    def __init__(self, llm_client):
        self.llm = llm_client
        self.exemplars = []

    def build_exemplars(self, questions: list[str], n_clusters: int = 5,
                        samples_per_cluster: int = 2):
        """
        从问题集中自动构建 CoT 示例
        """
        # Step 1: 问题聚类（使用 Embedding + K-Means）
        question_vectors = [self._embed(q) for q in questions]

        from sklearn.cluster import KMeans
        kmeans = KMeans(n_clusters=n_clusters)
        clusters = kmeans.fit_predict(question_vectors)

        # Step 2: 从每个聚类中选择最接近聚类中心的问题
        selected_indices = []
        for c in range(n_clusters):
            cluster_indices = [i for i, label in enumerate(clusters) if label == c]
            cluster_center = kmeans.cluster_centers_[c]

            # 按距离中心排序，选最近的 samples_per_cluster 个
            sorted_idx = sorted(
                cluster_indices,
                key=lambda i: self._distance(question_vectors[i], cluster_center)
            )
            selected_indices.extend(sorted_idx[:samples_per_cluster])

        # Step 3: 为选中的问题自动生成推理链
        self.exemplars = []
        for idx in selected_indices:
            question = questions[idx]
            # 使用 Zero-shot CoT 生成推理链
            cot_result = self.llm.generate(
                f"{question}\n\nLet's think step by step."
            )
            # 验证：至少包含最终答案
            answer = self._extract_answer(cot_result)
            if answer:
                self.exemplars.append({
                    "question": question,
                    "cot": cot_result,
                    "answer": answer
                })

        return self.exemplars

    def generate_with_auto_cot(self, question: str) -> str:
        """使用自动生成的 CoT 示例回答问题"""
        if not self.exemplars:
            raise RuntimeError("请先调用 build_exemplars()")

        prompt = ""
        for ex in self.exemplars:
            prompt += f"Question: {ex['question']}\n"
            prompt += f"Thought: {ex['cot']}\n"
            prompt += f"Answer: {ex['answer']}\n\n"

        prompt += f"Question: {question}\nThought:"
        return self.llm.generate(prompt)
```

## 背景

Auto-CoT 由来自 University of California, Los Angeles (UCLA) 等机构的研究者在 2022 年的论文 "Automatic Chain of Thought Prompting in Large Language Models" 中提出。

Few-shot CoT 虽然在推理任务上效果出色，但它有一个严重的实际瓶颈：**示例工程成本极高**。高质量的 CoT 示例需要：
- 针对特定领域的问题理解
- 编写清晰正确的推理步骤
- 人工验证每个示例的正确性
- 当问题分布变化时重新编写

在 Agent 系统中，Agent 可能面临各种不同的任务域——今天的 Agent 处理代码问题，明天处理财务问题，后天处理医疗问题。为每个域手动准备 CoT 示例是不现实的。Auto-CoT 让 Agent 能够从少量用户问题或历史交互中自举（bootstrapping）出高质量的推理示例。

## 之前针对这个问题的做法与结果

| 做法 | 描述 | 效果与局限 |
|------|------|-----------|
| 手动编写 CoT 示例（Few-shot CoT） | 人工为每个任务编写推理链示例 | 效果最好但成本最高。需要领域专家 + Prompt 工程师协作。难以扩展到多领域 |
| Zero-shot CoT | 直接加 "Let's think step by step" | 零成本但效果不稳定。某些任务上比 Few-shot 差 10-20% |
| 固定示例库 | 预先准备固定的示例集合，所有问题用相同示例 | 示例与问题的匹配度差。当问题类型多样时，少数固定示例无法覆盖 |

## 核心矛盾

| 矛盾维度 | 手动 CoT 示例 | 自动 CoT 示例 |
|---------|-------------|--------------|
| 质量 | 高（人工精心编写） | 中等（可能包含推理错误） |
| 覆盖度 | 窄（受限于人力） | 宽（可大规模生成） |
| 成本 | 高（人力成本） | 低（计算成本） |
| 可扩展性 | 差 | 好 |
| 领域适配 | 需要领域专家 | 零适配成本 |
| 示例多样性 | 有限 | 可覆盖多种问题类型 |

## 当前主流优化方向

1. **示例质量过滤**——自动生成的推理链可能包含错误。当前优化方向包括：用 LLM 验证 LLM（让另一个 LLM 验证推理链的正确性）、用执行结果验证（当推理涉及代码或公式时实际执行验证）、用 Self-Consistency 投票筛选高置信度推理链
2. **动态示例选择**——不再是"对所有问题使用同样的示例"，而是从自动构建的示例库中检索与当前问题最相似的 3-5 个示例（Few-shot 的示例数量），小模型可能需要更多示例，大模型可能需要更少。使用向量相似度检索
3. **错误驱动的示例迭代**——当 LLM 用当前示例回答错误时，自动将该错误案例补充到示例库中，形成持续改进的闭环
4. **少样本自适应**——根据目标模型的大小和任务难度，自动调整示例数量：小模型需要更多示例（8-16 个），大模型可能 2-3 个就够

## 实现的最大挑战

1. **生成示例的质量控制**——LLM 自动生成的推理链可能包含错误或幻觉。如果用有问题的示例来引导后续推理，错误会被放大。如何自动检测和过滤低质量示例是最核心的挑战
2. **聚类效果的依赖**——Auto-CoT 的效果高度依赖聚类质量。如果聚类不能有效区分问题类型，生成的示例集可能冗余或有偏差。而 Embedding 的质量和聚类算法的选择直接影响结果
3. **"垃圾进垃圾出"循环**——如果生成示例的 LLM 本身能力有限，生成的示例质量差，用这些差示例引导推理又得到差结果，形成负反馈循环

## 能力边界与结果边界

**能做什么：**
- 替代手动 CoT 示例编写，大幅降低 Prompt 工程的人力成本
- 动态适应不同领域和问题类型
- 在海量问题场景下自动构建高质量的推理示例库
- 作为持续学习的闭环组件：从错误中自动学习并改进

**不能做什么：**
- 不能超过生成示例的 LLM 的能力上限（Auto-CoT 通常用 GPT-4 或同等能力模型生成示例用于较弱模型）
- 在高度专业的领域（如医学诊断、法律推理），自动生成的示例质量可能不如领域专家
- 不能解决"LLM 在推理链中产生幻觉"的底层问题

**产生的结果：**
- 自动构建的（问题、推理链、答案）示例库
- 每次推理时动态检索的最优示例集
- 示例质量的评估指标（通过验证率统计）

## 与其他技术的区别

| 对比技术 | 核心区别 |
|---------|---------|
| Zero-shot CoT | Auto-CoT 用自动生成的示例进行 Few-shot 引导；Zero-shot CoT 无示例 |
| Manual Few-shot CoT | Auto-CoT 示例生成自动完成；Manual 需要人工编写 |
| Self-Consistency | Auto-CoT 关注"示例的自动生成"；SC 关注"多次推理的答案聚合" |
| Active Learning | Auto-CoT 在问题空间中主动采样代表性示例，与 Active Learning 思想相通 |

## 核心优势

1. **零人力成本**——从问题集到 CoT 示例的全自动流水线，无需 Prompt 工程师介入
2. **动态适配**——Agent 每遇到新的任务域，可以即时生成该域的 CoT 示例，无需预先准备
3. **持续优化能力**——Agent 系统运行时间越长，积累的示例越多，推理质量持续提升
4. **跨模型迁移**——用强模型（如 GPT-4）生成的 CoT 示例可以服务于弱模型（如开源模型），实现"能力蒸馏"

## 工程优化方向

1. **混合示例策略**——以自动生成示例为主，但允许人工标注少量"黄金示例"作为基础种子（seed exemplars），保证基础质量
2. **示例质量评分模型**——训练一个轻量级分类器（如基于 BERT 的评分模型）来评估自动生成的推理链质量，替代昂贵的 LLM 验证
3. **增量式示例库更新**——使用向量数据库存储示例，每次新问题通过语义检索匹配最相关示例；当检索结果不够好时触发新示例的自动生成

## 适合场景的判断标准

**应该使用 Auto-CoT 的场景：**
- 需要覆盖大量不同领域的推理任务
- 缺乏人力编写 CoT 示例
- 问题类型持续变化或新增
- 有强模型可供生成示例（或可以调用 API）

**不应使用 Auto-CoT 的场景：**
- 任务域非常固定，手动编写的少量示例就能完美覆盖
- 生成质量无法验证（完全无事实依据的创意任务）
- 问题量极少（例如总共有 10 个问题，手动编写更快）
