# 并行函数调用机制

## 简单介绍

并行函数调用（Parallel Function Calling）允许 LLM 在一次响应中同时调用多个独立工具。这大大减少了多工具场景的交互轮次，显著提升了 Agent 的效率和用户体验。

## 基本原理

核心思路很简单：**当多个工具调用之间没有依赖关系时，让 LLM 一次性输出所有工具调用声明**。

### OpenAI 的并行调用

```json
// 响应中的 tool_calls 数组
{
  "tool_calls": [
    {
      "id": "call_1",
      "function": { "name": "get_weather", "arguments": "{\"location\":\"北京\"}" }
    },
    {
      "id": "call_2",
      "function": { "name": "get_weather", "arguments": "{\"location\":\"上海\"}" }
    },
    {
      "id": "call_3",
      "function": { "name": "get_stock_price", "arguments": "{\"symbol\":\"AAPL\"}" }
    }
  ]
}
```

客户端收到后，**并行执行**所有工具调用，然后一次性将结果注入下一轮消息。

### 依赖关系的处理

并非所有工具都能并行——有些工具依赖于其他工具的输出：

```
步骤 1（并行）:
  get_user_id("张三") → "user_123"
  get_current_time() → "2024-01-15 14:00"

步骤 2（依赖于步骤 1）:
  get_user_schedule("user_123", "2024-01-15") → ["会议A", "会议B"]
```

实现依赖图调度的 DAG 方案不在 LLM 协议层面解决，而是由 Agent 框架处理。

## 历史背景

- GPT-4 (2023.06 初版)：一次只调用一个工具
- GPT-4-1106-preview (2023.11)：引入并行函数调用
- 这是 LLM 提供商之间的关键差异化能力，后续所有主流模型都跟进

## 核心优势

| 维度 | 串行调用 | 并行调用 |
|------|---------|----------|
| 交互轮次 | N 步调用 = N 轮交互 | 1 轮即可 | 
| 总延迟 | 各步延迟累加 | 等于最慢的那个工具 |
| 用户体验 | 等待时间长 | 响应更快 |
| 实现复杂度 | 简单 | 需要并发执行逻辑 |
| Token 消耗 | 每步有中间推理 | 一次推理即可 |

## 工程实现

```python
import asyncio

async def execute_parallel(tool_calls, tool_registry):
    """并行执行多个工具调用"""
    async def execute_one(tc):
        tool = tool_registry[tc.function.name]
        args = json.loads(tc.function.arguments)
        try:
            result = await tool["handler"](**args)
            return tc.id, result, None
        except Exception as e:
            return tc.id, None, str(e)
    
    tasks = [execute_one(tc) for tc in tool_calls]
    return await asyncio.gather(*tasks)
```

## 限制

1. **依赖感知的局限**：LLM 并不总能准确判断工具间是否有依赖关系——它可能把所有工具都并行化，导致下游工具拿到不完整的数据
2. **执行顺序不确定性**：如果多个工具写入同一个资源，并行执行会产生竞态条件
3. **错误级联复杂**：五个工具中的某一个失败了，其他四个的成功结果是否还有意义？

## 优化方向

- **混合调度**：LLM 输出全部工具调用后，由 Agent 框架自动识别依赖关系，构建 DAG 分步执行
- **条件并行**：某些场景下串行更可靠（写入操作），某些场景下并行是必须的（批量查询）
