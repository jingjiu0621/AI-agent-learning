# 14.2.5 SLA 设计 — Service Level Agreement Design

> **核心思想**：企业级 Agent 系统的 SLA（服务等级协议）比传统系统复杂得多——Agent 不仅涉及可用性和延迟，还涉及**推理质量**、**工具调用准确性**和**行为一致性**等非确定性维度。SLA 设计的目标是在"LLM 的非确定性"和"企业服务的确定性承诺"之间建立可衡量、可追踪、可问责的桥梁。

---

## 1. 基本原理

### 1.1 Agent SLA 的四维模型

传统 SLA 通常关注**可用性**和**性能**，Agent 系统需要扩展到四个维度：

```
                    传统 SLA                               Agent SLA
                    ─────────                               ─────────
                    ┌──── 可用性 ────┐                ┌──── 可用性 ────┐
                    │ 99.9% Uptime   │                │ LLM API + 平台 │
                    └────────────────┘                └────────────────┘
                    ┌──── 延迟 ──────┐                ┌──── 延迟 ──────┐
                    │ P99 < 200ms    │                │ 多步任务 < 30s │
                    └────────────────┘                └────────────────┘
                                                      ┌──── 质量 ──────┐
                                                      │ 准确/相关/安全 │
                                                      └────────────────┘
                                                      ┌──── 新鲜度 ────┐
                                                      │ 知识截止日期   │
                                                      └────────────────┘
```

| SLA 维度 | 传统系统 | Agent 系统 | 关键挑战 |
|----------|----------|-----------|----------|
| **可用性** | 99.9%+ uptime | 99.9%+ + LLM API 可用性 | LLM API 不在控制范围内（第三方依赖） |
| **延迟** | P99 < 200ms | 简单任务 < 3s，复杂 < 60s | LLM 推理时间非确定且差异大 |
| **质量** | 100% 正确 | 准确率 85-95%（依赖任务） | LLM 输出非确定，无法保证 100% 正确 |
| **新鲜度** | 实时 | 依赖知识库更新周期 | LLM 知识截止日期，需要 RAG 补充 |

### 1.2 背景与演进

**之前怎么做**：
- **SLA 1.0（基础设施级）**：仅关注服务器 uptime，99.9% 可用性
- **SLA 2.0（应用级）**：关注 API 响应时间和错误率
- **SLA 3.0（Agent 级）**：需要关注推理质量、工具准确性和行为一致性

**核心矛盾**：企业要求**确定性承诺**（"系统 x% 的时间内正常工作"），但 Agent 基于 LLM 的推理本质上是**概率性的**。你不能像保证 API 响应时间一样保证"Agent 的推理 99.9% 正确"。

### 1.3 核心原则

```python
SLA_DESIGN_PRINCIPLES = {
    "分维度承诺": "对不同维度使用不同的 SLA 粒度，不为质量提供绝对保证",
    "分层 SLO": "基础设施层(SLO) ≠ Agent 行为层(SLO)，分开定义和测量",
    "错误预算": "为每个维度定义错误预算，用于指导发布决策",
    "持续校准": "SLA 不是一成不变的，需要基于实际数据持续调整",
    "透明报告": "定期发布 SLA 达成报告，包括未达标的原因分析",
}
```

---

## 2. 四维 SLA 详解

### 2.1 可用性 SLA (Availability)

```
可用性 SLA 组件:

┌─────────────────────────────────────────────────────┐
│  Agent 平台可用性                                    │
│  ├── API 网关可用性 (99.99%)                        │
│  ├── Agent 编排服务 (99.95%)                        │
│  ├── 向量数据库可用性 (99.95%)                       │
│  ├── 工具执行服务 (99.9%)                           │
│  └── 知识库服务 (99.9%)                             │
├─────────────────────────────────────────────────────┤
│  LLM API 可用性 (第三方依赖)                         │
│  ├── 主要提供商 (OpenAI/Anthropic) P99 API 可用性    │
│  └── 降级提供商（备用模型）                          │
├─────────────────────────────────────────────────────┤
│  复合可用性计算                                      │
│  平台可用性 × LLM 可用性 = 端到端可用性               │
│  99.9% × 99.95% = 99.85%                            │
└─────────────────────────────────────────────────────┘
```

#### 复合可用性计算

```python
class AvailabilityCalculator:
    """Agent 系统复合可用性计算"""
    
    def __init__(self):
        self.components = {
            "api_gateway": {"sla": 0.9999, "weight": 0.1},
            "agent_orchestrator": {"sla": 0.9995, "weight": 0.2},
            "vector_db": {"sla": 0.9995, "weight": 0.15},
            "tool_executor": {"sla": 0.999, "weight": 0.15},
            "knowledge_base": {"sla": 0.999, "weight": 0.1},
            "llm_primary": {"sla": 0.9995, "weight": 0.2},
            "llm_fallback": {"sla": 0.99, "weight": 0.0},  # 仅故障时使用
        }
    
    def calculate_compound_availability(self) -> float:
        """
        计算复合可用性
        
        串联组件: 所有组件必须同时可用
        A_compound = A1 × A2 × A3 × ...
        """
        availability = 1.0
        for name, config in self.components.items():
            if config["weight"] > 0:  # 主路径组件
                availability *= config["sla"]
        return availability
    
    def calculate_with_llm_failover(self) -> float:
        """
        带 LLM 故障转移的可用性
        
        并联组件: (主 LLM 可用) + (主 LLM 不可用 × 备用 LLM 可用)
        A_llm = A_primary + (1 - A_primary) × A_fallback
        """
        primary = self.components["llm_primary"]["sla"]
        fallback = self.components["llm_fallback"]["sla"]
        llm_availability = primary + (1 - primary) * fallback
        
        # 其他组件串联
        other = 1.0
        for name, config in self.components.items():
            if name not in ("llm_primary", "llm_fallback") and config["weight"] > 0:
                other *= config["sla"]
        
        return other * llm_availability
    
    def monthly_downtime(self, availability: float) -> str:
        """将可用性转换为每月停机时间"""
        minutes_per_month = 30 * 24 * 60
        downtime_minutes = minutes_per_month * (1 - availability)
        
        if downtime_minutes < 1:
            return f"{downtime_minutes * 60:.0f} 秒/月"
        elif downtime_minutes < 60:
            return f"{downtime_minutes:.1f} 分钟/月"
        else:
            return f"{downtime_minutes / 60:.1f} 小时/月"
```

### 2.2 延迟 SLA (Latency)

Agent 系统的延迟比传统 API 复杂得多，因为任务可能有不同的复杂度：

```
延迟 SLA 分层:

┌────────────────────────────────────────────────────┐
│  层 1: 简单查询 (5%)                                │
│  示例: "现在几点了？" "今天的天气如何？"              │
│  目标: P50 < 1s, P95 < 3s                          │
│                                                        │
│  层 2: 知识检索 (40%)                               │
│  示例: "公司的年假政策是什么？"                      │
│  目标: P50 < 3s, P95 < 8s                          │
│                                                        │
│  层 3: 多步推理 (35%)                               │
│  示例: "分析上季度销售数据，找出下降最多的区域"       │
│  目标: P50 < 10s, P95 < 25s                        │
│                                                        │
│  层 4: 复杂任务 (15%)                               │
│  示例: "写一份竞品分析报告并发送给团队"              │
│  目标: P50 < 30s, P95 < 60s                        │
│                                                        │
│  层 5: 长时间任务 (5%)                              │
│  示例: "爬取 100 个网页并生成摘要"                  │
│  目标: P50 < 5min, P95 < 15min                     │
└────────────────────────────────────────────────────┘
```

#### 延迟测量代码

```python
import time
from collections import defaultdict

class LatencySLAMonitor:
    """延迟 SLA 监控"""
    
    def __init__(self):
        # 各层级的延迟目标
        self.latency_targets = {
            "simple_query": {"p50": 1.0, "p95": 3.0, "p99": 5.0},
            "knowledge_retrieval": {"p50": 3.0, "p95": 8.0, "p99": 15.0},
            "multi_step": {"p50": 10.0, "p95": 25.0, "p99": 45.0},
            "complex_task": {"p50": 30.0, "p95": 60.0, "p99": 120.0},
            "long_running": {"p50": 300.0, "p95": 900.0, "p99": 1800.0},
        }
        self.latency_records: dict[str, list[float]] = defaultdict(list)
    
    def record_latency(self, task_type: str, latency_seconds: float):
        """记录任务延迟"""
        self.latency_records[task_type].append(latency_seconds)
    
    def check_sla_compliance(self, task_type: str) -> dict:
        """检查特定类型的延迟 SLA 达标情况"""
        records = self.latency_records.get(task_type, [])
        if not records:
            return {"compliant": True, "reason": "no_data"}
        
        sorted_records = sorted(records)
        n = len(sorted_records)
        
        p50 = sorted_records[n // 2]
        p95 = sorted_records[int(n * 0.95)]
        p99 = sorted_records[int(n * 0.99)]
        
        targets = self.latency_targets[task_type]
        
        return {
            "task_type": task_type,
            "p50": p50,
            "p95": p95,
            "p99": p99,
            "p50_target": targets["p50"],
            "p50_compliant": p50 <= targets["p50"],
            "p95_compliant": p95 <= targets["p95"],
            "p99_compliant": p99 <= targets["p99"],
            "overall_compliant": (
                p50 <= targets["p50"]
                and p95 <= targets["p95"]
                and p99 <= targets["p99"]
            ),
        }
```

### 2.3 质量 SLA (Quality)

质量 SLA 是 Agent 系统最独特的挑战——如何衡量和保证非确定性输出的质量？

```python
class QualitySLADefinition:
    """质量 SLA 定义"""
    
    # 不同任务类型的质量指标
    QUALITY_METRICS = {
        "question_answering": {
            "accuracy": 0.90,       # 答案准确性
            "faithfulness": 0.95,   # 忠于检索内容
            "completeness": 0.85,   # 信息完整性
        },
        "code_generation": {
            "syntax_correctness": 0.95,
            "functional_correctness": 0.85,
            "security_scan_pass": 0.99,
        },
        "data_analysis": {
            "calculation_accuracy": 0.98,
            "interpretation_quality": 0.85,
            "visualization_correctness": 0.90,
        },
        "summarization": {
            "relevance": 0.90,
            "conciseness": 0.85,
            "hallucination_free": 0.95,
        },
        "translation": {
            "translation_accuracy": 0.92,
            "fluency": 0.90,
            "terminology_correctness": 0.90,
        },
    }
    
    def __init__(self):
        self.quality_records: list[QualityRecord] = []
        self.evaluator = LLMAsJudge()  # LLM 评审
    
    async def evaluate_quality(
        self,
        task_type: str,
        input_text: str,
        output_text: str,
        context: dict,
    ) -> QualityEvaluation:
        """评估输出质量"""
        
        metrics = self.QUALITY_METRICS.get(task_type, {})
        if not metrics:
            return QualityEvaluation(overall_score=1.0, passed=True)
        
        # 使用 LLM 评审评估各维度
        scores = {}
        for metric, threshold in metrics.items():
            score = await self.evaluator.evaluate(
                metric=metric,
                input=input_text,
                output=output_text,
                context=context,
            )
            scores[metric] = {
                "score": score,
                "threshold": threshold,
                "passed": score >= threshold,
            }
        
        overall = sum(s["score"] for s in scores.values()) / len(scores)
        passed = all(s["passed"] for s in scores.values())
        
        evaluation = QualityEvaluation(
            task_type=task_type,
            scores=scores,
            overall_score=overall,
            passed=passed,
            timestamp=datetime.utcnow(),
        )
        
        self.quality_records.append(evaluation)
        return evaluation
    
    def get_quality_report(self, hours: int = 24) -> dict:
        """生成质量 SLA 报告"""
        cutoff = datetime.utcnow() - timedelta(hours=hours)
        recent = [r for r in self.quality_records if r.timestamp > cutoff]
        
        if not recent:
            return {"status": "no_data"}
        
        # 按任务类型聚合
        by_type = defaultdict(list)
        for r in recent:
            by_type[r.task_type].append(r)
        
        report = {}
        for task_type, records in by_type.items():
            passed = sum(1 for r in records if r.passed)
            report[task_type] = {
                "total": len(records),
                "passed": passed,
                "pass_rate": passed / len(records),
                "avg_score": sum(r.overall_score for r in records) / len(records),
                "compliant": (passed / len(records)) >= 0.95,
            }
        
        return report
```

质量 SLA 的注意点：

```
质量 SLA 的重要约束:

1. 不承诺 100% 正确
   Agent 基于 LLM，LLM 的输出本质上是概率性的
   正确做法: 承诺"在人工审核抽样中，准确率不低于 90%"

2. 区分"可接受"和"完全正确"
   在某些场景下，部分正确的答案也可以接受
   正确做法: 定义"通过"标准，而非"完美"标准

3. 质量是统计承诺，不是单次承诺
   不能保证"每一次回答都正确"
   但可以承诺"过去 30 天，95% 的回答满足质量标准"
```

### 2.4 新鲜度 SLA (Freshness)

Agent 的知识新鲜度是很多企业客户关心的问题——他们需要确保 Agent 使用的是最新的知识。

```python
class FreshnessSLA:
    """知识新鲜度 SLA"""
    
    def __init__(self):
        self.knowledge_sources = {
            "product_docs": {
                "update_frequency": "daily",
                "max_age": "24h",
                "last_updated": None,
            },
            "internal_wiki": {
                "update_frequency": "hourly",
                "max_age": "1h",
                "last_updated": None,
            },
            "customer_data": {
                "update_frequency": "realtime",
                "max_age": "5min",
                "last_updated": None,
            },
            "public_knowledge_base": {
                "update_frequency": "weekly",
                "max_age": "7d",
                "last_updated": None,
            },
        }
    
    async def check_freshness(self) -> FreshnessReport:
        """检查各知识源的新鲜度"""
        
        results = {}
        for source_name, config in self.knowledge_sources.items():
            age = datetime.utcnow() - config["last_updated"]
            max_age = self._parse_duration(config["max_age"])
            
            results[source_name] = {
                "last_updated": config["last_updated"],
                "age": age,
                "max_age": max_age,
                "fresh": age < max_age,
                "overdue_by": age - max_age if age > max_age else None,
            }
        
        return FreshnessReport(
            all_fresh=all(r["fresh"] for r in results.values()),
            source_results=results,
            timestamp=datetime.utcnow(),
        )
    
    def sla_statement(self) -> str:
        """SLA 声明文案"""
        return (
            "我们承诺:\n"
            "1. 产品文档在过去 24 小时内完成索引\n"
            "2. 内部 Wiki 在过去 1 小时内同步\n"
            "3. 客户数据在变更后 5 分钟内生效\n"
            "4. 知识库索引周期不超过 7 天\n\n"
            "Agent 回答会标注知识截止日期，方便您判断信息的时效性。"
        )
```

---

## 3. 错误预算 (Error Budget)

错误预算是 SLA 管理中最实用的工具——它量化了"可接受的不达标"。

### 3.1 四维错误预算

```python
class ErrorBudget:
    """Agent 错误预算"""
    
    def __init__(self):
        self.budgets = {
            "availability": {
                "total": 100,     # 每月 100 基点 (99.9% = 1% 错误预算)
                "remaining": 100,
                "unit": "基点 (0.01%)",
            },
            "latency": {
                "total": 5,       # 每月 5% 的请求可以超时
                "remaining": 5,
                "unit": "百分比",
            },
            "quality": {
                "total": 5,       # 每月 5% 的回答可以不达标
                "remaining": 5,
                "unit": "百分比",
            },
            "cost": {
                "total": 110,     # 每月不超过预算的 110%
                "remaining": 10,
                "unit": "百分比",
            },
        }
    
    def consume(self, dimension: str, amount: float):
        """消耗错误预算"""
        if dimension in self.budgets:
            self.budgets[dimension]["remaining"] -= amount
    
    def is_exhausted(self, dimension: str) -> bool:
        """检查特定维度的错误预算是否耗尽"""
        return self.budgets[dimension]["remaining"] <= 0
    
    def should_block_release(self) -> bool:
        """
        错误预算是否已经耗尽到需要阻止发布?
        
        原则: 如果任一关键维度的预算耗尽，阻止发布
        """
        critical_dimensions = ["availability", "quality"]
        return any(
            self.is_exhausted(d) for d in critical_dimensions
        )
    
    def report(self) -> dict:
        """生成错误预算报告"""
        report = {}
        for dim, budget in self.budgets.items():
            consumed = budget["total"] - budget["remaining"]
            report[dim] = {
                "total": budget["total"],
                "remaining": budget["remaining"],
                "consumed_percent": (consumed / budget["total"]) * 100,
                "status": "ok" if budget["remaining"] > budget["total"] * 0.5
                          else "warning" if budget["remaining"] > 0
                          else "exhausted",
            }
        return report
```

### 3.2 错误预算驱动的发布决策

```
错误预算状态 → 发布决策:

┌─────────────────────────────────────────────────────────────────┐
│  错误预算剩余 > 50% → 正常发布                                   │
│  没有限制，可以按正常节奏发布新功能、新 Prompt                     │
│                                                                 │
│  错误预算剩余 20-50% → 谨慎发布                                 │
│  只发布修复和改进，不发布高风险变更                               │
│  所有发布需要额外的质量验证                                       │
│                                                                 │
│  错误预算剩余 0-20% → 冻结发布                                  │
│  停止所有非关键变更                                              │
│  团队专注于改进 SLA 达标率                                       │
│                                                                 │
│  错误预算耗尽 → 紧急修复模式                                    │
│  回滚最近的可疑变更                                              │
│  启动根因分析                                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. SLA 监测与报告

### 4.1 实时 SLA 仪表盘

```python
class SLADashboard:
    """SLA 仪表盘数据聚合"""
    
    async def get_dashboard_data(self) -> dict:
        """生成 SLA 仪表盘数据"""
        
        now = datetime.utcnow()
        window_24h = now - timedelta(hours=24)
        window_30d = now - timedelta(days=30)
        
        return {
            "current_status": await self._get_current_status(),
            "24h_compliance": {
                "availability": await self._calc_availability(window_24h),
                "latency": await self._calc_latency_compliance(window_24h),
                "quality": await self._calc_quality_compliance(window_24h),
                "freshness": await self._check_freshness(),
            },
            "30d_trend": {
                "daily_availability": await self._daily_metric("availability", window_30d),
                "daily_latency_p95": await self._daily_metric("latency_p95", window_30d),
                "daily_quality_score": await self._daily_metric("quality_score", window_30d),
            },
            "error_budget_remaining": self.error_budget.report(),
            "incidents": await self._get_recent_incidents(24),
        }
    
    def visualize_sla_dashboard(self) -> str:
        """ASCII SLA 仪表盘（用于内部展示）"""
        status = self.get_dashboard_data()  # 同步简化版本
        
        return f"""
╔════════════════════════════════════════════════════════╗
║              Agent SLA 仪表盘 (24h)                     ║
╠════════════════════════════════════════════════════════╣
║  可用性:  ████████████░░░  92.3%  目标: 99.9%           ║
║  延迟:    ██████████████░  97.1%  目标: 95% 达标        ║
║  质量:    ███████████████  99.2%  目标: 95% 达标        ║
║  新鲜度:  ██████████████░  96.5%  目标: 100% 新鲜       ║
║                                                         ║
║  错误预算:                                            ║
║    可用性: ████████░░░░░░  42% 剩余  ■■■■ 警告          ║
║    质量:   ██████████████  82% 剩余  ■■■■ 正常          ║
║                                                         ║
║  最近事件:                                            ║
║    ✓ 02:15 LLM API 抖动 (2分钟, 自动恢复)               ║
║    ✓ 昨日质量审计: 98.3% 通过率                         ║
║    ⚠ 知识库索引延迟: 排队中 (预计 15 分钟)              ║
╚════════════════════════════════════════════════════════╝
        """
```

### 4.2 SLA 报告周期

```
实时告警:
  立即通知: 可用性下降、错误率飙升、质量严重下滑
  渠道: PagerDuty / Slack / 短信

日报:
  覆盖: 昨日 SLA 各维度达标情况
  受众: 工程团队
  形式: 自动生成的 Slack 摘要

周报:
  覆盖: 当周趋势、错误预算消耗、主要事件
  受众: 工程经理 + 产品经理
  形式: 邮件 / 共享文档

月报:
  覆盖: 当月 SLA 总达成率、未达标分析、改进计划
  受众: 管理层 + 客户（可选）
  形式: 正式报告 + 客户会议

季度回顾:
  覆盖: SLA 目标校准、新维度引入、技术改进路线图
  受众: 工程 + 产品 + 管理层
  形式: 会议 + 文档
```

---

## 5. 降级与补偿策略

### 5.1 SLA 违约补偿

```python
class SLACompensation:
    """SLA 违约补偿计算"""
    
    COMPENSATION_TABLE = {
        "availability": {
            "< 99.9%": "当月费用 10% 减免",
            "< 99.5%": "当月费用 25% 减免",
            "< 99.0%": "当月费用 50% 减免",
            "< 95.0%": "当月费用 100% 减免",
        },
        "quality": {
            "连续 7 天低于 90%": "当月费用 10% 减免",
            "连续 30 天低于 90%": "当月费用 25% 减免",
        },
    }
    
    def calculate_compensation(
        self, monthly_fee: float, sla_violations: list
    ) -> float:
        """计算当月应减免金额"""
        
        max_credit = 0.0
        for violation in sla_violations:
            if violation["dimension"] == "availability":
                actual = violation["actual"]
                if actual < 95.0:
                    credit = monthly_fee
                elif actual < 99.0:
                    credit = monthly_fee * 0.5
                elif actual < 99.5:
                    credit = monthly_fee * 0.25
                elif actual < 99.9:
                    credit = monthly_fee * 0.1
                else:
                    credit = 0
                max_credit = max(max_credit, credit)
        
        # 通常补偿不超过当月费用的 100%
        return min(max_credit, monthly_fee)
```

### 5.2 降级策略

当 SLA 无法达成时，Agent 系统应该优雅降级：

```python
class SLADegradationStrategy:
    """SLA 驱动的降级策略"""
    
    DEGRADATION_LEVELS = {
        "full": {
            "description": "全功能",
            "llm_model": "gpt-4",
            "max_tools": 20,
            "max_steps": 50,
        },
        "reduced": {
            "description": "降级——切换高效模型",
            "llm_model": "gpt-4o-mini",
            "max_tools": 10,
            "max_steps": 20,
        },
        "minimal": {
            "description": "最低——仅基于知识库回答",
            "llm_model": "gpt-4o-mini",
            "max_tools": 3,
            "max_steps": 5,
        },
        "fallback": {
            "description": "兜底——返回预设答案",
            "llm_model": None,  # 不使用 LLM
            "max_tools": 0,
            "max_steps": 1,
        },
    }
    
    def get_degradation_level(
        self, current_sla: dict
    ) -> str:
        """根据当前 SLA 状态决定降级级别"""
        
        # 可用性严重不足
        if current_sla.get("availability", 100) < 95:
            return "minimal"
        
        # 延迟严重超标
        if current_sla.get("latency_p95", 0) > 60:
            return "reduced"
        
        # 质量显著下降
        if current_sla.get("quality_score", 100) < 80:
            return "reduced"
        
        # 错误预算关键维度耗尽
        if current_sla.get("error_budget_exhausted"):
            return "minimal"
        
        return "full"
```

---

## 6. 能力边界

### 6.1 能做到的

- **量化承诺**：为可用性、延迟、质量、新鲜度定义可衡量的 SLA
- **错误预算管理**：用量化的错误预算指导发布决策
- **自动降级**：SLA 不达标时自动降级，保护核心功能
- **透明度报告**：定期发布 SLA 达成情况
- **补偿机制**：SLA 违约的自动补偿计算

### 6.2 做不到的

- **保证 100% 回答质量**：LLM 的概率性本质使绝对保证不可行
- **对 LLM API 的完全控制**：第三方 API 的可用性不在控制范围内
- **消除所有延迟波动**：LLM 推理延迟受到模型大小、请求负载等因素影响
- **一次性完美 SLA**：SLA 需要基于实际运行数据持续校准

---

## 7. 核心优势

- **明确预期**：客户和工程团队对系统性能有共同的理解
- **量化决策**：错误预算消除了"感觉上可用"的模糊判断
- **系统化改进**：SLA 数据驱动持续的架构优化
- **商业保护**：明确的违约补偿保护客户权益

---

## 8. 常见失败模式

| 失败模式 | 表现 | 根因 | 解决方案 |
|----------|------|------|----------|
| SLA 过度承诺 | 承诺 99.99% 可用性但 LLM API 不可控 | 未区分自有组件和第三方依赖 | 分别定义自有和依赖 SLA |
| 质量 SLA 模糊 | "回答准确"无法量化 | 未定义质量标准 | 基于 LLM 评审的量化评估 |
| 错误预算不用 | 预算耗尽仍然发布 | 未建立发布门禁 | 自动化发布检查 |
| 延迟 SLA 一刀切 | 所有任务用相同的延迟目标 | 未区分任务复杂度 | 分层延迟目标 |
| SLA 不公开 | 只有工程团队知道 | 缺乏透明度文化 | 定期发布 SLA 报告 |

---

> **核心结论**：Agent 系统的 SLA 不是传统基础设施 SLA 的简单扩展，而是需要全新的四维框架（可用性 + 延迟 + 质量 + 新鲜度）。关键在于：不为概率性输出承诺绝对正确，而是建立统计性、可衡量、持续改进的质量承诺。错误预算是连接 SLA 和工程决策的最佳工具——它让"系统有多可靠"从模糊感知变成可量化的管理指标。
