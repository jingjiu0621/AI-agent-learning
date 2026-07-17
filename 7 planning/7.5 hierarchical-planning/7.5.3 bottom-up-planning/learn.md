# 7.5.3 Bottom-Up Planning — 自底向上从原子合成

## 简单介绍

自底向上规划（Bottom-Up Planning）是自顶向下的逆向过程——从可用的原子操作出发，将它们组合成更高级的能力模块，逐步合成为解决问题的完整方案。这种方法适合"工具驱动"的规划场景。

## 基本原理

自底向上规划的过程：

1. **盘点可用操作**：当前有哪些可用的工具和执行单元？
2. **构建能力模块**：将原子操作组合为可复用的能力单元
3. **匹配任务需求**：分析任务需要哪些能力
4. **组合方案**：用能力模块搭建解决方案

```
原子操作: search() | extract() | summarize() | translate() | format()
                ↓
能力模块: research() = search() + extract() + summarize()
          multilingual() = translate() + format()
                ↓
解决方案: research( topic ) + multilingual( research_result )
```

## 背景

自底向上规划在机器人学和自动化领域更为常见——机器人的运动规划从基本动作（前进、转弯、抓取）开始组合。在 LLM Agent 中，这种方法特别适合"已知工具集但任务不确定"的场景——先从"我能做什么"出发，再看"如何用这些能力完成任务"。

## 之前针对这个问题的做法与结果

1. **纯自顶向下**：从目标出发分解，不关心底层可用能力。结果：规划可能脱离实际——规划器假设有某个能力但实际上没有。
2. **纯自底向上**：从原子操作开始组合。结果：实际可行但可能错过高层的优化机会。
3. **双向规划**：自顶向下分解目标，自底向上组合能力，在中间层相遇。结果：最稳健但实现最复杂。

## 核心矛盾

**探索 vs 利用**：自底向上倾向于利用已有的能力模块（因为方便），可能忽略了"虽然组合复杂但更优的新方案"。需要平衡"用已有的"和"创造新的"。

## 工程实现要点

```python
class BottomUpPlanner:
    def __init__(self, tools: dict):
        self.tools = tools
        self.capabilities = {}  # 已发现的能力模块
    
    async def plan(self, task: str) -> Plan:
        # 1. 分析任务需要的能力
        required_caps = await self._analyze_required_capabilities(task)
        
        # 2. 匹配已有能力模块
        plan_steps = []
        for cap in required_caps:
            if cap in self.capabilities:
                plan_steps.append(self.capabilities[cap])
            else:
                # 3. 没有现成模块→从原子操作组合
                new_cap = await self._compose_capability(cap)
                self.capabilities[cap] = new_cap
                plan_steps.append(new_cap)
        
        return Plan(steps=plan_steps)
```

核心优势是"可行性优先"——生成的方案 100% 可执行，因为每一步都基于确实可用的工具。适合工具集有限且固定的场景。

## 适合场景

- Agent 的工具集固定、工具数量有限但功能强大的场景
- 需要保证"规划出来的每步都可执行"的高可靠性场景
- 与自顶向下规划结合，做双向一致性验证
