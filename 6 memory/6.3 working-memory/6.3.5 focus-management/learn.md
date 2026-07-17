# 6.3.5 focus-management — 注意力焦点管理

## 简单介绍

注意力焦点管理（Focus Management）是工作记忆中决定 Agent **当前应该关注什么**的机制。在复杂多步骤任务中，Agent 同时面对大量信息——用户指令、工具返回、系统消息、历史上下文——焦点管理帮助 Agent 在这些信息中识别"当前最重要的事"，避免注意力分散和任务偏离。

如果把工作记忆比作一个"桌面"，焦点管理就是桌面上那盏可调节的聚光灯——它决定光照在什么地方，以及光照的亮度。

## 基本原理

### 注意力的有限性

Agent 的注意力是**容量有限且选择性**的：

```
LLM 在每一步能"真正关注"的信息远小于上下文中的总量。
焦点管理的目标：最大化有效注意力的利用率。

┌──────────────────────────────────────────────┐
│                  上下文窗口                     │
│  [系统指令] [工具定义] [对话历史] [检索内容] [用户输入] │
│                                              │
│         ████████████████                      │
│         焦点区域（实际被高度关注的部分）           │
└──────────────────────────────────────────────┘
```

### 焦点域（Focus Field）

焦点管理将工作记忆划分为不同的焦点区域：

```python
@dataclass
class FocusField:
    """Agent 当前的注意力焦点域"""
    primary: str | None           # 主要焦点：当前核心任务
    secondary: list[str]          # 次要焦点：相关但非核心
    context: list[str]            # 背景信息：不需要活跃关注
    excluded: list[str]           # 明确排除：不相关内容

    def is_focused_on(self, topic: str) -> bool:
        return topic in self.primary or topic in self.secondary
```

### 焦点转移（Focus Shift）

当 Agent 需要切换任务或上下文时，焦点管理执行焦点转移：

```
时间 ──────────────────────────────────────────────→

焦点: [任务 A] ──→ [任务 A(验证)] ──→ [任务 B] ──→ [任务 B]
状态:  分析数据       确认结果         新需求       执行方案
        ████████      ████            █████████      ██████
        高聚焦         中聚焦           高聚焦         高聚焦
                    ↑焦点转移↑                    ↑焦点转移↑
```

## 核心机制

### 1. 焦点定位（Focus Targeting）

决定"当前应该关注什么"的策略：

```python
class FocusTarget:
    """焦点定位——基于多种信号决定注意力方向"""

    def determine_focus(self, context: Context) -> FocusField:
        signals = []

        # 信号 1: 用户最近的明确指令
        if user_intent := self._extract_intent(context.recent_user_msg):
            signals.append(("user_intent", user_intent, 1.0))

        # 信号 2: 当前任务阶段
        if task_phase := context.state.get("current_phase"):
            required_focus = TASK_FOCUS_MAP.get(task_phase, [])
            signals.append(("task_phase", required_focus, 0.8))

        # 信号 3: 工具调用返回的异常
        if error := context.last_tool_error:
            signals.append(("error", f"处理错误: {error}", 0.9))

        # 信号 4: 未完成的子任务
        if pending := context.state.get("pending_subtasks", []):
            signals.append(("pending", pending[0], 0.7))

        # 信号 5: 周期性检查（长时间未关注某重要项）
        if stale := self._check_stale_items(context):
            signals.append(("stale", stale, 0.5))

        # 综合决策
        return self._resolve_signals(signals)
```

**信号优先级**：
```
错误/异常    >   用户明确指令    >   任务关键阶段    >   待办提醒    >   环境通知

（发生错误时，无论当前在做什么，Agent 应优先关注错误处理）
```

### 2. 焦点维护（Focus Maintenance）

防止注意力漂移——在长时间任务中保持焦点一致性：

```python
class FocusMaintenance:
    """焦点维护——防止注意力漂移"""

    def __init__(self, focus_decay: float = 0.1):
        self.focus_history: list[FocusRecord] = []
        self.focus_decay = focus_decay  # 每轮衰减率

    def update(self, current_focus: FocusField, round_num: int):
        """更新焦点记录并检查漂移"""
        self.focus_history.append(FocusRecord(
            round=round_num,
            primary=current_focus.primary,
            secondary=current_focus.secondary,
        ))

        # 检查焦点漂移
        drift = self._detect_drift()
        if drift > DRIFT_THRESHOLD:
            self._refocus()

    def _detect_drift(self) -> float:
        """检测焦点是否在漂移"""
        if len(self.focus_history) < 3:
            return 0.0

        recent = self.focus_history[-3:]
        topics = [r.primary for r in recent if r.primary]

        if len(set(topics)) / len(topics) > 0.5:
            # 超过一半的历史焦点不同 → 漂移
            return len(set(topics)) / len(topics)
        return 0.0

    def _refocus(self):
        """重新聚焦到原始任务"""
        original_focus = self.focus_history[0].primary
        self.current_focus.primary = original_focus
```

### 3. 焦点切换（Focus Switching）

优雅地在不同任务之间切换注意力：

```python
class FocusSwitcher:
    """焦点切换——保存和恢复焦点上下文"""

    def __init__(self):
        self.focus_stack: list[FocusSnapshot] = []  # 焦点栈

    def switch_to(self, new_focus: FocusField, reason: str):
        """切换到新焦点"""
        # 保存当前焦点（压栈）
        if self.current_focus:
            snapshot = FocusSnapshot(
                focus=self.current_focus,
                context=self._capture_context(),
                timestamp=time.time(),
            )
            self.focus_stack.append(snapshot)

        # 切换到新焦点
        self.current_focus = new_focus
        self.switch_reason = reason

    def switch_back(self):
        """切回上一个焦点"""
        if self.focus_stack:
            snapshot = self.focus_stack.pop()
            self.current_focus = snapshot.focus
            self._restore_context(snapshot.context)

            # 注入"已回归"提示
            return f"[已从 {snapshot.focus.primary} 回到 {self.current_focus.primary}]"
        return ""

    def _capture_context(self) -> dict:
        """捕获当前焦点的上下文快照"""
        return {
            "question": self.current_focus.primary,
            "partial_result": self.partial_result,
            "considered_options": self.considered_options,
        }
```

## 焦点管理策略

### 策略 1：单焦点（Single Focus）

一次只关注一件事——适合线性任务。

```
优点：注意力高度集中，不易出错
缺点：无法处理多任务交织
适用：简单的顺序任务
```

### 策略 2：焦点轮转（Focus Round-Robin）

在多个任务间周期性轮转注意力。

```python
def round_robin_focus(tasks: list[Task], interval: int = 3) -> FocusField:
    """按轮次轮转焦点"""
    current_round = get_global_round()
    task_idx = (current_round // interval) % len(tasks)
    return FocusField(
        primary=tasks[task_idx].description,
        secondary=[t.description for t in tasks if t != tasks[task_idx]],
        context=[],
        excluded=[]
    )
```

**优点**：公平分配注意力
**缺点**：频繁切换有上下文切换成本
**适用**：需要同时监控多个独立任务流

### 策略 3：优先级驱动（Priority-Driven）

根据任务的优先级动态分配注意力。

```
优先级队列：
  [P0] 用户中断/错误   ← 立即抢占
  [P1] 当前任务核心步骤 ← 主要焦点
  [P2] 次要任务/常规   ← 空闲时关注
  [P3] 背景信息/日志   ← 被动接收
```

**优点**：重要的任务总是优先
**缺点**：优先级评估本身有成本
**适用**：需要实时响应紧急事件的 Agent

### 策略 4：状态驱动焦点（State-Driven Focus）

根据 Agent 当前所处的状态自动决定关注点。

```python
STATE_FOCUS_MAP = {
    "understanding": lambda ctx: FocusField(
        primary="用户需求理解",
        secondary=["关键实体", "约束条件"],
        context=["历史背景"],
        excluded=["无关细节"],
    ),
    "planning": lambda ctx: FocusField(
        primary="任务分解与规划",
        secondary=["资源评估", "依赖分析"],
        context=["原始需求"],
        excluded=["执行细节"],
    ),
    "executing": lambda ctx: FocusField(
        primary=f"执行步骤 {ctx.step}/{ctx.total_steps}",
        secondary=["当前步骤的工具结果", "是否需要调整计划"],
        context=["完整计划"],
        excluded=["已完成步骤的细节"],
    ),
    "verifying": lambda ctx: FocusField(
        primary="验证执行结果",
        secondary=["与预期对比", "异常情况"],
        context=["原始需求", "执行计划"],
        excluded=["中间计算细节"],
    ),
}
```

## 与 Prompt 中焦点指示的交互

焦点管理最终需要作用于 LLM 的输出——通过 Prompt 中的焦点指示实现：

```python
def inject_focus_instruction(focus: FocusField) -> str:
    """生成焦点指示注入到 Prompt"""
    lines = ["<focus>"]
    if focus.primary:
        lines.append(f"  primary: {focus.primary}")
    if focus.secondary:
        lines.append(f"  secondary: {', '.join(focus.secondary[:2])}")
    if focus.excluded:
        lines.append(f"  ignore: {', '.join(focus.excluded[:3])}")
    lines.append("</focus>")
    return "\n".join(lines)
```

在系统提示中的典型用法：
```
你现在关注的是：{focus.primary}
次要关注：{focus.secondary}
不需要关注的：{focus.excluded}

请将主要精力放在主要关注点上。
```

## 背景

### 为什么需要显式的焦点管理

早期 Agent（2023-2024）完全依赖 LLM 的内在注意力机制来关注信息——没有显式的焦点管理。但实践中发现：

1. **注意力稀释**：上下文越长，LLM 对关键信息的关注越弱（Lost in the Middle）
2. **任务漂移**：在长时间多步骤任务中，Agent 容易"忘记"正在做什么，转而关注不相关的细节
3. **上下文切换成本**：用户突然切换话题时，Agent 需要多轮才能"回过神来"
4. **无效循环**：Agent 反复关注同一个已解决的问题，因为"不记得已经解决过"

显式焦点管理解决了"LLM 上下文中有很多信息，但不知道哪个是此刻最重要的"问题。

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **聚焦 vs 灵活** | 聚焦程度越高，对突发相关信息的敏感度越低 |
| **稳定 vs 响应** | 维护焦点稳定防止漂移，但过度稳定导致无法快速响应新需求 |
| **广焦点 vs 深焦点** | 广焦点覆盖更多但精度低，深焦点精度高但可能遗漏其他重要信号 |
| **自动 vs 用户控制** | 自动焦点管理减少用户负担，但可能错误判断用户的注意力意愿 |

## 当前主流优化方向

1. **预测性焦点**：基于任务类型和历史模式，预测 Agent 下一步应该关注什么
2. **多尺度焦点**：不同时间尺度上维持不同粒度的焦点（如宏观任务目标保持稳定，微观执行步骤频繁切换）
3. **注意力审计**：跟踪 Agent 每轮实际关注的内容（通过分析 LLM 输出中的引用），与预期焦点对比，自动调整
4. **用户注意力信号**：利用用户的行为（重复提到某话题、强调标记）作为焦点信号的增强输入
5. **焦点学习**：积累焦点决策的经验数据，训练焦点策略模型

## 工程优化方向

1. **焦点日志**：记录每轮的焦点状态和转移原因，用于事后分析和调试
2. **焦点预算**：为不同焦点等级分配 Token 预算（主要焦点 50%，次要 30%，背景 20%）
3. **焦点辅助提示**：在系统 Prompt 底部动态插入焦点指示，帮助 LLM 集中注意力
4. **回退机制**：焦点切换时保留上一个焦点的上下文快照，确保可以无损切回
5. **焦点可视化**：开发时提供焦点状态可视化界面，便于调试焦点管理策略

## 能力边界与结果边界

**能做的**：
- 在多步骤、多任务的复杂场景中维持 Agent 的注意力一致性
- 通过显式焦点指示减少 LLM 的注意力分散（经验上可提升任务完成率 15-30%）
- 支持优雅的上下文切换，减少切换后的"迷茫"轮次

**不能做的**：
- ❌ 改变 LLM 底层注意力机制（Lost in the Middle 是架构问题，焦点管理只能在 Prompt 层面缓解）
- ❌ 确保 Agent 100% 遵循焦点指示（LLM 可能忽略焦点提示）
- ❌ 替代良好的任务分解和状态管理——焦点管理是辅助，不能替代清晰的逻辑

**结果边界**：
- 焦点管理对短上下文（<4K）任务的效果有限（此时 LLM 天然注意力集中）
- 焦点管理在长上下文（>32K）和复杂多步骤任务中效果最显著
- 过度聚焦可能导致 Agent 忽视环境中的重要变化

## 与其他内容的区别

| 对比 | 焦点管理 | 任务分解(7.1) | 工作记忆维护(6.3) |
|------|---------|-------------|-----------------|
| 关注 | 应该关注什么 | 应该做什么 | 当前状态是什么 |
| 操作 | 设置注意力方向 | 分解任务 | 维护状态变量 |
| 产物 | 焦点指令 | 任务列表 | 状态记录 |
| 时间视角 | 当前 | 未来 | 现在 |
| 成功信号 | 注意力集中 | 步骤完成 | 状态准确 |

## 核心优势

1. **减少注意力浪费**：让 LLM 把有限的"推理资源"集中在关键信息上
2. **增强行为一致性**：Agent 在长时间任务中保持"记得当前在做什么"
3. **支持复杂任务交织**：通过焦点栈和切换机制，Agent 可以优雅地处理多任务
4. **可调试性**：焦点日志提供了"Agent 为什么关注 X"的可解释性

## 适合场景的判断标准

- **任务复杂度**：单步骤任务 → 不需要焦点管理；多步骤复杂任务 → 需要
- **上下文长度**：短上下文（<4K）→ 效果有限；长上下文（>16K）→ 有明显收益
- **多任务程度**：单任务线性 → 简单焦点；多任务交织 → 需要焦点切换机制
- **历史模式**：Agent 行为经常漂移 → 必须改善焦点管理

**最适合**：复杂多步骤推理 Agent、需要实时响应中断的生产 Agent、长时间自主运行的 Agent

**最不适合**：单轮问答、完全确定性的流水线任务、上下文极短（<2K）的场景
