# 9.6.2 query-rewriting — 查询改写

## 简单介绍

查询改写（Query Rewriting / Query Reformulation）是指**将用户的原始查询转化为更适合检索系统处理的形式**的技术。用户输入的查询往往简短、模糊、含口语化表达，而检索系统的索引基于规范化的书面语言。改写就是在这两者之间架设桥梁。

## 基本原理

### 改写类型一览

| 改写类型 | 原始查询 | 改写后 |
|---------|---------|--------|
| **扩展** | "苹果股票" | "Apple Inc. (AAPL) 股票价格 近期走势" |
| **规范化** | "咋用python读csv啊" | "如何使用 Python 读取 CSV 文件" |
| **消除歧义** | "苹果" | "Apple 公司——电子产品与软件开发" |
| **翻译** | "Transformer原理" | "Transformer architecture self-attention mechanism explanation" |
| **查询补充** | ""（多轮对话） | 继承上一轮上下文补充完整查询 |

### 核心改写策略

```python
class QueryRewriter:
    def rewrite(self, query: str, context: QueryContext = None) -> str:
        """将原始查询改写为检索友好格式"""
        
        # 1. 方法选择
        strategy = self._select_strategy(query)
        
        if strategy == 'expansion':
            return self._expand_query(query)
        elif strategy == 'normalization':
            return self._normalize_query(query)
        elif strategy == 'disambiguation':
            return self._disambiguate_query(query, context)
        elif strategy == 'contextualization':
            return self._contextualize_query(query, context)
        
        return query  # 不需要改写
    
    def _expand_query(self, query: str) -> str:
        """查询扩展：补充同义词、相关术语"""
        entities = extract_entities(query)
        expanded = query
        
        for entity in entities:
            synonyms = self.thesaurus.get(entity, [])
            if synonyms:
                expanded = expanded.replace(entity, 
                    f"{entity} ({' '.join(synonyms[:2])})")
        
        return expanded
```

## 背景：为什么查询改写有效

Embedding 模型的核心限制：**它在一个连续的语义空间中计算相似度，但用户的自然语言和文档的语言分布天然不同**。

```
用户查询分布:
  短(平均3-8个词)
  口语化
  含指代("它"、"那个")
  有错别字
  模糊歧义

文档索引分布:
  长(完整句子/段落)
  书面化
  显式引用
  标准化术语
  精确无歧义

改写就是将用户分布"对齐"到文档分布
```

## 之前是怎么做的

| 阶段 | 方法 | 效果 |
|------|------|------|
| 传统 IR (pre-2020) | 查询扩展 + 伪相关反馈 | 有限改善 |
| 早期 RAG (2023) | 直接使用原始查询 | 次优检索结果 |
| PreQRAG (2024) | Classify + Rewrite 两阶段 | Recall@5 提升 20-40% |
| LLM-based (2025) | LLM 生成多个改写版本 + 检索后投票 | 最先进但成本高 |

## 主流优化方向

### 1. LLM 驱动的智能改写

```python
def llm_query_rewrite(query: str, context: str = "") -> str:
    """使用 LLM 生成检索友好版的查询"""
    
    prompt = f"""Rewrite this search query to be more effective for 
    retrieving relevant information from a knowledge base.
    
    Rules:
    - Expand abbreviations and acronyms
    - Use standard terminology
    - Add context-specific keywords
    - Keep the core intent unchanged
    
    Original query: {query}
    Context: {context or '(none)'}
    
    Rewritten query (concise, search-optimized):"""
    
    return llm_generate(prompt, max_tokens=100, temperature=0.1)
```

### 2. 多版本生成 + 投票

```python
def multi_rewrite_and_vote(query: str, retriever, n_versions: int = 3):
    """生成多个改写版本，分别检索后投票融合结果"""
    
    # 使用不同策略生成多个版本
    versions = [
        query,                                          # 原始版本
        llm_query_rewrite(query),                       # LLM 改写
        expand_query_with_synonyms(query),               # 同义词扩展
    ]
    
    # 分别检索
    all_results = []
    for v in versions:
        results = retriever.retrieve(v, top_k=10)
        all_results.append(results)
    
    # RRF 融合多路结果
    fused = reciprocal_rank_fusion(all_results)
    
    return fused[:10]  # 返回融合后的 Top-10
```

### 3. 结构化查询生成

适用于需要多个检索维度的复杂查询：

```python
def generate_structured_query(query: str) -> dict:
    """将自然语言查询转为结构化多字段检索条件"""
    
    prompt = f"""Parse this query into structured search fields:
    
    Query: "2024年 Google 的机器学习论文"
    
    Expected output format:
    {{
        "keywords": ["machine learning", "deep learning", "AI"],
        "organization": ["Google", "Google Research"],
        "year": "2024",
        "document_type": ["paper", "publication"],
        "language": "en"
    }}
    
    Query: {query}
    Structured:"""
    
    return llm_generate_json(prompt)
```

### 4. 反事实改写

```python
def counterfactual_rewrite(query: str, original_docs: list[str]) -> str:
    """当原始查询检索结果差时，基于失败的反事实改写"""
    
    if len(original_docs) < MIN_RESULTS:
        # 基于"检索失败"信号进行激进改写
        # 提取用户查询中的关键实体
        entities = extract_entities(query)
        # 使用更宽泛的术语
        broader_terms = expand_to_broader_terms(entities)
        # 生成更通用的改写
        return f"{' '.join(broader_terms)} overview guide tutorial"
    
    return query
```

## 查询改写方法对比

| 方法 | 召回提升 | 延迟开销 | Token 成本 | 适用场景 |
|------|---------|---------|-----------|---------|
| 同义词扩展 | +5-10% | ~2ms | 0 | 简单事实查询 |
| 规则改写(拼写纠正等) | +5-15% | ~5ms | 0 | 口语化查询 |
| 小模型改写 (T5/flan-T5) | +10-20% | ~30ms | 低 | 通用场景 |
| LLM 改写 | +20-40% | ~200ms | 中 | 高价值/复杂查询 |
| 多版本投票 | +25-45% | ~500ms | 高 | 最关键场景 |

## 最大挑战

1. **过改写**：改写后查询偏离原意，检索到不相关的内容
2. **改写延迟**：LLM 改写的 latency 可能达到 200-500ms，是检索延迟的 5-10 倍
3. **领域特异性**：通用 LLM 不知道特定领域的术语偏好，可能用错术语
4. **改写与检索的联合优化**：分开优化改写和检索可能不是全局最优

## 能力边界

- ✅ 显著提升模糊短查询的检索质量
- ✅ 减少口语化和不规范表述的负面影响
- ✅ 在 80% 的场景中改写后的检索效果优于原始查询
- ❌ 对于非常清晰的精确查询，改写可能是多余的甚至起反作用
- ❌ 高专业性术语（如医学、法律）需要领域特定的改写知识
- ❌ 改写不能创造索引中不存在的信息

## 工程优化

1. **改写决策门控**：只对低置信度查询进行改写，清晰查询直接检索
2. **改写缓存**：相同查询的改写结果缓存，TTL 根据索引更新频率设置
3. **A/B 测试**：在线评估改写前后的效果差异（离线用 Recall，在线用用户满意度）
4. **渐进式改写**：先加同义词扩展（快速），如果效果不够再触发 LLM 改写（慢速）

## 推荐论文

- **PreQRAG (2025)**: Classify and Rewrite for Enhanced RAG
- **Query2Doc (2023)**: 用 LLM 生成假设性文档作为查询扩展
- **HyDE (2022)**: 假设性文档嵌入
- **UniRAG (2024)**: 统一分解改写推理框架
