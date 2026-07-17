# 3.3.2 automatic-optimize — 自动优化（DSPy, OPRO, APE）

## 简单介绍

自动优化方法旨在将 Prompt 优化从"人工试错"转变为"算法搜索"。核心思路是将 Prompt 视为可优化的参数，在定义好的评估指标驱动下，利用搜索、采样或梯度信号自动迭代改进 Prompt。代表性方法包括 DSPy（编程框架）、OPRO（LLM 作为优化器）和 APE（自动 Prompt 工程师）。

## 基本原理

自动优化的通用范式是"搜索 + 评估 + 反馈"的闭环：

```python
def auto_optimize(prompt_template, eval_fn, search_space):
    best_prompt = prompt_template
    best_score = -inf
    for iteration in range(max_iterations):
        candidates = propose_candidates(search_space, best_prompt)
        for candidate in candidates:
            score = eval_fn(candidate)
            if score > best_score:
                best_prompt, best_score = candidate, score
    return best_prompt
```

不同方法的差异在于 `propose_candidates` 的实现方式。

## 背景

2023 年后，研究者发现 Prompt 优化存在严重的不连续性——微调措辞可能带来效果跳变，传统基于梯度的优化方法不适用。于是出现了三种主要思路：DSPy 的编译式优化、OPRO 的 LLM-as-optimizer、APE 的大规模采样筛选。

## 之前针对这个问题的做法与结果

在此之前，Prompt 优化完全依赖人工。社区积累了大量的 Prompt 技巧和模板，但缺乏系统性。自动优化的出现使得：
- 搜索空间从"一个人的直觉"扩展到"算法能探索的范围"
- 发现了大量反直觉的有效 Prompt 模式
- 但早期自动优化方法计算成本高，且容易过拟合到评估集

## 核心矛盾

自动优化的核心矛盾是**探索广度与评估成本的对立**。搜索空间越大（尝试更多 Prompt 变体），找到更好 Prompt 的概率越高，但每次评估都需要运行 LLM，成本线性增长。DSPy 通过少样本优化缓解此问题，OPRO 通过 LLM 的语义理解减少搜索盲目性，APE 通过先采样再筛选的 two-stage 策略控制成本。

## 当前主流优化方向

- **DSPy**：通过声明式编程将 Prompt 优化编译为可微分的程序优化问题
- **OPRO（Optimization by PROmpting）**：让 LLM 扮演优化器，基于历史结果提出新的 Prompt 变体
- **APE（Automatic Prompt Engineer）**：大规模生成候选 Prompt 后用评估指标筛选最佳者
- **混合方法**：先用 APE 做结构级探索，再用 DSPy 做参数级精调

```python
# DSPy 风格示例
import dspy

class MyTask(dspy.Signature):
    """处理用户查询并返回答案"""
    question = dspy.InputField()
    answer = dspy.OutputField()

pipeline = dspy.Predict(MyTask)
optimizer = dspy.teleprompt.BootstrapFewShot()
optimized = optimizer.compile(pipeline, trainset=trainset)
```

## 实现的最大挑战

- **评估指标设计**：自动优化的上限由评估指标的质量决定——指标错了，优化方向就错了
- **过拟合风险**：在固定测试集上优化的 Prompt 可能对其他输入泛化不佳
- **计算成本**：一次完整的自动优化可能需要数百到数千次 LLM 调用，小型团队难以承受
- **可解释性差**：自动找到的 Prompt 为什么有效，往往难以解释

## 能力边界与结果边界

- 自动优化擅长参数级精调，不擅长结构级重新设计
- 需要有高质量的标注数据集作为评估基础
- 优化收益在 5-20% 的相对提升范围内，期望"翻倍式"改进不现实
- 不同方法的适用边界由搜索策略决定——DSPy 适合结构固定的任务，APE 适合探索性任务

## 与其他技术的区别

- vs 人工优化：自动优化不依赖人类直觉，但需要更多计算资源
- vs 进化算法：自动优化通常使用 LLM 生成候选（语义驱动），进化算法使用随机变异（无语义）
- vs 元 Prompt：元 Prompt 是单次 LLM 调用中的自我反思，自动优化是多轮迭代的搜索过程

## 核心优势

- **可扩展**：计算资源充足时可以持续迭代
- **可复现**：同样的设置跑出的结果一致，方便团队协作
- **发现反直觉模式**：可能找到人类想不到的有效措辞
- **减轻评估疲劳**：将评估工作自动化，解放人力

## 工程优化方向

- 缓存评估结果避免重复计算
- 使用分层搜索：先粗搜结构，再精搜措辞
- 结合人工审核：自动优化产生的候选 Prompt 应经人工审查后再上线
- 持续集成：将自动优化纳入开发流水线，数据集更新时自动重新优化

## 适合场景的判断标准

- 任务定义清晰且有稳定的评估指标
- 拥有足够规模的标注数据集（至少 100-500 条）
- 计算预算允许数百次 LLM 调用
- 已有基线 Prompt，希望在此基础上持续精调
