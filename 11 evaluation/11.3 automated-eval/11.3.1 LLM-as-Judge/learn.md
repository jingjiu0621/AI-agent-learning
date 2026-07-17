# 11.3.1 LLM-as-Judge — 用语言模型评审 Agent

## 简单介绍

LLM-as-Judge 是当前自动评估领域最核心的方法。基本思路非常简单：**让一个 LLM 扮演评审者角色，对另一个 Agent 的输出进行评分、判别或反馈。**

```
        被评估的 Agent                   LLM 法官
    ┌──────────────────┐           ┌──────────────────┐
    │   输入任务        │           │   评审 Prompt     │
    │       ↓          │           │   "你是一个评审…"  │
    │   Agent 执行     │ ───输出──→│                  │
    │   (思考+工具+…)  │           │   分析 + 评分     │
    │       ↓          │           │       ↓          │
    │   最终输出        │           │   评分结果        │
    └──────────────────┘           └──────────────────┘
```

这个方法最早由 MT-Bench、AlpacaEval 等基准广泛使用，后来成为 Agent 评估的标准做法。2025-2026 年，LLM-as-Judge 已经发展出成熟的实践体系：结构化评分标准（Rubric）、多法官投票（LLM Juries）、校准技术、偏差检测等。

## 基本原理

### LLM-as-Judge 的工作流程

```
步骤 1: 定义评估标准
  ┌──────────────────────────────────────────┐
  │ "请从以下维度评分（1-5 分）：             │
  │   1. 准确性：答案是否准确无误              │
  │   2. 完整性：是否覆盖了所有必要信息         │
  │   3. 效率：是否用最少的步骤完成任务         │
  │   4. 安全性：是否有任何不安全的内容         │
  │   请以 JSON 格式输出评分和理由"            │
  └──────────────────────────────────────────┘

步骤 2: 构造评审 Prompt
  System: "你是一个严格的 Agent 评审专家……"
  User:   "以下是需要评审的任务和 Agent 的输出……"

步骤 3: 调用 LLM 评审
  judge_response = llm.chat(evaluation_prompt)

步骤 4: 解析评分结果
  score = parse_json(judge_response)

步骤 5: 质量检查
  if check_judge_reliability(score):
      record_score(score)
  else:
      re_evaluate()  # 法官输出格式不对，重试
```

### 三种评审模式

```python
# 模式 1: 直接评分 (Scoring)
SCORING_PROMPT = """
你是一个 Agent 任务评审专家。请对以下 Agent 的输出进行评分。

任务：{task_description}
Agent 输出：{agent_output}

请从以下维度评分（1-5 分）：
{criteria}

请以 JSON 格式输出：
{{
    "scores": {{"维度名": 分数}},
    "total_score": 总分,
    "rationale": "评分解释"
}}
"""

# 模式 2: 通过/不通过 (Pass/Fail)
PASS_FAIL_PROMPT = """
判断以下 Agent 是否成功完成了任务。

任务要求：{task_description}
Agent 输出：{agent_output}

判断标准：{pass_criteria}

请回答：PASS 或 FAIL，并简要说明理由。
格式：{{"verdict": "PASS/FAIL", "reason": "..."}}
"""

# 模式 3: 错误分析 (Error Analysis)
ERROR_ANALYSIS_PROMPT = """
分析以下 Agent 在执行任务时出现的错误。

任务：{task_description}
Agent 执行轨迹：{trajectory}

请识别：
1. 错误发生在哪一步？
2. 错误的类型是什么？（推理错误/工具选择/参数错误/其他）
3. 错误的根本原因是什么？
4. 如何修复？

以 JSON 格式输出分析结果。
"""
```

## 背景

### 从传统指标到 LLM-as-Judge

```
传统自动评估指标及其局限性：

BLEU (机器翻译)
  基于 n-gram 精确匹配
  问题：Agent 输出可能和参考答案完全不同但正确
  例：任务"查北京天气"，Agent 说"北京今天 25°C"，
      参考答案"北京当前气温 25 度"→ BLEU 得分很低但内容正确

ROUGE (文本摘要)
  基于召回率的 n-gram 匹配
  问题：同样不适合同义表达

BERTScore
  基于 BERT 嵌入的语义相似度
  比 BLEU/ROUGE 好，但：
  - 无法判断事实正确性
  - 对否定句敏感度不够
  - 无法评估多步推理质量

LLM-as-Judge 的优势：
  - 可以理解语义而不是词面匹配
  - 可以判断事实正确性
  - 可以利用推理能力分析多步骤回答
  - 可以给出维度的多维度评价
  - 可以提供评分理由（可解释性）
```

### LLM-as-Judge 的核心假设

LLM-as-Judge 的有效性依赖一个关键假设：**评审 LLM 的能力要足够强，能够可靠地判别被评审输出的质量。**

```
三个层次的能力要求：

1. 理解能力
   读懂任务描述和 Agent 输出
   理解评估标准的含义

2. 判别能力
   能区分"好"和"更好"
   能识别细微的错误
   能在多个维度上独立评分

3. 自洽能力
   对相似的输出给出相似的评分
   评分标准稳定（不会今天严明天松）
   不受无关因素影响（长度、格式、位置）

经验法则：
  使用与被评估模型相同或更强的模型作为 Judge
  使用不同模型家族的 Judge 可以减少 self-enhancement bias
```

## 当前实践与最新成果

### 2025-2026 年的关键进展

```
研究方向                  核心发现                                        影响
───────────────────  ─────────────────────────────────────────────  ─────────────────
Calibrate Don't      从噪声 LLM 评判中做标签高效估计                   降低了人工标注需求
Curate (2025)        不需要大量人工标注即可获得可靠信号

PaTaRM (ACL 2026)    Pairwise + Pointwise 信号融合                    融合了两种方法的优势
                    偏好感知的自适应奖励建模                            适用面更广

JudgeBench           LLM 法官能力的标准化评估基准                      提供了法官选型的依据
(ICLR 2025)          GPT-5/Claude 5 作为法官准确率最高

Multi-Judge Voting   3+ 弱法官集成效果优于 1 个强法官                   降低了法官偏差风险
(LLM Juries)
                    独立采样减少系统性偏差

ContextualJudgeBench 长上下文下法官准确率显著下降                       长文本 Agent 评估
(ACL 2025)                                                            需要专门考虑

Rubric-guided       结构化评分标准使方差降低 30-50%                     2025 后的主流方法
Eval (IEEE 2025)    比自由提示的 LLM 评判更可靠
```

### 法官选型策略

不同法官模型的评估质量差异显著：

```python
JUDGE_SELECTION_GUIDE = {
    "高精度场景": {
        "推荐法官": ["GPT-5", "Claude 5", "Gemini 2.5 Pro"],
        "配置": "多法官投票（3+），严格 Rubric",
        "适用": "上线门禁、关键决策、报告数据",
        "成本": "高",
        "参考准确率": "~90-95%（JudgeBench 基准）"
    },
    "常规评估场景": {
        "推荐法官": ["GPT-4o", "Claude 4 Sonnet", "Gemini 2.0"],
        "配置": "单法官 + 结构化标准 + 人工抽检",
        "适用": "日常开发迭代、回归测试",
        "成本": "中",
        "参考准确率": "~80-88%"
    },
    "低成本快速评估": {
        "推荐法官": ["LLaMA-3-70B", "Qwen2.5-72B", "DeepSeek-V3"],
        "配置": "简单 Rubric + 批量评分",
        "适用": "快速验证、大规模预筛选",
        "成本": "低",
        "参考准确率": "~70-80%"
    },
    "特殊场景": {
        "跨模型评估": "使用不同模型家族法官（避免 self-enhancement）",
        "长上下文评估": "使用上下文窗口大的法官模型",
        "多语言评估": "使用目标语言能力强的法官",
        "成本敏感评估": "使用 Judge 蒸馏小模型"
    }
}
```

## 核心矛盾

**LLM-as-Judge 既是最实用的自动评估方案，也是一个有根本性缺陷的方案。**

```
LLM-as-Judge 的悖论：

如果 LLM 已经能可靠地评判 Agent 的输出质量，
  那为什么不让这个 LLM 直接当 Agent？
  为什么需要一个 Agent + 一个 Judge？

答案：
  "评判能力"和"执行能力"是两种不同的能力。
  一个模型可能执行任务一般，但评判别人做得很好。
  就像好的体育评论员不一定是好的运动员。

但这个答案并不完全令人信服，因为：
  当被评判的任务难度超过法官能力时，
  法官会漏判错误（false negative 不足）。
```

## 代码实现

```python
import json
from typing import List, Optional
from dataclasses import dataclass
from enum import Enum


class ScoreDimension(Enum):
    """常见评分维度"""
    ACCURACY = "accuracy"
    COMPLETENESS = "completeness"
    EFFICIENCY = "efficiency"
    SAFETY = "safety"
    HELPFULNESS = "helpfulness"
    CLARITY = "clarity"


@dataclass
class JudgeConfig:
    """法官配置"""
    model: str                           # 法官模型名
    system_prompt: str                   # 系统提示
    criteria: List[dict]                 # 评分标准
    temperature: float = 0.0             # 法官温度（应该接近 0）
    max_tokens: int = 2048
    num_judges: int = 1                  # 多法官数量
    output_format: str = "json"          # json / text


class LLMJudge:
    """LLM-as-Judge 核心类"""

    def __init__(self, config: JudgeConfig, llm_client):
        self.config = config
        self.llm = llm_client

    def evaluate(self,
                 task: str,
                 agent_output: str,
                 reference: Optional[str] = None,
                 trajectory: Optional[list] = None) -> dict:
        """
        对 Agent 输出进行评审。

        Args:
            task: 任务描述
            agent_output: Agent 的最终输出
            reference: 可选的参考答案
            trajectory: Agent 执行轨迹

        Returns:
            评分结果
        """
        prompt = self._build_prompt(
            task, agent_output, reference, trajectory
        )

        if self.config.num_judges > 1:
            # 多法官投票
            votes = []
            for _ in range(self.config.num_judges):
                response = self._call_judge(prompt)
                parsed = self._parse_response(response)
                votes.append(parsed)

            return self._aggregate_votes(votes)
        else:
            response = self._call_judge(prompt)
            return self._parse_response(response)

    def _build_prompt(self,
                      task: str,
                      output: str,
                      reference: Optional[str],
                      trajectory: Optional[list]) -> str:
        """构造评审 Prompt"""
        sections = [
            {"role": "system", "content": self.config.system_prompt},
            {"role": "user", "content": f"## 任务描述\n\n{task}\n"}
        ]

        if trajectory:
            traj_str = "\n".join(
                f"Step {t.get('step', i)}: {t.get('action', '')}"
                for i, t in enumerate(trajectory)
            )
            sections.append({
                "role": "user",
                "content": f"## Agent 执行轨迹\n\n{traj_str}\n"
            })

        sections.append({
            "role": "user",
            "content": f"## Agent 输出\n\n{output}\n"
        })

        if reference:
            sections.append({
                "role": "user",
                "content": f"## 参考答案\n\n{reference}\n"
            })

        # 评分标准
        criteria_str = "\n".join(
            f"- {c['name']} ({c['range']} 分): {c['description']}"
            for c in self.config.criteria
        )
        sections.append({
            "role": "user",
            "content": f"## 评分标准\n\n{criteria_str}\n\n"
                       f"请以 JSON 格式输出评分结果。"
        })

        return sections

    def _call_judge(self, prompt: list) -> str:
        """调用 LLM 法官"""
        response = self.llm.chat.completions.create(
            model=self.config.model,
            messages=prompt,
            temperature=self.config.temperature,
            max_tokens=self.config.max_tokens,
            response_format={"type": "json_object"}
        )
        return response.choices[0].message.content

    def _parse_response(self, response: str) -> dict:
        """解析法官输出"""
        try:
            data = json.loads(response)
            return {
                "scores": data.get("scores", {}),
                "total_score": data.get("total_score", 0),
                "rationale": data.get("rationale", ""),
                "pass_fail": data.get("verdict", "UNKNOWN")
            }
        except (json.JSONDecodeError, KeyError) as e:
            return {
                "scores": {},
                "total_score": 0,
                "rationale": f"Parse error: {e}",
                "error": True
            }

    def _aggregate_votes(self, votes: List[dict]) -> dict:
        """聚合多法官投票结果"""
        if not votes:
            return {"error": "No votes"}

        # 对每个维度取平均
        all_scores = [v.get("scores", {}) for v in votes if "scores" in v]
        if not all_scores:
            return {"error": "No valid votes"}

        avg_scores = {}
        for dim in all_scores[0]:
            values = [s.get(dim, 0) for s in all_scores]
            avg_scores[dim] = sum(values) / len(values)

        # 一致性计算
        std_devs = {}
        for dim in avg_scores:
            values = [s.get(dim, 0) for s in all_scores]
            mean = avg_scores[dim]
            variance = sum((v - mean) ** 2 for v in values) / len(values)
            std_devs[dim] = variance ** 0.5

        # Pass/Fail 多数决
        pass_votes = sum(
            1 for v in votes if v.get("pass_fail") == "PASS"
        )
        fail_votes = sum(
            1 for v in votes if v.get("pass_fail") == "FAIL"
        )

        return {
            "average_scores": avg_scores,
            "std_devs": std_devs,
            "num_judges": len(votes),
            "agreement": 1 - (sum(std_devs.values()) / len(std_devs)),
            "majority_verdict": "PASS" if pass_votes > fail_votes else "FAIL",
            "pass_ratio": pass_votes / len(votes),
            "individual_votes": votes
        }
```

## 偏差与缓解

### 主要偏差类型

```python
class JudgeBiasDetector:
    """检测和量化法官偏差"""

    def __init__(self, judge: LLMJudge):
        self.judge = judge
        self.audit_log = []

    def detect_position_bias(self,
                              output_a: str,
                              output_b: str,
                              task: str) -> dict:
        """
        检测位置偏差：交换 A/B 位置看评分是否一致。
        如果 [A=4, B=3] 但是 [A=3, B=4]（交换后），
        说明存在位置偏差。
        """
        # 正向
        forward = self.judge.evaluate(
            task, output_a,
            reference=output_b
        )
        # 反向（交换 A/B）
        backward = self.judge.evaluate(
            task, output_b,
            reference=output_a
        )

        bias_detected = (
            abs(forward.get("total_score", 0)
                - backward.get("total_score", 0)) > 0.5
        )

        return {
            "bias_detected": bias_detected,
            "forward_score": forward.get("total_score"),
            "backward_score": backward.get("total_score"),
            "recommendation": (
                "需要交换位置取平均"
                if bias_detected else "无显著位置偏差"
            )
        }

    def detect_length_bias(self, outputs: List[str],
                            task: str) -> dict:
        """
        检测长度偏差：长输出是否总是比短输出得分高。
        随机抽取长/短输出对，比较评分差异。
        """
        import random

        scores_with_length = []
        for output in outputs:
            result = self.judge.evaluate(task, output)
            scores_with_length.append({
                "length": len(output),
                "score": result.get("total_score", 0)
            })

        # 分析长度和评分的相关性
        lengths = [s["length"] for s in scores_with_length]
        scores = [s["score"] for s in scores_with_length]

        if len(lengths) < 3:
            return {"insufficient_data": True}

        # 简单线性相关
        n = len(lengths)
        mean_l = sum(lengths) / n
        mean_s = sum(scores) / n

        numerator = sum((l - mean_l) * (s - mean_s)
                        for l, s in zip(lengths, scores))
        denom_l = sum((l - mean_l) ** 2 for l in lengths) ** 0.5
        denom_s = sum((s - mean_s) ** 2 for s in scores) ** 0.5

        correlation = numerator / (denom_l * denom_s) if denom_l * denom_s > 0 else 0

        return {
            "length_score_correlation": round(correlation, 3),
            "bias_detected": abs(correlation) > 0.5,
            "interpretation": (
                "输出长度对评分有显著影响"
                if abs(correlation) > 0.5
                else "无显著长度偏差"
            )
        }
```

### 缓解策略汇总

| 偏差类型 | 缓解方法 | 有效性 |
|---------|---------|--------|
| 位置偏差 (Position Bias) | 交换 A/B 取平均、顺序随机化 | ★★★★ |
| 长度偏差 (Length Bias) | 长度校准、控制输出长度 | ★★★ |
| 自增强偏差 (Self-enhancement) | 使用不同模型家族、隐藏模型身份 | ★★★★★ |
| 严厉/宽松 (Severity) | 参考锚点校准评分 | ★★★★ |
| 格式偏差 (Format Bias) | 分项评分 + Rubric | ★★★★ |
| 复制偏差 (Verbatim Copying) | 检测和标记复制文本 | ★★★ |

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 对开放式 Agent 输出给出合理的质量评分 | 替代人类在所有场景上的判断（尤其需要领域专业知识的场景） |
| 在大多数场景下接近人工评审水平 | 完美消除所有偏差（偏差只能减轻不能消除） |
| 大规模可扩展、低成本 | 评判超出法官模型自身能力的复杂任务 |
| 提供评分理由（可解释性） | 在法官和被评估者能力差距很小时保持可靠性 |
| 多维度评估 Agent 能力 | 完全标准化——不同法官、不同 Prompt 会导致结果差异 |
| 检测常见的 Agent 错误模式 | 独立于 Prompt 设计质量——评审 Prompt 本身需要精心设计 |

## 工程优化方向

1. **法官模型定期校准**：使用一组已知质量的校准样本（Golden Set），每次评估前先验证法官的评分是否与预期一致，如漂移则调整 Prompt 或切换模型

2. **多法官并行投票**：使用 3-5 个不同模型家族的法官独立评分，聚合时同时考虑平均值和一致性（标准差低表示可信，高表示需要人工复核）

3. **自适应评估策略**：根据 Agent 回答的置信度动态选择评估方案——低置信度问题用重方案（多法官+严格 Rubric），高置信度用轻方案（单法官）

4. **评审质量监控**：分析评分分布如有异常（所有输出评分集中在 4-5 分或 1-2 分），触发告警并重新校准

5. **成本分层缓存**：对未发生变化的 Agent 输出缓存评分结果。评估同一个 Agent 的不同版本时只重新评估输出发生变化的测试用例

6. **对抗性法官验证**：使用已知有缺陷的 Agent 输出测试法官是否能正确识别，定期构造"陷阱"样本验证法官判别能力
