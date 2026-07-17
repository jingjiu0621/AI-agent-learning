# 13.2.4 自动扩缩容 (Auto Scaling) -- Agent 系统的弹性伸缩

## 学习目标

读完本章，你将能够：

- 解释为什么传统的 CPU 利用率 HPA 对 Agent 工作负载无效
- 设计 Agent 系统专用的多信号扩缩容策略（队列深度 + 在途请求 + 错误率）
- 使用 KEDA 和自定义指标实现 Agent 系统的事件驱动扩缩容
- 理解速率限制与扩缩容之间的耦合关系，避免"扩容即降级"
- 为不同的生产规模选择正确的扩缩容策略

---

## 1. 背景与问题：为什么 Agent 扩缩容和传统 Web 服务完全不同

### 1.1 Agent 工作负载的独特性

传统 Web 服务的扩展模型基于一个核心假设：**请求是短寿命的（毫秒到秒级），且 CPU/内存是用量的直接代理指标**。一个典型的 Web 请求：

```
客户端 ──HTTP 请求──→ Web 服务器 ──查询 DB──→ 渲染 HTML ──→ 返回响应
                    CPU ↑ 内存 ↑        等待 I/O        CPU ↑
                    整个过程 50-500ms
```

Agent 请求完全不同。一个典型的 Agent 推理循环：

```
客户端请求
    │
    ▼
Agent 进程 ──→ LLM API 调用 ──→ 等待 1-5 秒 ──→ LLM 返回
    │                                                │
    │    ←────────── CPU 几乎为 0 ──────────→         │
    │                                                │
    │──→ 工具调用 ──→ 等待外部 API ──→ 工具结果返回 ──→│
    │              CPU 几乎为 0          │            │
    │                                    ▼            │
    │                        LLM API 再次调用 ←────────┘
    │                              │
    │                         等待 1-5 秒
    │                              │
    ▼                              ▼
最终响应 ───────── 总耗时 10-30 秒 ────────→
```

关键观察：**Agent Pod 的 CPU 利用率在 5-15% 之间徘徊，即便是队列已经堆满、用户已经感受到明显延迟**。这是因为 Agent 的主要工作在"等待 LLM API 返回"，这是一个纯 I/O 等待操作。

### 1.2 扩缩容悖论：扩容可能反而降低系统性能

```
扩容前的状态（2 个 Pod）：
  Pod A: 队列 5 个请求, 3 个在途 LLM 调用
  Pod B: 队列 3 个请求, 2 个在途 LLM 调用
  LLM API Key: 还剩 15 个并发配额 
  → 系统健康

扩容到 4 个 Pod 后的状态：
  Pod A: 队列 3 个请求, 3 个在途 LLM 调用
  Pod B: 队列 2 个请求, 3 个在途 LLM 调用
  Pod C: 队列 5 个请求（新流量导入）, 4 个在途 LLM 调用（立即发出）
  Pod D: 队列 4 个请求, 4 个在途 LLM 调用（立即发出）
  LLM API Key: 14 个在途调用 = 超出配额!
  → 所有 Pod 同时收到 429 Too Many Requests
  → 所有 Pod 进入指数退避重试
  → 有效吞吐量急剧下降
  → 用户感受到的延迟从 10s 飙升到 60s+
```

这是一个典型的**扩容灾难**：增加 Agent 实例并没有线性增加吞吐量，因为共享的 LLM API 速率限制成了真正的瓶颈。新 Pod 启动后会立即"争夺"有限的 API 并发配额，导致所有现有实例也受影响。

### 1.3 冷启动问题

Agent Pod 的启动不是简单的加载二进制文件。一个新的 Agent 实例在上线前需要：

```
Pod 启动时间线
──────────────────────────────────────────────────────────────
0s     K8s 调度 Pod 到节点
 │
1-3s   拉取容器镜像（如果不在缓存中：10-60s）
 │
3-5s   启动 Python 进程，加载框架（FastAPI、LangChain 等）
 │
5-8s   建立 Redis 连接池（TCP 握手 + 认证）
 │
9-12s  建立向量数据库连接池 + 加载 Embedding 模型到 GPU（如果有）
 │
12-15s 预热 LLM API 连接池（HTTP 连接复用）
 │
15-20s 首次健康检查通过，加入服务发现，开始接收流量
 │
20-25s 首次 LLM 调用（新 Pod 需要建立与 LLM API 的全新连接，DNS 解析 + TLS 握手）
       → 首次请求额外延迟 1-2s
──────────────────────────────────────────────────────────────
总计: 20-25s 才能处理第一个请求
```

在流量突增场景下，这 20-25s 的延迟可能导致请求积压进一步恶化。

---

## 2. 之前是怎么做的

### 2.1 基于 CPU/内存的 K8s HPA

最"传统"的做法是直接使用 Kubernetes 内置的 HorizontalPodAutoscaler：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: agent-hpa-cpu
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agent-worker
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**为什么这对 Agent 无效？**

```
典型 Agent Pod 的 CPU 利用率时间序列（每 15s 采集一次）：

时间 →  0s    15s    30s    45s    60s    75s    90s    105s   120s
Pod A   8%    12%    6%     10%    9%     7%     11%    8%     10%
Pod B   7%    11%    8%     9%     10%    6%     9%     12%    8%

队列深度  50    120    200    150    80     60     200    300    500  ← 飙高但 HPA 不知道
延迟P95 2s    5s     8s     12s    6s     3s     10s    15s    25s

HPA 看到的: CPU 平均 9% → 低于目标 80% → 扩容不需要 → 缩容到 minReplicas
用户感受到的: 越来越慢 → 超时 → 错误 → 投诉
```

CPU 指标不仅不反映问题，还给出**完全相反的信号**：CPU 越低，说明 Pod 越空闲，系统"越健康"——但实际上队列正在堆积。

### 2.2 固定 Pod 数量和手动扩缩容

很多团队在早期使用固定数量的 Pod：

```
Phase 1: 固定 2 个 Pod → 低峰期浪费资源，高峰时段用户超时
Phase 2: 固定 10 个 Pod → 高峰期够用，低峰期浪费 70% 资源，月费多出 $2000+
Phase 3: 人工值班扩缩容
         运维: "监控告警了，我去加 5 个 Pod"
         运维: "忘了缩回来，多跑了一晚上，额外花了 $500"
```

### 2.3 基于 Cron 的定时扩缩容

对于可预测的流量模式（如工作时段高峰期），定时扩缩容能部分解决问题：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: agent-hpa-cron
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agent-worker
  minReplicas: 2
  maxReplicas: 20
  behaviors:
    scaleDown:
      policies:
      - type: Pods
        value: 1
        periodSeconds: 300  # 缓慢缩容
---
# 结合 K8s CronJob 在 8:00 和 18:00 手动调整 minReplicas
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-up-morning
spec:
  schedule: "0 8 * * 1-5"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: kubectl
            image: bitnami/kubectl
            command:
            - kubectl
            - scale
            - deployment/agent-worker
            - --replicas=10
```

**局限性：**

- 无法应对突发流量（营销活动、故障转移导致的流量叠加）
- Agent 工作负载的波动性比 Web 服务更大（LLM 响应时间的波动会"放大"流量变化）
- 难以预测 Agent Chain 复杂度的分布变化（同一个 QPS，简单问答和深度推理的资源消耗相差 10x）

---

## 3. 核心矛盾

### 3.1 用什么信号驱动扩缩容？

这是 Agent 扩缩容最根本的问题。一个 Agent 系统中有多个潜在的扩缩容信号，每个都有其盲区：

```
可用的信号及其局限性：

信号                    Agent 负载相关性    延迟感知    问题
────────────────────    ────────────────    ──────    ──────────────────────
CPU 利用率              低 (0.1-0.3)       否        LLM 等待占主导，CPU 不反映负载
内存使用率               低 (0.2-0.4)       否        内存通常稳定，不随请求波动
请求 QPS                中 (0.5-0.7)       部分      不考虑请求复杂度差异
队列深度 (Queue Depth)   高 (0.8-0.9)       是        最佳单一指标，但需配置缓冲区大小
在途 LLM 请求数         高 (0.8-0.9)       是        直接反映 LLM API 使用压力
P50/P95 延迟            高 (0.7-0.9)       是        滞后指标（问题发生后才能检测）
错误率 (429/5xx)        中 (0.5-0.7)       否        滞后信号，且是非连续的
API 速率限制余量         高 (0.8-0.9)       是        需要 API provider 支持返回头信息
Token 消耗速率           中 (0.5-0.7)       否        与负载相关但滞后
```

**核心洞察：没有单一指标完美反映 Agent 扩容需求。最佳实践是组合多个信号进行加权决策。**

### 3.2 扩容速度的困境

```
扩容速度的两难：

需要扩容：3 分钟内新增 10 个 Pod
   │
   ├── 太快 (30s 内全量启动)
   │    ├── 10 个新 Pod 同时开始发 LLM 请求
   │    ├── API 速率限制瞬间被击穿
   │    ├── 所有 Pod（包括旧的）都收到 429
   │    ├── 指数退避重试导致有效容量骤降
   │    └── → 系统性能不升反降
   │
   └── 太慢 (分批扩容，每 60s 加 2 个)
        ├── 新 Pod 逐步上线
        ├── API 速率限制安全
        ├── 但队列仍在增长
        ├── 用户延迟持续恶化
        └── → 扩容速度追不上负载增长
```

**解决方案是引入速率限制感知的扩容节奏**：扩容时不仅要问"需要多少 Pod"，还要问"增加的 Pod 能获得多少 API 配额"。

### 3.3 有状态扩缩容的矛盾

Agent 系统天然有状态。每个会话包含：

```
每个 Agent 会话的"状态包裹"大小：

┌──────────────────────────────────────────────────┐
│                 Agent 会话状态                      │
├──────────────────────────────────────────────────┤
│ 1. 对话历史: 5-50K tokens (~20-200KB)             │
│ 2. 推理轨迹: 中间思考步骤，工具调用记录              │
│ 3. 已获取的上下文: 检索到的文档、数据库查询结果       │
│ 4. 会话级变量: 用户信息、任务状态、临时的中间数据      │
│ 5. 工具调用认证: OAuth token、API key 引用          │
│ 6. 记忆缓存: 最近几轮交互的向量 Embedding           │
└──────────────────────────────────────────────────┘
```

当缩容一个正在处理会话的 Pod 时：

```
缩容前:
  Pod-A (Session-123) ──→ LLM 调用进行中 ──→ 等待响应
  Pod-B (Session-456) ──→ 工具执行中

缩容决定: 移除 Pod-A

问题 1: Session-123 的 LLM 调用怎么办？
  → 如果强制终止，用户的请求丢失，需要重试
  → 如果等待调用完成再销毁，缩容延迟可能长达 30s+

问题 2: Session-123 迁移到 Pod-C 后
  → Pod-C 需要加载完整的对话上下文
  → 重建内存中的缓存状态
  → 首次响应延迟增加 1-3s

问题 3: 会话亲和性 (Session Affinity)
  → 负载均衡器需要把 Session-123 路由到 Pod-C
  → 如果亲和性配置不正确，用户可能被路由到空 Pod
```

---

## 4. 当前主流方案

### 4.1 KEDA: Kubernetes Event-Driven Autoscaling

KEDA 已经成为 Agent 系统事件驱动扩缩容的事实标准。它通过"Scaler"抽象层连接各种指标源，并将指标转换为 Kubernetes HPA 可理解的格式。

```
KEDA 架构概览:

                         ┌─────────────┐
                         │  Kubernetes  │
                         │   API Server │
                         └──────┬──────┘
                                │
                    ┌───────────┴───────────┐
                    │     KEDA Operator     │
                    │  ┌─────────────────┐  │
                    │  │  Metrics Adapter │  │
                    │  └─────────────────┘  │
                    └──┬───────┬───────┬───┘
                       │       │       │
              ┌────────┘       │       └────────┐
              ▼                ▼                 ▼
       ┌────────────┐  ┌────────────┐  ┌──────────────┐
       │  Prometheus │  │  RabbitMQ  │  │   Custom     │
       │  Scaler    │  │  Scaler   │  │  HTTP Scaler │
       └────────────┘  └────────────┘  └──────────────┘
              │                │                │
              ▼                ▼                ▼
       Agent 队列长度    消息积压数        LLM API 余量
```

#### KEDA ScaledObject 的核心配置

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: agent-worker-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agent-worker
  minReplicaCount: 2
  maxReplicaCount: 20
  pollingInterval: 15        # 每 15s 检查一次指标
  cooldownPeriod: 120        # 指标低于阈值后等待 120s 再缩容
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 180   # 缩容稳定窗口
          policies:
          - type: Percent
            value: 20                       # 每次最多缩 20% Pod
            periodSeconds: 60
        scaleUp:
          stabilizationWindowSeconds: 30    # 扩容稳定窗口（短一些）
          policies:
          - type: Percent
            value: 100                      # 每次最多翻倍
            periodSeconds: 30
          - type: Pods
            value: 4                        # 或者最多加 4 个 Pod
            periodSeconds: 15               # 每 15s 评估一次（快速扩容）
          selectPolicy: Max                 # 取两个策略中更激进的
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring:9090
      metricName: agent_queue_depth_total
      query: |
        sum(rate(agent_queue_depth{namespace="production"}[1m]))
      threshold: "50.0"
      activationThreshold: "10.0"           # 低于此值时缩容到 minReplicaCount
```

#### KEDA 各 Scaler 在 Agent 系统中的典型用途

| Scaler 类型 | Agent 系统中最适合 | 信号延迟 | 配置复杂度 |
|------------|------------------|---------|-----------|
| Prometheus | 所有自定义 Agent 指标（队列深度、延迟、错误率） | 15-60s | 中 |
| RabbitMQ / Kafka | 请求队列积压（事件驱动的 Agent 处理） | 实时 | 低 |
| HTTP (自定义) | 调用外部监控系统的 Agent 特定信号 | 取决于外部系统 | 中 |
| Postgres / MySQL | 数据库中的任务表行数（任务型 Agent） | 实时 | 低 |
| Redis | Redis 列表长度（直接读取内存队列） | 实时 | 低 |
| CPU / Memory | 仍可作为辅助信号 | 15s | 最低 |
| K8s Workload | 基于其他工作负载的扩缩（级联扩缩容） | 分钟级 | 低 |

#### KEDA 的高级模式：多触发器组合

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: agent-multi-trigger
spec:
  scaleTargetRef:
    name: agent-worker
  minReplicaCount: 2
  maxReplicaCount: 30
  triggers:
  # 主触发器：队列深度（权重最高）
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring:9090
      metricName: agent_queue_depth
      query: avg(agent_queue_depth{queue="agent-requests"}) > 10
      threshold: "1.0"           # 当 > 10 时触发
  # 辅助触发器：P95 延迟偏离基线
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring:9090
      metricName: agent_latency_p95_ratio
      query: |
        (
          histogram_quantile(0.95, sum(rate(agent_request_duration_seconds_bucket[5m])) by (le))
          /
          avg_over_time(histogram_quantile(0.95, sum(rate(agent_request_duration_seconds_bucket[5m])) by (le))[7d:5m])
        )
      threshold: "2.0"           # P95 延迟超过基线的 2 倍时触发
  # 减速触发器：错误率过高时停止扩容
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring:9090
      metricName: agent_error_rate_too_high
      query: |
        rate(agent_request_errors_total{error_type="429"}[2m])
        / ignoring(error_type)
        rate(agent_requests_total[2m])
        > 0.05
      threshold: "1.0"           # 429 错误率 > 5% → 触发降级模式
```

### 4.2 基于指标的 Agent 专用扩缩容

KEDA 很好，但它是通用框架。Agent 系统的最佳实践是构建**多层级指标**体系：

```
Agent 扩缩容指标体系：

┌──────────────────────────────────────────────────────────────┐
│                     L1: 领先指标 (Leading)                      │
│  用于扩容决策 — 提前检测负载变化                                 │
│                                                              │
│  队列深度 ──── 最重要的单一指标                                  │
│  ├── 绝对深度: 当前请求队列中的等待数                              │
│  ├── 变化率: 队列增长速率 (d(queue)/dt > 0 表示需要扩容)          │
│  └── 预估等待时间: queue_depth / (consumption_rate_per_pod × pods) │
│                                                              │
│  在途 LLM 请求数                                               │
│  ├── 绝对值: 当前所有 Pod 发起的 LLM 调用总数                     │
│  └── API 配额占比: 在途请求 / API 并发配额                        │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                     L2: 同步指标 (Coincident)                   │
│  用于验证扩容决策是否有效                                       │
│                                                              │
│  请求延迟: P50/P95/P99                                          │
│  ├── 升高中 → 扩容还不够                                         │
│  └── 下降中 → 扩容正在生效                                       │
│                                                              │
│  Pod 利用率: 每个 Pod 实际处理的并发请求数                        │
│  ├── 高利用率 + 低延迟 = 系统健康                                │
│  └── 高利用率 + 高延迟 = 需要扩容                                │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                     L3: 滞后指标 (Lagging)                      │
│  用于缩容决策和回顾分析                                         │
│                                                              │
│  错误率: 429 限流、5xx 服务端错误、超时                          │
│  ├── 错误率上升 → 停止扩容，甚至可能需要降级                      │
│  └── 错误率持续下降 → 扩容策略有效                               │
│                                                              │
│  Token 消耗速率                                                │
│  ├── 衡量实际处理能力                                           │
│  └── 结合每请求 Token 数分析扩缩容效率                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### 复合扩缩容评分模型

```python
# composite_scorer.py -- 扩缩容决策引擎

class AgentAutoScaler:
    """
    Agent 系统的复合扩缩容决策引擎。

    将多个信号加权组合成一个"扩缩容分数"，
    同时考虑 LLM API 速率限制作为硬约束。
    """

    def __init__(self, config: dict):
        self.queue_depth_weight = config.get("queue_depth_weight", 0.4)
        self.latency_weight = config.get("latency_weight", 0.3)
        self.error_rate_weight = config.get("error_rate_weight", 0.1)
        self.inflight_weight = config.get("inflight_weight", 0.2)

        # API 速率限制
        self.api_rate_limit = config["api_rate_limit"]  # 最大并发数
        self.api_safety_margin = config.get("api_safety_margin", 0.8)  # 安全阈值 80%
        self.max_safe_pods = config.get("max_safe_pods",
            int(self.api_rate_limit * self.api_safety_margin / 3))
        # 假设每个 Pod 处理约 3 个并发 LLM 请求

        # 扩缩容参数
        self.scale_up_threshold = config.get("scale_up_threshold", 0.7)
        self.scale_down_threshold = config.get("scale_down_threshold", 0.3)
        self.min_replicas = config.get("min_replicas", 2)
        self.max_replicas = min(
            config.get("max_replicas", 20),
            self.max_safe_pods
        )

    def compute_score(self, metrics: dict) -> float:
        """
        计算当前 Agent 系统的复合扩缩容评分。

        评分范围 0.0~1.0:
          < 0.3 → 可以缩容
          0.3-0.7 → 维持不变
          > 0.7 → 需要扩容

        Parameters
        ----------
        metrics : dict
            {
                "queue_depth": 150,
                "queue_growth_rate": 5.0,  # 每秒增长请求数
                "inflight_llm_requests": 45,
                "p95_latency_ms": 8500,
                "baseline_p95_latency_ms": 2000,
                "error_rate_429": 0.02,
                "active_pods": 10,
                "api_rate_limit_remaining": 20,
            }
        """
        score = 0.0
        reasons = []

        # ── 1. 队列深度评分 ─────────────────────────────────
        queue_depth = metrics.get("queue_depth", 0)
        active_pods = metrics.get("active_pods", self.min_replicas)
        # 标准化: 每个 Pod 的理想并发处理能力约 5 个排队请求
        ideal_queue_per_pod = 5
        expected_queue = active_pods * ideal_queue_per_pod
        queue_ratio = queue_depth / max(expected_queue, 1)
        queue_score = min(queue_ratio / 3.0, 1.0)  # 3x 预期 = 满评分
        score += queue_score * self.queue_depth_weight
        if queue_score > 0.5:
            reasons.append(f"queue_depth_high({queue_depth:.0f}, score={queue_score:.2f})")

        # ── 2. 延迟偏离评分 ─────────────────────────────────
        p95 = metrics.get("p95_latency_ms", 0)
        baseline = metrics.get("baseline_p95_latency_ms", p95)
        if baseline > 0:
            latency_ratio = p95 / baseline
            latency_score = min((latency_ratio - 1.0) / 3.0, 1.0)  # 4x = 满评分
            latency_score = max(latency_score, 0.0)
        else:
            latency_score = 0.0
        score += latency_score * self.latency_weight
        if latency_score > 0.5:
            reasons.append(f"latency_high({latency_ratio:.1f}x baseline, score={latency_score:.2f})")

        # ── 3. 错误率评分（上限）────────────────────────────
        error_rate = metrics.get("error_rate_429", 0)
        # 错误率高 → 扩容无效甚至有害，限制评分
        if error_rate > 0.1:  # 超过 10% 的请求被限流
            score = min(score, 0.5)  # 禁止扩容
            reasons.append(f"error_rate_too_high({error_rate:.2%}, capped_at_0.5)")
        elif error_rate > 0.05:
            score = min(score, 0.7)  # 限制扩容
            error_penalty = (error_rate - 0.05) * 10  # 5-10% 之间线性降分
            score -= error_penalty * self.error_rate_weight
            reasons.append(f"error_rate_elevated({error_rate:.2%}, penalty={error_penalty:.2f})")

        # ── 4. 在途请求评分 ─────────────────────────────────
        inflight = metrics.get("inflight_llm_requests", 0)
        remaining = metrics.get("api_rate_limit_remaining", self.api_rate_limit)
        total_capacity = inflight + remaining
        if total_capacity > 0:
            usage_ratio = inflight / total_capacity
            inflight_score = min(usage_ratio * 1.5, 1.0)  # >66% 使用率 => 高分
        else:
            inflight_score = 0.0
        score += inflight_score * self.inflight_weight
        if inflight_score > 0.6:
            reasons.append(f"api_usage_high({usage_ratio:.1%}, score={inflight_score:.2f})")

        self._last_score = score
        self._last_reasons = reasons
        return score

    def desired_replicas(self, metrics: dict) -> tuple:
        """
        根据当前指标和评分计算目标副本数。

        Returns
        -------
        (desired_replicas, reasoning)
        """
        score = self.compute_score(metrics)
        current = metrics.get("active_pods", self.min_replicas)

        # 检查 API 速率限制作为硬约束
        inflight = metrics.get("inflight_llm_requests", 0)
        remaining = metrics.get("api_rate_limit_remaining", self.api_rate_limit)
        api_headroom = remaining / max(self.api_rate_limit, 1)

        # 硬约束: 如果 API 余量不足，不能扩容
        if api_headroom < 0.2 and score > self.scale_up_threshold:
            # API 只剩 20% 余量，强制冷却
            reasons = self._last_reasons + ["API_RATE_LIMIT_NEARING_CAP, SCALE_UP_BLOCKED"]
            return current, reasons

        # 基本逻辑
        if score > self.scale_up_threshold:
            # 扩容: 线性映射评分到副本数
            target = int(current * (1 + (score - self.scale_up_threshold) * 2))
            target = min(target, self.max_replicas)
            target = max(target, self.min_replicas)
            reasons = self._last_reasons + [f"scale_up: {current} -> {target}"]
            return target, reasons

        elif score < self.scale_down_threshold:
            # 缩容: 缓慢缩容
            target = max(
                int(current * (1 - (self.scale_down_threshold - score) * 1.5)),
                self.min_replicas
            )
            reasons = self._last_reasons + [f"scale_down: {current} -> {target}"]
            return target, reasons

        reasons = self._last_reasons + ["stable"]
        return current, reasons
```

### 4.3 自定义 HPA + Prometheus Adapter

如果不想引入 KEDA，也可以使用 Prometheus Adapter 将自定义指标暴露给 HPA：

```yaml
# prometheus-adapter 配置: 将 Agent 指标映射到 K8s 自定义指标
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    # 队列深度 → 自定义指标
    - seriesQuery: agent_queue_depth_total
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: "agent_queue_depth_total"
        as: "agent_queue_depth"
      metricsQuery: |
        sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)

    # 在途 LLM 请求数 → 自定义指标
    - seriesQuery: agent_inflight_llm_requests
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: "agent_inflight_llm_requests"
        as: "inflight_llm"
      metricsQuery: |
        sum(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)

    # 合并多个指标的复合指标
    - seriesQuery: agent_composite_score
      resources:
        overrides:
          namespace:
            resource: namespace
      name:
        matches: "agent_composite_score"
        as: "agent_composite_score"
      metricsQuery: |
        avg(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)
```

对应的 HPA：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: agent-hpa-custom-metrics
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agent-worker
  minReplicas: 2
  maxReplicas: 20
  metrics:
  # 主指标: 队列深度
  - type: Pods
    pods:
      metric:
        name: agent_queue_depth
      target:
        type: AverageValue
        averageValue: "10"     # 每个 Pod 平均队列深度 > 10 时扩容
  # 辅助指标: 在途 LLM 请求数
  - type: Pods
    pods:
      metric:
        name: inflight_llm
      target:
        type: AverageValue
        averageValue: "5"      # 每个 Pod 平均超过 5 个在途请求时扩容
  # 行为配置
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 180
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

### 4.4 KEDA External Scaler: 自定义 Agent 外部 Scaler

当内置 Scaler 无法满足 Agent 特有的扩缩容逻辑时，可以实现 KEDA External Scaler：

```
KEDA External Scaler 协议:

┌──────────────┐     gRPC          ┌──────────────────────┐
│  KEDA        │◄─────────────────►│  External Scaler     │
│  Operator    │  IsActive(ctx)    │  (gRPC Server)       │
│              │  GetMetrics(ctx)  │                      │
│              │  StreamActive(ctx)│  ├── 读取 LLM API    │
└──────────────┘                   │  │   余量 (从 Redis) │
                                   │  ├── 计算复合评分     │
                                   │  ├── 检查速率限制状态  │
                                   │  └── 返回扩缩容建议    │
                                   └──────────────────────┘
```

```python
# external_scaler_server.py -- 使用 gRPC 实现 KEDA External Scaler

import grpc
from concurrent import futures
from typing import Optional, Iterator
import redis.asyncio as redis
import json
import time
import asyncio
import logging

logger = logging.getLogger(__name__)

# 假设已经生成了 keda_external_scaler_pb2 和 keda_external_scaler_pb2_grpc
# 实际部署时需要从 KEDA 仓库编译 proto 文件
# https://github.com/kedacore/keda/blob/main/pkg/scalers/external/grpc/external.proto

class AgentExternalScaler(external_scaler_pb2_grpc.ExternalScalerServicer):
    """
    为 Agent 系统定制的 KEDA External Scaler。

    整合多个 Agent 特有的指标来决定是否需要扩缩容：
    - LLM API 速率限制余量
    - Agent 队列深度（带趋势预测）
    - 会话活跃数（长连接 Agent 会话）
    - Token 消耗速度
    """

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self._scaler_configs = {}

    async def IsActive(
        self,
        request: external_scaler_pb2.ScalerIsActiveRequest,
        context: grpc.aio.ServicerContext
    ) -> external_scaler_pb2.ScalerIsActiveResponse:
        """
        KEDA 定期调用此方法检查是否需要扩容。

        返回值:
          result: True = 需要扩容（指标非零）
          metrics: 当前指标值（用于 KEDA 做决策）
        """
        scaler_config = json.loads(request.scalerConfig)
        min_queue_trigger = int(scaler_config.get("minQueueTrigger", "10"))

        # 从 Redis 读取 Agent 系统的实时指标
        queue_depth = await self._get_queue_depth()
        inflight = await self._get_inflight_count()
        rate_limit_remaining = await self._get_rate_limit_remaining()

        # 主判断: 队列深度 > 阈值
        is_active = queue_depth > min_queue_trigger

        # 辅助判断: API 余量充足时才应该扩容
        # 如果 API 余量已经很低，即使队列有积压也不应该扩容
        if inflight > 0 and rate_limit_remaining < inflight * 0.2:
            logger.warning(
                f"API rate limit nearly exhausted: {rate_limit_remaining} remaining, "
                f"{inflight} in-flight. Blocking scale-up."
            )
            # 返回 active=false 阻止进一步扩容（但保持现有 Pod）
            is_active = False

        return external_scaler_pb2.ScalerIsActiveResponse(
            result=is_active,
            metrics=[
                external_scaler_pb2.MetricValue(
                    name="agent_queue_depth",
                    value=str(queue_depth),
                    type=external_scaler_pb2.MetricType.GAUGE
                ),
                external_scaler_pb2.MetricValue(
                    name="agent_inflight_llm",
                    value=str(inflight),
                    type=external_scaler_pb2.MetricType.GAUGE
                )
            ]
        )

    async def GetMetricSpec(
        self,
        request: external_scaler_pb2.ScalerGetMetricSpecRequest,
        context: grpc.aio.ServicerContext
    ) -> external_scaler_pb2.ScalerGetMetricSpecResponse:
        """
        向 KEDA 描述此 Scaler 提供的指标及其类型。
        KEDA 用这些信息创建内部的 HPA 对象。
        """
        scaler_config = json.loads(request.scalerConfig)

        # 注册两个指标：
        # 1. 队列深度（目标值）
        # 2. 复合评分（目标值）
        return external_scaler_pb2.ScalerGetMetricSpecResponse(
            metricSpecs=[
                external_scaler_pb2.MetricSpec(
                    type="External",
                    metricName="agent_queue_depth",
                    targetSize=int(scaler_config.get("queueTargetPerPod", "10"))
                ),
                external_scaler_pb2.MetricSpec(
                    type="External",
                    metricName="agent_composite_score",
                    targetSize=70   # 复合评分 > 70 触发扩容
                )
            ]
        )

    async def GetMetrics(
        self,
        request: external_scaler_pb2.ScalerGetMetricsRequest,
        context: grpc.aio.ServicerContext
    ) -> external_scaler_pb2.ScalerGetMetricsResponse:
        """
        KEDA 调用此方法获取当前指标值（在每个 pollingInterval）。
        """
        queue_depth = await self._get_queue_depth()
        inflight = await self._get_inflight_count()
        error_rate = await self._get_error_rate()
        latency_p95 = await self._get_p95_latency()

        # 计算复合评分
        composite_score = self._compute_composite_score(
            queue_depth=queue_depth,
            inflight=inflight,
            error_rate=error_rate,
            latency_p95=latency_p95
        )

        return external_scaler_pb2.ScalerGetMetricsResponse(
            metricValues=[
                external_scaler_pb2.MetricValue(
                    name="agent_queue_depth",
                    value=str(queue_depth),
                    type=external_scaler_pb2.MetricType.GAUGE
                ),
                external_scaler_pb2.MetricValue(
                    name="agent_composite_score",
                    value=str(composite_score),
                    type=external_scaler_pb2.MetricType.GAUGE
                )
            ]
        )

    def _compute_composite_score(
        self,
        queue_depth: int,
        inflight: int,
        error_rate: float,
        latency_p95: float
    ) -> int:
        """
        内部复合评分计算。

        分数范围 0-100:
        0-30:  低负载，可缩容
        30-70: 正常负载
        70-100: 高负载，需要扩容
        """
        queue_score = min(queue_depth / 20, 50)     # 队列最多贡献 50 分
        inflight_score = min(inflight / 5, 20)       # 在途请求最多贡献 20 分
        latency_score = min(latency_p95 / 1000, 20)  # 延迟最多贡献 20 分
        error_penalty = error_rate * 100             # 错误率作为扣分项

        raw_score = queue_score + inflight_score + latency_score - error_penalty
        return max(min(int(raw_score), 100), 0)

    async def _get_queue_depth(self) -> int:
        """从 Redis 读取请求队列深度"""
        val = await self.redis.get("agent:queue_depth")
        return int(val) if val else 0

    async def _get_inflight_count(self) -> int:
        """从 Redis 读取当前在途 LLM 请求数"""
        val = await self.redis.get("agent:inflight_llm")
        return int(val) if val else 0

    async def _get_rate_limit_remaining(self) -> int:
        """从 Redis 读取 LLM API 剩余配额"""
        val = await self.redis.get("agent:api_remaining")
        return int(val) if val else 100

    async def _get_error_rate(self) -> float:
        """从 Redis 读取最近 5 分钟的 429 错误率"""
        val = await self.redis.get("agent:error_rate_429")
        return float(val) if val else 0.0

    async def _get_p95_latency(self) -> float:
        """从 Redis 读取 P95 延迟（毫秒）"""
        val = await self.redis.get("agent:latency_p95_ms")
        return float(val) if val else 0.0


# gRPC 服务器启动
async def serve(redis_client: redis.Redis, port: int = 9000):
    server = grpc.aio.server(futures.ThreadPoolExecutor(max_workers=10))
    external_scaler_pb2_grpc.add_ExternalScalerServicer_to_server(
        AgentExternalScaler(redis_client), server
    )
    server.add_insecure_port(f"[::]:{port}")
    logger.info(f"External Scaler server starting on port {port}")
    await server.start()
    await server.wait_for_termination()
```

对应的 KEDA ScaledObject 配置（使用 External Scaler）：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: agent-external-scaler
spec:
  scaleTargetRef:
    name: agent-worker
  minReplicaCount: 2
  maxReplicaCount: 20
  pollingInterval: 10
  cooldownPeriod: 90
  triggers:
  - type: external
    metadata:
      scalerAddress: agent-external-scaler-svc:9000
      minQueueTrigger: "10"
      queueTargetPerPod: "10"
      # 以下为自定义元数据，会传递给 External Scaler
      tls: "false"
    metricType: AverageValue
```

### 4.5 预测性扩缩容

基于历史模式的预扩缩容可以将扩容延迟降低 60% 以上。核心思路：在负载到达之前准备好资源。

```
预测性扩缩容的工作流程:

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  历史指标采集     │    │  预测模型        │    │  预先扩缩容      │
│                 │    │                 │    │                 │
│  - 过去 30 天   │───►│  - Prophet      │───►│  - 提前 5 分钟  │
│    队列深度      │    │  - ARIMA        │    │    调整 replica  │
│  - 请求量时间序列 │    │  - LSTM         │    │  - 与事件日历   │
│  - 事件日历     │    │  - 简单周期平均   │    │    结合         │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                      │
                                                      ▼
                                           ┌────────────────────┐
                                           │  KEDA ScaledObject │
                                           │  minReplicas 已更新  │
                                           └────────────────────┘
```

```yaml
# 预测性扩缩容的具体实现：CronJob 更新 KEDA ScaledObject 的 minReplicas
# 使用简单的"过去 7 天同一时段"的平均值预测

apiVersion: batch/v1
kind: CronJob
metadata:
  name: predictive-scale-update
spec:
  schedule: "*/15 * * * *"  # 每 15 分钟运行一次
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaler-updater
          containers:
          - name: predictor
            image: agent-predictor:latest
            env:
            - name: KEDA_SCALED_OBJECT
              value: "agent-worker-scaler"
            - name: NAMESPACE
              value: "production"
            - name: PREDICTION_WINDOW_MINUTES
              value: "15"
            - name: MIN_REPLICAS
              value: "2"
            - name: MAX_REPLICAS
              value: "30"
            command:
            - python
            - /app/predict_and_scale.py
```

```python
# predict_and_scale.py — 预测下一时段所需副本数

import os
import json
from datetime import datetime, timedelta
import requests
import numpy as np

PROMETHEUS_URL = os.environ.get("PROMETHEUS_URL", "http://prometheus:9090")
SCALED_OBJECT = os.environ["KEDA_SCALED_OBJECT"]
NAMESPACE = os.environ["NAMESPACE"]
PREDICTION_WINDOW = int(os.environ.get("PREDICTION_WINDOW_MINUTES", "15"))
MIN_REPLICAS = int(os.environ.get("MIN_REPLICAS", "2"))
MAX_REPLICAS = int(os.environ.get("MAX_REPLICAS", "30"))


def query_prometheus(query: str, time: datetime) -> float:
    """查询 Prometheus 在指定时间点的指标值"""
    url = f"{PROMETHEUS_URL}/api/v1/query"
    params = {
        "query": query,
        "time": time.timestamp()
    }
    resp = requests.get(url, params=params, timeout=10)
    data = resp.json()
    if data["status"] != "success" or not data["data"]["result"]:
        return 0.0
    try:
        return float(data["data"]["result"][0]["value"][1])
    except (IndexError, ValueError, KeyError):
        return 0.0


def predict_required_pods() -> int:
    """
    预测下一时段所需的 Pod 数量。

    方法: 取过去 7 天同一时段（前后 15 分钟窗口）的最大队列深度平均值，
    然后估算所需 Pod 数。

    简单周期预测: 过去 7 天同一 15 分钟窗口的请求量平均值。
    对于 Agent 系统，这种方法已经足够好，因为流量模式通常具有
    明显的时间周期性（工作日 vs 周末、白天 vs 夜晚）。
    """
    now = datetime.now()
    target_time = now + timedelta(minutes=PREDICTION_WINDOW)

    # 查询过去 7 天同一时间段的队列深度
    queue_depths = []
    for days_ago in range(1, 8):
        t = target_time - timedelta(days=days_ago)
        # 取该时间点前后 7 分钟的数据（共 15 分钟窗口）
        query = (
            f'avg_over_time('
            f'  agent_queue_depth_total'
            f'  [{PREDICTION_WINDOW}m]'
            f')'
        )
        depth = query_prometheus(query, t)
        if depth > 0:
            queue_depths.append(depth)

    if not queue_depths:
        return MIN_REPLICAS

    # 使用 P75 而不是平均值来预留余量
    predicted_queue = np.percentile(queue_depths, 75)

    # 估算所需 Pod 数
    # 假设每个 Pod 能处理 5 个队列请求（含等待 + 处理）
    capacity_per_pod = 5

    # 对 Agent 来说，还需要考虑请求的复杂度分布
    # 增加 30% 的安全系数
    safety_factor = 1.3

    required = int(np.ceil(predicted_queue / capacity_per_pod * safety_factor))
    return max(MIN_REPLICAS, min(required, MAX_REPLICAS))


def update_scaled_object_min_replicas(pods: int):
    """通过 kubectl patch 命令更新 KEDA ScaledObject 的 minReplicas"""
    patch = json.dumps({
        "spec": {
            "minReplicaCount": pods
        }
    })
    cmd = [
        "kubectl", "patch", "scaledobject",
        SCALED_OBJECT,
        "-n", NAMESPACE,
        "--type=merge",
        f"--patch={patch}"
    ]
    import subprocess
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode == 0:
        print(f"Successfully updated minReplicas to {pods}")
    else:
        print(f"Failed to update: {result.stderr}")


if __name__ == "__main__":
    predicted = predict_required_pods()
    current_query = (
        'sum(avg_over_time(agent_queue_depth_total[5m]))'
    )
    current_depth = query_prometheus(current_query, datetime.now())
    print(f"Predicted queue depth: estimated {predicted} Pods needed")
    print(f"Current queue depth: {current_depth:.0f}")

    update_scaled_object_min_replicas(predicted)
```

### 4.6 Agent 指标的 Prometheus Exporter

为了让扩缩容体系正常工作，Agent 应用本身需要暴露正确的指标：

```python
# agent_metrics.py — Agent 应用的 Prometheus 指标定义

import time
import threading
from prometheus_client import (
    Counter, Gauge, Histogram, generate_latest,
    start_http_server, REGISTRY
)
from contextlib import contextmanager
from typing import Optional

# ─── Agent 请求指标 ──────────────────────────────────────────

agent_requests_total = Counter(
    "agent_requests_total",
    "Agent 请求总数",
    ["namespace", "agent_type", "status"]
)

agent_request_duration_seconds = Histogram(
    "agent_request_duration_seconds",
    "Agent 请求处理耗时（秒）",
    ["namespace", "agent_type"],
    buckets=(0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0, 60.0, 120.0)
    # Agent 请求的桶分布更宽（最长可达数分钟）
)

# ─── 扩缩容核心指标 ──────────────────────────────────────────

agent_queue_depth = Gauge(
    "agent_queue_depth",
    "当前请求队列深度",
    ["namespace", "queue_name"]
)

agent_inflight_llm_requests = Gauge(
    "agent_inflight_llm_requests",
    "当前在途的 LLM API 请求数",
    ["namespace", "llm_provider", "model"]
)

agent_inflight_tool_calls = Gauge(
    "agent_inflight_tool_calls",
    "当前在途的工具调用数",
    ["namespace", "tool_name"]
)

# ─── LLM API 速率限制指标 ───────────────────────────────────

agent_api_rate_limit_remaining = Gauge(
    "agent_api_rate_limit_remaining",
    "LLM API 剩余并发配额",
    ["namespace", "llm_provider"]
)

agent_api_rate_limit_total = Gauge(
    "agent_api_rate_limit_total",
    "LLM API 总并发配额",
    ["namespace", "llm_provider"]
)

agent_api_429_rate = Gauge(
    "agent_api_429_rate",
    "过去 5 分钟的 429 错误率",
    ["namespace", "llm_provider"]
)

# ─── 延迟指标 ────────────────────────────────────────────────

agent_llm_latency_seconds = Histogram(
    "agent_llm_latency_seconds",
    "LLM API 调用延迟（秒）",
    ["namespace", "llm_provider", "model"],
    buckets=(0.5, 1.0, 2.0, 3.0, 5.0, 10.0, 15.0, 30.0)
)

agent_tool_latency_seconds = Histogram(
    "agent_tool_latency_seconds",
    "工具调用延迟（秒）",
    ["namespace", "tool_name"],
    buckets=(0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0)
)

# ─── 会话指标 ────────────────────────────────────────────────

agent_active_sessions = Gauge(
    "agent_active_sessions",
    "当前活跃的 Agent 会话数",
    ["namespace"]
)

agent_session_duration_seconds = Histogram(
    "agent_session_duration_seconds",
    "Agent 会话持续时间（秒）",
    ["namespace", "agent_type"],
    buckets=(10, 30, 60, 120, 300, 600, 1800, 3600)
)

# ─── 复合评分指标（扩缩容决策用） ────────────────────────────

agent_scale_composite_score = Gauge(
    "agent_scale_composite_score",
    "扩缩容复合评分 (0-100)",
    ["namespace"]
)

agent_scale_decision = Gauge(
    "agent_scale_decision",
    "扩缩容决策: -1=缩容, 0=维持, 1=扩容",
    ["namespace"]
)


class AgentMetricsCollector:
    """
    Agent 应用指标收集器。

    在每个 Agent 请求的生命周期中更新关键指标，
    供扩缩容系统 (KEDA/Prometheus Adapter) 使用。
    """

    def __init__(self, namespace: str = "production"):
        self.namespace = namespace
        self._start_http_server()

    def _start_http_server(self, port: int = 8000):
        """启动 Prometheus metrics HTTP 端点"""
        start_http_server(port)
        print(f"Agent metrics endpoint started on :{port}/metrics")

    @contextmanager
    def track_request(self, agent_type: str):
        """跟踪一个 Agent 请求的完整生命周期"""
        start = time.monotonic()
        try:
            yield
            agent_requests_total.labels(
                namespace=self.namespace,
                agent_type=agent_type,
                status="success"
            ).inc()
        except Exception:
            agent_requests_total.labels(
                namespace=self.namespace,
                agent_type=agent_type,
                status="error"
            ).inc()
            raise
        finally:
            duration = time.monotonic() - start
            agent_request_duration_seconds.labels(
                namespace=self.namespace,
                agent_type=agent_type
            ).observe(duration)

    @contextmanager
    def track_llm_call(self, provider: str, model: str):
        """跟踪一次 LLM API 调用"""
        agent_inflight_llm_requests.labels(
            namespace=self.namespace,
            llm_provider=provider,
            model=model
        ).inc()
        start = time.monotonic()
        try:
            yield
        finally:
            duration = time.monotonic() - start
            agent_llm_latency_seconds.labels(
                namespace=self.namespace,
                llm_provider=provider,
                model=model
            ).observe(duration)
            agent_inflight_llm_requests.labels(
                namespace=self.namespace,
                llm_provider=provider,
                model=model
            ).dec()

    def set_queue_depth(self, depth: int, queue_name: str = "default"):
        """设置当前队列深度"""
        agent_queue_depth.labels(
            namespace=self.namespace,
            queue_name=queue_name
        ).set(depth)

    def set_api_rate_limit(self, remaining: int, total: int, provider: str):
        """更新 LLM API 速率限制信息"""
        agent_api_rate_limit_remaining.labels(
            namespace=self.namespace,
            llm_provider=provider
        ).set(remaining)
        agent_api_rate_limit_total.labels(
            namespace=self.namespace,
            llm_provider=provider
        ).set(total)

    def report_composite_score(
        self, score: int, decision: int
    ):
        """
        上报扩缩容复合评分和决策。

        Parameters
        ----------
        score : int
            0-100 的复合评分
        decision : int
            -1 = 缩容, 0 = 维持, 1 = 扩容
        """
        agent_scale_composite_score.labels(
            namespace=self.namespace
        ).set(score)
        agent_scale_decision.labels(
            namespace=self.namespace
        ).set(decision)
```

---

## 5. 扩缩容策略对比

### 5.1 横向对比

| 维度 | K8s HPA (CPU) | K8s HPA (自定义指标) | KEDA | 预测性扩缩容 | Serverless |
|------|---------------|---------------------|------|------------|------------|
| **适用 Agent 负载** | 无效 | 中等 | **最推荐** | 中等（配合 KEDA） | 适合短期任务 |
| **指标来源** | CPU/Memory 内置 | Prometheus Adapter | 30+ Scaler 类型 | 历史时间序列 | 请求数自动 |
| **扩容响应速度** | 慢 (60-120s) | 中 (30-60s) | **快 (15-30s)** | **最快 (提前触发)** | 即时 |
| **缩容速度** | 慢（默认 5 分钟） | 可配置 | 可配置 + cooldown | 平缓 | 快（但可能有冷启动） |
| **速率限制感知** | 否 | 需要自建 | 支持（通过 External Scaler） | 部分 | 由平台管理 |
| **有状态会话支持** | 差 | 差 | **好（Graceful Shutdown）** | 中 | **差（无状态要求）** |
| **配置复杂度** | 低 | 中 | 中 | 高 | 最低 |
| **运维复杂度** | 低 | 中 | 中-高 | 高 | 低 |
| **成本效率** | 低 | 中 | **高（scale-to-zero）** | 高 | **最高（按调用计费）** |

### 5.2 HPA vs VPA vs Event-Driven

Agent 系统在不同场景下可能需要不同的伸缩方式：

```
伸缩方式选择:

水平扩缩 (HPA + KEDA):
  ┌─ 适合: 无状态或外部化状态的 Agent Worker
  ├─ 原理: 增加/减少 Pod 数量
  ├─ 优点: 灵活性高，不受单机资源限制
  ├─ 缺点: 冷启动延迟，有状态迁移复杂
  └─ Agent 场景: 请求处理级 Worker Pod

垂直扩缩 (VPA):
  ┌─ 适合: 资源需求波动的单实例
  ├─ 原理: 调整 Pod 的 CPU/Memory 请求和限制
  ├─ 优点: 无需重新调度 Pod，状态不移失
  ├─ 缺点: 受单机上限限制，Pod 需要重启
  └─ Agent 场景: 加载大模型的推理 Pod、Embedding 服务

事件驱动 (KEDA):
  ┌─ 适合: 消息队列驱动的 Agent 处理
  ├─ 原理: 根据队列积压等事件信号伸缩
  ├─ 优点: 粒度更细，响应更快
  ├─ 缺点: 需要事件源
  └─ Agent 场景: 异步 Agent 处理管道
```

### 5.3 四种模式的选择矩阵

```
                     请求类型
                ┌──────────┴──────────┐
               同步（在线）              异步（离线）
                │                        │
        ┌───────┴───────┐       ┌───────┴───────┐
      有状态             无状态   有状态           无状态
        │                │        │                │
        ▼                ▼        ▼                ▼
  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │ Predictive │  │ KEDA +    │  │ KEDA +     │  │ KEDA      │
  │ + KEDA     │  │ Prometheus│  │ RabbitMQ   │  │ scale-to- │
  │ + Session  │  │ HPA       │  │ + State    │  │ zero      │
  │ Affinity   │  │           │  │ Snapshot   │  │           │
  └────────────┘  └────────────┘  └────────────┘  └────────────┘
  │               │               │               │
  │ Agent Chat    │ Stateless     │ Async Agent   │ 批量推理    │
  │ 多轮对话       │ Tool Call    │ Pipeline      │ 数据处理    │
  │ 高一致性要求   │ 低状态要求    │ 中等一致性    │ 无状态      │
  └───────────────┘              └───────────────┘             │
```

---

## 6. 能力边界与限制

### 6.1 扩容的上限：LLM API 速率限制

这是 Agent 扩缩容最根本的硬约束：

```
一个典型的生产系统 API 配额的约束：

OpenAI API 示例（Tier 5 账户）:
  GPT-4o:        10,000 RPM (每分钟请求数)
                  500,000 TPM (每分钟 Token 数)
                  并行: 根据 TPM 动态调整

每个 Agent Pod 的典型消耗:
  每个请求:  ~500-2000 input tokens + ~200-500 output tokens
  每个请求耗时: 1-5s
  每个 Pod 并发: 10 个同时进行中的 LLM 调用
  一个 Pod 的吞吐量: 10 请求 / 平均 3s = ~3.3 QPS

配额约束计算:
  RPM 约束: 10,000 RPM / (10 Pods × 10 并行 / Pod × 60s / 3s 每请求)
           = 10,000 / (10 × 10 × 20) = ...实际上 RPM 约束更宽松

  TPM 约束 (通常更紧):
  假设每请求 ~2000 tokens
  每个 Pod 每分钟消耗: 10 并行 × (60/3) × 2000 = 400,000 TPM
  理论上 500,000 TPM 配额: 只够 1.25 个 Pod!
  
  但这是所有 Pod 共享一个 API Key 的情况。
```
---

**扩容的硬天花板:**

```
硬件天花板:
  max_pods = floor(API 配额 / 单 Pod 消耗) × 安全系数

  示例 (单 Key 下):
    配额: 500,000 TPM
    单 Pod: ~400,000 TPM (满负荷)
    理论上: 只能跑 1 个 Pod...显然不合理 (因为不可能所有 Pod 同时满负荷)

  更现实的计算:
    P50 利用率: 40%
    有效单 Pod 消耗: 400,000 × 0.4 = 160,000 TPM
    Max Pods: 500,000 / 160,000 = 3 个 Pod
    安全系数 80%: 最大 2-3 个 Pod

  突破方案: 多个 API Key 轮转
    Key Pool: 5 个 Key
    每个 Key 的配额: 500,000 TPM
    总配额: 2,500,000 TPM
    Max Pods: 2,500,000 / 160,000 = ~15 个 Pod
```

### 6.2 缩容的下限：活跃会话数

Agent 系统不能无限制缩容，有两个关键限制：

```
1. 活跃会话数下限:
   min_replicas >= ceil(active_sessions / max_sessions_per_pod)
   
   假设每个 Pod 最多处理 50 个并发会话:
   - 当前活跃会话: 80
   - min_replicas >= ceil(80 / 50) = 2
   
   强行缩到 1 个 Pod 会导致:
   - 80 个会话挤在一个 Pod
   - 资源争抢，延迟飙升
   - Pod OOMKilled 的风险

2. 最小冗余:
   即使没有活跃会话，也需要至少 1-2 个 Pod 做"守夜人"
   - 接收新连接
   - 维持与外部系统的连接（WebSocket、消息队列）
   - 保证首次请求的响应速度
```

### 6.3 滞后效应与振荡

```
经典 Agent 扩缩容振荡场景:

┌──────┬─────────────────────────────────────────────────────┐
│ 时间 │  事件                                                │
├──────┼─────────────────────────────────────────────────────┤
│  T0  │  队列深度: 120 → 触发扩容 (KEDA 检测到)               │
│  T1  │  KEDA 更新 HPA target → HPA 开始扩容 (增加 4 个 Pod) │
│  T2  │  新 Pod 开始启动 (拉取镜像、加载依赖)                   │
│  T3  │  (15s 后) 队列深度: 200 → 再次触发扩容                 │
│  T4  │  HPA 再次调整，又要加 4 个 Pod                         │
│  T5  │  第一批 4 个 Pod 上线 → 队列开始被消费                  │
│  T6  │  第二批 4 个 Pod 上线 → 队列快速下降                    │
│  T7  │  队列深度: 10 → 触发缩容                               │
│  T8  │  HPA 开始缩容 (但稳定窗口 180s 阻止立即缩容)             │
│  T9  │  (180s 后) 队列深度: 5 → 缩容 2 个 Pod                │
│ T10  │  又有突发流量 → 队列再次堆积 → 重新扩容                │
└──────┴─────────────────────────────────────────────────────┘

问题分析:
  - KEDA 检测 → HPA 执行之间有 15-30s 延迟
  - Pod 启动有 20-25s 冷启动延迟
  - 从检测到 Pod 就绪: 30-60s
  - 在这 30-60s 内如果队列继续增长, 会触发第二轮扩容
  - 当两轮扩容的 Pod 同时上线，会导致过度扩容
  - 过度扩容后缩容，遇到下一波突发流量又扩容

这称为"锯齿效应" — 副本数在短时间内反复上下波动。
```

解决方案：

```yaml
# KEDA 配置中防止振荡的关键参数
spec:
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 180   # 缩容稳定窗口：180s 内不重复缩容
          policies:
          - type: Percent
            value: 20                       # 每次最多缩 20%
            periodSeconds: 60
          selectPolicy: Min                 # 缩容选最保守的策略
        scaleUp:
          stabilizationWindowSeconds: 30    # 扩容稳定窗口：30s
          policies:
          - type: Percent
            value: 100                      # 可翻倍
            periodSeconds: 30
          - type: Pods
            value: 2                        # 或者每次最多加 2 个
            periodSeconds: 30               # (注意：这里设 30s 而不是 15s)
          selectPolicy: Max
  pollingInterval: 20                       # 每 20s 检查一次（不是 5s）
  cooldownPeriod: 120                       # 指标低于阈值后等 120s 再缩容
```

### 6.4 冷启动时间

| 冷启动项 | 时间范围 | 对 Agent 的影响 |
|---------|---------|----------------|
| 容器镜像拉取 | 2-60s | 最慢的环节（尤其在大镜像时） |
| Python 运行时加载 | 2-5s | 固定开销 |
| 框架初始化 (LangChain, LlamaIndex) | 3-8s | 依赖插件数量 |
| Embedding 模型加载 | 2-30s | 如果 Pod 需要 GPU 加载 |
| Redis/DB 连接池建立 | 1-3s | 主要是网络握手 |
| LLM API 连接预热 | 1-3s | DNS + TLS，首次调用额外延迟 |
| 向量数据库连接 | 1-2s | 如果有 |
| **总冷启动** | **10-80s** | **严重影响快速扩容的时效性** |

---

## 7. 工程优化方向

### 7.1 复合评分优化：多信号加权公式

将多个不完美的信号组合成一个更可靠的扩缩容信号。

```
复合评分公式 (production v2):

score = W_q × S_queue + W_l × S_latency + W_e × S_error + W_i × S_inflight

各分量:
  S_queue = min(queue_depth / (pods × target_per_pod), 1.0)
  S_latency = clamp((p95 / baseline) - 1, 0, 1)        # 延迟偏离基线程度
  S_error = exp(rate_429 * 10) - 1                       # 指数放大的错误率
  S_inflight = min(inflight / (api_limit × safety), 1.0) # API 配额使用率

权重参考:
  W_q = 0.35  # 队列深度 (主要信号)
  W_l = 0.25  # 延迟偏离 (验证信号)
  W_e = 0.15  # 错误率 (减速信号)
  W_i = 0.25  # 在途请求 (前置信号)

减速规则:
  if S_error > 0.3: score *= 0.5    # 错误率 > 3% 时减半扩容
  if S_error > 0.8: score = 0       # 错误率 > 8% 时禁止扩容
  if api_headroom < 0.2: score = min(score, 0.3)  # API 余量不足时限制
```

### 7.2 优雅缩容：连接排空

当 Pod 被缩容时，必须优雅地处理正在进行的 Agent 会话：

```
缩容流程:

1. K8s 标记 Pod 为 "Terminating"
   ├── 从 Service Endpoint 中移除
   └── 不再接收新请求

2. PreStop Hook 触发 (kubectl exec 或 HTTP 端点)
   ├── 通知 Agent 进程: 准备关闭
   ├── 停止接受新任务
   └── 记录当前正在处理的会话列表

3. 正在进行的 LLM 调用的处理:
   ├── 等待进行中的 LLM 调用完成 (最多等待 graceful_timeout)
   ├── 将已经返回的 LLM 结果保存到 Redis (状态快照)
   └── 标记会话状态为 "可迁移"

4. 在途工具调用的处理:
   ├── 等待工具调用完成 (最多 10s)
   └── 保存中间结果到共享存储

5. 会话迁移:
   ├── 将会话状态序列化到 Redis
   └── 通知负载均衡器更新会话映射

6. Pod 进程退出
   └── K8s 发送 SIGTERM → SIGKILL (如果超出 terminationGracePeriodSeconds)
```

```yaml
# Deployment 配置中的优雅缩容设置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-worker
spec:
  replicas: 5
  template:
    spec:
      terminationGracePeriodSeconds: 120     # 给予 120s 完成善后工作
      containers:
      - name: agent
        image: agent-app:latest
        ports:
        - containerPort: 8000
          name: http
        - containerPort: 8001
          name: metrics
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # 通知 Agent 进程开始优雅关闭
                curl -X POST http://localhost:8000/internal/shutdown \
                  --max-time 5 || true
                # 等待连接排空
                sleep 30
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 20
      # 使用 PodDisruptionBudget 减少同时终止的 Pod 数
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: agent-worker-pdb
spec:
  minAvailable: 2          # 至少保持 2 个 Pod 可用
  selector:
    matchLabels:
      app: agent-worker
```

### 7.3 预热 Pod 池

为了减少冷启动对扩容速度的影响，可以维护一个"预热 Pod 池"：

```
预热 Pod 池架构:

                      ┌──────────────────────┐
                      │  Agent 预热池          │
                      │  (Pre-warmed Pool)    │
                      │                       │
                      │  ┌────┐ ┌────┐ ┌────┐  │
                      │  │Pod1│ │Pod2│ │Pod3│  │  ← 已完成初始化
                      │  │    │ │    │ │    │  │    但未加入 Service
                      │  └────┘ └────┘ └────┘  │    (空闲等待)
                      └──────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │ 队列深度触发        │ 预测性触发           │ 定时补充
          ▼                    ▼                      ▼
   ┌──────────────────────────────────────────────────┐
   │                  Service Mesh                    │
   │  (预热 Pod 通过 readinessProbe 切换 → 接收流量)    │
   └──────────────────────────────────────────────────┘
```

实现方式：使用 KEDA 的 `scaleDown` 行为保留预热 Pod：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: agent-worker-scaler
spec:
  minReplicaCount: 3           # 至少 3 个 Pod（含 1 个额外预热）
  maxReplicaCount: 20
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300   # 5 分钟稳定窗口
          policies:
          - type: Percent
            value: 10                       # 每次最多缩 10%
            periodSeconds: 60
        scaleUp:
          policies:
          - type: Percent
            value: 100
            periodSeconds: 15               # 快速扩容
```

或者使用专用的预热管理器：

```python
# warm_pool_manager.py — 预热 Pod 池管理器

class WarmPoolManager:
    """
    维护一个"预热 Pod 池"，在流量到达之前就准备好资源。

    预热池的 Pod 已经完成:
    - 镜像拉取
    - Python 框架加载
    - 连接池建立 (Redis, DB, Vector DB)
    - LLM API TLS 握手

    这些 Pod 在 readinessProbe 返回 "not ready" 之前不会接收流量。
    当需要扩容时，只需将 Pod 标记为 "ready" 即可。
    """

    def __init__(self, k8s_client, deployment_name: str, namespace: str):
        self.client = k8s_client
        self.deployment_name = deployment_name
        self.namespace = namespace
        self.warm_target = 2  # 目标预热数

    async def ensure_warm_pool(self):
        """
        确保预热 Pod 池中有足够数量的空闲 Pod。

        策略:
        - 当活跃 Pod 利用率 > 70% 时，增加预热 Pod
        - 当活跃 Pod 利用率 < 30% 时，减少预热 Pod
        - 始终保留 min_warm 个预热 Pod
        """
        current_replicas = await self._get_current_replicas()
        active_utilization = await self._get_active_utilization()

        # 根据利用率动态调整预热池大小
        if active_utilization > 0.7:
            # 高负载：增加预热池
            warm_needed = min(
                self.warm_target + 1,
                current_replicas * 0.3  # 预热池不超过当前副本的 30%
            )
        elif active_utilization < 0.3:
            # 低负载：减少预热池
            warm_needed = max(1, self.warm_target - 1)
        else:
            warm_needed = self.warm_target

        await self._update_warm_pool(warm_needed)
```

### 7.4 会话迁移

当缩容需要移除一个处理中会话的 Pod 时，需要将会话迁移到其他 Pod：

```
Agent 会话迁移协议:

┌──────┐                  ┌──────┐                  ┌──────┐
│Pod-A │                  │Redis │                  │Pod-B │
│(缩容)│                  │(共享)│                  │(接收)│
└──┬───┘                  └──┬───┘                  └──┬───┘
   │                         │                         │
   │ 1. 收到 SIGTERM         │                         │
   │───→ 进入 PreStop        │                         │
   │                         │                         │
   │ 2. 序列化会话状态        │                         │
   │───→ session:123 → JSON──►                         │
   │    ├─ 对话历史            │                         │
   │    ├─ 当前进行中的 LLM    │                         │
   │    ├─ 已获取的上下文      │                         │
   │    └─ 会话级变量         │                         │
   │                         │                         │
   │ 3. 检查是否有进行中的     │                         │
   │    LLM 调用              │                         │
   │───→ 等待完成（最多 30s）   │                         │
   │───→ 保存结果到 Redis     │                         │
   │                         │                         │
   │ 4. 通知负载均衡器       │                         │
   │───→ session:123 → Pod-B  │                         │
   │                         │                         │
   │                   5. 接收会话 → 解序列化            │
   │                         │◄────── 唤醒              │
   │                         │                         │
   │ 6. 进程退出              │                         │
   │───→ exit(0)             │                         │
   │                         │                         │
   │                   7. 继续处理会话                    │
   │                         │───→ 发送到客户端          │
```

### 7.5 混沌工程：测试自动扩缩容

在生产环境中，扩缩容逻辑的正确性需要用混沌工程验证：

```
Agent 扩缩容混沌实验矩阵:

实验 1: 流量突增          实验 2: 速率限制跌落
┌──────────────────┐      ┌──────────────────┐
│ 模拟: QPS 5→200  │      │ 模拟: API 配额   │
│ 预期: KEDA 扩容   │      │      500→50 TPM  │
│ 验证:            │      │ 预期: 停止扩容    │
│ - 扩容是否及时    │      │ 验证:            │
│ - 是否触发 429    │      │ - 是否触发降级    │
│ - 恢复时间        │      │ - 错误率是否可控  │
└──────────────────┘      └──────────────────┘

实验 3: Pod 故障         实验 4: 消息队列积压
┌──────────────────┐      ┌──────────────────┐
│ 模拟: 随机杀 Pod  │      │ 模拟: 消息堆积    │
│ 预期: K8s 重新调度 │      │      10K/分钟    │
│ 验证:            │      │ 预期: 触发扩容    │
│ - 会话是否中断    │      │ 验证:            │
│ - 恢复时间        │      │ - 消费速率跟上    │
│ - 是否触发级联    │      │ - 是否有消息丢失  │
└──────────────────┘      └──────────────────┘
```

可以使用 `chaos-mesh` 或 `litmus` 实现这些实验：

```yaml
# chaos-mesh 实验: 测试流量突增下的扩缩容行为
apiVersion: chaos-mesh.org/v1alpha1
kind: HTTPChaos
metadata:
  name: agent-scale-test
spec:
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: agent-worker
  target: Request
  port: 8000
  abort: false
  delay: "2000ms"     # 给所有 Agent 请求增加 2s 延迟
  duration: "5m"      # 持续 5 分钟
  scheduler:
    cron: "@every 30m"  # 每 30 分钟执行一次
---
# 同时观察扩缩容行为
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: agent-pod-stress
spec:
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: agent-worker
  stressors:
    cpu:
      workers: 1
      load: 80         # CPU 80% → 测试 CPU 压力下是否误缩容
      duration: "3m"
```

---

## 8. 适用场景判断

### 8.1 阶段选择矩阵：何时使用什么策略

```
Agent 系统扩缩容策略演进路径:

                  用户规模
                   │
  Phase 1          │ 1-100 用户
  ───────          │
  手动/固定        │ 策略: 固定 2-4 个 Pod
                   │ 状态: 单体应用，不需要扩缩容
                   │ 关注: 正确性和功能，不是弹性
                   │
                   ▼
  Phase 2          │ 100-10K 用户
  ───────          │
  HPA + KEDA       │ 策略: KEDA + Prometheus Scaler
                   │ 指标: 队列深度 + 在途请求
                   │ 缩放: 2-20 Pods
                   │ 关注: 可靠性和成本平衡
                   │
                   ▼
  Phase 3          │ 10K+ 用户
  ───────          │
  预测 + 混合      │ 策略: Predictive + KEDA + External Scaler
                   │ 指标: 复合评分 + 时间序列预测
                   │ 缩放: 5-100 Pods
                   │ 关注: 成本效率 + 用户体验
                   │
                   ▼
  Phase 4          │ 100K+ 用户
  ───────          │
  全球部署 + 精细  │ 策略: 多区域预测 + API Key 池负载均衡
  优化              │ 指标: 每个区域独立扩缩容
                   │ 缩放: 10-500 Pods per region
                   │ 关注: 全球延迟优化 + Token 成本
```

### 8.2 成本分析

```
自动扩缩容的投入产出分析:

投入 (每月):
  KEDA Operator 运维:  ~0.5 人天
  自定义指标开发维护:   ~2 人天
  Prometheus 基础设施:  ~$100-200 (或自建)
  External Scaler 开发: ~5 人天 (一次性)
  ─────────────────────────────────
  总投入: ~$500-1500/月 + 初始 5 人天

产出 (按 10 个 Pod 平均):
  未使用自动扩缩容:
    - 固定 10 Pods × 24h × 30d = 7200 Pod-小时
    - 实际需要: 平均 4 Pods (峰值 10, 低峰 2)
    - 浪费: 7200 - 3000 ≈ 4200 Pod-小时
    - 成本浪费: ~$800-1200/月

  使用 KEDA 自动扩缩容:
    - 动态 2-10 Pods，平均 4.5 Pods
    - Pod-小时: 4.5 × 24 × 30 = 3240
    - 节约: ~$600-900/月

  节省比例: ~50-70%

何时值得配置自动扩缩容？
  固定成本 < 节省成本 + 运维成本
  对于 Agent 系统，通常月成本 > $1000 时自动扩缩容即可回本
```

### 8.3 决策指南

```
是否需要自动扩缩容？

Q1: 你的 Agent 系统是否 24/7 运行？
  ├── 是 → 需要 (非高峰时段可大幅缩容降本)
  └── 否 → 不需要 (按需启动即可)

Q2: 日用户量是否超过 1000 或峰值 QPS > 10？
  ├── 是 → 需要 (手动管理开始产生压力)
  └── 否 → 不需要 (单实例的并发能力就够了)

Q3: 流量模式是否有突发峰值？
  ├── 是 → 需要 (突发流量不可预测)
  └── 否 → Cron 定时扩缩容即可 (模式固定)

Q4: 是否使用多个 LLM API Key 或共享 API Key？
  ├── 多 Key → 需要 External Scaler (每 Key 配额度管理)
  └── 单 Key → KEDA 即可 (只需要管理一个速率限制)

Q5: Agent 会话是长时间(>5分钟)还是短会话？
  ├── 长时间 → 需要优雅缩容 + 会话迁移
  └── 短时间 → 标准缩容即可 (会话自然结束)
```

---

## 9. 参考架构

### 9.1 多信号 KEDA + Prometheus 完整配置

```
端到端扩缩容信号流:

                         ┌─────────────────────────┐
                         │     Agent Worker Pods    │
                         │                         │
  ┌──────────┐           │  ┌──────────────────┐   │
  │ 用户请求  │──────────►│  │  Agent 应用       │   │
  └──────────┘           │  │                   │   │
                         │  │  /metrics 端点     │───┼───► Prometheus
                         │  │  Prometheus 指标   │   │     ↑
                         │  └──────────────────┘   │     │
                         │         │               │     │
                         │         │ Redis 读写     │     │
                         │         ▼               │     │
                         │  ┌──────────────────┐   │     │
                         │  │  Redis           │   │     │
                         │  │  (队列/状态/缓存)  │   │     │
                         │  └──────────────────┘   │     │
                         └─────────────────────────┘     │
                                                         │
  ┌──────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────┐
│  Prometheus Server                                       │
│  ┌────────────────────────────────────────────────────┐  │
│  │  agent_queue_depth (Gauge)                         │  │
│  │  agent_inflight_llm_requests (Gauge)               │  │
│  │  agent_api_rate_limit_remaining (Gauge)            │  │
│  │  agent_api_429_rate (Gauge)                        │  │
│  │  agent_scale_composite_score (Gauge)               │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│  KEDA Operator                                           │
│  ┌────────────────────────────────────────────────────┐  │
│  │  ScaledObject                                      │  │
│  │  ├── trigger: prometheus (队列深度)                  │  │
│  │  ├── trigger: prometheus (延迟比)                    │  │
│  │  ├── trigger: prometheus (错误率 → 减速)              │  │
│  │  ├── cooldownPeriod: 120                            │  │
│  │  └── pollingInterval: 15                            │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│  Kubernetes HPA (由 KEDA 管理)                             │
│  minReplicas: 2   maxReplicas: 20                         │
│  Behavior: 渐进扩容、慢速缩容                                │
└──────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│  Agent Worker Deployment                                  │
│  replica: 动态 (2-20)                                     │
│  每个 Pod:                                                │
│  ├── FastAPI / aiohttp server                            │
│  ├── LLM API 连接池                                       │
│  ├── Redis 连接                                          │
│  ├── Vector DB 连接                                      │
│  ├── /metrics (Prometheus)                               │
│  └── /health (Readiness/Liveness)                        │
└──────────────────────────────────────────────────────────┘
```

### 9.2 完整 ScaledObject 配置

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: agent-production-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agent-worker
  minReplicaCount: 2
  maxReplicaCount: 20
  pollingInterval: 15
  cooldownPeriod: 120
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 180
          policies:
          - type: Percent
            value: 10
            periodSeconds: 60
          selectPolicy: Min
        scaleUp:
          stabilizationWindowSeconds: 30
          policies:
          - type: Percent
            value: 100
            periodSeconds: 30
          - type: Pods
            value: 2
            periodSeconds: 20
          selectPolicy: Max
  triggers:
  # 主触发器: 队列深度信号
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring:9090
      metricName: agent_queue_depth
      query: |
        sum(avg_over_time(agent_queue_depth{namespace="production"}[2m]))
      threshold: "150.0"                                  # 总队列深度 > 150
      activationThreshold: "30.0"

  # 辅助触发器: 延迟比 (P95 延迟 / 基线延迟 > 2)
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring:9090
      metricName: agent_latency_ratio
      query: |
        (
          histogram_quantile(0.95,
            sum(rate(agent_request_duration_seconds_bucket{namespace="production"}[5m]))
            by (le)
          )
          /
          (
            avg_over_time(
              histogram_quantile(0.95,
                sum(rate(agent_request_duration_seconds_bucket{namespace="production"}[5m]))
                by (le)
              )[7d:5m]
            )
            + 0.01
          )
        )
      threshold: "2.0"                                    # P95 延迟 > 基线 2 倍
      activationThreshold: "1.5"

  # 辅助触发器: API 配额使用率 > 60%
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring:9090
      metricName: agent_api_usage_ratio
      query: |
        agent_inflight_llm_requests{namespace="production"}
        /
        agent_api_rate_limit_total{namespace="production"}
      threshold: "0.6"

  # 减速触发器: 错误率 > 5% 时触发（使用 negative 逻辑）
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring:9090
      metricName: agent_error_rate_high
      query: |
        rate(agent_request_errors_total{namespace="production", error_type="429"}[5m])
        /
        rate(agent_requests_total{namespace="production"}[5m])
      threshold: "0.05"                                    # 429 错误率阈值
```

### 9.3 预测 + 反应式混合扩缩容架构

```yaml
# 双 ScaledObject 架构: 预测设置 minReplicas, 反应式动态调整

---
# ScaledObject 1: 反应式扩缩容（基于实时指标）
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: agent-reactive-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agent-worker
  minReplicaCount: 2
  maxReplicaCount: 30
  pollingInterval: 15
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring:9090
      metricName: agent_queue_depth
      query: sum(agent_queue_depth{namespace="production"})
      threshold: "100"
---
# CronJob: 预测性更新 minReplicas
apiVersion: batch/v1
kind: CronJob
metadata:
  name: predictive-min-update
  namespace: production
spec:
  schedule: "*/30 * * * *"  # 每 30 分钟运行一次
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: keda-updater
          containers:
          - name: predictor
            image: python:3.11-slim
            command:
            - python
            - -c
            - |
              import json, subprocess, datetime, statistics

              # 从 Prometheus 获取最近 7 天同一时段的数据
              # 计算 P75 队列深度 -> 估算需要的 min Pods
              # 然后 patch KEDA ScaledObject 的 minReplicaCount

              # 简化版: 基于星期几和小时决定 min replicas
              now = datetime.datetime.now()
              hour = now.hour

              # 业务高峰期 (9-12, 14-18): 需要 5 个 Pod
              # 非高峰期: 需要 2 个 Pod
              if (9 <= hour < 12) or (14 <= hour < 18):
                  min_pods = 5
              elif (8 <= hour < 22):
                  min_pods = 3
              else:
                  min_pods = 2

              # 周末减半
              if now.weekday() >= 5:
                  min_pods = max(2, min_pods // 2)

              patch = json.dumps({
                  "spec": {"minReplicaCount": min_pods}
              })
              result = subprocess.run([
                  "kubectl", "patch", "scaledobject",
                  "agent-reactive-scaler",
                  "-n", "production",
                  "--type=merge",
                  f"--patch={patch}"
              ], capture_output=True, text=True)
              print(f"Set minReplicas to {min_pods}: {result.stdout}")
```

---

## 总结

### Agent 自动扩缩容的核心原则

1. **CPU 是错误信号** -- Agent Pod 在等待 LLM API 返回时 CPU 几乎为 0，真正的瓶颈在 API 速率限制和队列深度

2. **组合信号优于单一信号** -- 队列深度（领先指标）+ 延迟偏离（同步指标）+ 错误率（滞后指标）= 可靠决策

3. **扩容必须感知速率限制** -- 无视 API 配额的扩容会触发全局 429，导致所有实例同时降级

4. **缩容必须优雅** -- Agent 会话不能在缩容时被粗暴终止，需要连接排空和状态迁移

5. **预测 + 反应式优于纯反应式** -- 预测解决冷启动延迟，反应式处理突发流量，两者结合最可靠

6. **冷启动是硬约束** -- 20-80s 的 Pod 启动延迟意味着扩容策略必须前馈（预测）而不是反馈（滞后指标）

7. **KEDA 是 Agent 扩缩容的最佳起点** -- 内置 30+ Scaler，支持 External Scaler，scale-to-zero 节省成本

### 常见陷阱

```
❌ 用 CPU > 80% 做 Agent 扩缩容
    → 队列堆积但 CPU=5%，HPA 反向缩容 → 雪崩

❌ 扩容不检查 API 余量
    → 新 Pod 涌入 LLM 请求 → API 限流 → 所有 Pod 重试 → 系统瘫痪

❌ 缩容不等待正在处理的 Agent
    → 用户请求被中断 → 重试后上下文丢失 → 糟糕的用户体验

❌ 只用一个 API Key
    → 扩容天花板取决于单 Key 配额 → 扩容到 3 个 Pod 就撞墙

❌ pollingInterval < 10s
    → Prometheus 查询压力大 + KEDA 反复判断 + 无意义振荡

❌ 没有稳定窗口 (stabilizationWindow)
    → 副本数剧烈振荡 → Pod 频繁创建/销毁 → 资源碎片 + 成本浪费
```
