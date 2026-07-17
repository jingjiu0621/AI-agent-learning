# 9.2.1 dense-retrieval — 稠密检索：嵌入向量与语义匹配

## 简单介绍

稠密检索（Dense Retrieval）是指将文本通过神经网络编码为低维稠密向量（dense embeddings），在向量空间中基于语义相似度进行检索的技术。与传统的关键词匹配（BM25）不同，稠密检索理解的是"意思"而非"字词"——它能将"怎么修车"和"汽车故障排除方法"映射到相近的向量位置，即使它们没有任何共同词汇。

在 RAG 系统中，稠密检索是连接索引与生成的桥梁：它将用户查询转化为向量，在预先构建的文档向量库中搜索最相似的片段，为 LLM 提供相关上下文。

```
用户查询 "如何预防高血压？"
      │
      ▼
  ┌─────────────┐
  │ Embedding   │  ← 编码为稠密向量 [0.23, -0.45, 0.78, ..., 0.12]  （768/1024/1536 维）
  │ 模型        │
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐     ┌──────────────────────┐
  │ 向量相似度   │────→│ 文档向量索引          │
  │ 搜索 (ANN)   │     │ [doc1_vec, doc2_vec, │
  └──────┬──────┘     │  doc3_vec, ...]       │
         │            └──────────────────────┘
         ▼
  ┌─────────────┐
  │ Top-K 文档   │──→ 送入 LLM 生成回答
  └─────────────┘
```

## 基本原理

### 文本向量化（Embedding）

稠密检索的核心是将离散的文本转化为连续的稠密向量。每个维度捕捉语义的某个潜在特征，整个向量空间构成一个"语义地图"。

```
"猫"     → [0.32, -0.12, 0.87, ..., 0.03]   ← 语义相近的向量距离近
"猫咪"   → [0.31, -0.10, 0.85, ..., 0.04]   
"狗"     → [0.28, -0.08, 0.72, ..., 0.01]   ← 较近，属于宠物范畴
"汽车"   → [-0.45, 0.67, -0.21, ..., 0.55]  ← 较远，不同语义领域
"微积分" → [-0.78, 0.12, -0.54, ..., 0.89]  ← 很远，抽象学术概念
```

embedding 模型的训练目标就是让语义相关的文本在向量空间中距离近，语义无关的距离远。

### 向量相似度度量

三种最常用的相似度计算方法：

```
余弦相似度 (Cosine Similarity)
    cos(q, d) = (q · d) / (||q|| × ||d||)
    值域 [-1, 1]，1 表示方向完全一致
    最常用，不受向量长度影响

点积 (Dot Product)
    dot(q, d) = Σ(q_i × d_i)
    值域无界，受向量长度影响
    适合归一化后的向量（此时等价于余弦相似度）

欧氏距离 (L2 Distance)
    L2(q, d) = sqrt(Σ(q_i - d_i)^2)
    值域 [0, +∞)，0 表示完全相同
    需要转换为相似度（如取负或倒数）
```

**选择建议**：余弦相似度是大多数场景的默认选择。当使用训练好的双编码器（bi-encoder）且模型指定了相似度函数时，应优先遵循模型推荐。

```python
import numpy as np

def cosine_similarity(vec_a: np.ndarray, vec_b: np.ndarray) -> float:
    """计算两个向量的余弦相似度"""
    dot_product = np.dot(vec_a, vec_b)
    norm_a = np.linalg.norm(vec_a)
    norm_b = np.linalg.norm(vec_b)
    return dot_product / (norm_a * norm_b + 1e-10)

def dot_product(vec_a: np.ndarray, vec_b: np.ndarray) -> float:
    """计算点积"""
    return float(np.dot(vec_a, vec_b))

def euclidean_distance(vec_a: np.ndarray, vec_b: np.ndarray) -> float:
    """计算欧氏距离"""
    return float(np.linalg.norm(vec_a - vec_b))

def similarity_to_score(similarity: float, method: str = "cosine") -> float:
    """将相似度映射到 0-1 分数"""
    if method == "cosine":
        return (similarity + 1) / 2  # [-1, 1] → [0, 1]
    elif method == "dot":
        return 1 / (1 + np.exp(-similarity))  # sigmoid 归一化
    elif method == "l2":
        return 1 / (1 + similarity)  # 距离越小，分数越高
    return similarity
```

### Dense Passage Retrieval（DPR）架构

DPR 是稠密检索的经典架构，由 Karpukhin et al. (2020) 提出。其核心是**双编码器（Dual-Encoder / Bi-Encoder）**设计：

```
                     ┌──────────────────────────────┐
                     │         相似度计算              │
                     │     sim(q, p) = cos(E_Q(q),   │
                     │         E_P(p))               │
                     └──────┬───────────────┬────────┘
                            │               │
              ┌─────────────┴─────┐   ┌─────┴─────────────┐
              │    Query Encoder   │   │   Passage Encoder  │
              │    (E_Q, BERT)     │   │   (E_P, BERT)     │
              │                    │   │                   │
              │  [CLS] 查 询 [SEP] │   │ [CLS] 文 档 [SEP] │
              └─────────┬─────────┘   └─────────┬──────────┘
                        │                       │
                   "如何预防                   "高血压患者
                   高血压？"                    应当注意低盐
                                               饮食、规律运动..."
                   
    Query Encoder 输出: [0.12, 0.87, ..., 0.34] (768d)
    Passage Encoder 输出: [0.15, 0.82, ..., 0.30] (768d)
    
    余弦相似度: 0.92 → 高度匹配
```

**双编码器 vs 交叉编码器**：

```
双编码器 (Bi-Encoder)         交叉编码器 (Cross-Encoder)
┌─────┐    ┌─────┐           ┌──────────────────┐
│Query│    │Doc  │           │ [CLS] Query [SEP]│
│Encoder│  │Encoder│         │      Doc A [SEP] │
└──┬───┘  └──┬───┘           └────────┬─────────┘
   └────┬────┘                        │
        ▼                             ▼
    相似度计算                      [CLS] 表示 → 二分类分数
    可预先计算文档向量              不可预先计算
    适合大规模检索                  适合精排（rerank）

特征:
  - 文档向量可离线建索引         特征:
  - 检索速度快 (ms 级)            - 每对查询-文档都要过模型
  - 精度相对较低                   - 精度更高
  - 支持批量相似度计算              - 延迟高 (无法大规模)
```

## 背景与演进

```
时间线
│
├── 1960s-2010s: 稀疏检索时代
│   ├── TF-IDF (1960s): 词频-逆文档频率，开创了信息检索的统计方法
│   └── BM25 (1994): 在 TF-IDF 基础上引入文档长度归一化和饱和频率
│       → 优势：可解释、零样本、高效
│       → 局限：词法鸿沟（lexical gap）—— "车"和"汽车"算不同词
│
├── 2013-2018: 词向量时代
│   ├── Word2Vec / GloVe: 将词映射为稠密向量
│   └── 但词级向量无法直接用于段落级检索
│
├── 2019-2020: 预训练语言模型推动突破
│   ├── BERT (2019): 上下文感知的句子表示
│   ├── DPR (Karpukhin, 2020): 正式提出双编码器稠密检索框架
│   └── Sentence-BERT (Reimers, 2020): 让 BERT 生成可比的句子向量
│
├── 2021-2022: 稠密检索的繁荣
│   ├── ColBERT (Khattab, 2020): 延迟交互（late interaction）范式
│   │   └── query 和 doc 分别编码 token 级向量 → 交互时计算 MaxSim
│   ├── Contriever (2022): 无监督对比学习稠密检索
│   └── GTR / E5: 大规模文本到文本对比训练
│
├── 2023-2024: 开放模型井喷
│   ├── BGE (BAAI): 中英双语强性能，MTEB 榜首常驻
│   ├── text-embedding-3 (OpenAI, 2024): 1536d，支持动态维度
│   ├── Jina Embeddings: 8K 上下文窗口
│   └── NV-Embed (NVIDIA): 1024d，MTEB 多个任务第一
│
└── 2025-2026: 多模态与超长上下文
    ├── Qwen3-Embedding: 阿里通义，支持 32K 上下文
    ├── Gemini Embedding: Google，8192 维度选项
    ├── BGE-M3: 支持多语言、多粒度、多向量
    └── ColBERTv2 / PLAID: 延迟交互进入实用阶段
```

## 核心矛盾

### 语义理解 vs 精确匹配

稠密检索的最大优势也是其最大弱点——它理解"意思"而非"字词"。

```
场景：搜索 "iPhone 15 的电池容量是多少？"

稠密检索可能返回：
  ✓ "iPhone 15 Pro Max 配备 4422mAh 电池"（语义匹配成功）
  ✗ "手机续航能力对比分析"（语义接近但不够精确）
  ✗ "iPad 的电池健康度检测方法"（"电池"语义相关但实体不同）

BM25 精确匹配可能返回：
  ✗ "iPhone 15" 相关但包含 "电池容量" 的文档
  ✗ "苹果手机的续航表现"（没有"电池容量"这个词 → 0 分）

稠密检索擅长语义泛化，但在需要精确字面匹配时反而不如 BM25。
```

### 训练数据依赖性

稠密检索模型的性能高度依赖于训练数据的质量和覆盖范围：

| 维度 | 问题 | 影响 |
|------|------|------|
| 领域覆盖 | 模型在通用语料（维基百科/网络文本）上训练 | 专业领域（法律、医学、金融）效果下降 |
| 语言平衡 | 英文数据通常占主导 | 小语种检索质量显著低于英语 |
| 标注质量 | 需要(query, positive, negative)三元组 | 人工标注成本高，自动构造有噪声 |
| 数据偏移 | 训练数据分布 ≠ 实际应用分布 | 线上效果低于标注集指标 |

### 领域迁移（Domain Shift）问题

```
训练域 (维基百科)              应用域 (医疗病历)
"如何治疗感冒？"                "患者主诉上呼吸道感染症状"
    │                               │
    ▼                               ▼
[0.23, 0.56, -0.12, ...]       [0.56, -0.33, 0.78, ...]
    │                               │
    └─────────── 语义距离远 ─────────┘
    
同一语义在不同领域的向量表示差异巨大，导致检索效果断崖式下降。
```

## 主流优化方向

### 1. 双编码器改进

```
Bi-Encoder 改进路线：

基础 Bi-Encoder         改进: 交叉注意力          改进: 模型蒸馏
┌──────┐ ┌──────┐       ┌──────┐ ┌──────┐        ┌──────┐ ┌──────┐
│Query │ │Doc   │       │Query │ │Doc   │        │小模型 │ │小模型 │
│BERT  │ │BERT  │       │BERT  │ │BERT  │        │(6层) │ │(6层) │
└──┬───┘ └──┬───┘       └──┬───┘ └──┬───┘        └──┬───┘ └──┬───┘
   └───┬────┘               └───┬────┘               └───┬────┘
       ▼                         ▼                        ▼
   [CLS] 点积              [CLS] 拼接 + FFN           蒸馏自大模型
                             精度 ↑ 5-8%               精度下降 < 2%
                                                       速度 ↑ 3-5x
```

- **ColBERT 式延迟交互**：查询和文档分别编码为 token 级向量，检索时计算最大相似度（MaxSim），在精度和效率间取得更好平衡
- **多向量表示**：每个文档用多个向量表示（如 ColBERTv2的 token 级向量），检索精度更高但存储成本也更高

### 2. 训练策略优化

```python
# 难负样本挖掘（Hard Negative Mining）核心逻辑
def hard_negative_mining(query: str, positive_doc: str, candidate_pool: list, encoder, top_k: int = 50):
    """
    从候选池中挑选最难区分的负样本：
    与 query 语义相近（得分高）但与 positive_doc 不相关的文档
    """
    query_vec = encoder.encode(query)
    pos_vec = encoder.encode(positive_doc)
    
    scores = []
    for doc in candidate_pool:
        doc_vec = encoder.encode(doc)
        score = cosine_similarity(query_vec, doc_vec)
        scores.append((score, doc))
    
    # 按得分降序排列，取 Top-K 作为难负样本
    scores.sort(key=lambda x: x[0], reverse=True)
    hard_negatives = []
    
    for score, doc in scores[:top_k]:
        # 排除正样本本身
        if doc == positive_doc:
            continue
        # 确保与 query 语义相关（高得分）
        # 但又不是正确答案
        hard_negatives.append(doc)
    
    return hard_negatives

# 批内负样本（In-Batch Negatives）
# 原理：在一个 batch 内，每个 query 的正样本是 batch 中对应对的文档，
#       其他 batch 内的文档作为该 query 的负样本
# 优势：无需额外计算，充分利用 batch 内数据
def in_batch_negative_loss(query_embs, doc_embs, temperature=0.05):
    """
    query_embs: [batch_size, dim] — batch 内所有 query 的向量
    doc_embs:   [batch_size, dim] — batch 内所有文档的向量
    """
    batch_size = query_embs.shape[0]
    
    # 相似度矩阵 [batch_size, batch_size]
    sim_matrix = cosine_similarity_matrix(query_embs, doc_embs)
    
    # 对角线是正样本对，其余都是负样本
    labels = np.arange(batch_size)
    
    # 使用交叉熵损失：最大化对角线（正样本）相似度
    sim_matrix = sim_matrix / temperature
    loss = cross_entropy_loss(sim_matrix, labels)
    
    return loss
```

### 3. 量化感知训练（Quantization Aware Training）

```python
# 训练时模拟量化效果，使模型适应低精度表示
# 减少向量存储空间而不显著损失检索精度

def simulated_quantization(vectors: np.ndarray, bits: int = 8) -> np.ndarray:
    """模拟量化过程（训练时前向传播用）"""
    # 找到每维度的范围
    dim_min = vectors.min(axis=0)
    dim_max = vectors.max(axis=0)
    
    # 量化到 [0, 2^bits - 1]
    max_val = 2 ** bits - 1
    quantized = ((vectors - dim_min) / (dim_max - dim_min + 1e-10) * max_val)
    quantized = np.round(quantized)
    
    # 反量化回浮点（但精度已降低）
    dequantized = quantized / max_val * (dim_max - dim_min) + dim_min
    
    return dequantized
```

### 4. 混合检索融合

稠密检索 + 稀疏检索的互补融合是当前生产实践的主流方向：

| 融合策略 | 做法 | 效果 |
|---------|------|------|
| 线性加权 | score = α × dense_score + (1-α) × sparse_score | 简单有效，α 需调参 |
| RRF（倒数秩融合） | score = Σ 1/(k + rank_dense + rank_sparse) | 无需调参，鲁棒性好 |
| 级联检索 | 先用稀疏检索快速过滤 → 再用稠密精排 | 效率优先 |
| 学习融合 | 训练一个轻量模型学习融合权重 | 理论最优，需标注数据 |

```python
def reciprocal_rank_fusion(dense_ranking: list, sparse_ranking: list, k: int = 60) -> list:
    """
    倒数秩融合（Reciprocal Rank Fusion）
    dense_ranking, sparse_ranking: [doc_id, ...] 按得分降序排列
    """
    scores = {}
    for rank, doc_id in enumerate(dense_ranking):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)
    for rank, doc_id in enumerate(sparse_ranking):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)
    
    # 按融合得分排序
    fused = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return [doc_id for doc_id, _ in fused]
```

## 能力边界与结果边界

### 稠密检索擅长

| 场景 | 示例 | 表现 |
|------|------|------|
| 语义同义匹配 | "怎么减肥" ↔ "体重管理方法" | ⭐⭐⭐⭐⭐ |
| 多语言跨语言检索 | 中文 query 搜英文文档 | ⭐⭐⭐⭐ |
| 模糊/开放问题 | "人工智能的最新突破" | ⭐⭐⭐⭐ |
| 长文本检索 | 段落/篇章级匹配 | ⭐⭐⭐⭐ |
| 概念级检索 | "函数式编程思想" | ⭐⭐⭐⭐ |

### 稠密检索不擅长

| 场景 | 示例 | 失败原因 |
|------|------|---------|
| 精确数字/代码检索 | "版本号 v2.3.1"、"import os" | embedding 对精确 token 不敏感 |
| 稀有实体匹配 | 特定产品型号、专有名词 | 训练数据中未出现或出现极少 |
| 同义词反义混淆 | "高血糖" vs "低血糖" | 语义接近但含义相反 |
| 长尾概念 | 冷门领域术语 | 训练数据覆盖不足 |
| 更新后的知识 | 2026年新政策条款 | 模型训练有截止日期 |

```
性能随领域距离变化的典型曲线：

召回率
  │
1.0│        ┌──┐
  │       ┌┘  └┐
0.8│      ┌┘    └┐
  │     ┌┘       └┐
0.6│    ┌┘         └┐
  │   ┌┘            └┐
0.4│  ┌┘              └┐
  │ ┌┘                  └┐
0.2│─┘                    └────
  │
  └────────────────────────────► 领域漂移程度
   通用领域    相近领域    完全不同领域
```

## 与其他检索方式的区别

| 维度 | 稠密检索（Dense） | 稀疏检索（Sparse/BM25） | 混合检索（Hybrid） |
|------|-------------------|------------------------|-------------------|
| 表示方式 | 低维稠密向量 (768d~1536d) | 高维稀疏向量 (词汇袋, 数万-数百万维) | 两者结合 |
| 匹配原理 | 语义相似度匹配 | 关键词精确匹配（TF-IDF 加权） | 语义+关键词 |
| 计算效率 | 依赖 ANN 索引（近似搜索） | 倒排索引，亚毫秒级 | 两者都要算，融合排序 |
| 存储大小 | 每个文档 768~1536 个 float | 每个文档的稀疏词频表示 | 两者都存 |
| 冷启动能力 | ❌ 需要训练好的 embedding 模型 | ✅ 开箱即用，无需训练 | 依赖稠密部分 |
| 可解释性 | ⭐⭐ 黑盒，"为什么匹配？"说不清 | ⭐⭐⭐⭐⭐ "因为都含有词 X" | ⭐⭐⭐ |
| 领域适配 | ⭐⭐ 通用模型在特定领域可能下降 | ⭐⭐⭐⭐⭐ 词频统计天然适应领域 | ⭐⭐⭐⭐ |
| OOV（未登录词）| ✅ 可处理未见过的词 | ❌ 完全无法匹配未在文档中出现的词 | 互补 |
| 精确匹配能力 | ⭐⭐ 同义词泛化有时反而有害 | ⭐⭐⭐⭐⭐ 精确匹配无可替代 | ⭐⭐⭐⭐ |

### 稠密检索 vs ColBERT 式延迟交互

ColBERT 属于稠密检索的变体，但与传统双编码器有本质区别：

```
传统双编码器:                       ColBERT 延迟交互:
query → [0.1, 0.5, ...] (1个向量)   query → [[0.1, 0.2], [0.3, 0.4], ...] (每个 token 一个向量)
doc   → [0.3, 0.7, ...] (1个向量)   doc   → [[0.5, 0.6], [0.7, 0.8], ...] (每个 token 一个向量)
             │                                      │
             ▼                                      ▼
      cos(1个query向量,              MaxSim: 每个 query token 找最相似
          1个doc向量)                      的 doc token → 求和
      = 1 次相似度计算               = query_len × doc_len 次相似度计算

优势: 快，适合大规模                优势: 精度更高，保留细粒度匹配
缺点: 精度上限低                    缺点: 计算量大，需要特殊索引（PLAID）
```

## 核心优势

### 1. 语义匹配能力

稠密检索突破了词法鸿沟，让"意思相近"的文本在向量空间中聚集：

```
"汽车" ──→ ──→ ──→ ──→ ──→ "车辆"
"车"   ──→ ──→ ──→ ──→ ──→ "轿车"
"automobile" ──→ ──→ ──→ "car"

→ 所有这些词在稠密空间中距离都很近
→ BM25 只能匹配完全相同的词形
```

### 2. 多语言与跨语言能力

现代 embedding 模型（如 BGE-M3、text-embedding-3）天然支持多语言，甚至跨语言检索：

| 模型 | 支持语言 | 跨语言检索 |
|------|---------|-----------|
| OpenAI text-embedding-3 | 100+ 语言 | ✅ 强 |
| BGE-M3 | 100+ 语言 | ✅ 强 |
| BGE-large-zh | 中英文为主 | ✅ 中英 |
| Cohere Embed | 100+ 语言 | ✅ 强 |
| Qwen3-Embedding | 多语言 | ✅ 强 |

```python
# 跨语言检索示例：中文 query → 英文文档
query_cn = "深度学习在医疗影像中的应用"
doc_en = "Deep learning has shown remarkable performance in medical image analysis..."
# 现代 embedding 模型可以直接将两者映射到相近的向量空间
```

### 3. 跨模态扩展

稠密检索的范式可以扩展到多模态：

```
文本编码器 → 文本向量
图像编码器 → 图像向量
音频编码器 → 音频向量

    同一向量空间 → 跨模态检索
    ┌─────────────────────────┐
    │   "夕阳海滩"  ─────────┐│
    │                        ││
    │   🖼️ (夕阳照片) ──────┘│
    │                        │
    │   🎵 (海浪声) ──────── │
    └─────────────────────────┘
    → 一个 embedding 空间同时编码文本、图像、音频
    → 用文字搜索图片、用图片搜索音频... 多种跨模态检索
```

## 工程优化

### 1. Embedding 缓存

```python
import hashlib
import json
from pathlib import Path
import numpy as np

class EmbeddingCache:
    """嵌入向量缓存，避免重复计算"""
    
    def __init__(self, cache_dir: str = "./embedding_cache"):
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(parents=True, exist_ok=True)
        self.memory_cache = {}  # LRU 可以用更复杂的策略
    
    def _hash_text(self, text: str) -> str:
        return hashlib.sha256(text.encode('utf-8')).hexdigest()
    
    def get(self, text: str) -> np.ndarray:
        text_hash = self._hash_text(text)
        # 优先从内存缓存读取
        if text_hash in self.memory_cache:
            return self.memory_cache[text_hash]
        # 再从磁盘读取
        cache_path = self.cache_dir / f"{text_hash}.npy"
        if cache_path.exists():
            vec = np.load(cache_path)
            self.memory_cache[text_hash] = vec
            return vec
        return None
    
    def set(self, text: str, vector: np.ndarray):
        text_hash = self._hash_text(text)
        self.memory_cache[text_hash] = vector
        cache_path = self.cache_dir / f"{text_hash}.npy"
        np.save(cache_path, vector)
```

### 2. 批量推理（Batch Inference）

```python
def batch_encode(texts: list, model, batch_size: int = 32, show_progress: bool = True) -> np.ndarray:
    """
    批量编码，显著提升吞吐量
    GPU 上 batch_size 越大，吞吐越高（显存允许范围内）
    """
    all_embeddings = []
    
    for i in range(0, len(texts), batch_size):
        batch_texts = texts[i:i + batch_size]
        # 批处理：模型同时编码 batch_size 个文本
        batch_embeddings = model.encode(batch_texts)
        all_embeddings.append(batch_embeddings)
    
    return np.vstack(all_embeddings)

# 性能对比：
# 单个编码 1000 条文本: ~30 秒
# Batch 编码 1000 条文本 (batch=32): ~3 秒 (10x 加速)
```

### 3. 维度规约（Dimensionality Reduction）

OpenAI text-embedding-3 支持通过 `dimensions` 参数动态截断维度，无需额外操作：

```python
# OpenAI 原生支持维度截断
response = openai.Embedding.create(
    model="text-embedding-3-large",
    input="需要编码的文本",
    dimensions=256  # 从 1536 维降到 256 维，精度损失 < 5%
)

# 开源的降维方法：
from sklearn.decomposition import PCA

def reduce_dimensions(vectors: np.ndarray, target_dim: int = 256) -> np.ndarray:
    """使用 PCA 降低向量维度，减少存储和检索开销"""
    pca = PCA(n_components=target_dim)
    reduced = pca.fit_transform(vectors)
    print(f"降维: {vectors.shape[1]}d → {target_dim}d")
    print(f"保留方差比: {pca.explained_variance_ratio_.sum():.2%}")
    return reduced
```

### 4. 近似最近邻搜索（ANN）

FAISS（Facebook AI Similarity Search）是工业界最流行的 ANN 库：

```python
import faiss
import numpy as np

class DenseRetriever:
    """基于 FAISS 的稠密检索引擎"""
    
    def __init__(self, dim: int, index_type: str = "flat"):
        """
        index_type:
          - "flat": 暴力搜索，精确但慢（适合 < 10K 文档）
          - "ivf": 倒排文件索引，近似搜索（适合 10K ~ 1M 文档）
          - "hnsw": 层级小世界图，高速近似搜索（适合 > 1M 文档）
          - "pq": 乘积量化，极致压缩内存（适合 > 10M 文档）
        """
        self.dim = dim
        self.documents = []
        self.index = self._build_index(index_type)
    
    def _build_index(self, index_type: str):
        if index_type == "flat":
            return faiss.IndexFlatIP(self.dim)  # 内积索引（等价于余弦，如果向量已归一化）
        
        elif index_type == "ivf":
            # IVF (Inverted File): 训练聚类 + 倒排
            nlist = 100  # 聚类中心数量
            quantizer = faiss.IndexFlatIP(self.dim)
            index = faiss.IndexIVFFlat(quantizer, self.dim, nlist, faiss.METRIC_INNER_PRODUCT)
            index.nprobe = 10  # 搜索时检查的聚类数（越大越精确但越慢）
            return index
        
        elif index_type == "hnsw":
            # HNSW (Hierarchical Navigable Small World)
            index = faiss.IndexHNSWFlat(self.dim, 32)  # 32 = 每个节点的连接数
            index.hnsw.efConstruction = 200  # 建索引时的搜索范围
            index.hnsw.efSearch = 64  # 搜索时的探索范围
            return index
        
        elif index_type == "pq":
            # PQ (Product Quantization): 压缩向量到 1/4 ~ 1/16 大小
            m = 8  # 子空间数
            nbits = 8  # 每个子空间的编码位数
            index = faiss.IndexPQ(self.dim, m, nbits, faiss.METRIC_INNER_PRODUCT)
            return index
        
        else:
            raise ValueError(f"Unknown index type: {index_type}")
    
    def add_documents(self, documents: list, embeddings: np.ndarray):
        """添加文档及其向量"""
        if not self.index.is_trained and isinstance(self.index, faiss.IndexIVF):
            # IVF 需要先训练
            self.index.train(embeddings)
        
        self.index.add(embeddings)
        self.documents.extend(documents)
    
    def search(self, query_vector: np.ndarray, top_k: int = 10) -> list:
        """
        搜索 Top-K 最相似文档
        返回: [(doc_id, score, document_text), ...]
        """
        # FAISS 要求输入为 2D 数组
        query_vector = query_vector.reshape(1, -1)
        scores, indices = self.index.search(query_vector, top_k)
        
        results = []
        for score, idx in zip(scores[0], indices[0]):
            if idx >= 0 and idx < len(self.documents):
                results.append({
                    "doc_id": int(idx),
                    "score": float(score),
                    "text": self.documents[idx]
                })
        
        return results

# ============ 使用示例 ============

# 1. 准备数据
documents = [
    "高血压患者应当注意低盐饮食",
    "规律运动有助于控制血压",
    "糖尿病需要定期监测血糖水平",
    "高血脂患者应减少动物脂肪摄入",
    "每天喝八杯水对健康有益",
]

# 2. 生成向量（假设已有 embedding 模型）
# embeddings = model.encode(documents)  # [5, dim]
# 示例用随机向量
dim = 384
embeddings = np.random.random((len(documents), dim)).astype(np.float32)
embeddings = embeddings / np.linalg.norm(embeddings, axis=1, keepdims=True)

# 3. 构建索引
retriever = DenseRetriever(dim=dim, index_type="flat")
retriever.add_documents(documents, embeddings)

# 4. 搜索
query_vec = np.random.random(dim).astype(np.float32)
query_vec = query_vec / np.linalg.norm(query_vec)

results = retriever.search(query_vec, top_k=3)
for r in results:
    print(f"Score: {r['score']:.4f} | {r['text']}")

# ============ FAISS 索引性能对比 ============
# 假设: 100 万条 768 维向量
#
# 索引类型   | 建索引时间 | 搜索耗时(1条) | 内存 | Recall@10
# Flat       | 0 秒      | 100 ms       | 3 GB | 100%
# IVF(100)   | 30 秒     | 2 ms         | 3 GB | 95%
# HNSW       | 180 秒    | 1 ms         | 4 GB | 99%
# PQ(8x8)    | 60 秒     | 5 ms         | 0.4 GB | 85%
# IVF+PQ     | 40 秒     | 3 ms         | 0.5 GB | 90%
```

### 5. 使用向量数据库

```python
# ============ ChromaDB（轻量级，适合原型开发） ============
import chromadb

chroma_client = chromadb.Client()
collection = chroma_client.create_collection(
    name="my_docs",
    embedding_function=None,  # 自行提供向量
    metadata={"hnsw:space": "cosine"}
)

# 添加文档
collection.add(
    documents=["高血压患者应当注意低盐饮食", "规律运动有助于控制血压"],
    embeddings=[[0.1, 0.2, ...], [0.3, 0.4, ...]],  # 自行生成的向量
    ids=["doc_001", "doc_002"],
    metadatas=[{"source": "medical_guide"}, {"source": "health_blog"}]
)

# 搜索
results = collection.query(
    query_embeddings=[[0.15, 0.25, ...]],
    n_results=5,
    where={"source": "medical_guide"}  # 支持元数据过滤
)


# ============ Qdrant（生产级，支持过滤和聚合） ============
from qdrant_client import QdrantClient
from qdrant_client.http.models import Distance, VectorParams, PointStruct

qdrant = QdrantClient("localhost", port=6333)

# 创建集合
qdrant.create_collection(
    collection_name="my_collection",
    vectors_config=VectorParams(size=384, distance=Distance.COSINE),
)

# 插入数据
qdrant.upsert(
    collection_name="my_collection",
    points=[
        PointStruct(id=1, vector=[0.1, 0.2, ...], payload={"text": "高血压..."}),
        PointStruct(id=2, vector=[0.3, 0.4, ...], payload={"text": "糖尿病..."}),
    ]
)

# 检索（支持预过滤）
qdrant.search(
    collection_name="my_collection",
    query_vector=[0.15, 0.25, ...],
    query_filter=None,  # 可设置 Filter 进行条件过滤
    limit=10,
)
```

## 场景适配指南

| 场景 | 推荐模型 | 向量维度 | 索引类型 | 向量数据库 | 检索策略 |
|------|---------|---------|---------|-----------|---------|
| 通用知识问答 (英文) | OpenAI text-embedding-3-small | 512 | HNSW | Qdrant / Pinecone | 稠密检索 |
| 通用知识问答 (中文) | BGE-large-zh / Qwen3-Embedding | 1024 | HNSW | Milvus | 稠密检索 |
| 多语言 RAG | BGE-M3 / Cohere Embed | 1024 | IVF+PQ | Milvus | 稠密+稀疏混合 |
| 代码检索 | CodeBERT / starencoder | 768 | IVF | FAISS | 稠密为主 + BM25 保底 |
| 法律文档检索 | BGE-large + 领域微调 | 1024 | Flat (精确) | FAISS | 稠密检索 + 关键词过滤 |
| 医疗文献检索 | PubMedBERT-based | 768 | HNSW | Weaviate | 稠密检索 + 元数据过滤 |
| 电商搜索 | OpenAI text-embedding-3-large | 256 | IVF | Qdrant | 稠密为主 + 属性过滤 |
| 学术论文检索 | SPECTER / BGE-large | 768 | HNSW | FAISS + PGVector | 稠密检索 + 引文信息 |
| 新闻实时检索 | 轻量模型 (MiniLM) | 384 | Flat | ChromaDB | 稠密检索 + 时间过滤 |
| 十亿级文档检索 | NV-Embed + 多级索引 | 1024 | IVF+PQ+HNSW 多级 | Milvus / Qdrant | 先粗筛再精排 |
| 低延迟 (<10ms) | 量化模型 (int8) | 256 | HNSW (efSearch 调小) | FAISS | 提前量化 + 缓存 |
| 高精度 (>95% Recall@10) | 不量化 | 全维度 | Flat / IVF (nprobe 大) | FAISS | 暴力搜索 |

### 模型选择速查

```python
# 当前主流 embedding 模型配置参考（2025-2026）
#
# 模型名                   | 维度 | 最大输入 | 语言     | 相对性能 | 建议使用场景
# -------------------------|------|---------|----------|---------|-------------
# text-embedding-3-large   | 1536 | 8191    | 多语言    | ⭐⭐⭐⭐⭐ | 预算充足的生产环境
# text-embedding-3-small   | 512  | 8191    | 多语言    | ⭐⭐⭐⭐  | 性价比最优
# BGE-large-zh (v1.5)     | 1024 | 512     | 中文为主  | ⭐⭐⭐⭐⭐ | 中文 RAG 首选
# BGE-M3                  | 1024 | 8192    | 100+语言  | ⭐⭐⭐⭐⭐ | 多语言+长文本
# Qwen3-Embedding         | 1024 | 32768   | 多语言    | ⭐⭐⭐⭐⭐ | 超长上下文检索
# NV-Embed-2B             | 1024 | 4096    | 英文为主  | ⭐⭐⭐⭐⭐ | MTEB 榜首级性能
# Cohere Embed v3         | 1024 | 512     | 100+语言  | ⭐⭐⭐⭐  | 企业级多语言
# Jina Embeddings v3      | 1024 | 8192    | 多语言    | ⭐⭐⭐⭐  | 长文本场景
# E5-mistral-7b            | 4096 | 4096    | 英文      | ⭐⭐⭐⭐  | 高质量但计算成本高
# Instructor-XL            | 768  | 512     | 英文      | ⭐⭐⭐⭐  | 需要定制指令(task_id)

EMBEDDING_CONFIGS = {
    "openai_large": {
        "model": "text-embedding-3-large",
        "dim": 1536,
        "max_tokens": 8191,
        "provider": "OpenAI",
        "cost_per_1k_tokens": 0.00013,  # USD
        "recommended_top_k_ratio": 0.01,  # 建议 Top-K = total_docs * 0.01
    },
    "openai_small": {
        "model": "text-embedding-3-small",
        "dim": 512,
        "max_tokens": 8191,
        "provider": "OpenAI",
        "cost_per_1k_tokens": 0.00002,
        "recommended_top_k_ratio": 0.02,
    },
    "bge_large_zh": {
        "model": "BAAI/bge-large-zh-v1.5",
        "dim": 1024,
        "max_tokens": 512,
        "provider": "HuggingFace (BAAI)",
        "cost_per_1k_tokens": 0,  # 自部署
        "recommended_top_k_ratio": 0.01,
        "is_open_source": True,
        "notice": "query 需要加前缀 '为这个句子生成表示以用于检索相关文章：'"
    },
    "bge_m3": {
        "model": "BAAI/bge-m3",
        "dim": 1024,
        "max_tokens": 8192,
        "provider": "HuggingFace (BAAI)",
        "cost_per_1k_tokens": 0,
        "recommended_top_k_ratio": 0.01,
        "is_open_source": True,
        "supports_dense_sparse_multi": True,
    },
}
```

### 端到端稠密检索管线

```python
# ============ 完整稠密检索管线 ============
import numpy as np
from typing import List, Dict, Optional
import json

class DenseRAGPipeline:
    """端到端稠密检索管线：编码 → 索引 → 检索"""
    
    def __init__(self, model, embedding_dim: int):
        self.model = model
        self.dim = embedding_dim
        self.cache = EmbeddingCache()
        self.index = None  # FAISS 索引
        self.documents: List[Dict] = []
    
    def encode(self, text: str) -> np.ndarray:
        """带缓存的文本编码"""
        cached = self.cache.get(text)
        if cached is not None:
            return cached
        
        vec = self.model.encode(text)
        vec = vec / np.linalg.norm(vec)  # 归一化（用于余弦相似度）
        self.cache.set(text, vec)
        return vec
    
    def batch_encode_documents(self, documents: List[Dict], batch_size: int = 64):
        """
        批量编码并构建索引
        documents: [{"id": str, "text": str, "metadata": dict}, ...]
        """
        self.documents = documents
        texts = [d["text"] for d in documents]
        
        # 批量编码，显示进度
        all_vectors = []
        for i in range(0, len(texts), batch_size):
            batch = texts[i:i+batch_size]
            batch_vecs = self.model.encode(batch)
            # 归一化
            batch_vecs = batch_vecs / np.linalg.norm(batch_vecs, axis=1, keepdims=True)
            all_vectors.append(batch_vecs)
        
        all_vectors = np.vstack(all_vectors).astype(np.float32)
        
        # 构建 FAISS HNSW 索引
        self.index = faiss.IndexHNSWFlat(self.dim, 32)
        self.index.hnsw.efConstruction = 200
        self.index.add(all_vectors)
        
        return len(all_vectors)
    
    def retrieve(self, query: str, top_k: int = 10, threshold: Optional[float] = None) -> List[Dict]:
        """
        检索：query → 向量 → ANN 搜索 → 过滤 → 返回结果
        """
        # 编码查询
        q_vec = self.encode(query).astype(np.float32).reshape(1, -1)
        
        # ANN 搜索
        scores, indices = self.index.search(q_vec, top_k * 2)  # 多要一些，后续过滤
        
        results = []
        for score, idx in zip(scores[0], indices[0]):
            if idx < 0 or idx >= len(self.documents):
                continue
            
            doc = self.documents[idx]
            
            # 可选阈值过滤
            if threshold is not None and score < threshold:
                continue
            
            results.append({
                "doc_id": doc["id"],
                "text": doc["text"],
                "metadata": doc.get("metadata", {}),
                "score": float(score),
                "rank": len(results) + 1,
            })
            
            if len(results) >= top_k:
                break
        
        return results
```

## 性能监控与评估

### 关键指标

```python
# 稠密检索系统应持续监控的指标
MONITORING_METRICS = {
    "检索质量": {
        "Recall@K": "前 K 个结果中相关文档的比例",
        "MRR": "第一个相关结果的倒数排名",
        "NDCG@K": "考虑排序位置的归一化折损累计增益",
        "Precision@K": "前 K 个结果中相关文档的比例",
    },
    "性能": {
        "P95/P99 延迟": "检索响应时间",
        "QPS": "每秒查询数",
        "索引刷新延迟": "新文档可被检索到的时间差",
    },
    "资源": {
        "向量存储量": "当前存储的向量总数",
        "内存使用": "FAISS 索引常驻内存",
        "Embedding API 调用量": "调用次数和费用",
    },
}

def evaluate_retrieval(retriever, eval_queries: List[Dict], k: int = 10):
    """
    评估检索质量
    eval_queries: [{"query": str, "relevant_docs": [doc_id, ...]}, ...]
    """
    recall_at_k = []
    mrr = []
    
    for item in eval_queries:
        query = item["query"]
        relevant = set(item["relevant_docs"])
        
        results = retriever.retrieve(query, top_k=k)
        retrieved = [r["doc_id"] for r in results]
        
        # Recall@K
        hits = sum(1 for d in retrieved if d in relevant)
        recall_at_k.append(hits / len(relevant) if relevant else 0)
        
        # MRR
        for rank, doc_id in enumerate(retrieved, 1):
            if doc_id in relevant:
                mrr.append(1.0 / rank)
                break
        else:
            mrr.append(0)
    
    return {
        "Recall@{}".format(k): np.mean(recall_at_k),
        "MRR": np.mean(mrr),
        "Total_queries": len(eval_queries),
    }
```

---

稠密检索是 RAG 系统的核心引擎。选择 embedding 模型如同为系统选择"感知器官"——决定了系统能理解什么、不能理解什么。在实践中，**没有最好的模型，只有最适合场景的模型**。对于生产系统，推荐从混合检索入手（稠密 + 稀疏互补），并通过持续的评估迭代来优化检索质量。
