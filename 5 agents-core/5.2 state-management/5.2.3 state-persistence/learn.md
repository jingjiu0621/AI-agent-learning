# 5.2.3 state-persistence — 状态持久化（Redis / Database / 文件）

## 简单介绍

Agent 状态需要持久化以支持会话恢复、故障转移和跨进程共享。不同的持久化方案有着不同的性能特征和适用场景。选择合适的持久化策略是 Agent 运维的基础。

## 基本原理

### 持久化层次

```
Agent 运行时内存状态
    │
    ├─→ Redis：会话状态（高性能，有 TTL）
    ├─→ Database：任务状态（结构化，可查询）
    ├─→ 文件系统：大数据状态（日志、中间文件）
    └─→ 事件日志：不可变审计轨迹（追加写入）
```

### 数据流

```python
# 伪代码：状态持久化流程
class StateManager:
    def save(self, agent_id, state):
        # 1. 序列化
        serialized = self.serialize(state)
        # 2. 写入缓存（Redis）
        self.cache.set(f"agent:{agent_id}", serialized, ttl=3600)
        # 3. 异步写入数据库
        self.queue.enqueue(self.db_save, agent_id, state)
        # 4. 追加事件日志
        self.event_log.append({
            "agent_id": agent_id,
            "event": "state_saved",
            "timestamp": now(),
            "state_summary": state.summary()
        })
```

## 背景与演进

| 时期 | 方案 | 特点 |
|------|------|------|
| 原型 | 内存存储 | 重启即失 |
| 开发 | 文件系统 JSON | 简单但不安全 |
| 生产 | Redis + DB | 高性能 + 持久化 |
| 企业 | 事件溯源 + CQRS | 可审计可恢复 |

## 存储方案对比

| 方案 | 读性能 | 写性能 | 持久性 | 查询能力 | 运维成本 |
|------|--------|--------|--------|---------|---------|
| 内存 | 极快 | 极快 | 无 | 有限 | 零 |
| Redis | 快 | 快 | 可选 | 有限 | 低 |
| PostgreSQL | 中 | 中 | 强 | 强 | 中 |
| MongoDB | 中 | 中 | 强 | 强 | 中 |
| 文件 | 慢 | 慢 | 强 | 无 | 零 |

## 核心矛盾

**性能 vs 持久性 vs 可查询性** — 三者无法同时最优：
- Redis 快但不耐久的查询
- DB 耐久的查询但慢
- 文件系统最慢但最简单

## 主流优化方向

1. **分层存储**：Redis 做主存 + DB 做冷备
2. **异步持久化**：同步写 Redis，异步刷 DB
3. **状态快照 + WAL**：定期全量快照 + 增量 WAL
4. **TTL 自动过期**：会话状态设置 TTL 自动清理

## 实现挑战

1. **序列化兼容性**：Python 对象 → JSON → 反序列化的类型丢失
2. **并发写入冲突**：同一 Agent 的并发请求导致状态覆盖
3. **部分更新**：只更新状态的一个字段而非全量替换
4. **存储成本**：大量 Agent 的长留存状态占用存储

## 能力边界

- 没有存储系统能同时满足低延迟、高持久、强一致的全部需求
- 状态序列化总是有信息损失的（如函数/类引用无法序列化）

## 核心优势

分层的持久化架构 **兼顾了运行时的性能需求和运维的数据安全需求**。

## 工程优化

1. 使用 MessagePack / Protocol Buffers 替代 JSON 提高序列化性能
2. Redis 使用 Pipeline 批量写入减少网络开销
3. 对不活跃会话设置渐进式降级（先清理详细状态，保留摘要）
4. 定期执行状态存储清理策略

## 场景判断

- 开发/测试：文件系统 JSON 足够
- 单机生产：Redis 足够
- 分布式生产：Redis Cluster + 分布式 DB
