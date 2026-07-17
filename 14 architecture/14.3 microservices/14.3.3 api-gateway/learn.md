# API 网关设计 (API Gateway)

## 一、基本原理

API 网关是 Agent 系统的**统一入口**，位于客户端与后端微服务之间，负责请求路由、协议转换、安全认证、流量控制、响应聚合等跨切面关注点。

```
                    Agent 系统边界
    +-----------------------------------------------------------+
    |                                                           |
    |   +----------+                                            |
    |   |  Client   |                                            |
    |   | (Web/App) |                                            |
    |   +-----+----+                                            |
    |         |                                                 |
    |         |  HTTP/WebSocket                                 |
    |         v                                                 |
    |   +--------------------------------------------------+    |
    |   |                  API Gateway                      |    |
    |   |  +----------------+  +------------------------+   |    |
    |   |  |  路由层 Router |  | 中间件链 Middleware     |   |    |
    |   |  +----------------+  | - Auth                 |   |    |
    |   |  |  协议转换       |  | - Rate Limit          |   |    |
    |   |  | HTTP<->gRPC    |  | - Logging             |   |    |
    |   |  | WebSocket<->gRPC|  | - Circuit Breaker    |   |    |
    |   |  +----------------+  | - Request Transform   |   |    |
    |   |                      +------------------------+   |    |
    |   +--------------------------------------------------+    |
    |              |          |           |          |           |
    |              v          v           v          v           |
    |         +-------+ +-------+ +-------+ +--------+           |
    |         |Reason | |Memory | |Tool   | |Planner |           |
    |         |Svc    | |Svc    | |Exec   | |Svc     |           |
    |         +-------+ +-------+ +-------+ +--------+           |
    |                                                           |
    +-----------------------------------------------------------+
```

### 1.1 核心职责

| 职责 | 说明 | Agent 系统中的特殊要求 |
|------|------|----------------------|
| 请求路由 | 将客户端请求转发到正确的后端服务 | 根据 Agent ID/会话 ID 路由到特定推理实例 |
| 认证授权 | 验证客户端身份，传播认证上下文 | 需要将用户身份透传给下游推理服务 |
| 速率限制 | 控制客户端请求频率 | 需要按用户、按 Agent 类型、按 Token 不同粒度限流 |
| 协议转换 | 转换客户端与后端之间的协议 | HTTP <-> gRPC, WebSocket <-> gRPC 双向流 |
| 流式聚合 | 合并多个后端的流式响应 | 多 Agent 协作时的响应聚合 |
| 熔断降级 | 保护后端服务不被过载 | 推理服务慢启动、冷启动保护 |
| 会话管理 | 维护会话亲和性和状态 | Agent 推理会话的有状态路由 |

### 1.2 Agent 网关的流量模型

```
传统 API 网关流量:
  Client -> Gateway -> Service A (50ms)
  Client -> Gateway -> Service B (50ms)
  单次请求耗时 ~50ms，请求之间无关联

Agent 网关流量:
  Client -> Gateway -> Reasoning Svc (15s)  ← 长连接
           Gateway -> Tool Exec (500ms)       ← 子请求
           Gateway -> Memory Svc (10ms)       ← 异步
           Gateway -> Client (streaming 15s)  ← 持续推送
  单次 Agent 调用耗时 ~15s，持续双向流量

差异:
  - 连接时长: 50ms -> 15s (300x)
  - 并发连接数: 常规网关可处理 10K QPS
  - Agent 网关: 需要处理 10K * 15s = 150K 个并发长连接
  - 内存消耗: 每个连接需要保持 WebSocket/gRPC 流状态
```

---

## 二、背景与演进

### 2.1 没有网关的问题

```
无网关架构 (直接连接):
                              +----------------+
   Client A ----------HTTP---> | Reasoning Svc  |
   Client B ----------HTTP---> | (认证逻辑重复) |
   Client C ----------WebSocket| (限流逻辑重复) |
   Client D ----------gRPC---->| (协议不统一)   |
                              +----------------+
   Client E ----------HTTP---> | Tool Executor  |
   Client F ----------HTTP---> | (认证逻辑重复) |
                              +----------------+
   Client G ----------HTTP---> | Memory Service |
                              +----------------+

问题:
  1. 每个服务重复实现认证、限流、日志
  2. 客户端需要知道所有服务的地址和协议
  3. 重构后端时客户端必须跟着改
  4. 无法统一管理 API 版本和流量策略
  5. 安全策略散落在各服务中，难以审计
  6. 无法统一处理 CORS、请求验证等跨切面逻辑
```

### 2.2 网关引入后的架构演进

```
第一阶段: 简单代理
  +--------+     +----------+     +-----------+
  | Client | --> |  Reverse  | --> |  Service  |
  +--------+     |  Proxy   |     |    A      |
                 | (Nginx)  |     +-----------+
                 +----------+

第二阶段: API 网关
  +--------+     +----------+     +-----------+
  | Client | --> |   API    | --> | Service A |
  +--------+     | Gateway  | --> | Service B |
                 | (路由+   | --> | Service C |
                 |  限流+   |     +-----------+
                 |  认证)   |
                 +----------+

第三阶段: Agent 网关
  +--------+     +-------------+     +-------------+
  | Client | --> |  Agent      | --> | Reasoning   |
  +--------+     |  Gateway    | --> | Memory      |
                 | (流式聚合+  | --> | Tool Exec   |
                 |  会话路由+  | --> | Planner     |
                 |  协议转换)  |     +-------------+
                 |  WebSocket<->gRPC 双向流 |
                 |  状态感知路由         |
                 |  推理会话管理         |
                 +-------------+
```

---

## 三、Agent 网关 vs 传统网关

### 3.1 核心差异对比

| 维度 | 传统 API 网关 | Agent 网关 |
|------|--------------|-----------|
| 连接模型 | 短连接 (HTTP 请求-响应) | 长连接 (WebSocket + gRPC 流) |
| 路由依据 | URL 路径 + HTTP 方法 | URL + Session ID + Agent ID |
| 状态管理 | 无状态，水平扩展简单 | 有状态，需要会话亲和性 |
| 流处理 | 透传完整 HTTP 响应 | 聚合和转换流式 WebSocket/gRPC 消息 |
| 协议转换 | REST <-> REST | HTTP/WS <-> gRPC 双向流 |
| 限流粒度 | QPS/IP 级别 | Token 级别、推理时间级别 |
| 故障模式 | 快速失败 | 优雅降级（部分结果返回） |
| 超时处理 | 固定超时 (如 30s) | 动态超时 (推理时间不确定) |
| 缓存策略 | 响应缓存 | 推理结果缓存 + 中间状态缓存 |

### 3.2 传统网关做不到的事情

```python
# 传统网关 (如 Kong/Nginx) 的局限:

# 1. 无法理解 Agent 协议
request:  POST /api/agent/reason
body:     {"prompt": "分析这个数据"}

# 传统网关: 转发到后端, 等待完整响应, 返回
# Agent 需要: 逐步流式返回推理 token, 网关需要理解并转发

# 2. 无法处理会话亲和性
# Agent 推理有状态 (推理上下文在内存中)
# 传统网关: 轮询/随机路由到任意实例
# Agent 需要: 同一个会话的所有请求必须路由到同一个推理实例

# 3. 无法聚合多路流
# 多 Agent 协作时:
# Agent A 的推理流 + Agent B 的推理流 -> 网关合并排序 -> 客户端
# 传统网关: 无法同时管理多个流并排序
```

---

## 四、网关功能详解

### 4.1 功能全景图

```
                       Agent Gateway
                            |
       +--------------------+--------------------+
       |                    |                    |
  请求路由            安全防护             流量控制
       |                    |                    |
  基于 URL 路由       JWT 验证            用户级限流
  基于 Session 路由    API Key 管理        Token 级限流
  基于 Agent 路由      RBAC 授权          并发控制
  灰度路由           认证上下文传递        请求队列
       |                    |                    |
  协议转换            流式处理            可观测性
       |                    |                    |
  HTTP <-> gRPC       WebSocket 代理      访问日志
  WebSocket <-> gRPC   SSE 聚合           指标采集
  Protobuf <-> JSON   流式消息转换        分布式追踪
       |                    |                    |
  弹性能力            会话管理            安全加固
       |                    |                    |
  熔断 (Circuit Breaker)   Session 亲和性   IP 白名单
  重试 (Retry)             Session 重建     WAF 防护
  超时控制                 Session 迁移     SQL 注入防护
  降级 (Fallback)          Context 传递     请求签名
```

### 4.2 请求路由 (Request Routing)

```
路由策略:

1. 静态路由 (基于 URL):
   POST /api/v1/agent/reason  -> Reasoning Service
   POST /api/v1/agent/memory  -> Memory Service
   GET  /api/v1/agent/tools   -> Tool Registry

2. 动态路由 (基于 Session):
   Header: X-Session-Id: sess-abc123
   -> 查找 Session-abc123 绑定的推理实例
   -> 路由到该实例 (会话亲和性)

3. 内容路由 (基于 Agent 类型):
   POST /api/v1/agent/reason
   Body: {"agent_type": "code_agent"}
   -> 路由到 code-agent 推理池
   
4. 灰度路由 (基于用户/百分比):
   Header: X-User-Id: user_100
   -> user_100 属于灰度组 -> 路由到 v2 推理服务
   -> 其他用户 -> 路由到 v1 推理服务
```

### 4.3 认证与授权传播 (Authentication Propagation)

```
认证信息流:

+--------+   JWT Token   +---------+   Token + ServiceIdentity   +----------+
| Client | ------------> | Gateway | --------------------------> | Service  |
+--------+               +---------+                              +----------+
                              |
                    验证 JWT 签名
                    提取 UserID, Role
                    添加 Internal Headers:
                      X-User-Id: user_123
                      X-User-Role: admin
                      X-Request-Id: req-abc
                      X-Gateway-Signature: <hmac>
```

### 4.4 速率限制 (Rate Limiting)

```
Agent 系统中多维度限流:

+------------------+
|    Rate Limiter   |
+------------------+
        |
   +----+----+
   |         |
   v         v
 固定窗口   滑动窗口  令牌桶  并发限制
  (简单)   (精确)   (突发)  (连接数)
  
维度:
  1. 用户级别:  user_123 -> 10 req/min  (防止单个用户滥用)
  2. API 级别:  /agent/reason -> 100 req/min (保护推理服务)
  3. Token 级别: user_123 -> 100K tokens/min (LLM Token 配额)
  4. 并发级别:  单用户最大同时推理: 3
  5. 全局级别:  推理服务总并发: 100

限流响应:
  HTTP 429 Too Many Requests
  {
    "error": "rate_limit_exceeded",
    "message": "请求过于频繁, 请稍后重试",
    "retry_after_ms": 5000,
    "limit": 10,
    "remaining": 0,
    "reset_at": "2025-01-01T00:01:00Z"
  }
```

### 4.5 流式聚合 (Streaming Aggregation)

```
多源流式数据聚合:

用户问题: "分析这份财报并给出投资建议"

+---------+     WebSocket      +---------+     gRPC 双向流     +-----------+
| Client  | <----------------> | Gateway | <-----------------> | Financial |
|         |   stream: tokens   |         |   stream: thoughts  | Agent     |
+---------+                    +---------+                    +-----------+
                                    |        gRPC 流     +-----------+
                                    +------------------> | Market    |
                                    |   stream: data     | Data Svc  |
                                    |                    +-----------+
                                    |        gRPC 流     +-----------+
                                    +------------------> | Report    |
                                    |   stream: chunks   | Generator |
                                                          +-----------+

网关聚合逻辑:
  1. Financial Agent: "让我先分析财报数据..." (推理 token)
  2. Market Data Svc: {pe_ratio: 25.3, revenue_growth: "15%"} (数据块)
  3. Report Generator: "### 财务分析\n营收增长15%..." (报告片段)
  
  网关合并后发送给客户端:
  {
    "type": "thought",
    "source": "financial_agent",
    "content": "让我先分析财报数据..."
  }
  {
    "type": "data_chunk",
    "source": "market_data",
    "content": {"pe_ratio": 25.3, "revenue_growth": "15%"}
  }
  {
    "type": "thought",
    "source": "financial_agent",
    "content": "基于市场数据, 该公司的估值..."
  }
```

### 4.6 熔断 (Circuit Breaker)

```
Agent 网关熔断状态机:

          +--------+
          | CLOSED |  <-- 正常运行, 请求通过
          +---+----+
              |
         失败计数 >= 阈值
              |
              v
          +--------+    超时时间到    +---------+
          | OPEN   | --------------> | HALF-OPEN|  <-- 尝试恢复
          +--------+                 +----+----+
               |                          |
          请求快速失败                 试探请求成功
          (返回降级响应)                    |
               |                          v
               |                     +--------+
               +-------------------->| CLOSED |  <-- 恢复
                                     +--------+

Agent 场景的熔断配置:
  阈值: 5s 内 50% 请求失败
  超时: 30s (比传统服务的 10s 更长, 因为推理时间长)
  半开试探数: 3
  降级响应: 返回缓存结果或友好错误

失败类型:
  - 5xx 错误
  - 超时 (推理超时)
  - 连接拒绝 (服务崩溃/重启)
  - 流中断 (gRPC 流异常断开)
```

---

## 五、代码示例

### 5.1 FastAPI Agent Gateway 完整实现

```python
"""
Agent API Gateway - 基于 FastAPI 的 Agent 网关

核心功能:
  1. 认证中间件 (JWT 验证)
  2. 速率限制 (基于 Redis)
  3. 会话路由 (Session 亲和性)
  4. WebSocket 流式代理 (WebSocket <-> gRPC 双向流)
  5. 熔断保护
  6. 请求/响应转换
"""

import asyncio
import hashlib
import hmac
import json
import logging
import time
import uuid
from contextlib import asynccontextmanager
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
from typing import AsyncGenerator, Optional

import grpc
import httpx
import jwt
import redis.asyncio as redis
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Request, Response
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse, StreamingResponse

# ============================
# 数据模型
# ============================

class AgentService(Enum):
    REASONING = "reasoning"
    MEMORY = "memory"
    TOOL_EXECUTOR = "tool"
    PLANNER = "planner"


@dataclass
class SessionRoute:
    """会话路由信息"""
    session_id: str
    user_id: str
    agent_type: str
    reasoning_instance: str  # gRPC target, e.g. "10.0.1.5:50051"
    created_at: float
    last_active: float
    status: str = "active"  # active | draining | closed


@dataclass
class RateLimitConfig:
    """限流配置"""
    requests_per_minute: int = 30
    tokens_per_minute: int = 100000
    concurrent_sessions: int = 3
    burst_size: int = 5


@dataclass
class CircuitBreakerState:
    """熔断器状态"""
    service: str
    failures: int = 0
    successes: int = 0
    last_failure_time: float = 0
    state: str = "closed"  # closed | open | half_open
    open_until: float = 0
    failure_threshold: int = 10
    recovery_timeout: float = 30.0
    half_open_max_requests: int = 3
    half_open_requests: int = 0


# ============================
# 中间件: 认证
# ============================

class AuthMiddleware:
    """JWT 认证中间件"""

    def __init__(self, jwt_secret: str, jwt_algorithm: str = "HS256"):
        self.secret = jwt_secret
        self.algorithm = jwt_algorithm

    async def verify_token(self, token: str) -> dict:
        """
        验证 JWT Token 并返回 payload
        
        Returns:
            dict: {"user_id": str, "role": str, "permissions": list}
        """
        try:
            payload = jwt.decode(
                token,
                self.secret,
                algorithms=[self.algorithm],
                options={"require": ["sub", "exp", "role"]},
            )
            return {
                "user_id": payload["sub"],
                "role": payload["role"],
                "permissions": payload.get("permissions", []),
                "agent_quota": payload.get("agent_quota", {}),
            }
        except jwt.ExpiredSignatureError:
            raise AuthError("Token 已过期", 401)
        except jwt.InvalidTokenError as e:
            raise AuthError(f"无效的 Token: {str(e)}", 401)

    def create_internal_token(self, user_info: dict, target_service: str) -> str:
        """
        创建内部服务通信 Token (service-to-service)
        
        下游服务通过此 Token 验证调用来源
        """
        payload = {
            "sub": user_info["user_id"],
            "role": user_info["role"],
            "permissions": user_info["permissions"],
            "iss": "api-gateway",
            "aud": target_service,
            "exp": datetime.utcnow() + timedelta(seconds=30),  # 短 TTL
            "iat": datetime.utcnow(),
            "jti": str(uuid.uuid4()),
        }
        return jwt.encode(payload, self.secret, algorithm=self.algorithm)


class AuthError(Exception):
    def __init__(self, message: str, status_code: int = 401):
        self.message = message
        self.status_code = status_code


# ============================
# 中间件: 速率限制
# ============================

class RateLimiter:
    """基于 Redis 的分布式速率限制器"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def check_rate_limit(
        self,
        user_id: str,
        limit_type: str,
        limit: int,
        window_seconds: int = 60,
    ) -> tuple[bool, dict]:
        """
        检查速率限制 (滑动窗口算法)
        
        Returns:
            (allowed: bool, headers: dict)
        """
        key = f"ratelimit:{limit_type}:{user_id}"
        now = time.time()
        window_start = now - window_seconds

        # Redis 流水线: 原子操作
        pipe = self.redis.pipeline()
        pipe.zremrangebyscore(key, 0, window_start)  # 移除窗口外的记录
        pipe.zcard(key)                                 # 当前窗口内的请求数
        pipe.zadd(key, {str(now): now})                 # 添加当前请求
        pipe.expire(key, window_seconds * 2)            # 设置 TTL
        results = await pipe.execute()

        current_count = results[1]  # zcard 结果
        allowed = current_count <= limit

        headers = {
            "X-RateLimit-Limit": str(limit),
            "X-RateLimit-Remaining": str(max(0, limit - current_count)),
            "X-RateLimit-Reset": str(int(window_start + window_seconds)),
        }

        return allowed, headers

    async def check_concurrent_limit(
        self, user_id: str, max_concurrent: int
    ) -> bool:
        """检查并发会话数限制"""
        key = f"concurrent:{user_id}"
        current = await self.redis.get(key) or 0
        return int(current) < max_concurrent

    async def increment_concurrent(self, user_id: str):
        """增加并发计数"""
        key = f"concurrent:{user_id}"
        await self.redis.incr(key)
        await self.redis.expire(key, 3600)  # 1h 自动过期

    async def decrement_concurrent(self, user_id: str):
        """减少并发计数"""
        key = f"concurrent:{user_id}"
        await self.redis.decr(key)


# ============================
# 中间件: 熔断器
# ============================

class CircuitBreaker:
    """服务级熔断保护"""

    def __init__(self):
        self._states: dict[str, CircuitBreakerState] = {}

    def get_state(self, service: str) -> CircuitBreakerState:
        if service not in self._states:
            self._states[service] = CircuitBreakerState(service=service)
        return self._states[service]

    async def call(self, service: str, func, *args, **kwargs):
        """
        带熔断保护的服务调用
        
        使用方式:
            result = await circuit_breaker.call("reasoning", make_grpc_call, ...)
        """
        state = self.get_state(service)
        
        if state.state == "open":
            if time.time() >= state.open_until:
                # 进入半开状态
                state.state = "half_open"
                state.half_open_requests = 0
            else:
                raise CircuitBreakerOpenError(
                    f"服务 {service} 熔断中, 请稍后重试"
                )

        if state.state == "half_open":
            if state.half_open_requests >= state.half_open_max_requests:
                raise CircuitBreakerOpenError(
                    f"服务 {service} 半开状态, 试探请求已满"
                )
            state.half_open_requests += 1

        try:
            result = await func(*args, **kwargs)
            state.successes += 1
            
            # 半开状态下成功 -> 关闭熔断器
            if state.state == "half_open" and state.half_open_requests > 0:
                state.state = "closed"
                state.failures = 0
                print(f"[CircuitBreaker] {service} 恢复, 已关闭熔断器")
            
            return result

        except Exception as e:
            state.failures += 1
            state.last_failure_time = time.time()
            
            if state.state == "half_open":
                state.state = "open"
                state.open_until = time.time() + state.recovery_timeout
                print(f"[CircuitBreaker] {service} 半开试探失败, 重新打开熔断器")
            elif state.failures >= state.failure_threshold:
                state.state = "open"
                state.open_until = time.time() + state.recovery_timeout
                print(f"[CircuitBreaker] {service} 失败次数超限, 打开熔断器, "
                      f"恢复时间 {state.recovery_timeout}s")
            
            raise


class CircuitBreakerOpenError(Exception):
    pass


# ============================
# 会话路由管理器
# ============================

class SessionRouter:
    """
    会话路由管理器
    
    维护 Agent 推理会话与后端实例的映射关系，确保同一会话的
    所有请求路由到同一个推理实例 (会话亲和性)。
    """

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self._local_cache: dict[str, SessionRoute] = {}

    async def get_or_create_route(
        self, session_id: str, user_id: str, agent_type: str
    ) -> SessionRoute:
        """
        获取已有路由或创建新路由
        
        Redis 数据结构:
            Key: session:{session_id}:route
            Field-Value 哈希:
                reasoning_instance -> "10.0.1.5:50051"
                user_id -> "user_123"
                agent_type -> "code_agent"
                created_at -> 1234567890.0
        """
        route_key = f"session:{session_id}:route"
        
        # 尝试从 Redis 获取已有路由
        cached = await self.redis.hgetall(route_key)
        if cached:
            route = SessionRoute(
                session_id=session_id,
                user_id=cached.get(b"user_id", b"").decode(),
                agent_type=cached.get(b"agent_type", b"").decode(),
                reasoning_instance=cached.get(b"reasoning_instance", b"").decode(),
                created_at=float(cached.get(b"created_at", b"0")),
                last_active=time.time(),
            )
            # 更新本地缓存
            self._local_cache[session_id] = route
            return route

        # =============================================
        # 一致性哈希: 选择推理实例
        # =============================================
        reason_instances = await self._get_available_instances("reasoning")
        selected = self._consistent_hash(session_id, reason_instances)
        
        route = SessionRoute(
            session_id=session_id,
            user_id=user_id,
            agent_type=agent_type,
            reasoning_instance=selected,
            created_at=time.time(),
            last_active=time.time(),
        )
        
        # 写入 Redis
        await self.redis.hset(route_key, mapping={
            "user_id": user_id,
            "agent_type": agent_type,
            "reasoning_instance": selected,
            "created_at": str(route.created_at),
        })
        await self.redis.expire(route_key, 3600)  # 1h TTL
        
        self._local_cache[session_id] = route
        return route

    def _consistent_hash(self, key: str, nodes: list[str]) -> str:
        """
        一致性哈希: 确保同一 session 始终选择同一实例
        
        当节点数量变化时, 只有 1/N 的映射关系需要重新分配
        """
        if not nodes:
            raise NoAvailableInstanceError("没有可用的推理服务实例")
        
        hash_val = int(hashlib.md5(key.encode()).hexdigest(), 16)
        return nodes[hash_val % len(nodes)]

    async def _get_available_instances(self, service: str) -> list[str]:
        """从服务发现获取可用实例列表"""
        # 实际实现: 查询 Consul / etcd / K8s API
        # 此处简化处理
        instances = {
            "reasoning": [
                "10.0.1.5:50051",
                "10.0.1.6:50051",
                "10.0.1.7:50051",
            ],
            "memory": ["10.0.2.5:50052"],
            "tool": ["10.0.3.5:50053"],
        }
        return instances.get(service, [])

    async def invalidate_session(self, session_id: str):
        """会话结束时清除路由"""
        await self.redis.delete(f"session:{session_id}:route")
        self._local_cache.pop(session_id, None)


class NoAvailableInstanceError(Exception):
    pass


# ============================
# gRPC 流式代理
# ============================

class GrpcStreamProxy:
    """
    gRPC 双向流代理
    
    将客户端的 WebSocket 消息转换为 gRPC 流, 并将 gRPC
    服务端的流式响应转发回 WebSocket。
    """

    def __init__(self, channel_pool: dict[str, grpc.aio.Channel]):
        self.channels = channel_pool  # target -> grpc channel

    async def proxy_reasoning_stream(
        self, websocket: WebSocket, session_id: str, target: str
    ):
        """
        WebSocket <-> gRPC 双向流代理
        
        流程图:
        
        Client (WebSocket)         Gateway                Reasoning Svc (gRPC)
        +---------------+         +---------+              +----------------+
        | WS Text Frame | ------> | JSON -> | ------>      | gRPC StreamReq |
        | user_message  |         | Protobuf |              |                |
        +---------------+         +---------+              +----------------+
                                                             |
        +---------------+         +---------+               |
        | WS Text Frame | <------ | Protobuf| <-------------| gRPC StreamRes |
        | thought_chunk |         | -> JSON |               | thought        |
        | tool_call     |         |         |               | tool_call      |
        | session_status|         |         |               | session_update |
        +---------------+         +---------+               +----------------+
        """

        channel = self.channels.get(target)
        if not channel:
            channel = grpc.aio.insecure_channel(
                target,
                options=[
                    ("grpc.keepalive_time_ms", 10000),
                    ("grpc.keepalive_timeout_ms", 5000),
                    ("grpc.max_reconnect_backoff_ms", 1000),
                ],
            )
            self.channels[target] = channel

        # 创建 gRPC stub
        from agent_pb2_grpc import ReasoningServiceStub
        from agent_pb2 import (
            StreamReasonRequest, StreamReasonResponse,
            KeepAlive,
        )

        stub = ReasoningServiceStub(channel)

        # =============================================
        # 启动双向流
        # =============================================
        async def ws_to_grpc():
            """WebSocket 消息 -> gRPC 请求流"""
            try:
                while True:
                    # 从 WebSocket 接收客户端消息
                    raw = await websocket.receive_text()
                    msg = json.loads(raw)

                    # 转换为 gRPC StreamReasonRequest
                    grpc_req = StreamReasonRequest()

                    if msg["type"] == "user_message":
                        grpc_req.user_message = msg["content"]
                    elif msg["type"] == "tool_result":
                        grpc_req.tool_result.tool_call_id = msg["tool_call_id"]
                        grpc_req.tool_result.tool_name = msg["tool_name"]
                        grpc_req.tool_result.result = json.dumps(msg["result"])
                        grpc_req.tool_result.is_error = msg.get("is_error", False)
                    elif msg["type"] == "cancel":
                        grpc_req.cancel.SetInParent()
                    elif msg["type"] == "ping":
                        grpc_req.ping.timestamp = int(time.time() * 1000)

                    yield grpc_req

            except WebSocketDisconnect:
                # 客户端断开, 发送取消信号
                cancel = StreamReasonRequest()
                cancel.cancel.SetInParent()
                yield cancel

        async def grpc_to_ws(grpc_stream):
            """gRPC 响应流 -> WebSocket 消息"""
            try:
                async for response in grpc_stream:
                    which = response.WhichOneof("payload")

                    if which == "thought":
                        # 推理步骤 -> WebSocket JSON
                        step = response.thought
                        ws_msg = {
                            "type": "thought_chunk",
                            "session_id": session_id,
                            "step_id": step.step_id,
                            "sequence": step.sequence,
                            "step_type": StepType.Name(step.step_type),
                            "content": step.content,
                            "metadata": dict(step.metadata),
                        }
                        await websocket.send_json(ws_msg)

                    elif which == "tool_call":
                        # 工具调用请求 -> WebSocket
                        tc = response.tool_call
                        ws_msg = {
                            "type": "tool_call",
                            "session_id": session_id,
                            "tool_call_id": tc.tool_call_id,
                            "tool_name": tc.tool_name,
                            "arguments": json.loads(tc.arguments),
                            "timeout_ms": tc.timeout_ms,
                        }
                        await websocket.send_json(ws_msg)

                    elif which == "session":
                        # 会话状态更新
                        ws_msg = {
                            "type": "session_status",
                            "session_id": session_id,
                            "status": ReasoningStatus.Name(response.session.status),
                            "total_steps": response.session.total_steps,
                        }
                        await websocket.send_json(ws_msg)

                    elif which == "error":
                        # 错误信息
                        ws_msg = {
                            "type": "error",
                            "code": response.error.code,
                            "message": response.error.message,
                            "is_recoverable": response.error.is_recoverable,
                        }
                        await websocket.send_json(ws_msg)

                    elif which == "pong":
                        pass  # 心跳响应, 不转发到客户端

            except grpc.aio.AioRpcError as e:
                # gRPC 流异常
                await websocket.send_json({
                    "type": "error",
                    "code": "GRPC_STREAM_ERROR",
                    "message": f"推理流异常: {e.code()}",
                    "is_recoverable": e.code() != grpc.StatusCode.UNIMPLEMENTED,
                })

        # =============================================
        # 启动双向代理
        # =============================================
        grpc_stream = stub.StreamReason(ws_to_grpc())
        await grpc_to_ws(grpc_stream)


# ============================
# FastAPI 应用
# ============================

@dataclass
class GatewayConfig:
    """网关配置"""
    jwt_secret: str = "your-jwt-secret-here"
    redis_url: str = "redis://localhost:6379/0"
    cors_origins: list[str] = field(default_factory=lambda: ["*"])
    allowed_agent_types: list[str] = field(
        default_factory=lambda: ["code_agent", "data_agent", "chat_agent"]
    )
    rate_limits: dict[str, RateLimitConfig] = field(default_factory=dict)


class AgentGateway:
    """Agent API Gateway 主应用"""

    def __init__(self, config: GatewayConfig):
        self.config = config
        self.app = FastAPI(title="Agent API Gateway", version="1.0.0")
        self._setup_middleware()
        self._setup_routes()

        # 核心组件 (在 lifespan 中初始化)
        self.redis: Optional[redis.Redis] = None
        self.auth: Optional[AuthMiddleware] = None
        self.rate_limiter: Optional[RateLimiter] = None
        self.circuit_breaker: Optional[CircuitBreaker] = None
        self.session_router: Optional[SessionRouter] = None
        self.grpc_channels: dict[str, grpc.aio.Channel] = {}
        self.grpc_proxy: Optional[GrpcStreamProxy] = None

    def _setup_middleware(self):
        """配置 FastAPI 中间件"""
        self.app.add_middleware(
            CORSMiddleware,
            allow_origins=self.config.cors_origins,
            allow_credentials=True,
            allow_methods=["*"],
            allow_headers=["*"],
        )

        @self.app.middleware("http")
        async def auth_middleware(request: Request, call_next):
            """全局认证中间件"""
            # 跳过健康检查和公开端点
            if request.url.path in ["/health", "/metrics", "/openapi.json"]:
                return await call_next(request)

            auth_header = request.headers.get("Authorization", "")
            if not auth_header.startswith("Bearer "):
                return JSONResponse(
                    status_code=401,
                    content={"error": "missing_token", "message": "缺少认证令牌"},
                )

            token = auth_header[7:]
            try:
                user_info = await self.auth.verify_token(token)
                request.state.user = user_info
            except AuthError as e:
                return JSONResponse(
                    status_code=e.status_code,
                    content={"error": "auth_failed", "message": e.message},
                )

            return await call_next(request)

        @self.app.middleware("http")
        async def rate_limit_middleware(request: Request, call_next):
            """全局速率限制中间件"""
            if not hasattr(request.state, "user"):
                return await call_next(request)

            user_id = request.state.user["user_id"]
            path = request.url.path

            # 根据路径选择限流配置
            config = self.config.rate_limits.get(
                path, RateLimitConfig()
            )

            allowed, headers = await self.rate_limiter.check_rate_limit(
                user_id=user_id,
                limit_type="api",
                limit=config.requests_per_minute,
            )

            if not allowed:
                return JSONResponse(
                    status_code=429,
                    content={
                        "error": "rate_limit_exceeded",
                        "message": "请求过于频繁, 请稍后重试",
                        "retry_after_ms": 60_000,
                    },
                    headers=headers,
                )

            response = await call_next(request)
            for k, v in headers.items():
                response.headers[k] = v
            return response

    def _setup_routes(self):
        """配置 API 路由"""
        app = self.app

        @app.get("/health")
        async def health_check():
            """健康检查"""
            return {
                "status": "healthy",
                "timestamp": datetime.utcnow().isoformat(),
                "version": "1.0.0",
            }

        @app.post("/api/v1/agent/reason")
        async def start_reasoning(request: Request):
            """启动推理 (REST 接口 - 非流式)"""
            body = await request.json()
            user = request.state.user

            # 1. 速率限制检查
            allowed, _ = await self.rate_limiter.check_rate_limit(
                user_id=user["user_id"],
                limit_type="reasoning",
                limit=user.get("agent_quota", {}).get("max_requests", 10),
            )
            if not allowed:
                return JSONResponse(status_code=429, content={"error": "rate_limit"})

            # 2. 会话路由
            session_id = str(uuid.uuid4())
            route = await self.session_router.get_or_create_route(
                session_id=session_id,
                user_id=user["user_id"],
                agent_type=body.get("agent_type", "chat_agent"),
            )

            # 3. 熔断保护下的 gRPC 调用
            try:
                result = await self.circuit_breaker.call(
                    "reasoning",
                    self._make_grpc_unary_call,
                    route.reasoning_instance,
                    body,
                )
            except CircuitBreakerOpenError as e:
                return JSONResponse(
                    status_code=503,
                    content={
                        "error": "service_unavailable",
                        "message": str(e),
                        "fallback": True,
                    },
                )

            return {
                "session_id": session_id,
                "result": result,
            }

        @app.websocket("/api/v1/agent/reason/stream/{session_id}")
        async def reason_stream(websocket: WebSocket, session_id: str):
            """
            WebSocket 流式推理接口
            
            通信协议:
                客户端 -> 网关: JSON 消息
                网关 -> 客户端: JSON 消息 (流式推理步骤)
            """
            await websocket.accept()

            # 验证 Token (从 query string 获取)
            token = websocket.query_params.get("token", "")
            try:
                user_info = await self.auth.verify_token(token)
            except AuthError:
                await websocket.send_json({
                    "type": "error",
                    "code": "AUTH_FAILED",
                    "message": "认证失败",
                })
                await websocket.close(code=4001)
                return

            # 限流检查
            allowed, _ = await self.rate_limiter.check_rate_limit(
                user_id=user_info["user_id"],
                limit_type="streaming",
                limit=5,  # 流式连接数限制
            )
            if not allowed:
                await websocket.send_json({
                    "type": "error",
                    "code": "RATE_LIMITED",
                    "message": "并发流式连接过多",
                })
                await websocket.close(code=4002)
                return

            # 获取会话路由
            agent_type = websocket.query_params.get("agent_type", "chat_agent")
            route = await self.session_router.get_or_create_route(
                session_id=session_id,
                user_id=user_info["user_id"],
                agent_type=agent_type,
            )

            # 开启并发计数
            await self.rate_limiter.increment_concurrent(user_info["user_id"])

            try:
                # WebSocket <-> gRPC 流式代理
                await self.grpc_proxy.proxy_reasoning_stream(
                    websocket, session_id, route.reasoning_instance
                )
            except Exception as e:
                logging.error(f"流式代理异常: {e}", exc_info=True)
                try:
                    await websocket.send_json({
                        "type": "error",
                        "code": "STREAM_ERROR",
                        "message": "推理流异常中断",
                    })
                except Exception:
                    pass
            finally:
                await self.rate_limiter.decrement_concurrent(user_info["user_id"])
                await self.session_router.invalidate_session(session_id)

        @app.post("/api/v1/tools/execute")
        async def execute_tool(request: Request):
            """工具执行代理"""
            body = await request.json()
            user = request.state.user

            # 认证传播: 创建内部 Token
            internal_token = self.auth.create_internal_token(
                user_info={"user_id": user["user_id"], "role": user["role"]},
                target_service="tool-executor",
            )

            # 代理请求到 Tool Executor Service
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    "http://tool-executor:8003/api/execute",
                    json=body,
                    headers={
                        "X-Internal-Token": internal_token,
                        "X-Request-Id": str(uuid.uuid4()),
                    },
                    timeout=30.0,
                )

            return response.json()

    async def _make_grpc_unary_call(self, target: str, body: dict) -> dict:
        """一元 gRPC 调用 (非流式)"""
        channel = self.grpc_channels.get(target)
        if not channel:
            channel = grpc.aio.insecure_channel(target)
            self.grpc_channels[target] = channel

        from agent_pb2 import ReasonRequest
        from agent_pb2_grpc import ReasoningServiceStub

        stub = ReasoningServiceStub(channel)
        response = await stub.Reason(
            ReasonRequest(
                prompt=body.get("prompt", ""),
                agent_type=body.get("agent_type", ""),
                session_id=body.get("session_id", ""),
            ),
            timeout=300.0,  # 5 分钟推理超时
        )
        return {"content": response.content, "steps": list(response.steps)}

    async def startup(self):
        """应用启动时初始化"""
        # 连接 Redis
        self.redis = redis.from_url(self.config.redis_url, decode_responses=True)
        await self.redis.ping()

        # 初始化组件
        self.auth = AuthMiddleware(jwt_secret=self.config.jwt_secret)
        self.rate_limiter = RateLimiter(self.redis)
        self.circuit_breaker = CircuitBreaker()
        self.session_router = SessionRouter(self.redis)
        self.grpc_proxy = GrpcStreamProxy(self.grpc_channels)

        print(f"[Gateway] 启动完成, Redis: {self.config.redis_url}")
        print(f"[Gateway] 允许的 Agent 类型: {self.config.allowed_agent_types}")

    async def shutdown(self):
        """应用关闭时清理"""
        # 关闭 gRPC 通道
        for target, channel in self.grpc_channels.items():
            await channel.close()
        # 关闭 Redis
        if self.redis:
            await self.redis.close()
        print("[Gateway] 已关闭所有连接")


# ============================
# 启动入口
# ============================

@asynccontextmanager
async def lifespan(app: FastAPI):
    """FastAPI lifespan 事件"""
    gateway = app.state.gateway
    await gateway.startup()
    yield
    await gateway.shutdown()


def create_app(config: Optional[GatewayConfig] = None) -> FastAPI:
    """创建并配置 Agent 网关应用"""
    if config is None:
        config = GatewayConfig(
            rate_limits={
                "/api/v1/agent/reason": RateLimitConfig(
                    requests_per_minute=30,
                    concurrent_sessions=3,
                ),
                "/api/v1/agent/reason/stream": RateLimitConfig(
                    requests_per_minute=60,
                    concurrent_sessions=5,
                ),
            }
        )

    gateway = AgentGateway(config)
    gateway.app.state.gateway = gateway
    gateway.app.router.lifespan_context = lifespan
    return gateway.app


if __name__ == "__main__":
    import uvicorn

    app = create_app()
    uvicorn.run(
        app,
        host="0.0.0.0",
        port=8080,
        workers=4,      # 多 worker 进程
        loop="uvloop",  # 高性能事件循环
        log_level="info",
    )
```

### 5.2 网关配置示例 (YAML)

```yaml
# agent-gateway-config.yaml
gateway:
  name: agent-gateway-prod
  version: "1.0.0"
  host: "0.0.0.0"
  port: 8080
  workers: 4

# 服务发现
discovery:
  type: consul
  consul_host: "consul:8500"
  service_tags:
    reasoning: ["agent", "reasoning"]
    memory: ["agent", "memory"]
    tool: ["agent", "tool-executor"]

# 路由规则
routes:
  - id: reason-stream
    path: /api/v1/agent/reason/stream/{session_id}
    protocol: websocket
    target_service: reasoning
    timeout: 300s
    rate_limit: 5/s
    session_affinity: true

  - id: reason-unary
    path: /api/v1/agent/reason
    method: POST
    protocol: grpc
    target_service: reasoning
    timeout: 300s
    rate_limit: 30/min

  - id: tool-execute
    path: /api/v1/tools/execute
    method: POST
    protocol: http
    target_service: tool-executor
    timeout: 60s
    rate_limit: 100/min

  - id: memory
    path: /api/v1/memory/*
    protocol: http
    target_service: memory
    timeout: 5s
    rate_limit: 200/min

# 限流策略
rate_limits:
  default:
    requests_per_minute: 60
    burst: 10
  premium_users:
    requests_per_minute: 300
    burst: 30
  agent_types:
    code_agent:
      max_concurrent: 5
      max_duration: 600s
    chat_agent:
      max_concurrent: 10
      max_duration: 120s

# 熔断策略
circuit_breakers:
  reasoning:
    failure_threshold: 20
    recovery_timeout: 30s
    half_open_max: 3
  tool-executor:
    failure_threshold: 10
    recovery_timeout: 10s
    half_open_max: 5

# TLS 配置
tls:
  enabled: true
  cert_path: /etc/certs/gateway.crt
  key_path: /etc/certs/gateway.key
  min_version: TLSv1.3
```

---

## 六、最大挑战

### 6.1 有状态路由 (会话亲和性)

```
问题: Agent 推理是有状态的

每个推理会话包含:
  - LLM 上下文 (prompt + 历史消息, 可能 >100K tokens)
  - 推理状态机 (当前步骤、等待的tool call、中间结果)
  - 内存缓存 (向量索引、对话摘要)

如果路由到不同实例:
  实例 A: 有 session_123 的完整推理上下文
  实例 B: 没有 context, 需要从 0 开始重建 -> 浪费 LLM Token!

解决方案:

1. 一致性哈希:
   session_id -> hash -> 实例
   优点: 实例增减时只有 1/N 的会话需要迁移
   缺点: 热点问题 (某些 session 特别活跃)

2. 粘性 Session (Sticky Session):
   Cookie/Header 绑定到实例
   优点: 简单
   缺点: 实例故障时会话丢失

3. 外部状态存储:
   推理上下文存储在 Redis/数据库
   任何实例都可以恢复
   优点: 无状态路由
   缺点: 序列化/反序列化开销, 延迟增加

4. 建议方案: 一致性哈希 + 外部状态备份
   - 正常时: 一致性哈希路由, 状态在内存
   - 故障时: 从外部存储恢复状态到新实例
   - 容忍度: 秒级故障转移, N-1 故障不丢会话
```

### 6.2 流式编排 (Streaming Orchestration)

```
多路流式数据聚合的复杂性:

场景: 一个查询触发多个 Agent 并行推理

用户: "比较 A 公司和 B 公司的财务健康状况"

网关:
  -> Agent-A (分析 A 公司): 流式返回推理过程
  -> Agent-B (分析 B 公司): 流式返回推理过程
  
  网关需要:
  1. 同时维持 2 个 gRPC 双向流
  2. 接收 Agent-A 和 Agent-B 的流式输出
  3. 合并排序后发送给客户端
  4. 其中一个 Agent 失败时, 部分结果仍可用

挑战:
  - 流量控制: 一个 Agent 流快, 另一个慢, 如何 merge?
  - 内存管理: 流式数据缓冲, 防止 OOM
  - 错误隔离: Agent-A 失败不影响 Agent-B
  - 排序对齐: 需要时间戳/序列号对齐两个流
```

### 6.3 网关成为瓶颈

```
性能分析:

传统网关瓶颈:
  CPU: 加解密 (TLS)、序列化 (JSON)
  内存: 连接管理、请求体缓冲
  网络: 带宽、连接数

Agent 网关瓶颈:
  CPU: 协议转换 (WebSocket <-> gRPC)、Protobuf 编解码
  内存: 每个推理连接保持 15s+ 的 WebSocket + gRPC 双流
        10,000 并发连接 * 每连接 50KB 缓冲区 = 500MB
  网络: 持续流式数据传输
  连接数: 10,000 并发推理 * 每个推理 3 个后端连接 = 30,000 连接

解决方案:
  1. 水平扩展: 多网关实例 + 前置负载均衡
  2. 异步非阻塞: asyncio / uvloop 事件循环
  3. 连接池: gRPC 通道复用
  4. 零拷贝: 减少不必要的内存复制
  5. 协议优化: 二进制协议 (Protobuf) 替代 JSON
  6. 分层网关: L4 负载均衡 -> L7 网关 -> 业务网关
```

### 6.4 网关分层架构

```
大型 Agent 系统的分层网关设计:

                      Internet
                         |
                   +-----------+
                   |  CDN/DNS  |  边缘加速, DDoS 防护
                   +-----+-----+
                         |
                   +-----------+
                   |  L4 LB    |  四层负载均衡 (IPVS/DPVS)
                   +-----+-----+
                         |
                   +-----------+
                   |  L7 LB    |  七层负载均衡 (Nginx/HAProxy)
                   +-----+-----+   TLS 终止, 简化路由
                         |
          +--------------+--------------+
          |              |              |
    +-----v-----+  +----v------+  +----v------+
    | Agent     |  | Agent     |  | Agent     |  <- Agent Gateway 实例
    | Gateway 1 |  | Gateway 2 |  | Gateway 3 |     (水平扩展)
    +-----+-----+  +----+------+  +----+------+
          |              |              |
    +-----v--------------v--------------v------+
    |           Service Mesh (Istio)            |  <- 服务间通信
    |     mTLS, 流量管理, 可观测性             |     流量加密
    +-----+--------------+--------------+------+
          |              |              |
    +-----v-----+  +----v------+  +----v------+
    | Reasoning |  | Memory    |  | Tool      |
    | Service   |  | Service   |  | Executor  |
    +-----------+  +-----------+  +-----------+
```

---

## 七、与 BFF 模式对比

### 7.1 BFF (Backend-for-Frontend) 与 Agent Gateway 的区别

```
+----------------+     +----------------+
|   BFF 模式      |     |  Agent Gateway |
+----------------+     +----------------+

BFF 模式 (Backend-for-Frontend):
  +----------+      +----------+      +----------+
  | Web App  | ---> | BFF-Web  | ---> | Service  |
  +----------+      +----------+      |   A      |
  +----------+      +----------+      |   B      |
  | Mobile   | ---> | BFF-Mob  |      |   C      |
  +----------+      +----------+      +----------+

  特点:
    - 每个客户端类型有独立的 BFF
    - BFF 负责客户端特定数据聚合
    - 通常由前端团队维护
    - 聚焦于"数据怎么展示"

Agent Gateway:
  +----------+      +-------------+     +----------+
  | Web App  | ---> |             | --> | Reasoning|
  +----------+      |   Agent     |     | Service  |
  +----------+      |   Gateway   | --> | Memory   |
  | Mobile   | ---> |             |     | Service  |
  +----------+      +-------------+     | Tool     |
                                         | Executor |
                                         +----------+

  特点:
    - 统一的入口 (不分客户端类型)
    - 聚焦于"推理怎么路由"
    - 由基础设施/后端团队维护
    - 流式处理、会话管理、协议转换
```

### 7.2 何时两者共存

```
大型 Agent 系统的最佳实践: BFF + Agent Gateway 共存

+----------+     +----------+     +-------------+     +----------+
| Web App  | --> | BFF-Web  | --> |             | --> | Reasoning|
+----------+     +----------+     |   Agent     |     | Service  |
+----------+     +----------+     |   Gateway   |     +----------+
| Mobile   | --> | BFF-Mob  | --> |             | --> | Tool     |
+----------+     +----------+     |   (统一入口) |     | Executor |
+----------+     +----------+     |             |     +----------+
| 3rd Party| --> | API      | --> |             |
|   API    |     | Gateway  |     +-------------+
+----------+     +----------+

各层职责:
  BFF: 
    - 客户端特定的数据格式转换
    - UI 状态管理
    - 页面级数据聚合
    
  Agent Gateway:
    - 推理会话路由
    - 流式推理代理
    - 认证/限流 (跨客户端)
    - Agent 协议处理
    
  适用场景:
    BFF 处理"页面需要什么数据"
    Gateway 处理"推理请求去哪里"
```

---

## 八、能力边界

### 8.1 何时使用 Service Mesh 替代

```
API Gateway 与 Service Mesh 的关系:

+------------------+     +------------------+
|  API Gateway     |     |  Service Mesh    |
+------------------+     +------------------+

网关的职责:
  - 南北向流量 (客户端 -> 服务)
  - API 管理
  - 认证 / 限流
  - 协议转换
  - 流式聚合

Service Mesh 的职责:
  - 东西向流量 (服务 -> 服务)
  - mTLS 加密
  - 服务发现
  - 流量分割 (灰度/金丝雀)
  - 重试/超时
  - 可观测性 (链路追踪)

何时用 Service Mesh (而不是在网关实现):

1. 服务间通信治理:
   网关负责 Client->Service, Mesh 负责 Service->Service
   不要在网关实现服务发现和重试逻辑 (已经太复杂)

2. 流量加密:
   Service Mesh 提供自动 mTLS, 无需修改代码
   网关做边缘 TLS 终止, Mesh 做内部 mTLS

3. 灰度发布:
   Service Mesh 支持按 header/权重流量分割
   网关只需要路由到 "service-reasoning"
   Mesh 决定 v1 还是 v2

4. 可观测性:
   Service Mesh 自动注入 tracing header
   网关只需要透传 tracing context
```

### 8.2 网关 Anti-Patterns

```
Agent 网关实现中需要避免的反模式:

1. ❌ 网关处理业务逻辑
   错误: 网关中实现 Prompt 模板拼接
   正确: 网关只负责路由和代理, 业务逻辑在推理服务

2. ❌ 网关直接操作数据库
   错误: 网关中查询用户 Agent 配置
   正确: 网关通过内部 API 调用配置服务

3. ❌ 网关阻塞等待
   错误: 同步等待所有工具结果再返回
   正确: 流式返回部分结果, 异步等待工具

4. ❌ 网关保存推理状态
   错误: 在网关内存中保存推理上下文
   正确: 推理状态保存在推理服务或外部存储

5. ❌ 网关作为单点瓶颈
   错误: 所有流量都必须经过网关单实例
   正确: 网关水平扩展 + 前置负载均衡

6. ❌ 过度协议转换
   错误: JSON -> Protobuf -> Avro -> 内部格式
   正确: 最小化转换次数, 首选统一的内部协议
```

### 8.3 规模参考

```
不同规模下的网关配置参考:

初创期 Agent (< 100 日均请求):
  - 单网关实例 (1 CPU, 2GB RAM)
  - Nginx 反向代理 + 简单的认证限流
  - 直接 HTTP/gRPC 代理
  - 开发成本: 低

成长型 Agent (1000 - 10000 日均请求):
  - 2-3 网关实例 (2 CPU, 4GB RAM)
  - FastAPI/Kong 网关
  - JWT 认证 + Redis 限流
  - WebSocket 流式代理
  - 开发成本: 中

大规模 Agent (10000+ 日均请求):
  - 5+ 网关实例 (4 CPU, 8GB RAM)
  - 分层网关 (L4 LB -> L7 LB -> Agent Gateway)
  - 完整的认证/限流/熔断/监控
  - Service Mesh (Istio)
  - 自定义 Agent 流式协议优化
  - 开发成本: 高

千万级 Agent:
  - 多集群网关部署 (多区域)
  - 全局负载均衡 (Anycast DNS)
  - 网关本地缓存 + 分布式限流
  - 自定义高性能网关 (Go/Rust)
  - 边缘计算: 在 CDN 边缘节点部署轻量网关
  - 开发成本: 极高
```

---

## 九、总结

Agent API 网关是 Agent 系统架构中的关键组件，它承载了从客户端到后端推理服务的所有流量，并处理传统网关没有的**流式推理代理**、**有状态会话路由**、**协议转换**等核心功能。

关键设计要点:

1. **流式优先**: Agent 推理是流式的，网关必须支持 WebSocket 和 gRPC 双向流的透明代理
2. **会话亲和**: 推理是有状态的，网关必须确保同一会话的所有请求路由到同一实例
3. **分层架构**: 大型系统中采用 L4 LB -> L7 LB -> Agent Gateway -> Service Mesh 的分层设计
4. **弹性设计**: 推理时间长、失败代价高，网关必须提供熔断、超时控制、优雅降级
5. **轻量职责**: 网关不做业务逻辑、不操作数据库、不保存状态——保持网关的专注和可靠
6. **可观测性**: 每个推理请求跨越网关、推理服务、工具服务，全链路追踪是调试的基础

API 网关是 Agent 系统的"前门"，一个设计良好的网关能让 Agent 系统更可靠、更安全、更易于扩展。
