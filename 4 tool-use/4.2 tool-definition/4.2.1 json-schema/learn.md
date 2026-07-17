# JSON Schema 工具定义语法

## 简单介绍

JSON Schema 是工具定义的核心语言。它提供了一套声明式语法来描述工具的参数结构、类型约束、默认值和校验规则。几乎所有主流 LLM 的 Function Calling 协议都使用 JSON Schema（或其子集）来定义工具参数。

## 基本原理

### 核心结构

```json
{
  "name": "send_email",
  "description": "发送电子邮件",
  "parameters": {
    "type": "object",
    "properties": {
      "to": {
        "type": "string",
        "description": "收件人邮箱"
      },
      "subject": {
        "type": "string",
        "description": "邮件主题"
      },
      "body": {
        "type": "string",
        "description": "邮件正文"
      },
      "priority": {
        "type": "string",
        "enum": ["low", "normal", "high"],
        "description": "优先级",
        "default": "normal"
      }
    },
    "required": ["to", "subject", "body"]
  }
}
```

### 核心类型

| 类型 | 说明 | 示例 |
|------|------|------|
| `string` | 字符串 | 名称、描述、内容 |
| `number` | 数字（整数+浮点） | 价格、数量 |
| `integer` | 整数 | ID、计数、页码 |
| `boolean` | 布尔值 | 开关、标记 |
| `array` | 数组 | 列表、集合 |
| `object` | 嵌套对象 | 复合结构 |

### 常用约束

| 约束 | 适用类型 | 说明 |
|------|---------|------|
| `enum` | string | 限制为特定值集合 |
| `minLength`/`maxLength` | string | 字符串长度限制 |
| `minimum`/`maximum` | number | 数值范围 |
| `pattern` | string | 正则表达式 |
| `items` | array | 数组元素的 Schema |
| `additionalProperties` | object | 是否允许额外属性 |

## 历史背景

JSON Schema 最初是作为 REST API 的请求体验证标准（配合 OpenAPI 使用）出现的。LLM 的 Function Calling 借用了这套语法——这很自然，因为大多数后端的工具本身就是 REST API。

以前的做法是：API 团队维护 OpenAPI 文档 → 前端/客户端根据文档调用。现在：JSON Schema 工具定义 → LLM 根据 Schema 自动决定调用。

## 关键区别：REST API Schema vs FC Schema

| 维度 | REST API Schema | FC Schema |
|------|----------------|-----------|
| 使用方 | 开发者 | LLM |
| 容错性 | 严格（拒绝非法请求） | 宽松（LLM 可能出格式错） |
| 描述的重要性 | 中（开发者看文档） | 高（LLM 靠描述理解用途） |
| 嵌套复杂度 | 可以很深 | 建议扁平 |
| 类型系统 | 完整标准 | 子集（各平台裁剪不同） |

## 最佳实践

1. **描述要精确**：`description` 字段是 LLM 理解参数含义的主要依据
2. **优先使用 enum**：限制参数值范围能显著降低参数幻觉
3. **避免深度嵌套**：扁平结构（参数在顶层）比深层嵌套更可靠
4. **提供合理默认值**：减少 LLM 需要决策的参数数量
5. **required 列表要精简**：只标记真正必需的参数

## 局限

- LLM 提供商支持 JSON Schema 子集，并非全部标准（尤其是复杂的组合 Schema 如 `anyOf`, `oneOf`）
- 复杂嵌套的 Schema 会增加模型输出参数的出错率
- Schema 本身的 Token 消耗需要纳入成本考量
