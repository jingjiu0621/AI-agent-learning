# 3.4 system-design — System Prompt 设计模式

## 本节概述

System Prompt（系统提示）是 AI Agent 开发中最关键的"元指令"——它定义了 Agent 的身份、行为边界、输出规范和整体行为基调。与 User Prompt（用户提示）不同，System Prompt 由开发者编写、在对话开始时注入，并持续作用于 Agent 的整个生命周期。本节系统性地覆盖 System Prompt 的六大核心设计模式：角色设定、指令层级、约束列举、格式规范、少样本嵌入和安全边界。

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|---------|------|
| 3.4.1 persona-pattern | 如何让 Agent 扮演一个可信、稳定的角色？身份设定如何影响行为边界？ | ★★★☆☆ |
| 3.4.2 instruction-hierarchy | 系统级指令、用户级请求、工具级约束如何在冲突时裁决优先级？ | ★★★★☆ |
| 3.4.3 constraint-listing | 如何清晰定义 Agent 的"不能做"清单？约束过多会抑制能力吗？ | ★★★☆☆ |
| 3.4.4 format-specification | 如何设计输出结构使下游系统可靠解析？格式要求和灵活性如何平衡？ | ★★★☆☆ |
| 3.4.5 few-shot-embedding | 少样本示例放在 System Prompt 的什么位置效果最好？数量和长度如何控制？ | ★★★★☆ |
| 3.4.6 security-boundary | System Prompt 如何抵御注入攻击？拒绝策略如何设计才能既安全又不误伤？ | ★★★★★ |

## 核心思想

1. **System Prompt 是 Agent 的"宪法"**——它在 Agent 启动时即确定行为框架，所有后续交互都在这个框架内进行
2. **模式组合优于单一技巧**——角色 + 指令层级 + 约束 + 格式 + 示例 + 安全边界，组合使用效果远好于单一模式
3. **可维护性优先于完备性**——System Prompt 会持续演化，良好的组织结构比一次性"写全"更加重要
4. **安全不是附加功能**——安全边界应在 System Prompt 设计之初就纳入，而非上线后打补丁

## 关键工程启示

- System Prompt 的 token 占用是隐性成本——每增加一段指令，就减少一段可用上下文
- 指令冲突的默认裁决规则是"具体 > 通用"——越具体的指令优先级越高
- 少样本示例的边际效益递减——3~5 个精心设计的示例通常优于 10 个随意示例
- 安全边界定义本质上是"误报 vs 漏报"的权衡——过于激进会降低 Agent 可用性

## 向下关联

- → 4 tool-use：System Prompt 中的工具描述格式直接影响 Function Calling 的准确率
- → 5 agents-core：Agent 主循环的设计与 System Prompt 的指令层级结构需保持一致
- → 6 memory：System Prompt 中关于记忆存储和检索的指令决定了 Agent 的长期行为一致性
