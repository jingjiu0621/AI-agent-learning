# 3.2.3 Prompt Chaining — Prompt 链式组装与管道

## 简单介绍

Prompt Chaining（提示链）是一种将复杂任务拆解为多个子任务，每个子任务使用专门的 Prompt 处理，并将前一步的输出作为后一步的输入的技术。这是一种"分而治之"的 Agent 架构模式。与单次调用中完成所有工作不同，Chain 允许开发者对每个步骤独立设计 Prompt、控制逻辑和错误处理，从而构建可靠的复杂工作流。

## 基本原理

每个 Prompt Chain 是一个有向无环图（DAG），其中每个节点是一个"处理步骤"，每条边是"数据流"。节点可以是 LLM 调用（使用特定 Prompt）、工具函数调用、条件判断或数据变换。链的执行引擎遍历这个图，将上一步的输出解析并注入到下一步的 Prompt 中。

```python
class PromptChain:
    def __init__(self):
        self.steps = []
        self.context = {}

    def add_step(self, name, prompt_template, parser=None):
        """parser: 从 LLM 输出中提取结构化数据的函数"""
        self.steps.append({
            "name": name,
            "template": prompt_template,
            "parser": parser or (lambda x: x),
        })

    async def run(self, initial_input):
        self.context["user_input"] = initial_input
        for step in self.steps:
            prompt = step["template"].render(**self.context)
            raw_output = await llm_call(prompt)
            parsed = step["parser"](raw_output)
            self.context[step["name"]] = parsed
            self.context["last_output"] = raw_output
        return self.context
```

## 背景

最初的 Agent 设计是一个单一 Prompt + 工具调用循环。这种方式在简单任务上表现良好，但随着任务复杂度增加，暴露出多个问题：单次 Prompt 需要"一次性"理解整个任务流程；复杂指令之间的相互干扰；错误在一个步骤发生后难以精确回退；难以对流程中的中间结果进行人工审核。

## 之前针对这个问题的做法与结果

**做法一：大而全的单一 Prompt** — 将整个任务的指令写在一个 Prompt 中，要求 LLM 一次性完成。优点是延迟最低（一次调用），缺点是：Prompt 长度失控；LLM 容易遗漏中间步骤；错误定位困难；无法在步骤之间注入人工反馈。

**做法二：ReAct 循环** — Agent 在"思考-行动-观察"的循环中完成复杂任务。比单一 Prompt 灵活，但本质上仍然是 LLM 自行控制流程，开发者无法对中间步骤施加精确控制。当 Agent 在循环中"跑偏"时，很难纠正。

**做法三：硬编码状态机** — 开发者用代码显式定义每一步和状态转换。控制力最强，但开发成本高，灵活性低，且每步的判断逻辑（应当转 FSM 的哪个状态）仍然需要 LLM 参与。

## 核心矛盾

**精确控制与模型灵活性之间的矛盾**。Chain 给了开发者对每个步骤的精确控制，但代价是增加了延迟（多次 LLM 调用）和开发复杂度。每一步都需要精心设计 Prompt 和输出解析器，且步骤之间存在数据依赖风险——上一步的错误格式会级联传播到后续步骤。过度设计 Chain 会让系统僵化，失去 LLM 的灵活应变能力。

## 当前主流优化方向

1. **并行链（Parallel Chain/Fan-out）** — 多个独立的子任务并行执行，最后合并结果。适用于信息检索、多角度分析等场景。例如同时检索知识库和搜索引擎，合并结果后输入下一步决策 Prompt。

2. **条件链（Conditional Chain）** — 根据上一步的输出结果动态决定下一步走哪个分支。通过条件判断，链可以自适应地选择不同的处理路径，而不是固定执行所有步骤。

3. **循环链（Iterative Chain）** — 允许链中的某个节点重复执行，直到满足终止条件。例如"生成代码 → 测试 → 修复错误 → 重新测试"的循环，直到全部测试通过。

4. **门控链（Gate Chain）** — 在关键步骤之间插入"门控"节点，由 LLM 或规则判断是否继续、回退或终止。门控机制为 Chain 增加了安全护栏。

5. **路由链（Router Chain）** — 第一步是一个路由器 Prompt，判断当前请求应该走哪个子链。这是条件链的高级形式，适合多意图 Agent 场景。

```python
# 路由器链示例
ROUTER_PROMPT = """
用户输入: {user_input}

判断这是哪类请求:
1. 信息查询 - 需要查找知识库
2. 代码编写 - 需要生成或修改代码
3. 问题排查 - 需要诊断错误
4. 数据处理 - 需要处理文件或数据

只输出数字。
"""
router_output = await llm_call(ROUTER_PROMPT)
if router_output == "1":
    await info_retrieval_chain.run(user_input)
elif router_output == "2":
    await code_gen_chain.run(user_input)
# ...
```

## 实现的最大挑战

**错误传播和恢复**是最大挑战。链中任何一个步骤的失败（LLM 格式错误、解析异常、工具超时）如果不妥善处理，会导致后续所有步骤基于错误的数据执行。恢复策略包括：（1）每个步骤设置重试机制和最大重试次数；（2）关键步骤设置 Checkpoint，失败后回滚到上一个 Checkpoint；（3）降级策略——某一步失败时使用默认值或跳过该步骤；（4）中间结果持久化，使得系统崩溃后可以从最近的 Checkpoint 恢复。

## 能力边界与结果边界

Prompt Chaining 擅长"可拆解的任务流程"，即一个复杂任务可以自然分解为多个子步骤（如报告生成 = 收集数据 + 分析 + 撰写）。但对于"高度创造性的连贯输出"（如写一首诗或即兴演讲），链式处理反而可能破坏流畅性。此外，Chaining 无法解决步骤划分不合理导致的"信息损失"——如果第一步的摘要丢掉了关键信息，后续步骤无法找回。

## 与其他技术的区别

与 Tool Use 的区别：Tool Use 是 Agent 在单步决策中选择调用哪个工具，Chaining 是开发者预先编排的多步流程。与 Conditional Prompts 的区别：Conditional Prompts 是"根据条件选择不同的单个 Prompt"，Chaining 是"多个有序的 Prompt 步骤"。在实际系统中，Conditional Prompts 常用于 Chain 的某个核心步骤中。

## 核心优势

- **可观测性**：每一轮的输入输出都可单独记录和分析，便于调试
- **可控性**：开发者可以在每个步骤注入校验规则、人工审核点和安全策略
- **可组合性**：Chain 本身可以作为更大 Chain 的一个步骤，实现层次化编排
- **任务聚焦**：每个子 Prompt 只需关注单一目标，Prompt 设计更简单、更精确

## 工程优化方向

1. **延迟优化**：非关键步骤使用快速模型；有依赖的步骤串行，无依赖的步骤并行；对长链启用流式中间结果展示
2. **缓存优化**：对幂等的步骤（如数据检索结果）建立缓存，避免重复调用
3. **可视化管理**：开发 Chain 的 DAG 可视化工具，拖拽式编排和调试
4. **自动 Chain 生成**：由 LLM 分析任务后自动生成 Chain 结构（Agentic RAG、AutoGPT 思路的延续）

## 适合场景的判断标准

以下情况适合使用 Chaining：（1）任务可清晰拆解为 3 个以上子步骤；（2）某些中间结果需要人工审核或跨步骤共享；（3）不同步骤需要不同的模型/工具/参数配置；（4）需要对任务中间步骤做详细日志记录。如果任务简单（一步推理即可完成），单一 Prompt 调用更高效。
