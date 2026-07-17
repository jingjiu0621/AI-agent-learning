# 6.2.1 向量数据库选型

## 简单介绍

向量数据库是专门为存储和检索高维向量数据而设计的数据库系统，是 AI Agent 长期记忆中处理非结构化信息（文本、图像、代码）的核心存储引擎。它将原始数据通过嵌入模型转换为向量表示，然后基于向量距离度量实现语义级别的相似搜索，使 Agent 能够"理解"信息的含义而非仅仅匹配关键词。

## 基本原理

向量数据库的核心运作流程分为三个阶段：

```
原始文本 → [Embedding Model] → 向量 [0.12, -0.34, 0.78, ...] → [索引构建] → 可搜索索引
用户查询 → [Embedding Model] → 查询向量 → [ANN 搜索] → Top-K 结果
```

### 近似最近邻搜索（ANN）

与精确的 KNN（K-Nearest Neighbors）不同，生产级向量数据库使用 ANN（Approximate Nearest Neighbor）算法，在精度略有牺牲的条件下将搜索速度提升数个数量级。主流 ANN 算法包括：

| 算法 | 核心思想 | 优势 | 劣势 | 代表数据库 |
|------|---------|------|------|-----------|
| HNSW (Hierarchical Navigable Small World) | 分层跳表 + 小世界图 | 高召回、低延迟 | 索引构建慢、内存占用高 | Qdrant, Weaviate |
| IVF (Inverted File Index) | K-Means 聚类 + 倒排 | 内存效率高、构建快 | 召回率略低 | Milvus |
| PQ (Product Quantization) | 向量压缩 + ADC 距离计算 | 极致压缩、低内存 | 精度损失较大 | Milvus, Faiss |
| DiskANN | 图结构 + SSD 存储 | 支持超大规模（十亿级） | 延迟受 I/O 影响 | 部分定制方案 |

## 背景与演进

向量数据库的发展经历了三个阶段：

**第一阶段：学术工具（2010s 前）** — 以 Faiss（Facebook）、ScANN（Google）为代表的算法库，提供高效的 ANN 算法实现，但缺少持久化、分布式、CRUD 等数据库特性，仅适合批处理场景。

**第二阶段：专用向量数据库（2019-2022）** — Pinecone（2019）、Weaviate（2019）、Qdrant（2020）、Milvus（2019）等原生向量数据库诞生，将 ANN 搜索与数据库管理功能整合，提供 RESTful API、数据持久化、水平扩展等企业级特性。

**第三阶段：传统数据库集成（2023-至今）** — PostgreSQL（pgvector）、Redis（RediSearch）、Elasticsearch 等传统数据存储开始内置向量支持，模糊了向量数据库与传统数据库的边界。同时，Agent 系统的爆发式增长推动了向量数据库在多模态搜索、记忆检索场景的广泛应用。

## 核心矛盾

### 1. 召回率 vs 搜索速度

- 高召回率要求更全面的图遍历或更少的量化损失，这会增加搜索延迟。
- 实际调优需要在 R@10（前十召回率）和 P99 延迟之间做权衡，通常目标设定为 R@10 > 95% 且 P99 < 50ms。

### 2. 索引构建成本 vs 数据新鲜度

- HNSW 等高性能索引的构建成本高昂，频繁写入会触发索引重建。
- 实时写入场景需要在"批量索引 + 定期重建"和"增量索引 + 性能降级"之间取舍。

### 3. 维度数 vs 存储效率

- 更高维度（1536d、3072d）提供更好的语义区分度，但显著增加存储成本和搜索延迟。
- 嵌入维度每增加一倍，搜索计算量增加约四倍（O(d * n) 复杂度）。

## 主流优化方向

1. **量化压缩**：通过标量量化（SQ）或乘积量化（PQ）将 32 位浮点向量压缩为 8 位甚至 1 位（二值化），存储压缩比可达 4x-32x。
2. **分层存储**：热数据在内存中使用 HNSW 索引，冷数据在 SSD 上使用 DiskANN，通过 LRU 策略自动管理存储层级。
3. **分片与分区**：按用户/租户维度进行分区（partitioning），减少每次搜索涉及的向量数量。
4. **预过滤与后过滤**：在 ANN 搜索前后附加标量过滤条件，减少候选集大小或过滤不相关结果。
5. **GPU 加速**：RAPIDS cuVS 等库利用 GPU 并行计算加速索引构建和搜索，吞吐可提升 10-50x。

## 产生的结果

- 搜索延迟从秒级降低到毫秒级（P50 < 10ms，P99 < 100ms）。
- 支持十亿级向量规模的存储和检索。
- 召回率在调优后可达到 95%-99%（与精确搜索对比）。
- 查询吞吐量在分布式部署下可达数万 QPS。

## 最大挑战

### 1. 数据新鲜度（Consistency）

写入后立即搜索可能无法检索到刚刚写入的数据（最终一致性）。在 Agent 场景中，刚存储的重要记忆无法立即读取可能导致推理错误。解决方案包括写入后强制刷新索引、使用读写分离架构、或采用短时间窗口的朴素搜索兜底。

### 2. 元数据过滤的性能衰减

当存储空间按租户/用户分区时，分区内数据量仍可能很大。更严重的是跨分区查询（Global Search）性能急剧下降。当元数据过滤条件高度选择性时（如过滤掉 99% 的数据），很多向量数据库无法将过滤下推到索引层，导致"先搜后滤"的低效模式。

### 3. 嵌入漂移（Embedding Drift）

当升级嵌入模型时，新旧向量位于不同语义空间，无法直接混合检索。需要实施双模型部署期、背景重新嵌入（backfill re-embedding）等迁移策略。

## 能力边界

- **不支持事务**：大多数向量数据库不提供 ACID 事务保证，不适合需要强一致性的金融级应用。
- **不支持复杂查询**：无法执行 JOIN、子查询、聚合等关系数据库的标准操作。
- **写入吞吐瓶颈**：单节点写入吞吐通常在几百到几千 QPS，远低于 KV 存储。
- **冷启动问题**：空数据库首次使用时，缺少足够数据支撑语义检索的有效性。

## 区别对比：主流向量数据库

### 功能对比矩阵

| 维度 | Chroma | Pinecone | Weaviate | Qdrant | Milvus |
|------|--------|----------|----------|--------|--------|
| **开源** | 是 (Apache 2.0) | 否 (闭源 SaaS) | 是 (BSD-3) | 是 (Apache 2.0) | 是 (Apache 2.0) |
| **部署方式** | 嵌入式/客户端 | 全托管云 | 自托管/云 | 自托管/云 | 自托管/云 |
| **索引算法** | HNSW | 专有优化 | HNSW | HNSW | IVF/HNSW/DiskANN |
| **标量过滤** | 有限 | 支持 | 支持 | 强支持 | 支持 |
| **多租户** | 不支持 | Pod 隔离 | Class 级别 | Collection 级别 | Collection 级别 |
| **分布式** | 不支持 | 原生支持 | Kubernetes | 原生支持 | 原生支持 |
| **持久化** | 文件/RAM | 自动管理 | 磁盘/WAL | 磁盘/WAL | 磁盘/WAL |
| **API** | Python SDK | REST/gRPC | GraphQL/REST | REST/gRPC | REST/gRPC/Python |

### 性能与成本对比

| 维度 | Chroma | Pinecone | Weaviate | Qdrant | Milvus |
|------|--------|----------|----------|--------|--------|
| 搜索速度 (P50) | 5-20ms | 5-15ms | 10-30ms | 3-10ms | 5-50ms |
| 写入吞吐 | < 100/s | 500-2000/s | 200-1000/s | 300-1500/s | 500-5000/s |
| 内存效率 | 低 (全载入) | 中 | 中 | 高 (内存映射) | 高 (分层) |
| 最大规模 | 百万级 | 十亿级 | 亿级 | 亿级 | 十亿级 |
| 托管成本 (1M 向量/月) | 免费 | ~$200-500 | ~$100-300 | ~$80-200 | ~$100-400 |

## 选型维度

### 1. 性能

- **搜索延迟敏感**：Qdrant（内存映射 + HNSW 优化出色，P50 最低）> Pinecone > Milvus
- **高吞吐写入**：Milvus（批量写入优化优秀）> Qdrant > Weaviate
- **大规模（亿级以上）**：Milvus（成熟的分片和索引策略）> Pinecone > Qdrant

### 2. 扩展性

- **垂直扩展（Scale Up）**：Chroma（单进程）< Weaviate < Qdrant < Milvus < Pinecone
- **水平扩展（Scale Out）**：Milvus（存算分离架构）和 Pinecone（全托管）最成熟
- **多区域部署**：Pinecone（原生多云）> Milvus（需自行运维）> Qdrant

### 3. 成本

- **无成本启动**：Chroma（本地运行，零成本，适合开发原型）
- **中型规模经济性**：Qdrant 自托管（内存映射技术降低硬件需求）> Weaviate
- **大规模最优**：Milvus 自托管（存算分离，计算和存储可独立扩缩）> Pinecone 托管
- **运维隐形成本**：自托管方案（Qdrant/Milvus）需要 DBA 人力，全托管（Pinecone）运维成本为零但单价更高

### 4. 托管 vs 自建决策矩阵

| 团队情况 | 推荐方案 | 理由 |
|---------|---------|------|
| 个人开发者/原型验证 | Chroma | 零部署，Python 包直接嵌入 |
| < 10 万向量，小团队 | Qdrant 自托管 | Docker 一键部署，运维极简 |
| 10 万 - 1000 万向量，无运维团队 | Pinecone | Serverless，零运维 |
| 1000 万以上，有 SRE 支持 | Milvus | 成熟分布式，成本可控 |
| 已有 PostgreSQL 基础设施 | pgvector | 无需引入新数据库，集成简单 |

### 5. 生态

- **Python 生态集成**：Chroma（LangChain、LlamaIndex 原生支持）> Milvus > Qdrant
- **多语言 SDK**：Milvus（Python/Java/Go/Node.js/Rust）> Qdrant > Weaviate
- **Kubernetes 原生**：Milvus（Operator 成熟度最高）> Weaviate > Qdrant
- **监控与可观测性**：Pinecone（内置监控面板）> Milvus（Prometheus + Grafana 集成）> 自建方案

## 核心优势

- **语义理解能力**：超越关键词匹配，捕捉同义词、上下文和隐含含义。
- **多模态扩展**：同一系统可同时存储文本、图像、代码的向量表示。
- **渐进式知识积累**：新知识写入后立即可检索，无需重新训练模型。
- **线性可扩展**：通过分片和分布式架构，理论无上限。

## 工程优化

### 索引参数调优

```python
# Qdrant HNSW 参数优化示例
{
    "hnsw": {
        "m": 16,            # 每层最大连接数 (8-64)，越大召回越高，内存越多
        "ef_construct": 200, # 构建时的动态列表大小 (100-500)，越大索引质量越高
        "ef_search": 256,   # 搜索时的动态列表大小 (64-512)，越大召回越高
        "full_scan_threshold": 10000  # 低于此大小的段使用全扫描
    }
}
```

### 高效写入策略

```python
# 批量写入 + 异步确认
async def batch_upsert(client, collection, vectors, batch_size=256):
    for i in range(0, len(vectors), batch_size):
        batch = vectors[i:i + batch_size]
        await client.upsert(
            collection_name=collection,
            points=batch,
            wait=False  # 异步写入，不等待索引刷新
        )
    # 最终强制刷新一次
    await client.flush(collection)
```

### 分层记忆缓存

对于 Agent 场景，引入本地缓存层命中高频访问的记忆：

```python
class CachedVectorMemory:
    def __init__(self, vector_db, local_cache_size=1000, ttl=300):
        self.db = vector_db
        self.cache = LRUCache(maxsize=local_cache_size, ttl=ttl)

    async def search(self, query_vector, top_k=10):
        cache_key = hash(query_vector.tobytes())
        if cache_key in self.cache:
            return self.cache[cache_key]

        results = await self.db.search(query_vector, top_k)
        self.cache[cache_key] = results
        return results
```

## 场景判断

### 适合使用向量数据库的场景

- **语义搜索**：需要理解查询意图而非精确匹配关键词（如"最近讨论过的技术方案"）
- **开放域记忆**：记忆内容格式不固定，无法预定义结构化 schema
- **多模态关联**：需要跨文本、图像、代码的统一检索接口
- **冷启动后可容忍不完全召回**：Agent 可以接受"漏掉了一些相关内容"

### 不适合使用向量数据库的场景

- **精确查找**：用户 ID 查找、配置获取 — 用 KV 或 SQL 更高效
- **高频更新**：每秒数千次的写入 — 结构化存储或消息队列更适合
- **强事务要求**：需要 ACID 保证的关键业务数据
- **结构化查询**：涉及多条件组合过滤、排序、聚合的分析场景
