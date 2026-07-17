# JSON Schema 强制输出

## 简单介绍

JSON Schema 强制输出是指让 LLM 的输出必须符合预定义的 JSON Schema 结构。这是 OpenAI 结构化输出模式（Structured Outputs）的核心能力——不仅保证输出是合法 JSON，还保证输出的**字段名、类型、枚举值**完全符合定义的 Schema。

## 基本原理

OpenAI 的结构化输出（Structured Outputs）通过 **Strict Function Calling** 机制实现：

```python
from openai import OpenAI
client = OpenAI()

# 定义 Schema
schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {"type": "integer"},
        "skills": {
            "type": "array",
            "items": {"type": "string"}
        }
    },
    "required": ["name", "age", "skills"],
    "additionalProperties": False
}

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "张三，25岁，会Python和Java"}],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "user_info",
            "strict": True,
            "schema": schema
        }
    }
)

# 100% 符合 Schema
# → {"name": "张三", "age": 25, "skills": ["Python", "Java"]}
```

## Strict 模式做了什么

| 特性 | 普通 JSON Mode | Strict JSON Schema |
|------|---------------|-------------------|
| JSON 合法性 | 保证 | 保证 |
| 字段名固定 | 不保证 | 保证 |
| 类型约束 | 不保证 | 保证 |
| Enum 约束 | 不保证 | 保证 |
| 不产出额外字段 | 不保证 | 保证（additionalProperties: false） |
| 嵌套对象 | 可能出错 | 保证 |

### 实现原理

OpenAI 通过**逻辑推理层面的约束**（而非简单的 Prompt 指令）来实现 Strict Mode：

1. 在模型训练阶段加入了"严格遵循 Schema"的示范数据
2. 推理时通过特殊的解码逻辑保证输出符合 Schema
3. additionalProperties: false 是默认的——不会输出 Schema 之外的字段

## 与 Function Calling 的关系

JSON Schema 强制输出本质上是对 Function Calling 的泛化：

```
Function Calling: tools 参数中的 JSON Schema → 强制输出
Structured Outputs: response_format 中的 JSON Schema → 强制输出
```

区别：FC 用于工具调用场景，Structured Outputs 用于通用场景。

## 能力边界

| 限制 | 说明 |
|------|------|
| Schema 嵌套深度 | 最大嵌套深度有限 |
| 递归 Schema | 不支持递归定义（如树形结构） |
| 所有 Schema 特性 | only strict subset（不支持 oneOf, anyOf 等） |
| Token 消耗 | Schema 定义本身消耗 Token |
| 模型版本 | 仅最新模型（GPT-4o, Claude 3.5+）支持 |

## 工程启示

- JSON Schema 强制输出是 Agent 系统"结构化数据提取"的最佳方案
- 它比 Function Calling 更通用——不仅是工具参数，任何 JSON 输出都适用
- 在 Agent 的中间推理步骤中，使用 Structured Outputs 可以保证推理结构的稳定性
- 如果使用自建模型，Outlines 等约束解码库可以替代此功能
- 注意：目前只有 OpenAI 提供原生的 `response_format: json_schema`，其他平台（Claude、Google）主要通过 Function Calling 实现类似效果
