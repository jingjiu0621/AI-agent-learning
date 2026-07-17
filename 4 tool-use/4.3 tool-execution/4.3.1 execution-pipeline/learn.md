# 执行管道架构设计

## 简单介绍

执行管道（Execution Pipeline）是接收 LLM 输出的工具调用声明、将其转换为实际函数执行、并返回结果的中间层架构。一个好的管道设计决定了工具执行的可靠性、可扩展性和可观测性。

## 管道架构

```
LLM 输出 tool_call
      ↓
  ┌──────────────────┐
  │  1. 解析层        │  解析 JSON arguments，校验类型
  │  2. 路由层        │  根据 name 找到对应的 handler
  │  3. 前置处理层     │  权限检查、参数转换、上下文注入
  │  4. 执行层        │  实际运行 handler
  │  5. 后置处理层     │  结果格式化、截断、缓存
  │  6. 错误处理层     │  异常捕获、重试、降级
  └──────────────────┘
      ↓
  注入结果到下一轮 LLM 调用
```

## 各层详解

### 1. 解析层

```python
def parse_tool_call(tool_call) -> ParsedCall:
    name = tool_call.function.name
    try:
        args = json.loads(tool_call.function.arguments)
    except json.JSONDecodeError:
        args = repair_json(tool_call.function.arguments)  # 容错解析
    return ParsedCall(name=name, args=args)
```

关键点：LLM 可能输出不合法 JSON——需要有容错解析机制。

### 2. 路由层

```python
# 注册中心模式
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, ToolHandler] = {}
    
    def register(self, name: str, handler: ToolHandler):
        self._tools[name] = handler
    
    def resolve(self, name: str) -> ToolHandler:
        if name not in self._tools:
            raise ToolNotFoundError(f"未知工具: {name}")
        return self._tools[name]
```

### 3. 前置处理层

横切关注点（Cross-cutting Concerns）的统一处理：

- **权限校验**：当前用户有权调用此工具吗？
- **参数填充**：从上下文自动填充某些参数（如当前用户 ID）
- **限流检查**：该工具的调用频率是否超过限制
- **审计日志**：记录谁在何时调用了什么工具

### 4. 执行层

```python
async def execute_handler(handler, args, context):
    # context 包含：当前用户、会话状态、请求 ID 等
    return await handler.execute(**args, _context=context)
```

### 5. 后置处理层

```python
def post_process(result: Any, max_tokens: int = 2000) -> str:
    # 限制结果大小
    text = json.dumps(result, ensure_ascii=False)
    if len(text) > max_tokens * 4:  # 粗略 Token 估算
        text = truncate_or_summarize(text, max_tokens)
    return text
```

### 6. 错误处理层

统一的异常处理策略，将技术错误转化为 LLM 可理解的错误信息。

## 设计模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| 管道模式 | 顺序执行各个处理阶段 | 通用工具执行 |
| 中间件模式 | 可插拔的横切关注点 | 需要灵活扩展 |
| 装饰器模式 | 用注解声明工具属性 | 开发友好 |
| 注册中心模式 | 中心化工具管理 | 大型系统 |

## 核心挑战

1. **性能开销**：每个阶段都有延迟，管道长了影响响应速度
2. **状态传递**：上下文信息如何在各层间传递？
3. **错误隔离**：一个工具的问题不应影响管道本身
4. **可测试性**：管道各阶段需要能独立测试

## 工程启示

- 管道设计初期就考虑可观测性（每阶段的耗时、错误率都要有指标）
- 不要在管道中执行耗时操作（超过 5s 的任务应考虑异步执行）
- 保持管道各阶段的"单一职责"——不要把一个阶段塞太多逻辑
