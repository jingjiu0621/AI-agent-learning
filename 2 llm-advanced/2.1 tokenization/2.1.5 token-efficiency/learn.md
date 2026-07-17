# Token 效率优化：减少 Token 数的工程技巧

## 简单介绍

Token 效率优化是指在保留核心信息的前提下，尽量减少 Token 消耗的技术和策略。在 Agent 系统中，每次 API 调用都按 Token 计费，Token 效率直接影响运营成本。更重要的是，更少的 Token 意味着模型能在同样的上下文预算中处理更多信息。

## 为什么 Token 效率很重要

| 场景 | 原始 Token 数 | 优化后 Token 数 | 成本节省 |
|------|--------------|----------------|---------|
| 复杂的 System Prompt | 2000 | 1200 | 40% |
| 工具定义注入 | 5000 (30 个工具) | 3000 (精简描述) | 40% |
| 历史对话摘要 | 4000 | 800 | 80% |
| 长文档处理 | 80000 | 20000 | 75% |

## 优化策略

### 1. Prompt 精简

```python
# ❌ 冗余的 Prompt
system_prompt = """
你是一个非常有用的 AI 助手，可以帮助用户解决各种问题。
你的名字是"小助手"。你的创造者是 OpenAI。
当用户提问时，你需要仔细分析问题，然后给出详细的回答。
回答应该包括以下部分：首先理解问题，然后思考解决方案，
最后给出答案。答案应该清晰、准确、有用。
"""

# ✅ 精简后的 Prompt
system_prompt = """
你是助手"小助手"，由 OpenAI 创建。
职责：分析用户问题 → 思考解决方案 → 给出答案。
回答要求：清晰、准确、有用。
"""
# Token 节省约 40-50%
```

### 2. 工具描述优化

```json
// ❌ 冗余描述
{
  "name": "get_weather",
  "description": "这个工具用于获取指定城市的天气数据。它调用外部天气 API 来获取最新的气象信息，包括温度、湿度、风速、天气状况等。返回的数据是 JSON 格式。"
}

// ✅ 精简描述
{
  "name": "get_weather",
  "description": "获取指定城市的天气（温度、湿度、风力、天气状况）"
}
// Token 节省约 60%
```

### 3. 结果压缩

```python
def compress_tool_result(result: dict) -> str:
    """压缩工具执行结果"""
    if len(result) > 1000:
        # 只保留关键字段
        return json.dumps({
            "summary": result.get("summary", ""),
            "count": result.get("total", 0),
            "top_3": result.get("items", [])[:3]
        }, ensure_ascii=False) + f"\n[共 {result.get('total', 0)} 条结果，仅显示前 3 条]"
    return json.dumps(result, ensure_ascii=False)
```

### 4. 结构化数据优化

```python
# ❌ 大段 JSON 原始返回（消耗大量 Token）
return json.dumps(all_data)

# ✅ 精简后的返回
return json.dumps({
    "count": len(data),
    "items": [{"id": d["id"], "name": d["name"]} for d in data[:5]],
    "has_more": len(data) > 5
})
```

### 5. 历史对话压缩

```python
class HistoryCompressor:
    def compress(self, history: list, budget: int) -> str:
        """在 Token 预算内压缩历史"""
        # 只保留最近的 N 轮完整对话
        recent = history[-3:]  # 保留最近 3 轮完整
        # 之前的对话用摘要替代
        if len(history) > 3:
            earlier = self.summarize(history[:-3])
            return f"[历史摘要]: {earlier}\n" + self.format_recent(recent)
        return self.format_recent(history)
```

## 不同场景的 Token 密度

| 内容类型 | Token/字符比 | 说明 |
|---------|-------------|------|
| 英文文本 | 1:4 | 每 1 Token ≈ 4 个英文字符 |
| 英文单词 | 1:0.75 | 每 1 Token ≈ 0.75 个单词 |
| 中文文本 | 1:1.5-2 | 每 1 Token ≈ 1.5-2 个汉字 |
| JSON 数据 | 1:3-5 | JSON 键名和结构符号消耗 Token |
| 代码 | 1:2-4 | 代码中的空格和换行也是 Token |
| Base64 编码 | 1:1 | 编码数据 Token 消耗极高 |

## 核心挑战

1. **精度损失**：过度压缩可能丢失关键信息
2. **摘要的准确性**：LLM 生成的历史摘要可能遗漏重要细节
3. **压缩 vs 上下文窗口**：压缩越多，模型看到的原始信息越少

## 最佳实践

- 优先精简冗余描述和指令，而非压缩关键数据
- 工具描述精简到 "核心功能 + 触发关键词" 即可
- 历史对话使用"最近 N 轮完整 + 之前摘要"的分层策略
- 监控 Token 消耗趋势，识别消耗异常的工具或 Prompt
- 结构化输出时避免嵌套过深，展平数据可减少 Token
