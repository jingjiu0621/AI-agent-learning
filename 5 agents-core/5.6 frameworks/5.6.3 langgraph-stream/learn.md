# 5.6.3 langgraph-stream — LangGraph 流式与中断机制

## 简单介绍

LangGraph 的流式（Streaming）和中断（Interrupt）机制允许 Agent 在执行过程中逐节点输出结果，并支持在特定节点暂停等待用户输入。

## 基本原理

### 流式模式

```python
# 节点级别流式
for event in graph.stream({"messages": []}, config):
    # 每个节点执行完成后获得一个事件
    match event:
        case ("think", state):
            ui.show_thinking(state["messages"][-1])
        case ("act", state):
            ui.show_action(state["tool_calls"])
        case ("__end__", state):
            ui.show_final(state)
```

### 中断机制

```python
# 在图中插入中断点
graph = builder.compile(
    interrupt_before=["act"],  # 在 act 节点前暂停
    interrupt_after=["observe"]  # 在 observe 节点后暂停
)

# 第一次执行：在 act 前中断
result = graph.invoke({"messages": []}, config)
# → 用户确认后继续
result2 = graph.invoke(None, config)  # 从中断点继续
```

### 事件类型

| 事件 | 触发时机 | 用途 |
|------|---------|------|
| __start__ | 图开始执行 | 初始化 |
| node_name | 节点执行完成 | 获取节点输出 |
| __end__ | 图执行完成 | 获取最终结果 |
| updates | 仅变更的部分 | 增量更新 |

## 核心矛盾

**流式粒度 vs 性能开销**：
- 节点级流式：粒度适中，每个节点输出一次
- Token 级流式：更细粒度但实现更复杂
- 事件太多：客户端处理不过来

## 工程优化

1. 使用 stream_mode="updates" 只推送变更部分
2. 中断点设置在需要用户确认的关键决策节点
3. 结合 astream_events 获取更细粒度的事件
4. 中断恢复时验证外部状态是否变化

## 场景判断

- 自动模式：stream_mode="values" 获取完整状态
- UI 模式：stream_mode="updates" 增量渲染
- 人工审核：interrupt_before 关键决策节点
