# 9.7.2 generation-metrics — 生成评估指标

## 简单介绍

生成评估指标是衡量 RAG 系统生成的回答质量的量化标准。与检索评估（衡量"检索到的文档好不好"）不同，生成评估关注的是**"基于检索结果生成的回答好不好"**——是否正确、是否忠于检索结果、是否完整回答了用户问题。

## 三大核心生成指标

### 1. Faithfulness（忠实度/归因）

**定义**：生成的回答是否忠于检索到的文档，不包含检索结果中不支持的信息。

这是 RAG 生成评估中最重要的指标——因为 RAG 的核心优势就是"基于外部知识"，如果 LLM 生成了检索结果中没有的信息（即幻觉），那就丧失了 RAG 的意义。

```python
def evaluate_faithfulness(generated_answer: str, 
                          retrieved_contexts: list[str]) -> float:
    """评估生成回答是否忠实于检索文档"""
    
    # 方法：将回答拆分为声明（claims），逐一验证是否被检索文档支持
    
    # Step 1: 从回答中提取可验证的声明
    claims = extract_claims(generated_answer)
    # ["Transformer 使用自注意力机制",
    #  "Transformer 在 2017 年被提出",
    #  "Transformer 不需要 RNN"]
    
    # Step 2: 验证每个声明是否被检索文档支持
    supported = 0
    for claim in claims:
        is_supported = check_claim_against_contexts(claim, retrieved_contexts)
        if is_supported:
            supported += 1
    
    return supported / len(claims) if claims else 1.0


def check_claim_against_contexts(claim: str, contexts: list[str]) -> bool:
    """使用 LLM-as-Judge 验证声明"""
    prompt = f"""Determine if the CLAIM is directly supported by the CONTEXT.
    
    CONTEXT:
    {' '.join(contexts[:3])}
    
    CLAIM: {claim}
    
    Is this claim directly supported by the context? Answer ONLY 'Yes' or 'No':"""
    
    result = llm_generate(prompt, max_tokens=5)
    return result.strip().lower() == 'yes'
```

**典型的问题**：LLM 可能将自身的预训练知识与检索结果混合。如果 LLM 用自己的知识代替检索结果，说明 RAG 的"行为约束"失败了。

### 2. Answer Relevance（回答相关性）

**定义**：生成的回答是否直接针对用户的查询，不包含无关信息。

```python
def evaluate_answer_relevance(question: str, answer: str) -> float:
    """评估回答与问题的相关性"""
    
    # 方法：从回答反向生成假设问题，计算与原始问题的相似度
    prompt = f"""Generate {NUM_HYPO_QUESTIONS} questions that this answer addresses:
    
    Answer: {answer}
    
    Generate questions (one per line):"""
    
    generated_questions = llm_generate_list(prompt, num=3)
    
    # 计算生成的假设问题和原始问题的语义相似度
    question_emb = embed(question)
    similarities = []
    
    for gq in generated_questions:
        gq_emb = embed(gq)
        sim = cosine_similarity(question_emb, gq_emb)
        similarities.append(sim)
    
    return np.mean(similarities)
```

### 3. Completeness（完整性）

**定义**：回答是否覆盖了用户问题的所有方面。

```python
def evaluate_completeness(question: str, answer: str) -> float:
    """评估回答的完整度"""
    
    prompt = f"""Analyze if the answer COMPLETELY addresses all aspects of the question.
    
    Question: {question}
    Answer: {answer}
    
    First, list the sub-questions or aspects implied by the question.
    Then, for each aspect, determine if it's addressed in the answer.
    
    Output JSON:
    {{
        "aspects": ["aspect 1", "aspect 2", ...],
        "covered": [true, false, ...],
        "completeness_score": 0.85
    }}"""
    
    result = llm_generate_json(prompt)
    covered = sum(result['covered'])
    total = len(result['covered'])
    return covered / total if total > 0 else 1.0
```

## 其他重要生成指标

### 4. Context Utilization（上下文利用率）

**评价 LLM 是否有效使用了检索到的上下文**：

```python
def evaluate_context_utilization(answer: str, retrieved_contexts: list[str]) -> float:
    """评估 LLM 多大程度上利用了检索到的文档"""
    
    usage_scores = []
    for ctx in retrieved_contexts:
        # 回答中是否有来自该上下文的关键信息？
        key_info = extract_key_information(ctx)
        used = sum(1 for info in key_info if info in answer)
        usage_scores.append(used / len(key_info) if key_info else 0)
    
    return np.mean(usage_scores)
```

### 5. Conciseness（简洁度）

```python
def evaluate_conciseness(question: str, answer: str) -> float:
    """评估回答是否简明扼要（没有不必要的冗余）"""
    # 理想回答长度与真实回答长度的比率
    ideal_length = len(question) * 3  # 粗略估计理想回答长度
    actual_length = len(answer)
    
    if actual_length <= ideal_length * 1.2:
        return 1.0  # 简洁
    else:
        return max(0, 1.0 - (actual_length - ideal_length) / ideal_length)
```

## 指标使用场景矩阵

| 场景 | 最重要的指标 | 原因 |
|------|------------|------|
| **企业知识库问答** | Faithfulness | 事实准确性要求最高 |
| **客服自动回复** | Answer Relevance + Completeness | 必须准确理解并回答用户问题 |
| **研究辅助** | Completeness + Context Utilization | 需要全面覆盖且充分利用资料 |
| **代码生成** | Faithfulness（API 使用正确性） | 不能使用不存在的 API |
| **摘要生成** | Conciseness + Faithfulness | 信息浓缩且忠于原文 |

## 评估方法对比

| 方法 | 成本 | 可扩展性 | 与人类判断一致性 | 适用阶段 |
|------|------|---------|----------------|---------|
| 人工评估 | 高 | 低 | 金标准 | 最终验收 |
| LLM-as-Judge | 中 | 高 | 0.6-0.8 一致性 | 日常迭代 |
| 专用评估模型(如 PROMETHEUS) | 低 | 高 | 0.7-0.85 一致性 | CI/CD |
| 参考指标(ROUGE/BLEU) | 极低 | 最高 | 0.3-0.5 一致性 | 快速筛查 |

## 最大挑战

1. **Faithfulness 的粒度认定**：一个"部分正确"的声明怎么评分？"Transformer 在 2017 年被 Google 提出"——如果检索文档只提到"2017"没提"Google"？
2. **回答的相关性主观性**：用户认为相关 vs 评估者认为相关可能不同
3. **多答案模式**：一个问题可能有多个正确的回答方式，完整性评估难以客观
4. **上下文质量耦合**：如果检索到的文档质量差，LLM 生成效果必然差——但这是检索的问题不是生成的问题

## 能力边界

- ✅ 有效检测 Faithfulness 问题（LLM 会引入检索文档中不存在的信息）
- ✅ 量化回答与问题的相关性
- ✅ 在 80% 的案例中与人工评估一致
- ❌ 无法检测"检索文档本身就错了"（如文档包含过时信息）
- ❌ 对创造性、主观性问题的评估不可靠（LLM 无法判断"好"的创意回答）
- ❌ 跨语言评估（如日文检索+中文生成）时 Faithfulness 检测精度下降

## 推荐工具

- **RAGAS**: Faithfulness + Answer Relevance + Context Precision
- **TruLens**: Answer Relevance + Groundedness + Context Relevance
- **DeepEval**: Faithfulness + Hallucination + Toxicity 等
- **PROMETHEUS**: 专用评估模型，与人类判断一致性高
