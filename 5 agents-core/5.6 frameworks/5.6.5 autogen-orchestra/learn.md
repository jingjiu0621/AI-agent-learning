# 5.6.5 autogen-orchestra — AutoGen 源码分析

## 简单介绍

AutoGen（Microsoft）是一个以"对话驱动"为核心的多 Agent 框架。Agent 之间通过对话（Conversation）交流，Orchestrator 管理对话流程。其核心思想是：Agent 的交互本质是持续的对话式消息交换。

## 基本原理

### 核心实体

```
┌─────────────────────────────────────────┐
│  AssistantAgent                          │
│  - LLM 驱动的 Agent                     │
│  - 生成回复、发起工具调用               │
└──────────────────┬──────────────────────┘
                   │ 对话消息交换
┌──────────────────▼──────────────────────┐
│  UserProxyAgent                          │
│  - 代表人类的代理                       │
│  - 执行工具调用（在真实环境运行代码）    │
│  - 将执行结果返回给 AssistantAgent      │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  GroupChat                               │
│  - 多 Agent 群组对话                    │
│  - 内置发言顺序管理                     │
└─────────────────────────────────────────┘
```

### 执行流程（Assistant + UserProxy）

```
UserProxy → [发送任务]
    ↓
AssistantAgent → [生成回复，可能包含 tool_call]
    ↓
UserProxyAgent → [执行工具，返回结果]
    ↓
AssistantAgent → [分析结果，继续推理或给出最终回答]
    ↓
...循环直到完成...
```

### GroupChat 流程

```
GroupChat 包含: Researcher, Writer, Critic, Admin

1. Admin: "我们需要一份市场分析报告"
2. Researcher: "我搜索到了以下数据..."
3. Writer: "基于这些数据，我起草了报告..."
4. Critic: "报告缺少竞争对手分析部分"
5. Researcher: "让我补充查询..."
6. Admin: "可以了，报告完成"
```

## 背景与演进

AutoGen 的设计理念是"Agent = 对话参与者"。Agent 的智能不仅体现在独立推理，更体现在 Agent 间的对话交互中涌现的群体智能。

## 核心矛盾

**对话自由 vs 对话效率**：
- 开放式对话让 Agent 充分交流 → 可能漫无边际
- 结构化对话 → 效率高 → 限制了涌现行为

## 核心优势

相比 LangGraph 和 CrewAI：
- **更自然的 Agent 交互模型**：Agent 之间像人一样对话
- **代码执行能力**：UserProxyAgent 天然支持代码执行
- **灵活的群组管理**：GroupChat 支持动态发言

## 源码核心

| 组件 | 核心方法 | 功能 |
|------|---------|------|
| ConversableAgent | receive/send/generate_reply | 基础对话能力 |
| AssistantAgent | generate_reply | LLM 生成回复 |
| UserProxyAgent | execute_function | 工具/代码执行 |
| GroupChat | agent_selection | 管理发言顺序 |

## 工程优化

1. 设置 max_round 防止无限对话
2. 在 system_message 中明确 Agent 的职责边界
3. 使用 GroupChatManager 的 speaker_selection_method 控制发言（auto/round_robin/random）
4. 关键工具需要人工确认（UserProxyAgent 的 human_input_mode）

## 场景判断

适合 AutoGen 的场景：
- 多 Agent 辩论/讨论
- 需要代码执行的 Agent 系统
- Agent 之间需要动态交流而非固定流程
