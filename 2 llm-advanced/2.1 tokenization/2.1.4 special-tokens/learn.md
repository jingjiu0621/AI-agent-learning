# Special Tokens 的作用与设计

## 简单介绍

Special Tokens（特殊 Token）是 LLM 词表中具有特定语义功能的预留 Token。它们不表示具体的词语，而是作为控制信号——标记文本结构、切换角色、指示边界、触发特定行为。理解 Special Tokens 对构建 Agent 系统时的 Prompt 设计和消息格式化至关重要。

## 常见 Special Tokens

### 1. 序列边界标记

```
Text Token:
  GPT: <|endoftext|>  → 文本结束 / 序列分隔
  Llama: <s>          → 序列开始
  Llama: </s>         → 序列结束
  BERT: [CLS]         → 分类标记（放在开头）
  BERT: [SEP]         → 分隔标记（放在句子之间）
```

**作用**：告诉模型"这是一段独立的文本"或"两个句子在这里分开"。

### 2. 角色标记（Chat Model 专用）

```
GPT 的 ChatML:
  <|im_start|>system   → 系统消息开始
  <|im_start|>user     → 用户消息开始
  <|im_start|>assistant → 助手消息开始
  <|im_end|>           → 任何消息结束

Llama 的对话格式:
  <<SYS>>              → 系统消息包裹
  [INST]               → 用户指令开始
  [/INST]              → 用户指令结束
```

**作用**：区分对话中不同的角色和消息边界——这是 Agent 多轮对话的基础机制。

### 3. 功能标记

```
GPT 的辅助标记:
  <|tool_call|>        → 函数调用开始  
  <|tool_result|>      → 函数调用结果

Claude 的 Content Block:
  没有显式 Special Token，通过 JSON 结构区分
```

**作用**：指示 LLM 何时开始输出工具调用，何时输出普通文本。

### 4. 填充和掩码标记

```
BERT: [MASK]          → 被掩盖的词（用于 MLM 训练）
BERT: [PAD]           → 填充标记（用于批量对齐长度）
T5: <extra_id_0>      → 跨度损坏的占位符
```

## Special Tokens 的设计原则

| 原则 | 说明 | 反例 |
|------|------|------|
| 可区分性 | Special Token 不能与普通文本混淆 | 如果普通文本包含 "<|endoftext|>"，模型会误解 |
| 完整性 | 成对出现的 Tag 必须配对 | 有 [INST] 必须有 [/INST] |
| 词表预留 | Special Token 在训练时就在词表中 | 不能在训练后新增（需扩展词表） |
| 唯一性 | 不同功能的 Token 不应冲突 | <s> 不能同时表示开始和结束 |

## 对 Agent 系统的影响

```python
# 构建 Agent 的多轮对话时，需要理解底层 Special Tokens
def build_chat_messages(messages, tokenizer):
    """将消息格式化为 LLM 可理解的 Token 序列"""
    formatted = ""
    for msg in messages:
        role = msg["role"]
        content = msg["content"]
        # ChatML 格式
        formatted += f"<|im_start|>{role}\n{content}<|im_end|>\n"
    formatted += "<|im_start|>assistant\n"
    return tokenizer.encode(formatted)
```

## 工程启示

- 使用框架（如 LangChain、OpenAI SDK）时，框架会处理 Special Tokens 的注入——不需要手动处理
- 如果直接调用 API（原始 HTTP 请求），需要确保消息格式与模型的预期一致
- 不同模型的 Special Token 设计不同——同一个 Prompt 在不同模型上效果不同，部分原因就是 Special Tokens 不同
- 在流式响应中，Special Tokens 可能会作为单独的 Chunk 出现——解析时需要正确处理
- **不要在用户输入中包含 Special Tokens**——这可能导致注入攻击或格式破坏，应过滤或转义
