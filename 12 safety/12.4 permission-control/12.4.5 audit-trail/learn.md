# 12.4.5 audit-trail — 权限审计轨迹

## 简单介绍

权限审计轨迹（Audit Trail）是指对 Agent 系统中每一次权限决策——授予、拒绝、升级、降级——进行完整记录，形成不可篡改的审计日志链。这些日志不仅用于事后追溯与合规审查，还通过实时分析和异常检测，成为主动防御的关键数据源。

在 AI Agent 系统中，审计轨迹回答一个核心问题：**谁、在什么时候、因为什么、请求了什么权限，结果是允许还是拒绝？**

## 基本原理

审计轨迹的核心原则很简单但至关重要：**每一个权限决策都必须被记录，以供后续分析。**

```
权限决策发生
    │
    ▼
┌─────────────────────────────────────┐
│  1. 捕获决策事件                     │
│     • 谁发起的请求（用户 ID / Session ID）
│     • 请求了什么（工具 / 资源 / 操作）
│     • 决策结果（允许 / 拒绝 / 升级）
│     • 决策依据（哪条策略？风险评分？）
│     • 时间戳（精确到毫秒）
├─────────────────────────────────────┤
│  2. 构建不可篡改的日志条目           │
│     • 序列化事件数据                 │
│     • 计算哈希值 + 链接前一条日志    │
│     • 可选：数字签名                 │
├─────────────────────────────────────┤
│  3. 持久化存储                       │
│     • 写入热存储（内存 / SSD）       │
│     • 批量刷入温存储（数据库）        │
│     • 归档至冷存储（对象存储）        │
├─────────────────────────────────────┤
│  4. 分析与告警                       │
│     • 实时异常检测                   │
│     • 定期合规报告                   │
│     • 安全事件回溯                   │
└─────────────────────────────────────┘
```

审计日志与普通日志的区别：

| 维度 | 普通日志 | 审计日志 |
|------|---------|---------|
| 用途 | 调试、监控 | 合规、安全、取证 |
| 准确性 | 允许丢失 | 不允许丢失/篡改 |
| 不可篡改性 | 无要求 | 必须保证 |
| 存储周期 | 天/周级 | 月/年级 |
| 访问权限 | 开发人员可读 | 仅审计员 + 合规工具可读 |
| 格式 | 自由文本 | 结构化、Schema 严格 |

## 背景

### Regulatory Requirements（法规驱动）

审计日志不是可选项——在多数据保护法规中，它是强制性要求：

- **GDPR 第 5 条**：要求对个人数据处理活动进行记录，包括访问权限的授予和使用
- **SOC 2**：明确要求对系统访问、权限变更进行审计跟踪
- **HIPAA**：要求对受保护健康信息（PHI）的每一次访问进行记录
- **PCI DSS 第 10 条**：要求对所有访问支付系统环境的行为进行审计追踪
- **CCPA/CPRA**：要求记录消费者数据访问和删除请求的处理过程

### Internal Security（内部安全需求）

法规之外，审计轨迹也是内部安全基础设施的基石：

- **事后取证**：安全事故发生后，审计日志是还原攻击路径的唯一可靠来源
- **责任认定**：明确某个权限决策由谁做出、基于什么策略，避免责任模糊
- **权限优化**：通过分析审计数据，发现过度授权或权限滥用，持续优化权限策略
- **内部合规**：满足组织内部的安全审计和合规检查要求

### Proactive Threat Detection（主动威胁检测）

现代审计系统正从事后追溯转向实时防御：

- **异常行为检测**：基于审计日志的实时流式分析，在攻击进行中即发出告警
- **权限爬坡检测**：检测 Agent 或用户短时间内权限快速升级的模式（Privilege Escalation）
- **横向移动检测**：通过跨 Session 的审计关联，发现攻击者的横向移动行为
- **零信任验证**：每一次权限请求都被记录和验证，为零信任架构提供数据支持

## 核心矛盾

审计轨迹系统设计面临多重矛盾，需要在架构中做出权衡：

### 1. 详细日志 vs 隐私保护

| 矛盾 | 说明 |
|------|------|
| 审计需要详细 | 为了有效审计，需要记录请求的具体参数、操作的目标资源、甚至是请求体的内容 |
| 隐私要求最小化 | GDPR 等法规要求数据最小化，不应记录不必要的个人信息 |
| 平衡方案 | 记录事件类型和元数据但不记录具体内容；对敏感字段进行 Tokenization 或匿名化 |

**实践建议**：审计日志中不记录请求的 payload 内容，只记录 metadata（操作类型、资源 ID、决策结果）。需要 payload 时，通过引用 ID 从安全存储中按需拉取。

### 2. 存储成本 vs 保留期限

| 矛盾 | 说明 |
|------|------|
| 合规要求长期保留 | SOC 2 要求至少 12 个月，金融行业要求 5-7 年，部分法规要求永久保留 |
| 审计日志体积巨大 | 一个中等规模的 Agent 系统每天可产生数亿条审计事件 |
| 平衡方案 | 分层存储策略：热存储（7天）→ 温存储（3个月）→ 冷存储（数年） |

**实践建议**：高吞吐场景下使用日志采样 + 聚合，关键安全事件（拒绝、升级、异常）全量保留，低风险事件（常规允许）采样保留。

### 3. 实时性 vs 性能开销

| 矛盾 | 说明 |
|------|------|
| 审计需要实时 | 安全告警需要准实时（秒级延迟），否则攻击发生时无法及时响应 |
| 同步写入影响性能 | 每次权限决策都同步写入审计日志会显著增加延迟，High-throughput 场景下不可接受 |
| 平衡方案 | 异步批量写入 + 内存缓冲，关键事件同步写入，常规事件异步写入 |

**实践建议**：区分事件优先级——关键事件（DENY、ESCALATION）同步写入确保不丢失；常规事件（ALLOW）异步批量写入。

### 4. 不可篡改 vs 存储灵活性

| 矛盾 | 说明 |
|------|------|
| 审计日志需要防篡改 | 哈希链、数字签名、WORM（Write Once Read Many）存储 |
| 日志需要可管理 | 日志轮转、压缩、删除过期数据、迁移存储层 |
| 平衡方案 | 哈希链 + 签名保证完整性；分段存储允许按 segment 管理生命周期 |

**实践建议**：将审计日志切分为固定大小的 segments（如每小时一个 segment），每个 segment 有独立的哈希链。Segment 过期后可以安全删除而不会破坏整体链的完整性。

## 详细内容

### 1. Permission Audit Event Schema（权限审计事件 Schema）

审计事件需要回答 Who / What / When / Where / Why 五个问题：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "PermissionAuditEvent",
  "type": "object",
  "required": [
    "event_id", "timestamp", "event_type",
    "actor", "resource", "decision",
    "session_id"
  ],
  "properties": {
    "event_id": {
      "type": "string",
      "description": "全局唯一事件 ID (ULID / UUIDv7)"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "事件发生时间（UTC，毫秒精度）"
    },
    "event_type": {
      "type": "string",
      "enum": [
        "permission_granted",
        "permission_denied",
        "permission_escalated",
        "permission_revoked",
        "permission_expired",
        "approval_requested",
        "approval_granted",
        "approval_denied",
        "role_assigned",
        "role_removed"
      ]
    },
    "actor": {
      "type": "object",
      "required": ["id", "type"],
      "properties": {
        "id": { "type": "string" },
        "type": { "type": "string", "enum": ["user", "agent", "service", "system"] },
        "session_id": { "type": "string" },
        "ip_address": { "type": "string", "format": "ipv4" },
        "user_agent": { "type": "string" }
      }
    },
    "resource": {
      "type": "object",
      "required": ["type", "id"],
      "properties": {
        "type": { "type": "string", "enum": ["tool", "api", "data", "file", "system"] },
        "id": { "type": "string" },
        "name": { "type": "string" },
        "action": { "type": "string" }
      }
    },
    "decision": {
      "type": "object",
      "required": ["result", "reason"],
      "properties": {
        "result": { "type": "string", "enum": ["allow", "deny", "escalate", "require_approval"] },
        "reason": { "type": "string" },
        "policy_id": { "type": "string" },
        "risk_score": { "type": "number", "minimum": 0, "maximum": 1 },
        "matched_rules": { "type": "array", "items": { "type": "string" } }
      }
    },
    "context": {
      "type": "object",
      "properties": {
        "conversation_id": { "type": "string" },
        "input_tokens": { "type": "integer" },
        "previous_actions": { "type": "array", "items": { "type": "string" } },
        "environment": { "type": "string" }
      }
    }
  }
}
```

**事件类型详解**：

| 事件类型 | 触发条件 | 示例场景 |
|---------|---------|---------|
| `permission_granted` | 权限检查通过 | Agent 请求搜索工具 → 自动授予 |
| `permission_denied` | 权限检查拒绝 | Agent 试图删除数据库 → 策略拒绝 |
| `permission_escalated` | 权限动态升级 | 用户确认后，Agent 从 L1 临时升至 L2 |
| `permission_revoked` | 权限被主动撤销 | 管理员终止异常 Session 的权限 |
| `permission_expired` | 临时权限到期 | 30 分钟临时邮件权限过期 |
| `approval_requested` | 需要人工审批 | Agent 请求转账 → 发出审批请求 |
| `approval_granted` | 审批通过 | 用户确认高风险操作 |
| `approval_denied` | 审批拒绝 | 用户拒绝高风险操作 |
| `role_assigned` | 角色绑定 | 新 Session 分配默认角色 |
| `role_removed` | 角色解绑 | Session 结束回收角色 |

### 2. Tamper-Proof Audit（防篡改审计）

审计日志的可信度取决于其不可篡改性。以下是四种主流的防篡改技术：

#### 2.1 Hash Chain（哈希链）

最基本的防篡改机制。每条日志记录包含前一条记录的哈希值，形成链式结构。

```
Log[0]: { data_0, hash_0 = H(data_0) }
Log[1]: { data_1, prev_hash = hash_0, hash_1 = H(data_1 || hash_0) }
Log[2]: { data_2, prev_hash = hash_1, hash_2 = H(data_2 || hash_1) }
Log[3]: { data_3, prev_hash = hash_2, hash_3 = H(data_3 || hash_2) }
```

**验证方法**：任意修改链中一条日志都会导致后续所有日志的 `prev_hash` 不匹配。验证者只需从第一条开始依次计算即可检测篡改。

**局限性**：
- 只能检测篡改，不能防止篡改（攻击者如果拥有写入权限仍然可以改写日志链）
- 链越长，验证成本越高
- 单点故障——链头丢失则全链不可验证

#### 2.2 Digital Signature（数字签名）

对每个日志条目或每个日志 segment 使用私钥签名，验证者使用公钥验证。

```
Log entry:
{
  "event": { ... },
  "signature": "base64(RSASSA-PSS(sk, H(event)))"
}

Log segment (batch):
{
  "segment_id": "2026-07-17-14",
  "first_event": "evt_001",
  "last_event": "evt_999",
  "root_hash": "H(H(evt_001) || H(evt_002) || ... || H(evt_999))",
  "signature": "base64(Ed25519(sk, segment_id || root_hash))",
  "timestamp": "2026-07-17T14:00:00Z"
}
```

**优势**：
- 私钥保护得当的情况下，攻击者无法伪造签名
- 可以公开验证（公钥可以公开）
- 支持 segment 级的验证，不需要暴露所有原始数据

**密钥管理挑战**：
- 私钥泄露则签名失效
- 需要使用 HSM（Hardware Security Module）或 KMS（Key Management Service）保护签名密钥
- 密钥轮转需要妥善处理签名验证链

#### 2.3 Append-Only Storage（只追加存储）

利用存储系统的只追加特性来防止篡改或删除：

| 实现方式 | 描述 | 适合场景 |
|---------|------|---------|
| WORM 存储 | 写入后不可修改（Write Once Read Many） | 合规归档 |
| 审计专用 DB | 如 Amazon Audit Manager、Azure Purview | 云原生 |
| Immutable 对象存储 | S3 Object Lock、GCS Bucket Lock | 大规模日志归档 |
| Append-only FS | 只追加文件系统或日志结构文件 | 本地部署 |

#### 2.4 Blockchain-Verifiable Logs（区块链可验证日志）

将日志摘要定期发布到区块链或分布式账本，利用区块链的不可篡改特性。

```
每小时：
  1. 计算该小时所有审计事件的 Merkle Root
  2. 将 Merkle Root 发布到区块链（如 Ethereum、Hyperledger）
  3. 保留区块链交易 ID 作为验证证明

验证时：
  1. 提取事件的 Merkle Proof
  2. 使用区块链上记录的 Merkle Root 验证
```

**优势**：
- 最高级别的不可篡改保证
- 去中心化验证，不依赖单一信任锚点
- 适合需要第三方审计的场景

**成本**：
- 区块链写入需要 gas 费用（公链）
- 延迟较高（区块确认时间）
- 不适合高频写入

**分层方案（推荐）**：

```
高频事件 → 本地哈希链（秒级）
中频聚合 → 数字签名 segment（小时级）
低频锚定 → 区块链发布（天级）
```

### 3. Audit Log Storage（审计日志存储）

#### 3.1 Hot / Warm / Cold Tiering（分层存储）

| 层级 | 存储介质 | 保留时间 | 查询性能 | 成本 | 用途 |
|------|---------|---------|---------|------|------|
| Hot | 内存 / Redis / SSD | 24 小时 | <1ms | 高 | 实时告警、实时监控 |
| Warm | PostgreSQL / Elasticsearch | 30-90 天 | <100ms | 中 | 日常查询、报表分析 |
| Cold | S3 / GCS / Azure Blob | 1-7 年 | 秒-分级 | 低 | 合规归档、取证回溯 |

**数据流动**：

```
Agent Runtime ──→ Hot (Redis Stream) ──→ Warm (PostgreSQL) ──→ Cold (S3)
                      │                       │
                      ▼                       ▼
                 实时分析引擎              报表/搜索
```

#### 3.2 Retention Policies（保留策略）

```yaml
retention_policies:
  hot:
    duration: 24h
    max_size: 100GB
    action: delete_after_consumed

  warm:
    duration: 90d
    max_size: 10TB
    action:
      type: compress_and_move
      target: cold
    aggregation:
      enabled: true
      window: 1h
      metrics: [event_count, deny_rate, unique_users]

  cold:
    duration: 7y
    max_size: unlimited
    action:
      type: archive_with_hashchain
      target: glacier
    encryption:
      enabled: true
      algorithm: AES-256-GCM
```

**保留策略设计要点**：
- 法规保留期是最低要求，不是推荐值——建议在此基础上增加缓冲期
- 不同事件类型可以有不同的保留期（如拒绝事件保留更长）
- 冷存储建议使用压缩（审计日志文本压缩比可达 10:1）
- 删除过期日志前需确认合规要求已满足

#### 3.3 Log Rotation（日志轮转）

```python
# 日志轮转策略示例
rotation_strategies = {
    "size_based": {
        "trigger": "max_file_size = 500MB",
        "action": "close_current → archive → create_new",
        "naming": "audit_{YYYY-MM-DD}_{SEQ}.log.gz"
    },
    "time_based": {
        "trigger": "interval = 1 hour",
        "action": "finalize_segment → sign → archive",
        "naming": "segment_{YYYY-MM-DD-HH}.signed.log"
    },
    "event_based": {
        "trigger": "event_count = 100000",
        "action": "batch_write → clear_buffer",
        "naming": "batch_{SEQ}.avro"
    }
}
```

### 4. Permission Usage Analytics（权限使用分析）

审计日志是权限优化的金矿。通过分析可以回答以下问题：

#### 4.1 Access Patterns（访问模式分析）

```sql
-- 最常使用的工具 Top 10
SELECT
    resource.name,
    COUNT(*) as access_count,
    COUNT(DISTINCT actor.id) as unique_users,
    AVG(decision.risk_score) as avg_risk
FROM audit_events
WHERE timestamp > NOW() - INTERVAL '30 days'
    AND decision.result = 'allow'
GROUP BY resource.name
ORDER BY access_count DESC
LIMIT 10;

-- 权限拒绝最多的场景
SELECT
    resource.type,
    resource.action,
    decision.reason,
    COUNT(*) as deny_count
FROM audit_events
WHERE timestamp > NOW() - INTERVAL '7 days'
    AND decision.result = 'deny'
GROUP BY resource.type, resource.action, decision.reason
ORDER BY deny_count DESC;
```

**关键分析维度**：
- **时间维度**：按小时/天/周聚合，识别高峰期和异常时段
- **用户维度**：按用户/Session 聚合，识别特权用户和异常行为
- **资源维度**：按工具/API 聚合，识别高热度资源和被滥用资源
- **决策维度**：允许 vs 拒绝分布，识别过于严格或宽松的策略

#### 4.2 Unusual Behavior Detection（异常行为检测）

基于审计日志的统计基线，检测偏离正常模式的行为：

```python
# 异常检测规则示例
anomaly_rules = [
    {
        "name": "sudden_deny_spike",
        "description": "拒绝率在短时间内急剧上升",
        "metric": "deny_rate",
        "window": "5m",
        "threshold": "baseline_avg + 3 * baseline_std",
        "severity": "high"
    },
    {
        "name": "unusual_access_time",
        "description": "在非工作时间访问高权限工具",
        "condition": "hour NOT BETWEEN 9 AND 18 AND resource.level >= L3",
        "severity": "medium"
    },
    {
        "name": "rare_tool_access",
        "description": "访问了历史上很少被使用的工具",
        "condition": "resource.name NOT IN top_90_percent_accessed",
        "severity": "low"
    },
    {
        "name": "rapid_permission_change",
        "description": "短时间内权限快速变化",
        "metric": "permission_change_count",
        "window": "1m",
        "threshold": "> 5",
        "severity": "critical"
    }
]
```

#### 4.3 Privilege Creep Detection（权限蠕变检测）

权限蠕变是指用户或 Agent 随着时间推移积累了大量非必要的权限。审计日志可以检测这类问题：

```sql
-- 检测权限蠕变：用户权限随时间增长的曲线
SELECT
    actor.id,
    DATE_TRUNC('week', timestamp) as week,
    COUNT(DISTINCT resource.name || '.' || resource.action) as unique_permissions
FROM audit_events
WHERE decision.result = 'allow'
GROUP BY actor.id, week
ORDER BY actor.id, week;

-- 检测未被使用的已授权工具
SELECT DISTINCT
    actor.id,
    p.tool_name
FROM assigned_permissions p
LEFT JOIN audit_events e
    ON e.actor.id = p.user_id
    AND e.resource.name = p.tool_name
    AND e.timestamp > NOW() - INTERVAL '90 days'
WHERE e.event_id IS NULL;
```

### 5. Real-Time Permission Monitoring（实时权限监控）

#### 5.1 Alert on Unexpected Grants（异常授予告警）

```python
# 实时告警规则 —— 基于 CEL (Common Expression Language) 风格的表达式

alert_rules = {
    "unexpected_high_risk_grant": {
        "condition": (
            "event.event_type == 'permission_granted' "
            "AND event.resource.level >= 'L3' "
            "AND event.actor.role != 'admin'"
        ),
        "action": "send_slack_alert AND create_ticket",
        "cooldown": "5m"
    },
    "first_time_deny_escalation": {
        "condition": (
            "event.event_type == 'permission_denied' "
            "AND event.actor.is_new_session == true "
            "AND event.actor.previous_denies == 0"
        ),
        "action": "log_and_tag_session",
        "cooldown": "1h"
    }
}
```

#### 5.2 Alert on Denied Attempts（拒绝尝试告警）

连续拒绝往往是攻击的预兆：

| 模式 | 阈值 | 严重程度 | 响应 |
|------|------|---------|------|
| 单用户连续拒绝 | > 5 次 / 分钟 | Medium | 标记 Session、增加监控 |
| 批量拒绝 | > 20 次 / 分钟 (全局) | High | 限速、触发详细日志 |
| 同一资源被多人拒绝 | > 3 个不同用户 / 分钟 | High | 检查资源配置 |
| 拒绝后成功 | 拒绝→允许切换 | Medium | 记录上下文供分析 |

#### 5.3 Alert on Escalation Patterns（权限升级模式告警）

可疑的权限升级模式：

```
⚠ 快速升级链：L0 → L2 → L3 → L4 在 5 分钟内
   → 可能是攻击者快速测试权限边界

⚠ 非常规升级路径：直接从 L0 跳到 L3 跳过中间等级
   → 可能是权限配置错误或绕过机制

⚠ 降级后立即升级：降级后 1 分钟内再次申请升级
   → 可能是攻击者尝试不同路径

⚠ 跨 Session 升级：同一用户在多个 Session 中反复升级
   → 可能是权限模型本身有问题，用户需要但权限不足
```

### 6. Cross-Session Permission Tracking（跨 Session 权限追踪）

#### 6.1 User-Level Permission Usage History（用户级权限使用历史）

```sql
-- 用户权限使用时间线
SELECT
    timestamp,
    session_id,
    event_type,
    resource.type || ':' || resource.name as resource,
    decision.result
FROM audit_events
WHERE actor.id = 'user_12345'
    AND timestamp > NOW() - INTERVAL '30 days'
ORDER BY timestamp DESC
LIMIT 100;
```

**追踪维度**：
- **跨 Session 频率**：用户在多个 Session 中使用同一权限的频率
- **权限使用成熟度**：用户对特定工具的使用熟练度（次数、成功率）
- **权限请求序列**：用户在 Session 内请求权限的顺序模式

#### 6.2 Anomaly Detection Across Sessions（跨 Session 异常检测）

将单个 Session 的行为与用户的历史基线比较：

```python
class CrossSessionAnomalyDetector:
    """跨 Session 异常检测器"""

    def __init__(self):
        self.user_baselines = {}  # user_id → PermissionBaseline

    async def build_baseline(self, user_id: str, history: List[AuditEvent]):
        """基于历史审计事件构建用户行为基线"""
        events = [e for e in history if e.actor.id == user_id]

        baseline = PermissionBaseline(
            user_id=user_id,
            total_sessions=len(set(e.session_id for e in events)),
            common_tools=self._extract_top_tools(events, top_k=20),
            access_hours=self._extract_access_hours(events),
            typical_deny_rate=self._calculate_deny_rate(events),
            permission_transition_matrix=self._build_transition_matrix(events)
        )
        self.user_baselines[user_id] = baseline
        return baseline

    async def check_session_anomaly(self, session_events: List[AuditEvent]) -> AnomalyReport:
        """检查当前 Session 是否偏离用户基线"""
        user_id = session_events[0].actor.id
        baseline = self.user_baselines.get(user_id)
        if not baseline:
            return AnomalyReport(status="insufficient_data")

        anomalies = []

        # 1. 检测新工具访问
        current_tools = set(e.resource.name for e in session_events)
        new_tools = current_tools - baseline.common_tools
        if new_tools:
            anomalies.append(Anomaly(
                type="new_tool_access",
                severity="medium",
                details=f"首次访问工具: {new_tools}"
            ))

        # 2. 检测非正常时段访问
        current_hours = set(e.timestamp.hour for e in session_events)
        unusual_hours = current_hours - baseline.access_hours
        if unusual_hours:
            anomalies.append(Anomaly(
                type="unusual_access_hour",
                severity="low",
                details=f"非习惯时段: {unusual_hours}"
            ))

        # 3. 检测拒绝率异常
        current_deny_rate = self._calculate_deny_rate(session_events)
        if current_deny_rate > baseline.typical_deny_rate * 3:
            anomalies.append(Anomaly(
                type="elevated_deny_rate",
                severity="high",
                details=f"拒绝率 {current_deny_rate:.1%} (基线: {baseline.typical_deny_rate:.1%})"
            ))

        # 4. 检测权限转换异常
        current_transitions = self._build_transition_matrix(session_events)
        unusual_transitions = self._find_unusual_transitions(
            current_transitions, baseline.permission_transition_matrix
        )
        if unusual_transitions:
            anomalies.append(Anomaly(
                type="unusual_permission_flow",
                severity="critical",
                details=f"异常权限转换: {unusual_transitions}"
            ))

        return AnomalyReport(
            user_id=user_id,
            session_id=session_events[0].session_id,
            anomalies=anomalies,
            risk_score=self._calculate_risk_score(anomalies)
        )
```

## Example Code

### PermissionAuditor：完整的 Python 实现

以下实现包含防篡改审计日志、实时告警和查询分析功能。

```python
"""
permission_auditor.py — 权限审计系统

功能:
  1. 防篡改审计日志记录（哈希链 + 数字签名）
  2. 实时告警
  3. 审计查询与分析
  4. 日志轮转与生命周期管理
"""

import hashlib
import hmac
import json
import time
import uuid
from datetime import datetime, timezone
from typing import Optional, List, Dict, Any
from dataclasses import dataclass, field, asdict
from collections import defaultdict
import logging

logger = logging.getLogger(__name__)


# ─── 数据模型 ─────────────────────────────────────────────────

@dataclass
class AuditEvent:
    """审计事件"""
    event_id: str
    timestamp: str
    event_type: str
    actor_id: str
    actor_type: str
    session_id: str
    resource_type: str
    resource_id: str
    resource_action: str
    decision_result: str
    decision_reason: str
    policy_id: Optional[str] = None
    risk_score: Optional[float] = None

    def serialize(self) -> str:
        """序列化为 JSON 字符串"""
        return json.dumps(asdict(self), sort_keys=True, ensure_ascii=False)


@dataclass
class SignedLogEntry:
    """带哈希链和签名的日志条目"""
    event: AuditEvent
    prev_hash: str
    hash: str
    signature: Optional[str] = None
    sequence: int = 0

    def verify_chain(self, prev_entry: "SignedLogEntry") -> bool:
        """验证前一条记录的哈希链"""
        expected_prev_hash = self._compute_hash(prev_entry)
        return self.prev_hash == expected_prev_hash

    @staticmethod
    def _compute_hash(entry: "SignedLogEntry") -> str:
        """计算日志条目的哈希值"""
        data = entry.event.serialize() + entry.prev_hash + str(entry.sequence)
        return hashlib.sha256(data.encode()).hexdigest()


# ─── 权限审计器 ───────────────────────────────────────────────

class PermissionAuditor:
    """
    权限审计器
    
    负责：
    - 记录权限决策事件
    - 维护防篡改的哈希链
    - 提供实时告警
    - 支持审计查询
    """

    def __init__(
        self,
        signing_key: Optional[bytes] = None,
        enable_chain: bool = True,
        batch_size: int = 100,
        flush_interval: float = 5.0,
    ):
        self.signing_key = signing_key
        self.enable_chain = enable_chain
        self.batch_size = batch_size
        self.flush_interval = flush_interval

        # 哈希链状态
        self._chain: List[SignedLogEntry] = []
        self._last_hash: str = "0" * 64  # 初始哈希（全零）

        # 批量写入缓冲区
        self._buffer: List[AuditEvent] = []
        self._last_flush = time.monotonic()

        # 告警规则
        self._alert_rules: List[Dict] = []

        # 拒绝计数器（用于实时告警）
        self._deny_counter: Dict[str, int] = defaultdict(int)
        self._deny_window_start: float = time.monotonic()

    def record(
        self,
        event_type: str,
        actor_id: str,
        actor_type: str,
        session_id: str,
        resource_type: str,
        resource_id: str,
        resource_action: str,
        decision_result: str,
        decision_reason: str,
        policy_id: Optional[str] = None,
        risk_score: Optional[float] = None,
    ) -> str:
        """
        记录一条审计事件
        
        Returns:
            event_id: 事件 ID
        """
        event = AuditEvent(
            event_id=str(uuid.uuid4()),
            timestamp=datetime.now(timezone.utc).isoformat(),
            event_type=event_type,
            actor_id=actor_id,
            actor_type=actor_type,
            session_id=session_id,
            resource_type=resource_type,
            resource_id=resource_id,
            resource_action=resource_action,
            decision_result=decision_result,
            decision_reason=decision_reason,
            policy_id=policy_id,
            risk_score=risk_score,
        )

        # 实时告警检查
        self._check_alerts(event)

        # 拒绝率监控
        if decision_result == "deny":
            self._track_deny(event)

        # 写入缓冲区
        self._buffer.append(event)

        # 如果缓冲区满或达到间隔时间，批量刷入
        if (
            len(self._buffer) >= self.batch_size
            or (time.monotonic() - self._last_flush) >= self.flush_interval
        ):
            self.flush()

        return event.event_id

    def flush(self):
        """将缓冲区中的事件刷入持久化存储"""
        if not self._buffer:
            return

        for event in self._buffer:
            signed_entry = self._sign_event(event)
            self._chain.append(signed_entry)
            self._persist(signed_entry)

        self._buffer.clear()
        self._last_flush = time.monotonic()
        logger.debug(f"Flushed {len(self._buffer)} audit events to storage")

    def _sign_event(self, event: AuditEvent) -> SignedLogEntry:
        """创建带签名和哈希链的日志条目"""
        sequence = len(self._chain)

        entry = SignedLogEntry(
            event=event,
            prev_hash=self._last_hash,
            hash="",  # 暂留
            sequence=sequence,
        )

        # 计算当前条目的哈希
        entry.hash = SignedLogEntry._compute_hash(entry)

        # 可选：数字签名
        if self.signing_key and self.enable_chain:
            data_to_sign = (entry.hash + str(sequence)).encode()
            entry.signature = hmac.new(
                self.signing_key, data_to_sign, hashlib.sha256
            ).hexdigest()

        # 更新链状态
        self._last_hash = entry.hash

        return entry

    def _persist(self, entry: SignedLogEntry):
        """持久化存储日志条目（示例：写入文件）"""
        log_line = json.dumps({
            "event": asdict(entry.event),
            "prev_hash": entry.prev_hash,
            "hash": entry.hash,
            "signature": entry.signature,
            "sequence": entry.sequence,
        }, ensure_ascii=False)
        # 实际实现中写入数据库或日志文件
        # self._storage.append(log_line)

    # ── 哈希链验证 ──────────────────────────────────────────

    def verify_chain_integrity(self) -> VerificationReport:
        """
        验证整个哈希链的完整性
        
        Returns:
            VerificationReport: 验证报告
        """
        if not self._chain:
            return VerificationReport(valid=True, checked=0, errors=[])

        errors = []
        for i in range(1, len(self._chain)):
            if not self._chain[i].verify_chain(self._chain[i - 1]):
                errors.append(f"Chain broken at entry {i}: hash mismatch")
                break

        # 验证第一条记录的前哈希是否为初始值
        if self._chain and self._chain[0].prev_hash != "0" * 64:
            errors.append("First entry prev_hash is not the initial hash")

        # 验证签名（如果启用）
        if self.signing_key:
            for entry in self._chain:
                if entry.signature:
                    expected_sig = hmac.new(
                        self.signing_key,
                        (entry.hash + str(entry.sequence)).encode(),
                        hashlib.sha256,
                    ).hexdigest()
                    if entry.signature != expected_sig:
                        errors.append(f"Signature mismatch at entry {entry.sequence}")

        return VerificationReport(
            valid=len(errors) == 0,
            checked=len(self._chain),
            errors=errors,
        )

    # ── 实时告警 ────────────────────────────────────────────

    def add_alert_rule(self, rule: Dict):
        """添加告警规则"""
        self._alert_rules.append(rule)

    def _check_alerts(self, event: AuditEvent):
        """检查事件是否触发告警规则"""
        for rule in self._alert_rules:
            if self._match_rule(event, rule):
                self._fire_alert(rule, event)

    def _match_rule(self, event: AuditEvent, rule: Dict) -> bool:
        """检查事件是否匹配规则（简化实现）"""
        conditions = rule.get("conditions", {})
        for field, expected in conditions.items():
            actual = getattr(event, field, None)
            if actual != expected:
                return False
        return True

    def _fire_alert(self, rule: Dict, event: AuditEvent):
        """触发告警"""
        alert = {
            "rule": rule["name"],
            "severity": rule.get("severity", "info"),
            "event_id": event.event_id,
            "timestamp": event.timestamp,
            "message": rule.get("message", "").format(
                actor=event.actor_id,
                resource=f"{event.resource_type}:{event.resource_id}",
                action=event.resource_action,
            ),
        }
        logger.warning(f"AUDIT ALERT: {alert['severity']} — {alert['message']}")
        # 实际实现中，推送告警到消息队列
        # self._alert_channel.publish(alert)

    def _track_deny(self, event: AuditEvent):
        """跟踪拒绝事件以检测拒绝风暴"""
        now = time.monotonic()
        # 每秒重置计数器
        if now - self._deny_window_start > 1.0:
            self._deny_counter.clear()
            self._deny_window_start = now

        self._deny_counter[event.actor_id] += 1

        if self._deny_counter[event.actor_id] > 5:
            logger.critical(
                f"Deny storm detected: user {event.actor_id} "
                f"had {self._deny_counter[event.actor_id]} denies in 1 second"
            )

    # ── 审计查询 ────────────────────────────────────────────

    def query(
        self,
        actor_id: Optional[str] = None,
        session_id: Optional[str] = None,
        event_type: Optional[str] = None,
        decision_result: Optional[str] = None,
        start_time: Optional[str] = None,
        end_time: Optional[str] = None,
        limit: int = 100,
    ) -> List[AuditEvent]:
        """
        审计查询
        
        支持按 actor / session / event_type / decision / 时间范围过滤
        """
        results = []
        for entry in self._chain:
            event = entry.event
            if actor_id and event.actor_id != actor_id:
                continue
            if session_id and event.session_id != session_id:
                continue
            if event_type and event.event_type != event_type:
                continue
            if decision_result and event.decision_result != decision_result:
                continue
            if start_time and event.timestamp < start_time:
                continue
            if end_time and event.timestamp > end_time:
                continue
            results.append(event)
            if len(results) >= limit:
                break
        return results

    # ── 分析功能 ────────────────────────────────────────────

    def permission_dashboard(self, hours: int = 24) -> Dict:
        """
        权限使用仪表盘数据
        
        Returns:
            dict: 包含各项指标的仪表盘数据
        """
        cutoff = datetime.now(timezone.utc).isoformat()
        # 简化: 实际应传时间参数
        recent_events = [
            e for e in self._chain
            if e.event.timestamp >= cutoff  # simplified
        ]

        total = len(recent_events)
        if total == 0:
            return {"status": "no_data"}

        denies = [e for e in recent_events if e.event.decision_result == "deny"]
        allows = [e for e in recent_events if e.event.decision_result == "allow"]
        escalations = [e for e in recent_events if e.event.event_type == "permission_escalated"]

        # 按资源类型聚合
        by_resource = defaultdict(int)
        for e in recent_events:
            key = f"{e.event.resource_type}/{e.event.resource_action}"
            by_resource[key] += 1

        # 按用户聚合
        by_user = defaultdict(lambda: {"allow": 0, "deny": 0})
        for e in recent_events:
            uid = e.event.actor_id
            by_user[uid][e.event.decision_result] += 1

        # 找出常被拒绝的用户
        top_denied_users = sorted(
            by_user.items(),
            key=lambda x: x[1]["deny"],
            reverse=True,
        )[:5]

        return {
            "period_hours": hours,
            "total_events": total,
            "allow_rate": len(allows) / total if total > 0 else 0,
            "deny_rate": len(denies) / total if total > 0 else 0,
            "escalation_rate": len(escalations) / total if total > 0 else 0,
            "top_resources": dict(sorted(by_resource.items(), key=lambda x: x[1], reverse=True)[:10]),
            "top_denied_users": [
                {"user": uid, "denies": counts["deny"]}
                for uid, counts in top_denied_users
            ],
            "chain_integrity": self.verify_chain_integrity().valid,
            "chain_length": len(self._chain),
        }


@dataclass
class VerificationReport:
    """哈希链验证报告"""
    valid: bool
    checked: int
    errors: List[str]


# ── 使用示例 ─────────────────────────────────────────────────

if __name__ == "__main__":
    import os

    # 初始化审计器（使用环境变量中的签名密钥）
    signing_key = os.environ.get("AUDIT_SIGNING_KEY", "dev-key").encode()
    auditor = PermissionAuditor(
        signing_key=signing_key,
        enable_chain=True,
        batch_size=50,
        flush_interval=2.0,
    )

    # 添加告警规则
    auditor.add_alert_rule({
        "name": "high_risk_access",
        "severity": "high",
        "conditions": {
            "resource_type": "tool",
            "resource_action": "delete",
        },
        "message": "{actor} attempted to delete {resource}",
    })

    # 记录权限事件
    event_id = auditor.record(
        event_type="permission_granted",
        actor_id="agent_001",
        actor_type="agent",
        session_id="session_abc",
        resource_type="tool",
        resource_id="search_engine",
        resource_action="search",
        decision_result="allow",
        decision_reason="auto_grant: L0 tool",
        policy_id="policy_default",
        risk_score=0.1,
    )
    print(f"Recorded event: {event_id}")

    # 记录拒绝事件
    auditor.record(
        event_type="permission_denied",
        actor_id="agent_001",
        actor_type="agent",
        session_id="session_abc",
        resource_type="tool",
        resource_id="email_sender",
        resource_action="send",
        decision_result="deny",
        decision_reason="policy: L1 requires approval",
        policy_id="policy_l1_tools",
        risk_score=0.6,
    )

    # 刷入缓冲区
    auditor.flush()

    # 验证哈希链完整性
    report = auditor.verify_chain_integrity()
    print(f"Chain valid: {report.valid}, entries checked: {report.checked}")

    # 查询审计记录
    results = auditor.query(actor_id="agent_001", limit=10)
    print(f"Query returned {len(results)} results")

    # 仪表盘数据
    dashboard = auditor.permission_dashboard(hours=24)
    print(f"Dashboard: {json.dumps(dashboard, indent=2, ensure_ascii=False)}")
```

## Capability Boundaries（能力边界）

### 1. 日志本身必须被保护

审计日志记录了所有权限决策的详细信息，使其成为攻击者的高价值目标。如果攻击者获得了审计日志的写入权限，他们可以：

- 掩盖自己的攻击痕迹（删除或篡改日志）
- 读取其他用户的权限行为模式（信息泄露）
- 通过分析拒绝记录，发现系统的安全漏洞

**防护措施**：

```
┌─────────────────────────────────────────┐
│            审计日志安全层                  │
├─────────────────────────────────────────┤
│ 写入保护                                  │
│   • 仅审计服务自身有写入权限               │
│   • 使用独立服务账号，与 Agent 系统隔离     │
│   • 写入端点和凭据不与其他服务共享          │
├─────────────────────────────────────────┤
│ 读取保护                                  │
│   • 审计日志读取需要独立权限               │
│   • 默认不可读，仅合规/安全团队可访问      │
│   • 读取行为本身也要被审计                 │
├─────────────────────────────────────────┤
│ 删除保护                                  │
│   • 使用 WORM 或 Append-Only 存储         │
│   • 删除操作需要多人审批                   │
│   • 保留删除操作日志（谁在何时删除了什么）  │
├─────────────────────────────────────────┤
│ 传输保护                                  │
│   • TLS 加密传输                          │
│   • 日志在写入前进行签名                   │
│   • 端到端完整性验证                       │
└─────────────────────────────────────────┘
```

### 2. 存储成本

审计日志的存储成本可能远超预期：

**成本估算**：

```
假设：
  • 每次权限决策产生约 500 字节的审计事件
  • 系统每秒处理 1,000 次权限决策
  • = 每天约 86,400,000 次决策
  • = 每天约 43 GB 原始数据
  • = 通过压缩（10:1）后约 4.3 GB
  • = 每年约 1.5 TB 压缩数据

分层存储成本（以 AWS 为例，2026 年）：
  • Hot (SSD):      43 GB × 30 天 = 1.3 TB → ~$300/月
  • Warm (EBS):      4.3 GB × 90 天 = 0.4 TB → ~$40/月
  • Cold (S3 Glacier): 4.3 GB × 残留天数 → ~$10/月
  • 总存储成本: ~$350/月（初始），随时间累积增长
```

**优化策略**：
- 常规允许事件采样（记录 10%），关键事件（拒绝、升级）全量保留
- 事件保留原始数据 7 天后聚合为统计摘要（保留原始数据的高频维度）
- 使用列式存储格式（Parquet / ORC）压缩存储
- 设置存储上限和告警，避免成本失控

### 3. 性能开销

审计不应该成为系统的性能瓶颈：

| 场景 | 同步写入开销 | 异步写入开销 |
|------|------------|------------|
| 每次决策都要写 | +1-5ms 延迟 | +0.1-0.5ms 延迟 |
| 批量写（100条/批） | N/A | 整体延迟可忽略 |
| 哈希链计算 | +0.01-0.1ms | 无影响 |
| 签名计算 | +0.1-1ms（HMAC）/+1-10ms（RSA） | 无影响 |

**结论**：使用异步批量写入 + 缓冲区设计，审计系统对权限决策的性能影响可以控制在 1% 以内。

### 4. 审计日志的隐私风险

审计日志自身可能成为隐私泄露的通道：

- **过度记录**：如果审计日志包含了用户的 Prompt 输入或 Agent 的完整输出，这些内容可能包含个人信息
- **长期留存放大风险**：日志保留时间越长，被泄露时的危害越大
- **跨系统关联**：审计日志如果需要与其它系统关联，可能可以重建用户行为的完整画像

**缓解措施**：
- 对审计日志中的敏感字段进行脱敏（Masking）或令牌化（Tokenization）
- 设置不同的保留期：元数据保留长期，payload 数据短期
- 审计日志的访问需要独立审批，与常规日志访问隔离

### 5. 分布式系统下的挑战

在分布式 Agent 系统中，审计日志面临额外挑战：

- **时钟不同步**：不同节点的审计日志时间戳可能不一致，难以重建全局事件顺序
- **原子性**：一个跨多个服务的权限决策，其审计事件需要在所有节点上原子记录
- **一致性**：主节点故障时，审计事件可能丢失

**应对方案**：
- 使用逻辑时钟（Lamport Clock / Hybrid Logical Clock）而非物理时钟
- 使用分布式事务 ID（Trace ID / Span ID）关联跨节点的审计事件
- 审计事件先写入本地队列，再通过可靠的日志聚合服务（如 Kafka）汇总

## Comparison：审计方案对比

### 结构化 vs 非结构化审计

| 维度 | 结构化审计 | 非结构化审计 |
|------|-----------|-------------|
| Schema | 严格定义（JSON Schema / Avro / Protobuf） | 自由文本（Syslog / 自由 JSON） |
| 查询能力 | 强——可按任意字段过滤、聚合 | 弱——依赖全文搜索 |
| 存储效率 | 高——列式压缩效果好 | 低——文本冗余 |
| 扩展成本 | 高——修改 Schema 需要迁移 | 低——新增字段直接追加 |
| 兼容性 | 强——工具链丰富（Spark、Presto） | 弱——需自定义解析 |
| 推荐场景 | 生产系统、合规需求 | 开发/调试环境、无严格合规要求 |

**建议**：生产系统使用结构化审计（Avro / Protobuf），开发调试使用非结构化审计。

### Schema 设计选择

| Schema 格式 | 优势 | 劣势 | 适用场景 |
|------------|------|------|---------|
| JSON | 人类可读、灵活 | 体积大、解析慢 | 开发环境、小型系统 |
| Avro | 压缩高、Schema 可演化 | 需 Schema Registry | 大数据生态（Kafka + Hadoop） |
| Protobuf | 体积最小、速度最快 | Schema 管理成本高 | 高性能场景、微服务架构 |
| Parquet | 列式压缩、分析友好 | 写入延迟高 | 批量分析、OLAP |
| CSV | 最通用、工具最多 | 类型信息缺失 | 简单导入/导出 |

**推荐组合**：

```
热存储（实时）:  JSON 或 Protobuf
温存储（查询）:  Parquet（从 Avro 转换）
冷存储（归档）:  Avro + Snappy 压缩
```

### 审计方案全面对比

| 方案 | 防篡改 | 查询能力 | 实时性 | 成本 | 运维复杂度 | 适用规模 |
|------|--------|---------|--------|------|-----------|---------|
| 本地文件 + 哈希链 | 中 | 弱 | 低 | 低 | 低 | 小 |
| ELK Stack | 弱 | 强 | 高 | 中 | 中 | 中 |
| AWS CloudTrail + S3 | 强 | 中 | 中 | 中 | 低 | 大 |
| 专用审计 DB（Immudb） | 强 | 强 | 高 | 高 | 高 | 中 |
| 区块链锚定 | 最强 | 弱 | 低 | 高 | 高 | 高合规场景 |
| 自研（哈希链 + 签名） | 强 | 中 | 高 | 中 | 中 | 大 |

## Engineering Optimization（工程优化）

### 1. Log Sampling for High-Volume Events（高频事件日志采样）

不是所有审计事件都需要被完全记录。对于高频率、低风险的常规允许事件，可以采用采样策略：

```python
class AdaptiveSampler:
    """
    自适应采样器
    
    策略：
    - 高风险事件（拒绝、升级、高危操作）：100% 采样
    - 中风险事件（常规允许、角色分配）：按比例采样
    - 低风险事件（健康检查、内部服务通信）：自适应采样
    """

    def __init__(self, base_rate: float = 0.1, min_events: int = 1000):
        self.base_rate = base_rate
        self.min_events = min_events
        self.event_counters = defaultdict(int)

    def should_sample(self, event: AuditEvent) -> bool:
        """判断是否应该记录此事件"""
        # 高风险事件 —— 全量记录
        if event.decision_result in ("deny", "escalate"):
            return True
        if event.resource_type == "system" and event.resource_action in ("delete", "modify"):
            return True
        if event.risk_score and event.risk_score > 0.7:
            return True

        # 中风险事件 —— 按比例采样
        if event.risk_score and event.risk_score > 0.3:
            return self._sample_with_rate(event, self.base_rate * 2)

        # 低风险事件 —— 自适应采样
        return self._adaptive_sample(event)

    def _sample_with_rate(self, event: AuditEvent, rate: float) -> bool:
        """按固定比例采样"""
        key = self._event_key(event)
        # 基于哈希的确定性采样（保证同一用户/工具系列采样或全不采）
        hash_val = int(hashlib.md5(key.encode()).hexdigest(), 16)
        return (hash_val % 10000) < (rate * 10000)

    def _adaptive_sample(self, event: AuditEvent) -> bool:
        """自适应采样：根据事件频率动态调整采样率"""
        key = self._event_key(event)
        self.event_counters[key] += 1
        count = self.event_counters[key]

        # 前 min_events 条全量记录，之后按递减比例采样
        if count <= self.min_events:
            return True
        rate = 1.0 / (1 + (count - self.min_events) / self.min_events)
        return self._sample_with_rate(event, rate)

    def _event_key(self, event: AuditEvent) -> str:
        """事件分类键"""
        return f"{event.actor_type}:{event.resource_type}:{event.resource_action}"
```

**采样策略选择**：

| 策略 | 描述 | 适用场景 |
|------|------|---------|
| 固定比例采样 | 按固定比例（如 10%）记录 | 均匀分布的高频事件 |
| 自适应采样 | 高频降低采样率，低频提高 | 流量波动大的系统 |
| 分层采样 | 按类别分层，每层独立采样 | 类别分布不均 |
| 蓄水池采样 | 保证随机子集代表性 | 需要代表性的样本 |
| 优先级采样 | 高风险全量，低风险抽样 | 安全审计（推荐） |

### 2. Async Batch Writing（异步批量写入）

批量写入是降低审计系统性能开销的关键技术：

```python
import asyncio
from typing import List, Optional


class AsyncBatchAuditWriter:
    """
    异步批量审计日志写入器
    
    设计特点：
    - 事件先写入内存缓冲区
    - 达到批量大小或间隔时间后异步刷入
    - 支持背压（Backpressure）控制
    - 支持优雅关闭（Graceful Shutdown）
    """

    def __init__(
        self,
        storage_backend,
        batch_size: int = 500,
        max_buffer: int = 10000,
        flush_interval: float = 3.0,
        max_retries: int = 3,
    ):
        self.storage = storage_backend
        self.batch_size = batch_size
        self.max_buffer = max_buffer
        self.flush_interval = flush_interval
        self.max_retries = max_retries

        self._buffer: asyncio.Queue = asyncio.Queue(maxsize=max_buffer)
        self._batch: List[AuditEvent] = []
        self._running = False
        self._flush_event = asyncio.Event()

    async def start(self):
        """启动后台刷入协程"""
        self._running = True
        asyncio.create_task(self._flush_loop())

    async def stop(self, timeout: float = 10.0):
        """优雅关闭：等待缓冲区清空"""
        self._running = False
        self._flush_event.set()
        await asyncio.sleep(timeout)

    async def write(self, event: AuditEvent) -> bool:
        """
        写入一条审计事件（非阻塞）
        
        Returns:
            bool: 是否成功写入缓冲区（False 表示缓冲区满，事件丢失）
        """
        try:
            self._buffer.put_nowait(event)
            return True
        except asyncio.QueueFull:
            logger.error("Audit buffer full! Event dropped.")
            # 关键事件（拒绝）不能丢失 —— 这里应该触发告警
            if event.decision_result == "deny":
                logger.critical(
                    f"CRITICAL: Deny audit event dropped! "
                    f"actor={event.actor_id}, resource={event.resource_id}"
                )
            return False

    async def _flush_loop(self):
        """后台刷入循环"""
        while self._running or not self._buffer.empty():
            try:
                # 等待直到有数据或超时
                while self._buffer.empty() and self._running:
                    await asyncio.sleep(self.flush_interval * 0.1)

                # 从缓冲区收集一批事件
                self._batch.clear()
                while not self._buffer.empty() and len(self._batch) < self.batch_size:
                    try:
                        self._batch.append(self._buffer.get_nowait())
                    except asyncio.QueueEmpty:
                        break

                if self._batch:
                    await self._flush_batch_with_retry(self._batch)

            except Exception as e:
                logger.error(f"Audit flush error: {e}")
                await asyncio.sleep(1.0)

    async def _flush_batch_with_retry(self, batch: List[AuditEvent]):
        """带重试的批量写入"""
        last_error = None
        for attempt in range(self.max_retries):
            try:
                await self.storage.write_batch(batch)
                return
            except Exception as e:
                last_error = e
                await asyncio.sleep(0.5 * (2 ** attempt))  # 指数退避

        logger.error(
            f"Failed to flush audit batch after {self.max_retries} retries: {last_error}"
        )
        # 写入死信队列或本地文件作为最后兜底
        await self._write_to_dead_letter(batch)


# 使用示例
async def example():
    storage = SomeStorageBackend()
    writer = AsyncBatchAuditWriter(
        storage_backend=storage,
        batch_size=500,
        flush_interval=3.0,
    )

    await writer.start()

    # 模拟写入事件
    for i in range(1000):
        event = AuditEvent(...)  # 创建事件对象
        success = await writer.write(event)
        if not success:
            print(f"Event {i} dropped due to backpressure")

    await writer.stop()
```

### 3. 写入路径优化清单

| 优化 | 描述 | 效果 |
|------|------|------|
| 内存缓冲 | 先写入内存队列，批量刷入 | 减少 IO 次数 100x |
| 连接池复用 | 复用数据库/存储连接 | 减少连接建立开销 |
| 批量压缩 | 批量数据压缩后再写入 | 减少网络传输量 5-10x |
| 列式存储 | 使用 Parquet/ORC 格式 | 压缩比提高 2-3x |
| 预分配文件 | 预先分配固定大小的日志文件 | 减少文件系统碎片 |
| I/O 合并 | 将多个小写入合并为一个大写入 | 提升磁盘吞吐 |
| 写入分级 | 关键事件写入高性能存储，常规写入低成本存储 | 成本优化 |
| 缓存哈希 | 缓存最近事件的哈希值 | 减少重复计算 |

### 4. 系统指标监控

审计系统自身需要被监控：

```
关键监控指标：
  • audit_events_written       — 写入事件数（速率）
  • audit_events_dropped       — 丢弃事件数（不应 > 0）
  • audit_buffer_usage         — 缓冲区使用率
  • audit_flush_latency        — 刷入延迟（P50 / P99）
  • audit_flush_errors         — 刷入错误数
  • audit_storage_usage        — 存储使用量
  • audit_chain_verified       — 哈希链验证结果

告警规则：
  • 丢弃事件 > 0  → P1 告警（审计完整性受损）
  • 刷入延迟 > 5s → P2 告警（可能影响事件采集）
  • 存储使用 > 80% → P3 告警（需要扩容或轮转）
  • 哈希链验证失败 → P0 告警（安全事件！）
```

---

## 总结

权限审计轨迹是 AI Agent 安全体系的"黑匣子"——它不仅记录每一次权限决策的完整上下文，还通过实时分析和跨 Session 追踪，将审计从被动的合规工具转变为主动的安全防御能力。核心设计原则：

1. **每个决策都要记录**：没有审计的权限控制是不完整的
2. **日志必须防篡改**：哈希链 + 数字签名保证审计可信度
3. **存储必须分层**：热/温/冷分层的存储策略平衡成本和合规要求
4. **审计数据要用于分析**：从日志中提炼异常检测、权限优化的价值
5. **审计系统自身要安全**：保护审计日志不被攻击者利用或篡改
