# 5.1.6 loop-variants — 循环变体：ReAct Loop / Reflexion Loop / Plan-Execute Loop

## 简单介绍

Agent 主循环并非只有一种实现方式。不同的任务场景需要不同的循环结构。最常见的三种循环变体是 ReAct Loop（推理-行动交织）、Reflexion Loop（带自我反思）、Plan-Execute Loop（先规划后执行）。理解每种变体的适用场景是设计高效 Agent 的关键。

## 基本原理

### ReAct Loop

最基础的循环变体，Thought-Action-Observation 三者紧密交织：

```
Thought₁ → Action₁ → Observation₁ → Thought₂ → Action₂ → ... → Final Answer
```

- **特点**：推理与执行交替进行，无独立规划阶段
- **适合**：探索性任务，不确定下一步需要什么信息

### Reflexion Loop

在 ReAct 基础上增加反思（Reflection）环节：

```
Trajectory₁ → Reflection → Trajectory₂ → Reflection → ... → Best Result
     ↑                                          ↓
     └──────── 基于反思改进策略 ──────────────┘
```

- **特点**：多轮尝试，每轮结束后反思改进
- **适合**：需要试错的复杂推理任务

### Plan-Execute Loop

将规划与执行分离为两个阶段：

```
规划阶段：
  ┌─────────────────────────────────┐
  │  Plan: Step 1 → Step 2 → Step 3 │
  └─────────────────────────────────┘
              ↓
执行阶段：
  Step 1 → Observation → Step 2 → ... → Output
              ↓ (如失败则重规划)
            Re-plan
```

- **特点**：先制定全局计划再逐步执行
- **适合**：目标任务明确、步骤可预见的场景

## 背景与演进

| 时期 | 主要变体 | 代表作 |
|------|---------|--------|
| 2022.10 | ReAct | ReAct: Synergizing Reasoning and Acting |
| 2023.03 | Reflexion | Reflexion: Language Agents with Verbal Reinforcement |
| 2023.06 | Plan-Execute | Plan-and-Solve Prompting |
| 2024 | 混合变体 | LangGraph 支持自定义循环控制 |

## 核心矛盾

**灵活性 vs 可控性**：
- ReAct 最灵活但最不可控（可能乱转）
- Plan-Execute 最可控但最不灵活（无法应对意外）
- Reflexion 在两者之间但成本最高

## 主流优化方向

1. **自适应变体选择**：根据任务难度自动选择循环变体
2. **混合变体**：高层次用 Plan-Execute，子任务用 ReAct
3. **增强 ReAct**：ReAct + 隐式规划（在 Thought 中加入计划成分）
4. **轻量 Reflexion**：仅在关键节点反思，而非每轮都反思

## 三种变体的对比

| 维度 | ReAct | Reflexion | Plan-Execute |
|------|-------|-----------|-------------|
| 规划 | 隐式（在思考中） | 隐式 + 事后反思 | 显式（独立阶段） |
| 错误恢复 | 实时纠偏 | 反思后整轮重试 | 失败时重规划 |
| Token 消耗 | 中 | 高 | 中-高 |
| 适用任务 | 探索性 | 复杂推理 | 结构化任务 |
| 实现复杂度 | 低 | 中 | 中-高 |
| 可预测性 | 低 | 中 | 高 |

## 实现挑战

1. **变体间切换**：运行时动态切换变体的代价高
2. **Reflexion 深度控制**：反思太浅无效果，太深浪费 Token
3. **Plan-Execute 适应性**：计划赶不上变化，需要处理不可预见的中间状态
4. **混合架构的状态管理**：不同变体的状态模式不同，整合复杂

## 能力边界

- ReAct 无法处理需要全局优化的任务（只见树木不见森林）
- Reflexion 不适用于实时交互场景（延迟太高）
- Plan-Execute 无法应对目标模糊或探索性任务

## 核心优势

了解三种变体让你能 **因任务制宜**——选择正确的循环模式可以让 Agent 的效率提升数倍，成本降低数倍。

## 工程优化

1. 默认使用 ReAct（最简单、经过广泛验证）
2. 在 ReAct 基础上按需叠加反思或规划组件
3. 用 LangGraph 等图编排框架实现自定义循环控制
4. 记录不同变体在真实任务上的性能指标，数据驱动优化

## 场景判断

| 场景 | 推荐变体 | 原因 |
|------|---------|------|
| 信息查询 | ReAct | 快速，不需要全局规划 |
| 数学/逻辑推理 | Reflexion | 需要试错和自我纠正 |
| 数据分析流水线 | Plan-Execute | 步骤可预先确定 |
| 未知领域探索 | ReAct | 灵活性最重要 |
| 代码生成 | Plan-Execute | 先设计架构再实现 |
| 复杂问答 | Reflexion | 需要反复验证答案 |
