# 工具封装模式（Wrapper 层设计）

## 简单介绍

工具封装（Tool Wrapper）是将原始函数、API、服务接口包装为标准化工具接口的设计模式。Wrapper 层隔离了"原始实现"与"Agent 可消费的接口"，使得底层改动不会影响 Agent 的工具使用。

## Wrapper 层职责

```
Agent / LLM
    ↓ 标准化的 Tool Call
Wrapper 层
    ├── 参数转换（Agent 格式 → 实现格式）
    ├── 认证注入（自动附加 Token / API Key）
    ├── 错误翻译（技术错误 → 友好错误）
    ├── 结果格式化（原始响应 → 统一格式）
    ├── 上下文注入（用户 ID、追踪 ID 等）
    └── 横切关注点（日志、监控、限流）
        ↓ 标准化调用
具体实现（数据库 / API / 文件系统）
```

## 设计模式

### 1. 函数式 Wrapper

```python
# 原始函数
def get_weather_raw(city: str, api_key: str) -> dict:
    """原始 API 调用，需要 api_key"""
    return requests.get(f"https://api.weather.com?city={city}&key={api_key}")

# Wrapper
@tool_wrapper
async def get_weather(location: str, unit: str = "celsius") -> dict:
    """获取指定城市的天气。参数: location-城市名, unit-温度单位"""
    api_key = get_context().api_keys["weather"]  # 自动注入
    raw = get_weather_raw(location, api_key)
    return {
        "temperature": raw["temp"],
        "condition": raw["weather"],
        "humidity": raw["humidity"]
    }
```

### 2. 类式 Wrapper

```python
class WeatherTool(ToolBase):
    """基于类的工具封装，更复杂的状态管理"""
    
    name = "get_weather"
    description = "获取指定城市的天气"
    
    parameters = {
        "type": "object",
        "properties": {
            "location": {"type": "string", "description": "城市名"},
            "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
        }
    }
    
    async def execute(self, location: str, unit: str = "celsius"):
        # 认证注入
        token = await self.auth.get_token()
        try:
            data = await self.api.get_weather(location, token)
            return self.format(data, unit)
        except ApiError as e:
            return self.handle_error(e)
```

### 3. 装饰器式 Wrapper

```python
@register_tool(
    name="get_weather",
    category="查询",
    timeout=10.0,
    retries=2
)
async def get_weather(location: str):
    """获取指定城市的天气"""
    ...
```

## Wrapper 层需要考虑的横切关注点

| 关注点 | 实现方式 | 示例 |
|--------|---------|------|
| 认证 | 自动从上下文获取 | api_key = ctx.api_keys["weather"] |
| 限流 | 装饰器/中间件 | @rate_limit(100/min) |
| 日志 | 自动记录入参出参 | log.info(f"call {name} with {args}") |
| 追踪 | OpenTelemetry span | with tracer.start_span(name) |
| 缓存 | 返回结果缓存 | cache.get(key) or cache.set(key) |
| 重试 | 自动重试瞬时错误 | @retry(max_attempts=3) |

## 核心挑战

1. **参数映射复杂**：原始 API 的参数形式可能和 LLM 能理解的形式完全不同（如原始 API 需要 user_id=123，但 LLM 只知道 username="张三"）
2. **错误语义转换**：原始 500 错误 → "服务暂时不可用"，需要精心设计错误消息
3. **保持灵活性**：Wrapper 层不应该过于抽象，限制底层工具的能力表达

## 最佳实践

- Wrapper 层坚持"薄"的原则——只做转换和适配，不添加业务逻辑
- 每个 Wrapper 有清晰的"输入契约"和"输出契约"——定义好输入输出格式
- Wrapper 的错误消息应该让 LLM "能修正"——而不是"你错了"而是"参数 X 应该这样传"
