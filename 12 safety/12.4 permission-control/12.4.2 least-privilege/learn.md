# 12.4.2 Least Privilege — 最小权限原则

## 简单介绍

最小权限原则（Principle of Least Privilege, PoLP）要求系统中的每个主体（用户、进程、Agent）只拥有完成其任务所**必需的最小权限集**。在 AI Agent 安全中，这意味着 Agent 只被授予当前任务需要的工具、参数范围和数据访问权限——不多也不少。当 Agent 被 Prompt 注入攻击或错误调用时，受限的权限就是最后的防线：攻击者可能控制了 Agent 的"大脑"，但无法超越 Agent 的"权限边界"。

传统最小权限在 Agent 时代面临新的挑战：Agent 的任务是动态的、不可预测的，无法预先知道 Agent 需要哪些权限。这催生了 **Zero Standing Privileges（ZSP）**、**Just-in-Time 权限提升**和**会话级临时授权**等新型最小权限实践。

## 基本原理

最小权限原则的核心主张简洁而有力：

> **"A subject should only have the minimum privileges needed to perform its task."**
>
> — Jerome Saltzer, 1975

```
最小权限的四个核心特征：

  1. 必要性（Necessity）    ── 每个权限必须有明确的任务理由
  2. 最小性（Minimality）   ── 权限刚好够用，没有冗余
  3. 时效性（Temporality）  ── 权限随任务结束自动回收
  4. 可审计性（Auditability）── 每次权限使用都有记录
```

在 Agent 场景中，最小权限的推理链条如下：

```
如果 Agent 不需要发邮件
  → 就不应该拥有 email.send 权限
    → 即使 Prompt 注入让 Agent "想" 发邮件
      → Agent 也没有能力执行发邮件操作
        → 攻击面被有效缩小
```

**核心思想**：安全不是建立在"Agent 不会做坏事"的假设上，而是建立在"Agent 即使想做坏事也做不到"的约束上。这是一种**悲观安全模型**——默认不信任，逐项授予信任。

## 背景

最小权限原则的思想渊源和发展历程：

```
1975 ── Saltzer & Schroeder
    │    《The Protection of Information in Computer Systems》
    │    提出 8 条安全设计原则，最小权限位列其中
    │    原文: "Every program and every privileged user of the system
    │           should operate using the least set of privileges
    │           necessary to complete the job."
    │
    ▼
1980s-1990s ── 操作系统最小权限
    │    UNIX 用户/组权限模型
    │    Windows ACL
    │    沙箱技术 (Janus, Systrace)
    │
    ▼
2000s ── 应用层最小权限
    │    RBAC (Role-Based Access Control)
    │    OAuth 2.0 细粒度 Scope
    │    AWS IAM 最小权限策略
    │    Google BeyondCorp (Zero Trust)
    │
    ▼
2010s ── 微服务最小权限
    │    Service Mesh 中的细粒度权限
    │    Kubernetes RBAC
    │    容器安全 (rootless, seccomp, AppArmor)
    │
    ▼
2023-2024 ── Agent 最小权限初期
    │    Agent 框架 (LangChain, AutoGPT, Claude Tool Use)
    │    出现工具级权限控制
    │    开始意识到参数级权限的重要性
    │
    ▼
2025-2026 ── Agent 最小权限成熟期
         Zero Standing Privileges (ZSP)
         Just-in-Time 权限提升
         Hermes Blank Slate 模式
         运行时权限监控与自动回收
         工具调用级细粒度权限分解
```

**关键分水岭**：传统最小权限是**静态的**——用户在创建时分配角色和权限，权限在会话期间保持不变。Agent 最小权限是**动态的**——权限随任务上下文变化，每次工具调用前都需要重新评估。

```
传统最小权限思维:
  "这个用户是什么角色？需要什么权限？"

Agent 最小权限思维:
  "这个 Agent 当前要调用什么工具？用什么参数？访问什么数据？
   这些参数和数据范围是否超出当前任务需要？
   是否有低权限的替代方案？"
```

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **最小权限 vs 最大能力** | Agent 的能力在于它能做很多事；最小权限限制它只能做某些事。二者天然矛盾——权限越少越安全，但能力越弱越没用。找到"刚好够用"的平衡点是最小权限的核心挑战 |
| **动态任务 vs 静态权限** | Agent 的任务在对话开始前是未知的，但权限需要在工具调用时就已经决定。"提前分配"和"动态评估"之间的矛盾需要运行时决定机制来解决 |
| **权限粒度 vs 管理复杂度** | 权限粒度越细越安全，但管理复杂度也随之指数增长。工具级权限简单但粗放；参数级权限精确但需要大量的策略配置和维护 |
| **用户体验 vs 安全提示** | 每次 Agent 需要额外权限都弹窗询问用户，用户会感到烦躁和"弹窗疲劳"，最终可能导致用户不加思考地批准所有请求——安全机制形同虚设 |
| **自动审批 vs 人工确认** | 全自动审批体验好但风险高；全人工确认安全但效率低。如何根据风险级别动态决定审批方式是核心设计抉择 |
| **权限回收 vs 任务连续性** | 多步任务中，过早回收权限会导致任务中断；过晚回收会增加安全风险。权限的生命周期需要精细管理 |

## 详细内容

### 1. Privilege Decomposition — 权限分解

权限分解是 Agent 最小权限的基础。它将粗粒度的"工具权限"拆解为三个维度的细粒度权限：

```
传统工具级权限:
  Agent 有或没有 "send_email" 权限

Agent 细粒度权限分解:
  ┌──────────────────────────────────────────┐
  │  函数级 (Function-Level)                  │
  │  ├── email.send                           │
  │  ├── email.read                           │
  │  ├── email.delete                         │
  │  └── email.list                           │
  │                                          │
  │  参数级 (Parameter-Level)                 │
  │  ├── to:     仅限特定域名                  │
  │  ├── cc:     不允许 cc                    │
  │  ├── body:   允许发送模版内容               │
  │  └── attachments: 不允许附件               │
  │                                          │
  │  数据级 (Data-Level)                      │
  │  ├── 只能访问 /data/reports/ 目录          │
  │  ├── 只能读取，不能写入                    │
  │  └── 只能读取已审批的客户记录               │
  │                                          │
  │  执行级 (Execution-Level)                 │
  │  ├── 同步执行（不能异步后台运行）            │
  │  ├── 超时限制：30秒                        │
  │  └── 速率限制：每分钟最多 5 次              │
  └──────────────────────────────────────────┘
```

**权限分解示例**：`read_database` 工具的粒度分解

```python
class DatabasePrivilege:
    """
    数据库操作的最小权限分解
    将单一的 "数据库访问权限" 拆解为多个维度的精细控制
    """

    @dataclass
    class TablePermission:
        table: str
        operations: set  # {"select", "insert", "update", "delete"}
        row_filter: Optional[str] = None  # SQL WHERE 子句限制可访问行
        column_mask: Optional[set] = None  # 可访问的列（None = 全部）
        row_limit: int = 1000  # 最多返回行数
        timeout_seconds: int = 30

    @dataclass
    class QueryPermission:
        allowed_patterns: List[str]  # 允许的 SQL 模式（如 "SELECT ... FROM orders WHERE ..."）
        forbidden_keywords: set = field(default_factory=lambda: {"DROP", "ALTER", "TRUNCATE", "DELETE"})
        max_join_tables: int = 3
        requires_where: bool = True  # SELECT 必须带 WHERE 条件

    # 实际应用示例
    customer_service_agent = {
        "tables": [
            TablePermission(
                table="orders",
                operations={"select"},
                row_filter="customer_id = {session.customer_id}",  # 只能看自己的订单
                column_mask={"id", "product", "amount", "status", "created_at"},
                row_limit=50,
            ),
            TablePermission(
                table="products",
                operations={"select"},
                column_mask={"id", "name", "price", "stock"},
                row_limit=100,
            ),
        ],
        "query": QueryPermission(
            allowed_patterns=["SELECT"],
            forbidden_keywords={"DROP", "ALTER", "TRUNCATE", "DELETE", "INSERT", "UPDATE"},
        ),
        "rate_limit": {
            "queries_per_minute": 20,
            "concurrent_queries": 2,
        }
    }
```

**权限分解的原则**：

1. **按需分解（Decompose by Need）**：不是越细越好，而是根据安全风险等级决定分解粒度。低风险工具（如天气查询）无需分解；高风险工具（如数据库访问、文件系统操作）需要精细分解。

2. **维度正交（Orthogonal Dimensions）**：函数、参数、数据、执行四个维度相互独立，一个维度的限制不应影响另一个维度的评估。

3. **默认拒绝（Default Deny）**：任何未明确授予的权限维度默认值为"拒绝"——Agent 只能使用明确授予的参数值，只能访问明确授权的数据范围。

### 2. Zero Standing Privileges (ZSP) — 零常驻权限

**ZSP 的核心思想**：Agent 在**空闲状态**下没有任何权限。权限只在需要执行特定操作时才被临时授予，操作完成后立即回收。

```
传统权限模型:
  ┌──────────────────────────────────────────────┐
  │  Agent 启动                                    │
  │     │                                          │
  │     ▼                                          │
  │  ┌──────────────┐                              │
  │  │ 分配所有权限   │ ← 一次性授予整个会话需要的权限 │
  │  └──────┬───────┘                              │
  │         ▼                                      │
  │  ┌──────────────┐                              │
  │  │ 执行所有任务   │ ← 权限在整个会话期间保持有效   │
  │  └──────┬───────┘                              │
  │         ▼                                      │
  │  ┌──────────────┐                              │
  │  │ 会话结束回收   │                              │
  │  └──────────────┘                              │
  └──────────────────────────────────────────────┘

ZSP 权限模型:
  ┌──────────────────────────────────────────────┐
  │  Agent 启动                                    │
  │     │                                          │
  │     ▼                                          │
  │  ┌──────────────┐                              │
  │  │ 零权限        │ ← Agent 没有常驻权限           │
  │  └──────┬───────┘                              │
  │         ▼                                      │
  │  ┌──────────────┐     ┌─────────────────┐      │
  │  │ 需要执行操作   │────►│ JIT 权限评估     │      │
  │  └──────┬───────┘     │  ┌───────────┐  │      │
  │         │             │  │ 安全策略检查 │  │      │
  │         │             │  └─────┬─────┘  │      │
  │         │             │        ▼        │      │
  │         │             │  ┌───────────┐  │      │
  │         │             │  │ 临时授权    │  │      │
  │         │             │  │ (TTL=30s) │  │      │
  │         │             │  └───────────┘  │      │
  │         ▼             └──────┬──────────┘      │
  │  ┌──────────────┐            │                  │
  │  │ 执行操作      │◄───────────┘                  │
  │  └──────┬───────┘                               │
  │         ▼                                       │
  │  ┌──────────────┐                               │
  │  │ 权限自动回收   │ ← TTL 到期或操作完成           │
  │  └──────────────┘                               │
  │         │                                       │
  │         ▼                                       │
  │         └──→ 回到零权限状态 ←── 循环              │
  └──────────────────────────────────────────────┘
```

**ZSP 的核心优势**：

- **攻击面最小化**：Agent 在 99% 的时间内没有任何权限，攻击窗口从"整个会话期"缩短为"单次工具调用的几十毫秒"
- **权限泄露无害**：即使 Agent 的凭证被窃取，窃取者获得的也是一个零权限凭证
- **自清洁（Self-Cleaning）**：权限自动过期，无需手动回收
- **审计完整性**：每次权限授予都是独立的审计事件，权限使用记录完整

**ZSP 的挑战**：

- **延迟开销**：每次工具调用前都需要进行权限评估和授予，增加了响应延迟（通常 5-50ms）
- **实现复杂度**：需要运行时权限引擎、策略评估器、临时凭证管理器等多个组件协同
- **缓存策略**：频繁请求同一权限时，需要智能缓存以避免重复评估

### 3. Toolset Pinning — 工具集固定

Toolset Pinning 指在 Agent 任务开始时，将 Agent 可以调用的工具集**固定**在一个最小范围内。Agent 不能动态发现或加载新工具——它只能使用启动时分配的工具。

```
Toolset Pinning 示例：

  任务：查询天气并写笔记
  ─────────────────────────────────
  全部可用工具（100+）          Agent 实际可调用工具（3）
  ┌─────────────────────┐      ┌─────────────────────┐
  │ search_web          │      │ weather.query       │ ✓
  │ weather.query       │      │ notes.create        │ ✓
  │ send_email          │      │ search_knowledge     │ ✓
  │ delete_file         │      │                     │
  │ shell_execute       │      │                     │
  │ database_query      │      │ 所有其他工具不可见    │
  │ notes.create        │      │ 或返回"无权调用"      │
  │ notes.delete        │      └─────────────────────┘
  │ calendar.manage     │
  │ payment.transfer    │
  │ search_knowledge    │
  │ ...                 │
  └─────────────────────┘
```

**实现方式**：

```python
class ToolsetPinner:
    """
    工具集固定管理器——为每个任务分配最小工具集
    """

    def __init__(self):
        # 预定义的工具集模板
        self.tool_templates = {
            "weather_query": {"weather.query", "search_knowledge"},
            "note_taking": {"notes.create", "notes.read", "search_knowledge"},
            "email_compose": {"email.send", "contacts.lookup", "templates.load"},
            "data_analysis": {"database.query", "file.read", "code.execute_python"},
            "customer_support": {
                "orders.query", "products.search", "tickets.create",
                "tickets.update", "knowledge_base.search",
            },
        }

    def pin_toolset(self, task: str, user_role: str, context: dict) -> set:
        """
        根据任务描述为用户固定工具集

        策略:
        1. 从任务描述中提取意图
        2. 匹配预定义工具集模板
        3. 根据用户角色裁剪工具集
        4. 返回最终固定的工具集
        """
        # 意图匹配
        matched_tools = self._match_intent_to_tools(task)

        # 角色约束
        role_tools = self._apply_role_restrictions(user_role)

        # 取交集——角色和任务的双重约束
        pinned = matched_tools & role_tools

        # 记录审计
        self._audit_toolset_pinning(task, user_role, pinned)

        return pinned

    def check_tool_allowed(self, tool_name: str, pinned_set: set) -> bool:
        """检查工具是否在当前固定的工具集中"""
        if tool_name not in pinned_set:
            raise PermissionError(
                f"Tool '{tool_name}' is not in the pinned toolset ({pinned_set}). "
                f"This operation has been blocked by Toolset Pinning."
            )
        return True
```

**Toolset Pinning 的粒度级别**：

| 级别 | 说明 | 示例 | 安全性 | 灵活性 |
|------|------|------|--------|--------|
| L0: 无限制 | Agent 可调用任何工具 | 全部 100+ 工具 | ❌ 最低 | ✅ 最高 |
| L1: 角色级 | 按角色固定工具集 | "客服 Agent 只能调用客服相关工具" | ★☆☆☆☆ | ★★★★★ |
| L2: 任务级 | 按任务固定工具集 | "查天气任务只能调用天气工具" | ★★★☆☆ | ★★★★☆ |
| L3: 意图级 | 按实时意图固定 | "Agent 要查天气 → 临时允许 weather.query" | ★★★★★ | ★★★☆☆ |
| L4: 步骤级 | 按任务步骤固定 | "Step 1 只能查天气，Step 2 才能写笔记" | ★★★★★★ | ★★☆☆☆ |

### 4. Data Scope Limiting — 数据范围限制

Data Scope Limiting 确保 Agent 只能访问完成任务所需的**最小数据子集**，包括：只读 vs 读写、数据子集、行级/字段级权限。

```
数据范围限制的层级：

  ┌──────────────────────────────────────────────────┐
  │  数据库 (Database Level)                         │
  │  ├── ╳ 禁止访问 customer_pii 表                   │
  │  ├── ✓ 允许访问 products 表                       │
  │  └── ✓ 允许访问 orders 表                         │
  │                                                  │
  ├── 行级 (Row Level)                                │
  │  ├── ✓ 只能查看当前客户的订单                      │
  │  └── ╳ 不能查看其他客户的订单                      │
  │                                                  │
  ├── 字段级 (Field Level)                            │
  │  ├── ✓ 可查看：product, amount, status            │
  │  ├── ╳ 不可查看：credit_card, cvv, ssn            │
  │  └── ╳ 不可查看：internal_notes                   │
  │                                                  │
  └── 操作级 (Operation Level)                        │
     ├── ✓ SELECT（只读查询）                         │
     ├── ╳ INSERT（禁止写入）                         │
     ├── ╳ UPDATE（禁止修改）                         │
     └── ╳ DELETE（禁止删除）                         │
```

**数据范围限制的 Python 实现**：

```python
@dataclass
class DataScope:
    """数据范围定义"""
    allowed_tables: set
    row_filter: Optional[str]  # SQL WHERE 或 Python 过滤表达式
    allowed_fields: dict  # {table: [field1, field2, ...]}
    read_only: bool = True
    max_records: int = 100
    allowed_aggregations: set = None  # {count, sum, avg} 等


class DataScopeLimiter:
    """
    数据范围限制器——确保 Agent 只能访问允许的数据子集
    """

    @staticmethod
    def apply_row_filter(query: str, scope: DataScope) -> str:
        """
        在 SQL 查询中自动插入行级过滤条件

        示例：
        原始:    SELECT * FROM orders
        应用后:  SELECT * FROM orders WHERE customer_id = 12345
        """
        if not scope.row_filter:
            return query

        # 简化的 SQL 注入——实际应使用参数化查询
        if "WHERE" in query.upper():
            return f"({query}) AND ({scope.row_filter})"
        else:
            return f"{query} WHERE {scope.row_filter}"

    @staticmethod
    def mask_fields(results: list, scope: DataScope, table: str) -> list:
        """
        对查询结果进行字段级别的遮盖

        将不允许的字段替换为 '***REDACTED***' 或 None
        """
        allowed = scope.allowed_fields.get(table, set())
        masked_results = []

        for row in results:
            masked_row = {}
            for field, value in row.items():
                if field in allowed:
                    masked_row[field] = value
                else:
                    masked_row[field] = "***REDACTED***"
            masked_results.append(masked_row)

        return masked_results

    @staticmethod
    def enforce_operation(operation: str, scope: DataScope) -> bool:
        """检查操作类型是否在允许范围内"""
        write_ops = {"INSERT", "UPDATE", "DELETE", "DROP", "ALTER", "TRUNCATE"}

        if scope.read_only:
            op_upper = operation.upper().split()[0]
            if op_upper in write_ops:
                return False

        return True


# ============================================================
# 使用示例
# ============================================================

customer_support_scope = DataScope(
    allowed_tables={"orders", "products", "tickets"},
    row_filter="customer_id = current_session_customer_id()",  # 动态绑定当前客户
    allowed_fields={
        "orders": {"id", "product", "amount", "status", "created_at"},
        "products": {"id", "name", "price", "stock_level"},
        "tickets": {"id", "subject", "status", "created_at", "resolved_at"},
    },
    read_only=True,
    max_records=50,
)
```

### 5. Time-Boxed Privileges — 时间盒权限

时间盒权限为权限授予设置**明确的过期时间**（TTL）。权限不是"授予后永久有效"，而是"授权后 30 秒自动回收"。

```
Time-Boxed Privilege 生命周期：

  授权                     到期自动回收
   │                          │
   ▼                          ▼
   ┌──────────────────────────┐
   │  权限有效窗口              │
   │                          │
   │  ┌──── 可续期 ────┐      │
   │  │ 如果任务未完成，   │      │
   │  │ 可申请延期        │      │
   │  └─────────────────┘      │
   └──────────────────────────┘
   ▲                          ▲
   │                          │
  TTL 开始                  TTL 到期
   (授予时刻)                 (自动回收)
```

**Key Design Decisions**：

```python
import time
import uuid
from dataclasses import dataclass, field
from typing import Optional, Callable
from enum import Enum


class PrivilegeState(Enum):
    GRANTED = "granted"
    EXPIRED = "expired"
    REVOKED = "revoked"
    PENDING = "pending"


@dataclass
class TimeBoxedPrivilege:
    """时间盒权限——带 TTL 的临时权限"""

    privilege_id: str = field(default_factory=lambda: f"priv_{uuid.uuid4().hex[:8]}")
    tool: str = ""
    params_scope: dict = field(default_factory=dict)
    data_scope: str = ""
    ttl_seconds: int = 30  # 默认 30 秒过期
    max_extensions: int = 2  # 最多续期次数
    granted_at: float = field(default_factory=time.time)
    expires_at: float = field(init=False)
    state: PrivilegeState = PrivilegeState.PENDING
    extension_count: int = 0

    def __post_init__(self):
        self.expires_at = self.granted_at + self.ttl_seconds

    @property
    def is_expired(self) -> bool:
        return time.time() > self.expires_at

    @property
    def remaining_seconds(self) -> float:
        return max(0.0, self.expires_at - time.time())

    def extend(self, additional_seconds: int = 30) -> bool:
        """
        续期——在最大续期次数内延长 TTL
        """
        if self.extension_count >= self.max_extensions:
            return False

        self.expires_at += additional_seconds
        self.extension_count += 1
        return True

    def revoke(self):
        """手动回收"""
        self.state = PrivilegeState.REVOKED
        self.expires_at = time.time()  # 立即过期

    def check_valid(self) -> bool:
        """检查权限是否仍然有效"""
        if self.state == PrivilegeState.REVOKED:
            return False
        if self.is_expired:
            self.state = PrivilegeState.EXPIRED
            return False
        return True


class TimeBoxedPrivilegeManager:
    """
    时间盒权限管理器——管理所有 Agent 的临时权限
    自动回收过期权限，支持续期和回调通知
    """

    def __init__(self):
        self._active_privileges: dict[str, TimeBoxedPrivilege] = {}
        self._on_revoke_callbacks: list[Callable] = []
        self._cleanup_interval = 60  # 每 60 秒清理一次
        self._last_cleanup = time.time()

    def grant(self, tool: str, params_scope: dict = None,
              data_scope: str = "", ttl: int = 30) -> TimeBoxedPrivilege:
        """授予一个时间盒权限"""
        privilege = TimeBoxedPrivilege(
            tool=tool,
            params_scope=params_scope or {},
            data_scope=data_scope,
            ttl_seconds=ttl,
        )
        privilege.state = PrivilegeState.GRANTED
        self._active_privileges[privilege.privilege_id] = privilege
        return privilege

    def verify_and_execute(self, privilege_id: str,
                           action: Callable, fallback: Callable = None) -> any:
        """
        验证权限有效后执行操作——权限+执行原子化
        """
        priv = self._active_privileges.get(privilege_id)

        if not priv or not priv.check_valid():
            if fallback:
                return fallback()
            raise PermissionError(f"Privilege {privilege_id} is expired or invalid")

        try:
            result = action()
            return result
        finally:
            self._auto_revoke_if_complete(priv)

    def register_revoke_callback(self, callback: Callable):
        """注册权限回收回调——可用于日志、监控、通知"""
        self._on_revoke_callbacks.append(callback)

    def _auto_revoke_if_complete(self, privilege: TimeBoxedPrivilege):
        """如果权限已不需要则自动回收"""
        privilege.revoke()
        for cb in self._on_revoke_callbacks:
            cb(privilege)

    def cleanup_expired(self):
        """清理过期权限——定期执行"""
        now = time.time()
        expired_ids = [
            pid for pid, priv in self._active_privileges.items()
            if priv.is_expired or priv.state in (PrivilegeState.EXPIRED, PrivilegeState.REVOKED)
        ]
        for pid in expired_ids:
            priv = self._active_privileges.pop(pid)
            for cb in self._on_revoke_callbacks:
                cb(priv)

        self._last_cleanup = now
```

### 6. Principle of Least Authority (POLA) — 最小授权原则

POLA 是对象能力模型（Object-Capability Model）在 Agent 权限中的应用。与传统的 ACL（访问控制列表）模型不同，POLA 的核心思想是：

> **如果你不能引用一个对象，你就不能操作它。**
> 权限不是"列表中的条目"，而是"你手中持有的钥匙"。

```
ACL 模型 vs POLA 模型

ACL 模型（传统）:
  Agent ──→ 权限检查 ──→ 资源
              │
              ▼
          权限列表
          {read:/data/files/,
           write:/data/tmp/}

  → 权限是 Agent 的属性，Agent 可以访问任何在列表中的资源
  → 如果 Agent 被攻破，攻击者可以读取整个列表中的资源

POLA 模型（对象能力）:
  Agent ──→ 调用工具 ──→ 检查能力引用 ──→ 执行
                           │
                           ▼
                       持有的"能力"对象
                       {cap: read_file('/data/report.pdf')}

  → 权限是调用者持有的能力（Capability），而非中心化的列表
  → Agent 只能操作它持有"能力引用"的对象
  → 能力（Capability）不能被伪造——你只能通过合法途径获得它
```

**POLA 在 Agent 中的实现**：

```python
class Capability:
    """
    不可伪造的能力（Capability）
    ——持有者即拥有权限，能力是运行时传递的
    """

    def __init__(self, action: str, resource: str,
                 params_constraint: dict = None, ttl: int = 30):
        self._action = action
        self._resource = resource
        self._params_constraint = params_constraint or {}
        self._expires_at = time.time() + ttl
        self._used = False
        self._nonce = uuid.uuid4().hex  # 防止能力伪造

    def invoke(self, params: dict = None) -> any:
        """使用能力执行操作"""
        if time.time() > self._expires_at:
            raise CapabilityExpiredError("Capability has expired")

        # 参数约束检查
        if self._params_constraint:
            self._check_params(params)

        self._used = True
        return self._execute(params)

    def _check_params(self, params: dict):
        """检查参数是否在能力允许的范围内"""
        for key, allowed_values in self._params_constraint.items():
            if key in params:
                if isinstance(allowed_values, list):
                    if params[key] not in allowed_values:
                        raise CapabilityViolation(
                            f"Parameter '{key}={params[key]}' not in "
                            f"allowed values: {allowed_values}"
                        )
                elif isinstance(allowed_values, type):
                    if not isinstance(params[key], allowed_values):
                        raise CapabilityViolation(
                            f"Parameter '{key}' must be of type {allowed_values}"
                        )

    def _execute(self, params: dict) -> any:
        """执行实际的操作——在子类中实现"""
        raise NotImplementedError

    def is_valid_for(self, action: str, resource: str) -> bool:
        """检查能力是否匹配请求的操作和资源"""
        return self._action == action and self._resource == resource


# 示例：创建具体的 Capability
class ReadFileCapability(Capability):
    """读取文件的能力——只能读取指定路径的文件"""

    def __init__(self, filepath: str):
        super().__init__(
            action="read_file",
            resource=filepath,
            params_constraint={
                "encoding": [str],
            },
            ttl=30,
        )
        self._filepath = filepath

    def _execute(self, params: dict) -> str:
        encoding = params.get("encoding", "utf-8")
        with open(self._filepath, "r", encoding=encoding) as f:
            return f.read()


# Agent 通过持有 Capability 来获得权限
class CapabilityBasedAgent:
    """基于能力的 Agent——只持有最小必要的能力集"""

    def __init__(self):
        self._capabilities: list[Capability] = []

    def grant_capability(self, cap: Capability):
        """注入能力——这是 Agent 获得权限的唯一方式"""
        self._capabilities.append(cap)

    def call_tool(self, action: str, resource: str, params: dict = None) -> any:
        """
        调用工具——只有在持有对应能力时才可执行
        """
        for cap in self._capabilities:
            if cap.is_valid_for(action, resource):
                return cap.invoke(params)

        raise PermissionError(
            f"No capability held for {action} on {resource}. "
            f"Current capabilities: "
            f"{[(c._action, c._resource) for c in self._capabilities]}"
        )
```

### 7. The "Hermes Blank Slate" Pattern — Hermes 空白石板模式

Hermes Blank Slate 模式（源自 Google Hermes 安全框架的概念）是最严格的最小权限实现：**每个 Agent 会话从零权限开始**。Agent 一开始什么都不能做，权限完全通过用户的显式请求或策略评估逐步授予。

**与传统模式的对比**：

```
传统模式（Partial Least Privilege）:
  ┌─────────────┐
  │ Agent 启动   │──→ 分配基础权限集（"可能需要的权限"）
  └─────────────┘
  每个 Agent 默认拥有一组"基线权限"
  用户或系统在 Agent 启动时已经授予了一定的信任

Blank Slate 模式（Strictest）:
  ┌─────────────┐
  │ Agent 启动   │──→ 权限为空集（"你什么都做不了"）
  └─────────────┘
  Agent 默认没有任何权限
  权限逐项、按需、临时授予
```

**工作流程**：

```
用户: "帮我查一下北京的天气"

Agent 会话启动
  │
  ▼
┌─────────────────────────────────────┐
│ Agent 状态：零权限                    │
│ 工具列表：[]（没有任何可用工具）        │
└─────────────────────────────────────┘
  │
  ▼
Agent 理解用户意图：需要查询天气
  │
  ▼
Agent 向权限系统请求：需要 weather.query 权限
  │
  ▼
权限系统评估：
  ├── 用户角色 → 允许
  ├── 任务类型 → 低风险
  ├── 数据范围 → 公共数据
  └── 决策 → 自动授予
  │
  ▼
┌─────────────────────────────────────┐
│ Agent 获得临时权限                     │
│ Capability: weather.query            │
│ TTL: 30s                             │
│ Scope: {"city": "Beijing"}           │
└─────────────────────────────────────┘
  │
  ▼
Agent 调用 weather.query(city="Beijing")
  → 返回天气数据
  → 权限在调用后立即回收
  │
  ▼
┌─────────────────────────────────────┐
│ Agent 状态：回到零权限                 │
│ Capabilities: []（用完即销毁）         │
└─────────────────────────────────────┘
  │
  ▼
Agent 继续处理——如果需要其他操作，再次请求权限
```

**实现原则**：

```python
class HermesBlankSlateAgent:
    """
    Hermes Blank Slate 模式——零权限启动，按需提权

    核心原则:
    1. 启动时零权限
    2. 权限必须基于明确的意图请求
    3. 权限是临时的一次性授予
    4. 权限用完立即回收
    5. 用户可以通过指令提升权限（但需要确认）
    """

    def __init__(self, privilege_manager: TimeBoxedPrivilegeManager):
        self._privilege_manager = privilege_manager
        self._active_privileges: list[TimeBoxedPrivilege] = []
        self._user_intent = None

    async def process_request(self, user_message: str):
        """
        处理用户请求——每次处理都遵循 Blank Slate 流程
        """
        # Phase 1: 理解意图（纯 LLM 推理，不需要任何权限）
        intent = await self._analyze_intent(user_message)
        self._user_intent = intent

        # Phase 2: 分解为工具调用计划（纯推理）
        plan = await self._decompose_into_tool_calls(intent)

        for step in plan:
            # Phase 3: 为每一步请求权限
            privilege = await self._request_privilege_for_step(step)

            if not privilege:
                # 权限不足——要么请求用户授权，要么降级处理
                step_result = await self._handle_insufficient_privilege(step)
            else:
                # Phase 4: 在权限范围内执行
                step_result = await self._execute_with_privilege(step, privilege)

            # Phase 5: 权限自动回收（通过 TTL 或手动）
            self._active_privileges = [
                p for p in self._active_privileges if not p.check_valid()
            ]

        # Phase 6: 确保所有权限已回收
        assert len(self._active_privileges) == 0, "Leaked privileges detected!"

    async def _analyze_intent(self, message: str) -> dict:
        """分析用户意图——纯推理，不需要任何权限"""
        # 调用 LLM 进行意图识别
        return {
            "action": "query_weather",
            "params": {"city": "Beijing"},
            "risk_level": "low",
        }

    async def _request_privilege_for_step(self, step: dict) -> Optional[TimeBoxedPrivilege]:
        """向权限系统请求特定步骤的权限"""
        tool_name = step.get("tool")
        params = step.get("params", {})

        # 自动评估——低风险操作自动授权
        if self._auto_approve(step):
            privilege = self._privilege_manager.grant(
                tool=tool_name,
                params_scope=params,
                ttl=30,
            )
            self._active_privileges.append(privilege)
            return privilege

        # 中高风险操作需要用户确认
        user_approved = await self._request_user_approval(tool_name, params)
        if user_approved:
            privilege = self._privilege_manager.grant(
                tool=tool_name,
                params_scope=params,
                ttl=60,
            )
            self._active_privileges.append(privilege)
            return privilege

        return None
```

## 示例代码

```python
"""
LeastPrivilegeManager — Agent 最小权限管理器
集成了 ZSP、Just-in-Time 权限提升、自动回收
"""

import time
import uuid
import json
import logging
from dataclasses import dataclass, field
from typing import Optional, Callable, Awaitable
from enum import Enum, auto


# ─── 基础类型 ─────────────────────────────────────────

class PrivilegeLevel(Enum):
    """权限风险等级"""
    LEVEL_0_READONLY = auto()       # L0: 只读，自动授予
    LEVEL_1_LOW_RISK_WRITE = auto() # L1: 低风险写，自动授权
    LEVEL_2_MEDIUM_RISK = auto()    # L2: 中风险，需上下文验证
    LEVEL_3_HIGH_RISK = auto()      # L3: 高风险，需用户确认
    LEVEL_4_DANGEROUS = auto()      # L4: 危险，默认拒绝


@dataclass
class PrivilegeRequest:
    """权限请求"""
    tool_name: str
    params: dict
    user_id: str
    session_id: str
    task_context: str
    risk_level: PrivilegeLevel
    request_id: str = field(default_factory=lambda: f"req_{uuid.uuid4().hex[:8]}")

    def can_auto_approve(self) -> bool:
        """是否可以自动批准"""
        return self.risk_level in (
            PrivilegeLevel.LEVEL_0_READONLY,
            PrivilegeLevel.LEVEL_1_LOW_RISK_WRITE,
        )


@dataclass
class PrivilegeGrant:
    """权限授予结果"""
    grant_id: str = field(default_factory=lambda: f"grant_{uuid.uuid4().hex[:8]}")
    request_id: str = ""
    tool_name: str = ""
    params_scope: dict = field(default_factory=dict)
    granted_at: float = field(default_factory=time.time)
    expires_at: float = 0.0
    max_calls: int = 1            # 最多调用次数
    call_count: int = 0
    revoked: bool = False

    @property
    def is_valid(self) -> bool:
        return (
            not self.revoked
            and time.time() < self.expires_at
            and self.call_count < self.max_calls
        )

    def use(self) -> bool:
        """消耗一次调用机会"""
        if not self.is_valid:
            return False
        self.call_count += 1
        return True


class PrivilegeAuditor:
    """权限审计器——记录所有权限事件"""

    def __init__(self):
        self._events: list[dict] = []
        self._anomalies: list[dict] = []

    def record_grant(self, request: PrivilegeRequest, grant: PrivilegeGrant,
                     auto_approved: bool):
        """记录权限授予"""
        self._events.append({
            "type": "grant",
            "timestamp": time.time(),
            "request_id": request.request_id,
            "grant_id": grant.grant_id,
            "tool": request.tool_name,
            "risk_level": request.risk_level.name,
            "auto_approved": auto_approved,
            "user": request.user_id,
        })

    def record_revocation(self, grant_id: str, reason: str):
        """记录权限回收"""
        self._events.append({
            "type": "revoke",
            "timestamp": time.time(),
            "grant_id": grant_id,
            "reason": reason,
        })

    def record_denial(self, request: PrivilegeRequest, reason: str):
        """记录权限拒绝"""
        event = {
            "type": "deny",
            "timestamp": time.time(),
            "request_id": request.request_id,
            "tool": request.tool_name,
            "risk_level": request.risk_level.name,
            "reason": reason,
            "user": request.user_id,
        }
        self._events.append(event)

        # 检测异常模式
        self._check_anomalies(event)

    def _check_anomalies(self, event: dict):
        """检测权限相关的异常模式"""
        # 高频拒绝——可能是在试探权限边界
        recent_denials = [
            e for e in self._events
            if e["type"] == "deny"
            and e["user"] == event["user"]
            and time.time() - e["timestamp"] < 60
        ]
        if len(recent_denials) > 10:
            self._anomalies.append({
                "type": "high_frequency_denials",
                "user": event["user"],
                "count": len(recent_denials),
                "timestamp": time.time(),
                "severity": "medium",
            })

    def get_anomaly_report(self) -> list[dict]:
        """获取异常报告"""
        return self._anomalies


class LeastPrivilegeManager:
    """
    最小权限管理器——Agent 权限管理的核心

    核心能力:
    - Zero Standing Privileges: Agent 初始零权限
    - Just-in-Time 权限提升: 按需授予临时权限
    - 自动回收: 权限到期或使用后自动回收
    - 参数级约束: 限制工具调用的参数范围
    - 审计: 记录所有权限事件
    """

    # 工具的风险等级配置
    TOOL_RISK_MAP = {
        # L0: 只读操作
        "search_web": PrivilegeLevel.LEVEL_0_READONLY,
        "weather.query": PrivilegeLevel.LEVEL_0_READONLY,
        "knowledge_base.search": PrivilegeLevel.LEVEL_0_READONLY,
        "file.read": PrivilegeLevel.LEVEL_0_READONLY,
        # L1: 低风险写
        "notes.create": PrivilegeLevel.LEVEL_1_LOW_RISK_WRITE,
        "notes.update": PrivilegeLevel.LEVEL_1_LOW_RISK_WRITE,
        "calendar.create_draft": PrivilegeLevel.LEVEL_1_LOW_RISK_WRITE,
        # L2: 中风险
        "email.send": PrivilegeLevel.LEVEL_2_MEDIUM_RISK,
        "file.write": PrivilegeLevel.LEVEL_2_MEDIUM_RISK,
        "database.query": PrivilegeLevel.LEVEL_2_MEDIUM_RISK,
        # L3: 高风险
        "file.delete": PrivilegeLevel.LEVEL_3_HIGH_RISK,
        "email.send_bulk": PrivilegeLevel.LEVEL_3_HIGH_RISK,
        "payment.transfer": PrivilegeLevel.LEVEL_3_HIGH_RISK,
        # L4: 危险操作
        "shell.execute": PrivilegeLevel.LEVEL_4_DANGEROUS,
        "system.config_modify": PrivilegeLevel.LEVEL_4_DANGEROUS,
        "admin.user_create": PrivilegeLevel.LEVEL_4_DANGEROUS,
    }

    # 参数级约束规则
    PARAM_CONSTRAINTS = {
        "email.send": {
            "to": {
                "type": "domain_whitelist",
                "allowed_domains": ["company.com"],
                "message": "只能发送到公司域名邮箱",
            },
            "cc": {
                "type": "deny_all",
                "message": "不允许抄送",
            },
            "attachments": {
                "type": "max_size_mb",
                "max_mb": 5,
                "message": "附件不能超过 5MB",
            },
        },
        "file.read": {
            "path": {
                "type": "path_prefix",
                "allowed_prefixes": ["/data/reports/", "/data/shared/"],
                "message": "只能读取 /data/reports/ 和 /data/shared/ 目录下的文件",
            },
        },
        "database.query": {
            "sql": {
                "type": "forbidden_keywords",
                "keywords": ["DROP", "ALTER", "TRUNCATE", "DELETE", "INSERT", "UPDATE"],
                "message": "只允许 SELECT 查询",
            },
            "limit": {
                "type": "max_value",
                "max": 1000,
                "message": "查询结果不能超过 1000 行",
            },
        },
    }

    def __init__(self):
        self._auditor = PrivilegeAuditor()
        self._active_grants: dict[str, PrivilegeGrant] = {}
        self._on_approval_callback: Optional[Callable[[PrivilegeRequest], Awaitable[bool]]] = None
        self._logger = logging.getLogger("LeastPrivilegeManager")

    def set_approval_callback(self, callback: Callable[[PrivilegeRequest], Awaitable[bool]]):
        """设置审批回调——用于向用户发送审批请求"""
        self._on_approval_callback = callback

    async def request_privilege(self, tool_name: str, params: dict,
                                user_id: str, session_id: str,
                                task_context: str = "") -> Optional[PrivilegeGrant]:
        """
        请求工具调用权限——Just-in-Time 权限提升的入口

        流程:
        1. 检查工具的风险等级
        2. 检查参数约束
        3. 低风险自动授权 / 高风险请求审批
        4. 授予临时权限（含 TTL）
        """
        risk_level = self.TOOL_RISK_MAP.get(tool_name, PrivilegeLevel.LEVEL_2_MEDIUM_RISK)

        request = PrivilegeRequest(
            tool_name=tool_name,
            params=params,
            user_id=user_id,
            session_id=session_id,
            task_context=task_context,
            risk_level=risk_level,
        )

        # Step 1: 参数约束检查
        param_check = self._check_param_constraints(tool_name, params)
        if not param_check["allowed"]:
            self._auditor.record_denial(request, param_check["reason"])
            self._logger.warning(
                f"参数约束拒绝: {tool_name} | {param_check['reason']}"
            )
            return None

        # Step 2: 权限评估
        if request.can_auto_approve():
            # 低风险——自动授予
            grant = self._create_grant(request, ttl=30)
            self._auditor.record_grant(request, grant, auto_approved=True)
            self._logger.info(f"自动授权: {tool_name} | grant_id={grant.grant_id}")
            return grant

        # Step 3: 中高风险——需要审批
        if self._on_approval_callback:
            approved = await self._on_approval_callback(request)
            if approved:
                ttl = 60 if risk_level == PrivilegeLevel.LEVEL_2_MEDIUM_RISK else 30
                if risk_level == PrivilegeLevel.LEVEL_4_DANGEROUS:
                    ttl = 15  # 危险操作只给极短 TTL
                grant = self._create_grant(request, ttl=ttl)
                self._auditor.record_grant(request, grant, auto_approved=False)
                self._logger.info(f"人工授权: {tool_name} | grant_id={grant.grant_id}")
                return grant
            else:
                self._auditor.record_denial(request, "用户拒绝审批")
                self._logger.info(f"用户拒绝: {tool_name}")
                return None
        else:
            # 没有审批回调，默认拒绝中高风险操作
            self._auditor.record_denial(request, "无审批回调配置，默认拒绝")
            self._logger.warning(f"默认拒绝: {tool_name}（无审批回调）")
            return None

    def verify_and_call(self, grant_id: str, tool_func: Callable,
                        params: dict) -> any:
        """
        验证权限后调用工具——权限验证+执行原子化

        Args:
            grant_id: 权限授予 ID
            tool_func: 工具调用函数
            params: 调用参数（必须在权限范围内）

        Returns:
            工具调用结果

        Raises:
            PermissionError: 权限无效或参数超出范围
        """
        grant = self._active_grants.get(grant_id)
        if not grant:
            raise PermissionError(f"授权 {grant_id} 不存在")

        if not grant.is_valid:
            self._logger.warning(
                f"权限已失效: {grant_id} | "
                f"expired={time.time() > grant.expires_at}, "
                f"calls={grant.call_count}/{grant.max_calls}"
            )
            self._auditor.record_revocation(grant_id, "权限已失效")
            raise PermissionError(f"授权 {grant_id} 已过期或已达最大调用次数")

        # 检查参数是否超出授权的参数范围
        self._validate_params_scope(params, grant.params_scope)

        if not grant.use():
            raise PermissionError(f"授权 {grant_id} 已耗尽调用次数")

        result = tool_func(**params)

        # 如果是单次调用权限，调用后自动标记回收
        if grant.call_count >= grant.max_calls:
            self.revoke_privilege(grant_id, "调用次数用尽")

        return result

    def revoke_privilege(self, grant_id: str, reason: str = "手动回收"):
        """回收权限——立即失效"""
        grant = self._active_grants.get(grant_id)
        if grant:
            grant.revoked = True
            self._active_grants.pop(grant_id, None)
            self._auditor.record_revocation(grant_id, reason)
            self._logger.info(f"权限回收: {grant_id} | reason={reason}")

    def cleanup_session(self, session_id: str):
        """清除会话的所有权限——会话结束时的清理操作"""
        session_grants = [
            gid for gid, g in self._active_grants.items()
            if g.request_id and session_id in g.request_id  # simplified
        ]
        for gid in session_grants:
            self.revoke_privilege(gid, "会话结束")

    def get_audit_report(self) -> dict:
        """获取审计报告"""
        return {
            "events": self._auditor._events,
            "anomalies": self._auditor.get_anomaly_report(),
            "active_grants_count": len(self._active_grants),
        }

    # ─── 内部方法 ─────────────────────────────────

    def _check_param_constraints(self, tool: str, params: dict) -> dict:
        """检查参数约束"""
        constraints = self.PARAM_CONSTRAINTS.get(tool, {})
        for param_name, constraint in constraints.items():
            if param_name in params:
                value = params[param_name]
                rule_type = constraint["type"]

                if rule_type == "domain_whitelist":
                    allowed = constraint["allowed_domains"]
                    if isinstance(value, str):
                        domain = value.split("@")[-1] if "@" in value else value
                        if domain not in allowed:
                            return {"allowed": False, "reason": constraint["message"]}

                elif rule_type == "deny_all":
                    return {"allowed": False, "reason": constraint["message"]}

                elif rule_type == "path_prefix":
                    allowed_prefixes = constraint["allowed_prefixes"]
                    if not any(value.startswith(p) for p in allowed_prefixes):
                        return {"allowed": False, "reason": constraint["message"]}

                elif rule_type == "forbidden_keywords":
                    if isinstance(value, str):
                        value_upper = value.upper()
                        for kw in constraint["keywords"]:
                            if kw in value_upper:
                                return {"allowed": False, "reason": constraint["message"]}

                elif rule_type == "max_value":
                    if isinstance(value, (int, float)) and value > constraint["max"]:
                        return {"allowed": False, "reason": constraint["message"]}

        return {"allowed": True}

    def _create_grant(self, request: PrivilegeRequest, ttl: int = 30) -> PrivilegeGrant:
        """创建权限授予记录"""
        grant = PrivilegeGrant(
            request_id=request.request_id,
            tool_name=request.tool_name,
            params_scope=request.params,
            expires_at=time.time() + ttl,
            max_calls=1 if request.risk_level == PrivilegeLevel.LEVEL_4_DANGEROUS else 5,
        )
        self._active_grants[grant.grant_id] = grant
        return grant

    def _validate_params_scope(self, actual_params: dict, granted_scope: dict):
        """验证实际参数是否在授予的参数范围内"""
        for key, value in actual_params.items():
            if key in granted_scope:
                granted_value = granted_scope[key]
                # 如果授予的参数是精确值，实际参数必须匹配
                if isinstance(granted_value, str):
                    if value != granted_value:
                        raise PermissionError(
                            f"参数 '{key}' 值 '{value}' 超出授权范围 "
                            f"(允许: '{granted_value}')"
                        )


# ============================================================
# 使用示例
# ============================================================

async def demo_least_privilege():
    """演示最小权限管理器的工作流程"""

    manager = LeastPrivilegeManager()

    # 设置审批回调
    async def approval_handler(request: PrivilegeRequest) -> bool:
        print(f"\n  [审批请求] 工具: {request.tool_name}")
        print(f"  [审批请求] 参数: {request.params}")
        print(f"  [审批请求] 风险等级: {request.risk_level.name}")
        # 在实际系统中，这里会向用户发送确认请求
        return True  # 模拟用户批准

    manager.set_approval_callback(approval_handler)

    # ── 场景 1: 只读操作 —— 自动授权 ──
    print("\n" + "=" * 60)
    print("场景 1: 只读操作 (自动授权)")
    print("=" * 60)

    grant1 = await manager.request_privilege(
        tool_name="weather.query",
        params={"city": "Beijing"},
        user_id="user_123",
        session_id="session_abc",
        task_context="用户查询天气",
    )
    if grant1:
        print(f"  ✓ 获得授权: {grant1.grant_id}")
        print(f"    有效期至: {time.ctime(grant1.expires_at)}")

        # 模拟工具调用
        def query_weather(city: str):
            return f"晴天，25°C，{city}"

        result = manager.verify_and_call(grant1.grant_id, query_weather, {"city": "Beijing"})
        print(f"  执行结果: {result}")

    # ── 场景 2: 参数约束拒绝 ──
    print("\n" + "=" * 60)
    print("场景 2: 参数约束拒绝")
    print("=" * 60)

    grant2 = await manager.request_privilege(
        tool_name="email.send",
        params={
            "to": "attacker@gmail.com",
            "subject": "Hello",
            "body": "Test",
        },
        user_id="user_123",
        session_id="session_abc",
    )
    if grant2 is None:
        print("  ✓ 参数约束生效——邮件发送被拒绝（非公司域名）")

    # ── 场景 3: ZSP 自动回收 ──
    print("\n" + "=" * 60)
    print("场景 3: 权限使用后自动回收")
    print("=" * 60)

    grant3 = await manager.request_privilege(
        tool_name="notes.create",
        params={"title": "Test Note", "content": "Hello"},
        user_id="user_123",
        session_id="session_abc",
    )
    if grant3:
        def create_note(title: str, content: str):
            return f"笔记创建成功: {title}"

        # 第一次调用
        result1 = manager.verify_and_call(grant3.grant_id, create_note,
                                          {"title": "Test", "content": "Hello"})
        print(f"  第一次调用: {result1}")

        # 尝试使用已回收的权限
        try:
            result2 = manager.verify_and_call(grant3.grant_id, create_note,
                                              {"title": "Another", "content": "World"})
        except PermissionError as e:
            print(f"  ✓ 权限已回收，拒绝重复使用: {e}")

    # ── 场景 4: 高风险操作需审批 ──
    print("\n" + "=" * 60)
    print("场景 4: 高风险操作 (需要用户审批)")
    print("=" * 60)

    grant4 = await manager.request_privilege(
        tool_name="file.delete",
        params={"path": "/data/tmp/old_report.pdf"},
        user_id="user_123",
        session_id="session_abc",
        task_context="用户要求删除过时文件",
    )
    if grant4:
        print(f"  ✓ 用户批准，获得权限: {grant4.grant_id}")
        print(f"    危险操作 TTL: {grant4.expires_at - grant4.granted_at:.0f}s")
    else:
        print("  ╳ 用户拒绝")
```

## 能力边界

**最小权限能做到的：**

- 将攻击面从"Agent 的所有可用工具"缩小到"当前任务必需的少量工具"
- 即使 Agent 被 Prompt 注入完全控制，权限限制也能阻止大多数破坏性操作
- 通过 ZSP 将权限暴露窗口从"整个会话"缩短到"单次工具调用的几十毫秒"
- 通过参数约束防止 Agent 使用工具的"危险模式"（如发送邮件到外部域名）
- 通过细粒度审计追踪每次权限使用，为事后分析和取证提供完整记录

**最小权限做不到的：**

- ❌ 无法防御**权限内的攻击**——如果 Agent 需要的权限本身就是危险的（如 "delete_file"），最小权限无法阻止 Agent 在权限范围内删除文件
- ❌ 无法防御**权限提升攻击**——如果 Agent 可以调用 "admin.grant_permission" 来给自己提权，最小权限机制本身会被绕过
- ❌ 无法解决**权限粒度的根本矛盾**——粒度过粗则失去保护意义，粒度过细则管理复杂且易误判
- ❌ 无法自动判断**权限的"最小"边界**——确定 "Agent 到底需要什么权限" 本身是一个困难问题，需要人工策略配置
- ❌ 无法防御**组合攻击**——多个低风险权限组合可能造成高风险后果（例如：读文件 + 发邮件 = 数据泄露），这种"权限组合"风险难以静态检测
- ❌ 无法避免**审批疲劳**——频繁的权限审批提示会导致用户不加思考地批准所有请求，使最小权限机制失效

**粒度限制与复杂性权衡：**

| 粒度 | 安全效果 | 管理复杂度 | 运行时开销 | 适用场景 |
|------|---------|-----------|-----------|---------|
| 工具级 | 低 | 极低 | 无 | 低风险 Agent（简单问答） |
| 参数级 | 中 | 低 | <1ms | 中等风险 Agent（客服、内容生成） |
| 数据级 | 高 | 中 | 1-5ms | 高风险 Agent（数据库访问、文件操作） |
| 执行级 | 极高 | 高 | 5-20ms | 极高风险场景（金融交易、系统管理） |

## 传统最小权限 vs Agent 最小权限

| 维度 | 传统最小权限 | Agent 最小权限 |
|------|------------|---------------|
| **权限主体** | 用户、进程、服务账户 | LLM + 工具调用链 + 外部数据源 |
| **授权时机** | 系统启动时 / 用户登录时 | 每次工具调用前（Just-in-Time） |
| **权限粒度** | 文件级、API 级、数据库级 | 工具级 + 参数级 + 数据级 + 执行级 |
| **权限时效** | 持久化（持续到手动回收） | 临时（TTL 到期自动回收） |
| **权限判定依据** | 用户角色、组成员身份 | 对话上下文、用户意图、任务风险、历史行为 |
| **默认状态** | 有限权限（"基线权限"） | 零权限（ZSP, Blank Slate） |
| **权限变更** | 需管理员手动操作 | 运行时自动评估 + 用户审批 |
| **攻击面** | 用户凭证泄露 | Prompt 注入 + 工具调用劫持 + 凭证泄露 |
| **权限审计** | 登录日志、操作日志 | 每次权限授予 + 每次工具调用 + 每次拒绝 |
| **组合风险** | 很少考虑 | 需要检测权限组合的隐性风险 |
| **成熟度** | 40+ 年，理论成熟，工具有效 | 3 年，仍在快速演化，标准未定 |
| **典型实现** | UNIX 权限、RBAC、AWS IAM | ZSP 引擎、POLA 能力模型、运行时策略评估器 |

## 工程优化方向

### 1. 自动化权限分析

通过静态和动态分析，自动确定 Agent 任务所需的最小权限集，减少人工策略配置的工作量：

```python
class AutomatedPrivilegeAnalyzer:
    """
    自动化权限分析器——通过分析 Agent 的任务计划来推断所需权限
    """

    def analyze_task_plan(self, task_plan: list[dict]) -> dict:
        """
        分析任务计划（工具调用序列）并推断所需的最小权限集

        Args:
            task_plan: [
                {"tool": "search_web", "params": {"query": "..."}},
                {"tool": "weather.query", "params": {"city": "..."}},
            ]

        Returns:
            {"tools": {...}, "params": {...}, "data": {...}}
        """
        required_tools = {}
        for step in task_plan:
            tool = step["tool"]
            params = step.get("params", {})

            if tool not in required_tools:
                # 对于字符串参数，只记录"这是一个字符串"而非实际值
                param_shape = {
                    k: type(v).__name__ for k, v in params.items()
                }
                required_tools[tool] = {
                    "count": 0,
                    "param_shape": param_shape,
                    "sample_params": params,
                }
            required_tools[tool]["count"] += 1

        return required_tools

    def recommend_privileges(self, tool_usage: dict) -> dict:
        """
        根据使用模式推荐权限配置
        """
        recommendations = {}
        for tool, usage in tool_usage.items():
            rec = {
                "risk_level": LeastPrivilegeManager.TOOL_RISK_MAP.get(
                    tool, PrivilegeLevel.LEVEL_2_MEDIUM_RISK
                ).name,
                "suggested_ttl": max(15, usage["count"] * 10),
                "suggested_max_calls": usage["count"] + 2,  # 预留余量
                "param_constraints": self._infer_param_constraints(
                    tool, usage["param_shape"]
                ),
            }
            recommendations[tool] = rec

        return recommendations

    def _infer_param_constraints(self, tool: str, param_shape: dict) -> dict:
        """根据参数类型推断可能的约束"""
        # 从历史使用模式中学习参数约束
        # 简化实现：标记所有字符串参数为"需要确认"
        constraints = {}
        for param, ptype in param_shape.items():
            if ptype == "str":
                constraints[param] = "verify_on_use"
        return constraints
```

### 2. 权限使用监控

实时监控 Agent 的权限使用情况，检测异常行为：

```python
class PrivilegeMonitor:
    """
    权限使用监控器——实时监控、异常检测、报表生成
    """

    def __init__(self):
        self._usage_stats: dict[str, list] = {}  # tool_name -> [timestamps]
        self._denial_stats: dict[str, int] = {}   # tool_name -> count
        self._anomaly_threshold = {
            "denial_rate_per_minute": 10,    # 每分钟拒绝超过 10 次
            "tool_calls_per_minute": 100,     # 每分钟工具调用超过 100 次
            "escalation_frequency": 5,        # 每分钟权限提升超过 5 次
        }

    def record_usage(self, tool: str, user: str, session: str):
        """记录一次权限使用"""
        now = time.time()
        if tool not in self._usage_stats:
            self._usage_stats[tool] = []
        self._usage_stats[tool].append({
            "timestamp": now,
            "user": user,
            "session": session,
        })

    def record_denial(self, tool: str, reason: str):
        """记录一次权限拒绝"""
        self._denial_stats[tool] = self._denial_stats.get(tool, 0) + 1

    def check_anomaly(self, tool: str) -> Optional[str]:
        """
        检查工具使用是否存在异常

        Returns: 异常描述字符串，或 None 表示正常
        """
        now = time.time()

        # 检查工具调用频率
        recent_calls = [
            u for u in self._usage_stats.get(tool, [])
            if now - u["timestamp"] < 60
        ]
        if len(recent_calls) > self._anomaly_threshold["tool_calls_per_minute"]:
            return (
                f"工具 '{tool}' 在最近 1 分钟内被调用了 {len(recent_calls)} 次, "
                f"超过阈值 {self._anomaly_threshold['tool_calls_per_minute']}"
            )

        return None

    def generate_report(self, window_minutes: int = 60) -> dict:
        """生成权限使用报告"""
        now = time.time()
        window = window_minutes * 60

        top_tools = sorted(
            [
                (tool, len([
                    u for u in usages if now - u["timestamp"] < window
                ]))
                for tool, usages in self._usage_stats.items()
            ],
            key=lambda x: x[1],
            reverse=True,
        )[:10]

        return {
            "window_minutes": window_minutes,
            "total_calls": sum(len(v) for v in self._usage_stats.values()),
            "total_denials": sum(self._denial_stats.values()),
            "top_tools": top_tools,
            "denial_by_tool": dict(sorted(
                self._denial_stats.items(),
                key=lambda x: x[1],
                reverse=True,
            )[:5]),
        }
```

### 3. 权限权限右尺寸调整（Rightsizing Recommendations）

基于历史使用数据自动调整权限配置——移除从未使用的权限，缩小过度授权的范围：

```python
class PrivilegeRightsizer:
    """
    权限右尺寸调整器——基于实际使用数据优化权限配置

    核心策略:
    1. 移除 30 天内未使用的权限
    2. 缩小 TTL——如果平均使用时间 < 当前 TTL
    3. 降低 max_calls——如果实际调用次数 < 授予次数
    4. 提升风险等级——如果工具从未被拒绝过
    """

    def __init__(self, usage_history: list[dict]):
        self.history = usage_history

    def generate_recommendations(self) -> list[dict]:
        """生成权限调整建议"""
        recommendations = []

        # 按工具分组分析
        tool_stats = self._analyze_by_tool()

        for tool, stats in tool_stats.items():
            recs = []

            # 建议 1: 移除未使用的权限
            if stats["total_calls"] == 0:
                recs.append({
                    "type": "remove",
                    "tool": tool,
                    "reason": f"30 天内从未使用",
                    "safety_impact": "降低攻击面",
                })

            # 建议 2: 缩短 TTL
            avg_duration = stats["avg_duration_seconds"]
            current_ttl = stats["current_ttl"]
            if avg_duration and current_ttl and avg_duration * 2 < current_ttl:
                recs.append({
                    "type": "reduce_ttl",
                    "tool": tool,
                    "from": current_ttl,
                    "to": int(avg_duration * 3),  # 3 倍平均时间作为缓冲
                    "reason": f"平均使用时长 {avg_duration:.1f}s, 远小于当前 TTL {current_ttl}s",
                    "safety_impact": f"权限暴露窗口缩短 {current_ttl - int(avg_duration * 3)}s",
                })

            # 建议 3: 收紧参数约束
            unused_params = stats.get("unused_params", [])
            if unused_params:
                recs.append({
                    "type": "restrict_params",
                    "tool": tool,
                    "params_to_remove": unused_params,
                    "reason": f"以下参数从未在历史使用中出现: {unused_params}",
                    "safety_impact": "参数级攻击面减小",
                })

            if recs:
                recommendations.extend(recs)

        return recommendations

    def _analyze_by_tool(self) -> dict:
        """按工具分析使用统计数据"""
        tool_data = {}
        for entry in self.history:
            tool = entry["tool"]
            if tool not in tool_data:
                tool_data[tool] = {
                    "total_calls": 0,
                    "durations": [],
                    "used_params": set(),
                    "current_ttl": entry.get("ttl", 30),
                }
            tool_data[tool]["total_calls"] += 1
            tool_data[tool]["durations"].append(entry.get("duration", 0))
            tool_data[tool]["used_params"].update(entry.get("params", {}).keys())

        # 计算平均值
        for tool, data in tool_data.items():
            durations = data["durations"]
            data["avg_duration_seconds"] = (
                sum(durations) / len(durations) if durations else None
            )

        return tool_data
```

### 4. 生产环境检查清单

- [ ] 是否实现了 Zero Standing Privileges（Agent 启动时无任何权限）？
- [ ] 是否对每个工具进行了风险等级分类（L0-L4）？
- [ ] 是否实现了参数级约束（域名白名单、路径前缀、禁止关键词）？
- [ ] 低风险操作是否实现了自动授权（无需用户确认）？
- [ ] 高风险操作是否要求用户明确确认？
- [ ] 危险操作是否默认拒绝+最短 TTL？
- [ ] 权限是否设置了 TTL 并在到期后自动回收？
- [ ] 单次调用权限是否在调用后立即回收？
- [ ] 是否实现了 Toolset Pinning（工具集固定）？
- [ ] 是否实现了数据范围限制（行级、字段级）？
- [ ] 是否注册了权限回收回调（日志、监控）？
- [ ] 是否实现了权限使用监控和异常检测？
- [ ] 是否有定期的权限审计报告？
- [ ] 是否有自动化权限分析工具来辅助策略配置？
- [ ] 是否检测并限制了权限组合的隐性风险？
- [ ] 是否考虑了审批疲劳的缓解策略？
- [ ] Agent 工具调用链中的中间结果是否受到权限约束？
- [ ] 会话结束时是否进行了权限清理？
