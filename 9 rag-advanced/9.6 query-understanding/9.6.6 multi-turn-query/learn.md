# 9.6.6 multi-turn-query — 多轮查询上下文管理

## 简单介绍

多轮查询管理（Multi-Turn Query Management）是指在**多轮对话中维持、更新和利用历史上下文**来理解和处理当前查询的技术。在对话式 RAG 中，用户的当前问题往往引用或依赖之前轮次的上下文——如果没有多轮管理能力，每轮独立检索会丢失对话的连续性。

## 基本原理

### 多轮对话中的查询依赖类型

| 依赖类型 | 示例 | 需要的上下文 |
|---------|------|------------|
| **指代消解** | "它的作者是谁？" | 上一轮提到的"它"指代的对象 |
| **省略补充** | "那 Python 呢？" | 上一轮在比较什么（如语言性能比较） |
| **追问深入** | "为什么？" | 上一轮给出的解释内容 |
| **条件继承** | "如果数据量更大呢？" | 之前讨论的前提条件 |
| **话题切换** | "换个话题，说说数据库" | 需要清空旧的上下文 |

### 多轮查询处理的完整流程

```
Round 1:
  用户: "Transformer 和 RNN 有什么区别？"
  系统: [检索 + 回答] → "Transformer 使用自注意力机制..."
  保存上下文:
    - entities: [Transformer, RNN]
    - topic: "深度学习模型对比"
    - last_answer_summary: "Transformer 自注意力 vs RNN 循环结构"

Round 2:
  用户: "那BERT呢？它属于哪种？"   ← 省略了"和 Transformer 比"
  处理:
    1. 检测到省略 → 从上文补充 → "BERT 和 Transformer 是什么关系？"
    2. 指代消解 → "它" → Transformer
    3. 上下文继承 → 保持"模型对比"的检索策略
  检索: "BERT Transformer 关系 架构"
  回答: "BERT 是基于 Transformer 的 Encoder 部分的模型..."

Round 3:
  用户: "为什么现在都用这个？"   ← "这个"指代 Transformer
  处理:
    1. 指代消解 → "这个" → Transformer
    2. 追问检测 → 需要延续上一轮的回答
    3. 补充完整 → "为什么现在大家都使用 Transformer 而不是 RNN？"
  检索: "Transformer 流行原因 RNN 被取代"
```

## 多轮查询处理的核心技术

### 1. 指代消解（Coreference Resolution）

```python
def resolve_references(query: str, history: list[Exchange]) -> str:
    """解析当前查询中的指代，补充完整信息"""
    
    prompt = f"""Resolve all references in the current question using the conversation history.

    History:
    {format_history(history)}
    
    Current question: {query}
    
    If the current question contains references (it, they, this, that, the above, etc.), 
    rewrite it as a self-contained question. If not, return the original.
    
    Rewritten question:"""
    
    return llm_generate(prompt, max_tokens=100)
```

### 2. 对话状态追踪（Dialogue State Tracking）

```python
class DialogueState:
    """追踪对话状态"""
    def __init__(self):
        self.topic: str = None               # 当前话题
        self.entities: dict = {}              # 提到的实体 {name: type}
        self.last_answer_type: str = None     # 上次回答的类型
        self.query_count: int = 0             # 当前话题下的轮次
        self.active_filters: dict = {}        # 激活的检索过滤器
    
    def update(self, query: str, intent: Intent, response: str):
        self.query_count += 1
        
        # 检测话题是否切换
        new_topic = extract_topic(query)
        if new_topic and new_topic != self.topic:
            self.topic = new_topic
            self.query_count = 1
        elif not new_topic:
            pass  # 沿用当前话题
        
        # 更新实体
        new_entities = extract_entities(query)
        for ent in new_entities:
            if ent not in self.entities:
                self.entities[ent] = {'first_mention': self.query_count}
        
        # 更新活动过滤器
        filters = extract_filters(query)
        if filters:
            self.active_filters.update(filters)
    
    def get_context_for_retrieval(self) -> dict:
        """生成检索时需要的上下文信息"""
        return {
            'topic': self.topic,
            'recent_entities': list(self.entities.keys())[-5:],
            'active_filters': self.active_filters,
            'turn_count': self.query_count,
        }
```

### 3. 上下文压缩与缓存

多轮对话中历史可能很长，需要压缩：

```python
class ContextCompressor:
    def __init__(self, max_history_tokens: int = 2000):
        self.max_tokens = max_history_tokens
    
    def compress_history(self, history: list[Exchange]) -> str:
        """将对话历史压缩为适合检索参考的格式"""
        
        current_tokens = sum(ex.token_count for ex in history)
        
        if current_tokens <= self.max_tokens:
            return format_full_history(history)
        
        # 保留最近几轮完整 + 之前轮次的摘要
        recent = history[-3:]  # 保留最近3轮完整
        older = history[:-3]   # 前面的进行摘要
        
        older_summary = self._summarize_exchanges(older)
        
        return f"""
        [Previous conversation summary]: {older_summary}
        
        [Recent exchanges]:
        {format_full_history(recent)}
        """
    
    def _summarize_exchanges(self, exchanges: list[Exchange]) -> str:
        """将多轮对话压缩为一句话摘要"""
        prompt = f"""Summarize this conversation in one sentence:
        
        {format_full_history(exchanges)}
        
        One-sentence summary:"""
        
        return llm_generate(prompt, max_tokens=50)
```

### 4. 多轮检索上下文维护

```python
class MultiTurnRetriever:
    """支持多轮对话的检索器"""
    
    def __init__(self, base_retriever):
        self.retriever = base_retriever
        self.state = DialogueState()
    
    def retrieve(self, query: str, history: list[Exchange], top_k: int = 10):
        # Step 1: 指代消解
        resolved_query = resolve_references(query, history)
        
        # Step 2: 检测意图并更新状态
        intent = detect_intent(resolved_query, history)
        self.state.update(resolved_query, intent)
        
        # Step 3: 构建多轮增强的检索查询
        enhanced_query = self._enhance_query(resolved_query, intent)
        
        # Step 4: 执行检索（使用增强后的查询）
        results = self.retriever.retrieve(
            enhanced_query, 
            top_k=top_k,
            filters=self.state.active_filters
        )
        
        # Step 5: 多样性重排序——避免与之前检索的结果过于重复
        if len(history) > 0:
            previous_results = history[-1].retrieved_docs
            results = self._diversify(results, previous_results)
        
        return results
    
    def _enhance_query(self, query: str, intent: Intent) -> str:
        """用对话状态增强检索查询"""
        parts = [query]
        
        if self.state.topic and self.state.topic not in query:
            parts.append(f"(在 [{self.state.topic}] 的上下文中)")
        
        if self.state.active_filters:
            filters_str = ", ".join(f"{k}={v}" for k, v in self.state.active_filters.items())
            parts.append(f"filter: {filters_str}")
        
        return " ".join(parts)
```

## 多轮处理策略对比

| 策略 | 上下文保留 | Token 成本 | 实现复杂度 | 适用场景 |
|------|-----------|-----------|-----------|---------|
| 无历史（每轮独立） | 无 | 低 | 低 | 简单问答 |
| 拼接原始历史 | 全部 | 高 | 低 | 短对话 |
| 摘要历史 | 核心信息 | 中 | 中 | 通用场景 |
| 状态追踪 + 摘要 | 结构化 | 中-低 | 高 | 复杂多轮 |
| 全量+滑动窗口 | 最近完整 | 中 | 中 | 需要完整近期的场景 |

## 最大挑战

1. **长期依赖**：用户 20 轮前提到的条件在 21 轮被引用——如何不丢失也不过度保留
2. **话题漂移**：用户逐步从 A 聊到 Z，中间每个过渡都自然，但最终话题与起始完全无关——需要判断何时丢弃旧上下文
3. **错误传播**：上一轮的检索/回答错误会影响下一轮的上下文质量
4. **隐式否定**："不对，不是那个意思"——需要理解用户否定了上一轮的哪个部分

## 能力边界

- ✅ 处理 3-5 轮内的指代消解和上下文继承
- ✅ 管理对话状态的显式更新和切换
- ✅ 在 80% 的场景中正确理解省略和指代
- ❌ 超过 10 轮的复杂对话中，长期依赖经常丢失
- ❌ 跨多轮的隐式否定和修正难以完美跟踪
- ❌ 需要外部知识才理解的指代（用户说"上周说的那篇论文"）

## 工程优化

1. **分层历史管理**：完整的原始历史存数据库，上下文中只保留摘要+最近 N 轮
2. **显式确认**：关键上下文切换时向用户确认"我们还在讨论 X 吗？"
3. **固定大小状态表示**：使用固定长度的结构化解表示对话状态，避免无限增长
4. **预构建上下文标识**：为常见话题构建标识，快速识别话题回归

## 推荐工具

- **LangChain ConversationBufferMemory**: 简单对话缓冲
- **LangChain ConversationSummaryMemory**: 摘要型记忆
- **Mem0 / Zep**: 持久化对话记忆管理
- **DSPy 的多轮 Signature**: 结构化多轮处理
