# 8.1.3 async-comm — 异步通信：消息队列 / Pub-Sub

## 简单介绍

异步通信让 Agent 发送消息后无需等待响应即可继续工作，接收方在准备好时处理消息。这是构建高吞吐、弹性多 Agent 系统的关键模式，通过消息队列（Message Queue）或发布-订阅（Pub-Sub）中间件解耦各 Agent。

## 基本原理

### 消息队列模式（Point-to-Point）

```
                    ┌──────────────┐
Sender ──► Queue ──►│   Message    │──► Receiver
                    │   Broker     │──► Receiver
                    └──────────────┘
```

- 每条消息被一个消费者处理
- 支持负载均衡（多个 Worker 争抢消息）
- 消息持久化，不丢失

### 发布-订阅模式（Pub-Sub）

```
                   ┌──────────────┐
Publisher ──► Topic ──►│    Broker    │──► Subscriber A
                   │              │──► Subscriber B
                   └──────────────┘──► Subscriber C
```

- 每条消息被所有订阅者接收
- 发布者和订阅者完全解耦
- 支持通配符订阅和消息过滤

### 基本工作流

```
Agent A (发送)                     Agent B (接收)
    │                                  │
    │── 1. 序列化消息 ──►              │
    │── 2. 发送到 Queue/Topic ──►      │
    │   3. 立即返回（不阻塞）            │
    │                                  │
    │   （A 继续其他工作）               │── 4. 从 Queue 拉取
    │                                  │── 5. 反序列化
    │                                  │── 6. 处理消息
    │                                  │── 7. 确认/回执
    │◄──── 可选回调/通知 ───────────────│
```

## 背景与演进

- **前期（2022-2023）**：多 Agent 系统主要用同步 HTTP 通信。当 Agent 数量超过 3-5 个时，同步阻塞导致严重的级联延迟——一个慢 Agent 拖垮整个系统。
- **消息中间件引入（2023）**：开发者在 Agent 架构中引入 Redis Pub/Sub 和 RabbitMQ。解决了阻塞问题，但引入了消息序列化、可靠投递、死信处理等新挑战。
- **事件驱动 Agent（2024）**：基于 Apache Kafka / Redpanda 的事件驱动架构成为主流，Agent 不再"调用"其他 Agent，而是"发布事件"并"响应事件"。系统松耦合达到新高度。
- **当前趋势**：CloudEvents 规范在多 Agent 系统中的采用、Serverless Agent 与消息队列的天然集成、以及基于事件溯源（Event Sourcing）的 Agent 状态管理。

## 核心矛盾

**"强一致性 vs 高吞吐"——异步通信的经典权衡**：

| 同步的强一致性 | 异步的高吞吐 |
|---------------|-------------|
| 即时知道结果 | 结果需要轮询或回调 |
| 事务边界清晰 | 分布式事务困难 |
| 错误立即暴露 | 错误延迟发现 |
| 级联阻塞 | 背压和削峰填谷 |
| 吞吐受限（等待） | 吞吐极高（流水线） |

## 主流优化方向

1. **消息持久化**：消息写入磁盘而非纯内存，崩溃后可恢复
2. **死信队列（DLQ）**：无法处理的消息转入 DLQ，避免阻塞管道
3. **消息去重**：幂等消费者 + 消息 ID 去重表（Redis）
4. **批量消费**：每次拉取一批消息批量处理，提高吞吐
5. **延迟队列**：消息在指定时间后才可见（用于重试、定时任务）
6. **消息压缩**：大消息启用 snappy / zstd 压缩，减少网络带宽
7. **背压控制**：消费者负载高时通知生产者降低发送速率

## 三种主流消息中间件对比

| 特性 | Redis Pub/Sub | RabbitMQ | Apache Kafka |
|------|--------------|----------|-------------|
| 模型 | Pub/Sub | Queue + Exchange | Partition Log |
| 持久化 | ❌（默认） | ✅ | ✅ |
| 消息顺序 | 不保证 | 单队列有序 | 分区内有序 |
| 吞吐量 | ~10万/s | ~数万/s | ~百万/s |
| 延迟 | <1ms | <1ms | ~2-5ms |
| 消息回溯 | ❌ | ❌ | ✅（基于 offset） |
| 消费者组 | ❌ | ✅ | ✅ |
| 典型 Agent 场景 | 实时通知 | 任务队列 | 事件溯源/日志 |

## 能力边界与结果边界

- **消息大小限制**：Kafka 默认 1MB，RabbitMQ 建议 <10MB，Redis 建议 <1MB
- **消息保留**：Kafka 可保留数天/永久，RabbitMQ ACK 后删除，Redis Pub/Sub 无存储
- **交付保证**：RabbitMQ/Kafka 支持 at-least-once，Redis Pub/Sub at-most-once
- **消费顺序**：Kafka 分区内有序，RabbitMQ 单队列有序，Redis 无序
- **不适用场景**：需要即时双向交互、强实时性要求、事务性操作

## 与其他通信模式的区别

- **vs 同步通信（8.1.2）**：异步解耦发送和接收，但引入了消息确认、重试、幂等性等额外复杂性。
- **vs 广播（8.1.4）**：Pub-Sub 本质上是"有过滤的广播"——消息精确发送给订阅者，不是无差别广播。
- **vs 消息格式（8.1.1）**：异步关注的是传输和投递语义，消息格式关注的是序列化和结构。

## 核心优势

- **解耦**：发送方和接收方互不感知彼此的存在，可以独立部署和扩展
- **弹性**：消息队列缓冲峰值流量，系统不会因突发负载崩溃
- **可靠**：消息持久化保障崩溃后不丢失
- **可扩展**：消费者可以水平扩展并行处理

## 工程优化

1. **消息分区键**：按 Agent ID / 任务类型分区，保证同一 Agent 的消息有序
2. **消费幂等**：基于消息 ID 的 Redis 去重，防止重复处理
3. **批量确认**：每 N 条或每 T 毫秒批量 ACK，减少网络往返
4. **监控告警**：监控队列深度、消费延迟、死信数量
5. **优雅关闭**：关闭前处理完已消费但未 ACK 的消息

## 适用场景

| 场景 | 推荐中间件 | 原因 |
|------|-----------|------|
| Agent 间任务队列 | RabbitMQ | 灵活的路由、可靠的投递 |
| 事件驱动 Agent | Kafka | 高吞吐、消息回溯、持久化 |
| 实时状态通知 | Redis Pub/Sub | 低延迟、轻量级 |
| 多 Agent 日志收集 | Kafka | 高吞吐、持久化、重放 |
| 跨数据中心 Agent | RabbitMQ（联邦插件） | 网络不稳定下的可靠性 |

## 示例代码

### Redis Pub/Sub 实现

```python
import redis.asyncio as redis
import json
import uuid
import asyncio
from typing import Callable, Awaitable

class AsyncAgentComm:
    """基于 Redis Pub/Sub 的异步 Agent 通信"""

    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self.pubsub = self.redis.pubsub()
        self.handlers: dict[str, Callable] = {}

    async def publish(self, channel: str, message: dict):
        """发布消息到频道"""
        payload = json.dumps({
            "id": f"msg_{uuid.uuid4().hex[:12]}",
            "from": channel,
            "payload": message,
            "timestamp": asyncio.get_event_loop().time()
        })
        await self.redis.publish(channel, payload)
        print(f"[Pub] {channel}: {message.get('type', 'unknown')}")

    async def subscribe(self, channel: str,
                        handler: Callable[[dict], Awaitable[None]]):
        """订阅频道并注册处理器"""
        await self.pubsub.subscribe(channel)
        self.handlers[channel] = handler

    async def start_listening(self):
        """开始监听订阅的消息"""
        async for message in self.pubsub.listen():
            if message["type"] != "message":
                continue
            data = json.loads(message["data"])
            channel = message["channel"].decode()
            if handler := self.handlers.get(channel):
                asyncio.create_task(handler(data))
            else:
                print(f"[Warn] 未注册处理器: {channel}")

    async def close(self):
        await self.pubsub.unsubscribe()
        await self.redis.close()


# 使用示例
async def main():
    # 创建通信层
    comm = AsyncAgentComm()

    # 研究员 Agent 监听 research_task 频道
    async def researcher_handler(msg):
        query = msg["payload"]["query"]
        print(f"[Researcher] 收到研究任务: {query}")
        # 模拟研究过程...
        await asyncio.sleep(2)
        result = {"query": query, "papers": ["Paper A", "Paper B"]}
        # 发布结果到 research_result 频道
        await comm.publish("research_result", {
            "type": "research_complete",
            "correlation_id": msg["id"],
            "result": result
        })

    await comm.subscribe("research_task", researcher_handler)

    # 写作 Agent 监听 research_result 频道
    async def writer_handler(msg):
        print(f"[Writer] 收到研究成果: {msg['result']}")
        # 开始写作...
        await comm.publish("article_ready", {
            "type": "article_complete",
            "content": f"基于研究成果的文章: {msg['result']['papers']}"
        })

    await comm.subscribe("research_result", writer_handler)

    # 启动监听
    asyncio.create_task(comm.start_listening())

    # 模拟发布任务
    await comm.publish("research_task", {
        "type": "research_request",
        "query": "multi-agent communication protocols"
    })

    await asyncio.sleep(5)
    await comm.close()

asyncio.run(main())
```

### Kafka 事件驱动 Agent

```python
from kafka import KafkaProducer, KafkaConsumer
import json

class EventDrivenAgent:
    """基于 Kafka 的事件驱动 Agent"""

    def __init__(self, name: str, group_id: str,
                 bootstrap_servers: str = "localhost:9092"):
        self.name = name
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode(),
            key_serializer=lambda k: k.encode()
        )
        self.consumer = KafkaConsumer(
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            value_deserializer=lambda v: json.loads(v.decode()),
            auto_offset_reset="earliest"
        )

    def emit(self, event_type: str, data: dict,
             partition_key: str = None):
        """发布事件"""
        event = {
            "type": event_type,
            "source": self.name,
            "data": data
        }
        self.producer.send(
            topic=event_type,
            key=partition_key or self.name,
            value=event
        )
        print(f"[{self.name}] 发布事件: {event_type}")

    def on(self, event_type: str, handler, timeout_ms: int = 1000):
        """订阅事件"""
        self.consumer.subscribe([event_type])
        for msg in self.consumer:
            event = msg.value
            if event["type"] == event_type:
                handler(event)
```
