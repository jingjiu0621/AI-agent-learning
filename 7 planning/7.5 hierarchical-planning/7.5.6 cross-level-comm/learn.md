# 7.5.6 Cross-Level Communication — 跨层级通信协调

## 简单介绍

跨层级通信（Cross-Level Communication）是分层规划中各层之间信息交换的机制。没有良好的通信机制，分层规划的各层就会变成"信息孤岛"——上层不知道下层的实际进展，下层不了解上层的战略意图。

## 基本原理

跨层通信的类型：

| 方向 | 内容 | 触发条件 |
|------|------|---------|
| 自上而下 | 目标、约束、优先级 | 规划/重新规划时 |
| 自下而上 | 进度、问题、建议 | 步骤完成/异常时 |
| 横向 | 资源协调、依赖同步 | 多个子目标并行时 |

通信原则：
- 上层告诉下层"做什么"和"为什么"，不告诉"怎么做"
- 下层告诉上层"做到哪了"和"遇到了什么"，不发送全部执行日志
- 异常报告需要包含"发生了什么 + 我尝试了什么 + 需要上层做什么"

## 背景

在软件工程中，分层架构的层间通信通常通过定义明确的接口（API）实现。在 Agent 的分层规划中，层间通信更类似"组织沟通"——上下级之间的信息同步、汇报和指令下达。这种类比有助于设计更自然的通信协议。

## 之前针对这个问题的做法与结果

1. **无通信**：上层规划后完全交给下层，不追踪进展。结果：上层对执行情况一无所知，无法在发生偏差时纠正。
2. **全量通信**：下层将每一步的执行细节都上报给上层。结果：信息充分但通信成本高，上层被细节淹没。
3. **按需通信**：只在关键节点（阶段完成、失败、请求决策时）通信。结果：当前最实用方案。

## 核心矛盾

**通信带宽 vs 信息充分性**：通信过多则上层负担重、token 消耗大；通信过少则上层不能及时掌握情况做出正确决策。

## 工程实现要点

```python
class CrossLevelComm:
    """层间通信管理器"""
    
    def __init__(self):
        self.report_threshold = ReportThreshold.SIGNIFICANT
    
    def upward_report(self, status: ExecutionStatus) -> Report:
        """下层向上层报告"""
        if self.report_threshold == ReportThreshold.ALL:
            return FullReport(status)
        elif self.report_threshold == ReportThreshold.SIGNIFICANT:
            # 只报告关键事件
            events = []
            for step in status.completed:
                if step.is_milestone or step.has_deviation:
                    events.append(step)
            return SignificantEventReport(events)
        elif self.report_threshold == ReportThreshold.ERROR_ONLY:
            return ErrorReport(status.errors)
    
    def downward_instruction(self, goal: str, context: dict) -> Instruction:
        """上层向下层下达指令"""
        # 只传递"什么"和"为什么"，不传递"怎么做"
        return Instruction(
            objective=goal,
            rationale=context.get("rationale", ""),
            constraints=context.get("constraints", []),
            success_criteria=context.get("success_criteria", []),
            available_resources=context.get("resources", {}),
            # 不包含具体执行方法
        )
```

最大挑战是"关键事件"的判定——下层在执行时，如何判断哪些事件需要上报、哪些可以忽略？一般原则：影响后续决策的（如关键步骤成功/失败）、需要上层协调的（如资源不足）、改变任务假设的（如发现新的关键信息）。

## 适合场景

- 多 Agent 系统中的层级协作
- 长周期任务中需要阶段性汇报的场景
- 分层 Plan-and-Execute 中的协调基础设施
