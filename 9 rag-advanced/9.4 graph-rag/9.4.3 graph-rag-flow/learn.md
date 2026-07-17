# 9.4.3 Graph RAG Flow — 检索 → 子图 → 推理

> **核心问题**：如何利用知识图谱的结构化信息，增强 LLM 在多跳推理和复杂问答场景下的能力？

---

## 1. 背景与问题

### 1.1 Vector RAG 的困境：多跳推理

传统 Vector RAG（向量检索 + LLM 生成）在处理**多跳推理** (Multi-hop Reasoning) 时表现不佳。原因在于向量检索本质上是一个**语义相似度匹配**过程，而非**关系推理**过程。

```
多跳推理示例:

问题: "图灵奖得主的博士导师是谁？"

需要推理路径:
  实体1: ??? (某位图灵奖得主)
    关系1: 获得 → 图灵奖
  实体2: ??? (博士导师)
    关系2: 博士导师

实际的推理:
  "Yoshua Bengio" ──获得──→ "图灵奖(2018)"
       │
       ├──博士导师──→ "Geoffrey Hinton"
       │
       └──学生──→ "Ian Goodfellow"

Vector RAG 的行为:
  Query → [向量化] → [相似度匹配 top-k 文档块]
  → 文档块可能包含 "Bengio 获得了图灵奖"
  → 文档块可能包含 "Geoffrey Hinton 是 Bengio 的博士后导师"
  → LLM 需要从两块文本中推理出完整路径

问题: 如果两个事实不在同一文档块中，且语义相似度排序后，
     其中一个块被截断，LLM 就无法完成推理。
```

### 1.2 Graph RAG 的核心差异

```
Vector RAG:                    Graph RAG:
┌──────────────┐               ┌──────────────┐
│  用户查询      │               │  用户查询      │
└──────┬───────┘               └──────┬───────┘
       │                              │
       ▼                              ▼
┌──────────────┐               ┌──────────────┐
│  向量检索     │               │  实体识别      │
│  (语义相似度) │               │  (Query→Entity)│
└──────┬───────┘               └──────┬───────┘
       │                              │
       ▼                              ▼
┌──────────────┐               ┌──────────────┐
│  Top-K 块    │               │  子图检索      │
│  (文档片段)   │               │  (知识图谱)    │
└──────┬───────┘               └──────┬───────┘
       │                              │
       ▼                              ▼
┌──────────────┐               ┌──────────────┐
│  LLM 生成    │               │  子图序列化    │
│  (基于文档块) │               │  (结构化→文本) │
└──────────────┘               └──────┬───────┘
                                     │
                                     ▼
                              ┌──────────────┐
                              │  LLM 推理+    │
                              │  文档上下文融合│
                              └──────────────┘

关键区别:
  Vector RAG: 检索"相关文档"→ LLM 在文档中找答案
  Graph RAG:  检索"相关实体和关系"→ LLM 在图上推理路径
```

### 1.3 两者互补性

| 维度 | Vector RAG | Graph RAG |
|------|-----------|-----------|
| **多跳推理** | 弱（语义匹配无法跨越关系跳转） | 强（图结构天然支持关系遍历） |
| **信息广度** | 强（覆盖所有文档内容） | 弱（只覆盖被抽取的结构化知识） |
| **精确事实** | 中（依赖文档中是否明确出现） | 强（三元组精确表达事实） |
| **开放域问答** | 强（不限制知识范围） | 弱（受限于图谱覆盖度） |
| **更新成本** | 低（直接索引新文档） | 高（需要重新构建三元组） |
| **实现复杂度** | 低 | 高 |

**核心洞察**：Graph RAG 不是替代 Vector RAG，而是在多跳推理场景下补足 Vector RAG 的短板。最佳实践是两者融合。

---

## 2. Graph RAG 三阶段流程

Graph RAG 的完整流程可分为三个阶段：实体映射 → 子图检索 → 子图推理。

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Graph RAG 三阶段流程                             │
│                                                                     │
│  阶段1: Query → Entity Mapping         阶段2: Subgraph Retrieval     │
│  ┌─────────────────────────┐           ┌────────────────────────┐   │
│  │ "图灵奖得主的博士导师     │           │  Seed Entities          │   │
│  │  是谁？"                 │           │  {Bengio, 图灵奖}       │   │
│  └────────────┬────────────┘           └───────────┬────────────┘   │
│               │                                    │                │
│               ▼                                    ▼                │
│  ┌─────────────────────────┐           ┌────────────────────────┐   │
│  │  实体识别 + 链接         │           │  BFS/DFS k-hop 扩展    │   │
│  │  NER → Entity Linking   │           │  ↓                     │   │
│  │  ↓                      │           │  (Bengio)——获得——(图灵奖) │   │
│  │  Seed Entities:         │           │    │                    │   │
│  │  {Bengio, 图灵奖}       │           │  (Hinton)              │   │
│  └─────────────────────────┘           └───────────┬────────────┘   │
│                                                    │                │
│  阶段3: Subgraph → Reasoning                       ▼                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  子图序列化:                                                │    │
│  │  "(Bengio, WON, Turing Award); (Bengio, ADVISED_BY, Hinton)"│    │
│  │                                                            │    │
│  │  LLM 推理:                                                 │    │
│  │  "The Turing Award winner Bengio was advised by Hinton,    │    │
│  │   so the answer is Geoffrey Hinton."                       │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.1 阶段1: 查询到实体映射 (Query → Entity)

#### 2.1.1 问题定义

自然语言查询如何准确映射到图谱中的实体节点？这是 Graph RAG 的第一步，也是最关键的一步——如果实体映射错了，后续所有检索和推理都会错。

#### 2.1.2 三种实现方案

**方案A: NER + Entity Linking（传统方式）**

```python
"""
方案A: NER 识别问题中的命名实体 → Entity Linking 映射到 KG 节点
"""

def query_to_entities_ner(
    query: str,
    kg_index: dict,  # {"entity_name": {"entity_id": str, "type": str, "aliases": list}}
    llm_client
) -> list:
    """
    使用 NER 从查询中识别实体，链接到知识图谱节点。

    Args:
        query: 用户问题
        kg_index: 知识图谱的实体名索引
        llm_client: LLM API 客户端

    Returns:
        匹配到的图谱实体 ID 列表
    """
    # 第一步: 用 NER 从问题中提取实体名
    entities = extract_entities_with_llm(
        query,
        entity_types=["PERSON", "ORG", "LOC", "PRODUCT", "EVENT"],
        llm_client=llm_client
    )

    # 第二步: 链接到图谱节点
    matched_ids = []
    for ent in entities:
        entity_name = ent["text"]

        # 精确匹配
        if entity_name in kg_index:
            matched_ids.append(kg_index[entity_name]["entity_id"])
            continue

        # 别名匹配
        for indexed_name, info in kg_index.items():
            if entity_name.lower() in info.get("aliases", []):
                matched_ids.append(info["entity_id"])
                break

        # 模糊匹配（使用字符串相似度）
        best_match, best_score = None, 0
        for indexed_name in kg_index:
            score = _string_similarity(entity_name.lower(), indexed_name.lower())
            if score > best_score and score > 0.8:
                best_score = score
                best_match = kg_index[indexed_name]["entity_id"]

        if best_match:
            matched_ids.append(best_match)

    return matched_ids


def _string_similarity(s1: str, s2: str) -> float:
    """简化的字符串相似度（基于字符交集）"""
    set1, set2 = set(s1), set(s2)
    intersection = set1 & set2
    union = set1 | set2
    return len(intersection) / len(union) if union else 0
```

**方案B: LLM 直接识别（推荐方式）**

```python
"""
方案B: 让 LLM 直接识别查询涉及的实体，并映射到图谱。

优势:
  1. 不需要独立 NER 管线
  2. 可以利用上下文消除歧义
  3. 可以处理复杂查询中隐含的实体
"""

QUERY_ENTITY_MAPPING_PROMPT = """你是知识图谱查询分析专家。
分析以下用户问题，找出需要在知识图谱中查询的实体。

知识图谱中的可用实体类型:
{entity_types}

请输出一个 JSON 数组，每个元素包含:
- "query_text": 查询中提到的文本
- "matched_entity": 图谱中最可能匹配的实体名
- "entity_type": 实体类型
- "role_in_query": 该实体在查询中的作用描述

如果查询中隐含了未明确写出的实体（例如"图灵奖得主"暗示某位获奖者），
也需要识别出来并标记为 "inferred": true。

问题: {query}

输出 JSON:
"""


def query_to_entities_llm(
    query: str,
    entity_types: dict,
    llm_client,
    kg_entity_names: list
) -> list:
    """
    让 LLM 分析查询，提取需要查询的实体。
    """
    prompt = QUERY_ENTITY_MAPPING_PROMPT.format(
        entity_types=entity_types,
        query=query
    )

    response = llm_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0,
        response_format={"type": "json_object"}
    )

    import json
    result = json.loads(response.choices[0].message.content)

    # 将 LLM 输出的实体名匹配到图谱中的实际实体
    matched = []
    for item in result:
        best_match = _fuzzy_match(item["matched_entity"], kg_entity_names)
        if best_match:
            matched.append({
                "query_mention": item["query_text"],
                "kg_entity": best_match,
                "type": item["entity_type"],
                "inferred": item.get("inferred", False)
            })

    return matched


def _fuzzy_match(name: str, candidates: list, threshold: float = 0.7) -> str:
    """模糊匹配实体名"""
    from difflib import SequenceMatcher

    best_match = None
    best_ratio = 0

    for candidate in candidates:
        ratio = SequenceMatcher(None, name.lower(), candidate.lower()).ratio()
        if ratio > best_ratio:
            best_ratio = ratio
            best_match = candidate

    return best_match if best_ratio >= threshold else None
```

**方案C: 向量相似度匹配实体名**

```python
"""
方案C: 将实体名向量化，用语义相似度匹配。

适合场景: 查询文本与图谱实体名存在语义差异
  例如: 查询中提到"深度学习之父" → 匹配到实体"Geoffrey Hinton"
"""

def query_to_entities_vector(
    query: str,
    entity_embeddings: dict,  # {"entity_name": np.array}
    embedding_model,
    top_k: int = 5
) -> list:
    """
    使用向量相似度匹配查询到实体名。

    Args:
        query: 用户查询
        entity_embeddings: 实体名→向量的映射
        embedding_model: 文本嵌入模型
        top_k: 返回前 K 个匹配实体

    Returns:
        [(entity_name, similarity_score), ...]
    """
    query_embedding = embedding_model.encode(query)

    scores = []
    for entity_name, entity_emb in entity_embeddings.items():
        similarity = cosine_similarity(query_embedding, entity_emb)
        scores.append((entity_name, similarity))

    scores.sort(key=lambda x: x[1], reverse=True)
    return scores[:top_k]


def cosine_similarity(a, b) -> float:
    """余弦相似度"""
    import numpy as np
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
```

#### 2.1.3 三种方案对比

| 维度 | NER + Entity Linking | LLM 直接识别 | 向量相似度匹配 |
|------|---------------------|-------------|---------------|
| **精度** | 高（精确匹配时） | 中高 | 中（依赖嵌入质量） |
| **召回** | 中（别名问题） | 高（可理解上下文） | 高（语义匹配） |
| **已知实体覆盖** | 依赖别名表 | 灵活 | 依赖向量质量 |
| **隐式实体推断** | 否 | 是 | 否 |
| **延迟** | 低（+1次LLM调用） | 中等（1次LLM调用） | 极低（计算相似度） |
| **推荐场景** | 实体名精确 | 复杂查询 | 简短的实体查询 |

### 2.2 阶段2: 子图检索与扩展 (Subgraph Retrieval)

获取种子实体后，需要从图谱中检索出与问题相关的子图。核心问题是：**应该扩展多远？**

#### 2.2.1 固定跳数扩展 (k-hop Neighbor Expansion)

```
1-hop 扩展:

       ┌───────────────────────┐
       │   (Bengio)            │  ← 种子实体
       │    │                  │
       │ (获得)  (博士导师)    │  ← 1-hop 关系
       │    │       │          │
       │   ▼       ▼           │
       │ (图灵奖) (Hinton)     │  ← 1-hop 实体
       └───────────────────────┘

2-hop 扩展:

       ┌───────────────────────────────┐
       │   (Bengio)                    │
       │    │         │                │
       │ (获得)    (博士导师)           │
       │    │         │                │
       │   ▼         ▼                │
       │ (图灵奖)   (Hinton)           │
       │               │              │
       │         (学生) │              │  ← 2-hop 关系
       │               │              │
       │               ▼              │
       │           (Goodfellow)       │  ← 2-hop 实体
       └───────────────────────────────┘
```

```python
def k_hop_expansion(
    graph: KnowledgeGraph,
    seed_entity_ids: list,
    k: int = 2,
    relation_filter: list = None
) -> dict:
    """
    k-hop 邻居扩展。

    Args:
        graph: 知识图谱
        seed_entity_ids: 种子实体 ID 列表
        k: 扩展跳数
        relation_filter: 只考虑指定关系类型（None=全部）

    Returns:
        {
            "nodes": set[entity_id],
            "edges": list[(head_id, relation, tail_id)],
            "entity_names": {entity_id: name}
        }
    """
    visited_nodes = set(seed_entity_ids)
    current_frontier = set(seed_entity_ids)
    collected_edges = []
    entity_names = {}

    # 为每个种子实体获取名称
    for eid in seed_entity_ids:
        entity_names[eid] = graph.entities[eid]["name"]

    for hop in range(k):
        next_frontier = set()
        for node_id in current_frontier:
            # 查找从 node_id 出发的所有边
            for rel in graph.relations:
                if rel["head_id"] == node_id:
                    if relation_filter and rel["relation"] not in relation_filter:
                        continue
                    if rel["tail_id"] not in visited_nodes:
                        next_frontier.add(rel["tail_id"])
                        entity_names[rel["tail_id"]] = graph.entities[rel["tail_id"]]["name"]
                    collected_edges.append((node_id, rel["relation"], rel["tail_id"]))

                # 反向边（如果图谱是无向的或有反向关系）
                if rel["tail_id"] == node_id:
                    if relation_filter and rel["relation"] not in relation_filter:
                        continue
                    if rel["head_id"] not in visited_nodes:
                        next_frontier.add(rel["head_id"])
                        entity_names[rel["head_id"]] = graph.entities[rel["head_id"]]["name"]
                    collected_edges.append((node_id, f"INV_{rel['relation']}", rel["head_id"]))

        visited_nodes.update(next_frontier)
        current_frontier = next_frontier

    return {
        "nodes": visited_nodes,
        "edges": collected_edges,
        "entity_names": entity_names
    }
```

#### 2.2.2 基于 PageRank 的节点排序

对于扩展出的节点，并非全部与问题相关。需要排序和剪枝。

```
Personalized PageRank for Graph RAG:

  原理: 以查询实体作为 PageRank 的"个性化源"，
  计算图中各节点相对于查询的相关性得分。

  输入: 种子实体 (Bengio, 图灵奖) + 扩展的 k-hop 子图
  输出: 每个节点的相关性得分

          (Bengio) ────────── (图灵奖)
            │   score: 1.0      score: 0.9
            │
            │
          (Hinton)
       score: 0.75
            │
            │
       (Goodfellow)     (Minsky)
       score: 0.4        score: 0.05 ← 不相关，被剪枝
```

```python
def personalized_pagerank(
    graph: KnowledgeGraph,
    seed_entity_ids: list,
    damping: float = 0.85,
    max_iter: int = 100,
    tol: float = 1e-6
) -> dict:
    """
    Personalized PageRank 计算节点相关性。

    与标准 PageRank 的区别：
    标准 PageRank 的 teleport 分布是均匀的；
    个性化 PageRank 的 teleport 分布集中在种子节点。

    Args:
        graph: 知识图谱
        seed_entity_ids: 种子实体 ID（查询涉及的实体）
        damping: 阻尼因子
        max_iter: 最大迭代次数
        tol: 收敛容差

    Returns:
        {entity_id: score}
    """
    all_entity_ids = list(graph.entities.keys())
    n = len(all_entity_ids)
    idx_map = {eid: i for i, eid in enumerate(all_entity_ids)}

    # 构建邻接矩阵（出边）
    out_degree = {eid: 0 for eid in all_entity_ids}
    adj = {eid: [] for eid in all_entity_ids}

    for rel in graph.relations:
        head, tail = rel["head_id"], rel["tail_id"]
        adj[head].append(tail)
        out_degree[head] = out_degree.get(head, 0) + 1
        adj[tail].append(head)  # 无向图模式
        out_degree[tail] = out_degree.get(tail, 0) + 1

    # 初始化得分向量
    scores = {eid: 1.0 / n for eid in all_entity_ids}

    # 个性化 teleport 向量：种子节点概率更高
    teleport = {eid: 0.0 for eid in all_entity_ids}
    seed_mass = 1.0 / len(seed_entity_ids)
    for eid in seed_entity_ids:
        teleport[eid] = seed_mass

    # 迭代计算
    for _ in range(max_iter):
        new_scores = {eid: 0.0 for eid in all_entity_ids}
        max_change = 0.0

        for eid in all_entity_ids:
            # teleport 部分
            new_scores[eid] += (1 - damping) * teleport[eid]

            # 随机游走部分
            for neighbor in adj[eid]:
                if out_degree[neighbor] > 0:
                    new_scores[eid] += damping * scores[neighbor] / out_degree[neighbor]

        # 检查收敛
        for eid in all_entity_ids:
            change = abs(new_scores[eid] - scores[eid])
            max_change = max(max_change, change)

        scores = new_scores
        if max_change < tol:
            break

    return scores


def rank_and_filter_nodes(
    pp_scores: dict,
    top_k: int = 20,
    min_score: float = 0.01
) -> list:
    """
    根据 PageRank 得分排序和过滤节点。

    Returns:
        排序后的 [(entity_id, score), ...]
    """
    sorted_nodes = sorted(pp_scores.items(), key=lambda x: x[1], reverse=True)
    return [(eid, score) for eid, score in sorted_nodes
            if score >= min_score][:top_k]
```

#### 2.2.3 基于关系和属性的过滤

不是所有关系类型都对回答问题有价值。在检索子图时，可以根据问题类型过滤关系。

```python
def relation_filter_for_query(query: str, llm_client) -> list:
    """
    根据查询类型，推断需要关注的关系类型。

    例如:
    "谁创立了苹果公司?" → ["FOUNDED_BY", "CEO_OF"]
    "苹果公司的总部在哪?" → ["HEADQUARTERED_IN", "LOCATED_IN"]
    "BERT 模型基于什么架构?" → ["BASED_ON", "ARCHITECTURE"]
    """
    prompt = f"""分析以下问题，判断需要在知识图谱中查询哪些关系类型。

问题: {query}

输出 JSON 格式的关系类型列表，使用英文:
["CEO_OF", "LOCATED_IN", "FOUNDED_BY", ...]

只输出 JSON 数组:
"""

    response = llm_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0,
        response_format={"type": "json_object"}
    )

    import json
    return json.loads(response.choices[0].message.content)
```

#### 2.2.4 路径查询 vs 子图查询

```
路径查询:
  目标: 找到两个实体之间的特定路径
  方法: 最短路径 / 受限路径搜索
  输出: "Bengio -[ADVISED_BY]-> Hinton"
  适用: 明确源和目标的查询（"A 和 B 有什么关系?"）

子图查询:
  目标: 获取一个实体周围的完整上下文
  方法: k-hop 扩展 + 节点排序
  输出: 以种子实体为中心的子图
  适用: 开放式查询（"介绍一下 Bengio 的研究背景"）
```

```python
def shortest_path_query(
    graph: KnowledgeGraph,
    source_entity_id: str,
    target_entity_id: str,
    max_depth: int = 6
) -> list:
    """
    BFS 最短路径查询。

    适用场景：明确知道两个实体，但不知道它们之间的具体关系。
    "Geoffrey Hinton 和 Yoshua Bengio 有什么关系？"
    """
    from collections import deque

    # BFS
    queue = deque([(source_entity_id, [source_entity_id], [])])
    visited = {source_entity_id}

    while queue:
        current, path, relations = queue.popleft()
        depth = len(path) - 1

        if depth > max_depth:
            continue

        # 遍历出边
        for rel in graph.relations:
            if rel["head_id"] == current:
                next_node = rel["tail_id"]
                if next_node == target_entity_id:
                    return path + [next_node], relations + [rel["relation"]]
                if next_node not in visited and depth < max_depth:
                    visited.add(next_node)
                    queue.append((next_node, path + [next_node], relations + [rel["relation"]]))

            # 反向边
            if rel["tail_id"] == current:
                next_node = rel["head_id"]
                if next_node == target_entity_id:
                    return path + [next_node], relations + [f"INV_{rel['relation']}"]
                if next_node not in visited and depth < max_depth:
                    visited.add(next_node)
                    queue.append((next_node, path + [next_node], relations + [f"INV_{rel['relation']}"]))

    return [], []


def subgraph_query(
    graph: KnowledgeGraph,
    seed_entity_ids: list,
    k: int = 2,
    top_k_nodes: int = 30
) -> dict:
    """
    子图查询：获取种子实体周围的局部子图。
    适用场景：开放域查询。
    """
    # 1. k-hop 扩展
    raw_subgraph = k_hop_expansion(graph, seed_entity_ids, k=k)

    # 2. PageRank 排序
    scores = personalized_pagerank(graph, seed_entity_ids)

    # 3. 剪枝：只保留 top-k 节点
    top_nodes = rank_and_filter_nodes(scores, top_k=top_k_nodes)
    top_node_ids = set(eid for eid, _ in top_nodes)

    filtered_edges = [
        (h, r, t) for h, r, t in raw_subgraph["edges"]
        if h in top_node_ids and t in top_node_ids
    ]

    return {
        "nodes": top_node_ids,
        "edges": filtered_edges,
        "scores": {eid: score for eid, score in top_nodes},
        "entity_names": raw_subgraph["entity_names"]
    }
```

### 2.3 阶段3: 子图到推理 (Subgraph → LLM Reasoning)

检索到子图后，最关键的步骤是将**图结构转化为 LLM 可以理解的形式**。

#### 2.3.1 子图序列化策略

**策略1: 三元组列表**——最直接的方式

```python
def serialize_as_triples(subgraph: dict) -> str:
    """
    将子图序列化为三元组列表文本。
    """
    lines = ["以下是从知识图谱中检索到的相关事实:\n"]
    for head, relation, tail in subgraph["edges"]:
        head_name = subgraph["entity_names"].get(head, head)
        tail_name = subgraph["entity_names"].get(tail, tail)
        lines.append(f"- ({head_name}, {relation}, {tail_name})")

    return "\n".join(lines)

# 输出示例:
# 以下是从知识图谱中检索到的相关事实:
# - (Yoshua Bengio, WON, Turing Award 2018)
# - (Yoshua Bengio, ADVISED_BY, Geoffrey Hinton)
# - (Geoffrey Hinton, WON, Turing Award 2018)
# - (Yoshua Bengio, STUDENT_OF, Ian Goodfellow)
```

**策略2: 路径描述**——更适合线性推理

```python
def serialize_as_paths(subgraph: dict, seed_entity_ids: list) -> str:
    """
    将子图序列化为以种子实体为中心的路径描述。
    """
    lines = [f"以查询实体为中心的关联路径:\n"]

    for seed_id in seed_entity_ids:
        seed_name = subgraph["entity_names"].get(seed_id, seed_id)
        paths = _extract_paths(subgraph, seed_id, max_depth=3)

        for path in paths:
            path_str = f"  {seed_name}"
            for i, (rel, node) in enumerate(path):
                node_name = subgraph["entity_names"].get(node, node)
                if rel.startswith("INV_"):
                    path_str += f" <—[{rel[4:]}]— {node_name}"
                else:
                    path_str += f" —[{rel}]—> {node_name}"
            lines.append(path_str)

    return "\n".join(lines)


def _extract_paths(subgraph: dict, start_node: str, max_depth: int = 3) -> list:
    """从 start_node 出发的所有路径"""
    paths = []
    frontier = [[(None, start_node)]]

    for depth in range(max_depth):
        new_frontier = []
        for path in frontier:
            last_node = path[-1][1]
            extended = False

            for head, rel, tail in subgraph["edges"]:
                if head == last_node and tail != start_node:
                    new_path = path + [(rel, tail)]
                    if not _has_cycle(new_path):
                        new_frontier.append(new_path)
                        extended = True
                        paths.append(new_path[1:])  # 去掉起始节点
                elif tail == last_node and head != start_node:
                    new_path = path + [(f"INV_{rel}", head)]
                    if not _has_cycle(new_path):
                        new_frontier.append(new_path)
                        extended = True
                        paths.append(new_path[1:])

        frontier = new_frontier
        if not frontier:
            break

    return paths


def _has_cycle(path: list) -> bool:
    """检测路径是否有环"""
    nodes = [node for _, node in path]
    return len(nodes) != len(set(nodes))
```

**策略3: GraphPrompt**——带图结构的 Prompt 模板

```python
GRAPH_PROMPT_TEMPLATE = """你是一个知识图谱推理专家。以下是查询相关的结构化知识。

┌─────────────────────────────────────────────┐
│  知识图谱上下文                              │
├─────────────────────────────────────────────┤
│  实体:                                       │
│  {entity_list}                               │
│                                             │
│  关系（以三元组形式列出）:                     │
│  {triples}                                   │
│                                             │
│  关联路径:                                   │
│  {paths}                                     │
└─────────────────────────────────────────────┘

用户问题: {query}

请基于以上知识图谱信息，按照以下步骤推理:
1. 确定问题涉及的实体
2. 跟踪实体之间的关系路径
3. 综合路径中的信息得出答案
4. 如果图谱信息不足以回答问题，明确说明

答案:
"""


def build_graph_prompt(
    subgraph: dict,
    query: str,
    seed_entity_ids: list
) -> str:
    """构建带图结构的 Prompt"""
    # 实体列表
    entity_list = "\n".join([
        f"  - {subgraph['entity_names'].get(eid, eid)}"
        for eid in subgraph["nodes"]
    ])

    # 三元组列表
    triples = "\n".join([
        f"  - ({subgraph['entity_names'].get(h, h)}, {r}, {subgraph['entity_names'].get(t, t)})"
        for h, r, t in subgraph["edges"]
    ])

    # 路径描述
    paths = serialize_as_paths(subgraph, seed_entity_ids)

    return GRAPH_PROMPT_TEMPLATE.format(
        entity_list=entity_list,
        triples=triples,
        paths=paths,
        query=query
    )
```

#### 2.3.2 策略对比

| 策略 | Token 消耗 | 推理友好度 | 结构保留度 | 适合的查询类型 |
|------|-----------|-----------|-----------|--------------|
| **三元组列表** | 低 | 中 | 低（丢失连接结构） | 简单事实查询 |
| **路径描述** | 中 | 高（展示推理链） | 中 | 多跳推理 |
| **GraphPrompt** | 中高 | 高（多视角信息） | 高（保留图结构） | 复杂分析型查询 |
| **邻接描述** | 中 | 中 | 中 | 中心实体查询 |

#### 2.3.3 与文档上下文联合注入

最佳实践不是让 LLM 只看图，而是**联合注入图信息 + 文档块**：

```python
def build_hybrid_context(
    subgraph: dict,
    query: str,
    vector_docs: list[str],
    seed_entity_ids: list,
    max_tokens: int = 4000
) -> str:
    """
    构建混合上下文：图谱子图 + 相关的文档块。

    策略:
    1. 图优先：子图信息压缩度高，优先放入
    2. 文档补充：用剩余 Token 放入最相关的文档块
    3. 标注来源：让 LLM 知道哪些信息来自图，哪些来自文档
    """
    import tiktoken
    encoder = tiktoken.get_encoding("cl100k_base")

    # 构建图部分
    triples = serialize_as_triples(subgraph)
    paths = serialize_as_paths(subgraph, seed_entity_ids)

    graph_section = f"""【知识图谱上下文】
实体关系（三元组）:
{triples}

关联路径:
{paths}
"""
    graph_tokens = len(encoder.encode(graph_section))
    remaining = max_tokens - graph_tokens

    # 构建文档部分（使用剩余 tokens）
    docs_section = "\n【相关文档片段】\n"
    docs_tokens = len(encoder.encode(docs_section))
    remaining -= docs_tokens

    for doc in vector_docs:
        doc_text = f"文档片段：{doc}\n---\n"
        doc_tokens = len(encoder.encode(doc_text))
        if remaining - doc_tokens > 100:  # 至少留 100 tokens 给答案
            docs_section += doc_text
            remaining -= doc_tokens
        else:
            break

    return graph_section + docs_section
```

---

## 3. 关键算法详解

### 3.1 BFS/DFS 在 Graph RAG 中的应用

```
BFS（广度优先） vs DFS（深度优先）在 Graph RAG 中的选择:

查询类型: "Hinton 教过哪些知名学生?"
  → BFS: 从 Hinton 出发，依次展开所有学生关系
  → 结果: (Hinton, STUDENT_OF, Bengio), (Hinton, STUDENT_OF, Sutskever), ...

查询类型: "Bengio 和 Hinton 之间有什么关系?"
  → DFS: 寻找从 Bengio 到 Hinton 的路径
  → 结果: Bengio → 共同获奖 → Hinton

选择策略:
  - 开放查询 (Open-ended) → BFS（宽泛的上下文）
  - 路径查询 (Path-seeking) → DFS（关系追踪）
  - 溯源查询 (Source tracing) → BFS + 深度限制（防止发散）
```

### 3.2 Personalized PageRank 详解

```
PPR 在 Graph RAG 中的直观理解:

想象在图上随机游走:
  1. 从种子实体出发（如 Bengio）
  2. 每次以概率 α 回到种子实体（重启）
  3. 以概率 (1-α) 随机选择一条边走到邻居

经过无限步后，每个节点被访问的概率就是该节点的 PPR 得分。

这捕捉了"节点相对于查询的相关性"：
  - 距离种子实体越近 → 得分越高
  - 与种子实体连接越多的节点 → 得分越高
  - 孤立但高.PageRank 的节点 → 仍可能得分低

优势:
  - 不仅考虑跳数，还考虑图结构（节点度、连接强度）
  - 可有效处理图中"枢纽"节点（hubs）
  - 参数 α 控制"探索半径"
    α 大 → 近距离探索（局部性更强）
    α 小 → 远距离探索（覆盖更广）
```

### 3.3 基于路径的检索 vs 基于子图的检索

```
                        基于路径                       基于子图
               ┌─────────────────────────┐  ┌─────────────────────────┐
  目标         │ 特定实体间的连接关系       │  │ 实体周围的完整知识结构     │
  输出         │ 线性路径 (路径列表)        │  │ 图结构 (节点+边)         │
  查询类型     │ "A 和 B 的关系?"          │  │ "介绍一下 A"             │
  算法         │ BFS 最短路径 / DFS 路径   │  │ k-hop + PageRank        │
  信息完整性   │ 低（只保留路径信息）        │  │ 高（保留上下文）          │
  Token消耗   │ 低                        │  │ 高                      │
  推理难度     │ 低（线性）                 │  │ 中（需要理解图结构）       │
               │                           │  │                          │
  举例:        │ Yoshua Bengio             │  │ (Bengio)                 │
               │   ─[ADVISED_BY]─→         │  │   │     │     │         │
               │   Geoffrey Hinton         │  │ (获奖) (导师) (学生)     │
               │                           │  │   │     │     │         │
               │                           │  │ (图灵奖)(Hinton)(Goodf.) │
               └─────────────────────────┘  └─────────────────────────┘
```

### 3.4 LLM Graph Reasoning 前沿方法

```
GraphToken (2024):
  将图结构编码为特殊的 Token 嵌入，使 Transformer 能够原生理解图结构。
  
  Token 序列: [CLS] [ENTITY_Bengio] [REL_WON] [ENTITY_TuringAward] [SEP]

GPT-Graph:
  将图推理分解为多步，每一步 LLM 决定"下一步访问哪个节点"。
  
  Step 1: 当前在 Bengio，可选边: [WON→TuringAward, ADVISED_BY→Hinton]
  Step 2: 选择 ADVISED_BY→Hinton
  Step 3: 当前在 Hinton，检查是否回答了问题...

GraphRAG (Microsoft, 2024):
  使用社区检测 (Community Detection) 将图划分为模块，
  对每个社区生成文本摘要，检索时先匹配社区再匹配节点。

  优点: 可以回答需要综合多个实体信息的全局性问题
  缺点: 社区构建成本高，不适合细粒度检索
```

**这些方法的共同趋势**：
1. 从"图结构→文本"的简单转换，走向"图原生推理"
2. 多步推理 Agent 化（LLM 主动决策下一跳）
3. 社区级 + 节点级联合检索

---

## 4. 多跳推理的挑战

### 4.1 跳数选择：信息不足 vs 噪声引入

```
跳数 k 的选择权衡:

k=1（过少）:                             k=3（过多）:
┌────────────────────┐                  ┌────────────────────────────┐
│ (Bengio)           │                  │        (Bengio)            │
│    │               │                  │     /    |    \            │
│ (Hinton)           │                  │ (图灵奖) (Hinton) (Goodf.) │
│                    │                  │    /    /   \     |        │
│ 信息不足!           │                  │ (LeCun) (Sutsk.) (LeCun)  │
│ 无法确认是否需要    │                  │   |       |       |        │
│ 更多上下文          │                  │ (CNN)   (Seq2Seq) (CNN)   │
└────────────────────┘                  │   |       |       |        │
                                        │ (ImageNet) (...)  (...)    │
                                        │                            │
                                        │ 噪声过多!                    │
                                        │ 难以聚焦问题核心              │
                                        └────────────────────────────┘

动态跳数选择策略:
  1. 先做 1-hop → 评估子图是否足够回答问题
  2. 如果不够 → 扩展到 2-hop → 再评估
  3. 直到找到答案或达到最大跳数
```

```python
def adaptive_subgraph_search(
    graph: KnowledgeGraph,
    seed_entity_ids: list,
    query: str,
    llm_client,
    max_hops: int = 3
) -> dict:
    """
    自适应跳数扩展：逐步扩展子图，每次扩展后评估是否足够回答问题。

    这种策略可以有效控制子图大小，避免 k 固定导致的欠扩展或过扩展。
    """
    current_subgraph = {
        "nodes": set(seed_entity_ids),
        "edges": [],
        "entity_names": {eid: graph.entities[eid]["name"] for eid in seed_entity_ids}
    }

    for hop in range(1, max_hops + 1):
        # 扩展子图
        expanded = k_hop_expansion(graph, seed_entity_ids, k=hop)
        current_subgraph = {
            "nodes": set(expanded["nodes"]),
            "edges": expanded["edges"],
            "entity_names": expanded["entity_names"]
        }

        # 检查是否足够回答问题
        prompt = f"""以下是从知识图谱中检索到的信息：

{serialize_as_triples(current_subgraph)}

问题: {query}

基于以上信息，是否能完整回答该问题？
- 如果信息足够，回答 "SUFFICIENT"
- 如果信息不足，需要更多图谱信息，回答 "INSUFFICIENT"
- 如果问题无法由图谱信息回答，回答 "UNABLE"

只输出一个词。
"""

        response = llm_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.0,
            max_tokens=10
        )

        verdict = response.choices[0].message.content.strip()

        if verdict == "SUFFICIENT":
            print(f"在第 {hop}-hop 找到足够信息")
            break
        elif verdict == "UNABLE":
            print("问题无法通过图谱信息回答")
            break

    return current_subgraph
```

### 4.2 路径发散（Path Explosion）

```
路径发散问题:

在大型知识图谱中，一个实体可能关联数百个其他实体。

例如:
  "微软" 1-hop 邻居可能包含:
    - 产品: Windows, Office, Azure, Xbox, Surface, ...
    - 高管: Satya Nadella, Bill Gates, Steve Ballmer, ...
    - 收购: LinkedIn, GitHub, Activision, ...
    - 合作方: Intel, AMD, SAP, ...
    - 竞争对手: Google, Apple, Amazon, ...

如果全量扩展 → 子图规模指数级增长 → Token 消耗爆炸 → LLM 难以处理

解决方案:
  1. 关系过滤: 只保留问题相关的关系类型
  2. PageRank 排序: 只保留 top-k 节点
  3. 基于 LLM 的定向扩展: LLM 决定下一步访问哪个邻居
  4. 度剪枝: 跳过度超过阈值的高枢纽节点
```

```python
def pruned_subgraph_expansion(
    graph: KnowledgeGraph,
    seed_entity_ids: list,
    query: str,
    llm_client,
    k: int = 2,
    max_nodes: int = 30,
    max_edges: int = 50
) -> dict:
    """
    带剪枝的子图扩展。

    策略:
    1. 先用 LLM 识别需要关注的关系类型
    2. k-hop 扩展 + 关系过滤
    3. PageRank 排序，只保留 top-k 节点
    4. 如果仍然过大，进一步裁剪
    """
    # 1. LLM 识别相关关系类型
    relevant_relations = relation_filter_for_query(query, llm_client)
    print(f"查询相关的关系类型: {relevant_relations}")

    # 2. 带关系过滤的 k-hop 扩展
    subgraph = k_hop_expansion(
        graph, seed_entity_ids, k=k,
        relation_filter=relevant_relations if relevant_relations else None
    )

    # 3. PageRank 排序 + 剪枝
    scores = personalized_pagerank(graph, seed_entity_ids)
    top_nodes = rank_and_filter_nodes(scores, top_k=max_nodes)

    top_node_ids = set(eid for eid, _ in top_nodes)
    filtered_edges = [
        (h, r, t) for h, r, t in subgraph["edges"]
        if h in top_node_ids and t in top_node_ids
    ]

    # 4. 如果边仍然过多，按置信度/重要性裁剪
    if len(filtered_edges) > max_edges:
        # 按 PageRank 得分加权排序
        edge_scores = []
        for h, r, t in filtered_edges:
            edge_score = scores.get(h, 0) + scores.get(t, 0)
            edge_scores.append((edge_score, h, r, t))

        edge_scores.sort(reverse=True, key=lambda x: x[0])
        filtered_edges = [(h, r, t) for _, h, r, t in edge_scores[:max_edges]]

    return {
        "nodes": top_node_ids,
        "edges": filtered_edges,
        "scores": {eid: scores[eid] for eid in top_node_ids},
        "entity_names": subgraph["entity_names"]
    }
```

### 4.3 不完整图上的推理

知识图谱永远不可能是完美的。实体缺失、关系遗漏是常态。Graph RAG 需要优雅处理信息缺失。

```python
def reasoning_with_missing_info(
    subgraph: dict,
    query: str,
    vector_docs: list[str],
    llm_client
) -> str:
    """
    不完整图上的推理。

    策略:
    1. 先尝试用图信息回答问题
    2. 如果信息不足，从文档块中补充
    3. 如果文档也不足，明确告知用户知识缺口
    """
    prompt = f"""基于以下知识图谱信息和相关文档回答问题。

【知识图谱信息】
{serialize_as_triples(subgraph)}

【相关文档片段】
{chr(10).join(vector_docs[:3])}

【问题】
{query}

【推理规则】
1. 优先使用知识图谱中确认的事实
2. 如果图谱信息不足，尝试从文档中寻找
3. 如果图谱和文档都缺乏完整信息，基于已有知识做有依据的推理
   但要明确标注哪些部分是基于推理的
4. 如果完全无法回答，请说明缺什么信息

【答案】
"""

    response = llm_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.3,
    )

    return response.choices[0].message.content
```

---

## 5. 与 Vector RAG 的深度融合

实践证明，Graph RAG 和 Vector RAG 的最佳关系是**互补**而非替代。

### 5.1 混合架构设计

```
                   ┌─────────────────────────┐
                   │     用户查询             │
                   └───────────┬─────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                 │
              ▼                ▼                 ▼
   ┌─────────────────┐  ┌─────────────────┐
   │  Vector RAG 分支 │  │  Graph RAG 分支   │
   │                  │  │                  │
   │  1. 向量化查询    │  │  1. 实体映射      │
   │  2. 检索 Top-K    │  │  2. 子图检索      │
   │  3. 生成候选回答  │  │  3. 子图序列化    │
   └────────┬─────────┘  └────────┬─────────┘
            │                     │
            └──────────┬──────────┘
                       │
                       ▼
              ┌─────────────────────┐
              │  结果融合            │
              │  - RRF (Reciprocal   │
              │    Rank Fusion)      │
              │  - 加权合并          │
              │  - 分层策略          │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │  LLM 最终推理        │
              │  综合图+文信息生成    │
              └─────────────────────┘
```

### 5.2 LangChain 中结合 Graph + Vector 检索

```python
"""
LangChain 混合检索实现：同时使用向量存储和图数据库。
"""

from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import FAISS
from langchain.graphs import Neo4jGraph
from langchain.chains import GraphCypherQAChain
from langchain.llms import OpenAI
from typing import List


class HybridGraphVectorRetriever:
    """
    混合检索器：同时从向量数据库和知识图谱检索。
    """

    def __init__(
        self,
        vector_store: FAISS,
        graph: Neo4jGraph,
        llm_client,
        embedding_model
    ):
        self.vector_store = vector_store
        self.graph = graph
        self.llm = llm_client
        self.embedding_model = embedding_model

    def retrieve(
        self,
        query: str,
        top_k_vector: int = 5,
        top_k_graph: int = 20
    ) -> dict:
        """
        同时从向量和图检索，返回融合结果。
        """
        # 1. 向量检索
        vector_docs = self.vector_store.similarity_search(query, k=top_k_vector)
        vector_texts = [doc.page_content for doc in vector_docs]

        # 2. 图检索
        # 2.1 实体识别
        entities = self._extract_entities(query)

        # 2.2 子图查询
        subgraph = None
        if entities:
            subgraph = self._query_graph(entities, top_k_graph)

        return {
            "vector_docs": vector_texts,
            "graph_subgraph": subgraph,
            "entities": entities
        }

    def _extract_entities(self, query: str) -> list:
        """从查询中提取实体"""
        # 使用 NER 或 LLM 提取实体
        entities = extract_entities_with_llm(
            query,
            entity_types=["PERSON", "ORG", "LOC", "PRODUCT"],
            llm_client=self.llm
        )
        return [e["text"] for e in entities]

    def _query_graph(self, entity_names: list, max_nodes: int) -> dict:
        """
        用 Cypher 查询图数据库获取子图。

        对于 Neo4j 等图数据库，可以使用 Cypher 查询：
        MATCH (n)-[r]-(m)
        WHERE n.name IN $entity_names
        RETURN n, r, m
        LIMIT $max_nodes
        """
        # 在实际的 Neo4j 集成中：
        # query = """
        # MATCH (n)-[r]-(m)
        # WHERE n.name IN $entity_names
        # RETURN n.name as source, type(r) as relation, m.name as target
        # LIMIT $max_nodes
        # """
        # result = self.graph.query(query, params={"entity_names": entity_names})
        #
        # 这里用模拟实现
        return {
            "entities": entity_names,
            "max_nodes": max_nodes
        }

    def generate(self, query: str, retriever_output: dict) -> str:
        """
        基于检索结果生成回答。
        """
        vector_docs = retriever_output["vector_docs"]
        subgraph = retriever_output["graph_subgraph"]

        # 构建混合 Prompt
        prompt_parts = ["基于以下信息回答问题:\n"]

        if vector_docs:
            prompt_parts.append("【相关文档】")
            prompt_parts.extend(vector_docs[:3])
            prompt_parts.append("")

        if subgraph and subgraph.get("entities"):
            prompt_parts.append("【知识图谱实体】")
            prompt_parts.append(", ".join(subgraph["entities"]))
            prompt_parts.append("")

        prompt_parts.append(f"【问题】{query}")
        prompt_parts.append("【回答】")

        prompt = "\n".join(prompt_parts)

        response = self.llm.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3
        )

        return response.choices[0].message.content
```

### 5.3 结果融合策略对比

```
RRF (Reciprocal Rank Fusion):
  score(d) = Σ 1 / (k + rank_i(d))
  
  将向量检索和图检索的结果按排名融合。
  图检索的结果（子图序列化后）和文档块在同一排名体系中。

  优势: 不需要权重调参，对排名敏感，但不过度依赖绝对分数

加权合并:
  score = α * vector_score + (1-α) * graph_score
  
  需要根据任务调参 α。
  事实性问题→高 α（图更重要）
  开放式问答→低 α（文档更重要）

分层策略:
  第一层: 图中是否有精确匹配的三元组？
    是 → 直接使用图结果
    否 → 第二层: 文档搜索
    文档有 → 使用文档结果
    都无 → LLM 自身知识 / 明确告知无法回答
```

```python
def rrf_fusion(
    vector_results: list[tuple[str, float]],   # [(doc_text, score), ...]
    graph_results: list[tuple[str, float]],    # [(triple_text, score), ...]
    k: int = 60
) -> list[tuple[str, float]]:
    """
    Reciprocal Rank Fusion 融合两个检索结果。

    Args:
        vector_results: 向量检索结果（按得分降序）
        graph_results: 图检索结果（按得分降序）
        k: RRF 常数（通常 60）

    Returns:
        融合后的结果（按 RRF 得分降序）
    """
    rrf_scores = {}

    # 向量检索结果
    for rank, (doc, _) in enumerate(vector_results, start=1):
        rrf_scores[doc] = rrf_scores.get(doc, 0) + 1 / (k + rank)

    # 图检索结果
    for rank, (triple, _) in enumerate(graph_results, start=1):
        rrf_scores[triple] = rrf_scores.get(triple, 0) + 1 / (k + rank)

    # 排序
    sorted_items = sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)
    return sorted_items


def weighted_fusion(
    vector_results: list[tuple[str, float]],
    graph_results: list[tuple[str, float]],
    alpha: float = 0.3
) -> list[tuple[str, float]]:
    """
    加权融合。

    alpha: 图结果的权重。(1-alpha) 是向量结果的权重。
    对于事实性查询，建议 alpha > 0.5；
    对于开放式查询，建议 alpha < 0.3。
    """
    # 归一化得分
    def normalize(results):
        if not results:
            return []
        scores = [s for _, s in results]
        max_score = max(scores) if scores else 1
        return [(text, s / max_score) for text, s in results]

    norm_vector = normalize(vector_results)
    norm_graph = normalize(graph_results)

    # 合并得分
    fused = {}
    for text, score in norm_vector:
        fused[text] = fused.get(text, 0) + (1 - alpha) * score

    for text, score in norm_graph:
        fused[text] = fused.get(text, 0) + alpha * score

    return sorted(fused.items(), key=lambda x: x[1], reverse=True)
```

---

## 6. 性能与延迟分析

### 6.1 延迟对比

```
Graph RAG vs Vector RAG 各阶段延迟 (估算):

Vector RAG:
  ┌─────────────────────────────────┬────────────┐
  │  阶段                            │  延迟       │
  ├─────────────────────────────────┼────────────┤
  │  1. 查询嵌入                     │  100-500ms  │
  │  2. 向量检索 (Top-K, ~10M)       │  50-200ms   │
  │  3. 文档拼接 + Prompt 构建        │  <10ms      │
  │  4. LLM 生成                     │  500-3000ms │
  │  ─────────────────────────────── │            │
  │  总计 (不含 LLM 生成)             │  150-710ms  │
  └─────────────────────────────────┴────────────┘

Graph RAG:
  ┌─────────────────────────────────┬────────────┐
  │  阶段                            │  延迟       │
  ├─────────────────────────────────┼────────────┤
  │  1. 查询实体映射 (NER + LLM)     │  500-2000ms │
  │  2. 图检索 (k-hop + 排序)        │  10-500ms   │
  │  3. 子图序列化 + Prompt 构建     │  10-50ms    │
  │  4. LLM 生成                     │  500-3000ms │
  │  ─────────────────────────────── │            │
  │  总计 (不含 LLM 生成)             │  520-2550ms │
  └─────────────────────────────────┴────────────┘

结论:
  - Graph RAG 在检索阶段比 Vector RAG 慢（主要瓶颈在实体映射）
  - LLM 生成阶段都可以通过选用小模型优化
  - 对于多跳推理，Graph RAG 的额外延迟通常是值得的
```

### 6.2 Token 消耗分析

```
子图序列化的 Token 消耗:

假设子图有 N 个节点，E 条边：

三元组列表: ~(E × 平均关系名长度) tokens
路径描述:   ~(N × 平均路径长度) tokens
GraphPrompt: ~(N + E) × 1.5 tokens

实际例子:
  k=1, 种子=2个实体, 图有 5 节点, 8 条边
    三元组列表: ~120 tokens
    路径描述:   ~200 tokens
    GraphPrompt: ~350 tokens

  k=2, 种子=2个实体, 图有 20 节点, 45 条边
    三元组列表: ~500 tokens
    路径描述:   ~900 tokens
    GraphPrompt: ~1200 tokens

  k=3, 种子=2个实体, 图有 80 节点, 200 条边
    三元组列表: ~2000 tokens  (已经接近上下文限制)
    路径描述:   ~4000+ tokens (可能超限)
    GraphPrompt: ~5000+ tokens (可能超限)

最佳实践: 将子图 Token 消耗控制在 2000 tokens 以内，
         保留空间给文档块和 LLM 生成。
```

### 6.3 缓存策略

```python
class GraphRAGCache:
    """
    Graph RAG 缓存系统。

    缓存层级:
    Level 1: 查询 → 答案 缓存（针对重复查询）
    Level 2: 实体 → 子图 缓存（避免重复图检索）
    Level 3: 节点 → PPR 得分 缓存（节点的相关性得分）
    """

    def __init__(self, ttl_seconds: int = 3600):
        self.answer_cache = {}       # query_hash -> (answer, timestamp)
        self.subgraph_cache = {}     # entity_tuple_hash -> subgraph
        self.ppr_cache = {}          # seed_set_hash -> ppr_scores
        self.ttl = ttl_seconds

    def get_cached_answer(self, query: str) -> str:
        """Level 1: 检查是否有完全相同的查询缓存"""
        import hashlib, time
        query_hash = hashlib.md5(query.encode()).hexdigest()

        if query_hash in self.answer_cache:
            answer, timestamp = self.answer_cache[query_hash]
            if time.time() - timestamp < self.ttl:
                return answer
        return None

    def cache_answer(self, query: str, answer: str):
        import hashlib, time
        query_hash = hashlib.md5(query.encode()).hexdigest()
        self.answer_cache[query_hash] = (answer, time.time())

    def get_cached_subgraph(self, entity_names: tuple) -> dict:
        """Level 2: 相同实体组合的子图缓存"""
        key = tuple(sorted(entity_names))
        return self.subgraph_cache.get(key)

    def cache_subgraph(self, entity_names: list, subgraph: dict):
        key = tuple(sorted(entity_names))
        self.subgraph_cache[key] = subgraph

    def get_cached_ppr(self, seed_ids: tuple) -> dict:
        """Level 3: PPR 得分缓存"""
        key = tuple(sorted(seed_ids))
        return self.ppr_cache.get(key)

    def cache_ppr(self, seed_ids: list, scores: dict):
        key = tuple(sorted(seed_ids))
        self.ppr_cache[key] = scores

    def invalidate(self, pattern: str = None):
        """缓存失效"""
        if pattern is None:
            self.answer_cache.clear()
            self.subgraph_cache.clear()
            self.ppr_cache.clear()
        else:
            # 根据模式选择性失效
            # 例如 invalidate("entity:Apple") 使所有涉及 Apple 的缓存失效
            pass


class CachedGraphRAGPipeline:
    """
    带缓存的 Graph RAG 管线。
    """

    def __init__(self, graph, llm_client, vector_store=None):
        self.graph = graph
        self.llm = llm_client
        self.vector_store = vector_store
        self.cache = GraphRAGCache()

    def query(self, question: str) -> str:
        # 检查 Level 1 缓存
        cached = self.cache.get_cached_answer(question)
        if cached:
            print("缓存命中（答案级）")
            return cached

        # 实体映射
        entities = query_to_entities_llm(
            question, ENTITY_TYPES, self.llm,
            list(self.graph.entities.keys())
        )

        if not entities:
            # 没有匹配实体 → 降级到纯 Vector RAG
            return self._fallback_to_vector(question)

        # 检查 Level 2 缓存
        entity_names = tuple(e["kg_entity"] for e in entities)
        subgraph = self.cache.get_cached_subgraph(entity_names)

        if not subgraph:
            subgraph = subgraph_query(
                self.graph,
                [e["kg_entity"] for e in entities],
                k=2
            )
            self.cache.cache_subgraph(entity_names, subgraph)

        # 构建 Prompt 并推理
        context = build_hybrid_context(
            subgraph, question,
            vector_docs=[]  # 可传入 Vector RAG 结果
        )

        answer = self._generate_with_context(question, context)
        self.cache.cache_answer(question, answer)
        return answer

    def _fallback_to_vector(self, question: str) -> str:
        """降级到向量检索"""
        # 简化的降级实现
        return "无法在图谱中找到相关实体。"

    def _generate_with_context(self, question: str, context: str) -> str:
        response = self.llm.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "基于提供的知识图谱信息回答问题。"},
                {"role": "user", "content": f"{context}\n\n问题: {question}\n\n答案:"}
            ],
            temperature=0.3
        )
        return response.choices[0].message.content
```

---

## 7. 全书架思考：从 Graph RAG 到 Agentic Graph Reasoning

### 7.1 当前 Graph RAG 的局限

1. **一次性检索 → 静态推理**：当前方式是在检索后一次性交给 LLM。如果 LLM 发现信息不足，无法主动回退。
2. **图结构未被 LLM 原生理解**：序列化后的图结构丢失了拓扑信息，LLM 只能看到"文本化"的图。
3. **答案生成缺乏验证**：LLM 可能基于图信息正确推理，也可能幻觉。没有机制验证推理链。

### 7.2 未来方向：Agentic Graph Reasoning

```
第一代: Graph RAG (当前)
  查询 → 检索子图 → LLM 生成答案（一次完成）
  图是"被动"的——检索一次后不再交互

第二代: Agentic Graph Reasoning (趋势)
  查询 → Agent 初始化 → 多步推理循环:
    Step 1: 检索种子实体子图
    Step 2: LLM 评估是否足够 → 如果不够，决定下一步
    Step 3: Agent 在图上有策略地导航（访问新节点）
    Step 4: 整合多步信息 → 生成答案
    Step 5: 验证推理链的完整性

  图是"主动"的——Agent 在图上自由导航，按需扩展
```

```python
"""
Agentic Graph Reasoning 概念实现。

核心思想：
  LLM 作为 Agent 在图上导航，每一步决定：
  1. 当前信息是否足够回答问题？
  2. 如果不够，下一步访问哪个邻居？
  3. 已经访问了哪些节点，如何整合信息？
"""


class GraphReasoningAgent:
    """
    图推理 Agent：在知识图谱上自主导航完成多跳推理。
    """

    def __init__(self, graph: KnowledgeGraph, llm_client, max_steps: int = 5):
        self.graph = graph
        self.llm = llm_client
        self.max_steps = max_steps
        self.visited_nodes = set()
        self.collected_triples = []
        self.reasoning_path = []

    def reason(self, query: str) -> str:
        """
        在图上自主推理回答问题。
        """
        # Step 1: 识别种子实体
        entities = query_to_entities_llm(query, {}, self.llm, list(self.graph.entities.keys()))
        if not entities:
            return "无法在图谱中找到相关实体。"

        current_entities = entities
        step = 0

        while step < self.max_steps:
            step += 1
            print(f"\n推理步骤 {step}/{self.max_steps}")

            # 获取当前实体邻居
            new_triples = self._explore_neighbors(current_entities)
            self.collected_triples.extend(new_triples)

            # 评估是否足够
            verdict = self._evaluate_sufficiency(query)

            if verdict["sufficient"]:
                print("  → 信息足够，生成答案")
                return self._generate_answer(query)

            elif verdict.get("next_entities"):
                # 需要继续探索
                print(f"  → 继续探索: {verdict['reason']}")
                current_entities = verdict["next_entities"]

            else:
                # 无法获取更多信息
                print(f"  → 无法获取更多信息: {verdict['reason']}")
                break

        # 达到最大步数 → 基于已有信息回答
        return self._generate_answer(query, partial=True)

    def _explore_neighbors(self, entities: list) -> list:
        """探索实体的邻居"""
        new_triples = []

        for entity in entities:
            entity_id = self.graph.get_entity_id(entity.get("kg_entity", entity))
            if not entity_id or entity_id in self.visited_nodes:
                continue

            self.visited_nodes.add(entity_id)

            # 获取 1-hop 邻居
            for rel in self.graph.relations:
                if rel["head_id"] == entity_id:
                    new_triples.append(rel)
                elif rel["tail_id"] == entity_id:
                    new_triples.append(rel)

        return new_triples

    def _evaluate_sufficiency(self, query: str) -> dict:
        """评估当前信息是否足够回答问题"""
        context = serialize_as_triples({
            "edges": [(r["head_id"], r["relation"], r["tail_id"]) for r in self.collected_triples],
            "entity_names": self.graph.entities
        })

        prompt = f"""你是一个图推理 Agent。以下是从知识图谱收集到的信息：

{context}

你的任务: 判断当前信息是否足够回答用户问题。

如果足够: 输出 {{"sufficient": true}}
如果不够: 输出 {{"sufficient": false, "reason": "缺少XX信息", "next_entities": ["下一步需要探索的实体名"]}}

只输出 JSON。

问题: {query}
"""

        import json
        response = self.llm.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.0,
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)

    def _generate_answer(self, query: str, partial: bool = False) -> str:
        """基于收集的信息生成最终答案"""
        context = serialize_as_triples({
            "edges": [(r["head_id"], r["relation"], r["tail_id"]) for r in self.collected_triples],
            "entity_names": self.graph.entities
        })

        partial_note = ""
        if partial:
            partial_note = "\n注意: 以下信息可能不完整，请基于已知信息给出最佳答案，并注明不确定性。"

        prompt = f"""基于以下知识图谱信息回答问题。

{context}
{partial_note}

问题: {query}

答案:
"""

        response = self.llm.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3
        )

        return response.choices[0].message.content
```

### 7.3 设计原则总结

```
Graph RAG 系统设计的十个原则:

1. 图不是万能的：Vector RAG + Graph RAG 互补 > 任一单独
2. 精度 > 覆盖：小图高精度 > 大图低精度
3. 实体映射是瓶颈：查不到正确实体 = 整个流程失败
4. 跳数是核心参数：k=1 太浅，k=3 太深，动态选择最优
5. 剪枝不可少：不剪枝的 k-hop 在真实图上会指数级爆炸
6. Token 预算是硬约束：子图序列化消耗的 tokens 必须可预测
7. Cache 是生命线：图检索比向量检索更慢，缓存策略至关重要
8. 降级策略是安全网：图查不到时自动降级到向量检索
9. 推理路径可追溯：让 LLM 展示用了哪些三元组得出答案
10. 持续优化闭环：用户反馈 → 图谱补全 → 检索质量提升
```

---

## 总结

Graph RAG 通过三个阶段（实体映射、子图检索、子图推理）将知识图谱的结构化信息引入 LLM 推理过程，特别擅长解决传统 Vector RAG 难以应对的多跳推理问题。然而，Graph RAG 并非万能的：它依赖高质量的图谱构建，在实体映射阶段存在瓶颈，且子图检索的 Token 消耗需要精心控制。

在实践中，Graph RAG 的最佳定位是**Vector RAG 的增强组件**而非替代品。两者融合的混合架构，结合缓存、剪枝和动态跳数选择策略，能够在多跳推理场景下显著提升 RAG 系统的能力上限。

未来的发展方向是 Agentic Graph Reasoning——让 LLM 作为 Agent 在图上主动导航，通过多步推理和自评估机制，实现更灵活和更准确的图基推理。
