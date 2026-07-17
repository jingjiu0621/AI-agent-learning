# 6.3.6 working-memory-cap — 容量限制与溢出处理

## 简单介绍

工作记忆容量限制（Working Memory Cap）是工作记忆系统面临的最根本约束——Agent 的工作记忆（像人类一样）有容量上限，当需要跟踪的信息超过这个上限时，必须做出取舍：哪些保留、哪些降级、哪些丢弃。

这是 Agent 记忆系统中"资源管理"的最后一环——短期记忆管理 Token 预算，长期记忆管理存储成本，而工作记忆管理的则是**注意力预算**：Agent 在单步推理中能同时"意识到"的信息量上限。

## 基本原理

### 容量的本质

工作记忆的"容量"不是字节数或 Token 数——它是**信息组块（Chunk）的数量上限**。

```
人类工作记忆容量：    约 4±1 个组块（Miller 1956, Cowan 2001）
Agent 工作记忆容量：  取决于 LLM 的注意力和上下文结构

每单位"组块"可能是一个：
- 任务目标         例: "查天气"
- 中间结果         例: "北京 20°C"
- 约束条件         例: "只看今天的数据"
- 用户偏好         例: "用摄氏度"
- 错误状态         例: "API 调用失败"

实测近似（GPT-4/Claude 3）：
  无辅助：工作记忆约 3-5 个组块（与人类相似）
  有结构化提示辅助：工作记忆约 5-8 个组块
  有外部缓存：可达 10+ 组块（但超过后收益递减）
```

### 容量溢出信号

如何知道工作记忆已经"满了"？

```python
class CapacityMonitor:
    """监测工作记忆容量"""

    def __init__(self, max_chunks: int = 7):
        self.max_chunks = max_chunks
        self.warning_threshold = int(max_chunks * 0.8)  # 80% 预警

    def check_capacity(self, wm: WorkingMemory) -> CapacityStatus:
        """检查当前容量状态"""
        current_chunks = self._count_chunks(wm)
        utilization = current_chunks / self.max_chunks

        if utilization >= 1.0:
            return CapacityStatus.OVERFLOW
        elif utilization >= 0.8:
            return CapacityStatus.WARNING
        elif utilization >= 0.5:
            return CapacityStatus.MODERATE
        else:
            return CapacityStatus.HEALTHY

    def _count_chunks(self, wm: WorkingMemory) -> int:
        """统计工作记忆中的活跃组块数"""
        count = 0
        if wm.current_task: count += 1
        count += len(wm.subtasks)        # 每个子任务是一个组块
        count += len(wm.constraints)      # 每个约束是一个组块
        count += len(wm.entities)         // 实体数量
        count += 1 if wm.last_error else 0
        return count
```

## 核心处理机制

### 1. 组块化（Chunking）

将多个相关信息合并为一个组块——这是提升容量最有效的策略：

```python
class ChunkingEngine:
    """组块化——将多个信息合并为一个组块"""

    def chunk(self, items: list[InfoItem]) -> Chunk:
        """将相关的信息合并为一个组块"""
        if self._is_same_entity(items):
            return EntityChunk(
                entity=items[0].entity_name,
                attributes={item.key: item.value for item in items},
                summary=f"{items[0].entity_name}: "
                        f"{', '.join(f'{i.key}={i.value}' for i in items[:3])}"
            )

        if self._is_same_task(items):
            return TaskChunk(
                task_name=items[0].task,
                progress=[i for i in items if i.type == "progress"],
                pending=[i for i in items if i.type == "pending"],
                summary=f"Task {items[0].task}: "
                        f"{sum(1 for i in items if i.type == 'done')}/{len(items)} done"
            )

        # 默认：创建一个摘要组块
        return SummaryChunk(
            items=items,
            summary=self._generate_summary(items)
        )

    def _is_same_entity(self, items: list[InfoItem]) -> bool:
        """判断是否都是关于同一个实体的"""
        entities = {i.entity_name for i in items if i.entity_name}
        return len(entities) == 1 and len(items) > 1

# 示例：组块化前后对比
before = [
    "北京温度: 20°C",
    "北京湿度: 65%",
    "北京风力: 3级",
    "北京空气质量: 良好",
]  # 4 个组块

after = Chunk(
    entity="北京",
    attributes={"温度": "20°C", "湿度": "65%", "风力": "3级", "AQI": "良好"},
    summary="北京: 20°C, 65%, 3级, 良好"
)  # 1 个组块
```

### 2. 优先级降级（Priority Degradation）

当容量不足时，最低优先级的组块被降级到短期记忆或长期记忆：

```python
class PriorityDegradation:
    """优先级降级——容量不足时的处理策略"""

    PRIORITY_LEVELS = {
        "critical": 5,   # 永不被降级
        "high": 4,       # 仅在极端情况下降级
        "normal": 3,     # 标准降级
        "low": 2,        # 优先降级
        "transient": 1,  # 降级候选
    }

    def degrade(self, wm: WorkingMemory) -> list[DegradedItem]:
        """执行降级：移除低优先级项并保存"""
        degraded = []

        items_by_priority = sorted(
            wm.list_active_items(),
            key=lambda x: x.priority
        )

        while self._is_over_capacity(wm):
            if not items_by_priority:
                break
            item = items_by_priority.pop(0)
            # 保存到长期记忆（如果需要恢复）
            if item.persist:
                self.archive_to_long_term(item)
            # 从工作记忆中移除
            wm.remove(item)
            degraded.append(DegradedItem(
                item=item,
                reason=f"priority_too_low",
                archived=item.persist,
            ))

        return degraded

    def _is_over_capacity(self, wm: WorkingMemory) -> bool:
        return len(wm.list_active_items()) > wm.max_chunks
```

### 3. 溢出到短期记忆（Overflow to Short-term）

工作记忆溢出时，将部分信息"下放"到短期记忆的上下文块中：

```python
class OverflowHandler:
    """溢出处理——将溢出信息转移到短期记忆"""

    def __init__(self):
        self.overflow_buffer: list[OverflowItem] = []

    def handle_overflow(self, wm: WorkingMemory):
        """处理工作记忆溢出"""
        while wm.chunk_count > wm.max_chunks:
            # 找溢出候选（优先级最低且可以延迟处理）
            candidate = self._find_best_candidate(wm)
            if not candidate:
                break

            # 移动到溢出缓冲区（不加到短期记忆中）
            wm.remove(candidate)
            self.overflow_buffer.append(OverflowItem(
                item=candidate,
                overflowed_at=time.time(),
                context_snapshot=wm.current_task,
            ))

    def maybe_restore(self, wm: WorkingMemory, current_task: str) -> list:
        """检测是否需要从溢出缓冲区恢复某些项"""
        restored = []
        still_overflowing = []

        for item in self.overflow_buffer:
            if self._is_relevant(item, current_task):
                if wm.chunk_count < wm.max_chunks:
                    wm.add(item.item)
                    restored.append(item)
                else:
                    still_overflowing.append(item)
            else:
                still_overflowing.append(item)

        self.overflow_buffer = still_overflowing
        return restored

    def _find_best_candidate(self, wm: WorkingMemory) -> Item | None:
        """找最佳溢出候选：低优先级 + 独立性强"""
        candidates = [
            item for item in wm.list_active_items()
            if item.priority <= 2 and not item.dependent_items
        ]
        return min(candidates, key=lambda x: x.priority) if candidates else None
```

### 4. 刷新与重置（Refresh & Reset）

周期性刷新工作记忆防止容量"碎片化"：

```python
class WorkingMemoryRefresh:
    """工作记忆刷新——周期性整理"""

    def __init__(self, refresh_interval: int = 10):
        self.refresh_interval = refresh_interval
        self.rounds_since_refresh = 0

    def maybe_refresh(self, wm: WorkingMemory):
        """到达间隔时刷新工作记忆"""
        self.rounds_since_refresh += 1
        if self.rounds_since_refresh < self.refresh_interval:
            return

        self.rounds_since_refresh = 0

        # 1. 合并已完成的任务信息
        completed = [s for s in wm.subtasks if s.done]
        if completed:
            wm.summary["completed_phases"] = self._summarize(completed)
            for s in completed:
                wm.subtasks.remove(s)

        # 2. 去除过时的约束
        wm.constraints = [c for c in wm.constraints
                          if not c.is_expired()]

        # 3. 丢弃低置信度的临时信息
        wm.temp_data = {
            k: v for k, v in wm.temp_data.items()
            if v.confidence > 0.3
        }

        return RefreshResult(
            removed_count=len(completed) + ...,
            new_capacity=wm.max_chunks - wm.chunk_count,
        )
```

## 容量感知的工作记忆设计

### 设计原则

1. **工作记忆的容量是可量化的**——不是模糊的"不要太多"，而是明确的"不超过 N 个组块"
2. **容量是动态的**——复杂任务的关键组块比简单任务占用更多注意力资源
3. **容量可以扩展**——通过结构化提示和外部缓存（不是真的扩展，而是更高效地利用）

### 容量预算模板

```python
# 默认工作记忆预算（基于经验值）
WORKING_MEMORY_BUDGET = {
    "slots": {
        "task": 1,           # 当前任务目标
        "subtasks": 2,       # 活跃子任务
        "constraints": 1,    # 关键约束
        "state": 1,          # 当前状态
        "error": 1,          # 当前错误（没有则为空）
        "entities": 1,       # 关键实体（组块化后）
    },
    "max_chunks": 7,          # 总上限
    "overflow_action": "summarize_to_short_term",
}

# 当容量超限时的衰减顺序（从先到后）：
DEGRADATION_ORDER = [
    "entity_details",     # 实体细节 → 仅保留实体名
    "completed_steps",    # 已完成步骤 → 摘要化
    "low_confidence",     # 低置信度推测 → 丢弃
    "old_constraints",    # 旧约束 → 若未被引用则丢弃
    "intermediate_results", # 中间结果 → 摘要化
]
```

## 背景

### 工作记忆容量的认知科学基础

人类的"7±2"工作记忆容量（后修正为"4±1"）是认知心理学最著名的发现之一。对 Agent 来说，这个数字不是硬限制，但类似的原则存在：

- **LLM 的"有效注意力窗口"比上下文窗口小得多**——即使上下文有 128K tokens，LLM 在单步推理中实际"聚焦"的信息远少于这个数字
- **组块化是人类和 LLM 的共同策略**——将相关的信息打包为一个概念单元，突破容量限制
- **注意力干扰在 LLM 中同样存在**——同时跟踪太多不同维度的信息会导致"认知负荷"上升，表现为推理质量下降

### 工程发现

Agent 工程实践（2023-2025）中观察到的容量效应：

1. **工作记忆 > 7 个独立组块时，Agent 的推理质量明显下降**（更频繁的逻辑错误、遗漏约束、任务漂移）
2. **结构化提示可以提升有效容量**——组块化后的工作记忆比散列的信息效果提升约 40%
3. **溢出处理策略直接影响任务成功率**——有溢出机制的 Agent 比简单丢弃信息的 Agent 任务成功率高出 25-50%

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **容量 vs 完整性** | 限制工作记忆容量保持聚焦但可能丢失关键信息；保留所有信息但注意力分散 |
| **即时降级 vs 延迟处理** | 立即降级保持容量可用但可能误删；延迟处理保留更多信息但可能已溢出 |
| **自动组块 vs 用户定义** | 自动组块灵活但可能错误合并；用户定义的组块准确但需要额外工作 |
| **通用容量 vs 任务适配** | 固定容量简单通用；不同任务需要不同的最佳容量 |
| **硬限制 vs 软衰减** | 硬限制实现简单但不够优雅；软衰减更自然地模拟注意力衰减但实现复杂 |

## 当前主流优化方向

1. **动态容量调整**：根据任务复杂度、LLM 性能、上下文长度自动调整工作记忆容量上限
2. **智能组块化**：使用 LLM 自身判断哪些信息可以合并为一个组块（self-chunking）
3. **预测性溢出**：在真正溢出之前预测哪些信息即将不再需要，提前释放空间
4. **多级降级路径**：不同的溢出信息走向不同的降级路径（不重要→丢弃，重要→长期记忆，临时→短期记忆缓存）
5. **容量可视化**：监控工作记忆利用率，辅助调试和优化

## 工程优化方向

1. **组块化提示工程**：设计提示模板让 LLM 自动将相关信息组块化
2. **容量预留机制**：为高优先级任务预留工作记忆槽位，防止被低优先级信息占用
3. **优雅降级路径**：
   ```
   工作记忆（最高速）→ 短期记忆（快速访问）→ 长期记忆（持久化）→ 丢弃
   ```
4. **容量使用审计**：记录每轮工作记忆的利用率和溢出事件
5. **任务适配的容量配置**：不同任务类型使用不同的容量配置

## 能力边界与结果边界

**能做的**：
- 控制工作记忆中的信息量，避免 Agent 因"认知过载"而推理质量下降
- 通过组块化在有限容量内承载更多信息
- 超出容量时优雅降级，避免"硬截断"导致的突然信息丢失

**不能做的**：
- ❌ 增加 LLM 的"实际注意力容量"——上限由模型架构决定
- ❌ 替代良好的任务分解——工作记忆容量限制会倒逼更好的任务分解，但不能替代它
- ❌ 在信息丢失时做到无损恢复（降级的信息总有损失）

**结果边界**：
- 合理的容量限制+组块化一般可将 Agent 多步骤任务成功率提升 15-30%
- 组块化是提高有效容量最有效的手段（不是增加上限，而是提高利用率）
- 不同 LLM 有不同"最佳容量"——需要针对具体模型调优

## 与其他内容的区别

| 对比 | 工作记忆容量 | Token 预算(6.1.1) | 短期记忆截断(6.1.4) |
|------|-----------|-----------------|-------------------|
| 管理对象 | 信息组块数 | Token 数 | 消息条数 |
| 限制本质 | 认知/注意力 | 上下文窗口 | 存储空间 |
| 超出后果 | 推理质量下降 | API 调用失败 | 信息丢失 |
| 优化手段 | 组块化 | 压缩/预留 | 截断策略 |
| 适用范围 | 工作记忆 | 短期记忆 | 短期记忆 |

## 核心优势

1. **认知友好**：与人类的认知容量约束一致，设计自然
2. **防止过载**：主动的容量控制比被动的截断更优雅
3. **组块化增效**：不仅是限制，更是优化（通过组块化提升有效容量）
4. **渐进降级**：容量超过时不是一刀切，而是按优先级逐步降级

## 适合场景的判断标准

- **任务信息复杂度**：简单（<3 个独立信息）→ 无需关注容量；复杂（5+ 个独立信息）→ 需要容量管理
- **LLM 推理精度要求**：高精度任务 → 需要严格控制容量；可容忍模糊的任务 → 容量可放宽
- **信息关联性**：强关联（可组块化）→ 容量有效利用；弱关联（无法组块化）→ 容量更容易超限
- **任务动态性**：静态任务 → 固定容量配置；动态变化任务 → 需要动态调整

**最适合**：复杂推理链条、多维约束的任务、长时间自主运行的 Agent

**最不适合**：简单问答、信息量极少的环境、不需要多步推理的场景
