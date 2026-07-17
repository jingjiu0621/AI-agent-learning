# 6.6.5 外部 API 记忆（External API Memory）

## 简单介绍

外部 API 记忆（External API Memory）是一种"不存储"的记忆模式——AI Agent 不将信息存放在本地记忆系统中，而是通过远程 API 调用，按需从外部系统获取或写入信息。这种模式的本质是将**存储责任转移给第三方服务**，Agent 本身只扮演"查询者"或"调用者"的角色。

与知识图谱（6.6.1）、数据库（6.6.3）、文档存储（6.6.4）等外部记忆模式不同，API 记忆的显著特征是：

- **Agent 不拥有数据**：数据存在于远端系统的管辖范围内，Agent 通过授权接口访问
- **无持久化负担**：不需要管理本地存储、索引、备份等基础设施
- **语义由 API 契约定义**：信息的含义和结构由 API 提供方决定，而非 Agent 自身
- **结果即答案**：API 返回的数据本身就是"记忆"，无需额外转换

典型场景：天气查询 Agent 调用气象 API 获取实时温度——它不需要"记住"天气，只需要知道"该调用哪个 API、参数怎么传"。

## 基本原理

外部 API 记忆的核心模式是 **查询→响应（Query → Response）**，而非传统记忆的 **存储→检索（Store → Retrieve）**。

```text
传统记忆模式：
  Agent → [存储数据] → [索引数据] → [检索数据] → 使用

API 记忆模式：
  Agent → [构造请求] → [发送请求] → [解析响应] → 使用
```

这个看似简单的差异带来了深刻的影响：

| 维度 | 传统记忆 | API 记忆 |
|------|----------|----------|
| 数据所有权 | Agent 拥有数据的副本 | 数据属于外部系统 |
| 一致性管理 | Agent 负责 | 外部系统负责（通常更强） |
| 存储成本 | 需要磁盘/内存空间 | 零本地存储 |
| 访问延迟 | 毫秒级（本地 I/O） | 毫秒到秒级（网络 I/O） |
| 数据新鲜度 | 取决于同步策略 | 始终实时（前提是 API 在线） |
| 可用性 | 依赖本地系统 | 依赖网络和远端服务 |

### API 即记忆接口

在外部 API 记忆范式中，API 是 Agent 感知外部世界的**感官器官**：

```
  [Agent]
     │
     ├─ API 调用 (query) ──→ [外部系统] ──→ 获取实时信息
     │                          │
     └─ API 调用 (mutate) ──→ [外部系统] ──→ 更新远端状态
```

每一次 API 调用，都是 Agent 从外部系统"读取一段记忆"或"写入一段记忆"的过程。

## 背景与演进

### 第一阶段：单体内部存储

AI 系统的早期记忆完全依赖内部存储。无论是基于规则的系统还是早期的机器学习模型，记忆都存储在本地文件、内存变量或单体数据库中。数据封闭在系统内部，不对外暴露接口。

### 第二阶段：分布式 API 存储

随着微服务架构和云计算的兴起，数据存储从单体走向分布式。系统开始通过 API 暴露存储能力：

```
  [应用层] → [REST API] → [存储层]
```

此阶段 API 是**应用组件之间的通信手段**，而非 Agent 的记忆模式。

### 第三阶段：工具调用范式（Tool Use）

LLM 的崛起催生了工具调用（Tool Use / Function Calling）范式。模型不再是单纯的文本生成器，而是能够调用外部工具来获取信息：

```
  [LLM] → [发现需要外部信息] → [调用工具函数] → [获取结果] → [融入推理]
```

OpenAI 的 Function Calling（2023 年 6 月）、Anthropic 的 Tool Use（2023 年底）、以及后续的 MCP（Model Context Protocol）协议，标志着 API 正式成为 Agent 的"记忆器官"。详见 [4.5 custom-tools](../4.5%20custom-tools/learn.md) 模块。

### 第四阶段：外部记忆 API

当前阶段，外部 API 记忆已从单纯的工具调用演变为体系化的记忆层。专门的 Agent 记忆服务（如 Mem0、Redis Agent Memory Server、SuperMemory）提供面向 Agent 的 API：

```
  [Agent] → [Memory API] → [记忆服务]
     │                         ├─ 语义搜索
     │                         ├─ 记忆存储
     │                         ├─ 记忆合并
     │                         └─ 遗忘管理
```

同时，MCP 协议的出现使得 Agent 可以通过标准化的接口发现和调用任意外部 API 作为记忆源，形成了 "API-as-Memory" 的完整生态。

## 核心矛盾

外部 API 记忆面临的根本矛盾是：**零本地存储成本 vs. 高访问延迟 + 网络依赖**。

```
成本收益权衡：
  
  优势面                    劣势面
  ┌─────────────────┐      ┌──────────────────────┐
  │ 零存储基础设施   │      │ 网络延迟（20-500ms）   │
  │ 无需数据备份     │      │ 依赖网络可用性         │
  │ 自动获得最新数据  │      │ API 速率限制           │
  │ 弹性伸缩（服务方）│      │ 每次调用有费用         │
  │ 安全责任外包     │      │ 供应商锁定             │
  └─────────────────┘      └──────────────────────┘
  
  决策核心：每次外部访问的成本能否被获取到的"信息价值"覆盖？
```

这一矛盾决定了以下设计原则：

1. **高频访问的信息不应通过 API 记忆获取**——应该缓存到本地
2. **延迟敏感的场景需要本地 fallback**——不能让 Agent 等待网络
3. **API 调用需要兜底策略**——API 宕机时要有降级方案

## 五种 API 记忆模式详解

### 1. 查询式记忆（Query Memory）

**定义**：通过 RESTful API 的 GET 请求从远程系统获取事实信息的模式。这是最基本、最广泛的 API 记忆模式。

**工作原理**：

```text
  Agent                     API 服务                     数据源
    │                          │                          │
    │──── GET /api/v1/users ───→│                          │
    │                          │───── 查询数据 ───────────→│
    │                          │←──── 返回结果 ───────────│
    │←── JSON Response ────────│                          │
    │                          │                          │
    │  {                       │                          │
    │    "id": 42,             │                          │
    │    "name": "Alice",      │                          │
    │    "role": "engineer"    │                          │
    │  }                       │                          │
```

**典型场景**：
- 用户信息查询：Agent 调用 CRM API 获取客户资料
- 知识库检索：Agent 调用 Wiki API 搜索相关文档
- 实时数据查询：Agent 调用股票/天气/交通 API

**特点**：
- 无副作用（只读）
- 结果可缓存（Cache-Control、ETag）
- 幂等（同一请求多次调用结果相同）

**Python 示例**：

```python
import httpx
from typing import Any
from datetime import datetime


class QueryMemoryAdapter:
    """
    查询式 API 记忆适配器 —— 通过 RESTful GET 请求获取"记忆"。
    """

    def __init__(self, base_url: str, api_key: str, timeout: float = 5.0):
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key
        self.client = httpx.AsyncClient(
            base_url=self.base_url,
            headers={"Authorization": f"Bearer {self.api_key}"},
            timeout=timeout,
        )

    async def recall(self, endpoint: str, params: dict[str, Any] | None = None) -> dict[str, Any]:
        """
        从外部 API 回忆/获取信息。

        Args:
            endpoint: API 路径，如 "/users/42"
            params: 查询参数，如 {"q": "keyword"}

        Returns:
            API 响应数据

        Raises:
            APIUnavailableError: API 不可用时的降级异常
        """
        try:
            response = await self.client.get(endpoint, params=params)
            response.raise_for_status()
            return response.json()
        except httpx.TimeoutException:
            raise APIUnavailableError("API 超时，无法获取记忆")
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 429:
                raise RateLimitError("请求过于频繁，请稍后重试")
            if e.response.status_code == 404:
                return {"error": "not_found", "message": "未找到相关信息"}
            raise APIUnavailableError(f"API 错误: {e.response.status_code}")

    async def close(self):
        await self.client.aclose()


class APIUnavailableError(Exception):
    """API 不可用时抛出的异常。"""

class RateLimitError(Exception):
    """API 速率限制异常。"""
```

---

### 2. 事务式记忆（Transactional Memory）

**定义**：通过 POST/PUT/PATCH/DELETE 等具有副作用的 API 操作，在外部系统中写入或修改状态的模式。Agent 通过 API 调用在远端留下"记忆痕迹"。

**工作原理**：

```text
  Agent                     API 服务                     数据库
    │                          │                          │
    │──── POST /api/orders ────→│                          │
    │     {                     │                          │
    │       "item": "book",    │                          │
    │       "qty": 2           │───── INSERT ────────────→│
    │     }                    │                          │
    │                          │←──── 返回 ID ───────────│
    │←── 201 Created ──────────│                          │
    │     { "order_id": 789 }  │                          │
```

**典型场景**：
- 订单创建：Agent 在电商系统创建订单
- 工单提交：Agent 在客服系统提交工单
- 状态更新：Agent 更新项目管理工具的任务状态
- 消息发送：Agent 通过消息 API 发送通知

**特点**：
- 有副作用（非幂等，需谨慎调用）
- 需要事务语义（确认、回滚）
- 通常需要审计日志

**Python 示例**：

```python
import httpx
from dataclasses import dataclass
from typing import Optional


@dataclass
class TransactionResult:
    """事务操作结果。"""
    success: bool
    remote_id: Optional[str] = None
    error: Optional[str] = None


class TransactionalMemoryAdapter:
    """
    事务式 API 记忆适配器 —— 通过有副作用的 API 在远端系统写入记忆。
    """

    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url.rstrip("/")
        self.client = httpx.Client(
            base_url=self.base_url,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            timeout=10.0,
        )

    def remember(  # 通过 API 写入"记忆"
        self, endpoint: str, data: dict, idempotency_key: Optional[str] = None
    ) -> TransactionResult:
        """
        通过 POST 请求在远端系统记录信息。

        Args:
            endpoint: API 路径
            data: 要存储的数据
            idempotency_key: 幂等键，防止重复提交

        Returns:
            操作结果
        """
        headers = {}
        if idempotency_key:
            headers["Idempotency-Key"] = idempotency_key

        try:
            response = self.client.post(endpoint, json=data, headers=headers)
            if response.status_code in (200, 201):
                result = response.json()
                return TransactionResult(
                    success=True,
                    remote_id=result.get("id"),
                )
            return TransactionResult(success=False, error=response.text)
        except httpx.RequestError as e:
            return TransactionResult(success=False, error=str(e))

    def forget(self, endpoint: str) -> TransactionResult:
        """
        通过 DELETE 请求从远端系统删除信息。
        """
        try:
            response = self.client.delete(endpoint)
            if response.status_code in (200, 204):
                return TransactionResult(success=True)
            return TransactionResult(success=False, error=response.text)
        except httpx.RequestError as e:
            return TransactionResult(success=False, error=str(e))
```

**幂等性最佳实践**：

始终为写操作提供幂等键（Idempotency-Key），确保网络重试不会导致重复操作：

```python
import uuid

# 为每个写操作生成唯一幂等键
idem_key = str(uuid.uuid4())

result = txn_memory.remember(
    "/api/orders",
    {"item": "book", "qty": 2},
    idempotency_key=idem_key,
)
```

---

### 3. 订阅式记忆（Subscription Memory）

**定义**：通过 WebSocket、SSE（Server-Sent Events）或 Webhook 等推送机制，实时获取外部系统信息的模式。Agent 不需要主动查询，信息会"推"到 Agent 面前。

**工作原理**：

```text
  Agent                     API 服务                     数据源
    │                          │                          │
    │──── WebSocket Connect ───→│                          │
    │                          │                          │
    │←── 实时推送数据 ──────────│                          │
    │    { event: "price_change" }                         │
    │                          │                          │
    │←── 实时推送数据 ──────────│                          │
    │    { event: "alert_triggered" }                     │
    │                          │                          │
    │──── 或注册 Webhook ──────→│                          │
    │    { callback_url: ... }  │                          │
```

**典型场景**：
- 实时行情：Agent 通过 WebSocket 接收股票价格更新
- 事件驱动：Agent 监听 CI/CD 流水线的状态变更事件
- 通知推送：Agent 通过 Webhook 接收即时消息通知
- 流式数据：Agent 订阅 IoT 设备的传感器数据流

**特点**：
- 实时性最高（推送 vs 轮询）
- 需要维持长连接
- 适合持续运行的 Agent

**Python 示例**：

```python
import asyncio
import json
from typing import Callable, Awaitable
from websockets.asyncio.client import connect


class SubscriptionMemoryAdapter:
    """
    订阅式 API 记忆适配器 —— 通过 WebSocket 实时接收远端"记忆"。
    """

    def __init__(self, ws_url: str, token: str):
        self.ws_url = ws_url
        self.token = token
        self._handlers: dict[str, list[Callable[[dict], Awaitable[None]]]] = {}
        self._running = False

    def on(self, event_type: str, handler: Callable[[dict], Awaitable[None]]):
        """注册事件处理器。"""
        self._handlers.setdefault(event_type, []).append(handler)

    async def listen(self):
        """持续监听 WebSocket 推送。"""
        self._running = True
        async with connect(
            self.ws_url,
            additional_headers={"Authorization": f"Bearer {self.token}"},
        ) as websocket:
            while self._running:
                message = await websocket.recv()
                data = json.loads(message)
                event_type = data.get("type", "*")
                # 分发给注册的处理器
                for handler in self._handlers.get(event_type, []):
                    asyncio.create_task(handler(data))
                for handler in self._handlers.get("*", []):
                    asyncio.create_task(handler(data))

    def stop(self):
        self._running = False


# 使用示例
async def handle_price_change(data: dict):
    print(f"价格变动: {data['symbol']} → ${data['price']}")


async def main():
    adapter = SubscriptionMemoryAdapter(
        ws_url="wss://api.example.com/ws",
        token="your-token",
    )
    adapter.on("price_change", handle_price_change)
    await adapter.listen()
```

---

### 4. GraphQL 记忆（GraphQL Memory）

**定义**：通过 GraphQL 查询语言精确获取外部图结构数据的模式。与 RESTful API 不同，GraphQL 允许 Agent 在**一次查询中精确指定需要哪些字段**，避免 Over-fetching 和 Under-fetching。

**工作原理**：

```text
REST 方式（Over-fetching）:
  GET /api/users/42
  → 返回 { id, name, email, address, phone, avatar, created_at, ... }
  → Agent 只需要 name 和 email，但收到了全部字段

GraphQL 方式（精确查询）:
  POST /graphql
  query {
    user(id: 42) {
      name
      email
    }
  }
  → 返回 { "data": { "user": { "name": "Alice", "email": "alice@example.com" } } }
  → Agent 只收到需要的字段
```

**典型场景**：
- 多实体关联查询：Agent 同时获取用户、订单、商品信息
- 嵌套数据需求：Agent 需要"用户的好友列表的收藏文章"
- 移动/低带宽场景：需要最小化数据传输量

**特点**：
- 精确字段选择——减少传输量和解析成本
- 单次查询替代多次 REST 调用
- 类型安全——Schema 定义了所有可能查询

**Python 示例**：

```python
import httpx
from typing import Any


class GraphQLMemoryAdapter:
    """
    GraphQL 记忆适配器 —— 通过精确查询从远端图获取记忆。
    """

    def __init__(self, endpoint: str, api_key: str):
        self.endpoint = endpoint
        self.client = httpx.Client(
            base_url=endpoint,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            timeout=10.0,
        )

    async def query(
        self,
        query_str: str,
        variables: dict[str, Any] | None = None,
    ) -> dict[str, Any]:
        """
        执行 GraphQL 查询。

        Args:
            query_str: GraphQL 查询语句
            variables: 查询变量

        Returns:
            结构化的查询结果
        """
        payload = {"query": query_str}
        if variables:
            payload["variables"] = variables

        response = await self.client.post("", json=payload)
        response.raise_for_status()
        result = response.json()

        if "errors" in result:
            raise GraphQLError(result["errors"])

        return result["data"]

    async def recall_user_with_orders(
        self, user_id: str
    ) -> dict[str, Any]:
        """
        通过一次 GraphQL 查询获取"用户 + 订单"的复合记忆。
        相当于 REST 方式的 2 次 API 调用。
        """
        query_str = """
        query GetUserWithOrders($userId: ID!) {
            user(id: $userId) {
                id
                name
                email
                orders(limit: 10) {
                    id
                    status
                    total
                    items {
                        name
                        quantity
                    }
                }
            }
        }
        """
        return await self.query(query_str, variables={"userId": user_id})


class GraphQLError(Exception):
    def __init__(self, errors: list[dict]):
        messages = [e.get("message", str(e)) for e in errors]
        super().__init__("; ".join(messages))
```

---

### 5. gRPC 记忆（gRPC Memory）

**定义**：通过 gRPC 协议进行高性能、结构化数据交换的模式。gRPC 使用 Protocol Buffers 作为接口定义语言和数据序列化格式，适用于对延迟和吞吐量要求极高的场景。

**工作原理**：

```text
Agent（gRPC Client）                    gRPC Server
    │                                      │
    │──── HTTP/2 Stream ──────────────────→│
    │     Protobuf 序列化请求               │
    │                                      │
    │←── Protobuf 序列化响应 ──────────────│
    │                                      │
  优势：
  - HTTP/2 多路复用（减少连接开销）
  - Protobuf 二进制编码（比 JSON 快 3-10 倍）
  - 强类型接口定义（.proto 文件）
  - 支持流式传输（Server/Client/Bidirectional Streaming）
```

**典型场景**：
- 低延迟记忆查询：毫秒级记忆检索
- 大规模数据交换：批量记忆同步
- 流式记忆推送：持续的数据流处理
- 微服务间记忆共享：Agent 子服务之间的高效通信

**特点**：
- 最高性能——二进制协议、HTTP/2 多路复用
- 强类型契约——.proto 文件定义接口和数据模型
- 内置流式支持——单向/双向流
- 生态成熟——负载均衡、认证、监控一应俱全

**Python 示例**（使用 grpcio）：

```python
# 首先定义 .proto 文件：
#
#   syntax = "proto3";
#   service MemoryService {
#       rpc Recall (MemoryRequest) returns (MemoryResponse);
#       rpc StreamMemories (MemoryStreamRequest) returns (stream MemoryEvent);
#   }
#   message MemoryRequest {
#       string agent_id = 1;
#       string query = 2;
#       int32 max_results = 3;
#   }
#   message MemoryResponse {
#       repeated MemoryItem items = 1;
#   }

import grpc
from typing import AsyncIterator
import memory_pb2
import memory_pb2_grpc


class GRPCMemoryAdapter:
    """
    gRPC 记忆适配器 —— 通过高性能 gRPC 协议访问远端记忆。
    """

    def __init__(self, server_address: str):
        self.channel = grpc.aio.insecure_channel(
            server_address,
            options=[
                ("grpc.max_send_message_length", 10 * 1024 * 1024),
                ("grpc.max_receive_message_length", 10 * 1024 * 1024),
                ("grpc.keepalive_time_ms", 10000),
            ],
        )
        self.stub = memory_pb2_grpc.MemoryServiceStub(self.channel)

    async def recall(
        self, agent_id: str, query: str, max_results: int = 10
    ) -> list[dict]:
        """
        通过 gRPC 调用从远端检索记忆。
        """
        request = memory_pb2.MemoryRequest(
            agent_id=agent_id,
            query=query,
            max_results=max_results,
        )
        response = await self.stub.Recall(request)
        return [
            {"id": item.id, "content": item.content, "score": item.score}
            for item in response.items
        ]

    async def stream_memories(
        self, agent_id: str
    ) -> AsyncIterator[dict]:
        """
        通过 gRPC 流式接口持续接收记忆事件。
        """
        request = memory_pb2.MemoryStreamRequest(agent_id=agent_id)
        async for event in self.stub.StreamMemories(request):
            yield {
                "type": event.event_type,
                "data": event.payload,
                "timestamp": event.timestamp.ToDatetime(),
            }

    async def close(self):
        await self.channel.close()
```

### 五类模式决策矩阵

```text
决策因子               查询式     事务式     订阅式     GraphQL    gRPC
─────────────────────────────────────────────────────────────────────────
延迟要求                 中        高         低         中        极低
数据量级                 小        小         持续       中        大
实时性需求               低        低         高         低        中
查询复杂度               低        低         低         高        中
传输效率                 中        中         中         高        极高
协议开销                 低        低         中         中        极低
实现复杂度               低        低         中         中        高
```

## API 记忆 vs 本地存储对比

```text
对比维度           本地存储             API 记忆               API 记忆风险点
────────────────────────────────────────────────────────────────────────────
延迟                1-10ms              20-500ms               网络波动时 >1s
一致性             强一致（本地 ACID）   最终一致（取决于远端）  远端事务不透明
可用性             高（99.99%）         中（99.9% 取决于 SLA）  Provider 宕机
存储成本           磁盘/内存费用        按调用次数付费          高频调用成本激增
数据新鲜度         取决于同步频率       始终实时                但缓存可能返回旧数据
安全性             需要自身防护         信任链 + 认证           数据泄露风险
扩展性             垂直扩展瓶颈         水平弹性扩展            费用也随调用量扩展
维护成本           需要 DBA             零维护（供应商负责）    供应商锁定
```

### 何时选择 API 记忆

```text
选择 API 记忆，当：
  ✔ 数据已经存在于某个外部系统中（无需重复存储）
  ✔ 需要实时/最新数据（API 返回即最新）
  ✔ 本地存储基础设施成本高或不可用
  ✔ 数据的权威源头是第三方系统
  ✔ 你信任外部系统的可用性和安全性

选择本地存储，当：
  ✔ 延迟是首要约束（亚毫秒级响应）
  ✔ 需要离线可用（网络可能断开）
  ✔ 数据量大且频繁访问（API 调用成本过高）
  ✔ 数据敏感不允许外传（隐私合规）
  ✔ 需要复杂的事务和一致性保证
```

## 实现挑战详解

### 1. 认证与授权（Authentication & Authorization）

API 记忆面临的首要挑战是身份验证。Agent 必须安全地管理凭证并处理不同的认证机制。

**常见认证方式**：

```text
认证方式        机制                          适用场景
────────────────────────────────────────────────────────
API Key         请求头携带静态密钥              简单查询接口
Bearer Token    JWT / OAuth 2.0 token         用户上下文相关
OAuth 2.0       授权码流程 + refresh token     代表用户操作
mTLS            双向 TLS 证书                  高安全内部服务
Service Account 云平台服务身份                  云端 Agent
```

**最佳实践**：

```python
import os
from datetime import datetime, timedelta
import jwt  # PyJWT


class AuthManager:
    """
    API 认证管理器 —— 集中管理凭证和 token 生命周期。
    """

    def __init__(self):
        self._tokens: dict[str, dict] = {}  # service -> {token, expires_at}

    def get_headers(self, service: str) -> dict[str, str]:
        """获取指定服务的认证请求头。"""
        if service == "github":
            return {"Authorization": f"Bearer {os.environ['GITHUB_TOKEN']}"}
        elif service == "internal":
            return self._get_internal_auth()
        raise ValueError(f"Unknown service: {service}")

    def _get_internal_auth(self) -> dict[str, str]:
        """使用 JWT 做内部服务间认证。"""
        now = datetime.utcnow()
        if (
            service := self._tokens.get("internal")
        ) and service["expires_at"] > now:
            return {"Authorization": f"Bearer {service['token']}"}

        # 生成新 token（5 分钟有效期）
        payload = {
            "sub": "agent-service",
            "iat": now,
            "exp": now + timedelta(minutes=5),
        }
        token = jwt.encode(payload, os.environ["JWT_SECRET"], algorithm="HS256")
        self._tokens["internal"] = {"token": token, "expires_at": now + timedelta(minutes=5)}
        return {"Authorization": f"Bearer {token}"}
```

### 2. 速率限制（Rate Limiting）

几乎所有公开 API 都有速率限制。Agent 必须优雅地处理 429 Too Many Requests 响应。

**应对策略**：

```python
import asyncio
import time
from collections import deque


class RateLimiter:
    """
    令牌桶速率限制器 —— 控制 API 调用频率。
    """

    def __init__(self, max_calls: int, period: float = 1.0):
        self.max_calls = max_calls
        self.period = period
        self.calls: deque[float] = deque()

    async def acquire(self):
        """获取调用许可，必要时等待。"""
        now = time.monotonic()
        # 清理过期的调用记录
        while self.calls and self.calls[0] < now - self.period:
            self.calls.popleft()

        if len(self.calls) >= self.max_calls:
            # 等待最早调用的冷却期结束
            wait_time = self.calls[0] + self.period - now
            if wait_time > 0:
                await asyncio.sleep(wait_time)

        self.calls.append(time.monotonic())

    async def __aenter__(self):
        await self.acquire()
        return self

    async def __aexit__(self, *args):
        pass


# 使用示例
rate_limiter = RateLimiter(max_calls=10, period=1.0)  # 每秒最多 10 次

async def safe_api_call(url: str):
    async with rate_limiter:
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
            # 处理 429
            if response.status_code == 429:
                retry_after = int(response.headers.get("Retry-After", "5"))
                await asyncio.sleep(retry_after)
                return await safe_api_call(url)
            response.raise_for_status()
            return response.json()
```

### 3. 错误处理（Retry, Fallback, Circuit Breaker）

网络不可靠，API 会宕机。Agent 需要多层次的容错策略。

**三级容错模型**：

```text
第一级：Retry（重试）
  ┌─────────────┐   失败    ┌──────────────┐
  │ API 调用尝试 │───→──────│ 指数退避重试   │──→ 成功
  └─────────────┘           │ 最多 3 次     │
                            └──────────────┘
                                  │ 仍失败
                                  ↓
第二级：Fallback（降级）
                            ┌──────────────┐
                            │ 使用缓存数据   │──→ 返回缓存（可能过时）
                            └──────────────┘
                                  │ 无缓存
                                  ↓
第三级：Circuit Breaker（熔断）
                            ┌──────────────┐
                            │ 停止调用 API   │──→ 直接返回错误
                            │ 冷却期后恢复   │    或默认值
                            └──────────────┘
```

**Python 实现**：

```python
import asyncio
import random
from datetime import datetime, timedelta
from typing import Any, Callable, Awaitable


class CircuitBreaker:
    """
    断路器 —— 防止 Agent 反复调用已宕机的 API。
    """

    STATE_CLOSED = "CLOSED"       # 正常，允许调用
    STATE_OPEN = "OPEN"           # 熔断，拒绝调用
    STATE_HALF_OPEN = "HALF_OPEN" # 半开，尝试恢复

    def __init__(self, failure_threshold: int = 5, recovery_timeout: float = 30.0):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = self.STATE_CLOSED
        self.failure_count = 0
        self.last_failure_time: datetime | None = None

    async def call(self, fn: Callable[[], Awaitable[Any]], fallback: Any = None) -> Any:
        """安全地调用 API，熔断状态下直接返回 fallback。"""
        if self.state == self.STATE_OPEN:
            # 检查是否可以进入半开状态
            if self.last_failure_time and (
                datetime.utcnow() - self.last_failure_time
            ).total_seconds() >= self.recovery_timeout:
                self.state = self.STATE_HALF_OPEN
            else:
                return fallback

        try:
            result = await fn()
            # 调用成功，重置
            self.failure_count = 0
            self.state = self.STATE_CLOSED
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = datetime.utcnow()
            if self.failure_count >= self.failure_threshold:
                self.state = self.STATE_OPEN
            return fallback


class MemoryAPIClient:
    """
    具有完整容错能力的 API 记忆客户端。
    """

    def __init__(self, base_url: str, cache: dict[str, Any] | None = None):
        self.base_url = base_url
        self.cache = cache or {}
        self.circuit_breaker = CircuitBreaker(failure_threshold=5, recovery_timeout=30.0)

    async def recall_with_resilience(self, key: str) -> Any:
        """带容错的 API 记忆调用。"""
        async def api_call():
            async with httpx.AsyncClient() as client:
                resp = await client.get(f"{self.base_url}/memory/{key}")
                resp.raise_for_status()
                return resp.json()

        result = await self.circuit_breaker.call(api_call, fallback=self.cache.get(key))
        if result is None:
            raise MemoryUnavailableError(f"无法获取记忆: {key}")
        return result


class MemoryUnavailableError(Exception):
    """记忆不可用异常。"""
```

### 4. 结果缓存（Response Caching）

为减少 API 调用次数和延迟，Agent 可以对 API 结果做本地缓存。

**缓存策略**：

```python
import time
from collections import OrderedDict
from typing import Any, Optional


class TTLCache:
    """
    TTL 缓存 —— 带过期时间的 API 响应缓存。
    """

    def __init__(self, default_ttl: float = 60.0, max_size: int = 1000):
        self.default_ttl = default_ttl
        self.max_size = max_size
        self._store: dict[str, tuple[float, Any]] = {}
        self._lru: OrderedDict[str, None] = OrderedDict()

    def get(self, key: str) -> Optional[Any]:
        """获取缓存。"""
        if key not in self._store:
            return None
        expires_at, value = self._store[key]
        if time.monotonic() > expires_at:
            del self._store[key]
            self._lru.pop(key, None)
            return None
        # 更新 LRU 位置
        self._lru.move_to_end(key)
        return value

    def set(self, key: str, value: Any, ttl: Optional[float] = None):
        """设置缓存。"""
        if len(self._store) >= self.max_size:
            # 淘汰最久未使用的
            oldest = next(iter(self._lru))
            del self._store[oldest]
            self._lru.pop(oldest, None)
        self._store[key] = (time.monotonic() + (ttl or self.default_ttl), value)
        self._lru[key] = None
        self._lru.move_to_end(key)

    def invalidate(self, key: str):
        """使缓存失效。"""
        self._store.pop(key, None)
        self._lru.pop(key, None)
```

### 5. 数据格式转换（API Response → Memory Format）

不同外部 API 返回的数据格式各异，Agent 需要统一的转换层。

```python
from abc import ABC, abstractmethod
from typing import Any


class MemoryNormalizer(ABC):
    """
    API 响应规范化器 —— 将不同 API 的响应转换为统一的记忆格式。
    """

    @abstractmethod
    def normalize(self, raw_response: dict[str, Any]) -> dict[str, Any]:
        """将原始 API 响应转为标准记忆格式。"""
        ...


class GitHubAPINormalizer(MemoryNormalizer):
    """GitHub API 响应 → 统一记忆格式。"""

    def normalize(self, raw: dict[str, Any]) -> dict[str, Any]:
        return {
            "source": "github",
            "entity_type": "repository",
            "id": str(raw["id"]),
            "name": raw["full_name"],
            "description": raw.get("description", ""),
            "url": raw["html_url"],
            "metadata": {
                "stars": raw.get("stargazers_count", 0),
                "language": raw.get("language"),
                "updated_at": raw.get("updated_at"),
            },
            "raw": raw,  # 保留原始数据作为可选项
        }


class WeatherAPINormalizer(MemoryNormalizer):
    """天气 API 响应 → 统一记忆格式。"""

    def normalize(self, raw: dict[str, Any]) -> dict[str, Any]:
        return {
            "source": "weather",
            "entity_type": "forecast",
            "id": f"{raw['location']}_{raw['date']}",
            "location": raw["location"],
            "temperature": raw["current"]["temp_c"],
            "condition": raw["current"]["condition"]["text"],
            "metadata": {
                "humidity": raw["current"]["humidity"],
                "wind_kph": raw["current"]["wind_kph"],
                "fetched_at": raw.get("timestamp"),
            },
        }


class MemoryFormatTransformer:
    """
    统一的记忆格式转换器。
    将各种外部 API 的响应标准化为 Agent 可消费的记忆格式。
    """

    def __init__(self):
        self._normalizers: dict[str, MemoryNormalizer] = {}

    def register(self, api_source: str, normalizer: MemoryNormalizer):
        self._normalizers[api_source] = normalizer

    def transform(self, api_source: str, raw_response: dict[str, Any]) -> dict[str, Any]:
        normalizer = self._normalizers.get(api_source)
        if not normalizer:
            raise ValueError(f"未注册的 API 源: {api_source}")
        return normalizer.normalize(raw_response)
```

## 完整 API 记忆适配器模式

将上述所有模式组合为一个统一的适配器：

```python
import asyncio
import logging
from typing import Any, Optional

logger = logging.getLogger(__name__)


class APIMemoryAdapter:
    """
    统一的外部 API 记忆适配器。

    整合了认证、速率限制、断路器、缓存、格式转换等所有关注点。
    """

    def __init__(
        self,
        name: str,
        base_url: str,
        api_key: str,
        normalizer: MemoryNormalizer,
        cache_ttl: float = 60.0,
        rate_limit: int = 30,
    ):
        self.name = name
        self.base_url = base_url.rstrip("/")
        self.normalizer = normalizer
        self.cache = TTLCache(default_ttl=cache_ttl)
        self.rate_limiter = RateLimiter(max_calls=rate_limit, period=1.0)
        self.circuit_breaker = CircuitBreaker(failure_threshold=5)
        self._client = httpx.AsyncClient(
            base_url=self.base_url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=10.0,
        )

    async def recall(self, endpoint: str, params: dict | None = None) -> dict[str, Any]:
        """
        从外部 API 记忆中检索信息。

        集成：缓存 → 限速 → 断路器 → API 调用 → 格式转换 → 缓存写入
        """
        cache_key = f"{endpoint}:{str(params)}"

        # 1. 检查缓存
        cached = self.cache.get(cache_key)
        if cached is not None:
            logger.debug(f"[{self.name}] 缓存命中: {cache_key}")
            return cached

        # 2. 通过断路器 + 限速器调用 API
        async def api_call():
            async with self.rate_limiter:
                response = await self._client.get(endpoint, params=params)
                response.raise_for_status()
                raw = response.json()
                # 3. 格式转换
                normalized = self.normalizer.normalize(raw)
                # 4. 写入缓存
                self.cache.set(cache_key, normalized)
                return normalized

        result = await self.circuit_breaker.call(api_call, fallback=self.cache.get(cache_key))
        if result is None:
            raise MemoryUnavailableError(
                f"[{self.name}] 无法获取记忆，且无可用缓存"
            )
        return result

    async def remember(self, endpoint: str, data: dict) -> bool:
        """
        向外部 API 写入记忆。
        """
        try:
            async with self.rate_limiter:
                response = await self._client.post(endpoint, json=data)
                response.raise_for_status()
                # 写入成功后，使相关缓存失效
                self.cache.invalidate(endpoint)
                return True
        except httpx.HTTPError as e:
            logger.error(f"[{self.name}] 写入失败: {e}")
            return False

    async def close(self):
        await self._client.aclose()


# 使用示例
async def demo():
    # 配置天气查询适配器
    weather_adapter = APIMemoryAdapter(
        name="weather-api",
        base_url="https://api.weather.com",
        api_key="your-api-key",
        normalizer=WeatherAPINormalizer(),
        cache_ttl=300.0,    # 天气缓存 5 分钟
        rate_limit=10,      # 每秒 10 次
    )

    # 查询天气
    try:
        memory = await weather_adapter.recall(
            "/v1/current",
            params={"location": "Beijing"},
        )
        print(f"北京当前温度: {memory['temperature']}°C")
    except MemoryUnavailableError as e:
        print(f"记忆不可用: {e}")
    finally:
        await weather_adapter.close()
```

## 工程优化

### 1. 连接池（Connection Pooling）

复用 HTTP 连接减少握手开销：

```python
# httpx 默认使用连接池，可通过以下方式优化
client = httpx.AsyncClient(
    limits=httpx.Limits(
        max_keepalive_connections=20,   # 最大保持存活连接数
        max_connections=100,            # 最大并发连接数
        keepalive_expiry=60.0,          # 连接保持时间
    ),
)
```

### 2. 请求批处理（Request Batching）

将多个独立 API 请求合并为批量请求：

```python
class BatchAPIRequester:
    """
    API 请求批处理器 —— 合并短时间内对同一端点的请求。
    """

    def __init__(self, batch_window: float = 0.05):
        self.batch_window = batch_window
        self._queue: list[tuple[str, dict, asyncio.Future]] = []
        self._task: asyncio.Task | None = None

    async def submit(self, endpoint: str, params: dict) -> Any:
        """提交单个请求，并在批处理窗口中合并。"""
        future = asyncio.get_event_loop().create_future()
        self._queue.append((endpoint, params, future))

        if self._task is None or self._task.done():
            self._task = asyncio.create_task(self._flush())

        return await future

    async def _flush(self):
        await asyncio.sleep(self.batch_window)
        batch = self._queue[:]
        self._queue.clear()

        # 按 endpoint 分组并发
        groups: dict[str, list] = {}
        for endpoint, params, future in batch:
            groups.setdefault(endpoint, []).append((params, future))

        async with httpx.AsyncClient() as client:
            for endpoint, items in groups.items():
                # 批量调用（取决于 API 是否支持 batch）
                ids = [p.get("id") for p, _ in items]
                response = await client.get(endpoint, params={"ids": ",".join(ids)})
                data = response.json()
                # 分发结果
                for (params, future), item in zip(items, data):
                    future.set_result(item)
```

### 3. 响应缓存与 TTL 策略

```text
数据类型          推荐 TTL         原因
───────────────────────────────────────────
用户资料          5-30 分钟       相对稳定
天气数据          5-15 分钟       频繁更新但不必实时
股票价格          几秒            高频变动
知识库文档        1-24 小时       很少变动
认证 token        到期的 80%      安全窗口
```

### 4. 优雅降级（Graceful Degradation）

当 API 不可用时，Agent 不应崩溃，而应降级：

```python
class DegradationManager:
    """
    降级管理器 —— 在 API 不可用时提供替代方案。
    """

    def __init__(self):
        self._fallbacks: dict[str, Any] = {}
        self._last_known_good: dict[str, Any] = {}

    def register_fallback(self, api_name: str, fallback_value: Any):
        """注册 API 不可用时的默认值。"""
        self._fallbacks[api_name] = fallback_value

    def on_success(self, api_name: str, value: Any):
        """记录最后一次成功获取的值。"""
        self._last_known_good[api_name] = value

    def get(self, api_name: str) -> Any:
        """按优先级：最近成功值 > 注册的 fallback > None。"""
        return (
            self._last_known_good.get(api_name)
            or self._fallbacks.get(api_name)
        )
```

## 与 4.5 Custom Tools 的关系

外部 API 记忆与 [4.5 custom-tools](../../4%20tool-use/4.5%20custom-tools/learn.md) 模块有紧密的重叠：

```text
Custom Tools 视角：
  Agent 将 API 调用封装为"工具函数"，LLM 通过函数调用来"使用工具"
  → 焦点在：工具声明、参数注入、返回值解析

External API Memory 视角：
  Agent 将 API 调用视为"记忆操作"，通过 API 来"回忆"和"记住"
  → 焦点在：数据格式、缓存策略、容错机制、一致性

交集：
  两者都涉及 Agent 调用外部 API
  两者都需要处理认证、限速、错误
  两者都面临网络延迟和可用性问题

差异：
  Custom Tools 关注"能做什么"（工具的能力范围）
  API Memory 关注"知道什么"（获取的信息本身）
```

**示例：工具调用 vs 记忆调用的视角差异**：

```text
同样的天气 API 调用：

作为 Tool（4.5 视角）：
  def get_weather(location: str) -> str:
      \"\"\"获取天气信息。\"\"\"
      response = call_weather_api(location)
      return response

作为 Memory（6.6.5 视角）：
  weather_memory = APIMemoryAdapter("weather", ...)
  # "回忆"天气信息
  memory = await weather_memory.recall("/current", {"q": location})
  # 自动应用缓存、限速、断路器
```

外部 API 记忆可以看作 **Custom Tools 在记忆域的特化应用**——当工具调用返回的信息被当作"记忆"使用时，工具就成为了记忆通道。

## 能力边界

### 1. 供应商锁定（Vendor Lock-in）

一旦 Agent 深度依赖某个外部 API 作为记忆源：
- API 接口变更 → Agent 需要修改调用代码
- API 停止服务 → Agent 失去该记忆源
- 数据格式变化 → 解析逻辑需要更新
- 定价策略变化 → 运行成本不可控

**缓解策略**：引入适配器层隔离 API 变更；多 API 源冗余。

### 2. 网络依赖（Network Dependency）

外部 API 记忆完全依赖网络：
- 网络断开 → 记忆不可访问
- 网络延迟 → 响应延迟
- 网络不稳定 → 间歇性失败

**缓解策略**：本地缓存 + 降级策略 + 离线模式。

### 3. 每次调用的成本（Per-Call Cost）

与传统本地存储不同，外部 API 调用通常按量计费：
- 高频调用 → 成本快速累积
- 数据量大 → 传输费用增加
- 复杂的业务逻辑 → 多次 API 调用

**缓解策略**：缓存命中率优化、请求批处理、选择按需而非轮询。

### 4. 实时性语义不一致

外部 API 的"实时性"不可靠：
- 数据库复制延迟
- CDN 缓存未过期
- 负载均衡导致的陈旧数据

```text
时刻 T:  外部系统更新数据
         ↓
         API 服务器 A（已有最新数据）
         API 服务器 B（尚未复制，返回旧数据）
         ↓
Agent:   两次调用得到不同结果 —— 因为路由到了不同节点
```

**缓解策略**：请求带版本号或时间戳检查；接受最终一致性。

### 5. 安全与隐私

- API 密钥管理（泄露风险）
- 数据通过公网传输（中间人攻击）
- 外部系统存储 Agent 的数据（数据主权问题）

## 最大挑战：平衡新鲜度 vs 延迟 vs 成本

外部 API 记忆的根本三角矛盾：

```text
            新鲜度（Freshness）
               /\
              /  \
             /    \
            / 理想  \
           /  平衡点 \
          /__________\
      延迟（Latency）  成本（Cost）

三个目标不可兼得：
  - 最新数据（实时调用）→ 高延迟 + 高成本
  - 低延迟（用缓存）    → 数据可能过时 + 缓存成本
  - 低成本（少调用）    → 数据过时 + 无延迟保障
```

### 动态平衡策略

```python
class AdaptiveMemoryStrategy:
    """
    自适应记忆策略 —— 根据场景自动调整新鲜度与延迟的平衡。
    """

    def __init__(self):
        self._latency_budget = 200.0    # 延迟预算（毫秒）
        self._staleness_tolerance = 60.0  # 可接受的数据过时时间（秒）

    def should_use_cache(self, data_age_seconds: float) -> bool:
        """判断是否可以使用缓存。"""
        return data_age_seconds < self._staleness_tolerance

    def adjust_for_urgency(self, urgency: float):
        """
        根据任务紧急程度调整策略。
        urgency: 0.0（容忍过时）~ 1.0（要求最新）
        """
        if urgency > 0.8:
            self._staleness_tolerance = 5.0    # 高紧迫：容忍 5 秒过时
            self._latency_budget = 500.0       # 可接受更高延迟
        elif urgency < 0.2:
            self._staleness_tolerance = 600.0  # 低紧迫：容忍 10 分钟
            self._latency_budget = 50.0        # 延迟要低
```

### 实际权衡指南

```text
场景                    策略                   权衡
──────────────────────────────────────────────────────────
用户查询天气            缓存 5-15 分钟          轻微过时换低延迟
股票交易决策            实时调用未缓存          最新数据 > 延迟和成本
聊天助手查百科          缓存 1 小时             过时代价低
医疗诊断辅助            实时 + 多源交叉验证      新鲜度至关重要
智能家居控制            实时调用（安全关键）      安全 > 成本
内容推荐                缓存 + 批量预取          批量换低成本
```

## 场景判断：何时使用 API 记忆

### 强烈推荐使用 API 记忆

1. **数据已存在于外部系统**
   - 企业 SaaS 集成（Salesforce、Jira、Slack）
   - 公共数据 API（天气、地理、新闻、百科）
   - 已经有微服务架构的存量数据

2. **实时性要求高于延迟容忍**
   - 实时监控 Agent
   - 交易决策 Agent
   - 事件响应 Agent

3. **本地存储不可行**
   - 无状态 Serverless 函数
   - 边缘设备（存储空间有限）
   - 需要合规的数据隔离

### 强烈推荐使用本地存储

1. **高频访问的核心数据**
   - Agent 的日常工作记忆
   - 频繁使用的知识库
   - 用户偏好配置

2. **延迟敏感的操作**
   - 实时对话响应
   - 毫秒级决策链
   - 嵌入式 Agent

3. **敏感数据的合规要求**
   - 数据不能离开本地网络
   - 需要 GDPR/HIPAA/SOC2 审计

### 混合策略推荐

大多数生产系统采用 **API + 本地缓存** 的混合模式：

```text
             ┌──────────────┐
             │  Agent       │
             │  推理引擎     │
             └──────┬───────┘
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
  ┌──────────────┐   ┌──────────────┐
  │  本地缓存     │   │  外部 API     │
  │  (高频/核心)  │   │  (低频/实时)  │
  │  SQLite/Redis │   │  REST/gRPC   │
  └──────────────┘   └──────────────┘
          │                   │
          └─────────┬─────────┘
                    │
                    ▼
          ┌──────────────┐
          │  统一的记忆    │
          │  抽象层       │
          └──────────────┘
```

## 总结

外部 API 记忆是一种通过远程 API 调用实现"不存储"的记忆模式。它的核心价值在于**零存储基础设施**和**始终实时**，但代价是**网络延迟**、**可用性依赖**和**按量计费**。

在设计 Agent 记忆架构时，API 记忆不是本地存储的替代品，而是互补品：

- 用 API 记忆来**接入外部世界的数据**
- 用本地缓存来**稀释 API 记忆的延迟和成本**
- 用断路器和降级策略来**保护 Agent 免受外部系统故障的影响**
- 用适配器层来**隔离 API 接口变更的影响**

最终，最稳健的 Agent 记忆系统不是"纯 API"也不是"纯本地"——而是根据每个数据维度的**新鲜度要求**、**访问频次**和**可用性预算**，动态地在两者之间做出权衡。

---

**上一篇**: [6.6.4 文档存储](../6.6.4%20document-store/learn.md)
**下一篇**: [6.6.6 记忆联邦](../6.6.6%20memory-federation/learn.md)
**相关模块**: [4.5 Custom Tools](../../4%20tool-use/4.5%20custom-tools/learn.md)
