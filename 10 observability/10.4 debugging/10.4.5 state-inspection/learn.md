# 10.4.5 State Inspection — Agent 状态快照检查

## 简单介绍

State Inspection（状态检查）关注的是 Agent 在任意时刻的"内部状态"——已经收集了哪些信息、当前在做什么、还有哪些任务待完成、当前的工作记忆内容是什么。与"日志"记录过去的事件不同，状态检查回答的是 **"Agent 现在处于什么状态"**。

## 基本原理

Agent 的状态抽象为几个核心维度：

```python
@dataclass
class AgentStateSnapshot:
    """Agent 在某一时刻的完整状态快照"""

    # 1. 执行状态
    status: str                  # running / paused / completed / error
    current_step: int
    total_steps_planned: int
    current_phase: str           # thinking / acting / observing

    # 2. 任务状态
    task_stack: list[Task]       # 任务栈（子任务嵌套）
    completed_tasks: list[str]
    pending_tasks: list[str]
    current_task: Task

    # 3. 工作记忆
    working_memory: dict         # 关键信息缓存
    collected_info: list[str]    # 已收集的信息摘要
    hypotheses: list[str]        # 当前的假设/推断

    # 4. 消息历史状态
    message_count: int
    oldest_message_age: float    # 最早消息到现在的时间
    context_utilization: float   # 上下文利用率 %
    compression_status: str      # none / partial / compressed

    # 5. 工具状态
    tools_called: list[str]      # 本轮已调用的工具
    last_tool_result: Any        # 最近一次工具返回的结果
    tool_call_count: int
```

状态检查在调试中的应用：

```python
class StatefulAgent:
    def get_state_snapshot(self) -> AgentStateSnapshot:
        """生成当前状态快照"""

        return AgentStateSnapshot(
            status=self._status,
            current_step=self._step_counter,
            total_steps_planned=len(self._plan) if self._plan else 0,

            # 任务栈展示
            task_stack=[Task(
                id=t.id,
                description=t.description,
                status=t.status,
                depends_on=[d.id for d in t.dependencies],
            ) for t in self._task_stack],

            # 工作记忆缓存
            working_memory={
                "last_search_query": self._last_search_query,
                "intermediate_results": list(self._results_cache.keys()),
                "current_focus": self._focus,
            },
        )

    def check_state_health(self) -> list[StateWarning]:
        """检查状态健康度"""
        warnings = []
        if len(self._task_stack) > MAX_TASK_DEPTH:
            warnings.append(StateWarning("任务栈过深", self._task_stack))
        if self._context_utilization > 0.9:
            warnings.append(StateWarning("上下文即将用尽", f"{self._context_utilization:.0%}"))
        if self._detect_loop():
            warnings.append(StateWarning("检测到可能的死循环"))
        return warnings
```

## 背景

传统软件的"状态"是由代码中变量值定义的，开发者可以设置断点查看所有变量的值。但 Agent 的状态更复杂：

1. **隐式状态**——部分"状态"不在显式的变量中，而是隐含在消息历史中（如"之前说过什么影响了 Agent 的当前理解"）
2. **动态状态**——Agent 的状态随着每个 ReAct 循环快速变化
3. **多层次状态**——会话状态、任务状态、环境状态、记忆状态的多个层次叠加

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 只看变量 | print 检查 Agent 对象的字段 | 只有部分状态暴露为字段，大量隐式状态被忽略 |
| 看日志推断状态 | 从日志记录推断 Agent 当前的状态 | 延迟大，不精确，无法回答"Agent 现在在想什么" |
| 不检查状态 | 依赖 Agent 的最终输出来分析问题 | 无法诊断"中间过程"的问题 |
| 看 LLM 上下文 | 直接查看 LLM 的 messages 数组 | 信息量太大，状态信息散落在大量消息中 |

## 当前主流优化方向

1. **可恢复的状态快照**——Agent 的状态快照设计为不仅可查看，还可以恢复。在调试时修改状态后从修改点继续执行
2. **状态差异对比**——对比"预期的状态"和"实际的状态"之间的差异，自动标记异常（如"本应该已经确认用户身份，状态显示尚未确认"）
3. **状态可视化**——用流程图或树状图展示 Agent 的任务栈和进度，而非文字描述
4. **声明式状态检查**——在 Agent 框架中声明"关键状态不变式"（如"任何时候 agent_id 不能为空"、"task_stack 不能超过 10 层"），运行时自动检查

## 实现的最大挑战

1. **完整状态的序列化**——Agent 的某些状态可能是不可序列化的（如数据库连接、网络句柄），需要特殊处理才能在快照中展示
2. **快照大小**——一个包含完整消息历史的 Agent 状态快照可能达到数 MB，频繁快照会影响性能
3. **状态一致性问题**——在 Agent 执行过程中获取状态快照可能捕获到"不一致状态"（如正在修改任务栈时拍摄的快照）
4. **异步状态的快照**——并发执行多个工具调用时，各个协程的状态需要同步快照

## 能力边界

**能做什么：**
- 精确了解 Agent 在任意时刻的执行进度和状态
- 检查 Agent 的内部推理是否基于正确的信息
- 发现状态异常（死循环、任务栈溢出、上下文即将用尽）
- 在调试中可以查看和修改 Agent 的状态

**不能做什么：**
- 不能自动修复 Agent 的状态问题——快照暴露问题，修复需要手动或自动恢复机制
- 不能完全捕获所有 Agent 的隐式状态——LLM 的"内部状态"（推理过程中形成的理解）对调试器不可见
- 不能保证快照的零开销——获取快照总是有性能代价的

## 最终工程优化

1. **增量状态快照**——不每次生成完整快照，而是记录状态的变化（diff），只在需要时重建完整快照，大幅降低性能开销
2. **快照轮转存储**——保留最近的 N 个快照（如 5 个），循环覆盖，便于回溯 Agent 最近几步的状态变化
3. **不可变状态设计**——在 Agent 框架中使用不可变状态对象，快照可以直接获取引用而不需要深拷贝，零开销产生快照
4. **状态不变式自动告警**——将"任务栈深度 > 10"、"同一工具调用 > 5 次"、"上下文利用率 > 95%"等状态异常自动转化为告警，无需人工检查
