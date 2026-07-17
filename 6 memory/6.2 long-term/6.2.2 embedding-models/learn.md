# 6.2.2 嵌入模型选型

## 简单介绍

嵌入模型（Embedding Model）是将非结构化数据（文本、代码、图像）转换为固定长度高维向量的神经网络模型。在 Agent 长期记忆系统中，嵌入模型位于最前端，直接决定向量检索的质量上限——再优秀的向量数据库也无法弥补低质量嵌入带来的语义损失。选择合适的嵌入模型是构建高效记忆系统的基础性决策。

## 基本原理

### 嵌入的本质

嵌入模型将语义相似性映射为向量空间中的几何邻近性：

```
"我喜欢编程" → [0.12, -0.34, 0.78, 0.56, ..., 0.21]  (1536 维)
"I love coding" → [0.15, -0.31, 0.80, 0.52, ..., 0.25]  (1536 维)
// 语义相近→向量距离小 (余弦相似度 ≈ 0.95)

"Java是一种编程语言" → [0.28, -0.41, 0.65, 0.12, ..., -0.08]  (1536 维)
// 语义不同→向量距离大 (余弦相似度 ~ 0.3)
```

### 关键数学概念

| 概念 | 说明 | 对检索的影响 |
|------|------|-------------|
| 余弦相似度 | 测量向量方向一致性，范围 [-1, 1] | 最常用的相似度量，对向量长度不敏感 |
| 点积 | 未归一化的余弦相似度 | 当所有向量已归一化时等价于余弦 |
| L2 距离（欧氏距离） | 向量间的直线距离 | 对向量长度敏感，需确保适用性 |
| 维度 | 向量长度（如 384d、768d、1536d、3072d） | 越高表达能力越强，计算和存储成本越高 |

## 背景与演进

嵌入模型的发展经历了四代跨越：

**第一代：静态嵌入（2013-2018）** — Word2Vec、GloVe、FastText 等词级嵌入，每个词对应固定向量，无法处理一词多义。

**第二代：上下文嵌入（2018-2020）** — BERT、RoBERTa 引入上下文感知，同一词在不同语境下得到不同向量，但句子级嵌入仍需池化操作（CLS token / mean pooling）。

**第三代：专用嵌入模型（2021-2023）** — Sentence-BERT、BGE、Cohere Embed 等专为文本嵌入设计的模型，通过对比学习（Contrastive Learning）训练，在 MTEB（Massive Text Embedding Benchmark）等基准上表现大幅提升。

**第四代：多模态与超长上下文（2024-至今）** — OpenAI text-embedding-3、Voyage-2 等模型支持 8192+ token 上下文，部分模型（如 Nomic Embed）支持图像和文本的统一嵌入空间。

## 核心矛盾

### 1. 嵌入质量 vs 推理成本

- 更大模型（如 7B 参数）通常产生更好的嵌入，但推理延迟和成本显著增加。
- 实际部署中需要在 MTEB 分数和 P50 延迟之间权衡：小模型（~300M 参数）延迟 5-10ms，大模型（~7B 参数）延迟 50-200ms。

### 2. 通用性 vs 领域特异性

- 通用嵌入模型在多领域平均表现好，但在特定领域（法律、医疗、代码）不如微调过的专用模型。
- 选择通用模型简化运维但可能损失关键领域的检索精度。

### 3. 维度数 vs 存储-计算效率

- 以 OpenAI text-embedding-3 为例，提供两种维度可选：256d（更快更省）和 3072d（更高精度）。
- 降低维度通过 Matryoshka Representation Learning 实现，精度损失通常小于 5%，但存储和搜索成本降低 5-10 倍。

## 主流优化方向

1. **Matryoshka 嵌入（MRL）**：训练模型使得向量的前缀子集（前 256/512/1024 维）自身就是一个高质量嵌入，允许同一模型在不同维度间灵活切换，精度-成本可动态调整。
2. **量化感知训练（QAT）**：在训练阶段模拟量化操作，使模型在 INT8 精度下仍保持 FP32 的 95%+ 精度。
3. **对比学习改进**：通过难例挖掘（Hard Negative Mining）、批内负采样（In-batch Negatives）、知识蒸馏等技术提升嵌入判别力。
4. **长文档编码**：通过 Sliding Window Attention 或 Hierarchical Embedding 处理超过模型上下文限制的长文本。

## 产生的结果

- MTEB 平均分数从 2022 年的 58 分提升到 2024 年的 68+ 分（text-embedding-3-large 达到 64.6，Cohere Embed v3 达到 66.7）。
- 嵌入延迟从 100ms+（BERT-base）降低到 5-10ms（轻量模型 + GPU 部署）。
- 支持上下文从 512 token 扩展到 8192+ token。
- 多语言能力显著提升，BGE-M3 等模型支持 100+ 语言。

## 最大挑战

### 1. 嵌入漂移（Embedding Drift）

升级嵌入模型后，新产生的向量与历史向量位于不同语义空间，无法直接混合检索。解决方案：
- **双模型运行期**：新旧模型同时运行，逐步回填（backfill）历史向量。
- **对齐映射**：训练一个轻量映射函数将旧嵌入空间映射到新空间。
- **渐进式迁移**：按数据创建时间分批迁移，每批迁移后验证检索质量。

### 2. 长文本编码瓶颈

Agent 对话记忆经常包含超出模型上下文限制的文本。简单截断会丢失关键信息。分层嵌入（将文本分块后独立嵌入，检索时聚合）或多向量表示（ColBERT 式的后期交互）是主流解决方案。

### 3. 领域适应性

通用模型在垂直领域（代码、医疗、法律）的表现可能下降 10-20%。需要收集领域正负样本对进行微调，或选择已有领域优化的专用模型（如代码用 code-search-ada、法律用 Legal-BERT）。

## 能力边界

- **不可逆性**：嵌入是单向编码，无法从向量还原原始文本，信息已不可逆地损失。
- **上下文限制**：通常 512-8192 token，超出部分无法编码。即使支持长上下文，长文本的嵌入质量也会下降。
- **无法理解精确数字与否定**：嵌入擅长捕捉"主题"，但在精确数值比较和逻辑否定上表现不佳。例如"价格大于 100"和"价格小于 100"可能产生相似的嵌入。
- **时效性**：训练数据截止后的新概念（新术语、新梗）无法被准确嵌入。

## 区别对比：主流嵌入模型

### 闭源商业模型

| 维度 | text-embedding-3-small | text-embedding-3-large | Cohere Embed v3 | Voyage-2 |
|------|----------------------|----------------------|-----------------|----------|
| **公司** | OpenAI | OpenAI | Cohere | Voyage AI |
| **最大维度** | 1536 | 3072 | 1024 | 1024 |
| **默认维度** | 1536 | 3072 | 1024 | 1024 |
| **上下文窗口** | 8191 | 8191 | 512 | 4000 |
| **MTEB 分数** | 62.3 | 64.6 | 66.7 | 64.4 |
| **价格（$/1M tokens）** | $0.020 | $0.130 | $0.100 | $0.090 |
| **延迟（P50）** | ~20ms | ~50ms | ~40ms | ~35ms |
| **MRL 支持** | 是 (256-1536) | 是 (256-3072) | 否 | 否 |
| **多语言** | 优秀 | 优秀 | 良好 | 良好 |

### 开源模型

| 维度 | BGE-large-en-v1.5 | BGE-M3 | Nomic-embed-text-v1 | instructor-xl | E5-mistral-7b |
|------|-------------------|--------|---------------------|---------------|----------------|
| **开源协议** | MIT | MIT | Apache 2.0 | MIT | MIT |
| **参数量** | 335M | 567M | 137M | 1.5B | 7B |
| **最大维度** | 1024 | 1024 | 768 | 768 | 4096 |
| **上下文窗口** | 512 | 8192 | 8192 | 512 | 4096 |
| **MTEB 分数** | 64.2 | 64.5 | 62.4 | 61.5 | 66.6 |
| **多语言** | 仅英文 | 100+ 语言 | 仅英文 | 仅英文 | 仅英文 |
| **硬件需求** | CPU/低端 GPU | CPU/低端 GPU | CPU | GPU (4GB+) | GPU (16GB+) |
| **部署** | ONNX/Sentence-Transformers | ONNX | ONNX | Sentence-Transformers | vLLM |

## 选型维度

### 1. 质量（MTEB 分数）

- 追求极致精度：E5-mistral-7b (66.6) > Cohere Embed v3 (66.7) > text-embedding-3-large (64.6)
- 注：MTEB 分数是通用基准，实际效果需在自身数据上评估，差异可能达 10-20%。

### 2. 成本

| 规模 | 每日请求 | 推荐方案 | 月成本估算 |
|------|---------|---------|-----------|
| 个人/原型 | < 1000 | BGE-M3 (自部署) | ~$5-10 (GPU 实例) |
| 小团队 | < 10000 | text-embedding-3-small | ~$30-60 |
| 中型 | < 100000 | Cohere Embed v3 或 text-embedding-3-large | ~$100-500 |
| 大规模 | > 100000 | BGE-M3 自部署 | ~$500-2000 (定制硬件) |

### 3. 维度选择与向量存储成本

使用 MRL（Matryoshka Representation Learning）技术，同一模型可以在不同维度下使用：

```python
import openai

# 使用 text-embedding-3-small 生成不同维度的嵌入
# 存储成本随维度线性下降，搜索速度随维度超线性提升

embeddings_1536d = openai.embeddings.create(
    model="text-embedding-3-small",
    input="Agent memory system design",
    dimensions=1536  # 默认维度
)  # 存储: 1536 * 4 bytes = 6KB/向量

embeddings_256d = openai.embeddings.create(
    model="text-embedding-3-small",
    input="Agent memory system design",
    dimensions=256   # 仅为默认的 1/6
)  # 存储: 1KB/向量，搜索更快，精度略有下降
```

经验规则：维度减半，搜索速度提升约 2-3 倍，精度下降通常在 1-5%。

### 4. 延迟需求

| 场景 | 延迟要求 | 推荐方案 |
|------|---------|---------|
| 实时对话 Agent | < 50ms | BGE-small, text-embedding-3-small |
| 批量处理 | < 500ms | text-embedding-3-large, BGE-large |
| 离线处理 | 无限制 | E5-mistral-7b, 专用微调模型 |

### 5. 本地部署 vs API 调用

| 对比维度 | 本地部署 | API 调用 |
|---------|---------|---------|
| 每次推理成本 | 固定硬件成本 | 按 token 付费 |
| 延迟 | ~5-50ms（取决于硬件） | ~20-100ms（含网络） |
| 数据隐私 | 完全控制 | 数据离站 |
| 运维负担 | GPU 管理、模型更新 | 零运维 |
| 弹性扩展 | 需预置容量 | 自动弹性 |
| 模型灵活性 | 可微调、可更换 | 仅支持模型列表 |

## 核心优势

- **通用知识编码**：一个预训练模型即可覆盖广泛领域的语义理解，免去特征工程。
- **语义连续性**：嵌入空间是连续的，相似内容在空间中邻近，支持模糊匹配。
- **降维兼容**：通过 MRL 可在部署后灵活调整维度，精度-成本可按需权衡。
- **与 LLM 生态兼容**：嵌入模型与 LLM 的 tokenizer 和语义空间高度一致。

## 工程优化

### Embedding 缓存策略

嵌入计算是 Agent 记忆系统中的性能瓶颈之一。缓存策略可以显著降低延迟和成本：

```python
import hashlib
import json
from functools import lru_cache
import diskcache

class EmbeddingCache:
    """多级嵌入缓存"""

    def __init__(self, embedding_model, cache_dir="./embedding_cache"):
        self.model = embedding_model
        # 一级：内存 LRU 缓存（高频命中）
        self.mem_cache = lru_cache(maxsize=10000)(self._embed)
        # 二级：磁盘持久缓存（跨进程共享）
        self.disk_cache = diskcache.Cache(cache_dir)
        self.disk_cache.expire = 86400 * 30  # 30 天过期

    def get_embedding(self, text: str) -> list:
        text_hash = hashlib.sha256(text.encode()).hexdigest()
        # 优先内存缓存
        try:
            return self.mem_cache(text)
        except TypeError:  # lru_cache 不支持可哈希之外的参数
            pass

        # 其次磁盘缓存
        if text_hash in self.disk_cache:
            return self.disk_cache[text_hash]

        # 重新计算并缓存
        vector = self.model.embed(text)
        self.disk_cache[text_hash] = vector
        return vector

    def _embed(self, text: str) -> list:
        return self.model.embed(text)
```

### 批处理优化

```python
class BatchEmbeddingProcessor:
    """批处理嵌入，利用 GPU 并行计算"""

    def __init__(self, model, batch_size=32):
        self.model = model
        self.batch_size = batch_size

    async def embed_many(self, texts: list[str]) -> list[list[float]]:
        all_vectors = []
        for i in range(0, len(texts), self.batch_size):
            batch = texts[i:i + self.batch_size]
            vectors = await self.model.aembed(batch)
            all_vectors.extend(vectors)
        return all_vectors
```

### 维度降级策略（MRL）

```python
def get_adaptive_dimension(vector: list, priority: str = "speed"):
    """根据当前优先级截断向量"""
    dim_thresholds = {
        "max_precision": 3072,
        "balanced": 1024,
        "speed": 256,
        "max_speed": 64
    }
    target_dim = dim_thresholds[priority]
    return vector[:target_dim]
```

## 场景判断

### 适合使用专用嵌入的场景

- **问答系统 / RAG**：需要从海量文档中检索相关片段，语义理解直接决定检索质量。
- **对话记忆检索**：Agent 需要从历史对话中找到语义相关的内容片段。
- **多语言知识库**：用户提问和存储内容可能在不同语言。
- **代码检索**：开发者需要根据自然语言描述找到相关代码片段。

### 适合使用通用嵌入的场景

- **原型验证**：快速验证 Agent 记忆系统功能。
- **多领域覆盖**：无需针对特定领域微调，快速上线。
- **资源受限**：无法部署大模型或承担高昂的推理成本。

### 需要微调的信号

1. 在领域测试集上，通用嵌入的召回率低于 70%。
2. 领域有特殊术语、缩写或概念，通用模型无法区分。
3. 需要特定的相似度行为（如要求代码功能相似而非语法相似）。
