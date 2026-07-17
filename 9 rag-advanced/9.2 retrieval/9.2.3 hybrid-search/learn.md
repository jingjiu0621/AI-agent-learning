# 9.2.3 hybrid-search — 混合检索：稠密 + 稀疏的融合检索

## 简单介绍

混合检索（Hybrid Search）是同时使用**稀疏检索**（如 BM25、TF-IDF）和**稠密检索**（Dense Vector / Embedding 语义搜索）两种方式，并将结果融合排序的检索策略。单纯依赖稠密检索时，模型可能丢失精确关键词匹配能力（如专有名词、缩写、ID 号）；单纯依赖稀疏检索时，模型无法理解同义词和语义变体。混合检索试图同时保留两者的优势。

核心思想是：**让 keyword exact match 兜底精确命中，让 dense semantic search 兜底语义泛化，两者互为补充**。

## 基本原理

### 工作流程

```
用户 Query
    │
    ├──→ BM25/稀疏检索 ──→ 得到稀疏结果列表 S = [d₁, d₂, ..., dₖ]
    │
    └──→ Dense/稠密检索 ──→ 得到稠密结果列表 D = [d₁, d₂, ..., dₖ]
    │
    └──→ 融合策略 ──→ 合并/重排序 → 最终 Top-K
         ├── RRF（倒数排序融合）
         ├── 加权分数融合（Score Normalization + Weighted Sum）
         └── 学习型融合（Learning to Rank / 训练排序器）
```

### 三种核心融合策略

| 策略 | 实现难度 | 确定性 | 效果上限 | 适用场景 |
|------|---------|--------|---------|---------|
| RRF（倒数排序融合） | ⭐ 低 | ⭐⭐⭐⭐⭐ 确定 | ⭐⭐⭐ 中等 | 通用兜底，生产最常用 |
| 加权分数融合 | ⭐⭐⭐ 中 | ⭐⭐⭐ 依赖权重 | ⭐⭐⭐⭐ 较高 | 有验证集调权的场景 |
| 学习型融合 | ⭐⭐⭐⭐⭐ 高 | ⭐⭐ 依赖训练 | ⭐⭐⭐⭐⭐ 最高 | 数据充足的专业场景 |

### 1. RRF（Reciprocal Rank Fusion）

RRF 不关心原始分数，只关心文档在各自结果列表中的**排名位置**。每条文档的融合分数 = 各列表倒数排名的和。

```python
def reciprocal_rank_fusion(
    sparse_results: list[tuple[str, float]],  # [(doc_id, score), ...]
    dense_results: list[tuple[str, float]],
    k: int = 60  # RRF 常数，通常 60
) -> list[tuple[str, float]]:
    """RRF 融合：按排名位置而非原始分数融合"""
    rrf_scores: dict[str, float] = {}
    
    # 稀疏部分：排名从 1 开始
    for rank, (doc_id, _) in enumerate(sparse_results, start=1):
        rrf_scores[doc_id] = rrf_scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    
    # 稠密部分：排名从 1 开始
    for rank, (doc_id, _) in enumerate(dense_results, start=1):
        rrf_scores[doc_id] = rrf_scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    
    # 按 RRF 分数降序排列
    sorted_docs = sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)
    return sorted_docs
```

**RRF 关键参数 `k`**：控制排名衰减速度。`k` 越小，靠前文档的权重差异越大（即第一名 vs 第二名差距更大）。通常 `k=60` 是各数据集上的稳健默认值。

### 2. 分数归一化 + 加权融合

当需要直接使用原始分数时，必须先归一化到同一尺度（BM25 分数和 cosine similarity 的分布完全不同）。

```python
import numpy as np
from typing import List, Tuple

def min_max_normalize(scores: List[float]) -> List[float]:
    """Min-Max 归一化到 [0, 1]"""
    scores = np.array(scores, dtype=float)
    min_s, max_s = scores.min(), scores.max()
    if max_s == min_s:
        return np.zeros_like(scores).tolist()
    return ((scores - min_s) / (max_s - min_s)).tolist()

def gaussian_normalize(scores: List[float]) -> List[float]:
    """高斯归一化：假设分数服从正态分布"""
    scores = np.array(scores, dtype=float)
    mean, std = scores.mean(), scores.std()
    if std == 0:
        return np.zeros_like(scores).tolist()
    # Z-score 后映射到 [0, 1]
    z = (scores - mean) / std
    return (1.0 / (1.0 + np.exp(-z))).tolist()  # Sigmoid 映射

def rank_based_normalize(scores: List[float]) -> List[float]:
    """基于排名的归一化：1 - (rank - 1) / total"""
    n = len(scores)
    return [1.0 - i / n for i in range(n)]

def weighted_combination(
    sparse_results: List[Tuple[str, float]],
    dense_results: List[Tuple[str, float]],
    alpha: float = 0.3,          # 稀疏权重
    normalize_fn: str = "minmax"  # 归一化方法
) -> List[Tuple[str, float]]:
    """加权融合：稀疏 × alpha + 稠密 × (1-alpha)"""
    norm_fns = {
        "minmax": min_max_normalize,
        "gaussian": gaussian_normalize,
        "rank": rank_based_normalize,
    }
    norm_fn = norm_fns.get(normalize_fn, min_max_normalize)
    
    # 提取并归一化分数
    sparse_ids = [doc[0] for doc in sparse_results]
    dense_ids = [doc[0] for doc in dense_results]
    
    sparse_scores = norm_fn([doc[1] for doc in sparse_results])
    dense_scores = norm_fn([doc[1] for doc in dense_results])
    
    # 构建分数字典
    combined: dict[str, float] = {}
    for i, doc_id in enumerate(sparse_ids):
        combined[doc_id] = combined.get(doc_id, 0.0) + alpha * sparse_scores[i]
    for i, doc_id in enumerate(dense_ids):
        combined[doc_id] = combined.get(doc_id, 0.0) + (1 - alpha) * dense_scores[i]
    
    return sorted(combined.items(), key=lambda x: x[1], reverse=True)
```

### 3. 完整 Weighted Hybrid Retriever 类

```python
from typing import List, Dict, Optional, Callable, Tuple
import numpy as np

class WeightedHybridRetriever:
    """可配置的加权混合检索器"""
    
    def __init__(
        self,
        sparse_retriever: Callable,
        dense_retriever: Callable,
        alpha: float = 0.3,
        normalize: str = "minmax",
        top_k: int = 10,
    ):
        self.sparse_retriever = sparse_retriever
        self.dense_retriever = dense_retriever
        self.alpha = alpha
        self.normalize = normalize
        self.top_k = top_k
    
    def retrieve(self, query: str) -> List[Dict]:
        """混合检索入口"""
        # 并行调用两种检索（实际生产中应并发执行）
        sparse_out = self.sparse_retriever(query, top_k=self.top_k * 2)
        dense_out = self.dense_retriever(query, top_k=self.top_k * 2)
        
        # 融合
        fused = weighted_combination(
            [(d["id"], d["score"]) for d in sparse_out],
            [(d["id"], d["score"]) for d in dense_out],
            alpha=self.alpha,
            normalize_fn=self.normalize
        )
        
        # 重建文档对象
        doc_map = {d["id"]: d for d in sparse_out + dense_out}
        result = []
        for doc_id, score in fused[:self.top_k]:
            if doc_id in doc_map:
                doc = dict(doc_map[doc_id])
                doc["hybrid_score"] = score
                result.append(doc)
        return result
    
    def set_alpha(self, alpha: float):
        """动态调整稀疏/稠密权重"""
        self.alpha = max(0.0, min(1.0, alpha))
```

## 背景与演进

- **纯稀疏时代（1990s-2010s）**：BM25 / TF-IDF 是搜索引擎的基石，依赖关键词精确匹配。词袋模型，无法理解语义。

- **纯稠密检索兴起（2019-2022）**：BERT 等预训练模型 + Dense Passage Retrieval (DPR) 证明语义检索在大规模语料上的有效性。Cosine similarity 在语义匹配上远超 BM25。

- **发现稠密的盲区（2022-2023）**：实际生产中发现稠密检索对**实体精确匹配**（产品 SKU、人名、法律条文编号）、**低频词**、**领域术语**表现不稳定。Elasticsearch 社区的讨论表明很多生产系统退回到了"BM25 + 向量"双轨方案。

- **混合检索成为生产标准（2023-2024）**：RRF 被大量采用，Elasticsearch 原生支持 `knn + query` 混合搜索，Pinecone、Weaviate、Qdrant 等向量数据库纷纷内置混合检索 API。

- **自适应融合时代（2024-2025）**：不再用固定权重，而是根据 query 特征动态选择融合模式或调整 alpha。学习型排序器（Learning to Rank）被引入融合阶段。

- **当前趋势（2025-2026）**：混合检索已成为生产 RAG 系统的**默认标配**。纯稠密或纯稀疏只在特定约束下使用（如纯语义问答只用稠密、代码搜索依赖精确符号匹配可只用 BM25）。主要优化方向转为**自适应权重**、**query-aware 路由**和**延迟优化**。

## 核心矛盾

**分数分布不匹配（Normalization Mismatch）**：

```
BM25 分数分布：
  - 无上界（理论上可到很大值）
  - 随文档长度变化
  - 同一 query 的分数可比性差

Dense 分数分布：
  - 通常在 [-1, 1] 或 [0, 1] 范围（cosine similarity）
  - 嵌入空间中的相对距离有效
  - 所有文档分数在同一量级

直接加权求和 → 某一方的分数主导融合结果
```

详细矛盾维度：

| 矛盾 | 说明 | 影响 |
|------|------|------|
| 分数尺度不同 | BM25 无上界 vs Cosine 有界 [0,1] | 直接加权会偏向 BM25 |
| 排名方差不同 | BM25 对高频词敏感，分数波动大 | 融合结果不稳定 |
| 覆盖率不同 | 稀疏只在精确匹配处产生分数，稠密在所有文档上都有分数 | 融合时稀疏结果稀疏性导致权重分配不均匀 |
| 最优权重不固定 | 不同 query、不同领域的最佳 alpha 各不相同 | 固定权重泛化能力有限 |

## 主流优化方向

### 1. 自适应权重（Adaptive / Context-aware Alpha）

根据 query 的类型动态调整 alpha：关键词明确的 query 加大稀疏权重，语义开放型 query 加大稠密权重。

```python
class AdaptiveHybridRetriever(WeightedHybridRetriever):
    """基于 query 类型自动调整权重的混合检索器"""
    
    def _classify_query(self, query: str) -> str:
        """判断 query 属于哪种类型"""
        # 启发式规则
        import re
        
        # 包含特定符号/格式 → 实体查询
        if re.search(r'[A-Z]{2,}\d+', query):  # 产品编号: SKU-123, ABC789
            return "entity"
        if re.search(r'^[A-Z][a-z]+ [A-Z][a-z]+$', query):  # 人名
            return "entity"
        if re.search(r'\d{4}-\d{2}-\d{2}', query):  # 日期
            return "factoid"
        if len(query.split()) <= 3:  # 短查询
            return "keyword"
        if len(query) > 50:  # 长查询通常更具语义性
            return "semantic"
        
        return "general"
    
    def _query_to_alpha(self, query: str) -> float:
        """根据 query 类型返回 alpha（稀疏权重）"""
        qtype = self._classify_query(query)
        
        alpha_map = {
            "entity":  0.7,   # 实体查询 → 偏重 BM25 精确匹配
            "keyword": 0.6,   # 关键词查询 → 偏重匹配
            "factoid": 0.5,   # 事实性查询 → 均衡
            "general": 0.3,   # 通用查询 → 偏重语义
            "semantic": 0.2,  # 语义查询 → 偏重稠密
        }
        return alpha_map.get(qtype, 0.3)
    
    def retrieve(self, query: str) -> List[Dict]:
        """自动调整权重后进行检索"""
        alpha = self._query_to_alpha(query)
        self.set_alpha(alpha)
        return super().retrieve(query)
```

### 2. Learning to Rank 融合（LTR Fusion）

训练一个轻量级排序模型（LambdaRank、LightGBM），输入特征包括：BM25 分数、稠密分数、RRF 分数、文档长度、query-doc 共现特征等，输出最终的排序分数。

```python
# 伪代码：LTR 特征工程
def build_ltr_features(query: str, doc: dict, bm25_score: float, dense_score: float) -> List[float]:
    """构建 Learning to Rank 特征向量"""
    return [
        bm25_score,
        dense_score,
        bm25_score * dense_score,               # 交叉特征
        len(query.split()),                      # query 长度
        len(doc["text"].split()),                # 文档长度
        len(set(query.lower().split()) & set(doc["text"].lower().split())),  # 共现词数
        1.0 / (1.0 + abs(bm25_score - dense_score)),  # 分数一致性
        # ... 还可加入 query 类型 one-hot、文档来源等
    ]
```

### 3. Query-Aware 模式选择

不融合两路结果，而是根据 query 判断使用哪一种检索模式，更节省计算资源。

```
Query
  │
  ├── Query 分类器
  │     ├── "实体密集" → 纯 BM25
  │     ├── "语义开放" → 纯稠密
  │     ├── "混合需要" → 全混合检索
  │     └── "快速模式" → 仅 BM25（低延迟兜底）
  │
  └──→ 路由到对应检索器
```

## 能力边界与结果边界

### 混合检索最适合的场景

| 场景 | 说明 | 混合检索提升幅度 |
|------|------|-----------------|
| 专有名词/实体精确匹配 | 产品名、人名、地名、编号 | ⭐⭐⭐⭐⭐ 大幅提升 |
| 同义词/近义表达 | "car" ↔ "automobile"、"买方" ↔ "购买方" | ⭐⭐⭐⭐ 显著提升 |
| 中英混合/多语言 | 术语混杂的场景 | ⭐⭐⭐⭐ 显著提升 |
| 长短 Query 混合 | 既有短关键词又有长问句 | ⭐⭐⭐ 中等提升 |
| 领域特定术语 | 法律、医疗、金融等特有称谓 | ⭐⭐⭐⭐⭐ 大幅提升 |

### 纯稠密就够用的场景

| 场景 | 说明 | 原因 |
|------|------|------|
| 纯语义相似度问答 | "怎样提高睡眠质量" → 语义相近段落 | 无精确匹配需求 |
| 大规模开放域 QA | 无需精确关键词 | 语义空间全覆盖 |
| 同义改写丰富的知识库 | 知识库本身已覆盖大量表达变体 | 稠密检索已足够泛化 |

### 纯稀疏就够用的场景

| 场景 | 说明 | 原因 |
|------|------|------|
| 代码搜索 | 符号名、函数名精确匹配 | 语义检索反而引入噪声 |
| 产品 SKU 查询 | ABC-123-XYZ 精确命中 | 不需要语义理解 |
| 法律条文引用 | "第 X 条第 Y 款" 精确匹配 | 语义检索可能匹配到类似但错误条款 |
| 日志检索 | ERROR-CODE-404 精确匹配 | 关键词精确性比语义更重要 |

## 与纯稠密/纯稀疏的区别

| 维度 | 纯稀疏（BM25） | 纯稠密（Dense） | 混合检索 |
|------|---------------|----------------|---------|
| 检索精度（精确匹配） | ⭐⭐⭐⭐⭐ 最高 | ⭐⭐ 不稳定 | ⭐⭐⭐⭐ 高 |
| 检索精度（语义匹配） | ⭐⭐ 几乎无效 | ⭐⭐⭐⭐⭐ 最高 | ⭐⭐⭐⭐ 高 |
| 延迟 | ⭐⭐⭐⭐⭐ 低（毫秒级） | ⭐⭐⭐ 中（含编码时间） | ⭐⭐ 中高（两路+融合） |
| 索引构建成本 | ⭐⭐⭐⭐⭐ 几乎为零 | ⭐⭐ 需生成 embedding | ⭐⭐ 两者都要 |
| 存储开销 | ⭐⭐⭐⭐ 倒排索引较小 | ⭐⭐ 向量索引大 | ⭐ 两套索引 |
| OOV（未登录词）处理 | ⭐ 无法处理 | ⭐⭐⭐⭐⭐ 可处理 | ⭐⭐⭐⭐ 好 |
| 冷启动能力 | ⭐⭐⭐⭐⭐ 无需训练 | ⭐⭐ 需要编码模型 | ⭐⭐ 依赖编码模型 |
| 可解释性 | ⭐⭐⭐⭐⭐ 关键词可追溯 | ⭐⭐ 黑盒向量 | ⭐⭐⭐ 稀疏部分可解释 |
| 实现复杂度 | ⭐⭐⭐⭐⭐ 极简单 | ⭐⭐⭐ 中等 | ⭐⭐ 较复杂 |

## 核心优势

- **鲁棒性（Robustness）**：无论 query 是关键词密集还是语义开放，混合检索都有保障。稀疏在精确匹配上兜底，稠密在语义泛化上兜底，两者互补使整体检索质量在各种 query 类型上都稳定。

- **覆盖最全面**：混合检索能同时覆盖"用户忘掉精确术语但记得语义"和"用户给出精确产品编号"两种极端情况，单一检索方式无法做到。

- **实践中效果提升显著**：在 MTEB 等基准上，混合检索（BM25 + Dense + RRF）在 Recall@K 上通常比单一方式高 5-15%。在专有名词多的领域（法律、医疗、电商），提升幅度可达 20%+。

- **工程可渐进式集成**：可在保留原有 BM25 索引的基础上，逐步添加向量索引和混合融合逻辑，无需推翻重建。

## 工程优化

### 1. BM25 预计算 + 缓存

```python
class CachedHybridRetriever:
    """带缓存的混合检索器"""
    
    def __init__(self, sparse_retriever, dense_retriever, cache_ttl: int = 3600):
        self.sparse = sparse_retriever
        self.dense = dense_retriever
        self.cache: dict[str, tuple] = {}  # query → (稀疏结果, 稠密结果, 时间戳)
        self.cache_ttl = cache_ttl
    
    def retrieve(self, query: str) -> List[Dict]:
        # 1. 查缓存（短时内相同 query 直接返回）
        import time
        now = time.time()
        if query in self.cache:
            sparse_out, dense_out, ts = self.cache[query]
            if now - ts < self.cache_ttl:
                return self._fuse(sparse_out, dense_out)
        
        # 2. 并行执行两路检索
        import asyncio
        async def parallel_retrieve():
            sparse_task = asyncio.to_thread(self.sparse, query, top_k=20)
            dense_task = asyncio.to_thread(self.dense, query, top_k=20)
            return await asyncio.gather(sparse_task, dense_task)
        
        sparse_out, dense_out = asyncio.run(parallel_retrieve())
        
        # 3. 缓存结果
        self.cache[query] = (sparse_out, dense_out, now)
        
        # 4. 融合返回
        return self._fuse(sparse_out, dense_out)
```

### 2. 延迟预算分配

混合检索最核心的工程挑战是**延迟翻倍**。常用优化手段：

```
总延迟预算 = 300ms（SLA）
    │
    ├── 两路并行：Max(稀疏延迟, 稠密延迟) + 融合延迟
    │   └── 典型：Max(30ms BM25, 80ms 向量搜索) + 1ms RRF ≈ 81ms
    │
    ├── 串行兜底：先跑稀疏 + 快速判断是否需要稠密补充
    │   └── 如果 BM25 Top-1 分数 > 阈值 → 跳过稠密
    │
    ├── 截断召回：两路都只召回 top-50 再融合取 top-10
    │   └── BM25 召回 50 条 vs 1000 条对排名影响很小，但延迟减半
    │
    └── 向量量化（PQ / Scalar Quantization）降低向量搜索延迟
        └── 4-bit 量化牺牲 ~1% 精度换取 4x 性能提升
```

### 3. 批量预处理 BM25 索引

```python
# BM25 索引可预计算持久化，避免每次启动重建
import pickle
from rank_bm25 import BM25Okapi

class PersistentBM25Index:
    """持久化 BM25 索引，避免重复建索引"""
    
    def __init__(self, index_path: str):
        self.index_path = index_path
        self.bm25 = None
        self.documents = []
    
    def build_or_load(self, documents: List[str]):
        import os
        if os.path.exists(self.index_path):
            with open(self.index_path, "rb") as f:
                self.bm25, self.documents = pickle.load(f)
        else:
            self.documents = documents
            tokenized = [doc.split() for doc in documents]
            self.bm25 = BM25Okapi(tokenized)
            with open(self.index_path, "wb") as f:
                pickle.dump((self.bm25, self.documents), f)
    
    def search(self, query: str, top_k: int = 10):
        tokenized_query = query.split()
        scores = self.bm25.get_scores(tokenized_query)
        top_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)[:top_k]
        return [(i, scores[i]) for i in top_indices]
```

## 场景适配指南

| 场景 | 推荐融合策略 | 推荐 alpha | 推荐归一化 | 说明 |
|------|-------------|-----------|-----------|------|
| 客服知识库 QA | RRF（k=60） | — | — | 实体精准 + 语义泛化双重要求 |
| 法律合同审查 | 加权融合 | 0.6~0.7（偏稀疏） | Min-Max | 条文编号精确匹配关键 |
| 医疗文献检索 | 加权融合 | 0.3~0.4（偏稠密） | 高斯归一化 | 同义术语多，语义匹配更重要 |
| 电商产品搜索 | 自适应加权 | 动态 alpha | Rank-based | 混合 query 类型多（SKU/品牌/描述） |
| 代码库检索 | 纯稀疏（BM25） | 1.0 | — | 符号名精确匹配即可 |
| 开放域问答 | RRF | — | — | 混合类型，RRF 鲁棒性最好 |
| 企业内部文档 | Learning to Rank | — | — | 有足够的点击/反馈数据 |
| 实时搜索（低延迟） | 仅 BM25 + LTR Flighting | 1.0 | — | 混合太慢时降级 |

## 示例代码：生产级混合检索管线

### LangChain 混合检索配置

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings

# 1. 构建 BM25 检索器
bm25_retriever = BM25Retriever.from_texts(documents)
bm25_retriever.k = 10

# 2. 构建稠密检索器
vectorstore = Chroma.from_texts(documents, OpenAIEmbeddings())
dense_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})

# 3. LangChain EnsembleRetriever（内置 RRF）
hybrid_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, dense_retriever],
    weights=[0.3, 0.7],      # 加权 RRF（可选）
    c=60                      # RRF 常数
)

# 使用
results = hybrid_retriever.invoke("如何配置混合检索？")
```

### Elasticsearch 原生混合搜索（kNN + Query）

```python
# Elasticsearch 原生支持 hybrid search（ES 8.8+）
ES_HYBRID_QUERY = {
    "retriever": {
        "rrf": {
            "retrievers": [
                {
                    "standard": {
                        "query": {
                            "match": {
                                "content": {
                                    "query": "混合检索配置方法",
                                    "boost": 0.3  # 稀疏权重
                                }
                            }
                        }
                    }
                },
                {
                    "knn": {
                        "field": "content_embedding",
                        "query_vector": [...],  # query embedding
                        "k": 10,
                        "num_candidates": 100,
                        "boost": 0.7  # 稠密权重
                    }
                }
            ],
            "rank_window_size": 50,
            "rank_constant": 60
        }
    }
}
```

### Weaviate 混合搜索 API

```python
# Weaviate 原生支持 hybrid search
import weaviate

client = weaviate.connect_to_local()

response = client.collections.get("Document").query.hybrid(
    query="混合检索如何工作",
    alpha=0.5,            # 0=纯稠密, 1=纯稀疏
    limit=10,
    fusion_type="rrf",    # 或 "ranked"
)

for obj in response.objects:
    print(obj.properties["content"])
```

### Pinecone 混合搜索

```python
# Pinecone 通过 metadata 过滤 + 向量搜索实现近似混合
import pinecone

pc = pinecone.Pinecone(api_key="...")
index = pc.Index("my-index")

# Pinecone 本身是纯稠密，混合通过稀疏向量 API 实现
# 使用 sparse_values 参数
query_results = index.query(
    vector=dense_embedding,           # 稠密向量
    sparse_vector=bm25_sparse_vec,    # BM25 稀疏向量表示
    top_k=10,
    alpha=0.5,                        # 混合比例
)
```

### Qdrant 混合搜索

```python
# Qdrant 支持 named vectors + payload 过滤实现混合
from qdrant_client import QdrantClient
from qdrant_client.models import (
    NamedVector, NamedSparseVector,
    FusionQuery, Fusion, SearchRequest
)

client = QdrantClient("localhost", port=6333)

# Qdrant 支持 FusionQuery 做混合检索
search_result = client.search_batch(
    collection_name="my_collection",
    requests=[
        SearchRequest(
            vector=NamedVector(
                name="dense",
                vector=dense_embedding
            ),
            limit=10,
            with_payload=True,
        ),
    ]
)

# 或者在 Qdrant 2.10+ 中使用 Fusion API
fusion_result = client.fusion(
    collection_name="my_collection",
    prefetch=[
        SearchRequest(
            vector=NamedVector(name="dense", vector=dense_embedding),
            limit=20,
        ),
        SearchRequest(
            sparse=NamedSparseVector(name="bm25", vector=sparse_vector),
            limit=20,
        ),
    ],
    query=FusionQuery(fusion=Fusion.RRF),  # RRF or DBSF
    limit=10,
)
```

## 示例代码：生产级自适应混合检索管线

```python
"""
生产级自适应混合检索管线
功能：query 分类 → 动态 alpha → 并行检索 → RRF 融合 → 可选重排序
"""
import asyncio
from typing import List, Dict, Optional, Callable
from dataclasses import dataclass
import time
import re

@dataclass
class RetrievalResult:
    doc_id: str
    text: str
    score: float
    source: str  # "sparse" / "dense" / "hybrid"

class AdaptiveHybridPipeline:
    """端到端自适应混合检索管线"""
    
    def __init__(
        self,
        sparse_retriever: Callable,
        dense_retriever: Callable,
        reranker: Optional[Callable] = None,
        default_alpha: float = 0.3,
        latency_sla_ms: int = 500,
    ):
        self.sparse = sparse_retriever
        self.dense = dense_retriever
        self.reranker = reranker
        self.default_alpha = default_alpha
        self.latency_sla = latency_sla_ms / 1000.0
    
    def classify_query(self, query: str) -> str:
        """query 分类: entity / keyword / semantic / general"""
        # 实体检测（产品 ID、编号等）
        if re.search(r'\b[A-Z]{2,}[-]?\d+\b', query):
            return "entity"
        # 短查询 ≤ 3 词
        if len(query.split()) <= 3:
            return "keyword"
        # 长查询 > 20 词
        if len(query.split()) > 20:
            return "semantic"
        return "general"
    
    def get_alpha(self, query: str) -> float:
        """根据 query 类型获取 alpha"""
        alpha_map = {
            "entity": 0.7, "keyword": 0.6,
            "semantic": 0.2, "general": 0.3,
        }
        return alpha_map.get(self.classify_query(query), self.default_alpha)
    
    async def retrieve(self, query: str) -> List[RetrievalResult]:
        start = time.time()
        alpha = self.get_alpha(query)
        top_k = 10
        
        # 并行执行两路检索
        sparse_task = asyncio.to_thread(self.sparse, query, top_k * 2)
        dense_task = asyncio.to_thread(self.dense, query, top_k * 2)
        
        sparse_out, dense_out = await asyncio.gather(sparse_task, dense_task)
        
        # RRF 融合
        k = 60
        rrf_scores: dict[str, float] = {}
        
        for rank, doc in enumerate(sparse_out, 1):
            rrf_scores[doc["id"]] = rrf_scores.get(doc["id"], 0) + (1 - alpha) / (k + rank)
        for rank, doc in enumerate(dense_out, 1):
            rrf_scores[doc["id"]] = rrf_scores.get(doc["id"], 0) + alpha / (k + rank)
        
        # 排序取 top_k
        ranked = sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)[:top_k]
        
        # 构建结果
        doc_map = {d["id"]: d for d in sparse_out + dense_out}
        results = []
        for doc_id, score in ranked:
            if doc_id in doc_map:
                doc = doc_map[doc_id]
                results.append(RetrievalResult(
                    doc_id=doc_id,
                    text=doc["text"],
                    score=score,
                    source="hybrid",
                ))
        
        # 可选：reranker 二次排序
        if self.reranker and results:
            results = await asyncio.to_thread(
                self.reranker, query, results
            )
        
        elapsed = time.time() - start
        print(f"Hybrid retrieval: {elapsed*1000:.1f}ms | alpha={alpha:.2f} | results={len(results)}")
        
        if elapsed > self.latency_sla:
            print(f"WARNING: Exceeded SLA ({elapsed*1000:.0f}ms > {self.latency_sla*1000:.0f}ms)")
        
        return results[:top_k]


# ========== 使用示例 ==========
async def main():
    # 假设已经构建好 sparse 和 dense retriever
    pipeline = AdaptiveHybridPipeline(
        sparse_retriever=lambda q, top_k: [
            {"id": f"s{i}", "text": f"sparse result {i} for {q}", "score": 10 - i}
            for i in range(top_k)
        ],
        dense_retriever=lambda q, top_k: [
            {"id": f"d{i}", "text": f"dense result {i} for {q}", "score": 0.95 - i*0.05}
            for i in range(top_k)
        ],
    )
    
    queries = [
        "如何提高睡眠质量",           # semantic
        "SKU-12345 的价格",           # entity
        "Python 异步编程",            # general
    ]
    
    for q in queries:
        results = await pipeline.retrieve(q)
        print(f"\nQuery: {q}")
        print(f"Type: {pipeline.classify_query(q)} | Alpha: {pipeline.get_alpha(q):.2f}")
        for r in results[:3]:
            print(f"  [{r.score:.4f}] {r.text}")
    
    asyncio.run(main())
```

Sources:
- [Hybrid Retrieval Fusion: RRF vs Weighted vs Learned](https://dev.to/gabrielanhaia/hybrid-retrieval-fusion-rrf-vs-weighted-vs-learned-when-each-wins-26i1)
- [What Is Hybrid Search? FutureAGI RAG Guide (2026)](https://futureagi.com/glossary/hybrid-search/)
- [Hybrid Search for RAG: BM25 + Vector Search Tutorial (2025)](https://app.ailog.fr/en/blog/guides/hybrid-search-rag)
- [RAG at Scale: How to Build Production AI Systems in 2026](https://redis.io/blog/rag-at-scale/)
- [Hybrid Search for RAG: Fix Retrieval Accuracy in AI](https://www.pingcap.com/blog/hybrid-search-rag-retrieval-accuracy/)
- [DAT: Dynamic Alpha Tuning for Hybrid Retrieval (arXiv 2503.23013)](https://ar5iv.labs.arxiv.org/html/2503.23013)
- [Query-Adaptive Hybrid Retrieval](https://github.com/EgwDean/Query-Adaptive-Hybrid-Retrieval)
- [Hybrid RAG: Dense and Sparse Retrieval for Better AI Answers](https://atlan.com/know/hybrid-rag/)
- [production-rag-systems-guide](https://github.com/girijesh-ai/ai-interview-codex/blob/main/gen-ai/production-rag-systems-guide.md)
