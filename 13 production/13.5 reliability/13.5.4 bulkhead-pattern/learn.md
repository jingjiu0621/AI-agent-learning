# Bulkhead Pattern —— LLM Agent 系统的隔舱模式

> **核心挑战**：Agent 系统中一个高负载的请求可能消耗大量资源——长时间的 LLM 调用、大量的工具执行、膨胀的上下文窗口——如果不做资源隔离，一个"嘈杂邻居" (Noisy Neighbor) 可能会耗尽共享资源池，导致其他完全正常的请求也被拖垮。隔舱模式的核心思想是将资源划分为独立的隔离区域，确保一个舱室的故障不会蔓延到其他舱室。

---

## 1. 基本原理

### 1.1 什么是隔舱模式

隔舱模式的名字来自船舶设计——船体被分为多个独立的水密隔舱，一个舱室进水不会导致整艘船沉没：

```
船舶隔舱:                          Agent 隔舱:

┌────┬────┬────┬────┬────┐        ┌────────────────────┐
│    │    │    │    │    │        │ Tenant A 隔舱       │
│ 水密│ 水密│ 水密│ 水密│ 水密│        │ 连接池: 10         │
│ 隔舱│ 隔舱│ 隔舱│ 隔舱│ 隔舱│        │ 并发: 5           │
│    │    │    │    │    │        │ 队列: 50           │
└────┴────┴────┴────┴────┘        └────────────────────┘
  ↑    ↑    ↑    ↑    ↑           ┌────────────────────┐
 一个  其他  继续                  │ Tenant B 隔舱       │
 舱室  舱室  航行                  │ 连接池: 20         │
 进水  不受                   │ 并发: 10          │
      影响                  │ 队列: 100          │
                                └────────────────────┘
```

### 1.2 为什么 Agent 需要隔舱

Agent 系统的资源消耗模式与传统 Web 服务有本质差异：

```
传统 Web 服务:
  请求 A: CPU 10ms + IO 20ms = 30ms
  请求 B: CPU 10ms + IO 20ms = 30ms
  资源消耗:  可预测 + 均匀分布

Agent 系统:
  请求 A: LLM 2s + 5次工具调用 + 10K Token = 高消耗
  请求 B: LLM 0.5s + 1次工具调用 + 2K Token = 低消耗
  请求 C: LLM 循环 + 15次工具调用 + 50K Token = 爆炸性消耗
  资源消耗:  不可预测 + 长尾分布

没有隔舱的后果:
  请求 C (高消耗) 占用了所有 LLM API 连接池 → 请求 A 和 B 排队等待 → 全部变慢
  请求 C 的 Token 消耗占满了上下文预算 → 其他请求无上下文可用
  请求 C 的长时间执行阻塞了工作线程 → 其他请求无法调度
```

### 1.3 隔舱的维度

Agent 系统可以在多个维度上实施隔舱：

```
资源维度:
  ┌─ 连接池: LLM API 连接、数据库连接、工具 API 连接
  ├─ 线程池/协程池: Agent 执行线程、LLM 调用线程
  ├─ 内存: 上下文窗口、记忆缓存、中间结果
  ├─ Token 配额: 输入 Token、输出 Token、总 Token 预算
  └─ 速率: API 调用频率、工具调用频率

租户/任务维度:
  ┌─ Tenant: 每个客户独立资源池
  ├─ 任务类型: 简单问答 vs 复杂研究 (不同资源需求)
  ├─ 优先级: VIP 用户 vs 免费用户 (不同资源分配)
  └─ 模型: GPT-4 池 vs GPT-4o-mini 池 (不同成本结构)
```

---

## 2. 核心技术与实现

### 2.1 线程/协程隔舱

最基本的隔舱：限制每个舱室的最大并发数。

```python
import asyncio
from dataclasses import dataclass, field
from typing import Optional
import time

@dataclass
class BulkheadConfig:
    """隔舱配置"""
    max_concurrent: int = 10          # 最大并发数
    max_queue_size: int = 100         # 最大队列长度
    queue_timeout: float = 30.0       # 队列等待超时

class Bulkhead:
    """
    隔舱实现: 使用信号量控制并发, 使用队列控制等待。

    核心机制:
    - 当并发数 < max_concurrent: 请求立即执行
    - 当并发数 >= max_concurrent: 请求进入队列等待
    - 当队列满: 请求被拒绝 (快速失败)
    """

    def __init__(self, name: str, config: BulkheadConfig):
        self.name = name
        self.config = config
        self.semaphore = asyncio.Semaphore(config.max_concurrent)
        self.queue: asyncio.Queue = asyncio.Queue(maxsize=config.max_queue_size)
        self.active_count = 0
        self.queued_count = 0
        self.rejected_count = 0

    async def execute(self, fn, timeout: Optional[float] = None):
        """
        在隔舱内执行函数。

        如果当前并发未满, 立即执行。
        如果并发已满, 进入队列等待。
        如果队列已满, 拒绝请求。
        """
        # 尝试进入队列
        if self.active_count >= self.config.max_concurrent:
            try:
                self.queued_count += 1
                await asyncio.wait_for(
                    self.queue.get(),
                    timeout=timeout or self.config.queue_timeout,
                )
            except asyncio.TimeoutError:
                self.queued_count -= 1
                self.rejected_count += 1
                raise BulkheadFullError(
                    f"Bulkhead '{self.name}' queue full after waiting"
                )

        async with self.semaphore:
            self.active_count += 1
            self.queued_count -= 1 if self.queued_count > 0 else 0
            try:
                result = await fn()
                return result
            finally:
                self.active_count -= 1
                # 唤醒队列中等待的下一个请求
                if self.queue.qsize() > 0:
                    self.queue.put_nowait(None)

    @property
    def utilization(self) -> float:
        """当前隔舱利用率"""
        return self.active_count / self.config.max_concurrent


class BulkheadFullError(Exception):
    """隔舱已满异常"""
    pass
```

### 2.2 Agent 专用隔舱系统

Agent 系统需要多维度隔舱管理：

```python
class AgentBulkheadManager:
    """
    Agent 隔舱管理器。

    管理多个维度的隔舱:
    - 按 Tenant 隔离: 每个客户独立资源
    - 按任务类型隔离: 简单/复杂任务不同资源
    - 按模型隔离: GPT-4 / GPT-4o-mini 不同连接池
    - 按工具隔离: 每个工具独立连接池
    """

    def __init__(self):
        # 按 Tenant 隔离
        self.tenant_bulkheads: dict[str, Bulkhead] = {}
        # 按任务类型隔离
        self.task_bulkheads: dict[str, Bulkhead] = {}
        # 按 Provider/模型隔离
        self.model_bulkheads: dict[str, Bulkhead] = {}
        # 全局默认隔舱
        self.default_bulkhead = Bulkhead(
            "default",
            BulkheadConfig(max_concurrent=20, max_queue_size=200),
        )

    def register_tenant(
        self, tenant_id: str, config: Optional[BulkheadConfig] = None
    ):
        """注册 Tenant 隔舱"""
        self.tenant_bulkheads[tenant_id] = Bulkhead(
            f"tenant:{tenant_id}",
            config or BulkheadConfig(max_concurrent=5, max_queue_size=50),
        )

    def register_task_type(
        self, task_type: str, config: Optional[BulkheadConfig] = None
    ):
        """注册任务类型隔舱"""
        self.task_bulkheads[task_type] = Bulkhead(
            f"task:{task_type}",
            config or BulkheadConfig(max_concurrent=10, max_queue_size=100),
        )

    def register_model(
        self, model: str, config: Optional[BulkheadConfig] = None
    ):
        """注册模型隔舱"""
        self.model_bulkheads[model] = Bulkhead(
            f"model:{model}",
            config or BulkheadConfig(max_concurrent=8, max_queue_size=80),
        )

    async def execute(
        self,
        fn,
        tenant_id: Optional[str] = None,
        task_type: Optional[str] = None,
        model: Optional[str] = None,
    ):
        """
        在多层隔舱中执行。

        请求必须通过所有相关隔舱才能执行。
        这确保了即使某个 Tenant 满载, 也不会影响其他 Tenant 的资源。
        """
        bulkheads = []

        # 收集所有相关隔舱
        if tenant_id and tenant_id in self.tenant_bulkheads:
            bulkheads.append(("tenant", self.tenant_bulkheads[tenant_id]))
        if task_type and task_type in self.task_bulkheads:
            bulkheads.append(("task", self.task_bulkheads[task_type]))
        if model and model in self.model_bulkheads:
            bulkheads.append(("model", self.model_bulkheads[model]))

        if not bulkheads:
            return await self.default_bulkhead.execute(fn)

        # 构建嵌套执行链: 外层 ⇒ 内层 ⇒ 实际执行
        async def chain(index: int = 0):
            if index >= len(bulkheads):
                return await fn()
            name, bh = bulkheads[index]
            return await bh.execute(lambda: chain(index + 1))

        return await chain()

    def get_status(self) -> dict:
        """获取所有隔舱的状态报告"""
        return {
            "tenants": {
                tid: {
                    "active": bh.active_count,
                    "queued": bh.queued_count,
                    "rejected": bh.rejected_count,
                    "utilization": bh.utilization,
                    "max_concurrent": bh.config.max_concurrent,
                }
                for tid, bh in self.tenant_bulkheads.items()
            },
            "tasks": {
                tt: {
                    "active": bh.active_count,
                    "queued": bh.queued_count,
                    "utilization": bh.utilization,
                }
                for tt, bh in self.task_bulkheads.items()
            },
            "models": {
                m: {
                    "active": bh.active_count,
                    "queued": bh.queued_count,
                    "utilization": bh.utilization,
                }
                for m, bh in self.model_bulkheads.items()
            },
        }
```

### 2.3 连接池隔舱

为 LLM API 连接和工具 API 连接做单独隔离：

```python
class ConnectionPoolBulkhead:
    """
    连接池隔舱。

    确保每个下游服务 (LLM Provider / 工具 API) 的连接池独立,
    一个服务的连接问题不会影响其他服务。
    """

    def __init__(self):
        self.pools: dict[str, asyncio.Queue] = {}
        self.pool_configs: dict[str, tuple[int, int]] = {}  # (min, max)

    def register_service(
        self, service: str, min_connections: int = 2, max_connections: int = 10
    ):
        """注册服务连接池"""
        self.pool_configs[service] = (min_connections, max_connections)
        # 初始化最小连接数
        self.pools[service] = asyncio.Queue(maxsize=max_connections)
        for _ in range(min_connections):
            self.pools[service].put_nowait(None)

    async def acquire(self, service: str, timeout: float = 5.0) -> bool:
        """获取连接"""
        pool = self.pools.get(service)
        if not pool:
            raise ValueError(f"Unknown service: {service}")

        try:
            await asyncio.wait_for(pool.get(), timeout=timeout)
            return True
        except asyncio.TimeoutError:
            return False

    def release(self, service: str):
        """释放连接"""
        pool = self.pools.get(service)
        if pool:
            pool.put_nowait(None)

    async def execute_with_connection(
        self, service: str, fn, timeout: float = 5.0
    ):
        """获取连接 → 执行 → 释放连接"""
        acquired = await self.acquire(service, timeout)
        if not acquired:
            raise ConnectionTimeoutError(
                f"No available connection for '{service}'"
            )
        try:
            return await fn()
        finally:
            self.release(service)


class ConnectionTimeoutError(Exception):
    pass
```

### 2.4 Token 预算隔舱

Agent 特有：按隔舱分配 Token 预算，防止一个请求消耗所有 Token：

```python
class TokenBudgetBulkhead:
    """
    Token 预算隔舱。

    按 Tenant / 任务类型分配 Token 配额。
    每个舱室有独立的输入/输出 Token 预算,
    超过预算的请求被降级或拒绝。
    """

    def __init__(self):
        self.budgets: dict[str, dict] = {}

    def register_budget(
        self,
        compartment: str,
        max_input_tokens: int = 100000,
        max_output_tokens: int = 50000,
        max_total_tokens: int = 500000,
        reset_interval: float = 3600.0,  # 每小时重置
    ):
        """注册 Token 预算"""
        self.budgets[compartment] = {
            "max_input": max_input_tokens,
            "max_output": max_output_tokens,
            "max_total": max_total_tokens,
            "used_input": 0,
            "used_output": 0,
            "used_total": 0,
            "reset_interval": reset_interval,
            "last_reset": time.time(),
        }

    def check_budget(
        self,
        compartment: str,
        estimated_input: int,
        estimated_output: int,
    ) -> tuple[bool, str]:
        """
        检查是否有足够的 Token 预算。

        Returns:
            (是否通过, 失败原因)
        """
        budget = self.budgets.get(compartment)
        if not budget:
            return True, ""

        self._maybe_reset(compartment)

        if budget["used_total"] + estimated_input + estimated_output > budget["max_total"]:
            return False, f"Total token budget exceeded for '{compartment}'"

        if budget["used_input"] + estimated_input > budget["max_input"]:
            return False, f"Input token budget exceeded for '{compartment}'"

        return True, ""

    def consume(self, compartment: str, input_tokens: int, output_tokens: int):
        """消耗 Token 配额"""
        budget = self.budgets.get(compartment)
        if budget:
            budget["used_input"] += input_tokens
            budget["used_output"] += output_tokens
            budget["used_total"] += input_tokens + output_tokens

    def _maybe_reset(self, compartment: str):
        """定期重置配额"""
        budget = self.budgets[compartment]
        if time.time() - budget["last_reset"] >= budget["reset_interval"]:
            budget["used_input"] = 0
            budget["used_output"] = 0
            budget["used_total"] = 0
            budget["last_reset"] = time.time()

    def get_usage(self, compartment: str) -> dict:
        """获取配额使用情况"""
        budget = self.budgets.get(compartment, {})
        if not budget:
            return {}
        return {
            "input_usage": budget["used_input"] / budget["max_input"],
            "output_usage": budget["used_output"] / budget["max_output"],
            "total_usage": budget["used_total"] / budget["max_total"],
            "input_remaining": budget["max_input"] - budget["used_input"],
            "output_remaining": budget["max_output"] - budget["used_output"],
        }
```

### 2.5 队列隔舱

当并发已满时，不同舱室的请求需要独立的等待队列：

```python
class PriorityBulkheadQueue:
    """
    带优先级的隔舱队列。

    高优先级请求在队列中插队, 确保 VIP 用户不受免费用户流量影响。
    """

    def __init__(self, max_size: int = 100):
        self.max_size = max_size
        # 三个优先级队列
        self.high: asyncio.Queue = asyncio.Queue(maxsize=max_size)
        self.medium: asyncio.Queue = asyncio.Queue(maxsize=max_size)
        self.low: asyncio.Queue = asyncio.Queue(maxsize=max_size)

    async def enqueue(self, item, priority: int = 0):
        """
        入队。
        priority: 0=low, 1=medium, 2=high
        """
        queues = {0: self.low, 1: self.medium, 2: self.high}
        q = queues.get(priority, self.medium)
        await q.put(item)

    async def dequeue(self):
        """
        出队: 按优先级从高到低取。
        """
        for q in [self.high, self.medium, self.low]:
            if q.qsize() > 0:
                return await q.get()
        # 所有队列空
        raise asyncio.QueueEmpty()

    def qsize(self) -> int:
        return self.high.qsize() + self.medium.qsize() + self.low.qsize()
```

---

## 3. 隔舱配置指南

不同场景需要不同的隔舱配置：

| 场景 | 隔舱维度 | 推荐配置 | 说明 |
|------|---------|---------|------|
| 单租户 Agent | 任务类型 | 简单=10并发, 复杂=3并发 | 防止复杂任务阻塞简单任务 |
| 多租户 SaaS | Tenant | 每租户 5 并发 | 一个租户的流量爆发不影响其他租户 |
| 多模型路由 | 模型 | GPT-4=5并发, GPT-4o-mini=20并发 | 昂贵模型限制并发, 便宜模型放开 |
| 多工具 Agent | 工具 | 每工具 3 连接 | 一个第三方 API 故障不影响其他工具 |
| 生产级系统 | 全维度 | Tenant+任务+模型+工具 | 最全面的隔离 |

### 3.1 隔舱大小计算

```
隔舱大小的确定因素:

  1. 期望的并发数: 该舱室预期的最大并发请求数
  2. 平均响应时间: LLM 调用 + 工具调用的平均延迟
  3. 可接受的排队时间: 用户能等多长时间
  4. 资源限制: LLM API 配额、连接数限制、内存限制

  计算公式 (Little's Law):
    并发数 = 吞吐量 × 平均响应时间

  示例:
    吞吐量: 10 请求/秒
    平均响应时间: 3 秒 (LLM 调用)
    必要并发: 10 × 3 = 30

    如果限制并发为 10, 则:
    最大吞吐量 = 10 / 3 ≈ 3.3 请求/秒
    超出部分排队或拒绝
```

### 3.2 隔舱耗尽后的行为

当隔舱资源耗尽时，系统应表现出可控的行为：

```
隔舱满载时的策略:

  1. 排队等待 (默认)
     - 优点: 请求不会丢失
     - 缺点: 延迟增加
     - 适用: 交互式 Agent (用户可等待)

  2. 快速拒绝
     - 优点: 立即返回错误, 不浪费资源
     - 缺点: 请求丢失
     - 适用: 实时 Agent (不能等待)

  3. 降级执行
     - 优点: 请求仍被处理, 只是能力降级
     - 缺点: 实现复杂
     - 适用: 关键业务 (宁可降级不可拒绝)

  4. 异步排队
     - 优点: 请求被异步处理, 结果回调
     - 缺点: 需要回调机制
     - 适用: 非实时 Agent (如报告生成)
```

---

## 4. 能力边界

### 4.1 隔舱模式的局限

```
隔舱模式做不到:
  ┌─ 减少总资源消耗 (只做隔离, 不做节省)
  ├─ 防止所有类型的故障 (如 LLM 语义错误)
  ├─ 完全消除"嘈杂邻居"效应 (只能限制影响范围)
  └─ 替代限流 (隔舱和限流是互补关系, 不是替代关系)
```

### 4.2 隔舱 vs 其他模式

| 模式 | 解决问题的角度 | 与隔舱的关系 |
|------|--------------|-------------|
| Rate Limiting | 控制请求速率 | 互补: 限流限制总量, 隔舱隔离内部 |
| Circuit Breaker | 快速失败防止级联 | 互补: CB 防止重试风暴, 隔舱防止资源争用 |
| Bulkhead | 资源隔离 | 核心模式 |
| Queue | 请求缓冲 | 隔舱通常内置队列 |
| Timeout | 防止无限等待 | 隔舱中需要超时来释放资源 |

---

## 5. 工程优化方向

### 5.1 动态隔舱

静态的隔舱大小不能适应所有场景。动态隔舱根据实时负载自动调整：

```python
class DynamicBulkhead(Bulkhead):
    """
    动态隔舱: 根据负载和使用率自动调整并发数。
    """

    def __init__(self, name: str, config: BulkheadConfig):
        super().__init__(name, config)
        self.min_concurrent = max(1, config.max_concurrent // 4)
        self.max_concurrent = config.max_concurrent * 2
        self.usage_history: list[float] = []
        self.last_adjustment = time.time()

    async def adjust(self):
        """每 60 秒根据使用率调整并发上限"""
        if time.time() - self.last_adjustment < 60:
            return

        usage = self.utilization
        self.usage_history.append(usage)
        if len(self.usage_history) > 10:
            self.usage_history.pop(0)

        avg_usage = sum(self.usage_history) / len(self.usage_history)

        if avg_usage > 0.8 and self.config.max_concurrent < self.max_concurrent:
            # 持续高负载 → 扩容
            self.config.max_concurrent = min(
                self.max_concurrent,
                int(self.config.max_concurrent * 1.5),
            )
        elif avg_usage < 0.2 and self.config.max_concurrent > self.min_concurrent:
            # 持续低负载 → 缩容
            self.config.max_concurrent = max(
                self.min_concurrent,
                int(self.config.max_concurrent * 0.8),
            )

        self.last_adjustment = time.time()
```

### 5.2 监控指标

```
隔舱监控:
  ├── 使用率: active / max_concurrent (目标 < 70%)
  ├── 队列深度: 当前排队请求数
  ├── 排队时间: 请求在队列中的平均等待时间
  ├── 拒绝率: 因隔舱满被拒绝的请求比例 (目标 < 1%)
  ├── 每舱室吞吐量: 各舱室完成的请求数
  └── 舱室间不平衡度: 最忙/最闲舱室的差异
```

### 5.3 实践检查清单

```
[ ] 1. 每个外部依赖 (LLM Provider / 工具) 有独立连接池
[ ] 2. 多租户场景按 Tenant 分配独立资源配额
[ ] 3. 不同任务类型使用不同的并发限制
[ ] 4. 隔舱队列有大小限制和超时机制
[ ] 5. Token 预算按舱室分配
[ ] 6. 隔舱已满时有明确的降级/拒绝行为
[ ] 7. 高优先级请求可插队 (VIP 通道)
[ ] 8. 动态调整隔舱大小 (根据负载自动扩缩)
[ ] 9. 隔舱状态可观测 (Prometheus metrics)
[ ] 10. 定期测试隔舱隔离效果 (压力测试)
```

---

> **总结**：隔舱模式是防止"嘈杂邻居"问题的关键手段。在 Agent 系统中，隔舱可以在多个维度实施：按 Tenant 隔离（多租户场景）、按任务类型隔离（简单 vs 复杂任务）、按模型隔离（大/小模型不同的连接池）、按工具隔离（每个外部 API 独立连接池）。核心设计要点是：(1) 每个关键资源共享独立的隔舱；(2) 隔舱满时的降级行为要明确；(3) 隔舱大小应动态调整；(4) 与限流、熔断、超时等模式协同运作。
