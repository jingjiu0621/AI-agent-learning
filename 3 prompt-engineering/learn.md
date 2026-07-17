# 3 prompt-engineering — 提示工程高级技术与系统设计

## 模块概述

提示工程（Prompt Engineering）是 LLM Agent 开发中的核心技能。与简单"写 Prompt"不同，**高级提示工程**涉及系统化的 Prompt 设计、动态组装、自动优化和生产级管理。随着 LLM 能力提升，Prompt 已从"输入文本"演变为 Agent 系统的"编程接口"——你是通过自然语言指令编程，而非传统代码。

## 内容结构

| 章节 | 主题 | 核心问题 | 难度 |
|------|------|---------|------|
| 3.1 advanced-techniques | 高级推理提示技术 | 如何设计 Prompt 引导 LLM 进行复杂推理？ | ★★★☆☆ |
| 3.2 dynamic-prompts | 动态 Prompt 组装与管理 | 如何编程式地生成和管理大量 Prompt？ | ★★★☆☆ |
| 3.3 prompt-optimization | Prompt 优化 | 如何系统化地改进和优化 Prompt？ | ★★★★☆ |
| 3.4 system-design | System Prompt 设计模式 | 如何设计生产级的 System Prompt？ | ★★★★☆ |

## 核心思想

1. **Prompt 就是代码**——需要版本管理、测试、优化和持续维护，不能"写一次用到老"
2. **推理模式 > 内容编写**——引导 LLM 的思考过程（如何想）比告诉它该说什么更重要
3. **分层设计**——System Prompt、User Prompt、Tool Prompt 各有职责，需分层设计而非混在一起
4. **动态 > 静态**——好的 Prompt 系统能根据上下文动态调整，而非千篇一律的模板

## 关键工程启示

- Prompt 是 Agent 系统的**性能上限瓶颈**——模型能力固定后，Prompt 质量决定一切
- 从"写 Prompt"到"设计 Prompt 系统"——单条 Prompt 优化有天花板，系统设计才有突破
- 可评估才能可优化——不建立评估体系，Prompt 优化只能是碰运气

## 前置要求

- Module 1 foundation 完成
- 有 LLM API 使用经验
- 理解 CoT、ReAct 等基本推理模式

## 向下关联

- → 4 tool-use：Prompt 工程直接影响工具调用的准确性和效率
- → 5 agents-core：Agent 主循环本质上是一个"Prompt 运行引擎"
- → 7 planning：高级推理提示技术是 Agent 规划能力的核心实现手段
