# 约束解码：Grammar-constrained 生成

## 简单介绍

约束解码（Constrained Decoding / Grammar-constrained Generation）通过**在模型生成 Token 的过程中实时干预 Token 选择**，确保输出**100% 符合预定义的语法或格式**。与 JSON Mode 的"软约束"不同，约束解码是"硬约束"——非法 Token 被直接屏蔽，模型只能从合法 Token 中选择。

## 基本原理

### 对比：无约束 vs 约束解码

```
无约束生成:
  模型可以生成任何 Token...
  → {"name": "John"   ← 可能输出非 JSON
  → 可能输出 markdown ```json
  → 字段名可能与预期不符

约束解码生成:
  每一步只允许生成"符合 Schema 的 Token"
  → 解析: 当前在 "object" 上下文中
  → 预期下一个 Token: "{" 或 空格
  → 屏蔽所有其他 Token
  → 100% 输出合法 JSON
```

### 实现机制

```python
class ConstrainedDecoder:
    """简化的约束解码器"""
    def __init__(self, grammar: dict):
        self.grammar = grammar  # JSON Schema 或自定义语法
    
    def get_allowed_tokens(self, partial_output: str) -> list[int]:
        """根据当前部分输出和语法，返回允许的 Token ID 列表"""
        # 1. 解析部分输出，确定当前在语法中的位置
        state = self._parse_state(partial_output)
        
        # 2. 基于语法，找出所有合法的下一个符号
        allowed_chars = self._next_allowed(state)
        
        # 3. 将合法符号映射到 Token ID
        allowed_tokens = []
        for token_id, token_str in self.tokenizer.vocab.items():
            # 检查该 Token 是否以合法字符开头
            if any(token_str.startswith(c) for c in allowed_chars):
                # 还需要检查 Token 的后续部分不会导致非法
                if self._is_valid_prefix(partial_output + token_str):
                    allowed_tokens.append(token_id)
        
        return allowed_tokens
    
    def generate(self, model, prompt: str):
        """约束解码生成"""
        output = ""
        while True:
            logits = model.forward(output)
            allowed = self.get_allowed_tokens(output)
            
            # 屏蔽非法 Token 的 logits
            masked_logits = logits.clone()
            mask = torch.ones(len(logits), dtype=torch.bool)
            mask[allowed] = False
            masked_logits[mask] = -float('inf')
            
            # 从合法 Token 中选择
            next_token = sample(masked_logits)
            output += self.tokenizer.decode(next_token)
            
            if self._is_complete(output):
                break
        return output
```

## 代表性方案

### Outlines

Python 库，支持 JSON Schema、正则表达式、上下文无关语法：

```python
from outlines import generate, models

model = models.transformers("mistralai/Mistral-7B")

# JSON Schema 约束
schema = """{
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {"type": "integer", "minimum": 0}
    },
    "required": ["name", "age"]
}"""

generator = generate.json(model, schema)
result = generator("提取用户信息：张三，25 岁")
# {"name": "张三", "age": 25}  ← 100% 符合 Schema
```

### lm-format-enforcer

基于有限状态机的格式强制：

```python
from lm_format_enforcer import CharacterLevelParser

parser = CharacterLevelParser(json_schema=my_schema)
allowed_tokens = parser.get_allowed_tokens(logits)
```

## 约束解码 vs Function Calling

| 维度 | Function Calling | Constrained Decoding |
|------|-----------------|---------------------|
| 100% 合规 | 否（~95-99%） | 是（100%） |
| 实现位置 | API 层面 | 客户端/推理引擎 |
| 流式支持 | 是 | 部分 |
| 灵活性 | 仅函数调用 | 任何语法 |
| 模型兼容 | 所有 FC 模型 | 所有自回归模型 |
| 延迟 | 低 | 中（额外的 Token 过滤） |

## 核心挑战

1. **Token 边界问题**：合法的下一个字符可能跨 Token——一个 Token 的开头合法但结尾不合法，需要回退
2. **性能开销**：每一步都需要检查所有 Token 的合法性（词表大小约 50K-100K）
3. **复杂语法**：JSON Schema 的嵌套、组合（oneOf, anyOf）需要复杂的有限状态自动机
4. **流式支持**：约束解码在流式推理中需要特殊的处理逻辑

## 工程启示

- 约束解码是追求"100% 可靠性"的生产系统的终极选择
- 但增加了推理延迟（约 10-30%），需要在可靠性和速度间权衡
- Outlines 是目前最成熟的约束解码库
- 对 Agent 系统来说，约束解码最适合：工具参数输出、中间推理步骤格式化
- 如果使用托管 API（OpenAI, Claude），无法使用约束解码——只能使用 FC 或 JSON Mode
- 如果自建推理（vLLM, TGI），vLLM 已经内置了 guided decoding（基于 Outlines）
