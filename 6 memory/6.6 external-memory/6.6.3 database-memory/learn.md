# 6.6.3 关系型数据库作为记忆后端（Database Memory）

## 简单介绍

**数据库记忆（Database Memory）** 是指 AI Agent 使用关系型数据库（如 PostgreSQL、SQLite、MySQL）作为持久化记忆后端的模式。在这种架构中，Agent 将自身的经验、知识、对话历史、用户偏好等结构化和半结构化信息，按照预定义的表模式（Schema）存入关系数据库，并通过 SQL 查询实现精确检索和复杂聚合。

与知识图谱（6.6.1）的图结构语义存储不同，数据库记忆的核心是**关系模型**——一切信息被组织为行（记录）和列（字段）的二维表结构，通过外键和 JOIN 操作表达实体间的关联。这种模式的优势在于：

- **强一致性保证**：ACID 事务确保 Agent 的记忆操作要么完全成功要么完全失败，不会出现"一半记住一半忘记"的状态
- **精确查询能力**：SQL 提供精确的字段匹配、范围过滤、排序和聚合，Agent 可以精确回答"上周三下午三点和用户讨论过什么"这样的问题
- **成熟的生态系统**：数十年的数据库工程积累，备份、恢复、复制、监控等运维工具链完整
- **标准化接口**：SQL 是通用语言，切换数据库厂商的成本相对较低

典型的数据库记忆场景包括：

```
Agent 类型        数据库记忆的内容                         查询模式
──────────────────────────────────────────────────────────────────
个人助理 Agent     用户偏好、日程安排、联系人信息           精确查询用户配置
客服 Agent         历史对话记录、工单状态、客户信息          按用户 ID/时间范围检索
代码生成 Agent     项目配置、API 密钥、代码片段存档          键值式精确查找
数据分析 Agent     数据源配置、分析任务历史、缓存结果         复杂聚合查询
```

## 基本原理

### 关系模型作为记忆结构

关系型数据库将 Agent 的记忆组织为**实体-关系模型**。每一个记忆实体映射为一张表，实体之间的关联通过外键实现。

```text
记忆的实体-关系映射：

  记忆实体                       数据库表
  ┌──────────────────────┐      ┌─────────────────────────────┐
  │ Agent 的一次交互经验  │ ──→  │ agent_memories              │
  │                      │      │ ├─ id            (PK)       │
  │ • 交互时间           │      │ ├─ agent_id                 │
  │ • 用户消息           │      │ ├─ session_id               │
  │ • Agent 响应         │      │ ├─ user_message   (TEXT)    │
  │ • 使用工具           │      │ ├─ agent_response (TEXT)    │
  │ • 满意度评分         │      │ ├─ tools_used     (JSONB)   │
  │                      │      │ ├─ satisfaction   (INTEGER) │
  │                      │      │ ├─ created_at     (TIMESTAMP)│
  │                      │      │ └─ metadata       (JSONB)   │
  └──────────────────────┘      └─────────────────────────────┘
```

### SQL 作为记忆检索语言

Agent 通过 SQL 语句从数据库中"回忆"信息。SQL 的查询能力决定了 Agent 能够以何种精度访问过去的记忆：

```sql
-- 精确匹配：昨天和特定用户的对话
SELECT * FROM agent_memories
WHERE agent_id = 'assistant-v2'
  AND user_message ILIKE '%退款%'
  AND created_at >= CURRENT_DATE - INTERVAL '1 day'
ORDER BY created_at DESC;

-- 范围查询：最近一周满意度低于 3 分的交互
SELECT session_id, user_message, satisfaction
FROM agent_memories
WHERE satisfaction < 3
  AND created_at >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY satisfaction ASC;

-- 聚合分析：最常用的工具排名
SELECT tools_used->>'name' AS tool_name,
       COUNT(*) AS usage_count
FROM agent_memories,
     jsonb_array_elements(tools_used) AS tools_used
GROUP BY tool_name
ORDER BY usage_count DESC
LIMIT 10;
```

### ACID 属性对 Agent 记忆的意义

| ACID 属性 | 数据库保证 | 对 Agent 记忆的意义 |
|-----------|-----------|-------------------|
| **原子性（Atomicity）** | 事务全部执行或全部回滚 | Agent 不会陷入"半记忆"状态——要么完整记录一次交互，要么什么都不记录 |
| **一致性（Consistency）** | 数据始终满足约束条件 | Agent 的记忆不会出现引用断裂（如指向不存在的 session） |
| **隔离性（Isolation）** | 并发事务互不干扰 | 多个 Agent 实例同时写入记忆时不会互相污染 |
| **持久性（Durability）** | 已提交事务永不被丢失 | Agent 重启后记忆完好无损，即使系统崩溃也不会丢失已确认的记忆 |

### 三层记忆模型

在数据库记忆架构中，Agent 的记忆被组织为三个层次：

```text
                   ┌─────────────────────────────┐
                   │    语义层（Semantic Layer）    │
                   │  高层抽象，面向 Agent 推理     │
                   │  "用户喜欢简洁的回答"          │
                   └─────────────┬───────────────┘
                                 │ 由应用层逻辑推导
                                 ▼
                   ┌─────────────────────────────┐
                   │    结构化层（Structured Layer）│
                   │  表结构存储，面向 SQL 查询    │
                   │  {session_id, user_msg, resp}│
                   └─────────────┬───────────────┘
                                 │ 由 ORM 映射
                                 ▼
                   ┌─────────────────────────────┐
                   │    物理存储层（Storage Layer）│
                   │  磁盘页/B-tree 索引，面向 I/O │
                   │  PostgreSQL 数据文件          │
                   └─────────────────────────────┘
```

Agent 通常工作在最上层（语义层），通过 ORM 工具（如 SQLAlchemy）操作中层，数据库引擎自动管理底层存储。

## 背景与演进

### 演进时间线

| 时期 | 阶段 | 代表技术 | 核心特征 | 对 Agent 记忆的意义 |
|------|------|----------|----------|-------------------|
| 1960s-1970s | 文件存储时代 | 文件系统、ISAM | 数据以文件形式存储，无结构化查询能力 | Agent 只能读写文件，检索需全量扫描 |
| 1970s-1980s | 关系模型诞生 | System R、Ingres | Codd 提出关系模型，SQL 语言诞生 | Agent 首次具备结构化查询能力 |
| 1980s-1990s | 商用 RDBMS 成熟 | Oracle、DB2、SQL Server | 事务处理、并发控制、查询优化器成熟 | Agent 可以获得 ACID 级记忆保证 |
| 1990s-2000s | 开源 RDBMS 崛起 | MySQL、PostgreSQL | 开源数据库性能追平商用，社区生态繁荣 | Agent 记忆后端的成本大幅降低 |
| 2000s-2010s | 互联网规模扩展 | 分库分表、读写分离、NoSQL | 关系数据库开始支持水平扩展和异构存储 | Agent 可以处理亿级记忆条目 |
| 2010s-2020s | 智能化与多模型 | PostgreSQL 扩展生态 | JSONB、全文检索、列存储、地理空间等 | Agent 可以在同一数据库存储结构化+半结构化记忆 |
| 2020s-至今 | 向量融合 | pgvector、pg_embedding | 关系数据库原生支持向量相似度搜索 | Agent 可以在同一系统中进行精确 SQL 查询和语义检索 |

### 从文件到数据库：Agent 记忆存储的演进

```text
第一阶段：内存存储
  [Agent] ──→ 内存变量/字典
  优点：最快（纳秒级访问）
  缺点：进程退出后全部丢失，无法跨会话共享
  适用：仅当前会话的临时记忆

第二阶段：文件存储
  [Agent] ──→ JSON/YAML 文件
  优点：可以持久化，人工可读
  缺点：无查询能力，需全量加载，并发写入冲突
  适用：配置存储、小型知识库

第三阶段：关系型数据库
  [Agent] ──→ PostgreSQL/SQLite
  优点：ACID 保证、SQL 查询、并发控制、索引加速
  缺点：Schema 固定、无语义理解、扩展复杂度随数据量上升
  适用：结构化记忆、高频事务型记忆

第四阶段：专用向量存储
  [Agent] ──→ Pinecone/Weaviate/Qdrant
  优点：原生语义检索、向量索引、相似度搜索
  缺点：不支持精确查询、事务能力弱、生态较新
  适用：非结构化语义记忆、RAG 场景

第五阶段：融合存储（当前趋势）
  [Agent] ──→ PostgreSQL + pgvector
  优点：同时支持 SQL 精确查询和向量语义检索
  缺点：配置复杂、索引维护成本高
  适用：需要"精确+语义"双重能力的 Agent
```

### Agent 记忆架构的演进驱动力

推动演进的核心矛盾是**记忆规模**与**检索精度**之间的张力：

```
记忆规模增长曲线：
  100 条记录 ────→ 文件存储够用（全量扫描仍可接受）
  10K 条记录 ────→ 需要数据库索引（B-tree 加速精确匹配）
  1M 条记录 ──────→ 需要分区、分片、读写分离
  1B 条记录 ──────→ 需要分布式数据库、列存储
```

每一次存储升级都是因为 Agent 的记忆规模突破了上一阶段的容量或性能瓶颈。

## 核心矛盾

### 结构化查询精度 vs. 语义灵活性

数据库记忆面临的根本矛盾是：**SQL 查询的精确性 vs. 自然语言的模糊性**。

```text
                          ┌─────────────────────┐
                          │    Agent 需要回忆     │
                          │  "上次聊过的那个项目"  │
                          └──────────┬──────────┘
                                     │
                          ┌──────────▼──────────┐
                          │    矛盾爆发点         │
                          │                     │
                          │  "那个项目" = 什么？  │
                          │  项目名称？项目编号？  │
                          │  上个月？上周？昨天？  │
                          └──────────┬──────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
  │ 精确 SQL 查询     │  │ 模糊 LIKE 查询    │  │ 无奈的全表扫描    │
  │                  │  │                  │  │                  │
  │ SELECT * FROM    │  │ SELECT * FROM    │  │ SELECT * FROM    │
  │ projects WHERE   │  │ projects WHERE   │  │ projects         │
  │ name = 'X'       │  │ name LIKE '%项%' │  │ -- 然后应用层过滤 │
  │                  │  │                  │  │                  │
  │ 结果：精确但可能   │  │ 结果：全面但可能   │  │ 结果：全面但极慢  │
  │ 遗漏（"那个项目"  │  │ 噪声（返回不相关  │  │                 │
  │ 可能不是精确名称）│  │ 的记录）          │  │                 │
  └──────────────────┘  └──────────────────┘  └──────────────────┘
```

### 矛盾的具体表现

1. **命名差异**：用户说的"上次那个项目"和数据库中的项目名称可能不是同一个字符串

2. **隐含条件**："最近"是多近？Agent 需要将自然语言时间表达转换为 SQL 的 INTERVAL

3. **模糊关联**：记忆可能分散在多个表中，Agent 需要知道如何 JOIN，但可能不知道表之间的关联关系

4. **不完备查询条件**：用户只记得部分信息（"我记得文件名里有 report..."），无法构造精确的 WHERE 条件

### 工程上的折中

```text
精度 ←──────────────────────────→ 灵活性
      │              │
      ▼              ▼
  精确 SQL        全文检索 +      向量相似度
  完全匹配        PostgreSQL      pgvector
                 tsvector
      │              │              │
  精确但不宽容    灵活但有噪声    语义理解好但
  适用：用户ID、   适用：文本内容  精度不如 SQL
  交易记录、配置  搜索           适用：非结构化
```

### 矛盾的解决思路

解决这一矛盾有两条路径：

1. **增强数据库的语义能力**：引入全文检索（tsvector）、向量相似度搜索（pgvector）、模糊匹配（pg_trgm）等扩展，让关系数据库能够处理"模糊"查询

2. **增强 Agent 的 SQL 生成能力**：让 LLM 直接将自然语言查询转换为 SQL，通过 Few-shot 学习或 Fine-tuning 提高 SQL 生成的准确率（Text-to-SQL 范式）

实践中，大多数生产系统采用**混合策略**——核心实体用精确 SQL，文本内容用全文检索，语义查询用向量搜索。

## 主流优化方向

### 1. Hybrid SQL + Vector（pgvector, pg_embedding）

**核心思想**：在传统关系数据库上叠加向量相似度搜索能力，使 Agent 能够在一次查询中同时利用 SQL 的精确过滤和向量的语义检索。

**工作原理**：

```text
传统 SQL 查询：
  SELECT * FROM memories WHERE agent_id = 'a1' AND created_at > '2024-01-01'

传统向量检索：
  SELECT * FROM vector_store WHERE vector <-> query_embedding < 0.5

Hybrid 查询（pgvector + SQL）：
  SELECT m.*, m.embedding <-> :query_vec AS distance
  FROM agent_memories m
  WHERE m.agent_id = 'a1'
    AND m.created_at > '2024-01-01'
  ORDER BY distance ASC
  LIMIT 10;
```

**优势**：单一查询即可完成"结构过滤 + 语义排序"，无需在数据库和向量存储之间搬运数据。

**代价**：向量索引（IVFFlat、HNSW）需要额外的内存和索引维护成本。

---

### 2. Schema 设计模式

Agent 记忆的 Schema 设计遵循两种经典模式：

#### 事件溯源模式（Event Sourcing Pattern）

将 Agent 的每一次记忆操作建模为不可变的事件流：

```sql
CREATE TABLE agent_events (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        VARCHAR(64) NOT NULL,
    session_id      VARCHAR(64) NOT NULL,
    event_type      VARCHAR(32) NOT NULL,   -- 'message', 'tool_call', 'observation', 'decision'
    event_data      JSONB NOT NULL,          -- 事件载荷
    parent_event_id BIGINT REFERENCES agent_events(id),  -- 因果关联
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_agent_events_agent_time ON agent_events (agent_id, created_at DESC);
CREATE INDEX idx_agent_events_session ON agent_events (session_id, created_at);
CREATE INDEX idx_agent_events_event_type ON agent_events (agent_id, event_type);
```

**优点**：完整的历史可追溯性、支持时间旅行查询、自然支持审计
**缺点**：查询当前状态需要重放事件流、存储量随时间线性增长

#### 实体-属性-值模式（EAV Pattern）

适用于 Agent 需要记忆的实体属性动态变化、无法预定义完整 Schema 的场景：

```sql
CREATE TABLE agent_entity_types (
    type_id     SERIAL PRIMARY KEY,
    agent_id    VARCHAR(64) NOT NULL,
    type_name   VARCHAR(128) NOT NULL,   -- 'project', 'contact', 'note'
    UNIQUE(agent_id, type_name)
);

CREATE TABLE agent_entities (
    entity_id   BIGSERIAL PRIMARY KEY,
    type_id     INTEGER NOT NULL REFERENCES agent_entity_types(type_id),
    entity_key  VARCHAR(256) NOT NULL,   -- 实体标识，如 'project-42'
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(type_id, entity_key)
);

CREATE TABLE agent_entity_attributes (
    attribute_id   BIGSERIAL PRIMARY KEY,
    entity_id      BIGINT NOT NULL REFERENCES agent_entities(entity_id) ON DELETE CASCADE,
    attr_name      VARCHAR(128) NOT NULL,      -- 'name', 'status', 'deadline'
    attr_value     TEXT NOT NULL,               -- 所有值统一存为 TEXT
    attr_type      VARCHAR(16) DEFAULT 'text',  -- 'text', 'number', 'date', 'json'
    updated_at     TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(entity_id, attr_name)
);
```

**优点**：Schema 完全灵活，Agent 可以动态"学习"新的属性类型
**缺点**：查询复杂（需要 PIVOT），性能低于固定列模式，类型安全靠应用层保障

---

### 3. 连接池与查询优化

连接池是数据库记忆架构中不可或缺的组件。Agent 通常以无状态或轻量状态运行，每次需要记忆操作时创建数据库连接的成本极高。

**连接池的作用**：

```text
无连接池：
  [Agent] ──→ 建立 TCP 连接（~1ms） ──→ 认证握手（~5ms） ──→ 查询（~0.3ms） ──→ 关闭连接
  每次查询的额外开销：~6ms（90% 消耗在连接建立上）

有连接池：
  [Agent] ──→ 从池中获取已有连接（~0.01ms） ──→ 查询（~0.3ms） ──→ 归还连接到池
  每次查询的额外开销：~0.01ms（连接复用，几乎为零）
```

**关键优化参数**：

| 参数 | 说明 | 推荐值 | 过大/过小的影响 |
|------|------|--------|---------------|
| `pool_size` | 连接池大小 | CPU 核心数 × 2 | 过大：浪费内存；过小：查询排队 |
| `max_overflow` | 超出 pool_size 的临时连接数 | 10-20 | 过大：峰值时资源耗尽 |
| `pool_timeout` | 等待连接的超时时间（秒） | 5-30 | 过小：突发流量时查询失败 |
| `pool_recycle` | 连接最大存活时间（秒） | 3600 | 过大：连接可能被防火墙断开 |

---

### 4. 全文检索集成

关系数据库内置的全文检索能力是数据库记忆的重要补充：

| 数据库 | 全文检索机制 | 特色 |
|--------|-------------|------|
| PostgreSQL | tsvector / tsquery | 词干分析、词典支持、排名函数 |
| MySQL | FULLTEXT 索引 | InnoDB 原生支持、布尔模式、自然语言模式 |
| SQLite | FTS5 扩展 | 轻量级、BM25 排序、分词器插件 |

PostgreSQL 的全文检索尤其适合 Agent 记忆场景：

```sql
-- 创建记忆表时包含全文检索向量
CREATE TABLE agent_memories (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        VARCHAR(64) NOT NULL,
    content         TEXT NOT NULL,
    content_tsv     TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('simple', coalesce(content, ''))
    ) STORED,
    embedding       VECTOR(384),               -- pgvector 列
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 全文检索索引
CREATE INDEX idx_memories_fts ON agent_memories USING GIN (content_tsv);

-- 精确 + 全文 + 向量三重检索
SELECT id, content,
       ts_rank(content_tsv, query) AS text_rank,
       embedding <-> :query_vec AS vec_distance
FROM agent_memories,
     plainto_tsquery('simple', :keyword) AS query
WHERE content_tsv @@ query
ORDER BY text_rank * 0.3 + (1 - vec_distance) * 0.7 DESC
LIMIT 20;
```

---

### 5. 时间序列分区

Agent 的记忆天然具有时间维度——新记忆不断产生，旧记忆逐渐被遗忘。时间分区使数据库可以高效管理这一时间序列特性。

**分区策略**：

```sql
-- PostgreSQL 按月分区
CREATE TABLE agent_memories (
    id          BIGSERIAL,
    agent_id    VARCHAR(64) NOT NULL,
    content     TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- 创建每月分区
CREATE TABLE agent_memories_2024_q1 PARTITION OF agent_memories
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE agent_memories_2024_q2 PARTITION OF agent_memories
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
CREATE TABLE agent_memories_2024_q3 PARTITION OF agent_memories
    FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');
CREATE TABLE agent_memories_2024_q4 PARTITION OF agent_memories
    FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');

-- 按 agent_id 哈希分区（适用于多租户场景）
CREATE TABLE agent_memories_agent_a PARTITION OF agent_memories
    FOR VALUES IN ('agent-alpha', 'agent-beta');
CREATE TABLE agent_memories_agent_b PARTITION OF agent_memories
    FOR VALUES IN ('agent-gamma', 'agent-delta');
```

**分区的好处**：

- **查询剪枝（Partition Pruning）**：查询 `WHERE created_at >= '2024-06-01'` 时只扫描 Q3 分区
- **高效数据淘汰**：超过保留期限的分区直接 DROP，比 DELETE 快上千倍（无需 VACUUM）
- **并行查询**：每个分区可以独立扫描
- **分级存储**：可以将历史分区移到廉价存储（如 HDD 或对象存储）

**典型保留策略**：

```text
记忆类型             保留期限      分区粒度      淘汰方式
─────────────────────────────────────────────────────────
会话级记忆             24 小时      按天       DROP 过天分区
短期记忆             7-30 天        按周       DROP 过周分区
长期用户偏好         永久保留        按月       保留全部分区
性能基线数据         90 天          按天       聚合后 DELETE
```

## 产生的结果

### 1. ACID 保证的持久化

数据库记忆最直接的结果是 Agent 获得了**企业级的持久化保证**：

- 系统崩溃（断电、OOM、内核 Panic）后，已提交的记忆不会丢失
- 并发写入时，隔离级别确保 Agent 不会读到"一半写入"的脏数据
- 约束检查（UNIQUE、FOREIGN KEY、CHECK）防止 Agent 产生自相矛盾的记忆

```text
Agent 写入一次对话：
  ┌──────────────┐
  │ BEGIN;       │
  │ INSERT INTO  │──→ 服务器断电！
  │ sessions...  │
  │ INSERT INTO  │
  │ messages...  │
  │ COMMIT;      │
  └──────────────┘
       │
       ▼
  PostgreSQL WAL（预写日志）：事务已记录但未提交
  重启后：WAL 回放时发现事务不完整 → 自动回滚
  Agent 恢复后：看到的是"这次对话不存在"的一致状态
  而不是"有 session 但没消息"的断裂状态
```

### 2. 复杂的时序查询

Agent 可以对过去的记忆执行丰富的时序分析：

```sql
-- "这个用户对哪个话题最感兴趣？"（按关键词频次排序）
SELECT
    substring(content FROM '(#\w+)') AS topic,
    COUNT(*) AS mention_count,
    MIN(created_at) AS first_seen,
    MAX(created_at) AS last_seen
FROM agent_memories
WHERE agent_id = 'assistant-v1'
  AND session_id IN (
      SELECT session_id FROM sessions WHERE user_id = 'user-42'
  )
GROUP BY topic
ORDER BY mention_count DESC;

-- "我是不是回复得越来越慢了？"（Agent 自省）
SELECT
    DATE_TRUNC('day', created_at) AS day,
    AVG(
        EXTRACT(EPOCH FROM (responded_at - created_at))
    ) AS avg_response_time_seconds
FROM agent_memories
WHERE agent_id = 'assistant-v1'
GROUP BY day
ORDER BY day;
```

### 3. 跨会话记忆

关系数据库使 Agent 能够实现真正的跨会话记忆——今天的对话可以利用昨天甚至上个月的上下文：

```sql
-- 跨会话提取用户偏好的隐式模式
SELECT
    user_message,
    agent_response,
    created_at
FROM agent_memories
WHERE agent_id = 'assistant-v1'
  AND user_id = 'user-42'
  AND event_type = 'message'
  AND (
      user_message ILIKE '%喜欢%'
      OR user_message ILIKE '%偏好%'
      OR user_message ILIKE '%不喜欢%'
  )
ORDER BY created_at DESC
LIMIT 20;
```

### 4. 结构化分析和报表

数据库对聚合分析的原生支持使 Agent 具备自我监控能力：

```sql
-- Agent 使用情况统计
SELECT
    DATE_TRUNC('hour', created_at) AS hour,
    COUNT(*) AS total_interactions,
    COUNT(DISTINCT user_id) AS unique_users,
    AVG(satisfaction) AS avg_satisfaction,
    COUNT(*) FILTER (WHERE tools_used IS NOT NULL) AS tool_usage_count
FROM agent_memories
WHERE created_at >= NOW() - INTERVAL '24 hours'
GROUP BY hour
ORDER BY hour;
```

## 最大挑战

### Schema 迁移与演进的 Agent 知识

数据库记忆面临的最大挑战是：**Agent 的知识会持续演化，但数据库 Schema 的变更成本极高**。

**问题描述**：

```text
时间线：

T0：Agent 启动，使用简单 Schema
    CREATE TABLE memories (
        id SERIAL PRIMARY KEY,
        content TEXT,
        created_at TIMESTAMP
    );

T1：Agent 学会"关注用户情绪"——需要存储情感评分
    → ALTER TABLE memories ADD COLUMN sentiment_score FLOAT;

T2：Agent 学会"记忆关联"——需要建立记忆之间的链接
    → CREATE TABLE memory_links (
          source_id INT REFERENCES memories(id),
          target_id INT REFERENCES memories(id),
          relation_type VARCHAR(32)
      );

T3：Agent 学会"长期和短期记忆区分"——需要添加优先级字段
    → ALTER TABLE memories ADD COLUMN priority INT DEFAULT 0;
    → CREATE INDEX idx_priority ON memories (priority);

T4：Agent 的学习路径再次延伸...
    → 第 10 次 Schema 变更时，生产环境已经积累了 5TB 数据
    → ALTER TABLE 可能锁表数小时
    → 回滚困难，数据迁移风险大
```

**核心困境**：

```
固定 Schema vs. 演化知识

固定 Schema 的优势：         演化知识的优势：
  • 查询性能好                • Agent 能力不受限
  • 数据完整性高               • 可以适应新场景
  • 应用代码简单               • 响应环境变化
  • DBA 喜欢                  • 持续改进

冲突本质：Agent 作为"持续学习的系统"，其知识结构不断演化，
而关系数据库的 Schema 设计哲学是"先设计，后使用"。
```

### 应对策略

#### 策略一：JSONB 柔性列

使用 PostgreSQL 的 JSONB 类型作为"逃生舱"，将非核心、易变的信息存入 JSONB 字段：

```sql
CREATE TABLE agent_memories (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        VARCHAR(64) NOT NULL,
    memory_type     VARCHAR(32) NOT NULL,    -- 结构化类型标识
    core_fields     JSONB NOT NULL,          -- 必须有的核心字段
    extra_fields    JSONB NOT NULL DEFAULT '{}',  -- 灵活扩展的额外字段
    embedding       VECTOR(384),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 通过 GIN 索引支持 JSONB 内部查询
CREATE INDEX idx_extra_fields ON agent_memories USING GIN (extra_fields);

-- 查询示例
SELECT * FROM agent_memories
WHERE agent_id = 'assistant-v1'
  AND memory_type = 'user_interaction'
  AND extra_fields @> '{"sentiment": "positive"}';
```

**优点**：无需 ALTER TABLE 即可扩展记忆字段
**缺点**：JSONB 查询不能利用传统 B-tree 索引，GIN 索引的写入代价较高

#### 策略二：增量迁移（Online Migration）

对于需要严格 Schema 的场景，采用零停机迁移策略：

```python
import sqlalchemy as sa
from sqlalchemy import text
from typing import Generator


class OnlineSchemaMigrator:
    """
    零停机 Schema 迁移器 —— 分阶段演进数据库 Schema，
    不阻塞 Agent 的读写操作。
    """

    def __init__(self, engine: sa.Engine):
        self.engine = engine

    def expand_table(
        self,
        table_name: str,
        new_columns: list[sa.Column],
        batch_size: int = 10000,
    ):
        """
        分阶段给表添加新列，避免长时间锁表。

        阶段 1：添加列为 NULLABLE（轻量操作，不锁表）
        阶段 2：分批回填数据
        阶段 3：添加 NOT NULL 约束或默认值
        """
        metadata = sa.MetaData()
        table = sa.Table(table_name, metadata, autoload_with=self.engine)

        # 阶段 1: 添加可空列
        for col in new_columns:
            col.nullable = True
            with self.engine.connect() as conn:
                conn.execute(
                    text(f"ALTER TABLE {table_name} ADD COLUMN IF NOT EXISTS {col.name} {col.type.compile(self.engine.dialect)}")
                )
                conn.commit()

        # 阶段 2: 分批回填数据
        with self.engine.connect() as conn:
            total = conn.execute(
                text(f"SELECT COUNT(*) FROM {table_name} WHERE {' IS NULL AND '.join(c.name for c in new_columns)} {' IS NULL' if len(new_columns) > 1 else ''}")
            ).scalar() or 0

            processed = 0
            while processed < total:
                conn.execute(
                    text(f"""
                        UPDATE {table_name}
                        SET {', '.join(f'{c.name} = DEFAULT' for c in new_columns)}
                        WHERE ctid IN (
                            SELECT ctid FROM {table_name}
                            WHERE {' IS NULL AND '.join(c.name for c in new_columns)} {' IS NULL' if len(new_columns) > 1 else ''}
                            LIMIT {batch_size}
                        )
                    """)
                )
                conn.commit()
                processed += batch_size

        # 阶段 3: 对新列添加 NOT NULL（可选，锁表但时间短）
        # 注意：需确保数据已全部回填
        for col in new_columns:
            if col.nullable is False:
                with self.engine.connect() as conn:
                    conn.execute(
                        text(f"ALTER TABLE {table_name} ALTER COLUMN {col.name} SET NOT NULL")
                    )
                    conn.commit()
```

#### 策略三：版本化事件存储 + 投影

最彻底的解决方案是采用事件溯源：存储原始事件（永远不修改），通过投影（Projection）生成当前状态的 Schema：

```text
事件流（不可变的源）：
  Event{type: "memory_created", data: {content: "..."}}
  Event{type: "memory_tagged", data: {tag: "important"}}
  Event{type: "memory_priority_set", data: {priority: 5}}

投影（物化视图）：
  Materialized View: current_memory_state
  ├─ id
  ├─ content          ← 来自 memory_created 事件
  ├─ tags[]           ← 来自多个 memory_tagged 事件的聚合
  ├─ priority         ← 来自 memory_priority_set 事件的最新值
  └─ updated_at       ← 最后事件的时间

当 Agent 演化出新需求时：
  1. 定义新的事件类型（无需迁移旧数据）
  2. 编写新的投影逻辑
  3. 重放历史事件生成新的物化视图
  4. 旧视图和新视图并行存在，直到 Agent 完全迁移
```

这种模式在灵活性上最优，但实现复杂度最高，需要事件总线和投影引擎支持。

## 能力边界

### 关系数据库记忆能做什么

| 能力 | 说明 | 示例 |
|------|------|------|
| **精确事实检索** | 按精确条件查找记忆 | "用户 ID 为 42 的邮箱是什么？" |
| **范围查询** | 按时间、数值范围过滤 | "过去 7 天的对话" |
| **聚合分析** | 计数、求和、平均、分组 | "今天处理了多少请求？" |
| **排序分页** | 按任何字段排序、分页 | "最近 10 条重要的记忆" |
| **多表关联** | JOIN 多个实体 | "这个用户的订单列表" |
| **事务一致性** | 原子性的多步记忆操作 | "同时更新会话状态和记录消息" |
| **时间点恢复** | 恢复到任意历史时间点 | "恢复 10 分钟前的记忆状态" |
| **并发控制** | 多 Agent 实例安全并发 | "10 个 Agent 同时写入不会冲突" |
| **数据完整性** | 外键约束、唯一约束 | "不会出现悬挂引用" |

### 关系数据库记忆不能做什么

| 限制 | 说明 | 后果 |
|------|------|------|
| **无语义理解** | 不知道"苹果"是一种水果还是一家公司 | 无法区分同名不同义的实体 |
| **无法处理模糊查询** | "那个蓝色的东西"无法对应 SQL 条件 | Agent 必须精确描述查询条件 |
| **Schema 刚性** | 表结构预先定义，不可动态扩展 | Agent 学到新概念时需要迁移数据 |
| **不支持隐式关联** | 不会自动发现"用户 A 和 B 都买了同一本书" | 需要显式编写 JOIN 或关联表 |
| **全文检索有限** | 只有关键词匹配，无同义词、概念泛化 | "笔记本电脑"搜不到"laptop"（无配置时） |
| **无知识推理** | 不会推导"A 是 B 的上级，B 是 C 的上级 → A 是 C 的上级" | 多跳关系需要应用层实现 |
| **无主动遗忘** | 数据只增不减，除非显式 DELETE | 需要单独的归档/清理策略 |
| **存储非结构化困难** | 长文本、图片、音频不适合直接存储 | 通常用 TEXT 或外部存储方案 |
| **扩展成本非线性** | 随着数据量增长，查询性能可能急剧下降 | 需要分区、分片、读写分离等复杂架构 |

### 关键限制的深入分析

#### 语义盲区

```text
用户说："帮我查查那个小李负责的项目"

数据库里有：
  ┌────────────┬──────────┐
  │  project   │  owner   │
  ├────────────┼──────────┤
  │  智慧城市   │  李明    │
  │  金融平台   │  李芳    │
  │  医疗AI     │  李伟    │
  │  物联网     │  张强    │
  └────────────┴──────────┘

Agent 的 SQL 可能生成：
  SELECT * FROM projects WHERE owner LIKE '%李%';
  → 返回 3 行，Agent 需要额外询问"哪个小李？"

理想的语义理解：
  "小李" → 昵称推断 → "李明"（因为"明"读作 ming，不常用于昵称"小X"模式）
  → 但这需要外部常识或知识图谱支持
```

#### 模式刚性

```text
Agent 初始设计时只知道"记忆 = 文本 + 时间戳"

表结构：
  CREATE TABLE memories (id, content, created_at);

3 个月后，Agent 学会了：
  - 记忆需要分类（类型）
  - 记忆需要优先级
  - 记忆需要关联引用
  - 记忆需要情感标签

但表结构不支持以上任何一种新属性。
每个新属性都需要：ALTER TABLE + 数据迁移 + 应用代码更新。
```

## 对比

### 关系型数据库 vs 向量数据库 vs 知识图谱 vs 文档存储

| 对比维度 | 关系型数据库（PostgreSQL） | 向量数据库（Pinecone/Qdrant） | 知识图谱（Neo4j） | 文档存储（MongoDB） |
|----------|--------------------------|------------------------------|-------------------|-------------------|
| **数据模型** | 关系表（行×列） | 向量 + metadata | 节点 + 边（有向图） | 无 Schema 的 JSON 文档 |
| **查询方式** | SQL（精确匹配、JOIN、聚合） | 向量相似度（ANN 搜索） | Cypher/Gremlin（图遍历） | 类 JSON 查询（find、aggregate） |
| **查询精度** | 极高（精确值匹配） | 近似（ANN，99%+ recall） | 高（精确图匹配） | 高（精确字段匹配） |
| **语义搜索** | 弱（需扩展 pgvector） | 原生支持（核心能力） | 弱（需结合外部 NLP） | 弱（需 Atlas Search） |
| **事务支持** | 完整 ACID | 有限或无 | 完整 ACID | 文档级原子性 |
| **一致性模型** | 强一致性（可配置） | 最终一致性 | 强一致性 | 可调一致性 |
| **Schema 约束** | 严格（ALTER TABLE 成本高） | 弹性（metadata 可随意加） | 灵活（动态属性） | 无 Schema（同一集合内文档可异构） |
| **连接/关联查询** | JOIN（成熟优化器） | 不支持 | 图遍历（高效多跳） | $lookup（性能低于 JOIN） |
| **全文检索** | tsvector（内置，功能强大） | 依赖元数据过滤 | 有限 | 文本索引（不错） |
| **向量搜索** | pgvector 扩展（功能完整） | 原生（核心优势） | 不支持 | Atlas Vector Search |
| **水平扩展** | 复杂（分片/分库） | 原生分布式 | 企业版支持 | 原生分片 |
| **运维复杂度** | 中（需 DBA 经验） | 低（托管服务） | 中（需图数据库经验） | 低-中 |
| **查询延迟（精确）** | 1-10ms | N/A（不支持精确查询） | 10-100ms（图遍历） | 1-10ms |
| **查询延迟（语义）** | 10-50ms（pgvector，100K 条） | 5-10ms | N/A | N/A |
| **典型用例** | 结构化记忆、交易日志、配置 | RAG 检索、语义记忆 | 实体关系推理、分析 | 对话历史、非结构化日志 |
| **适用 Agent 类型** | 需要精确事实和事务保证的 Agent | 需要语义理解和开放域问答的 Agent | 需要多跳推理的 Agent | 需要灵活 Schema 的 Agent |
| **学习曲线** | 低（SQL 普及度高） | 中（向量概念 + API） | 高（图模型 + Cypher） | 低（类 JSON） |

### 当数据库记忆胜出的场景

```
┌─────────────────────────────────────────────────────────┐
│              数据库记忆是最佳选择的场景                     │
├─────────────────────────────────────────────────────────┤
│  1. 需要 ACID 事务保证。Agent 不能接受"写一半"的状态。     │
│  2. 查询以精确匹配为主（按 ID、时间、状态过滤）。           │
│  3. 需要复杂的聚合分析（统计、报表、趋势）。                │
│  4. 数据之间的关系是结构化的（用户↔订单↔商品）。            │
│  5. 需要成熟的运维工具链（备份、恢复、监控、审计）。         │
│  6. 团队已有 SQL 和关系数据库的使用经验。                  │
└─────────────────────────────────────────────────────────┘
```

### 当其他方案胜出的场景

```
┌─────────────────────────────────────────────────────────┐
│              向量数据库更适合的场景                         │
├─────────────────────────────────────────────────────────┤
│  1. 核心需求是语义相似度搜索（"找一篇和这个类似的文章"）。    │
│  2. Agent 需要处理大量非结构化文本。                       │
│  3. 对查询的精确性要求不严格（近似值即可）。                │
│  4. 需要极致的语义搜索延迟（<10ms）。                      │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              知识图谱更适合的场景                          │
├─────────────────────────────────────────────────────────┤
│  1. 核心需求是多跳推理（"A 认识 B，B 认识 C → A 了解 C"）。│
│  2. 实体之间的关系数量多且复杂。                           │
│  3. 需要路径分析和网络分析。                              │
│  4. 实体类型动态增加，无法预定义 Schema。                  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              文档存储更适合的场景                          │
├─────────────────────────────────────────────────────────┤
│  1. 数据结构频繁变化，无法预定义固定的 Schema。            │
│  2. 存储的是完整的文档/对话记录而非结构化字段。             │
│  3. 需要快速的原型开发，不想受 Schema 约束。               │
│  4. 数据量大但关系简单（主要是独立文档的集合）。            │
└─────────────────────────────────────────────────────────┘
```

## 工程优化

### 1. pgvector 混合搜索

pgvector 是 PostgreSQL 的向量相似度搜索扩展，使关系数据库能够执行近似最近邻（ANN）搜索，同时保留完整的 SQL 能力。

**安装与配置**：

```sql
-- 安装扩展
CREATE EXTENSION vector;

-- 创建带有向量列的记忆表
CREATE TABLE agent_memories (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        VARCHAR(64) NOT NULL,
    memory_type     VARCHAR(32) NOT NULL DEFAULT 'general',
    content         TEXT NOT NULL,
    content_tsv     TSVECTOR GENERATED ALWAYS AS (
                        to_tsvector('simple', content)
                    ) STORED,
    embedding       VECTOR(1536),  -- OpenAI text-embedding-3-small 维度
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 创建索引
-- IVFFlat 索引（适合 100K-1M 条）
CREATE INDEX idx_memories_ivfflat ON agent_memories
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);  -- lists ≈ sqrt(row_count)

-- HNSW 索引（适合 1M+ 条，精度更高）
-- CREATE INDEX idx_memories_hnsw ON agent_memories
--     USING hnsw (embedding vector_cosine_ops);
```

**混合查询：SQL 过滤 + 向量排序**：

```sql
-- 场景："找 agent_id='assistant-v1' 最近 7 天内，和某段文本语义相似的记忆"
SELECT
    id,
    content,
    metadata,
    created_at,
    1 - (embedding <=> :query_embedding) AS similarity
FROM agent_memories
WHERE agent_id = :agent_id
  AND memory_type = 'conversation'
  AND created_at >= NOW() - INTERVAL '7 days'
  -- 可选：全文检索预过滤
  AND (content_tsv @@ plainto_tsquery('simple', :keyword) OR :keyword = '')
ORDER BY embedding <=> :query_embedding
LIMIT 20;
```

**性能调优**：

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `lists`（IVFFlat） | 索引列表数，越大精度越高但构建越慢 | `sqrt(n_rows)` |
| `probes`（IVFFlat） | 搜索时的探针数，越大召回率越高但越慢 | `lists / 10` |
| `ef_construction`（HNSW） | 构建时的动态列表大小 | 128-256 |
| `ef_search`（HNSW） | 搜索时的动态列表大小 | 40-100 |
| `maintenance_work_mem` | 索引构建时的内存 | 至少 1GB |

```sql
-- 调优示例：设置探针数
SET ivfflat.probes = 20;

-- 或针对特定连接设置
-- 更高的 probes 值提高召回率，降低速度
```

---

### 2. 连接池（PgBouncer）

对于多 Agent 实例部署，数据库连接是稀缺资源。PgBouncer 是 PostgreSQL 的轻量级连接池中间件。

**架构**：

```text
无连接池：
  [Agent-1] ──→ PostgreSQL
  [Agent-2] ──→ PostgreSQL
  [Agent-3] ──→ PostgreSQL
  [Agent-4] ──→ PostgreSQL
  ...
  [Agent-N] ──→ PostgreSQL
  → N 个 Agent 需要 N 个连接
  → PostgreSQL 的 max_connections 通常只有 100-500

有 PgBouncer：
  [Agent-1] ──┐
  [Agent-2] ──┤
  [Agent-3] ──┼──→ [PgBouncer] ──→ PostgreSQL (池化连接)
  [Agent-4] ──┤                │
  ...          │               └─ 通常只有 10-50 个真实连接
  [Agent-N] ──┘
  → Agent 数量与数据库连接数解耦
  → PgBouncer 可以处理数千个客户端连接
```

**配置示例**（`pgbouncer.ini`）：

```ini
[databases]
agent_memory = host=127.0.0.1 port=5432 dbname=agent_memory

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432

; 连接池模式
; session:  每个连接绑定到整个会话（默认，兼容性最好）
; transaction: 每个事务后归还连接到池（推荐）
; statement: 每个查询后归还（最高效，但不支持事务）
pool_mode = transaction

; 每个用户/数据库组合的最大连接数
default_pool_size = 25
max_client_conn = 500

; 连接生命周期
server_idle_timeout = 600      ; 10 分钟
client_idle_timeout = 1800     ; 30 分钟
server_lifetime = 3600         ; 1 小时

; 查询超时
query_timeout = 30             ; 单个查询最多 30 秒
query_wait_timeout = 10        ; 等待可用连接的最大时间
```

**Python 集成**：

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# 使用 PgBouncer 的事务模式连接
ASYNC_DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:6432/agent_memory"

engine = create_async_engine(
    ASYNC_DATABASE_URL,
    pool_size=10,              # 保持少量连接
    max_overflow=5,
    pool_pre_ping=True,        # 使用前检查连接健康
    pool_recycle=1800,         # 30 分钟回收
    echo=False,
)

AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

---

### 3. 读写分离

对于读多写少的 Agent 记忆场景（典型比例：80% 读、20% 写），读写分离架构可以显著提升吞吐量。

**架构**：

```text
                 ┌──────────────┐
                 │   Agent 实例   │
                 └──────┬───────┘
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
      ┌──────────────┐   ┌──────────────┐
      │  写操作路由    │   │  读操作路由    │
      │  INSERT/UPDATE │   │  SELECT      │
      │  DELETE       │   │              │
      └──────┬───────┘   └──────┬───────┘
             │                  │
             ▼                  ▼
      ┌──────────────┐   ┌──────────────┐
      │  主库 (Primary)│   │  只读副本      │
      │  读写均可     │───│  (Read Replica)│
      │              │   │  只读         │
      └──────────────┘   └──────────────┘
             │
             └─── 流复制 (Streaming Replication) ──→ [更多只读副本]
```

**实现**：

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import Session


class DatabaseRouter:
    """
    读写分离路由器 —— 将读操作路由到只读副本，写操作路由到主库。
    """

    def __init__(self, master_url: str, replica_urls: list[str]):
        self.master = create_engine(
            master_url,
            pool_size=10,
            max_overflow=20,
        )
        self.replicas = [
            create_engine(url, pool_size=20, max_overflow=40)
            for url in replica_urls
        ]
        self._replica_index = 0

    def get_writer(self) -> Session:
        """获取写会话（路由到主库）。"""
        return Session(self.master)

    def get_reader(self) -> Session:
        """获取读会话（轮询到某个只读副本）。"""
        engine = self.replicas[self._replica_index % len(self.replicas)]
        self._replica_index += 1
        return Session(engine)


# 使用示例
router = DatabaseRouter(
    master_url="postgresql://user:pass@master.internal:5432/agent_memory",
    replica_urls=[
        "postgresql://user:pass@replica-1.internal:5432/agent_memory",
        "postgresql://user:pass@replica-2.internal:5432/agent_memory",
    ],
)

# Agent 的写操作
def record_memory(agent_id: str, content: str):
    with router.get_writer() as session:
        session.execute(
            "INSERT INTO agent_memories (agent_id, content) VALUES (:aid, :c)",
            {"aid": agent_id, "c": content},
        )
        session.commit()

# Agent 的读操作
def recall_memories(agent_id: str):
    with router.get_reader() as session:
        return session.execute(
            "SELECT * FROM agent_memories WHERE agent_id = :aid ORDER BY created_at DESC LIMIT 50",
            {"aid": agent_id},
        ).fetchall()
```

---

### 4. 时间/Agent 分区

对于多 Agent 多租户场景，数据量增长极快，分区是维持性能的关键手段。

**综合分区策略**：

```sql
-- 先按 agent_id 列表分区，再按月范围子分区
CREATE TABLE agent_memories (
    id          BIGSERIAL,
    agent_id    VARCHAR(64) NOT NULL,
    content     TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY LIST (agent_id);

-- 每个 Agent 一个分区
CREATE TABLE agent_memories_a1 PARTITION OF agent_memories
    FOR VALUES IN ('agent-alpha')
    PARTITION BY RANGE (created_at);

CREATE TABLE agent_memories_a1_2024_q1 PARTITION OF agent_memories_a1
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE agent_memories_a1_2024_q2 PARTITION OF agent_memories_a1
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

**自动创建分区**：

```python
from datetime import datetime, timedelta
from sqlalchemy import text


class PartitionManager:
    """
    分区管理器 —— 自动创建和维护时间分区。
    """

    def __init__(self, engine, table_name: str, interval_days: int = 90):
        self.engine = engine
        self.table_name = table_name
        self.interval_days = interval_days

    def ensure_partition(self, for_date: datetime | None = None):
        """确保目标日期的分区存在。"""
        if for_date is None:
            for_date = datetime.utcnow()

        partition_name = self._partition_name(for_date)
        start_date = self._partition_start(for_date)
        end_date = start_date + timedelta(days=self.interval_days)

        with self.engine.connect() as conn:
            # 检查分区是否存在
            exists = conn.execute(
                text(f"""
                    SELECT EXISTS (
                        SELECT 1 FROM pg_class WHERE relname = :pname
                    )
                """),
                {"pname": partition_name},
            ).scalar()

            if not exists:
                conn.execute(text(f"""
                    CREATE TABLE {partition_name} PARTITION OF {self.table_name}
                    FOR VALUES FROM (:start) TO (:end)
                """), {
                    "start": start_date.isoformat(),
                    "end": end_date.isoformat(),
                })
                conn.commit()

                # 可选：在分区上创建索引
                conn.execute(text(f"""
                    CREATE INDEX idx_{partition_name}_created
                    ON {partition_name} (created_at DESC)
                """))
                conn.commit()

    def drop_old_partitions(self, before_date: datetime):
        """删除早于指定日期的分区。"""
        with self.engine.connect() as conn:
            partitions = conn.execute(text(f"""
                SELECT inhrelid::regclass::text AS partition_name,
                       pg_get_expr(relpartbound, inhrelid) AS boundary
                FROM pg_inherits
                JOIN pg_class ON pg_class.oid = inhrelid
                WHERE inhparent = :table_name::regclass
            """), {"table_name": self.table_name}).fetchall()

            for partition_name, _ in partitions:
                # 解析分区边界，判断是否早于 before_date
                # 实际使用中需要更健壮的解析逻辑
                if self._should_drop(partition_name, before_date):
                    conn.execute(text(f"DROP TABLE {partition_name}"))
                    conn.commit()

    def _partition_name(self, dt: datetime) -> str:
        quarter = (dt.month - 1) // 3 + 1
        return f"{self.table_name}_{dt.year}_q{quarter}"

    def _partition_start(self, dt: datetime) -> datetime:
        quarter = (dt.month - 1) // 3
        return datetime(dt.year, quarter * 3 + 1, 1)

    def _should_drop(self, partition_name: str, before: datetime) -> bool:
        # 简化实现：分区名包含日期信息
        try:
            parts = partition_name.split("_")
            year = int(parts[-2])
            q = int(parts[-1][1])
            partition_start = datetime(year, (q - 1) * 3 + 1, 1)
            return partition_start < before
        except (ValueError, IndexError):
            return False
```

---

### 5. 预编译语句

对于 Agent 反复执行的同一类查询（如"按 session_id 检索对话"），预编译语句（Prepared Statement）可以消除 SQL 解析和查询计划生成的开销。

**原理**：

```text
每次普通查询：
  [Agent] ──→ "SELECT * FROM memories WHERE id = 1" ──→ SQL 解析 ──→ 计划生成 ──→ 执行
  [Agent] ──→ "SELECT * FROM memories WHERE id = 2" ──→ SQL 解析 ──→ 计划生成 ──→ 执行
  [Agent] ──→ "SELECT * FROM memories WHERE id = 3" ──→ SQL 解析 ──→ 计划生成 ──→ 执行

使用预编译语句：
  [Agent] ──→ PREPARE: "SELECT * FROM memories WHERE id = $1" ──→ 解析 + 计划（一次）
  [Agent] ──→ EXECUTE: $1 = 1                            ──→ 只执行
  [Agent] ──→ EXECUTE: $1 = 2                            ──→ 只执行
  [Agent] ──→ EXECUTE: $1 = 3                            ──→ 只执行
```

**ORM 层集成**：

```python
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession


class PreparedMemoryQueries:
    """
    预编译记忆查询 —— 重用执行计划，减少 SQL 解析开销。
    """

    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_memories_by_session(self, session_id: str, limit: int = 100):
        """按 session_id 检索记忆（高频查询）。"""
        stmt = text("""
            SELECT id, content, memory_type, metadata, created_at
            FROM agent_memories
            WHERE session_id = :session_id
            ORDER BY created_at ASC
            LIMIT :limit
        """ )
        result = await self.session.execute(
            stmt,
            {"session_id": session_id, "limit": limit},
        )
        return result.fetchall()

    async def count_memories_today(self, agent_id: str):
        """统计今天的记忆数量（监控查询）。"""
        stmt = text("""
            SELECT COUNT(*) FROM agent_memories
            WHERE agent_id = :agent_id
              AND created_at >= CURRENT_DATE
        """)
        result = await self.session.execute(stmt, {"agent_id": agent_id})
        return result.scalar()

    async def get_recent_by_type(self, agent_id: str, memory_type: str, limit: int = 20):
        """按类型检索最近的记忆（高频查询）。"""
        stmt = text("""
            SELECT id, content, metadata, created_at
            FROM agent_memories
            WHERE agent_id = :agent_id
              AND memory_type = :memory_type
            ORDER BY created_at DESC
            LIMIT :limit
        """)
        result = await self.session.execute(stmt, {
            "agent_id": agent_id,
            "memory_type": memory_type,
            "limit": limit,
        })
        return result.fetchall()
```

注意：SQLAlchemy 2.x 自动支持 Prepared Statement 缓存（通过 `Connection.execution_options(compiled_cache={})`），一般无需手动管理。

---

### 6. WAL 复制与高可用

PostgreSQL 的 WAL（Write-Ahead Log）是实现高可用和灾难恢复的基石。

**WAL 在 Agent 记忆场景中的作用**：

```text
WAL 复制流：

  [主库]                        [备库]
  ┌──────────────┐             ┌──────────────┐
  │ Agent 写入    │             │              │
  │ 记忆数据      │             │              │
  │      ↓        │             │              │
  │ WAL 写入磁盘  │──WAL 传输──→│ WAL 接收     │
  │      ↓        │             │      ↓       │
  │ 数据写入数据页 │             │ WAL 回放     │
  │      ↓        │             │      ↓       │
  │ 返回成功给    │             │ 数据页更新    │
  │ Agent         │             │              │
  └──────────────┘             └──────────────┘

Agent 感知到的主库故障转移：
  1. 主库宕机
  2. 心跳检测超时
  3. 运维工具（Patroni / repmgr）提升备库为主库
  4. Agent 连接自动切换到新主库
  5. Agent 无感知（或仅重试当前操作）
```

**同步 vs 异步复制权衡**：

| 复制模式 | 数据安全性 | 写入延迟 | 适用场景 |
|----------|-----------|---------|---------|
| 同步复制 | 极高（零数据丢失） | 较高（等待备库确认） | 金融级记忆、不可丢失的关键数据 |
| 异步复制 | 高（可能丢失少量 WAL） | 低（不等待备库） | 一般 Agent 记忆、可容忍秒级丢失 |

## 代码示例

### 示例 1：SQLAlchemy 模型定义

一个完整的、生产可用的 Agent 记忆存储模型：

```python
"""
agent_memory_models.py
Agent 记忆存储的 SQLAlchemy ORM 模型定义。
"""
import enum
import uuid
from datetime import datetime, timezone
from typing import Any

from sqlalchemy import (
    Column, Integer, BigInteger, String, Text, Float,
    DateTime, ForeignKey, Enum, Index, UniqueConstraint,
    JSON, TypeDecorator,
)
from sqlalchemy.dialects.postgresql import UUID, JSONB, TSVECTOR, ARRAY
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func

Base = declarative_base()


def utcnow():
    return datetime.now(timezone.utc)


# ─── 自定义类型：确保 UUID 兼容不同数据库 ────────────────
class GUID(TypeDecorator):
    """平台无关的 GUID 类型。"""
    impl = String(36)
    cache_ok = True

    def process_bind_param(self, value, dialect):
        if value is None:
            return None
        return str(value)

    def process_result_value(self, value, dialect):
        if value is None:
            return None
        return uuid.UUID(value) if isinstance(value, str) else value


# ─── 枚举 ──────────────────────────────────────────────
class MemoryType(str, enum.Enum):
    """记忆类型枚举。"""
    CONVERSATION = "conversation"       # 对话记录
    OBSERVATION = "observation"         # 观察结论
    DECISION = "decision"               # 决策记录
    USER_PREFERENCE = "user_preference" # 用户偏好
    TOOL_RESULT = "tool_result"         # 工具调用结果
    SYSTEM_NOTE = "system_note"         # 系统笔记


class MemoryImportance(str, enum.Enum):
    """记忆重要性等级。"""
    TRANSIENT = "transient"   # 瞬时（可随时遗忘）
    NORMAL = "normal"         # 普通
    IMPORTANT = "important"   # 重要
    CRITICAL = "critical"     # 关键（永不遗忘）


# ─── 核心表 ────────────────────────────────────────────

class AgentSession(Base):
    """Agent 会话表 —— 管理 Agent 的运行会话。"""
    __tablename__ = "agent_sessions"

    id = Column(GUID, primary_key=True, default=uuid.uuid4)
    agent_id = Column(String(128), nullable=False, index=True)
    session_type = Column(String(32), default="interactive")
    status = Column(String(16), default="active")  # active | closed | archived
    metadata_ = Column("metadata", JSONB, default=dict)
    started_at = Column(DateTime(timezone=True), server_default=func.now())
    ended_at = Column(DateTime(timezone=True), nullable=True)
    token_count = Column(BigInteger, default=0)  # 会话消耗的 token 数

    memories = relationship("AgentMemory", back_populates="session")

    __table_args__ = (
        Index("idx_session_agent_status", "agent_id", "status"),
        Index("idx_session_started", "started_at"),
    )


class AgentMemory(Base):
    """
    Agent 记忆主表 —— 存储所有类型的 Agent 记忆。

    设计原则：
    - 核心字段固定（便于精确查询）
    - 扩展字段使用 JSONB（保持 Schema 弹性）
    - 支持文本全文检索和向量语义检索
    """
    __tablename__ = "agent_memories"

    # ── 主键与标识 ──
    id = Column(BigInteger, primary_key=True, autoincrement=True)
    memory_id = Column(GUID, default=uuid.uuid4, unique=True, nullable=False)
    agent_id = Column(String(128), nullable=False, index=True)

    # ── 会话关联 ──
    session_id = Column(GUID, ForeignKey("agent_sessions.id", ondelete="SET NULL"),
                        nullable=True, index=True)
    session = relationship("AgentSession", back_populates="memories")

    # ── 记忆分类 ──
    memory_type = Column(Enum(MemoryType), nullable=False, default=MemoryType.CONVERSATION)
    importance = Column(Enum(MemoryImportance), nullable=False, default=MemoryImportance.NORMAL)

    # ── 记忆内容 ──
    summary = Column(String(500), nullable=True)          # 简短摘要（用于快速浏览）
    content = Column(Text, nullable=False)                 # 完整的记忆内容
    content_tsv = Column(TSVECTOR, nullable=True)          # 全文检索向量（由数据库触发器维护）

    # ── 向量嵌入（pgvector） ──
    # 注意：VECTOR 类型需要 pgvector 扩展，开发和测试中可注释
    # embedding = Column(VECTOR(1536), nullable=True)

    # ── 元数据 ──
    source = Column(String(64), default="agent")           # 记忆来源 agent | user | system | tool
    tools_used = Column(JSONB, default=list)               # 关联的工具调用
    metadata_ = Column("metadata", JSONB, default=dict)    # 扩展元数据
    tags = Column(ARRAY(String(32)), default=list)         # 标签

    # ── 关联 ──
    parent_id = Column(BigInteger, ForeignKey("agent_memories.id"),
                       nullable=True)                      # 父记忆（用于记忆链）
    parent = relationship("AgentMemory", remote_side=[id], backref="children")

    # ── 时间戳 ──
    created_at = Column(DateTime(timezone=True), server_default=func.now(), index=True)
    updated_at = Column(DateTime(timezone=True), server_default=func.now(),
                        onupdate=func.now())
    expires_at = Column(DateTime(timezone=True), nullable=True)  # 过期时间（用于自动遗忘）

    # ── 索引 ──
    __table_args__ = (
        Index("idx_memory_agent_type_time", "agent_id", "memory_type", "created_at"),
        Index("idx_memory_importance", "agent_id", "importance", "created_at"),
        Index("idx_memory_expires", "expires_at"),
        # 全文检索索引（PostgreSQL 特有）
        # Index("idx_memory_fts", "content_tsv", postgresql_using="gin"),
        __table_args__ = {"schema": "public"}
    )

    def __repr__(self):
        return f"<AgentMemory(id={self.id}, type={self.memory_type}, agent={self.agent_id})>"


# ─── 辅助表 ────────────────────────────────────────────

class MemoryLink(Base):
    """
    记忆关联表 —— 在记忆之间建立显式链接。

    例如：工具调用结果链接到触发该调用的用户消息。
    """
    __tablename__ = "memory_links"

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    source_memory_id = Column(BigInteger, ForeignKey("agent_memories.id", ondelete="CASCADE"),
                              nullable=False)
    target_memory_id = Column(BigInteger, ForeignKey("agent_memories.id", ondelete="CASCADE"),
                              nullable=False)
    relation_type = Column(String(32), nullable=False)  # "causes", "references", "derives_from", "continues"
    weight = Column(Float, default=1.0)                  # 关联强度
    created_at = Column(DateTime(timezone=True), server_default=func.now())

    __table_args__ = (
        Index("idx_memory_link_source", "source_memory_id"),
        Index("idx_memory_link_target", "target_memory_id"),
        UniqueConstraint("source_memory_id", "target_memory_id", "relation_type",
                         name="uq_memory_link"),
    )


class MemoryTagStats(Base):
    """
    记忆标签统计表 —— 缓存标签的使用频率和最后使用时间。
    避免每次查询都 COUNT 标签。
    """
    __tablename__ = "memory_tag_stats"

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    agent_id = Column(String(128), nullable=False)
    tag = Column(String(32), nullable=False)
    count = Column(Integer, default=0)
    last_used = Column(DateTime(timezone=True), server_default=func.now())

    __table_args__ = (
        UniqueConstraint("agent_id", "tag", name="uq_agent_tag"),
        Index("idx_tag_stats_agent_count", "agent_id", "count"),
    )
```

---

### 示例 2：pgvector 混合搜索实现

```python
"""
hybrid_search.py
基于 PostgreSQL + pgvector 的混合搜索实现。
同时支持 SQL 精确过滤、全文检索和向量语义搜索。
"""
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Any, Protocol
import numpy as np

from sqlalchemy import text, bindparam
from sqlalchemy.ext.asyncio import AsyncSession


@dataclass
class SearchResult:
    """统一的搜索结果结构。"""
    id: int
    content: str
    memory_type: str
    summary: str | None
    score: float                     # 综合得分（0-1）
    text_score: float | None         # 文本匹配得分
    vector_score: float | None       # 向量相似度得分
    metadata: dict[str, Any] = field(default_factory=dict)
    created_at: Any = None


class EmbeddingProvider(Protocol):
    """嵌入提供者协议。"""
    async def embed(self, text: str) -> list[float]:
        ...


class HybridMemorySearch:
    """
    混合记忆搜索引擎。

    支持三种检索模式：
    1. SQL 精确过滤（按 agent_id、memory_type、时间范围）
    2. 全文检索（tsvector tsquery）
    3. 向量语义搜索（pgvector cosine distance）

    三种模式可以任意组合。
    """

    def __init__(
        self,
        session: AsyncSession,
        embedder: EmbeddingProvider,
    ):
        self.session = session
        self.embedder = embedder

    async def search(
        self,
        agent_id: str,
        query_text: str | None = None,
        memory_types: list[str] | None = None,
        importance_levels: list[str] | None = None,
        tags: list[str] | None = None,
        time_range: tuple[Any, Any] | None = None,       # (start, end)
        use_fts: bool = True,
        use_vector: bool = True,
        vector_weight: float = 0.5,
        limit: int = 20,
        offset: int = 0,
    ) -> list[SearchResult]:
        """
        混合搜索入口。

        Args:
            agent_id: Agent 标识
            query_text: 查询文本（用于全文检索和向量搜索）
            memory_types: 过滤的记忆类型列表
            importance_levels: 过滤的重要性等级列表
            tags: 过滤的标签列表
            time_range: (start_datetime, end_datetime)
            use_fts: 是否使用全文检索
            use_vector: 是否使用向量搜索
            vector_weight: 向量得分在综合得分中的权重
            limit: 返回结果数
            offset: 偏移量

        Returns:
            搜索结果列表，按综合得分降序排列
        """
        if not use_fts and not use_vector:
            return await self._exact_search(
                agent_id, query_text, memory_types, importance_levels,
                tags, time_range, limit, offset,
            )
        return await self._hybrid_search(
            agent_id, query_text, memory_types, importance_levels,
            tags, time_range, use_fts, use_vector, vector_weight,
            limit, offset,
        )

    async def _exact_search(
        self, agent_id: str, query_text: str | None,
        memory_types, importance_levels, tags, time_range,
        limit: int, offset: int,
    ) -> list[SearchResult]:
        """纯 SQL 精确搜索。"""
        conditions = ["agent_id = :agent_id"]
        params: dict[str, Any] = {"agent_id": agent_id}

        if query_text:
            conditions.append("content ILIKE :query_text")
            params["query_text"] = f"%{query_text}%"

        if memory_types:
            conditions.append("memory_type = ANY(:memory_types)")
            params["memory_types"] = memory_types

        if importance_levels:
            conditions.append("importance = ANY(:importance_levels)")
            params["importance_levels"] = importance_levels

        if tags:
            conditions.append("tags @> :tags")
            params["tags"] = tags

        if time_range:
            start, end = time_range
            if start:
                conditions.append("created_at >= :time_start")
                params["time_start"] = start
            if end:
                conditions.append("created_at <= :time_end")
                params["time_end"] = end

        where_clause = " AND ".join(conditions)
        sql = text(f"""
            SELECT id, content, memory_type, summary, metadata, created_at
            FROM agent_memories
            WHERE {where_clause}
            ORDER BY created_at DESC
            LIMIT :limit OFFSET :offset
        """)

        result = await self.session.execute(sql, {
            **params, "limit": limit, "offset": offset,
        })
        rows = result.fetchall()

        return [
            SearchResult(
                id=row.id,
                content=row.content,
                memory_type=row.memory_type,
                summary=row.summary,
                score=1.0,  # 精确匹配
                text_score=1.0,
                vector_score=None,
                metadata=dict(row.metadata),
                created_at=row.created_at,
            )
            for row in rows
        ]

    async def _hybrid_search(
        self, agent_id: str, query_text: str | None,
        memory_types, importance_levels, tags, time_range,
        use_fts: bool, use_vector: bool, vector_weight: float,
        limit: int, offset: int,
    ) -> list[SearchResult]:
        """混合搜索（全文 + 向量）。"""
        if not query_text:
            return await self._exact_search(
                agent_id, None, memory_types, importance_levels,
                tags, time_range, limit, offset,
            )

        params: dict[str, Any] = {
            "agent_id": agent_id,
            "limit": limit,
            "offset": offset,
        }

        # 构建 SELECT 和 ORDER BY 子句
        select_parts = [
            "id", "content", "memory_type", "summary", "metadata", "created_at"
        ]
        score_parts = []

        if use_fts:
            params["fts_query"] = text(
                "plainto_tsquery('simple', :fts_text)"
            ).bindparams(bindparam("fts_text", query_text))
            # 实际执行时使用字符串和函数调用
            select_parts.append(
                "ts_rank(coalesce(content_tsv, ''::tsvector), "
                "plainto_tsquery('simple', :fts_text)) AS text_score"
            )
            params["fts_text"] = query_text  # 实际参数
            score_parts.append("COALESCE(text_score, 0)")

        if use_vector:
            # 需要先获取查询文本的向量嵌入
            query_embedding = await self.embedder.embed(query_text)
            embedding_str = "[" + ",".join(str(v) for v in query_embedding) + "]"
            select_parts.append(
                "1 - (embedding <=> :query_vec::vector) AS vector_score"
            )
            params["query_vec"] = embedding_str
            score_parts.append("COALESCE(vector_score, 0)")

        # 综合得分
        if len(score_parts) == 2:
            select_parts.append(
                f"({score_parts[0]} * {1 - vector_weight} + "
                f"{score_parts[1]} * {vector_weight}) AS combined_score"
            )
        elif len(score_parts) == 1:
            select_parts.append(f"{score_parts[0]} AS combined_score")

        # 构建 WHERE 条件
        conditions = ["agent_id = :agent_id"]

        if memory_types:
            conditions.append("memory_type = ANY(:memory_types)")
            params["memory_types"] = memory_types

        if importance_levels:
            conditions.append("importance = ANY(:importance_levels)")
            params["importance_levels"] = importance_levels

        if tags:
            conditions.append("tags @> :tags")
            params["tags"] = tags

        if time_range:
            start, end = time_range
            if start:
                conditions.append("created_at >= :time_start")
                params["time_start"] = start
            if end:
                conditions.append("created_at <= :time_end")
                params["time_end"] = end

        where_clause = " AND ".join(conditions)

        sql_text = f"""
            SELECT {', '.join(select_parts)}
            FROM agent_memories
            WHERE {where_clause}
            ORDER BY combined_score DESC NULLS LAST
            LIMIT :limit OFFSET :offset
        """

        result = await self.session.execute(text(sql_text), params)
        rows = result.fetchall()

        results = []
        for row in rows:
            score = float(row.combined_score) if hasattr(row, 'combined_score') and row.combined_score is not None else 0.0
            text_score = float(row.text_score) if hasattr(row, 'text_score') and row.text_score is not None else None
            vector_score = float(row.vector_score) if hasattr(row, 'vector_score') and row.vector_score is not None else None

            results.append(SearchResult(
                id=row.id,
                content=row.content,
                memory_type=row.memory_type,
                summary=row.summary,
                score=score,
                text_score=text_score,
                vector_score=vector_score,
                metadata=dict(row.metadata) if row.metadata else {},
                created_at=row.created_at,
            ))

        return results
```

---

### 示例 3：记忆 CRUD 与会话管理

```python
"""
memory_crud.py
Agent 记忆的完整 CRUD 操作，包含会话管理和自动清理。
"""
from __future__ import annotations

from datetime import datetime, timedelta, timezone
from typing import Any
import uuid

from sqlalchemy import text, select, delete, update
from sqlalchemy.ext.asyncio import AsyncSession

from .agent_memory_models import (
    AgentMemory, AgentSession, MemoryLink, MemoryType, MemoryImportance,
)


class AgentMemoryStore:
    """
    Agent 记忆存储 —— 封装所有记忆的读写操作。

    提供以下功能：
    - 会话管理（开始/结束会话）
    - 记忆写入（单条/批量）
    - 记忆读取（按 ID/会话/类型/时间）
    - 记忆更新（修改内容/重要性/标签）
    - 记忆删除（单条/批量清理）
    - 自动过期（清理过期记忆）
    """

    def __init__(self, session: AsyncSession, agent_id: str):
        self.session = session
        self.agent_id = agent_id

    # ── 会话管理 ────────────────────────────────────────

    async def start_session(
        self,
        session_type: str = "interactive",
        metadata: dict | None = None,
    ) -> AgentSession:
        """开始一个新的 Agent 会话。"""
        agent_session = AgentSession(
            id=uuid.uuid4(),
            agent_id=self.agent_id,
            session_type=session_type,
            status="active",
            metadata_=metadata or {},
            started_at=datetime.now(timezone.utc),
        )
        self.session.add(agent_session)
        await self.session.commit()
        return agent_session

    async def end_session(self, session_id: uuid.UUID, token_count: int = 0):
        """结束一个会话。"""
        stmt = (
            update(AgentSession)
            .where(AgentSession.id == session_id)
            .values(
                status="closed",
                ended_at=datetime.now(timezone.utc),
                token_count=token_count,
            )
        )
        await self.session.execute(stmt)
        await self.session.commit()

    # ── 记忆写入 ────────────────────────────────────────

    async def remember(
        self,
        content: str,
        memory_type: MemoryType = MemoryType.CONVERSATION,
        session_id: uuid.UUID | None = None,
        importance: MemoryImportance = MemoryImportance.NORMAL,
        summary: str | None = None,
        source: str = "agent",
        metadata: dict | None = None,
        tags: list[str] | None = None,
        parent_id: int | None = None,
        ttl_seconds: int | None = None,   # 过期时间（秒）
    ) -> AgentMemory:
        """
        记录一条记忆。

        Args:
            content: 记忆内容
            memory_type: 记忆类型
            session_id: 关联的会话 ID
            importance: 重要性级别
            summary: 简短摘要
            source: 记忆来源
            metadata: 扩展元数据
            tags: 标签列表
            parent_id: 父记忆 ID（用于记忆链）
            ttl_seconds: 生存时间（过期后自动清理）

        Returns:
            创建的 AgentMemory 实例
        """
        expires_at = None
        if ttl_seconds:
            expires_at = datetime.now(timezone.utc) + timedelta(seconds=ttl_seconds)

        memory = AgentMemory(
            memory_id=uuid.uuid4(),
            agent_id=self.agent_id,
            session_id=session_id,
            memory_type=memory_type,
            importance=importance,
            summary=summary,
            content=content,
            source=source,
            metadata_=metadata or {},
            tags=tags or [],
            parent_id=parent_id,
            expires_at=expires_at,
        )
        self.session.add(memory)
        await self.session.commit()
        await self.session.refresh(memory)
        return memory

    async def remember_batch(
        self,
        memories: list[dict[str, Any]],
    ) -> list[AgentMemory]:
        """
        批量写入记忆。

        Args:
            memories: 记忆数据列表，每项包含 remember() 的参数

        Returns:
            创建的 AgentMemory 实例列表
        """
        created = []
        for mem in memories:
            m = await self.remember(**mem)
            created.append(m)
        return created

    async def link_memories(
        self,
        source_id: int,
        target_id: int,
        relation_type: str = "references",
        weight: float = 1.0,
    ):
        """在两个记忆之间建立关联。"""
        link = MemoryLink(
            source_memory_id=source_id,
            target_memory_id=target_id,
            relation_type=relation_type,
            weight=weight,
        )
        self.session.add(link)
        await self.session.commit()

    # ── 记忆读取 ────────────────────────────────────────

    async def recall_by_id(self, memory_id: int) -> AgentMemory | None:
        """按 ID 精确检索记忆。"""
        stmt = select(AgentMemory).where(
            AgentMemory.id == memory_id,
            AgentMemory.agent_id == self.agent_id,
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def recall_session(
        self,
        session_id: uuid.UUID,
        limit: int = 100,
        offset: int = 0,
    ) -> list[AgentMemory]:
        """检索整个会话的所有记忆。"""
        stmt = (
            select(AgentMemory)
            .where(
                AgentMemory.session_id == session_id,
                AgentMemory.agent_id == self.agent_id,
            )
            .order_by(AgentMemory.created_at.asc())
            .limit(limit)
            .offset(offset)
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def recall_by_type(
        self,
        memory_type: MemoryType,
        limit: int = 50,
        offset: int = 0,
    ) -> list[AgentMemory]:
        """按类型检索记忆。"""
        stmt = (
            select(AgentMemory)
            .where(
                AgentMemory.agent_id == self.agent_id,
                AgentMemory.memory_type == memory_type,
            )
            .order_by(AgentMemory.created_at.desc())
            .limit(limit)
            .offset(offset)
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def recall_by_time_range(
        self,
        start: datetime,
        end: datetime | None = None,
        limit: int = 100,
    ) -> list[AgentMemory]:
        """按时间范围检索记忆。"""
        conditions = [
            AgentMemory.agent_id == self.agent_id,
            AgentMemory.created_at >= start,
        ]
        if end:
            conditions.append(AgentMemory.created_at <= end)

        stmt = (
            select(AgentMemory)
            .where(*conditions)
            .order_by(AgentMemory.created_at.desc())
            .limit(limit)
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def recall_by_tags(
        self,
        tags: list[str],
        match_all: bool = False,
        limit: int = 50,
    ) -> list[AgentMemory]:
        """
        按标签检索记忆。

        Args:
            tags: 要匹配的标签列表
            match_all: True=需要匹配所有标签，False=匹配任一即可
        """
        if match_all:
            # 需要同时包含所有标签（PostgreSQL ARRAY 包含操作）
            stmt = (
                select(AgentMemory)
                .where(
                    AgentMemory.agent_id == self.agent_id,
                    AgentMemory.tags.contains(tags),  # @> 操作
                )
                .order_by(AgentMemory.created_at.desc())
                .limit(limit)
            )
        else:
            # 包含任一标签（使用 && 重叠操作）
            stmt = (
                select(AgentMemory)
                .where(
                    AgentMemory.agent_id == self.agent_id,
                    AgentMemory.tags.overlap(tags),  # && 操作
                )
                .order_by(AgentMemory.created_at.desc())
                .limit(limit)
            )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def recall_important(
        self,
        min_importance: MemoryImportance = MemoryImportance.IMPORTANT,
        limit: int = 20,
    ) -> list[AgentMemory]:
        """检索重要记忆。"""
        importance_order = {
            MemoryImportance.TRANSIENT: 0,
            MemoryImportance.NORMAL: 1,
            MemoryImportance.IMPORTANT: 2,
            MemoryImportance.CRITICAL: 3,
        }
        min_val = importance_order[min_importance]

        # 使用原始 SQL 因为枚举比较在不同数据库中表现不同
        stmt = text("""
            SELECT * FROM agent_memories
            WHERE agent_id = :agent_id
              AND importance IN ('important', 'critical')
            ORDER BY
                CASE importance
                    WHEN 'critical' THEN 0
                    WHEN 'important' THEN 1
                    ELSE 2
                END,
                created_at DESC
            LIMIT :limit
        """)
        result = await self.session.execute(stmt, {
            "agent_id": self.agent_id,
            "limit": limit,
        })
        return [AgentMemory(**row._mapping) for row in result]

    # ── 记忆更新 ────────────────────────────────────────

    async def update_memory(
        self,
        memory_id: int,
        **kwargs,
    ) -> AgentMemory | None:
        """
        更新记忆的属性。

        可更新字段：content, summary, importance, metadata, tags
        """
        allowed_fields = {
            "content", "summary", "importance", "metadata", "tags",
        }
        update_data = {k: v for k, v in kwargs.items() if k in allowed_fields}

        if not update_data:
            return None

        stmt = (
            update(AgentMemory)
            .where(
                AgentMemory.id == memory_id,
                AgentMemory.agent_id == self.agent_id,
            )
            .values(**update_data)
        )
        await self.session.execute(stmt)
        await self.session.commit()

        # 返回更新后的数据
        return await self.recall_by_id(memory_id)

    # ── 记忆删除 ────────────────────────────────────────

    async def forget(self, memory_id: int) -> bool:
        """删除单条记忆。"""
        stmt = delete(AgentMemory).where(
            AgentMemory.id == memory_id,
            AgentMemory.agent_id == self.agent_id,
        )
        result = await self.session.execute(stmt)
        await self.session.commit()
        return result.rowcount > 0

    async def forget_session(self, session_id: uuid.UUID) -> int:
        """删除整个会话的记忆。"""
        stmt = delete(AgentMemory).where(
            AgentMemory.session_id == session_id,
            AgentMemory.agent_id == self.agent_id,
        )
        result = await self.session.execute(stmt)
        await self.session.commit()
        return result.rowcount

    async def forget_expired(self) -> int:
        """清理所有过期记忆。"""
        stmt = delete(AgentMemory).where(
            AgentMemory.agent_id == self.agent_id,
            AgentMemory.expires_at <= datetime.now(timezone.utc),
        )
        result = await self.session.execute(stmt)
        await self.session.commit()
        return result.rowcount

    async def cleanup_old_memories(
        self,
        before: datetime,
        batch_size: int = 1000,
    ) -> int:
        """批量清理早期记忆（分批次执行以避免长事务）。"""
        total_deleted = 0
        while True:
            stmt = text("""
                DELETE FROM agent_memories
                WHERE ctid IN (
                    SELECT ctid FROM agent_memories
                    WHERE agent_id = :agent_id
                      AND created_at < :before
                    LIMIT :batch_size
                )
            """)
            result = await self.session.execute(stmt, {
                "agent_id": self.agent_id,
                "before": before,
                "batch_size": batch_size,
            })
            await self.session.commit()
            if result.rowcount == 0:
                break
            total_deleted += result.rowcount
        return total_deleted

    # ── 统计查询 ────────────────────────────────────────

    async def memory_count(self, memory_type: MemoryType | None = None) -> int:
        """统计记忆数量。"""
        conditions = [AgentMemory.agent_id == self.agent_id]
        if memory_type:
            conditions.append(AgentMemory.memory_type == memory_type)

        stmt = select(AgentMemory.id).where(*conditions)
        result = await self.session.execute(stmt)
        return len(result.fetchall())

    async def get_memory_stats(self) -> dict[str, Any]:
        """获取记忆统计信息。"""
        stmt = text("""
            SELECT
                COUNT(*) AS total,
                COUNT(*) FILTER (WHERE created_at >= CURRENT_DATE) AS today,
                COUNT(DISTINCT memory_type) AS type_count,
                AVG(LENGTH(content))::INTEGER AS avg_content_length,
                COUNT(*) FILTER (WHERE importance IN ('important', 'critical')) AS important_count,
                COUNT(*) FILTER (WHERE expires_at IS NOT NULL) AS expirable_count
            FROM agent_memories
            WHERE agent_id = :agent_id
        """)
        result = await self.session.execute(stmt, {"agent_id": self.agent_id})
        row = result.one()
        return dict(row._mapping)
```

---

### 示例 4：全文检索集成

```python
"""
fulltext_search.py
PostgreSQL 全文检索集成 —— 为 Agent 提供基于关键词的模糊搜索能力。
使用 tsvector/tsquery 实现比 ILIKE 更高效、更智能的文本搜索。
"""
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Any

from sqlalchemy import text, func, column, table
from sqlalchemy.ext.asyncio import AsyncSession


@dataclass
class FTSResult:
    """全文检索结果。"""
    id: int
    content: str
    headline: str           # 带高亮的摘要片段
    rank: float             # 相关性排名
    created_at: Any = None


class FullTextSearch:
    """
    PostgreSQL 全文检索适配器。

    支持：
    - 多语言分词（英语、中文等）
    - 排名与排序（ts_rank / ts_rank_cd）
    - 高亮显示（ts_headline）
    - 查询建议（词典扩展、前缀匹配）
    """

    def __init__(self, session: AsyncSession):
        self.session = session

    async def search(
        self,
        agent_id: str,
        query: str,
        language: str = "simple",
        limit: int = 20,
        offset: int = 0,
        highlight_max_words: int = 30,
    ) -> list[FTSResult]:
        """
        全文检索记忆内容。

        Args:
            agent_id: Agent 标识
            query: 搜索关键词
            language: 分词语言配置（simple/english/chinese）
            limit: 返回结果数
            offset: 偏移量
            highlight_max_words: 高亮片段的最大词数

        Returns:
            按相关性降序排列的结果列表
        """
        # 使用 PostgreSQL 的 plainto_tsquery 将用户输入转为查询向量
        # plainto_tsquery 会自动对输入进行分词并插入 & 操作符
        sql = text(f"""
            SELECT
                id,
                content,
                ts_headline(
                    '{language}',
                    content,
                    plainto_tsquery('{language}', :query_text),
                    'MaxWords={highlight_max_words}, MinWords=10, StartSel=<mark>, StopSel=</mark>'
                ) AS headline,
                ts_rank(
                    content_tsv,
                    plainto_tsquery('{language}', :query_text)
                ) AS rank
            FROM agent_memories
            WHERE agent_id = :agent_id
              AND content_tsv @@ plainto_tsquery('{language}', :query_text)
            ORDER BY rank DESC
            LIMIT :limit OFFSET :offset
        """)

        result = await self.session.execute(sql, {
            "agent_id": agent_id,
            "query_text": query,
            "limit": limit,
            "offset": offset,
        })

        return [
            FTSResult(
                id=row.id,
                content=row.content,
                headline=row.headline,
                rank=float(row.rank),
            )
            for row in result
        ]

    async def search_advanced(
        self,
        agent_id: str,
        query: str,
        language: str = "simple",
        use_prefix: bool = False,     # 启用前缀匹配（：* 操作符）
        min_rank: float = 0.01,       # 最低排名阈值
        limit: int = 20,
    ) -> list[FTSResult]:
        """
        高级全文检索。

        支持：
        - AND (&)、OR (|)、NOT (!) 布尔操作
        - 前缀匹配（:*
        - 短语搜索（用引号括起）
        """
        if use_prefix:
            # 将查询中的每个词转为前缀匹配
            tsquery_expr = func.plainto_tsquery(language, query).op("::")(text("tsquery"))
            # 实际上需要用 to_tsquery + :* 后缀实现前缀匹配
            words = query.strip().split()
            prefix_terms = " & ".join(f"{word}:*" for word in words)
            tsquery_clause = f"to_tsquery('{language}', :tsquery_str)"
            params = {
                "agent_id": agent_id,
                "tsquery_str": prefix_terms,
                "limit": limit,
                "min_rank": min_rank,
            }
        else:
            tsquery_clause = f"plainto_tsquery('{language}', :query_text)"
            params = {
                "agent_id": agent_id,
                "query_text": query,
                "limit": limit,
                "min_rank": min_rank,
            }

        sql = text(f"""
            SELECT
                id,
                content,
                ts_headline(
                    '{language}',
                    content,
                    {tsquery_clause},
                    'StartSel=<mark>, StopSel=</mark>'
                ) AS headline,
                ts_rank(content_tsv, {tsquery_clause}) AS rank
            FROM agent_memories
            WHERE agent_id = :agent_id
              AND content_tsv @@ {tsquery_clause}
              AND ts_rank(content_tsv, {tsquery_clause}) >= :min_rank
            ORDER BY rank DESC
            LIMIT :limit
        """)

        result = await self.session.execute(sql, params)
        return [
            FTSResult(
                id=row.id,
                content=row.content,
                headline=row.headline,
                rank=float(row.rank),
            )
            for row in result
        ]

    async def search_with_category(
        self,
        agent_id: str,
        query: str,
        memory_type: str | None = None,
        language: str = "simple",
        limit: int = 20,
    ) -> list[FTSResult]:
        """
        带分类过滤的全文检索。

        在全文检索的同时按 memory_type 做精确过滤，
        实现"关键词 + 分类"的多维搜索。
        """
        type_filter = "AND memory_type = :memory_type" if memory_type else ""

        sql = text(f"""
            SELECT
                id,
                content,
                memory_type,
                ts_headline(
                    '{language}',
                    content,
                    plainto_tsquery('{language}', :query_text),
                    'StartSel=<mark>, StopSel=</mark>'
                ) AS headline,
                ts_rank(
                    content_tsv,
                    plainto_tsquery('{language}', :query_text)
                ) AS rank
            FROM agent_memories
            WHERE agent_id = :agent_id
              AND content_tsv @@ plainto_tsquery('{language}', :query_text)
              {type_filter}
            ORDER BY rank DESC
            LIMIT :limit
        """)

        params = {
            "agent_id": agent_id,
            "query_text": query,
            "limit": limit,
        }
        if memory_type:
            params["memory_type"] = memory_type

        result = await self.session.execute(sql, params)
        return [
            FTSResult(
                id=row.id,
                content=row.content,
                headline=row.headline,
                rank=float(row.rank),
                created_at=None,
            )
            for row in result
        ]

    async def get_search_suggestions(
        self,
        agent_id: str,
        partial_word: str,
        limit: int = 10,
    ) -> list[str]:
        """
        搜索建议 —— 基于已有数据返回补全建议。

        使用 PostgreSQL 的 pg_trgm 扩展实现模糊前缀匹配。
        """
        sql = text("""
            SELECT DISTINCT word
            FROM (
                SELECT regexp_split_to_table(
                    lower(content), E'\\W+'
                ) AS word
                FROM agent_memories
                WHERE agent_id = :agent_id
            ) words
            WHERE word % :partial      -- 相似度匹配
               OR word LIKE :pattern   -- 前缀匹配
            ORDER BY similarity(word, :partial) DESC
            LIMIT :limit
        """)

        result = await self.session.execute(sql, {
            "agent_id": agent_id,
            "partial": partial_word.lower(),
            "pattern": f"{partial_word.lower()}%",
            "limit": limit,
        })
        return [row.word for row in result]

    async def rebuild_tsvector(self):
        """
        重建全文检索索引。

        在批量导入数据后，或在修改了分词配置后执行。
        """
        sql = text("""
            UPDATE agent_memories
            SET content_tsv = to_tsvector('simple', coalesce(content, ''))
            WHERE content_tsv IS NULL
               OR content_tsv = ''::tsvector
        """)
        result = await self.session.execute(sql)
        await self.session.commit()
        return result.rowcount
```

---

### 示例 5：完整的集成使用示例

```python
"""
demo.py
数据库记忆的完整使用示例 —— 展示如何在实际 Agent 中使用上述所有组件。
"""
import asyncio
import uuid
from datetime import datetime, timedelta, timezone

from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

from agent_memory_models import Base, AgentMemory, MemoryType, MemoryImportance
from memory_crud import AgentMemoryStore
from hybrid_search import HybridMemorySearch
from fulltext_search import FullTextSearch


# ── 模拟嵌入提供者 ──────────────────────────────────────
class DummyEmbedder:
    """测试用的嵌入提供者（实际使用时应替换为 OpenAI/Embedding API）。"""
    async def embed(self, text: str) -> list[float]:
        # 返回固定维度的随机向量（仅用于演示）
        import random
        random.seed(hash(text) % (2**31))
        return [random.random() for _ in range(384)]


# ── 主演示 ──────────────────────────────────────────────

async def main():
    # 1. 初始化数据库连接
    DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/agent_memory"

    engine = create_async_engine(DATABASE_URL, echo=True)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession)

    # 2. 开始使用
    agent_id = "assistant-v1"

    async with AsyncSessionLocal() as session:
        store = AgentMemoryStore(session, agent_id)

        # ── 创建会话 ──
        agent_session = await store.start_session(
            session_type="conversation",
            metadata={"user_id": "user-42", "channel": "web"},
        )
        print(f"会话开始: {agent_session.id}")

        # ── 写入记忆 ──
        mem1 = await store.remember(
            content="用户询问如何退款，我解释了退款流程：登录账户 -> 订单中心 -> 申请退款 -> 1-3 个工作日到账",
            memory_type=MemoryType.CONVERSATION,
            session_id=agent_session.id,
            importance=MemoryImportance.NORMAL,
            tags=["退款", "流程说明", "客服"],
            metadata={"user_satisfaction": 4},
        )
        print(f"记忆已记录: id={mem1.id}")

        mem2 = await store.remember(
            content="用户表示对等待时间不满意，建议优化退款处理速度",
            memory_type=MemoryType.OBSERVATION,
            session_id=agent_session.id,
            importance=MemoryImportance.IMPORTANT,
            tags=["用户反馈", "优化建议", "退款"],
            metadata={"feedback_type": "complaint", "urgency": "medium"},
        )
        print(f"观察已记录: id={mem2.id}")

        # ── 建立记忆关联 ──
        await store.link_memories(
            source_id=mem2.id,
            target_id=mem1.id,
            relation_type="references",
        )
        print("记忆关联已建立")

        # ── 精确检索 ──
        recalled = await store.recall_by_id(mem1.id)
        print(f"精确检索: {recalled.content[:50]}...")

        # ── 按标签检索 ──
        tagged = await store.recall_by_tags(["退款"])
        print(f"标签检索: 找到 {len(tagged)} 条相关记忆")

        # ── 按重要性检索 ──
        important = await store.recall_important()
        print(f"重要记忆: {len(important)} 条")

        # ── 全文检索 ──
        fts = FullTextSearch(session)
        fts_results = await fts.search(
            agent_id=agent_id,
            query="退款流程",
            language="simple",
        )
        print(f"全文检索: 找到 {len(fts_results)} 条")

        # ── 统计 ──
        stats = await store.get_memory_stats()
        print(f"记忆统计: {stats}")

        # ── 结束会话 ──
        await store.end_session(agent_session.id, token_count=1500)
        print("会话已结束")

    # 3. 使用混合搜索
    async with AsyncSessionLocal() as session:
        hybrid = HybridMemorySearch(session, DummyEmbedder())

        results = await hybrid.search(
            agent_id=agent_id,
            query_text="退款等待时间",
            memory_types=["conversation", "observation"],
            importance_levels=["normal", "important"],
            use_fts=True,
            use_vector=False,   # 演示时关闭向量搜索（需要 pgvector 安装）
            limit=10,
        )
        print(f"混合搜索: 找到 {len(results)} 条结果")
        for r in results:
            print(f"  [{r.score:.3f}] {r.content[:60]}...")

    await engine.dispose()
    print("数据库连接已关闭")


if __name__ == "__main__":
    asyncio.run(main())
```

## 场景判断

### 决策矩阵

```text
判断问题                           是 → 倾向数据库记忆   否 → 考虑其他方案
──────────────────────────────────────────────────────────────────────────────
1. 数据是否需要 ACID 保证？         数据库记忆             文件/文档存储
2. 查询是否以精确匹配为主？         数据库记忆             向量数据库
3. 是否需要复杂 JOIN 和聚合？      数据库记忆             文档存储（弱 JOIN）
4. 数据结构是否相对稳定？          数据库记忆             EAV/文档存储
5. 数据量是否在 GB-TB 级别？       数据库记忆             分布式 NoSQL
6. 团队是否有 SQL 经验？           数据库记忆             学习新的查询语言
7. 是否需要成熟的运维工具？         数据库记忆             较新的存储系统
8. 是否需要事务性写入？            数据库记忆             最终一致性系统
9. 是否需要语义相似度搜索？         数据库+pgvector        纯向量数据库
10. Agent 知识结构是否频繁变化？    调整策略               文档存储/图数据库
```

### 详细场景分析

```text
场景 A：客服 Agent
  - 存储：历史对话、用户信息、工单状态
  - 查询：按用户 ID/时间/关键词精确检索
  - 推荐：数据库记忆
  - 理由：数据结构固定（对话、用户、工单），查询条件明确，
          需要 ACID 保证（不能丢失工单状态变更）
  - 开源方案：PostgreSQL + SQLAlchemy

场景 B：通用知识问答 RAG Agent
  - 存储：大量非结构化文档、知识片段
  - 查询：语义相似度搜索（"找和这个问题相关的知识"）
  - 推荐：向量数据库
  - 理由：查询天然是语义性的，精确匹配意义不大
  - 替代方案：PostgreSQL + pgvector（需要混合搜索时）

场景 C：个人助理 Agent
  - 存储：用户偏好、日程、联系人、笔记
  - 查询：精确（"我的邮箱是多少"）+ 模糊（"上周的会议"）
  - 推荐：数据库记忆 + 全文检索
  - 理由：精确查询为主，但需要文本搜索能力
  - 开源方案：PostgreSQL + tsvector + pg_trgm

场景 D：金融分析 Agent
  - 存储：交易记录、市场数据、分析报告
  - 查询：复杂聚合分析（"Q3 的月均交易量趋势"）
  - 推荐：数据库记忆（必须）
  - 理由：SQL 的 GROUP BY / window function 是唯一高效的方案
  - 开源方案：TimescaleDB（时间序列优化版 PostgreSQL）

场景 E：多跳推理 Agent
  - 存储：实体关系网络（人物、公司、产品之间的关联）
  - 查询："A 投资的公司中有哪些和 B 合作过？"
  - 推荐：知识图谱
  - 理由：深层关系遍历在关系数据库中的 SQL 复杂且性能差
  - 替代方案：数据库记忆 + 应用层图遍历（数据量小时可行）

场景 F：对话历史存档
  - 存储：原始对话日志（每天数百万条）
  - 查询：按时间段聚合（"昨天有多少用户咨询"）
  - 推荐：文档存储/时间序列数据库
  - 理由：高写入吞吐、低查询复杂度、无事务要求
```

### 推荐组合策略

大多数生产级 Agent 实际上使用**混合架构**：

```text
┌───────────────────────────────────────────────────────┐
│                  Agent 记忆系统                           │
├───────────────────────────────────────────────────────┤
│                                                       │
│  核心记忆层（PostgreSQL）                                │
│  ┌─────────────────────────────────────────────────┐  │
│  │ • 用户配置、偏好、账户信息                         │  │
│  │ • 会话管理、Token 用量追踪                        │  │
│  │ • 工具调用记录、API 密钥管理                       │  │
│  │ • 需要 ACID 和精确查询的结构化数据                  │  │
│  │ • 复杂报表和聚合分析                              │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  语义检索层（pgvector / 向量数据库）                     │
│  ┌─────────────────────────────────────────────────┐  │
│  │ • 长文本语义索引                                  │  │
│  │ • 相似记忆检索（"和这段对话类似的情况"）             │  │
│  │ • RAG 知识库                                     │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  知识推理层（知识图谱）                                  │
│  ┌─────────────────────────────────────────────────┐  │
│  │ • 实体关系网络                                    │  │
│  │ • 多跳推理查询                                   │  │
│  │ • 因果链分析                                     │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
└───────────────────────────────────────────────────────┘
```

### 数据库选型速查

| 数据库 | 适合场景 | 不适合场景 | 推荐 Agent 类型 |
|--------|---------|-----------|----------------|
| **PostgreSQL** | 综合场景、需要扩展性、需要 pgvector | 纯键值存储、超高写入吞吐 | 大多数 Agent（推荐首选） |
| **SQLite** | 单机/嵌入式 Agent、开发原型、低并发 | 高并发、分布式部署 | 桌面 Agent、移动端 Agent |
| **MySQL** | 生态兼容性要求高、已有 MySQL 基础设施 | 需要复杂窗口函数、GIS | 与 PHP/Java 项目绑定的 Agent |
| **TimescaleDB** | 时间序列记忆、IoT Agent | 复杂关联查询 | 监控 Agent、时序分析 Agent |
| **CockroachDB** | 需要全球分布式、多区域部署 | 单机部署（太重） | 全球分布的 Agent 集群 |

## 总结

关系型数据库作为 Agent 记忆后端，是一种"古典但远未过时"的方案。在向量数据库和知识图谱大行其道的当下，数据库记忆凭借 ACID 保证、SQL 精确查询和成熟的运维生态，仍然是 Agent 结构化记忆的基石。

**核心价值**：当 Agent 需要**精确地记住和回忆事实**（用户配置、交易记录、会话历史）时，没有比关系数据库更合适的方案。

**适用边界**：当 Agent 的知识开始以非结构化文本为主、需要语义相似度检索、或者 Schema 频繁变化时，数据库记忆需要与其他存储方案配合使用。

**未来方向**：pgvector、pg_embedding 等扩展正在模糊关系数据库和向量数据库的边界，未来的 Agent 记忆系统可能不再需要"关系型 vs 向量"的二选一，而是在同一数据库中同时获得两种能力。

---

**上一篇**: [6.6.2 图遍历记忆](../6.6.2%20graph-traversal/learn.md)
**下一篇**: [6.6.4 文档存储](../6.6.4%20document-store/learn.md)
**相关模块**: [4.5 Custom Tools](../../4%20tool-use/4.5%20custom-tools/learn.md)
