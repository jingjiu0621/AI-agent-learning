# Batch API 与异步结果轮询

## 简单介绍

Batch API（批量 API）允许将大量请求一次性提交给 LLM 服务，服务端异步处理完成后统一返回结果。与实时推理不同，Batch API 通常有更低的成本（OpenAI Batch API 提供 **50% 折扣**）和更高的吞吐量，但延迟较数分钟到数小时。适合不需要实时响应的场景。

## 基本原理

```
实时 API:
  客户端 → 请求 → 服务端 → 实时处理 → 响应 → 客户端
  特点: 低延迟（秒级），标准价格

Batch API:
  客户端 → 批量请求 → 服务端 → 排队处理 → 轮询结果 → 客户端
  特点: 高延迟（分钟~小时级），50% 折扣
```

### OpenAI Batch API

```python
import json

# 1. 准备批量请求（JSONL 格式）
batch_requests = [
    {
        "custom_id": "req-001",
        "method": "POST",
        "url": "/v1/chat/completions",
        "body": {
            "model": "gpt-4o",
            "messages": [{"role": "user", "content": "北京天气"}],
            "max_tokens": 100
        }
    },
    {
        "custom_id": "req-002",
        "method": "POST",
        "url": "/v1/chat/completions",
        "body": {
            "model": "gpt-4o",
            "messages": [{"role": "user", "content": "上海天气"}],
            "max_tokens": 100
        }
    }
]

# 2. 上传文件
file = client.files.create(
    file=json.dumps({"requests": batch_requests}),
    purpose="batch"
)

# 3. 创建批处理
batch = client.batches.create(
    input_file_id=file.id,
    endpoint="/v1/chat/completions",
    completion_window="24h"
)

# 4. 轮询结果
while True:
    batch_status = client.batches.retrieve(batch.id)
    if batch_status.status == "completed":
        result_file = client.files.content(batch_status.output_file_id)
        results = json.loads(result_file)
        break
    elif batch_status.status == "failed":
        raise Exception(f"Batch failed: {batch_status}")
    await asyncio.sleep(60)  # 每分钟轮询一次
```

## 与实时 API 的对比

| 维度 | 实时 API | Batch API |
|------|---------|-----------|
| 延迟 | 毫秒~秒级 | 分钟~小时级 |
| 成本 | 标准价格 | 50-80%（折扣） |
| 并发 | 受限流限制 | 高（排队处理） |
| 适用 | 用户交互 | 后端批量处理 |
| 可靠性 | 独立请求 | 批量失败影响全部 |

## 异步结果轮询模式

```python
class BatchProcessor:
    """批量任务处理器"""
    def __init__(self, client, batch_interval=60):
        self.client = client
        self.batch_interval = batch_interval
        self.pending_batches = {}
    
    async def submit_batch(self, requests: list) -> str:
        """提交一批请求"""
        file = await self._upload_requests(requests)
        batch = await self.client.batches.create(
            input_file_id=file.id,
            endpoint="/v1/chat/completions",
            completion_window="24h"
        )
        self.pending_batches[batch.id] = {
            "status": "pending",
            "requests": len(requests)
        }
        return batch.id
    
    async def poll_all(self):
        """轮询所有待处理的批次"""
        while self.pending_batches:
            for batch_id in list(self.pending_batches.keys()):
                status = await self.client.batches.retrieve(batch_id)
                if status.status == "completed":
                    results = await self._download_results(status)
                    await self._dispatch_results(results)
                    del self.pending_batches[batch_id]
                elif status.status == "failed":
                    # 处理失败
                    del self.pending_batches[batch_id]
            
            if self.pending_batches:
                await asyncio.sleep(self.batch_interval)
```

## Agent 系统中的 Batch API 适用场景

| 场景 | 适用 Batch? | 说明 |
|------|------------|------|
| 用户实时对话 | ❌ | 需要实时响应 |
| 批量数据分类 | ✅ | 几万条数据，不需要即时 |
| 离线评估 | ✅ | 评估集推理，几小时可接受 |
| 批量工具调用 | 部分 | 如果工具调用没有状态依赖 |
| 大批量翻译 | ✅ | 非交互式，数量大 |
| 知识库构建 | ✅ | Embedding 生成等 |

## 核心挑战

1. **结果关联**：需要跟踪 batch 中每个请求和响应的对应关系——使用 custom_id
2. **部分失败**：batch 中部分请求可能失败，需要单独处理
3. **延迟不可控**：Batch API 的处理时间取决于服务端排队情况
4. **依赖管理**：如果多个 batch 之间有依赖关系，顺序难以保证

## 工程启示

- Batch API 适合**离线**和**大量非交互式**的场景——成本比实时 API 节省约 50%
- 在 Agent 系统的评估管道中，Batch API 是最经济的选择
- 使用 custom_id 跟踪每个请求，便于结果关联和调试
- 设置合理的超时（Batch 最长 24 小时），超时后检查失败原因
- 不要因为 Batch 便宜就把所有请求都走 Batch——用户体验优先的请求必须走实时 API
