# 7.1.1 Hierarchical Split — 层次分解：顶层目标 → 子任务 → 原子步骤

## 简单介绍

层次分解（Hierarchical Split）是将一个高层目标逐层细化为更小、更具体的子任务，直到每个子任务都足够简单可以直接执行。这是人类规划问题最自然的方式——从"做什么"出发，不断追问"怎么做"，直到得到可执行的步骤清单。

## 基本原理

层次分解的核心是**递归细化**：每个非原子任务都可以被分解为若干个更具体的子任务，直到所有叶子节点都是原子操作（即 Agent 可以一步完成的操作）。这种分解形成了树状结构：

```
顶层目标: 准备一份市场分析报告
├── 1. 数据收集
│   ├── 1.1 查询内部销售数据
│   ├── 1.2 爬取竞品公开信息
│   └── 1.3 收集行业报告摘要
├── 2. 数据分析
│   ├── 2.1 计算市场份额趋势
│   ├── 2.2 识别关键竞争差异
│   └── 2.3 SWOT 分析
└── 3. 报告生成
    ├── 3.1 撰写报告草稿
    ├── 3.2 制作数据可视化
    └── 3.3 格式整理与输出
```

## 背景

在 LLM 出现之前，层次分解主要见于 HTN（Hierarchical Task Network）规划系统，由人类专家手工定义分解规则和任务网络。早期的 Agent 系统（如 STRIPS、SOAR）都需要预定义的领域知识来驱动分解。LLM 的出现使得动态的、基于语义理解的自动分解成为可能，Agent 可以直接利用 LLM 的常识知识来分解未见过的新任务。

## 之前针对这个问题的做法与结果

1. **手工编写分解模板**：为每种任务类型预定义分解树。结果：可靠但无法泛化到新任务，维护成本极高。
2. **基于规则的分解**：用 if-then 规则匹配任务模式。结果：覆盖度有限，复杂任务的规则组合爆炸。
3. **LLM 一次性分解**：让 LLM 一步输出完整的层次分解。结果：简单任务效果好，但复杂任务容易遗漏步骤或产生幻觉。

## 核心矛盾

分解的**完备性**与**可执行性**之间的权衡：分解越细，每个子任务越容易执行，但分解树变得庞大，管理开销增加；分解越粗，规划越快，但子任务可能仍然太复杂，需要二次规划。

## 当前主流优化方向

1. **两阶段分解**：先用 LLM 做粗粒度分解，再对每个粗粒度任务独立进行细化。这利用了"分而治之"的策略，每次 LLM 调用只需关注较小的上下文。
2. **模板引导分解**：对常见任务类型（文档生成、数据分析、代码开发等）预定义分解模板，LLM 只需填充模板中的变量而非从头生成。
3. **交互式细化**：Agent 先输出一级分解，执行完一级子任务后，根据中间结果再决定如何进一步分解剩余部分。
4. **验证驱动的分解**：每个子任务附带可验证的完成条件（success criteria），确保分解结果是可测试的。

## 实现的最大挑战

```python
class HierarchicalDecomposer:
    def __init__(self, llm, max_depth=3):
        self.llm = llm
        self.max_depth = max_depth
    
    def decompose(self, goal: str, context: dict, depth=0) -> list[TaskNode]:
        if depth >= self.max_depth:
            return [TaskNode(goal, is_atomic=True)]
        
        # 判断当前目标是否已经是原子任务
        if self._is_atomic(goal, context):
            return [TaskNode(goal, is_atomic=True)]
        
        # LLM 分解
        subtasks = self._llm_decompose(goal, context, depth)
        
        # 递归细化
        result = []
        for subtask in subtasks:
            sub_context = {**context, "parent_goal": goal, "sibling_tasks": subtasks}
            children = self.decompose(subtask.description, sub_context, depth + 1)
            result.append(TaskNode(subtask.description, children=children))
        
        return result
    
    def _is_atomic(self, goal: str, context: dict) -> bool:
        """判断一个目标是否已经是原子任务——即能否被一步完成"""
        # 实现方式：规则 + LLM 判断
        ...
    
    def _llm_decompose(self, goal: str, context: dict, depth: int) -> list[Subtask]:
        prompt = f"""
        将以下目标分解为 3-5 个具体的子任务。
        目标: {goal}
        上下文: {context}
        当前分解深度: {depth}
        
        要求:
        - 子任务之间尽量独立
        - 每个子任务有明确的输出
        - 保持顺序合理性
        """
        ...
```

最大挑战在于**何时停止分解**：`_is_atomic` 的判断需要平衡粒度与效率，而 LLM 在这方面的判断并不稳定。

## 能力边界和结果边界

- **能力**：能够将大多数在 LLM 知识范围内的任务进行合理分解
- **边界**：对于高度专业化的领域（如量化交易策略、外科手术流程），缺乏领域知识的 LLM 分解质量无法保证
- **结果**：分解质量直接决定了后续执行的成功率，好的分解能让成功率提升 2-3 倍

## 核心优势

相比手工编码的规划器，基于 LLM 的层次分解无需预定义领域模型，可零样本适应新任务。相比扁平分解，层次结构天然支持"执行过程中动态细化"——不需一开始就规划到最细粒度。

## 工程优化方向

- 对高频任务建立分解模板缓存，避免重复 LLM 调用
- 引入"执行反馈"机制：子任务执行失败时，回退到父任务重新分解
- 使用并行分解：当多个子任务相互独立时，同时进行细化和后续执行

## 适合场景

- 目标是开放式的、创造性的（写报告、做分析、设计系统）
- 任务存在明显的"整体-部分"层次关系
- 需要将结果结构化呈现而不是扁平步骤列表
