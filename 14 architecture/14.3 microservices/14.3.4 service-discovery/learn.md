# 服务发现机制 — Service Discovery in Agent Microservices

> **核心思想**：服务发现是微服务架构的"电话簿"——让 Agent 的各个微服务能够动态找到彼此的位置，而不需要硬编码 IP 地址。在 Agent 系统中，由于 LLM 推理的突发性、工具服务的快速扩缩容、以及会话级路由需求，服务发现不仅是一个基础设施问题，更直接影响 Agent 行为的正确性和连续性。

---

## 1. 基本原理

### 1.1 什么是服务发现

在 Agent 微服务架构中，一个完整的推理-行动周期可能涉及多个服务的协作：

```
Agent 处理一次用户请求的典型服务调用链:

  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
  │  Agent 编排器    │     │  LLM 推理服务    │     │  工具执行服务    │
  │  (Orchestrator)  │────►│  (Inference)     │────►│  (Tool Exec)    │
  │  10 个实例       │     │  20 个实例       │     │  5 个实例       │
  └─────────────────┘     └─────────────────┘     └─────────────────┘
          │                        │                        │
          ▼                        ▼                        ▼
  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
  │  记忆服务       │     │  知识检索服务    │     │  状态持久化服务  │
  │  (Memory)       │     │  (Knowledge)     │     │  (State Store)  │
  │  8 个实例       │     │  15 个实例       │     │  3 个实例       │
  └─────────────────┘     └─────────────────┘     └─────────────────┘
```

**关键问题**：每个服务实例的 IP:Port 是动态变化的（扩缩容、故障转移、滚动更新），Agent 编排器如何知道当前"LLM 推理服务"的哪个实例是可用的？

**服务发现的核心功能**：

```
  服务注册 (Register)          服务发现 (Discover)         健康检查 (Health Check)
  ┌────────────┐              ┌────────────┐              ┌────────────┐
  │ "我上线了!  │──── register─►│  注册中心   │◄──discover──│ "LLM 推理   │
  │  10.0.1.5: │              │  (Registry) │              │  服务在哪?" │
  │  8080"     │              │            │              └────────────┘
  └────────────┘              │  llm-svc:  │                    ▲
  推理服务实例 1               │  10.0.1.5:8080  │              │
                              │  10.0.2.8:8080  │     ┌────────┴────────┐
  ┌────────────┐              │  10.0.3.2:8080  │     │ Agent 编排器    │
  │ "我也上线了!"│──── register─►│            │────►│  Orchestrator   │
  │  10.0.2.8: │              └────────────┘     └─────────────────┘
  │  8080"     │                    │
  └────────────┘                    ▼
  推理服务实例 2             ┌────────────────┐
                           │  健康检查器     │
                           │  (Health Probe)│
                           │  ping:8080/health│
                           └────────────────┘
```

### 1.2 为什么 Agent 系统离不开服务发现

| 场景 | 没有服务发现会发生什么 | 服务发现如何解决 |
|------|------------------------|-----------------|
| **水平扩展** | 新增推理服务实例后，编排器无法感知 | 新实例自动注册，编排器立即发现 |
| **故障转移** | 某个推理实例宕机，请求继续发往失效地址 | 健康检查摘除故障实例，自动路由到健康实例 |
| **滚动更新** | 旧版本实例下线、新版本上线期间请求中断 | 平滑迁移：旧实例注销前完成处理，新实例注册后接管 |
| **地理位置** | 全球部署的 Agent 需要就近路由 | 根据 Registry metadata 做地理位置感知路由 |
| **按能力调度** | 需要指定模型版本（GPT-4 vs GPT-4o-mini）的推理 | 通过标签/元数据注册能力信息，按需筛选 |

---

## 2. 背景与演进

服务发现的演进路径反映了系统规模的增长和运维复杂度的提升：

```
演进路线图:

  阶段 1                   阶段 2                    阶段 3                     阶段 4
  ┌──────────┐          ┌──────────┐            ┌──────────┐              ┌──────────┐
  │ 静态配置  │          │ DNS 轮询 │            │ 服务注册  │              │ 服务网格  │
  │ (硬编码)  │────────►│ (Round-  │───────────►│ 中心     │─────────────►│ (Sidecar │
  │          │          │  Robin)  │            │ (Consul/  │              │  Proxy)   │
  │          │          │          │            │  etcd)    │              │          │
  └──────────┘          └──────────┘            └──────────┘              └──────────┘
       │                     │                       │                         │
       │ 手动管理            │ DNS TTL 缓存           │ 健康检查 +              │ 透明代理 +
       │ IP 变更需重启       │ 负载均衡有限           │ 服务健康状态             │ Istio/Linkerd
       │ 扩展性差            │ 无法处理端口动态       │ 细粒度控制               │ 完全解耦业务
```

### 2.1 阶段一：静态配置文件

```
# config/services.yaml (硬编码时代)
services:
  llm-inference: "10.0.1.5:8080"
  tool-executor: "10.0.2.8:8081"
  memory-store: "10.0.3.2:6379"
  knowledge-retriever: "10.0.4.1:50051"

问题:
  • IP 变更 → 所有依赖方都要修改配置 → 部署重启
  • 无法应对实例增减
  • 没有故障转移能力
  • 测试环境/生产环境配置难同步
```

### 2.2 阶段二：DNS 轮询

```
    ┌──────────────┐      DNS 查询 "llm-svc.example.com"
    │ Agent        │─────────────────────────────────────────►┌──────────────┐
    │ 编排器       │                                          │   DNS 服务器  │
    │              │◄─────────────────────────────────────────│              │
    └──────────────┘    返回: [10.0.1.5, 10.0.2.8, 10.0.3.2]  └──────────────┘
                                                                    │
                                                                    │ A 记录
                                                                    ▼
     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
     │ 推理服务 1   │  │ 推理服务 2   │  │ 推理服务 3   │
     │ 10.0.1.5     │  │ 10.0.2.8     │  │ 10.0.3.2     │
     └──────────────┘  └──────────────┘  └──────────────┘

局限:
  • DNS TTL 缓存导致服务变更延迟 (30s-300s)
  • DNS 无法感知服务健康状态
  • DNS 无法区分端口 (SRV 记录支持有限)
  • Agent 需要重试逻辑来处理故障实例
```

### 2.3 阶段三：服务注册中心

```
当前主流方案，以 Consul 为例:

  ┌───────────────────────────────────────────────────────────────┐
  │                    服务注册中心 (Consul Cluster)                  │
  │                                                               │
  │   ┌────────────────┐     ┌────────────────┐                    │
  │   │  Server 1       │────│  Server 2       │                    │
  │   │  (Leader)       │    │  (Follower)     │                    │
  │   └───────┬────────┘     └────────┬───────┘                    │
  │           │                       │                            │
  │   ┌───────▼────────┐     ┌────────▼───────┐                    │
  │   │  Server 3       │     │  Server 4       │                    │
  │   │  (Follower)     │     │  (Follower)     │                    │
  │   └────────────────┘     └────────────────┘                    │
  │           │                       │                            │
  │           └───────────┬───────────┘                            │
  │                       │                                        │
  │              Raft 共识算法确保一致性                             │
  └───────────────────────────────────────────────────────────────┘
               │                              │
      register /health                   discover /watch
               │                              │
     ┌─────────▼─────────┐         ┌──────────▼──────────┐
     │ 推理服务 1         │         │ Agent 编排器         │
     │ register("llm-svc",│         │ watch("llm-svc")    │
     │  10.0.1.5:8080)    │         │ 实时接收服务变更事件  │
     └───────────────────┘         └─────────────────────┘

优势:
  • 实时健康检查 (秒级检测)
  • 服务变更事件推送 (Watch机制)
  • 支持元数据 (模型版本、区域、权重)
  • 多数据中心支持
```

### 2.4 阶段四：服务网格 (Service Mesh)

```
Agent 系统 + 服务网格的部署拓扑:

  ┌───────────────── Pod / 容器 ─────────────────┐
  │  ┌──────────────────────────────────────────┐ │
  │  │        Agent 编排器容器                    │ │
  │  │  (业务逻辑：推理调度、工具编排)             │ │
  │  │  所有服务调用 → localhost:15001           │ │
  │  └────────────────┬─────────────────────────┘ │
  │                   │                            │
  │  ┌────────────────▼─────────────────────────┐ │
  │  │        Envoy Sidecar Proxy                │ │
  │  │  (透明拦截所有流量)                        │ │
  │  │  服务发现 + 负载均衡 + 重试 + 熔断          │ │
  │  │  → 从 Pilot/Control Plane 获取服务列表     │ │
  │  └──────────────────────────────────────────┘ │
  └────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────┐
  │        Istio Control Plane (控制面)              │
  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
  │  │ Pilot    │ │ Mixer    │ │ Citadel      │   │
  │  │ (服务发现 │ │ (策略)   │ │ (证书管理)    │   │
  │  │  配置推送)│ │          │ │              │   │
  │  └──────────┘ └──────────┘ └──────────────┘   │
  └────────────────────────────────────────────────┘

优势:
  • 完全对业务代码无侵入
  • mTLS 自动加密服务间通信
  • 流量控制：灰度发布、流量镜像
  • 统一的可观测性
```

---

## 3. 服务发现模式

### 3.1 客户端发现模式 (Client-Side Discovery)

Agent 编排器直接查询注册中心，自行选择目标实例。

```
  ┌───────────────────┐     "给我 llm-svc 的实例列表"
  │  Agent 编排器      │────────────────────────────────►┌──────────────┐
  │  (Service Consumer)│                                 │  注册中心     │
  │                    │◄────────────────────────────────│  (Consul)     │
  │  内建负载均衡逻辑   │   返回: [{ip,port,model,region}] └──────────────┘
  │  轮询/一致性哈希   │
  │  会话亲和性        │
  └────────┬──────────┘
           │ 选择: 10.0.1.5:8080 (model=gpt-4, region=us-east)
           │
           ▼
  ┌───────────────────┐
  │  LLM 推理服务实例   │
  │  10.0.1.5:8080    │
  └───────────────────┘

优点:
  • 延迟最低 (直接调用，无需中间代理)
  • 可实现客户端级细粒度路由策略
  • 减少一次网络跳转

缺点:
  • 每个客户端都需要实现服务发现逻辑
  • 语言/框架绑定 (不同语言需要不同库)
  • 客户端版本碎片化问题
```

**Consul 示例 — Agent 编排器的客户端发现**：

```python
import consul
import httpx
import asyncio
from typing import Dict, List, Optional, Any

class AgentServiceDiscovery:
    """基于 Consul 的客户端服务发现 —— 专为 Agent 系统定制"""

    def __init__(self, consul_host: str = "localhost", consul_port: int = 8500):
        self.consul = consul.Consul(host=consul_host, port=consul_port)
        self._cache: Dict[str, List[Dict]] = {}
        self._watchers: Dict[str, asyncio.Task] = {}

    async def discover_llm_service(
        self,
        model: str = "gpt-4",
        region: str = "",
        min_healthy: int = 1,
    ) -> Optional[Dict]:
        """发现最合适的 LLM 推理服务实例"""
        _, services = self.consul.health.service(
            "llm-inference",
            passing=True,       # 只返回通过健康检查的实例
        )

        # 按元数据过滤 (model 版本, region)
        candidates = []
        for svc in services:
            meta = svc.get("Service", {}).get("Meta", {})
            if model and meta.get("model") != model:
                continue
            if region and meta.get("region") != region:
                continue
            candidates.append(svc)

        if len(candidates) < min_healthy:
            raise RuntimeError(
                f"可用 LLM 实例不足: 需要 {min_healthy}, 实际 {len(candidates)}"
            )

        # 简单的负载感知选择 (选当前负载最低的实例)
        selected = min(
            candidates,
            key=lambda s: int(s.get("Service", {}).get("Meta", {}).get("load", 0))
        )
        svc_info = selected["Service"]
        return {
            "id": svc_info["ID"],
            "host": svc_info["Address"],
            "port": svc_info["Port"],
            "model": svc_info.get("Meta", {}).get("model"),
            "region": svc_info.get("Meta", {}).get("region"),
        }

    async def discover_tool_service(
        self,
        tool_name: str,
        capability: str = "",
    ) -> Optional[Dict]:
        """按工具名称和能力发现工具执行服务"""
        _, services = self.consul.health.service(
            f"tool-{tool_name}",
            passing=True,
        )
        if not services:
            # 降级: 尝试发现通用工具执行器
            _, services = self.consul.health.service(
                "tool-executor-generic",
                passing=True,
            )
        if not services:
            return None

        svc = services[0]["Service"]
        return {
            "host": svc["Address"],
            "port": svc["Port"],
            "tool": tool_name,
        }

    def register_agent_instance(
        self,
        service_name: str,
        service_id: str,
        host: str,
        port: int,
        metadata: Dict[str, str],
    ):
        """Agent 实例启动时向 Consul 注册自己"""
        self.consul.agent.service.register(
            name=service_name,
            service_id=service_id,
            address=host,
            port=port,
            meta=metadata,
            check=consul.Check().tcp(host, port, interval="10s", timeout="5s"),
        )

    def deregister(self, service_id: str):
        """Agent 实例优雅关闭时注销"""
        self.consul.agent.service.deregister(service_id)

    def watch_service(self, service_name: str, callback):
        """监听服务变更 (Consul Watch 机制)"""
        index = None
        while True:
            index, data = self.consul.health.service(
                service_name, index=index, wait="30s"
            )
            callback(service_name, data)
```

### 3.2 服务器端发现模式 (Server-Side Discovery)

客户端不直接查询注册中心，而是通过负载均衡器发起请求。

```
                    ┌──────────────────────────┐
                    │   AWS ALB / K8s Service   │
                    │   (负载均衡器)              │
                    │                           │
                    │   统一入口 + 健康检查       │
                    │   自动注册/发现             │
                    └──────────┬────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
       ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
       │ 推理服务 1   │ │ 推理服务 2   │ │ 推理服务 3   │
       │ (滚动更新中)  │ │ (活跃)       │ │ (活跃)       │
       └──────────────┘ └──────────────┘ └──────────────┘

优点:
  • 客户端简单 (只需知道负载均衡器地址)
  • 负载均衡统一管理
  • 对客户端语言/框架无要求

缺点:
  • 多一次网络跳转 (额外延迟约 1-5ms)
  • 负载均衡器成为单点/瓶颈
  • Agent 的会话亲和路由需要额外配置 (Sticky Session)
```

**Kubernetes Service 示例 — 服务器端服务发现**：

```yaml
# k8s-service.yaml
---
# Headless Service: Agent 需要直接发现所有实例 IP (用于会话亲和路由)
apiVersion: v1
kind: Service
metadata:
  name: llm-inference-headless
  namespace: agent-system
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec:
  clusterIP: None  # Headless: DNS 返回所有 Pod IP
  selector:
    app: llm-inference
  ports:
    - name: grpc
      port: 50051
      targetPort: 50051
---
# 普通 Service: 简单轮询负载均衡
apiVersion: v1
kind: Service
metadata:
  name: llm-inference
  namespace: agent-system
spec:
  selector:
    app: llm-inference
  ports:
    - name: grpc
      port: 50051
      targetPort: 50051
  sessionAffinity: ClientIP  # 会话亲和: 相同客户端 IP 路由到相同后端
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # 会话保持 1 小时
---
# StatefulSet: 确保 Agent 会话的稳定性
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: llm-inference
  namespace: agent-system
spec:
  serviceName: llm-inference-headless
  replicas: 5
  selector:
    matchLabels:
      app: llm-inference
  template:
    metadata:
      labels:
        app: llm-inference
    spec:
      containers:
        - name: inference
          image: agent/llm-inference:2.1.0
          ports:
            - containerPort: 50051
              name: grpc
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          # Agent 专用健康检查
          readinessProbe:
            exec:
              command:
                - /health-check
                - --check=model_loaded
                - --check=queue_depth_lt_100
            initialDelaySeconds: 30  # 模型加载时间
            periodSeconds: 15
            timeoutSeconds: 10
          livenessProbe:
            exec:
              command:
                - /health-check
                - --check=process_alive
            initialDelaySeconds: 60
            periodSeconds: 30
          lifecycle:
            preStop:
              exec:
                command:
                  - /graceful-shutdown
                  - --drain-timeout=120  # Agent 推理不能强行中断
```

### 3.3 服务网格模式 (Service Mesh)

完全通过 Sidecar Proxy 透明处理服务发现和通信。

```
Agent 服务网格的流量路径:

  Agent 编排器                           LLM 推理服务
  ┌────────────────────┐                ┌────────────────────┐
  │ 业务容器           │                │ 业务容器           │
  │ 代码:              │                │ 代码:              │
  │ client.call(       │                │ process_request()  │
  │   "llm-svc",       │                │                    │
  │   request)          │                │                    │
  └────────┬───────────┘                └────────▲───────────┘
           │ 通过 localhost 发送到 Sidecar        │
           ▼  (127.0.0.1:15001)                  │
  ┌────────────────────┐                ┌────────┴───────────┐
  │ Envoy Sidecar      │                │ Envoy Sidecar      │
  │                    │                │                    │
  │ 1. 从控制面获取     │──────mTLS─────►│ 1. 接收请求        │
  │    服务端点列表     │   (双向TLS)    │ 2. 负载均衡         │
  │ 2. 选择最合适的实例  │                │ 3. 转发到本地容器   │
  │ 3. 建立 mTLS 连接   │                │                    │
  └────────────────────┘                └────────────────────┘
           │                                       │
           ▼                                       ▼
  ┌─────────────────────────────────────────────────────┐
  │          Istio 控制面 (Pilot)                         │
  │                                                      │
  │  服务发现: 从 Kubernetes API + ServiceRegistry 获取   │
  │  配置下发: xDS 协议推送到所有 Envoy                    │
  │  流量规则: 权重路由 / 故障注入 / 镜像流量              │
  └─────────────────────────────────────────────────────┘

对 Agent 系统的核心价值:
  • 无侵入: Agent 代码不需要任何服务发现逻辑
  • 灰度发布: 新模型版本只路由 5% 流量
  • 故障注入: 测试 Agent 在服务故障时的行为
  • 流量镜像: 生产流量复制到测试环境做对比
```

---

## 4. Agent 系统的特殊需求

### 4.1 会话感知路由 (Session-Aware Routing)

Agent 的推理过程是有状态的——同一个用户的多次交互属于同一个"会话"，应该路由到相同的后端服务实例以保持推理上下文。

```
场景: 用户进行多轮复杂对话

  用户: "帮我分析这份财务报表"                   会话 ID: session-abc
        │
        ▼
  ┌─────────────┐    一致性哈希 / Session ID
  │ API 网关     │────────────────────────────────┐
  └─────────────┘                                │
                                                  ▼
  ┌─────────────────────────────────────────────────────┐
  │                Agent 编排器集群                       │
  │                                                     │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐│
  │  │ 实例 A        │  │ 实例 B        │  │ 实例 C        ││
  │  │ (处理 session- │  │              │  │              ││
  │  │  abc 的对话)   │  │              │  │              ││
  │  └──────────────┘  └──────────────┘  └──────────────┘│
  │        │                                               │
  │        ▼ (所有后续请求必须到达实例 A)                    │
  │  ┌──────────────┐                                     │
  │  │ 局部状态:     │                                     │
  │  │ • 对话历史    │                                     │
  │  │ • 推理轨迹    │                                     │
  │  │ • 已验证信息  │                                     │
  │  │ • 暂存结果    │                                     │
  │  └──────────────┘                                     │
  └─────────────────────────────────────────────────────┘
```

**实现方案 — Consul 元数据 + 一致性哈希**：

```python
import hashlib

class SessionAwareRouter:
    """基于 Session ID 的会话亲和路由"""

    def __init__(self, discovery: AgentServiceDiscovery):
        self.discovery = discovery

    def select_instance(self, service: str, session_id: str, instances: list) -> dict:
        """相同 session_id 总是路由到相同实例"""
        if not instances:
            raise RuntimeError(f"服务 {service} 无可用实例")

        # 一致性哈希: session_id → 实例索引
        hash_val = int(hashlib.sha256(session_id.encode()).hexdigest(), 16)
        idx = hash_val % len(instances)
        return instances[idx]

    async def resolve_with_session(
        self,
        service: str,
        session_id: str,
        model: str = "",
    ) -> dict:
        """会话感知的服务发现"""
        instances = await self.discovery.discover_llm_service(
            model=model, min_healthy=1
        )
        # 但 discover 返回单个实例...需要进一步改造
        pass

# 更好的方式: 注册时携带会话亲和标签
class SessionAwareRegistry:
    """在注册中心中标记会话亲和的实例"""

    def __init__(self, consul_client):
        self.consul = consul_client

    def register_with_session_tag(
        self,
        service_name: str,
        instance_id: str,
        host: str,
        port: int,
        session_id: str = "",
    ):
        """Agent 实例注册时携带会话信息"""
        tags = ["session-aware"]
        if session_id:
            tags.append(f"session-{session_id}")

        self.consul.agent.service.register(
            name=service_name,
            service_id=instance_id,
            address=host,
            port=port,
            tags=tags,
            meta={
                "session_id": session_id,
                "session_affinity": "true",
            },
            check=consul.Check().tcp(host, port, interval="5s"),
        )
```

### 4.2 基于能力发现 (Capability-Based Discovery)

Agent 的服务发现不是简单地按服务名查找，而是按"能力"查找——找到能完成特定任务的实例。

```
传统服务发现:                                     Agent 能力发现:
"找到 llm-inference 服务"                         "找到一个能运行 Python 代码的推理服务"
        │                                                  │
        ▼                                                  ▼
  ┌──────────────┐                                  ┌──────────────┐
  │ llm-inference │                                  │ 能力注册中心   │
  │  实例列表      │                                  │              │
  └──────────────┘                                  │ llm-svc-A:   │
                                                    │  • model=gpt-4│
                                                    │  • code_exec=yes│
                                                    │  • max_tokens=128K│
                                                    │              │
                                                    │ tool-svc-B:  │
                                                    │  • tool=browser│
                                                    │  • headless=yes│
                                                    │  • rate_limit=100/m│
                                                    └──────────────┘

Agent 的能力发现流程:

  Agent 需要: "分析这份 PDF 并总结"
    │
    ├── 需要能力: document_parser (PDF), llm_reasoning, summarizer
    │
    ├── 查询注册中心:
    │    GET /capabilities?need=document_parser&format=pdf
    │    → 返回 [svc-A:1, svc-B:3, svc-C:2]  (按能力匹配度排序)
    │
    ├── 选择最佳服务:
    │    选 svc-B: 支持 PDF、支持中文、负载最低
    │
    └── 路由请求 → svc-B
```

**自定义能力注册中心实现**：

```python
import json
import time
import redis.asyncio as redis
from typing import Dict, List, Optional, Set

class CapabilityRegistry:
    """
    基于 Redis 的 Agent 能力注册中心
    支持: 能力注册 / 能力发现 / 能力权重视图
    """

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.ttl = 60  # 实例心跳超时时间

    async def register(
        self,
        instance_id: str,
        service_name: str,
        host: str,
        port: int,
        capabilities: Dict[str, any],
    ):
        """注册服务实例及其能力集"""
        key = f"agent:instance:{instance_id}"

        # 存储实例基础信息
        instance_data = {
            "service_name": service_name,
            "host": host,
            "port": port,
            "capabilities": json.dumps(capabilities),
            "registered_at": time.time(),
            "ttl": self.ttl,
        }
        await self.redis.hset(key, mapping=instance_data)
        await self.redis.expire(key, self.ttl)

        # 为每个能力建立索引 (能力 → 实例ID)
        for cap_name, cap_config in capabilities.items():
            cap_key = f"agent:capability:{cap_name}"
            await self.redis.sadd(cap_key, instance_id)
            await self.redis.expire(cap_key, self.ttl)

        # 维护按能力类型排序的集合: 支持按匹配度评分
        for cap_name, cap_config in capabilities.items():
            score_key = f"agent:capability:score:{cap_name}"
            # 能力匹配度 (0-100), 越高越优先
            score = cap_config.get("score", 50) if isinstance(cap_config, dict) else 50
            await self.redis.zadd(score_key, {instance_id: score})
            await self.redis.expire(score_key, self.ttl)

    async def find_by_capability(
        self,
        required_caps: List[str],
        min_match: int = 1,
    ) -> List[Dict]:
        """
        按能力查找服务实例
        required_caps: 必需的能力列表
        min_match: 至少匹配的能力数
        """
        if not required_caps:
            return []

        # 找到同时拥有所有必需能力的实例 (集合交集)
        cap_keys = [f"agent:capability:{cap}" for cap in required_caps]
        instance_ids = await self.redis.sinter(cap_keys)

        if not instance_ids or len(instance_ids) < min_match:
            return []

        # 获取实例详细信息
        result = []
        for inst_id in instance_ids:
            data = await self.redis.hgetall(f"agent:instance:{inst_id}")
            if data:
                result.append({
                    "instance_id": inst_id,
                    "host": data.get(b"host", b"").decode(),
                    "port": int(data.get(b"port", 0)),
                    "service_name": data.get(b"service_name", b"").decode(),
                    "capabilities": json.loads(
                        data.get(b"capabilities", b"{}").decode()
                    ),
                })

        return result

    async def find_best_match(
        self,
        required_caps: List[str],
        weights: Dict[str, float] = None,
    ) -> Optional[Dict]:
        """按加权能力匹配找到最佳实例"""
        if not required_caps:
            return None

        # 使用排序集合: 能力匹配度的加权和
        all_scores = {}
        for cap in required_caps:
            score_key = f"agent:capability:score:{cap}"
            scores = await self.redis.zrevrange(
                score_key, 0, -1, withscores=True
            )
            weight = weights.get(cap, 1.0) if weights else 1.0
            for inst_id, score in scores:
                inst_id = inst_id.decode() if isinstance(inst_id, bytes) else inst_id
                all_scores[inst_id] = all_scores.get(inst_id, 0) + score * weight

        if not all_scores:
            return None

        # 选总分最高的
        best_id = max(all_scores, key=all_scores.get)
        data = await self.redis.hgetall(f"agent:instance:{best_id}")
        if not data:
            return None

        return {
            "instance_id": best_id,
            "host": data.get(b"host", b"").decode(),
            "port": int(data.get(b"port", 0)),
            "service_name": data.get(b"service_name", b"").decode(),
            "total_score": all_scores[best_id],
        }

    async def heartbeat(self, instance_id: str):
        """心跳续约"""
        await self.redis.expire(f"agent:instance:{instance_id}", self.ttl)
```

### 4.3 动态 "Scale-to-Zero" 处理

Agent 系统的流量模式通常是突发性的——可能长时间空闲，然后突然涌入大量请求。Scale-to-Zero 模式在没有请求时缩减到 0 个实例，有请求时快速启动。

```
Agent 系统的典型流量模式:

  请求量
    │      ╱╲                    ╱╲
    │     ╱  ╲                  ╱  ╲      ← 用户工作时间
    │    ╱    ╲                ╱    ╲
    │   ╱      ╲──────────────╱      ╲
    │  ╱                                        ← 深夜无人使用
    │ ╱
    └──────────────────────────────────────────→ 时间
         ↓ 缩容到 0           ↓ 急速扩容

Scale-to-Zero 对服务发现的挑战:

  1. 注册中心中不存在该服务 → 调用方必须等待冷启动
  2. 冷启动时间 = 容器启动(5-10s) + 模型加载(10-30s) + 注册(1s)
  3. Agent 不能无限等待 → 需要 Fallback 策略
  4. 多个 Agent 同时发现服务不存在 → 雪崩式启动
```

**Scale-to-Zero 处理策略**：

```python
class ScaleToZeroHandler:
    """
    Scale-to-Zero 场景下的服务发现处理器
    实现: 预 warming + 请求排队 + 快速 fallback
    """

    def __init__(self, discovery, scaler):
        self.discovery = discovery
        self.scaler = scaler          # K8s HPA / 自定义 scaler
        self.pending_requests: Dict[str, asyncio.Event] = {}
        self.max_wait_time = 45       # 最大等待时间 (秒)

    async def discover_or_wait(
        self,
        service: str,
        session_id: str,
        timeout: float = 30.0,
    ) -> Optional[Dict]:
        """发现服务实例，如果不存在则等待冷启动"""
        instance = await self._try_discover(service)
        if instance:
            return instance

        # 服务不存在 (scale-to-zero) → 触发扩容
        await self._trigger_scale_up(service)

        # 等待实例就绪 (带超时)
        wait_event = asyncio.Event()
        self.pending_requests[service] = wait_event

        try:
            await asyncio.wait_for(
                wait_event.wait(),
                timeout=min(timeout, self.max_wait_time),
            )
            return await self._try_discover(service)
        except asyncio.TimeoutError:
            # 超时 → 使用降级方案
            return await self._fallback_strategy(service, session_id)
        finally:
            self.pending_requests.pop(service, None)

    async def _try_discover(self, service: str) -> Optional[Dict]:
        """尝试发现服务"""
        try:
            return await self.discovery.discover_llm_service(model="gpt-4")
        except Exception:
            return None

    async def _trigger_scale_up(self, service: str):
        """触发扩容"""
        await self.scaler.scale_to(service, target_replicas=3)

    async def on_instance_ready(self, service: str):
        """服务实例就绪后的回调"""
        if service in self.pending_requests:
            self.pending_requests[service].set()

    async def _fallback_strategy(
        self,
        service: str,
        session_id: str,
    ) -> Optional[Dict]:
        """降级策略: 当服务不可用时的处理"""
        # 策略1: 使用备用服务 (更小/更慢的模型)
        fallback = await self.discovery.discover_llm_service(model="gpt-4o-mini")
        if fallback:
            return {**fallback, "fallback": True}

        # 策略2: 使用本地缓存的结果
        # 策略3: 返回错误，让 Agent 编排器决定如何处理
        return None
```

---

## 5. 健康检查策略

### 5.1 通用健康检查 — Liveness/Readiness

```
Agent 服务的健康检查层次:

  ┌─────────────────────────────────────────────────────────────┐
  │  Liveness Probe (存活检测)                                   │
  │  问题: "这个服务还在运行吗?"                                  │
  │  失败时: K8s 重启容器                                        │
  │  检查项:                                                     │
  │    • 进程是否存活                                            │
  │    • 主线程是否响应                                          │
  │    • gRPC/HTTP 端口是否监听                                   │
  │                                                              │
  │  示例: curl -f http://localhost:8080/healthz                  │
  └─────────────────────────────────────────────────────────────┘
                               │
  ┌─────────────────────────────────────────────────────────────┐
  │  Readiness Probe (就绪检测)                                  │
  │  问题: "这个服务准备好接收请求了吗?"                           │
  │  失败时: 从服务发现中摘除 (但不重启)                          │
  │  检查项:                                                     │
  │    • 模型是否已加载完毕                                      │
  │    • 缓存是否预热完成                                        │
  │    • 依赖的数据库/消息队列连接是否正常                         │
  │    • 积压队列深度是否可接受                                   │
  │                                                              │
  │  示例: curl -f http://localhost:8080/ready                    │
  └─────────────────────────────────────────────────────────────┘
                               │
  ┌─────────────────────────────────────────────────────────────┐
  │  Agent Startup Probe (启动检测)                               │
  │  问题: "模型加载完了吗?"                                      │
  │  作用: 避免 liveness 在模型加载期间误杀                       │
  │  检查项:                                                     │
  │    • 模型权重是否加载到 GPU                                   │
  │    • Tokenizer 是否初始化                                     │
  │    • 初始化的工具连接是否就绪                                  │
  │                                                              │
  │  示例: curl -f http://localhost:8080/startup                  │
  └─────────────────────────────────────────────────────────────┘
```

### 5.2 Agent 专用健康指标

```python
class AgentHealthChecker:
    """
    Agent 服务专用的健康检查器
    不仅检查进程存活，还检查 Agent 的"认知健康"
    """

    def __init__(self):
        self.metrics = {
            "llm_latency_p99": 5000,       # ms, LLM 推理 P99 延迟
            "tool_availability": {},         # 工具名称 → 是否可用
            "queue_depth": 0,               # 等待队列深度
            "session_count": 0,            # 当前活跃会话数
            "memory_usage_pct": 0,          # 内存使用率
            "last_inference_success": True,  # 最近一次推理是否成功
            "model_loaded": False,          # 模型是否加载
        }

    async def check_readiness(self) -> dict:
        """丰富的就绪检查"""
        status = {
            "ready": True,
            "checks": [],
            "llm_health": await self._check_llm_health(),
            "tool_health": await self._check_tool_health(),
            "resource_health": self._check_resource_usage(),
        }

        # LLM 健康检查
        llm_ok = status["llm_health"]["healthy"]
        if not llm_ok:
            status["ready"] = False
            status["checks"].append("llm: unhealthy")

        # 工具可用性检查
        for tool, available in status["tool_health"].items():
            if not available:
                status["checks"].append(f"tool:{tool}: unavailable")

        # 资源压力检测
        if status["resource_health"]["memory_pct"] > 90:
            status["ready"] = False
            status["checks"].append("resource: memory pressure")

        return status

    async def _check_llm_health(self) -> dict:
        """LLM 推理健康检查: 发一条简单的推理请求"""
        try:
            start = time.time()
            # 模拟: 发一个"Hello"到 LLM，验证返回速度
            # result = await self.llm.infer("Say 'ok'", max_tokens=5)
            latency_ms = (time.time() - start) * 1000

            return {
                "healthy": latency_ms < self.metrics["llm_latency_p99"],
                "latency_ms": latency_ms,
                "model_loaded": self.metrics["model_loaded"],
            }
        except Exception as e:
            return {
                "healthy": False,
                "error": str(e),
                "model_loaded": self.metrics.get("model_loaded", False),
            }

    async def _check_tool_health(self) -> dict:
        """检查所有注册工具的健康状态"""
        healthy = {}
        for tool_name, tool_info in self.metrics["tool_availability"].items():
            try:
                # 发轻量级 ping 到工具服务
                # async with httpx.AsyncClient() as client:
                #     resp = await client.get(tool_info["health_endpoint"])
                #     healthy[tool_name] = resp.status_code == 200
                healthy[tool_name] = True
            except Exception:
                healthy[tool_name] = False
        return healthy

    def _check_resource_usage(self) -> dict:
        """资源使用检查"""
        import psutil
        process = psutil.Process()
        return {
            "memory_pct": process.memory_percent(),
            "cpu_pct": process.cpu_percent(),
            "open_fds": process.num_fds(),
        }
```

### 5.3 优雅关闭 — 保护进行中的推理

Agent 推理过程不能像普通 HTTP 服务那样强行终止——这会丢失推理上下文、浪费已消耗的 Token。

```
关闭流程:

  正常服务                收到 SIGTERM               PreStop Hook
  ┌──────────┐           ┌──────────────┐         ┌──────────────┐
  │ 处理请求   │          │ 停止接收新请求 │         │ 等待进行中    │
  │ 处理请求   │─────────►│ 从注册中心    │────────►│ 的推理完成    │
  │ 处理请求   │          │ 注销          │         │ (最多等 120s)│
  └──────────┘           └──────────────┘         └──────┬───────┘
                                                         │
                    ┌────────────────────────────────────┤
                    ▼                                    ▼
           ┌──────────────┐                    ┌────────────────┐
           │ 保存状态快照   │                    │ 关闭连接池      │
           │ (推理轨迹到    │                    │ (数据库/消息队列)│
           │  Event Store) │                    └────────────────┘
           └──────────────┘
                    │
                    ▼
           ┌────────────────┐
           │ 进程退出        │
           │ (Exit 0)       │
           └────────────────┘
```

```python
import signal
import asyncio

class GracefulShutdown:
    """Agent 服务优雅关闭管理器"""

    def __init__(self, registry, discovery, service_id: str):
        self.registry = registry            # 注册中心客户端
        self.discovery = discovery
        self.service_id = service_id
        self.active_sessions: set = set()   # 正在处理的会话
        self.shutdown_timeout = 120         # 最长等待时间
        self._shutdown_event = asyncio.Event()

    async def register_session(self, session_id: str):
        """记录一个开始处理的会话"""
        self.active_sessions.add(session_id)

    async def complete_session(self, session_id: str):
        """记录一个完成的会话"""
        self.active_sessions.discard(session_id)

    async def shutdown(self, signum=None, frame=None):
        """优雅关闭流程"""
        print(f"[Shutdown] 开始优雅关闭, 活跃会话: {len(self.active_sessions)}")

        # Step 1: 从注册中心注销 (不再接收新请求)
        await self._deregister_from_discovery()
        print("[Shutdown] 已从注册中心注销")

        # Step 2: 等待进行中的推理完成
        await self._wait_for_active_sessions()
        print("[Shutdown] 所有活跃会话已完成")

        # Step 3: 保存推理轨迹快照到持久化存储
        await self._save_state_snapshot()
        print("[Shutdown] 状态快照已保存")

        # Step 4: 关闭外部连接
        await self._close_connections()
        print("[Shutdown] 连接已关闭")

        # Step 5: 退出
        self._shutdown_event.set()

    async def _deregister_from_discovery(self):
        """从所有注册中心注销"""
        self.registry.deregister(self.service_id)
        # 如果是 K8s, Readiness 返回 false 让 Service 自动摘除
        # 如果是 Consul, 调用 agent.service.deregister

    async def _wait_for_active_sessions(self):
        """等待活跃会话完成 (带超时)"""
        if not self.active_sessions:
            return

        waited = 0
        while self.active_sessions and waited < self.shutdown_timeout:
            print(
                f"[Shutdown] 等待 {len(self.active_sessions)} 个会话完成..."
            )
            await asyncio.sleep(5)
            waited += 5

        if self.active_sessions:
            print(
                f"[Shutdown] 超时, 强制终止 {len(self.active_sessions)} 个会话"
            )

    async def _save_state_snapshot(self):
        """保存当前状态快照"""
        # 对于每个活跃会话, 将推理轨迹写入持久化存储
        pass

    async def _close_connections(self):
        """关闭外部连接"""
        # 关闭数据库连接池、消息队列连接、HTTP 连接池
        pass

# 注册信号处理器
def setup_graceful_shutdown(handler: GracefulShutdown):
    loop = asyncio.get_event_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(
            sig,
            lambda s=sig: asyncio.create_task(handler.shutdown(s))
        )
```

---

## 6. 代码示例

### 6.1 Python Consul/Eureka 注册客户端

```python
# comprehensive_service_registry.py
import socket
import time
import threading
from typing import Dict, Optional
import requests

class ConsulRegistryClient:
    """生产级 Consul 服务注册客户端 (支持心跳续约)"""

    def __init__(
        self,
        consul_host: str = "localhost",
        consul_port: int = 8500,
        service_name: str = "agent-service",
        service_port: int = 8080,
        tags: list = None,
        meta: dict = None,
        health_endpoint: str = "/health",
    ):
        self.consul_url = f"http://{consul_host}:{consul_port}"
        self.service_name = service_name
        self.service_id = f"{service_name}-{socket.gethostname()}-{service_port}"
        self.service_port = service_port
        self.tags = tags or ["agent", "llm-inference"]
        self.meta = meta or {}
        self.health_endpoint = health_endpoint
        self._running = False
        self._heartbeat_thread = None

    def register(self):
        """向 Consul 注册服务"""
        host = self._get_private_ip()

        registration = {
            "ID": self.service_id,
            "Name": self.service_name,
            "Address": host,
            "Port": self.service_port,
            "Tags": self.tags,
            "Meta": self.meta,
            "Check": {
                "HTTP": f"http://{host}:{self.service_port}{self.health_endpoint}",
                "Interval": "10s",
                "Timeout": "5s",
                "DeregisterCriticalServiceAfter": "30s",
            },
            "Weights": {"Passing": 1, "Warning": 1},
        }

        resp = requests.put(
            f"{self.consul_url}/v1/agent/service/register",
            json=registration,
        )
        resp.raise_for_status()
        print(f"[Consul] 注册成功: {self.service_id}")

        # 启动心跳
        self._running = True
        self._heartbeat_thread = threading.Thread(
            target=self._heartbeat_loop, daemon=True
        )
        self._heartbeat_thread.start()

    def deregister(self):
        """从 Consul 注销"""
        self._running = False
        resp = requests.put(
            f"{self.consul_url}/v1/agent/service/deregister/{self.service_id}"
        )
        resp.raise_for_status()
        print(f"[Consul] 注销成功: {self.service_id}")

    def _heartbeat_loop(self):
        """心跳续约线程"""
        while self._running:
            try:
                # Consul TTL check 心跳
                requests.put(
                    f"{self.consul_url}/v1/agent/check/pass/"
                    f"service:{self.service_id}",
                    timeout=5,
                )
            except Exception as e:
                print(f"[Consul] 心跳失败: {e}")
            time.sleep(5)  # TTL 的一半 (Health check interval = 10s)

    def _get_private_ip(self) -> str:
        """获取本机私有 IP"""
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        try:
            s.connect(("10.255.255.255", 1))
            return s.getsockname()[0]
        except Exception:
            return "127.0.0.1"
        finally:
            s.close()


class EurekaRegistryClient:
    """Netflix Eureka 注册客户端"""

    def __init__(
        self,
        eureka_url: str = "http://localhost:8761/eureka",
        service_name: str = "AGENT-LLM-INFERENCE",
        service_port: int = 8080,
    ):
        self.eureka_url = eureka_url
        self.service_name = service_name
        self.instance_id = f"{socket.gethostname()}:{service_name}:{service_port}"
        self.service_port = service_port
        self.host = self._get_private_ip()

    def register(self):
        """向 Eureka 注册"""
        payload = {
            "instance": {
                "instanceId": self.instance_id,
                "hostName": self.host,
                "app": self.service_name,
                "ipAddr": self.host,
                "port": {"$": self.service_port, "@enabled": "true"},
                "vipAddress": self.service_name,
                "status": "UP",
                "healthCheckUrl": f"http://{self.host}:{self.service_port}/health",
                "statusPageUrl": f"http://{self.host}:{self.service_port}/info",
                "dataCenterInfo": {
                    "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
                    "name": "MyOwn",
                },
            }
        }

        resp = requests.post(
            f"{self.eureka_url}/apps/{self.service_name}",
            json=payload,
            headers={"Content-Type": "application/json"},
        )
        resp.raise_for_status()

    def send_heartbeat(self) -> bool:
        """发送心跳"""
        resp = requests.put(
            f"{self.eureka_url}/apps/{self.service_name}/{self.instance_id}"
        )
        return resp.status_code == 200

    def _get_private_ip(self) -> str:
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        try:
            s.connect(("10.255.255.255", 1))
            return s.getsockname()[0]
        except Exception:
            return "127.0.0.1"
        finally:
            s.close()
```

### 6.2 K8s Headless Service 配置

```yaml
# agent-headless-service.yaml
# 用于 Agent 系统的 K8s 服务发现完整配置

---
apiVersion: v1
kind: Namespace
metadata:
  name: agent-system
---
# Headless Service: 用于 StatefulSet 的稳定网络标识
apiVersion: v1
kind: Service
metadata:
  name: agent-orchestrator-hl
  namespace: agent-system
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec:
  clusterIP: None
  selector:
    app: agent-orchestrator
  ports:
    - name: grpc
      port: 50051
---
# 普通 ClusterIP Service: 用于无状态负载均衡
apiVersion: v1
kind: Service
metadata:
  name: agent-orchestrator
  namespace: agent-system
  annotations:
    # 会话亲和: 相同 session_id 路由到相同 Pod
    service.agent/session-affinity: "cookie"
spec:
  selector:
    app: agent-orchestrator
  ports:
    - name: grpc
      port: 50051
      targetPort: 50051
    - name: http
      port: 8080
      targetPort: 8080
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 7200
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: agent-orchestrator
  namespace: agent-system
spec:
  serviceName: agent-orchestrator-hl
  replicas: 5
  selector:
    matchLabels:
      app: agent-orchestrator
  template:
    metadata:
      labels:
        app: agent-orchestrator
        agent-discovery: "consul"    # 使用 Consul 做增强发现
    spec:
      serviceAccountName: agent-sa   # 用于访问 K8s API 做自发现
      terminationGracePeriodSeconds: 120  # 优雅关闭宽限期
      containers:
        - name: orchestrator
          image: agent/orchestrator:3.2.0
          ports:
            - containerPort: 50051
              name: grpc
            - containerPort: 8080
              name: http
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            # 让 Agent 知道通过 K8s API 做服务发现
            - name: DISCOVERY_MODE
              value: "k8s-api"
            - name: CONSUL_HOST
              value: "consul-server.consul-system:8500"
          readinessProbe:
            httpGet:
              path: /ready    # Agent 自定义就绪检查
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 20
          lifecycle:
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                    echo "注销 Consul..."
                    consul deregister || true
                    echo "等待进行中推理完成..."
                    sleep 30
                    echo "退出"
---
# 用于 K8s API 服务发现的 RBAC
apiVersion: v1
kind: ServiceAccount
metadata:
  name: agent-sa
  namespace: agent-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: agent-discovery-role
rules:
  - apiGroups: [""]
    resources: ["endpoints", "services", "pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: agent-discovery-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: agent-discovery-role
subjects:
  - kind: ServiceAccount
    name: agent-sa
    namespace: agent-system
```

---

## 7. 实现决策树

选择哪种服务发现方式取决于 Agent 系统的规模、部署环境和运维能力：

```
服务发现方案选型决策树:

开始: 你的 Agent 系统部署在哪里?
  │
  ├── Kubernetes 集群?
  │     ├── 只需要基本发现 → K8s Service (ClusterIP)
  │     ├── 需要会话亲和性 → K8s Service + sessionAffinity: ClientIP
  │     ├── 需要按能力发现 → K8s Service + Consul (双平面)
  │     ├── 已有 Istio/Linkerd → 服务网格 (Sidecar) 方式
  │     └── 需要灰度发布/流量镜像 → 服务网格 (Istio)
  │
  ├── 虚拟机/裸机部署?
  │     ├── 服务数 < 10 → Consul / etcd (直接注册)
  │     ├── 服务数 10-50 → Consul + 客户端负载均衡
  │     └── 服务数 > 50 → Consul + Nginx/HAProxy (服务器端)
  │
  ├── 云原生 (AWS/Azure/GCP)?
  │     ├── AWS → ALB (服务器端) + Cloud Map (注册)
  │     ├── Azure → Azure Load Balancer + Service Fabric
  │     └── GCP → GCLB + Traffic Director
  │
  └── 混合部署/多云?
        └── 服务网格 (Istio + Multi-Cluster)

根据 Agent 系统特性的额外判断:

  Agent 需要会话亲和路由?
    ├── 是 → 一致性哈希 / Sticky Session
    └── 否 → 简单轮询 / 最少连接

  Agent 需要按能力发现?
    ├── 是 → 自定义能力注册中心 (CapabilityRegistry)
    └── 否 → 标准服务名发现

  Agent 流量模式?
    ├── 突发型, 有 Scale-to-Zero → 增加冷启动检测 + 请求排队
    ├── 稳定流量 → 标准 HPA + 就绪检测
    └── 不可预测 → 服务网格 + 自适应负载均衡

推荐方案速查表:
  ┌────────────────────┬─────────────────┬──────────────────────┐
  │ 场景               │ 推荐方案         │ 理由                  │
  ├────────────────────┼─────────────────┼──────────────────────┤
  │ 小规模 (<5 服务)   │ K8s Service     │ 最简单, 零配置        │
  │ 中等规模 (5-20)    │ Consul + 客户端  │ 灵活, 支持元数据路由   │
  │ 大规模 (20-100)    │ 服务网格 (Istio) │ 运维解耦, 统一策略     │
  │ 多云部署            │ Consul + Mesh   │ 跨数据中心同步         │
  │ 企业合规 (mTLS)    │ 服务网格         │ 透明加密, 证书轮换     │
  │ Agent 能力发现      │ Consul Meta +   │ 元数据标签 +          │
  │                    │ Redis 能力索引   │ 自定义能力索引         │
  │ 高吞吐低延迟        │ 客户端发现       │ 少一次网络跳转         │
  └────────────────────┴─────────────────┴──────────────────────┘
```

---

## 8. 最大挑战

### 8.1 网络分区 (Network Partition)

当注册中心集群发生网络分区时，不同分区的 Agent 服务看到不同的服务视图。

```
正常状态:                            网络分区发生后:

  ┌────────────┐    ┌────────────┐      ┌────────────┐    ┌────────────┐
  │ 注册中心    │────│ 注册中心    │      │ 注册中心    │    │ 注册中心    │
  │ 节点 A      │    │ 节点 B      │      │ 节点 A      │    │ 节点 B      │
  │ (Leader)    │    │ (Follower)  │      │ (认为自己是  │    │ (选举新     │
  │            │    │            │      │  Leader)    │    │  Leader)    │
  └────────────┘    └────────────┘      └────────────┘    └────────────┘
         │                │                   │                 │
         ▼                ▼                   ▼                 ▼
  ┌────────────┐    ┌────────────┐      ┌────────────┐    ┌────────────┐
  │ Agent A    │    │ Agent B    │      │ Agent A    │    │ Agent B    │
  │ 看到 2 个   │    │ 看到 2 个   │      │ 只看到自己   │    │ 只看到自己   │
  │ 推理实例    │    │ 推理实例    │      │ (分区 A)    │    │ (分区 B)    │
  └────────────┘    └────────────┘      └────────────┘    └────────────┘

影响:
  • 分区 A 的 Agent 认为分区 B 的服务不健康 → 请求堆积到 A 的服务
  • 分区 B 的服务继续运行但被认为死亡 → Consul 标记为 "critical"
  • 网络恢复后: 注册中心数据冲突需要合并
  • 脑裂 (Split-Brain): 两个独立的 Leader 同时服务

应对策略:
  1. Raft 共识: 大多数节点存活才能选出 Leader (容忍少数节点故障)
  2. 客户端缓存: 即使注册中心不可用, 使用缓存继续工作
  3. 断路器: 当服务调用失败率升高时主动熔断
  4. 多数据中心: 每个数据中心独立注册中心
```

### 8.2 陈旧注册数据 (Stale Registry)

注册中心的数据变更不会立即传播到所有客户端，导致使用已失效的服务地址。

```
时间线:

  t0: 服务实例 A 注册   注册中心: [A, B, C]   所有客户端正确
  t1: 实例 C 宕机       注册中心: [A, B, C]   健康检查还没执行
  t2: Agent 获取服务列表  返回: [A, B, C]     返回了 C!
  t3: Agent 调用 C       连接失败!             请求失败
  t4: 健康检查发现 C 宕机  注册中心: [A, B]    更新了, 但已经晚了

  ← 窗口期: 健康检查间隔 (通常 5-15s) →

陈旧数据的根因:
  • 健康检查间隔 (Check Interval): 5-15s 的检测周期
  • DNS TTL: 30-300s 的缓存时间
  • 客户端缓存: 客户端可能会缓存服务列表
  • 网络延迟: 变更事件传播需要时间

Agent 系统中的影响:
  • 调用已宕机的工具服务 → Agent 推理中断
  • 调用已过载的推理服务 → 请求超时 → Agent 重试 → 加重负载
  • 路由到正在缩容的实例 → 连接拒绝

应对策略:
  1. 客户端重试 + 退避: 失败后重试其他实例
  2. 主动健康检查: 调用前先快速 ping 一下
  3. 短 TTL: Consul 可配置检查间隔为 1s (但有开销)
  4. 熔断器: 跟踪实例的成功/失败率
  5. 服务网格: Sidecar 本地缓存, 快速切换
```

### 8.3 启动竞态条件 (Startup Race)

Agent 服务启动时，多个组件几乎同时注册和发现，导致时序竞争。

```
典型启动竞态:

  时间→
  ┌───────────────────────────────────────────────────────────────┐
  │ Agent 编排器  │         │                  │                  │
  │ 启动流程:      │ 启动   │ 开始注册           │ 开始发现服务      │
  │               │        │ 到 Consul         │                  │
  ├───────────────┼────────┼──────────────────┼──────────────────┤
  │ LLM 推理服务   │  启动   │ 注册到 Consul     │ 模型未加载完       │
  │               │        │ (但模型还在加载)    │ Ready? No!        │
  ├───────────────┼────────┼──────────────────┼──────────────────┤
  │ Agent 编排器   │        │                  │ 发现 llm-svc      │
  │               │        │                  │ → 发送推理请求     │
  ├───────────────┼────────┼──────────────────┼──────────────────┤
  │ LLM 推理服务   │        │                  │ 收到请求但模型     │
  │               │        │                  │ 未加载完成 → 503   │
  └───────────────┴────────┴──────────────────┴──────────────────┘

另一种竞态:
  ┌───────────────────────────────────────────────────────────────┐
  │ 服务 A 注册    │  注册 OK                                     │
  │ 服务 B 启动    │              依赖服务 A, 开始发现              │
  │ 健康检查       │  还没检查到                                    │
  │ 服务 B         │  服务 A 不存在? → 启动失败                     │
  └───────────────┴───────────────────────────────────────────────┘

解决策略:
  1. Readiness Probe + Startup Probe: K8s 提供的启动时序管理
  2. 初始化等待: 启动时等待依赖服务就绪再注册
  3. 延迟发现: 启动后延迟一段时间再开始发现
  4. 失败重试: 发现失败时指数退避重试
  5. Init Container: 在业务容器启动前等待依赖
```

```python
class StartupRaceHandler:
    """解决启动竞态的初始化管理器"""

    def __init__(self, discovery, dependencies: list):
        self.discovery = discovery
        self.dependencies = dependencies  # ["llm-inference", "tool-browser"]
        self.max_retries = 30
        self.retry_delay = 2  # 秒

    async def wait_for_dependencies(self) -> bool:
        """等待所有依赖服务就绪后才注册自己"""
        print(f"[Init] 等待依赖服务就绪: {self.dependencies}")

        for dep in self.dependencies:
            ready = False
            for attempt in range(self.max_retries):
                instance = await self.discovery.discover_llm_service(
                    model="gpt-4"
                )
                if instance:
                    print(f"[Init] 依赖 {dep} 已就绪")
                    ready = True
                    break

                print(
                    f"[Init] 依赖 {dep} 未就绪, "
                    f"等待 {self.retry_delay}s (尝试 {attempt+1}/{self.max_retries})"
                )
                await asyncio.sleep(self.retry_delay)

            if not ready:
                print(f"[Init] 依赖 {dep} 超时, 启动降级模式")
                return False

        print("[Init] 所有依赖就绪, 开始注册自己")
        return True
```

---

## 9. 总结与最佳实践

### 最佳实践清单

1. **服务发现与健康检查一起设计**：没有健康检查的服务发现是虚假的发现
2. **客户端缓存**：即使注册中心短暂不可用，Agent 也不应该停止工作
3. **多级发现**：优先本地 (K8s DNS) → 其次中心 (Consul) → 最后静态备选
4. **会话感知**：Agent 的推理连续性要求路由层面的会话亲和
5. **优雅降级**：服务不可用时提供明确的回退策略，而不是让 Agent 无限等待
6. **元数据丰富**：在注册信息中包含足够的能力、版本、负载信息
7. **启动时序**：Readiness 确保真正就绪后再接收请求
8. **可观测性**：记录服务发现的成功/失败、缓存命中/未命中

### 参考与延伸

- **HashiCorp Consul**: Service Discovery & Service Mesh 官方文档
- **Netflix Eureka**: 客户端服务发现模式的经典实现
- **Kubernetes DNS for Services**: K8s 内建服务发现原理
- **Istio Service Mesh**: 服务网格模式的参考实现
- **etcd**: 分布式键值存储, 常用作服务注册中心后端
- **CNCF Cloud Native Landscape**: Service Discovery 分类全景
