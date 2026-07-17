# 9.4.1 knowledge-graph-basics — 知识图谱基础：实体、关系、三元组

## 背景与问题

### 传统 RAG 的"文档盲区"

当用户提出这样的问题时：

> "阿尔伯特·爱因斯坦提出了哪些理论？这些理论中哪些在 21 世纪被实验验证？"

传统 Vector RAG 的处理方式是这样的：

1. 将问题转为向量，在文档库中搜索语义相似的 Chunk
2. 找到一篇介绍爱因斯坦理论的文章和一篇介绍 LIGO 实验的文章
3. LLM 尝试从这两段文本中推理答案

这个过程存在三个根本性问题：

```
问题一：文档孤岛
═══════════════════════════════════════
[文档 A: 爱因斯坦的理论]                [文档 B: 21世纪物理实验]
  ・狭义相对论 (1905)                     ・LIGO 探测引力波 (2015)
  ・广义相对论 (1915)                     ・ATLAS 发现 Higgs (2012)
  ・光电效应 (1905)                       ・Event Horizon 拍黑洞 (2019)
  ・质能方程 E=mc²                       ・IceCube 探测中微子 (2013)

  Chunk 之间的关联：无（独立的文本片段）
  向量检索能找到这些文档，但无法"连接"爱因斯坦的理论和 LIGO 实验

问题二：关系缺失
═══════════════════════════════════════
"光电效应"和"广义相对论"都是爱因斯坦提出的，
但传统 RAG 无法表达"提出"这个关系。
它只能告诉你"这些词出现在同一段文本中"。

问题三：实体模糊
═══════════════════════════════════════
"苹果"在文档中出现了：
  - "牛顿在苹果树下发现万有引力" → 水果
  - "苹果公司在 2023 年发布了 Vision Pro" → 科技公司
  - "他每天吃一个苹果" → 水果
传统 RAG 把所有这些"苹果"当作相同的 token 处理
```

### 知识图谱的核心洞见

知识图谱（Knowledge Graph）的核心想法极其简洁：**把世界建模为"事物"（实体）以及事物之间的"关系"**。

```
世界不是文档的集合，而是实体的网络。

  不是：[文档 A: "爱因斯坦提出相对论"] + [文档 B: "牛顿发现引力"]
  而是：
    爱因斯坦 ──提出──→ 相对论
    牛顿     ──发现──→ 万有引力
    爱因斯坦 ──影响──→ 现代物理学
    牛顿     ──影响──→ 经典力学
```

这种表示方式的威力在于：**一旦将知识从"文本"中解放出来变成"结构"，你就可以沿着关系链任意探索、推理、连接**。

---

## 核心概念

### 1. 实体（Entity）

实体是知识图谱中最基本的构建单元——代表真实世界中一个**可区分**的事物。

```
实体的特征：
  ┌─────────────────────────────────────────────────────┐
  │                                                       │
  │  必须有唯一标识（Identity）                              │
  │  例如：Q937（Wikidata 中爱因斯坦的 ID）                  │
  │                                                       │
  │  必须有类型（Type）                                     │
  │  例如：Person, Organization, Location, Event, Concept │
  │                                                       │
  │  可以有属性（Attribute/Property）                        │
  │  例如：出生日期、别名、描述                               │
  │                                                       │
  │  必须参与关系（Relation）                                │
  │  孤立实体没有意义——实体通过关系相互定义                    │
  │                                                       │
  └─────────────────────────────────────────────────────┘
```

实体类型示例：

| 实体类型 | 示例 | 典型属性 |
|---------|------|---------|
| Person（人物） | 爱因斯坦, 牛顿, 图灵 | name, birthDate, nationality |
| Organization（组织） | 谷歌, NASA, 普林斯顿 | name, foundedDate, headquarters |
| Location（地点） | 纽约, 月球, 瑞士 | name, coordinates, population |
| Event（事件） | 二战, COVID-19 大流行 | name, startDate, endDate |
| Concept（概念） | 相对论, 引力, 熵 | name, definition, field |
| Artifact（人工物） | iPhone, 蒙娜丽莎 | name, creator, creationDate |

### 2. 关系（Relation）

关系是连接两个实体的有向边，定义了实体之间的语义联系。

```
关系的基本形式：

    (实体 A) ────[关系]────▶ (实体 B)

    爱因斯坦 ──提出──→ 相对论
      ↑                    ↑
    主体(Subject)     客体(Object)
    关系的发出者      关系的接收者
```

常见关系类型：

| 关系 | 语义 | 示例 |
|------|------|------|
| `提出/发现` | 因果关系、创造关系 | 爱因斯坦 提出 相对论 |
| `位于` | 空间关系 | 普林斯顿 位于 新泽西 |
| `出生于` | 时间-空间关系 | 爱因斯坦 出生于 乌尔姆 |
| `受雇于` | 组织关系 | 爱因斯坦 受雇于 普林斯顿大学 |
| `属于` | 分类关系 | 相对论 属于 物理学 |
| `影响` | 抽象关系 | 相对论 影响 现代宇宙学 |
| `先于/后于` | 时序关系 | 狭义相对论 先于 广义相对论 |

**关键点**：关系是有向的，但并不意味着你可以只从一个方向理解。逆关系同样重要：

```
正向: 爱因斯坦 ──提出──→ 相对论
反向: 相对论 ──被提出──→ 爱因斯坦

正向: 谷歌 ──开发──→ Android
反向: Android ──被开发──→ 谷歌

查询时既可以用正向也可以用反向遍历，设计 Schema 时需要同时考虑两个方向。
```

### 3. 属性（Attribute）

属性是实体的特征描述，不同于关系——属性指向一个**字面量**（字符串、数字、日期），而不是另一个实体。

```
属性 vs 关系 的区别：

  关系：实体 → 实体                           属性：实体 → 字面量
  ─────────────────                            ─────────────────
  爱因斯坦 ──提出──→ 相对论                      爱因斯坦 ──姓名──→ "阿尔伯特·爱因斯坦"
  爱因斯坦 ──受雇于──→ 普林斯顿大学                爱因斯坦 ──出生日期──→ "1879-03-14"
  爱因斯坦 ──出生于──→ 乌尔姆                     爱因斯坦 ──身高──→ 175 (cm)

  属性存储字面量（Literal），关系存储对另一个实体的引用（Reference）
```

### 4. RDF 三元组

RDF（Resource Description Framework）是 W3C 制定的知识图谱数据模型标准。它的基本单元是**三元组（Triple）**：

```
结构: (Subject, Predicate, Object)
       (主体,     谓词,     客体)

主体（Subject）  → 一定是实体（用 URI 标识）
谓词（Predicate）→ 关系（用 URI 标识，表示关系类型）
客体（Object）   → 可以是实体（URI）或字面量（Literal）

示例：
  <http://example.org/AlbertEinstein> 
      <http://example.org/proposed> 
      <http://example.org/TheoryOfRelativity>

  <http://example.org/AlbertEinstein>
      <http://example.org/birthDate>
      "1879-03-14"^^xsd:date
```

RDF 的关键特性：

- **每个实体和关系都有全局唯一的 URI**——这是 RDF 能被 Web 规模共享的基础
- **三元组是最小的不可再分的知识单元**——一个三元组表达一个原子事实
- **所有三元组构成一个有向标签图**——Subject 和 Object 是节点，Predicate 是有向边标签
- **无 Schema 约束**——任何三元组都可以添加到图中（Open World Assumption）

### 5. RDF 序列化格式

RDF 有多种序列化方式，最常用的两种：

**Turtle 格式**（易读，适合手写）：

```turtle
@prefix ex: <http://example.org/> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

# 定义实体
ex:AlbertEinstein a ex:Person ;
    ex:name "阿尔伯特·爱因斯坦"@zh ;
    ex:birthDate "1879-03-14"^^xsd:date ;
    ex:birthPlace ex:Ulm ;
    ex:proposed ex:SpecialRelativity ,
                ex:GeneralRelativity ,
                ex:PhotoelectricEffect .

ex:Ulm a ex:City ;
    ex:name "乌尔姆"@zh ;
    ex:country ex:Germany .

ex:GeneralRelativity a ex:Theory ;
    ex:name "广义相对论"@zh ;
    ex:verifiedBy ex:LIGOExperiment .

ex:LIGOExperiment a ex:Experiment ;
    ex:name "LIGO 实验"@zh ;
    ex:discovery ex:GravitationalWave .
```

**JSON-LD 格式**（适合程序处理）：

```json
{
  "@context": {
    "ex": "http://example.org/",
    "xsd": "http://www.w3.org/2001/XMLSchema#"
  },
  "@id": "ex:AlbertEinstein",
  "@type": "ex:Person",
  "ex:name": "阿尔伯特·爱因斯坦",
  "ex:birthDate": {
    "@value": "1879-03-14",
    "@type": "xsd:date"
  },
  "ex:proposed": [
    {"@id": "ex:SpecialRelativity"},
    {"@id": "ex:GeneralRelativity"},
    {"@id": "ex:PhotoelectricEffect"}
  ]
}
```

### 6. 属性图模型（Property Graph）

与 RDF 不同，属性图模型（由 Neo4j 推广）采用更灵活的设计：

```
属性图的核心元素：
═══════════════════════════════════════════

  节点（Node）
    ├── 标签（Label）：分类标记，如 :Person, :Theory
    ├── 属性（Properties）：KV 对，如 {name: "爱因斯坦", birthDate: "1879-03-14"}
    └── 可以属于多个标签：如 :Person 和 :Scientist

  关系（Relationship）
    ├── 类型（Type）：如 PROPOSED, BORN_IN
    ├── 方向（Direction）：有向，从起点到终点
    ├── 属性（Properties）：关系和节点一样可以有属性
    │   如 {year: 1915, confidence: 0.95}
    └── 每个关系必须有起点和终点

  属性图示例（Neo4j Cypher 表示）：

  (albert:Person {name: "爱因斯坦", birthDate: "1879-03-14"})
      -[:PROPOSED {year: 1915}]→
      (relativity:Theory {name: "广义相对论"})

  (relativity)
      -[:VERIFIED_BY {year: 2016}]→
      (ligo:Experiment {name: "LIGO", year: 2015})
```

### 7. 本体（Ontology）

本体是知识图谱的 Schema——定义了有哪些实体类型、关系类型、属性的约束。

```
本体 vs 实例数据：
═══════════════════════════════════════════

  本体层（TBox — Terminology Box）              实例层（ABox — Assertion Box）
  ─────────────────────────────                  ──────────────────────────
  定义"有哪些类型"                              填充"具体有哪些实体"
  定义"类型之间的关系"                            填充"实体之间的具体关系"
  定义"属性约束"                                填充"实体的具体属性值"

  本体层示例：                                  实例层示例：
  Person ⊑ Entity                              Einstein :Person
  Theory ⊑ Entity                              Relativity :Theory
  Scientist ⊑ Person                           Newton :Person ∧ :Scientist
  proposes(Person, Theory)                     proposes(Einstein, Relativity)
  bornIn(Person, Location)                     bornIn(Einstein, Ulm)
```

本体设计是知识图谱构建中最关键的工程决策之一。一个设计良好的本体能显著提升下游查询的效果。

### 8. 有向标签图的数学定义

从图论的角度，知识图谱可以被严谨地定义为：

```
G = (V, E, L, φ)

其中：
  V  = 节点集合（实体，包括字面量）
  E  ⊆ V × V 边集合（关系）
  L  = 标签集合（类型标签）
  φ  = 标签函数：V ∪ E → L
      将每个节点映射到其类型标签
      将每条边映射到其关系类型

约束：
  每条边 (u, v) ∈ E 表示一个有向关系
  节点和边可以有多个标签（多标签图）
  标签可以带属性（KV 对）

示例中的子图：
  V = {爱因斯坦, 相对论, LIGO, 引力波}
  E = {(爱因斯坦, 相对论), (相对论, 引力波), (LIGO, 引力波)}
  L = {Person, Theory, Experiment, Concept, proposed, predicts, observed}
  φ(爱因斯坦) = Person
  φ(爱因斯坦, 相对论) = proposed
  φ(相对论, 引力波) = predicts
  φ(LIGO, 引力波) = observed
```

---

## RDF vs Property Graph 对比

RDF 和 Property Graph 虽然都是图模型，但设计哲学和使用场景有显著差异：

| 对比维度 | RDF (Resource Description Framework) | Property Graph (属性图) |
|---------|--------------------------------------|----------------------|
| **数据模型** | 三元组 (Subject-Predicate-Object) | 节点-关系-属性 |
| **标识方式** | 全局 URI（唯一标识每个资源） | 局部 ID（数据库内部 ID） |
| **Schema** | 可选（RDFS/OWL），开放世界假设 | 可选（有些 DB 如 Neo4j 无强制 Schema） |
| **属性挂载** | 属性本质上是三元组（字面量作为 Object） | 节点和关系直接有属性字典 |
| **查询语言** | SPARQL | Cypher（Neo4j）, Gremlin, GraphQL |
| **标准化** | W3C 标准，Web 互操作性强 | 厂商驱动，互操作性弱 |
| **推理支持** | 原生支持（RDFS/OWL 推理器） | 需要外部推理引擎 |
| **表达能力** | OWL 2 提供丰富的公理（等价、互斥、传递等） | 主要靠应用层实现逻辑 |
| **性能** | 三元组存储，JOIN 多 | 原生图存储，遍历快 |
| **生态** | 学术界、开放数据（Wikidata, DBpedia） | 工业界（Neo4j, Amazon Neptune） |
| **适合场景** | 数据集成、开放链接数据、推理密集型 | 事务型应用、实时图遍历、企业级应用 |

### 查询语言对比

**SPARQL 查询 RDF**：

```sparql
# 查询：爱因斯坦提出了哪些理论？
PREFIX ex: <http://example.org/>

SELECT ?theoryName WHERE {
  ex:AlbertEinstein ex:proposed ?theory .
  ?theory ex:name ?theoryName .
}

# 结果：
# ┌─────────────────────┐
# │ theoryName           │
# ├─────────────────────┤
# │ "狭义相对论"        │
# │ "广义相对论"        │
# │ "光电效应"          │
# └─────────────────────┘

# 更复杂的查询：爱因斯坦提出的，在21世纪被验证的理论
PREFIX ex: <http://example.org/>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

SELECT ?theoryName ?experiment ?year WHERE {
  ex:AlbertEinstein ex:proposed ?theory .
  ?theory ex:name ?theoryName .
  ?experiment ex:verifiedTheories ?theory .
  ?experiment ex:year ?year .
  FILTER(?year >= 2000)
}
```

**Cypher 查询 Property Graph**：

```cypher
// 查询：爱因斯坦提出了哪些理论？
MATCH (albert:Person {name: "爱因斯坦"})-[r:PROPOSED]->(theory:Theory)
RETURN theory.name AS theoryName

// 结果：
// ┌─────────────────────┐
// │ theoryName           │
// ├─────────────────────┤
// │ "狭义相对论"        │
// │ "广义相对论"        │
// │ "光电效应"          │
// └─────────────────────┘

// 更复杂的查询：爱因斯坦提出的，在21世纪被验证的理论
MATCH (albert:Person {name: "爱因斯坦"})-[p:PROPOSED]->(theory:Theory)
MATCH (experiment:Experiment)-[v:VERIFIED]->(theory)
WHERE v.year >= 2000
RETURN theory.name AS theoryName, 
       experiment.name AS experiment, 
       v.year AS year
```

**能力对比**：

| 查询能力 | SPARQL | Cypher |
|---------|--------|--------|
| 基本图模式匹配 | ✅ MATCH？ | ✅ MATCH |
| 属性过滤 | ✅ FILTER | ✅ WHERE |
| 可选匹配 | ✅ OPTIONAL | ✅ OPTIONAL MATCH |
| 路径查询 | ✅ 正则路径语法 | ✅ 变长路径 |
| 聚合 | ✅ GROUP BY/HAVING | ✅ WITH/UNWIND |
| 子查询 | ✅ Sub-SELECT | ✅ CALL Subquery |
| 更新 | ✅ INSERT/DELETE | ✅ CREATE/MERGE/DELETE |
| 联邦查询 | ✅ SERVICE 关键字 | ❌ 不支持原生 |
| OWL 推理 | ✅ 直接支持 | ❌ 需额外工具 |

---

## 主流知识图谱介绍

### 开放知识图谱

| 名称 | 规模 | 特点 | 查询方式 |
|------|------|------|---------|
| **Wikidata** | ~1.1 亿实体, ~13 亿声明 | 社区维护、多语言、CC0 协议 | SPARQL endpoint, API |
| **DBpedia** | ~500 万实体 | 从 Wikipedia 自动抽取 | SPARQL endpoint |
| **YAGO** | ~1000 万实体 | 结合 Wikipedia + WordNet, 高精度 | SPARQL |
| **ConceptNet** | ~800 万节点 | 常识知识图谱 | API, JSON |
| **Google Knowledge Graph** | ~70 亿实体 | 最大的商业 KG，不公开 | Knowledge Graph API |

### 领域知识图谱

| 领域 | 图谱名称 | 用途 |
|------|---------|------|
| 生物医学 | Gene Ontology (GO) | 基因功能标准化描述 |
| 医学 | SNOMED-CT | 临床术语标准化 |
| 药物 | DrugBank | 药物-靶点-疾病关系 |
| 蛋白质 | UniProt | 蛋白质序列和功能知识 |
| 化学 | PubChem | 化合物和化学反应 |

### 企业知识图谱

| 产品 | 类型 | 特性 |
|------|------|------|
| Neo4j | 图数据库 | 属性图模型，Cypher 语言，成熟生态 |
| Amazon Neptune | 托管图数据库 | 支持 RDF + Property Graph 双模式 |
| ArangoDB | 多模型数据库 | 图 + 文档 + KV 三合一 |
| JanusGraph | 分布式图数据库 | Hadoop 生态，适合超大规模 |
| TigerGraph | 图分析平台 | 高性能图分析，内置并行处理 |

---

## 核心挑战

### 1. 知识图谱的不完整性（Open World Assumption）

知识图谱本质上是**永远不完整的**。在 RDF/OWL 的世界中这被称为开放式世界假设（OWA）：

> 如果 KG 中不存在某个事实，不代表该事实为假，只代表"没有记录"。

```
开放世界 vs 封闭世界：
═══════════════════════════════════════════

KG 中已知: 爱因斯坦出生于乌尔姆
KG 中不存在: 爱因斯坦的国籍

  封闭世界假设（数据库思维）          开放世界假设（KG 思维）
  ──────────────────────             ─────────────────────
  国籍字段为 NULL                   KG 中没有国籍信息
  → 爱因斯坦没有国籍                  → 国籍未知，不是没有
  → 查询返回空结果                    → 查询需要处理不确定性
  → 系统表现为"假"                    → 系统表现为"不知道"

这对 RAG 的影响：
  - 不能因为 KG 没有某条事实就断言该事实不存在
  - KG 检索的结果需要标注"不完全"——"根据现有知识库，未发现..."
  - 需要设计"未知"的 fallback 策略（降级到向量检索或 LLM 自身知识）
```

### 2. 实体对齐与消歧

同一实体在不同数据源中可能以不同的名称或 ID 出现。实体对齐（Entity Alignment）是将这些不同的表示对应到同一实体的过程：

```
对齐挑战：
═══════════════════════════════════════════

数据源 A:                              数据源 B:
  "Albert Einstein"                    "A. Einstein"
  born in "Ulm"                        born in "Ulm, Germany"
  worked at "ETH Zurich"               employed by "ETH Zuerich"
  proposed "General Relativity"        "General Theory of Relativity"

对齐目标：识别出这些是同一套实体

常见对齐挑战：
  ┌─────────────────────────────────────────────┐
  │                                             │
  │  1. 名称变体：Einstein vs A. Einstein       │
  │  2. 语言差异：爱因斯坦 vs Albert Einstein   │
  │  3. 缩写：IBM vs International Business     │
  │             Machines Corporation            │
  │  4. 粒度差异："纽约" vs "纽约市" vs "NY"   │
  │  5. 同音异议："苹果"（水果 vs 公司）        │
  │  6. 时间变化：IBM 曾用名 Computing-         │
  │             Tabulating-Recording Company    │
  │                                             │
  └─────────────────────────────────────────────┘

常用方法：
  - 基于字符串相似度（Levenshtein, Jaccard）
  - 基于向量嵌入（同一实体在向量空间中接近）
  - 基于图结构（对齐实体的邻居结构相似）
  - LLM-based（利用 LLM 的常识推理能力）
```

### 3. 图谱稀疏性问题

知识图谱通常是高度稀疏的——绝大多数实体只有极少数关系：

```
稀疏性模式：
═══════════════════════════════════════════

幂律分布（Power Law）：
  20% 的核心实体拥有 80% 的关系
  80% 的长尾实体只有 1-3 条关系

节点度数分布：
  ┌─────────────────────────────────────────────────┐
  │ 度数         │ 实体比例 │ 典型实体              │
  ├─────────────────────────────────────────────────┤
  │ 1000+        │   <0.1%  │ 维基百科/超级明星实体 │
  │ 100-999      │    1%    │ 重要人物/概念          │
  │ 10-99        │    9%    │ 普通实体              │
  │ 2-9          │   30%    │ 大多数实体            │
  │ 1            │   40%    │ 孤立实体              │
  │ 0            │   20%    │ 未连接节点            │
  └─────────────────────────────────────────────────┘

对检索的影响：
  - 长尾实体信息不足，子图展开后上下文贫瘠
  - 核心实体邻居过多，子图爆炸（爱因斯坦有上千条关系）
  - 需要智能剪枝策略：只选择与查询最相关的 Top-K 条关系
  - 稀疏区域需要融合向量检索来补充信息
```

### 4. 动态更新与时效性

知识是不断变化的，KG 需要处理：

```
更新层级与策略：
═══════════════════════════════════════════

层级一：属性更新（最常见）
  ┌ 实体的属性值变化（如：CEO 变更、年龄增长）
  └ 方案：直接覆盖或版本化（保留历史值）

层级二：关系更新（较频繁）
  ┌ 新增或删除实体间关系（如：公司收购事件）
  └ 方案：增量添加边，不影响已有图结构

层级三：实体新增（中等频率）
  ┌ 新实体进入知识库（如：新产品发布）
  └ 方案：实体匹配 → 消歧 → 链接到现有实体

层级四：Schema 变更（低频高影响）
  ┌ 新增/删除/重命名实体类型或关系类型
  └ 方案：重写整个 KG 或做模式迁移

时效性挑战：
  - 实体抽取模型基于历史训练数据，无法识别最新实体
  - LLM 的 NER 能力受限于上下文窗口和训练截止日期
  - 已建立的关系可能随时间失效（"前任CEO"和"CEO"混淆）
  - 需要引入时间戳（Temporal KG），每条三元组标注有效时间
```

---

## Python 示例

### 示例 1：用 NetworkX 构建简单知识图谱

```python
"""
使用 NetworkX 构建和查询一个简单的学术知识图谱
展示：实体创建、关系添加、子图查询、路径搜索
"""

import networkx as nx
import matplotlib.pyplot as plt
from typing import Any, Dict, List, Tuple

# ============================================================
# 1. 创建空图
# ============================================================
KG = nx.MultiDiGraph()  # 使用有向多重图（支持两个节点间多条边）

# ============================================================
# 2. 添加节点（实体）
# ============================================================
# 节点 = 实体，节点属性 = 实体属性
entities = [
    ("AlbertEinstein", {
        "type": "Person",
        "name": "阿尔伯特·爱因斯坦",
        "name_en": "Albert Einstein",
        "birthDate": "1879-03-14",
        "deathDate": "1955-04-18",
        "nationality": "德国/瑞士/美国"
    }),
    ("IsaacNewton", {
        "type": "Person",
        "name": "艾萨克·牛顿",
        "name_en": "Isaac Newton",
        "birthDate": "1643-01-04",
        "deathDate": "1727-03-31",
        "nationality": "英国"
    }),
    ("SpecialRelativity", {
        "type": "Theory",
        "name": "狭义相对论",
        "name_en": "Special Relativity",
        "year": 1905,
        "field": "物理学"
    }),
    ("GeneralRelativity", {
        "type": "Theory",
        "name": "广义相对论",
        "name_en": "General Relativity",
        "year": 1915,
        "field": "物理学"
    }),
    ("PhotoelectricEffect", {
        "type": "Theory",
        "name": "光电效应",
        "name_en": "Photoelectric Effect",
        "year": 1905,
        "field": "量子力学"
    }),
    ("ClassicalMechanics", {
        "type": "Theory",
        "name": "经典力学",
        "name_en": "Classical Mechanics",
        "year": 1687,
        "field": "物理学"
    }),
    ("LIGO", {
        "type": "Experiment",
        "name": "LIGO 实验",
        "name_en": "LIGO Experiment",
        "year": 2015,
        "field": "引力波天文学"
    }),
    ("GravitationalWave", {
        "type": "Concept",
        "name": "引力波",
        "name_en": "Gravitational Wave",
        "description": "时空的涟漪"
    }),
    ("NobelPrize", {
        "type": "Award",
        "name": "诺贝尔物理学奖",
        "year": 1921
    }),
    ("MaxPlanck", {
        "type": "Person",
        "name": "马克斯·普朗克",
        "name_en": "Max Planck",
        "nationality": "德国"
    }),
]

for entity_id, attrs in entities:
    KG.add_node(entity_id, **attrs)

# ============================================================
# 3. 添加边（关系）
# ============================================================
# 边 = 关系，边属性 = 关系属性
relations = [
    ("AlbertEinstein", "proposed", "SpecialRelativity", {"year": 1905}),
    ("AlbertEinstein", "proposed", "GeneralRelativity", {"year": 1915}),
    ("AlbertEinstein", "proposed", "PhotoelectricEffect", {"year": 1905}),
    ("AlbertEinstein", "won", "NobelPrize", {"year": 1922}),
    ("IsaacNewton", "proposed", "ClassicalMechanics", {"year": 1687}),
    ("GeneralRelativity", "predicts", "GravitationalWave", {}),
    ("LIGO", "verified", "GeneralRelativity", {"year": 2016}),
    ("LIGO", "observed", "GravitationalWave", {"year": 2015}),
    ("MaxPlanck", "influenced", "AlbertEinstein", {"field": "量子理论"}),
    ("AlbertEinstein", "influenced", "ModernPhysics_placeholder", {}),
    ("ClassicalMechanics", "influenced", "SpecialRelativity", {}),
]

for src, rel, tgt, attrs in relations:
    if src in KG and tgt in KG:  # 确保两个实体都存在
        KG.add_edge(src, tgt, key=rel, relation=rel, **attrs)
    else:
        print(f"跳过关系 {src} -[{rel}]-> {tgt}：实体不存在")

# ============================================================
# 4. 基本查询操作
# ============================================================

def get_entity_info(graph: nx.MultiDiGraph, entity_id: str) -> Dict:
    """获取实体的基本信息和所有关系"""
    if entity_id not in graph:
        return {"error": f"实体 {entity_id} 不存在"}

    info = dict(graph.nodes[entity_id])

    # 收集出边（作为 Subject 的关系）
    out_relations = []
    for _, tgt, key, data in graph.out_edges(entity_id, data=True, keys=True):
        out_relations.append({
            "relation": data.get("relation", key),
            "target": tgt,
            "target_name": graph.nodes[tgt].get("name", tgt),
            "target_type": graph.nodes[tgt].get("type", "Unknown"),
            "attributes": {k: v for k, v in data.items() if k != "relation"}
        })
    info["out_relations"] = out_relations

    # 收集入边（作为 Object 的关系）
    in_relations = []
    for src, _, key, data in graph.in_edges(entity_id, data=True, keys=True):
        in_relations.append({
            "relation": data.get("relation", key),
            "source": src,
            "source_name": graph.nodes[src].get("name", src),
            "source_type": graph.nodes[src].get("type", "Unknown"),
            "attributes": {k: v for k, v in data.items() if k != "relation"}
        })
    info["in_relations"] = in_relations

    return info


def multi_hop_query(
    graph: nx.MultiDiGraph,
    start_entity: str,
    max_hops: int = 2,
    relation_filter: str = None
) -> List[List[Tuple]]:
    """
    多跳查询：从起始实体出发，探索 max_hops 步以内的路径

    返回路径列表，每条路径是一系列 (实体, 关系, 实体) 三元组
    """
    paths = []
    visited = set()

    def dfs(current: str, path: List[Tuple], depth: int):
        if depth > max_hops:
            return
        visited.add(current)

        for _, neighbor, key, data in graph.out_edges(current, data=True, keys=True):
            rel = data.get("relation", key)
            if relation_filter and rel != relation_filter:
                continue
            new_path = path + [(current, rel, neighbor)]
            paths.append(new_path)
            if neighbor not in visited:
                dfs(neighbor, new_path, depth + 1)

        visited.remove(current)

    dfs(start_entity, [], 0)
    return paths


# ============================================================
# 5. 实际查询演示
# ============================================================

print("=" * 60)
print("知识图谱查询演示")
print("=" * 60)

# 5.1 查询单个实体的信息
print("\n[查询] 爱因斯坦的信息：")
info = get_entity_info(KG, "AlbertEinstein")
if "error" not in info:
    print(f"  名称: {info.get('name')} / {info.get('name_en')}")
    print(f"  国籍: {info.get('nationality')}")
    print(f"  出生: {info.get('birthDate')}")
    print(f"  出边关系 ({len(info['out_relations'])} 条):")
    for r in info["out_relations"]:
        attr_str = ", ".join(f"{k}={v}" for k, v in r["attributes"].items())
        print(f"    ──[{r['relation']}]──▶ {r['target_name']} ({r['target_type']}) [{attr_str}]")
    print(f"  入边关系 ({len(info['in_relations'])} 条):")
    for r in info["in_relations"]:
        attr_str = ", ".join(f"{k}={v}" for k, v in r["attributes"].items())
        print(f"    ◀──[{r['relation']}]── {r['source_name']} ({r['source_type']}) [{attr_str}]")

# 5.2 多跳推理查询
print("\n[多跳查询] 从爱因斯坦出发，2 跳内的所有路径：")
paths = multi_hop_query(KG, "AlbertEinstein", max_hops=2)
for path in paths:
    path_str = " → ".join(
        f"{KG.nodes[step[0]].get('name', step[0])} --[{step[1]}]--> "
        f"{KG.nodes[step[2]].get('name', step[2])}"
        for step in [path[-1]]  # 只显示最后一步
        if len(path) == 1
    )
    if not path_str:  # 多跳路径
        parts = []
        for step in path:
            src_name = KG.nodes[step[0]].get('name', step[0])
            rel = step[1]
            tgt_name = KG.nodes[step[2]].get('name', step[2])
            parts.append(f"{src_name} --[{rel}]--> {tgt_name}")
        path_str = " → ".join(parts)
    print(f"  {path_str}")

# 5.3 特定问题的推理路径
print("\n[推理查询] '爱因斯坦的理论被什么实验验证？'")
# 路径：爱因斯坦 → 提出 → 广义相对论 → 被验证 → LIGO
results = multi_hop_query(KG, "AlbertEinstein", max_hops=3)
# 筛选出经过 proposed → verified 的路径
for path in results:
    if len(path) >= 2:
        rels = [step[1] for step in path]
        if "proposed" in rels and "verified" in rels:
            print(f"  推理路径:")
            for i, step in enumerate(path):
                src_name = KG.nodes[step[0]].get('name', step[0])
                rel = step[1]
                tgt_name = KG.nodes[step[2]].get('name', step[2])
                arrow = "──▶" if i < len(path) - 1 else ""
                print(f"    {src_name} ──[{rel}]──▶ {tgt_name}")

# 5.4 图的基本统计信息
print("\n[图谱统计]")
print(f"  实体数（节点）: {KG.number_of_nodes()}")
print(f"  关系数（边）: {KG.number_of_edges()}")
print(f"  实体类型分布:")
type_dist = {}
for _, data in KG.nodes(data=True):
    t = data.get("type", "Unknown")
    type_dist[t] = type_dist.get(t, 0) + 1
for t, count in sorted(type_dist.items(), key=lambda x: -x[1]):
    print(f"    {t}: {count}")

# ============================================================
# 6. 可视化图结构（控制台 ASCII）
# ============================================================
print("\n[图谱结构示意]")
print("""
                    ┌─────────────────┐
                    │   马克斯·普朗克   │
                    └────────┬────────┘
                             │ influenced
                             ▼
              ┌─────────────────────────┐
              │   阿尔伯特·爱因斯坦     │
              │   (Person)              │
              └────┬──────┬──────┬──────┘
                   │      │      │
         proposed  │      │      │  won
                   │      │      │
    ┌──────────────┼──────┼──────┼──────────────┐
    ▼              ▼      ▼      ▼              ▼
┌──────────┐ ┌──────────┐ ┌──────────────┐ ┌──────────┐
│狭义相对论│ │广义相对论│ │  光电效应    │ │诺贝尔奖  │
│(Theory)  │ │(Theory)  │ │ (Theory)     │ │(Award)   │
└──────────┘ └────┬─────┘ └──────────────┘ └──────────┘
                  │ predicts
                  ▼
           ┌──────────────┐
           │   引力波      │
           │  (Concept)    │
           └──────┬───────┘
                  │ observed
                  ▼
           ┌──────────────┐
           │ LIGO 实验    │
           │ (Experiment) │
           └──────────────┘
                  │ verified
                  ▼
           ┌──────────────┐
           │ 广义相对论    │
           └──────────────┘
""")
```

### 示例 2：Neo4j Cypher 创建和查询

```python
"""
使用 Neo4j 的 Cypher 查询语言创建和查询知识图谱
注意：需要安装 neo4j Python 驱动，并运行 Neo4j 实例

pip install neo4j
"""

from neo4j import GraphDatabase
from typing import Dict, List, Optional

# ============================================================
# Neo4j 数据库连接
# ============================================================
# 默认连接信息（Neo4j Desktop 或 AuraDB）
URI = "bolt://localhost:7687"
USER = "neo4j"
PASSWORD = "password"  # 替换为实际的密码


class KnowledgeGraphDB:
    """封装 Neo4j 操作的知识图谱数据库客户端"""

    def __init__(self, uri: str, user: str, password: str):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))

    def close(self):
        self.driver.close()

    # ----------------------------------------------------------
    # Schema 定义：创建约束和索引
    # ----------------------------------------------------------
    def create_schema(self):
        """创建实体类型约束和索引（性能优化）"""
        with self.driver.session() as session:
            # 确保 Person 节点的 name 属性唯一
            session.run(
                "CREATE CONSTRAINT IF NOT EXISTS FOR (p:Person) REQUIRE p.name IS UNIQUE"
            )
            session.run(
                "CREATE CONSTRAINT IF NOT EXISTS FOR (t:Theory) REQUIRE t.name IS UNIQUE"
            )
            session.run(
                "CREATE CONSTRAINT IF NOT EXISTS FOR (e:Experiment) REQUIRE e.name IS UNIQUE"
            )
            # 创建索引加速查询
            session.run("CREATE INDEX IF NOT EXISTS FOR (p:Person) ON (p.name)")
            session.run("CREATE INDEX IF NOT EXISTS FOR (t:Theory) ON (t.name)")
            print("Schema 创建完成")

    # ----------------------------------------------------------
    # 数据插入：创建实体和关系
    # ----------------------------------------------------------
    def create_person(self, name: str, name_cn: str,
                      birth_date: str, nationality: str):
        """创建一个 Person 实体"""
        with self.driver.session() as session:
            result = session.run(
                """
                MERGE (p:Person {name: $name})
                SET p.name_cn = $name_cn,
                    p.birth_date = $birth_date,
                    p.nationality = $nationality
                RETURN p
                """,
                name=name, name_cn=name_cn,
                birth_date=birth_date, nationality=nationality
            )
            return result.single()

    def propose_theory(self, person_name: str,
                       theory_name: str, theory_name_cn: str,
                       year: int, field: str):
        """创建 Person ──提出──→ Theory 关系"""
        with self.driver.session() as session:
            # MERGE 确保不重复创建相同实体
            result = session.run(
                """
                MERGE (t:Theory {name: $theory_name})
                SET t.name_cn = $theory_name_cn,
                    t.year = $year,
                    t.field = $field
                WITH t
                MATCH (p:Person {name: $person_name})
                MERGE (p)-[r:PROPOSED {year: $year}]->(t)
                RETURN p, r, t
                """,
                person_name=person_name,
                theory_name=theory_name,
                theory_name_cn=theory_name_cn,
                year=year,
                field=field
            )
            return result.single()

    def add_experiment_verification(
        self, experiment_name: str, experiment_name_cn: str,
        experiment_year: int, theory_name: str, verification_year: int
    ):
        """创建 Experiment ──验证──→ Theory 关系"""
        with self.driver.session() as session:
            result = session.run(
                """
                MERGE (e:Experiment {name: $experiment_name})
                SET e.name_cn = $experiment_name_cn,
                    e.year = $experiment_year
                WITH e
                MATCH (t:Theory {name: $theory_name})
                MERGE (e)-[r:VERIFIED {year: $verification_year}]->(t)
                RETURN e, r, t
                """,
                experiment_name=experiment_name,
                experiment_name_cn=experiment_name_cn,
                experiment_year=experiment_year,
                theory_name=theory_name,
                verification_year=verification_year
            )
            return result.single()

    # ----------------------------------------------------------
    # 查询：多跳推理
    # ----------------------------------------------------------
    def query_theories_verified(self, person_name: str):
        """
        查询某位科学家提出的、后来被实验验证的理论

        Cypher 路径：
        (Person) -[:PROPOSED]-> (Theory) <-[:VERIFIED]- (Experiment)
        """
        with self.driver.session() as session:
            result = session.run(
                """
                MATCH (p:Person {name: $person_name})
                      -[prop:PROPOSED]->(t:Theory)
                MATCH (e:Experiment)-[ver:VERIFIED]->(t)
                RETURN p.name AS scientist,
                       t.name AS theory,
                       t.name_cn AS theory_cn,
                       prop.year AS proposed_year,
                       e.name AS experiment,
                       ver.year AS verified_year
                ORDER BY ver.year DESC
                """,
                person_name=person_name
            )
            return [record.data() for record in result]

    def query_influence_chain(self, person_name: str):
        """
        查询科学影响链：某人影响了谁，这些人又影响了谁

        Cypher 变长路径匹配：
        (Person) -[:INFLUENCED*1..2]-> (Person)
        """
        with self.driver.session() as session:
            result = session.run(
                """
                MATCH path = (p:Person {name: $person_name})
                             -[:INFLUENCED*1..2]->(other:Person)
                RETURN [node IN nodes(path) | node.name] AS chain,
                       length(path) AS hop_count
                """,
                person_name=person_name
            )
            return [record.data() for record in result]

    def query_entity_network(self, entity_name: str, max_depth: int = 2):
        """
        返回实体周围的完整网络（包含所有实体类型）

        用于 RAG 检索：获取实体为中心的局部子图
        """
        with self.driver.session() as session:
            # 使用 Cypher 的可变长路径匹配
            result = session.run(
                """
                MATCH path = (start)
                    -[r*1..$max_depth]-(connected)
                WHERE start.name = $entity_name
                RETURN [node IN nodes(path) | 
                        {id: node.name, 
                         type: labels(node)[0], 
                         props: properties(node)}] AS entities,
                       [rel IN relationships(path) | 
                        {type: type(rel), 
                         props: properties(rel)}] AS relations,
                       length(path) AS depth
                LIMIT 50
                """,
                entity_name=entity_name,
                max_depth=max_depth
            )
            return [record.data() for record in result]

    # ----------------------------------------------------------
    # 批量插入示例
    # ----------------------------------------------------------
    def seed_data(self):
        """插入示例数据"""
        # 创建科学家
        self.create_person(
            "Albert Einstein", "阿尔伯特·爱因斯坦",
            "1879-03-14", "德国/瑞士/美国"
        )
        self.create_person(
            "Isaac Newton", "艾萨克·牛顿",
            "1643-01-04", "英国"
        )

        # 添加提出的理论
        self.propose_theory(
            "Albert Einstein", "Special Relativity", "狭义相对论",
            1905, "物理学"
        )
        self.propose_theory(
            "Albert Einstein", "General Relativity", "广义相对论",
            1915, "物理学"
        )
        self.propose_theory(
            "Albert Einstein", "Photoelectric Effect", "光电效应",
            1905, "量子力学"
        )

        # 添加验证实验
        self.add_experiment_verification(
            "LIGO", "LIGO激光干涉引力波天文台", 2015,
            "General Relativity", 2016
        )

        print("示例数据插入完成")


# ============================================================
# 使用示例
# ============================================================
if __name__ == "__main__":
    print("=" * 60)
    print("Neo4j 知识图谱操作演示")
    print("=" * 60)

    try:
        db = KnowledgeGraphDB(URI, USER, PASSWORD)

        # 初始化 Schema
        db.create_schema()

        # 插入种子数据
        db.seed_data()

        # 执行多跳查询
        print("\n[查询] 爱因斯坦提出的、被实验验证的理论：")
        results = db.query_theories_verified("Albert Einstein")
        for r in results:
            print(f"  {r['scientist']} 在 {r['proposed_year']} 年提出"
                  f" {r['theory']} ({r['theory_cn']})")
            print(f"    → 由 {r['experiment']} 在 {r['verified_year']} 年验证")

        # 查询实体网络
        print("\n[查询] 爱因斯坦的实体网络：")
        network = db.query_entity_network("Albert Einstein", max_depth=2)
        for item in network[:3]:  # 显示前 3 条路径
            entities = [e["id"] for e in item["entities"]]
            rels = [r["type"] for r in item["relations"]]
            print(f"  路径 (深度 {item['depth']}): {' -- '.join(entities)}")
            print(f"  关系: {' -- '.join(rels)}")

    except Exception as e:
        print(f"\n数据库连接失败: {e}")
        print("这是预期的——没有运行中的 Neo4j 实例。")
        print("\nCypher 查询可直接在 Neo4j Browser 或 AuraDB 中执行。")
    finally:
        if 'db' in locals():
            db.close()

    # 打印 Cypher 查询供参考
    print("\n\n" + "=" * 60)
    print("关键 Cypher 查询（可直接在 Neo4j Browser 中运行）")
    print("=" * 60)
    print("""
    -- 1. 查询爱因斯坦提出的所有理论 --
    MATCH (p:Person {name: 'Albert Einstein'})-[r:PROPOSED]->(t:Theory)
    RETURN p.name, t.name, r.year

    -- 2. 多跳查询：理论 → 验证实验 --
    MATCH (p:Person {name: 'Albert Einstein'})
          -[prop:PROPOSED]->(t:Theory)
    OPTIONAL MATCH (e:Experiment)-[ver:VERIFIED]->(t)
    RETURN p.name AS Scientist,
           t.name AS Theory,
           t.year AS ProposedYear,
           COLLECT(DISTINCT e.name) AS VerifiedBy

    -- 3. 获取实体为中心的局部子图 --
    MATCH path = (start {name: 'Albert Einstein'})-[r*1..2]-(connected)
    RETURN path
    LIMIT 50

    -- 4. 社区检测：按研究领域聚类 --
    MATCH (t:Theory)
    RETURN t.field AS Field, COUNT(t) AS TheoryCount, COLLECT(t.name) AS Theories
    """)
```

---

## 知识图谱在 RAG 中的角色

### 核心流程

```
                    用户查询："爱因斯坦的理论被什么验证？"

                            │
                    ┌───────▼───────┐
                    │   实体识别     │
                    │  (Entity       │
                    │   Recognition)  │
                    └───────┬───────┘
                            │ 提取出: "爱因斯坦"
                            │
                    ┌───────▼───────┐
                    │   实体链接     │
                    │  (Entity       │
                    │   Linking)     │
                    └───────┬───────┘
                            │ 匹配到 KG 实体: "AlbertEinstein"
                            │
                    ┌───────▼───────┐
                    │   子图检索     │
                    │  (Subgraph     │
                    │   Retrieval)   │
                    └───────┬───────┘
                            │ 扩展 1-2 跳: 爱因斯坦→提出→广义相对论
                            │                       ←验证←LIGO
                            │
                    ┌───────▼───────┐
                    │   序列化       │
                    │  (Serialization)│
                    │                │
                    │  三元组 → 文本  │
                    └───────┬───────┘
                            │ LLM 上下文:
                            │ "爱因斯坦提出了广义相对论(1915)。
                            │  LIGO实验在2016年验证了广义相对论。"
                            │
                    ┌───────▼───────┐
                    │   LLM 生成     │
                    │   (Generation) │
                    └───────────────┘
                            │
                        最终回答
```

### 三种集成模式

**模式一：KG 作为唯一检索源**

适用于知识库完全是结构化数据的场景（如企业产品目录、法规知识库）。

```
查询 → 实体识别 → KG 检索 → 上下文 → LLM 生成
```

**模式二：KG + 向量检索双通道**

最主流的模式——KG 处理精确关系检索，Vector 处理语义检索，结果融合。

```
查询 → 实体识别 ──→ KG 检索 ──→ 子图上下文
     └──→ 向量检索 ──→ 文本块上下文
                          │
                    结果融合 (Fusion)
                          │
                     LLM 生成
```

**模式三：KG 作为检索增强 Agent 的工具**

在 Agentic RAG 中，KG 查询是 Agent 的一个工具。

```
Agent 主循环:
  Think → 需要结构化关系信息 → 调用 kg_query("爱因斯坦") → 获取子图
  Think → 需要文档细节 → 调用 vector_search("爱因斯坦 理论") → 获取文档
  Think → 融合信息 → 生成回答
```

### 序列化策略

从 KG 检索到的子图不能直接输入 LLM——需要序列化为文本。以下是几种策略：

| 策略 | 描述 | 适用场景 | Token 效率 |
|------|------|---------|-----------|
| **三元组列表** | 每行一个 `(主体, 关系, 客体)` | 简单关系检索 | 高 |
| **自然语言描述** | `"爱因斯坦在1915年提出了广义相对论"` | 需要丰富上下文 | 中 |
| **JSON 结构** | `{"subject": ..., "relation": ..., "object": ...}` | 需要结构化处理 | 低 |
| **表格** | 关系类型汇总表 | 多实体多关系 | 高 |
| **图路径文本** | `爱因斯坦 → 提出 → 广义相对论` | 路径推理 | 高 |

**推荐的序列化格式**——混合使用三元组和自然语言：

```
=== 检索到的知识图谱上下文 ===

【核心实体】
  阿尔伯特·爱因斯坦 (Person)
    - 国籍: 德国/瑞士/美国
    - 出生: 1879-03-14
    
【关系路径】
  爱因斯坦 --[提出(1905)]--> 狭义相对论
  爱因斯坦 --[提出(1915)]--> 广义相对论
  爱因斯坦 --[提出(1905)]--> 光电效应
  广义相对论 --[预测]--> 引力波
  LIGO实验 --[验证(2016)]--> 广义相对论

【推理链】
  爱因斯坦提出广义相对论预测了引力波，LIGO实验在2016年首次观测到引力波，验证了广义相对论。
```

---

## 能力边界

### 知识图谱擅长什么

1. **精确事实查询**："爱因斯坦的出生地是哪里？" → 从 KG 直接读取属性，零幻觉
2. **多跳关系推理**："A 的 B 被 C 如何？" → 沿关系链路径遍历
3. **实体计数与聚合**："诺贝尔奖得主中有多少来自德国？" → 图聚合操作
4. **路径发现**："A 和 B 之间有什么关系？" → 最短路径或所有路径搜索
5. **层次结构查询**："XX 公司的组织架构是什么？" → 树形结构遍历
6. **约束满足查询**："找到所有 30 岁以下、来自中国的科学家" → 属性过滤 + 关系匹配

### 知识图谱不擅长什么

1. **模糊语义匹配**："找一篇关于科学发现的好文章" → 不知道"好"的语义，需要向量检索
2. **非结构化文本理解**："分析这篇文章的情感倾向" → KG 不适合做全文分析
3. **隐式关系发现**："哪些科学家可能合作？" → 需要预测，KG 只记录已知事实
4. **数值近似查询**："找价格在 100 元左右的产品" → 需要近似搜索，KG 精确相等
5. **长尾/冷启动实体**：只有 1 条关系的实体提供的上下文几乎为零
6. **开放域问答**："写一首关于科学的诗" → 不需要结构化知识，LLM 本身即可

### 边界指南

```
决策图：什么时候用 KG，什么时候用向量检索？

用户查询
    │
    ├── 查询包含明确的实体名称？
    │     └── 是 → 可以用 KG
    │     └── 否 → 考虑向量检索
    │
    ├── 查询需要多步推理才能回答？
    │     └── 是 → KG 为佳（关系遍历）
    │     └── 否 → 向量检索可能足够
    │
    ├── 需要精确数字/日期/ID？
    │     └── 是 → KG（精确匹配）
    │     └── 否 → 向量检索（语义近似）
    │
    ├── 查询是"找某种东西"（模糊）？
    │     └── 是 → 向量检索（语义搜索）
    │     └── 否 → 考虑 KG
    │
    └── 最佳实践：KG + 向量检索 混合使用
         - KG 负责精确关系检索
         - 向量检索负责语义补充
         - 两者互补而非替代

典型失败案例：
  ❌ 只用 KG 回答"帮我找一些关于量子计算的论文"
     → 失败原因：KG 存储论文的结构化元数据，不存储论文内容
     → 解法：KG 找到相关作者/领域 → 向量检索在论文原文中搜索
  
  ❌ 只用向量检索回答"图灵奖得主中哪些也是诺贝尔奖得主"
     → 失败原因：两个事实分散在不同文档中，向量检索无法关联
     → 解法：KG 存储"获奖"关系 → 集合交集查询
```

---

## 关键工程启示

1. **Schema 设计即知识建模**：不要在没想清楚"我的领域有哪些实体和关系"之前就开始构建。Schema 设计是 KG 项目中最重要、也最容易被低估的步骤。

2. **三元组质量是生命线**：一个错误的三元组（如"爱因斯坦_提出_量子计算"）会在多跳推理中放大错误，导致 LLM 生成荒谬回答。在构建时加入置信度过滤和人工校验是必要的。

3. **RDF 适合数据集成，Property Graph 适合应用开发**：如果你的 KG 需要跨组织共享和链接（像 Wikidata），使用 RDF/SPARQL。如果你在构建单一应用，Property Graph/Cypher 的开发效率更高。

4. **不要重建 Wikidata**：对于通用知识（历史人物、科学发现、地理位置等），直接复用 Wikidata 或 DBpedia 的现成数据。只对领域特有知识（如公司内部的产品架构、法规条款）进行自建。

5. **子图大小 = Token 成本**：每个扩展到子图中的节点和边都对应 LLM 上下文中的 token。控制子图展开的深度（1-2 跳为宜）和宽度（Top-5 或 Top-10 关系）。

6. **KG 不是 static dump**：知识在变化，KG 必须更新。设计增量更新管道——而不是定期全量重建——是生产级系统的要求。

7. **评估 KG 质量的三个维度**：
   - 准确性（Accuracy）：三元组是否与源文档一致？
   - 覆盖率（Coverage）：重要实体和关系是否都被抽取了？
   - 时效性（Freshness）：过时的三元组是否被更新或标记？

---

## 与本节其他子主题的关联

```
9.4.1 知识图谱基础 (你在这里)
    │
    ├──→ 9.4.2 Graph Construction
    │     如何从非结构化文本中抽取出 实体/关系/三元组
    │     NER 模型选择、LLM-based 抽取、增量更新
    │
    ├──→ 9.4.3 Graph RAG Flow
    │     子图检索 → 序列化 → LLM 上下文
    │     不同的检索策略和上下文组装方式
    │
    ├──→ 9.4.4 Graph Query
    │     SPARQL/Cypher 查询构造、LLM-to-Query 转换
    │     基于 实体/关系/三元组 的高效查询
    │
    ├──→ 9.4.5 Community Detection
    │     实体社区的发现、社区摘要生成
    │     利用 KG 结构进行全局知识组织
    │
    └──→ 9.4.6 Microsoft GraphRAG
         大规模 KG 构建、索引管线设计
         实体/关系/三元组 在工业级系统中的工程落地
```

---

## 延伸阅读与参考

- RDF 1.1 概念与抽象语法: https://www.w3.org/TR/rdf11-concepts/
- SPARQL 1.1 查询语言: https://www.w3.org/TR/sparql11-query/
- Neo4j Cypher 手册: https://neo4j.com/docs/cypher-manual/current/
- Wikidata SPARQL 端点: https://query.wikidata.org/
- NetworkX 文档: https://networkx.org/documentation/stable/
