# 自回归生成原理：逐 Token 预测

## 简单介绍

自回归（Auto-regressive）生成是目前所有主流 LLM（GPT、Claude、Llama 等）使用的基本推理方式。它的核心思想极其简单：**一次生成一个 Token，并将已生成的 Token 作为输入来预测下一个 Token**。这种"迈一步看一步"的方式决定了 LLM 推理的根本特征——不能并行生成。

## 基本原理

### 数学表达

自回归模型将序列的联合概率分解为条件概率的乘积：

```
P(序列) = P(Token₁) × P(Token₂|Token₁) × P(Token₃|Token₁,Token₂) × ...

即: P(T₁, T₂, ..., Tₙ) = ∏P(Tₖ | T₁, T₂, ..., Tₖ₋₁)
```

### 生成过程

```python
def generate_greedy(prompt: str, model, max_tokens: int = 100) -> str:
    """贪心自回归生成"""
    input_ids = tokenizer.encode(prompt)
    
    for _ in range(max_tokens):
        # 1. 模型前向传播 → 计算下一个 Token 的概率分布
        logits = model.forward(input_ids)
        next_token_logits = logits[-1]  # 只看最后一个位置
        
        # 2. 选择概率最高的 Token（贪心策略）
        next_token_id = argmax(next_token_logits)
        
        # 3. 将新 Token 追加到序列
        input_ids.append(next_token_id)
        
        # 4. 检查终止条件
        if next_token_id == EOS_TOKEN_ID:
            break
    
    return tokenizer.decode(input_ids[len(prompt):])
```

### 解码策略

自回归的"选择下一个 Token"有多种策略：

| 策略 | 说明 | 特点 |
|------|------|------|
| Greedy Decoding | 每次选概率最高的 Token | 确定性、可能重复 |
| Temperature Sampling | 按概率分布随机采样 | 多样性、可控 randomness |
| Top-K Sampling | 只从概率最高 K 个 Token 中采样 | 控制候选范围 |
| Top-P (Nucleus) Sampling | 累积概率达到 P 的 Token 集合中采样 | 动态候选数量 |
| Beam Search | 维护多个候选序列 | 质量高、计算量大 |

## 历史背景

自回归语言模型的根源可以追溯到 2013 年的 Word2Vec 和 2015 年的 LSTM 语言模型。2017 年 Transformer 架构提出后，2018 年的 GPT-1 证明了自回归 Transformer 的有效性。2019-2020 年 GPT-2/GPT-3 展示了"规模化"的威力。

之前的方法（机器翻译时代）使用 Encoder-Decoder 架构，源语言一次性编码，目标语言逐词输出——本质上也是自回归，但受到上下文长度的限制。

## 核心矛盾

| 优势 | 劣势 |
|------|------|
| 数学形式简洁优雅 | 无法并行生成（逐个 Token） |
| 训练与推理一致性（Teacher Forcing） | 推理延迟随 Token 数线性增长 |
| 灵活性高（可以控制生成过程） | 长序列时前面的错误会累积 |

最大的矛盾是：**自回归生成需要大量连续的计算，但 GPU 是为大规模并行计算设计的**——每个 Token 只能用一个微小的"步骤"来计算。

## 自回归 vs 非自回归

| 维度 | 自回归 | 非自回归（如 Masked LM） |
|------|--------|------------------------|
| 生成方式 | 从左到右逐个生成 | 一次全部生成（或迭代式） |
| 质量 | 高（每个 Token 基于完整上下文） | 中（缺乏从左到右的依赖建模） |
| 速度 | 慢 | 快 |
| 代表模型 | GPT, Llama, Claude | BERT, T5 |

## 工程启示

- 自回归生成造成的延迟是 Agent 系统必须接受的事实——可以通过 streaming 来改善用户体验
- 长输出时，前期的错误 Token 会"污染"后续生成——Reflexion 等纠错机制很有价值
- 使用更小的模型或更短的输出可以显著降低延迟
- Temperature 参数对工具调用的准确性影响很大——工具调用场景推荐用较低的 Temperature (0-0.3)
