# Flash Attention：IO 感知的注意力算法

## 简单介绍

Flash Attention 是一种 IO 感知（IO-aware）的精确注意力算法，由 Tri Dao 等人于 2022 年提出。它不是近似注意力（如 Sparse Attention），而是计算**完全精确**的标准注意力，但通过分块（Tiling）技术和重计算来大幅减少 GPU 高带宽内存（HBM）的读写。在实践中，Flash Attention 可以将注意力计算的推理速度提升 2-4 倍。

## 基本原理

### 核心洞察：计算比内存访问更便宜

GPU 有不同类型的存储：

```
GPU 计算单元（SM）
   ↑↓  极快（~20 TB/s）
片上 SRAM（共享内存，~192KB）
   ↑↓  较慢（~2 TB/s）
HBM（全局显存，~80GB）
```

标准注意力实现的问题是：**频繁地在 HBM 和 SRAM 之间搬运大矩阵**——计算本身快，但数据搬运慢。

### 标准注意力 vs Flash Attention

```
标准注意力:
  1. 从 HBM 读取 Q, K          ← 慢
  2. 计算 S = Q·K^T（在 SRAM）
  3. 将 S 写回 HBM             ← 慢
  4. 从 HBM 读取 S             ← 慢
  5. 计算 P = softmax(S)（在 SRAM）
  6. 将 P 写回 HBM             ← 慢
  7. 从 HBM 读取 P             ← 慢
  8. 计算 O = P·V（在 SRAM）
  9. 将 O 写回 HBM             ← 慢
  → HBM 读写次数: O(N² + N²d) ❌

Flash Attention:
  1. 将 Q, K, V 分块（Tiles）
  2. 每个块在 SRAM 中:
     - 计算局部 S = Q_block · K_block^T
     - 在线 softmax（无需等待全局）
     - 计算局部 O = softmax(局部) · V_block
  3. 累加所有块的局部结果
  → HBM 读写次数: O(N²d / M) ✅（M = SRAM 大小）
```

### 数学技巧：在线 Softmax

标准 softmax 需要知道所有元素才能计算：

```
softmax(x_i) = exp(x_i) / Σexp(x_j)
```

Flash Attention 通过"在线 softmax"（Online Softmax/RESCALE）技巧，可以在只看到部分数据时就计算：先计算局部 softmax，遇到新数据时重新缩放之前的计算结果。

## 性能对比

| 序列长度 | 标准注意力 (ms) | Flash Attention (ms) | 加速比 |
|---------|----------------|---------------------|--------|
| 1K | 2.1 | 1.1 | 1.9x |
| 2K | 8.3 | 2.9 | 2.9x |
| 4K | 33.0 | 7.8 | 4.2x |
| 8K | 138.0 | 22.0 | 6.3x |
| 16K | 560.0 | 65.0 | 8.6x |

序列越长，加速效果越明显（因为 HBM 访问的减少效果与序列长度平方相关）。

## 历史背景

在 Flash Attention 之前，注意力计算的优化主要集中在"减少计算量"——Sparse Attention, Reformer, Linformer 等近似方法。这些方法虽然加速了，但牺牲了注意力矩阵的完整性。

Flash Attention 的核心贡献是：**'不减少计算量，而是减少不必要的数据搬运'**。优化数据传输模式比减少计算更有效。

## Flash Attention v2/v3

| 版本 | 改进 | 提速 |
|------|------|------|
| v1 (2022) | 基本 IO 感知分块 | 2-4x |
| v2 (2023) | 减少非矩阵乘操作、更好的并行 | 1.5-2x over v1 |
| v3 (2024) | Hopper GPU 的异步处理 | 1.5-2x over v2 |

## 核心挑战

1. **硬件绑定**：Flash Attention 需要针对每种 GPU 架构编写 CUDA 内核
2. **分块大小选择**：块大小太大放不进 SRAM，太小管理开销大
3. **算法复杂度高**：在线 softmax 的并行化实现比较 tricky

## 工程启示

- Flash Attention 现在已经是主流训练/推理框架的标准组件（PyTorch 2.0+ 内置）
- Agent 系统中处理长上下文（大型文档、长历史）时，Flash Attention 带来的加速约 4-8x
- Flash Attention 不改变模型输出——是完全精确的，所以不需要担心兼容性问题
- Flash Attention 对 KV Cache 的压缩和量化不产生任何影响，独立优化
