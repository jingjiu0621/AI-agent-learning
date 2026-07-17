# Circuit Breaker —— LLM Agent 系统的熔断器模式

> **核心挑战**：Agent 系统的依赖链比传统应用更长且更脆弱：一次 Agent 请求可能调用 3-10 次 LLM API，每次 LLM 调用又可能触发多次工具调用。如果某个依赖（如 LLM Provider、外部 API）持续故障，不做防护的 Agent 系统会在数秒内产生大量重试，导致级联故障、成本爆炸和下游系统过载。熔断器的核心任务是**快速失败**，在依赖不可用时及时刹车，防止故障蔓延到整个系统。

---

## 1. 基本原理

### 1.1 熔断器状态机

经典的熔断器有三个状态，Agent 场景下增加两个扩展状态以适应 LLM 特有的灰度恢复需求：

```
                   ┌──────────────────┐
                   │                  │
                   │    CLOSED        │◄────────── 正常状态: 请求通过
                   │                  │           失败计数: 0
                   └────────┬─────────┘
                            │ 连续失败数 > 阈值
                            ▼
                   ┌──────────────────┐
          ┌───────│                  │────────┐
          │       │    OPEN          │        │
          │       │  (熔断开启)       │        │ 拒绝所有请求
          │       └────────┬─────────┘        │ 快速返回错误
          │                │ 超时后进入半开     │
          │                ▼                  │
          │       ┌──────────────────┐        │
          │       │   HALF_OPEN      │        │
          │       │  (半开探测)       │────────┘ 尝试放行少量请求
          │       └────────┬─────────┘              探测依赖是否恢复
          │                │
          │     成功 ┌─────┴─────┐ 失败
          │         ▼           ▼
          │  ┌──────────┐ ┌──────────┐
          │  │  CLOSED  │ │  OPEN    │
          │  │ (恢复)    │ │ (再次熔断)│
          │  └──────────┘ └──────────┘
          │
          │  Agent 扩展状态:
          │
          │       ┌──────────────────┐
          │       │  DEGRADED        │  ← Agent 特有: 熔断但使用降级
          │       │  (降级熔断)       │     不返回错误, 返回降级内容
          │       └──────────────────┘
          │
          │       ┌──────────────────┐
          │       │  PERMA_OPEN      │  ← Agent 特有: 人工介入
          │       │  (永久熔断)       │     需要运维确认后才能恢复
          │       └──────────────────┘
          │
          └─────── 恢复路径:
                    CLOSED ← HALF_OPEN ← OPEN
                    CLOSED ← 人工恢复 ← PERMA_OPEN
```

### 1.2 为什么 Agent 需要熔断器

没有熔断器的 Agent 系统在依赖故障时的表现：

```
时间线 - 无熔断器:

  T=0:   LLM API 开始返回 503
  T=1:   Agent A 发起 LLM 调用 → 失败 → 重试 → 失败 → 重试
  T=2:   Agent B 发起 LLM 调用 → 失败 → 重试 → 失败 → 重试
  T=3:   100 个 Agent 同时在重试 → API 压力倍增
  T=4:   重试风暴 → Provider 降级更多请求
  T=5:   每个 Agent 都重试 3 次 → 300 次无效调用 → 成本 $15+
  T=6:   错误传播到上游 → 用户请求超时 → 客户端也重试 → 恶性循环

时间线 - 有熔断器:

  T=0:   LLM API 开始返回 503
  T=1:   Agent A 调用失败 → 熔断器计数 +1
  T=2:   Agent B 调用失败 → 熔断器计数 +2
  T=3:   连续 5 次失败 → 熔断器 OPEN → 后续请求立刻返回"服务不可用"
  T=4:   后续 95 个请求无需等待, 直接获得降级响应
  T=5:   5 分钟后 HALF_OPEN → 试探请求成功 → CLOSED → 恢复正常
  T=6:   无效调用: 5 次 → 成本 < $0.25
```

---

## 2. 核心技术与实现

### 2.1 通用熔断器

```python
import time
import asyncio
from enum import Enum
from typing import Optional, Callable
from dataclasses import dataclass

class CircuitState(Enum):
    CLOSED = "closed"           # 正常
    OPEN = "open"               # 熔断
    HALF_OPEN = "half_open"     # 半开探测
    DEGRADED = "degraded"       # 降级熔断 (Agent 扩展)
    PERMA_OPEN = "perma_open"   # 永久熔断 (需人工)

@dataclass
class CircuitBreakerConfig:
    """熔断器配置"""
    failure_threshold: int = 5          # 连续失败 N 次后开启熔断
    recovery_timeout: float = 30.0      # 熔断持续 N 秒后进入半开
    half_open_max_requests: int = 3     # 半开状态允许的最大请求数
    success_threshold: int = 2          # 半开状态下连续成功 N 次后关闭
    degraded_fallback: Optional[str] = None  # 降级时的默认响应

class CircuitBreaker:
    """
    状态机熔断器实现。
    跟踪调用结果, 在失败达到阈值时快速拒绝请求。
    """

    def __init__(self, config: CircuitBreakerConfig):
        self.config = config
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = 0.0
        self.half_open_requests = 0
        self.last_state_change = time.time()

    async def call(self, fn: Callable, fallback: Optional[Callable] = None):
        """
        执行受熔断器保护的调用。

        Args:
            fn: 实际要执行的函数
            fallback: 熔断时的降级函数
        """
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time >= self.config.recovery_timeout:
                self._enter_half_open()
            else:
                return await self._handle_open(fallback)

        if self.state == CircuitState.HALF_OPEN:
            if self.half_open_requests >= self.config.half_open_max_requests:
                return await self._handle_open(fallback)
            self.half_open_requests += 1

        try:
            result = await fn()
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            if fallback:
                return await fallback()
            raise

    def _on_success(self):
        self.failure_count = 0
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.config.success_threshold:
                self._enter_closed()

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.state == CircuitState.HALF_OPEN:
            self._enter_open()
        elif self.failure_count >= self.config.failure_threshold:
            self._enter_open()

    def _enter_open(self):
        self.state = CircuitState.OPEN
        self.last_failure_time = time.time()
        self.half_open_requests = 0

    def _enter_half_open(self):
        self.state = CircuitState.HALF_OPEN
        self.success_count = 0
        self.half_open_requests = 0

    def _enter_closed(self):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0

    async def _handle_open(self, fallback: Optional[Callable]):
        """熔断开启时的处理"""
        if fallback:
            return await fallback()
        raise CircuitBreakerOpenError(
            f"Circuit breaker is OPEN. "
            f"Will retry in {self.config.recovery_timeout - (time.time() - self.last_failure_time):.0f}s"
        )


class CircuitBreakerOpenError(Exception):
    """熔断器开启异常"""
    pass
```

### 2.2 Agent 级熔断器

Agent 场景需要多层次的熔断——从单次 LLM 调用级别到整个 Agent 任务级别：

```python
class AgentCircuitBreaker:
    """
    Agent 专用的分层熔断器。

    管理三类熔断:
      1. LLM Provider 级别 (整个 API 服务)
      2. 工具级别 (特定外部 API)
      3. Agent 任务级别 (当前执行的任务)
    """

    def __init__(self):
        # 每个 Provider 独立熔断
        self.provider_breakers: dict[str, CircuitBreaker] = {}
        # 每个工具独立熔断
        self.tool_breakers: dict[str, CircuitBreaker] = {}
        # 全局熔断 (所有 Provider 熔断时触发)
        self.global_breaker = CircuitBreaker(
            CircuitBreakerConfig(
                failure_threshold=50,      # 全局阈值更高
                recovery_timeout=60.0,
            )
        )

    def register_provider(self, name: str, config: Optional[CircuitBreakerConfig] = None):
        """注册 Provider 熔断器"""
        self.provider_breakers[name] = CircuitBreaker(
            config or CircuitBreakerConfig(failure_threshold=5, recovery_timeout=30.0)
        )

    def register_tool(self, name: str, config: Optional[CircuitBreakerConfig] = None):
        """注册工具熔断器"""
        self.tool_breakers[name] = CircuitBreaker(
            config or CircuitBreakerConfig(failure_threshold=3, recovery_timeout=15.0)
        )

    async def call_llm(
        self,
        provider: str,
        fn: Callable,
        fallback_provider: Optional[str] = None,
    ) -> dict:
        """
        调用 LLM API (带 Provider 级熔断)。

        如果当前 Provider 熔断, 尝试备用 Provider。
        所有 Provider 都熔断时触发全局熔断。
        """
        breaker = self.provider_breakers.get(provider)
        if not breaker:
            raise ValueError(f"Unknown provider: {provider}")

        try:
            return await breaker.call(fn)
        except CircuitBreakerOpenError:
            if fallback_provider and fallback_provider in self.provider_breakers:
                return await self.call_llm(fallback_provider, fn)
            # 所有 Provider 熔断
            await self.global_breaker._on_failure()
            raise

    async def call_tool(
        self,
        tool_name: str,
        fn: Callable,
        fallback_fn: Optional[Callable] = None,
    ) -> dict:
        """
        调用工具 (带工具级熔断)。
        """
        breaker = self.tool_breakers.get(tool_name)
        if not breaker:
            raise ValueError(f"Unknown tool: {tool_name}")

        return await breaker.call(fn, fallback_fn)

    def is_llm_available(self, provider: str) -> bool:
        """快速检查 Provider 是否可用 (不做实际调用)"""
        breaker = self.provider_breakers.get(provider)
        if not breaker:
            return False
        return breaker.state in (CircuitState.CLOSED, CircuitState.HALF_OPEN)

    def any_llm_available(self) -> bool:
        """是否有任意 Provider 可用"""
        return any(
            b.state in (CircuitState.CLOSED, CircuitState.HALF_OPEN)
            for b in self.provider_breakers.values()
        )

    def get_status_report(self) -> dict:
        """获取熔断器状态报告"""
        return {
            "providers": {
                name: {
                    "state": b.state.value,
                    "failure_count": b.failure_count,
                }
                for name, b in self.provider_breakers.items()
            },
            "tools": {
                name: {
                    "state": b.state.value,
                    "failure_count": b.failure_count,
                }
                for name, b in self.tool_breakers.items()
            },
            "global": self.global_breaker.state.value,
        }
```

### 2.3 Agent 任务级熔断

除了基础设施级别，Agent 的"逻辑"也需要熔断——当 Agent 连续出现语义偏差时，应触发任务级熔断：

```python
class TaskLevelCircuitBreaker:
    """
    Agent 任务级熔断器。

    检测 Agent 行为异常 (循环、重复错误、语义偏差),
    在"逻辑层面"快速失败, 而不是让 Agent 继续无效执行。
    """

    def __init__(
        self,
        max_steps: int = 15,
        max_semantic_errors: int = 3,
        max_same_action: int = 3,
    ):
        self.max_steps = max_steps
        self.max_semantic_errors = max_semantic_errors
        self.max_same_action = max_same_action

        self.step_count = 0
        self.semantic_errors = 0
        self.action_history: list[tuple[str, str]] = []  # (tool, args_hash)
        self.thought_patterns: list[str] = []  # 最近的 thought 内容

    def record_step(self, thought: str, action: Optional[tuple[str, str]] = None):
        """记录 Agent 执行的一步"""
        self.step_count += 1
        self.thought_patterns.append(thought)

        if action:
            tool_name, args = action
            args_hash = str(hash(str(args)))
            self.action_history.append((tool_name, args_hash))

    def record_semantic_error(self, error: str):
        """记录语义错误"""
        self.semantic_errors += 1

    def check_trip(self) -> Optional[str]:
        """检查是否应该触发任务级熔断。返回熔断原因或 None。"""

        if self.step_count >= self.max_steps:
            return f"Max steps exceeded ({self.step_count} >= {self.max_steps})"

        if self.semantic_errors >= self.max_semantic_errors:
            return f"Too many semantic errors ({self.semantic_errors})"

        # 检测相同 Action 重复
        if len(self.action_history) >= self.max_same_action:
            recent = self.action_history[-self.max_same_action:]
            if all(a[0] == recent[0][0] for a in recent):
                return f"Repeated same action {self.max_same_action} times: {recent[0][0]}"

        # 检测 Thought 循环 (内容过于相似)
        if len(self.thought_patterns) >= 4:
            recent = self.thought_patterns[-4:]
            if len(set(recent)) <= 2:
                return "Agent appears to be in a thought loop"

        return None

    def reset(self):
        """重置任务级熔断器"""
        self.step_count = 0
        self.semantic_errors = 0
        self.action_history = []
        self.thought_patterns = []
```

### 2.4 自适应熔断阈值

固定的熔断阈值不能满足所有场景。好的熔断器应该根据实时数据动态调整：

```python
class AdaptiveCircuitBreaker(CircuitBreaker):
    """
    自适应熔断器: 根据动态数据调整熔断参数。
    """

    def __init__(self, config: CircuitBreakerConfig):
        super().__init__(config)
        self.recent_latencies: list[float] = []
        self.latency_baseline: Optional[float] = None
        self.failure_rate_window: list[bool] = []  # True=成功, False=失败
        self.adaptive_threshold = config.failure_threshold
        self.min_threshold = 3
        self.max_threshold = 20

    def record_latency(self, latency_ms: float):
        """记录请求延迟"""
        self.recent_latencies.append(latency_ms)
        if len(self.recent_latencies) > 100:
            self.recent_latencies.pop(0)
        # 计算 P50 基线
        if len(self.recent_latencies) >= 10:
            sorted_lat = sorted(self.recent_latencies)
            self.latency_baseline = sorted_lat[len(sorted_lat) // 2]

    def _on_failure(self):
        super()._on_failure()
        self.failure_rate_window.append(False)
        self._adapt_threshold()

    def _on_success(self):
        super()._on_success()
        self.failure_rate_window.append(True)
        # 只保留最近 100 条记录
        if len(self.failure_rate_window) > 100:
            self.failure_rate_window.pop(0)
        self._adapt_threshold()

    def _adapt_threshold(self):
        """
        根据失败率自适应调整阈值。

        规则:
        - 当总体失败率低但突然连续失败时 → 降低阈值 (更敏感)
        - 当总体失败率高但失败是分散的时 → 提高阈值 (减少误判)
        """
        if len(self.failure_rate_window) < 20:
            return

        # 总体失败率
        failures = sum(1 for r in self.failure_rate_window if not r)
        overall_failure_rate = failures / len(self.failure_rate_window)

        # 近期失败率 (最近 10 个)
        recent = self.failure_rate_window[-10:]
        recent_failures = sum(1 for r in recent if not r)
        recent_failure_rate = recent_failures / len(recent)

        if recent_failure_rate > 0.3 and overall_failure_rate < 0.05:
            # 突然大量失败, 降低阈值快速熔断
            self.adaptive_threshold = max(
                self.min_threshold,
                self.adaptive_threshold - 1
            )
        elif overall_failure_rate > 0.1 and recent_failure_rate > 0.3:
            # 系统性高失败率, 提高阈值避免误触
            self.adaptive_threshold = min(
                self.max_threshold,
                self.adaptive_threshold + 1
            )

    def get_dynamic_threshold(self) -> int:
        """获取当前自适应阈值"""
        return self.adaptive_threshold
```

### 2.5 熔断恢复策略

熔断器恢复比传统系统更复杂——LLM Provider 故障恢复往往不是瞬间的：

```python
class RecoveryStrategy(Enum):
    STANDARD = "standard"             # 标准: 超时后 HALF_OPEN
    GRADUAL = "gradual"               # 渐进: 逐步放量
    PROBE_BASED = "probe_based"       # 探测: 先发健康检查请求
    MANUAL = "manual"                 # 人工: 需要确认

class GradualRecoveryCircuitBreaker(CircuitBreaker):
    """
    渐进恢复熔断器。

    半开状态不分一次性试探, 而是逐步放开流量:
    10% → 25% → 50% → 100%
    每个阶段监控错误率, 错误率过高则回到 OPEN。
    """

    def __init__(self, config: CircuitBreakerConfig):
        super().__init__(config)
        self.recovery_level = 0.0  # 0.0 ~ 1.0
        self.request_count = 0
        self.error_count = 0
        self.recovery_stages = [0.1, 0.25, 0.5, 0.75, 1.0]

    def should_allow_request(self) -> bool:
        """判断是否应该允许请求通过"""
        if self.state != CircuitState.HALF_OPEN:
            return super().should_allow_request()

        # 渐进恢复: 只放行部分流量
        request_id = id(self)  # 简化示例
        hash_val = hash(f"{request_id}{self.request_count}")
        probability = hash_val % 10000 / 10000.0

        return probability < self.recovery_level

    def advance_recovery(self):
        """推进到下一个恢复阶段"""
        current_idx = self.recovery_stages.index(self.recovery_level) \
            if self.recovery_level in self.recovery_stages else -1
        next_idx = min(current_idx + 1, len(self.recovery_stages) - 1)
        self.recovery_level = self.recovery_stages[next_idx]
        self.error_count = 0

    def on_half_open_result(self, success: bool):
        """处理半开状态下的请求结果"""
        self.request_count += 1
        if not success:
            self.error_count += 1

        if self.error_count >= 2:
            # 错误率过高, 回到 OPEN
            self._enter_open()
        elif self.recovery_level >= 1.0 and self.request_count >= 20:
            # 100% 流量且稳定 → CLOSED
            self._enter_closed()
        elif self.request_count >= 50:
            # 当前级别稳定 → 推进到下一阶段
            self.advance_recovery()
            self.request_count = 0
```

---

## 3. Agent 特有熔断场景

### 3.1 Provider 熔断 + 自动切换

```python
class ProviderFailoverManager:
    """
    Provider 故障转移管理器。

    监控多个 LLM Provider 的健康状态,
    自动将流量从不健康的 Provider 转移到健康的 Provider。
    """

    def __init__(self, providers: dict[str, dict]):
        self.providers = providers
        self.breakers = {
            name: CircuitBreaker(CircuitBreakerConfig(
                failure_threshold=3,
                recovery_timeout=30.0,
            ))
            for name in providers
        }
        self.current_primary = list(providers.keys())[0]

    async def execute_llm_call(
        self, messages: list[dict], tools: list[dict] = None
    ) -> dict:
        """执行 LLM 调用, 带自动故障转移"""

        # 优先级排序: 当前 primary > 未熔断 > 半开 > 已熔断
        candidates = sorted(
            self.providers.keys(),
            key=lambda name: self._provider_score(name),
            reverse=True,
        )

        errors = []
        for provider_name in candidates:
            breaker = self.breakers[provider_name]
            if breaker.state == CircuitState.OPEN:
                continue

            try:
                result = await breaker.call(
                    lambda: self.providers[provider_name]["client"].invoke(
                        messages, tools
                    )
                )
                # 成功后更新 primary
                self.current_primary = provider_name
                return result
            except Exception as e:
                errors.append({provider_name: str(e)})
                continue

        raise AllProvidersFailedError(f"All providers failed: {errors}")

    def _provider_score(self, name: str) -> float:
        """
        计算 Provider 优先级分数。
        primary+50, CLOSED+30, HALF_OPEN+10, 成功率高再+20
        """
        breaker = self.breakers[name]
        score = 0.0
        if name == self.current_primary:
            score += 50
        if breaker.state == CircuitState.CLOSED:
            score += 30
        elif breaker.state == CircuitState.HALF_OPEN:
            score += 10
        return score
```

### 3.2 工具级熔断

Agent 的工具调用也需要熔断——特别是当外部 API 持续失败时：

```python
class ToolCircuitBreakerManager:
    """
    工具级熔断管理器。
    每个工具独立的熔断器, 避免一个工具的故障影响其他工具。
    """

    def __init__(self):
        self.breakers: dict[str, CircuitBreaker] = {}

    def register_tool(
        self, tool_name: str,
        failure_threshold: int = 5,
        recovery_timeout: float = 30.0,
    ):
        self.breakers[tool_name] = CircuitBreaker(
            CircuitBreakerConfig(
                failure_threshold=failure_threshold,
                recovery_timeout=recovery_timeout,
            )
        )

    async def execute_tool(self, tool_name: str, fn, fallback=None):
        """执行工具调用, 受熔断器保护"""
        breaker = self.breakers.get(tool_name)
        if not breaker:
            return await fn()

        try:
            return await breaker.call(fn)
        except CircuitBreakerOpenError:
            # 工具熔断 → 尝试降级
            if fallback:
                return await fallback()
            raise ToolNotAvailableError(f"Tool '{tool_name}' is circuit-broken")

    def on_tool_success(self, tool_name: str):
        """报告工具调用成功"""
        if tool_name in self.breakers:
            self.breakers[tool_name]._on_success()

    def on_tool_failure(self, tool_name: str):
        """报告工具调用失败"""
        if tool_name in self.breakers:
            self.breakers[tool_name]._on_failure()
```

---

## 4. 熔断器集成架构

完整地将熔断器集成到 Agent 系统中：

```
                                  ┌──────────────────────┐
                                  │   Agent Request       │
                                  └──────────┬───────────┘
                                             │
                                             ▼
                                  ┌──────────────────────┐
                                  │  Task Level Breaker   │
                                  │  步骤/循环/语义监控    │
                                  └──────────┬───────────┘
                                             │
                                             ▼
                                  ┌──────────────────────┐
                                  │  Global Breaker       │
                                  │  (所有 Provider 熔断)  │
                                  └──────────┬───────────┘
                                             │
                                    ┌────────┴────────┐
                                    ▼                  ▼
                           ┌────────────────┐  ┌────────────────┐
                           │ Provider A     │  │ Provider B     │
                           │ Breaker        │  │ Breaker        │
                           └────────┬───────┘  └────────┬───────┘
                                    │                   │
                                    ▼                   ▼
                           ┌──────────────────────────────────┐
                           │         Tool Breakers            │
                           │  ┌──────┐┌──────┐┌──────┐┌──────┐│
                           │  │Tool A││Tool B││Tool C││Tool D││
                           │  └──────┘└──────┘└──────┘└──────┘│
                           └──────────────────────────────────┘
```

---

## 5. 能力边界

### 5.1 熔断器不能解决什么

```
熔断器做不到:
  ┌─ 防止首次故障 (熔断需要先有失败才能触发)
  ├─ 区分"故障"和"慢请求" (需要超时模式配合)
  ├─ 自动修复依赖 (熔断器只切断流量, 不修复下游)
  └─ 保证语义正确性 (熔断器只看"调用是否成功", 不看"内容是否正确")
```

### 5.2 熔断器配置参考

| 场景 | failure_threshold | recovery_timeout | half_open_max_requests | 说明 |
|------|-------------------|------------------|----------------------|------|
| LLM Provider | 5 | 30s | 3 | Provider 故障恢复较慢, 需耐心 |
| 代码执行工具 | 3 | 15s | 2 | 代码沙箱故障通常短期可恢复 |
| 外部 HTTP API | 4 | 20s | 3 | 网络问题通常在几分钟内缓解 |
| 数据库查询 | 3 | 10s | 2 | DB 故障恢复快, 阈值可低 |
| Agent 任务级 | 3 次语义错误 | — | — | 一次任务内, 不等超时直接熔断 |
| 全局熔断 | 50 | 60s | 5 | 保守阈值, 避免大规模熔断 |

### 5.3 常见反模式

```
❌ "熔断阈值设大一点, 更安全"
   阈值太大 = 熔断永远不触发 = 等于没有熔断器。
   熔断器的目的不是"尽可能不触发", 而是"在需要的时候果断触发"。

❌ "所有工具共享一个熔断器"
   工具 A 故障会影响工具 B。应该每个工具/依赖独立熔断器。

❌ "熔断只是返回错误"
   熔断应该是降级链的一环: 熔断 → 降级 → 缓存 → 人工。
   返回"系统错误"是最差的降级。

❌ "熔断后无限期等待"
   没有恢复机制的熔断最终会成为"永久性功能禁用"。
   必须有 HALF_OPEN + 自动恢复逻辑。
```

---

## 6. 工程优化方向

### 6.1 实施步骤

```
1. 识别依赖: 列出所有外部依赖 (LLM Provider / 工具 API / 数据库 / 缓存)
2. 阈值配置: 为每个依赖设置初始熔断参数
3. 注册熔断器: 在 AgentCircuitBreaker 中注册所有依赖
4. 集成降级链: 熔断开启时触发 DegradationManager
5. 添加监控: 熔断事件记录 + 告警
6. 启用自适应: 基于历史数据优化阈值
7. 混沌测试: 通过故障注入验证熔断器有效性
```

### 6.2 监控指标

```
熔断器监控:
  ├── 熔断率: 熔断开启时间 / 总运行时间 (目标 < 1%)
  ├── 熔断次数: 每个依赖的熔断触发频率
  ├── 平均恢复时间: OPEN → CLOSED 的平均时间
  ├── 半开成功率: HALF_OPEN 试探请求的成功率
  └── 无效请求减少量: 熔断器节省的无效调用次数
```

### 6.3 实践检查清单

```
[ ] 1. 每个外部依赖有独立的熔断器
[ ] 2. 熔断阈值按场景差异化配置
[ ] 3. 熔断开启时触发降级链 (不只是报错)
[ ] 4. 半开状态有试探机制 + 渐进恢复
[ ] 5. 熔断事件有告警通知
[ ] 6. Agent 任务级熔断能检测循环和语义偏差
[ ] 7. 全局熔断在所有 Provider 不可用时触发
[ ] 8. 熔断器状态可观测 (Prometheus metrics)
[ ] 9. 熔断器支持人工干预 (PERMA_OPEN 模式)
[ ] 10. 定期测试熔断器有效性
```

---

> **总结**：熔断器是防止级联故障的关键防线。在 Agent 系统中，熔断器需要在三个层级工作：LLM Provider 级别（防止 API 重试风暴）、工具级别（隔离外部服务故障）、Agent 任务级别（检测逻辑层面的异常行为）。核心设计要点是：(1) 每个依赖独立熔断；(2) 熔断后触发降级而非简单报错；(3) 支持自适应阈值和渐进恢复；(4) 熔断器状态可观测、可人工干预。
