# 9.2.6 Retrieval Latency Optimization — 检索延迟优化：从毫秒到亚毫秒

## 简单介绍

检索延迟（Retrieval Latency）是生产级 RAG 系统最关键的指标之一。它指从用户发出查询到检索系统返回候选文档的总时间。在对话式 AI 中，检索延迟直接影响用户体验的流畅度——用户期望的是即时响应，而不是等待数秒的"正在搜索..."。

```
用户输入 Query
    │
    ├─ Embedding 生成 ───── embedding 模型推理延迟（5-50ms）
    │
    ├─ ANN 向量搜索 ─────── 索引查找延迟（1-100ms）
    │
    ├─ BM25 关键词检索 ──── 倒排索引查找（1-20ms）
    │
    ├─ 多路结果融合 ─────── RRF / 分数归一化（1-5ms）
    │
    └─ 重排 ────────────── 交叉编码器重排（50-500ms）
                                    ↓
                            总延迟 = 各个环节累加
```

目标：在**召回率不显著下降**的前提下，将 P99 检索延迟控制在 100ms 以内，甚至 10ms 以内。

## 基本原理

### 延迟来源分解

| 延迟来源 | 典型耗时 | 占比（典型管线） | 优化手段 |
|---------|---------|----------------|---------|
| Embedding 生成 | 5-50ms | 20-40% | 模型量化、GPU 推理、缓存 |
| ANN 向量搜索 | 1-100ms | 10-50% | 索引类型选择、参数调优 |
| BM25 检索 | 1-20ms | 5-15% | 倒排索引优化、Term 字典 |
| 网络传输 | 0.5-10ms | 2-10% | 连接池、就近部署 |
| 结果融合/排序 | 1-5ms | 2-5% | 向量化计算 |
| 交叉编码器重排 | 50-500ms | 20-60% | 级联重排、轻量模型 |
| 元数据过滤 | 0.5-5ms | 1-5% | 位图索引 |

### 延迟的 S 曲线特性

```
延迟 (ms)
  ↑
  │                  ╱
  │               ╱
  │            ╱      ← 非线性增长区（索引参数不当）
  │         ╱
  │      ╱← 线性增长区（正常扩展）
  │   ╱
  │╱
  └────────────────────→ 数据规模
```

向量检索的延迟随数据规模增长呈 S 曲线：小规模时延迟低且增长缓慢，到达某个临界点后延迟急剧上升，最终趋于平缓。

## 背景与演进

### 精确检索时代（2010s）

- **暴力搜索（Brute Force）**：计算查询与所有文档的相似度，O(n) 复杂度
- **100 万条数据延迟**：> 1s（单机 384 维向量）
- **只能用于小规模场景**

### ANN 索引时代（2018-2020）

FAISS（2018）和 ScaNN（2020）等 ANN 库的出现彻底改变了检索延迟格局：

```
延迟对比（100 万条，768 维）
Brute Force:  ~800ms
FAISS IVF:    ~20ms   (98% 召回率)
FAISS HNSW:   ~5ms    (99% 召回率)
FAISS PQ:     ~2ms    (95% 召回率)
```

### 量化时代（2021-2023）

- 产品量化（PQ）将向量压缩 4-16 倍
- 标量量化（SQ8）将 float32 压缩为 uint8
- 二进制量化（BQ）将向量压缩为二进制位

### 硬件加速时代（2023-2025）

- **GPU 加速**：cuVS / RAFT 库在 GPU 上实现 ANN，延迟降至亚毫秒级
- **FPGA 加速**：部分云厂商提供 FPGA 加速的向量检索
- **近似存储计算**：在存储层直接做向量过滤

### 当前趋势（2025-2026）

- 磁盘感知 ANN（DiskANN）：SSD 级索引，支持百亿级检索
- 流式索引：支持实时插入而不重建索引
- 学习型索引：ML 模型直接预测向量位置
- 存算分离架构：独立扩缩检索和存储

## 核心矛盾

### 延迟 vs 准确率

这是检索系统最根本的权衡。100% 召回率 = 暴力搜索 = 最慢。

```
召回率     实现方式                      P99 延迟（100万条）
100%       Brute Force（暴力搜索）        ~800ms
 99%       HNSW（ef_search=512）          ~20ms
 98%       HNSW（ef_search=128）          ~5ms
 95%       IVF-PQ（nprobe=32）            ~3ms
 90%       IVF-PQ（nprobe=8）             ~1ms
 85%       PCQ（Product+Scalar量化）       ~0.5ms
```

关键问题：**你的业务场景需要多少召回率？**

| 场景 | 可接受最低召回率 | 理由 |
|------|----------------|------|
| 事实性 QA | > 95% | 漏掉正确答案 = 错误输出 |
| 开放域闲聊 | > 85% | 即使不是最优结果也能回答 |
| 推荐系统 | > 90% | 需要保证多样性 + 相关性 |
| 代码搜索 | > 98% | 精确匹配至关重要 |

## 主流优化方向

### 1. HNSW 算法（Hierarchical Navigable Small World）

HNSW 是目前最流行的 ANN 算法之一，构建多层图结构实现对数级别的搜索复杂度。

```
HNSW 结构示意：
Layer 3  ○────○────○          ← 顶层（稀疏，长距离跳跃）
            /    │
Layer 2  ○─○───○─○──○        ← 中间层
         / │   │   │
Layer 1  ○─○─○─○─○─○─○      ← 底层（稠密，精细搜索）
```

| HNSW 参数 | 作用 | 推荐值 | 对延迟影响 |
|-----------|------|--------|-----------|
| M | 每节点最大连接数 | 16-64 | M 越大，图越稠密，搜索越快但建索引越慢 |
| ef_construction | 建索引时的搜索宽度 | 100-400 | 越大索引质量越高，建索引越慢 |
| ef_search | 搜索时的动态候选集大小 | 50-500 | 越大召回率越高，延迟越高 |

### 2. IVF 倒排索引（Inverted File Index）

将向量空间划分为 K 个 Voronoi 单元，搜索时只查询最近的几个单元。

```
IVF 结构示意：
               ┌───┐
   查询点 →    │ q │
               └───┘
       │
       ▼
┌──────┴──────┐
│  Voronoi 图  │      ← K 个聚类中心
│  ┌─┐ ┌─┐ ┌─┐│
│  │C1│ │C2│ │C3│
│  └─┘ └─┘ └─┘│
│   │     │     │
│  ┌─┐   ┌─┐   ┌─┐
│  │v1│   │v2│   │v3│   ← 每个单元内的向量
│  │v3│   │v4│   │v6│
│  └─┘   └─┘   └─┘
└────────────────┘
```

| IVF 参数 | 作用 | 推荐值 | 对延迟影响 |
|----------|------|--------|-----------|
| nlist | 聚类中心数量 | sqrt(N) ~ 4*sqrt(N) | 越大搜索精度越高，但建索引越慢 |
| nprobe | 搜索时访问的单元数 | 10-100 | 越大召回率越高，延迟线性增长 |

### 3. 量化技术

**产品量化（Product Quantization, PQ）**：
- 将向量切分为 M 个子空间
- 每个子空间用 K 个聚类中心编码（通常 K=256）
- 压缩比：4 字节 × M → 每条向量

**标量量化（Scalar Quantization, SQ）**：
- float32 → uint8：精度损失 < 1%，存储减少 75%
- 是生产环境最常用的量化手段

**二进制量化（Binary Quantization, BQ）**：
- float32 → 1 bit：存储减少 32 倍
- 召回率下降 5-10%，但速度提升 10 倍+

| 量化类型 | 压缩比 | 召回率损失 | 延迟提升 | 适用场景 |
|---------|--------|-----------|---------|---------|
| 无量化 (float32) | 1x | 0% | 1x | 小规模、高精度 |
| 标量量化 SQ8 | 4x | <1% | 2-3x | 通用场景 |
| 产品量化 PQ4x8 | 8x | 2-5% | 4-8x | 大规模向量 |
| 产品量化 PQ2x8 | 16x | 5-10% | 8-16x | 超大规模 |
| 二进制量化 | 32x | 8-15% | 10-30x | 粗筛 + 后续重排 |

### 4. GPU 加速

使用 GPU 进行批量向量搜索，延迟可降低 10-50 倍：

```python
# FAISS GPU 索引示例
import faiss

res = faiss.StandardGpuResources()  # 创建 GPU 资源
gpu_index = faiss.index_cpu_to_gpu(res, 0, cpu_index)  # CPU → GPU
```

GPU 特别适合需要同时处理大量查询的场景（batch search），但对单条查询的提升有限（因为 PCIe 传输延迟）。

## 能力边界与结果边界

### 召回率 vs QPS 权衡

```
QPS (Queries Per Second)
  ↑
  │              ● Brute Force (100% recall, 50 QPS)
  │        ● HNSW ef=512 (99% recall, 2K QPS)
  │    ● HNSW ef=128 (98% recall, 10K QPS)
  │  ● IVF-PQ nprobe=32 (95% recall, 20K QPS)
  │ ● IVF-PQ nprobe=8 (90% recall, 50K QPS)
  └────────────────────────────→ Recall
```

| 配置 | 100 万条 P99 延迟 | 1000 万条 P99 | QPS（单机） |
|------|------------------|--------------|------------|
| Brute Force (L2) | 800ms | 8s | 50 |
| FAISS Flat (CPU) | 30ms | 300ms | 500 |
| FAISS IVF (nprobe=32) | 8ms | 60ms | 2,000 |
| FAISS HNSW (ef=128) | 3ms | 15ms | 10,000 |
| FAISS IVF-PQ (nprobe=32) | 2ms | 10ms | 20,000 |
| FAISS HNSW+PQ | 1ms | 5ms | 50,000 |
| GPU-cuVS (HNSW) | 0.2ms | 1ms | 100,000+ |

### 量化精度损失的边界

当 PQ 码本大小 K < 256 或子空间数 M < 4 时，召回率断崖式下降。PQ 在 64 维以上的向量上效果较好，低维向量量化后区分度损失过大。

```
向量维度 vs PQ 有效压缩比：
128 维 → 最大压缩 8x 仍可保持 95% 召回率
64 维  → 最大压缩 4x 仍可保持 95% 召回率
32 维  → 不建议使用 PQ，推荐 SQ8
```

## 不同索引方法的区别

| 维度 | Brute Force (Flat) | IVF | HNSW | PQ | IVF-PQ |
|------|-------------------|-----|------|-----|--------|
| 搜索方式 | 精确全扫描 | 聚类 + 局部扫描 | 层级图遍历 | 压缩距离计算 | 聚类 + 压缩 |
| 建索引时间 | 0 | 中 | 慢 | 中 | 中 |
| 搜索速度（100万） | 慢 (800ms) | 快 (10ms) | 很快 (3ms) | 快 (5ms) | 最快 (2ms) |
| 召回率（@10） | 100% | 90-97% | 95-99%+ | 85-95% | 90-97% |
| 内存占用 | 高 (float32) | 中 | 高 (图结构) | 低 | 低 |
| 支持增量添加 | 是 | 否（需重建） | 是 | 否 | 否 |
| 参数调优难度 | 无 | 中 | 低-中 | 高 | 高 |
| 最优规模 | N < 10万 | 10万-1亿 | 100万-10亿 | 100万-1亿 | 1000万-10亿 |
| 实现复杂度 | 极简 | 简单 | 中等 | 复杂 | 中等 |

## 核心优势

1. **亚 10ms 百万级检索**：HNSW + SQ8 可在 10ms 内完成 100 万条向量的 99% 召回率搜索
2. **存储压缩 4-32 倍**：量化技术让向量索引从内存密集型变为内存友好型
3. **水平扩展**：分片 + 分布式检索支撑千亿级向量
4. **灵活权衡**：通过调整 ef_search / nprobe 等参数，在延迟和精度之间无极调节
5. **硬件无关**：同一套索引库（FAISS）可以在 CPU/GPU 上无缝切换

## 工程优化

### 1. HNSW 参数调优

```python
import faiss
import time
import numpy as np

def tune_hnsw_parameters(
    xb: np.ndarray,          # 数据集 (N, d)
    xq: np.ndarray,          # 查询集 (Q, d)
    gt: np.ndarray,          # 真实最近邻 (Q, K)
    dim: int = 768,
    m_values: list = None,
    ef_search_values: list = None,
):
    """HNSW 参数网格搜索"""
    if m_values is None:
        m_values = [16, 32, 48, 64]
    if ef_search_values is None:
        ef_search_values = [64, 128, 256, 512]
    
    results = []
    for m in m_values:
        for ef_s in ef_search_values:
            # 构建索引
            index = faiss.IndexHNSWFlat(dim, m)
            index.hnsw.efConstruction = 200  # 固定 efConstruction
            index.add(xb)
            
            # 设置搜索参数
            index.hnsw.efSearch = ef_s
            
            # 基准测试
            start = time.perf_counter()
            D, I = index.search(xq, 10)
            elapsed = (time.perf_counter() - start) * 1000
            
            # 计算召回率
            recall = np.mean([
                len(np.intersect1d(I[i], gt[i])) / min(10, len(gt[i]))
                for i in range(len(xq))
            ])
            
            results.append({
                "M": m,
                "efSearch": ef_s,
                "latency_ms": round(elapsed / len(xq), 2),
                "recall": round(recall, 4),
                "qps": round(len(xq) / (elapsed / 1000), 0),
            })
    
    return results

# 使用示例
# results = tune_hnsw_parameters(xb, xq, gt)
# 最优配置通常是：M=32, efSearch=128（延迟与召回率的最佳权衡）
```

### 2. 语义缓存（Semantic Cache）

缓存相同或相似查询的检索结果，避免重复执行昂贵的检索管线：

```python
import numpy as np
from collections import OrderedDict
from typing import List, Optional, Tuple


class SemanticCache:
    """基于语义相似度的检索结果缓存"""
    
    def __init__(
        self,
        embedding_model,
        max_size: int = 1000,
        similarity_threshold: float = 0.92,
        cache_ttl_seconds: int = 300,
    ):
        self.embedding_model = embedding_model
        self.max_size = max_size
        self.similarity_threshold = similarity_threshold
        self.cache_ttl = cache_ttl_seconds
        
        self.cache: OrderedDict = OrderedDict()  # LRU 缓存
        self.embedding_cache: dict = {}  # query → embedding
    
    def _get_embedding(self, query: str) -> np.ndarray:
        """获取 / 缓存 query 的 embedding"""
        if query not in self.embedding_cache:
            self.embedding_cache[query] = self.embedding_model.encode(query)
        return self.embedding_cache[query]
    
    def _cosine_similarity(self, a: np.ndarray, b: np.ndarray) -> float:
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
    
    def get(self, query: str) -> Optional[list]:
        """查找缓存：如果找到语义相似查询，返回缓存结果"""
        query_emb = self._get_embedding(query)
        
        for cached_query, (results, timestamp) in reversed(list(self.cache.items())):
            # TTL 检查
            if time.time() - timestamp > self.cache_ttl:
                continue
            
            cached_emb = self._get_embedding(cached_query)
            similarity = self._cosine_similarity(query_emb, cached_emb)
            
            if similarity >= self.similarity_threshold:
                # LRU: 移到末尾
                self.cache.move_to_end(cached_query)
                return results
        
        return None
    
    def set(self, query: str, results: list):
        """写入缓存"""
        # LRU 淘汰
        while len(self.cache) >= self.max_size:
            self.cache.popitem(last=False)
        
        self.cache[query] = (results, time.time())
    
    def get_cache_hit_rate(self) -> float:
        """计算缓存命中率"""
        # 需要在类外部通过装饰器或包装器统计
        pass
```

### 3. 结果缓存（Result Cache）

对确定性查询（完全相同）做精确匹配缓存，命中率不如语义缓存但零开销：

```python
from functools import lru_cache
import hashlib
import json

class ExactMatchCache:
    """精确匹配结果缓存"""
    
    def __init__(self, maxsize: int = 10000):
        self.cache = lru_cache(maxsize=maxsize)
    
    def _make_key(self, query: str, top_k: int, filters: dict = None) -> str:
        key_dict = {"q": query, "k": top_k, "f": filters or {}}
        return hashlib.md5(json.dumps(key_dict, sort_keys=True).encode()).hexdigest()
    
    def retrieve(self, query: str, top_k: int, retrieve_fn):
        """缓存装饰：存在缓存直接返回，否则执行 retrieve_fn"""
        @self.cache
        def cached_retrieve(cache_key: str):
            return retrieve_fn(query, top_k)
        
        key = self._make_key(query, top_k)
        return cached_retrieve(key)
```

### 4. 连接池（Connection Pooling）

减少与向量数据库之间的连接建立开销：

```python
from queue import Queue
import threading

class VectorDBConnectionPool:
    """向量数据库连接池"""
    
    def __init__(self, create_connection, max_size: int = 10):
        self._pool = Queue(maxsize=max_size)
        self._create = create_connection
        self._max_size = max_size
        self._size = 0
        self._lock = threading.Lock()
    
    def acquire(self, timeout: float = 1.0):
        """获取连接（如果池为空则创建新连接）"""
        try:
            return self._pool.get(block=True, timeout=timeout)
        except:
            with self._lock:
                if self._size < self._max_size:
                    conn = self._create()
                    self._size += 1
                    return conn
            raise TimeoutError("All connections are busy")
    
    def release(self, conn):
        """归还连接"""
        self._pool.put(conn)
    
    def __enter__(self):
        self._conn = self.acquire()
        return self._conn
    
    def __exit__(self, *args):
        self.release(self._conn)


# 使用示例
class RetrieverWithPool:
    def __init__(self, pool: VectorDBConnectionPool):
        self.pool = pool
    
    def search(self, query_vector, top_k: int):
        with self.pool as conn:
            return conn.search(query_vector, top_k)
```

### 5. Pre-fetching（预取）

利用用户思考或输入时间，预先生成 embedding 或预检索：

```python
class PrefetchRetriever:
    """预取检索器：在用户输入时提前检索"""
    
    def __init__(self, base_retriever, prefetch_strategy="prefix"):
        self.base = base_retriever
        self.prefetch_cache = {}
    
    def on_user_typing(self, prefix: str):
        """用户正在输入时预检索"""
        if len(prefix) >= 3:
            # 预检索前几个字符匹配的结果
            candidates = self.base.retrieve(prefix, top_k=20)
            self.prefetch_cache[prefix] = candidates
    
    def retrieve(self, query: str, top_k: int = 10):
        """优先使用预取结果"""
        # 找到最长匹配的预取前缀
        matched = ""
        for prefix in sorted(self.prefetch_cache.keys(), key=len, reverse=True):
            if query.startswith(prefix):
                matched = prefix
                break
        
        if matched and len(matched) >= len(query) * 0.5:
            # 在预取结果上做重排（比全库检索快得多）
            candidates = self.prefetch_cache[matched]
            return self._rerank(query, candidates)[:top_k]
        
        return self.base.retrieve(query, top_k)
```

### 6. 索引选择策略

不同规模和数据量下的推荐索引类型：

| 数据规模 | 推荐索引 | 参数建议 | 预期延迟 |
|---------|---------|---------|---------|
| < 1 万 | Flat (暴力) | 无参数 | < 1ms |
| 1 万-10 万 | HNSWFlat | M=16, efSearch=64 | 1-3ms |
| 10 万-100 万 | HNSWFlat / IVFFlat | M=32, efSearch=128 / nlist=4096, nprobe=32 | 3-10ms |
| 100 万-1000 万 | HNSWSQ8 / IVFPQ | M=32, efSearch=128 / nlist=16384, nprobe=64 | 5-20ms |
| 1000 万-1 亿 | IVFPQ / HNSWPQ | M=64, PQ M=8, efSearch=256 | 10-50ms |
| > 1 亿 | 分片 + IVFPQ | 多分片 + PQ | 20-200ms |

## 场景适配指南

| 场景 | 数据量 | 延迟要求 | 推荐索引 | 额外优化 |
|------|-------|---------|---------|---------|
| 实时语音助手 | 100 万 | < 20ms | HNSWSQ8 (M=16, ef=64) | GPU 推理 + 结果缓存 |
| 电商搜索 | 1000 万 | < 30ms | IVFPQ (nprobe=32) | 类目过滤预筛 + PQ 压缩 |
| 企业知识库 | 500 万 | < 50ms | HNSWFlat (M=32, ef=128) | 语义缓存 + 连接池 |
| 学术检索 | 1 亿 | < 100ms | IVFPQ + 分片 | 查询路由 + 批量检索 |
| 代码搜索 | 100 万 | < 20ms | HNSWFlat (M=16, ef=64) | 语言过滤预筛 |
| 代码补全 | 1000 万 | < 5ms | IVF-PQ (nprobe=8) | 预取 + 连接池 + GPU |
| 社交媒体推荐 | 10 亿 | < 50ms | 分片 IVFPQ | 分布式 + 二进制量化 |
| 深度研究助手 | 1000 万 | < 200ms | HNSWFlat (M=64, ef=256) | 全精度 + 交叉编码器 |

## 示例代码：生产级延迟优化工具链

```python
import time
import numpy as np
import faiss
from dataclasses import dataclass
from typing import List, Callable, Optional


@dataclass
class BenchmarkResult:
    index_type: str
    latency_ms: float
    recall: float
    qps: float
    memory_mb: float
    params: dict


class LatencyBenchmark:
    """检索延迟基准测试工具"""
    
    def __init__(self, dim: int = 768, metric: str = "L2"):
        self.dim = dim
        self.metric = faiss.METRIC_L2 if metric == "L2" else faiss.METRIC_INNER_PRODUCT
    
    def build_indexes(self, xb: np.ndarray) -> dict:
        """构建所有类型的索引用于对比"""
        n, d = xb.shape
        indexes = {}
        
        # 1. Flat（暴力搜索，基准线）
        index_flat = faiss.IndexFlatL2(d)
        index_flat.add(xb)
        indexes["Flat (Brute Force)"] = index_flat
        
        # 2. IVF
        nlist = int(4 * np.sqrt(n))
        quantizer = faiss.IndexFlatL2(d)
        index_ivf = faiss.IndexIVFFlat(quantizer, d, nlist)
        index_ivf.train(xb)
        index_ivf.add(xb)
        indexes["IVFFlat"] = index_ivf
        
        # 3. HNSW
        index_hnsw = faiss.IndexHNSWFlat(d, 32)  # M=32
        index_hnsw.hnsw.efConstruction = 200
        index_hnsw.add(xb)
        indexes["HNSWFlat"] = index_hnsw
        
        # 4. PQ
        pq_m = 8  # 子空间数
        index_pq = faiss.IndexPQ(d, pq_m, 8)  # 8 bits per sub-vector
        index_pq.train(xb)
        index_pq.add(xb)
        indexes["PQ"] = index_pq
        
        # 5. IVF-PQ
        index_ivfpq = faiss.IndexIVFPQ(quantizer, d, nlist, pq_m, 8)
        index_ivfpq.train(xb)
        index_ivfpq.add(xb)
        indexes["IVFPQ"] = index_ivfpq
        
        # 6. HNSW + SQ8
        index_hnsw_sq = faiss.IndexHNSWSQ(d, faiss.ScalarQuantizer.QT_8bit, 32)
        index_hnsw_sq.hnsw.efConstruction = 200
        index_hnsw_sq.train(xb)
        index_hnsw_sq.add(xb)
        indexes["HNSWSQ8"] = index_hnsw_sq
        
        return indexes
    
    def benchmark(
        self,
        indexes: dict,
        xq: np.ndarray,
        gt: np.ndarray,
        top_k: int = 10,
        warmup: int = 10,
        runs: int = 100,
        search_params: Optional[dict] = None,
    ) -> List[BenchmarkResult]:
        """基准测试所有索引"""
        
        if search_params is None:
            search_params = {
                "IVFFlat": {"nprobe": 32},
                "IVFPQ": {"nprobe": 32},
            }
        
        results = []
        
        for name, index in indexes.items():
            # 设置搜索参数
            if name in search_params:
                params = search_params[name]
                if hasattr(index, 'nprobe'):
                    index.nprobe = params["nprobe"]
            
            # Warmup
            for _ in range(warmup):
                index.search(xq[:1], top_k)
            
            # 正式测试
            latencies = []
            for i in range(runs):
                start = time.perf_counter()
                D, I = index.search(xq[i:i+1], top_k)
                latencies.append((time.perf_counter() - start) * 1000)
            
            # 计算召回率
            D, I = index.search(xq[:len(gt)], top_k)
            recalls = []
            for i in range(len(gt)):
                recall = len(np.intersect1d(I[i], gt[i])) / min(top_k, len(gt[i]))
                recalls.append(recall)
            
            # 内存占用估算
            memory = index.ntotal * self._estimate_bytes_per_vector(name)
            
            results.append(BenchmarkResult(
                index_type=name,
                latency_ms=round(float(np.median(latencies)), 2),
                recall=round(float(np.mean(recalls)), 4),
                qps=round(1000 / np.median(latencies), 0),
                memory_mb=round(memory / (1024 * 1024), 2),
                params=search_params.get(name, {}),
            ))
        
        return results
    
    def _estimate_bytes_per_vector(self, index_type: str) -> int:
        """估算每种索引的每向量占用字节数"""
        estimates = {
            "Flat (Brute Force)": self.dim * 4,           # float32
            "IVFFlat": self.dim * 4,                       # float32 + id mapping
            "HNSWFlat": self.dim * 4 + 32 * 4,             # float32 + graph edges
            "PQ": self.dim // 2,                            # 8 bits per sub-vector
            "IVFPQ": self.dim // 2 + 4,                     # PQ + id mapping
            "HNSWSQ8": self.dim * 1 + 32 * 4,              # uint8 + graph edges
        }
        return estimates.get(index_type, self.dim * 4)


# ============================================================
# 自适应索引选择器
# ============================================================
class AdaptiveIndexSelector:
    """根据数据规模和延迟要求自动选择索引类型和参数"""
    
    @staticmethod
    def recommend(n_vectors: int, dim: int, max_latency_ms: float, required_recall: float = 0.95) -> dict:
        """推荐索引类型和参数"""
        
        recommendations = []
        
        # 规则1：小规模直接 Flat
        if n_vectors < 10000:
            recommendations.append({
                "index": "Flat",
                "params": {},
                "expected_latency_ms": 0.5,
                "expected_recall": 1.0,
                "memory_mb": n_vectors * dim * 4 / (1024*1024),
            })
        
        # 规则2：中等规模用 HNSW
        if n_vectors < 1000000:
            recommendations.append({
                "index": "HNSWFlat",
                "params": {"M": 32, "efSearch": 128},
                "expected_latency_ms": 5,
                "expected_recall": 0.99,
                "memory_mb": n_vectors * (dim * 4 + 32 * 4) / (1024*1024),
            })
        
        # 规则3：大规模用 HNSW + SQ
        if n_vectors < 10000000:
            recommendations.append({
                "index": "HNSWSQ8",
                "params": {"M": 32, "efSearch": 128},
                "expected_latency_ms": 8,
                "expected_recall": 0.98,
                "memory_mb": n_vectors * (dim * 1 + 32 * 4) / (1024*1024),
            })
        
        # 规则4：超大规模用 IVFPQ
        recommendations.append({
            "index": "IVFPQ",
            "params": {
                "nlist": int(4 * np.sqrt(n_vectors)),
                "nprobe": 32,
                "pq_m": 8,
            },
            "expected_latency_ms": 10 + n_vectors / 500000,
            "expected_recall": 0.95,
            "memory_mb": n_vectors * (dim // 2 + 4) / (1024*1024),
        })
        
        # 按延迟过滤
        feasible = [r for r in recommendations 
                    if r["expected_latency_ms"] <= max_latency_ms 
                    and r["expected_recall"] >= required_recall]
        
        if not feasible:
            return recommendations[-1]  # 返回最高延迟的推荐
        
        # 选择延迟最低的可行方案
        return min(feasible, key=lambda r: r["expected_latency_ms"])


# ============================================================
# 延迟优化管线封装
# ============================================================
class OptimizedRetrievalPipeline:
    """整合了缓存 + 连接池 + 索引选择的优化检索管线"""
    
    def __init__(
        self,
        embedding_model,
        vector_index,
        semantic_cache: Optional[SemanticCache] = None,
        exact_cache: Optional[ExactMatchCache] = None,
        connection_pool: Optional[VectorDBConnectionPool] = None,
    ):
        self.embedder = embedding_model
        self.index = vector_index
        self.semantic_cache = semantic_cache
        self.exact_cache = exact_cache
        self.pool = connection_pool
    
    def retrieve(self, query: str, top_k: int = 10) -> list:
        timer = {}
        t0 = time.time()
        
        # 1. 精确缓存检查（零开销命中）
        if self.exact_cache:
            cached = self.exact_cache.get(query, top_k)
            if cached:
                return cached
        
        # 2. 语义缓存检查（毫秒级）
        if self.semantic_cache:
            t1 = time.time()
            cached = self.semantic_cache.get(query)
            timer["cache_lookup"] = (time.time() - t1) * 1000
            if cached:
                return cached
        
        # 3. 生成 embedding
        t1 = time.time()
        query_vec = self.embedder.encode(query)
        timer["embedding"] = (time.time() - t1) * 1000
        
        # 4. 向量检索
        t1 = time.time()
        if self.pool:
            with self.pool as conn:
                distances, indices = conn.search(query_vec.reshape(1, -1), top_k)
        else:
            distances, indices = self.index.search(query_vec.reshape(1, -1), top_k)
        timer["search"] = (time.time() - t1) * 1000
        
        # 5. 构建结果
        results = [{"id": int(idx), "score": float(1.0 - dist)}
                   for idx, dist in zip(indices[0], distances[0])]
        
        # 6. 写入缓存
        if self.semantic_cache:
            self.semantic_cache.set(query, results)
        
        timer["total"] = (time.time() - t0) * 1000
        print(f"Latency breakdown: {timer}")
        
        return results
```
