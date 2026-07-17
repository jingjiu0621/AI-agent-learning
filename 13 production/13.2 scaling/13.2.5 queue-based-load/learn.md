# 13.2.5 队列负载平滑 (Queue-based Load Smoothing)

**Agent 系统的负载特征与传统 Web 服务有本质不同：请求是高度突发性的，且失败会在系统内部自我放大。** 没有队列缓冲的生产级 Agent 系统，就像没有蓄水池的自来水厂——水源来多少，管道就必须送多少，水压稍有不稳整个系统就会崩溃。

```
传统 Web 负载 vs Agent 负载
──────────────────────────
传统 Web:  ▁▃▅▇▅▃▁▃▅▇▅▃▁    (平滑、可预测)
Agent:     ▁▁▁████▁▁▁██████▁  (突发、长尾)

Agent 负载的核心特征:
├── 请求到达时间: 用户消息集中 (上班、午休、晚间)
├── 处理时间: LLM API P50 1s, P95 8s, P99 20s+  (10-20x 波动)
├── 嵌套调用: 一个请求可能触发 3-10 步 Agent 链
└── 自放大: 失败 -> 重试 -> 更多请求 -> 更大压力
```

## 1. 背景与问题

### 1.1 Agent 工作负载为什么比传统 Web 服务更突发

传统 Web 服务的负载突发主要由"用户行为"驱动。Agent 系统的负载突发由三个独立维度叠加而成，突发程度呈乘法放大。

```
突发叠加模型:
─────────────
用户行为突发          ×  LLM 延迟波动          ×  任务链长度
(同一时刻 N 人触发)     (P50 1s vs P99 20s)      (1 个请求 -> N 步)

            ┌─────────────────────────────────────────┐
            │  突发放大系数 = 用户突发 × 延迟波动 × 链长  │
            │  示例: 10 用户同时触发                     │
            │          × 某步延迟 20s (P99 事件)         │
            │          × 平均链长 5 步                    │
            │          = 某个时刻可能堆叠 100+ 并发调用     │
            └─────────────────────────────────────────┘
```

**延迟波动是 Agent 特有的问题。** 一个简单查找任务可能 0.5s 完成，同一个模型处理复杂推理时可能需要 15s+。在传统 Web 服务中，请求处理时间通常服从较窄的正态分布；在 Agent 系统中，请求时间的分布是幂律分布——尾部极长。

```
Agent 请求延迟分布 (示意):
─────────────────────
概率
↑
│    ██
│    ██████
│    ████████████
│    ████████████████           ██
│    ██████████████████████     ████          ██    ██
└─────────────────────────────────────────────────────→ 延迟
     P50: 1s    P75: 3s    P90: 8s  P95: 15s P99: 25s

这意味着: 100 个请求中，有 5 个请求的耗时是"正常"请求的 15-25 倍。
这 5 个请求在队列中阻塞了后续 95 个请求。
```

### 1.2 双重问题：实例过载 + API 限流同时发生

Agent 系统的瓶颈不是单一的。一个突发请求潮会同时冲击两个薄弱环节：

```
        突发请求 (50 QPS)
              │
              ▼
     ┌────────────────┐
     │                │      ┌──────────────────┐
     │  Agent 实例池   │─────▶│  每个实例:        │
     │  (3 个 Pod)     │     │  - CPU 正常 (I/O) │
     │                │     │  - 连接数激增      │
     └────────────────┘     │  - 内存增长 (OOM)  │
              │              └──────────────────┘
              ▼
     ┌────────────────┐
     │  LLM API 网关   │─────▶  429 Rate Limit!
     │  (每分钟 1000   │
     │   请求限制)      │
     └────────────────┘
              │
              ▼
     ┌────────────────┐
     │  工具/数据库     │─────▶  连接池耗尽!
     │  (连接池 50)    │
     └────────────────┘

双重打击:
1. Agent 实例: 活跃连接超过 asyncio 事件循环处理能力 → 超时
2. API 限流: LLM 返回 429 → 请求失败 → Agent 重试 → 更多请求
```

### 1.3 自造 DDoS：Agent 重试放大

这是 Agent 系统独有的危险模式。传统服务的重试放大通常是 1:1（一个失败重试一次）。Agent 系统的重试放大会形成正反馈回路：

```
起始: 10 个用户同时触发 Agent

第 1 步: Agent 发送 10 个 LLM 请求 → 全部成功 → 返回需要工具调用
第 2 步: 工具执行 (数据库查询) → 50% 超时 (连接池最大)
第 3 步: 工具失败 → Agent 决定重试 → 又发 10 个请求
         ┌─────────────────────────────────────────┐
         │ 此刻系统中已有:                          │
         │  - 原始 10 个请求仍在等待                  │
         │  - 新注入 10 个重试请求                    │
         │  - 每个等待的请求上下文占用内存              │
         └─────────────────────────────────────────┘
第 4 步: LLM API 收到 20 个请求 → 部分返回 429
第 5 步: 429 → Agent 重试 (指数退避) → 但请求仍在堆积

结果: 10 个用户 → 50+ 次 LLM API 调用 → 系统雪崩
```

**雪崩三阶段：**

```
阶段 1: 正常负载
  QPS: 10  │███░░░░░░░░░░░░░░░░░░│  系统正常
  延迟: 2s  │███░░░░░░░░░░░░░░░░░░│

阶段 2: 突发到达 + 重试放大
  QPS: 40  │████████████░░░░░░░░░░│  队列开始堆积
  延迟: 8s  │████████████░░░░░░░░░░│  用户开始感知延迟

阶段 3: 雪崩
  QPS: 80+ │██████████████████████│  队列满，请求被拒
  延迟: 30s │██████████████████████│  大量超时，体验崩溃
```

### 1.4 链式调用放大效应

每个用户请求不是"一个"请求——它是一个由 LLM 调用、工具执行、记忆检索组成的多步骤链。传统 Web 中 1 个用户请求 = 1 个后端请求。Agent 中 1 个用户请求 = N 个后端请求。

```
一个典型 Agent 请求的生命周期:
─────────────────────────────

用户输入: "分析上季度的销售数据并生成报告"

步骤                                        LLM 调用    工具调用
────                                        ───────    ──────
1. 意图理解: "user wants sales analysis"      1          -
2. 数据查询: SQL query 数据库                   -         1
3. 结果分析: "summarize these numbers"         1          -
4. 可视化: "generate chart data"               -         1
5. 报告撰写: "write executive summary"         1          -
6. 格式转换: "export to PDF"                   -         1
                                                 ────     ────
                        总计:                    3次       3次

1 个用户请求 → 6 次子调用 → 突发放大 6x

如果 10 个用户同时请求 → 60 次子调用 → 但峰值可能更高
(因为子步骤之间有等待和重叠，瞬时并发可能达到 15-20)
```

## 2. 之前是怎么做的

在队列机制成为 Agent 系统标配之前，业界尝试过多种"直接处理"模式——它们都共享同一个根本缺陷：将请求到达和请求处理强耦合在一起。

### 2.1 同步请求处理（无队列模式）

最简单的架构：请求到达后直接分配处理线程/协程，没有中间缓冲层。

```
同步处理架构:
─────────────

用户 1 ─────▶┌──────────┐─────▶ LLM API
用户 2 ─────▶│ Agent    │─────▶ LLM API
用户 3 ─────▶│ 直接处理   │─────▶ LLM API
用户 4 ─────▶│ (无队列)  │─────▶ LLM API
用户 5 ─────▶│          │─────▶ LLM API
              └──────────┘
              ↑ 请求直接穿透到 LLM

问题:
├── 无缓冲能力: 10 QPS 突发直接变成 10 个 LLM 并发调用
├── 无失败隔离: 一个慢请求阻塞整个处理管道
├── 无优先级: 简单查询和复杂分析同等待遇
└── 无背压: Agent 实例不知道 LLM API 是否过载
```

```python
# 反模式示例: 直接同步处理
class NaiveAgentServer:
    """没有队列的 Agent 服务器——高负载下必然崩溃"""
    
    async def handle_request(self, user_request: dict) -> dict:
        # 请求直接进入处理
        # 如果同时来 100 个请求，这里就有 100 个并发 LLM 调用
        result = await self.agent_executor.run(user_request["message"])
        return {"status": "ok", "result": result}
    
    # 问题: 没有任何控制机制
    # - LLM API 返回 429 怎么办？→ 直接失败
    # - LLM API 变慢怎么办？→ 请求堆积在 asyncio 事件循环
    # - 优先级高的请求怎么办？→ 和低优先级请求一起排队（无差别）
```

### 2.2 线程池 + 有界信号量

改进版：限制并发处理数，但超过限制的请求直接被拒绝。

```
有界线程池:
───────────

请求到达 ──▶┌──────────────────┐   如果池满 → 503 Rejected
            │ 有界信号量 (10)   │
            └────────┬─────────┘
                     ▼
            ┌──────────────────┐
            │ Worker Pool (10) │──▶ LLM API
            │ Worker Pool (10) │──▶ LLM API
            │ ...              │
            └──────────────────┘

优点: 限制了最大并发，保护实例不被完全压垮
缺点: 请求被拒绝而不是延迟处理 → "负载削减"而非"负载平滑"
```

```python
import asyncio
from typing import Optional

class SemaphoreProtectedServer:
    """使用信号量限制并发，但溢出时直接拒绝"""
    
    def __init__(self, max_concurrency: int = 10):
        self.semaphore = asyncio.Semaphore(max_concurrency)
        self.agent_executor = AgentExecutor()
    
    async def handle_request(self, user_request: dict) -> dict:
        # 尝试获取信号量
        acquired = await self.semaphore.acquire()
        if not acquired:
            # 直接拒绝——连排队的机会都不给
            return {"status": "error", "code": 503, 
                    "message": "系统繁忙，请稍后重试"}
        
        try:
            result = await self.agent_executor.run(user_request["message"])
            return {"status": "ok", "result": result}
        finally:
            self.semaphore.release()
    
    # 问题:
    # 1. 负载尖峰时，大量用户收到 503，体验差
    # 2. 没有区分请求优先级，VIP 用户和免费用户同样被拒
    # 3. 信号量释放后瞬间涌入大量请求，形成"发球机效应"
```

### 2.3 Nginx/HAProxy 连接限制

在网关层限制连接数是最简单粗暴的方式：

```
Nginx 连接限制:
───────────────

                      ┌──────────────┐
用户 ──▶ Nginx ──▶    │ limit_conn   │──▶ Agent 实例
用户 ──▶ Nginx ──▶    │ limit_req    │──▶ Agent 实例
用户 ──▶ Nginx ──▶    │ 超出: 503    │──▶ Agent 实例
                      └──────────────┘

问题:
├── Nginx 不理解 Agent 协议——它能看到 HTTP 连接，但看不到"这个请求是简单还是复杂"
├── 无法基于请求内容做优先级：简单查询（1s 完成）和复杂分析（30s 完成）同等对待
├── 连接数 ≠ 负载: 2 个复杂 Agent 请求可能比 20 个简单查询压力更大
└── 无法提供背压: Nginx 不知道 LLM API 的速率限制状态
```

### 2.4 传统方案的共同缺陷

```
方法               缺陷类型              根本问题
────               ──────               ────────
同步无队列          过载崩溃              请求到达 = 请求处理
有界信号量          直接拒绝(503)         负载削减而非平滑
Nginx 限流         盲目拒绝              不理解 Agent 语义
Hystrix 熔断       二值状态(开/关)       只有熔断没有缓冲

核心教训:
请求到达率 ≠ 系统处理能力
└── 需要中间层解耦: 队列 = 压力缓冲器 + 速率转换器 + 优先级仲裁器
```

## 3. 核心矛盾

队列引入了一个根本的张力：缓冲越久越平滑，但用户等待越久。

### 3.1 LLM API 速率限制 vs Agent 响应时间 SLA

```
核心矛盾图示:
───────────────

LLM API 要求:                     用户期望:
┌──────────────────────┐          ┌──────────────────────┐
│ 每分钟最多 1000 请求  │          │ 请求必须在 5s 内响应   │
│ 超过返回 429          │    VS    │ 否则用户流失            │
│ 速率必须平滑（无尖峰） │          │ 交互式体验要求实时       │
└──────────────────────┘          └──────────────────────┘
         │                                │
         └──────────┬─────────────────────┘
                    ▼
          ┌────────────────────────┐
          │ 队列的设计困局           │
          │                        │
          │ 队列太短 → 无法平滑突发  │
          │          → LLM 429 错误 │
          │                        │
          │ 队列太长 → 吸收突发能力强 │
          │          → 用户等待更久  │
          └────────────────────────┘
```

**定量分析：**

```
假设:
- LLM API 速率限制: 1000 RPM (每分钟请求)
- Agent 平均处理时间: 3s (含 LLM 调用 + 工具执行)  
- 用户 SLA: P95 响应时间 < 10s
- 突发流量: 正常 10 QPS → 突发 50 QPS

计算:
  没有队列: 50 QPS 突发 → 每分钟 3000 请求 → 远超 1000 RPM → 大量 429
  有队列:
  　队列长度 L, 处理速率 R = 1000/60 ≈ 16.7 QPS
  　突发持续 T 秒, 突发速率 B = 50 QPS
  　队列积压 = (B - R) × T = (50 - 16.7) × T = 33.3 × T
  　
  　如果 T = 10s: 积压 333 个请求
  　等待时间 = 333 / 16.7 ≈ 20s → 超过 SLA 10s ❌
  
  　需要更快的消费速率或更短的突发持续时间:
  　如果 R = 30 QPS (多个 API Key 聚合): 积压 200 个，等待 6.7s ✅
```

### 3.2 队列长度的黄金分割

队列太短和太长都会导致问题，关键是在用户体验和系统稳定性之间找到平衡。

```
队列长度决策框架:
─────────────────

queue_length = target_wait_time × processing_rate

其中:
- target_wait_time: 用户愿意等待的最大排队时间 (通常 2-5s)
- processing_rate: 系统的 LLM API 处理速率 (受速率限制约束)

示例:
  LLM API 限制: 1000 RPM ≈ 16.7 QPS
  目标等待时间: 5s
  理想队列长度: 5 × 16.7 ≈ 83 个请求
  
  但! 这只能吸收 5s 的突发。
  如果突发是 50 QPS 持续 30s → 积压 (50-16.7)×30 = 999 个请求
  等待时间 = 999/16.7 ≈ 60s → 远超 SLA

解决方案: 不是加长队列，而是:
├── 1. 动态增加消费能力 (弹性扩展 Agent 实例 + 多个 API Key)
├── 2. 优先级排队 (VIP 请求插队，普通请求排队)
├── 3. 降级处理 (复杂请求降级为简单模式)
└── 4. 拒绝策略 (告知用户 "当前繁忙，预计等待 X 秒")
```

### 3.3 优先级反转：简单请求被复杂链阻塞

这是 Agent 队列中最容易被忽视的问题。

```
优先级反转:
───────────

队列中的请求:
┌──────────────────────────────────────┐
│ 位置 1: [复杂请求] 分析财报 + 生成报告  │ ← 预计耗时 30s
│ 位置 2: [复杂请求] 多轮对话 + 工具调用  │ ← 预计耗时 20s
│ 位置 3: [简单请求] "现在几点？"         │ ← 预计耗时 0.5s
│ 位置 4: [简单请求] "1+1=?"            │ ← 预计耗时 0.5s
│ 位置 5: [复杂请求] 写一篇 5000 字文章   │ ← 预计耗时 60s
└──────────────────────────────────────┘

如果 FIFO 处理:
- 简单请求等待 50s 才被处理 (30s + 20s)
- 但实际只需要 0.5s！
- 用户体验: "为什么问个时间要等 50 秒？"

这就是优先级反转: 
短请求不可见地被长请求阻塞
```

**更隐蔽的优先级反转：多级队列嵌套**

```
Agent 系统内部存在多级队列:
─────────────────────────

外部请求队列          Agent 处理          LLM API 调用队列
┌──────────┐         ┌──────────┐       ┌──────────┐
│ 请求 A   │ ──▶     │ Agent A  │ ──▶   │ API 调用  │
│ 请求 B   │         │ Agent B  │       │ API 调用  │
│ 请求 C   │         │ Agent C  │       │ API 调用  │
└──────────┘         └──────────┘       └──────────┘

问题: Agent 在处理请求 A 时，会向 LLM 发起 5 次调用。
      每次调用都在 LLM API 队列中排队。
      请求 B 可能只需要 1 次 LLM 调用。
      
      结果: 请求 B 的 LLM 调用被请求 A 的 5 次调用阻塞。
      即使请求 B 在"外部队列"中优先级更高也无济于事。
```

## 4. 当前主流方案

### 4.1 队列负载均衡模式 (Queue-Based Load Leveling)

这是最核心的模式：通过引入队列解耦请求到达和请求处理。

```
核心架构:
─────────

    生产者 (Producer)                    消费者 (Consumer)
    ┌──────────────┐                   ┌──────────────┐
    │ 用户请求到达   │                   │ Agent 工作实例 │
    │ API Gateway   │                   │ Worker Pool   │
    │ Webhook       │                   │              │
    └──────┬───────┘                   └──────▲────────┘
           │                                  │
           │   ┌──────────────────────┐        │
           └──▶│      队列 (Queue)     │────────┘
               │                      │
               │  • 缓冲突发请求        │
               │  • 解耦生产者和消费者   │
               │  • 速率转换           │
               │  • 失败隔离           │
               └──────────────────────┘

解耦带来的好处:
├── 生产者无需等待消费者: 请求入队后立即返回 "已接受"
├── 消费者独立伸缩: 可以独立增减工作实例
├── 突发吸收: 队列充当压力缓冲器
├── 故障隔离: 消费者故障不影响生产者
└── 速率匹配: 队列将突发流量转换为平滑的消费速率
```

**关键设计决策：队列的位置**

```
选项 A: 网关层队列
  ┌──────┐   ┌──────┐   ┌────────┐   ┌──────────┐
  │ 用户  │──▶│ 队列  │──▶│ 网关   │──▶│ Agent    │
  └──────┘   └──────┘   └────────┘   └──────────┘
  优点: 简单，用户立即得到响应
  缺点: 无法做基于内容的优先级路由

选项 B: 应用层队列
  ┌──────┐   ┌──────┐   ┌────────┐   ┌──────────┐
  │ 用户  │──▶│ 网关  │──▶│ 队列    │──▶│ Agent    │
  └──────┘   └──────┘   └────────┘   └──────────┘
  优点: 可以按请求内容做优先级和路由
  缺点: 用户请求阻塞在网关直到入队完成

选项 C: 双队列
  ┌──────┐   ┌────────┐   ┌────────┐   ┌──────────┐
  │ 用户  │──▶│ 网关队列│──▶│ 网关   │──▶│ 应用队列  │──▶ Agent
  └──────┘   └────────┘   └────────┘   └──────────┘
  优点: 端到端背压，精细控制
  缺点: 复杂，延迟增加
```

### 4.2 优先级队列 (Priority Queueing)

为不同类型的 Agent 请求分配不同的优先级，确保重要或短耗时的请求不被阻塞。

#### 4.2.1 三级优先级模型

```
Agent 请求优先级体系:
─────────────────────

优先级 1 (Critical) — 交互式，实时
├── 用户主动发送的消息 (对话中的最新消息)
├── 支付/交易相关的 Agent 操作
├── 紧急警报处理 (安全相关)
├── SLA 保证的高价值客户请求
└── 特征: 延迟敏感，预计处理时间短

优先级 2 (Normal) — 后台处理
├── 多步 Agent 链 (需要多轮推理)
├── 文档分析/生成任务
├── 非紧急的数据处理请求
├── 异步工作流 (审批、通知)
└── 特征: 可接受几秒到几十秒的延迟

优先级 3 (Batch) — 批量/低优先级
├── 批量数据处理任务
├── 定期报告生成
├── 后台知识库更新
├── 非交互式的预处理
└── 特征: 可接受分钟级延迟，适合后台消费
```

#### 4.2.2 优先级反转预防机制

优先级队列的核心问题是"饥饿"——低优先级请求可能永远得不到处理。

```
解决方案 1: Aging (老化机制)
─────────────────────────

每个请求在队列中等待的时间会提升其"有效优先级":

初始状态:
┌────────────────────────────────────────────────┐
│ Priority 1 │ [Req-A: 0s] [Req-B: 0s]           │
│ Priority 2 │ [Req-C: 0s] [Req-D: 0s]           │
│ Priority 3 │ [Req-E: 0s] [Req-F: 0s]           │
└────────────────────────────────────────────────┘

等待 30 秒后:
┌────────────────────────────────────────────────┐
│ Priority 1 │ [Req-E-aging:30s] [Req-A:30s]     │
│ Priority 2 │ [Req-F-aging:30s] [Req-C:30s]     │
│ Priority 3 │ (空 — Req-E 已被提升)              │
└────────────────────────────────────────────────┘

Aging 策略:
├── 每等待 t 秒，有效优先级 +1 级
├── 示例: P3 请求等待 30s → 按 P2 处理
├── 等待 60s → 按 P1 处理
└── 确保所有请求最终都能被处理
```

```
解决方案 2: Priority Boosting (优先级提升)
─────────────────────────────────────

在特定条件下人为提升请求优先级:

条件 1: 预计处理时间短的请求
  └── "现在几点？" 预计 0.5s → 即使 P3，也提升到 P1
    
条件 2: 等待时间超阈值的请求
  └── 任何请求等待 > 30s → 提升到 P1
    
条件 3: 资源受限的请求
  └── 已经占用了上下文 (Token) 的请求
  └── 已经完成部分步骤的请求 → 提升优先级以尽快释放上下文

条件 4: 特定用户/租户
  └── SLA 保证的高价值客户请求 → 固定 P1
```

#### 4.2.3 多队列架构

```
多优先级队列架构:
─────────────────

请求入队:
                                         ┌─────────────────┐
                                    ┌───▶│ Priority 1 队列   │──▶ 快速 Agent 池
                                    │    │ (交互式查询)     │    (低延迟，小模型)
                                    │    └─────────────────┘
                                    │
          ┌──────────┐      ┌───────┤    ┌─────────────────┐
          │ 分类器    │──────│───────┼───▶│ Priority 2 队列   │──▶ 标准 Agent 池
          │          │      │       │    │ (分析推理)       │    (平衡延迟)
          │ 请求类型  │      │       │    └─────────────────┘
          │ 用户等级  │      │       │
          │ 预计耗时  │      │       │    ┌─────────────────┐
          └──────────┘      │       └───▶│ Priority 3 队列   │──▶ 批量 Agent 池
                            │            │ (批处理)         │    (高吞吐，大模型)
                            │            └─────────────────┘
                            │
                            │            ┌─────────────────┐
                            └───────────▶│ Aging 提升队列    │──▶ 紧急处理
                                         │ (等待超时的请求   │
                                         │  自动升级)       │
                                         └─────────────────┘

消费者配置:
├── P1: 3-5 个专用消费者，高优先级，短超时
├── P2: 10-20 个消费者，标准配置
├── P3: 2-3 个消费者，后台运行
└── Aging: 1-2 个消费者，优先级可抢占
```

### 4.3 Agent 特定的队列策略

纯通用队列理论不足以解决 Agent 系统的独特需求。以下是经过生产验证的 Agent 专用队列策略。

#### 4.3.1 多租户隔离队列

在多租户 Agent 平台中，一个租户的突发流量不应影响其他租户。

```
Per-Tenant Queue 架构:
─────────────────────

                       ┌──────────────────────┐
                       │    全局路由调度器       │
                       │    公平队列算法        │
                       └──┬───┬───┬───┬───┬───┘
                          │   │   │   │   │
         ┌────────────────┘   │   │   │   └──────────────┐
         │             ┌──────┘   │   └──────┐           │
         ▼             ▼          ▼          ▼           ▼
   ┌──────────┐  ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ 租户 A    │  │ 租户 B    │ │ 租户 C    │ │ 租户 D    │
   │ 队列      │  │ 队列      │ │ 队列      │ │ 队列      │
   │ (深度 50) │  │ (深度 20) │ │ (深度 100)│ │ (深度 10) │
   └─────┬────┘  └─────┬────┘ └─────┬────┘ └─────┬────┘
         │             │           │           │
         └─────────────┼───────────┼───────────┘
                       ▼           ▼
                ┌────────────────────────┐
                │     Worker Pool        │
                │     (共享消费者)        │
                └────────────────────────┘

公平调度算法 (Weighted Fair Queueing):
├── 每个租户有最小保证配额
├── 空闲租户的配额可重新分配给忙碌租户
├── 防止"吵闹邻居"问题
└── 支持租户级别优先级 (企业客户 > 个人客户)
```

#### 4.3.2 按模型类型分队列

不同 LLM 模型有完全不同的延迟和成本特征，应该使用独立的队列。

```
Per-Model Queue:
───────────────

请求分类
  │
  ├── "简单翻译/摘要"     ──▶ Haiku 队列 ──▶ Haiku API (快速, 低成本)
  ├── "代码生成/分析"     ──▶ Sonnet 队列 ──▶ Sonnet API (平衡)
  ├── "复杂推理/研究"     ──▶ Opus 队列  ──▶ Opus API (最强大, 最贵)
  └── "专用领域任务"      ──▶ 本地模型队列 ──▶ 私有部署模型

为什么需要分队列:
├── 不同模型有独立的速率限制 (API Key 级别)
├── 不同模型的延迟差异巨大 (Haiku 0.5s vs Opus 10s)
├── 不同模型的成本差异巨大 (10x-30x)
└── 可以独立控制每个模型的并发 (Haiku 高并发, Opus 低并发)
```

#### 4.3.3 按任务类型分队列

```
Per-Task-Type Queue:
────────────────────

请求类型分类:

┌─────────────────────────────────────────────────────────────┐
│ 请求到达 → 分类器 (Classifier)                                │
│                                                              │
│ 分类依据:                                                     │
│ ├── 意图: lookup / analysis / generate / execute             │
│ ├── 复杂度: simple (1-2步) / medium (3-5步) / complex (6+步) │
│ ├── 上下文大小: small (<4K) / medium (<32K) / large (>32K)   │
│ └── 数据依赖: stateless / stateful (需要会话历史)              │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼──────────────────────┐
        ▼                     ▼                      ▼
┌──────────────┐     ┌──────────────┐      ┌──────────────┐
│ 简单查询队列   │     │ 标准分析队列   │      │ 复杂生成队列   │
│              │     │              │      │              │
│ 特征:        │     │ 特征:        │      │ 特征:        │
│ 1-2 步 LLM   │     │ 3-5 步链     │      │ 6+ 步链      │
│ 延迟 < 2s    │     │ 延迟 < 10s   │      │ 延迟 < 60s   │
│ 无工具调用    │     │ 有工具调用    │      │ 多轮工具调用  │
│              │     │              │      │              │
│ 消费者: 2-3   │     │ 消费者: 5-8   │      │ 消费者: 1-2  │
│ 模型: Haiku   │     │ 模型: Sonnet │      │ 模型: Opus   │
└──────────────┘     └──────────────┘      └──────────────┘
```

#### 4.3.4 混合队列架构（生产推荐）

这是目前业界 Agent 平台实践中被证明最有效的模式：

```
生产级混合队列架构:
───────────────────

                                ┌─────────────────────────┐
                                │    全局入队调度器         │
                                │    (请求分类 + 路由)      │
                                └──────────┬──────────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
                    ▼                      ▼                      ▼
           ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
           │ 租户 A          │   │ 租户 B          │   │ 租户 C          │
           │ ┌────────────┐  │   │ ┌────────────┐  │   │ ┌────────────┐  │
           │ │ P1: Haiku  │  │   │ │ P1: Haiku  │  │   │ │ P1: Haiku  │  │
           │ │ P2: Sonnet │  │   │ │ P2: Sonnet │  │   │ │ P2: Sonnet │  │
           │ │ P3: Opus   │  │   │ │ P3: Opus   │  │   │ │ P3: Opus   │  │
           │ └────────────┘  │   │ └────────────┘  │   │ └────────────┘  │
           └────────────────┘   └────────────────┘   └────────────────┘
                                           │
                                           ▼
                              ┌─────────────────────────┐
                              │    全局工作调度器          │
                              │    (权重轮询 + 公平队列)   │
                              └──────────┬──────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
                    ▼                    ▼                    ▼
           ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
           │ Haiku Worker   │   │ Sonnet Worker  │   │ Opus Worker    │
           │ Pool           │   │ Pool           │   │ Pool           │
           │ (高并发, 20+ ) │   │ (中并发, 10 )  │   │ (低并发, 3 )   │
           └────────────────┘   └────────────────┘   └────────────────┘
```

### 4.4 背压机制 (Back-pressure)

队列不能是无限缓冲区——当队列满时，需要一个机制告诉上游"慢下来"。

#### 4.4.1 基于队列深度的背压

```
背压信号:
─────────

队列深度                    状态                    行为
────────                    ────                    ────
0% - 60%   ████████░░░░    正常                    正常接收请求
60% - 80%  ████████████░░  警告                    返回 429 给不重要的请求
80% - 95%  ██████████████  过载                    拒绝新请求，返回 Retry-After
95% - 100% ████████████████ 危急                    丢包 (优先丢弃低优先级)

实现:
├── 队列深度 < 阈值 → 正常处理
├── 队列深度 > 阈值 → 返回 HTTP 429 + Retry-After Header
├── 队列深度 > 紧急阈值 → 熔断 (Circuit Breaker)
└── 队列深度恢复正常 → 逐步恢复流量 (避免 thundering herd)
```

#### 4.4.2 自适应并发窗口

受 TCP 拥塞控制启发，动态调整 Agent 的并发消费数。

```
Adaptive Concurrency Control:
─────────────────────────────

类似 TCP Vegas/CUBIC 的拥塞控制:

初始状态:
  window = 10 (初始并发数)

成功处理周期:
  如果没有错误 → window += 1 (增加窗口)

错误检测:
  如果出现 429 (LLM API 限流) → window ×= 0.7 (快速减半)
  如果出现超时 → window ×= 0.9 (缓慢减少)
  
稳定状态:
  如果窗口稳定 → 保持 (在拥塞窗口附近震荡)

代码映射:
  window → Agent 同时处理的请求数 (inflight requests)
  RTT → Agent 端到端响应时间 (从入队到完成)
  丢包 → LLM API 429 或超时失败
  拥塞避免 → 窗口缓慢增长直到出现失败
```

```python
class AdaptiveConcurrencyController:
    """自适应并发控制器——防止 LLM API 限流"""
    
    def __init__(self, 
                 initial_window: int = 10,
                 min_window: int = 2,
                 max_window: int = 50):
        self.window = initial_window
        self.min_window = min_window
        self.max_window = max_window
        self.inflight = 0
        self.success_count = 0
        self.fail_count = 0
    
    async def acquire(self):
        """获取一个并发槽位"""
        while self.inflight >= self.window:
            await asyncio.sleep(0.01)  # 等待槽位
        self.inflight += 1
    
    def release(self, success: bool):
        """释放槽位并更新窗口"""
        self.inflight -= 1
        
        if success:
            self.success_count += 1
            self.fail_count = 0
            # AIMD: 每成功一个，微小增加窗口
            if self.success_count >= self.window:
                self.window = min(self.window + 1, self.max_window)
                self.success_count = 0
        else:
            self.fail_count += 1
            self.success_count = 0
            # 失败时快速减半
            self.window = max(self.window // 2, self.min_window)
```

#### 4.4.3 协作式背压

完整的端到端背压——从消费者到队列到生产者到用户。

```
协作式背压:
───────────

用户 ──▶ API ──▶ 队列 ──▶ 消费者 ──▶ LLM API
 ▲        ▲       ▲         ▲          ▲
 │        │       │         │          │
 └────────┴───────┴─────────┴──────────┘
           背压信号链

背压传播路径:
1. LLM API 返回 429 → 消费者降低消费速率
2. 消费者速度变慢 → 队列深度增加 → 队列入口开始拒绝
3. 队列拒绝 → API 网关返回 429/503 → 客户端重试策略触发
4. 客户端收到 Retry-After → 用户侧等待 → 自然降低请求率

实现要点:
├── 每一层都要传递背压信号，不能"静默消化"
├── 背压信号要包含"建议等待时间" (Retry-After)
├── 客户端必须遵守 Retry-After 指示
└── 需要 end-to-end 链路追踪来诊断背压源头
```

### 4.5 延迟转移与队列控制

队列不仅仅是一个缓冲区，它还是一个"延迟转移器"：将瞬时高延迟（突发导致的超时风险）转换为可控的排队延迟。

#### 4.5.1 动态优先级调整

```
基于等待时间的优先级调整:
─────────────────────────

请求在队列中等待不同时间后的处理优先级:

等待 0-5s: 按原始优先级处理
  └── P1 → 优先处理
  └── P2 → 正常处理
  └── P3 → 有空闲资源时处理

等待 5-15s: 低优先级开始提升
  └── P1 → 优先处理 (不变)
  └── P2 → 按 P1 处理 (老化提升)
  └── P3 → 按 P2 处理 (老化提升)

等待 > 15s: 所有请求紧急处理
  └── 所有请求 → 按 P1 处理 (防止丢失)
  └── 触发告警: 队列等待时间异常
```

#### 4.5.2 最早截止时间优先调度 (EDF)

对于有时间约束的 Agent 请求，EDF 是更自然的调度策略。

```
EDF 调度:
─────────

每个 Agent 请求携带一个 deadline:
├── "用户期望在 5s 内得到回复"
├── "批量任务需要在 1 小时内完成"
├── "定时任务截止时间为 10:00"

调度算法:
1. 队列中的请求按 deadline 排序 (最早 deadline 先出队)
2. 如果当前时间 > deadline → 请求"过期"，降级处理
3. 过期请求: 返回缓存结果 / 简化处理 / 返回错误

实现示例:
  ┌────────────────────────────────────────┐
  │ 队列时间线:                              │
  │                                         │
  │ now                                         t+5s      t+10s
  │  │──[Req-A DL:2s]──[Req-B DL:4s]──[Req-C DL:8s]──▶
  │     ↑ 先处理          ↑ 次优先         ↑ 可等待
  │                                         │
  │ 如果 t+2s 到达时 Req-A 还没出队:         │
  │  → Req-A 过期 → 返回"抱歉，处理超时"      │
  └────────────────────────────────────────┘
```

#### 4.5.3 强化学习驱动的队列调度

前沿实践：使用 RL 优化队列调度策略。

```
RL-based Queue Scheduling:
───────────────────────────

状态空间 (State):
├── 每个队列的当前深度
├── 每个队列的等待时间分布 (P50, P95, P99)
├── LLM API 当前可用配额
├── 当前活跃 Worker 数
├── 请求到达速率 (近期平均)
└── 系统负载指标 (CPU, 内存, 连接数)

动作空间 (Action):
├── 从哪个队列取出下一个请求
├── 每个队列分配多少个 Worker
├── 是否要拒绝某个优先级的请求
├── 是否需要降级处理
└── 是否需要弹性伸缩

奖励函数 (Reward):
├── 正奖励: SLA 达标 + 高吞吐 + 低成本
├── 负奖励: SLA 违反 + API 429 + 队列溢出
└── 目标: 在所有约束下最大化长期累积奖励

生产实践:
├── 先用模拟环境训练 (基于历史流量回放)
├── 在低风险时段上线 (shadow mode)
├── 逐步提升策略影响比例
└── 始终保留一个降级策略 (fallback) 到传统方案
```

## 5. 代码实现

### 5.1 基于 asyncio 的优先级队列

一个生产级的优先级 Agent 队列：

```python
"""
Agent 优先级队列 — 基于 asyncio 的生产者-消费者模式

特性:
- 多优先级支持 (critical, normal, batch)
- 基于等待时间的老化升级
- 自适应的并发控制
- 背压信号 (基于队列深度)
"""
import asyncio
import enum
import time
import uuid
from dataclasses import dataclass, field
from typing import Any, Optional


class Priority(enum.IntEnum):
    """优先级枚举 — 数值越小优先级越高"""
    CRITICAL = 0   # 交互式请求，即时处理
    NORMAL = 1     # 标准 Agent 请求
    BATCH = 2      # 后台批处理任务


@dataclass(order=True)
class AgentTask:
    """队列中的 Agent 任务"""
    priority: int
    enqueue_time: float = field(compare=False)
    task_id: str = field(compare=False, default_factory=lambda: uuid.uuid4().hex[:8])
    payload: dict = field(compare=False, default=None)
    deadline: Optional[float] = field(compare=False, default=None)
    
    @property
    def wait_time(self) -> float:
        return time.monotonic() - self.enqueue_time
    
    @property
    def effective_priority(self) -> int:
        """老化机制: 等待时间越长, 有效优先级越高"""
        # 每等待 10 秒提升一个优先级等级 (但不能超过 CRITICAL)
        aging_boost = int(self.wait_time / 10)
        return max(Priority.CRITICAL, self.priority - aging_boost)
    
    @property
    def is_expired(self) -> bool:
        if self.deadline is None:
            return False
        return time.monotonic() > self.deadline


class AgentPriorityQueue:
    """支持优先级老化机制的任务队列"""
    
    def __init__(self):
        self._queues = {
            Priority.CRITICAL: [],
            Priority.NORMAL: [],
            Priority.BATCH: [],
        }
        self._condition = asyncio.Condition()
    
    async def put(self, task: AgentTask):
        """向队列中添加任务"""
        async with self._condition:
            self._queues[Priority(task.priority)].append(task)
            self._condition.notify()
    
    async def get(self) -> Optional[AgentTask]:
        """获取最高优先级的可用任务"""
        async with self._condition:
            while True:
                task = self._get_highest_priority_task()
                if task is not None:
                    return task
                await self._condition.wait()
    
    def _get_highest_priority_task(self) -> Optional[AgentTask]:
        """考虑老化机制后, 返回有效优先级最高的任务"""
        now = time.monotonic()
        
        for priority_level in [Priority.CRITICAL, Priority.NORMAL, Priority.BATCH]:
            queue = self._queues[priority_level]
            
            # 移除过期任务
            valid_tasks = [t for t in queue if not t.is_expired]
            # 过期任务处理: 计数和告警
            expired_count = len(queue) - len(valid_tasks)
            if expired_count > 0:
                print(f"[WARN] 过期任务数: {expired_count}")
            self._queues[priority_level] = valid_tasks
            
            if valid_tasks:
                # 按有效优先级排序 (考虑了老化)
                valid_tasks.sort(key=lambda t: t.effective_priority)
                task = valid_tasks.pop(0)
                self._queues[priority_level] = valid_tasks
                return task
        
        return None
    
    async def qsize(self) -> dict:
        """返回各队列深度"""
        async with self._condition:
            return {
                "critical": len(self._queues[Priority.CRITICAL]),
                "normal": len(self._queues[Priority.NORMAL]),
                "batch": len(self._queues[Priority.BATCH]),
            }


class AgentWorkerPool:
    """自适应消费者 Worker 池"""
    
    def __init__(
        self,
        queue: AgentPriorityQueue,
        max_concurrency: int = 20,
        min_workers: int = 2,
        llm_rate_limit: int = 100,  # LLM API 每分钟允许的请求数
    ):
        self.queue = queue
        self.max_concurrency = max_concurrency
        self.min_workers = min_workers
        self.llm_api_quota = llm_rate_limit  # 每分钟配额
        
        # 运行时状态
        self.active_workers = 0
        self.llm_requests_this_minute = 0
        self.quota_reset_time = time.monotonic()
        self.is_running = False
    
    async def start(self):
        """启动 Worker 池"""
        self.is_running = True
        workers = [self._worker_loop(i) for i in range(self.min_workers)]
        await asyncio.gather(*workers)
    
    async def stop(self):
        self.is_running = False
    
    async def _worker_loop(self, worker_id: int):
        """单个 Worker 的主循环"""
        while self.is_running:
            # 从队列获取任务
            task = await self.queue.get()
            
            # 检查 LLM API 配额
            await self._wait_for_llm_quota()
            
            # 执行任务
            self.active_workers += 1
            self.llm_requests_this_minute += 1
            try:
                await self._execute_task(task, worker_id)
            except Exception as e:
                print(f"[ERROR] Worker {worker_id}: 任务 {task.task_id} 失败: {e}")
            finally:
                self.active_workers -= 1
    
    async def _wait_for_llm_quota(self):
        """等待有可用的 LLM API 配额"""
        now = time.monotonic()
        # 每分钟重置
        if now - self.quota_reset_time >= 60:
            self.llm_requests_this_minute = 0
            self.quota_reset_time = now
        
        while self.llm_requests_this_minute >= self.llm_api_quota:
            # 等待配额重置
            wait_time = 60 - (now - self.quota_reset_time)
            if wait_time > 0:
                print(f"[BACKPRESSURE] LLM 配额耗尽, 等待 {wait_time:.1f}s")
                await asyncio.sleep(min(wait_time, 1.0))
            now = time.monotonic()
            if now - self.quota_reset_time >= 60:
                self.llm_requests_this_minute = 0
                self.quota_reset_time = now
    
    async def _execute_task(self, task: AgentTask, worker_id: int):
        """执行 Agent 任务"""
        wait_time = task.wait_time
        effective_priority = task.effective_priority
        
        print(f"[Worker {worker_id}] 处理任务 {task.task_id} "
              f"(原优先级: {task.priority}, 有效: {effective_priority}, "
              f"等待: {wait_time:.1f}s)")
        
        # 模拟 Agent 处理
        # 实际场景: 这里调用 Agent 执行器, 包含 LLM 调用 + 工具执行
        await asyncio.sleep(task.payload.get("processing_time", 1.0))
        
        print(f"[Worker {worker_id}] 完成 {task.task_id}")
```

### 5.2 Redis 后端优先级工作队列

纯内存队列无法跨进程共享。生产环境通常使用 Redis 实现分布式队列：

```python
"""
Redis 后端工作队列 — 支持优先级和消费者组

使用 Redis Streams + Sorted Set 实现优先级队列。
Redis Streams 保证 at-least-once 交付, Sorted Set 提供优先级排序。
"""
import json
import time
import asyncio
from typing import Optional, Callable
from dataclasses import dataclass
import redis.asyncio as aioredis


@dataclass
class RedisQueueConfig:
    redis_url: str = "redis://localhost:6379/0"
    stream_key: str = "agent:work:stream"
    priority_key: str = "agent:work:priority"
    consumer_group: str = "agent-workers"
    consumer_name: str = "worker-1"
    max_len: int = 10000  # 最大队列长度
    claim_timeout: int = 300  # 任务认领超时 (秒)


class RedisPriorityQueue:
    """
    基于 Redis Stream + Sorted Set 的优先级队列
    
    架构说明:
    - Redis Stream: 按时间序存储所有任务 (持久化 + 消费者组)
    - Sorted Set: 按优先级排序的任务索引 (用于优先级出队)
    - 两者结合: 既保证消息不丢, 又支持优先级调度
    """
    
    def __init__(self, config: RedisQueueConfig):
        self.config = config
        self.redis: Optional[aioredis.Redis] = None
    
    async def connect(self):
        """连接 Redis"""
        self.redis = await aioredis.from_url(
            self.config.redis_url,
            decode_responses=True
        )
        # 创建消费者组 (如果不存在)
        try:
            await self.redis.xgroup_create(
                self.config.stream_key,
                self.config.consumer_group,
                id="0",
                mkstream=True
            )
        except aioredis.ResponseError as e:
            if "BUSYGROUP" not in str(e):
                raise
    
    async def enqueue(self, task: dict, priority: int = 1):
        """
        向队列中添加任务
        
        Args:
            task: 任务数据
            priority: 优先级 (0=最高, 1=正常, 2=最低)
        """
        task_id = f"task:{time.time_ns()}:{hash(json.dumps(task, sort_keys=True))}"
        
        # 1. 写入 Stream (持久化存储)
        await self.redis.xadd(
            self.config.stream_key,
            {"task_id": task_id, "payload": json.dumps(task)},
            maxlen=self.config.max_len
        )
        
        # 2. 写入 Sorted Set (优先级索引)
        # score = priority * 1e15 + timestamp (确保同优先级按时间序)
        score = priority * 1e15 + time.time_ns() % int(1e15)
        await self.redis.zadd(self.config.priority_key, {task_id: score})
        
        return task_id
    
    async def dequeue(self) -> Optional[dict]:
        """
        从队列中取出最高优先级的任务
        
        流程:
        1. 从 Sorted Set 获取优先级最高的 task_id
        2. 从 Stream 中获取对应的任务数据
        3. 从 Sorted Set 中删除已出队的记录
        """
        # 1. 获取最高优先级任务
        results = await self.redis.zrange(
            self.config.priority_key, 0, 0, withscores=True
        )
        if not results:
            return None
        
        task_id, score = results[0]
        
        # 2. 从 Stream 读取任务数据
        # 使用 XRANGE 按 task_id 查找
        entries = await self.redis.xrange(
            self.config.stream_key, min=task_id, max=task_id
        )
        
        if not entries:
            # Stream 中可能已被清理, 从 Sorted Set 中删除
            await self.redis.zrem(self.config.priority_key, task_id)
            return None
        
        stream_id, data = entries[0]
        payload = json.loads(data["payload"])
        priority = int(score // 1e15)
        
        # 3. 从 Sorted Set 中删除
        await self.redis.zrem(self.config.priority_key, task_id)
        
        return {
            "task_id": task_id,
            "payload": payload,
            "priority": priority,
            "stream_id": stream_id,
        }
    
    async def ack(self, stream_id: str):
        """确认任务完成"""
        await self.redis.xack(
            self.config.stream_key,
            self.config.consumer_group,
            stream_id
        )
    
    async def queue_depth(self) -> int:
        """获取队列深度"""
        return await self.redis.zcard(self.config.priority_key)


class RedisBackedAgentQueue:
    """
    生产级 Agent 工作队列 — 完整示例
    
    特性:
    - 分布式: 多个 Worker 共享同一个队列
    - 持久化: Redis Stream 持久化, 崩溃恢复
    - 优先级: Sorted Set 实现优先级调度
    - 控制: 自适应并发 + 背压
    """
    
    def __init__(
        self,
        redis_config: RedisQueueConfig,
        min_workers: int = 2,
        max_workers: int = 20,
        llm_rate_limit: int = 1000,  # 每分钟
    ):
        self.queue = RedisPriorityQueue(redis_config)
        self.min_workers = min_workers
        self.max_workers = max_workers
        self.llm_rate_limit = llm_rate_limit
        
        # 运行时状态
        self.current_workers = min_workers
        self.running = False
        self.metrics = {
            "enqueued": 0,
            "dequeued": 0,
            "failed": 0,
            "llm_429_count": 0,
        }
    
    async def start(self):
        """启动队列处理"""
        await self.queue.connect()
        self.running = True
        
        # 启动 Worker
        workers = [
            self._worker_loop(f"worker-{i}")
            for i in range(self.min_workers)
        ]
        await asyncio.gather(*workers)
    
    async def enqueue_request(
        self,
        user_message: str,
        priority: int = 1,
        context: Optional[dict] = None,
    ) -> str:
        """向队列提交 Agent 请求"""
        task = {
            "type": "agent_request",
            "message": user_message,
            "context": context or {},
            "enqueue_time": time.time(),
        }
        task_id = await self.queue.enqueue(task, priority=priority)
        self.metrics["enqueued"] += 1
        return task_id
    
    async def _worker_loop(self, worker_name: str):
        """Worker 主循环"""
        llm_calls_this_minute = 0
        quota_reset = time.monotonic()
        
        while self.running:
            # 1. 出队
            task = await self.queue.dequeue()
            if task is None:
                await asyncio.sleep(0.1)
                continue
            
            # 2. 检查 LLM API 配额
            now = time.monotonic()
            if now - quota_reset >= 60:
                llm_calls_this_minute = 0
                quota_reset = now
            
            if llm_calls_this_minute >= self.llm_rate_limit:
                # LLM 配额耗尽, 等待
                wait = 60 - (now - quota_reset)
                print(f"[{worker_name}] LLM 配额耗尽, 等待 {wait:.0f}s")
                # 将任务放回队列 (重新入队)
                await self.queue.enqueue(
                    task["payload"], priority=task["priority"]
                )
                await asyncio.sleep(min(wait, 5))
                continue
            
            # 3. 处理任务
            llm_calls_this_minute += 1
            self.metrics["dequeued"] += 1
            
            try:
                await self._process_agent_task(task, worker_name)
                await self.queue.ack(task["stream_id"])
            except Exception as e:
                self.metrics["failed"] += 1
                print(f"[{worker_name}] 处理失败: {e}")
                
                # 判断是否为 429
                if "429" in str(e):
                    self.metrics["llm_429_count"] += 1
                    # 背压: 降低消费速度
                    await asyncio.sleep(2.0)
    
    async def _process_agent_task(self, task: dict, worker: str):
        """实际的 Agent 任务处理"""
        payload = task["payload"]
        print(f"[{worker}] 处理: {payload['message'][:50]}...")
        
        # 模拟: 实际的 LLM API 调用
        await asyncio.sleep(0.5)
        
        print(f"[{worker}] 完成")
    
    async def get_metrics(self) -> dict:
        """返回运行时指标"""
        depth = await self.queue.queue_depth()
        return {
            **self.metrics,
            "queue_depth": depth,
            "active_workers": self.current_workers,
        }
```

### 5.3 自适应消费者池

根据系统状态动态调整消费能力：

```python
"""
自适应消费者池 — 动态调整并发 Worker 数量

策略:
- 队列深度增加 → 增加 Worker (弹性)
- LLM API 返回 429 → 减少 Worker (背压)
- 系统空闲 → 减少 Worker (节约资源)
"""
import asyncio
import time
from typing import Optional


class AdaptiveWorkerPool:
    """
    自适应 Worker 池
    
    类似 TCP 拥塞控制的 AIMD 策略:
    - Additive Increase: 平稳增长 Worker 数
    - Multiplicative Decrease: 遇到限流时快速减少
    
    核心指标:
    - inflight: 正在处理的请求数
    - queue_depth: 等待队列深度
    - llm_429_rate: LLM API 限流频率
    - processing_time_p50: 处理延迟中位数
    """
    
    def __init__(
        self,
        redis_queue: RedisPriorityQueue,
        min_workers: int = 2,
        max_workers: int = 50,
        target_queue_depth: int = 100,
        target_processing_time: float = 5.0,  # 秒
        scale_up_factor: float = 1.5,
        scale_down_factor: float = 0.5,
    ):
        self.queue = redis_queue
        self.min_workers = min_workers
        self.max_workers = max_workers
        self.target_queue_depth = target_queue_depth
        self.target_processing_time = target_processing_time
        self.scale_up_factor = scale_up_factor
        self.scale_down_factor = scale_down_factor
        
        # 运行时状态
        self.current_workers = min_workers
        self.active_tasks: set = set()
        self.processing_times: list = []
        self.llm_429_count = 0
        self.total_processed = 0
        self.last_scale_time = time.monotonic()
        
        # 控制信号
        self._stop_event = asyncio.Event()
    
    async def start(self):
        """启动自适应池"""
        # 启动自适应调节器
        asyncio.create_task(self._auto_scaler())
        
        # 启动初始 Worker
        workers = [
            self._worker_loop(i)
            for i in range(self.min_workers)
        ]
        await asyncio.gather(*workers)
    
    async def _auto_scaler(self):
        """自适应调节主循环"""
        while not self._stop_event.is_set():
            await asyncio.sleep(5)  # 每 5 秒评估一次
            
            depth = await self.queue.queue_depth()
            current = self.current_workers
            
            # 计算指标
            avg_processing_time = (
                sum(self.processing_times[-50:]) / 
                max(len(self.processing_times[-50:]), 1)
            )
            
            # 决策逻辑
            should_scale_up = (
                depth > self.target_queue_depth * 1.5
                and current < self.max_workers
                and self.llm_429_count < 3  # 没有频繁限流
            )
            
            should_scale_down = (
                depth < self.target_queue_depth * 0.3
                and current > self.min_workers
            )
            
            should_back_off = (
                self.llm_429_count >= 5  # 频繁限流, 必须减少负载
            )
            
            if should_back_off:
                new_count = max(int(current * 0.5), self.min_workers)
                print(f"[AUTO-SCALER] 背压减少: {current} → {new_count} "
                      f"(429 过多: {self.llm_429_count})")
                await self._set_worker_count(new_count)
                
            elif should_scale_up:
                new_count = min(int(current * self.scale_up_factor), self.max_workers)
                print(f"[AUTO-SCALER] 扩容: {current} → {new_count} "
                      f"(队列深度: {depth})")
                await self._set_worker_count(new_count)
                
            elif should_scale_down:
                new_count = max(int(current * self.scale_down_factor), self.min_workers)
                print(f"[AUTO-SCALER] 缩容: {current} → {new_count} "
                      f"(队列深度: {depth})")
                await self._set_worker_count(new_count)
            
            # 重置限流计数器 (每分钟)
            self.llm_429_count = max(0, self.llm_429_count - 1)
    
    async def _set_worker_count(self, new_count: int):
        """调整 Worker 数量"""
        # 实际实现: 管理 asyncio.Task 的创建和取消
        self.current_workers = new_count
        self.last_scale_time = time.monotonic()
    
    async def _worker_loop(self, worker_id: int):
        """单个 Worker 的执行循环"""
        while not self._stop_event.is_set():
            # 检查是否有足够的 Worker 槽位
            if worker_id >= self.current_workers:
                await asyncio.sleep(1)
                continue
            
            task = await self.queue.dequeue()
            if task is None:
                await asyncio.sleep(0.1)
                continue
            
            start_time = time.monotonic()
            self.active_tasks.add(worker_id)
            
            try:
                await self._execute_task(task, worker_id)
                
                # 记录处理时间
                elapsed = time.monotonic() - start_time
                self.processing_times.append(elapsed)
                self.total_processed += 1
                
            except Exception as e:
                error_str = str(e)
                if "429" in error_str:
                    self.llm_429_count += 1
            
            finally:
                self.active_tasks.discard(worker_id)
    
    async def _execute_task(self, task: dict, worker_id: int):
        """执行具体的 Agent 任务"""
        # 实际实现: LLM API 调用 + 工具执行
        await asyncio.sleep(0.5)  # 模拟


class QueueMetrics:
    """队列指标收集器"""
    
    def __init__(self, window_size: int = 300):  # 5 分钟窗口
        self.window_size = window_size
        self.enqueue_times: list = []
        self.dequeue_times: list = []
        self.processing_times: list = []
        self.errors: list = []
    
    def record_enqueue(self):
        self.enqueue_times.append(time.monotonic())
        if len(self.enqueue_times) > self.window_size:
            self.enqueue_times.pop(0)
    
    def record_dequeue(self):
        self.dequeue_times.append(time.monotonic())
        if len(self.dequeue_times) > self.window_size:
            self.dequeue_times.pop(0)
    
    def record_processing(self, duration: float):
        self.processing_times.append(duration)
        if len(self.processing_times) > self.window_size:
            self.processing_times.pop(0)
    
    @property
    def arrival_rate(self) -> float:
        """请求到达速率 (QPS)"""
        if len(self.enqueue_times) < 2:
            return 0
        window = self.enqueue_times[-1] - self.enqueue_times[0]
        return len(self.enqueue_times) / window if window > 0 else 0
    
    @property
    def processing_rate(self) -> float:
        """处理速率 (QPS)"""
        if len(self.processing_times) < 2:
            return 0
        total = sum(self.processing_times[-100:])
        return len(self.processing_times[-100:]) / total if total > 0 else 0
    
    @property
    def queue_growth_rate(self) -> float:
        """队列增长速率 (正 = 积压, 负 = 消耗)"""
        return self.arrival_rate - self.processing_rate
    
    def summary(self) -> dict:
        """队列健康摘要"""
        return {
            "arrival_rate_qps": round(self.arrival_rate, 2),
            "processing_rate_qps": round(self.processing_rate, 2),
            "queue_growth_rate": round(self.queue_growth_rate, 2),
            "avg_processing_time_s": round(
                sum(self.processing_times[-50:]) / max(len(self.processing_times[-50:]), 1),
                2
            ),
            "p95_processing_time_s": round(
                sorted(self.processing_times[-100:])[
                    int(len(self.processing_times[-100:]) * 0.95)
                ] if len(self.processing_times[-100:]) > 0 else 0,
                2
            ),
        }
```

### 5.4 熔断器与队列深度监控联合

熔断器不应该只基于"错误率"——它应该结合队列深度来做更智能的决策：

```python
"""
熔断器与队列深度联合监控

传统熔断器: 基于错误率开/关
增强熔断器: 错误率 + 队列深度 + LLM API 配额状态 → 多级保护
"""
import enum
import time
from typing import Optional


class CircuitState(enum.Enum):
    CLOSED = "closed"          # 正常
    WARNING = "warning"        # 队列深度较高, 开始降级
    HALF_OPEN = "half_open"    # 尝试恢复
    BACKPRESSURE = "backpressure"  # 背压开启, 拒绝低优先级
    OPEN = "open"              # 熔断开启


class AgentCircuitBreaker:
    """
    Agent 系统专用熔断器
    
    维度:
    - 错误率: LLM API 429/5xx 比例
    - 队列深度: 当前队列积压程度
    - 等待时间: 请求在队列中的等待时间
    
    状态:
    1. CLOSED: 一切正常, 所有请求进入队列
    2. WARNING: 队列深度 > 80% 或错误率 > 5%, 降级 P3 请求
    3. HALF_OPEN: 背压模式, 只接受 P1 请求
    4. BACKPRESSURE: 所有新请求返回 429 + Retry-After
    5. OPEN: 全熔断, 返回 503
    """
    
    def __init__(
        self,
        queue_max_depth: int = 1000,
        error_threshold: float = 0.1,    # 10% 错误率触发熔断
        queue_threshold: float = 0.8,    # 80% 队列深度触发警告
        recovery_timeout: float = 30.0,  # 半开状态尝试恢复时间
    ):
        self.queue_max_depth = queue_max_depth
        self.error_threshold = error_threshold
        self.queue_threshold = queue_threshold
        self.recovery_timeout = recovery_timeout
        
        self.state = CircuitState.CLOSED
        self.last_state_change = time.monotonic()
        self.error_count = 0
        self.total_count = 0
        self.last_error_time = 0
    
    def record_success(self):
        """记录一次成功"""
        self.total_count += 1
    
    def record_failure(self, error_type: str = "unknown"):
        """记录一次失败"""
        self.total_count += 1
        self.error_count += 1
        self.last_error_time = time.monotonic()
        
        # 更新状态
        self._evaluate_state()
    
    def _evaluate_state(self):
        """评估当前状态"""
        error_rate = self.error_count / max(self.total_count, 1)
        now = time.monotonic()
        
        if self.state == CircuitState.CLOSED:
            if error_rate > self.error_threshold:
                self._transition_to(CircuitState.OPEN)
        
        elif self.state == CircuitState.OPEN:
            if now - self.last_state_change > self.recovery_timeout:
                self._transition_to(CircuitState.HALF_OPEN)
        
        elif self.state == CircuitState.HALF_OPEN:
            # 半开状态: 如果错误率还是高, 回到 OPEN
            if error_rate > self.error_threshold:
                self._transition_to(CircuitState.OPEN)
            else:
                self._transition_to(CircuitState.CLOSED)
    
    def _transition_to(self, new_state: CircuitState):
        """状态转换"""
        self.state = new_state
        self.last_state_change = time.monotonic()
        # 重置计数器
        if new_state == CircuitState.CLOSED:
            self.error_count = 0
            self.total_count = 0
    
    def get_queue_decision(self, current_depth: int, task_priority: int) -> dict:
        """
        根据当前状态和队列深度返回操作决策
        
        Returns:
            {
                "accept": bool,         # 是否接受请求
                "priority_boost": int,  # 优先级调整
                "reason": str,          # 决策原因
                "retry_after": float,   # 建议重试等待时间 (秒)
            }
        """
        depth_ratio = current_depth / max(self.queue_max_depth, 1)
        
        # 默认决策
        decision = {
            "accept": True,
            "priority_boost": 0,
            "reason": "normal",
            "retry_after": 0,
        }
        
        if self.state == CircuitState.OPEN:
            return {
                "accept": False,
                "priority_boost": 0,
                "reason": "circuit_open",
                "retry_after": self.recovery_timeout,
            }
        
        if self.state == CircuitState.BACKPRESSURE:
            if task_priority > 0:  # 非 P1 请求被拒绝
                return {
                    "accept": False,
                    "priority_boost": 0,
                    "reason": "backpressure_active",
                    "retry_after": 5.0,
                }
            # P1 请求通过
            return decision
        
        if self.state == CircuitState.WARNING:
            if task_priority >= 2:  # P3 (批量) 请求被拒绝
                return {
                    "accept": False,
                    "priority_boost": 0,
                    "reason": "high_load_reject_batch",
                    "retry_after": 10.0,
                }
            # P1/P2 请求被接受, 但标记为低优先级
            decision["priority_boost"] = 1 if task_priority == 2 else 0
        
        # 基于队列深度的额外决策
        if depth_ratio > 0.95:
            # 非常深的队列: 所有非 P1 请求拒绝
            if task_priority > 0:
                decision["accept"] = False
                decision["reason"] = "queue_full"
                decision["retry_after"] = 15.0
        
        elif depth_ratio > 0.8 and task_priority >= 2:
            # 深队列: P3 请求拒绝
            decision["accept"] = False
            decision["reason"] = "queue_deep_reject_batch"
            decision["retry_after"] = 10.0
        
        return decision
```

## 6. 能力边界与限制

队列不是银弹。理解队列的边界才能正确使用它。

### 6.1 队列持久化与性能的权衡

```
持久化等级与性能:
─────────────────

持久化等级          延迟             吞吐量        崩溃恢复
─────────          ────             ──────        ────────
无持久化 (内存)       < 1μs           100M+/s       全部丢失
                                                    
Redis AOF (每秒)     1-10ms          10K-100K/s    最多丢 1s 数据
                                                    
Redis AOF (每次)     10-50ms         1K-10K/s      不丢数据
                                                    
Kafka / Pulsar      1-10ms          100K-1M/s     不丢数据 (ISR)
                                                    
SQS / Pub/Sub       10-100ms        1K-100K/s     不丢数据

选择建议:
├── 你的 Agent 系统可以丢消息吗？
│   ├── 可以 → 内存队列 (低延迟)
│   ├── 有限容忍 → Redis AOF (平衡)
│   └── 绝对不能 → Kafka / SQS (高可靠性)
│
├── 注意: 持久化带来的不仅仅是延迟
│   ├── 运维复杂度大幅增加
│   ├── 需要处理死信队列 (DLQ)
│   └── 消息可能重复投递 (at-least-once)
│
└── 一个常见工程实践:
    内存队列 + Redis 持久化双重写入
    正常时读内存 (低延迟), 崩溃时从 Redis 恢复
```

### 6.2 队列溢出策略

当队列满了怎么办？不只是"拒绝请求"这么简单。

```
队列溢出策略对比:
─────────────────

策略                    行为                     适用场景
────                    ────                     ────────
Tail Drop (尾部丢弃)     新请求被拒                 简单粗暴, 但公平
                         先到先服务

Random Early Drop       队列深度 > 阈值时          防止全球同步
(RED, 随机早期丢弃)      概率性丢弃新请求            (thundering herd)
                                                 避免全有全无

Priority Drop          溢出时优先丢弃低            多优先级场景
(优先级丢弃)            优先级的请求

Head Drop (头部丢弃)     溢出时丢弃队列头            实时性要求高
                         部最早的请求                宁可丢旧请求

Agent 推荐方案: 优先级感知的随机早期丢弃
─────────────────────────────────────────
算法:
  1. 计算队列深度百分比: p = depth / max_depth
  2. 如果 p < 0.7: 不丢弃
  3. 如果 p >= 0.7: 丢弃概率 = min(1, (p - 0.7) / 0.3)
  4. 但低优先级请求的丢弃概率 × 2
  5. 高优先级请求的丢弃概率 × 0.5
```

### 6.3 分布式消费者的协调问题

多个 Agent 实例从同一个队列消费时，存在微妙的协调问题。

```
问题 1: 惊群效应 (Thundering Herd)
─────────────────────────────────

场景:
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ Worker A  │     │ Worker B  │     │ Worker C  │
  └─────┬────┘     └─────┬────┘     └─────┬────┘
        │                │                │
        └────────────────┼────────────────┘
                         │
                    ┌────▼────┐
                    │ 队列为空  │
                    └─────────┘

新任务到达时, 所有等待的 Worker 都被唤醒
但只有一个能拿到任务, 其他 Worker 空转

解决: 使用 Redis Stream 消费者组 (XREADGROUP)
    每个消息只投递给一个消费者组成员
```

```
问题 2: 同一 API Key 的跨节点协调
─────────────────────────────────

             ┌─────────────────────────────┐
             │       LLM API Key 配额        │
             │       每分钟 1000 RPM          │
             └─────────────────────────────┘
                        ▲
          ┌─────────────┼─────────────┐
          │             │             │
    ┌─────┴────┐  ┌─────┴────┐  ┌─────┴────┐
    │ Node 1   │  │ Node 2   │  │ Node 3   │
    │ 400 RPM  │  │ 500 RPM  │  │ 600 RPM  │  ← 每个节点独立计数!
    └──────────┘  └──────────┘  └──────────┘
                          ↑
                    总计 1500 RPM → 超过配额!
                    
问题: 每个节点不知道其他节点的消费量

解决方案:
├── 方案 A: 中心化配额服务器 (Redis 计数器)
│   ┌──────────────┐
│   │ Redis INCR    │ ← 所有节点在调用 API 前原子递增
│   └──────────────┘     配额计数器, 超过则等待
│
├── 方案 B: 配额分区 (Partitioned Quota)
│   Node 1: 333 RPM, Node 2: 333 RPM, Node 3: 334 RPM
│   缺点: 浪费配额 (Node 3 用不完别的也不能用)
│
└── 方案 C: 本地桶 + 中心协调 (推荐)
    ┌──────────────────────────────────┐
    │ 每个节点: 本地 Token Bucket 100/m  │
    │ + 全局 Redis 配额池 (补充本地桶)   │
    │ 本地桶用完 → 从中心池借 Token     │
    └──────────────────────────────────┘
```

```
问题 3: 僵尸任务与重新入队
─────────────────────────

场景:
  Worker A 领取了任务 → 处理过程中崩溃
  → 任务"丢失"在 Stream 中 → 永远不会被确认
  → 称为"僵尸任务" (zombie task / ghost message)

Redis Stream 的解决: Pending Entries List (PEL)
  ├── 每个未确认的消息都在 PEL 中
  ├── 其他消费者可以 CLAIM 超时的 PEL 消息
  └── XAUTOCLAIM: 自动重新分配超时消息

关键参数: 超时时间设置
  ├── 太短 → 正常处理中的任务被错误重新分配 (重复处理)
  ├── 太长 → 真正崩溃的任务等待太久才恢复
  └── 建议: Processing Time P99 × 3
```

### 6.4 队列监控盲点

"健康的队列深度"可能掩盖正在发生的问题。

```
盲点 1: 队列深度正常, 但处理质量下降
────────────────────────────────────

  ┌─────────────────┐
  │ 队列深度: 50      │ ← "看起来正常"
  │ 等待时间: 2s      │ ← "看起来正常"
  └─────────────────┘
  
  但! Worker 正在返回降级结果:
  ┌─────────────────┐
  │ Worker: 使用快    │ ← LLM API 返回 429 过多
  │ 速但质量低的模型   │    Worker 自动降级到小模型
  │ Response: "我不知道"│ ← 用户得到的回答质量很差
  └─────────────────┘

解决: 除了队列指标, 还要监控:
├── 模型使用分布 (Haiku vs Sonnet vs Opus 比例)
├── 响应质量评分 (语义相似度、拒绝回答率)
├── LLM API 429 频率
└── 降级处理占比
```

```
盲点 2: 队列深度下降, 但是因为 Worker 在"作弊"
────────────────────────────────────────────

  ┌─────────────────┐
  │ 队列深度: 120 → 80 │ ← "系统在恢复!"
  └─────────────────┘

  但真相:
  ┌─────────────────┐
  │ Worker: 主动丢弃   │ ← 为了提高吞吐, Worker 跳过
  │ 部分复杂请求       │    了复杂任务, 只处理简单任务
  │ 队列里剩下: 全部是  │ ← 简单任务处理完了
  │ 复杂请求           │    复杂请求无人能处理
  └─────────────────┘

解决: 监控队列中的请求类型分布:
├── 简单请求占比 vs 复杂请求占比
├── 每种类型的处理时间趋势
├── 各类型的完成率 (completion rate)
└── Worker 任务选择偏好 (是否偏向简单任务)
```

```
盲点 3: 队列等待时间短, 但端到端延迟高
────────────────────────────────────

用户视角的延迟:
用户发送请求 ──▶ ┌────── 队列 ──────▶ ┌── LLM API ──▶ 用户收到回复
                 │ 等待 500ms         │  调用 15s
                 └───────────────────┘  └─────────────┘
                 ↑ 队列指标: "健康"     ↑ LLM API 延迟高
                                      (P99 事件)

队列监控看到: 等待时间 500ms ← "正常"
用户感受到: 15.5s ← "系统崩了!"

解决: 追踪端到端延迟, 不要只看队列:
├── 用户感知延迟 (E2E)
├── 队列等待时间 (组件级别)
├── LLM API 调用时间 (组件级别)
├── 工具执行时间 (组件级别)
└── 延迟分解: 队列占 x%, LLM 占 y%, 工具占 z%
```

## 7. 与其他方案对比

### 7.1 队列技术对比矩阵

```
Agent 系统通用队列技术选型:
───────────────────────────

维度               In-Memory Queue    Redis List/Stream       RabbitMQ
────               ───────────────    ────────────────        ─────────
部署依赖            无                 Redis 实例             Erlang 运行时
延迟                < 1μs             1-5ms                  5-20ms
吞吐量              极高 (内存速度)     高 (10万+ QPS)         高 (数万 QPS)
持久化              无                 RDB/AOF 可选           磁盘持久化
消息路由能力         无 (FIFO)         简单 (Stream 组)        丰富 (Exchange)
消费者组             N/A               原生支持               原生支持
优先级              自行实现           通过 Sorted Set         原生支持 (但有限)
死信队列             无                自行实现                原生支持
运维复杂度          无                 低 (一个 Redis)         中 (集群管理)
消息顺序            严格 FIFO          分区内有序              分区内有序
投递语义            至多一次            至少一次                至少一次 (带确认)
批量消费             不适用             支持 (XREAD count)     支持 (basic.qos)
TTL/过期            不适用             Stream 支持            支持 (per queue/msg)
延时消息            不适用             自行实现 (ZSET)         原生支持 (x-delay)

维度               Apache Kafka        AWS SQS               Google Pub/Sub
────               ────────────        ───────                ──────────────
部署依赖            ZooKeeper/KRaft     无 (托管)              无 (托管)
延迟                5-50ms              50-500ms              100-500ms
吞吐量              极高 (百万+/s)       高 (数万 QPS)          高 (数十万 QPS)
持久化              强 (磁盘, 复制)      强 (多AZ复制)           强 (多区域复制)
消息路由能力         弱 (Topic 模式)      简单 (Queue 模式)       简单 (Topic 模式)
消费者组             原生 (CG)           原生 (Visibility)      原生 (Subscription)
优先级               不支持              支持 (Redrive)         不支持
死信队列             原生 (DLQ)          原生 (DLQ)             原生 (DLQ)
运维复杂度          高 (集群运维)        无 (SaaS)              无 (SaaS)
消息顺序            分区内有序            基本 FIFO (非严格)     宽松排序
投递语义            至少一次 / 精确一次    至少一次                至少一次
批量消费             原生               原生 (10条)             原生 (批量发布)
TTL/过期            原生 (log retention) 原生                   原生
延时消息             不支持               原生                   原生
```

### 7.2 Agent 工作负载的队列 vs Pub/Sub vs Stream

```
Agent 场景的选择指南:
────────────────────

场景 1: Agent 任务分发 (1 对多 Worker)
  ┌─────────────┐
  │ 每个任务只需要    │ → 队列 (Queue)  ✓ 最佳选择
  │ 一个 Worker 处理 │    Redis Stream / SQS / RabbitMQ
  └─────────────┘

场景 2: Agent 事件广播 (1 对多订阅者)
  ┌─────────────┐
  │ Agent 完成事件  │ → Pub/Sub (Topic) ✓ 最佳选择
  │ Webhook 通知   │    Kafka / RabbitMQ Topic
  │ 监控系统订阅    │
  └─────────────┘

场景 3: Agent 活动日志 (追加写 + 回溯读)
  ┌─────────────┐
  │ Agent 行为追踪  │ → Stream / Log ✓ 最佳选择
  │ 审计日志       │    Kafka / Pulsar
  │ 调试回放       │
  └─────────────┘

场景 4: 混合架构 (最常见的 Agent 生产环境)
  ┌─────────────┐
  │ Agent 任务:     │
  │  1. 请求入队    │ Queue (Redis / SQS)
  │     → Worker 消费 │
  │  2. 处理完成    │ Pub/Sub (Kafka)
  │     → 事件广播   │
  │  3. 状态变更    │ Stream (Kafka)
  │     → 持久化日志 │
  └─────────────┘
```

### 7.3 投递语义对 Agent 系统的影响

```
Agent 任务投递语义:
──────────────────

至多一次 (At-most-once):
├── 消息可能丢, 但不会重复
├── 适用: Agent 日志、监控指标、非关键通知
├── 风险: 丢失 Agent 任务
└── 实现: 内存队列、非持久化、fire-and-forget

至少一次 (At-least-once):
├── 消息不丢, 但可能重复
├── 适用: Agent 任务分发 ✔ (主流方案)
├── 风险: 需要任务幂等性
└── 实现: Redis Stream (with ACK), SQS, Kafka

精确一次 (Exactly-once):
├── 消息不丢也不重复
├── 适用: 金融、交易类 Agent 任务
├── 代价: 性能下降 3-5x, 实现复杂
└── 实现: Kafka transactional + idempotent producer

Agent 系统的实际建议:
────────────────────

├── 大多数 Agent 任务: 至少一次 + 幂等处理
│   └── 任务 ID 去重: 如果收到重复任务, 检查是否已处理
│   └── 天然幂等的 Agent 操作: "生成摘要"重复执行也无害
│
├── 需要精确一次的场景:
│   ├── 扣费/支付类 Agent 操作
│   ├── 资源变更 (创建、删除)
│   └── 状态变更通知 (API 回调)
│
└── 幂等性实现:
    任务 ID → 已处理集合 (Redis SET with TTL)
    处理前检查 ID 是否在集合中
    处理后将 ID 加入集合 (TTL = 去重窗口)
```

### 7.4 何时用简单有界队列 vs 完整消息代理

```
决策树: 什么情况下用什么
──────────────────────

您的 Agent 系统需要队列吗?
  │
  ├── < 100 用户, 单实例, 单 API Key
  │   └── asyncio.Queue (有界) ✓
  │       不需要消息代理
  │
  ├── < 1000 用户, 多实例, 多 API Key
  │   └── Redis 队列 ✓ (轻量, 够用)
  │       Redis List / Redis Streams
  │
  ├── < 10K 用户, 多实例, 多租户
  │   └── 需要高级路由和持久化
  │       └── RabbitMQ / Redis Streams ✓
  │           需要死信队列、TTL、优先级
  │
  ├── < 100K 用户, 大规模生产
  │   └── 需要高吞吐 + 持久化 + 多消费者
  │       └── Kafka / Pulsar ✓
  │           需要批量处理、日志回放、审计
  │
  └── > 100K 用户, 全球部署
      └── 混合架构: 区域队列 + 全局事件总线
          ├── 区域: Redis Streams (低延迟)
          ├── 全局: Kafka / Pulsar (跨区域复制)
          └── 监控: Prometheus + Grafana
```

## 8. 工程优化方向

### 8.1 队列感知的自动扩缩容 (KEDA)

Kubernetes 上最自然的方案：使用 KEDA scaler 基于队列深度自动扩缩 Agent Pod。

```
KEDA 架构:
──────────

                         ┌──────────────────┐
                         │    KEDA Operator  │
                         │    (ScaledObject) │
                         └────────┬─────────┘
                                  │ 监控队列深度
                                  │ 调整 Deployment 副本数
                                  │
                    ┌─────────────┼──────────────┐
                    │             │              │
                    ▼             ▼              ▼
             ┌──────────┐ ┌──────────┐  ┌──────────┐
             │ Agent    │ │ Agent    │  │ Agent    │
             │ Pod 1    │ │ Pod 2    │  │ Pod N    │
             └──────────┘ └──────────┘  └──────────┘
                    ▲             ▲              ▲
                    │             │              │
                    └─────────────┼──────────────┘
                                  │ 消费
                         ┌────────┴────────┐
                         │  任务队列 (Redis) │
                         │                 │
                         │  队列深度: 当前值 │──▶ KEDA Scaler
                         └─────────────────┘
```

```yaml
# KEDA ScaledObject 配置示例
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: agent-queue-scaler
spec:
  scaleTargetRef:
    name: agent-worker  # 目标 Deployment
  pollingInterval: 10   # 每 10 秒检查一次
  cooldownPeriod: 60    # 等待 60 秒再缩容
  minReplicaCount: 2    # 最小 Pod 数
  maxReplicaCount: 20   # 最大 Pod 数
  triggers:
    - type: redis-streams
      metadata:
        address: redis://redis-service:6379/0
        stream: agent:work:stream
        consumerGroup: agent-workers
        # 每个分区期望的未处理消息数
        # 总副本数 = max(ceil(pending / 100), minReplica)
        pendingMessagesCount: "100"
    - type: prometheus
      metadata:
        # 补充指标: LLM API 队列等待时间
        serverAddress: http://prometheus:9090
        metricName: llm_queue_wait_seconds_p95
        threshold: "5.0"
        query: |
          histogram_quantile(0.95,
            rate(llm_api_wait_seconds_bucket[5m]))
```

### 8.2 预测性队列管理

不等到队列满了才扩容——根据趋势预测未来负载。

```
预测性扩缩容:
─────────────

核心算法: 队列深度变化率预测

当前队列深度: d(t)
过去 N 个采样点: [d(t-N), d(t-N+1), ..., d(t)]

短期预测 (未来 30s):
  d(t+Δt) ≈ d(t) + Δt × d'(t)
  其中 d'(t) = (d(t) - d(t-Δt)) / Δt (一阶导数)

示例:
  队列深度: 50, 增长率: 5 QPS
  预测 30s 后: 50 + 30 × 5 = 200
  
  如果 200 > 队列安全阈值 (100):
  → 现在就开始扩容, 不等队列满了

长期预测 (未来 5min):
  使用简化的 ARIMA 模型
  或 基于历史流量的时间序列模式匹配
```

```python
class PredictiveQueueScaler:
    """基于队列深度变化率的预测性扩缩容"""
    
    def __init__(self, history_window: int = 60):
        self.depth_history: list = []  # [(timestamp, depth), ...]
        self.history_window = history_window
    
    def record_depth(self, depth: int):
        now = time.monotonic()
        self.depth_history.append((now, depth))
        # 只保留最近的 N 个采样点
        if len(self.depth_history) > self.history_window:
            self.depth_history.pop(0)
    
    def predict_depth(self, seconds_ahead: int = 30) -> float:
        """预测未来 seconds_ahead 秒后的队列深度"""
        if len(self.depth_history) < 3:
            return self.depth_history[-1][1] if self.depth_history else 0
        
        # 计算队列深度变化率 (使用最近 10s 的数据)
        recent = [p for p in self.depth_history 
                  if p[0] >= time.monotonic() - 10]
        if len(recent) < 2:
            return self.depth_history[-1][1]
        
        # 线性回归求斜率
        times = [p[0] for p in recent]
        depths = [p[1] for p in recent]
        n = len(recent)
        
        # 简化: (n*Σxy - Σx*Σy) / (n*Σx² - (Σx)²)
        sum_x = sum(times)
        sum_y = sum(depths)
        sum_xy = sum(x * y for x, y in zip(times, depths))
        sum_xx = sum(x * x for x in times)
        
        slope = (n * sum_xy - sum_x * sum_y) / (n * sum_xx - sum_x * sum_x + 1e-10)
        
        current_depth = depths[-1]
        predicted = current_depth + slope * seconds_ahead
        
        return max(0, predicted)
    
    def should_scale(self, 
                     current_workers: int,
                     max_workers: int,
                     target_depth: int = 100) -> tuple:
        """
        返回 (should_scale_up, should_scale_down, reason)
        """
        predicted_30s = self.predict_depth(30)
        predicted_60s = self.predict_depth(60)
        
        scale_up = (
            predicted_30s > target_depth * 1.5
            and current_workers < max_workers
        )
        
        scale_down = (
            predicted_60s < target_depth * 0.3
            and current_workers > 2
        )
        
        reason = ""
        if scale_up:
            reason = (f"预测 30s 后深度 {predicted_30s:.0f} "
                      f"(当前 {self.depth_history[-1][1] if self.depth_history else 0})")
        elif scale_down:
            reason = (f"预测 60s 后深度 {predicted_60s:.0f}")
        
        return scale_up, scale_down, reason
```

### 8.3 智能批处理

将相似的 Agent 请求合并为一次 LLM 调用。

```
请求批处理:
───────────

没有批处理:
  请求 A: "翻译 'hello' 到中文"
  请求 B: "翻译 'world' 到中文"
  → 2 次 LLM 调用, 2 次 Token 开销 (每次都有 Prompt 重复)

有批处理:
  Batch: ["翻译 'hello' 到中文", "翻译 'world' 到中文"]
  → 1 次 LLM 调用, 共享 Prompt 前缀

但! 批处理有代价:
├── 延迟: 需要等待足够多的请求凑成一批 (通常会延迟 100-500ms)
├── 复杂性: 需要拆解批量回复为单个响应
└── 适用性: 不是所有 Agent 请求都能批处理
```

```python
class BatchProcessor:
    """
    Agent 请求批处理器
    
    工作原理:
    1. 收集同类请求到 batch buffer
    2. 等待 batch_timeout 或 buffer 满
    3. 一批发送到 LLM
    4. 将响应拆解为单个回复
    
    适用场景:
    - 翻译、摘要、分类等无状态任务
    - 共享 System Prompt 的请求
    - 工具调用的并行执行
    """
    
    def __init__(self, 
                 batch_size: int = 10,
                 batch_timeout: float = 0.5):
        self.batch_size = batch_size
        self.batch_timeout = batch_timeout
        self.buffer: list = []
        self.buffer_lock = asyncio.Lock()
        self.flush_event = asyncio.Event()
    
    async def submit(self, request: dict) -> asyncio.Future:
        """提交请求并获得 Future (异步等待结果)"""
        future = asyncio.get_event_loop().create_future()
        
        async with self.buffer_lock:
            self.buffer.append((request, future))
            if len(self.buffer) >= self.batch_size:
                self.flush_event.set()
        
        # 启动 flush 协程 (如果还没启动)
        asyncio.ensure_future(self._maybe_flush())
        
        return await future
    
    async def _maybe_flush(self):
        """检查是否需要 flush"""
        await asyncio.sleep(self.batch_timeout)
        if self.flush_event.is_set():
            return
        self.flush_event.set()
    
    async def flush(self):
        """执行批处理"""
        await self.flush_event.wait()
        self.flush_event.clear()
        
        async with self.buffer_lock:
            batch = self.buffer[:]
            self.buffer.clear()
        
        if not batch:
            return
        
        try:
            # 批量执行 (合并 LLM 调用)
            prompts = [req["message"] for req, _ in batch]
            responses = await self._batch_llm_call(prompts)
            
            # 分发结果
            for (_, future), response in zip(batch, responses):
                if not future.done():
                    future.set_result(response)
                    
        except Exception as e:
            for _, future in batch:
                if not future.done():
                    future.set_exception(e)
    
    async def _batch_llm_call(self, prompts: list) -> list:
        """
        实际的批量 LLM 调用
        
        实现方案取决于 LLM 提供商:
        - OpenAI: 使用 Chat Completions (多个独立请求)
        - Anthropic: 独立的 API 调用 (不支持原生批处理)
        - 自定义: 合并到同一个 Prompt 中 (格式需特殊处理)
        """
        # 方案 1: 并行执行 (OpenAI)
        # tasks = [client.chat_completions(p) for p in prompts]
        # return await asyncio.gather(*tasks)
        
        # 方案 2: 合并 Prompt (自定义模型)
        # batch_prompt = "\n---\n".join(
        #     f"Request {i}: {p}" for i, p in enumerate(prompts)
        # )
        # response = await client.complete(batch_prompt)
        # return parse_batch_response(response)
        
        # 当前: 模拟
        await asyncio.sleep(0.5)
        return [{"response": f"processed: {p[:20]}"} for p in prompts]
```

### 8.4 队列可观测性

没有可观测性的队列是一个黑箱——你无法知道请求为什么慢。

```
关键队列指标:
────────────

延迟指标 (Latency):
├── queue_wait_time: 请求在队列中的等待时间
├── queue_wait_time_p50/p95/p99: 等待时间分布
├── e2e_latency: 端到端延迟 (用户感知)
└── processing_time: 实际处理时间

吞吐指标 (Throughput):
├── enqueue_rate: 入队速率 (QPS)
├── dequeue_rate: 出队速率 (QPS)
├── queue_growth_rate: 队列增长速率 (= 入队 - 出队)
└── consumer_utilization: 消费者利用率

深度指标 (Depth):
├── queue_depth: 当前队列深度 (绝对值)
├── queue_depth_pct: 队列容量百分比
├── queue_overflow_count: 溢出次数
└── queue_aging_distribution: 队列中不同等待时间的请求分布

错误指标 (Errors):
├── llm_429_rate: LLM API 限流频率
├── task_failure_rate: 任务失败率
├── task_timeout_rate: 任务超时率
└── dead_letter_queue_depth: 死信队列深度

健康指标 (Health):
├── queue_health_score: 综合健康评分
├── sla_breach_rate: SLA 违反率
└── circuit_breaker_state: 熔断器状态
```

```python
class QueueTelemetry:
    """队列可观测性采集器 — 用于 Prometheus + Grafana"""
    
    def __init__(self, registry=None):
        # 使用 Prometheus client 采集
        from prometheus_client import Counter, Histogram, Gauge
        
        self.enqueue_total = Counter(
            'agent_queue_enqueue_total', 'Total enqueued requests',
            ['priority', 'tenant']
        )
        
        self.dequeue_total = Counter(
            'agent_queue_dequeue_total', 'Total dequeued requests',
            ['status']  # success, failed, timeout, retry
        )
        
        self.queue_depth = Gauge(
            'agent_queue_depth', 'Current queue depth',
            ['queue_name', 'priority']
        )
        
        self.wait_time = Histogram(
            'agent_queue_wait_seconds', 'Request wait time in queue',
            buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0, 60.0]
        )
        
        self.processing_time = Histogram(
            'agent_processing_seconds', 'Request processing time',
            buckets=[0.5, 1.0, 2.0, 5.0, 10.0, 20.0, 30.0, 60.0]
        )
        
        self.active_workers = Gauge(
            'agent_active_workers', 'Number of active worker tasks'
        )
        
        self.circuit_state = Gauge(
            'agent_circuit_breaker_state',
            'Circuit breaker state (0=closed, 1=half-open, 2=open)'
        )
    
    def record_enqueue(self, priority: int = 1, tenant: str = "default"):
        self.enqueue_total.labels(priority=priority, tenant=tenant).inc()
    
    def record_wait(self, wait_time: float):
        self.wait_time.observe(wait_time)
    
    def record_processing(self, duration: float):
        self.processing_time.observe(duration)
    
    def update_queue_depth(self, depth: int, queue: str = "main", priority: str = "all"):
        self.queue_depth.labels(queue_name=queue, priority=priority).set(depth)
    
    def update_workers(self, count: int):
        self.active_workers.set(count)
```

```
Grafana 告警规则示例:
────────────────────

# 等待时间过长
- alert: QueueWaitTimeHigh
  expr: agent_queue_wait_seconds_p95 > 10
  for: 5m
  annotations:
    summary: "队列等待时间超过 10s (P95)"

# 队列深度快速增长
- alert: QueueDepthRapidGrowth  
  expr: rate(agent_queue_depth[5m]) > 100
  for: 2m
  annotations:
    summary: "队列深度快速增长 (> 100/s)"

# 死信队列堆积
- alert: DeadLetterQueueGrowing
  expr: agent_dead_letter_queue_depth > 100
  for: 5m
  annotations:
    summary: "死信队列堆积超过 100"

# 消费者空闲异常
- alert: ConsumerStarvation
  expr: agent_active_workers == 0 and agent_queue_depth > 0
  for: 1m
  annotations:
    summary: "消费者完全空闲但队列非空"
```

## 9. 适用场景判断

### 9.1 何时需要队列

不是每个 Agent 系统都需要队列。一个简单的评估框架：

```
队列必要性评估:
───────────────

标准 1: 突发比 (Burst Ratio)
  突发比 = 峰值 QPS / 平均 QPS
  
  < 2x: 突发很轻, 可能不需要队列
  2-5x: 建议使用队列
  > 5x: 强烈需要队列

标准 2: LLM API 限流频率
  每周 429 错误数:
  
  < 10: 速率限制还没成为问题
  10-100: 需要队列来平滑请求
  > 100: 队列是必须品

标准 3: 多租户隔离需求
  需要为不同租户提供隔离吗？
  
  否: 单一队列就够了
  是: 需要 per-tenant 队列架构

标准 4: 任务优先级
  不同请求有不同的重要性吗？
  
  否: FIFO 队列足够
  是: 需要优先级队列

评分矩阵:
─────────

维度              低分 (0 分)    中分 (1 分)    高分 (2 分)
────              ──────────    ──────────    ──────────
突发比            < 2x           2-5x           > 5x
429 频率/周       < 10           10-100         > 100
多租户            单租户          2-5 租户       > 5 租户
优先级            无              两级            三级+

总分:
0-2 分:                          不需要队列 → 同步处理即可
3-5 分:                          简单队列 → asyncio.Queue / Redis List
6-8 分:                          完整队列 → Redis Streams / RabbitMQ
8+ 分:                           高级队列 → Kafka / SQS + 优先级 + 多租户
```

### 9.2 队列技术选型矩阵

```
决策矩阵:
─────────

你的需求                     推荐技术                      理由
────────                     ────────                      ────

低延迟 < 5ms                 内存队列 / Redis List          无需持久化, 极致性能
  + 单进程                    asyncio.Queue                 
 
轻量持久化                    Redis Streams                 持久化 + 消费者组
  + 多 Worker                 + 可选的 Sorted Set           队列能力够用
  + 至少一次交付                                          运维简单

严格排序 + 重试               RabbitMQ                      原生死信队列
  + 复杂路由                  + 死信队列                     灵活的路由规则
  + TTL/延时                  + 延时消息插件

高吞吐 > 10万 QPS             Kafka / Pulsar                分区并行消费
  + 长时间回溯                 + 日志保留                     消息回放能力
  + 审计追踪                                             持久化可靠

无运维 (SaaS)                 AWS SQS                      零运维
  + 自动扩缩容                 / Google Pub/Sub              自动伸缩
  + 全球部署                                             全球可用

混合需求                      Redis Streams                  基础设施简单
  + 有限预算                   + 有限度的 Kafka             大部分场景够用
  + 中等规模                                             升级路径清晰
```

### 9.3 "无队列"架构

在某些场景下，队列带来的复杂度大于其价值。

```
无队列架构适用于:
─────────────────

1. 原型验证 / MVP 阶段
   └── < 100 用户, 单实例
   └── 代码简单, 快速迭代
   └── 这时候做队列属于"过早优化"
   └── asyncio.Queue 足够了

2. 内部工具 / 团队工具
   └── 同时在线用户 < 20
   └── 使用频率低 (每天几十次)
   └── 突发负载可控
   └── 用户能容忍偶尔的延迟

3. 批处理模式
   └── 所有请求都是非实时的
   └── "提交后等通知"模式
   └── 不需要交互式响应
   └── 数据库表本身就是队列

4. 无状态短任务
   └── 每个请求只需要 1 次 LLM 调用
   └── 没有多步链
   └── 延迟基本恒定 (没有 10x 波动)
   └── 直接请求 LLM API 即可

无队列架构方案:
────────────────

方案: asyncio.Queue + 信号量
┌────────────────────────────────────────────────┐
│                                                 │
│  用户请求 ──▶ asyncio.Queue (有界) ──▶ Worker Pool │
│                 │                     │          │
│                 │ maxsize=100        │ max_workers=10
│                 │                     │          │
│  溢出时: 503    │                     │ LLM API  │
│  (返回 503,     │                     │          │
│   建议稍后重试)  │                     │          │
└────────────────────────────────────────────────┘

优点:
├── 零外部依赖 (不需要 Redis / Kafka)
├── 延迟最低 (< 1μs 入队)
├── 代码简单 (纯 asyncio)
└── 运维成本为零

缺点:
├── 进程崩溃 → 所有排队请求丢失
├── 不能跨进程共享
├── 重启后队列消失
└── 扩展性受限 (单机资源上限)
```

### 9.4 生产环境推荐配置

```
不同规模的推荐架构:
──────────────────

规模 1: 创业期 (1-100 用户)
┌─────────────────────────────────────────────────────────┐
│  用户 ──▶ API 网关 ──▶ asyncio.Queue ──▶ Agent 实例      │
│                        (内存有界队列)                      │
│  扩容: 纵向扩展 (更好的 CPU/内存)                          │
│  队列: Python asyncio.Queue + 信号量                       │
│  持久化: 不需要                                            │
│  备份: 不需要                                              │
└─────────────────────────────────────────────────────────┘

规模 2: 成长期 (100-1K 用户)
┌─────────────────────────────────────────────────────────┐
│  用户 ──▶ 负载均衡 ──▶ FastAPI                         │
│                           │                              │
│                     ┌─────▼──────┐                      │
│                     │  Redis Queue │ ← 多实例共享队列     │
│                     │  (Stream)    │                      │
│                     └─────┬──────┘                      │
│                           │                              │
│                     ┌─────▼──────┐                      │
│                     │  Agent Pod ║ N                    │
│                     └────────────┘                      │
│  队列: Redis Stream + Sorted Set (优先级)                 │
│  持久化: Redis AOF (每秒)                                │
│  消费者组: Redis Consumer Group                          │
└─────────────────────────────────────────────────────────┘

规模 3: 规模化 (1K-100K 用户)
┌─────────────────────────────────────────────────────────┐
│  用户 ──▶ 网关 ──▶ 分类器 ──▶ P1 队列 ──▶ Agent 实例     │
│                                 P2 队列 ──▶ Agent 实例    │
│                                 P3 队列 ──▶ Batch 实例    │
│                                                            │
│  队列: Redis Streams (主) + Kafka (事件总线)                 │
│  优先级: 3 级 + Aging                                       │
│  多租户: Per-tenant 子队列                                   │
│  自动扩缩容: KEDA (基于队列深度)                               │
│  可观测性: Prometheus + Grafana                              │
│  熔断: Redis 深度 + 错误率联合熔断                             │
└─────────────────────────────────────────────────────────┘

规模 4: 全球化 (100K+ 用户)
┌─────────────────────────────────────────────────────────┐
│  区域 A:                                                  │
│  用户 ──▶ 区域网关 ──▶ 区域队列 ──▶ 区域 Agent 池           │
│                            │                              │
│  区域 B:                  │ 跨区域复制                      │
│  用户 ──▶ 区域网关 ──▶ 区域队列 ──▶ 区域 Agent 池           │
│                            │                              │
│  全局:                    ▼                              │
│                     ┌──────────────┐                      │
│                     │  Kafka 全局    │ ← 跨区域事件同步     │
│                     │  事件总线      │                      │
│                     └──────────────┘                      │
│                                                            │
│  区域队列: Redis Streams (低延迟)                            │
│  全局事件: Kafka (持久化, 跨区域复制)                         │
│  全局协调: 分布式配额 + 一致性哈希                             │
└─────────────────────────────────────────────────────────┘
```

## 总结

队列负载平滑是 Agent 生产系统的基础设施层核心组件。它不仅仅是一个"缓冲区"，而是请求的流控中心、优先级仲裁器和压力转换器。

```
核心要点回顾:
────────────

1. Agent 负载天生突发: 用户行为 × LLM 延迟波动 × 链式调用 = 10-50x 突发
2. 无队列 = 双重打击: Agent 实例过载 + LLM API 429 同时发生
3. 优先级 + 老化 = 公平而不饥饿: P1-P3 三级 + Aging 防止低优先级饿死
4. 多维度队列: Per-tenant + Per-model + Per-task-type 混合架构
5. 背压是必须的: 队列不能无限深, 必须告诉上游"慢下来"
6. 可观测性是基础: 队列深度 ≠ 健康度, 需要多维指标
7. 选型要看规模: 创业期 asyncio.Queue, 成长期 Redis, 规模化 Kafka
8. 给用户的承诺: 队列增加延迟, 但避免了系统崩溃和大规模 503

最终建议:
├── 从小做起: 先用 asyncio.Queue, 有需求再升级到 Redis
├── 先做优先级: 两级优先级 (交互式 vs 后台) 立竿见影
├── 监控先行: 没有可观测性的队列是黑箱
├── 设置超时: 队列中的每个请求都应该有 TTL
└── 预留降级: 队列满时不是只有"拒绝", 还有"简化处理"
```
