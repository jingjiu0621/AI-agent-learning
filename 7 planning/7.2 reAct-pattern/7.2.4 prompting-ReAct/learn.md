# 7.2.4 Prompting ReAct — ReAct 的 Prompt 工程设计

## 简单介绍

ReAct 的行为模式高度依赖 Prompt 的设计——Agent 的推理质量、工具选择准确性、错误处理能力，很大程度上由 System Prompt 和 Few-shot 示例的质量决定。本节深入剖析如何设计驱动 ReAct 循环的 Prompt。

## 基本原理

ReAct Prompt 的核心构成：

1. **系统角色定义**：告诉 Agent 它是一个"能够思考、调用工具、观察结果的 AI 助手"
2. **可用工具描述**：格式统一的工具列表（名称、功能、参数）
3. **响应格式约束**：强制 Agent 按 Thought/Action/Observation 格式输出
4. **行为规范**：什么情况做什么事（如观察为空时怎么办）
5. **Few-shot 示例**：1-3 个完整的 ReAct 轨迹示例

## 背景

ReAct 论文中使用了一种特定格式的 Prompt，后续的 LangChain、AutoGen 等框架对其进行了工程化改造。实践中发现，Prompt 的设计细节对 ReAct 行为有决定性影响——格式约束比内容描述更重要，示例选择比示例数量更重要。

## 之前针对这个问题的做法与结果

1. **论文原版格式**：用 Thought/Action/Action Input/Observation 标记。结果：效果良好但格式较冗长。
2. **LangChain 改进版**：引入 AI 前缀和 Human 前缀，更清晰地分隔角色。结果：多轮对话中更稳定。
3. **JSON 模式**：将 Thought/Action/Observation 用 JSON 表示。结果：解析更可靠，但限制了 LLM 自由生成的灵活性。
4. **Markdown 格式**：用 ``` 代码块包裹 Action。结果：人类可读性好，但 LLM 有时会忘记闭合代码块。

## 核心矛盾

**格式严格性 vs LLM 灵活性**：格式越严格，Agent 的输出越容易被程序解析，但 LLM 在严格格式下的"思考"可能变得生硬和模式化；格式越宽松，LLM 能更自然地思考，但解析 Agent 输出的代码复杂度增加，出错率上升。

## 当前主流优化方向

1. **结构化输出约束**：使用 LLM 的 JSON Mode 或 Tool Use 机制来替代自由文本格式——Action 直接映射为 Function Calling，Thought 作为文本输出。这是当前生产系统最推荐的方案。
2. **动态 Few-shot 选择**：根据当前任务，从示例库中检索最相似的 ReAct 轨迹作为示例，而非使用固定的示例集。
3. **渐进式指令**：将 ReAct 行为规范分层——底层规则始终有效（如"必须输出 Thought"），上层规则可根据场景切换（如"遇到错误时最多重试 2 次"）。
4. **Hint 注入**："如果你觉得需要调用工具，请先想清楚需要什么参数"——这类 Hint 能显著提升工具调用的参数准确性。

## 实现的最大挑战

```python
class ReActPromptBuilder:
    def build_system_prompt(self, tools: list[Tool], examples: list[Example] = None) -> str:
        prompt = f"""你是一个能够思考、使用工具、观察结果的 AI 助手。
你的工作模式是通过反复的"思考→行动→观察"循环来完成任务。

## 可用工具
{self._format_tools(tools)}

## 输出格式
每次回复必须包含以下内容：
1. Thought: 你当前的思考（你观察到什么、你打算做什么、为什么）
2. Action: 你要调用的工具名称
3. Action Input: 工具调用参数（JSON 格式）

如果任务已完成，输出：
Thought: 任务完成。
Final Answer: [最终答案]

## 行为规则
- 每次只能调用一个工具
- 仔细阅读观察结果后再决定下一步
- 如果工具返回错误，分析错误原因后重试或更换工具
- 如果工具返回空结果，换一种方式查询
- 最多进行 {self.max_steps} 次循环"""

        if examples:
            prompt += "\n\n## 示例\n" + self._format_examples(examples)
        
        return prompt
    
    def _format_tools(self, tools: list[Tool]) -> str:
        """格式化工具有效描述——描述质量比数量更重要"""
        lines = []
        for t in tools:
            lines.append(f"- {t.name}: {t.description}")
            lines.append(f"  参数: {json.dumps(t.parameters)}")
        return "\n".join(lines)
    
    def _format_examples(self, examples: list[Example]) -> str:
        """格式化 few-shot 示例——完整的 ReAct 轨迹"""
        parts = []
        for i, ex in enumerate(examples):
            parts.append(f"--- 示例 {i+1} ---")
            parts.append(ex.trajectory)  # 完整的 Thought/Action/Observation 序列
        return "\n".join(parts)
```

最大挑战是**Prompt 中工具描述的完整性 vs 上下文窗口**：当工具数量超过 10-20 个时，工具描述本身就会占据大量 token。解决方案包括：动态工具选择（只注入与当前任务相关的工具描述）、工具描述压缩（精简但保留关键信息）、工具分组（按领域分组，先选组再选工具）。

## 能力边界和结果边界

- **能力**：精心设计的 Prompt 可以让 ReAct Agent 在 90% 以上的场景中正确遵循 ReAct 格式
- **边界**：Prompt 无法完全消除 LLM 的格式错误（偶尔会跳过 Thought、不闭合 JSON）；长对话后 LLM 可能"忘记"格式要求
- **结果**：好的 Prompt 设计可以让工具选择准确率从 70% 提升到 90%+，错误恢复率从 50% 提升到 80%+

## 核心优势

Prompt 是 ReAct 行为控制最直接、最灵活的手段。无需修改代码，通过调整 Prompt 就可以改变 Agent 的推理深度、工具选择策略、错误处理方式等核心行为。

## 工程优化方向

- Prompt 版本管理：每次 Prompt 修改都要记录版本、测试效果、灰度上线
- 自动 Prompt 优化：使用 DSPy 等框架自动搜索最优 ReAct Prompt 模板
- 运行时 Prompt 注入：在特定步骤动态注入 Hint（如"这一步可能出错，请仔细检查参数"）
- A/B 测试框架：对不同 Prompt 变体做在线评估，数据驱动决策

## 适合场景

- 所有使用 ReAct 的 Agent 系统
- 需要精细控制 Agent 行为的应用
- 作为 Prompt Engineering 技能在 Agent 领域的具体应用场景
