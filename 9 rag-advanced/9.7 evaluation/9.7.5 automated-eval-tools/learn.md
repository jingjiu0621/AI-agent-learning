# 9.7.5 automated-eval-tools — 自动评估工具（RAGAS / TruLens / DeepEval）

## 简单介绍

自动评估工具（Automated Evaluation Tools）是**将 RAG 评估指标实现为可复用工具**的软件库。它们提供标准化的指标计算、评估管线编排、结果可视化等功能，让开发者不需要从零实现评估逻辑。

## 三大主流工具对比

| 维度 | **RAGAS** | **TruLens** | **DeepEval** |
|------|-----------|-------------|-------------|
| 发布年份 | 2023 | 2023 | 2024 |
| 核心指标 | Faithfulness, Relevance, Precision, Recall | Groundedness, Relevance, QA Correctness | Faithfulness, Hallucination, Toxicity, RAGAS 兼容 |
| 评估模式 | 离线评估 | 在线 + 离线 | 离线 + CI 集成 |
| 数据存储 | 内存 | 数据库（SQLite/PostgreSQL） | 内存 + 可选日志 |
| LLM 依赖 | 必须 LLM-as-Judge | 必须 LLM-as-Judge | 可选（支持非 LLM 指标） |
| 自定义指标 | 中等 | 高（反馈函数） | 高 |
| 排错功能 | 中 | 高（App 端到端追踪） | 中 |
| 中文支持 | 好（依赖底层 LLM） | 好 | 好 |

## RAGAS 详解

### 核心指标

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)

# 准备数据
dataset = {
    "question": ["What is transformer architecture?", "Explain RAG"],
    "answer": ["Transformer uses self-attention...", "RAG is..."],
    "contexts": [
        ["Transformer is a neural network architecture..."],
        ["Retrieval-Augmented Generation combines..."]
    ],
    "ground_truth": [
        "Transformer is a neural network architecture...",
        "RAG combines retrieval with generation..."
    ]
}

# 运行评估
result = evaluate(
    dataset=dataset,
    metrics=[
        faithfulness,           # 忠实度：回答是否被上下文支持
        answer_relevancy,       # 相关性：回答是否针对问题
        context_precision,      # 上下文精度：检索结果的相关性
        context_recall,         # 上下文召回：检索结果的覆盖度
    ]
)

print(result)
# {'faithfulness': 0.95, 'answer_relevancy': 0.88, ...}
```

### RAGAS 的高级用法：可分解评估

```python
from ragas.metrics import FaithfulnesswithHHEM

# RAGAS 的 Faithfulness 评估过程（可分解）
faithfulness_scorer = FaithfulnesswithHHEM()

# Step 1: 将回答拆分为声明（claims）
claims = faithfulness_scorer._generate_claims(answer, question)
# ["Transformer uses self-attention", 
#  "Transformer was introduced in 2017"]

# Step 2: 验证每个声明
verdicts = faithfulness_scorer._validate_claims(claims, contexts)
# [True, True]

# Step 3: 计算忠实度
score = sum(verdicts) / len(verdicts)
```

## TruLens 详解

### 反馈函数（Feedback Functions）

```python
from trulens_eval import Feedback, TruLens
from trulens_eval.feedback.provider.openai import OpenAI

# 初始化评估提供者
provider = OpenAI()

# 定义三个核心反馈函数
f_groundedness = (
    Feedback(provider.groundedness_measure_with_cot_reasons)
    .on(TruLens.select_context())      # 输入：检索上下文
    .on_output()                        # 输入：生成回答
)

f_answer_relevance = (
    Feedback(provider.relevance_with_cot_reasons)
    .on_input()                         # 输入：用户问题
    .on_output()                        # 输入：生成回答
)

f_context_relevance = (
    Feedback(provider.context_relevance_with_cot_reasons)
    .on_input()                         # 输入：用户问题
    .on(TruLens.select_context())       # 输入：检索上下文
)

# 组装评估器
tru_recorder = TruLens(
    app_id='RAG_v1',
    feedbacks=[
        f_groundedness,
        f_answer_relevance,
        f_context_relevance
    ]
)

# 使用装饰器追踪 RAG 调用
@tru_recorder.record
def rag_query(question: str) -> dict:
    contexts = retriever.retrieve(question)
    answer = llm.generate(question, contexts)
    return {
        'answer': answer,
        'contexts': [c.text for c in contexts]
    }
```

### TruLens Dashboard

TruLens 提供内置的可视化仪表盘，可以：

1. **追踪每次 RAG 调用的评估分数**
2. **对比不同版本 RAG 系统的表现**
3. **下钻分析具体失败的案例**
4. **设置告警阈值**（当 Faithfulness < 0.7 时告警）

```python
# 启动仪表盘
from trulens_eval import Tru
Tru().run_dashboard()

# Dashboard 地址: http://localhost:8501
```

## DeepEval 详解

### 集成于测试框架

```python
from deepeval import assert_test
from deepeval.metrics import (
    FaithfulnessMetric,
    HallucinationMetric,
    AnswerRelevancyMetric,
    ContextualPrecisionMetric,
)
from deepeval.test_case import LLMTestCase

# 定义测试用例
test_case = LLMTestCase(
    input="What is RAG?",
    actual_output="RAG combines retrieval with generation...",
    retrieval_context=["Retrieval-Augmented Generation is..."],
    expected_output="A technique that combines retrieval and generation"
)

# 定义指标
faithfulness = FaithfulnessMetric(threshold=0.7)

# 运行测试
def test_rag_faithfulness():
    assert_test(test_case, [faithfulness])

# 在 CI 中运行
# pytest test_rag.py
```

### 批量评估

```python
from deepeval import evaluate
from deepeval.dataset import EvaluationDataset

# 构建评估数据集
dataset = EvaluationDataset()
for sample in eval_samples:
    dataset.add_test_case(
        LLMTestCase(
            input=sample.query,
            actual_output=sample.answer,
            expected_output=sample.ground_truth,
            retrieval_context=sample.contexts,
        )
    )

# 批量运行评估
results = evaluate(
    test_cases=dataset.test_cases,
    metrics=[
        FaithfulnessMetric(),
        HallucinationMetric(),
        AnswerRelevancyMetric(),
        ContextualPrecisionMetric(),
        ContextualRecallMetric(),
    ]
)

# 生成报告
results.print_report()
# 导出
results.to_csv("eval_results.csv")
```

## 工具选型决策树

```
你的需求是什么？
    │
    ├── 快速原型验证 → RAGAS（最简 API）
    │
    ├── 生产环境监控 → TruLens（Dashboard + 持续追踪）
    │
    ├── CI/CD 集成 → DeepEval（pytest 原生支持）
    │
    ├── 需要自定义复杂逻辑 → TruLens（反馈函数灵活）
    │
    ├── 中文场景 → 都可以（依赖底层评估 LLM）
    │
    └── 零外部依赖 → 手动实现（不推荐）
```

## 工具的局限性

1. **所有工具都依赖 LLM-as-Judge**：评估的准确性受底层 Judge LLM 限制
2. **Judge LLM 偏见**：GPT-4 做 Judge 时会偏向 GPT-4 生成的回答
3. **成本叠加**：评估一次 RAG 调用可能需要 2-3 次额外 LLM 调用
4. **中文支持上限**：工具本身支持中文，但评估精度受限于中文 LLM 的判断能力

## 推荐的评估架构

```
                    ┌──────────────────────────┐
                    │    评估 Pipeline          │
                    │                          │
                    │  ┌──────────────────┐    │
                    │  │ Fast Check       │    │
                    │  │ ROUGE/BLEU/METEOR │    │ ← 每次部署前
                    │  │ <30 秒            │    │
                    │  └────────┬─────────┘    │
                    │           ▼              │
                    │  ┌──────────────────┐    │
                    │  │ Standard Eval    │    │
                    │  │ RAGAS + DeepEval │    │ ← 每次变更
                    │  │ <5 分钟           │    │
                    │  └────────┬─────────┘    │
                    │           ▼              │
                    │  ┌──────────────────┐    │
                    │  │ Deep Analysis    │    │
                    │  │ TruLens + Human  │    │ ← 每周/版本
                    │  │ 20-30 分钟        │    │
                    │  └──────────────────┘    │
                    └──────────────────────────┘
```

## 推荐工具

- **RAGAS**: https://github.com/explodinggradients/ragas
- **TruLens**: https://github.com/truera/trulens
- **DeepEval**: https://github.com/confident-ai/deepeval
- **LangSmith**: 商业版，提供评估和追踪一体
