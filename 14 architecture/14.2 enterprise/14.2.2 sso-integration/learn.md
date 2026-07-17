# 14.2.2 SSO 集成 — Single Sign-On Integration

> **核心思想**：企业级 Agent 系统必须与企业现有的身份管理系统（IdP）集成，支持单点登录（SSO）、目录服务同步和统一的身份治理。SSO 集成不是简单的"能用企业邮箱登录"，而是需要在 **认证、授权、身份传播、会话管理、委托授权**五个层面与企业身份体系深度对接。

---

## 1. 基本原理

### 1.1 什么是 Agent 系统的 SSO

SSO 让用户使用企业已有的身份凭证（如 AD 账号）登录 Agent 平台，无需额外注册。更重要的是，用户的**身份、角色、权限**从 IdP 自动同步到 Agent 系统，作为所有 Agent 操作的基础授权依据。

```
企业身份管理流程:

┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ 用户登录  │───►│ IdP 认证  │───►│ Token    │───►│ Agent    │
│ (浏览器)  │    │ (Okta/AZ)│    │ 签发     │    │ 平台     │
└─────────┘    └──────────┘    └──────────┘    └──────────┘
                                                    │
                                          ┌─────────▼─────────┐
                                          │  身份传播全链路      │
                                          │                    │
                                          │  Token → Agent      │
                                          │    → LLM → 工具     │
                                          │    → 知识库 → 日志   │
                                          └────────────────────┘
```

### 1.2 背景与演进

**之前怎么做**：
- **Phase 1（独立账号）**：每个系统都有自己的用户表和密码——用户需要记 N 套密码
- **Phase 2（LDAP 集成）**：系统从 LDAP/AD 读取用户信息，但每个系统独立实现登录逻辑
- **Phase 3（SSO 标准协议）**：采用 SAML/OAuth 2.0/OIDC 标准协议，统一的认证入口
- **Phase 4（身份即服务）**：Okta/Azure AD/Auth0 等 IdP 提供完整的身份生命周期管理

**核心矛盾**：Agent 的**自主、长时间运行**特性与 SSO Token 的**短生命周期**之间的矛盾。

传统 SSO 假设：
- 用户操作是同步的（请求-响应）
- Token 有效期可以很短（15-60 分钟）
- 每次操作都需要用户上下文

Agent 的现实：
- 任务可能运行数小时
- Agent 需要自主调用工具
- 工具调用需要原始用户的授权上下文

### 1.3 四层集成模型

```
┌─────────────────────────────────────────────────────┐
│  层 1: 认证 (Authentication)                          │
│  用户是谁？                                          │
│  协议: OIDC / SAML / LDAP                            │
│  产出: ID Token (用户身份)                           │
├─────────────────────────────────────────────────────┤
│  层 2: 授权 (Authorization)                          │
│  用户能做什么？                                      │
│  协议: OAuth 2.0 / SCIM                              │
│  产出: Access Token (权限范围)                       │
├─────────────────────────────────────────────────────┤
│  层 3: 身份传播 (Identity Propagation)               │
│  用户身份如何在 Agent 链路中传递？                    │
│  机制: Token Exchange / On-Behalf-Of                 │
│  产出: 链式身份上下文                                │
├─────────────────────────────────────────────────────┤
│  层 4: 委托授权 (Delegated Authorization)            │
│  Agent 能用用户身份做什么？                           │
│  机制: OAuth Scopes / 细粒度工具授权                  │
│  产出: Agent 的权限边界                              │
└─────────────────────────────────────────────────────┘
```

---

## 2. 主流协议详解

### 2.1 OAuth 2.0 + OIDC（现代标准）

OAuth 2.0 负责授权，OIDC (OpenID Connect) 在 OAuth 2.0 之上增加身份认证层。

```
OIDC 认证流程:

┌─────────┐          ┌──────────┐          ┌──────────┐
│  浏览器   │          │ Agent 平台 │          │    IdP    │
│   (用户)  │          │ (RP)      │          │  (OP)    │
└────┬─────┘          └─────┬────┘          └─────┬────┘
     │                      │                      │
     │  1. 访问 Agent 应用   │                      │
     │─────────────────────►│                      │
     │                      │                      │
     │  2. 重定向到 IdP      │                      │
     │◄─────────────────────│                      │
     │                      │                      │
     │  3. 用户认证          │                      │
     │────────────────────────────────────────────►│
     │                      │                      │
     │  4. 授权码返回        │                      │
     │◄────────────────────────────────────────────│
     │                      │                      │
     │  5. 授权码交换 Token  │                      │
     │                      │─────────────────────►│
     │                      │                      │
     │  6. ID Token +       │                      │
     │     Access Token     │                      │
     │                      │◄─────────────────────│
     │                      │                      │
     │  7. 验证 Token        │                      │
     │                      │                      │
     │  8. 创建会话          │                      │
     │◄─────────────────────│                      │
```

#### Token 类型

| Token | 用途 | 格式 | 生命周期 | 包含信息 |
|-------|------|------|----------|----------|
| ID Token | 用户身份认证 | JWT | 短（1h） | sub, name, email, tenant_id |
| Access Token | 访问资源授权 | JWT/不透明 | 短（1h） | scopes, roles, permissions |
| Refresh Token | 获取新 Access Token | 不透明 | 长（24h+） | 无（后端关联 session）|

#### Token 结构示例

```json
{
  "iss": "https://idp.company.com",
  "sub": "user_12345",
  "aud": "agent-platform",
  "exp": 1712345678,
  "iat": 1712342078,
  "name": "张三",
  "email": "zhangsan@company.com",
  "tenant_id": "tenant_finance_001",
  "roles": ["admin", "analyst"],
  "groups": ["finance-team", "data-access"],
  "permissions": [
    "agent:chat",
    "agent:tools:sql_query:read",
    "agent:tools:email:write"
  ]
}
```

### 2.2 SAML 2.0（企业遗留系统）

许多大型企业仍在使用 SAML 2.0，尤其是在与 Active Directory Federation Services (ADFS) 集成时。

```
SAML 与 OIDC 的关键区别:

SAML: XML 格式断言, 浏览器重定向流, SOAP 绑定
OIDC: JSON Token, REST API, 更轻量

选择建议:
  ┌─ 新系统 → OIDC（推荐）
  ├─ Azure AD/Okta → OIDC（都支持）
  ├─ ADFS 4.0+ → OIDC 也可用
  └─ 遗留 ADFS/ADFS 3.0 → SAML
```

### 2.3 LDAP / Active Directory（内部目录）

当 Agent 系统部署在企业内网时，可能需要直接与 LDAP 目录服务集成。

| 操作 | LDAP 协议 | 用途 |
|------|-----------|------|
| 认证 | BIND | 验证用户密码 |
| 用户搜索 | SEARCH | 按条件查找用户 |
| 组查询 | SEARCH (memberOf) | 获取用户所属组 |
| 属性读取 | SEARCH | 获取用户属性（邮箱、电话等） |
| 用户同步 | 定时导出/SCIM | 增量同步用户到 Agent 平台 |

---

## 3. 核心挑战：Agent 长时间运行 vs 短 Token

### 3.1 问题描述

Agent 的一个任务可能持续数分钟到数小时，而 Access Token 通常只有 1 小时有效期。

```
时间线:
├── 0:00 ── 用户登录，获取 Access Token (有效期 1h)
├── 0:01 ── 用户下达复杂任务："分析上季度所有客户的销售数据并发送报告"
├── 0:30 ── Agent 正在执行第 3 步（共 10 步）
├── 0:45 ── Agent 调用分析工具 ✅ (Token 有效)
├── 1:00 ── ❌ Token 过期！
├── 1:05 ── Agent 调用邮件工具 ❌ 失败
└── 任务中断
```

### 3.2 解决方案：Token 刷新 + 委托授权

#### 方案 A：Refresh Token 轮转（推荐）

```python
import time
import httpx
from dataclasses import dataclass

@dataclass
class TokenSet:
    access_token: str
    refresh_token: str
    expires_at: float
    token_type: str = "Bearer"

class TokenManager:
    """安全的 Token 管理（含自动刷新）"""
    
    def __init__(self, client_id: str, client_secret: str, token_url: str):
        self.client_id = client_id
        self.client_secret = client_secret
        self.token_url = token_url
        self._tokens: dict[str, TokenSet] = {}  # session_id -> TokenSet
    
    async def get_valid_token(self, session_id: str) -> str:
        """获取有效的 Access Token（自动刷新）"""
        token_set = self._tokens.get(session_id)
        if not token_set:
            raise ValueError("Session not found")
        
        # 检查是否需要刷新（提前 5 分钟刷新）
        if time.time() + 300 > token_set.expires_at:
            await self._refresh_token(session_id, token_set.refresh_token)
            token_set = self._tokens[session_id]
        
        return token_set.access_token
    
    async def _refresh_token(self, session_id: str, refresh_token: str):
        """使用 Refresh Token 获取新的 Access Token"""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.token_url,
                data={
                    "grant_type": "refresh_token",
                    "refresh_token": refresh_token,
                    "client_id": self.client_id,
                    "client_secret": self.client_secret,
                },
            )
        
        if response.status_code != 200:
            # Refresh Token 也过期了 → 需要用户重新登录
            del self._tokens[session_id]
            raise PermissionError("Session expired, user re-authentication required")
        
        data = response.json()
        self._tokens[session_id] = TokenSet(
            access_token=data["access_token"],
            refresh_token=data.get("refresh_token", refresh_token),  # Token Rotation
            expires_at=time.time() + data["expires_in"],
        )
```

#### 方案 B：OAuth 2.0 Token Exchange (RFC 8693)

```
Agent 使用 On-Behalf-Of (OBO) 流程:

用户 Token → Agent → IdP → 作用域受限的子 Token

             ┌───────────────────────┐
用户 Token   │  Token Exchange Request│  受限子 Token
──►───────── │  subject_token=...    │ ──►──────────
             │  requested_scopes=...  │
             │  actor_token=agent_svc │
             └───────────────────────┘
```

```python
class TokenExchangeService:
    """OAuth 2.0 Token Exchange 实现"""
    
    async def exchange_for_delegated_token(
        self,
        user_token: str,
        requested_scopes: list[str],
        agent_service_id: str,
    ) -> TokenSet:
        """用用户 Token 换取 Agent 可用的委派 Token"""
        
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.token_url,
                data={
                    "grant_type": "urn:ietf:params:oauth:grant-type:token-exchange",
                    "subject_token": user_token,
                    "subject_token_type": "urn:ietf:params:oauth:token-type:access_token",
                    "actor_token": self._get_service_token(agent_service_id),
                    "actor_token_type": "urn:ietf:params:oauth:token-type:access_token",
                    "scope": " ".join(requested_scopes),
                    "resource": self.agent_resource_id,
                },
            )
        
        data = response.json()
        return TokenSet(
            access_token=data["access_token"],
            refresh_token=data.get("refresh_token", ""),
            expires_at=time.time() + data["expires_in"],
        )
```

#### 方案 C：长任务 Token 策略

对于真正长时间的 Agent 任务（数小时）：

```python
class LongRunningTaskTokenStrategy:
    """
    长任务 Token 策略：
    1. 任务启动时获取短期用户 Token
    2. 换取长期服务 Token（仅限特定作用域）
    3. 使用服务 Token 执行任务
    4. 审计日志记录用户原始授权
    
    注意：此策略需要仔细评估安全影响
    """
    
    async def start_agent_task(
        self, user_token: str, task_config: dict
    ) -> str:
        """启动长任务并创建任务级 Token"""
        
        # 1. 验证用户权限
        user_info = await self.verify_token(user_token)
        
        # 2. 创建任务级授权（记录到审计日志）
        task_id = str(uuid.uuid4())
        task_scope = self._compute_task_scope(task_config)
        
        await self.audit_log.log(
            event="task_authorized",
            user_id=user_info["sub"],
            task_id=task_id,
            scope=task_scope,
            timestamp=datetime.utcnow(),
        )
        
        # 3. 签发任务级服务 Token（有效期 = 任务最大时长）
        task_token = self.issue_service_token(
            service_id="agent-long-running",
            scope=task_scope,
            expiry=timedelta(hours=24),
            audit_context={"original_user": user_info["sub"], "task_id": task_id},
        )
        
        return task_token
```

### 3.3 推荐策略组合

| 场景 | Token 策略 | 安全等级 |
|------|-----------|----------|
| 普通对话（< 1h） | 直接使用用户 Access Token | ★★★★★ |
| 多步任务（1-4h） | Refresh Token 轮转 | ★★★★☆ |
| 长时任务（4h+） | Token Exchange + 服务 Token | ★★★☆☆ |
| 定时任务/批处理 | 独立服务账号（非用户委托） | ★★★★★ |

---

## 4. 身份传播链路

Agent 系统的最大特殊性在于：**用户的身份需要在 LLM、工具、知识库、缓存、日志每一层都保持可追溯**。

```python
from dataclasses import dataclass, field
import uuid

@dataclass
class PropagatedIdentity:
    """全链路传播的身份信息"""
    
    # 原始用户
    user_id: str
    user_name: str
    user_email: str
    
    # 租户
    tenant_id: str
    tenant_name: str
    
    # 角色与权限
    roles: list[str] = field(default_factory=list)
    permissions: list[str] = field(default_factory=list)
    
    # 会话跟踪
    session_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    request_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    
    # 委托信息
    delegated_by: str = ""  # 谁委托了这次操作
    delegation_reason: str = ""
    
    def to_llm_context(self) -> str:
        """注入到 LLM System Prompt 的身份信息"""
        return (
            f"当前用户信息:\n"
            f"  用户: {self.user_name} ({self.user_email})\n"
            f"  部门: {self.tenant_name}\n"
            f"  角色: {', '.join(self.roles)}\n"
            f"  操作权限: {', '.join(self.permissions)}\n"
        )
    
    def to_header(self) -> dict:
        """传递给下游服务的 HTTP Header"""
        return {
            "X-User-ID": self.user_id,
            "X-Tenant-ID": self.tenant_id,
            "X-User-Roles": ",".join(self.roles),
            "X-Request-ID": self.request_id,
            "X-Session-ID": self.session_id,
        }

class IdentityPropagator:
    """身份传播中间件——确保身份在 Agent 全链路中一致"""
    
    def __init__(self):
        self._context_var = contextvars.ContextVar("identity")
    
    def set_identity(self, identity: PropagatedIdentity):
        """设置当前请求的身份"""
        self._context_var.set(identity)
    
    def get_identity(self) -> PropagatedIdentity:
        """获取当前身份"""
        return self._context_var.get()
    
    async def propagate_to_tool_call(
        self, tool_name: str, params: dict
    ) -> tuple[dict, dict]:
        """在工具调用中传播身份"""
        identity = self.get_identity()
        
        # 注入身份头
        headers = identity.to_header()
        
        # 审计日志
        await self.audit_log.log({
            "event": "tool_call",
            "tool_name": tool_name,
            "user_id": identity.user_id,
            "tenant_id": identity.tenant_id,
            "request_id": identity.request_id,
            "params": self._mask_sensitive_params(params),
        })
        
        return params, headers
```

---

## 5. 工具调用中的身份传递

Agent 调用外部工具时，身份信息必须传递到被调用的服务。

```python
class AuthenticatedToolClient:
    """携带用户身份的 HTTP 工具客户端"""
    
    def __init__(self, identity_propagator: IdentityPropagator):
        self.identity_propagator = identity_propagator
    
    async def call_tool(self, tool_config: dict, params: dict) -> dict:
        """调用外部工具（携带身份上下文）"""
        
        identity = self.identity_propagator.get_identity()
        
        # 1. Token Exchange：获取作用域受限的子 Token
        scoped_token = await self.exchange_token(
            user_token=identity.access_token,  # 原始用户 Token
            tool_name=tool_config["name"],
            scopes=tool_config.get("required_scopes", []),
        )
        
        # 2. 使用子 Token 调用工具
        async with httpx.AsyncClient() as client:
            response = await client.post(
                tool_config["endpoint"],
                json=params,
                headers={
                    "Authorization": f"Bearer {scoped_token}",
                    "X-Request-ID": identity.request_id,
                },
            )
        
        # 3. 记录审计
        await self.audit_log.log({
            "event": "tool_execution",
            "tool": tool_config["name"],
            "user": identity.user_id,
            "request_id": identity.request_id,
            "status": response.status_code,
        })
        
        return response.json()
```

---

## 6. Agent 特有的身份场景

### 6.1 多用户协作

多个用户在同一 Agent 会话中的身份处理：

```python
@dataclass
class MultiUserSession:
    """多用户协作会话"""
    session_id: str
    participants: list[PropagatedIdentity]
    primary_user: PropagatedIdentity  # 当前操作者
    
    def get_combined_permissions(self) -> set[str]:
        """所有参与者的权限并集"""
        perms = set()
        for p in self.participants:
            perms.update(p.permissions)
        return perms
    
    def user_can(self, user_id: str, permission: str) -> bool:
        """检查特定用户是否有权限"""
        user = next((p for p in self.participants if p.user_id == user_id), None)
        return user and permission in user.permissions
```

### 6.2 Agent-to-Agent 身份传递

多 Agent 协作时，A 调用 B 需要传递用户身份：

```python
class CrossAgentIdentity:
    """跨 Agent 身份包装器"""
    
    def to_message(self, identity: PropagatedIdentity) -> dict:
        """将用户身份编码到 Agent 间消息中"""
        return {
            "type": "identity_context",
            "payload": {
                "original_user_id": identity.user_id,
                "original_tenant_id": identity.tenant_id,
                "delegation_chain": identity.delegation_chain + [identity.user_id],
                "session_id": identity.session_id,
                "permissions": identity.permissions,
            }
        }
```

---

## 7. 能力边界

### 7.1 能做到的

- **统一认证入口**：所有 Agent 功能通过企业 IdP 认证
- **角色/权限映射**：企业角色 → Agent 工具权限的自动映射
- **身份可追溯**：每个操作都能追溯到原始用户
- **会话生命周期管理**：登录、登出、超时、并发控制
- **SCIM 用户同步**：IdP 中用户变更自动同步到 Agent 平台

### 7.2 做不到的

- **解决 Agent 误判**：身份正确 ≠ Agent 决策正确
- **覆盖最终用户授权**：Agent 使用工具时，原始用户可能不在现场确认
- **跨 IdP 联盟**：不同企业的 IdP 之间需要额外适配
- **完全离线认证**：SSO 依赖 IdP 在线，离线场景需要其他方案

---

## 8. 核心优势

- **降低摩擦**：用户无需额外注册，用企业账号直接登录
- **统一治理**：身份变更（入职/离职/转岗）在 IdP 操作，Agent 平台自动同步
- **合规满足**：满足 SOC 2、SOX 等合规要求的身份管理
- **审计追溯**：每个 Agent 操作都与具体用户关联

---

## 9. 工程优化方向

### 9.1 减少 Token 刷新延迟

```python
class OptimizedTokenStrategy:
    """预刷新 + 批量刷新优化"""
    
    async def pre_refresh_tokens(self):
        """在 Token 过期前主动刷新"""
        now = time.time()
        for session_id, token_set in self._tokens.items():
            if token_set.expires_at - now < 600:  # 剩余 < 10 分钟
                await self._refresh_background(session_id, token_set.refresh_token)
```

### 9.2 缓存 Token 减少 IdP 压力

```python
class TokenCache:
    """Token 验证结果缓存"""
    
    def __init__(self, redis_client):
        self.redis = redis_client
        self.cache_ttl = 300  # 5 分钟缓存
    
    async def verify_and_cache(self, token: str) -> dict:
        # 1. 尝试从缓存获取
        cache_key = f"token_verify:{hashlib.sha256(token.encode()).hexdigest()}"
        cached = await self.redis.get(cache_key)
        if cached:
            return json.loads(cached)
        
        # 2. 调用 IdP 验证
        result = await self.idp.verify_token(token)
        
        # 3. 写入缓存（不超过 Token 本身有效期）
        ttl = min(self.cache_ttl, result["exp"] - time.time())
        await self.redis.setex(cache_key, int(ttl), json.dumps(result))
        
        return result
```

### 9.3 离线降级

```python
class DegradedAuthMode:
    """IdP 不可用时的降级认证"""
    
    async def authenticate_offline(self, request) -> bool:
        """当 IdP 不可用时的降级策略"""
        
        if not self._is_idp_available():
            # 1. 检查是否有本地缓存的会话
            session = await self._get_cached_session(request)
            if session and not session.is_expired():
                return True
            
            # 2. 短时降级：允许已有会话继续
            if self._within_degraded_window():
                return True
            
            # 3. 长时降级：需要人工审批
            await self._notify_admin("IdP 离线超过阈值，需要手动授权")
            return False
        
        return await self._normal_auth(request)
```

---

## 10. 常见失败模式

| 失败模式 | 表现 | 根因 | 解决方案 |
|----------|------|------|----------|
| Token 过期导致任务中断 | 长时间 Agent 任务中途失败 | 未实现 Token 刷新 | Refresh Token + 自动刷新 |
| 身份泄漏 | 工具日志中记录了密码 | 工具参数未脱敏 | 参数自动脱敏 + 审计过滤 |
| 权限放大 | Agent 使用了不应有的权限 | Token 作用域过大 | 最小权限 + Token Exchange |
| 身份混淆 | 多 Agent 协作时用了错误的身份 | 身份传播链路断裂 | 强制身份上下文传递 |
| 会话固定 | 攻击者复用他人会话 | 缺少会话绑定 | IP/Device 指纹绑定 |

---

> **核心结论**：Agent 系统的 SSO 集成比传统 Web 应用复杂得多——关键在于解决"短 Token vs 长任务"的矛盾，并确保用户在 Agent 全链路中的可追溯性。推荐从 OIDC 开始，采用 Refresh Token 轮转 + Token Exchange 的组合策略，逐步建立完整的身份传播体系。
