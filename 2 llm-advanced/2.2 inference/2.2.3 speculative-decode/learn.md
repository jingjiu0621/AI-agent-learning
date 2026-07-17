# Speculative Decoding：草稿模型加速

## 简单介绍

Speculative Decoding（推测解码）是一种"以小博大"的推理加速技术。核心思路是：**用一个更小更快的"草稿模型"先生成多个候选 Token，然后让大模型一次性验证这些 Token 的正确性**。由于验证需要的计算量远小于生成，且验证可以批量处理，整体速度得到显著提升。

## 基本原理

```
标准自回归生成（慢）:
  大模型: 生成 Token₁ → 生成 Token₂ → 生成 Token₃ → ...（逐个生成）

Speculative Decoding（快）:
  草稿模型: 生成 Token₁ → Token₂ → Token₃ → Token₄（快速生成猜测）
  大模型:   ←────── 一次性验证所有猜测 ──────→（批量验证）
  
  如果验证通过 → 接受所有猜测 Token
  如果某步有误 → 回退到该位置，让大模型自己生成
```

### 简化实现

```python
def speculative_decoding(draft_model, target_model, prefix, gamma=5):
    """
    draft_model: 小/快的草稿模型
    target_model: 大/慢的目标模型
    gamma: 每次生成的候选 Token 数量
    """
    # 1. 用草稿模型生成 gamma 个候选 Token
    draft_tokens = []
    for _ in range(gamma):
        token = draft_model.generate_one(prefix + draft_tokens)
        draft_tokens.append(token)
    
    # 2. 用目标模型一次性验证候选 Token
    #    （验证一次前向传播即可获得所有候选 Token 的概率）
    all_tokens = prefix + draft_tokens
    target_logits = target_model.forward(all_tokens)
    
    # 3. 逐个检查候选 Token 是否被目标模型"接受"
    accepted = []
    for i in range(gamma):
        p_draft = draft_model.prob(prefix + accepted, draft_tokens[i])
        p_target = target_model.prob(prefix + accepted, draft_tokens[i])
        
        if random() < min(1, p_target / p_draft):
            # 接受这个 Token
            accepted.append(draft_tokens[i])
        else:
            # 拒绝并从目标模型的分布采样
            adjusted_dist = max(0, p_target - p_draft)
            corrected_token = sample(adjusted_dist)
            accepted.append(corrected_token)
            break
    
    return accepted
```

## 为什么能加速

| 步骤 | 标准解码 | Speculative Decoding |
|------|---------|---------------------|
| 生成 N 个 Token | N 次大模型前向传播 | γ 次小模型前向 + (N/γ) 次大模型验证 |
| 时间 | N × T_large | N × T_small + (N/γ) × T_large |
| 加速比 | 1x | ~γ × (当 T_small << T_large) |

典型场景：γ=4，小模型速度是大模型的 10 倍，加速约 2-3 倍。

## 历史背景

Speculative Decoding 由 Leviathan 等（2022）和 Chen 等（2023）分别独立提出。核心洞察是：**当前 Token 的某些"简单的"预测（如常见的词汇搭配）不需要大模型来处理，小模型就已经足够好**。

## 核心挑战

| 挑战 | 说明 | 缓解方案 |
|------|------|----------|
| 草稿模型质量 | 如果草稿模型总是猜错，加速效果消失 | 用同一系列的大小模型配对 |
| 草稿模型选择 | 需要找到快而准的草稿模型 | 使用目标模型的精简版本 |
| 拒绝开销 | 拒绝后需要额外的采样计算 | 优化拒绝采样逻辑 |
| 硬件利用率 | 小模型可能无法充分利用 GPU | 使用更轻量的架构 |

## 与标准解码对比

| 维度 | 标准自回归 | Speculative Decoding |
|------|-----------|---------------------|
| 生成质量 | 完全一致 | **完全一致**（数学上等价） |
| 延迟 | 高 | 低（典型 2-3x 加速） |
| 实现复杂度 | 低 | 高 |
| 草稿模型需求 | 不需要 | 需要 |
| 适用场景 | 所有 | 批量推理、低延迟场景 |

关键点：Speculative Decoding 的输出分布与标准解码**完全相同**——它不是一个近似方法。

## 工程启示

- Speculative Decoding 最适合批量推理场景和低延迟需求
- 最佳草稿模型是目标模型的"蒸馏小版本"（同一系列），不同系列配对可能效果不好
- γ 值不是越大越好——太大的 γ 会导致草稿模型"猜"太多了错误
- Agent 系统的工具调用场景中，工具名和参数通常不是"常见词汇"——草稿模型可能猜不准，加速效果有限
- 推理服务（vLLM, TensorRT-LLM）已经内置了 Speculative Decoding 支持
