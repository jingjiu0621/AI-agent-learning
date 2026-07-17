# 工具认证与授权集成

## 简单介绍

许多工具需要认证（你是谁）和授权（你能做什么）才能使用。将认证信息集成到工具调用中需要精心设计——既要让 LLM 能够方便地调用需要认证的工具，又要确保安全凭证不会被泄露或滥用。

## 认证模型

```
Agent 认证
├── 静态认证（工具级别）
│   ├── 内置 API Key（服务账号）
│   └── 环境变量凭据
├── 用户级认证（用户级别）
│   ├── OAuth 2.0 令牌
│   ├── Session Cookie
│   └── JWT Token
└── 动态认证（请求级别）
    ├── 按需刷新 Token
    └── 临时凭证
```

## 认证注入方案

### 方案 1：Wrapper 层自动注入（推荐）

```python
class AuthenticatedTool:
    def __init__(self, auth_provider: AuthProvider):
        self.auth = auth_provider
    
    async def execute(self, **kwargs):
        token = await self.auth.get_valid_token()
        # Token 自动注入到请求中，对 LLM 透明
        headers = {"Authorization": f"Bearer {token}"}
        return await self._call_api(headers=headers, **kwargs)
```

**优点**：LLM 不需要感知认证，减少出错
**缺点**：无法处理用户级权限差异

### 方案 2：参数传递认证

```json
{
  "name": "send_email",
  "parameters": {
    "properties": {
      "to": {"type": "string"},
      "subject": {"type": "string"},
      // 没有 auth 参数——框架自动处理
    }
  }
}
```

### 方案 3：上下文感知认证

```python
class ContextAwareAuth:
    async def get_token(self, ctx: ExecutionContext) -> str:
        """根据上下文获取合适级别的 Token"""
        if ctx.user_id:
            # 用户级 Token——权限最小
            return await self.user_token_service.get(ctx.user_id)
        elif ctx.service_account:
            # 服务账号 Token——权限较大
            return await self.service_account_token_service.get()
        else:
            # 匿名 Token——权限最小
            return await self.anonymous_token_service.get()
```

## OAuth 2.0 集成

```python
class OAuthToolWrapper:
    """OAuth 认证的工具封装"""
    
    def __init__(self, client_id: str, client_secret: str, 
                 scopes: list[str], redirect_uri: str):
        self.client_id = client_id
        self.client_secret = client_secret
        self.scopes = scopes
        self.redirect_uri = redirect_uri
    
    async def ensure_token(self, user_id: str) -> str:
        """确保有有效的 Token，必要时刷新"""
        token = await self.storage.get_token(user_id)
        if not token:
            raise AuthRequiredError(
                f"用户 {user_id} 需要授权才能使用此工具",
                auth_url=self._build_auth_url()
            )
        if token.is_expired():
            token = await self._refresh(token)
            await self.storage.save_token(user_id, token)
        return token.access_token
```

## 安全最佳实践

| 实践 | 说明 |
|------|------|
| 不暴露凭据给 LLM | Token、API Key 不应出现在 LLM 能看到的任何消息中 |
| 最小权限原则 | 每个工具只申请它需要的最小权限范围 |
| Token 有效期短 | 短期 Token 减少泄露风险 |
| 用户确认 | 高权限操作（删除、修改）需要用户二次确认 |
| 审计日志 | 记录每次工具调用的用户身份、操作内容和时间 |

## 核心挑战

1. **Token 刷新**：用户 Token 可能在 Agent 多轮交互中过期——需要透明刷新
2. **多用户共享**：多租户系统中，不同用户的 Token 不同，不能混用
3. **权限错误处理**：权限不足时，错误消息需要指导用户如何获取权限
4. **服务间认证**：Agent 后端到工具服务的 mTLS 或服务账号认证

## 工程启示

- 认证集成的最好方式是对 LLM "完全透明"——LLM 不需要知道认证的存在
- 权限不足的错误应包含"如何解决"的指导（"请联系管理员开通 XX 权限"）
- 不要在工具定义中包含任何与认证相关的参数——那是 Wrapper 层的职责
