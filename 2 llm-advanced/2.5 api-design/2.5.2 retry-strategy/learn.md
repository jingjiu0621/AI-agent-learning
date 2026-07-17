# 重试策略：指数退避、jitter、幂等性保证

## 简单介绍

LLM API 调用可能因为网络问题、服务端限流、临时故障等原因失败。重试策略决定了如何在失败后安全重试。**指数退避**确定了重试间隔，**jitter**增加了随机性防止重试风暴，**幂等性保证**防止重试导致重复操作。

## 重试的必要性

| 失败原因 | 频率 | 是否可重试 | 处理方式 |
|---------|------|-----------|----------|
| 网络超时 | 低-中 | 是 | 直接重试 |
| 429 限流 | 中 | 是 | 退避后重试 |
| 5xx 服务端错误 | 低 | 是 | 退避后重试 |
| 4xx 客户端错误 | 低 | 否 | 修复请求 |
| 无效响应 | 低 | 是 | 重试直到有效格式 |

## 核心重试策略

### 1. 指数退避

```python
import time
import random

async def retry_with_backoff(fn, max_retries=3, base_delay=1.0):
    """指数退避重试"""
    last_exception = None
    for attempt in range(max_retries + 1):
        try:
            return await fn()
        except RetryableError as e:
            if attempt == max_retries:
                raise last_exception
            
            delay = base_delay * (2 ** attempt)  # 1, 2, 4, 8...
            await asyncio.sleep(delay)
            last_exception = e
```

### 2. 带 Jitter 的指数退避

Jitter 解决了"所有客户端在同一时刻重试"的重试风暴问题：

```python
async def retry_with_jitter(fn, max_retries=3, base_delay=1.0, max_delay=60.0):
    """带抖动的指数退避"""
    for attempt in range(max_retries + 1):
        try:
            return await fn()
        except RetryableError as e:
            if attempt == max_retries:
                raise
            
            # 指数退避
            delay = min(base_delay * (2 ** attempt), max_delay)
            # 加入 ±50% 的随机抖动
            jitter = random.uniform(-delay * 0.5, delay * 0.5)
            await asyncio.sleep(delay + jitter)
```

### 3. 基于 HTTP 状态码的重试

```python
class LLMRetryHandler:
    def should_retry(self, response) -> bool:
        status = response.status_code
        if status == 429:  # 限流
            self.backoff = min(self.backoff * 2, 60)
            return True
        elif status >= 500:  # 服务端错误
            return True
        elif status == 408:  # 超时
            return True
        elif 400 <= status < 500:  # 客户端错误
            return False  # 不重试
        return False
    
    def get_retry_delay(self, response) -> float:
        # 优先使用服务端建议的等待时间
        retry_after = response.headers.get("Retry-After")
        if retry_after:
            return float(retry_after)
        return self.backoff
```

## 幂等性保证

重试最关键的问题：**如果请求已成功，但响应在网络中丢失了怎么办？重试会导致重复操作吗？**

### 使用幂等键

```python
import uuid

class LLMClient:
    def __init__(self):
        self.executed_keys = set()  # 记录已执行的幂等键
    
    async def chat(self, messages, tools=None, idempotency_key=None):
        key = idempotency_key or str(uuid.uuid4())
        
        # 检查是否已执行
        if key in self.executed_keys:
            return self.cached_responses[key]  # 返回缓存的结果
        
        # 发送请求并包含幂等键
        response = await self._api_call(
            messages=messages,
            tools=tools,
            idempotency_key=key  # 服务端用此键去重
        )
        
        self.executed_keys.add(key)
        self.cached_responses[key] = response
        return response
```

## 完整重试策略实现

```python
class LLMClientWithRetry:
    def __init__(self, api_key, max_retries=3, base_delay=1.0):
        self.api_key = api_key
        self.max_retries = max_retries
        self.base_delay = base_delay
        self._executed = {}  # 幂等键缓存
    
    async def chat(self, messages, tools=None, idempotency_key=None):
        key = idempotency_key or str(uuid.uuid4())
        
        # 幂等性检查
        if key in self._executed:
            return self._executed[key]
        
        # 重试循环
        for attempt in range(self.max_retries + 1):
            try:
                response = await self._api_call(messages, tools)
                self._executed[key] = response
                return response
                
            except RateLimitError as e:
                if attempt == self.max_retries:
                    raise
                delay = float(e.headers.get("Retry-After", self.base_delay * (2 ** attempt)))
                await asyncio.sleep(delay + random.uniform(0, 1))
                
            except ServerError as e:
                if attempt == self.max_retries:
                    raise
                await asyncio.sleep(self.base_delay * (2 ** attempt) + random.uniform(0, 1))
                
            except TimeoutError:
                if attempt == self.max_retries:
                    raise
                # 超时：不确定是否执行了，谨慎不重试（或使用幂等键）
                continue
        
        raise MaxRetriesExceeded()
```

## 最佳实践

| 策略 | 推荐值 | 说明 |
|------|--------|------|
| 最大重试次数 | 3 次 | 超过 3 次后成功率提升不大 |
| 初始退避 | 1s | 从 1 秒开始 |
| 最大退避 | 60s | 避免无限等待 |
| Jitter 范围 | ±50% | 有效防止重试风暴 |
| 可重试错误 | 429, 5xx, 超时 | 4xx 客户端错误不可重试 |
| 幂等性 | 所有非幂等操作 | 特别是在 Agent 工具调用场景 |
