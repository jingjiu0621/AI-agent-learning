# 6.6.4 文档存储（Document Store）

## 简单介绍

文档存储记忆（Document Store Memory）是一种以**半结构化文档**为基本单位的 AI Agent 外部记忆模式。它利用 NoSQL 文档数据库（如 MongoDB、Elasticsearch、Couchbase）作为记忆后端，将 Agent 的经验、对话历史、知识片段等信息存储为独立的自描述文档。

与传统关系型数据库将数据拆分为行和列的严格模式不同，文档存储允许每个"记忆单元"拥有自己的结构。一段 Agent 记忆可以是这样的：

```json
{
  "_id": "mem_conv_20260716_001",
  "type": "conversation",
  "session_id": "sess_a1b2c3",
  "agent_id": "customer-support-v2",
  "user_id": "usr_42",
  "messages": [
    {"role": "user", "content": "我的订单还没到", "ts": "2026-07-16T10:00:00Z"},
    {"role": "assistant", "content": "请提供订单号", "ts": "2026-07-16T10:00:05Z"},
    {"role": "user", "content": "ORD-20260701-888", "ts": "2026-07-16T10:00:12Z"}
  ],
  "context": {
    "intent": "order_inquiry",
    "sentiment": "frustrated",
    "priority": "high"
  },
  "metadata": {
    "created_at": "2026-07-16T10:00:00Z",
    "message_count": 3,
    "resolved": false
  }
}
```

同一个集合中的另一条记忆可以有完全不同的字段：

```json
{
  "_id": "mem_kb_ai_agents",
  "type": "knowledge",
  "agent_id": "customer-support-v2",
  "topic": "AI Agent 架构",
  "content": "AI Agent 由感知层、推理层、行动层和记忆层组成...",
  "tags": ["architecture", "agent", "memory"],
  "embeddings": [0.12, -0.34, 0.56, ...],
  "source": "internal_wiki",
  "version": 2,
  "related_topics": ["tool_use", "planning"],
  "access_count": 152
}
```

这种**无模式（Schema-less）**的灵活性，使得文档存储成为 AI Agent 记忆系统的理想选择——Agent 的记忆形态是动态演化的，而非预先定义好的固定表格。

### 文档存储记忆的三种核心用途

| 用途 | 说明 | 典型文档结构 |
|------|------|-------------|
| **对话记忆** | 存储 Agent 与用户的完整交互历史 | 嵌套 messages 数组、上下文标签、时间线 |
| **知识记忆** | 存储 Agent 可检索的知识片段 | 结构化内容、标签、来源、嵌入向量 |
| **事件记忆** | 存储 Agent 的决策和行动日志 | 事件类型、输入/输出、推理链、耗时 |

## 基本原理

### 文档导向存储（Document-Oriented Storage）

文档存储的核心思想是：**将相关数据聚合在一起，而不是分散到多个表中**。

```text
关系型数据库方式（规范化）：
  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  │ conversations│    │ messages    │    │ user_sessions│
  ├─────────────┤    ├─────────────┤    ├─────────────┤
  │ id          │    │ id          │    │ id          │
  │ user_id     │──→ │ conv_id     │──→ │ user_id     │
  │ started_at  │    │ role        │    │ device      │
  │ status      │    │ content     │    │ ip_address  │
  └─────────────┘    │ timestamp   │    └─────────────┘
                     │ tokens_used │
                     └─────────────┘
  查询一次对话需要 3 次 JOIN → 性能开销 + 编程复杂度

文档存储方式（反规范化）：
  ┌─────────────────────────────────────────────┐
  │ document: conversation                      │
  │ {                                           │
  │   _id: "conv_001",                          │
  │   user: { id: "usr_42", device: "mobile" }, │
  │   messages: [                               │
  │     { role: "user", content: "...", ts }    │
  │   ],                                        │
  │   metadata: { status, tokens, ... }         │
  │ }                                           │
  └─────────────────────────────────────────────┘
  一次查询获取完整对话 → 零 JOIN
```

### JSON / BSON 文档模型

大多数文档数据库使用 JSON 或其二进制变体 BSON 作为数据模型：

- **JSON（JavaScript Object Notation）**：文本格式，人类可读，Elasticsearch 使用
- **BSON（Binary JSON）**：二进制编码，MongoDB 使用，支持更多数据类型（Date、Binary、ObjectId）

**BSON 相对于 JSON 的扩展类型**：

```text
JSON 类型:         BSON 扩展类型:
  string              string (+ UTF-8 自动检测)
  number              int32, int64, double, decimal128
  boolean             boolean
  null                null
  array               array
  object              object
  ── 无 ──            ObjectId (12 字节唯一标识)
  ── 无 ──            Date (64 位毫秒时间戳)
  ── 无 ──            Binary (任意字节)
  ── 无 ──            Regex  (正则表达式)
  ── 无 ──            Timestamp (特殊递增计数器)
```

### 文档存储的四个核心特征

**1. Schema-less（无模式）**

同一个集合中的文档可以有不同的字段集合。Agent 的新记忆类型可以随时添加，无需执行 ALTER TABLE：

```python
# 第一条文档——有 embedding 字段
db.memories.insert_one({
    "type": "knowledge",
    "content": "Python 是一种动态语言",
    "embedding": [0.1, 0.2, 0.3],
})

# 第二条文档——没有 embedding，但有 tags 和 priority
db.memories.insert_one({
    "type": "knowledge",
    "content": "Redis 是一种 KV 存储",
    "tags": ["database", "cache"],
    "priority": 5,
})
```

**2. 文档自描述**

每个文档都包含描述其内容的字段（如 `type`、`metadata`），使得文档"自带 Schema"。Agent 可以在运行时通过检查文档字段来判断如何处理它。

**3. 嵌套结构**

文档支持深度嵌套的数组和子文档，天然适合表达 Agent 记忆的层次结构：

```json
{
  "episode": {
    "id": "ep_003",
    "goal": "为用户查询订单状态",
    "steps": [
      {
        "action": "call_order_api",
        "input": {"order_id": "ORD-888"},
        "output": {"status": "shipped", "eta": "2026-07-18"},
        "duration_ms": 450,
        "tokens_used": 120
      }
    ]
  }
}
```

**4. 内建索引**（详见"工程优化"章节）

### 数据建模方式：嵌入 vs 引用

这是文档存储中最核心的设计决策：

```text
嵌入（Embedding）                     引用（Reference）
─────────────────────────────         ─────────────────────────────
文档包含子文档                      文档包含对其他文档的 ID
┌──────────────────┐                 ┌──────────────────┐
│ conversation     │                 │ conversation     │
│ ├─ messages[0]   │                 │ ├─ message_ids:  │
│ ├─ messages[1]   │                 │ │  ["m1","m2"]   │
│ └─ messages[2]   │                 │ └─ user_id: "u1" │
└──────────────────┘                 └──────────────────┘
                                     ┌──────────────────┐
优点：                               │ message "m1"     │
• 一次读取获得全部数据               │ message "m2"     │
• 无 JOIN 开销                       └──────────────────┘
• 原子性写入
                                    优点：
缺点：                               • 避免文档膨胀
• 文档有大小限制（MongoDB 16MB）    • 支持独立访问子文档
• 更新子文档需要重写整个文档         • 减少数据重复
```

**嵌入优先原则**：在文档存储中，通常先考虑嵌入，只有在以下情况才使用引用：

```text
决定因素                    嵌入              引用
────────────────────────────────────────────────
子文档数量                  少（<100）        多（>1000）
子文档独立性                不独立            需要独立查询
子文档更新频率              低                高（频繁变动）
文档总大小                  <16MB             >16MB
原子性要求                  要求原子性        不要求
```

## 背景与演进

### 第一阶段：平面文件时代（1960s-1980s）

早期的 AI 系统使用平面文件（Flat Files）存储"记忆"——纯文本文件、CSV、或简单的二进制格式。每条"记忆"是一行文本或一个固定长度的记录。

```text
记忆存储：/var/agent/memory.dat
────────────────────────────────
2026-07-01 10:00:00|user|你好
2026-07-01 10:00:05|assistant|你好！有什么可以帮助你的？
2026-07-01 10:00:12|user|我的订单还没到
```

**局限**：无索引、无查询能力、无并发控制、格式解析靠约定。

### 第二阶段：层次化与 XML 时代（1990s-2000s）

随着 XML 的兴起，Agent 记忆开始使用结构化标记语言：

```xml
<memory>
  <conversation id="conv_001" timestamp="2026-07-16">
    <message role="user" ts="10:00:00">你好</message>
    <message role="assistant" ts="10:00:05">你好！有什么可以帮助你的？</message>
    <context intent="greeting" sentiment="neutral"/>
  </conversation>
</memory>
```

XML 提供了层次结构和 Schema 定义（DTD/XSD），但解析开销大、查询语言（XPath/XQuery）复杂、存大文本时效率低。

### 第三阶段：JSON 文档数据库兴起（2000s-2010s）

CouchDB（2005）和 MongoDB（2009）引领了 JSON 文档数据库的革命：

```text
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │   2005       │    │   2009       │    │   2010s      │
  │  CouchDB     │    │  MongoDB     │    │  爆发期      │
  │  • REST API  │    │  • BSON      │    │  • 云计算    │
  │  • MVCC      │    │  • 复制集    │    │  • 大数据    │
  │  • MapReduce │    │  • 分片      │    │  • JSON 通用 │
  └──────────────┘    └──────────────┘    └──────────────┘
```

关键转折：Web 2.0 应用需要灵活的数据模型来快速迭代产品功能。文档数据库允许开发人员在不改 Schema 的情况下添加字段，正好匹配了敏捷开发的节奏。

### 第四阶段：搜索与分析融合（2010s-2020s）

Elasticsearch（2010）将文档存储与全文搜索深度整合：

```text
传统文档存储                   搜索型文档存储
[MongoDB/Couchbase]            [Elasticsearch/OpenSearch]
  │                               │
  ├─ 按 ID 查询                   ├─ 全文搜索（相关性评分）
  ├─ 按字段过滤                   ├─ 聚合分析（Aggregations）
  ├─ 聚合管道                     ├─ 实时索引
  └─ TTL 索引                     └─ 日志/时间序列优化
```

**Elasticsearch 的本质也是一个文档存储**——每个文档是 JSON 对象，存储在倒排索引中而非 B-tree 中。

### 第五阶段：AI Agent 记忆专用存储（2020s-至今）

AI Agent 的发展催生了专门针对 Agent 记忆优化的文档存储模式：

```text
传统文档存储 → Agent 记忆存储的演变

1. 增加向量字段支持
   MongoDB Atlas Vector Search / Elasticsearch dense_vector
   → 实现了语义搜索 + 关键词搜索的混合检索

2. 增加 TTL（Time-To-Live）语义
   自动过期不再重要的短期记忆

3. 增加 Change Streams 支持
   实时监听记忆变化 → 触发 Agent 行为

4. 增加记忆层级
   工作记忆 → 短期记忆 → 长期记忆 的自动迁移

5. 增加多租户分片策略
   不同 Agent/用户的数据隔离
```

## 核心矛盾

### Schema 灵活性 vs. 查询可预测性

文档存储的核心矛盾在于：**自由是有代价的**。

```text
                 Schema 灵活 ←──────────→ Schema 严格
                 │                            │
    文档存储       │      关系型数据库           │
                 │                            │
  优势：           │    优势：                   │
  • 快速迭代       │    • 查询可预测             │
  • 适应变化       │    • 强类型保证             │
  • 多态数据       │    • JOIN 优化              │
  • 低建模成本     │    • 引用完整性             │
                 │                            │
  代价：           │    代价：                   │
  • 查询不确定     │    • 迁移成本高             │
  • 弱类型保证     │    • 迭代速度慢             │
  • 无 JOIN       │    • ORM 映射复杂           │
  • 冗余数据       │    • 层次数据扁化           │
```

### 四个具体矛盾维度

**1. 查询性能不可预测**

同一集合中文档结构不同，导致索引效果不确定：

```text
情况 A：所有文档都有 indexed_field
  → 索引正常工作 → 查询快

情况 B：只有 50% 的文档有 indexed_field
  → 索引只覆盖 50% 数据 → 另一半需要全表扫描

情况 C：indexed_field 在不同文档中是不同类型
  { "ts": "2026-07-16" }         ← string
  { "ts": 2026-07-16T00:00:00Z } ← date
  → 索引类型冲突 → 查询行为不确定
```

**2. 数据冗余与一致性问题**

反规范化（Denormalization）引入了数据冗余，导致更新异常：

```python
# 用户信息嵌入在每条对话文档中
db.conversations.insert_one({
    "conv_id": "c1",
    "user": {"name": "张三", "email": "zhang@example.com"},
    "messages": [...]
})
db.conversations.insert_one({
    "conv_id": "c2",
    "user": {"name": "张三", "email": "zhang@example.com"},
    "messages": [...]
})

# 问题：当张三的邮箱更新为 zhang_new@example.com 时，
# 需要更新所有嵌入用户信息的对话文档
# → 要么容忍不一致，要么付出多次写入的代价
```

**3. 无 JOIN 带来的应用层负担**

文档存储不支持（或不鼓励）JOIN。当确实需要关联数据时，负担转移到应用层：

```python
# 应用层手动关联（两次查询 + 内存拼接）
conversation = db.conversations.find_one({"conv_id": "c1"})
user = db.users.find_one({"user_id": conversation["user_id"]})

# 或者使用 $lookup（MongoDB 聚合管道的左外连接）
# 但性能远不如关系型数据库的原生 JOIN
```

**4. 事务范围受限**

MongoDB 4.0+ 支持多文档事务，但范围有限：

```text
关系型数据库：                    文档存储：
  事务范围任意                     事务范围有限
  ┌─────────────────┐             ┌─────────────────┐
  │ BEGIN TRANS;    │             │ session.start() │
  │ UPDATE table_a; │             │ op1 (doc A)     │
  │ UPDATE table_b; │             │ op2 (doc B)     │
  │ UPDATE table_c; │             │ # 同一分片上 OK │
  │ COMMIT;         │             │ # 跨分片 → 性能差│
  └─────────────────┘             └─────────────────┘
```

## 主流优化方向

### 1. MongoDB Atlas Search / Elasticsearch 混合搜索

文档存储正从纯结构化查询走向"结构化 + 语义"的混合搜索：

```text
传统文档存储查询：
  db.memories.find({"tags": "agent", "type": "knowledge"})

混合搜索（关键词 + 语义）：
  db.memories.aggregate([
    {
      "$search": {
        "index": "memory_search",
        "compound": {
          "must": [
            {"text": {"query": "AI Agent", "path": "content"}},
            {"exists": {"path": "tags"}}
          ]
        }
      }
    },
    {
      "$vectorSearch": {
        "queryVector": [0.1, -0.2, ...],
        "path": "embeddings",
        "numCandidates": 100,
        "limit": 10
      }
    }
  ])
```

**为什么这对 Agent 重要**：Agent 的记忆检索不仅需要精确匹配（"找到标签为 agent 的记忆"），还需要语义相似度检索（"找到和当前问题相关的记忆"）。混合搜索在一次查询中同时满足两个需求。

### 2. 反规范化记忆文档设计模式

针对 Agent 记忆的常见反规范化模式：

```text
模式 1：单文档完整对话
  将所有消息嵌入到一个对话文档中
  适合：完整上下文需要的场景
  代价：文档随对话增长而膨胀

模式 2：分页消息文档
  将对话拆分为固定大小（如 50 条）的消息块
  适合：长对话、流式对话
  代价：检索时需合并多个文档

模式 3：主从文档
  主文档（摘要）+ 从文档（详细消息）
  适合：需要快速浏览 + 按需加载详情
  代价：双文档写入的原子性

模式 4：事件溯源文档
  每个 Action 作为一个独立事件文档
  适合：审计、回放、调试
  代价：重建状态需要聚合所有事件
```

### 3. Change Streams 实时记忆更新

文档数据库的 Change Streams 使 Agent 可以实时响应记忆变化：

```python
# MongoDB Change Streams — 实时监听记忆变更
async def watch_memory_changes():
    pipeline = [
        {"$match": {"operationType": {"$in": ["insert", "update", "replace"]}}}
    ]
    async with db.memories.watch(pipeline) as stream:
        async for change in stream:
            doc_id = change["documentKey"]["_id"]
            operation = change["operationType"]
            
            if operation == "insert":
                await on_new_memory(doc_id)
            elif operation == "update":
                await on_memory_updated(doc_id)
            
            # 触发 Agent 行为
            await agent.react_to_memory_change(doc_id, operation)
```

**应用场景**：
- 多个 Agent 共享记忆池：一个 Agent 写入的记忆，其他 Agent 实时感知
- 记忆老化系统：当记忆被标记为"过期"时，触发归档流程
- 实时记忆同步：Agent 实例之间的记忆同步

### 4. 分片策略与多租户隔离

当为多个 Agent（或用户）服务时，文档存储的分片策略至关重要：

```text
常见分片键策略：

1. 按 agent_id 哈希分片（均匀分布）
   shard key: { agent_id: "hashed" }
   优点：写入均匀分布
   缺点：范围查询需广播到所有分片

2. 按租户 ID 范围分片（数据局部性）
   shard key: { tenant_id: 1, created_at: -1 }
   优点：同一租户的数据在连续块上
   缺点：热分片问题（大租户占用单个分片）

3. 按复合键分片（均衡 + 局部性）
   shard key: { agent_id: "hashed", type: 1 }
   优点：写入均匀，查询可定向
   缺点：不能单独按 type 查询

场景               推荐分片键                 原因
─────────────────────────────────────────────────────
多租户 SaaS Agent   { tenant_id: 1 }          按租户隔离
单 Agent 大规模记忆   { date: -1 }             时间序列访问模式
多 Agent 协作        { agent_id: "hashed" }    写入负载均衡
```

### 5. TTL 索引与记忆生命周期管理

Agent 的记忆是有生命周期的——不是所有记忆都需要永久保留。

```python
from datetime import datetime, timedelta

# 在 MongoDB 中创建 TTL 索引
# 短期记忆：1 小时后自动删除
db.short_term_memories.create_index(
    {"created_at": 1},
    expireAfterSeconds=3600  # 1 小时
)

# 中期记忆：7 天后自动删除
db.medium_term_memories.create_index(
    {"last_accessed_at": 1},
    expireAfterSeconds=604800  # 7 天
)

# 条件 TTL（MongoDB 5.0+）：根据条件决定是否过期
# 只有 resolved=True 的对话才自动过期
db.conversations.create_index(
    {"resolved_at": 1},
    partialFilterExpression={"resolved": True, "type": "conversation"},
    expireAfterSeconds=86400 * 30  # 30 天
)
```

**三级记忆生命周期**：

```text
记忆层级            TTL          存储位置          访问频率
─────────────────────────────────────────────────────────
工作记忆（WM）     分钟级        内存/RAM           极高
短期记忆（STM）    小时-天       文档数据库          高
长期记忆（LTM）    月-永久       文档数据库+归档     低

自动迁移流程：
  Agent 推理
     │
     ▼
  ┌──────────────┐
  │ 工作记忆      │ ←→ 当前对话上下文（内存）
  └──────┬───────┘
         │ 上下文窗口溢出
         ▼
  ┌──────────────┐
  │ 短期记忆      │ ←→ 活跃对话文档（MongoDB, TTL=24h）
  └──────┬───────┘
         │ 对话结束 / 重要性评估
         ▼
  ┌──────────────┐
  │ 长期记忆      │ ←→ 精华记忆文档（MongoDB, TTL=永不过期）
  │              │     + 冷数据归档（S3/Glacier）
  └──────────────┘
```

### 6. 聚合管道（Aggregation Pipeline）记忆分析

MongoDB 的聚合管道是强大的记忆分析和转换工具：

```python
# 聚合管道：Agent 对话摘要 + 指标统计
pipeline = [
    # 1. 过滤出某 Agent 的对话记忆
    {"$match": {
        "agent_id": "customer-support-v2",
        "type": "conversation",
        "created_at": {"$gte": start_of_day}
    }},
    # 2. 按 Agent 分组统计
    {"$group": {
        "_id": "$agent_id",
        "total_conversations": {"$sum": 1},
        "total_messages": {"$sum": "$metadata.message_count"},
        "resolved_count": {"$sum": {"$cond": ["$metadata.resolved", 1, 0]}},
        "avg_sentiment": {"$avg": "$context.sentiment_score"},
        "top_intents": {"$push": "$context.intent"}
    }},
    # 3. 计算指标
    {"$addFields": {
        "resolution_rate": {
            "$divide": ["$resolved_count", "$total_conversations"]
        },
        "avg_messages_per_conv": {
            "$divide": ["$total_messages", "$total_conversations"]
        }
    }},
    # 4. 格式化输出
    {"$project": {
        "_id": 0,
        "agent_id": "$_id",
        "summary": {
            "会话数": "$total_conversations",
            "消息总数": "$total_messages",
            "解决率": {"$round": ["$resolution_rate", 2]},
            "平均消息数": {"$round": ["$avg_messages_per_conv", 1]},
            "常见意图": "$top_intents"
        }
    }}
]

result = db.memories.aggregate(pipeline)
```

## 产生的结果

文档存储作为 Agent 记忆后端，带来了以下关键能力：

### 1. Schema 自由演化

Agent 的记忆结构可以随着业务需求变化而变化，无需停机迁移：

```text
版本 1（v1.0）：最小对话模型
  { "messages": [{"role": "user", "content": "..."}] }

版本 2（v1.1）：增加元数据
  { "messages": [...], "metadata": {"created_at": "...", "duration_ms": 123} }

版本 3（v2.0）：增加上下文
  { "messages": [...], "metadata": {...}, "context": {"intent": "...", "sentiment": "..."} }

版本 4（v2.1）：增加多模态支持
  { "messages": [{"role": "user", "content": "...", "attachments": [{"type": "image", "url": "..."}]}],
    "metadata": {...}, "context": {...} }
```

**关键结果**：Agent 的记忆系统与产品迭代解耦。不需要在每次添加新字段时执行 ALTER TABLE。

### 2. 丰富文本搜索

Elasticsearch 等文档存储提供强大的全文搜索能力：

```text
关键词搜索     "AI Agent 记忆"
模糊搜索       "AI Agnet~2"（允许 2 个编辑距离）
短语搜索       "\"长期记忆\" 最近 7 天"
前缀搜索       "知识*"
多字段搜索     "content:Agent AND tags:memory"
相关性排序     _score 降序排列
高亮显示       <em>AI Agent</em> 记忆
```

对于 Agent 的知识检索场景，全文搜索比向量搜索更适合精确的术语匹配（如代码片段、产品名、版本号）。

### 3. 嵌套数据的自然表达

Agent 记忆中的许多数据结构天然是嵌套的，文档存储正好匹配：

```json
{
  "推理记录": {
    "初始问题": "订单状态",
    "推理步骤": [
      {"步骤": 1, "操作": "识别意图", "结果": "订单查询"},
      {"步骤": 2, "子推理": [
        {"子步骤": 2.1, "操作": "提取订单号", "来源": "用户消息"},
        {"子步骤": 2.2, "操作": "调用 API", "参数": {"订单号": "ORD-888"}}
      ]},
      {"步骤": 3, "操作": "格式化回复", "模板": "order_status"}
    ],
    "最终输出": "您的订单已发货，预计 7 月 18 日到达。"
  }
}
```

在关系型数据库中表达这种结构需要 4-5 个关联表；在文档存储中只需要一个文档。

### 4. 水平扩展

文档数据库天生支持水平扩展（分片），使 Agent 记忆可以跨越数百台服务器：

```text
文档存储的水平扩展：
                    ┌─────────┐
                    │  Router  │
                    │ (mongos) │
                    └────┬────┘
           ┌─────────────┼─────────────┐
           │             │             │
        ┌──┴──┐      ┌──┴──┐      ┌──┴──┐
        │Shard│      │Shard│      │Shard│
        │  A  │      │  B  │      │  C  │
        │40GB │      │42GB │      │39GB │
        └─────┘      └─────┘      └─────┘
           │             │             │
        ┌──┴──┐      ┌──┴──┐      ┌──┴──┐
        │Repl │      │Repl │      │Repl │
        │Set A│      │Set B│      │Set C│
        │3 节点│     │3 节点│     │3 节点│
        └─────┘      └─────┘      └─────┘
```

当单个服务器的容量不足时，增加新的分片即可扩展 Agent 的记忆容量，而无需停机。

### 5. 时间序列友好的自动过期

Agent 的短期记忆天然具有时间属性——旧的对话不再需要保留。文档存储的 TTL 索引和自动归档功能使记忆生命周期管理变得简单：

```python
# 数据分层策略
storage_tiers = {
    "hot": {
        "backend": "MongoDB SSD",
        "ttl": "7 天",
        "purpose": "活跃 Agent 的短期和中期记忆"
    },
    "warm": {
        "backend": "MongoDB HDD",
        "ttl": "90 天",
        "purpose": "近期结束的对话和决策日志"
    },
    "cold": {
        "backend": "S3 + Parquet",
        "ttl": "永久",
        "purpose": "合规归档和离线分析"
    }
}
```

## 最大挑战

### 分布式一致性——Agent 记忆的因果一致性

当 Agent 记忆分布存储在多个分片/节点上，一致性成为最棘手的挑战。

#### 问题场景

```text
场景：两个 Agent 实例协作处理同一用户请求

时间线 T:
┌──── Agent A ────┐            ┌──── Agent B ────┐
│ 1. 读取记忆:    │            │                 │
│    conv_001     │            │                 │
│                 │            │                 │
│ 2. 写入记忆:    │            │ 1'. 读取记忆:   │
│    添加消息 #3  │            │    conv_001     │
│    ─→ 写入 shard 1           │    ╰→ 未看到消息 #3
│                 │            │                 │
│ 3. 写入记忆:    │            │ 2'. 写入记忆:   │
│    添加消息 #4  │            │    添加消息 #5  │
│    ─→ 写入 shard 2           │    ─→ 写入 shard 1
│                 │            │                 │
问题: Agent B 在步骤 2' 写入消息 #5 时，
      尚未看到 Agent A 的消息 #3，
      但消息 #5 可能引用/依赖消息 #3 中的信息
      → Agent B 的记忆上下文不完整
```

#### 具体一致性挑战

**1. 读己之写（Read-your-writes）一致性**

Agent A 写入一条记忆后，立即读取可能读不到（因为复制延迟）：

```text
Agent A                       Primary                    Secondary
  │                             │                          │
  │── insert memory_doc ──────→│ (写成功)                  │
  │                             │                          │
  │── find memory_doc ────────→│ (找到)                    │
  │                             │                          │
  │── (切换节点)               │                          │
  │                             │                          │
  │── find memory_doc ────────────────────────────────→ (未复制到, 没找到)
  │                             │                          │
  Agent A: "我刚写进去的数据怎么就没了？？"
```

**缓解方案**：

```python
class MongoMemoryStore:
    """
    具有读偏好控制的文档存储记忆层。
    使用 "primary" 读偏好确保读己之写一致性。
    """

    def __init__(self, uri: str, database: str, collection: str):
        from pymongo import MongoClient, ReadPreference

        self.client = MongoClient(uri)
        # 写操作默认写入 primary
        # 读操作指定从 primary 读取（保证一致性）
        self.db = self.client.get_database(database)
        self.collection = self.db.get_collection(collection)

    def write_with_ack(self, doc: dict) -> str:
        """写操作：等待大多数节点确认 (w: majority)。"""
        result = self.collection.insert_one(
            doc,
            # w="majority" 确保写入已传播到大多数节点
            w="majority",
            # journal=True 确保写入已持久化到日志
            journal=True,
        )
        return str(result.inserted_id)

    def read_with_consistency(self, filter: dict) -> dict | None:
        """
        读操作：从 primary 读取，牺牲部分读取性能换取强一致性。
        对于短期记忆（很快会被再次读取），这很重要。
        """
        return self.collection.find_one(
            filter,
            read_preference=ReadPreference.PRIMARY,
        )

    def read_with_performance(self, filter: dict) -> dict | None:
        """
        读操作：从 secondary 读取（高吞吐，低延迟，弱一致）。
        对于长期记忆（写后不立即读取），这是更好的选择。
        """
        return self.collection.find_one(
            filter,
            read_preference=ReadPreference.SECONDARY_PREFERRED,
        )
```

**2. 部分更新与原子性**

一个记忆文档的部分更新可能跨分片，导致部分成功部分失败：

```python
# MongoDB 中部分更新的行为取决于操作范围

# 单文档更新（原子性保证）
db.memories.update_one(
    {"_id": "conv_001"},
    {"$push": {"messages": {"role": "user", "content": "你好"}}}
)
# → 原子性 ✓

# 多文档更新（非原子性，一轮一轮执行）
db.memories.update_many(
    {"agent_id": "agent_v2", "type": "conversation"},
    {"$set": {"metadata.processed": True}}
)
# → 非原子性，部分文档可能更新成功，部分失败
# → 需要应用层补偿逻辑

# 跨文档事务（MongoDB 4.0+，有限原子性）
with client.start_session() as session:
    with session.start_transaction():
        db.memories.update_one(
            {"_id": "conv_001"},
            {"$set": {"status": "archived"}},
            session=session,
        )
        db.memories.update_one(
            {"_id": "conv_002"},
            {"$set": {"status": "archived"}},
            session=session,
        )
        # 事务提交 → 要么全部成功，要么全部回滚
```

**3. 因果一致性（Causal Consistency）**

对于 Agent 来说，记忆事件之间的因果关系比严格全局顺序更重要：

```text
因果一致性要求：
  如果事件 A 因果地影响事件 B，那么所有观察者先看到 A，再看到 B。

Agent 场景的例子：
  事件 1: Agent 接收用户消息"我的订单号是 ORD-888"（cause）
  事件 2: Agent 调用查询订单 API 获取结果（effect）
  事件 3: Agent 回复用户"您的订单已发货"（effect of effect）

  任何观察这个 Agent 记忆的外部系统必须看到：
    事件 1 → 事件 2 → 事件 3 （这个顺序）
  但不要求看到与这个因果关系链无关的其他事件。
```

MongoDB 4.2+ 支持因果一致性会话：

```python
from pymongo import MongoClient
from pymongo.client_session import ClientSession

class CausalMemoryStore:
    """
    具有因果一致性保障的文档存储记忆层。
    保证 Agent 的操作顺序在分布式环境中被正确维护。
    """

    def __init__(self, uri: str):
        self.client = MongoClient(uri)
        self.db = self.client.agent_memory

    def create_causal_session(self) -> ClientSession:
        """创建一个保证因果一致性的会话。"""
        session = self.client.start_session(causal_consistency=True)
        return session

    async def agent_reasoning_flow(self, user_message: str):
        """
        使用因果一致性会话的 Agent 推理流。
        确保：接收消息 → 推理 → 生成响应 的顺序在所有节点上一致。
        """
        with self.create_causal_session() as session:
            # 1. 存储用户消息
            msg_id = self.db.events.insert_one(
                {"type": "user_message", "content": user_message},
                session=session,
            ).inserted_id

            # 2. 读取相关记忆（一定能看到自己在同一会话中的写入）
            relevant_memories = list(
                self.db.memories.find(
                    {"agent_id": "my_agent"},
                    session=session,
                )
            )

            # 3. 存储推理过程
            reasoning_id = self.db.events.insert_one(
                {"type": "reasoning", "input_msg_id": str(msg_id)},
                session=session,
            ).inserted_id

            # 4. 存储响应
            response_id = self.db.events.insert_one(
                {"type": "response", "reasoning_id": str(reasoning_id)},
                session=session,
            ).inserted_id

            # 因果顺序保证：
            # msg_id → reasoning_id → response_id
            # 任何使用同一 session 的读取一定按此顺序看到事件
```

**4. 跨分片事务性能**

MongoDB 的跨分片事务需要两阶段提交，延迟显著高于单分片操作：

```text
操作类型                 延迟（p50）    延迟（p99）
───────────────────────────────────────────────
单文档写入               2-5ms         10-20ms
单分片多文档事务          5-10ms        20-50ms
跨分片多文档事务          20-100ms      200-500ms
```

对于 Agent 记忆操作，建议的设计原则：

```text
设计原则                   原因
─────────────────────────────────────────────
优先单文档操作             跨分片事务成本高
将因果相关数据放在同一分片  避免跨分片协调
容忍最终一致性            减少一致性协议开销
使用补偿事务而非强事务      降低延迟和复杂度
```

### 5. 文档大小限制

MongoDB 的单个文档最大为 16MB。对于长对话 Agent：

```python
class ConversationManager:
    """
    处理长对话的文档大小管理。
    当对话增长超过阈值时自动分片。
    """

    MAX_DOC_SIZE_BYTES = 10 * 1024 * 1024  # 预留 6MB 安全余量

    async def append_message(self, conv_id: str, message: dict):
        """追加消息，同时监控文档大小。"""
        conv = await self.db.conversations.find_one({"_id": conv_id})

        if not conv:
            # 新对话
            await self.db.conversations.insert_one({
                "_id": conv_id,
                "messages": [message],
                "message_count": 1,
                "segment": 0,
            })
            return

        # 估算新文档大小
        estimated_size = len(json.dumps(conv)) + len(json.dumps(message))
        if estimated_size > self.MAX_DOC_SIZE_BYTES:
            # 当前段已满，创建新段
            new_segment = conv["segment"] + 1
            await self.db.conversations.insert_one({
                "_id": f"{conv_id}_seg_{new_segment}",
                "messages": [message],
                "message_count": 1,
                "segment": new_segment,
                "prev_segment": conv_id if conv["segment"] == 0
                    else f"{conv_id}_seg_{conv['segment']}",
            })
        else:
            await self.db.conversations.update_one(
                {"_id": conv_id},
                {
                    "$push": {"messages": message},
                    "$inc": {"message_count": 1},
                }
            )
```

## 能力边界

### 文档存储能做什么

```text
✅ 适合文档存储的任务：

  1. 任意结构的 Agent 记忆
     对话历史、知识片段、事件日志、决策树
     每个文档可以有不同的字段集合

  2. 全文搜索
     精确关键词匹配、短语搜索、模糊搜索
     相关性评分（TF-IDF、BM25）
     搜索结果高亮

  3. 嵌套数据
     多级嵌套的推理链、分层知识
     数组内的子文档数组

  4. 水平扩展
     按 agent_id、tenant_id 或时间范围分片
     数百 TB 级的 Agent 记忆存储

  5. 快速迭代
     添加新字段不影响现有数据
     无需 Schema 迁移
     适合敏捷开发的 Agent 项目
```

### 文档存储不能做什么

```text
❌ 不适合文档存储的任务：

  1. 复杂关系查询
     无 JOIN——关联数据需要在应用层拼接
     多对多关系建模困难（需要双倍引用处理）

  2. 强引用完整性
     不会自动阻止对不存在引用的写入
     孤立的引用需要应用层清理

  3. 复杂事务
     跨分片事务性能差，应尽量避免
     缺少像 SQL 那样的隔离级别（SERIALIZABLE 等）

  4. 强类型保证
     同一个集合中文档结构可以完全不一致
     "类型错误"在写入时才被发现，而非定义时

  5. 数据一致性
     默认是最终一致性
     强一致模式会牺牲性能和可用性
```

## 对比

### 文档存储 vs 其他存储方案

```text
对比维度               文档存储               关系型数据库            向量数据库            键值存储
────────────────────────────────────────────────────────────────────────────────────────────────
数据模型               JSON/BSON 文档         行 + 列 + 外键          向量嵌入             键 → 值

Schema                Schema-less（无模式）   固定的 Schema           通常是固定维度         无模式（值任意）

查询方式              字段查询 + 聚合管道       SQL JOIN + 子查询       向量相似度搜索         按键查询 + 简单扫描
                        + 全文搜索                                       + 元数据过滤

完整性保证             无（应用层负责）         引用完整性（FK）         N/A                  N/A

事务                  跨分片事务弱             强 ACID                 读后写                CAS 操作

索引类型              B-tree + 全文 + 地理     B-tree + Hash           HNSW / IVF           哈希表

水平扩展              原生支持                 复杂（分库分表）         支持                  原生支持

最适合 Agent 的场景    对话历史、知识库、日志    事务数据、计费、     语义检索、记忆匹配    缓存、会话状态、计数器
                       结构化记忆               用户账户               RAG 检索

最不适合的场景         多对多关联、强一致性需求  Schema 频繁变化        精确关键词匹配        范围查询、聚合

延迟（p50 读）         毫秒级（1-10ms）        毫秒级（1-5ms）         毫秒级（2-20ms）      亚毫秒级（<1ms）

写入吞吐               高（10K-100K ops/s）    中（1K-10K ops/s）      高（10K+ ops/s）       极高（100K+ ops/s）

典型数据库             MongoDB, Elasticsearch   PostgreSQL, MySQL       Pinecone, Qdrant      Redis, DynamoDB
                        Couchbase
```

### 文档存储 vs 关系型数据库：Agent 记忆建模对比

```text
场景：Agent 记忆——存储带有用户信息的对话历史

文档存储方案（MongoDB）：
  db.conversations.insertOne({
    _id: "conv_abc",
    user: { id: 42, name: "张三", tier: "premium" },
    messages: [
      { role: "user", content: "退款怎么操作？", ts: "..." },
      { role: "assistant", content: "请提供订单号", ts: "..." },
    ],
    context: { intent: "refund", sentiment: "frustrated" },
    resolution: { status: "open", escalated: false }
  })

关系型方案（PostgreSQL）：
  -- 需要 4 个表 + 2 个 JOIN
  SELECT * FROM conversations c
  JOIN users u ON c.user_id = u.id
  JOIN messages m ON m.conversation_id = c.id
  LEFT JOIN resolution r ON r.conversation_id = c.id
  WHERE c.id = 'conv_abc';

对比结果：
  - 文档存储：1 次查询，0 JOIN，5ms
  - 关系型：4 表关联，2+ JOIN，15ms
  - 但关系型：可以轻松查询"所有 premium 用户的未解决退款请求"
  - 文档存储：需要建复合索引或应用层过滤
```

## 工程优化

### 1. MongoDB 聚合管道记忆摘要

将原始对话记忆聚合法 Agent 可消费的摘要信息：

```python
from datetime import datetime, timedelta
from pymongo import MongoClient, ASCENDING, DESCENDING, TEXT, HASNED
from pymongo.collection import Collection


class MemoryAnalytics:
    """
    基于 MongoDB 聚合管道的记忆分析引擎。
    提供 Agent 对话摘要、趋势分析、异常检测。
    """

    def __init__(self, collection: Collection):
        self.collection = collection

    def agent_daily_summary(self, agent_id: str, date: datetime) -> dict:
        """
        生成单个 Agent 的每日记忆摘要。
        用于 Agent 的"自我反思"——回顾一天的工作。
        """
        start = date.replace(hour=0, minute=0, second=0, microsecond=0)
        end = start + timedelta(days=1)

        pipeline = [
            {"$match": {
                "agent_id": agent_id,
                "type": "conversation",
                "metadata.created_at": {"$gte": start, "$lt": end}
            }},
            # 展开消息数组以分析消息级指标
            {"$unwind": "$messages"},
            # 按意图分组
            {"$group": {
                "_id": "$context.intent",
                "count": {"$sum": 1},
                "avg_messages": {"$avg": "$metadata.message_count"},
                "resolved": {"$sum": {"$cond": [
                    {"$eq": ["$metadata.resolved", True]}, 1, 0
                ]}},
                "total_tokens": {"$sum": "$metadata.total_tokens"},
                "sentiments": {"$push": "$context.sentiment"},
            }},
            # 计算解决率
            {"$addFields": {
                "resolution_rate": {
                    "$cond": [
                        {"$gt": ["$count", 0]},
                        {"$divide": ["$resolved", "$count"]},
                        0
                    ]
                }
            }},
            # 按数量降序排序
            {"$sort": {"count": -1}},
        ]

        results = list(self.collection.aggregate(pipeline))
        return {
            "agent_id": agent_id,
            "date": date.isoformat(),
            "total_conversations": sum(r["count"] for r in results),
            "by_intent": results,
        }

    def top_knowledge_topics(self, agent_id: str, limit: int = 10) -> list[dict]:
        """
        分析 Agent 使用频率最高的知识主题。
        帮助 Agent 或开发者了解"知识记忆的热点区域"。
        """
        pipeline = [
            {"$match": {
                "agent_id": agent_id,
                "type": "knowledge",
                "access_count": {"$gte": 1}
            }},
            {"$sort": {"access_count": -1}},
            {"$limit": limit},
            {"$project": {
                "topic": 1,
                "content": {"$substrCP": ["$content", 0, 200]},
                "access_count": 1,
                "tags": 1,
                "last_accessed": 1,
            }}
        ]
        return list(self.collection.aggregate(pipeline))

    def conversation_timeline(
        self, agent_id: str, user_id: str, limit: int = 20
    ) -> list[dict]:
        """
        获取特定用户与 Agent 之间完整的对话时间线。
        """
        pipeline = [
            {"$match": {
                "agent_id": agent_id,
                "user_id": user_id,
                "type": "conversation",
            }},
            {"$sort": {"metadata.created_at": -1}},
            {"$limit": limit},
            {"$project": {
                "_id": 1,
                "session_id": 1,
                "message_count": 1,
                "context.intent": 1,
                "context.sentiment": 1,
                "metadata.resolved": 1,
                "metadata.created_at": 1,
                "messages": {"$slice": ["$messages", 0, 3]},  # 只取前 3 条摘要
            }}
        ]
        return list(self.collection.aggregate(pipeline))


# 使用示例
analytics = MemoryAnalytics(db.memories)
summary = analytics.agent_daily_summary(
    agent_id="customer-support-v2",
    date=datetime.utcnow(),
)
```

### 2. Elasticsearch 倒排索引 + 向量混合搜索

Elasticsearch 同时提供倒排索引（关键词）和向量索引（语义），特别适合 Agent 的混合记忆检索。

**索引映射定义**：

```python
from elasticsearch import Elasticsearch


class AgentMemoryIndex:
    """
    Elasticsearch 记忆索引管理器。
    配置倒排索引 + 向量索引的混合搜索方案。
    """

    def __init__(self, es_client: Elasticsearch, index_name: str = "agent_memory"):
        self.es = es_client
        self.index_name = index_name

    def create_index(self, vector_dim: int = 768):
        """
        创建同时支持全文搜索和向量搜索的索引。
        vector_dim 取决于使用的嵌入模型（如 text-embedding-ada-002 是 1536）。
        """
        mapping = {
            "settings": {
                "analysis": {
                    "analyzer": {
                        "ik_max_word": {  # 中文分词
                            "type": "custom",
                            "tokenizer": "ik_max_word",
                        },
                        "english_analyzer": {
                            "type": "standard",
                        }
                    }
                },
                "index": {
                    "number_of_shards": 3,
                    "number_of_replicas": 2,
                }
            },
            "mappings": {
                "properties": {
                    # === 结构化字段 ===
                    "memory_id": {"type": "keyword"},
                    "agent_id": {"type": "keyword"},
                    "type": {"type": "keyword"},        # conversation / knowledge / event
                    "user_id": {"type": "keyword"},
                    "session_id": {"type": "keyword"},

                    # === 全文字段（中文 + 英文双分析器）===
                    "content": {
                        "type": "text",
                        "analyzer": "ik_max_word",
                        "fields": {
                            "english": {
                                "type": "text",
                                "analyzer": "standard",
                            },
                            "keyword": {  # 精确匹配场景
                                "type": "keyword",
                            }
                        }
                    },
                    "title": {"type": "text", "analyzer": "ik_max_word"},

                    # === 标签与分类 ===
                    "tags": {"type": "keyword"},
                    "intent": {"type": "keyword"},
                    "sentiment": {"type": "keyword"},

                    # === 时间字段 ===
                    "created_at": {"type": "date"},
                    "last_accessed": {"type": "date"},
                    "expire_at": {"type": "date"},

                    # === 数值字段 ===
                    "access_count": {"type": "integer"},
                    "importance_score": {"type": "float"},

                    # === 向量字段 ===
                    "embedding": {
                        "type": "dense_vector",
                        "dims": vector_dim,
                        "index": True,
                        "similarity": "cosine",
                    },

                    # === 元数据（动态映射）===
                    "metadata": {
                        "type": "object",
                        "dynamic": True,
                    }
                }
            }
        }

        if self.es.indices.exists(index=self.index_name):
            self.es.indices.delete(index=self.index_name)

        self.es.indices.create(index=self.index_name, body=mapping)
        return f"索引 {self.index_name} 创建成功 (向量维度: {vector_dim})"
```

**混合搜索实现**：

```python
class HybridMemorySearcher:
    """
    混合记忆搜索器。
    同时使用全文搜索和向量搜索，结合 BM25 和余弦相似度的结果。
    """

    def __init__(self, es_client: Elasticsearch, index_name: str):
        self.es = es_client
        self.index_name = index_name

    def hybrid_search(
        self,
        query: str,
        query_vector: list[float] | None = None,
        agent_id: str | None = None,
        memory_type: str | None = None,
        size: int = 10,
        alpha: float = 0.5,
    ) -> list[dict]:
        """
        混合搜索：BM25 全文搜索 + 向量语义搜索。

        Args:
            query: 文本查询
            query_vector: 查询文本的向量嵌入
            agent_id: 按 Agent 过滤
            memory_type: 按记忆类型过滤
            size: 返回结果数
            alpha: 混合权重 (0=仅向量, 1=仅全文)

        Returns:
            排序后的搜索结果列表
        """
        # 构建过滤器
        filters = []
        if agent_id:
            filters.append({"term": {"agent_id": agent_id}})
        if memory_type:
            filters.append({"term": {"type": memory_type}})

        # 构建查询
        must_queries = []

        # 全文搜索部分
        if query:
            must_queries.append({
                "multi_match": {
                    "query": query,
                    "fields": ["content^2", "title^1.5", "tags^1"],
                    "type": "best_fields",
                    "operator": "or",
                    "minimum_should_match": "60%",
                }
            })

        # 向量搜索部分
        has_vector = query_vector is not None and len(query_vector) > 0

        if has_vector:
            # Elasticsearch 使用 kNN 搜索向量
            knn_query = {
                "field": "embedding",
                "query_vector": query_vector,
                "k": size * 2,
                "num_candidates": 100,
            }

        # 组合查询
        if has_vector and alpha < 1.0:
            # Elasticsearch 的混合搜索 (>= 8.8)
            body = {
                "query": {
                    "bool": {
                        "must": must_queries,
                        "filter": filters,
                    }
                },
                "knn": {
                    "field": "embedding",
                    "query_vector": query_vector,
                    "k": size * 2,
                    "num_candidates": 100,
                    "filter": {
                        "bool": {"filter": filters}
                    } if filters else None,
                },
                "size": size,
                # 混合排名 (RRF: Reciprocal Rank Fusion)
                "rank": {
                    "rrf": {
                        "rank_constant": 60,
                        "window_size": 200,
                    }
                }
            }
        else:
            # 纯全文搜索或纯向量搜索
            if query and not has_vector:
                body = {
                    "query": {
                        "bool": {
                            "must": must_queries,
                            "filter": filters,
                        }
                    },
                    "size": size,
                }
            elif has_vector and not query:
                body = {
                    "knn": knn_query,
                    "query": {
                        "bool": {"filter": filters}
                    } if filters else {"match_all": {}},
                    "size": size,
                }
            else:
                body = {
                    "query": {"match_all": {}},
                    "size": size,
                }

        response = self.es.search(index=self.index_name, body=body)
        return [
            {
                "id": hit["_id"],
                "score": hit["_score"],
                "source": hit["_source"],
            }
            for hit in response["hits"]["hits"]
        ]

    def search_by_similar_memory(
        self, memory_id: str, size: int = 5
    ) -> list[dict]:
        """
        找到与某条记忆相似的记忆（更像此记忆）。
        适用于 Agent 的"联想"功能。
        """
        # 先获取原始记忆的向量
        source = self.es.get(index=self.index_name, id=memory_id)
        if not source["found"]:
            return []

        source_doc = source["_source"]
        embedding = source_doc.get("embedding")
        if not embedding:
            return []

        # 用向量搜索相似内容
        body = {
            "knn": {
                "field": "embedding",
                "query_vector": embedding,
                "k": size + 1,  # +1 因为会包含自身
                "num_candidates": 50,
            },
            "size": size + 1,
        }

        response = self.es.search(index=self.index_name, body=body)
        results = []
        for hit in response["hits"]["hits"]:
            if hit["_id"] != memory_id:
                results.append({
                    "id": hit["_id"],
                    "score": hit["_score"],
                    "source": hit["_source"],
                })
        return results[:size]
```

### 3. Change Stream 实时记忆事件处理

```python
import asyncio
import json
import logging
from pymongo import MongoClient, change_stream
from typing import Callable, Awaitable

logger = logging.getLogger(__name__)


class MemoryChangeStreamProcessor:
    """
    记忆变更流处理器。
    实时监听文档存储中的 Agent 记忆变更，触发后续处理。
    """

    def __init__(self, mongo_uri: str, db_name: str, collection_name: str):
        self.client = MongoClient(mongo_uri)
        self.db = self.client[db_name]
        self.collection = self.db[collection_name]
        self._handlers: dict[str, list[Callable]] = {}
        self._running = False

    def on(self, event: str, handler: Callable[[dict], Awaitable[None]]):
        """
        注册记忆变更事件处理器。

        event 类型:
          - "insert": 新记忆创建
          - "update": 记忆更新
          - "replace": 记忆替换
          - "delete": 记忆删除
          - "*": 所有类型
        """
        self._handlers.setdefault(event, []).append(handler)

    async def start(self):
        """启动变更流监听。"""
        self._running = True
        pipeline = [
            {"$match": {
                "operationType": {
                    "$in": ["insert", "update", "replace", "delete"]
                }
            }},
            # 可选：只关注特定字段变更
            # {"$match": {
            #     "updateDescription.updatedFields.context.intent": {"$exists": True}
            # }}
        ]

        logger.info("记忆变更流监听器已启动")
        async with self.collection.watch(pipeline) as stream:
            while self._running:
                try:
                    change = await stream.next()
                    await self._dispatch(change)
                except Exception as e:
                    logger.error(f"变更流处理错误: {e}")
                    await asyncio.sleep(1)  # 避免忙等

    def stop(self):
        self._running = False

    async def _dispatch(self, change: dict):
        """分发变更事件到注册的处理器。"""
        event_type = change["operationType"]
        doc_id = change["documentKey"]["_id"]
        logger.debug(f"记忆变更事件: {event_type} | 文档: {doc_id}")

        # 构建事件对象
        event = {
            "type": event_type,
            "doc_id": str(doc_id),
            "timestamp": change.get("clusterTime"),
        }

        # 如果是更新操作，附上变更的字段
        if event_type == "update":
            event["updated_fields"] = change.get("updateDescription", {}).get(
                "updatedFields", {}
            )

        # 分发到特定类型处理器
        for handler in self._handlers.get(event_type, []):
            try:
                await handler(event)
            except Exception as e:
                logger.error(f"处理器 {handler.__name__} 执行失败: {e}")

        # 分发到全局处理器
        for handler in self._handlers.get("*", []):
            try:
                await handler(event)
            except Exception as e:
                logger.error(f"全局处理器 {handler.__name__} 执行失败: {e}")


# ========== 实际应用场景 ==========

# 场景 1：新记忆写入时自动构建向量索引
async def on_memory_insert(event: dict):
    """
    当一条新的知识记忆被写入时，自动生成向量嵌入。
    这是 Agent 记忆系统的"索引自动化"。
    """
    from your_embedding_service import generate_embedding

    doc_id = event["doc_id"]
    db = MongoClient()["agent_memory"]
    doc = db.memories.find_one({"_id": doc_id})

    if doc and doc.get("type") == "knowledge" and not doc.get("embedding"):
        # 生成向量嵌入
        embedding = generate_embedding(doc["content"])
        db.memories.update_one(
            {"_id": doc_id},
            {"$set": {"embedding": embedding, "indexed": True}}
        )
        logger.info(f"记忆 {doc_id} 向量索引完成")


# 场景 2：对话解决后触发记忆摘要
async def on_conversation_resolved(event: dict):
    """
    当对话被标记为 resolved 时，触发记忆摘要生成。
    这有助于从原始对话中提取"精华记忆"存入长期记忆。
    """
    if event.get("type") != "update":
        return

    fields = event.get("updated_fields", {})
    if fields.get("metadata.resolved") == True:
        doc_id = event["doc_id"]
        db = MongoClient()["agent_memory"]
        conversation = db.memories.find_one({"_id": doc_id})

        if not conversation:
            return

        # 生成对话摘要
        summary = generate_conversation_summary(conversation)

        # 将摘要存入长期记忆
        db.long_term_memories.insert_one({
            "type": "summary",
            "source_conv_id": doc_id,
            "agent_id": conversation.get("agent_id"),
            "content": summary,
            "intent": conversation.get("context", {}).get("intent"),
            "created_at": datetime.utcnow(),
        })
        logger.info(f"对话 {doc_id} 摘要已存入长期记忆")


def generate_conversation_summary(conv: dict) -> str:
    """从原始对话生成摘要。"""
    messages = conv.get("messages", [])
    intent = conv.get("context", {}).get("intent", "unknown")

    # 这里可以使用 LLM 生成摘要，或者用规则提取
    # 简单版本：取关键信息
    user_msgs = [m["content"] for m in messages if m["role"] == "user"]
    assistant_msgs = [m["content"] for m in messages if m["role"] == "assistant"]

    summary = (
        f"[{intent}] "
        f"用户问题: {user_msgs[0] if user_msgs else 'N/A'} | "
        f"Agent 回复: {assistant_msgs[-1] if assistant_msgs else 'N/A'}"
    )
    return summary


# 启动变更流监听
async def run_change_stream():
    processor = MemoryChangeStreamProcessor(
        mongo_uri="mongodb://localhost:27017",
        db_name="agent_memory",
        collection_name="memories",
    )

    processor.on("insert", on_memory_insert)
    processor.on("update", on_conversation_resolved)
    processor.on("*", lambda e: logger.debug(f"事件: {e['type']}"))

    await processor.start()
```

### 4. Shard Key 选择策略

分片键的选择直接影响 Agent 记忆系统的写入吞吐和查询性能：

```python
class ShardKeyAnalyzer:
    """
    分片键分析与建议工具。
    根据 Agent 记忆的访问模式，推荐最优分片键。
    """

    # 候选分片键及其评估
    CANDIDATES = [
        {
            "key": {"agent_id": "hashed"},
            "description": "按 agent_id 哈希均匀分布",
            "write_distribution": "均匀",
            "range_queries": "需广播",
            "targeted_queries": "高效（单 agent）",
            "best_for": "多 Agent 共享集群，写入量大",
        },
        {
            "key": {"tenant_id": 1, "created_at": -1},
            "description": "按租户 + 时间范围",
            "write_distribution": "热点（大租户）",
            "range_queries": "高效（时间范围扫描）",
            "targeted_queries": "高效",
            "best_for": "时间序列 Agent 记忆",
        },
        {
            "key": {"type": 1, "_id": "hashed"},
            "description": "按记忆类型 + 哈希 ID",
            "write_distribution": "较好",
            "range_queries": "可优化（按 type）",
            "targeted_queries": "高效（type + ID）",
            "best_for": "混合记忆类型（对话+知识+事件）",
        },
        {
            "key": {"user_id": "hashed"},
            "description": "按 user_id 哈希",
            "write_distribution": "均匀",
            "range_queries": "需广播",
            "targeted_queries": "高效（按用户查询）",
            "best_for": "C 端 Agent，以用户为中心的记忆",
        },
    ]

    @staticmethod
    def analyze_workload(agent_count: int, daily_ops: int,
                         primary_query: str) -> list:
        """
        根据工作负载分析推荐分片键。

        Args:
            agent_count: Agent 实例数量
            daily_ops: 每日操作数
            primary_query: 主要查询模式
                "by_agent": 按 Agent 查询
                "by_user": 按用户查询
                "by_time": 按时间范围查询
                "by_type": 按记忆类型查询
        """
        query_map = {
            "by_agent": {"agent_id": "hashed"},
            "by_user": {"user_id": "hashed"},
            "by_time": {"tenant_id": 1, "created_at": -1},
            "by_type": {"type": 1, "_id": "hashed"},
        }

        recommended_key = query_map.get(primary_query, {"_id": "hashed"})

        # 生成 shard key 命令
        shard_command = {
            "shardCollection": "agent_memory.memories",
            "key": recommended_key,
        }

        # 如果是范围分片，建议预先创建分片标签
        if isinstance(recommended_key, dict):
            first_value = list(recommended_key.values())[0]
            if isinstance(first_value, int) and first_value in (1, -1):
                shard_command["numInitialChunks"] = max(
                    agent_count * 2, 100  # 至少 100 个预分片
                )

        return {
            "recommended_key": recommended_key,
            "shard_command": shard_command,
            "reasoning": f"基于主要查询模式 '{primary_query}' 推荐",
            "caveats": [
                "分片键一旦设置不可修改（需重新导入数据）",
                "确保分片键在写操作中总是存在",
                "避免使用单调递增的键作为分片键（会造成写热点）",
            ]
        }
```

### 5. 索引策略与查询优化

针对 Agent 记忆的常见查询模式，建立精确的索引策略：

```python
class MemoryIndexOptimizer:
    """
    记忆索引优化器。
    根据 Agent 记忆的典型查询模式，创建最优索引组合。
    """

    # 常见 Agent 记忆查询模式及对应的索引
    QUERY_PATTERNS = {
        # 查询 Agent 某时间段的对话
        "conversations_by_agent_and_time": {
            "filter": {"agent_id": "...", "type": "conversation",
                       "metadata.created_at": {"$gte": ..., "$lt": ...}},
            "index": [
                ("agent_id", ASCENDING),
                ("type", ASCENDING),
                ("metadata.created_at", DESCENDING),
            ],
            "covered": True,
        },

        # 查询用户的最近对话
        "recent_by_user": {
            "filter": {"user_id": "...", "type": "conversation"},
            "sort": {"metadata.created_at": -1},
            "index": [
                ("user_id", ASCENDING),
                ("type", ASCENDING),
                ("metadata.created_at", DESCENDING),
            ],
            "covered": True,
        },

        # 查询活跃的知识条目
        "active_knowledge": {
            "filter": {"type": "knowledge", "access_count": {"$gte": 10}},
            "sort": {"access_count": -1},
            "index": [
                ("type", ASCENDING),
                ("access_count", DESCENDING),
            ],
        },

        # 全文搜索（需要文本索引）
        "fulltext_search": {
            "filter": {"$text": {"$search": "..."}},
            "index": [
                ("content", TEXT),
                ("title", TEXT),
            ],
            "index_type": "text",
        },

        # TTL 自动清理
        "ttl_cleanup": {
            "filter": {"expire_at": {"$lte": "..."}},
            "index": [
                ("expire_at", ASCENDING),
            ],
            "index_type": "ttl",
            "extra_options": {"expireAfterSeconds": 0},
        },
    }

    @staticmethod
    def create_optimal_indexes(collection: Collection, workload_type: str = "general"):
        """
        为集合创建最优的索引组合。

        Args:
            collection: MongoDB 集合
            workload_type: 工作负载类型
                "general"        - 通用 Agent 记忆
                "conversation"   - 对话密集型
                "knowledge"      - 知识检索密集型
                "event_log"      - 事件日志密集型
        """
        patterns = {
            "general": [
                "conversations_by_agent_and_time",
                "recent_by_user",
                "active_knowledge",
                "fulltext_search",
                "ttl_cleanup",
            ],
            "conversation": [
                "conversations_by_agent_and_time",
                "recent_by_user",
                "ttl_cleanup",
            ],
            "knowledge": [
                "active_knowledge",
                "fulltext_search",
            ],
            "event_log": [
                "ttl_cleanup",
            ],
        }

        selected = patterns.get(workload_type, patterns["general"])
        indexes_created = []

        for pattern_name in selected:
            pattern = MemoryIndexOptimizer.QUERY_PATTERNS.get(pattern_name)
            if not pattern:
                continue

            index_keys = pattern["index"]
            index_options = {
                "name": f"idx_{pattern_name}",
                "background": True,  # 后台构建，不阻塞写入
            }

            # 为文本索引添加选项
            if pattern.get("index_type") == "text":
                weights = {}
                for key, _ in index_keys:
                    if key in ("content", "title", "tags"):
                        weights[key] = 10  # content 权重最高
                index_options["weights"] = weights

            # TTL 索引
            if pattern.get("index_type") == "ttl":
                index_options["expireAfterSeconds"] = 0

            try:
                collection.create_index(
                    index_keys,
                    **index_options,
                )
                indexes_created.append(pattern_name)
            except Exception as e:
                logger.error(f"创建索引 {pattern_name} 失败: {e}")

        return indexes_created

    @staticmethod
    def analyze_query_explain(explain_result: dict) -> dict:
        """
        分析查询执行计划，给出优化建议。
        """
        stages = explain_result.get("queryPlanner", {}).get(
            "winningPlan", {}
        ).get("stages", [])

        # 检查是否使用了 COLLSCAN（全表扫描）
        has_collscan = any(
            "COLLSCAN" in json.dumps(stage)
            for stage in stages
        )

        # 检查是否使用了 SORT 而非索引排序
        has_sort = any(
            stage.get("stage") == "SORT"
            for stage in stages
        )

        recommendations = []
        if has_collscan:
            recommendations.append("⚠️ 全表扫描 - 请为查询字段创建索引")
        if has_sort and not has_collscan:
            recommendations.append("ℹ️ 内存排序 - 考虑创建覆盖排序的复合索引")

        return {
            "has_full_scan": has_collscan,
            "has_memory_sort": has_sort,
            "stage_count": len(stages),
            "recommendations": recommendations,
        }
```

### 6. 复制集与高可用

```python
class HighAvailabilityConfig:
    """
    高可用配置生成器。
    为 Agent 记忆系统配置 MongoDB 复制集。
    """

    @staticmethod
    def generate_replica_set_config(
        replicas: int = 3,
        priority_primary: int = 2,
        priority_secondary: int = 1,
        write_concern: str = "majority",
        read_preference: str = "secondaryPreferred",
    ) -> dict:
        """
        生成复制集配置。

        Args:
            replicas: 节点数（建议奇数：3, 5, 7）
            priority_primary: 主节点优先级
            priority_secondary: 从节点优先级
            write_concern: 写关注级别
            read_preference: 读偏好

        Returns:
            复制集配置
        """
        return {
            "replication": {
                "replSetName": "agent_memory_rs",
                "oplogSizeMB": 10240,  # 10GB oplog
                "enableMajorityReadConcern": True,
            },
            "members": [
                {"_id": i, "host": f"mongodb-node-{i}:27017",
                 "priority": priority_primary if i == 0 else priority_secondary}
                for i in range(replicas)
            ],
            "settings": {
                "getLastErrorDefaults": {"w": write_concern},
            },
            "read_preference": read_preference,
            "write_concern": {
                "w": write_concern,
                "j": True,  # 写操作写入日志
                "wtimeoutMS": 5000,
            },
            # 对于 Agent 记忆而言，建议的故障转移策略
            "election_priority": {
                "rationale": "Agent 可以容忍秒级中断，所以优先数据一致性而非快速恢复",
                "settings": {
                    "electionTimeoutMillis": 10000,  # 10 秒未检测到 primary 触发选举
                    "heartbeatIntervalMillis": 2000,  # 2 秒心跳
                }
            }
        }
```

## 代码示例：完整的文档存储 Agent 记忆系统

下面是一个将上述所有模式整合在一起的完整实现：

```python
"""
agent_document_memory.py
完整的文档存储 Agent 记忆系统。

整合了：
  - MongoDB 文档模型
  - 全文搜索（Elasticsearch）
  - 实时变更流
  - 混合搜索（全文 + 向量）
  - 记忆生命周期管理（TTL）
  - 因果一致性
  - 聚合分析
"""

import asyncio
import json
import logging
import time
from datetime import datetime, timedelta
from dataclasses import dataclass, field
from typing import Any, Optional

from pymongo import MongoClient, ASCENDING, DESCENDING, TEXT, HASNED
from pymongo.errors import DuplicateKeyError, BulkWriteError
from elasticsearch import Elasticsearch

logger = logging.getLogger(__name__)


# ========== 数据模型 ==========

@dataclass
class MemoryDocument:
    """通用记忆文档模型。"""
    memory_id: str
    agent_id: str
    type: str                    # conversation / knowledge / event
    content: str
    created_at: datetime = field(default_factory=datetime.utcnow)
    tags: list[str] = field(default_factory=list)
    embedding: Optional[list[float]] = None
    metadata: dict[str, Any] = field(default_factory=dict)
    ttl_seconds: Optional[int] = None  # None = 永不过期

    def to_mongo(self) -> dict:
        """转换为 MongoDB 文档。"""
        doc = {
            "_id": self.memory_id,
            "agent_id": self.agent_id,
            "type": self.type,
            "content": self.content,
            "created_at": self.created_at,
            "tags": self.tags,
            "metadata": self.metadata,
        }
        if self.embedding:
            doc["embedding"] = self.embedding
        if self.ttl_seconds:
            doc["expire_at"] = self.created_at + timedelta(seconds=self.ttl_seconds)
        return doc

    def to_es(self) -> dict:
        """转换为 Elasticsearch 文档。"""
        doc = {
            "memory_id": self.memory_id,
            "agent_id": self.agent_id,
            "type": self.type,
            "content": self.content,
            "title": self.metadata.get("title", ""),
            "tags": self.tags,
            "created_at": self.created_at.isoformat(),
            "metadata": self.metadata,
        }
        if self.embedding:
            doc["embedding"] = self.embedding
        return doc


# ========== 文档存储记忆管理器 ==========

class DocumentMemoryStore:
    """
    统一的文档存储记忆管理器。

    提供 Agent 记忆的 CRUD、搜索、分析和生命周期管理。
    同时使用 MongoDB 和 Elasticsearch 作为后端。
    """

    def __init__(
        self,
        mongo_uri: str = "mongodb://localhost:27017",
        es_hosts: list[str] = None,
        db_name: str = "agent_memory",
        mongo_collection: str = "memories",
        es_index: str = "agent_memory",
        vector_dim: int = 768,
    ):
        # MongoDB 连接
        self.mongo_client = MongoClient(mongo_uri)
        self.mongo_db = self.mongo_client[db_name]
        self.mongo_coll = self.mongo_db[mongo_collection]

        # Elasticsearch 连接
        self.es = Elasticsearch(es_hosts or ["http://localhost:9200"])
        self.es_index = es_index
        self.vector_dim = vector_dim

        # 初始化索引
        self._init_indexes()

    def _init_indexes(self):
        """初始化 MongoDB 和 ES 索引。"""
        # MongoDB 索引
        self.mongo_coll.create_index([
            ("agent_id", ASCENDING),
            ("type", ASCENDING),
            ("created_at", DESCENDING),
        ], name="agent_type_time", background=True)

        self.mongo_coll.create_index([
            ("agent_id", ASCENDING),
            ("type", ASCENDING),
        ], name="agent_type", background=True)

        self.mongo_coll.create_index(
            [("expire_at", ASCENDING)],
            name="ttl_expire",
            expireAfterSeconds=0,
            background=True,
        )

        self.mongo_coll.create_index(
            [("content", TEXT), ("tags", TEXT)],
            name="text_search",
            background=True,
        )

        # ES 索引
        if not self.es.indices.exists(index=self.es_index):
            mapping = {
                "settings": {
                    "analysis": {
                        "analyzer": {
                            "ik_max_word": {
                                "type": "custom",
                                "tokenizer": "ik_max_word",
                            }
                        }
                    },
                    "number_of_shards": 3,
                    "number_of_replicas": 1,
                },
                "mappings": {
                    "properties": {
                        "memory_id": {"type": "keyword"},
                        "agent_id": {"type": "keyword"},
                        "type": {"type": "keyword"},
                        "content": {
                            "type": "text",
                            "analyzer": "ik_max_word",
                        },
                        "tags": {"type": "keyword"},
                        "created_at": {"type": "date"},
                        "embedding": {
                            "type": "dense_vector",
                            "dims": self.vector_dim,
                            "index": True,
                            "similarity": "cosine",
                        },
                        "metadata": {"type": "object", "dynamic": True},
                    }
                }
            }
            self.es.indices.create(index=self.es_index, body=mapping)
            logger.info(f"ES 索引 {self.es_index} 创建完成")

    # ========== 写操作 ==========

    def write(self, memory: MemoryDocument) -> str:
        """
        写入一条 Agent 记忆到文档存储。
        同时写入 MongoDB 和 Elasticsearch。

        使用因果一致性会话确保写入的先后顺序。
        """
        mongo_doc = memory.to_mongo()
        es_doc = memory.to_es()

        with self.mongo_client.start_session(causal_consistency=True) as session:
            # 写入 MongoDB
            self.mongo_coll.insert_one(mongo_doc, session=session)

            # 写入 Elasticsearch
            self.es.index(
                index=self.es_index,
                id=memory.memory_id,
                body=es_doc,
                refresh="wait_for",  # 等待索引刷新，保证可搜索
            )

        logger.debug(f"记忆写入成功: {memory.memory_id}")
        return memory.memory_id

    def write_batch(self, memories: list[MemoryDocument]) -> int:
        """
        批量写入记忆（事务性）。
        适用于 Agent 批量同步短期记忆到长期记忆。
        """
        if not memories:
            return 0

        mongo_docs = [m.to_mongo() for m in memories]
        es_docs = [(m.memory_id, m.to_es()) for m in memories]

        with self.mongo_client.start_session() as session:
            with session.start_transaction():
                # MongoDB 批量写入
                result = self.mongo_coll.insert_many(
                    mongo_docs, session=session, ordered=False
                )

                # ES 批量写入
                bulk_body = []
                for mid, doc in es_docs:
                    bulk_body.append({"index": {"_index": self.es_index, "_id": mid}})
                    bulk_body.append(doc)

                if bulk_body:
                    self.es.bulk(body=bulk_body, refresh="wait_for")

        logger.info(f"批量写入完成: {len(memories)} 条")
        return len(result.inserted_ids)

    # ========== 读操作 ==========

    def get_by_id(self, memory_id: str) -> Optional[dict]:
        """
        按 ID 获取记忆文档。
        使用 MongoDB 的 primary 读偏好保证读己之写一致性。
        """
        doc = self.mongo_coll.find_one(
            {"_id": memory_id},
            read_preference=PRIMARY,  # 需要 from pymongo import ReadPreference
        )
        return doc

    def search(
        self,
        query: str,
        agent_id: Optional[str] = None,
        memory_type: Optional[str] = None,
        tags: Optional[list[str]] = None,
        query_vector: Optional[list[float]] = None,
        limit: int = 10,
        offset: int = 0,
    ) -> list[dict]:
        """
        复合搜索：关键词 + 向量 + 过滤器。

        这是 Agent 记忆检索的核心方法。
        关键词搜索精确匹配，向量搜索语义匹配，两者互补。
        """
        # 构建 Elasticsearch 查询
        must_clauses = []

        # 关键词搜索
        if query:
            must_clauses.append({
                "multi_match": {
                    "query": query,
                    "fields": ["content^2", "tags^1.5"],
                    "type": "best_fields",
                    "operator": "or",
                }
            })

        # 过滤器
        filters = []
        if agent_id:
            filters.append({"term": {"agent_id": agent_id}})
        if memory_type:
            filters.append({"term": {"type": memory_type}})
        if tags:
            filters.append({"terms": {"tags": tags}})

        es_body = {
            "query": {
                "bool": {
                    "must": must_clauses if must_clauses else [{"match_all": {}}],
                    "filter": filters,
                }
            },
            "from": offset,
            "size": limit,
        }

        # 如果有向量，添加向量搜索
        has_vector = query_vector is not None and len(query_vector) > 0
        if has_vector:
            es_body["knn"] = {
                "field": "embedding",
                "query_vector": query_vector,
                "k": limit * 2,
                "num_candidates": 100,
            }
            # 使用 RRF 混合排名
            es_body["rank"] = {
                "rrf": {
                    "rank_constant": 60,
                    "window_size": 200,
                }
            }

        response = self.es.search(index=self.es_index, body=es_body)
        return [
            {
                "id": hit["_id"],
                "score": hit["_score"],
                "source": hit["_source"],
            }
            for hit in response["hits"]["hits"]
        ]

    def get_conversation_history(
        self,
        agent_id: str,
        user_id: str,
        limit: int = 20,
    ) -> list[dict]:
        """
        获取 Agent 与特定用户的对话历史。
        按时间倒序排列。
        """
        docs = self.mongo_coll.find(
            {"agent_id": agent_id, "type": "conversation",
             "metadata.user_id": user_id},
        ).sort("created_at", -1).limit(limit)

        return list(docs)

    def get_agent_summary(self, agent_id: str) -> dict:
        """
        获取 Agent 的记忆统计摘要。
        使用 MongoDB 聚合管道。
        """
        pipeline = [
            {"$match": {"agent_id": agent_id}},
            {"$group": {
                "_id": "$type",
                "count": {"$sum": 1},
                "avg_content_length": {"$avg": {"$strLenCP": "$content"}},
                "newest": {"$max": "$created_at"},
            }},
        ]

        results = list(self.mongo_coll.aggregate(pipeline))

        summary = {"agent_id": agent_id, "memory_types": {}}
        for r in results:
            summary["memory_types"][r["_id"]] = {
                "count": r["count"],
                "avg_length": round(r["avg_content_length"], 1),
                "newest": r["newest"],
            }

        # 添加总量统计
        total = self.mongo_coll.count_documents({"agent_id": agent_id})
        summary["total_memories"] = total

        return summary

    # ========== 删除与过期 ==========

    def delete(self, memory_id: str) -> bool:
        """从存储中删除记忆。"""
        result = self.mongo_coll.delete_one({"_id": memory_id})
        self.es.delete(index=self.es_index, id=memory_id, ignore=[404])
        return result.deleted_count > 0

    def delete_by_agent(self, agent_id: str) -> int:
        """删除某个 Agent 的所有记忆（常用于 Agent 退役）。"""
        result = self.mongo_coll.delete_many({"agent_id": agent_id})

        # ES 批量删除
        self.es.delete_by_query(
            index=self.es_index,
            body={"query": {"term": {"agent_id": agent_id}}},
        )

        return result.deleted_count

    def archive_old_memories(self, days: int = 90) -> int:
        """
        将超过指定天数的记忆归档（从主存储移动到冷存储）。
        这是记忆生命周期管理的关键功能。
        """
        cutoff = datetime.utcnow() - timedelta(days=days)

        # 找出过期记忆
        old_memories = list(self.mongo_coll.find(
            {"created_at": {"$lt": cutoff}, "type": {"$ne": "archived"}},
        ))

        if not old_memories:
            return 0

        # 标记为已归档（实际环境中可能移到 S3 / Glacier）
        ids = [m["_id"] for m in old_memories]
        self.mongo_coll.update_many(
            {"_id": {"$in": ids}},
            {"$set": {
                "type": "archived",
                "archived_at": datetime.utcnow(),
            }}
        )

        logger.info(f"归档完成: {len(old_memories)} 条记忆 (超过 {days} 天)")
        return len(old_memories)

    # ========== 变更流监听 ==========

    async def watch_changes(self, agent_id: Optional[str] = None):
        """
        监听记忆变更流。
        可以让 Agent 实时响应其他组件写入的记忆。
        """
        pipeline = []
        if agent_id:
            pipeline.append({
                "$match": {
                    "fullDocument.agent_id": agent_id,
                }
            })

        logger.info(f"开始监听记忆变更 (agent_id={agent_id})")
        async with self.mongo_coll.watch(pipeline) as stream:
            async for change in stream:
                op = change["operationType"]
                doc = change.get("fullDocument", {})
                doc_id = change["documentKey"]["_id"]

                logger.info(f"记忆变更 [{op}]: {doc_id}")

                # 分发事件（实际应用中这里可以触发 Agent 行为）
                yield {
                    "operation": op,
                    "doc_id": str(doc_id),
                    "document": doc,
                    "timestamp": change.get("clusterTime"),
                }

    # ========== 记忆清理（TTL 手动触发） ==========

    def cleanup_expired(self) -> int:
        """
        手动清理过期记忆。
        虽然 TTL 索引会自动删除，但手动触发可以即时回收空间。
        """
        result = self.mongo_coll.delete_many({
            "expire_at": {"$lte": datetime.utcnow()}
        })
        return result.deleted_count

    def close(self):
        """关闭所有连接。"""
        self.mongo_client.close()


# ========== 使用示例 ==========

async def demo_document_store():
    """
    完整的使用示例：
    展示文档存储 Agent 记忆系统的各个功能。
    """
    store = DocumentMemoryStore(
        mongo_uri="mongodb://localhost:27017",
        es_hosts=["http://localhost:9200"],
        vector_dim=768,
    )

    # 1. 写入对话记忆
    conversation = MemoryDocument(
        memory_id="conv_demo_001",
        agent_id="support_agent_v1",
        type="conversation",
        content="用户咨询退款政策，Agent 解释了 30 天退款保证",
        tags=["refund", "policy", "customer_service"],
        metadata={
            "user_id": "usr_001",
            "session_id": "sess_abc",
            "message_count": 5,
            "resolved": True,
            "intent": "refund_inquiry",
        },
        ttl_seconds=86400 * 7,  # 7 天过期
    )

    # 2. 写入知识记忆
    knowledge = MemoryDocument(
        memory_id="kb_refund_policy",
        agent_id="support_agent_v1",
        type="knowledge",
        content="退款政策：购买后 30 天内可无条件退款。"
                "退款处理时间 3-5 个工作日。特殊商品除外。",
        tags=["policy", "refund", "terms"],
        metadata={
            "source": "admin_upload",
            "version": 2,
            "author": "policy_team",
        },
        ttl_seconds=None,  # 知识永久保留
    )

    store.write(conversation)
    store.write(knowledge)
    print("✅ 记忆写入完成")

    # 3. 搜索记忆
    results = store.search(
        query="退款政策怎么处理",
        agent_id="support_agent_v1",
        limit=5,
    )
    print(f"🔍 搜索结果: {len(results)} 条")

    for r in results:
        print(f"  - [{r['source']['type']}] {r['source']['content'][:50]}...")

    # 4. 获取 Agent 摘要
    summary = store.get_agent_summary("support_agent_v1")
    print(f"📊 Agent 记忆摘要: {summary}")

    # 5. 获取对话历史
    history = store.get_conversation_history("support_agent_v1", "usr_001")
    print(f"📝 对话历史: {len(history)} 条")

    # 6. 清理过期记忆
    cleaned = store.cleanup_expired()
    print(f"🧹 清理过期记忆: {cleaned} 条")

    store.close()


if __name__ == "__main__":
    asyncio.run(demo_document_store())
```

## 场景判断

### 决策矩阵：何时使用文档存储

```text
决策因子                 文档存储    关系型DB    向量DB      KV 存储   文件系统
──────────────────────────────────────────────────────────────────────────
数据结构是否频繁变化？       ✅ 推荐     ❌ 不推荐    ✅ 尚可     ✅ 尚可     ✅ 尚可
需要复杂 JOIN 查询？        ❌ 不推荐    ✅ 推荐     ❌ 不推荐    ❌ 不推荐   ❌ 不推荐
需要全文搜索？              ✅ 推荐     ⚠️ 需扩展    ❌ 不推荐    ❌ 不推荐   ⚠️ 需工具
需要语义相似度检索？        ⚠️ 需扩展   ❌ 不推荐    ✅ 推荐     ❌ 不推荐   ❌ 不推荐
数据量 > 100GB？           ✅ 推荐     ⚠️ 尚可     ✅ 推荐     ✅ 推荐     ✅ 推荐
需要强事务？               ❌ 不推荐    ✅ 推荐     ❌ 不推荐    ⚠️ 有限    ❌ 不推荐
读延迟 < 1ms 硬要求？      ❌ 不推荐    ❌ 不推荐    ❌ 不推荐    ✅ 推荐     ❌ 不推荐
数据结构深度嵌套？          ✅ 推荐     ❌ 不推荐    ⚠️ 尚可     ❌ 不推荐   ❌ 不推荐
需要引用完整性？           ❌ 不推荐    ✅ 推荐     ❌ 不推荐    ❌ 不推荐   ❌ 不推荐
成本敏感（硬件）？          ⚠️ 适中     ✅ 低       ❌ 高       ✅ 低      ✅ 极低
```

### Agent 记忆场景推荐

```text
场景                                  推荐方案                   原因
─────────────────────────────────────────────────────────────────────────
多轮对话历史存储                      文档存储（MongoDB）       灵活 Schema、嵌入消息数组
知识库/FAQ 检索                        文档存储 + ES            全文搜索 + 向量搜索混合
Agent 推理日志审计                     文档存储（MongoDB）       自描述文档、聚合分析
用户画像与偏好                        文档存储（Couchbase）      灵活字段、低延迟读取
语义记忆检索（RAG）                   向量存储为主 + 文档存储    向量搜索 + 元数据过滤
Agent 会话缓存                        KV 存储（Redis）          亚毫秒延迟、TTL 自动过期
计费与交易记录                        关系型数据库               强事务、引用完整性
代码片段记忆     MongoDB             文档存储（带全文索引）     灵活 Schema + 全文搜索
多 Agent 协作共享记忆池                文档存储 + Change Streams  实时同步、灵活结构
Agent 冷数据归档                      文档存储 + S3             低成本、长期保存
```

### 选择指南：三个关键问题

```text
问题 1：你的记忆数据结构是固定的还是动态的？
  固定 ➔ 考虑关系型数据库
  动态 ➔ ➔ 考虑文档存储（✅）

问题 2：检索记忆的方式是什么？
  精确 ID 查询或简单过滤 ➔ KV 存储或文档存储均可
  全文关键词搜索 ➔ ➔ ➔ ➔ ➔ ➔ 文档存储 + 全文索引（✅）
  语义相似度 ➔ ➔ ➔ ➔ ➔ ➔ ➔ 向量数据库
  复杂多表关联 ➔ ➔ ➔ ➔ ➔ ➔ 关系型数据库

问题 3：你的 Agent 记忆需要多强的一致性？
  强一致性（读己之写） ➔ 文档存储 + primary 读偏好（✅）
  事务性（多文档原子性） ➔ 关系型数据库（更优）
  最终一致性（可接受短暂不一致） ➔ 文档存储 + 分片（✅ 可扩展性最好）
```

### 典型架构推荐

```text
小型 Agent 项目（单 Agent，< 1M 记忆条目）：
  ┌──────────────────────────────────┐
  │  单一 MongoDB 复制集              │
  │  3 节点，SSD 存储                 │
  │  无需 ES，使用 MongoDB 文本索引     │
  │  TTL 自动管理短期记忆过期          │
  └──────────────────────────────────┘

中型 Agent 项目（多 Agent，10M-100M 条目）：
  ┌──────────────────────────────────┐
  │  MongoDB 分片集群（3 分片）        │
  │  + Elasticsearch（3 节点）        │
  │  分片键: {agent_id: hashed}      │
  │  混合搜索: BM25 + 向量            │
  │  Change Streams 实时同步          │
  └──────────────────────────────────┘

大型 Agent 项目（数百 Agent，B+ 级条目）：
  ┌──────────────────────────────────┐
  │  MongoDB 分片集群（10+ 分片）      │
  │  + Elasticsearch 集群（10+ 节点）  │
  │  + 冷存储归档 S3/Glacier          │
  │  多层记忆：                        │
  │    热：MongoDB SSD（30 天）        │
  │    温：MongoDB HDD（90 天）        │
  │    冷：S3 Parquet（永久）          │
  │  CQRS 模式：写 MongoDB，读 ES      │
  │  Change Streams → 事件驱动         │
  └──────────────────────────────────┘
```

## 总结

文档存储是 AI Agent 记忆系统中**最 versatile** 的外部记忆方案。它在 Schema 灵活性、查询能力、扩展性和易用性之间取得了良好的平衡。

```text
文档存储记忆的核心理念：

  一个 JSON 文档 = 一个完整的记忆单元

  优势：
    ✅ 无模式 → 记忆结构随 Agent 演化
    ✅ 嵌套 → 自然表达层次化记忆
    ✅ 全文搜索 → 精确关键词检索
    ✅ 分片 → 水平扩展到 TB 级
    ✅ TTL → 自动化记忆生命周期

  劣势：
    ❌ 无 JOIN → 关联查询在应用层
    ❌ 弱一致性 → 默认最终一致
    ❌ 无引用完整性 → 孤引用需应用层处理
    ❌ 事务范围有限 → 跨分片事务昂贵

  最佳位置：
    结构化和非结构化数据的"桥梁"
    需要灵活 Schema 但对一致性要求不极端的 Agent 场景
    "大部分的 Agent 记忆需求可以用文档存储解决"
```

**选择文档存储的关键信号**：你的 Agent 记忆数据结构在频繁变化——今天存"对话消息"，明天可能还需要存"对话中的图片引用"、"用户情绪变化曲线"、"Agent 置信度评分"。如果用关系型数据库，每个新字段都需要 ALTER TABLE 迁移；用文档存储，直接写入新字段即可。

这就是文档存储对 AI Agent 的独特价值——**让记忆系统跟上 Agent 能力的快速迭代**。

---

**上一篇**: [6.6.3 数据库记忆](../6.6.3%20database-memory/learn.md)
**下一篇**: [6.6.5 外部 API 记忆](../6.6.5%20external-api-memory/learn.md)
**相关模块**: [6.6.1 知识图谱](../6.6.1%20knowledge-graph/learn.md), [6.6.3 数据库记忆](../6.6.3%20database-memory/learn.md)
