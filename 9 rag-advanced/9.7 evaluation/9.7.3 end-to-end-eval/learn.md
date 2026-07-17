# 9.7.3 end-to-end-eval — 端到端总体质量评分

## 简单介绍

端到端评估（End-to-End Evaluation）是将 RAG 系统作为一个整体进行评估——不单独看检索或生成，而是衡量**从用户输入到最终输出**的完整链路质量。端到端评估的核心理念是：**用户不关心检索精度或忠实度分数，只关心最终的答案是否解决了他们的问题。**

## 基本原理

### 端到端评估的层次结构

```
Level 1: 组件指标（Component Metrics）
  检索:  Precision@k, Recall@k, MRR, NDCG
  生成:  Faithfulness, Answer Relevance, Completeness
  → 这些是过程指标，开发人员用

Level 2: 任务指标（Task Metrics）
  任务完成率: 用户的最终目标是否达成
  错误率:     回答中有多少事实性错误
  关键信息覆盖率: 回答覆盖了多少用户需要的关键信息
  → 这些是结果指标，PM 用

Level 3: 体验指标（Experience Metrics）
  用户满意度: NPS / CSAT 评分
  任务耗时:   用户花了多长时间完成任务
  放弃率:     用户在多少步后放弃
  → 这些是业务指标，业务方用
```

## 端到端评估的三种主要方法

### 1. LLM-as-Judge 综合评分

```python
def end_to_end_llm_judge(query: str, answer: str, 
                          contexts: list[str]) -> dict:
    """让 LLM 从多维度对 RAG 回答进行综合评分"""
    
    prompt = f"""Evaluate this RAG system's response comprehensively.

QUERY: {query}
RETRIEVED CONTEXTS: {' '.join(contexts[:3])}
GENERATED ANSWER: {answer}

Score each dimension 0-10:
1. Correctness (正确性): Is the answer factually correct given the contexts?
2. Relevance (相关性): Does the answer directly address the query?
3. Completeness (完整性): Does it cover all aspects of the query?
4. Groundedness (归因): Is every claim supported by the contexts?
5. Clarity (清晰度): Is the answer well-structured and easy to understand?
6. Conciseness (简洁): Is it free of unnecessary information?

Output JSON:
{{
    "dimension_scores": {{"correctness": 8, "relevance": 9, ...}},
    "overall_score": 8.2,
    "strengths": ["..."],
    "weaknesses": ["..."],
    "suggestions": ["..."]
}}"""
    
    return llm_generate_json(prompt)
```

### 2. 任务完成率评估（Task Success Rate）

```python
def evaluate_task_completion(test_cases: list[TestCase]) -> dict:
    """评估 RAG 系统的任务完成率"""
    
    results = []
    for tc in test_cases:
        answer = rag_system.answer(tc.query)
        
        # 使用 LLM 判断任务是否成功
        success = judge_task_completion(
            query=tc.query, 
            answer=answer,
            expected=tc.expected_key_points
        )
        
        results.append({
            'query': tc.query,
            'success': success,
            'answer': answer
        })
    
    total = len(results)
    successes = sum(1 for r in results if r['success'])
    
    return {
        'task_success_rate': successes / total,
        'total_cases': total,
        'successful_cases': successes,
        'failed_cases': total - successes
    }


def judge_task_completion(query: str, answer: str, 
                          expected_key_points: list[str]) -> bool:
    """判断回答是否覆盖了所有关键点"""
    prompt = f"""Did the answer address ALL key aspects of the question?
    
    Question: {query}
    
    Key aspects to cover:
    {expected_key_points}
    
    Answer: {answer}
    
    Respond ONLY: "PASS" if all key aspects are addressed, "FAIL" otherwise."""
    
    result = llm_generate(prompt, max_tokens=5)
    return result.strip() == "PASS"
```

### 3. 对比评估（A/B Test / Pairwise Comparison）

```python
def pairwise_evaluation(queries: list[str], 
                         system_a_outputs: list[str],
                         system_b_outputs: list[str]) -> dict:
    """对比两个 RAG 系统的输出质量"""
    
    a_wins = 0
    b_wins = 0
    ties = 0
    
    for query, a_out, b_out in zip(queries, system_a_outputs, system_b_outputs):
        prompt = f"""Compare two responses and choose the BETTER one.
        
        Query: {query}
        
        Response A: {a_out}
        Response B: {b_out}
        
        Which is better considering correctness, completeness, and clarity?
        Answer ONLY: "A" or "B" or "Tie":"""
        
        result = llm_generate(prompt, max_tokens=5)
        if 'A' in result and 'B' not in result:
            a_wins += 1
        elif 'B' in result:
            b_wins += 1
        else:
            ties += 1
    
    total = len(queries)
    return {
        'system_a_win_rate': a_wins / total,
        'system_b_win_rate': b_wins / total,
        'tie_rate': ties / total,
        'preferred': 'A' if a_wins > b_wins else 'B'
    }
```

## 端到端评估框架

```python
class RAGEvaluator:
    """完整的 RAG 端到端评估框架"""
    
    def __init__(self, test_suite: list[TestCase]):
        self.test_suite = test_suite
    
    def evaluate(self, rag_system) -> EvalReport:
        """对 RAG 系统进行完整评估"""
        
        all_results = []
        
        for tc in self.test_suite:
            # 执行 RAG 流程
            result = rag_system.query(tc.query)
            
            # 收集各层指标
            eval_result = {
                'query': tc.query,
                'retrieved_docs': result.retrieved_docs,
                'answer': result.answer,
                
                # Layer 1: 检索指标
                'retrieval_metrics': compute_retrieval_metrics(
                    result.retrieved_docs, tc.relevant_docs
                ),
                
                # Layer 2: 生成指标
                'generation_metrics': compute_generation_metrics(
                    tc.query, result.answer, result.retrieved_docs
                ),
                
                # Layer 3: 端到端指标
                'task_success': judge_task_completion(
                    tc.query, result.answer, tc.expected_key_points
                ),
            }
            
            all_results.append(eval_result)
        
        return self._aggregate(all_results)
    
    def _aggregate(self, results: list) -> EvalReport:
        """汇总所有测试案例的评估结果"""
        return EvalReport(
            # 汇总检索指标
            avg_precision=np.mean([r['retrieval_metrics']['precision'] for r in results]),
            avg_recall=np.mean([r['retrieval_metrics']['recall'] for r in results]),
            avg_mrr=np.mean([r['retrieval_metrics']['mrr'] for r in results]),
            
            # 汇总生成指标
            avg_faithfulness=np.mean([r['generation_metrics']['faithfulness'] for r in results]),
            avg_relevance=np.mean([r['generation_metrics']['answer_relevance'] for r in results]),
            avg_completeness=np.mean([r['generation_metrics']['completeness'] for r in results]),
            
            # 汇总端到端指标
            task_success_rate=np.mean([r['task_success'] for r in results]),
            total_cases=len(results),
        )
```

## 端到端评估的设计原则

1. **分层分析**：当端到端指标变差时，能下钻到具体是哪一层的问题
2. **场景细分**：不同查询类型分开评估（事实性 vs 分析性 vs 指令性）
3. **错误分析**：除了聚合分数外，必须有定性错误分析
4. **回归检测**：增量改进后，确保旧场景不退化

## 最大挑战

1. **参考标准不一致**：不同评估者对"好回答"的标准不同
2. **长文本评估难**：超过 2000 token 的回答端到端评估质量下降明显
3. **评估成本**：使用 LLM-as-Judge 进行端到端评估的 token 成本是整个评估的主要开销
4. **场景覆盖**：测试集无法覆盖所有真实用户场景

## 推荐工具

- **RAGAS**: 端到端评估管线
- **TruLens**: 反馈函数式评估
- **LangSmith**: 评估数据集 + 自动化评估
- **MLflow Evaluation**: 内置 RAG 评估能力
