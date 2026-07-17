# ReAct Agent：推理与行动交织

## 简单介绍

ReAct（Reasoning + Acting）是目前最广泛使用的 Agent 模式，由 Yao 等于 2022 年提出。核心思想是将推理（Reasoning）和行动（Acting）交织在一起——Agent 每推理一步、执行一个行动、观察结果、再推理下一步。这种模式融合了 CoT 的显式推理和工具调用的能力。

## 基本原理

### ReAct 循环

```
Thought: 我需要查找天气数据 → 我应该调用天气 API
Action: get_weather(location="北京")
Observation: {"temp": 28, "condition": "晴"}
Thought: 北京今天 28 度，晴天。用户穿短袖就行
Action: 生成回复
```

### ReAct 的四个步骤

| 步骤 | 说明 | 示例 |
|------|------|------|
| Thought | LLM 分析当前状态，决定要做什么 | "我需要查北京的天气" |
| Action | 选择一个工具并生成参数 | get_weather(city="北京") |
| Observation | 工具返回的结果 | {"temp": 28, "condition": "晴"} |
| ...循环... | 基于 Observation 再次 Thought | 直到任务完成 |

### 终止条件
- Agent 认为任务已完成 → 生成最终回复
- 达到最大步数限制
- 检测到循环
- 用户主动中断

## 核心优势

| 维度 | ReAct | CoT（纯推理） |
|------|-------|--------------|
| 信息获取 | 工具调用获取外部信息 | 仅依赖模型知识 |
| 行动能力 | 可执行操作（API/代码） | 仅文本回复 |
| 纠错能力 | 观察结果可纠正错误推理 | 推理错误无法自纠 |
| Token 消耗 | 更高 | 中等 |
| 实现复杂度 | 中（需工具系统） | 低 |

## 限制

1. **循环陷阱**：Agent 可能在 Thought-Action-Observation 中循环，无法推进
2. **Token 消耗大**：每一步的 Thought + Observation 都消耗 Token
3. **效率问题**：每步只做一个行动，复杂的任务需要很多步
4. **早期错误放大**：错误推理会导致后续所有步骤偏离方向

## 工程实现

```
system: 你是一个可以调用工具的助手。
        每次回复格式：
        Thought: 你的推理
        Action: 工具名(参数)
        Observation: (工具返回结果，由系统填充)
        ...循环直到可以回答...

user: 北京的天气怎么样？

assistant: 
Thought: 用户想知道北京的天气，我需要调用天气查询工具。
Action: get_weather("北京")

(系统注入 Observation)
Observation: {"temp": 28, "condition": "晴", "humidity": 40%}

assistant:
Thought: 北京目前 28 度，晴天，湿度 40%。天气很好。
Action: finish("北京今天 28 度，晴朗，湿度 40%，适合外出。")
```

## 优化方向

1. **并行 Action**：一次生成多个独立工具调用（Parallel Function Calling）
2. **结构化 Thought**：用 JSON Schema 约束 Thought 格式，提升稳定性
3. **步数上限 + 进度检测**：防止无限循环
4. **Observation 压缩**：对大型返回结果自动摘要
