# 11.3.6 eval-pipeline — 自动评估管道

## 简单介绍

Eval Pipeline（自动评估管道）是将评估从"手动偶尔做一次"变成"自动持续运行"的桥梁。你有了测试集，有了评分方法（LLM-as-Judge、Rubric、Unit Test），但如果没有管道把这些串起来，评估仍然是一件需要人工触发、手动分析、难以重复的事情。

```
有评估方法 ≠ 有持续评估

手动评估的日常：
  开发新功能 → 手动准备数据 → 手动跑 Agent
  → 手动调 LLM 评分 → 手动看结果
  → 第二天 PR 合并 → 又要再来一次

评估管道解决的问题：
  一键运行、自动对比基线、即时告警、历史追踪、CI/CD 集成
```

**关键思想：评估管道让"评估"本身可重复、可审计、可自动化——它是 Agent 从原型走向生产的必备基础设施。**

## 基本原理

所有 eval pipeline 都遵循同一个基础架构：

```
Test Dataset ──→ Agent Runner ──→ Scorer ──→ Reporter ──→ Alert/Store
                                                                 │
                                                            Dashboard
```

| 环节 | 做什么 | 产出什么 |
|------|--------|---------|
| Test Loader | 加载测试用例 | 结构化的 TestCase 列表 |
| Agent Runner | 在 Agent 上执行每个测试 | Agent 输出、轨迹、耗时 |
| Scorer | 用各种方法评分 | 每个测试的评分结果 |
| Result Aggregator | 统计汇总、分组分析 | 聚合统计数据 |
| Reporter | 生成报告 | Markdown/HTML/JSON 报告 |
| Alert Engine | 检测回归、触发通知 | 告警事件 |

### 批量 vs 增量

```
批量评估：每次跑所有测试，耗时最长但最全面 → 版本发布前全量验证
增量评估：只跑受变更影响的测试，快但有遗漏风险 → 开发中快速验证
回归检测：与历史基线对比，自动标出退步 → 持续监控质量趋势
```

生产环境通常是**增量 + 定期全量**的混合策略：每次 PR 跑增量，每天凌晨跑全量，每周出详细报告。

## 管道核心组件

所有的组件串起来构成完整管道：

```
                        ┌──────────────────────────────────────────┐
                        │         EvalPipeline Orchestrator        │
                        └────────────┬─────────────────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              ▼                      ▼                      ▼
     ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
     │   Test Loader   │   │  Agent Runner   │   │     Scorer     │
     │  Dataset v1/2   │──▶│  并发/超时/重试  │──▶│  LLM Judge    │
     │  Stratified     │   │  Rate Limit    │   │  Exact Match  │
     │  Versioning     │   │                │   │  Composite    │
     └────────────────┘   └────────────────┘   └───────┬────────┘
                                                        │
                         ┌──────────────────────────────┼────────────┐
                         ▼                              ▼            │
                ┌────────────────┐          ┌────────────────┐       │
                │  Aggregator    │          │   Reporter     │       │
                │  统计/分组/趋势  │─────────▶│  Markdown/JSON │       │
                └────────────────┘          └───────┬────────┘       │
                                                     │               │
                                            ┌────────▼────────┐     │
                                            │  Alert Engine   │     │
                                            │  回归/阈值/通知  │     │
                                            └─────────────────┘     │
                         ┌──────────────────────────────────────────┘
                         ▼
                ┌────────────────┐
                │   Dashboard    │
                │   & Storage    │
                └────────────────┘
```

### Test Loader — 测试加载器

从各种来源加载测试用例，支持采样和版本控制。

```python
from dataclasses import dataclass, field
from typing import List, Optional
import random
from pathlib import Path


@dataclass
class TestCase:
    task_id: str
    task: str
    expected: Optional[str] = None
    category: str = "general"
    difficulty: str = "medium"


class TestLoader:
    def __init__(self, version: str = "latest"):
        self.version = version

    def load_from_file(self, path: str) -> List[TestCase]:
        ext = Path(path).suffix
        if ext == ".json":
            return self._load_json(path)
        elif ext == ".jsonl":
            return self._load_jsonl(path)
        raise ValueError(f"Unsupported: {ext}")

    def sample(self, cases: List[TestCase],
               n: int, strategy: str = "random") -> List[TestCase]:
        """采样控制评估成本"""
        if strategy == "stratified":
            return self._stratified_sample(cases, n)
        return random.sample(cases, min(n, len(cases)))

    def _stratified_sample(self, cases, n):
        from collections import defaultdict
        by_cat = defaultdict(list)
        for c in cases:
            by_cat[c.category].append(c)
        per = max(1, n // len(by_cat))
        sampled = []
        for cat_cases in by_cat.values():
            sampled.extend(random.sample(cat_cases, min(per, len(cat_cases))))
        return sampled
```

**关键设计：支持版本化（"用 V2 还是 V3 测试集？"）和采样（"全跑 1000 条还是预算只够 100 条？"）。**

### Agent Runner — 代理执行器

并发执行 Agent，处理超时、重试、速率限制。

```python
import asyncio, time
from dataclasses import dataclass
from typing import Optional, Callable, List


@dataclass
class RunResult:
    task_id: str
    output: Optional[str] = None
    error: Optional[str] = None
    duration_ms: float = 0.0
    token_count: int = 0
    cost_usd: float = 0.0


class AgentRunner:
    def __init__(self, agent_fn: Callable, max_concurrency: int = 5,
                 timeout_ms: int = 60000, max_retries: int = 2,
                 rate_limit_rps: float = 10.0):
        self.agent_fn = agent_fn
        self.sem = asyncio.Semaphore(max_concurrency)
        self.timeout_ms = timeout_ms
        self.max_retries = max_retries
        self.rate_limit_rps = rate_limit_rps
        self._last_req = 0.0

    async def run_single(self, task: str) -> RunResult:
        start = time.monotonic()
        await self._rate_limit()
        for attempt in range(self.max_retries + 1):
            try:
                result = await asyncio.wait_for(
                    self.agent_fn(task), timeout=self.timeout_ms / 1000
                )
                return RunResult(task_id=hash(task), output=str(result),
                                 duration_ms=(time.monotonic() - start) * 1000)
            except asyncio.TimeoutError:
                if attempt == self.max_retries:
                    return RunResult(task_id=hash(task), error="Timeout")
            except Exception as e:
                if attempt == self.max_retries:
                    return RunResult(task_id=hash(task), error=str(e))

    async def run_batch(self, tasks: List[str]) -> List[RunResult]:
        return await asyncio.gather(*[self.run_single(t) for t in tasks])

    async def _rate_limit(self):
        elapsed = time.monotonic() - self._last_req
        if elapsed < 1.0 / self.rate_limit_rps:
            await asyncio.sleep(1.0 / self.rate_limit_rps - elapsed)
        self._last_req = time.monotonic()
```

### Scorer — 评分器

```python
from abc import ABC, abstractmethod


class Scorer(ABC):
    @abstractmethod
    async def score(self, task: str, output: str,
                    expected: Optional[str] = None) -> dict: ...


class LLMJudgeScorer(Scorer):
    def __init__(self, llm_client, rubric: Optional[dict] = None):
        self.llm = llm_client
        self.rubric = rubric

    async def score(self, task, output, expected=None) -> dict:
        response = await self.llm.chat.completions.create(
            messages=[{"role": "user", "content": self._build_prompt(task, output)}],
            temperature=0.0
        )
        return self._parse(response)


class ExactMatchScorer(Scorer):
    async def score(self, task, output, expected=None) -> dict:
        if expected is None:
            return {"score": 0.0}
        return {"score": 1.0 if output.strip() == expected.strip() else 0.0}


class CompositeScorer:
    """多评分器编排——加权综合"""
    def __init__(self, scorers: List[Scorer], weights: List[float]):
        self.scorers = scorers
        self.weights = weights

    async def score_all(self, task, output, expected=None) -> dict:
        results = []
        for scorer, w in zip(self.scorers, self.weights):
            r = await scorer.score(task, output, expected)
            r["weight"] = w
            results.append(r)
        ws = sum(r["score"] * r["weight"] for r in results) / sum(self.weights)
        return {"weighted_score": round(ws, 4), "details": results}
```

### Aggregator & Alert Engine

```python
import numpy as np
from collections import defaultdict


class ResultAggregator:
    def aggregate(self, results: List[dict]) -> dict:
        scores = [r.get("score", 0) for r in results]
        return {
            "mean": float(np.mean(scores)), "median": float(np.median(scores)),
            "std": float(np.std(scores)), "min": float(np.min(scores)),
            "max": float(np.max(scores)),
            "p25": float(np.percentile(scores, 25)),
            "p75": float(np.percentile(scores, 75)),
            "p90": float(np.percentile(scores, 90)),
            "count": len(scores), "error_count": sum(1 for r in results if r.get("error")),
            "passed": sum(1 for s in scores if s >= 0.5),
        }

    def by_category(self, results: List[dict]) -> dict:
        groups = defaultdict(list)
        for r in results:
            groups[r.get("category", "general")].append(r)
        return {cat: self.aggregate(items) for cat, items in groups.items()}


class AlertEngine:
    def __init__(self, thresholds: dict = None):
        self.t = thresholds or {"mean_drop": 0.05, "error_rise": 0.03}

    def check(self, current: dict, baseline: dict) -> List[dict]:
        alerts = []
        drop = baseline["mean"] - current["mean"]
        if drop > self.t["mean_drop"]:
            alerts.append({"type": "mean_drop", "severity": "warning",
                           "message": f"Mean dropped {drop:.3f}"})
        err_cur = current.get("error_count", 0) / max(current.get("count", 1), 1)
        err_base = baseline.get("error_count", 0) / max(baseline.get("count", 1), 1)
        if err_cur - err_base > self.t["error_rise"]:
            alerts.append({"type": "error_rise", "severity": "critical",
                           "message": f"Error rate up {err_cur - err_base:.1%}"})
        return alerts
```

### 完整管道编排

```python
from dataclasses import dataclass
from datetime import datetime
import time
from typing import Optional


@dataclass
class EvalReport:
    run_id: str
    timestamp: str
    stats: dict
    category_stats: dict
    alerts: list
    total_cost: float
    duration_seconds: float


class EvalPipeline:
    def __init__(self, loader, runner, scorer, reporter,
                 alert_engine, aggregator, config=None):
        self.loader = loader
        self.runner = runner
        self.scorer = scorer
        self.reporter = reporter
        self.alert_engine = alert_engine
        self.aggregator = aggregator
        self.config = config or {}

    async def run(self, dataset_path: str,
                  baseline: Optional[dict] = None) -> EvalReport:
        run_id = f"eval_{datetime.now().strftime('%Y%m%d_%H%M%S')}"
        start = time.monotonic()

        cases = self.loader.load_from_file(dataset_path)
        if self.config.get("sample_size"):
            cases = self.loader.sample(cases, self.config["sample_size"])

        run_results = await self.runner.run_batch([c.task for c in cases])

        score_results = []
        for tc, rr in zip(cases, run_results):
            if rr.error:
                score_results.append({"task_id": tc.task_id,
                                      "category": tc.category,
                                      "score": 0.0, "error": rr.error})
            else:
                scores = await self.scorer.score_all(tc.task, rr.output)
                score_results.append({"task_id": tc.task_id, "category": tc.category,
                                      "score": scores["weighted_score"],
                                      "cost": rr.cost_usd})

        stats = self.aggregator.aggregate(score_results)
        cat_stats = self.aggregator.by_category(score_results)
        alerts = self.alert_engine.check(stats, baseline) if baseline else []

        summary = self.reporter.generate_summary(stats, run_id)
        self.reporter.save(summary, run_id)

        return EvalReport(run_id=run_id, timestamp=datetime.now().isoformat(),
                          stats=stats, category_stats=cat_stats, alerts=alerts,
                          total_cost=sum(r.get("cost", 0) for r in score_results),
                          duration_seconds=time.monotonic() - start)
```

## 并发执行与性能

### 并行策略

```
任务级并行：每个测试独立执行 → 最大并发，风险：API 限速
阶段级管道：全部执行完再统一评分 → 可加缓存，适合评分依赖上下文
流式并行：执行完立刻评分 → 流水线模式，适合长测试尽早看结果
```

### 缓存策略

```python
class EvalCache:
    """三级缓存：输出 → 评分 → 基线"""
    def __init__(self, cache_dir: str = "./.eval_cache"):
        import os; os.makedirs(cache_dir, exist_ok=True)
        self.cache_dir = cache_dir

    def _key(self, task: str, config: str) -> str:
        import hashlib
        return hashlib.md5(f"{task}:{config}".encode()).hexdigest()

    def get(self, task: str, config: str) -> Optional[str]:
        import os, json
        path = f"{self.cache_dir}/{self._key(task, config)}.json"
        if os.path.exists(path):
            with open(path) as f:
                return json.load(f).get("output")
        return None

    def set(self, task: str, config: str, output: str):
        import json
        with open(f"{self.cache_dir}/{self._key(task, config)}.json", "w") as f:
            json.dump({"output": output}, f)
```

```
L1 输出缓存：相同任务+相同Agent配置 → 跳过Agent执行
L2 评分缓存：相同输出+相同Rubric → 跳过LLM评分
L3 历史基线：缓存聚合统计 → 回归比较
```

### 成本追踪

```python
class CostTracker:
    def __init__(self, budget: float):
        self.budget = budget
        self.spent = 0.0

    def would_exceed(self, cost: float) -> bool:
        return self.spent + cost > self.budget

    def record(self, cost: float):
        self.spent += cost

    @property
    def remaining(self) -> float:
        return self.budget - self.spent
```

## 回归检测

回归检测是 eval pipeline 的核心价值——**不跑管道不知道变好还是变差了**。

```
检测方法一览：
┌────────────────┬──────────────────────────────┬────────────────────┐
│ 方法            │ 原理                          │ 适用场景            │
├────────────────┼──────────────────────────────┼────────────────────┤
│ 均值下降检测    │ 当前平均分 < 基线 - 阈值        │ 整体质量监控        │
│ 统计显著性检验  │ t-test 计算 p-value            │ 小样本需要统计信心  │
│ 类别回归分析    │ 按功能/难度分组检测              │ 定位具体问题域      │
│ 异常值检测      │ Z-score / IQR 发现极端低分      │ 发现极端失败案例    │
│ 趋势分析        │ 滑动窗口+斜率检测持续下降        │ 长期退化早期预警    │
└────────────────┴──────────────────────────────┴────────────────────┘
```

```python
from scipy import stats as scipy_stats


class RegressionDetector:
    def detect(self, current: List[float], baseline: List[float],
               p_threshold: float = 0.05) -> dict:
        t_stat, p_val = scipy_stats.ttest_ind(baseline, current)
        effect = np.mean(baseline) - np.mean(current)
        return {"degraded": p_val < p_threshold and effect > 0,
                "p_value": float(p_val), "effect_size": float(effect),
                "mean_baseline": float(np.mean(baseline)),
                "mean_current": float(np.mean(current))}

    def by_category(self, current, baseline, categories) -> List[dict]:
        regressions = []
        for cat in categories:
            cur = current.get(cat, {}).get("scores", [])
            base = baseline.get(cat, {}).get("scores", [])
            if not cur or not base:
                continue
            result = self.detect(cur, base)
            if result["degraded"]:
                regressions.append({"category": cat, "drop": result["effect_size"],
                                    "p_value": result["p_value"]})
        return regressions
```

### 告警示例

```
[INFO]  Run Complete: eval_20260717_143022
[INFO]  500 cases | Pass: 423 (84.6%) | Cost: $12.47 | 3m42s
[WARN]  Regression detected!

  指标            基线      当前       变化
─────────────────────────────────────────────
  Overall Mean   0.842    0.801     ▼ -4.9%
  Accuracy       0.890    0.823     ▼ -7.5%  (p=0.003)
  Error Rate     1.2%     4.0%      ▲ +2.8%

[ALERT] Accuracy dropped 7.5% (p=0.003)
[ALERT] Error rate increased 2.8%
```

## CI/CD 集成

```yaml
# .github/workflows/eval-pipeline.yml
name: Agent Eval Pipeline
on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * *'   # 每日全量

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install -r requirements.txt

      - name: PR eval
        if: github.event_name == 'pull_request'
        run: python run_pipeline.py --dataset ./tests/pr_sample.jsonl \
             --baseline ./baselines/latest.json --budget 5.0 --fail-on-regression

      - name: Nightly eval
        if: github.event_name == 'schedule'
        run: python run_pipeline.py --dataset ./tests/full_dataset.jsonl \
             --baseline ./baselines/latest.json --save-report
```

### 集成模式与门禁

```
PR 级快速评估：100-200 条采样，5-15min，$1-5 → 门禁：有退化阻止合并
夜间全量评估：1000-5000 条，30-120min，$10-50 → 详细报告+存档
发布前深度评估：全量+对抗样本，2-8h，$50-200 → 需人工复核
```

```python
class EvalGate:
    def __init__(self, config: dict = None):
        self.max_drop = (config or {}).get("max_score_drop", 0.03)
        self.max_error = (config or {}).get("max_error_rate", 0.05)

    def should_block(self, report: EvalReport) -> tuple:
        reasons = []
        for a in report.alerts:
            if a["type"] == "mean_drop" and a["delta"] > self.max_drop:
                reasons.append(f"Score drop {a['delta']:.3f} > {self.max_drop}")
        err_rate = (report.stats.get("error_count", 0)
                    / max(report.stats.get("count", 1), 1))
        if err_rate > self.max_error:
            reasons.append(f"Error rate {err_rate:.1%} > {self.max_error:.1%}")
        return (len(reasons) > 0, reasons)
```

## 工具与框架

```
工具                  核心能力                     适用场景
────────────────────  ──────────────────────────  ─────────────────
LangSmith             端到端 eval pipeline        LangChain 深度集成
MLflow                实验追踪+模型注册            已有 ML 基础设施
Weights & Biases      实验跟踪+可视化              研究导向团队
Arize AI/WhyLabs      生产监控+漂移检测            生产 Agent 持续监控
Airflow/Prefect       通用工作流编排               复杂多阶段管道
```

## 生产级实践

### 多阶段管道

```
Stage 1: 冒烟测试 (Smoke) — 10-20 条关键测试，1-2 分钟，失败即阻断
Stage 2: 完整评估 (Full) — 500-2000 条，30-60 分钟，全面评估+回归检测
Stage 3: 深度评估 (Deep) — 全量+对抗样本，2-8 小时，含人工复核
```

### 测试集卫生

```
黄金测试集 (Golden Set): ~200 条手动精选，长期不变，每个版本必须通过
留出测试集 (Held-out Set): 从未用于调试，防止过拟合，每季度更新
动态测试集 (Dynamic Set): 从真实用户请求采样，持续更新反映实际
```

### 版本化

```
数据版本化: test_dataset_v1 → v2 → v3（DVC / Git LFS）
配置版本化: agent_v1.2.3, rubric_v2, pipeline_config_v5
运行版本化: eval_20260717_v1.2.3（每次记录完整配置快照，确保可重复）
```

### 人工复核

```python
class HumanReviewLoop:
    def __init__(self, threshold: float = 0.3):
        self.threshold = threshold

    def needs_review(self, result: dict) -> bool:
        details = result.get("details", {}).get("details", [])
        if len(details) >= 2:
            scores = [s["score"] for s in details]
            if max(scores) - min(scores) > self.threshold:
                return True
        if 0.4 < result.get("score", 0) < 0.6:  # 接近及格线
            return True
        return bool(result.get("error"))

    def flag(self, results: List[dict]) -> List[dict]:
        flagged = []
        for r in results:
            if self.needs_review(r):
                r["needs_review"] = True
                flagged.append(r)
        return flagged
```

## 挑战与局限

### 维护复杂度

```
管道本身成了另一个需要维护的系统：
  ├─ 组件升级：换 LLM API → 更新 Runner
  ├─ 测试集更新：新功能 → 新测试 → 新评分标准
  ├─ 缓存失效：改动 schema → 旧缓存不可用
  └─ 依赖管理：各组件各自更新 → 兼容性问题
缓解：保持接口简洁、Docker 容器化、管道自身也纳入评估
```

### 回归检测可靠性 & 成本

```
假阳性：报告退步了但其实没有 → 采样偏差、评分方差大 → 统计检验降低误报
假阴性：退步了但没检测到 → 测试集不敏感、阈值太宽松 → 多方法交叉验证

测试集漂移：用户使用方式在变但测试集还是半年前的
  症状：测试集分数高但用户投诉在增加 → 定期用真实流量更新

成本爆炸：2000 条 × 3 个 Judge × 每次 PR = 大量 API 调用
  缓解：分层评估、缓存优化、预算硬上限、智能采样
```

### 能做与不能做

| 能做 | 不能做 |
|------|--------|
| 自动大规模评估，解放人工 | 完全替代人类判断——复杂案例仍需复核 |
| 自动检测质量退步，及时告警 | 保证 100% 无假阳性/假阴性的回归检测 |
| 提供可重复、可审计的评估流程 | 消除测试集质量问题（偏差、漂移） |
| CI/CD 集成实现质量门禁 | 解决所有成本问题——大规模评估仍昂贵 |
| 追踪长期质量趋势 | 自动适应数据分布变化——需维护测试集 |

## 未来方向

```
持续在线评估：
  当前管道 CI/定时触发 → 未来生产中实时采样评估
  检测异常立刻告警甚至自动回滚
  挑战：实时延迟、隐私合规、反馈循环

自适应测试集：
  上次发现"数学推理"退步 → 这次多选数学推理测试
  像"评估版的主动学习"，资源用在刀口上

AI 生成测试用例：
  从真实日志、边界条件、已知失败模式自动生成
  挑战：生成质量保证，需要"测试生成器"的元评估

自改进管道：
  管道自己优化自己——重采样、评分标准微调、阈值自适应、法官轮换
  目前学术界探索中，生产环境还不成熟
```

## 实施路线图

```
第一阶段：跑起来
  ├─ 有一个可用的 eval pipeline
  ├─ 能自动跑测试、出报告
  └─ 知道花了多少钱、得了多少分

第二阶段：用起来
  ├─ CI/CD 集成，PR 自动触发
  ├─ 回归检测，退化即告警
  ├─ 基线管理，版本间可比较
  └─ 报告可视化，团队都能看懂

第三阶段：优化起来
  ├─ 智能采样，控制成本
  ├─ 缓存策略，提高速度
  ├─ 自适应阈值，减少噪音
  ├─ 人工复核闭环，持续改进
  └─ 测试集卫生，防止漂移

不要第一天就追求完美——先让它跑起来，再让它聪明起来。
```
