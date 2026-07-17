# 13.3.3 cache-layer-design -- 缓存分层：L1(LRU) + L2(Redis) + L3(DB)

**核心论点：单一缓存层无法同时满足 Agent 系统对低延迟、高命中率和持久化的需求 -- L1(LRU 内存) 提供纳秒级访问但容量有限，L2(Redis) 提供毫秒级分布式共享，L3(DB) 提供持久化但访问较慢。多层缓存的本质是 "用空间换时间，用层次换性价比"。**

在 AI Agent 的生产部署中，缓存不是优化锦上添花的可选项，而是决定系统能否承载真实用户负载的必需品。Agent 的推理链路涉及多次 LLM 调用、工具调用、知识检索和中间结果拼接 -- 每一跳都可能引入数百毫秒到数秒的延迟。如果没有合理的缓存分层架构，每一次相同或相似的请求都会穿透到 LLM，导致不可接受的响应延迟和失控的 Token 消耗成本。

---

## 1. 总体架构

三层缓存形成一个逐级回退的查找链路。每一层在速度、容量和持久性上有不同的权衡：

```
  Request
     │
     ▼
  ┌─────────────────────┐
  │  L1: LRU Cache      │  ◄── 纳秒级, 进程本地, 容量 MB~GB
  │  (Python dict + lru)│
  └─────────┬───────────┘
            │
      ┌─────┴─────┐
      │   Hit?    │──── Yes ──► Return response
      └─────┬─────┘
            │ No
            ▼
  ┌─────────────────────┐
  │  L2: Redis Cluster  │  ◄── 毫秒级, 多进程共享, 容量 GB~TB
  │  (Distributed KV)   │
  └─────────┬───────────┘
            │
      ┌─────┴─────┐
      │   Hit?    │──── Yes ──► Backfill L1 ──► Return
      └─────┬─────┘
            │ No
            ▼
  ┌─────────────────────┐
  │  L3: Database       │  ◄── 10~100ms, 持久化, 容量 TB+
  │  (PostgreSQL + BLOB)│
  └─────────┬───────────┘
            │
      ┌─────┴─────┐
      │   Hit?    │──── Yes ──► Backfill L2+L1 ──► Return
      └─────┬─────┘
            │ No
            ▼
  ┌─────────────────────┐
  │  LLM Inference       │  ◄── 数百 ms~数秒, 成本最高
  │  (Compute + Token)  │
  └─────────┬───────────┘
            │
            ▼
  Store result ──► L3 ──► L2 ──► L1 ──► Return
  (write-back chain)
```

关键设计原则：
- **读穿透（Read-Through）**：请求逐层穿透，命中后逐层回填（Backfill）。
- **写回传（Write-Back）**：LLM 返回结果后，依次写入 L3 → L2 → L1，保证下层包含上层数据的超集。
- **每一层独立淘汰**：各层有独立的容量上限和淘汰策略，互不阻塞。

---

## 2. 每一层的详细分析

### 2.1 L1: Memory LRU Cache

**特点**：
- 访问延迟：纳秒级（通常 < 100ns）
- 容量：受限于单进程堆内存，典型配置 256MB ~ 4GB
- 生命周期：与进程生命周期绑定，进程重启后丢失
- 一致性：强一致（进程内无竞争）

**使用场景**：
- 高频重复请求：同一 prompt 在短时间内被多次发起（如 Agent 在 ReAct 循环中的重复思考步骤）
- 中间计算结果：Token 分词结果、Embedding 向量、函数调用 schema 解析
- 会话级上下文：当前对话轮次内的临时数据

**实现方式对比**：

| 方案 | 优势 | 劣势 | 适用场景 |
|------|------|------|----------|
| `functools.lru_cache` | 零依赖, 内置 | 无 TTL, 无容量控制 | 纯函数计算结果 |
| `cachetools` | TTL + LRU + LFU | 线程安全需额外处理 | 一般生产场景 |
| 自定义（dict + heapq/OrderedDict） | 完全控制 | 需要自行维护 | 特殊淘汰策略 |

**示例：基于 `cachetools` 的 L1 层**

```python
from cachetools import LRUCache, TTLCache
import time
import threading

class L1MemoryCache:
    """L1 内存缓存层：TTL + LRU 混合策略"""

    def __init__(self, maxsize: int = 8192, ttl: int = 300):
        # 主缓存：TTLCache 自带过期，但淘汰基于插入时间而非访问时间
        self._cache = TTLCache(maxsize=maxsize, ttl=ttl)
        # 访问计数器：用于实现 LRU-like 行为
        self._access_count = {}
        self._lock = threading.Lock()

    def get(self, key: str) -> dict | None:
        with self._lock:
            if key in self._cache:
                self._access_count[key] = self._access_count.get(key, 0) + 1
                return self._cache[key]
            return None

    def set(self, key: str, value: dict):
        with self._lock:
            self._cache[key] = value
            self._access_count[key] = self._access_count.get(key, 0) + 1

    def invalidate(self, key: str):
        with self._lock:
            self._cache.pop(key, None)
            self._access_count.pop(key, None)

    @property
    def hit_rate(self) -> float:
        total = sum(self._access_count.values())
        if total == 0:
            return 0.0
        # 近似计算：当前仍在缓存中的 key 的访问占比
        in_cache = sum(v for k, v in self._access_count.items() if k in self._cache)
        return in_cache / total
```

**L1 最佳实践**：
- 设置合理的 `maxsize`：通常为 QPS × 峰值容忍时间（如 1000 QPS × 30s = 30000 条目）
- 区分高频和低频 key：高频 key 手动固定在 L1 中不被逐出（pin 机制）
- 监控 eviction 数量：如果 L1 逐出率超过 30%，说明容量不足，需要扩容或调整策略

---

### 2.2 L2: Redis Cache

**特点**：
- 访问延迟：毫秒级（通常 1~10ms，取决于网络 RTT）
- 容量：受限于 Redis 实例内存，典型配置 8GB ~ 256GB
- 共享性：多进程 / 多实例共享同一缓存池
- 数据结构：除 KV 外还支持 List、Set、Sorted Set、Hash、Stream

**使用场景**：
- 跨实例缓存共享：多副本 Agent 服务共享同一缓存
- 分布式锁：防止缓存惊群（Cache Stampede）
- 计数和限流：API 调用频率控制、Token 用量统计
- 缓存预热：启动时从 L2 批量加载热点数据

**实现要点**：

```python
import json
import hashlib
import redis.asyncio as aioredis

class L2RedisCache:
    """L2 Redis 缓存层：JSON 序列化 + TTL + 淘汰策略"""

    def __init__(self, redis_client: aioredis.Redis, default_ttl: int = 3600):
        self._redis = redis_client
        self._default_ttl = default_ttl

    def _make_key(self, namespace: str, key: str) -> str:
        # 加入命名空间和 hash 前缀，避免 key 冲突
        key_hash = hashlib.md5(key.encode()).hexdigest()
        return f"l2:{namespace}:{key_hash}"

    async def get(self, namespace: str, key: str) -> dict | None:
        redis_key = self._make_key(namespace, key)
        data = await self._redis.get(redis_key)
        if data is None:
            return None
        # 异步更新 access time，用于 LRU 近似统计
        await self._redis.expire(redis_key, self._default_ttl, nx=True)
        return json.loads(data)

    async def set(
        self,
        namespace: str,
        key: str,
        value: dict,
        ttl: int | None = None,
    ):
        redis_key = self._make_key(namespace, key)
        data = json.dumps(value, default=str)
        await self._redis.set(
            redis_key,
            data,
            ex=ttl or self._default_ttl,
        )

    async def set_nx(self, namespace: str, key: str, ttl: int = 30) -> bool:
        """SET if Not eXists: 用于分布式锁"""
        redis_key = self._make_key(namespace, key)
        return await self._redis.set(redis_key, "locked", nx=True, ex=ttl)
```

**Redis 淘汰策略选择**：

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `allkeys-lru` | 所有 key 按 LRU 淘汰 | 通用缓存，推荐 |
| `volatile-ttl` | 仅淘汰有过期时间的 key，按 TTL 最短优先 | 混合存储（持久 + 缓存）|
| `allkeys-lfu` | 按访问频率淘汰 | 访问模式极度偏斜的场景 |
| `noeviction` | 不淘汰，写入失败 | 需要保证写入成功的场景 |

**L2 最佳实践**：
- 使用 Redis Cluster 或 Elasticache 做水平扩展，单实例不超过 64GB 内存
- 大 value（> 10KB）考虑压缩：`zlib` 或 `lz4`，减少内存和网络开销
- 使用 Pipeline 批量操作减少 RTT
- 为不同 namespace 设置不同的 TTL 默认值

---

### 2.3 L3: Database Cache

**特点**：
- 访问延迟：10~100ms（取决于查询复杂度和索引）
- 容量：几乎无限（TB 级别，受限于磁盘）
- 持久性：持久化存储，支持 ACID 事务
- 可查询性：支持复杂条件过滤和时间范围查询

**使用场景**：
- 长期缓存：跨天/跨周的缓存数据
- 缓存重建：从 L3 批量恢复 L1 + L2
- 离线分析：缓存命中趋势、用户行为挖掘
- 大 Value 存储：超过 Redis 容量限制的大响应（如完整 RAG 检索结果）

**实现示例**：

```python
import sqlalchemy as sa
from sqlalchemy.ext.asyncio import AsyncSession
from datetime import datetime
import uuid

class L3DatabaseCache:
    """L3 数据库缓存层：持久化 + 可查询"""

    def __init__(self, session_factory):
        self._session_factory = session_factory

    async def get(self, key_hash: str, namespace: str) -> dict | None:
        async with self._session_factory() as session:
            result = await session.execute(
                sa.text(
                    "SELECT value, expires_at FROM cache_entries "
                    "WHERE key_hash = :key_hash AND namespace = :namespace "
                    "AND (expires_at IS NULL OR expires_at > :now)"
                ),
                {"key_hash": key_hash, "namespace": namespace, "now": datetime.utcnow()},
            )
            row = result.one_or_none()
            if row is None:
                return None
            return {
                "value": row[0],
                "expires_at": row[1],
            }

    async def set(
        self,
        key_hash: str,
        namespace: str,
        value: dict,
        ttl: int | None = None,
    ):
        async with self._session_factory() as session:
            expires_at = datetime.utcnow().timestamp() + ttl if ttl else None
            await session.execute(
                sa.text(
                    "INSERT INTO cache_entries (id, key_hash, namespace, value, created_at, expires_at) "
                    "VALUES (:id, :key_hash, :namespace, :value, :now, :expires_at) "
                    "ON CONFLICT (key_hash, namespace) DO UPDATE SET "
                    "value = EXCLUDED.value, expires_at = EXCLUDED.expires_at, updated_at = :now"
                ),
                {
                    "id": str(uuid.uuid4()),
                    "key_hash": key_hash,
                    "namespace": namespace,
                    "value": json.dumps(value),
                    "now": datetime.utcnow(),
                    "expires_at": datetime.utcfromtimestamp(expires_at) if expires_at else None,
                },
            )
            await session.commit()
```

**表结构设计建议**：

```sql
CREATE TABLE cache_entries (
    id UUID PRIMARY KEY,
    key_hash VARCHAR(64) NOT NULL,       -- MD5/SHA256 of cache key
    namespace VARCHAR(128) NOT NULL,     -- 命名空间隔离
    value JSONB NOT NULL,                -- 缓存内容
    size_bytes INTEGER NOT NULL DEFAULT 0, -- value 大小，用于统计
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,              -- NULL 表示永不过期
    access_count BIGINT NOT NULL DEFAULT 0,
    last_accessed_at TIMESTAMPTZ,
    UNIQUE (key_hash, namespace)
);

CREATE INDEX idx_cache_expires ON cache_entries (expires_at)
    WHERE expires_at IS NOT NULL;
CREATE INDEX idx_cache_namespace ON cache_entries (namespace);
CREATE INDEX idx_cache_last_accessed ON cache_entries (last_accessed_at NULLS LAST);
```

---

## 3. 层间协调机制

### 3.1 穿透回填（Pass-through Backfill）

标准的多层缓存读取流程。在查找过程中同步回填，保证访问过的数据在上层可用：

```python
class MultiLayerCache:
    """三层缓存协调器：提供统一的 get/set 接口"""

    def __init__(self, l1: L1MemoryCache, l2: L2RedisCache, l3: L3DatabaseCache):
        self._l1 = l1
        self._l2 = l2
        self._l3 = l3

    async def get(self, namespace: str, key: str) -> dict | None:
        # Step 1: Try L1 (fastest)
        l1_key = f"{namespace}:{key}"
        result = self._l1.get(l1_key)
        if result is not None:
            return self._wrap_meta(result, layer="l1")

        # Step 2: Try L2
        result = await self._l2.get(namespace, key)
        if result is not None:
            # Backfill L1 asynchronously (don't block the response)
            self._l1.set(l1_key, result)
            return self._wrap_meta(result, layer="l2")

        # Step 3: Try L3
        key_hash = hashlib.sha256(key.encode()).hexdigest()
        result = await self._l3.get(key_hash, namespace)
        if result is not None:
            # Backfill L2 and L1
            await self._l2.set(namespace, key, result["value"])
            self._l1.set(l1_key, result["value"])
            return self._wrap_meta(result["value"], layer="l3")

        return None  # Complete miss -- caller should invoke LLM

    async def set(self, namespace: str, key: str, value: dict, ttl: int | None = None):
        """逐层写入：从最慢到最快。下层写入成功后写入上层。

        这个顺序保证如果写入中途失败，下层已经有数据，不会完全丢失。
        """
        key_hash = hashlib.sha256(key.encode()).hexdigest()
        l1_key = f"{namespace}:{key}"

        # Write to L3 first (persistent store)
        await self._l3.set(key_hash, namespace, value, ttl=ttl)
        # Write to L2
        await self._l2.set(namespace, key, value, ttl=ttl)
        # Write to L1
        self._l1.set(l1_key, value)

    def _wrap_meta(self, value: dict, layer: str) -> dict:
        """包装元数据，方便调试和可观测性"""
        value["_cache_meta"] = {
            "hit_layer": layer,
            "timestamp": time.time(),
        }
        return value
```

### 3.2 回写策略（Write-back / Write-behind）

对于延迟敏感的场景，可以采用异步回写：

```python
class WriteBackCache(MultiLayerCache):
    """回写式缓存：L1 同步写入，L2/L3 异步刷入"""

    def __init__(self, l1, l2, l3, flush_interval: float = 1.0):
        super().__init__(l1, l2, l3)
        self._write_buffer: list[tuple] = []
        self._flush_interval = flush_interval
        self._flush_lock = asyncio.Lock()
        self._flush_task: asyncio.Task | None = None

    async def set(self, namespace: str, key: str, value: dict, ttl: int | None = None):
        l1_key = f"{namespace}:{key}"
        # L1 立即写入
        self._l1.set(l1_key, value)

        # L2/L3 缓冲后异步写入
        async with self._flush_lock:
            self._write_buffer.append((namespace, key, value, ttl))
            if self._flush_task is None:
                self._flush_task = asyncio.create_task(self._periodic_flush())

    async def _periodic_flush(self):
        await asyncio.sleep(self._flush_interval)
        async with self._flush_lock:
            batch = self._write_buffer.copy()
            self._write_buffer.clear()
            self._flush_task = None

        # 批量写入 L2 和 L3
        for namespace, key, value, ttl in batch:
            try:
                key_hash = hashlib.sha256(key.encode()).hexdigest()
                await self._l3.set(key_hash, namespace, value, ttl=ttl)
                await self._l2.set(namespace, key, value, ttl=ttl)
            except Exception as e:
                logger.error(f"Write-back failed for {namespace}:{key}: {e}")
                # 失败写回队列尾部，或发送到死信队列
```

**风险控制**：
- 回写缓冲区不能无限增长：设置最大长度，超过时同步阻塞写入
- 进程崩溃时可能丢失缓冲数据：L1 容量内最后一次 flush 之间的数据
- 建议结合 L1 的持久化快照（如每 5 秒 dump 到本地文件）降低风险

### 3.3 缓存预热的分层加载

新服务上线或缓存清空后，可以按分层策略逐步预热：

```python
class CacheWarmupStrategy:
    """分层缓存预热策略"""

    PRIORITY_TIERS = {
        "critical": 0,   # 必须预热
        "high": 1,       # 尽快预热
        "medium": 2,     # 后台预热
        "low": 3,        # 按需加载（不主动预热）
    }

    async def warmup(self, cache: MultiLayerCache):
        # Phase 1: 从 L3 加载 critical 数据到 L2
        critical_keys = await self._load_hot_keys_from_l3(tier="critical")
        await self._batch_warm_l2(cache, critical_keys)

        # Phase 2: 从 L2 批量加载到 L1（L1 容量有限，只加载最热的子集）
        l1_hot_set = self._select_l1_candidates(critical_keys, max_entries=8192)
        for key in l1_hot_set:
            result = await cache.get(key.namespace, key.key)
            # get 操作会自动回填 L1
            _ = result

        # Phase 3: 后台预热 high/medium 数据
        asyncio.create_task(self._background_warmup(cache, tiers=["high", "medium"]))
```

---

## 4. Agent 特有优化

### 4.1 Token 感知的分层分配

不同的 LLM 调用有不同的延迟敏感度和 Token 成本。根据响应特征决定缓存层级：

```python
class TokenAwarePlacementPolicy:
    """Token 感知的缓存放置策略"""

    # 短响应：Token 数 < 200, 延迟敏感 -> L1 优先
    # 中响应：Token 数 200~2000 -> L2 优先
    # 长响应：Token 数 > 2000 -> L3 优先（L1 容量效率低）

    SHORT_THRESHOLD = 200
    LONG_THRESHOLD = 2000

    def decide_placement(self, response: dict) -> dict:
        token_count = response.get("usage", {}).get("completion_tokens", 0)

        if token_count < self.SHORT_THRESHOLD:
            # 短响应：存入所有层
            return {"l1": True, "l2": True, "l3": True, "ttl": 3600}
        elif token_count < self.LONG_THRESHOLD:
            # 中响应：跳过 L1，存入 L2 + L3
            return {"l1": False, "l2": True, "l3": True, "ttl": 7200}
        else:
            # 长响应：仅存入 L3，L2 压缩后存入
            return {
                "l1": False,
                "l2": True,
                "l2_compress": True,
                "l3": True,
                "ttl": 86400,
            }
```

### 4.2 Session-aware 分层

Agent 的会话（Session）具有天然的局部性。当前活跃 Session 的数据应该优先停留在 L1：

```python
class SessionAwareCache:
    """Session 感知的缓存分层"""

    def __init__(self, multi_layer: MultiLayerCache, session_ttl: int = 600):
        self._cache = multi_layer
        # session_id -> set of cache keys
        self._session_keys: dict[str, set] = {}

    async def get_for_session(self, session_id: str, namespace: str, key: str):
        cache_key = f"session:{session_id}:{namespace}:{key}"
        result = await self._cache.get(namespace, cache_key)

        if result is not None:
            # 活跃 session 的命中记录
            self._touch_session(session_id, cache_key)

        return result

    async def set_for_session(self, session_id: str, namespace: str, key: str, value: dict):
        cache_key = f"session:{session_id}:{namespace}:{key}"
        # 当前 session 数据短 TTL 在 L1
        await self._cache.set(namespace, cache_key, value, ttl=300)
        self._touch_session(session_id, cache_key)

    def _touch_session(self, session_id: str, cache_key: str):
        if session_id not in self._session_keys:
            self._session_keys[session_id] = set()
        self._session_keys[session_id].add(cache_key)

    def on_session_end(self, session_id: str):
        """Session 结束时：将 session 数据从 L1 降级到 L2/L3"""
        keys = self._session_keys.pop(session_id, set())
        for key in keys:
            self._cache._l1.invalidate(key)  # 从 L1 移除
        # L2/L3 的数据保留，供后续跨 session 复用
```

### 4.3 压缩存储策略

不同层级有不同的存储成本，压缩策略应该不同：

| 层级 | 存储格式 | 压缩策略 | 解压开销 |
|------|----------|----------|----------|
| L1 | 原始 Python 对象 | 无压缩（解压开销 > 内存节省）| 0 |
| L2 | JSON string | `zlib.level=3`（速度优先）| ~0.1ms |
| L3 | JSONB | `zlib.level=6`（压缩率优先）| ~0.5ms |

```python
import zlib
import json

class CompressedCacheLayer:
    """支持压缩的缓存层装饰器"""

    def __init__(self, inner_layer, threshold: int = 1024, level: int = 3):
        self._inner = inner_layer
        self._threshold = threshold  # 超过此大小才压缩
        self._level = level

    async def get(self, namespace: str, key: str) -> dict | None:
        raw = await self._inner.get(namespace, key)
        if raw is None:
            return None
        if raw.get("_compressed"):
            raw["value"] = json.loads(
                zlib.decompress(raw["value"].encode("latin1"))
            )
            raw["_compressed"] = False
        return raw

    async def set(self, namespace: str, key: str, value: dict, ttl: int | None = None):
        serialized = json.dumps(value)
        if len(serialized) > self._threshold:
            compressed = zlib.compress(serialized.encode("latin1"), level=self._level)
            value = {
                "_compressed": True,
                "value": compressed.decode("latin1"),
                "original_size": len(serialized),
                "compressed_size": len(compressed),
            }
        await self._inner.set(namespace, key, value, ttl=ttl)
```

---

## 5. 缓存统计与可观测性

没有可观测性的缓存系统是盲人摸象。每个缓存层都需要独立监控。

### 5.1 每层命中率监控

```python
import prometheus_client as prom

class CacheMetrics:
    """分层缓存指标收集"""

    def __init__(self, prefix: str = "agent_cache"):
        self.hits = prom.CounterVec(
            f"{prefix}_hits_total",
            "Total cache hits by layer",
            ["layer", "namespace"],
        )
        self.misses = prom.CounterVec(
            f"{prefix}_misses_total",
            "Total cache misses by layer",
            ["layer", "namespace"],
        )
        self.size = prom.GaugeVec(
            f"{prefix}_size_bytes",
            "Current cache size in bytes",
            ["layer"],
        )
        self.evictions = prom.CounterVec(
            f"{prefix}_evictions_total",
            "Total evictions by layer",
            ["layer", "reason"],
        )
        self.latency = prom.HistogramVec(
            f"{prefix}_latency_seconds",
            "Cache operation latency by layer and operation",
            ["layer", "operation"],
            buckets=(.0001, .0005, .001, .005, .01, .05, .1, .5, 1.0),
        )
```

### 5.2 关键指标解读

| 指标 | 健康值 | 告警阈值 | 含义 |
|------|--------|----------|------|
| L1 命中率 | > 80% | < 60% | 热点数据是否有效利用 |
| L2 命中率 | > 60% | < 40% | 分布式缓存是否有效 |
| L3 命中率 | > 30% | < 15% | 持久缓存是否提供价值 |
| L1 逐出率 | < 10% | > 30% | L1 容量不足 |
| 平均穿透深度 | < 1.5 | > 2.5 | 缓存有效性整体指标 |
| 缓存回填延迟 | < 5ms | > 20ms | 回填逻辑是否过重 |

### 5.3 成本效益分析

缓存不是免费的。每一层都有明确的成本结构：

```
成本效益计算模型：

  L1 成本 = 内存占用 × 单位内存价格
  L2 成本 = Redis 实例费用 + 网络带宽
  L3 成本 = 数据库存储 + IOPS

  节省 = (避免的 LLM 调用次数 × 每次 LLM 调用成本) - 缓存层总成本

  每层 ROI = 该层贡献的节省 / 该层成本

  推荐：L1 命中避开 LLM 调用，ROI 最高（40:1 ~ 100:1）
       L2 命中，ROI 中等（10:1 ~ 40:1）
       L3 命中，ROI 较低（3:1 ~ 10:1）
```

通过监控 `命中率 × LLM 单次成本 / 缓存运营成本` 来动态判断是否值得增加缓存容量。

---

## 6. 完整示例：生产级 MultiLayerCache

综合所有上述设计，一个生产级的三层缓存协调器：

```python
import asyncio
import hashlib
import json
import logging
import time
from collections.abc import Callable
from dataclasses import dataclass

logger = logging.getLogger(__name__)


@dataclass
class CacheConfig:
    """三层缓存配置"""
    l1_maxsize: int = 16384
    l1_ttl: int = 300          # 5 minutes
    l2_default_ttl: int = 3600  # 1 hour
    l3_default_ttl: int = 86400 # 24 hours
    enable_l1: bool = True
    enable_l2: bool = True
    enable_l3: bool = True


class ProductionMultiLayerCache:
    """生产级三层缓存协调器"""

    def __init__(
        self,
        config: CacheConfig,
        l1_factory: Callable,
        l2_client,
        l3_session_factory,
        metrics: CacheMetrics | None = None,
    ):
        self.config = config
        self._l1 = l1_factory(maxsize=config.l1_maxsize, ttl=config.l1_ttl) if config.enable_l1 else None
        self._l2 = L2RedisCache(l2_client, config.l2_default_ttl) if config.enable_l2 else None
        self._l3 = L3DatabaseCache(l3_session_factory) if config.enable_l3 else None
        self._metrics = metrics or CacheMetrics()
        self._locks: dict[str, asyncio.Lock] = {}  # per-key lock to prevent stampede

    async def get(self, namespace: str, key: str) -> dict | None:
        start = time.perf_counter()
        key_hash = hashlib.sha256(key.encode()).hexdigest()
        l1_key = f"{namespace}:{key}"

        # ---- L1 Lookup ----
        if self._l1:
            result = self._l1.get(l1_key)
            if result is not None:
                self._metrics.hits.labels(layer="l1", namespace=namespace).inc()
                self._metrics.latency.labels(layer="l1", operation="get").observe(
                    time.perf_counter() - start
                )
                return result

        # ---- L2 Lookup ----
        if self._l2:
            result = await self._l2.get(namespace, key)
            if result is not None:
                self._metrics.hits.labels(layer="l2", namespace=namespace).inc()
                # Backfill L1
                if self._l1:
                    self._l1.set(l1_key, result)
                self._metrics.latency.labels(layer="l2", operation="get").observe(
                    time.perf_counter() - start
                )
                return result

        # ---- L3 Lookup ----
        if self._l3:
            result = await self._l3.get(key_hash, namespace)
            if result is not None:
                self._metrics.hits.labels(layer="l3", namespace=namespace).inc()
                # Backfill L2 and L1
                value = result["value"]
                if self._l2:
                    await self._l2.set(namespace, key, value)
                if self._l1:
                    self._l1.set(l1_key, value)
                self._metrics.latency.labels(layer="l3", operation="get").observe(
                    time.perf_counter() - start
                )
                return result

        # ---- Complete Miss ----
        self._metrics.misses.labels(layer="all", namespace=namespace).inc()
        self._metrics.latency.labels(layer="miss", operation="get").observe(
            time.perf_counter() - start
        )
        return None

    async def set(self, namespace: str, key: str, value: dict, ttl: int | None = None):
        key_hash = hashlib.sha256(key.encode()).hexdigest()
        l1_key = f"{namespace}:{key}"

        # 从最下层开始写入，逐步向上
        if self._l3:
            await self._l3.set(key_hash, namespace, value, ttl=ttl or self.config.l3_default_ttl)
        if self._l2:
            await self._l2.set(namespace, key, value, ttl=ttl or self.config.l2_default_ttl)
        if self._l1:
            self._l1.set(l1_key, value)

    async def get_or_compute(
        self,
        namespace: str,
        key: str,
        compute_fn: Callable[[], dict],
    ) -> dict:
        """缓存穿透保护：防止同一 key 的并发请求都穿透到 LLM"""

        # 尝试获取
        cached = await self.get(namespace, key)
        if cached is not None:
            return cached

        # 获取 per-key 锁，防止惊群
        lock_key = f"{namespace}:{key}"
        if lock_key not in self._locks:
            self._locks[lock_key] = asyncio.Lock()

        async with self._locks[lock_key]:
            # Double-check: 持有锁后再次尝试获取
            cached = await self.get(namespace, key)
            if cached is not None:
                return cached

            # 实际计算（LLM 调用）
            logger.info(f"Cache miss for {namespace}:{key}, computing...")
            value = await compute_fn()

            # 写入缓存
            await self.set(namespace, key, value)
            return value

    async def invalidate(self, namespace: str, key: str):
        """跨层失效"""
        key_hash = hashlib.sha256(key.encode()).hexdigest()
        l1_key = f"{namespace}:{key}"

        if self._l1:
            self._l1.invalidate(l1_key)
        if self._l2:
            await self._l2.delete(namespace, key)
        if self._l3:
            await self._l3.delete(key_hash, namespace)
```

这个生产级实现的核心设计决策：

1. **per-key 锁**：防止缓存惊群（Cache Stampede），避免 N 个并发请求同时穿透到 LLM
2. **Double-check 模式**：获取锁后二次检查，最大程度减少 LLM 调用
3. **从下到上的写入顺序**：保证数据不会因中间失败而完全丢失
4. **逐层解耦**：每层可以独立禁用、独立扩容、独立监控
5. **可观测性注入**：操作级别延迟采集，便于调试和容量规划

---

## 总结

三层缓存架构不是一成不变的模板，而是一种设计哲学：

- **L1** 解决的是 "热点数据的极速访问" 问题 -- 用内存换取纳秒级响应
- **L2** 解决的是 "分布式共享和一致性" 问题 -- 用网络换取毫秒级共享
- **L3** 解决的是 "持久化和历史复用" 问题 -- 用磁盘换取可靠性和大容量

在 AI Agent 系统中，这三层缺一不可。没有 L1，高频的 ReAct 循环将反复请求 LLM；没有 L2，多实例部署无法共享缓存收益；没有 L3，服务重启后缓存全部丢失，需要数小时重建。

投入时间设计好这三层的协调机制，是 Agent 系统从原型走向生产的关键一步。
