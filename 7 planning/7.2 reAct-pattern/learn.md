# 7.2 ReAct Pattern — ReAct 模式深度解析

ReAct（Reasoning + Acting）是 Agent 领域最具影响力的模式之一，它证明了推理与行动的协同远胜于任何单独一方。本节从原理到实现深度拆解 ReAct 的核心机制。

## 本节概述

ReAct 是支撑大多数现代 Agent 系统的基础架构。它的核心思想很简单——Agent 不应该先思考再行动，也不应该盲目行动，而是应该**边想边做**：产生一个思考（Thought），据此执行一个行动（Action），观察结果（Observation），然后基于新的观察产生下一个思考。这个 Thought-Action-Observation 循环是所有 Agent 系统的"CPU 指令周期"。

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|----------|------|
| 7.2.1 Reasoning Tracing | 推理轨迹如何生成与记录？ | ★★★☆☆ |
| 7.2.2 Action-Observation | 行动-观察循环如何设计与实现？ | ★★★☆☆ |
| 7.2.3 ReAct Variants | ReAct 有哪些有价值变体？ | ★★★★☆ |
| 7.2.4 Prompting ReAct | 如何用 Prompt 驱动有效 ReAct？ | ★★★★☆ |
| 7.2.5 ReAct Limitations | ReAct 的边界与陷阱在哪？ | ★★★☆☆ |
| 7.2.6 ReAct Implementation | 从零实现 ReAct 需哪些组件？ | ★★★★☆ |

## 核心思想

1. **推理与行动相互增强**：Thought 指导 Action 的选择，Observation 修正后续 Thought 的方向——二者形成正反馈循环。
2. **轨迹可见性**：ReAct 的所有中间步骤（Thought/Action/Observation）都是可读的，这提供了天然的可解释性和调试能力。
3. **无状态循环**：每一步 ReAct 循环都是自包含的——Agent 通过消息历史维护状态，而非内部可变状态。
4. **循环即计算**：对于复杂任务，增加循环次数（而不是增加单次推理的复杂度）是更可靠的扩展方式。

## 关键工程启示

- ReAct 循环的 prompt 设计是成败关键：格式约束比内容约束更重要
- 最大循环次数必须设置硬上限（通常 10-20 次），防止无限循环
- 每次循环的完整轨迹（Thought+Action+Observation）必须追加到消息历史中
- ReAct 在简单任务上的"过度思考"问题需要通过阈值机制避免

## 向下关联

- -> 7.3 Reflection: 在 ReAct 基础上增加反思步骤
- -> 7.4 Plan-and-Execute: Plan 阶段生成计划，Execute 阶段用 ReAct 执行每个步骤
- -> 8.5 Swarm Patterns: 多 Agent 系统中的每个 Agent 内部运行自己的 ReAct 循环
