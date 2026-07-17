# Claude Tool Use 协议详解

## 简单介绍

Anthropic 在 2023 年末为 Claude 引入 Tool Use 能力，Claude 3 系列模型（Sonnet、Opus、Haiku）提供完整的工具调用支持。与 OpenAI 的 FC 相比，Claude 的工具使用协议在设计理念和细节上有显著差异——尤其是在思考过程可见性、工具结果注入方式等方面。

## 基本原理

### API 格式

```
请求：
{
  "model": "claude-3-5-sonnet-20241022",
  "system": "你是一个天气助手...",
  "messages": [...],
  "tools": [
    {
      "name": "get_weather",
      "description": "获取指定城市的天气",
      "input_schema": {
        "type": "object",
        "properties": {
          "location": { "type": "string" }
        },
        "required": ["location"]
      }
    }
  ]
}

响应（流式）：
{
  "type": "content_block_start",
  "content_block": { "type": "tool_use", "id": "toolu_xxx", "name": "get_weather" }
}
{
  "type": "content_block_delta",
  "delta": { "type": "input_json_delta", "partial_json": "{\"location\":\"北京\"}" }
}
```

### 非流式响应结构

```json
{
  "content": [
    { "type": "text", "text": "我来查北京的天气" },
    {
      "type": "tool_use",
      "id": "toolu_xxx",
      "name": "get_weather",
      "input": { "location": "北京" }
    }
  ]
}
```

### 工具结果注入

将结果以 `tool_result` 类型的 content block 传回：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_xxx",
      "content": "{\"temp\": 28, \"condition\": \"晴\"}"
    }
  ]
}
```

## Claude 与 OpenAI 的关键区别

| 维度 | OpenAI FC | Claude Tool Use |
|------|-----------|-----------------|
| 思考可见性 | tool_calls 不包含推理过程 | tool_use 和 text（思考）可以共存 |
| 参数格式 | JSON Schema 在 function.parameters 中 | 直接在 input_schema 中 |
| 结果注入 | tool 角色 message | tool_result 类型 content block |
| 流式调用 | 一次完成（多个 tool_calls 同时返回） | JSON delta 流式输出参数 |
| 工具定义位置 | 请求级参数 | 请求级 tools 参数 |
| 并行调用 | 原生支持（since gpt-4-1106） | 模型自主决定，推荐同时提供 |

## 核心优势

1. **思考过程可见**：Claude 可以在同一响应中既输出思考文本（text block）又输出工具调用（tool_use block），开发者能看到模型的推理过程
2. **流式 JSON**：工具参数以 `input_json_delta` 流式输出，用户体验更好（参数逐步呈现）
3. **内容块模型**：响应由独立的 content block 组成，每种类型有清晰的 schema，处理逻辑更干净

## 局限

1. **Stream 处理复杂**：需要处理多种 event 类型（content_block_start, content_block_delta, content_block_stop）
2. **工具数量限制**：单次请求注入大量工具定义时表现下降
3. **JSON 流式问题**：`input_json_delta` 是 JSON 片段，需要客户端累积拼接

## 最佳实践

- 使用 system prompt 中的指令帮助 Claude 决定何时调用工具（比 OpenAI 更依赖 prompt 质量）
- 对于复杂任务，让 Claude 先用 text 思考再输出 tool_use
- 工具结果需要包含足够信息，帮助 Claude 判断下一步行动
