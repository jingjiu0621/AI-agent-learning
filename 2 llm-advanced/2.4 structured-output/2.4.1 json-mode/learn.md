# OpenAI / Claude JSON Mode 原理

## 简单介绍

JSON Mode 是 LLM API 提供的一种输出模式，通过 API 参数指示模型输出合法的 JSON 格式内容。这是最简单的结构化输出方案——不改变模型本身，只是在 Prompt 层面引导模型输出 JSON。

## 基本原理

### OpenAI JSON Mode

```python
response = openai.chat.completions.create(
    model="gpt-4o",
    response_format={"type": "json_object"},  # 开启 JSON Mode
    messages=[
        {"role": "system", "content": "你需要输出 JSON 格式。"},
        {"role": "user", "content": "提取文章信息：标题、作者、日期"}
    ]
)
# 输出保证是合法 JSON
# {"title": "...", "author": "...", "date": "..."}
```

### Claude JSON Mode

Claude 没有显式的 JSON Mode 参数，但通过 system prompt 引导效果类似：

```python
response = anthropic.messages.create(
    model="claude-3-5-sonnet-20241022",
    system="始终以 JSON 格式输出。不要包含 markdown 格式标记。",
    messages=[{"role": "user", "content": "提取文章信息：标题、作者、日期"}]
)
```

## 实现机制

JSON Mode 在 API 层面做了两件事：
1. **System Prompt 增强**：自动在 system prompt 尾部添加"请输出 JSON 格式"的指令
2. **Token 级别引导**：在模型输出的初始阶段，强制选择 JSON 相关的 Token 路径

它不是 100% 保证输出合法 JSON——只是概率上大幅提高。

## 可靠性对比

| 方法 | JSON 合法性 | Schema 符合度 | 适用场景 |
|------|------------|--------------|----------|
| 纯 Prompt 指令 | ~70-80% | ~50-60% | 原型开发 |
| OpenAI JSON Mode | ~95% | ~70-80% | 通用 |
| Function Calling | ~99%+ | ~95% | 工具调用 |
| Constrained Decoding | 100% | 100% | 生产系统 |

## 优势与局限

**优势**：
- 零额外配置，开箱即用
- 不需要定义复杂的 Schema
- 兼容流式输出

**局限**：
- 只保证 JSON 合法性（括号匹配），不保证 Schema 合规
- 字段名可能意外变化（"title" → "Title" → "article_title"）
- 仍然可能输出模型幻觉的数据
- 嵌套 JSON 容易出错

## 工程启示

- JSON Mode 适合"快速原型"和"内部工具"，不适合需要严格 Schema 的生产场景
- 如果输出的 JSON 由代码消费，始终在外层做 JSON Schema 校验
- JSON Mode + Function Calling 可以组合使用——FC 保证 Schema，JSON Mode 保证合法性
- 输出后处理时，使用 Pydantic 或 JSON Schema 校验库做二次验证
