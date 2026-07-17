# 10.4.3 Prompt Inspection — Prompt 检查

## 简单介绍

Prompt Inspection（Prompt 检查）是 Agent 调试中**最有价值**的手段——也是最容易被忽视的手段。许多 Agent 的"奇怪行为"归根结底是 Prompt 的问题：系统指令被用户输入覆盖了、工具描述的优先级错了、历史消息被截断改变了上下文的含义。Prompt Inspection 的核心是回答 **"Agent 实际看到了什么？"**

## 基本原理

Agent 向 LLM 发送的完整消息结构远比开发者设想的复杂：

```python
# 开发者想象中的 Prompt
"用户问天气，帮我查一下"

# Agent 实际发送的 Prompt
[
    {"role": "system", "content": "你是天气助手...（500 tokens）"},
    {"role": "user", "content": "我住在北京，今天天气怎么样？"},
    {"role": "assistant", "content": "让我查一下（调用工具 get_location）"},
    {"role": "tool", "content": "{"city": "Beijing"}"},
    {"role": "assistant", "content": "好的，我用北京查天气..."},
    # ... 还有 5 轮历史消息（2000+ tokens） ...
    {"role": "user", "content": "那明天呢？"}  ← 最新的用户输入
]

# Prompt Inspector 需要展示的内容：
# 1. 完整的 messages 数组（所有角色）
# 2. 系统 Prompt（Agent 定义 + 工具定义）
# 3. 历史消息压缩后的摘要（如果是压缩过的）
# 4. 当前上下文窗口的利用率（like "8,500/10,000 tokens used"）
```

## Prompt Inspection 的内容

```python
class PromptInspector:
    """Prompt 检查器"""

    def inspect(self, agent: "Agent") -> InspectionResult:
        messages = agent.get_prompt_messages()

        return InspectionResult(
            # 1. 基础信息
            total_tokens=count_tokens(messages),
            max_tokens=agent.context_limit,
            utilization=total_tokens / max_tokens * 100,

            # 2. 各部分分解
            breakdown={
                "system_prompt": {
                    "tokens": count_tokens(messages[0]),
                    "content": messages[0]["content"],
                },
                "tool_definitions": {
                    "tokens": count_tokens(agent.tool_definitions),
                    "tools": list(agent.tool_definitions.keys()),
                },
                "message_history": {
                    "tokens": count_tokens(messages[1:-1]),
                    "num_messages": len(messages) - 2,
                    "oldest_message_age": self._get_oldest_age(messages),
                },
                "current_input": {
                    "tokens": count_tokens(messages[-1]),
                    "content": messages[-1]["content"],
                },
            },

            # 3. 问题检测
            issues=[
                Issue(
                    type="token_warning",
                    severity="warning",
                    message="上下文利用率 85%，接近 100% 上限",
                ),
                Issue(
                    type="tool_definition",
                    severity="info",
                    message="当前定义了 5 个工具，共占用 1200 tokens",
                ),
            ],
        )
```

## 背景

在传统开发中，"检查输入数据"是最基本的调试手段。但 Agent 的"输入"（Prompt）不是由开发者硬编码的——它是由 Agent 框架动态组装的，包含系统指令、工具定义、历史消息、用户输入等杂糅在一起的复杂体。开发者往往会假设"Agent 收到的 Prompt 和我写的 System Prompt 差不多"，但实际上两者差距极大。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 不检查 Prompt | 假设 Agent 收到"我写的那个 System Prompt" | 完全忽略了工具定义、历史消息等大量上下文的影响 |
| 从 LLM 日志看 Prompt | 查看 LLM API 的调用日志 | 能看到完整的 messages，但格式不友好，难以解析 |
| 手动提取 messages 数组 | 在代码中 print(messages) | 信息太多，刷屏严重，忽略关键细节 |
| 只看最后一条消息 | 检查当前用户输入是否正确传入了 | 忽略了系统 Prompt 和历史消息的影响 |

## 核心矛盾

| 矛盾维度 | 不看 Prompt | 看完整 Prompt | 结构化检查 |
|---------|-----------|-------------|-----------|
| 信息量 | 0 | 极大 | 适中（有筛选） |
| 问题发现率 | 0% | 高 | 高 |
| 可操作性 | N/A | 信息过载难以操作 | 有指导性建议 |
| 性能影响 | 无 | 收集完整 Prompt 有开销 | 低 |

## 当前主流优化方向

1. **差异对比**——对比"开发者期望的 Prompt"和"实际发送的 Prompt"之间的差异，自动标出意外添加或丢失的内容
2. **Prompt 历史版本快照**——记录 Agent 每次迭代的 Prompt 版本快照，支持对比不同版本的 Prompt 变化
3. **消息分割视图**——将消息按来源分组：系统 Prompt（开发者写的）、工具定义（框架自动生成的）、历史消息（之前轮次的对话）、用户输入（用户写的），每组的 Token 数一目了然
4. **注入检测**——自动检查用户输入是否可能与系统指令冲突（Prompt 注入检测），标记可能的注入风险

## 实现的最大挑战

1. **Prompt 的"完整"捕获**——Prompt 可能在多个地方被修改（Agent 框架 + 历史管理 + 上下文压缩），需要从最终发送给 LLM 的消息数组中捕获，而非从中间变量获取
2. **大型 Prompt 的展示**——一个包含 8K tokens 的 Prompt 在控制台或 UI 中完全展示起来很困难，需要折叠/展开/搜索功能
3. **历史消息压缩后的还原**——当历史消息被摘要压缩后，Inspector 需要展示的是压缩后的版本（"这里原来有 5 轮对话，已被压缩为摘要"），而非原始版本
4. **实时性要求**——Agent 生产环境中的 Prompt Inspection 不能显著增加延迟，需要采样或只收集元数据

## 能力边界

**能做什么：**
- 看到 Agent 实际发送给 LLM 的完整 Prompt
- 分析 Prompt 的 Token 消耗分布
- 检测 Prompt 的潜在问题（过长、工具定义缺失、系统指令被覆盖）
- 比较不同版本的 Prompt 差异

**不能做什么：**
- 不能通过"看 Prompt"就解决所有问题——即使 Prompt 是正确的，LLM 的输出也可能有幻觉
- 不能保证看到的 Prompt 是"完整的"——如果 Agent 框架在发送前又修改了 messages，Inspector 可能捕获不到最终版本
- 不能自动修复 Prompt 问题——检测到问题后仍需人工干预

## 最终工程优化

1. **装饰器式 Prompt 捕获**——通过装饰器在 `llm.generate()` 调用时自动捕获完整的 messages 参数，是最可靠的捕获时机
2. **可视化 Prompt 分析器**——将 Token 消耗按消息来源可视化（饼图/柱状图），让开发者直观看到"哪部分吃了最多 Token"
3. **Prompt 检查快捷键**——在逐步调试模式中，每次暂停时通过 `p` 命令直接查看当前 Prompt 的关键信息（Token 数、消息数、最后一条消息内容）
4. **错误自动关联**——当 Agent 出错时（如返回无效 JSON），自动记录此时发送的 Prompt 内容，将错误与对应的 Prompt 绑定，便于根因分析
