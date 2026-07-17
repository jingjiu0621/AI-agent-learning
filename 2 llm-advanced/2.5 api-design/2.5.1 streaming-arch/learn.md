# 流式架构：SSE / WebSocket / Chunked Transfer

## 简单介绍

流式架构（Streaming）允许 LLM 在生成 Token 的同时将结果逐步推送给客户端，而不是等待所有 Token 都生成完再一次性返回。这对 Agent 系统至关重要——用户可以实时看到 LLM 的推理过程和工具调用声明。主要的流式传输技术有 SSE、WebSocket 和 Chunked Transfer。

## 为什么需要流式

| 维度 | 非流式（等待完整响应） | 流式 |
|------|---------------------|------|
| 用户体验 | 需要等待数秒才能看到第一个字 | 几百毫秒就开始显示 |
| 首 Token 延迟 | 等于完整生成时间 | 极短 |
| 网络传输 | 一次性大请求 | 持续的小数据包 |
| 中间状态 | 无法暴露 | 可以逐步暴露思考、工具调用 |

核心原因：**人类不能忍受屏幕上没有反馈的空白等待**。即使总处理时间相同，流式体验也远优于非流式。

## 三种流式方案

### 1. SSE（Server-Sent Events）

最常用的 LLM 流式方案。基于 HTTP，服务器单向推送事件。

```
客户端 → HTTP 请求 → 服务器
                    ↓ SSE 流
客户端 ← event: token\ndata: "今天"  ←
客户端 ← event: token\ndata: "天气"  ←
客户端 ← event: token\ndata: "真好"  ←
客户端 ← event: done\ndata: [DONE]  ←
```

```python
# OpenAI Python SDK 的流式使用
stream = client.chat.completions.create(
    model="gpt-4",
    messages=[...],
    stream=True  # 启用流式
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
    if chunk.choices[0].delta.tool_calls:
        # 流式工具调用
        handle_tool_call_delta(chunk)
```

**优点**：基于标准 HTTP，实现简单，防火墙友好
**缺点**：单向（只能服务器→客户端），不适合双向通信

### 2. WebSocket

全双工通信协议，支持客户端和服务器双向实时通信。

```
客户端 ←→ WebSocket 连接 ←→ 服务器
  客户端 → {"type": "chat", "content": "北京天气"}
  服务器 → {"type": "token", "content": "今天"}
  服务器 → {"type": "token", "content": "天气"}
  服务器 → {"type": "tool_call", "tool": "get_weather", "args": {...}}
  客户端 → {"type": "tool_result", "result": {...}}
  服务器 → {"type": "token", "content": "北京"}
  服务器 → {"type": "done"}
```

**优点**：双向通信，支持中断（客户端中断生成）
**缺点**：比 SSE 复杂的连接管理

### 3. Chunked Transfer Encoding

HTTP/1.1 标准的分块传输——服务器可以逐步发送响应体而不告知总长度。

```
HTTP 响应头: Transfer-Encoding: chunked
  响应体:
    chunk 1: "今天"
    chunk 2: "天气"
    chunk 3: "真好"
    ...
```

**优点**：完全兼容 HTTP，不需要特殊协议
**缺点**：需要客户端自行解析分块

## LLM 流式输出的事件类型

以 OpenAI 的流式事件为例：

| 事件 | 说明 | 触发时机 |
|------|------|----------|
| `content_block_delta` | 文本 Token | 每生成一个 Token |
| `content_block_stop` | 文本块结束 | 完整段落/思考结束 |
| `tool_call_delta` | 工具参数片段 | 生成工具调用参数时 |
| `tool_call_stop` | 工具调用完成 | 工具参数完整时 |
| `message_stop` | 消息完整结束 | 全部生成完毕 |

## 流式架构设计

```python
class StreamingAgent:
    """支持流式的 Agent 框架"""
    
    async def chat(self, message: str):
        # 1. 启动 SSE 流
        async with self.start_stream() as stream:
            
            # 2. LLM 流式生成响应
            async for chunk in self.llm.stream(messages):
                
                if chunk.type == "text":
                    # 3. 直接推送文本 Token 给客户端
                    await stream.send(chunk.data)
                    
                elif chunk.type == "tool_call":
                    # 4. 客户端收到工具调用声明
                    await stream.send({
                        "type": "tool_use",
                        "tool": chunk.name,
                        "args": chunk.args
                    })
                    
                    # 5. 等待客户端返回执行结果
                    result = await stream.receive()
                    # 将结果注入下一轮推理
                    messages.append(result)
```

## SSE vs WebSocket 的选择

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 纯输出流（如 Chat） | SSE | 简单，够用 |
| 需要客户端中断 | WebSocket | 双向通信 |
| Agent 工具调用流 | WebSocket | 需要客户端反馈结果 |
| 防火墙严苛环境 | SSE | 只走标准 HTTP |
| 高并发场景 | SSE | 无连接管理开销 |

## 工程启示

- 对于纯文本流式输出，SSE 是最简单的选择
- 对于 Agent 的工具流式场景，推荐 WebSocket——因为 Agent 需要"思考→工具执行→继续思考"的交互循环
- 流式场景下的"首个 Token 时间"（TTFT）是核心指标——优化 Prompt 长度 | 硬件带宽可以减少 TTFT
- 部分平台（OpenAI, Anthropic）的流式返回包含多种事件类型——解析器需要正确处理所有事件
- Agent 系统的流式应该暴露中间思考过程（Thought），这可以给用户提供更好的交互体验
