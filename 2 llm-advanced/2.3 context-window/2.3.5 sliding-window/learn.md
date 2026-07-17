# Sliding Window Attention

## 简单介绍

Sliding Window Attention（滑动窗口注意力）是降低长序列注意力计算复杂度的一种方法。核心思想是：**每个 Token 只与它邻近的 W 个 Token 计算注意力，而不是与序列中的所有 Token**。这使得注意力的计算复杂度从 O(n²) 降到 O(n×W)，其中 W 是窗口大小（固定值）。

## 基本原理

```
标准注意力（Full Attention）:
  Token₀ 注意: Token₀ Token₁ Token₂ Token₃ ... Tokenₙ₋₁
  Token₁ 注意: Token₀ Token₁ Token₂ Token₃ ... Tokenₙ₋₁
  Token₂ 注意: Token₀ Token₁ Token₂ Token₃ ... Tokenₙ₋₁
  ...
  → 每个 Token 都要看所有其他 Token ❌ O(n²)

滑动窗口注意力（W=3）:
  Token₀ 注意: Token₀ Token₁ Token₂ | ...
  Token₁ 注意: Token₀ Token₁ Token₂ Token₃ | ...
  Token₂ 注意: Token₀ Token₁ Token₂ Token₃ Token₄ | ...
  Token₃ 注意: | Token₁ Token₂ Token₃ Token₄ Token₅ | ...
  Token₄ 注意: | Token₂ Token₃ Token₄ Token₅ Token₆ | ...
  ...
  → 每个 Token 只看前后 W 个 Token ✅ O(n×W)
```

### 注意力掩码

```python
def create_sliding_window_mask(seq_len: int, window_size: int) -> torch.Tensor:
    """创建滑动窗口注意力掩码"""
    mask = torch.zeros(seq_len, seq_len)
    for i in range(seq_len):
        start = max(0, i - window_size // 2)
        end = min(seq_len, i + window_size // 2 + 1)
        mask[i, start:end] = 1
    return mask
```

## 为什么 W 可以很小

直观理解：**语言是局部结构化的**——一个词主要受它附近的词影响。"我今天去**公园**散步"中，"公园"主要受"去"和"散步"的影响，而不是开头的"我"。

这让滑动窗口在大多数场景下都能很好地工作。

## 变体：Dilated Sliding Window

使用空洞（dilation）来扩大有效感受野：

```
层级 1: 窗口 W=3, dilation=1 → 感受野 3
层级 2: 窗口 W=3, dilation=2 → 感受野 5  
层级 4: 窗口 W=3, dilation=4 → 感受野 9
...

通过堆叠多个层级，感受野呈指数增长，而计算量保持线性。
```

## 适用场景

| 场景 | 滑动窗口效果 | 说明 |
|------|-------------|------|
| 代码生成 | 好 | 代码的局部依赖很强 |
| 对话 | 好 | 主要依赖最近几轮对话 |
| 机器翻译 | 好 | 句子内部处理 |
| 长文档问答 | 差 | 问题的答案可能在文档开头 |
| 多步推理 | 差 | 推理链可能跨越长距离 |

**滑动窗口对 Agent 系统的影响**：如果 Agent 需要在长对话前期提到的事实来支持后期的决策，滑动窗口可能会"忘记"那些信息。

## 代表模型

| 模型 | 窗口大小 | 实现方式 |
|------|---------|----------|
| Mistral 7B | 4096 | Sliding Window Attention |
| MPT | 2048 | ALiBi + 滑动窗口 |
| Longformer | 512-4096 | 全局+滑动窗口混合 |
| BigBird | 动态 | 全局+随机+滑动窗口 |

Mistral 7B 的 Sliding Window 特别值得关注：它使用 4096 的窗口，但通过多层堆叠实现了 131K 的有效感受野。

## 核心挑战

1. **长距离依赖丢失**：窗口之外的 Token 完全看不到
2. **KV Cache 无法复用**：滑动窗口的 KV Cache 随着窗口的滑动不断替换，不能简单的 Prefix Caching
3. **窗口大小调优**：窗口太小丢失信息，窗口太大失去优化意义

## 工程启示

- 对于 Agent 系统，纯滑动窗口不适合需要"回忆早期信息"的场景
- 混合方案（全局 Token + 滑动窗口）可能是更好的选择——关键信息用全局注意力，普通信息用滑动窗口
- Mistral 7B 的叠层方案有效解决了感受野问题，可以借鉴其思路
- 如果使用滑动窗口模型，Agent 应该主动将关键信息"推到窗口内"（通过摘要或重复）
- 注意：大多数主流模型（GPT-4, Claude, Llama）不使用纯滑动窗口，而是使用 Full Attention
