# 9.2.2 sparse-retrieval — 稀疏检索：BM25 / TF-IDF / 学习型稀疏检索

## 简单介绍

稀疏检索（Sparse Retrieval）是信息检索领域最经典的检索范式，核心思想是**将文档和查询表示为高维稀疏向量**——向量维度等于词汇表大小，每个维度对应一个词项，非零值表示该词在文档中的重要性权重。与稠密检索（Dense Retrieval）不同，稀疏检索不依赖语义嵌入，而是基于词频统计和倒排索引实现快速匹配。

稀疏检索的典型代表包括 TF-IDF、BM25 及其变体，以及近年出现的**学习型稀疏检索（Learned Sparse Retrieval）**如 SPLADE、uniCOIL、DeepImpact。这些方法将深度学习的表达能力与传统倒排索引的效率相结合，正成为 RAG 系统中不可替代的一环。

## 基本原理

### 倒排索引（Inverted Index）

稀疏检索的核心数据结构是**倒排索引**。与正向索引（doc → terms）相反，倒排索引是 terms → docs 的映射：

```
正向索引:  Doc1 → {the, cat, sat, on, mat}
           Doc2 → {the, dog, ran}

倒排索引:  the  → {Doc1, Doc2}
           cat  → {Doc1}
           sat  → {Doc1}
           on   → {Doc1}
           mat  → {Doc1}
           dog  → {Doc2}
           ran  → {Doc2}
```

```python
# 从零构建倒排索引
from collections import defaultdict, Counter
import math
import re

def build_inverted_index(documents: list[str]) -> dict:
    """构建倒排索引：词项 → [(doc_id, 词频), ...]"""
    index = defaultdict(list)
    doc_lengths = []
    
    for doc_id, doc in enumerate(documents):
        terms = doc.lower().split()
        doc_lengths.append(len(terms))
        tf = Counter(terms)
        
        for term, freq in tf.items():
            index[term].append((doc_id, freq))
    
    return dict(index), doc_lengths

# 倒排索引检索
def search_inverted(index: dict, query: str, doc_lengths: list, N: int):
    """基于倒排索引的布尔检索 + TF 求和"""
    terms = query.lower().split()
    scores = defaultdict(float)
    
    for term in terms:
        if term in index:
            for doc_id, freq in index[term]:
                scores[doc_id] += freq  # 简单 TF 求和
    
    return sorted(scores.items(), key=lambda x: -x[1])
```

### TF-IDF

TF-IDF（Term Frequency-Inverse Document Frequency）是最经典的词项权重计算方法：

- **词频 TF(t,d)**：词项 t 在文档 d 中出现的次数（或归一化值）
- **逆文档频率 IDF(t)**：log(N / df(t))，其中 N 是文档总数，df(t) 是包含词项 t 的文档数

TF-IDF 的核心思想是：**一个词项在文档中出现的次数越多（TF 高），且在整个文档集合中出现的文档越少（IDF 高），则该词项对文档的区分度越高**。

```python
# TF-IDF 从头实现
class TfidfVectorizer:
    def __init__(self, documents: list[str]):
        self.documents = documents
        self.N = len(documents)
        self.vocab = {}
        self.idf = {}
        self._build_vocab()
        self._compute_idf()
    
    def _build_vocab(self):
        """构建词汇表"""
        vocab_set = set()
        for doc in self.documents:
            for term in doc.lower().split():
                vocab_set.add(term)
        self.vocab = {term: idx for idx, term in enumerate(sorted(vocab_set))}
    
    def _compute_idf(self):
        """计算每个词项的 IDF"""
        df = defaultdict(int)
        for doc in self.documents:
            terms = set(doc.lower().split())
            for term in terms:
                df[term] += 1
        
        for term, doc_freq in df.items():
            self.idf[term] = math.log((self.N + 1) / (doc_freq + 1)) + 1
    
    def transform(self, text: str) -> dict:
        """将文本转换为 TF-IDF 稀疏向量"""
        terms = text.lower().split()
        tf = Counter(terms)
        max_tf = max(tf.values()) if tf else 1
        
        vector = {}
        for term, freq in tf.items():
            if term in self.vocab:
                tf_norm = freq / max_tf  # 归一化 TF
                vector[self.vocab[term]] = tf_norm * self.idf.get(term, 0)
        
        return vector
```

### BM25 算法详解

BM25（Best Matching 25）是概率检索模型的代表，也是目前最广泛使用的稀疏检索算法。BM25 的核心评分公式：

```
Score(Q, D) = Σ [ IDF(q_i) × TF_BM25(q_i, D) ]

其中：
TF_BM25(q_i, D) = f(q_i, D) × (k1 + 1) / (f(q_i, D) + k1 × (1 - b + b × |D| / avg_dl))

IDF(q_i) = log((N - n(q_i) + 0.5) / (n(q_i) + 0.5) + 1)
```

参数含义：
- **f(q_i, D)**：词项 q_i 在文档 D 中的出现频率
- **|D|**：文档 D 的长度（词数）
- **avg_dl**：文档集合的平均长度
- **N**：文档集合中的文档总数
- **n(q_i)**：包含词项 q_i 的文档数
- **k1**：控制词频饱和度的参数（默认 1.2 ~ 2.0）
- **b**：控制文档长度归一化的强度（默认 0.75）

#### 公式深度解读

**TF 部分的 k1 饱和效应**：高频词带来的收益是递减的。例如，"检索" 一词在文档中出现 10 次并不是出现 1 次的 10 倍相关。k1 控制了这个饱和速度——k1 越小，饱和越快。

**长度归一化的 b 参数**：b = 0 时不进行长度归一化（长文档因词项多天然分数更高）；b = 1 时完全归一化。默认 b = 0.75 是因为经验表明大多数场景下长文档需要适度惩罚（部分长文档包含更多无关内容）。

```python
# BM25 从头实现
class BM25:
    def __init__(self, documents: list[str], k1: float = 1.5, b: float = 0.75):
        self.k1 = k1
        self.b = b
        self.documents = documents
        self.N = len(documents)
        self.avg_dl = 0
        self.doc_lengths = []
        self.df = defaultdict(int)    # document frequency
        self.doc_term_freqs = []      # term frequency per doc
        self._build_index()
    
    def _build_index(self):
        """构建索引并计算统计数据"""
        for doc in self.documents:
            terms = doc.lower().split()
            self.doc_lengths.append(len(terms))
            tf = Counter(terms)
            self.doc_term_freqs.append(tf)
            for term in set(terms):
                self.df[term] += 1
        
        self.avg_dl = sum(self.doc_lengths) / self.N
    
    def _idf(self, term: str) -> float:
        """BM25 IDF 公式"""
        n = self.df.get(term, 0)
        return math.log((self.N - n + 0.5) / (n + 0.5) + 1)
    
    def score(self, query: str, doc_id: int) -> float:
        """计算查询与文档的 BM25 相关性得分"""
        query_terms = query.lower().split()
        score = 0.0
        dl = self.doc_lengths[doc_id]
        tf_dict = self.doc_term_freqs[doc_id]
        
        for term in query_terms:
            if term in tf_dict:
                tf = tf_dict[term]
                # BM25 TF 部分
                tf_norm = tf * (self.k1 + 1) / (tf + self.k1 * (1 - self.b + self.b * dl / self.avg_dl))
                score += self._idf(term) * tf_norm
        
        return score
    
    def search(self, query: str, top_k: int = 10) -> list[tuple[int, float]]:
        """检索 Top-K 文档"""
        scores = [(doc_id, self.score(query, doc_id)) for doc_id in range(self.N)]
        scores.sort(key=lambda x: -x[1])
        return scores[:top_k]


# 使用 rank_bm25 库
# pip install rank_bm25
from rank_bm25 import BM25Okapi, BM25L, BM25Plus

corpus = [
    "稀疏检索是 RAG 系统的重要组件",
    "BM25 是最经典的稀疏检索算法",
    "稠密检索基于语义嵌入向量",
    "SPLADE 是学习型稀疏检索的代表"
]

tokenized_corpus = [doc.split() for doc in corpus]
bm25 = BM25Okapi(tokenized_corpus, k1=1.5, b=0.75)

query = "稀疏检索 BM25"
tokenized_query = query.split()
scores = bm25.get_scores(tokenized_query)
top_k = bm25.get_top_n(tokenized_query, corpus, n=3)

print("Top 检索结果:", top_k)
```

### TF-IDF 使用 scikit-learn

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

corpus = [
    "稀疏检索（Sparse Retrieval）使用词频统计进行匹配",
    "稠密检索（Dense Retrieval）使用语义嵌入进行匹配",
    "BM25 是稀疏检索中最流行的排序算法",
    "学习型稀疏检索结合了深度学习的表达能力",
]

vectorizer = TfidfVectorizer(
    analyzer='char_wb',    # 基于字符的 n-gram（保留词边界）
    ngram_range=(1, 2),   # unigram + bigram
    max_features=5000,
    sublinear_tf=True,     # 使用 1 + log(tf) 替代原始 tf
)

doc_vectors = vectorizer.fit_transform(corpus)
query = "稀疏检索 BM25"
query_vector = vectorizer.transform([query])
scores = cosine_similarity(query_vector, doc_vectors).flatten()

for i, score in enumerate(scores):
    print(f"Doc {i}: {corpus[i][:30]}... → score={score:.4f}")
```

## 背景与演进

```
Boolean Model (1950s)
    │   精确布尔匹配，无排序
    │
    ├── Vector Space Model (1970s, Salton)
    │    TF-IDF 权重，余弦相似度排序
    │    │
    │    ├── BM25 (1990s, Robertson & Sparck Jones)
    │    │   概率检索模型，引入词频饱和 + 长度归一化
    │    │    │
    │    │    ├── BM25F (2004) — 支持结构化字段权重
    │    │    ├── BM25L (2015) — 改进长文档的 TF 饱和
    │    │    └── BM25+ (2015) — 添加下界平滑避免零分
    │    │
    │    └── 学习型稀疏检索 (2020-2024)
    │         SPLADE, uniCOIL, DeepImpact
    │         用 Transformer 实现 query/doc 扩展
    │
    └── 稠密检索 (2019-)
         DPR, ColBERT, Contriever
         语义嵌入向量检索
```

### 关键演进里程碑

1. **Boolean Model（1950s）**：最早的检索模型，基于布尔逻辑（AND/OR/NOT）精确匹配。没有排序能力，结果集要么包含要么不包含。

2. **Vector Space Model（1970s, Gerard Salton）**：将文档和查询映射到高维词项空间，用 TF-IDF 计算权重，用余弦相似度排序。首次实现**相关性排序**。

3. **BM25（1990s, Robertson & Zaragoza）**：在概率检索框架（PRF）下提出的 BM25 算法，引入两个关键创新——**词频饱和**（k1 参数控制高频词收益递减）和**文档长度归一化**（b 参数控制）。BM25 在此后三十年间一直是信息检索领域的事实标准。

4. **BM25 变体（2015）**：
   - **BM25F**：将 BM25 扩展到结构化文档。不同字段（标题、正文、标签）使用不同的权重和 b 参数，适合 XML/HTML 文档检索
   - **BM25L**：针对长文档改进。在 TF 公式中加入 +1 常数，避免长文档中高频词的过度惩罚
   - **BM25+**：在 TF 公式中加入下界 δ（通常 0.5~1.0），确保即使词频很低也能获得基础分数

5. **学习型稀疏检索（2020-2024）**：利用 Transformer 模型自动进行**查询扩展（Query Expansion）**和**文档扩展（Document Expansion）**，输出仍然是稀疏向量，兼容倒排索引。

## 核心矛盾

### 词汇鸿沟（Lexical Gap / Vocabulary Mismatch）

稀疏检索最根本的局限性：**查询和文档使用不同词汇描述同一概念时，稀疏检索无法匹配**。

```
查询："如何修理汽车发动机"
文档："汽车引擎维修指南"

BM25 匹配情况：
  "修理"  → 文档中有 "维修"（不同词 → 不匹配）
  "发动机" → 文档中有 "引擎"（同义词 → 不匹配）
  "汽车"  → 文档中有 "汽车"（相同词 → 匹配）

结果：3 个关键概念中仅 1 个匹配，相关性评分偏低
```

这就是**词汇鸿沟**问题（Vocabulary Mismatch Problem）：
- **同义词**：""发动机"" vs ""引擎""，""电脑"" vs ""计算机""
- **上下位词**：""水果"" vs ""苹果""，""动物"" vs ""狗""
- **形态变化**：""run"" vs ""running""，""analyse"" vs ""analysis""
- **缩写**：""IR"" vs ""Information Retrieval""，""LLM"" vs ""Large Language Model""
- **翻译差异**：""深度学习"" vs ""Deep Learning""
- **专业术语偏好**：不同学术圈对同一概念使用不同术语

### 稀疏检索的其他核心矛盾

| 矛盾 | 描述 | 影响 |
|------|------|------|
| 精确匹配 vs 语义泛化 | 匹配准确但漏掉同义 | 高精度低召回 |
| 统计相关 vs 语义相关 | 词频高≠语义相关 | TF 分数可能误导 |
| 稀疏性 vs 存储效率 | 向量维度大但非零值少 | 倒排索引高效但丢失语义关系 |
| 词汇表覆盖 vs 时效性 | 新词/专业术语需更新词汇表 | 冷启动问题 |

## 主流优化方向

### 学习型稀疏检索（Learned Sparse Retrieval）

学习型稀疏检索的核心思路：**用深度模型学习查询和文档的"扩展表示"**——不再局限于文本中实际出现的词汇，而是学会为每个输入激活词表中的相关词汇，然后在这些激活的词汇上构建稀疏向量。

#### SPLADE（SParse Lexical AnD Expansion）

SPLADE（2021, Formal et al.）是学习型稀疏检索的代表作，基于 BERT 架构：

**核心机制**：
1. 将查询/文档输入 BERT 编码
2. 取 BERT 最后一层隐藏状态的输出，通过一个线性层映射到词汇表空间
3. 对所有 token 位置的输出进行 **log-saturation + max pooling**（取跨位置的最大值）
4. 训练时加入 **FLOPS 正则化**惩罚过多非零值（控制稀疏度）

```
原始查询："car engine repair"
         │
    [BERT Encoder]
         │
    [Linear Projection → vocab_size]
         │
    [Log-saturation + Max Pooling]
         │
    稀疏向量: {car:2.3, engine:2.1, repair:1.8, 
               auto:0.9, motor:0.7, fix:0.6, 
               vehicle:0.5, mechanic:0.4, ...}
                      ↑
               自动扩展的同义词/相关词
```

```python
# SPLADE 推理示例（使用 HuggingFace Transformers）
from transformers import AutoTokenizer, AutoModelForMaskedLM
import torch
import torch.nn.functional as F

class SPLADE:
    def __init__(self, model_name: str = "naver/splade-v3"):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForMaskedLM.from_pretrained(model_name)
        self.model.eval()
    
    def encode(self, text: str) -> dict[str, float]:
        """将文本编码为稀疏向量 {词项: 权重}"""
        inputs = self.tokenizer(
            text, 
            return_tensors="pt", 
            truncation=True, 
            max_length=512
        )
        
        with torch.no_grad():
            outputs = self.model(**inputs)
            logits = outputs.logits  # [1, seq_len, vocab_size]
        
        # Log-saturation + Max Pooling
        # 1. ReLU 激活
        relu_logits = F.relu(logits)  # [1, seq_len, vocab_size]
        # 2. log(1 + x) 饱和
        saturated = torch.log1p(relu_logits)  # [1, seq_len, vocab_size]
        # 3. 跨 token 位置取最大值
        max_vals, _ = torch.max(saturated, dim=1)  # [1, vocab_size]
        
        # 提取非零维度
        indices = torch.where(max_vals.squeeze() > 0)[0]
        weights = max_vals.squeeze()[indices]
        
        # 映射回词汇
        sparse_vec = {}
        for idx, weight in zip(indices.tolist(), weights.tolist()):
            token = self.tokenizer.decode([idx])
            if token and not token.startswith("["):  # 排除特殊 token
                sparse_vec[token] = round(weight, 4)
        
        return dict(sorted(sparse_vec.items(), key=lambda x: -x[1]))


# 使用 SPLADE
splade = SPLADE("naver/splade-v3")
query_vec = splade.encode("car engine repair")
print("SPLADE 扩展结果:", list(query_vec.items())[:10])
# 输出示例: [("car", 2.3), ("engine", 2.1), ("repair", 1.8), 
#             ("auto", 0.9), ("motor", 0.7), ("fix", 0.6), ...]
```

**SPLADE 系列演进**：
- **SPLADE-v1**（2021）：基于 BERT-base，使用 MLM head 投影到词汇空间
- **SPLADE-v2**（2022）：改进训练策略（蒸馏 + 硬负样本挖掘），显著提升效果
- **SPLADE-v3**（2023-2024）：进一步优化，加入查询编码器权重共享，SPLADE-Index 包实现超快速检索
- **DistilSPLADE**：使用 DistilBERT 蒸馏，在保持效果的同时大幅降低推理成本

#### uniCOIL（Contextualized Inverted List）

uniCOIL（2022, Lin & Ma）的思路更直接：
1. 使用 BERT 编码器为 query 和 doc 中的每个词项生成一个上下文相关的权重
2. **用一个线性层替代 IDF**，直接预测每个词项对检索的重要性
3. 输出仍然是稀疏向量，但权重是学习得到的而非统计得到的

与 SPLADE 的区别：uniCOIL 不做词汇扩展，只对已出现的词项加权；SPLADE 可以进行词汇扩展。

#### DeepImpact

DeepImpact（2022, Mallia et al.）：
1. 使用 BERT 编码文档段落
2. 将词项上下文表示投影到一个权重 + 一个"impact score"
3. 训练时使用 pairwise ranking loss
4. 特别注重**索引效率**——预计算文档的词项权重，检索时直接使用

### 各种方法对比

| 方法 | 词汇扩展 | 查询端开销 | 索引端开销 | 效果 | 存储 |
|------|---------|-----------|-----------|------|------|
| BM25 | 无 | 无 | 无 | 基准线 | 极低 |
| SPLADE | 有（双向） | 高（BERT 编码） | 高（全文档编码） | ⭐⭐⭐⭐⭐ | 中 |
| uniCOIL | 无 | 中（单层编码） | 高 | ⭐⭐⭐⭐ | 中 |
| DeepImpact | 有限 | 无 | 高 | ⭐⭐⭐⭐ | 中-低 |
| COIL | 无（但上下文感知） | 中 | 中 | ⭐⭐⭐⭐ | 中 |

## 能力边界与结果边界

### 稀疏检索表现优异的领域

稀疏检索在以下场景中仍然不可替代：

```
领域          ───  原因
─────────────────────────────────────────
法律检索      ───  法条编号、案号、关键词精确匹配
医疗文献检索  ───  医学术语、ICD 编码、药物名称
专利搜索      ───  专利号、IPC 分类号、技术术语精确匹配
商品搜索      ───  产品型号、品牌名、SKU 精确匹配
学术文献      ───  引文、DOI、作者名、特定术语
代码搜索      ───  函数名、变量名、API 调用精确匹配
合同审查      ───  条款编号、金额数字、日期精确匹配
```

### 检索场景能力矩阵

| 场景 | BM25 | SPLADE | 稠密检索（DPR） | 最优选择 |
|------|------|--------|----------------|---------|
| 精确术语匹配（"第 XX 条"） | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | BM25 |
| 同义/近义检索 | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 稠密/SPLADE |
| 跨语言检索 | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | 稠密 |
| 罕见词/新词 | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | BM25 |
| 长文档检索 | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | SPLADE |
| 低资源领域 | ⭐⭐⭐⭐ | ⭐⭐ | ⭐ | BM25（无需训练） |
| 多语义/模糊查询 | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | 稠密 |

### 稀疏 vs 稠密的根本差异

```ascii
稀疏检索:
  向量维度 = |词汇表|（通常 50K ~ 500K）
  非零值数 = 文档中的词项数（通常几十到几百）
  存储: {term_id: weight, ...}
  匹配: 精确词项匹配 + 统计加权

稠密检索:
  向量维度 = 嵌入维度（通常 768 ~ 1024）
  非零值数 = 全部维度都是非零（密集向量）
  存储: [0.12, -0.05, 0.33, ...]（浮点数组）
  匹配: 语义空间中的拓扑接近度
```

## 与稠密检索的区别

| 维度 | 稀疏检索（Sparse） | 稠密检索（Dense） |
|------|-------------------|------------------|
| **向量表示** | 高维（词汇表大小），极度稀疏（>99% 零值） | 低维固定长度（128~1024），完全稠密（全部非零） |
| **匹配机制** | 精确词项匹配 + 统计加权（TF, IDF） | 语义空间中的向量距离/相似度 |
| **词汇鸿沟** | 严重——同义词/同义表达完全无法匹配 | 天然克服——语义相近的词编码后距离近 |
| **训练需求** | 无（BM25/TF-IDF 零样本可用）；SPLADE 需要训练 | 必须训练——没有预训练嵌入模型无法使用 |
| **可解释性** | ⭐⭐⭐⭐⭐——清楚知道哪些词匹配了、权重多少 | ⭐⭐——黑盒匹配，不知道"为什么"匹配 |
| **领域迁移** | ⭐⭐⭐⭐⭐——零样本跨领域，无需额外训练 | ⭐⭐——领域切换通常需要领域微调 |
| **精确匹配** | ⭐⭐⭐⭐⭐——"第123条" 精确到字符 | ⭐⭐——相近语义可能模糊精确边界 |
| **存储空间** | 极低——只需存储非零值（倒排索引） | 高——每个文档存储完整密集向量 |
| **检索速度** | 极快——倒排索引 + 仅处理命中词项的文档 | 取决于索引结构——HNSW/IVF 可加速但精度有损失 |
| **新词处理** | 好——词表中已有的新词可直接匹配 | 差——OOV（Out of Vocabulary）问题 |
| **更新成本** | 低——新增/修改文档只需更新倒排索引 | 中——需要重新编码文档为嵌入向量 |
| **语义理解** | 无——纯统计匹配，不理解含义 | 强——理解词项和句子的语义 |
| **多语言/跨语言** | 差——不同语言词项无法匹配 | 好——跨语言嵌入可将不同语言映射到同一空间 |
| **硬件要求** | 极低——纯 CPU 即可高效运行 | 中——ANN 索引需要较多内存，GPU 加速更优 |
| **场景适配** | 精确匹配、特定术语检索、低资源场景 | 语义检索、模糊查询、对话式搜索 |

## 核心优势

1. **零样本可用（Zero-shot）**：BM25 和 TF-IDF 完全不需要训练数据。新领域新语种的文档拿来就能建索引、就能检索——这是稠密检索无法比拟的，极大降低了冷启动成本。

2. **完全可解释（Interpretable）**：检索结果可以追溯到具体的匹配词项，清楚知道"为什么这个文档被检索到"。这在法律、医疗等对可解释性有严格要求的场景至关重要。

3. **高效低成本（Efficient）**：
   - 倒排索引检索时间复杂度 O(number of matching terms × posting list length)
   - 完全无需 GPU，单机即可处理亿级文档
   - 索引构建和更新极快

4. **精确关键词匹配（Exact Match）**：对于 ID 号、日期、金额、条款编号等精确信息，稠密检索的模糊匹配反而有害——只有稀疏检索能精确命中。

5. **鲁棒（Robust）**：不会出现"离群"检索结果。稠密检索可能在嵌入空间中出现意外的"近邻"，BM25 的匹配逻辑严格受限于命中的词项。

6. **与稠密检索天然互补**：混合检索（Hybrid Search）结合稀疏（精确匹配）和稠密（语义匹配）的优点，通常显著优于任何单一方法——这是当前 RAG 系统的最佳实践。

## 工程优化

### BM25 参数调优

**k1 参数**控制词频饱和度：

| k1 值 | 行为 | 适合场景 |
|-------|------|---------|
| 0.5 ~ 1.0 | 快速饱和，高频词影响迅速下降 | 短文本、查询词少 |
| 1.2 ~ 1.5 | 常规设定，平衡饱和速度 | 通用场景（**最常用范围**） |
| 1.5 ~ 2.0 | 慢速饱和，高频词持续贡献分数 | 长文档、查询词多 |
| 2.0+ | 几乎不饱和，近似原始 TF | 特殊场景（全词频匹配） |

**b 参数**控制文档长度归一化：

| b 值 | 行为 | 适合场景 |
|------|------|---------|
| 0.0 | 不进行长度归一化，长文档天然分数高 | 文档长度均匀的场景 |
| 0.5 ~ 0.75 | 中等长度惩罚 | 通用场景（**默认推荐 0.75**） |
| 0.75 ~ 1.0 | 强长度惩罚 | 文档长度差异大的场景 |
| 1.0 | 完全长度归一化 | 长度严重影响相关性时 |

**调优经验**（行业最佳实践，2024-2025 共识）：
- 通用场景：k1=1.5, b=0.75 是 Robertsons 原始推荐值，也是多数场景的默认起始值
- 短文本搜索（如标题搜索）：k1=1.2, b=0.5（减少长度惩罚，标题一般短且精炼）
- 长文档搜索（如论文全文）：k1=1.5~2.0, b=0.8~1.0（强长度惩罚，避免长文档分数虚高）
- 代码搜索：k1=2.0, b=0.3（代码段落长度相近，减少归一化以强调函数内匹配）
- 一些工业界应用将 k1 调至 0.9~1.2，b 调至 0.4~0.6 以适配短文本搜索；ES 默认 k1=1.2, b=0.75
- **最佳实践：在验证集上网格搜索 k1 ∈ [0.5, 3.0], b ∈ [0.0, 1.0]**

### 索引压缩优化

```python
# 倒排索引压缩策略
class CompressedInvertedIndex:
    """使用差值编码（delta encoding）和变长整数（varint）压缩"""
    
    def __init__(self):
        self.index = {}
        self.doc_freqs = {}
    
    def add_document(self, doc_id: int, terms: list[str]):
        """添加文档到索引"""
        for term in set(terms):
            if term not in self.index:
                self.index[term] = []
                self.doc_freqs[term] = 0
            
            # 使用差值编码存储 doc_id（需要预先排序）
            posting_list = self.index[term]
            if not posting_list:
                posting_list.append(doc_id)
            else:
                # 存储与前一个 doc_id 的差值
                prev = posting_list[-1]
                posting_list.append(doc_id - prev)
            
            self.doc_freqs[term] += 1
```

### 停用词处理

```python
STOP_WORDS = set([
    "的", "了", "在", "是", "我", "有", "和", "就", "不", "人",
    "都", "一", "一个", "上", "也", "很", "到", "说", "要", "去",
    "你", "会", "着", "没有", "看", "好", "自己", "这",
    "a", "an", "the", "is", "are", "was", "were", "in", "on", "at",
    "to", "for", "of", "and", "or", "it", "this", "that", "with"
])

def remove_stopwords(tokens: list[str]) -> list[str]:
    return [t for t in tokens if t.lower() not in STOP_WORDS]

# 最佳实践：不要简单移除停用词
# 在某些查询中，停用词可能携带意义（如 "to be or not to be"）
# 建议：仅对高 IDF 的停用词进行降权而非直接移除
```

### 词干化与词形还原

```python
# 英文词干化（Stemming）
from nltk.stem import PorterStemmer, SnowballStemmer

stemmer = PorterStemmer()
print(stemmer.stem("running"))    # → "run"
print(stemmer.stem("retrieval"))  # → "retriev"
print(stemmer.stem("documents"))  # → "document"

# 中文分词（中文检索必须先分词）
# pip install jieba
import jieba

text = "稀疏检索是RAG系统的重要组成部分"
tokens = list(jieba.cut(text))
print(tokens)  # → ["稀疏", "检索", "是", "RAG", "系统", "的", "重要", "组成部分"]
```

### Elasticsearch BM25 配置

```json
// Elasticsearch BM25 相似度配置
PUT /my_index
{
  "settings": {
    "index": {
      "similarity": {
        "custom_bm25": {
          "type": "BM25",
          "k1": 1.5,
          "b": 0.75
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "similarity": "custom_bm25"
      },
      "content": {
        "type": "text",
        "similarity": "custom_bm25"
      }
    }
  }
}

// BM25F 风格：不同字段不同权重
GET /my_index/_search
{
  "query": {
    "multi_match": {
      "query": "稀疏检索 BM25",
      "fields": {
        "title^3",      // 标题权重 3x
        "content^1"     // 正文权重 1x
      }
    }
  }
}
```

### 混合检索（Hybrid Search）架构

```python
# 稀疏 + 稠密混合检索
class HybridRetriever:
    def __init__(self, 
                 bm25_index: BM25,
                 dense_encoder,
                 alpha: float = 0.3):
        """
        alpha: 稀疏检索权重 (0 ~ 1)
        最终得分 = alpha * sparse_score + (1-alpha) * dense_score
        """
        self.bm25 = bm25_index
        self.dense_encoder = dense_encoder
        self.alpha = alpha
    
    def search(self, query: str, top_k: int = 10):
        # 稀疏检索
        sparse_results = self.bm25.search(query, top_k=top_k * 2)
        
        # 稠密检索
        query_emb = self.dense_encoder.encode(query)
        dense_results = self.dense_encoder.search(query_emb, top_k=top_k * 2)
        
        # 分数归一化（分别归一化到 [0, 1] 再加权融合）
        sparse_scores = self._normalize(sparse_results)
        dense_scores = self._normalize(dense_results)
        
        # RRF（Reciprocal Rank Fusion）融合
        # score = 1 / (k + rank_sparse) + 1 / (k + rank_dense)
        k = 60  # RRF 常数
        hybrid_scores = {}
        all_docs = set()
        
        for rank, (doc_id, _) in enumerate(sparse_results):
            all_docs.add(doc_id)
            hybrid_scores[doc_id] = hybrid_scores.get(doc_id, 0) + 1 / (k + rank + 1)
        
        for rank, (doc_id, _) in enumerate(dense_results):
            all_docs.add(doc_id)
            hybrid_scores[doc_id] = hybrid_scores.get(doc_id, 0) + 1 / (k + rank + 1)
        
        # 排序返回
        sorted_docs = sorted(hybrid_scores.items(), key=lambda x: -x[1])
        return sorted_docs[:top_k]
```

## 场景适配指南

| 场景 | 推荐方法 | k1 | b | 备注 |
|------|---------|----|---|------|
| **通用文本检索** | BM25 | 1.5 | 0.75 | 默认配置，多数场景的起点 |
| **短文本/标题搜索** | BM25 | 1.2 | 0.5 | 减少长度归一化 |
| **长文档/论文检索** | BM25 | 2.0 | 0.9 | 强长度惩罚避免长文档霸榜 |
| **法律/合同检索** | BM25 + 自定义分词器 | 1.5 | 0.75 | 保留法条编号、特殊字符 |
| **医疗文献检索** | BM25 + MeSH 词表扩展 | 1.5 | 0.75 | 结合领域词典扩展 |
| **代码库检索** | BM25 | 2.0 | 0.3 | 代码长度均匀，减少归一化 |
| **语义理解场景** | SPLADE | — | — | 需要语义扩展但保留关键词能力 |
| **高精度+高召回要求** | 混合检索 BM25 + Dense | 1.5 | 0.75 | alpha=0.3~0.5 需调优 |
| **低资源/无训练数据** | BM25 | 1.5 | 0.75 | 零样本且效果有保障 |
| **可解释性要求高** | BM25 | — | — | 匹配词项透明可追溯 |
| **实时检索/流式索引** | BM25 + 倒排索引 | 1.5 | 0.75 | 新增文档秒级生效 |
| **跨语言检索** | 稠密检索（非稀疏场景） | — | — | 稀疏不擅长跨语言 |
| **电子商务/产品搜索** | BM25 + 属性提权（BM25F） | 1.5 | 0.5 | 标题权重 3x，属性 2x，描述 1x |

### 通用选择流程

```
需要检索的文档 → 
    是否需要精确匹配（ID、编号、法条、日期）？
    ├── 是 → 必须使用稀疏检索（BM25/SPLADE）
    │      是否需要语义理解/同义扩展？
    │      ├── 是 → SPLADE 或 混合检索
    │      └── 否 → BM25
    │
    └── 否 → 是否有训练数据/嵌入模型？
            ├── 是 → 稠密检索（或混合检索）
            └── 否 → BM25（零样本最佳选择）

最终建议：生产 RAG 系统优先采用 混合检索（BM25 + Dense），
          alpha 在 0.3~0.5 之间调优，兼顾精确与语义。
```

### 总结

稀疏检索是 RAG 系统的基石。尽管稠密检索在语义理解上有本质优势，但 BM25 及其变体凭借**零样本可用、完全可解释、极致高效、精确匹配**等不可替代的特性，在 RAG 系统的检索管道中始终占有一席之地。2024-2025 年的最新趋势表明，**混合检索（Hybrid Search）**——稀疏检索与稠密检索的结合——是生产级 RAG 系统的最优实践，而学习型稀疏检索（如 SPLADE-v3）正成为连接稀疏与稠密两个世界的桥梁。
