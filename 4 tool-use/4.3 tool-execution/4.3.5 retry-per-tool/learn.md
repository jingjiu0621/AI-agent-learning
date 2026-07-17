# 单工具级别重试策略

## 简单介绍

工具调用并不是一次就成功的——网络抖动、服务限流、临时故障都会导致失败。单工具级别的重试策略针对特定工具的特点设计不同的重试行为，在不引起副作用的前提下尽可能提高成功率。

## 重试决策树

```
工具失败
  ├── 不可重试的错误（参数错误、权限不足）
  │   └── 直接返回错误，不重试
  ├── 可重试的错误（网络超时、5xx 错误）
  │   ├── 幂等工具 → 直接重试
  │   └── 非幂等工具 → 需要检查是否已执行
  └── 不确定的错误（连接提前关闭）
      └── 重试 + 幂等键（idempotency key）
```

## 重试策略对比

| 策略 | 说明 | 适用场景 | 风险 |
|------|------|----------|------|
| 立即重试 | 失败后立即重试 1 次 | 网络闪断 | 无缓解期 |
| 固定间隔 | 每次固定间隔后重试 | 简单场景 | 可能加剧负载 |
| 指数退避 | 间隔指数增长（1s, 2s, 4s, 8s...） | 通用场景 | 长等待 |
| 抖动退避 | 指数 + 随机偏移 | 分布式系统 | 略复杂 |
| 渐进退避 | 线性增长间隔 | 限流场景 | 慢恢复 |

### 带抖动的指数退避

```python
import random
import asyncio

async def retry_with_backoff(
    fn, 
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
):
    last_error = None
    for attempt in range(max_retries + 1):
        try:
            return await fn()
        except RetryableError as e:
            if attempt == max_retries:
                raise
            delay = min(base_delay * (2 ** attempt), max_delay)
            jitter = random.uniform(0, delay * 0.1)  # 10% 抖动
            await asyncio.sleep(delay + jitter)
            last_error = e
    raise last_error
```

## 工具幂等性检查表

重试前必须确认工具是否幂等：

| 工具类型 | 幂等? | 重试策略 |
|---------|-------|----------|
| 查询天气 | ✅ 是 | 安全重试 |
| 搜索数据库 | ✅ 是 | 安全重试 |
| 发送邮件 | ❌ 否 | 需要幂等键 |
| 创建订单 | ❌ 否 | 需要幂等键 |
| 删除用户 | ❌ 否 | 需要幂等键 |
| 生成图片 | 部分 - 相同参数可能不同结果 | 有限重试 |

## 幂等键实现

```python
import uuid

class IdempotentExecutor:
    def __init__(self):
        self.executed = {}  # key -> result
    
    async def execute(self, tool_name, args, idempotency_key=None):
        key = idempotency_key or f"{tool_name}:{hash(frozenset(args.items()))}"
        
        if key in self.executed:
            return self.executed[key]  # 返回之前的结果
        
        result = await call_tool(tool_name, args)
        self.executed[key] = result
        return result
```

## 重试条件配置

```python
retry_configs = {
    "search_database": {
        "max_retries": 2,
        "retryable_errors": [TimeoutError, ConnectionError],
        "non_retryable_errors": [PermissionError, ValidationError],
        "base_delay": 0.5,
        "max_delay": 5.0,
    },
    "call_external_api": {
        "max_retries": 3,
        "retryable_errors": [TimeoutError, ConnectionError, HTTPError(5xx)],
        "non_retryable_errors": [HTTPError(4xx)],
        "base_delay": 2.0,
        "max_delay": 30.0,
    },
}
```

## 核心挑战

1. **重试风暴**（Retry Storm）：所有 Agent 实例同时重试可能导致系统雪崩——用抖动 + 熔断器解决
2. **副作用**：非幂等工具重试可能产生重复操作——用幂等键解决
3. **LLM 感知重试**：LLM 也可能"重试"（发现出错后再次调用同一工具），需要与框架的重试协调
