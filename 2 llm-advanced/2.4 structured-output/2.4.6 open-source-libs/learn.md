# 开源结构化输出库（Outlines, JsonFormer, lm-format-enforcer）

## 简单介绍

开源结构化输出库在自建 LLM 推理时使用，通过约束解码技术强制模型输出符合预定义格式。它们填补了"托管 API 的 JSON Mode/FC"和"自建模型的结构化需求"之间的空白。

## 三大主流库对比

| 库 | 语言 | 核心方法 | 支持格式 | 性能 |
|------|------|---------|---------|------|
| Outlines | Python | FSM + Regex | JSON, Regex, Pydantic | 中 |
| JsonFormer | Python | Partial JSON + FSM | JSON | 快 |
| lm-format-enforcer | Python | FSM + Token Heuristic | JSON, Regex, 自定义 | 快 |

## Outlines（最成熟）

```python
from outlines import generate, models
import outlines.text.generate as text_gen

# 加载模型
model = models.transformers("mistralai/Mistral-7B-Instruct")

# 方案 1: JSON Schema 约束
import pydantic
class User(pydantic.BaseModel):
    name: str
    age: int
    email: str | None = None

generator = generate.json(model, User)
result = generator("提取用户信息：张三，25岁，zhang@email.com")
# User(name="张三", age=25, email="zhang@email.com") ← Pydantic 对象

# 方案 2: 正则约束
generator = text_gen.regex(model, r"\d{3}-\d{3}-\d{4}")
result = generator("生成一个电话号码")
# "123-456-7890"

# 方案 3: 选择约束（只能从列表中选择）
generator = text_gen.choice(model, ["晴天", "多云", "下雨"])
result = generator("今天的天气")
# "晴天"
```

### Outlines 的核心机制

1. **FSM 构建**：从 JSON Schema/正则/Pydantic 模型构建有限状态自动机
2. **Token 过滤**：在每一步推理中，根据 FSM 当前状态过滤非法 Token
3. **贪心 + 回溯**：当某个 Token 导致状态机进入死胡同时，回退到上一步

## JsonFormer（专注 JSON）

```python
from json_former import JsonFormer

former = JsonFormer(
    schema={
        "type": "object",
        "properties": {
            "title": {"type": "string"},
            "score": {"type": "number", "minimum": 0, "maximum": 100}
        }
    }
)

# 在推理循环中使用
while not former.is_complete():
    allowed_tokens = former.get_allowed_tokens()
    filtered_logits = mask_logits(logits, allowed_tokens)
    next_token = sample_from(filtered_logits)
    former.update(next_token)
```

### JsonFormer 特点

- 专门为 JSON 格式设计，比通用的 Outlines 更高效
- 支持嵌套 JSON 和复杂的 Schema
- 集成简单——只需要在推理循环中插入几行代码
- 对非 JSON 格式不支持

## lm-format-enforcer（最轻量）

```python
from lm_format_enforcer import CharacterLevelParser

# 正则约束
parser = CharacterLevelParser(regex=r"\w+@\w+\.\w+")

# 在推理循环中使用
allowed_tokens = parser.get_allowed_tokens(prefix_tokens, next_token_logits)
```

## 选择指南

| 需求 | 推荐 | 原因 |
|------|------|------|
| 需要 JSON + 正则 + 自定义 | Outlines | 功能最全面 |
| 只要 JSON，性能优先 | JsonFormer | 专门优化 |
| 简单集成，最低依赖 | lm-format-enforcer | 最轻量 |
| 需要 Pydantic 集成 | Outlines | 原生 Pydantic 支持 |
| 与 vLLM 集成 | Outlines（vLLM 内置） | 内置支持 |

## 核心挑战

1. **性能开销**：filter 所有 Token 的步骤增加了推理延迟（10-30%）
2. **Token 边界问题**：一个 Token 可能跨多个语法单元，过滤逻辑复杂
3. **复杂 Schema 的 FSM 构建**：oneOf, anyOf 等组合 Schema 导致 FSM 状态爆炸
4. **与推理引擎集成**：需要修改模型的推理循环，不同引擎的集成方式不同

## 与托管 API 方案对比

| 维度 | 开源库（自建） | 托管 API（JSON Mode / FC） |
|------|--------------|--------------------------|
| 延迟 | 更高（频繁过滤） | 更低 |
| 可靠性 | 100%（硬约束） | ~95-99% |
| 灵活性 | 极高（任何格式） | 仅限预定义格式 |
| 部署复杂度 | 高（自建推理） | 低（API 调用） |
| 成本 | 取决于硬件 | 按 Token 计费 |

## 工程启示

- 如果需要 100% 的格式保证，约束解码是唯一选择
- 托管 API 的可靠性（95-99%）对于大多数 Agent 场景已经足够
- 约束解码最适合：自建模型、对格式要求极其严格的生产系统
- Outlines 已经集成到 vLLM 的 guided decoding 功能中——如果使用 vLLM，不需要单独集成
- 约束解码在"流式场景"下需要额外处理——部分解码可能导致 Token 级别的延迟
