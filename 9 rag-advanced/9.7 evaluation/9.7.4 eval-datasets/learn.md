# 9.7.4 eval-datasets — 评估数据集

## 简单介绍

RAG 评估数据集是用于**系统化评估 RAG 系统质量**的标注数据集合。一个标准的 RAG 评估数据点包含：用户查询（Query）、检索参考（Relevant Documents / Gold Passages）和理想回答（Ground Truth Answer）。评估数据的质量直接决定了评估结果的可信度。

## 评估数据集的结构

```python
@dataclass
class RAGEvalSample:
    query: str                           # 用户查询
    query_type: str                      # 查询类型（factual/analytical/...）
    domain: str                          # 领域（tech/medical/legal/...）
    gold_passages: list[str]             # 理想检索结果（文档ID或文本）
    ground_truth_answer: str             # 理想回答
    metadata: dict                       # 额外信息（难度、来源等）

@dataclass
class RAGEvalDataset:
    name: str                            # 数据集名称
    samples: list[RAGEvalSample]         # 样本列表
    language: str                        # 语言
    size: int                            # 大小
    domain: str                          # 领域
    avg_query_length: float              # 平均查询长度
```

## 主流 RAG 评估数据集

### 1. 通用领域

| 数据集 | 规模 | 查询类型 | 特点 | 适用场景 |
|-------|------|---------|------|---------|
| **Natural Questions** | 307K | 事实性 | Google 搜索真实查询，Wikipedia 文档 | 通用事实检索 |
| **TriviaQA** | 95K | 事实性 | 复杂事实性问题，多文档推理 | 多文档检索 |
| **HotpotQA** | 113K | 多跳推理 | 需要多步推理才能回答 | 多跳检索评估 |
| **2WikiMultihop** | 192K | 多跳推理 | 来自 Wikipedia 的合成多跳问题 | 多跳 RAG |
| **BEIR** | 各组 5-100K | 混合 | 18 个检索基准的集合 | 通用检索评估 |

### 2. 特定领域

| 数据集 | 规模 | 领域 | 特点 |
|-------|------|------|------|
| **PubMedQA** | 1K | 生物医学 | 基于 PubMed 摘要的问答 |
| **BioASQ** | 4K | 生物医学 | 生物医学语义检索 |
| **LegalBench** | 10K+ | 法律 | 法律检索与推理 |
| **FinanceBench** | 10K | 金融 | 金融报告问答 |
| **CodeRAG-Bench** | 2K+ | 编程 | 代码检索与代码生成 |

### 3. RAG 专门评估数据集

```python
# RGB (RAG Benchmark) - 专门为 RAG 系统设计的评估
class RGBDataset:
    """
    评估维度:
    - 抗噪声: 检索结果中包含大量噪声时系统的表现
    - 否定: 当查询含否定时（"不包含 X 的实现"）
    - 多维度: 同时检索多个维度的信息
    - 抽象: 需要概括而非提取
    - 时间敏感性: 关注时间相关信息的处理
    """
    pass

# RECALL (Retrieval-Augmented Generation Benchmark)
class RECALLDataset:
    """
    评估维度:
    - 忠实度: 回答是否忠于检索文档
    - 完整性: 回答是否完整
    - 基座模型差异: 不同 LLM 在 RAG 下的表现差异
    """
    pass
```

## 自定义评估数据集的构建

### 1. 基于现有文档自动生成

```python
def generate_eval_dataset_from_docs(documents: list[dict], 
                                     n_samples: int = 100) -> list[RAGEvalSample]:
    """基于文档集合自动生成评估数据集"""
    
    samples = []
    
    for doc in tqdm(documents[:n_samples * 3]):  # 需要更多候选
        # 使用 LLM 基于文档生成问题和答案
        prompt = f"""Based on this document, generate a question-answer pair 
        that tests whether a RAG system can correctly retrieve and answer.
        
        Document: {doc['text'][:1000]}
        
        The question should require information EXCLUSIVELY from this document.
        
        Output JSON:
        {{
            "question": "...",
            "answer": "...",
            "difficulty": "easy/medium/hard",
            "required_passage": "the exact passage needed"
        }}"""
        
        result = llm_generate_json(prompt)
        
        samples.append(RAGEvalSample(
            query=result['question'],
            gold_passages=[result['required_passage']],
            ground_truth_answer=result['answer'],
            query_type='factual',
            domain=doc.get('domain', 'general'),
            metadata={'difficulty': result['difficulty'], 'source': doc['id']}
        ))
        
        if len(samples) >= n_samples:
            break
    
    return samples
```

### 2. 人工标注评估集

```python
def create_human_annotated_dataset(
    queries: list[str],
    document_store: DocumentStore,
    annotators: list[Annotator]
) -> RAGEvalDataset:
    """人工标注评估数据集"""
    
    samples = []
    
    for query in queries:
        # 标注者确定理想检索结果
        relevant_docs = annotators[0].find_relevant_docs(query, document_store)
        
        # 交叉验证
        for annotator in annotators[1:]:
            docs2 = annotator.find_relevant_docs(query, document_store)
            # 取交集或投票
            relevant_docs = vote_intersection([relevant_docs, docs2])
        
        # 标注者编写理想答案
        ideal_answer = annotators[0].write_ideal_answer(query, relevant_docs)
        
        samples.append(RAGEvalSample(
            query=query,
            gold_passages=relevant_docs,
            ground_truth_answer=ideal_answer,
            query_type=classify_query_type(query),
            domain=detect_domain(query),
            metadata={'annotator_count': len(annotators)}
        ))
    
    return RAGEvalDataset(
        name="custom_human_annotated",
        samples=samples,
        language="zh",
        size=len(samples),
        domain="general"
    )
```

## 数据集选择的决策矩阵

| 你的需求 | 推荐数据集 | 原因 |
|---------|-----------|------|
| 通用 RAG 检索评估 | BEIR / RGB | 覆盖全面，标准化 |
| 多跳推理能力 | HotpotQA / 2WikiMultihop | 天然的多跳设计 |
| 领域特定评估 | PubMedQA / LegalBench | 领域术语和文档结构 |
| 代码 RAG | CodeRAG-Bench | 代码结构相关 |
| 抗噪声评估 | RGB (Robustness) | 专门设计噪声场景 |
| 中文 RAG | CLAP / C-MRC | 中文语言和文档 |

## 评估数据集的"脏数据"问题

| 问题 | 影响 | 缓解方法 |
|------|------|---------|
| **引用了不存在的文档** | 无法准确判断检索召回 | 定期校验文档 ID 有效性 |
| **答案过时** | good answer 标记为错误 | 设置数据集的过期时间 |
| **标注者偏差** | 评估结果不稳定 | 多人交叉标注 + 一致性检验 |
| **查询-答案泄露** | 答案泄露了检索信息 | 分 query/retrieval/answer 三阶段标注 |
| **查询分布单一** | 评估结果不反映真实场景 | 多样化的查询来源 |

## 最大挑战

1. **数据集与真实用户分布的差异**：学术界数据集（NQ、TriviaQA）和真实产用户查询的分布差异很大
2. **标注质量保证**：大规模数据集的标注质量难以保证
3. **数据时效性**：知识密集型领域的数据集 6-12 个月就会过时
4. **多语言覆盖**：非英语语言的 RAG 评估数据集严重不足

## 工程建议

1. **采用混合评估**：标准数据集（定位问题）+ 领域数据集（验证适配）+ 线上采样（验证真实效果）
2. **数据集版本管理**：将评估数据集视为代码一样做版本控制
3. **定期更新**：至少每季度更新一次数据集，反映最新知识和用户需求
4. **分层测试**：基础能力（NQ/TriviaQA）→ 领域能力（领域数据）→ 定制能力（自定义数据）

## 推荐资源

- **BEIR**: https://github.com/beir-cellar/beir
- **RGB**: https://github.com/FlagOpen/FlagEmbedding/tree/master/LongEmbed
- **RAGAS 数据集**: RAGAS 框架内置的评估数据集
- **CodeRAG-Bench**: https://github.com/coderag-bench
