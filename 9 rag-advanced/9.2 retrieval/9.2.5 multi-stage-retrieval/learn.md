# 9.2.5 Multi-Stage Retrieval — 多级检索：级联过滤与渐进式精排

## 简单介绍

多级检索（Multi-Stage Retrieval）是一种**分阶段、渐进式**的检索架构。核心思想是：不试图用一个检索器一步到位找到最相关结果，而是构建一个从"宽召回 → 粗过滤 → 精排序"的流水线，每个阶段用不同粒度的模型和方法逐步缩小候选集、提升精度。

```
用户 Query
    │
    ├─ Stage 1: 元数据过滤（Metadata Filtering）── 快速剪枝，排除无关文档
    │
    ├─ Stage 2: 粗召回（Coarse Retrieval）      ── 宽召回 Top-K（如 BM25 + 向量检索）
    │
    ├─ Stage 3: 精排序（Reranking）              ── 交叉编码器重排 Top-N
    │
    └─ Stage 4: 后处理（Post-processing）        ── 去重、去噪、上下文组装
```

这种架构将检索从"单次查找"转变为"多轮过滤"，每一轮都在更小的候选集上做更精细的计算。

## 基本原理

### 漏斗架构（Funnel Architecture）

多级检索的本质是一个**信息漏斗**：上层宽、下层窄，每层丢弃大部分候选，只保留最可能相关的少量结果进入下一层。

```
候选池大小         计算成本/精度
    │                   │
    │ 1,000,000        很低（元数据索引扫描）
    ▼                     ▼
    │ 100,000          低（BM25 / 轻量向量索引）
    ▼                     ▼
    │ 10,000           中（ANN 向量检索）
    ▼                     ▼
    │ 1,000            高（双编码器 + MIPS）
    ▼                     ▼
    │ 100              很高（交叉编码器重排）
    ▼                     ▼
    │ 10              最高（LLM 精排）
```

### 各阶段的角色定义

| 阶段 | 候选集量级 | 模型/方法 | 延迟 | 目标 |
|------|-----------|-----------|------|------|
| 元数据过滤 | 全部 → 10%-50% | 倒排索引 / Bloom Filter | 1-5ms | 快速排除明显无关文档 |
| 粗召回 | 10万 → 1万 | BM25 + 向量检索(双编码器) | 10-50ms | 保证高召回率 |
| 精排序 | 1万 → 100 | 交叉编码器(Cross-Encoder) | 50-200ms | 提升精度 |
| LLM 重排 | 100 → 10 | LLM 打分 | 500ms-2s | 最终决策 |

### 级联公式

```
P(relevant | q) = P(survive_s1 | q) × P(survive_s2 | q, s1) × ... × P(survive_sn | q, s_n-1)
```

每一阶段的漏检率会累积，因此**粗召回阶段的召回率必须极高**（通常 >95%），否则后续阶段再精细也无法找回已丢失的相关文档。

## 背景与演进

### 单阶段检索（2019-2021）

最早的 RAG 系统使用单阶段检索：一个 BM25 或一个 embedding 向量检索搞定全部。简单直接，但问题明显：
- BM25 无法捕捉语义匹配
- 纯向量检索在大规模下精度不够
- 两者在"召回率"和"精度"上难以兼得

### 两阶段检索：Retrieve + Rerank（2022-2023）

```
[Query] → 双编码器检索 Top-100 → 交叉编码器重排 Top-10 → [Result]
```

这是多级检索的雏形。双编码器（Dual Encoder）负责宽召回，交叉编码器（Cross-Encoder）负责精排序。相比单阶段，Recall@10 提升 10-20%，但引入了一个额外的重排步骤。

### 多阶段检索（2023-2024）

随着 RAG 系统规模化，两阶段逐渐演变为多阶段：

```
Query → Metadata Filtering → Hybrid Retrieval → Reranking → LLM Refinement → Result
```

- **元数据过滤**：时间范围、文档类型、权限等属性的快速筛选
- **混合检索**：BM25 + 向量检索的融合，兼顾关键词和语义
- **RRF 融合**：Reciprocal Rank Fusion 合并多路结果
- **重排**：交叉编码器对 Top-N 重新打分
- **LLM 辅助**：用 LLM 对最终结果做上下文验证

### 当前趋势（2025-2026）

- **自适应阶段选择**：根据 query 复杂度动态决定使用多少阶段
- **路由式检索**：先分类 query 类型，路由到不同的检索管线
- **端到端可学习管线**：用强化学习或蒸馏优化各阶段的剪枝策略

## 核心矛盾

### 延迟 vs 精度

```
更多的阶段 = 更高的精度 + 更长的延迟
```

| 阶段数 | 召回率(Recall@10) | P99 延迟 | 适用范围 |
|--------|------------------|---------|---------|
| 1（仅向量检索） | 75-85% | 20ms | 实时聊天 |
| 2（检索 + 重排） | 85-92% | 100ms | 通用 QA |
| 3（过滤 + 检索 + 重排） | 90-95% | 200ms | 知识库搜索 |
| 4（过滤 + 混合检索 + 重排 + LLM） | 93-97% | 1-3s | 深度研究 |

### 在哪停下？—— 性价比递减

核心问题是：**在哪一阶段停下性价比最高？**

```
收益
 ↑
 │   Stage 1 ████████████████ (60% 收益, 10% 延迟)
 │   Stage 2 ████████████     (25% 收益, 40% 延迟)
 │   Stage 3 █████            (10% 收益, 30% 延迟)
 │   Stage 4 ██               (5% 收益, 20% 延迟)
 └──────────────────────────→ 累积延迟
```

通常 Stage 2 是性价比最高的点，Stage 3 在业务关键场景值得加，Stage 4 只对高价值查询使用。

## 主流优化方向

### 1. 级联检索（Cascading Retrieval）

将多个检索器串行级联，前一阶段的输出作为后一阶段的输入，每个阶段使用不同模型：

```
Stage 1: BM25（关键词匹配）       → 宽召回 Top-500
Stage 2: 双编码器向量检索（语义）  → Top-100
Stage 3: 交叉编码器重排（精细匹配）→ Top-10
```

### 2. 渐进式过滤（Progressive Filtering）

先做开销最小的过滤（元数据），再做中等开销的过滤（粗检索），最后做高开销的精排：

```python
def progressive_filter(query, corpus):
    # Step 1: 元数据剪枝（毫秒级）
    candidates = metadata_filter(query, corpus)
    # Step 2: BM25 粗筛（10ms 级）
    candidates = bm25_retrieve(query, candidates, top_k=500)
    # Step 3: 向量检索精筛（50ms 级）
    candidates = vector_retrieve(query, candidates, top_k=100)
    # Step 4: 交叉编码器重排（200ms 级）
    results = cross_encoder_rerank(query, candidates, top_k=10)
    return results
```

### 3. 阶段特定模型选择

每个阶段使用不同复杂度/成本的模型：

| 阶段 | 模型类型 | 推荐模型 | 维度 | 相对成本 |
|------|---------|---------|------|---------|
| 粗召回 | 稀疏检索 | BM25 | - | 1x |
| 粗召回 | 轻量双编码器 | e5-small / bge-small | 384 | 2x |
| 精排序 | 标准双编码器 | e5-base / bge-base | 768 | 5x |
| 重排 | 交叉编码器 | bge-reranker-v2 / Cohere Rerank | - | 20x |
| 最终排序 | LLM | GPT-4o / Claude | - | 100x+ |

### 4. 提前退出策略（Early Exit）

如果当前阶段检索结果置信度已经足够高，跳过后续阶段：

```python
def adaptive_retrieve(query, early_exit_threshold=0.95):
    # Stage 1: 粗召回
    results = coarse_retrieve(query, top_k=20)
    if results[0].score > early_exit_threshold:
        return results[:5]  # 置信度高，直接返回
    # Stage 2: 精排序
    results = rerank(query, results)
    if results[0].score > early_exit_threshold:
        return results[:5]
    # Stage 3: LLM 重排（仅低置信度时）
    return llm_rerank(query, results)
```

## 能力边界与结果边界

### 收益递减规律

```
阶段数 → 召回率增长曲线
1 stage:  80% recall @ 20ms
2 stages: 90% recall @ 100ms  (gain: +10%)
3 stages: 94% recall @ 300ms  (gain: +4%)
4 stages: 96% recall @ 1.2s   (gain: +2%)
5 stages: 97% recall @ 3s     (gain: +1%)
```

超过 3 个阶段后，收益增长急剧放缓，但延迟线性增长。

### 瓶颈分析

| 瓶颈 | 原因 | 缓解方案 |
|------|------|---------|
| 粗召回漏检 | 第一阶段召回率不足 | 使用混合检索 + 扩大 Top-K |
| 重排质量 | 交叉编码器对长文本不敏感 | 分段重排 + 聚合 |
| 级联误差累积 | 前阶段错误传递到后续 | 每阶段保持冗余（过采样） |
| 延迟不可控 | 最坏情况走完全部阶段 | 动态超时 + 提前退出 |

## 与单阶段检索的区别

| 维度 | 单阶段检索 | 多级检索 |
|------|-----------|---------|
| 架构复杂度 | 低 — 一个检索器搞定 | 高 — 多个组件串/并联 |
| 召回率 | 中等（70-85%） | 高（90-97%） |
| 精度(P@10) | 中等（60-75%） | 高（80-92%） |
| P99 延迟 | 低（10-30ms） | 中-高（100ms-3s） |
| 计算成本 | 低 — 一次向量计算 | 高 — 多阶段多次计算 |
| 可解释性 | 低 — 黑盒打分 | 高 — 每阶段可追溯 |
| 扩展性 | 随数据量增加精度下降快 | 可通过增加阶段维持精度 |
| 适用场景 | 实时性要求高、数据量小 | 质量要求高、数据量大 |
| 运维成本 | 低 | 中-高（需要调优每个阶段） |

## 核心优势

1. **资源效率最大化**：低成本的粗检索处理大部分候选，高成本的精细模型只作用于少量候选 —— 在有限预算下实现最优精度

2. **渐进式质量提升**：每个阶段专注于不同的匹配维度（关键词 → 语义 → 细粒度交互），互补性强

3. **灵活组合**：可自由插拔各阶段组件 —— 轻量场景只用 2 阶段，高要求场景用 4 阶段

4. **可观测性**：每个阶段的输出可单独审视，便于诊断漏检或误检

5. **容错性**：即使某阶段表现不佳，后续阶段可以弥补（前提是粗召回没有漏掉）

## 工程优化

### 1. 阶段超时（Stage Timeout）

为每个阶段设置独立超时，防止慢查询拖垮整体延迟：

```python
import asyncio

async def stage_with_timeout(stage_fn, timeout_ms: int, fallback_fn=None):
    """给每个阶段设置超时，超时则使用兜底策略"""
    try:
        result = await asyncio.wait_for(stage_fn(), timeout=timeout_ms / 1000)
        return result
    except asyncio.TimeoutError:
        return fallback_fn() if fallback_fn else []
```

### 2. 并行 vs 串行阶段

- **串行模式**（默认）：Stage1 → Stage2 → Stage3，候选集逐级缩小
- **并行模式**：多路检索同时执行，结果合并（适合混合检索场景）

```
串行: 过滤 → 向量检索 → 重排        总延迟 = 累加
并行: ┌─ BM25 ─┐                    总延迟 = max(各路)
      ├─ 向量A ─┼─ RRF 融合 → 重排
      └─ 向量B ┘
```

混合检索阶段内部可以并行，但重排阶段通常需要等全部结果到齐。

### 3. 动态阶段跳过

基于查询的早期置信度指标，动态跳过不必要的阶段：

```python
class AdaptiveStageSelector:
    def __init__(self):
        self.stage_configs = {
            "metadata_filter": {"latency_ms": 5,  "recall_impact": 0.1},
            "bm25":            {"latency_ms": 15, "recall_impact": 0.3},
            "vector_search":   {"latency_ms": 40, "recall_impact": 0.4},
            "rerank":          {"latency_ms": 150,"recall_impact": 0.2},
        }
    
    def select_stages(self, query: str, latency_budget_ms: int):
        """在延迟预算内选择最优的阶段组合"""
        # 简单实现：贪心选择收益/延迟比最高的阶段
        remaining = latency_budget_ms
        selected = ["metadata_filter"]  # 始终先做过滤
        
        candidates = sorted(
            [(k, v) for k, v in self.stage_configs.items() if k not in selected],
            key=lambda x: x[1]["recall_impact"] / x[1]["latency_ms"],
            reverse=True
        )
        
        for name, config in candidates:
            if config["latency_ms"] <= remaining:
                selected.append(name)
                remaining -= config["latency_ms"]
        
        return selected
```

### 4. 过采样（Over-sampling）

每个阶段保留的候选数应多于最终所需数，防止级联误差累积：

```
粗召回: Top-1000 → 向量检索: Top-200 → 重排: Top-20 → 最终: Top-10
```

每个阶段的剪枝率应控制在 5-10 倍以内。

### 5. 结果缓存

对高频查询缓存多级检索的最终结果，避免重复执行全管线：

```python
from functools import lru_cache
import hashlib

class MultiStageCache:
    def __init__(self, cache_size: int = 1000):
        self.cache = lru_cache(maxsize=cache_size)(self._uncached_retrieve)
    
    def _hash_query(self, query: str) -> str:
        return hashlib.md5(query.encode()).hexdigest()
    
    def retrieve(self, query: str):
        return self.cache(self._hash_query(query))
    
    def _uncached_retrieve(self, query_hash: str):
        # 实际的多级检索逻辑
        pass
```

## 场景适配指南

| 场景 | 数据规模 | 延迟要求 | 推荐阶段数 | 推荐配置 |
|------|---------|---------|-----------|---------|
| 实时聊天助手 | < 100 万 | < 50ms | 1-2 | 向量检索 + 可选重排 |
| 企业知识库搜索 | 100 万-1000 万 | < 200ms | 2-3 | 元数据过滤 + 混合检索 + 重排 |
| 学术文献检索 | 1000 万+ | < 500ms | 3 | 过滤 + BM25 + 向量检索 + 交叉编码器 |
| 法律合同审查 | 10 万-100 万 | < 1s | 3-4 | 过滤 + 混合检索 + 重排 + LLM 验证 |
| 电商商品搜索 | 100 万-1 亿 | < 100ms | 2-3 | 类目过滤 + 向量检索 + 重排 |
| 深度研究助手 | 100 万+ | < 5s | 4 | 过滤 + 多路检索 + 重排 + LLM 精排 |
| 代码库检索 | 10 万-100 万 | < 200ms | 2 | 结构过滤 + 向量检索 |
| 多模态检索 | 10 万-1000 万 | < 1s | 3 | 文本过滤 + 向量检索 + 跨模态重排 |

## 示例代码：生产级多级检索管线

```python
from typing import List, Dict, Optional, Callable
import asyncio
import time
from dataclasses import dataclass
import numpy as np


@dataclass
class RetrievalResult:
    doc_id: str
    text: str
    score: float
    metadata: dict
    stage: str  # 哪个阶段的产出


class Stage:
    """单个检索阶段基类"""
    
    def __init__(self, name: str, timeout_ms: int = 100):
        self.name = name
        self.timeout_ms = timeout_ms
    
    def retrieve(self, query: str, candidates: List[RetrievalResult]) -> List[RetrievalResult]:
        raise NotImplementedError


class MetadataFilterStage(Stage):
    """元数据过滤阶段"""
    
    def __init__(self, metadata_index: dict):
        super().__init__("metadata_filter", timeout_ms=10)
        self.metadata_index = metadata_index  # {field: {value: [doc_ids]}}
    
    def retrieve(self, query: str, candidates: List[RetrievalResult]) -> List[RetrievalResult]:
        # 从 query 中提取过滤条件
        filters = self._extract_filters(query)
        
        if not filters:
            return candidates
        
        # 对每个候选文档检查是否满足所有过滤条件
        result = []
        for doc in candidates:
            if all(doc.metadata.get(k) == v for k, v in filters.items()):
                result.append(doc)
        
        return result
    
    def _extract_filters(self, query: str) -> dict:
        """从查询中提取元数据过滤条件（简化实现）"""
        filters = {}
        # 实际实现会用 NER 或规则提取
        if "2024" in query:
            filters["year"] = "2024"
        if "en" in query or "english" in query.lower():
            filters["language"] = "en"
        return filters


class VectorSearchStage(Stage):
    """向量检索阶段"""
    
    def __init__(self, embeddings, index, top_k: int = 100):
        super().__init__("vector_search", timeout_ms=50)
        self.embeddings = embeddings
        self.index = index
        self.top_k = top_k
    
    def retrieve(self, query: str, candidates: List[RetrievalResult]) -> List[RetrievalResult]:
        query_emb = self.embeddings.encode(query)
        
        if candidates:
            # 在候选子集上检索
            doc_ids = [c.doc_id for c in candidates]
            # 实际中会使用子索引或暴力搜索
            results = self._search_subset(query_emb, doc_ids, self.top_k)
        else:
            # 全量检索
            distances, indices = self.index.search(
                query_emb.reshape(1, -1), self.top_k
            )
            results = [
                RetrievalResult(
                    doc_id=str(idx),
                    text=f"doc_{idx}",
                    score=float(1.0 - dist),
                    metadata={},
                    stage=self.name
                )
                for idx, dist in zip(indices[0], distances[0])
            ]
        
        return results
    
    def _search_subset(self, query_emb: np.ndarray, doc_ids: List[str], top_k: int):
        """在候选子集上搜索（简化实现）"""
        # 实际实现会构建临时索引或暴力扫描
        pass


class RerankStage(Stage):
    """交叉编码器重排阶段"""
    
    def __init__(self, cross_encoder, top_k: int = 10):
        super().__init__("rerank", timeout_ms=200)
        self.cross_encoder = cross_encoder
        self.top_k = top_k
    
    def retrieve(self, query: str, candidates: List[RetrievalResult]) -> List[RetrievalResult]:
        if len(candidates) <= self.top_k:
            return candidates
        
        pairs = [(query, c.text) for c in candidates]
        scores = self.cross_encoder.predict(pairs)
        
        # 按交叉编码器分数重新排序
        ranked = sorted(
            zip(candidates, scores),
            key=lambda x: x[1],
            reverse=True
        )
        
        return [
            RetrievalResult(
                doc_id=c.doc_id,
                text=c.text,
                score=float(s),
                metadata=c.metadata,
                stage=self.name
            )
            for c, s in ranked[:self.top_k]
        ]


class CascadingRetriever:
    """级联检索器：管理多级检索管线"""
    
    def __init__(self, stages: List[Stage], early_exit_score: float = 0.95):
        self.stages = stages
        self.early_exit_score = early_exit_score
    
    async def retrieve(self, query: str, initial_candidates: Optional[List[RetrievalResult]] = None) -> List[RetrievalResult]:
        pipeline_start = time.time()
        candidates = initial_candidates or []
        
        for stage in self.stages:
            stage_start = time.time()
            
            try:
                candidates = await asyncio.wait_for(
                    asyncio.to_thread(stage.retrieve, query, candidates),
                    timeout=stage.timeout_ms / 1000
                )
            except asyncio.TimeoutError:
                # 阶段超时，使用上一阶段的输出
                print(f"Stage {stage.name} timed out, skipping")
                continue
            
            stage_time = (time.time() - stage_start) * 1000
            print(f"Stage {stage.name}: {len(candidates)} candidates in {stage_time:.1f}ms")
            
            # 提前退出检查
            if candidates and candidates[0].score >= self.early_exit_score:
                print(f"Early exit at stage {stage.name} (score={candidates[0].score:.3f})")
                break
            
            if not candidates:
                print(f"Empty candidates after stage {stage.name}")
                break
        
        total_time = (time.time() - pipeline_start) * 1000
        print(f"Total pipeline: {total_time:.1f}ms")
        return candidates[:10]  # 最终返回 Top-10


# ============================================================
# 使用示例
# ============================================================
async def main():
    # 构建阶段管线
    stages = [
        MetadataFilterStage(metadata_index={}),
        VectorSearchStage(embeddings=None, index=None, top_k=500),
        RerankStage(cross_encoder=None, top_k=20),
    ]
    
    retriever = CascadingRetriever(stages, early_exit_score=0.92)
    
    # 执行检索
    results = await retriever.retrieve("2024年人工智能发展趋势")
    
    for r in results:
        print(f"[{r.stage}] {r.doc_id}: score={r.score:.4f} | {r.text[:50]}...")


# ============================================================
# 混合检索：多路并行 + RRF 融合
# ============================================================
def reciprocal_rank_fusion(results_list: List[List[RetrievalResult]], k: int = 60) -> List[RetrievalResult]:
    """Reciprocal Rank Fusion：多路检索结果融合"""
    score_dict = {}
    
    for results in results_list:
        for rank, doc in enumerate(results):
            doc_id = doc.doc_id
            # RRF 分数 = 1 / (k + rank)
            score_dict[doc_id] = score_dict.get(doc_id, 0) + 1.0 / (k + rank + 1)
    
    # 按融合分数排序
    ranked = sorted(score_dict.items(), key=lambda x: x[1], reverse=True)
    return [r for r, _ in ranked]


class HybridRetrievalStage(Stage):
    """混合检索阶段：多路并行 BM25 + 向量 + RRF 融合"""
    
    def __init__(self, bm25_retriever, vector_retriever, top_k: int = 100):
        super().__init__("hybrid_retrieval", timeout_ms=80)
        self.bm25 = bm25_retriever
        self.vector = vector_retriever
        self.top_k = top_k
    
    def retrieve(self, query: str, candidates: List[RetrievalResult]) -> List[RetrievalResult]:
        # 并行执行 BM25 和向量检索
        bm25_results = self.bm25.retrieve(query, candidates)
        vector_results = self.vector.retrieve(query, candidates)
        
        # RRF 融合
        fused_ids = reciprocal_rank_fusion([bm25_results, vector_results])
        
        # 取 Top-K
        id_set = set(fused_ids[:self.top_k])
        result_dict = {r.doc_id: r for r in bm25_results + vector_results}
        
        return [result_dict[doc_id] for doc_id in fused_ids if doc_id in id_set]


# ============================================================
# 自适应阶段选择器
# ============================================================
class AdaptiveRetriever:
    """根据查询复杂度和延迟预算自适应选择阶段"""
    
    def __init__(self, all_stages: Dict[str, Stage]):
        self.all_stages = all_stages
        
        # 阶段性能画像（通过基准测试获得）
        self.stage_profiles = {
            "metadata": {"latency_ms": 5,  "recall_gain": 0.05},
            "bm25":     {"latency_ms": 15, "recall_gain": 0.25},
            "vector":   {"latency_ms": 40, "recall_gain": 0.35},
            "hybrid":   {"latency_ms": 60, "recall_gain": 0.40},
            "rerank":   {"latency_ms": 150,"recall_gain": 0.20},
        }
    
    def _estimate_query_complexity(self, query: str) -> float:
        """估计查询复杂度（0-1），用于决定是否使用更多阶段"""
        length = len(query)
        has_specific = any(c in query for c in ["\"", "'", "：", "、"])
        return min(1.0, (length / 50) + (0.2 if has_specific else 0))
    
    def _select_stages(self, query: str, latency_budget_ms: int) -> List[str]:
        """在延迟预算内选择最优阶段组合（贪心算法）"""
        complexity = self._estimate_query_complexity(query)
        
        # 简单查询使用较少阶段
        if complexity < 0.3:
            return ["vector"]
        if complexity < 0.6:
            return ["bm25", "vector", "rerank"]
        
        # 复杂查询使用更多阶段
        remaining = latency_budget_ms
        selected = []
        
        # 按 recall_gain / latency 排序
        ranked = sorted(
            self.stage_profiles.items(),
            key=lambda x: x[1]["recall_gain"] / max(x[1]["latency_ms"], 1),
            reverse=True
        )
        
        for name, profile in ranked:
            if profile["latency_ms"] <= remaining:
                selected.append(name)
                remaining -= profile["latency_ms"]
        
        return selected
    
    async def retrieve(self, query: str, latency_budget_ms: int = 200) -> List[RetrievalResult]:
        stage_names = self._select_stages(query, latency_budget_ms)
        print(f"Adaptive: selected stages = {stage_names}")
        
        candidates = []
        for name in stage_names:
            stage = self.all_stages[name]
            candidates = await asyncio.wait_for(
                asyncio.to_thread(stage.retrieve, query, candidates),
                timeout=stage.timeout_ms / 1000
            )
        
        return candidates[:10]
```
