# 9.7.1 retrieval-metrics — 检索评估指标

## 简单介绍

检索评估指标是衡量 RAG 系统检索模块效果的量化标准。检索是 RAG 的第一道关卡——如果检索到的文档不相关，后续 LLM 无论如何努力都无法生成正确答案。因此，检索指标的优化是 RAG 系统优化的首要任务。

## 核心指标详解

### 1. 精确率与召回率（Precision & Recall）

```
检索结果: [✅, ✅, ❌, ✅, ❌, ❌, ✅, ❌]  (8个结果，4个相关)
总相关文档: 10个

Precision@8 = 4/8 = 0.5   (检索结果中有多少是相关的)
Recall@8   = 4/10 = 0.4   (所有相关文档中检索到了多少)
F1@8       = 2 * 0.5 * 0.4 / (0.5 + 0.4) = 0.44
```

```python
def compute_precision_recall(retrieved: list[str], 
                              relevant: set[str], 
                              k: int = None):
    """计算 Precision@k 和 Recall@k"""
    if k:
        retrieved = retrieved[:k]
    
    retrieved_set = set(retrieved)
    true_positives = len(retrieved_set & relevant)
    
    precision = true_positives / len(retrieved) if retrieved else 0
    recall = true_positives / len(relevant) if relevant else 0
    f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0
    
    return {'precision': precision, 'recall': recall, 'f1': f1}
```

**RAG 中的实践建议**:
- **Precision 更重要**的场景：LLM 上下文窗口有限，噪声文档会干扰生成
- **Recall 更重要**的场景：需要覆盖所有相关信息点（如分析性查询）

### 2. MRR（Mean Reciprocal Rank）

```python
def compute_mrr(retrieved_lists: list[list[str]], 
                relevant_sets: list[set[str]]) -> float:
    """计算 MRR: 第一个相关结果的排名的倒数，再平均"""
    
    reciprocal_ranks = []
    
    for retrieved, relevant in zip(retrieved_lists, relevant_sets):
        for rank, doc in enumerate(retrieved, 1):
            if doc in relevant:
                reciprocal_ranks.append(1.0 / rank)
                break
        else:
            reciprocal_ranks.append(0.0)  # 未找到相关文档
    
    return sum(reciprocal_ranks) / len(reciprocal_ranks) if reciprocal_ranks else 0.0

# MRR = 1: 第一个结果总是相关的
# MRR = 0.5: 平均第一个相关结果在第二位
# MRR = 0.1: 平均第一个相关结果在第十位
```

**适用场景**：RAG 只需要 Top-1/Top-3 结果时（如单事实问答）

### 3. NDCG（Normalized Discounted Cumulative Gain）

支持**多级相关性**的指标（不仅仅是"相关/不相关"二元判断）：

```python
def compute_ndcg(retrieved: list[str], 
                 relevance_scores: dict[str, int], 
                 k: int = 10):
    """
    relevance_scores: {doc_id: score}, score ∈ [0, 3]
    0 = 不相关, 1 = 弱相关, 2 = 相关, 3 = 高度相关
    """
    retrieved = retrieved[:k]
    
    # DCG: 位置折扣的累加收益
    dcg = 0
    for i, doc in enumerate(retrieved):
        rel = relevance_scores.get(doc, 0)
        dcg += rel / np.log2(i + 2)  # log2(1+1) = 1 for position 1
    
    # IDCG: 理想排序下的最大 DCG
    ideal = sorted(relevance_scores.values(), reverse=True)[:k]
    idcg = sum(rel / np.log2(i + 2) for i, rel in enumerate(ideal))
    
    return dcg / idcg if idcg > 0 else 0.0
```

### 4. Hit Rate（命中率）

```python
def compute_hit_rate(retrieved_lists: list[list[str]], 
                     relevant_sets: list[set[str]], 
                     k: int = 5) -> float:
    """Hit@k: 前k个结果中是否包含至少一个相关文档"""
    
    hits = 0
    for retrieved, relevant in zip(retrieved_lists, relevant_sets):
        if any(doc in relevant for doc in retrieved[:k]):
            hits += 1
    
    return hits / len(retrieved_lists) if retrieved_lists else 0.0
```

## 各指标的 RAG 场景选择指南

| 指标 | 含义 | 什么时候用 | 什么时候不用 |
|------|------|-----------|------------|
| **Recall@k** | 检索覆盖了多少相关信息 | 需要全面信息的场景（分析报告） | 只需要一个精确答案时 |
| **Precision@k** | 检索结果中有多少有用 | 上下文窗口有限时（Token 敏感） | 需要高召回的多源分析 |
| **MRR** | 第一个正确答案排多前 | 只需要一个正确答案时（事实问答） | 需要多个答案时 |
| **NDCG** | 排序质量（多级相关性） | 需要精细化排序评估 | 只有二元相关性标注时 |
| **Hit@k** | 是否"捡到"相关信息 | 关注系统是否有"可能性" | 关注排序精度时 |

## 检索评估中的难点

1. **标注成本高**：需要人工标注每个查询的"理想检索结果列表"
2. **相关性是主观的**：同一文档对两个用户可能相关度不同
3. **动态相关性**：在多轮对话中，之前轮次已提供的信息会影响后续的"相关性"
4. **计算的近似问题**：在实际 RAG 中无法获得"全部相关文档"的确切数量，Recall 只能近似计算

## 工程实践建议

1. **将指标嵌入 CI/CD**：每次索引更新或检索策略变更后自动运行检索指标测试
2. **分场景评估**：不同查询类型（事实性/分析性）分别评估，不混合计算
3. **关注 P@1**：生产 RAG 中 P@1（第一个检索结果的相关性）是最直接影响用户体验的指标
4. **结合业务指标**：检索指标只是中间指标，最终需要关联用户满意度

## 推荐工具

- **RAGAS**: 提供 Context Precision / Context Recall 的计算
- **TruLens**: 支持检索与生成分离的评估
- **DeepEval**: 多种检索指标的自动化计算
- **pytrec_eval**: TREC 标准的评估工具
