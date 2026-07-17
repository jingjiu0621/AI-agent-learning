# 9.6.3 query-decomposition — 子问题拆分

## 简单介绍

子问题拆分（Query Decomposition）是指将**复杂查询拆分为多个更简单的子查询**的技术。其核心理念来自"分而治之"——一个需要多步推理或多源知识的复杂问题，直接检索往往会失败，但如果将其拆分为 LLM 可以分别回答的子问题，再组合结果，效果会大幅提升。

## 基本原理

### 需要拆分的查询特征

```
未拆分的查询:
"Transformer 相比 RNN 的优势是什么，以及它是如何解决长距离依赖问题的？"

拆分为:
子问题1: "Transformer 相比 RNN 的主要优势有哪些？"
子问题2: "Transformer 如何解决长距离依赖问题？"
```

需要拆分的查询通常具有以下特征：

| 特征 | 示例 | 说明 |
|------|------|------|
| **多实体** | "比较 Python、Java 和 Go 的并发模型" | 涉及 3 个不同的实体 |
| **多关系** | "ChatGPT 的技术基础和它的社会影响" | 技术和影响是两个不同的关系维度 |
| **因果链** | "为什么全球变暖会导致极端天气增多？" | 需要理解因果关系链 |
| **条件依赖** | "如果美联储加息，对美股和 A 股有什么不同影响？" | 条件和不同市场的对比 |

## 背景：为什么需要查询分解

向量检索的基本限制：**retriever 的"语义空间"无法同时精确匹配多个不同维度的信息**。

当一个查询同时涉及"A 的优势"和"B 的原理"时，embedding 模型会在两者之间"平均"，导致两个维度都检索不精确。

## 拆分的三种模式

### 1. 顺序分解（Sequential Decomposition）

子问题之间有依赖关系，需要按顺序回答：

```python
def sequential_decompose(query: str) -> list[str]:
    """分解为有顺序依赖的子问题"""
    
    prompt = f"""Decompose this question into sequential sub-questions.
    Each sub-question depends on the previous one.
    
    Example:
    Question: "What was the impact of the 2008 financial crisis on tech startups?"
    Sub-questions:
    1. "What caused the 2008 financial crisis?"
    2. "How did the 2008 financial crisis affect the investment environment?"
    3. "Which tech startups were founded or failed during 2008-2010?"
    4. "What long-term changes did the crisis bring to the tech startup ecosystem?"
    
    Question: {query}
    Sub-questions:"""
    
    return llm_generate_list(prompt)
```

### 2. 并行分解（Parallel Decomposition）

子问题之间独立，可以并行检索：

```python
def parallel_decompose(query: str) -> list[str]:
    """分解为可并行检索的独立子问题"""
    
    prompt = f"""Decompose this question into independent sub-questions 
    that can be answered in parallel.
    
    Question: "Compare the features, pricing, and performance of AWS Lambda and Google Cloud Functions"
    
    Independent sub-questions:
    1. "What are the features of AWS Lambda?"
    2. "What are the features of Google Cloud Functions?"
    3. "How does AWS Lambda pricing work?"
    4. "How does Google Cloud Functions pricing work?"
    5. "What is the performance benchmark between AWS Lambda and Google Cloud Functions?"
    
    Question: {query}
    Sub-questions:"""
    
    return llm_generate_list(prompt)
```

### 3. 层级分解（Hierarchical Decomposition）

将查询拆分为树状结构，从抽象到具体：

```python
def hierarchical_decompose(query: str, depth: int = 0, max_depth: int = 2):
    """递归层级分解查询"""
    if depth >= max_depth:
        return [query]
    
    prompt = f"""Decompose this question into 2-3 more specific sub-questions.
    
    Question: {query}
    Sub-questions:"""
    
    sub_questions = llm_generate_list(prompt)
    
    result = []
    for sq in sub_questions:
        result.extend(hierarchical_decompose(sq, depth + 1, max_depth))
    
    return result if result else [query]
```

## 分解算法的 Python 实现

```python
class QueryDecomposer:
    def __init__(self, use_llm: bool = True):
        self.use_llm = use_llm
    
    def decompose(self, query: str) -> list[SubQuery]:
        """主分解入口"""
        # Step 1: 判断是否需要分解
        complexity = self._assess_complexity(query)
        if complexity == 'simple':
            return [SubQuery(query, is_original=True)]
        
        # Step 2: 选择分解策略
        strategy = self._choose_strategy(query)
        
        # Step 3: 执行分解
        sub_questions = self._execute_decomposition(query, strategy)
        
        # Step 4: 标注子问题的依赖关系
        return self._annotate_dependencies(sub_questions)
    
    def _assess_complexity(self, query: str) -> str:
        """评估查询复杂度"""
        # 简单启发式评估
        question_count = len(re.findall(r'[？?]', query))
        entity_count = len(extract_entities(query))
        conjunction_count = len(re.findall(r'以及|并且|和|与|还有|此外', query))
        
        if question_count >= 2 or entity_count >= 3 or conjunction_count >= 2:
            return 'complex'
        elif entity_count >= 2:
            return 'medium'
        return 'simple'
    
    def _execute_decomposition(self, query: str, strategy: str) -> list[SubQuery]:
        if self.use_llm:
            return self._llm_decompose(query, strategy)
        else:
            return self._rule_decompose(query, strategy)
    
    def _rule_decompose(self, query: str, strategy: str) -> list[SubQuery]:
        """基于规则的分解（不依赖 LLM，适合低延迟场景）"""
        # 按连接词拆分
        conjunctions = ['以及', '并且', '而且', '和', '与', '还有', '此外', '另外']
        
        parts = [query]
        for conj in conjunctions:
            new_parts = []
            for part in parts:
                if conj in part:
                    new_parts.extend(part.split(conj))
                else:
                    new_parts.append(part)
            parts = new_parts
        
        return [SubQuery(p.strip(), is_original=False) for p in parts if p.strip()]

class SubQuery:
    def __init__(self, text: str, is_original: bool = False):
        self.text = text
        self.is_original = is_original
        self.dependencies: list[int] = []  # 依赖的子查询索引
        self.status: str = 'pending'
        self.result: str = None
```

## 分解策略对比

| 策略 | 适用查询 | 可并行度 | 延迟 |
|------|---------|---------|------|
| 不分解 | 单跳简单查询 | - | 低 |
| 并行分解 | 多实体独立关系 | 高 | 中 |
| 顺序分解 | 因果链/时间线 | 低 | 高 |
| 层级分解 | 复杂多层次问题 | 中 | 最高 |

## 最大挑战

1. **分解粒度的控制**：拆得太细 → 过多子问题（token 开销大）和失去整体上下文；拆得太粗 → 仍太复杂
2. **子问题依赖管理**：顺序分解时，后一个问题的答案依赖前一个的检索结果
3. **信息冗余**：不同子问题检索到相同的内容，造成 token 浪费
4. **聚合困难**：将多个子问题的结果聚合为连贯回答需要额外的处理

## 能力边界

- ✅ 处理包含 2-4 个明确子问题的复合查询
- ✅ 对"比较"类查询效果最好（因为天然有明确的分界）
- ✅ 对"因果链"类查询有效（逐步推理）
- ❌ 隐含多个子问题但表达上是一个完整句子的查询（"BERT 的成功对 NLP 领域有什么深远影响？"——可能不需要拆分）
- ❌ 子问题之间有复杂的相互依赖时难以管理
- ❌ 查询本身很短但需要复杂推理时（"为什么天是蓝的？"——单问题多步推理，不适合拆分）

## 工程优化

1. **自适应分解决策**：先用快速分类器判断是否需要分解，避免对简单问题过度工程
2. **缓存子问题结果**：相同子问题的检索结果可以在多个查询间复用
3. **并行检索执行**：独立的子问题使用多路并发检索
4. **聚合模板**：预定义常见的聚合模式（比较、因果、列举）

## 推荐论文

- **Least-to-Most Prompting (2022)**: 从简到繁逐步推理
- **Query Decomposition for RAG (2024)**: 探索-利用平衡
- **UniRAG (2024)**: 统一的分解-推理-改写框架
