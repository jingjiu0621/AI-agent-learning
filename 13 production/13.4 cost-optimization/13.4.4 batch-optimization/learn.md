# 批量优化 (Batch Optimization)

**将多个请求打包为批次统一提交，利用 API 批量端点与请求级聚合来摊薄固定开销，可在大规模非实时场景下将推理成本降低 50% 以上。**

---

## 1. 基本原理

### 1.1 批处理降本的三条路径

| 路径 | 机制 | 典型降幅 |
|------|------|----------|
| **Provider Batch API** | 服务商利用空闲算力异步处理，将折扣转嫁给用户 | 50%（OpenAI Batch API） |
| **共享前缀优化** | 多个请求共用相同的 system prompt + tool definitions，减小实际传输量与 KV 缓存命中 | 30–70% tokens |
| **固定开销摊薄** | 网络往返、认证、请求解析等固定开销在一次批处理中分摊 | 视批量大小而定 |

### 1.2 同步批处理 vs 异步批处理

```
同步批处理:
  Req ──┐
  Req ──┤──→ [Batch API] ──→ 等待所有完成 ──→ 返回全部结果
  Req ──┘

异步批处理:
  Req ──┐
  Req ──┤──→ [Submit Batch] ──→ 立即返回 batch_id
  Req ──┘                            ↓
                            轮询 /results 端点
                            每个结果独立就绪后回调
```

- **同步**：适用于必须等待全部结果才能继续的场景（如批量分类），简单但延迟受最慢请求制约。
- **异步**：适用于可容忍推迟获取结果的场景（如离线数据处理），能充分利用批处理折扣。

### 1.3 延迟-成本权衡

批处理的核心代价是 **等待时间**：系统需收集足够请求才能形成批次。此等待时间与批量大小呈正相关，而成本节省与批量大小呈边际递减关系。在实际系统中，存在一个最优批量窗口（通常为 5–60 秒），在此窗口内等待带来的延迟损失恰好被成本节省抵偿。

### 1.4 何时使用批处理

| 适合批处理 | 不适合批处理 |
|-----------|-------------|
| 日活高、请求密度大的系统 | 实时语音/聊天交互 |
| 非实时后台任务（日志分析、内容审核） | 交互式编码助手 |
| 离线数据处理 / ETL 流水线 | 低流量场景（等待填充批次得不偿失） |
| 批量评测（eval pipeline） | 用户期望 1 秒内响应的场景 |
| RAG 离线嵌入生成 | 单次调用、偶发性请求 |

---

## 2. 批处理模式

### 2.1 Provider Batch API

#### OpenAI Batch API

- **折扣**：标准模型 50%，无需额外配置
- **窗口**：24 小时内完成
- **流程**：提交 JSONL → 获取 `batch_id` → 轮询 `/v1/batches/{batch_id}` → 下载结果
- **文件格式**：每行一个完整的 ChatCompletion 请求 JSON

```
POST /v1/batches
{
  "input_file_id": "file-...",
  "endpoint": "/v1/chat/completions",
  "completion_window": "24h"
}
```

#### AWS Bedrock Batch Inference

- 基于 S3 输入/输出：将请求写入 S3 → 创建 Batch Inference Job → 从输出 S3 读取结果
- 适合已使用 AWS 生态的团队

#### Google Vertex AI Batch Prediction

- 支持 BigQuery 源和目标
- 输出可写入 BigQuery 表或 Cloud Storage

#### 对比表

| 服务商 | API 端点 | 折扣幅度 | 完成窗口 | 输入方式 |
|--------|---------|----------|----------|---------|
| OpenAI | `/v1/batches` | 50% | 24h | JSONL 文件上传 |
| AWS Bedrock | CreateInferenceJob | 约 40–50% | 无保证 | S3 JSONL |
| Google Vertex | BatchPredictionJob | 最高 40% | 无保证 | BigQuery / GCS |

### 2.2 请求级批处理 (Request-Level Batching)

**共享系统提示词批处理**：当多个用户查询使用相同的 system prompt 时，将请求合并发送，利用 prompt caching 减少重复计算。

```
用户 A: system=A + "南京天气"
用户 B: system=A + "北京天气"
    ↓ 合并
batch = [system=A+"南京天气", system=A+"北京天气"]
    ↓ 共享前缀命中 KV 缓存
```

**工具执行批处理**：Agent 在单轮推理中发起多个并行工具调用（function calling 的 parallel_tool_calls 参数），利用 API 的并行工具执行能力。

**嵌入批处理**：将多条文本合并为一次嵌入 API 调用，显著降低每向量成本。

```python
# 批处理嵌入
texts = ["doc1", "doc2", "doc3", "doc4"]
response = client.embeddings.create(
    model="text-embedding-3-small",
    input=texts  # 一次调用处理多条
)
```

### 2.3 提示级批处理 (Prompt-Level Batching)

**多轮对话打包**：将多轮用户输入拼接为单次请求中的多条消息，一次性获得全部回复。

**批量分类**：在一个 prompt 中列出多个待分类项，利用 LLM 的文本理解能力一次性输出所有分类结果。

```
System: 对以下每条文本进行情感分类（positive/negative/neutral），
        以 JSON 数组格式返回。
User:   ["产品质量非常好",
         "发货太慢了，差评",
         "还可以"]
```

**共享前缀**：将相同的 system prompt、tool definitions 提取为公共前缀，使请求在传输和推理时共享该部分 token。

---

## 3. 核心技术与实现

### 3.1 OpenAI Batch API 提交 + 轮询

```python
import json, time
from openai import OpenAI

client = OpenAI()

# 准备 batch 请求
batch_requests = []
for i, prompt in enumerate(prompts):
    batch_requests.append({
        "custom_id": f"req-{i:04d}",
        "method": "POST",
        "url": "/v1/chat/completions",
        "body": {
            "model": "gpt-4o-mini",
            "messages": [{"role": "user", "content": prompt}],
            "max_tokens": 1000,
        }
    })

# 写入 JSONL
with open("/tmp/batch_input.jsonl", "w") as f:
    for req in batch_requests:
        f.write(json.dumps(req) + "\n")

# 上传文件
file_obj = client.files.create(
    file=open("/tmp/batch_input.jsonl", "rb"),
    purpose="batch"
)

# 创建 Batch Job
batch = client.batches.create(
    input_file_id=file_obj.id,
    endpoint="/v1/chat/completions",
    completion_window="24h"
)

# 轮询完成
batch_id = batch.id
while True:
    status = client.batches.retrieve(batch_id)
    if status.status == "completed":
        break
    elif status.status == "failed":
        raise Exception("Batch failed")
    time.sleep(60)

# 下载结果
result = client.files.content(status.output_file_id)
for line in result.text.strip().split("\n"):
    data = json.loads(line)
    print(f"{data['custom_id']}: {data['response']['choices'][0]['message']}")
```

### 3.2 请求队列 + 批量窗口

```python
import asyncio
from dataclasses import dataclass, field
from typing import List, Callable

@dataclass
class BatchQueue:
    window_seconds: float = 10.0
    max_batch_size: int = 100
    queue: List[dict] = field(default_factory=list)
    flush_callback: Callable = None

    async def add(self, request: dict):
        self.queue.append(request)
        if len(self.queue) >= self.max_batch_size:
            await self._flush()

    async def _flush(self):
        if not self.queue:
            return
        batch = self.queue[:]
        self.queue.clear()
        await self.flush_callback(batch)

    async def run(self):
        while True:
            await asyncio.sleep(self.window_seconds)
            await self._flush()
```

### 3.3 优先级感知批处理

```python
import asyncio
from heapq import heappush, heappop

class PriorityBatchQueue:
    def __init__(self, urgent_threshold: float = 1.0):
        self.normal_queue = []
        self.urgent_threshold = urgent_threshold
        self.urgent_queue = []

    async def add(self, request: dict, priority: str = "normal"):
        if priority == "urgent":
            self.urgent_queue.append(request)
            await self._flush_urgent()
        else:
            self.normal_queue.append(request)

    async def _flush_urgent(self):
        """紧急请求立即提交，不等待批处理窗口"""
        if self.urgent_queue:
            batch = self.urgent_queue[:]
            self.urgent_queue.clear()
            await submit_batch(batch)  # 直接调用 API，不走批量折扣

    async def flush_normal(self):
        """普通请求累积后统一批处理"""
        if self.normal_queue:
            batch = self.normal_queue[:]
            self.normal_queue.clear()
            await submit_batch_api(batch)  # 走批量折扣
```

### 3.4 Agent 批处理模式

```python
class AgentBatchProcessor:
    """
    收集相似的 Agent 推理任务，打包为批次提交。
    适用于：批量 Agent 评测、离线场景模拟、批量内容审核。
    """
    async def collect_and_batch(self, tasks: List[dict]):
        # 按 task_type 分组，同类任务共享 system prompt
        groups = {}
        for task in tasks:
            key = task.get("type", "default")
            groups.setdefault(key, []).append(task)

        results = {}
        for task_type, group in groups.items():
            system_prompt = self._get_system_prompt(task_type)
            batch_prompts = [t["prompt"] for t in group]

            batch_responses = await self._batch_call(
                system_prompt=system_prompt,
                user_prompts=batch_prompts,
            )
            for i, t in enumerate(group):
                results[t["id"]] = batch_responses[i]

        return results
```

---

## 4. 场景对比

| 场景 | 推荐模式 | 预期降本 | 延迟影响 |
|------|---------|----------|---------|
| 离线数据标注 / 文本分类 | Provider Batch API | 50% | 数小时（可接受） |
| RAG 离线嵌入生成 | 嵌入批处理 | 40–60% | 秒级（无实时要求） |
| 在线 Agent 工具并行调用 | 请求级批处理 | 10–20% | 几乎无影响 |
| Agent 评测流水线 | Agent 批处理器 + Batch API | 50–70% | 分钟至小时级 |
| 实时对话 | 不做批处理 | 0% | — |
| 批量内容审核 | Provider Batch API | 50% | 24h 内 |

### 架构示意

```
                    ┌─────────────────────┐
                    │   Incoming Requests  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Priority Classifier│
                    └────┬──────────┬─────┘
                         │          │
                   urgent │          │ normal
                         │          │
              ┌──────────▼──┐  ┌────▼──────────┐
              │  Direct API │  │Batch Collector │
              │ (no batch)  │  │(window: 10s)  │
              └──────┬──────┘  └────┬──────────┘
                     │              │
              ┌──────▼──────┐  ┌────▼──────────┐
              │  Response   │  │Provider Batch │
              │ (real-time) │  │API (50% off)  │
              └─────────────┘  └────┬──────────┘
                                    │
                              ┌────▼──────────┐
                              │Poll & Distribute│
                              │  (24h window)  │
                              └────────────────┘
```

---

## 5. 能力边界

### 5.1 供应商批量大小限制

| 服务商 | 单批次最大请求数 | 单文件最大大小 |
|--------|----------------|--------------|
| OpenAI | 50,000 条 | 100 MB (JSONL) |
| AWS Bedrock | 无硬限制 | 取决于 S3 |
| Google Vertex | 无硬限制 | 取决于 GCS |

### 5.2 延迟-成本帕累托曲线

```
成本节省
  ↑
  |   #####
  |  #     ####
  | #          ###
  |#              ###
  |                   ##### (继续增大批量收益递减)
  +──────────────────────────→ 等待时间（延迟）
```

并非批量越大越好。超过某个点后，增加等待时间带来的成本节省微乎其微。每个系统都需通过实验找到自己的帕累托最优窗口。

### 5.3 错误处理与部分失败

- **全批次失败**：Provider 返回 batch failed（如格式错误、超时）。需要重试整个批次。
- **部分失败**：批次中个别请求失败。需解析 `response.status_code`，对失败项单独重试。
- **空结果**：批次完成但 output_file 中缺少某些 `custom_id`。实现超时补偿机制。

### 5.4 幂等性与去重

批处理必须保证每条请求被处理且仅被处理一次：

```python
def deduplicate(requests: list) -> list:
    seen = set()
    unique = []
    for req in requests:
        cid = req.get("custom_id")
        if cid and cid not in seen:
            seen.add(cid)
            unique.append(req)
    return unique
```

---

## 6. 工程实践

### 6.1 批量大小与窗口的动态调优

参数 | 调优方向 | 监控指标
-----|---------|---------
批量窗口 | 从 5s 开始，逐步增大至延迟不可接受 | 请求排队时间
批量大小 | 从 10 开始，增大至成本递减拐点 | 单请求平均成本
并发批次 | 根据 API 配额限制调整 | 429 错误率

### 6.2 混合策略落地要点

1. **分类路由**：请求到达时先判断优先级，紧急走实时通道，非紧急入批量队列。
2. **动态窗口**：低峰期使用较长窗口（30–60s）积攒更多请求；高峰期缩短窗口（5–10s）防止队列膨胀。
3. **超时降级**：批量窗口超时后未满批次也强制提交，避免请求无限等待。
4. **结果缓存**：相同输入的请求去重，直接返回缓存结果，节省批次槽位。
5. **监控告警**：跟踪 Batch API 完成率、平均等待时间、部分失败率，设置阈值告警。

### 6.3 Agent 场景专项建议

- **并行工具调用**：启用 `parallel_tool_calls=true`，让 Agent 在一次推理中发起多个工具调用，减少往返次数。
- **批量评测**：将 Agent 评测流水线改为批量提交，利用 Batch API 50% 折扣大幅降低评测成本。
- **离线数据清洗**：对于日志分析、对话审核等非实时任务，全天数据统一在凌晨提交 Batch API，次日检查结果。
- **嵌入生成**：RAG 管道的文档分块嵌入使用 `client.embeddings.create(input=[...])` 批处理，避免逐条调用。

### 6.4 常见陷阱

- **批处理 + streaming 不兼容**：Batch API 不支持 streaming，必须使用非流式请求。
- **批处理超时**：24 小时内未完成可能失败，需设计重试逻辑。
- **部分结果利用**：即使批次尚未全部完成，可先返回已就绪的结果（异步批次模式）。
- **成本监控偏移**：Batch API 折扣是隐式的（使用 Batch 端点自动享受），需在成本报表中区分 Batch 与非 Batch 用量。
