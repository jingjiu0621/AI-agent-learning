# 13.3.5 cache-warming -- 缓存预热

## 核心论点

**Agent 系统的冷启动问题比传统 web 服务严重得多——一个没有缓存的 Agent 在启动初期不仅响应慢，而且 Token 成本会飙升 2-3 倍。缓存在预热到稳定状态前，命中率从 0% 逐渐爬升，这个"冷窗口期"的长度直接决定了部署初期的用户体验和成本。缓存预热就是把冷窗口期从"自然缓慢填充"加速到"主动预填充"。**

在传统 Web 服务中，冷缓存意味着几次数据库查询的延迟——几十毫秒的代价。在 Agent 系统中，一次缓存未命中意味着一次完整的 LLM 调用：数秒的延迟，数千甚至数万 Token 的消耗。以一个典型的生产 Agent 为例，System Prompt 可能在 2000 Token 以上，工具定义文档可能达到 3000-5000 Token，一次完整的 Prompt 拼接加上 LLM 推理可能在 5000-10000 Token 级别。如果这些内容每次都要重新计算，成本差异是数量级的。

---

## 为什么 Agent 的冷启动问题更严重

### LLM 调用成本远高于 DB 查询

一条 Redis 查询耗时 1-5ms，一条 PostgreSQL 查询耗时 10-100ms。而一次 LLM 调用（即使是缓存未命中后的回填）耗时 500ms-5s，成本是前者的 50-1000 倍。这意味着 Agent 冷启动期的每一次缓存未命中，代价都异常高昂。

```
冷启动代价对比（每次未命中）:

  Web App DB 查询:     ~50ms    成本: 几乎可忽略
  Agent LLM 调用:      ~2000ms   成本: ~$0.003-0.03 per call
  倍数:                 40x       倍数: 100-1000x
```

### Agent 的常见模式变化少但计算成本高

Agent 系统中的许多计算模式具有"少变高成本"的特性：

- **System Prompt**: 部署后几乎不变，但每次拼接都要重复计算 Tokenization + KV Cache
- **工具定义 (Tools Definition)**: 每个 Agent 的工具集相对固定，但定义文档可能包含数百行 JSON Schema
- **Few-shot Examples**: 精选示例通常不会频繁变化，但作为 Prompt 的一部分反复传输
- **Function Call 结果解析**: 同一工具的返回结果结构相同，解析逻辑可复用

这些内容的共同点是：极少变化，但每次重新计算都很昂贵。它们天然适合预热缓存。

### 生产部署中的缓存池重建

在典型的生产部署场景中，缓存池会频繁重建：

```
滚动更新 (Rolling Update):
  Instance 1 (warm)          Instance 2 (warm)          Instance 3 (warm)
       |                           |                           |
  Instance 1' (cold)          Instance 2' (cold)          Instance 3' (cold)
       |        <- 逐个替换，每个新实例都要重新预热      |

蓝绿部署 (Blue-Green):
  [Blue Pool: warm]   ->   [Green Pool: cold]   ->   [Green Pool: warming...]
                                                              |
                                                    切换到 Green 时的瞬间冷启动
  
金丝雀发布 (Canary):
  10% new instances -> cold -> Team 需要决定: 先预热再接入流量? 还是边服务边预热?
```

每次部署都会创建一个全新的缓存池（如果缓存是本地且不与持久化存储绑定的），预热策略直接决定了部署期间的体验。

---

## 预热策略

### 静态预热（Static Warming）

静态预热基于历史数据预先确定哪些缓存条目需要加载。它不依赖实时流量，可提前规划。

**实现方式**:

1. **历史日志分析**: 从生产日志中提取过去 N 天的缓存访问模式
2. **热门 Key 提取**: 按访问频率排序，截取 Top-K 的缓存 Key
3. **离线生成**: 在部署前预计算这些 Key 对应的缓存值
4. **批量加载**: 服务启动时或启动前将预计算值写入缓存

**适用场景**:

- System Prompt 内容（Agent 行为定义、角色设定）
- 工具定义 Schema（每次调用都需序列化的 JSON Schema）
- 常见 FAQ 的 Embedding 向量
- 固定的 Few-shot Example 序列

**Python 示例**:

```python
import json
import pickle
from pathlib import Path
from typing import Any, Dict, List

class StaticWarmer:
    """基于历史日志的静态预热引擎"""
    
    def __init__(self, cache_client, history_path: str, top_k: int = 1000):
        self.cache = cache_client
        self.history_path = Path(history_path)
        self.top_k = top_k
    
    def analyze_history(self) -> Dict[str, int]:
        """从历史访问日志中提取热门 Key 及其频率"""
        key_freq: Dict[str, int] = {}
        
        for log_file in self.history_path.glob("access_log_*.jsonl"):
            with open(log_file) as f:
                for line in f:
                    record = json.loads(line)
                    key = record.get("cache_key")
                    if key:
                        key_freq[key] = key_freq.get(key, 0) + 1
        
        # 按频率降序排列
        sorted_keys = sorted(key_freq.items(), key=lambda x: -x[1])
        return dict(sorted_keys[:self.top_k])
    
    def generate_cache_values(self, hot_keys: Dict[str, int]) -> Dict[str, str]:
        """离线生成缓存值（模拟 LLM 响应预计算）"""
        values = {}
        for key in hot_keys:
            # 这里替换为实际的预计算逻辑
            # 例如: 对常见 Prompt 预计算 Token 计数、预拼接模板
            values[key] = self._precompute(key)
        return values
    
    def _precompute(self, key: str) -> str:
        """实际的预计算逻辑——可按 Key 类型分发"""
        if key.startswith("system_prompt:"):
            return self._compile_system_prompt(key)
        elif key.startswith("tool_schema:"):
            return self._serialize_tool_schema(key)
        elif key.startswith("embedding:"):
            return self._precompute_embedding(key)
        return ""
    
    def warm_up(self, values: Dict[str, str], ttl: int = 3600):
        """批量加载缓存条目"""
        pipeline = self.cache.pipeline()
        for key, value in values.items():
            pipeline.setex(key, ttl, value)
        pipeline.execute()
```

### 动态预热（Dynamic Warming）

动态预热根据实时流量模式和预测算法动态决定预热内容。相比静态预热，它能更好地适应流量变化。

**核心机制**:

```
实时流量 -> 滑动窗口计数器 -> 热点检测 -> 预取队列 -> 后台 Worker 填充缓存
                             |
                   预测模型(可选) -> 趋势分析 -> 提前预取
```

**渐进式预热（Staggered Warming）**: 避免所有缓存条目同时写入导致的"缓存惊群"，动态预热应分批次、限速执行。

```python
import asyncio
import time
from collections import defaultdict, deque
from typing import Set, Callable, Awaitable

class DynamicWarmer:
    """动态预热引擎——基于实时流量自适应预热"""
    
    def __init__(
        self, 
        cache_client,
        window_seconds: int = 60,
        hot_threshold: int = 10,
        warm_speed: int = 50,   # 每秒最多预热条目数
    ):
        self.cache = cache_client
        self.window = window_seconds
        self.threshold = hot_threshold
        self.speed = warm_speed
        self.access_counter: Dict[str, deque] = defaultdict(deque)
        self.previous_hot_keys: Set[str] = set()
        self._warming = False
    
    def record_access(self, key: str):
        """记录一次缓存访问"""
        now = time.time()
        self.access_counter[key].append(now)
        # 清理窗口外的记录
        while self.access_counter[key] and \
              self.access_counter[key][0] < now - self.window:
            self.access_counter[key].popleft()
    
    def detect_hot_keys(self) -> Set[str]:
        """检测当前窗口内的热点 Key"""
        now = time.time()
        hot_keys = set()
        for key, timestamps in self.access_counter.items():
            # 清理并检查
            while timestamps and timestamps[0] < now - self.window:
                timestamps.popleft()
            if len(timestamps) >= self.threshold:
                hot_keys.add(key)
        return hot_keys
    
    async def warm_cycle(self, fetcher: Callable[[str], Awaitable[str]]):
        """执行一次预热周期"""
        current_hot = self.detect_hot_keys()
        new_hot = current_hot - self.previous_hot_keys
        
        # 仅对新出现的热点 Key 执行预热
        # 已预热过的 Key 不需要重复预热（除非已过期）
        for key in new_hot:
            if await self.cache.exists(key):
                continue
            
            # 速率限制——渐进式预热的核心
            value = await fetcher(key)
            await self.cache.setex(key, 300, value)
            await asyncio.sleep(1.0 / self.speed)  # 限速
        
        self.previous_hot_keys = current_hot
    
    async def start(self, fetcher, interval: float = 10.0):
        """启动周期性预热"""
        self._warming = True
        while self._warming:
            await self.warm_cycle(fetcher)
            await asyncio.sleep(interval)
```

### 延迟预热（Lazy Warming）

延迟预热不在启动时做任何事，而是在首次缓存未命中时同步回填，同时异步预取相关联的缓存条目。

**适用场景**: 长尾查询、不可预测的热点、启动速度优先的场景。

```
首次请求 Key=A (MISS)
  -> 同步: 回填 Key=A 到缓存
  -> 异步: 预取 Key=A_related_1, Key=A_related_2, Key=A_related_3
  -> 下次请求 Key=A_related_1 (HIT)
```

```python
class LazyWarmer:
    """延迟预热——首次访问时回填 + 关联预取"""
    
    def __init__(self, cache_client, prefetch_radius: int = 3):
        self.cache = cache_client
        self.radius = prefetch_radius
    
    async def get_or_compute(
        self, 
        key: str, 
        compute_fn: Callable[[], Awaitable[str]],
        related_keys_fn: Callable[[str], List[str]]
    ) -> str:
        """获取缓存，未命中则计算并预取关联"""
        cached = await self.cache.get(key)
        if cached is not None:
            return cached
        
        # 同步计算当前 Key
        value = await compute_fn()
        await self.cache.setex(key, 300, value)
        
        # 异步预取关联 Key
        asyncio.create_task(
            self._prefetch_related(key, related_keys_fn)
        )
        
        return value
    
    async def _prefetch_related(
        self, 
        key: str, 
        related_fn: Callable[[str], List[str]]
    ):
        """预取关联缓存条目——不阻塞主流程"""
        related_keys = related_fn(key)
        for rel_key in related_keys[:self.radius]:
            if await self.cache.exists(rel_key):
                continue
            # 这里填入实际的关联计算逻辑
            # 例如: 同一 Session 的下一个预期 Prompt
            pass
```

---

## 预热数据源

| 数据源 | 内容 | 获取方式 | 更新频率 |
|--------|------|----------|----------|
| 生产日志 | 缓存 Key 访问频率、响应时间 | 日志采集系统（ELK/Loki） | 实时/准实时 |
| 离线评估集 | 标准测试查询集 | 标注/回放工具 | 每版本 |
| 用户行为轨迹 | 点击流、对话路径 | 埋点系统 | 每日 |
| 知识库内容 | 文档 Embedding、索引 | 离线 ETL Pipeline | 按需 |

```python
# 预热数据源的统一抽象
from enum import Enum
from dataclasses import dataclass, field
from typing import Generator

class DataSourceType(Enum):
    PRODUCTION_LOG = "production_log"
    OFFLINE_EVAL = "offline_eval"
    USER_BEHAVIOR = "user_behavior"
    KNOWLEDGE_BASE = "knowledge_base"

@dataclass
class WarmDataRecord:
    key: str
    value_generator: Callable[[], str]
    priority: int  # 0-100, 越高越优先预热
    ttl: int = 3600
    source: DataSourceType = DataSourceType.PRODUCTION_LOG

class DataSourceManager:
    """管理多个预热数据源的统一接口"""
    
    def __init__(self, sources: Dict[DataSourceType, Any]):
        self.sources = sources
    
    def get_warm_records(self) -> Generator[WarmDataRecord, None, None]:
        """聚合所有数据源的预热记录，按优先级排序"""
        records = []
        for source_type, source in self.sources.items():
            for record in source.fetch():
                record.source = source_type
                records.append(record)
        
        records.sort(key=lambda r: -r.priority)
        yield from records
```

---

## 预热 Pipeline 架构

```
+------------------+     +------------------+     +------------------+
|  Historical Logs  |     |  Scheduled Jobs  |     |  Knowledge Base  |
|  (ELK / Loki)     |     |  (Cron / Airflow)|     |  (Vector Store)  |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                         |
         v                        v                         v
+---------------------------------------------------------------+
|                    Analysis Layer                              |
|  +------------------+  +------------------+                   |
|  | Hot Key Detector |  | Trend Predictor  |                   |
|  | (频率统计/滑动   |  | (时序预测/ARIMA) |                   |
|  |  窗口计数)       |  |                  |                   |
|  +--------+---------+  +--------+---------+                   |
+-----------|-------------------|-------------------------------+
            v                   v
+---------------------------------------------------------------+
|                    Hot Key List                                |
|  +----------------------------------------------------------+ |
|  | [(key, priority, ttl), (key, priority, ttl), ...]        | |
|  | 排序: priority DESC, 截取: top N                          | |
|  +----------------------------------------------------------+ |
+----------------------------|----------------------------------+
                             |
                             v
+---------------------------------------------------------------+
|                    Cache Populator                             |
|  +----------------------------------------------------------+ |
|  |  Worker Pool (8-16 workers)                              | |
|  |  并发控制: Semaphore                                     | |
|  |  速率限制: Token Bucket                                  | |
|  |  分阶段: L3 -> L2 -> L1                                  | |
|  +----------------------------------------------------------+ |
+------------|-------------------|-------------------|----------+
             v                   v                   v
      +-----------+       +-----------+       +-----------+
      |  L3 Cache |       |  L2 Cache |       |  L1 Cache |
      |  (Disk)   |       |  (Redis)  |       |  (Memory) |
      +-----------+       +-----------+       +-----------+
```

---

## 避免预热惊群

预热惊群（Warming Thundering Herd）是指大量预热请求同时涌向后端服务，导致后端过载或缓存节点 OOM。这是预热过程中最常见也最危险的陷阱。

### 并发控制

```python
import asyncio
from typing import List, Tuple

class ControlledWarmer:
    """带并发控制的预热引擎"""
    
    def __init__(self, max_concurrency: int = 8, rate_per_second: int = 100):
        self.semaphore = asyncio.Semaphore(max_concurrency)
        self.rate_limiter = asyncio.Semaphore(rate_per_second)
        self._last_rate_reset = 0
    
    async def warm_batch(
        self, 
        entries: List[Tuple[str, str, int]]  # (key, value, ttl)
    ) -> int:
        """分批预热，带并发和速率控制"""
        warmed = 0
        
        async def _warm_one(key: str, value: str, ttl: int):
            async with self.semaphore:
                # 速率限制
                now = asyncio.get_event_loop().time()
                if now - self._last_rate_reset > 1.0:
                    self._last_rate_reset = now
                    # reset rate limiter (simplified)
                await self.cache.setex(key, ttl, value)
                return 1
        
        tasks = [_warm_one(k, v, t) for k, v, t in entries]
        results = await asyncio.gather(*tasks)
        return sum(results)
```

### 分阶段预热

预热不应该同时把所有层次的缓存填满。合理的顺序是：从最底层（最便宜、容量最大）开始，逐层向上。

```
分阶段预热策略:

Phase 0: 预计算（离线）
  [生成所有缓存 Value] -> 写入预热队列

Phase 1: L3 Cache (Disk/SSD)
  [大容量、低成本]     -> 填充 Embedding 向量、历史会话
  |--- 限速: 1000 items/sec ---|

Phase 2: L2 Cache (Redis)
  [中等容量、中等速度] -> 填充 LLM 响应、工具 Schema
  |--- 限速: 200 items/sec  ---|

Phase 3: L1 Cache (Local Memory)
  [小容量、极快速度]   -> 填充 System Prompt、高频查询
  |--- 限速: 50 items/sec   ---|

Phase 4: 预热验证
  [检查命中率]         -> 确认预热达标后接入流量
```

### Staggered Warming（交错预热）

将预热条目分成多个小批次，在时间上交错执行，避免瞬时负载尖峰。

```python
def staggered_warm_batches(
    hot_keys: List[str], 
    batch_size: int = 50, 
    delay_ms: int = 100
) -> Generator[List[str], None, None]:
    """生成交错预热批次"""
    for i in range(0, len(hot_keys), batch_size):
        yield hot_keys[i:i + batch_size]

# 使用
for batch in staggered_warm_batches(all_keys, batch_size=50):
    asyncio.run(warm_batch(batch))
    await asyncio.sleep(0.1)  # 100ms 间隔
```

---

## Python 代码示例：完整 CacheWarmer

```python
"""
cache_warmer.py -- 完整的缓存预热引擎

支持:
  - 静态预热（基于历史日志）
  - 动态预热（基于实时流量）
  - 并发控制（信号量 + 速率限制）
  - 进度追踪（tqdm）
  - 预热验证（预热后命中率检查）
"""

import asyncio
import json
import logging
import time
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Callable, Dict, List, Optional, Set, Tuple

logger = logging.getLogger(__name__)


@dataclass
class WarmingResult:
    total_keys: int = 0
    succeeded: int = 0
    failed: int = 0
    skipped: int = 0
    duration_seconds: float = 0.0
    hit_rate_before: float = 0.0
    hit_rate_after: float = 0.0
    
    @property
    def success_rate(self) -> float:
        if self.total_keys == 0:
            return 0.0
        return self.succeeded / self.total_keys * 100
    
    @property
    def cost_saved_per_hour(self) -> float:
        """估算预热后每小时节省的 LLM 调用成本"""
        # 假设每个缓存命中节省 1 次 LLM 调用 (~$0.01)
        hit_rate_increase = self.hit_rate_after - self.hit_rate_before
        requests_per_hour = 1000  # 假设值，应根据实际调整
        return hit_rate_increase * requests_per_hour * 0.01


class BaseWarmer(ABC):
    """预热引擎基类"""
    
    def __init__(
        self,
        cache_client,
        max_concurrency: int = 8,
        rate_limit: int = 100,
        ttl: int = 3600,
    ):
        self.cache = cache_client
        self.semaphore = asyncio.Semaphore(max_concurrency)
        self.rate_limit = rate_limit
        self.ttl = ttl
        self._warming_result = WarmingResult()
    
    @abstractmethod
    async def collect_keys(self) -> List[Tuple[str, Callable[[], str]]]:
        """收集需要预热 Key 列表及值生成函数"""
        ...
    
    async def warm_up(self) -> WarmingResult:
        """执行预热主流程"""
        start_time = time.time()
        
        # 1. 预热前的命中率
        self._warming_result.hit_rate_before = await self._measure_hit_rate()
        
        # 2. 收集预热 Key
        keys = await self.collect_keys()
        self._warming_result.total_keys = len(keys)
        logger.info(f"Collected {len(keys)} keys for warming")
        
        # 3. 执行预热
        async def _warm_one(key: str, value_fn: Callable[[], str]) -> bool:
            async with self.semaphore:
                try:
                    # 跳过已存在的缓存
                    if await self.cache.exists(key):
                        self._warming_result.skipped += 1
                        return True
                    
                    value = value_fn() if callable(value_fn) else value_fn
                    await self.cache.setex(key, self.ttl, value)
                    self._warming_result.succeeded += 1
                    return True
                except Exception as e:
                    logger.error(f"Failed to warm key {key}: {e}")
                    self._warming_result.failed += 1
                    return False
        
        # 分批执行以控制速率
        batch_size = self.rate_limit
        for i in range(0, len(keys), batch_size):
            batch = keys[i:i + batch_size]
            tasks = [_warm_one(k, v) for k, v in batch]
            await asyncio.gather(*tasks)
            if i + batch_size < len(keys):
                await asyncio.sleep(1)  # 每秒一批
        
        # 4. 预热后的命中率
        await asyncio.sleep(1)  # 等待缓存生效
        self._warming_result.hit_rate_after = await self._measure_hit_rate()
        self._warming_result.duration_seconds = time.time() - start_time
        
        return self._warming_result
    
    async def _measure_hit_rate(self, sample_keys: int = 100) -> float:
        """测量当前缓存命中率"""
        import random
        all_keys = [f"test:{i}" for i in range(sample_keys)]
        hits = 0
        for key in all_keys:
            if await self.cache.exists(key):
                hits += 1
        return hits / sample_keys if sample_keys > 0 else 0.0


class ProductionLogWarmer(BaseWarmer):
    """基于生产日志的静态预热——具体实现"""
    
    def __init__(self, cache_client, log_path: str, **kwargs):
        super().__init__(cache_client, **kwargs)
        self.log_path = log_path
    
    async def collect_keys(self) -> List[Tuple[str, Callable[[], str]]]:
        # 从日志文件读取热门 Key
        keys = []
        with open(self.log_path) as f:
            for line in f:
                record = json.loads(line)
                key = record["cache_key"]
                # 对于 Agent System Prompt 预热
                if key.startswith("agent:sys_prompt:"):
                    agent_id = key.split(":")[-1]
                    keys.append((
                        key,
                        lambda aid=agent_id: self._build_system_prompt(aid)
                    ))
        return keys
    
    def _build_system_prompt(self, agent_id: str) -> str:
        """预构建 System Prompt 的缓存值"""
        # 实际逻辑：拼接 Prompt 模板 + 工具定义
        return json.dumps({
            "agent_id": agent_id,
            "compiled_prompt": f"<compiled system prompt for {agent_id}>",
            "tool_definitions": f"<tool schemas for {agent_id}>",
            "token_count": 2048
        })


# === 使用示例 ===
async def main():
    import aioredis
    
    redis = await aioredis.from_url("redis://localhost:6379")
    
    warmer = ProductionLogWarmer(
        cache_client=redis,
        log_path="/var/log/agent/cache_keys.jsonl",
        max_concurrency=8,
        rate_limit=200,
        ttl=3600,
    )
    
    result = await warmer.warm_up()
    
    print(f"Warming completed:")
    print(f"  Total:    {result.total_keys}")
    print(f"  Succeeded: {result.succeeded}")
    print(f"  Failed:   {result.failed}")
    print(f"  Skipped:  {result.skipped}")
    print(f"  Duration: {result.duration_seconds:.2f}s")
    print(f"  Hit rate: {result.hit_rate_before:.1%} -> {result.hit_rate_after:.1%}")
    print(f"  Est. cost saved/h: ${result.cost_saved_per_hour:.2f}")

# asyncio.run(main())
```

---

## 预热效果评估

### 关键指标

| 指标 | 计算方法 | 目标值 |
|------|----------|--------|
| 预热后命中率提升 | `H_after - H_before` | > 40% |
| 预热覆盖率 | `warmed_keys / total_active_keys` | > 80% |
| 预热耗时 | 从开始到完成的秒数 | < 启动窗口期 |
| 预热开销 | 预热消耗的资源 / 无预热时的损失 | < 10% |
| ROI | `(节省成本 - 预热开销) / 预热开销` | > 5x |

### 成本节省计算

```
假设:
  - 每小时请求数: 10,000
  - 无预热时缓存命中率: 20%
  - 预热后缓存命中率: 65%
  - 每次 LLM 调用成本: $0.01 (约 1000 Token)
  - 每次缓存命中成本: $0.00001 (Redis 查询)

每小时 LLM 调用成本（无预热）:
  10,000 * (1 - 0.20) * $0.01 = $80.00

每小时 LLM 调用成本（有预热）:
  10,000 * (1 - 0.65) * $0.01 = $35.00

每小时节省: $45.00
每日节省: $1,080.00

预热开销:
  预热计算耗时: 15 分钟
  预热消耗 Token: 500,000 (约 $5.00)
  
预热 ROI: ($45.00 * 24 - $5.00) / $5.00 = 215x
```

### 预热常见陷阱与解决

| 陷阱 | 表现 | 解决方案 |
|------|------|----------|
| 预热过期 | 预热后缓存迅速过期，命中率回退 | 延长 TTL + 异步续期 |
| 预热失真 | 预热的条目实际很少被访问 | 基于更长时间窗口分析历史 |
| 预热 OOM | 预热数据量超过缓存容量，导致淘汰 | 优先级排序 + 限制预热总量 |
| 预热惊群 | 预热请求压垮后端服务 | 并发控制 + 分阶段 + 速率限制 |
| 预热延迟 | 预热过慢，流量接入时仍未完成 | 提前预热 + Pipeline 并行 |

---

## 总结

缓存预热在 Agent 生产系统中不是可选项，而是必选项。核心要点：

1. **Agent 冷启动代价远超传统服务**——LLM 调用的延迟和成本使每次缓存未命中都代价高昂
2. **三种预热策略互补使用**——静态预热保底，动态预热适应变化，延迟预热处理长尾
3. **预热需要工程化设计**——并发控制、分阶段、速率限制是避免惊群的关键
4. **预热效果可量化评估**——命中率提升和成本节省应作为持续监控指标

在下一节（13.3.6）中，我们将探讨如何将缓存从单机扩展到分布式环境，以支持多实例 Agent 系统的统一缓存需求。
