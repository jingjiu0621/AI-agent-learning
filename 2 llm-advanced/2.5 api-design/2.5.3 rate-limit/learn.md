# 速率限制处理与请求排队

## 简单介绍

速率限制（Rate Limiting）是 LLM API 提供商（OpenAI, Anthropic, Google 等）保护其基础设施的机制——限制客户端在单位时间内能发出的请求数量。Agent 系统可能同时发起大量请求（工具调用、多 Agent 并行），因此必须正确处理限流。

## 常见的限流类型

| 类型 | 维度 | 示例 | 影响 |
|------|------|------|------|
| RPM（每分钟请求数） | 请求频率 | 60 RPM | 高并发场景 |
| TPM（每分钟 Token 数） | Token 消耗 | 100k TPM | 大批量场景 |
| RPD（每天请求数） | 每日总量 | 10k RPD | 长期使用 |
| Concurrency（并发数） | 同时进行的请求 | 5 并发 | 并行 Agent |

## 限流检测

API 返回 429 状态码（Too Many Requests）指示限流：

```python
async def check_rate_limit(response):
    """从响应头检测限流信息"""
    return {
        "is_limited": response.status_code == 429,
        "retry_after": response.headers.get("Retry-After"),  # 秒
        "remaining": response.headers.get("X-RateLimit-Remaining"),
        "reset_at": response.headers.get("X-RateLimit-Reset"),
        "limit": response.headers.get("X-RateLimit-Limit")
    }
```

## 请求排队策略

### 1. 令牌桶算法（Token Bucket）

```python
import asyncio

class TokenBucket:
    """令牌桶速率限制器"""
    def __init__(self, rate: float, burst: int):
        """
        rate: 每秒恢复的令牌数
        burst: 最大突发请求数
        """
        self.rate = rate
        self.burst = burst
        self.tokens = burst
        self.last_refill = time.time()
    
    async def acquire(self):
        """获取一个令牌，如不足则等待"""
        while True:
            self._refill()
            if self.tokens >= 1:
                self.tokens -= 1
                return
            # 等待直到有足够的令牌
            wait_time = 1.0 / self.rate
            await asyncio.sleep(wait_time)
    
    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        self.tokens = min(self.burst, self.tokens + elapsed * self.rate)
        self.last_refill = now
```

### 2. 排队执行器

```python
class RateLimitedExecutor:
    """支持限流的请求排队执行器"""
    def __init__(self, rpm: int = 60, concurrency: int = 5):
        self.semaphore = asyncio.Semaphore(concurrency)
        self.min_interval = 60.0 / rpm  # 请求间隔
        self.last_request = 0
    
    async def execute(self, fn, *args, **kwargs):
        async with self.semaphore:
            await self._wait_interval()
            return await fn(*args, **kwargs)
    
    async def _wait_interval(self):
        """确保请求间隔"""
        elapsed = time.time() - self.last_request
        if elapsed < self.min_interval:
            await asyncio.sleep(self.min_interval - elapsed)
        self.last_request = time.time()
```

### 3. 分级限流

不同优先级的请求使用不同的配额：

```python
class PriorityRateLimiter:
    def __init__(self):
        # 高优先级：用户交互请求（关键路径）
        self.high_priority = TokenBucket(rate=5, burst=3)
        # 低优先级：后台预取、预处理
        self.low_priority = TokenBucket(rate=2, burst=1)
    
    async def acquire(self, priority: str = "normal"):
        if priority == "high":
            await self.high_priority.acquire()
        elif priority == "low":
            if self.low_priority.tokens < 1:
                # 低优先级没有配额时，借用高优先级的
                await self.high_priority.acquire()
            else:
                await self.low_priority.acquire()
        else:  # normal
            await self.high_priority.acquire()
```

## Agent 系统的限流场景

| 场景 | 限流影响 | 策略 |
|------|---------|------|
| 单 Agent 调用 | 低并发 | 简单的退避即可 |
| 多 Agent 并行 | 高并发 | 需要令牌桶 + 队列 |
| 工具调用很多 | Token 消耗高 | 监控 TPM |
| 批量处理 | 长时间运行 | TPM + RPD 双重控制 |
| 重试带来的流量放大 | 重试加重限流 | 幂等键 + 退避 |

## 核心挑战

1. **重试风暴**：限流后，所有等待的请求同时重试，加剧限流——需要用 Jitter + 队列控制
2. **TPM 难精确控制**：每次请求的 Token 消耗不同，TPM 控制比 RPM 更难
3. **不同模型的配额不同**：GPT-4 的配额通常比 GPT-4o 低——需要分别管理
4. **限流响应缓慢**：429 响应本身就消耗 Token 和配额

## 工程启示

- 在 Agent 系统中，优先按**并发数**控制，其次按请求频率
- 对于非交互式任务（大批量处理），使用**异步队列 + 消费池**模式，消费池的速率受限于 API 配额
- 监控 429 错误率——如果频繁出现，考虑升级 API Plan 或降低并发
- 流式请求的限流计算方式不同——一个流式连接算一次请求，但 Token 消耗持续计入
- 对 429 的响应，始终使用服务端的 Retry-After 头，而不是用自己的退避策略
