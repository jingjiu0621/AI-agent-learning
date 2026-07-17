# 6.1.4 truncation-strategies — 截断策略：头部 / 尾部 / 关键信息保留

## 简单介绍

截断策略（Truncation Strategies）是当上下文窗口即将超出 Token 预算时，决定**丢弃哪些内容**的策略集合。截断的本质是**信息有损压缩**——如何在丢弃内容的同时，尽可能保留对当前任务最有价值的信息。

这是短期记忆中最实用也最需要权衡的技术——选错策略可能导致 Agent 丢失关键指令、遗忘用户目标或产生幻觉。

## 基本原理

### 截断的核心问题

假设上下文窗口满了（128K tokens），需要释放 20K 空间：

```
当前上下文布局：
┌────┬──────┬──────┬──────┬──────┬──────┬──────┐
│系统│工具  │历史  │历史  │历史  │最近  │最近  │
│指令│定义  │对话1 │对话2 │对话3 │用户  │助手  │
│2K  │30K   │20K   │20K   │20K   │20K   │16K   │
└────┴──────┴──────┴──────┴──────┴──────┴──────┘
总: 128K  →  需释放 20K
```

问题是：**丢哪些？**

### Transformer 注意力分布特性

理解截断策略的基础是 LLM 注意力机制的分布特性：

```
注意力权重分布（近似）：
高 │    ＊    ＊
   │   ＊＊   ＊＊   ＊＊
   │  ＊＊＊  ＊＊＊  ＊＊＊  ＊＊＊
低 │＿＿＿＿＿＿＿＿＿＿＿＿＿＿＿＿＿＿
   │ 开头    中间    结尾    位置
   │ Lost    ↓       Recency
   │ in the          Bias
   │ Middle
```

- **Lost in the Middle**：Transformer 对长上下文中间部分的内容注意力最低
- **Recency Bias**：对接近输出位置的最新内容注意力最高
- **Primacy Effect**：对开头部分（系统指令、任务定义）也有较高注意力

## 主流截断策略

### 1. 头部截断（Head Truncation）

```
策略：保留最近 N 条消息，丢弃最早的历史
```

```
[系统指令] [工具定义] [对话1] [对话2] [对话3] [用户] [助手]
  └─ 保留 ─┘ 保留 ──┘  ✂丢弃  ✂丢弃  保留    保留   保留
```

**特点**：
- **实现最简单**：直接删除列表头部的历史消息
- **保留系统指令**：系统指令和工具定义不受影响
- **丢失历史上下文**：最早的交互信息完全丢失
- **适用场景**：短期会话、独立任务（历史依赖弱）

```python
def head_truncate(messages: list, max_tokens: int, reserved: int) -> list:
    """头部截断：丢弃最早的非系统消息"""
    token_count = sum(count_tokens(m) for m in messages)
    if token_count <= max_tokens:
        return messages

    # 保护系统消息和最新的保留条数
    system_msgs = [m for m in messages if m["role"] == "system"]
    tail_msgs = messages[-reserved:] if reserved > 0 else []

    # 从中间消息中丢弃最早的
    protected = set(id(m) for m in system_msgs + tail_msgs)
    candidates = [m for m in messages if id(m) not in protected]

    result = system_msgs + tail_msgs
    budget = max_tokens - sum(count_tokens(m) for m in result)

    # 从最新到最旧逐条添加
    for m in reversed(candidates):
        tok = count_tokens(m)
        if budget - tok >= 0:
            result.insert(len(system_msgs), m)
            budget -= tok

    return sorted(result, key=lambda m: messages.index(m))
```

### 2. 尾部截断（Tail Truncation）

```
策略：只保留最早的部分，丢弃最新的内容（极少使用）
```

**特点**：
- **几乎不用**：因为会丢失最近的用户输入
- **极端场景**：仅在用户指令已完全执行、仅需最终总结时考虑
- **风险高**：丢失当前请求的上下文

### 3. 中间截断（Middle Truncation / Lost-in-the-Middle Aware）

```
策略：保留头部（系统指令）和尾部（最新交互），压缩中间内容
```

```
[系统指令] [工具定义] [摘要化的历史] [最近对话] [用户] [助手]
  └─ 保留 ─┘ 保留 ──┘ └─ 压缩 ─┘ └─── 保留 ────┘
```

**特点**：
- **符合注意力分布**：头部和尾部的注意力最高，中间最低
- **需要摘要能力**：对中间历史进行摘要压缩
- **实现复杂度中**：需要额外的 LLM 调用来生成摘要
- **最常用策略**：在保留关键信息和节省空间之间取得平衡

```python
def middle_truncate(messages: list, max_tokens: int) -> list:
    """中间截断+摘要：保留首尾，压缩中间"""
    system_msgs = [m for m in messages if m["role"] == "system"]
    tool_defs = [m for m in messages if m.get("tool_calls")]
    recent = messages[-10:]  # 最新 10 条

    fixed = system_msgs + tool_defs
    fixed_tokens = sum(count_tokens(m) for m in fixed)
    recent_tokens = sum(count_tokens(m) for m in recent)

    remaining = max_tokens - fixed_tokens - recent_tokens

    if remaining <= 0:
        # 空间严重不足：丢弃工具定义，只保留系统和最近
        return system_msgs + recent[-int(max_tokens * 0.3):]

    # 从中间区域选择性地包含高优先级的消息
    middle = messages[len(fixed):-10]
    budget = remaining
    selected = []
    for m in reversed(middle):  # 从最新的中间消息开始
        tok = count_tokens(m)
        if tok <= budget:
            selected.insert(0, m)
            budget -= tok

    return fixed + selected + recent
```

### 4. 语义截断（Semantic Truncation）

```
策略：基于语义相关性丢弃最不重要的内容
```

**特点**：
- **最智能**：不是简单按位置裁，而是评估每条消息的信息重要性
- **需要评分模型**：需要判断每条消息对当前任务的贡献
- **开销大**：评分本身消耗 Token 和计算资源
- **适合长会话**：对话历史极长时，语义筛选优于位置截断

```python
def semantic_truncate(messages: list, max_tokens: int, current_task: str) -> list:
    """语义截断：按信息重要性保留"""
    scored = []
    for msg in messages:
        importance = estimate_importance(msg, current_task)
        scored.append((importance, msg))

    # 按重要性降序排列
    scored.sort(key=lambda x: x[0], reverse=True)

    budget = max_tokens
    selected = []
    for importance, msg in scored:
        tok = count_tokens(msg)
        if tok <= budget:
            selected.append(msg)
            budget -= tok

    return selected  # 注意：失去了原始顺序，需要按位置重排序
```

### 5. 滑动窗口截断（Sliding Window Truncation）

```
策略：固定大小的窗口，新消息加入时最旧消息被挤出
```

```
─ 最早 ────────────────────────── 最新 ─→
 [消息1] [消息2] [消息3] ... [消息N]
                           ←── 窗口大小 W ─→
 消息1 被挤出 ← [消息N+1] 加入
```

**特点**：
- **实现极简**：FIFO 队列，新进旧出
- **可预测性强**：行为简单，容易调试
- **无优先级概念**：所有消息平等对待
- **适合流式场景**：持续对话中自然滚动

```python
class SlidingWindow:
    def __init__(self, window_size: int = 100):
        self.window_size = window_size
        self.buffer: deque[Message] = deque(maxlen=window_size)

    def append(self, msg: Message) -> list[Message]:
        """添加消息，返回被挤出的消息"""
        evicted = []
        if len(self.buffer) == self.window_size:
            evicted.append(self.buffer[0])
        self.buffer.append(msg)
        return evicted
```

### 6. 分层截断（Hierarchical Truncation）

```
策略：不同层级消息有不同优先级和保留策略
```

```
优先级层级：
  Level 0（永不丢弃）: 系统指令、安全约束
  Level 1（高优先级）: 用户当前任务、关键工具结果
  Level 2（中优先级）: 历史交互、检索文档
  Level 3（低优先级）: 日志、冗余输出、调试信息
```

**特点**：
- **精细控制**：每类消息有明确的保留策略
- **配置灵活**：可根据任务调整各层级的 Token 分配
- **实现较复杂**：需要消息分类逻辑
- **生产推荐**：实际生产系统中最常用的模式

```python
class HierarchicalTruncation:
    LEVELS = {
        "critical": 0,   # 永不丢弃
        "high": 1,       # 尽量保留
        "normal": 2,     # 按预算保留
        "low": 3,        # 优先丢弃
    }

    def truncate(self, messages: list, max_tokens: int) -> list:
        grouped = {0: [], 1: [], 2: [], 3: []}
        for msg in messages:
            level = self._classify(msg)
            grouped[level].append(msg)

        result = []
        result.extend(grouped[0])  # critical: always keep

        for level in [1, 2]:
            for msg in grouped[level]:
                tok = count_tokens(msg)
                if sum(count_tokens(m) for m in result) + tok <= max_tokens:
                    result.append(msg)

        # level 3: 尝试用摘要替代
        if grouped[3]:
            summary = self._summarize(grouped[3])
            if sum(count_tokens(m) for m in result) + count_tokens(summary) <= max_tokens:
                result.append(summary)

        return result
```

## 策略对比

| 策略 | 实现难度 | 信息保留 | 计算开销 | 适用场景 |
|------|---------|---------|---------|---------|
| 头部截断 | ★☆☆☆☆ | 低 | 极低 | 短期简单对话 |
| 中间截断 | ★★☆☆☆ | 中 | 低 | 通用场景 |
| 语义截断 | ★★★★☆ | 高 | 高 | 长会话、知识型任务 |
| 滑动窗口 | ★☆☆☆☆ | 中 | 极低 | 流式对话 |
| 分层截断 | ★★★☆☆ | 较高 | 中 | 生产系统 |

## 背景

### 为什么需要专门的截断策略

早期 LLM 应用（2022-2023）使用简单的"超出就报错"策略。但随着 Agent 系统变复杂，出现了一些问题：

1. **粗暴截断的连锁反应**：直接丢弃早期消息，导致 Agent 忘记任务目标、工具使用方式、用户偏好
2. **Lost in the Middle 的忽视**：简单保留首尾的截断策略没有解决中间区域的内容浪费问题
3. **一刀切的局限性**：不同重要性的消息被同等对待，关键信息可能被冗余信息挤掉

### 关键的认知突破

截断策略的核心认知是：**不是所有 token 都平等**。系统指令中的"不要调用危险工具"比用户说的"今天天气不错"重要得多。有效的截断不是均匀丢弃，而是基于信息价值的差异化保留。

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **保留系统指令 vs 保留用户输入** | 系统指令定义了 Agent 的行为边界，但用户输入包含当前任务需求 |
| **保留历史 vs 保留最近** | 早期历史可能包含关键上下文，最近消息包含当前任务目标 |
| **保留原文 vs 保留摘要** | 原文精确但占空间，摘要省空间但可能丢失细节 |
| **完整性 vs 可用性** | 保留更多信息可能让窗口超限导致请求失败 |

## 当前主流优化方向

1. **自适应截断**：根据当前任务的复杂度、历史长度、工具使用情况动态选择截断策略
2. **重要性感知截断**：结合语义分析，自动评估每条消息的信息价值，优先保留高价值内容
3. **分层预算管理**：为不同类型的消息分配固定的 Token 预算，互不挤占
4. **渐进式退化**：先在预算内用原文，超限后用摘要，再超限后丢弃——逐步降级而非一次性截断
5. **多轮截断记忆**：记录被截断内容的关键信息（如"已讨论过 X 话题"），在后续轮次中需要时重新注入

## 工程优化方向

1. **Token 预算预分配**：
   ```python
   BUDGET = {
       "system": 0.15,      # 系统指令 15%
       "tools": 0.20,       # 工具定义 20%
       "history": 0.25,     # 历史对话 25%
       "recent": 0.25,      # 最近交互 25%
       "reserved": 0.15,    # 回复预留 15%
   }
   ```

2. **截断触发阈值**：设置软阈值（80%）和硬阈值（100%），软阈值触发轻度截断，硬阈值触发强制截断

3. **截断审计日志**：记录每次截断的决策（丢了什么、为什么丢），便于调试和优化截断策略

4. **回退机制**：如果截断后 Agent 表现异常（频繁重复、逻辑断裂），自动切换更保守的截断策略

## 能力边界与结果边界

**能做的**：
- 在有限上下文窗口内最大化有效信息密度
- 通过策略组合减少关键信息丢失的概率
- 维持 Agent 在多轮对话中的任务连续性

**不能做的**：
- ❌ 无法做到无损截断——截断必然丢失信息
- ❌ 无法完全避免 Lost in the Middle 问题（架构层面限制）
- ❌ 不能替代长期记忆——如果关键信息被截断且没有持久化，它真的丢了

**结果边界**：
- 好的截断策略可以让 Agent 的有效对话轮数提升 3-5 倍
- 任何截断策略在上下文极度紧张时都会显著影响 Agent 表现
- 截断策略应该与长期记忆配合使用——被截断的关键信息应优先保存到长期记忆

## 与其他策略的区别

| 对比 | 截断策略 | 摘要压缩 | 消息合并 |
|------|---------|---------|---------|
| 操作 | 丢弃消息 | 重写为更短的表述 | 将多条合并为一条 |
| 信息损失 | 完全丢失 | 部分丢失 | 结构丢失 |
| Token 节省 | 最大 | 中等 | 中等 |
| 实现复杂度 | 低-中 | 高 | 中 |
| 适用场景 | Token 极度紧张 | 需要保留语义 | 格式化输出 |

## 核心优势

1. **实现简单见效快**：头部截断或滑动窗口可在 10 行代码内实现，显著提升多轮对话能力
2. **策略可组合**：不同策略可以组合成混合策略（如分层 + 语义），适应不同场景
3. **确定性行为**：相比 LLM 生成摘要，截断行为是可预测和可调试的
4. **零额外 Token 消耗**：截断本身不产生新的 Token

## 适合场景的判断标准

- **消息数量**：简单对话（<10 轮）→ 不需要截断；长对话（>20 轮）→ 需要中等策略；极长对话（>50 轮）→ 需要高级策略
- **任务类型**：独立任务 → 头部截断足够；依赖上下文的会话 → 需要语义/分层截断
- **消息异构性**：只有对话 → 滑动窗口；混合系统指令/工具/检索 → 分层截断
- **容错度**：高容错 → 简单截断；低容错 → 需要在截断前优先考虑摘要替代方案

**最适合**：生产环境中的长会话 Agent、多工具调用 Agent、需要维持多轮任务上下文的系统

**最不适合**：每轮会话都依赖完整早期上下文的任务（应当使用长期记忆代替截断）
