# 13.2.3 rate-limiting — 速率限制

**Agent 系统的速率限制不是传统的 API 保护问题——核心矛盾在于：LLM API 的消费速率有 Provider 强加的三重天花板（RPM、TPM、IPM），而 Agent 的工作负载天然具有突发性和不可预测性。** 一个 Agent 会话可能在 2 秒内调用 3 次 LLM API，也可能休眠 30 秒等待工具结果。如果没有精细的速率限制，一次 Agent 的突发请求就可能耗尽整月的 Token 预算，或触发 Provider 的限流导致级联超时。

## 背景与问题

### Agent 速率限制的三层问题

Agent 系统面临的速率限制问题有三个独立的层面，每一层的约束条件不同：

```
速率限制问题全景:

Layer 1: Provider API Limits (外部硬约束)
┌─────────────────────────────────────────────┐
│  OpenAI:     RPM 10K / TPM 2M / IPM 2M      │
│  Anthropic:  RPM 1K  / TPM 100K             │
│  Google:     RPM 1.5K / TPM 1M              │
│  违反 → HTTP 429 → 指数退避 → 延迟累积       │
└─────────────────────────────────────────────┘

Layer 2: Cost Budget (内部硬约束)
┌─────────────────────────────────────────────┐
│  Monthly Budget: $10,000                    │
│  GPT-4o: $5/1M input + $15/1M output       │
│  Haiku: $0.25/1M input + $1.25/1M output   │
│  超出 → 服务降级 / 模型降级 / 拒绝服务        │
└─────────────────────────────────────────────┘

Layer 3: Multi-tenant Fairness (内部软约束)
┌─────────────────────────────────────────────┐
│  Tenant A (白标客户):  max 100 RPM          │
│  Tenant B (免费用户):   max 10 RPM          │
│  Tenant C (内部测试):   max 200 RPM         │
│  违反 → 服务质量下降但不会完全拒绝            │
└─────────────────────────────────────────────┘
```

这三个层面的约束必须同时满足，任何一层的越界都会导致系统不可用或超支。

### Agent 自动重试如何放大限流问题

Agent 系统的自动重试机制有着与传统 API 客户端完全不同的放大效应：

```
传统 API 客户端的限流:
  请求 → 429 → 退避 → 重试 → 成功
  失败成本: 1 次请求的延迟

Agent 的限流放大链:
  请求 → 429 → Agent 等待退避
         → 但 Agent 还在保持上下文 (Token 被占用)
         → 退避结束重试 → 再 429 → 再退避
         → 上下文超长 → Token 消耗暴增
         → 嵌套 Agent 调用: 父 Agent 等待子 Agent
         → 子 Agent 被限流 → 父 Agent 也在等
         → 级联超时 → 整个会话失败
  失败成本: N 次重试 + 浪费的上下文 Token + 级联会话超时
```

这种放大效应使得 Agent 系统的速率限制比传统 API 网关严格得多——不能简单地"等到有配额再发"，而需要在请求发出之前就精确计算并预留资源。

### 不对称的成本结构

```
非 Agent 系统:   限流等待 = 浪费 1 次网络往返
                 成本 = 几乎为零

Agent 系统:      限流等待 = Agent 保持上下文窗口 (数千 Token)
                          重试消耗额外 Token
                          工具结果需要重新计算或缓存
                          用户感知延迟增加 (Agent 反应变慢)
                 成本 = Token 消耗 × 重试次数 + 用户体验下降
```

一个被限流反复打断的 Agent 会话，其总 Token 消耗可能是正常会话的 3-5 倍——因为在退避期间积累的上下文消息（错误信息、重试状态、部分结果）都会计入 Token 消耗。

## 之前是怎么做的

早期 Agent 系统对速率限制的处理非常朴素，大多数方案在分布式场景下存在严重缺陷。

### 方案一：简单的固定窗口计数器

```python
# naive_rate_limiter.py — 早期实现
import time
from collections import defaultdict

class FixedWindowRateLimiter:
    """每个 API Key 的固定窗口计数器"""
    
    def __init__(self, max_requests: int = 100, window: int = 60):
        self.max_requests = max_requests
        self.window = window  # 窗口大小（秒）
        self.counters: dict[str, list] = defaultdict(list)  # api_key -> [timestamps]
    
    def allow_request(self, api_key: str) -> bool:
        now = time.time()
        window_start = now - self.window
        
        # 清理过期记录
        self.counters[api_key] = [
            t for t in self.counters[api_key] 
            if t > window_start
        ]
        
        if len(self.counters[api_key]) >= self.max_requests:
            return False
        
        self.counters[api_key].append(now)
        return True

# 问题:
# 1. 每个进程独立计数器，多实例时完全失效
# 2. 窗口边界突变：窗口末尾大量请求积累，下一窗口开始瞬间全部通过
# 3. 没有 Token 感知，只统计请求次数
# 4. 进程重启计数器清零
```

**存在的问题：**

```
固定窗口的边界效应:

请求数
^
100 |                ████████████████
    |                ████████████████    ← 窗口末尾突发 100 请求
 50 |                ████████████████
    |████████████████                  ← 下一窗口开始又 100 请求
  0 +─────────────────────────────────→ 时间
    0s              60s              120s

实际每秒速率在窗口边界处可能瞬时达到 200/秒 (2x 限制)
```

### 方案二：单进程 Token Bucket

```python
# token_bucket.py — 单进程版本
import time
import threading

class LocalTokenBucket:
    """进程内的 Token Bucket——不跨进程共享"""
    
    def __init__(self, rate: float, capacity: int):
        self.rate = rate          # Token 补充速率（个/秒）
        self.capacity = capacity  # 最大容量
        self.tokens = capacity    # 当前 Token 数
        self.last_refill = time.monotonic()
        self.lock = threading.Lock()
    
    def consume(self, tokens: int = 1) -> bool:
        with self.lock:
            self._refill()
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            return False
    
    def _refill(self):
        now = time.monotonic()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, 
                          self.tokens + elapsed * self.rate)
        self.last_refill = now

# 问题:
# 1. 多实例部署时每个实例有自己的 Token Bucket，总速率 = N × rate
# 2. 进程重启后桶重新填满，可能瞬间超限
# 3. threading.Lock 在 asyncio 中阻塞事件循环
```

### 方案三：硬限制直接拒绝

```python
class HardLimitMiddleware:
    """到达限制就直接拒绝——没有降级、没有排队"""
    
    async def __call__(self, request):
        if await self.is_rate_limited(request.user_id):
            return HTTPResponse(status=429, body="Rate limit exceeded")
            # ↑ 没有指示重试时间，没有降级选项
        return await self.app(request)
```

### 方案四：共享 API Key 问题

```
所有 Agent 实例共用一个 API Key:
                         OpenAI API
                            ↑
                   ┌────────┴────────┐
                   │    API Key X     │
                   └────────┬────────┘
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
   ┌────────────┐   ┌────────────┐   ┌────────────┐
   │ Agent Pod 1 │   │ Agent Pod 2 │   │ Agent Pod 3 │
   │ (突发 500   │   │ (正常 10   │   │ (正常 10   │
   │ 请求/秒)   │   │ 请求/秒)   │   │ 请求/秒)   │
   └────────────┘   └────────────┘   └────────────┘
   
   结果: Pod 1 突发 → 全局 429 → Pod 2 和 Pod 3 全部被限流
   一个 Agent 的突发行为阻塞了所有其他 Agent
```

这些早期方案的核心缺陷在于：**没有认识到 Agent 系统的速率限制本质上是一个分布式资源调度问题，而不是单机限流问题。**

## 核心矛盾

### 矛盾一：API Key 级别的限制 vs 分布式的 Agent 实例

LLM Provider 的速率限制是 per-account/per-key 的，但 Agent 实例可能是 10 个、100 个甚至更多：

```
         ┌──────────────────────────────────────┐
         │       LLM Provider API               │
         │       Rate Limit: 10K RPM / Key      │
         └────────────────┬─────────────────────┘
                          │
               API Key "sk-xxx..."
                          │
     ┌──────────┬──────────┼──────────┬──────────┐
     │          │          │          │          │
     ▼          ▼          ▼          ▼          ▼
   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐
   │Pod 1│   │Pod 2│   │Pod 3│   │Pod 4│   │Pod N│
   └─────┘   └─────┘   └─────┘   └─────┘   └─────┘
   每个 Pod 无法感知其他 Pod 的消费量
   总消费 = N 个 Pod 消费之和，但每个 Pod 以为只有自己
   → 实际速率可能超出限制 N 倍
```

**解决方案：** 需要一个全局协调者（通常是 Redis）来同步所有实例的速率计数。

### 矛盾二：突发性 Agent 工作负载 vs 平滑的 API 消耗要求

Agent 的 Token 消耗特征与传统 API 调用完全不同：

```
传统 API 调用模式:    ═══║═══║═══║═══║═══║═══
                     稳定、可预测的请求间隔

Agent 推理的 Token 消耗模式:
第一阶段: 系统提示词加载     ████████░░░░░░░░  8000 Token (输入)
第二阶段: 用户问题 + 上下文   ██████████████░░  12000 Token (输入)  
第三阶段: LLM 响应           ░░░░░░░░████████  4000 Token (输出)
第四阶段: 工具调用           ██░░░░░░░░░░░░░░  500 Token (输入)
第五阶段: 工具结果注入       ████████████░░░░  6000 Token (输入)
第六阶段: LLM 二次响应       ░░░░░░░░████████  3000 Token (输出)

总 Token: ~33,500，集中在 2-3 秒内爆发
平均速率 ≈ 11,000 TPM，峰值 ≈ 16,000 TPM
```

一个 Agent 会话在工具调用返回结果后，会突然注入大量 Token 到上下文并触发新一轮 LLM 调用。这种"静默 → 突发 → 静默 → 突发"的模式使得传统的平滑速率限制策略完全失效。

### 矛盾三：成本控制 vs 用户体验

```
成本控制策略          用户体验影响          适用场景
────────────────      ──────────────       ────────────────
硬限制 (Hard Cap)     请求被拒绝，用户困惑   预算严格固定的内部工具
                      可能丢失重要会话数据

软限制 (Soft Cap)     响应变慢但不会拒绝    面向客户的生产系统
                      主动降级到更小模型

排队 (Queue)          延迟增加，但保证处理   异步任务/批处理场景
                      用户需要等待更长时间

降级 (Degradation)    使用缓存或简单规则     高并发低价值场景
                      回答质量下降

混合 (Hybrid)         付费用户无影响         多层级定价 SaaS
                      免费用户受限制
```

核心挑战是：**Agent 会话在处理中途不能简单地"拒绝"——用户已经提出了问题，Agent 已经开始推理，中途拒绝意味着之前所有 Token 消耗全部浪费。**

## 当前主流方案

### 算法基础：Token Bucket

Token Bucket 是 LLM 速率限制最基础的算法，但生产级实现需要分布式版本：

```python
# distributed_token_bucket.py — 基于 Redis 的分布式 Token Bucket
import time
import hashlib
import redis.asyncio as aioredis

class DistributedTokenBucket:
    """基于 Redis 的分布式 Token Bucket
    支持 Token 感知的速率限制（不仅仅是请求计数）
    """
    
    def __init__(
        self,
        redis_client: aioredis.Redis,
        key: str,
        rate: float,        # Token 补充速率 / 秒
        capacity: int,      # 桶的最大容量
        default_tokens: int = 1,  # 每次默认消耗 Token 数
    ):
        self.redis = redis_client
        self.key = f"rate_limit:token_bucket:{key}"
        self.rate = rate
        self.capacity = capacity
        self.default_tokens = default_tokens
    
    async def consume(self, tokens: int | None = None) -> bool:
        """消耗 Token，返回是否允许请求"""
        tokens = tokens or self.default_tokens
        
        # Lua 脚本保证原子性
        lua_script = """
        local key = KEYS[1]
        local now = tonumber(ARGV[1])
        local rate = tonumber(ARGV[2])
        local capacity = tonumber(ARGV[3])
        local requested = tonumber(ARGV[4])
        
        -- 获取当前 Token 数和时间戳
        local bucket = redis.call('hmget', key, 'tokens', 'last_refill')
        local tokens = tonumber(bucket[1]) or capacity
        local last_refill = tonumber(bucket[2]) or now
        
        -- 计算应补充的 Token
        local elapsed = now - last_refill
        local new_tokens = math.min(capacity, tokens + elapsed * rate)
        
        -- 检查是否足够
        if new_tokens >= requested then
            new_tokens = new_tokens - requested
            redis.call('hmset', key, 
                       'tokens', new_tokens, 
                       'last_refill', now)
            redis.call('expire', key, 86400)  -- TTL 1天
            return 1  -- 允许
        else
            -- 更新 last_refill 但保持当前 Token 数
            redis.call('hmset', key,
                       'tokens', new_tokens,
                       'last_refill', now)
            return 0  -- 拒绝
        end
        """
        
        sha = hashlib.sha1(lua_script.encode()).hexdigest()
        
        try:
            result = await self.redis.evalsha(
                sha, 1, self.key,
                str(time.time()),
                str(self.rate),
                str(self.capacity),
                str(tokens)
            )
            return bool(result)
        except redis.exceptions.NoScriptError:
            result = await self.redis.eval(
                lua_script, 1, self.key,
                str(time.time()),
                str(self.rate),
                str(self.capacity),
                str(tokens)
            )
            return bool(result)
    
    async def get_available_tokens(self) -> float:
        """查询当前可用 Token 数（不消耗）"""
        result = await self.redis.hmget(
            self.key, 'tokens', 'last_refill'
        )
        tokens = float(result[0] or self.capacity)
        last_refill = float(result[1] or time.time())
        elapsed = time.time() - last_refill
        return min(self.capacity, tokens + elapsed * self.rate)
```

### 算法进阶：滑动窗口计数器

滑动窗口比固定窗口更平滑。生产环境最常用的是滑动窗口计数器（近似算法，比滑动日志省内存）：

```python
# sliding_window_counter.py — Redis 滑动窗口计数器
import time
import hashlib
import redis.asyncio as aioredis

class SlidingWindowRateLimiter:
    """基于 Redis 排序集的滑动窗口计数器
    比固定窗口更平滑，比 Token Bucket 更精确
    """
    
    def __init__(
        self,
        redis_client: aioredis.Redis,
        key: str,
        max_requests: int,   # 窗口内最大请求数
        window_size: int,    # 窗口大小（秒）
    ):
        self.redis = redis_client
        self.key = f"rate_limit:sliding:{key}"
        self.max_requests = max_requests
        self.window_size = window_size
    
    async def allow_request(self, weight: int = 1) -> tuple[bool, dict]:
        """
        检查是否允许请求
        返回: (是否允许, 当前状态)
        """
        now = time.time()
        window_start = now - self.window_size
        
        lua_script = """
        local key = KEYS[1]
        local now = tonumber(ARGV[1])
        local window_start = tonumber(ARGV[2])
        local max_requests = tonumber(ARGV[3])
        local weight = tonumber(ARGV[4])
        
        -- 移除窗口外的旧记录
        redis.call('zremrangebyscore', key, 0, window_start)
        
        -- 计算当前窗口内的总权重
        local current = redis.call('zcard', key)
        
        -- 检查是否超过限制
        if current >= max_requests then
            -- 获取最早的记录过期时间
            local oldest = redis.call('zrange', key, 0, 0, 'withscores')
            local reset_after = 0
            if oldest[2] then
                reset_after = tonumber(oldest[2]) + max_requests - now
            end
            
            return {0, current, math.max(0, reset_after)}
        end
        
        -- 添加当前请求记录
        local member = now .. ':' ..  tostring(math.random())
        redis.call('zadd', key, now, member)
        redis.call('expire', key, math.ceil(self.window_size * 2))
        
        return {1, current + 1, 0}
        """
        
        sha = hashlib.sha1(lua_script.encode()).hexdigest()
        
        try:
            result = await self.redis.evalsha(
                sha, 1, self.key,
                str(now), str(window_start),
                str(self.max_requests), str(weight)
            )
        except redis.exceptions.NoScriptError:
            result = await self.redis.eval(
                lua_script, 1, self.key,
                str(now), str(window_start),
                str(self.max_requests), str(weight)
            )
        
        allowed = bool(result[0])
        current_count = int(result[1])
        reset_after = float(result[2])
        
        return allowed, {
            "current": current_count,
            "limit": self.max_requests,
            "remaining": self.max_requests - current_count if allowed else 0,
            "reset_after": reset_after,
            "window_size": self.window_size,
        }
```

### 多层速率限制架构

生产级 Agent 系统需要四个独立的速率限制层，每一层关注不同的约束：

```
多层速率限制架构:

                                    ┌──────────────────────┐
                                    │     Agent Request     │
                                    └──────────┬───────────┘
                                               │
                                    ┌──────────▼───────────┐
                                    │  Layer 4: 成本预算限制  │
                                    │  (月度/日度 Token 上限) │
                                    │  Redis key: cost:{ymd} │
                                    └──────────┬───────────┘
                                               │ 超出月度预算 → 拒绝
                                    ┌──────────▼───────────┐
                                    │  Layer 3: 租户公平性   │
                                    │  (每租户独立限制)       │
                                    │  Redis key: tenant:{id}│
                                    └──────────┬───────────┘
                                               │ 超出租户配额 → 降级
                                    ┌──────────▼───────────┐
                                    │  Layer 2: API Key 限制 │
                                    │  (映射到 Provider 限制) │
                                    │  Redis key: apikey:{k} │
                                    └──────────┬───────────┘
                                               │ 超出 Provider 限制 → 429
                                    ┌──────────▼───────────┐
                                    │  Layer 1: 全局 Token   │
                                    │  预算 (总消费控制)      │
                                    │  Redis key: global     │
                                    └──────────┬───────────┘
                                               │ 超出全局预算 → 排队
                                               │
                                    ┌──────────▼───────────┐
                                    │    LLM Provider API    │
                                    └──────────────────────┘
```

每一层的职责和实现：

```python
# multi_layer_limiter.py — 多层速率限制协调器
import time
import logging
from dataclasses import dataclass
from enum import Enum

logger = logging.getLogger(__name__)


class LimitDecision(Enum):
    ALLOW = "allow"
    REJECT = "reject"
    DEGRADE = "degrade"   # 降级到小模型
    QUEUE = "queue"       # 排队等待
    CACHE = "cache"       # 尝试缓存命中


@dataclass
class LimitResult:
    decision: LimitDecision
    layer: str
    reason: str
    retry_after: float = 0.0
    suggested_model: str | None = None


class MultiLayerRateLimiter:
    """四层速率限制协调器
    按 Layer 4 → 3 → 2 → 1 顺序检查
    任何一层拒绝则请求被限制
    """
    
    def __init__(self, redis_client):
        self.redis = redis_client
        self.layers: list[RateLimitLayer] = []
    
    def add_layer(self, layer: "RateLimitLayer"):
        """添加限制层"""
        self.layers.append(layer)
    
    async def check(
        self,
        request: "AgentRequest",
    ) -> LimitResult:
        """逐层检查请求是否允许"""
        
        for layer in self.layers:
            result = await layer.check(request)
            
            if result.decision != LimitDecision.ALLOW:
                logger.warning(
                    f"Rate limited by {layer.name}: "
                    f"{result.reason}"
                )
                return result
        
        # 所有层都通过
        return LimitResult(
            decision=LimitDecision.ALLOW,
            layer="all",
            reason="ok"
        )
    
    async def record(self, request: "AgentRequest", tokens_used: int):
        """在所有层记录本次消耗——只在确认请求成功后调用"""
        for layer in self.layers:
            await layer.record(request, tokens_used)


# ─── 各层的具体实现 ───

class RateLimitLayer:
    """速率限制层基类"""
    
    def __init__(self, name: str, redis_client):
        self.name = name
        self.redis = redis_client
    
    async def check(self, request: "AgentRequest") -> LimitResult:
        raise NotImplementedError
    
    async def record(self, request: "AgentRequest", tokens_used: int):
        """记录实际消耗——只在请求成功后调用"""
        pass


class CostBudgetLayer(RateLimitLayer):
    """成本预算限制层 (Layer 4)
    确保月度/日度 Token 消耗不超过预算
    """
    
    def __init__(
        self,
        redis_client,
        monthly_token_budget: int,
        daily_token_budget: int | None = None,
    ):
        super().__init__("cost_budget", redis_client)
        self.monthly_budget = monthly_token_budget
        self.daily_budget = daily_token_budget or (monthly_token_budget // 30)
    
    async def check(self, request: "AgentRequest") -> LimitResult:
        now = time.time()
        ymd = time.strftime("%Y%m%d", time.localtime(now))
        ym = ymd[:6]  # YYYYMM
        
        # 检查日预算
        daily_key = f"cost:daily:{ymd}"
        daily_used = int(await self.redis.get(daily_key) or 0)
        estimated = daily_used + request.estimated_tokens
        
        if estimated > self.daily_budget:
            reset_time = 86400 - (now % 86400)
            return LimitResult(
                decision=LimitDecision.DEGRADE,
                layer=self.name,
                reason=f"Daily budget exceeded: {daily_used}/{self.daily_budget}",
                retry_after=reset_time,
                suggested_model="gpt-4o-mini"  # 自动降级
            )
        
        # 检查月预算
        monthly_key = f"cost:monthly:{ym}"
        monthly_used = int(await self.redis.get(monthly_key) or 0)
        
        if monthly_used > self.monthly_budget:
            return LimitResult(
                decision=LimitDecision.REJECT,
                layer=self.name,
                reason=f"Monthly budget exhausted: {monthly_used}/{self.monthly_budget}",
            )
        
        return LimitResult(
            decision=LimitDecision.ALLOW,
            layer=self.name,
            reason="ok"
        )
    
    async def record(self, request: "AgentRequest", tokens_used: int):
        """记录实际消耗"""
        now = time.time()
        ymd = time.strftime("%Y%m%d", time.localtime(now))
        ym = ymd[:6]
        
        pipe = self.redis.pipeline()
        pipe.incrby(f"cost:daily:{ymd}", tokens_used)
        pipe.expire(f"cost:daily:{ymd}", 86400 * 2)  # 2 天 TTL
        pipe.incrby(f"cost:monthly:{ym}", tokens_used)
        pipe.expire(f"cost:monthly:{ym}", 86400 * 35)  # 35 天 TTL
        await pipe.execute()


class TenantFairnessLayer(RateLimitLayer):
    """租户公平性限制层 (Layer 3)
    确保每个租户/用户不会过度占用共享资源
    """
    
    def __init__(
        self,
        redis_client,
        default_rpm: int = 60,
        tier_config: dict[str, dict] | None = None,
    ):
        super().__init__("tenant_fairness", redis_client)
        self.default_rpm = default_rpm
        # 层级配置: {"premium": {"rpm": 500, "tpm": 100000}, ...}
        self.tier_config = tier_config or {}
    
    async def check(self, request: "AgentRequest") -> LimitResult:
        tenant_id = request.tenant_id
        tier = request.tenant_tier or "default"
        config = self.tier_config.get(tier, {"rpm": self.default_rpm, "tpm": 50000})
        
        # 使用滑动窗口计数器
        limiter = SlidingWindowRateLimiter(
            self.redis,
            key=f"tenant:{tenant_id}:rpm",
            max_requests=config["rpm"],
            window_size=60,
        )
        
        allowed, status = await limiter.allow_request()
        
        if not allowed:
            return LimitResult(
                decision=LimitDecision.QUEUE,
                layer=self.name,
                reason=f"Tenant {tenant_id} rate limit: {status['current']}/{status['limit']}",
                retry_after=status.get("reset_after", 10),
            )
        
        # Token 限制
        token_limiter = SlidingWindowRateLimiter(
            self.redis,
            key=f"tenant:{tenant_id}:tpm",
            max_requests=config["tpm"],
            window_size=60,
        )
        
        token_allowed, token_status = await token_limiter.allow_request(
            weight=request.estimated_tokens
        )
        
        if not token_allowed:
            return LimitResult(
                decision=LimitDecision.QUEUE,
                layer=self.name,
                reason=f"Tenant {tenant_id} TPM limit",
                retry_after=token_status.get("reset_after", 10),
            )
        
        return LimitResult(
            decision=LimitDecision.ALLOW,
            layer=self.name,
            reason="ok"
        )


class APIKeyLimitLayer(RateLimitLayer):
    """API Key 限制层 (Layer 2)
    映射到 LLM Provider 的实际限制
    支持 RPM + TPM + IPM 三维度
    """
    
    def __init__(
        self,
        redis_client,
        api_key: str,
        rpm: int = 10000,    # Requests Per Minute
        tpm: int = 2000000,  # Tokens Per Minute
        ipm: int = 2000000,  # Input Tokens Per Minute (某些 Provider 单独限制输入)
    ):
        super().__init__("api_key_limit", redis_client)
        self.api_key = api_key
        self.rpm = rpm
        self.tpm = tpm
        self.ipm = ipm
    
    async def check(self, request: "AgentRequest") -> LimitResult:
        key_prefix = f"apikey:{self.api_key}"
        
        pipe = self.redis.pipeline()
        
        # 检查 RPM——使用滑动窗口
        pipe.zcard(f"{key_prefix}:rpm")
        # 检查 TPM——使用滑动窗口
        pipe.zcard(f"{key_prefix}:tpm")
        # 检查 IPM（如果请求是输入密集型）
        if request.is_input_heavy:
            pipe.zcard(f"{key_prefix}:ipm")
        
        results = await pipe.execute()
        current_rpm = results[0]
        current_tpm = results[1]
        current_ipm = results[2] if request.is_input_heavy else 0
        
        if current_rpm >= self.rpm:
            return LimitResult(
                decision=LimitDecision.QUEUE,
                layer=self.name,
                reason=f"RPM limit: {current_rpm}/{self.rpm}",
                retry_after=10,
            )
        
        estimated_tpm = current_tpm + request.estimated_tokens
        if estimated_tpm >= self.tpm:
            return LimitResult(
                decision=LimitDecision.QUEUE,
                layer=self.name,
                reason=f"TPM limit: ~{estimated_tpm}/{self.tpm}",
                retry_after=5,
            )
        
        if request.is_input_heavy:
            estimated_ipm = current_ipm + request.estimated_input_tokens
            if estimated_ipm >= self.ipm:
                return LimitResult(
                    decision=LimitDecision.QUEUE,
                    layer=self.name,
                    reason=f"IPM limit: ~{estimated_ipm}/{self.ipm}",
                    retry_after=5,
                )
        
        return LimitResult(
            decision=LimitDecision.ALLOW,
            layer=self.name,
            reason="ok"
        )
    
    async def record(self, request: "AgentRequest", tokens_used: int):
        """记录实际消耗"""
        key_prefix = f"apikey:{self.api_key}"
        now = time.time()
        
        pipe = self.redis.pipeline()
        
        # RPM: 记录 1 个请求
        pipe.zadd(f"{key_prefix}:rpm", {f"{now}:{id(request)}": now})
        pipe.zremrangebyscore(f"{key_prefix}:rpm", 0, now - 60)
        pipe.expire(f"{key_prefix}:rpm", 120)
        
        # TPM: 记录 Token 消耗
        pipe.zadd(f"{key_prefix}:tpm", {f"{now}:{id(request)}": now})
        # 使用成员权重存储 Token 数（Lua 中按权重计算）
        
        await pipe.execute()


class GlobalBudgetLayer(RateLimitLayer):
    """全局 Token 预算层 (Layer 1)
    整个系统的总消费控制——防止所有 API Key 并行打满
    """
    
    def __init__(
        self,
        redis_client,
        global_tpm: int,
        global_concurrency: int,
    ):
        super().__init__("global_budget", redis_client)
        self.global_tpm = global_tpm
        self.global_concurrency = global_concurrency
    
    async def check(self, request: "AgentRequest") -> LimitResult:
        # 并发数限制——使用 Redis Counter
        concurrency_key = "global:concurrency"
        current = await self.redis.incr(concurrency_key)
        
        if current > self.global_concurrency:
            await self.redis.decr(concurrency_key)
            return LimitResult(
                decision=LimitDecision.QUEUE,
                layer=self.name,
                reason=f"Global concurrency limit: {current}/{self.global_concurrency}",
                retry_after=5,
            )
        
        return LimitResult(
            decision=LimitDecision.ALLOW,
            layer=self.name,
            reason="ok"
        )
    
    async def record(self, request: "AgentRequest", tokens_used: int):
        """释放并发槽位"""
        await self.redis.decr("global:concurrency")
```

### Token 感知 vs 请求感知

两种速率限制策略的核心区别：

```
请求感知 (Request-aware):
              ┌──────────┐
              │ 1 请求 =  │     → 计数 +1
              │ 1 单位    │
              └──────────┘

  优点: 实现简单，开销低
  缺点: 一个 100 Token 的请求和一个 100K Token 的请求没有区别
        Agent 的 Token 消耗差异可达 1000x+

Token 感知 (Token-aware):
              ┌──────────┐
              │ 估算 Token │     → 计数 +N
              │ 消耗量    │
              └──────────┘

  优点: 精确反映 Provider 的实际计费单位
        Agent 场景必需的——因为 Agent 的 Token 消耗波动极大
  缺点: 需要预估算，精度 ±20-30%
        需要额外的 LLM Token 估算器
```

```python
# token_estimator.py — Token 消耗估算器
import tiktoken
from dataclasses import dataclass

@dataclass
class TokenEstimate:
    total: int
    input_tokens: int
    output_tokens: int
    confidence: float  # 0.0 ~ 1.0


class TokenEstimator:
    """基于消息内容的 Token 消耗预估算
    用在请求发出之前，为速率限制提供 Token 感知决策
    """
    
    def __init__(self, model: str = "gpt-4"):
        try:
            self.encoder = tiktoken.encoding_for_model(model)
        except KeyError:
            self.encoder = tiktoken.get_encoding("cl100k_base")
        
        # Token 估算系数 (经验值)
        self.char_to_token_ratio = 0.25  # 英文
        self.code_to_token_ratio = 0.33  # 代码
        self.chinese_ratio = 0.6        # 中文 (每个字 ≈ 0.6 Token)
    
    def estimate_from_messages(self, messages: list[dict]) -> TokenEstimate:
        """从消息列表估算 Token 消耗"""
        total_chars = 0
        input_chars = 0
        
        for msg in messages:
            content = msg.get("content", "")
            if isinstance(content, str):
                char_count = len(content)
                total_chars += char_count
                if msg.get("role") != "assistant":
                    input_chars += char_count
        
        # 估算输入 Token
        input_tokens = self._estimate_tokens_from_text(
            sum(len(m.get("content", "")) for m in messages 
                if m.get("role") != "assistant")
        )
        
        # 估算输出 Token (一般为输入的 20-40%)
        output_tokens = int(input_tokens * 0.3)
        
        # 加上格式开销 (~3 Token 每条消息)
        format_overhead = len(messages) * 3
        
        # 加上系统提示词
        system_messages = [m for m in messages if m.get("role") == "system"]
        system_tokens = sum(
            self.encoder.encode(m.get("content", ""))
            for m in system_messages
        )
        
        total = input_tokens + output_tokens + format_overhead + system_tokens
        
        return TokenEstimate(
            total=total,
            input_tokens=input_tokens,
            output_tokens=output_tokens,
            confidence=0.7,  # 预估算的置信度
        )
    
    def estimate_from_text(self, text: str) -> int:
        """从纯文本估算 Token 数"""
        return self._estimate_tokens_from_text(text)
    
    def _estimate_tokens_from_text(self, text: str) -> int:
        """智能估算 Token——结合精确和近似"""
        if len(text) < 1000:
            # 短文本：直接编码
            return len(self.encoder.encode(text))
        else:
            # 长文本：采样估算
            sample_size = min(2000, len(text))
            sample = text[:sample_size]
            sample_tokens = len(self.encoder.encode(sample))
            return int(sample_tokens * len(text) / sample_size * 1.05)
    
    def estimate_output_tokens(self, max_tokens: int = 4096) -> int:
        """估算输出 Token——通常由模型配置的 max_tokens 决定"""
        return max_tokens


class AgentAwareTokenEstimator(TokenEstimator):
    """Agent 感知的 Token 估算器
    考虑 Agent 特有的 Token 消耗模式：
    - 工具调用结果注入
    - 多轮对话积累
    - 系统提示词
    """
    
    TOOL_OVERHEAD = 50   # 每次工具调用的格式开销
    MEMORY_OVERHEAD = 100  # 每次记忆读取的开销
    
    def estimate_agent_request(
        self,
        messages: list[dict],
        tool_results: list[dict] | None = None,
        memory_docs: list[str] | None = None,
        expected_steps: int = 1,
    ) -> TokenEstimate:
        """估算一次完整 Agent 推理的 Token 消耗"""
        
        base = self.estimate_from_messages(messages)
        
        # 工具调用开销
        tool_tokens = 0
        if tool_results:
            for r in tool_results:
                tool_tokens += (
                    self.estimate_from_text(r.get("content", ""))
                    + self.TOOL_OVERHEAD
                )
        
        # 记忆开销
        memory_tokens = 0
        if memory_docs:
            for doc in memory_docs:
                memory_tokens += (
                    self.estimate_from_text(doc)
                    + self.MEMORY_OVERHEAD
                )
        
        # 多步推理：每一步都会产生新的输入 + 输出
        step_overhead = 0
        if expected_steps > 1:
            # 每一步的输出会成为下一步输入的一部分
            step_input = base.output_tokens * 0.8
            step_output = base.output_tokens * 0.6
            step_overhead = (expected_steps - 1) * (step_input + step_output)
        
        total = base.total + tool_tokens + memory_tokens + step_overhead
        
        return TokenEstimate(
            total=total,
            input_tokens=base.input_tokens + tool_tokens + memory_tokens,
            output_tokens=base.output_tokens + step_overhead,
            confidence=0.5 if expected_steps > 3 else 0.7,
        )
```

### 并发限制 vs 速率限制

这是一个关键区分——两者解决不同的问题：

```
速率限制 (Rate Limit)
  ┌─────────────────────────────────────────────┐
  │ 定义: 单位时间内的请求/Token 数量上限         │
  │ 控制: X requests per minute                 │
  │       Y tokens per minute                   │
  │ 目标: 匹配 Provider 的 API 限制              │
  │ 问题: 长时间运行的 Agent 会话即使速率低        │
  │       也可能因持续消耗 Token 而超限           │
  └─────────────────────────────────────────────┘

并发限制 (Concurrency Limit)
  ┌─────────────────────────────────────────────┐
  │ 定义: 同时进行中的请求/会话数量上限            │
  │ 控制: max N in-flight requests              │
  │ 目标: 防止队列堆积和资源耗尽                   │
  │ 关键: Agent 会话持续 5-30 秒，               │
  │       一个会话可能发起多次 LLM 调用            │
  │       并发限制 = 同时处理的会话数上限           │
  └─────────────────────────────────────────────┘

关系:
  速率限制 = 单位时间内的总吞吐上限
  并发限制 = 任意时刻的并行度上限
  两者互为补充——同时设置才能有效保护系统

Agent 会话的典型关系:
  10 个并发会话 × 每个会话平均 3 次 LLM 调用 × 每次 2000 Token
  → 并发: 10 会话 (并发限制)
  → 吞吐: 30 LLM 请求 / 会话时长 (速率限制)
  → Token: 60000 Token / 会话时长
```

```python
# concurrency_limiter.py — 基于信号量的并发限制器
import asyncio
from contextlib import asynccontextmanager

class ConcurrencyLimiter:
    """并发限制器——控制同时进行中的 Agent 会话数
    对于长时间运行的 Agent 会话，这比纯速率限制更重要
    """
    
    def __init__(self, max_concurrent: int):
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.active_count = 0
        self.wait_count = 0
        self.max_concurrent = max_concurrent
    
    @asynccontextmanager
    async def acquire(self):
        """获取并发槽位"""
        self.wait_count += 1
        try:
            async with self.semaphore:
                self.active_count += 1
                self.wait_count -= 1
                yield
        finally:
            self.active_count -= 1
    
    async def try_acquire(self, timeout: float = 5.0) -> bool:
        """尝试获取并发槽位（带超时）"""
        try:
            await asyncio.wait_for(self.semaphore.acquire(), timeout=timeout)
            self.active_count += 1
            return True
        except asyncio.TimeoutError:
            return False
    
    @property
    def utilization(self) -> float:
        """当前并发利用率"""
        return self.active_count / self.max_concurrent
    
    @property
    def queue_depth(self) -> int:
        """等待队列深度"""
        return self.wait_count


class TokenAwareConcurrencyLimiter(ConcurrencyLimiter):
    """Token 感知的并发限制器
    限制的不是会话数，而是所有运行会话的预测 Token 总和
    """
    
    def __init__(self, max_concurrent_tokens: int):
        super().__init__(max_concurrent_tokens)  # 复用信号量机制
        self.max_concurrent_tokens = max_concurrent_tokens
        self.active_tokens = 0
    
    async def try_acquire_token(
        self, estimated_tokens: int, timeout: float = 5.0
    ) -> bool:
        """尝试获取指定 Token 的并发槽位"""
        # 简化实现：如果剩余 Token 足够则允许
        if self.active_tokens + estimated_tokens > self.max_concurrent_tokens:
            return False
        self.active_tokens += estimated_tokens
        return True
    
    def release_token(self, tokens: int):
        """释放 Token 槽位"""
        self.active_tokens = max(0, self.active_tokens - tokens)
```

### 自适应速率限制

静态速率限制假设 Provider 的限制是固定不变的。实际上，Provider 的限流阈值在不同时段、不同负载下会有波动。自适应限流根据实时的错误率和延迟动态调整发送速率：

```python
# adaptive_rate_limiter.py — 自适应速率限制器
import time
import asyncio
from collections import deque

class AdaptiveRateLimiter:
    """自适应速率限制器
    根据 API 返回的错误率和延迟动态调整发送速率
    可在 Provider 限流阈值下降时提前降低速率，减少 429 错误
    
    参考文献: 
    - AIMD (Additive Increase Multiplicative Decrease) 拥塞控制
    - Netflix 的适应性限流模式
    """
    
    def __init__(
        self,
        redis_client,
        key: str,
        min_rate: int = 10,      # RPM 最小值 (保底)
        max_rate: int = 10000,   # RPM 最大值 (Provider 限制)
        initial_rate: int = 1000,
        window_size: int = 60,   # 评估窗口 (秒)
    ):
        self.redis = redis_client
        self.key = key
        self.min_rate = min_rate
        self.max_rate = max_rate
        self.current_rate = initial_rate
        self.window_size = window_size
        
        # 滑动窗口内的统计
        self.recent_errors = deque(maxlen=100)
        self.recent_latencies = deque(maxlen=100)
        
        # AIMD 参数
        self.increment = 50       # 加性增 (+50 RPM)
        self.decrement = 0.5      # 乘性减 (×0.5)
        
        # 自适应周期
        self.last_adjustment = time.time()
        self.adjustment_interval = 10  # 每 10 秒评估一次
    
    async def record_result(
        self, 
        latency: float, 
        status_code: int,
        is_rate_limited: bool,
    ):
        """记录一次 API 调用的结果"""
        self.recent_latencies.append(latency)
        self.recent_errors.append(is_rate_limited)
        
        # 自适应调整
        now = time.time()
        if now - self.last_adjustment >= self.adjustment_interval:
            await self._adjust_rate()
            self.last_adjustment = now
    
    async def _adjust_rate(self):
        """根据近期指标调整速率"""
        if not self.recent_errors:
            return
        
        # 计算近期错误率
        error_rate = sum(self.recent_errors) / len(self.recent_errors)
        
        # 计算近期 P95 延迟
        sorted_latencies = sorted(self.recent_latencies)
        if sorted_latencies:
            p95_idx = int(len(sorted_latencies) * 0.95)
            p95_latency = sorted_latencies[min(p95_idx, len(sorted_latencies) - 1)]
        else:
            p95_latency = 0
        
        old_rate = self.current_rate
        
        if error_rate > 0.1:
            # 错误率 > 10%: 快速降速 (乘性减)
            self.current_rate = max(
                self.min_rate,
                int(self.current_rate * (1.0 - error_rate))
            )
            reason = f"高错误率 {error_rate:.1%} → 降速"
        
        elif error_rate > 0.05:
            # 错误率 5-10%: 适度降速
            self.current_rate = max(
                self.min_rate,
                int(self.current_rate * 0.8)
            )
            reason = f"中等错误率 {error_rate:.1%} → 降速"
        
        elif p95_latency > 5.0:
            # P95 延迟 > 5s: Provider 可能过载，降速
            self.current_rate = max(
                self.min_rate,
                int(self.current_rate * 0.9)
            )
            reason = f"高延迟 P95={p95_latency:.1f}s → 降速"
        
        elif error_rate < 0.01 and p95_latency < 2.0:
            # 一切正常: 缓慢增加 (加性增)
            self.current_rate = min(
                self.max_rate,
                self.current_rate + self.increment
            )
            reason = f"运行正常 → 增速"
        
        else:
            reason = "维持当前速率"
        
        # 更新 Redis (持久化动态速率)
        await self.redis.set(
            f"adaptive_rate:{self.key}",
            self.current_rate,
            ex=300,  # 5 分钟 TTL
        )
        
        if self.current_rate != old_rate:
            print(f"[AdaptiveRate] {reason}: {old_rate} → {self.current_rate} RPM")
    
    async def get_allowed_rate(self) -> int:
        """获取当前允许的速率"""
        # 优先使用 Redis 中的动态速率
        redis_rate = await self.redis.get(f"adaptive_rate:{self.key}")
        if redis_rate:
            self.current_rate = int(redis_rate)
        return self.current_rate
```

### 优雅降级策略

当所有速率限制层都拒绝请求时，Agent 不应该简单地返回错误。应该有一条完整的降级链：

```
降级链 (Degradation Chain):

                    Agent Request
                         │
                    ┌────▼────┐
                    │ 正常调用  │ ←──── 全速率
                    │ LLM API  │
                    └────┬────┘
                         │ 被限流
                    ┌────▼────┐
                    │ 语义缓存  │ ←──── 缓存命中
                    │ 命中检查  │       (降低 50-70% 调用)
                    └────┬────┘
                         │ 未命中
                    ┌────▼────┐
                    │ 降级模型  │ ←──── 用小模型处理
                    │ (Haiku)  │       (成本降至 1/10)
                    └────┬────┘
                         │ 仍被限流
                    ┌────▼────┐
                    │ 排队等待  │ ←──── 确定等待时间
                    │ (Queue)  │       (适合非实时场景)
                    └────┬────┘
                         │ 队列满
                    ┌────▼────┐
                    │ 返回缓存  │ ←──── 返回最近相似回答
                    │ 旧结果    │       (降低质量但不拒绝)
                    └────┬────┘
                         │ 无缓存
                    ┌────▼────┐
                    │ 拒绝请求  │ ←──── 最后手段
                    │ (429)    │       带有 Retry-After
                    └─────────┘
```

```python
# graceful_degradation.py — 优雅降级协调器
import asyncio
import logging
from dataclasses import dataclass

logger = logging.getLogger(__name__)


@dataclass
class DegradationResult:
    success: bool
    content: str | None
    source: str  # "normal" | "cache" | "degraded_model" | "stale_cache" | "queued" | "rejected"
    model_used: str | None
    latency: float


class GracefulDegradationHandler:
    """优雅降级协调器
    在速率受限时按优先级尝试降级策略
    """
    
    def __init__(
        self,
        primary_llm,        # 主模型 (如 GPT-4o, Sonnet)
        fallback_llm,       # 降级模型 (如 GPT-4o-mini, Haiku)
        semantic_cache,     # 语义缓存
        rate_limiter,       # 多层速率限制器
        queue,              # 异步队列
    ):
        self.primary = primary_llm
        self.fallback = fallback_llm
        self.cache = semantic_cache
        self.rate_limiter = rate_limiter
        self.queue = queue
    
    async def execute_with_degradation(
        self,
        request: "AgentRequest",
        max_retries: int = 2,
    ) -> DegradationResult:
        """尝试正常执行，被限流时自动降级"""
        start = time.time()
        
        # ─── 层级 1: 尝试正常调用 ───
        result = await self.rate_limiter.check(request)
        
        if result.decision == LimitDecision.ALLOW:
            try:
                response = await self.primary.chat(request.messages)
                await self.rate_limiter.record(request, response.usage.total_tokens)
                return DegradationResult(
                    success=True,
                    content=response.content,
                    source="normal",
                    model_used="gpt-4o",
                    latency=time.time() - start,
                )
            except Exception as e:
                logger.error(f"Primary model failed: {e}")
                # 继续降级
        
        # ─── 层级 2: 尝试缓存命中 ───
        cache_result = await self.cache.search(request.messages[-1]["content"])
        if cache_result and cache_result.similarity > 0.92:
            logger.info(f"Cache hit (similarity={cache_result.similarity:.2f})")
            return DegradationResult(
                success=True,
                content=cache_result.response,
                source="cache",
                model_used=None,
                latency=time.time() - start,
            )
        
        # ─── 层级 3: 尝试降级模型 ───
        if result.decision in (LimitDecision.DEGRADE, LimitDecision.QUEUE):
            # 降级请求——使用小模型
            degraded_request = request.clone()
            degraded_request.estimated_tokens = request.estimated_tokens // 2
            
            degraded_check = await self.rate_limiter.check(degraded_request)
            if degraded_check.decision == LimitDecision.ALLOW:
                try:
                    response = await self.fallback.chat(request.messages)
                    await self.rate_limiter.record(degraded_request, 
                                                   response.usage.total_tokens)
                    logger.info("Degraded to fallback model")
                    return DegradationResult(
                        success=True,
                        content=response.content,
                        source="degraded_model",
                        model_used="gpt-4o-mini",
                        latency=time.time() - start,
                    )
                except Exception as e:
                    logger.error(f"Fallback model also failed: {e}")
        
        # ─── 层级 4: 排队 ───
        if result.decision == LimitDecision.QUEUE:
            queue_result = await self._try_queue(request, result.retry_after)
            if queue_result:
                return queue_result
        
        # ─── 层级 5: 返回过期的缓存结果 ───
        stale_result = await self.cache.search(
            request.messages[-1]["content"],
            min_similarity=0.7  # 降低相似度阈值
        )
        if stale_result:
            logger.warning("Returning stale cache result")
            return DegradationResult(
                success=True,
                content=stale_result.response,
                source="stale_cache",
                model_used=None,
                latency=time.time() - start,
            )
        
        # ─── 层级 6: 最终拒绝 ───
        logger.error("All degradation strategies exhausted, rejecting request")
        return DegradationResult(
            success=False,
            content=None,
            source="rejected",
            model_used=None,
            latency=time.time() - start,
        )
    
    async def _try_queue(
        self,
        request: "AgentRequest",
        retry_after: float,
    ) -> DegradationResult | None:
        """尝试将请求加入队列"""
        try:
            result = await asyncio.wait_for(
                self.queue.enqueue_and_wait(request),
                timeout=min(retry_after + 10, 30),  # 最多等 30 秒
            )
            return DegradationResult(
                success=True,
                content=result,
                source="queued",
                model_used="gpt-4o",
                latency=retry_after,  # 近似等待时间
            )
        except asyncio.TimeoutError:
            return None
```

### 完整的速率限制协调器

将上述所有组件整合为一个完整的速率限制系统：

```python
# rate_limit_orchestrator.py — 完整速率限制协调器
import asyncio
import logging
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)


@dataclass
class AgentRequest:
    """Agent 请求——包含速率限制所需的所有上下文"""
    request_id: str
    tenant_id: str
    tenant_tier: str = "default"
    messages: list[dict] = field(default_factory=list)
    estimated_tokens: int = 0
    estimated_input_tokens: int = 0
    is_input_heavy: bool = False
    priority: int = 0  # 0=最高, 10=最低
    max_steps: int = 5
    
    def clone(self) -> "AgentRequest":
        return AgentRequest(
            request_id=self.request_id + "_degraded",
            tenant_id=self.tenant_id,
            tenant_tier=self.tenant_tier,
            messages=self.messages,
            estimated_tokens=self.estimated_tokens // 2,
            estimated_input_tokens=self.estimated_input_tokens // 2,
            is_input_heavy=self.is_input_heavy,
            priority=self.priority + 1,
            max_steps=self.max_steps,
        )


class RateLimitOrchestrator:
    """完整的速率限制协调器——集成所有层
    生产级用法:
    
    orchestrator = RateLimitOrchestrator(
        redis_client=redis,
        monthly_budget=50_000_000,  # 50M Token/月
        api_key="sk-xxx",
        openai_rpm=10000,
        openai_tpm=2000000,
        tenant_configs={
            "premium": {"rpm": 500, "tpm": 100000},
            "standard": {"rpm": 100, "tpm": 20000},
            "free": {"rpm": 20, "tpm": 4000},
        },
    )
    """
    
    def __init__(
        self,
        redis_client,
        monthly_budget: int,
        api_key: str,
        openai_rpm: int = 10000,
        openai_tpm: int = 2000000,
        tenant_configs: dict[str, dict] | None = None,
        global_concurrency: int = 500,
        adaptive: bool = True,
    ):
        # Token 估算器
        self.token_estimator = AgentAwareTokenEstimator()
        
        # 多层速率限制器
        self.multi_limiter = MultiLayerRateLimiter(redis_client)
        
        # Layer 4: 成本预算
        self.multi_limiter.add_layer(
            CostBudgetLayer(
                redis_client,
                monthly_token_budget=monthly_budget,
            )
        )
        
        # Layer 3: 租户公平性
        self.multi_limiter.add_layer(
            TenantFairnessLayer(
                redis_client,
                default_rpm=100,
                tier_config=tenant_configs,
            )
        )
        
        # Layer 2: API Key 限制
        self.multi_limiter.add_layer(
            APIKeyLimitLayer(
                redis_client,
                api_key=api_key,
                rpm=openai_rpm,
                tpm=openai_tpm,
            )
        )
        
        # Layer 1: 全局预算
        self.multi_limiter.add_layer(
            GlobalBudgetLayer(
                redis_client,
                global_tpm=openai_tpm * 2,  # 全局 = API Key 限制 × 2
                global_concurrency=global_concurrency,
            )
        )
        
        # 自适应限流
        self.adaptive = adaptive
        if adaptive:
            self.adaptive_limiter = AdaptiveRateLimiter(
                redis_client,
                key="global",
                min_rate=100,
                max_rate=openai_rpm,
            )
        
        # 优雅降级
        self.concurrency_limiter = TokenAwareConcurrencyLimiter(
            max_concurrent_tokens=openai_tpm // 10
        )
    
    async def should_allow(
        self,
        messages: list[dict],
        tenant_id: str,
        tenant_tier: str = "default",
        max_steps: int = 5,
        tool_results: list[dict] | None = None,
        memory_docs: list[str] | None = None,
    ) -> LimitResult:
        """检查请求是否允许——在调用 LLM API 之前调用"""
        
        # 1. 估算 Token 消耗
        estimate = self.token_estimator.estimate_agent_request(
            messages=messages,
            tool_results=tool_results,
            memory_docs=memory_docs,
            expected_steps=max_steps,
        )
        
        request = AgentRequest(
            request_id=f"req_{id(messages)}",
            tenant_id=tenant_id,
            tenant_tier=tenant_tier,
            messages=messages,
            estimated_tokens=estimate.total,
            estimated_input_tokens=estimate.input_tokens,
            is_input_heavy=estimate.input_tokens > estimate.total * 0.8,
            max_steps=max_steps,
        )
        
        # 2. 自适应限流检查
        if self.adaptive:
            allowed_rate = await self.adaptive_limiter.get_allowed_rate()
            # 如果自适应限流器显著降低了速率，检查是否需要排队
            # (这里简化处理，实际需要更精细的队列控制)
        
        # 3. 并发限制检查（Token 感知）
        if not await self.concurrency_limiter.try_acquire_token(
            estimate.total, timeout=1.0
        ):
            return LimitResult(
                decision=LimitDecision.QUEUE,
                layer="concurrency",
                reason=f"Token concurrency limit exceeded",
                retry_after=5.0,
            )
        
        # 4. 逐层速率限制检查
        result = await self.multi_limiter.check(request)
        
        if result.decision != LimitDecision.ALLOW:
            self.concurrency_limiter.release_token(estimate.total)
        
        return result
    
    async def record_success(
        self,
        messages: list[dict],
        actual_tokens: int,
        latency: float,
        was_rate_limited: bool = False,
    ):
        """记录一次成功的 API 调用"""
        estimate = self.token_estimator.estimate_from_messages(messages)
        
        request = AgentRequest(
            request_id=f"record_{id(messages)}",
            tenant_id="system",
            messages=messages,
            estimated_tokens=estimate.total,
        )
        
        await self.multi_limiter.record(request, actual_tokens)
        
        if self.adaptive:
            await self.adaptive_limiter.record_result(
                latency=latency,
                status_code=200,
                is_rate_limited=was_rate_limited,
            )
    
    async def handle_rate_limited(
        self,
        request: AgentRequest,
        limit_result: LimitResult,
    ) -> LimitResult:
        """被限流后的处理——自动尝试降级"""
        handler = GracefulDegradationHandler(
            primary_llm=None,  # 外部注入
            fallback_llm=None,
            semantic_cache=None,
            rate_limiter=self.multi_limiter,
            queue=None,
        )
        
        if self.adaptive:
            await self.adaptive_limiter.record_result(
                latency=0,
                status_code=429,
                is_rate_limited=True,
            )
        
        return limit_result
```

## 能力边界与限制

### 1. 分布式计数的精度 vs 性能权衡

```
精度-性能权衡谱:

高精度 ─────────────────────────────────── 低精度
│                                            │
▼                                            ▼

精确计数 (Redis Sorted Set)        近似计数 (Redis HyperLogLog/CMS)
  ┌─────────────────────┐           ┌─────────────────────┐
  │ 每个请求存一条记录     │           │ 概率数据结构          │
  │ 误差: 0%             │           │ 误差: 1-3%           │
  │ 内存: O(N)           │           │ 内存: O(1)           │
  │ TPM 下 10K 请求:     │           │ TPM 下 10K 请求:     │
  │   ~1MB/分钟          │           │   ~2KB               │
  │ 适合: 小规模精确控制   │           │ 适合: 大规模近似控制   │
  └─────────────────────┘           └─────────────────────┘

典型选择: 同时使用两者
  RPM 维度 → 滑动窗口 (精确，请求量少)
  TPM 维度 → 近似计数 (Token 量大，允许误差)
```

### 2. Token 估算误差

```python
# token_estimation_error.py — Token 估算的误差分析
"""
Token 预估算的误差来源:

1. 输出 Token 无法预知
   └── 只能假设 max_tokens 或根据历史统计
   └── 误差: ±30-50%

2. 中文字符与 Token 的非线性关系
   └── 英文: 1 Token ≈ 3-4 字符
   └── 中文: 1 Token ≈ 1-2 字符
   └── 中英混合: 难以精确估算

3. 工具调用结果的 Token 消耗
   └── Agent 在调用工具之前不知道结果长度
   └── 一次搜索引擎调用可能返回 1K 或 100K Token
   └── 误差: ±50-200%

4. 多步推理的 Token 消耗链
   └── 每一步的输出成为下一步的输入
   └── 误差随步数增加而指数级放大
   └── 5 步推理: 估算误差可达 ±100-300%

缓解策略:
  ├── 对输出 Token 使用保守估计 (max_tokens × 0.8)
  ├── 对工具结果使用最大长度截断 (max 4000 Token)
  ├── 使用历史统计校准 (同类型请求的实际 Token 消耗均值)
  └── 预留 20% 的余量在速率限制中 (safety margin)
"""

# 实际工程中的做法：设置安全余量 (safety margin)
SAFETY_MARGIN = 0.20  # 预留 20% 的余量

def adjusted_limit(provider_limit: int) -> int:
    """应用安全余量后的限制——留出缓冲防止误差导致的超限"""
    return int(provider_limit * (1 - SAFETY_MARGIN))

# 例如: TPM 限制 2,000,000 → 实际使用限制 1,600,000
# 剩余的 400,000 TPM 作为误差缓冲
```

### 3. 分布式时钟偏差

```
时钟偏差对速率限制的影响:

         时间线
         │
Redis ───┤  t=10.0  ← Redis 服务器时间
         │
Pod A ───┤  t=10.2  ← Pod A 时钟快 0.2s (请求过早计入窗口)
         │
Pod B ───┤  t=9.8   ← Pod B 时钟慢 0.2s (请求过晚计入窗口)
         │

窗口边界偏移:
  预期窗口: [0s, 60s)
  实际窗口 (Pod A): [-0.2s, 59.8s)  ← 窗口偏移 0.2s
  实际窗口 (Pod B): [0.2s, 60.2s)   ← 窗口偏移 -0.2s

  最坏情况: A 和 B 的窗口完全不重叠，有效速率翻倍
  缓解: 所有时间以 Redis 时间为准，不在客户端判断窗口
```

### 4. 降级链的完整决策流程

```
速率受限时的决策树:

请求进入
│
├── 检查 Layer 4 (成本预算)
│   ├── 未超 → 继续
│   └── 超限 → 检查是否能降级模型
│       ├── 可降级 → 自动切换到 Haiku/GPT-4o-mini
│       └── 不可降级 → 拒绝 (429)
│
├── 检查 Layer 3 (租户公平性)
│   ├── 未超 → 继续
│   └── 超限 → 检查租户排队优先级
│       ├── 高优先级 (付费) → 从其他租户借调配额
│       └── 低优先级 (免费) → 排队/拒绝
│
├── 检查 Layer 2 (API Key)
│   ├── 未超 → 继续
│   └── 超限 → 检查是否有备用 API Key
│       ├── 有备用 → 切换到备用 Key (轮转)
│       └── 无备用 → 排队/返回缓存/拒绝
│
├── 检查 Layer 1 (全局并发)
│   ├── 未超 → 允许请求
│   └── 超限 → 排队
│       ├── 队列有空位 → 排队等待 (返回预计等待时间)
│       └── 队列已满 → 返回降级缓存结果
│
└── 全部通过 → 请求允许，执行后消耗计入所有层
```

## 与其他方案对比

### 速率限制算法对比

| 算法 | 精度 | 内存开销 | 突发容忍 | 实现复杂度 | 适应 Agent 场景 |
|------|------|---------|---------|-----------|---------------|
| **固定窗口 (Fixed Window)** | 低 | O(1) | 差 | 极简 | 不推荐 |
| **滑动日志 (Sliding Log)** | 最高 | O(N) | 好 | 中 | 短时精确控制 |
| **滑动窗口计数器** | 中高 | O(N) | 好 | 低 | **推荐通用方案** |
| **Token Bucket** | 中 | O(1) | 最好 | 低 | **推荐 Agent 场景** |
| **Leaky Bucket** | 中 | O(1) | 无 | 低 | Agent 不适用 |
| **GCRA** | 高 | O(1) | 好 | 高 | 高性能场景 |

**各算法详解：**

```
固定窗口 (Fixed Window)
  ┌─────────────────────────────────────────┐
  │ 原理: 每 X 秒重置计数器                   │
  │ 优点: 最简实现，只需一个 Redis counter    │
  │ 缺点: 窗口边界突发 (2x 瞬时限流)           │
  │ 内存: O(1) — 1 个 key                    │
  │ Agent: ❌ 边界突发对 Agent 的长会话有害    │
  └─────────────────────────────────────────┘

滑动日志 (Sliding Log)
  ┌─────────────────────────────────────────┐
  │ 原理: 记录每个请求的时间戳                 │
  │ 优点: 最精确，无边界效应                   │
  │ 缺点: 内存占用与请求数成正比               │
  │ 内存: O(N) — N 个请求                     │
  │   N=10K → ~320KB/分钟                   │
  │   N=100K → ~3.2MB/分钟                  │
  │ Agent: 适合小规模精确控制                  │
  └─────────────────────────────────────────┘

滑动窗口计数器 (Sliding Window Counter)
  ┌─────────────────────────────────────────┐
  │ 原理: 前一个窗口 × 重叠比例 + 当前窗口     │
  │ 优点: 内存高效，边界平滑                   │
  │ 缺点: 近似计算，有少量误差                  │
  │ 内存: O(1) — 2 个 counter                 │
  │ Agent: ✅ 推荐的通用方案                   │
  │   精度足够，实现简单                       │
  └─────────────────────────────────────────┘

Token Bucket (令牌桶)
  ┌─────────────────────────────────────────┐
  │ 原理: 恒定速率补充 Token，突发时可消费      │
  │ 优点: 允许短时突发，自然匹配 LLM 使用模式    │
  │ 缺点: 不保证速率完全平滑                    │
  │ 内存: O(1) — 2 个值 (tokens, timestamp)   │
  │ Agent: ✅✅ 最适合 Agent 场景               │
  │   原因: Agent 工作负载天然突发性             │
  │         Token Bucket 允许突发 + 平滑       │
  │         两者完美匹配                        │
  └─────────────────────────────────────────┘

Leaky Bucket (漏桶)
  ┌─────────────────────────────────────────┐
  │ 原理: 请求进入队列，恒定速率流出             │
  │ 优点: 输出最平滑                           │
  │ 缺点: 无法应对突发 (队列满即丢)             │
  │ 内存: O(队列长度)                          │
  │ Agent: ❌ 不适用                          │
  │   原因: Agent 需要突发能力                  │
  │         漏桶刻意平滑，丢弃突发请求            │
  └─────────────────────────────────────────┘

GCRA (Generic Cell Rate Algorithm)
  ┌─────────────────────────────────────────┐
  │ 原理: 基于理论到达时间 (TAT) 的精确控制     │
  │ 优点: O(1) 内存，精确到毫秒级               │
  │ 缺点: 实现复杂，理解成本高                   │
  │ 内存: O(1) — 1 个值 (TAT)                 │
  │ Agent: ⚠ 高性能场景适用                   │
  │   (如 Redis 实例间毫秒级调度)              │
  └─────────────────────────────────────────┘
```

### 集中式 vs 分布式速率限制

| 维度 | 集中式 (Redis) | 分布式 (本地) |
|------|---------------|-------------|
| 精度 | 精确 (全局视图) | 近似 (每个实例独立) |
| 延迟 | +1~3ms (网络往返) | 0 (本地内存) |
| 可用性 | Redis 宕机 → 限流失效 | 实例独立运行 |
| 扩展性 | Redis 瓶颈 (单线程) | 线性扩展 |
| 复杂性 | 需要 Redis 运维 | 零外部依赖 |
| 适合场景 | 需要全局精确控制 | 允许 ±20% 误差 |

**混合策略：** 大多数生产系统同时使用两者

```python
# hybrid_limiter.py — 混合速率限制
class HybridRateLimiter:
    """混合速率限制：本地 + 中央
    本地: 低延迟，预处理
    中央: 精确控制，全局协调
    """
    
    def __init__(
        self,
        redis_client,
        key: str,
        local_rate: int = 100,      # 本地：最多处理 100 RPM
        local_burst: int = 20,       # 本地：允许 20 突发
        central_rate: int = 1000,    # 中央：全局 1000 RPM
    ):
        # 本地：Token Bucket (低延迟)
        self.local = LocalTokenBucket(local_rate, local_rate + local_burst)
        # 中央：滑动窗口 (精确)
        self.central = SlidingWindowRateLimiter(
            redis_client, 
            key, 
            central_rate, 
            60
        )
    
    async def allow_request(self) -> bool:
        # 先检查本地 (99% 的请求在此决定)
        if not self.local.consume():
            return False
        
        # 再检查中央 (同步协调)
        allowed, _ = await self.central.allow_request()
        if not allowed:
            # 中央拒绝，但本地已消耗——返回 Token
            # 简化处理：本地 Token 很快补充，影响小
            return False
        
        return True
    
    # 优化: 本地桶定期从中央同步
```

### Provider 特定的速率限制特性

```
OpenAI:
  三维度限制: RPM + TPM + IPM (Input Tokens Per Minute)
  Tier 1:   200 RPM / 40K TPM / 40K IPM  (免费用户)
  Tier 2:  1K RPM  / 80K TPM / 80K IPM
  Tier 3:  2K RPM  / 160K TPM / 160K IPM
  Tier 4:  5K RPM  / 600K TPM / 600K IPM
  Tier 5:  10K RPM / 2M TPM / 2M IPM     (最高)
  特点: 
  ├── 达到任一维度即触发 429
  ├── Rate limit 是 per-organization 的 (共享 API Key)
  ├── 响应头: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset
  └── 建议: TPM 是现代 Agent 最常达到的限制

Anthropic:
  二维度限制: RPM + TPM
  Tier 1:   50 RPM / 10K TPM
  Tier 2:   200 RPM / 50K TPM
  Tier 3:   500 RPM / 100K TPM
  Tier 4:   1K RPM / 200K TPM
  Tier 5:   2K RPM / 400K TPM
  特点:
  ├── 没有 IPM 单独限制 (输入和输出共享 TPM)
  ├── 对长上下文更友好 (200K 上下文窗口)
  ├── 429 响应包含 retry-after-ms 头
  └── Anthropic 的限流更倾向于"软"——请求可能被延迟而不是拒绝

Google (Gemini):
  三维度: RPM + TPM + RPD (Requests Per Day)
  Free:     60 RPM / 1M TPM / 1,500 RPD
  Pay-as-you-go: 2K RPM / 4M TPM / 无 RPD 限制
  特点:
  ├── RPD 是额外的维度——对高频 Agent 不友好
  ├── 某些模型有 GenAI 平台限制 (独立于 API 限制)
  └── TPM 限制比 OpenAI 宽松，但 RPM 限制更严格

速率限制键映射策略:
  ┌─────────────────────────────────────────────┐
  │ Agent 系统应该为每个 Provider 保存            │
  │ 独立的速率限制状态                            │
  │                                              │
  │ 示例:                                        │
  │   rate_limit:openai:sk-xxx:rpm               │
  │   rate_limit:openai:sk-xxx:tpm               │
  │   rate_limit:anthropic:sk-yyy:rpm            │
  │   rate_limit:anthropic:sk-yyy:tpm            │
  │   rate_limit:gemini:zzz:rpm                  │
  │   rate_limit:gemini:zzz:tpm                  │
  │   rate_limit:gemini:zzz:rpd                  │
  └─────────────────────────────────────────────┘
```

## 工程优化方向

### 方向一：预测性速率限制

在达到限制之前主动降低速率，避免触发 429：

```python
# predictive_limiter.py — 预测性速率限制
class PredictiveRateLimiter:
    """预测性速率限制
    根据近期消耗趋势预测何时会达到限制，提前降速
    
    核心思想:
    不等到撞墙 (429) 再处理，而是预测墙在哪里
    """
    
    def __init__(self, redis_client, key: str, limit: int):
        self.redis = redis_client
        self.key = key
        self.limit = limit
        self.history = deque(maxlen=300)  # 最近 5 分钟的秒级数据
    
    async def predict_limit_time(self) -> float | None:
        """预测达到限制的时间"""
        if len(self.history) < 60:
            return None
        
        # 计算移动平均消耗率 (Token/秒)
        recent = list(self.history)[-60:]  # 最近 60 秒
        avg_rate = sum(recent) / len(recent)
        
        if avg_rate <= 0:
            return None
        
        # 获取当前消耗
        current = await self.redis.get(f"{self.key}:current") or 0
        current = int(current)
        
        remaining = self.limit - current
        if remaining <= 0:
            return 0  # 已经达到限制
        
        # 预测剩余时间
        predicted_seconds = remaining / avg_rate
        return predicted_seconds
    
    async def should_pre_reduce(self) -> bool:
        """是否应该提前降速"""
        time_to_limit = await self.predict_limit_time()
        
        if time_to_limit is None:
            return False
        
        # 如果预计在 30 秒内达到限制，提前降速
        THRESHOLD = 30  # 秒
        return time_to_limit < THRESHOLD
    
    async def get_suggested_rate(self) -> float:
        """获取建议的速率"""
        time_to_limit = await self.predict_limit_time()
        
        if time_to_limit is None or time_to_limit > 60:
            return 1.0  # 全速
        
        # 线性降速: 剩余时间越少，速率越低
        # 距离限制 30 秒时 → 降到 80%
        # 距离限制 10 秒时 → 降到 50%
        # 距离限制 0 秒时 → 降到 10%
        reduction_factor = max(0.1, time_to_limit / 30)
        return reduction_factor
```

### 方向二：批量感知速率限制

Agent 系统可以积累相似请求，使用 Provider 的 Batch API 提高效率：

```python
# batch_aware_limiter.py — 批量感知速率限制
class BatchAwareRateLimiter:
    """批量感知速率限制
    积累相似的 Agent 请求，在速率限制窗口内批量发送
    
    适用场景:
    - Embedding 请求 (容易批量)
    - 相同 System Prompt 的 Chat 请求
    - 非实时的后台 Agent 任务
    """
    
    BATCH_WINDOW = 0.5  # 500ms 的批量窗口
    
    def __init__(self, redis_client, batch_api, max_batch_size: int = 20):
        self.redis = redis_client
        self.batch_api = batch_api
        self.max_batch_size = max_batch_size
        self.pending: dict[str, list] = defaultdict(list)
        self.locks: dict[str, asyncio.Lock] = defaultdict(asyncio.Lock)
    
    async def submit_with_batching(
        self,
        batch_key: str,
        request: dict,
    ) -> dict:
        """提交请求，自动批量化"""
        async with self.locks[batch_key]:
            self.pending[batch_key].append(request)
            
            if len(self.pending[batch_key]) >= self.max_batch_size:
                # 批量满了，立即发送
                return await self._flush_batch(batch_key)
        
        # 等待批量窗口
        await asyncio.sleep(self.BATCH_WINDOW)
        
        async with self.locks[batch_key]:
            if self.pending[batch_key]:
                return await self._flush_batch(batch_key)
    
    async def _flush_batch(self, batch_key: str) -> list[dict]:
        """发送批量请求"""
        batch = self.pending.pop(batch_key, [])
        if not batch:
            return []
        
        # 批量 API 调用消耗 1 个 RPM 配额而不是 N 个
        # ⚠ 但 Token 消耗是各请求之和
        results = await self.batch_api(batch)
        return results
```

### 方向三：速率限制感知的调度

在接近限制时，调度器应该优先处理重要请求：

```python
# rate_aware_scheduler.py — 速率限制感知的请求调度
class RateAwareScheduler:
    """速率限制感知的请求调度器
    在接近速率限制时，优先处理高优先级请求
    
    调度策略:
    ├── 远低于限制 (剩余 > 50%): FIFO，不区分优先级
    ├── 接近限制 (剩余 10-50%): 高优先级优先
    ├── 非常接近限制 (剩余 < 10%): 只处理最高优先级
    └── 达到限制 (剩余 = 0): 全部排队或拒绝
    """
    
    async def schedule(self, request: AgentRequest) -> LimitDecision:
        # 获取当前速率限制状态
        remaining_ratio = await self._get_remaining_ratio()
        
        if remaining_ratio > 0.5:
            # 资源充足: 直接放行
            return LimitDecision.ALLOW
        
        elif remaining_ratio > 0.1:
            # 资源紧张: 高优先级优先
            if request.priority <= 2:  # CRITICAL 或 HIGH
                return LimitDecision.ALLOW
            else:
                return LimitDecision.QUEUE
        
        elif remaining_ratio > 0:
            # 资源非常紧张: 只处理最关键的请求
            if request.priority == 0:  # CRITICAL
                return LimitDecision.ALLOW
            else:
                return LimitDecision.QUEUE
        
        else:
            # 资源耗尽
            return LimitDecision.REJECT
```

### 方向四：成本感知的速率限制

不同的请求使用不同的 Token 预算：

```python
# cost_aware_limiter.py — 成本感知速率限制
class CostTier:
    """成本层级定义"""
    CHEAP = "cheap"        # Haiku/GPT-4o-mini: $0.15-0.60/1M Token
    STANDARD = "standard"  # Sonnet/GPT-4o: $3-5/1M Token
    EXPENSIVE = "expensive" # Opus: $15-75/1M Token


class CostAwareRateLimiter:
    """成本感知的速率限制器
    不同成本层级的请求有不同的消耗速率和预算
    
    公式:
      cost_weight = token_count × model_cost_multiplier
      限制: sum(cost_weight over window) ≤ max_cost_weight
    
    效果:
      同样消耗 1000 Token:
      - Haiku: cost_weight = 1 (基准)
      - Sonnet: cost_weight = 20 (20倍)
      - Opus: cost_weight = 100 (100倍)
      
      → 一个 Opus 请求消耗的配额 = 100 个 Haiku 请求
      → 自动激励使用更经济的模型
    """
    
    COST_MULTIPLIER = {
        "gpt-4o": 20,
        "gpt-4o-mini": 1,
        "gpt-4-turbo": 15,
        "claude-3-opus": 100,
        "claude-3-sonnet": 20,
        "claude-3-haiku": 1,
    }
    
    # 示例：每月预算 $10000
    # Haiku 可以消费 10M Token
    # Sonnet 只能消费 500K Token
    # Opus 只能消费 100K Token
    
    def __init__(self, redis_client, budget: int, period_days: int = 30):
        self.redis = redis_client
        self.daily_budget = budget / period_days
    
    async def get_cost_weight(self, model: str, tokens: int) -> int:
        """获取成本权重——影响速率限制消耗"""
        multiplier = self.COST_MULTIPLIER.get(model, 10)
        return tokens * multiplier
    
    async def check_cost_budget(
        self, model: str, estimated_tokens: int, tenant_id: str
    ) -> bool:
        """检查成本预算是否充足"""
        weight = await self.get_cost_weight(model, estimated_tokens)
        key = f"cost_weight:daily:{tenant_id}:{time.strftime('%Y%m%d')}"
        
        current = await self.redis.get(key) or 0
        current = int(current)
        
        return (current + weight) <= self.daily_budget
```

## 适用场景判断

### 如何确定速率限制的数值

速率限制的具体数值取决于三个维度：

```
如何确定 RPM/TPM 限制:

1. 从 Provider 规格出发:
   ┌─────────────────────────────────────┐
   │ OpenAI Tier 5: 10K RPM, 2M TPM     │
   │ 减去 20% 安全余量:                  │
   │ 有效限制: 8K RPM, 1.6M TPM         │
   │ → 这是硬上限，不可超过               │
   └─────────────────────────────────────┘

2. 从成本预算出发:
   ┌─────────────────────────────────────┐
   │ 月度预算: $10,000                   │
   │ GPT-4o: $5/1M input                 │
   │ 每月可用 Token: 10,000 / 5 * 1M    │
   │              = 2,000M input Token   │
   │ 日均预算: 2,000M / 30 ≈ 67M Token  │
   │ 对应 TPM (日): 67M / 1440 ≈ 46K    │
   │ → 如果 Provider 限制 > 预算限制     │
   │   则预算限制是实际瓶颈                │
   └─────────────────────────────────────┘

3. 从 SLO 要求出发:
   ┌─────────────────────────────────────┐
   │ P99 响应时间 < 5s                   │
   │ 平均 LLM 调用延迟: 2s               │
   │ 最大重试次数: 2                     │
   │ → 队列长度应 < (5 - 2 - 2) / 2s    │
   │   = 0.5 → 最多 1 个排队请求         │
   │ → 这意味着几乎不能有队列堆积          │
   │ → 速率限制必须非常保守               │
   └─────────────────────────────────────┘

最终限制 = min(Provider 限制, 预算限制, SLO 限制)
```

### 速率限制策略选择

根据系统的规模和要求选择不同的策略：

```
                        Agent 速率限制策略选择
                        ┌─────────────────────┐
                        │                     │
          ┌─────────────┤  用户量 / 并发数     ├─────────────┐
          │             │                     │             │
          ▼             └─────────────────────┘             ▼
    小规模 (< 100)          中等 (100-10K)          大规模 (> 10K)
         │                      │                        │
         ▼                      ▼                        ▼
  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐
  │ 单层 Token    │    │ 多层限制器        │    │ 自适应 + 多层     │
  │ Bucket       │    │ (Redis 集中)      │    │ + 预测 + 成本     │
  │ (本地内存)    │    │ + 简单降级        │    │ + AI 辅助调度     │
  │ + asyncio    │    │ + 租户隔离        │    │ + 多 Provider     │
  │ Semaphore    │    │                  │    │ 自动切换          │
  └──────┬───────┘    └──────┬───────────┘    └────────┬─────────┘
         │                   │                         │
         ▼                   ▼                         ▼
  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │   Token Bucket + Sliding Window 组合                     │
  │   这是 LLM Agent 最常用的生产模式                         │
  │                                                         │
  │   Token Bucket: 容量 = 突发容忍度                         │
  │   Sliding Window: 精确控制持续速率                        │
  │                                                         │
  │   组合效果: ⎧ Token Bucket 吸收短时突发                    │
  │            ⎨ Sliding Window 防止长期超限                    │
  │            ⎩ 两者配合覆盖 Agent 的所有负载模式              │
  └─────────────────────────────────────────────────────────┘
```

### 量化评估：不同策略的成本和效果

```
场景: 100 并发 Agent 会话，每个会话平均 3 次 LLM 调用
      Provider: OpenAI Tier 5 (10K RPM / 2M TPM)
      运行 24 小时

策略                      429 错误率    Token 浪费    用户影响     运维成本
────                      ─────────    ──────────    ────────     ────────
无限制 (裸奔)               15-25%      30-50%        频繁超时     最低
固定窗口                    8-15%       15-25%        窗口边界卡顿   低
单机 Token Bucket          5-10%       10-20%        偶尔卡顿      低
Redis 滑动窗口              1-3%         3-8%         平滑         中
多层限制器                  0.5-1%       1-3%         几乎无感      中
自适应 + 多层              0.1-0.5%     0.5-2%       无感          高
自适应 + 预测 + 成本感知    <0.1%        <0.5%        完美         高
```

### 反模式清单

```
❌ "在 Agent 代码里加上 time.sleep() 来限流"
   sleep 阻塞事件循环，影响所有其他 Agent
   → 使用 asyncio.sleep() + Semaphore

❌ "所有 API Key 共享一个速率限制器"
   一个 Key 用完配额后不影响其他 Key
   → 每个 Key 独立限制器

❌ "用 Token 计数代替请求计数"
   两者都需要——RPM 和 TPM 是独立的限制维度
   → 同时跟踪 RPM 和 TPM

❌ "先发请求，收到 429 再退避"
   Agent 的上下文窗口在等待期间被浪费
   → 在发送前进行预测性限制

❌ "速率限制只做全局不做租户"
   一个租户的突发流量会耗尽所有配额
   → 必须做租户级别的隔离

❌ "Redis 宕机就无限流"
   Redis 不可用时应使用本地备份限制器
   → 实现 fallback 到本地 Token Bucket

❌ "所有请求使用相同的速率限制"
   不同模型消耗的 Token 和成本差异可达 100x
   → 使用成本加权速率限制
```

### 生产环境检查清单

```
速率限制系统上线检查清单:

□ 1. 所有四个限制层均已实现 (预算/租户/Key/全局)
□ 2. Token 估算器设置了 20% 安全余量
□ 3. 每个 LLM Provider 有独立的限制状态
□ 4. Redis 操作使用 Lua 脚本保证原子性
□ 5. 降级链完整: 缓存 → 小模型 → 排队 → 过期缓存 → 拒绝
□ 6. 429 响应包含 Retry-After 头 (或等效信息)
□ 7. 自适应限流已启用，根据错误率和延迟自动调整
□ 8. 并发限制与速率限制同时使用 (两者互补)
□ 9. Redis 宕机时有本地 fallback 策略
□ 10. 所有限制指标已接入监控和告警
□ 11. 成本预算限制有手动覆盖机制 (紧急提额)
□ 12. 限流日志包含足够信息用于事后分析

监控仪表盘指标:
  - 各层拒绝率趋势 (Layer 1-4)
  - 当前有效速率 vs 最大速率
  - 自适应限流的动态速率变化
  - 降级链各环节命中率
  - 429 错误率趋势
  - 租户级别的配额使用率
  - Token 消耗预测 vs 实际消耗
  - 成本消耗 vs 预算剩余
```
