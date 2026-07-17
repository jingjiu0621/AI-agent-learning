# 有状态工具：会话、事务、游标

## 简单介绍

有状态工具在调用之间维护内部状态。典型的例子包括数据库事务（BEGIN → 操作 → COMMIT/ROLLBACK）、分页查询（第 1 页 → 第 2 页 → ...）、多步骤审批流程。有状态工具的核心挑战是如何在 LLM 的无状态调用之间管理状态。

## 有状态 vs 无状态

| 维度 | 无状态工具 | 有状态工具 |
|------|-----------|-----------|
| 每次调用 | 独立的、完整的 | 依赖之前的调用 |
| 例 | 查天气、搜网页 | BEGIN→INSERT→COMMIT |
| 重试安全 | 安全 | 不安全（要幂等） |
| 实现复杂度 | 低 | 高 |
| 状态存储 | 不需要 | 需要（内存/Redis/DB） |

## 状态管理模式

### 1. 会话 Token 模式

```python
class SessionManager:
    """基于会话 Token 的状态管理"""
    
    def __init__(self):
        self.sessions = {}  # session_id -> state
    
    def create_session(self, tool_name: str, context: dict) -> str:
        session_id = str(uuid.uuid4())
        self.sessions[session_id] = {
            "tool": tool_name,
            "state": context,
            "created_at": time.now(),
            "expires_at": time.now() + timedelta(minutes=30)
        }
        return session_id
    
    def get_session(self, session_id: str) -> dict | None:
        session = self.sessions.get(session_id)
        if session and session["expires_at"] > time.now():
            return session["state"]
        return None
```

### 2. 隐式状态模式

工具的状态隐藏在工具实现内部，LLM 不需要感知：

```python
class PaginatedSearchTool:
    """LLM 不需要知道分页的存在"""
    
    async def search(self, query: str, page: int = 1) -> dict:
        results = await self.api.search(query, page=page, size=10)
        return {
            "results": results.items,
            "total": results.total,
            "has_more": results.has_more,
            "next_page": page + 1 if results.has_more else None,
            "suggestion": "如果需要更多结果，可以再用本工具并指定 page=下一页"
        }
```

### 3. 工作流模式

```python
# 数据库事务作为有状态工作流
tools = [
    {
        "name": "begin_transaction",
        "description": "开始一个新的数据库事务，返回事务 ID"
    },
    {
        "name": "execute_in_transaction",
        "description": "在已有事务中执行 SQL",
        "parameters": {
            "properties": {
                "transaction_id": {"description": "begin_transaction 返回的事务 ID"},
                "sql": {"description": "SQL 语句"}
            }
        }
    },
    {
        "name": "commit_transaction",
        "description": "提交事务"
    },
    {
        "name": "rollback_transaction",
        "description": "回滚事务"
    }
]
```

## 状态清理

有状态工具最关键的问题：**谁来清理状态？**

```
BEGIN_TRANSACTION → 执行操作 → COMMIT (正常)
BEGIN_TRANSACTION → 执行操作 → ... (LLM 忘记了 COMMIT)
                     ← 事务一直开着，占用资源
```

**解决方案**：
1. **自动超时释放**：事务/会话在 N 分钟无操作后自动回滚
2. **LLM 引导**：在工具描述中强调"使用后必须提交/关闭"
3. **状态监控告警**：监控悬挂的事务和未关闭的会话
4. **异常回滚**：Agent 出错时自动清理所有状态

## 核心挑战

1. **LLM "忘记"提交/关闭**——这是有状态工具最常见的问题
2. **并发冲突**——多个 Agent 实例操作同一状态
3. **状态持久化**——Agent 重启后状态能否恢复
4. **状态可见性**——LLM 需要了解当前状态才能正确决策

## 最佳实践

- 有状态工具设置为**最终超时自动清理**——即使 LLM 忘记 cleanup
- 每个有状态工具提供"查询当前状态"的能力——让 LLM 可以自查
- 状态信息不要全量暴露给 LLM——只暴露决策需要的部分
- 优先级：无状态 > 隐式有状态 > 显式有状态
