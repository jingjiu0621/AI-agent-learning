# 10.5.5 Real-Time Visualization — 实时流式可视化

## 简单介绍

Real-Time Visualization（实时流式可视化）在 Agent 执行的同时将中间状态实时呈现给用户。与"事后分析"的死后检查不同，实时可视化让用户可以看到 Agent **"正在做什么"**——思考、调用工具、等待结果、生成回复。

## 基本原理

实时可视化的技术架构：

```
Agent 执行                         前端展示
┌──────────┐     SSE/WebSocket     ┌──────────────┐
│ Step 1   │ ──────────────────→  │ 思维流:      │
│  Thought  │     事件流           │ "用户需要查   │
│  Action   │                     │  天气..."     │
│  Observe  │                     │              │
└──────────┘                     │ 工具: 🌤️     │
         │                       │  搜索天气...  │
         │   事件:               │  [=====] 70% │
         │   thought_start       │              │
         │   action_invoke       │ Token: 450   │
         │   action_result       │ ⏱️ 2.3s      │
         │   thought_end         └──────────────┘
```

事件流协议设计：

```python
@dataclass
class VisEvent:
    """可视化事件"""
    type: str        # thought_start / thought_token / action_invoke
                     # action_result / observation / error / complete
    agent_id: str
    # 不同类型携带不同 payload
    payload: dict
    timestamp: float

# 事件序列示例：
[
    {"type": "thought_start", "agent": "weather", "payload": {"step": 3}},
    {"type": "thought_token", "agent": "weather",
     "payload": {"token": "用户想查"}},          # 流式 Token
    {"type": "thought_token", "agent": "weather",
     "payload": {"token": "北京天气"}},
    {"type": "action_invoke", "agent": "weather",
     "payload": {"tool": "get_weather", "params": "city=Beijing"}},
    {"type": "action_result", "agent": "weather",
     "payload": {"result": "22°C", "duration_ms": 320}},
    {"type": "thought_end", "agent": "weather",
     "payload": {"summary": "已获取天气信息，准备回复"}},
]
```

## 背景

Agent 的响应时间通常在秒级到分钟级——远长于传统 API。如果用户在等待过程中只能看到"转圈"或"正在输入"，体验很差。实时可视化让用户在等待过程中可以感受到"Agent 在做事"——每一步 Think / Action / Observation 都实时可见。

这与 LLM 的"流式输出"（Typewriter Effect）有异曲同工之处，但维度更丰富——不仅仅展示最终的回答文字，还展示 Agent 的整个工作过程。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 无实时展示 | 用户看到"正在输入..." | 用户不知道 Agent 在做什么，等待焦虑 |
| 仅流式文本 | 只展示 LLM 的 token 流 | 看不到工具调用和思考过程 |
| 静态步骤计数 | "正在第 3/5 步" | 信息太少，用户不知道每一步的具体内容 |
| 轮询模式 | 前端定期轮询 Agent 状态 | 延迟高、浪费资源、不能展示流式过程 |

## 当前主流优化方向

1. **增量式 UI 更新**——可视化界面不是整体重绘，而是每次事件驱动局部更新（新 Thought 加入、工具调用状态变更），保持界面流畅
2. **动画化工具调用**——工具调用中显示动画（如"搜索中..."的放大镜动画、"代码执行中"的终端动画），提升等待体验
3. **预测性进度**——基于历史数据预测当前步骤的预计剩余时间（"通常这一步需要 2-3 秒"），给用户明确的等待预期
4. **实时 Token 计数器**——在界面上实时展示消耗的 Token 数和预估成本，让用户对"代价"有感知

## 实现的最大挑战

1. **事件频率控制**——Agent 的每一步可能产生大量事件（特别是流式 Thought Token），不加节制的发送会导致前端性能崩溃。需要事件节流（throttle）和合并（batch）
2. **前端渲染性能**——Agent 执行 30 步后，前端可能积累了 1000+ 个事件节点，DOM 渲染和更新的性能压力大
3. **网络延迟**——事件从 Agent 到前端的网络传输可能导致可视化"延后"于 Agent 的实际执行
4. **连接稳定性**——Agent 长时间执行（5 分钟以上），WebSocket 可能断连，需要自动重连和状态同步

## 能力边界

**能做什么：**
- 让用户看到 Agent"正在工作"的过程，减少等待焦虑
- 实时展示 Token 消耗、延迟等指标
- 在 Agent 执行过程中即时发现异常（如工具调用超时）
- 提升 Agent 产品的透明度和可信度

**不能做什么：**
- 不能展示 Agent 还未发生的事情——可视化只能"跟随"Agent 的执行
- 不能解决 Agent 执行慢的问题——实时可视化不优化执行速度
- 不能在不影响用户体验的情况下展示全部细节——信息的详细程度需要权衡

## 最终工程优化

1. **事件采样策略**——放弃每个 Token 都推送的"极致粒度"，改为每 200ms 推送一次增量，每次推送携带最近 200ms 的事件摘要
2. **虚拟列表**——对于长执行链（30+ 步），只渲染可见区域的事件节点（视口内 + 前后各 5 个缓冲），其余节点用虚拟滚动替代 DOM 渲染
3. **WebSocket 心跳 + 重连**——每 5 秒发送一次 ping/pong 保持连接，断连后自动重连并请求最近的事件快照同步状态
4. **渐进式详情展开**——默认只展示事件标题（"调用 get_weather"），点击后展开详情（参数、结果、耗时），避免初始信息过载
