# tiktoken 库与 Token 精确计数

## 简单介绍

tiktoken 是 OpenAI 开源的超高速 Tokenizer 库，用 Rust 实现，Python 绑定。它是 GPT-4、GPT-3.5、Claude 等模型的底层 Tokenizer 引擎。理解 tiktoken 的使用对于 Agent 开发中的 Token 预算管理、成本控制和上下文窗口管理至关重要。

## 基本原理

tiktoken 的核心是 BPE（Byte Pair Encoding）的快速实现：

```
文本 → BPE 编码 → Token ID 列表 → 每个 Token 对应词表中的 ID
```

### 基础用法

```python
import tiktoken

# 获取特定模型的 Tokenizer
enc = tiktoken.encoding_for_model("gpt-4")  # 或 gpt-3.5-turbo, text-embedding-3-small

# 编码：文本 → Token IDs
tokens = enc.encode("Hello, World!")
# → [15496, 11, 995, 0]

# 解码：Token IDs → 文本
text = enc.decode(tokens)
# → "Hello, World!"

# 计数
count = len(tokens)  # 4 个 Token
```

### 不同模型使用不同的 Encoding

| 模型 | Encoding 名称 | 特点 |
|------|--------------|------|
| GPT-4, GPT-4o | cl100k_base | 50,257 词表 |
| GPT-3.5-turbo | cl100k_base | 同上 |
| GPT-3 (ada, babbage) | r50k_base | 更旧的编码 |
| text-embedding-3-small | cl100k_base | 同上 |
| text-davinci-003 | p50k_base | 中间版本 |
| Claude（Anthropic） | 自定义 BPE | 非 OpenAI 标准 |

## 历史背景

在 tiktoken 之前，Token 计数主要靠**估算**：
- 英文："1 Token ≈ 0.75 单词"
- 中文："1 Token ≈ 1.5-2 个汉字"

这些估算在生产级场景下误差太大，因为不同文本的 Token 密度差异极大：
```
"Hello, World!" → 4 Tokens
"Hello, World, this is a test of token counting" → 8 Tokens

"你好世界" → 3-4 Tokens（取决于模型）
"抗噪能力出色鲁棒性好" → 特殊领域术语可能消耗更多 Token
```

## 核心用处：Agent 的 Token 预算管理

```python
class TokenBudget:
    def __init__(self, model: str, max_tokens: int = 128000):
        self.enc = tiktoken.encoding_for_model(model)
        self.max_tokens = max_tokens
        self.history_tokens = 0
    
    def count(self, text: str) -> int:
        """精确计算文本的 Token 数"""
        return len(self.enc.encode(text))
    
    def can_add(self, text: str) -> bool:
        """检查是否能在预算内添加新内容"""
        needed = self.count(text)
        return (self.history_tokens + needed) <= self.max_tokens
    
    def truncate_to_fit(self, text: str, budget: int) -> str:
        """在 Token 预算内截断文本"""
        tokens = self.enc.encode(text)
        return self.enc.decode(tokens[:budget])
```

## 能力边界

1. **只能计算 Token 数，不能分割语义**：截断时可能切断在句子中间
2. **不同模型的 Encoding 不同**：必须在同一个模型系列内使用
3. **Token 计数 != API 输入计数**：API 输入包含 system prompt、历史消息等额外开销

## 与其他方案对比

| 方案 | 速度 | 准确度 | 适用 |
|------|------|--------|------|
| tiktoken | 极快（Rust） | 精确 | OpenAI 模型 |
| huggingface tokenizers | 快（Rust） | 精确 | HuggingFace 模型 |
| Anthropic Token Counter | 快 | 精确 | Claude 模型 |
| 字数估算 | 即时 | 粗略 | 原型开发 |

## 工程启示

- Agent 系统中，每次调用 LLM 之前都应该精确计算 Token 消耗
- Token 预算需要预留 10-20% 的 buffer，因为输出 Token 也需要计数
- 用 tiktoken 实现"安全截断"时，注意截断的 token 序列可能无法完美解码为人类可读文本
