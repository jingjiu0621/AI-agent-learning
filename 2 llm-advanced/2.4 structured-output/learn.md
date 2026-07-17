# 2.4 structured-output — 结构化输出

## 概念定义

结构化输出是指让 LLM 生成符合特定格式（JSON、XML、类型约束等）的内容，而非自由文本。对于 Agent 系统，结构化输出几乎是必备能力——工具调用参数、推理步骤、中间状态都需要结构化表达。

## 核心矛盾

| 维度 | 自由文本 | 结构化输出 |
|------|---------|-----------|
| 人类可读性 | 高 | 中 |
| 机器可解析性 | 低 | 高 |
| LLM 友好度 | 高（原生） | 中（需要引导） |
| 可靠性 | 不需要约束 | 需要约束保证 |

核心矛盾：**LLM 天生善于生成自然语言，但 Agent 系统需要机器可解析的结构化数据**。

## 方案演进

| 方案 | 可靠性 | 复杂度 | 适用 |
|------|--------|--------|------|
| Prompt 指令 + 正则解析 | 低 | 低 | 快速原型 |
| JSON Mode（API 参数） | 中 | 低 | 通用 |
| Function Calling | 高 | 中 | 工具调用 |
| Constrained Decoding | 最高 | 高 | 生产系统 |
| JSON Schema 约束输出 | 高 | 中 | 复杂 Schema |

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|---------|------|
| 2.4.1 json-mode | OpenAI/Claude 的 JSON Mode 原理和用法 | ★★★☆☆ |
| 2.4.2 constrained-decoding | Grammar-constrained 解码如何保证 100% 合法输出？ | ★★★★☆ |
| 2.4.3 json-schema-enforce | 如何用 JSON Schema 强制输出结构？ | ★★★☆☆ |
| 2.4.4 tool-call-structure | Function Calling 的结构化输出本质 | ★★★☆☆ |
| 2.4.5 partial-json | 流式场景下部分 JSON 如何解析？ | ★★★★☆ |
| 2.4.6 open-source-libs | Outlines, JsonFormer, lm-format-enforcer 等开源库对比 | ★★★☆☆ |
