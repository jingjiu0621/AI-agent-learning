# 8.1.4 broadcast — 广播模式：信息分发

## 简单介绍

广播模式让一个 Agent 将消息同时发送给系统中的所有（或一组）Agent。与点对点通信不同，广播不关心谁接收——消息被"投射"到整个网络或特定群体。广播是最简单的信息分发方式，但也是通信复杂度最高的模式。

## 基本原理

```
                ┌──────────────┐
                │  Agent A     │
                │  (Broadcaster)│
                └──────┬───────┘
                       │
               "status_update"
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Agent B  │ │ Agent C  │ │ Agent D  │
   │(Sub)     │ │(Sub)     │ │(Sub)     │
   └──────────┘ └──────────┘ └──────────┘
```

### 三种广播语义

| 类型 | 范围 | 可靠性 | 复杂度 |
|------|------|--------|--------|
| 全广播（Fan-out） | 所有 Agent | 尽力而为 | O(n) |
| 组播（Multicast） | 特定组 | 可配置 | O(k) k=组大小 |
| 层级广播（Hierarchical） | 子树 | 逐层确认 | O(log n) |

## 背景与演进

- **前期（2022）**：多 Agent 系统使用"循环遍历"模拟广播——Orchestrator Agent 逐个调用其他 Agent 的 API 发送相同消息。O(n) 的时间复杂度和点对点的错误处理让实现变得很糟糕。
- **Pub/Sub 中间件（2023）**：Redis Pub/Sub 和 RabbitMQ Fanout Exchange 被引入，实现了"一次发送，多处接收"。但缺乏针对性，所有 Agent 收到所有消息，浪费处理资源。
- **分层广播（2024）**：大型系统开始使用分层广播——Orchestrator 通知 Group Leader，Leader 转发给组成员。复杂度从 O(n) 降低到 O(log n)。
- **当前趋势**：基于内容的过滤广播（Content-Based Routing），Agent 声明自己感兴趣的消息类型，中间件只在匹配时才转发。

## 核心矛盾

**"信息覆盖 vs 通信效率"——广播的根本权衡**：

| 覆盖优先 | 效率优先 |
|----------|---------|
| 确保所有 Agent 知情 | 只通知相关的 Agent |
| 实现简单 | 需要订阅/过滤机制 |
| 适合全局状态通知 | 可能遗漏某些 Agent |
| 带宽浪费严重 | 带宽友好 |
| Agent 自过滤 | 中间件预过滤 |

## 主流优化方向

1. **分层广播树**：建立 Agent 层级树，广播沿着树扩散，减少根节点压力
2. **内容过滤路由**：Agent 声明消息类型订阅，中间件智能路由
3. **增量广播**：只广播变化部分（类似 git diff），而非完整状态
4. **去重机制**：广播消息设置唯一 ID，接收方基于 ID 去重
5. **速率限制**：限制单位时间内广播频率，防止广播风暴
6. **有损广播**：非关键消息允许丢失，关键消息单独确认

## 能力边界与结果边界

| 指标 | 小规模 (<10 Agents) | 中规模 (10-100) | 大规模 (>100) |
|------|-------------------|----------------|---------------|
| 全广播延迟 | <1ms | 1-5ms | 5-50ms |
| 网络带宽 | 低 | 中 | 高（需优化） |
| 消息丢失率 | 0.1% | 0.5% | 1-5%（无确认） |
| 推荐模式 | 全广播 | 组播 | 分层+过滤 |

- **不适用场景**：大规模系统（>500 Agents）的全广播、需要确认送达的关键指令、带宽受限的网络环境

## 与其他通信模式的区别

- **vs 同步通信（8.1.2）**：同步是一对一、要响应；广播是一对多、不关心响应。
- **vs 异步通信（8.1.3）**：Pub/Sub 本质上是"有订阅关系的广播"。广播是 Pub/Sub 的特例——不关心谁接收。
- **vs 消息队列**：队列是点对点（一条消息一个消费者），广播是一对多。

## 核心优势

- **极速信息扩散**：一次操作即可通知整个系统
- **简单语义**：发送者不需要知道谁在接收
- **全局一致性**：所有 Agent 同时收到相同信息
- **弹性扩展（读端）**：加新 Agent 不需修改发送方

## 工程优化

1. **广播 ID + 去重表**：每个广播消息带唯一 ID，接收方维护 LRU 去重缓存
2. **分层广播实现**：建立物理/逻辑拓扑，广播沿树传播
3. **自适应广播**：根据 Agent 活跃度和相关性动态决定广播范围
4. **压缩广播**：对大型广播内容启用压缩（snappy / zstd）
5. **广播审计**：记录广播消息的生命周期和到达率

## 适用场景

| 场景 | 说明 | 推荐实现 |
|------|------|----------|
| 全局配置更新 | 所有 Agent 需立即获知配置变更 | Redis Pub/Sub |
| 状态心跳 | Agent 定期广播存活状态 | UDP 组播 |
| 事件通知 | 系统级别事件（开始/停止/错误） | RabbitMQ Fanout |
| 模型更新推送 | 所有 Agent 使用相同的模型版本 | Kafka（广播 consumer group） |
| 全局告警 | 异常检测后的全网通知 | 分层广播树 |

## 示例代码

### Redis Pub/Sub 广播实现

```python
import redis.asyncio as redis
import json
import asyncio

class BroadcastChannel:
    """基于 Redis 的广播频道"""

    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self.pubsub = self.redis.pubsub()

    async def broadcast(self, channel: str, message: dict):
        """向频道广播消息"""
        payload = json.dumps({
            **message,
            "_broadcast": True,
            "_channel": channel
        })
        count = await self.redis.publish(channel, payload)
        print(f"[Broadcast] 发送到 {channel}: {count} 个接收者")
        return count

    async def listen(self, channel: str, agent_name: str):
        """Agent 监听广播频道"""
        await self.pubsub.subscribe(channel)
        print(f"[{agent_name}] 开始监听广播: {channel}")
        async for msg in self.pubsub.listen():
            if msg["type"] == "message":
                data = json.loads(msg["data"])
                yield data

# 使用示例
async def broadcast_example():
    channel = BroadcastChannel()

    async def agent_listener(name: str):
        async for msg in channel.listen("system:events", name):
            print(f"[{name}] 收到广播: {msg.get('event')}")

    # 多个 Agent 监听
    asyncio.create_task(agent_listener("Agent-Alpha"))
    asyncio.create_task(agent_listener("Agent-Beta"))
    asyncio.create_task(agent_listener("Agent-Gamma"))

    await asyncio.sleep(0.5)

    # 广播全局事件
    await channel.broadcast("system:events", {
        "event": "config_update",
        "data": {"model": "gpt-4", "temperature": 0.7}
    })

    await asyncio.sleep(1)

asyncio.run(broadcast_example())
```

### 分层广播实现

```python
import asyncio

class HierarchicalBroadcaster:
    """分层广播树实现"""

    def __init__(self, agent_id: str, parent=None,
                 children: list = None):
        self.agent_id = agent_id
        self.parent = parent
        self.children = children or []

    async def broadcast_down(self, message: dict, ttl: int = 3):
        """向下广播（从根向叶子）"""
        if ttl <= 0:
            return
        print(f"[{self.agent_id}] 转发广播: {message.get('type')}")
        tasks = []
        for child in self.children:
            tasks.append(
                child.broadcast_down(message, ttl - 1)
            )
        await asyncio.gather(*tasks)

    async def broadcast_up(self, message: dict, ttl: int = 3):
        """向上广播（从叶子向根）"""
        if ttl <= 0:
            return
        print(f"[{self.agent_id}] 向上汇报: {message.get('type')}")
        if self.parent:
            await self.parent.broadcast_up(message, ttl - 1)
```

### 去重广播接收

```python
from collections import OrderedDict

class DedupBroadcastReceiver:
    """带去重的广播接收器"""

    def __init__(self, max_seen: int = 1000):
        self.seen = OrderedDict()  # LRU 缓存
        self.max_seen = max_seen

    def is_duplicate(self, message_id: str) -> bool:
        if message_id in self.seen:
            return True
        self.seen[message_id] = True
        if len(self.seen) > self.max_seen:
            self.seen.popitem(last=False)  # 移除最旧的
        return False

    def receive(self, raw_message: dict) -> bool:
        """接收消息，如果是重复则丢弃"""
        msg_id = raw_message.get("id", "")
        if self.is_duplicate(msg_id):
            print(f"[Dedup] 丢弃重复消息: {msg_id}")
            return False
        print(f"[Dedup] 处理新消息: {msg_id}")
        return True
```
