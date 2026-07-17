# Adaptive RAG — 自适应检索策略

---

## 1. 背景与问题

### 1.1 检索策略的"一刀切"困境

传统 RAG 系统采用**固定检索策略**——无论用户查询的复杂度、类型和领域如何，都执行相同的检索流程：嵌入查询 → 向量搜索 top-k → LLM 生成。这种"一刀切"（One-size-fits-all）方法存在根本性问题：

- **简单查询的过度检索**：对于事实性查询（如"爱因斯坦的出生年份"），完整的检索-生成流程是计算资源的浪费。模型本身的参数化知识足以直接回答。
- **复杂查询的检索不足**：对于多步推理查询（如"分析 Transformer 架构对 NLP 领域的影响，并与 RNN 进行比较"），单次检索无法覆盖所有信息需求，导致生成内容片面。
- **策略-查询失配**：简单查询使用了复杂策略（延迟高、成本高），复杂查询使用了简单策略（质量差、幻觉多），两者都产生次优结果。

### 1.2 核心矛盾

Adaptive RAG 试图解决的核心矛盾是：**查询的异构性与检索策略的同构性之间的不匹配**。

不同查询对检索的需求存在本质差异：
- 有的查询"不需要检索"（LLM 参数化知识足够）
- 有的查询"需要单次精确检索"（事实查证）
- 有的查询"需要多步渐进检索"（复杂推理）

一个优秀的 RAG 系统应该像一名经验丰富的研究员一样：**先判断问题类型，再决定查阅策略**。

```
查询类型光谱:
├── 简单事实型: "Paris is the capital of which country?"
├── 单步查证型: "What is the latest version of Python?"
├── 比较分析型: "Compare AWS Lambda and Google Cloud Functions"
├── 多步推理型: "How will the adoption of LLMs impact 
│                the job market for software engineers?"
└── 开放式探索型: "What are the emerging trends in AI 
                  safety research?"
```

---

## 2. 核心思想

### 2.1 查询复杂度驱动的自适应

Adaptive RAG 的核心思想可以用一句话概括：**根据查询的复杂度，动态选择和组合最佳的检索策略**。

系统中的查询被划分为三个层级：

```
                 ┌─────────────────────────────────────┐
                 │              User Query              │
                 └────────────────┬────────────────────┘
                                  │
                                  ▼
                 ┌─────────────────────────────────────┐
                 │         Query Classifier             │
                 └──────┬──────────┬──────────┬────────┘
                        │          │          │
                   (A)  │    (B)   │    (C)   │
                 ┌──────▼───┐ ┌───▼────┐ ┌───▼──────────┐
                 │  Simple  │ │Moderate│ │   Complex    │
                 │          │ │        │ │              │
                 │ 直接生成  │ │单次检索 │ │ 多步检索 +   │
                 │ (无检索)  │ │ + 生成  │ │ 聚合 + 生成  │
                 └──────────┘ └────────┘ └──────────────┘
                        │          │          │
                        └──────────┼──────────┘
                                   ▼
                 ┌─────────────────────────────────────┐
                 │           Final Answer               │
                 └─────────────────────────────────────┘
```

### 2.2 三种基本策略

| 策略类型 | 策略名称 | 检索次数 | LLM 调用次数 | 适用查询类型 | 典型延迟 |
|----------|----------|----------|-------------|-------------|---------|
| A | DirectGenerate | 0 | 1 | 简单事实型 | 低 |
| B | SingleRetrieval | 1 | 1 | 单步查证型 | 中 |
| C | MultiStepRetrieval | N | N+1 | 复杂推理型 | 高 |

---

## 3. 核心论文

### 3.1 Adaptive RAG (Jeong et al., 2024)

Adaptive RAG 由 Jeong 等人在 2024 年提出，发表于 ACL 2024。论文的全称是 "Adaptive RAG: Adaptive Retrieval-Augmented Generation via Query Complexity Classification"。

**核心贡献**：

1. **查询复杂度分类器（Query Complexity Classifier）**：提出了一个将查询分类为 A/B/C 三类的分类器，可以根据查询的复杂度自动分配策略。
2. **策略路由框架**：设计了一个简洁但有效的策略路由框架，将分类结果映射到对应的检索-生成策略。
3. **多任务训练范式**：提出了一种联合训练分类器和各策略模块的方法，使分类器能够感知不同策略的适用边界。

### 3.2 论文的核心洞察

> "Different queries require different amounts of retrieval. One retrieval does not fit all." — Adaptive RAG 论文核心论点

论文通过大量实验证明：
- 约 30-40% 的查询实际上不需要检索（直接生成效果更好）
- 约 40-50% 的查询需要一次精确检索
- 约 10-20% 的查询需要多步迭代检索

### 3.3 关键实验结果

在多个基准测试上，Adaptive RAG 相比固定策略的 RAG 取得了显著提升：
- QA 准确率提升 5-12%
- 平均推理延迟降低 30-50%（因为大量简单查询跳过了检索）
- 检索成本降低 40-60%

---

## 4. 架构设计

### 4.1 完整系统架构

```
                           ┌─────────────────────────┐
                           │     User Query          │
                           └───────────┬─────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Frontend / API Gateway                       │
│  (Query preprocessing: normalization, cache lookup, guardrails)  │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Query Complexity Classifier                   │
│                                                                   │
│    ┌─────────────────────┐  ┌─────────────────────────────┐      │
│    │  Rule-based Filter  │  │    ML-based Classifier      │      │
│    │  (keyword, length)  │  │  (BERT / LLM / Ensemble)   │      │
│    └──────────┬──────────┘  └──────────────┬──────────────┘      │
│               │                             │                     │
│               └──────────┬──────────────────┘                    │
│                          ▼                                        │
│                   ┌──────────────┐                               │
│                   │  Decision:   │                               │
│                   │ A / B / C   │                               │
│                   └──────┬───────┘                               │
└──────────────────────────┼────────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────────┐
          ▼                ▼                    ▼
┌─────────────────┐ ┌──────────────┐ ┌──────────────────────────┐
│ Strategy A      │ │ Strategy B   │ │ Strategy C               │
│ DirectGenerate  │ │ SingleRetrie │ │ MultiStepRetrieval       │
│                 │ │              │ │                          │
│ LLM.generate(q) │ │ docs =       │ │ sub_queries = decompose(q)│
│                 │ │  retriever(q)│ │ for sq in sub_queries:   │
│                 │ │ LLM.         │ │   docs += retriever(sq)  │
│                 │ │  generate(   │ │ docs = merge(docs)       │
│                 │ │   context,q) │ │ LLM.generate(context,q)  │
└────────┬────────┘ └──────┬───────┘ └──────────────┬───────────┘
         │                 │                        │
         └─────────────────┼────────────────────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │   Final Answer   │
                  └──────────────────┘
```

### 4.2 策略选择逻辑

```
                    ┌─────────────┐
                    │  Query      │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ Cache Hit?  │──Yes──→ Return Cached Answer
                    └──────┬──────┘
                           │ No
                    ┌──────▼──────┐
                    │ Rule Filter │
                    │  (Fast Path)│
                    └──────┬──────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
      ┌──────────┐  ┌────────────┐  ┌──────────────┐
      │ Keyword  │  │ Length/    │  │ Intent/      │
      │ Match    │  │ Complexity │  │ Domain Detect│
      └─────┬────┘  └──────┬─────┘  └──────┬───────┘
            │              │               │
            └──────────────┼───────────────┘
                           ▼
                   ┌───────────────┐
                   │ ML Classifier │
                   │ (Confidence)  │
                   └───────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    Confidence      Confidence         Confidence
    > 0.9           0.6 - 0.9          < 0.6
    Direct Use      Use ML Decision   Fallback to
          │                │           Default (B)
          ▼                ▼                ▼
    ┌──────────┐    ┌────────────┐   ┌────────────┐
    │ Select   │    │ Select     │   │ Select     │
    │ Strategy │    │ Strategy   │   │ Strategy B │
    └──────────┘    └────────────┘   └────────────┘
```

---

## 5. 查询分类器设计

查询分类器是 Adaptive RAG 的"大脑"，直接影响整个系统的效果。以下是三种主流实现方案：

### 5.1 基于 LLM 的分类（高精度，高成本）

```python
class LLMQueryClassifier:
    """
    基于 LLM 的查询分类器
    使用 few-shot prompting 实现高精度分类
    """
    def __init__(self, llm_client):
        self.llm = llm_client
        self.few_shot_examples = [
            {
                "query": "What is the capital of France?",
                "type": "A",
                "reason": "Simple factual question, answer is fixed and well-known"
            },
            {
                "query": "What is the latest version of PyTorch?",
                "type": "B",
                "reason": "Needs up-to-date retrieval to verify current version"
            },
            {
                "query": "Compare and contrast the architectural differences "
                         "between GPT-4 and Claude 3, analyzing their impact "
                         "on reasoning capabilities",
                "type": "C",
                "reason": "Multi-aspect comparative analysis requiring "
                         "retrieval of multiple pieces of information"
            }
        ]
    
    def classify(self, query: str) -> str:
        prompt = self._build_prompt(query)
        response = self.llm.generate(prompt)
        return self._parse_type(response)
    
    def _build_prompt(self, query: str) -> str:
        examples_text = "\n\n".join([
            f"Query: {ex['query']}\nType: {ex['type']}"
            for ex in self.few_shot_examples
        ])
        return f"""Classify the following query into type A, B, or C:

Type A: Simple queries that can be answered directly without retrieval.
Type B: Moderate queries requiring a single retrieval step.
Type C: Complex queries requiring multiple retrieval steps.

Examples:
{examples_text}

Query: {query}
Type:"""
```

**优势**：精度最高，可以利用 LLM 的语义理解和世界知识
**劣势**：成本高、延迟高（每次分类需要一次 LLM 调用）

### 5.2 基于轻量模型的分类（高效率，可接受精度）

```python
class BERTQueryClassifier:
    """
    基于 BERT 的轻量级查询分类器
    在特定领域数据集上微调后，精度可与 LLM-based 方案媲美
    """
    def __init__(self, model_path: str = "bert-base-uncased"):
        self.tokenizer = self._load_tokenizer(model_path)
        self.model = self._load_model(model_path)
        self.labels = ['A', 'B', 'C']
    
    def classify(self, query: str) -> str:
        inputs = self.tokenizer(
            query,
            return_tensors="pt",
            truncation=True,
            max_length=128,
            padding=True
        )
        outputs = self.model(**inputs)
        probabilities = torch.softmax(outputs.logits, dim=-1)
        predicted_idx = torch.argmax(probabilities, dim=-1).item()
        
        return self.labels[predicted_idx]
    
    def classify_with_confidence(self, query: str) -> tuple:
        """返回分类结果和置信度"""
        inputs = self.tokenizer(query, return_tensors="pt", 
                               truncation=True, max_length=128)
        outputs = self.model(**inputs)
        probabilities = torch.softmax(outputs.logits, dim=-1)
        confidence, predicted_idx = torch.max(probabilities, dim=-1)
        
        return self.labels[predicted_idx.item()], confidence.item()
```

**优势**：推理延迟低（<10ms）、成本低、可本地部署
**劣势**：需要标注数据微调、对领域外查询泛化能力有限

### 5.3 基于规则的分类（零成本，低精度）

```python
class RuleBasedQueryClassifier:
    """
    基于规则的查询分类器
    利用查询的语言特征进行快速分类
    """
    
    # 类型 C 关键词：需要多步检索的复杂查询
    COMPLEX_INDICATORS = [
        "compare", "contrast", "difference between", "similarities",
        "analyze", "evaluate", "discuss the impact", "relationship between",
        "how does", "why does", "explain the process",
        "compare and contrast", "pros and cons",
        "advantages and disadvantages",
        # 中文指标
        "比较", "对比", "区别", "分析", "影响",
        "关系", "过程", "为什么", "如何影响",
        "优缺点", "利弊"
    ]
    
    # 类型 A 关键词：不需要检索的简单查询
    SIMPLE_INDICATORS = [
        "what is", "who is", "when did", "define",
        "capital of", "population of",
        # 中文指标
        "什么是", "是谁", "是什么", "定义",
        "的首都是", "的人口是"
    ]
    
    def classify(self, query: str) -> str:
        query_lower = query.lower()
        
        # Step 1: 检查是否包含复杂指标
        if any(indicator in query_lower 
               for indicator in self.COMPLEX_INDICATORS):
            # 进一步检查长度：长查询更可能是复杂的
            if len(query.split()) > 15:
                return "C"
            return "B"  # 中等长度但有比较词
        
        # Step 2: 检查是否包含简单指标
        if any(indicator in query_lower 
               for indicator in self.SIMPLE_INDICATORS):
            if len(query.split()) < 10:
                return "A"
        
        # Step 3: 基于长度的默认策略
        word_count = len(query.split())
        if word_count < 8:
            return "A"
        elif word_count < 20:
            return "B"
        else:
            return "C"
```

**优势**：零推理成本、完全可解释、易于调试
**劣势**：精度有限、无法处理复杂语义、维护成本随规则增加

### 5.4 混合分类器（生产推荐方案）

```python
class HybridQueryClassifier:
    """
    混合查询分类器：规则快速筛选 + ML 精确分类
    结合了规则的速度和 ML 的精度
    """
    def __init__(self, ml_classifier, rule_classifier=None):
        self.ml_classifier = ml_classifier
        self.rule_classifier = rule_classifier or RuleBasedQueryClassifier()
    
    def classify(self, query: str) -> tuple:
        """
        返回 (strategy_type, confidence, source)
        """
        # Step 1: 规则分类器快速判断（O(1) 时间）
        rule_result = self.rule_classifier.classify(query)
        
        # Step 2: 如果是明显简单的查询，直接返回
        if rule_result == "A" and self._is_high_confidence_rule(query):
            return ("A", 0.95, "rule")
        
        # Step 3: 其他情况使用 ML 分类器
        ml_result, confidence = self.ml_classifier.classify_with_confidence(
            query
        )
        
        # Step 4: 当 ML 置信度低时，回退到规则结果
        if confidence < 0.5 and rule_result is not None:
            return (rule_result, 0.6, "rule_fallback")
        
        return (ml_result, confidence, "ml")
    
    def _is_high_confidence_rule(self, query: str) -> bool:
        """判断规则分类结果是否高置信"""
        query_lower = query.lower()
        simple_triggers = [
            "what is", "who is", "when is", "define",
            "capital", "population"
        ]
        return any(query_lower.startswith(t) for t in simple_triggers)
```

---

## 6. 策略库设计

### 6.1 策略接口

```python
from abc import ABC, abstractmethod
from typing import List, Dict, Any, Optional

class RetrievalStrategy(ABC):
    """检索策略抽象基类"""
    
    @abstractmethod
    def execute(self, query: str) -> Dict[str, Any]:
        """
        执行检索策略并返回结果
        返回: {
            'answer': str,
            'context': List[Document],
            'metadata': Dict
        }
        """
        pass
    
    @abstractmethod
    def name(self) -> str:
        """策略名称"""
        pass

class DirectGenerateStrategy(RetrievalStrategy):
    """策略 A：直接生成（无检索）"""
    
    def __init__(self, llm):
        self.llm = llm
    
    def name(self) -> str:
        return "direct_generate"
    
    def execute(self, query: str) -> Dict[str, Any]:
        start_time = time.time()
        
        answer = self.llm.generate(query)
        
        return {
            "answer": answer,
            "context": [],
            "metadata": {
                "strategy": self.name(),
                "retrieval_count": 0,
                "latency_ms": (time.time() - start_time) * 1000
            }
        }

class SingleRetrievalStrategy(RetrievalStrategy):
    """策略 B：单次检索 + 生成"""
    
    def __init__(self, retriever, llm, top_k: int = 5):
        self.retriever = retriever
        self.llm = llm
        self.top_k = top_k
    
    def name(self) -> str:
        return "single_retrieval"
    
    def execute(self, query: str) -> Dict[str, Any]:
        start_time = time.time()
        
        # 单次检索
        docs = self.retriever.retrieve(query, top_k=self.top_k)
        
        # 构建增强上下文
        context = self._format_context(docs)
        
        # 生成
        prompt = f"""Context:\n{context}\n\nQuestion: {query}\nAnswer:"""
        answer = self.llm.generate(prompt)
        
        return {
            "answer": answer,
            "context": docs,
            "metadata": {
                "strategy": self.name(),
                "retrieval_count": 1,
                "total_docs": len(docs),
                "latency_ms": (time.time() - start_time) * 1000
            }
        }
    
    def _format_context(self, docs) -> str:
        return "\n\n".join([d.content for d in docs])

class MultiStepRetrievalStrategy(RetrievalStrategy):
    """策略 C：多步检索 + 聚合 + 生成"""
    
    def __init__(self, retriever, llm, decomposer=None, merger=None):
        self.retriever = retriever
        self.llm = llm
        self.decomposer = decomposer or self._default_decomposer
        self.merger = merger or self._default_merger
    
    def name(self) -> str:
        return "multi_step_retrieval"
    
    def execute(self, query: str) -> Dict[str, Any]:
        start_time = time.time()
        retrieval_count = 0
        all_docs = []
        
        # Step 1: 查询分解
        sub_queries = self.decomposer(query)
        
        # Step 2: 对每个子查询执行检索
        for sq in sub_queries:
            docs = self.retriever.retrieve(sq, top_k=3)
            all_docs.extend(docs)
            retrieval_count += 1
        
        # Step 3: 结果合并与去重
        merged_docs = self.merger(all_docs)
        
        # Step 4: 基于合并结果生成
        context = self._format_context(merged_docs)
        prompt = f"""You are analyzing multiple pieces of information 
to answer a complex question.

Context:\n{context}\n\nQuestion: {query}

Provide a comprehensive answer that synthesizes all relevant information:"""
        answer = self.llm.generate(prompt)
        
        # Step 5: 可选的二次检索（对生成中的缺失信息进行补充）
        if self._needs_supplement(answer, query):
            supplement_query = self._generate_supplement_query(answer, query)
            extra_docs = self.retriever.retrieve(supplement_query, top_k=2)
            all_docs.extend(extra_docs)
            retrieval_count += 1
            # 重新生成
            context = self._format_context(merged_docs + extra_docs)
            answer = self.llm.generate(
                f"Context:\n{context}\n\nQuestion: {query}\nComprehensive Answer:"
            )
        
        return {
            "answer": answer,
            "context": all_docs,
            "metadata": {
                "strategy": self.name(),
                "retrieval_count": retrieval_count,
                "sub_queries": sub_queries,
                "total_docs": len(all_docs),
                "latency_ms": (time.time() - start_time) * 1000
            }
        }
    
    def _default_decomposer(self, query: str) -> List[str]:
        """将复杂查询分解为多个子查询"""
        prompt = f"""Decompose this complex query into simpler sub-queries:
Query: {query}
Sub-queries (one per line):"""
        result = self.llm.generate(prompt)
        return [q.strip() for q in result.split('\n') if q.strip()]
    
    def _default_merger(self, docs) -> List:
        """合并并去重"""
        seen = set()
        merged = []
        for doc in docs:
            if doc.content not in seen:
                seen.add(doc.content)
                merged.append(doc)
        return merged
    
    def _needs_supplement(self, answer: str, query: str) -> bool:
        """判断是否需要补充检索"""
        # 如果答案过短或包含不确定性表述，触发补充
        uncertainty_markers = [
            "I'm not sure", "I don't have", "insufficient",
            "based on the limited", "might", "could be",
            "不确定", "没有足够", "可能"
        ]
        return any(m in answer for m in uncertainty_markers)
    
    def _generate_supplement_query(self, answer: str, query: str) -> str:
        """从答案中提取缺失信息生成补充查询"""
        prompt = f"""Original question: {query}
Current answer: {answer}
What additional information is needed? Generate a search query:"""
        return self.llm.generate(prompt)
```

### 6.2 策略路由核心

```python
class AdaptiveRAG:
    """
    Adaptive RAG 主类
    根据查询复杂度动态路由到不同策略
    """
    def __init__(
        self,
        classifier: QueryClassifier,
        strategies: Dict[str, RetrievalStrategy],
        fallback_strategy: str = "moderate",
        monitor=None
    ):
        self.classifier = classifier
        self.strategies = strategies
        self.fallback_strategy = fallback_strategy
        self.monitor = monitor or NoopMonitor()
    
    def answer(self, query: str) -> Dict[str, Any]:
        """
        自适应 RAG 主流程：
        1. 分类查询类型
        2. 选择对应策略
        3. 执行策略
        4. 可选的反馈循环
        """
        # Step 1: 查询分类
        query_type, confidence, source = self.classifier.classify(query)
        
        # Step 2: 策略选择
        strategy = self._select_strategy(query_type, confidence)
        
        # Step 3: 策略执行
        result = strategy.execute(query)
        
        # Step 4: 记录监控指标
        self.monitor.record({
            "query": query,
            "classified_as": query_type,
            "confidence": confidence,
            "strategy_used": strategy.name(),
            "latency_ms": result["metadata"]["latency_ms"],
            "retrieval_count": result["metadata"]["retrieval_count"]
        })
        
        return result
    
    def _select_strategy(
        self, query_type: str, confidence: float
    ) -> RetrievalStrategy:
        """根据分类结果和置信度选择策略"""
        
        type_to_key = {
            "A": "simple",
            "B": "moderate",
            "C": "complex"
        }
        
        strategy_key = type_to_key.get(query_type, self.fallback_strategy)
        
        # 低置信度时使用保守策略（默认走 B）
        if confidence < 0.5:
            strategy_key = self.fallback_strategy
        
        return self.strategies[strategy_key]
```

---

## 7. 动态切换与反馈机制

### 7.1 运行时策略切换

```python
class FeedbackAwareAdaptiveRAG(AdaptiveRAG):
    """
    带运行时反馈的自适应 RAG
    根据效果反馈动态调整策略选择
    """
    
    def __init__(self, *args, adaptation_window: int = 100, **kwargs):
        super().__init__(*args, **kwargs)
        self.adaptation_window = adaptation_window
        self.feedback_buffer = []
        self.strategy_distribution = {"A": 0, "B": 0, "C": 0}
    
    def answer_with_feedback(self, query: str) -> Dict[str, Any]:
        result = self.answer(query)
        
        # 隐式反馈收集（基于答案质量指标）
        feedback = self._compute_implicit_feedback(result)
        self.feedback_buffer.append(feedback)
        
        # 滑动窗口内重新评估策略分布
        if len(self.feedback_buffer) >= self.adaptation_window:
            self._rebalance_strategies()
        
        return result
    
    def _compute_implicit_feedback(self, result: Dict) -> Dict:
        """计算隐式反馈分数"""
        metadata = result["metadata"]
        
        # 基于答案质量的启发式反馈
        answer = result["answer"]
        quality_score = 1.0
        
        # 惩罚过短答案
        if len(answer) < 20:
            quality_score *= 0.5
        
        # 惩罚不确定性表述
        uncertainty_markers = ["I'm not sure", "I don't know", "不确定", "不知道"]
        if any(m in answer for m in uncertainty_markers):
            quality_score *= 0.7
        
        # 惩罚高延迟
        if metadata["latency_ms"] > 10000:  # 超过10秒
            quality_score *= 0.8
        
        return {
            "strategy": metadata["strategy"],
            "quality": quality_score,
            "latency": metadata["latency_ms"]
        }
    
    def _rebalance_strategies(self):
        """重新平衡策略权重"""
        if not self.feedback_buffer:
            return
        
        # 计算各策略的平均质量
        strategy_quality = {"A": [], "B": [], "C": []}
        for fb in self.feedback_buffer:
            strategy = fb["strategy"]
            strategy_quality[strategy].append(fb["quality"])
        
        avg_quality = {
            s: sum(qs) / len(qs) if qs else 0
            for s, qs in strategy_quality.items()
        }
        
        # 如果某个策略持续低质量，调整分类器的决策边界
        for strategy, quality in avg_quality.items():
            if quality < 0.3 and self.strategy_distribution.get(strategy, 0) > 0:
                # 该策略效果持续不佳，减少使用频率
                self.classifier.adjust_bias(strategy, -0.1)
        
        self.feedback_buffer = []
```

---

## 8. 与 Agentic RAG 的关系

### 8.1 概念图谱

```
RAG 系统的自动化程度光谱:

固定策略 RAG ←────────── Adaptive RAG ──────────→ Agentic RAG
     │                        │                        │
 单一策略                  预定义策略集             完全自主决策
 无路由                   基于分类路由             基于推理路由
 无自适应                 有边界自适应             无边界自适应
 无工具调用               有限工具使用             任意工具使用
```

### 8.2 核心区别

| 维度 | Adaptive RAG | Agentic RAG |
|------|-------------|-------------|
| **决策机制** | 分类器驱动的策略路由 | LLM/Agent 自主推理决策 |
| **策略集** | 固定/预定义（2-4种） | 动态构建，不限定 |
| **工具使用** | 检索器 + 有限工具 | 任意工具（API、DB、计算等） |
| **状态管理** | 无状态或轻量状态 | 有状态（对话历史、记忆） |
| **可解释性** | 高（策略路由路径清晰） | 中（Agent 推理过程可能不透明） |
| **控制粒度** | 粗粒度（策略级） | 细粒度（步骤级） |
| **实现复杂度** | 低-中 | 高 |
| **鲁棒性** | 高（行为边界清晰） | 中（Agent 可能偏离预期） |

### 8.3 Adaptive RAG 作为 Agentic RAG 的特例

从更广阔的视角看，Adaptive RAG 可以视为 Agentic RAG 的一个**特例**——当 Agent 的决策空间被约束为"少数几个预定义策略的有限集合"时，Agentic RAG 退化为 Adaptive RAG。

```
Agentic RAG:
  Agent 看到查询 → 推理需要什么信息 → 决定调用哪些工具 →
  执行工具 → 观察结果 → 决定下一步 → ... → 生成答案

Adaptive RAG:
  分类器 看到查询 → 判断复杂度 → 映射到策略 →
  执行策略 → 生成答案
                      ↑
               Agentic RAG 的"有限决策空间"版本
```

这意味着，当你在 Adaptive RAG 中增加更多策略类别、引入更复杂的路由逻辑时，你实际上是在向 Agentic RAG 方向演进。

---

## 9. 工程挑战

### 9.1 分类准确率

查询分类器是整个系统的"命门"——分类错误会导致策略-查询失配：

| 实际类型 | 被分为 | 后果 |
|----------|--------|------|
| A (简单) | C (复杂) | 不必要的检索增加延迟和成本 |
| B (中等) | A (简单) | 需要检索时未检索，导致幻觉 |
| C (复杂) | B (中等) | 检索不足，答案片面 |
| A (简单) | B (中等) | 微小延迟增加，影响不大 |

**缓解方案**：
- 在低置信度时回退到保守策略（默认 B）
- 使用 Ensemble 方法集成多个分类器
- 持续收集误分类样本进行针对性优化

### 9.2 策略边界模糊

A/B/C 三类策略之间的边界并非天然存在，而是人为划分的。许多查询处于边界区域，分类存在固有歧义。

**缓解方案**：
- 引入 Soft Assignment：不硬分到某个类别，而是分配概率向量 [p_A, p_B, p_C]
- 使用概率加权混合策略（混合检索结果）
- 动态调整策略边界（根据生产数据反馈）

### 9.3 冷启动问题

新部署的 Adaptive RAG 系统面临冷启动困境：
- 分类器没有足够的领域数据
- 策略参数没有经过调优
- 阈值设定缺乏依据

**缓解方案**：
- 初始阶段使用纯策略 B（单次检索）采集基线数据
- 使用基于规则的分类器作为初始方案，逐步切换为 ML 方案
- 在线学习：随着数据积累动态更新分类器

### 9.4 策略执行的资源开销

多步检索策略（C）的资源消耗可能是单步检索（B）的 3-5 倍。需要精确控制复杂策略的使用比例。

```python
class BudgetAwareAdaptiveRAG(AdaptiveRAG):
    """预算感知的自适应 RAG"""
    
    def __init__(self, *args, 
                 budget_config: Dict = None, 
                 **kwargs):
        super().__init__(*args, **kwargs)
        self.budget_config = budget_config or {
            "max_complex_per_hour": 100,
            "max_retrieval_per_query": 10,
            "cost_tracking_window": 3600
        }
        self.usage_counter = {
            "simple": 0,
            "moderate": 0,
            "complex": 0
        }
    
    def _select_strategy(self, query_type, confidence):
        """预算约束下的策略选择"""
        # 检查预算
        if query_type == "C":
            if self.usage_counter["complex"] >= \
               self.budget_config["max_complex_per_hour"]:
                # 预算不足，降级为 B
                query_type = "B"
        
        strategy = super()._select_strategy(query_type, confidence)
        self.usage_counter[strategy.name()] += 1
        return strategy
```

---

## 10. 2025-2026 前沿进展

### 10.1 SPARKLE: 结构化策略框架

SPARKLE（Structured Policy for Adaptive Retrieval and Knowledge Learning）提出了更精细的策略分解方式：

- 将策略拆解为原子操作（检索、重写、过滤、聚合、生成）
- 使用 Planner 动态组合原子操作
- 引入了操作之间的依赖关系和执行顺序约束

```
传统 Adaptive RAG:  [分类] → [固定策略 A/B/C]

SPARKLE:  [理解] → [规划原子操作序列] → [执行序列] → [评估效果]
                ↑                              │
                └──────────────────────────────┘
                     (迭代优化执行计划)
```

### 10.2 QuDAR: 查询感知的自适应检索

QuDAR（Query-aware Dynamic Adaptive Retrieval）在查询分类的基础上引入了**感知维度**：

1. **领域感知**：不同领域的查询需要不同的检索源
2. **时间感知**：时效性要求高的查询优先检索最新文档
3. **用户感知**：不同用户的语言习惯和知识背景影响检索策略
4. **任务感知**：不同下游任务（QA、摘要、对话）需要不同的检索粒度

### 10.3 INKER: 内外知识融合

INKER（Internal and External Knowledge Fusion with Adaptive Retrieval）聚焦于 Adaptive RAG 中的一个关键挑战——如何融合内部知识（LLM 参数化知识）和外部知识（检索结果）：

- **策略 A（简单）**：优先使用内部知识
- **策略 B（中等）**：平衡使用内外部知识
- **策略 C（复杂）**：优先使用外部知识，内部知识作为补充

INKER 提出了知识冲突检测机制，当内外部知识冲突时，根据查询类型和知识源的可靠性动态决定信任哪种知识来源。

### 10.4 其他重要进展

- **Adaptive RAG + ReAct**：将 Adaptive RAG 的策略路由与 ReAct 的推理-行动循环结合，在策略执行过程中支持动态调整
- **Multi-level Adaptive RAG**：从查询复杂度、领域、用户意图等多个层面进行分层自适应
- **Cost-aware Adaptive RAG**：引入成本预算约束，在效果和成本之间进行 Pareto 优化
- **Online Adaptive RAG**：支持策略的在线更新，根据实时反馈不断优化分类器和策略参数

---

## 11. 适用场景总结

| 场景 | 推荐度 | 说明 |
|------|--------|------|
| 查询类型多样化 | ★★★★★ | Adaptive RAG 的核心优势 |
| 对延迟敏感 | ★★★★★ | 简单查询跳过检索大幅降低平均延迟 |
| 对成本敏感 | ★★★★★ | 减少不必要的检索和 LLM 调用 |
| 知识库覆盖全面 | ★★★★☆ | 各类查询都能找到对应信息 |
| 需要在线学习 | ★★★★☆ | 反馈机制支持持续优化 |
| 查询类型单一 | ★☆☆☆☆ | 自适应引入额外复杂度但无收益 |
| 对精度要求极高 | ★★★☆☆ | 分类错误可能导致策略失配 |
| 小型系统 | ★★★☆☆ | 自适应框架的工程复杂度较高 |

最佳实践建议：
1. **初始阶段**：使用基于规则的分类器 + 默认回退到策略 B
2. **增长阶段**：切换为混合分类器，收集生产数据训练 ML 模型
3. **成熟阶段**：引入反馈机制和预算控制，实现完全自适应
4. **进阶阶段**：向 Agentic RAG 演进，增加更多策略选项和工具

---

## 参考资源

1. Jeong, S., et al. (2024). "Adaptive RAG: Adaptive Retrieval-Augmented Generation via Query Complexity Classification." ACL 2024.
2. SPARKLE. "Structured Policy for Adaptive Retrieval and Knowledge Learning." 2025.
3. QuDAR. "Query-aware Dynamic Adaptive Retrieval." 2025.
4. INKER. "Internal and External Knowledge Fusion with Adaptive Retrieval." 2025.
5. Gao, Y., et al. (2024). "Retrieval-Augmented Generation for Large Language Models: A Survey."
