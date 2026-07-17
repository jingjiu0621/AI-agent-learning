# 3.2.2 Context Injection — 动态上下文注入技术

## 简单介绍

Context Injection（上下文注入）是指将运行时获取的动态数据——用户输入、历史对话、工具返回结果、外部数据库查询等——精确地注入到 Prompt 中的技术。它是动态 Prompt 系统的数据通道。模板引擎决定了变量的"位置"，Context Injection 决定了变量的"来源"和"何时注入"。在 AI Agent 开发中，上下文注入的质量直接影响 LLM 对当前状态的理解准确度。

## 基本原理

上下文注入的核心流程是一个三元组：**数据源 → 数据变换 → 注入策略**。数据源定义从何处获取上下文（对话历史、工具结果、知识库等），数据变换负责将原始数据转为 LLM 友好的格式（截断、摘要、结构化），注入策略决定何时以及以何种优先级注入这些数据到当前 Prompt 中。

```python
class ContextInjector:
    def __init__(self):
        self.sources = {}  # name -> ContextSource

    def register_source(self, name, source_fn, priority=0, max_tokens=500):
        self.sources[name] = {
            "fn": source_fn,
            "priority": priority,
            "max_tokens": max_tokens,
        }

    def build_context(self, max_total_tokens=3000):
        """构建最终注入 Prompt 的上下文块"""
        sorted_sources = sorted(
            self.sources.values(),
            key=lambda s: s["priority"],
            reverse=True,
        )
        context_blocks = []
        remaining = max_total_tokens

        for source in sorted_sources:
            if remaining <= 0:
                break
            data = source["fn"]()
            block = self._truncate(data, min(remaining, source["max_tokens"]))
            context_blocks.append(block)
            remaining -= self._count_tokens(block)

        return "\n\n".join(context_blocks)
```

## 背景


## 之前针对这个问题的做法与结果

**做法一：全量注入** — 将所有可用上下文全部塞入 Prompt。优点是简单，缺点是指数级增长的 Token 消耗，容易触顶上下文窗口，且 LLM 在大量噪声中"迷失"关键信息（Lost in the Middle 现象）。

**做法二：手动截断** — 开发者手动选择注入哪些字段。优点是可控，缺点是不灵活——当场景变化时需要修改代码，且无法适应不同的上下文窗口大小。

**做法三：固定优先级** — 预定义各类上下文的优先级顺序，按优先级裁剪。比手动截断灵活，但仍然是静态策略，无法根据用户意图动态调整什么信息对当前任务更重要。

## 核心矛盾

**信息丰富度与注意力稀释之间的矛盾**。注入更多上下文可以给 LLM 更多参考信息，但过多的上下文（尤其是中间部分的上下文）反而会降低 LLM 对关键信息的关注度。研究（Liu et al., 2023 的 Lost in the Middle 论文）表明，LLM 对输入开头和结尾的信息最敏感，中间部分的信息容易被忽略。因此，上下文注入不仅仅是"放进去"，更是"放在哪里"和"什么时候甚至可以不放"的精细决策问题。

## 当前主流优化方向

1. **结构化注入** — 使用 XML/JSON/Markdown 对不同类型的上下文做标签化封装，使得 LLM 能更好地区分信息来源。例如 `<conversation_history>...</conversation_history>` 与 `<tool_result>...</tool_result>`。

2. **选择性注入** — 通过意图识别判断当前需要哪些上下文。例如用户问"刚才的代码还有 bug 吗"时，只需注入最近的代码相关对话，而非整段历史。

3. **分层摘要注入** — 对长对话历史或大型文档，先生成摘要再注入摘要而非原始内容。多层摘要（按时间/主题分层）在信息保留和 Token 节约之间取得平衡。

4. **动态优先级注入** — 根据当前任务动态调整上下文的优先级。例如工具调用失败后，错误信息的优先级瞬时升高；用户提到某个项目名时，该项目相关文档的优先级提升。

5. **滑动窗口注入** — 对于流式对话，保持一个滑动的上下文窗口，近期消息完整保留，早期消息缩略或丢弃。

## 实现的最大挑战

**上下文窗口的 Token 预算管理**是最大挑战。Agent 的上下文窗口是固定的（如 128K tokens），但需要容纳：系统 Prompt、历史对话、当前用户输入、工具调用记录和结果、检索到的知识片段。这些组件的长度是动态变化的。一个高效的注入系统需要：（1）实时计算各组件 Token 数；（2）根据优先级动态分配预算；（3）在预算不足时智能降级（用摘要替代全文、丢弃低优先级内容）；（4）保证系统 Prompt 等核心部分永远不会被截断。

```python
class TokenBudgetManager:
    def __init__(self, total_budget=128000):
        self.total_budget = total_budget

    def allocate(self, components):
        """
        components: [{"name": str, "tokens": int, "min_tokens": int, "priority": int}]
        """
        # 第一阶段：保证最低需求
        for c in components:
            c["allocated"] = min(c["min_tokens"], c["tokens"])

        remaining = self.total_budget - sum(c["allocated"] for c in components)

        # 第二阶段：按优先级分配剩余预算
        sorted_comps = sorted(components, key=lambda c: c["priority"])
        for c in sorted_comps:
            extra = min(c["tokens"] - c["allocated"], remaining)
            c["allocated"] += extra
            remaining -= extra
            if remaining <= 0:
                break

        return {c["name"]: c["allocated"] for c in components}
```

## 能力边界与结果边界

Context Injection 能保证"正确的信息在正确的位置"，但无法保证"LLM 会正确使用这些信息"。注入是将数据放入窗口，但 LLM 是否注意到并使用这些数据取决于模型自身能力、注入格式和位置。此外，好的注入策略需要准确的意图理解——错误判断用户意图会导致注入不相关的上下文，反而干扰 LLM。

## 与其他技术的区别

与 Template Engine 的区别：模板引擎关注"格式和位置"，Context Injection 关注"来源和选择"。与 Prompt Chaining 的区别：Context Injection 是一次性的上下文收集，Chain 是多步的信息处理。实际项目中，Context Injection 的输出作为模板引擎的输入变量。

## 核心优势

- **Token 效率**：不是把所有上下文都塞进去，而是精心选择最相关的部分
- **质量提升**：结构化的上下文注入让 LLM 更容易区分信息来源，减少幻觉
- **可配置性**：不同的 Agent 场景可以配置不同的注入策略和优先级

## 工程优化方向

1. **上下文命中率监控**：追踪哪些注入的上下文被 LLM 实际使用（通过输出中引用到的信息来推断），优化注入策略
2. **压缩率自适应**：根据当前 Token 使用率自动调整各组件压缩率（摘要 vs 截断 vs 完整）
3. **预测性预注入**：根据用户输入的前几个 token 预判需要哪些上下文，提前异步加载，减少首 token 延迟
4. **注入效果 A/B 测试**：对比不同注入策略对下游任务准确率的影响

## 适合场景的判断标准

以下情况需要专门的 Context Injection 系统：（1）Agent 需要访问外部数据（数据库、API、知识库）；（2）对话历史较长（超过 10 轮）；（3）有多个不同的上下文来源需要按优先级管理；（4）上下文窗口出现频繁的"超出预算"警告。如果 Agent 是无状态的一次性问答，则不需要复杂的注入系统。
