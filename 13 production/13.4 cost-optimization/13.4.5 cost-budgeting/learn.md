# 13.4.5 成本预算与配额管理

**LLM Agent 系统的成本预算与配额管理是一套将不可预测的模型调用开销转化为可规划、可控制、可追溯的治理体系，是生产级 Agent 系统区别于实验原型的关键基础设施。**

---

## 1. 基本原理与重要性

### 1.1 为什么传统预算模型失效

传统云服务成本（服务器、数据库、带宽）具有**确定性** —— 你为固定资源付费，账单可精确预估。LLM API 的成本模型则完全不同：

- **按 Token 计费**：成本 = 输入 Token 数 × 单价 + 输出 Token 数 × 单价，每一轮对话的真实成本在结束前不可知
- **Agent 自循环风险**：一个存在 bug 的 ReAct Agent 可能陷入无限循环，每分钟消耗数百美元
- **不可控的上下文窗口**：长文档处理、多轮复杂推理能一次性消耗数万 Token

业界已有真实案例：某团队在生产环境部署了一个未设预算约束的客服 Agent，因 Agent 在工具调用链中陷入死循环，一小时内产生了 **12,000 美元的 API 费用**，且没有任何告警通知，直到月末账单出现才发现。

### 1.2 预算系统的三大支柱

```
┌─────────────────────────────────────────────────────┐
│                 成本预算系统                          │
├─────────────┬─────────────────┬─────────────────────┤
│  治理       │     护栏        │     可观测性        │
│ (Governance)│  (Guardrails)   │  (Observability)    │
├─────────────┼─────────────────┼─────────────────────┤
│ 谁可以花    │ 硬上限阻断      │ 实时成本追踪        │
│ 花多少      │ 软上限告警      │ 成本归因分析        │
│ 花在哪      │ 自动降级        │ 预算使用趋势        │
└─────────────┴─────────────────┴─────────────────────┘
```

没有预算系统的 Agent 系统：无法回答"花了多少钱"、"谁花的"、"花得值不值"三个基本问题，也无法防范成本失控风险。

---

## 2. 预算体系架构

### 2.1 整体架构

```
                         ┌─────────────────────────┐
                         │      Budget Dashboard     │
                         │  (成本看板与告警界面)     │
                         └───────────┬─────────────┘
                                     │
            ┌────────────────────────┼────────────────────────┐
            │                        │                        │
            ▼                        ▼                        ▼
   ┌────────────────┐   ┌────────────────────┐   ┌────────────────┐
   │   Cost Tracker  │   │   Budget Enforcer   │   │  Quota Manager │
   │  (实时 Token 计数) │◄──┤   (策略引擎)        ├──►│  (用户级配额)   │
   │  写 Redis/ES   │   │ 硬/软限判断         │   │  Redis 计数器  │
   └───────┬────────┘   └──────────┬─────────┘   └───────┬────────┘
           │                      │                       │
           ▼                      ▼                       ▼
   ┌────────────────────────────────────────────────────────────┐
   │                   LLM API Gateway 层                        │
   │  (拦截所有 LLM 调用 → 注入上下文 → 路由 → 记录日志)        │
   └────────────────────────────────────────────────────────────┘
```

### 2.2 分层预算模型

Agent 系统的预算需要从多个维度分层设置，每个维度承担不同的治理职责：

| 预算层级 | 范围 | 示例 | 用途 |
|---------|------|------|------|
| **全局预算 (Global)** | 整个系统 | 月上限 $5,000 | 防止总成本失控 |
| **租户预算 (Per-Tenant)** | 单个客户 | 每客户月上限 $500 | SaaS 公平使用 |
| **用户预算 (Per-User)** | 单个用户 | 每日上限 $10 | 防用户滥用 |
| **功能预算 (Per-Feature)** | 特定功能 | 文件分析功能月上限 $1,000 | 功能级成本优化 |
| **会话预算 (Per-Session)** | 单次对话 | 单次会话上限 $2 | 防 Agent 死循环 |

---

## 3. 核心技术与实现

### 3.1 Token 成本计数器中间件

所有 LLM 调用必须经过一个统一的中间件层，该层负责实时记录 Token 消耗并将其上报给成本追踪系统。

```python
import time
import logging
from dataclasses import dataclass, field
from typing import Optional

# ── 成本模型配置 ──────────────────────────────────────────
MODEL_PRICING = {
    "gpt-4o":          {"input": 5.00 / 1_000_000, "output": 15.00 / 1_000_000},
    "gpt-4o-mini":     {"input": 0.15 / 1_000_000, "output": 0.60 / 1_000_000},
    "claude-sonnet-4": {"input": 3.00 / 1_000_000, "output": 15.00 / 1_000_000},
    "claude-haiku-3":  {"input": 0.25 / 1_000_000, "output": 1.25 / 1_000_000},
}


@dataclass
class UsageRecord:
    """单次 LLM 调用的用量记录"""
    user_id: str
    feature_id: str
    session_id: str
    model: str
    input_tokens: int
    output_tokens: int
    timestamp: float = field(default_factory=time.time)

    @property
    def cost(self) -> float:
        prices = MODEL_PRICING.get(self.model, MODEL_PRICING["gpt-4o"])
        return self.input_tokens * prices["input"] + self.output_tokens * prices["output"]


class CostTracker:
    """成本追踪器 —— 拦截所有 LLM 调用并记录消耗"""

    def __init__(self, storage_backend):
        self.storage = storage_backend  # Redis / Elasticsearch / Prometheus

    def wrap_llm_call(self, func, user_id: str, feature_id: str, model: str):
        """装饰器模式：包裹 LLM 调用，自动记录用量"""

        def wrapper(messages, **kwargs):
            # 调用前记录输入 Token 数
            input_tokens = self._count_tokens(messages)

            start = time.monotonic()
            response = func(messages, **kwargs)
            elapsed = time.monotonic() - start

            # 从响应中提取用量
            output_tokens = self._extract_output_tokens(response)

            record = UsageRecord(
                user_id=user_id,
                feature_id=feature_id,
                session_id=kwargs.get("session_id", "unknown"),
                model=model,
                input_tokens=input_tokens,
                output_tokens=output_tokens,
            )
            self.storage.store(record)

            logger.info(
                "LLM call | user=%s | feature=%s | model=%s | cost=$%.4f | latency=%.2fs",
                user_id, feature_id, model, record.cost, elapsed,
            )
            return response

        return wrapper

    def _count_tokens(self, messages) -> int:
        # 实际项目中应使用 tiktoken 或 Anthropic Tokenizer
        return sum(len(m["content"]) for m in messages)

    def _extract_output_tokens(self, response) -> int:
        # 从 API 响应中提取 usage 字段
        return getattr(response, "usage", {}).get("output_tokens", 0)


logger = logging.getLogger(__name__)
```

### 3.2 预算执行器（Budget Enforcer）

预算执行器在每次 LLM 调用前检查当前消耗，判断是否允许继续调用。

```python
from enum import Enum, auto
from datetime import datetime, timedelta


class BudgetLevel(Enum):
    OK = auto()
    SOFT_LIMIT_WARN = auto()   # 已超过软上限，发出告警
    HARD_LIMIT_BLOCK = auto()  # 已超过硬上限，阻断调用


class BudgetEnforcer:
    """预算执行器 —— 在每次 API 调用前判断是否允许执行"""

    def __init__(self, cost_tracker: CostTracker, redis_client):
        self.tracker = cost_tracker
        self.redis = redis_client
        # 预算配置（实际应从配置中心读取）
        self.limits = {
            "global_daily":   {"soft": 150.0, "hard": 200.0},
            "global_monthly": {"soft": 4000.0, "hard": 5000.0},
            "per_user_daily": {"soft": 8.0, "hard": 10.0},
        }

    def check(self, user_id: str, estimated_cost: float = 0) -> BudgetLevel:
        """在调用前检查预算状态"""
        now = datetime.utcnow()
        today_key = f"cost:daily:{now.strftime('%Y%m%d')}"
        month_key = f"cost:monthly:{now.strftime('%Y%m')}"
        user_key  = f"cost:user:{user_id}:daily:{now.strftime('%Y%m%d')}"

        # 读取当前消耗（累加值，由 CostTracker 异步写入）
        global_daily  = float(self.redis.get(today_key) or 0)
        global_month  = float(self.redis.get(month_key) or 0)
        user_daily    = float(self.redis.get(user_key) or 0)

        projected = global_daily + estimated_cost

        # 优先级：硬限 > 软限
        if projected >= self.limits["global_daily"]["hard"]:
            self._alert("HARD_LIMIT", f"Global daily budget exceeded: ${projected:.2f}")
            return BudgetLevel.HARD_LIMIT_BLOCK

        if user_daily >= self.limits["per_user_daily"]["hard"]:
            return BudgetLevel.HARD_LIMIT_BLOCK

        if projected >= self.limits["global_daily"]["soft"]:
            self._alert("SOFT_LIMIT", f"Global daily soft limit reached: ${projected:.2f}")
            return BudgetLevel.SOFT_LIMIT_WARN

        if global_month >= self.limits["global_monthly"]["hard"]:
            return BudgetLevel.HARD_LIMIT_BLOCK

        return BudgetLevel.OK

    def _alert(self, level: str, message: str):
        """触发告警（发送到 PagerDuty / Slack / 邮件）"""
        logger.warning("[%s] %s", level, message)
        # 实际实现：发送 webhook 到告警通道
```

### 3.3 告警阈值策略

预算消耗是渐进式的，不同阶段的告警应有不同的响应动作：

```
预算使用率 0% ────────────────────────────────────────── 100%
            │      │      │      │      │              │
            │  50% │  80% │  90% │  95% │    100%      │
            │  INFO│ WARN │ERROR │CRITICAL│  BLOCK      │
            ▼      ▼      ▼      ▼      ▼              ▼
动作:   记录日志 Slack通知 钉钉告警 电话告警 阻断调用+建incident
模型:  gpt-4o   gpt-4o  gpt-4o-mini claude-haiku 降级为缓存回复
```

---

## 4. 配额与限流策略

### 4.1 用户级配额管理（基于 Redis）

```python
import time


class QuotaManager:
    """基于 Redis 滑动窗口的用户配额管理器"""

    def __init__(self, redis_client):
        self.redis = redis_client

    # ── 配额配置 ───────────────────────────────────
    # 免费用户：每日 100K 输入 Token
    # Pro 用户：每日 1M 输入 Token
    # 企业用户：每日 10M 输入 Token
    TIER_LIMITS = {
        "free":  {"daily_input_tokens": 100_000,  "daily_requests": 100},
        "pro":   {"daily_input_tokens": 1_000_000, "daily_requests": 1_000},
        "enterprise": {"daily_input_tokens": 10_000_000, "daily_requests": 10_000},
    }

    def check_quota(self, user_id: str, tier: str,
                    input_tokens: int) -> tuple[bool, dict]:
        """检查用户配额是否足够，返回 (允许, 当前用量信息)"""
        limits = self.TIER_LIMITS.get(tier, self.TIER_LIMITS["free"])
        today = time.strftime("%Y%m%d")
        token_key = f"quota:tokens:{user_id}:{today}"
        req_key   = f"quota:requests:{user_id}:{today}"

        # 原子递增并获取当前值
        token_used = self.redis.incrby(token_key, input_tokens)
        req_used   = self.redis.incrby(req_key, 1)

        # 首次写入时设置过期时间（次日凌晨过期）
        if token_used == input_tokens:
            self.redis.expireat(token_key, self._next_midnight())
        if req_used == 1:
            self.redis.expireat(req_key, self._next_midnight())

        allowed = (token_used <= limits["daily_input_tokens"]
                   and req_used <= limits["daily_requests"])

        return allowed, {
            "token_used": token_used,
            "token_limit": limits["daily_input_tokens"],
            "requests_used": req_used,
            "requests_limit": limits["daily_requests"],
        }

    def _next_midnight(self) -> int:
        now = time.time()
        return int(((now // 86400) + 1) * 86400)
```

### 4.2 成本感知的限流策略

传统的请求频率限流（RPM/TPM）无法应对成本波动。成本感知限流的核心思想是：**当预算消耗速度异常时，主动降低服务质量以控制成本。**

```
┌──────────────────────────────────────────────┐
│              LLM Router                       │
│                                              │
│   Budget OK ──────► gpt-4o (高质量模型)      │
│                                              │
│   Soft Limit ─────► gpt-4o-mini (降级)       │
│                                              │
│   Hard Limit ─────► cache-only (只回复缓存)   │
│                                              │
│   Quota Exceeded ─► 返回 429 + Retry-After  │
└──────────────────────────────────────────────┘
```

实现要点：

- **主动降级（Proactive Throttling）**：在预算达到 80% 时自动将非关键路径切换到低成本模型，而非等到 100% 时暴力阻断
- **反应式限流（Reactive Throttling）**：监控每秒消耗速率，当速率超过基线 3 倍时立即降级
- **优雅降级（Graceful Degradation）**：从不直接返回 500 错误 —— 降级到便宜模型、使用缓存回复、或提示用户"当前服务繁忙，请稍后再试"

---

## 5. 成本归因与分摊

### 5.1 请求标签体系

每次 LLM 调用必须携带可归因的标签，这是成本分析的基础：

```python
@dataclass
class CostTags:
    """每次 LLM 请求的归因标签"""
    user_id: str          # 用户标识
    tenant_id: str        # 租户标识（SaaS 场景）
    feature_id: str       # 功能标识（chat / code / analysis / search）
    session_id: str       # 会话标识
    model: str            # 模型名称
    tool_name: str        # 调用方工具名称（若通过 tool calling）
    environment: str      # production / staging / development
    cost_center: str      # 成本中心（财务分摊用）
```

所有标签存储在日志系统或时序数据库中，支持按任意维度聚合查询。

### 5.2 Showback 与 Chargeback

| 模型 | 描述 | 适用场景 |
|------|------|---------|
| **Showback** | 展示成本但不强制收费 | 内部团队、开发环境 |
| **Chargeback** | 将成本计入对应部门预算 | 对外 SaaS、客户计费 |

### 5.3 成本分析维度

```sql
-- 以 ClickHouse 为例的成本查询
-- 按功能维度聚合当日成本
SELECT
    feature_id,
    sum(input_tokens + output_tokens) AS total_tokens,
    sum(cost) AS total_cost,
    count() AS request_count
FROM llm_cost_logs
WHERE timestamp >= now() - INTERVAL 1 DAY
GROUP BY feature_id
ORDER BY total_cost DESC;
```

```
示例输出：
┌─feature_id─┬─total_tokens─┬─total_cost─┬─request_count─┐
│ chat       │   12,345,678 │    $2,345  │       123,456 │
│ code       │    8,234,567 │    $1,890  │        45,678 │
│ analyze    │    3,456,789 │      $678  │        12,345 │
│ search     │      567,890 │       $89  │        89,012 │
└────────────┴──────────────┴────────────┴───────────────┘
```

---

## 6. 工程实践

### 6.1 生产环境清单

在将 Agent 系统部署到生产环境前，必须逐项检查以下内容：

- [ ] 全局硬预算上限已配置（建议月上限 + 日上限双层保护）
- [ ] 软预算阈值告警已接入 Slack / 钉钉 / PagerDuty
- [ ] 每个用户/用户的配额已根据 Tier 正确配置
- [ ] 降级策略已实现并测试（优雅降级到便宜模型）
- [ ] Agent 循环检测机制已启用（单会话最大调用次数限制）
- [ ] 成本监控仪表盘已搭建（每 5 分钟刷新一次）
- [ ] 异常检测规则已建立（小时成本突增 50% 自动告警）

### 6.2 常见陷阱与应对

| 陷阱 | 表现 | 解决方案 |
|------|------|---------|
| **批量处理绕过预算** | 批量任务成本被低估 | 批量任务单独计算并预审批额度 |
| **多模型切换成本失控** | 1% 切换触发概率导致不可预测成本 | 成本估算时取加权平均值并加安全边际 |
| **缓存穿透** | 高频请求绕过缓存直接调用模型 | 设置二级缓存 + 熔断机制 |
| **归因丢失** | 异步任务中无法追踪用户 | 强制上下文传播：使用 OpenTelemetry 传递 span context |
| **账单延迟** | 云厂商账单滞后 24 小时 | 本地实时估算 + 厂商账单核对（双轨制） |

### 6.3 定期预算回顾流程

```
每周回顾 ──→ 对比实际消耗 vs 预算基线
    │
    ▼
分析偏差 ──→ 是否由新功能/用户增长/异常导致？
    │
    ▼
调整预算 ──→ 更新预算分配或降级策略
    │
    ▼
评估 ROI ──→ 成本增量是否带来了对应的业务价值？
    │
    ▼
优化行动 ──→ 是否可以通过 prompt 压缩、缓存、模型蒸馏降本？
```

---

## 总结

成本预算与配额管理不是限制 Agent 能力的手段，而是让 Agent 系统**可持续运行**的基础设施。一个设计良好的预算系统应当做到：

1. **看得见** —— 实时掌握每一分钱花在哪里
2. **管得住** —— 在成本失控前自动干预
3. **算得清** —— 将成本精确归因到用户、功能和团队
4. **长得大** —— 业务增长时成本增长可预期、可控制

记住：**没有预算约束的 Agent 系统，不是生产系统，而是一个实验。**
