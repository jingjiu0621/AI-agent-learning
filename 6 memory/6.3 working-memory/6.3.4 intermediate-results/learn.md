# 6.3.4 中间结果缓存与引用 (Intermediate Results)

## 简单介绍

中间结果缓存是工作记忆中负责存储和检索子任务输出的组件。当 Agent 执行一个子任务后，其输出结果被缓存起来，后续步骤可以通过引用标识符直接获取这些结果，而无需重新执行或从零开始推理。这类似于程序设计中函数的返回值缓存（memoization），但作用于语义任务层面。

## 基本原理

中间结果缓存的核心思想是**计算结果的复用**。Agent 在多步骤任务中经常需要反复引用之前步骤的输出：步骤 2 可能需要步骤 1 的结果，步骤 5 可能需要步骤 2 和步骤 3 的结果。如果没有中间结果缓存，Agent 只能：

1. 从对话历史中重新提取——不可靠且消耗 token
2. 重新执行该步骤——浪费计算资源
3. 依赖 LLM 自己记住——受限于上下文窗口

中间结果缓存通过以下机制解决这些问题：

```python
class IntermediateResultCache:
    def __init__(self):
        self.store = {}        # result_id -> Result
        self.dep_graph = {}    # result_id -> set[result_id]  依赖关系

    def store_result(self, result_id, content, metadata):
        self.store[result_id] = Result(content=content, metadata=metadata)

    def get_result(self, result_id):
        return self.store.get(result_id)

    def has_result(self, result_id):
        return result_id in self.store
```

## 背景与演进

| 阶段 | 形式 | 局限 |
|------|------|------|
| **无缓存** | Agent 从对话中重新提取或重新计算 | 效率极低，长上下文后完全不可用 |
| **变量绑定** | 用模板变量引用结果 (如 {{result_1}}) | 只支持简单替换，无法处理依赖 |
| **键值缓存** | 基于标识符的显式缓存 | 需要手动管理标识符和生命周期 |
| **依赖图缓存** | 自动跟踪结果间的依赖关系 | 实现复杂，但支持自动失效和增量更新 |
| **语义缓存** | 基于语义相似度的自动匹配 | 有匹配误差风险 |

## 核心矛盾

**缓存粒度 vs 缓存命中率**：缓存粒度过细（每个原子步骤的结果都缓存）会产生大量缓存条目，管理成本高；缓存粒度过粗（只缓存大块任务的最终结果）则降低了复用可能性。找到合适的粒度是中间结果缓存设计的核心挑战。

**依赖跟踪 vs 惰性失效**：当某个上游结果被更新时，所有依赖它的下游结果都需要失效（invalidation）。但精确跟踪所有依赖关系会带来 O(n^2) 的管理成本，而惰性失效（使用前才验证）又可能导致使用了过期结果。

## 主流优化方向

### 1. 引用机制

引用机制决定了 Agent 如何在工作记忆中标识和获取中间结果：

**显式引用**：通过唯一 ID 引用

```json
{
  "step_1": {"result_id": "search_weather", "content": "今日气温 25-30°C"},
  "step_2": {"result_id": "decide_activity",
              "depends_on": ["search_weather"],
              "content": "适合户外活动"}
}
```

**隐式引用**：通过语义描述匹配

```python
# Agent 表达需求："我需要之前查到的天气数据"
# 缓存系统自动匹配最相关的缓存条目
result = cache.semantic_search("天气", "气温", "weather")
```

**路径引用**：通过任务栈路径定位

```python
# 引用路径：/task_002/subtask_001/result
result = cache.get_by_path("/2/1/result")
```

### 2. 依赖跟踪

依赖图维护结果之间的"生产者-消费者"关系，这是实现自动失效和增量计算的基础：

```python
class DependencyGraph:
    def add_dependency(self, consumer_id, producer_id):
        """记录 consumer 依赖于 producer"""
        self.edges[consumer_id].add(producer_id)

    def invalidate(self, changed_result_id):
        """当某个结果变化时，递归失效所有依赖它的结果"""
        affected = set()
        def dfs(node):
            for consumer, producers in self.edges.items():
                if node in producers and consumer not in affected:
                    affected.add(consumer)
                    dfs(consumer)
        dfs(changed_result_id)
        return affected

    def get_upstream_chain(self, result_id):
        """获取某个结果的上游依赖链"""
        chain = []
        def dfs(node):
            for consumer, producers in self.edges.items():
                if consumer == node:
                    for p in producers:
                        if p not in chain:
                            chain.append(p)
                            dfs(p)
        dfs(result_id)
        return chain
```

### 3. 过期策略

中间结果的缓存时间需要精心设计：

```python
def is_result_valid(result, current_context):
    """判断缓存结果是否仍然可用"""
    # 时间戳过期
    if result.is_stale(ttl=result.metadata.get('ttl', 3600)):
        return False

    # 上下文过期
    if result.depends_on_stale_context(current_context):
        return False

    # 语义过期
    if result.metadata.get('volatile', False):
        return False  # 易变结果始终重新计算

    return True
```

### 4. 增量计算

当子任务的输入发生部分变化时，不需要重新计算整个子任务，而是只计算变化部分：

```python
def incremental_update(cache, changed_dependency_id):
    """当依赖发生变化时，增量更新受影响的结果"""
    affected = cache.dep_graph.invalidate(changed_dependency_id)
    for result_id in topological_sort(affected):
        # 只重新计算这个 result，利用已经有效的其他依赖
        result = recompute(result_id,
                          [cache.get_result(dep)
                           for dep in cache.dep_graph.edges[result_id]])
        cache.store_result(result_id, result.content, result.metadata)
```

## 产生的结果

高效的中间结果缓存带来：
- **执行效率大幅提升**：计算结果可以反复引用，避免重复执行
- **引用一致性**：所有引用的结果都是同一个来源，不会出现"同一步骤在两个地方产生不同结果"的矛盾
- **支持分支探索**：Agent 可以探索不同路径，每个分支的中间结果相互隔离，互不干扰
- **增量执行**：计划局部调整时只需要重新计算受影响的部分

## 最大挑战

**缓存一致性与 Agent 行为的不确定性之间的矛盾**：在传统软件工程中，缓存一致性有成熟的理论（如 Cache Coherence Protocol）。但在 Agent 场景中，两个严重问题使这一挑战更加困难：

1. **结果变异**：LLM 的非确定性输出意味着即使是相同的输入和相同的步骤，两次执行的结果也可能不同。缓存认为"相同的计算"，实际上可能产生有语义差异的结果。

2. **隐式依赖**：Agent 可能无意中使用了缓存结果之外的信息（如对话历史中的某个细节），导致缓存依赖追踪与实际计算之间的不一致。

## 能力边界

中间结果缓存无法解决：
1. **需要外部验证的结果**：缓存的结果可能是错误的，如果没有验证机制，错误会被传播到后续所有步骤
2. **超长中间结果**：单个子任务产生的结果可能巨大（如整个代码库的搜索），缓存这些结果可能比重新计算消耗更多资源
3. **时效性敏感的结果**：实时数据（如股价、天气）缓存的有效窗口极短，缓存经常失效
4. **副作用**：子任务可能有外部副作用（如发送了邮件），缓存的"结果"不包括对副作用的回滚

## 区别对比

| 维度 | 显式引用 | 隐式引用 | 路径引用 |
|------|---------|---------|---------|
| 精确度 | 高 | 中 | 高 |
| 灵活性 | 低 | 高 | 中 |
| 实现成本 | 低 | 高 | 中 |
| 调试难度 | 易 | 难 | 中 |
| 适用场景 | 确定性流程 | 动态探索 | 嵌套任务 |

## 核心优势

中间结果缓存最核心的价值是将 Agent 从**线性执行**提升为**有向无环图执行**——下游步骤可以自由引用任意上游步骤的结果，而不受执行顺序的严格限制。这使得 Agent 可以灵活地调整执行顺序、并行执行无依赖的任务、以及在局部修改后增量更新。

## 工程优化

### 缓存大小控制

```python
class BoundedCache:
    def __init__(self, max_entries=100, max_total_tokens=32000):
        self.max_entries = max_entries
        self.max_total_tokens = max_total_tokens

    def store_result(self, result_id, content, ...):
        # 检查总 token 数
        while self._total_tokens() + len(content) > self.max_total_tokens:
            self._evict_one()
        # 检查条目数
        while len(self.store) >= self.max_entries:
            self._evict_one()
        self.store[result_id] = ...

    def _evict_one(self):
        # LRU + 最小下游依赖数的组合策略
        candidate = min(self.store.values(),
                       key=lambda r: (r.last_access, r.downstream_count))
        del self.store[candidate.id]
```

### 序列化与恢复

```python
def serialize_cache(cache):
    """将缓存序列化为 Agent 可读的格式"""
    return {
        rid: {
            "summary": result.get_summary(),  # 结果摘要
            "size": result.token_count,
            "deps": list(cache.dep_graph.edges.get(rid, [])),
            "consumers": list(cache.dep_graph.consumers_of(rid)),
        }
        for rid, result in cache.store.items()
    }
```

## 场景判断

| 场景 | 缓存策略 | 理由 |
|------|---------|------|
| 多步数据分析 | 显式引用 + 依赖图 | 下游步骤明确引用上游结果 |
| 代码生成与调试 | 显式引用 + 增量更新 | 修改代码后只重新编译受影响部分 |
| 信息检索聚合 | 隐式引用 + LRU 淘汰 | 检索结果多，语义匹配更灵活 |
| 对话管理系统 | 短 TTL + 显式引用 | 对话状态变化快，缓存需要快速失效 |
| 复杂规划 Agent | 路径引用 + 依赖图失效 | 任务具有树状分解结构 |
