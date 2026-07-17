# 11.1.6 composite-score — 综合评分：加权多指标融合

## 简单介绍

综合评分将多个评估指标融合为一个统一的数值，解决"每个指标都说了一部分，但没有一个指标说全了"的问题。单一指标容易被针对性地优化（Goodhart's Law：当一个指标成为目标时，它就不再是一个好指标），而多个指标同时看又难以做出"哪个 Agent 更好"的明确判断。综合评分回答一个关键问题：**综合考虑所有维度，哪个 Agent 的综合表现最优？**

## 基本原理

### 综合评分的核心流程

```
原始指标矩阵              归一化                 加权融合              最终评分
┌──────────────┐      ┌──────────────┐      ┌──────────────┐      ┌──────────┐
│ Agent A      │      │ Agent A      │      │ Agent A      │      │  A: 0.85 │
│  成功率: 85% │      │  成功率: 0.85│      │  成功率 x0.4 │      │          │
│  步数: 5     │  →   │  步数: 0.75 │  →   │  步数 x0.2   │  →   │  B: 0.72 │
│  成本: 0.3  │      │  成本: 0.70 │      │  成本 x0.1   │      │          │
│  可靠性: 0.8│      │  可靠性: 0.80│      │  可靠性 x0.3 │      │  C: 0.78 │
└──────────────┘      └──────────────┘      └──────────────┘      └──────────┘
```

## 背景

### 为什么需要综合评分？

单个指标的局限性：

```
单个指标评估的盲区：

只看成功率：
  Agent X: 成功率 90% → 看起来很优秀
  Agent Y: 成功率 85% → 看起来稍差
  但 Agent X 每次需要 50 步 + 100K Tokens，成本是 Agent Y 的 10 倍

只看效率：
  Agent X: 平均 2 步完成 → 看起来高效
  Agent Y: 平均 8 步完成 → 看起来冗长
  但 Agent X 只处理简单任务，Agent Y 处理复杂任务

只看可靠性（一致性）：
  Agent X: 一致性 98% → 看起来很稳定
  Agent Y: 一致性 70% → 看起来不稳定
  但 Agent X 只会说"我不确定"，Agent Y 会尝试回答且大部分正确
```

核心洞察：**单一指标可以被游戏（Gamed），但多指标同时被游戏的难度呈指数增长。**

### 多准则决策（MCDM）理论基础

综合评分本质上是多准则决策（Multi-Criteria Decision Making, MCDM）问题。MCDM 的核心要素包括：

- **决策矩阵**：行 = 候选方案（Agents），列 = 评估指标
- **权重向量**：每个指标的重要性
- **聚合函数**：如何将多维度分数合并

## 核心矛盾

**简单加权 vs 维度间的复杂依赖关系。** 最常见的加权平均法假设各指标之间相互独立——但实际中指标之间高度耦合：成功率和效率通常负相关，可靠性和适应性常常此消彼长。更复杂的方法（如 TOPSIS、AHP）试图捕捉这些关系，但引入了主观性和计算复杂度。

```
加权平均法
 优点：简单、透明、可解释
 缺点：假设指标独立、权重主观、不能处理非线性关系
  
   vs

TOPSIS / AHP
 优点：考虑指标间关系、理论基础扎实
 缺点：计算复杂、权重获取难度大、可解释性差
```

**权重选择的主观性是所有综合评分方法都无法回避的问题。**

## 评估方法

### 1. 加权平均法（Weighted Sum Model, WSM）

最基础的综合评分方法，将各指标得分乘以权重后求和。

```python
class WeightedScorer:
    """加权平均法综合评分"""
    
    def __init__(self, weights: dict):
        """
        weights: {"metric_name": weight_value}
        所有权重之和应为 1.0
        """
        total = sum(weights.values())
        self.weights = {
            k: v / total for k, v in weights.items()
        }
        self.metric_config = {}
    
    def set_metric_config(self, metric: str, 
                           higher_is_better: bool = True,
                           min_val: float = 0.0,
                           max_val: float = 1.0) -> None:
        """配置每个指标的取值范围和方向"""
        self.metric_config[metric] = {
            "higher_is_better": higher_is_better,
            "min_val": min_val,
            "max_val": max_val
        }
    
    def normalize(self, raw_value: float, 
                   metric: str) -> float:
        """Min-Max 归一化"""
        cfg = self.metric_config.get(metric, {
            "higher_is_better": True,
            "min_val": 0.0,
            "max_val": 1.0
        })
        
        normalized = (raw_value - cfg["min_val"]) / \
                     max(cfg["max_val"] - cfg["min_val"], 1e-8)
        normalized = max(0.0, min(1.0, normalized))
        
        if not cfg["higher_is_better"]:
            normalized = 1.0 - normalized
        
        return normalized
    
    def score(self, metrics: dict) -> dict:
        """
        计算综合加权评分
        metrics: {"metric_name": raw_value}
        """
        if not self.weights:
            return {"composite_score": 0.0, "error": "weights not set"}
        
        weighted_sum = 0.0
        detail = {}
        
        for metric, weight in self.weights.items():
            if metric not in metrics:
                continue
            
            raw = metrics[metric]
            norm = self.normalize(raw, metric)
            contribution = norm * weight
            weighted_sum += contribution
            
            detail[metric] = {
                "raw": raw,
                "normalized": norm,
                "weight": weight,
                "contribution": contribution
            }
        
        return {
            "composite_score": weighted_sum,
            "detail": detail,
            "weights": self.weights
        }
    
    
    def compare_agents(self, 
                        agent_scores: dict) -> list:
        """
        比较多个 Agent 的综合评分
        agent_scores: {"agent_name": {"metric": value}}
        """
        results = []
        for name, metrics in agent_scores.items():
            result = self.score(metrics)
            results.append({
                "agent": name,
                "composite_score": result["composite_score"],
                "detail": result["detail"]
            })
        
        results.sort(key=lambda x: x["composite_score"], 
                     reverse=True)
        return results


# 使用示例
weights = {
    "success_rate": 0.35,
    "efficiency": 0.20,
    "cost": 0.10,
    "robustness": 0.15,
    "reliability": 0.20
}

scorer = WeightedScorer(weights)
scorer.set_metric_config("success_rate", higher_is_better=True, min_val=0, max_val=1.0)
scorer.set_metric_config("efficiency", higher_is_better=True, min_val=0, max_val=1.0)
scorer.set_metric_config("cost", higher_is_better=False, min_val=0, max_val=1.0)
scorer.set_metric_config("robustness", higher_is_better=True, min_val=0, max_val=1.0)
scorer.set_metric_config("reliability", higher_is_better=True, min_val=0, max_val=1.0)

agents = {
    "Agent-Alpha": {"success_rate": 0.88, "efficiency": 0.75, 
                     "cost": 0.30, "robustness": 0.80, "reliability": 0.85},
    "Agent-Beta": {"success_rate": 0.92, "efficiency": 0.60, 
                    "cost": 0.50, "robustness": 0.70, "reliability": 0.90},
    "Agent-Gamma": {"success_rate": 0.75, "efficiency": 0.90, 
                     "cost": 0.15, "robustness": 0.65, "reliability": 0.70}
}

ranking = scorer.compare_agents(agents)
# [
#   {"agent": "Agent-Alpha", "composite_score": 0.830},
#   {"agent": "Agent-Beta",  "composite_score": 0.790},
#   {"agent": "Agent-Gamma", "composite_score": 0.728}
# ]
```

### 2. TOPSIS 方法

TOPSIS（Technique for Order Preference by Similarity to Ideal Solution）基于"最优方案应距离正理想解最近、距离负理想解最远"的思想。

```python
import numpy as np

class TOPSISScorer:
    """TOPSIS 多指标综合评分"""
    
    def __init__(self, weights: list, 
                 benefit_criteria: list):
        """
        weights: 各指标的权重列表
        benefit_criteria: 哪些指标是越大越好（True=benefit, False=cost）
        """
        self.weights = np.array(weights)
        self.benefit_criteria = benefit_criteria
    
    def score(self, decision_matrix: np.ndarray) -> dict:
        """
        计算 TOPSIS 综合评分
        decision_matrix: shape = [n_agents, n_criteria]
        """
        n_agents, n_criteria = decision_matrix.shape
        
        # 1. 向量归一化
        norm_matrix = decision_matrix / np.sqrt(
            np.sum(decision_matrix ** 2, axis=0))
        
        # 2. 加权归一化
        weighted_matrix = norm_matrix * self.weights
        
        # 3. 确定正理想解和负理想解
        ideal_best = np.zeros(n_criteria)
        ideal_worst = np.zeros(n_criteria)
        
        for j in range(n_criteria):
            if self.benefit_criteria[j]:
                ideal_best[j] = np.max(weighted_matrix[:, j])
                ideal_worst[j] = np.min(weighted_matrix[:, j])
            else:
                ideal_best[j] = np.min(weighted_matrix[:, j])
                ideal_worst[j] = np.max(weighted_matrix[:, j])
        
        # 4. 计算到正/负理想解的距离
        dist_best = np.sqrt(
            np.sum((weighted_matrix - ideal_best) ** 2, axis=1))
        dist_worst = np.sqrt(
            np.sum((weighted_matrix - ideal_worst) ** 2, axis=1))
        
        # 5. 计算相对接近度
        # 避免除以零
        denom = dist_best + dist_worst
        denom = np.where(denom == 0, 1e-8, denom)
        closeness = dist_worst / denom
        
        return {
            "scores": closeness.tolist(),
            "ranking": np.argsort(-closeness).tolist(),
            "ideal_best": ideal_best.tolist(),
            "ideal_worst": ideal_worst.tolist(),
            "dist_to_best": dist_best.tolist(),
            "dist_to_worst": dist_worst.tolist()
        }
    
    def compare_agents(self, 
                        agent_names: list,
                        decision_matrix: np.ndarray) -> list:
        """
        返回排序后的 Agent 对比结果
        """
        result = self.score(decision_matrix)
        
        rankings = []
        for idx, rank_pos in enumerate(result["ranking"]):
            rankings.append({
                "rank": idx + 1,
                "agent": agent_names[rank_pos],
                "topsis_score": result["scores"][rank_pos],
                "dist_to_ideal": result["dist_to_best"][rank_pos],
                "dist_to_worst": result["dist_to_worst"][rank_pos]
            })
        
        return rankings


# 使用示例
# 指标: [success_rate, efficiency, robustness, reliability]
# 注意 efficiency 越低越好（步数少），所以是 cost 指标
agents = ["Agent-A", "Agent-B", "Agent-C"]
matrix = np.array([
    [0.90, 5, 0.80, 0.85],   # Agent-A
    [0.85, 3, 0.75, 0.92],   # Agent-B
    [0.95, 8, 0.90, 0.70]    # Agent-C
])

topsis = TOPSISScorer(
    weights=[0.30, 0.25, 0.25, 0.20],
    benefit_criteria=[True, False, True, True]
)

ranking = topsis.compare_agents(agents, matrix)
```

### 3. 雷达图 / 蛛网图评分

可视化辅助的评分方法，通过多边形的面积来综合评估。

```python
import math

class RadarScorer:
    """雷达图综合评分"""
    
    def __init__(self, n_dimensions: int):
        self.n_dims = n_dimensions
    
    def compute_polygon_area(self, 
                              normalized_scores: list) -> float:
        """
        计算归一化得分在雷达图上的多边形面积
        面积越大，综合表现越好
        
        normalized_scores: [0-1] 每个维度的得分
        """
        n = len(normalized_scores)
        angles = [2 * math.pi * i / n for i in range(n)]
        
        # 多边形顶点坐标
        x_coords = [s * math.cos(a) 
                    for s, a in zip(normalized_scores, angles)]
        y_coords = [s * math.sin(a) 
                    for s, a in zip(normalized_scores, angles)]
        
        # 鞋带公式求多边形面积
        area = 0.0
        for i in range(n):
            j = (i + 1) % n
            area += x_coords[i] * y_coords[j]
            area -= x_coords[j] * y_coords[i]
        
        area = abs(area) / 2.0
        
        # 归一化：除以最大可能面积（正 n 边形的面积，所有顶点距离为 1）
        max_area = n / (2 * math.tan(math.pi / n))
        normalized_area = area / max_area
        
        return normalized_area
    
    def score_with_balance(self, 
                            scores: dict) -> dict:
        """
        综合评分 + 平衡度评估
        scores: {"metric_name": normalized_score(0-1)}
        """
        values = list(scores.values())
        names = list(scores.keys())
        
        # 多边形面积评分
        area_score = self.compute_polygon_area(values)
        
        # 平衡度：各维度得分的均匀程度
        mean = float(np.mean(values))
        std = float(np.std(values))
        balance = 1.0 - min(std / max(mean, 0.01), 1.0)
        
        # 短板效应：最低维度的得分对整体有惩罚
        min_score = min(values)
        penalty = 1.0 - (1.0 - min_score) ** 2
        
        # 综合评分 = 面积评分 * 平衡度 * 短板惩罚
        composite = area_score * balance * penalty
        
        return {
            "metrics": scores,
            "area_score": area_score,
            "balance": balance,
            "min_dimension_penalty": penalty,
            "weakest_dimension": names[values.index(min_score)],
            "weakest_score": min_score,
            "composite_score": composite,
            "radar_interpretation": "balanced_strong" if composite > 0.6
                        else "imbalanced" if balance < 0.5
                        else "moderate"
        }


# 使用示例
radar = RadarScorer(n_dimensions=5)

agent_scores = {
    "success_rate": 0.85,
    "efficiency": 0.70,
    "robustness": 0.90,
    "reliability": 0.80,
    "adaptability": 0.65
}

result = radar.score_with_balance(agent_scores)
```

### 4. Pareto 前沿分析

不是生成单一分数，而是识别出"不被任何其他 Agent 在所有维度上支配"的 Pareto 最优 Agent 集合。

```python
class ParetoFrontier:
    """Pareto 前沿分析"""
    
    def __init__(self, objectives: list):
        """
        objectives: [{"name": str, "higher_is_better": bool}]
        """
        self.objectives = objectives
    
    def is_dominated(self, agent_a: np.ndarray, 
                      agent_b: np.ndarray) -> bool:
        """
        判断 Agent A 是否被 Agent B 支配
        支配定义：B 在所有维度上不差于 A，且至少有一个维度严格优于 A
        """
        n_dims = len(agent_a)
        better_in_any = False
        
        for i in range(n_dims):
            higher_better = self.objectives[i]["higher_is_better"]
            
            if higher_better:
                a_better = agent_a[i] > agent_b[i]
                b_better = agent_b[i] > agent_a[i]
            else:
                a_better = agent_a[i] < agent_b[i]
                b_better = agent_b[i] < agent_a[i]
            
            if b_better:
                better_in_any = True
            elif a_better:
                # A 至少在一个维度上优于 B，B 不支配 A
                return False
        
        return better_in_any
    
    def find_pareto_frontier(self, 
                              agent_scores: dict) -> dict:
        """
        找出 Pareto 前沿
        agent_scores: {"agent_name": [score1, score2, ...]}
        """
        names = list(agent_scores.keys())
        scores = np.array([agent_scores[n] for n in names])
        
        n_agents = len(names)
        is_pareto = [True] * n_agents
        
        for i in range(n_agents):
            for j in range(n_agents):
                if i == j:
                    continue
                if self.is_dominated(scores[i], scores[j]):
                    is_pareto[i] = False
                    break
        
        pareto_agents = [
            names[i] for i in range(n_agents) if is_pareto[i]
        ]
        dominated_agents = [
            names[i] for i in range(n_agents) if not is_pareto[i]
        ]
        
        return {
            "pareto_frontier": pareto_agents,
            "dominated_agents": dominated_agents,
            "n_pareto_optimal": len(pareto_agents),
            "n_dominated": len(dominated_agents)
        }
    
    def compute_dominance_depth(self, 
                                 agent_scores: dict) -> dict:
        """
        计算每个 Agent 的支配深度（被多少层 Agent 支配）
        用于对非 Pareto 前沿的 Agent 进行分层排序
        """
        names = list(agent_scores.keys())
        scores = np.array([agent_scores[n] for n in names])
        n_agents = len(names)
        
        domination_count = np.zeros(n_agents, dtype=int)
        
        for i in range(n_agents):
            for j in range(n_agents):
                if i == j:
                    continue
                if self.is_dominated(scores[i], scores[j]):
                    domination_count[i] += 1
        
        return {
            names[i]: {
                "domination_depth": int(domination_count[i]),
                "rank": int(np.unique(domination_count).tolist().index(
                    domination_count[i])) + 1
            }
            for i in range(n_agents)
        }
```

### 5. 层次分析法（AHP）

通过成对比较确定指标权重，减少权重主观性。

```python
class AHPScorer:
    """层次分析法（AHP）权重计算"""
    
    def __init__(self):
        # Saaty 标度
        self.scale = {
            "equal": 1,
            "moderate": 3,
            "strong": 5,
            "very_strong": 7,
            "extreme": 9
        }
        # 随机一致性指数 RI
        self.ri = {1: 0, 2: 0, 3: 0.58, 4: 0.90, 5: 1.12,
                   6: 1.24, 7: 1.32, 8: 1.41, 9: 1.45, 10: 1.49}
    
    def compute_weights(self, 
                         pairwise_matrix: list) -> dict:
        """
        从成对比较矩阵计算权重
        
        pairwise_matrix[i][j] = 指标 i 相对于指标 j 的重要性
        取值：1（同等重要）到 9（极端重要）
        其中 pairwise_matrix[j][i] = 1 / pairwise_matrix[i][j]
        """
        matrix = np.array(pairwise_matrix, dtype=float)
        n = matrix.shape[0]
        
        # 1. 计算几何平均
        geometric_means = np.exp(
            np.sum(np.log(matrix), axis=1) / n)
        
        # 2. 归一化得到权重
        weights = geometric_means / np.sum(geometric_means)
        
        # 3. 一致性检验
        weighted_sum = matrix @ weights
        lambda_max = np.mean(weighted_sum / weights)
        
        # 一致性指标 CI
        ci = (lambda_max - n) / (n - 1)
        
        # 一致性比率 CR
        ri = self.ri.get(n, 1.49)
        cr = ci / ri if ri > 0 else 0
        
        return {
            "weights": weights.tolist(),
            "consistency_ratio": cr,
            "is_consistent": cr < 0.1,
            "lambda_max": lambda_max,
            "ci": ci,
            "n_criteria": n
        }
    
    def get_priority_weights(self, 
                              comparisons: list) -> dict:
        """
        comparisons: [{"i": "metric_a", "j": "metric_b", 
                        "value": 3, "meaning": "moderate"}]
        返回各指标的 AHP 权重
        """
        # 构建成对比较矩阵
        metrics = list(set(
            [c["i"] for c in comparisons] + 
            [c["j"] for c in comparisons]
        ))
        n = len(metrics)
        idx_map = {m: i for i, m in enumerate(metrics)}
        
        matrix = np.ones((n, n))
        for c in comparisons:
            i, j = idx_map[c["i"]], idx_map[c["j"]]
            matrix[i][j] = c["value"]
            matrix[j][i] = 1.0 / c["value"]
        
        result = self.compute_weights(matrix.tolist())
        
        return {
            "metrics": metrics,
            "weights": {
                metrics[i]: result["weights"][i]
                for i in range(n)
            },
            "consistency_ratio": result["consistency_ratio"],
            "is_consistent": result["is_consistent"],
            "usage_note": (
                "CR < 0.1 表示判断一致性可接受"
                if result["is_consistent"]
                else "CR >= 0.1，建议重新审视成对比较"
            )
        }


# 使用示例
ahp = AHPScorer()

# 成对比较：成功率 vs 效率 = 3（成功率稍微重要）
#          成功率 vs 鲁棒性 = 2（成功率略重要）
#          效率 vs 鲁棒性 = 1/2（鲁棒性比效率重要）
comparisons = [
    {"i": "success_rate", "j": "efficiency", "value": 3},
    {"i": "success_rate", "j": "robustness", "value": 2},
    {"i": "efficiency", "j": "robustness", "value": 1/2},
]

weights = ahp.get_priority_weights(comparisons)
```

### 6. 星级综合评分

灵感来自电商评分，将连续分数映射到离散星级。

```python
class StarRatingScorer:
    """星级综合评分（1-5 星）"""
    
    def __init__(self, criteria: list):
        """
        criteria: [{"name": str, "weight": float, "thresholds": list}]
        thresholds: 每个星级对应的阈值，如 [0.0, 0.4, 0.6, 0.8, 0.9]
        """
        self.criteria = criteria
    
    def compute_star_rating(self, 
                             scores: dict) -> dict:
        """
        计算星级综合评分
        scores: {"criterion_name": normalized_score(0-1)}
        """
        total_weighted_stars = 0.0
        total_weight = 0.0
        detail = []
        
        for criterion in self.criteria:
            name = criterion["name"]
            weight = criterion["weight"]
            thresholds = criterion["thresholds"]
            
            raw_score = scores.get(name, 0)
            stars = self._score_to_stars(raw_score, thresholds)
            
            total_weighted_stars += stars * weight
            total_weight += weight
            
            detail.append({
                "criterion": name,
                "score": raw_score,
                "stars": stars,
                "weight": weight
            })
        
        overall_stars = total_weighted_stars / max(total_weight, 1e-8)
        
        return {
            "overall_stars": round(overall_stars, 1),
            "overall_stars_display": "★" * int(overall_stars) + 
                                     "☆" * (5 - int(overall_stars)),
            "detail": detail,
            "criteria_count": len(self.criteria)
        }
    
    def _score_to_stars(self, score: float, 
                         thresholds: list) -> float:
        """将连续分数映射到 1-5 星"""
        for stars, threshold in enumerate(thresholds, 1):
            if score < threshold:
                return stars - 0.5 if stars > 1 else 1.0
        return 5.0
```

## 指标归一化方法

不同指标的量纲和取值范围不同，归一化是综合评分的前提。

```python
class MetricNormalizer:
    """多种指标归一化方法"""
    
    @staticmethod
    def min_max_normalize(values: list, 
                           new_min: float = 0.0,
                           new_max: float = 1.0) -> list:
        """Min-Max 归一化：保留原始分布形状"""
        arr = np.array(values, dtype=float)
        old_min = np.min(arr)
        old_max = np.max(arr)
        
        if old_max == old_min:
            return [new_min] * len(values)
        
        normalized = new_min + (arr - old_min) / (old_max - old_min) * \
                     (new_max - new_min)
        return normalized.tolist()
    
    @staticmethod
    def z_score_normalize(values: list) -> list:
        """Z-Score 归一化（标准化）：适合正态分布的数据"""
        arr = np.array(values, dtype=float)
        mean = np.mean(arr)
        std = np.std(arr)
        
        if std == 0:
            return [0.0] * len(values)
        
        return ((arr - mean) / std).tolist()
    
    @staticmethod
    def robust_normalize(values: list) -> list:
        """鲁棒归一化：使用中位数和 IQR，对异常值不敏感"""
        arr = np.array(values, dtype=float)
        median = np.median(arr)
        q1 = np.percentile(arr, 25)
        q3 = np.percentile(arr, 75)
        iqr = q3 - q1
        
        if iqr == 0:
            return [0.5] * len(values)
        
        normalized = (arr - median) / iqr
        return normalized.tolist()
    
    @staticmethod
    def rank_normalize(values: list) -> list:
        """排名归一化：只关注相对顺序，消除量纲影响"""
        arr = np.array(values)
        n = len(arr)
        ranks = np.argsort(np.argsort(arr))  # 0 = 最小, n-1 = 最大
        return (ranks / (n - 1)).tolist() if n > 1 else [1.0]
    
    @staticmethod
    def softmax_normalize(values: list, 
                           temperature: float = 1.0) -> list:
        """Softmax 归一化：放大优秀和糟糕之间的差距"""
        arr = np.array(values, dtype=float) / temperature
        exp_arr = np.exp(arr - np.max(arr))  # 数值稳定处理
        return (exp_arr / np.sum(exp_arr)).tolist()
```

## 可视化综合评分

```python
class CompositeScoreVisualizer:
    """综合评分可视化辅助"""
    
    @staticmethod
    def prepare_radar_data(agent_scores: dict) -> dict:
        """
        准备雷达图数据
        agent_scores: {"agent": {"metric": value}}
        """
        metrics = list(list(agent_scores.values())[0].keys())
        
        return {
            "metrics": metrics,
            "agents": [
                {
                    "name": name,
                    "values": [scores[m] for m in metrics]
                }
                for name, scores in agent_scores.items()
            ]
        }
    
    @staticmethod
    def generate_score_summary(composite_results: list) -> str:
        """
        生成可读的综合评分摘要
        
        composite_results: WeightedScorer.compare_agents() 的返回值
        """
        lines = ["=== Agent 综合评分排名 ==="]
        lines.append(f"{'排名':<4} {'Agent':<15} {'综合得分':<10} "
                     f"{'最佳维度':<12} {'最弱维度':<12}")
        lines.append("-" * 55)
        
        for entry in composite_results:
            rank = composite_results.index(entry) + 1
            name = entry["agent"]
            score = entry["composite_score"]
            detail = entry["detail"]
            
            best = max(detail.items(), 
                       key=lambda x: x[1]["contribution"])
            worst = min(detail.items(), 
                        key=lambda x: x[1]["contribution"])
            
            lines.append(
                f"{rank:<4} {name:<15} {score:<10.3f} "
                f"{best[0]:<12} {worst[0]:<12}"
            )
        
        return "\n".join(lines)
```

## 实现挑战

### 1. 权重选择的主观性

```
不同角色对权重的偏好差异：

                   开发者        产品经理        客户
成功率             0.30          0.40          0.50
效率（步数）        0.25          0.15          0.05
成本               0.10          0.20          0.30
鲁棒性             0.20          0.10          0.05
可靠性             0.15          0.15          0.10

相同数据，不同权重得到不同排名：
开发者：Agent-A > Agent-B > Agent-C
产品经理：Agent-B > Agent-A > Agent-C
客户：Agent-B > Agent-C > Agent-A

→ 综合评分不是客观真理，而是主观偏好的数学化表达
```

### 2. 极端值敏感度

```python
def sensitivity_analysis(scorer: WeightedScorer,
                          base_scores: dict,
                          metric_to_vary: str,
                          value_range: list) -> list:
    """
    敏感性分析：观察某个指标变化时综合评分的波动
    """
    results = []
    
    for value in value_range:
        modified = dict(base_scores)
        modified[metric_to_vary] = value
        
        result = scorer.score(modified)
        results.append({
            "metric_value": value,
            "composite_score": result["composite_score"],
            "delta": result["composite_score"] - results[0]["composite_score"]
            if results else 0
        })
    
    return results
```

### 3. 指标之间的相关性

```
指标相关矩阵（示例数据）：

              成功率    效率    鲁棒性   可靠性   适应性
成功率         1.00   -0.65    0.45    0.30    0.20
效率          -0.65    1.00   -0.30   -0.20   -0.10
鲁棒性         0.45   -0.30    1.00    0.55   -0.25
可靠性         0.30   -0.20    0.55    1.00   -0.40
适应性         0.20   -0.10   -0.25   -0.40    1.00

问题：
- 成功率和效率强负相关（-0.65），加权平均重复计算了相同信息
- 鲁棒性和可靠性中度正相关（0.55），部分维度信息冗余
- 解决方案：使用 PCA 去相关，或只保留正交维度
```

### 4. 归一化方法选择

```
归一化方法对比：

原始值: [0.85, 0.90, 0.70, 0.95, 0.60]

Min-Max:  [0.714, 0.857, 0.286, 1.000, 0.000]
  → 保留分布，但极端值影响大

Z-Score:  [0.316, 0.949, -0.949, 1.581, -1.897]
  → 假设正态分布，有负数不易解释

Rank:     [0.50, 0.75, 0.25, 1.00, 0.00]
  → 丢失绝对量信息，只保留排序

Softmax:  [0.210, 0.258, 0.146, 0.307, 0.079]
  → 放大了差距（最小值的权重极低）

选择建议：
- 分布已知 → Z-Score
- 有极端异常值 → 鲁棒归一化
- 只看相对好坏 → Rank 归一化
- 需要区分度 → Softmax
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 将多维度指标融合为一个综合数值 | 消除权重选择的主观性 |
| 对多个 Agent 进行综合排名 | 自动确定"正确"的权重 |
| 识别 Agent 的最强和最弱维度 | 预测真实场景中的用户偏好 |
| 通过敏感性分析发现关键指标 | 处理指标之间的非线性交互 |
| 可视化多维度对比 | 解决维度之间的信息冗余（需要 PCA 等降维方法） |

## 与其他指标的关系

- **综合评分不是替代而是汇总**：各子指标仍然保留，综合评分只用于快速对比和决策
- **子指标是综合评分的原料**：子指标质量决定了综合评分的上限
- **成功率通常是最高权重的指标**：但在某些场景（如成本敏感场景）权重分配会不同
- **效率 + 成本指标通常合并处理**：避免在综合评分中过度惩罚 Agent 的消耗
- **可靠性 + 鲁棒性指标之间的相关性需要关注**：高相关性的指标可能造成重复计算

```
综合评分体系分层：

        第 1 层：子指标层
        成功率   效率   鲁棒性   可靠性   适应性
          │       │       │       │       │
        第 2 层：维度聚合层
        有效性得分   效率得分   稳定性得分
          │            │           │
        第 3 层：综合评分层
              综合评分
          （加权 / TOPSIS / 雷达图）
```

## 工程优化方向

1. **自动化权重学习（Automated Weight Learning）**：通过历史数据（用户反馈、任务完成情况）自动学习最优权重，减少人工干预

```python
def learn_weights_from_feedback(historical_data: list) -> dict:
    """
    从历史反馈数据中学习指标权重
    historical_data: [{"metrics": {...}, "user_rating": 0-5}]
    """
    from sklearn.linear_model import LinearRegression
    
    X = []
    y = []
    
    for data in historical_data:
        metrics = data["metrics"]
        X.append([
            metrics.get("success_rate", 0),
            metrics.get("efficiency", 0),
            metrics.get("robustness", 0),
            metrics.get("reliability", 0),
            metrics.get("adaptability", 0)
        ])
        y.append(data["user_rating"] / 5.0)
    
    model = LinearRegression()
    model.fit(X, y)
    
    weights = model.coef_
    weights = np.maximum(weights, 0)
    weights = weights / np.sum(weights)
    
    return {
        "success_rate": weights[0],
        "efficiency": weights[1],
        "robustness": weights[2],
        "reliability": weights[3],
        "adaptability": weights[4]
    }
```

2. **动态加权（Dynamic Weighting）**：根据任务类型动态调整权重

```python
class DynamicWeightScorer:
    """根据任务类型动态调整权重"""
    
    def __init__(self):
        self.task_profiles = {
            "factual_qa": {
                "success_rate": 0.50,
                "efficiency": 0.20,
                "robustness": 0.10,
                "reliability": 0.20
            },
            "creative_writing": {
                "success_rate": 0.25,
                "efficiency": 0.10,
                "robustness": 0.15,
                "reliability": 0.10,
                "adaptability": 0.40
            },
            "data_analysis": {
                "success_rate": 0.35,
                "efficiency": 0.30,
                "robustness": 0.15,
                "reliability": 0.15,
                "adaptability": 0.05
            },
            "customer_service": {
                "success_rate": 0.40,
                "efficiency": 0.10,
                "robustness": 0.25,
                "reliability": 0.25
            },
            "code_generation": {
                "success_rate": 0.40,
                "efficiency": 0.15,
                "robustness": 0.15,
                "reliability": 0.20,
                "adaptability": 0.10
            }
        }
    
    def score(self, task_type: str, 
               metrics: dict) -> dict:
        """根据任务类型动态评分"""
        if task_type not in self.task_profiles:
            task_type = "factual_qa"  # 默认
        
        weights = self.task_profiles[task_type]
        scorer = WeightedScorer(weights)
        
        result = scorer.score(metrics)
        result["task_profile"] = task_type
        
        return result
```

3. **集成评分（Ensemble Scoring）**：同时使用多种评分方法，取综合结果

```python
class EnsembleScorer:
    """集成多种评分方法"""
    
    def __init__(self, methods: list):
        """
        methods: [{"name": str, "scorer": object, "weight": float}]
        """
        self.methods = methods
    
    def ensemble_score(self, agent_scores: dict) -> dict:
        """
        集成评分：多种方法投票
        """
        method_results = []
        
        for method in self.methods:
            scorer = method["scorer"]
            result = scorer.score(agent_scores)
            method_results.append({
                "method": method["name"],
                "score": result["composite_score"]
                if "composite_score" in result
                else result.get("scores", [0])[0],
                "weight": method["weight"]
            })
        
        # 加权集成
        total_weight = sum(m["weight"] for m in method_results)
        ensemble = sum(
            m["score"] * m["weight"] for m in method_results
        ) / total_weight
        
        # 一致性：各种方法的结果是否接近
        scores = [m["score"] for m in method_results]
        consistency = 1.0 - float(np.std(scores))
        
        return {
            "ensemble_score": ensemble,
            "method_consistency": consistency,
            "method_details": method_results
        }
```

4. **敏感性分析报告**：每次输出综合评分时附带权重敏感性分析，让用户了解评分的稳定性

5. **场景化权重模板**：预置多种场景的权重模板（高可靠性场景、低成本场景、高效率场景），用户直接选用而非从零定义

## 确定适用场景

综合评分最适合：
- 需要快速比较多个 Agent 的决策场景
- 上线门禁（判断 Agent 是否达到上线标准）
- A/B 测试的最终评判标准
- 向非技术 stakeholders 汇报 Agent 性能

不适合：
- 需要深入诊断 Agent 问题的场景（此时应该看子指标）
- 指标之间存在强烈的非线性交互（此时需要更复杂的模型）
- 权重无法达成共识的团队（政治问题无法用数学解决）
- 仅有一个关键指标的场景（综合评分反而稀释了信息）
