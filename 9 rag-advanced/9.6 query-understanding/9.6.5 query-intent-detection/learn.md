# 9.6.5 query-intent-detection — 查询意图检测

## 简单介绍

查询意图检测（Query Intent Detection）是**理解用户提问背后的深层目的**的技术。与查询分类（对查询表面的"形式"分类）不同，意图检测关注的是用户**"为什么"**问这个问题——他们的真实需求是什么，最终想完成什么任务。

## 基本原理

### 意图 vs 分类的区别

```
查询: "Python 的多线程"

分类（表面形式）:
  - 类型: "概念解释"
  - 复杂度: "中等"

意图（深层目的）:
  - "想了解 Python 多线程的用法"           → 教程/入门
  - "想知道 Python 多线程和 GIL 的关系"      → 进阶
  - "想用 Python 多线程优化程序性能"         → 实际问题解决
  - "想确认 Python 多线程是否适合 IO 密集型任务" → 技术选型
```

### 意图维度体系

一个完整的意图检测系统需要覆盖以下维度：

```python
class QueryIntent:
    def __init__(self):
        self.task_intent: str = None     # 任务意图
        self.information_need: str = None # 信息需求类型
        self.urgency: str = None          # 紧迫度
        self.expertise_level: str = None  # 用户的专业水平
        self.desired_output: str = None   # 期望的输出形式
```

### 常见的意图模式

```python
INTENT_PATTERNS = {
    'learn_basics': {
        'pattern': ['入门', '基础', '什么是', '教程', 'guide', 'tutorial'],
        'response_style': 'educational_step_by_step',
        'retrieval_strategy': 'introductory_materials',
    },
    'solve_problem': {
        'pattern': ['报错', 'error', 'bug', '坏了', '不工作', '失败'],
        'response_style': 'troubleshooting',
        'retrieval_strategy': 'error_solutions_and_debugging',
    },
    'compare_options': {
        'pattern': ['区别', 'vs', '对比', '哪个', '选择', '选型'],
        'response_style': 'comparison_table',
        'retrieval_strategy': 'comparison_and_benchmarks',
    },
    'get_summary': {
        'pattern': ['总结', '概述', '摘要', '大意', '重点'],
        'response_style': 'concise_summary',
        'retrieval_strategy': 'overview_and_key_points',
    },
    'deep_dive': {
        'pattern': ['原理', '源码', '底层', '机制', '实现方式', 'how it works'],
        'response_style': 'detailed_technical',
        'retrieval_strategy': 'in_depth_sources',
    },
    'practical_implementation': {
        'pattern': ['代码', '实现', 'demo', '示例', '例子', 'practice'],
        'response_style': 'code_example',
        'retrieval_strategy': 'code_and_examples',
    },
}
```

## 背景：为什么需要意图检测

在传统搜索中，用户意图已经是一个核心课题（搜索"苹果" vs 搜索"苹果官网"是完全不同的意图）。在 RAG 系统中，意图检测更重要，因为：

1. **检索策略差异**："入门教程"和"源码分析"需要检索完全不同的内容
2. **回答风格差异**：解决问题的回答需要步骤化，概念解释需要通俗易懂
3. **上下文管理**：如果用户想解决 bug，需要提供调试建议，而非纯理论

## 意图检测的方法

### 1. LLM 意图分析

```python
def llm_intent_analysis(query: str, history: list[dict] = None) -> Intent:
    """使用 LLM 检测深层意图"""
    
    context = format_history(history) if history else "(no prior context)"
    
    prompt = f"""Analyze the user's DEEP INTENT behind this query.
    Not just what they asked, but WHY they are asking it.
    
    User context: {context}
    Current query: {query}
    
    Analyze:
    1. What is the user's primary goal?
    2. What prior knowledge do they likely have?
    3. What form of answer would be most helpful?
    4. What potential follow-up needs might they have?
    5. Is there any implied urgency or frustration?
    
    Response format:
    {{
        "primary_intent": "learn/solve/compare/decide/create/explore",
        "goal": "one sentence describing the real goal",
        "knowledge_level": "beginner/intermediate/expert",
        "answer_format": "step_by_step/code_example/comparison/conceptual",
        "urgency": "low/medium/high",
        "follow_up_topics": ["topic1", "topic2"]
    }}"""
    
    return llm_generate_json(prompt)
```

### 2. 隐式信号检测

从语言模式推断用户状态：

```python
def detect_implicit_signals(query: str) -> dict:
    """检测查询中隐含的用户信号"""
    
    signals = {}
    
    # 挫败感检测
    frustration_words = ['不工作', '失败', '报错', '坑', '垃圾', '没用', '崩溃']
    signals['frustration'] = any(w in query for w in frustration_words)
    
    # 紧急程度检测
    urgency_words = ['急', '紧急', ASAP, 'urgent', '马上', '立刻']
    signals['urgency'] = any(w in query for w in urgency_words)
    
    # 专业水平推断
    advanced_terms = extract_advanced_terms(query)
    signals['likely_expertise'] = 'advanced' if len(advanced_terms) > 2 else 'intermediate'
    
    # 时间敏感性
    temporal_words = ['最新', '2024', '2025', '当前', '最近', '新']
    signals['time_sensitive'] = any(w in query for w in temporal_words)
    
    return signals
```

### 3. 会话历史意图追踪

```python
class IntentTracker:
    """追踪多轮对话中的意图变化"""
    
    def __init__(self):
        self.intent_history: list[Intent] = []
        self.current_intent: Intent = None
    
    def update(self, query: str, response: str):
        """更新意图追踪状态"""
        new_intent = self._detect_intent(query)
        
        # 检测意图是否切换
        if self.current_intent:
            switch_type = self._classify_intent_switch(
                self.current_intent, new_intent
            )
            
            if switch_type == 'refinement':
                # 用户在同一个大意图下深入
                new_intent.parent_intent = self.current_intent.primary_intent
            elif switch_type == 'shift':
                # 用户切换了话题
                pass
            elif switch_type == 'follow_up':
                # 用户追问上一条回答的细节
                new_intent.references_last_response = True
        
        self.current_intent = new_intent
        self.intent_history.append(new_intent)
    
    def _classify_intent_switch(self, old: Intent, new: Intent) -> str:
        if old.primary_intent == new.primary_intent:
            return 'refinement'
        
        # 一些常见的意图切换模式
        follow_up_patterns = [
            (old.primary_intent == 'learn' and 'example' in new.query),
            (old.primary_intent == 'solve' and 'why' in new.query),
        ]
        
        if any(follow_up_patterns):
            return 'follow_up'
        
        return 'shift'
```

## 不同意图的检索策略映射

```python
def intent_to_retrieval_strategy(intent: Intent) -> RetrievalPlan:
    """根据意图决定检索策略"""
    
    strategy_map = {
        'learn_basics': RetrievalPlan(
            sources=['tutorials', 'documentation_getting_started'],
            top_k=5,
            diversity=0.3,  # 中等多样性
            time_range='all',
        ),
        'solve_problem': RetrievalPlan(
            sources=['stack_overflow', 'github_issues', 'debug_guides'],
            top_k=10,
            diversity=0.7,  # 高多样性（多种可能的解决方案）
            time_range='recent',  # 最新解决方案
        ),
        'compare_technologies': RetrievalPlan(
            sources=['comparison_articles', 'benchmarks', 'reviews'],
            top_k=8,
            diversity=0.8,
            time_range='last_2_years',
        ),
        'deep_technical': RetrievalPlan(
            sources=['research_papers', 'source_code', 'specifications'],
            top_k=15,
            diversity=0.5,
            time_range='all',
        ),
    }
    
    return strategy_map.get(
        intent.primary_intent, 
        RetrievalPlan(sources=['general'], top_k=5, diversity=0.5)
    )
```

## 意图检测方法对比

| 方法 | 准确率 | 延迟 | 可解释性 | 维护成本 |
|------|--------|------|---------|---------|
| 关键词规则 | 60-70% | ~1ms | 高 | 高 |
| 小模型分类器 | 75-85% | ~20ms | 中 | 中 |
| **LLM 零样本** | 80-90% | ~200ms | 高 | 极低 |
| **LLM + CoT** | 85-95% | ~400ms | 最高 | 极低 |

## 最大挑战

1. **用户意图的模糊性**：有时候用户自己都不清楚想得到什么
2. **多意图融合**：一个查询可能包含多个意图（"我想了解 Python 并发编程，顺便比较一下多线程和协程的优劣"）
3. **跨文化差异**：不同语言/文化背景下，表达同样意图的语言模式不同
4. **隐式意图**：用户问"A"但实际想知道"B"（"什么是 Transformer？" → 实际想了解的是"为什么 Transformer 比 RNN 好"）

## 能力边界

- ✅ 检测 80%+ 的常见用户意图模式
- ✅ 通过意图映射优化检索策略和回答风格
- ✅ 追踪多轮对话中的意图流转变换
- ❌ 无法读取用户未表达的深层需求（用户自己没意识到的需求）
- ❌ 无法应对高度域特定且无训练数据的意图
- ❌ 受语言表达能力的显著影响（不善表达的用户难以准确检测）

## 工程优化

1. **级联检测**：先快速规则匹配，不明确时回退到 LLM
2. **个性化意图模型**：根据用户历史交互调整意图检测的倾向
3. **意图验证**：生成回答前让 LLM 确认"我理解您的需求是 X，对吗？"
4. **反馈学习**：用户对回答的满意度反馈指导意图模型的更新
