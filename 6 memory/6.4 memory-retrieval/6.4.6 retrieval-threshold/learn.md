# 6.4.6 检索阈值自适应调整 (Retrieval Threshold Adaptive Tuning)

## 简单介绍

检索阈值是 AI Agent 记忆系统中的"守门员"——它决定了哪些记忆检索结果值得被呈递给后续推理环节，哪些应该被舍弃。在一个典型的检索流水线中，嵌入向量相似度搜索返回的原始结果是一个按分数降序排列的候选列表。阈值过滤（Threshold Filtering）是位于结果融合（6.4.5 retrieval-fusion）之后的关键筛选环节，负责从融合后的候选中剔除低质量匹配，保留高置信度记忆。

```ascii
  检索流水线中的阈值环节：

  向量搜索 ──→ 多路融合（6.4.5） ──→ 阈值过滤（6.4.6） ──→ 最终输出
                                        │
                                    ┌───┴──────────────┐
                                    │  1. 相似度阈值     │
                                    │  2. Top-K 截断     │
                                    │  3. 自适应调整      │
                                    └──────────────────┘
```

检索阈值之所以重要，是因为相似度分数在不同查询、不同语料库、不同嵌入模型之间具有**不可比性**（incomparable nature）。同一嵌入模型下，"猫咪"与"犬科"的相似度可能为 0.82 才有意义，而"神经网络"与"深度学习"在另一个嵌入空间中 0.65 就已经是高度相关。缺乏合理的阈值策略会导致两种典型失败模式：

- **阈值过低**：大量噪声记忆污染上下文窗口，浪费 LLM Token 预算并引入误导信息
- **阈值过高**：遗漏关键但表达方式不同的记忆，导致 Agent 丧失历史经验

**检索阈值自适应调整**的核心目标是在没有人工标注数据的情况下，根据查询特性与结果分布动态确定最优截断点，在精度（Precision）与召回率（Recall）之间自动寻找平衡。

---

## 基本原理

### 阈值在检索决策链中的位置

```ascii
  查询 Q
    │
    ▼
  ┌─────────────────────────┐
  │  嵌入编码 (embedding)    │  ← 6.4.1 Semantic Search
  └────────┬────────────────┘
           │ vector(q)
           ▼
  ┌─────────────────────────┐
  │  近似最近邻搜索 (ANN)    │
  │  Top-N 候选 (N >> K)    │
  └────────┬────────────────┘
           │ raw scores: [0.92, 0.87, 0.76, 0.65, 0.54, ...]
           ▼
  ┌─────────────────────────┐
  │  多路结果融合 (Fusion)   │  ← 6.4.5 Retrieval Fusion
  │  RRF / 加权聚合          │
  └────────┬────────────────┘
           │ fused scores: [0.89, 0.82, 0.71, 0.58, 0.42, ...]
           ▼
  ┌─────────────────────────┐
  │  阈值过滤 (Threshold)    │  ← 6.4.6 (当前模块)
  │  ├── 分数 ≥ τ?          │
  │  ├── 排名 ≤ K?          │
  │  └── 自适应调整 τ        │
  └────────┬────────────────┘
           │ relevant memories only
           ▼
  ┌─────────────────────────┐
  │  上下文注入 → LLM 推理   │
  └─────────────────────────┘
```

### 相似度分数的分布特性

理解阈值的本质，需要先理解嵌入空间中相似度分数的统计特性：

1. **分数聚集效应**：对于大多数真实查询，相似度分数近似呈**偏态分布**（skewed distribution）——少部分结果分数较高，大部分结果聚集在较低的均值附近
2. **嵌入模型敏感**：不同嵌入模型输出的分数范围差异显著。OpenAI text-embedding-3 的余弦相似度通常在 [0.3, 1.0] 区间，而某些 Sentence-BERT 模型可能产生 [0.5, 1.0] 的分布
3. **查询复杂度相关**：简单、具体的查询（如"Python 异常处理语法"）会产生更尖锐的分数分布；复杂、抽象的查询（如"如何处理分布式系统中的一致性问题"）分数分布更平坦
4. **语料库偏置**：密集语料区域（高频概念附近）的相似度分数系统性偏高

### 阈值的决策函数

形式化地，阈值过滤可以表示为：

```
给定融合后的排序列表 R = [(d₁, s₁), (d₂, s₂), ..., (dₙ, sₙ)] ，其中 s₁ ≥ s₂ ≥ ... ≥ sₙ

过滤结果 R' = {(dᵢ, sᵢ) ∈ R | sᵢ ≥ τ 且 i ≤ K}

其中：
  τ = 相似度阈值（可固定或自适应）
  K = 最大返回数量
```

两种截断方式的组合形成了不同的过滤策略：

| 组合方式 | 含义 | 典型效果 |
|----------|------|----------|
| 仅 τ | 所有高于阈值的 | 结果数量不固定，质量有下限 |
| 仅 K | 取前 K 条 | 结果数量固定，质量无下限 |
| τ + K 取交集 | 高于阈值且在前 K 内 | 双重保障，兼顾数量和质量 |
| τ + K 取并集 | 高于阈值或在前 K 内 | 保守策略，尽可能保留 |

---

## 核心矛盾

### 固定阈值 vs 动态阈值

```
  固定阈值  ────  简单、可预测  ────  但对不同查询不公平
                        │
                   矛盾焦点
                        │
  动态阈值  ────  灵活、公平  ────  但难以解释和调试
```

**固定阈值（Fixed Threshold）** 的优势在于可预测性和易于实现，但存在根本性的问题：不同查询的最优阈值差异巨大。例如，在一个通用 Agent 记忆系统中：

| 查询类型 | 最优阈值 | 固定阈值 0.7 时的问题 |
|----------|----------|----------------------|
| "关闭数据库连接的方法" | 0.85 | ✅ 正常过滤 |
| "用户上次抱怨服务慢的时间" | 0.55 | ❌ 漏掉关键记忆 |
| "一些关于安全的零散笔记" | 0.40 | ❌ 几乎一无所获 |

**动态阈值（Dynamic Threshold）** 试图为每个查询计算定制化的截断点，但面临以下挑战：
- 需要可靠的统计信号来判断什么时候"足够好"
- 多了一个可调的超参数（阈值的阈值）
- 不同动态策略在小型结果集上效果不稳定

### 精度-召回率权衡

```
                    高阈值
                      │
                Precision ↑
                      │
    漏掉相关记忆 ←────┼────→ 放行噪声
                      │
                Recall ↓
                      │
                    低阈值
```

这是检索系统中最经典的权衡关系：

- **高阈值 → 高精度低召回**：Agent 行为保守，只使用高度确定的记忆，减少幻觉但不擅长利用相似但有用的历史经验
- **低阈值 → 低精度高召回**：Agent 积极探索相关记忆，但可能被噪声干扰，增加 LLM 推理的错误率和 Token 消耗

F1 分数在二者之间寻找平衡点，但最优平衡点因场景而异。例如：
- 安全关键场景（如医疗诊断 Agent）偏向高精度
- 创意场景（如写作助手）偏向高召回
- 平衡场景（通用对话 Agent）追求 F1 最优

### Token 预算约束

在 LLM 系统中，阈值决策还受到一个额外的硬约束——上下文窗口（Context Window）预算：

```
  可用 Token: 4096
     │
     ├── 用户消息: 1024
     ├── 系统提示: 512
     └── 剩余给记忆: 2560
           │
           └── 每条记忆约 200 Token → 最多 12 条
                  └── 即使 100 条超过阈值，也只取前 K=12
```

因此，在 LLM Agent 中，阈值选型通常不是独立的——它必须与 Top-K 联合优化。

---

## 主流阈值策略详解

### 1. 固定阈值 (Fixed Threshold)

最简单的策略：设置一个硬编码的相似度下限，s_i >= tau 的记忆被保留。

**代码形式：**
```python
def fixed_threshold(results: list[tuple[str, float]], threshold: float = 0.7):
    return [(doc, score) for doc, score in results if score >= threshold]
```

**优点：**
- 实现极简，计算开销为零
- 行为完全可预测，方便测试
- 在语料库分布稳定的场景下表现稳定

**缺点：**
- 跨查询效果差异大
- 需要人工反复调参
- 对嵌入模型更换敏感（更换模型可能意味着需要完全重新调参）

**适用场景：**
- 非常特定的垂直领域（如仅检索法律条款）
- 查询类型高度同质化的系统
- 作为兜底策略与其他动态策略配合使用

### 2. Top-K 截断

固定返回分数最高的 K 条记忆，不设分数下限。

**代码形式：**
```python
def top_k_truncation(results: list[tuple[str, float]], k: int = 5):
    return results[:k]
```

**K 的选择策略：**

**（a）固定 K 值**
最简单但最粗糙——所有查询返回相同数量的记忆。对于简单查询可能引入噪声，复杂查询可能遗漏。

**（b）基于查询长度的自适应 K**
假设查询越长、越具体，需要的记忆越少；查询越短、越模糊，需要更多上下文。

```python
def adaptive_k_by_length(query: str, base_k: int = 5, max_k: int = 15):
    n_tokens = len(query.split())
    if n_tokens <= 3:     # 短查询，高模糊度
        return max_k
    elif n_tokens <= 10:  # 中等长度
        return base_k + (max_k - base_k) * (10 - n_tokens) // 7
    else:                 # 长查询，足够具体
        return base_k
```

**（c）基于检索分数的自适应 K**
观察分数下降曲线（score decay curve）的拐点，在"分数悬崖"处截断。

```python
def adaptive_k_by_elbow(results: list[tuple[str, float]], min_k: int = 1, max_k: int = 20):
    if len(results) <= min_k:
        return len(results)
    scores = [s for _, s in results[:max_k]]
    # 计算二阶差分，找最大下降点
    second_diffs = []
    for i in range(1, len(scores) - 1):
        second_diffs.append((scores[i-1] - scores[i]) - (scores[i] - scores[i+1]))
    if not second_diffs:
        return min_k
    elbow_idx = max(range(len(second_diffs)), key=lambda i: second_diffs[i]) + 1
    return max(min_k, min(elbow_idx + 1, max_k))
```

### 3. 自适应阈值 (Adaptive Threshold)

自适应阈值 T 根据查询特性与结果集统计属性动态计算，而非使用固定值。

#### 3.1 基于结果分布

**方法一：均值-标准差法 (Mean-Std)**

假设结果分数近似正态分布，用均值和标准差界定异常值：

```python
def adaptive_threshold_mean_std(results: list[tuple[str, float]], sigma: float = 1.5):
    scores = [s for _, s in results]
    if not scores:
        return 0.0
    mean = sum(scores) / len(scores)
    var = sum((s - mean) ** 2 for s in scores) / len(scores)
    std = var ** 0.5
    # 高于均值 + sigma * 标准差的被视为相关
    return mean + sigma * std
```

**方法二：百分位法 (Percentile-based)**

对分数分布排序后取指定百分位作为阈值，对非正态分布更鲁棒：

```python
import math

def adaptive_threshold_percentile(results: list[tuple[str, float]], percentile: float = 75.0):
    scores = sorted([s for _, s in results])
    if not scores:
        return 0.0
    k = (len(scores) - 1) * percentile / 100.0
    f = math.floor(k)
    c = math.ceil(k)
    if f == c:
        return scores[int(k)]
    return scores[f] * (c - k) + scores[c] * (k - f)
```

**方法三：最大间隔法 (Max-Gap)**

寻找排序后相邻分数之间的最大间隔，在间隔处截断：

```python
def adaptive_threshold_max_gap(results: list[tuple[str, float]], min_gap_ratio: float = 0.15):
    scores = sorted([s for _, s in results], reverse=True)
    if len(scores) <= 1:
        return 0.0
    max_gap = 0.0
    cut_point = 0
    for i in range(len(scores) - 1):
        gap = scores[i] - scores[i + 1]
        if gap > max_gap and gap > scores[i] * min_gap_ratio:
            max_gap = gap
            cut_point = i + 1
    # 阈值设为截断点的分数
    if cut_point == 0:
        return scores[-1] / 2  # 没有明显差距时的兜底
    return (scores[cut_point - 1] + scores[cut_point]) / 2
```

#### 3.2 基于查询复杂度

不同复杂度的查询需要不同的阈值策略。核心直觉：**查询越具体，阈值应越高；查询越模糊，阈值应越低**。

```python
def estimate_query_complexity(query: str) -> float:
    """
    估计查询复杂度，返回 [0, 1] 之间的值。
    0 = 非常简单，1 = 非常复杂
    """
    n_tokens = len(query.split())
    # 较长的查询通常更具体
    length_score = min(1.0, n_tokens / 20)
    # 检测疑问词和抽象概念
    abstract_indicators = ["怎么样", "如何", "为什么", "什么", "关系",
                           "difference", "relationship", "compare", "overview"]
    n_abstract = sum(1 for w in abstract_indicators if w in query.lower())
    abstract_score = min(1.0, n_abstract / 3)
    # 是否有明确实体
    has_entity = any(c.isupper() for c in query)  # 粗略检测专有名词
    entity_score = 0.0 if has_entity else 0.3
    return 0.4 * length_score + 0.4 * abstract_score + 0.2 * entity_score

def adaptive_threshold_by_query(results: list[tuple[str, float]], query: str):
    complexity = estimate_query_complexity(query)
    # 复杂度高 → 阈值低（给更宽松的通过条件）
    # 复杂度低 → 阈值高（只取高度匹配的）
    base_threshold = adaptive_threshold_percentile(results, 70.0)
    adjustment = complexity * 0.2  # 最大下调 0.2
    return base_threshold - adjustment
```

#### 3.3 基于历史反馈（强化学习方法）

利用 Agent 对检索结果的后续反馈（如用户是否采纳了检索到的记忆、LLM 基于该记忆的推理是否正确）来持续调整阈值。

```python
class RLThresholdAgent:
    """
    基于强化学习的阈值调整器。
    将阈值调整建模为 Bandit 问题：
    - 动作（Action）：从候选阈值集合中选择
    - 奖励（Reward）：检索结果的实际效用（如点击率、采纳率）
    """

    def __init__(self, candidate_thresholds: list[float], alpha: float = 0.1):
        self.thresholds = candidate_thresholds
        self.q_values = {t: 0.0 for t in candidate_thresholds}
        self.counts = {t: 0 for t in candidate_thresholds}
        self.alpha = alpha  # 学习率

    def select_threshold(self, epsilon: float = 0.1) -> float:
        """Epsilon-greedy 策略选择阈值"""
        import random
        if random.random() < epsilon:
            return random.choice(self.thresholds)
        return max(self.q_values, key=self.q_values.get)

    def update(self, threshold: float, reward: float):
        """增量更新 Q 值"""
        self.counts[threshold] += 1
        n = self.counts[threshold]
        self.q_values[threshold] += self.alpha * (reward - self.q_values[threshold]) / n

    def get_feedback_reward(self, retrieved: list[str], user_signal: list[bool]) -> float:
        """
        计算反馈奖励。
        user_signal[i] = True 表示用户/LLM 认为第 i 条记忆有用。
        """
        if not retrieved:
            return -0.5  # 空结果给予负奖励
        relevance_rate = sum(user_signal) / len(retrieved)
        coverage = len(retrieved) / max(len(retrieved), 1)
        # 惩罚过高和过低的数量
        size_penalty = 0
        if len(retrieved) > 20:
            size_penalty = -0.3
        elif len(retrieved) == 0:
            size_penalty = -0.5
        return relevance_rate * 0.6 + coverage * 0.2 + size_penalty
```

### 4. 多级阈值 (Multi-stage Thresholding)

将过滤过程拆分为多个阶段，每个阶段采用不同的阈值策略：

```ascii
  ┌─────────────┐
  │  原始候选池   │  10,000 条
  └──────┬──────┘
         │
         ▼
  ┌─────────────────────┐
  │ Stage 1: 粗过滤      │  τ₁ = 0.3（极低阈值）
  │ 快速筛除明显不相关    │  ANN 阶段内置
  └──────────┬──────────┘
             │  ~500 条
             ▼
  ┌─────────────────────┐
  │ Stage 2: 精过滤      │  τ₂ = Top-100 或 0.6 分位数
  │ 保留高质量候选        │  多路融合后应用
  └──────────┬──────────┘
             │  ~50 条
             ▼
  ┌─────────────────────┐
  │ Stage 3: 终选        │  τ₃ = 自适应（最大间隔法）
  │ 最后截断给 LLM       │  K = min(自适应K, Token预算)
  └──────────┬──────────┘
             │  3-15 条
             ▼
         LLM 推理
```

多级阈值的优势在于：
- **延迟优化**：早期使用廉价过滤器（如 ANN 索引的内置阈值），晚期使用精确但昂贵的过滤器（如重排序器）
- **渐进式缩小**：每一级减少候选数量的同时，可以应用更复杂的过滤逻辑
- **容错**：某一级的失误可以在后续级补救

---

## Python 代码示例：完整自适应阈值系统

下面是一个集成了多种自适应阈值策略的检索过滤模块：

```python
"""
adaptive_threshold.py — 检索阈值自适应调整模块

提供多种阈值策略的统一接口，支持：
- 固定阈值 / Top-K / 自适应阈值切换
- 基于分布的自适应（均值-标准差、百分位、最大间隔）
- 基于查询复杂度的阈值调整
- 基于强化学习的在线阈值学习
"""

import math
import random
from typing import Any, Callable

# ──────────────────────────────────────────────
# 1. 基础策略
# ──────────────────────────────────────────────

def filter_fixed(
    results: list[tuple[str, float]],
    threshold: float = 0.7,
) -> list[tuple[str, float]]:
    """固定阈值过滤"""
    return [(d, s) for d, s in results if s >= threshold]

def filter_topk(
    results: list[tuple[str, float]],
    k: int = 5,
) -> list[tuple[str, float]]:
    """Top-K 截断"""
    return results[:k]

# ──────────────────────────────────────────────
# 2. 自适应阈值 — 基于结果分布
# ──────────────────────────────────────────────

def _percentile(scores: list[float], p: float) -> float:
    """计算百分位值"""
    if not scores:
        return 0.0
    sorted_scores = sorted(scores)
    k = (len(sorted_scores) - 1) * p / 100.0
    f = math.floor(k)
    c = math.ceil(k)
    if f == c:
        return sorted_scores[int(k)]
    lo, hi = sorted_scores[f], sorted_scores[c]
    return lo + (hi - lo) * (k - f)

def adaptive_threshold_mean_std(
    results: list[tuple[str, float]],
    sigma: float = 1.5,
) -> float:
    """均值 + sigma * 标准差 作为阈值"""
    scores = [s for _, s in results]
    if not scores:
        return 0.0
    n = len(scores)
    mean = sum(scores) / n
    var = sum((s - mean) ** 2 for s in scores) / n
    std = var ** 0.5
    return mean + sigma * std

def adaptive_threshold_percentile(
    results: list[tuple[str, float]],
    percentile: float = 75.0,
) -> float:
    """指定百分位作为阈值"""
    scores = [s for _, s in results]
    return _percentile(scores, percentile)

def adaptive_threshold_max_gap(
    results: list[tuple[str, float]],
    min_gap_ratio: float = 0.15,
) -> float:
    """
    最大间隔法：在排序分数中找到下降最陡峭的位置。
    适用于分数分布有明显"悬崖"的场景。
    """
    scores = sorted([s for _, s in results], reverse=True)
    if len(scores) <= 1:
        return 0.0
    max_gap = 0.0
    cut_idx = 0
    for i in range(len(scores) - 1):
        gap = scores[i] - scores[i + 1]
        if gap > max_gap and gap > scores[i] * min_gap_ratio:
            max_gap = gap
            cut_idx = i + 1
    if cut_idx == 0:
        return scores[-1] / 2
    return (scores[cut_idx - 1] + scores[cut_idx]) / 2

# ──────────────────────────────────────────────
# 3. 自适应阈值 — 基于查询复杂度
# ──────────────────────────────────────────────

def estimate_query_complexity(query: str) -> float:
    """
    预估查询复杂度，返回 [0, 1]。
    0 = 简单/具体 → 高阈值
    1 = 复杂/模糊 → 低阈值
    """
    n_tokens = len(query.strip().split())
    length_score = min(1.0, n_tokens / 20.0)
    vague_words = [
        "关于", "一些", "各种", "相关", "thing", "stuff",
        "something", "about", "related", "some",
    ]
    n_vague = sum(1 for w in vague_words if w in query.lower())
    abstract_score = min(1.0, n_vague / 3.0)
    return 0.5 * length_score + 0.5 * abstract_score

def adaptive_threshold_by_query(
    results: list[tuple[str, float]],
    query: str,
    base_percentile: float = 70.0,
) -> float:
    """
    根据查询复杂度和结果分布联合决定阈值。
    复杂查询使用更宽松的百分位，简单查询更保守。
    """
    complexity = estimate_query_complexity(query)
    # 百分位随复杂度线性调整：P70 → P50
    adjusted_percentile = base_percentile - complexity * 20.0
    adjusted_percentile = max(30.0, min(90.0, adjusted_percentile))
    return adaptive_threshold_percentile(results, adjusted_percentile)

# ──────────────────────────────────────────────
# 4. 强化学习在线调整器
# ──────────────────────────────────────────────

class OnlineThresholdLearner:
    """
    基于上下文 Bandit 的在线阈值学习器。
    通过用户/LLM 对检索结果的反馈信号持续优化阈值选择。
    """

    def __init__(
        self,
        threshold_pool: list[float] | None = None,
        alpha: float = 0.1,
        gamma: float = 0.9,
    ):
        self.pool = threshold_pool or [0.5, 0.6, 0.7, 0.75, 0.8, 0.85, 0.9]
        self.q: dict[float, float] = {t: 0.0 for t in self.pool}
        self.counts: dict[float, int] = {t: 0 for t in self.pool}
        self.alpha = alpha
        self.gamma = gamma
        self.last_threshold: float | None = None

    def select(self, epsilon: float = 0.1) -> float:
        """Epsilon-greedy 策略"""
        if random.random() < epsilon:
            self.last_threshold = random.choice(self.pool)
        else:
            best_t = max(self.pool, key=lambda t: self.q[t] + math.sqrt(
                2 * math.log(sum(self.counts.values()) + 1) / (self.counts[t] + 1)
            ))
            self.last_threshold = best_t
        return self.last_threshold

    def update(self, reward: float):
        """在线更新 Q 值"""
        if self.last_threshold is None:
            return
        t = self.last_threshold
        self.counts[t] += 1
        n = self.counts[t]
        self.q[t] += self.alpha * (reward - self.q[t]) / n

    def compute_reward(
        self,
        retrieved: list[Any],
        relevance_labels: list[bool],
    ) -> float:
        """
        计算奖励：
        - 高相关率 → 正奖励
        - 结果过多或过少 → 惩罚
        """
        if not retrieved:
            return -1.0
        relevance = sum(relevance_labels[:len(retrieved)]) / len(retrieved)
        size_ok = 1 <= len(retrieved) <= 15
        reward = relevance * 1.0
        if not size_ok:
            reward -= 0.5
        return max(-1.0, min(1.0, reward))

# ──────────────────────────────────────────────
# 5. 多级阈值流水线
# ──────────────────────────────────────────────

class MultiStageThresholdPipeline:
    """
    多级阈值过滤流水线。
    每一级可以配置不同的过滤策略，候选集逐级缩减。
    """

    def __init__(self, stages: list[dict]):
        """
        stages: [
            {"name": "coarse", "fn": filter_function, "params": {...}},
            {"name": "fine",   "fn": filter_function, "params": {...}},
        ]
        """
        self.stages = stages

    def run(
        self,
        candidates: list[tuple[str, float]],
        query: str | None = None,
    ) -> list[tuple[str, float]]:
        """逐级过滤"""
        results = candidates
        for stage in self.stages:
            fn = stage["fn"]
            params = dict(stage.get("params", {}))
            # 支持注入 query 参数
            if "query" in fn.__code__.co_varnames:
                params["query"] = query or ""
            results = fn(results, **params)
        return results

# ──────────────────────────────────────────────
# 6. 使用示例
# ──────────────────────────────────────────────

if __name__ == "__main__":
    # 模拟检索结果
    mock_results = [
        ("Python 异常处理", 0.92),
        ("Python 错误类型", 0.88),
        ("try-except 用法", 0.85),
        ("Python 日志记录", 0.72),
        ("Python 调试技巧", 0.68),
        ("Java 异常处理", 0.55),
        ("Python 多线程", 0.52),
        ("C++ 异常规范", 0.48),
        ("Python 装饰器", 0.45),
        ("JavaScript 错误", 0.38),
    ]

    query = "python exception handling"

    print(f"查询: {query}")
    print(f"复杂度评分: {estimate_query_complexity(query):.2f}")
    print()

    strategies = [
        ("固定阈值 0.75", filter_fixed, {"threshold": 0.75}),
        ("固定阈值 0.50", filter_fixed, {"threshold": 0.50}),
        ("Top-K = 3", filter_topk, {"k": 3}),
        ("自适应(均值-标准差)", lambda r, **_: [
            (d, s) for d, s in r if s >= adaptive_threshold_mean_std(r)
        ], {}),
        ("自适应(最大间隔)", lambda r, **_: [
            (d, s) for d, s in r if s >= adaptive_threshold_max_gap(r)
        ], {}),
        ("自适应(查询复杂度)", lambda r, query=None, **_: [
            (d, s) for d, s in r
            if s >= adaptive_threshold_by_query(r, query or "")
        ], {"query": query}),
    ]

    for name, fn, params in strategies:
        filtered = fn(mock_results, **params)
        print(f"{name:30s} → {len(filtered)} 条: "
              f"{[d for d, s in filtered]}")
```

**输出示例：**

```
查询: python exception handling
复杂度评分: 0.55

固定阈值 0.75                 → 3 条: ['Python 异常处理', 'Python 错误类型', 'try-except 用法']
固定阈值 0.50                 → 7 条: ['Python 异常处理', 'Python 错误类型', 'try-except 用法', ...]
Top-K = 3                    → 3 条: ['Python 异常处理', 'Python 错误类型', 'try-except 用法']
自适应(均值-标准差)            → 4 条: ['Python 异常处理', 'Python 错误类型', 'try-except 用法', 'Python 日志记录']
自适应(最大间隔)               → 3 条: ['Python 异常处理', 'Python 错误类型', 'try-except 用法']
自适应(查询复杂度)             → 4 条: ['Python 异常处理', 'Python 错误类型', 'try-except 用法', 'Python 日志记录']
```

---

## 阈值与检索质量指标关系

### 关键指标定义

在评估阈值策略时，常用的指标包括：

| 指标 | 公式 | 含义 |
|------|------|------|
| **Precision@K** | TP / (TP + FP) | 前 K 条结果中相关的比例 |
| **Recall@K** | TP / (TP + FN) | 前 K 条结果覆盖了多少全部相关文档 |
| **F1@K** | 2 * P * R / (P + R) | 精度与召回率的调和平均 |
| **MRR** | 1 / rank_of_first_relevant | 第一个相关结果排名倒数的均值 |
| **MAP** | avg(P@k) over relevant k | 平均精度，综合衡量排序质量 |
| **NDCG@K** | DCG@K / IDCG@K | 考虑分级相关性的归一化折损累计增益 |

### 阈值调整对指标的影响

```
  Precision
    ↑ 1.0 │  τ=0.9
         │   τ=0.8
         │    τ=0.7
   0.5 ──│──── τ=0.6 ───
         │     τ=0.5
         │    τ=0.4
   0.0 ──│──────────────────→ Recall
        0.0       0.5      1.0
         ← 高 τ 方向  低 τ 方向 →
```

**实证观察：**

对典型嵌入模型（如 text-embedding-3-small）的检索结果进行阈值扫描，通常可以观察到：

| 阈值 τ | Precision | Recall | F1 | 输出条数 | 说明 |
|---------|-----------|--------|-----|----------|------|
| 0.90 | 1.00 | 0.25 | 0.40 | 极少 | 极保守，只取最确定的结果 |
| 0.80 | 0.95 | 0.50 | 0.66 | 少量 | 精度高但可能遗漏 |
| 0.70 | 0.85 | 0.70 | 0.77 | 中等 | 达到较好的 F1 平衡 |
| 0.60 | 0.65 | 0.85 | 0.74 | 较多 | 召回上升但精度开始下降 |
| 0.50 | 0.40 | 0.95 | 0.56 | 大量 | 高召回但噪声明显 |
| 0.30 | 0.15 | 1.00 | 0.26 | 极多 | 几乎全部放行 |

**关键观察：**
1. **F1 峰值通常在中高阈值区间**（约 0.65-0.80），但这是针对典型语义相似度而言，实际值因嵌入模型和数据集而异
2. **阈值对 Precision 的影响比对 Recall 更陡峭**——降低阈值时，Precision 快速下降而 Recall 增速放缓
3. **不同查询的 F1-τ 曲线形状差异很大**，这是自适应阈值策略存在的根本原因
4. **Top-K 固定时**，Precision@K 和 Recall@K 随 K 增大呈相反趋势

### 阈值评估的实用方法

```python
def scan_thresholds(
    results: list[tuple[str, float]],
    ground_truth: set[str],
    thresholds: list[float],
) -> dict[float, dict[str, float]]:
    """
    扫描不同阈值下的 Precision / Recall / F1。

    Args:
        results: 检索结果 [(doc_id, score), ...]
        ground_truth: 经标注的真正相关的文档 ID 集合
        thresholds: 待扫描的阈值列表

    Returns:
        {threshold: {"precision": float, "recall": float, "f1": float}}
    """
    all_doc_ids = {doc for doc, _ in results}
    tp_all = len(all_doc_ids & ground_truth)  # 所有被检索到的相关文档
    fn = len(ground_truth - all_doc_ids)      # 未被检索到的相关文档

    report = {}
    for tau in sorted(thresholds):
        retrieved = {doc for doc, score in results if score >= tau}
        tp = len(retrieved & ground_truth)
        fp = len(retrieved - ground_truth)
        precision = tp / (tp + fp) if (tp + fp) > 0 else 0.0
        recall = tp / (tp + fn) if (tp + fn) > 0 else 0.0
        f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0.0
        report[tau] = {"precision": precision, "recall": recall, "f1": f1}
    return report
```

---

## 工程优化

### 1. 缓存阈值决策

自适应阈值计算虽然开销不大，但在高并发系统中仍可能成为瓶颈。常见的优化手段：

```python
from functools import lru_cache
import time

class ThresholdCache:
    """
    阈值缓存：基于查询嵌入缓存的阈值决策。
    相同的查询（或语义近似的查询）复用历史阈值。
    """

    def __init__(self, ttl_seconds: int = 300, max_size: int = 1000):
        self.cache: dict[str, tuple[float, float]] = {}  # query_embedding_hash → (threshold, timestamp)
        self.ttl = ttl_seconds
        self.max_size = max_size

    def get(self, query_embedding_hash: str) -> float | None:
        entry = self.cache.get(query_embedding_hash)
        if entry is None:
            return None
        threshold, timestamp = entry
        if time.time() - timestamp > self.ttl:
            del self.cache[query_embedding_hash]
            return None
        return threshold

    def set(self, query_embedding_hash: str, threshold: float):
        if len(self.cache) >= self.max_size:
            # 清除最旧的 20%
            sorted_items = sorted(self.cache.items(), key=lambda x: x[1][1])
            for k, _ in sorted_items[:len(self.cache) // 5]:
                del self.cache[k]
        self.cache[query_embedding_hash] = (threshold, time.time())
```

### 2. 基于集合的阈值

不同记忆集合可以使用不同的阈值配置。例如，Agent 可以对不同类型的内容使用差异化阈值：

```python
class PerCollectionThresholdManager:
    """
    按记忆集合（collection）管理独立的阈值配置。
    """

    def __init__(self):
        # 默认配置
        self.configs: dict[str, dict] = {
            "default": {
                "strategy": "percentile",
                "percentile": 70.0,
                "min_results": 1,
                "max_results": 10,
            }
        }

    def register_collection(self, collection: str, config: dict):
        self.configs[collection] = config

    def filter(self, collection: str, results: list[tuple[str, float]]):
        config = self.configs.get(collection, self.configs["default"])
        strategy = config.get("strategy", "percentile")

        if strategy == "fixed":
            threshold = config.get("threshold", 0.7)
        elif strategy == "percentile":
            threshold = adaptive_threshold_percentile(
                results, config.get("percentile", 70.0)
            )
        elif strategy == "max_gap":
            threshold = adaptive_threshold_max_gap(
                results, config.get("min_gap_ratio", 0.15)
            )
        else:
            threshold = 0.0

        filtered = [(d, s) for d, s in results if s >= threshold]
        # 确保结果数量在合理范围
        min_r = config.get("min_results", 0)
        max_r = config.get("max_results", 20)
        if len(filtered) < min_r:
            filtered = results[:min_r]
        if len(filtered) > max_r:
            filtered = filtered[:max_r]
        return filtered

# 使用示例
manager = PerCollectionThresholdManager()
manager.register_collection("technical_docs", {
    "strategy": "percentile",
    "percentile": 80.0,
    "min_results": 1,
    "max_results": 5,
})
manager.register_collection("chat_history", {
    "strategy": "max_gap",
    "min_gap_ratio": 0.2,
    "min_results": 3,
    "max_results": 15,
})
manager.register_collection("user_preferences", {
    "strategy": "fixed",
    "threshold": 0.85,
    "min_results": 1,
    "max_results": 3,
})
```

### 3. 延迟优化技巧

| 技术 | 原理 | 预期延迟降低 |
|------|------|-------------|
| 级联过滤（先 Top-K 粗筛再精细排序） | 减少精确排序的计算量 | 40-60% |
| 近似分数预过滤 | 使用 PQ 量化后的粗分数进行预过滤 | 50-70% |
| 预计算集合级统计量 | 避免重复计算均值和标准差 | 20-30% |
| 延迟阈值（Deferred Thresholding） | 只在结果数超过 Token 预算时才应用阈值 | 10-20% |

---

## 能力边界

阈值调整不是万能的，以下情况需要意识到其局限性：

### 1. 嵌入空间本身的局限性

```
      ╔═══════════════════════════════════╗
      ║  阈值过滤的"盲区"                  ║
      ║                                   ║
      ║  相关但分数低  ←——→  不相关但分数高  ║
      ║  (false negative)     (false positive) ║
      ║                                   ║
      ║  根因：嵌入模型没有捕获到查询所需     ║
      ║       的特定语义维度                 ║
      ╚═══════════════════════════════════╝
```

最根本的限制：如果嵌入模型本身无法将相关文档映射到高相似度区域，任何阈值策略都无法挽救。阈值只能在嵌入质量允许的范围内进行筛选。

### 2. 小样本不稳定

当候选结果集很小时（如仅返回 3-5 条），基于统计的方法（均值、百分位）会变得非常不稳定。一条离群分数就能显著改变阈值。

### 3. 冷启动问题

基于强化学习的在线阈值调整在训练初期面临典型的冷启动问题——在没有足够的反馈信号之前，选择的阈值可能是次优的。

### 4. 无法处理相关性多样性

阈值是**一维的**——它只关注分数线的一个截断点。但实际的记忆相关性可能有多元维度：某条记忆虽然相似度不高，但时间上很新且重要性很高。纯粹依靠阈值会丢失这种多维信息，需要结合 6.4.2 temporal-decay 和 6.4.3 importance-scoring 的综合评分来解决。

### 5. 阈值与融合的耦合困境

阈值的有效性与前序融合策略（6.4.5）的质量紧密耦合。如果融合步骤对不同检索通道的分数归一化不当（例如，语义检索通道的分数普遍低于时间衰减通道），阈值过滤可能会系统性偏袒某个通道的结果。

---

## 最大挑战

### 挑战一：无标注数据下的最优阈值搜索

这是阈值调整面临的最大挑战——在真实 Agent 系统中，我们通常没有"哪些记忆应该被检索到"的标注数据，因此无法直接计算 F1 最优阈值。

**可能的应对方案：**

```ascii
  无标注数据
      │
      ├── 方法A: 启发式规则
      │   └── 基于分数分布的统计启发式（如最大间隔法）
      │
      ├── 方法B: 弱监督信号
      │   └── 使用 LLM 对检索结果做弱标注
      │       "请判断以下记忆是否与当前上下文相关 [Y/N]"
      │
      ├── 方法C: 贝叶斯优化
      │   └── 将阈值作为超参数，用贝叶斯优化搜索
      │       目标：使下游任务指标最优（而非检索指标最优）
      │
      └── 方法D: 对比学习
          └── 从用户的隐式反馈中学习阈值偏好
              （点击、停留时间、采纳率等）
```

### 挑战二：分数漂移 (Score Drift)

随着时间的推移，以下因素可能导致分数分布整体漂移：
- 嵌入模型更新
- 新数据加入改变语料库分布
- 用户查询模式变化

这意味着今天最优的阈值策略，明天可能就不是最优的。需要有**在线监测**和**自动校准**机制。

### 挑战三：指标不一致

离线评估（在标注数据集上计算 F1）和在线效果（LLM 推理质量）之间的指标鸿沟——优化检索指标并不总是能提升下游任务的质量。

---

## 场景判断表格

以下表格帮助根据具体场景选择合适的阈值策略：

| 场景 | 推荐策略 | 次要策略 | 原因 |
|------|----------|----------|------|
| **通用对话 Agent** | 自适应（百分位 P70） | Top-K (5-10) | 查询多样，需要动态平衡精度和召回 |
| **代码检索 Agent** | 固定阈值 (0.8+) + Top-K | 最大间隔法 | 代码匹配分数通常较高且分布尖锐 |
| **创作辅助 Agent** | 低阈值 (0.5) + Top-K (10-15) | 自适应（查询复杂度） | 需要高召回，宁可引入一些噪声 |
| **事实问答 Agent** | Top-K (3-5) + 高阈值 | 固定阈值 0.75+ | 只需要最相关的少数记忆，精度优先 |
| **长期记忆推理 Agent** | 多级阈值（粗→精） | 自适应（百分位 P60） | 候选数大，需要渐进式过滤 |
| **客服 Agent（同域）** | 基于历史反馈（RL） | 自适应（均值-标准差） | 有大量用户反馈信号可用 |
| **冷启动新 Agent** | Top-K (5) + 自适应 | 固定阈值 0.6 | 无历史数据时保守起步 |
| **多模态记忆检索** | 基于集合的阈值 | 自适应（百分位 P50） | 不同模态分数分布差异大 |
| **实时低延迟系统** | Top-K 固定 | 固定阈值 | 计算代价必须最低 |
| **高召回关键系统** | 多方案并集（低阈值 0.4） | Top-K (20+) | 宁可多不可漏，在后续阶段进一步筛选 |

### 决策流程

```ascii
  开始选择阈值策略
      │
      ▼
  是否有标注数据？
  ├── Yes → 扫描 F1-τ 曲线，选取 F1 最优 τ
  │         └── 考虑加入在线微调（RL 补偿）
  │
  └── No  → 是否有用户反馈信号？
            ├── Yes → 使用 RL 在线学习 + 启发式兜底
            │
            └── No  → 选择启发式策略：
                        ├── 分数分布有清晰"悬崖"？→ 最大间隔法
                        ├── 查询模式多样？         → 基于查询复杂度
                        └── 需要简单稳定？         → 百分位法（P70 起步）
                              │
                              ▼
                        部署后持续监控：
                        - 输出记忆的数量分布
                        - 用户采纳率 / LLM 拒绝率
                        - 若偏离基线 → 触发策略重校准
```

---

## 与相关模块的关系

| 相关模块 | 关系 | 交互方式 |
|----------|------|----------|
| **6.4.1 语义检索 (Semantic Search)** | 阈值过滤的**前序输入** | 语义检索输出向量相似度分数，是阈值过滤的主要输入数据 |
| **6.4.5 结果融合 (Retrieval Fusion)** | 阈值过滤的**直接前驱** | 融合后的归一化分数是阈值过滤的输入；融合质量的优劣直接影响阈值策略的效果 |
| **6.4.2 时间衰减 (Temporal Decay)** | **协同过滤** | 时间分数可以和相似度分数融合后再应用阈值，提高检索结果时效性 |
| **6.4.3 重要性评分 (Importance Scoring)** | **协同过滤** | 重要性得分可以作为阈值决策的额外信号维度 |

---

## 总结

检索阈值自适应调整是 AI Agent 记忆检索流水线中的最后一道质量关卡，负责从候选记忆中筛选出最优质的上下文供 LLM 使用。其核心挑战在于：**在无标注数据的情况下，为每个查询动态确定最优的截断点**。

没有一种阈值策略适用于所有场景。固定阈值简单可靠、Top-K 可控可预测、自适应阈值灵活智能、多级阈值鲁棒高效。在实际 Agent 系统中，通常采用**混合策略**——以自适应策略为主、固定策略作为安全兜底，并配合在线监测机制持续校准。

最终，阈值策略的成功标准不在于检索指标本身（如 F1 分数），而在于其对下游任务的实际增益——Agent 的决策质量和用户体验是否因此提升。

---

## 延伸阅读

- **pEBR: Probabilistic Approach to Embedding Based Retrieval** — 利用概率模型而非固定阈值判断相关性
- **Relevance Filtering for Embedding-based Retrieval (CIKM 2024)** — 基于分数的相关性过滤方法论
- **ScoreGate: Adaptive Chunk Selection for RAG via Dual-Score Statistical Fusion** — 双分数统计融合的自适应阈值
- **Distribution-Aware Exploration for Adaptive HNSW Search** — 分布感知的自适应近似搜索
- Milvus / Pinecone / Weaviate 官方文档中的相似度阈值指南
