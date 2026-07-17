# 13.1.2 k8s-deploy — Kubernetes 部署

**Kubernetes 是生产级 Agent 系统的事实标准编排平台。** 当 Agent 从"单个服务"演变为"多个协作组件"时（Agent 服务 + 向量数据库 + 缓存 + 工具执行沙箱），K8s 提供了服务发现、自动伸缩、滚动更新、自愈等关键能力。但 Agent 系统在 K8s 上的部署比传统 Web 服务复杂得多——因为有状态管理、LLM API 延迟波动、以及非确定性错误处理等独特挑战。

## 背景与问题

### 之前是怎么做的？
Docker Compose 编排多服务 Agent 系统，所有容器运行在同一台机器上。当流量增长时，手动复制容器实例，使用 Nginx 做负载均衡。

### 这样做的问题
- 单机瓶颈：一台机器的资源有限
- 无法自动伸缩：高峰时需要人工介入
- 服务发现：容器 IP 变化后需要手动更新配置
- 滚动更新：停机部署影响用户体验
- 自愈缺失：容器崩溃后不会自动重启

## Agent 在 K8s 上的架构

```
                     ┌──────────────────────────────────────────┐
                     │          Ingress (nginx-controller)       │
                     │   /api/* → agent-service                  │
                     │   /ws/*  → agent-service (WebSocket)      │
                     │   /admin/* → admin-service                │
                     └──────────────────┬───────────────────────┘
                                        │
                     ┌──────────────────▼───────────────────────┐
                     │         Agent Service (ClusterIP)         │
                     │    Label: app=agent, tier=core             │
                     │    SessionAffinity: ClientIP              │
                     └──────────────────┬───────────────────────┘
                                        │
          ┌─────────────────────────────┼─────────────────────────────┐
          │                             │                             │
          ▼                             ▼                             ▼
┌──────────────────┐          ┌──────────────────┐          ┌──────────────────┐
│  Agent Pod (core) │          │   Agent Pod (mem) │          │  Agent Pod (tools)│
│  ┌──────────────┐│          │  ┌──────────────┐│          │  ┌──────────────┐│
│  │ LLM Client   ││          │  │ Memory Mgr   ││          │  │ Tool Executor ││
│  │ Prompt Mgr   ││          │  │ Vector Client ││          │  │ Code Sandbox  ││
│  │ State Mgr    ││          │  └──────────────┘│          │  └──────────────┘│
│  └──────────────┘│          └──────────────────┘          └──────────────────┘
└──────────────────┘          └──────────────────┘          └──────────────────┘
          │                             │                             │
          ▼                             ▼                             ▼
┌──────────────────┐          ┌──────────────────┐          ┌──────────────────┐
│  Redis Service    │          │  PG Service       │          │  Qdrant Service  │
│  (StatefulSet)    │          │  (StatefulSet)    │          │  (StatefulSet)   │
│  + Sentinel       │          │  + WAL-G 备份    │          │  + 持久卷        │
└──────────────────┘          └──────────────────┘          └──────────────────┘
```

## 核心部署配置

### Deployment

```yaml
# agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-core
  namespace: agent-prod
  labels:
    app: agent
    tier: core
spec:
  replicas: 3
  # 滚动更新配置
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0        # 零停机部署
      maxSurge: 1              # 允许额外启动一个 Pod
  selector:
    matchLabels:
      app: agent
      tier: core
  template:
    metadata:
      labels:
        app: agent
        tier: core
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      # 优先调度到不同节点
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: agent
      containers:
        - name: agent
          image: registry.example.com/agent:1.2.3
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: metrics
          env:
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: llm-keys
                  key: openai-api-key
            - name: ANTHROPIC_API_KEY
              valueFrom:
                secretKeyRef:
                  name: llm-keys
                  key: anthropic-api-key
            - name: REDIS_URL
              value: "redis://redis-svc:6379"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: connection-string
            - name: LOG_LEVEL
              value: "INFO"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector:4318"
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          # Agent 特有的就绪探针——不仅要检查端口，还要检查 LLM 连接
          readinessProbe:
            httpGet:
              path: /readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 60
            timeoutSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: agent-config
              mountPath: /app/config
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: agent-config
          configMap:
            name: agent-config
        - name: tmp
          emptyDir: {}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
```

### 服务与 Session Affinity

Agent 的对话状态导致需要会话亲和性：

```yaml
# agent-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: agent-svc
  namespace: agent-prod
  labels:
    app: agent
spec:
  selector:
    app: agent
    tier: core
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: metrics
      port: 9090
      targetPort: 9090
  # Agent 需要 Session Affinity 保持对话状态
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 1800  # 30 分钟
```

### HPA (水平自动伸缩)

Agent 的伸缩指标与传统服务不同：

```yaml
# agent-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: agent-core-hpa
  namespace: agent-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agent-core
  minReplicas: 3
  maxReplicas: 20
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # 避免频繁缩容
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    # 自定义指标：Agent 特有
    - type: Pods
      pods:
        metric:
          name: agent_active_sessions
        target:
          type: AverageValue
          averageValue: 50
```

### ConfigMap 配置

```yaml
# agent-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: agent-config
  namespace: agent-prod
data:
  agent.yaml: |
    agent:
      name: production-agent
      max_steps: 20
      max_tokens_per_step: 4000
      context_limit: 128000
      memory:
        short_term:
          type: sliding_window
          window_size: 20
        long_term:
          type: vector_store
          url: http://qdrant-svc:6333
          collection: agent_memory
      tools:
        timeout: 30s
        max_parallel: 3
        sandbox:
          enabled: true
          image: sandbox:latest
      streaming:
        enabled: true
        buffer_size: 100
      cost_control:
        max_daily_tokens: 10000000
        alert_threshold: 0.8
```

### Secrets

```yaml
# agent-secrets.yaml (实际使用 sealed-secrets 或 External Secrets Operator)
apiVersion: v1
kind: Secret
metadata:
  name: llm-keys
  namespace: agent-prod
type: Opaque
stringData:
  openai-api-key: "sk-..."
  anthropic-api-key: "sk-ant-..."
---
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: agent-prod
type: Opaque
stringData:
  connection-string: "postgresql://agent:password@postgres-svc:5432/agent_db"
```

## Agent 有状态组件部署

虽然推荐"无状态化"Agent，但有些组件天生有状态：

### Redis (记忆缓存)

```yaml
# redis-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: agent-prod
spec:
  serviceName: redis-svc
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          command:
            - redis-server
            - "--appendonly yes"
            - "--save 900 1"
            - "--save 300 10"
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-data
              mountPath: /data
          resources:
            requests:
              memory: "512Mi"
              cpu: "200m"
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
```

## Agent 部署的 K8s 最佳实践

### 1. Pod 中断预算 (PDB)

Agent 会话可能持续数分钟，需要保证不中断：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: agent-core-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: agent
      tier: core
```

### 2. 网络策略

最小化网络访问：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agent-network-policy
spec:
  podSelector:
    matchLabels:
      app: agent
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: api-gateway
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - port: 6379
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
    # LLM API 外部访问
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - port: 443
```

### 3. 资源配额命名空间

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: agent-quota
  namespace: agent-prod
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 40Gi
    limits.cpu: "20"
    limits.memory: 80Gi
    persistentvolumeclaims: 10
```

### 4. 监控集成

```yaml
# agent-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: agent-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: agent
  namespaceSelector:
    matchNames:
      - agent-prod
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

## 常见问题与解决方案

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Agent 响应慢 | LLM API 延迟高 | 增加 Agent 副本数、超时调大 |
| Pod OOMKilled | 上下文窗口膨胀 | 限制 Token 预算、启用历史压缩 |
| 对话状态丢失 | Pod 重启路由到不同实例 | SessionAffinity + 外部状态存储 |
| 滚动更新中断 | 更新时存量请求被断开 | maxUnavailable=0，优雅关闭 |
| LLM API 限流 | 多 Pod 并发调用 | 外部限流器、请求排队 |
| 启动慢 | 镜像大/依赖多 | 多阶段构建、预热镜像 |

## 与 Serverless 的比较

| 维度 | K8s | Serverless |
|------|-----|------------|
| 冷启动 | 无（Pod 常驻） | 有（几百 ms~几秒） |
| 成本模型 | 固定 + 弹性 | 纯按量 |
| 运维复杂度 | 高 | 低 |
| 状态管理 | 好（PVC/本地） | 差（纯外部） |
| GPU 支持 | 好 | 差 |
| 网络延迟 | 低 | 中 |
| 适合的 Agent 类型 | 在线、实时、复杂 | 批处理、事件驱动 |

## 能力边界

**K8s 部署适合：**
- ✅ 日均万级请求以上的生产 Agent
- ✅ 需要高可用（99.9%+）的 Agent 服务
- ✅ 多组件协作的复杂 Agent 系统
- ✅ 需要 GPU 推理的 Agent

**K8s 部署的挑战：**
- ❌ 运维团队需要 K8s 专业能力
- ❌ 小规模（日均 < 1000）时性价比低
- ❌ Agent 的无状态化改造需要额外工作
- ❌ 调试 Agent 行为在分布式环境中更复杂
