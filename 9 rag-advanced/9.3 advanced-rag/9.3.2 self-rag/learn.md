# 9.3.2 Self-RAG — 自我反思式检索

## 概述

Self-RAG（Self-Reflective Retrieval-Augmented Generation）是由 Asai 等人于 2023 年提出、发表于 ICLR 2024 的一种新型 RAG 范式。其核心思想是：**让 LLM 在生成过程中自主决定是否需要检索、检索结果是否相关、以及是否应该使用检索结果**。与传统的"每轮必检"不同，Self-RAG 通过训练模型输出特殊的 **Reflection Token**（反思令牌），在生成过程中动态控制检索行为和生成质量。

## 背景与核心矛盾

### 传统 RAG 的缺陷

| 问题 | 表现 | 后果 |
|------|------|------|
| 无差别检索 | 每轮输入都检索固定数量文档 | 简单问题浪费 Token，增加延迟 |
| 检索噪声污染 | 不相关文档混入上下文 | 误导 LLM 产生幻觉或错误回答 |
| 无法自我纠正 | 生成后不验证检索质量 | 错误无法被发现和修正 |
| 缺乏细粒度控制 | 对每个 Token/片段的生成不加区分 | 已掌握的知识也依赖外部检索 |

### 核心矛盾

**"何时检索"和"是否信任检索结果"这两个关键决策完全由外部规则决定，而非 LLM 自身。** 传统 RAG 用固定的 Top-K 检索策略，假设每个查询都需要相同数量的外部知识——这明显不合理。简单的事实性问题（如"2+2=?"）不需要检索，而复杂的问题（如"2025 年诺贝尔物理学奖得主是谁？"）需要检索，且检索结果需要被验证。

## Self-RAG 核心原理

### 1. 反思令牌（Reflection Tokens）

Self-RAG 的核心创新是在模型词汇表中引入**特殊令牌**，这些令牌在训练过程中被学习，在推理时由模型自动生成，用于控制检索和生成行为：

```
[Retrieve] / [NoRetrieve]       → 是否检索
[IsRel] / [IsNotRel]            → 检索结果是否相关
[IsSup] / [IsNotSup]            → 检索结果是否支持生成内容
[IsUseful] / [NotUseful]        → 检索结果对生成是否有帮助
```

### 2. 分段生成机制

Self-RAG 的生成过程是**段落级别**的，而非整个序列：

```
输入: "2024年巴黎奥运会什么时候开幕？"

Step 1: 模型评估是否需要检索
  → 输出 [Retrieve] → 执行检索

Step 2: 检索到相关文档后
  → 检查相关性 → 输出 [IsRel]（相关）
  → 检查支持性 → 输出 [IsSup]（支持）

Step 3: 生成段落
  → "2024年巴黎奥运会于7月26日开幕。" 
  → 输出 [IsUseful]（有用）

Step 4: 继续下一段落或结束
  → 输出 [NoRetrieve] → 直接生成结束令牌
```

### 3. 训练方法

Self-RAG 的训练分为三个阶段：

**阶段 1：数据构建**
- 使用 Critic Model 对检索结果进行标注
- 为每个段落生成 Reflection Token 标签
- 构建 (query, passage, reflection_label) 三元组

**阶段 2：模型微调**
- 在标准语言模型基础上进行指令微调
- 训练模型在合适的位置生成 Reflection Token
- 使用插入式训练（interleaved training）

**阶段 3：推理策略**
- 在推理时，模型先生成 Reflection Token
- 根据 Token 决定是否检索
- 对检索结果进行验证后再生成实际内容

## 架构全景图

```
                                   ┌─────────────────────┐
                                   │    用户查询 (Query)   │
                                   └──────────┬──────────┘
                                              │
                                   ┌──────────▼──────────┐
                                   │   Self-RAG 主模型    │
                                   │  (微调后的 LLM)      │
                                   └──────────┬──────────┘
                                              │
                                 ┌────────────▼────────────┐
                                 │     Step 1: 检索决策     │
                                 │  [Retrieve]/[NoRetrieve] │
                                 └────────────┬────────────┘
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │ [NoRetrieve]            │ [Retrieve]              │
                    ▼                         ▼                         │
          ┌──────────────────┐     ┌──────────────────┐                │
          │ 直接生成（无检索） │     │   调用检索器      │                │
          │ LLM 内部知识回答   │     │   Vector/BM25    │                │
          └──────────────────┘     └────────┬─────────┘                │
                                            │                          │
                                 ┌──────────▼──────────┐              │
                                 │  Step 2: 相关性判定  │              │
                                 │ [IsRel]/[IsNotRel]  │              │
                                 └──────────┬──────────┘              │
                                            │                          │
                    ┌───────────────────────┼─────────────────┐        │
                    │ [IsNotRel]            │ [IsRel]         │        │
                    ▼                       ▼                 │        │
          ┌──────────────────┐   ┌──────────────────┐        │        │
          │ 丢弃检索结果      │   │ Step 3: 支持性判定 │        │        │
          │ 降级或重新检索     │   │ [IsSup]/[IsNotSup]│        │        │
          └──────────────────┘   └────────┬─────────┘        │        │
                                         │                   │        │
                    ┌────────────────────┼────────────┐      │        │
                    │ [IsNotSup]         │ [IsSup]    │      │        │
                    ▼                    ▼            │      │        │
          ┌──────────────────┐  ┌──────────────────┐ │      │        │
          │ 修正生成内容      │  │ 保持生成内容      │ │      │        │
          │ 重新表述          │  │                │ │      │        │
          └──────────────────┘  └────────┬─────────┘ │      │        │
                                         │           │      │        │
                                         ▼           ▼      ▼        │
                              ┌──────────────────────────┐          │
                              │ Step 4: 有用性判定        │          │
                              │ [IsUseful]/[NotUseful]    │          │
                              └──────────┬───────────────┘          │
                                         │                          │
                                         ▼                          │
                              ┌──────────────────────┐              │
                              │   最终答案（分段输出）  │              │
                              └──────────────────────┘              │
                                                                     │
                              ←────── 循环下一段或结束 ──────────────┘
```

## 推理流程伪代码

```python
class SelfRAG:
    def __init__(self, model, retriever, reflection_tokens):
        """
        model: 微调后的 Self-RAG 语言模型
        retriever: 外部检索器
        reflection_tokens: {Retrieve, NoRetrieve, IsRel, IsNotRel, IsSup, IsNotSup, IsUseful, NotUseful}
        """
        self.model = model
        self.retriever = retriever
        self.tokens = reflection_tokens

    def generate(self, query: str, max_segments: int = 5):
        context = []
        final_answer = []

        for _ in range(max_segments):
            # 准备当前状态
            prompt = self._build_prompt(query, context)

            # Step 1: 生成 Reflection Token — 决定是否检索
            next_token = self.model.generate_next_token(prompt)
            
            if next_token == self.tokens['Retrieve']:
                # 执行检索
                docs = self.retriever.retrieve(query, k=3)
                
                # Step 2: 判断检索结果相关性
                rel_score = self.model.evaluate_relevance(query, docs)
                if rel_score < REL_THRESHOLD:
                    # 检索结果不相关 → 标记为 [IsNotRel]，丢弃结果
                    context.append({"role": "system", "content": "[IsNotRel] 检索结果不相关，丢弃"})
                    continue  # 重新尝试
                
                context.append({"role": "system", "content": "[IsRel] 检索结果相关"})
                context.extend(self._doc_to_context(docs))
                
                # Step 3: 生成段落
                segment = self.model.generate_segment(prompt, context)
                
                # Step 4: 验证生成内容是否被检索支持
                sup_score = self.model.evaluate_support(segment, docs)
                if sup_score < SUP_THRESHOLD:
                    segment = self.model.revise_segment(segment, docs)  # 修正
                    context.append({"role": "system", "content": f"[IsNotSup] 修正后: {segment}"})
                else:
                    context.append({"role": "system", "content": f"[IsSup] 支持: {segment}"})
                
                final_answer.append(segment)

            else:  # [NoRetrieve]
                # 无需检索，直接生成（依赖模型内部知识）
                segment = self.model.generate_segment(prompt, [])
                final_answer.append(segment)
                
                # 检查是否应结束
                if self.model.is_end_of_response(segment):
                    break

        return " ".join(final_answer)

    def _build_prompt(self, query, context):
        """构建包含历史上下文的 prompt"""
        prompt = [
            {"role": "user", "content": f"Question: {query}"},
            {"role": "assistant", "content": "I'll answer step by step, using [Retrieve] when needed."}
        ]
        for ctx in context:
            prompt.append(ctx)
        return prompt

    def _doc_to_context(self, docs):
        return [{"role": "system", "content": f"[Document]: {d}"} for d in docs]
```

## 与 Agentic RAG 的对比

| 维度 | Self-RAG | Agentic RAG |
|------|----------|-------------|
| **实现方式** | 模型微调（Fine-tuned） | 架构模式（Prompt-based） |
| **检索决策** | 模型内置 Reflection Token | LLM 思考决定（Tool Use） |
| **灵活性** | 受限——Token 类型固定 | 高——可自定义工具和策略 |
| **泛化能力** | 受限——需微调专有模型 | 高——通用 LLM 即可 |
| **推理成本** | 较低——单次生成 | 较高——多轮思考-行动循环 |
| **延迟** | 低——无多轮交互 | 较高——可能多步迭代 |
| **可控性** | 好——Token 级别的细粒度控制 | 中等——Prompt 级别的控制 |
| **工程复杂度** | 高——需要训练和部署专有模型 | 低——Prompt 工程即可 |

## 优势与局限

### 核心优势

1. **细粒度控制**：Reflection Token 让模型在每个段落级别自主决策，而非整个回答
2. **检索效率**：避免不必要的检索，节省 Token 和延迟
3. **质量保证**：IsRel/IsSup/IsUseful 三级验证机制确保生成质量
4. **可解释性**：Reflection Token 序列就是推理轨迹，可追踪可审计

### 能力边界

1. **模型依赖**：需要微调专用模型，不能即插即用
2. **Token 固定**：Reflection Token 类型和数量是固定的，不能动态扩展
3. **段落粒度**：粒度限制为段落级别，更细粒度的 Token 级别控制不可行
4. **泛化限制**：在训练数据覆盖外的领域表现下降
5. **多语言支持**：主要在英文上训练，中文等语言需要额外适配

### 最大挑战

**训练数据的质量直接决定 Self-RAG 的上限。** Reflection Token 的标注需要 Critic Model 生成高质量标签，而 Critic Model 本身也有误判风险。此外，训练需要在检索决策、相关性判断、支持性验证三个维度同时优化，多任务学习的平衡极具挑战。

## 2025-2026 前沿进展

1. **Agentic Self-RAG**：将 Self-RAG 的反思机制与多 Agent 协同结合，用多个专用 Agent 分别负责检索决策、相关性判断和生成验证
2. **Self-RAG + Tool Use**：扩展 Reflection Token 类型以支持更多工具（计算器、代码执行、数据库查询）
3. **轻量级 Self-RAG**：通过知识蒸馏，将 Self-RAG 能力迁移到更小的模型
4. **跨语言 Self-RAG**：扩展到多语言场景，支持更多语言的 Reflection Token 训练
5. **自适应阈值**：从固定阈值（REL_THRESHOLD）进化到动态阈值，根据查询复杂度自适应调整

## 适用场景

| 场景 | 推荐度 | 原因 |
|------|--------|------|
| 高精度知识问答 | ⭐⭐⭐⭐⭐ | 三级验证机制确保答案可靠 |
| 开放域对话 | ⭐⭐⭐⭐ | 按需检索避免过度检索 |
| 企业知识库查询 | ⭐⭐⭐⭐ | 需要答案可追溯和可验证 |
| 实时对话系统 | ⭐⭐⭐ | 延迟低但需要部署专有模型 |
| 多语言场景 | ⭐⭐ | 需要额外训练和适配 |
| 快速原型验证 | ⭐ | 需要模型微调，不适合快速迭代 |

## 工程优化方向

1. **Reflection Token 级联策略**：设计优先级规则，减少不必要的完整评估链
2. **延迟检索**：仅在 IsRel 阶段通过后才加载完整文档内容（先加载摘要）
3. **批量验证**：多个段落的 IsRel/IsSup 判断可以并行执行
4. **混合部署**：简单查询用 Self-RAG，复杂查询升级到 Agentic RAG
5. **Reflection Token 压缩**：在上下文窗口中压缩历史的 Reflection Token 轨迹
