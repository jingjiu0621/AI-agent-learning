# 7.5.1 Abstraction Levels — 抽象层次：战略 → 战术 → 执行

## 简单介绍

抽象层次（Abstraction Levels）是分层规划的基础——将规划的"粒度"划分为不同层次，每层对应不同级别的抽象。经典的"战略→战术→执行"三层模型是应用最广泛的分层方式。

## 基本原理

三层抽象模型：

| 层次 | 时间跨度 | 粒度 | 关注点 | 使用的 LLM |
|------|---------|------|--------|-----------|
| 战略（Strategy） | 长期 | 阶段级 | 目标、方向、限制 | 最强模型 + 低 temperature |
| 战术（Tactic） | 中期 | 步骤链级 | 方法、资源、顺序 | 中等模型 |
| 执行（Execution） | 短期 | 单步级 | 参数、格式、工具调用 | 可换小模型 |

```
战略层: "在 Q4 完成用户增长目标"
  └── 子目标1: "提升新用户注册转化率"
  └── 子目标2: "降低老用户流失率"
  
战术层 (针对子目标1):
  ├── 步骤1: 分析当前注册漏斗数据
  ├── 步骤2: 确定转化率最大的优化点
  ├── 步骤3: 设计 A/B 测试方案
  └── 步骤4: 实施并监控 A/B 测试
  
执行层 (针对战术步骤1):
  ├── 调用 analytics API 获取最近 30 天数据
  ├── 用 Python 计算各步骤转化率
  └── 生成漏斗可视化报告
```

## 背景

抽象层次的概念来自软件工程（分层架构）、认知科学（人类问题解决的分层策略）和经典 AI（抽象层次规划）。Simon (1969) 的"The Sciences of the Artificial" 中已经指出：复杂系统的设计本质上是建立抽象层次。

## 之前针对这个问题的做法与结果

1. **扁平规划**：所有步骤在同一抽象层次。结果：简单任务可行，但面对复杂任务时规划器会被细节淹没。
2. **两层抽象**：战略+执行。结果：好于扁平，但战略到执行的跨度太大，容易出现"战略性正确、战术性错误"。
3. **三层抽象**：战略+战术+执行。结果：当前实践中最有效的折中，每层的抽象跨度适中。

## 核心矛盾

**信息丢失**：每经过一层抽象，从上层传递到下层的"上下文"会丢失一些信息。抽象层次越多，信息丢失越严重，但每层的关注点越聚焦。

## 工程实现

```python
class AbstractionPlanner:
    """三层抽象规划器"""
    
    async def hierarchical_plan(self, task: str) -> ExecutionPlan:
        # 战略层：确定大方向
        strategy = await self._strategic_analysis(task)
        
        # 战术层：为每个战略目标制定战术
        tactical_plans = []
        for goal in strategy["goals"]:
            tactic = await self._tactic_planning(goal, strategy["constraints"])
            tactical_plans.append(tactic)
        
        # 执行层：为每个战术步骤生成具体执行计划
        execution_steps = []
        for tactic in tactical_plans:
            for step in tactic["steps"]:
                exec_detail = await self._execution_detail(step)
                execution_steps.append(exec_detail)
        
        return ExecutionPlan(
            strategy=strategy,
            tactics=tactical_plans,
            execution=execution_steps
        )
```

最大挑战是**层间抽象对齐**——战略层的"提升转化率"在战术层可能被理解为"修改页面设计"或"优化广告投放"或"改善产品定价"，不同理解会导致完全不同的执行路径。需要通过明确的"目标描述规范"来确保一致性。

## 适合场景

- 任何需要"先想清楚做什么，再想怎么做，最后动手做"的复杂任务
- 多 Agent 系统中不同角色需要不同抽象级别的信息
- 需要向人类 Stakeholder 展示"高层规划"而不仅仅是一堆步骤
