# 11.2 benchmarks — 基准测试：标准化评估体系

## 简单介绍

基准测试（Benchmark）是评估体系的核心支柱——它提供了一套标准化的任务集、评估指标和评分协议，使得不同 Agent 系统之间可以进行公平、可复现的性能比较。如果说指标（Metrics）定义了"怎么算好"，那么基准就定义了"在什么任务上算好"。

一个好的基准应该像一把标准的尺子：不管谁来量、什么时候量，结果应该是一致的。但在 Agent 领域，这把尺子本身的设计就是一个巨大挑战——Agent 的任务太多样化、太依赖环境，以至于任何单一基准都只能反映能力的一个侧面。

## 基本原理

### 基准的三要素

```
┌──────────────────────────────────────────────────────────────────┐
│                     Agent 基准测试的三要素                         │
│                                                                   │
│  1. 任务集 (Task Suite)                                           │
│     ┌──────────────────────────────────────────────────────┐     │
│     │  • 一组精心设计的评估任务                              │     │
│     │  • 覆盖不同的难度层级和能力维度                        │     │
│     │  • 有明确的正确答案或评价标准                          │     │
│     └──────────────────────────────────────────────────────┘     │
│                                                                   │
│  2. 评估指标 (Metrics)                                            │
│     ┌──────────────────────────────────────────────────────┐     │
│     │  • 任务完成率、准确率、通过率等量化指标                │     │
│     │  • 可能有严格/宽松两种记分方式                        │     │
│     │  • 支持跨系统/跨模型的横向对比                        │     │
│     └──────────────────────────────────────────────────────┘     │
│                                                                   │
│  3. 评估协议 (Evaluation Protocol)                                │
│     ┌──────────────────────────────────────────────────────┐     │
│     │  • 标准化的执行流程（环境设置、输入输出格式）          │     │
│     │  • 自动化的评分脚本                                  │     │
│     │  • 可复现的运行环境（Docker、版本锁定）               │     │
│     └──────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### 基准评估的一般流程

```
                ┌──────────────┐
                │  加载任务集    │
                │  (JSON/YAML)  │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │  初始化环境    │
                │  (Docker/VM)  │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │  Agent 执行   │
                │  思考→工具→   │
                │  观察→输出    │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │  验证结果      │
                │  自动化评分    │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │  汇总报告      │
                │  分数+轨迹    │
                └──────────────┘
```

## 分类体系

### 通用 Agent 基准

评估 Agent 在广泛任务上的综合能力，不限定特定领域。

| 基准 | 发布方 | 任务数量 | 核心能力 | 难度 |
|------|--------|---------|---------|------|
| GAIA | Meta FAIR | 466 | 多步推理+工具使用+多模态 | 高 |
| AgentBench | 清华/CMU/Oregon | 多种环境 | 多维度 Agent 能力 | 中高 |
| HELM-AE | Stanford CRFM | 多种场景 | 整体 Agent 表现 | 中高 |

### 专用领域基准

针对特定应用场景设计，评估 Agent 在某个垂直领域的能力。

| 基准 | 领域 | 任务数量 | 核心能力 |
|------|------|---------|---------|
| SWE-bench | 软件工程 | 2294 (Full) | 代码修改、Bug 修复 |
| ToolBench | 工具使用 | 超过 3000 | API 调用、工具组合 |
| WebArena | 网页交互 | 812 | 网页导航、表单填写 |
| VisualWebArena | 视觉网页 | 910 | 视觉+网页交互 |

### 对话式基准

评估对话质量和模型的基础交互能力，不强调工具使用。

| 基准 | 类型 | 核心指标 |
|------|------|---------|
| MT-Bench | 多轮对话质量 | LLM-as-Judge 评分 |
| Chatbot Arena | 众包对比 | Elo 分数 |
| AlpacaEval | 单轮指令遵循 | 胜率 vs 参考模型 |

### 分类体系图谱

```
                    Agent 基准测试分类
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
    ┌───┴───┐        ┌─────┴─────┐      ┌────┴────┐
    │ 通用型  │        │  专用型    │      │ 对话型  │
    ├───────┤        ├───────────┤      ├─────────┤
    │ GAIA  │        │ SWE-bench │      │MT-Bench │
    │AgentBench│      │ ToolBench  │      │Arena    │
    │ HELM-AE│        │ WebArena  │      │AlpacaEval│
    └───────┘        └───────────┘      └─────────┘
                           │
                    ┌──────┴──────┐
                    │ 专精能力     │
                    │ 可操作性强  │
                    └─────────────┘
```

## 背景：从 NLP 基准到 Agent 基准

### 经典 NLP 基准

在 Agent 兴起之前，NLP 领域的基准测试主要评估"模型能力"而非"Agent 能力"：

```
2018: GLUE  ─── 9 项 NLU 任务，BERT 在此一举成名
2019: SuperGLUE ─ GLUE 升级版，更难的 8 项任务
2020: CLUE ─── 中文 NLU 基准
2021: BIG-bench ─ 204 项任务，评估 LLM 的广泛能力
```

这些基准的共同特点：**给定输入，模型输出答案，评测一对一的匹配**。模型不需要调用工具、不需要多步推理、不需要与环境交互。

### Agent 基准的革命性变化

Agent 基准和传统 NLP 基准有本质区别：

```
传统基准: 输入 → 模型 → 输出 → 比较答案
              (单步、静态、无环境)

Agent 基准: 输入 → Agent → 思考 → 选工具 → 调用 → 观察
            → 再思考 → ... → 最终输出 → 验证结果
              (多步、动态、强环境依赖)
```

这带来了几个根本性的变化：

1. **从"知道什么"到"能做什么"**: 传统基准测试模型的知识储备，Agent 基准测试模型在真实场景中的行动能力
2. **从单步到多步**: Agent 基准涉及多步推理链，中间步骤的决策质量直接影响最终结果
3. **从静态到动态**: Agent 在运行过程中会收到环境反馈，需要根据新信息调整计划
4. **从封闭到开放**: 很多 Agent 任务没有唯一的正确答案——不同路径可能都算"成功"

## 核心矛盾

**标准化评估 vs 真实场景复杂性**——这是 Agent 基准面临的根本矛盾。

```
标准化评估的要求                      真实场景的特征
─────────────────                    ────────────────
可重复性                            不可重复的环境状态
公平对比                            不同的工具和 API
自动化评分                          模糊的成功标准
题目数量有限                        无限的长尾场景
任务定义明确                        需求不断变化
```

这个矛盾导致了一个普遍现象：**基准分数高不等于实际表现好**。一个在 SWE-bench 上拿到 70% 的 Agent，在真实的软件开发流程中可能表现远不如预期——因为真实 Issue 的上下文更复杂、环境配置更混乱、需求更模糊。

### 基准过拟合

当社区过度关注某个基准时，就会出现针对该基准的"刷分"行为：

```
模型在 benchmark X 上表现好
  → 大家用 benchmark X 衡量
  → 研究者针对 X 优化
  → 分数越来越高
  → 但实际能力提升有限
  → benchmark X 失去区分度
  → 需要新的 benchmark Y
```

这就是所谓的"基准寿命周期"。解决方法是不断更新基准集合，以及使用"隐藏集"（hold-out set）来防止过拟合。

## 基准评估框架 Comparison Table

### 主要 Agent 基准对比

| 基准 | 核心能力 | 任务数 | 主要指标 | 难度 | 是否需要环境 |
|------|---------|--------|---------|------|------------|
| GAIA | 通用推理+工具使用 | 466 (公开165) | 任务完成率 | 高 | 是 (搜索/文件) |
| SWE-bench | 代码修改 | 2294 | % Resolved | 高 | 是 (Docker) |
| ToolBench | API 调用 | >3000 | 通过率 | 中 | 是 (API 模拟) |
| WebArena | 网页交互 | 812 | 任务成功率 | 高 | 是 (自建网站) |
| AgentBench | 多维度 | 8 环境 | 综合评分 | 中高 | 是 (多种) |
| MT-Bench | 对话质量 | 80 (多轮) | LLM 评分 | 中 | 否 |
| Chatbot Arena | 综合对话 | 众包 | Elo 分数 | 中 | 否 |
| BIG-bench | 知识推理 | 204 | 准确率 | 中 | 否 |

### 基线 vs SOTA 参考

| 基准 | 人类基线 | 初始基线 | 当前 SOTA (2026) |
|------|---------|----------|-----------------|
| GAIA | ~92% | GPT-4 ~15% | MiroThinker 82.7% (Val-165) |
| SWE-bench Verified | ~90%+ | GPT-4 ~1.7% | Claude 4.5 Opus 76.80% |
| WebArena | ~78% | GPT-4 ~10% | ~40%+ |
| MT-Bench | -- | GPT-3.5 ~6.5 | GPT-5 / Claude 4 ~9.0+ |

## 基准设计原则

一个好的 Agent 基准应该满足以下五项基本原则：

### 1. 效度 (Validity)

基准是否真正衡量了它声称要衡量的能力？一个标榜"通用 Agent 能力"的基准如果只包含编程题，就是效度不足。

### 2. 信度 (Reliability)

相同条件下重复评估应该得到相同或高度接近的结果。对于 Agent 的非确定性输出，这一点尤其困难——需要多次运行取统计结果。

### 3. 区分度 (Discrimination)

基准能否有效区分不同能力的 Agent？如果一个基准上所有模型都得到 90%+ 或都得到 10%-，说明区分度不足（天花板效应或地板效应）。

```
天花板效应: 所有系统都接近满分，无法区分优劣
  ┌────┬────┬────┬────┬────┐
  │ A  │ B  │ C  │ D  │ E  │
  │ 95%│ 96%│ 94%│ 95%│ 97%│
  └────┴────┴────┴────┴────┘

良好区分度: 分数有合理分布
  ┌────┬────┬────┬────┬────┐
  │ A  │ B  │ C  │ D  │ E  │
  │ 82%│ 71%│ 65%│ 53%│ 41%│
  └────┴────┴────┴────┴────┘

地板效应: 所有系统都接近零分
  ┌────┬────┬────┬────┬────┐
  │ A  │ B  │ C  │ D  │ E  │
  │ 2% │ 3% │ 1% │ 4% │ 2% │
  └────┴────┴────┴────┴────┘
```

### 4. 效率 (Efficiency)

评估开销是否在可接受范围内？SWE-bench 的一次完整评估需要运行数千个 Docker 容器，耗时数小时甚至数天，成本很高。

### 5. 可更新性 (Updateability)

基准能否随技术发展而更新？静态基准很快会被过拟合，需要定期引入新任务。

## 实现挑战

### 1. 数据污染 (Data Contamination)

Agent 底层的 LLM 可能在训练数据中见过基准测试的题目或答案。

```python
class DataContaminationDetector:
    """检测基准数据是否被模型训练数据污染"""

    def __init__(self, model_name: str, cutoff_date: str):
        self.model_name = model_name
        self.cutoff_date = cutoff_date

    def check_contamination(self, benchmark_task: dict) -> dict:
        """检查单个任务是否存在污染风险"""

        risk_factors = []

        # 1. 任务创建时间 vs 模型训练截止时间
        task_create_date = benchmark_task.get("created_date")
        if task_create_date and task_create_date < self.cutoff_date:
            risk_factors.append("task_created_before_cutoff")

        # 2. 任务是否在公开数据集中
        if benchmark_task.get("is_public"):
            risk_factors.append("publicly_available")

        # 3. 任务是否被爬虫收录
        if benchmark_task.get("in_web_crawl"):
            risk_factors.append("in_common_crawl")

        contamination_risk = len(risk_factors) / 3

        return {
            "task_id": benchmark_task["id"],
            "risk_factors": risk_factors,
            "contamination_risk": contamination_risk,
            "is_likely_contaminated": contamination_risk > 0.5
        }
```

### 2. 任务泄露 (Task Leakage)

评测过程中 Agent 可能无意中看到未来的测试任务。

### 3. 基准过拟合 (Benchmark Overfitting)

当社区反复使用同一个基准时，系统会隐性地针对该基准的特征进行优化。

```python
class BenchmarkOverfitDetector:
    """检测是否对基准过拟合"""

    def __init__(self):
        self.scores_by_version = {}

    def add_score(self, version: str, score: float):
        self.scores_by_version[version] = score

    def detect_overfit(self, holdout_scores: dict) -> dict:
        """对比公开基准和隐藏集的分数差异"""

        public_avg = sum(self.scores_by_version.values()) / len(self.scores_by_version)
        holdout_avg = sum(holdout_scores.values()) / len(holdout_scores)
        gap = public_avg - holdout_avg

        return {
            "public_benchmark_avg": round(public_avg, 2),
            "holdout_set_avg": round(holdout_avg, 2),
            "performance_gap": round(gap, 2),
            "overfit_severity": (
                "high" if gap > 0.15
                else "medium" if gap > 0.08
                else "low"
            ),
            "recommendation": (
                "需要更新测试集" if gap > 0.15
                else "正常范围内" if gap < 0.05
                else "建议关注趋势"
            )
        }
```

## Code Example: 基准测试运行框架

```python
import json
import time
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any


@dataclass
class BenchmarkTask:
    """基准测试任务定义"""
    task_id: str
    description: str
    difficulty: str  # easy, medium, hard
    category: str     # reasoning, coding, browsing, etc.
    expected_answer: Any = None
    tools_allowed: list = field(default_factory=list)
    timeout: int = 300


@dataclass
class TaskResult:
    """单个任务的评估结果"""
    task_id: str
    agent_name: str
    passed: bool
    score: float
    steps_taken: int
    execution_time: float
    agent_output: str = ""
    error_message: str = ""


class BaseBenchmark(ABC):
    """基准测试基类——所有基准的通用框架"""

    def __init__(self, name: str, tasks: list):
        self.name = name
        self.tasks = tasks
        self.results = []

    @abstractmethod
    def setup_environment(self, task: BenchmarkTask):
        """设置任务环境（Docker、API 等）"""
        pass

    @abstractmethod
    def run_agent(self, task: BenchmarkTask) -> str:
        """让 Agent 执行任务"""
        pass

    @abstractmethod
    def validate_result(self, task: BenchmarkTask,
                        agent_output: str) -> tuple:
        """验证 Agent 的输出是否正确"""
        pass

    @abstractmethod
    def teardown_environment(self, task: BenchmarkTask):
        """清理环境"""
        pass

    def evaluate_task(self, task: BenchmarkTask) -> TaskResult:
        """执行单个任务的完整评估流程"""

        start_time = time.time()
        steps = 0
        error = None

        try:
            self.setup_environment(task)
            agent_output = self.run_agent(task)
            passed, score = self.validate_result(task, agent_output)
        except Exception as e:
            agent_output = ""
            passed = False
            score = 0.0
            error = str(e)
        finally:
            self.teardown_environment(task)

        execution_time = time.time() - start_time

        return TaskResult(
            task_id=task.task_id,
            agent_name=self.name,
            passed=passed,
            score=score,
            steps_taken=steps,
            execution_time=execution_time,
            agent_output=agent_output,
            error_message=error or ""
        )

    def run_full_evaluation(self, num_runs: int = 1) -> dict:
        """运行完整的基准评估（可多次取平均）"""

        all_results = []
        for run_id in range(num_runs):
            run_results = []
            for task in self.tasks:
                result = self.evaluate_task(task)
                run_results.append(result)
            all_results.append(run_results)
            print(f"Run {run_id + 1}/{num_runs} completed")

        return self.aggregate_results(all_results)

    def aggregate_results(self, all_results: list) -> dict:
        """汇总多次运行的结果"""

        overall_scores = []
        level_scores = {}

        for run in all_results:
            passed = sum(1 for r in run if r.passed)
            rate = passed / len(run)
            overall_scores.append(rate)

            for result in run:
                task = self._get_task(result.task_id)
                level = task.difficulty
                if level not in level_scores:
                    level_scores[level] = []
                level_scores[level].append(1.0 if result.passed else 0.0)

        avg_score = sum(overall_scores) / len(overall_scores)
        score_std = (
            sum((s - avg_score) ** 2 for s in overall_scores)
            / len(overall_scores)
        ) ** 0.5

        return {
            "benchmark": self.name,
            "num_runs": len(all_results),
            "avg_score": round(avg_score, 4),
            "score_std": round(score_std, 4),
            "by_level": {
                lv: round(sum(sc) / len(sc), 4)
                for lv, sc in level_scores.items()
            },
            "num_tasks": len(self.tasks),
            "passed_tasks": int(avg_score * len(self.tasks))
        }

    def _get_task(self, task_id: str) -> BenchmarkTask:
        for task in self.tasks:
            if task.task_id == task_id:
                return task
        raise ValueError(f"Task {task_id} not found")


class SimpleBenchmarkRunner(BaseBenchmark):
    """简化的基准运行器示例"""

    def setup_environment(self, task: BenchmarkTask):
        print(f"Setting up for task {task.task_id}: {task.description[:50]}...")

    def run_agent(self, task: BenchmarkTask) -> str:
        """这里接入实际的 Agent 系统"""
        _ = task
        return "simulated output"

    def validate_result(self, task: BenchmarkTask,
                        agent_output: str) -> tuple:
        """简化的验证逻辑"""
        _ = agent_output
        import random
        passed = random.random() > 0.3
        score = 1.0 if passed else 0.0
        return passed, score

    def teardown_environment(self, task: BenchmarkTask):
        print(f"Teardown for task {task.task_id}")
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 提供标准化的跨系统对比 | 完全反映真实场景表现 |
| 识别能力短板和退化 | 覆盖所有长尾场景 |
| 追踪技术进展趋势 | 预测 Agent 在新任务上的表现 |
| 促进社区共同进步 | 避免被针对性刷分和过拟合 |
| 为研究提供可复现基准 | 替代领域特定的评估 |
| 帮助发现系统性缺陷 | 衡量用户体验和满意度 |

## 工程优化方向

1. **多基准联合评估**: 不要只依赖一个基准，使用多个互补基准进行综合评估，覆盖不同能力维度
2. **分层采样**: 大规模基准集可以使用分层采样减少评估成本（先评估小样本，达标后再全量评估）
3. **持续更新机制**: 定期引入新任务，淘汰被过拟合的老任务，保持基准的有效性
4. **隐藏 Hold-out 集**: 保留一部分不公开的测试题，只在最终评估时使用，防止针对性优化
5. **过程日志标准化**: 不仅要记录最终分数，还要记录执行轨迹（trace），便于分析和调试
6. **成本记录**: 同时追踪分数和成本，避免"不计代价提分"的工程倾向
7. **Docker 化环境绑定**: 所有环境配置使用 Docker 或类似技术锁定，确保可复现性

## 与评估体系的关系

```
评估体系全貌
─────────────────────────────────────────────────────

 11.1 Metrics          11.2 Benchmarks       11.3 Automated Eval
 ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
 │ 定义"怎么算好"│ ──→  │ 定义"在哪测" │ ──→  │ 定义"怎么测" │
 │ 成功率       │       │ GAIA        │       │ LLM-as-Judge│
 │ 效率        │       │ SWE-bench   │       │ 自动化评分  │
 │ 鲁棒性      │       │ AgentBench   │       │ 验证脚本    │
 │ 适应性      │       │ WebArena    │       │            │
 └─────────────┘       └─────────────┘       └─────────────┘
        ↑                      ↑                      ↑
        └──────────────────────┴──────────────────────┘
                               │
                        ┌──────┴──────┐
                        │ 11.5 Regression│
                        │ ─────────── │
                        │ 持续追踪    │
                        │ 回归检测    │
                        └─────────────┘
```

- **指标（11.1）** 定义基准使用什么度量标准
- **基准（11.2）** 提供标准化的测试任务
- **自动化评估（11.3）** 提供评分手段（如 LLM-as-Judge）
- **回归测试（11.5）** 利用基准持续追踪版本间的性能变化
- **对抗测试（11.4）** 补充基准覆盖不到的边界和攻击场景
