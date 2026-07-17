# 6.2.3 结构化记忆

## 简单介绍

结构化记忆（Structured Memory）是利用传统关系型数据库（RDBMS）作为 AI Agent 长期记忆的后端存储，以高度组织化的表格形式管理可结构化的人类知识和 Agent 内部状态。与向量存储的"模糊语义匹配"不同，结构化记忆擅长精确查询、条件过滤和多维关联分析，是 Agent 记忆系统中不可或缺的"结构化骨架"。

## 基本原理

结构化记忆的本质是将 Agent 需要持久化的信息按照预定义 Schema 组织为行和列，通过 SQL 查询实现精确的 CRUD 操作和复杂条件检索。

### 核心数据模型

```
用户表 (users)                   对话表 (conversations)             知识条目表 (knowledge_items)
┌──────────────┐               ┌──────────────────────┐          ┌──────────────────────────┐
│ user_id (PK) │──┐           │ conv_id (PK)         │          │ item_id (PK)             │
│ name         │  └───┐       │ user_id (FK)         │──┐       │ conv_id (FK)             │
│ preferences  │      │       │ created_at            │  └───┐  │ content_summary          │
│ created_at   │      │       │ summary               │      │  │ importance_score         │
│ last_active  │      │       │ token_count           │      │  │ created_at               │
│ metadata     │      │       │ metadata              │      │  │ category                 │
└──────────────┘      │       └──────────────────────┘      │  │ tags                     │
                      │                                      │  │ access_count             │
                      │                                      │  └──────────────────────────┘
                      │                                      │
                      │   记忆条目表 (memory_entries)         │
                      │   ┌──────────────────────────┐       │
                      └───┤ entry_id (PK)            │       │
                          │ user_id (FK)             │───────┘
                          │ conv_id (FK)             │
                          │ entry_type               │
                          │ content                  │
                          │ embedding_id              │
                          │ created_at               │
                          │ expires_at               │
                          └──────────────────────────┘
```

### 典型查询模式

```sql
-- 精确检索：获取特定用户的所有高重要性记忆
SELECT content, created_at, importance_score
FROM memory_entries
WHERE user_id = 'user_123'
  AND importance_score >= 0.8
  AND expires_at > NOW()
ORDER BY created_at DESC
LIMIT 20;

-- 时间范围查询：过去一周内创建的记忆
SELECT content, category, created_at
FROM memory_entries
WHERE created_at >= NOW() - INTERVAL '7 days'
ORDER BY importance_score DESC;

-- 聚合分析：各类别记忆的数量和平均重要性
SELECT category,
       COUNT(*) as count,
       AVG(importance_score) as avg_importance
FROM memory_entries
GROUP BY category
ORDER BY count DESC;

-- 跨表关联：找出活跃用户的未过期记忆
SELECT u.name, m.content, m.created_at
FROM users u
JOIN memory_entries m ON u.user_id = m.user_id
WHERE u.last_active > NOW() - INTERVAL '30 days'
  AND m.expires_at > NOW()
  AND m.entry_type = 'preference';
```

## 背景与演进

结构化记忆在 Agent 系统中的应用经历了三个阶段：

**第一阶段：简单配置存储（2015-2019）** — 早期对话系统使用 SQLite 或键值存储存储用户偏好和会话状态，结构简单，通常是单表扁平存储。这个阶段最大的问题是无法处理非结构化信息，Agent 的"记忆"仅限于预设字段。

**第二阶段：记忆的关系建模（2020-2022）** — 随着 Agent 系统复杂度提升，多表关联、时间衰减、重要性评分等机制被引入。这个阶段的代表性工作包括：SQLAlchemy ORM 管理的多表记忆架构、基于触发器的自动过期清理。

**第三阶段：混合记忆架构（2023-至今）** — 结构化存储不再独立运作，而是作为混合记忆系统的一部分。结构化存储管理精确的元数据和关系，向量存储管理非结构化内容，两者通过 ID 关联（structured metadata + vector content pattern）。同时，pgvector 等扩展使得 PostgreSQL 成为融合结构化 + 向量的统一平台。

## 核心矛盾

### 1. Schema 刚性 vs 记忆的不可预测性

Agent 需要记忆的内容无法在设计时完全穷举。固定 Schema 在面对新型记忆类型时需要 DDL 操作（ALTER TABLE），而 Agent 运行时的动态信息可能无法适配预定义结构。解决方案包括 JSON/JSONB 字段、EAV（Entity-Attribute-Value）模式或 Schema-less 文档字段。

### 2. 精确性 vs 灵活性

SQL 查询精确到行和列，但也要求用户明确知道查询条件。Agent 的"模糊回忆"（"我记得讨论过关于性能优化的内容"）很难直接翻译为精确的 SQL WHERE 子句。结构化记忆需要和向量搜索配合才能实现灵活检索。

### 3. 写入开销 vs 记忆容量

Agent 每轮交互可能产生大量可记录信息，如果每条都写入结构化存储，写入压力会很大。而当记忆条目积累到亿级时，即使索引完善的表，复杂查询的延迟也会上升。需要引入批量写入、分区表、异步持久化等策略。

## 主流优化方向

1. **JSON/JSONB 灵活列**：使用 PostgreSQL 的 JSONB 类型存储变结构元数据，既保留 SQL 的查询能力（JSONB 支持索引和部分查询），又具备 Schema-less 的灵活性。
2. **时间分区 + 自动淘汰**：按时间范围（天/周/月）分区表，结合 TTL（Time-To-Live）策略和定时任务，自动清理过时记忆。
3. **物化视图（Materialized View）**：将频繁查询的复杂 JOIN 结果预计算为物化视图，定期刷新，查询延迟从秒级降到毫秒级。
4. **读写分离 + 连接池**：读扩展（Read Replica）分担查询压力，PgBouncer 等连接池管理器优化连接管理。
5. **全文搜索集成**：利用 PostgreSQL 的内置全文搜索（tsvector/tsquery）或扩展的全文检索引擎，在结构化存储内部实现关键词搜索，避免额外依赖。

## 产生的结果

- 精确查询延迟：< 1ms（主键查询），< 10ms（条件查询 + 索引），< 100ms（复杂 JOIN + 聚合）。
- 单表可支持数亿行记录（合理索引 + 分区）。
- 写入吞吐：单节点可达 10,000+ 行/秒（批量写入）。
- JSONB 灵活列可覆盖 80% 以上的"半结构化记忆"场景。

## 最大挑战

### 1. Schema 演化（Schema Evolution）

Agent 的记忆需求随时间变化，不断需要新增、修改或删除字段。PostgreSQL 的 DDL（如 ALTER TABLE ADD COLUMN）在大型表上是阻塞操作，可能导致分钟级的写入中断。

**解决方案**：
- **零宕机迁移工具**：pgroll、gh-ost-like 的迁移工具。
- **双 Schema 兼容期**：新旧 Schema 并存，逐步迁移数据。
- **JSONB 兜底**：对于可能变化的字段，预先存储在 JSONB 列中，避免频繁 DDL。

### 2. 模糊检索能力不足

结构化存储无法处理"与这段内容语义相似"的查询。这是结构化和向量存储互补的根本原因。任何纯结构化的 Agent 记忆系统都需要某种形式的语义检索补充。

### 3. 运维复杂度

与无服务器向量数据库相比，自托管 PostgreSQL 需要 DBA 技能：配置 WAL 归档、流复制、VACUUM 调优、查询计划分析等。这在小团队场景下是显著负担。

## 能力边界

- **不支持高维向量搜索**：原生 SQL 不支持高效的近似最近邻搜索（pgvector 是扩展，不是核心功能）。
- **写扩展困难**：水平扩展（分片）比 NoSQL 数据库复杂得多，通常需要应用层分片逻辑或 Citus 等扩展。
- **不擅长期时间序列**：虽然可以存储时间序列数据，但性能和压缩率远不如专门的时序数据库（TimescaleDB、InfluxDB）。
- **全文搜索质量有限**：PostgreSQL 全文搜索在中文分词、语义理解上远不如 Elasticsearch。

## 区别对比：SQLite vs PostgreSQL

| 对比维度 | SQLite | PostgreSQL |
|---------|--------|-----------|
| **部署方式** | 嵌入式，无需服务进程 | 独立服务进程 |
| **并发写入** | 单写入者（文件锁） | 多写入者（MVCC） |
| **存储上限** | ~281 TB（理论），实际 < 100 GB | 无限制 |
| **JSON 支持** | 有限（JSON1 扩展） | 原生 JSONB + 索引 |
| **全文搜索** | FTS5 扩展（英文好） | tsvector/tsquery（支持中文扩展） |
| **复制** | 不支持 | 流复制、逻辑复制 |
| **分词** | 无原生中文支持 | zhparser、jieba 扩展 |
| **适用规模** | 单 Agent / 单设备 | 服务器端、多租户 |
| **启动成本** | 零配置 | 需要配置和运维 |
| **pgvector** | 不支持 | 原生支持 |

### SQLite 适用场景

- 个人 Agent / 桌面应用 / 移动端 Agent
- 嵌入式设备上的本地记忆存储
- 开发原型、单用户场景
- 需要在本地完全离线运行

### PostgreSQL 适用场景

- 多用户 Agent 平台（服务端）
- 需要高并发和大容量存储
- 混合存储架构（结构化 + 向量）
- 需要数据备份、复制、高可用

## 核心优势

- **精确性**：SQL 查询结果确定、可重复、可验证，适合需要精准匹配的记忆场景（用户 ID、配置项、时间戳）。
- **关系建模**：天然支持实体之间的关联（用户-对话-记忆之间的多对多关系），通过 JOIN 可以跨越多个维度查询。
- **事务保证**：ACID 事务确保记忆操作的原子性和一致性，不会出现写入一半崩溃导致数据不一致的情况。
- **成熟生态**：30+ 年发展，工具链完备（迁移、备份、监控、GUI）。

## 工程优化

### Schema 设计最佳实践

```sql
-- 记忆条目表（优化设计示例）
CREATE TABLE memory_entries (
    entry_id        BIGSERIAL PRIMARY KEY,
    user_id         VARCHAR(64) NOT NULL,
    session_id      VARCHAR(64),
    entry_type      VARCHAR(32) NOT NULL,  -- 'fact', 'preference', 'event', 'observation'
    importance      FLOAT NOT NULL DEFAULT 0.5,  -- [0, 1]
    content_text    TEXT,
    content_json    JSONB,                -- 变结构内容
    embedding_id    VARCHAR(64),          -- 关联的向量存储 ID
    metadata        JSONB DEFAULT '{}',   -- 元数据（可扩展）
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ,          -- NULL 表示永不过期
    access_count    INTEGER DEFAULT 0,
    last_accessed   TIMESTAMPTZ
);

-- 关键索引
CREATE INDEX idx_memory_user_type ON memory_entries(user_id, entry_type);
CREATE INDEX idx_memory_created ON memory_entries(user_id, created_at DESC);
CREATE INDEX idx_memory_importance ON memory_entries(user_id, importance DESC);
CREATE INDEX idx_memory_expires ON memory_entries(expires_at) WHERE expires_at IS NOT NULL;
-- JSONB 索引
CREATE INDEX idx_memory_metadata ON memory_entries USING GIN (metadata);
-- 全文搜索索引
CREATE INDEX idx_memory_fts ON memory_entries USING GIN (to_tsvector('english', content_text));
```

### 自动过期与衰减

```sql
-- 定期清理过时记忆
CREATE OR REPLACE FUNCTION clean_expired_memories()
RETURNS void AS $$
BEGIN
    DELETE FROM memory_entries
    WHERE expires_at IS NOT NULL
      AND expires_at < NOW() - INTERVAL '1 hour';  -- 1 小时宽限期
END;
$$ LANGUAGE plpgsql;

-- 每小时执行一次
SELECT cron.schedule('0 * * * *', 'SELECT clean_expired_memories();');

-- 重要性衰减：长时间未访问的记忆自动降权
UPDATE memory_entries
SET importance = importance * 0.95
WHERE last_accessed < NOW() - INTERVAL '30 days'
  AND importance > 0.1;
```

### 批量写入优化（Python）

```python
import asyncpg
from dataclasses import dataclass

@dataclass
class MemoryEntry:
    user_id: str
    entry_type: str
    content_text: str
    importance: float
    metadata: dict = None
    expires_at: str = None

class StructuredMemoryStore:
    def __init__(self, dsn: str):
        self.dsn = dsn
        self.pool = None

    async def connect(self):
        self.pool = await asyncpg.create_pool(self.dsn, min_size=5, max_size=20)

    async def batch_insert(self, entries: list[MemoryEntry]):
        """批量插入记忆条目"""
        async with self.pool.acquire() as conn:
            await conn.executemany(
                """
                INSERT INTO memory_entries
                    (user_id, entry_type, content_text, importance, metadata, expires_at)
                VALUES ($1, $2, $3, $4, $5::jsonb, $6::timestamptz)
                """,
                [
                    (e.user_id, e.entry_type, e.content_text,
                     e.importance, json.dumps(e.metadata or {}), e.expires_at)
                    for e in entries
                ]
            )

    async def search_exact(self, user_id: str, entry_type: str = None,
                           min_importance: float = 0.0, limit: int = 20):
        """精确检索记忆条目"""
        async with self.pool.acquire() as conn:
            if entry_type:
                rows = await conn.fetch(
                    """SELECT * FROM memory_entries
                       WHERE user_id = $1 AND entry_type = $2
                         AND importance >= $3 AND (expires_at IS NULL OR expires_at > NOW())
                       ORDER BY importance DESC, created_at DESC
                       LIMIT $4""",
                    user_id, entry_type, min_importance, limit
                )
            else:
                rows = await conn.fetch(
                    """SELECT * FROM memory_entries
                       WHERE user_id = $1 AND importance >= $2
                         AND (expires_at IS NULL OR expires_at > NOW())
                       ORDER BY importance DESC, created_at DESC
                       LIMIT $3""",
                    user_id, min_importance, limit
                )
            return [dict(row) for row in rows]
```

### JSONB 内容检索示例

```sql
-- 在 JSONB metadata 中检索特定结构的数据
SELECT * FROM memory_entries
WHERE metadata @> '{"source": "web", "confidence": 0.9}';

-- JSONB 路径查询
SELECT * FROM memory_entries
WHERE metadata #>> '{location, city}' = 'Beijing';

-- JSONB 存在性检查
SELECT * FROM memory_entries
WHERE metadata ? 'priority';
```

## 场景判断

### 适合结构化记忆的场景

- **用户偏好和配置**：明确的键值对关系，精确匹配（"语言偏好：中文"）。
- **权限和身份信息**：需要精确匹配和安全保证（"只有用户 X 可以访问 Y"）。
- **时间线/事件序列**：Agent 需要按时间顺序精确回溯历史。
- **计数和聚合统计**：Agent 需要"最近 7 天处理了多少个任务"等精确统计。
- **关系查询**：Agent 需要理解实体之间的关联（"用户 A 在项目 B 中担任 C 角色"）。

### 不适合结构化记忆的场景

- **非结构化语义搜索**：Agent 需要"回忆关于性能优化的讨论"这种模糊查询。
- **快速原型/频繁 Schema 变更**：Agent 的记忆类型还在快速迭代。
- **高写入负载的纯日志场景**：Agent 的每步操作日志（此时时序数据库更合适）。
