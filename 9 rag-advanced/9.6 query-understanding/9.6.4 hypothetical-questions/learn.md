# 9.6.4 hypothetical-questions — HyDE 假设性文档生成

## 简单介绍

HyDE（Hypothetical Document Embeddings，假设性文档嵌入）是一种**通过 LLM 生成"假设性文档"来辅助检索**的技术。其核心思想出奇地简单但有效：**不做"查询 → 检索文档"，而是做"查询 → LLM 生成一个假设的理想文档 → 用这个假文档去检索真实文档"**。

## 基本原理

### HyDE 的工作流程

```
传统检索:
  用户查询 → [Embedding] → 向量查询 → [相似度搜索] → 真实文档
  "苹果公司" → [0.1, 0.3, ...] → 在向量空间中搜索 → ...

HyDE 检索:
  用户查询 → [LLM] → 假设文档 → [Embedding] → 向量查询 → [搜索] → 真实文档
  "苹果公司" → "Apple Inc. 是一家..." → [0.5, 0.8, ...] → 更精确搜索 → ...
```

### 为什么 HyDE 有效

核心洞察：**embedding 模型在"文档-文档"空间的相似度匹配远比"查询-文档"空间准确**。

```python
# 传统查询-文档相似度
query_emb = embed("苹果公司的营收情况")
doc1_emb  = embed("Apple Inc. reported revenue of $394B in 2024")
similarity = cosine_similarity(query_emb, doc1_emb)  # 可能只有 0.75

# HyDE 的文档-文档相似度
hypothetical_doc = llm_generate("根据'苹果公司的营收情况'生成一段描述性文本")
# → "Apple Inc.（苹果公司）在2024年取得了显著的营收增长。根据最新的..."

hyp_doc_emb = embed(hypothetical_doc)
doc1_emb    = embed("Apple Inc. reported revenue of $394B in 2024")
similarity  = cosine_similarity(hyp_doc_emb, doc1_emb)  # 可能达到 0.92
```

## 背景：检索中的"分布偏移"问题

向量检索中的根本问题：**查询和文档之间存在分布偏移（Distribution Shift）**。

- 查询通常是短文本（3-10个词），信息密度低
- 文档是长文本（100-1000+词），信息密度高
- Embedding 模型在两种文本上的表现不均匀

用短查询的 embedding 去匹配长文档的 embedding，效果天然不如"长文本匹配长文本"。

## HyDE 的完整实现

```python
class HyDERetriever:
    def __init__(self, doc_embeddings: np.ndarray, 
                 documents: list[str], llm_client):
        self.doc_embeddings = doc_embeddings
        self.documents = documents
        self.llm = llm_client
    
    def retrieve(self, query: str, top_k: int = 10) -> list[dict]:
        """HyDE 检索主流程"""
        
        # Step 1: 生成假设性文档
        hyde_doc = self._generate_hypothetical_document(query)
        
        # Step 2: 对假设文档做 Embedding
        hyde_embedding = self._embed(hyde_doc)
        
        # Step 3: 用假设文档向量检索真实文档
        scores = self._compute_similarity(hyde_embedding)
        top_indices = np.argsort(scores)[-top_k:][::-1]
        
        results = []
        for idx in top_indices:
            results.append({
                'document': self.documents[idx],
                'score': float(scores[idx]),
                'hyde_doc': hyde_doc  # 供调试使用
            })
        
        return results
    
    def _generate_hypothetical_document(self, query: str) -> str:
        """根据查询生成一段类似真实文档的文本"""
        
        prompt = f"""Write a paragraph that would serve as an ideal 
        document to answer the following question. 
        The paragraph should be factual, detailed, and directly relevant.
        
        Question: {query}
        
        Hypothetical document:"""
        
        return self.llm.generate(prompt, max_tokens=200)
```

### 不同查询类型的 HyDE Prompt

```python
def hyde_prompt_by_type(query: str, query_type: str) -> str:
    """根据查询类型选择合适的 HyDE 生成 prompt"""
    
    prompts = {
        'factual': f"""Write a concise factual passage that answers:
        {query}
        Passage (factual, specific):""",
        
        'explanatory': f"""Write an educational passage explaining:
        {query}
        Passage (clear, pedagogical):""",
        
        'code': f"""Write documentation or code comments that describe:
        {query}
        Documentation:""",
        
        'comparison': f"""Write a passage that compares and contrasts:
        {query}
        Comparative passage:""",
    }
    
    return prompts.get(query_type, prompts['factual'])
```

## 主要变体

### 1. 多 HyDE 生成 + 聚合

```python
def multi_hyde(query: str, n: int = 3, top_k: int = 10):
    """生成多个假设文档，分别检索后聚合"""
    
    all_results = []
    
    for i in range(n):
        # 使用不同的 temperature 生成多样化的假设文档
        hyde_doc = generate_hypothetical_doc(query, temperature=0.3 + i * 0.3)
        results = retrieve_with_embedding(hyde_doc, top_k)
        all_results.append(results)
    
    # 使用 RRF 融合多路结果
    return reciprocal_rank_fusion(all_results)
```

### 2. 查询-文档混合 Embedding

```python
def hybrid_embedding(query: str, alpha: float = 0.3) -> np.ndarray:
    """同时使用原始查询和 HyDE 文档的混合向量"""
    
    query_emb = embed_model.encode(query)
    hyde_doc = generate_hypothetical_document(query)
    hyde_emb = embed_model.encode(hyde_doc)
    
    # 加权混合: 原始查询权重 + 假设文档权重
    hybrid = (1 - alpha) * query_emb + alpha * hyde_emb
    return hybrid
```

### 3. HyDE + 查询感知权重

```python
def adaptive_hyde(query: str, retriever) -> list[dict]:
    """自适应 HyDE：根据查询特性决定 alpha 权重"""
    
    # 查询越短/模糊，HyDE 权重越大
    query_length = len(query)
    entity_count = len(extract_entities(query))
    
    # 短查询、少实体的查询更需要 HyDE
    alpha = max(0.3, 1.0 - (query_length / 100) - (entity_count / 10))
    alpha = min(0.8, alpha)  # 限制在 [0.3, 0.8]
    
    return retriever.hybrid_retrieve(query, alpha=alpha)
```

## HyDE vs 其他方法的对比

| 方法 | 检索质量 | 延迟开销 | Token 成本 | 额外依赖 |
|------|---------|---------|-----------|---------|
| 原始查询检索 | 基准 | 0 | 0 | 无 |
| 查询扩展 | +5-15% | ~5ms | 0 | 同义词表 |
| 查询改写 | +10-25% | ~200ms | 低-中 | LLM |
| **HyDE** | **+15-30%** | **~300ms** | **中** | **LLM** |
| 多 HyDE 投票 | +20-35% | ~800ms | 高 | LLM |

## 最大挑战

1. **LLM 生成质量依赖**：如果 LLM 生成的假设文档偏离了真实索引的分布，效果适得其反
2. **延迟开销**：HyDE 在检索之前增加了一次 LLM 调用，端到端延迟增加 200-500ms
3. **领域知识**：LLM 对特定领域（医学、法律）生成假设文档的能力有限
4. **与改写的关系**：HyDE 和查询改写都是"在检索前用 LLM 处理查询"，需要选择或组合

## 能力边界

- ✅ 短查询、模糊查询效果提升显著
- ✅ 对常见知识领域的检索改善明显
- ✅ 实现简单，无特殊基础设施需求
- ❌ LLM 本身不知道的领域知识无法生成有效的假设文档
- ❌ 如果真实索引中根本没有相关信息，HyDE 无法创造信息
- ❌ 对非常长的假设文档，embedding 可能会"稀释"关键信息

## 工程优化

1. **HyDE 缓存**：相同或相似查询的假设文档结果可缓存
2. **非对称编码**：使用不同的 encoder 编码假设文档和真实文档（如果能 fine-tune）
3. **轻量 HyDE**：使用小模型（T5-small）生成假设文档，而非大 LLM
4. **选择性 HyDE**：只有原始检索置信度低时才触发 HyDE

## 推荐论文和资料

- **HyDE (2022)**: "Precise Zero-Shot Dense Retrieval without Relevance Labeling" — 原始论文
- **Query2Doc (2023)**: 用 LLM 生成文档作为查询扩展
- **Contriever (2022)**: 无监督稠密检索，HyDE 的常用基础模型
