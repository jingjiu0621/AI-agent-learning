# 6.3.3 进度跟踪 (Progress Tracking)

## 简单介绍

进度跟踪是工作记忆中负责维护任务完成状态的核心组件。它持续记录"已经做了什么、正在做什么、还要做什么"，为 Agent 提供任务执行的全局视角。没有进度跟踪，Agent 就像一个没有保存点的游戏玩家——一旦中断或需要切换任务，就必须从头开始。

## 基本原理

进度跟踪的核心机制是**状态标记与状态转换**。每个任务或步骤被赋予一个状态，随着执行推进，状态按照预定义的流程转换。进度跟踪器维护这些状态的完整视图，并在 Agent 需要决策时提供以下信息：

1. **已完成的工作**：避免重复执行已经完成的任务步骤
2. **当前的工作**：明确 Agent 此刻应该做什么
3. **剩余的工作**：帮助 Agent 规划后续步骤和时间分配
4. **阻塞的工作**：识别哪些任务因为依赖未满足而无法推进

进度跟踪与任务栈的区别：任务栈关注的是**嵌套结构**（任务的父子关系），进度跟踪关注的是**时间状态**（任务在生命周期中的位置）。二者互补，共同构成完整的执行视图。

## 背景与演进

| 阶段 | 形式 | 局限 |
|------|------|------|
| **隐式跟踪** | 依赖 LLM 从对话历史推断进度 | 不可靠，长上下文后容易出错 |
| **显式状态列表** | 将 todo/done 列表写入 prompt | 静态列表，不支持状态转换 |
| **状态机跟踪** | 每个任务有明确的状态转换图 | 需要预定义状态，灵活度有限 |
| **动态进度推理** | Agent 根据上下文自动推断并更新进度 | 实现复杂，对 LLM 推理能力要求高 |

## 核心矛盾

**精确性 vs 灵活性**：精确的进度跟踪需要预定义任务结构和状态转换规则（如传统的 Gantt 图或 PERT 图），但 Agent 面对的任务往往是动态变化的——新的子任务可能随时被引入，原有计划可能需要调整。过于刚性的进度跟踪无法适应变化，过于灵活的跟踪又无法提供可靠的进度视图。

**更新频率 vs 干扰成本**：频繁更新进度记录可以提高准确性，但每次更新都消耗 token 和计算资源。如果 Agent 每执行一个原子操作就更新一次进度，进度跟踪本身就会成为主要开销。

## 主流优化方向

### 1. 状态标记系统

基于有限状态机的状态标记系统是最常见的实现方式：

```python
class TaskStatus(Enum):
    NOT_STARTED = "not_started"        # 尚未开始
    IN_PROGRESS = "in_progress"        # 正在执行
    BLOCKED = "blocked"                # 被依赖项阻塞
    PAUSED = "paused"                  # 主动暂停
    COMPLETED = "completed"            # 成功完成
    FAILED = "failed"                  # 执行失败
    SKIPPED = "skipped"                # 被跳过（不满足执行条件）
    ROLLED_BACK = "rolled_back"        # 已回滚

    def can_transition_to(self, new_status):
        """定义合法的状态转换"""
        valid_transitions = {
            TaskStatus.NOT_STARTED: {TaskStatus.IN_PROGRESS, TaskStatus.SKIPPED},
            TaskStatus.IN_PROGRESS: {TaskStatus.COMPLETED, TaskStatus.FAILED,
                                     TaskStatus.BLOCKED, TaskStatus.PAUSED},
            TaskStatus.BLOCKED:     {TaskStatus.IN_PROGRESS, TaskStatus.SKIPPED},
            TaskStatus.PAUSED:      {TaskStatus.IN_PROGRESS, TaskStatus.SKIPPED},
            TaskStatus.COMPLETED:   {TaskStatus.ROLLED_BACK},  # 允许回滚
            TaskStatus.FAILED:      {TaskStatus.IN_PROGRESS},  # 允许重试
        }
        return new_status in valid_transitions.get(self, set())
```

### 2. 完成度计算

完成度不是简单的"已完成任务数 / 总任务数"，而应该考虑：

```python
def calculate_progress(task_tree):
    """递归计算任务树的加权完成度"""
    total_weight = 0
    completed_weight = 0

    for task in task_tree.children:
        weight = task.weight or 1.0  # 每个任务可以有不同权重
        total_weight += weight

        if task.status == TaskStatus.COMPLETED:
            completed_weight += weight
        elif task.children:
            # 递归计算子任务完成度
            child_progress = calculate_progress(task)
            completed_weight += weight * child_progress

    return completed_weight / total_weight if total_weight > 0 else 0.0
```

### 3. 进度感知的任务规划

进度跟踪不仅是被动的记录工具，还可以主动参与任务规划决策：

```python
def should_continue_current_path(progress_tracker):
    """基于进度判断是否应该继续当前路径"""
    recent_progress = progress_tracker.recent_progress(window=5)
    if recent_progress < MIN_PROGRESS_THRESHOLD:
        # 最近 N 步几乎没有推进 -> 建议更换策略
        return False
    return True

def estimate_remaining_effort(progress_tracker):
    """基于历史进度速率预估剩余时间"""
    completed = progress_tracker.completed_count()
    rate = progress_tracker.completion_rate()  # 任务/步
    remaining = progress_tracker.remaining_count()
    estimated_steps = remaining / rate if rate > 0 else float('inf')
    return estimated_steps
```

### 4. 增量更新与日志压缩

为避免进度更新信息膨胀，需要压缩机制：

```python
class CompressedProgressLog:
    def add_update(self, task_id, new_status, timestamp):
        # 只保留每个任务的最新状态转换
        self.latest_status[task_id] = (new_status, timestamp)

    def get_summary(self):
        # 生成压缩摘要
        completed = sum(1 for s, _ in self.latest_status.values()
                       if s == TaskStatus.COMPLETED)
        active = sum(1 for s, _ in self.latest_status.values()
                    if s == TaskStatus.IN_PROGRESS)
        blocked = sum(1 for s, _ in self.latest_status.values()
                     if s == TaskStatus.BLOCKED)
        return f"进度: {completed} 完成, {active} 进行中, {blocked} 阻塞"
```

## 产生的结果

良好的进度跟踪带来：
- **目标持久性**：Agent 不会在长时间执行中丢失原始目标
- **恢复能力**：中断后可以快速定位到断点继续执行
- **效率提升**：避免重复已完成的工作
- **可预测性**：基于历史进度率可以预估任务完成时间
- **透明度**：用户或上级 Agent 可以随时查看任务进展

## 最大挑战

**进度幻觉**：当任务状态维护不当或 Agent 对"完成"的定义与实际情况偏离时，会产生进度幻觉——看起来进度在推进，实际关键部分没有完成。典型的例子是 Agent 完成了"编写代码"的步骤，但代码有 bug 需要调试——在进度跟踪视角中，"编写代码"已经 100% 完成，但实际工作只完成了 60%。这种"完成定义不一致"是进度跟踪中最隐蔽的风险。

## 能力边界

进度跟踪无法处理：
1. **不确定完成条件**：当任务没有明确的"完成"标准时，进度跟踪失去了意义
2. **循环迭代任务**：需要反复修改的任务（如调参、优化），进度不是单调递增的
3. **外部依赖进度**：依赖外部系统（如人工审批）的任务，Agent 无法准确跟踪
4. **创造性探索任务**：探索性任务没有预定义的进度里程碑

## 区别对比

| 维度 | 被动式跟踪 | 主动式跟踪 | 预测式跟踪 |
|------|-----------|-----------|-----------|
| 更新方式 | 任务完成后标记 | 执行中连续更新 | 基于模型预测 |
| 准确度 | 高但不及时 | 及时但开销大 | 不确定性高 |
| 适用性 | 步骤明确的流程 | 长期运行的任务 | 有历史数据的场景 |
| 实现成本 | 低 | 中 | 高 |
| 对 LLM 依赖 | 低 | 中 | 高 |

## 核心优势

进度跟踪的核心价值是让 Agent 具备**执行连续性**和**全局可见性**。在执行连续性方面，Agent 不会因为上下文切换或中断而丢失任务状态；在全局可见性方面，Agent 始终知道自己处于整个任务流程中的哪个位置。

## 工程优化

### 检查点机制

```python
class CheckpointManager:
    def save_checkpoint(self, progress_tracker):
        checkpoint = {
            "timestamp": now(),
            "completed": list(progress_tracker.completed_tasks),
            "current": progress_tracker.current_task_id,
            "remaining": list(progress_tracker.remaining_tasks),
            "blocked": list(progress_tracker.blocked_tasks),
        }
        self.checkpoints.append(checkpoint)
        # 只保留最近 N 个检查点
        if len(self.checkpoints) > MAX_CHECKPOINTS:
            self.checkpoints.pop(0)
```

### 关键路径感知

```python
def highlight_critical_path(tasks, dependencies):
    """识别关键路径上的任务，给予更高关注度"""
    # 使用拓扑排序计算最早/最晚开始时间
    # 浮动时间为 0 的任务在关键路径上
    critical_tasks = find_critical_path(tasks, dependencies)
    for task in critical_tasks:
        task.priority += CRITICAL_PATH_BOOST
```

## 场景判断

| 场景 | 进度跟踪策略 | 理由 |
|------|-------------|------|
| 批处理任务 | 简单计数器 + 检查点 | 任务同质，完成度 = 完成数/总数 |
| 多阶段工作流 | 状态机 + 加权完成度 | 阶段重要性不同，需要加权 |
| 探索式分析 | 里程碑标记 + 日志摘要 | 无法预定义步骤，用里程碑定位 |
| 监控与告警 Agent | 主动跟踪 + 预测 | 需要预判异常趋势 |
| 代码开发 Agent | 关键路径 + 检查点 | 需要识别阻塞进度的关键任务 |
