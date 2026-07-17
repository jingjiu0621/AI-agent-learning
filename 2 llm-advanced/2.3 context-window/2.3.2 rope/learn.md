# RoPE 旋转位置编码详解

## 简单介绍

RoPE（Rotary Position Embedding，旋转位置编码）由苏剑林于 2023 年提出，是目前 LLM 领域使用最广泛的位置编码方案（GPT-4、Llama 2/3、Gemma、Mistral、Qwen、Yi 等）。RoPE 的核心思想是**通过旋转矩阵对 Query 和 Key 向量进行变换，在计算注意力分数时自然地引入相对位置信息**。

## 基本原理

### 核心公式

RoPE 不直接给 Embedding 加位置信息，而是旋转 Query 和 Key 向量：

```
标准注意力:
  score(q, k) = q · k

带 RoPE 的注意力:
  score(q_m, k_n) = (R(m)·q) · (R(n)·k) = q · R(m-n)·k
  ↑ 旋转 m 位置的 q    ↑ 旋转 n 位置的 k     ↑ 只依赖于相对位置 m-n

其中 R(m) 是一个由 m 决定的旋转矩阵。
```

### 2D 旋转举例

对于二维向量 [x, y]，旋转角度 θ 的旋转矩阵：

```
R(θ) = [cos θ  -sin θ]
       [sin θ   cos θ]

旋转结果: [x', y'] = [x·cosθ - y·sinθ, x·sinθ + y·cosθ]
```

RoPE 将高维向量分成多个 2D 子空间，每个子空间以不同的角速度旋转：

```python
def rope(x: torch.Tensor, positions: torch.Tensor, base=10000.0):
    """简化的 RoPE 实现"""
    seq_len, d_model = x.shape
    
    # 每个 2D 对的基础频率
    theta = 1.0 / (base ** (torch.arange(0, d_model, 2) / d_model))
    
    # 位置乘以频率
    angles = positions[:, None] * theta[None, :]  # [seq_len, d_model/2]
    
    # 构建旋转后的向量
    cos = torch.cos(angles)[:, :, None]  # [seq_len, d_model/2, 1]
    sin = torch.sin(angles)[:, :, None]
    
    # 将 x 拆成 2D 对
    x_reshape = x.reshape(seq_len, -1, 2)  # [seq_len, d_model/2, 2]
    
    # 应用旋转
    x_rotated = torch.stack([
        x_reshape[..., 0] * cos[..., 0] - x_reshape[..., 1] * sin[..., 0],
        x_reshape[..., 0] * sin[..., 0] + x_reshape[..., 1] * cos[..., 0]
    ], dim=-1)
    
    return x_rotated.reshape(seq_len, d_model)
```

## RoPE 的关键特性

### 1. 相对位置

```
score(q_m, k_n) = f(q_m, m, k_n, n) = g(q_m, k_n, m-n)

注意力分数只依赖于 q、k 和 (m-n) —— 相对位置！
```

### 2. 长距离衰减

RoPE 有一个天然的属性：**两个 Token 之间的距离越远，注意力分数的上限越小**。这符合直觉——远处的 Token 对当前 Token 的影响应该更小。

### 3. 可扩展性

RoPE 支持**位置插值**——将位置索引缩放可以扩展到更长的序列：

```python
# 训练时: 位置最大到 4096
# 推理时: 需要用位置 8192
# RoPE 方案: 将位置 8192 映射到 0-4096 范围内
scale = 4096 / 8192
position = int(original_position * scale)
```

## 与其他方案对比

| 方案 | 相对位置 | 外推能力 | 计算开销 | 代表模型 |
|------|---------|---------|---------|---------|
| Sinusoidal | 否 | 无 | 低 | Transformer |
| Relative Bias | 是 | 有限 | 低 | T5 |
| RoPE | 是 | 好 | 低 | GPT-4, Llama, Mistral |
| ALiBi | 是 | 最好 | 极低 | MPT |

## 能力边界

- RoPE 能外推到比训练长度更长的序列，但长距离下的注意力质量会下降
- 需要配合位置插值或 NTK-aware 等方法实现大规模窗口扩展
- base 频率（通常是 10000）的选择影响长距离性能——更大的 base 值有助于长上下文

## 工程启示

- RoPE 在模型推理时完全在"算数层面"工作，不需要额外的表查找或存储——实现高效
- 当前主流模型的 RoPE 实现细节略有差异（base 值、旋转维度等），不能直接互换
- 在 Agent 系统的长上下文场景中，RoPE + YaRN/NTK 扩展是目前最实用的方案
- RoPE 的 fine-tuning 不改变位置编码的数学形式——训练时只需在更长序列上继续训练即可适应
