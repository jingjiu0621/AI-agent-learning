# 13.3.6 distributed-cache -- 分布式缓存（Redis Cluster / Memcached）

## 核心论点

**当 Agent 系统从单实例扩展到多实例时，本地内存缓存变成了一个个孤岛——实例 A 缓存了的 LLM 响应，实例 B 无法复用。分布式缓存让 N 个 Agent 实例共享同一个缓存池，从 N 个独立缓存变为统一的缓存池。核心矛盾在于：分布式缓存的网络开销（毫秒级）与本地缓存的延迟（纳秒级）之间的取舍，以及缓存一致性维护的复杂性。**

要理解这个核心矛盾，可以看一个具体的数字对比：

```
访问延迟对比:
  L1 本地内存缓存:    ~0.1ms (纳秒级, 无网络)
  L2 分布式 Redis:    ~1-5ms (毫秒级, 有网络)
  L3 LLM 直接调用:    ~500-5000ms (秒级)

命中率与延迟的权衡:
  纯本地缓存:    延迟最低(0.1ms), 但 N 个实例命中率独立
  纯分布式缓存: 延迟中等(3ms),  但命中率=N倍单实例
  本地+分布式:   延迟大部分 0.1ms, 穿透时 3ms
```

没有分布式缓存的 Agent 集群：同样的 LLM 请求在不同实例上被重复计算。以一个 10 实例的集群为例，假设单个实例的本地缓存命中率为 60%，理论上共享缓存可将有效命中率提升到 95% 以上。

---

## 何时需要分布式缓存

### 硬性条件

| 条件 | 说明 | 阈值 |
|------|------|------|
| Agent 实例数 | 多个实例同时服务 | >= 2 |
| 重复请求率 | 不同实例收到相同/相似请求的比例 | > 20% |
| LLM 成本敏感度 | Token 消耗直接影响运营成本 | 任何水平 |
| 会话状态共享 | 用户请求需要被任意实例处理 | Session 跨实例 |
| 部署策略 | 蓝绿部署/滚动更新期间缓存不丢失 | 持久化要求 |

### 缓存碎片化问题

当没有分布式缓存时，N 个实例的缓存是相互隔离的。假设每个实例有 80% 的本地命中率，理论上共享缓存后命中率计算如下：

```
无分布式缓存（本地缓存独立）:
  实例 A: 100 请求, 80 本地命中, 20 穿透 -> 命中率 80%
  实例 B: 100 请求, 80 本地命中, 20 穿透 -> 命中率 80%
  实例 C: 100 请求, 80 本地命中, 20 穿透 -> 命中率 80%
  Total:  300 请求, 240 命中 -> 全局命中率 80%
  Total LLM 调用: 60 次

有分布式缓存（共享缓存池）:
  实例 A: 100 请求, 80 本地+10 分布式命中, 10 穿透
  实例 B: 100 请求, 80 本地+15 分布式命中, 5 穿透  
  实例 C: 100 请求, 80 本地+12 分布式命中, 8 穿透
  Total:  300 请求, 280+命中 -> 全局命中率 93%+
  Total LLM 调用: 20+ 次

节省: 60 -> 20 次 LLM 调用, 成本降低 66%
```

---

## 技术选型对比

| 特性 | Redis Cluster | Memcached | Valkey |
|------|-------------|-----------|--------|
| 数据结构 | String / List / Hash / Set / Sorted Set / Stream / Bitmap | 仅 String | 同 Redis |
| 持久化 | RDB 快照 + AOF 日志 | 无 | 同 Redis |
| 集群模式 | 官方 Cluster（16384 slots）+ Sentinel HA | 客户端一致性哈希 | 同 Redis（兼容） |
| 复制 | 主从异步复制 | 无内置 | 同 Redis |
| Lua 脚本 | 支持 | 不支持 | 支持 |
| 发布订阅 | 支持 | 不支持 | 支持 |
| 适用 Agent 场景 | 缓存 + 会话 + 队列 + 限流 | 纯 KV 缓存 | 同 Redis（开源替代） |

### 为什么 Redis / Valkey 是 Agent 系统的标准选择

Agent 系统的缓存需求远不止"存字符串取字符串"。Redis（及其后起之秀 Valkey）的丰富数据结构使其成为 Agent 缓存的天然选择：

- **String**: 存 LLM 响应的 JSON 序列化结果
- **Hash**: 存 Agent Session 的复杂状态（每个 field 一个属性）
- **Sorted Set**: 维护缓存访问频率排名，用于智能淘汰
- **Set**: 实现缓存标签系统（Tag-based Invalidation）
- **Stream**: Agent 事件日志的轻量消息队列
- **List**: Agent 任务队列的 FIFO 实现

Valkey 是 Redis 在 2024 年协议变更后的 Linux Foundation 托管分支，完全兼容 Redis 协议。Valkey 9.0 已达到每秒 10 亿次请求的性能水平。对于新项目，Valkey 是一个无许可证风险的 Redis 替代。

```python
# 统一抽象：支持 Redis/Valkey/Memcached 的客户端适配器
from abc import ABC, abstractmethod
from typing import Any, Dict, List, Optional, Tuple

class DistributedCacheBackend(ABC):
    """分布式缓存后端抽象接口"""
    
    @abstractmethod
    async def get(self, key: str) -> Optional[str]: ...
    
    @abstractmethod
    async def setex(self, key: str, ttl: int, value: str): ...
    
    @abstractmethod
    async def exists(self, key: str) -> bool: ...
    
    @abstractmethod
    async def delete(self, key: str): ...
    
    @abstractmethod
    async def pipeline(self): ...
    
    # Agent 特定的高级操作
    @abstractmethod
    async def hgetall(self, key: str) -> Dict[str, str]: ...
    
    @abstractmethod
    async def zadd(self, key: str, score: float, member: str): ...
    
    @abstractmethod
    async def zrange(self, key: str, start: int, end: int, desc: bool = False) -> List[str]: ...


class RedisClusterBackend(DistributedCacheBackend):
    """Redis Cluster 实现"""
    
    def __init__(self, startup_nodes: List[Tuple[str, int]], password: str = None):
        import redis.asyncio as aioredis
        from redis.asyncio.cluster import RedisCluster
        
        self.client = RedisCluster(
            startup_nodes=[
                {"host": h, "port": p} for h, p in startup_nodes
            ],
            password=password,
            decode_responses=True,
        )
    
    async def get(self, key: str) -> Optional[str]:
        return await self.client.get(key)
    
    async def setex(self, key: str, ttl: int, value: str):
        await self.client.setex(key, ttl, value)
    
    async def exists(self, key: str) -> bool:
        return bool(await self.client.exists(key))
    
    async def delete(self, key: str):
        await self.client.delete(key)
    
    async def pipeline(self):
        # Redis Cluster 的 pipeline 需要针对同一 node
        # 简化实现：如果确认 keys 在同一 slot，返回 pipeline
        return self.client.pipeline()
    
    async def hgetall(self, key: str) -> Dict[str, str]:
        return await self.client.hgetall(key)
    
    async def zadd(self, key: str, score: float, member: str):
        await self.client.zadd(key, {member: score})
    
    async def zrange(self, key: str, start: int, end: int, desc: bool = False) -> List[str]:
        return await self.client.zrange(key, start, end, desc=desc)
```

---

## Agent 缓存的数据结构设计

### String: LLM 响应缓存

```python
# LLM 响应缓存的 JSON 结构
llm_response_cache = {
    "cache:llm:prompt_md5:abc123": json.dumps({
        "response": "I am an AI assistant...",
        "model": "claude-haiku-4-5-20251001",
        "usage": {
            "input_tokens": 1520,
            "output_tokens": 423,
        },
        "cached_at": 1700000000.123,
        "ttl": 3600,
    })
}

# Key 设计模式
# cache:llm:{prompt_hash}:      LLM 响应缓存
# cache:agent:{agent_id}:sys:   System Prompt 缓存
# cache:tool:{tool_name}:schema: 工具定义缓存
# cache:embed:{text_hash}:      Embedding 缓存
```

### Hash: Session 状态

Agent Session 通常包含多个属性，Hash 结构允许只更新其中部分字段而不影响其他。

```python
# Agent Session 状态存储
# HSET session:{session_id} field value

session_data = {
    "conversation_history": "[...truncated...]",
    "current_tools": '["search", "calculator"]',
    "user_context": '{"timezone": "Asia/Shanghai"}',
    "token_budget": "4000",
    "state_machine": "awaiting_tool_call",
}

# 优势：可以单独更新 token_budget 而无需重写整个 Session
# HINCRBY session:{session_id} token_budget -500
```

### Sorted Set: 缓存访问频率

用于实现 LFU（Least Frequently Used）淘汰策略。

```python
# 记录每个缓存 Key 的访问频率
# ZINCRBY cache:freq:global 1 "cache:llm:prompt_md5:abc123"

# 获取当前最热门的 Top-10 Key
# ZREVRANGE cache:freq:global 0 9 WITHSCORES

# 定时清理低频 Key
# ZREMRANGEBYRANK cache:freq:global 0 -10001  (保留 Top-10000)
```

### Set: 缓存标签系统

支持按标签批量失效——当某个 Agent 的工具定义更新时，需要失效所有关联的缓存。

```python
# 标签系统：一个 Key 属于多个标签
# SADD tag:agent:weather_bot "cache:llm:prompt:abc" "cache:tool:weather:schema"
# SADD tag:version:v2 "cache:llm:prompt:abc" "cache:llm:prompt:def"

# 按标签批量失效
# SMEMBERS tag:agent:weather_bot -> ["cache:llm:prompt:abc", "cache:tool:weather:schema"]
# DEL cache:llm:prompt:abc cache:tool:weather:schema
# DEL tag:agent:weather_bot
```

---

## 分片与一致性哈希

### Redis Cluster 的 Slot 机制

Redis Cluster 将 Key 空间分为 16384 个 Hash Slot。每个 Key 映射到哪个 Slot 由 CRC16(key) % 16384 决定。每个节点负责一部分 Slot 范围。

```
Redis Cluster Slot 分布 (3 节点):

             +-----------+
             | Node A    |
             | Slot 0-5500|
             +-----------+
                   |
    +--------------+--------------+
    |                             |
+-----------+             +-----------+
| Node B    |             | Node C    |
| Slot 5501-11000|          | Slot 11001-16383|
+-----------+             +-----------+

Key 路由流程:
  1. Client 计算 CRC16("cache:llm:prompt:abc") % 16384 = 12345
  2. Client 查找 Slot 12345 所在的节点 = Node C
  3. Client 直接连接 Node C 读写

MOVED 重定向:
  - 如果 Client 的 slot 映射已过时，节点返回 MOVED 错误
  - Client 更新本地 slot 映射并重定向请求
```

### 热点 Key 的处理

当所有请求命中同一个 Key 时（例如公共的 System Prompt），该 Key 所在的 Redis 节点会成为瓶颈。

```python
# 热点 Key 散列方案

import hashlib
from typing import List

class HotKeySharding:
    """将热点 Key 分散到多个物理 Key 上"""
    
    def __init__(self, shard_count: int = 16):
        self.shard_count = shard_count
    
    def shard_key(self, base_key: str) -> List[str]:
        """生成 base_key 的 N 个分片 Key"""
        return [f"{base_key}:shard:{i}" for i in range(self.shard_count)]
    
    def read_shard(self, base_key: str) -> str:
        """读取时随机选择一个分片（期望所有分片内容一致）"""
        import random
        idx = random.randint(0, self.shard_count - 1)
        return f"{base_key}:shard:{idx}"
    
    def write_all_shards(self, base_key: str, value: str, ttl: int):
        """写入时写入所有分片（确保数据一致性）"""
        # 注意：这会增加 N-1 倍的写入负载
        # 适用于"读远多过写"的热点 Key（如 System Prompt）
        pass

# 另一种方案：本地 + 分布式两级缓存
# 热点 System Prompt 在每台机器本地也存一份
```

---

## 高可用架构

### Redis Sentinel 模式

```
           +-----------+
           | Sentinel  |  (奇数个, >=3)
           | Cluster   |
           +------+----+
                  |
    +-------------+-------------+
    |             |             |
+-------+    +-------+    +-------+
| Master|    | Slave |    | Slave |
| (RW)  |--->| (RO)  |--->| (RO)  |
+-------+    +-------+    +-------+
     |
     | 故障时 Sentinel 自动提升 Slave 为 Master
     v
+-------+    +-------+
| New   |    | Slave |
|Master |--->| (RO)  |
+-------+    +-------+
```

### Redis Cluster 自动故障转移

```
正常状态:
  Node A (master, slots 0-5500)
  Node B (master, slots 5501-11000)  
  Node C (master, slots 11001-16383)
  Node A_replica (replica of A)
  Node B_replica (replica of B)
  Node C_replica (replica of C)

Node A 宕机:
  -> Cluster 检测 A 不可达
  -> 多数 master 达成共识
  -> A_replica 提升为 master
  -> slots 0-5500 重新可用

网络分区 (split-brain):
  -> Redis Cluster 要求多数节点才能提供服务
  -> 少数分区不可写, 保证 CP
```

### 缓存双写 + 故障降级

```python
import asyncio
from typing import Optional, Callable, Awaitable

class ResilientCacheClient:
    """带故障降级的分布式缓存客户端"""
    
    def __init__(
        self,
        redis_client,
        local_cache: Optional[dict] = None,
        fallback_to_local: bool = True,
    ):
        self.redis = redis_client
        self.local = local_cache or {}
        self.fallback_to_local = fallback_to_local
        self._healthy = True
    
    async def get(self, key: str) -> Optional[str]:
        """优先分布式缓存, 故障降级到本地"""
        try:
            value = await self.redis.get(key)
            if value is not None:
                self._healthy = True
                # 写入本地缓存作为备份
                self.local[key] = value
            return value
        except (ConnectionError, TimeoutError, OSError) as e:
            self._healthy = False
            logger.warning(f"Redis unavailable, falling back to local: {e}")
            if self.fallback_to_local:
                return self.local.get(key)
            return None
    
    async def setex(self, key: str, ttl: int, value: str):
        """分布式写入, 同时更新本地"""
        self.local[key] = value
        try:
            await self.redis.setex(key, ttl, value)
            self._healthy = True
        except (ConnectionError, TimeoutError, OSError) as e:
            self._healthy = False
            logger.warning(f"Redis unavailable, write to local only: {e}")
    
    async def health_check(self) -> dict:
        """返回缓存层健康状态"""
        try:
            await self.redis.ping()
            return {"healthy": True, "mode": "distributed"}
        except Exception:
            if self.fallback_to_local:
                return {"healthy": True, "mode": "local_fallback"}
            return {"healthy": False, "mode": "unavailable"}
```

---

## 序列化与压缩

LLM 响应通常是文本密集型 JSON，压缩可以显著减少网络传输和内存占用。不同的压缩算法在速度、压缩比和 CPU 开销上有不同的权衡。

### 压缩算法对比

| 算法 | 压缩比 (文本) | 压缩速度 | 解压速度 | CPU 开销 |
|------|--------------|----------|----------|----------|
| 无压缩 | 1.0x | - | - | 无 |
| gzip (level 6) | 3-5x | 慢 | 中 | 高 |
| snappy | 1.5-2.5x | 快 | 极快 | 低 |
| zstd (level 3) | 3-6x | 中 | 快 | 中 |
| zstd (level 1) | 2.5-4x | 快 | 快 | 低 |

### 按响应大小动态选择

```python
import gzip
import snappy
import zstandard as zstd
from typing import Tuple

class CacheCompressor:
    """智能缓存压缩器——按响应大小动态选择算法"""
    
    # 阈值: 小于此值不压缩
    MIN_COMPRESS_SIZE = 1024  # 1KB
    # 阈值: 超过此值使用高压缩比算法
    HIGH_COMPRESS_SIZE = 10240  # 10KB
    
    @staticmethod
    def compress(data: str) -> Tuple[bytes, str]:
        """
        压缩并返回 (压缩后数据, 算法标识)
        算法标识用于解压时选择对应算法
        """
        raw_bytes = data.encode("utf-8")
        size = len(raw_bytes)
        
        if size < CacheCompressor.MIN_COMPRESS_SIZE:
            # 小数据: 不压缩
            return raw_bytes, "none"
        
        if size < CacheCompressor.HIGH_COMPRESS_SIZE:
            # 中数据: snappy (速度优先)
            compressed = snappy.compress(raw_bytes)
            return compressed, "snappy"
        else:
            # 大数据: zstd (压缩比优先)
            cctx = zstd.ZstdCompressor(level=3)
            compressed = cctx.compress(raw_bytes)
            return compressed, "zstd"
    
    @staticmethod
    def decompress(data: bytes, algorithm: str) -> str:
        """根据算法标识解压"""
        if algorithm == "none":
            return data.decode("utf-8")
        elif algorithm == "snappy":
            return snappy.decompress(data).decode("utf-8")
        elif algorithm == "zstd":
            dctx = zstd.ZstdDecompressor()
            return dctx.decompress(data).decode("utf-8")
        else:
            raise ValueError(f"Unknown compression algorithm: {algorithm}")
```

---

## Python 代码示例：完整 DistributedCache

```python
"""
distributed_cache.py -- Agent 系统的分布式缓存封装

支持:
  - Redis Cluster 连接池
  - 序列化/反序列化
  - 智能压缩 (自动按大小选择算法)
  - 自动重连
  - 降级到本地缓存
  - 指标收集
"""

import asyncio
import json
import logging
import time
from dataclasses import dataclass
from typing import Any, Callable, Dict, List, Optional, Tuple

logger = logging.getLogger(__name__)


@dataclass
class CacheStats:
    hits: int = 0
    misses: int = 0
    local_hits: int = 0
    bytes_saved: int = 0
    compression_ratio: float = 1.0
    
    @property
    def hit_rate(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0.0
    
    def report(self) -> dict:
        return {
            "hit_rate": f"{self.hit_rate:.2%}",
            "distributed_hits": self.hits - self.local_hits,
            "local_fallback_hits": self.local_hits,
            "misses": self.misses,
            "compression_ratio": f"{self.compression_ratio:.2f}x",
            "bytes_saved": self.bytes_saved,
        }


class DistributedCache:
    """Agent 系统分布式缓存统一封装"""
    
    def __init__(
        self,
        backend: DistributedCacheBackend,
        local_cache_size: int = 1000,
        local_cache_ttl: int = 60,
        enable_compression: bool = True,
        stats_interval: float = 60.0,
    ):
        self.backend = backend
        self.local_cache: Dict[str, Tuple[str, float]] = {}
        self.local_max_size = local_cache_size
        self.local_ttl = local_cache_ttl
        self.enable_compression = enable_compression
        self.stats = CacheStats()
        
        # 自动重连状态
        self._reconnecting = False
        self._max_retries = 3
        self._retry_delay = 0.5
        
        # 启动定期统计输出
        self._stats_task = asyncio.create_task(
            self._periodic_stats(stats_interval)
        )
    
    async def get(
        self,
        key: str,
        compute_fn: Optional[Callable[[], Awaitable[str]]] = None,
        ttl: int = 3600,
    ) -> str:
        """
        获取缓存值——多级缓存查找:
          1. L1 本地缓存 (纳秒级)
          2. L2 分布式缓存 (毫秒级) 
          3. 计算回填 (秒级, 仅当提供 compute_fn)
        """
        # L1: 本地缓存
        now = time.time()
        if key in self.local_cache:
            value, expire = self.local_cache[key]
            if expire > now:
                self.stats.hits += 1
                self.stats.local_hits += 1
                return value
            else:
                del self.local_cache[key]
        
        # L2: 分布式缓存
        try:
            raw = await self._with_retry(self.backend.get, key)
            if raw is not None:
                value = await self._deserialize(raw)
                self._update_local_cache(key, value)
                self.stats.hits += 1
                return value
        except Exception as e:
            logger.warning(f"Distributed cache get failed: {e}")
            # 检查本地缓存中是否有过期但可用的值
            if key in self.local_cache:
                value, _ = self.local_cache[key]
                self.stats.hits += 1
                self.stats.local_hits += 1
                return value
        
        # MISS: 计算回填
        self.stats.misses += 1
        if compute_fn is None:
            raise KeyError(f"Cache miss for {key} and no compute_fn provided")
        
        value = await compute_fn()
        await self.set(key, value, ttl)
        return value
    
    async def set(self, key: str, value: str, ttl: int = 3600):
        """写入分布式缓存 + 本地缓存"""
        self._update_local_cache(key, value)
        
        try:
            raw = await self._serialize(value)
            await self._with_retry(self.backend.setex, key, ttl, raw)
        except Exception as e:
            logger.warning(f"Distributed cache set failed: {e}")
    
    async def delete(self, key: str):
        """失效缓存（分布式 + 本地）"""
        self.local_cache.pop(key, None)
        try:
            await self._with_retry(self.backend.delete, key)
        except Exception as e:
            logger.warning(f"Distributed cache delete failed: {e}")
    
    def _update_local_cache(self, key: str, value: str):
        """更新本地缓存（LRU 淘汰）"""
        if len(self.local_cache) >= self.local_max_size:
            # 移除最早过期的条目
            oldest = min(
                self.local_cache.items(),
                key=lambda x: x[1][1]
            )
            del self.local_cache[oldest[0]]
        
        self.local_cache[key] = (
            value,
            time.time() + self.local_ttl
        )
    
    async def _serialize(self, value: str) -> str:
        """序列化 + 可选压缩"""
        if not self.enable_compression:
            return value
        
        compressed, algorithm = CacheCompressor.compress(value)
        
        # 压缩后数据可能比原始还大（小数据），此时不压缩
        if algorithm == "none" or len(compressed) >= len(value.encode()):
            return json.dumps({
                "alg": "none",
                "data": value,
            })
        
        ratio = len(value.encode()) / len(compressed)
        self.stats.compression_ratio = ratio
        self.stats.bytes_saved += len(value.encode()) - len(compressed)
        
        return json.dumps({
            "alg": algorithm,
            "data": compressed.hex(),  # bytes -> hex string for Redis
        })
    
    async def _deserialize(self, raw: str) -> str:
        """解包 + 解压"""
        try:
            packed = json.loads(raw)
            algorithm = packed.get("alg", "none")
            data = packed["data"]
            
            if algorithm == "none":
                return data
            
            compressed = bytes.fromhex(data)
            return CacheCompressor.decompress(compressed, algorithm)
        except (json.JSONDecodeError, KeyError, ValueError):
            # 兼容未压缩的原始数据
            return raw
    
    async def _with_retry(self, fn: Callable, *args, **kwargs) -> Any:
        """带自动重连的 Redis 操作"""
        last_exc = None
        for attempt in range(self._max_retries):
            try:
                return await fn(*args, **kwargs)
            except (ConnectionError, TimeoutError, OSError) as e:
                last_exc = e
                logger.warning(
                    f"Redis operation failed (attempt {attempt + 1}): {e}"
                )
                if attempt < self._max_retries - 1:
                    await asyncio.sleep(
                        self._retry_delay * (2 ** attempt)
                    )
                    await self._try_reconnect()
        raise last_exc
    
    async def _try_reconnect(self):
        """尝试重连 Redis"""
        if self._reconnecting:
            return
        self._reconnecting = True
        try:
            # 具体重连逻辑取决于 backend 实现
            logger.info("Attempting to reconnect to Redis...")
        finally:
            self._reconnecting = False
    
    async def _periodic_stats(self, interval: float):
        """定期输出缓存统计"""
        while True:
            await asyncio.sleep(interval)
            report = self.stats.report()
            logger.info(f"Cache stats: {json.dumps(report)}")
    
    async def close(self):
        """优雅关闭"""
        self._stats_task.cancel()
        # 清理 backend 连接


# === 使用示例 ===

async def example_usage():
    # 假设已有 Redis Cluster 后端
    backend = RedisClusterBackend([
        ("redis-node-1", 6379),
        ("redis-node-2", 6379),
        ("redis-node-3", 6379),
    ])
    
    cache = DistributedCache(
        backend=backend,
        local_cache_size=1000,
        enable_compression=True,
    )
    
    # Agent 调用缓存
    async def call_llm(prompt: str) -> str:
        """模拟 LLM 调用"""
        await asyncio.sleep(0.5)  # 模拟 LLM 延迟
        return f"Response to: {prompt[:50]}..."
    
    prompt = "What is the weather in Shanghai?"
    cache_key = f"llm:resp:{hash(prompt)}"
    
    # 第一次调用：MISS -> 计算回填
    response = await cache.get(
        cache_key,
        compute_fn=lambda: call_llm(prompt),
        ttl=3600,
    )
    print(f"Response (computed): {response}")
    
    # 第二次调用：HIT
    response = await cache.get(
        cache_key,
        compute_fn=lambda: call_llm(prompt),
        ttl=3600,
    )
    print(f"Response (cached): {response}")
    
    # 输出统计
    print(cache.stats.report())
    # -> {'hit_rate': '50.00%', 'distributed_hits': 1, 
    #     'local_fallback_hits': 1, 'misses': 1, 
    #     'compression_ratio': '3.50x', 'bytes_saved': 1536}

# asyncio.run(example_usage())
```

---

## 热点 Key 问题

### 问题根源

在 Agent 系统中，某些 Key 会被所有实例和所有请求同时命中：

- **全局 System Prompt**: 如 "你是一个智能助手"——所有 Session 共享
- **高频工具定义**: 如 `search_tool` 的 Schema——每次对话都会引用
- **通用 Few-shot**: 所有 Agent 共用的示例

这些 Key 在 Redis Cluster 中会被路由到同一个节点，造成单节点瓶颈。

### 多层缓存解决方案

```
请求 -> L1 本地缓存 (每实例) -> HIT (99% 情况)
    |
    MISS -> L2 分布式缓存 (读多写少的共享池)
         |
         MISS -> L3 计算 (LLM 调用)
```

对于 System Prompt 这样的全局热点，L1 本地缓存可以拦截绝大部分请求，只有 L1 失效或新实例启动时才穿透到 L2。

```python
class TieredCacheStrategy:
    """多层缓存策略——专门应对热点 Key"""
    
    def __init__(self, distributed: DistributedCache):
        self.distributed = distributed
        # 热点 Key 的本地缓存使用更长的 TTL
        self.hot_local_cache: Dict[str, Tuple[str, float]] = {}
        self.hot_ttl = 300  # 热点本地缓存 5 分钟
        self.cold_ttl = 60  # 普通本地缓存 1 分钟
    
    def is_hot_key(self, key: str) -> bool:
        """判断是否为热点 Key"""
        return key.startswith("sys_prompt:") or \
               key.startswith("tool_schema:global:")
    
    async def get(self, key: str, **kwargs):
        if self.is_hot_key(key):
            # 热点 Key: 延长本地缓存时间
            # 即使分布式缓存更新了, 本地最多滞后 hot_ttl 秒
            # 对于不变的系统配置, 这是可以接受的
            return await self._get_hot(key, **kwargs)
        return await self.distributed.get(key, **kwargs)
    
    async def _get_hot(self, key: str, **kwargs):
        now = time.time()
        if key in self.hot_local_cache:
            value, expire = self.hot_local_cache[key]
            if expire > now:
                return value
        
        value = await self.distributed.get(key, **kwargs)
        self.hot_local_cache[key] = (value, now + self.hot_ttl)
        return value
```

---

## 监控与运维

### 关键监控指标

| 指标 | 获取方式 | 告警阈值 | 说明 |
|------|----------|----------|------|
| 缓存命中率 | `INFO stats hits/misses` | < 60% | 低于阈值说明缓存效率低 |
| 内存使用率 | `INFO memory used_memory` | > 80% maxmemory | 可能导致 OOM 或大量淘汰 |
| 淘汰率 | `INFO stats evicted_keys` | > 100/s | 淘汰过多说明内存不足 |
| 慢查询 | `SLOWLOG GET 10` | > 50ms | 慢查询影响 Agent 响应时间 |
| 连接数 | `INFO clients connected_clients` | > maxclients 80% | 连接耗尽导致新请求失败 |
| 复制延迟 | `INFO replication master_repl_offset` | > 1000 | 主从延迟可能导致数据不一致 |
| 缓存穿透 | 监控 MISS + DB 查询率 | 同比 > 200% | 大量不存在 Key 的请求 |
| 缓存击穿 | 监控单个热点 Key 的瞬时 MISS | 突增 > 500% | 热点 Key 过期被打穿 |

### 缓存穿透/击穿/雪崩防护

```
缓存穿透 (Cache Penetration):
  问题: 请求的 Key 在缓存和 DB 中都不存在，每次都穿透
  防护: Bloom Filter 预检测 + 空值缓存 (short TTL)
  
  代码:
  ```python
  # 空值缓存
  value = await cache.get(key)
  if value is None:
      value = await db.query(key)
      if value is None:
          # 空值也缓存，但 TTL 很短
          await cache.setex(key, 30, "__NULL__")
          return None
  ```

缓存击穿 (Cache Breakdown):
  问题: 单个热点 Key 在过期瞬间被大量请求同时打到 DB
  防护: 互斥锁 (Mutex) + 本地缓存 + 逻辑过期
  
  代码:
  ```python
  # 互斥锁重建缓存
  async def rebuild_cache(key):
      if await cache.setnx(f"lock:{key}", "1", ttl=5):
          value = await compute_fn()
          await cache.setex(key, 3600, value)
          await cache.delete(f"lock:{key}")
          return value
      await asyncio.sleep(0.1)
      return await cache.get(key)  # 等待其他线程重建
  ```

缓存雪崩 (Cache Avalanche):
  问题: 大量缓存同时过期，所有请求瞬间打到 DB
  防护: TTL 增加随机偏移 + 多级缓存 + 限流熔断
  
  代码:
  ```python
  import random
  
  # 在 TTL 基础上增加随机偏移
  def jitter_ttl(base_ttl: int, jitter_percent: float = 0.1) -> int:
      jitter = int(base_ttl * jitter_percent)
      return base_ttl + random.randint(-jitter, jitter)
  ```
```

### 内存碎片整理

Redis 长期运行后会产生内存碎片，特别是频繁写入和淘汰大 Value（如 LLM 响应缓存）时。

```shell
# 检查内存碎片率
redis-cli INFO memory | grep mem_fragmentation_ratio
# mem_fragmentation_ratio: 1.5  (正常 < 1.5, > 2.0 需要处理)

# 自动碎片整理 (Redis 4.0+)
redis-cli CONFIG SET activedefrag yes
redis-cli CONFIG SET active-defrag-threshold-lower 10
redis-cli CONFIG SET active-defrag-threshold-upper 100
redis-cli CONFIG SET active-defrag-ignore-bytes 100mb
redis-cli CONFIG SET active-defrag-cycle-min 5
redis-cli CONFIG SET active-defrag-cycle-max 75

# 手动触发 (线上谨慎使用)
redis-cli DEBUG DEFRAGMENT-memory
```

### 运维检查清单

```
每日巡检:
  [ ] 缓存命中率是否正常 (> 60%)
  [ ] 内存使用率是否在预期范围 (< 70%)
  [ ] 淘汰率是否异常 (< 100/s)
  [ ] 主从复制延迟是否正常 (< 1000 offset)
  [ ] Redis 慢查询日志检查

部署前:
  [ ] maxmemory 设置是否正确
  [ ] 淘汰策略是否合理 (allkeys-lru / volatile-lru)
  [ ] 是否配置了密码和 ACL
  [ ] 持久化策略是否匹配 RPO 要求
  [ ] 监控告警阈值已配置

容量规划:
  [ ] 当前内存使用量
  [ ] 每日增长量
  [ ] 按 Agent 请求量预测扩容时间
  [ ] 考虑使用 Redis 的 内存碎片率 + 峰值内存
```

---

## 总结

分布式缓存是 Agent 系统从单实例走向多实例的关键基础设施。核心要点：

1. **Redis/Valkey 是 Agent 系统的标准选择**——丰富的数据结构不仅支持缓存，还能承载 Session、队列、限流等职责
2. **多层缓存架构是最佳实践**——L1 本地内存解决延迟，L2 分布式解决命中率，L3 计算作为兜底
3. **序列化与压缩不可忽视**——LLM 响应的压缩可以节省 3-5x 的网络和内存开销
4. **高可用是生产级要求**——Sentinel/Cluster 自动故障转移 + 客户端降级到本地缓存
5. **热点 Key 需要特殊处理**——散列 + 本地缓存 + 多层分离是有效方案

在 Agent 生产部署中，分布式缓存不仅是性能手段，更是成本控制的核心工具。每提升 10% 的缓存命中率，就是 10% 的 LLM 成本节约——这在规模化部署中可能是每月数万美元的差距。
