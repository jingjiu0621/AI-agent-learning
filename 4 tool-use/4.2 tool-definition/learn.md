# 4.2 tool-definition — 工具定义规范

## 概念定义

工具定义是将现实世界的 API、函数、服务抽象为 LLM 可理解的 Schema 的过程。工具定义的质量直接决定了 LLM 能否准确选择和使用工具——这是整个工具调用链条中最关键也最容易被低估的环节。

## 核心原则

| 原则 | 说明 |
|------|------|
| 精确性优先 | Schema 定义越精确，LLM 调用错误的概率越低 |
| 描述即代码 | 工具描述（description）与代码逻辑同等重要 |
| 最少参数 | 只暴露必要的参数，减少 LLM 的决策负担 |
| 一致命名 | 命名风格统一，让 LLM 能根据名称推测用途 |

## 为什么工具定义值得深究

很多初学者认为工具定义就是写几个 JSON Schema——几分钟的事。但实际上：

1. **LLM 对工具描述极其敏感**：描述中一个词的偏差可能让 LLM 在 50 个工具中选错
2. **参数设计影响成功率**：参数过多导致幻觉参数，过少导致表达力不足
3. **命名语义影响推理**：getUserByName 和 fetch_user_profile 对 LLM 的意义完全不同

## 主流方案对比

| 方案 | 适用场景 | 复杂度 |
|------|---------|--------|
| JSON Schema | OpenAI/Claude 原生、通用 | 低 |
| OpenAPI/Swagger | REST API 已有定义 | 中 |
| TypeScript 类型 | Node.js 生态 | 低 |
| Protobuf | 高性能场景 | 高 |
| 自动生成 | 从代码/API 文档生成 | 中 |

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|---------|------|
| 4.2.1 json-schema | JSON Schema 的核心语法和最佳实践 | ★★★☆☆ |
| 4.2.2 openapi-spec | 如何将 OpenAPI 规范转化为工具定义？ | ★★★☆☆ |
| 4.2.3 description-engineering | 如何写让 LLM 准确选中的工具描述？ | ★★★★☆ |
| 4.2.4 parameter-design | 参数设计的原则和陷阱 | ★★★☆☆ |
| 4.2.5 naming-convention | 工具和参数的命名如何影响 LLM 行为？ | ★★☆☆☆ |
| 4.2.6 auto-generation | 从 API 定义自动生成工具 Schema | ★★★☆☆ |
