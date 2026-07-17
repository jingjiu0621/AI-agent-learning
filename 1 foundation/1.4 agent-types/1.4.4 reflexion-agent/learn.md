# Reflexion Agent：带自我反思的迭代改进

## 简单介绍

Reflexion 由 Shinn 等人在 2023 年提出，在 ReAct 的基础上增加了**自我反思**能力——Agent 不仅执行任务，还回顾自己的执行过程，识别错误，并将经验保存到记忆中作为未来行动的参考。这使得 Agent 可以从失败中学习，而无需重新训练模型。

## 基本原理

### Reflexion 循环

```
传统 ReAct 循环：
Thought → Action → Observation → Thought → ...

Reflexion 循环：
执行 → 评估执行结果 → 反思(总结经验) → 保存到记忆 → 下次改进
```

### 核心组件

| 组件 | 功能 | 实现 |
|------|------|------|
| Actor | 执行任务（ReAct Agent） | LLM + Prompt + 工具 |
| Evaluator | 评估任务执行质量 | LLM 自我评估或外部评估 |
| 记忆 | 保存反思结果 | 向量数据库或语义记忆 |
| 反思触发 | 何时触发反思 | 失败/低分/周期性 |

### 反思深度

| 深度 | 内容 | 示例 |
|------|------|------|
| 表层 | 纠正具体错误 | "不应该用 Search 工具，应该用 Calculator" |
| 中层 | 改善策略 | "在处理数学问题时先确认数值类型" |
| 深层 | 重构方法 | "对于这类问题，我应该先规划步骤再执行" |

## 核心价值

1. **无需重训练**：可以在推理时持续改进行为
2. **错误复用**：一次错误 → 经验保存 → 永不再犯
3. **可定制评估**：可以根据不同标准评估 Agent 行为

## 限制

1. **反思质量取决于 LLM 能力**：LLM 可能不能正确识别自己的错误
2. **Token 开销大**：反思 + 再执行消耗大量 Token
3. **过度反思**：Agent 可能陷入"反思循环"，反复修改但无法前进
4. **经验泛化困难**：反思经验可能过于具体，难以应用到不同场景

## 工程实现

```python
def reflexion_loop(task, max_attempts=3):
    memory = []
    for attempt in range(max_attempts):
        # 执行任务（带记忆）
        result = execute_with_memory(task, memory)
        
        # 评估结果
        score = evaluator(result)
        
        if score >= threshold:
            return result
        
        # 反思
        reflection = reflect(task, result, memory)
        memory.append(reflection)
    
    return best_result
```

## 优化方向

1. **选择性反思**：只在关键失败节点反思（而非每次都反思）
2. **经验结构化**：将反思经验组织为"场景 → 策略"映射
3. **衰减机制**：旧经验自动降低权重，避免过时知识干扰
4. **外部验证**：使用外部验证器替代 LLM 自我评估
