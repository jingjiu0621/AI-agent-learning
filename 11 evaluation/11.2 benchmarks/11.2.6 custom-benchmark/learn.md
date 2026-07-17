# 11.2.6 custom-benchmark — 自定义基准构建

## 简单介绍

现有的 Agent 基准（GAIA、SWE-bench、WebArena 等）覆盖了通用能力，但**真实生产环境总是需要专有的评测集**。你自己的 Agent 要在你自己的业务场景、数据、工具链和用户群体上工作良好——没有一个公开基准能完全代表你的场景。

自定义基准构建，就是针对具体的 Agent 应用场景，设计、构建和维护一套专属的评测体系。

```
公开基准 vs 自定义基准
────────────────────────────────────────────────────────
公开基准                        自定义基准
────                             ────────
告诉你在通用场景下 Agent 多强    告诉你在你的场景下 Agent 多强
与同行公平对比                   与历史版本对比
全社区共享                       你自己的资产
更新慢，容易被过拟合              及时更新，反映业务变化
覆盖不了你的长尾业务              专门覆盖你的业务场景
```

## 基本原理

### 自定义基准的四大组件

```
                        ┌──────────────────────────────┐
                        │    自定义基准系统              │
                        ├──────────────────────────────┤
                        │ 1. 任务生成 (Task Generation)  │  ← 怎么出题
                        │ 2. 参考答案 (Ground Truth)    │  ← 什么是对的
                        │ 3. 评估执行 (Eval Runner)     │  ← 怎么执行
                        │ 4. 评分聚合 (Score Aggregation) │ ← 怎么算分
                        └──────────────────────────────┘
```

### 任务源的 5 种渠道

```
任务来源                       方法                    优势                缺点
───────────────────────  ───────────────────────  ─────────────────  ──────────────────────
1. 真实用户日志采样      从生产环境取实际用户请求     最真实                隐私处理复杂
                        去隐私后作为评测任务                             用户意图不一定明确

2. 人工编写             领域专家/标注团队写测试用例    质量高、可控         成本高、扩展性差
                        覆盖边界情况和典型场景                           可能遗漏某些模式

3. LLM 生成            用 LLM 根据种子需求生成任务    成本低、数量大         需要人工校验质量
                        人工审核后加入评测集                              可能产生模式化任务

4. 模版化自动生成        定义参数化模板                 可大量扩展             模板模式可能固化
                        (如"查询 {商品类别} 的价格")                      需要丰富的参数池

5. 用户反馈挖掘         从用户投诉/差评中提炼测试用例    高价值                 数量有限
                        防止类似问题再次出现                               需要仔细甄别根因
```

### 任务模板工程

```python
import random
import json
from typing import List
from dataclasses import dataclass, field


@dataclass
class EvalTask:
    """评测任务"""
    id: str
    description: str                # 给 Agent 的指令
    expected_answer: str            # 标准答案
    category: str                   # 功能分类
    difficulty: str                 # easy / medium / hard
    tags: list = field(default_factory=list)
    context: dict = field(default_factory=dict)    # 初始上下文


@dataclass
class TaskTemplate:
    """任务模板——参数化生成"""
    id_template: str
    instruction_template: str       # 带 {placeholder} 的模板
    expected_template: str          # 答案模板（可计算）
    category: str
    difficulty: str
    slots: dict                     # 参数定义 {name: [可选值列表]}


class TaskGenerator:
    """基于模板的任务生成器"""

    def __init__(self, templates: List[TaskTemplate]):
        self.templates = templates

    def generate(self, n: int = 100) -> List[EvalTask]:
        """批量生成评测任务"""
        tasks = []

        for t in self.templates:
            for _ in range(n // len(self.templates)):
                # 从 slots 中采样参数
                params = {}
                for slot_name, options in t.slots.items():
                    if isinstance(options, list):
                        params[slot_name] = random.choice(options)
                    elif callable(options):
                        params[slot_name] = options()
                    else:
                        params[slot_name] = options

                # 填充模板
                instruction = t.instruction_template.format(**params)
                expected = t.expected_template.format(**params)

                task = EvalTask(
                    id=t.id_template.format(**params),
                    description=instruction,
                    expected_answer=expected,
                    category=t.category,
                    difficulty=t.difficulty,
                    tags=list(params.keys())
                )
                tasks.append(task)

        random.shuffle(tasks)
        return tasks


# 使用示例
SHOPPING_TEMPLATES = [
    TaskTemplate(
        id_template="price-query-{product}",
        instruction_template="查询 {product} 的当前价格",
        expected_template="{product} 当前价格为 {price} 元",
        category="information_retrieval",
        difficulty="easy",
        slots={
            "product": ["iPhone 15", "MacBook Air", "AirPods Pro", "iPad Air"],
            "price": lambda: random.randint(100, 10000)
        }
    ),
    TaskTemplate(
        id_template="order-status-{order_id}",
        instruction_template="查询订单 {order_id} 的当前状态",
        expected_template="订单 {order_id} 的状态是 {status}",
        category="order_query",
        difficulty="easy",
        slots={
            "order_id": lambda: f"ORD-{random.randint(10000, 99999)}",
            "status": ["已发货", "配送中", "已完成", "已取消"]
        }
    ),
    TaskTemplate(
        id_template="complex-search-{category}-{min_price}-{max_price}",
        instruction_template=(
            "在 {category} 类目中搜索价格在 {min_price} "
            "到 {max_price} 元之间的商品，按 {sort_by} 排序，"
            "返回前 {top_k} 个结果"
        ),
        expected_template="返回 {category} 类目下价格 {min_price}-{max_price} 元的商品列表",
        category="search",
        difficulty="hard",
        slots={
            "category": ["电子产品", "家居", "图书", "服装"],
            "min_price": [100, 500, 1000],
            "max_price": [1000, 5000, 10000],
            "sort_by": ["价格升序", "销量降序", "评分降序"],
            "top_k": [3, 5, 10]
        }
    )
]
```

## 背景

### 为什么不能只靠公开基准？

```
典型的故事：
  团队用 GAIA + SWE-bench 评估他们的 Agent，分数都不错。
  上线后客服投诉飙升——Agent 处理不了这个行业的特定术语。

原因：
  GAIA 测试多步推理——但你们的 Agent 面对的更多是简单的信息查询
  SWE-bench 测试代码修改——但你们的 Agent 主要处理客服问题
  WebArena 测试网页交互——但你们的 Agent 主要调用内部 API

结论：
  公开基准提供的是"通用能力下界"，不是"你场景下的实际表现"。
```

### 自定义基准的典型成熟度阶段

```
Level 0: 无基准
  凭感觉判断 Agent 好坏 — "好像比上周好一点"

Level 1: 即兴测试
  几个人工编写的测试用例 — 覆盖面很窄，但比没有强
  每次评估手动运行 — 费时费力

Level 2: 结构化任务集
  50-200 个覆盖主要功能的测试用例
  可以半自动运行 — 但参考答案维护困难

Level 3: 自动化管线
  200-1000+ 任务，模板化生成，自动评分
  集成到 CI/CD 流程中 — 每次部署自动跑
  有开发集/回归集/隐藏集的三区划分

Level 4: 持续进化体系
  从真实用户反馈中自动挖掘新任务
  定期分析评测集质量（饱和度、区分度、过拟合程度）
  支持 A/B 测试和多维度评分
```

## 核心矛盾

**覆盖度与维护成本的矛盾。**

```
扩展评测集 → 覆盖更多场景 → 评测更可靠
  但 → 构建参考答案更耗时 → 验证更复杂 → 维护负担更重

缩小评测集 → 维护简单
  但 → 漏掉关键场景 → 评测不可靠 → 退化检测不到

关键问题：
  覆盖度每提升 10%，维护成本可能增加 30%
  需要找到"够用"的平衡点
```

## 三区划分策略

这是自定义基准中最核心的工程实践之一：

```
开发集 (Dev Set)         回归集 (Regression Set)     隐藏集 (Hold-out Set)
─────────────────        ─────────────────────        ────────────────────
用途：日常调试和优化       用途：防止回归              用途：最终验证
数量：~50%                数量：~30%                  数量：~20%
可见性：完全可见            可见性：部分可见（可查失败）   可见性：不可见
更新频率：频繁更新          更新频率：定期补充            更新频率：随版本更新

交互模式：
                      ┌─────────────┐
                      │ 在开发集上   │
                      │ 迭代优化     │
                      └──────┬──────┘
                             ▼
                      ┌─────────────┐
                      │ 在回归集上   │◄── 检测是否把以前的
                      │ 验证效果     │    好结果搞坏了
                      └──────┬──────┘
                             ▼
                      ┌─────────────┐
                      │ 在隐藏集上   │◄── 最终评估，防止
                      │ 最终评分     │    过拟合到开发集
                      └─────────────┘
```

```python
@dataclass
class EvalDataset:
    """三区划分的数据集"""
    name: str
    dev: List[EvalTask]          # 开发集
    regression: List[EvalTask]   # 回归集
    holdout: List[EvalTask]      # 隐藏集

    @staticmethod
    def split(tasks: List[EvalTask],
              dev_ratio: float = 0.5,
              regression_ratio: float = 0.3,
              holdout_ratio: float = 0.2) -> 'EvalDataset':
        """按比例划分为三区"""
        assert abs(dev_ratio + regression_ratio + holdout_ratio - 1.0) < 0.01

        shuffled = tasks.copy()
        random.shuffle(shuffled)
        n = len(shuffled)
        dev_end = int(n * dev_ratio)
        reg_end = dev_end + int(n * regression_ratio)

        return EvalDataset(
            name="custom_benchmark",
            dev=shuffled[:dev_end],
            regression=shuffled[dev_end:reg_end],
            holdout=shuffled[reg_end:]
        )
```

## 评估执行框架

```python
class CustomBenchmark:
    """自定义基准评估框架"""

    def __init__(self, dataset: EvalDataset, verbose: bool = False):
        self.dataset = dataset
        self.verbose = verbose
        self.results_history = []

    def evaluate(self, agent_callable, split: str = "dev") -> dict:
        """在指定分集上评估 Agent"""
        tasks = getattr(self.dataset, split)
        results = []

        for task in tasks:
            try:
                agent_response = agent_callable(task.description)
                passed, score = self.grade(task, agent_response)
            except Exception as e:
                agent_response = f"ERROR: {e}"
                passed = False
                score = 0.0

            results.append({
                "task_id": task.id,
                "passed": passed,
                "score": score,
                "agent_output": agent_response,
                "category": task.category,
                "difficulty": task.difficulty
            })

        return self.aggregate(results)

    def grade(self, task: EvalTask, response: str) -> tuple:
        """
        评分函数——需要根据具体场景定制。

        几种常见的评分策略：
          1. 精确匹配：response == task.expected_answer
          2. 包含匹配：task.expected_answer in response
          3. 语义匹配：embedding 相似度 > 阈值
          4. LLM 评判：让 LLM 判断是否达标
          5. 正则匹配：re.search(pattern, response)
        """
        # 默认使用简单的包含匹配
        passed = task.expected_answer.lower() in response.lower()
        score = 1.0 if passed else 0.0
        return passed, score

    def aggregate(self, results: list) -> dict:
        """汇总评分"""
        n = len(results)
        success_rate = sum(1 for r in results if r["passed"]) / n

        # 按类别聚合
        by_category = {}
        for r in results:
            cat = r["category"]
            if cat not in by_category:
                by_category[cat] = []
            by_category[cat].append(r["passed"])

        # 按难度聚合
        by_difficulty = {}
        for r in results:
            diff = r["difficulty"]
            if diff not in by_difficulty:
                by_difficulty[diff] = []
            by_difficulty[diff].append(r["passed"])

        return {
            "num_tasks": n,
            "success_rate": success_rate,
            "by_category": {
                k: sum(v) / len(v) for k, v in by_category.items()
            },
            "by_difficulty": {
                k: sum(v) / len(v) for k, v in by_difficulty.items()
            },
            "avg_score": sum(r["score"] for r in results) / n
        }

    def regression_check(self, new_results: dict) -> dict:
        """回归检测——与历史结果对比"""
        if not self.results_history:
            self.results_history.append(new_results)
            return {"regression_detected": False, "message": "基线已建立"}

        baseline = self.results_history[-1]
        regressions = []

        # 检查总体回归
        delta = new_results["success_rate"] - baseline["success_rate"]
        if delta < -0.05:
            regressions.append({
                "type": "overall",
                "from": baseline["success_rate"],
                "to": new_results["success_rate"],
                "delta": delta
            })

        # 检查各难度的回归
        for diff in baseline["by_difficulty"]:
            old = baseline["by_difficulty"].get(diff, 0)
            new = new_results["by_difficulty"].get(diff, 0)
            if new < old - 0.08:
                regressions.append({
                    "type": f"difficulty_{diff}",
                    "from": old,
                    "to": new,
                    "delta": new - old
                })

        self.results_history.append(new_results)

        return {
            "regression_detected": len(regressions) > 0,
            "regressions": regressions,
            "total_change": delta
        }
```

## 评分方式决策矩阵

不同的评分方式适用于不同的场景，选择合适的评分策略是自定义基准的关键设计决策：

```
评分方式              适合场景                  优点                  缺点
─────────────────  ──────────────────────  ──────────────────  ─────────────────────
精确匹配             答案确定的任务              零歧义               太严格，忽略同义表达
(如 SQL 生成)       答案必须完全正确                                   写法的多样性被惩罚

包含匹配             自由文本回答                简单高效             可能漏检不精确匹配
(关键词检测)         Agent 输出包含关键信息                           容易误判

正则/编程匹配         有明确格式的输出            灵活精确              维护成本高
(pattern)           如日期、金额、代码                               pattern 需要持续更新

语义相似度            开放式回答                 容忍自然语言表达       阈值设定困难
(embedding cosine)  无标准答案                                     需要调试合适的阈值

LLM-as-Judge        答案开放、需要判断质量         最灵活               成本高，有偏差
                     需要推理评判              可以给出部分分数       可靠性需要验证

多评委投票            高标准场景                  减少偏差              成本线性增加
(3+ LLM)            需要高可靠性                                      结果合并需要策略

人工评分              终极验证                  最准确               不可扩展
                     小批量关键场景                                   速度慢
```

## 维护与进化

### 基准质量监控

```python
class BenchmarkHealthMonitor:
    """监控自定义基准的健康状况"""

    def __init__(self, benchmark: CustomBenchmark):
        self.benchmark = benchmark
        self.history = []

    def compute_saturation(self, run_results: dict) -> float:
        """
        计算基准饱和度。

        饱和度 = 当前基准区分不同 Agent 版本的能力。
        如果所有 Agent 版本得分都在 95-98% 之间，饱和度很高——需要更新。
        """
        scores = run_results.get("per_version_scores", [])
        if len(scores) < 2:
            return 0.0

        score_range = max(scores) - min(scores)
        # 分数范围 < 5% → 高度饱和
        # 分数范围 > 20% → 区分度良好
        saturation = max(0, 1 - score_range / 0.20)
        return saturation

    def compute_contamination_risk(self,
                                    model_training_date: str) -> float:
        """
        估算基准数据被模型训练数据污染的风险。

        如果基准任务是在模型训练截止日期之前创建的，
        模型可能"见过"类似的数据。
        """
        from datetime import datetime
        cutoff = datetime.fromisoformat(model_training_date)

        tasks_before_cutoff = 0
        for task in self.benchmark.dataset.dev:
            # 假设任务有创建日期元数据
            task_date = task.context.get("created_date")
            if task_date and datetime.fromisoformat(task_date) < cutoff:
                tasks_before_cutoff += 1

        total = len(self.benchmark.dataset.dev)
        return tasks_before_cutoff / total if total > 0 else 0

    def suggest_updates(self) -> list:
        """建议基准更新"""
        suggestions = []
        if self.compute_saturation({"per_version_scores": [95, 96, 94, 97]}) > 0.7:
            suggestions.append("基准可能已饱和，建议添加更难的新任务")
        if self.compute_contamination_risk("2024-06-01") > 0.5:
            suggestions.append("超过 50% 的任务创建于模型训练截止日期前，建议补充新任务")
        return suggestions
```

### 任务淘汰与更新的生命周期

```
每个任务都有一个生命周期：

  创建 → 加入开发集 → 反复使用 → 模型逐渐适应 → 区分度下降 → 淘汰/归档

建议的生命周期管理：
  1. 开发集任务：活跃期 3-6 个月
  2. 回归集任务：活跃期 6-12 个月（防止退化的重要屏障）
  3. 隐藏集任务：最多使用 1 年，到期更换
  4. 淘汰任务归档到"历史库"，用于长期趋势分析
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 衡量 Agent 在你业务场景上的表现 | 完全替代公开基准在通用能力上的评估 |
| 检测版本间的性能退化和提升 | 预测从未见过的新场景的表现 |
| 引导优化方向（聚焦你的核心场景） | 覆盖所有长尾和边界情况 |
| 建立团队内部的质量门禁标准 | 评估用户体验等主观质量维度 |
| 长期追踪能力演进趋势 | 作为绝对能力的权威声明（定制太强） |

## 工程优化方向

1. **数据集版本管理**：用 Git LFS 或 DVC 管理评测数据集版本，每次更新记录变更日志，确保历史可追溯

2. **参考答案半自动生成**：先用 LLM 生成参考答案，人工审核修正，比完全人工编写效率高 10 倍

3. **分层采样评估**：在全量开发集上评估成本太高时，按类别+难度分层采样一个子集（如 20%），快速获取近似结果

4. **失败分析自动化**：每次评估后自动对失败任务聚类（按类别、难度、Agent 行为模式），自动生成失败分析报告

5. **逐步淘汰机制**：定期（每月/每季度）计算每个任务的区分度和稳定性，自动标记低质量任务供人工确认淘汰

6. **与 CI/CD 集成**：PR 提交时自动运行回归集（全量）和开发集子集（快速），合并前用隐藏集做最终门控

## 与公开基准的配合使用

```
推荐的自定义+公开基准配合策略：

  公开基准（每月跑1次）
  ────────────────────
  GAIA          → 监控通用推理能力
  SWE-bench     → 监控代码能力
  ToolBench     → 监控工具使用能力
  WebArena      → 监控网页交互能力
  目的：确保 Agent 的通用能力没有退化

  自定义基准（每次部署跑）
  ─────────────────────────
  核心业务场景集  → 确保核心功能正常
  回归集         → 防止已知问题复发
  用户投诉转化集  → 确保投诉场景已修复
  目的：确保 Agent 在业务场景上表现良好

  两者关系：
  公开基准是"安全网"——防止 Agent 在通用能力上退步
  自定义基准是"计分板"——衡量 Agent 的实际业务价值
```
