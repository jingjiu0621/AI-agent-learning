# 5.6.2 langgraph-checkpoint — LangGraph Checkpointer 与持久化

## 简单介绍

LangGraph 的 Checkpointer 机制是 Agent 状态的"时间机器"——它保存每一步的 State 快照，支持暂停、恢复和时间旅行调试。

## 基本原理

### Checkpoint 工作流

```
Step 1: invoke
    ↓
State 1 → 写入 Checkpointer（线程1）
    ↓
Step 2: invoke  
    ↓
State 2 → 写入 Checkpointer（线程1）
    ↓
... 暂停 ...
    ↓
从 Checkpointer 加载 State 1
    ↓
Step 2': 从不同路径继续
```

### 关键 API

```python
# 使用 MemorySaver 作为检查点存储
from langgraph.checkpoint import MemorySaver

memory = MemorySaver()
graph = builder.compile(checkpointer=memory)

# 第一次执行
config = {"configurable": {"thread_id": "thread-1"}}
result = graph.invoke({"messages": []}, config)

# 从 checkpointer 恢复
for state in graph.get_state_history(config):
    print(state)  # 可访问每一步的状态
```

### Checkpointer 接口

```python
class BaseCheckpointSaver:
    def put(self, config, checkpoint, metadata):
        """保存检查点"""
    
    def get(self, config):
        """获取最新的检查点"""
    
    def list(self, config):
        """列出所有检查点"""
```

## 核心设计

| 功能 | 说明 |
|------|------|
| Thread ID | 每个对话/任务一个独立线程 |
| Checkpoint | 每一步的完整 State 快照 |
| Metadata | 关联的元数据（时间、步骤号） |
| State History | 完整的历史状态列表 |

## 核心矛盾

**检查点粒度 vs 存储开销**：
- 每步检查点实现精确恢复
- 但长时间运行的任务积累大量检查点

## 工程优化

1. 使用 PostgresSaver 替代 MemorySaver 实现持久化
2. 定期清理旧检查点（保留最近 N 个）
3. thread_id 的命名规范（如 user_id:session_id）
4. 恢复时验证 State 的完整性

## 场景判断

- 开发调试：MemorySaver + get_state_history 做时间旅行
- 生产部署：PostgresSaver 实现持久化
- 对话 Agent：thread_id = user_id:session_id
