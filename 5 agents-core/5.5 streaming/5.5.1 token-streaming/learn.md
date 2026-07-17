# 5.5.1 token-streaming — Token 流推送

## 简单介绍

Token Streaming 是 Agent 将 LLM 生成的文本逐 Token 推送给客户端的技术。这是最基础的流式能力，解决"用户等待时看到空白屏幕"的体验问题。

## 基本原理

### SSE 推送示例

```
// 服务端：逐 Token 推送
GET /chat/completions (stream=true)

data: {"type": "token", "content": "我"}
data: {"type": "token", "content": "正在"}
data: {"type": "token", "content": "查"}
data: {"type": "token", "content": "询"}
data: {"type": "token", "content": "天"}
data: {"type": "token", "content": "气"}
data: {"type": "done", "content": ""}
```

### Agent 流 vs 简单 LLM 流

| 类型 | 推送内容 | 示例 |
|------|---------|------|
| 简单 LLM 流 | 仅文本 Token | "今天天气很好" |
| Agent 流 | 文本 + 工具调用 + 状态事件 | "我在查天气..." + tool_call + "结果是25°C" |

## 背景与演进

早期 LLM 应用使用非流式请求，用户等待 5-10 秒后一次性看到结果。SSE 流式成为标准后，用户获得感时从 10 秒降至 200ms（首个 Token 时间）。

## 核心矛盾

**实时性 vs 输出质量**：
- 逐字推送 → 用户感知快 → 断句不完整，体验可能割裂
- 攒批推送 → 语义完整 → 延迟增加

## 主流优化方向

1. **智能攒批**：自然边界（标点、换行）处 flush，非固定间隔
2. **分层推送**：优先推送简短的状态更新，长的文本 Token 批处理
3. **Token 平滑**：控制推送速率，避免客户端渲染跟不上
4. **抢占式内容**：先推送占位内容，再逐步更新

## 实现挑战

1. **缓冲与刷新**：既要攒批 flush，又要即时推送状态
2. **连接稳定性**：长连接可能中断，需要断线重连
3. **浏览器兼容**：EventSource API 的限制和回退方案

## 能力边界

- SSE 仅支持服务端到客户端单向流
- 流式协议在反代（Nginx）中可能需要特殊配置
- 不适用于需要双向实时通信的场景（需要 WebSocket）

## 核心优势

Token 流推送极大改善用户感知延迟——从"等待 10 秒"变为"200ms 开始看到内容"。

## 工程优化

1. 使用 Server-Sent Events（SSE）作为默认选择
2. 首个 Token 尽快推送（<500ms）
3. 启用 HTTP/2 避免 SSE 的连接数量限制
4. 在网络不稳定场景支持自动重连（Last-Event-ID）

## 场景判断

- 聊天 Agent：必须流式
- 分析报告 Agent：可批量也可流式
- 后台批处理 Agent：不需要流式
