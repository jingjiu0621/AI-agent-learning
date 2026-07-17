# 7.1.3 DAG Plan — DAG 规划：并行任务识别

## 简单介绍

DAG（Directed Acyclic Graph，有向无环图）规划是顺序规划的扩展——在识别出步骤间依赖关系后，将没有依赖关系的步骤安排为并行执行。这能显著减少端到端的执行时间，尤其是当某些步骤耗时较长（如网络请求、大模型推理）时。

## 基本原理

DAG 规划的核心洞察是：**不是所有步骤都需要串行**。给定一组带有依赖标注的任务，我们可以：

1. 构建依赖有向图（节点=任务，边=依赖关系）
2. 识别出"同层"节点——那些没有依赖关系的任务
3. 将同层节点分配为并行执行
4. 下一层节点必须在所有前置依赖完成后启动

形式上，如果任务 A 和 B 没有直接的或传递的依赖关系（即既没有 A→B 也没有 B→A），则 A 和 B 可以并行执行。

```
顺序执行: A → next(C) → next(E)    总时间 = 3 个单位
           B → next(D)
           
DAG 并行:    A   B   (并行层 1)
              ↘ ↙
              C   D   (并行层 2 - 等待 A,B 都完成才能开始 C)
              ↘ ↙
               E      (并行层 3)
总时间 = 3 个单位 (但前提是 A/B 耗时相近)
```

## 背景

DAG 调度在大数据领域（如 Spark、Airflow、TensorFlow 计算图）已经有非常成熟的实践。在 Agent 场景中引入 DAG 规划，本质上是将大数据领域成熟的**计算图执行范式**引入到 LLM Agent 的规划层，以解决串行 ReAct 循环效率过低的问题。

## 之前针对这个问题的做法与结果

1. **纯串行执行**：ReAct 默认的 Thought→Action→Observation→Thought 循环，每一步都等待前一步完成。结果：简单可靠，但 LLM 推理和工具调用是典型 I/O 密集型操作，串行浪费了大量等待时间。
2. **手动并行编码**：在代码层面用 asyncio.gather 或 ThreadPool 硬编码并行。结果：高效但缺乏灵活性，每增加新任务都要修改代码。
3. **固定 DAG 模板**：对特定场景（如研究助手=搜索+搜索+分析）预定义 DAG。结果：减少了重复规划开销，但无法适应任务变化。

## 核心矛盾

**并行加速 vs 复杂性爆炸**：并行越多，端到端延迟越低，但执行引擎的复杂性（错误处理、状态合并、死锁预防）显著增加。特别是一个并行分支失败时，如何处理已启动的其他分支和等待中的下游步骤。

## 当前主流优化方向

1. **LLM 生成依赖图**：让 LLM 输出结构化的依赖定义（JSON 格式的节点和边），由代码做拓扑排序和分层调度。这结合了 LLM 的语义理解和代码的正确性保证。
2. **动态并行度调整**：根据当前系统负载、API 速率限制、可用资源动态决定实际并行数（而非全量并行）。
3. **增量 DAG 调度**：不是等所有任务规划完再执行，而是规划一层执行一层——规划器在前面跑，执行器在后面追。
4. **选择性并行**：仅对预计耗时超过阈值的步骤进行并行（避免为微任务付出并行开销）。

## 实现的最大挑战

```python
class DAGScheduler:
    def build_layers(self, tasks: list[AnnotatedTask]) -> list[list[str]]:
        """
        将依赖图转化为"层"结构，每层内的任务可并行。
        使用拓扑排序 + 最长路径分层。
        """
        graph = self._build_graph(tasks)
        
        # 计算每个节点的层号（最长路径长度）
        layer_ids = {}
        for node in self._topological_sort(graph):
            if not graph[node]["depends_on"]:
                layer_ids[node] = 0
            else:
                layer_ids[node] = max(
                    layer_ids[dep] + 1 for dep in graph[node]["depends_on"]
                )
        
        # 按层分组
        layers = defaultdict(list)
        for node, lid in layer_ids.items():
            layers[lid].append(node)
        
        return [layers[i] for i in sorted(layers.keys())]
    
    async def execute_layer(self, layer: list[str], context: dict) -> dict:
        """
        执行某一层的所有任务（并行），返回合并后的上下文。
        """
        tasks = {
            tid: self._execute_single(tid, context)
            for tid in layer
        }
        
        # 等待该层全部完成（或某个失败）
        results = await asyncio.gather(*tasks.values(), return_exceptions=True)
        
        # 合并结果到上下文
        for tid, result in zip(layer, results):
            if isinstance(result, Exception):
                # 处理策略：重试 / 跳过 / 整体失败
                if self._is_critical(tid):
                    raise result
                continue
            context[tid] = result
        
        return context
    
    async def execute_all(self, tasks: list[AnnotatedTask]) -> dict:
        layers = self.build_layers(tasks)
        context = {}
        for layer in layers:
            context = await self.execute_layer(layer, context)
        return context
```

最大挑战是**错误传播**：当并行层中一个任务失败时，如何处理同层其他任务的结果？如果依赖该失败任务的下游任务需要其输出，整个 DAG 需要重新规划或修复。这需要引入"替代路径"或"优雅降级"策略。

## 能力边界和结果边界

- **能力**：将端到端执行时间降低 30-70%（取决于并行度），适合 I/O 密集型 Agent
- **边界**：对于 LLM 推理占主导的任务（每一步都需要前一步的推理结论），并行收益有限；DAG 的构建质量高度依赖 LLM 的依赖标注准确性
- **结果**：在独立子任务越多、每个子任务耗时越长的场景，加速效果越明显

## 核心优势

DAG 规划在"保持正确性"和"最大化吞吐"之间找到了工程上的最佳平衡点。相比纯串行，它大幅提升了效率；相比完全自由并行，它通过依赖约束保证了结果一致性。

## 与其他内容区别

| 特性 | 顺序规划 | DAG 规划 | 层次规划 |
|------|---------|---------|---------|
| 执行模式 | 串行 | 并行+串行 | 递归串行 |
| 依赖表达 | 全局线性 | 局部依赖 | 父-子依赖 |
| 优化目标 | 正确性 | 延迟 | 可管理性 |
| 适用规模 | 小 | 中 | 大 |

## 工程优化方向

- 实现"分层栅栏"（barrier）：每层执行结束后做一次状态一致性检查
- 引入"投机执行"：对某分支成功率较高的下游任务，在其上游未完成时就开始准备（类似 CPU 的分支预测）
- DAG 可视化模块：将依赖图和执行状态实时可视化，对调试复杂 Agent 行为至关重要

## 适合场景

- 信息收集类任务（多个搜索/查询可同时进行）
- 代码生成+测试（生成代码和写测试可并行，然后合并运行）
- 数据分析（数据清洗和特征工程的部分步骤可并行）
- 任何涉及多个独立 API 调用的场景
