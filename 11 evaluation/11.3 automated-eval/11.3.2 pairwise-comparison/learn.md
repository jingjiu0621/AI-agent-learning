# 11.3.2 pairwise-comparison — 成对比较与 A/B 测试

## 简单介绍

Pairwise Comparison（成对比较）是评估 Agent 的另一种自动评估模式——不直接给单个输出打分，而是让 LLM 或人类在两个候选输出中**选择更好的一个**。

```
方法 A: 直接评分 (Pointwise)
  Agent 输出 ──→ 给我打 1-5 分 ──→ 得分: 4.2

方法 B: 成对比较 (Pairwise)
  输出 A ──→ 比较 A 和 B ──→ A 比 B 好
  输出 B ──→ 哪个更好？    ──→ (胜率统计)
```

核心思想：**人类（和 LLM）在"比较"任务上比"绝对评分"任务更一致。**

## 基本原理

### 为什么成对比较可能更可靠？

```
绝对评分（给 1-5 分）的问题：
  评分者 A: "3 分算中等"
  评分者 B: "3 分算很差"
  → 两个人的 3 分不是同一个概念
  → 分数不可比

成对比较的优势：
  问："A 和 B 哪个更好？"
  → 大部分人（和 LLM）会给出相同的答案
  → 结果更一致
```

这个直觉已经被大量研究证实——成对比较的评分者间一致性显著高于直接评分（Cohen's Kappa 通常高 0.1-0.2）。

### Elo 评分系统

成对比较的结果通常使用 Elo 评分系统来转化为排名分数。Elo 系统最初用于国际象棋排名，被 Chatbot Arena 引入 LLM 评估领域后成为标准做法。

```
Elo 评分基本原理：

  每个参赛者有一个 Elo 分数（初始值 1000/1200）
  每次比较后分数更新：
    预期胜率 E(A) = 1 / (1 + 10^((RB - RA) / 400))
    实际结果 S(A) = 1 (A 获胜), 0.5 (平局), 0 (A 失败)
    更新: RA_new = RA + K × (S(A) - E(A))

  其中 K 是"灵敏度系数"（通常 4-32）
  K 越大 → 新比赛对分数影响越大
  K 越小 → 分数越稳定

  400 是缩放因子——Elo 分差 400 意味着强者的胜率是弱者的 10 倍
```

## 背景

### Pairwise 评估的起源

```
在 LLM-as-Judge 之前...

人工评估时代：
  "A 和 B，哪个翻译更好？"
  → 比较任务 vs 评分任务
  → 比较更直观，评分者分歧更小

Chatbot Arena (LMSYS) 的贡献：
  让用户匿名比较两个模型的回答
  收集海量成对偏好 → 用 Elo 系统排名
  证明了成对比较在规模上的可行性

MT-Bench：
  设计了 80 道多轮问题
  使用 GPT-4 作为法官进行成对比较
  首次系统验证了 LLM-as-Judge 在成对比较中的可靠性

AlpacaEval：
  使用 GPT-4 作为法官，比较待评估模型和参考模型（GPT-3.5/4）
  自动计算"胜率"（Win Rate）作为指标
```

### 2025-2026 年关于 Pairwise 的争议与进展

```
The Comparative Trap (BlackboxNLP 2025):
  ─────────────────────────────────────────
  关键发现：成对比较会放大 LLM 法官的偏好偏差。
  当 LLM 被强制在 A 和 B 之间选择时，
  它会表现出比"合理"更强的偏好——即使 A 和 B 质量相近。

  这个放大的偏好导致排名不稳定：
    • 微小的胜率差异被放大为显著的排名差距
    • 不同的法官模型可能给出完全不同的排名顺序
    • 法官特别倾向于选择"更符合自己风格"的输出

Pairwise or Pointwise? (arXiv 2504.14716):
  ─────────────────────────────────────────
  系统性比较两种协议：
    • Pointwise (Rubric 评分) → 偏差方差更低
    • Pairwise (成对比较) → 排名敏感度更高
    • 没有绝对优劣——取决于具体目标

  核心建议：
    目标是"精确排名"→ 用 Pairwise
    目标是"质量监控"→ 用 Pointwise/Rubric
    希望两者兼得 → 用混合方法 (PaTaRM)

PaTaRM: Preference-aware Task-adaptive Reward Modeling (ACL 2026):
  ─────────────────────────────────────────
  提出融合方案：
    同时利用 Pairwise 和 Pointwise 信号
    任务自适应地调整两种信号的权重
    在排名精度和偏差方差之间取得平衡
```

## 核心矛盾

**成对比较的统计效率 vs 偏差放大。**

```
成对比较的优势：
  • 更符合人类直觉——比较比打分容易
  • 评分者间一致性更高
  • 可以产生精细排名（Elo 分数）

成对比较的问题：
  • 比较次数爆炸——N 个输出需要 O(N²) 次比较
  • "比较陷阱"——LLM 在强制选择时放大偏好
  • 排名不稳定——少量偏好翻转导致排名巨变
  • 信息损失——只知道"谁赢了"，不知道"赢了多少"

IBM Research (ICLR 2026) 的惊人发现：
  在 Chatbot Arena 中，只需改变 3-5% 的偏好数据
  （几十条样本），就能翻转前排模型的排名顺序。
  意味着目前的 Elo 排名可能并不可靠。
```

## 代码实现

```python
import random
import math
from typing import List, Optional
from dataclasses import dataclass
from itertools import combinations


@dataclass
class ComparisonResult:
    """单次成对比较的结果"""
    winner: str      # "A", "B", or "TIE"
    rationale: str
    judge_model: str


class PairwiseJudge:
    """成对比较法官"""

    def __init__(self, llm_client, model_name: str):
        self.llm = llm_client
        self.model_name = model_name

    def compare(self, output_a: str, output_b: str,
                task: str) -> ComparisonResult:
        """
        比较两个输出，返回哪个更好。

        通过交换 A/B 位置取平均来消除位置偏差。
        """
        # 正向比较
        result_forward = self._single_comparison(
            output_a, output_b, task, order="AB"
        )

        # 反向比较（交换位置）
        result_backward = self._single_comparison(
            output_a, output_b, task, order="BA"
        )

        # 综合结果
        return self._resolve_comparison(
            result_forward, result_backward
        )

    def _single_comparison(self,
                            output_a: str,
                            output_b: str,
                            task: str,
                            order: str) -> str:
        """单次比较（不交换位置）"""

        if order == "AB":
            first, second = output_a, output_b
        else:
            first, second = output_b, output_a

        prompt = f"""你是一个严格的 AI 输出评审专家。请比较以下两个
Agent 对同一任务的回答，判断哪个更好。

## 任务
{task}

## 回答 A
{first}

## 回答 B
{second}

请输出你选择的更好回答（A 或 B 或 TIE），并说明理由。
格式：{{"choice": "A/B/TIE", "rationale": "..."}}
"""
        response = self.llm.chat.completions.create(
            model=self.model_name,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.0,
            response_format={"type": "json_object"}
        )

        import json
        data = json.loads(response.choices[0].message.content)
        choice = data.get("choice", "TIE")

        # 如果是反向比较，将结果映射回原始 A/B
        if order == "BA":
            if choice == "A":
                return "B"
            elif choice == "B":
                return "A"
            else:
                return "TIE"
        return choice

    def _resolve_comparison(self,
                             forward: str,
                             backward: str) -> ComparisonResult:
        """解析两次比较的综合结果"""
        if forward == backward:
            winner = forward
        elif forward == "TIE" or backward == "TIE":
            winner = "TIE"
        else:
            # 两次结果不一致（A>B but B>A 这种矛盾）
            winner = "TIE"  # 保守：视为平局

        return ComparisonResult(
            winner=winner,
            rationale=(f"Forward: {forward}, Backward: {backward}"),
            judge_model=self.model_name
        )


class EloRating:
    """Elo 评分系统"""

    def __init__(self, initial_rating: float = 1000.0,
                 k_factor: float = 16.0):
        self.ratings = {}
        self.initial_rating = initial_rating
        self.k_factor = k_factor
        self.history = []

    def get_rating(self, agent_id: str) -> float:
        return self.ratings.get(agent_id, self.initial_rating)

    def expected_score(self, rating_a: float,
                       rating_b: float) -> float:
        """计算 A 对 B 的预期胜率"""
        return 1.0 / (1.0 + 10.0 ** ((rating_b - rating_a) / 400.0))

    def update(self, winner_id: str, loser_id: str,
               is_tie: bool = False):
        """更新 Elo 分数"""
        r_winner = self.get_rating(winner_id)
        r_loser = self.get_rating(loser_id)

        if is_tie:
            # 平局：双方预期胜率之差的一半
            e_winner = self.expected_score(r_winner, r_loser)
            e_loser = self.expected_score(r_loser, r_winner)
            s_winner = 0.5
            s_loser = 0.5
        else:
            e_winner = self.expected_score(r_winner, r_loser)
            e_loser = self.expected_score(r_loser, r_winner)
            s_winner = 1.0
            s_loser = 0.0

        new_winner = r_winner + self.k_factor * (s_winner - e_winner)
        new_loser = r_loser + self.k_factor * (s_loser - e_loser)

        self.ratings[winner_id] = new_winner
        self.ratings[loser_id] = new_loser

        self.history.append({
            "winner": winner_id,
            "loser": loser_id,
            "tie": is_tie,
            "ratings": {
                winner_id: new_winner,
                loser_id: new_loser
            }
        })

    def batch_compare(self, comparisons: List[tuple]):
        """批量处理多个比较结果"""
        for comp in comparisons:
            if comp[2] == "TIE":
                self.update(comp[0], comp[1], is_tie=True)
            else:
                self.update(comp[0], comp[1])

    def get_ranking(self) -> List[tuple]:
        """获取排名列表"""
        sorted_ratings = sorted(
            self.ratings.items(),
            key=lambda x: x[1],
            reverse=True
        )
        return [
            (agent_id, round(rating, 1))
            for agent_id, rating in sorted_ratings
        ]


class AgentTournament:
    """Agent 锦标赛——多轮成对比较"""

    def __init__(self, judge: PairwiseJudge,
                 tasks: List[str]):
        self.judge = judge
        self.tasks = tasks
        self.elo = EloRating()

    def run_tournament(self,
                       agent_outputs: dict) -> dict:
        """
        运行完整锦标赛。

        Args:
            agent_outputs: {agent_name: {task_id: output}}

        Returns:
            排名、Elo 分数、比较细节
        """
        agents = list(agent_outputs.keys())

        # 为每个任务分配初始化 Elo 分
        for agent in agents:
            self.elo.ratings[agent] = self.elo.initial_rating

        comparisons_run = 0

        # 每个任务上运行成对比较
        for task_idx, task in enumerate(self.tasks):
            agent_pairs = list(combinations(agents, 2))
            random.shuffle(agent_pairs)

            for agent_a, agent_b in agent_pairs:
                output_a = agent_outputs[agent_a][task_idx]
                output_b = agent_outputs[agent_b][task_idx]

                result = self.judge.compare(
                    output_a, output_b, task
                )

                if result.winner == "TIE":
                    self.elo.update(agent_a, agent_b, is_tie=True)
                elif result.winner == "A":
                    self.elo.update(agent_a, agent_b)
                else:
                    self.elo.update(agent_b, agent_a)

                comparisons_run += 1

        return {
            "ranking": self.elo.get_ranking(),
            "comparisons_run": comparisons_run,
            "num_agents": len(agents),
            "num_tasks": len(self.tasks)
        }
```

## 实践指南

### 什么时候用 Pairwise，什么时候用 Pointwise

```
使用 Pairwise（成对比较）：
  ✅ 排名锦标赛（A vs B vs C，谁最好？）
  ✅ 模型选型（哪个模型当 Agent 更好？）
  ✅ 版本对比（新版本 vs 旧版本，有进步吗？）
  ✅ 精细化排名（需要区分质量很接近的系统）

使用 Pointwise（直接评分）：
  ✅ 质量监控（绝对质量有没有下降？）
  ✅ 门禁判断（这个版本可以上线吗？）
  ✅ 持续跟踪（质量随时间变化的趋势）
  ✅ 大规模评估（评估成本更重要时）

混合使用（推荐）：
  ✅ 先用 Pointwise 做快速筛选
  ✅ 对接近阈值的结果上 Pairwise 做精细判断
  ✅ 用 Pairwise 结果校准 Pointwise 的评分偏差
```

### 减少 Pairwise 偏差的推荐实践

```
1. 位置随机化
   每对输出随机分配 A/B 位置
   或者跑两次（AB 和 BA）取综合结果

2. 控制比较数量
   不必跑 O(N²) 次比较——用 Top-K 或聚类抽样
   例如：15 个 Agent，只比较 Top-5 和随机选 5 个

3. 使用不同法官复核
   主比较用一种法官，抽查用另一种法官
   如果两种法官的结果不一致，说明输出质量接近，标记为平局

4. 报告置信度
   不只是报告"Android A > Agent B"，
   还要报告"A 赢 B 的胜率是多少"
   (如 E(A) > E(B) 的差有多大)

5. 注意"平局"选项
   不要强制法官在 A/B 之间选择
   提供"平局/TIE"选项可以避免放大微小差异
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 产出精细的 Agent 能力排名 | 给出绝对质量分数（只知道谁更好，不知道好多少） |
| 减少评分者之间的尺度差异问题 | 避免比较陷阱导致的偏好放大 |
| 在质量接近的系统中保持一定区分度 | 在 N 很大的情况下保持统计可靠性（比较次数爆炸） |
| 通过交换位置等技术减少位置偏差 | 完全消除所有形式的法官偏差 |
| 产生可解释的比较结果和理由 | 适用于单个 Agent 的绝对质量判断（无比较对象） |

## 工程优化方向

1. **自适应比较采样**：不跑 O(N²) 次全比较，先用少量比较做粗排，然后对排名接近的 Agent 做精细比较，降低比较总次数

2. **跨任务结果融合**：不同任务上的比较结果分开计算 Elo，最后用加权平均融合，避免"容易任务上胜率高"的偏差

3. **法官分歧检测**：当正向和反向比较结果不一致时，不仅记为平局，还要分析分歧原因——可能是 A 和 B 质量确实接近，也可能是某个输出在特定方面突出

4. **增量更新**：新 Agent 加入时不需要重新跑所有比较，只需与新 Agent 相关的 N-1 次比较，Elo 系统自然支持增量更新

5. **校准胜率**：Elo 分数难以跨实验比较，可以额外报告"相对于参考模型的胜率"（Win Rate vs Reference），使结果更易理解
