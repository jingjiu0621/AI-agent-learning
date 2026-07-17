# Chain-of-Thought (CoT) — 思维链提示

## 简单介绍

Chain-of-Thought（思维链提示）是高级推理提示技术中最基础也最核心的方法。它的核心思想极其简单：**在让 LLM 回答问题之前，先让它生成一步步的推理过程。** 这种"先思考再回答"的模式极大地提升了 LLM 在数学、逻辑、常识推理等复杂任务上的表现。

CoT 有两种主要变体：**零样本 CoT**（Zero-shot CoT，在 Prompt 末尾加上 "Let's think step by step"）和 **少样本 CoT**（Few-shot CoT，在 Prompt 中提供包含推理步骤的示例）。对于 Agent 开发场景，CoT 是几乎所有推理任务的基础组件。

## 基本原理

CoT 的本质是一种**推理过程的外部化**。LLM 在预训练阶段已经"见过"大量包含逐步推理的文本（教科书、技术文档、论文等），CoT Prompt 的作用是激活这些内化的推理能力，让 LLM 在 token 生成过程中"边想边写"。

零样本 CoT 的典型实现：

```python
def zero_shot_cot(prompt: str) -> str:
    """零样本思维链：追加触发短语"""
    cot_prompt = f"{prompt}\n\nLet's think step by step."
    return llm.generate(cot_prompt)
```

少样本 CoT 的典型实现：

```python
def few_shot_cot(question: str, examples: list[dict]) -> str:
    """少样本思维链：提供带推理步骤的示例"""
    prompt = ""
    for ex in examples:
        prompt += f"Q: {ex['question']}\n"
        prompt += f"A: {ex['reasoning']}\n"
        prompt += f"Therefore, the answer is {ex['answer']}.\n\n"
    prompt += f"Q: {question}\nA:"
    return llm.generate(prompt)
```

关键机制在于：当 LLM 生成 "Let's think step by step" 之后，它后续的所有 token 都会沿着"推理模式"而非"直接回答模式"生成。这改变了 LLM 的条件概率分布——从 P(answer | question) 变为 P(answer | question, reasoning_steps)。

## 背景

CoT 由 Google 在 2022 年的论文 "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models" 中正式提出。在此之前，LLM 的 Prompt 设计主要关注"给指令、得答案"的简单模式，LLM 被当作一个巨大的知识库直接查询。

然而随着模型规模的增大，研究人员发现：更大的模型虽然在知识记忆上更强，但在需要多步推理的任务上仍然表现不佳。原因在于 LLM 的 next-token-prediction 训练范式天然倾向于"快速匹配"而非"深度思考"。

## 之前针对这个问题的做法与结果

| 做法 | 描述 | 效果与局限 |
|------|------|-----------|
| 直接提问（Standard Prompting） | 直接问"X 等于多少？"期望 LLM 直接输出答案 | 在简单任务上尚可，但在 GSM8K 等数学推理数据集上准确率不到 20%（GPT-3 175B） |
| 少样本示例（Few-shot Prompting） | 提供几个问答对作为示例 | 比零样本好，但 LLM 学到的是"答案模式"而非"推理模式"，在需要新推理的任务上提升有限 |
| 分步指令（Step-by-step instruction） | 在 Prompt 中写"请分步骤回答" | 有一定效果，但缺乏"推理链示例"的引导力，LLM 仍倾向于跳步 |

## 核心矛盾

| 矛盾维度 | 直接回答模式 | 思维链模式 |
|---------|-------------|-----------|
| 推理深度 | 浅层，易跳步 | 深层，步骤完整 |
| Token 消耗 | 低 | 高（3-5 倍） |
| 可解释性 | 差 | 好（推理过程可见） |
| 准确性 | 低（复杂任务） | 高（复杂任务） |
| 速度 | 快 | 慢 |
| 稳定性 | 方差大 | 方差减小但仍有波动 |

## 当前主流优化方向

1. **CoT 示例自动选择**——从示例库中动态选择与当前问题最相似的 CoT 示例，而非使用固定示例（如 Auto-CoT）
2. **结构化 CoT**——不仅要求"逐步推理"，还要求按特定结构（JSON、XML、Markdown 列表）组织推理过程，便于 Agent 解析中间结果
3. **多语言 CoT**——在非英语场景下，推理语言的选择对效果有显著影响，研究者正在探索"用英语思考，用母语回答"的混合模式
4. **CoT 与工具调用结合**——在推理步骤中插入工具调用（如"我需要计算这个表达式，调用 calculator 工具"），实现推理与行动的深度耦合

## 实现的最大挑战

1. **推理链的忠实性**——LLM 可能生成看似合理但实际上错误的推理步骤（hallucinated reasoning）。CoT 暴露了推理过程，但并不保证推理的正确性
2. **Token 预算管理**——在 Agent 系统中，CoT 推理会消耗大量 context window，可能挤压后续工具调用或系统指令的空间
3. **错误传播**——早期推理步骤中的微小错误会在后续步骤中被放大，形成"级联错误"（cascading errors）

## 能力边界与结果边界

**能做什么：**
- 数学应用题（GSM8K、MATH）上准确率提升 2-3 倍
- 常识推理（CSQA、StrategyQA）上的显著提升
- 符号推理、逻辑推理等结构化任务
- 作为 Agent 推理管道的基础组件

**不能做什么：**
- 需要事实性知识（knowledge-intensive）的任务——CoT 不能弥补模型知识缺失
- 需要外部验证的推理——CoT 不能自我纠错
- 极其简单的任务（如"2+2=?"）——CoT 反而浪费 token

**产生的结果：**
- 更长的 output（推理 + 答案）
- 更高的首 token 延迟（因为需要先生成推理过程）
- 可审计的推理轨迹

## 与其他技术的区别

| 对比技术 | 核心区别 |
|---------|---------|
| Standard Prompting | CoT 生成显式推理步骤；Standard 直接输出答案 |
| Few-shot Prompting | CoT 示例包含推理过程；Few-shot 示例只包含问答对 |
| Tree-of-Thoughts | CoT 单路径；ToT 多路径搜索 |
| ReAct | CoT 纯推理；ReAct 推理 + 行动交替 |

## 核心优势

1. **实现成本极低**——只需要在 Prompt 末尾加一句话，零工程开销
2. **通用性强**——几乎适用于所有需要推理的场景，是"默认开启"的推理技术
3. **可解释性好**——Agent 的每一步决策都有推理记录，便于调试和审计
4. **与其他技术正交**——CoT 可以与 ToT、SC、ReAct 等技术叠加使用，不冲突

## 工程优化方向

1. **推理长度控制**——通过 Prompt 约束（"请用不超过 3 步推理"）或 max_tokens 参数限制推理链长度，在质量和延迟之间取得平衡
2. **结构化解析**——让 CoT 输出按固定格式（如 JSON step list），便于代码解析下游步骤
3. **条件性 CoT**——只在需要推理的任务上触发 CoT，简单任务直接回答。可以通过 LLM 自评估问题复杂度来决定是否启用 CoT

## 适合场景的判断标准

**应该使用 CoT 的场景：**
- 任务需要多步逻辑推理（数学、代码、规划）
- 需要可审计的推理过程（金融、医疗、法律等合规场景）
- 作为更复杂推理技术（ToT、GoT）的基础组件

**不应使用 CoT 的场景：**
- 任务极其简单直接
- 响应延迟是硬约束（CoT 会增加 200ms-2s 的推理时间）
- 模型本身能力不足以生成有效推理（小模型如 7B 以下 CoT 效果不明显甚至更差）
