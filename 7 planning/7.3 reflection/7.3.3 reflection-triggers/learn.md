# 7.3.3 Reflection Triggers — 反思触发条件

## 简单介绍

反思触发条件（Reflection Triggers）决定了 Agent 何时应该停下来反思。不是每次行动后都需要反思——反思本身有成本。好的触发机制能让 Agent 在"最需要反思的时候"停下来思考，而在"一切顺利的时候"保持高效执行。

## 基本原理

反思触发的核心决策是：**现在是否需要停下来思考？** 这取决于几个因素：

1. **执行结果**：任务成功 / 失败 / 部分成功
2. **置信度**：Agent 对自己决策的确信程度
3. **异常模式**：是否检测到重复错误、异常行为
4. **外部信号**：用户反馈、系统告警、时间阈值

## 背景

在 Reflexion 论文的原始实现中，反思是在 Agent 完成任务（或任务失败）后触发的。但在实际工程中，等待任务完全结束后再反思可能太晚了——如果 Agent 在任务中途就已经走错了方向，越早发现越早纠正。因此，反思触发从"事后"向"事中"演变。

## 之前针对这个问题的做法与结果

1. **仅结束时反思**：任务成功或达到最大步数后触发。结果：简单，但错误的路径可能已经浪费了大量步骤和 token。
2. **固定步数间隔**：每 N 步触发一次反思。结果：均衡但机械——有时不需要反思也强制反思，有时需要反思还没到步数。
3. **仅失败时反思**：只有工具调用返回错误时才反思。结果：针对性强，但忽略了"结果正确但方法低效"的改进空间。

## 核心矛盾

**反思频率的权衡**：频繁反思能更早纠正错误，但每次反思消耗 token 和延迟。不频繁反思效率高，但可能耗费更多步骤在不正确的路径上。最优频率取决于任务本身。

## 当前主流优化方向

```python
class ReflectionTrigger:
    """多因素反思触发决策引擎"""
    
    def __init__(self, config: TriggerConfig):
        self.config = config
        self.step_count = 0
        self.error_count = 0
        self.recent_actions = []
    
    def should_reflect(self, context: ExecutionContext) -> bool:
        """综合多个信号判断是否需要反思"""
        triggers = []
        
        # 1. 结果触发：工具调用失败
        if context.last_observation.status == "error":
            self.error_count += 1
            triggers.append(("error", 1.0))
        
        # 2. 置信度触发：Agent 表达不确定性
        if self._detects_uncertainty(context.last_thought):
            triggers.append(("low_confidence", 0.8))
        
        # 3. 循环检测触发：检测到重复模式
        if self._detects_loop(context.recent_actions):
            triggers.append(("loop_detected", 0.9))
        
        # 4. 进度触发：超过预期步数仍未完成
        if self.step_count > self.config.expected_steps:
            triggers.append(("overdue", 0.5))
        
        # 5. 定期触发：间隔 N 步
        if self.config.periodic and self.step_count % self.config.periodic == 0:
            triggers.append(("periodic", 0.3))
        
        # 综合决策：最高信号超过阈值则触发
        max_signal = max((s[1] for s in triggers), default=0)
        should_reflect = max_signal >= self.config.threshold
        
        if should_reflect:
            self._log_trigger(triggers)
        
        self.step_count += 1
        return should_reflect
    
    def _detects_uncertainty(self, thought: str) -> bool:
        """检测 Thought 中的不确定性表达"""
        uncertainty_markers = ["不确定", "可能", "maybe", "not sure", 
                               "I think", "perhaps", "不确定哪个"],
        return any(marker in thought.lower() for marker in uncertainty_markers)
    
    def _detects_loop(self, recent_actions: list[str]) -> bool:
        """检测近期行动是否在重复循环"""
        if len(recent_actions) < 3:
            return False
        # 连续 3 次相同行动
        return len(set(recent_actions[-3:])) == 1
    
    def _log_trigger(self, triggers: list[tuple[str, float]]):
        """记录触发事件，用于分析和调优"""
        logger.info(f"反思触发: {triggers}")
```

## 实现的最大挑战

**避免"反射太频繁"和"反射太迟"两个极端**：反射太频繁让 Agent 陷入"过度思考"——反复分析但执行很少；反射太迟让 Agent 在错误路径上浪费大量步骤。工程上可以设置"最小间隔"（两次反思之间至少执行 N 步）和"最大间隔"（每 N 步强制反思一次）。

## 能力边界和结果边界

- **能力**：能覆盖 80% 以上的"需要反思"的场景（错误、循环、不确定、超时）
- **边界**：无法检测"结果正确但推理链有隐含错误"的情况——如果 Agent 偶然蒙对了，触发机制可能错误地认为一切顺利
- **结果**：好的触发机制能将反思的有效性和效率平衡到位——反思次数减少 40%，但捕获的问题数不变

## 核心优势

相比固定策略，多因素触发引擎能自适应不同任务的需求：简单任务几乎不触发反思，复杂任务在关键节点触发反思，错误场景密集触发反思。这实现了"在需要反思的时候反思，不需要的时候不浪费"。

## 工程优化方向

- 触发信号的历史分析：统计哪些信号在实际中产生了最有价值的反思，调优信号权重
- 任务类型自适应：对数据分析类任务，增加"数据异常"检测触发；对代码类任务，增加"编译错误"触发
- 用户可配置的触发策略：提供"谨慎模式"（低阈值，频繁反思）和"高效模式"（高阈值，减少反思）
- 触发-反思-行动的闭环收益追踪：记录每次反思是否带来了可量化的改进，用于持续优化触发策略

## 适合场景

- 任何使用了反思机制的 Agent 系统
- 需要在执行效率和安全之间做权衡的生产场景
- 任务类型多样、无法单一固定触发策略的通用 Agent
