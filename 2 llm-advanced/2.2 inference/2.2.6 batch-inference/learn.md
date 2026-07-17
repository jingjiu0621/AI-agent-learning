# 批量推理与动态 Batching

## 简单介绍

批量推理（Batch Inference）是将多个独立的请求打包在一起，一次性交给 GPU 处理的技术。由于 GPU 的并行计算特性，同时处理 N 个请求的耗时通常只比处理 1 个请求多一点点——因此能够显著提升吞吐量。动态 Batching（Continuous Batching）进一步优化了"等待低效"的问题——不用等所有请求都到齐才开始批处理。

## 基本原理

### 为什么批量处理更快

```
单条处理（每个请求单独跑）:
  请求A: |--- 10ms ---|
  请求B:         |--- 10ms ---|
  请求C:                  |--- 10ms ---|
  总耗时: 30ms, 吞吐量: 3/30ms = 100 req/s

批处理（三个请求一起跑）:
  请求A: |--|
  请求B: |--|  ← 共享一次前向传播
  请求C: |--|
  总耗时: 12ms, 吞吐量: 3/12ms = 250 req/s
```

吞吐量提升的关键：GPU 一次处理的请求数量增加时，计算量的增长是**次线性**的。

## 静态 Batching vs 动态 Batching

### 静态 Batching（传统）

等待 N 个请求到齐后才一起处理：

```
请求A 到了... 等待
请求B 到了... 等待
请求C 到了... 齐了！开始批处理
```

**问题**：如果某个请求很慢，所有请求都得等它。

### 动态（连续）Batching

Continuous Batching（vLLM 首创）的核心思想：**当一个请求生成完毕后，立即插入新请求**。

```
时间 →
┌──────────────────────────────────────────┐
│ Batch 1: A(生成中), B(生成中), C(生成中)   │
│ C 完成！                                  │
│ Batch 1: A(继续), B(继续)                 │
│ Batch 1 + D(新加入): A, B, D              │
│ A 完成！                                  │
│ Batch 2: B(继续), D(继续) + E(新加入)     │
└──────────────────────────────────────────┘
```

### Iteration-level Scheduling

更精细的调度——每次前向传播后重新安排批次：

```python
class ContinuousBatchingScheduler:
    def __init__(self, max_batch_size=8):
        self.running = []  # 正在生成中的请求
        self.waiting = []  # 等待中的请求
        self.max_batch_size = max_batch_size
    
    def schedule(self):
        """每次前向传播后重新安排批次"""
        # 移除已完成的请求
        self.running = [r for r in self.running if not r.finished]
        
        # 填充新的请求到空位
        empty_slots = self.max_batch_size - len(self.running)
        while empty_slots > 0 and self.waiting:
            self.running.append(self.waiting.pop(0))
            empty_slots -= 1
        
        return self.running  # 当前这个周期的处理批次
```

## 批处理优化的关键维度

| 维度 | 挑战 | 优化方案 |
|------|------|----------|
| 序列长度对齐 | 不同请求的序列长度不同 | Padding / 动态 padding |
| Memory 管理 | 每个请求的 KV Cache 不同 | PagedAttention（分页管理） |
| 调度延迟 | 频繁调度消耗 CPU 资源 | 预热 + 缓存调度策略 |
| Fairness | 长序列请求"饿死"短序列 | 按长度加权调度 |

## 工程实践

```python
class BatchExecutor:
    def __init__(self, model, max_batch=8):
        self.model = model
        self.max_batch = max_batch
        self.pending = []
    
    async def submit(self, request):
        """提交推理请求"""
        if len(self.pending) >= self.max_batch:
            await self._flush()
        
        self.pending.append(request)
        
        if len(self.pending) >= self.max_batch:
            await self._flush()
    
    async def _flush(self):
        """批量处理所有待处理的请求"""
        if not self.pending:
            return
        
        batch = self.pending
        self.pending = []
        
        # 填充到相同长度
        padded = pad_sequences(batch)
        
        # 一次前向传播
        outputs = self.model.generate(padded)
        
        # 分发结果
        for req, out in zip(batch, outputs):
            await req.callback(out)
```

## 核心挑战

1. **请求排队延迟**：为了凑批处理而等待，可能增加单个请求的延迟
2. **GPU 显存碎片**：不同批次的不同序列长度导致 KV Cache 碎片
3. **抢占和优先级**：高优请求应该插队还是排队？

## 与 Agent 系统的关系

- Agent 系统通常有多个 LLM 调用（工具调用 + 推理），Batch 策略决定了系统吞吐量
- 使用推理服务（vLLM, TGI, Triton）时，Batch 策略由服务管理，Agent 不需要关心
- 如果自建推理服务，Continuous Batching 是必须的——它比 Static Batching 的吞吐量高 2-3x
