# 9.6.1 query-classification — 查询分类

## 简单介绍

查询分类（Query Classification）是将用户输入按照**任务类型、知识领域、复杂度、检索策略**等维度进行分类，以便后续选择最合适的处理路径。它是查询理解流水线的第一步——分类决定了后续所有的处理策略。

## 基本原理

### 分类维度体系

一个完整的 RAG 查询分类系统通常包含多个分类维度：

```python
class QueryClassifier:
    """多维查询分类器"""
    
    def classify(self, query: str, history: list[dict] = None) -> QueryType:
        return QueryType(
            task=self._classify_task(query),           # 任务类型
            complexity=self._classify_complexity(query), # 复杂度
            domain=self._classify_domain(query),        # 领域
            retrieval=self._select_retrieval_strategy(query), # 检索策略
            temporal=self._detect_temporal_aspect(query)  # 时间性
        )
```

### 分类维度的详细说明

#### 维度 1：任务类型（Task Type）

| 任务类型 | 描述 | 示例 | 检索策略 |
|---------|------|------|---------|
| **factual** | 事实性查询 | "法国首都是什么？" | 单次检索 + 精确匹配 |
| **explanatory** | 解释性查询 | "解释一下量子纠缠" | 检索综述/教程类文档 |
| **instructional** | 指令性/如何做 | "如何用 Python 读取 CSV？" | 检索代码示例 + 教程 |
| **comparative** | 比较性 | "PostgreSQL vs MySQL 区别" | 双路检索 + 对比总结 |
| **analytical** | 分析性 | "自动驾驶的技术挑战" | 多源检索 + 综合分析 |
| **creative** | 创造性 | "写一首关于 AI 的诗" | 可不检索（依赖 LLM 自身） |
| **summarization** | 摘要性 | "总结这篇论文" | 检索全文 + 摘要生成 |

```python
def _classify_task(self, query: str) -> str:
    """使用轻量分类器判断查询的任务类型"""
    # 方法1：LLM 分类（高精度，但延迟高）
    if USE_LLM_CLASSIFIER:
        return self._llm_classify(query, TASK_TYPES)
    
    # 方法2：规则分类（快速，适合已知模式）
    patterns = {
        'factual': [r'什么是', r'是谁', r'位于', r'多少'],
        'instructional': [r'如何', r'怎么', r'怎样', r'步骤'],
        'comparative': [r'区别', r'对比', r'vs', r'相比'],
        'summarization': [r'总结', r'概括', r'摘要'],
    }
    for task_type, regexes in patterns.items():
        if any(re.search(p, query) for p in regexes):
            return task_type
    
    # 方法3：小模型分类（平衡精度与延迟）
    return self._fasttext_classify(query, model=TASK_CLASSIFIER)
```

#### 维度 2：复杂度（Complexity Level）

| 级别 | 描述 | 示例 | 处理方式 |
|------|------|------|---------|
| **simple** | 单跳、单一实体 | "Python 的 list 怎么用？" | 单次检索 |
| **medium** | 多实体、单一关系 | "Python 和 Java 语法区别" | 双路检索 + 比较 |
| **complex** | 多跳推理 | "2024 年诺贝尔物理学奖得主的研究方向是什么？" | 查询分解 + 多步检索 |
| **compound** | 复合多问题 | "什么是 Transformer？它和 RNN 比有什么优势？为什么现在主流用它？" | 分解为子问题 |

```python
def _classify_complexity(self, query: str) -> str:
    """评估查询的复杂度"""
    # 使用简单的启发式规则
    num_entities = len(extract_entities(query))
    has_multi_hop = bool(re.search(r'为什么|如何|怎样|影响|导致', query))
    num_questions = query.count('？') + query.count('?')
    
    if num_questions > 2:
        return 'compound'
    elif num_entities > 3 and has_multi_hop:
        return 'complex'
    elif num_entities > 1:
        return 'medium'
    else:
        return 'simple'
```

#### 维度 3：检索策略选择

```python
def _select_retrieval_strategy(self, query: str) -> RetrievalStrategy:
    """根据查询特征选择最优检索策略"""
    task = self._classify_task(query)
    
    strategy_map = {
        'factual': RetrievalStrategy(
            retriever='hybrid',        # 混合检索（精确 + 语义）
            rerank=True,               # 需要重排序
            top_k=5,                   # 少量结果
            min_relevance=0.7,         # 高相关性阈值
        ),
        'instructional': RetrievalStrategy(
            retriever='semantic',
            rerank=False,
            top_k=10,
            min_relevance=0.5,
        ),
        'analytical': RetrievalStrategy(
            retriever='diverse',       # 多样性检索（多角度）
            rerank=True,
            top_k=15,
            min_relevance=0.4,
        ),
    }
    
    return strategy_map.get(task, DEFAULT_STRATEGY)
```

## 背景：为什么需要查询分类

早期 RAG 系统对所有查询使用相同的检索策略——相同的 top_k、相同的检索器、相同的重排序。但实践发现：

- 事实性查询需要**高精度**、**少量**结果
- 分析性查询需要**多来源**、**多样性**结果
- 创造性查询可能**不需要检索**

对所有查询使用"一刀切"的检索策略是在同时**浪费性能和损害质量**。

## 分类方法对比

| 方法 | 精度 | 延迟 | 可解释性 | 维护成本 |
|------|------|------|---------|---------|
| 规则/正则 | 中 | 极低(~1ms) | 高 | 高(规则需持续维护) |
| 传统 ML (fastText/朴素贝叶斯) | 中-高 | 低(~10ms) | 中 | 中(需要标注数据) |
| 小模型 BERT (distilbert) | 高 | 中(~50ms) | 低 | 低(fine-tune 一次) |
| LLM (GPT-4o mini) | 最高 | 高(~200ms) | 高 | 极低(零样本/少样本) |

## 最大挑战

1. **分类边界模糊**：一个查询可能同时包含事实性和分析性元素（"Transformer 的原理是什么？它的优势在哪？"）
2. **冷启动**：新领域系统没有历史查询数据来训练分类器
3. **分类错误的传播**：分类错误会导致后续所有策略选择错误
4. **延迟叠加**：在 LLM 已经需要处理用户查询的情况下，额外加一次 LLM 分类调用增加延迟

## 能力边界

- ✅ 准确分类 80-90% 的常见查询类型
- ✅ 分类延迟可控制在 50ms 以内（非 LLM 方法）
- ✅ 支持自定义分类维度扩展
- ❌ 无法 100% 准确（总有边界 case）
- ❌ 跨语言分类比单语言困难
- ❌ 隐式意图（用户没明确问但实际想知道）难以检测

## 工程优化

1. **级联分类器**：先快速规则分类 → 置信度低时回退到 LLM 分类
2. **缓存分类结果**：对相同或相似查询缓存分类结果
3. **在线学习**：用户反馈（点赞/点踩）作为弱监督信号更新分类器
4. **多标签分类**：允许一个查询属于多个类别，选择融合策略

## 推荐工具

- **fastText**: 轻量级文本分类
- **SetFit**: 少样本场景的 sentence embedding 分类
- **langfuse / mlflow**: 分类结果追踪和实验
- **Custom regex engine**: 规则简单时可自建
