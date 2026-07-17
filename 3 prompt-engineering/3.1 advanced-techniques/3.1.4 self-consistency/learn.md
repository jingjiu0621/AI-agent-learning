# Self-Consistency — 自一致性提示

## 简单介绍

Self-Consistency（自一致性）是一种极具工程实用性的推理增强技术。它的核心思想是：**与其相信 LLM 单次推理的结果，不如让它推理多次然后投票选出现频率最高的答案。** 这与集成学习（Ensemble Learning）的原理类似——多个弱分类器的组合优于单一强分类器。

与 ToT 和 GoT 在推理过程中进行路径交互不同，Self-Consistency 的策略是"独立采样 → 聚合答案"——多条推理链并行运行，互不干扰，最后通过 Majority Voting（多数投票）或 Weighted Voting（加权投票）选出最终答案。它的实现极其简单，但却能带来稳定且显著的准确率提升。

## 基本原理

Self-Consistency 的工作流程分为三个步骤：

1. **多样本生成**：使用 CoT 对同一个问题生成 N 条独立的推理链（N 通常为 5-20），每条链的温度参数（temperature）设置为 > 0（通常 0.5-0.8），以确保输出多样性
2. **答案提取**：从每条推理链中提取最终答案
3. **聚合投票**：对所有答案进行投票，出现频率最高的作为最终输出

```python
def self_consistency(question: str, n_samples: int = 10, temperature: float = 0.7):
    """
    自一致性：多次推理 + 投票聚合
    """
    reasoning_chains = []

    # Step 1: 生成 N 条独立的 CoT 推理链
    for i in range(n_samples):
        chain = llm.generate(
            f"{question}\n\nLet's think step by step.",
            temperature=temperature,
            max_tokens=500
        )
        reasoning_chains.append(chain)

    # Step 2: 从每条推理链中提取答案
    answers = []
    for chain in reasoning_chains:
        answer = extract_answer(chain)
        answers.append(answer)

    # Step 3: 投票选出最终答案
    # 加权投票：根据推理链的置信度加权
    from collections import Counter
    vote_count = Counter(answers)

    # 简单多数投票
    final_answer = vote_count.most_common(1)[0][0]

    # 也可以返回置信度
    confidence = vote_count[final_answer] / n_samples

    return final_answer, confidence, reasoning_chains


def extract_answer(chain: str) -> str:
    """从 CoT 推理链中提取最终答案"""
    # 简单实现：取 "answer is X" 或最后一行
    # 在实际工程中，可以让 LLM 专门提取答案
    lines = chain.strip().split("\n")
    for line in reversed(lines):
        if "answer" in line.lower():
            return line.strip()
    return lines[-1].strip()
```

## 背景

Self-Consistency 由 Google 在 2022 年的论文 "Self-Consistency Improves Chain of Thought Reasoning in Language Models" 中提出，紧随 CoT 论文之后。

研究者观察到：即使是使用完全相同的 CoT Prompt，LLM 在多次推理时也会产生不同的推理路径和不同的答案。这种**推理的不确定性**在 temperature > 0 时被放大。这种不确定性通常被视为模型的"不稳定性"，但 Self-Consistency 巧妙地利用它——将这种不确定性从缺陷转化为优势，通过"多数投票"机制让稳定出现的答案胜出。

## 之前针对这个问题的做法与结果

| 做法 | 描述 | 效果与局限 |
|------|------|-----------|
| 单次 CoT | 一次推理得到一个答案 | 结果方差大，好时很好差时很差，不稳定。在 GSM8K 上单次 CoT 准确率约 55-60% |
| 温度采样 + 人工选优 | 多次生成后人工挑选最佳答案 | 不可扩展，无法自动化，失去了 LLM 的自动化优势 |
| Beam Search Decoding | 在 token 级别使用 beam search 找到最可能的 token 序列 | 对推理任务效果有限。Beam Search 倾向于高频 token 序列，但在推理任务中，正确的推理路径不一定是最可能的 token 路径 |

## 核心矛盾

| 矛盾维度 | 单次推理 | 自一致性 |
|---------|---------|---------|
| 结果稳定性 | 差，单次波动大 | 好，通过投票降低方差 |
| Token 成本 | 低 | 高（N 倍） |
| 延迟 | 低 | 高（串行 N 次或并行 N 次 API 调用） |
| 实现复杂度 | 低 | 低（主要逻辑是投票） |
| 多样性来源 | 无 | 依赖 temperature 控制的随机性 |
| 可并行性 | N/A | 可完全并行（N 条推理链互不依赖） |

## 当前主流优化方向

1. **加权投票策略**——不再是简单的"一人一票"，而是根据推理链的置信度（如生成概率 logprob 的均值）、推理链长度、或自评估分数进行加权投票
2. **自适应样本数**——根据问题的难度或首次推理的置信度动态调整采样次数。简单问题用 3-5 次，复杂问题用 10-20 次
3. **答案聚类**——当答案空间是连续值或文本时，使用聚类算法（如编辑距离聚类、语义相似度聚类）先聚类再投票，而非严格的字符串匹配
4. **混合 Temperature 策略**——不同的 temperature 值适合不同类型的探索：低温（0.1-0.3）产生高概率的保守推理，高温（0.8-1.0）产生有创意的多样推理。混合使用效果更好

## 实现的最大挑战

1. **答案提取的歧义性**——从自然语言推理链中提取最终答案可能很困难。LLM 可能在推理链的不同位置说出了多个"答案"（"答案是 42"后又补充"但另一种可能是 43"），需要稳健的提取逻辑
2. **投票的"从众陷阱"**——如果 LLM 的某个系统性错误导致多次推理都得出相同但错误的答案（例如所有推理链都忘了进位），Self-Consistency 会信心满满地给出错误答案。投票机制不能纠正系统偏差
3. **成本-收益的边际递减**——样本数增加到一定程度后，准确率提升趋于平缓。10 次和 20 次的投票结果可能完全相同，但成本翻倍

## 能力边界与结果边界

**能做什么：**
- 减少 CoT 推理的方差，使结果稳定可靠
- 在算术推理（GSM8K、MATH）上提升 5-15 个百分点
- 在常识推理、符号推理上也有显著提升
- 提供内置的"置信度"指标（投票占比 = 置信度）

**不能做什么：**
- 不能纠正模型的系统性偏差（所有推理链共享同一个偏差源则投票无效）
- 不能处理需要路径间交互的推理（每条链独立，无协作）
- 不能提高单次推理的上限能力（模型本身不会的不可能通过投票变会）

**产生的结果：**
- 确定性的最终答案 + 置信度分数
- N 条推理链（可用于事后分析"不同推理路径的差异"）
- 答案分布（展示所有可能的答案及其支持率，提供更丰富的信息）

## 与其他技术的区别

| 对比技术 | 核心区别 |
|---------|---------|
| Chain-of-Thought | SC = 多次 CoT + 投票聚合；CoT = 单次推理 |
| Tree-of-Thoughts | SC 的路径是独立的、无交互的；ToT 的路径有评估和剪枝交互 |
| Ensemble (模型集成) | SC 是对同模型的多次采样；Ensemble 是不同模型的输出聚合 |
| Beam Search | Beam Search 在 token 级别搜索；SC 在推理链级别聚合 |

## 核心优势

1. **极低的实现门槛**——不需要修改推理逻辑本身，只需要对现有的 CoT 进行多次调用 + 投票聚合
2. **天然并行**——N 条推理链完全独立，可以并行调用 API，端到端延迟几乎等于单次推理延迟（而非 N 倍）
3. **提供了置信度指标**——投票的一致性就是答案的置信度，这为后续决策提供了有价值的信号（如投票一致性低则触发重新推理或人类介入）
4. **与任何推理技术正交**——SC 可以与 CoT、ToT、ReAct 等技术叠加使用，只要它们的输出可提取为可投票的形式

## 工程优化方向

1. **答案提取专用 Prompt**——不要用字符串匹配提取答案，而是用一个专门的"答案提取" LLM 调用从每条推理链中提取结构化答案（JSON 格式），提高提取准确率
2. **流式聚合**——在流式 API 调用中，当 N - k 条推理链已经给出相同答案时，取消剩余的调用，实现"提前终止"以节省成本
3. **动态 Temperature 策略**——对首次推理置信度高的任务用低 temperature 和少样本，对置信度低的任务用高 temperature 和多样本，实现成本自适应

## 适合场景的判断标准

**应该使用 Self-Consistency 的场景：**
- 任务有确定性答案（数学、选择题、分类），便于投票
- 需要对推理结果的置信度做量化评估
- 有并行调用的基础设施支持
- 更关注结果的稳定性而非单次推理的创造性

**不应使用 Self-Consistency 的场景：**
- 答案空间是开放式的、无法投票的（如创意写作、开放式问答）
- 每次推理成本很高（如使用了长上下文或多工具调用）
- 首 token 延迟是关键 KPI（即使并行调用，总调用量仍会增加负载）
