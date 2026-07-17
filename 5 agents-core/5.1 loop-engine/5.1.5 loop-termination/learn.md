# 5.1.5 loop-termination — 终止条件：任务完成 / 最大步数 / 异常

## 简单介绍

Loop Termination 定义了 Agent 主循环何时结束。没有良好的终止机制，Agent 可能无限循环（浪费 Token）、过早终止（任务未完成），或在异常状态下继续运行（产出错误结果）。终止策略是 Agent 可靠性的核心保障。

## 基本原理

Agent 循环需要三类终止条件：

```
                  ┌─ 正常终止：任务成功完成
循环 → 检查终止 → ├─ 资源终止：达到步数/Token/时间上限
                  ├─ 异常终止：检测到不可恢复的错误
                  └─ 外部终止：用户手动中断/上游取消
```

### 检查时机

- **循环前检查**：进入新迭代前检查是否应该继续
- **行动后检查**：工具执行后检查是否需要因结果质量终止
- **观察后检查**：解析观察后判断任务是否已达目标

## 背景与演进

| 时期 | 终止机制 | 问题 |
|------|------|------|
| 早期 | 无显式终止，LLM 自行判断 | 不可靠、无限循环频繁 |
| ReAct | 固定最大步数 + LLM 输出 final 标记 | 简单有效但不够精细 |
| 现代架构 | 多重条件组合 + 置信度评分 | 复杂但可靠 |

## 核心矛盾

**任务完成度 vs 资源消耗**：
- 允许更多循环 → 更可能完成任务 → Token 消耗增加
- 提前终止 → 节省资源 → 可能任务未完成

## 主流优化方向

1. **分级终止策略**：简单任务用严格步数限制，关键任务用多重验证
2. **置信度基终止**：Agent 对当前结果的自评置信度低于阈值时继续迭代
3. **增量式终止**：如果连续 N 轮结果无改善，即使未完成也终止
4. **人类确认门**：关键决策点暂停循环，等待人类确认后再继续
5. **动态最大步数**：根据任务复杂度动态分配步数预算

## 常见的终止模式

### 基于输出的终止

```python
def should_terminate(agent_state):
    # 检测 LLM 是否输出了最终答案标记
    if agent_state.last_action == "final_answer":
        return True, "task_complete"
    # 检测结果是否满足质量阈值
    if agent_state.confidence_score > 0.95:
        return True, "high_confidence"
    return False, None
```

### 基于资源的终止

```python
def resource_exhausted(agent_state):
    if agent_state.step_count >= MAX_STEPS:
        return True, "max_steps_reached"
    if agent_state.total_tokens >= TOKEN_BUDGET:
        return True, "token_budget_exceeded"
    if agent_state.elapsed_time >= TIME_LIMIT:
        return True, "timeout"
    return False, None
```

### 异常终止

```python
def fatal_error(agent_state):
    if agent_state.critical_tool_failed:
        return True, "tool_unavailable"
    if agent_state.detected_loop_pattern:
        return True, "infinite_loop_detected"
```

## 实现挑战

1. **循环检测**：判断 Agent 是在"深入思考"还是"原地转圈"是难点
2. **部分成功处理**：任务只完成了 80%，是提交还是继续？
3. **上下游一致**：终止状态需要正确传递给调用方或下游系统
4. **用户等待体验**：长循环中用户需要感知进度而非"卡住"

## 能力边界

- 无法 100% 判断"是否真的完成了任务"——LLM 可能错误自信
- 循环检测无法区分"相似的步骤"和"重复的步骤"
- 终止决策受限于对任务目标的理解精确度

## 核心优势

完善的终止机制确保 Agent **不会无限运行、不会浪费 Token、不会在错误状态下持续产出**，是生产级部署的前提条件。

## 工程优化

1. 设置多层终止条件（硬限制 + 软限制）
2. 记录终止原因用于事后分析和优化
3. 对"接近完成"的结果实施抢救策略而非直接丢弃
4. 提供手动中断 API 供用户和上游系统调用

## 场景判断

不同场景的终止策略：
- **交互式 Agent**：用户可随时确认终止，最大步数设短（5-10步）
- **批处理 Agent**：置信度 ≥ 0.9 才终止，最大步数设长（20+）
- **关键任务 Agent**：需要人类审核确认才能终止
