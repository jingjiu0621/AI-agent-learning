# 7.1.2 Sequential Plan — 顺序规划：步骤依赖与约束

## 简单介绍

顺序规划（Sequential Plan）处理的是子任务之间存在严格先后依赖关系的场景——某些步骤必须在前置步骤完成后才能开始。这是任务分解中最常见也最直观的约束类型，对应有向无环图中的线性链条。

## 基本原理

顺序规划的核心是**依赖关系管理**。每个子任务可能有三个关键依赖属性：

1. **输入依赖**：该步骤需要哪些前置步骤的输出
2. **资源依赖**：该步骤需要哪些资源（且这些资源可能被前置步骤占用）
3. **时序依赖**：该步骤必须在某个时间窗口内执行

依赖关系决定了步骤的执行顺序。形式化表示为：如果步骤 B 依赖于步骤 A 的输出，则 A 必须在 B 之前执行，记作 A → B。

## 背景

顺序规划是规划领域最早被研究的问题之一，从 1970 年代的 STRIPS 规划器到现代的 LLM Agent，顺序依赖管理一直是核心课题。传统的 AI 规划（如 PDDL 规划器）靠形式化推理保证顺序的正确性，但要求领域被精确建模。LLM 的语义理解能力使得在非形式化场景下推断依赖关系成为可能。

## 之前针对这个问题的做法与结果

1. **PDDL 形式化规划**：将任务用 Planning Domain Definition Language 描述，用规划器（如 FastDownward）求解。结果：保证正确性，但建模成本极高，无法适应开放域任务。
2. **硬编码流程**：if-then-else 形式的固定流程。结果：执行效率高，但无法处理未预见的变体。
3. **LLM 列出有序列表**：让 LLM 直接输出带顺序的步骤列表。结果：简单场景有效，复杂场景容易遗漏依赖或产生循环依赖。

## 核心矛盾

**并行潜力 vs 顺序安全**：串行执行最安全（每个步骤都能看到完整的前置状态），但效率最低。尝试并行可能引入竞态条件和状态不一致，但能大幅提升效率。

## 当前主流优化方向

1. **隐式依赖推断**：让 LLM 在每个步骤的描述中标注其依赖的前置步骤和输出（而不是让 LLM 直接规划顺序）。然后由代码逻辑根据依赖关系拓扑排序，保证正确性。
2. **步骤间上下文显式传递**：每个步骤的输入/输出用 JSON Schema 定义，执行引擎自动验证并传递，减少依赖遗漏。
3. **动态重新排序**：当某个步骤执行失败或输出不符合预期时，动态调整后续步骤的顺序或跳过/替代相关步骤。
4. **约束传播**：当某个步骤的实际输出与预期不符时，自动检测哪些下游步骤受影响并触发重新规划。

## 实现的最大挑战

```python
class SequentialPlanner:
    def plan(self, tasks: list[TaskNode]) -> list[ExecutionStep]:
        # Step 1: 让 LLM 标注每个任务的依赖
        annotated = self._annotate_dependencies(tasks)
        
        # Step 2: 构建依赖图
        graph = {}
        for task in annotated:
            graph[task.id] = {
                "depends_on": task.depends_on,  # 前置任务 ID 列表
                "produces": task.produces,       # 输出变量列表
                "description": task.description
            }
        
        # Step 3: 拓扑排序 - Kahn 算法
        in_degree = {tid: len(info["depends_on"]) for tid, info in graph.items()}
        queue = [tid for tid, deg in in_degree.items() if deg == 0]
        sorted_tasks = []
        
        while queue:
            current = queue.pop(0)
            sorted_tasks.append(current)
            for tid, info in graph.items():
                if current in info["depends_on"]:
                    in_degree[tid] -= 1
                    if in_degree[tid] == 0:
                        queue.append(tid)
        
        # Step 4: 检测循环依赖
        if len(sorted_tasks) != len(tasks):
            raise ValueError("检测到循环依赖，需要重新规划")
        
        return [ExecutionStep(tid, graph[tid]) for tid in sorted_tasks]
    
    def _annotate_dependencies(self, tasks: list[TaskNode]) -> list[AnnotatedTask]:
        """让 LLM 识别每个步骤的输入依赖和输出产物"""
        prompt = f"""
        分析以下任务列表，对每个任务标注：
        1. depends_on: 需要前置哪些任务的输出（用任务ID表示）
        2. produces: 本任务执行后会产生什么输出（变量名列表）
        
        任务列表:
        {tasks}
        """
        ...
```

最大挑战在于**依赖推断的准确性**：LLM 可能遗漏某些隐式依赖，或者错误地将无关步骤标注为依赖。这需要工程上的防御，如依赖验证器检查是否有任务需要的变量没有被前置任务产生。

## 能力边界和结果边界

- **能力**：能有效处理 10-20 个步骤的线性依赖链，能检测明显的循环依赖
- **边界**：不能保证所有隐式依赖都被发现；对长链（>30 步）的规划会随长度增加而质量下降
- **结果**：正确的顺序规划能将多步任务的端到端成功率提升 40-60%，优于 Agent 自由发挥

## 核心优势

顺序规划是其他所有规划模式的基础——DAG 规划本质上是在顺序规划的基础上识别并行机会，层次规划则是对顺序步骤的分组抽象。

## 工程优化方向

- 建立常见任务的依赖模式库（如"搜索→提取→总结"是一个反复出现的 3 步模式）
- 执行时实时验证依赖：如果步骤 A 实际并未产生步骤 B 需要的输出，立即暂停 B
- 结合中间结果缓存：对相同的步骤+相同输入，直接返回缓存结果跳过执行

## 适合场景

- 流水线式任务（数据 ETL、文档处理、代码编译部署）
- 有明显输入-输出链路的任务（搜索→分析→报告）
- 需要严格保证执行顺序的安全关键场景
