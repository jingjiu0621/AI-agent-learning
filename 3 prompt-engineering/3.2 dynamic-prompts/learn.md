# 3.2 dynamic-prompts — 动态 Prompt 组装与管理

## 本节概述

动态提示工程（Dynamic Prompt Engineering）关注如何编程式地生成、组装和管理 Prompt。在 AI Agent 开发中，单条静态 Prompt 无法应对千变万化的用户请求和上下文状态。本模块深入讲解模板引擎、上下文注入、链式组装、条件分支、版本管理和多语言切换六大核心技术，帮助开发者构建灵活、可维护的 Prompt 生产系统。

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|---------|------|
| 3.2.1 Template Engine | 如何用模板引擎结构化生成 Prompt？ | ★★★☆☆ |
| 3.2.2 Context Injection | 如何将运行时上下文精准注入 Prompt？ | ★★★☆☆ |
| 3.2.3 Prompt Chaining | 如何将复杂任务拆解为多步 Prompt 管道？ | ★★★★☆ |
| 3.2.4 Conditional Prompts | 如何根据条件动态选择不同的 Prompt 策略？ | ★★★☆☆ |
| 3.2.5 Version Control | 如何管理 Prompt 的版本、变更与回滚？ | ★★☆☆☆ |
| 3.2.6 Multi-lang Prompts | 如何实现多语言、多角色的 Prompt 切换？ | ★★☆☆☆ |

## 核心思想

1. **Prompt as Code** — Prompt 不再是静态文本，而是由代码生成的动态产物，需纳入软件工程管理流程
2. **关注点分离** — Prompt 的内容（what to say）和逻辑（when/how to say）应当分离，模板负责内容，代码负责逻辑
3. **渐进式组装** — 复杂 Prompt 由多个简单片段按需组合而成，而非一次性写出完整内容
4. **上下文感知** — 动态 Prompt 的核心价值在于根据当前上下文（用户意图、对话历史、工具结果）自动调整输出

## 关键工程启示

- 动态 Prompt 系统是 Agent 主循环的"翻译层"——将内部状态转换为 LLM 能理解的自然语言指令
- 模板引擎是基础设施，上下文注入是数据流，链式组装是控制流——三者的组合决定了 Agent 的智能上限
- 动态不等于复杂——好的动态 Prompt 系统应当在增加灵活性的同时，保持各模块的可测试性和可观测性

## 向下关联

- → 3.3 prompt-optimization：动态组装是 Prompt 优化的前提——没有结构化的生成，就无法系统化地改进
- → 3.4 system-design：System Prompt 设计模式需要动态组装技术来支持角色切换和上下文感知
- → 5 agents-core：Agent 主循环中的 Prompt 管理就是本节技术的工程实践
