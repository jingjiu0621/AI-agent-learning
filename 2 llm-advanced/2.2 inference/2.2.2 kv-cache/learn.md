# KV Cache 机制与显存优化

## 简单介绍

KV Cache（Key-Value 缓存）是 LLM 推理中最基础也最重要的优化技术。在自回归生成过程中，每生成一个新 Token，模型都需要计算当前所有 Token 之间的注意力。KV Cache 通过缓存历史 Token 的 Key 和 Value 矩阵，避免了每次生成时对历史 Token 的重复计算——将注意力计算复杂度从 O(n³) 降低到 O(n²)（按序列长度计）。

## 基本原理

### 不缓存时的问题

假设我们要生成长度为 N 的序列：

```
生成第 1 个 Token: 计算 1 个位置的所有注意力 → 计算量 O(1²)
生成第 2 个 Token: 重新计算前 2 个位置的所有注意力 → O(2²)
生成第 3 个 Token: 重新计算前 3 个位置的所有注意力 → O(3²)
...
生成第 N 个 Token: 重新计算前 N 个位置的所有注意力 → O(N²)

总计算量: O(1² + 2² + 3² + ... + N²) = O(N³) ❌
```

### KV Cache 如何优化

每次生成新 Token 时，只计算**新 Token 的 Key 和 Value**，并缓存历史 Token 的 K/V：

```
生成第 1 个 Token: 计算 K₁,V₁ → 缓存 [K₁,V₁]
生成第 2 个 Token: 计算 K₂,V₂ → 缓存 [K₁,V₁,K₂,V₂]
                    注意力只计算新 Token 的 Query 与所有缓存的 Key/Value
生成第 3 个 Token: 计算 K₃,V₃ → 缓存 [K₁,V₁,K₂,V₂,K₃,V₃]
...

每步计算量: O(N)（只计算最新的 Query 与所有 K/V 的注意力）
总计算量: O(1 + 2 + 3 + ... + N) = O(N²) ✅
```

### 简化的实现

```python
class AttentionWithKVcache:
    def __init__(self):
        self.k_cache = []  # 缓存的 Key 
        self.v_cache = []  # 缓存的 Value
    
    def step(self, x_t):
        """生成第 t 个 Token 时的注意力计算"""
        # Q: 只计算当前 Token 的 Query
        q_t = self.W_q(x_t)
        
        # K, V: 计算当前 Token 的 Key 和 Value
        k_t = self.W_k(x_t)
        v_t = self.W_v(x_t)
        
        # 缓存
        self.k_cache.append(k_t)
        self.v_cache.append(v_t)
        
        # 注意力：当前 Q 与所有缓存的 K, V
        K = stack(self.k_cache)  # [t, d_k]
        V = stack(self.v_cache)  # [t, d_v]
        
        attn_weights = softmax(q_t @ K.T / sqrt(d_k))
        output = attn_weights @ V
        
        return output
```

## 显存占用分析

KV Cache 的显存占用是推理中的主要瓶颈：

```
KV Cache 大小 = 2 × n_layers × n_heads × d_head × seq_len × dtype_bytes

示例（Llama 2 7B）:
  n_layers = 32
  n_heads = 32
  d_head = 128
  seq_len = 4096
  dtype = FP16 (2 bytes)
  
  每层 KV Cache = 2 × 32 × 128 × 4096 × 2 = 64 MB
  总 KV Cache = 64 MB × 32 层 ≈ 2 GB
```

## 优化策略

| 策略 | 效果 | 实现难度 |
|------|------|----------|
| Multi-Query Attention (MQA) | 多 Query 共享 K/V，减少缓存量 | 低（架构级） |
| Grouped-Query Attention (GQA) | MQA 和 MHA 的折中 | 中 |
| KV Cache 量化 | 用 INT8/INT4 存储 Cache | 中 |
| PagedAttention (vLLM) | 分页式 KV Cache 管理，减少碎片 | 高 |
| Prefix Caching | 共享相同前缀的 KV Cache | 中 |

## 工程启示

- KV Cache 是"用显存换算力"的经典例子——更大的 Cache = 更快的推理，但需要更多显存
- 长上下文推理时（如处理 128K 文档），KV Cache 的显存占用会远超模型权重本身
- 在生产环境中，KV Cache 的内存管理是推理引擎（vLLM, TensorRT-LLM）的核心优化点
- Agent 系统中，如果 Agent 需要处理大量历史信息，KV Cache 的显存消耗需要纳入资源规划
- 并行批处理时，不同请求的序列长度不同，KV Cache 的大小也不同——需要 Padding 或特殊管理
