# 事件驱动 Agent — Event-Driven Agent Architecture

> **核心思想**：Agent 不是通过主动循环来驱动推理，而是响应外部或内部事件来触发推理-行动周期。这种架构将 Agent 从"主动执行者"转变为"反应式处理器"，能够自然地处理异步、并发和实时场景。

---

## 1. 基本原理

### 1.1 什么是事件驱动 Agent

事件驱动 Agent 的核心是 **事件总线 (Event Bus)** 或 **消息代理 (Message Broker)**。Agent 组件之间不直接通信，而是通过发布和订阅事件来协作。

```
传统 Agent (主动循环):
  用户输入 → [Agent Loop: Thought → Action → Obs] → 响应
                     ↑                                    │
                     └────────── 主动轮询/等待 ────────────┘

事件驱动 Agent (反应式):
  
  用户输入 → ──────┐
                    ▼
            ┌──────────────┐     ┌──────────────┐
            │  事件总线     │────►│  Agent 实例   │
            │  (Event Bus)  │     │  (Event-driven│
            │              │◄────│    handler)   │
            └──────────────┘     └──────────────┘
                    ▲                    │
                    │                    ▼
              外部事件            工具调用/新事件
              (Webhook/        (执行结果作为新事件
               Cron/Sensor)     继续驱动处理)
```

### 1.2 背景与演进

**之前怎么做**：最早的 Agent 系统采用主动轮询方式 (Polling-based)。Agent 定期检查是否有新的任务，然后开始处理。这种方式在任务率较低时浪费大量资源，在突发高峰时又响应不及时。

```
轮询模式的问题:
  ┌─ 空闲时不断轮询浪费 Token 和 API 费用
  ├─ 高峰时轮询间隔成为瓶颈
  ├─ 难以处理多源并发事件
  └─ 系统总吞吐受限于轮询频率
```

**核心矛盾**：Agent 的推理过程本身就是异步的 (LLM API 调用延迟 500ms-5s)，但传统的同步循环模型要求 Agent 在整个推理期间保持阻塞等待。这个矛盾在以下场景尤为突出：
- 多个 Agent 需要共享同一个 LLM 后端
- Agent 需要同时监听多个外部事件源
- Agent 的空闲等待时间远大于实际处理时间

### 1.3 事件驱动如何解决

事件驱动模式通过 **异步解耦** 来解决上述矛盾：

```
Agent 处理一个事件的完整事件流:

时间线 →
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ 事件到达   │───►│ 事件分类   │───►│ Agent    │───►│ 推理完成  │───►│ 工具调用  │
│ (Event)   │    │ (Classify)│    │ 推理     │    │ (Result) │    │ (Action) │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                      │
                                                      ▼
                                              ┌──────────────┐
                                              │ 结果事件     │
                                              │ (Event Bus)  │
                                              └──────────────┘
                                                      │
                          ┌───────────────────────────┤
                          ▼                           ▼
                  ┌──────────────┐           ┌──────────────┐
                  │ 后续 Agent   │           │ 响应组装     │
                  │ 继续处理     │           │ 返回用户     │
                  └──────────────┘           └──────────────┘
```

**关键特性**：
- **非阻塞**: Agent 推理时不会阻塞事件总线，其他事件可以并行处理
- **松耦合**: 事件发布者不需要知道谁在处理事件
- **弹性**: 可以通过增加订阅者来扩展处理能力
- **持久化**: 事件可以持久化到消息队列，防止丢失

---

## 2. 架构详解

### 2.1 核心组件

```
事件驱动 Agent 架构:
┌────────────────────────────────────────────────────────────────────┐
│  事件源 (Event Sources)                                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ 用户请求 │ │ Webhook  │ │ Cron     │ │ 传感器    │ │ 系统事件  │ │
│  │ (HTTP)  │ │ (外部API)│ │ (定时)   │ │ (IoT)    │ │ (内部)   │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
└────────────────────────────────────────────────────────────────────┘
         │           │           │           │           │
         ▼           ▼           ▼           ▼           ▼
┌────────────────────────────────────────────────────────────────────┐
│  事件接入层 (Event Ingestion)                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Event Gateway / API Gateway                                 │  │
│  │  协议转换 (HTTP/WebSocket/gRPC) → 内部事件格式              │  │
│  │  认证、限流、事件 Schema 验证                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────────────┐
│  事件总线 (Event Bus / Message Broker)                             │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Topic: agent.task.create    ──► 订阅者: Scheduler           │  │
│  │  Topic: agent.task.complete  ──► 订阅者: ResultAggregator   │  │
│  │  Topic: agent.thought.ready  ──► 订阅者: ActionSelector     │  │
│  │  Topic: agent.action.result  ──► 订阅者: MemoryUpdater      │  │
│  │  Topic: agent.error          ──► 订阅者: ErrorHandler       │  │
│  │                                                              │  │
│  │  支撑: Redis Streams / Kafka / RabbitMQ / NATS              │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────────────┐
│  事件处理器 (Event Handlers / Agent Instances)                     │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────┐│
│  │ Agent 实例 1 │ │ Agent 实例 2 │ │ Agent 实例 3 │ │ ...        ││
│  │ (推理引擎)   │ │ (推理引擎)   │ │ (推理引擎)   │ │ (扩展中)   ││
│  │ 事件→Thought│ │ 事件→Thought│ │ 事件→Thought│ │            ││
│  │ →Action→Obs │ │ →Action→Obs │ │ →Action→Obs │ │            ││
│  └──────────────┘ └──────────────┘ └──────────────┘ └────────────┘│
└────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────────────┐
│  事件存储 (Event Store)                                            │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  事件日志 (Event Sourcing)                                   │  │
│  │  • 所有事件按顺序持久化存储                                   │  │
│  │  • 支持事件回溯和重放 (Replay)                                │  │
│  │  • 审计追踪、调试、恢复                                       │  │
│  │                                                              │  │
│  │  支撑: Apache Kafka / EventStoreDB / PostgreSQL              │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

### 2.2 事件 Schema 设计

事件驱动 Agent 的核心是定义清晰的事件 Schema：

```python
from dataclasses import dataclass, field
from typing import Any, Dict, Optional
from datetime import datetime
import uuid

@dataclass
class AgentEvent:
    """Agent 系统中的通用事件格式"""
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    event_type: str = ""              # 如: "thought.complete", "action.required"
    source: str = ""                  # 事件来源 Agent/组件 ID
    timestamp: datetime = field(default_factory=datetime.utcnow)
    correlation_id: str = ""          # 关联 ID，用于追踪事件链
    session_id: str = ""              # 会话 ID
    payload: Dict[str, Any] = field(default_factory=dict)
    metadata: Dict[str, Any] = field(default_factory=dict)
    
    # 事件 Schema 验证
    def validate(self) -> bool:
        assert self.event_type, "event_type is required"
        assert self.source, "source is required"
        return True

# 事件类型定义
EVENT_TYPES = {
    # 任务生命周期
    "task.created":        {"schema": {"task_id", "input", "context"}},
    "task.started":        {"schema": {"task_id", "assigned_agent"}},
    "task.progress":       {"schema": {"task_id", "progress_pct", "message"}},
    "task.completed":      {"schema": {"task_id", "result", "metrics"}},
    "task.failed":         {"schema": {"task_id", "error", "attempt"}},
    
    # Agent 推理事件
    "thought.generated":   {"schema": {"agent_id", "thought", "confidence"}},
    "action.selected":     {"schema": {"agent_id", "tool", "params"}},
    "action.executed":     {"schema": {"agent_id", "tool", "result"}},
    "observation.parsed":  {"schema": {"agent_id", "observation", "summary"}},
    
    # 系统事件
    "system.alert":        {"schema": {"level", "message", "component"}},
    "system.scale":        {"schema": {"reason", "target_count"}},
    "system.config_change":{"schema": {"key", "old_value", "new_value"}},
}
```

### 2.3 事件处理流程 (完整示例)

```python
import asyncio
from abc import ABC, abstractmethod
from typing import Callable, Dict, List, Optional

class EventBus(ABC):
    """事件总线抽象"""
    @abstractmethod
    async def publish(self, event: AgentEvent):
        pass
    
    @abstractmethod
    async def subscribe(self, event_type: str, handler: Callable):
        pass
    
    @abstractmethod
    async def unsubscribe(self, event_type: str, handler: Callable):
        pass

class InMemoryEventBus(EventBus):
    """内存事件总线实现 (用于开发和测试)"""
    def __init__(self):
        self.subscribers: Dict[str, List[Callable]] = {}
        self.event_log: List[AgentEvent] = []
    
    async def publish(self, event: AgentEvent):
        self.event_log.append(event)
        handlers = self.subscribers.get(event.event_type, [])
        # 并行处理所有订阅者
        await asyncio.gather(
            *[handler(event) for handler in handlers],
            return_exceptions=True
        )
    
    async def subscribe(self, event_type: str, handler: Callable):
        if event_type not in self.subscribers:
            self.subscribers[event_type] = []
        self.subscribers[event_type].append(handler)
    
    async def unsubscribe(self, event_type: str, handler: Callable):
        if event_type in self.subscribers:
            self.subscribers[event_type].remove(handler)

class EventDrivenAgent:
    """事件驱动的 Agent 基类"""
    def __init__(self, agent_id: str, event_bus: EventBus, llm_client):
        self.agent_id = agent_id
        self.event_bus = event_bus
        self.llm = llm_client
        self.tools: Dict[str, Callable] = {}
        
    async def register_handlers(self):
        """注册事件处理器"""
        await self.event_bus.subscribe("task.created", self.handle_task)
        await self.event_bus.subscribe("action.result", self.handle_action_result)
        await self.event_bus.subscribe("system.config_change", self.handle_config_change)
    
    async def handle_task(self, event: AgentEvent):
        """处理新任务事件"""
        # 1. 发送任务开始事件
        await self.event_bus.publish(AgentEvent(
            event_type="task.started",
            source=self.agent_id,
            correlation_id=event.correlation_id,
            payload={"task_id": event.payload["task_id"]}
        ))
        
        # 2. Agent 推理循环 (事件驱动版)
        context = event.payload.get("input", "")
        step = 0
        max_steps = event.payload.get("max_steps", 10)
        
        while step < max_steps:
            # 3. LLM 推理 → 生成 Thought
            thought = await self.llm.reason(context)
            
            await self.event_bus.publish(AgentEvent(
                event_type="thought.generated",
                source=self.agent_id,
                correlation_id=event.correlation_id,
                payload={"thought": thought, "step": step}
            ))
            
            # 4. 从 Thought 中选择 Action
            action = self.parse_action(thought)
            
            if action["type"] == "final_answer":
                # 任务完成
                await self.event_bus.publish(AgentEvent(
                    event_type="task.completed",
                    source=self.agent_id,
                    correlation_id=event.correlation_id,
                    payload={"result": action["content"], "steps": step}
                ))
                return
            
            if action["type"] == "tool_call":
                # 5. 执行工具 (异步)
                result = await self.execute_tool(action)
                
                await self.event_bus.publish(AgentEvent(
                    event_type="action.executed",
                    source=self.agent_id,
                    correlation_id=event.correlation_id,
                    payload={
                        "tool": action["tool_name"],
                        "params": action["params"],
                        "result": result
                    }
                ))
                
                # 6. 更新上下文
                context = f"{context}\n{action['tool_name']} 返回: {result}"
            
            step += 1
    
    async def handle_action_result(self, event: AgentEvent):
        """处理异步工具执行结果"""
        # 某些工具可能异步返回结果，通过事件注入
        pass
    
    async def execute_tool(self, action: Dict) -> str:
        """执行工具"""
        tool_name = action["tool_name"]
        if tool_name in self.tools:
            return await self.tools[tool_name](**action["params"])
        return f"错误: 未知工具 {tool_name}"

# 使用示例
async def main():
    bus = InMemoryEventBus()
    agent = EventDrivenAgent("agent-1", bus, mock_llm())
    await agent.register_handlers()
    
    # 通过事件触发任务
    await bus.publish(AgentEvent(
        event_type="task.created",
        source="user",
        payload={
            "task_id": "task-001",
            "input": "分析这份销售数据并生成报告",
            "max_steps": 15
        }
    ))
    
    # 等待处理完成
    await asyncio.sleep(5)
```

---

## 3. 高级设计

### 3.1 事件溯源 (Event Sourcing) 与状态恢复

事件驱动 Agent 的一个重要优势是可以使用事件溯源来保存完整的 Agent 执行历史：

```
Agent 状态 = 所有历史事件的累积结果

状态重建:
  1. 从 Event Store 读取 Agent 的所有历史事件
  2. 按时间顺序重放 (Replay) 事件
  3. 每一步应用事件到状态机
  4. 最终状态 = 完全恢复的 Agent 状态

恢复粒度:
  • 全量恢复: 从头重放所有事件 (慢但精确)
  • 快照+增量: 从最近快照恢复 + 重放增量事件 (快)
  • 指定时间点: 恢复到特定时间戳的状态 (调试用)
```

```python
class EventSourcedAgent:
    """基于事件溯源的 Agent 状态管理"""
    def __init__(self, agent_id: str, event_store):
        self.agent_id = agent_id
        self.event_store = event_store
        self.state = {
            "tasks": [],
            "current_task": None,
            "conversation_history": [],
            "tool_states": {},
            "memory": {}
        }
        self.event_handlers = {
            "task.created": self.apply_task_created,
            "task.started": self.apply_task_started,
            "thought.generated": self.apply_thought,
            "action.executed": self.apply_action,
            "task.completed": self.apply_task_completed,
        }
    
    async def replay(self, from_snapshot: bool = True):
        """从事件存储中重建状态"""
        events = await self.event_store.get_events(self.agent_id)
        
        # 如果有快照，从快照开始
        if from_snapshot:
            snapshot = await self.event_store.get_latest_snapshot(self.agent_id)
            if snapshot:
                self.state = snapshot.state
                # 只重放快照之后的事件
                snapshot_idx = next(
                    i for i, e in enumerate(events) 
                    if e.event_id == snapshot.event_id
                )
                events = events[snapshot_idx + 1:]
        
        # 重放事件
        for event in events:
            handler = self.event_handlers.get(event.event_type)
            if handler:
                handler(event)
    
    def apply_task_created(self, event: AgentEvent):
        self.state["tasks"].append({
            "id": event.payload["task_id"],
            "input": event.payload["input"],
            "status": "pending"
        })
    
    def apply_task_completed(self, event: AgentEvent):
        task_id = event.payload["task_id"]
        for task in self.state["tasks"]:
            if task["id"] == task_id:
                task["status"] = "completed"
                task["result"] = event.payload.get("result")
```

### 3.2 事件驱动的多 Agent 协作

事件驱动模式下，多 Agent 协作变得自然——Agent 之间不直接通信，而是通过发布事件来间接交互：

```
场景: 文档问答系统

事件流:
┌──────────┐    task.created     ┌──────────┐
│  用户     │───────────────────►│  协调     │
│  (请求)   │                    │  Agent   │
└──────────┘                    └─────┬─────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                  │
                    ▼                 ▼                  ▼
            ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
            │ 检索 Agent   │ │ 推理 Agent   │ │ 验证 Agent   │
            │ (搜索文档)   │ │ (生成答案)   │ │ (检查准确性) │
            └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
                   │                │                 │
                   │  doc.found    │  answer.ready   │  verified
                   ▼                ▼                 ▼
            ┌──────────────────────────────────────────────────┐
            │              事件总线 (Event Bus)                   │
            │  task.created → 协调 Agent                        │
            │  doc.found → 推理 Agent                           │
            │  answer.ready → 验证 Agent                        │
            │  verified → 协调 Agent → 响应客户                  │
            └──────────────────────────────────────────────────┘

优势:
  • 每个 Agent 独立部署、独立扩展
  • 添加新 Agent 只需要订阅相关事件
  • Agent 故障不影响其他 Agent
  • 支持事件重放用于调试和测试
```

### 3.3 基于 Redis Streams 的生产级实现

```python
import redis.asyncio as redis
import json
from typing import Optional

class RedisStreamEventBus(EventBus):
    """基于 Redis Streams 的生产级事件总线"""
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = None
        self.redis_url = redis_url
        self.consumer_group = "agent-group"
        self.handlers: Dict[str, List[Callable]] = {}
    
    async def connect(self):
        self.redis = await redis.from_url(self.redis_url)
        # 创建消费者组 (如果不存在)
        for topic in EVENT_TYPES:
            try:
                await self.redis.xgroup_create(
                    topic, self.consumer_group, id="0", mkstream=True
                )
            except redis.ResponseError:
                pass  # 组已存在
    
    async def publish(self, event: AgentEvent):
        event.validate()
        await self.redis.xadd(
            event.event_type,        # Stream key = event type
            {
                "data": json.dumps({
                    "event_id": event.event_id,
                    "event_type": event.event_type,
                    "source": event.source,
                    "timestamp": event.timestamp.isoformat(),
                    "correlation_id": event.correlation_id,
                    "session_id": event.session_id,
                    "payload": event.payload,
                    "metadata": event.metadata
                })
            },
            maxlen=10000  # 保留最近 10000 条事件
        )
    
    async def subscribe(self, event_type: str, handler: Callable):
        if event_type not in self.handlers:
            self.handlers[event_type] = []
        self.handlers[event_type].append(handler)
    
    async def listen_loop(self, batch_size: int = 10):
        """持续监听事件的主循环"""
        while True:
            for event_type, handlers in self.handlers.items():
                # 从 Stream 读取新事件
                results = await self.redis.xreadgroup(
                    self.consumer_group,
                    f"consumer-{id(self)}",
                    {event_type: ">"},  # 只读取未处理的消息
                    count=batch_size,
                    block=1000  # 阻塞 1 秒
                )
                
                if not results:
                    continue
                
                for stream_name, messages in results:
                    for msg_id, msg_data in messages:
                        try:
                            event_data = json.loads(msg_data[b"data"])
                            event = AgentEvent(**event_data)
                            
                            # 并行处理所有处理器
                            await asyncio.gather(
                                *[h(event) for h in handlers],
                                return_exceptions=True
                            )
                            
                            # 确认处理完成
                            await self.redis.xack(
                                event_type, self.consumer_group, msg_id
                            )
                        except Exception as e:
                            # 将失败的消息移到死信队列
                            await self.redis.xadd(
                                f"dead_letter:{event_type}",
                                {"msg_id": msg_id, "error": str(e)}
                            )
```

---

## 4. 适用场景与能力边界

### 4.1 最佳适用场景

| 场景 | 原因 | 案例 |
|------|------|------|
| **异步处理** | 事件总线天然支持异步非阻塞 | 文档批量处理、邮件自动回复 |
| **实时响应** | 事件触发延迟 < 1ms | 客服机器人实时响应 |
| **多源输入** | 统一的事件接入层 | IoT 数据处理、监控告警 |
| **流水线处理** | 事件驱动链式处理 | 内容审核流水线 |
| **高可扩展** | 水平扩展订阅者 | 大规模 SaaS 服务 |

### 4.2 不适用场景

| 场景 | 原因 | 替代方案 |
|------|------|----------|
| **简单问答** | 事件驱动增加不必要的复杂度 | 单体 Agent |
| **强一致性** | 事件最终一致性不满足要求 | 单体/管道架构 |
| **固定流程** | 管道架构更直接 | 管道架构 |
| **低延迟交互** | 事件序列化/反序列化增加延迟 | 单体/管道架构 |
| **小规模系统** | 基础设施开销 > 收益 | 单体 Agent |

### 4.3 能力边界

```
事件驱动 Agent 的核心挑战:

┌─────────────────────────────────────────────────────┐
│ 挑战                    影响                        │
├─────────────────────────────────────────────────────┤
│ 事件顺序保证            可能需要全局排序            │
│ 至少一次 vs 精确一次    消息投递语义权衡            │
│ 事件重复处理            需要幂等性设计              │
│ 调试困难                事件流追踪复杂              │
│ 状态一致性              最终一致性的业务妥协        │
│ 事件 Schema 演进        向后兼容 Schema 管理        │
│ 死信处理                失败事件的正确处理          │
│ 背压 (Backpressure)     消费者跟不上生产者          │
└─────────────────────────────────────────────────────┘
```

---

## 5. 工程优化方向

| 方向 | 方法 | 效果 |
|------|------|------|
| **事件压缩** | 批量处理多个事件，合并推理 | 减少 LLM API 调用 30-50% |
| **优先级队列** | 高优先级事件优先处理 | 降低 P95 延迟 |
| **事件预聚合** | 窗口内相似事件合并 | 减少重复处理 |
| **自适应并发** | 根据事件积压动态调整消费者 | 提高吞吐 |
| **事件采样** | 高频率事件只采样处理 | 降低成本 |
| **预测预取** | 根据事件模式预初始化 Agent | 减少冷启动延迟 |
