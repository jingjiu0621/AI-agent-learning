# 多轮函数调用与状态跟踪

## 简单介绍

多轮函数调用指的是 Agent 在多次 LLM 推理-工具执行循环中完成一个复杂任务的过程。与单次调用不同，多轮场景需要跟踪对话状态、维护工具调用之间的依赖关系，并在多次交互中保持上下文一致性。

## 核心挑战

### 1. 状态累积

```
第 1 轮: LLM思考 → 调用 search_users("张三") → 返回 user_123
第 2 轮: LLM思考 → 调用 get_schedule(user_123) → 返回 ["会议A","会议B"]  
第 3 轮: LLM思考 → 调用 book_room(会议室, 时间) → ...
```

每一轮的工具结果都会追加到对话历史中。**关键是**：第 2 轮的 LLM 需要记得第 1 轮找到了 user_123，并正确使用它。

### 2. 上下文窗口耗尽

每轮交互都会增加 Token 消耗。多轮复杂任务可能耗尽上下文窗口：

| 轮次 | Token 增加 | 累计 Token | 状态 |
|------|-----------|-----------|------|
| 1 | ~500 | ~500 | 正常 |
| 5 | ~2500 | ~5000 | 正常 |
| 10 | ~2500 | ~15000 | 可能需要压缩 |
| 20 | ~2500 | ~40000+ | 接近限制 |

### 3. 历史压缩与信息丢失

当需要对历史消息做截断或摘要时，可能丢失关键状态信息：

```
摘要：用户询问了张三的日程，找到了 user_123，查看了排期...
丢失细节：book_room 需要的具体参数（房间号、时间段）
```

## 解决方案

### 方案 1：显式状态存储

不依赖 LLM 上下文来维护状态，而是在外部存储中追踪：

```python
class AgentState:
    def __init__(self):
        self.data = {}  # 显式存储关键状态
    
    def set(self, key, value, source_tool=None):
        self.data[key] = {"value": value, "source": source_tool}
    
    def get(self, key):
        return self.data.get(key)
```

### 方案 2：消息窗口 + 关键信息摘要

```
对话历史:
  [窗口: 最近 5 轮完整消息]
  [摘要: 前面 15 轮的摘要]
  [关键状态: user_id=user_123, 已确认可用时间="14:00-16:00"]
```

### 方案 3：结构化 Memory

将中间结果写入长期记忆（向量存储或结构化数据），Agent 在需要时主动检索。

## 历史背景

早期的 Agent 实现（ReAct、AutoGPT）采用"全部塞入上下文"策略，用完整的历史消息来追踪状态。这是一种简单但粗暴的方式，在任务变长时迅速失效。

## 工程示例

```python
class MultiTurnAgent:
    def __init__(self, max_history=10):
        self.history = []
        self.state_store = {}  # 显式状态
        self.max_history = max_history
    
    async def step(self, message):
        self.history.append(message)
        
        # 控制上下文预算
        if len(self.history) > self.max_history:
            # 摘要最早的消息
            summary = await self.summarize(self.history[:-self.max_history])
            self.history = [{"role": "system", "content": f"历史摘要: {summary}"}] + self.history[-self.max_history:]
        
        response = await llm.chat(self.history, tools=...)
        
        if response.tool_calls:
            for tc in response.tool_calls:
                tool_result = await execute_tool(tc)
                # 自动提取关键状态
                self.state_store.update(extract_key_values(tc, tool_result))
                self.history.append(tool_result)
        
        return response
```

## 最佳实践

1. **分离推理和存储**：LLM 负责推理决策，外部存储维护状态
2. **关键信息冗余**：在历史摘要和状态存储中都保留关键信息
3. **定期状态检查**：定期让 LLM 审视当前存储的状态是否完整准确
4. **明确过期策略**：哪些状态长期有效、哪些状态单次有效、哪些状态随会话过期
