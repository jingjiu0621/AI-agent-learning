# 6.6.6 记忆联邦（Memory Federation）

## 简单介绍

记忆联邦（Memory Federation）是一种**统一查询层**设计模式，它允许 AI Agent 通过单一接口同时查询多个异构记忆源（向量数据库、关系型数据库、知识图谱、文档存储、外部 API），并将来自不同源的结果融合为一致的答案返回给 Agent。

在现实的 Agent 架构中，记忆从来不是单一种类。一个生产级 Agent 可能同时拥有：

- **向量数据库**（如 Pinecone、Chroma）存储语义嵌入，用于相似度搜索
- **关系型数据库**（如 PostgreSQL）存储结构化业务数据
- **知识图谱**（如 Neo4j）存储实体关系网络
- **文档存储**（如 MongoDB）存储非结构化文档
- **外部 API**（如 CRM、Wiki）提供实时数据

如果没有联邦层，Agent 需要自己决定"查哪个源、怎么查、怎么合并结果"——这既增加了 Agent 推理的负担，也使得记忆访问逻辑分散在 Agent 的各个推理环节中。

```text
无联邦架构：
  Agent                              Agent 必须自己做所有决策
    │
    ├──→ "我该查向量库还是知识图谱？"
    ├──→ "先查向量库得到 5 个结果"
    ├──→ "再用这些结果去查知识图谱"
    ├──→ "两个源的结果怎么合并？"
    ├──→ "有一个源超时了，怎么办？"
    └──→ …… 认知负荷极大

有联邦架构：
  Agent
    │
    └──→ "帮我查一下 'Transformer 架构的演进'"
          │
          ▼
  ┌────────────────────────────────────────────┐
  │         记忆联邦层 (Memory Federation)       │
  │                                            │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │ 向量数据库 │  │ 知识图谱  │  │ 文档存储  │ │
  │  │(语义相似) │  │(实体关系) │  │(全文检索) │ │
  │  └──────────┘  └──────────┘  └──────────┘ │
  │                                            │
  │  并行查询 → 结果融合 → 去重 → 排序 → 返回   │
  └────────────────────────────────────────────┘
    │
    └──→ Agent 拿到统一结果
```

可以把记忆联邦理解为 Agent 的**"记忆负载均衡器"**——它屏蔽了底层的存储异构性，向上提供一个统一的 `query(query_text) → List[MemoryItem]` 接口，让 Agent 专注在"问什么"而不是"怎么查"。

### 核心定义

```text
记忆联邦 = 查询分解 + 并行执行 + 结果融合 + 排序去重
```

| 角色 | 类比 | 说明 |
|------|------|------|
| Agent | 用户 | 只需要提出查询需求，无需关心数据来源 |
| 联邦引擎 | 搜索引擎 | 分解查询、分发到各数据源、融合结果 |
| 各记忆源 | 垂直搜索引擎 | 各自在自己的数据范围内执行最优搜索 |
| 融合算法 | 排序算法 | 将多个排序结果合并为一个统一排序 |

## 基本原理

记忆联邦的核心流程可以分解为五个阶段：

```text
   Agent Query
       │
       ▼
┌──────────────────┐
│ 阶段 1: 查询分析  │  理解意图、提取关键词、识别查询类型
│ (Query Analysis) │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 阶段 2: 查询路由  │  根据分析结果决定查询哪些源
│ (Query Routing)  │  生成各源的查询计划
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 阶段 3: 并行执行  │  向各源并发发出查询请求
│ (Parallel Exec)  │  带超时控制和断路保护
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 阶段 4: 结果融合  │  收集各源结果，执行融合算法
│ (Result Fusion)  │  RRF / 加权投票 / 重排序
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 阶段 5: 去重排序  │  消除重复项，统一评分排序
│ (Dedup & Rank)   │  返回 Top-K 给 Agent
└────────┬─────────┘
         │
         ▼
   Unified Results
```

### 工作原理详述

**第一阶段——查询分析**：

联邦引擎接收 Agent 的查询请求后，首先对查询进行语义分析：

```text
查询: "2024 年 Q4 的销售数据中，哪些客户的复购率最高？"

分析结果:
  - 查询类型: 结构化分析查询 (结构化 + 语义)
  - 关键词: ["销售数据", "复购率", "2024年Q4", "客户"]
  - 意图: 分析型查询，需要聚合计算 + 语义检索
  - 涉及实体: 客户、销售订单、时间范围
  - 适合的源: [关系型数据库, 向量数据库]
```

**第二阶段——查询路由**：

基于分析结果，路由模块决定：

```text
路由决策:
  ┌─ 关系型数据库: "SELECT customer_id, COUNT(*) as repurchase_count
  │                  FROM orders WHERE order_date >= '2024-10-01'
  │                  GROUP BY customer_id HAVING repurchase_count > 1
  │                  ORDER BY repurchase_count DESC LIMIT 20"
  │
  └─ 向量数据库: "query_embedding('高复购率客户特征') limit 20"
```

**第三阶段——并行执行**：

联邦引擎并发地向各源发起查询。每个源被抽象为实现了统一接口的适配器。

```text
并行执行:
  时间线 ──────────────────────────────────────────────→
         │─────────── 向量库 50ms ───────→ 返回结果
         │───────── 关系型数据库 200ms ──────────────→ 返回结果
         │── 文档存储 30ms ──→ 返回结果
         │
         └── 总耗时 ≈ max(50, 200, 30) = 200ms
```

**第四阶段——结果融合**：

各源返回的结果格式各异，融合模块需要：

1. 统一标准化：将各源结果转为统一格式
2. 评分归一化：将各源的评分映射到统一尺度
3. 融合排序：使用 RRF 或加权融合算法生成新排序
4. 去重：消除跨源重复

**第五阶段——返回结果**：

返回 Top-K 个融合后的记忆项给 Agent：

```python
[
    MemoryItem(source="pg", content="客户 A: 复购率 85%", score=0.95),
    MemoryItem(source="vector", content="高复购客户特征: 月消费>3次", score=0.91),
    MemoryItem(source="pg", content="客户 B: 复购率 72%", score=0.87),
    # ...
]
```

### 适配器模式

联邦引擎通过适配器模式统一访问异构记忆源。每个记忆源只需要实现一个简单的接口：

```python
class MemorySource(ABC):
    """记忆源适配器接口。"""

    @abstractmethod
    async def query(self, query: Query) -> list[ResultItem]:
        """执行查询，返回结果列表。"""
        ...

    @property
    @abstractmethod
    def source_type(self) -> str:
        """返回源类型标识。"""
        ...

    @property
    @abstractmethod
    def health_status(self) -> bool:
        """健康检查。"""
        ...
```

这个抽象使得联邦引擎完全与底层存储技术解耦——可以随时添加新的记忆源而不影响现有逻辑。

## 背景与演进

### 第一阶段：单一存储（Single Store）

最早的 AI Agent 记忆系统采用"大一统"方案——所有记忆数据放在一个存储系统中。

```text
特性:
  - 单一数据库存所有（通常是一个向量库或关系型数据库）
  - 所有查询走同一个接口
  - 简单、运维成本低

问题:
  - 向量库不适合精确匹配和聚合查询
  - 关系型数据库不适合语义搜索
  - 一种存储无法最优处理所有类型的检索
  - 随着 Agent 能力扩展，单一存储成为瓶颈
```

典型案例：早期的对话 Agent 将所有对话历史存入单一的 Redis 或 MongoDB，只能做关键词匹配，无法理解语义。

### 第二阶段：多存储专业化（Multiple Specialized Stores）

Agent 能力扩展后，开发者开始为不同类型的记忆选择最合适的存储：

```text
演进方向:
  [Agent]
     │
     ├── 语义记忆 → 向量数据库 (Chroma/Pinecone)
     ├── 事实记忆 → 关系型数据库 (PostgreSQL)
     ├── 关系记忆 → 知识图谱 (Neo4j)
     ├── 文档记忆 → 文档存储 (MongoDB)
     └── 实时记忆 → 外部 API

优势:
  - 每种记忆类型获得最优的存储和检索性能
  - 不同类型的查询隔离，互不影响

问题:
  - Agent 代码中充斥对不同存储的直接调用
  - 查询逻辑分散：if-else 判断该查哪个源
  - 跨源查询时，Agent 需要自己实现结果融合
  - 新存储加入需要修改 Agent 核心代码
```

此阶段典型的 Agent 代码看起来像这样：

```python
# 反面示例：Agent 直接操作多个存储
class AgentWithoutFederation:
    async def query_memory(self, question: str):
        # Agent 必须自己判断从哪查
        if "语义相似" in question:
            return await self.vector_store.search(question)
        elif "具体数据" in question:
            return await self.sql_database.query(question)
        elif "实体关系" in question:
            return await self.knowledge_graph.traverse(question)
        else:
            # 只好全查一遍，自己合并
            vec_results = await self.vector_store.search(question)
            sql_results = await self.sql_database.query(question)
            kg_results = await self.knowledge_graph.traverse(question)
            # 做简单的列表拼接
            return vec_results + sql_results + kg_results  # 没有去重，没有排序
```

这种模式的主要问题：
- 查询路由逻辑与 Agent 推理逻辑耦合
- 跨源融合逻辑简陋（简单拼接而非智能融合）
- 缺乏统一的超时控制和错误处理
- 新加记忆源需要修改 Agent 代码

### 第三阶段：记忆联邦（Memory Federation）

借鉴数据库领域的 **联邦查询（Federated Query）**和搜索引擎领域的**分布式检索（Distributed Search）**理念，社区发展出了记忆联邦模式：

```text
[Agent] → 统一查询接口 → [联邦引擎] → [适配器1 → 向量库]
                                      → [适配器2 → 关系库]
                                      → [适配器3 → 知识图谱]
                                      → [适配器4 → 外部 API]
                                      → [适配器5 → 文档存储]

                        ← 融合结果返回
```

这一演进与数据库领域的"Polyglot Persistence"（多语言持久化）运动如出一辙。微服务架构中，不同服务使用不同数据库；在 Agent 架构中，不同记忆类型使用不同存储——两者最终都需要一个联邦层来提供统一访问。

### Polyglot Persistence 在 Agent 架构中的映射

```text
微服务 Polyglot Persistence:              Agent 记忆联邦:
┌────────────────────────┐               ┌────────────────────────┐
│ 服务A → MongoDB        │               │ 语义记忆 → 向量库      │
│ 服务B → PostgreSQL     │               │ 事实记忆 → SQL 数据库  │
│ 服务C → Redis          │               │ 关系记忆 → 知识图谱    │
│ 服务D → Elasticsearch  │               │ 文档记忆 → 文档存储    │
└────────┬───────────────┘               └────────┬───────────────┘
         │                                        │
         ▼                                        ▼
┌────────────────────────┐               ┌────────────────────────┐
│ API Gateway / BFF     │               │ 记忆联邦层              │
│ (统一服务入口)         │               │ (统一记忆接口)          │
└────────────────────────┘               └────────────────────────┘
```

## 核心矛盾

记忆联邦面临的根本矛盾是：**统一访问的便利性 vs 源特定优化的丧失**。

```text
               ┌─────────────────────────────┐
               │     记忆联邦的核心矛盾        │
               │                             │
               │  统一接口便利性              │
               │        ⇅                   │
               │  各源独有能力丧失            │
               └─────────────────────────────┘

具体表现:
  ┌────────────────────────────────────────────────────────┐
  │ 联邦层提供统一接口 = 取所有源的能力交集                │
  │                                                       │
  │  源 A (向量库) 能力: 语义搜索、聚类、降维              │
  │  源 B (关系库) 能力: 精确查询、聚合、事务              │
  │  源 C (图数据库) 能力: 图遍历、路径分析、社区发现      │
  │                                                       │
  │  联邦统一接口: query(text) → List[Item]               │
  │  → 图数据库的图遍历能力暴露不出来                      │
  │  → 关系数据库的聚合能力暴露不出来                      │
  │  → 向量库的聚类能力暴露不出来                          │
  └────────────────────────────────────────────────────────┘
```

### 矛盾一：通用接口 vs 源特异能力

联邦层提供的是一个通用的查询接口（通常是文本输入 → 排序结果列表输出）。这个接口是各源能力的"交集"——但交集中的能力往往是最基本的检索，丢失了大量各源的独特能力。

```text
源特异能力的丧失:

  向量数据库特有能力:
    - 向量聚类（k-means、DBSCAN）→ 联邦接口不暴露
    - 降维可视化（t-SNE、UMAP） → 联邦接口不暴露
    - 向量算术（king - man + woman） → 联邦接口不暴露

  知识图谱特有能力:
    - 多跳图遍历 → 联邦接口不暴露
    - 最短路径分析 → 联邦接口不暴露
    - 社区发现算法 → 联邦接口不暴露

  关系型数据库特有能力:
    - 复杂聚合（GROUP BY, HAVING, window functions） → 联邦接口不暴露
    - 多表 JOIN → 联邦接口不暴露
    - 事务一致性 → 联邦接口不暴露
```

**缓解方案**：联邦层可以支持"透传"模式——当 Agent 明确指定某个源并传递源特定查询时，联邦层作为透明代理直接转发：

```python
# 通用查询（联邦处理）
results = await federation.query("Transformer 的发展历史")

# 源特定查询（透传）
vector_results = await federation.query_raw(
    source="vector_db",
    raw_params={"vector": embedding, "metric": "cosine", "k": 100}
)
```

### 矛盾二：查询延迟 vs 结果完整性

联邦查询的总延迟 = max(各源延迟)，这意味着最慢的源拖累整个查询。

```text
        快源 (10ms)    ────────────────────── 等待慢源
        快源 (20ms)    ──────────────────── 等待慢源
        快源 (50ms)    ───────────────── 等待慢源
        慢源 (500ms)   ────────────────────────────────── → 500ms

        Agent 等待 500ms 才能拿到结果

        但其中 50ms 时就已有了 3/4 的数据
        → 是否可以渐进返回？先给部分结果，再补齐？
```

```text
完整性-延迟权衡:
                            
  100% ┤                   ╔══════════════╗
       │                   ║ 等待全部结果 ║
       │                   ║ 最慢源决定   ║
  75%  ┤       ╔══════════╗║ 延迟         ║
       │       ║          ║║              ║
       │       ║ 3/4 结果 ║║              ║
  50%  ┤  ═════║──────────║║──────────────║──→
       │       ║ 可用     ║║ 完整但慢     ║
  25%  ┤       ║          ║║              ║
       │       ╚══════════╝║              ║
  0%   ┤                   ╚══════════════╝
       └──────────────────────────────────────→
              低延迟              高延迟
```

### 矛盾三：结果质量 vs 融合复杂度

融合算法越复杂，结果质量可能越高，但计算开销和延迟也越大。

```text
融合算法复杂度谱系:

  低复杂度           →           高复杂度
  ─────────────────────────────────────────────────
  简单拼接           RRF        加权融合    学习排序
  无去重             有去重      动态权重    (LTR)
  快                较快         中等        慢
  质量低            中等         好          最好

  权衡:
    - RRF: O(N log N) 时间，质量可以接受
    - 加权融合: 需要维护权重，有冷启动问题
    - LTR: 需要标注数据训练模型，成本高
```

## 主流优化方向

### 1. Query Router（智能查询路由）

不是所有查询都需要查所有源。智能路由可以根据查询类型、源的健康状况和成本预算，选择最合适的源子集。

**基于意图分类的路由**：

```text
查询: "上海的天气怎么样？"
  └─ 意图分类: 实时查询
     └─ 路由: [外部天气 API] （不需要查向量库和知识图谱）

查询: "Transformer 和 BERT 的关系"
  └─ 意图分类: 知识查询
     └─ 路由: [知识图谱, 文档存储] （查实体关系和文档）

查询: "用户对产品的整体评价如何？"
  └─ 意图分类: 语义搜索 + 聚合
     └─ 路由: [向量数据库, 关系型数据库]
```

**最佳源选择算法**：

```python
class QueryRouter:
    """
    基于多维度评估的智能查询路由。
    """

    def __init__(self):
        # 每个源的评估维度：[相关性, 延迟, 成本, 健康状况]
        self.source_profiles = {
            "vector_db": {"capabilities": ["semantic", "similarity"],
                          "cost_per_query": 0.001, "avg_latency_ms": 50},
            "sql_db": {"capabilities": ["exact", "aggregation", "structured"],
                       "cost_per_query": 0.0001, "avg_latency_ms": 200},
            "knowledge_graph": {"capabilities": ["relation", "graph_traversal"],
                                "cost_per_query": 0.005, "avg_latency_ms": 500},
        }

    async def route(self, query_analysis: QueryAnalysis,
                    budget_ms: float = 500.0) -> list[str]:
        """
        根据查询分析和延迟预算选择最佳源集合。
        """
        candidates = []
        for name, profile in self.source_profiles.items():
            relevance = self._assess_relevance(
                query_analysis, profile["capabilities"]
            )
            if relevance > 0.3 and profile["avg_latency_ms"] <= budget_ms:
                candidates.append((name, relevance))

        # 按相关性排序，选择 Top-K
        candidates.sort(key=lambda x: x[1], reverse=True)
        return [name for name, score in candidates[:3]]

    def _assess_relevance(self, analysis: QueryAnalysis,
                          capabilities: list[str]) -> float:
        """评估查询与源能力的匹配度。"""
        # 查询需要的特性
        needed = set(analysis.characteristics)
        available = set(capabilities)
        if not needed:
            return 0.0
        return len(needed & available) / len(needed)
```

### 2. Result Fusion（结果融合）

结果融合是联邦层的核心环节。来自不同源的结果集各有不同的评分体系、覆盖范围和排序逻辑，融合算法的目标是将这些异构结果整合为一个统一、一致的排序。

**融合策略分类**：

```text
策略                    方法                  适用场景
────────────────────────────────────────────────────────────
RRF                    rank-based fusion     各源评分不可比
加权融合                score * weight        有历史准确率数据
交叉重排序              cross-encoder         需要高精度，可接受延迟
学习排序 (LTR)          ML model             有标注数据，离线训练
贪婪融合                top-from-each         简单场景，低延迟
```

### 3. Parallel Query Execution（并行查询执行）

联邦引擎需要高效地并发执行多个源查询，同时处理超时、错误和部分故障。

**执行模型**：

```text
顺序执行（低效）:
  源A (50ms) → 源B (200ms) → 源C (30ms) = 总计 280ms

并行执行（高效）:
  源A ──── 50ms ────┐
  源B ──── 200ms ───┤  并发执行
  源C ──── 30ms ────┘  = 总计 200ms
```

**带超时的并行执行**：

```python
import asyncio


async def parallel_query(sources: list[MemorySource],
                         query: Query,
                         timeout: float = 1.0) -> dict[str, list[ResultItem]]:
    """
    并行执行查询，带独立超时控制。
    超时的源返回空列表，不影响其他源。
    """
    async def query_with_timeout(source: MemorySource) -> tuple[str, list[ResultItem]]:
        try:
            results = await asyncio.wait_for(
                source.query(query),
                timeout=timeout
            )
            return source.name, results
        except asyncio.TimeoutError:
            logger.warning(f"源 {source.name} 查询超时 ({timeout}s)")
            return source.name, []
        except Exception as e:
            logger.error(f"源 {source.name} 查询失败: {e}")
            return source.name, []

    tasks = [query_with_timeout(src) for src in sources]
    results = await asyncio.gather(*tasks)

    return {name: items for name, items in results}
```

### 4. Source Health Monitoring（源健康监控）

联邦引擎需要实时监控每个记忆源的健康状态，以便在故障时做出智能决策。

**监控指标**：

```text
指标                采集方式                  作用
────────────────────────────────────────────────────────
查询延迟 (p50/p99)   每次请求记录             识别性能退化
错误率               错误计数 / 总请求         触发熔断
可用性               心跳检测                 识别宕机
数据新鲜度           上次更新标记             判断缓存是否可用
负载                 排队长度                 动态调整并发
```

**熔断器模式**：

```text
每个记忆源都有一个独立的熔断器：

正常状态 (CLOSED)
  │  错误率 > 阈值
  ▼
熔断状态 (OPEN)
  │  冷却时间到
  ▼
半开状态 (HALF_OPEN)
  │  测试请求成功 → 恢复正常
  │  测试请求失败 → 继续熔断
  ▼
正常状态 (CLOSED) 或 继续熔断 (OPEN)
```

### 5. Query Decomposition（查询分解）

复杂查询可能涉及多个维度的信息需求，需要分解为针对不同源的子查询。

**分解策略**：

```text
原始查询: "2024 年销量最高的产品有什么用户反馈？"

分解:
  ┌── 子查询 A (SQL 数据库): "SELECT product_id FROM sales
  │                             WHERE year = 2024
  │                             ORDER BY volume DESC LIMIT 10"
  │   → 结果: [product_1, product_3, product_7, ...]
  │
  └── 子查询 B (向量数据库): "search('用户对高销量产品的反馈')"
      → 结合子查询 A 的结果做 filtered search
      → 结果: ["产品很实用但包装需要改进", ...]

依赖关系: 子查询 B 依赖子查询 A 的结果 → 不能完全并行
```

**分解原则**：

```text
查询分解决策树:

  查询是否涉及多个维度？
    ├── 否 → 单源查询，不需要分解
    └── 是 → 子查询之间有无依赖关系？
              ├── 无依赖 → 完全并行执行
              └── 有依赖 → 构建 DAG 执行计划
                             ├── 先执行无依赖节点
                             ├── 中间结果传递
                             └── 最后执行依赖节点
```

### 6. Caching Federation Results（联邦结果缓存）

对高频查询或耗时查询的结果进行缓存，避免每次都需要跨源 fan-out。

**缓存策略**：

```text
缓存层级:
  ┌─────────────────────────────────────────────┐
  │  L1: 进程内缓存 (内存)                       │
  │  TTL: 几秒 ~ 几分钟                          │
  │  速度: < 1μs                                 │
  ├─────────────────────────────────────────────┤
  │  L2: 分布式缓存 (Redis / Memcached)          │
  │  TTL: 几分钟 ~ 几小时                        │
  │  速度: ~ 1ms                                 │
  ├─────────────────────────────────────────────┤
  │  L3: 联邦结果持久化 (数据库)                  │
  │  TTL: 几小时 ~ 几天                          │
  │  速度: ~ 10ms                                │
  └─────────────────────────────────────────────┘

缓存失效策略:
  - TTL 过期（最常见）
  - 源数据更新时主动失效（需要源端通知）
  - 查询结果与缓存结果差异度过大时自动过期
  - 显式清除（Agent 确认数据已过时）
```

**缓存键设计**：

```python
def make_cache_key(query_text: str, sources: list[str],
                   params: dict | None = None) -> str:
    """
    生成联邦查询的缓存键。
    包含查询内容、查询源和参数，确保唯一性。
    """
    import hashlib
    import json

    key_parts = [
        query_text.strip().lower(),
        "_".join(sorted(sources)),
        json.dumps(params, sort_keys=True) if params else "",
    ]
    raw_key = "|".join(key_parts)
    return hashlib.sha256(raw_key.encode()).hexdigest()
```

## Fusion Algorithms Deep Dive

融合算法是记忆联邦中最核心的技术组件。下面详细讨论六种融合算法。

### 1. RRF（Reciprocal Rank Fusion）

RRF 是联邦查询中最常用的融合算法。它的核心思想是：**不依赖各源的绝对评分，而是基于排名位置进行融合**。这使得它特别适合各源评分尺度不同的场景。

**算法公式**：

```text
对于每个文档 d，它的 RRF 分数为：

  RRF_score(d) = Σ 1 / (k + rank_i(d))

其中:
  - rank_i(d) 是文档 d 在源 i 中的排名（从 1 开始）
  - k 是平滑常数（通常 k=60 效果较好）
  - 求和符号覆盖所有查询的源

含义:
  - 排名越靠前，贡献越大（1/1, 1/2, 1/3, ...）
  - k 值越大，排名差异的影响越小
  - 如果在源 i 中没有出现，则不贡献分数
```

**RRF 的特性**：

```text
k=60 时的分数示例:

           Rank 1    Rank 10    Rank 100
  贡献     1/61≈0.016 1/70≈0.014 1/160≈0.006

  含义: 前 10 名的分数差距不大
        百名之后的分数急剧下降
        多个源的共同排名项会获得更高的总分

k 值的影响:
  k 较小 (如 5):      排名差异显著，强调顶部结果
  k 适中 (如 60):     推荐值，平衡好各源的贡献
  k 较大 (如 200):    排名差异变小，结果更多样
```

**Python 实现**：

```python
from collections import defaultdict
from typing import NamedTuple


class RankedItem(NamedTuple):
    """来自某个源的有序结果项。"""
    item_id: str
    rank: int  # 从 1 开始
    source: str
    score: float | None = None  # 原始得分（可选）


def reciprocal_rank_fusion(
    source_results: dict[str, list[RankedItem]],
    k: int = 60,
) -> list[tuple[str, float]]:
    """
    对多个源的排序结果执行 RRF 融合。

    Args:
        source_results: {源名称 -> [有序结果列表]}
        k: 平滑常数，默认 60

    Returns:
        [(item_id, rrf_score)] 按 RRF 分数降序排列
    """
    # 累积每个 item 的 RRF 分数
    rrf_scores: dict[str, float] = defaultdict(float)

    for source_name, items in source_results.items():
        for item in items:
            # RRF 公式: 1 / (k + rank)
            rrf_scores[item.item_id] += 1.0 / (k + item.rank)

    # 按分数降序排列
    sorted_items = sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)
    return sorted_items


# 使用示例
def demo_rrf():
    # 假设有三个源返回的结果
    vector_results = [
        RankedItem("doc_a", 1, "vector_db", 0.95),
        RankedItem("doc_b", 2, "vector_db", 0.90),
        RankedItem("doc_c", 3, "vector_db", 0.85),
    ]
    sql_results = [
        RankedItem("doc_b", 1, "sql_db", 100),
        RankedItem("doc_d", 2, "sql_db", 85),
        RankedItem("doc_e", 3, "sql_db", 70),
    ]
    kg_results = [
        RankedItem("doc_a", 1, "knowledge_graph", 0.8),
        RankedItem("doc_f", 2, "knowledge_graph", 0.7),
    ]

    fused = reciprocal_rank_fusion({
        "vector_db": vector_results,
        "sql_db": sql_results,
        "knowledge_graph": kg_results,
    })

    print("RRF 融合结果:")
    for item_id, score in fused:
        print(f"  {item_id}: {score:.4f}")

    # 输出:
    #   doc_a: 0.0328  (出现在 2 个源的最佳位置)
    #   doc_b: 0.0323  (出现在 2 个源)
    #   doc_c: 0.0159  (只出现在向量库第 3)
    #   doc_d: 0.0156  (只出现在 SQL 第 2)
    #   doc_e: 0.0154  (只出现在 SQL 第 3)
    #   doc_f: 0.0154  (只出现在知识图谱第 2)
```

### 2. 加权融合（Weighted Fusion）

当不同源的历史表现或可信度不同时，可以为每个源分配不同的权重。

**公式**：

```text
Weighted_score(d) = Σ w_i × normalized_score_i(d)

其中:
  - w_i 是源 i 的权重（Σ w_i = 1.0）
  - normalized_score_i(d) 是文档 d 在源 i 中的归一化评分 [0, 1]
```

**权重调整策略**：

```text
静态权重:
  - 由领域专家设定
  - 例如: 向量库 0.4, SQL 0.3, 知识图谱 0.3
  - 长期稳定，不随场景变化

动态权重:
  - 基于历史准确率自动调整
  - 例如: 某源历史准确率 90%，权重 = 0.9
  - 自适应环境变化

基于反馈的权重:
  - 根据 Agent 对结果的满意度调整
  - Agent 使用 fusion 结果后给出反馈
  - 反馈信号增强或衰减各源权重
```

**Python 实现**：

```python
import numpy as np
from collections import defaultdict


def weighted_fusion(
    source_results: dict[str, list[RankedItem]],
    source_weights: dict[str, float],
    k: int = 60,
) -> list[tuple[str, float]]:
    """
    加权融合 —— 结合 RRF 和源权重。

    Args:
        source_results: {源名称 -> [有序结果列表]}
        source_weights: {源名称 -> 权重} (自动归一化)
        k: RRF 平滑常数

    Returns:
        [(item_id, weighted_score)] 按分数降序排列
    """
    # 归一化权重
    total = sum(source_weights.values())
    norm_weights = {name: w / total for name, w in source_weights.items()}

    scores: dict[str, float] = defaultdict(float)

    for source_name, items in source_results.items():
        weight = norm_weights.get(source_name, 1.0 / len(source_results))
        for item in items:
            rrf_contrib = 1.0 / (k + item.rank)
            scores[item.item_id] += weight * rrf_contrib

    return sorted(scores.items(), key=lambda x: x[1], reverse=True)


class AdaptiveWeightManager:
    """
    自适应权重管理器 —— 根据源的历史表现动态调整权重。
    """

    def __init__(self, source_names: list[str], learning_rate: float = 0.1):
        self.weights = {name: 1.0 / len(source_names)
                        for name in source_names}
        self.accuracy_history: dict[str, list[float]] = {
            name: [] for name in source_names
        }
        self.lr = learning_rate

    def record_feedback(self, source_name: str, was_useful: bool):
        """
        记录 Agent 对某源结果的反馈。

        Args:
            source_name: 源名称
            was_useful: Agent 认为该源的结果是否有用
        """
        self.accuracy_history[source_name].append(1.0 if was_useful else 0.0)

        # 每 10 次反馈更新一次权重
        if len(self.accuracy_history[source_name]) >= 10:
            self._update_weights()

    def _update_weights(self):
        """基于最近准确率更新权重。"""
        recent_accuracy = {}
        for name, history in self.accuracy_history.items():
            recent = history[-10:]  # 最近 10 次
            recent_accuracy[name] = np.mean(recent)

        # 分数转权重
        total = sum(recent_accuracy.values())
        if total > 0:
            for name in self.weights:
                old_weight = self.weights[name]
                target = recent_accuracy[name] / total
                self.weights[name] = (1 - self.lr) * old_weight + self.lr * target

    def get_weights(self) -> dict[str, float]:
        return dict(self.weights)
```

### 3. Late Fusion vs Early Fusion

**Early Fusion（早期融合）**：

在检索之前或检索过程中就进行融合。查询被分解后，各源几乎同时执行，结果在检索完成后立即融合。

```text
查询
  │
  ├──→ [源 A 检索] ──→ 结果 A ──┐
  ├──→ [源 B 检索] ──→ 结果 B ──┤──→ 融合 → 最终排序
  ├──→ [源 C 检索] ──→ 结果 C ──┘
  └──→ [源 D 检索] ──→ 结果 D ──┘

特点:
  - 并行执行，延迟 = max(源延迟)
  - 融合层看不到各源的中间状态
  - 适合各源结果独立且可合并的场景
```

**Late Fusion（晚期融合）**：

延迟融合决策，先收集各源的候选集和元信息，再进行更精细的融合。

```text
查询
  │
  ├──→ [源 A 检索] ──→ 候选 A + 元信息 ──┐
  ├──→ [源 B 检索] ──→ 候选 B + 元信息 ──┤
  ├──→ [源 C 检索] ──→ 候选 C + 元信息 ──┤──→ [交叉重排序] → 最终结果
  └──→ [源 D 检索] ──→ 候选 D + 元信息 ──┘
                                          │
                                    可以结合各源的
                                    元信息做更精准的判断

特点:
  - 可以引入交叉编码器做二次排序
  - 延迟更高（需要额外排序步骤）
  - 精度通常优于 Early Fusion
```

**对比**：

```text
维度        Early Fusion        Late Fusion
───────────────────────────────────────────────
延迟        低 (max 源延迟)     高 (额外排序时间)
精度        中等                 高
实现复杂度   低                   高
资源消耗    低                   高（需要加载排序模型）
适用场景    实时性要求高         精度要求高
扩展性      好（加源简单）       中（排序模型需更新）
```

### 4. 交叉源去重（Cross-Source Deduplication）

不同源可能返回相同或高度相似的内容。联邦融合前必须去重，否则同一内容会占据多个排名位置。

**去重策略**：

```text
策略一: 基于 ID 去重（精确匹配）
  - 文档有全局唯一 ID
  - 简单快速
  - 局限性: 不同源的 ID 系统可能不同

策略二: 基于内容哈希去重（近似匹配）
  - 对内容取 SimHash 或 MinHash
  - 可以检测相似但不完全相同的内容
  - 适合: 不同源存储了同一信息的不同表述

策略三: 基于语义相似度去重
  - 使用 embedding 计算余弦相似度
  - 相似度 > 阈值时视为重复
  - 最准确但计算开销大

策略四: 混合策略
  - 先用 ID 去重（快）
  - 再用 SimHash 去重（中）
  - 最后用语义去重（慢但精确）
```

**Python 实现**：

```python
import hashlib
from datasketch import MinHash, MinHashLSH


class Deduplicator:
    """
    多策略去重器 —— 组合多种去重策略。
    """

    def __init__(self, threshold: float = 0.85):
        self.threshold = threshold

    def dedup_by_id(self, items: list[dict]) -> list[dict]:
        """基于 ID 去重。"""
        seen: set[str] = set()
        deduped = []
        for item in items:
            item_id = item.get("id") or hashlib.md5(
                item.get("content", "").encode()
            ).hexdigest()
            if item_id not in seen:
                seen.add(item_id)
                deduped.append(item)
        return deduped

    def dedup_by_similarity(self, items: list[dict]) -> list[dict]:
        """基于 MinHash 相似度去重。"""
        # 构建 MinHash LSH 索引
        lsh = MinHashLSH(threshold=self.threshold, num_perm=128)
        result = []

        for i, item in enumerate(items):
            content = item.get("content", "")
            if not content:
                result.append(item)
                continue

            m = MinHash(num_perm=128)
            for word in content.split():
                m.update(word.encode())

            # 检查是否有相似项
            similar = lsh.query(m)
            if not similar:
                lsh.insert(f"item_{i}", m)
                result.append(item)

        return result

    def dedup_hybrid(self, items: list[dict]) -> list[dict]:
        """混合去重：先 ID 去重，再相似度去重。"""
        # 第一步: ID 去重
        id_deduped = self.dedup_by_id(items)
        # 第二步: 相似度去重
        final = self.dedup_by_similarity(id_deduped)
        return final
```

## 产生的结果

记忆联邦产生的核心价值是**统一记忆抽象层**，它从根本上改变了 Agent 与记忆系统的交互方式。

### 1. 单一记忆 API

无论底层有多少种记忆源，Agent 面对的始终是一个统一的查询接口：

```python
# Agent 只需这一行
results = await memory_federation.query("Transformer 架构演进")

# 而不是这样：
# vec = await vector_db.search(...)
# sql = await sql_db.query(...)
# kg = await kg_db.traverse(...)
# doc = await doc_store.search(...)
# results = manual_merge(vec, sql, kg, doc)
```

**对比**：

```text
无联邦的 Agent 代码:
  约 30-50 行 // 查询路由 + 各源调用 + 手动融合

有联邦的 Agent 代码:
  约 1 行     // memory_federation.query(text)

代码减少: 97%+
认知负荷下降: Agent 不需要理解底层存储细节
```

### 2. 优雅降级（Graceful Degradation）

当某个记忆源故障时，联邦层自动降级，不阻塞整个查询：

```text
场景: 知识图谱服务宕机

无联邦:
  Agent 调用知识图谱 → 超时 → Agent 捕获异常
  → "我需要查知识图谱但失败了，我该怎么做？"
  → 要么放弃整个查询，要么写复杂的 fallback 逻辑

有联邦:
  Agent 调用联邦接口 → 联邦引擎并行查询
  → 知识图谱超时 → 联邦引擎记录错误，跳过该源
  → 其他源正常返回 → 联邦引擎融合后返回
  → Agent 收到正常结果（少了知识图谱的信息）
  → 联邦引擎在日志中记录: [WARN] knowledge_graph 超时
```

**效果**：

```text
单源故障影响:
  - 无联邦: Agent 任务可能完全失败
  - 有联邦: Agent 获得部分结果，任务继续

多源故障影响:
  - 无联邦: Agent 代码中的 try-catch 异常复杂
  - 有联邦: 只要有至少一个源正常工作，就能返回结果

完美的渐进式体验:
  所有源正常 → 最佳结果
  1 个源故障 → 良好结果（损失一部分信息）
  2 个源故障 → 基础结果（关键信息还在）
  所有源故障 → 返回空结果 + 错误报告（Agent 做降级决策）
```

### 3. 最佳全源检索（Best-of-All-Worlds Retrieval）

联邦层让 Agent 获得"每个源的最佳能力"：

```text
同一查询在不同源中获得的信息:

  向量数据库:
    → "Transformer 使用自注意力机制，2017 年提出"
    → 语义匹配，找到最相关的内容片段

  知识图谱:
    → "Transformer → 提出者: Vaswani et al. → 论文: Attention is All You Need"
    → 实体关系，提供精确的结构化信息

  文档存储:
    → "Transformer 架构由 encoder 和 decoder 组成，每层包含多头注意力和 FFN"
    → 全文搜索，找到包含特定关键词的段落

  外部 API（如 arXiv）:
    → "Attention Is All You Need 被引用 10万+ 次"
    → 实时数据，提供最新的统计信息

  融合结果:
    1. [实体] Transformer → Vaswani et al., 2017 (知识图谱)
    2. [概念] 自注意力机制 (向量数据库)
    3. [结构] Encoder-Decoder 架构 (文档存储)
    4. [统计] 引用 10万+ 次 (外部 API)
    → Agent 获得一个信息全面、多角度的答案
```

### 4. 可插拔架构

新记忆源的加入不需要修改 Agent 代码，也不需要修改联邦引擎核心逻辑：

```text
添加新源的步骤:

  步骤 1: 实现 MemorySource 适配器
    class NewSourceAdapter(MemorySource):
        async def query(self, query):
            ...
        ...

  步骤 2: 注册到联邦引擎
    federation.register_source("new_source", NewSourceAdapter(...))

  步骤 3: 完成 ✓ (Agent 代码零修改)

这种可插拔架构使得:
  - 记忆系统可以随 Agent 需求增长而扩展
  - 不同团队可以独立开发和维护不同的记忆源
  - 迁移到新的存储技术时，只需要替换对应的适配器
```

## 最大挑战：性能退化

记忆联邦最突出的问题是**性能退化**——联邦层的整体延迟总是被最慢的记忆源所拖累。

### 短板效应分析

```text
联邦查询延迟公式:

  Latency_total = max(Latency_s1, Latency_s2, ..., Latency_sn) + Latency_fusion

  Latency_fusion 通常可以忽略 (几毫秒)
  所以: Latency_total ≈ max(各源延迟)

如果各源延迟差异极大:
  源 A: 10ms  (缓存层)
  源 B: 50ms  (向量库)
  源 C: 200ms (SQL 数据库)
  源 D: 800ms (知识图谱)
  ──────────────────────
  总延迟: ~800ms

  → 86% 的时间在等一个源
  → 源 D 拖累了所有查询
  → 即使 Agent 只需要来自源 A 的信息
```

### 影响范围

```text
性能退化在不同场景下的影响:

实时对话 Agent:
  用户期望响应 < 1 秒
  如果最慢源 2 秒 → 用户体验严重下降
  影响: 高

批量处理 Agent:
  处理时间 10-30 秒
  多等 1-2 秒影响不大
  影响: 低

流式 Agent:
  可以渐进输出结果
  先显示部分结果，慢源结果补上
  影响: 中（可控）
```

### 三大解决方案

**方案 1：逐源超时（Per-Source Timeout）**

```text
为每个源设置独立的超时时间：

  源 A（向量库）:  超时 100ms
  源 B（SQL）:     超时 500ms
  源 C（知识图谱）: 超时 300ms
  源 E（外部 API）: 超时 2000ms

查询总超时 = max(各源超时) = 2000ms

但: 源 A 在 100ms 内返回了结果
    源 B 在 200ms 内返回了结果
    源 C 超时 → 跳过
    源 E 在 1500ms 返回结果

有效延迟: 1500ms (等到了源 E)
最差延迟: 2000ms (等所有源超时)

优势: 慢源不会拖死整个查询
代价: 丢失超时源的数据
```

**方案 2：渐进式渲染（Progressive Rendering）**

```text
借鉴前端领域的"渐进式渲染"理念：

  时间线:
  0ms      ┌── 发起联邦查询
  50ms     │── 源 A 返回 → 联邦引擎立即输出部分结果
           │   Agent 开始处理 A 的结果
  100ms    │── 源 B 返回 → 融合后追加输出
           │   Agent 更新推理
  200ms    │── 源 C 超时跳过
  500ms    │── 源 D 返回 → 最终融合输出
           │   Agent 完成推理

  效果: Agent 在 50ms 时就开始工作了！
  实际总处理时间 = 50ms (第一批) + 后续增量更新
  用户不会感到在"等待"
```

**方案 3：投机执行（Speculative Execution）**

```text
基于历史数据预测最慢源的结果，提前返回：

  历史模式:
    - 源 D（知识图谱）80% 的情况下返回 Top-5 相同
    - 只有 20% 的情况有新结果

  投机策略:
    1. 并行执行所有源
    2. 当快源返回后，使用最近一次源 D 的缓存结果
    3. 立即进行融合并返回
    4. 源 D 返回后，如果结果不同，更新并重新排序
    5. 通知 Agent 有更新

  收益:
    - 90% 的情况下 Agent 在 50ms 就拿到了"几乎正确"的结果
    - 10% 的情况需要 800ms 等待源 D 的新结果
    - 平均延迟大幅降低
```

**综合方案——自适应延迟管理器**：

```python
import asyncio
import time
from typing import AsyncIterator


class ProgressiveFusionEngine:
    """
    支持渐进式渲染的联邦引擎。

    快源结果先返回，慢源结果后补充，Agent 可以边接收边处理。
    """

    def __init__(self, sources: dict[str, MemorySource],
                 global_timeout: float = 2.0):
        self.sources = sources
        self.global_timeout = global_timeout

    async def query_progressive(self, query: Query) -> AsyncIterator[list[ResultItem]]:
        """
        渐进式查询 —— 每个源的结果就绪后立即 yield。

        Agent 可以这样使用:
          async for partial_results in engine.query_progressive(query):
              agent.update_memory(partial_results)
        """
        async def query_source(name: str, source: MemorySource) -> tuple[str, list[ResultItem]]:
            try:
                start = time.monotonic()
                results = await asyncio.wait_for(
                    source.query(query),
                    timeout=self._get_source_timeout(name)
                )
                elapsed = (time.monotonic() - start) * 1000
                logger.info(f"源 {name} 返回 {len(results)} 条结果 ({elapsed:.0f}ms)")
                return name, results
            except asyncio.TimeoutError:
                logger.warning(f"源 {name} 超时")
                return name, []
            except Exception as e:
                logger.error(f"源 {name} 错误: {e}")
                return name, []

        # 按预期延迟排序源（快源先执行）
        sorted_sources = sorted(
            self.sources.items(),
            key=lambda x: self._expected_latency(x[0])
        )

        # 并行执行，但结果逐个 yield
        tasks = {
            name: asyncio.create_task(query_source(name, src))
            for name, src in sorted_sources
        }

        # 使用 as_completed 逐个获取结果
        for coro in asyncio.as_completed(tasks.values(),
                                          timeout=self.global_timeout):
            name, results = await coro
            if results:  # 只 yield 有结果的源
                yield results

    def _get_source_timeout(self, source_name: str) -> float:
        """根据源类型返回超时时间。"""
        timeouts = {
            "vector_db": 0.5,
            "sql_db": 2.0,
            "knowledge_graph": 3.0,
            "document_store": 1.0,
            "external_api": 5.0,
        }
        return timeouts.get(source_name, 1.0)

    def _expected_latency(self, source_name: str) -> float:
        """返回该源的预期延迟（秒），用于排序。"""
        latencies = {
            "vector_db": 0.05,
            "sql_db": 0.2,
            "document_store": 0.1,
            "knowledge_graph": 0.5,
            "external_api": 0.8,
        }
        return latencies.get(source_name, 0.5)
```

## 能力边界

### 1. 源类型无限扩展（通过适配器）

联邦架构理论上可以接入任意类型的记忆源——只要实现 `MemorySource` 接口：

```text
已支持的源类型:
  ├── 向量数据库 (Pinecone, Chroma, Weaviate, Qdrant)
  ├── 关系型数据库 (PostgreSQL, MySQL, SQLite)
  ├── 知识图谱 (Neo4j, Amazon Neptune)
  ├── 文档存储 (MongoDB, Elasticsearch)
  ├── Key-Value 存储 (Redis, DynamoDB)
  ├── 外部 API (REST, gRPC, GraphQL)
  ├── 文件系统 (本地文件, S3, GCS)
  └── 其他 Agent 的记忆 (peer-to-peer federation)

每种新源只需要:
  1. 实现 MemorySource 接口
  2. 注册到联邦引擎
  3. 配置超时和权重
```

### 2. 查询延迟下界

```text
联邦查询的延迟下界不可能低于:

  Latency_best = min(first_source_result)

即使所有源都极快，联邦层至少需要:
  - 查询分发开销 (网络 + 序列化)
  - 最慢的查询源的时间
  - 融合计算时间

理论最小延迟 ≈ 2-5ms (所有源都在本地内存中)

实际典型延迟:
  - 全部本地源: 10-100ms
  - 包含远程源: 100-1000ms
  - 包含外部 API: 200ms-5s
```

### 3. 结果质量依赖融合算法

联邦查询的结果质量**上限**取决于融合算法的质量：

```text
融合算法质量对结果的影响:

输入:
  源 A: [doc_1(0.95), doc_2(0.90), doc_3(0.85)]
  源 B: [doc_3(100), doc_4(80), doc_5(60)]

RRF 融合 (k=60):
  → doc_3: 1/61 + 1/61 = 0.0328  (第 1 名)
  → doc_1: 1/61 = 0.0164           (第 2 名)
  → doc_2: 1/62 = 0.0161           (第 3 名)
  → doc_4: 1/62 = 0.0161           (并列第 3)
  → doc_5: 1/63 = 0.0159           (第 5 名)

加权融合 (A:0.7, B:0.3):
  → doc_1: 0.7*0.95 + 0.3*0 = 0.665 (第 1 名)
  → doc_2: 0.7*0.90 + 0.3*0 = 0.630 (第 2 名)
  → doc_3: 0.7*0.85 + 0.3*1 = 0.595 (第 3 名)
  → doc_4: 0.7*0 + 0.3*0.8 = 0.240 (第 4 名)

结论: 不同融合算法产生不同排序，选择融合算法本身就是一个重要决策
```

### 4. 无法克服个体源限制

联邦层不能弥补单个源的固有限制：

```text
各源的固有限制:

  向量数据库:
    - 稀疏语义理解能力（对罕见查询效果差）
    - 联邦层无法改善 → 需要更好的 embedding 模型

  关系型数据库:
    - 不支持语义搜索
    - 联邦层无法改善 → 可以补充向量搜索结果

  外部 API:
    - 受 API 速率限制
    - 联邦层无法加速 → 需要缓存 + 降级策略

  知识图谱:
    - 图谱不完备时召回率低
    - 联邦层无法补全 → 需要多源互补

重要原则: 联邦层组合已有能力，无法创造新能力
```

### 5. 增加架构复杂度

```text
引入联邦层的成本:

  新增组件:
    - 联邦引擎服务（新的部署单元）
    - 适配器层（每个源一个适配器）
    - 融合计算模块
    - 健康监控系统
    - 缓存层

  新增的运维关注点:
    - 联邦引擎本身的可用性和扩展性
    - 各源连接池管理
    - 超时和断路器配置
    - 监控报警（每个源 + 联邦层）
    - 联邦层的性能问题排查（分布式追踪）

  复杂度量化:
    项目规模        无联邦        有联邦
    ─────────────────────────────────────
    组件数          3-5            5-10
    配置项          10-20          30-50
    运维脚本        1-3            5-10
    分布式追踪     不需要           需要 (跨源追踪)
```

## 对比：联邦 vs 其他方案

### 方案一：单一大一统存储（Single Unified Store）

```text
思路: 将所有记忆数据迁移到一个全能型存储中（如 PostgreSQL + pgvector）

优势:
  - 架构最简单
  - 运维最方便
  - 事务一致性最强
  - 查询延迟最低（本地查询）

劣势:
  - pgvector 的向量检索性能不如专用向量库
  - 知识图谱能力弱（需要额外扩展）
  - 扩展性受限于单体存储
  - 如果存量数据已经在多个系统中，迁移成本极高

适用场景:
  - 项目早期，数据量小
  - 查询类型单一（主要是向量搜索 + 简单过滤）
  - 没有历史存量数据需要兼容
```

### 方案二：ETL 数据整合（ETL-Based Consolidation）

```text
思路: 定期从各源 ETL 抽取数据，加载到一个统一的分析存储中

流程:
  [源 A] ──→ 定时 ETL ──┐
  [源 B] ──→ 定时 ETL ──┼──→ [统一数据仓库] ←── [Agent 查询]
  [源 C] ──→ 定时 ETL ──┘

优势:
  - 查询时只有一个源，延迟可控
  - 可以预先做数据清洗和关联
  - 数据质量高

劣势:
  - 数据不是实时的（ETL 周期决定新鲜度）
  - 需要大量的 ETL 开发和维护工作
  - 数据量大时 ETL 过程耗时长
  - 新加源需要修改 ETL 流水线

适用场景:
  - 对数据一致性要求高
  - 数据实时性要求不高（小时级/天级更新）
  - 已有数据仓库基础设施
```

### 方案三：Agent 自行多源查询（Manual Multi-Query）

```text
思路: 不给 Agent 联邦层，让 Agent 自己决定查哪些源、怎么查、怎么合并

优势:
  - 实现最简单（不需要联邦基础设施）
  - Agent 有最大灵活性（可以自创查询策略）
  - 适合需要深度定制查询逻辑的场景

劣势:
  - Agent 的推理负担重（每次都要"想"怎么查）
  - 查询逻辑分散在各处，难以维护
  - 每个 Agent 都要实现自己的融合逻辑
  - 新加源需要修改所有 Agent 的代码
  - 错误处理不一致（各写各的 try-catch）

适用场景:
  - 原型开发阶段
  - 查询逻辑 Agent 自身最清楚
  - 只有 1-2 个 Agent，改动可控
```

### 方案四：记忆联邦（Memory Federation）—— 本方案

```text
思路: 联邦层作为记忆中间件，统一管理查询路由、并行执行和结果融合

优势:
  - Agent 零负担（只需要调一个接口）
  - 各源独立演进（加源不改 Agent 代码）
  - 优雅降级（单源故障不瘫痪）
  - 统一监控和运维
  - 可插拔架构

劣势:
  - 架构复杂（新增联邦层组件）
  - 性能受最慢源拖累（需要渐进式渲染等优化）
  - 源特定能力被接口抽象隐藏

适用场景:
  - 中大型 Agent 项目（3+ 个记忆源）
  - 需要统一管理和运维
  - 需要高可用性和容错能力
  - 团队有多个人维护不同记忆系统
```

### 四方案对比矩阵

```text
维度              大一统存储    ETL 整合    Agent 自查询    记忆联邦
────────────────────────────────────────────────────────────────────
架构复杂度         低          中           低              高
查询延迟           低          低           中              中-高
数据实时性         实时         延迟         实时             实时
一致性             强          强           弱              最终
容错能力           低          高           低              高
扩展性             低          中           中              高
Agent 负担         低          低           高              低
开发维护成本        低          高           中              中-高
数据迁移成本       高          高           低              低
运维复杂度         低          中           低              中
```

## 工程优化

### 1. 超时管理与截止时间传播

联邦查询需要精细的超时控制机制。关键在于**截止时间（Deadline）传播**——每个子查询知道自己还有多少时间可用。

**原理**：

```text
Agent 设定: "我在 2 秒内需要结果"

截止时间传播:
  联邦引擎收到请求: remaining = 2000ms
    ├── 查询分解 + 路由: 50ms → remaining = 1950ms
    ├── 并行查询各源:
    │   ├── 源 A: timeout = 1950ms × 0.3 = 585ms
    │   ├── 源 B: timeout = 1950ms × 0.5 = 975ms
    │   └── 源 C: timeout = 1950ms × 0.2 = 390ms
    ├── 结果收集: 50ms → remaining = 1900ms - max(各源实际耗时)
    └── 融合计算: remaining × 0.1 (最多保留 10% 给融合)
```

**Python 实现**：

```python
import asyncio
import time


class Deadline:
    """
    截止时间管理器 —— 在整个联邦查询链路中传播时间预算。
    """

    def __init__(self, timeout_ms: float):
        self.deadline = time.monotonic() + timeout_ms / 1000.0
        self.timeout_ms = timeout_ms

    @property
    def remaining_ms(self) -> float:
        """剩余时间（毫秒）。"""
        remaining = (self.deadline - time.monotonic()) * 1000.0
        return max(0, remaining)

    @property
    def expired(self) -> bool:
        return self.remaining_ms <= 0

    def allocate(self, fraction: float, name: str = "") -> float:
        """
        分配一部分剩余时间给某个子任务。

        Args:
            fraction: 0.0-1.0 的百分比
            name: 子任务名称（用于日志）

        Returns:
            分配的时间预算（毫秒）
        """
        allocated = self.remaining_ms * fraction
        logger.debug(f"[Deadline] {name}: 分配 {allocated:.0f}ms "
                     f"(剩余 {self.remaining_ms:.0f}ms)")
        return allocated


class TimeoutAwareFederationEngine:
    """
    超时感知的联邦引擎 —— 每个源查询分配独立时间预算。
    """

    def __init__(self, sources: dict[str, MemorySource]):
        self.sources = sources

    async def query(self, query: Query, timeout_ms: float = 2000.0) -> list[ResultItem]:
        deadline = Deadline(timeout_ms)

        # 阶段 1: 路由和分解（分配 5% 时间预算）
        route_budget = deadline.allocate(0.05, "query_routing")
        if route_budget < 10:
            route_budget = 10  # 至少 10ms
        route_plan = await asyncio.wait_for(
            self._route_query(query),
            timeout=route_budget / 1000.0
        )

        if deadline.expired:
            logger.warning("查询在路由阶段已超时")
            return []

        # 阶段 2: 并行查询（分配 85% 时间预算）
        query_budget = deadline.allocate(0.85, "parallel_query")
        source_timeout = query_budget / max(len(route_plan.sources), 1)

        async def query_source(name: str) -> list[ResultItem]:
            source = self.sources[name]
            try:
                return await asyncio.wait_for(
                    source.query(query),
                    timeout=source_timeout / 1000.0
                )
            except asyncio.TimeoutError:
                logger.warning(f"[{name}] 超时 ({source_timeout:.0f}ms)")
                return []
            except Exception as e:
                logger.error(f"[{name}] 错误: {e}")
                return []

        tasks = [query_source(name) for name in route_plan.sources]
        source_results = await asyncio.gather(*tasks)
        results_by_source = dict(zip(route_plan.sources, source_results))

        # 阶段 3: 融合（分配 10% 时间预算）
        if not deadline.expired:
            fusion_budget = deadline.allocate(0.10, "fusion")
            if fusion_budget > 5:
                return await self._fuse(results_by_source)

        # 没有融合时间了，直接拼接
        return self._fast_merge(results_by_source)

    async def _route_query(self, query: Query) -> "RoutePlan":
        """查询路由 —— 选择查询哪些源。"""
        # 此处省略具体路由逻辑
        return RoutePlan(sources=list(self.sources.keys()))

    async def _fuse(self, results: dict[str, list[ResultItem]]) -> list[ResultItem]:
        """执行完整融合。"""
        # 使用 RRF 融合
        fused = reciprocal_rank_fusion(
            {name: [RankedItem(...) for ...] for name, items in results.items()}
        )
        return [ResultItem(id=id, score=score) for id, score in fused]

    def _fast_merge(self, results: dict[str, list[ResultItem]]) -> list[ResultItem]:
        """快速合并 —— 不做融合，直接取每个源 Top-1 交替排列。"""
        merged = []
        max_len = max((len(items) for items in results.values()), default=0)
        for i in range(max_len):
            for name in results:
                if i < len(results[name]):
                    merged.append(results[name][i])
        return merged
```

### 2. 自适应源选择

根据源的实时性能和查询需求，动态决定查询哪些源。

**选择策略**：

```python
class AdaptiveSourceSelector:
    """
    自适应源选择器 —— 根据实时指标决定跳过哪些源。
    """

    def __init__(self, sources: dict[str, MemorySource]):
        self.sources = sources
        self.latency_history: dict[str, list[float]] = {
            name: [] for name in sources
        }
        self.error_history: dict[str, list[bool]] = {
            name: [] for name in sources
        }

    def record_result(self, source_name: str, latency_ms: float, success: bool):
        """记录一次查询结果。"""
        self.latency_history[source_name].append(latency_ms)
        self.error_history[source_name].append(not success)

        # 只保留最近 100 次记录
        if len(self.latency_history[source_name]) > 100:
            self.latency_history[source_name].pop(0)
            self.error_history[source_name].pop(0)

    def select_sources(self, query_type: str,
                       max_latency_ms: float = 500.0) -> list[str]:
        """
        选择参与查询的源。

        Args:
            query_type: 查询类型（"semantic", "exact", "hybrid"）
            max_latency_ms: 可接受的最大延迟

        Returns:
            选中的源名称列表
        """
        candidates = []

        for name in self.sources:
            # 1. 检查健康状况
            if self._error_rate(name) > 0.3:  # 错误率 > 30% 跳过
                logger.info(f"跳过源 {name}: 错误率 {self._error_rate(name):.1%}")
                continue

            # 2. 检查延迟
            avg_latency = self._avg_latency(name)
            if avg_latency > max_latency_ms * 1.5:
                logger.info(f"跳过源 {name}: 平均延迟 {avg_latency:.0f}ms > 阈值")
                continue

            # 3. 检查查询类型匹配
            if self._type_match(name, query_type):
                candidates.append(name)

        return candidates

    def _error_rate(self, name: str) -> float:
        history = self.error_history.get(name, [])
        if not history:
            return 0.0
        return sum(history) / len(history)

    def _avg_latency(self, name: str) -> float:
        history = self.latency_history.get(name, [])
        if not history:
            return 0.0
        return sum(history) / len(history)

    def _type_match(self, source_name: str, query_type: str) -> bool:
        """判断源是否支持该查询类型。"""
        type_map = {
            "vector_db": ["semantic", "hybrid"],
            "sql_db": ["exact", "hybrid"],
            "knowledge_graph": ["relation", "exact"],
            "document_store": ["fulltext", "hybrid"],
            "external_api": ["realtime", "exact"],
        }
        return query_type in type_map.get(source_name, [])
```

### 3. 联邦结果缓存

```python
import hashlib
import json
import time
from collections import OrderedDict


class FederatedCache:
    """
    联邦查询结果缓存。

    缓存整个联邦查询的融合结果，避免重复 fan-out 查询。
    """

    def __init__(self, capacity: int = 1000, default_ttl: float = 60.0):
        self.capacity = capacity
        self.default_ttl = default_ttl
        self._store: dict[str, tuple[float, list[dict]]] = {}
        self._lru = OrderedDict()

    def _make_key(self, query_text: str, sources: frozenset[str],
                  params: dict | None = None) -> str:
        """生成缓存键。"""
        raw = json.dumps({
            "query": query_text.strip().lower(),
            "sources": sorted(sources),
            "params": params,
        }, sort_keys=True)
        return hashlib.sha256(raw.encode()).hexdigest()

    def get(self, query_text: str, sources: frozenset[str],
            params: dict | None = None) -> list[dict] | None:
        """获取缓存的联邦查询结果。"""
        key = self._make_key(query_text, sources, params)
        if key not in self._store:
            return None

        expires_at, results = self._store[key]
        if time.monotonic() > expires_at:
            # 过期
            del self._store[key]
            self._lru.pop(key, None)
            return None

        # 更新 LRU
        self._lru.move_to_end(key)
        return results

    def set(self, query_text: str, sources: frozenset[str],
            results: list[dict], ttl: float | None = None,
            params: dict | None = None):
        """缓存联邦查询结果。"""
        key = self._make_key(query_text, sources, params)

        # 容量管理：淘汰最旧
        if len(self._store) >= self.capacity:
            oldest = next(iter(self._lru))
            del self._store[oldest]
            self._lru.pop(oldest, None)

        self._store[key] = (time.monotonic() + (ttl or self.default_ttl), results)
        self._lru[key] = None
        self._lru.move_to_end(key)

    def invalidate_for_source(self, source_name: str):
        """
        当某个源的数据更新时，使所有涉及该源的缓存失效。

        注意：这是开销较大的操作，需要遍历所有缓存项。
        """
        invalidated = []
        for key in list(self._store.keys()):
            meta = json.loads(
                hashlib.sha256(bytes.fromhex(key))
            )  # 简化处理，实际需要存储元信息
            invalidated.append(key)
        for key in invalidated:
            self._store.pop(key, None)
            self._lru.pop(key, None)
```

### 4. 查询计划优化

对复杂的多步骤查询，构建最优的执行计划。

```python
from dataclasses import dataclass, field


@dataclass
class QueryPlan:
    """
    查询执行计划 —— 描述查询的 DAG 执行流程。
    """
    nodes: list["PlanNode"] = field(default_factory=list)
    edges: list[tuple[int, int]] = field(default_factory=list)  # (from, to)


@dataclass
class PlanNode:
    """计划节点 —— 一个子查询步骤。"""
    id: int
    source: str
    query: str
    depends_on: list[int] = field(default_factory=list)
    estimated_cost: float = 0.0


class QueryOptimizer:
    """
    查询优化器 —— 生成最优执行计划。
    """

    def __init__(self, sources: dict[str, MemorySource]):
        self.sources = sources
        # 源的统计信息
        self.source_stats: dict[str, dict] = {
            name: {
                "avg_latency_ms": 100,
                "selectivity": 0.1,  # 过滤率
                "throughput": 100,    # QPS
            }
            for name in sources
        }

    def optimize(self, plan: QueryPlan) -> QueryPlan:
        """
        对查询计划进行优化：
        1. 重排序节点（代价最小的先执行）
        2. 合并可并行的节点
        3. 下推过滤条件
        """
        # 1. 预估代价
        for node in plan.nodes:
            node.estimated_cost = self._estimate_cost(node)

        # 2. 按依赖关系拓扑排序，同级按代价升序
        optimized_nodes = self._topological_sort(plan.nodes)

        # 3. 识别可并行节点（无依赖关系的节点）
        # 已在执行层面处理，计划层面保持 DAG

        return QueryPlan(nodes=optimized_nodes, edges=plan.edges)

    def _estimate_cost(self, node: PlanNode) -> float:
        """预估节点的执行代价（毫秒）。"""
        stats = self.source_stats.get(node.source, {})
        base_latency = stats.get("avg_latency_ms", 100)

        # 查询复杂度因子
        query_length = len(node.query)
        complexity_factor = 1.0 + (query_length / 1000) * 0.1

        return base_latency * complexity_factor

    def _topological_sort(self, nodes: list[PlanNode]) -> list[PlanNode]:
        """拓扑排序，同级节点按代价升序排列。"""
        visited = set()
        result = []

        def dfs(node_id: int):
            if node_id in visited:
                return
            visited.add(node_id)
            node = next(n for n in nodes if n.id == node_id)

            # 先处理依赖
            for dep_id in node.depends_on:
                dfs(dep_id)

            result.append(node)

        # 先处理无依赖的节点（按代价升序）
        independent = sorted(
            [n for n in nodes if not n.depends_on],
            key=lambda n: n.estimated_cost
        )
        dependent = [n for n in nodes if n.depends_on]

        all_sorted = independent + dependent
        for node in all_sorted:
            if node.id not in visited:
                dfs(node.id)

        return result
```

### 5. 监控体系

完整的联邦层监控需要覆盖每个源和联邦整体两个维度。

**关键指标**：

```python
from dataclasses import dataclass, field
from datetime import datetime
import statistics


@dataclass
class SourceMetrics:
    """单个记忆源的监控指标。"""
    source_name: str
    query_count: int = 0
    success_count: int = 0
    error_count: int = 0
    timeout_count: int = 0
    latencies_ms: list[float] = field(default_factory=list)
    result_counts: list[int] = field(default_factory=list)
    last_query_time: datetime | None = None

    @property
    def error_rate(self) -> float:
        if self.query_count == 0:
            return 0.0
        return self.error_count / self.query_count

    @property
    def p50_latency(self) -> float:
        if not self.latencies_ms:
            return 0.0
        return statistics.median(self.latencies_ms)

    @property
    def p99_latency(self) -> float:
        if not self.latencies_ms:
            return 0.0
        sorted_lat = sorted(self.latencies_ms)
        idx = int(len(sorted_lat) * 0.99)
        return sorted_lat[min(idx, len(sorted_lat) - 1)]

    @property
    def avg_results(self) -> float:
        if not self.result_counts:
            return 0.0
        return statistics.mean(self.result_counts)

    def record(self, latency_ms: float, success: bool,
               is_timeout: bool, result_count: int):
        """记录一次查询。"""
        self.query_count += 1
        if success:
            self.success_count += 1
        else:
            self.error_count += 1
        if is_timeout:
            self.timeout_count += 1
        self.latencies_ms.append(latency_ms)
        self.result_counts.append(result_count)
        self.last_query_time = datetime.utcnow()

        # 只保留最近 1000 条记录
        if len(self.latencies_ms) > 1000:
            self.latencies_ms.pop(0)
            self.result_counts.pop(0)

    def report(self) -> dict:
        """生成监控报告。"""
        return {
            "source": self.source_name,
            "query_count": self.query_count,
            "success_count": self.success_count,
            "error_rate": f"{self.error_rate:.1%}",
            "p50_latency_ms": f"{self.p50_latency:.0f}",
            "p99_latency_ms": f"{self.p99_latency:.0f}",
            "avg_results_per_query": f"{self.avg_results:.1f}",
            "last_query": str(self.last_query_time),
        }


@dataclass
class FederationMetrics:
    """联邦整体监控指标。"""
    total_queries: int = 0
    avg_fusion_time_ms: float = 0.0
    source_metrics: dict[str, SourceMetrics] = field(default_factory=dict)

    def record_source_query(self, source: str, latency_ms: float,
                            success: bool, is_timeout: bool,
                            result_count: int):
        if source not in self.source_metrics:
            self.source_metrics[source] = SourceMetrics(source_name=source)
        self.source_metrics[source].record(
            latency_ms, success, is_timeout, result_count
        )
        self.total_queries += 1

    def report(self) -> dict:
        """生成完整监控报告。"""
        source_reports = {
            name: metrics.report()
            for name, metrics in self.source_metrics.items()
        }
        return {
            "total_federated_queries": self.total_queries,
            "source_count": len(self.source_metrics),
            "sources": source_reports,
            "alerts": self._generate_alerts(),
        }

    def _generate_alerts(self) -> list[str]:
        """根据指标生成告警。"""
        alerts = []
        for name, metrics in self.source_metrics.items():
            if metrics.error_rate > 0.1:
                alerts.append(f"[WARN] {name} 错误率 {metrics.error_rate:.1%} > 10%")
            if metrics.timeout_count > 10:
                alerts.append(f"[WARN] {name} 超时次数 {metrics.timeout_count}")
            if metrics.p99_latency > 1000:
                alerts.append(f"[WARN] {name} p99 延迟 {metrics.p99_latency:.0f}ms > 1s")
        return alerts
```

## 代码示例

### 完整记忆联邦引擎实现

```python
"""
记忆联邦引擎 —— 完整实现。

包含:
  - 适配器模式（MemorySource 接口）
  - 多种记忆源实现（向量库、SQL、知识图谱、文档存储）
  - 查询路由
  - 并行执行 + 超时控制
  - RRF 融合 + 加权融合
  - 去重
  - 熔断器
  - 监控
"""

import asyncio
import hashlib
import logging
import time
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, AsyncIterator

logger = logging.getLogger(__name__)


# ═══════════════════════════════════════════
# 基础数据结构
# ═══════════════════════════════════════════

@dataclass
class ResultItem:
    """统一的结果项。"""
    id: str
    content: str
    score: float
    source: str
    metadata: dict[str, Any] = field(default_factory=dict)


@dataclass
class Query:
    """统一查询对象。"""
    text: str
    top_k: int = 10
    filters: dict[str, Any] = field(default_factory=dict)


@dataclass
class RoutePlan:
    """路由计划。"""
    sources: list[str]
    decomposed_queries: dict[str, str] = field(default_factory=dict)


# ═══════════════════════════════════════════
# 记忆源适配器接口
# ═══════════════════════════════════════════

class MemorySource(ABC):
    """记忆源抽象基类。"""

    def __init__(self, name: str):
        self.name = name

    @abstractmethod
    async def query(self, query: Query) -> list[ResultItem]:
        """执行查询，返回结果列表。"""
        ...

    @abstractmethod
    async def health_check(self) -> bool:
        """健康检查。"""
        ...

    @property
    @abstractmethod
    def source_type(self) -> str:
        """源类型标识。"""
        ...


class VectorDBSource(MemorySource):
    """向量数据库适配器（模拟实现）。"""

    def __init__(self, name: str = "vector_db"):
        super().__init__(name)
        # 模拟数据
        self._documents = [
            {"id": "v1", "content": "Transformer 使用自注意力机制处理序列数据",
             "score": 0.95},
            {"id": "v2", "content": "BERT 是基于 Transformer 的预训练模型",
             "score": 0.92},
            {"id": "v3", "content": "GPT 使用单向 Transformer 解码器架构",
             "score": 0.88},
        ]

    async def query(self, query: Query) -> list[ResultItem]:
        await asyncio.sleep(0.05)  # 模拟 50ms 延迟
        return [
            ResultItem(
                id=doc["id"],
                content=doc["content"],
                score=doc["score"],
                source=self.name,
            )
            for doc in self._documents[:query.top_k]
        ]

    async def health_check(self) -> bool:
        return True

    @property
    def source_type(self) -> str:
        return "vector"


class SQLDatabaseSource(MemorySource):
    """SQL 数据库适配器（模拟实现）。"""

    def __init__(self, name: str = "sql_db"):
        super().__init__(name)
        self._records = [
            {"id": "s1", "content": "论文: Attention Is All You Need (2017)",
             "score": 1.0},
            {"id": "s2", "content": "作者: Vaswani et al., Google Research",
             "score": 1.0},
        ]

    async def query(self, query: Query) -> list[ResultItem]:
        await asyncio.sleep(0.2)  # 模拟 200ms 延迟
        keyword = query.text.lower()
        results = [
            ResultItem(
                id=rec["id"],
                content=rec["content"],
                score=rec["score"] if keyword in rec["content"].lower() else 0.3,
                source=self.name,
            )
            for rec in self._records
        ]
        return [r for r in results if r.score > 0]

    async def health_check(self) -> bool:
        return True

    @property
    def source_type(self) -> str:
        return "structured"


class KnowledgeGraphSource(MemorySource):
    """知识图谱适配器（模拟实现）。"""

    def __init__(self, name: str = "knowledge_graph"):
        super().__init__(name)
        self._triples = [
            ("Transformer", "提出者", "Vaswani et al."),
            ("Transformer", "发表年份", "2017"),
            ("Transformer", "核心组件", "自注意力机制"),
            ("BERT", "基于", "Transformer"),
            ("GPT", "基于", "Transformer"),
        ]

    async def query(self, query: Query) -> list[ResultItem]:
        await asyncio.sleep(0.3)  # 模拟 300ms 延迟
        results = []
        keyword = query.text.lower()
        for i, (subj, pred, obj) in enumerate(self._triples):
            if keyword in subj.lower() or keyword in obj.lower():
                results.append(ResultItem(
                    id=f"kg_{i}",
                    content=f"{subj} -- {pred} --> {obj}",
                    score=0.9 if keyword in subj.lower() else 0.7,
                    source=self.name,
                    metadata={"subject": subj, "predicate": pred, "object": obj},
                ))
        return results

    async def health_check(self) -> bool:
        return True

    @property
    def source_type(self) -> str:
        return "graph"


class DocumentStoreSource(MemorySource):
    """文档存储适配器（模拟实现）。"""

    def __init__(self, name: str = "document_store"):
        super().__init__(name)
        self._docs = [
            {"id": "d1",
             "content": "Transformer 架构由堆叠的编码器和解码器层组成。"
                        "每层包含多头自注意力机制和前馈神经网络。",
             "score": 0.85},
            {"id": "d2",
             "content": "自注意力机制允许模型在处理序列时关注所有位置的"
                        "信息，解决了 RNN 的长距离依赖问题。",
             "score": 0.80},
        ]

    async def query(self, query: Query) -> list[ResultItem]:
        await asyncio.sleep(0.1)  # 模拟 100ms 延迟
        return [
            ResultItem(id=doc["id"], content=doc["content"],
                       score=doc["score"], source=self.name)
            for doc in self._docs[:query.top_k]
        ]

    async def health_check(self) -> bool:
        return True

    @property
    def source_type(self) -> str:
        return "document"


class ExternalAPISource(MemorySource):
    """外部 API 适配器（模拟实现）。"""

    def __init__(self, name: str = "external_api"):
        super().__init__(name)
        self._api_data = {
            "transformer": {
                "citation_count": "100,000+",
                "influence": "开创性工作",
                "applications": ["NLP", "CV", "语音"],
            }
        }

    async def query(self, query: Query) -> list[ResultItem]:
        await asyncio.sleep(0.4)  # 模拟 400ms 网络延迟
        keyword = query.text.lower()
        results = []
        for term, data in self._api_data.items():
            if keyword in term or term in keyword:
                results.append(ResultItem(
                    id=f"api_{term}",
                    content=f"{term}: 引用 {data['citation_count']}, "
                            f"影响: {data['influence']}",
                    score=0.95,
                    source=self.name,
                    metadata=data,
                ))
        return results

    async def health_check(self) -> bool:
        return True

    @property
    def source_type(self) -> str:
        return "external"


# ═══════════════════════════════════════════
# 查询路由器
# ═══════════════════════════════════════════

class QueryRouter:
    """
    查询路由器 —— 基于意图分类选择最优的查询源组合。
    """

    def __init__(self, sources: dict[str, MemorySource]):
        self.sources = sources
        self._source_priorities = {
            ("semantic",): ["vector_db", "document_store"],
            ("exact",): ["sql_db"],
            ("relation",): ["knowledge_graph"],
            ("realtime",): ["external_api"],
            ("semantic", "exact"): ["vector_db", "sql_db", "document_store"],
            ("semantic", "relation"): ["vector_db", "knowledge_graph"],
            ("all",): list(sources.keys()),
        }

    def _classify(self, query: Query) -> tuple[str, ...]:
        """对查询进行分类。"""
        text = query.text.lower()
        characteristics = []

        # 简单的基于规则的分类（生产环境应使用 ML 模型）
        if any(kw in text for kw in ["关系", "关联", "连接", "路径", "谁"]):
            characteristics.append("relation")
        if any(kw in text for kw in ["数据", "统计", "数量", "多少", "聚合"]):
            characteristics.append("exact")
        if any(kw in text for kw in ["实时", "最新", "当前", "天气", "价格"]):
            characteristics.append("realtime")
        if any(kw in text for kw in ["什么", "如何", "为什么", "介绍",
                                      "概念", "原理"]):
            characteristics.append("semantic")

        if not characteristics:
            characteristics.append("semantic")  # 默认语义查询

        return tuple(sorted(set(characteristics)))

    def route(self, query: Query, max_sources: int = 3) -> RoutePlan:
        """
        根据查询类型选择最优源组合。

        Args:
            query: 查询
            max_sources: 最大源数量

        Returns:
            路由计划
        """
        query_type = self._classify(query)
        logger.info(f"查询类型: {query_type}")

        # 查找匹配的源优先级列表
        for type_key, candidates in self._source_priorities.items():
            if set(query_type).issubset(set(type_key)):
                selected = [s for s in candidates
                           if s in self.sources][:max_sources]
                return RoutePlan(sources=selected)

        # 回退：使用所有健康源
        return RoutePlan(sources=list(self.sources.keys())[:max_sources])


# ═══════════════════════════════════════════
# 熔断器
# ═══════════════════════════════════════════

class CircuitBreakerState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"


class CircuitBreaker:
    """
    熔断器 —— 保护联邦层不反复调用故障源。
    """

    def __init__(self, failure_threshold: int = 5,
                 recovery_timeout: float = 30.0):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = CircuitBreakerState.CLOSED
        self.failure_count = 0
        self.last_failure_time = 0.0

    async def call(self, fn, fallback=None):
        """安全地调用带熔断保护的函数。"""
        if self.state == CircuitBreakerState.OPEN:
            if time.monotonic() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitBreakerState.HALF_OPEN
                logger.info(f"熔断器进入半开状态")
            else:
                logger.warning(f"熔断器开启，跳过调用")
                return fallback

        try:
            result = await fn()
            # 成功：重置
            self.failure_count = 0
            self.state = CircuitBreakerState.CLOSED
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.monotonic()
            if self.failure_count >= self.failure_threshold:
                self.state = CircuitBreakerState.OPEN
                logger.warning(f"熔断器触发开启 (阈值={self.failure_threshold})")
            return fallback


# ═══════════════════════════════════════════
# 融合算法
# ═══════════════════════════════════════════

class FusionAlgorithm:
    """融合算法基类。"""

    def fuse(self, results_by_source: dict[str, list[ResultItem]],
             top_k: int = 10) -> list[ResultItem]:
        """将多源结果融合为统一排序。"""
        raise NotImplementedError


class RRFFusion(FusionAlgorithm):
    """RRF 融合算法。"""

    def __init__(self, k: int = 60):
        self.k = k

    def fuse(self, results_by_source: dict[str, list[ResultItem]],
             top_k: int = 10) -> list[ResultItem]:
        scores: dict[str, float] = {}
        items_map: dict[str, ResultItem] = {}

        for source, items in results_by_source.items():
            for rank, item in enumerate(items, start=1):
                rrf_score = 1.0 / (self.k + rank)
                if item.id in scores:
                    scores[item.id] += rrf_score
                else:
                    scores[item.id] = rrf_score
                    items_map[item.id] = item

        # 按 RRF 分数排序
        sorted_ids = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        top_ids = [item_id for item_id, _ in sorted_ids[:top_k]]

        result = []
        for item_id in top_ids:
            item = items_map[item_id]
            item.score = scores[item_id]
            result.append(item)

        return result


class WeightedFusion(FusionAlgorithm):
    """加权融合算法。"""

    def __init__(self, weights: dict[str, float]):
        self.weights = weights

    def fuse(self, results_by_source: dict[str, list[ResultItem]],
             top_k: int = 10) -> list[ResultItem]:
        scores: dict[str, float] = {}
        items_map: dict[str, ResultItem] = {}
        source_item_ranks: dict[str, dict[str, int]] = {}

        for source, items in results_by_source.items():
            weight = self.weights.get(source, 1.0)
            source_item_ranks[source] = {}
            for rank, item in enumerate(items, start=1):
                weighted_score = weight * (1.0 / rank)
                if item.id in scores:
                    scores[item.id] += weighted_score
                else:
                    scores[item.id] = weighted_score
                    items_map[item.id] = item
                source_item_ranks[source][item.id] = rank

        sorted_ids = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        top_ids = [item_id for item_id, _ in sorted_ids[:top_k]]

        result = []
        for item_id in top_ids:
            item = items_map[item_id]
            item.score = scores[item_id]
            result.append(item)

        return result


# ═══════════════════════════════════════════
# 去重器
# ═══════════════════════════════════════════

class Deduplicator:
    """跨源去重器。"""

    def __init__(self, strategy: str = "content_hash"):
        self.strategy = strategy

    def dedup(self, items: list[ResultItem]) -> list[ResultItem]:
        """去除重复项。"""
        if self.strategy == "id":
            return self._dedup_by_id(items)
        elif self.strategy == "content_hash":
            return self._dedup_by_content(items)
        else:
            return items

    def _dedup_by_id(self, items: list[ResultItem]) -> list[ResultItem]:
        seen: set[str] = set()
        result = []
        for item in items:
            if item.id not in seen:
                seen.add(item.id)
                result.append(item)
        return result

    def _dedup_by_content(self, items: list[ResultItem]) -> list[ResultItem]:
        seen: set[str] = set()
        result = []
        for item in items:
            # 使用内容的前 50 个字符的哈希作为近似指纹
            fingerprint = hashlib.md5(
                item.content[:50].encode()
            ).hexdigest()
            if fingerprint not in seen:
                seen.add(fingerprint)
                result.append(item)
        return result


# ═══════════════════════════════════════════
# 监控器
# ═══════════════════════════════════════════

class MetricsCollector:
    """监控指标收集器。"""

    def __init__(self):
        self.source_latencies: dict[str, list[float]] = {}
        self.source_errors: dict[str, int] = {}
        self.source_successes: dict[str, int] = {}
        self.fusion_times: list[float] = []
        self.query_count = 0

    def record_source_query(self, source: str, latency_ms: float, success: bool):
        if source not in self.source_latencies:
            self.source_latencies[source] = []
            self.source_errors[source] = 0
            self.source_successes[source] = 0

        self.source_latencies[source].append(latency_ms)
        if success:
            self.source_successes[source] += 1
        else:
            self.source_errors[source] += 1

        # 只保留最近 1000 条
        if len(self.source_latencies[source]) > 1000:
            self.source_latencies[source].pop(0)

    def record_fusion(self, time_ms: float):
        self.fusion_times.append(time_ms)
        if len(self.fusion_times) > 1000:
            self.fusion_times.pop(0)

    def record_query(self):
        self.query_count += 1

    def get_report(self) -> dict[str, Any]:
        """生成监控报告。"""
        report = {
            "total_queries": self.query_count,
            "sources": {},
        }
        for source in self.source_latencies:
            latencies = self.source_latencies[source]
            avg_lat = sum(latencies) / len(latencies) if latencies else 0
            total = self.source_successes[source] + self.source_errors[source]
            error_rate = self.source_errors[source] / total if total > 0 else 0
            report["sources"][source] = {
                "avg_latency_ms": round(avg_lat, 1),
                "error_rate": round(error_rate, 3),
                "total_requests": total,
            }
        return report


# ═══════════════════════════════════════════
# 联邦引擎（主入口）
# ═══════════════════════════════════════════

class MemoryFederationEngine:
    """
    记忆联邦引擎主类。

    统一管理多个异构记忆源，提供单一的 query 接口。
    """

    def __init__(
        self,
        sources: dict[str, MemorySource] | None = None,
        fusion_algorithm: str = "rrf",
        source_weights: dict[str, float] | None = None,
        default_timeout_ms: float = 2000.0,
        enable_circuit_breaker: bool = True,
    ):
        self.sources: dict[str, MemorySource] = sources or {}
        self.default_timeout_ms = default_timeout_ms
        self.enable_circuit_breaker = enable_circuit_breaker

        # 初始化路由器
        self.router = QueryRouter(self.sources)

        # 初始化融合算法
        self.fusion_algorithm = self._create_fusion_algorithm(
            fusion_algorithm, source_weights
        )

        # 初始化解耦器
        self.deduplicator = Deduplicator(strategy="content_hash")

        # 初始化熔断器（每个源一个）
        self.circuit_breakers: dict[str, CircuitBreaker] = {}
        if enable_circuit_breaker:
            for name in self.sources:
                self.circuit_breakers[name] = CircuitBreaker(
                    failure_threshold=5, recovery_timeout=30.0
                )

        # 初始化监控
        self.metrics = MetricsCollector()

    def register_source(self, name: str, source: MemorySource):
        """注册新的记忆源。"""
        self.sources[name] = source
        if self.enable_circuit_breaker:
            self.circuit_breakers[name] = CircuitBreaker(
                failure_threshold=5, recovery_timeout=30.0
            )
        logger.info(f"注册记忆源: {name} ({source.source_type})")

    def _create_fusion_algorithm(self, algo: str,
                                  weights: dict[str, float] | None) -> FusionAlgorithm:
        if algo == "rrf":
            return RRFFusion(k=60)
        elif algo == "weighted":
            return WeightedFusion(weights or {})
        else:
            return RRFFusion(k=60)

    async def query(self, query_text: str, top_k: int = 10,
                    timeout_ms: float | None = None) -> list[ResultItem]:
        """
        统一查询接口 —— Agent 只需调用此方法。

        Args:
            query_text: 查询文本
            top_k: 返回结果数
            timeout_ms: 超时时间（毫秒）

        Returns:
            融合排序后的结果列表
        """
        self.metrics.record_query()
        query = Query(text=query_text, top_k=top_k)

        # 阶段 1: 查询路由
        plan = self.router.route(query)

        if not plan.sources:
            logger.warning("没有可查询的记忆源")
            return []

        logger.info(f"路由计划: {plan.sources}")

        # 阶段 2: 并行执行
        timeout = (timeout_ms or self.default_timeout_ms) / 1000.0
        results_by_source: dict[str, list[ResultItem]] = {}

        async def query_source(name: str) -> tuple[str, list[ResultItem], float]:
            source = self.sources[name]
            start = time.monotonic()

            async def execute():
                return await source.query(query)

            # 可选：熔断保护
            if self.enable_circuit_breaker and name in self.circuit_breakers:
                result = await self.circuit_breakers[name].call(
                    execute, fallback=[]
                )
            else:
                try:
                    result = await asyncio.wait_for(execute(), timeout=timeout)
                except asyncio.TimeoutError:
                    logger.warning(f"[{name}] 查询超时")
                    result = []

            elapsed = (time.monotonic() - start) * 1000
            return name, result, elapsed

        tasks = [query_source(name) for name in plan.sources]
        completed = await asyncio.gather(*tasks, return_exceptions=True)

        for result in completed:
            if isinstance(result, Exception):
                logger.error(f"查询执行异常: {result}")
                continue
            name, items, elapsed = result
            results_by_source[name] = items
            self.metrics.record_source_query(name, elapsed, success=len(items) > 0)

        # 阶段 3: 结果去重
        all_items: list[ResultItem] = []
        for items in results_by_source.values():
            all_items.extend(items)
        all_items = self.deduplicator.dedup(all_items)

        # 阶段 4: 结果融合
        fusion_start = time.monotonic()
        if results_by_source:
            # 按源分组传给融合算法
            fused = self.fusion_algorithm.fuse(
                results_by_source, top_k=top_k
            )
        else:
            fused = []

        fusion_time = (time.monotonic() - fusion_start) * 1000
        self.metrics.record_fusion(fusion_time)

        return fused[:top_k]

    async def stream_query(self, query_text: str, top_k: int = 10,
                           timeout_ms: float | None = None) -> AsyncIterator[ResultItem]:
        """
        流式查询 —— 结果分批到达。

        Agent 可以边接收边处理，不必等待所有源完成。
        """
        query = Query(text=query_text, top_k=top_k)
        plan = self.router.route(query)

        if not plan.sources:
            return

        timeout = (timeout_ms or self.default_timeout_ms) / 1000.0

        async def query_source(name: str) -> tuple[str, list[ResultItem]]:
            source = self.sources[name]
            try:
                items = await asyncio.wait_for(
                    source.query(query), timeout=timeout
                )
                return name, items
            except Exception as e:
                logger.warning(f"[{name}] 流式查询失败: {e}")
                return name, []

        tasks = {
            name: asyncio.create_task(query_source(name))
            for name in plan.sources
        }

        yielded_ids: set[str] = set()

        # as_completed 逐个 yield
        for coro in asyncio.as_completed(tasks.values()):
            name, items = await coro
            for item in items:
                if item.id not in yielded_ids:
                    yielded_ids.add(item.id)
                    yield item


# ═══════════════════════════════════════════
# 使用示例
# ═══════════════════════════════════════════

async def demo_basic_query():
    """基础查询演示。"""
    # 初始化记忆源
    sources = {
        "vector_db": VectorDBSource(),
        "sql_db": SQLDatabaseSource(),
        "knowledge_graph": KnowledgeGraphSource(),
        "document_store": DocumentStoreSource(),
        "external_api": ExternalAPISource(),
    }

    # 初始化联邦引擎
    engine = MemoryFederationEngine(
        sources=sources,
        fusion_algorithm="rrf",
        default_timeout_ms=3000.0,
        enable_circuit_breaker=True,
    )

    # 执行查询
    results = await engine.query(
        "Transformer 架构", top_k=5, timeout_ms=2000.0
    )

    print("=" * 60)
    print("记忆联邦查询结果")
    print("=" * 60)
    for i, item in enumerate(results, 1):
        print(f"\n{i}. [{item.source}] 分数={item.score:.4f}")
        print(f"   {item.content}")

    # 打印监控报告
    print("\n" + "=" * 60)
    print("监控报告")
    print("=" * 60)
    report = engine.metrics.get_report()
    for source, metrics in report["sources"].items():
        print(f"  {source}: 延迟={metrics['avg_latency_ms']}ms, "
              f"错误率={metrics['error_rate']}, "
              f"请求数={metrics['total_requests']}")


async def demo_streaming_query():
    """流式查询演示。"""
    engine = MemoryFederationEngine(
        sources={
            "vector_db": VectorDBSource(),
            "sql_db": SQLDatabaseSource(),
            "document_store": DocumentStoreSource(),
        },
    )

    print("流式查询结果（逐条到达）:")
    async for item in engine.stream_query("Transformer", top_k=5):
        print(f"  → [{item.source}] {item.content[:50]}...")


async def demo_fusion_algorithms():
    """融合算法对比演示。"""
    vector_results = [
        ResultItem("v1", "Transformer 使用自注意力", 0.95, "vector"),
        ResultItem("v2", "BERT 基于 Transformer", 0.90, "vector"),
        ResultItem("v3", "GPT 使用 Transformer", 0.85, "vector"),
    ]
    sql_results = [
        ResultItem("s1", "Attention Is All You Need (2017)", 1.0, "sql"),
        ResultItem("s2", "Vaswani et al., Google", 0.95, "sql"),
    ]
    kg_results = [
        ResultItem("v1", "Transformer 使用自注意力", 0.8, "graph"),
        ResultItem("kg1", "Transformer -- 提出者 --> Vaswani", 0.9, "graph"),
    ]

    sources = {
        "vector": vector_results,
        "sql": sql_results,
        "graph": kg_results,
    }

    # RRF 融合
    rrf = RRFFusion(k=60)
    rrf_results = rrf.fuse(sources, top_k=5)
    print("RRF 融合结果:")
    for r in rrf_results:
        print(f"  {r.id}: {r.score:.4f}")

    # 加权融合
    weighted = WeightedFusion(weights={"vector": 0.5, "sql": 0.3, "graph": 0.2})
    w_results = weighted.fuse(sources, top_k=5)
    print("\n加权融合结果:")
    for r in w_results:
        print(f"  {r.id}: {r.score:.4f}")


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)

    async def main():
        print("=== 基础查询演示 ===")
        await demo_basic_query()
        print("\n=== 流式查询演示 ===")
        await demo_streaming_query()
        print("\n=== 融合算法对比 ===")
        await demo_fusion_algorithms()

    asyncio.run(main())
```

### 预期输出

```text
=== 基础查询演示 ===
2026-07-16 12:00:00 INFO 查询类型: ('semantic',)
2026-07-16 12:00:00 INFO 路由计划: ['vector_db', 'document_store']
============================================================
记忆联邦查询结果
============================================================

1. [vector_db] 分数=0.0164
   Transformer 使用自注意力机制处理序列数据

2. [document_store] 分数=0.0156
   Transformer 架构由堆叠的编码器和解码器层组成...

3. [vector_db] 分数=0.0147
   BERT 是基于 Transformer 的预训练模型

============================================================
监控报告
============================================================
  vector_db: 延迟=50.0ms, 错误率=0, 请求数=1
  document_store: 延迟=100.0ms, 错误率=0, 请求数=1
```

## 场景判断：单一存储 vs 联邦架构决策矩阵

### 决策矩阵

```text
评估维度              权重   单存储得分   联邦得分   说明
────────────────────────────────────────────────────────────
记忆源数量             25%    0-100      0-100    见下
查询异构性             20%    0-100      0-100    见下
实时性要求             15%    0-100      0-100    见下
开发团队规模           10%    0-100      0-100    见下
容错要求               10%    0-100      0-100    见下
数据迁移成本           10%    0-100      0-100    见下
运维能力               10%    0-100      0-100    见下
```

**评分指南**：

```text
记忆源数量:
  - 1-2 个源 → 单存储 100 / 联邦 20
  - 3-5 个源 → 单存储 50  / 联邦 80
  - 5+ 个源  → 单存储 10  / 联邦 100

查询异构性:
  - 全是语义搜索 → 单存储 90 / 联邦 40
  - 语义 + 结构化 → 单存储 40 / 联邦 90
  - 语义 + 结构化 + 图 + 文档 → 单存储 10 / 联邦 100

实时性要求:
  - 所有源都是低延迟 (<50ms) → 单存储 80 / 联邦 60
  - 部分源高延迟 (>500ms) → 单存储 30 / 联邦 80

开发团队规模:
  - 1-2 人 → 单存储 90 / 联邦 40
  - 3-5 人 → 单存储 70 / 联邦 70
  - 5+ 人 → 单存储 30 / 联邦 90
```

### 决策流程图

```text
开始: 选择记忆架构方案
  │
  ├── 记忆源数量 ≤ 2?
  │     ├── 是 → 查询类型是否单一？
  │     │         ├── 是 → 选择单一存储 ✓
  │     │         └── 否 → 考虑联邦或 ETL 整合
  │     └── 否 → 继续
  │
  ├── 有高延迟源 (≥500ms)?
  │     ├── 是 → 需要渐进式渲染
  │     │         Agent 能处理流式结果？
  │     │           ├── 是 → 联邦 + 流式 ✓
  │     │           └── 否 → 考虑 ETL 整合
  │     └── 否 → 继续
  │
  ├── 团队规模 ≥ 3 人？
  │     ├── 是 → 联邦架构可维护性好 ✓
  │     └── 否 → 继续
  │
  ├── 需要容错（单源故障不瘫痪）？
  │     ├── 是 → 联邦架构 ✓
  │     └── 否 → 继续
  │
  └── 综合评分比较
        ├── 单存储得分 > 联邦得分 → 选择单一存储
        └── 联邦得分 > 单存储得分 → 选择联邦架构
```

### 典型场景推荐

```text
场景                                  推荐方案          理由
────────────────────────────────────────────────────────────
个人助手 Agent（1 个向量库）             单一存储          源少，查询单一
企业数据分析 Agent（SQL + 向量+知识图谱） 记忆联邦          多源异构查询
实时客服 Agent（向量库 + API）          混合（联邦+缓存）  需要低延迟+实时数据
研究型 Agent（5+ 源，容忍延迟）          记忆联邦          多源全覆盖
团队开发的复杂 Agent（3人+）            记忆联邦          并行开发需求
快速原型                                单存储+逐步迁移    先快后稳
```

### 什么时候应该避免联邦

```text
以下情形强烈不建议使用记忆联邦：

1. 延迟是唯一决胜因素
   - 实时在线推理，毫秒必争
   - 联邦层增加的开销不可接受

2. 只有 1 个记忆源
   - 联邦层完全是过度工程
   - 直接使用该源的原生接口更高效

3. 查询类型极其单一
   - 所有查询都是同一模式（如全部是语义搜索）
   - 单一存储足够满足需求

4. 团队没有运维分布式系统的能力
   - 联邦层需要监控、部署、排错的能力
   - 不如用简单的单存储方案

5. Agent 本身就是单一用途
   - 不需要处理不同类型的查询
   - 联邦的灵活性是多余的
```

## 总结

记忆联邦（Memory Federation）是 AI Agent 发展到多存储阶段的必然产物。它解决了 Agent 面对多个异构记忆源时的**统一访问**问题，让 Agent 可以集中精力在推理和决策上，而不是在"该查哪个数据库"和"怎么合并结果"上。

**核心价值**：

| 价值 | 一句话概括 |
|------|-----------|
| 统一接口 | Agent 只需一个 query() 调用，屏蔽所有底层差异 |
| 容错性 | 单源故障不会导致整个查询失败 |
| 最佳检索 | 每种记忆类型使用最合适的存储技术 |
| 可扩展 | 新存储源即插即用，不改 Agent 代码 |
| 渐进增强 | 可以从单一存储逐步演进到联邦架构 |

**核心代价**：

| 代价 | 严重程度 | 缓解方案 |
|------|---------|---------|
| 性能退化（最慢源拖累） | 高 | 超时、渐进渲染、投机执行 |
| 结果质量依赖融合算法 | 中 | 选择合适的融合算法 + 调参 |
| 架构复杂度增加 | 中 | 良好设计 + 充分文档化 |
| 源特异能力丢失 | 低 | 支持透传模式作为逃生舱 |

**最佳实践**：

1. **先用单一存储，必要时再联邦**——不要在一开始就引入联邦层的复杂度
2. **为每个源设置合理的超时**——不让一个慢源拖死所有查询
3. **选择适合场景的融合算法**——RRF 是通用好选择，加权融合在有权重信息时更好
4. **实现渐进式渲染**——让 Agent 可以边接收结果边处理
5. **全面监控**——每个源的延迟、错误率、贡献度都要可视
6. **缓存热门查询结果**——减少重复的跨源 fan-out
7. **保留透传接口**——需要源特异能力时可以跳过联邦层

最终，记忆联邦不是银弹——它是一种设计模式，在正确的场景下能极大简化 Agent 的记忆管理，但错误的场景下会引入不必要的复杂度。理解它的能力边界和适用条件，是做出正确架构决策的关键。

---

**上一篇**: [6.6.5 外部 API 记忆](../6.6.5%20external-api-memory/learn.md)
**下一篇**: 返回 [6.6 外部记忆概览](../learn.md)
**相关模块**: [6.6.1 知识图谱](../6.6.1%20knowledge-graph/learn.md), [6.6.3 数据库记忆](../6.6.3%20database-memory/learn.md), [6.6.4 文档存储](../6.6.4%20document-store/learn.md)
