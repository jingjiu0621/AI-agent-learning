# 执行上下文：状态共享与隔离

## 简单介绍

工具执行过程中，多个工具可能需要共享某些状态（如当前用户身份、请求 ID、认证 Token），同时又不互相干扰。执行上下文（Execution Context）解决了"共享"与"隔离"这对矛盾——让需要共享的（用户信息、配置）可以被访问，同时保证各工具执行互不干扰。

## 什么是执行上下文

执行上下文是一个贯穿整个 Agent 执行周期的数据容器，包含了：

```python
@dataclass
class ExecutionContext:
    # 请求级别的上下文（只读）
    request_id: str
    user_id: str
    user_role: str
    
    # 会话级别的上下文（可读写）
    session_id: str
    session_data: dict
    
    # 认证信息
    auth_token: str
    api_keys: dict
    
    # 环境配置
    config: dict  # 超时、重试等配置
    
    # 追踪信息
    trace_id: str
    parent_span_id: str
```

## 上下文传递方式

### 1. 显式传递（推荐）

```python
async def handle_get_weather(location: str, _context: ExecutionContext):
    log.info(f"用户 {_context.user_id} 查询天气")
    api_key = _context.api_keys["weather"]
    return await call_weather_api(location, api_key)
```

### 2. 隐式传递（ContextVar）

```python
import contextvars

current_context = contextvars.ContextVar("execution_context")

async def execute_with_context(handler, args, context):
    token = current_context.set(context)
    try:
        return await handler(**args)
    finally:
        current_context.reset(token)

# 工具内部无需显式传参即可访问上下文
def get_current_user():
    ctx = current_context.get()
    return ctx.user_id
```

### 3. 依赖注入

```python
# 工具声明时需要上下文参数
tools = [
    ToolDef(
        name="get_weather",
        handler=get_weather,
        inject_context=True,  # 框架自动注入上下文
    )
]
```

## 隔离级别

| 级别 | 说明 | 实现方式 | 适用场景 |
|------|------|----------|----------|
| 请求级 | 每次请求独立上下文 | 每次请求创建新 context | API 调用 |
| 会话级 | 同一会话共享上下文 | Redis 存储会话 context | 多轮对话 |
| 用户级 | 同一用户共享 | 数据库持久化 | 长期状态 |
| 工具级 | 工具之间隔离 | 子 context | 安全敏感工具 |

## 上下文常见内容

```
├── 身份信息
│   ├── user_id / username
│   ├── roles / permissions
│   └── auth_token
├── 追踪信息
│   ├── trace_id / span_id
│   └── request_start_time
├── 配置信息
│   ├── environment (dev/staging/prod)
│   ├── region
│   └── feature_flags
└── 会话状态
    ├── conversation_id
    ├── previous_tool_results
    └── accumulated_context
```

## 上下文安全问题

1. **上下文污染**：一个工具修改了上下文的共享状态，影响其他工具——优先使用不可变上下文
2. **上下文泄漏**：工具不应该访问的敏感数据（如其他用户的 Token）被误传
3. **上下文过大**：过多的上下文数据导致 Token 浪费——按需注入

## 最佳实践

- 默认使用**不可变上下文**，只在明确需要时才允许修改
- 上下文中的敏感数据（密码、Token）使用引用而非值传递
- 为每个工具声明"我需要的上下文字段"——按需注入而非全量注入
- 异步环境中使用 ContextVar 确保线程/协程安全
