# 9.7.6 human-eval-framework — 人工评估框架

## 简单介绍

人工评估框架（Human Evaluation Framework）是指**由人类评审者对 RAG 系统的输出进行质量评估**的方法论和工具体系。虽然自动评估工具（RAGAS、TruLens）效率高，但**人工评估仍然是评估的"金标准"**——特别是涉及事实准确性、回答质量和用户体验时，机器无法完全替代人的判断。

## 为什么人工评估不可替代

| 评估维度 | 自动评估 | 人工评估 | 说明 |
|---------|---------|---------|------|
| **事实准确性** | ✅ 可检测明显矛盾 | ✅ 可检测细微错误 | LLM 无法识别自身知识盲区 |
| **用户满意度** | ❌ 无法判断 | ✅ 真实感受 | 用户"觉得好用"无法被量化 |
| **创造性质量** | ❌ 无法评估 | ✅ 审美判断 | 创意写作、回答风格等 |
| **文化适当性** | ❌ 难以判断 | ✅ 敏感度高 | 特定文化背景的用语得体 |
| **覆盖完整性** | ✅ 可判断是否有遗漏 | ✅ 更细致 | 专家能发现"缺了重要部分" |
| **时效一致性** | ❌ 难以判断 | ✅ 上下文连贯 | 多轮对话的连贯性 |

## 人工评估的两种模式

### 1. 众包评估（Crowd-sourced Evaluation）

适用于大规模评估，但评审者不是领域专家：

```python
@dataclass
class CrowdEvalTask:
    query: str
    answer: str
    contexts: list[str]
    
    # 评估维度（评分 1-5）
    dimensions: dict = field(default_factory=lambda: {
        "relevance": "回答是否相关？",
        "completeness": "回答是否完整？",
        "clarity": "回答是否清晰易懂？",
        "helpfulness": "回答是否有帮助？",
    })
    
    # 指导说明
    instructions: str = """
    请根据以下维度对回答进行评分（1-5分）：
    1 = 很差, 2 = 较差, 3 = 一般, 4 = 较好, 5 = 很好
    """
```

**评估指标**：
- **Cohen's Kappa**: 评审者间一致性（>0.6 表示可接受）
- **Fleiss' Kappa**: 多评审者一致性

### 2. 专家评估（Expert Evaluation）

适用于需要领域知识的评估：

```python
@dataclass
class ExpertEvalTask:
    query: str
    answer: str
    contexts: list[str]
    domain: str
    
    # 专家才具备的能力
    expert_only_dimensions: dict = field(default_factory=lambda: {
        "factual_accuracy": "有没有事实性错误？",
        "domain_appropriateness": "是否符合领域惯例？",
        "source_fidelity": "是否准确反映了检索源的信息？",
        "missing_critical_info": "是否遗漏了关键信息？",
    })
    
    # 专家背景要求
    required_expertise: list[str] = field(default_factory=list)
```

## 人工评估的完整流程

### 阶段 1：设计评估方案

```python
class HumanEvalFramework:
    """人工评估框架"""
    
    def __init__(self, eval_name: str):
        self.eval_name = eval_name
        self.tasks: list[EvalTask] = []
        self.reviewers: list[Reviewer] = []
        self.results: list[EvalResult] = []
    
    def design_evaluation(
        self,
        queries: list[str],
        outputs: list[str],
        contexts: list[list[str]],
        eval_type: str = "expert",  # "crowd" or "expert"
        dimensions: list[str] = None,
    ):
        """设计评估任务"""
        for i, (q, a, c) in enumerate(zip(queries, outputs, contexts)):
            self.tasks.append(EvalTask(
                id=i,
                query=q,
                answer=a,
                contexts=c,
                dimensions=dimensions or DEFAULT_DIMENSIONS,
                eval_type=eval_type,
            ))
```

### 阶段 2：评审者选择与培训

```python
def select_and_train_reviewers(
    self, 
    candidates: list[ReviewerCandidate],
    eval_type: str,
) -> list[Reviewer]:
    """筛选和培训评审者"""
    
    selected = []
    
    for candidate in candidates:
        # 一致性测试
        test_agreement = self._consistency_test(candidate)
        if test_agreement < 0.6:
            continue  # 一致性不足，淘汰
        
        # 校准培训
        self._calibration_training(candidate, eval_type)
        
        selected.append(Reviewer(
            id=candidate.id,
            name=candidate.name,
            expertise=candidate.expertise,
            eval_type=eval_type,
        ))
    
    return selected

def _calibration_training(self, reviewer, eval_type: str):
    """校准培训：让评审者理解评分标准"""
    
    calibration_samples = [
        # 每个样本包含：查询、回答、参考评分（gold standard）
        CalibrationSample(
            query="What is RAG?",
            answer="RAG is a technique...",
            gold_scores={"relevance": 5, "completeness": 4}
        ),
        # ... 更多校准样本
    ]
    
    for sample in calibration_samples:
        reviewer_score = reviewer.rate(sample)
        # 反馈差异
        feedback = self._calculate_feedback(reviewer_score, sample.gold_scores)
        reviewer.receive_feedback(feedback)
```

### 阶段 3：执行评估

```python
def execute_evaluation(self) -> list[EvalResult]:
    """执行人工评估"""
    
    # 分配任务（每个任务至少 3 人评估）
    assignment = self._assign_tasks(min_reviewers_per_task=3)
    
    results = []
    for task, reviewers in assignment.items():
        task_results = []
        for reviewer in reviewers:
            score = reviewer.rate(task)
            task_results.append(EvalResult(
                task_id=task.id,
                reviewer_id=reviewer.id,
                scores=score.scores,
                comments=score.comments,
            ))
        
        # 计算评审者间一致性
        agreement = self._calculate_inter_rater_agreement(task_results)
        
        # 如果一致性太低（<0.5），需要复议
        if agreement < 0.5:
            task_results = self._adjudication_round(task_results)
        
        results.extend(task_results)
    
    return results
```

### 阶段 4：结果分析与报告

```python
def analyze_results(self, results: list[EvalResult]) -> EvalReport:
    """分析人工评估结果"""
    
    # 聚合评分
    dimension_scores = defaultdict(list)
    for r in results:
        for dim, score in r.scores.items():
            dimension_scores[dim].append(score)
    
    # 计算统计指标
    report = EvalReport(
        name=self.eval_name,
        overall_score=np.mean([np.mean(v) for v in dimension_scores.values()]),
        dimension_scores={
            dim: {
                'mean': np.mean(scores),
                'std': np.std(scores),
                'median': np.median(scores),
                'p25': np.percentile(scores, 25),
                'p75': np.percentile(scores, 75),
            }
            for dim, scores in dimension_scores.items()
        },
        inter_rater_agreement=self._calculate_overall_agreement(results),
        sample_size=len(set(r.task_id for r in results)),
        reviewer_count=len(set(r.reviewer_id for r in results)),
        # 定性分析
        common_issues=self._extract_common_issues(results),
        outlier_cases=self._find_outliers(results),
    )
    
    return report
```

## 评估设计中的关键决策

### 评分量表选择

| 量表类型 | 示例 | 优点 | 缺点 |
|---------|------|------|------|
| Likert 5 点 | 1-5 分 | 简单，广泛使用 | 评分者间差异大 |
| Likert 7 点 | 1-7 分 | 更精细 | "中间"选项倾向 |
| 连续量表 | 0-100 | 统计灵活 | 评分者不一致 |
| 排名 | A > B > C | 避免绝对标准问题 | 只能做对比 |

### 评估维度数量

- **3-5 个维度**：最佳实践。太多维度会增加评审者疲劳
- **每个维度要有明确的描述和示例**：
  ```
  好: 忠实度(1-5): 回答中的每个声明是否都被检索文档支持？
  差: 质量(1-5): 回答质量如何？
  ```

## 人工评估 vs 自动评估的互补

```
    高成本                     人工评估（金标准）
    高精度                     │
                              │   * 关键场景
                              │   * 发布前验收
                              │   * 新场景发现
                              │
    低成本            自动评估（快速反馈）
    低精度             │
                      │   * 日常开发迭代
                      │   * CI/CD 集成
                      │   * 回归检测
                      │
                      效率
                      高
```

## 最大挑战

1. **评审者间一致性**：不同人对"好回答"有不同标准，需要持续校准
2. **评审者疲劳**：每轮评估 50+ 样本后，评分质量明显下降
3. **评估成本**：专家评估每小时 $50-100，大规模评估成本极高
4. **评估偏差**：首因效应、近因效应、光环效应等心理学偏差
5. **规模限制**：人工评估通常只能覆盖几百个样本（自动化可覆盖数万）

## 能力边界

- ✅ 发现自动评估无法发现的细粒度质量问题
- ✅ 为自动评估提供"金标准"训练数据
- ✅ 验证自动评估指标与真实用户体验的一致性
- ❌ 无法规模化（成本线性增长）
- ❌ 无法完全消除主观偏差
- ❌ 领域专家评估者难以招募

## 工程建议

1. **建立黄金标准集**：对 200-500 个代表性样本进行人工评估，作为标准化参考
2. **定期验证自动评估**：每季度用人工评估校验自动评估的一致性
3. **设计校准流程**：每次评估前用标准样本校准评审者
4. **自动评估+人工抽检**：用自动评估覆盖全部，人工重点抽检低置信度和边缘 case
5. **打分与评论结合**：评分+文字评论的组合比纯评分提供更多信息

## 推荐工具

- **Label Studio**: 开源标注工具，支持 RAG 评估任务配置
- **Argilla**: 面向 NLP 的人工反馈平台
- **Scale AI / Surge AI**: 专业人工评估服务提供商
- **LangSmith 人工评估**: LangSmith 提供的标注界面
- **自定义 Web 界面**: 对于特定需求，使用 Streamlit/Gradio 快速搭建
