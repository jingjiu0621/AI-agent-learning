# 7.1.6 Decomposition Eval — 分解质量评估

## 简单介绍

分解质量评估（Decomposition Evaluation）是对任务分解结果进行系统性评判的过程。不是所有分解都是好分解——遗漏了关键步骤、步骤粒度不一致、依赖关系标注错误等都是常见的分解质量问题。只有建立了评估体系，才能持续改进 Agent 的规划质量。

## 基本原理

评估任务分解质量，需要从多个维度进行：

1. **完备性**（Completeness）：所有必要的子任务是否都被覆盖
2. **正交性**（Orthogonality）：子任务之间是否有不必要的重叠
3. **可执行性**（Executability）：每个子任务是否足够具体，可以被 Agent 一步或多步执行
4. **顺序合理性**（Ordering）：步骤顺序是否合理，依赖关系是否正确
5. **粒度一致性**（Granularity）：各分支的分解深度是否大致均衡

## 背景

在软件工程中，模块划分质量有成熟的耦合度（Coupling）和内聚度（Cohesion）评估体系。在项目管理中，WBS（Work Breakdown Structure）有 100% 规则和"8/80 小时"粒度指南。Agent 的任务分解评估借鉴了这些思想，但需要适应 LLM 生成分解的独特特点——分解的逻辑合理性比严格的数学形式更重要。

## 之前针对这个问题的做法与结果

1. **人工评审**：由人检查分解结果。结果：最可靠，但无法规模化。
2. **执行成功率代理**：用最终任务执行成功率来反推分解质量。结果：粗粒度有效，但无法区分是分解问题还是执行问题。
3. **LLM 自我评估**：让 LLM 给自己生成的分解打分。结果：有参考价值，但 LLM 作为自评者有明显偏差，倾向于高估自己的分解质量。

## 核心矛盾

**评估成本 vs 评估价值**：全面评估一次分解的质量需要多次 LLM 调用或人工介入，成本不低。但如果不评估，就无法发现分解质量的系统性缺陷。关键在于找到"足够好"的评估精度，控制评估成本在被评估任务成本的 10% 以内。

## 当前主流优化方向

1. **多维评分卡**：用多个 LLM 评委（Multi-Judge）分别评估不同维度，取各维度的分数汇聚为综合评分。每个 LLM 评委只专注于一个维度，减少"一评多"的认知偏差。
2. **检查列表驱动**：设计一套分解质量检查清单（Checklist），用 LLM 逐项检查。清单项包括"是否存在缺失步骤"、"步骤边界是否清晰"、"是否有冗余步骤"等。
3. **模拟执行验证**：让 LLM 模拟执行分解后的步骤序列，在"思维中运行"以发现潜在问题。开销较大但能发现深层逻辑缺陷。
4. **对比基线**：将有经验的 Agent 生成的分解与基线（如简单 CoT 生成的步骤列表）对比，评估改进程度。

## 实现的最大挑战

```python
class DecompositionEvaluator:
    DIMENSIONS = [
        "completeness",    # 是否覆盖了所有必要步骤
        "orthogonality",   # 步骤之间是否有重复
        "executability",   # 每个步骤是否可执行
        "ordering",        # 顺序是否合理
        "granularity",     # 粒度是否一致
    ]
    
    def evaluate(self, goal: str, decomposition: list[TaskNode]) -> EvalReport:
        scores = {}
        details = {}
        
        for dim in self.DIMENSIONS:
            score, detail = self._judge_dimension(dim, goal, decomposition)
            scores[dim] = score
            details[dim] = detail
        
        overall = self._aggregate(scores)
        
        return EvalReport(
            overall_score=overall,
            dimension_scores=scores,
            issues=details["critical_issues"],
            suggestions=details["improvement_suggestions"]
        )
    
    def _judge_dimension(self, dimension: str, goal: str, decomposition) -> tuple[float, str]:
        """让 LLM 评委评估单个维度"""
        prompt = f"""
        你是一个任务分解质量评估专家。请评估以下分解在"{dimension}"维度的表现。
        评分标准: 0-100 分，60 分及格，80 分良好，90 分优秀。
        
        原始目标: {goal}
        分解结果:
        {self._format_decomposition(decomposition)}
        
        请输出 JSON:
        {{
            "score": float,
            "strengths": [str],
            "weaknesses": [str],
            "critical_issues": [str],
            "improvement_suggestions": [str]
        }}
        """
        result = self.llm.predict(prompt)
        return json.loads(result)
    
    def _aggregate(self, scores: dict) -> float:
        """综合评分（可加权）"""
        weights = {
            "completeness": 0.30,
            "orthogonality": 0.15,
            "executability": 0.25,
            "ordering": 0.20,
            "granularity": 0.10,
        }
        return sum(scores[dim] * weights[dim] for dim in scores)
```

最大挑战是**评估标准的主观性**：不同评估者对"好的分解"有不同的理解，同一份分解在不同评委眼中评分可能差异很大。解决方案是使用多个评委取平均，明确定义每个评分维度的具体标准（Rubric），以及定期用"标定数据"（标准答案）检测评估器的一致性。

## 能力边界和结果边界

- **能力**：可以检测出 70-80% 的明显分解缺陷（遗漏步骤、顺序错误、粒度不一致）
- **边界**：无法准确评估分解的"最优性"——是否存在更好的分解方式；对高度领域特定的任务，缺乏领域知识的评估者可能错过深层问题
- **结果**：定期评估+改进的分解质量，相比不评估的基线，任务端到端成功率可提升 15-30%

## 核心优势

建立量化评估体系是将 Agent 规划能力从"凭感觉"推进到"可衡量、可优化"的关键基础设施。只有量化了分解质量，才能比较不同策略、不同 prompt、不同 LLM 的规划效果。

## 工程优化方向

- 自动化评估管道：在 CI/CD 中加入分解质量回归测试，防止新改动降低分解质量
- 评估结果驱动改进：将评估中发现的常见缺陷模式（如"总是遗漏数据验证步骤"）反馈到分解 prompt 中
- 让评估本身更便宜：训练一个小模型专门做分解质量评分，替代每次调用大模型评估
- 建立"黄金分解"数据集：对典型任务人工标注最优分解，作为评估的 ground truth

## 适合场景

- 生产环境中需要持续监控 Agent 规划质量的系统
- 在多种分解策略之间做 A/B 测试时的评价标准
- Agent 自我改进循环中的反馈信号来源
- 对比不同 LLM 或不同 Prompt 的规划能力
