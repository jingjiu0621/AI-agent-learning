# 9.4.4 Graph Query — 图谱查询：Cypher / SPARQL / GraphQL

> **核心问题**：知识图谱构建完成后，如何高效、灵活地查询其中的结构化知识？在 Graph RAG 场景中，图查询语言是 LLM 与知识图谱之间的"检索接口"，查询语言的选型直接影响 RAG 系统的召回质量与延迟。

---

## 1. 背景与问题

### 1.1 为什么需要图查询语言？

知识图谱本质上是一个由节点（实体）和边（关系）构成的图结构数据。与关系数据库的 SQL 不同，图查询语言需要原生支持：

- **多跳关系遍历**：如"查找与某药物相互作用的药物所治疗的疾病"
- **变长路径匹配**：如"查找两个人物之间的最短社交路径"
- **模式匹配**：如"查找所有满足'作者-写了-论文-引用-论文'模式的路径"
- **语义推理**：如"根据继承关系推断间接类别成员关系"

### 1.2 RDF 图 vs 属性图：两种世界观

图数据库领域存在两大阵营，它们对"什么是图"的理解不同，也因此使用不同的查询语言：

```
                 图数据库世界
                /            \
            RDF 图          属性图
           /    \            /    \
      语义 Web  知识图谱  Neo4j  Amazon Neptune
       (SPARQL)           (Cypher) (Gremlin)
```

| 维度 | RDF 图 (RDF Graph) | 属性图 (Property Graph) |
|---|---|---|
| 基本单元 | 三元组 (Subject, Predicate, Object) | 节点 + 关系 + 属性 |
| 数据模型 | 严格的三元组结构，每个边都有独立 URI | 节点和关系都可携带 Key-Value 属性 |
| 语义基础 | 形式化语义 (OWL/RDFS)，支持推理 | 无内置推理，依赖应用层逻辑 |
| 标准性 | W3C 标准，跨平台互操作 | 各厂商方言，Cypher/Gremlin |
| 代表语言 | **SPARQL** | **Cypher** (Neo4j) / **Gremlin** (TinkerPop) |
| 典型图库 | Wikidata, DBpedia, YAGO | Neo4j, Amazon Neptune, ArangoDB |

### 1.3 Graph RAG 中的查询语言角色

在 Graph RAG 流程中，图查询语言处于"检索"这一关键环节：

```
用户查询
    │
    ▼
┌─────────────────┐
│  Query Intent    │  ← LLM 理解自然语言
│   Understanding  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  NL2GraphQuery   │  ← LLM 将意图转为图查询语句
│  (LLM 生成查询)   │     (Cypher / SPARQL / GraphQL)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Query Execution │  ← 在图数据库上执行查询
│  (图数据库执行)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Result          │  ← 查询结果作为上下文
│  Post-Processing │     送入 LLM 生成最终回答
└────────┬────────┘
         │
         ▼
      回答
```

两种主流模式：
1. **LLM 生成查询** (Text-to-Cypher/SPARQL)：用户问自然语言 → LLM 生成图查询 → 执行 → 结果作为上下文
2. **预编查询模板**：预定义常见问题的查询模板，LLM 负责参数填充

---

## 2. Cypher — Neo4j 属性图查询语言

### 2.1 设计哲学

Cypher 的设计灵感来自 SQL 和 ASCII 艺术，使用**图案匹配** (Pattern Matching) 的方式来描述图结构。它试图让查询语句看起来就像是在画图：

```
(:Person {name: 'Alice'})-[ :KNOWS ]->(:Person {name: 'Bob'})
```

这种视觉化语法极大地降低了图查询的认知门槛。

### 2.2 核心子句

#### MATCH — 模式匹配（Cypher 的核心）

```cypher
// 查找一个节点
MATCH (n:Person)
WHERE n.name = 'Alice'
RETURN n

// 查找带标签的节点
MATCH (movie:Movie {title: 'The Matrix'})
RETURN movie

// 带关系的模式匹配
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN p.name, m.title
```

**节点模式语法**：
```
(变量:标签 {属性名: 属性值})
```

#### WHERE — 条件过滤

```cypher
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
WHERE p.born >= 1970
  AND m.releaseYear > 2000
  AND m.title STARTS WITH 'The'
RETURN p.name, m.title
```

#### RETURN — 结果返回

支持：返回节点/关系、属性、别名、聚合函数、去重

```cypher
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN p.name AS actor_name, 
       COUNT(m) AS movie_count,
       COLLECT(m.title) AS movies
ORDER BY movie_count DESC
LIMIT 10
```

#### CREATE / MERGE — 创建和合并

```cypher
// 创建节点和关系
CREATE (p:Person {name: 'Alice', age: 30})
CREATE (m:Movie {title: 'Inception', year: 2010})
CREATE (p)-[:ACTED_IN {role: 'Cobb'}]->(m)

// MERGE: 如果存在则不创建（类似 UPSERT）
MERGE (p:Person {id: '123'})
  ON CREATE SET p.name = 'Alice', p.createdAt = timestamp()
  ON MATCH SET p.lastSeen = timestamp()
```

### 2.3 关系模式详解

关系是 Cypher 中最强大的特性：

```cypher
// 基本关系模式
(n)-[:KNOWS]->(m)           // 有向关系
(n)-[:KNOWS]-(m)            // 无向关系（忽略方向）
(n)-[:KNOWS|:FRIENDS]->(m)  // 多类型关系

// 变长关系（Variable-Length Patterns）
(n)-[:KNOWS*2..4]->(m)      // 2到4跳
(n)-[:KNOWS*..4]->(m)       // 最多4跳
(n)-[:KNOWS*2..]->(m)       // 至少2跳，无上限
(n)-[:KNOWS*]->(m)          // 1到任意跳（慎用！）

// 路径变量
MATCH path = (a:Person {name: 'Alice'})-[:KNOWS*1..3]->(b:Person)
RETURN path, length(path) AS path_length
```

### 2.4 进阶查询示例

#### 例1：社交网络中的推荐系统

```cypher
// 为 Alice 推荐可能认识的人 — 朋友的 friend
MATCH (alice:Person {name: 'Alice'})-[:KNOWS]->(friend:Person)-[:KNOWS]->(fof:Person)
WHERE NOT (alice)-[:KNOWS]->(fof)
  AND alice <> fof
RETURN fof.name, COUNT(friend) AS common_friends
ORDER BY common_friends DESC
LIMIT 5
```

#### 例2：药物相互作用分析

```cypher
// 查找与给定药物相互作用的所有药物，以及它们所治疗的疾病
MATCH (drug:Drug {name: 'Aspirin'})-[:INTERACTS_WITH]->(other:Drug)
OPTIONAL MATCH (other)-[:TREATS]->(disease:Disease)
RETURN drug.name AS source_drug,
       other.name AS interacting_drug,
       COLLECT(DISTINCT disease.name) AS treats_diseases,
       COUNT(disease) AS disease_count
ORDER BY disease_count DESC
```

#### 例3：最短路径与路径分析

```cypher
// 查找两个人之间的最短路径
MATCH p = shortestPath(
  (a:Person {name: 'Alice'})-[:KNOWS*]-(b:Person {name: 'David'})
)
RETURN [node IN nodes(p) | node.name] AS path_names,
       length(p) AS path_length

// 所有路径（带上限）
MATCH p = (a:Person {name: 'Alice'})-[:KNOWS*1..4]-(b:Person {name: 'David'})
RETURN p, length(p) AS path_length
ORDER BY path_length
LIMIT 10
```

#### 例4：子图查询

```cypher
// 查询一个共同作者的协作子图
MATCH (author:Author {name: 'John'})-[:CO_AUTHOR_WITH*1..2]-(coauthor:Author)
MATCH (coauthor)-[:WROTE]->(paper:Paper)
WHERE paper.year >= 2020
RETURN coauthor.name AS coauthor,
       COLLECT(DISTINCT paper.title) AS papers,
       COUNT(DISTINCT paper) AS paper_count
ORDER BY paper_count DESC
```

### 2.5 聚合与数据处理

```cypher
// 统计每种类型的电影数量及平均评分
MATCH (m:Movie)
WHERE m.releaseYear >= 2010
RETURN m.genre AS genre,
       COUNT(m) AS movie_count,
       ROUND(AVG(m.rating), 2) AS avg_rating,
       COLLECT(m.title)[0..5] AS sample_titles
ORDER BY movie_count DESC

// 分组统计
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN p.born AS birth_decade,
       COUNT(DISTINCT p) AS actor_count,
       COUNT(m) AS total_movies
ORDER BY birth_decade
```

### 2.6 Cypher 的适用场景

- **Neo4j 用户**：Neo4j 是最流行的图数据库，Cypher 是默认语言
- **属性图场景**：事务处理、实时查询、复杂模式匹配
- **Graph RAG 推荐**：当知识图谱构建在 Neo4j 上时，Cypher 是自然选择
- **团队学习成本低**：语法接近 SQL，开发人员上手快

---

## 3. SPARQL — RDF 图的语义查询

### 3.1 设计哲学

SPARQL (SPARQL Protocol and RDF Query Language) 是 W3C 标准的 RDF 图查询语言。它基于**三元组模式匹配** (Triple Pattern Matching)，查询的"形状"就是数据中要找的子图形状。

RDF 图的基本单元是三元组 `<Subject, Predicate, Object>`，可以看作"主语-谓语-宾语"：

```
<http://example.org/Alice> <http://xmlns.com/foaf/0.1/knows> <http://example.org/Bob> .
```

SPARQL 的查询模式在视觉上与 RDF 数据形状一致，使用变量（如 `?person`）来占位。

### 3.2 查询形式

SPARQL 有四种查询形式：

| 形式 | 作用 | 返回类型 |
|---|---|---|
| `SELECT` | 返回投影变量的绑定值 | 表格 |
| `CONSTRUCT` | 根据结果构建新的 RDF 图 | RDF 图 |
| `ASK` | 返回是否存在匹配 | Boolean |
| `DESCRIBE` | 返回描述匹配资源的 RDF 图 | RDF 图 |

### 3.3 基本 SELECT 查询

```sparql
# 查找所有名为 "Alice" 的人以及他们知道的人
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>

SELECT ?person ?friend ?friendName
WHERE {
  ?person rdf:type foaf:Person .
  ?person foaf:name "Alice" .
  ?person foaf:knows ?friend .
  ?friend foaf:name ?friendName .
}
```

### 3.4 三元组模式详解

```sparql
# 不同的三元组模式

# 模式1: 固定 S, 变量 P, 变量 O
<http://example.org/alice> ?property ?value .

# 模式2: 变量 S, 固定 P, 变量 O
?person foaf:name ?name .

# 模式3: 变量 S, 变量 P, 固定 O
?person ?property "Alice" .

# 模式4: 全部变量
?s ?p ?o .

# 模式5: 使用 RDF 集合
?person foaf:knows [ foaf:name "Bob" ] .   # 空白节点
```

### 3.5 FILTER 与条件过滤

```sparql
PREFIX dbo: <http://dbpedia.org/ontology/>
PREFIX dbr: <http://dbpedia.org/resource/>

SELECT ?city ?population
WHERE {
  ?city rdf:type dbo:City .
  ?city dbo:populationTotal ?population .
  ?city dbo:country dbr:China .
  
  FILTER(?population > 1000000)
  FILTER REGEX(STR(?city), "Shanghai")   # 字符串匹配
}
ORDER BY DESC(?population)
LIMIT 20
```

### 3.6 OPTIONAL 与左外连接

```sparql
# 查找所有电影及其导演（如果已知），也返回没有导演信息的电影
PREFIX dbo: <http://dbpedia.org/ontology/>

SELECT ?movie ?title ?director
WHERE {
  ?movie rdf:type dbo:Film .
  ?movie dbo:title ?title .
  
  OPTIONAL {
    ?movie dbo:director ?director .
  }
}
LIMIT 100
```

### 3.7 UNION 与多模式匹配

```sparql
# 查找导演或主演特定电影的创作者
PREFIX dbo: <http://dbpedia.org/ontology/>

SELECT ?creator ?role ?movie
WHERE {
  ?movie dbo:title "The Matrix"@en .
  
  { ?movie dbo:director ?creator . BIND("director" AS ?role) }
  UNION
  { ?movie dbo:starring ?creator . BIND("star" AS ?role) }
}
```

### 3.8 属性路径 — SPARQL 的"变长关系"

SPARQL 1.1 引入了**属性路径** (Property Paths)，功能与 Cypher 的变长关系类似：

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX dbo: <http://dbpedia.org/ontology/>

# 查找所有子类（传递闭包）
SELECT ?subclass
WHERE {
  dbo:Person rdfs:subClassOf* ?subclass .
}

# 查找朋友的朋友（两跳）
PREFIX foaf: <http://xmlns.com/foaf/0.1/>

SELECT ?person ?fof
WHERE {
  ?person foaf:knows/foaf:knows ?fof .
}

# 使用正则路径操作符
# |  — 或（选择）
# /  — 序列（然后）
# ?  — 可选（0或1次）
# *  — 零次或多次
# +  — 一次或多次

# 查找同辈或叔伯关系
?person (foaf:parent/foaf:sibling)+ ?relative .

# 查找间接合作关系
?researcher dbo:co-author+ ?collaborator .
```

### 3.9 联邦查询 — SERVICE

```sparql
# 同时查询 Wikidata 和 DBpedia
PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX wdt: <http://www.wikidata.org/prop/direct/>
PREFIX dbo: <http://dbpedia.org/ontology/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?cityName ?population ?dbpediaAbstract
WHERE {
  # 从 Wikidata 获取城市人口
  SERVICE <https://query.wikidata.org/sparql> {
    ?city wdt:P31 wd:Q515 .              # instance of city
    ?city rdfs:label ?cityName .
    ?city wdt:P1082 ?population .
    FILTER(LANG(?cityName) = "en")
  }
  
  # 从 DBpedia 获取城市描述
  SERVICE <http://dbpedia.org/sparql> {
    ?dbpediaCity dbo:populationTotal ?population .
    ?dbpediaCity rdfs:comment ?dbpediaAbstract .
    FILTER(LANG(?dbpediaAbstract) = "en")
  }
}
```

### 3.10 CONSTRUCT 查询

```sparql
# 从电影数据构建一个简化的社交网络图
PREFIX dbo: <http://dbpedia.org/ontology/>
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX ex: <http://example.org/>

CONSTRUCT {
  ?actor1 ex:workedWith ?actor2 .
  ?actor2 ex:workedWith ?actor1 .
}
WHERE {
  ?movie dbo:starring ?actor1 .
  ?movie dbo:starring ?actor2 .
  FILTER(?actor1 != ?actor2)
}
```

### 3.11 SPARQL 适用场景

- **语义网/开放数据**：Wikidata、DBpedia、GeoNames 都提供 SPARQL 端点
- **需要推理的图**：RDFS/OWL 推理能力使 SPARQL 适合需要隐式知识推导的场景
- **跨数据源联邦查询**：SERVICE 关键字支持查询多个 SPARQL 端点
- **学术研究与生命科学**：GeneOntology、Bio2RDF 等标准使用 RDF/SPARQL

---

## 4. GraphQL for Graphs

### 4.1 GraphQL 的设计定位

GraphQL 并非专门的图数据库查询语言，它最初是由 Facebook 开发的**API 查询语言**。其核心优势是：

- **客户端精确指定所需字段**，避免 over-fetching / under-fetching
- **强类型 Schema**，自文档化
- **单一端点**替代 REST 的多个端点

当 GraphQL 用于图数据库时，它扮演的是"应用层查询语言"的角色，提供了比 Cypher/SPARQL 更友好的 API 接口。

### 4.2 Dgraph Schema 与 GraphQL

Dgraph 是目前最流行的原生 GraphQL 支持图数据库：

```graphql
# Schema 定义
type Person {
  name: String! @search(by: [hash, trigram])
  age: Int
  friends: [Person] @reverse
  actedIn: [Movie] @reverse
}

type Movie {
  title: String! @search(by: [fulltext])
  releaseYear: Int
  actors: [Person]
}
```

```graphql
# 查询
{
  queryPerson(name: "Alice") {
    name
    age
    friends {
      name
    }
    actedIn {
      title
      releaseYear
    }
  }
}

# 带过滤和排序的查询
{
  queryMovie(filter: { releaseYear: { ge: 2000 } }, order: { asc: title }) {
    title
    releaseYear
    actors(filter: { age: { gt: 30 } }) {
      name
    }
  }
}
```

### 4.3 Amazon Neptune 与 GraphQL

Amazon Neptune 支持通过 AWS AppSync 提供 GraphQL 接口，底层查询引擎仍使用 SPARQL (RDF 模式) 或 Gremlin (属性图模式)：

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  GraphQL     │────▶│  AppSync     │────▶│  Neptune     │
│  Client      │     │  (Resolver)  │     │  (Engine)    │
└──────────────┘     └──────────────┘     └──────────────┘
                            │
                      ┌─────┴──────┐
                      │            │
                      ▼            ▼
                 SPARQL         Gremlin
                 (RDF)         (属性图)
```

### 4.4 GraphQL vs Cypher vs SPARQL 对比

| 维度 | GraphQL | Cypher | SPARQL |
|---|---|---|---|
| **定位** | API 查询语言 | 图数据库查询语言 | RDF 图查询语言 |
| **图模型** | 自定义 (Schema 定义) | 属性图 | RDF 三元组 |
| **主要用途** | 应用层 API | Neo4j 原生查询 | 语义网/Wikidata |
| **是否 W3C 标准** | 否 (由 Meta/GraghQL 基金会维护) | 否 (Neo4j 开源) | 是 |
| **推理能力** | 无内置 | 弱 (需 APOC 插件) | 强 (RDFS/OWL) |
| **联邦查询** | Schema Stitching | APOC 插件 | 原生 SERVICE |
| **变长路径** | 无原生支持 | 原生 `[*n..m]` | 属性路径 |
| **学习曲线** | 低-中 | 中 | 中-高 |
| **Graph RAG 适用性** | 仅作为 API 封装 | 属性图场景首选 | 语义知识图谱场景首选 |

---

## 5. NL2GraphQuery — 自然语言转图查询

### 5.1 核心挑战

将自然语言翻译为图查询语言的难点在于：

1. **图 Schema 理解**：LLM 需要知道节点类型、关系类型、属性名
2. **语义对齐**：自然语言表达可能不直接对应图模型术语
3. **多跳推理**：复杂问题需要多步图遍历
4. **错误容忍**：LLM 生成的查询可能语法错误或语义错误

### 5.2 Schema-Aware Prompt 设计

以下是 Text-to-Cypher 的 Prompt 模板设计：

```
你是一个 Neo4j Cypher 查询生成专家。
根据给定的图 Schema 和用户问题，生成准确的 Cypher 查询。

===== 图 Schema =====
节点类型：
- Person: {name, age, email, occupation}
- Movie: {title, releaseYear, genre, rating}
- Company: {name, industry, location}

关系类型：
- (:Person)-[:ACTED_IN {role, salary}]->(:Movie)
- (:Person)-[:DIRECTED]->(:Movie)
- (:Person)-[:WORKS_FOR]->(:Company)
- (:Person)-[:KNOWS {since}]->(:Person)

===== 规则 =====
1. 只返回 Cypher 查询语句，不要额外解释
2. 总是使用 LIMIT 限制结果数量（默认 50）
3. 对于模糊的实体名称，使用 CONTAINS 或 STARTS WITH
4. 复杂多跳查询使用变长关系优化
5. 使用参数化查询（$param）而非字面量

===== 示例 =====
用户：谁导演了《盗梦空间》？
Cypher: MATCH (p:Person)-[:DIRECTED]->(m:Movie {title: '盗梦空间'})
        RETURN p.name AS director_name

用户：查找与Alice有共同朋友的人
Cypher: MATCH (alice:Person {name: 'Alice'})-[:KNOWS]->(common:Person)-[:KNOWS]->(fof:Person)
        WHERE NOT (alice)-[:KNOWS]->(fof) AND alice <> fof
        RETURN fof.name AS suggested_friend, COUNT(common) AS common_friends
        ORDER BY common_friends DESC LIMIT 10

===== 用户问题 =====
{user_query}
```

### 5.3 Few-Shot 示例自动选择

当 Schema 包含大量节点/关系类型时，将所有信息都塞进 Prompt 不现实。更有效的策略是**基于语义相似度动态选择 Few-Shot 示例**：

```
                         用户查询
                            │
                            ▼
┌───────────────────────────────────────┐
│  Embedding Model                      │
│  (将用户查询转为向量)                   │
└───────────────┬───────────────────────┘
                │
                ▼
┌───────────────────────────────────────┐
│  示例检索: 从示例库中选出 top-k         │
│  语义最相似的 Q-Cypher 对              │
└───────────────┬───────────────────────┘
                │
                ▼
┌───────────────────────────────────────┐
│  LLM: Schema + 选中示例 + 用户查询      │
│  → 生成 Cypher                         │
└───────────────┬───────────────────────┘
                │
                ▼
      生成的 Cypher 查询语句
```

示例库构建示例：

```python
import numpy as np
from openai import OpenAI

client = OpenAI()

# 预定义的查询示例库
EXAMPLE_POOL = [
    {
        "question": "谁导演了《盗梦空间》？",
        "cypher": "MATCH (p:Person)-[:DIRECTED]->(m:Movie {title: '盗梦空间'}) RETURN p.name"
    },
    {
        "question": "Alice有哪些朋友？",
        "cypher": "MATCH (p:Person {name: 'Alice'})-[:KNOWS]->(friend) RETURN friend.name"
    },
    {
        "question": "评分最高的10部电影",
        "cypher": "MATCH (m:Movie) RETURN m.title, m.rating ORDER BY m.rating DESC LIMIT 10"
    },
    # ... 更多示例对
]

def embed_text(text: str) -> list[float]:
    """将文本转为向量"""
    resp = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return resp.data[0].embedding

def select_examples(query: str, k: int = 3) -> list[dict]:
    """基于语义相似度选择最相关的示例"""
    query_emb = np.array(embed_text(query))
    
    scored = []
    for example in EXAMPLE_POOL:
        ex_emb = np.array(embed_text(example["question"]))
        similarity = np.dot(query_emb, ex_emb) / (
            np.linalg.norm(query_emb) * np.linalg.norm(ex_emb)
        )
        scored.append((similarity, example))
    
    scored.sort(key=lambda x: x[0], reverse=True)
    return [ex for _, ex in scored[:k]]
```

### 5.4 查询验证与纠错

LLM 生成的图查询可能有语法或逻辑错误，需要验证和纠错机制：

```python
from neo4j import GraphDatabase

def validate_and_execute_cypher(driver: GraphDatabase.driver, 
                                 cypher_query: str, 
                                 max_retries: int = 2) -> list:
    """
    验证Cypher查询语法，尝试执行，失败时尝试修复
    """
    for attempt in range(max_retries + 1):
        try:
            # 先用 EXPLAIN 验证语法
            with driver.session() as session:
                explain_result = session.run(f"EXPLAIN {cypher_query}")
                explain_result.consume()  # 不抛出异常说明语法正确
                
                # 语法正确，执行查询
                result = session.run(cypher_query)
                return [record.data() for record in result]
                
        except Exception as e:
            error_msg = str(e)
            print(f"查询执行失败 (attempt {attempt + 1}): {error_msg}")
            
            if attempt < max_retries:
                # 使用 LLM 修复查询
                cypher_query = repair_query_with_llm(cypher_query, error_msg)
            else:
                raise RuntimeError(f"查询生成失败: {error_msg}")

def repair_query_with_llm(bad_query: str, error: str) -> str:
    """用 LLM 修复错误的 Cypher 查询"""
    # ... 调用 LLM 修复逻辑
    # 输入：错误的查询 + 错误信息
    # 输出：修复后的查询
    pass
```

### 5.5 完整的 NL2Cypher 流程

```python
import json
from typing import Optional
from openai import OpenAI
from neo4j import GraphDatabase

class NL2CypherEngine:
    """自然语言转Cypher查询引擎"""
    
    def __init__(
        self,
        neo4j_uri: str,
        neo4j_user: str,
        neo4j_password: str,
        openai_api_key: str,
        model: str = "gpt-4o"
    ):
        self.llm_client = OpenAI(api_key=openai_api_key)
        self.neo4j_driver = GraphDatabase.driver(
            neo4j_uri, auth=(neo4j_user, neo4j_password)
        )
        self.model = model
        self.schema = self._extract_schema()
        self.example_pool = self._load_example_pool()
    
    def _extract_schema(self) -> dict:
        """从 Neo4j 数据库提取 Schema 信息"""
        with self.neo4j_driver.session() as session:
            # 获取节点标签
            labels_result = session.run(
                "CALL db.labels()"
            )
            labels = [record["label"] for record in labels_result]
            
            # 获取关系类型
            rels_result = session.run(
                "CALL db.relationshipTypes()"
            )
            rel_types = [record["relationshipType"] for record in rels_result]
            
            # 获取属性信息
            schema = {}
            for label in labels:
                prop_result = session.run(
                    f"""
                    MATCH (n:{label})
                    RETURN keys(n) AS props
                    LIMIT 100
                    """
                )
                props = set()
                for record in prop_result:
                    props.update(record["props"])
                schema[label] = list(props)
            
            return {
                "node_labels": labels,
                "relationship_types": rel_types,
                "node_properties": schema
            }
    
    def _load_example_pool(self) -> list[dict]:
        """加载预定义示例库"""
        return [
            {"question": "...", "cypher": "..."},
            # ...
        ]
    
    def query(self, natural_query: str) -> dict:
        """
        将自然语言转为Cypher查询并执行
        
        Args:
            natural_query: 用户自然语言查询
            
        Returns:
            {
                "cypher": 生成的Cypher语句,
                "results": 查询结果,
                "answer": LLM生成的最终回答
            }
        """
        # 1. 选择 Few-Shot 示例
        examples = self._select_examples(natural_query)
        
        # 2. 构建 Prompt
        prompt = self._build_prompt(natural_query, examples)
        
        # 3. LLM 生成 Cypher
        cypher = self._generate_cypher(prompt)
        
        # 4. 验证和执行
        try:
            results = self._validate_and_execute(cypher)
        except Exception as e:
            results = []
            cypher = f"-- 查询生成失败: {str(e)}"
        
        # 5. LLM 生成自然语言回答
        answer = self._generate_answer(natural_query, results)
        
        return {
            "cypher": cypher,
            "results": results,
            "answer": answer
        }
    
    def _build_prompt(self, query: str, examples: list[dict]) -> str:
        """构建 Schema-Aware Prompt"""
        schema_str = json.dumps(self.schema, indent=2, ensure_ascii=False)
        examples_str = "\n\n".join([
            f"用户：{ex['question']}\nCypher: {ex['cypher']}"
            for ex in examples
        ])
        
        return f"""
你是Neo4j Cypher专家。根据Schema和示例生成查询。

Schema:
{schema_str}

示例:
{examples_str}

用户：{query}
Cypher:
"""
    
    def _generate_cypher(self, prompt: str) -> str:
        """调用 LLM 生成 Cypher 查询"""
        resp = self.llm_client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.1  # 低温度确保一致性
        )
        return resp.choices[0].message.content.strip()
    
    def _validate_and_execute(self, cypher: str) -> list:
        """验证并执行查询"""
        with self.neo4j_driver.session() as session:
            # 先 EXPLAIN 验证
            session.run(f"EXPLAIN {cypher}").consume()
            result = session.run(cypher, timeout=30)
            return [record.data() for record in result]
    
    def _generate_answer(self, query: str, results: list) -> str:
        """基于查询结果生成自然语言回答"""
        resp = self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "基于查询结果回答用户问题，简洁准确"},
                {"role": "user", "content": f"问题: {query}\n\n查询结果: {json.dumps(results, ensure_ascii=False, indent=2)[:3000]}"}
            ],
            temperature=0.3
        )
        return resp.choices[0].message.content
    
    def _select_examples(self, query: str, k: int = 3) -> list[dict]:
        """选择最相似的示例（简化版本）"""
        # 实践中应使用 embedding 相似度
        return self.example_pool[:k]
    
    def close(self):
        self.neo4j_driver.close()
```

---

## 6. 性能优化

### 6.1 索引策略

不同类型的查询需要不同类型的索引：

#### 标签索引 (Label Index)

Neo4j 默认使用标签索引来定位起始节点。

```cypher
// 创建单属性索引
CREATE INDEX person_name_idx FOR (n:Person) ON (n.name)

// 创建复合索引
CREATE INDEX person_age_name_idx FOR (n:Person) ON (n.age, n.name)

// 创建唯一约束（自动创建索引）
CREATE CONSTRAINT unique_person_id FOR (n:Person) REQUIRE n.id IS UNIQUE
```

#### 全文索引 (Full-Text Index)

用于文本搜索和模糊匹配：

```cypher
// 创建全文索引
CREATE FULLTEXT INDEX movie_title_fulltext FOR (n:Movie) ON EACH [n.title]

// 使用全文索引查询
CALL db.index.fulltext.queryNodes('movie_title_fulltext', 'inception OR matrix')
YIELD node, score
RETURN node.title, score
ORDER BY score DESC
```

#### 向量索引 (Vector Index)

用于语义搜索和相似度匹配（Graph RAG 的关键）：

```cypher
// 创建向量索引（Neo4j 5.11+）
CREATE VECTOR INDEX entity_embeddings FOR (n:Entity) ON (n.embedding)
OPTIONS {indexConfig: {
  `vector.dimensions`: 1536,
  `vector.similarity_function`: 'cosine'
}}

// 向量相似度查询
MATCH (n:Entity)
WHERE n.embedding IS NOT NULL
WITH n, vector.similarity.cosine(n.embedding, $query_embedding) AS score
ORDER BY score DESC
LIMIT 10
RETURN n.name, n.description, score
```

### 6.2 查询计划分析

使用 `EXPLAIN` 和 `PROFILE` 来分析查询性能：

```cypher
// EXPLAIN: 查看执行计划（不实际执行）
EXPLAIN MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
WHERE p.name = 'Tom Hanks'
RETURN m.title

// PROFILE: 执行并统计信息
PROFILE MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
WHERE p.name = 'Tom Hanks'
RETURN m.title
```

执行计划示例解读：

```
计划树（PROFILE 输出）：
                    ┌─────────────┐
                    │  Result     │  2 ms
                    │  5 rows     │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │  Projection │  1 ms
                    │  m.title    │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │  Expand(All)│  8 ms    ← 最耗时的操作
                    │  (p)-[]->(m)│  5 rows  ← 扫描行数
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │  NodeIndex  │  2 ms    ← 使用索引快速定位
                    │  Seek       │          ← 关键优化点
                    │  :Person    │
                    │  name=$param│
                    └─────────────┘
```

常见性能瓶颈：
1. **全图扫描**：未使用索引，遍历所有节点
2. **无界变长路径**：`[*]` 无上限，可能指数爆炸
3. **笛卡尔积**：多个未关联的 MATCH 子句
4. **大量聚合**：对大数据集做 GROUP BY

### 6.3 分页与限制

```cypher
// 始终添加 LIMIT 防止返回过量数据
MATCH (n:Person)
RETURN n.name
LIMIT 100

// 分页（使用 SKIP 和 LIMIT）
MATCH (n:Person)
RETURN n.name
ORDER BY n.name
SKIP 50 LIMIT 50

// 高效分页（使用查询游标方式，避免 SKIP 的偏移量扫描）
MATCH (n:Person)
WHERE n.name > $last_seen_name  // 游标条件
RETURN n.name
ORDER BY n.name
LIMIT 50
```

### 6.4 查询超时控制

```python
# Neo4j Driver 超时设置
from neo4j import GraphDatabase

driver = GraphDatabase.driver(
    "bolt://localhost:7687",
    auth=("neo4j", "password"),
    connection_timeout=30,         # 连接超时
    max_connection_lifetime=3600   # 连接最大存活时间
)

# 事务级超时
with driver.session() as session:
    result = session.run(
        "MATCH (n)-[*1..5]->(m) RETURN n, m",
        timeout=10  # 查询执行超时（秒）
    )
```

---

## 7. 三种查询语言对比总结

### 7.1 综合对比表

| 维度 | Cypher | SPARQL | GraphQL |
|---|---|---|---|
| **图模型** | 属性图 (Property Graph) | RDF 三元组 | 自定义类型系统 |
| **标准制定者** | Neo4j (开源) | W3C 标准 | GraphQL 基金会 |
| **主要数据库** | Neo4j, Amazon Neptune*, Memgraph | Virtuoso, Jena, BlazeGraph, Neptune | Dgraph, Hasura, AWS AppSync |
| **查询本质** | 图模式匹配 | 三元组模式匹配 | 字段选择与递归 |
| **变长路径** | 原生支持 `[*n..m]` | 属性路径 `path+` | 无原生支持 |
| **模式匹配能力** | 强 (视觉化语法) | 强 (但语法冗长) | 中等 (需递归查询) |
| **推理支持** | 弱 (需 APOC) | 强 (RDFS/OWL) | 无 |
| **联邦查询** | 需 APOC | 原生 SERVICE 支持 | Schema Stitching |
| **写入能力** | 完整 (CRUD) | INSERT/DELETE | Mutation |
| **事务支持** | 完整 ACID | 有限 | 依赖底层 |
| **学习曲线** | 中 | 中高 | 低中 |
| **Graph RAG 适用性** | ⭐⭐⭐⭐⭐ 属性图首选 | ⭐⭐⭐⭐ 语义图首选 | ⭐⭐ API 封装层 |

> \* Amazon Neptune 同时支持属性图模式 (Gremlin) 和 RDF 模式 (SPARQL)

### 7.2 Graph RAG 场景中的选型建议

```
知识图谱数据模型 → 属性图 (Property Graph) → Cypher → Neo4j
                → RDF 图 (RDF Graph)        → SPARQL → Wikidata/Jena
                → 混合/API层                 → GraphQL → Dgraph/AppSync
```

**选型决策树**：

```
你的知识图谱是哪种模型？
├── 属性图 (Property Graph)
│   ├── 使用 Neo4j → 选择 Cypher ⭐
│   ├── 使用 Amazon Neptune → Gremlin 或 SPARQL
│   └── 使用 Memgraph → Cypher
│
├── RDF 图 (RDF Graph)
│   ├── 需要推理能力 → SPARQL ⭐
│   ├── 需要开放数据互联 → SPARQL ⭐
│   └── 主要做简单图遍历 → 考虑迁移到属性图
│
└── 不确定 / 从零开始
    ├── 大多数 Graph RAG 场景 → 属性图 + Cypher ⭐
    ├── 语义网/开放数据场景 → RDF + SPARQL
    └── 对外提供图 API → GraphQL 封装底层
```

### 7.3 综合建议

1. **对于大多数 Graph RAG 应用**：推荐使用 Neo4j + Cypher。Neo4j 生态成熟、工具链丰富（如 LangChain 的 GraphCypherQAChain），Cypher 语法直观，开发效率高。

2. **对于语义 Web 场景**：使用 SPARQL。如果数据来源是 Wikidata/DBpedia，SPARQL 是唯一选择。联邦查询能力让 SPARQL 在跨数据源场景中不可替代。

3. **对于 API 层**：如果需要对外提供图数据 API，考虑用 GraphQL 包装底层的 Cypher/SPARQL 查询，对外提供更友好的接口。

4. **NL2GraphQuery 实践要点**：
   - **Schema 信息是 Prompt 的核心** — 缺乏 Schema 信息时 LLM 的生成质量会大幅下降
   - **Few-Shot 示例动态选择** — 基于语义相似度选择示例比固定示例效果好得多
   - **验证 + 重试机制是必备的** — LLM 生成的查询不可能 100% 正确
   - **限制查询范围** — 始终在查询中加 LIMIT，防止 LLM 生成无界查询导致超时

---

## 8. 深入思考

### 8.1 查询语言会成为 Graph RAG 的瓶颈吗？

是的，在当前的 Graph RAG 架构中，NL2Query 环节的准确率是最大的瓶颈之一。即使在 GPT-4 级别模型上，Text-to-Cypher 的准确率也通常在 60-80% 之间。这意味着：

- 约 20-40% 的查询需要重试或回退
- 错误的查询可能返回错误结果但看起来合理（"幻觉传播"）
- 复杂多跳查询的准确率会进一步下降

**缓解策略**：
1. 退回到"关键词匹配 + 向量检索"混合模式
2. 预定义常用查询模板，LLM 只做参数填充
3. 查询结果验证（如检查返回字段与问题的一致性）

### 8.2 图查询语言的未来趋势

1. **SQL/GQL 统一**：GQL (Graph Query Language) 正在成为 ISO 标准，有望统一图查询语言
2. **自然语言接口**：直接使用自然语言查询图数据库，LLM 作为翻译层
3. **向量 + 图的混合查询**：将向量相似度搜索与图遍历结合（如 Neo4j 的向量索引联合查询）
4. **声明式到意图式**：从"如何查询"到"我要什么"，LLM 自动规划查询路径

### 8.3 关于 GraphQL 的误区

需要澄清一个常见误区：**GraphQL 的"Graph"不是指图数据库的"图"**。GraphQL 的名称来自其能够查询"关系图"（应用数据的关系网络），而非针对图数据库优化。虽然像 Dgraph 这样的图数据库支持 GraphQL，但 GraphQL 本质上是一个 API 层协议，不是图遍历语言。在 Graph RAG 中，GraphQL 的角色是 API 封装，底层仍然是 Cypher/SPARQL 执行真正的图遍历。

---

> **总结**：Cypher 和 SPARQL 是 Graph RAG 系统中连接 LLM 与知识图谱的"桥梁"。Cypher 适合属性图（Neo4j 生态），SPARQL 适合 RDF 语义图（开放数据生态）。NL2GraphQuery 是实现自然语言检索图知识的关键技术，核心挑战在于 Schema 理解、动态示例选择和查询验证纠错。在实际工程中，建议采用"LLM 生成 + 模板回退 + 混合检索"的多层策略来保证查询的可靠性和召回率。
