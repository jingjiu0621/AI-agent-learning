# 工具使用示例动态选取

## 简单介绍

Few-shot 示例（在 Prompt 中给 LLM 提供工具使用的例子）能显著提升工具选择的准确性。但当工具有很多时，不可能为每个工具都提供示例（上下文窗口限制）。动态示例选取的策略是：从工具的使用日志中检索与当前请求最相关的使用案例，只注入这些精选示例。

## 基本原理

```
工具使用日志库
  [用户: "北京天气"] → 调用了 get_weather
  [用户: "发邮件给张三"] → 调用了 send_email
  [用户: "上海天气"] → 调用了 get_weather
      ↓ 语义匹配
当前请求: "杭州热吗？"
      ↓ 检索
Top-2 示例:
  - "北京天气" → get_weather(location="北京")
  - "上海天气" → get_weather(location="上海")
      ↓ 注入 Prompt
```

## 示例的作用

```python
# 没有示例的 Prompt
system: "你可以使用 get_weather 工具"

# 有示例的 Prompt
system: """
你可以使用以下工具：
- get_weather: 获取天气

示例：
用户: 北京天气怎么样？
助手: 我来查一下
工具调用: get_weather(location="北京")
工具返回: {"temp": 28, "condition": "晴"}
助手: 北京今天 28 度，晴天。

当前请求：杭州热吗？
"""
```

**影响**：好的示例可以让工具选择准确率提升 10-20%。

## 示例库构建

```python
class ExampleLibrary:
    def __init__(self):
        self.examples = []
        self.embeddings = None
    
    def add_example(self, user_query: str, tool_calls: list, context: dict = None):
        """添加一个使用示例"""
        self.examples.append({
            "query": user_query,
            "tool_calls": tool_calls,
            "context": context or {},
            "embedding": self._embed(user_query)
        })
    
    def search(self, query: str, top_k: int = 3) -> list:
        """检索与当前请求最相关的示例"""
        query_emb = self._embed(query)
        scored = []
        for ex in self.examples:
            sim = cosine_similarity(query_emb, ex["embedding"])
            scored.append((sim, ex))
        scored.sort(reverse=True)
        return [ex for _, ex in scored[:top_k]]
```

## 示例选取策略

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| 最近使用 | 取最新的成功案例 | 通用 |
| 语义相似 | 与当前请求最相似的 | 效果最好 |
| 多样性采样 | 覆盖不同工具 | 新用户/冷启动 |
| 基于错误 | 曾经选错的示例作为反面教材 | 高频错误场景 |

## 示例的格式

```json
{
  "query": "杭州热吗？",
  "tool_calls": [
    {
      "tool": "get_weather",
      "args": {"location": "杭州"},
      "result": {"temp": 35, "condition": "晴"}
    }
  ],
  "success": true,
  "tokens_used": 150
}
```

## 反面示例（Negative Examples）

有时"不应该怎么做"的例子比"应该怎么做"更有价值：

```
添加反面示例到 Prompt：

❌ 错误示例：
用户: 删除用户 12345
助手: 直接调用 delete_user("12345")
→ 结果：出错，因为没有先确认用户身份

✅ 正确做法：
用户: 删除用户 12345
助手: 让我先确认一下用户身份
工具调用: get_user("12345")
...
```

## 核心挑战

1. **示例库维护**：工具更新后，旧示例可能失效——需要定期清理
2. **Token 预算**：每个示例消耗 50-200 Token，3 个示例就是 150-600 Token——需要在效果和成本间权衡
3. **示例代表性**：低频工具的示例很少，检索时可能找不到合适的例子
4. **示例时效性**：半年前的例子可能已经不再适用（API 变了）

## 最佳实践

- 为目标工具数量的 10%-20% 提供示例（20 个工具则准备 2-4 个示例）
- 定期（每周）清理和更新示例库
- 对高频错误的情况，专门准备"纠正示例"
- 示例的 Tokens 消耗计入总上下文预算管理
