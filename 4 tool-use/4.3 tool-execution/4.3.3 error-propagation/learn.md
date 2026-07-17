# 错误传播与优雅降级

## 简单介绍

工具执行过程中可能发生各种错误。如果原始错误信息直接传给 LLM，LLM 可能产生错误的理解和决策。错误传播和优雅降级机制确保了错误的"翻译"——将技术性错误转化为 LLM 可理解、可处理的结构化描述。

## 错误分类

```
工具执行错误
├── 输入错误（LLM 侧的错）
│   ├── 参数名错误
│   ├── 参数类型错误
│   ├── 参数值越界
│   └── 必选参数缺失
├── 执行错误（工具侧的错）
│   ├── 网络超时/断连
│   ├── 上游 API 错误
│   ├── 权限不足
│   └── 资源不存在
└── 系统错误（Agent 框架侧的错）
    ├── 工具未注册
    ├── 执行超时
    └── 沙箱异常
```

## 错误处理策略

### 1. 错误标准化

```python
class ToolResult:
    """统一的工具执行结果"""
    def __init__(self, success: bool, data=None, error: ToolError = None):
        self.success = success
        self.data = data
        self.error = error
    
    def to_llm_message(self) -> str:
        """转换为 LLM 可读的消息"""
        if self.success:
            return json.dumps(self.data, ensure_ascii=False)
        else:
            return f"[错误] {self.error.message}\n[建议] {self.error.suggestion}"
```

### 2. 按错误类型处理

| 错误类型 | 处理方式 | 示例消息 |
|---------|----------|----------|
| 参数错误 | 明确指出错误参数和正确格式 | "参数 'date' 不是合法格式，正确格式为 YYYY-MM-DD" |
| 权限不足 | 说明原因并提供替代方案 | "你没有删除该资源的权限，可以尝试联系管理员" |
| 资源不存在 | 确认资源 ID 并建议检查 | "用户 ID 12345 不存在，请确认后再查询" |
| 网络超时 | 提示暂时不可用 | "服务暂时无响应，请稍后重试" |
| 工具未注册 | 内部错误 | "[系统错误] 工具不可用，请尝试其他操作" |

### 3. 优雅降级路径

```
成功的工具调用 → 正常返回结果
     ↓ 失败
第一次重试（相同参数） → 成功则继续
     ↓ 失败
第二次重试（简化参数） → 成功则继续
     ↓ 失败
降级方案（替代工具） → 成功则继续
     ↓ 失败
返回人类可理解的错误 → LLM 调整策略
```

### 4. LLM 可理解的错误结构

```json
{
  "error": {
    "type": "validation_error",
    "message": "参数 'priority' 的值 'urgent' 不在允许范围内",
    "allowed_values": ["low", "normal", "high"],
    "suggestion": "请使用 low / normal / high 中的一种"
  }
}
```

## 工程实现

```python
class ToolExecutor:
    ERROR_SUGGESTIONS = {
        "validation": "请检查参数是否在允许范围内",
        "auth": "你可能没有执行此操作的权限",
        "not_found": "请确认资源 ID 是否正确",
        "timeout": "服务暂时不可用，请稍后重试",
    }
    
    async def execute(self, tool_call, context):
        try:
            args = json.loads(tool_call.function.arguments)
            result = await self._run(tool_call.function.name, args)
            return ToolResult(success=True, data=result)
        except ValidationError as e:
            return ToolResult(success=False, error=ToolError(
                type="validation", message=str(e),
                suggestion=self.ERROR_SUGGESTIONS["validation"]
            ))
        except TimeoutError:
            return ToolResult(success=False, error=ToolError(
                type="timeout", message="请求超时",
                suggestion=self.ERROR_SUGGESTIONS["timeout"]
            ))
        # ... 其他错误类型
```

## 核心原则

1. **错误要对 LLM "友好"**：错误消息应该指导 LLM 如何修正，而非指责
2. **区分瞬时和永久错误**：瞬时错误自动重试，永久错误返回给 LLM 决策
3. **不要泄露敏感信息**：错误消息中不要包含凭据、内部 IP 等敏感数据
4. **保持错误结构一致**：统一的错误格式让 LLM 更容易理解处理
