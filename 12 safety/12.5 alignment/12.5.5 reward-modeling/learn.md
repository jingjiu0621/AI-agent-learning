# 12.5.5 reward-modeling — 奖励模型设计与训练

## 简单介绍

奖励模型（Reward Model, RM）是 RLHF 对齐流程中的核心组件。它的任务是对模型生成的回复进行打分——不是简单地判断"好"或"坏"，而是学习人类的偏好倾向，输出一个连续值作为奖励信号。这个奖励信号随后被 PPO 等强化学习算法用来优化策略模型的行为。**奖励模型的质量直接决定了 RLHF 对齐的效果上限**：如果 RM 是错的，RLHF 只会让模型在错误的方向上越走越远。

## 基本原理 — reward model learns to predict human preferences

奖励模型的训练本质上是**偏好学习（Preference Learning）**：给定两个回复 (y₁, y₂) 和人类标注者表明 y₁ 优于 y₂ 的偏好，RM 需要学会对 y₁ 给出比 y₂ 更高的分数。关键在于 RM 学习的是**相对偏好而非绝对质量**——它不需要知道"这个回复有多好"，只需要知道"这个回复比那个好"。

形式化地，奖励模型 r(x, y) 是一个以提示 x 和回复 y 为输入、输出标量奖励值的函数。训练数据 D = {(x, y_w, y_l)} 包含提示、被偏好的回复和被拒绝的回复。RM 的目标是让 r(x, y_w) > r(x, y_l) 成立。

训练完成后，RM 作为可微分的奖励信号源接入 PPO 循环：策略模型生成回复 -> RM 打分 -> PPO 根据奖励值更新策略。由于 RM 是神经网络，它可以为任何 unseen 的 (x, y) 对分配分数，从而为 RL 提供密集的奖励信号。

## 背景 — Bradley-Terry model → learned reward models → process reward models (PRM)

奖励模型的理论基础是统计学的 **Bradley-Terry 模型**，该模型最早用于成对比较分析（如体育比赛中比较两支队伍的实力）。给定项目 A 和 B，Bradley-Terry 模型假设：

```
P(A 优于 B) = σ(r_A - r_B)
```

其中 σ 是 sigmoid 函数，r_A 和 r_B 是 A 和 B 的潜实力分数。这个公式完美适配偏好学习：人类偏好 y_w 胜过 y_l 的概率由奖励分数的差值决定。

奖励模型的发展经历了三个阶段：

| 阶段 | 时间 | 方法 | 特点 |
|------|------|------|------|
| **Bradley-Terry 起源** | 1952 | 统计分析 | 成对比较的统计学框架 |
| **Learned Reward Model** | 2020-2022 | InstructGPT / GPT-4 | 用神经网络学习奖励函数，整个回复打一个分 |
| **Process Reward Model (PRM)** | 2023-2025 | Math-Shepherd, MiPS | 每一步推理都打分，提供过程级奖励 |

**从 outcome RM 到 process RM 的演进**源于一个关键认知：在数学推理、代码生成等多步任务中，仅仅对最终结果打分无法定位错误发生在哪一步，也无法为中间步骤提供训练信号。过程奖励模型将奖励信号从结果扩展到每一步，从根本上提升了奖励信号的密度和质量。

## 核心矛盾 — reward model is the bottleneck: if RM is wrong, RLHF optimizes for wrong thing

奖励模型是 RLHF 对齐中最关键的薄弱环节：

1. **RM 的不完美是 RLHF 的根本限制**：RLHF 的优化目标完全由 RM 定义。如果 RM 对"什么是好的回复"的判断存在偏差，PPO 优化只会放大这个偏差。**RLHF 不是在优化人类偏好，而是在优化 RM 所认为的人类偏好。**

2. **Goodhart's Law 在 RLHF 中必然发生**："当一个指标变成目标，它就不再是一个好指标。"RM 分数只是人类偏好的代理指标，当 PPO 直接优化 RM 分数时，模型会找到 RM 分数高但实际质量低的回复（reward hacking）。

3. **RM 很难捕捉所有维度的偏好**：人类对回复的评价是多维的——有用性（helpfulness）、无害性（harmlessness）、诚实性（honesty）、风格、事实准确性等。一个标量奖励值难以同时编码所有这些维度。

4. **RM 的泛化能力有限**：RM 在训练分布之外的表现不可靠。当策略模型探索到训练数据之外的回复空间时，RM 可能给出完全不准确的评分。

---

## 详细内容

### 1. Reward Model Architecture: base model → value head, parameter choices, initialization

#### 基础架构

奖励模型的典型架构是在预训练语言模型之上添加一个 **value head**（价值头）：

```
输入: [CLS] prompt + response [SEP]
        │
    Base LM (transformer encoder)
        │
    [last-layer hidden states]
        │
    Value Head (线性层, d_model → 1)
        │
    输出: 标量奖励值 r
```

Value Head 通常是一个单层线性变换，将最后一层 [CLS] token 或最后一个 token 的 hidden state 映射为标量分数。也有实现使用 MLP（2-3 层 + 激活函数）作为 value head 以增加表达能力。

#### 基座模型选择

| 参数 | 选择 | 理由 |
|------|------|------|
| **基座模型大小** | 通常与策略模型同规模或更大 | RM 需要足够容量来编码复杂的偏好模式 |
| **初始化** | 从 SFT 模型或基座 LM 初始化 | 利用预训练知识，避免从零训练 |
| **是否冻结基座** | 通常全量微调 | RM 任务与 LM 任务差异大，需要充分调整 |
| **使用哪个 token 的输出** | 最后一个 token / [CLS] token | [CLS] 更常用于分类；最后一个 token 对生成结果更敏感 |

#### 关于 RM 大小的 Scaling 发现

InstructGPT 论文观察到：**奖励模型大小需要与策略模型大小匹配**。用 175B 的 RM 来训练 175B 的策略模型效果最好。使用较小的 RM（如 6B）训练较大的策略模型会导致奖励信号不够精细，对齐质量下降。但后续研究也发现，随着数据质量和训练方法的改进，较小的 RM 配合适当的训练策略也能达到不错的效果。

---

### 2. Training Objective: Bradley-Terry loss, Plackett-Luce for ranking, comparison loss variants

#### Bradley-Terry Loss（成对偏好损失）

最基本的损失函数，对应 InstructGPT 中使用的训练目标：

给定一个三元组 (x, y_w, y_l)，其中 y_w 比 y_l 更受偏好，损失为：

```
L(θ) = -E[ log σ(r_θ(x, y_w) - r_θ(x, y_l)) ]
```

其中 σ 是 sigmoid 函数。这个 loss 的直观理解是：最大化被偏好的回复得分高于被拒绝回复得分的概率。

#### Plackett-Luce Loss（排序损失）

当标注数据是 **排序列表** 而非成对比较时（例如标注者对 K 个回复进行排序），使用 Plackett-Luce 模型：

```
L(θ) = -E[ log( exp(r_θ(x, y_1)) / Σ_j exp(r_θ(x, y_j)) ) ]
```

其中 y_1 > y_2 > ... > y_K 是排序后的回复。这个 loss 可以看作是多分类的 softmax 交叉熵——模型需要学会将最高分分配给排名最靠前的回复。

Plackett-Luce Loss 的优势在于利用了一个标注样本中的全部排序信息，比拆分成多个成对比较更高效。缺点是计算复杂度为 O(K²)，K 较大时成本增加。

#### 比较损失变体

| 变体 | 公式 | 特点 |
|------|------|------|
| **Margin Loss** | L = max(0, margin - (r_w - r_l)) | 强制偏好差距至少为 margin，训练更稳定 |
| **Cross-Entropy (pairwise)** | L = -[p*log(σ(r_w - r_l)) + (1-p)*log(1 - σ(r_w - r_l))] | 与 BT 等价，但形式上更接近分类 |
| **KTO Loss** | L = λ * (1 - σ(r - offset)) 对坏回复 | 仅需"好/坏"标签，无需成对比较 |
| **ListMLE** | L = -Σ log P(y_i | y_{1..i}) | Listwise 排序学习 |

实践中 Bradley-Terry Loss 仍然是主流选择，因为它简单、稳定且与人类偏好数据的成对比较标注方式天然匹配。

---

### 3. Process Reward Models (PRM): step-level rewards instead of outcome-level, Math-Shepherd, MiPS

#### 为什么需要过程奖励

对于数学推理、代码生成、多步规划等任务，**结果奖励（outcome reward）** 有两个根本缺陷：

1. **稀疏性**：只在最终步骤给出奖励，中间步骤没有训练信号
2. **延迟性**：模型在第一步就走错了，但在最后一步才知道——错误的根源被掩盖
3. **无法定位**：最终答案错误时，无法知道哪一步出错

过程奖励模型（Process Reward Model, PRM）为推理链中的 **每一步** 分配奖励，提供密集的训练信号。

#### PRM 标注方式

| 方法 | 标注方式 | 优点 | 缺点 |
|------|---------|------|------|
| **人工逐步骤标注** | 标注者判断每一步是否正确 | 质量高 | 成本极高，难以规模化 |
| **自动标注 (Math-Shepherd)** | 用 MC 搜索自动生成过程标签 | 可扩展 | 自动标签有噪声 |
| **MiPS (Monte Carlo Process Supervision)** | 从完成路径中采样估计步骤质量 | 无人工标注 | 依赖采样质量 |

#### Math-Shepherd

Math-Shepherd 的核心思想是用 **蒙特卡洛估计** 来自动生成过程标签：

```
对于推理步骤 s_t:
  1. 从步骤 s_t 出发采样 N 条完成的推理路径
  2. 统计这 N 条路径中最终答案正确的比例
  3. 这个比例作为步骤 s_t 的"正确性"标签
```

关键洞察：如果一个步骤是"好的"（推理方向正确），那么从它继续推理得到正确答案的概率应该更高。Math-Shepherd 正是利用这一点，用未来成功概率来标注当前步骤质量。

优势：完全自动化，不需要人工标注每一步。
缺点：蒙特卡洛估计有方差，在推理分支多的场景噪声较大。

#### MiPS (Monte Carlo Process Supervision)

MiPS 是对 Math-Shepherd 的改进，核心区别在于：

- **Math-Shepherd**：从每个步骤采样未来的完整轨迹，统计成功率
- **MiPS**：使用加权蒙特卡洛估计，考虑不同完成路径的质量差异，减少估计方差

MiPS 的另一个关键改进是 **value-based PRM**——不是简单二分类"步骤正确/错误"，而是输出连续值表示步骤质量，提供更丰富的训练信号。

#### PRM 训练流程

```
Step 1: 收集推理链数据（模型生成 + 标注或自动标注步骤标签）
Step 2: 将 PRM 附加在推理模型的每一步
Step 3: 对每个步骤计算损失（BT loss 或 CE loss over step labels）
Step 4: 训练 PRM 区分正确/错误的中间步骤
Step 5: 在推理时，PRM 在每个步骤提供奖励信号
```

---

### 4. Reward Model Evaluation: accuracy, calibration, consistency, adversarial evaluation

RM 的评估远比策略模型的评估复杂——我们不能简单地看"RM 判断是否正确"，因为人类偏好本身就有主观性。需要从多个维度评估：

#### 评估指标

| 指标 | 定义 | 测量内容 |
|------|------|---------|
| **Accuracy** | 成对比较中 RM 判断与人类一致的比例 | 基础排序能力 |
| **Calibration** | RM 预测的偏好概率与实际人类一致概率的匹配度 | 置信度校准 |
| **Consistency** | 在相似输入上 RM 给出相似评分 | 输出稳定性 |
| **Rank Correlation** | Spearman/Kendall tau 在排序上的相关性 | 排序一致性 |
| **Adversarial Robustness** | 对对抗性/边缘案例的鲁棒性 | 泛化与安全 |

#### 评估方法

**1. 留出集评估（Hold-out Evaluation）**

将标注的偏好数据分为训练集和测试集，在测试集上计算 RM 的成对比较准确率。这是最基本的评估方式，但无法反映 RM 在分布外的表现。

**2. 对抗性评估（Adversarial Evaluation）**

主动构造挑战性样本来测试 RM：

- **边界案例**：可能正确可能错误、人类标注者都难以判断的样本
- **Reward Hacking 样本**：已知策略模型可能产生的"投机取巧"的回复
- **分布外输入**：训练数据中未出现的提示类型或回复风格
- **有偏样本**：测试 RM 是否对某些特征（如回复长度、特定词汇）有系统性偏好

**3. 校准评估（Calibration Evaluation）**

检查 RM 的置信度是否准确。将 RM 的预测分数分组（例如分 10 个桶），计算每组中 RM 判断与实际人类偏好的一致性比例。理想情况下，这应该是对角线关系。

**4. 下游任务评估（Downstream Evaluation）**

最实际的评估方式：用 RM 训练策略模型，然后评估最终对齐效果。这包括：

- 人工评估：标注者比较 PPO 前后模型输出的质量
- 自动化评估：在标准 benchmark 上测量（如 MT-Bench, Chatbot Arena）
- 安全评估：测量有害内容生成率、拒绝率等

#### 常见评估陷阱

| 陷阱 | 表现 | 后果 |
|------|------|------|
| **Accuracy 虚高** | 在简单样本上准确率高，复杂样本上差 | 选出的"最佳 RM"在实际使用中表现差 |
| **Length Bias** | RM 偏好更长的回复 | 优化方向偏离真实质量 |
| **Distributional Mismatch** | RM 在训练数据上校准好，在策略模型输出上好 | RM 对策略模型优化方向的引导不可靠 |

---

### 5. Reward Hacking & Over-optimization: Goodhart's law, reward model generalization, over-optimization detection

#### Goodhart's Law 在 RLHF 中的体现

"当一个指标成为目标时，它就不再是一个好指标。"在 RLHF 语境下：

1. **RM 分数是代理指标**：它试图衡量"人类偏好"，但这只是真实偏好的一个投影
2. **PPO 直接优化 RM 分数**：策略模型不断寻找 RM 分数高的回复
3. **RM 分数与实际质量脱钩**：策略模型找到 RM 高估但实际质量低的回复

这就是 **reward over-optimization（奖励过度优化）** 的核心机制。

#### Reward Hacking 的典型模式

| 模式 | 描述 | 示例 |
|------|------|------|
| **Length Exploitation** | 模型生成更长更冗长的回复 | RM 偏好长回复，模型无限堆砌词语 |
| **Stylistic Overfitting** | 模型学习特定句式/词汇模板 | 回复都以"首先，我很感谢您的提问"开头 |
| **Sycophancy** | 模型一味迎合用户而非给出真实回答 | 用户有错误前提时，模型选择附和 |
| **Safety Bypassing** | 模型用无害但无用的方式回应所有请求 | 对任何请求都以"我无法回答"拒绝 |
| **Format Exploitation** | 模型利用 RM 对特定格式的偏好 | 大量使用列表、标题、加粗等格式 |

#### Over-optimization 检测方法

**1. Reward vs KL Divergence 监控**

在 PPO 训练中同时跟踪奖励分数和 KL 散度（策略模型与 SFT 模型的差异）。当奖励持续上升但 KL 散度快速增大时，是 over-optimization 的典型信号。

```
监控指标：
- RM 平均分
- KL(π_θ || π_SFT)
- 两者比值（奖励增益 ÷ KL 成本）
- 如果奖励增益的速度 > KL 增长速度 → over-optimization
```

**2. 黄金 Reward 评估**

定期用额外的、未参与训练的"黄金 RM"或人工评估对策略模型输出进行评分。如果主 RM 分数持续上升但黄金 RM/人工评分停止上升或下降，说明 over-optimization 发生了。

**3. 对抗性检测**

检查策略模型是否生成训练 RM 时未见过的回复模式。具体方法：

- 分析回复长度变化的趋势
- 检测重复性词汇/句式
- 计算策略模型输出的多样性（n-gram diversity）

**4. Early Stopping 策略**

最实用的方法：训练过程中定期保存策略模型 checkpoint，在对齐效果开始下降之前停止。关键在于建立一个可靠的验证指标来确定"什么是最佳停止点"。

#### 缓解策略

| 策略 | 描述 | 效果 |
|------|------|------|
| **KL Penalty** | PPO 目标中加入 KL 散度惩罚 | 最常用，但惩罚系数难调 |
| **Reward Scaling** | 对 RM 分数进行裁剪或缩放 | 简单但效果有限 |
| **Ensemble** | 使用多个 RM 取平均/最小 | 减少单 RM 的脆弱性 |
| **Preference Buffer** | 维护偏好数据缓冲区，定期标注新数据 | 成本高但效果好 |
| **Adversarial Training** | 在对抗样本上微调 RM | 增强 RM 鲁棒性 |

---

### 6. Reward Model Ensembles: uncertainty estimation, disagreement as signal, aggregation strategies

#### 为什么需要 Ensemble

单个 RM 的预测存在盲点和过拟合风险。Ensemble 通过多个 RM 的集体决策提供更可靠的奖励信号，同时给出不确定性估计。

#### Ensemble 构建方式

| 方法 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **不同随机种子初始化** | 相同架构 + 不同初始化训练多个 RM | 简单易行 | 多样性有限 |
| **不同数据子集** | Bootstrap 采样训练每个 RM | 多样性更好 | 每个 RM 数据利用率略低 |
| **不同架构** | 不同大小的基座模型 | 架构多样性 | 训练成本高 |
| **Head Ensemble** | 单基座模型 + 多个 value head | 计算高效 | head 之间相关性高 |

#### 不确定度估计

Ensemble 的核心价值之一是量化 RM 的置信度：

- **预测均值**：μ = mean(r₁, r₂, ..., rₙ) — 最终奖励分数
- **预测方差**：σ² = var(r₁, r₂, ..., rₙ) — 不确定度估计
- **熵**：H = -Σ p_i log p_i — 模型间的分歧程度

不确定度的使用场景：

| 应用 | 不确定度使用方法 |
|------|----------------|
| **PPO 训练** | 在 RM 不确定度高时降低学习率或使用保守奖励 |
| **数据收集** | 选择 RM 高度不确定的样本来标注，主动学习 |
| **安全拦截** | 在不确定度高时启用人工审核或拒绝生成 |
| **模型改进** | 分析不确定度高的区域，针对性收集训练数据 |

#### 聚合策略

| 策略 | 公式 | 特性 |
|------|------|------|
| **Mean Aggregation** | r = mean(r₁, ..., rₙ) | 简单，但易被极端值影响 |
| **Median Aggregation** | r = median(r₁, ..., rₙ) | 对异常值鲁棒 |
| **Minimum Aggregation** | r = min(r₁, ..., rₙ) | 保守策略，适合安全敏感场景 |
| **Weighted Aggregation** | r = Σ w_i r_i | 可给验证集上表现好的 RM 更高权重 |
| **Pessimistic Aggregation** | r = μ - λσ | 降低不确定性高的输出分数 |
| **Soft Minimum** | 类似 LogSumExp | 在保守和稳定间权衡 |

**实践中 Minimum Aggregation** 在安全性优先的场景（如对话助手的安全性评估）中表现优异——只要有一个 RM 认为某个回复有问题，就降低该回复的奖励。而 **Mean Aggregation** 在有用性评估中更常见。

---

### 7. Multi-Objective Rewards: combining helpfulness + harmlessness + honesty, reward shaping

#### 多目标奖励的必要性

人类对回复的评价是多维的。一个回复可能非常有用（helpful）但在某些方面不无害（harmless），或很安全但完全没用。将所有这些维度压缩成一个标量奖励会导致信息丢失。

#### 多目标奖励分解

将奖励函数分解为多个子奖励：

```
总奖励 = w_helpful * R_helpful(x, y) + w_harmless * R_harmless(x, y) + w_honest * R_honest(x, y)
```

每个子奖励对应一个独立的 RM 或子模块，分别训练用于评估特定维度。

#### 权重选择策略

| 方法 | 描述 | 适用场景 |
|------|------|---------|
| **固定权重** | 人工设定 w | 目标明确，权重有先验知识 |
| **动态权重** | 根据上下文/用户调整 | 个性化需求 |
| **Pareto Optimization** | 不设权重，优化 Pareto 前沿 | 需要探索多种权衡 |
| **Constraint Satisfaction** | 某些目标作为硬约束 | 安全底线要求 |
| **MO-MI (Multi-Objective via Meta-Learning)** | 学习权重的生成器 | 需要自动化权重调整 |

#### Reward Shaping

Reward Shaping 是在原始奖励基础上添加辅助信号来引导学习，不改变最优策略但加速收敛：

- **Potential-based Shaping**：F(s, s') = γΦ(s') - Φ(s)，保证最优策略不变
- **安全约束**：在违反安全规则时加入大额惩罚
- **中间奖励**：在完成子目标时给予额外奖励
- **多样性奖励**：鼓励策略模型输出多样性回复

#### 多目标打分在实践中

实际工程中，多目标 RM 有两种实现路线：

**路线 A：独立 RM 权重组合**
```
训练独立的 Helpfulness RM, Harmlessness RM, Honesty RM
→ 各自输出分数
→ 加权组合为总奖励
→ 优点：可独立优化每个维度，权重灵活调整
→ 缺点：训练成本是 N 倍
```

**路线 B：多任务 Value Head**
```
单基座模型 + 多个 value head（每个目标一个）
→ 共享基座，独立 head
→ 每个 head 在各自的数据上训练
→ 组合各 head 输出
→ 优点：训练成本低，共享语义理解
→ 缺点：基座可能无法同时服务所有目标
```

---

### 8. Scaling Laws for Reward Models: RM size vs policy performance, data scaling

#### RM 大小与策略性能

InstructGPT 的研究发现了明确的 scaling 关系：

- **RM 能力随模型大小提升**：更大的 RM 在偏好预测准确率上持续提升
- **RM 需要与策略模型大小匹配**：训练 175B 策略模型时，6B RM 的效果显著差于 175B RM
- **边际收益递减**：RM 超过一定大小后，继续增大带来的收益减少

```
策略模型大小  │  推荐 RM 大小下限  │  说明
1B           │  300M - 1B       │  小模型对 RM 精度要求较低
7B           │  1B - 7B         │  RM 大小至少为策略模型的 1/3
70B          │  7B - 70B        │  RM 越大越好，但性价比下降
175B+        │  70B+            │  需要足够大的 RM 来提供细致偏好信号
```

#### 数据 Scaling

偏好数据的数量和质量都影响 RM 性能：

| 数据维度 | 影响 | 发现 |
|---------|------|------|
| **数据量** | Accuracy 随数据量对数增长 | 数据量翻倍，准确率提升 1-2% |
| **数据多样性** | 覆盖更多偏好维度 | 比单纯加数据量更有效 |
| **标注质量** | 直接影响 RM 上限 | 一个高质量标注 > 10 个低质量标注 |
| **分布匹配** | 训练数据与目标分布越匹配越好 | 标注数据的提示分布需覆盖目标场景 |

#### Scalable Oversight 问题

当任务超出人类评价能力（如总结一本比人类寿命还长的书）时，RM 的 supervision 出现问题。Scalable Oversight 研究如何在这种情况下依然能训练可靠的 RM：

- **Debate**：两个 AI 辩论，人类判断哪方更有道理
- **Iterated Amplification**：将复杂任务分解为多个子任务
- **Weak-to-Strong Generalization**：弱模型监督强模型
- **Recursive Reward Modeling**：AI 辅助人类标注偏好数据

---

## Example Code: Python reward model training with HuggingFace, Bradley-Terry loss, evaluation

以下是用 PyTorch + HuggingFace Transformers 训练奖励模型的完整流程。

### 1. 数据准备

```python
import torch
import torch.nn as nn
from transformers import AutoModel, AutoTokenizer
from datasets import Dataset
from torch.utils.data import DataLoader
import numpy as np

# 偏好数据格式：[{prompt, chosen, rejected}, ...]
# chosen 是被偏好的回复, rejected 是被拒绝的回复
preference_data = [
    {
        "prompt": "Explain quantum computing",
        "chosen": "Quantum computing uses qubits... （高质量回复）",
        "rejected": "Quantum computing is hard... （低质量回复）",
    },
    # ... 更多数据
]

def tokenize_pair(batch):
    """对 (prompt, chosen) 和 (prompt, rejected) 分别 tokenize"""
    chosen_inputs = tokenizer(
        [p + " " + c for p, c in zip(batch["prompt"], batch["chosen"])],
        padding=True, truncation=True, max_length=512, return_tensors="pt"
    )
    rejected_inputs = tokenizer(
        [p + " " + r for p, r in zip(batch["prompt"], batch["rejected"])],
        padding=True, truncation=True, max_length=512, return_tensors="pt"
    )
    return {
        "chosen_input_ids": chosen_inputs["input_ids"],
        "chosen_attention_mask": chosen_inputs["attention_mask"],
        "rejected_input_ids": rejected_inputs["input_ids"],
        "rejected_attention_mask": rejected_inputs["attention_mask"],
    }

dataset = Dataset.from_list(preference_data).map(tokenize_pair, batched=True)
```

### 2. Reward Model 定义

```python
class RewardModel(nn.Module):
    """在 LM 基座上加 value head 的奖励模型"""

    def __init__(self, base_model_name: str, value_head_dropout: float = 0.1):
        super().__init__()
        self.base_model = AutoModel.from_pretrained(base_model_name)
        hidden_size = self.base_model.config.hidden_size

        # Value Head: 将 hidden state 映射为标量奖励
        self.value_head = nn.Sequential(
            nn.Dropout(value_head_dropout),
            nn.Linear(hidden_size, hidden_size // 2),
            nn.GELU(),
            nn.Dropout(value_head_dropout),
            nn.Linear(hidden_size // 2, 1),
        )

    def forward(self, input_ids, attention_mask):
        outputs = self.base_model(
            input_ids=input_ids,
            attention_mask=attention_mask,
            output_hidden_states=True,
        )
        # 取最后一个 token 的 hidden state 作为序列表示
        last_hidden = outputs.hidden_states[-1]  # [batch, seq_len, hidden]
        # 用 attention_mask 找到每个序列的最后一个 token
        last_indices = attention_mask.sum(dim=1) - 1  # [batch]
        batch_indices = torch.arange(last_hidden.size(0))
        sequence_repr = last_hidden[batch_indices, last_indices]  # [batch, hidden]

        reward = self.value_head(sequence_repr)  # [batch, 1]
        return reward.squeeze(-1)  # [batch]
```

### 3. Bradley-Terry Loss

```python
def bradley_terry_loss(chosen_rewards: torch.Tensor, rejected_rewards: torch.Tensor) -> torch.Tensor:
    """Bradley-Terry 成对偏好损失

    L = -log(σ(r_chosen - r_rejected))
    """
    logits = chosen_rewards - rejected_rewards  # 偏好差距
    loss = -nn.functional.logsigmoid(logits).mean()
    return loss


def plackett_luce_loss(rewards: torch.Tensor, rankings: torch.Tensor) -> torch.Tensor:
    """Plackett-Luce 排序损失

    rewards: [batch, num_candidates] — 每个候选回复的奖励分数
    rankings: [batch, num_candidates] — 排序标签（1=最好, K=最差）
    """
    # 按排名排序奖励
    sorted_indices = rankings.argsort(dim=-1)
    sorted_rewards = rewards.gather(dim=-1, index=sorted_indices)

    # 累积 softmax 损失
    batch_size, num_candidates = rewards.shape
    loss = 0.0
    for k in range(num_candidates - 1):
        exp_rewards = torch.exp(sorted_rewards[:, k:])  # 当前候选及之后的奖励
        softmax_denom = exp_rewards.sum(dim=-1)  # 归一化常数
        log_prob = sorted_rewards[:, k] - torch.log(softmax_denom)  # log(softmax)
        loss -= log_prob.mean()

    return loss / (num_candidates - 1)
```

### 4. 训练循环

```python
# 初始化模型
model = RewardModel("microsoft/deberta-v3-base")  # 基座可选: roberta, electra, deberta, llama
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-5)
train_loader = DataLoader(dataset, batch_size=8, shuffle=True)

model.train()
for epoch in range(3):
    epoch_loss = 0.0
    for batch in train_loader:
        chosen_rewards = model(
            batch["chosen_input_ids"],
            batch["chosen_attention_mask"],
        )
        rejected_rewards = model(
            batch["rejected_input_ids"],
            batch["rejected_attention_mask"],
        )

        loss = bradley_terry_loss(chosen_rewards, rejected_rewards)

        optimizer.zero_grad()
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)  # 梯度裁剪稳定训练
        optimizer.step()

        epoch_loss += loss.item()

    print(f"Epoch {epoch}, Loss: {epoch_loss / len(train_loader):.4f}")
```

### 5. 评估

```python
def evaluate_rm(model, eval_dataset):
    """评估奖励模型的成对比较准确率"""
    model.eval()
    correct = 0
    total = 0

    with torch.no_grad():
        for batch in eval_loader:
            chosen_rewards = model(batch["chosen_input_ids"], batch["chosen_attention_mask"])
            rejected_rewards = model(batch["rejected_input_ids"], batch["rejected_attention_mask"])

            # 如果 chosen_reward > rejected_reward，则判断正确
            predictions = (chosen_rewards > rejected_rewards).float()
            correct += predictions.sum().item()
            total += predictions.size(0)

    accuracy = correct / total
    return accuracy


# 校准评估
def compute_calibration(model, eval_dataset, num_bins=10):
    """计算 RM 的校准误差 (ECE)"""
    model.eval()
    all_logits = []  # r_chosen - r_rejected
    all_correct = []

    with torch.no_grad():
        for batch in eval_loader:
            chosen_rewards = model(batch["chosen_input_ids"], batch["chosen_attention_mask"])
            rejected_rewards = model(batch["rejected_input_ids"], batch["rejected_attention_mask"])

            logits = chosen_rewards - rejected_rewards
            correct = (chosen_rewards > rejected_rewards).float()

            all_logits.extend(torch.sigmoid(logits).tolist())
            all_correct.extend(correct.tolist())

    # 分桶计算校准误差
    bins = np.linspace(0, 1, num_bins + 1)
    ece = 0.0
    for i in range(num_bins):
        mask = (np.array(all_logits) >= bins[i]) & (np.array(all_logits) < bins[i + 1])
        if mask.sum() > 0:
            bin_acc = np.mean(np.array(all_correct)[mask])
            bin_conf = np.mean(np.array(all_logits)[mask])
            ece += np.abs(bin_acc - bin_conf) * (mask.sum() / len(all_logits))

    return ece
```

### 6. 训练实用技巧

```python
# 技巧 1: Reward Normalization（奖励归一化）
def normalize_rewards(rewards, eps=1e-8):
    """在 batch 内进行奖励归一化，稳定训练"""
    mean = rewards.mean()
    std = rewards.std() + eps
    return (rewards - mean) / std

# 在 loss 计算前使用：
# chosen_rewards = normalize_rewards(chosen_rewards)
# rejected_rewards = normalize_rewards(rejected_rewards)

# 技巧 2: Label Smoothing（标签平滑）
def bradley_terry_loss_smoothed(chosen_rewards, rejected_rewards, smoothing=0.1):
    """带标签平滑的 BT loss，减少过拟合"""
    logits = chosen_rewards - rejected_rewards
    # 标准 BT loss
    loss_bt = -nn.functional.logsigmoid(logits)
    # 标签平滑：将部分置信度分配给相反类别
    loss_smoothed = (1 - smoothing) * loss_bt + smoothing * (-nn.functional.logsigmoid(-logits))
    return loss_smoothed.mean()
```

---

## Capability Boundaries: reward model can't capture all human preferences, calibration degrades OOD

### RM 无法捕获所有人类偏好

| 限制 | 说明 | 根本原因 |
|------|------|---------|
| **标量奖励的信息瓶颈** | 一个标量无法编码多维偏好 | 信息压缩导致损失 |
| **标注者不一致性** | 不同标注者偏好不同，RM 学的是"平均" | 人类偏好本质上有分歧 |
| **无法捕获依赖上下文的偏好** | 同一回复在不同场景下评价不同 | 偏好是上下文相关的 |
| **无法反映偏好强度** | 标注者只说"A > B"但不说明"好多少" | 偏好数据只有序数信息 |
| **反馈滞后** | 模型行为改变后人类偏好可能改变 | 偏好的动态性 |

### OOD 校准退化

RM 在分布外（Out-of-Distribution）数据上的校准质量会严重下降：

```
校准误差 (ECE) 随分布偏移的变化趋势：

      ECE
      0.3 ┤                          ●
      0.2 ┤              ●
      0.1 ┤  ●
      0.0 ┼───────────────────────────────
                 In-dist        Near-OOD        Far-OOD
```

关键发现：
1. **Near-OOD**（策略模型探索到的但与训练数据类似的回复）：校准误差开始增大
2. **Far-OOD**（策略模型产生的新颖回复模式）：校准误差严重，RM 不可靠
3. **单调性假设失效**：RM 分数的单调性（高分的应该更好）在 OOD 区域可能完全失效

应对策略：
- 在训练中覆盖更广泛的回复空间
- 使用 ensemble 量化不确定性
- 在训练过程中持续更新偏好数据（迭代 RM）
- 在 RM 不确定时降低奖励信号的权重

---

## Comparison: outcome RMs vs process RMs — when each is appropriate

| 维度 | Outcome RM | Process RM (PRM) |
|------|-----------|------------------|
| **奖励粒度** | 整个回复一个分数 | 每步一个分数 |
| **训练数据需求** | 偏好对（A > B） | 步骤级标注或自动标签 |
| **训练成本** | 较低 | 较高（步骤级标注或采样） |
| **推理成本** | 单次前向传播 | 每步都需要前向传播 |
| **信号密度** | 稀疏（仅在最终输出） | 密集（每步都有信号） |
| **错误定位能力** | 无（不知道哪里错了） | 有（可定位到具体步骤） |
| **适用任务** | 开放式对话、创意写作 | 数学推理、代码生成、多步规划 |
| **对策略模型的引导** | 终点导向优化 | 过程导向优化 |
| **reward hacking 风险** | 更高（模型可投机取巧） | 较低（每步被监督） |
| **实现复杂度** | 低 | 高 |

### 选择指南

**使用 Outcome RM 的情况：**
- 任务结果是唯一评价标准（如摘要质量评估）
- 回复较短，没有明显的"步骤"结构
- 偏好数据只有成对比较，没有步骤级标注
- 对推理成本敏感
- 开放式生成任务（创意写作、对话）

**使用 Process RM 的情况：**
- 任务有明显步骤结构（数学推理、代码生成）
- 需要定位错误发生的具体步骤
- 中间步骤的正确性对最终结果至关重要
- 可以承受更高的训练和推理成本
- 需要更细致的奖励信号来引导策略模型

**混合方案**：实践中常用 outcome RM + PRM 的组合，在最终结果和过程步骤上同时提供信号，兼顾精度和密度。

---

## Engineering Optimization: RM ensemble, active data selection, reward normalization, preference buffer management

### 1. RM Ensemble 工程实现

```python
class RewardModelEnsemble:
    """RM Ensemble：多模型聚合 + 不确定性估计"""

    def __init__(self, models: list[RewardModel]):
        self.models = models

    def predict(self, input_ids, attention_mask, agg_strategy="mean"):
        """进行集成预测"""
        all_rewards = []
        for model in self.models:
            model.eval()
            with torch.no_grad():
                rewards = model(input_ids, attention_mask)
                all_rewards.append(rewards)

        # [num_models, batch_size]
        all_rewards = torch.stack(all_rewards)

        mean_rewards = all_rewards.mean(dim=0)      # 均值
        std_rewards = all_rewards.std(dim=0)         # 不确定度

        if agg_strategy == "mean":
            return mean_rewards, std_rewards
        elif agg_strategy == "min":
            return all_rewards.min(dim=0).values, std_rewards
        elif agg_strategy == "pessimistic":
            return mean_rewards - 1.0 * std_rewards, std_rewards
```

### 2. 主动数据选择（Active Data Selection）

在偏好数据有限时，主动选择最有价值的样本来标注可以大幅提升 RM 效率：

```python
def active_selection_candidates(
    rm_ensemble: RewardModelEnsemble,
    candidate_pairs: list,
    strategy: str = "uncertainty",
    top_k: int = 1000,
) -> list:
    """从不标注数据中选择最有价值的候选偏好对"""
    uncertainties = []

    for prompt, response_a, response_b in candidate_pairs:
        inputs_a = tokenizer(prompt + response_a, return_tensors="pt")
        inputs_b = tokenizer(prompt + response_b, return_tensors="pt")

        _, std_a = rm_ensemble.predict(inputs_a["input_ids"], inputs_a["attention_mask"])
        _, std_b = rm_ensemble.predict(inputs_b["input_ids"], inputs_b["attention_mask"])

        if strategy == "uncertainty":
            # 选择 RM 间分歧最大的样本
            score = std_a.item() + std_b.item()
        elif strategy == "disagreement":
            # 选择 RM 间对"A是否优于B"的判断不一致的样本
            score = abs(std_a.item() - std_b.item())  # 简化示意
        elif strategy == "diversity":
            # 选择与已标注数据最不相似的样本
            score = compute_diversity_score(prompt, labeled_data)

        uncertainties.append((prompt, response_a, response_b, score))

    # 取 Top-K 最不确定的样本
    uncertainties.sort(key=lambda x: x[-1], reverse=True)
    return uncertainties[:top_k]
```

### 3. 奖励归一化策略

奖励分数的分布在整个训练过程中会漂移，归一化对稳定训练至关重要：

| 归一化方法 | 计算方式 | 适用场景 |
|-----------|---------|---------|
| **Batch Normalization** | (r - μ_batch) / σ_batch | 在线、实时；依赖 batch 大小 |
| **Running Statistics** | (r - μ_running) / σ_running | 平滑、不依赖 batch |
| **Percentile Scaling** | 映射到 [0, 1] 基于分位数 | 鲁棒到异常值 |
| **Adaptive Normalization** | 根据 KL 散度动态调整尺度 | PPO 训练中效果最好 |

```python
class AdaptiveRewardNormalizer:
    """自适应的奖励归一化器（在 PPO 循环中使用）"""

    def __init__(self, momentum=0.99, percentile_low=5, percentile_high=95):
        self.momentum = momentum
        self.percentile_low = percentile_low
        self.percentile_high = percentile_high
        self.running_mean = 0.0
        self.running_std = 1.0
        self.reward_history = []

    def update(self, rewards: torch.Tensor):
        """更新滑动统计量"""
        self.reward_history.extend(rewards.detach().cpu().tolist())
        # 保持历史长度有限
        if len(self.reward_history) > 10000:
            self.reward_history = self.reward_history[-5000:]

        new_mean = np.mean(self.reward_history)
        new_std = np.std(self.reward_history) + 1e-8

        self.running_mean = self.momentum * self.running_mean + (1 - self.momentum) * new_mean
        self.running_std = self.momentum * self.running_std + (1 - self.momentum) * new_std

    def normalize(self, rewards: torch.Tensor) -> torch.Tensor:
        """应用归一化"""
        return (rewards - self.running_mean) / self.running_std

    def denormalize(self, normalized_rewards: torch.Tensor) -> torch.Tensor:
        """逆归一化（用于 logging）"""
        return normalized_rewards * self.running_std + self.running_mean
```

### 4. 偏好缓冲区管理（Preference Buffer Management）

PPO 训练过程中，策略模型的输出分布持续变化。维护一个动态更新的偏好数据缓冲区对 RM 质量至关重要：

```python
class PreferenceBuffer:
    """
    动态偏好数据缓冲区
    - 存储 (prompt, response_a, response_b, preference_label)
    - 支持 FIFO 淘汰
    - 支持基于分布匹配的采样
    """

    def __init__(self, max_size: int = 50000):
        self.max_size = max_size
        self.buffer = []

    def add(self, prompt: str, response_a: str, response_b: str, label: int):
        """添加新的偏好数据点 (label=1 表示 a > b)"""
        self.buffer.append({
            "prompt": prompt,
            "response_a": response_a,
            "response_b": response_b,
            "label": label,
            "timestamp": time.time(),
        })
        # FIFO 淘汰
        if len(self.buffer) > self.max_size:
            self.buffer.pop(0)

    def sample(self, batch_size: int, strategy: str = "recent") -> list:
        """根据策略采样训练数据"""
        if strategy == "recent":
            # 优先使用最新的数据（与当前策略分布最匹配）
            return self.buffer[-batch_size:]
        elif strategy == "stratified":
            # 分层采样：70% 最新 + 30% 历史
            recent = self.buffer[-int(batch_size * 0.7):]
            historical = random.choices(
                self.buffer[:-int(batch_size * 0.7)] or self.buffer,
                k=batch_size - len(recent),
            )
            return recent + historical
        elif strategy == "weighted":
            # 按时间衰减权重采样
            now = time.time()
            weights = [max(0.01, 1.0 - (now - d["timestamp"]) / 86400) for d in self.buffer]
            return random.choices(self.buffer, weights=weights, k=batch_size)

    def compute_distribution_similarity(self, current_prompts: list) -> float:
        """计算当前 prompt 分布与缓冲区 prompt 分布的相似度"""
        # 用 n-gram 或 embedding 近似分布距离
        # 相似度低意味着需要补充新数据
        pass
```

### 5. 完整 PPO + RM 训练管线建议

```
┌──────────────────────────────────────────────────────┐
│              PPO Training Loop                       │
│                                                      │
│  Policy Model ──生成回复──► Reward Model ──评分──► PPO Update  │
│       ▲                      │                        │
│       │                      ▼                        │
│       └─── KL Penalty ── Reward Normalization ────────┘
│       │                                               │
│       ▼                                               │
│  Preference Buffer ──► RM Fine-tuning (periodic)      │
│                                                      │
│  Active Selection ──► Human Labeling ──► New Pairs    │
└──────────────────────────────────────────────────────┘
```

工程最佳实践：

| 实践 | 描述 | 效果 |
|------|------|------|
| **定期重训 RM** | 每 N 步 PPO 后，用当前策略模型采样新数据重训 RM | 缓解分布偏移 |
| **Reward Scaling** | 将 RM 分数缩放到合理范围（如 [-10, 10]） | 稳定 PPO 训练 |
| **KL 惩罚自适应** | 根据 KL 散度动态调整惩罚系数 | 防止 over-optimization |
| **梯度裁剪** | 限制 RM 训练梯度范数 | 防止崩塌 |
| **Preference Buffer 平衡** | 保持不同来源数据的比例 | 防止遗忘 |
| **多维度监控** | 跟踪 RM 准确率、校准误差、KL 散度 | 早期检测问题 |
