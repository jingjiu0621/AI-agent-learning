# 6.1.6 working-memory-link — 短期↔工作记忆的桥接

## 简单介绍

短期记忆与工作记忆的桥接（Working Memory Link）是 Agent 记忆架构中**信息升级**的通道——将短期记忆中"可能重要"的信息筛选、结构化并提升到工作记忆中，使其从"原始上下文数据"转变为"可操作的当前任务状态"。

如果说短期记忆是 LLM 的"草稿纸"（所有可见信息），工作记忆就是"草稿纸上被圈出来的关键信息"。桥接机制决定了哪些内容被圈出来、以什么形式组织、以及什么时候需要回写。

## 基本原理

### 为什么需要桥接

短期记忆和工作记忆在 Agent 系统中的角色完全不同：

```
短期记忆（数据级）                  工作记忆（状态级）
┌─────────────────────┐          ┌──────────────────────┐
│ "用户说：查北京天气"  │────桥接──→│ current_task: 查询天气 │
│ "工具返回：20°C"     │   筛选    │ target_city: 北京    │
│ "助手说：好的"       │   结构化  │ result: 20°C/晴朗    │
│ "用户又说：上海呢"   │   提要    │ next_action: 查上海  │
│ ... 更多原始消息 ...  │          │ status: 进行中       │
└─────────────────────┘          └──────────────────────┘
    短期记忆                             工作记忆
    · 所有上下文                           · 当前焦点
    · 原始格式                             · 结构化表示
    · 持续变化                             · 相对稳定
    · 大量冗余                             · 精简关键
```

### 桥接的核心操作

桥接机制执行三个核心转换：

```
筛选 (Filter) ──→ 结构化 (Structure) ──→ 注入上下文 (Contextualize)
     │                    │                       │
     │ 从原始消息中        │ 将提取的信息         │ 将工作记忆状态
     │ 提取与当前任务      │ 组织为结构化的       │ 注入回短期记忆
     │ 相关的关键信息      │ 任务状态             │ 指导 LLM 行为
```

### 双向流动

桥接不是单向的"短期→工作记忆"，而是双向的：

```
短期记忆 ──(提升)──→ 工作记忆
    ↑                    │
    └────(情境化)────────┘
    
    提升（Promotion）：短期记忆中的关键信息升级为工作记忆状态
    情境化（Contextualization）：工作记忆状态回注短期记忆，指导 LLM
```

## 核心机制

### 1. 注意力筛选（Attention Filtering）

从短期记忆中筛选出需要提升到工作记忆的关键信息。

```python
class AttentionFilter:
    """从短期记忆中筛选关键信息"""

    def __init__(self):
        self.extractors = {
            "task": TaskExtractor(),
            "entities": EntityExtractor(),
            "preferences": PreferenceExtractor(),
            "state_changes": StateChangeDetector(),
        }

    def promote_to_working_memory(self, short_term: ShortTermMemory) -> dict:
        """从短期记忆提取关键信息到工作记忆"""
        working_memory = {}

        for key, extractor in self.extractors.items():
            info = extractor.extract(short_term.messages)
            if info:
                working_memory[key] = info

        return working_memory
```

**筛选维度**：

| 维度 | 提取什么 | 示例 |
|------|---------|------|
| 任务目标 | 用户当前要求 Agent 完成的核心任务 | "帮我写一篇关于气候变化的文章" |
| 关键实体 | 人名、地名、日期、数字等命名实体 | "北京""2024-12-01""100万元" |
| 用户偏好 | 用户在交互中表达出的偏好信息 | "我喜欢简洁的回答""用表格展示" |
| 状态变更 | 任务进度的显式变化 | "步骤 1 已完成""用户已确认方案" |
| 约束条件 | 用户设定的限制和边界 | "不要使用外部 API""控制在 500 字内" |
| 未完成事项 | 尚未解决的待办和问题 | "需要查证 X 数据""等用户确认 Y" |

### 2. 结构化表示（Structured Representation）

从原始文本消息中提取结构化的工作记忆状态。

```python
@dataclass
class WorkingMemoryState:
    """工作记忆的规范化结构"""
    current_task: str | None = None
    subtasks: list[SubTask] = field(default_factory=list)
    entities: dict[str, EntityInfo] = field(default_factory=dict)
    preferences: list[Preference] = field(default_factory=list)
    constraints: list[Constraint] = field(default_factory=list)
    pending_items: list[str] = field(default_factory=list)
    last_update: float = 0.0

    def to_context_block(self) -> str:
        """格式化为可注入短期记忆的上下文块"""
        lines = ["<current_state>"]
        if self.current_task:
            lines.append(f"  task: {self.current_task}")
        if self.subtasks:
            for st in self.subtasks:
                status = "✓" if st.done else "○"
                lines.append(f"  {status} {st.description}")
        if self.preferences:
            lines.append("  preferences:")
            for p in self.preferences[-3:]:  # 最近 3 条偏好
                lines.append(f"    - {p.content}")
        lines.append("</current_state>")
        return "\n".join(lines)
```

### 3. 冲突检测（Conflict Detection）

当短期记忆中的新信息与工作记忆冲突时，需要解决冲突。

```python
class ConflictResolver:
    """工作记忆冲突检测与解决"""

    def check_conflicts(self, new_info: dict, current_wm: WorkingMemoryState) -> list[Conflict]:
        conflicts = []

        # 任务目标变更
        if "task" in new_info and current_wm.current_task:
            if new_info["task"] != current_wm.current_task:
                conflicts.append(Conflict(
                    type="task_change",
                    old=current_wm.current_task,
                    new=new_info["task"],
                    severity="high"
                ))

        # 偏好矛盾
        if "preferences" in new_info:
            for new_pref in new_info["preferences"]:
                for old_pref in current_wm.preferences:
                    if new_pref.contradicts(old_pref):
                        conflicts.append(Conflict(
                            type="preference_conflict",
                            old=old_pref.content,
                            new=new_pref.content,
                            severity="medium"
                        ))

        return conflicts

    def resolve(self, conflicts: list[Conflict]) -> WorkingMemoryState:
        """解决冲突（新信息优先，但高严重性冲突需要确认）"""
        for c in conflicts:
            if c.severity == "high":
                # 标记需要用户确认
                self.pending_confirmations.append(c)
            else:
                # 新信息覆盖旧信息
                self._apply_new(c)
        return self.state
```

### 4. 上下文情境化（Contextualization）

将工作记忆状态回注到短期记忆中，让 LLM "知道"当前任务状态。

```python
class Contextualizer:
    """工作记忆情境化——注入短期记忆"""

    def inject(self, working_memory: WorkingMemoryState,
               short_term: ShortTermMemory, strategy: str = "prefix"):
        """将工作记忆状态注入短期记忆"""

        context_block = self._format_context(working_memory)

        if strategy == "prefix":
            # 放在消息列表前面（紧接系统指令后）
            short_term.inject_at_position(1, {
                "role": "system",
                "content": f"## Current State\n{context_block}"
            })
        elif strategy == "suffix":
            # 放在消息列表末尾（最靠近输出）
            short_term.append({
                "role": "system",
                "content": f"[Internal State: {context_block}]"
            })
        elif strategy == "tool_result":
            # 伪装成工具结果注入
            short_term.append({
                "role": "tool",
                "tool_call_id": "_internal_state",
                "content": context_block
            })

    def _format_context(self, wm: WorkingMemoryState) -> str:
        lines = []
        if wm.current_task:
            lines.append(f"- Current task: {wm.current_task}")
        done = [s for s in wm.subtasks if s.done]
        pending = [s for s in wm.subtasks if not s.done]
        if done:
            lines.append(f"- Completed: {', '.join(s.description for s in done)}")
        if pending:
            lines.append(f"- Pending: {', '.join(s.description for s in pending)}")
        return "\n".join(lines)
```

## 桥接模式的演进

### 模式 1：隐式桥接（最简）

LLM 在生成过程中自然利用短期记忆中的信息，工作记忆隐式存在于模型的注意力中——无需显式桥接。

```
没有显式的工作记忆：LLM 注意力在短期记忆中直接寻找信息
适用：简单、短上下文任务
问题：注意力分散、没有结构化状态追踪
```

### 模式 2：显式注入桥接（推荐）

将工作记忆显式格式化为一个上下文块，周期性注入到短期记忆中。

```
每 N 轮或状态变更时注入：
[系统指令] [工作记忆状态] [工具定义] [对话历史...]
              ↑
         周期性刷新
适用：中等复杂度任务
优势：结构清晰、可调试
```

### 模式 3：事件驱动桥接（高级）

基于事件（任务完成、用户新输入、工具返回）触发工作记忆的更新和注入。

```python
class EventDrivenBridge:
    """事件驱动的桥接"""

    def __init__(self):
        self.wm = WorkingMemoryState()
        self.handlers = {
            "user_message": self._on_user_message,
            "tool_result": self._on_tool_result,
            "task_complete": self._on_task_complete,
            "error": self._on_error,
        }

    def process_event(self, event: Event, short_term: ShortTermMemory):
        handler = self.handlers.get(event.type)
        if handler:
            updates = handler(event)
            if updates:
                self.wm.apply(updates)
                # 仅在有关键变化时注入
                if self._is_significant(updates):
                    contextualizer.inject(self.wm, short_term)
```

## 背景

### 为什么需要独立的桥接机制

早期 Agent 不区分短期记忆和工作记忆——它们共享同一个消息列表。随着 Agent 系统变复杂，问题出现了：

1. **信息稀释**：任务相关的重要信息淹没在大量对话历史中
2. **状态跟踪困难**：在多步任务中，Agent 很难持续追踪"当前在做什么"
3. **上下文切换丢失**：当用户切换话题或任务时，旧任务的工作状态没有被正确归档
4. **LLM 注意力局限**：即使信息在上下文中，LLM 也可能没有"注意到"它

桥接机制解决了"信息在上下文里 ≠ LLM 在关注它"的问题。

### 认知科学基础

人类的**工作记忆**是一个容量有限的系统——一次只能处理 4±1 个"组块"。但工作记忆不是短期记忆的简单子集，它是一个**主动处理系统**：

- **中央执行系统**：控制注意力的分配、任务的切换
- **语音回路**：维护语言信息
- **视觉空间画板**：维护视觉和空间信息

Agent 的工作记忆桥接借鉴了"主动维护"的思想——不是被动地"记住"，而是主动地"提取、组织、维护"关键任务状态。

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **桥接频率 vs 开销** | 频繁更新工作记忆保持精确但消耗 Token；低频更新节省资源但可能状态过时 |
| **自动提取 vs 用户控制** | 自动提取高效但可能遗漏或误判；用户精确但增加交互成本 |
| **结构化 vs 灵活性** | 结构化表示易于 LLM 理解但可能过于 rigid；自由文本灵活但解析困难 |
| **稳定性 vs 敏感性** | 工作记忆需要稳定才能提供可靠引导，但过于稳定可能对新信息反应迟钝 |

## 当前主流优化方向

1. **自适应桥接频率**：根据任务复杂度、状态变更速率、Token 剩余量动态决定何时注入工作记忆状态
2. **多级桥接**：核心状态（任务目标）永驻短期记忆，动态状态（当前步骤）周期性注入，临时状态（工具参数）按需注入
3. **压缩表示**：工作记忆状态使用尽可能紧凑的格式（结构化标记、缩写、编码）
4. **增量更新**：不再全量注入工作记忆，而是只注入变化的部分（diff）
5. **预测性桥接**：基于任务模式预测即将需要的信息，提前提升到工作记忆

## 工程优化方向

1. **优先级缓存**：工作记忆在短期记忆中标记为"高优先级"，在截断时优先保留
2. **版本化工作记忆**：每次更新保持版本号，LLM 可以引用"状态 v3"来明确自己基于哪个版本做决策
3. **心跳注入**：不需要每次对话轮次都注入工作记忆，但设置一个最大间隔（如每 5 轮必须注入一次）
4. **用户确认环**：对于关键状态变更（如任务切换），要求 LLM 先确认再更新工作记忆
5. **工作记忆序列化**：会话结束时将工作记忆序列化到长期记忆，下次会话自动恢复

## 能力边界与结果边界

**能做的**：
- 将短期记忆中散乱的关键信息组织为结构化的任务状态
- 在长时间任务中维持 Agent 对"当前在做什么"的清晰认知
- 支持任务切换和状态恢复的多轮交互

**不能做的**：
- ❌ 不能替代短期记忆中信息的完整性（工作记忆是"索引"而非"完整副本"）
- ❌ 无法保证提取的信息 100% 正确（依赖提取算法的质量）
- ❌ 不能解决 LLM 根本不关注注入状态的问题（虽然很少见）

**结果边界**：
- 良好的桥接可以让 Agent 在长任务中的状态保持一致性提升 50-100%
- 桥接的频率和质量直接影响 Agent 在多步骤任务中的成功率
- 桥接策略需要针对具体任务调优，没有通用的最佳配置

## 与其他内容的区别

| 对比 | 短期-工作桥接 | 记忆检索(6.4) | 记忆整合(6.5) |
|------|-------------|-------------|-------------|
| 方向 | 短期→工作（提取）| 长期→短期（加载）| 短期→长期（沉淀）|
| 操作 | 筛选+结构化 | 搜索+评分 | 摘要+合并 |
| 频率 | 每轮或每事件 | 按需 | 空闲/批量 |
| 结果 | 任务状态 | 相关文档 | 长期知识 |
| 数据量 | 小（KB） | 中（KB-MB） | 大（长期积累）|

## 核心优势

1. **注意力引导**：显式告诉 LLM "现在最重要的是什么"，减少注意力分散
2. **状态持久性**：即使短期记忆被截断，工作记忆中的关键状态仍然保留
3. **结构化输出**：工作记忆的结构化表示比原始文本更适合 LLM 理解
4. **可调试性**：工作记忆的内容和变化可以被显式记录和审计

## 适合场景的判断标准

- **任务复杂度**：简单问答 → 不需要显式桥接；多步骤任务 → 需要桥接；复杂工作流 → 需要事件驱动桥接
- **上下文长度**：短上下文（<4K）→ 隐式桥接即可；长上下文（>32K）→ 需要显式注入
- **任务切换频率**：单任务 → 简单桥接；频繁切换 → 需要状态版本化和恢复机制
- **状态跟踪需求**：不需要状态跟踪 → 无桥接；需要跟踪进度 → 必须桥接

**最适合**：多步骤推理 Agent、工具调用密集型 Agent、需要长时间执行复杂任务的系统

**最不适合**：单轮问答、纯文本生成（不涉及任务状态管理）、简单信息检索
