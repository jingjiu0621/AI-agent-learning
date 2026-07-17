# AutoGen：对话驱动多 Agent 与代码执行

## 简单介绍

AutoGen 是微软推出的多 Agent 框架，核心设计是**对话驱动**——Agent 之间通过对话来完成任务。与 CrewAI 的"分配任务给角色"不同，AutoGen 的 Agent 通过群聊/双人对话来讨论、协商和解决问题。同时，AutoGen 深度集成**代码执行沙箱**。

## 核心设计

```
核心概念:
- AssistantAgent: LLM 驱动的 Agent（推理+生成）
- UserProxyAgent: 代表人类（可以执行代码）
- GroupChat: 多 Agent 群聊
- GroupChatManager: 管理群聊流程

核心能力:
- 代码执行: Agent 可以生成代码并在沙箱中执行
- 对话管理: 谁发言？何时停止？如何总结？
- 工具注册: 为 Agent 注册自定义工具
```

## 核心优势

| 优势 | 说明 |
|------|------|
| 对话式协作 | Agent 通过自然语言讨论解决问题 |
| 代码执行 | 内置的 Python 代码执行沙箱 |
| 灵活的发言策略 | 顺序、轮询、自动选择发言人 |
| 微软生态 | 与 Azure AI、Semantic Kernel 深度集成 |

## 主要不足

| 不足 | 说明 |
|------|------|
| 对话开销 | Agent 间对话消耗大量 Token |
| 收敛困难 | 多 Agent 可能一直讨论不达成结论 |
| 学习曲线 | 概念复杂（GroupChat 管理、发言策略） |
| 单一 Agent 场景弱 | 主要为多 Agent 设计 |

## 适用场景

- **编程任务**：生成代码 → 执行 → 根据结果修改的循环
- **多 Agent 辩论**：不同视角讨论一个复杂问题
- **人机协作**：UserProxyAgent 让人类参与决策

## 不适合

- 单 Agent 任务
- Token 敏感的批量处理场景
