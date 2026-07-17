# 7.6.3 Fallback Strategies — 回退策略

## 简单介绍

回退策略（Fallback Strategies）是 Agent 在不确定性高或执行失败时采取的安全措施。好的回退策略让 Agent 在无法完美完成任务时，仍然能提供有价值的服务或至少不造成伤害。

## 基本原理

回退策略的层次（从优到次）：

1. **自动修复**：重试、参数调整、工具替换（不需要人参与）
2. **降级服务**：提供部分结果而非完整结果（如只给出数据不给出分析）
3. **替代方案**：用不同的方法达到相同或类似的目的
4. **主动求助**：向用户或上级 Agent 请求帮助
5. **安全终止**：确认无法完成，报告失败并说明原因

## 背景

回退策略是工程系统（从网络协议到微服务）的标配能力。在 Agent 系统中引入回退概念，本质是将"所有请求都必须成功"的假设替换为"为失败做好预案"的工程思维。

## 之前针对这个问题的做法与结果

1. **无回退**：失败直接报错。结果：不友好，用户体验差。
2. **仅重试**：失败就重试，直到成功或配额用完。结果：对临时故障有效，对系统性错误无效。
3. **固定回退链**：预定义失败→重试→降级→求助的链条。结果：结构清晰但缺乏灵活性。

## 核心矛盾

**回退的成本 vs 回退的价值**：某些回退（如人工求助）成本很高——但比让用户得到一个错误答案更好。关键在于判断什么时候值得启动成本更高的回退。

## 工程实现要点

```python
class FallbackManager:
    async def execute_with_fallback(self, task: Callable, 
                                     fallback_chain: list[Fallback]) -> Result:
        errors = []
        
        for attempt, fallback in enumerate(fallback_chain):
            try:
                result = await fallback.fn(task)
                if self._is_acceptable(result):
                    return result
                errors.append(f"Attempt {attempt}: unacceptable result")
            except Exception as e:
                errors.append(f"Attempt {attempt}: {str(e)}")
            
            # 最后一次尝试失败才返回错误
            if attempt == len(fallback_chain) - 1:
                self._escalate_to_human(task, errors)
                return Result(
                    success=False,
                    errors=errors,
                    message="已升级到人工处理"
                )
        
        return Result(success=True)
```

核心机制是"回退链"——一个有序的备选方案列表，按成本递增排列。执行时，从最低成本的方案开始尝试，失败则自动移动到下一个。

## 适合场景

- 生产环境的 Agent 系统
- 对不可靠外部服务的调用
- 任何不希望直接向用户展示错误信息的场景
