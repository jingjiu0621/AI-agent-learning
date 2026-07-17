# 12.4.4 approval-workflow — 审批流程

## 简单介绍

Approval workflow 是 AI Agent 安全体系中的关键环节，它引入人类介入（Human-in-the-Loop, HITL）作为高风险操作的"安全阀"。当 Agent 即将执行可能造成重大影响的动作（如删除资源、修改生产配置、调用付费 API、访问敏感数据）时，审批流程要求先获得人类授权再执行。这套机制在 Agent 的自主性与人类控制之间建立了一道可控的屏障。

## 基本原理 — HITL for high-risk actions, approval as "safety valve"

核心设计原则是：**Agent 默认拥有低风险操作的完整自主权，但涉及高风险的转向必须暂停并等待人类裁决。** 这类似电路中的保险丝——正常情况下电流自由流动，但异常时保险丝熔断切断电路。

```
Agent 自主执行 ──→ 风险检测 ──→ [安全] ──→ 继续执行
                          │
                    [高风险] ──→ 审批请求 ──→ 人类裁决 ──→ [批准] ──→ 执行
                                                      └── [拒绝] ──→ 中止 + 记录
```

概念基础：

- **HITL**：人类作为决策环路中的必要节点，而非旁观者。Agent 不能绕过人类做出关键决策。
- **最小权限原则**：即使 Agent 拥有执行某操作的凭证，审批流程在运行时再次验证 "本次操作是否真的被允许"。
- **审计线索**：每一次审批请求、决策和执行结果都被记录，形成完整的可追溯链条。
- **安全阀隐喻**：审批不是常态，而是异常保护机制。设计目标是在"从不打扰用户"和"每次都要批准"之间找到平衡点。

## 背景 — from "always approve" → "risk-based approve" → "adaptive approval"

审批流程的设计经历了三个阶段的演进：

### Phase 1: "Always Approve"（早期 Agent 系统）

最初的 Agent 系统对所有外部操作都要求人类确认，以保证安全。这种模式简单直接，但带来了严重的可用性问题——用户被频繁的确认对话框打断，体验破碎，效率低下。

```
执行每个操作前: "确认执行 X？" → 用户点击确认 → 执行操作
             → "确认执行 Y？" → 用户点击确认 → 执行操作
             → "确认执行 Z？" → ...

结果: 用户疲劳、Agent 失去"代理"意义
```

### Phase 2: "Risk-Based Approve"（当前主流设计）

系统引入风险评分机制，根据操作类型、目标资源、数据敏感度等维度自动评估风险等级。

```
低风险操作 (auto-allow):  Agent 自主执行，无需人类介入
中风险操作 (notify/confirm): Agent 执行但通知用户，或等待简单确认
高风险操作 (approve):      暂停执行，等待人类审批
极高风险操作 (multi-approve): 需要多人审批
```

### Phase 3: "Adaptive Approval"（未来方向）

系统学习用户的行为模式，自适应调整审批策略：

- 用户一贯批准某类操作 → 降级为 notify-only
- 用户某时间段内频繁拒绝某类操作 → 升级为 multi-approve
- 根据历史决策相似性自动处理（案例推理）
- 上下文感知：同样是删除操作，删除沙箱环境 vs 生产环境的触发阈值不同

## 核心矛盾 — speed vs safety, user fatigue from too many approval requests

### 主要矛盾

| 矛盾 | 描述 |
|------|------|
| 速度 vs 安全 | 审批等待期可能长达数分钟到数小时，拖慢 Agent 任务完成速度；但跳过审批可能带来安全事故 |
| 用户疲劳 | 过多审批请求导致用户习惯性点击"批准"，使审批流程丧失实际保护意义（类似"验证码疲劳"） |
| 可用性 | 如果用户不在设备前（会议中、通勤中），Agent 任务完全卡死，影响用户体验 |
| 误判成本 | 风险评分过高 → 增加用户负担；风险评分过低 → 安全漏洞 |

### 负面影响曲线

```
用户满意度
    ↑
 高  ┊        ╱╲
     ┊      ╱    ╲
 中  ┊    ╱        ╲________ 安全漏洞区
     ┊  ╱
 低  ┊╱
    └────────────────────────→ 审批请求频率
       低频(安全)      高频(疲劳)
```

### 缓解策略

- **智能批处理**：将多个相似操作合并为一个审批请求
- **风险自适应**：根据历史行为动态调整触发阈值
- **上下文总结**：每次审批请求附带简洁的上下文摘要，降低用户决策成本
- **降级机制**：审批超时时提供默认安全策略（通常是拒绝），而非死锁

## 详细内容

### 1. Approval Level Design: 审批等级设计

审批等级是一个从"完全自主"到"多人批准"的连续谱系。

#### 等级模型

```
Level 0: AUTO_ALLOW
  描述：Agent 完全自主执行，无需通知人类
  示例：读取公开数据、生成内部报告、格式化输出
  特点：零延迟，最高效率

Level 1: NOTIFY_ONLY
  描述：Agent 执行操作的同时通知人类（事后通知）
  示例：发送消息到聊天频道、创建低优先级工单
  特点：用户知情但无需响应，适合可逆操作

Level 2: CONFIRM
  描述：Agent 执行前需要人类简单确认（单击确认）
  示例：删除非关键缓存、重启开发服务器
  特点：低延迟，用户只需"OK"，无需填写理由

Level 3: APPROVE
  描述：Agent 暂停执行，等待人类明确批准（带审批理由）
  示例：删除生产数据库记录、修改 IAM 策略、调用付费 API
  特点：需要用户查看上下文并做出决策，通常记录审批理由

Level 4: MULTI_PARTY_APPROVE
  描述：需要 N 个审批人中至少 M 人批准（如 2/3 多数）
  示例：修改金融交易金额、发布重大产品变更、删除用户数据
  特点：防止单点决策失误，通常需要多人协作
```

#### 实现示例

```python
from enum import Enum, auto

class ApprovalLevel(Enum):
    AUTO_ALLOW = auto()      # Level 0
    NOTIFY_ONLY = auto()     # Level 1
    CONFIRM = auto()         # Level 2
    APPROVE = auto()         # Level 3
    MULTI_PARTY_APPROVE = auto()  # Level 4

    @property
    def requires_human(self) -> bool:
        return self.value >= self.APPROVE.value

    @property
    def blocks_execution(self) -> bool:
        return self.value >= self.CONFIRM.value
```

---

### 2. Risk-Based Triggering: 风险触发的审批机制

审批不是对所有操作一视同仁，而是根据风险评估结果决定是否需要审批以及需要什么等级的审批。

#### 触发维度

| 维度 | 因素 | 示例 |
|------|------|------|
| **工具类型** | 工具本身的危险等级 | `rm -rf` > `ls`，`DELETE /api/users` > `GET /api/users` |
| **操作参数** | 参数的具体数值 | `limit=10000` > `limit=10`，`DROP TABLE` > `SELECT` |
| **数据敏感度** | 被操作数据的分类等级 | 个人身份信息 > 匿名数据，生产数据 > 测试数据 |
| **上下文** | 操作发生的环境 | 生产环境 > 开发环境，业务高峰期 > 低峰期 |
| **频率** | 操作在单位时间内的发生次数 | 批量操作 > 单次操作，重复执行 > 首次执行 |
| **可逆性** | 操作是否可撤销 | 不可逆操作（删除）> 可逆操作（创建草稿）|
| **影响范围** | 操作影响的资源数量 | 全量更新 > 单条更新，跨部门 > 单部门 |

#### Risk Scoring Model

```
risk_score = (
    tool_risk_weight * 0.30 +
    param_risk_weight * 0.25 +
    sensitivity_weight * 0.25 +
    context_weight * 0.10 +
    frequency_weight * 0.05 +
    reversibility_weight * 0.05
)
```

阈值映射到审批等级：

| Risk Score | Approval Level |
|------------|----------------|
| 0 - 0.2    | AUTO_ALLOW     |
| 0.2 - 0.4  | NOTIFY_ONLY    |
| 0.4 - 0.6  | CONFIRM        |
| 0.6 - 0.8  | APPROVE        |
| 0.8 - 1.0  | MULTI_PARTY_APPROVE |

#### 动态阈值（Advanced）

生产系统通常采用动态阈值而非固定阈值：

```python
class DynamicThresholdAdjuster:
    def __init__(self, base_thresholds: dict):
        self.thresholds = base_thresholds
        self.history = []

    def adjust_for_user_fatigue(self, recent_approval_rate: float):
        """如果用户近期拒绝率过高，可能表明阈值过低"""
        if recent_approval_rate < 0.3:
            # 用户频繁拒绝 → 可能是过多低价值审批请求
            self.thresholds["approve"] += 0.05
        elif recent_approval_rate > 0.95:
            # 用户几乎全部批准 → 审批疲劳，阈值需要调整
            self.thresholds["approve"] += 0.10

    def adjust_for_time_of_day(self, hour: int):
        """非工作时间提高审批阈值"""
        if hour < 8 or hour > 18:
            self.thresholds["approve"] += 0.10

    def adjust_for_recent_incidents(self, recent_incident_count: int):
        """近期安全事故后降低审批阈值"""
        if recent_incident_count > 0:
            self.thresholds["approve"] -= 0.10 * recent_incident_count
```

---

### 3. Approval UI/UX: 多通道审批界面

审批请求需要以最低认知负担的方式送达用户，并允许用户快速做出决策。

#### 审批渠道

| 通道 | 适用场景 | 延迟 | 表达能力 |
|------|----------|------|----------|
| **In-App Toast/Notification** | Agent 在用户同界面操作时 | 实时 | 低（仅摘要） |
| **桌面推送** | Agent 后台运行，用户可能在其他应用 | 秒级 | 中 |
| **Email** | 用户不在计算机前 | 分钟级 | 高（可附带完整上下文） |
| **移动推送** | 用户离开工位 | 秒-分级 | 低-中 |
| **Slack/Teams Bot** | 团队协作场景 | 秒级 | 中-高 |
| **API Callback** | 自动化审批系统集成 | 可配置 | 高 |

#### 审批请求消息设计

每条审批请求应包含以下结构化信息：

```
[审批请求 #req_20240717_001] ──────────────────────
  操作: 删除 S3 存储桶
  目标: production-app-logs (us-east-1)
  发起: Agent "数据清理助手" (session #sess_abc123)
  风险评分: 0.82 (HIGH)
  触发规则: tool=s3.delete_bucket, env=production
  上下文: 存储桶已经过 90 天生命周期规则清理，确认无剩余对象
  影响: 不可逆操作，但已确认数据已备份至 archive-bucket
  ┌──────────────────────────────┐
  │   [批准]  [拒绝并给出理由]   │
  └──────────────────────────────┘
  超时: 15 分钟内未响应将自动拒绝
```

#### 设计原则

1. **上下文充分但简洁**：用户能在 5 秒内理解"发生了什么"和"如果拒绝会怎样"
2. **一键决策**：批准和拒绝都应尽可能减少点击次数
3. **拒绝理由可选但鼓励**：对后续分析有价值，但不应该成为用户拒绝的障碍
4. **紧迫感提示**：显示超时时间，帮助用户确定优先级
5. **链接到详细信息**：如果用户需要了解更多，提供指向详细日志的链接

---

### 4. Timeout Handling: 超时处理

审批超时是必须处理的核心场景——用户可能不在、忽略通知、或者审批系统故障。

#### 超时策略

```python
class TimeoutStrategy(Enum):
    DEFAULT_DENY = "default_deny"      # 默认拒绝（安全优先）
    DEFAULT_ALLOW = "default_allow"     # 默认允许（效率优先）
    ESCALATE = "escalate"              # 升级到更高级审批人
    DEFER = "defer"                    # 推迟到后续审查批次
    RETRY = "retry"                    # 重试通知（限 N 次）

class TimeoutConfig:
    def __init__(self):
        self.default_strategy = TimeoutStrategy.DEFAULT_DENY
        self.timeout_by_level = {
            ApprovalLevel.CONFIRM: 300,          # 5 分钟
            ApprovalLevel.APPROVE: 900,          # 15 分钟
            ApprovalLevel.MULTI_PARTY_APPROVE: 3600,  # 1 小时
        }
        self.escalation_chain = ["user", "manager", "oncall"]
        self.max_retries = 3
        self.retry_interval = 60  # 秒

    def handle_timeout(self, request: ApprovalRequest):
        if self.default_strategy == TimeoutStrategy.DEFAULT_DENY:
            request.deny(reason="审批超时（默认拒绝）")
            self._rollback_preconditions(request)

        elif self.default_strategy == TimeoutStrategy.ESCALATE:
            next_approver = self._get_next_approver(request)
            request.assign_to(next_approver)
            request.notify()

        elif self.default_strategy == TimeoutStrategy.DEFER:
            request.status = "deferred"
            request.defer_reason = "审批超时，已加入待处理队列"
```

#### Graceful Degradation 流程

```
审批请求发出
    │
    ├── 用户在时间内响应 → 按批准/拒绝处理
    │
    └── 超时未响应
         │
         ├── [DEFAULT_DENY] 执行拒绝 + 回滚任何前置操作
         │   └── 通知用户"因超时已自动拒绝操作 #req_id"
         │
         ├── [ESCALATE] 升级到上一级审批人
         │   └── 通知原审批人"已升级至 {manager}"
         │
         └── [RETRY] 重试通知（最多 N 次）
             └── 仍无响应 → 最终 fallback 到 DEFAULT_DENY
```

#### 安全建议

- 生产环境默认使用 `DEFAULT_DENY` 策略
- 所有超时决策必须记录到审计日志
- 超时后的拒绝应该执行操作级别的回滚（如果有前置准备步骤）
- 对长时间运行的任务，在任务开始时做预审批，而非在执行到关键步骤时

---

### 5. Batch Approval: 批量审批

当 Agent 需要执行大量同类操作时，逐条审批会带来严重的用户疲劳。批量审批将多个操作聚合为一次审批。

#### 聚合策略

```python
class BatchApprovalAggregator:
    def __init__(self, window_seconds: int = 60, max_batch_size: int = 100):
        self.window = window_seconds
        self.max_batch_size = max_batch_size
        self.pending_ops = []
        self.timer = None

    def add_operation(self, operation: Operation):
        self.pending_ops.append(operation)
        if len(self.pending_ops) >= self.max_batch_size:
            self._flush_batch()
        elif self.timer is None:
            self._start_timer()

    def _flush_batch(self):
        if not self.pending_ops:
            return
        batch = BatchApprovalRequest(
            operations=list(self.pending_ops),
            summary=self._generate_summary(),
            total_risk=self._aggregate_risk(),
        )
        self.pending_ops.clear()
        self.timer = None
        self._send_for_approval(batch)

    def _generate_summary(self) -> str:
        ops = self.pending_ops
        return (
            f"批量操作: {len(ops)} 项\n"
            f"操作类型: {ops[0].tool} (均为同类操作)\n"
            f"目标: {len(set(o.target for o in ops))} 个独立资源\n"
            f"总风险评分: {self._aggregate_risk():.2f}\n"
            f"首项: {ops[0].summary}\n"
            f"末项: {ops[-1].summary}"
        )
```

#### 批量审批的类型

| 类型 | 描述 | 示例 |
|------|------|------|
| **同类聚合** | 相同类型的多个操作合并 | 删除 50 个过期缓存条目 |
| **定时批处理** | 窗口期内累积的操作统一提交 | 每 5 分钟审批一次 |
| **上下文关联批处理** | 同一任务链中的多个步骤 | "更新 10 个配置文件 + 重启服务" |
| **重复任务批处理** | 周期性的相同任务 | 每日数据清理任务，批准后自动执行 7 天 |

---

### 6. Approval Chain: 审批链

审批链定义了当需要多人批准时的流转路径和规则。

#### 单人审批 → 多人审批 → 升级路径

```python
class ApprovalChain:
    """
    审批链配置

    最简单的形式: [user]
    复杂形式: [user_a, user_b] (AND) — 两人都批准
              [user_a, user_b] (OR)  — 任一人批准
              升级: [user] → timeout → [manager] → timeout → [director]
    """

    def __init__(self, config: dict):
        self.steps = []
        for step_config in config["steps"]:
            self.steps.append(ApprovalStep(
                approvers=step_config["approvers"],
                logic=step_config.get("logic", "AND"),   # AND / OR
                timeout=step_config.get("timeout", 900),
                escalation=step_config.get("escalation"),  # 下一个 step
            ))

    @classmethod
    def single_approver(cls, user_id: str) -> "ApprovalChain":
        return cls({"steps": [
            {"approvers": [user_id], "logic": "OR", "timeout": 900}
        ]})

    @classmethod
    def multi_approver_and(cls, user_ids: list) -> "ApprovalChain":
        return cls({"steps": [
            {"approvers": user_ids, "logic": "AND", "timeout": 3600}
        ]})

    @classmethod
    def with_escalation(cls, primary: str, secondary: str, tertiary: str = None) -> "ApprovalChain":
        steps = [
            {"approvers": [primary], "logic": "OR", "timeout": 900},
            {"approvers": [secondary], "logic": "OR", "timeout": 3600},
        ]
        if tertiary:
            steps.append({"approvers": [tertiary], "logic": "OR", "timeout": 7200})
        return cls({"steps": steps})
```

#### 审批链模式

| 模式 | 规则 | 适用场景 |
|------|------|----------|
| 单人批准 | 任意一人批准即可 | 常规操作 |
| 多人与(AND) | 所有人必须批准 | 敏感数据操作 |
| 多人或(OR) | 任意一人批准即可 | 团队快速响应 |
| 多人多数决 | N 人中至少 M 人批准 | 高危操作（如 2/3）|
| 升级链 | 超时 → 上级审批 | 根据紧急程度逐级上报 |
| 会签链 | 按顺序逐级审批 | 财务审批（员工→主管→财务）|

---

### 7. Deferred Approval: 延迟审批

不是所有操作都需要即时执行。延迟审批模式允许用户在方便的时候集中处理审批请求。

#### 核心概念

```
用户发出指令: "帮我每周五清理一次日志文件"

时间线:
  ┌─── Agent 执行 ──→ 需要审批 ──→ 加入延迟队列
  │
  ├── 5分钟后 ──→ [可选] 用户主动打开审批面板批量处理
  │
  ├── 预定时间 ──→ 定期批量审批提醒
  │
  └── 操作执行 ──→ Agent 在获得批准后执行
```

#### Deferred Approval Manager

```python
class DeferredApprovalManager:
    """
    管理延迟审批队列，支持：
    - 操作排队（不阻塞主流程）
    - 定时批量审查
    - 优先级排序
    - 预约执行时间
    """

    def __init__(self):
        self.queue = PriorityQueue()
        self.review_schedule = None  # cron expression

    def defer(self, request: ApprovalRequest, execute_at: datetime = None):
        """
        将操作加入延迟审批队列。
        execute_at: 指定执行时间（若不指定，则在批量审查时处理）
        """
        item = DeferredItem(
            request=request,
            deferred_at=datetime.now(),
            execute_at=execute_at or datetime.now() + timedelta(hours=4),
            priority=self._calculate_priority(request),
        )
        self.queue.put(item)

    def review_batch(self, max_items: int = 20) -> list:
        """用户发起批量审查，返回一批待审批项"""
        batch = []
        while not self.queue.empty() and len(batch) < max_items:
            item = self.queue.get()
            if item.execute_at <= datetime.now():
                batch.append(item)
            else:
                # 未到执行时间，放回队列
                self.queue.put(item)
                break
        return batch

    def get_overdue_items(self) -> list:
        """获取已经超过执行时间但未审批的项"""
        now = datetime.now()
        overdue = []
        temp = []
        while not self.queue.empty():
            item = self.queue.get()
            if item.execute_at <= now:
                overdue.append(item)
            else:
                temp.append(item)
        for item in temp:
            self.queue.put(item)
        return overdue
```

#### "Approve Later" 模式

```
"在我审批之前，Agent 先把操作准备好（如生成 SQL），我确认后一键执行"
         ↓
Step 1: Agent 生成操作草案（如 DELETE 语句）
Step 2: 加入审批队列，"待执行"
Step 3: 用户审查所有待执行操作
Step 4: 批量批准 → Agent 按顺序执行
Step 5: 结果汇总报告
```

---

## Example Code: Python ApprovalWorkflowManager

下面是一个完整的审批工作流管理器实现，涵盖风险触发、多通道审批、超时处理和批量审批。

```python
"""
approval_workflow_manager.py

完整的审批工作流管理器，支持：
- 风险评分驱动的审批等级路由
- 多通道审批（in-app, email, slack, api callback）
- 审批超时处理与升级链
- 批量审批聚合
- 完整的审计日志
"""

import enum
import logging
import hashlib
import json
import time
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum, auto
from typing import Optional, Callable
from queue import PriorityQueue

logger = logging.getLogger(__name__)


# ─── Enums ───────────────────────────────────────────────────────

class ApprovalLevel(Enum):
    AUTO_ALLOW = "auto_allow"
    NOTIFY_ONLY = "notify_only"
    CONFIRM = "confirm"
    APPROVE = "approve"
    MULTI_PARTY_APPROVE = "multi_party_approve"

    @property
    def requires_execution_block(self) -> bool:
        return self.value in ("confirm", "approve", "multi_party_approve")

    @property
    def requires_human_decision(self) -> bool:
        return self.value in ("approve", "multi_party_approve")


class ApprovalDecision(Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"
    TIMED_OUT = "timed_out"
    ESCALATED = "escalated"
    DEFERRED = "deferred"


class ApprovalChannel(Enum):
    IN_APP = "in_app"
    EMAIL = "email"
    MOBILE_PUSH = "mobile_push"
    SLACK = "slack"
    API_CALLBACK = "api_callback"


class TimeoutStrategy(Enum):
    DEFAULT_DENY = "default_deny"
    DEFAULT_ALLOW = "default_allow"
    ESCALATE = "escalate"
    DEFER = "defer"


# ─── Data Models ─────────────────────────────────────────────────

@dataclass
class Operation:
    """表示一个待执行的 Agent 操作"""
    tool: str
    action: str
    parameters: dict
    target: str
    environment: str
    summary: str
    risk_score: float = 0.0
    sensitivity_level: str = "low"

    def fingerprint(self) -> str:
        """操作指纹，用于批处理和缓存"""
        raw = f"{self.tool}:{self.action}:{self.target}"
        return hashlib.sha256(raw.encode()).hexdigest()[:16]


@dataclass
class ApprovalRequest:
    """审批请求"""
    id: str
    operation: Operation
    level: ApprovalLevel
    channels: list[ApprovalChannel]
    approvers: list[str]
    status: ApprovalDecision = ApprovalDecision.PENDING
    reason: str = ""
    created_at: datetime = field(default_factory=datetime.now)
    timeout_seconds: int = 900
    escalation_chain: list[str] = field(default_factory=list)

    def is_expired(self) -> bool:
        return (datetime.now() - self.created_at).total_seconds() > self.timeout_seconds


@dataclass
class ApprovalResult:
    """审批结果"""
    request_id: str
    decision: ApprovalDecision
    approved_by: Optional[str] = None
    reason: str = ""
    decided_at: datetime = field(default_factory=datetime.now)
    approval_chain_level: int = 0
    risk_score_at_decision: float = 0.0


# ─── Risk Scorer ─────────────────────────────────────────────────

class RiskScorer:
    """
    风险评分器，基于多维度评估操作风险
    """

    # 工具风险基准
    TOOL_RISK_BASELINE = {
        "file.read": 0.05,
        "file.write": 0.30,
        "file.delete": 0.75,
        "db.query": 0.20,
        "db.execute": 0.70,
        "db.drop_table": 0.95,
        "network.get": 0.05,
        "network.post": 0.25,
        "network.delete": 0.60,
        "auth.create_token": 0.50,
        "auth.revoke": 0.80,
        "admin.change_config": 0.85,
        "admin.delete_resource": 0.90,
        "payment.charge": 0.75,
        "payment.refund": 0.70,
        "email.send": 0.15,
        "notification.send": 0.05,
    }

    # 数据敏感度权重
    SENSITIVITY_WEIGHTS = {
        "public": 0.0,
        "internal": 0.15,
        "confidential": 0.35,
        "restricted": 0.55,
        "pii": 0.70,
        "financial": 0.80,
    }

    # 环境权重
    ENVIRONMENT_WEIGHTS = {
        "local": 0.0,
        "dev": 0.10,
        "staging": 0.20,
        "production": 0.50,
    }

    # 权重配置
    WEIGHT_TOOL = 0.35
    WEIGHT_PARAM = 0.20
    WEIGHT_SENSITIVITY = 0.25
    WEIGHT_ENVIRONMENT = 0.15
    WEIGHT_REVERSIBILITY = 0.05

    def __init__(self):
        self.thresholds = {
            ApprovalLevel.AUTO_ALLOW: 0.20,
            ApprovalLevel.NOTIFY_ONLY: 0.40,
            ApprovalLevel.CONFIRM: 0.60,
            ApprovalLevel.APPROVE: 0.80,
            ApprovalLevel.MULTI_PARTY_APPROVE: 1.0,
        }

    def compute(self, operation: Operation) -> tuple[float, ApprovalLevel]:
        """
        计算风险评分并映射到审批等级。
        返回 (risk_score, approval_level)
        """
        # 1. 工具风险
        tool_key = f"{operation.tool}.{operation.action}"
        tool_risk = self.TOOL_RISK_BASELINE.get(tool_key, 0.30)

        # 2. 参数风险（检查参数中是否包含危险值）
        param_risk = self._compute_param_risk(operation.parameters)

        # 3. 数据敏感度
        sensitivity_risk = self.SENSITIVITY_WEIGHTS.get(
            operation.sensitivity_level, 0.10
        )

        # 4. 环境风险
        env_risk = self.ENVIRONMENT_WEIGHTS.get(
            operation.environment, 0.10
        )

        # 5. 可逆性
        reversibility_risk = 0.10 if self._is_reversible(operation) else 0.70

        # 加权综合
        risk_score = (
            tool_risk * self.WEIGHT_TOOL +
            param_risk * self.WEIGHT_PARAM +
            sensitivity_risk * self.WEIGHT_SENSITIVITY +
            env_risk * self.WEIGHT_ENVIRONMENT +
            reversibility_risk * self.WEIGHT_REVERSIBILITY
        )

        # 裁剪到 [0, 1]
        risk_score = max(0.0, min(1.0, risk_score))

        # 映射到审批等级
        level = self._map_to_level(risk_score)

        return risk_score, level

    def _compute_param_risk(self, params: dict) -> float:
        """分析参数中的风险因素"""
        risk = 0.0
        param_str = json.dumps(params).lower()

        # 检查高风险关键词
        high_risk_keywords = ["drop", "truncate", "delete", "remove", "purge",
                              "shutdown", "restart", "all", "*", "%"]
        for kw in high_risk_keywords:
            if kw in param_str:
                risk += 0.15

        # 检查大量数据操作
        if "limit" in params:
            limit = params["limit"]
            if isinstance(limit, (int, float)) and limit > 1000:
                risk += 0.20
            elif isinstance(limit, (int, float)) and limit > 100:
                risk += 0.10

        if "batch_size" in params:
            batch = params["batch_size"]
            if isinstance(batch, (int, float)) and batch > 100:
                risk += 0.15

        if "recursive" in params and params["recursive"] is True:
            risk += 0.25

        if "force" in params and params["force"] is True:
            risk += 0.20

        return min(risk, 1.0)

    def _is_reversible(self, operation: Operation) -> bool:
        """判断操作是否可逆"""
        irreversible_actions = [
            "delete", "drop", "truncate", "remove", "purge",
            "shutdown", "revoke", "terminate",
        ]
        return not any(a in operation.action.lower()
                       for a in irreversible_actions)

    def _map_to_level(self, risk_score: float) -> ApprovalLevel:
        """风险分数 → 审批等级"""
        for level, threshold in sorted(
            self.thresholds.items(), key=lambda x: x[1]
        ):
            if risk_score <= threshold:
                return level
        return ApprovalLevel.MULTI_PARTY_APPROVE

    def update_thresholds(self, new_thresholds: dict):
        """动态调整阈值"""
        for level, val in new_thresholds.items():
            if level in self.thresholds:
                self.thresholds[level] = val


# ─── Approval Channels ───────────────────────────────────────────

class ApprovalChannelHandler(ABC):
    """审批通道抽象基类"""

    def __init__(self, channel: ApprovalChannel):
        self.channel = channel
        self.priority = self._default_priority()

    def _default_priority(self) -> int:
        priorities = {
            ApprovalChannel.IN_APP: 1,
            ApprovalChannel.MOBILE_PUSH: 2,
            ApprovalChannel.SLACK: 3,
            ApprovalChannel.EMAIL: 4,
            ApprovalChannel.API_CALLBACK: 5,
        }
        return priorities.get(self.channel, 99)

    @abstractmethod
    def send_approval_request(self, request: ApprovalRequest) -> bool:
        """发送审批请求到指定通道"""

    @abstractmethod
    def check_response(self, request_id: str) -> Optional[ApprovalDecision]:
        """检查用户在通道上是否已经响应"""


class InAppChannelHandler(ApprovalChannelHandler):
    """应用内通知通道"""

    def __init__(self, notification_callback: Callable):
        super().__init__(ApprovalChannel.IN_APP)
        self.callback = notification_callback
        self._responses = {}

    def send_approval_request(self, request: ApprovalRequest) -> bool:
        try:
            payload = {
                "type": "approval_request",
                "request_id": request.id,
                "title": f"审批请求: {request.operation.summary}",
                "risk_level": request.level.value,
                "risk_score": f"{request.operation.risk_score:.2f}",
                "details": {
                    "tool": request.operation.tool,
                    "action": request.operation.action,
                    "target": request.operation.target,
                    "env": request.operation.environment,
                },
                "actions": [
                    {"id": "approve", "label": "批准", "style": "primary"},
                    {"id": "reject", "label": "拒绝", "style": "danger"},
                ],
                "timeout": request.timeout_seconds,
            }
            self.callback(payload)
            return True
        except Exception as e:
            logger.error(f"InApp 通知失败: {e}")
            return False

    def check_response(self, request_id: str) -> Optional[ApprovalDecision]:
        return self._responses.get(request_id)

    def register_response(self, request_id: str, decision: str, user: str):
        self._responses[request_id] = ApprovalDecision.APPROVED \
            if decision == "approve" else ApprovalDecision.REJECTED


class SlackChannelHandler(ApprovalChannelHandler):
    """Slack Bot 审批通道"""

    def __init__(self, webhook_url: str, channel: str = "#approvals"):
        super().__init__(ApprovalChannel.SLACK)
        self.webhook_url = webhook_url
        self.channel = channel
        self._responses = {}

    def send_approval_request(self, request: ApprovalRequest) -> bool:
        # 在实际系统中会调用 Slack Webhook API
        blocks = [
            {"type": "section", "text": {
                "type": "mrkdwn",
                "text": f"*审批请求 #{request.id}*\n{request.operation.summary}"
            }},
            {"type": "section", "fields": [
                {"type": "mrkdwn", "text": f"*工具:* {request.operation.tool}"},
                {"type": "mrkdwn", "text": f"*环境:* {request.operation.environment}"},
                {"type": "mrkdwn", "text": f"*风险评分:* {request.operation.risk_score:.2f}"},
                {"type": "mrkdwn", "text": f"*超时:* {request.timeout_seconds}s"},
            ]},
            {"type": "actions", "elements": [
                {"type": "button", "text": {"type": "plain_text", "text": "批准"},
                 "style": "primary", "value": "approve", "action_id": f"approve_{request.id}"},
                {"type": "button", "text": {"type": "plain_text", "text": "拒绝"},
                 "style": "danger", "value": "reject", "action_id": f"reject_{request.id}"},
            ]},
        ]
        # requests.post(self.webhook_url, json={"channel": self.channel, "blocks": blocks})
        logger.info(f"Slack 审批请求已发送到 {self.channel}: request_id={request.id}")
        return True

    def check_response(self, request_id: str) -> Optional[ApprovalDecision]:
        return self._responses.get(request_id)


# ─── Batch Approval ──────────────────────────────────────────────

class BatchApprovalAggregator:
    """
    批量审批聚合器。
    将窗口期内的同类操作聚合为一条审批请求。
    """

    def __init__(self, window_seconds: int = 60, max_batch_size: int = 100):
        self.window = window_seconds
        self.max_batch_size = max_batch_size
        self._pending: dict[str, list[Operation]] = {}
        self._timers: dict[str, float] = {}

    def add(self, operation: Operation) -> Optional[ApprovalRequest]:
        """
        添加操作到待批队列。
        如果达到批量阈值或操作不可聚合，立即返回审批请求。
        否则返回 None（等待窗口期结束）。
        """
        key = self._batch_key(operation)

        if key not in self._pending:
            self._pending[key] = []
            self._timers[key] = time.time() + self.window

        self._pending[key].append(operation)

        # 达到最大批量 → 立即 Flush
        if len(self._pending[key]) >= self.max_batch_size:
            return self._flush(key)

        return None

    def _batch_key(self, operation: Operation) -> str:
        """生成批量分组的 key（同类操作聚合）"""
        return f"{operation.tool}:{operation.action}:{operation.environment}"

    def _flush(self, key: str) -> Optional[ApprovalRequest]:
        """将指定 key 的待处理操作聚合成一个审批请求"""
        operations = self._pending.pop(key, [])
        self._timers.pop(key, None)

        if not operations:
            return None

        op_count = len(operations)
        avg_risk = sum(op.risk_score for op in operations) / op_count

        summary = (
            f"批量操作: {op_count} 项 "
            f"({operations[0].tool}.{operations[0].action}) "
            f"平均风险: {avg_risk:.2f}"
        )

        return ApprovalRequest(
            id=f"batch_{int(time.time())}_{hash(key) % 10000:04d}",
            operation=Operation(
                tool=operations[0].tool,
                action=operations[0].action,
                parameters={"batch_size": op_count},
                target=f"{op_count} resources",
                environment=operations[0].environment,
                summary=summary,
                risk_score=avg_risk,
            ),
            level=self._level_for_risk(avg_risk),
            channels=[ApprovalChannel.IN_APP, ApprovalChannel.SLACK],
            approvers=[],
            escalation_chain=[],
        )

    def flush_expired_windows(self) -> list[ApprovalRequest]:
        """清理已过期的窗口，生成批量审批请求"""
        requests = []
        now = time.time()
        for key in list(self._timers.keys()):
            if now >= self._timers[key]:
                req = self._flush(key)
                if req:
                    requests.append(req)
        return requests

    def _level_for_risk(self, risk: float) -> ApprovalLevel:
        if risk <= 0.2:
            return ApprovalLevel.AUTO_ALLOW
        elif risk <= 0.4:
            return ApprovalLevel.NOTIFY_ONLY
        elif risk <= 0.6:
            return ApprovalLevel.CONFIRM
        elif risk <= 0.8:
            return ApprovalLevel.APPROVE
        return ApprovalLevel.MULTI_PARTY_APPROVE


# ─── Approval Chain ──────────────────────────────────────────────

class ApprovalChain:
    """
    审批链：支持单人、多人、级链升级模式。

    配置示例:
        chain = ApprovalChain([
            {"approvers": ["alice"], "logic": "OR", "timeout": 300},
            {"approvers": ["bob", "carol"], "logic": "AND", "timeout": 900},
            {"approvers": ["dave"], "logic": "OR", "timeout": 3600},
        ])
    """

    def __init__(self, steps: list[dict]):
        self.steps = steps
        self.current_step = 0
        self._decisions: dict[str, list[ApprovalDecision]] = {}

    @property
    def current_approvers(self) -> list[str]:
        """当前步骤的审批人列表"""
        if self.current_step < len(self.steps):
            return self.steps[self.current_step]["approvers"]
        return []

    @property
    def current_logic(self) -> str:
        """当前步骤的逻辑（AND/OR）"""
        if self.current_step < len(self.steps):
            return self.steps[self.current_step].get("logic", "OR")
        return "OR"

    @property
    def current_timeout(self) -> int:
        """当前步骤的超时时间（秒）"""
        if self.current_step < len(self.steps):
            return self.steps[self.current_step].get("timeout", 900)
        return 900

    def record_decision(self, approver: str, decision: ApprovalDecision) -> Optional[ApprovalDecision]:
        """
        记录审批人决策。
        返回当前步骤是否已满足条件（批准/拒绝/继续等待）
        """
        if self.current_step not in self._decisions:
            self._decisions[self.current_step] = []

        self._decisions[self.current_step].append(decision)

        return self._evaluate_step()

    def _evaluate_step(self) -> Optional[ApprovalDecision]:
        """评估当前步骤是否已完成"""
        decisions = self._decisions.get(self.current_step, [])
        logic = self.current_logic

        if logic == "AND":
            if all(d == ApprovalDecision.APPROVED for d in decisions):
                return ApprovalDecision.APPROVED
            if any(d == ApprovalDecision.REJECTED for d in decisions):
                return ApprovalDecision.REJECTED

        elif logic == "OR":
            if any(d == ApprovalDecision.APPROVED for d in decisions):
                return ApprovalDecision.APPROVED
            if all(d == ApprovalDecision.REJECTED for d in decisions):
                return ApprovalDecision.REJECTED

        return None  # 仍在等待

    def escalate(self) -> bool:
        """升级到下一个步骤"""
        if self.current_step < len(self.steps) - 1:
            self.current_step += 1
            return True
        return False


# ─── Main Approval Workflow Manager ──────────────────────────────

class ApprovalWorkflowManager:
    """
    审批工作流管理器（核心类）

    使用示例:
        manager = ApprovalWorkflowManager()
        result = await manager.execute_with_approval(
            operation=Operation(...),
            user_context={"user_id": "alice", "role": "admin"},
        )
    """

    def __init__(self, risk_scorer: RiskScorer = None):
        self.risk_scorer = risk_scorer or RiskScorer()
        self.channels: dict[ApprovalChannel, ApprovalChannelHandler] = {}
        self.batch_aggregator = BatchApprovalAggregator()
        self.audit_log: list[ApprovalResult] = []
        self.active_requests: dict[str, ApprovalRequest] = {}
        self.timeout_strategy = TimeoutStrategy.DEFAULT_DENY

    def register_channel(self, handler: ApprovalChannelHandler):
        """注册审批通道"""
        self.channels[handler.channel] = handler

    async def execute_with_approval(self, operation: Operation,
                                     user_context: dict) -> ApprovalResult:
        """
        带审批的执行入口。
        1. 计算风险
        2. 确定审批等级
        3. 不阻塞 → 直接执行
        4. 需要审批 → 启动审批流程
        """
        # Step 1: 风险评分
        risk_score, approval_level = self.risk_scorer.compute(operation)
        operation.risk_score = risk_score

        # Step 2: 检查是否可以直接执行
        if approval_level == ApprovalLevel.AUTO_ALLOW:
            return ApprovalResult(
                request_id="auto_allow",
                decision=ApprovalDecision.APPROVED,
                approved_by="system",
                reason="风险评分低于阈值，自动允许",
                risk_score_at_decision=risk_score,
            )

        # Step 3: 检查是否只需通知
        if approval_level == ApprovalLevel.NOTIFY_ONLY:
            # 异步通知，不阻塞
            self._send_notification(operation)
            return ApprovalResult(
                request_id="notify_only",
                decision=ApprovalDecision.APPROVED,
                approved_by="system",
                reason=f"风险评分 {risk_score:.2f}，已执行并通知用户",
                risk_score_at_decision=risk_score,
            )

        # Step 4: 需要审批，尝试批量聚合
        if approval_level in (ApprovalLevel.CONFIRM, ApprovalLevel.APPROVE):
            batch_request = self.batch_aggregator.add(operation)
            if batch_request is None:
                # 已加入批量队列，等待窗口期
                return ApprovalResult(
                    request_id="batched",
                    decision=ApprovalDecision.DEFERRED,
                    reason="已加入批量审批队列",
                    risk_score_at_decision=risk_score,
                )
            # 批量窗口已满或超时，发出批量审批
            return await self._process_approval(batch_request, user_context)

        # Step 5: 多人员批准
        request = self._create_approval_request(operation, approval_level, user_context)
        return await self._process_approval(request, user_context)

    async def _process_approval(self, request: ApprovalRequest,
                                 user_context: dict) -> ApprovalResult:
        """处理审批请求的核心逻辑"""
        self.active_requests[request.id] = request

        # Step 1: 通过注册的通道发送审批请求
        sent = False
        for channel in request.channels:
            handler = self.channels.get(channel)
            if handler:
                handler.send_approval_request(request)
                sent = True

        if not sent:
            logger.warning(f"审批请求 {request.id} 没有可用的通知通道")
            return ApprovalResult(
                request_id=request.id,
                decision=ApprovalDecision.REJECTED,
                reason="没有可用的审批通知通道，默认拒绝",
                risk_score_at_decision=request.operation.risk_score,
            )

        # Step 2: 等待审批决策（包含超时处理）
        if request.level == ApprovalLevel.CONFIRM:
            decision = await self._wait_for_decision(
                request, timeout=request.timeout_seconds
            )
        else:
            # APPROVE / MULTI_PARTY_APPROVE 使用审批链
            decision = await self._process_approval_chain(request)

        # Step 3: 记录审计日志
        result = ApprovalResult(
            request_id=request.id,
            decision=decision,
            approved_by=user_context.get("user_id", "unknown"),
            risk_score_at_decision=request.operation.risk_score,
        )
        self.audit_log.append(result)
        self.active_requests.pop(request.id, None)

        return result

    async def _wait_for_decision(self, request: ApprovalRequest,
                                  timeout: int) -> ApprovalDecision:
        """
        等待用户决策（轮询审批通道）
        实际生产环境应使用 WebSocket / Webhook 回调
        """
        start = time.time()
        while time.time() - start < timeout:
            for channel in request.channels:
                handler = self.channels.get(channel)
                if handler:
                    decision = handler.check_response(request.id)
                    if decision in (ApprovalDecision.APPROVED,
                                    ApprovalDecision.REJECTED):
                        return decision
            await self._sleep(1)  # 自适应轮询间隔

        # 超时处理
        return self._handle_timeout(request)

    async def _process_approval_chain(self, request: ApprovalRequest) -> ApprovalDecision:
        """处理审批链（含升级逻辑）"""
        chain = ApprovalChain([
            {"approvers": request.approvers, "logic": "AND"
             if request.level == ApprovalLevel.MULTI_PARTY_APPROVE else "OR",
             "timeout": request.timeout_seconds},
        ])

        if request.escalation_chain:
            for approver in request.escalation_chain:
                chain.steps.append({
                    "approvers": [approver],
                    "logic": "OR",
                    "timeout": request.timeout_seconds * 2,
                })

        while True:
            decision = await self._wait_for_decision(
                request, timeout=chain.current_timeout
            )

            if decision in (ApprovalDecision.APPROVED, ApprovalDecision.REJECTED):
                return decision

            # 超时 → 尝试升级
            if not chain.escalate():
                return self._handle_timeout(request)

            # 在新步骤中继续等待
            logger.info(f"审批已升级到第 {chain.current_step + 1} 级")

    def _handle_timeout(self, request: ApprovalRequest) -> ApprovalDecision:
        """处理审批超时"""
        strategy = request.level == ApprovalLevel.MULTI_PARTY_APPROVE and \
                   TimeoutStrategy.ESCALATE or self.timeout_strategy

        if strategy == TimeoutStrategy.DEFAULT_DENY:
            logger.info(f"审批超时（默认拒绝）: request_id={request.id}")
            return ApprovalDecision.TIMED_OUT

        elif strategy == TimeoutStrategy.DEFAULT_ALLOW:
            logger.warning(f"审批超时（默认允许，不安全）: request_id={request.id}")
            return ApprovalDecision.APPROVED

        else:
            # DEFER
            logger.info(f"审批已推迟: request_id={request.id}")
            return ApprovalDecision.DEFERRED

    def _send_notification(self, operation: Operation):
        """发送 NOTIFY_ONLY 通知"""
        notification = {
            "type": "approval_notification",
            "severity": "info",
            "message": f"Agent 已执行操作: {operation.summary}",
            "details": {
                "tool": operation.tool,
                "action": operation.action,
                "target": operation.target,
                "risk_score": f"{operation.risk_score:.2f}",
            },
        }
        for handler in self.channels.values():
            if handler.channel in (ApprovalChannel.IN_APP, ApprovalChannel.SLACK):
                handler.send_approval_request(
                    ApprovalRequest(
                        id=f"notify_{int(time.time())}",
                        operation=operation,
                        level=ApprovalLevel.NOTIFY_ONLY,
                        channels=[handler.channel],
                        approvers=[],
                    )
                )

    def _create_approval_request(self, operation: Operation,
                                  level: ApprovalLevel,
                                  user_context: dict) -> ApprovalRequest:
        """创建审批请求对象"""
        return ApprovalRequest(
            id=f"req_{int(time.time())}_{hash(str(operation)) % 10000:04d}",
            operation=operation,
            level=level,
            channels=[ApprovalChannel.IN_APP, ApprovalChannel.SLACK,
                      ApprovalChannel.EMAIL],
            approvers=[user_context.get("user_id", "unknown")],
            timeout_seconds={
                ApprovalLevel.CONFIRM: 300,
                ApprovalLevel.APPROVE: 900,
                ApprovalLevel.MULTI_PARTY_APPROVE: 3600,
            }.get(level, 900),
            escalation_chain=user_context.get("escalation_chain", []),
        )

    async def _sleep(self, seconds: int):
        """异步休眠（示例：实际使用 asyncio.sleep）"""
        import asyncio
        await asyncio.sleep(seconds)

    def get_approval_stats(self) -> dict:
        """获取审批统计信息"""
        total = len(self.audit_log)
        approved = sum(1 for r in self.audit_log
                       if r.decision == ApprovalDecision.APPROVED)
        rejected = sum(1 for r in self.audit_log
                       if r.decision == ApprovalDecision.REJECTED)
        timed_out = sum(1 for r in self.audit_log
                        if r.decision == ApprovalDecision.TIMED_OUT)

        return {
            "total_requests": total,
            "approved": approved,
            "rejected": rejected,
            "timed_out": timed_out,
            "approval_rate": approved / total if total > 0 else 0,
            "rejection_rate": rejected / total if total > 0 else 0,
        }


# ─── Usage Example ───────────────────────────────────────────────

def example_usage():
    """使用示例"""

    # 初始化审批管理器
    manager = ApprovalWorkflowManager()

    # 注册审批通道
    manager.register_channel(InAppChannelHandler(
        notification_callback=lambda payload: print(f"[InApp] {payload}")
    ))
    manager.register_channel(SlackChannelHandler(
        webhook_url="https://hooks.slack.com/services/xxx",
        channel="#agent-approvals",
    ))

    # 定义操作
    safe_op = Operation(
        tool="file", action="read",
        parameters={"path": "/tmp/report.txt"},
        target="/tmp/report.txt",
        environment="dev",
        summary="读取开发环境报告文件",
        sensitivity_level="internal",
    )

    risky_op = Operation(
        tool="db", action="execute",
        parameters={"query": "DELETE FROM users WHERE active=0",
                     "limit": 10000},
        target="production-db.users",
        environment="production",
        summary="从生产数据库批量删除非活跃用户",
        sensitivity_level="pii",
    )

    # 执行安全操作（自动允许）
    result1 = await manager.execute_with_approval(
        safe_op, {"user_id": "alice"}
    )
    # → decision=APPROVED, reason="自动允许"

    # 执行高风险操作（需要审批）
    result2 = await manager.execute_with_approval(
        risky_op, {"user_id": "alice", "escalation_chain": ["bob", "carol"]}
    )
    # → 发送审批请求到 InApp + Slack
    # → 等待用户决策
```

---

## Capability Boundaries（能力边界）

### 1. 审批疲劳（Approval Fatigue）

**问题**：当审批请求频率过高时，用户开始不加审查地批准所有请求，使审批流程形同虚设。

**表现**：
- 用户"习惯性批准"，不再阅读上下文
- 用户开始忽略通知（通知疲劳）
- 误点"批准"的概率上升

**缓解策略**：
- 控制审批频率上限（如每小时最多 N 次主动审批请求）
- 对重复的同类操作自动降级为 notify-only
- 引入"强制关注"机制：高风险审批要求用户输入理由或验证码
- 动态阈值调整：用户疲劳度高时提高触发阈值

### 2. 延迟 / 可用性（Latency / Availability）

**问题**：审批等待时间对 Agent 任务进度的影响不可忽视。

| 场景 | 延迟 | 影响 |
|------|------|------|
| 用户在设备前 | 5-30 秒 | 低 |
| 用户通过移动端审批 | 30 秒 - 5 分钟 | 中 |
| 用户不在 / 忽略通知 | 15 分钟 - 数小时 | 灾难性 |
| 需要多人员批准 | 数小时 - 数天 | 极高 |

**缓解**：
- 预审批（在执行可能触发高风险的步骤前预先请求批准）
- 并行审批通道（同时在多个通道推送）
- 用户预设"静默时段"审批策略
- 可预配置的"信任规则"绕过某些审批

### 3. 用户必须可用（User Must Be Available）

**问题**：多级审批链假设链上的用户随时可被联系到，但现实中并非如此。

**限制**：
- 非工作时间（晚上、周末、节假日）
- 用户休假、病假
- 审批链上的用户离职或转岗
- 跨时区协作

**设计考虑**：
- 支持备用审批人（backup approver）
- 审批链必须有兜底机制（如最终默认拒绝）
- 支持预设调度规则（工作时间内升级到在线审批人）
- 节假日日历集成

### 4. 审批流程本身的安全（Security of Approval Process）

**问题**：审批机制本身可能被攻击者利用。

**威胁**：
- 审批请求伪造（攻击者触发审批并批准）
- 审批通道劫持（如 Slack token 泄露）
- 审批人身份冒用

**缓解**：
- 审批决策必须经过签名验证
- 审批通道需要独立认证（不能仅靠 bearer token）
- 关键操作需要"双因素审批"（在通知通道外另行验证）
- 所有审批流水记录不可篡改（append-only audit log）

---

## Comparison: auto vs notify vs confirm vs multi-approve

| 维度 | AUTO_ALLOW | NOTIFY_ONLY | CONFIRM | APPROVE | MULTI_PARTY_APPROVE |
|------|-----------|-------------|---------|---------|-------------------|
| **人类介入** | 无 | 事后通知 | 事前点击确认 | 事前审批（需理由）| 多人事前审批 |
| **执行延迟** | 无 | 无 | 短（秒级）| 中（分级）| 长（小时级）|
| **用户负担** | 无 | 低 | 中 | 高 | 极高 |
| **安全性** | 低（无防护）| 中（仅审计）| 中高 | 高 | 极高 |
| **适用场景** | 读操作、计算 | 可逆操作、低风险写 | 中等风险写 | 高风险写、删除 | 生产环境变更、财务 |
| **阻断执行** | 否 | 否 | 是 | 是 | 是 |
| **审计保障** | 有日志 | 有日志+通知 | 有日志+决策记录 | 有日志+审批理由 | 完整审计 |
| **用户体验** | 最佳 | 良好 | 可接受 | 受影响 | 显著受影响 |
| **误报成本** | 无 | 低 | 中 | 高 | 极高 |
| **典型风险分** | 0-0.2 | 0.2-0.4 | 0.4-0.6 | 0.6-0.8 | 0.8-1.0 |

---

## Engineering Optimization: 工程优化

### Approval Routing Optimization（审批路由优化）

#### 智能匹配原则

1. **最近审批者优先**：如果某个操作和用户最近批准过的操作相似度高，优先路由给该用户
2. **上下文切换最小化**：避免将同一任务链中的审批路由给不同的人
3. **负载均衡**：在多个审批人之间平均分配审批请求，避免单点压力
4. **领域专长匹配**：将数据库相关操作路由给 DBA，安全相关操作路由给安全工程师

```python
class SmartApprovalRouter:
    """
    智能审批路由，基于以下因素选择最优审批人：
    - 历史审批记录
    - 审批人当前负载
    - 领域匹配度
    - 响应速度预期
    """

    def __init__(self):
        self.approver_profiles: dict[str, ApproverProfile] = {}

    def select_best_approver(self, operation: Operation,
                              candidates: list[str]) -> str:
        return max(
            candidates,
            key=lambda a: self._compute_suitability(operation, a)
        )

    def _compute_suitability(self, operation: Operation,
                              approver: str) -> float:
        profile = self.approver_profiles.get(approver)
        if not profile:
            return 0.5  # 默认中等适合度

        score = 0.0
        score += profile.domain_match.get(operation.tool, 0) * 0.4
        score += (1.0 - profile.current_load) * 0.3  # 低负载加分
        score += profile.avg_response_speed * 0.2     # 响应速度
        score += profile.acceptance_rate * 0.1         # 历史接受率

        return score
```

### Caching Approved Operations（审批缓存）

对已审批的操作进行缓存，避免同一操作的重复审批。

```python
class ApprovalCache:
    """
    审批缓存。
    当操作指纹匹配且未过期时，直接返回已批准的审批结果。
    """

    def __init__(self, ttl_seconds: int = 3600):
        self._cache: dict[str, tuple[ApprovalResult, float]] = {}
        self.ttl = ttl_seconds

    def get(self, operation: Operation) -> Optional[ApprovalResult]:
        fingerprint = self._fingerprint(operation)
        if fingerprint in self._cache:
            result, timestamp = self._cache[fingerprint]
            if time.time() - timestamp < self.ttl:
                return result
            else:
                del self._cache[fingerprint]
        return None

    def set(self, operation: Operation, result: ApprovalResult):
        fingerprint = self._fingerprint(operation)
        self._cache[fingerprint] = (result, time.time())

    def _fingerprint(self, operation: Operation) -> str:
        # 用操作特征 + 环境 = 缓存键
        raw = f"{operation.tool}:{operation.action}:{operation.target}:{operation.environment}"
        return hashlib.sha256(raw.encode()).hexdigest()

    def invalidate(self, pattern: str):
        """按模式删除缓存条目"""
        to_delete = [k for k in self._cache if pattern in k]
        for k in to_delete:
            del self._cache[k]

    @property
    def hit_rate(self) -> float:
        """缓存命中率（监控指标）"""
        # 在实际系统中需要跟踪 hit/miss 计数
        return 0.0
```

### 其他优化策略

#### Adaptive Polling Interval（自适应轮询间隔）

```python
class AdaptivePoller:
    """
    自适应审批轮询器。
    根据历史响应模式动态调整轮询频率，降低系统负载。
    """

    def __init__(self, base_interval: float = 1.0):
        self.interval = base_interval
        self.history: list[float] = []

    def record_response_time(self, seconds: float):
        self.history.append(seconds)
        if len(self.history) > 100:
            self.history.pop(0)
        self._recalibrate()

    def _recalibrate(self):
        if len(self.history) < 10:
            return
        avg_response = sum(self.history) / len(self.history)
        # 用户通常在前 20% 的时间段内响应
        fast_response = sorted(self.history)[len(self.history) // 5]
        # 轮询间隔设为最快响应时间的 1/10（保证至少 10 次轮询检查）
        self.interval = max(0.1, fast_response / 10)
```

#### Parallel Channel Dispatch（并行通道分发）

同时通过多个通道发送审批请求，任一通道的响应即视为有效决策：

```python
async def parallel_dispatch(request: ApprovalRequest,
                             channels: list[ApprovalChannelHandler]):
    """
    并行向所有注册通道发送审批请求。
    先响应的通道将结果作为决策。
    """
    responses = await asyncio.gather(*[
        handler.send_and_wait(request)
        for handler in channels
    ], return_exceptions=True)

    for response in responses:
        if isinstance(response, ApprovalDecision) and \
           response in (ApprovalDecision.APPROVED, ApprovalDecision.REJECTED):
            return response

    return ApprovalDecision.TIMED_OUT
```

#### Operation Pre-Authorization（操作预授权）

```python
class PreAuthToken:
    """
    预授权令牌。
    Agent 在执行复杂任务链前，可以先申请一个预授权令牌，
    在后续步骤中使用该令牌绕过审批。

    令牌生命周期:
    1. Agent 请求预授权（附带完整操作计划）
    2. 用户审查计划并颁发令牌（有效期 + 限制条件）
    3. Agent 使用令牌执行操作（无需逐条审批）
    4. 令牌过期或使用后自动失效
    """

    def __init__(self, operations: list[Operation],
                 expires_at: datetime,
                 issued_by: str):
        self.operations = operations
        self.expires_at = expires_at
        self.issued_by = issued_by
        self.token = hashlib.sha256(
            json.dumps([op.fingerprint() for op in operations],
                       sort_keys=True).encode()
        ).hexdigest()
        self.used = set()

    def is_valid_for(self, operation: Operation) -> bool:
        if datetime.now() > self.expires_at:
            return False
        if operation.fingerprint() in self.used:
            return False
        return any(op.fingerprint() == operation.fingerprint()
                   for op in self.operations)

    def consume(self, operation: Operation) -> bool:
        if self.is_valid_for(operation):
            self.used.add(operation.fingerprint())
            return True
        return False
```

---

## 总结

Approval workflow 是 AI Agent 安全体系中的核心护栏机制，它通过引入人类审批来防止 Agent 在高风险操作中做出不当决策。设计的关键是在安全性和可用性之间取得平衡——太严格的审批会使用户疲劳并降低 Agent 的价值，太宽松的审批则使安全机制形同虚设。

工程上需要综合运用风险评分、多通道通知、超时处理、批量聚合、审批链、操作缓存和预授权等多种技术，并根据用户行为和上下文动态调整审批策略。最重要的是，审批流程本身必须是安全的（防伪造、防劫持）且可审计的（所有决策可追溯）。
