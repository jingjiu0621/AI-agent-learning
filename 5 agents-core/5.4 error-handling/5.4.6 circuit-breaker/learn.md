# 5.4.6 circuit-breaker — 熔断机制

## 简单介绍

熔断（Circuit Breaker）是一种防止级联故障的机制。当 Agent 检测到某个服务持续失败时，"断开"对该服务的调用，快速失败而非继续等待，同时防止故障扩散到整个系统。

## 基本原理

### 熔断器三状态

```
正常（CLOSED）
  │ 连续失败数 > 阈值
  ↓
断开（OPEN）
  │ 等待超时时间
  ↓
半开（HALF_OPEN）
  │ 尝试一次请求
  ├── 成功 → 回到 CLOSED
  └── 失败 → 回到 OPEN
```

### 状态转换示例

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.state = "CLOSED"
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = 0
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenError("Service unavailable")
        
        try:
            result = func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        except:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise
```

## 背景与演进

熔断模式来自分布式系统（如 Netflix Hystrix），在 Agent 系统中用于保护 Agent 不受外部服务故障的影响，同时防止 Agent 在故障服务上浪费时间和 Token。

## 核心矛盾

**故障检测的及时性 vs 误判率**：
- 阈值太低 → 容易误判（临时抖动就熔断）
- 阈值太高 → 检测到故障时已经造成了大量浪费

## 主流优化方向

1. **自适应阈值**：根据服务的正常故障率动态调整阈值
2. **分阶梯断**：不同严重程度的不同熔断级别
3. **熔断降级联动**：熔断触发后自动切换到降级方案
4. **请求采样**：在 OPEN 状态也偶尔放行少量请求测试恢复
5. **熔断感知路由**：熔断后自动路由到备用服务

## 实现挑战

1. **状态共享**：多 Agent 实例如何共享熔断状态
2. **慢调用检测**：调用不失败但特别慢，是否算"故障"
3. **级联恢复**：多个熔断的服务恢复时可能引发二次雪崩
4. **测试熔断逻辑**：熔断触发和恢复路径在测试中难以覆盖

## 能力边界

- 熔断保护的是调用方而不是服务方
- 熔断不能替代重试，两者是互补的（重试在前，熔断在后）
- 熔断决策机制本身也可能故障

## 核心优势

熔断防止 Agent 在已故障的服务上 **"无限等待和重试"**，是保障整体系统稳定性的关键。

## 工程优化

1. 每个外部服务独立熔断器（隔离故障域）
2. 熔断阈值按服务重要性差异化（核心服务阈值更低）
3. 熔断事件记录到监控和告警
4. 半开状态的试探请求设置超时

## 场景判断

适合熔断的场景：
- Agent 依赖的外部 API 服务质量不稳定
- 高并发场景下保护下游服务
- 关键业务需要避免级联故障

不需要熔断的场景：
- 本地确定性操作（不会"故障")
- Agent 只有一个外部依赖
