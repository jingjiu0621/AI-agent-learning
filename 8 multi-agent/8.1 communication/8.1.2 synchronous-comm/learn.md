# 8.1.2 synchronous-comm — 同步通信：请求-响应

## 简单介绍

同步通信是最直观的 Agent 间交互模式：一个 Agent 发送请求后阻塞等待响应。它简单、直接，适合需要即时结果的场景，但在高延迟或长任务中会阻塞发送方。

## 基本原理

```
Agent A                    Agent B
   │                          │
   ├── Request ──────────────►│
   │                          │
   │     (A 阻塞等待)          │
   │                          │
   │◄──── Response ──────────┤
   │                          │
   ▼                          ▼
     时间 → → → → → → →
```

### 核心模型

- **一对一**：一个发送者对应一个接收者
- **请求-响应配对**：通过 correlation_id 匹配请求和响应
- **阻塞语义**：发送方在收到响应前不会继续执行
- **超时机制**：等待超过时限则视为失败

### 实现方式

| 传输层 | 特点 | 适用场景 |
|--------|------|----------|
| HTTP/REST | 无状态、防火墙友好 | 跨网络 Agent 通信 |
| gRPC | 双向流、强类型、高性能 | 内部 Agent 集群 |
| WebSocket | 持久连接、低延迟 | 实时交互 |
| 进程内 IPC | 零序列化开销 | 同一进程的 Agent |

## 背景与演进

- **早期（2022）**：多 Agent 系统使用简单的 HTTP 回调——Agent A 调用 Agent B 的 API 并等待 JSON 响应。结果就是"回调地狱"，代码难以维护，错误处理分散。
- **gRPC 引入（2023）**：LangGraph 和 AutoGen 开始在内部使用 gRPC 进行 Agent 间通信，利用其 streaming 和 deadline 机制。性能提升明显，但引入 gRPC 的复杂度。
- **WebSocket 普及（2024）**：对于需要持续通信的场景（如多 Agent 辩论），WebSocket 成为主流，避免了 HTTP 的每次握手开销。
- **当前趋势**：混合使用——控制面用 HTTP（管理操作），数据面用 gRPC/WebSocket（高频通信）。

## 核心矛盾

**同步通信的核心问题是"阻塞等待 vs 资源利用"**：

| 优势 | 劣势 |
|------|------|
| 编程模型简单直观 | 发送方被阻塞，利用率低 |
| 天然请求-响应配对 | 长任务导致长时间等待 |
| 错误处理直接 | 级联失败风险（A 等 B，B 等 C） |
| 调试容易 | 网络分区时悬挂 |
| 事务语义明确 | 不适合流式/事件驱动场景 |

## 主流优化方向

1. **超时与截止时间**：为每个请求设置 deadline，防止无限等待
2. **请求折叠**：合并短时间内对同一 Agent 的多个请求
3. **优先级队列**：高优先级请求插队，低优先级请求等待
4. **连接池**：复用 TCP 连接减少握手开销
5. **负载感知路由**：根据接收方当前负载选择目标
6. **部分响应**：允许接收方先返回中间结果，再返回完整结果

## 能力边界与结果边界

- **最大等待时间**：典型设置 5-30 秒，超过则触发降级
- **并发连接数**：单 Agent 通常限制 10-100 个并发请求
- **最大请求体**：取决于传输层，通常 1-10MB
- **不适用场景**：长时间运行的任务（>1 分钟）、流式生成、事件通知

## 与其他通信模式的区别

- **vs 异步通信（8.1.3）**：同步是阻塞等待，异步是发完即走。同步强一致性，异步高吞吐。
- **vs 广播（8.1.4）**：同步是一对一，广播是一对多。同步需要响应，广播不需要。
- **vs Web API**：传统 API 的请求-响应模式被直接复用，但 Agent 场景增加了上下文携带、优先级、TTL 等概念。

## 核心优势

- **编程简单**：请求-响应模式是程序员最熟悉的模型
- **强一致性**：发送方知道请求一定被处理（或明确失败）
- **调试容易**：单一请求-响应的生命周期清晰
- **事务支持**：天然支持 ACID-like 操作语义

## 工程优化

1. **Deadline 传播**：A→B→C 调用链中，deadline 逐层传递
2. **超时熔断**：连续超时达到阈值后快速失败
3. **请求去重**：基于消息 ID 实现幂等处理
4. **背压机制**：接收方负载高时通知发送方减速
5. **优雅降级**：超时后尝试缓存或近似结果

## 适用场景

| 场景 | 推荐实现 | 原因 |
|------|----------|------|
| 查询/检索类交互 | HTTP/gRPC | 响应快、确定性强 |
| Agent 间配置同步 | gRPC | 强类型、deadline |
| 协调器→Worker 指令 | HTTP | 简单、防火墙友好 |
| 跨进程 Agent 通信 | gRPC | 高性能、双向流 |

## 示例代码

### HTTP 同步通信

```python
import requests
import json
import uuid
import time

class SyncAgentClient:
    """同步 HTTP Agent 客户端"""

    def __init__(self, base_url: str, timeout: float = 30.0):
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout
        self.session = requests.Session()

    def send_request(self, to_agent: str, payload: dict,
                     priority: int = 0) -> dict:
        """发送同步请求"""
        message = {
            "id": f"msg_{uuid.uuid4().hex[:12]}",
            "from": self.base_url,
            "to": to_agent,
            "type": "request",
            "payload": payload,
            "priority": priority,
            "timestamp": time.time()
        }
        start = time.time()
        try:
            resp = self.session.post(
                f"{self.base_url}/agent/message",
                json=message,
                timeout=self.timeout
            )
            resp.raise_for_status()
            elapsed = time.time() - start
            result = resp.json()
            print(f"请求 {message['id']} -> {to_agent}: {elapsed:.2f}s")
            return result
        except requests.Timeout:
            raise TimeoutError(
                f"请求 {message['id']} 超时 ({self.timeout}s)"
            )
        except requests.RequestException as e:
            raise ConnectionError(
                f"请求 {message['id']} 失败: {e}"
            )

# 使用示例
orchestrator = SyncAgentClient("http://agent-orch:8080")
researcher = SyncAgentClient("http://agent-research:8081")

try:
    # 同步等待研究 Agent 的结果
    result = researcher.send_request(
        to_agent="researcher",
        payload={"task": "search", "query": "latest AI papers"}
    )
    print(f"研究结果: {result}")
except TimeoutError:
    print("研究 Agent 超时，使用缓存结果")
```

### gRPC 服务定义

```protobuf
service AgentService {
  rpc SendMessage (AgentMessage) returns (AgentResponse);
  rpc StreamMessages (stream AgentMessage) returns (stream AgentResponse);
}

message AgentResponse {
  string id = 1;
  string correlation_id = 2;
  StatusCode status = 3;
  bytes result = 4;
  string error_message = 5;

  enum StatusCode {
    OK = 0;
    ERROR = 1;
    TIMEOUT = 2;
    RETRY_LATER = 3;
  }
}
```

### 带 deadline 传播的同步调用链

```python
import asyncio
import time

class SyncChain:
    """带 deadline 传播的同步调用链"""

    async def call_with_deadline(self, agent_url: str,
                                 payload: dict, deadline: float):
        """在 deadline 前完成调用"""
        remaining = deadline - time.time()
        if remaining <= 0:
            raise TimeoutError("deadline 已过期")

        async with aiohttp.ClientSession() as session:
            async with session.post(
                agent_url,
                json={"payload": payload, "deadline": deadline},
                timeout=aiohttp.ClientTimeout(total=remaining)
            ) as resp:
                return await resp.json()

    async def orchestrate(self, task: dict):
        """协调多个 Agent，带 deadline 传播"""
        global_deadline = time.time() + 60  # 全局 60s deadline

        # Step 1: 研究（分配 30s）
        research_result = await self.call_with_deadline(
            "http://researcher:8081",
            task["research_query"],
            deadline=time.time() + 30
        )

        # Step 2: 写作（使用剩余时间）
        write_result = await self.call_with_deadline(
            "http://writer:8082",
            {"research": research_result, **task},
            deadline=global_deadline
        )
        return write_result
```
