# 6.2.4 key-value-memory — KV 存储（Redis / Memcached）

## 简单介绍

KV 存储（Key-Value Memory）是 Agent 长期记忆中最轻量、最快速的存储方案——以"键-值"对的形式存储和检索信息。与向量存储的语义搜索和结构化存储的关系查询不同，KV 存储提供 **O(1) 的精确查找**，适合存储需要快速访问的简单数据：会话状态、配置信息、缓存结果、用户身份标识等。

在 Agent 系统中，KV 存储通常扮演"工作记忆的后备存储"和"长期记忆的快速缓存"两个角色。

## 基本原理

### 数据模型

KV 存储的核心模型极其简单：

```
key  →  value
           |
           ├── 字符串 (String)    → "user_123:preference:theme = dark"
           ├── 哈希 (Hash)        → "user_123:session = {task, status, step}"
           ├── 列表 (List)        → "agent:recent_errors = [err1, err2, err3]"
           ├── 集合 (Set)         → "task:completed_ids = {id1, id2, id3}"
           └── 有序集合 (Sorted Set) → "memories:importance = {mem1: 0.9, mem2: 0.7}"
```

### 键设计模式

在 Agent 记忆系统中，键的命名规则直接影响检索效率：

```python
# 命名空间式键设计（推荐）
KEY_PATTERNS = {
    "session":        "agent:{agent_id}:session:{session_id}",
    "preference":     "user:{user_id}:preference:{key}",
    "task_state":     "agent:{agent_id}:task:{task_id}:state",
    "cache":          "cache:{agent_id}:{query_hash}",
    "counter":        "stats:{agent_id}:{metric}:count",
    "recent_memory":  "agent:{agent_id}:recent:{timestamp}",
}

# 示例
key = f"user:u_42:preference:language"  # → "en"
key = f"agent:a_7:session:sess_abc:step"  # → 3
```

### 读写操作

```python
class KVMemory:
    """基于 KV 存储的 Agent 记忆"""

    def __init__(self, redis_client: Redis):
        self.redis = redis_client
        self.default_ttl = 3600  # 1 hour

    # --- 精确读写 ---
    def get(self, namespace: str, key: str) -> Any | None:
        return self.redis.get(f"{namespace}:{key}")

    def set(self, namespace: str, key: str, value: Any, ttl: int | None = None):
        self.redis.setex(f"{namespace}:{key}", ttl or self.default_ttl, value)

    # --- 哈希操作（结构化对象）---
    def hget(self, namespace: str, key: str, field: str) -> Any | None:
        return self.redis.hget(f"{namespace}:{key}", field)

    def hset(self, namespace: str, key: str, mapping: dict):
        self.redis.hset(f"{namespace}:{key}", mapping= mapping)

    # --- 过期管理 ---
    def expire(self, namespace: str, key: str, seconds: int):
        self.redis.expire(f"{namespace}:{key}", seconds)

    # --- 原子操作（用于计数/限流）---
    def increment(self, namespace: str, key: str, amount: int = 1) -> int:
        return self.redis.incrby(f"{namespace}:{key}", amount)

    # --- 批量读取 ---
    def mget(self, namespace: str, keys: list[str]) -> list[Any | None]:
        return self.redis.mget([f"{namespace}:{k}" for k in keys])
```

## 在 Agent 记忆系统中的角色

KV 存储在 Agent 记忆层次中承担三个关键角色：

### 角色 1：工作记忆的持久化后端

```
工作记忆（内存中）              KV 存储（Redis）
┌─────────────────┐           ┌──────────────────────┐
│ current_task     │──持久化──→│ agent:a1:task:current │
│ current_step     │           │ agent:a1:step:current │
│ retry_count      │           │ agent:a1:retry:count  │
│ collected_data   │           │ agent:a1:data:collected│
└─────────────────┘           └──────────────────────┘
  随进程消失                     跨进程/跨会话存活
```

### 角色 2：长期记忆的缓存层

```
读取路径：
LLM 需要信息 → KV 缓存检查 → 命中 → 直接返回
                           → 未命中 → 查询向量/结构化存储 → 写入缓存

写入路径：
新信息产生 → 写入向量/结构化存储 → 同时写入 KV 缓存（预热）
```

### 角色 3：高频操作的快速存储

适合 KV 存储的记忆类型：

| 记忆类型 | 键模式 | 值结构 | TTL |
|---------|--------|--------|-----|
| 会话 Token | `session:{id}:token` | 字符串 | 与 Token 剩余寿命一致 |
| 用户偏好缓存 | `user:{id}:prefs` | Hash | 1 小时 |
| 任务进度 | `task:{id}:progress` | Hash | 任务完成 TTL |
| 最近 N 条工具调用 | `agent:{id}:recent_tools` | List(固定长度) | 1 天 |
| 错误计数 | `stats:{id}:errors` | Counter | 按需重置 |
| 幂等性键 | `idempotent:{key}` | 字符串 | 5 分钟 |

## 背景

### 为什么 Agent 需要 KV 存储

在 Agent 系统的演进过程中，早期方案（2023-2024）使用内存字典（Python dict）存储会话状态，问题明显：

1. **进程崩溃丢失**：Agent 服务重启后所有状态消失
2. **无法水平扩展**：多实例部署时状态不同步
3. **查询能力有限**：无法按时间、类型等条件筛选

KV 存储（特别是 Redis）解决了这些问题：O(1) 读写、进程隔离、支持 TTL 自动过期、发布订阅通知、内置数据结构。

### 与其他存储的定位差异

```
延迟对比（近似值）：
  KV 存储 (Redis):      0.1-1 ms    ← 最低延迟
  结构化存储 (SQLite):   0.5-5 ms
  结构化存储 (PostgreSQL): 1-10 ms
  向量存储 (Chroma):     10-50 ms
  向量存储 (Milvus):     10-100 ms

容量对比（近似值）：
  KV 存储:               内存受限（GB~TB）
  结构化存储:             磁盘容量（TB~PB）
  向量存储:              磁盘容量（TB~PB）
```

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **内存成本 vs 访问速度** | KV 存储（特别是 Redis）将所有数据放在内存中，速度快但成本高 |
| **精确查询 vs 语义搜索** | KV 只支持精确键匹配，无法进行语义搜索——你需要知道"找什么"才能找到 |
| **简单模型 vs 复杂关系** | KV 没有关系模型，无法表达"用户 A 的所有记忆"这类集合查询（需额外的索引键）|
| **持久性 vs 性能** | 开启持久化（RDB/AOF）影响性能；纯内存模式有数据丢失风险 |

## 当前主流优化方向

1. **混合 KV 分层**：热数据在 Redis（内存），冷数据在 SSD-backed KV（RocksDB），自动迁移
2. **键压缩**：长键名使用哈希缩短（`user:abc123:preference:language_zh` → 8 字节 hash）
3. **管道批量操作**：批量读写减少网络 RTT（pipeline/mget/mset）
4. **值序列化优化**：使用 MessagePack、Protocol Buffers 替代 JSON 做值序列化，减少空间和解析开销
5. **缓存失效策略**：结合 LLM 的语义判断主动失效缓存，而非仅依赖 TTL

## 工程优化方向

1. **连接池管理**：使用连接池复用连接，避免每次操作创建新连接
2. **本地缓存层**：在 Agent 进程内加一层 LRU 缓存，减少对 Redis 的访问次数
3. **序列化选择**：
   ```python
   # JSON（可读性好，空间大）
   "agent:state" = '{"task":"search","step":3,"status":"running"}'
   
   # MessagePack（空间省 30%，解析快 2x）
   "agent:state" = b'\x83\xa4task\xa6search\xa4step\x03\xa6status\xa7running'
   
   # Pickle（最快的 Python 序列化，不安全）
   # 注意：只用于内部进程，不要接受外部 Pickle 数据
   ```

4. **键过期策略**：
   - 明确区分永久记忆（无 TTL）和临时缓存（合理 TTL）
   - 使用 `EXPIRE` 命令设置滑动过期（每次访问刷新 TTL）
   - 对于需要批量过期的键，使用键模式扫描 + UNLINK 异步删除

5. **监控指标**：跟踪缓存命中率、平均延迟、内存使用量、过期键数量

## 能力边界与结果边界

**能做的**：
- 以亚毫秒级延迟读写简单数据（100 字节以内最优）
- 通过 TTL 自动清理过时记忆，无需手动管理
- 支持计数器、排行榜、消息队列等高级数据结构

**不能做的**：
- ❌ 语义搜索——无法"找到与 X 相似的记忆"
- ❌ 复杂查询——无法做 JOIN、子查询、聚合
- ❌ 存储大对象——推荐不超过 10MB/值
- ❌ 跨键事务——Redis 的事务能力有限（不支持回滚）

**结果边界**：
- KV 存储应只占 Agent 总记忆存储的 10-20%（其余由向量和结构化存储承担）
- KV 存储最适合存储"你确切知道要查什么"的信息
- Redis 单实例 QPS 可达 10 万+，足以支撑大多数 Agent 系统

## 与其他方案的对比

| 对比维度 | KV 存储 | 向量存储 | 结构化存储 |
|---------|---------|---------|-----------|
| 查询方式 | 精确键查找 | 语义相似度 | SQL/过滤 |
| 写入延迟 | <1ms | 10-50ms | 1-10ms |
| 读取延迟 | <1ms | 10-100ms | 1-10ms |
| 语义理解 | 无 | 有 | 无 |
| 容量上限 | 内存上限 | 磁盘上限 | 磁盘上限 |
| 成本/TB | 高（内存） | 中（SSD） | 低（HDD/SSD） |
| 适用数据 | 会话状态/缓存 | 非结构化文本 | 结构化记录 |

## 核心优势

1. **极速访问**：在 Agent 的推理链路上，KV 操作的时间几乎可以忽略不计
2. **原子操作**：INCR、HSETNX 等原子操作在并发场景下无需锁
3. **内置过期**：TTL 机制天然适配"有时效的记忆"场景
4. **成熟生态**：Redis 有丰富的客户端库、监控工具、集群方案

## 适合场景的判断标准

- **数据结构**：简单结构化数据（字符串/数字/JSON 对象）→ 适合；大文本/向量 → 不适合
- **访问模式**：精确键查询 → 适合；模糊搜索/语义匹配 → 不适合
- **时效性**：临时数据（会话/缓存/计数器）→ 非常适合；永久知识 → 需搭配其他存储
- **延迟要求**：<1ms 要求 → KV 存储首选；可接受 10ms+ → 其他选择也可

**最适合**：会话状态跟踪、用户配置缓存、工具调用幂等性检查、频率限制、工作记忆快照

**最不适合**：文档知识库、语义记忆搜索、关系数据查询、大文件存储
