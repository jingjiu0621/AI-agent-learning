# 6.2.5 hybrid-storage — 混合存储架构：向量 + 结构化 + 全文

## 简单介绍

混合存储（Hybrid Storage）是 Agent 长期记忆中最接近"生产就绪"的架构方案——它不依赖单一的存储引擎，而是将向量数据库、关系型数据库、全文检索引擎**组合使用**，发挥各自的优势来处理不同类型和不同维度的记忆查询。

单一存储方案总有力不能及的场景：向量存储擅长语义搜索但不擅长精确过滤，SQL 擅长结构化查询但不理解语义，全文搜索擅长关键词匹配但无法捕捉意图。混合存储通过**多路检索 + 结果融合**的方式，实现"既要...又要..."。

## 基本原理

### 核心思想

混合存储的核心假设是：**没有一种存储引擎能完美解决所有记忆检索需求**。因此，让多个引擎各司其职：

```
用户查询 "找出去年北京的气温数据，温度 > 30°C 的那些天"
         │
         ├─→ 向量存储：语义理解 "气温数据" "去年" "北京"
         │   └─→ 返回 Top-K 语义相关文档
         │
         ├─→ 结构化存储：精确过滤 "city=北京 AND temp>30 AND year=2024"
         │   └─→ 返回 SQL 查询结果集
         │
         └─→ 全文搜索：关键词匹配 "气温" "北京" "高温"
             └─→ 返回 BM25 评分结果
                      │
                  ┌───┴──────────┐
                  │  结果融合引擎  │
                  │  RRF / 加权  │
                  └───┬──────────┘
                      │
                  ┌───┴──────────┐
                  │  去重+重排序  │
                  └──────────────┘
                      │
                 最终推荐结果
```

### 三种存储的互补关系

| 维度 | 向量存储 | 结构化存储(SQL) | 全文搜索 |
|------|---------|---------------|---------|
| 查询方式 | 语义相似度 | 精确条件过滤 | 关键词匹配 |
| 理解层次 | 概念/意图 | 事实/关系 | 字面/模式 |
| 过滤能力 | 弱（后过滤） | 强（WHERE 子句） | 中（布尔查询） |
| 排序依据 | 向量距离 | 自定义排序 | BM25/TF-IDF 评分 |
| 适用数据 | 非结构化文本 | 结构化记录 | 长文本 |

## 混合检索模式

### 模式 1：级联检索（Cascaded Retrieval）

先窄后宽，先用最快/最精确的引擎过滤，再用其他引擎补充：

```python
def cascaded_retrieve(query: str, filters: dict):
    """级联检索：先精确过滤，再语义补充"""
    # Stage 1: 结构化精确过滤
    exact_results = sql_query(
        "SELECT * FROM memories WHERE user_id = %s AND created_at > %s",
        (filters["user_id"], filters["since"])
    )

    # Stage 2: 在精确结果中做语义重排序
    if exact_results:
        query_emb = embed(query)
        scored = [
            (mem, cosine_similarity(query_emb, mem["embedding"]))
            for mem in exact_results
        ]
        return sorted(scored, key=lambda x: -x[1])[:10]

    # Stage 3: 无精确匹配时回退到纯语义搜索
    return vector_search(query, top_k=10)
```

**适用场景**：先已知条件过滤，再语义排序。如"查找用户最近 7 天的记忆，并按与当前问题的相关度排序"。

### 模式 2：并行检索 + 融合（Parallel Retrieval + Fusion）

多个引擎同时检索，结果融合：

```python
def hybrid_retrieve(query: str, top_k: int = 10):
    """多路并行检索"""
    # 并行发起三个检索请求
    vector_results = vector_search(query, top_k * 2)
    sql_results = sql_search(query, top_k * 2)
    fts_results = fulltext_search(query, top_k * 2)

    # 结果融合（RRF 算法）
    fused = reciprocal_rank_fusion(
        [vector_results, sql_results, fts_results],
        weights=[0.5, 0.3, 0.2],  # 权重可调
    )

    return fused[:top_k]


def reciprocal_rank_fusion(
    result_lists: list[list[dict]],
    weights: list[float] | None = None,
    k: int = 60
) -> list[dict]:
    """RRF 融合：通过排名倒数加权聚合"""
    scores = {}
    for rank_list, weight in zip(result_lists, weights or [1.0] * len(result_lists)):
        for rank, item in enumerate(rank_list):
            doc_id = item["id"]
            scores[doc_id] = scores.get(doc_id, 0.0) + weight / (k + rank + 1)

    # 按融合得分排序
    ranked = sorted(scores.items(), key=lambda x: -x[1])

    # 映射回原始文档
    id_to_item = {item["id"]: item
                  for lst in result_lists for item in lst}
    return [id_to_item[doc_id] for doc_id, _ in ranked]
```

**适用场景**：需要同时从多个维度找信息的复杂查询。如"关于北京的天气和交通信息"——向量找语义相关，SQL 过滤具体城市，全文搜关键词。

### 模式 3：分层检索引擎（Layered Retrieval）

不同抽象级别的存储层，按需选择检索层：

```python
class LayeredMemoryStore:
    """分层记忆存储"""

    def __init__(self):
        self.layers = {
            "summary": SummaryStore(),      # 摘要层：高层概述
            "detail": VectorStore(),        # 细节层：具体内容
            "raw": FullTextStore(),         # 原始层：完整记录
            "metadata": SQLStore(),         # 元数据层：时间/类型/标签
        }

    def retrieve(self, query: str, depth: str = "auto"):
        """按深度检索"""
        if depth == "auto":
            # 先查摘要，根据摘要决定是否深入
            summaries = self.layers["summary"].search(query)
            if self._needs_detail(query, summaries):
                details = self.layers["detail"].search(query)
                return self._merge(summaries, details)
            return summaries
        else:
            return self.layers[depth].search(query)
```

## 数据同步策略

混合存储的核心工程挑战是**多存储引擎之间的数据一致性**。

### 策略 1：写入放大（Write-Amplified）

每次写入同时写入所有引擎：

```
┌──────────────┐
│  新记忆产生    │
└──────┬───────┘
       │
  ┌────┴────┐
  │ 写入向量  │  ← embedding + 存储
  ├─────────┤
  │ 写入 SQL  │  ← 结构化字段提取 + INSERT
  ├─────────┤
  │ 写入全文  │  ← 分词 + 倒排索引
  └─────────┘
```

**优点**：读取时数据一致，无需实时融合
**缺点**：写入延迟高（3x 写入），存储空间大（3x 冗余）

### 策略 2：主从同步（Primary-Replica）

一个主存储，其余通过 CDC（Change Data Capture）同步：

```python
class HybridStore:
    """主从同步混合存储"""

    def __init__(self):
        self.primary = SQLStore()      # SQL 为主存储
        self.vector_replica = VectorStore()
        self.fts_replica = FullTextStore()
        self.queue = AsyncQueue()      # 异步同步队列

    def write(self, memory: Memory):
        # 同步写入主存储
        mem_id = self.primary.insert(memory)

        # 异步同步到副本
        self.queue.enqueue({
            "type": "sync",
            "targets": ["vector", "fts"],
            "data": memory.to_dict(),
        })
        return mem_id

    def _sync_worker(self):
        """后台同步进程"""
        while task := self.queue.dequeue():
            if "vector" in task["targets"]:
                self.vector_replica.upsert(task["data"])
            if "fts" in task["targets"]:
                self.fts_replica.index(task["data"])
```

**优点**：主存储写入延迟不受影响，最终一致性在 Agent 场景通常可接受
**缺点**：短暂的不一致窗口（通常 <1s），读取副本可能拿到旧数据

## 背景

### 从单一存储到混合存储的演进

**第一代（2023）**：所有记忆存在一个向量数据库。问题：无法做精确过滤（"只查上周的"），元数据查询效率低。

**第二代（2024 初）**：向量数据库 + 元数据过滤（预过滤/后过滤）。问题：过滤能力受限于各向量数据库的实现（有的不支持复杂过滤），大结果集性能差。

**第三代（2024 中至今）**：混合架构——向量语义 + SQL 精确 + 全文关键词，多路并行检索后融合结果。

这一演进反映了 Agent 系统对记忆检索的要求在不断提高：从"找到相似的"到"找到相似且符合条件的"再到"找到最相关的，不管它是以什么维度被找到的"。

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **检索质量 vs 延迟** | 多路检索质量更高，但每增加一路就增加一次查询延迟 |
| **写入延迟 vs 一致性** | 写所有引擎保证一致性但写入慢；写主同步副本快但有不一致窗口 |
| **存储成本 vs 检索精度** | 冗余存储增加成本但提高检索覆盖率；减少冗余降低成本但可能漏掉关键结果 |
| **融合复杂性 vs 结果质量** | 复杂的融合算法（如学习排序）效果好但实现和维护成本高 |

## 当前主流优化方向

1. **动态路由检索**：根据查询类型自动选择检索引擎组合（事实查询→SQL，概念查询→向量，混合查询→全部）
2. **自适应权重**：根据历史点击数据自动调整各路检索结果的融合权重
3. **延迟优化**：向量检索降维加速（如从 1536d → 256d）、全文搜索使用倒排索引缓存
4. **冷热分层**：热数据全量索引（向量+SQL+全文），冷数据仅保留向量索引
5. **增量索引**：新记忆实时建索引，不需要全量重建

## 工程优化方向

1. **统一查询接口**：屏蔽底层多引擎的差异，对外提供统一的查询 API
   ```python
   class HybridMemory:
       def query(self, q: Query) -> list[Memory]:
           # 内部路由到合适的引擎组合
           pass
   ```

2. **降级策略**：某个引擎不可用时（如向量数据库宕机），自动降级到其他引擎，不中断服务

3. **结果多样性保证**：确保最终结果中不全是来自同一引擎或同一来源的"同质化"结果

4. **A/B 测试框架**：支持对比不同融合策略、不同权重配置在实际查询中的效果

5. **监控告警**：跟踪各引擎的延迟、召回率、融合后排序的 NDCG 指标

## 能力边界与结果边界

**能做的**：
- 在同一个查询中同时利用语义、精确条件和关键词匹配
- 通过多路检索 + 融合显著提升召回率（通常比单引擎高 20-40%）
- 支持优雅降级——单个引擎故障不会导致整个记忆系统不可用

**不能做的**：
- ❌ 消除数据不一致（最终一致性是现实的，强一致性成本太高）
- ❌ 解决底层引擎各自的固有问题（如向量存储的"语义漂移"、全文搜索的"同义词盲区"）
- ❌ 无限扩展——多路检索的资源消耗是倍数的

**结果边界**：
- 三路混合（向量+SQL+全文）通常是一个"甜点"——再增加引擎的边际收益递减
- 混合检索的延迟 ≈ 最慢的那一路 + 融合开销
- 混合存储的运维复杂度大约是单一存储的 2-3 倍

## 与其他方案的对比

| 对比 | 单一存储 | 混合存储 |
|------|---------|---------|
| 实现难度 | 低 | 高 |
| 检索召回率 | 中 | 高（+20-40%）|
| 写入延迟 | 低 | 中（同步）或低（异步）|
| 运维复杂度 | 低 | 中-高 |
| 灵活性 | 低 | 高 |
| 资源消耗 | 1x | 2-3x |

## 核心优势

1. **互补性**：每个引擎弥补其他引擎的盲区——语义搜索找不到精确匹配，SQL 补；SQL 无法语义理解，向量补
2. **鲁棒性**：单个引擎故障不导致系统不可用，自动降级到剩余引擎
3. **可扩展性**：可以按需添加新的检索维度（如添加图片相似度引擎）
4. **渐进式采用**：可以从一个引擎开始，逐步添加更多引擎

## 适合场景的判断标准

- **查询多样性**：查询类型单一 → 单一存储够用；查询类型多样（事实+语义+关键词）→ 需要混合
- **召回率要求**：可以忍受 60-70% 召回 → 单一存储；需要 90%+ 召回 → 混合存储
- **延迟要求**：极低延迟（<10ms）→ 避免混合（至少避免多路并行）；可接受 50-200ms → 适合混合
- **运维能力**：小团队/个人项目 → 优先按需单引擎；有运维支持 → 适合混合

**最适合**：生产级 RAG 系统、需要高精度检索的知识型 Agent、多维度记忆查询场景

**最不适合**：原型验证阶段、延迟敏感场景、简单检索需求（单引擎够用时不应为了技术而技术）
