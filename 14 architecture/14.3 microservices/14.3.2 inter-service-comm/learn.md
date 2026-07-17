# 服务间通信协议 (Inter-Service Communication)

## 一、基本原理

在 AI Agent 系统中，一个复杂的 Agent 通常由多个微服务组成：推理服务 (Reasoning Service)、工具执行服务 (Tool Executor)、记忆服务 (Memory Service)、规划服务 (Planner Service) 等。这些服务需要高效、可靠地相互通信，才能协同完成一个完整的 Agent 推理循环。

```
                    +-----------+
                    |  Gateway  |
                    +-----+-----+
                          |
          +---------------+---------------+
          |               |               |
    +-----v-----+  +-----v-----+  +-----v-----+
    | Reasoning  |  |   Tool    |  |  Memory   |
    |  Service   |  | Executor  |  |  Service  |
    +-----+------+  +-----+-----+  +-----+-----+
          |               |               |
          +-------+-------+-------+-------+
                  |               |
           +------v------+ +------v------+
           |   Message   | |   State     |
           |    Queue     | |   Store     |
           +-------------+ +-------------+
```

### 1.1 通信模式分类

| 维度 | 类型 | 说明 |
|------|------|------|
| 时序 | 同步 (Sync) | 调用方阻塞等待响应 |
| 时序 | 异步 (Async) | 调用方不等待，通过回调/轮询获取结果 |
| 方向 | 一元 (Unary) | 单一请求 -> 单一响应 |
| 方向 | 流式 (Streaming) | 持续数据传输（Agent 思维链） |
| 耦合 | 点对点 (Point-to-Point) | 直接调用目标服务 |
| 耦合 | 发布-订阅 (Pub-Sub) | 通过消息中间件解耦 |

### 1.2 Agent 通信的核心挑战

传统微服务通信主要传输结构化业务数据（订单、用户、商品），而 Agent 服务通信传输的是 **推理状态**、**思维链**、**部分结果**。这带来了本质差异：

- **流式本质**：Agent 的思维是逐步展开的，需要部分结果实时推送
- **长时连接**：一次 Agent 推理可能持续数十秒到数分钟
- **状态爆炸**：每一步推理都可能产生中间状态
- **失败代价高**：推理到第 N 步时服务崩溃，重做代价极大

---

## 二、背景与演进

### 2.1 从单体 Agent 到微服务 Agent

```
时间线:

2019-2021        2022-2023           2024-2025
+--------+     +------------+      +----------------+
| 单体   | --> | 功能拆分    | --> | 通信协议专业化  |
| Agent  |     | LLM+工具   |     | 流式+事件驱动   |
+--------+     +------------+      +----------------+

单体时代:
  [Agent Process] --直接函数调用--> [LLM] [Tools] [Memory]
  问题: 无法扩展，LLM 调用阻塞整个进程

拆分时代:
  [ReasoningSvc] --REST/HTTP--> [ToolSvc]
  [ReasoningSvc] --REST/HTTP--> [MemorySvc]
  问题: REST 请求-响应模型不适合流式思维

专业化时代（当前）:
  [ReasoningSvc] --gRPC 双向流--> [ToolSvc]
  [ReasoningSvc] --Kafka 事件--> [MemorySvc]
  [ReasoningSvc] --WebSocket--> [Gateway -> Client]
```

### 2.2 为什么 Agent 服务间通信与传统微服务不同

| 特性 | 传统微服务 | Agent 微服务 |
|------|-----------|-------------|
| 请求时长 | 毫秒级 | 秒~分钟级（多次 LLM 调用） |
| 数据传输 | 完整业务对象 | 流式 token + 中间推理状态 |
| 失败模式 | 请求失败 -> 重试 | 部分失败 -> 状态恢复/降级 |
| 状态管理 | 无状态为主 | 有状态推理上下文 |
| 通信模式 | 同步为主 | 异步+流式为主 |
| 数据大小 | 固定结构 | 变长（思维链长度不确定） |

### 2.3 Agent 推理循环中的通信流

```
用户请求
    |
    v
+--------+     gRPC 双向流     +----------------+
|Gateway |<------------------>| Reasoning Svc  |
+--------+                     +-------+--------+
                                        |
                       +----------------+----------------+
                       |                |                |
                  gRPC 流          HTTP/REST         Kafka/Redis
                       |                |                |
                 +-----v-----+   +-----v-----+   +------v------+
                 | Planner   |   |   Tool    |   |   Memory   |
                 | Service   |   | Executor  |   |  Service   |
                 +-----------+   +-----------+   +-------------+
                       |                |
                  (推理步骤分解)    (工具调用结果)

典型通信顺序:
1. Reasoning Svc -> Planner Svc:  "将问题分解为子任务" (gRPC unary)
2. Planner Svc -> Reasoning Svc:  "子任务列表: [A, B, C]" (gRPC unary)
3. Reasoning Svc -> Tool Executor: "执行工具 A" (gRPC 双向流)
4. Tool Executor -> Reasoning Svc: "工具结果: {data}" (流式返回)
5. Reasoning Svc -> Memory Svc:   "保存推理状态" (Kafka 异步)
6. Reasoning Svc -> Gateway:      "思维 token: 正在分析..." (WebSocket)
```

---

## 三、通信协议对比

### 3.1 协议全景图

```
协议选择决策树:

服务间通信需要?
    |
    +-- 实时性强? 需要流?
    |       |
    |       +-- 是 --> 服务间? --> gRPC 双向流
    |       |              |-> 客户端? --> WebSocket
    |       +-- 否 --> 数据量大? --> gRPC
    |                    |-> 数据量小? --> REST
    |
    +-- 异步解耦? 需要削峰?
    |       |
    |       +-- 高吞吐持久化? --> Kafka
    |       |-> 轻量内存? --> Redis Streams
    |       |-> 任务分发? --> RabbitMQ
    |
    +-- 事件通知? 
            |
            +-- 广播? --> Redis Pub/Sub
            |-> 可靠? --> Kafka
```

### 3.2 gRPC (双向流式通信)

**适用场景**：Agent 推理过程流式传输、思维链(Chain-of-Thought)流式返回、Planner-Reasoner 之间流式协调

```
gRPC 在 Agent 系统中的核心优势:

传统 REST:
  POST /reason {prompt}
  200 OK {result}
  问题: 必须等待整个推理完成才能拿到结果

gRPC 双向流:
  client:  rpc Reason(stream Thought) returns (stream Thought)
  客户端流: 发送用户输入 + 工具结果
  服务端流: 逐步返回推理 token

  Time -->
  Client:  [UserInput] [ToolResult1] [ToolResult2]
  Server:  [Token1]...[TokenN] [Action] [Token1]...[TokenN]
```

**Protobuf 定义示例**：

```protobuf
syntax = "proto3";

package agent.reasoning.v1;

// ============================
// Agent 推理服务核心消息类型
// ============================

// --- 推理步骤类型枚举 ---
enum StepType {
  STEP_TYPE_UNSPECIFIED = 0;
  THOUGHT = 1;         // Agent 思考过程
  ACTION = 2;          // Agent 决定执行的动作
  OBSERVATION = 3;     // 工具执行后的观察结果
  FINAL_ANSWER = 4;    // 最终答案
  ERROR = 5;           // 错误状态
}

// --- 推理状态枚举 ---
enum ReasoningStatus {
  REASONING_STATUS_UNSPECIFIED = 0;
  IN_PROGRESS = 1;     // 推理进行中
  WAITING_TOOL = 2;    // 等待工具结果
  COMPLETED = 3;       // 推理完成
  FAILED = 4;          // 推理失败
  CANCELLED = 5;       // 推理被取消
}

// --- 推理步骤消息 ---
message ReasoningStep {
  string step_id = 1;           // 步骤唯一 ID (UUID)
  string session_id = 2;        // 会话 ID
  int32 sequence = 3;           // 步骤序号
  StepType step_type = 4;       // 步骤类型
  string content = 5;           // 步骤内容（JSON 序列化）
  map<string, string> metadata = 6;   // 元数据（token 数、耗时等）
  int64 timestamp = 7;          // Unix 时间戳（毫秒）
  string parent_step_id = 8;    // 父步骤 ID（用于嵌套推理）
  repeated string child_step_ids = 9;  // 子步骤 ID 列表
}

// --- 推理会话消息 ---
message ReasoningSession {
  string session_id = 1;
  string user_id = 2;
  string agent_id = 3;
  ReasoningStatus status = 4;
  int32 total_steps = 5;
  int64 started_at = 6;
  int64 updated_at = 7;
  map<string, string> context = 8;  // 推理上下文
}

// --- 工具调用消息 ---
message ToolCall {
  string tool_call_id = 1;
  string tool_name = 2;
  string arguments = 3;            // JSON 序列化的参数
  string result = 4;               // JSON 序列化的结果
  bool is_error = 5;
  int64 duration_ms = 6;
}

// ============================
// 服务定义
// ============================

service ReasoningService {
  // 双向流式推理 - Agent 核心通信模式
  rpc StreamReason(stream StreamReasonRequest) returns (stream StreamReasonResponse);
  
  // 一元推理 - 简单查询（不需要流式）
  rpc Reason(ReasonRequest) returns (ReasonResponse);
  
  // 获取推理会话状态
  rpc GetSession(GetSessionRequest) returns (ReasoningSession);
  
  // 取消推理
  rpc CancelReason(CancelReasonRequest) returns (CancelReasonResponse);
}

message StreamReasonRequest {
  oneof payload {
    string user_message = 1;       // 用户输入
    ToolCall tool_result = 2;      // 工具执行结果
    CancelSignal cancel = 3;       // 取消信号
    KeepAlive ping = 4;            // 心跳保活
  }
}

message StreamReasonResponse {
  oneof payload {
    ReasoningStep thought = 1;     // 推理步骤
    ToolCallRequest tool_call = 2; // 请求执行工具
    SessionUpdate session = 3;     // 会话状态更新
    ErrorInfo error = 4;           // 错误信息
    KeepAlive pong = 5;            // 心跳响应
  }
}

message ToolCallRequest {
  string tool_call_id = 1;
  string tool_name = 2;
  string arguments = 3;
  int64 timeout_ms = 4;
}

message ErrorInfo {
  string code = 1;
  string message = 2;
  string step_id = 3;
  bool is_recoverable = 4;
}

message KeepAlive {
  int64 timestamp = 1;
}
```

**Python gRPC 服务端实现**：

```python
import asyncio
import json
import uuid
from datetime import datetime
from typing import AsyncGenerator

import grpc
from google.protobuf.json_format import MessageToJson

# 假设由 protoc 生成
from agent_pb2 import (
    StreamReasonRequest, StreamReasonResponse,
    ReasoningStep, ToolCallRequest, SessionUpdate,
    StepType, ReasoningStatus, KeepAlive
)
from agent_pb2_grpc import ReasoningServiceServicer, add_ReasoningServiceServicer_to_server


class AgentReasoningServicer(ReasoningServiceServicer):
    """Agent 推理服务 - 处理流式推理请求"""

    def __init__(self, llm_client, tool_registry, memory_service):
        self.llm = llm_client
        self.tools = tool_registry
        self.memory = memory_service
        self.active_sessions: dict[str, dict] = {}

    async def StreamReason(
        self,
        request_iterator: AsyncGenerator[StreamReasonRequest, None],
        context: grpc.aio.ServicerContext,
    ) -> AsyncGenerator[StreamReasonResponse, None]:
        """
        双向流式推理 - Agent 的核心通信模式
        
        通信流程:
        Client -> Server: [用户消息, 工具结果1, 工具结果2, ...]
        Server -> Client: [Thought, ToolCall, Thought, ToolCall, FinalAnswer]
        """

        session_id = str(uuid.uuid4())
        self.active_sessions[session_id] = {
            "status": "in_progress",
            "started_at": datetime.utcnow(),
            "step_count": 0,
        }

        try:
            # 建立推理上下文
            reasoning_context = {"session_id": session_id, "history": []}

            # 启动推理循环
            async for step in self._reasoning_loop(reasoning_context, request_iterator):
                yield step

                # 检查客户端是否断开
                if context.cancelled():
                    await self._handle_cancellation(session_id)
                    break

        except grpc.aio.AioRpcError as e:
            yield StreamReasonResponse(
                error=ErrorInfo(
                    code="GRPC_ERROR",
                    message=f"gRPC 通信错误: {e.code()}",
                    is_recoverable=e.code() != grpc.StatusCode.INTERNAL,
                )
            )
        finally:
            self.active_sessions.pop(session_id, None)

    async def _reasoning_loop(
        self,
        context: dict,
        request_iterator: AsyncGenerator[StreamReasonRequest, None],
    ) -> AsyncGenerator[StreamReasonResponse, None]:
        """核心推理循环 - 每一步都产生流式输出"""

        context["step_count"] = 0
        pending_tool_call = None

        # 处理客户端流式输入
        async for request in request_iterator:
            which = request.WhichOneof("payload")

            if which == "user_message":
                # 处理用户消息 -> 开始推理
                async for response in self._generate_thoughts(
                    context, request.user_message
                ):
                    yield response

            elif which == "tool_result":
                # 处理工具执行结果 -> 继续推理
                async for response in self._process_tool_result(
                    context, request.tool_result
                ):
                    yield response

            elif which == "cancel":
                yield StreamReasonResponse(
                    session=SessionUpdate(
                        session_id=context["session_id"],
                        status=ReasoningStatus.CANCELLED,
                    )
                )
                return

            elif which == "ping":
                yield StreamReasonResponse(pong=KeepAlive(timestamp=0))

    async def _generate_thoughts(self, context, user_message):
        """生成推理思维链 - 每个 token 或步骤都流式返回"""

        llm_stream = self.llm.stream_chat(
            messages=[{"role": "user", "content": user_message}],
            tools=self.tools.get_definitions(),
        )

        thought_buffer = []
        async for chunk in llm_stream:
            # 将 LLM token 封装为 ReasoningStep
            context["step_count"] += 1
            step = ReasoningStep(
                step_id=str(uuid.uuid4()),
                session_id=context["session_id"],
                sequence=context["step_count"],
                step_type=StepType.THOUGHT,
                content=chunk.delta,
                metadata={
                    "token_count": str(len(chunk.delta)),
                    "timestamp": str(datetime.utcnow().timestamp()),
                },
            )
            thought_buffer.append(chunk.delta)

            yield StreamReasonResponse(thought=step)

            # 检测是否需要调用工具
            if chunk.tool_call:
                yield StreamReasonResponse(
                    tool_call=ToolCallRequest(
                        tool_call_id=str(uuid.uuid4()),
                        tool_name=chunk.tool_call.function.name,
                        arguments=json.dumps(chunk.tool_call.function.arguments),
                        timeout_ms=30000,
                    )
                )
                context["pending_tool"] = chunk.tool_call
                return  # 等待工具结果

    async def _process_tool_result(self, context, tool_result):
        """处理工具结果并继续推理"""

        # 将工具结果作为观察加入推理上下文
        observation_step = ReasoningStep(
            step_id=str(uuid.uuid4()),
            session_id=context["session_id"],
            sequence=context["step_count"] + 1,
            step_type=StepType.OBSERVATION,
            content=json.dumps({
                "tool": tool_result.tool_name,
                "result": tool_result.result,
                "error": tool_result.is_error,
            }),
            metadata={"duration_ms": str(tool_result.duration_ms)},
        )
        yield StreamReasonResponse(thought=observation_step)

        # 继续推理
        async for response in self._generate_thoughts(context, tool_result.result):
            yield response


async def serve_grpc(host="0.0.0.0", port=50051):
    """启动 gRPC 服务"""
    server = grpc.aio.server(
        maximum_concurrent_rpcs=100,
        options=[
            ("grpc.max_send_message_length", 50 * 1024 * 1024),  # 50MB
            ("grpc.max_receive_message_length", 50 * 1024 * 1024),
            ("grpc.keepalive_time_ms", 10000),  # 10s 心跳
            ("grpc.keepalive_timeout_ms", 5000),  # 5s 超时
        ],
    )

    servicer = AgentReasoningServicer(
        llm_client=LLMClient(),
        tool_registry=ToolRegistry(),
        memory_service=MemoryServiceClient(),
    )
    add_ReasoningServiceServicer_to_server(servicer, server)

    server.add_insecure_port(f"{host}:{port}")
    print(f"[gRPC] Agent Reasoning Service 启动在 {host}:{port}")

    await server.start()
    await server.wait_for_termination()


if __name__ == "__main__":
    asyncio.run(serve_grpc())
```

### 3.3 REST/HTTP (工具结果传输)

**适用场景**：工具执行结果返回、服务状态查询、简单 CRUD 操作

```
REST 在 Agent 系统中的定位:

+----------------+     HTTP POST /api/v1/tools/execute     +----------------+
| Reasoning Svc  | --------------------------------------> |  Tool Executor |
|                | <-------------------------------------- |                |
+----------------+     { result: "...", status: "ok" }     +----------------+

优点:
  - 生态成熟，调试方便 (curl, Postman)
  - 防火墙友好，不需要特殊网络配置
  - 适合工具执行这种"请求-响应"模式

缺点:
  - 无法流式传输推理 token
  - 请求头开销大 (HTTP/1.1 每次都要建连)
  - 单向通信，服务端无法主动推送
```

### 3.4 消息队列 (Kafka / RabbitMQ / Redis Streams)

**适用场景**：Agent 事件广播、异步协调、状态同步、日志持久化

```
Kafka 在 Agent 系统中的拓扑:

                    +------------------+
                    |  Topic: agent.   |
                    |  reasoning.events |
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
         +----v----+   +----v----+   +----v----+
         | Memory  |   | Monitor |   | Logging |
         | Service |   | Service |   | Service |
         +---------+   +---------+   +---------+
         
              +--------------+--------------+
              |              |              |
         +----v----+   +----v----+   +----v----+
         | Cache   |   | Audit   |   | Replay  |
         | Service |   | Service |   | Service |
         +---------+   +---------+   +---------+

Topic 设计:
  agent.reasoning.steps       -> 每一步推理
  agent.tool.executions       -> 工具调用事件
  agent.session.lifecycle     -> 会话生命周期
  agent.errors                -> 错误事件
  agent.metrics               -> 性能指标
```

**Python Kafka 异步事件发布/消费**：

```python
import asyncio
import json
import uuid
from datetime import datetime
from dataclasses import dataclass, field, asdict
from typing import Optional

from aiokafka import AIOKafkaProducer, AIOKafkaConsumer
from aiokafka.errors import KafkaError


# ============================
# Agent 事件数据模型
# ============================

@dataclass
class AgentEvent:
    """Agent 事件基类"""
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    event_type: str = ""
    session_id: str = ""
    agent_id: str = ""
    timestamp: int = field(default_factory=lambda: int(datetime.utcnow().timestamp() * 1000))
    payload: dict = field(default_factory=dict)

    def to_json(self) -> str:
        return json.dumps(asdict(self), default=str)

    @classmethod
    def from_json(cls, data: str) -> "AgentEvent":
        return cls(**json.loads(data))


@dataclass
class ReasoningStepEvent(AgentEvent):
    """推理步骤事件"""
    event_type: str = "reasoning.step"
    step_type: str = "thought"       # thought | action | observation
    step_content: str = ""
    token_count: int = 0
    duration_ms: int = 0


@dataclass
class ToolExecutionEvent(AgentEvent):
    """工具执行事件"""
    event_type: str = "tool.execution"
    tool_name: str = ""
    arguments: dict = field(default_factory=dict)
    result: dict = field(default_factory=dict)
    is_error: bool = False
    duration_ms: int = 0


@dataclass
class SessionLifecycleEvent(AgentEvent):
    """会话生命周期事件"""
    event_type: str = "session.lifecycle"
    action: str = "created"          # created | updated | completed | failed
    status: str = "in_progress"
    total_steps: int = 0
    error_message: Optional[str] = None


# ============================
# Kafka 事件发布器
# ============================

class AgentEventPublisher:
    """Agent 事件发布器 - 基于 Kafka"""

    TOPICS = {
        "reasoning.step": "agent.reasoning.steps",
        "tool.execution": "agent.tool.executions",
        "session.lifecycle": "agent.session.lifecycle",
        "error": "agent.errors",
    }

    def __init__(self, bootstrap_servers: str = "localhost:9092"):
        self.producer: Optional[AIOKafkaProducer] = None
        self.bootstrap_servers = bootstrap_servers

    async def start(self):
        """初始化生产者"""
        self.producer = AIOKafkaProducer(
            bootstrap_servers=self.bootstrap_servers,
            acks="all",                     # 等待所有副本确认
            compression_type="snappy",       # 压缩减少网络开销
            max_request_size=10 * 1024 * 1024,  # 10MB
            linger_ms=5,                     # 批量发送延迟 5ms
            batch_size=16384,                # 16KB 批量大小
        )
        await self.producer.start()
        print(f"[Kafka] 事件发布器已连接: {self.bootstrap_servers}")

    async def stop(self):
        """关闭生产者"""
        if self.producer:
            await self.producer.stop()

    async def publish(self, event: AgentEvent, key: Optional[str] = None):
        """
        发布事件到 Kafka
        
        Args:
            event: Agent 事件
            key: 分区键 (通常使用 session_id 确保同一个会话的消息在同一个分区)
        """
        topic = self.TOPICS.get(event.event_type.split(".")[-1], "agent.events.general")
        # 实际上更准确的做法: 从 event_type 映射
        topic = self._resolve_topic(event.event_type)

        try:
            # Kafka 要求 key 和 value 都是 bytes
            message_key = (key or event.session_id or event.event_id).encode("utf-8")
            message_value = event.to_json().encode("utf-8")

            future = await self.producer.send(
                topic=topic,
                key=message_key,
                value=message_value,
                timestamp_ms=event.timestamp,
            )

            # 等待发送确认
            record_metadata = await future
            return {
                "topic": record_metadata.topic,
                "partition": record_metadata.partition,
                "offset": record_metadata.offset,
            }

        except KafkaError as e:
            print(f"[Kafka] 发布事件失败: {e}")
            # 降级: 写入本地日志
            self._fallback_log(event)
            raise

    def _resolve_topic(self, event_type: str) -> str:
        """根据事件类型解析 topic 名称"""
        mapping = {
            "reasoning.step": "agent.reasoning.steps",
            "reasoning.step.thought": "agent.reasoning.steps",
            "reasoning.step.action": "agent.reasoning.steps",
            "reasoning.step.observation": "agent.reasoning.steps",
            "tool.execution": "agent.tool.executions",
            "tool.execution.start": "agent.tool.executions",
            "tool.execution.complete": "agent.tool.executions",
            "session.lifecycle": "agent.session.lifecycle",
            "session.created": "agent.session.lifecycle",
            "session.completed": "agent.session.lifecycle",
            "session.failed": "agent.session.lifecycle",
            "error": "agent.errors",
        }
        return mapping.get(event_type, "agent.events.general")

    def _fallback_log(self, event: AgentEvent):
        """Kafka 不可用时的降级处理"""
        import logging
        logger = logging.getLogger("agent.events.fallback")
        logger.error(f"Kafka unavailable, event dropped: {event.event_id} type={event.event_type}")


# ============================
# Kafka 事件消费者（Memory Service 示例）
# ============================

class MemoryEventConsumer:
    """Memory Service 的事件消费者 - 异步消费推理事件"""

    def __init__(self, bootstrap_servers: str, group_id: str = "memory-service"):
        self.consumer: Optional[AIOKafkaConsumer] = None
        self.bootstrap_servers = bootstrap_servers
        self.group_id = group_id
        self.handlers = {}

    def register_handler(self, event_type: str, handler):
        """注册事件处理器"""
        self.handlers[event_type] = handler

    async def start(self):
        """启动消费者"""
        self.consumer = AIOKafkaConsumer(
            "agent.reasoning.steps",
            "agent.tool.executions",
            "agent.session.lifecycle",
            "agent.errors",
            bootstrap_servers=self.bootstrap_servers,
            group_id=self.group_id,
            enable_auto_commit=True,
            auto_commit_interval_ms=5000,
            auto_offset_reset="earliest",  # 从头消费（用于状态恢复）
        )
        await self.consumer.start()
        print(f"[Kafka] Memory Service 消费者已启动, group_id={self.group_id}")

    async def stop(self):
        if self.consumer:
            await self.consumer.stop()

    async def consume_loop(self):
        """事件消费主循环"""
        try:
            async for msg in self.consumer:
                event = AgentEvent.from_json(msg.value.decode("utf-8"))
                handler = self.handlers.get(event.event_type)
                if handler:
                    try:
                        await handler(event)
                    except Exception as e:
                        print(f"[Memory] 处理事件失败: {e}, event_id={event.event_id}")
                        # 错误事件写入死信队列
                        await self._dead_letter(event, str(e))
                else:
                    print(f"[Memory] 未注册的事件类型: {event.event_type}")
        finally:
            await self.stop()

    async def _dead_letter(self, event: AgentEvent, error: str):
        """将处理失败的事件写入死信队列"""
        event.payload["dlq_error"] = error
        event.payload["dlq_timestamp"] = datetime.utcnow().isoformat()
        # 写入死信 topic
        await self.consumer._producer.send_and_wait(
            "agent.dead_letter_queue",
            key=event.event_id.encode(),
            value=event.to_json().encode(),
        )


# ============================
# Redis Streams 替代方案（轻量级）
# ============================

class RedisStreamEventBus:
    """基于 Redis Streams 的轻量级事件总线"""
    
    def __init__(self, redis_client):
        self.redis = redis_client
        self.maxlen = 10000  # 每个 stream 最大保留事件数

    async def publish(self, stream: str, event: AgentEvent):
        """发布事件到 Redis Stream"""
        await self.redis.xadd(
            stream,
            event.to_dict(),  # Redis 的 field-value 格式
            maxlen=self.maxlen,
            approximate=True,
        )

    async def consume(self, stream: str, group: str, consumer: str):
        """从 Redis Stream 消费事件"""
        while True:
            messages = await self.redis.xreadgroup(
                group, consumer,
                {stream: ">"},  # ">" 表示只读新消息
                count=10,
                block=2000,     # 阻塞 2s
            )
            if messages:
                for msg_id, msg_data in messages[0][1]:
                    event = AgentEvent(**msg_data)
                    yield event
                    await self.redis.xack(stream, group, msg_id)
```

### 3.5 WebSocket (实时流式推送到客户端)

**适用场景**：Agent 思维链实时展示、部分结果推送到前端、双向交互

```
WebSocket 在 Agent 系统中的定位:

+----------+    WebSocket    +----------+    gRPC 流    +------------+
| Browser  | <------------> | Gateway  | <------------> | Reasoning  |
| / Client |    思维 Token   |          |    推理步骤    | Service    |
+----------+                 +----------+               +------------+

协议转换:
  Client <-> Gateway:      WebSocket JSON 消息
  Gateway <-> Reasoning:   gRPC 双向流 (binary protobuf)

消息格式 (JSON):

客户端 -> 网关:
{
  "type": "user_message",
  "session_id": "sess-xxx",
  "content": "帮我写一个 Python 脚本"
}

网关 -> 客户端:
{
  "type": "thought_chunk",       // 推理 token 块
  "session_id": "sess-xxx",
  "sequence": 1,
  "content": "首先我需要理解用户需求...",
  "step_type": "thought",
  "metadata": {"token_count": 12}
}
{
  "type": "tool_call",           // 工具调用通知
  "session_id": "sess-xxx",
  "sequence": 2,
  "tool_name": "python_executor",
  "arguments": {"code": "print('hello')"}
}
{
  "type": "session_status",      // 会话状态更新
  "session_id": "sess-xxx",
  "status": "waiting_tool"
}
{
  "type": "error",               // 错误信息
  "session_id": "sess-xxx",
  "code": "TOOL_TIMEOUT",
  "message": "工具执行超时",
  "is_recoverable": true
}
```

---

## 四、同步 vs 异步通信

### 4.1 决策矩阵

| 场景 | 推荐模式 | 理由 |
|------|---------|------|
| Agent 推理步骤传输 | 流式同步 (gRPC 双向流) | 实时性要求高，部分结果需立即处理 |
| 工具执行结果返回 | 同步 (REST) 或流式 | 结果大小可预期，需要等待后继续推理 |
| 记忆持久化 | 异步 (Kafka/Redis) | 不阻塞推理循环，可批量写入 |
| 监控指标上报 | 异步 (Kafka) | 丢失可接受，吞吐量优先 |
| 会话状态同步 | 异步 (Pub/Sub) | 多个服务需要知道会话状态变更 |
| 规划服务调用 | 同步 (gRPC) | 需要规划结果后才能继续推理 |
| 日志收集 | 异步 (Kafka) | 大量、可延迟、可丢失 |
| 错误告警 | 异步 (消息队列) | 不阻塞主流程，但需可靠投递 |

### 4.2 Agent 推理中的时序控制

```
同步模式（推理循环核心路径）:

[Reasoning Svc] --gRPC 流--> [Planner]     -- 等待规划结果 --
[Reasoning Svc] <--gRPC 流-- [Planner]     -- 流式返回规划 --
[Reasoning Svc] --gRPC 流--> [Tool Exec]   -- 等待工具结果 --
[Reasoning Svc] <--gRPC 流-- [Tool Exec]   -- 流式返回结果 --
[Reasoning Svc] --gRPC 流--> [LLM]         -- 等待 LLM 响应 --
[Reasoning Svc] <--gRPC 流-- [LLM]         -- 流式返回 token --
                              ^
                              |--- 这里需要等待，但等待时有心跳保活

异步模式（非核心路径）:

[Reasoning Svc] --发布事件--> [Kafka]       -- 不等待 --
[Reasoning Svc] --继续推理--> [Next Step]   -- 记忆持久化在后台完成 --
                              ^
                              |--- 如果记忆服务落后，推理使用缓存状态
```

### 4.3 同步等待的超时与降级

```python
import asyncio
from contextlib import asynccontextmanager


class SyncCallWithTimeout:
    """带超时和降级的同步调用封装"""

    def __init__(self, default_timeout_ms=30000):
        self.default_timeout = default_timeout_ms / 1000

    @asynccontextmanager
    async def timeout_context(self, service_name: str, timeout_ms: int = None):
        """
        带超时的同步调用上下文
        
        使用方式:
            async with sync_call.timeout_context("ToolExecutor", 5000) as ctx:
                result = await call_tool()
                ctx.result = result
        """
        timeout = (timeout_ms or self.default_timeout) / 1000
        try:
            result = await asyncio.wait_for(
                self._tracked_call(service_name),
                timeout=timeout,
            )
            yield result
        except asyncio.TimeoutError:
            # 超时降级: 如果工具超时，返回缓存结果或错误
            print(f"[Timeout] {service_name} 调用超时 ({timeout}s)")
            fallback = await self._get_fallback(service_name)
            yield fallback

    async def _tracked_call(self, service_name):
        """带追踪的调用"""
        # 实际调用 + OpenTelemetry span
        pass

    async def _get_fallback(self, service_name):
        """获取降级响应"""
        fallbacks = {
            "ToolExecutor": {"result": "工具超时", "error": True},
            "MemoryService": {"status": "cache_only", "data": None},
            "Planner": {"steps": [], "fallback": True},
        }
        return fallbacks.get(service_name, {"error": "service_timeout"})
```

---

## 五、消息协议设计

### 5.1 Agent 消息类型体系

```
Agent 通信中的核心消息类型:

+------------------+
|    Envelope      |  消息信封 - 所有消息的通用包装
|  - message_id    |
|  - session_id    |
|  - service_from  |
|  - service_to    |
|  - timestamp     |
|  - trace_id      |  -> 分布式追踪
|  - message_type  |
+--------+---------+
         |
         +--+--------------------------------+
            |                                |
    +-------v--------+             +----------v--------+
    |   ControlMsg   |             |    DataMsg        |
    | - heart_beat   |             | - Thought         |
    | - cancel       |             | - Action          |
    | - error        |             | - Observation     |
    | - ack          |             | - ToolResult      |
    +----------------+             | - FinalAnswer     |
                                   | - SessionStatus   |
                                   +-------------------+
```

### 5.2 消息序列化开销对比

| 协议 | 序列化大小 (1KB 思维消息) | 序列化耗时 | 反序列化耗时 | 语言绑定 |
|------|--------------------------|-----------|------------|---------|
| Protobuf | ~200 bytes | 2μs | 3μs | 强类型 |
| JSON | ~1.2KB | 10μs | 15μs | 动态 |
| MessagePack | ~500 bytes | 5μs | 8μs | 动态 |
| Avro | ~250 bytes | 3μs | 4μs | 强类型 |
| Thrift | ~220 bytes | 2.5μs | 3.5μs | 强类型 |

**在 Agent 系统中的选择建议**：
- **服务间核心链路** (gRPC): Protobuf — 高性能、强类型、流式原生支持
- **事件总线** (Kafka): Avro 或 Protobuf — Schema Registry 确保兼容性
- **外部 API**: JSON — 兼容性最好，调试方便
- **实时推送** (WebSocket): JSON 或 MessagePack — 前端解析方便

### 5.3 Schema 兼容性设计

```protobuf
// Agent 消息的 Schema 演进策略 - 向后兼容

// v1 (初始版)
message Thought {
  string content = 1;          // 固定字段: 永不删除
  int32 sequence = 2;
}

// v2 (添加元数据)
message Thought {
  string content = 1;          // 保持不变
  int32 sequence = 2;          // 保持不变
  map<string, string> metadata = 3;  // 新字段: optional, 默认空
  repeated string tags = 4;          // 新字段: repeated 天然可选
}

// 兼容规则:
// 1. 永远不删除字段编号 (reserved)
// 2. 新字段使用新编号
// 3. 不改变已有字段的类型
// 4. 使用 optional 或默认值处理新字段缺失
```

---

## 六、最大挑战

### 6.1 网络延迟在推理循环中的影响

```
推理循环中的延迟累积:

用户输入
  |
  v
[Reasoning] --1. 调用 Planner-->  [Planner] (延迟: 5ms)
  |                              [Planner] (推理: 200ms)
  | <--2. 返回计划--------------  [Planner] (延迟: 5ms)
  |
  | --3. 第一次 LLM 调用--------> [LLM]     (延迟: 10ms + 生成: 3000ms)
  | <--4. 返回推理步骤----------- [LLM]     (延迟: 10ms)
  |
  | --5. 调用 Tool Executor-----> [Tool]    (延迟: 5ms + 执行: 500ms)
  | <--6. 返回工具结果----------- [Tool]    (延迟: 5ms)
  |
  | --7. 第二次 LLM 调用--------> [LLM]     (延迟: 10ms + 生成: 2000ms)
  | <--8. 返回最终答案----------- [LLM]     (延迟: 10ms)
  |
  v
最终响应

延迟分析:
  总延迟 = Σ(网络延迟) + Σ(服务处理延迟)
         = (5+5)+(5+5)+(5+5)+(5+5)+(5+5) + (200+3000+500+2000)
         = 50ms + 5700ms
         
  网络延迟占比: 50 / 5750 ≈ 0.87%
  
  结论: 在 Agent 系统中，LLM 推理延迟占主导 (>90%)，网络延迟影响相对较小
```

**关键洞察**：Agent 系统的主要延迟瓶颈是 LLM 推理本身，而非服务间通信。但这不意味着通信协议不重要——通信的**可靠性**和**流式能力**对用户体验至关重要。

### 6.2 部分失败处理

```python
class PartialFailureHandler:
    """Agent 推理中的部分失败处理策略"""

    STRATEGIES = {
        "tool_timeout": {
            "action": "retry_or_skip",
            "max_retries": 2,
            "fallback_message": "工具执行超时，已跳过"
        },
        "memory_unavailable": {
            "action": "use_local_cache",
            "max_retries": 0,
            "fallback_message": "记忆服务不可用，使用本地缓存"
        },
        "llm_error": {
            "action": "retry_with_backoff",
            "max_retries": 3,
            "fallback_message": "LLM 调用失败，正在重试"
        },
        "planner_error": {
            "action": "degrade_to_direct",
            "max_retries": 1,
            "fallback_message": "规划服务异常，降级为直接推理"
        },
    }

    def __init__(self):
        self.retry_counts = {}

    async def handle_failure(
        self, service: str, error: Exception, context: dict
    ) -> dict:
        """处理服务调用失败"""
        strategy = self.STRATEGIES.get(service, {})
        action = strategy.get("action", "fail_fast")

        if action == "retry_or_skip":
            return await self._retry_with_limit(service, error, context, strategy)
        elif action == "use_local_cache":
            return {"source": "local_cache", "data": context.get("local_cache", {})}
        elif action == "degrade_to_direct":
            return {"mode": "no_planner", "direct": True}
        elif action == "fail_fast":
            raise error  # 不可恢复的错误，直接向上抛
        return {"error": str(error), "is_recoverable": False}

    async def _retry_with_limit(self, service, error, context, strategy):
        """有限重试"""
        key = f"{service}:{context.get('session_id', 'unknown')}"
        self.retry_counts[key] = self.retry_counts.get(key, 0) + 1

        if self.retry_counts[key] <= strategy.get("max_retries", 2):
            wait = 2 ** self.retry_counts[key] * 0.1  # 指数退避
            await asyncio.sleep(wait)
            return {"action": "retry", "delay_ms": wait * 1000}
        else:
            # 超过重试次数，返回降级响应
            return {"action": "skip", "reason": strategy.get("fallback_message")}
```

### 6.3 消息序列化开销

```
序列化开销测试 (10,000 条 Agent Thought 消息):

Protobuf:
  序列化:  1.8μs/op  (总计 18ms)
  反序列化: 2.1μs/op  (总计 21ms)
  大小:    180 bytes/msg
  总带宽:  1.8MB

JSON:
  序列化:  8.5μs/op  (总计 85ms)
  反序列化: 12.3μs/op (总计 123ms)
  大小:    1.1KB/msg
  总带宽:  11MB

MessagePack:
  序列化:  4.2μs/op  (总计 42ms)
  反序列化: 5.8μs/op  (总计 58ms)
  大小:    0.4KB/msg
  总带宽:  4MB

结论:
  - 对于高频推理步骤流, Protobuf 节省约 80% 带宽和 5x 序列化时间
  - 对于低频工具结果, JSON 的便利性可以接受
  - WebSocket 到前端使用 JSON 因为浏览器原生支持
```

---

## 七、能力边界

### 7.1 什么时候通信开销大于收益

```
决策: 是否需要服务间通信?

问题: "我的 Agent 只有 3 个工具，日请求量 < 1000"

分析:
  单体架构: 1 个进程, 0 次网络调用
  微服务架构: N 个服务, M 次网络调用
  
  单体延迟: LLM 推理时间 + 工具执行时间
  微服务延迟: LLM 推理时间 + 工具执行时间 + 网络延迟
  
  额外的网络延迟: ~50ms (同机房内)
  
  当 LLM 推理时间 = 5s 时, 50ms 的额外延迟占比 1%
  收益: 独立扩展、团队自治、技术栈独立
  成本: 增加 1% 延迟 + 运维复杂性
  
  结论: 对于小规模 Agent, 单体更合适

何时通信成为瓶颈:
  - 推理步骤极多 (>100 步): 每次步骤间通信累积
  - 高频短推理 (<1s): 网络延迟占比 > 10%
  - 跨区域部署: 延迟从 5ms 上升到 50-200ms
  - 消息体极大 (工具返回 100MB+ 数据)
```

### 7.2 通信协议选择建议总结

```
小型 Agent 系统 (< 10 服务, < 1K QPS):
  - 服务间: gRPC (少量服务, 强类型, 流式)
  - 事件: Redis Streams (部署简单, 无需额外中间件)
  - 外部: REST/JSON (兼容性)

中型 Agent 系统 (10-50 服务, 1K-10K QPS):
  - 服务间: gRPC + 连接池
  - 事件: Kafka (持久化, 多消费者)
  - 外部: WebSocket (流式) + REST (管理)
  - 服务网格: Istio 基础 (mTLS, 流量管理)

大型 Agent 系统 (> 50 服务, > 10K QPS):
  - 服务间: gRPC + Service Mesh (Istio)
  - 事件: Kafka 多集群 (按领域拆分 Topic)
  - 外部: WebSocket + SSE + HTTP/2
  - 协议转换: Envoy 代理层处理
  - 自定义: 针对 Agent 流式通信的优化协议
```

---

## 八、总结

Agent 服务间通信协议的选择需要权衡**实时性**、**可靠性**、**吞吐量**和**复杂度**。核心原则：

1. **推理路径用 gRPC 双向流**：Agent 的思维过程天然是流式的，gRPC 提供了高效的流式传输和强类型约束
2. **事件驱动用 Kafka**：用于解耦 Agent 的非核心路径（记忆持久化、监控、日志），保证核心推理不被阻塞
3. **外部推送用 WebSocket**：实时向客户端推送推理 token, 提供更好的用户体验
4. **工具结果用 REST/gRPC**：工具调用是请求-响应模式，两种协议都适合
5. **序列化选 Protobuf**：服务间核心链路使用 Protobuf 减少带宽和序列化开销
6. **异步是 Agent 系统设计的第一原则**：核心推理路径之外的所有操作都应异步化，避免阻塞推理循环

在 Agent 系统中，通信协议的设计不仅关乎数据如何传输，更关乎推理如何组织、状态如何维护、失败如何恢复。选择适当的通信模式，是构建可靠、可扩展 Agent 系统的基础。
