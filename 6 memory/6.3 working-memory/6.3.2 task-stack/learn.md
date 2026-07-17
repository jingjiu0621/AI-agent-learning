# 6.3.2 任务栈 (Task Stack) — 子任务嵌套与优先级

## 简单介绍

任务栈是一个类似程序调用栈（call stack）的数据结构，用于管理 Agent 在执行复杂任务时产生的嵌套子任务。当 Agent 在完成一个主任务的过程中发现需要先完成某个前置子任务时，这个子任务被"压栈"；当子任务完成后，Agent "出栈"回到父任务继续执行。任务栈确保了多层嵌套任务的有序执行和正确返回。

## 基本原理

任务栈的核心思想来源于计算机科学中的栈式执行模型，但与函数调用栈有三个重要区别：

1. **任务粒度更大**：栈中的每个"帧"对应一个语义完整的任务单元，而非一条机器指令
2. **执行可能异步**：子任务可能涉及等待外部工具返回结果，栈需要支持挂起和恢复
3. **优先级动态变化**：Agent 可能在执行过程中发现更紧急的任务需要插队

任务栈的基本操作：

```
初始状态: [主任务]                              // 栈底
遇到子任务: [主任务, 子任务A]                    // 子任务A 压栈
又遇子任务: [主任务, 子任务A, 子任务A-1]         // 子任务A-1 压栈
子任务完成: [主任务, 子任务A]                    // 子任务A-1 出栈
发现依赖:   [主任务, 子任务A, 前置任务B]         // 更紧急的子任务插队
```

## 背景与演进

| 阶段 | 特点 | 局限 |
|------|------|------|
| **无任务栈** | Agent 线性执行，遇到子任务就"忘掉"父任务 | 无法回到父任务，任务链断裂 |
| **简单 LIFO 栈** | 严格按照后进先出的顺序管理子任务 | 无法处理插队和优先级 |
| **优先级队列** | 引入优先级评分，高优先级任务可插队 | 实现复杂，可能导致优先级反转 |
| **协作式多任务栈** | 任务可以主动让出执行权（yield），支持协程式切换 | 对 Agent 的自主调度能力要求高 |

## 核心矛盾

**执行深度 vs 上下文保持**：任务栈越深，Agent 需要同时维护的上下文越多。深栈意味着 Agent 可能在完成第 N 层子任务后忘记最初的顶层目标——"忘了最初要干什么"是复杂 Agent 最常见的问题之一。

**插队 vs 公平性**：允许高优先级任务插队可以提高任务效率，但过度插队可能导致低优先级任务"饿死"——永远得不到执行。

## 主流优化方向

### 1. 任务帧设计

每个任务帧（Task Frame）通常包含：

```json
{
  "taskId": "task_003",
  "parentId": "task_001",
  "goal": "从数据库查询用户订单历史",        // 任务目标
  "state": "running",                        // pending | running | paused | completed | failed
  "context": {},                             // 任务级别的上下文
  "partialResults": [],                      // 已产生的部分结果
  "priority": 5,                             // 优先级 (1-10)
  "depth": 2,                                // 栈深度
  "createdAt": "2026-07-16T10:30:00Z",
  "dependencyIds": ["task_002"]              // 依赖的任务
}
```

### 2. 深度控制与安全阀

任务栈深度必须有限制，否则 Agent 可能陷入无限嵌套：

```python
MAX_STACK_DEPTH = 5

def push_task(new_task):
    if current_depth() >= MAX_STACK_DEPTH:
        # 策略A: 拒绝入栈，要求当前任务先完成
        raise StackOverflowError("任务栈深度超限")
        # 策略B: 展平——将子任务合并到父任务中执行
        merge_into_parent(new_task)
        # 策略C: 挂起整个栈，切换到平坦模式
        switch_to_flat_mode()
```

### 3. 优先级调度

优先级调度需要解决两个关键问题：

**优先级反转**：低优先级任务持有高优先级任务需要的资源时，需要临时提升低优先级任务的优先级。

```python
def schedule_next():
    # 检查是否有因资源依赖需要提升优先级的任务
    for task in blocked_tasks:
        if task.holding_resource_for_high_pri:
            task.boosted_priority = task.holding_resource_for_high_pri
    # 选择当前最高优先级的可执行任务
    return max(runnable_tasks, key=lambda t: t.effective_priority())
```

**任务抢占**：高优先级任务到来时，是否需要中断当前正在执行的低优先级任务？

```python
def on_high_priority_arrival(new_task):
    current = peek_stack()
    if new_task.priority > current.priority + THRESHOLD:
        current.state = "paused"    # 暂停当前任务
        current.save_checkpoint()   # 保存检查点
        push_task(new_task)         # 新任务入栈
    else:
        enqueue_to_pending(new_task)  # 加入等待队列
```

### 4. 任务状态机

每个任务帧可以处于以下状态：

```
         ┌─────────────────────────────────────┐
         │                                     │
    ┌────────┐    ┌─────────┐    ┌─────────┐   │
    │ pending│───>│ running │───>│completed│   │
    └────────┘    └─────────┘    └─────────┘   │
       │              │                         │
       │         ┌────┴────┐                    │
       │         │         │                    │
       │    ┌────────┐ ┌────────┐               │
       └───>│ paused │ │ failed │               │
            └────────┘ └────────┘               │
                  │                             │
                  └─────────────────────────────┘
```

## 产生的结果

设计良好的任务栈带来：
- **任务可追踪**：每个子任务的来源、依赖、状态都可查询
- **错误隔离**：某个子任务失败不会直接导致整个任务链崩溃
- **资源合理分配**：优先级机制确保重要任务不会因为嵌套过深而被遗忘
- **恢复能力**：任务栈的快照支持意外中断后的恢复

## 最大挑战

**上下文断裂**：当栈深度超过 3-4 层时，LLM 在处理当前任务时可能已经"遗忘"了顶层任务的完整上下文。任务帧中如何携带足够的父任务上下文、而又不使每个帧过于臃肿，是一个两难问题。

## 能力边界

任务栈的线性拓扑结构限制了它无法有效建模：
1. **图状依赖**：当子任务的依赖关系是 DAG（有向无环图）而非树时，任务栈不适合
2. **长期挂起任务**：需要在栈中挂起数分钟甚至数小时的任务（如等待人工审批）
3. **协作型多 Agent 任务**：多个 Agent 各自维护独立的任务栈，协调变得极为复杂

## 区别对比

| 维度 | 简单 LIFO 栈 | 优先级调度栈 | 协作式任务栈 |
|------|-------------|-------------|-------------|
| 调度策略 | 后进先出 | 动态优先级 | 主动让出 |
| 复杂度 | 低 | 中 | 高 |
| 实时性 | 无 | 支持紧急插队 | 依赖任务自觉 |
| 公平性 | 差 | 可能有饿死 | 好 |
| 适用场景 | 线性流程 | 多优先级场景 | 长期运行 Agent |

## 核心优势

任务栈为 Agent 提供了**结构化的问题分解能力**。没有任务栈，Agent 只能处理扁平任务；有了任务栈，Agent 可以将任意复杂的问题递归分解到可执行的粒度，然后逐层返回结果。

## 工程优化

### 上下文压缩传递

当子任务入栈时，不是复制整个父任务上下文，而是传递"最小必要上下文"：

```python
def create_child_context(parent_frame, child_task):
    # 只传递子任务实际需要的上下文子集
    needed_keys = analyze_context_needs(child_task.goal)
    return {
        key: parent_frame.context[key]
        for key in needed_keys
        if key in parent_frame.context
    }
```

### 栈可视化调试

```python
def visualize_stack(stack):
    for i, frame in enumerate(reversed(stack.frames)):
        indent = "  " * i
        print(f"{indent}[{frame.state}] {frame.goal[:50]} (depth={frame.depth})")
        if frame.state == "paused":
            print(f"{indent}  -> waiting for: {frame.waiting_for}")
```

## 场景判断

| 场景 | 任务栈策略 | 理由 |
|------|-----------|------|
| 数据分析流程 | 简单 LIFO 栈，深度 ≤ 3 | 分析流程天然是分步的，嵌套不深 |
| 多 Agent 协作 | 浅栈 + 消息队列 | 子任务可能被其他 Agent 执行，不需要深栈 |
| 实时监控 Agent | 优先级抢占栈 | 告警处理必须能够中断当前低优任务 |
| 代码生成与调试 | 深度控制栈（≤ 4） | 编写->运行->调试->修复 是经典嵌套模式 |
| 工作流自动化 | 协作式任务栈 + 定时恢复 | 可能包含长时间等待步骤（人工审批） |
