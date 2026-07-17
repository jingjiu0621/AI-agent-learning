# 12.4.1 tool-authorization — 工具授权

**工具授权决定了 Agent "能用什么工具、怎么用"。** 如果说权限控制（12.4）是定义 Agent 能做什么的"政策"，那么工具授权就是执行这些政策的"机制"——它确保 Agent 调用工具的每次请求都经过了身份验证、授权判定和策略执行。没有严格的工具授权，最小权限原则（12.4.2）就是空谈。

## 简单介绍

工具授权（Tool Authorization）是 Agent 安全架构中连接"权限策略"和"工具执行"的关键桥梁。传统 API 授权关注的是"用户是否有权调用这个 API"，而 Agent 工具授权需要回答一个更复杂的问题："当前 Agent 实例、在本次对话上下文中、为完成当前任务、是否可以使用这个工具的特定功能？"

```
  权限策略 (WHAT)                   工具授权 (HOW)                   工具执行 (DO)
  ┌──────────────┐                ┌────────────────┐              ┌──────────────┐
  │ Agent 可以    │                │ 验证 Agent 身份  │              │ 执行工具调用  │
  │ 访问文件系统  │ ────策略输入───►│ 检查工具访问权限  │ ────授权────►│              │
  │ Agent 可以    │                │ 检查参数合法性   │   通过       │ 返回执行结果  │
  │ 使用 search   │                │ 检查调用上下文   │              │              │
  └──────────────┘                └────────────────┘              └──────────────┘
                                          │
                                          ▼
                                   ┌──────────────┐
                                   │   授权拒绝    │
                                   │ • 无权限拒绝  │
                                   │ • 参数越界拒绝 │
                                   │ • 人工审批中   │
                                   └──────────────┘
```

Agent 工具授权涉及的技术栈覆盖了从传统 API 授权（OAuth 2.0、API Key）到现代 Agent 安全协议（MCP 安全模型、工具级 RBAC/ABAC），需要解决的核心问题包括：Agent 身份的认证与委托、工具访问的细粒度控制、动态上下文的授权决策、以及授权结果的高效缓存与撤销。

## 基本原理 — authorization vs authentication, OAuth2/MAC/RBAC/ABAC for tools

### Authentication vs Authorization

这是理解工具授权最基础的区别：

| 概念 | Authentication（认证） | Authorization（授权） |
|------|----------------------|----------------------|
| 回答的问题 | "你是谁？" | "你能做什么？" |
| Agent 场景 | Agent 的身份凭证、所属用户 | Agent 能调用哪些工具、哪些参数 |
| 机制 | API Key, JWT, OAuth2 Token, mTLS | RBAC, ABAC, Scope, Policy |
| 时机 | 每次请求先认证 | 认证后判定授权 |
| 失败结果 | 401 Unauthorized（实际是未认证） | 403 Forbidden |

在 Agent 工具授权利，两者的关系是：

```
用户认证 ──► Agent 身份凭证 ──► 工具授权决策
  │                                 │
  │ 用户登录获取 Token              │ Agent 用 Token 请求工具调用
  ▼                                 ▼
┌────────┐                    ┌──────────────┐
│ AuthN  │                    │    AuthZ     │
│        │                    │              │
│ • 用户 │                    │ • Agent 身份  │
│ • OAuth│                    │ • 工具作用域  │
│ • SSO  │                    │ • 参数校验   │
└────────┘                    └──────────────┘
```

### OAuth 2.0 在工具授权中的应用

OAuth 2.0 是 Agent 工具授权中最核心的委托授权协议。Agent 作为 OAuth 2.0 的 Client，代表用户访问受保护的工具资源：

```
  ┌──────┐          ┌──────────┐          ┌───────────┐
  │ User │          │  Agent   │          │ Tool API  │
  └──┬───┘          └────┬─────┘          └─────┬─────┘
     │                   │                      │
     │ 1. 用户登录 Agent  │                      │
     ├──────────────────►│                      │
     │                   │                      │
     │ 2. Agent 请求授权 │                      │
     │    (scope: tools) │                      │
     ├──────────────────►│                      │
     │                   │                      │
     │ 3. 用户授权 Agent │                      │
     │    (批准工具范围)   │                      │
     ├──────────────────►│                      │
     │                   │                      │
     │ 4. Agent 获取     │                      │
     │    Access Token   │                      │
     │    (受限工具权限)   │                      │
     ├──────────────────►│                      │
     │                   │                      │
     │      5. Agent 用 Token 调用工具 ─────────►│
     │                   │                      │
     │                   │   6. 验证 Token      │
     │                   │     ┌──────────┐     │
     │                   │     │ AuthZ    │     │
     │                   │     │ Server   │     │
     │                   │     │ • Scope  │     │
     │                   │     │ • Expiry │     │
     │                   │     │ • Revoke │     │
     │                   │     └──────────┘     │
     │                   │                      │
     │                   │   7. 返回结果 ◄──────│
     │ 8. 展示结果 ◄─────┤                      │
     ├──────────────────┤                      │
```

### MAC (Message Authentication Code) 在工具授权中的应用

MAC 用于确保 Agent 工具调用请求的完整性和真实性，防止请求被篡改：

```
Agent ──► 构造工具调用请求
         │
         ├─► 使用共享密钥计算 HMAC(request_body + timestamp + nonce)
         │
         ├─► 发送: {request_body, signature: HMAC, timestamp, nonce}
         │
Tool API ──► 验证:
             │
             ├─► 检查 timestamp 是否在窗口内（防重放）
             ├─► 检查 nonce 是否已使用过（防重放）
             ├─► 用共享密钥重新计算 HMAC 并比对
             └─► 通过则执行，拒绝则返回 403
```

在 Agent 场景中，MAC 通常用于 Agent 与受信任工具之间的直接通信，尤其是在无法使用 OAuth 的本地工具或内部工具场景。

### RBAC (Role-Based Access Control) for Tools

RBAC 通过角色将 Agent 与工具权限关联，是最广泛使用的授权模型：

```
                    ┌──────────────┐
                    │   策略定义    │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
       ┌──────────┐ ┌──────────┐ ┌──────────┐
       │ 角色 A   │ │ 角色 B   │ │ 角色 C   │
       │ 只读     │ │ 读写     │ │ 管理员   │
       └────┬─────┘ └────┬─────┘ └────┬─────┘
            │            │            │
            ▼            ▼            ▼
       ┌──────────┐ ┌──────────┐ ┌──────────┐
       │search    │ │search    │ │search    │
       │read-file │ │read-file │ │read-file │
       │          │ │write-file│ │write-file│
       │          │ │send-mail │ │send-mail │
       │          │ │          │ │delete    │
       │          │ │          │ │admin-api │
       └──────────┘ └──────────┘ └──────────┘

Agent A (只读角色)  ──► 只能调用 search, read-file
Agent B (读写角色)  ──► 能调用 search, read-file, write-file, send-mail
Agent C (管理员角色) ──► 能调用所有工具
```

### ABAC (Attribute-Based Access Control) for Tools

ABAC 基于属性动态判定授权，粒度比 RBAC 更精细：

```
授权判定 = 主体属性 + 环境属性 + 资源属性 + 操作属性

主体属性:  Agent 类型、所属用户、安全等级、会话 ID
环境属性:  当前时间、IP 地址、网络环境、风险等级
资源属性:  工具类型、数据敏感度、资源所属域
操作属性:  操作类型（read/write/delete）、参数值范围

示例: Agent 在办公时间(环境)用只读操作(操作)访问公开数据(资源) → 允许
      Agent 在非办公时间(环境)用删除操作(操作)访问敏感数据(资源) → 拒绝
```

ABAC 的工具授权策略通常用策略语言（如 OPA/Rego）编写：

```rego
# OPA Rego 策略示例：Agent 工具授权
package agent.tools

# 默认拒绝
default allow = false

# 只读工具：搜索、读文件 — 任何认证 Agent 都可以用
allow {
    input.tool in ["search", "read_file"]
    input.agent.authenticated == true
}

# 写工具：写文件 — 需要 Agent 有 write 角色
allow {
    input.tool == "write_file"
    input.agent.roles[_] == "writer"
    input.resource.sensitivity != "high"
}

# 敏感工具：删除、转账 — 需要高风险审批完成
allow {
    input.tool in ["delete_file", "transfer_money"]
    input.agent.roles[_] == "admin"
    input.approval.status == "approved"
    input.approval.expires_at > time.now_ns()
}
```

## 背景 — evolution from API keys to Agent-aware authorization

工具授权经历了三个阶段，每个阶段都反映了 Agent 能力演进带来的新需求：

### 第一阶段：API Key 时代（传统 SaaS 集成）

Agent 集成外部工具时使用固定的 API Key，授权模型简单粗暴：

```
Agent ──► 配置文件: {weather_api_key: "xxx", search_api_key: "yyy"}
       │
       ├─► 所有 Agent 共享同一组 API Key
       ├─► 密钥泄露风险高（硬编码在配置中）
       ├─► 无法区分"哪个 Agent 在调用"
       ├─► 无法限制"Agent 能调用哪些 API"
       └─► 无法撤销单个 Agent 的访问权限
```

**问题**：API Key 本质上是"全有或全无"的授权——拿到 Key 就能调用全部功能，无法实现最小权限。Key 轮换需要更新所有 Agent 配置，运维成本高。

### 第二阶段：Scoped Token 时代（精细化授权）

引入 Token 的作用域（Scope）机制，实现粗粒度的权限分离：

```
                    ┌──────────────────┐
                    │   Auth Server    │
                    │                  │
                    │ scope:           │
                    │  • search:read   │
                    │  • files:read    │
                    │  • files:write   │
                    │  • email:send    │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
      ┌────────────┐ ┌────────────┐ ┌────────────┐
      │ Agent A    │ │ Agent B    │ │ Agent C    │
      │ Token:     │ │ Token:     │ │ Token:     │
      │ search:read│ │ search:read│ │ search:read│
      │ files:read │ │ files:read │ │ files:read │
      │            │ │ files:write│ │ files:write│
      │            │ │            │ │ email:send │
      └────────────┘ └────────────┘ └────────────┘
```

**进步**：Token 作用域实现了"每个 Agent 各自独立授权"、"按需分配最小权限"、"可独立撤销单个 Token"。

### 第三阶段：Agent-aware Authorization（Agent 原生授权）

进入 Agent 原生时代，授权不再仅基于"用户身份"，而是基于完整的 Agent 上下文：

```
传统 API 授权:
  "用户 X 可以调用 API Y"
  → 判定条件：用户身份 + API 路由

Agent 工具授权:
  "Agent 实例 A（代表用户 X）、在当前对话 C 中、
   为完成目标 T、在环境 E 下、是否可以调用工具 F、使用参数 P"
  → 判定条件：Agent身份 + 用户委托 + 对话上下文 + 任务目标 + 环境风险 + 工具参数
```

**Agent-aware 授权的关键特征：**

| 特征 | 说明 | 示例 |
|------|------|------|
| **身份委托链** | 授权决定沿着"用户 → Agent → 工具调用"传递 | 用户授予 Agent 读取邮件的权限，Agent 才能调用 Mail API |
| **上下文感知** | 授权决策考虑对话上下文和任务目标 | 在"帮我准备报销"对话中允许读取发票，但不允许发送邮件 |
| **动态范围** | 权限范围随对话进展动态变化 | 审批通过后临时授权高风险操作 |
| **按需授权** | Agent 在需要时才请求工具权限，而非启动时就全部获取 | Agent 先请求读文件权限，需要时才请求写权限 |
| **会话级生命周期** | Token 存活时间与对话绑定 | 对话结束 Token 自动失效（即使用户忘记登出） |

## 核心矛盾 — security vs usability tradeoff for tool access

工具授权中安全与易用性的矛盾是所有 Agent 系统设计中最棘手的权衡之一：

```
                        安全 vs 易用性
                              │
           ┌──────────────────┴──────────────────┐
           ▼                                     ▼
    ┌──────────────┐                    ┌──────────────────┐
    │  严格授权      │                    │  宽松授权          │
    │              │                    │                  │
    │ ✅ 绝对安全   │                    │ ✅ 用户体验好      │
    │ ✅ 最小权限   │                    │ ✅ Agent 效率高    │
    │ ✅ 攻击面小   │                    │ ✅ 少打断用户流程   │
    │              │                    │                  │
    │ ❌ 频繁审批   │                    │ ❌ 权限滥用风险     │
    │ ❌ Agent 卡顿 │                    │ ❌ 攻击面大        │
    │ ❌ 用户疲惫   │                    │ ❌ 难以追踪        │
    └──────────────┘                    └──────────────────┘
```

### 核心矛盾的具体表现

| 维度 | 安全优先 | 易用性优先 | 理想平衡点 |
|------|---------|-----------|-----------|
| 授权粒度 | 参数级授权（每个参数都要检查） | 工具级授权（一个工具一个权限） | 操作级授权（读/写分离） |
| 审批频率 | 每个工具调用都需确认 | 完全自动授权 | 按风险等级分级审批 |
| Token 有效期 | 5 分钟过期 | 24 小时不过期 | 会话生命周期 + 动态续期 |
| 撤销速度 | 实时撤销（每次调用检查） | 定期检查（每 5 分钟同步） | 事件驱动撤销 + 短 TTL |
| 策略复杂度 | 精细 ABAC 策略（百条以上规则） | 简单 RBAC（3 个角色） | RBAC + 关键操作 ABAC |

### Agent 授权特有的痛点

**1. 高频调用 vs 授权开销**

Agent 在单次任务中可能调用工具数十次，每次调用都做完整的授权检查会导致显著延迟：

```
无授权缓存：每个工具调用都过策略引擎
  Agent: call search   ──► 策略判定 (50ms) ──► 执行 (200ms) = 250ms
  Agent: call read     ──► 策略判定 (50ms) ──► 执行 (150ms) = 200ms
  Agent: call search   ──► 策略判定 (50ms) ──► 执行 (200ms) = 250ms
  Agent: call write    ──► 策略判定 (50ms) ──► 执行 (300ms) = 350ms
  ───────────────────────────────────────────── 总计: 1050ms（其中授权占 200ms）

有授权缓存：首次判定后缓存结果
  Agent: call search   ──► 缓存 MISS ──► 策略判定 (50ms) ──► 执行 (200ms) = 250ms
  Agent: call read     ──► 缓存 MISS ──► 策略判定 (50ms) ──► 执行 (150ms) = 200ms
  Agent: call search   ──► 缓存 HIT (1ms) ──► 执行 (200ms) = 201ms
  Agent: call write    ──► 缓存 MISS ──► 策略判定 (50ms) ──► 执行 (300ms) = 350ms
  ───────────────────────────────────────────── 总计: 1001ms（其中授权占 101ms）
```

高频工具场景下，授权缓存可以降低约 50% 的授权延迟。

**2. 用户打断 vs 授权透明度**

用户不想被频繁打断确认操作，但完全自动授权又让用户失去控制感：

```
"Agent 在后台发了一封邮件？我并没有同意..."
 "Agent 删除了我的文件？什么时候发生的事情？"

 vs

"又要确认？Agent 每做一步都要问我，还不如我自己来做..."
```

平衡方案是按风险分级：

| 风险级别 | 示例 | 授权方式 | 用户感知 |
|---------|------|---------|---------|
| 无风险 | 搜索、读公开数据 | 自动授权 | 用户无感知 |
| 低风险 | 读用户文件 | 自动授权 + 通知 | Agent 告知"正在读取 XXX" |
| 中风险 | 发送邮件、修改文档 | 一次确认管整个会话 | "Agent 需要发邮件权限，是否允许？" |
| 高风险 | 删除、转账 | 每次确认 | "Agent 要执行 XXX，请确认" |
| 极高风险 | 修改系统配置 | 多因素确认 | 二次确认 + 独立通知 |

**3. 授权信息的生命周期管理**

Agent 的运行是"对话式"的，与传统 session 有很大不同：

```
传统 Session:
  登录 ──► 获取 Token ──► Token 有效期内都可用 ──► 登出 ──► Token 失效

Agent 对话:
  对话开始 ──► Agent 获取基础授权 ──► 对话中动态申请权限
    │                                       │
    │  Agent: "我需要读你的日历"             │
    │  用户: "好的"                         │
    │                                       │
    │  Agent: "我需要以你的名义发邮件"        │
    │  用户: "等一下，为什么要发邮件？"        │
    │  Agent: "因为会议冲突需要通知参会者"     │
    │  用户: "好吧，仅这一次"                 │
    │                                       │
    └── 对话结束 ──► 所有权限自动回收 ────────┘
```

关键在于：授权不是"一次性设置"，而是贯穿整个对话的生命周期管理。

## 详细内容

### 1. Tool Authorization Models: scope-based, role-based (RBAC), attribute-based (ABAC), relationship-based (ReBAC)

#### 1.1 Scope-Based（作用域授权）

最基础的授权模型，将工具权限划分为命名空间化的作用域（Scope）：

```
典型 Scope 设计：
  tool:<tool_name>:<action>

示例:
  search:read       — 只读搜索
  files:read        — 读取文件
  files:write       — 写入文件  
  email:send        — 发送邮件
  admin:configure   — 管理配置

Agent Token 携带的 Scope 集合：
  {
    "scopes": ["search:read", "files:read", "email:send"]
  }
```

**优点**：简单直观、易于实现、主流 OAuth 原生支持。
**缺点**：粒度有限（无法表达"只能读特定文件"）、静态绑定、数量多时难以管理。

**适用场景**：工具数量有限（<50个）、权限需求简单的 Agent 系统。

#### 1.2 RBAC (Role-Based Access Control)

通过角色（Role）作为中间层，将 Agent 与权限解耦：

```
             Agent                      角色                      权限
      ┌─────────────────┐        ┌──────────────┐        ┌────────────────┐
      │ Agent: 客服助手   │───────►│ 角色: read-only│───────►│ search:read    │
      │ Agent: 数据分析师 │       │              │        │ files:read     │
      │ Agent: 自动化助理 │       │ 角色: writer  │        │ database:query │
      │                 │       │              │───────►│ files:read     │
      │                 │       │              │        │ files:write    │
      │                 │       │ 角色: admin   │        │ email:send     │
      │                 │       │              │        │ ...            │
      │                 │       │              │        │ files:*        │
      │                 │       │              │        │ admin:*        │
      └─────────────────┘       └──────────────┘        └────────────────┘
```

**RBAC 工具授权的核心数据结构：**

```python
# 角色定义
ROLES = {
    "read_only": {
        "tools": ["search", "read_file", "list_files"],
        "max_params": {"result_limit": 100},
        "deny": ["delete", "write", "admin"]
    },
    "writer": {
        "inherits": ["read_only"],
        "tools": ["write_file", "update_record", "send_notification"],
        "max_params": {"result_limit": 1000}
    },
    "admin": {
        "inherits": ["writer"],
        "tools": ["delete_file", "admin_config", "manage_users"],
        "max_params": {"result_limit": 10000}
    }
}

# Agent 角色绑定
AGENT_ROLES = {
    "agent-support-lite": ["read_only"],
    "agent-support-pro": ["writer"],
    "agent-devops": ["admin"]
}
```

**优点**：管理简单、符合组织直觉、权限变更只需改角色。
**缺点**：角色爆炸（细粒度时需要非常多角色）、静态绑定、无法表达上下文相关规则。

**适用场景**：角色划分清晰的团队/企业 Agent 平台。

#### 1.3 ABAC (Attribute-Based Access Control)

基于属性的动态授权模型，是 Agent 工具授权中最灵活的方案：

```
授权决策 = f(主体属性, 资源属性, 环境属性, 操作属性)

主体属性:
  - agent_id: "agent-sales-01"
  - agent_type: "sales_assistant"
  - user_role: "sales_rep"
  - security_clearance: "L2"

资源属性:
  - tool_name: "read_file"
  - resource_type: "document"
  - sensitivity: "internal"
  - owner: "sales_department"

环境属性:
  - time: "2026-07-17T14:30:00Z"
  - network: "corporate_vpn"
  - risk_score: 0.2
  - session_age: "45m"

操作属性:
  - action: "read"
  - params: {path: "/shared/sales/report.pdf", limit: 10}
```

**ABAC 策略引擎判定流程：**

```
  工具调用请求
      │
      ▼
  ┌─────────────────────────────────────┐
  │ 1. 属性收集                          │
  │    - 从请求中提取主体、资源、操作属性  │
  │    - 从上下文中提取环境属性           │
  │    - 从外部系统补全缺失属性           │
  └──────────────┬──────────────────────┘
                 ▼
  ┌─────────────────────────────────────┐
  │ 2. 策略匹配                          │
  │    - 加载所有授权策略                 │
  │    - 过滤出匹配当前请求的策略          │
  │    - 策略优先级排序                   │
  └──────────────┬──────────────────────┘
                 ▼
  ┌─────────────────────────────────────┐
  │ 3. 策略评估                          │
  │    - 逐条评估匹配的策略               │
  │    - 支持 allow/deny/conditional     │
  │    - Deny 优先（显式拒绝覆盖允许）     │
  └──────────────┬──────────────────────┘
                 ▼
  ┌─────────────────────────────────────┐
  │ 4. 决策输出                          │
  │    - Allow / Deny / 需要审批          │
  │    - 附带授权约束（如速率限制）        │
  │    - 决策日志（用于审计）              │
  └─────────────────────────────────────┘
```

**优点**：极灵活、细粒度、上下文感知、支持动态规则。
**缺点**：策略复杂难维护、性能开销大、调试困难。

**适用场景**：复杂企业环境、多租户平台、安全要求极高的 Agent 系统。

#### 1.4 ReBAC (Relationship-Based Access Control)

基于关系的授权模型，以图结构表示"用户/Agent/资源"之间的关系，在 Agent 多租户场景特别有用：

```
                    ┌────────────┐
                    │  组织: Acme │
                    └─────┬──────┘
                          │ member_of
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ 部门: R&D │   │ 部门: Sales│  │ 部门: HR  │
    └─────┬────┘   └─────┬────┘   └─────┬────┘
          │               │               │
          ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ Agent:   │   │ Agent:   │   │ Agent:   │
    │ 研发助手  │   │ 销售助手  │   │ HR 助手   │
    └────┬─────┘   └────┬─────┘   └────┬─────┘
         │               │               │
         │ can_access    │ can_access    │ can_access
         ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ Code Repo │   │ CRM      │   │ HR System│
    │ (R&D)    │   │ (Sales)  │   │ (HR)     │
    └──────────┘   └──────────┘   └──────────┘
```

**ReBAC 关系定义示例（Google Zanzibar 风格）：**

```
# 关系元组 (tuple): <namespace/object#relation@user>
# 格式: <资源类型/资源ID#关系@主体>

# Agent 与组织的关系
namespace/org-acme#member@user/alice

# Agent 与部门的关系
namespace/dept-rd#manager@user/alice
namespace/dept-rd#member@agent/rd-assistant

# 工具访问关系
namespace/tool-code-repo#access@agent/rd-assistant
namespace/tool-crm#access@agent/sales-assistant
```

**ReBAC 授权查询：**

```
查询: "Agent rd-assistant 可以访问 code_repo 吗？"

图遍历:
  1. 找到 agent/rd-assistant
  2. 找到 rd-assistant 的部门 → dept-rd
  3. 找到 dept-rd 可访问的资源 → tool/code-repo
  4. 检查 relationship: tool/code-repo#access@agent/rd-assistant
  5. 关系存在 → 授权通过
  
查询: "Agent rd-assistant 可以访问 CRM 吗？"
  
图遍历:
  1. 找到 agent/rd-assistant
  2. 找到 rd-assistant 的部门 → dept-rd
  3. 找到 dept-rd 可访问的资源 → tool/code-repo
  4. CRM 不在可访问列表中
  5. 关系不存在 → 授权拒绝
```

**优点**：天然适合多租户、关系可继承、支持大规模分布式授权（Google Zanzibar 架构）。
**缺点**：实现复杂、图遍历有性能开销、不适合简单的工具授权场景。

**适用场景**：多租户 Agent 平台、企业级 Agent 协作系统、细粒度的资源级授权。

---

**四种模型的关键对比：**

| 维度 | Scope | RBAC | ABAC | ReBAC |
|------|-------|------|------|-------|
| 粒度 | 粗 | 中 | 细 | 细 |
| 灵活性 | 低 | 中 | 高 | 高 |
| 复杂度 | 低 | 中 | 高 | 高 |
| 上下文感知 | 否 | 否 | 是 | 部分 |
| 运维成本 | 低 | 中 | 高 | 高 |
| 适用 Agent 规模 | 小 | 中 | 大 | 大 |
| 主流实现 | OAuth Scopes | Keycloak, Okta | OPA, AWS Cedar | Google Zanzibar, Auth0 FGA |

---

### 2. API Key Management for Tools: per-tool keys, scoped keys, key rotation, vault integration

#### 2.1 Per-Tool Keys（每个工具独立密钥）

每个工具使用独立的 API Key，实现工具级别的隔离：

```
配置文件（不推荐硬编码）:
  tools:
    search:
      api_key: "sk-search-xxxxx"
      base_url: "https://api.search.com/v1"
    weather:
      api_key: "sk-weather-yyyyy"
      base_url: "https://api.weather.com/v2"
    email:
      api_key: "sk-email-zzzzz"
      base_url: "https://api.email.com/v3"

优点:
  - 某个工具密钥泄露不影响其他工具
  - 可单独轮换某个工具的密钥
  - 可独立审计每个工具的调用
  
缺点:
  - 密钥管理成本高（Agent 集成 10 个工具就要管理 10 个密钥）
  - 密钥分发复杂（每个 Agent 实例都要获取所有关联工具的密钥）
```

#### 2.2 Scoped Keys（作用域密钥）

用一个密钥承载多个作用域，一个密钥管理所有工具的权限：

```
Agent Key (MASTER):
  Key: "sk-agent-abc123"
  Scopes: ["search:read", "files:read", "weather:read"]
  
Agent Key 不直接调用工具，而是用它换取每个工具的 Scoped Token：

  Agent Key ──► Auth Server ──► 验证 Key + Scopes
       │                            │
       │                            ▼
       │                    ┌────────────────┐
       │                    │ 颁发 Scoped     │
       │                    │ Tokens:         │
       │                    │                 │
       │                    │ search:  Token_A│
       │                    │ files:   Token_B│
       │                    │ weather: Token_C│
       │                    └────────┬───────┘
       │                             │
       ▼                             ▼
  Agent 用 Token_A 调用 search API
  Agent 用 Token_B 调用 files  API
  Agent 用 Token_C 调用 weather API
```

**优点**：集中管理、支持细粒度撤销、一次认证多次使用。
**缺点**：引入了 Auth Server 依赖，增加了系统复杂度。

#### 2.3 Key Rotation（密钥轮换）

密钥轮换是保护 API Key 安全的必要机制：

```
轮换策略:
  ┌─────────────────────────────────────────┐
  │ 主动轮换                                  │
  │  ├─ 定时轮换：每 90 天自动生成新密钥       │
  │  ├─ 事件驱动：检测到泄露、员工离职时立即轮换 │
  │  └─ 版本化：新旧密钥共存直到所有客户端迁移   │
  ├─────────────────────────────────────────┤
  │ 灰度过渡                                  │
  │  ├─ 阶段 1：生成新密钥但旧密钥仍有效        │
  │  ├─ 阶段 2：旧密钥标记为 "即将过期"         │
  │  ├─ 阶段 3：旧密钥过期，仅新密钥有效        │
  │  └─ 阶段 4：清除旧密钥                     │
  └─────────────────────────────────────────┘

轮换流程:
  1. Auth Server 生成新密钥对 (new_key_id, new_key_secret)
  2. 存储新密钥到 Vault，标记为 "active"
  3. 通知 Agent 运行时更新密钥（通过 API 或配置中心）
  4. 旧密钥保留在 "grace_period" 状态（通常 24-72 小时）
  5. Agent 在 grace_period 内迁移到新密钥
  6. 超时后旧密钥标记为 "revoked"，不再接受
```

#### 2.4 Vault Integration（密钥保险库集成）

生产环境必须使用密钥管理服务（Vault）而非配置文件管理 API Key：

```
  ┌──────────┐         ┌──────────┐         ┌──────────┐
  │  Agent   │         │  Vault   │         │  Tool    │
  │ Runtime  │         │ (密钥管理)│         │ Service  │
  └────┬─────┘         └────┬─────┘         └────┬─────┘
       │                    │                    │
       │ 1. 启动时请求工具密钥                    │
       ├───────────────────►│                    │
       │                    │                    │
       │ 2. Vault 验证身份  │                    │
       │    • Agent ID      │                    │
       │    • JWT Token     │                    │
       │    • 环境信息       │                    │
       │                    │                    │
       │ 3. 返回临时密钥     │                    │
       │    • 密钥: sk-xxx  │                    │
       │    • TTL: 1h      │                    │
       ◄───────────────────┤                    │
       │                    │                    │
       │ 4. 使用密钥调用工具 │                    │
       ├────────────────────────────────────────►│
       │                    │                    │
       │ 5. TTL 到期前续期   │                    │
       ├───────────────────►│                    │
       │                    │                    │
       │ 6. 对话结束撤销密钥  │                    │
       ├───────────────────►│                    │
       │                    │ 7. 密钥立即失效      │
       │                    ├───────────────────►│
       │                    │                    │
       │ 8. 下次调用被拒绝   │                    │
       ├────────────────────────────────────────►│
       │                    │        403 Forbidden│
       ◄────────────────────────────────────────┤
```

**Vault 集成最佳实践：**

```python
import hvac
from typing import Optional

class VaultKeyManager:
    """Vault 集成的密钥管理器"""
    
    def __init__(self, vault_addr: str, vault_token: str):
        self.client = hvac.Client(url=vault_addr, token=vault_token)
    
    async def get_tool_key(self, agent_id: str, tool_name: str) -> Optional[KeySecret]:
        """获取指定工具的临时密钥"""
        path = f"agent-tools/{agent_id}/{tool_name}"
        
        try:
            secret = self.client.secrets.kv.v2.read_secret_version(
                path=path, mount_point="agent-keys"
            )
            data = secret["data"]["data"]
            
            return KeySecret(
                key_id=data["key_id"],
                secret=data["secret"],
                expires_at=data["expires_at"],
                rotate_at=data.get("rotate_at")
            )
        except Exception:
            return None  # 密钥不存在或权限不足
    
    async def rotate_key(self, tool_name: str, agent_id: str) -> KeySecret:
        """轮换指定工具的密钥"""
        new_key = self._generate_key(tool_name, agent_id)
        
        # 写入新密钥（保留旧版本用于灰度过渡）
        self.client.secrets.kv.v2.create_or_update_secret(
            path=f"agent-tools/{agent_id}/{tool_name}",
            mount_point="agent-keys",
            secret={
                "key_id": new_key.key_id,
                "secret": new_key.secret,
                "expires_at": new_key.expires_at,
                "previous_secret": new_key.previous_secret,  # 旧密钥（过渡期）
                "previous_expires_at": new_key.previous_expires_at
            }
        )
        
        return new_key
```

---

### 3. OAuth 2.0 & OIDC for Agent Tool Access: authorization code flow with PKCE for Agents, device flow for headless Agents

#### 3.1 OAuth 2.0 Authorization Code Flow + PKCE for Agent

标准的 OAuth 2.0 授权码流程，加上 PKCE（Proof Key for Code Exchange）增强安全性，适用于有用户界面的 Agent 场景：

```
  用户 (资源所有者)        Agent (客户端)           Auth Server (授权服务器)         Tool API (资源服务器)
       │                      │                          │                          │
       │ 1. 发起工具授权请求   │                          │                          │
       │◄─────────────────────┤                          │                          │
       │                      │                          │                          │
       │ 2. 重定向到授权页面   │                          │                          │
       │◄─────────────────────┤                          │                          │
       │                      │                          │                          │
       │ 3. 用户登录并授权     │                          │                          │
       ├─────────────────────────────────────────────────►                          │
       │ (scope: search:read  │                          │                          │
       │  files:write         │                          │                          │
       │  email:send)         │                          │                          │
       │                      │                          │                          │
       │ 4. 授权码 (code)     │                          │                          │
       │◄─────────────────────────────────────────────────┤                          │
       │                      │                          │                          │
       │ 5. 授权码传递给 Agent │                          │                          │
       ├─────────────────────►│                          │                          │
       │                      │                          │                          │
       │                      │ 6. code + code_verifier  │                          │
       │                      │    (PKCE)               │                          │
       │                      ├─────────────────────────►│                          │
       │                      │                          │                          │
       │                      │ 7. 验证 code_verifier    │                          │
       │                      │    匹配 code_challenge   │                          │
       │                      │                          │                          │
       │                      │ 8. 返回 Access Token     │                          │
       │                      │    + Refresh Token       │                          │
       │                      │◄─────────────────────────┤                          │
       │                      │                          │                          │
       │                      │ 9. 用 Access Token        │                          │
       │                      │    调用工具               │                          │
       │                      ├────────────────────────────────────────────────────►│
       │                      │                          │                          │
       │                      │ 10. 验证 Token           │                          │
       │                      │     • 签名                │                          │
       │                      │     • Scope              │                          │
       │                      │     • Expiry             │                          │
       │                      │◄────────────────────────────────────────────────────┤
       │                      │                          │                          │
       │  11. 工具结果        │                          │                          │
       │◄─────────────────────┤                          │                          │
```

**PKCE 在 Agent 场景中的重要性：**

PKCE 防止授权码被拦截后的重放攻击。Agent 通常在浏览器（前端）中运行或与浏览器交互，授权码可能通过 URL 重定向传递，存在被拦截的风险：

```python
import secrets
import hashlib
import base64

class PKCEGenerator:
    """Agent 的 PKCE 验证器生成"""
    
    @staticmethod
    def generate_code_verifier() -> str:
        """生成 43-128 字符的随机验证器"""
        token = secrets.token_urlsafe(64)
        return token[:128]  # 最大 128 字符
    
    @staticmethod
    def generate_code_challenge(verifier: str) -> str:
        """从验证器生成 challenge（S256 方法）"""
        sha256 = hashlib.sha256(verifier.encode('ascii')).digest()
        return base64.urlsafe_b64encode(sha256).rstrip(b'=').decode('ascii')
    
    @staticmethod
    def create_pkce_pair() -> dict:
        """生成一对 PKCE 值"""
        verifier = PKCEGenerator.generate_code_verifier()
        challenge = PKCEGenerator.generate_code_challenge(verifier)
        return {
            "code_verifier": verifier,
            "code_challenge": challenge,
            "code_challenge_method": "S256"
        }
```

#### 3.2 OAuth 2.0 Device Flow for Headless Agents

对于无用户界面的 Agent（后台自动化 Agent、CLI Agent、定时任务 Agent），使用设备授权流程：

```
  用户（在浏览器）        无头 Agent（控制台）        Auth Server          Tool API
       │                      │                        │                    │
       │                      │ 1. 请求设备授权         │                    │
       │                      ├───────────────────────►│                    │
       │                      │                        │                    │
       │                      │ 2. 返回:               │                    │
       │                      │    device_code         │                    │
       │                      │    user_code: "AB-123" │                    │
       │                      │    verification_uri    │                    │
       │                      │    interval: 5s        │                    │
       │                      │◄───────────────────────┤                    │
       │                      │                        │                    │
       │ 3. 访问验证 URL      │                        │                    │
       │    (用户手动操作)      │                        │                    │
       │◄─────────────────────┤                        │                    │
       │                      │                        │                    │
       │ 4. 输入 code:"AB-123"│                        │                    │
       │                      │                        │                    │
       │ 5. 确认授权           │                        │                    │
       ├───────────────────────────────────────────────►│                    │
       │                      │                        │                    │
       │                      │ 6. 轮询 token 端点     │                    │
       │                      │    (每 5s)             │                    │
       │                      ├───────────────────────►│                    │
       │                      │                        │                    │
       │                      │ 7. 授权完成，返回       │                    │
       │                      │    Access Token        │                    │
       │                      │◄───────────────────────┤                    │
       │                      │                        │                    │
       │                      │ 8. 用 Token 调用工具    │                    │
       │                      ├───────────────────────────────────────────►│
```

**Device Flow 对 Agent 的意义：** 这是目前最实用的"用户 → Agent → 工具"授权委托方案。用户在自己的设备上完成 OAuth 授权，Agent（可能在云端运行）获得委托的 Token，代表用户调用工具。

#### 3.3 OIDC (OpenID Connect) for Agent Identity

OIDC 在 OAuth 2.0 之上增加身份层，让 Agent 不仅能获得工具访问权限，还能获取经过验证的用户身份信息：

```
Agent 获取的 ID Token (JWT 格式):
{
  "iss": "https://auth.mycompany.com",
  "sub": "user_abc_123",           // 用户唯一标识
  "aud": "agent-platform",
  "exp": 1777777777,
  "iat": 1777777177,
  "nonce": "random_nonce_value",
  
  // Agent 扩展声明
  "agent": {
    "id": "agent-sales-01",
    "type": "sales_assistant",
    "session_id": "sess_xyz_789"
  },
  
  // 工具授权信息（非标准，但实际常用）
  "tool_permissions": {
    "search:read": {"max_queries": 1000},
    "files:read": {"allowed_paths": ["/shared/sales/"]},
    "email:send": {"rate_limit": "10/hour"}
  }
}
```

**OIDC 在 Agent 工具授权中的关键作用：**

| 能力 | 说明 | 作用 |
|------|------|------|
| 身份验证 | 验证用户身份 | Agent 知道"谁"在发出指令 |
| 身份委托 | 用户授权 Agent 代表自己 | Tool API 看到的是用户身份而非 Agent |
| 单点登录 (SSO) | 用户一次登录，Agent 获得所有已授权工具的访问 | 减少授权操作次数 |
| 会话管理 | Token 刷新、过期、撤销 | Agent 在整个对话期间保持授权状态 |

---

### 4. MCP (Model Context Protocol) Authorization: MCP security model, capability negotiation, origin verification

#### 4.1 MCP Security Model

MCP（Model Context Protocol）是 Anthropic 提出的 Agent-工具通信协议标准，定义了工具授权的内置安全机制：

```
┌─────────────────────────────────────────────────────────┐
│                   MCP 安全架构                           │
│                                                         │
│  Host Application (宿主应用)                              │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Capability Negotiation (能力协商)                │    │
│  │  • 客户端声明需要的工具能力                        │    │
│  │  • 服务端声明提供的工具能力                        │    │
│  │  • 双方在初始化阶段确定授权的工具范围               │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                               │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Origin Verification (来源验证)                   │    │
│  │  • 验证客户端/服务端的来源身份                    │    │
│  │  • 检查通信通道的安全性（TLS/mTLS）               │    │
│  │  • 确认连接未被中间人劫持                         │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                               │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Permission Boundaries (权限边界)                 │    │
│  │  • 工具级权限：哪些工具可用                        │    │
│  │  • 参数级权限：工具参数的限制范围                   │    │
│  │  • 数据级权限：返回数据的过滤和脱敏                 │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

#### 4.2 Capability Negotiation（能力协商）

MCP 的能力协商在连接初始化阶段完成，是工具授权的第一道关卡：

```
Client (Agent)                          Server (Tool Provider)
      │                                        │
      │ 1. 初始化请求                           │
      │    {                                    │
      │      "capabilities": {                  │
      │        "tools": ["search", "read_file"],│
      │        "resources": ["docs", "images"], │
      │        "rate_limit": 100               │
      │      },                                 │
      │      "client_info": {                   │
      │        "name": "my-agent",              │
      │        "version": "1.0.0"               │
      │      }                                   │
      │    }                                    │
      ├────────────────────────────────────────►│
      │                                        │
      │                        2. 验证能力      │
      │                           • 检查请求的   │
      │                             工具是否在   │
      │                             允许列表     │
      │                           • 检查客户端   │
      │                             身份是否有效  │
      │                           • 检查速率限制  │
      │                           • 检查资源权限  │
      │                                        │
      │ 3. 初始化响应                           │
      │    {                                    │
      │      "capabilities": {                  │
      │        "tools": ["search", "read_file"],│
      │        "resources": ["docs"],           │
      │        "rate_limit": 50,                │
      │        "auth_required": [               │
      │          "email_send"                   │
      │        ]                                │
      │      },                                 │
      │      "server_info": {                   │
      │        "name": "tool-server",           │
      │        "version": "2.0.0"               │
      │      }                                   │
      │    }                                    │
      │◄────────────────────────────────────────┤
      │                                        │
      │ 4. Agent 使用已授权的能力                │
      │    • search → 允许                      │
      │    • read_file → 允许                   │
      │    • email_send → 需要额外授权           │
      │    • images → 服务端拒绝 (能力不足)       │
```

**协商的关键作用：** 这是一种"原则上的最小权限"——Agent 在初始化时只需要声明需要的工具，服务端只授予双方同意的权限。不同于传统 API 的"先开所有权限再逐渐限制"，MCP 的模型是"先关所有权限再按需开放"。

#### 4.3 Origin Verification（来源验证）

MCP 要求连接双方验证对方的来源身份：

```python
class MCPOriginVerifier:
    """MCP 来源验证"""
    
    async def verify_origin(self, connection: MCPConnection) -> VerificationResult:
        """验证 MCP 连接的来源"""
        # 1. TLS 证书验证
        tls_ok = await self._verify_tls_cert(connection.cert)
        if not tls_ok:
            return VerificationResult.FAIL("TLS 证书无效")
        
        # 2. 客户端身份验证
        client_verified = await self._verify_client_identity(
            client_id=connection.client_id,
            client_token=connection.token,
            expected_origin=connection.expected_origin
        )
        if not client_verified:
            return VerificationResult.FAIL("客户端身份验证失败")
        
        # 3. mTLS 双向验证（可选，用于高安全场景）
        if connection.require_mtls:
            mtls_ok = await self._verify_mtls(connection.client_cert)
            if not mtls_ok:
                return VerificationResult.FAIL("mTLS 验证失败")
        
        return VerificationResult.PASS()
    
    async def _verify_client_identity(self, client_id: str, 
                                       client_token: str,
                                       expected_origin: str) -> bool:
        """验证客户端身份是否匹配声明的来源"""
        # 解码 Token 获取声明
        claims = decode_jwt(client_token)
        
        # 验证 client_id 匹配
        if claims.get("sub") != client_id:
            return False
        
        # 验证 origin 匹配
        if claims.get("origin") != expected_origin:
            return False
        
        # 验证签名
        try:
            verify_jwt_signature(client_token)
            return True
        except:
            return False
```

#### 4.4 MCP Tool Authorization Flow (完整流程)

```
  Agent (MCP Client)               MCP Server                  Tool API (后端)
        │                              │                           │
        │ 1. 初始化连接                 │                           │
        │    (能力协商: search, read)   │                           │
        ├─────────────────────────────►│                           │
        │                              │                           │
        │ 2. 验证 Agent 身份            │                           │
        │    • client_id              │                           │
        │    • JWT token              │                           │
        │    • origin                 │                           │
        │                              │                           │
        │ 3. 返回已授权工具列表          │                           │
        │◄─────────────────────────────┤                           │
        │                              │                           │
        │ 4. 调用 search 工具          │                           │
        │    (tool_call + token)       │                           │
        ├─────────────────────────────►│                           │
        │                              │                           │
        │                    5. 本地授权检查                          │
        │                       • Agent 有没有 search 权限？         │
        │                       • 本次搜索参数是否合法？              │
        │                       • 速率限制是否已达上限？              │
        │                              │                           │
        │                              │ 6. 转发请求 + 服务端 Token  │
        │                              ├──────────────────────────►│
        │                              │                           │
        │                              │         7. 执行搜索并返回  │
        │                              │◄──────────────────────────┤
        │                              │                           │
        │ 8. 返回搜索结果               │                           │
        │◄─────────────────────────────┤                           │
```

---

### 5. Tool-Level Parameter Authorization: parameter-level restrictions, allowed value ranges, dynamic field filtering

#### 5.1 Parameter-Level Restrictions（参数级限制）

在工具级授权之上，进一步对参数进行精细化控制——这是 Agent 安全中最容易被忽视但极其重要的一环：

```
工具调用请求:
  tool: "send_email"
  params:
    to: "user@example.com"
    subject: "Hello"
    body: "Message content"
    attachments: ["/path/to/file.pdf"]
    priority: "high"

参数级授权检查:
  ┌─────────────────────────────────────┐
  │ 参数         │ 限制                  │  ──  通过/拒绝
  ├─────────────────────────────────────┤
  │ to           │ 必须在 allowed_domains │  ✅  example.com 允许
  │ subject      │ 不允许为空            │  ✅  "Hello" 非空
  │ body         │ 最大 10000 字符       │  ✅  15 字符 < 10000
  │ attachments  │ 不允许附件             │  ❌  拒绝！
  │ priority     │ 必须在 [low, normal]  │  ❌  "high" 不允许
  └─────────────────────────────────────┘
```

#### 5.2 Allowed Value Ranges（合法值范围）

为参数定义合法值的范围、模式、枚举：

```python
from enum import Enum
from typing import Any, Dict, List, Optional
import re

class ParameterRule:
    """参数规则定义"""
    
    def __init__(self, 
                 param_name: str,
                 param_type: type,
                 required: bool = False,
                 allowed_values: Optional[List[Any]] = None,
                 min_value: Optional[float] = None,
                 max_value: Optional[float] = None,
                 pattern: Optional[str] = None,
                 max_length: Optional[int] = None):
        self.param_name = param_name
        self.param_type = param_type
        self.required = required
        self.allowed_values = allowed_values
        self.min_value = min_value
        self.max_value = max_value
        self.pattern = pattern
        self.max_length = max_length
    
    def validate(self, value: Any) -> tuple[bool, str]:
        """验证参数值是否合法"""
        # 检查必填
        if self.required and value is None:
            return False, f"参数 {self.param_name} 是必填的"
        
        if value is None:
            return True, ""  # 非必填且为 None，跳过
        
        # 检查类型
        if not isinstance(value, self.param_type):
            return False, f"参数 {self.param_name} 类型错误: 期望 {self.param_type}, 实际 {type(value)}"
        
        # 检查枚举值
        if self.allowed_values and value not in self.allowed_values:
            return False, f"参数 {self.param_name} 值 '{value}' 不在允许列表中: {self.allowed_values}"
        
        # 检查数值范围
        if isinstance(value, (int, float)):
            if self.min_value is not None and value < self.min_value:
                return False, f"参数 {self.param_name} = {value} 小于最小值 {self.min_value}"
            if self.max_value is not None and value > self.max_value:
                return False, f"参数 {self.param_name} = {value} 大于最大值 {self.max_value}"
        
        # 检查正则模式
        if self.pattern and isinstance(value, str):
            if not re.match(self.pattern, value):
                return False, f"参数 {self.param_name} 不匹配模式 '{self.pattern}'"
        
        # 检查字符串长度
        if self.max_length and isinstance(value, str):
            if len(value) > self.max_length:
                return False, f"参数 {self.param_name} 长度 {len(value)} 超过最大长度 {self.max_length}"
        
        return True, ""


# 工具参数规则定义
TOOL_PARAM_RULES = {
    "search": {
        "query": ParameterRule("query", str, required=True, max_length=500),
        "max_results": ParameterRule("max_results", int, min_value=1, max_value=100),
        "source": ParameterRule("source", str, allowed_values=["web", "news", "images"])
    },
    "send_email": {
        "to": ParameterRule("to", str, required=True, pattern=r'^[\w\.-]+@[\w\.-]+\.\w+$'),
        "subject": ParameterRule("subject", str, max_length=200),
        "body": ParameterRule("body", str, max_length=50000),
        "priority": ParameterRule("priority", str, allowed_values=["low", "normal"]),
        "attachments": None  # 不允许附件参数
    },
    "read_file": {
        "path": ParameterRule("path", str, required=True, pattern=r'^/shared/[a-zA-Z0-9_/\.-]+$'),
        "encoding": ParameterRule("encoding", str, allowed_values=["utf-8", "ascii"]),
        "max_size": ParameterRule("max_size", int, min_value=1, max_value=10*1024*1024)  # 10MB
    }
}
```

#### 5.3 Dynamic Field Filtering（动态字段过滤）

根据 Agent 的权限级别，动态过滤工具返回数据的字段：

```
场景：Agent 查询用户信息

全部字段（管理员 Agent）:
  {
    "id": "user_001",
    "name": "张三",
    "email": "zhangsan@example.com",
    "phone": "138-0000-0000",
    "salary": 50000,
    "ssn": "123-45-6789",
    "department": "Engineering",
    "performance_reviews": [...]
  }

过滤后（普通客服 Agent）:
  {
    "id": "user_001",
    "name": "张三",
    "email": "zhangsan@example.com",
    "department": "Engineering"
    // phone, salary, ssn, performance_reviews 被过滤
  }
```

**实现方式（响应拦截 + 字段级策略）：**

```python
class FieldLevelFilter:
    """字段级过滤 — Agent 只能看到被授权的字段"""
    
    FIELD_POLICIES = {
        "agent-sales": {
            "user_info": ["id", "name", "email", "department"],
            "order_info": ["order_id", "amount", "status", "date"]
        },
        "agent-hr": {
            "user_info": ["id", "name", "email", "phone", "salary", "department"],
            "employee_record": ["*"]  # 所有字段
        },
        "agent-support": {
            "user_info": ["id", "name", "email"],
            "order_info": ["order_id", "status"]
        }
    }
    
    def filter_response(self, 
                        agent_type: str, 
                        resource_type: str, 
                        response_data: dict) -> dict:
        """根据 Agent 权限过滤响应数据"""
        allowed_fields = self.FIELD_POLICIES.get(agent_type, {}).get(resource_type, [])
        
        if not allowed_fields:
            return {}  # 无权限访问此资源
        
        if "*" in allowed_fields:
            return response_data  # 通配符 = 所有字段
        
        return {
            key: response_data[key]
            for key in allowed_fields
            if key in response_data
        }
```

**数据脱敏（在字段过滤基础上）：**

```python
class DataMaskingFilter:
    """在字段过滤基础上，对敏感数据进行脱敏"""
    
    MASK_RULES = {
        "phone": lambda v: v[:3] + "****" + v[-4:] if len(v) >= 7 else "****",
        "email": lambda v: v[:2] + "***@" + v.split("@")[1],
        "ssn": lambda v: "***-**-" + v[-4:],
        "credit_card": lambda v: "****-****-****-" + v[-4:],
    }
    
    def apply_masking(self, agent_type: str, data: dict, sensitivity: str) -> dict:
        """对响应数据应用脱敏"""
        # 低安全级别的 Agent 看到的敏感数据需要脱敏
        if agent_type in ["agent-support", "agent-sales"] and sensitivity == "high":
            masked = {}
            for key, value in data.items():
                if key in self.MASK_RULES:
                    masked[key] = self.MASK_RULES[key](value)
                else:
                    masked[key] = value
            return masked
        return data
```

---

### 6. Authorization Caching: reducing auth latency, revocation handling, short TTL for sensitive tools

#### 6.1 Authorization Cache Architecture

授权缓存是提高 Agent 工具调用性能的关键手段，但必须与安全要求平衡：

```
  工具调用请求
      │
      ▼
  ┌─────────────────────────────────┐
  │  1. 检查授权缓存                  │
  │     Key: agent_id + tool + params │
  │                                  │
  │  ┌──────────┐    ┌──────────┐   │
  │  │ Cache HIT│    │Cache MISS│   │
  │  └────┬─────┘    └────┬─────┘   │
  │       │               │          │
  │       ▼               ▼          │
  │  ┌──────────┐  ┌──────────────┐  │
  │  │ 返回缓存  │  │ 2. 策略引擎    │  │
  │  │ 授权结果  │  │    判定       │  │
  │  └──────────┘  └──────┬───────┘  │
  │                       │          │
  │                       ▼          │
  │                  ┌──────────┐    │
  │                  │ 3. 写入   │    │
  │                  │ 缓存 +    │    │
  │                  │ TTL      │    │
  │                  └──────────┘    │
  └─────────────────────────────────┘
```

#### 6.2 Cache Strategy by Tool Sensitivity

不同安全级别的工具采用不同的缓存策略：

```python
class AuthorizationCache:
    """分级授权缓存"""
    
    # 缓存配置：工具 → 缓存策略
    CACHE_CONFIG = {
        # 只读、低风险工具：长 TTL
        "search": {
            "ttl": 300,           # 5 分钟
            "negative_ttl": 60,   # 拒绝结果也缓存 1 分钟（防止重复查询）
            "max_entries": 1000,
            "invalidate_on": []   # 不需要主动失效
        },
        "read_file": {
            "ttl": 300,
            "negative_ttl": 60,
            "max_entries": 500,
            "invalidate_on": []
        },
        
        # 写操作、中风险工具：短 TTL
        "write_file": {
            "ttl": 60,            # 1 分钟
            "negative_ttl": 30,
            "max_entries": 200,
            "invalidate_on": ["permission_changed"]
        },
        "send_email": {
            "ttl": 30,            # 30 秒
            "negative_ttl": 15,
            "max_entries": 100,
            "invalidate_on": ["permission_changed", "user_revoked"]
        },
        
        # 高风险工具：几乎不缓存
        "delete_file": {
            "ttl": 5,             # 5 秒（几乎每次都重新判定）
            "negative_ttl": 5,
            "max_entries": 50,
            "invalidate_on": ["permission_changed", "user_revoked", "risk_score_changed"]
        },
        "transfer_money": {
            "ttl": 0,             # 不缓存（每次都要重新授权）
            "negative_ttl": 0,
            "max_entries": 10,
            "invalidate_on": ["*"]  # 任何事件都失效
        }
    }
    
    def __init__(self):
        self.cache = {}  # key → (result, expiry, negative)
    
    async def get_or_evaluate(self, 
                              auth_request: AuthRequest,
                              evaluator: Callable) -> AuthResult:
        """获取缓存的授权结果，或重新评估"""
        cache_key = self._make_cache_key(auth_request)
        config = self.CACHE_CONFIG.get(auth_request.tool, {})
        
        # 检查缓存
        if cache_key in self.cache:
            result, expiry = self.cache[cache_key]
            if time.time() < expiry:
                return result
            else:
                del self.cache[cache_key]
        
        # 重新评估
        result = await evaluator(auth_request)
        
        # 写入缓存
        if config.get("ttl", 0) > 0 and result.allowed:
            self.cache[cache_key] = (result, time.time() + config["ttl"])
        elif config.get("negative_ttl", 0) > 0 and not result.allowed:
            self.cache[cache_key] = (result, time.time() + config["negative_ttl"])
        
        return result
    
    def invalidate(self, event: CacheInvalidationEvent):
        """根据事件失效缓存"""
        for key, (result, expiry) in list(self.cache.items()):
            # 解析 cache_key 获取 tool_name
            tool_name = key.split(":")[0]
            config = self.CACHE_CONFIG.get(tool_name, {})
            
            invalidate_on = config.get("invalidate_on", [])
            if "*" in invalidate_on or event.type in invalidate_on:
                del self.cache[key]
                logger.info(f"授权缓存失效: {key}, 原因: {event.type}")
```

#### 6.3 Revocation Handling（撤销处理）

```python
class RevocationManager:
    """授权撤销管理器"""
    
    def __init__(self, cache: AuthorizationCache):
        self.cache = cache
        self.revocation_list = {}   # token_jti → revoked_at
        self.revocation_ttl = 3600  # 撤销记录保留 1 小时
    
    async def revoke_agent_access(self, agent_id: str, reason: str):
        """撤销 Agent 的所有工具访问权限"""
        event = CacheInvalidationEvent(
            type="agent_revoked",
            agent_id=agent_id,
            reason=reason
        )
        
        # 1. 失效所有相关缓存
        self.cache.invalidate_for_agent(agent_id)
        
        # 2. 记录撤销事件（用于审计）
        await self._log_revocation(agent_id, reason)
        
        # 3. 通知活跃 session 立即断开
        await self._notify_active_sessions(agent_id, reason)
    
    async def revoke_tool_access(self, agent_id: str, tool_name: str, reason: str):
        """撤销 Agent 对特定工具的访问"""
        event = CacheInvalidationEvent(
            type="tool_revoked",
            agent_id=agent_id,
            tool_name=tool_name,
            reason=reason
        )
        
        self.cache.invalidate_for_tool(agent_id, tool_name)
        await self._log_revocation(agent_id, tool_name, reason)
    
    async def check_revocation(self, token_jti: str) -> bool:
        """检查 Token 是否已被撤销"""
        # 1. 检查本地撤销列表
        if token_jti in self.revocation_list:
            return True  # 已撤销
        
        # 2. 如果本地没有，检查分布式撤销列表（Redis）
        revoked = await self._check_global_revocation_list(token_jti)
        if revoked:
            # 同步到本地
            self.revocation_list[token_jti] = time.time()
        
        return revoked
```

#### 6.4 Short TTL for Sensitive Tools

敏感工具的短 TTL 策略：

```
敏感工具（如 delete, transfer）的 TTL 接近 0：
  每次调用都重新授权 = 每次调用的延迟增加 20-50ms
  但确保了：权限变更能在 5 秒内生效

非敏感工具（如 search）的长 TTL：
  大部分调用命中缓存 = 授权延迟接近 0
  权限变更可能需要 5 分钟才生效
  ✅ 可以接受（搜索权限的变更不紧急）

平衡策略（推荐）：
  ┌────────────────────────────────────┐
  │ 工具类型       │ TTL  │ 缓存命中率  │
  ├────────────────────────────────────┤
  │ search:read    │ 5min │  ~95%     │
  │ files:read     │ 5min │  ~90%     │
  │ files:write    │ 1min │  ~80%     │
  │ email:send     │ 30s  │  ~60%     │
  │ delete:*       │ 0s   │  ~0%      │
  └────────────────────────────────────┘
```

---

## Example Code: Python ToolAuthorizationManager with RBAC + ABAC, OAuth integration

以下是一个结合 RBAC + ABAC 的完整工具授权管理器，支持 OAuth 集成：

```python
"""
tool_authorization.py

完整的 Agent 工具授权管理器，支持：
- RBAC 基础角色授权
- ABAC 属性级动态授权
- OAuth 2.0 Token 管理
- 参数级授权检查
- 授权缓存
- 撤销处理
"""

import time
import hashlib
import json
from enum import Enum
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Optional, Set
import re


# ═══════════════════════════════════════════════════════════════
# 基础数据类型
# ═══════════════════════════════════════════════════════════════

class AuthDecision(Enum):
    """授权决策结果"""
    ALLOW = "allow"
    DENY = "deny"
    REQUIRE_APPROVAL = "require_approval"


@dataclass
class AuthResult:
    """授权结果"""
    decision: AuthDecision
    tool: str
    agent_id: str
    reason: str = ""
    constraints: Dict[str, Any] = field(default_factory=dict)
    evaluated_at: float = 0.0
    
    def __post_init__(self):
        if not self.evaluated_at:
            self.evaluated_at = time.time()


@dataclass
class AuthRequest:
    """授权请求"""
    agent_id: str
    agent_type: str
    tool: str
    action: str  # read, write, delete, admin
    params: Dict[str, Any]
    user_identity: str
    session_id: str
    risk_score: float = 0.0
    context: Dict[str, Any] = field(default_factory=dict)


@dataclass
class ToolPolicy:
    """工具策略定义"""
    tool_name: str
    allowed_roles: List[str]
    required_attributes: Dict[str, Any]
    param_rules: Dict[str, 'ParameterRule']
    rate_limit: int = 0  # 0 = unlimited
    requires_approval: bool = False
    approval_if_risk_above: float = 0.7
    ttl_cache: int = 300  # 授权缓存 TTL（秒）
    allowed_actions: List[str] = None


# ═══════════════════════════════════════════════════════════════
# 参数验证规则
# ═══════════════════════════════════════════════════════════════

class ParameterRule:
    """参数验证规则"""
    
    def __init__(self, 
                 param_name: str,
                 param_type: type,
                 required: bool = False,
                 allowed_values: Optional[List[Any]] = None,
                 min_value: Optional[float] = None,
                 max_value: Optional[float] = None,
                 pattern: Optional[str] = None,
                 max_length: Optional[int] = None,
                 deny_values: Optional[List[Any]] = None):
        self.param_name = param_name
        self.param_type = param_type
        self.required = required
        self.allowed_values = allowed_values
        self.min_value = min_value
        self.max_value = max_value
        self.pattern = pattern
        self.max_length = max_length
        self.deny_values = deny_values or []
    
    def validate(self, value: Any) -> tuple[bool, str]:
        if self.required and value is None:
            return False, f"Required parameter missing: {self.param_name}"
        if value is None:
            return True, ""
        if not isinstance(value, self.param_type):
            return False, f"Type mismatch for {self.param_name}: expected {self.param_type.__name__}"
        if self.allowed_values and value not in self.allowed_values:
            return False, f"Value '{value}' not allowed for {self.param_name}"
        if value in self.deny_values:
            return False, f"Value '{value}' is denied for {self.param_name}"
        if isinstance(value, (int, float)):
            if self.min_value is not None and value < self.min_value:
                return False, f"Value {value} < min {self.min_value} for {self.param_name}"
            if self.max_value is not None and value > self.max_value:
                return False, f"Value {value} > max {self.max_value} for {self.param_name}"
        if self.pattern and isinstance(value, str):
            if not re.match(self.pattern, value):
                return False, f"Value '{value}' does not match pattern for {self.param_name}"
        if self.max_length and isinstance(value, str) and len(value) > self.max_length:
            return False, f"Value length {len(value)} > max {self.max_length} for {self.param_name}"
        return True, ""


# ═══════════════════════════════════════════════════════════════
# 策略定义引擎
# ═══════════════════════════════════════════════════════════════

class PolicyEngine:
    """策略引擎 — 管理 RBAC 角色 + ABAC 属性策略"""
    
    def __init__(self):
        # ── RBAC: 角色定义 ──
        self.roles = {
            "read_only": {
                "description": "只读访问",
                "inherits": [],
                "allowed_tools": {
                    "search": {"actions": ["read"], "max_results": 100},
                    "read_file": {"actions": ["read"], "max_size_mb": 5},
                    "list_files": {"actions": ["read"]}
                }
            },
            "writer": {
                "description": "读写访问",
                "inherits": ["read_only"],
                "allowed_tools": {
                    "write_file": {"actions": ["write"], "max_size_mb": 10},
                    "update_record": {"actions": ["write"]},
                    "send_notification": {"actions": ["write"], "rate": "10/hour"}
                }
            },
            "operator": {
                "description": "运营操作",
                "inherits": ["writer"],
                "allowed_tools": {
                    "send_email": {"actions": ["write"], "rate": "50/hour"},
                    "manage_docs": {"actions": ["read", "write", "delete"]}
                }
            },
            "admin": {
                "description": "管理员",
                "inherits": ["operator"],
                "allowed_tools": {
                    "delete_file": {"actions": ["delete"]},
                    "admin_config": {"actions": ["read", "write"]},
                    "manage_users": {"actions": ["read", "write", "delete"]},
                    "transfer_money": {"actions": ["write"], "requires_approval": True}
                }
            }
        }
        
        # ── ABAC: 属性策略 ──
        # 这些策略基于属性（agent_type、资源敏感度、环境等）动态判定
        self.attribute_policies: List[Dict] = [
            # 策略 1: 敏感数据只能由高安全等级 Agent 在办公网络访问
            {
                "name": "sensitive_data_restriction",
                "priority": 100,
                "effect": "deny",
                "conditions": {
                    "resource.sensitivity": "high",
                    "agent.security_level": {"$nin": ["L3", "L4"]}
                }
            },
            # 策略 2: 高风险操作必须在风险评分低于阈值时自动允许
            {
                "name": "high_risk_auto_deny",
                "priority": 90,
                "effect": "deny",
                "conditions": {
                    "request.risk_score": {"$gt": 0.8},
                    "tool": {"$in": ["delete_file", "transfer_money", "admin_config"]}
                }
            },
            # 策略 3: 非工作时间限制写操作
            {
                "name": "off_hours_write_restriction",
                "priority": 80,
                "effect": "require_approval",
                "conditions": {
                    "environment.is_off_hours": True,
                    "action": {"$in": ["write", "delete"]}
                }
            },
            # 策略 4: 白名单路径保护
            {
                "name": "path_traversal_protection",
                "priority": 100,
                "effect": "deny",
                "conditions": {
                    "tool": "read_file",
                    "params.path": {"$regex": r"\.\.\/|\.\.\\|\/etc\/|\/proc\/"}
                }
            }
        ]
        
        # ── Agent 角色绑定 ──
        self.agent_role_map = {
            "agent-support-lite": ["read_only"],
            "agent-support-pro": ["writer"],
            "agent-data-analyst": ["writer"],
            "agent-devops": ["admin"],
            "agent-finance": ["operator"]
        }
    
    def _resolve_role(self, role_name: str) -> Set[str]:
        """解析角色及其继承的所有权限"""
        resolved = set()
        role = self.roles.get(role_name)
        if not role:
            return resolved
        
        # 添加自身工具
        for tool_name, tool_config in role["allowed_tools"].items():
            for action in tool_config.get("actions", []):
                resolved.add(f"{tool_name}:{action}")
        
        # 递归添加继承角色的工具
        for inherited_role in role.get("inherits", []):
            resolved |= self._resolve_role(inherited_role)
        
        return resolved
    
    def get_agent_permissions(self, agent_type: str) -> Set[str]:
        """获取 Agent 类型所有被授权的工具:操作"""
        role_names = self.agent_role_map.get(agent_type, [])
        permissions = set()
        
        for role_name in role_names:
            permissions |= self._resolve_role(role_name)
        
        return permissions
    
    def evaluate_rbac(self, request: AuthRequest) -> Optional[AuthDecision]:
        """评估 RBAC 角色权限"""
        permissions = self.get_agent_permissions(request.agent_type)
        required = f"{request.tool}:{request.action}"
        
        # 检查是否有显式授权
        if required in permissions:
            return AuthDecision.ALLOW
        
        # 检查是否有通配符授权（如 files:*）
        wildcard = f"{request.tool}:*"
        if wildcard in permissions:
            return AuthDecision.ALLOW
        
        return AuthDecision.DENY
    
    def evaluate_abac(self, request: AuthRequest, environment: Dict) -> AuthDecision:
        """评估 ABAC 属性策略"""
        # 构建上下文
        context = {
            "agent": {
                "id": request.agent_id,
                "type": request.agent_type,
                "security_level": self._get_agent_security_level(request.agent_type)
            },
            "request": {
                "risk_score": request.risk_score
            },
            "resource": {
                "sensitivity": self._get_resource_sensitivity(request.tool)
            },
            "environment": environment,
            "tool": request.tool,
            "action": request.action,
            "params": request.params
        }
        
        # 按优先级排序策略
        sorted_policies = sorted(
            self.attribute_policies, 
            key=lambda p: p["priority"], 
            reverse=True
        )
        
        for policy in sorted_policies:
            if self._match_conditions(policy["conditions"], context):
                effect = policy["effect"]
                if effect == "deny":
                    return AuthDecision.DENY
                elif effect == "require_approval":
                    return AuthDecision.REQUIRE_APPROVAL
                # "allow" 类型的 ABAC 策略需要继续评估（allow 不否决）
        
        # 没有匹配的拒绝策略 → 返回 None 表示 ABAC 不做决定
        return None
    
    def _get_agent_security_level(self, agent_type: str) -> str:
        levels = {
            "agent-support-lite": "L1",
            "agent-support-pro": "L2",
            "agent-data-analyst": "L2",
            "agent-finance": "L3",
            "agent-devops": "L4"
        }
        return levels.get(agent_type, "L1")
    
    def _get_resource_sensitivity(self, tool: str) -> str:
        sensitivities = {
            "search": "low",
            "read_file": "medium",
            "list_files": "low",
            "write_file": "medium",
            "send_email": "medium",
            "delete_file": "high",
            "transfer_money": "high",
            "admin_config": "high"
        }
        return sensitivities.get(tool, "low")
    
    def _match_conditions(self, conditions: Dict, context: Dict) -> bool:
        """检查上下文是否匹配策略条件"""
        for key, condition in conditions.items():
            value = self._get_nested_value(context, key)
            
            if isinstance(condition, dict):
                for op, target in condition.items():
                    if op == "$gt" and not (isinstance(value, (int, float)) and value > target):
                        return False
                    if op == "$gte" and not (isinstance(value, (int, float)) and value >= target):
                        return False
                    if op == "$lt" and not (isinstance(value, (int, float)) and value < target):
                        return False
                    if op == "$in" and value not in target:
                        return False
                    if op == "$nin" and value in target:
                        return False
                    if op == "$regex" and not (isinstance(value, str) and re.search(target, value)):
                        return False
            else:
                if value != condition:
                    return False
        
        return True
    
    def _get_nested_value(self, context: Dict, key: str) -> Any:
        """从嵌套字典中安全提取值（如 'resource.sensitivity'）"""
        parts = key.split(".")
        current = context
        for part in parts:
            if isinstance(current, dict):
                current = current.get(part)
            else:
                return None
        return current


# ═══════════════════════════════════════════════════════════════
# 参数验证器
# ═══════════════════════════════════════════════════════════════

class ParameterValidator:
    """工具参数验证器"""
    
    # 工具参数规则定义
    TOOL_RULES = {
        "search": {
            "query": ParameterRule("query", str, required=True, max_length=500),
            "max_results": ParameterRule("max_results", int, min_value=1, max_value=100),
            "source": ParameterRule("source", str, 
                                     allowed_values=["web", "news", "images", "scholar"])
        },
        "read_file": {
            "path": ParameterRule("path", str, required=True, max_length=1000),
            "encoding": ParameterRule("encoding", str, 
                                       allowed_values=["utf-8", "ascii", "utf-16"]),
            "max_size": ParameterRule("max_size", int, min_value=1, max_value=50*1024*1024)
        },
        "write_file": {
            "path": ParameterRule("path", str, required=True, max_length=1000),
            "content": ParameterRule("content", str, required=True, max_length=10*1024*1024),
            "encoding": ParameterRule("encoding", str, allowed_values=["utf-8", "ascii"])
        },
        "send_email": {
            "to": ParameterRule("to", str, required=True, 
                                pattern=r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'),
            "cc": ParameterRule("cc", list, max_length=10),  # 最多 10 个抄送
            "subject": ParameterRule("subject", str, required=True, max_length=200),
            "body": ParameterRule("body", str, max_length=50000),
            "priority": ParameterRule("priority", str, 
                                       allowed_values=["low", "normal", "high"]),
            "attachments": None  # 不允许附件
        },
        "delete_file": {
            "path": ParameterRule("path", str, required=True, max_length=1000),
            "permanent": ParameterRule("permanent", bool, allowed_values=[False])
        },
        "transfer_money": {
            "amount": ParameterRule("amount", (int, float), required=True, 
                                     min_value=0.01, max_value=100000),
            "currency": ParameterRule("currency", str, 
                                       allowed_values=["USD", "EUR", "CNY", "JPY"]),
            "to_account": ParameterRule("to_account", str, required=True, 
                                         pattern=r'^[a-zA-Z0-9]{8,34}$'),
            "note": ParameterRule("note", str, max_length=500)
        }
    }
    
    @classmethod
    def validate(cls, tool: str, params: Dict) -> tuple[bool, List[str]]:
        """验证工具参数是否合法"""
        rules = cls.TOOL_RULES.get(tool, {})
        if rules is None:
            return False, [f"Tool '{tool}' has no parameter rules defined (may be denied)"]
        
        errors = []
        
        # 检查是否传入了未定义的参数
        for key in params:
            if key not in rules:
                errors.append(f"Unknown parameter '{key}' for tool '{tool}'")
        
        # 检查每个参数的规则
        for param_name, rule in rules.items():
            if rule is None:
                # None 表示不允许此参数
                if param_name in params:
                    errors.append(f"Parameter '{param_name}' is not allowed for tool '{tool}'")
                continue
            
            value = params.get(param_name)
            valid, error = rule.validate(value)
            if not valid:
                errors.append(error)
        
        return len(errors) == 0, errors


# ═══════════════════════════════════════════════════════════════
# OAuth 集成
# ═══════════════════════════════════════════════════════════════

class OAuthTokenManager:
    """OAuth 2.0 Token 管理器 — Agent 工具授权委托"""
    
    def __init__(self):
        self.tokens: Dict[str, Dict] = {}  # token_hash → token_info
        self.revoked_tokens: Set[str] = set()
    
    async def exchange_code_for_token(self, 
                                       auth_code: str,
                                       code_verifier: str,
                                       client_id: str,
                                       client_secret: str) -> Dict:
        """用授权码交换 Access Token（Authorization Code + PKCE Flow）"""
        # 在实际实现中，这会调用 Auth Server 的 /token 端点
        # 这里展示流程逻辑
        
        # 1. 验证 PKCE code_verifier
        # 2. 验证 client_id + client_secret
        # 3. Auth Server 返回 Access Token + Refresh Token
        
        access_token = self._generate_token(client_id, "access", ttl=3600)
        refresh_token = self._generate_token(client_id, "refresh", ttl=86400 * 30)
        
        token_info = {
            "access_token": access_token,
            "refresh_token": refresh_token,
            "token_type": "Bearer",
            "expires_in": 3600,
            "scope": "search:read files:read email:send",
            "client_id": client_id,
            "issued_at": time.time()
        }
        
        token_hash = self._hash_token(access_token)
        self.tokens[token_hash] = token_info
        
        return token_info
    
    async def validate_token(self, token: str, required_scope: str) -> bool:
        """验证 Token 是否有效并具有所需 scope"""
        # 1. 检查是否被撤销
        if token in self.revoked_tokens:
            return False
        
        # 2. 检查 Token 是否有效
        token_hash = self._hash_token(token)
        token_info = self.tokens.get(token_hash)
        if not token_info:
            return False
        
        # 3. 检查是否过期
        if time.time() > token_info["issued_at"] + token_info["expires_in"]:
            return False
        
        # 4. 检查 scope
        token_scopes = token_info.get("scope", "").split()
        if required_scope not in token_scopes:
            return False
        
        return True
    
    async def refresh_token(self, refresh_token: str) -> Optional[Dict]:
        """刷新 Access Token"""
        for token_hash, token_info in self.tokens.items():
            if token_info.get("refresh_token") == refresh_token:
                # 生成新的 Access Token
                new_access = self._generate_token(
                    token_info["client_id"], "access", ttl=3600
                )
                old_hash = self._hash_token(token_info["access_token"])
                
                # 更新记录
                token_info["access_token"] = new_access
                token_info["issued_at"] = time.time()
                
                # 更新索引
                self.tokens[self._hash_token(new_access)] = token_info
                if old_hash in self.tokens:
                    del self.tokens[old_hash]
                
                return {"access_token": new_access, "expires_in": 3600}
        
        return None
    
    async def revoke_token(self, token: str) -> bool:
        """撤销 Token"""
        self.revoked_tokens.add(token)
        token_hash = self._hash_token(token)
        if token_hash in self.tokens:
            # 也撤销对应的 refresh token
            refresh = self.tokens[token_hash].get("refresh_token")
            if refresh:
                self.revoked_tokens.add(refresh)
            return True
        return False
    
    def _generate_token(self, client_id: str, token_type: str, ttl: int) -> str:
        """生成模拟 Token"""
        import secrets
        raw = f"{client_id}:{token_type}:{secrets.token_hex(32)}:{ttl}"
        return base64_encode(raw)
    
    def _hash_token(self, token: str) -> str:
        return hashlib.sha256(token.encode()).hexdigest()


# ═══════════════════════════════════════════════════════════════
# 授权缓存
# ═══════════════════════════════════════════════════════════════

class AuthorizationCache:
    """分级授权缓存"""
    
    def __init__(self):
        self._cache: Dict[str, tuple[AuthResult, float]] = {}  # key → (result, expiry)
        self._max_size = 10000
        self._stats = {"hits": 0, "misses": 0, "invalidations": 0}
    
    def _make_key(self, request: AuthRequest) -> str:
        """生成缓存键"""
        key_parts = [
            request.agent_id,
            request.tool,
            request.action,
            json.dumps(request.params, sort_keys=True),
            request.session_id
        ]
        return ":".join(key_parts)
    
    def get(self, request: AuthRequest) -> Optional[AuthResult]:
        """获取缓存的授权结果"""
        key = self._make_key(request)
        entry = self._cache.get(key)
        
        if entry:
            result, expiry = entry
            if time.time() < expiry:
                self._stats["hits"] += 1
                return result
            else:
                del self._cache[key]
        
        self._stats["misses"] += 1
        return None
    
    def set(self, request: AuthRequest, result: AuthResult, ttl: int):
        """缓存授权结果"""
        if ttl <= 0:
            return
        
        key = self._make_key(request)
        self._cache[key] = (result, time.time() + ttl)
        
        # LRU 淘汰
        if len(self._cache) > self._max_size:
            oldest_key = min(self._cache, key=lambda k: self._cache[k][1])
            del self._cache[oldest_key]
    
    def invalidate_for_agent(self, agent_id: str):
        """失效指定 Agent 的所有缓存"""
        keys_to_delete = [
            key for key in self._cache
            if key.startswith(agent_id)
        ]
        for key in keys_to_delete:
            del self._cache[key]
        
        self._stats["invalidations"] += len(keys_to_delete)
    
    def invalidate_for_tool(self, agent_id: str, tool: str):
        """失效指定 Agent 对特定工具的缓存"""
        keys_to_delete = [
            key for key in self._cache
            if key.startswith(f"{agent_id}:{tool}")
        ]
        for key in keys_to_delete:
            del self._cache[key]
        
        self._stats["invalidations"] += len(keys_to_delete)
    
    @property
    def stats(self) -> Dict:
        return {
            **self._stats,
            "size": len(self._cache),
            "hit_rate": self._stats["hits"] / max(self._stats["hits"] + self._stats["misses"], 1)
        }


# ═══════════════════════════════════════════════════════════════
# 主授权管理器
# ═══════════════════════════════════════════════════════════════

class ToolAuthorizationManager:
    """Agent 工具授权管理器 — 统一入口"""
    
    def __init__(self):
        self.policy_engine = PolicyEngine()
        self.param_validator = ParameterValidator()
        self.oauth_manager = OAuthTokenManager()
        self.cache = AuthorizationCache()
        
        # 不同工具的缓存 TTL 配置
        self.tool_cache_ttl = {
            "search": 300,
            "read_file": 300,
            "list_files": 300,
            "write_file": 60,
            "send_email": 30,
            "update_record": 60,
            "delete_file": 0,     # 不缓存
            "transfer_money": 0,  # 不缓存
            "admin_config": 10
        }
    
    async def authorize(self, 
                        request: AuthRequest,
                        environment: Optional[Dict] = None) -> AuthResult:
        """完整的工具授权流程"""
        env = environment or {}
        
        # ── 步骤 1: 检查授权缓存 ──
        cached = self.cache.get(request)
        if cached:
            return cached
        
        # ── 步骤 2: 参数验证 ──
        params_valid, param_errors = ParameterValidator.validate(
            request.tool, request.params
        )
        if not params_valid:
            result = AuthResult(
                decision=AuthDecision.DENY,
                tool=request.tool,
                agent_id=request.agent_id,
                reason=f"Parameter validation failed: {'; '.join(param_errors)}"
            )
            # 参数错误的授权结果仅短暂缓存
            self.cache.set(request, result, ttl=10)
            return result
        
        # ── 步骤 3: RBAC 角色评估 ──
        rbac_decision = self.policy_engine.evaluate_rbac(request)
        if rbac_decision == AuthDecision.DENY:
            result = AuthResult(
                decision=AuthDecision.DENY,
                tool=request.tool,
                agent_id=request.agent_id,
                reason=f"Agent '{request.agent_type}' lacks RBAC permission for {request.tool}:{request.action}"
            )
            return result
        
        # ── 步骤 4: ABAC 属性策略评估 ──
        abac_decision = self.policy_engine.evaluate_abac(request, env)
        if abac_decision in (AuthDecision.DENY, AuthDecision.REQUIRE_APPROVAL):
            result = AuthResult(
                decision=abac_decision,
                tool=request.tool,
                agent_id=request.agent_id,
                reason=f"ABAC policy restricted access: tool={request.tool}, risk={request.risk_score}"
            )
            # ABAC 拒绝结果短暂缓存
            self.cache.set(request, result, ttl=15)
            return result
        
        # ── 步骤 5: OAuth Token 验证（如需要）──
        oauth_token = request.context.get("oauth_token")
        tool_scope = f"{request.tool}:{request.action}"
        
        if oauth_token:
            token_valid = await self.oauth_manager.validate_token(
                oauth_token, tool_scope
            )
            if not token_valid:
                return AuthResult(
                    decision=AuthDecision.DENY,
                    tool=request.tool,
                    agent_id=request.agent_id,
                    reason=f"OAuth token invalid or missing required scope '{tool_scope}'"
                )
        
        # ── 所有检查通过 ──
        ttl = self.tool_cache_ttl.get(request.tool, 60)
        result = AuthResult(
            decision=AuthDecision.ALLOW,
            tool=request.tool,
            agent_id=request.agent_id,
            reason="All authorization checks passed",
            constraints={"rate_limit": 100, "max_results": 100}
        )
        
        self.cache.set(request, result, ttl=ttl)
        return result


# ═══════════════════════════════════════════════════════════════
# 使用示例
# ═══════════════════════════════════════════════════════════════

async def demo():
    """演示 ToolAuthorizationManager 的使用"""
    authz = ToolAuthorizationManager()
    
    # 场景 1: 只读 Agent 搜索数据（应该通过）
    request_1 = AuthRequest(
        agent_id="agent-support-lite-01",
        agent_type="agent-support-lite",
        tool="search",
        action="read",
        params={"query": "AI Agent safety", "max_results": 10},
        user_identity="user_abc",
        session_id="sess_001",
        risk_score=0.1
    )
    
    result_1 = await authz.authorize(request_1)
    print(f"场景 1: {result_1.decision.value} — {result_1.reason}")
    # 输出: allow — All authorization checks passed
    
    # 场景 2: 只读 Agent 尝试删除文件（应该拒绝）
    request_2 = AuthRequest(
        agent_id="agent-support-lite-01",
        agent_type="agent-support-lite",
        tool="delete_file",
        action="delete",
        params={"path": "/shared/tmp.txt"},
        user_identity="user_abc",
        session_id="sess_001",
        risk_score=0.9
    )
    
    result_2 = await authz.authorize(request_2)
    print(f"场景 2: {result_2.decision.value} — {result_2.reason}")
    # 输出: deny — Agent 'agent-support-lite' lacks RBAC permission for delete_file:delete
    
    # 场景 3: 高风险操作 + 高风险评分（ABAC 拒绝）
    request_3 = AuthRequest(
        agent_id="agent-devops-01",
        agent_type="agent-devops",
        tool="delete_file",
        action="delete",
        params={"path": "/shared/config.yaml"},
        user_identity="user_admin",
        session_id="sess_002",
        risk_score=0.85  # 超过 0.8 阈值
    )
    
    result_3 = await authz.authorize(request_3, {"is_off_hours": False})
    print(f"场景 3: {result_3.decision.value} — {result_3.reason}")
    # 输出: deny — ABAC policy restricted access: tool=delete_file, risk=0.85
    
    # 场景 4: 参数验证拒绝（不允许附件）
    request_4 = AuthRequest(
        agent_id="agent-finance-01",
        agent_type="agent-finance",
        tool="send_email",
        action="write",
        params={
            "to": "user@example.com",
            "subject": "Report",
            "body": "Please find the report attached.",
            "attachments": ["report.pdf"]  # 不允许
        },
        user_identity="user_finance",
        session_id="sess_003",
        risk_score=0.3
    )
    
    result_4 = await authz.authorize(request_4, {"is_off_hours": False})
    print(f"场景 4: {result_4.decision.value} — {result_4.reason}")
    # 输出: deny — Parameter validation failed: Parameter 'attachments' is not allowed for tool 'send_email'
    
    # 场景 5: 带 OAuth Token 的授权
    request_5 = AuthRequest(
        agent_id="agent-support-pro-01",
        agent_type="agent-support-pro",
        tool="read_file",
        action="read",
        params={"path": "/shared/docs/manual.pdf"},
        user_identity="user_pro",
        session_id="sess_004",
        risk_score=0.2,
        context={"oauth_token": "valid_token_here"}
    )
    
    result_5 = await authz.authorize(request_5, {"is_off_hours": False})
    print(f"场景 5: {result_5.decision.value} — {result_5.reason}")
    
    # 查看缓存统计
    print(f"\n缓存统计: {authz.cache.stats}")
```

---

## Capability Boundaries: authorization covers access but not usage

工具授权有一个重要的认知边界：**授权控制的是"谁能访问"（access），而不是"使用者的意图"（usage）。**

```
授权（Authorization）解决的问题：
  ┌─────────────────────────────────────┐
  │ Agent X 可以调用工具 Y 吗？           │
  │ Agent X 可以调用 Z 参数吗？           │
  │ Agent X 以用户 U 的身份调用合法吗？     │
  └─────────────────────────────────────┘

授权不解决的问题：
  ┌─────────────────────────────────────┐
  │ Agent X 用工具 Y 做的事情合法/合理吗？  │
  │ Agent X 是否被 prompt 注入操纵了？     │
  │ Agent X 是否在滥用已授权的工具？        │
  │ 用户授权 Agent 发送邮件——              │
  │ 但 Agent 发的内容是钓鱼链接怎么办？     │
  └─────────────────────────────────────┘
```

### 授权边界的具体案例

| 场景 | 授权判定 | 授权通过？ | 真正的问题 |
|------|---------|-----------|-----------|
| Agent 代表用户读取公开文档 | read_file:doc.txt → RBAC 允许 | ✅ 通过 | 没有问题 |
| Agent 被注入后读取用户私钥 | read_file:id_rsa → RBAC 允许 | ✅ 通过（它有读文件权限） | ❌ 授权边界外：注入防御失败 |
| Agent 发送会议提醒邮件 | send_email:team@company.com → 授权允许 | ✅ 通过 | 没有问题 |
| Agent 被操纵发送钓鱼邮件 | send_email:external@attacker.com → 授权允许 | ✅ 通过 | ❌ 授权边界外：意图检测缺失 |
| Agent 删除缓存文件 | delete_file:/tmp/cache → ABAC 高风险拒绝 | ❌ 拒绝 | 授权拦截成功（正确的保护） |

### 授权边界的三种应对策略

```
策略一：在授权层做最大化限制（最严格）
  • 原理：对工具的能力做最保守的限制
  • 例：send_email 永远不允许带附件
  • 优点：简单、安全
  • 缺点：可能过度限制，影响有用功能的 Agent 使用

策略二：在授权层之外加 usage monitoring
  • 原理：授权层通过后，使用行为监控层检测异常
  • 例：Agent 突然在非办公时间大量发邮件 → 触发异常告警
  • 优点：不影响 Agent 功能，事后可追溯
  • 缺点：不是实时防护

策略三：分层防护（推荐）
  • 授权层：控制"能否调用"
  • 内容安全层：控制"调用是否合适"
  • 行为监控层：控制"使用模式是否异常"

三层协同的完整流程:
  工具调用请求
      │
      ▼
  ┌────────────────┐
  │ 授权层          │ ── 控制"能不能调用"
  │ • RBAC/ABAC     │
  │ • OAuth Token   │
  │ • 参数验证       │
  └───────┬────────┘
          │ 通过
          ▼
  ┌────────────────┐
  │ 内容安全层      │ ── 控制"调用的内容是否安全"
  │ • 输出护栏     │
  │ • 内容过滤     │
  │ • PII 检测     │
  └───────┬────────┘
          │ 通过
          ▼
  ┌────────────────┐
  │ 行为监控层      │ ── 控制"使用模式是否异常"
  │ • 异常检测     │
  │ • 速率分析     │
  │ • 模式匹配     │
  └───────┬────────┘
          │ 通过
          ▼
      执行工具调用
```

---

## Comparison: scope vs RBAC vs ABAC for different Agent scenarios

```
场景需求矩阵:

场景                          推荐模型         原因
─────────────────────────────────────────────────────────────────────
个人助手（简单工具，1-5 个）       Scope    最简单，基本够用
企业客服系统（多角色，50+ 工具）    RBAC    角色分明，易于管理
DevOps 自动化平台（复杂上下文）    ABAC    需要上下文感知的动态授权
多租户 SaaS Agent 平台           ReBAC   天然支持多租户资源隔离
金融/医疗 Agent（合规要求高）     ABAC+RBAC 双层保护
开源社区 Agent（轻量级部署）      Scope   零额外依赖，配置简单
企业全栈 Agent 平台              RBAC+ABAC 灵活与控制的平衡
```

### 详细对比

| 对比维度 | Scope | RBAC | ABAC | ReBAC |
|---------|-------|------|------|-------|
| **部署复杂度** | 低（只需定义 scope 列表） | 中（需要角色管理服务） | 高（需要策略引擎 + 属性源） | 高（需要图数据库 + 关系服务） |
| **安全等级** | ★★★☆☆ | ★★★★☆ | ★★★★★ | ★★★★☆ |
| **性能开销** | 低（常量时间查找） | 低（角色 -> 权限查表） | 中（策略匹配可能需要多次条件评估） | 中（图遍历深度影响性能） |
| **授权粒度** | 工具级别 | 工具 + 操作级别 | 参数 + 数据级别 | 资源 + 关系级别 |
| **动态策略** | 不支持 | 不支持（角色静态） | 完全支持（基于属性实时计算） | 支持（关系变化即更新） |
| **审计追踪** | 简单（谁用了什么工具） | 中等（谁用了什么角色下的什么工具） | 详细（为什么允许/拒绝的完整上下文） | 详细（完整的访问路径） |
| **适合场景** | 小型 Agent、个人项目 | 企业 Agent、团队协作 | 金融、医疗、高安全场景 | 多租户平台、社交图谱 |
| **与 OAuth 集成** | 原生支持（scope 字段） | 需要额外映射（role -> scope） | 间接集成（scope 作为属性之一） | 复杂（需要自定义映射） |
| **主流实现** | OAuth 2.0 Scopes | Keycloak, Auth0, 自建 | OPA, AWS Cedar, Casbin | Google Zanzibar, Auth0 FGA, Ory Keto |
| **配置管理** | YAML/JSON 列表 | 角色定义 + 角色分配 | 策略语言代码（如 Rego） | 关系元组（Tuples） |

### 选择决策树

```
你的 Agent 系统规模是？
  │
  ├── 小型（< 10 个工具，< 5 个角色）
  │     └── Scope ✅ 最简单，足够用
  │
  ├── 中型（10-50 个工具，5-20 个角色）
  │     │
  │     ├── 角色固定，不需要动态策略 → RBAC ✅
  │     └── 需要基于上下文动态调整权限
  │           └── RBAC + 少量 ABAC 规则 ✅
  │
  ├── 大型（50+ 工具，20+ 角色，多租户）
  │     │
  │     ├── 只有组织内部使用 → ABAC + RBAC ✅
  │     └── 多租户 + 资源隔离 → ReBAC ✅
  │
  └── 特大型（100+ 工具，跨组织协作）
        └── ABAC + ReBAC 混合模型 ✅
```

---

## Engineering Optimization: authorization caching, distributed auth decisions, policy-as-code

### 1. Authorization Caching Deep Dive

#### 分层缓存架构

```
  ┌──────────┐
  │ Agent    │
  │ Runtime  │ ←── L1: 内存缓存（Agent 进程内）
  └────┬─────┘        • 最快：< 1ms
       │              • 容量小：每个 Agent 独立
       │              • 生命周期：随 Agent 进程
       ▼
  ┌──────────┐
  │ Auth     │
  │ Service  │ ←── L2: Redis 分布式缓存
  └────┬─────┘        • 中等：1-5ms
       │              • 容量大：集群共享
       │              • 生命周期：可配置 TTL
       ▼
  ┌──────────┐
  │ Policy   │
  │ Engine   │ ←── L3: 策略引擎实时计算
  └──────────┘        • 最慢：20-100ms
                      • 完整策略评估
                      • 无缓存，每次都是最新结果
```

#### 缓存失效策略

```python
class DistributedCacheInvalidation:
    """分布式缓存失效策略"""
    
    INVALIDATION_EVENTS = {
        # 事件类型 → 失效范围
        "permission_updated": {
            "scope": "agent",      # 失效特定 Agent 的缓存
            "ttl_check": False     # 不需要等待 TTL，立即失效
        },
        "role_updated": {
            "scope": "role",       # 失效某一角色下所有 Agent 的缓存
            "ttl_check": False
        },
        "tool_policy_changed": {
            "scope": "tool",       # 失效特定工具的所有授权缓存
            "ttl_check": False
        },
        "user_revoked": {
            "scope": "user",       # 失效用户关联的所有 Agent 缓存
            "ttl_check": False
        },
        "session_ended": {
            "scope": "session",    # 失效特定会话的缓存
            "ttl_check": True      # 可以等待 TTL 自然过期
        }
    }
    
    def invalidate(self, event_type: str, target_id: str):
        """按事件类型失效缓存"""
        config = self.INVALIDATION_EVENTS.get(event_type)
        if not config:
            return
        
        if config["scope"] == "agent":
            # 发布到 Redis Pub/Sub: invalidate:agent:{agent_id}
            self.redis.publish(f"invalidate:agent:{target_id}", event_type)
        
        elif config["scope"] == "role":
            # 获取角色下的所有 Agent，分别失效
            agents = self.get_agents_by_role(target_id)
            for agent_id in agents:
                self.redis.publish(f"invalidate:agent:{agent_id}", event_type)
        
        elif config["scope"] == "tool":
            # 失效所有包含该工具的缓存项
            self.redis.publish(f"invalidate:tool:{target_id}", event_type)
```

### 2. Distributed Auth Decisions（分布式授权决策）

在大型 Agent 平台中，授权决策需要在分布式服务间协调：

```
          ┌──────────┐
          │  用户     │
          └────┬─────┘
               │
               ▼
     ┌─────────────────┐
     │   Agent Gateway  │ ←── 负载均衡
     └────────┬────────┘
              │
      ┌───────┼───────┐
      ▼       ▼       ▼
  ┌────────┐ ┌────────┐ ┌────────┐
  │ Agent  │ │ Agent  │ │ Agent  │
  │ 实例 1  │ │ 实例 2  │ │ 实例 3  │
  └───┬────┘ └───┬────┘ └───┬────┘
      │          │          │
      └──────────┼──────────┘
                 │
                 ▼
        ┌────────────────┐
        │   Auth Service  │ ←── 无状态，水平扩展
        │   (授权决策中心) │
        └───────┬────────┘
                │
        ┌───────┼───────┐
        ▼       ▼       ▼
    ┌──────┐ ┌──────┐ ┌──────┐
    │ Redis │ │Policy│ │Vault │
    │缓存   │ │引擎  │ │密钥  │
    └──────┘ └──────┘ └──────┘
```

**分布式授权设计原则：**

| 原则 | 说明 | 实现方式 |
|------|------|---------|
| **无状态授权服务** | Auth Service 不存储会话状态 | 所有必要信息在 Token 中（JWT） |
| **最终一致性** | 缓存失效不要求即时同步 | 短 TTL + 主动失效 + 被动过期 |
| **幂等授权决策** | 相同输入产生相同决策 | 纯函数式策略评估 |
| **故障降级** | Auth Service 宕机时怎么办 | 降级模式：拒绝所有（安全）或使用本地缓存（可用） |
| **请求追踪** | 跨服务追踪授权决策链 | 每个请求携带 trace_id |

### 3. Policy-as-Code（策略即代码）

将授权策略作为代码管理，而不是硬编码或配置——这是现代授权系统的最佳实践：

```
Policy-as-Code 工作流:

  1. 开发 ──► 用 Rego/Cedar/Typhon DSL 编写策略
  2. 测试 ──► 单元测试验证策略行为
  3. CI/CD ──► 自动测试、策略 lint、安全扫描
  4. 部署 ──► 推送到策略分发服务
  5. 运行 ──► 策略引擎实时评估
  6. 审计 ──► 所有策略决策可追踪到源码版本

  策略版本控制:
    policies/
    ├── agent-tools/
    │   ├── v1/
    │   │   ├── search.rego        # 搜索工具策略
    │   │   ├── files.rego         # 文件工具策略
    │   │   └── email.rego         # 邮件工具策略
    │   ├── v2/
    │   │   ├── search.rego        # 更新后的搜索策略
    │   │   └── ...
    │   └── tests/
    │       ├── search_test.rego
    │       ├── files_test.rego
    │       └── email_test.rego
    └── policy-config.yaml         # 策略分发配置
```

**OPA (Open Policy Agent) Rego 策略示例：**

```rego
# agent-tools/files.rego
# Agent 文件工具授权策略

package agent.files

import future.keywords.if
import future.keywords.in

# 默认拒绝
default allow := false

# ── 辅助函数 ──
# Agent 的安全等级
agent_security_level := level {
    levels := {"agent-support-lite": "L1", "agent-support-pro": "L2", 
               "agent-finance": "L3", "agent-devops": "L4"}
    level := levels[input.agent.type]
}

# 资源的敏感度
resource_sensitivity := sensitivity {
    sensitivities := {"read_file": "medium", "write_file": "medium", 
                      "delete_file": "high", "list_files": "low"}
    sensitivity := sensitivities[input.tool]
}

# 是否在办公时间 (09:00-18:00)
is_business_hours {
    hour := time.clock(time.now_ns())[0]
    hour >= 9
    hour < 18
}

# ── 策略规则 ──

# 规则 1: 只读工具对所有认证 Agent 开放
allow if {
    input.tool in ["list_files", "read_file"]
    input.action == "read"
    input.agent.authenticated == true
}

# 规则 2: 写文件需要 Writer 以上角色
allow if {
    input.tool == "write_file"
    input.action == "write"
    input.agent.roles[_] in ["writer", "operator", "admin"]
    resource_sensitivity != "high"
    is_business_hours  # 只能在办公时间写
}

# 规则 3: 限制路径遍历
deny[{"reason": "路径遍历攻击检测", "path": input.params.path}] {
    input.tool in ["read_file", "write_file", "delete_file"]
    contains(input.params.path, "../")
}

deny[{"reason": "路径遍历攻击检测", "path": input.params.path}] {
    input.tool in ["read_file", "write_file", "delete_file"]
    contains(input.params.path, "..\\")
}

# 规则 4: 删除文件需要 Admin 角色且低风险评分
allow if {
    input.tool == "delete_file"
    input.action == "delete"
    input.agent.roles[_] == "admin"
    input.risk_score < 0.5
}

# 规则 5: 非办公时间的写操作需要审批
allow if {
    input.tool in ["write_file", "delete_file"]
    not is_business_hours
    input.approval.status == "approved"
    input.agent.roles[_] in ["operator", "admin"]
}
```

**Policy-as-Code 最佳实践：**

| 实践 | 说明 | 收益 |
|------|------|------|
| 版本控制 | 策略存储在 Git 中 | 可追溯、可回滚、可 Code Review |
| 策略测试 | 为策略编写单元测试 | 防止回归、验证策略行为 |
| 逐步部署 | 先 10% 流量验证再全量 | 降低错误策略的影响范围 |
| 策略模拟 | 离线模拟策略变更的影响 | 提前发现意外拒绝或过度放行 |
| 监控告警 | 监控授权拒绝率、缓存命中率 | 及时发现策略问题和性能退化 |
| 策略文档 | 用注解和 README 说明策略意图 | 降低维护成本、帮助团队理解 |
| 策略 Lint | 自动检查策略中的常见错误 | 避免语法错误和逻辑缺陷 |

---

## 总结：工具授权的核心要点

```
┌─────────────────────────────────────────────────────────────────┐
│                    Agent 工具授权核心要点                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 认证 ≠ 授权                                                 │
│     认证说"你是谁"，授权说"你能做什么"                              │
│     • Agent 认证：验证 Agent 身份凭证                              │
│     • Agent 授权：判定 Agent 能调用哪些工具、哪些参数               │
│                                                                  │
│  2. 选择适合规模的授权模型                                         │
│     • 小型：Scope 就够了                                          │
│     • 中型：RBAC 最合适                                           │
│     • 大型：ABAC 提供需要的灵活性                                  │
│     • 多租户：ReBAC 天然适合                                       │
│                                                                  │
│  3. 授权必须分层                                                  │
│     • 工具级授权：控制"能用哪个工具"                                │
│     • 操作级授权：控制"能读/写/删"                                  │
│     • 参数级授权：控制"参数允许范围"                                │
│     • 数据级授权：控制"能看到哪些字段"                              │
│                                                                  │
│  4. OAuth 2.0 是 Agent 工具授权的基石                              │
│     • Authorization Code + PKCE：有用户界面的 Agent                │
│     • Device Flow：无头 Agent                                     │
│     • OIDC：额外提供身份验证                                       │
│                                                                  │
│  5. 授权缓存需要分级管理                                           │
│     • 低风险工具：长 TTL、高缓存命中率                              │
│     • 高风险工具：短 TTL 或不缓存                                  │
│     • 主动失效：事件驱动缓存清除                                   │
│                                                                  │
│  6. MCP 正在成为 Agent 工具协议标准                                │
│     • 能力协商：按需声明和授权                                      │
│     • 来源验证：确保连接方身份可信                                  │
│     • 权限边界：内置工具级和参数级限制                              │
│                                                                  │
│  7. 授权不解决所有问题                                             │
│     • 授权控制"能不能调用"，不控制"调用是否合理"                    │
│     • 需要结合内容安全层和行为监控层形成完整防护                    │
│                                                                  │
│  8. Policy-as-Code 是生产级授权的必由之路                           │
│     • 策略版本化、可测试、可审查                                    │
│     • 支持灰度部署和策略模拟                                       │
│     • 解耦策略逻辑和应用程序代码                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```
