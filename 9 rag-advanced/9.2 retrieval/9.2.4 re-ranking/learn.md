# 9.2.4 Re-Ranking — 重排序：用 Cross-Encoder 精排检索结果

## 简单介绍

重排序（Re-Ranking）是 RAG 检索阶段的第二道关卡——在第一阶段召回（Retrieval）拿到候选文档后，用更精确但更慢的模型对候选列表重新打分排序。核心思路是：**先用高效的 Bi-Encoder 从海量文档中快速召回 Top-K（如 K=100），再用精确的 Cross-Encoder 对这 K 个候选精排，选出最终送入 LLM 的 Top-N（如 N=5~10）**。

```
┌─────────────┐     Top-100      ┌─────────────┐     Top-5      ┌──────────────┐
│  Bi-Encoder  │ ──────────────> │ Cross-Encoder │ ───────────> │     LLM      │
│  (快速召回)   │   候选文档池     │  (精确重排)   │  精排结果     │  (生成答案)   │
└─────────────┘                  └─────────────┘                └──────────────┘
```

## 基本原理

### Bi-Encoder vs Cross-Encoder 架构差异

这是重排序最核心的技术背景。两种架构的关键区别在于 **Query 和 Document 何时交互**：

```
Bi-Encoder（第一阶段召回）
═══════════════════════════════
Query ──> Encoder ──> q_vec
Doc  ──> Encoder ──> d_vec
相似度 = cosine(q_vec, d_vec)     ← 交互发生在编码之后

Cross-Encoder（第二阶段重排）
═══════════════════════════════
[CLS] Query [SEP] Doc [SEP] ──> Encoder ──> 相关性分数
                                  ↑  交互发生在编码过程中（注意力机制跨 Query 和 Doc）
```

| 维度 | Bi-Encoder | Cross-Encoder |
|------|-----------|--------------|
| 编码方式 | Query 和 Doc 独立编码 | Query 和 Doc 拼接后一起编码 |
| 交互时机 | 编码之后（向量点积/余弦） | 编码之中（注意力跨文本交互） |
| 精度 | ⭐⭐ 中等 | ⭐⭐⭐⭐⭐ 很高 |
| 速度 | ⭐⭐⭐⭐⭐ 极快（可预计算文档向量） | ⭐⭐ 较慢（每对需重新计算） |
| 文档向量缓存 | 支持（索引时可预计算） | 不支持（必须实时拼接计算） |
| 适用阶段 | 第一阶段：海量候选召回 | 第二阶段：少量候选精排 |

Cross-Encoder 准确的原因在于：注意力机制让 Query 的每个 token 都能"看到" Document 的每个 token，从而捕捉到细粒度的语义匹配信号——比如 Query 中的"不是"和 Document 中的"否认"之间的精确语义关系。Bi-Encoder 则把 Query 和 Doc 压缩为独立向量，丢失了这种 token 级交互信息。

### 两阶段检索范式

```
第一阶段：Bi-Encoder 粗召
  - 向量检索（ANN）：Faiss / Annoy / HNSW
  - 关键词检索（BM25）：Elasticsearch / Lucene
  - 混合检索（Hybrid）：向量 + 关键词融合
  
第二阶段：Cross-Encoder 精排
  - 对 Top-K（50~200）候选计算精确相关性分数
  - 按分数降序取 Top-N（3~10）送入 LLM
```

### 重排序分数计算示例

```python
# Cross-Encoder 核心逻辑：Query + Doc 拼接输入，输出相关性分数
import torch
from transformers import AutoTokenizer, AutoModelForSequenceClassification

model_name = "cross-encoder/ms-marco-MiniLM-L-6-v2"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name)

query = "什么是注意力机制？"
doc = "注意力机制（Attention）允许模型在生成每个输出时聚焦于输入序列的不同部分..."

# Cross-Encoder: query 和 doc 拼接
inputs = tokenizer(
    query, doc,
    return_tensors="pt",
    truncation=True,
    max_length=512,
    padding=True
)

with torch.no_grad():
    outputs = model(**inputs)
    score = outputs.logits.squeeze().item()

print(f"相关性分数: {score:.4f}")  # 分数越高越相关
```

## 背景与演进

- **单阶段检索时代（2020-2022）**：早期 RAG 直接用 Bi-Encoder embedding 做向量检索，召回的 Top-K 就直接送入 LLM。精度受限于 embedding 模型的表达能力。

- **两阶段检索兴起（2022-2023）**：业界发现 Bi-Encoder 召回的 Top-1 往往不准，但 Top-20~50 中通常包含正确答案。引入 Cross-Encoder 做二次精排成为标配。代表工作包括 `cross-encoder/ms-marco-MiniLM-L-6-v2` 等轻量 Cross-Encoder 模型。

- **专用重排序模型爆发（2023-2024）**：
  - **Cohere Rerank**：最早商业化的重排序 API，v2/v3/v3.5/v4 持续迭代，支持多语言
  - **BGE-Reranker-v2**（BAAI 智源）：开源领域标杆，含 M3（多语言）、Gemma（大模型底座）、Flash（轻量快速）等变体
  - **Jina Reranker v3**（2025）：0.6B 参数的 listwise 重排序器，多语言 SOTA

- **LLM 作为重排序器（2024-2025）**：直接用生成式 LLM（GPT-4、Qwen 等）对候选列表重排，利用 LLM 的深层语义理解能力。代表工作：**RankGPT**（ChatGPT 逐段重排）、**RankZephyr**（7B 专用微调）、**RankLLM** 工具包。

- **当前趋势**：重排序已从"可有可无的优化"变为"生产 RAG 系统的标配组件"。NVIDIA、Amazon、Cohere、Jina 等均提供托管的 reranking API。开源生态中，BGE-Reranker、Jina Reranker、mxbai-rerank 构成三强格局。

## 核心矛盾

**精度 vs 延迟——重排序的核心权衡**：

```
                    Cross-Encoder 模型大小
小型 (100M) ──────────────────────────────> 大型 (7B+)
  低延迟、低精度                             高延迟、高精度
  ╔══════════════════════════════════════════════════════╗
  ║  候选池大小同样关键：K 越大 → 找到好文档的概率越高  ║
  ║  K 越大 → 重排序延迟越高（线性增长）                ║
  ╚══════════════════════════════════════════════════════╝
```

| 候选池大小 (K) | 召回含正确答案概率 | 重排序延迟 (1K候选) | 推荐场景 |
|---------------|-----------------|-------------------|---------|
| 20 | ~70% | 低 | 实时聊天、客服 |
| 50 | ~85% | 中 | 通用问答系统 |
| 100 | ~93% | 较高 | 知识库检索、文档 QA |
| 200 | ~97% | 高 | 高精度场景（法律、医疗） |
| 500+ | ~99% | 很高 | 离线批量处理、论文检索 |

## 主流优化方向

### 1. Pointwise vs Listwise 重排序

重排序的打分策略分两大类：

```
Pointwise（逐点打分）
  ┌──────────┐    分数
  │ Q + D₁  │ ──>  0.92
  │ Q + D₂  │ ──>  0.45    独立计算每对的分数，互不影响
  │ Q + D₃  │ ──>  0.78    简单直接，可并行
  └──────────┘

Listwise（列表式排序）
  ┌──────────────────┐
  │ Q + [D₁, D₂, D₃] │ ──> [2, 3, 1]  模型同时看到所有候选
  └──────────────────┘                理解文档间相对关系
                                        更准确但更慢
```

- **Pointwise**：Cross-Encoder 的标准模式，每对 (Query, Doc) 独立打分，然后排序。效率高，适合大规模候选。
- **Listwise**：同时输入 Query 和所有候选文档的标题/摘要，让模型直接输出排序结果。Jina Reranker v3、RankGPT 采用此方式，质量更高但受限于输入长度。

### 2. LLM-as-Reranker

用 LLM 替代 Cross-Encoder 做重排序，有三种模式：

```
模式 A：Pointwise 打分
  Prompt: "请评估以下文档与问题的相关性，输出 0-10 分。"
  输入: "{query}\n{document}"
  输出: "8"（分数）

模式 B：Listwise 排序
  Prompt: "请将以下文档按与问题的相关性从高到低排序。"
  输入: "{query}\n[1] ... \n[2] ... \n[3] ..."
  输出: "[3] > [1] > [2]"

模式 C：逐对比较（Pairwise）
  Prompt: "文档 A 和文档 B，哪个与问题更相关？"
  输入: "{query}\nA: ...\nB: ..."
  输出: "文档 A 更相关"
```

代表工作：
- **RankGPT**（2023）：用 ChatGPT 做 listwise 重排，在 BEIR 基准上超越当时所有专用 reranker，但延迟高、成本高
- **RankZephyr**（2024）：用 Zephyr-7B 微调为专用 reranker，接近 RankGPT 质量但模型小很多
- **RankLLM**（2025）：统一工具包，支持多种 LLM 后端的重排序

### 3. 高效 Cross-Encoder 变体

- **知识蒸馏**：大 Cross-Encoder（如 MiniLM-L-12）作为 Teacher，小模型（如 TinyBERT-L-2）作为 Student，在 reranking 任务上蒸馏，推理快 5-10 倍
- **自适应计算**：对容易判别的 (Query, Doc) 对提前退出（Early Exit），对模糊对深入计算
- **ONNX/TensorRT 加速**：将 Cross-Encoder 导出为 ONNX 格式，推理速度提升 2-3 倍

### 4. ColBERT 后期交互（Late Interaction）

ColBERT 提出了一种介于 Bi-Encoder 和 Cross-Encoder 之间的折中方案：

```
Query:  [q₁, q₂, q₃, ...]     每个 token 独立编码
Doc:    [d₁, d₂, d₃, ...]     每个 token 独立编码

相似度计算：对每个 q_i，取与所有 d_j 的最大相似度后求和
score = Σᵢ maxⱼ cos(q_i, d_j)

特点：
  - 比 Bi-Encoder 更精确（保留了 token 级匹配信号）
  - 比 Cross-Encoder 更快（Doc 向量可预计算）
  - 支持可解释性（能可视化哪些 token 匹配上了）
```

## 能力边界与结果边界

| 维度 | 小型 CE (100M) | 中型 CE (500M) | 大型 CE/LLM (7B+) |
|------|--------------|--------------|-----------------|
| 模型示例 | MiniLM-L-2 | BGE-Reranker-v2-M3 | RankZephyr / GPT-4 |
| 推理速度 | ⭐⭐⭐⭐⭐ 毫秒级 | ⭐⭐⭐ 毫秒级 | ⭐ 秒级 |
| 精度提升（vs BM25） | +5~10% | +10~20% | +15~30% |
| 跨领域泛化 | ⭐⭐ 一般 | ⭐⭐⭐ 良好 | ⭐⭐⭐⭐⭐ 优秀 |
| 长文档处理 | ⭐⭐ 512 token 限制 | ⭐⭐⭐ 8192 token | ⭐⭐⭐⭐⭐ 32K+ |
| 多语言支持 | ⭐⭐ 依赖训练数据 | ⭐⭐⭐⭐ 较好 | ⭐⭐⭐⭐⭐ 很强 |

### 重排序的"天花板"

重排序并不是万能的，它的质量上限受限于第一阶段召回的结果：
- **如果 Top-100 中不含正确答案，重排序也救不了**——这是"召回天花板"
- 当第一阶段已经达到极高精度（如 Top-5 精度 > 95%），重排序的边际收益很小
- 对于简单事实性问答（如"法国首都是什么"），第一阶段就能直接命中，重排序几乎没有提升

## 与其他策略的区别

- **重排序 vs 过滤（Filtering）**：过滤是丢弃低于阈值的文档，重排序是重新排列所有候选，保留排序信息。过滤可以作为重排序的前置步骤——先用简单规则快速过滤明显不相关文档，减少重排序候选数。

- **重排序 vs 查询改写（Query Rewriting）**：查询改写在检索之前优化用户输入（如扩展同义词、分解复杂问题），重排序在检索之后优化排序结果。两者互补而非互斥。

- **重排序 vs 上下文压缩（Context Compression）**：压缩是在将文档送入 LLM 之前提取关键片段、减少 token 消耗。重排序解决"哪些文档更相关"的问题，压缩解决"相关文档中哪些片段有用"的问题。

- **重排序 vs 提升 Top-K 再截断**：单纯提升第一阶段召回 K 值（如从 K=5 提到 K=50）而不做重排序，虽然增加了召回率但引入了大量噪声，反而降低 LLM 生成质量。**重排序的核心是"从多中选优"而非"单纯多给"**。

## 核心优势

- **显著精度提升 + 适中计算成本**：Cross-Encoder 重排序可使 RAG 系统最终答案质量提升 15-30%，而计算成本仅在几十到几百个候选对的范围内，远低于用 LLM 重新检索或多次生成。
- **即插即用**：不改变第一阶段检索逻辑，作为独立模块插入即可，与任何 embedding + 向量库兼容。
- **领域可微调**：Cross-Encoder 可以针对特定领域（法律、医疗、代码）微调，用小模型达到通用大模型的效果。
- **模型生态丰富**：从 100M 的轻量模型到 7B 的深度模型，从开源到商业 API，适配不同成本预算。

## 工程优化

### 1. 候选池大小调优

```python
def tune_candidate_pool(query: str, docs: list, reranker, 
                         pool_sizes=[20, 50, 100, 200], target_top_n=5):
    """搜索最佳候选池大小：平衡精度与延迟"""
    results = {}
    for k in pool_sizes:
        candidates = docs[:k]
        reranked = reranker.rerank(query, candidates)
        top_n = reranked[:target_top_n]
        
        precision = sum(1 for d in top_n if d["is_relevant"]) / target_top_n
        latency = reranker.last_latency_ms
        
        results[k] = {
            "precision": precision,
            "latency_ms": latency,
            "efficiency": precision / latency * 1000  # 精度/延迟比
        }
    
    # 选择效率最高的配置
    best_k = max(results, key=lambda k: results[k]["efficiency"])
    return best_k, results
```

### 2. Batch 推理

```python
def batch_rerank(query: str, candidates: list, model, tokenizer, batch_size=32):
    """批量推理：将候选文档分组送入 Cross-Encoder"""
    all_scores = []
    
    for i in range(0, len(candidates), batch_size):
        batch = candidates[i:i + batch_size]
        pairs = [(query, doc["text"]) for doc in batch]
        
        inputs = tokenizer(
            pairs,
            return_tensors="pt",
            truncation=True,
            max_length=512,
            padding=True
        )
        
        with torch.no_grad():
            outputs = model(**inputs)
            scores = outputs.logits.squeeze(-1).tolist()
            if isinstance(scores, float):
                scores = [scores]
            all_scores.extend(scores)
    
    # 按分数降序排列
    scored = list(zip(candidates, all_scores))
    scored.sort(key=lambda x: x[1], reverse=True)
    
    return [{"doc": doc, "score": score} for doc, score in scored]
```

### 3. 模型蒸馏

```python
# 用大模型（Teacher）蒸馏小模型（Student）用于重排序
# 伪代码示意
def distill_reranker(teacher_model, student_model, unlabeled_pairs):
    """
    步骤：
    1. 用 Teacher 对大量 (Query, Doc) 对打分 → 软标签
    2. 用软标签训练 Student，损失函数为 KL 散度
    3. 部署 Student 模型（推理快 5-10 倍）
    """
    for batch in unlabeled_pairs:
        with torch.no_grad():
            teacher_scores = teacher_model(batch)  # 软标签分数
        
        student_scores = student_model(batch)
        loss = kl_divergence(student_scores, teacher_scores)
        loss.backward()
        optimizer.step()
    
    return student_model  # 推理时用 Student
```

### 4. 缓存机制

```python
from functools import lru_cache
import hashlib
import json

class RerankerCache:
    """重排序结果缓存：相同的 (Query, Doc) 对避免重复计算"""
    
    def __init__(self, cache_size=10000):
        self.cache = {}
        self.cache_size = cache_size
    
    def _make_key(self, query: str, doc_id: str, doc_text: str) -> str:
        content = f"{query}|||{doc_id}|||{doc_text[:200]}"  # 前缀防过长
        return hashlib.md5(content.encode()).hexdigest()
    
    def get_score(self, query: str, doc_id: str, doc_text: str) -> float:
        key = self._make_key(query, doc_id, doc_text)
        return self.cache.get(key)
    
    def set_score(self, query: str, doc_id: str, doc_text: str, score: float):
        key = self._make_key(query, doc_id, doc_text)
        if len(self.cache) >= self.cache_size:
            # LRU 淘汰：移除最早的一半
            keys = list(self.cache.keys())
            for k in keys[:len(keys)//2]:
                del self.cache[k]
        self.cache[key] = score
```

### 5. 级联重排序（Cascading Rerankers）

```python
def cascading_rerank(query: str, candidates: list, stages: list):
    """
    级联重排序：用轻量模型先快速粗排减少候选数，再用重量模型精排
    
    stages: [
        {"model": light_model, "keep_top_k": 50},   # 100 -> 50
        {"model": medium_model, "keep_top_k": 20},   # 50  -> 20
        {"model": heavy_model, "keep_top_k": 5},     # 20  -> 5
    ]
    """
    current_candidates = candidates
    
    for stage in stages:
        model = stage["model"]
        keep_k = stage["keep_top_k"]
        
        scores = model.predict([(query, d["text"]) for d in current_candidates])
        
        scored = list(zip(current_candidates, scores))
        scored.sort(key=lambda x: x[1], reverse=True)
        current_candidates = [doc for doc, _ in scored[:keep_k]]
    
    return current_candidates  # 最终 Top-5
```

## 场景适配指南

| 场景 | 推荐重排序方案 | 候选池 K | 模型建议 |
|------|-------------|---------|---------|
| 实时客服问答 | 轻量 Cross-Encoder | 20-30 | MiniLM-L-6, BGE-Reranker-v2-Flash |
| 通用知识库 QA | 中型 Cross-Encoder | 50-100 | BGE-Reranker-v2-M3, mxbai-rerank |
| 企业文档搜索 | 级联方案 | 100-200 | 第一级 MiniLM，第二级 BGE-v2-M3 |
| 学术论文检索 | 大型 Cross-Encoder | 100-200 | BGE-Reranker-v2-Gemma, Cohere Rerank |
| 法律合同审查 | 中型 CE + 领域微调 | 50-100 | 基于法律语料微调的 Cross-Encoder |
| 代码检索 QA | 中型 CE | 30-50 | CodeBERT-based Cross-Encoder |
| 高精度离线处理 | LLM-as-Reranker | 100-200 | GPT-4 / Qwen listwise 排序 |
| 多语言场景 | 多语言专用模型 | 50-100 | BGE-Reranker-v2-M3, Jina Reranker v3 |

---

## 示例代码

### 示例 1：Cross-Encoder 重排序（sentence-transformers）

```python
from sentence_transformers import CrossEncoder
import numpy as np

# 加载 Cross-Encoder 模型
model = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2", max_length=512)

query = "什么是反向传播算法？"
candidates = [
    "前向传播是神经网络计算预测值的过程...",
    "反向传播（Backpropagation）通过链式法则计算梯度，是训练神经网络的核心算法...",
    "梯度下降是一种优化算法，用于最小化损失函数...",
    "反向传播算法由 Rumelhart 等人在 1986 年提出，它利用链式法则从输出层向输入层逐层传播误差梯度...",
    "神经网络由多层神经元组成，每层包含多个神经元节点..."
]

# 构造 (query, doc) 对
pairs = [(query, doc) for doc in candidates]

# 推理打分
scores = model.predict(pairs)

# 按分数排序
ranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)

print("重排序结果：")
for i, (doc, score) in enumerate(ranked, 1):
    print(f"  #{i} 分数={score:.4f} | {doc[:60]}...")
```

### 示例 2：Cohere Rerank API

```python
import cohere

co = cohere.Client("YOUR_API_KEY")

query = "What is the capital of France?"
documents = [
    "Paris is the capital and most populous city of France.",
    "France is a country in Western Europe.",
    "The Eiffel Tower is located in Paris.",
    "Lyon is a major city in France known for its cuisine."
]

# Cohere Rerank API v3
results = co.rerank(
    model="rerank-v3.5",  # 或 "rerank-v4-fast"
    query=query,
    documents=documents,
    top_n=3,               # 返回 Top-3
    max_chunks_per_doc=5   # 长文档分块，每块最多 512 tokens
)

print("重排序结果：")
for result in results.results:
    print(f"  索引={result.index} 分数={result.relevance_score:.4f}")
    print(f"  内容: {documents[result.index][:80]}...")
```

### 示例 3：BGE-Reranker 使用

```python
# BGE-Reranker 推荐的使用方式
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

model_name = "BAAI/bge-reranker-v2-m3"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name)
model.eval()

query = "机器学习中的过拟合问题"
docs = [
    "过拟合是指模型在训练数据上表现很好但在测试数据上表现差的现象...",
    "正则化技术通过在损失函数中加入惩罚项来防止过拟合...",
    "交叉验证是评估模型泛化能力的标准方法...",
]

pairs = [[query, doc] for doc in docs]

with torch.no_grad():
    inputs = tokenizer(
        pairs, 
        padding=True, 
        truncation=True, 
        return_tensors="pt", 
        max_length=512
    )
    scores = model(**inputs, return_dict=True).logits.view(-1).float()

scores_list = scores.tolist()
ranked = sorted(zip(docs, scores_list), key=lambda x: x[1], reverse=True)

print("BGE-Reranker-v2-M3 排序结果：")
for i, (doc, score) in enumerate(ranked, 1):
    print(f"  #{i} 分数={score:.4f} | {doc[:50]}...")
```

### 示例 4：LLM-as-Reranker（Pointwise 模式 + OpenAI API）

```python
from openai import OpenAI
import json

client = OpenAI(api_key="YOUR_API_KEY")

def llm_pointwise_rerank(query: str, documents: list, top_n: int = 5):
    """用 LLM 对每个文档独立打分（Pointwise 模式）"""
    
    scored_docs = []
    
    for idx, doc in enumerate(documents):
        prompt = f"""你是一个文档相关性评估专家。请评估以下文档与用户问题的相关性。
输出一个 0-10 的整数分数（0=完全不相关，10=完美匹配）。

用户问题: {query}

文档内容: {doc}

请只输出整数分数，不要包含其他内容。"""
        
        response = client.chat.completions.create(
            model="gpt-4o-mini",  # 使用 mini 模型降低成本
            messages=[{"role": "user", "content": prompt}],
            temperature=0,
            max_tokens=10
        )
        
        try:
            score = int(response.choices[0].message.content.strip())
        except ValueError:
            score = 0
        
        scored_docs.append({"doc": doc, "score": score, "index": idx})
    
    # 按分数降序排列
    scored_docs.sort(key=lambda x: x["score"], reverse=True)
    return scored_docs[:top_n]


# Listwise 模式：一次排序所有文档
def llm_listwise_rerank(query: str, documents: list, top_n: int = 5):
    """用 LLM 对文档列表整体排序（Listwise 模式）"""
    
    doc_list_text = "\n".join([
        f"[{i}] {doc[:200]}" for i, doc in enumerate(documents)
    ])
    
    prompt = f"""请按与问题的相关性从高到低重新排列以下文档。

问题: {query}

文档列表:
{doc_list_text}

请只输出重新排序后的文档索引（从最相关到最不相关），
以逗号分隔，例如: 3, 1, 0, 2, ...
不要输出其他内容。"""
    
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        temperature=0,
        max_tokens=100
    )
    
    # 解析排序后的索引
    indices = []
    for part in response.choices[0].message.content.strip().split(","):
        part = part.strip()
        if part.isdigit():
            indices.append(int(part))
    
    return [{"doc": documents[i], "index": i} for i in indices[:top_n]]
```

### 示例 5：完整重排序管线（Retrieve → Rerank → Top-K）

```python
from sentence_transformers import CrossEncoder, SentenceTransformer
import faiss
import numpy as np
from typing import List, Dict, Optional

class RAGRetrievalPipeline:
    """完整检索管线：Bi-Encoder 召回 + Cross-Encoder 重排序"""
    
    def __init__(
        self,
        embedding_model_name: str = "BAAI/bge-small-zh-v1.5",
        reranker_model_name: str = "BAAI/bge-reranker-v2-m3",
        device: str = "cpu"
    ):
        # 第一阶段：Bi-Encoder（快速召回）
        self.embedder = SentenceTransformer(embedding_model_name, device=device)
        
        # 第二阶段：Cross-Encoder（精确重排）
        self.reranker = CrossEncoder(reranker_model_name, max_length=512, device=device)
        
        self.index = None
        self.documents = []
        self.embedding_dim = 0
    
    def build_index(self, documents: List[Dict[str, str]]):
        """构建向量索引（离线预处理）"""
        self.documents = documents
        texts = [doc["text"] for doc in documents]
        
        print(f"正在编码 {len(texts)} 篇文档...")
        embeddings = self.embedder.encode(texts, show_progress_bar=True, batch_size=64)
        self.embedding_dim = embeddings.shape[1]
        
        # 构建 Faiss 索引
        self.index = faiss.IndexFlatIP(self.embedding_dim)  # 内积=余弦（归一化后）
        faiss.normalize_L2(embeddings)
        self.index.add(embeddings)
        
        print(f"索引构建完成，共 {self.index.ntotal} 篇文档")
    
    def retrieve(
        self,
        query: str,
        retrieve_k: int = 100,
        rerank_k: int = 10,
        final_k: int = 5
    ) -> List[Dict]:
        """
        完整检索流程：
        1. Bi-Encoder 召回 Top-K（retrieve_k）
        2. Cross-Encoder 重排序 Top-K（rerank_k）
        3. 返回最终 Top-N（final_k）
        """
        # 阶段一：Bi-Encoder 快速召回
        query_vec = self.embedder.encode(query, normalize_embeddings=True)
        query_vec = query_vec.reshape(1, -1).astype(np.float32)
        
        scores, indices = self.index.search(query_vec, retrieve_k)
        
        candidates = []
        for score, idx in zip(scores[0], indices[0]):
            if idx != -1:
                candidates.append({
                    "doc": self.documents[idx],
                    "retrieve_score": float(score),
                    "index": int(idx)
                })
        
        print(f"阶段一：召回 {len(candidates)} 篇候选文档")
        
        # 阶段二：Cross-Encoder 精确重排
        pairs = [(query, cand["doc"]["text"]) for cand in candidates]
        rerank_scores = self.reranker.predict(pairs)
        
        for cand, score in zip(candidates, rerank_scores):
            cand["rerank_score"] = float(score)
        
        # 按重排序分数降序排列
        candidates.sort(key=lambda x: x["rerank_score"], reverse=True)
        
        # 返回最终 Top-K
        final_results = candidates[:final_k]
        
        print(f"阶段二：重排序完成，返回 Top-{final_k}")
        
        return final_results


# 使用示例
pipeline = RAGRetrievalPipeline()

docs = [
    {"id": "1", "text": "反向传播通过链式法则计算梯度..."},
    {"id": "2", "text": "注意力机制让模型聚焦于输入的关键部分..."},
    # ... 更多文档
]

pipeline.build_index(docs)

results = pipeline.retrieve(
    query="什么是反向传播？",
    retrieve_k=100,   # 召回 100 篇
    rerank_k=10,      # 重排后保留 10 篇
    final_k=5         # 最终给 LLM 5 篇
)

for i, r in enumerate(results, 1):
    print(f"#{i} 文档[{r['doc']['id']}] "
          f"检索分={r['retrieve_score']:.4f} "
          f"重排分={r['rerank_score']:.4f}")
```

### 示例 6：重排序结果缓存

```python
import time
import hashlib
import pickle
from pathlib import Path

class PersistentRerankerCache:
    """持久化重排序缓存：避免重复计算相同 (Query, Doc) 对"""
    
    def __init__(self, cache_path: str = "./reranker_cache.pkl"):
        self.cache_path = Path(cache_path)
        self.cache = self._load_cache()
        self.hits = 0
        self.misses = 0
    
    def _load_cache(self) -> dict:
        if self.cache_path.exists():
            with open(self.cache_path, "rb") as f:
                return pickle.load(f)
        return {}
    
    def _save_cache(self):
        with open(self.cache_path, "wb") as f:
            pickle.dump(self.cache, f)
    
    def _make_key(self, query: str, doc_text: str) -> str:
        content = f"{query.strip().lower()}|||{doc_text.strip()[:300]}"
        return hashlib.sha256(content.encode()).hexdigest()
    
    def rerank_with_cache(self, query: str, candidates: list, reranker_fn) -> list:
        """带缓存的重排序"""
        results = []
        uncached_indices = []
        
        for i, doc in enumerate(candidates):
            key = self._make_key(query, doc["text"])
            if key in self.cache:
                doc["rerank_score"] = self.cache[key]
                results.append(doc)
                self.hits += 1
            else:
                uncached_indices.append(i)
        
        # 对未缓存的文档进行重排序
        if uncached_indices:
            uncached = [candidates[i] for i in uncached_indices]
            uncached_texts = [doc["text"] for doc in uncached]
            
            scores = reranker_fn(query, uncached_texts)
            
            for idx, score in zip(uncached_indices, scores):
                candidates[idx]["rerank_score"] = score
                key = self._make_key(query, candidates[idx]["text"])
                self.cache[key] = score
                self.misses += 1
            
            results = [candidates[i] for i in range(len(candidates))
                      if i in set(uncached_indices) | set(r["index"] for r in results)]
        
        # 按分数排序
        results.sort(key=lambda x: x["rerank_score"], reverse=True)
        
        # 定期持久化
        if (self.hits + self.misses) % 100 == 0:
            self._save_cache()
        
        hit_rate = self.hits / (self.hits + self.misses) * 100 if (self.hits + self.misses) > 0 else 0
        print(f"缓存命中率: {hit_rate:.1f}% ({self.hits}/{self.hits + self.misses})")
        
        return results
```

### 示例 7：不同重排序器性能对比

```python
import time
import torch
from sentence_transformers import CrossEncoder

def benchmark_rerankers(query: str, documents: list):
    """对比不同重排序模型的延迟和精度"""
    
    models = {
        "MiniLM-L-2 (22M)": "cross-encoder/ms-marco-MiniLM-L-2-v2",
        "MiniLM-L-6 (80M)": "cross-encoder/ms-marco-MiniLM-L-6-v2",
        "MiniLM-L-12 (160M)": "cross-encoder/ms-marco-MiniLM-L-12-v2",
        "BGE-Reranker-v2-M3 (568M)": "BAAI/bge-reranker-v2-m3",
    }
    
    results = {}
    pairs = [(query, doc["text"]) for doc in documents]
    
    for name, model_name in models.items():
        try:
            # 加载模型
            device = "cuda" if torch.cuda.is_available() else "cpu"
            model = CrossEncoder(model_name, max_length=512, device=device)
            
            # 预热
            _ = model.predict(pairs[:2])
            
            # 计时
            start = time.perf_counter()
            scores = model.predict(pairs)
            elapsed = time.perf_counter() - start
            
            results[name] = {
                "time_ms": elapsed * 1000,
                "time_per_pair_ms": elapsed * 1000 / len(pairs),
                "scores": scores.tolist(),
            }
            
            print(f"{name:30s} | {len(pairs)} 对 | "
                  f"{elapsed*1000:7.1f}ms 总计 | "
                  f"{elapsed*1000/len(pairs):5.1f}ms/对")
        
        except Exception as e:
            print(f"{name:30s} | 加载失败: {e}")
    
    return results


# 模拟运行
if __name__ == "__main__":
    test_query = "什么是强化学习中的策略梯度方法？"
    test_docs = [
        {"id": str(i), "text": f"这是第 {i} 篇候选文档的内容..."} 
        for i in range(100)
    ]
    
    print("=" * 70)
    print("重排序模型性能基准测试")
    print(f"候选文档数: {len(test_docs)}")
    print("=" * 70)
    
    benchmark_rerankers(test_query, test_docs)
```

---

## 小结

重排序是 RAG 系统中"性价比最高"的优化手段之一——用适中的计算成本换取 15-30% 的检索精度提升。其核心设计决策包括：

1. **候选池大小**：50-100 是多数场景的 sweet spot
2. **模型大小**：从 22M 到 7B，根据延迟预算选择
3. **Pointwise vs Listwise**：Pointwise 通用高效，Listwise 质量更高但更慢
4. **缓存机制**：高复用场景下缓存重排序结果可大幅降低延迟
5. **级联策略**：多级重排序在精度和效率间取得最佳平衡

当前最佳实践是"Bi-Encoder 粗召 + Cross-Encoder 精排"的两阶段范式，其中 BGE-Reranker-v2-M3（开源）和 Cohere Rerank（商业）是广泛采用的标杆方案。
