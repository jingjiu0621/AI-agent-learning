# 7.2.3 ReAct Variants — ReAct 变体（ReAct+CoT / ReAct+Reflexion）

## 简单介绍

ReAct 是一个基础框架而非一个僵化的模板。自 ReAct 论文发表以来，研究者社区和工程实践发展出了多种有价值的变体，将 ReAct 与其他技术结合以弥补其不足，扩展其能力边界。

## 基本原理

所有 ReAct 变体的核心思路都是"保留 Thought-Action-Observation 循环骨架，在特定环节增强特定能力"：

| 变体 | 增强点 | 核心改进 |
|------|--------|---------|
| ReAct+CoT | Thought 阶段 | 在 Thought 中加入分步推理链 |
| ReAct+Reflexion | 循环结束后 | 对整体表现进行反思并生成改进方案 |
| ReAct+Plan | Action 阶段前 | 先规划再行动，行动序列更结构化 |
| ReAct+Self-Consistency | 整体 | 多次运行取最一致答案 |
| Active ReAct | Action 选择 | 主动选择信息增益最大的行动 |

## 背景

原始 ReAct 论文（Shunyu Yao et al., 2022）在 HotpotQA 和 AlfWorld 上取得了显著效果，但实际应用中发现了一些局限：推理深度不够、没有错误纠正机制、行动选择较随机。后续工作通过融合 CoT、Reflexion、Plan-and-Execute 等技术的优点，不断扩展 ReAct 的能力边界。

## 之前针对这个问题的做法与结果

1. **纯粹 ReAct**：Thought 简短，直接指向行动。结果：效率高但推理深度不足，容易在复杂推理上出错。
2. **ReAct+CoT**：每个 Thought 都包含完整的推理链（"因为...所以..."）。结果：推理准确率提升 15-30%，但增加了每次循环的 token 开销。
3. **ReAct+Reflexion**：在 ReAct 循环结束后增加反思步骤，用 verbal reinforcement 改进后续尝试。结果：在需要多轮尝试的任务上效果显著，但增加了 20-40% 的延迟。

## 核心矛盾

**增强能力 vs 增加复杂度**：每个变体都增强了 ReAct 的某方面能力，但都带来了额外的 token 消耗、延迟或实现复杂度。选择哪个变体本质上是在"能力需要"和"成本预算"之间做权衡。

## 当前主流优化方向

1. **动态变体选择**：根据当前任务难度动态选择 ReAct 变体——简单任务用纯 ReAct，复杂任务切换到 ReAct+CoT，失败后升级到 ReAct+Reflexion。
2. **混合模式**：在同一个 ReAct 循环中，关键决策步骤用 CoT 式详细推理，常规操作步骤用简洁 Thought。
3. **轻量 Reflexion**：不在每次循环后都做完整反思，只在检测到"异常模式"时（如连续两次得到相同错误）触发反思。
4. **变体编译器**：将 ReAct 变体声明式配置（YAML/JSON），运行时动态编译为对应的 Prompt + 执行逻辑。

## 实现的最大挑战

```python
class ReActVariantSelector:
    """动态选择 ReAct 变体的调度器"""
    
    VARIANTS = {
        "standard": StandardReAct,
        "cot": ReActWithCoT,
        "reflexion": ReActWithReflexion,
        "plan": ReActWithPlanning,
    }
    
    def __init__(self, llm, config: ReActConfig):
        self.llm = llm
        self.config = config
        self.current_variant = config.default_variant
        self.attempt_count = 0
    
    async def execute(self, task: str) -> str:
        while self.attempt_count < self.config.max_attempts:
            variant = self._select_variant(task)
            agent = self.VARIANTS[variant](self.llm)
            
            result = await agent.execute(task)
            self.attempt_count += 1
            
            if result.success or self.attempt_count >= self.config.max_attempts:
                return result
            
            # 失败后升级变体
            self.current_variant = self._escalate_variant()
            # 失败信息传递给下次尝试
            task = f"{task}\n\n[Previous attempt failed: {result.error_summary}]"
    
    def _select_variant(self, task: str) -> str:
        """任务级变体选择"""
        if self.attempt_count == 0:
            # 首次尝试用默认变体
            if self._is_complex(task):
                return "cot"
            return self.config.default_variant
        # 重试时升级
        return self._escalate_variant()
    
    def _escalate_variant(self) -> str:
        """失败时升级到更强的变体"""
        escalation_path = ["standard", "cot", "plan", "reflexion"]
        current_idx = escalation_path.index(self.current_variant)
        next_idx = min(current_idx + 1, len(escalation_path) - 1)
        return escalation_path[next_idx]
```

最大挑战是**何时切换变体**：过早切换（其实再试一次就能成功）浪费了更强的变体，过晚切换（已经重复失败多次）浪费了时间和 token。需要综合判断：如果失败模式是"相同错误重复出现"，应立即切换；如果是"每次错误不同"，可能只是运气不好，可再试一次。

## 能力边界和结果边界

- **标准 ReAct**：80% 的日常任务足够，单循环 3-5 步
- **ReAct+CoT**：适合数学推理、多跳 QA、逻辑推理任务
- **ReAct+Reflexion**：适合代码调试、策略游戏、需要多轮尝试的任务
- **ReAct+Plan**：适合有明确阶段划分的复杂任务（需先收集信息再分析再决策）

## 核心优势

ReAct 变体框架让 Agent 可以从简单的基线开始，根据任务需要逐步增强，无需一开始就配置最重的方案。这种"渐进增强"的架构在实际生产系统中特别有价值。

## 工程优化方向

- 建立变体性能监控：记录每种变体在不同任务类型上的成功率和成本，数据驱动地优化选择策略
- 变体间共享轨迹：切换变体时，保留之前的 Thought-Action-Observation 历史，让新变体从已有进展上继续
- 编译而非解释：对高频任务，将选定的 ReAct 变体"编译"为固定的 prompt 模板 + 执行代码，减少运行时决策开销

## 适合场景

- 需要处理不同复杂度混合场景的生产系统
- 对 token 成本敏感但又要保障复杂任务完成率的场景
- 作为更高级 Agent 框架（如 AutoGen）的内部执行引擎
