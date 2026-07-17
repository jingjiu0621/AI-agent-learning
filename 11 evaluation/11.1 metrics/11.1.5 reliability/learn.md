# 11.1.5 reliability — 可靠性：一致性 / 可复现性

## 简单介绍

可靠性衡量 Agent 在相同输入下多次运行时，能否产生一致的、可复现的结果。传统软件中，给定相同的输入必然产生相同的输出——这是理所应当的。但在 LLM-based Agent 的世界里，同一个 Prompt 每次调用可能得到不同的回答，同一个任务 Agent 可能走完全不同的工具调用路径。可靠性评估回答一个关键问题：**Agent 的结果是可信的、可复现的，还是每次都在碰运气？**

## 基本原理

### 一致性的三个维度

```
┌──────────────────────────────────────────────────────────────┐
│  输出一致性（Output Consistency / Determinism）                │
│  相同的输入 → 相同的输出                                       │
│  衡量：同 Prompt 多次运行，输出的语义相似度                      │
│  问题："今天天气怎么样" 每次回答是否一致？                       │
├──────────────────────────────────────────────────────────────┤
│  行为一致性（Behavioral Consistency / Action Reproducibility） │
│  相同的任务 → 相同的工具调用路径                                 │
│  衡量：工具选择、参数、调用顺序的稳定性                          │
│  问题：Agent 每次都查同一张表，还是有时查错表？                   │
├──────────────────────────────────────────────────────────────┤
│  质量一致性（Quality Stability）                                │
│  多次运行的结果质量是否在可接受范围内波动                         │
│  衡量：输出评分的方差、极差                                     │
│  问题：这次回答 90 分，下次只有 40 分？                          │
└──────────────────────────────────────────────────────────────┘
```

### 统计基础

```
多次运行结果的分布分析：

运行 1:  得分 = 0.85
运行 2:  得分 = 0.82
运行 3:  得分 = 0.88
运行 4:  得分 = 0.45   ← 异常值（漂移）
运行 5:  得分 = 0.83
运行 6:  得分 = 0.86
运行 7:  得分 = 0.81
运行 8:  得分 = 0.84
运行 9:  得分 = 0.87
运行 10: 得分 = 0.83

均值 μ = 0.804
标准差 σ = 0.129
变异系数 CV = σ/μ = 0.160 → > 0.1，存在不可忽略的波动
P(得分 < 0.7) = 约 20% → 可靠性不足
```

## 背景

### 传统软件 vs LLM Agent 的可靠性差距

传统软件的可靠性是确定的：`f(x) = y`，对于同一个 x，永远返回同一个 y。测试只需要验证一次，通过了就是通过了。

LLM-based Agent 天然是非确定性的（Non-deterministic）：

- **Sampling 随机性**：LLM 的 temperature > 0 时，每次采样结果不同
- **路径分歧**：Agent 可能在不同运行中选择不同的工具调用顺序
- **上下文敏感**：微小的 Prompt 变化、甚至 Token 对齐差异都可能导致输出漂移
- **模型更新**：底层模型版本更新后，相同 Prompt 的行为可能变化

这意味着：**Agent 测试的一次通过不代表可靠**——你需要运行多次才能评估其真实稳定性。

### 非确定性的来源

```
非确定性来源树
                        LLM 输出随机性
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
       采样参数          模型推理路径        硬件/环境
      ┌──┴──┐         ┌────┴────┐        ┌────┴────┐
      │     │         │         │        │         │
   temperature  top_p  Kernel 算子   GPU 浮点   API 版本
   presence_penalty     优化差异   舍入差异     差异
```

## 核心矛盾

**创造性（Creativity）vs 一致性（Consistency）的 trade-off。** LLM 的 temperature 参数直接控制这个平衡：

```
temperature = 0.0 → 完全确定（greedy decoding），一致性最高，但可能机械重复
temperature = 0.3 → 轻度随机，适合事实性任务，一致性较好
temperature = 0.7 → 中等随机，适合创意写作，一致性明显下降
temperature = 1.0+ → 高度随机，一致性差，但可能产生意想不到的优质输出
```

核心问题是：Agent 既需要一定的创造性来应对开放任务，又需要足够的可靠性来确保生产环境稳定——这两个目标在根本上相互冲突。

## 评估方法

### 1. 多次运行一致性测试

最基本的可靠性评估：对同一个任务运行 N 次，测量输出的一致性。

```python
import numpy as np
from typing import Any, Callable
from sklearn.metrics.pairwise import cosine_similarity
from sentence_transformers import SentenceTransformer


class ConsistencyTester:
    """多次运行一致性测试"""
    
    def __init__(self, n_runs: int = 10, threshold: float = 0.8):
        self.n_runs = n_runs
        self.threshold = threshold
        self.embedder = SentenceTransformer("all-MiniLM-L6-v2")
    
    def evaluate_consistency(self, agent_fn: Callable, 
                             task_input: str) -> dict:
        """
        对同一个任务多次运行 Agent，评估输出一致性
        """
        outputs = []
        trajectories = []
        
        for i in range(self.n_runs):
            result = agent_fn(task_input)
            outputs.append(result["output"])
            trajectories.append(result.get("trajectory", []))
        
        # 语义一致性矩阵
        embeddings = self.embedder.encode(outputs)
        sim_matrix = cosine_similarity(embeddings)
        
        # 提取上三角（不含对角线）的相似度
        n = len(outputs)
        upper_tri = []
        for i in range(n):
            for j in range(i + 1, n):
                upper_tri.append(sim_matrix[i][j])
        
        semantic_consistency = float(np.mean(upper_tri))
        semantic_std = float(np.std(upper_tri))
        
        # 统计完全一致的比例（字面相等）
        exact_match_count = 0
        for i in range(n):
            for j in range(i + 1, n):
                if outputs[i] == outputs[j]:
                    exact_match_count += 1
        total_pairs = n * (n - 1) / 2
        exact_consistency = exact_match_count / total_pairs
        
        # 行为路径一致性
        trajectory_consistency = self._compute_trajectory_consistency(
            trajectories)
        
        return {
            "task": task_input,
            "n_runs": self.n_runs,
            "semantic_consistency": semantic_consistency,
            "semantic_std": semantic_std,
            "exact_match_rate": exact_consistency,
            "trajectory_consistency": trajectory_consistency,
            "passed": semantic_consistency >= self.threshold,
            "outputs": outputs,
            "similarity_matrix": sim_matrix.tolist()
        }
    
    def _compute_trajectory_consistency(self, 
                                         trajectories: list) -> float:
        """
        计算工具调用路径的一致性
        基于路径编辑距离（归一化）
        """
        if not trajectories or not trajectories[0]:
            return 1.0
        
        from Levenshtein import ratio as lev_ratio
        
        n = len(trajectories)
        path_sims = []
        
        for i in range(n):
            path_i = [step.get("tool", "") for step in trajectories[i]]
            for j in range(i + 1, n):
                path_j = [step.get("tool", "") for step in trajectories[j]]
                # 将路径转换为字符串比较
                str_i = " -> ".join(path_i)
                str_j = " -> ".join(path_j)
                path_sims.append(lev_ratio(str_i, str_j))
        
        return float(np.mean(path_sims)) if path_sims else 1.0
```

### 2. 方差分析与变异系数

使用统计方法量化一致性的波动程度。

```python
class VarianceAnalyzer:
    """可靠性方差分析"""
    
    def __init__(self):
        self.cv_threshold = 0.1   # 变异系数阈值
        self.iqr_threshold = 0.15  # IQR 阈值
    
    def analyze_score_variance(self, scores: list) -> dict:
        """
        分析多次运行得分的方差
        scores: 每次运行的质量评分列表
        """
        arr = np.array(scores)
        mean = float(np.mean(arr))
        std = float(np.std(arr))
        variance = float(np.var(arr))
        cv = std / max(mean, 1e-8)
        
        # 四分位距
        q1 = float(np.percentile(arr, 25))
        q3 = float(np.percentile(arr, 75))
        iqr = q3 - q1
        
        # 极差
        range_ = float(np.max(arr) - np.min(arr))
        
        # 异常值检测（超过 2 倍标准差）
        outliers = []
        for i, score in enumerate(scores):
            if abs(score - mean) > 2 * std:
                outliers.append({"run": i, "score": score})
        
        return {
            "n_runs": len(scores),
            "mean": mean,
            "std": std,
            "variance": variance,
            "coefficient_of_variation": cv,
            "cv_assessment": "stable" if cv < self.cv_threshold 
                        else "moderate" if cv < 2 * self.cv_threshold 
                        else "unstable",
            "q1": q1,
            "q3": q3,
            "iqr": iqr,
            "range": range_,
            "min": float(np.min(arr)),
            "max": float(np.max(arr)),
            "outlier_count": len(outliers),
            "outliers": outliers
        }
    
    def intraclass_correlation(self, 
                                score_matrix: list) -> dict:
        """
        计算组内相关系数（ICC）
        用于量化多次运行结果在不同维度上的一致性
        
        score_matrix: shape = [n_runs, n_dimensions]
        """
        arr = np.array(score_matrix)
        n_runs, n_dims = arr.shape
        
        # 总均值
        grand_mean = np.mean(arr)
        
        # 组间方差（不同运行间的差异）
        between_var = np.var(np.mean(arr, axis=1), ddof=1)
        
        # 组内方差（同一运行内不同维度的差异）
        within_var = np.mean(np.var(arr, axis=1, ddof=1))
        
        # ICC(1) - 单次测量可靠性
        icc_1 = between_var / (between_var + within_var)
        
        # ICC(k) - 多次测量均值可靠性
        icc_k = between_var / (between_var + within_var / n_runs)
        
        return {
            "icc_1": icc_1,
            "icc_k": icc_k,
            "interpretation": self._interpret_icc(icc_1)
        }
    
    def _interpret_icc(self, icc: float) -> str:
        if icc < 0.5:
            return "poor consistency"
        elif icc < 0.75:
            return "moderate consistency"
        elif icc < 0.9:
            return "good consistency"
        else:
            return "excellent consistency"
```

### 3. 语义一致性 vs 字面一致性

```python
class SemanticConsistencyChecker:
    """语义一致性与字面一致性的对比分析"""
    
    def __init__(self):
        self.embedder = SentenceTransformer("all-MiniLM-L6-v2")
    
    def compare_consistency_types(self, outputs: list) -> dict:
        """
        比较语义一致性和字面一致性
        场景：Agent 每次用不同的措辞表达相同的意思
        """
        n = len(outputs)
        
        # 字面一致：完全相同
        literal_matches = 0
        for i in range(n):
            for j in range(i + 1, n):
                if outputs[i].strip() == outputs[j].strip():
                    literal_matches += 1
        total_pairs = n * (n - 1) / 2
        literal_consistency = literal_matches / total_pairs
        
        # 语义一致：意思相同但措辞不同
        embeddings = self.embedder.encode(outputs)
        semantic_sims = []
        
        for i in range(n):
            for j in range(i + 1, n):
                sim = float(cosine_similarity(
                    [embeddings[i]], [embeddings[j]])[0][0])
                semantic_sims.append(sim)
        
        avg_semantic = float(np.mean(semantic_sims))
        semantic_consistent_pairs = sum(
            1 for s in semantic_sims if s > 0.85)
        semantic_consistency = semantic_consistent_pairs / len(semantic_sims)
        
        # 一致性差异分析
        consistency_gap = avg_semantic - literal_consistency
        
        return {
            "literal_consistency": literal_consistency,
            "semantic_consistency": semantic_consistency,
            "avg_semantic_similarity": avg_semantic,
            "consistency_gap": consistency_gap,
            # gap 大 → Agent 擅长换说法但保持意思一致
            # gap 小 → Agent 要么每次都一样（好），要么每次都不一样（坏）
            "interpretation": (
                "high_semantic_low_literal: 语义稳定但表达多样"
                if consistency_gap > 0.3
                else "balanced: 语义和字面接近"
                if consistency_gap < 0.1
                else "moderate: 存在一定表达差异"
            )
        }
```

### 4. 工具调用路径稳定性

```python
class TrajectoryStabilityChecker:
    """工具调用路径稳定性分析"""
    
    def analyze_trajectory_stability(self, 
                                      all_runs: list) -> dict:
        """
        分析多次运行中的工具调用路径稳定性
        all_runs: [{"run_id": int, "steps": [{"tool": str, "params": dict}]}]
        """
        # 1. 路径多样性：有多少种不同的工具调用序列？
        path_signatures = {}
        for run in all_runs:
            sig = tuple(step["tool"] for step in run["steps"])
            if sig not in path_signatures:
                path_signatures[sig] = []
            path_signatures[sig].append(run["run_id"])
        
        n_runs = len(all_runs)
        n_paths = len(path_signatures)
        
        # 路径多样性指数（1 = 全部一样，n_runs = 每条都不一样）
        path_diversity = n_paths / n_runs
        
        # 2. 主导路径占比
        dominant_path = max(path_signatures.items(), 
                           key=lambda x: len(x[1]))
        dominant_path_ratio = len(dominant_path[1]) / n_runs
        
        # 3. 参数稳定性（相同工具的参数一致性）
        param_stability = self._evaluate_param_stability(all_runs)
        
        # 4. 步骤数稳定性
        step_counts = [len(run["steps"]) for run in all_runs]
        step_cv = float(np.std(step_counts) / max(np.mean(step_counts), 1))
        
        return {
            "n_runs": n_runs,
            "distinct_trajectories": n_paths,
            "path_diversity_index": path_diversity,
            "dominant_path": list(dominant_path[0]),
            "dominant_path_ratio": dominant_path_ratio,
            "dominant_path_runs": dominant_path[1],
            "param_stability": param_stability,
            "step_count_mean": float(np.mean(step_counts)),
            "step_count_std": float(np.std(step_counts)),
            "step_count_cv": step_cv,
            "is_trajectory_stable": dominant_path_ratio > 0.7
        }
    
    def _evaluate_param_stability(self, all_runs: list) -> dict:
        """
        评估相同工具被调用时参数的稳定性
        """
        tool_params = {}
        
        for run in all_runs:
            for step in run["steps"]:
                tool = step["tool"]
                params = frozenset(step.get("params", {}).items())
                if tool not in tool_params:
                    tool_params[tool] = {}
                if params not in tool_params[tool]:
                    tool_params[tool][params] = 0
                tool_params[tool][params] += 1
        
        # 对每个工具计算参数一致性
        tool_stability = {}
        for tool, param_variants in tool_params.items():
            total_calls = sum(param_variants.values())
            dominant_params = max(param_variants.items(), 
                                  key=lambda x: x[1])
            tool_stability[tool] = {
                "variant_count": len(param_variants),
                "dominant_params_ratio": dominant_params[1] / total_calls,
                "dominant_params": dict(dominant_params[0])
            }
        
        avg_stability = float(np.mean([
            v["dominant_params_ratio"] 
            for v in tool_stability.values()
        ])) if tool_stability else 1.0
        
        return {
            "per_tool": tool_stability,
            "average_param_stability": avg_stability
        }
```

### 5. 置信度校准

衡量 Agent 对自己输出的置信度与实际正确率之间的一致性。

```python
class ConfidenceCalibration:
    """置信度校准评估"""
    
    def __init__(self, n_bins: int = 10):
        self.n_bins = n_bins
    
    def evaluate_calibration(self, 
                              predictions: list) -> dict:
        """
        评估置信度校准质量
        predictions: [{"confidence": 0.0-1.0, "correct": bool}]
        """
        confidences = np.array([p["confidence"] for p in predictions])
        correct = np.array([p["correct"] for p in predictions])
        
        # ECE（Expected Calibration Error）
        bin_boundaries = np.linspace(0, 1, self.n_bins + 1)
        ece = 0.0
        bin_data = []
        
        for i in range(self.n_bins):
            bin_mask = (confidences >= bin_boundaries[i]) & \
                       (confidences < bin_boundaries[i + 1])
            if not np.any(bin_mask):
                continue
            
            bin_conf = np.mean(confidences[bin_mask])
            bin_acc = np.mean(correct[bin_mask])
            bin_size = np.sum(bin_mask)
            
            bin_error = abs(bin_conf - bin_acc)
            ece += bin_error * (bin_size / len(predictions))
            
            bin_data.append({
                "bin_range": f"{bin_boundaries[i]:.1f}-{bin_boundaries[i+1]:.1f}",
                "count": int(bin_size),
                "avg_confidence": float(bin_conf),
                "avg_accuracy": float(bin_acc),
                "calibration_error": float(bin_error)
            })
        
        # MCE（Maximum Calibration Error）
        mce = float(max(
            d["calibration_error"] for d in bin_data
        )) if bin_data else 0.0
        
        return {
            "ece": float(ece),
            "mce": mce,
            "calibration_assessment": "well_calibrated" if ece < 0.05
                        else "moderate" if ece < 0.1
                        else "poorly_calibrated",
            "n_samples": len(predictions),
            "bin_details": bin_data,
            "avg_confidence": float(np.mean(confidences)),
            "avg_accuracy": float(np.mean(correct))
        }
```

## 温度与可靠性的关系

```
一致性
 1.0 │temperature=0.0 (greedy)
     │  ●━━━━━━━━━━━━━━━━━━━━━━━━
     │
 0.9 │       temperature=0.3
     │         ●━━━━━━━━━
     │
 0.8 │              temperature=0.7
     │                ●━━━━━
     │
 0.7 │                    temperature=1.0
     │                      ●━━
     │
 0.6 │                         temperature=1.5
     │                           ●
     │
     └──────────────────────────────────────────
       事实性       日常        创意        探索
             任务类型对可靠性的要求
```

## 实现挑战

### 1. 什么算"相同输出"？

```
用户问："Python 的列表推导式和 map 函数哪个更好？"

输出 A: "列表推导式通常更 Pythonic，可读性更好，建议优先使用。"
输出 B: "建议使用列表推导式，它比 map 更符合 Python 的设计哲学，代码更清晰。"
输出 C: "这是一个权衡。列表推导式可读性更好，但 map 配合 lambda 在某些场景更简洁。"

问题：
- A 和 B 语义相同但措辞不同 → 应该算"一致"
- C 的观点不同 → 应该算"不一致"
- 边界情况：A 和 B 都是合理答案，但 C 也是合理的

解决方案：定义"语义等价"的阈值，但阈值的选取本身是主观的。
```

### 2. 可靠性与任务难度的关系

```python
def reliability_vs_difficulty(agent, tasks: list) -> dict:
    """
    分析不同难度任务下的可靠性
    通常：任务越难，可靠性越低
    """
    difficulty_levels = {"easy": [], "medium": [], "hard": []}
    
    for task in tasks:
        level = task["difficulty"]
        scores = []
        
        for _ in range(10):
            result = agent.run(task["input"])
            scores.append(result.get("quality_score", 0))
        
        difficulty_levels[level].append({
            "task": task["input"],
            "mean": float(np.mean(scores)),
            "std": float(np.std(scores)),
            "cv": float(np.std(scores) / max(np.mean(scores), 1e-8))
        })
    
    return {
        level: {
            "avg_reliability": 1 - float(np.mean([d["cv"] for d in data])),
            "avg_score": float(np.mean([d["mean"] for d in data])),
            "tasks": data
        }
        for level, data in difficulty_levels.items()
    }
```

### 3. 评估成本

```
运行次数与可靠性置信度的关系：

10 次运行 → 检出 ±15% 的波动 → 粗略评估（5 分钟）
30 次运行 → 检出 ±8% 的波动 → 标准评估（15 分钟）
100 次运行 → 检出 ±4% 的波动 → 严格评估（50 分钟）
1000 次运行 → 检出 ±1.5% 的波动 → 学术级评估（8+ 小时）

权衡：评估可靠性本身需要大量运行，导致评估成本呈线性增长。
```

### 4. 非确定性漂移

```python
class NonDeterministicDriftDetector:
    """检测 Agent 可靠性在时间维度上的漂移"""
    
    def detect_drift(self, 
                     historical_runs: list) -> dict:
        """
        检测可靠性随着时间的变化
        historical_runs: [{"date": str, "consistency_score": float}]
        """
        scores = [r["consistency_score"] for r in historical_runs]
        dates = [r["date"] for r in historical_runs]
        
        # 滑动窗口方差
        window_size = max(10, len(scores) // 10)
        windows = []
        
        for i in range(len(scores) - window_size + 1):
            window = scores[i:i + window_size]
            windows.append({
                "window_start": dates[i],
                "window_end": dates[i + window_size - 1],
                "mean": float(np.mean(window)),
                "std": float(np.std(window))
            })
        
        # 整体趋势
        first_avg = windows[0]["mean"] if windows else 0
        last_avg = windows[-1]["mean"] if windows else 0
        drift_amount = last_avg - first_avg
        
        return {
            "drift_detected": abs(drift_amount) > 0.05,
            "drift_amount": drift_amount,
            "drift_direction": "improving" if drift_amount > 0 
                         else "degrading",
            "current_reliability": last_avg,
            "baseline_reliability": first_avg,
            "windows": windows
        }
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 检测输出波动并量化波动程度 | 判断"哪种输出是更好的"（需要人工/LLM Judge） |
| 识别工具调用路径的分歧 | 消除 LLM 固有的非确定性（temperature > 0 必然有随机性） |
| 评估置信度校准质量 | 在单次运行中评估可靠性（需要多次运行） |
| 发现可靠性随时间的漂移 | 在不增加成本的前提下提高评估精度 |
| 对比不同 temperature 设置的一致性 | 区分"创造性差异"和"错误差异" |

## 与其他指标的关系

- **成功率**：高可靠性 + 低成功率 = Agent 稳定地失败，问题出在能力不足而非随机性
- **鲁棒性**：鲁棒性关注输入变化时输出的稳定性，可靠性关注输入不变时输出的稳定性
- **适应性**：高度适应性的 Agent 在跨场景时可能牺牲了单场景的可靠性
- **综合评分**：可靠性是所有指标中"最基础的"——没有可靠性，单次评估结果的价值大打折扣
- **效率指标**：提高可靠性（多次运行取多数投票）需要额外的 Token 和时间成本

```
可靠性与其他指标的权衡矩阵：

                    ┌──────┬──────┬──────┬──────┐
                    │ 成功率 │ 效率  │ 鲁棒性 │ 适应性 │
    ┌───────────────┼──────┼──────┼──────┼──────┤
    │ 高可靠性      │  促进 │  抑制 │  促进 │  抑制 │
    │（确定性输出）  │       │      │      │      │
    ├───────────────┼──────┼──────┼──────┼──────┤
    │ 低可靠性      │  抑制 │  促进 │  抑制 │  促进 │
    │（高创造力输出） │       │      │      │      │
    └───────────────┴──────┴──────┴──────┴──────┘
```

## 工程优化方向

1. **Seed 控制**：对于需要确定性的场景，固定随机种子（模型级别 + 应用级别双重控制）

2. **Temperature 调度**：简单任务用低 temperature（0.0-0.2）确保可靠性，复杂任务用中等 temperature（0.3-0.5）保留推理灵活性

3. **多数投票（Self-Consistency）**：运行 N 次，选择最一致的输出或多数结果，牺牲成本换取可靠性

```python
def self_consistency_voting(agent_fn: Callable, 
                             task_input: str,
                             n_runs: int = 5,
                             strategy: str = "semantic_majority") -> dict:
    """
    自一致性多数投票提升可靠性
    """
    outputs = []
    for _ in range(n_runs):
        outputs.append(agent_fn(task_input))
    
    if strategy == "semantic_majority":
        # 找到与其他输出语义相似度最高的输出
        embedder = SentenceTransformer("all-MiniLM-L6-v2")
        embs = embedder.encode(outputs)
        
        best_idx = 0
        best_avg_sim = -1
        
        for i in range(len(outputs)):
            sims = [cosine_similarity([embs[i]], [embs[j]])[0][0]
                    for j in range(len(outputs)) if j != i]
            avg_sim = float(np.mean(sims))
            if avg_sim > best_avg_sim:
                best_avg_sim = avg_sim
                best_idx = i
        
        return {
            "selected_output": outputs[best_idx],
            "consensus_score": best_avg_sim,
            "n_runs": n_runs,
            "all_outputs": outputs
        }
    
    elif strategy == "exact_majority":
        from collections import Counter
        counter = Counter(outputs)
        most_common = counter.most_common(1)[0]
        
        return {
            "selected_output": most_common[0],
            "consensus_score": most_common[1] / n_runs,
            "n_runs": n_runs,
            "all_outputs": outputs
        }
```

4. **输出标准化（Output Normalization）**：通过后处理将输出映射到标准格式，减少表达差异

```
原始输出 → 标准化管道：

"今天天气很好，温度大概 25 度左右。"
    → 正则化：去除语气词
    → 格式统一：温度统一为 "XX°C"
    → 关键信息提取：{"weather": "sunny", "temperature": 25}
    → 标准化输出：在标准化结构上比较，而非原文
```

5. **可靠性监控看板**：持续追踪每个 Agent 版本的一致性指标，建立可靠性基线

6. **自适应 Temperature 调节**：根据任务类型自动选择最优 temperature，事实性任务用低温度，创意任务用高温度

## 确定适用场景

可靠性评估对以下场景至关重要：
- 金融交易、医疗建议等需要一致性输出的高风险场景
- 面向外部客户的 Agent（不一致的回答损害品牌信任）
- 自动化流水线中的 Agent（下游依赖上游的确定输出）
- 合规场景（需要可审计、可复现的 Agent 行为）
- 需要差分测试的场景（比较两个 Agent 版本时，需要排除随机噪声）

以下场景可以容忍较低的可靠性：
- 创意写作、头脑风暴类 Agent
- 内部工具（一致性要求低，有用即可）
- 探索性分析（每次分析都想看到不同角度）
- 原型验证阶段（先验证可行性，再优化可靠性）
