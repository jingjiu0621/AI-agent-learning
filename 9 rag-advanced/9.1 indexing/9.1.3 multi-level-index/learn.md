# 9.1.3 multi-level-index — 多级索引：摘要 + 全文 + 向量

## 简单介绍

多级索引是为同一份知识内容构建多个不同粒度的索引层，使检索系统能够根据查询的复杂度和精度需求，在不同层级之间灵活切换。典型的金字塔结构是：顶层摘要索引提供快速初筛 → 中层全文索引提供语义匹配 → 底层详细索引提供精确提取。多级索引解决了"单一索引粒度无法同时满足精度和召回率"的根本矛盾。

## 基本原理

### 三级金字塔架构

```
检索顺序
     │
     ▼   Level 1: 摘要索引（粗粒度）
     │   ┌─────────────────────────────────────┐
     │   │ 每篇文档/章节 → 1个摘要（~200 tokens） │
     │   │ 目的：快速定位相关文档，排除无关文档    │
     │   │ 存储：向量（摘要 embedding）           │
     │   └─────────────────────────────────────┘
     │                     │ 命中
     ▼   Level 2: 全文索引（中粒度）
     │   ┌─────────────────────────────────────┐
     │   │ 每个分块 → 向量 + BM25              │
     │   │ 目的：精确匹配相关段落               │
     │   │ 存储：向量 + 倒排索引               │
     │   └─────────────────────────────────────┘
     │                     │ 需要精确信息
     ▼   Level 3: 详细索引（细粒度）
     │   ┌─────────────────────────────────────┐
     │   │ 关键句/实体/数值 → 精确位置          │
     │   │ 目的：直接提取答案，无需全文重建      │
     │   │ 存储：结构化键值对                   │
     │   └─────────────────────────────────────┘
```

### 多级检索流程

```python
class MultiLevelRetriever:
    """三级索引检索器"""
    
    def __init__(self, summary_index, chunk_index, detail_index):
        self.summary_index = summary_index   # Level 1: 摘要索引
        self.chunk_index = chunk_index       # Level 2: 分块索引
        self.detail_index = detail_index     # Level 3: 细节索引
    
    async def retrieve(self, query: str, top_k: int = 5):
        """自顶向下三级检索"""
        
        # Stage 1: 摘要级初筛 —— 快速排除无关文档
        summary_results = await self.summary_index.search(query, top_k=top_k)
        candidate_docs = set(r["doc_id"] for r in summary_results)
        
        if not candidate_docs:
            return []  # 无相关文档，提前退出
        
        # Stage 2: 分块级精排 —— 在候选文档中精确匹配
        chunk_results = await self.chunk_index.search(
            query, 
            filter={"doc_id": candidate_docs},
            top_k=top_k
        )
        
        # Stage 3: 细节级提取 —— 从最优段落中提取精确信息
        detailed = []
        for chunk in chunk_results:
            details = await self.detail_index.extract(
                chunk["doc_id"], 
                chunk["chunk_index"],
                query=query
            )
            detailed.append({
                "chunk": chunk,
                "details": details,
                "relevance_score": chunk["score"]
            })
        
        return detailed
```

## 背景与演进

- **单级向量索引（2022-2023）**：早期 RAG 仅有一层向量索引——所有文档切成相同大小的块，全量做 embedding。问题是：无关文档的块占据了检索候选集的 Top-K，浪费了上下文预算，且检索精度受限于单一块大小。

- **摘要 + 全文双层（2023 中）**：LlamaIndex 引入 `SummaryIndex` 和 `RecursiveRetriever` 模式，先检索文档摘要定位相关文档，再在文档内搜索具体块。这显著提升了大型文档库的检索效率。

- **RAPTOR（2024）**：斯坦福的 RAPTOR 论文将多级索引推向新高度——自动将文档聚类为不同抽象层次的摘要（底层 → 中层 → 顶层），检索时自底向上递归搜索，实现了"从具体事实到高层概述"的全覆盖。

- **当前趋势**：多级索引 + Agentic Routing——让 Agent 根据查询类型自动选择索引层（事实性查询→底层，概览性查询→顶层）。同时多级索引与 Graph RAG 融合，形成"摘要图 + 内容图 + 实体图"的复合结构。

## 核心矛盾

**检索深度 vs 检索代价**：

```
单级索引         简单的 top-k 搜索 + 二次重排
                ║ 简单快速，但可能漏掉关键信息
                ║ 且缺乏检索可解释性
                
多级索引          由粗到精逐步缩小候选集
                ║ 检索精度更高，但延迟增加
                ║ 需要维护多个索引层
                ║ 且层级间的协调复杂
```

详细权衡：

| 方面 | 单级索引 | 双层索引 | 三级索引 |
|------|---------|---------|---------|
| 检索精度 | 基线 | +20-30% | +30-50% |
| 延迟（搜索） | 50ms | 80-120ms | 120-200ms |
| 存储开销 | 1x | 1.3-1.5x | 1.5-2x |
| 实现复杂度 | 低 | 中 | 高 |
| 大型库（>1M docs） | ⭐⭐ 性能显著下降 | ⭐⭐⭐⭐ 好 | ⭐⭐⭐⭐⭐ 最好 |
| 小型库（<10K docs） | ⭐⭐⭐⭐⭐ 够用 | ⭐⭐⭐ 过度设计 | ⭐⭐ 过度设计 |

## 主流优化方向

### 1. 摘要索引构建优化

摘要索引的质量取决于摘要本身的质量，而非块大小。

```python
async def build_summary_index(docs: List[Document], llm, embed_model):
    """用 LLM 生成文档摘要并索引"""
    summaries = []
    
    for doc in docs:
        # Step 1: LLM 生成高质量摘要
        summary = await llm.generate(
            f"用 2-3 句话概括以下文档的核心内容：\n\n{doc.text[:3000]}"
        )
        
        # Step 2: 摘要 embedding + 保持 doc_id 关联
        embedding = await embed_model.embed(summary)
        summaries.append({
            "doc_id": doc.id,
            "summary": summary,
            "embedding": embedding,
            "title": doc.title,
            "token_count": len(embedding),
            "chunk_count": len(doc.chunks)  # 关联下层块数量
        })
    
    return SummaryIndex(summaries)
```

### 2. RAPTOR 式多级摘要

RAPTOR（Recursive Abstractive Processing for Tree-Organized Retrieval）自动构建层次化摘要树。

```python
# RAPTOR 核心：文本聚类 → 摘要生成 → 递归构建
class RAPTORIndex:
    def __init__(self, embed_model, llm, max_levels=3):
        self.embed_model = embed_model
        self.llm = llm
        self.max_levels = max_levels
        self.tree = {}  # level -> list of nodes
    
    async def build(self, chunks: List[str]):
        """从底层块开始递归构建摘要树"""
        current_level = [
            {"text": c, "embedding": await self.embed_model.embed(c)}
            for c in chunks
        ]
        self.tree[0] = current_level
        
        for level in range(1, self.max_levels):
            if len(current_level) <= 1:
                break
            
            # 聚类：将语义相似的块分组
            clusters = self._cluster(current_level, k=min(5, len(current_level)//2))
            
            # 每组生成父级摘要
            next_level = []
            for cluster in clusters:
                combined = "\n\n".join(n["text"] for n in cluster)
                summary = await self.llm.generate(
                    f"将以下内容总结为一段话（保留关键信息）：\n{combined}"
                )
                next_level.append({
                    "text": summary,
                    "embedding": await self.embed_model.embed(summary),
                    "children": [n["text"] for n in cluster]  # 指向子节点
                })
            
            self.tree[level] = next_level
            current_level = next_level
    
    async def search(self, query: str, top_k: int = 5):
        """自顶向下搜索：从顶层开始，选择最优分支深入"""
        query_emb = await self.embed_model.embed(query)
        
        # 从顶层开始搜索
        for level in range(self.max_levels - 1, -1, -1):
            nodes = self.tree.get(level, [])
            if not nodes:
                continue
            
            # 当前层级找最优
            scored = [(n, cosine_sim(query_emb, n["embedding"])) for n in nodes]
            scored.sort(key=lambda x: -x[1])
            
            best = scored[0][0]
            if "children" in best and level > 0:
                # 深入子节点
                text = best["children"]
            else:
                return best["text"]
        
        return None
```

### 3. 三级索引的自适应路由

```python
async def adaptive_retrieve(query: str, multi_index, query_classifier):
    """根据查询类型自动选择索引层级"""
    
    # Step 1: 分类查询类型
    qtype = await query_classifier.classify(query)
    
    if qtype == "factoid":
        # 事实性问题（"XX 公司的 CEO 是谁？"）→ 直接搜索细节索引
        return await multi_index.detail_search(query)
    
    elif qtype == "summary":
        # 概述性问题（"XX 报告的要点是什么？"）→ 搜索摘要索引
        return await multi_index.summary_search(query)
    
    elif qtype == "analysis":
        # 分析性问题（"X 措施对 Y 的影响？"）→ 全文搜索 + 跨文档综合
        return await multi_index.hybrid_search(query, levels=[1, 2])
    
    else:
        # 默认：三级全链路
        return await multi_index.full_search(query)
```

## 能力边界与结果边界

| 维度 | 单级索引 | 双层索引 | 三级索引 | RAPTOR |
|------|---------|---------|---------|--------|
| 大型知识库精度 | ⭐⭐ 检索噪声大 | ⭐⭐⭐⭐ 摘要过滤有效 | ⭐⭐⭐⭐⭐ 最高 | ⭐⭐⭐⭐⭐ 同时覆盖全局 |
| 小型知识库 | ⭐⭐⭐⭐ 够用 | ⭐⭐⭐ 优势不明显 | ⭐⭐ 过度 | ⭐⭐ 过度 |
| 跨文档推理 | ⭐⭐ 难 | ⭐⭐⭐ 可跨文档摘要 | ⭐⭐⭐⭐ 可跨文档细节 | ⭐⭐⭐⭐⭐ 天然支持 |
| 查询延迟 | ⭐⭐⭐⭐⭐ 低 | ⭐⭐⭐⭐ 中 | ⭐⭐⭐ 较高 | ⭐⭐ 构建成本高 |
| 存储成本 | ⭐⭐⭐⭐⭐ 低 | ⭐⭐⭐⭐ 中 | ⭐⭐⭐ 较高 | ⭐⭐ 高 |
| 可解释性 | ⭐⭐ "黑盒" | ⭐⭐⭐ 可示踪到文档 | ⭐⭐⭐⭐ 可示踪到段落 | ⭐⭐⭐ 树路径可追踪 |

## 与其他索引策略的区别

- **多级索引 vs 层次索引**：多级构建同一份内容的多个粒度版本（摘要+全文+细节）；层次索引关注文档内部的结构关系（章节→段落→句子）
- **多级索引 vs 多路检索**：多级是在一个维度上的深度递进；多路是多个不同维度的平行探索（向量+BM25+知识图谱）
- **多级索引 vs 分块策略**：分块决定单级索引的最小单元粒度；多级是在分块之上构建更高层的组织关系
- **RAPTOR vs 传统多级**：RAPTOR 自动聚类分层，不依赖人工预定义的"摘要-分块-细节"结构，更灵活但控制力弱

## 核心优势

多级索引的核心优势在于**"先排除再精搜"**的漏斗策略能显著降低无关内容的检索干扰。当文档库超过 10 万篇时，单级 top-k 检索的"长尾噪声"问题非常严重——即使最相关的文档在 Top-10 内，第 6-10 位的噪声文档也会浪费上下文预算。多级索引的摘要层过滤能高效剪枝无关文档，使后续的精确检索在更干净的候选集上进行。

## 工程优化

1. **摘要层级延迟预算**：每级搜索分配严格的延迟预算（Level 1 < 30ms，Level 2 < 50ms，Level 3 < 100ms），任一阶段超时则跳过
2. **预计算摘要**：文档入库时用 LLM 生成摘要并缓存，避免检索时实时生成
3. **层级间缓存共享**：如果用户在同一会话中连续提问，缓存 Level 1 的摘要搜索结果避免重复计算
4. **动态路由权重**：根据查询中的词汇量、具体性来自动调整各索引层的权重比例
5. **冷启动策略**：新知识库先用单级索引运行，日访问量超过阈值后再构建多级索引

## 场景适配指南

| 场景 | 推荐层级 | 理由 |
|------|---------|------|
| 百万级文档知识库 | 三级（摘要 + 全文 + 细节） | 需要高效排除无关文档 |
| 企业 FAQ 系统 | 单级或双层 | 文档量少，查询固定 |
| 学术论文检索 | 双层（摘要 + 全文） | 摘要质量高，足以初筛 |
| 实时聊天知识库 | 双级（摘要 + 全文） | 延迟敏感，不能用三级 |
| 法律/医学文档 | 三级 + 细节实体索引 | 精度要求极高 |
| 全量文档分析 | RAPTOR | 需要跨文档综合 |
