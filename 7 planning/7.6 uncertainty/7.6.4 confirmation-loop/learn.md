# 7.6.4 Confirmation Loop — 确认循环：不确定时主动询问

## 简单介绍

确认循环（Confirmation Loop）是 Agent 在不确定性高时主动向用户请求确认或澄清的机制。这是 Agent 从"盲目执行"走向"审慎协作"的关键能力。

## 基本原理

确认循环的触发条件：

1. **目标不确定**：任务描述有歧义，Agent 不确定用户到底要什么
2. **关键决策点**：某个决策会影响整个任务的走向，需要用户确认
3. **高风险操作**：涉及数据删除、费用支出、权限变更等
4. **信息不足**：没有足够信息做出可靠决策

## 背景

在人机交互中，"确认"是最基本的信任建立机制。然而早期的 Agent 系统倾向于"猜测用户意图"而不是"询问用户意图"。确认循环将人类纳入决策循环（Human-in-the-Loop），在 Agent 自主性和人类控制之间找到平衡。

## 之前针对这个问题的做法与结果

1. **不确认**：Agent 自行猜测并执行。结果：70% 的情况猜对，30% 的情况需要用户纠正。
2. **全覆盖确认**：每个步骤都请求确认。结果：安全但用户体验差，用户感受像在"点击下一步"。
3. **智能确认**：仅在不确定性高时请求确认。结果：最佳体验，但"何时确认"的判断需要精心设计。

## 工程实现

```python
class ConfirmationLoop:
    def __init__(self, confidence_threshold=0.7):
        self.threshold = confidence_threshold
    
    async def decide_confirmation(self, context: dict) -> bool:
        """判断是否需要请求用户确认"""
        confidence = context.get("confidence", 0.5)
        
        # 低置信度→需要确认
        if confidence < self.threshold:
            return True
        
        # 高风险操作→需要确认
        if self._is_high_risk(context):
            return True
        
        # 信息不足→需要确认
        if self._has_ambiguity(context):
            return True
        
        return False
    
    async def ask_confirmation(self, question: str, 
                                options: list[str]) -> UserChoice:
        """向用户请求确认并提供选项"""
        response = await self._prompt_user(question, options)
        return UserChoice(
            choice=response.choice,
            additional_info=response.free_text
        )
```

最大挑战是**确认疲劳**——如果 Agent 频繁请求确认，用户会感到厌烦并开始"无脑点击确认"，丧失了确认的意义。解决方案：根据用户的历史行为自适应确认频率——总是确认的用户降低频率，经常纠正的用户增加频率。

## 适合场景

- 需要人机协作的半自主 Agent
- 高风险操作（写数据库、发送消息、支付）的安全保护
- 用户偏好不明确的新任务场景
