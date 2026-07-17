# 7.5.4 HTN Planning — HTN 规划在 LLM Agent 中的应用

## 简单介绍

HTN（Hierarchical Task Network，层次任务网络）是一种经典 AI 规划方法，它将任务分解为子任务的"方法"（Methods）编码为领域知识。在 LLM Agent 中，HTN 的思想被重新激活——LLM 扮演"方法库"和"任务分解器"的角色，不需要手工编码领域知识。

## 基本原理

HTN 的核心概念：

1. **原始任务**（Primitive Task）：可以直接执行的基本操作（对应工具调用）
2. **复合任务**（Compound Task）：需要分解为子任务的高级操作
3. **方法**（Method）：如何将一个复合任务分解为子任务集合的规则
4. **任务网络**（Task Network）：任务及其约束（顺序、依赖、资源共享）构成的图

经典 HTN 的方法是手工编写的：
```
方法: 准备报告 = 
  步骤1: 搜索资料 (复合任务) 
  步骤2: 分析数据 (复合任务)
  步骤3: 撰写报告 (原始任务: write_report)
  约束: 步骤1 → 步骤2 → 步骤3
```

LLM 版本中，"方法"由 LLM 根据任务描述动态生成。

## 背景

HTN 在经典 AI 规划领域有 30+ 年历史（始于 1990 年代的 SHOP 规划器）。它在游戏 AI（如《星际争霸》的 Bot）和工业流程规划中取得了成功。但传统 HTN 需要领域专家手工编码方法库，限制了泛化能力。LLM 的语义理解能力使得"方法"可以动态生成，解决了 HTN 长期以来的知识获取瓶颈。

## 之前针对这个问题的做法与结果

1. **经典 HTN（手工方法）**：领域专家为每种任务编写分解方法。结果：精确但不可扩展，开发成本极高。
2. **经典 HTN + 机器学习**：从示例中学习方法库。结果：比手工好但需要大量标注数据。
3. **LLM 作为方法库**：LLM 根据任务描述动态生成分解方法。结果：不需要事先编码，可以零样本适应新任务，但方法的可靠性不如手工编码。

## 核心矛盾

**方法可靠性 vs 覆盖度**：手工编码的方法 100% 可靠但覆盖度低，LLM 生成的 方法覆盖度高但可能包含错误。如何结合两者——用 LLM 覆盖长尾场景，用手工方法保证核心场景的可靠性。

## 工程实现要点

```python
class LLM_HTNPlanner:
    """基于 LLM 的 HTN 规划器"""
    
    def __init__(self, llm, primitive_tools: dict):
        self.llm = llm
        self.primitives = primitive_tools  # 原始任务（可直接调用）
        self.method_cache = {}  # 高频方法的缓存
    
    async def plan(self, task: str) -> list[PrimitiveTask]:
        """从任务开始，递归分解到原始任务"""
        network = await self._decompose(task, context={})
        return self._linearize(network)
    
    async def _decompose(self, task: str, context: dict) -> TaskNetwork:
        # 检查是否已经是原始任务
        if self._is_primitive(task):
            return TaskNode(task=task, is_primitive=True)
        
        # 检查缓存
        cache_key = self._normalize(task)
        if cache_key in self.method_cache:
            method = self.method_cache[cache_key]
        else:
            # LLM 生成分解方法
            method = await self._generate_method(task, context)
            self.method_cache[cache_key] = method
        
        # 按方法递归分解子任务
        subtask_networks = []
        for subtask in method.subtasks:
            sub_network = await self._decompose(subtask, context)
            subtask_networks.append(sub_network)
        
        return TaskNode(
            task=task,
            is_primitive=False,
            method=method,
            subtasks=subtask_networks
        )
    
    async def _generate_method(self, task: str, context: dict) -> Method:
        """LLM 为复合任务生成分解方法"""
        prompt = f"""
        为以下任务生成 HTN 分解方法:
        任务: {task}
        可用原始任务: {list(self.primitives.keys())}
        
        输出 JSON:
        {{
            "subtasks": [str],  // 子任务列表
            "constraints": {{  // 约束
                "ordering": [[0, 1]],  // [i, j] 表示 i 必须在 j 前
                "dependencies": {{"子任务名": ["前置条件"]}}
            }},
            "description": str  // 分解理由
        }}
        
        注意: 子任务必须是原始任务（可直接调用）或可进一步分解的复合任务。
        """
        return Method(**json.loads(await self.llm.predict(prompt)))
```

最大挑战是**分解方法的正确性保证**：经典 HTN 的方法被设计为"sound by construction"（构建时保证正确），但 LLM 生成的方法没有这样的保证。需要额外的验证步骤来检查分解是否正确。

## 适合场景

- 需要结构化规划的复杂长链任务
- 已有部分手工编码方法、需要 LLM 补充的场景
- 规划过程需要高度可解释性（HTN 的分解树天然可解释）
