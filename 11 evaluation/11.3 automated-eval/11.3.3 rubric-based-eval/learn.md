# 11.3.3 rubric-based-eval — 基于评分标准的结构化评估

## 简单介绍

Rubric-based Evaluation（基于评分标准的评估）是 LLM-as-Judge 的最重要演进方向。它不是让 LLM 自由评分，而是通过**预先定义好的结构化评分标准**来引导评判过程。

```
自由评分:
  "这段代码质量怎么样？给个分吧。"
  → LLM 凭感觉给分，标准模糊

Rubric 评分:
  "从以下 4 个维度评分，每个维度 3 个级别，每个级别有详细描述……"
  → LLM 按标准逐项评判，结果更可靠
```

核心区别在于：**Rubric 将"好"和"差"的具体标准明确定义出来，让法官和被评估者都对评判尺度有共同理解。**

2025-2026 年的研究一致表明，Rubric 引导的评估比自由提示的 LLM 评判方差降低 30-50%（IEEE 2025, arXiv 2605.30568），已成为自动评估的主流实践。

## 基本原理

### Rubric 的三要素

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Rubric 结构                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  维度 (Dimension)             定义了评估的方面                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 例：准确性 / 完整性 / 效率 / 安全性                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              ↓                                       │
│  级别 (Level)                定义了评分刻度                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 例：1-差 / 2-及格 / 3-良好 / 4-优秀 / 5-卓越                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              ↓                                       │
│  描述 (Descriptor)           每个级别在每个维度上的具体描述           │
│  └─────────────────────────────────────────────────────────────┘    │
│  例：准确性-5 分："输出完全正确，没有事实错误"                      │
│      完整性-3 分："覆盖了大部分要点，但有 1-2 个遗漏"               │
│      效率-1 分："使用了过多步骤，有明显冗余"                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 为什么 Rubric 有效？

```
自由评分的问题：
  "好"是什么？
  法官 A: 好 = 准确
  法官 B: 好 = 完整 + 清晰
  法官 C: 好 = 听起来聪明
  → 三个法官评的不是同一件事

Rubric 带来的改进：
  "好"被分解为具体维度
  "准确"在 Rubric 中被明确定义了
  → 所有法官按同一标准评分
  → 一致性大幅提升
```

### 解析型 Rubric vs 整体型 Rubric

```python
# 1. 解析型 Rubric (Analytic Rubric)
# 每个维度独立评分，最后加权汇总
ANALYTIC_RUBRIC = {
    "name": "Agent 任务完成质量评估",
    "dimensions": [
        {
            "name": "准确性 (Accuracy)",
            "weight": 0.35,
            "levels": [
                {"score": 5, "desc": "输出完全正确，无任何事实或逻辑错误"},
                {"score": 4, "desc": "输出基本正确，有微小可忽略的误差"},
                {"score": 3, "desc": "输出部分正确，有 1-2 处明显错误但不影响整体"},
                {"score": 2, "desc": "输出有多处错误，核心内容不准确"},
                {"score": 1, "desc": "输出完全错误或无关"}
            ]
        },
        {
            "name": "完整性 (Completeness)",
            "weight": 0.25,
            "levels": [
                {"score": 5, "desc": "覆盖所有必要内容，没有遗漏任何关键信息"},
                {"score": 4, "desc": "覆盖了几乎所有内容，遗漏不重要的细节"},
                {"score": 3, "desc": "覆盖了大部分内容，有 1-2 个重要的遗漏"},
                {"score": 2, "desc": "遗漏了多个重要内容，回答不完整"},
                {"score": 1, "desc": "完全没有回答任务要求的内容"}
            ]
        },
        {
            "name": "效率 (Efficiency)",
            "weight": 0.15,
            "levels": [
                {"score": 5, "desc": "使用最少的步骤完成任务，无冗余操作"},
                {"score": 4, "desc": "高效的执行，有少量非必要的步骤"},
                {"score": 3, "desc": "可以接受，有 1-2 个明显多余的步骤"},
                {"score": 2, "desc": "效率低，有多步冗余或重复"},
                {"score": 1, "desc": "效率极低，大量无用的步骤"}
            ]
        },
        {
            "name": "安全性 (Safety)",
            "weight": 0.25,
            "levels": [
                {"score": 5, "desc": "完全安全，无任何风险行为"},
                {"score": 4, "desc": "基本安全，有极低风险的表述"},
                {"score": 3, "desc": "存在潜在风险但受到控制"},
                {"score": 2, "desc": "有明显的不安全行为或输出"},
                {"score": 1, "desc": "严重违反安全规范"}
            ]
        }
    ],
    "scoring": {
        "method": "weighted_sum",
        "max_score": 5.0,
        "pass_threshold": 3.5
    }
}


# 2. 整体型 Rubric (Holistic Rubric)
# 只看整体质量，不分维度
HOLISTIC_RUBRIC = {
    "name": "Agent 整体表现评估",
    "levels": [
        {
            "score": 5,
            "desc": "完美完成任务，过程高效，结果准确，无需改进"
        },
        {
            "score": 4,
            "desc": "很好地完成了任务，有小瑕疵但不影响结果"
        },
        {
            "score": 3,
            "desc": "完成了大部分任务，有需要改进的地方但整体可接受"
        },
        {
            "score": 2,
            "desc": "仅部分完成任务，存在重大问题需要修复"
        },
        {
            "score": 1,
            "desc": "未能完成基本要求，需要重大改进"
        }
    ]
}
```

解析型 Rubric 适用于多维度分析场景（告诉我哪里需要改进），整体型 Rubric 适用于快速排名和分类。

## 背景

### Rubric 评估的演进

```
教育测量领域的起源（20 世纪 60 年代）
  Rubric 最初用于教育评分——确保论文评分一致
  定义清晰的评分标准可以减少不同老师之间的评分差异

引入 LLM 评估（2023）
  G-Eval: 第一个系统化的 LLM Rubric 评估框架
  使用 GPT-4 按维度评分，生成评分理由
  效果显著优于直接评分

结构化发展（2024）
  Rubric 模板化：不同任务类型定义不同的标准
  动态 Rubric：根据 Agent 输出自适应调整标准
  权重学习：通过校准数据学习最优维度权重

成熟化（2025-2026）
  Rubric 引导评分成主流共识（IEEE 2025）
  自动 Rubric 生成：LLM 根据任务描述自动创建评分标准
  动态 Rubric 精炼：通过法官反馈循环优化评分标准
  多模态 Rubric：扩展到图像、代码等非文本输出 (ICML 2026)
```

## 核心矛盾

**Rubric 的详细程度与通用性的矛盾。**

```
高度详细的 Rubric：
  评分一致性高 ✅
  ├─ 但适用范围窄——换一个任务就得重新设计
  ├─ 设计成本高——需要领域专家反复调试
  └─ 可能遗漏新出现的问题模式

高度通用的 Rubric：
  适用范围广 ✅
  ├─ 但评分一致性低
  ├─ 模糊的标准让法官回到"凭感觉评分"
  └─ 无法覆盖具体场景的特殊要求

平衡方案：分层 Rubric
  通用层：准确性、完整性、安全性（所有任务通用）
  特定层：针对任务类型的特殊标准（如 SQL 查询需要 "效率"）
  实例层：单个任务的特殊要求（如 "必须提到价格"）
```

## G-Eval 框架

G-Eval 是目前最流行的 Rubric 评估框架，由 NLP 团队提出（2023），其核心流程如下：

```
步骤 1: 定义 Rubric
  ┌──────────────────────────────────────────────────┐
  │ 定义评估维度和每个级别的描述                        │
  │ 维度：准确性                                      │
  │ 5分：输出完全正确...                              │
  │ 4分：基本正确...                                  │
  │ ...                                              │
  └──────────────────────────────────────────────────┘

步骤 2: 构建 Chain-of-Thought Prompt
  ┌──────────────────────────────────────────────────┐
  │ 让 LLM 在评分前先进行分析：                        │
  │ "请先逐步分析 Agent 输出，然后按 Rubric 评分"      │
  │ 分析：Agent 的输出首先...然后...                  │
  │ 评分：准确性=4，完整性=5，...                     │
  └──────────────────────────────────────────────────┘

步骤 3: 概率加权评分
  ┌──────────────────────────────────────────────────┐
  │ 不使用 argmax 选择最可能的分值，                    │
  │ 而是计算每个分数的概率分布，取期望值                 │
  │ P(5) = 0.1, P(4) = 0.7, P(3) = 0.2               │
  │ Expected = 5×0.1 + 4×0.7 + 3×0.2 = 3.9           │
  └──────────────────────────────────────────────────┘

步骤 4: 多维度聚合
  各维度得分 → 按权重加权 → 最终分数
```

```python
class GEvalRubric:
    """G-Eval 风格 Rubric 评估器"""

    def __init__(self, rubric: dict, llm_client):
        self.rubric = rubric
        self.llm = llm_client

    def evaluate(self, task: str, agent_output: str) -> dict:
        """基于 G-Eval 方法进行 Rubric 评估"""

        dimension_scores = {}

        for dim in self.rubric["dimensions"]:
            # Step 1: Chain-of-Thought 分析
            cot_prompt = self._build_cot_prompt(
                dim, task, agent_output
            )
            analysis = self.llm.chat.completions.create(
                messages=[{"role": "user", "content": cot_prompt}],
                temperature=0.0
            )
            analysis_text = analysis.choices[0].message.content

            # Step 2: 概率评分
            prob_prompt = self._build_probability_prompt(
                dim, task, agent_output, analysis_text
            )
            prob_response = self.llm.chat.completions.create(
                messages=[{"role": "user", "content": prob_prompt}],
                temperature=0.0,
                logprobs=True,
                top_logprobs=5  # 获取 top-5 概率
            )

            # Step 3: 计算期望分数
            expected_score = self._compute_expected_score(
                prob_response, dim
            )
            dimension_scores[dim["name"]] = {
                "expected_score": expected_score,
                "analysis": analysis_text
            }

        # Step 4: 加权聚合
        total = 0
        for dim in self.rubric["dimensions"]:
            dim_name = dim["name"]
            score = dimension_scores[dim_name]["expected_score"]
            weight = dim.get("weight", 1.0)
            total += score * weight

        total_weight = sum(
            d.get("weight", 1.0) for d in self.rubric["dimensions"]
        )
        final_score = total / total_weight if total_weight > 0 else 0

        return {
            "dimension_scores": dimension_scores,
            "final_score": round(final_score, 2),
            "passed": final_score >= self.rubric["scoring"]["pass_threshold"]
        }

    def _build_cot_prompt(self, dimension: dict,
                           task: str, output: str) -> str:
        """构造 CoT 分析 Prompt"""
        levels_desc = "\n".join(
            f"  {l['score']}分: {l['desc']}"
            for l in dimension["levels"]
        )
        return f"""分析以下 Agent 输出在"{dimension['name']}"维度上的表现。

## 任务
{task}

## Agent 输出
{output}

## 评分标准
{levels_desc}

请先逐步分析 Agent 的输出，然后给出该维度的评分。
分析要具体，引用输出中的内容作为证据。
"""

    def _compute_expected_score(self,
                                 response, dimension: dict) -> float:
        """基于 logprobs 计算期望分数"""
        # 从 response 中提取各分数的概率
        # 这里简化处理——实际需要使用 logprobs API
        import json
        try:
            content = response.choices[0].message.content
            score_match = json.loads(content)
            # 假设 LLM 输出 {"score": 4, "reasoning": "..."}
            return float(score_match.get("score", 3))
        except (json.JSONDecodeError, KeyError, ValueError):
            # Fallback: 从文本中解析分数
            text = response.choices[0].message.content
            for level in reversed(dimension["levels"]):
                if str(level["score"]) in text:
                    return float(level["score"])
            return 3.0
```

## Rubric 设计的最佳实践

### 1. 使用行为锚定

每个级别的描述应该是**可观察的行为**，而不是模糊的判断：

```
❌ 差的描述：
  "4 分：代码质量好"

✅ 好的描述：
  "4 分：代码结构清晰，有恰当的注释，遵循了语言的最佳实践，
   没有明显的性能问题，变量命名有意义"
```

### 2. 避免中心偏移倾向

Rubric 的级数设计会影响评分的分布：

```
3 级评分 (1/2/3)：
  评分集中在 2（中心偏移）
  → 区分度不足

5 级评分 (1/2/3/4/5)：
  评分分布在 2-4
  → 区分度较好

7 级评分 (1-7)：
  评分分布更广
  → 但评分者一致性可能下降

推荐：Agent 评估使用 4 级评分（去掉中间模糊选项）
  1: 差 / 2: 及格 / 3: 良好 / 4: 优秀
  强迫法官做出清晰区分
```

### 3. 每个维度独立描述

不要在 Rubric 中跨维度共享描述：

```
❌ 混在一起：
  3 分："回答正确, 比较完整"
  → "正确"是准确性，"完整"是完整性，混在一起了

✅ 分开放：
  准确性-3 分："回答内容基本正确，有不超过 1 处小错误"
  完整性-3 分："覆盖了大部分要求，遗漏了次要内容"
```

### 4. 动态生成 Rubric

```python
class DynamicRubricGenerator:
    """自动生成 Rubric（基于任务描述）"""

    def __init__(self, llm_client):
        self.llm = llm_client

    def generate(self, task: str,
                 agent_type: str = "general") -> dict:
        """根据任务描述自动生成评分标准"""
        prompt = f"""你是一个评估专家。请为以下 Agent 任务设计评分标准 (Rubric)。

任务描述: {task}
Agent 类型: {agent_type}

请设计一个 4 维度 × 4 级别的解析型 Rubric，要求：
1. 每个维度有明确的名称和权重（合计 100%）
2. 每个级别有具体的可观察行为描述
3. 维度应该针对这个具体任务定制

输出 JSON 格式（与以下结构相同）：
{{
    "dimensions": [
        {{
            "name": "维度名",
            "weight": 0.25,
            "levels": [
                {{"score": 4, "desc": "具体的优秀表现描述"}},
                {{"score": 3, "desc": "具体的良好表现描述"}},
                {{"score": 2, "desc": "具体的及格表现描述"}},
                {{"score": 1, "desc": "具体的差劲表现描述"}}
            ]
        }}
    ],
    "scoring": {{
        "method": "weighted_sum",
        "max_score": 4.0,
        "pass_threshold": 2.5
    }}
}}
"""
        response = self.llm.chat.completions.create(
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3,
            response_format={"type": "json_object"}
        )

        import json
        rubric = json.loads(response.choices[0].message.content)
        return rubric
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 大幅提升 LLM 评估的一致性和可靠性 | 消除所有法官偏差（只能减少不能消除） |
| 提供多维度的质量分析和改进方向 | 替代人类专家的深度评审 |
| 适应不同类型的任务（通过定制维度） | 自动适配所有可能出现的特殊场景 |
| 让评估结果更具可解释性（每维度得分+理由） | 独立于法官模型的能力——法官太弱，Rubric 再好也没用 |
| 通过权重设计反映不同场景的优先级 | 防止设计不良的 Rubric（设计 Rubric 本身需要技巧） |

## 工程优化方向

1. **Rubric 版本管理**：Rubric 本身需要像代码一样做版本管理——每次修改记录变更，对比不同 Rubric 版本评分结果的一致性

2. **自适应权重学习**：如果有历史评估数据，可以学习不同维度的最优权重。例如：用户投诉最多的维度应该给更高权重

3. **Rubric 退化检测**：定期检查 Rubric 是否仍然有效——如果所有 Agent 都在某个维度得满分，说明该维度可能已经太简单需要更新

4. **多级 Rubric 级联**：第一阶段用简单 Rubric 快速过滤掉明显不合格的输出，第二阶段用详细 Rubric 对候选输出做精细评分

5. **跨任务 Rubric 迁移**：相似类型的任务共享 Rubric 模板，减少重复设计——例如所有"查询类"任务共用一份准确性/完整性/效率 Rubric

6. **交互式 Rubric 调试**：用可视化工具分析 Rubric 评分分布，识别模糊的描述——如果某个级别的输出总是被误分到相邻级别，说明该级描述不够清晰

## 完整示例：一个 Agent 任务 Rubric

```json
{
    "rubric": {
        "name": "Agent 信息检索任务评估",
        "version": "2.1",
        "last_updated": "2026-07-01",
        "description": "用于评估 Agent 在信息检索类任务上的表现",
        "dimensions": [
            {
                "name": "准确性",
                "weight": 0.40,
                "levels": [
                    {"score": 4, "desc": "所有信息完全正确，无事实错误，无幻觉"},
                    {"score": 3, "desc": "信息基本正确，有不超过 1 处次要事实偏差"},
                    {"score": 2, "desc": "有 2-3 处事实错误，但核心信息正确"},
                    {"score": 1, "desc": "存在多处严重事实错误或完全无关的回答"}
                ]
            },
            {
                "name": "相关性",
                "weight": 0.30,
                "levels": [
                    {"score": 4, "desc": "直接回答用户问题，无无关信息，信息高度匹配需求"},
                    {"score": 3, "desc": "回答了核心问题，但有少量不相关的内容"},
                    {"score": 2, "desc": "部分回答了问题，但遗漏了重要方面或包含了较多无关内容"},
                    {"score": 1, "desc": "回答与问题无关或答非所问"}
                ]
            },
            {
                "name": "结构化与清晰度",
                "weight": 0.15,
                "levels": [
                    {"score": 4, "desc": "信息组织清晰，逻辑层次分明，易于阅读和理解"},
                    {"score": 3, "desc": "结构基本合理，但某些部分可以更清晰"},
                    {"score": 2, "desc": "信息组织混乱，需要费力理解"},
                    {"score": 1, "desc": "完全无序，几乎无法理解"}
                ]
            },
            {
                "name": "效率",
                "weight": 0.15,
                "levels": [
                    {"score": 4, "desc": "用最少的步骤和 Token 直接给出答案"},
                    {"score": 3, "desc": "步骤基本合理，有 1 处冗余但不影响"},
                    {"score": 2, "desc": "有多步冗余或大量不必要的 Token 消耗"},
                    {"score": 1, "desc": "极度低效，大量无意义的步骤或输出"}
                ]
            }
        ],
        "scoring": {
            "method": "weighted_sum",
            "max_score": 4.0,
            "pass_threshold": 2.5,
            "excellent_threshold": 3.5
        }
    }
}
```
