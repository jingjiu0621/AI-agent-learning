# OpenAI Function Calling 协议详解

## 简单介绍

OpenAI 在 2023 年 6 月发布的 GPT-4-0613 中首次引入了 Function Calling 能力。这是业界第一个广泛应用于生产环境的工具调用协议，定义了 LLM 如何声明需要调用的函数以及参数。此后，几乎所有主流 LLM 提供商（Anthropic、Google、Mistral 等）都推出了类似的协议。

## 基本原理

### API 格式

OpenAI FC 的核心是 messages 数组中的 `tool_calls` 字段：

```
请求：
{
  "model": "gpt-4",
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "获取指定城市的天气",
        "parameters": {
          "type": "object",
          "properties": {
            "location": { "type": "string", "description": "城市名" },
            "unit": { "type": "string", "enum": ["celsius", "fahrenheit"] }
          },
          "required": ["location"]
        }
      }
    }
  ],
  "tool_choice": "auto"
}

响应：
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [{
        "id": "call_xxx",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"location\":\"北京\",\"unit\":\"celsius\"}"
        }
      }]
    }
  }]
}
```

### tool_choice 参数

| 取值 | 行为 | 适用场景 |
|------|------|----------|
| "auto" | 模型自主决定是否调用工具 | 通用场景 |
| "none" | 不调用任何工具 | 纯对话场景 |
| {"type":"function","function":{"name":"xxx"}} | 强制调用指定工具 | 确定性路由 |

### 调用生命周期

```
1. 开发者将工具定义传给 API
2. LLM 分析对话 + 工具定义 → 输出 tool_calls
3. 客户端解析 tool_calls，执行对应函数
4. 客户端将结果作为 tool 角色 message 传回
5. LLM 基于结果继续推理
```

## 历史背景

在 FC 之前，使 LLM 调用工具的常见做法是：

1. **Prompt 指令法**：在 system prompt 中写格式指令 + 正则解析
2. **JSON Mode**：要求 LLM 输出 JSON，代码解析 JSON 中的工具名和参数

问题：解析不稳定，格式容易出错，模型经常"忘记"输出正确的格式

## 关键突破

FC 训练模型"原生输出函数调用 Token"，而非从文本中解析：

- **Token 级别的结构化输出**：模型在生成过程中就知道自己在输出函数调用
- **Schema 约束**：参数必须符合定义的 JSON Schema，降低幻觉参数概率
- **并行调用支持**（gpt-4-1106-preview+）：一次生成多个工具调用

## 核心优势

| 维度 | FC 方案 | 传统 Prompt 方案 |
|------|---------|-----------------|
| 格式稳定性 | 高（模型原生支持） | 低（依赖解析） |
| 参数准确性 | 高（Schema 约束） | 中（容易漂移） |
| 调试难度 | 低（标准格式） | 高（每次都要看原始输出） |
| 多工具场景 | 批量注入 | 上下文容量受限 |

## 局限

1. **绑定模型**：只有支持 FC 的模型可用
2. **Token 消耗**：工具定义本身消耗大量 Token（长描述 + 复杂 Schema 尤其）
3. **参数幻觉**：虽然比纯文本好，但仍然可能生成不存在的参数值
4. **工具选择可靠性**：工具数量多时选择准确率下降

## 最佳实践

- 工具描述要精确（影响模型选择的准确率）
- 参数尽量少，提供 enum 约束
- 复杂工具先用简单函数封装
