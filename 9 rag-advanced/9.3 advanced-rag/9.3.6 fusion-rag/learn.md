# 9.3.6 Fusion RAG — 多源生成融合

## 概述

Fusion RAG（多源融合检索增强生成）不是单一的检索策略，而是一系列让 RAG 系统**从多个知识源、多种检索方式、多轮检索结果中融合信息**的模式总称。核心思想是：单一检索源总是有偏的——稠密检索擅长语义但遗漏精确匹配，稀疏检索擅长关键词但缺乏语义理解，知识图谱擅长关系推理但覆盖有限。Fusion RAG 通过智能融合多个互补源的检索结果，显著提升回答的全面性和准确性。

## 背景与核心矛盾

### 单一检索源的局限

```
检索源全景：
┌────────────────────────────────────────────────────────────┐
│                    单一检索源的局限                         │
├────────────┬────────────────┬──────────────────────────────┤
│   检索源    │     强项        │          弱项               │
├────────────┼────────────────┼──────────────────────────────┤
│ Dense      │ 语义相似度      │ 忽略精确关键词、ID、数字     │
│ Sparse     │ 关键词精确匹配   │ 无法理解语义、同义词         │
│ KG         │ 关系推理        │ 覆盖有限、构建成本高         │
│ Web Search │ 实时信息        │ 噪声多、质量不可控           │
│ Code Index │ 代码结构        │ 泛化知识少                   │
└────────────┴────────────────┴──────────────────────────────┘
```

### 核心矛盾

**"多源"意味着多视角但也意味着多冲突。** 不同的检索源可能返回不一致甚至矛盾的信息——向量检索说"Paris 是法国的首都"，网络搜索可能返回"Paris Hilton 的最新动态"。Fusion RAG 要解决的不是"如何检索更多"，而是"如何在冲突中提炼共识，在互补中构建完整"。

## Fusion RAG 的三种融合层次

### 1. 结果级融合（Late Fusion）

最直观的方式：分别检索 → 合并结果 → 去重/重排序 → 统一生成。

```
Query → VectorDB → [DocV1, DocV2, DocV3]  ─┐
Query → BM25    → [DocB1, DocB2, DocB3]  ──┤──→ Merge & Rerank → Generate
Query → Web     → [DocW1, DocW2, DocW3]  ──┘
```

**优点**：各检索源独立，部署简单，易于扩展新源
**缺点**：延迟取决于最慢的源；分数归一化困难（各源分数尺度不同）

### 2. 特征级融合（Early Fusion）

在检索过程中就将多源信号融合，例如将向量检索和稀疏检索的分数在 embedding 空间统一：

```
Query → Hybrid Encoder → Unified Embedding Space → Unified Search → Generate
                                 ↑
                    Dense Signal + Sparse Signal
```

**优点**：统一的排序尺度，检索效率高
**缺点**：耦合度高，难以独立升级子系统

### 3. 生成级融合（Generative Fusion）

让 LLM 直接对多路检索结果进行"思考式"融合——分别阅读、比较、综合：

```
Query → Route to Multiple Retrievers
         ↓
Round 1: Docs → LLM: "基于这些信息回答" → Partial Answer
Round 2: Partial + New Docs → LLM: "对比并综合" → Refined Answer
Round 3: Refined + More Docs → LLM: "检查完整性" → Final Answer
```

**优点**：最灵活，能处理复杂冲突
**缺点**：Token 消耗大，延迟高

## 关键技术：多路融合算法

### RRF（Reciprocal Rank Fusion）

RRF 是最简单、最有效的多路融合方法之一：

```python
def rrf_merge(ranked_lists: list[list[Doc]], k: int = 60) -> list[tuple[str, float]]:
    """
    RRF: 对每条文档在各路排序中的位置做倒数加权
    score(d) = Σ(1 / (k + rank_i(d)))
    k 是平滑参数，通常取 60
    """
    scores = {}
    for rank_list in ranked_lists:
        for pos, doc in enumerate(rank_list):
            doc_id = doc['id']
            # 位置越靠前，得分越高；k 控制平滑程度
            scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + pos + 1)
    
    # 按融合得分排序
    ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return ranked
```

### CCD（Cross-Encoder 重排序融合）

用 Cross-Encoder 对多路结果统一打分：

```python
class CCD:
    """Cross-encoder Fusion: 先用 RRF 粗筛，再用 Cross-Encoder 精排"""
    def __init__(self, cross_encoder_model='BAAI/bge-reranker-v2-m3'):
        self.reranker = load_cross_encoder(cross_encoder_model)
    
    def fuse(self, query: str, result_lists: list[list[Doc]], top_k: int = 10):
        # Step 1: RRF 初筛（粗排）
        rrf_scores = rrf_merge(result_lists, k=60)
        candidates = [doc_id for doc_id, _ in rrf_scores[:50]]  # 放大候选集
        
        # Step 2: Cross-Encoder 精排（计算 query-doc 真实语义相关性）
        pairs = [(query, doc.text) for doc in candidates]
        scores = self.reranker.compute_score(pairs)
        
        # Step 3: 按 Cross-Encoder 分数重排序
        ranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)
        return ranked[:top_k]
```

## Recursive Retrieval（递归检索）

递归检索是 Fusion RAG 的一种重要变体——**不是多源并行，而是多步串行**。核心思想是：一次检索可能不够，用已检索到的内容来指导下一步检索。

```
Round 1: Q = "Transformer 和 RNN 的优缺点对比"
         Retrieve → docs about Transformer
        
Round 2: Q' = docs_content + "那么 RNN 呢？"
         Retrieve → docs about RNN

Round 3: Q'' = transformer_docs + rnn_docs + "综合对比"
         Retrieve → complementary comparison docs

Final:   Generate comprehensive comparison
```

```python
class RecursiveFusionRAG:
    def __init__(self, retriever, llm, max_depth=3):
        self.retriever = retriever
        self.llm = llm
        self.max_depth = max_depth
    
    def answer(self, query: str):
        collected_docs = []
        current_query = query
        
        for depth in range(self.max_depth):
            # 检索
            docs = self.retriever.retrieve(current_query, k=3)
            collected_docs.extend(docs)
            
            # 判断是否继续递归
            prompt = f"""基于当前收集的信息：
            {self._format_docs(collected_docs)}
            
            问题：{query}
            
            这些信息充分吗？如果不充分，还需要什么？输出：
            - SUFFICIENT（足够）→ 结束检索
            - NEED_MORE: <需要检索的问题>（不够，继续）
            """
            
            decision = self.llm.generate(prompt)
            
            if decision.startswith("SUFFICIENT"):
                break
            else:
                # 提取新问题，继续递归
                current_query = decision.replace("NEED_MORE:", "").strip()
        
        # 最终生成
        final_prompt = f"""基于以下信息回答：
        {self._format_docs(collected_docs)}
        
        问题：{query}
        """
        return self.llm.generate(final_prompt)
    
    def _format_docs(self, docs):
        return "\n\n".join([f"[{i+1}] {d}" for i, d in enumerate(docs)])
```

## HF-RAG：层级融合架构

HF-RAG（Hierarchical Fusion RAG）是 2025 年提出的一种先进融合框架，核心是**多源 × 多排序器的层级融合**：

```
                            ┌──────────────────────┐
                            │        Query          │
                            └──────────┬───────────┘
                                       │
            ┌──────────────────────────┼──────────────────────────┐
            ▼                          ▼                          ▼
    ┌───────────────┐        ┌───────────────┐        ┌───────────────┐
    │   Source 1    │        │   Source 2    │        │   Source 3    │
    │  (Vector DB)  │        │  (BM25 Index) │        │  (Web Search) │
    └───────┬───────┘        └───────┬───────┘        └───────┬───────┘
            │                        │                        │
    ┌───────▼───────┐        ┌───────▼───────┐        ┌───────▼───────┐
    │  Ranker 1    │        │  Ranker 2    │        │  Ranker 3    │
    │ (semantic)   │        │ (keyword)    │        │ (recency)    │
    └───────┬───────┘        └───────┬───────┘        └───────┬───────┘
            │                        │                        │
            └────────────────────────┼────────────────────────┘
                                     │
                          ┌──────────▼───────────┐
                          │   Level 1 Fusion     │
                          │   (RRF / Lightweight) │
                          └──────────┬───────────┘
                                     │
                          ┌──────────▼───────────┐
                          │   Global Reranker     │
                          │  (Cross-Encoder)      │
                          └──────────┬───────────┘
                                     │
                          ┌──────────▼───────────┐
                          │   Level 2 Fusion     │
                          │  (Generator-level    │
                          │   evidence merging)  │
                          └──────────┬───────────┘
                                     │
                          ┌──────────▼───────────┐
                          │   Final Generation   │
                          └──────────────────────┘
```

## 与 Hybrid Search 的区别

| 维度 | Hybrid Search | Fusion RAG |
|------|---------------|------------|
| **范围** | 检索层（Dense + Sparse） | 检索层 + 生成层 |
| **融合方式** | 分数归一化 + RRF | RRF + 重排序 + 内容融合 |
| **多源策略** | 统一检索 | 独立检索 + 差异化策略 |
| **冲突处理** | 排序优先级 | 多轮验证 + 综合判断 |
| **递归能力** | 无 | 支持递归和自适应 |
| **适用场景** | 通用知识检索 | 复杂推理、多源验证 |

## 工程挑战

### 1. 延迟叠加问题

最慢的检索源决定整体延迟。解决方案：
```python
class TimeboxedFusionRAG:
    """带超时控制的多源融合"""
    def __init__(self, retrievers, timeout_ms=500):
        self.retrievers = retrievers
        self.timeout = timeout_ms
    
    def parallel_retrieve(self, query):
        results = {}
        with ThreadPoolExecutor(max_workers=len(self.retrievers)) as executor:
            futures = {
                executor.submit(r.retrieve, query): name 
                for name, r in self.retrievers.items()
            }
            for future in as_completed(futures, timeout=self.timeout):
                name = futures[future]
                try:
                    results[name] = future.result()
                except TimeoutError:
                    logger.warning(f"{name} timed out, using fallback")
                    results[name] = self.retrievers[name].fallback()
        return results
```

### 2. 源间冲突消解

不同检索源返回矛盾信息时，需要冲突消解策略：

```
冲突类型          策略
─────────        ──────────
事实矛盾         多源投票 + 置信度加权
时间敏感         时间戳比较 + 来源权威性
覆盖不完整        递归补充 + 补全检索
模糊匹配不准确    Cross-Encoder 验证 + 语义阈值
```

### 3. 分数归一化

不同检索源的分数尺度不同，直接比较无效：
- Vector DB: 余弦相似度 [0, 1]
- BM25: 原始分数 [0, ~50+]
- Web: 搜索引擎 PageRank 分数 [0, 100]

```python
def normalize_scores(results, method='minmax'):
    """分数归一化，使各源分数可比"""
    if method == 'minmax':
        scores = [r['score'] for r in results]
        min_s, max_s = min(scores), max(scores)
        if max_s == min_s:
            return [{**r, 'score': 0.5} for r in results]
        return [{**r, 'score': (r['score'] - min_s) / (max_s - min_s)} 
                for r in results]
    elif method == 'rank':
        return [{**r, 'score': 1.0 / (idx + 1)} 
                for idx, r in enumerate(sorted(results, key=lambda x: x['score'], reverse=True))]
```

## 2025-2026 前沿进展

1. **HF-RAG**：层级融合 RAG，多源 × 多排序器 × 多级融合，显著提升检索质量
2. **Agent-Driven Fusion**：让 Agent 自主决定何时引入新源、何时停止检索、如何综合冲突信息
3. **PsyFusion-RAG**：面向心理领域的多源知识融合模型，验证了 Fusion RAG 在专业领域的有效性
4. **生成级融合自动化**：用 LLM 作为融合中心，自动检测信息冲突并驱动递归检索
5. **自适应源选择**：根据查询类型自动选择需要查询的源（而非全部），降低延迟开销

## 适用场景

| 场景 | 推荐度 | 原因 |
|------|--------|------|
| 复杂多跳问答 | ⭐⭐⭐⭐⭐ | 多源互补覆盖不同知识维度 |
| 事实核查 | ⭐⭐⭐⭐⭐ | 多源交叉验证降低幻觉 |
| 研究报告生成 | ⭐⭐⭐⭐ | 需要全面的多视角信息 |
| 企业知识库 | ⭐⭐⭐⭐ | 整合结构化 + 非结构化数据 |
| 实时新闻查询 | ⭐⭐⭐ | 需要权衡时效性和权威性 |
| 单源足够场景 | ⭐ | 过度设计，增加无谓延迟 |

## 工程优化方向

1. **源选择策略**：根据查询主题动态选择检索源子集，避免全量检索
2. **渐进式检索**：先检便宜源（缓存），再检中等（Vector DB），最后检慢源（Web）
3. **冲突检测前置**：在进入生成阶段前检测多源矛盾，提前触发冲突消解
4. **融合权重学习**：基于历史反馈学习各源在特定领域/查询类型上的权重
5. **增量融合**：新源加入时不必重新融合所有结果，只需增量更新融合分数
6. **缓存层级**：在 RRF 结果层、重排序层分别设置缓存，避免重复计算
