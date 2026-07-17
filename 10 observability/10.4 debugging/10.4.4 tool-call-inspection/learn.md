# 10.4.4 Tool Call Inspection — 工具调用检查

## 简单介绍

Tool Call Inspection（工具调用检查）专注于审查 Agent 的工具调用行为——Agent 选择了哪个工具、传入了什么参数、工具返回了什么结果、调用的耗时和状态。这是定位"Agent 执行错"问题的最直接手段——因为大多数 Agent 故障最终都表现为**调错了工具或传错了参数**。

## 基本原理

工具调用检查的四个维度：

```python
@dataclass
class ToolCallRecord:
    # 1. 选择层：Agent 为什么选了这个工具
    thought_before_call: str        # 调用前的 Thought
    available_tools: list[str]      # 当时可用的工具列表
    tool_selected: str              # 最终选择的工具
    selection_confidence: float     # 选择置信度（如果有）

    # 2. 参数层：传入的参数
    arguments: dict                 # 完整参数
    argument_validation: str        # 参数校验结果

    # 3. 执行层：工具执行情况
    execution_start: float
    execution_end: float
    duration_ms: float
    status: str                     # success / error / timeout

    # 4. 结果层：工具返回了什么
    result: Any                     # 完整结果（或摘要）
    result_size_bytes: int
    result_truncated: bool

    # 常见问题检测
    issues: list[str] = None        # 如"参数缺失必需字段"、"结果为空"
```

常见工具调用问题模式：

```python
COMMON_ISSUES = {
    "wrong_tool": "Agent 选择了错误的工具（应该用 search 却用了 calculator）",
    "missing_param": "参数缺少必需字段",
    "hallucinated_param": "参数中包含工具定义中不存在的字段（幻觉）",
    "wrong_value": "参数值明显错误（如负数给年龄字段）",
    "empty_result": "工具返回了空结果",
    "timeout": "工具调用超时",
    "duplicate_call": "Agent 用相同参数反复调用同一个工具",
    "unused_result": "工具返回了结果但 Agent 未在后续推理中使用",
}
```

## 背景

工具调用是 Agent 系统的"执行瓶颈"——所有 Agent 的意图最终都通过工具调用来实现。在传统系统中，"调用错了方法"是一个很罕见的程序 bug；但在 Agent 系统中，由于工具选择由 LLM 决定，"选错工具"是一种常态化的调试场景。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 只看日志 | 从 JSON 日志中查找 tool_name | 缺乏上下文，不知道"为什么选了别的工具" |
| 手动重试 | 在测试中手动调用 Agent | 无法精确复现"参数错误"的 bug |
| 仅检查工具函数 | 只测试工具函数本身的正确性 | 工具函数正确但 Agent 没正确调用（参数传错） |
| 不检查参数验证 | 假设 Agent 总是能正确生成参数 | 事实上 Agent 经常编造参数值 |

## 当前主流优化方向

1. **参数模式验证**——在工具调用时实时验证参数是否符合 Schema，检测到"参数幻觉"（Agent 传了工具定义中没有的字段）时立即记录
2. **工具选择合理性分析**——对比 Agent 选择的工具和使用数据分析"最合适"的工具，评估 Agent 的工具选择质量
3. **参数值的语义验证**——不仅校验类型和格式，还在语义层面检查参数值的合理性（如 "temperature 设为 200°C" 明显不合理）
4. **工具调用序列模式分析**——分析 Agent 的工具调用序列是否合理（如"应该先登录再查数据，但 Agent 直接查了数据"）
5. **失败调用自动分类**——将失败的工具调用自动分类：外部服务故障、Agent 参数错误、权限不足等

## 实现的最大挑战

1. **参数的"错误"判断**——Agent 传了 "city=Beijing"，这在语法上完全正确，但如果用户问的是上海，这个值就是错的。判断参数是否正确需要理解用户的真实意图
2. **工具选择错误的归因**——Agent 调错了工具，是 Agent 框架的 Prompt 问题、工具描述不清晰、还是 LLM 自身的能力局限？
3. **参数幻觉的检测**——Agent 可能编造工具定义中不存在的参数（LLM 幻觉），检测需要实时比对 Tool Schema

## 能力边界

**能做什么：**
- 精确定位工具调用中的参数错误
- 检测 Agent 是否选错了工具
- 验证工具调用结果的完整性和正确性
- 发现 Agent 的"工具调用模式"异常（如反复调用同一工具）

**不能做什么：**
- 不能自动修正参数——检测到问题后仍需 retry 或 human intervention
- 不能判断"Return 的结果正确与否"——日志只记录返回了什么，不验证正确性
- 不能完全避免工具选择错误——再精确的描述也不能保证 LLM 100% 选对

## 最终工程优化

1. **Schema 实时验证**——在工具调用执行前使用 JSON Schema 验证参数，验证失败时记录详细的"参数异常报告"（缺失的字段、类型错误、枚举值不符）
2. **工具选择混淆矩阵**——统计 Agent 在"该用工具 A"的时候选择了工具 B 的频次，发现有混淆的工具对时优化工具描述
3. **工具调用的 A/B 历史对比**——对比同一 Agent 在不同版本（模型升级、Prompt 修改前后）的工具调用模式变化，发现回归问题
4. **自动 Mock 工具**——调试时自动将工具调用替换为 Mock（使用历史记录中的结果），使开发者可以在不依赖外部服务的情况下复现和调试 Agent 行为
