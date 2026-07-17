# 7.2.2 Action-Observation — 行动-观察循环详解

## 简单介绍

行动-观察（Action-Observation）是 ReAct 循环中连接 Agent 内部推理和外部世界的桥梁。Agent 通过行动改变环境，通过观察感知变化，由此形成"思考→行动→观察→再思考"的闭环。

## 基本原理

Action-Observation 循环的核心逻辑：

1. **Action（行动）**：Agent 从可用工具中选择一个并调用它。行动可以是工具调用、内部计算或者向用户发送消息。
2. **Execution（执行）**：行动被实际执行——API 被调用、代码被运行、文件被读写。
3. **Observation（观察）**：执行结果被带回 Agent 的上下文，成为下一轮思考的输入。

```
循环单元:
  Thought: "我需要查找最新的销售数据"
    ↓
  Action: query_database(table="sales", period="2024-Q1")
    ↓
  [执行: 数据库查询]
    ↓
  Observation: "{'total_revenue': 1.2M, 'growth': 15%}"
    ↓
  Thought: "销售额增长15%，表现良好。接下来分析增长原因..."
```

## 背景

在 ReAct 提出之前，LLM 的工具使用主要有两种模式：一种是"先完整规划再执行"（Plan-then-Execute），无法根据中间结果动态调整；另一种是"单轮工具调用"，限定了 Agent 只能调用一次工具。ReAct 的 Action-Observation 循环首次实现了"多次工具调用 + 中间结果反馈"的动态交互模式。

## 之前针对这个问题的做法与结果

1. **单轮工具调用**：Agent 只调用一次工具，基于结果直接回答。结果：简单场景够用，但复杂任务需要多步工具链时无能为力。
2. **预定义工具链**：提前写好固定的工具调用序列（如搜索→提取→分析）。结果：对预期内的场景高效，但无法处理意外情况。
3. **无 Action 的纯推理**：Agent 仅靠内部知识回答，不使用外部工具。结果：知识截止、容易产生幻觉。

## 核心矛盾

**行动的粒度 vs 循环的效率**：每次 Action-Observation 循环都有固定开销（LLM 调用 + 上下文追加）。行动粒度过细导致循环次数过多、token 浪费；粒度过粗又无法充分利用观察结果来指导下一步行动。

## 当前主流优化方向

1. **批量行动**：在单次循环中允许 Agent 执行多个独立行动（如同时搜索多个关键词），减少循环次数。
2. **观察缓存**：如果某个工具+参数组合的结果已被查询过，直接返回缓存结果，跳过重复执行。
3. **投机执行**：根据历史模式预测 Agent 下一步可能需要的工具调用，提前准备结果。
4. **增量观察**：对于耗时操作（如长文档分析），返回中间进度观察而非等完全完成。

## 实现的最大挑战

```python
class ActionObservationLoop:
    def __init__(self, tools: dict, max_iterations=10):
        self.tools = tools
        self.max_iterations = max_iterations
        self.history = []
    
    async def execute_action(self, action: Action) -> Observation:
        """执行单个行动并返回观察结果"""
        if action.name not in self.tools:
            return Observation(
                status="error",
                content=f"未找到工具: {action.name}",
                metadata={"error_type": "tool_not_found"}
            )
        
        tool = self.tools[action.name]
        try:
            # 工具执行
            result = await tool.execute(action.parameters)
            
            # 构造观察
            observation = Observation(
                status="success",
                content=result,
                metadata={
                    "tool": action.name,
                    "duration_ms": result.duration_ms,
                    "tokens_used": getattr(result, 'tokens', None)
                }
            )
        except TimeoutError:
            observation = Observation(
                status="timeout",
                content="工具执行超时",
                metadata={"tool": action.name, "timeout_s": tool.timeout}
            )
        except Exception as e:
            observation = Observation(
                status="error",
                content=f"执行错误: {str(e)}",
                metadata={"tool": action.name, "error_type": type(e).__name__}
            )
        
        return observation
    
    def construct_observation_prompt(self, observation: Observation) -> str:
        """将观察结果格式化为 ReAct 循环中的 Observation 文本"""
        if observation.status == "success":
            # 对长结果做截断，避免撑爆上下文
            truncated = self._truncate(observation.content, max_chars=2000)
            return f"Observation: {truncated}"
        elif observation.status == "timeout":
            return f"Observation: 工具 {observation.metadata['tool']} 执行超时，请尝试其他方法或重试。"
        else:
            return f"Observation: 执行出错 ({observation.metadata.get('error_type', 'unknown')}): {observation.content}"
```

最大挑战在于**错误观察的恢复**：当观察结果是错误时，Agent 能否从中提取有用信息并调整策略，而不是陷入"重试同一个操作→得到同一个错误"的死循环。这需要观察结果包含足够丰富的错误上下文，以及 Agent 被 prompt 明确引导去"分析错误原因，而非盲目重试"。

## 能力边界和结果边界

- **能力**：支持任意数量、任意类型的工具调用链，能处理成功、错误、超时等各种结果
- **边界**：观察结果的长度受限于上下文窗口；无法处理需要 Agent 被动等待的异步事件（如"等某文件被上传"）
- **结果设计**：观察结果应该包含"what happened + why"，而不是原始的工具返回值——让 Agent 理解结果而不只是看到结果

## 核心优势

Action-Observation 循环实现了 Agent 与环境的持续交互。相比无工具的纯推理，它让 Agent 获取实时信息；相比固定工具链，它让 Agent 能根据中间结果动态调整后续行动。

## 工程优化

- 对大型观察结果（如网页内容、长文档）实现流式注入——Agent 先看到开头部分就开始思考，后续内容追加
- 建立观察结果的 "schema"——不同工具的输出结果统一为结构化格式，方便 Agent 解析
- 设计"观察太短"的降级策略：如果工具返回空结果，Agent 应该尝试其他工具或询问用户，而不是直接输出"没找到"

## 适合场景

- 所有需要与外部世界交互的 Agent（即绝大部分 Agent）
- 工具调用链可能根据中间结果动态变化的任务
- 需要处理错误和异常情况的生产环境
