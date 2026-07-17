# Function Calling 的结构化输出本质

## 简单介绍

Function Calling（工具调用）本质上是结构化输出的一种特殊形式。当 LLM 输出一个工具调用声明时，它必须遵守严格的结构（工具名 + 参数 Schema）——这与 JSON Schema 强制输出在技术上是一回事。理解这个本质可以帮助我们更灵活地运用 FC 的能力。

## FC 的结构化本质

### 同一件事的不同视角

```
视角 1: Function Calling
  模型输出 → {name: "get_weather", arguments: {location: "北京"}}
  用途: 工具调用

视角 2: 结构化输出
  模型输出 → {tool: "get_weather", params: {location: "北京"}}
  用途: 任何需要结构化数据的场景
  格式: 实质上是相同的 JSON 结构，只是用途不同
```

### 将 FC 用作通用结构化输出

我们可以利用 FC 协议来获取任意形式的结构化数据，而非仅限于工具调用：

```python
# 场景 1：文本分类
tools = [{
    "name": "classify_intent",
    "description": "对用户意图进行分类",
    "parameters": {
        "type": "object",
        "properties": {
            "intent": {
                "type": "string",
                "enum": ["查询天气", "设置提醒", "发送消息", "其他"]
            },
            "confidence": {
                "type": "number",
                "minimum": 0,
                "maximum": 1
            }
        },
        "required": ["intent", "confidence"]
    }
}]

response = llm.chat("明天会下雨吗？", tools=tools)
# tool_call: classify_intent(intent="查询天气", confidence=0.95)

# 场景 2：信息提取
tools = [{
    "name": "extract_entities",
    "description": "提取文本中的实体",
    "parameters": {
        "type": "object",
        "properties": {
            "people": {"type": "array", "items": {"type": "string"}},
            "places": {"type": "array", "items": {"type": "string"}},
            "dates": {"type": "array", "items": {"type": "string"}}
        },
        "required": ["people", "places", "dates"]
    }
}]
```

## "工具名"作为输出类型的标识

在 FC 作为结构化输出使用时，**工具名就是响应类型标识**：

```
classify_intent       → {intent, confidence}
extract_entities     → {people, places, dates}
generate_embedding   → {vector, dimensions}
analyze_sentiment    → {sentiment, score}
```

## FC vs 通用结构化输出

| 维度 | Function Calling 协议 | 通用 JSON Schema 输出 |
|------|----------------------|---------------------|
| 主要用途 | 工具调用 | 任何结构化输出 |
| 响应结构 | 包含 name + arguments | 纯 JSON |
| LLM 支持 | 几乎所有模型 | 仅部分模型 |
| 兼容性 | 最高 | 中 |
| 作为结构化输出使用 | 可以（技巧） | 原生 |

## 将 FC 作为通用方案的优缺点

**优点**：
- 兼容几乎所有支持 FC 的模型
- 输出结构严谨，参数校验好
- 不需要特殊 API 参数（不需要 JSON Mode）

**缺点**：
- 需要在外层包装"工具名"字段
- 不适合处理多个输出类型（一次只能调一个工具）
- 语义上的"滥用"——代码的可读性较差

## 工程启示

- 理解 FC = 结构化输出可以帮助你在不支持 JSON Mode 的模型上也能获得结构化输出
- 在 Agent 系统中，工具的"结果"也需要结构化——让工具返回结构化的 JSON，便于 LLM 理解
- 如果将 FC 用作通用结构化输出，建议添加合适的命名和描述来表明用途
- 对于多个不同类型的结构化输出需求，使用并行 FC 或分步进行
