# Corrective RAG (CRAG) — 检索结果自动修正

---

## 1. 背景与问题

### 1.1 传统 RAG 的隐含假设

传统检索增强生成（Retrieval-Augmented Generation, RAG）范式建立在一个关键假设之上：**检索到的文档总是与查询相关且有用**。在本体架构中，检索管道（Retrieval Pipeline）的输出被无条件地送入生成模块，中间没有任何质量验证环节。这一假设在实际生产环境中面临严峻挑战：

- **检索噪声（Retrieval Noise）**：向量数据库的近似最近邻搜索（ANN）在低维或高维空间中可能返回语义相似但事实无关的文档。例如，查询"2024年诺贝尔物理学奖得主"可能被错误匹配到"2024年诺贝尔文学奖"相关文档，仅因为两者在嵌入空间中距离较近。
- **检索失败（Retrieval Failure）**：当知识库中缺乏目标信息时，检索器会返回与查询弱相关甚至完全无关的文档。在缺乏相关性过滤的情况下，这些无关文档会被直接送入生成器。
- **幻觉放大（Hallucination Amplification）**：当检索到的噪声文档与 LLM 的参数化知识发生冲突时，LLM 倾向于优先采纳检索到的（错误的）外部信息，导致比纯生成更严重的幻觉现象。
- **黑盒问题**：传统 RAG 将检索结果直接送入 LLM，缺乏对检索质量的**可见性和可控性**。

### 1.2 核心矛盾

RAG 系统的核心矛盾在于：**检索器的优化目标（语义相似度）与生成器的需求（事实准确性）之间存在根本性偏差**。检索器返回 top-k 个"看起来最相似"的文档，但这些文档可能：
1. 在关键事实上与查询矛盾
2. 包含大量无关细节而缺乏关键信息
3. 多个文档之间互相矛盾
4. 全部来源于单一有偏见的信源

Corrective RAG（CRAG）的提出正是为了解决这一矛盾——**在检索之后、生成之前引入一个校正环节**，对检索结果进行显式质量评估和策略性修正。

---

## 2. 核心论文

### 2.1 Corrective RAG (CRAG, Yan et al., 2024)

CRAG 由 Yan 等人在 2024 年提出，论文发表于 arXiv（标题："Corrective Retrieval Augmented Generation"），是 RAG 纠错方向的奠基性工作。其核心贡献包括：

1. **检索相关性评估器（Relevance Evaluator）**：引入一个专门的评估模型，对检索文档与查询之间的相关性进行细粒度评分，将检索结果分为"正确"、"不正确"和"模糊"三个类别。
2. **分解-重组机制（Decompose-then-Recompose）**：针对模糊查询，提出将查询分解为多个子查询分别检索，再将结果重组，以提升对多意图查询的覆盖率。
3. **网络搜索回退（Web Search Fallback）**：当检索完全失败时，自动回退到网络搜索引擎，确保系统在知识库覆盖不足时仍能提供有用答案。

### 2.2 论文的核心洞察

CRAG 的关键洞察在于：**与其优化检索器的召回率，不如在检索后增加一个轻量级的质量验证环节**。这一思路借鉴了软件工程中的"防御性编程"理念——不假设上游组件的输出总是正确的，而是在接口处增加校验。

> "Retrieval is not always helpful; sometimes it is harmful. We need a mechanism to detect and correct retrieval failures before generation." — CRAG 论文核心论点

---

## 3. 核心架构

### 3.1 整体流程

```
                    ┌─────────────────────────────────────┐
                    │              User Query              │
                    └────────────────┬────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │            Retriever                 │
                    │    (Vector Search / BM25 / Hybrid)   │
                    └────────────────┬────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │        Relevance Evaluator           │
                    │   ┌─────────────────────────────┐    │
                    │   │  Score = Eval(query, docs)  │    │
                    │   └─────────────────────────────┘    │
                    └────────────┬──────────┬──────────┬───┘
                                 │          │          │
                  ┌──────────────┘    ┌─────┘    ┌─────┘
                  ▼                   ▼          ▼
           Score > TH_HIGH    Score < TH_LOW    TH_LOW ≤ Score ≤ TH_HIGH
                  │                   │                   │
                  ▼                   ▼                   ▼
     ┌─────────────────────┐  ┌──────────────┐  ┌─────────────────────────┐
     │   Direct Use (A)    │  │ Web Search   │  │  Decompose & Re-query   │
     │   docs as-is        │  │ Fallback (B) │  │  Split → Re-eval (C)   │
     └──────────┬──────────┘  └──────┬───────┘  └─────────────┬───────────┘
                │                    │                         │
                └────────────────────┼─────────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │           LLM Generator              │
                    │   (Analyze + Synthesize + Generate)  │
                    └────────────────┬────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │            Final Answer              │
                    └─────────────────────────────────────┘
```

### 3.2 三大分支详解

#### 分支 A：正确 → 直接使用（Correct Path）

当评估器判定检索文档与查询高度相关（Score > HIGH_THRESHOLD）时，系统信任检索结果，将其直接送入生成器。此路径的延迟最低，与传统 RAG 一致。

**适用场景**：知识库覆盖查询主题、嵌入质量高、查询意图明确。

#### 分支 B：不正确 → Web 搜索回退（Incorrect Path）

当评估器判定检索文档与查询基本无关（Score < LOW_THRESHOLD）时，系统认为知识库无法提供有效信息，自动切换到网络搜索（Web Search）。搜索结果经过清洗后送入生成器。

**适用场景**：知识库过时、查询涉及热点事件、知识库本身未覆盖该领域。

#### 分支 C：模糊 → 分解与重查（Ambiguous Path）

当评估器无法明确判定相关性（LOW_THRESHOLD ≤ Score ≤ HIGH_THRESHOLD）时，系统认为查询可能包含多个子意图，采用**分解-重组（Decompose-then-Recompose）**策略：

1. **分解（Decompose）**：将原始查询拆解为多个更聚焦的子查询
2. **重查（Re-query）**：对每个子查询执行检索+评估（递归调用 CRAG 流程）
3. **重组（Recompose）**：将所有子查询结果合并、去重、排序后送入生成器

**适用场景**：复合查询（如"比较 Transformer 和 Mamba 的优缺点"）、模糊查询（如"如何优化 AI 系统的性能"）。

---

## 4. 评估机制详解

### 4.1 Relevance Evaluator 设计

相关性评估器是 CRAG 的核心组件，其设计直接影响整个系统的质量。主要有三种实现路线：

#### 路线一：T5-based Evaluator（论文原始方法）

使用 T5 模型进行相关性评分，将其建模为一个序列分类任务：

```
Input:  "query: {query} document: {doc_text}"
Output: "relevant" / "irrelevant"
```

T5-based 方法的优势在于效率——T5-small 可以在 CPU 上实现亚毫秒级推理，适合高吞吐场景。但缺点是对语义理解的深度有限。

#### 路线二：LLM-based Evaluator

使用 GPT-4/Claude 等强 LLM 进行评分，可获得更精确的评估：

```
System: You are a relevance evaluator. Assess whether the retrieved
        document is relevant to answering the user's query.
User:   Query: {query}
        Document: {doc_text}
        Output only a score from 0.0 to 1.0:
```

LLM-based 方法精度更高，但延迟和成本也显著增加。通常用于低吞吐但在意质量的关键场景。

#### 路线三：Embedding-based Evaluator

直接计算查询嵌入与文档嵌入之间的余弦相似度作为相关性分数。这是最轻量的方案，但精度最差。

### 4.2 阈值设定策略

阈值设定是 CRAG 工程实践中的关键挑战：

| 策略 | 方法 | 优点 | 缺点 |
|------|------|------|------|
| 固定阈值 | 手动设定 HIGH=0.8, LOW=0.3 | 简单可控 | 缺乏适应性 |
| 动态阈值 | 基于历史分布计算分位数 | 自动适应数据分布 | 需要冷启动数据 |
| 自适应阈值 | 根据查询类型、领域调整 | 精细化控制 | 实现复杂度高 |
| 无阈值 | 使用 Softmax 概率直接路由 | 端到端可微 | 需要训练数据 |

**工程建议**：生产环境中推荐使用**动态阈值 + 人工干预通道**的组合策略。初始阶段使用固定阈值上线，收集足够数据后切换为基于百分位数的动态阈值。

### 4.3 Decompose-then-Recompose 策略

分解策略直接影响模糊查询的处理质量：

```
模糊查询: "Explain the differences and similarities between 
           supervised learning and reinforcement learning"

分解策略 A (按主题拆分):
  ├── Sub-query 1: "What is supervised learning?"  
  ├── Sub-query 2: "What is reinforcement learning?"
  └── Sub-query 3: "Compare supervised learning vs reinforcement learning"

分解策略 B (按方面拆分):  
  ├── Sub-query 1: "Definition and core concepts"
  ├── Sub-query 2: "Training process and feedback mechanism"
  ├── Sub-query 3: "Typical applications and use cases"

重组方法:
  └── 按子查询相关性加权合并 → 去除冗余 → 按逻辑顺序排列
```

**关键设计决策**：子查询分解应该保持**正交性**（子查询之间语义重叠最小）和**完整性**（子查询覆盖原始查询的所有方面）。

---

## 5. 实现关键

### 5.1 主逻辑伪代码

```python
import numpy as np
from typing import List, Dict, Any, Optional
from dataclasses import dataclass
from enum import Enum

class RelevanceLevel(Enum):
    CORRECT = "correct"
    AMBIGUOUS = "ambiguous"
    INCORRECT = "incorrect"

@dataclass
class Document:
    content: str
    metadata: Dict[str, Any]
    relevance_score: Optional[float] = None

class RelevanceEvaluator:
    """相关性评估器基类"""
    def evaluate(self, query: str, doc: Document) -> float:
        raise NotImplementedError

class T5RelevanceEvaluator(RelevanceEvaluator):
    """基于 T5 的轻量级评估器"""
    def __init__(self, model_name: str = "google/flan-t5-small"):
        # 实际加载模型
        self.model = self._load_model(model_name)
    
    def evaluate(self, query: str, doc: Document) -> float:
        input_text = f"query: {query} document: {doc.content[:512]}"
        # 模型推理，输出相关性概率
        score = self.model.predict(input_text)
        return score

class LLMRelevanceEvaluator(RelevanceEvaluator):
    """基于 LLM 的高精度评估器"""
    def __init__(self, llm_client):
        self.llm = llm_client
    
    def evaluate(self, query: str, doc: Document) -> float:
        prompt = f"""Assess relevance (0.0-1.0):
Query: {query}
Document: {doc.content[:1000]}
Score:"""
        response = self.llm.generate(prompt)
        return self._parse_score(response)

class CorrectiveRAG:
    """
    Corrective RAG 主类
    实现对检索结果的三路分支校正
    """
    def __init__(
        self,
        retriever,
        evaluator: RelevanceEvaluator,
        high_threshold: float = 0.8,
        low_threshold: float = 0.3,
        web_search_func=None,
        decomposer=None,
        merger=None,
        max_recursion_depth: int = 3
    ):
        self.retriever = retriever
        self.evaluator = evaluator
        self.high_threshold = high_threshold
        self.low_threshold = low_threshold
        self.web_search = web_search_func
        self.decomposer = decomposer or self._default_decomposer
        self.merger = merger or self._default_merger
        self.max_recursion_depth = max_recursion_depth
    
    def _classify_relevance(
        self, query: str, docs: List[Document]
    ) -> RelevanceLevel:
        """对检索结果集合进行整体相关性分类"""
        if not docs:
            return RelevanceLevel.INCORRECT
        
        # 计算文档集的最大相关性分数
        max_score = max(
            self.evaluator.evaluate(query, doc) for doc in docs
        )
        # 计算加权平均（位置权重衰减）
        weights = [0.5 ** i for i in range(len(docs))]
        weighted_score = sum(
            self.evaluator.evaluate(query, doc) * w
            for doc, w in zip(docs, weights)
        ) / sum(weights)
        
        effective_score = 0.7 * max_score + 0.3 * weighted_score
        
        if effective_score > self.high_threshold:
            return RelevanceLevel.CORRECT
        elif effective_score < self.low_threshold:
            return RelevanceLevel.INCORRECT
        else:
            return RelevanceLevel.AMBIGUOUS
    
    def _default_decomposer(self, query: str) -> List[str]:
        """默认的查询分解器：使用 LLM 分解"""
        prompt = f"""Decompose the following query into 2-4 simpler 
sub-queries. Each sub-query should be self-contained and answerable 
independently. Return as a numbered list.

Query: {query}

Sub-queries:"""
        # 调用 LLM 生成子查询
        sub_queries = self._llm_generate(prompt)
        return self._parse_list(sub_queries)
    
    def _default_merger(self, results: List[List[Document]]) -> List[Document]:
        """默认的结果合并器：去重并排序"""
        seen_contents = set()
        merged = []
        for doc_list in results:
            for doc in doc_list:
                if doc.content not in seen_contents:
                    seen_contents.add(doc.content)
                    merged.append(doc)
        return merged
    
    def retrieve_and_correct(
        self, query: str, depth: int = 0
    ) -> List[Document]:
        """CRAG 核心方法：检索并校正"""
        if depth > self.max_recursion_depth:
            # 防止无限递归
            return self.retriever.retrieve(query)
        
        # Step 1: 初始检索
        docs = self.retriever.retrieve(query, top_k=5)
        
        # Step 2: 相关性评估
        level = self._classify_relevance(query, docs)
        
        # Step 3: 根据评估结果执行三路分支
        if level == RelevanceLevel.CORRECT:
            # 分支 A：直接使用
            return docs
        
        elif level == RelevanceLevel.INCORRECT:
            # 分支 B：Web 搜索回退
            if self.web_search:
                web_results = self.web_search(query, num_results=3)
                return [Document(
                    content=r["snippet"],
                    metadata={"source": "web", "url": r["url"]}
                ) for r in web_results]
            else:
                # 无 Web 搜索可用时返回空
                return []
        
        else:  # AMBIGUOUS
            # 分支 C：分解与重查
            sub_queries = self.decomposer(query)
            sub_results = []
            for sq in sub_queries:
                # 递归调用 CRAG
                sub_docs = self.retrieve_and_correct(sq, depth + 1)
                sub_results.append(sub_docs)
            
            return self.merger(sub_results)
    
    def generate(self, query: str) -> str:
        """完整的检索-校正-生成流程"""
        corrected_docs = self.retrieve_and_correct(query)
        
        if not corrected_docs:
            return "无法获取相关信息，请稍后重试。"
        
        # 构建增强提示
        context = "\n\n".join([
            f"[{i+1}] {doc.content}" 
            for i, doc in enumerate(corrected_docs)
        ])
        
        prompt = f"""Based on the following retrieved documents, answer 
the user's question accurately. If the documents don't contain enough 
information, state that clearly.

Documents:
{context}

Question: {query}

Answer:"""
        
        return self._llm_generate(prompt)
```

### 5.2 降级策略的平滑切换

在实际系统中，三大分支之间的切换应该设计为**平滑降级**而非硬切换：

```python
class SoftCorrectiveRAG(CorrectiveRAG):
    """带平滑切换的 CRAG"""
    
    def retrieve_and_correct(self, query: str) -> List[Document]:
        docs = self.retriever.retrieve(query, top_k=10)
        
        # 对每个文档单独评估
        doc_scores = [
            self.evaluator.evaluate(query, doc) for doc in docs
        ]
        
        # 按分数分层处理
        high_conf_docs = [
            doc for doc, score in zip(docs, doc_scores)
            if score > self.high_threshold
        ]
        low_conf_docs = [
            doc for doc, score in zip(docs, doc_scores)
            if self.low_threshold <= score <= self.high_threshold
        ]
        
        corrected_docs = []
        
        # 高置信度文档直接使用
        corrected_docs.extend(high_conf_docs)
        
        # 对低置信度文档进行 Web 增强
        if low_conf_docs and self.web_search:
            web_results = self.web_search(query)
            corrected_docs.extend(web_results)
        
        # 如果仍然不足，对模糊文档进行分解重查
        if len(corrected_docs) < 3 and len(high_conf_docs) < 2:
            sub_queries = self.decomposer(query)
            for sq in sub_queries:
                sub_docs = self.retrieve_and_correct(sq)
                corrected_docs.extend(sub_docs)
        
        return self._deduplicate(corrected_docs)
```

---

## 6. 与 Self-RAG 的区别

CRAG 和 Self-RAG 虽然都旨在提升 RAG 的鲁棒性，但设计哲学和切入角度存在根本性差异：

| 维度 | CRAG (Corrective RAG) | Self-RAG |
|------|----------------------|----------|
| **核心思路** | 检索结果的**事后修正** | 生成过程的**自我反思** |
| **干预时机** | 检索之后、生成之前 | 生成过程中 |
| **评估对象** | 检索文档的相关性 | 生成的每个片段的正确性 |
| **控制粒度** | 文档级别 | Token/片段级别 |
| **是否需要额外模型** | 需要相关性评估器 | 需要 Reflection Token |
| **主要开销** | 评估器推理 + 可能的 Web 搜索 | 多次 LLM 推理（每次生成后反思） |
| **适用场景** | 检索质量波动大的场景 | 对生成质量要求极高的场景 |
| **协作可能** | 可以与 Self-RAG 叠加使用 | 可以与 CRAG 叠加使用 |

**关键区别总结**：
- CRAG 回答的是**"检索结果靠不靠谱"**的问题
- Self-RAG 回答的是**"生成内容靠不靠谱"**的问题
- 两者可以互补：先用 CRAG 确保输入质量，再用 Self-RAG 确保输出质量

---

## 7. 工程优化

### 7.1 评估器延迟优化

评估器是 CRAG 额外引入的最大延迟来源。优化策略包括：

```python
class OptimizedEvaluator:
    """带缓存和批处理的评估器优化"""
    
    def __init__(self, evaluator, cache_size: int = 1000):
        self.evaluator = evaluator
        self.cache = LRUCache(maxsize=cache_size)
    
    def evaluate(self, query: str, doc: Document) -> float:
        # Step 1: 缓存命中检查
        cache_key = hash(query + doc.content[:200])
        if cache_key in self.cache:
            return self.cache[cache_key]
        
        # Step 2: 快速预筛选（基于嵌入相似度）
        emb_score = self._embedding_similarity(query, doc)
        if emb_score < 0.1:  # 明显不相关，直接返回低分
            self.cache[cache_key] = emb_score
            return emb_score
        
        # Step 3: 只有通过预筛选的文档才走完整评估
        full_score = self.evaluator.evaluate(query, doc)
        self.cache[cache_key] = full_score
        return full_score
    
    def batch_evaluate(self, query: str, docs: List[Document]) -> List[float]:
        """批处理评估：对一批文档统一评估"""
        # 使用向量化操作减少模型调用
        return [self.evaluate(query, doc) for doc in docs]
    
    def _embedding_similarity(self, query: str, doc: Document) -> float:
        """快速嵌入相似度计算（用于预筛选）"""
        # 使用本地轻量嵌入模型计算
        query_emb = self._embed(query)
        doc_emb = self._embed(doc.content)
        return cosine_similarity(query_emb, doc_emb)
```

### 7.2 缓存策略

多级缓存设计对 CRAG 性能至关重要：

```
                  ┌──────────────────────┐
                  │    L1: Query Cache    │  (内存, TTL=5min)
                  │   query → docs_list   │
                  └──────────┬───────────┘
                             │
                  ┌──────────▼───────────┐
                  │  L2: Eval Cache      │  (内存, TTL=30min)
                  │ (query,doc_hash)→score│
                  └──────────┬───────────┘
                             │
                  ┌──────────▼───────────┐
                  │  L3: Web Result Cache│  (Redis, TTL=1h)
                  │   query → web_results │
                  └──────────────────────┘
```

### 7.3 阈值自适应

在长期运行中，固定阈值无法适应数据分布的变化。自适应策略可以提升系统稳定性：

```python
class AdaptiveThresholdManager:
    """
    基于历史数据的阈值自适应管理器
    滑动窗口 + 分位数自适应
    """
    def __init__(self, window_size: int = 1000):
        self.scores_window = []
        self.window_size = window_size
        self.high_percentile = 80  # 高分阈值对应第80百分位
        self.low_percentile = 30   # 低分阈值对应第30百分位
    
    def update(self, score: float, was_useful: bool = None):
        """更新分数窗口"""
        self.scores_window.append(score)
        if len(self.scores_window) > self.window_size:
            self.scores_window.pop(0)
    
    def get_thresholds(self) -> tuple:
        """获取当前阈值"""
        if len(self.scores_window) < 100:
            # 冷启动阶段返回默认值
            return (0.8, 0.3)
        
        scores = sorted(self.scores_window)
        high_idx = int(len(scores) * self.high_percentile / 100)
        low_idx = int(len(scores) * self.low_percentile / 100)
        
        return (scores[high_idx], scores[low_idx])
```

---

## 8. 能力边界

### 8.1 评估器本身可能误判

这是一个元问题（Meta-Problem）：**用来判断检索是否正确的评估器本身也可能出错**。评估器的错误类型包括：

- **假阳性（False Positive）**：评估器认为文档相关，实际不相关 → 引入噪声导致幻觉
- **假阴性（False Negative）**：评估器认为不相关，实际相关 → 错误触发 Web Fallback 增加延迟
- **校准误差（Calibration Error）**：评估器的置信度分数与实际准确率不匹配

缓解方案：
1. 使用多个评估器投票（Ensemble）
2. 引入评估器的不确定性度量，低置信度时走保守路径
3. 对评估器进行持续监控和定期重训练

### 8.2 Web Fallback 增加延迟

Web 搜索的延迟通常在 200ms-2s 之间，远高于向量检索（10-50ms）。在错误触发的场景下，延迟惩罚尤为明显。

缓解方案：
1. 异步预取：在检索的同时异步发起 Web 搜索，评估器快速返回时取消 Web 请求
2. 超时控制：为 Web Fallback 设置严格超时，超时后降级为"无检索"模式
3. 分级 Fallback：先尝试本地次级知识库，再尝试 Web 搜索

### 8.3 系统复杂度增加

CRAG 引入了新的组件和状态，系统整体复杂度上升：
- 评估器的维护成本（数据标注、模型更新）
- 阈值调优的人力成本
- 三路分支的测试覆盖成本

---

## 9. 2025-2026 前沿进展

### 9.1 Higress-RAG: 双混合检索 + CRAG

Higress-RAG 将 CRAG 的理念与混合检索（稀疏检索 + 稠密检索）深度结合，引入了**双向校正机制**：

```
                    ┌──────────────────┐
                    │     Query        │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
      ┌────────────┐ ┌────────────┐ ┌────────────┐
      │ BM25 (稀疏) │ │ Dense (稠密)│ │ 语义检索   │
      └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
            │              │              │
            └──────────────┼──────────────┘
                           ▼
                  ┌──────────────────┐
                  │  CRAG 校正器     │
                  │  (双通道评估)    │
                  └────────┬─────────┘
                           │
                  ┌────────▼─────────┐
                  │     Generator    │
                  └──────────────────┘
```

关键创新：**双通道评估**——分别评估稀疏检索结果和稠密检索结果的质量，在不同通道间进行交叉验证。当两个通道的评估结果一致时置信度更高；当不一致时触发进一步的分解和修正。

### 9.2 RAG-Critic: 评审机制

RAG-Critic 将 CRAG 的校正思想从"检索后"扩展到了**全流程**，引入了一个独立的评审器（Critic）对 RAG 系统的每个环节进行评审：

1. **Query Critic**：评审查询是否需要改写或扩展
2. **Retrieval Critic**：评审检索结果的质量（类似 CRAG）
3. **Generation Critic**：评审生成内容的准确性和完整性
4. **Answer Critic**：评审最终答案是否满足用户需求

RAG-Critic 可以被视为 CRAG 的"完全体"——将校正从单一环节扩展为贯穿全流程的评审体系。

### 9.3 其他重要进展

- **Iterative CRAG**：引入迭代校正机制，在每次生成后使用检索结果来修正生成，再用修正后的生成来指导下一轮检索
- **CRAG + Active Learning**：使用主动学习策略，对评估器不确定的样本进行人工标注，持续提升评估器质量
- **Multi-modal CRAG**：将 CRAG 扩展到多模态场景，对图像、表格、代码等多种形式的检索结果进行质量评估

---

## 10. 适用场景总结

| 场景 | 推荐度 | 说明 |
|------|--------|------|
| 知识库质量不稳定 | ★★★★★ | CRAG 的核心价值所在 |
| 查询多样性高 | ★★★★☆ | 分解-重组机制覆盖多意图查询 |
| 需要 Web 实时信息 | ★★★★★ | Fallback 到 Web Search |
| 低延迟要求 | ★★★☆☆ | 评估器和 Web Fallback 增加延迟 |
| 离线/内网环境 | ★★★☆☆ | Web Fallback 不可用，只剩评估+分解 |
| 高频简单查询 | ★★★☆☆ | 大多数查询不需要校正，评估器是额外开销 |
| 对小模型友好 | ★★★★☆ | 评估器使用小模型，不需要 LLM 在生成前自检 |

---

## 参考资源

1. Yan, S., et al. (2024). "Corrective Retrieval Augmented Generation." arXiv:2401.15884.
2. Higress-RAG 文档. "Dual Hybrid Retrieval with CRAG."
3. RAG-Critic. "Critique-based RAG Quality Improvement." 2025.
4. Self-RAG vs CRAG 对比分析. "When to Correct vs When to Reflect." 2025.
