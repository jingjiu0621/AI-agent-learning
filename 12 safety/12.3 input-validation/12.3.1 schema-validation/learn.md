# 12.3.1 schema-validation — Schema 验证

## 简单介绍

Schema 验证是 Agent 输入安全的第一道防线。它将用户输入、工具参数、系统配置等结构化数据与预定义的模式（Schema）进行比对，确保输入的结构、类型、格式和约束条件符合预期。在 Agent 系统中，Schema 验证不仅是类型检查——它还承担着防止工具误调用、阻断畸形攻击载荷、保证多步交互数据一致性的安全职责。

当前主流的 Schema 验证方案包括 JSON Schema（通用标准）、Pydantic（Python 生态）、Zod（TypeScript 生态）、TypeBox（TypeScript 编译时）等，每种方案在表达能力、运行时性能、安全特性上有不同的取舍。

## 基本原理

Schema 验证的核心是一个声明式匹配过程：定义 "输入应该长什么样"，然后检查实际输入是否匹配。

```
输入数据 → Schema 定义 → 编译/解析 → 逐字段验证 → 通过/拒绝
                                          │
                                    ┌─────┴─────┐
                                    │ type check │
                                    │ format     │
                                    │ pattern    │
                                    │ enum       │
                                    │ range      │
                                    │ required   │
                                    │ nested     │
                                    │ recursive  │
                                    └───────────┘
```

验证本质上是一种**契约执行**（Contract Enforcement）——调用方承诺提供符合 Schema 的数据，验证方确保这一承诺被遵守。Agent 场景下，这个契约有三方：

| 角色 | 提供内容 | 验证责任 |
|------|---------|---------|
| LLM（调工具） | 工具调用参数 | LLM 输出是否符合工具 Schema |
| 用户（输入） | 自然语言/指令 | 用户输入是否在安全 schema 范围内 |
| 外部系统（返回） | 工具返回值 | 返回数据是否符合预期 schema |

## 背景

### 从 API 验证到 Agent 输入验证

Schema 验证最早作为 REST API 的请求体验证标准出现。OpenAPI/Swagger 规范定义端点参数格式，服务端在反序列化时进行校验，拒绝格式错误的请求。

```
REST API 时代:                        Agent 时代:
┌─────────┐                          ┌─────────┐
│ 客户端   │ → JSON 请求 →  API     │ 用户    │ → 自然语言 →  Agent
│         │    Schema 验证   │         │    LLM 解析参数
└─────────┘                          └─────────┘
                                     user_input         tool_call
                                     Schema 验证    →   Schema 验证
```

Agent 场景下，Schema 验证的位置发生了变化：

1. **工具定义使用 Schema**：每个工具的参数定义本身就是 JSON Schema（OpenAI/Claude Tool Use 协议）。LLM 根据 Schema 生成参数，服务端在校验后执行。
2. **用户输入也需要 Schema**：用户的自由文本虽然不能严格 Schema 化，但提取出的结构化字段（如意图类别、实体值）需要验证。
3. **工具返回值反校验**：外部 API 返回的数据可能不符合预期 Schema，Agent 需要对结果做 Schema 验证以防止下游 misuse。

### Schema 在 Tool Definition 中的角色

```json
{
  "name": "execute_sql",
  "description": "Execute a SQL query on the database",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "The SQL query to execute",
        "pattern": "^(SELECT|INSERT|UPDATE|DELETE)\\s"  // 安全约束
      },
      "readonly": {
        "type": "boolean",
        "const": true  // 强制只读
      }
    },
    "required": ["query", "readonly"]
  }
}
```

这个 Schema 不仅定义了类型，还通过 `pattern` 限制 SQL 操作类型，通过 `const` 锁定只读模式——Schema 在这里承载了安全策略。

## 核心矛盾

| 矛盾 | 表现 | 后果 |
|------|------|------|
| **严格 Schema vs 自然语言不确定性** | Schema 要求精确结构，但用户描述天然模糊 | 误拒绝或 LLM 无法匹配 |
| **完备约束 vs 工具灵活性** | Schema 定义越精细，工具能处理的场景越窄 | 用户绕道或开发者加宽松 Schema |
| **复杂 Schema vs LLM 遵循能力** | 深度嵌套/多条件 Schema 超出 LLM 的上下文理解 | LLM 频繁生成违反 Schema 的参数 |
| **安全约束 vs 可用性** | 加 pattern/format 等限制防止攻击，但误伤正常输入 | 用户投诉率上升 |
| **验证性能 vs 高吞吐** | 大规模嵌套 Schema 验证耗时显著 | 端到端延迟增加 |

### 核心权衡：严格度与召回率

```
          ┌ 攻击拦截率 (Precision)
          │
  高严格 ┤    ✦ 安全
          │    \
          │     ✦ 生产平衡点
          │      \
  低严格 ┤       ✦ 灵活
          │
          └────────────────── 用户通过率 (Recall)
```

安全团队倾向严格 Schema（拦截所有异常），产品团队倾向宽松 Schema（不让用户受阻）。最佳平衡点取决于 Agent 的攻击面暴露程度。

## 详细内容

### 1. JSON Schema: Draft 2020-12

#### 核心验证关键字

| 关键字 | 作用 | Agent 安全相关性 |
|--------|------|-----------------|
| `type` | 值类型（string/number/object/array/boolean/null） | 基础类型安全 |
| `properties` | 对象字段定义 | 结构化参数约束 |
| `required` | 必填字段列表 | 防止 LLM 遗漏关键参数 |
| `additionalProperties` | 是否允许未定义字段 | **高危：应设为 false 防止注入额外字段** |
| `minProperties`/`maxProperties` | 对象属性数量范围 | 防止超大 payload |
| `items` | 数组元素 Schema | 数组元素类型约束 |
| `minItems`/`maxItems` | 数组长度范围 | 防止数组过大/过小 |
| `uniqueItems` | 数组元素去重 | 防止重复提交 |
| `enum` | 允许值集合 | 白名单约束 |
| `const` | 固定值 | 锁定敏感参数 |
| `minimum`/`maximum` | 数值范围（含 exclusive 变体） | 防止数值溢出 |
| `minLength`/`maxLength` | 字符串长度 | 防止超大文本/注入 |
| `pattern` | 正则表达式 | **模式匹配安全过滤** |
| `format` | 预定义格式（email/uri/date/ipv4/等） | 格式标准化 |
| `contentMediaType` | 内容媒体类型 | 防止类型混淆 |
| `contentEncoding` | 内容编码（如 base64） | 防止编码绕过 |
| `default` | 默认值 | 安全默认值策略 |
| `examples` | 示例值（非验证用） | LLM 理解辅助 |

#### Schema 组合关键字

这些关键字在 Agent Schema 中非常重要，因为它们定义了条件的逻辑关系：

```json
{
  "allOf": [
    { "type": "object" },
    { "required": ["action"] }
  ],
  "oneOf": [
    { "properties": { "action": { "const": "read" } },
      "required": ["query"] },
    { "properties": { "action": { "const": "write" } },
      "required": ["query", "data"] }
  ],
  "anyOf": [
    { "properties": { "format": { "enum": ["json", "csv"] } } },
    { "properties": { "format": { "const": "auto" } } }
  ],
  "not": {
    "properties": { "command": { "pattern": "DROP|DELETE|TRUNCATE" } }
  }
}
```

| 关键字 | 语义 | Agent 场景 |
|--------|------|-----------|
| `allOf` | 必须满足所有子 Schema | 合并多维度约束 |
| `oneOf` | 必须满足恰好一个子 Schema | 区分不同工具调用模式 |
| `anyOf` | 满足任意子 Schema 即通过 | 宽松匹配多个合法路径 |
| `not` | 必须不满足子 Schema | **安全黑名单（如禁止 SQL 关键字）** |

#### 条件式验证: `if/then/else`

```json
{
  "type": "object",
  "properties": {
    "operation": { "type": "string", "enum": ["query", "mutate"] },
    "sql": { "type": "string" }
  },
  "if": { "properties": { "operation": { "const": "query" } } },
  "then": {
    "properties": {
      "sql": { "pattern": "^SELECT\\s" }
    }
  },
  "else": {
    "properties": {
      "sql": { "pattern": "^(INSERT|UPDATE|DELETE)\\s" }
    }
  }
}
```

#### 安全相关 critical 关键字详解

**`additionalProperties: false` 是 Agent 安全的关键设置**

当设置为 `false` 时，任何未在 `properties` 中明确定义的字段都会被拒绝。这防止了：

- 攻击者在工具参数中注入额外字段
- LLM "发明" 新参数绕过安全限制
- 下游代码处理非预期的键名

```json
// 危险：允许额外属性
{
  "type": "object",
  "properties": {
    "command": { "type": "string" }
  }
  // 未设置 additionalProperties，默认 true
}

// 攻击者可传入：{"command": "ls", "shell": true, "sudo": true, "rm": "-rf /"}
// 这些额外字段可能被下游代码意外处理
```

**`pattern` 正则过滤的双刃剑**

正则可以用来限制字符串值的模式，但需要小心 ReDoS（正则拒绝服务攻击）：

```json
// 危险模式（可能被 ReDoS 攻击）：
{ "pattern": "^(a+)+$" }

// 安全模式：
{ "pattern": "^[a-zA-Z0-9_]{1,64}$",
  "maxLength": 64 }
```

### 2. Pydantic v2: Python Schema 定义

Pydantic v2（基于 Rust 的 pydantic-core）是 Python Agent 框架中最常用的 Schema 验证库。

#### 基础模型定义

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from typing import Annotated, Literal
from datetime import datetime

class ToolCall(BaseModel):
    """Agent 工具调用参数 Schema"""
    tool_name: str = Field(
        ...,  # ... 表示必填
        min_length=1,
        max_length=64,
        pattern=r'^[a-z_][a-z0-9_]*$'
    )
    arguments: dict = Field(
        default_factory=dict,
        max_length=100  # 最多 100 个参数
    )
    timestamp: datetime = Field(
        default_factory=datetime.utcnow
    )
    execution_mode: Literal["safe", "sandbox", "full"] = Field(
        default="safe"
    )

    @field_validator('arguments')
    @classmethod
    def check_argument_values(cls, v: dict) -> dict:
        """递归检查参数值长度"""
        for key, value in v.items():
            if isinstance(value, str) and len(value) > 10000:
                raise ValueError(
                    f"Argument '{key}' value exceeds 10000 chars"
                )
        return v
```

#### `model_validate` 核心方法

```python
# 同步验证
try:
    validated = ToolCall.model_validate(raw_input)
except ValidationError as e:
    # e.errors() 包含详细错误信息
    for err in e.errors():
        print(f"Field: {err['loc']}, Error: {err['msg']}, Type: {err['type']}")
```

#### 自定义安全验证器

```python
from pydantic import BaseModel, field_validator
import re

class UserQuery(BaseModel):
    content: str
    max_tokens: int = Field(default=4096, ge=1, le=128000)

    @field_validator('content')
    @classmethod
    def detect_prompt_injection(cls, v: str) -> str:
        """检测并标记潜在的 prompt 注入"""
        injection_patterns = [
            r'ignore\s+(all\s+)?(previous|above|prior)',
            r'disregard\s+(all\s+)?(previous|above|prior)',
            r'system\s+prompt',
            r'you\s+are\s+(now|not)',
        ]
        for pattern in injection_patterns:
            if re.search(pattern, v, re.IGNORECASE):
                raise ValueError(f"Potential injection detected: {pattern}")
        return v

    @model_validator(mode='after')
    def check_combined_risk(self) -> 'UserQuery':
        """跨字段组合风险检测"""
        risk_score = 0
        if len(self.content) > 5000:
            risk_score += 1
        if self.max_tokens > 32000:
            risk_score += 2
        if risk_score > 2:
            raise ValueError("Combined risk score exceeds threshold")
        return self
```

#### 错误处理体系

Pydantic v2 的错误处理层级：

```
ValidationError
├── line_errors: list[ErrorDict]
│   ├── type: 错误类型标识
│   ├── loc:  字段路径（支持嵌套）
│   ├── msg:  人类可读消息
│   ├── input: 原始输入值
│   │
│   └── ctx:  上下文（如 error: ValueError）
│       └── error: 原始异常

错误类型示例：
  - missing: 缺少必填字段
  - string_pattern_mismatch: 正则不匹配
  - string_too_short/long: 长度越界
  - literal_error: enum/const 值不在允许集合
  - json_invalid: JSON 解析失败
  - value_error: 自定义验证器抛出
```

```python
# 结构化错误处理
def handle_validation_errors(err: ValidationError) -> dict:
    """将验证错误转为安全日志和用户反馈"""
    errors = err.errors()
    log_payload = []
    user_messages = []

    for e in errors:
        field_path = " -> ".join(str(p) for p in e['loc'])
        error_type = e['type']

        # 安全敏感错误：记录完整上下文
        if error_type in (
            'value_error', 'literal_error', 'string_pattern_mismatch'
        ):
            log_payload.append({
                "field": field_path,
                "type": error_type,
                "input_truncated": str(e['input'])[:200],  # 截断防日志注入
                "msg": e['msg']
            })

        # 向用户展示的信息（不泄露 Schema 细节）
        user_messages.append(f"Field '{field_path}' is invalid")

    return {
        "log": log_payload,
        "user_message": "; ".join(user_messages),
        "rejected": True
    }
```

### 3. OpenAPI/Swagger Integration

Agent 工具定义通常直接映射到 OpenAPI 规范。现代 Agent 框架支持从 OpenAPI 规范自动生成工具 Schema。

#### OpenAPI -> Tool Schema 映射

```yaml
# OpenAPI 3.1 规范片段
paths:
  /api/users/{userId}:
    get:
      operationId: getUserById
      parameters:
        - name: userId
          in: path
          required: true
          schema:
            type: string
            pattern: '^usr_[a-zA-Z0-9]{24}$'
        - name: includeDeleted
          in: query
          schema:
            type: boolean
            default: false
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
```

自动转化为 Tool Schema：

```json
{
  "name": "getUserById",
  "description": "Get user details by user ID",
  "input_schema": {
    "type": "object",
    "properties": {
      "userId": {
        "type": "string",
        "pattern": "^usr_[a-zA-Z0-9]{24}$",
        "description": "User ID starting with usr_"
      },
      "includeDeleted": {
        "type": "boolean",
        "default": false
      }
    },
    "required": ["userId"],
    "additionalProperties": false
  }
}
```

#### OpenAPI 鉴权 Schema 安全特殊处理

```python
class OpenAPIToolSchemaBuilder:
    """从 OpenAPI 构建工具 Schema 时注入安全约束"""

    def build_schema(self, operation: dict) -> dict:
        schema = {
            "type": "object",
            "properties": {},
            "required": [],
            "additionalProperties": False  # 默认拒绝额外字段
        }

        for param in operation.get('parameters', []):
            prop = self._build_property(param)
            # 安全增强：自动注入约束
            if prop.get('type') == 'string':
                prop.setdefault('maxLength', 10000)
            if prop.get('type') == 'array':
                prop.setdefault('maxItems', 100)
            schema['properties'][param['name']] = prop
            if param.get('required'):
                schema['required'].append(param['name'])

        # 注入安全元数据
        schema['_security'] = {
            "auth_required": self._get_auth_requirement(operation),
            "rate_limit": self._get_rate_limit(operation),
            "allowed_roles": self._get_allowed_roles(operation)
        }
        return schema
```

### 4. Dynamic Schema Generation

Agent 场景中，静态 Schema 往往不够——因为 Agent 的上下文会影响合法的参数范围。

#### 从上下文生成 Schema

```python
class DynamicSchemaGenerator:
    """基于当前 Agent 上下文动态调整 Schema"""

    def generate_tool_schema(
        self,
        base_schema: dict,
        context: AgentContext
    ) -> dict:
        schema = deepcopy(base_schema)

        # 根据用户身份调整 allowable values
        if context.user_role == "admin":
            # 管理员可以操作所有数据库
            allowed_tables = schema['properties']['table']['enum']
        else:
            allowed_tables = self._get_user_allowed_tables(
                context.user_id
            )
            schema['properties']['table']['enum'] = allowed_tables

        # 根据当前会话状态调整必填字段
        if context.conversation_phase == "confirmation":
            schema['required'].append("confirmed")

        # 根据调用历史调整限制
        recent_calls = context.get_recent_calls(minutes=5)
        if len(recent_calls) > 10:
            schema['properties']['batch_size']['maximum'] = 10
        else:
            schema['properties']['batch_size']['maximum'] = 100

        return schema
```

#### Few-shot 示例动态注入

Few-shot 示例可以嵌入 Schema 中帮助 LLM 理解正确的参数格式：

```python
def build_schema_with_examples(
    base_schema: dict,
    examples: list[dict]
) -> dict:
    """将 few-shot 示例注入 schema 的 examples 字段"""
    schema = deepcopy(base_schema)

    # examples 是 JSON Schema 的关键字，
    # LLM 提供商（如 OpenAI）会读取它辅助生成
    for field_name, field_schema in schema['properties'].items():
        field_examples = [
            ex.get(field_name) for ex in examples
            if field_name in ex
        ]
        if field_examples:
            field_schema['examples'] = field_examples

    return schema
```

#### Schema 版本化

Agent 工具 Schema 会随产品迭代变化，需要版本管理：

```python
class SchemaVersionManager:
    """Schema 版本管理——保证向后兼容"""

    def __init__(self):
        self._registry: dict[str, list[SchemaVersion]] = {}

    def register_schema(
        self,
        tool_name: str,
        schema: dict,
        version: str,
        migration_fn: callable = None
    ):
        """注册新版本 Schema"""
        if tool_name not in self._registry:
            self._registry[tool_name] = []
        self._registry[tool_name].append(SchemaVersion(
            version=version,
            schema=schema,
            migration_fn=migration_fn
        ))

    def migrate_input(
        self,
        tool_name: str,
        input_data: dict,
        from_version: str,
        to_version: str
    ) -> dict:
        """将输入数据从旧版本 Schema 迁移到新版本"""
        versions = self._registry[tool_name]
        current = from_version
        data = input_data

        for v in versions:
            if v.version == current and v.migration_fn:
                data = v.migration_fn(data)
                current = v.version
            if current == to_version:
                break

        return data
```

### 5. Recursive Schema

Agent 工具经常需要处理嵌套或递归结构（如树形菜单、嵌套任务分解）。

#### JSON Schema 递归定义

JSON Schema 支持通过 `$ref` 引用自身：

```json
{
  "$defs": {
    "task": {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "description": { "type": "string" },
        "subtasks": {
          "type": "array",
          "items": { "$ref": "#/$defs/task" },
          "maxItems": 10
        }
      },
      "required": ["id", "description"]
    }
  },
  "$ref": "#/$defs/task"
}
```

#### Pydantic 递归模型

```python
from __future__ import annotations
from pydantic import BaseModel, Field
from typing import Optional

class TaskNode(BaseModel):
    id: str
    description: str = Field(max_length=500)
    subtasks: list[TaskNode] = Field(
        default_factory=list,
        max_length=10  # 限制子任务数量
    )

    # 深度限制——运行时检测
    @model_validator(mode='after')
    def check_depth(self, info) -> 'TaskNode':
        depth = _compute_depth(self)
        if depth > 5:
            raise ValueError("Task nesting depth exceeds limit (5)")
        return self


def _compute_depth(node: TaskNode, current: int = 0) -> int:
    """递归计算嵌套深度"""
    if not node.subtasks:
        return current
    return max(
        _compute_depth(child, current + 1)
        for child in node.subtasks
    )
```

#### 深度限制与循环引用防护

| 问题 | 方案 | 实现 |
|------|------|------|
| **无限递归** | 最大深度限制 | 递归模型中注入 depth counter |
| **循环引用** | 引用追踪 + visited set | 序列化时检测循环路径 |
| **栈溢出** | 迭代验证替代递归 | 使用栈模拟递归验证 |
| **性能退化** | 内存缓存已检查路径 | LRU cache 对 $ref 解析结果 |

```python
class RecursiveSchemaValidator:
    """带深度限制和循环检测的递归 Schema 验证"""

    def __init__(self, max_depth: int = 10):
        self.max_depth = max_depth

    def validate(self, schema: dict, data: any) -> list[str]:
        errors = []
        self._validate_recursive(
            schema, data, "$", errors, visited=set(), depth=0
        )
        return errors

    def _validate_recursive(
        self,
        schema: dict,
        data: any,
        path: str,
        errors: list[str],
        visited: set,
        depth: int
    ):
        if depth > self.max_depth:
            errors.append(f"{path}: exceeded max depth")
            return

        # 循环引用检测
        ref = schema.get('$ref')
        if ref:
            ref_key = f"{ref}:{id(data) if isinstance(data, (dict, list)) else str(data)}"
            if ref_key in visited:
                errors.append(f"{path}: circular reference detected")
                return
            visited.add(ref_key)

        # ... 递归验证逻辑 ...
```

### 6. Union/Discriminated Types

Agent 工具调用中，同一个工具可能有多种合法的参数组合——这对应了 Union 类型的需求。

#### JSON Schema oneOf（无判别器）

```json
{
  "type": "object",
  "properties": {
    "action": { "type": "string" }
  },
  "oneOf": [
    {
      "properties": {
        "action": { "const": "search" },
        "query": { "type": "string" },
        "max_results": { "type": "integer", "minimum": 1 }
      },
      "required": ["query"]
    },
    {
      "properties": {
        "action": { "const": "navigate" },
        "url": { "type": "string", "format": "uri" }
      },
      "required": ["url"]
    },
    {
      "properties": {
        "action": { "const": "click" },
        "selector": { "type": "string" },
        "timeout_ms": { "type": "integer", "default": 3000 }
      },
      "required": ["selector"]
    }
  ],
  "required": ["action"]
}
```

#### 判别式联合 (Discriminated Union)

Pydantic 支持带判别器的联合类型，验证效率远高于 `oneOf`：

```python
from pydantic import BaseModel, Field
from typing import Annotated, Union
from typing_extensions import TypeAlias

class SearchAction(BaseModel):
    action: str = "search"  # 字面量默认值作为判别器
    query: str = Field(..., min_length=1, max_length=500)
    max_results: int = Field(default=10, ge=1, le=100)

class NavigateAction(BaseModel):
    action: str = "navigate"
    url: str = Field(..., max_length=2048)
    new_tab: bool = False

class ClickAction(BaseModel):
    action: str = "click"
    selector: str = Field(..., max_length=256)
    timeout_ms: int = Field(default=3000, ge=100, le=30000)

class ReadAction(BaseModel):
    action: str = "read"

# 判别式 Union——Pydantic 通过 action 字段值快速确定类型
BrowserAction: TypeAlias = Annotated[
    Union[SearchAction, NavigateAction, ClickAction, ReadAction],
    Field(discriminator='action')
]

class ToolCall(BaseModel):
    action: BrowserAction
```

**性能差异**：判别式 Union 使用 O(1) 的字段值匹配而非 O(n) 的逐一尝试，在处理 10+ 分支 Union 时速度提升 10-50 倍。

#### 安全注意事项

```python
class ToolCallUnionValidator:
    """Union 类型的安全验证——防止类型混淆攻击"""

    def validate_with_security(
        self,
        raw_input: dict,
        action_types: dict[str, type[BaseModel]]
    ) -> BaseModel:
        action = raw_input.get('action')

        # 1. action 必须在白名单中
        if action not in action_types:
            raise SecurityError(f"Unknown action type: {action}")

        # 2. 不允许在 action 字段中传递非字符串值（类型混淆）
        if not isinstance(action, str):
            raise SecurityError("Action must be a string")

        # 3. 不允许 action 字段包含特殊字符（注入）
        if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', action):
            raise SecurityError("Invalid action identifier")

        # 4. 执行验证
        action_type = action_types[action]
        return action_type.model_validate(raw_input)
```

## Example Code: Python SchemaValidator

一个完整的 Schema 验证器，结合 Pydantic + JSON Schema 回退 + 自定义安全验证器：

```python
"""
SchemaValidator: Agent 输入 Schema 验证的统一入口
支持 Pydantic v2 模型和 JSON Schema 两种验证路径
"""

import json
import re
from typing import Any, Optional, Callable
from datetime import datetime, timedelta
from dataclasses import dataclass, field

try:
    from pydantic import BaseModel, ValidationError as PydanticValidationError
    HAS_PYDANTIC = True
except ImportError:
    HAS_PYDANTIC = False

try:
    from jsonschema import validate as js_validate
    from jsonschema.exceptions import ValidationError as JSONSchemaError
    HAS_JSONSCHEMA = True
except ImportError:
    HAS_JSONSCHEMA = False


# ─── 验证结果类型 ──────────────────────────────────────────────

@dataclass
class ValidationResult:
    passed: bool
    errors: list[dict] = field(default_factory=list)
    warnings: list[dict] = field(default_factory=list)
    sanitized_data: Any = None
    duration_ms: float = 0.0
    schema_name: str = ""

    @classmethod
    def success(cls, data: Any, schema_name: str = ""):
        return cls(passed=True, sanitized_data=data, schema_name=schema_name)

    @classmethod
    def failure(cls, errors: list, schema_name: str = ""):
        return cls(passed=False, errors=errors, schema_name=schema_name)

    def to_log(self) -> dict:
        return {
            "passed": self.passed,
            "error_count": len(self.errors),
            "schema": self.schema_name,
            "duration_ms": round(self.duration_ms, 2)
        }


# ─── 安全验证器基类 ────────────────────────────────────────────

class SecurityValidator:
    """可组合的安全验证器——每个验证器检查一个威胁维度"""

    def __init__(self, name: str):
        self.name = name

    def validate(self, data: Any) -> Optional[str]:
        """返回 None 表示通过，返回 str 表示失败原因"""
        raise NotImplementedError


class MaxLengthValidator(SecurityValidator):
    def __init__(self, max_len: int = 100000):
        super().__init__("max_length")
        self.max_len = max_len

    def validate(self, data: Any) -> Optional[str]:
        if isinstance(data, str) and len(data) > self.max_len:
            return f"String exceeds max length ({len(data)} > {self.max_len})"
        if isinstance(data, (list, dict)):
            serialized = json.dumps(data)
            if len(serialized) > self.max_len:
                return f"Serialized data exceeds max length"
        return None


class PatternBlocklistValidator(SecurityValidator):
    """黑名单正则模式匹配——拒绝已知攻击模式"""

    def __init__(self, patterns: list[str] = None):
        super().__init__("pattern_blocklist")
        self.patterns = patterns or [
            r'ignore\s+all\s+previous\s+instructions',
            r'you\s+are\s+(not|now)\s+',
            r'系统提示词',
            r'<script[^>]*>.*?</script>',
            r'\$\{.*?\}',      # template injection
            r'`.*?`',           # shell injection in backtick
        ]
        self._compiled = [re.compile(p, re.IGNORECASE | re.DOTALL)
                          for p in self.patterns]

    def validate(self, data: Any) -> Optional[str]:
        text = ""
        if isinstance(data, str):
            text = data
        elif isinstance(data, dict):
            text = json.dumps(data)

        for i, pattern in enumerate(self._compiled):
            if pattern.search(text):
                return f"Blocked by pattern #{i}: {self.patterns[i][:50]}..."
        return None


# ─── Schema Validator ──────────────────────────────────────────

class SchemaValidator:
    """
    统一 Schema 验证器

    工作流程：
    1. 安全预处理（长度/黑名单检查）
    2. Pydantic 模型验证（如果提供）
    3. JSON Schema 验证（如果提供）
    4. 自定义安全验证器链
    5. 结果汇总与日志
    """

    def __init__(
        self,
        pydantic_model: Optional[type] = None,
        json_schema: Optional[dict] = None,
        security_validators: list[SecurityValidator] = None,
        schema_name: str = "unnamed",
        sanitizer: Optional[Callable] = None,
    ):
        self.pydantic_model = pydantic_model
        self.json_schema = json_schema
        self.security_validators = security_validators or []
        self.schema_name = schema_name
        self.sanitizer = sanitizer

        # 编译 JSON Schema 引用（如果支持）
        if json_schema and HAS_JSONSCHEMA:
            from jsonschema import Draft202012Validator
            self._validator_cls = Draft202012Validator
        else:
            self._validator_cls = None

    def validate(self, raw_input: Any) -> ValidationResult:
        start = datetime.utcnow()

        # ── 阶段 1: 安全预处理 ──
        for sec_val in self.security_validators:
            err = sec_val.validate(raw_input)
            if err:
                dur = (datetime.utcnow() - start).total_seconds() * 1000
                return ValidationResult.failure(
                    errors=[{
                        "validator": sec_val.name,
                        "message": err,
                        "stage": "security_precheck"
                    }],
                    schema_name=self.schema_name
                )

        # ── 阶段 2: 数据清洗（可选） ──
        data = self.sanitizer(raw_input) if self.sanitizer else raw_input

        # ── 阶段 3: Pydantic 验证 ──
        if self.pydantic_model and HAS_PYDANTIC:
            try:
                validated = self.pydantic_model.model_validate(data)
                data = validated.model_dump()
            except PydanticValidationError as e:
                dur = (datetime.utcnow() - start).total_seconds() * 1000
                return ValidationResult.failure(
                    errors=[{
                        "validator": "pydantic",
                        "field": list(err['loc']),
                        "message": err['msg'],
                        "type": err['type'],
                        "stage": "pydantic"
                    } for err in e.errors()],
                    schema_name=self.schema_name
                )

        # ── 阶段 4: JSON Schema 验证 ──
        if self.json_schema and HAS_JSONSCHEMA and self._validator_cls:
            try:
                validator = self._validator_cls(self.json_schema)
                errors = list(validator.iter_errors(data))
                if errors:
                    dur = (datetime.utcnow() - start).total_seconds() * 1000
                    return ValidationResult.failure(
                        errors=[{
                            "validator": "json_schema",
                            "path": list(e.absolute_path),
                            "message": e.message,
                            "stage": "json_schema"
                        } for e in errors],
                        schema_name=self.schema_name
                    )
            except Exception as e:
                # Schema 本身可能有语法错误
                pass

        dur = (datetime.utcnow() - start).total_seconds() * 1000
        return ValidationResult.success(
            data=data,
            schema_name=self.schema_name
        )


# ─── 使用示例 ──────────────────────────────────────────────────

if __name__ == "__main__":
    # 定义 Pydantic 模型
    class SearchInput(BaseModel):
        query: str
        max_results: int = 10
        include_sensitive: bool = False

        # 安全验证器
        from pydantic import field_validator

        @field_validator('query')
        @classmethod
        def no_dangerous_patterns(cls, v):
            dangerous = ['DROP ', 'DELETE ', 'TRUNCATE ',
                         'rm -rf', 'sudo ', 'chmod 777']
            for d in dangerous:
                if d.lower() in v.lower():
                    raise ValueError(f"Dangerous pattern detected: {d}")
            return v

        @field_validator('include_sensitive')
        @classmethod
        def restrict_sensitive(cls, v, info):
            # 只有特定角色可以查看敏感数据
            # 这里假设 context 通过 info.data 传递
            return v  # 实际场景需要更严格的检查

    # 构建验证器
    validator = SchemaValidator(
        pydantic_model=SearchInput,
        json_schema={
            "type": "object",
            "properties": {
                "query": {"type": "string", "maxLength": 500},
                "max_results": {"type": "integer", "minimum": 1, "maximum": 100},
                "include_sensitive": {"type": "boolean"}
            },
            "required": ["query"],
            "additionalProperties": False
        },
        security_validators=[
            MaxLengthValidator(max_len=100000),
            PatternBlocklistValidator(),
        ],
        sanitizer=lambda x: x,  # 实际可实现 trim/strip/normalize
        schema_name="search_tool"
    )

    # 测试
    result = validator.validate({
        "query": "SELECT * FROM users",  # 会触发 Pydantic 验证器
        "max_results": 500,  # 会触发 JSON Schema (maximum)
    })

    print(f"Passed: {result.passed}")
    print(f"Errors: {result.errors}")
```

## Capability Boundaries

Schema 验证能做到什么，做不到什么——这一点对安全设计至关重要：

### Schema 不能验证的领域

| 边界 | 说明 | 攻击实例 |
|------|------|---------|
| **语义正确性** | Schema 无法理解参数的实际含义是否合理 | 查询 "delete all users" 语法正确，语义恶意 |
| **业务逻辑合法性** | Schema 无法判断操作在业务上是否允许 | 转账金额 -100 元（语法通过，业务意义错误） |
| **身份与权限** | Schema 不包含访问控制语义 | 低权限用户请求删除管理员账户 |
| **时序一致性** | Schema 验证单次调用，无法感知多步调用链的上下文 | 第一步获取 token，第二步用 token 执行危险操作 |
| **数据真实性** | Schema 无法验证参数值的真实性 | 提供伪造的订单号 `ord_000000000001` |
| **对抗性内容** | 复杂的 prompt 注入语义无法被 Schema 规则捕获 | 精心构造的指令越狱 |
| **编码混淆** | Schema 在解码前验证，但攻击可以利用编码绕过 | Base64/Hex 编码的恶意参数 |

### Schema 验证的安全边界

```
Schema 验证有效区                     Schema 验证盲区
───────────────────────              ───────────────────────
✓ 类型错误                          ✗ 语义恶意但类型正确
✓ 格式错误                          ✗ 业务逻辑违规
✓ 值域越界                          ✗ 权限越权操作
✓ 必填缺失                          ✗ 多步攻击编排
✓ 额外字段注入                      ✗ 对抗性提示注入
✓ 正则匹配黑名单                    ✗ 上下文相关攻击
✓ 嵌套深度超限                      ✗ 时间序列攻击
✓ 编码混淆（配合预处理）              ✗ 加密/混淆负载
```

### 设计原则：分层防御

Schema 验证是输入验证的**第一层**，不是唯一一层。始终保持分层防御思维：

```
输入 → Schema 验证 → 语义验证 → 权限检查 → 执行监控
         ↑              ↑           ↑           ↑
      语法安全        内容安全     访问控制    行为异常检测
```

## Comparison: JSON Schema vs Pydantic vs Zod vs TypeBox

| 维度 | JSON Schema | Pydantic v2 | Zod | TypeBox |
|------|------------|-------------|-----|---------|
| **语言** | 语言无关（JSON） | Python | TypeScript | TypeScript |
| **核心能力** | 声明式验证标准 | 运行时类型验证 | 运行时类型验证 | 编译时+运行时 |
| **验证速度** | 中等（Python 实现） | **极快**（Rust core） | 快 | 快 |
| **类型推断** | 无 | 从模型自动推断 | **从 schema 推断 TS 类型** | 生成 JSON Schema |
| **错误信息** | 通用 | **详细结构化** | 详细 | 通用 |
| **递归支持** | $ref 引用 | ✅ 自引用模型 | ✅ lazy/recursive | ✅ Type 递归 |
| **Union 类型** | oneOf/anyOf | Union + discriminated | discriminated union | Union |
| **自定义验证** | 无原生支持 | **field_validator** | refine/superRefine | 需要外部 |
| **JSON Schema 导出** | 本身即是 | model_json_schema() | 有限支持 | 本身即是 |
| **流式验证** | 无 | 无原生 | **partial/渐进式** | 无 |
| **树摇优化** | 无 | 无 | **支持** | **支持** |
| **生态系统** | 通用标准 | FastAPI/LangChain | tRPC/Next.js | 新兴 |
| **Agent 场景评分** | ★★★☆☆ 通用但缺安全 | ★★★★★ Python 最优 | ★★★★☆ TS 生态 | ★★★☆☆ 工具链待成熟 |

### 选型建议

```
纯 Python Agent（LangChain/CrewAI/AutoGen）     → Pydantic v2
TypeScript Agent（Vercel AI SDK/LangChain.js）  → Zod
需要跨语言 Schema 交换/序列化                    → JSON Schema
需要高性能 + JSON Schema 导出                    → TypeBox
工具定义提供给 LLM（OpenAI/Claude API）           → JSON Schema
```

### Pydantic 特别优势（Agent 安全场景）

Pydantic v2 在 Agent 安全验证中的独特优势：

1. **Rust 核心性能**：验证速度比纯 Python 的 JSON Schema 实现快 5-20 倍
2. **自定义验证器**：可以直接注入安全检测逻辑（注入检测、黑名单、频率检查）
3. **模型序列化/反序列化**：`model_dump()` 自动清洗多余字段
4. **FastAPI 深度集成**：Agent API 网关和工具执行器使用同一验证层
5. **LangChain 默认**：LangChain 工具定义直接使用 Pydantic 模型

## Engineering Optimization

### Schema 编译与缓存

Schema 解析/编译的成本远高于验证本身——每次验证都重新解析 Schema 是常见性能瓶颈。

```python
class SchemaCache:
    """多级 Schema 缓存——避免重复解析开销"""

    def __init__(self, max_size: int = 1000, ttl_seconds: int = 300):
        self._cache: dict[str, tuple[Any, float]] = {}
        self.max_size = max_size
        self.ttl = timedelta(seconds=ttl_seconds)

    def get_or_compile(
        self,
        schema_key: str,
        schema_factory: Callable[[], Any],
        compiler: Callable[[Any], Any]
    ) -> Any:
        """缓存 Schema 编译结果"""
        now = datetime.utcnow()

        # 缓存命中检查
        if schema_key in self._cache:
            compiled, timestamp = self._cache[schema_key]
            if now - timestamp < self.ttl:
                return compiled

        # 编译新 Schema
        schema = schema_factory()
        compiled = compiler(schema)

        # 缓存管理（LRU 近似）
        if len(self._cache) >= self.max_size:
            # 移除最老条目
            oldest = min(self._cache.keys(),
                         key=lambda k: self._cache[k][1])
            del self._cache[oldest]

        self._cache[schema_key] = (compiled, now)
        return compiled


# 使用示例
schema_cache = SchemaCache()

validator = schema_cache.get_or_compile(
    schema_key="user_query_v2",
    schema_factory=lambda: UserQuery.model_json_schema(),
    compiler=Draft202012Validator
)
```

### 编译性能数据

| 操作 | 耗时（首次） | 耗时（缓存后） | 加速比 |
|------|------------|---------------|--------|
| JSON Schema 解析 | 5-50ms | ~0ms | ∞ |
| Pydantic 模型编译 | 100-500ms | 2-8µs | 10,000x |
| Zod Schema 解析 | 2-10ms | ~0.01ms | 500x |
| TypeBox 编译 | 1-5ms | ~0.01ms | 200x |

### 验证错误信息的工程化

```python
class ValidationErrorFormatter:
    """
    验证错误格式化——给人和给 AI 的不同展示
    """

    @staticmethod
    def for_human(errors: list[dict]) -> str:
        """最终用户看到的简洁消息（不泄漏 Schema 细节）"""
        messages = []
        for e in errors[:3]:  # 最多显示 3 条
            field = " -> ".join(str(p) for p in e.get('field', []))
            msg = e.get('message', '')

            # 通用化消息
            if 'string_too_short' in e.get('type', ''):
                msg = "Value is too short"
            elif 'string_too_long' in e.get('type', ''):
                msg = "Value is too long"
            elif 'pattern_mismatch' in e.get('type', ''):
                msg = "Value contains invalid characters"
            elif 'missing' in e.get('type', ''):
                msg = f"Required field '{field}' is missing"

            messages.append(msg)

        return "Validation failed: " + "; ".join(messages)

    @staticmethod
    def for_llm(errors: list[dict]) -> str:
        """LLM 看到的结构化错误——帮助 LLM 修正参数"""
        structured = []
        for e in errors:
            structured.append({
                "field": list(e.get('loc', [])),
                "issue": e.get('msg', ''),
                "expected": e.get('ctx', {}).get('expected', ''),
                "received": str(e.get('input', ''))[:200]
            })
        return json.dumps({"validation_errors": structured,
                           "retry_suggestion": "Please fix the above fields"})

    @staticmethod
    def for_logging(errors: list[dict]) -> str:
        """安全日志（脱敏、去重、聚合）"""
        # 移除可能包含敏感数据的 input 值
        safe_errors = []
        for e in errors:
            safe = {k: v for k, v in e.items() if k != 'input'}
            safe['input_type'] = type(e.get('input')).__name__
            safe_errors.append(safe)
        return json.dumps(safe_errors, ensure_ascii=False)
```

### 部分验证（Partial Validation）

在 Agent 场景中，有时需要在输入处理过程中途验证已解析的部分：

```python
class PartialValidator:
    """
    部分验证——在流式输入或逐步构建参数时使用
    """

    def __init__(self, full_schema: dict):
        self.full_schema = full_schema
        self._required = set(full_schema.get('required', []))

    def validate_partial(self, data: dict) -> ValidationResult:
        """
        只验证已存在的字段，不要求必填字段都必须出现
        用于中间状态检查
        """
        errors = []
        properties = self.full_schema.get('properties', {})

        for key, value in data.items():
            if key not in properties:
                errors.append({
                    "field": [key],
                    "message": "Unknown field",
                    "type": "extra_field"
                })
                continue

            field_schema = properties[key]
            field_type = field_schema.get('type')

            # 类型检查（仅检查存在的字段）
            if field_type == 'string' and not isinstance(value, str):
                errors.append({
                    "field": [key],
                    "message": f"Expected string, got {type(value).__name__}",
                    "type": "type_error"
                })
            elif field_type == 'integer' and not isinstance(value, int):
                errors.append({
                    "field": [key],
                    "message": f"Expected integer, got {type(value).__name__}",
                    "type": "type_error"
                })

            # 值范围检查
            if isinstance(value, str):
                max_len = field_schema.get('maxLength')
                if max_len and len(value) > max_len:
                    errors.append({
                        "field": [key],
                        "message": f"Exceeds max length {max_len}",
                        "type": "string_too_long"
                    })

        return ValidationResult(
            passed=len(errors) == 0,
            errors=errors,
            sanitized_data=data
        )
```

### 验证流水线优化策略

```
┌─────────────────────────────────────────────────┐
│          验证流水线 Pipeline                       │
├─────────────────────────────────────────────────┤
│                                                   │
│  1. 粗筛（< 1µs）                                  │
│     ┌─ type check (is it a dict?)                 │
│     └─ exists check (are required keys present?)  │
│                                                   │
│  2. 精验（1-100µs per field）                      │
│     ┌─ per-field type/format/pattern              │
│     ├─ array length / object property count       │
│     └─ nested schema (deferred/on-demand)         │
│                                                   │
│  3. 安全扫描（10-100µs）                           │
│     ┌─ pattern blacklist                          │
│     ├─ max length check (all string fields)       │
│     └─ depth limit (recursive schemas)            │
│                                                   │
│  4. 语义验证（可选, 100-500ms）                     │
│     ┌─ LLM-as-Judge for high-risk inputs          │
│     └─ business rule enforcement                  │
│                                                   │
└─────────────────────────────────────────────────┘
```

快速失败（Fail Fast）策略：粗筛不通过就直接拒绝，不进入精验阶段。这个策略可以拦截大部分攻击流量，延迟极低。

### 生产配置示例

```yaml
# schema-validation-config.yaml
validation:
  default_schema:
    additionalProperties: false  # 全局拒绝额外字段
    max_depth: 10                # 递归最大深度

  performance:
    cache_ttl: 300               # Schema 缓存 TTL (秒)
    max_cache_size: 1000         # 最大缓存 Schema 数
    timeout_ms: 50               # 单次验证超时
    parallel: true               # 数组元素并行验证

  error_handling:
    max_errors_shown: 5          # 展示给用户的错误数
    log_input_sample: true       # 日志记录输入样本
    log_input_truncate: 200      # 日志截断长度

  security:
    pre_check:
      - max_length: 100000       # 拒绝超大输入
      - pattern_blocklist: true  # 黑名单正则
    fail_open: false             # 验证异常时拒绝（fail closed）
```

## 关键认知

1. **Schema 验证是安全的第一道门，不是最后一道**——它拦截格式错误和简单攻击，但需要与其他安全层配合才能防御复杂威胁。

2. **`additionalProperties: false` 是 Agent 工具 Schema 最重要的安全设置**——它防止攻击者和 LLM 自身注入未定义的参数。

3. **Pydantic v2 的 Rust 核心使其在性能上远超纯 Python 的 JSON Schema 实现**——在 Agent 高吞吐场景下选 Pydantic 而非 jsonschema 库。

4. **动态 Schema 是 Agent 独有的验证需求**——传统 API 的 Schema 是静态的，但 Agent 需要根据用户身份、会话状态、历史行为动态调整 Schema 范围。

5. **Schema 编译成本远高于验证成本**——始终缓存编译后的 Schema（或 Pydantic 模型），不要每次验证都重新解析。

6. **Union/Discriminated 类型**可以将 Agent 的多模式工具调用验证从 O(n) 降到 O(1)，对需要处理 10+ 动作类型的 Agent 至关重要。

7. **验证错误信息需要三层格式化**——给用户的（简洁通用）、给 LLM 的（帮助重试）、给安全日志的（详细脱敏）。
