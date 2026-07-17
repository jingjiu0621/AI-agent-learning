# 5.2.6 immutable-state — 不可变状态与事件溯源模式

## 简单介绍

不可变状态（Immutable State）和事件溯源（Event Sourcing）是从数据库领域引入 Agent 的设计模式。核心思想是：不修改状态，而是"追加"状态变更事件，当前状态是所有事件的"折叠"结果。这带来了无与伦比的审计能力和时间旅行能力。

## 基本原理

### 事件溯源流

```
Agent 启动时状态为空

事件 1: TaskAssigned { task: "分析数据", id: "t1" }
事件 2: ToolCalled { tool: "query_db", params: {...} }
事件 3: ToolResult { result: "42 rows", status: "ok" }
事件 4: ThoughtGenerated { thought: "数据足够，开始分析" }

当前状态 = fold([事件1, 事件2, 事件3, 事件4], 初始状态)
```

### 不可变状态的实现

```python
from dataclasses import dataclass
from typing import List, Optional

@dataclass(frozen=True)  # frozen=True 使实例不可变
class AgentState:
    step: int
    task: Optional[str]
    results: List[str]
    
    def apply(self, event):
        """每次变更创建新实例"""
        match event:
            case TaskAssigned(task):
                return AgentState(step=0, task=task, results=[])
            case ToolResult(result):
                return AgentState(
                    step=self.step + 1,
                    task=self.task,
                    results=self.results + [result]
                )
```

## 背景与演进

| 时期 | 方案 | 特点 |
|------|------|------|
| 可变状态 | 直接修改对象字段 | 简单但不可追溯 |
| 不可变对象 | 每次修改创建新副本 | 可追溯但性能开销 |
| 事件溯源 | 状态 = 事件流折叠 | 完整审计 + 时间旅行 |
| 混合模式 | 事件溯源 + 快照优化 | 兼顾审计和性能 |

## 核心矛盾

**可审计性 vs 性能**：
- 不可变/事件溯源 → 完整历史可查 → 存储和计算成本高
- 可变状态 → 高效 → 历史丢失

## 主流优化方向

1. **快照 + 增量**：每 N 个事件打一个全量快照，恢复时从最近的快照开始+后续事件
2. **事件压缩**：合并同类型事件减少存储
3. **并行折叠**：对独立的状态维度并行计算当前值
4. **惰性计算**：只在需要时折叠事件流

## 实现挑战

1. **事件 Schema 演化**：代码升级后旧事件与新结构不兼容
2. **事件膨胀**：长时间运行的 Agent 产生数百万事件
3. **查询效率**：需要遍历事件流才能知道当前值
4. **跨 Agent 事件排序**：分布式场景的事件全局序

## 能力边界

- 事件溯源的存储需求远大于可变状态
- 不是所有状态变更都能建模为事件（如连续值）
- 事件溯源系统的调试工具不如可变状态成熟

## 核心优势

事件溯源让 Agent 状态变得 **完全可审计、可回溯、可重放**，特别适合合规性要求高的场景。

## 工程优化

1. 每 100 个事件打一个快照
2. 使用 Apache Kafka / Pulsar 作为事件存储
3. 事件 Schema 使用 Protocol Buffers / Avro 保障兼容性
4. 设置事件存储的 TTL 和归档策略

## 场景判断

适合事件溯源的场景：
- 合规要求严格的场景（金融、医疗）
- 需要详细审计轨迹的 Agent
- 需要"时间旅行"调试能力

不适合事件溯源的场景：
- 简单的一次性 Agent
- 对延迟敏感的场景
- 状态极其简单的 Agent
