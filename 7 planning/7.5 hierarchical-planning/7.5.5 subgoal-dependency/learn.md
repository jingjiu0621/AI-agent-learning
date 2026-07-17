# 7.5.5 Subgoal Dependency — 子目标依赖管理

## 简单介绍

子目标依赖管理（Subgoal Dependency Management）处理的是分层规划中不同子目标之间的关系。在分层结构中，子目标之间可能存在依赖、冲突、优先级等关系，需要系统化的管理机制。

## 基本原理

子目标之间的关系类型：

1. **顺序依赖**（Sequential）：子目标 A 完成才能开始 B（A→B）
2. **资源共享**（Resource Sharing）：多个子目标竞争同一有限资源（如同一个 API 的配额）
3. **信息依赖**（Information）：子目标 B 需要 A 的执行结果作为输入
4. **互斥**（Mutual Exclusion）：子目标 A 和 B 不能同时实现（需要折中）
5. **条件依赖**（Conditional）：子目标 B 只在 A 的某个特定结果下才需要

## 背景

在经典 AI 规划中，子目标依赖通过"因果链"（Causal Link）和"威胁"（Threat）的概念形式化处理。在分层 LLM Agent 中，依赖关系主要通过自然语言描述的"前置条件"和"后置条件"来表达和检查。

## 之前针对这个问题的做法与结果

1. **隐式依赖管理**：将依赖关系编码在步骤描述中，让执行器通过阅读描述来理解依赖。结果：灵活但可能遗漏依赖。
2. **显式依赖标注**：每个步骤标注 depends_on 列表。结果：明确但增加了规划阶段的工作量。
3. **自动依赖检测**：规划器自动识别和标注依赖。结果：减少人工但 LLM 的依赖检测可能不完整。

## 核心矛盾

**依赖的精确性 vs 管理成本**：精确标注所有依赖关系需要深入分析每个步骤的输入输出，成本高；粗略标注可能遗漏关键依赖，导致执行时失败。

## 工程实现

```python
class SubgoalDependencyManager:
    def manage(self, subgoals: list[Subgoal]) -> DependencyGraph:
        graph = DependencyGraph()
        
        for sg in subgoals:
            # 分析每个子目标的输入输出
            inputs, outputs = self._analyze_io(sg)
            
            # 检测信息依赖
            for other in subgoals:
                if sg == other:
                    continue
                # 如果 sg 需要 other 的输出
                shared = inputs & other.outputs
                if shared:
                    graph.add_dependency(sg.id, other.id, shared)
        
        # 检测循环依赖
        if graph.has_cycle():
            # 尝试打破循环
            self._break_cycle(graph)
        
        return graph
```

最大挑战是**跨层依赖追踪**——上层子目标的依赖可能在下层执行时才暴露。实践中可以通过"依赖传播"机制解决：检测到下层的某个步骤依赖某资源时，自动将该依赖传播到上层对应子目标。

## 适合场景

- 子目标数量多（>5）且关系复杂的规划问题
- 分布式执行环境中需要精确调度资源
- 作为分层规划中确保一致性的基础设施
