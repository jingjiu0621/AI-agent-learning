# 12.6.1 Audit Logging — 审计日志

## 简单介绍

Audit Logging（审计日志）是 AI Agent 安全体系的基础设施层，负责以**完整、准确、防篡改**的方式记录 Agent 系统的所有关键行为。在 Agent 自主决策、调用工具、访问数据的场景下，审计日志不仅是事后追溯和事故调查的唯一可靠依据，也是实时监控异常行为和满足合规要求（SOC 2、ISO 27001、GDPR）的核心手段。

与传统的系统审计日志不同，Agent 审计日志需要捕获**思考轨迹（Thought Trajectory）**、**工具调用决策链**、**权限评估结果**等 AI 特有的语义信息，使日志不仅回答"发生了什么"，还能回答"为什么做出这个决定"。

---

## 基本原理

审计日志的核心是 **C.A.T.** 三元组原则：

| 维度 | 要求 | 含义 |
|------|------|------|
| **C**omplete（完整） | 不可遗漏任何关键事件 | 每次工具调用、每项数据访问、每个权限决策都必须记录 |
| **A**ccurate（准确） | 记录的内容与事实严格一致 | 时间戳精确到毫秒，事件参数记录真实值，不能被 Agent 伪造 |
| **T**amper-Proof（防篡改） | 写入后不可被修改或删除 | 通过哈希链、数字签名、仅追加存储（Append-Only）等技术保证 |

从架构角度看，审计日志系统需要满足以下设计原则：

- **不可否认性（Non-Repudiation）**：日志记录者不能否认已记录的行为
- **仅追加（Append-Only）**：已有记录不能被修改或覆盖，只能追加新记录
- **时序一致性（Temporal Consistency）**：日志事件按时间顺序排列，可精确重建时间线
- **可验证性（Verifiability）**：任何第三方可以独立验证日志的完整性和真实性
- **最小性能开销（Minimal Overhead）**：审计不应成为系统瓶颈，通常要求在 5% 以内

---

## 背景

Agent 审计日志的演进经历了三个阶段：

### 第一阶段：传统系统审计日志

传统软件系统的审计日志（如数据库审计、Web 服务器 access log）主要记录：
- 用户身份和 IP 地址
- 操作类型（CRUD）
- 时间戳和请求参数

**局限**：无法表达 Agent 的推理过程、多步决策链、以及"为何做出某个工具调用"。

### 第二阶段：区块链可验证日志

受区块链技术启发，审计日志引入了哈希链和共识验证机制：
- **结构透明度（Transparency）**：如 Certificate Transparency 的 Merkle Tree 日志
- **防篡改保证**：每个日志条目通过哈希指针链接到前一条
- **可审计性**：任何参与方可以独立验证日志的完整性

**局限**：完全去中心化对于企业内部 Agent 系统来说过于重，且性能开销较大。

### 第三阶段：Agent 特定追踪日志

当前面向 AI Agent 的审计日志系统，在传统审计之上新增了：
- **思考轨迹记录（Thought Trajectory Logging）**：Agent 的推理步骤和中间结论
- **工具调用决策日志（Tool Call Decision Logging）**：为什么选择某个工具、传入了什么参数
- **权限评估记录（Permission Evaluation Records）**：每一次权限检查的输入和输出
- **多 Agent 交互追踪（Multi-Agent Interaction Tracing）**：跨 Agent 的调用链

---

## 核心矛盾

Agent 审计日志面临四组核心矛盾，需要在实际系统中做权衡：

### 1. 完整性（Completeness）vs. 存储成本（Storage Cost）

| 完整性 | 存储成本 |
|--------|---------|
| 每条思考步骤、每次 token 生成都记录 -> 几十 MB/任务 | 低成本存储只能保留摘要 |
| 仅记录工具调用和结果 -> 几 KB/任务 | 高成本存储可保留全量 |

**权衡策略**：分层采样（Adaptive Sampling）——低风险操作按 1% 采样，高风险操作 100% 记录。

### 2. 详细程度（Detail）vs. 隐私（Privacy）

详细的 Agent 日志可能包含：
- 用户对话中提到的敏感信息（PII、医疗信息、财务数据）
- Agent 内部推理中泄露的业务机密
- 工具调用时传输的 API Key 或凭证

**权衡策略**：在日志写入前做自动化 PII 检测和脱敏（Masking），保留结构但替换敏感值。

### 3. 实时性（Real-Time）vs. 性能开销（Performance）

同步写入审计日志会增加用户请求的延迟：
- 每次工具调用后等待日志落盘 -> 增加 10-50ms 延迟
- 异步批量写入 -> 可能丢失最近几秒的日志

**权衡策略**：本地缓冲区 + 批量异步刷盘，并引入 Write-Ahead Log（WAL）保证崩溃不丢数据。

### 4. 可查询性（Queryability）vs. 防篡改（Tamper-Proof）

- 为了高效查询，通常使用索引数据库（如 Elasticsearch、ClickHouse）
- 为了防篡改，日志应存储为仅追加的不可变格式

**权衡策略**：采用双存储架构——不可变原始日志用于完整性验证，可索引副本用于查询，两者定期做哈希对账。

---

## 详细内容

### 1. Audit Event Schema: 审计事件模式

所有 Agent 操作需要一个标准化的审计事件格式。推荐采用类似 **CEF（Common Event Format）** 的扩展模式：

```
事件结构：
{
  "event_id": "uuid-v7",              // 全局唯一 event ID，按时间排序
  "timestamp": "2026-07-17T08:23:19.123Z", // 精确到毫秒的 UTC 时间
  "agent_id": "agent-session-abc-123",      // Agent 会话/实例标识
  "session_id": "session-uuid",             // 用户会话 ID
  "event_type": "tool_call",                // 事件类型枚举
  "severity": "INFO",                       // 严重级别：DEBUG/INFO/WARN/ERROR/CRITICAL

  // Who — 谁发起的操作
  "actor": {
    "type": "user|agent|system",
    "id": "user-001",
    "role": "admin|operator|end_user",
    "ip_address": "192.168.1.1",
    "session_token_hash": "sha256-abc..."
  },

  // What — 做了什么
  "action": {
    "type": "tool_call|data_access|permission_check|config_change|file_read|network_request",
    "name": "read_database",
    "params": {                             // 经过脱敏的参数
      "query": "SELECT * FROM users WHERE ...",
      "limit": 100
    },
    "result_summary": "returned 50 rows",
    "status": "success|failure|blocked"
  },

  // Where — 在什么上下文中
  "context": {
    "conversation_id": "conv-456",
    "previous_events": ["event-001", "event-002"],
    "environment": "production|staging|development",
    "agent_version": "claude-4.5-haiku"
  },

  // Why — 推理依据（Agent 特有）
  "rationale": {
    "thought_snippet": "User needs user data, I have read_database tool available...",
    "confidence": 0.92,
    "alternative_actions_considered": ["search_api", "cache_lookup"],
    "permission_evaluation": {
      "policy_applied": "data_read_policy_v3",
      "allowed": true,
      "reason": "user has explicit data.read permission"
    }
  },

  // Integrity — 完整性字段
  "integrity": {
    "hash": "sha256-hash-of-this-event",
    "prev_hash": "sha256-hash-of-previous-event",
    "signature": "base64-ed25519-signature"
  }
}
```

**事件类型枚举（Event Type Taxonomy）**：

| 事件类型 | 说明 | 示例 |
|---------|------|------|
| `session_start` / `session_end` | Agent 会话生命周期 | Agent 初始化/销毁 |
| `user_message` / `agent_message` | 消息交换 | 用户输入/Agent 回复 |
| `thought_step` | Agent 内部推理步 | 思考过程、中间结论 |
| `tool_call` / `tool_result` | 工具调用及响应 | 执行 SQL、调用 API |
| `permission_check` | 权限评估 | 检查是否允许某操作 |
| `data_access` | 数据访问 | 读取文件、查询数据库 |
| `config_change` | 配置变更 | Agent 参数修改 |
| `security_event` | 安全事件 | 异常检测、越权尝试 |
| `error_event` | 错误事件 | Agent 执行错误 |
| `human_intervention` | 人工介入 | 人工审核、覆盖、终止 |

---

### 2. Log Integrity: 日志完整性

#### 哈希链（Hash Chain）

每个日志条目包含前一个条目的哈希值，形成链式结构。任何对历史条目的修改都会导致后续哈希值不匹配。

```
Block 0 (Genesis)       Block 1                 Block 2
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│ event_id: 0  │         │ event_id: 1  │         │ event_id: 2  │
│ timestamp    │         │ timestamp    │         │ timestamp    │
│ data         │  ────→  │ data         │  ────→  │ data         │
│ prev_hash: 0 │         │ prev_hash: H0│         │ prev_hash: H1│
│ hash: H0     │         │ hash: H1     │         │ hash: H2     │
└─────────────┘         └─────────────┘         └─────────────┘
```

**哈希链验证算法**：
1. 从最新事件开始，向前遍历整个链
2. 对每个事件验证 `hash(event) == event.hash`
3. 验证 `event.prev_hash == previous_event.hash`
4. 如果所有验证通过，日志未被篡改

#### 数字签名（Digital Signature）

在每个日志条目或每批日志块上附加 Ed25519 或 ECDSA 签名：
- 私钥由专门的安全模块（HSM 或 Secure Enclave）保管
- 公钥公开发布，供审计方验证
- 时间戳由可信时间源（RFC 3161 TSA）签名，防止时间伪造

#### 仅追加存储（Append-Only Storage）

- 使用只追加文件（Append-Only File, AOF）格式
- 底层文件系统设置为 Immutable（Linux `chattr +a`）
- 数据库层面使用不支持 UPDATE/DELETE 的存储引擎（或通过触发器阻止）
- 云存储使用 Object Lock（S3 Object Lock / Azure WORM）

#### 定期验证（Periodic Verification）

- 每小时/每天对日志哈希链做全量验证
- 生成验证报告和 Merkle 根哈希
- 将 Merkle 根哈希发布到公开可验证的媒介（区块链、DNS TXT 记录、公共 DB）
- 实现**可审计的透明性（Auditable Transparency）**

---

### 3. Log Storage: 日志存储与分层

#### 分层存储架构（Tiered Storage）

| 层级 | 存储介质 | 查询性能 | 保留期 | 用途 |
|------|---------|---------|--------|------|
| **Hot（热）** | SSD / RAM（如 Redis Streams） | <10ms | 7 天 | 实时告警、在线查询 |
| **Warm（温）** | HDD / 本地磁盘（如 ClickHouse） | <100ms | 90 天 | 日常审计、调查分析 |
| **Cold（冷）** | S3 / Azure Blob / Glacier | 分钟级 | 1-7 年 | 合规归档、法律取证 |

#### 生命周期策略

```
写缓冲区 (Buffer) → Hot Tier (7d) → Compaction → Warm Tier (90d) → Compression → Cold Tier (6y) → Deletion
```

- **Compaction**：合并小文件，对日志做索引优化
- **Compression**：使用 Zstd 或 LZ4 压缩，通常可将日志压缩至原始大小的 10-20%
- **Encryption**：冷存储层必须使用 KMS 管理的加密密钥

#### 日志轮转（Log Rotation）

- 按大小（每 500MB）或按时间（每小时）轮转
- 轮转前的日志做哈希链完整性检查
- 使用预分配文件（Pre-Allocation）防止碎片化
- 保留最近 2 个轮转文件作为 Write-Ahead Log 崩溃恢复

---

### 4. Agent-Specific Audit: Agent 特有审计

这是 Agent 审计日志区别于传统审计的核心部分。

#### 思考轨迹日志（Thought Trajectory Logging）

记录 Agent 的每一步内部推理：

```json
{
  "event_type": "thought_step",
  "step_number": 7,
  "parent_step": 6,
  "content": "User wants to find orders from last month. I need to query the orders table.",
  "attention_focus": "order_date >= '2026-06-01'",
  "confidence": 0.85,
  "tokens_used": 142,
  "model": "claude-4-opus",
  "temperature": 0.3
}
```

**关键考虑**：
- 思考轨迹是 Agent 行为最详细的证据，但也是隐私风险最高的部分
- 建议对思考轨迹做层级化采样：完整保留高风险操作，摘要化记录低风险操作
- 思考轨迹日志应有独立的访问控制和审批流程

#### 工具调用决策日志（Tool Call Decision Logging）

```json
{
  "event_type": "tool_decision",
  "tool_selected": "query_database",
  "alternatives_considered": [
    {"tool": "search_api", "score": 0.3, "reason": "scored lower on relevance"},
    {"tool": "cache_lookup", "score": 0.5, "reason": "cache miss expected"}
  ],
  "selection_reason": "user explicitly requested raw data access, database query most appropriate",
  "decision_model": "router-v2",
  "latency_ms": 45
}
```

#### 权限评估记录（Permission Evaluation Records）

Agent 每次权限检查的完整审计：

```json
{
  "event_type": "permission_evaluation",
  "request": {
    "action": "read",
    "resource": "database.orders",
    "context": "user request for order analysis"
  },
  "policies_evaluated": ["rbac_order_view_v2", "data_classification_l3"],
  "result": "deny",
  "reason": "user role 'viewer' does not have read access to orders table",
  "override_available": true,
  "escalation_path": "request_approval from admin"
}
```

#### 多 Agent 协作追踪（Multi-Agent Interaction Tracing）

当系统包含多个 Agent 时，需要分布式追踪：
- 每个 Agent 会话有一个唯一的 `trace_id`
- Agent 之间的调用通过 `span_id` 和 `parent_span_id` 关联
- 使用类似 OpenTelemetry 的追踪语义

---

### 5. Log Query & Analysis: 日志查询与分析

#### 结构化查询接口

```sql
-- 示例查询：查找所有高风险操作
SELECT * FROM audit_log
WHERE severity IN ('WARN', 'ERROR', 'CRITICAL')
  AND timestamp >= NOW() - INTERVAL 24 HOUR
ORDER BY timestamp DESC;

-- 查询特定 Agent 会话的完整轨迹
SELECT * FROM audit_log
WHERE session_id = 'session-abc-123'
ORDER BY timestamp;

-- 查找越权尝试
SELECT * FROM audit_log
WHERE event_type = 'permission_evaluation'
  AND result = 'deny'
  AND context.environment = 'production';
```

#### 轨迹重建（Trace Reconstruction）

从审计日志重建 Agent 的完整决策路径：

```
用户输入
  └─ [thought_step #1] 分析用户意图
      └─ [tool_decision] 选择 search_knowledge_base
          ├─ [tool_call] search_knowledge_base(query="user data policy")
          │   └─ [tool_result] 返回 3 条相关文档
          └─ [thought_step #2] 评估结果
              └─ [tool_decision] 选择 write_response
                  └─ [tool_call] write_response(...)
                      └─ [agent_message] 回复用户
```

#### 时间线可视化

审计日志可以生成时间线视图，方便人工审查：
- 每个事件用不同颜色标识类型
- 事件之间的因果关系用箭头连接
- 异常事件用红色高亮
- 汇总统计：工具调用频率、平均响应时间、错误率

---

### 6. Audit Alerting: 审计告警

#### 基于阈值的告警（Threshold-Based Detection）

| 告警规则 | 阈值 | 严重级别 |
|---------|------|---------|
| 单位时间内相同操作重复次数过多 | >50次/分钟 | WARNING |
| 连续权限拒绝 | >5次/分钟 | CRITICAL |
| 异常时间段操作（如凌晨 2-5 点） | 非工作时间大量操作 | INFO |
| 数据导出量异常 | >1000行/次 | WARNING |
| Agent 会话持续时间异常 | >4小时 | INFO |

#### 基于机器学习的检测（ML-Based Detection）

- **行为基线建模**：为每个用户/Agent 建立正常行为模式
- **异常检测**：偏离基线 3 个标准差的行为触发告警
- **时序异常**：操作频率、操作序列的异常变化
- **图异常检测**：在 Agent 调用关系图中检测异常路径

#### 告警生命周期

```
触发条件 → 初步评估（自动） → 通知安全团队 → 人工调查 → 确认/误报 → 事件记录 → 规则调优
```

---

### 7. Privacy in Audit Logs: 审计日志中的隐私保护

#### PII 脱敏（PII Masking）

```python
# 日志写入前的自动化 PII 检测与脱敏
def mask_pii(event: dict) -> dict:
    # 替换电子邮件
    event = re.sub(r'\b[\w\.-]+@[\w\.-]+\.\w+\b', '***@***.***', str(event))
    # 替换电话号码
    event = re.sub(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b', '***-***-****', str(event))
    # 替换身份证/SSN
    event = re.sub(r'\b\d{3}-\d{2}-\d{4}\b', '***-**-****', str(event))
    # 从 params 和 result 中移除标记为 sensitive 的字段
    for field in ['api_key', 'password', 'token', 'secret', 'credit_card']:
        event['action']['params'].pop(field, None)
    return event
```

**重要原则**：
- 脱敏应在日志写入前完成（不在存储后做后处理）
- 使用确定性脱敏（Deterministic Masking）保持相同值的可关联性
- 敏感字段的原始值如果需要保留，应单独加密存储（有独立的访问审批流程）
- 考虑使用格式保持加密（Format-Preserving Encryption, FPE）替代简单掩码

#### 数据最小化（Data Minimization）

- 只记录完成审计目标所必需的最小数据集
- Agent 的完整思考轨迹默认不记录，仅在安全模式或特定会话中按需开启
- 工具调用参数默认为摘要模式（记录参数类型而非具体值）
- 引入"审计级别（Audit Level）"概念：`none | minimal | standard | verbose`

#### 传输与存储加密

| 阶段 | 加密措施 |
|------|---------|
| 传输中（In Transit） | mTLS + 双向认证 |
| 存储中（At Rest） | AES-256-GCM，密钥由 KMS 管理 |
| 密钥管理 | 定期轮转、HSM 保护、访问审计 |
| 日志备份 | 使用独立的加密密钥 |

---

## Example Code: Python AgentAuditLogger

以下实现演示了一个完整的 Agent 审计日志系统，包含哈希链、篡改检测和结构化事件模式。

```python
"""
AgentAuditLogger — tamper-proof audit logging for AI Agent systems.

Features:
- Hash-chain integrity verification
- Digital signature support (Ed25519)
- Structured event schema
- Batched async writing
- Tamper detection and reporting
"""

import hashlib
import json
import time
import uuid
from datetime import datetime, timezone
from typing import Any, Optional
from dataclasses import dataclass, field, asdict
from collections import deque


# ─── Event Schema ────────────────────────────────────────────────────────────

@dataclass
class AuditEvent:
    """Standardized audit event for Agent actions."""
    event_id: str
    timestamp: str
    agent_id: str
    session_id: str
    event_type: str          # tool_call, thought_step, permission_evaluation, etc.
    severity: str            # DEBUG, INFO, WARN, ERROR, CRITICAL

    actor_type: str          # user, agent, system
    actor_id: str
    actor_role: str

    action_type: str         # read, write, execute, config_change, etc.
    action_name: str
    action_params: dict
    action_status: str       # success, failure, blocked

    rationale_thought: Optional[str] = None
    permission_allowed: Optional[bool] = None
    permission_policy: Optional[str] = None

    prev_hash: str = ""
    hash: str = ""
    signature: str = ""

    def compute_hash(self) -> str:
        """Compute SHA-256 hash of this event's content (excluding signature)."""
        d = asdict(self)
        d.pop('hash', None)
        d.pop('signature', None)
        # Ensure deterministic serialization
        raw = json.dumps(d, sort_keys=True, ensure_ascii=False, default=str)
        return hashlib.sha256(raw.encode('utf-8')).hexdigest()

    def to_json(self) -> str:
        return json.dumps(asdict(self), ensure_ascii=False, default=str)


# ─── Audit Logger ────────────────────────────────────────────────────────────

class AgentAuditLogger:
    """
    Tamper-proof audit logger with hash-chain integrity.

    Usage:
        logger = AgentAuditLogger(agent_id="my-agent-v1")
        logger.log_event(
            event_type="tool_call",
            actor_id="user-001",
            action_name="read_database",
            action_params={"query": "SELECT 1"},
            action_status="success",
        )
        # Check integrity
        report = logger.verify_integrity()
    """

    def __init__(
        self,
        agent_id: str,
        session_id: Optional[str] = None,
        storage_backend: Any = None,  # e.g., file, database, S3
        signing_key: Optional[bytes] = None,
        batch_size: int = 50,
        flush_interval_seconds: float = 5.0,
    ):
        self.agent_id = agent_id
        self.session_id = session_id or str(uuid.uuid4())
        self.storage = storage_backend
        self.signing_key = signing_key
        self.batch_size = batch_size
        self.flush_interval = flush_interval_seconds

        self._chain: list[AuditEvent] = []
        self._buffer: deque[AuditEvent] = deque()
        self._last_hash: str = hashlib.sha256(b"genesis").hexdigest()
        self._event_count = 0
        self._start_time = time.time()

    def log_event(
        self,
        event_type: str,
        actor_id: str,
        action_name: str,
        action_params: Optional[dict] = None,
        action_status: str = "success",
        action_type: str = "execute",
        severity: str = "INFO",
        actor_type: str = "user",
        actor_role: str = "end_user",
        rationale_thought: Optional[str] = None,
        permission_allowed: Optional[bool] = None,
        permission_policy: Optional[str] = None,
    ) -> str:
        """Create and record a structured audit event. Returns event_id."""
        event_id = str(uuid.uuid4())
        timestamp = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.%f")[:-3] + "Z"

        event = AuditEvent(
            event_id=event_id,
            timestamp=timestamp,
            agent_id=self.agent_id,
            session_id=self.session_id,
            event_type=event_type,
            severity=severity,
            actor_type=actor_type,
            actor_id=actor_id,
            actor_role=actor_role,
            action_type=action_type,
            action_name=action_name,
            action_params=action_params or {},
            action_status=action_status,
            rationale_thought=rationale_thought,
            permission_allowed=permission_allowed,
            permission_policy=permission_policy,
            prev_hash=self._last_hash,
        )

        # Compute hash and chain
        event.hash = event.compute_hash()
        self._last_hash = event.hash

        # Sign if key available
        if self.signing_key:
            from nacl.signing import SigningKey
            sk = SigningKey(self.signing_key)
            signed = sk.sign(event.hash.encode('utf-8'))
            event.signature = signed.signature.hex()

        # Add to buffer
        self._buffer.append(event)
        self._chain.append(event)
        self._event_count += 1

        # Flush if batch threshold reached
        if len(self._buffer) >= self.batch_size:
            self.flush()

        return event_id

    def flush(self) -> int:
        """Flush buffered events to storage. Returns number of flushed events."""
        if not self._buffer:
            return 0

        batch = []
        while self._buffer:
            batch.append(self._buffer.popleft())

        if self.storage:
            records = [e.to_json() for e in batch]
            self.storage.write(records)

        return len(batch)

    def verify_integrity(self) -> dict:
        """
        Verify the integrity of the entire hash chain.

        Returns a report dict:
        {
            "valid": True/False,
            "total_events": N,
            "broken_links": [(index, expected_hash, actual_hash), ...],
            "first_break_index": N or None,
        }
        """
        report = {
            "valid": True,
            "total_events": len(self._chain),
            "broken_links": [],
            "first_break_index": None,
        }

        if not self._chain:
            return report

        # Verify genesis block
        prev_hash = hashlib.sha256(b"genesis").hexdigest()
        if self._chain[0].prev_hash != prev_hash:
            report["valid"] = False
            report["broken_links"].append((
                0,
                prev_hash,
                self._chain[0].prev_hash,
            ))

        # Walk the chain
        for i, event in enumerate(self._chain):
            # Verify prev_hash matches previous event's hash
            if event.prev_hash != prev_hash:
                report["valid"] = False
                report["broken_links"].append((
                    i,
                    prev_hash,
                    event.prev_hash,
                ))
                if report["first_break_index"] is None:
                    report["first_break_index"] = i

            # Verify event's own hash is correct
            computed = event.compute_hash()
            if computed != event.hash:
                report["valid"] = False
                report["broken_links"].append((
                    i,
                    computed,
                    event.hash,
                ))
                if report["first_break_index"] is None:
                    report["first_break_index"] = i

            # Update running hash
            prev_hash = event.hash

        return report

    def get_tamper_report(self) -> str:
        """Generate a human-readable tamper detection report."""
        report = self.verify_integrity()
        lines = [
            "=" * 60,
            f"Audit Log Integrity Report",
            "=" * 60,
            f"Agent ID:       {self.agent_id}",
            f"Session ID:     {self.session_id}",
            f"Total Events:   {report['total_events']}",
            f"Integrity Valid: {report['valid']}",
        ]
        if report['broken_links']:
            lines.append(f"\nBroken Links ({len(report['broken_links'])} found):")
            for idx, exp, actual in report['broken_links'][:10]:
                lines.append(f"  Event #{idx}: expected={exp[:16]}..., actual={actual[:16]}...")
        return "\n".join(lines)

    @property
    def performance_metrics(self) -> dict:
        elapsed = time.time() - self._start_time
        return {
            "total_events": self._event_count,
            "elapsed_seconds": round(elapsed, 2),
            "events_per_second": round(self._event_count / elapsed, 2) if elapsed > 0 else 0,
            "buffered_events": len(self._buffer),
        }


# ─── Usage Example ───────────────────────────────────────────────────────────

if __name__ == "__main__":
    import tempfile

    # Simulated storage backend
    class FileStorage:
        def __init__(self, path: str):
            self.path = path
        def write(self, records: list[str]):
            with open(self.path, "a", encoding="utf-8") as f:
                for r in records:
                    f.write(r + "\n")

    with tempfile.NamedTemporaryFile(mode="w", suffix=".audit", delete=False) as f:
        store = FileStorage(f.name)

    logger = AgentAuditLogger(
        agent_id="research-agent-v2",
        session_id="sess-20260717-001",
        storage_backend=store,
        batch_size=5,
    )

    # Simulate an Agent session
    session_events = [
        ("session_start", "system", "Agent initialized", "success"),
        ("user_message", "user-001", "Analyze Q3 sales data", "success"),
        ("thought_step", "agent", "User wants sales analysis…", "success"),
        ("tool_call", "agent", "query_database(q='SELECT * FROM sales WHERE quarter=3')", "success"),
        ("tool_result", "agent", "Returned 1000 rows", "success"),
        ("thought_step", "agent", "Data retrieved, now generating chart", "success"),
        ("tool_call", "agent", "generate_chart(type='bar')", "success"),
        ("agent_message", "agent", "Here is the Q3 sales analysis…", "success"),
        ("permission_evaluation", "system", "Check data export permission", "blocked"),
        ("security_event", "system", "Multiple permission denials detected", "failure"),
    ]

    for e_type, actor, desc, status in session_events:
        logger.log_event(
            event_type=e_type,
            actor_id=actor,
            action_name=desc,
            action_status=status,
        )

    # Flush remaining buffered events
    flushed = logger.flush()
    print(f"Flushed {flushed} events to storage")

    # Verify integrity
    print(logger.get_tamper_report())

    # Performance metrics
    print(f"\nPerformance: {logger.performance_metrics}")

    # ── Simulate tampering ──
    # Modify the 4th event's params (index 3, tool_call)
    tampered = logger._chain[3]
    tampered.action_params["_tampered"] = True
    tampered.hash = tampered.compute_hash()  # Attacker recomputes hash but does NOT update subsequent chain
    # Now verify
    print("\n" + logger.get_tamper_report())
    # The verification should detect:
    #   - Event #3: hash mismatch (tampered)
    #   - Event #4-#9: prev_hash mismatch (chain broken)
```

---

## Capability Boundaries: 能力边界

审计日志系统的安全性取决于其最薄弱的环节。以下是可能被攻击的点：

### 1. 采集阶段（Collection）

| 攻击面 | 描述 | 缓解措施 |
|--------|------|---------|
| Agent 绕过日志 | Agent 在执行操作前关闭日志功能 | 使用内核级强制日志钩子（eBPF / Auditd），Agent 无法干预 |
| 日志注入 | 攻击者向日志中写入伪造事件混淆真实事件 | 输入净化、日志格式严格校验、使用非可打印字符转义 |
| 日志丢弃 | Agent 在高频操作中使日志缓冲区溢出导致事件丢失 | 反压机制（Backpressure）、日志采样降级而非丢弃 |
| 时间戳伪造 | Agent 报送虚假时间戳 | 使用可信时间源（NTS / TSA），日志服务器统一加盖时间戳 |

### 2. 存储阶段（Storage）

| 攻击面 | 描述 | 缓解措施 |
|--------|------|---------|
| 日志修改 | 攻击者直接修改已存储的日志文件 | 哈希链、文件系统 Immutable 属性、定期外部验证 |
| 日志删除 | 攻击者删除日志文件掩盖痕迹 | 仅追加存储、WORM 存储、异地备份 |
| 密钥泄露 | 签名私钥泄露导致日志可被伪造 | HSM 保护、定期轮转、密钥泄露检测和应急响应 |
| 存储介质故障 | 硬盘损坏导致日志丢失 | RAID、异地多副本、定期备份验证 |

### 3. 传输阶段（Transmission）

| 攻击面 | 描述 | 缓解措施 |
|--------|------|---------|
| 中间人攻击 | 日志在传输过程中被截获或篡改 | mTLS、端到端加密 |
| 重放攻击 | 攻击者重新发送旧的日志事件 | Nonce + 时间戳窗口验证 |
| 日志拦截 | 攻击者阻止日志到达存储端 | 异步确认机制、失败重试、本地持久化缓存 |

### 4. 查询与展示阶段（Query & Display）

| 攻击面 | 描述 | 缓解措施 |
|--------|------|---------|
| 未授权访问 | 非审计人员查看日志内容 | RBAC、日志访问审计（日志的日志）|
| SQL 注入 | 通过查询接口攻击日志数据库 | 参数化查询、只读账户、查询限流 |
| 日志泄露 | 通过错误信息或调试接口泄露日志内容 | 生产环境禁用详细错误、日志查看器不展示敏感字段 |

---

## Comparison: Agent Audit vs. Traditional System Audit

| 维度 | 传统系统审计（System Audit） | Agent 审计（Agent Audit） |
|------|----------------------------|--------------------------|
| **审计对象** | 用户操作、系统事件、API 调用 | Agent 思考轨迹、工具调用决策、权限评估链 |
| **事件粒度** | 粗粒度（HTTP 请求、SQL 查询） | 细粒度（每一步推理、每个候选工具评分） |
| **因果关系** | 简单的请求-响应关系 | 复杂的多步推理链 + 分支决策树 |
| **日志量级** | MB/天（企业级） | GB/天（单个 Agent 可能每秒钟产生多个事件） |
| **可预测性** | 操作是可枚举、可预定义的 | Agent 操作可能是不可预见的（尤其在开放式环境中） |
| **语义丰富度** | 结构化字段（时间、IP、URL） | 包含自然语言推理过程、置信度评分、替代方案 |
| **隐私风险** | 较低（通常为结构化元数据） | 较高（可能包含完整的用户对话和内部推理） |
| **验证难度** | 验证操作是否被执行 | 验证 Agent "为什么"做出某个决定——需要评估推理合理性 |
| **合规要求** | SOX、PCI-DSS、HIPAA | 尚在演进中（NIST AI RMF、EU AI Act 正在建立规范） |
| **存储策略** | 集中式、长期归档 | 需要分层存储（热/温/冷）加上思考轨迹特殊处理 |
| **告警模式** | 基于签名匹配（已知攻击模式） | 需要行为基线 + 异常检测 + 语义理解 |

### Agent 审计特有的需求

传统审计不需要、但 Agent 审计必须解决的三个独特挑战：

1. **推理链可审计性**：不是简单记录"Agent 调用了工具 X"，而是记录"Agent 为什么选择工具 X 而不是工具 Y"，这要求捕获决策过程中的替代方案和评分。

2. **语义压缩**：Agent 的思考轨迹是自然语言文本，直接存储会导致存储爆炸。需要有效的摘要和压缩策略，同时保留可用于验证的语义信息。

3. **行为边界验证**：传统审计只需要验证"操作是否被允许"，Agent 审计需要验证"Agent 是否在授权的行为边界内操作"——即使每个操作单独看是合法的，操作序列可能构成越权行为。

---

## Engineering Optimization: 工程优化

### 1. 异步批量写入（Async Batch Writing）

```python
import asyncio
from collections import deque

class AsyncAgentAuditLogger:
    """Async version with batched writes."""

    def __init__(self, max_batch_size=100, flush_interval=1.0):
        self._buffer = deque()
        self._max_batch_size = max_batch_size
        self._flush_interval = flush_interval
        self._flush_task = None

    async def start(self):
        self._flush_task = asyncio.create_task(self._periodic_flush())

    async def stop(self):
        if self._flush_task:
            self._flush_task.cancel()
        await self.flush()

    async def log_event(self, event: AuditEvent):
        self._buffer.append(event)
        if len(self._buffer) >= self._max_batch_size:
            await self.flush()

    async def flush(self):
        if not self._buffer:
            return
        batch = list(self._buffer)
        self._buffer.clear()
        # Bulk write — single I/O call for many events
        await self._storage.bulk_write(batch)

    async def _periodic_flush(self):
        while True:
            await asyncio.sleep(self._flush_interval)
            await self.flush()
```

**收益**：
- 高吞吐场景下，批量写入相比逐条写入提升 10-50 倍
- 减少 I/O 次数，降低存储系统负载
- 有利于压缩和索引优化

### 2. 压缩存储（Compressed Storage）

| 压缩策略 | 压缩比 | CPU 开销 | 适用场景 |
|---------|--------|---------|---------|
| Gzip | 5-10x | 中等 | 冷存储归档 |
| Zstd | 8-15x | 低 | 温/冷存储，平衡压缩比和速度 |
| LZ4 | 2-4x | 极低 | 热存储，追求写入速度 |
| Columnar（Parquet/ORC） | 10-20x | 较高 | 分析查询优化场景 |

**推荐实践**：
- 热存储使用 LZ4 或不做压缩（追求写入吞吐）
- 温存储使用 Zstd（平衡压缩比和查询速度）
- 冷存储使用 Zstd 最大压缩级别（追求最小存储空间）

### 3. 日志采样策略（Log Sampling Strategies）

```python
class AdaptiveSampler:
    """
    Adaptive sampling based on risk level.

    High-risk operations:  always sampled (rate=1.0)
    Medium-risk:           adaptive based on current system load
    Low-risk:              fixed low sampling rate
    """

    def __init__(self, base_rate: float = 0.01):
        self.base_rate = base_rate
        self.current_load = 0.0

    def should_sample(self, event_type: str, severity: str) -> bool:
        # Always log critical/error events
        if severity in ('ERROR', 'CRITICAL'):
            return True

        # Always log security events
        if event_type in ('security_event', 'permission_evaluation'):
            return True

        # Always log tool calls with destructive potential
        if event_type == 'tool_call' and severity == 'WARN':
            return True

        # Adaptive sampling for low-risk events
        if self.current_load > 0.8:
            return random.random() < self.base_rate * 0.5
        return random.random() < self.base_rate

    def update_load(self, cpu_percent: float):
        self.current_load = cpu_percent / 100.0
```

| 采样类型 | 策略 | 风险 |
|---------|------|------|
| 固定比率采样 | 随机保留 1% 日志 | 可能丢失关键事件 |
| 自适应采样 | 低负载时提高采样率，高负载时降低 | 更智能，但仍可能丢失 |
| 分层采样 | 高/中/低风险不同采样率 | 常用策略，推荐 |
| 重要性采样 | 基于事件重要性评分动态调整 | 实现复杂，但效果最好 |

### 4. 缓冲与崩溃安全（Buffering & Crash Safety）

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│ Agent 调用   │ ──→ │ WAL Buffer   │ ──→ │ 持久化存储   │
│ log_event()  │     │ (本地磁盘)    │     │ (S3/DB/File) │
└─────────────┘     └──────────────┘     └──────────────┘
                          │
                    ┌─────┴─────┐
                    │ 崩溃恢复   │
                    │ 从 WAL     │
                    │ 重放未提交  │
                    │ 的日志     │
                    └───────────┘
```

- **Write-Ahead Log（WAL）**：在写入主存储前，先将日志写入本地 WAL 文件
- **崩溃恢复**：系统重启后检查 WAL，将未提交的日志重新发送到主存储
- **去重机制**：在日志事件中包含唯一 ID（UUID v7），存储端做幂等去重

### 5. 索引优化（Indexing Optimization）

```sql
-- 审计日志查询的核心索引策略
CREATE TABLE audit_log (
    event_id     UUID PRIMARY KEY,
    timestamp    DateTime64(3) CODEC(Zstd),
    agent_id     LowCardinality(String),
    session_id   UUID,
    event_type   LowCardinality(String),
    severity     LowCardinality(String),
    actor_id     String,
    actor_type   LowCardinality(String),
    action_name  LowCardinality(String),
    action_status LowCardinality(String),
    prev_hash    FixedString(64),
    hash         FixedString(64),
    payload      String CODEC(Zstd)   -- 详细 payload 单独压缩存储
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, event_type)
TTL timestamp + INTERVAL 90 DAY TO VOLUME 'cold';
```

### 6. 性能基准参考

| 操作 | 延迟（P50） | 延迟（P99） | 吞吐 |
|------|-----------|-----------|------|
| 单条日志记录（同步写文件） | 0.5ms | 2ms | 2,000 events/s |
| 批量写入（100条/批，同步） | 5ms | 15ms | 20,000 events/s |
| 批量写入（异步，100条/批） | 0.8ms | 5ms | 100,000+ events/s |
| 哈希链验证（10,000条） | 150ms | 300ms | — |
| 日志查询（带索引，7天范围） | 20ms | 100ms | — |

---

## 总结

审计日志是 AI Agent 安全体系的基础设施，它通过在"谁在什么时候做了什么以及为什么"这个核心问题上提供不可否认的证据，支撑着整个安全体系的可信度。

设计 Agent 审计日志系统时，应牢牢把握以下原则：

1. **防篡改是底线**：哈希链 + 数字签名 + 仅追加存储是必须的，不应妥协
2. **Agent 语义是核心**：记录思考轨迹、决策原因、替代方案评估是 Agent 审计区别于传统审计的关键
3. **分层是经济之道**：热/温/冷分层存储 + 自适应采样，在安全需求和成本之间取得平衡
4. **隐私是设计约束**：PII 脱敏应在写入前完成，最小化数据采集范围
5. **可验证性需端到端**：从采集、存储到查询，每个环节都需要可验证的完整性保证
