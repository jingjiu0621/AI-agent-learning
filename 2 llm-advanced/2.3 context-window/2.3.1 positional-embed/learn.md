# Positional Embedding 演进：绝对 → 相对 → RoPE

## 简单介绍

位置编码（Positional Embedding）是 Transformer 模型中注入"词语顺序信息"的机制。由于 Transformer 的自注意力机制本身是**置换不变的**（对输入的顺序不敏感），所以需要通过位置编码告诉模型"第一个词是哪个、第二个词是哪个"。位置编码的演进史，就是 LLM 上下文窗口扩展的历史。

## 基本原理

### 为什么需要位置编码

```
自注意力对输入:
  "我 爱 你" → 注意力计算时，这三个词的位置是等价的
  "你 爱 我" → 注意力计算仍然等价于上一句
  
没有位置编码，模型无法区分 "我打你" 和 "你打我" 的区别。
```

## 三种位置编码方案

### 1. 绝对位置编码（Absolute）

最早的方案，给每个位置分配一个固定的 Embedding 向量：

```python
# 简化的绝对位置编码
def absolute_positional_encoding(seq_len, d_model):
    """Sinusoidal 位置编码（Transformer 原版）"""
    pe = torch.zeros(seq_len, d_model)
    for pos in range(seq_len):
        for i in range(0, d_model, 2):
            pe[pos, i] = sin(pos / 10000 ** (i / d_model))
            pe[pos, i+1] = cos(pos / 10000 ** (i / d_model))
    return pe

# 使用：直接加到输入 Embedding 上
x = token_embedding + positional_encoding
```

**问题**：
- 训练时只看到了 512/1024 个位置编码，遇到更长的序列没有对应的编码
- 无法外推到比训练时更长的序列

### 2. 相对位置编码（Relative）

不记录"位置 i"，而是记录"位置 i 和位置 j 之间的偏移量"：

```
输入: "A B C D"
绝对编码: A:pos=0, B:pos=1, C:pos=2, D:pos=3
相对编码: B→A:offset=-1, B→C:offset=1, B→D:offset=2
```

**优势**：模型学到了"相邻词的关系"而不是"第 N 个位置的语义"——对序列长度变化更鲁棒。

### 3. RoPE（Rotary Position Embedding）

RoPE 是目前最为主流的位置编码方式（GPT-4, Llama, Mistral, qwen 等都在使用）。

| 方案 | 代表模型 | 上下文窗口 | 外推能力 |
|------|---------|-----------|---------|
| Sinusoidal (绝对) | Transformer 原版 | 512 | 无 |
| Learned Absolute | BERT, GPT-2 | 512-1024 | 无 |
| Relative Bias | T5 | 512 | 有限 |
| RoPE | GPT-4, Llama, Mistral | 4K-128K | 有（需扩展） |
| ALiBi | MPT | 4K-8K | 好 |

## 核心矛盾

- **绝对编码**：简单直接，但无法扩展序列长度
- **相对编码**：关系不变性更好，但实现复杂
- **RoPE**：旋转相对位置，兼顾性能和扩展性——目前的最优选择

## 工程启示

- RoPE 是目前 Agent 系统涉及的模型中最常见的位置编码方案
- RoPE 虽然支持"理论上的外推"，但直接外推到训练长度之外质量会下降——需要 YaRN 等扩展方法
- 位置编码的选择直接决定了模型能处理的上下文长度上限
- 对于 Agent 系统的长上下文场景，优先选择支持 RoPE + 窗口扩展技术的模型（GPT-4-128K, Claude 200K）
