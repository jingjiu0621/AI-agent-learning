# 6.4.5 多路检索结果融合（Retrieval Fusion）

## 简单介绍

多路检索结果融合（Retrieval Fusion）是将来自多条独立检索通道的结果合并为一个统一的、优化排序的输出列表的过程。在 AI Agent 的记忆检索系统中，单一的检索策略（如纯向量搜索或纯关键词匹配）各有其不可克服的盲区——语义检索擅长近义表达理解但无法精确匹配专有名词，时间衰减擅长捕获近期记忆但可能遗漏重要的远期信息，重要性评分擅长保留高价值记忆但无法感知当前上下文的即时需求。融合（Fusion）正是为了弥补单一检索通道的固有缺陷，通过综合各路检索信号来获得"1+1>2"的效果。

```
                    ┌──────────────────┐
                    │    Query Input   │
                    └────────┬─────────┘
                             │
             ┌───────────────┼───────────────┐
             ▼               ▼               ▼
     ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
     │ 语义向量检索  │ │ 关键词BM25  │ │ 时间衰减检索 │
     │ (6.4.1)     │ │ (精确匹配)   │ │ (6.4.2)     │
     └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
            │               │               │
            ▼               ▼               ▼
     ┌───────────────────────────────────────────┐
     │         多路结果融合（Retrieval Fusion）     │
     │                                           │
     │  ├── 分数归一化 (Score Normalization)       │
     │  ├── 排名聚合 (Rank Aggregation)           │
     │  └── 加权融合 (Weighted Combination)       │
     └───────────────────┬───────────────────────┘
                         ▼
                 ┌──────────────┐
                 │ 最终排序列表  │
                 │ Top-K 记忆   │
                 └──────────────┘
```

在 Agent 记忆系统的定位中，6.4.5 位于 6.4.1~6.4.4 四种检索策略之后、阈值过滤模块（6.4.6）之前，是整个检索管线的"整合枢纽"。

---

## 基本原理

检索融合的数学模型可以抽象为一个聚合函数：

```
Score_final(item) = AGGREGATE(Score_1(item), Score_2(item), ..., Score_n(item))
```

其中 `Score_i` 来自第 i 条检索通道的原始评分或排名信息，`AGGREGATE` 是融合策略函数。不同的融合方法本质上是 `AGGREGATE` 函数的不同设计选择——核心差异在于：

1. **输入信号的性质**：使用原始分数（score-based）还是仅使用排名位置（rank-based）
2. **通道间关系**：假设各通道独立（线性融合）还是带交互（学习排序）
3. **权重可调性**：权重是否固定、可调、或由数据驱动学习

```
融合方法分类树：

融合方法
├── 基于分数（Score-based）
│   ├── 加权线性融合        Weighted Linear
│   ├── 凸组合融合          Convex Combination
│   └── 最小-最大归一化融合  Min-Max Fusion
│
├── 基于排名（Rank-based）
│   ├── 倒数秩融合          RRF
│   ├── Borda Count
│   └── Condorcet Fusion    (CC)
│
└── 学习式（Learning-based）
    ├── 特征级联排序        Cascade Ranking
    ├── LambdaRank / LambdaMART
    └── 神经网络排序        Neural Ranker
```

---

## 背景与演进

多路结果融合并非 AI Agent 领域的特有概念，其思想源自信息检索（IR）领域数十年的研究成果。

| 阶段 | 时期 | 代表方法 | 核心思想 | 局限性 |
|------|------|----------|----------|--------|
| 简单合并 | 1990s | 直接拼接、取并集 | 各路结果去重后按原始顺序交替排列 | 分数不可比，排序质量差 |
| 加权线性 | 2000s | Min-Max 归一化 + 加权求和 | 将各通道分数缩放到统一范围后线性组合 | 权重依赖人工调参，对离群值敏感 |
| 排名融合 | 2000s | RRF、Borda Count、Condorcet | 绕过分数归一化，直接用排名位置计算融合分 | 忽略分数差异信息（高分与低分同等对待） |
| 统计学习 | 2010s | RankSVM、LambdaRank、LambdaMART | 用标注数据训练排序模型，特征工程驱动 | 需要大量标注数据，泛化依赖特征质量 |
| 深度学习 | 2020s | BERT-rank、ListNet、神经排序 | 端到端学习融合策略，无需手动设计权重 | 计算成本高，可解释性差 |
| Agent 融合 | 当前 | 混合策略 + 自适应权重 | 根据查询类型动态选择融合策略 | 需要运行时决策模块，系统复杂度高 |

关键转折点：

- **RRF 的提出（2009）**：Cormack 等人发现，仅靠排名位置信息就能获得优异融合效果，且无需分数归一化，极大地降低了融合系统的工程复杂度。此后 RRF 成为信息检索融合的事实基线。
- **Condorcet Fusion（2003）**：Montague 和 Aslam 将投票理论引入结果融合，提供了与 RRF 互补的另一种视角。
- **TREC 评测驱动**：历年 TREC（Text REtrieval Conference）的评测任务中，融合模块通常是参赛系统的标配组件，直接推动了多种融合方法的成熟。
- **RAG 与 Agent 的兴起（2023-2025）**：大语言模型的 Agent 应用需要从多种异构知识源（向量 DB、关键词索引、对话历史、工具描述）中检索信息，融合重新成为研究热点。

---

## 核心矛盾：Precision vs. Recall vs. Latency

检索融合本质上是在三个相互冲突的目标之间寻找平衡点：

```
              高精度 (Precision)
                    ▲
                   / \
                  /   \
                 / 理想 \
                /   点   \
               /         \
              /           \
             /─────────────\──▶ 高召回 (Recall)
              \           /
               \         /
                \       /
                 \     /
                  \   /
                   \ /
                    ▼
              低延迟 (Latency)
```

| 目标 | 含义 | 融合策略影响 | 代价 |
|------|------|-------------|------|
| **高精度** | 返回结果中尽量全是相关的 | 提高融合权重中高精度通道的占比、提高截断阈值 | 可能遗漏边缘但相关的记忆 |
| **高召回** | 不遗漏任何相关结果 | 融合所有通道结果、降低截断阈值 | 带来大量噪声，降低最终质量 |
| **低延迟** | 快速返回结果 | 减少融合通道数、使用简单融合方法（如 RRF） | 融合效果有限，无法处理复杂场景 |

在实际的 Agent 系统中，这三者的取舍受应用场景驱动：

- **对话型 Agent**：延迟优先（用户等待 < 500ms），精确度次之，召回最后。推荐使用轻量级 RRF 融合 2-3 路检索结果。
- **分析型 Agent**：精度和召回优先（需要完整的上下文信息），延迟可容忍到秒级。推荐使用加权线性融合或学习排序处理 3-5 路检索结果。
- **批处理 Agent**：召回优先于一切（非实时，关注数据完整性）。推荐所有通道全量融合 + 学习排序重排。

---

## 主流融合方法详解

### 1. 加权线性融合（Weighted Linear Fusion）

加权线性融合是最直观的方法：将各通道的分数进行归一化后，按权重加权求和得到最终得分。

```
Score(item) = Σ w_i × norm(score_i(item))
```

其中 `Σ w_i = 1`，`norm()` 是归一化函数。

#### 归一化方法

由于不同检索通道的原始分数分布不同，归一化是必需的：

| 归一化方法 | 公式 | 特点 |
|-----------|------|------|
| Min-Max | `(s - min) / (max - min)` | 简单，但对离群值敏感 |
| Z-Score | `(s - μ) / σ` | 抗离群，但假设正态分布 |
| Sigmoid | `1 / (1 + e^(-s))` | 将任意分数映射到 (0,1)，适合线性分数 |
| Softmax | `e^(s/T) / Σ e^(s_j/T)` | 带温度 T 控制分布平滑度 |
| 分位数归一化 | `rank(s) / N` | 无分布假设，鲁棒性强 |

#### Python 实现

```python
import numpy as np
from typing import Dict, List, Tuple, Any

class WeightedLinearFusion:
    """
    加权线性融合器
    支持多种归一化策略，支持通道级和查询级权重
    """

    def __init__(
        self,
        weights: Dict[str, float],
        norm_method: str = "minmax"
    ):
        """
        Args:
            weights: 通道名称 -> 权重, 如 {"semantic": 0.5, "keyword": 0.3, "temporal": 0.2}
            norm_method: 归一化方法, 支持 "minmax", "zscore", "sigmoid", "softmax", "quantile"
        """
        self.weights = weights
        self.norm_method = norm_method

    def _normalize(
        self,
        scores: List[float]
    ) -> List[float]:
        """将一组分数归一化到 [0, 1]"""
        scores = np.array(scores, dtype=float)

        if self.norm_method == "minmax":
            smin, smax = scores.min(), scores.max()
            if smax == smin:
                return np.ones_like(scores)
            return (scores - smin) / (smax - smin)

        elif self.norm_method == "zscore":
            mu, sigma = scores.mean(), scores.std()
            if sigma == 0:
                return np.ones_like(scores)
            # 将 z-score 映射到 (0, 1) 范围
            z = (scores - mu) / sigma
            return 1 / (1 + np.exp(-z))

        elif self.norm_method == "sigmoid":
            return 1 / (1 + np.exp(-scores))

        elif self.norm_method == "softmax":
            T = 1.0  # 温度参数
            exp_s = np.exp(scores / T)
            return exp_s / exp_s.sum()

        elif self.norm_method == "quantile":
            ranks = np.argsort(np.argsort(scores))
            return ranks / len(scores)

        else:
            raise ValueError(f"Unsupported norm_method: {self.norm_method}")

    def fuse(
        self,
        channel_results: Dict[str, List[Tuple[str, float]]]
    ) -> List[Tuple[str, float]]:
        """
        融合多通道检索结果

        Args:
            channel_results: {
                "semantic": [("mem_id_1", 0.92), ("mem_id_2", 0.85), ...],
                "keyword":  [("mem_id_2", 12.5), ("mem_id_3", 8.3), ...],
                ...
            }

        Returns:
            [("mem_id_1", 0.87), ("mem_id_3", 0.64), ...]  按最终分数降序
        """
        # Step 1: 收集所有候选记忆 ID
        all_ids = set()
        for channel, results in channel_results.items():
            for mem_id, _ in results:
                all_ids.add(mem_id)

        # Step 2: 归一化各通道分数
        normalized: Dict[str, Dict[str, float]] = {}
        for channel, results in channel_results.items():
            mem_ids = [r[0] for r in results]
            raw_scores = [r[1] for r in results]
            norm_scores = self._normalize(raw_scores)
            normalized[channel] = dict(zip(mem_ids, norm_scores))

        # Step 3: 加权计算融合分数
        final_scores: Dict[str, float] = {}
        for mem_id in all_ids:
            total = 0.0
            for channel, weight in self.weights.items():
                channel_score = normalized.get(channel, {}).get(mem_id, 0.0)
                total += weight * channel_score
            final_scores[mem_id] = total

        # Step 4: 按融合分数降序排列
        sorted_items = sorted(
            final_scores.items(),
            key=lambda x: x[1],
            reverse=True
        )
        return sorted_items


# ========== 使用示例 ==========

if __name__ == "__main__":
    # 模拟三路检索结果
    results = {
        "semantic": [
            ("mem_001", 0.95),   # 语义相关性极高
            ("mem_002", 0.87),
            ("mem_003", 0.72),
            ("mem_004", 0.45),
        ],
        "keyword": [
            ("mem_005", 15.2),   # BM25 分数, 范围 0~20+
            ("mem_003", 12.8),
            ("mem_001", 10.1),
            ("mem_006", 7.5),
        ],
        "temporal": [
            ("mem_007", 0.98),   # 近期记忆
            ("mem_006", 0.91),
            ("mem_001", 0.55),
            ("mem_003", 0.30),
        ],
    }

    # 配置权重: 语义最重要，关键词次之，时间影响最小
    fusion = WeightedLinearFusion(
        weights={"semantic": 0.5, "keyword": 0.3, "temporal": 0.2},
        norm_method="minmax"
    )

    fused = fusion.fuse(results)
    print("融合结果 (加权线性, minmax 归一化):")
    for mem_id, score in fused:
        print(f"  {mem_id}: {score:.4f}")
```

**优点**：
- 实现简单直观，易于调试
- 权重可解释性强（"语义更重要"对应更大的权重）
- 支持多种归一化策略适应不同分数分布

**缺点**：
- 权重高度依赖人工调优
- 归一化方法的选择对结果影响大
- 对离群值（异常高分）敏感
- 线性假设可能不符合实际交互模式

---

### 2. Reciprocal Rank Fusion（RRF）

RRF 是当前工业界最流行的无监督融合方法。它不依赖原始分数，仅利用排名位置信息，因此完全避免了分数归一化的问题。

#### 核心公式

```
RRF_score(item) = Σ 1 / (k + rank_i(item))
```

其中：
- `rank_i(item)` 是 item 在第 i 个检索通道中的排名位置（从 1 开始）
- `k` 是一个平滑常数，通常取 k=60（经典推荐值）

#### 原理分析

RRF 的核心直觉：
1. 排名越靠前的 item，其 1/(k+rank) 越大，对融合得分的贡献越大
2. 常数 k 控制排名下降的惩罚速度：k 越小，高排名和低排名的差异越大；k 越大，排名的差异越被平滑
3. 多条通道中同时排名靠前的 item 得分最高（即"多路共识"效应）

```
k 值影响分析：
k=1   时: rank_1→0.50, rank_10→0.09, rank_100→0.01   (高排名主导)
k=60  时: rank_1→0.016, rank_10→0.014, rank_100→0.006 (平滑过渡)
k=100 时: rank_1→0.010, rank_10→0.009, rank_100→0.005 (高度平滑)

推荐 k 值：k = 60（Cornack et al., SIGIR 2009 经典论文推荐）
```

#### Python 实现

```python
from collections import defaultdict
from typing import Dict, List, Tuple


def reciprocal_rank_fusion(
    channel_rankings: Dict[str, List[str]],
    k: int = 60,
    weights: Dict[str, float] = None,
) -> List[Tuple[str, float]]:
    """
    Reciprocal Rank Fusion (RRF) — 支持加权变体

    Args:
        channel_rankings: 通道名称 -> 有序 ID 列表
            {"semantic": ["mem_001", "mem_003", ...],
             "keyword":  ["mem_005", "mem_001", ...]}
        k: 平滑常数 (经典值 60)
        weights: 可选的通道权重, 默认等权

    Returns:
        [(mem_id, rrf_score), ...] 按 RRF 分数降序排列
    """
    if weights is None:
        weights = {ch: 1.0 for ch in channel_rankings}

    rrf_scores = defaultdict(float)

    for channel, ranking in channel_rankings.items():
        w = weights.get(channel, 1.0)
        for rank, mem_id in enumerate(ranking, start=1):
            rrf_scores[mem_id] += w / (k + rank)

    # 按分数降序排列
    sorted_items = sorted(
        rrf_scores.items(),
        key=lambda x: x[1],
        reverse=True
    )
    return sorted_items


# ========== 加权 RRF 变体 ==========

def weighted_rrf(
    channel_rankings: Dict[str, List[str]],
    channel_scores: Dict[str, List[float]],
    k: int = 60,
    alpha: float = 0.5,
) -> List[Tuple[str, float]]:
    """
    加权 RRF: 结合排名信息和置信度分数

    RRF_score(item) = Σ w_channel / (k + rank_i(item))
    其中 w_channel 基于该通道 top-1 的分数置信度

    alpha 控制排名 vs 置信度的平衡:
    - alpha=0: 纯 RRF (与标准 RRF 等价)
    - alpha=1: 完全由通道置信度加权
    """
    # 计算通道置信度 (基于 top-1 分数)
    conf = {}
    for ch, scores in channel_scores.items():
        if scores:
            conf[ch] = scores[0]  # top-1 分数作为通道置信度
        else:
            conf[ch] = 0.0

    # 归一化置信度
    max_conf = max(conf.values()) if conf else 1.0
    conf = {ch: c / max_conf for ch, c in conf.items()}

    # 融合: 排名分数 × 置信度加权
    scores = defaultdict(float)
    for channel, ranking in channel_rankings.items():
        for rank, mem_id in enumerate(ranking, start=1):
            rank_score = 1.0 / (k + rank)
            weighted = (1 - alpha) * rank_score + alpha * conf[channel] * rank_score
            scores[mem_id] += weighted

    return sorted(scores.items(), key=lambda x: x[1], reverse=True)


# ========== 使用示例 ==========

if __name__ == "__main__":
    rankings = {
        "semantic": ["mem_001", "mem_003", "mem_002", "mem_004"],
        "keyword":  ["mem_005", "mem_001", "mem_003", "mem_006"],
        "temporal": ["mem_007", "mem_006", "mem_001", "mem_003"],
    }

    print("=== 标准 RRF (k=60) ===")
    result = reciprocal_rank_fusion(rankings, k=60)
    for mem_id, score in result:
        print(f"  {mem_id}: {score:.4f}")

    print("\n=== 加权 RRF (alpha=0.3) ===")
    raw_scores = {
        "semantic": [0.95, 0.87, 0.72, 0.45],
        "keyword":  [15.2, 12.8, 10.1, 7.5],
        "temporal": [0.98, 0.91, 0.55, 0.30],
    }
    w_result = weighted_rrf(rankings, raw_scores, k=60, alpha=0.3)
    for mem_id, score in w_result:
        print(f"  {mem_id}: {score:.4f}")

    # 预期输出: mem_001 在三路中都排名 2-3 位, 融合后排名最高
    # mem_003 在两路中出现且排名靠前, 排名第二
```

#### RRF 的直觉理解

```
三路检索结果:
                       语义排名    关键词排名   时间排名
mem_001  ⟶                1           2           3
mem_003  ⟶                2           3           4
mem_006  ⟶                -           4           2

RRF 得分计算 (k=60):
mem_001: 1/(60+1) + 1/(60+2) + 1/(60+3) = 0.0164 + 0.0161 + 0.0159 = 0.0484  ⟶ 最高
mem_003: 1/(60+2) + 1/(60+3) + 1/(60+4) = 0.0161 + 0.0159 + 0.0156 = 0.0476  ⟶ 第二
mem_006: 1/(60+4) + 1/(60+2)             = 0.0156 + 0.0161           = 0.0317  ⟶ 第三

结论: mem_001 因为出现在所有三路中且排名稳定靠前, 获得了最高融合分。
```

**优点**：
- 不需要分数归一化（消除了融合系统中最头疼的问题）
- 实现简单，计算开销极低
- 对离群值和分数分布不敏感
- 效果稳定，在大量实验中被证明接近甚至超过调优后的加权融合

**缺点**：
- 忽略原始分数中的绝对差距信息（如 "相关性极高" 和 "勉强相关" 在 RRF 中只体现为排名差 1）
- k 值选择会影响结果，虽然 k=60 在大部分场景下工作良好
- 当某通道返回结果过少时，未出现的结果被隐含视为"排名无穷大"

---

### 3. Condorcet Fusion（CC / 一致性融合）

Condorcet Fusion 从投票理论中汲取灵感，将每条检索通道视为一个"投票者"，将待排序的记忆视为"候选人"。通过两两比较（pairwise comparison）来确定全局最优排序。

#### 核心思想

Condorcet 准则：如果一个候选人能在与所有其他候选人的一对一比较中获胜（获得更多投票者的支持），那么它应该排在全局第一位。

```
投票类比：
- 每条检索通道 = 一个投票者
- 每个记忆条目 = 一个候选人
- 通道对记忆 A 和 B 的排序 = 投票者对 A 和 B 的偏好

如果 A 在大多数通道中排名高于 B, 则 A "击败" B
如果某记忆击败了所有其他记忆, 它是 Condorcet 赢家
```

#### 数学定义

对于记忆集合中的任意两个条目 A 和 B：

```
C(A, B) = 通道中 rank(A) < rank(B) 的计数
          - 通道中 rank(A) > rank(B) 的计数

最终的全局排序应使 Condorcet 一致性最大化:
- 如果 C(A, B) > 0, 则 A 应排在 B 前面
- 如果 C(A, B) = 0, 则平局
```

#### 实现思路

由于实际中很少存在完美的 Condorcet 赢家（一个击败所有人的记忆），算法通常构造一个**近似解**：

```python
import itertools
from collections import defaultdict
from typing import Dict, List, Tuple


def condorcet_fusion(
    channel_rankings: Dict[str, List[str]]
) -> List[str]:
    """
    Condorcet Fusion: 基于成对比较的投票融合

    使用 Copeland 方法: 统计每个候选人在所有一对一比较中的净胜场次
    """
    # Step 1: 获取位置映射 (通道 -> 记忆 -> 排名)
    positions: Dict[str, Dict[str, int]] = {}
    all_memories = set()

    for channel, ranking in channel_rankings.items():
        pos_map = {}
        for rank, mem_id in enumerate(ranking, start=1):
            pos_map[mem_id] = rank
            all_memories.add(mem_id)
        positions[channel] = pos_map

    memories = list(all_memories)

    # Step 2: 计算成对偏好矩阵
    # wins[A][B] = 认为 A 优于 B 的通道数
    wins = defaultdict(lambda: defaultdict(int))

    for a, b in itertools.combinations(memories, 2):
        for channel, pos_map in positions.items():
            rank_a = pos_map.get(a, float('inf'))
            rank_b = pos_map.get(b, float('inf'))
            if rank_a < rank_b:
                wins[a][b] += 1
            elif rank_b < rank_a:
                wins[b][a] += 1
            # 平局 (两者都未出现时) 不加分

    # Step 3: Copeland 计分 = 净胜场次
    # 有更复杂的变体使用 Schulze 方法或 Ranked Pairs
    copeland_score = {}
    for mem in memories:
        wins_count = sum(
            1 for opponent in memories
            if wins[mem][opponent] > wins[opponent][mem]
        )
        losses_count = sum(
            1 for opponent in memories
            if wins[mem][opponent] < wins[opponent][mem]
        )
        copeland_score[mem] = wins_count - losses_count

    # Step 4: 按 Copeland 分数排序
    # 对分数相同的按原始排名中位数排序（tie-breaking）
    sorted_mems = sorted(
        memories,
        key=lambda m: (
            copeland_score[m],
            -_median_rank(m, positions)  # 中位数排名越高（数字越小）越优先
        ),
        reverse=True
    )

    return sorted_mems


def _median_rank(
    mem_id: str,
    positions: Dict[str, Dict[str, int]]
) -> float:
    """计算某记忆在所有通道中的中位数排名"""
    ranks = [
        pos[mem_id] for pos in positions.values()
        if mem_id in pos
    ]
    if not ranks:
        return float('inf')
    ranks.sort()
    n = len(ranks)
    if n % 2 == 0:
        return (ranks[n // 2 - 1] + ranks[n // 2]) / 2
    return ranks[n // 2]


# ========== 使用示例 ==========

if __name__ == "__main__":
    rankings = {
        "semantic": ["mem_A", "mem_B", "mem_C", "mem_D"],
        "keyword":  ["mem_B", "mem_A", "mem_D", "mem_C"],
        "temporal": ["mem_C", "mem_A", "mem_B", "mem_D"],
    }

    result = condorcet_fusion(rankings)
    print("Condorcet Fusion 结果:")
    for i, mem_id in enumerate(result, 1):
        print(f"  {i}. {mem_id}")
```

**优点**：
- 理论基础坚实，来自社会选择理论（Social Choice Theory）
- 天然处理"多路共识"——不是简单求和，而是考虑相对优劣关系
- 对单通道的异常排序不敏感（通道数越多越鲁棒）

**缺点**：
- 计算复杂度高：O(m × n²)，其中 m 是通道数，n 是候选记忆数
- 存在投票悖论（Condorcet paradox）：A > B, B > C, C > A 的循环可能发生
- 实现复杂度显著高于 RRF 和加权融合
- 当通道数很少时（如 2-3 路），成对比较的意义有限

---

### 4. Borda Count（波达计数法）

Borda Count 是另一种从投票理论借用的方法。每个通道为其排名中的每条记忆分配分数（排名越高分数越多），然后所有通道的分数汇总。

```
Borda 计分规则:
有 N 个候选记忆时:
  rank 1 → N 分
  rank 2 → N-1 分
  ...
  rank N → 1 分
  未出现 → 0 分

融合分数 = Σ Borda_score(item)
```

#### Python 实现

```python
from collections import defaultdict
from typing import Dict, List, Tuple


def borda_fusion(
    channel_rankings: Dict[str, List[str]],
    score_type: str = "standard"
) -> List[Tuple[str, float]]:
    """
    Borda Count 融合

    Args:
        channel_rankings: 通道名称 → 有序 ID 列表
        score_type:
            "standard"  — rank 1 → N, rank N → 1
            "modified"  — rank 1 → 1, rank N → 1/N (类似 RRF 但更平滑)
            "dowdall"   — rank 1 → 1, rank 2 → 1/2, rank 3 → 1/3 ...

    Returns:
        按 Borda 总分降序排列的列表
    """
    all_items = set()
    for ranking in channel_rankings.values():
        all_items.update(ranking)

    n_candidates = len(all_items)
    borda_scores = defaultdict(float)

    for channel, ranking in channel_rankings.items():
        n = len(ranking)

        for rank, mem_id in enumerate(ranking, start=1):
            if score_type == "standard":
                # 总候选 N 个，第 1 名 N 分，第 2 名 N-1 分...
                borda_scores[mem_id] += n_candidates - rank + 1
            elif score_type == "modified":
                # 第 1 名 1 分, 第 N 名 1/N 分
                borda_scores[mem_id] += 1.0 / rank
            elif score_type == "dowdall":
                # 第 1 名 1 分, 第 2 名 1/2 分...
                borda_scores[mem_id] += 1.0 / rank
            else:
                raise ValueError(f"Unknown score_type: {score_type}")

    return sorted(
        borda_scores.items(),
        key=lambda x: x[1],
        reverse=True
    )


# ========== 使用示例 ==========

if __name__ == "__main__":
    rankings = {
        "语义检索": ["mem_A", "mem_B", "mem_C", "mem_D", "mem_E"],
        "关键词":   ["mem_B", "mem_D", "mem_A", "mem_F", "mem_C"],
        "时间衰减": ["mem_C", "mem_A", "mem_G", "mem_B", "mem_D"],
    }

    for stype in ["standard", "modified", "dowdall"]:
        print(f"\nBorda ({stype}):")
        for mem_id, score in borda_fusion(rankings, stype)[:5]:
            print(f"  {mem_id}: {score:.2f}")
```

**优点**：
- 计算简单，复杂度 O(m × n)
- 比 RRF 更公平地对待排名中间位置的记忆（RRF 倾向于高排名）
- 多个变体适应不同需求

**缺点**：
- 标准 Borda 对未出现的记忆分配 0 分，相当于假设它们排在末尾
- 对通道返回的结果数量敏感（结果多的通道影响力更大）
- 无法处理通道间质量差异（所有通道等权）

---

### 5. 学习排序（Learning to Rank, LTR）

学习排序通过机器学习模型学习最优的排序权重，是融合方法中最强大但也最复杂的一类。

#### 三种学习范式

| 范式 | 输入 | 损失函数 | 代表算法 |
|------|------|----------|----------|
| **Pointwise** | 单个记忆 + 标签 | 回归/分类损失 | RankSVM, Subset Ranking |
| **Pairwise** | 记忆对 (A, B) + 相对偏好 | pairwise 分类损失 | RankNet, LambdaRank |
| **Listwise** | 完整列表 + 理想排列 | 列表级距离 | ListNet, LambdaMART, SoftRank |

对于 Agent 记忆检索融合场景，**Pairwise** 和 **Listwise** 方法更为适用，因为它们关注的是排序质量而非绝对分数。

#### 特征设计

特征工程是 LTR 成功的关键。以下是为记忆融合设计的特征模板：

```python
from dataclasses import dataclass, field
from typing import List, Optional


@dataclass
class FusionFeatures:
    """
    记忆检索融合的 LTR 特征模板
    每条候选记忆提取一组特征供排序模型使用
    """
    # === 各通道原始信号 ===
    semantic_score: float = 0.0          # 语义相似度 (0~1)
    keyword_score: float = 0.0           # BM25 分数 (可归一化)
    temporal_score: float = 0.0          # 时间衰减分数 (0~1)
    importance_score: float = 0.0        # 重要性评分 (0~1)

    # === 排名特征 (RRF 风格) ===
    semantic_rank: int = 999             # 语义检索的排名位置
    keyword_rank: int = 999              # 关键词检索的排名位置
    temporal_rank: int = 999             # 时间衰减排名的位置
    min_rank: int = 999                  # 各通道中最高的排名 (数值最小)
    num_channels_appeared: int = 0       # 出现在几个通道中

    # === 交叉特征 (Cross-channel) ===
    semantic_keyword_diff: float = 0.0   # 语义分与关键词分的差异
    score_variance: float = 0.0          # 各通道分数的方差 (高方差 = 不一致)
    rank_product: float = 0.0            # 不同通道排名的几何均值

    # === 元特征 (Meta) ===
    memory_age_hours: float = 0.0        # 记忆创建至今的小时数
    memory_access_count: int = 0         # 记忆被检索的次数
    is_recent: int = 0                   # 是否近 1 小时内创建
    query_length: int = 0                # 查询文本长度
    memory_length: int = 0               # 记忆文本长度

    def to_vector(self) -> List[float]:
        """将所有特征展平为向量"""
        return [
            self.semantic_score,
            self.keyword_score,
            self.temporal_score,
            self.importance_score,
            1.0 / (1.0 + self.semantic_rank),     # 转换成 RRF 风格分
            1.0 / (1.0 + self.keyword_rank),
            1.0 / (1.0 + self.temporal_rank),
            1.0 / (1.0 + self.min_rank),
            self.num_channels_appeared / 5.0,       # 假设最多 5 通道
            self.semantic_keyword_diff,
            self.score_variance,
            1.0 / (1.0 + self.rank_product),
            self.memory_age_hours / 8760.0,          # 归一化到年
            min(self.memory_access_count / 100, 1.0),
            self.is_recent,
            self.query_length / 100,
            self.memory_length / 500,
        ]
```

#### 训练管线示例

```python
import numpy as np
from typing import List, Dict, Any
# 这里使用 XGBoost 的 LambdaMART 作为排序模型示例
# 实际生产中可以替换为任何 LTR 模型


class LambdaMARTFusion:
    """
    基于 LambdaMART (XGBoost) 的学习排序融合器

    LambdaMART 是当前工业界最流行的 LTR 算法:
    - 使用 Gradient Boosting 框架
    - 损失函数基于 pairwise 排序误差的 Lambda 梯度
    - 支持 listwise 的 NDCG/MAP 优化目标
    """

    def __init__(
        self,
        model_path: Optional[str] = None,
        objective: str = "rank:ndcg",
        eval_metric: str = "ndcg@5",
    ):
        import xgboost as xgb
        if model_path:
            self.model = xgb.Booster()
            self.model.load_model(model_path)
        else:
            self.model = None

        self.params = {
            "objective": objective,
            "eval_metric": eval_metric,
            "eta": 0.1,
            "max_depth": 4,
            "subsample": 0.8,
            "colsample_bytree": 0.8,
            "seed": 42,
        }

    def extract_features(
        self,
        query_text: str,
        memory_id: str,
        channel_scores: Dict[str, float],
        channel_ranks: Dict[str, int],
        memory_meta: Dict[str, Any],
    ) -> FusionFeatures:
        """为单条记忆提取特征"""
        features = FusionFeatures()

        features.semantic_score = channel_scores.get("semantic", 0.0)
        features.keyword_score = channel_scores.get("keyword", 0.0)
        features.temporal_score = channel_scores.get("temporal", 0.0)
        features.importance_score = channel_scores.get("importance", 0.0)

        features.semantic_rank = channel_ranks.get("semantic", 999)
        features.keyword_rank = channel_ranks.get("keyword", 999)
        features.temporal_rank = channel_ranks.get("temporal", 999)

        ranks = [
            r for r in [
                features.semantic_rank,
                features.keyword_rank,
                features.temporal_rank,
            ] if r < 999
        ]
        features.min_rank = min(ranks) if ranks else 999
        features.num_channels_appeared = len(ranks)

        # 交叉特征
        features.semantic_keyword_diff = (
            features.semantic_score - features.keyword_score
        )
        scores = [
            s for s in [
                features.semantic_score,
                features.keyword_score,
                features.temporal_score,
            ] if s > 0
        ]
        features.score_variance = np.var(scores) if len(scores) > 1 else 0.0

        import math
        ranks_safe = [max(r, 1) for r in ranks]
        features.rank_product = (
            math.prod(ranks_safe) ** (1.0 / len(ranks_safe))
            if ranks_safe else 999
        )

        # 元特征
        features.memory_age_hours = memory_meta.get("age_hours", 0)
        features.memory_access_count = memory_meta.get("access_count", 0)
        features.is_recent = 1 if features.memory_age_hours < 1 else 0
        features.query_length = len(query_text)
        features.memory_length = memory_meta.get("text_length", 0)

        return features

    def rank(
        self,
        query_text: str,
        candidates: List[str],
        channel_scores: Dict[str, Dict[str, float]],
        channel_ranks: Dict[str, Dict[str, int]],
        memory_metas: Dict[str, Any],
    ) -> List[Tuple[str, float]]:
        """
        使用训练好的 LTR 模型对候选记忆进行排序

        Returns:
            [(mem_id, predicted_score), ...]
        """
        if self.model is None:
            raise ValueError("Model not loaded. Call train() or load_model() first.")

        import xgboost as xgb

        feature_vectors = []
        valid_ids = []

        for mem_id in candidates:
            meta = memory_metas.get(mem_id, {})
            scores = {ch: ch_scores.get(mem_id, 0.0)
                      for ch, ch_scores in channel_scores.items()}
            ranks = {ch: ch_ranks.get(mem_id, 999)
                     for ch, ch_ranks in channel_ranks.items()}
            feat = self.extract_features(query_text, mem_id, scores, ranks, meta)
            feature_vectors.append(feat.to_vector())
            valid_ids.append(mem_id)

        if not feature_vectors:
            return []

        dmat = xgb.DMatrix(np.array(feature_vectors))
        predictions = self.model.predict(dmat)

        result = sorted(
            zip(valid_ids, predictions),
            key=lambda x: x[1],
            reverse=True
        )
        return result

    def train(
        self,
        train_groups: List[Dict],
        val_groups: List[Dict],
        num_rounds: int = 100,
    ):
        """
        训练 LambdaMART 模型

        train_groups: 每个元素是 {"features": [[...], ...],
                                    "labels": [0/1/2...],
                                    "group": [N]}  # 每个 query 的文档数
        """
        import xgboost as xgb

        # 组装训练数据
        all_features = []
        all_labels = []
        all_groups = []

        for group in train_groups:
            all_features.extend(group["features"])
            all_labels.extend(group["labels"])
            all_groups.append(group["group"])

        dtrain = xgb.DMatrix(
            np.array(all_features),
            label=np.array(all_labels)
        )
        dtrain.set_group(all_groups)

        # 验证集同理
        val_features = []
        val_labels = []
        val_groups = []
        for group in val_groups:
            val_features.extend(group["features"])
            val_labels.extend(group["labels"])
            val_groups.append(group["group"])

        dval = xgb.DMatrix(
            np.array(val_features),
            label=np.array(val_labels)
        )
        dval.set_group(val_groups)

        self.model = xgb.train(
            self.params,
            dtrain,
            num_boost_round=num_rounds,
            evals=[(dtrain, "train"), (dval, "val")],
            early_stopping_rounds=10,
            verbose_eval=10,
        )
        return self
```

**优点**：
- 理论上可获得最优的融合效果（上限最高）
- 可灵活引入大量特征，捕捉通道间的复杂交互
- 端到端优化最终的排序指标（NDCG, MAP）

**缺点**：
- 需要大量标注数据（记忆相关性标签）
- 训练和部署成本高
- 特征工程工作量大
- 模型需要定期重新训练以适应数据分布变化
- 在小规模场景下效果可能不如精心调参的 RRF

---

### 6. 其他融合方法概览

除上述主要方法外，还有一些值得了解的融合方法：

| 方法 | 核心思想 | 特点 |
|------|----------|------|
| **CombMNZ** | 文档出现次数 × 分数求和 | 简单但有效，对多路共现结果有强化效应 |
| **线性插值 (Convex Comb.)** | `α×Score_A + (1-α)×Score_B` | 两路融合的标准方法，α 可动态调整 |
| **均值融合 (Average)** | 各路分数的算术/几何平均 | 极端简单，无权重，适合快速原型 |
| **中位数融合 (Median)** | 各路分数的中位数 | 抗离群值，对通道数有要求 |
| **最大融合 (Max)** | 取各路分数的最大值 | 适合"只要一路匹配就算"的场景 |
| **级联融合 (Cascade)** | 粗筛 + 精排的分阶段融合 | 先用低成本通道粗筛，再用高成本通道精排 |
| **自适应融合** | 根据查询类型动态选择方法 | 基于意图分类器，是最灵活但也最复杂的方案 |

---

## 方法对比表格

| 维度 | 加权线性 | RRF | Borda Count | Condorcet | LTR/LambdaMART |
|------|----------|-----|-------------|-----------|----------------|
| **输入需求** | 分数 + 权重 | 排名 | 排名 | 排名 | 多通道特征 |
| **分数归一化** | 必需 | 不需要 | 不需要 | 不需要 | 内部处理 |
| **实现复杂度** | 低 | 低 | 低 | 中 | 高 |
| **计算复杂度** | O(m×n) | O(m×n) | O(m×n) | O(m×n²) | O(m×n + 推理) |
| **调参难度** | 中（权重 + 归一化） | 低（k 一个参数） | 低（计分方式） | 低（无需显式参数） | 高（特征 + 超参） |
| **标注数据需求** | 无 | 无 | 无 | 无 | 大量 |
| **适应性** | 固定 | 固定 | 固定 | 固定 | 数据驱动更新 |
| **解释性** | 高 | 高 | 高 | 中 | 低（黑盒） |
| **效果上限** | 中 | 中高 | 中 | 中高 | 高 |
| **典型延迟** | <1ms | <1ms | <1ms | 1-10ms (n 大时) | 5-50ms |
| **对通道质量差异** | 敏感（权重调节） | 不敏感 | 不敏感 | 不敏感 | 自动学习 |
| **对离群值** | 敏感 | 不敏感 | 不敏感 | 有一定容忍度 | 取决于特征设计 |

---

## 融合策略选择决策指南

以下是一个决策流，帮助根据实际场景选择合适的融合方法：

```
开始:
  是否需要标注数据?
  ├── 否 ────────────── 进入无监督分支
  │   ├── 是否有分数可比?
  │   │   ├── 是 → 加权线性融合 (分数分布稳定时)
  │   │   └── 否 → 进入排名分支
  │   │       ├── 通道数 ≥ 5?
  │   │       │   ├── 是 → Condorcet (发挥投票优势)
  │   │       │   └── 否 → RRF (通用推荐)
  │   │       ├── 候选记忆数少(<50)且需要稳定排序?
  │   │       │   └── 是 → Borda Count
  │   │       └── 需要快速原型?
  │   │           └── 是 → RRF (k=60, 开箱即用)
  │   │
  │   └── 快速场景判断:
  │       实时对话回复 (>500ms 不可接受) → RRF
  │       非实时批处理 → 加权线性或 Condorcet
  │       边缘设备 (计算受限) → RRF 或 Borda
  │
  └── 是 ────────────── 进入有监督分支
      ├── 是否有大量历史日志?
      │   ├── 是 → LTR (LambdaMART/神经排序)
      │   │   ├── 特征丰富 → LambdaMART (XGBoost)
      │   │   ├── 特征极少 → RankNet (pairwise)
      │   │   └── 需要端到端 → ListNet/SoftRank
      │   └── 否 → 仍使用无监督方法, 同时积累数据
      │
      ├── 是否需要动态适应?
      │   └── 是 → 在线学习排序 (Online LTR)
      │
      └── 数据标注方式:
          隐式反馈 (点击/选择) → Pairwise LTR
          显式评分 (1-5) → Pointwise LTR
          排序对比 (A/B test logs) → Listwise LTR
```

---

## 工程优化

### 1. 分数归一化实践

分数归一化是加权线性融合中的关键步骤。以下是一些工程经验：

```python
class RobustNormalizer:
    """
    健壮的分数归一化器
    使用分位数裁剪 + Min-Max 的组合策略
    """

    def __init__(self, lower_quantile: float = 0.05, upper_quantile: float = 0.95):
        self.lower_q = lower_quantile
        self.upper_q = upper_quantile

    def normalize(self, scores: List[float]) -> List[float]:
        """分位数裁剪 + Min-Max 归一化"""
        arr = np.array(scores)

        # 1. 裁剪极端值
        lo = np.quantile(arr, self.lower_q)
        hi = np.quantile(arr, self.upper_q)
        clipped = np.clip(arr, lo, hi)

        # 2. Min-Max 归一化
        cmin, cmax = clipped.min(), clipped.max()
        if cmax == cmin:
            return np.ones_like(clipped)
        return (clipped - cmin) / (cmax - cmin)


# 归一化对融合结果的影响示例
def demonstrate_normalization_impact():
    """
    展示归一化的重要性:
    假设语义分数范围 [0.9, 0.95], 关键词分数范围 [1, 20]
    如果不归一化, 关键词通道将完全主导结果
    """
    semantic_scores = [0.90, 0.92, 0.95, 0.88]
    keyword_scores = [1.5, 20.0, 3.2, 15.0]

    # 不归一化直接加权 (50/50)
    raw_fused = [
        0.5 * s + 0.5 * k
        for s, k in zip(semantic_scores, keyword_scores)
    ]
    # raw_fused: [1.20, 10.46, 2.08, 7.94]
    # 完全由 keyword 主导!

    # 归一化后加权 (50/50)
    normalizer = RobustNormalizer()
    sem_norm = normalizer.normalize(semantic_scores)
    kw_norm = normalizer.normalize(keyword_scores)
    norm_fused = [0.5 * s + 0.5 * k for s, k in zip(sem_norm, kw_norm)]
    # 两组特征对最终结果有更均衡的影响

    return norm_fused
```

### 2. 性能优化

| 策略 | 方法 | 效果 |
|------|------|------|
| **预计算排名** | 各通道的检索结果排名可缓存，仅当记忆库更新时刷新 | 减少融合阶段计算量 |
| **惰性融合 (Lazy Fusion)** | 只对 Top-K 结果进行融合，不计算所有候选 | 将 n 从数百万降到数十 |
| **局部截断** | 各通道只返回 Top-N 用于融合（如 N=50） | 控制融合阶段输入规模 |
| **分桶融合** | 将记忆按类别分桶，各桶内独立融合再合并 | 支持并行加速 |
| **向量化计算** | 使用 numpy 数组运算替代 Python 循环 | 10-100x 加速 |

```python
import numpy as np
from typing import Dict, List, Tuple


def fast_rrf_vectorized(
    rankings: Dict[str, np.ndarray],  # 每个通道的排名数组, shape=(n,)
    k: int = 60
) -> np.ndarray:
    """
    向量化 RRF 实现

    Args:
        rankings: {channel: np.array([mem_id_indices])}
                  注意: 这里 mem_id 已预先映射为整数索引
        k: 平滑常数

    Returns:
        np.array 融合分数, 按 mem_id 索引排列
    """
    # 假设总的独特记忆数已知
    n_mems = max(
        idx.max() for idx in rankings.values()
    ) + 1 if rankings else 0

    scores = np.zeros(n_mems, dtype=np.float32)

    for channel, indices in rankings.items():
        ranks = np.arange(1, len(indices) + 1, dtype=np.float32)
        scores[indices] += 1.0 / (k + ranks)

    return scores


def lazy_rrf(
    rankings: Dict[str, List[str]],
    top_k: int = 20,
    k: int = 60
) -> List[Tuple[str, float]]:
    """
    惰性 RRF: 只关注排名靠前的记忆
    对于大部分场景, 排名 > 50 的记忆对 Top-K 最终结果几乎没有影响
    """
    from collections import defaultdict

    scores = defaultdict(float)

    for channel, ranking in rankings.items():
        # 每个通道只取前 top_k 个
        for rank, mem_id in enumerate(ranking[:top_k], start=1):
            scores[mem_id] += 1.0 / (k + rank)

    return sorted(scores.items(), key=lambda x: x[1], reverse=True)[:top_k]
```

### 3. 缓存策略

```python
class FusionCache:
    """
    融合结果缓存

    缓存命中时跳过融合计算, 适用于:
    - 短时间内相同查询的重复检索
    - 同一会话中上下文相似的查询
    """

    def __init__(self, ttl_seconds: int = 60, max_size: int = 1000):
        from collections import OrderedDict
        import time

        self.cache = OrderedDict()
        self.ttl = ttl_seconds
        self.max_size = max_size
        self._time = time

    def get(self, query_hash: str) -> List[Tuple[str, float]] | None:
        if query_hash not in self.cache:
            return None

        timestamp, result = self.cache[query_hash]
        if self._time.time() - timestamp > self.ttl:
            del self.cache[query_hash]
            return None

        # LRU: 移动到末尾
        self.cache.move_to_end(query_hash)
        return result

    def set(self, query_hash: str, result: List[Tuple[str, float]]):
        # 淘汰策略: TTL + LRU
        while len(self.cache) >= self.max_size:
            self.cache.popitem(last=False)

        self.cache[query_hash] = (self._time.time(), result)
```

---

## 能力边界

检索融合并非万能，在以下场景中它可能效果有限甚至适得其反：

### 1. 所有通道质量都很差

融合不会创造新信息。如果所有检索通道的质量都很差（embedding 模型训练不足、关键词索引不完整、时间衰减参数错误），融合只会生成一个"精心排序的糟糕结果"。此时应优先优化各通道的独立检索质量，而非在融合层做文章。

### 2. 通道间高度冗余

当多个通道返回高度相似的结果时（如语义检索和关键词检索在大部分查询上都给出相同的 Top-K），融合的边际收益接近于零。应简化通道设计而非增加融合复杂度。

```
症状:
- 各路结果的重叠率 > 70%
- 加入新通道后排序结果几乎不变
- 融合权重对结果不敏感（改变权重不改变排序）

对策:
- 合并冗余通道
- 检查各通道的差异化设计
- 使用多样性和新颖性指标来评估通道互补性
```

### 3. 查询意图极度明确

当查询本身就极其明确（如精确 ID 查询、时间范围已经很窄、或唯一匹配），单通道已经能返回完美结果，融合只会增加延迟。此时应使用"短路机制"跳过融合。

### 4. 实时性要求极高

在延迟预算低于 50ms 的场景中（如高频交易 Agent、实时语音 Agent），运行多路检索已属奢侈，再加上融合层更不可接受。

### 5. 缺乏评估标准

没有明确的排序质量指标时，无法判断融合是否改善了结果。盲目使用复杂融合方法会让问题更加不可诊断。

---

## 最大挑战

### 挑战 1：分数归一化的两难困境

这是融合系统中最根本、最棘手的工程问题。不同检索通道的原始分数具有不同的统计分布：

| 通道 | 分数范围 | 分布特征 | 典型值 |
|------|----------|----------|--------|
| 余弦相似度 | [0, 1] | 集中在 0.5-0.9 | 0.78 |
| BM25 | [0, 30+] | 长尾分布，高值稀疏 | 3-15 |
| 时间衰减 | [0, 1] | 通常非 0 即 1（大部分记忆接近 1） | 0.95 |
| 重要性分数 | [0, 1] | U 型分布（两极分化） | 0.2 or 0.8 |

归一化的困难在于：
- 没有"正确"的归一化方法——Min-Max 对离群值敏感，Z-Score 假设正态分布
- 归一化参数的在线估计具有滞后性
- 分布会随时间变化（记忆库的统计特性演变）

**工程建议**：若无明确理由，优先使用 RRF（绕过分归一化）。如果必须使用分数融合，推荐**分位数归一化 + 分位数裁剪**的组合。

### 挑战 2：权重调优的维度灾难

在加权融合中，m 个通道就有 m 个权重参数。当 m > 3 时，手动调参已不可行，而搜索空间随 m 指数增长。

**工程建议**：
- m ≤ 3 时：网格搜索 + 离线评测
- 3 < m < 7：贝叶斯优化或随机搜索
- m ≥ 7：使用 LTR 自动学习权重，或降维合并通道

### 挑战 3：性能与质量的平衡

每多一个通道，检索管线的延迟就增加一次查询的时间。融合阶段本身也有计算开销。端到端系统必须平衡检索深度（每条通道的返回数）和融合质量。

```
检索深度 vs. 融合质量经验法则:
- 如果各通道 Top-5 的重叠率 > 50%: 设置检索深度 = 10 就足够
- 如果各通道 Top-20 的重叠率 < 20%: 设置检索深度 ≥ 50
- 通用的起始配置: 检索深度 = 30, 融合后截断 = 10
```

### 挑战 4：冷启动与数据稀疏

启动融合系统的最大困难是缺乏评估数据：
- 无监督方法（RRF、Borda）可以快速上线
- 监督方法（LTR）需要积累成百上千条标注样本
- 权重的迭代优化需要在线 A/B 测试

---

## 场景判断

### 何时使用加权线性融合

- 各通道分数范围已知且稳定
- 有明确的业务优先级（如"语义匹配比时间重要 2 倍"）
- 需要可解释的排序（可以向用户解释为什么某记忆排在前面）
- 通道数 ≤ 3

### 何时使用 RRF

- **几乎可以作为融合的默认首选方法**
- 各通道分数不可比或分布差异大
- 需要快速工程实现（几分钟即可集成）
- 效果基线：RRF 通常在无监督方法中表现最佳
- 通道数 2-5 为最佳工作范围

### 何时使用 Condorcet Fusion

- 通道数 ≥ 5 时能发挥投票理论的统计优势
- 需要处理通道间的偏好冲突
- 排序质量对"记忆间的相对优先级"更敏感而非绝对分数
- 候选记忆数量可控（n < 200）

### 何时使用 Borda Count

- 需要比 RRF 更平滑地对待排名中间位置的记忆
- 各通道返回的结果数量均衡
- 作为 RRF 的对比基准

### 何时使用学习排序 (LTR)

- 有充足的历史交互数据和标注样本
- 系统需要长期迭代优化（LTR 的初始投入需要通过持续改进回收）
- 特征丰富（能提取 10+ 维度的有意义特征）
- 效果是目前最优结果（已经用 RRF 达到瓶颈）之后——不要跳过 RRF 直接上 LTR
- 有离线评估 + 在线 A/B 测试的基础设施

### 快速选择表

```
条件                                  → 推荐方法
─────────────────────────────────────────────────────────
首次实现，没有任何经验                     → RRF (k=60)
分数分布稳定，权重明确                     → 加权线性
需要极低延迟 (<5ms)                      → RRF 或简单均值
延迟容忍度高 (>100ms), 效果优先           → LTR
通道数量多 (≥5)                          → Condorcet 或 RRF
通道返回数量差异大                         → RRF (抵抗通道偏差)
需要向用户解释排序原因                     → 加权线性 (权重可展示)
边缘设备 / 计算资源受限                    → RRF 或 Borda
效果已到瓶颈，需要突破                     → LTR (LambdaMART)
无标注数据但希望动态适应                    → 有界在线 RRF 变体
做研究 / 需要多方法对比基准                 → 全部实现, 择优部署
```

### 混合策略建议

在实际生产系统中，**单一融合方法往往不够**。推荐采用分层/混合策略：

```python
def hybrid_fusion_pipeline(
    query: str,
    channel_results: Dict[str, List[Tuple[str, float]]],
    query_type: str,  # "factual" | "conversational" | "analytical" | "exploratory"
) -> List[Tuple[str, float]]:
    """
    混合融合策略:
    根据查询类型动态选择融合方法
    """
    if query_type == "factual":
        # 事实型查询: 精确匹配优先
        # 使用加权线性, 提高 keyword 权重
        return WeightedLinearFusion(
            weights={"semantic": 0.3, "keyword": 0.5, "temporal": 0.2}
        ).fuse(channel_results)

    elif query_type == "conversational":
        # 对话型查询: 需要快速语义匹配
        # RRF 是最平衡的选择
        rankings = {
            ch: [r[0] for r in results]
            for ch, results in channel_results.items()
        }
        return reciprocal_rank_fusion(rankings, k=60)

    elif query_type == "analytical":
        # 分析型查询: 精度优先, 容忍更高延迟
        # 使用 Condorcet 利用多通道投票
        rankings = {
            ch: [r[0] for r in results]
            for ch, results in channel_results.items()
        }
        return condorcet_fusion(rankings)

    elif query_type == "exploratory":
        # 探索型查询: 召回优先
        # 使用低阈值加权融合 + 多样性后处理
        return WeightedLinearFusion(
            weights={"semantic": 0.4, "keyword": 0.3, "temporal": 0.3},
            norm_method="softmax"
        ).fuse(channel_results)

    else:
        # 默认回退
        rankings = {
            ch: [r[0] for r in results]
            for ch, results in channel_results.items()
        }
        return reciprocal_rank_fusion(rankings, k=60)
```

---

## 与上游模块的交互

- **输入来源**：语义检索（6.4.1）提供向量相似度分数和排名、时间衰减（6.4.2）提供时效分数和排名、重要性评分（6.4.3）提供静态排序分数、多模态检索（6.4.4）提供跨模态匹配分数
- **输出目标**：将融合后的有序列表传递给阈值过滤模块（6.4.6），后者根据自适应阈值做最终截断
- **数据库交互**：融合阶段的候选记忆 ID 列表可用于批量化向记忆存储发起嵌入向量或元数据的回查（batch lookup），以获取更多排序特征
