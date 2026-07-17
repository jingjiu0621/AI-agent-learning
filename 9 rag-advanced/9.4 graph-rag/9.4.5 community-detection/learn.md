# 9.4.5 Community Detection — 社区发现（Louvain / Leiden）

> **核心问题**：知识图谱规模巨大（百万级节点），逐节点检索不现实。如何利用图的结构信息，将知识图谱划分为内聚的子图社区，从而实现层次化的摘要生成和高效检索？这是 Microsoft GraphRAG 的核心创新之一。

---

## 1. 背景与问题

### 1.1 为什么需要社区发现？

知识图谱中的实体天然具有**集群结构**（Cluster Structure）：

- 同一领域的实体之间连接密集（如"癌症-基因-药物"形成了一个密集子图）
- 跨领域的实体之间连接稀疏（如"癌症"和"足球"之间几乎没有直接关系）

这种结构特征催生了社区检测的核心思想：

```
   ┌─────────────────┐     ┌─────────────────┐
   │  生物医学知识子图  │     │  体育知识子图     │
   │                   │     │                  │
   │  基因──蛋白质     │     │   球员──俱乐部   │
   │    │    │         │     │     │    │       │
   │  疾病──药物       │     │   比赛──教练     │
   │                   │     │                  │
   └────────┬─────────┘     └────────┬─────────┘
            │                        │
            └──────────┬─────────────┘
                       │
             稀疏跨域连接（罕见）
```

**为什么要划分社区？**

1. **规模压缩**：将百万级节点压缩为千级社区，降低检索复杂度
2. **语义聚合**：同一社区的实体语义相关，可以生成社区级摘要
3. **层次化检索**：从粗粒度（高层社区）到细粒度（底层社区），渐进式获取信息
4. **减少噪音**：检索集中在相关子图，减少无关实体的干扰

### 1.2 Microsoft GraphRAG 中的社区发现定位

Microsoft GraphRAG 论文中，社区发现连接了"图构建"和"图检索"两个环节：

```
   实体抽取      关系抽取        社区划分       社区摘要        查询检索
┌──────────┐  ┌──────────┐  ┌────────────┐  ┌──────────┐  ┌──────────┐
│ 从文档   │─▶│ 抽取实体  │─▶│ 图社区划分  │─▶│ 生成社区  │─▶│ 基于社区  │
│ 抽取实体  │  │ 间关系    │  │ (Leiden)   │  │ 摘要     │  │ 摘要检索  │
└──────────┘  └──────────┘  └────────────┘  └──────────┘  └──────────┘
                                  │
                                  ▼
                          ┌────────────────┐
                          │ 层次化社区结构  │
                          │                │
                          │  Level 0 (细)  │
                          │  Level 1       │
                          │  Level 2       │
                          │  ...           │
                          │  Level n (粗)  │
                          └────────────────┘
```

---

## 2. 模块度（Modularity）—— 社区划分质量的度量

### 2.1 直观理解

模块度衡量的是：**社区内边的实际密度**与**同等图在随机连接情况下的期望密度**之间的差异。

```
直觉：
 ┌─────────────────────────────────────────────┐
 │ 好的社区：社区内连接密集，社区间连接稀疏      │
 │                                             │
 │  实际：A-B 相连, A-C 相连, B-C 相连 (3条边) │
 │  随机期望：3个节点的随机连接概率很低        │
 │  模块度 = 实际 - 期望 > 0 → 好的社区        │
 └─────────────────────────────────────────────┘
```

### 2.2 数学定义

模块度（Modularity）的数学定义有多种等价形式，以下是 Newman-Girvan 标准形式：

**核心公式**：

$$Q = \frac{1}{2m} \sum_{ij} \left[ A_{ij} - \frac{k_i k_j}{2m} \right] \delta(c_i, c_j)$$

其中：

| 符号 | 含义 |
|---|---|
| $Q$ | 模块度值，范围 [-1, 1] |
| $m$ | 图中边的总数 |
| $A_{ij}$ | 邻接矩阵，节点 i 和 j 之间有边时为 1，否则为 0 |
| $k_i$ | 节点 i 的度数（连接边的数量） |
| $c_i$ | 节点 i 所属的社区 |
| $\delta(c_i, c_j)$ | Kronecker delta，当 $c_i = c_j$ 时为 1（即节点在同一社区），否则为 0 |

**分解理解**：

$$Q = \frac{1}{2m} \sum_{ij} \underbrace{A_{ij}}_{\text{实际连接}} \delta(c_i, c_j) - \frac{1}{2m} \sum_{ij} \underbrace{\frac{k_i k_j}{2m}}_{\text{随机期望}} \delta(c_i, c_j)$$

- **第一项**：社区内实际存在的边数
- **第二项**：在随机连接假设下，社区内期望的边数
- $Q > 0$ 表示社区结构好于随机；$Q > 0.3$ 通常认为社区结构显著

### 2.3 模块度矩阵形式

对于两个社区的划分，定义：

$$B_{ij} = A_{ij} - \frac{k_i k_j}{2m}$$

则模块度可以写为：

$$Q = \frac{1}{2m} \sum_{ij} B_{ij} \delta(c_i, c_j)$$

### 2.4 模块度的局限

```
模块度最大化的问题：
┌───────────────────────────────────────────────────┐
│                                                    │
│  1. 分辨率限制 (Resolution Limit)                   │
│     ┌─────┐  ┌─────┐  ┌─────┐                      │
│     │ C1  │──│ C2  │──│ C3  │ ... ──│ Cn │         │
│     └─────┘  └─────┘  └─────┘                      │
│     每个社区内部完全连接，两两之间仅1条边              │
│     → 模块度最大化可能将多个小社区合并为一个大社区      │
│     → 无法发现小尺度的社区结构                        │
│                                                    │
│  2. 存在"模块度高原"                                │
│     多个划分方案具有非常接近的模块度值               │
│     算法可能收敛到次优解                             │
│                                                    │
│  3. 倾向于平衡大小的社区                             │
│     实际数据中社区大小通常高度偏态（幂律分布）        │
└───────────────────────────────────────────────────┘
```

---

## 3. Louvain 算法

### 3.1 算法概述

Louvain 算法（也称 Blondel 算法，2008年提出）是目前最流行的社区发现算法之一，以高效率和高模块度优化质量著称。

**核心思想**：贪心优化 + 层次化聚合

```
时间复杂度：O(n log n)
空间复杂度：O(m + n)
n = 节点数, m = 边数
```

### 3.2 算法步骤

Louvain 通过**两个阶段的迭代**来优化模块度：

#### Phase 1: 局部优化 (Local Optimization)

```
对每个节点 i：
  1. 移除 i 从其当前社区
  2. 计算将 i 移动到每个邻居社区的模块度增益 ΔQ
  3. 如果最大的 ΔQ > 0，将 i 移动到那个社区
  4. 重复直到没有节点移动
```

**模块度增益公式**：

将孤立的节点 i 移动到社区 C 的模块度增益为：

$$\Delta Q = \left[ \frac{\Sigma_{in} + 2k_{i,C}}{2m} - \left( \frac{\Sigma_{tot} + k_i}{2m} \right)^2 \right] - \left[ \frac{\Sigma_{in}}{2m} - \left( \frac{\Sigma_{tot}}{2m} \right)^2 - \left( \frac{k_i}{2m} \right)^2 \right]$$

其中：
- $\Sigma_{in}$：社区 C 内部边的权重之和
- $\Sigma_{tot}$：社区 C 内所有节点关联边的总权重
- $k_i$：节点 i 的度数
- $k_{i,C}$：节点 i 到社区 C 内节点的边权重之和
- $m$：全图边的总权重

#### Phase 2: 图聚合 (Aggregation)

优化完成后，将每个社区视为一个"超级节点"，构建新的层次图：

```
Phase 1 之后：           Phase 2 之后：
                        ┌──────────┐
  A ── B                │ Community│
  │     │               │   1     │
  C ── D                └──────────┘
                            │
  [社区划分: {A,B,C,D}]     │
                            ▼
                        ┌──────────┐
                        │ Community│
                        │   2     │
                        └──────────┘

  两个社区之间如果原来有 k 条边连接，
  则聚合后的超级节点之间有一条权重为 k 的边
```

### 3.3 完整迭代过程（ASCII 动画）

```
┌────────── 第一轮迭代 ──────────┐
│                                 │
│  Phase 1: 局部优化               │
│                                 │
│  初始图:           第一步优化后:    │
│  A───B  C───D      A───B  C───D  │
│  │   │  │   │  →   │   │  │   │  │
│  E───F  G───H      E───F  G───H  │
│                                 │
│  Step 1: 检查 A                  │
│  A 的邻居: B(社区1), E(社区1)    │
│  → A 留在社区1                  │
│                                 │
│  Step 2: 检查 B                  │
│  B 的邻居: A(社区1), F(社区1)    │
│  → B 留在社区1                  │
│                                 │
│  ... (多次迭代)                  │
│                                 │
│  最终分区:                       │
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐      │
│  │A,E│ │B,F│ │C,G│ │D,H│      │
│  └───┘ └───┘ └───┘ └───┘      │
│                                 │
├─────────────────────────────────┤
│                                 │
│  Phase 2: 图聚合                 │
│                                 │
│  聚合为超级节点:                  │
│    ┌────┐   w=2   ┌────┐       │
│    │ C1 │────────│ C2 │        │
│    │A,E │        │B,F │        │
│    └────┘        └────┘        │
│      │w=1          │w=1        │
│    ┌────┐        ┌────┐       │
│    │ C3 │────────│ C4 │        │
│    │C,G │  w=2   │D,H │        │
│    └────┘        └────┘       │
│                                 │
├─────────────────────────────────┤
│                                 │
│ ┌──────── 第二轮迭代 ────────┐  │
│ │ Phase 1 再次优化            │  │
│ │ → C1 和 C2 合并为更大的社区 │  │
│ │ → C3 和 C4 合并为更大的社区 │  │
│ │                            │  │
│ │ ┌─────────┐  ┌─────────┐ │  │
│ │ │C1∪C2    │──│C3∪C4    │ │  │
│ │ │A,B,E,F  │  │C,D,G,H  │ │  │
│ │ └─────────┘  └─────────┘ │  │
│ └──────────────────────────┘  │
│                                 │
│  模块度不再提升 → 算法终止       │
└─────────────────────────────────┘

最终社区层次结构：
Level 2: [{A,B,E,F}, {C,D,G,H}]        ← 粗粒度
Level 1: [{A,E}, {B,F}, {C,G}, {D,H}]  ← 细粒度
Level 0: [{A},{B},{C},{D},{E},{F},{G},{H}]  ← 原始节点
```

### 3.4 图形化示例

下面是一个更具体的例子，展示 Louvain 如何工作：

```
初始图（10个节点，明显存在两个密集区域）：

    ┌───── 社区 A ─────┐    ┌───── 社区 B ─────┐
    │                   │    │                   │
    │  A1 ─── A2        │    │  B1 ─── B2        │
    │  │ \    │         │    │  │ \    │         │
    │  │  \   │         │    │  │  \   │         │
    │  A3 ─── A4 ── X ─┼────┼── B3 ─── B4        │
    │  │    \ │         │    │  │    \ │         │
    │  A5 ─── A6        │    │  B5 ─── B6        │
    └───────────────────┘    └───────────────────┘
                       ↑
              连接两个社区的桥接节点 X

Step 1: 每个节点初始化为自己的社区
Step 2: 对每个节点，计算移动到相邻社区的 ΔQ
Step 3: A 系列节点之间 ΔQ 为正 → 聚合成社区 A
Step 4: B 系列节点之间 ΔQ 为正 → 聚合成社区 B
Step 5: X 连接到 A 和 B → 计算 ΔQ，取决于连接密度
        如果 X 到 A 有 3 条边，到 B 有 2 条边 → X 加入 A
Step 6: 图聚合，进入下一轮迭代
```

### 3.5 Louvain 的局限性

```
┌──────────────────────────────────────────────────┐
│               Louvain 的已知问题                    │
├──────────────────────────────────────────────────┤
│                                                    │
│  1. 分辨率限制 (Resolution Limit)                   │
│     - 可能合并较小的、有意义的社区                    │
│     - 例：在大型网络中，几个节点的紧密子图可能被忽略   │
│                                                    │
│  2. 可能产生 disconnected communities               │
│     - Louvain 的 Phase 1 只考虑模块度增益            │
│     - 可能将不相连的节点分配到同一社区                │
│     ┌─────┐                                        │
│     │  C1  │──(稀疏连接)──│  C2  │                  │
│     └─────┘              └─────┘                    │
│      ↑ 同一社区！但无直接连接                         │
│                                                    │
│  3. 随机性导致不稳定                                │
│     - 节点处理顺序影响结果                            │
│     - 多次运行可能得到不同划分                        │
│                                                    │
│  4. 可能收敛到次优解                                │
│     - 贪心算法的本质决定了它可能陷入局部最优          │
│                                                    │
└──────────────────────────────────────────────────┘
```

---

## 4. Leiden 算法 — Louvain 的改进

### 4.1 为什么需要 Leiden？

Leiden 算法（Traag et al., 2019）是对 Louvain 的改进，主要解决了 Louvain 的两个核心问题：
1. **Disconnected communities**（不连通社区）
2. **低质量的局部最优解**

### 4.2 Leiden 的三阶段结构

Leiden 在 Louvain 的两阶段基础上，增加了一个**细化阶段（Refinement Phase）**：

```
Louvain:  局部优化 → 图聚合 → 迭代
Leiden:   局部优化 → 细化分区 → 图聚合 → 迭代

新增的细化阶段：
  - 在局部优化后，对每个社区内部再次细化
  - 保证社区是连通的（connected）
  - 进一步优化模块度
```

#### Phase 1: 局部优化 (Local Movement)

与 Louvain 相同，但使用更严格的优化条件：

```
对每个节点 i：
  仅在以下情况下移动节点：
  1. 模块度增益 ΔQ > 0
  2. 目标社区与 i 有至少一条边相连（保证连通性）
```

#### Phase 2: 细化 (Refinement)

这是 Leiden 与 Louvain 最关键的差异：

```
对每个由 Phase 1 得到的社区 C：
  1. 将 C 中的每个节点初始化为独立子社区
  2. 在 C 内部运行局部优化，允许节点在 C 内合并为更优的子分区
  3. 保证每个子分区是连通的

  目的：打破 Phase 1 中可能形成的次优结构
```

**数学上看，细化阶段在寻找社区内部的"更优分裂"**：

```
Phase 1 后的分区：    细化阶段：
┌──────────────┐     ┌──────┐ ┌──────┐ ┌──────┐
│              │     │SubC1 │ │SubC2 │ │SubC3 │
│   社区 C     │  →  │      │ │      │ │      │
│ (可能次优)   │     └──────┘ └──────┘ └──────┘
└──────────────┘     ↑ 在 C 内部进一步优化
```

#### Phase 3: 图聚合 (Aggregation)

与 Louvain 相同，但基于细化后的分区进行聚合：

```
细化后的子分区 → 聚合为超级节点 → 构建新图
```

### 4.3 连通性保证

Leiden 的连通性保证是其与 Louvain 最显著的区别：

```
Louvain 可能产生的"不连通社区"：
  A ── B ── C    (A、B、C 被归为同一社区)
  │         │
  D         E    (但 D 只连 A，E 只连 C
       ┌───── 但 D 和 E 之间无路径可达！)

Leiden 的保证：
  A ── B ── C
  │         │       ← 细化阶段保证每个社区
  D         E         是强连通的子图
  ↑ D 和 E 不会被分在无路径可达的同一社区
```

### 4.4 算法流程对比

```
Louvain 算法流程：
================
输入: 图 G(V, E)
输出: 社区分区 P

while 模块度还在提升:
    # Phase 1: 局部优化
    for each node i in V:
        计算移动到邻居社区的 ΔQ
        if max(ΔQ) > 0:
            移动 i 到目标社区
    
    # Phase 2: 图聚合
    G ← 基于当前分区构建聚合图


Leiden 算法流程：
================
输入: 图 G(V, E)
输出: 社区分区 P

while 模块度还在提升:
    # Phase 1: 局部优化
    for each node i in V:
        计算移动到邻居社区的 ΔQ
        if max(ΔQ) > 0 AND 目标社区与 i 连通:
            移动 i 到目标社区
    
    # Phase 2: 细化 (差别在这里!)
    for each community C in 当前分区:
        将 C 分割为更优的子分区 P_C
        保证 P_C 中每个子分区是连通的
    
    # Phase 3: 图聚合
    G ← 基于细化后的分区构建聚合图
```

### 4.5 速度和稳定性优势

Leiden 不仅质量更好，**速度也更快**：

```
原因分析：
  1. Louvain 的 Phase 1 可能产生 disconnected communities
     导致后续迭代需要额外操作来修复
  
  2. Leiden 的细化阶段虽然增加了一个阶段，但：
     - 细化操作在社区内部进行，范围较小
     - 更好的初始分区 → 聚合后更优 → 更快收敛
     - 整体迭代次数减少

  3. 包含一个"快速局部移动"优化：
     - 只考虑可能改善模块度的移动
     - 跳过明显无收益的节点
```

实验结果（Traag et al., 2019）：

| 指标 | Louvain | Leiden |
|---|---|---|
| 模块度 (归一化) | 0.85 | **0.90** |
| 相对运行时间 | 1.0x | **0.6-0.8x** |
| disconnected 社区比例 | 5-20% | **0%** |
| 多次运行一致性 | 低 | **高** |

---

## 5. 在 Graph RAG 中的应用

### 5.1 Microsoft GraphRAG 的社区层次结构

Microsoft GraphRAG 的核心创新之一，就是利用 Leiden 算法的**层次化特性**来构建多粒度社区摘要：

```
                Level 2 (最粗粒度)
        ┌──────────────────────────┐
        │     全球/主题级社区      │
        │  "生物医学" "计算机科学" │
        └──────────┬──────────────┘
                   │
        ┌──────────┴──────────────┐
        │  Level 1                │
        │   "癌症研究" "AI研究"   │
        └──────────┬──────────────┘
                   │
        ┌──────────┴──────────────┐
        │  Level 0 (最细粒度)     │
        │  "肺癌" "乳腺癌"        │
        │  "GNN" "NLP"           │
        └─────────────────────────┘
```

**层次化的生成过程**：

```
原始知识图谱 (实体/关系)
    │
    ▼ Leiden 社区检测
    │
    ├── Level 0: 最细粒度社区 (数百个)
    │   └── 每个社区的实体列表 + 关系
    │       └── LLM 生成社区摘要 (Community Summary)
    │           └── "社区 A 包含基因 X、蛋白质 Y、疾病 Z，它们之间的关系是..."
    │
    ├── Level 1: 中间粒度社区 (数十个)
    │   └── 对 Level 0 的摘要进行摘要
    │       └── LLM 生成高层摘要
    │           └── "该领域涉及癌症信号通路相关的基因和蛋白质..."
    │
    └── Level 2: 最粗粒度社区 (数个)
        └── 对 Level 1 的摘要进行摘要
            └── LLM 生成总结摘要
                └── "生物医学知识图谱包含..."
```

### 5.2 社区摘要 (Community Summary)

每个社区生成的摘要包含以下要素：

```json
{
  "community_id": "C_12345",
  "level": 0,
  "entities": ["基因_TP53", "蛋白质_p53", "疾病_肺癌"],
  "relationships": [
    {"source": "基因_TP53", "target": "蛋白质_p53", "type": "编码"},
    {"source": "蛋白质_p53", "target": "疾病_肺癌", "type": "相关"}
  ],
  "summary": "该社区聚焦于 TP53 基因及其编码的 p53 蛋白在肺癌中的作用。TP53 是一个重要的抑癌基因，其突变与多种癌症相关。p53 蛋白参与细胞周期调控、DNA 修复和细胞凋亡等关键生物过程。在肺癌中，TP53 突变是常见的基因改变，影响肿瘤的发生、发展和治疗响应。",
  "key_findings": [
    "TP53 突变在非小细胞肺癌中发生率约 50%",
    "p53 蛋白通过调控 p21 表达来控制细胞周期",
    "TP53 突变状态影响化疗和靶向治疗的效果"
  ]
}
```

### 5.3 查询时的社区检索策略

Microsoft GraphRAG 根据查询类型采用不同的社区检索策略：

#### 全局查询 (Global Search) — 从高层社区摘要回答

```
用户查询: "知识图谱在医疗领域的主要应用有哪些？"
              │
              ▼
     ┌──────────────────────┐
     │ 选择高层社区摘要      │
     │ (Level 1-2)          │
     └──────────────────────┘
              │
              ▼
     ┌──────────────────────┐
     │ 向量检索 + 关键词匹配 │
     │ 找到最相关的社区摘要  │
     └──────────────────────┘
              │
              ▼
     ┌──────────────────────┐
     │ LLM 综合多社区摘要    │
     │ 生成全局回答          │
     └──────────────────────┘
              │
              ▼
        "知识图谱在医疗领域的应用包括：
         1. 药物发现 (社区A摘要提到...)
         2. 临床决策支持 (社区B摘要提到...)
         3. 基因组学分析 (社区C摘要提到...)"
```

**适用场景**：
- 需要全局概览的问题（"主要有哪些方向？"）
- 跨领域综合分析问题
- 需要多源信息综合的问题

**特点**：
- 检索效率高（只搜索少量高层摘要）
- 覆盖面广（高层摘要已经提炼了关键信息）
- 可能丢失细节（粗粒度摘要不可避免地丢失底层细节）

#### 局部查询 (Local Search) — 从特定社区精确检索

```
用户查询: "TP53 突变如何影响肺癌治疗？"
              │
              ▼
     ┌──────────────────────┐
     │ 实体识别 + 图扩展     │
     │ TP53 → 找出所在社区   │
     └──────────────────────┘
              │
              ▼
     ┌──────────────────────┐
     │ 选择 Level 0 社区     │
     │ "TP53-p53-肺癌"社区   │
     └──────────────────────┘
              │
              ▼
     ┌──────────────────────┐
     │ 检索社区摘要 +        │
     │ 社区内具体实体关系     │
     └──────────────────────┘
              │
              ▼
     ┌──────────────────────┐
     │ LLM 使用精确信息      │
     │ 生成针对性回答        │
     └──────────────────────┘
              │
              ▼
        "TP53 突变对肺癌治疗的影响包括：
         1. 某些化疗药物对 TP53 突变型肿瘤效果较差
         2. TP53 状态是免疫治疗反应的预测生物标志物
         3. 正在开发针对 TP53 突变的小分子药物..."
```

**适用场景**：
- 需要精确事实的问题（"某药物如何起作用？"）
- 特定实体相关的查询
- 需要引用具体关系路径的问题

**特点**：
- 精确度高（基于实体级别的图数据）
- 可以追溯事实来源（具体的关系路径）
- 检索范围小（只涉及相关社区）

### 5.4 混合检索策略

在实际系统中，全局和局部检索通常是**组合使用**的：

```
用户查询
    │
    ▼
┌──────────────────────┐
│ 查询分类器 (LLM)      │
│ 判断查询类型          │
└──────┬───────┬───────┘
       │       │
   全局型    局部型
       │       │
       ▼       ▼
┌─────────┐ ┌─────────┐
│ 全局检索 │ │ 局部检索 │
│ 高层摘要 │ │ 精确检索 │
│ 快速回答 │ │ 细节补充 │
└────┬────┘ └────┬────┘
     │           │
     └─────┬─────┘
           │
           ▼
    ┌──────────────┐
    │ LLM 综合生成  │
    │ 有层次地回答  │
    └──────────────┘
```

---

## 6. 算法对比表

| 维度 | Louvain | Leiden |
|---|---|---|
| **提出年份** | 2008 (Blondel et al.) | 2019 (Traag et al.) |
| **算法阶段** | 2阶段 (局部优化 + 聚合) | 3阶段 (局部优化 + 细化 + 聚合) |
| **连通性保证** | ❌ 可能产生不连通社区 | ✅ 保证社区连通 |
| **时间复杂度** | O(n log n) | O(n log n) |
| **相对速度** | 基准 (1.0x) | 更快 (0.6-0.8x) |
| **模块度质量** | 好 (相对 0.85) | 更好 (相对 0.90) |
| **稳定性** | 中 (受节点顺序影响) | 高 (多次运行结果一致) |
| **可扩展性** | 百万级 | 千万级 |
| **分辨率限制** | 有 | 有（但影响更小） |
| **GraphRAG 使用** | 早期版本 | **推荐版本** ⭐ |
| **实现难度** | 简单 | 中等 |
| **主要库支持** | python-louvain, NetworkX, igraph | igraph, leidenalg, NetworkX |

---

## 7. 工程实践

### 7.1 使用 igraph 进行 Leiden 社区检测

```python
"""
使用 Leiden 算法对知识图谱进行社区检测
实际运行于 Microsoft GraphRAG 类似的流程
"""

import igraph as ig
import leidenalg
import networkx as nx
import json
from typing import Dict, List, Tuple
from collections import defaultdict


def build_graph_from_knowledge_graph(
    entities: List[dict],
    relationships: List[dict]
) -> ig.Graph:
    """
    将知识图谱（实体+关系列表）构建为 igraph 图对象
    
    Args:
        entities: [{"id": "E1", "name": "TP53", "type": "Gene"}, ...]
        relationships: [{"source": "E1", "target": "E2", "type": "encodes"}, ...]
    
    Returns:
        加权图（关系类型可以用于权重或作为边属性）
    """
    g = ig.Graph(directed=False)  # GraphRAG 通常使用无向图
    
    # 添加节点
    entity_map = {}
    for i, entity in enumerate(entities):
        g.add_vertex(
            name=entity["id"],
            label=entity.get("name", entity["id"]),
            entity_type=entity.get("type", "Unknown")
        )
        entity_map[entity["id"]] = i
    
    # 添加边（无向）
    edge_count = defaultdict(int)
    for rel in relationships:
        src = entity_map.get(rel["source"])
        tgt = entity_map.get(rel["target"])
        if src is not None and tgt is not None:
            # 去重计数作为权重
            pair = (min(src, tgt), max(src, tgt))
            edge_count[pair] += 1
    
    edges = list(edge_count.keys())
    weights = list(edge_count.values())
    g.add_edges(edges)
    g.es["weight"] = weights
    
    return g, entity_map


def detect_communities_leiden(
    graph: ig.Graph,
    resolution: float = 1.0,
    seed: int = 42
) -> Tuple[List[int], ig.VertexClustering]:
    """
    使用 Leiden 算法进行社区检测
    
    Args:
        graph: igraph 图对象
        resolution: 分辨率参数
            - < 1.0: 产生更少的、更大的社区
            - > 1.0: 产生更多、更小的社区
            - = 1.0: 默认
        seed: 随机种子，确保可重复性
    
    Returns:
        (membership_list, clustering_object)
    """
    # Leiden 算法
    partition = leidenalg.find_partition(
        graph,
        leidenalg.RBConfigurationVertexPartition,  # 使用 Reichardt-Bornholdt 模型
        weights="weight" if "weight" in graph.es.attribute_names() else None,
        resolution_parameter=resolution,
        seed=seed
    )
    
    # partition 对象自带了层次结构信息
    membership = partition.membership
    
    return membership, partition


def extract_community_hierarchy(
    graph: ig.Graph,
    partition,
    max_levels: int = 5
) -> Dict[int, Dict]:
    """
    提取社区层次结构（利用 Leiden 的层次化聚合）
    
    Microsoft GraphRAG 使用这个层次结构来生成多粒度摘要
    
    Returns:
        {
            0: {  # Level 0 (最细粒度)
                "n_communities": 100,
                "communities": {
                    0: {"nodes": [...], "size": 10},
                    1: {"nodes": [...], "size": 15},
                    ...
                }
            },
            1: {  # Level 1 (聚合后的社区)
                ...
            },
            ...
        }
    """
    hierarchy = {}
    
    # 将分区结果转换为层次结构
    # Leiden 的分区对象支持 .level 属性访问层次
    current_membership = partition.membership
    
    # Level 0: 原始分区
    communities_l0 = defaultdict(list)
    for node_idx, comm_id in enumerate(current_membership):
        communities_l0[comm_id].append(
            graph.vs[node_idx]["name"]
        )
    
    hierarchy[0] = {
        "n_communities": len(communities_l0),
        "communities": {
            cid: {
                "nodes": nodes,
                "size": len(nodes)
            }
            for cid, nodes in communities_l0.items()
        }
    }
    
    # 更高层次的聚合可以从 partition 的层次信息中提取
    # 这里简化为只输出 Level 0
    # 实际 GraphRAG 使用聚合图来构建多级层次
    
    return hierarchy


def generate_community_summary(
    community_nodes: List[str],
    relationships: List[dict],
    entity_details: Dict[str, dict],
    llm_client,
    model: str = "gpt-4o"
) -> str:
    """
    使用 LLM 为单个社区生成摘要
    
    这是 Microsoft GraphRAG 的核心环节之一：
    每个社区 → 社区摘要 → 用于检索
    """
    # 构建社区上下文字符串
    node_info = []
    for node_id in community_nodes:
        detail = entity_details.get(node_id, {})
        node_info.append(
            f"- {detail.get('name', node_id)} ({detail.get('type', 'Unknown')}): "
            f"{detail.get('description', '')}"
        )
    
    # 过滤出社区内的关系
    community_node_set = set(community_nodes)
    community_rels = [
        r for r in relationships
        if r["source"] in community_node_set and r["target"] in community_node_set
    ]
    
    rel_info = [
        f"- {entity_details.get(r['source'], {}).get('name', r['source'])} "
        f"--[{r['type']}]--> "
        f"{entity_details.get(r['target'], {}).get('name', r['target'])}"
        for r in community_rels
    ]
    
    context = (
        "以下是知识图谱中的一个社区，包含以下实体和关系:\n\n"
        "实体:\n" + "\n".join(node_info) + "\n\n"
        "关系:\n" + "\n".join(rel_info)
    )
    
    # 调用 LLM 生成摘要
    response = llm_client.chat.completions.create(
        model=model,
        messages=[
            {
                "role": "system",
                "content": (
                    "你是一个知识图谱分析专家。给定一个社区中的实体和关系列表，"
                    "生成一个简洁但信息丰富的社区摘要。摘要应该：\n"
                    "1. 概括该社区的核心主题\n"
                    "2. 提及关键实体和它们之间的关系\n"
                    "3. 总结重要的发现或模式\n"
                    "4. 为后续检索提供有意义的上下文"
                )
            },
            {"role": "user", "content": context}
        ],
        temperature=0.3,
        max_tokens=1000
    )
    
    return response.choices[0].message.content


def full_graphrag_community_pipeline(
    entities: List[dict],
    relationships: List[dict],
    llm_client,
    resolution: float = 1.0
) -> dict:
    """
    完整的 GraphRAG 社区检测和摘要生成流水线
    
    这个流水线模拟了 Microsoft GraphRAG 的核心步骤
    """
    # Step 1: 构建图
    print("Step 1: Building graph...")
    graph, entity_map = build_graph_from_knowledge_graph(entities, relationships)
    
    # Step 2: Leiden 社区检测
    print("Step 2: Detecting communities with Leiden...")
    membership, partition = detect_communities_leiden(
        graph, resolution=resolution
    )
    
    # Step 3: 提取层次结构
    print("Step 3: Extracting community hierarchy...")
    hierarchy = extract_community_hierarchy(graph, partition)
    
    # Step 4: 构建节点到社区的映射
    node_to_community = {}
    for node_name, comm_id in zip(
        [graph.vs[i]["name"] for i in range(len(graph.vs))],
        membership
    ):
        node_to_community[node_name] = comm_id
    
    entity_details = {e["id"]: e for e in entities}
    
    # Step 5: 为每个社区生成摘要
    print("Step 4: Generating community summaries...")
    community_summaries = {}
    for level, level_data in hierarchy.items():
        for comm_id, comm_data in level_data["communities"].items():
            print(f"  Generating summary for Level {level}, Community {comm_id} "
                  f"({comm_data['size']} nodes)...")
            summary = generate_community_summary(
                comm_data["nodes"],
                relationships,
                entity_details,
                llm_client
            )
            community_summaries[(level, comm_id)] = {
                "summary": summary,
                "nodes": comm_data["nodes"],
                "size": comm_data["size"]
            }
    
    return {
        "n_communities": len(set(membership)),
        "modularity": partition.modularity,
        "hierarchy": hierarchy,
        "membership": {
            node_id: node_to_community[node_id]
            for node_id in node_to_community
        },
        "summaries": community_summaries
    }


if __name__ == "__main__":
    # 示例数据
    sample_entities = [
        {"id": "E1", "name": "TP53", "type": "Gene", "description": "Tumor protein p53 gene"},
        {"id": "E2", "name": "p53", "type": "Protein", "description": "Cellular tumor antigen p53"},
        {"id": "E3", "name": "Lung Cancer", "type": "Disease", "description": "Malignant lung tumor"},
        # ... 更多实体
    ]
    
    sample_relationships = [
        {"source": "E1", "target": "E2", "type": "encodes"},
        {"source": "E2", "target": "E3", "type": "associated_with"},
        # ... 更多关系
    ]
    
    # 实际使用时，需要传入真实的 LLM client
    # result = full_graphrag_community_pipeline(
    #     sample_entities,
    #     sample_relationships,
    #     llm_client,
    #     resolution=1.0
    # )
    # print(f"Found {result['n_communities']} communities")
    # print(f"Modularity: {result['modularity']:.4f}")
```

### 7.2 使用 python-louvain 快速实验

```python
"""
使用 python-louvain (community) 库进行快速社区检测实验
适合小规模图和原型验证
"""

import community as community_louvain
import networkx as nx
import matplotlib.pyplot as plt
import numpy as np

def louvain_demo():
    """Louvain 算法的快速演示"""
    
    # 创建一个具有社区结构的人工图
    G = nx.karate_club_graph()  # Zachary's Karate Club (34节点)
    print(f"图信息: {G.number_of_nodes()} 节点, {G.number_of_edges()} 边")
    
    # 应用 Louvain 算法
    partition = community_louvain.best_partition(G)
    
    # 计算模块度
    modularity = community_louvain.modularity(partition, G)
    print(f"Louvain 模块度: {modularity:.4f}")
    
    # 统计社区分布
    comm_sizes = {}
    for node, comm_id in partition.items():
        comm_sizes[comm_id] = comm_sizes.get(comm_id, 0) + 1
    
    print(f"社区数量: {len(comm_sizes)}")
    print("社区大小分布:", sorted(comm_sizes.values(), reverse=True))
    
    # 可视化
    pos = nx.spring_layout(G, seed=42)
    colors = [partition[n] for n in G.nodes()]
    
    plt.figure(figsize=(10, 8))
    nx.draw(
        G, pos,
        node_color=colors,
        cmap=plt.cm.Set3,
        with_labels=True,
        node_size=300,
        edge_color="gray",
        alpha=0.8
    )
    plt.title(f"Louvain Community Detection\nModularity = {modularity:.4f}")
    plt.savefig("louvain_communities.png")
    plt.show()
    
    return partition, modularity


def leiden_vs_louvain_comparison():
    """比较 Leiden 和 Louvain 在同一数据集上的表现"""
    
    # 生成一个更大的测试图（SBM：Stochastic Block Model）
    np.random.seed(42)
    
    # 定义社区结构：5个社区，每个社区内部连接密集，社区之间连接稀疏
    sizes = [50, 80, 60, 40, 70]  # 5个社区的大小
    probs = [
        [0.5, 0.01, 0.01, 0.02, 0.01],
        [0.01, 0.6, 0.01, 0.01, 0.02],
        [0.01, 0.01, 0.55, 0.01, 0.01],
        [0.02, 0.01, 0.01, 0.5, 0.01],
        [0.01, 0.02, 0.01, 0.01, 0.65],
    ]
    
    G = nx.stochastic_block_model(sizes, probs, seed=42)
    print(f"SBM 图: {G.number_of_nodes()} 节点, {G.number_of_edges()} 边")
    
    # Louvain 社区检测
    partition_louvain = community_louvain.best_partition(G)
    modularity_louvain = community_louvain.modularity(partition_louvain, G)
    
    # Leiden 社区检测（使用 igraph）
    # 将 NetworkX 图转为 igraph
    g_ig = ig.Graph.from_networkx(G)
    partition_leiden = leidenalg.find_partition(
        g_ig,
        leidenalg.RBConfigurationVertexPartition,
        seed=42
    )
    modularity_leiden = partition_leiden.modularity
    
    print(f"\n=== Louvain vs Leiden 比较 ===")
    print(f"Louvain 社区数: {len(set(partition_louvain.values()))}, "
          f"模块度: {modularity_louvain:.4f}")
    print(f"Leiden  社区数: {len(set(partition_leiden.membership))}, "
          f"模块度: {modularity_leiden:.4f}")
    
    return {
        "louvain": {"modularity": modularity_louvain, "n_comm": len(set(partition_louvain.values()))},
        "leiden": {"modularity": modularity_leiden, "n_comm": len(set(partition_leiden.membership))}
    }
```

### 7.3 大规模图优化策略

当图规模达到百万级节点时，需要以下优化：

```python
"""
大规模图社区检测的工程优化策略
"""

def chunked_processing_optimizations():
    """
    大规模图社区检测的优化策略
    """
    strategies = {
        "内存优化": [
            "使用稀疏矩阵存储图结构 (scipy.sparse)",
            "只存储必要的边属性，避免冗余",
            "使用 int 而非 str 标识节点（压缩内存）",
            "分块加载，不一次加载全部到内存"
        ],
        "计算优化": [
            "使用多线程/多进程并行化局部移动计算",
            "GPU 加速邻接矩阵操作（cuGraph 库）",
            "增量更新：只重新计算变化的社区",
            "近似算法：采样部分节点进行社区检测"
        ],
        "采样策略": [
            "如果图太大无法全部加载，先对图进行分层采样",
            "在全图上运行快速粗化算法",
            "在采样子图上检测社区，然后投影到全图"
        ],
        "分布式方案": [
            "Apache Spark GraphX: Pregel-like 社区检测",
            "Distributed Louvain: 分片计算+合并",
            "Neo4j Graph Data Science: 内置分布式社区检测"
        ]
    }
    
    for category, items in strategies.items():
        print(f"\n{category}:")
        for item in items:
            print(f"  - {item}")


def incremental_community_detection():
    """
    增量社区检测：当图结构发生变化时，不需要完全重新计算
    """
    # 策略1: 局部更新 (Local Update)
    # 只重新计算受影响区域的社区
    # 新增节点时，只运行局部优化
    
    # 策略2: 标签传播 (Label Propagation)
    # 使用现有分区作为初始条件，快速更新
    
    # 策略3: 锚点社区 (Anchor Communities)
    # 预计算 "锚点社区"，新节点匹配最近的锚点
    pass
```

### 7.4 分辨率参数调优

分辨率参数控制社区粒度，直接影响 Graph RAG 的检索效果：

```python
def tune_resolution_parameter(graph: ig.Graph) -> float:
    """
    调优分辨率参数以找到最佳社区粒度
    
    在 Graph RAG 场景中，社区粒度影响：
    - 太粗 (resolution < 0.5): 社区太大，摘要失去特异性
    - 太细 (resolution > 2.0): 社区太小，摘要意义不大
    - 最佳 (resolution ~ 1.0): 平衡摘要的信息密度和覆盖面
    """
    resolutions = [0.3, 0.5, 0.8, 1.0, 1.2, 1.5, 2.0]
    
    results = []
    for r in resolutions:
        partition = leidenalg.find_partition(
            graph,
            leidenalg.RBConfigurationVertexPartition,
            resolution_parameter=r,
            seed=42
        )
        
        n_comm = len(set(partition.membership))
        mod = partition.modularity
        
        results.append({
            "resolution": r,
            "n_communities": n_comm,
            "modularity": mod,
            "avg_community_size": graph.vcount() / n_comm
        })
        
        print(f"resolution={r:.1f}: {n_comm:4d} 社区, "
              f"模块度={mod:.4f}, 平均大小={graph.vcount()/n_comm:.1f}")
    
    # 选择策略：
    # 1. 最大模块度对应的分辨率
    best_mod = max(results, key=lambda x: x["modularity"])
    # 2. 社区数量适中的分辨率（经验规则：社区大小在 10-100 节点之间）
    moderate = [r for r in results if 10 <= r["avg_community_size"] <= 100]
    
    if moderate:
        return moderate[len(moderate)//2]["resolution"]
    else:
        return best_mod["resolution"]
```

---

## 8. 能力边界与挑战

### 8.1 社区质量高度依赖图构建质量

```
知识图谱构建质量 → 社区检测质量 → 社区摘要质量 → 检索召回质量

常见问题：
┌──────────────────────────────────────────────────┐
│ 问题1: 关系抽取不完整                              │
│ 真实: A--B, A--C, B--C, C--D                     │
│ 抽取: A--B, A--C, C--D（漏了 B--C）               │
│ 结果: 正确的社区{A,B,C}被错误地划分为{A,B}和{C,D} │
│                                                   │
│ 问题2: 噪声关系（False Positive）                   │
│ 抽取了实际上不存在的错误关系                        │
│ 结果: 不相关的实体被合并到同一社区                   │
│                                                   │
│ 问题3: 实体识别不完整                               │
│ 同一实体被识别为多个不同节点（同义词未合并）          │
│ 结果: 社区被错误地分裂为多个碎片                      │
└──────────────────────────────────────────────────┘
```

### 8.2 重叠社区问题

Louvain/Leiden 都属于**硬聚类** (Hard Clustering) 算法——每个节点只能属于一个社区。但实际知识图谱中，节点通常属于多个语义群组：

```
     ┌──────────┐         ┌──────────┐
     │   基因    │         │   药物    │
     │   研究    │─────────│   研发    │
     └──────────┘    ↑    └──────────┘
                      │
                "EGFR" 同时属于两个领域
                      │
     ┌──────────┐    │    ┌──────────┐
     │   肺癌    │─────────│  靶向    │
     │   诊断    │         │  治疗    │
     └──────────┘         └──────────┘
```

**解决方案**：
1. **重叠社区检测算法**：如 BigCLAM (Cluster Affiliation Model for Big Networks)
2. **层次化软聚类**：在不同粒度上允许不同的社区归属
3. **节点复制**：为跨领域的实体创建"副本"，分到不同社区

### 8.3 社区动态更新（增量社区检测）

知识图谱是动态增长的——新实体和关系不断加入。每次全量运行社区检测的开销巨大：

```
问题: 知识图谱从 100 万节点增长到 110 万（+10%）
方案A: 全量重新运行 Leiden → O(1.1m log 1.1m) 重算全部
方案B: 增量更新 → O(新增节点数 × 局部影响范围)

增量更新的挑战：
1. 新增节点可能影响多个社区的边界
2. 累积的增量更新会导致社区质量下降
3. 需要在"更新频率"和"质量保持"之间权衡

实践中建议：
- 每次小批量更新（如每周）做局部优化
- 每个月做一次全量重新计算
- 监控模块度变化，低于阈值时触发全量重算
```

### 8.4 社区粒度选择对检索效果的敏感度

```
               社区粒度过粗                社区粒度最佳                社区粒度过细
               ───────────                ───────────                ───────────
社区大小:      500+ 节点                   10-50 节点                 1-5 节点
摘要质量:      过于泛化，失去了重要细节     信息密度高，覆盖完整      碎片化，缺乏上下文
检索召回:      高召回但低精确度            高召回 + 高精确度         低召回（错过关联信息）
检索延迟:      快（需检索的摘要少）         中                       慢（需检索的摘要多）
LLM 理解:      容易丢失关键事实             能给出有洞察的分析        太碎片化，无法形成整体认识

经验法则：
- Graph RAG 的社区大小目标：10-50 个实体/社区
- 通过分辨率参数控制：从 1.0 开始，根据知识图谱规模调整
- 根据下游任务验证：衡量社区划分对检索质量的实际影响
```

### 8.5 与其他检索技术的对比

```
社区检索 vs. 其他检索方法的优劣势：

百万节点知识图谱, 查询 "TP53 在肺癌中的作用"

向量检索 (Vector Search):
  ✓ 语义理解强：即使 Query 和 Document 用词不同也能匹配
  ✓ 实现简单：标准 Embedding + ANN 索引
  ✗ 忽略图结构：无法利用关系拓扑信息
  ✗ 上下文碎片化：每个节点的 embedding 独立，缺乏邻域信息

图遍历 (Graph Traversal):
  ✓ 精确关系推理：能给出"TP53 → encodes → p53 → associated_with → Lung Cancer"
  ✗ 计算开销大：多跳遍历在大图上指数级增长
  ✗ 需要精确的起始点：不知道起始实体就无法搜索

社区检索 (Community Retrieval):
  ✓ 结构感知：自动利用图的拓扑结构
  ✓ 层次化：从粗到细，灵活控制粒度
  ✓ 信息密度高：社区摘要浓缩了子图的关键信息
  ✗ 依赖社区质量：社区划分不好 → 检索质量下降
  ✗ 可能丢失跨社区关系：如果相关实体被分到不同社区

最佳实践：社区检索 + 向量检索 混合模式
- 社区检索：提供高层次的领域上下文
- 向量检索：补充社区检索可能漏掉的细节
```

---

## 9. 深入思考

### 9.1 社区发现在 Graph RAG 中的真正价值

社区发现不仅仅是为了"加速检索"，它的更深层价值在于：

1. **结构作为先验知识**：图的拓扑结构本身就蕴含了领域知识——密集连接的节点在语义上相关。社区发现将这种隐式结构显式化，并编码为可供 LLM 消费的摘要。

2. **层次化抽象**：知识是人类通过层次化抽象来理解的（从具体到一般）。社区发现的层次化特性恰好匹配了这种认知模式——从细粒度事实到高层概念。

3. **计算与质量的平衡**：完全基于嵌入的向量检索缺乏结构感知；完全基于图遍历又计算量太大。社区检测找到了一个高效的中间点——用较低的计算成本获取结构化的知识上下文。

### 9.2 社区发现不是银弹

```
社区发现的最大假设：图结构反映语义结构

这个假设在以下情况可能不成立：
1. 图构建有严重偏差时（如关系抽取 F1 < 0.5）
2. 图结构极度稀疏时（平均度数 < 2）
3. 实体类型高度异质（同一社区可能混入完全无关的实体）
4. 跨领域实体数量过多（"桥梁节点"会模糊社区边界）

在这些情况下，可能需要：
- 退回到纯向量检索
- 使用更鲁棒的图嵌入方法（如 GraphSAGE、Node2Vec）
- 引入元路径（Meta-Path）来指导社区划分
```

### 9.3 未来方向

1. **LLM 辅助社区发现**：使用 LLM 的语义理解能力来指导社区划分，而不仅仅依赖图结构
2. **动态社区追踪**：对知识图谱的时序变化进行增量社区检测
3. **可解释社区摘要**：社区摘要不仅要总结实体关系，还要能解释"为什么这些实体属于同一社区"
4. **多模态社区检测**：在图结构之外，融合文本语义、视觉特征等多种模态信息

---

> **总结**：社区发现是将大规模知识图谱转化为 Graph RAG 可用形式的关键技术。Leiden 算法因其连通性保证、更高的模块度质量和更好的稳定性，成为 Graph RAG 场景的首选。Microsoft GraphRAG 利用 Leiden 的层次化特性，构建了从细粒度到粗粒度的社区摘要嵌套索引，以此支持全局查询和局部查询两种检索策略。在实际工程中，需要根据知识图谱的规模和质量，谨慎选择分辨率参数和增量更新策略，平衡社区质量与计算成本。
