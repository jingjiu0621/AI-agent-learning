# 13.2.2 load-balancing — 负载均衡

**Agent 系统的负载均衡与传统 Web 服务的负载均衡有本质差异：Agent 请求是长时间存活、高度可变、且天然有状态的。** 传统的轮询或最小连接数策略在 Agent 场景下不仅无效，反而可能引发雪崩——如果将长会话频繁迁移，会话状态丢失和上下文重建的开销会远超负载均衡带来的收益。理解 Agent 负载均衡的核心矛盾，是构建生产级多实例 Agent 系统的前提。

## 1. 背景与问题

### Agent 负载均衡的核心挑战

一个标准的 Agent 请求生命周期与传统 HTTP 请求有着根本性的不同：

```
传统 HTTP 请求 (典型 50-500ms):
┌──────────────────────────────────────────────┐
│ 请求到达 → 处理 → 响应                        │
│ 无状态：哪个实例处理都一样                      │
│ 短连接：处理完即释放资源                        │
│ 延迟稳定：P99/P50 通常 < 3x                    │
└──────────────────────────────────────────────┘

Agent 请求 (典型 3-30s):
┌──────────────────────────────────────────────┐
│ 请求到达 → LLM API(1-15s) → 工具调用(0.5-5s) │
│         → LLM API(1-15s) → 响应              │
│ 有状态：需要访问历史对话上下文                    │
│ 长连接：实例需保持会话内存                       │
│ 延迟不可预测：P99/P50 可达 10x+               │
└──────────────────────────────────────────────┘
```

这些差异导致三个核心问题：

1. **会话亲和性 (Session Affinity) 要求**：Agent 的对话上下文通常缓存在实例内存中。如果每次请求都被路由到不同实例，每次都需要从 Redis 或数据库重建上下文——这会增加 200-500ms 的延迟，并增加系统整体负载。

2. **负载估算困难**：传统负载均衡器通过连接数或请求数估算负载，但对 Agent 来说，一个正在等待 LLM 响应的"空闲"连接和一个正在执行复杂工具链的"忙碌"连接，实际负载差异巨大。

3. **LLM API 速率限制的全局性**：多个 Agent 实例可能共享同一个 LLM API Key。如果每个实例独立感知不到全局速率，就会出现"每个实例都在限流阈值内，但总和超出"的情况，导致 429 错误和请求失败。

### 为什么轮询和最小连接数会失败

```
场景：3 个 Agent 实例, 6 个并发会话, 每个会话平均 5s

Round-Robin 轮询:
┌──────┐  ┌──────┐  ┌──────┐
│ S1(3s)│  │ S2(8s)│  │ S3(2s)│
│ S4(4s)│  │ S5(1s)│  │ S6(6s)│
├──────┤  ├──────┤  ├──────┤
│ 共 7s │  │ 共 9s │  │ 共 8s │  ← 看起来均衡
└──────┘  └──────┘  └──────┘
问题：谁在处理最复杂的任务？无法感知！

Least-Connections 最小连接数:
┌──────┐  ┌──────┐  ┌──────┐
│ S1(0) │  │ S2(2) │  │ S3(1) │
│ 新请求→ │  │       │  │       │
├──────┤  ├──────┤  ├──────┤
│ 其实 S1 │  │ S2 有两个 │  │ S3 有一个│
│ 在等 LLM │  │ 简单查询 │  │ 复杂任务 │
└──────┘  └──────┘  └──────┘
问题：连接数不反映真实负载！
```

## 2. 之前是怎么做的

### 传统负载均衡方案

在 Agent 系统出现之前，业界已有成熟的负载均衡方案：

| 方案 | 层级 | 策略 | 适用场景 | Agent 适用性 |
|------|------|------|----------|-------------|
| Nginx / HAProxy | L4/L7 | round-robin, leastconn, IP hash | 通用 HTTP/TCP | 仅适合作前端入口层 |
| K8s Service (ClusterIP) | L4 | round-robin (iptables/IPVS) | 容器内服务间调用 | 不支持会话亲和（除非 externalTrafficPolicy） |
| K8s Ingress (nginx-ingress) | L7 | 多种策略 | 外部流量入口 | 支持 cookie session affinity |
| AWS ALB / NLB | L4/L7 | 轮询、least outstanding request | 云原生 | ALB 支持 stickiness |
| Istio / Linkerd | L7 (sidecar) | 多种流量管理策略 | 服务网格 | 可配置 session affinity |
| DNS 负载均衡 | L3 | 轮询 IP 解析 | 地理级路由 | 粗粒度，适合多区域 |

### 为什么传统方案对 Agent 不够用

```
传统负载均衡器的假设：
  "请求之间是独立的"
  "请求处理时间短且可预测"
  "所有后端实例等价"

Agent 系统的现实：
  "请求属于一个对话会话"
  "处理时间从 1s 到 60s 不等"
  "实例负载取决于当前处理的任务复杂度"
  "LLM API 速率是整个集群的共享瓶颈"
```

具体来说：

1. **Nginx round-robin 对长连接无效**：Nginx 的 upstream 轮询在 HTTP/1.1 keepalive 下只在连接建立时分发，后续请求都走同一连接。Agent 的 WebSocket 或 SSE 连接更是如此。

2. **K8s Service 的 iptables 轮询会破坏会话**：K8s 默认 Service 在每个 TCP 连接建立时做一次 NAT 轮询。Agent 使用 WebSocket 长连接时没问题，但如果使用短轮询或 Server-Sent Events，每次请求可能被路由到不同 Pod。

3. **Least-connections 无法反映 Agent 负载**：一个 Agent 实例可能在同时处理 5 个"简单对话"（每个 1-2s），另一个在处理 1 个"复杂数据分析任务"（需要 30s + 多次 LLM 调用 + 工具链）。后者实际负载远高于前者，但连接数显示的却是 1 < 5。

4. **IP hash 在移动端失效**：移动用户的 IP 频繁切换（WiFi <-> 蜂窝网络），IP hash 无法保持会话。

### 容器化时代的过渡方案

在 Agent 系统出现之前，业界用以下方式尝试解决类似问题：

```yaml
# Nginx session affinity 配置（基于 cookie）
upstream agent_backend {
    ip_hash;  # 或使用 sticky cookie
    
    server agent-1:8000 weight=5;
    server agent-2:8000 weight=3;
    server agent-3:8000 weight=2;
    
    # 但请注意：ip_hash 在移动 / NAT 环境下不稳定
    # sticky cookie 在不支持 cookie 的 Agent 协议下无效
}

# K8s Ingress nginx session affinity
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "agent-session"
    nginx.ingress.kubernetes.io/session-cookie-expires: "3600"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "3600"
```

这些方案的问题是：
- 它们只解决了"会话分发"问题，没有解决"负载感知"问题
- 它们不知道 LLM 速率限制的存在
- 它们无法根据实例的实时状态（队列深度、API 配额剩余）做路由决策

## 3. 核心矛盾

### 矛盾一：无状态 vs 有状态

```
扩展的理想：
┌──────────────┐     ┌──────────────┐
│  无状态实例     │     │  无状态实例     │
│  ┌──────────┐ │     │  ┌──────────┐ │
│  │ 请求 →   │ │     │  │ 请求 →   │ │
│  │ 响应     │ │     │  │ 响应     │ │
│  └──────────┘ │     │  └──────────┘ │
│                │     │                │
│ 可以随意增减    │     │ 对等替换       │
└──────────────┘     └──────────────┘

Agent 的现实：
┌──────────────┐     ┌──────────────┐
│  Agent 实例 A  │     │  Agent 实例 B  │
│ ┌───────────┐ │     │ ┌───────────┐ │
│ │ 会话 #1   │ │     │ │ 会话 #2   │ │
│ │  上下文:  │ │     │ │  上下文:  │ │
│ │  5轮对话  │ │     │ │  3轮对话  │ │
│ │ 工具结果  │ │     │ │ 部分工具  │ │
│ │ ──缓存──  │ │     │ │  执行中   │ │
│ └───────────┘ │     │ └───────────┘ │
│ 会话不能随意迁移 │     │ 迁移代价很高  │
└──────────────┘     └──────────────┘
```

**矛盾本质**：水平扩展要求实例无状态（这样新实例可以随时加入，故障实例可以随时替换），但 Agent 会话要求实例有状态（上下文缓存在内存中，迁移需要重建）。解决方案不是彻底无状态，而是"状态外部化 + 会话亲和"的组合。

### 矛盾二：共享 LLM API Key 的"惊群效应"

```
     ┌──────────────────────────────┐
     │     LLM API Gateway          │
     │     Rate Limit: 1000 RPM     │
     └──────────┬───────────────────┘
                │
    ┌───────────┼───────────┐
    │           │           │
    ▼           ▼           ▼
┌────────┐ ┌────────┐ ┌────────┐
│ 实例 1  │ │ 实例 2  │ │ 实例 3  │
│ 350 RPM │ │ 340 RPM │ │ 340 RPM │ ← 每个都在限流内
│ 总计:   │ │        │ │        │
│ 1030 RPM│ │        │ │        │ ← 超出全局限制!
└────────┘ └────────┘ └────────┘
        产生 429 错误 → 重试 → 更多 429 → 雪崩
```

这个问题比传统限流更复杂，因为：
- 每个实例不知道其他实例的用量
- 限流是全局共享的（同一个 API Key）
- Agent 重试机制可能加剧拥堵
- 不同模型的速率限制不同（GPT-4 vs GPT-3.5）

### 矛盾三：任务长度的极端不均匀

```
Agent 任务长度分布 (实测数据):

请求类型        P50     P95     P99     P99/P50
───────        ───     ───     ───     ────────
简单问答       1.2s    3.1s    5.0s    4.2x
代码生成       3.5s    8.2s    15.0s   4.3x
数据分析       8.0s    25.0s   45.0s   5.6x
多工具链       12.0s   35.0s   60.0s   5.0x
长文档处理     20.0s   50.0s   90.0s   4.5x

如果随机分发：
  实例 A 运气好 → 3 个简单请求 (共 3.6s) → 几乎空闲
  实例 B 运气差 → 1 个长文档处理 (20s) → 满载

轮询分发无法解决这种方差。
```

## 4. 当前主流方案

### 4.1 Session Affinity 策略

Session affinity（粘性会话）确保同一用户/对话的所有请求被路由到同一个 Agent 实例。这是 Agent 负载均衡的"第一道防线"。

#### Cookie-based (最常用)

```
用户请求 → 负载均衡器
         │
         ├── 有 cookie? ──→ 提取 session_id ──→ 路由到对应实例
         │
         └── 无 cookie? ──→ 生成新 cookie ──→ 按策略选实例 ──→ 设置 cookie
```

```yaml
# nginx-ingress 配置示例
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "agent_session"
    nginx.ingress.kubernetes.io/session-cookie-expires: "3600"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "3600"
    nginx.ingress.kubernetes.io/affinity-mode: "persistent"
spec:
  rules:
  - host: agent.example.com
    http:
      paths:
      - path: /api/chat
        pathType: Prefix
        backend:
          service:
            name: agent-service
            port:
              number: 8000
```

```yaml
# Traefik 配置
http:
  routers:
    agent-router:
      rule: "Host(`agent.example.com`)"
      service: agent-service
      middlewares:
        - agent-sticky
      
  services:
    agent-service:
      loadBalancer:
        sticky:
          cookie:
            name: agent_session
            httpOnly: true
            
  middlewares:
    agent-sticky:
      stickiness:
        cookie:
          name: agent_session
```

```yaml
# AWS ALB 配置 (通过 Terraform)
resource "aws_lb_target_group" "agent" {
  name     = "agent-tg"
  port     = 8000
  protocol = "HTTP"
  
  stickiness {
    type            = "lb_cookie"
    cookie_duration = 3600
    enabled         = true
  }
}
```

#### Header-based (适合 WebSocket/SSE)

对于 WebSocket 或 SSE 长连接，无法使用 Cookie。此时可以将 session ID 放在请求 Header 中：

```yaml
# nginx 配置：基于 header 的一致性哈希
upstream agent_backend {
    hash $http_x_session_id consistent;  # consistent = 一致性哈希
    
    server agent-1:8000 max_fails=3 fail_timeout=30s;
    server agent-2:8000 max_fails=3 fail_timeout=30s;
    server agent-3:8000 max_fails=3 fail_timeout=30s;
}
```

#### Session Affinity 的局限性

```
Session Affinity 不能解决所有问题:

✅ 能解决的：
  - 避免频繁的上下文重建
  - 减少 Redis/DB 读取次数
  - 保持内存缓存命中率
  - 客户端无感知

❌ 不能解决的：
  - 实例故障后，该实例上的所有会话丢失
  - 新实例上线时，冷启动无会话
  - 负载不均（某些实例分到"重"用户）
  - LLM 全局速率限制
  - 不能做 A/B 测试或灰度发布
```

### 4.2 应用层负载均衡 (Application-level Load Balancing)

当 Session Affinity 不足以解决负载不均衡时，需要在应用层实现 Agent-aware 的负载均衡器。

#### 设计原理

```
                    ┌──────────────────────┐
                    │  Agent Load Balancer  │
                    │  (自定义应用层)        │
                    └──────┬───────────────┘
                           │
            ┌──────────────┼───────────────┐
            │              │               │
            ▼              ▼               ▼
     ┌────────────┐ ┌────────────┐ ┌────────────┐
     │ Agent 实例 1│ │ Agent 实例 2│ │ Agent 实例 3│
     └──────┬─────┘ └──────┬─────┘ └──────┬─────┘
            │              │               │
            ▼              ▼               ▼
     ┌──────────────────────────────────────────┐
     │            Metrics Collector             │
     │  ┌──────┐ ┌──────┐ ┌──────┐ ┌────────┐ │
     │  │CPU   │ │队列深度│ │活跃会话│ │LLM 配额│ │
     │  └──────┘ └──────┘ └──────┘ └────────┘ │
     └──────────────────────────────────────────┘

路由决策因素:
  weight = f(CPU, 队列深度, 活跃会话数, LLM 剩余配额, 平均延迟)
```

#### 实时指标采集

负载均衡器需要知道每个实例的实时状态。这可以通过两种方式实现：

1. **Push 模式**：Agent 实例定期上报指标到负载均衡器
2. **Pull 模式**：负载均衡器从 Agent 实例拉取指标

```
Push 模式 (推荐):
Agent 实例 → 每 2s → 上报到 Redis Pub/Sub
负载均衡器 ← 订阅 ← 

指标格式:
{
  "instance_id": "agent-3",
  "active_sessions": 12,
  "queue_depth": 3,
  "avg_latency_ms": 2340,
  "llm_quota_remaining": 850,
  "memory_usage_pct": 67,
  "cpu_usage_pct": 23
}
```

### 4.3 一致性哈希 (Consistent Hashing)

一致性哈希是 Agent 负载均衡中最重要的算法之一。它在保证会话亲和的同时，最小化扩缩容时的会话重分配。

#### 原理对比

```
普通取模哈希 (Modulo Hash):
  实例数 = 3                  实例数 = 4
  hash("session_A") % 3 = 1  hash("session_A") % 4 = 0  ← 会话迁移!
  hash("session_B") % 3 = 2  hash("session_B") % 4 = 2  ← 未迁移
  hash("session_C") % 3 = 0  hash("session_C") % 4 = 3  ← 会话迁移!
  
  扩容时：~75% 的会话需要迁移

一致性哈希 (Consistent Hashing):
  hash_ring = [agent_1, agent_2, agent_3]
  
  hash("session_A") → 顺时针到 agent_2
  hash("session_B") → 顺时针到 agent_3
  hash("session_C") → 顺时针到 agent_1
  
  加入 agent_4 后:
  hash("session_A") → 顺时针到 agent_2  ← 未迁移!
  hash("session_B") → 顺时针到 agent_3  ← 未迁移!
  hash("session_C") → 顺时针到 agent_1  ← 未迁移!
  只有落在 agent_4 加入位置附近的会话迁移
  
  扩容时：~25% 的会话需要迁移 (理论最小值)
```

```
一致性哈希环示意图:

               agent_3
           ╱     ●     ╲
         ╱               ╲
        │                 │
 agent_2 ●    新增       ● agent_1
        │    agent_4     │
         ╲    ↓        ╱
          ╲  ●       ╱
           ╲_______╱
           
           session_C → agent_1
           session_D → agent_4 (仅新增的这部分迁移)
           session_A → agent_2
           session_B → agent_3
```

#### 生产实现（Python + Redis）

```python
# consistent_hash.py — 基于 Redis 有序集合的一致性哈希实现
import hashlib
import redis
from typing import List, Optional

class ConsistentHashRouter:
    """基于一致性哈希的 Agent 会话路由器"""
    
    def __init__(
        self,
        redis_client: redis.Redis,
        ring_key: str = "agent:hash_ring",
        virtual_nodes: int = 150
    ):
        self.redis = redis_client
        self.ring_key = ring_key
        self.virtual_nodes = virtual_nodes  # 每个物理节点对应虚拟节点数
    
    def _hash(self, key: str) -> int:
        """计算 64 位哈希值"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    def add_node(self, node_id: str) -> None:
        """添加物理节点并生成虚拟节点"""
        pipeline = self.redis.pipeline()
        for i in range(self.virtual_nodes):
            vnode_key = f"{node_id}:vnode:{i}"
            hash_val = self._hash(vnode_key)
            # ZADD: 将虚拟节点加入有序集合
            pipeline.zadd(self.ring_key, {node_id: hash_val})
        pipeline.execute()
    
    def remove_node(self, node_id: str) -> None:
        """移除物理节点的所有虚拟节点"""
        # ZREMRANGEBYSCORE: 移除属于该节点的所有条目
        # 但更高效的方式是维护节点与虚拟节点的映射
        self.redis.zremrangebyscore(
            self.ring_key, 
            self._hash(f"{node_id}:vnode:0"),
            self._hash(f"{node_id}:vnode:{self.virtual_nodes - 1}")
        )
        # 实际生产建议用独立 set 维护节点列表，逐个移除
    
    def get_node(self, session_key: str) -> Optional[str]:
        """为 session 查找对应的节点"""
        if self.redis.zcard(self.ring_key) == 0:
            return None
        
        session_hash = self._hash(session_key)
        
        # ZRANK: 找到第一个 >= session_hash 的虚拟节点
        # ZRANGEBYSCORE: 获取该位置的节点
        results = self.redis.zrangebyscore(
            self.ring_key,
            session_hash,
            "+inf",
            start=0,
            num=1
        )
        
        if not results:
            # 如果没有更大的，回到最小的（环的起点）
            results = self.redis.zrange(self.ring_key, 0, 0)
        
        return results[0] if results else None


# 使用示例
router = ConsistentHashRouter(redis_client)
router.add_node("agent-pod-1")
router.add_node("agent-pod-2")
router.add_node("agent-pod-3")

# 会话路由
session_id = "user_12345_session_abc"
target = router.get_node(session_id)
# 总是返回同一个节点，除非节点增减
```

### 4.4 LLM-ware 负载均衡

在多模型部署场景下，负载均衡的决策维度还要加上模型选择。

#### 基于任务复杂度的模型路由

```
                     ┌──────────────────┐
                     │   Agent 请求到达    │
                     └────────┬─────────┘
                              │
                     ┌────────▼─────────┐
                     │  复杂度判断器       │
                     │  (规则 / ML 分类器) │
                     └────────┬─────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │ 简单查询       │ │ 中等复杂度     │ │ 复杂推理       │
     │ Haiku /       │ │ Sonnet /     │ │ Opus /        │
     │ GPT-4o-mini   │ │ GPT-4o       │ │ Claude 3 Opus │
     │ 成本: $0.15/M │ │ 成本: $2.50/M│ │ 成本: $15.00/M│
     │ P50: 0.8s     │ │ P50: 1.5s    │ │ P50: 3.0s     │
     └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
            │                │                │
            └────────────────┼────────────────┘
                             │
                     ┌──────▼──────┐
                     │ 负载均衡器    │
                     │ (实例间分发)  │
                     └─────────────┘
```

```python
# model_router.py — 基于任务复杂度的多模型路由
import asyncio
import time
from typing import Dict, List, Optional
from dataclasses import dataclass


@dataclass
class ModelEndpoint:
    """模型端点配置"""
    name: str                    # 如 "haiku", "sonnet", "opus"
    provider: str                # "anthropic", "openai"
    cost_per_1m_tokens: float    # 每百万 token 成本
    p50_latency: float           # 秒
    rate_limit_rpm: int          # 每分钟请求数
    current_rpm: int = 0         # 当前分钟已用请求数
    last_reset: float = 0.0      # 上次重置时间


class ComplexityAwareRouter:
    """复杂度感知的多模型路由 + 负载均衡"""
    
    def __init__(self):
        self.endpoints = {
            "cheap": ModelEndpoint("haiku", "anthropic", 0.15, 0.8, 1000),
            "medium": ModelEndpoint("sonnet", "anthropic", 2.50, 1.5, 500),
            "expensive": ModelEndpoint("opus", "anthropic", 15.00, 3.0, 100),
        }
    
    def classify_complexity(self, request: Dict) -> str:
        """判断请求复杂度"""
        # 简单规则：根据问题长度和关键词判断
        text = request.get("prompt", "")
        length = len(text)
        
        has_code = any(kw in text for kw in [
            "代码", "代码生成", "debug", "实现", "函数"
        ])
        has_analysis = any(kw in text for kw in [
            "分析", "推理", "论证", "比较", "设计", "架构"
        ])
        
        if has_analysis and length > 500:
            return "expensive"
        elif has_code or length > 200:
            return "medium"
        else:
            return "cheap"
    
    async def route_request(self, request: Dict) -> Dict:
        """路由请求到合适的模型 + 实例"""
        tier = self.classify_complexity(request)
        endpoint = self.endpoints[tier]
        
        # 检查速率限制
        now = time.time()
        if now - endpoint.last_reset > 60:
            endpoint.current_rpm = 0
            endpoint.last_reset = now
        
        if endpoint.current_rpm >= endpoint.rate_limit_rpm:
            # 速率超限，降级到下一级
            tiers = ["cheap", "medium", "expensive"]
            idx = tiers.index(tier)
            for lower_tier in tiers[idx-1::-1]:
                lower_endpoint = self.endpoints[lower_tier]
                if lower_endpoint.current_rpm < lower_endpoint.rate_limit_rpm:
                    tier = lower_tier
                    endpoint = lower_endpoint
                    break
            else:
                # 所有模型都限流，返回队列提示
                return {"error": "rate_limited", "retry_after": 30}
        
        endpoint.current_rpm += 1
        
        return {
            "selected_model": endpoint.name,
            "provider": endpoint.provider,
            "estimated_cost": endpoint.cost_per_1m_tokens,
            "estimated_latency": endpoint.p50_latency,
            "tier": tier,
        }
```

## 5. 代码示例：完整的 Agent 负载均衡器

下面是一个生产级的 Agent-aware 负载均衡器实现，综合了会话亲和、实例指标感知和 LLM 限流处理。

### 5.1 负载均衡器核心

```python
# agent_load_balancer.py — Agent 感知的负载均衡器
import asyncio
import json
import time
import hashlib
import random
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass, field
from enum import Enum


class LoadMetric(Enum):
    """负载均衡使用的指标类型"""
    ACTIVE_SESSIONS = "active_sessions"
    QUEUE_DEPTH = "queue_depth"
    AVG_LATENCY = "avg_latency_ms"
    LLM_QUOTA = "llm_quota_remaining"
    MEMORY_USAGE = "memory_usage_pct"
    CPU_USAGE = "cpu_usage_pct"


@dataclass
class AgentInstance:
    """Agent 实例状态"""
    id: str
    host: str
    port: int
    weight: float = 1.0
    
    # 实时负载指标 (由心跳更新)
    active_sessions: int = 0
    queue_depth: int = 0
    avg_latency_ms: float = 0.0
    llm_quota_remaining: int = 1000
    memory_usage_pct: float = 0.0
    cpu_usage_pct: float = 0.0
    
    # 内部状态
    last_heartbeat: float = 0.0
    is_healthy: bool = True
    consecutive_failures: int = 0
    
    def is_stale(self, timeout: float = 10.0) -> bool:
        """检查实例是否失联"""
        return time.time() - self.last_heartbeat > timeout
    
    def compute_score(self) -> float:
        """计算综合负载分数 (越低越空闲)"""
        if not self.is_healthy or self.is_stale():
            return float('inf')
        
        # 加权组合：各指标归一化后加权
        sessions_score = self.active_sessions / 100.0          # 归一化到 0-1
        queue_score = self.queue_depth / 20.0                   # 队列深度归一化
        memory_score = self.memory_usage_pct / 100.0
        latency_score = min(self.avg_latency_ms / 10000.0, 1.0)  # 10s 以上视为满负载
        quota_score = 1.0 - (self.llm_quota_remaining / 1000.0)  # 配额越少，负载越高
        
        return (
            sessions_score * 0.30 +
            queue_score * 0.25 +
            memory_score * 0.15 +
            latency_score * 0.15 +
            quota_score * 0.15
        ) / self.weight


class AgentLoadBalancer:
    """Agent-aware 负载均衡器"""
    
    def __init__(self):
        self.instances: Dict[str, AgentInstance] = {}
        self.session_map: Dict[str, str] = {}  # session_id -> instance_id
        self._lock = asyncio.Lock()
    
    def register_instance(self, instance: AgentInstance) -> None:
        """注册 Agent 实例"""
        self.instances[instance.id] = instance
    
    def update_metrics(self, instance_id: str, metrics: Dict) -> None:
        """更新实例指标 (从心跳接收)"""
        if instance_id not in self.instances:
            return
        
        inst = self.instances[instance_id]
        inst.active_sessions = metrics.get("active_sessions", inst.active_sessions)
        inst.queue_depth = metrics.get("queue_depth", inst.queue_depth)
        inst.avg_latency_ms = metrics.get("avg_latency_ms", inst.avg_latency_ms)
        inst.llm_quota_remaining = metrics.get("llm_quota_remaining", inst.llm_quota_remaining)
        inst.memory_usage_pct = metrics.get("memory_usage_pct", inst.memory_usage_pct)
        inst.cpu_usage_pct = metrics.get("cpu_usage_pct", inst.cpu_usage_pct)
        inst.last_heartbeat = time.time()
        inst.is_healthy = True
        inst.consecutive_failures = 0
    
    def select_instance_weighted(self) -> Optional[AgentInstance]:
        """基于加权负载分数选择实例 (负载越低，被选概率越高)"""
        active = [
            inst for inst in self.instances.values()
            if inst.is_healthy and not inst.is_stale()
        ]
        
        if not active:
            return None
        
        # 计算每个实例的负载分数
        scores = {inst.id: inst.compute_score() for inst in active}
        
        # 转换分数为权重 (负载越低权重越高)
        max_score = max(scores.values())
        if max_score == float('inf') or max_score == 0:
            return random.choice(active)
        
        # 反比权重: 空闲实例获得更高选择概率
        weights = {
            inst_id: max(0.1, max_score - score + 0.01)  # 避免零权重
            for inst_id, score in scores.items()
        }
        
        total_weight = sum(weights.values())
        r = random.uniform(0, total_weight)
        cumulative = 0.0
        
        for inst_id, weight in weights.items():
            cumulative += weight
            if r <= cumulative:
                return self.instances[inst_id]
        
        return active[-1]
    
    async def route_request(
        self,
        session_id: str,
        force_new: bool = False
    ) -> Tuple[Optional[AgentInstance], bool]:
        """
        路由请求到合适的 Agent 实例。
        返回: (instance, is_affinity_hit)
        """
        async with self._lock:
            # 1. 检查是否有现有会话亲和
            if not force_new and session_id in self.session_map:
                instance_id = self.session_map[session_id]
                if instance_id in self.instances:
                    inst = self.instances[instance_id]
                    if inst.is_healthy and not inst.is_stale():
                        return inst, True  # 会话亲和命中
                    else:
                        # 实例不可用，清除映射
                        del self.session_map[session_id]
            
            # 2. 选择最优实例
            instance = self.select_instance_weighted()
            if instance:
                self.session_map[session_id] = instance.id
                return instance, False  # 新绑定
            
            return None, False  # 无可用实例
    
    async def handle_heartbeat(self, instance_id: str, metrics: Dict) -> None:
        """处理实例心跳"""
        if instance_id not in self.instances:
            # 新实例自动注册
            self.register_instance(AgentInstance(
                id=instance_id,
                host=metrics.get("host", "unknown"),
                port=metrics.get("port", 8000),
            ))
        
        self.update_metrics(instance_id, metrics)
    
    def get_summary(self) -> Dict:
        """获取集群负载摘要"""
        instances = []
        for inst in self.instances.values():
            instances.append({
                "id": inst.id,
                "healthy": inst.is_healthy,
                "score": round(inst.compute_score(), 3),
                "sessions": inst.active_sessions,
                "queue": inst.queue_depth,
                "latency_ms": inst.avg_latency_ms,
                "llm_quota": inst.llm_quota_remaining,
            })
        
        return {
            "total_instances": len(self.instances),
            "healthy_instances": sum(1 for i in self.instances.values() if i.is_healthy),
            "active_sessions": sum(i.active_sessions for i in self.instances.values()),
            "instances": sorted(instances, key=lambda x: x["score"]),
        }
```

### 5.2 Agent 实例端指标上报

```python
# agent_metrics.py — Agent 实例端指标采集与上报
import asyncio
import aiohttp
import psutil
import os


class AgentMetricsReporter:
    """Agent 实例指标采集与上报
    
    每个 Agent 实例启动时运行此 reporter，
    定期向负载均衡器上报自身状态。
    """
    
    def __init__(
        self,
        instance_id: str,
        load_balancer_url: str,
        report_interval: float = 2.0,
    ):
        self.instance_id = instance_id
        self.load_balancer_url = load_balancer_url.rstrip("/")
        self.report_interval = report_interval
        self.process = psutil.Process(os.getpid())
        
        # 实例本地指标
        self.active_sessions = 0
        self.queue_depth = 0
        self.avg_latency_ms = 0.0
        self.llm_quota_remaining = 1000
    
    def collect_metrics(self) -> dict:
        """采集当前指标"""
        return {
            "instance_id": self.instance_id,
            "host": os.uname().nodename,
            "port": 8000,
            "active_sessions": self.active_sessions,
            "queue_depth": self.queue_depth,
            "avg_latency_ms": self.avg_latency_ms,
            "llm_quota_remaining": self.llm_quota_remaining,
            "memory_usage_pct": self.process.memory_percent(),
            "cpu_usage_pct": self.process.cpu_percent(),
            "timestamp": asyncio.get_event_loop().time(),
        }
    
    async def report_loop(self):
        """持续上报指标"""
        async with aiohttp.ClientSession() as session:
            while True:
                try:
                    metrics = self.collect_metrics()
                    async with session.post(
                        f"{self.load_balancer_url}/api/health",
                        json=metrics,
                        timeout=aiohttp.ClientTimeout(total=2)
                    ):
                        pass  # 不关心响应
                except (aiohttp.ClientError, asyncio.TimeoutError):
                    # 上报失败不应影响主流程
                    pass
                
                await asyncio.sleep(self.report_interval)
```

### 5.3 LLM Provider 故障转移

```python
# llm_failover.py — LLM Provider 故障转移与负载均衡
import asyncio
import time
from typing import Dict, Optional
from dataclasses import dataclass


@dataclass
class ProviderConfig:
    """LLM Provider 配置"""
    name: str
    api_base: str
    api_key: str
    model: str
    rate_limit_rpm: int
    cost_per_1m_tokens: float
    priority: int           # 越小优先级越高
    timeout_seconds: float = 30.0
    max_retries: int = 2


class LLMProviderBalancer:
    """LLM Provider 负载均衡与故障转移
    
    支持：
    - 多 provider 轮转 (主备切换)
    - 成本优化 (优先选便宜的)
    - 速率感知 (避免 429)
    - 故障自动转移 (熔断)
    """
    
    def __init__(self, providers: list):
        self.providers = sorted(providers, key=lambda p: p.priority)
        
        # 速率追踪
        self._rpm_count: Dict[str, int] = {}
        self._rpm_reset: Dict[str, float] = {}
        
        # 熔断状态
        self._circuit_breaker: Dict[str, dict] = {}
        
        # 延迟统计 (滑动窗口)
        self._latency_window: Dict[str, list] = {}
    
    def _check_rate_limit(self, provider: ProviderConfig) -> bool:
        """检查是否超过速率限制"""
        now = time.time()
        name = provider.name
        
        # 每分钟重置
        if now - self._rpm_reset.get(name, 0) > 60:
            self._rpm_count[name] = 0
            self._rpm_reset[name] = now
        
        return self._rpm_count.get(name, 0) < provider.rate_limit_rpm
    
    def _is_circuit_open(self, provider: ProviderConfig) -> bool:
        """检查熔断器状态"""
        state = self._circuit_breaker.get(provider.name)
        if not state:
            return False
        
        # 半开状态：过了冷却期尝试恢复
        if state["state"] == "open":
            if time.time() - state["opened_at"] > 60:  # 60s 冷却
                state["state"] = "half-open"
                return False
            return True
        
        return False
    
    def _record_failure(self, provider: ProviderConfig):
        """记录失败，触发熔断"""
        state = self._circuit_breaker.setdefault(provider.name, {
            "state": "closed",
            "failures": 0,
            "opened_at": 0,
        })
        
        state["failures"] += 1
        if state["failures"] >= 5:  # 连续 5 次失败触发熔断
            state["state"] = "open"
            state["opened_at"] = time.time()
    
    def _record_success(self, provider: ProviderConfig):
        """记录成功，关闭熔断"""
        state = self._circuit_breaker.get(provider.name)
        if state:
            state["failures"] = 0
            state["state"] = "closed"
    
    def _record_latency(self, provider: ProviderConfig, latency: float):
        """记录延迟"""
        window = self._latency_window.setdefault(provider.name, [])
        window.append(latency)
        # 保留最近 100 个样本
        if len(window) > 100:
            window.pop(0)
    
    def _get_avg_latency(self, provider: ProviderConfig) -> float:
        """获取平均延迟"""
        window = self._latency_window.get(provider.name, [])
        return sum(window) / len(window) if window else 0.0
    
    def select_provider(self, cost_sensitive: bool = True) -> Optional[ProviderConfig]:
        """选择最优 Provider
        
        Args:
            cost_sensitive: 是否优先考虑成本
        Returns:
            选中的 Provider，或 None (全部不可用)
        """
        candidates = []
        
        for provider in self.providers:
            if self._is_circuit_open(provider):
                continue
            if not self._check_rate_limit(provider):
                continue
            
            score = provider.priority
            if cost_sensitive:
                # 成本优化：将成本加入评分
                score += provider.cost_per_1m_tokens * 10
            
            candidates.append((score, provider))
        
        if not candidates:
            return None
        
        # 按优先级选择
        candidates.sort(key=lambda x: x[0])
        return candidates[0][1]
    
    async def call_with_failover(self, request_func, *args, **kwargs):
        """带故障转移的 API 调用
        
        用法:
            result = await balancer.call_with_failover(
                your_llm_call_function,
                prompt="...",
            )
        """
        last_error = None
        
        for attempt in range(len(self.providers) * 2):  # 最多尝试 2 轮
            provider = self.select_provider()
            if not provider:
                raise Exception("所有 Provider 均不可用（限流或熔断）")
            
            try:
                start = time.time()
                result = await asyncio.wait_for(
                    request_func(provider, *args, **kwargs),
                    timeout=provider.timeout_seconds
                )
                latency = time.time() - start
                
                self._record_success(provider)
                self._record_latency(provider, latency)
                self._rpm_count[provider.name] = \
                    self._rpm_count.get(provider.name, 0) + 1
                
                return result
                
            except asyncio.TimeoutError as e:
                self._record_failure(provider)
                last_error = e
                continue
            except Exception as e:
                # 区分 429 (限流) 和其他错误
                if "429" in str(e):
                    # 限流不触发熔断，只是跳过
                    self._rpm_count[provider.name] = \
                        self._rpm_count.get(provider.name, 0) + 100  # 快速用尽配额
                else:
                    self._record_failure(provider)
                last_error = e
                continue
        
        raise last_error  # 所有重试均失败
```

## 6. 能力边界与限制

### 6.1 Session Affinity 的失效场景

```
场景：实例宕机

正常情况:
  用户 A ──→ LB ──→ Agent-1 (亲和)
  用户 B ──→ LB ──→ Agent-2 (亲和)
  用户 C ──→ LB ──→ Agent-3 (亲和)

Agent-2 宕机后:
  用户 B ──→ LB ──→ Agent-1 (重新绑定)
                    ├── 会话上下文丢失
                    ├── 需要从 Redis 重建 (~300ms)
                    ├── 内存缓存失效
                    └── 用户感知到延迟增加

更严重的情况:
  大规模宕机 → 大量会话同时重建
  → Redis 读负载暴增 (惊群效应)
  → 所有新请求延迟增加
  → 连锁反应
```

**解决方案**：
- 预热的"备用会话"池
- 渐进式重建（不要同时重建所有会话）
- 使用 WebSocket 而非轮询，服务端主动通知会话迁移

### 6.2 负载均匀度 vs 缓存命中率的权衡

```
均匀分布策略:
  ├── 负载最均衡           ← 理论上的最优
  ├── 缓存命中率最低       ← 每个实例的语义缓存各自独立
  └── 整体效率不一定最高

会话亲和策略:
  ├── 负载可能不均衡       ← "重用户"集中到同一实例
  ├── 缓存命中率最高       ← 同一用户的相似请求复用缓存
  └── 整体效率可能更高     ← 缓存减少的 LLM 调用 > 负载不均的浪费

量化示例:
  100 用户，均匀分布 vs 会话亲和

  均匀分布 (5 实例，每实例 20 用户):
  缓存命中率: ~10% (用户间问题差异大)
  LLM 调用/小时: 1000 × 90% = 900 次
  Token 成本: $9.00

  会话亲和 (用户按哈希分布):
  缓存命中率: ~35% (同一用户上下文复用)
  LLM 调用/小时: 1000 × 65% = 650 次
  Token 成本: $6.50

  即使亲和策略导致 20% 负载不均，
  整体成本仍降低 28%。
```

### 6.3 集中式 vs 分布式负载均衡

| 维度 | 集中式负载均衡 | 分布式负载均衡 |
|------|--------------|--------------|
| 架构 | 中心 LBR 做决策 | 每个实例自选 target |
| 示例 | Nginx, AWS ALB, HAProxy | 客户端侧负载均衡, service mesh sidecar |
| 决策全局性 | 全局最优 | 局部最优 |
| 单点故障 | 是 (需要 HA 对) | 否 (无中心) |
| 复杂度 | 低 | 高 |
| 扩展上限 | 单 LBR 吞吐量 | 接近线性 |
| 协议感知 | 深 (L7) | 依赖实现 |
| Agent 适用性 | Phase 1-2 | Phase 3+ |

**Agent 系统的建议**：
- 小规模（Phase 1-2）：集中式 LBR（nginx + session affinity）足够
- 大规模（Phase 3+）：分布式 + 集中式混合（多级负载均衡）

## 7. 方案对比

### 综合对比矩阵

```
                     Nginx/      Custom       Istio/      DNS       K8s
                     HAProxy    Agent Router  Linkerd     LB        Service
                     ───────    ───────────   ───────     ───       ──────

会话亲和性           ★★★☆☆     ★★★★★        ★★★★☆      ★☆☆☆☆    ★★☆☆☆
  (Cookie/Header)    (支持)     (自定义)      (支持)

负载感知             ★★☆☆☆     ★★★★★        ★★★☆☆      ☆☆☆☆☆    ☆☆☆☆☆
  (基于实例指标)     (basic)    (全指标)      (连接数)

LLM 限流感知          ☆☆☆☆☆     ★★★★★        ☆☆☆☆☆      ☆☆☆☆☆    ☆☆☆☆☆
  (全局配额)         (不知)     (内置)

故障转移             ★★★★☆     ★★★★☆        ★★★★★      ★★☆☆☆    ★★★☆☆

部署复杂度           ★★★★★     ★★★☆☆        ★★☆☆☆      ★★★★★    ★★★★★
  (5=最简单)

扩展上限             ★★★★☆     ★★★★★        ★★★★★      ★★★★★    ★★★★☆

成本感知路由          ☆☆☆☆☆     ★★★★★        ☆☆☆☆☆      ☆☆☆☆☆    ☆☆☆☆☆
  (模型选择)

协议支持             ★★★★★     ★★★★☆        ★★★★★      ★☆☆☆☆    ★★☆☆☆
  (HTTP/WS/gRPC)

可观测性             ★★★☆☆     ★★★★★        ★★★★★      ☆☆☆☆☆    ★★☆☆☆
```

### 选型决策树

```
你的 Agent 系统规模?

├── ≤ 100 用户 (Phase 1)
│   └── 不需要负载均衡，单实例 + asyncio 足够
│
├── 100 - 1K 用户 (Phase 2 初期)
│   ├── 使用: nginx-ingress + cookie session affinity
│   ├── 或者: AWS ALB + stickiness
│   └── 不需要: 自定义 LB, service mesh
│
├── 1K - 10K 用户 (Phase 2 后期)
│   ├── 使用: 自定义 Agent Router (应用层负载均衡)
│   ├── 加上: 一致性哈希 + Redis 会话存储
│   ├── 考虑: Istio 做基础流量管理
│   └── 监控: LLM API 速率限制的全局视图
│
└── 10K+ 用户 (Phase 3)
    ├── 使用: 多级负载均衡
    │   ├── L1: DNS LB (地理路由)
    │   ├── L2: 全局 LBR (集中式管理)
    │   └── L3: 自定义 Agent Router (应用层决策)
    ├── 加上: 模型路由 (小/中/大模型)
    ├── 加上: 预测性负载均衡 (ML 预测)
    └── 关键: 成本感知路由 + 多 provider 故障转移
```

## 8. 工程优化方向

### 8.1 自适应负载均衡

基于实时指标动态调整路由权重，而不是使用静态配置。

```python
# adaptive_lb.py — 自适应负载均衡
class AdaptiveLoadBalancer:
    """自适应负载均衡器
    
    核心思路：根据历史表现动态调整权重
    - 响应慢的实例获得更低权重
    - 错误率高的实例获得更低权重
    - 通过指数移动平均平滑波动
    """
    
    def __init__(self, alpha: float = 0.3):
        self.alpha = alpha  # EMA 平滑系数
        self.instance_scores: Dict[str, float] = {}
        self.instance_errors: Dict[str, float] = {}
        self.instance_latency: Dict[str, float] = {}
    
    def update_instance_stats(
        self,
        instance_id: str,
        latency: float,
        is_error: bool,
        queue_depth: int,
    ):
        """更新实例统计 (使用 EMA)"""
        a = self.alpha
        
        # 延迟 EMA
        old_lat = self.instance_latency.get(instance_id, latency)
        self.instance_latency[instance_id] = a * latency + (1 - a) * old_lat
        
        # 错误率 EMA
        old_err = self.instance_errors.get(instance_id, 0.0)
        err_val = 1.0 if is_error else 0.0
        self.instance_errors[instance_id] = a * err_val + (1 - a) * old_err
        
        # 综合分数 (越低越好)
        lat_score = self.instance_latency[instance_id] / 10000.0  # 归一化
        err_score = self.instance_errors[instance_id] * 5         # 错误惩罚
        queue_score = queue_depth / 10.0
        
        self.instance_scores[instance_id] = lat_score + err_score + queue_score
```

### 8.2 预测性负载均衡

利用历史模式预测未来负载，提前调整路由策略。

```
数据采集层:
  ┌──────────────────────────────────────────┐
  │  收集维度:                                 │
  │  - 时间戳 (星期几, 几点, 是否节假日)        │
  │  - 用户来源 (哪个渠道)                      │
  │  - 请求类型 (问答/代码/分析)                │
  │  - 当前实例数                              │
  │  - 历史负载 (过去 5/15/30 分钟)             │
  └──────────────────────────────────────────┘

预测模型:
  ┌──────────────────────────────────────────┐
  │  方法 1: 时间序列 (ARIMA / Prophet)        │
  │  → 预测未来 5-30 分钟的请求量              │
  │                                           │
  │  方法 2: 基于规则的负载特征                 │
  │  → "工作日上午 10 点是代码生成请求高峰"     │
  │  → "周五下午简单查询比例升高"              │
  │                                           │
  │  方法 3: 在线学习 (简单线性回归)            │
  │  → 特征: 当天时段 + 历史趋势               │
  │  → 输出: 建议实例数 + 建议路由策略          │
  └──────────────────────────────────────────┘

执行层:
  ┌──────────────────────────────────────────┐
  │  - 提前启动新实例 (预热 30s)               │
  │  - 调整路由权重 (优先保留资源给高价值请求)   │
  │  - 降低非关键请求的模型等级                 │
  │  - 触发缓存预热                            │
  └──────────────────────────────────────────┘
```

```python
# predictive_lb.py — 简单预测性负载均衡
class PredictiveLoadBalancer:
    """基于历史模式的预测性负载均衡"""
    
    def __init__(self):
        # 存储每个时段的历史数据
        # { (weekday, hour): [load_samples] }
        self.history: Dict[Tuple[int, int], List[float]] = {}
    
    def record_load(self, load: float):
        """记录当前负载"""
        import datetime
        now = datetime.datetime.now()
        key = (now.weekday(), now.hour)
        
        if key not in self.history:
            self.history[key] = []
        
        samples = self.history[key]
        samples.append(load)
        # 保留最近 30 天数据
        if len(samples) > 30:
            samples.pop(0)
    
    def predict_load(self) -> float:
        """预测当前时段的预期负载"""
        import datetime
        now = datetime.datetime.now()
        key = (now.weekday(), now.hour)
        
        samples = self.history.get(key, [])
        if not samples:
            return 0.5  # 无数据时返回默认值
        
        # 使用 EMA 预测
        weights = [0.5 ** (len(samples) - i) for i in range(len(samples))]
        total_weight = sum(weights)
        return sum(
            s * w for s, w in zip(samples, weights)
        ) / total_weight
    
    def should_prewarm(self, current_load: float) -> bool:
        """判断是否需要预热新实例"""
        predicted = self.predict_load()
        # 如果预测负载 > 当前负载的 1.5 倍，提前准备
        return predicted > current_load * 1.5
```

### 8.3 多级负载均衡架构

大规模部署 (>10K 用户) 需要多级负载均衡：

```
                    ┌──────────────────────┐
                    │  DNS 负载均衡 (L3)     │
                    │  ┌──────────────────┐ │
                    │  │ us-east route    │ │
                    │  │ eu-west route    │ │
                    │  │ ap-southeast     │ │
                    │  └──────────────────┘ │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  全局 LBR (L4)        │
                    │  (Anycast IP)        │
                    │  按区域转发           │
                    └──────────┬───────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ Region LBR   │   │ Region LBR   │   │ Region LBR   │
   │ (L7 Ingress) │   │ (L7 Ingress) │   │ (L7 Ingress) │
   │ Session      │   │ Session      │   │ Session      │
   │ Affinity     │   │ Affinity     │   │ Affinity     │
   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
          │                    │                    │
          ▼                    ▼                    ▼
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ Agent Router │   │ Agent Router │   │ Agent Router │
   │ (应用层 LB)  │   │ (应用层 LB)  │   │ (应用层 LB)  │
   └──┬───┬───┬──┘   └──┬───┬───┬──┘   └──┬───┬───┬──┘
      │   │   │         │   │   │         │   │   │
      ▼   ▼   ▼         ▼   ▼   ▼         ▼   ▼   ▼
    Agent Pods         Agent Pods         Agent Pods
```

## 9. 适用场景判断

### 决策矩阵

```
评估维度：

用户规模:
  □ < 100:      策略 1 — 单实例，不需要负载均衡
  □ 100-1K:     策略 2 — Ingress session affinity
  □ 1K-10K:     策略 3 — 自定义 Agent Router
  □ 10K+:       策略 4 — 多级负载均衡

会话时长 (平均):
  □ < 10s:       传统 LB 可能足够
  □ 10-60s:      需要 session affinity
  □ > 60s:       必须 session affinity + 检查点机制

延迟敏感度:
  □ 实时对话:    需要低延迟，优先就近路由
  □ 可接受 5s+:  可以集中处理，优化成本
  □ 异步处理:    队列式负载均衡即可

LLM 成本约束:
  □ 宽松:        可以只用高性能模型
  □ 中等:        需要模型路由 (省钱)
  □ 严格:        必须多级模型缓存 + 成本感知路由

用户模式:
  □ 均匀分布:    任意策略均可
  □ 潮汐模式:    需要预测性扩缩容 + 自适应负载均衡
  □ 突发流量:    需要队列缓冲 + 降级策略

选择评分表:
  对于你的场景，对每项打分 (1-5):
  
  策略          用户规模  会话时长  延迟敏感  成本约束  用户模式  总分
  ──────        ────────  ────────  ────────  ────────  ────────  ────
  Session       3         5         4         2         3         17
  Affinity                                          
  
  Custom        5         4         5         5         5         24
  Agent Router                                       
  
  Service Mesh  4         3         3         2         3         15
  
  Multi-level   5         4         5         5         4         23
  LB
```

### Phase 匹配指南

| 阶段 | 推荐方案 | 关键配置 | 不要做的事 |
|------|---------|---------|-----------|
| Phase 1 (1-100 用户) | 无负载均衡 | 单实例 + asyncio | 引入 K8s 或 service mesh |
| Phase 2a (100-1K) | nginx-ingress cookie affinity | 1-3 个 Agent Pod | 自定义应用层 LB |
| Phase 2b (1K-10K) | 自定义 Agent Router | 一致性哈希 + Redis session | 多区域部署 |
| Phase 3 (10K+) | 多级 LB + 模型路由 | DNS + LBR + Agent Router | 全局共享单一 API Key |

### 常见陷阱

```
陷阱 1: "先上 K8s Ingress 负载均衡"
  一个 500 用户的 Agent 系统，不需要 K8s。
  → 2 台服务器 + nginx + Redis 就能处理，运维成本低 10x。

陷阱 2: "所有请求都用同一个模型"
  简单问候也用 Opus → 成本爆表。
  → 80% 简单请求用 Haiku，15% 用 Sonnet，5% 用 Opus。

陷阱 3: "负载均衡器只看连接数"
  连接数不反映 Agent 实际负载。
  → 必须看队列深度 + LLM 配额 + 平均延迟。

陷阱 4: "用户会话永不过期"
  Session affinity 映射无限增长。
  → 设置 TTL (通常 30 分钟无活动即过期)。

陷阱 5: "只依赖一种负载均衡策略"
  没有万能的策略。
  → 多策略组合：session affinity + 负载感知 + 模型路由 + 故障转移。
```

## 总结

Agent 系统的负载均衡不是一个独立的"加个反向代理"的问题，而是一个贯穿架构设计、状态管理和成本控制的系统工程。核心要点：

1. **会话亲和是基础**：没有 session affinity，Agent 的上下文管理成本和延迟都会失控。一致性哈希是生产环境的首选方案。

2. **负载感知是关键**：Agent 负载不能只看连接数，必须结合队列深度、LLM 配额、内存使用率等多维指标。

3. **LLM 限流是全局约束**：无论负载均衡做得多好，共享 API Key 的速率限制始终是集群的"天花板"。全局配额管理和 request coalescing 是必要的补充。

4. **成本感知是进阶**：当规模增长到 Token 成本成为主要支出时，负载均衡不仅要考虑"哪个实例空闲"，还要考虑"哪个模型更经济"。

5. **多级分层是方向**：大规模部署需要 DNS 层(地理) -> LBR 层(会话) -> Agent Router 层(负载感知) -> Provider 层(模型选择) 的多级路由。
