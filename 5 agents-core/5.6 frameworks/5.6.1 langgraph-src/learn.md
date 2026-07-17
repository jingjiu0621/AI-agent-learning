# 5.6.1 langgraph-src — LangGraph 源码分析

## 简单介绍

LangGraph 是 LangChain 生态的 Agent 编排框架，核心模型是 StateGraph（有向图）。Agent 的执行被建模为"节点（Node）"和"边（Edge）"的图遍历，节点是计算单元，边是控制流。

## 基本原理

### 核心概念

```
StateGraph
  ├── nodes: Dict[str, Node]  # 节点：执行一个计算
  │     └── Node: (State) → State
  ├── edges: List[Edge]        # 边：节点间的连接
  │     ├── 普通边：固定顺序
  │     └── 条件边：根据条件选择下一节点
  └── State: TypedDict         # 全局状态，在节点间传递
```

### 执行流程

```python
# 构建图
builder = StateGraph(AgentState)
builder.add_node("think", think_node)
builder.add_node("act", act_node)
builder.add_edge("think", "act")
builder.add_conditional_edges("act", decide_next, ["think", "end"])

# 编译执行
graph = builder.compile()
result = graph.invoke({"messages": []})
```

### LangGraph 的状态传递

```
Node A (修改 state.key1)
    │
    ↓ state = {key1: new_value, key2: old_value, ...}
Node B (修改 state.key2)
    │
    ↓ state = {key1: new_value, key2: new_value2, ...}
```

**关键设计**：每个节点只修改 State 的部分字段，未修改的字段自动传递给下一节点。

## 背景与演进

LangGraph 解决了 LangChain Chain 模型的两大问题：
1. Chain 是线性的，无法表达分支和循环
2. Chain 的状态管理隐式且混乱

LangGraph 将 Agent 执行建模为图，天然支持分支、循环和并行。

## 核心矛盾

**图表达力 vs 学习曲线**：
- 图模型可以表达任意复杂的执行流
- 但也意味着更多的概念（节点、边、条件边、State）

## 源码核心设计

| 组件 | 作用 | 关键实现 |
|------|------|---------|
| StateGraph | 图构建器 | 维护 nodes/edges 注册表 |
| CompiledGraph | 编译后的可执行图 | 拓扑排序 + 执行引擎 |
| Pregel | 底层执行引擎 | 超步（Superstep）执行模式 |
| Checkpointer | 状态持久化 | 每步保存 State 快照 |

## 核心优势

- **灵活表达**：任意复杂的 Agent 流程都可以用图表达
- **内置状态管理**：State 自动在节点间传递
- **可检查点**：每一步都可保存和恢复

## 工程优化

1. 节点职责单一：每个节点只做一件事（思考/行动/观察…）
2. State 定义最小化：只包含需要在节点间共享的数据
3. 条件边用函数而非 lambda（便于序列化）
4. 合理设置 recursion_limit 防止无限循环

## 场景判断

适合 LangGraph 的场景：
- 需要复杂分支和条件逻辑的 Agent
- 需要状态持久化和恢复的生产 Agent
- 需要流式输出的 Agent

不适合的场景：
- 简单的线性 Agent（Chain 更简单）
