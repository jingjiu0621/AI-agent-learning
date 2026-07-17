# 上下文压缩（LLMLingua, Selective Context）

## 简单介绍

上下文压缩是在不丢失核心信息的前提下，减小输入 Token 数量的技术。与窗口扩展（延长模型能看到的长度）不同，压缩是"让输入更精简"。这在 Agent 系统中特别有价值——长历史对话、长文档、多工具结果都可能被压缩。

## 为什么需要压缩

```
Agent 上下文组成:
  系统 Prompt:      500 Tokens
  工具定义:        2000 Tokens
  对话历史:        5000 Tokens
  当前查询:         100 Tokens
  工具结果:        3000 Tokens
  ──────────────────────
  总计:           10600 Tokens  ← 在 8K 窗口中就超了
```

## 压缩方案

### 1. 选择性上下文（Selective Context / Token 裁剪）

直接删除"不重要"的 Token 或句子：

```python
class SelectiveCompressor:
    """基于重要性评分的 Token 裁剪"""
    
    def __init__(self, model):
        self.model = model
    
    def compress(self, text: str, compression_ratio: float = 0.5) -> str:
        tokens = self.model.tokenizer.encode(text)
        
        # 1. 对每个 Token 计算"如果删除这个 Token"的困惑度变化
        importance_scores = []
        for i in range(len(tokens)):
            mask = tokens[:i] + tokens[i+1:]
            score = self.model.perplexity_difference(tokens, mask)
            importance_scores.append(score)
        
        # 2. 删除重要性最低的 Token
        n_remove = int(len(tokens) * compression_ratio)
        important_indices = np.argsort(importance_scores)[-len(tokens):]
        
        compressed_tokens = [tokens[i] for i in sorted(important_indices)]
        return self.model.tokenizer.decode(compressed_tokens)
```

### 2. LLMLingua（微软, 2023）

使用一个小模型（如 GPT-2 Small）来评估每个 Token 的重要性，然后删除不重要的 Token：

```
原始文本: "今天天气很好，我想去公园散步。"
重要性评分:
  "今天"  → 0.9
  "天气"  → 0.8
  "很好"  → 0.7
  "，"    → 0.1 ← 删除
  "我想"  → 0.8
  "去"    → 0.6
  "公园"  → 0.9
  "散步"  → 0.9
  "。"    → 0.1 ← 删除

压缩后: "今天天气很好我想去公园散步" (Token 减少约 20%)
```

**优势**：保留语义核心，压缩比高（40-60%）
**劣势**：需要独立的 Token 重要性评估模型，增加延迟

### 3. 摘要式压缩（使用 LLM）

让 LLM 对长内容进行摘要：

```python
def summarize_history(history: str, max_tokens: int = 500) -> str:
    return llm.chat(f"""请将以下对话历史压缩为不超过 {max_tokens} Token 的摘要。
保留关键信息：用户的意图、Agent 已经执行的操作、已经获取的结果。
对话历史：\n{history}""")
```

**优势**：压缩比极高（可达 80-90%）
**劣势**：可能丢失关键细节

## 压缩策略对比

| 方法 | 压缩比 | 信息保留 | 延迟 | 适用场景 |
|------|--------|---------|------|---------|
| Token 裁剪 | 20-40% | 中 | 低 | 工具结果、日志 |
| LLMLingua | 40-60% | 高 | 中 | 长文档 |
| 摘要压缩 | 60-90% | 中 | 高 | 历史对话 |
| 分层摘要 | 80-90% | 高 | 高 | 超长上下文 |

## 核心挑战

- **信息丢失的诊断**：压缩后如何知道是否丢失了关键信息？不丢失核心信息是最大的挑战
- **压缩比的动态调整**：不同的内容适合不同的压缩比——固定的压缩率不适合所有情况
- **引用准确度**：压缩可能导致数字、引用、名称等精确信息的丢失

## 工程启示

- 在 Agent 系统中，**工具结果比历史对话更适合压缩**——工具结果通常有大量结构化冗余数据
- 采用"分层"策略：最近 2-3 轮完整保留，之前的内容逐步压缩
- 压缩前先检查 Token 预算是否确实需要压缩——不必要的压缩影响 Agent 表现
- 压缩后加入"[以下内容经过压缩]"的标记，让 LLM 知道它看到的不是原始数据
- 关键数据（用户 ID、数字、日期）应该避免被压缩或裁剪
