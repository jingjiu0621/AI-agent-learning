# 函数调用生命周期

## 简单介绍

一次函数调用从开始到结束经历六个阶段：声明 → 触发 → 生成 → 解析 → 执行 → 注入。理解这个生命周期是设计可靠 Agent 系统的基础——不同阶段的失败需要不同的处理策略。

## 生命周期图示

```
声明 ──→ 触发 ──→ 生成 ──→ 解析 ──→ 执行 ──→ 注入
 │         │         │         │         │         │
 定义工具   模型决定   输出/流   类型转换   运行逻辑   结果回传
 Schema    调用工具   tool_call  参数校验  业务代码   LLM 推理
```

## 各阶段详解

### 1. 声明阶段

开发者将工具定义传给 API。关键行为：
- 定义 tool name, description, parameters (JSON Schema)
- 决定 token 预算：工具定义消耗的 Token 计入输入 Token

**常见问题**：描述不清晰导致 LLM 误选工具；Schema 太复杂消耗过多 Token

### 2. 触发阶段

LLM 分析对话历史 + 工具定义，决定是否调用工具以及调用哪个工具。

**影响因素**：
- 工具描述与用户意图的匹配度
- 模型版本（更聪明的模型选择更准）
- System prompt 中关于工具使用的指令

**常见问题**：模型应该调用工具但选择了直接回答；模型在不需要工具时调用了工具

### 3. 生成阶段

LLM 输出结构化的工具调用声明，包含工具名和参数。

**生成形式**：
- **非流式**：在一次响应中输出完整的 tool_calls（OpenAI）或 content blocks（Claude）
- **流式式**：逐步输出参数片段，需要客户端拼接

**常见问题**：
- 参数名幻觉（生成不存在的参数）
- 参数值超出 enum 范围
- 工具名拼写错误

### 4. 解析阶段

客户端接收 LLM 输出的 tool_call，解析参数并校验类型。

```
接收: {"name":"get_weather","arguments":"{\"location\":\"北京\"}"}
解析:
  1. 解析 JSON arguments → dict
  2. 校验参数名称是否合法
  3. 校验参数类型是否正确
  4. 校验必需参数是否齐全
```

### 5. 执行阶段

实际运行工具对应的函数逻辑。

**可能结果**：
| 结果 | 原因 | 处理方式 |
|------|------|----------|
| 成功 | 函数正常返回 | 格式化结果，注入下一轮 |
| 终端错误 | 参数无效、权限不足 | 返回错误描述，让 LLM 修正 |
| 瞬时错误 | 网络超时、限流 | 自动重试 |
| 部分结果 | 流式、长任务 | 返回中间状态 |

### 6. 注入阶段

将工具执行结果注入回对话上下文，供 LLM 继续推理。

**注入形式**：
- OpenAI: `{"role": "tool", "tool_call_id": "...", "content": "..."}`
- Claude: `{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "...", "content": "..."}]}`

**关键考量**：
- 结果是否要截断？大结果包含太多 Token
- 结果是否要摘要？帮助 LLM 更快理解
- 结果格式是否要规范化？统一结构让解析更可靠

## 生命周期中的失败点

| 阶段 | 失败模式 | 处理策略 |
|------|---------|----------|
| 声明 | Schema 错误 | 测试验证工具定义 |
| 触发 | 工具选择错误 | 优化描述 + 强制调用 |
| 生成 | 参数不合法 | 后验校验 + 重新生成 |
| 解析 | 格式错误 | 容错解析 + 回退 |
| 执行 | 运行时错误 | 重试 + 降级 |
| 注入 | 结果超长 | 截断 + 摘要 |

## 工程实现最小示例

```python
# 简化的生命周期实现
class FCLifecycle:
    def __init__(self, tools: list):
        self.tools = {t["name"]: t for t in tools}  # 1. 声明
    
    async def execute(self, user_message: str):
        response = await llm.chat(
            messages=[user_message],
            tools=list(self.tools.values())  # 2. 触发 + 3. 生成
        )
        for tool_call in response.tool_calls:
            tool = self.tools[tool_call.function.name]
            try:
                args = json.loads(tool_call.function.arguments)  # 4. 解析
                result = await tool["handler"](**args)  # 5. 执行
            except Exception as e:
                result = {"error": str(e)}
            # 6. 注入（下一轮消息会包含 tool_result）
        return result
```
