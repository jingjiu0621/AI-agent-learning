# 12.5.2 DPO — 直接偏好优化

## 简单介绍

直接偏好优化（Direct Preference Optimization, DPO）是 Rafailov 等人于 2023 年提出的对齐方法，核心思想是**直接利用偏好数据优化策略，无需训练独立的奖励模型**。DPO 通过数学变换将 RLHF 的强化学习目标重写为简单的二元交叉熵损失，在保持对齐效果的同时大幅降低训练复杂度。

相比 RLHF（需要 3 阶段：SFT + 奖励建模 + PPO），DPO 只需要 2 阶段：**SFT + DPO 训练**，且 DPO 阶段仅需一个简单的损失函数，不需要 reward model 在线推理、不需要 actor/critic 双网络、不需要 KL 散度自适应调整。这种简洁性使 DPO 迅速成为最广泛采用的偏好优化方法。

## 基本原理 — DPO reformulates RLHF objective to directly optimize from pairwise preferences without reward model

DPO 的核心洞察是：**RLHF 中奖励模型的最优策略可以解析表达为奖励函数的形式**。这一发现打破了"必须先有奖励模型才能优化策略"的思维定势。

### 数学推导核心三步

```
Step 1: RLHF 目标函数
─────────────────────────────────────────────────────────────────────
  max E[r(x,y)] - β · KL(π_θ || π_ref)
   π_θ

  其中：r(x,y) 是隐式奖励，β 控制 KL 惩罚强度
        π_θ 是策略模型，π_ref 是参考模型（通常是 SFT 模型）

Step 2: 最优策略的解析解
─────────────────────────────────────────────────────────────────────
  奖励函数可以用最优策略表示：
  
  r(x,y) = β · log(π_θ(y|x) / π_ref(y|x)) + β · log Z(x)

  其中 Z(x) 是配分函数，是输入 x 的函数（在 Bradley-Terry 模型中消去）

Step 3: 代入 Bradley-Terry 偏好模型
─────────────────────────────────────────────────────────────────────
  P(y_w > y_l | x) = σ(r(x,y_w) - r(x,y_l))
                   = σ(β · log(π_θ(y_w|x)/π_ref(y_w|x)) - β · log(π_θ(y_l|x)/π_ref(y_l|x)))

  最终 DPO 损失函数：
  L_DPO(π_θ; π_ref) = -E[ log σ( β · log(π_θ(y_w|x)/π_ref(y_w|x)) - β · log(π_θ(y_l|x)/π_ref(y_l|x)) ) ]

  其中 y_w 是被偏好的回答，y_l 是被拒绝的回答
```

### 直观理解

DPO 的损失函数可以理解为：**对偏好回答提高其相对概率，对拒绝回答降低其相对概率**。关键区别在于使用 `log π_θ / log π_ref` 的比值而不是绝对概率，这相当于在参考策略周围保持了一个隐式 KL 约束——模型的优化受到"不与参考模型偏离太远"的限制。

### DPO 与 RLHF 的等价性

| 方面 | RLHF | DPO |
|------|------|-----|
| 奖励函数 | 显式训练一个 Reward Model | 策略的似然比隐式定义奖励 |
| 优化目标 | PPO 最大化奖励 - KL 惩罚 | 闭式解直接优化偏好似然 |
| 策略约束 | PPO 的 clip 机制 + KL 惩罚 | 参考模型的隐式 KL 约束 |
| 数学形式 | 两阶段复杂优化 | 单阶段二元分类 |

## 背景 — DPO paper (NeurIPS 2023, Rafailov et al.) → rapid adoption → variants (IPO, KTO, ORPO, CPO)

### 时间线

```
2023 中期                   2023 底                   2024                      2025-2026
────────                   ──────                   ────                      ────────
DPO 论文                    DPO 成为主流             变体爆发                    工程成熟
(Rafailov et al.)           HuggingFace TRL 集成    IPO / KTO / ORPO / CPO     VERL 支持 70B+
                                    │                       │                        │
NeurIPS 2023                   开源社区快速采用       各方法解决不同缺陷         大规模工业化部署
                                    │                       │                        │
理论贡献：隐式奖励          Zephyr / Mistral 等     SimPO (无参考模型)          多阶段 DPO
闭式解                        模型使用 DPO           SPIN (自我博弈)             在线 DPO
```

### 关键里程碑

1. **DPO 论文**（2023.05 arXiv → 2023.12 NeurIPS）：Rafailov, Sharma, Mitchell 等人提出 DPO，证明偏好优化不需要显式奖励模型
2. **Zephyr-7B**（2023.10）：HuggingFace 使用 DPO 训练，以 7B 参数在 AlpacaEval 上超越 70B 模型，展示 DPO 的有效性
3. **HuggingFace TRL 集成**（2023.11）：`DPOTrainer` 成为最广泛使用的 DPO 训练库
4. **IPO 提出**（2023.10, Azar et al.）：解决 DPO 的过拟合问题
5. **KTO 提出**（2024.02, Ethayarajh et al.）：仅需好/坏标签，不需成对偏好
6. **ORPO 提出**（2024.03, Hong et al.）：联合 SFT + 偏好优化
7. **VERL**（2025）：字节跳动开源支持 70B+ 模型的高性能 DPO 训练框架

## 核心矛盾 — DPO is simpler but has different failure modes than RLHF

### 主要矛盾

| 矛盾 | 说明 |
|------|------|
| **简洁性 vs 鲁棒性** | DPO 更简单，但对数据和超参数更敏感，在低数据质量场景下不如 RLHF 稳定 |
| **隐式约束 vs 显式约束** | DPO 的 KL 约束是隐式的（通过参考模型），无法像 PPO 那样动态调整惩罚强度 |
| **离线 vs 在线** | 标准 DPO 是离线的（固定偏好数据集），无法利用在策略样本；在线 DPO 可缓解但增加复杂度 |
| **偏好优化 vs 分布外泛化** | DPO 在训练分布上有效，但在分布外场景可能退化比 RLHF 更严重 |
| **长度利用** | DPO 倾向学习"更长回答更好"的虚假相关，导致输出冗长 |

### DPO 特有风险

- **隐式奖励无法监测**：因为没有显式奖励模型，不能监控奖励分数变化来检测训练异常
- **偏好数据质量敏感**：一条错误偏好就能在 DPOLoss 中产生很大梯度（相比 PPO 通过 Value Network 和 Advantage 对异常数据更鲁棒）
- **参考模型固定**：标准 DPO 中 π_ref 是固定的，不随训练更新，在某些场景下限制了模型的可优化空间

## 详细内容

### 1. DPO Loss Function Derivation: from RLHF objective to binary cross-entropy over preferences

#### 从 RLHF 目标开始

RLHF 的原始优化目标：

```
max E[r(x,y)] - β · D_KL(π_θ(y|x) || π_ref(y|x))
 π_θ
```

这个目标在策略空间中的**最优解**有已知的闭式表达式：

```
π_θ(y|x) = (1/Z(x)) · π_ref(y|x) · exp(r(x,y) / β)

其中 Z(x) = Σ_y π_ref(y|x) · exp(r(x,y)/β) 是配分函数
```

#### 用策略表示奖励

解出奖励函数 r(x,y)：

```
r(x,y) = β · log(π_θ(y|x) / π_ref(y|x)) + β · log Z(x)
```

#### 代入 Bradley-Terry 模型

Bradley-Terry 模型定义偏好概率：

```
P(y_w > y_l | x) = exp(r(x,y_w)) / (exp(r(x,y_w)) + exp(r(x,y_l)))
                 = σ(r(x,y_w) - r(x,y_l))
```

代入奖励表达式后，配分函数 Z(x) 被消去：

```
P(y_w > y_l | x) = σ(β · log(π_θ(y_w|x)/π_ref(y_w|x)) - β · log(π_θ(y_l|x)/π_ref(y_l|x)))
```

#### 最终损失函数

最大化偏好似然 → 最小化负对数似然：

```
L_DPO(π_θ; π_ref) = -E_{(x,y_w,y_l) ~ D}[ 
    log σ( β · (log π_θ(y_w|x) - log π_ref(y_w|x) - log π_θ(y_l|x) + log π_ref(y_l|x)) )
]
```

定义隐式奖励差值：

```
让 u(x,y_w,y_l) = β · (log π_θ(y_w|x)/π_ref(y_w|x) - log π_θ(y_l|x)/π_ref(y_l|x))
则 L_DPO = -E[ log σ(u) ]
```

对偏好回答的梯度方向：

```
∇L_DPO = -β · E[ σ(-u) · (∇log π_θ(y_w|x) - ∇log π_θ(y_l|x)) ]
```

这个梯度形式的直观解释：
- 当模型已经正确区分偏好时（u 很大），σ(-u) ≈ 0，梯度很小
- 当模型判断错误时（u 很小或为负），σ(-u) ≈ 1，梯度很大
- 梯度同时增加偏好回答的概率并降低拒绝回答的概率

### 2. Reference Model: role of reference model in DPO, preventing overfitting to preferences

#### 参考模型的作用

参考模型 π_ref 在 DPO 中有三个关键作用：

1. **隐式 KL 约束**：通过 `log π_θ / log π_ref` 的比值，防止策略 π_θ 从参考策略漂移太远
2. **防止过拟合到偏好**：模型不能仅仅通过降低拒绝回答的概率来最小化损失，因为这会增大比值差异
3. **维持生成多样性**：限制策略更新幅度，避免模式崩塌（在某个输出上分配过高概率）

#### 参考模型的选择

| 选项 | 做法 | 适用场景 |
|------|------|---------|
| **SFT 模型** | 直接使用 DPO 训练前的 SFT 模型 | 最常用，标准做法 |
| **冻结初始策略** | 在 DPO 开始时保存一份策略副本作为参考 | 当 SFT 模型不可用或需特殊初始化时 |
| **迭代更新** | 每 N 步更新一次参考模型（使用当前策略） | 需要更大探索空间或训练不稳定时 |
| **EMA 模型** | 使用策略的指数移动平均作为参考 | 更平滑的参考分布，训练更稳定 |

#### 参考模型与训练稳定性的关系

```
参考模型过近（β 过小）：
  ─ 约束太弱，模型可能过拟合到偏好数据
  ─ 出现"奖励 hacking"：降低所有非偏好回答的概率而非提高偏好回答的概率

参考模型过远（β 过大）：
  ─ 约束太强，模型无法有效学习偏好
  ─ 对齐效果差，可能退化为 SFT 模型

参考模型对齐错误：
  ─ 当 π_ref 本身已对齐到错误方向时，DPO 难以纠正
  ─ 建议 π_ref 使用中立/通用的 SFT 模型
```

### 3. DPO Training Pipeline: data preparation, training loop, hyperparameter tuning (beta parameter)

#### 完整训练管线

```
Stage 1: SFT（监督微调）
─────────────────────────────────────────────────────────────────────
  数据：高质量指令-回答对（示范数据）
  目标：让模型学会任务格式和基础能力
  输出：SFT 模型（作为 DPO 的 π_ref 和 π_θ 初始点）

Stage 2: 偏好数据收集
─────────────────────────────────────────────────────────────────────
  对每个 prompt，让模型（可以是不同模型）生成多个回答
  人类/自动评估员标注偏好对 (y_w > y_l)
  格式：(prompt, chosen_response, rejected_response)

Stage 3: DPO 训练
─────────────────────────────────────────────────────────────────────
  初始化：π_θ = π_ref = SFT 模型
  循环：
    1. 采样 batch: (x, y_w, y_l) ~ D
    2. 计算 log π_θ(y_w|x), log π_θ(y_l|x)
    3. 计算 log π_ref(y_w|x), log π_ref(y_l|x)  [只需前向，不反向传播]
    4. 计算偏好比值：ratio = β · (logπ_θ(y_w) - logπ_ref(y_w) - logπ_θ(y_l) + logπ_ref(y_l))
    5. 计算损失：loss = -log σ(ratio)
    6. 反向传播更新 π_θ
    7. 可选：每 K 步对 π_ref 进行 EMA 更新

Stage 4: 评估
─────────────────────────────────────────────────────────────────────
  - 在保留的偏好测试集上计算准确率
  - 自动评估（AlpacaEval, MT-Bench, Chatbot Arena）
  - 人工评估
```

#### 超参数调优

| 超参数 | 推荐范围 | 影响 |
|--------|---------|------|
| **β** | 0.01 ~ 0.5 | KL 惩罚强度：越大越保守（接近 π_ref），越小越激进 |
| **学习率** | 1e-7 ~ 5e-6 | 比标准 SFT 学习率小 1-2 个数量级 |
| **Batch size** | 32 ~ 256 | 越大偏好信号越稳定 |
| **训练步数** | 200 ~ 2000 | 通常训练 1-3 个 epoch，更多易过拟合 |
| **优化器** | AdamW | 权重衰减 0.01 ~ 0.1 |

#### Beta 参数详细调优指南

```
β 过小（< 0.01）：
  → 隐式 KL 惩罚不足，模型极易过拟合
  → 症状：偏好准确率快速上升至 100%，但生成质量下降
  → 建议：如果训练损失在 3 步内降到 0.001 以下，增大 β

β 过大（> 0.5）：
  → 隐式 KL 惩罚过强，模型几乎不更新
  → 症状：偏好准确率不提升，损失几乎不变
  → 建议：如果训练 100 步后偏好准确率仍 < 60%，减小 β

β 调优经验法则：
  1. 从 β = 0.1 开始
  2. 观察训练集偏好准确率（期望：~70-85%）
  3. 如果 > 90%，增大 β；如果 < 60%，减小 β
  4. 检查生成样本的长度（期望：与 SFT 模型相当）
  5. 最终选择在验证集上生成质量最高的 β
```

#### 数据准备最佳实践

```python
# 偏好数据格式示例（HuggingFace datasets）
{
    "prompt": "什么是相对论？请用简单的话解释。",
    "chosen": "相对论是爱因斯坦提出的理论...（好的回答）",
    "rejected": "相对论是...（差的回答：太短/错误/冗长）",
    "source": "human_annotation",
    "domain": "physics",
    "difficulty": "easy"
}
```

数据质量检查清单：
- [ ] 偏好对差异明显：理想情况是 chosen 明显优于 rejected
- [ ] 无虚假相关：chosen 不总是更长的回答
- [ ] 覆盖多样场景：不同难度、领域、回答风格的 prompt
- [ ] 无冲突偏好：同一条 prompt 的不同偏好对不应矛盾
- [ ] 质量 > 数量：1000 对高质量偏好 > 10000 对噪音偏好

### 4. DPO vs RLHF Comparison: sample efficiency, stability, performance, compute requirements

#### 全面对比

| 维度 | RLHF (PPO) | DPO |
|------|-----------|-----|
| **训练阶段** | 3 阶段：SFT + RM + PPO | 2 阶段：SFT + DPO |
| **奖励模型** | 需要训练一个独立的 Reward Model | 不需要，隐式奖励 |
| **训练稳定性** | 较稳定（PPO 有 clip + KL 双重保护） | 较敏感（依赖 β 和数据质量） |
| **样本效率** | 低（在线采样 + 奖励推理） | 高（直接使用偏好数据） |
| **计算成本** | 高（4 个模型：Actor/Critic/Ref/Reward） | 低（2 个模型：Policy/Ref） |
| **内存占用** | 高（需要加载 4 个模型） | 低（2 个模型，可使用 LoRA） |
| **在线探索** | 天然支持（PPO 在线采样） | 不支持（标准 DPO 为离线） |
| **奖励监控** | 可以监控奖励分数 | 无显式奖励，难以监控 |
| **调参难度** | 高（PPO 的 clip/GAE/lr/KL_target 等） | 中（主要是 β 和学习率） |
| **分布式扩展** | 复杂（PPO 需要同步采样和更新） | 简单（类似标准 LLM 训练） |
| **代码量** | 1000+ 行（完整 RLHF 管线） | ~100 行（核心损失函数） |
| **流行度** | 工业界老牌标准 | 学术界和开源社区首选 |

#### 实验对比（典型 7B 模型）

| 指标 | RLHF (PPO) | DPO |
|------|-----------|-----|
| 训练时间（7B 模型） | 3-5 天（3 阶段合计） | 1-2 天（2 阶段合计） |
| GPU 需求 | 8-16 A100（含 RM 训练+推理） | 4-8 A100（无 RM） |
| 偏好测试准确率 | 75-82% | 72-80% |
| MT-Bench 分数 | 7.2-7.8 | 7.0-7.6 |
| AlpacaEval LC | 25-35% | 22-32% |
| 输出长度变化 | 变化小（PPO 有长度惩罚） | 可能变长 10-30% |
| 训练发散风险 | 低 | 中 |

### 5. DPO Variants: IPO, KTO, ORPO, CPO, VERL

#### IPO (Identity Preference Optimization)

**论文**：Azar et al., 2023, "A General Theoretical Paradigm for Preference Optimization"

**核心思想**：将 DPO 的偏好优化重写为回归问题，避免 DPO 对偏好数据的过拟合。

```
L_IPO = E[(log(π_θ(y_w|x)/π_ref(y_w|x)) - log(π_θ(y_l|x)/π_ref(y_l|x)) - τ^(-1))^2]

其中 τ 是控制偏好间隔的超参数
```

**相比 DPO 的改变**：
- 用平方损失替换 log-sigmoid 损失
- 显式定义了偏好间隔（preference margin）
- 训练更稳定，不易过拟合（损失不会降到 0）

**适用场景**：偏好数据存在噪声或分布不均匀时

#### KTO (Kahneman-Tversky Optimization)

**论文**：Ethayarajh et al., 2024, "KTO: Model Alignment as Prospect Theoretic Optimization"

**核心思想**：不需要成对偏好数据，只需知道单个回答是"好"还是"坏"。

```
L_KTO = E_y_chosen[ λ_chosen · (1 - σ(β · (log π_θ(y|x) - log π_ref(y|x) - z_ref))) ]
       + E_y_rejected[ λ_rejected · σ(β · (z_ref - (log π_θ(y|x) - log π_ref(y|x)))) ]

其中 z_ref = KL(π_θ || π_ref) 是隐式参考点
```

**相比 DPO 的改变**：
- 不需要配对数据，每个样本独立标注好/坏
- 受前景理论（Prospect Theory）启发，对"损失"（坏回答）赋予更高权重
- λ_chosen 和 λ_rejected 控制偏好/拒绝的权重比例

**适用场景**：偏好数据是单侧标注（如只有正面或只有负面反馈），而非成对比较

#### ORPO (Odds Ratio Preference Optimization)

**论文**：Hong et al., 2024, "ORPO: Monolithic Preference Optimization without Reference Model"

**核心思想**：联合 SFT + 偏好优化，消除单独的 SFT 阶段，也不需要参考模型。

```
L_ORPO = L_SFT + λ · L_OR

L_OR = -log σ(log odds_θ(y_w) - log odds_θ(y_l))
其中 odds_θ(y) = π_θ(y|x) / (1 - π_θ(y|x))
```

**相比 DPO 的改变**：
- SFT 和偏好优化在单阶段完成，不需要参考模型
- 使用胜率比（Odds Ratio）而不是概率比
- 训练更简单，只需一个模型

**适用场景**：希望简化训练管线，从基础模型直接对齐

#### CPO (Contrastive Preference Optimization)

**核心思想**：在 DPO 基础上增加对比学习项，让模型更好区分相似但不同的偏好。

```
L_CPO = L_DPO + α · L_contrastive

L_contrastive = -log(exp(sim(y_w, y_ref)) / (exp(sim(y_w, y_ref)) + exp(sim(y_l, y_ref))))
```

**相比 DPO 的改变**：
- 增加对比学习项，在表示空间拉近偏好回答、推远拒绝回答
- 对细粒度偏好区分更有效

**适用场景**：偏好差异细微（如回答风格不同但内容正确），需要精细对齐

#### VERL (Voltron-Enhanced RL)

**论文**：字节跳动开源, 2025

**核心思想**：高性能分布式 DPO 训练框架，支持 70B+ 参数模型。

**相比 DPO 的改变**：
- 优化的分布式训练策略（模型并行 + 数据并行混合）
- 支持在线 DPO（从当前策略采样偏好对）
- 集成梯度检查点、混合精度训练等工程优化
- 支持大规模 DPO + RLHF 混合训练

#### 变体选择指南

```
需要最小化改动？  ──────────────────► DPO （标准选择）

偏好数据有噪声？ ──────────────────► IPO （平方损失鲁棒）

只有单侧标注？  ──────────────────► KTO （不要求成对数据）

想简化管线？    ──────────────────► ORPO （单阶段，无参考模型）

偏好差异细微？  ──────────────────► CPO （对比学习更精细）

需要大规模训练？  ─────────────────► VERL （分布式高性能）

需要在线探索？    ─────────────────► 在线 DPO / 迭代 DPO
```

### 6. DPO Failure Modes: mode collapse, length exploitation, diversity reduction

#### 模式崩塌（Mode Collapse）

**现象**：模型将所有输入的输出集中到少数几种模式，多样性显著下降。

**原因**：
- DPO 损失同时增加偏好回答概率 + 降低拒绝回答概率
- 拒绝回答概率降低可能导致模型"遗忘"某些正常的生成模式
- 当拒绝回答恰好属于某类正常输出时，这类输出就会被整体抑制

**检测方法**：
```
训练前 VS 训练后的生成多样性对比：
- Self-BLEU: 训练后模型生成之间的 BLEU 升高
- Distinct-n: 训练后 distinct 1/2-gram 比例下降
- 温度敏感性: 不同温度下生成多样性降低
```

**缓解策略**：
- 使用 IPO（平方损失不鼓励零概率）
- 适当增大 β（加强 KL 约束）
- 在偏好数据中覆盖多样化的拒绝回答
- 混合 SFT 损失（添加 DPO 与 SFT 的联合训练）

#### 长度利用（Length Exploitation）

**现象**：模型学会生成更长的回答来提高被偏好的概率，即使长回答并不更好。

**原因**：
- 偏好数据中存在"长回答更好"的虚假相关
- DPO 隐式奖励倾向于利用这种虚假相关
- 模型发现增加长度可以稳定提高被偏好概率

**检测方法**：
```
- 训练过程中输出平均长度持续上升（>20% 增长）
- 长度与偏好准确率强相关（在测试集上控制长度后准确率下降）
- 短提示（如 "Hi"）也生成长段落
```

**缓解策略**：
- 在偏好数据中平衡长度（chosen 不总是更长的）
- 在损失中加入长度惩罚项
- 使用长度归一化的偏好比较
- 参考模型约束（增大 β）
- 使用 SimPO（SimPO 的奖励直接基于平均 token 对数概率）

#### 多样性降低（Diversity Reduction）

**现象**：同一 prompt 的多次生成高度相似，语义多样性下降。

**原因**：
- DPO 的极大似然目标鼓励确定性的偏好输出
- 与 RLHF 不同，DPO 没有在线探索机制
- 缺乏对输出的探索 → 策略快速收敛到狭窄分布

**缓解策略**：
- 增强偏好数据多样性（same prompt, multiple valid chosen responses）
- 使用温度退火（训练初期高温度保持多样性）
- 在损失中加入熵奖励项
- 迭代 DPO（从当前策略采样，保持探索）

#### 过拟合偏好数据

**现象**：训练集偏好准确率 > 95%，但生成质量和泛化能力下降。

**原因**：DPO 的简单损失函数极易过拟合。

**检测方法**：
```
1. 训练集/验证集偏好准确率差距 > 10%
2. 训练损失降到 < 0.001
3. 人类评估发现生成质量不如偏好准确率所暗示的
```

**缓解策略**：
- 更早停止训练（1-2 个 epoch 通常足够）
- 使用更大的 β
- 使用更为鲁棒的 IPO 损失
- 增加偏好数据的质量和多样性，而非数量
- 使用数据增强（如对拒绝回答添加轻微扰动）

### 7. When to Choose DPO vs RLHF vs Other Methods

#### 决策树

```
计算资源有限？
├─ 是 → DPO 或 ORPO（不需要奖励模型，训练成本低）
└─ 否 → 考虑 RLHF（更大可控性）

偏好数据质量高且量大？
├─ 是 → DPO 或 IPO（直接优化效果好）
└─ 否 → KTO 或 RLHF（对噪声更鲁棒）

需要在线探索和迭代改进？
├─ 是 → RLHF 或 在线 DPO（从当前策略采样）
└─ 否 → 标准 DPO（固定数据集）

需要细粒度控制（如按方面优化）？
├─ 是 → RLHF（可以用多个奖励模型）
└─ 否 → DPO（端到端偏好优化）

只能获得好/坏标签（而非成对比较）？
├─ 是 → KTO（单侧标注专用）
└─ 否 → DPO 或 IPO

生产环境部署（需要稳定性和可预测性）？
├─ 是 → RLHF 或 IPO（经过大规模验证）
└─ 否 → DPO（快速迭代实验）
```

#### 方法选择矩阵

| 场景 | 推荐方法 | 理由 |
|------|---------|------|
| 资源受限 | DPO / ORPO | 不需要奖励模型，2 个模型即可 |
| 高噪声数据 | IPO / RLHF | 对噪声偏好更鲁棒 |
| 单侧标注 | KTO | 不要求成对数据 |
| 多奖励融合 | RLHF | 容易集成多个奖励信号 |
| 快速原型 | DPO | 实现简单，迭代快 |
| 最佳性能 | RLHF (在线) | 可在线探索，理论保证更强 |
| 最小工程量 | ORPO | 单阶段，无参考模型 |
| 超大规模 | VERL / RLHF | 分布式框架成熟 |
| 安全性优先 | RLHF + DPO 混合 | 结合两者优势 |

## Example Code: Python DPO implementation using HuggingFace TRL, with data preparation

### 完整的 DPO 训练代码

```python
"""
DPO 训练示例：从数据准备到训练完成
使用 HuggingFace TRL 的 DPOTrainer
"""

import torch
from datasets import Dataset, load_dataset
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    TrainingArguments,
    HfArgumentParser,
)
from trl import DPOTrainer, DPOConfig
from peft import LoraConfig, get_peft_model
from typing import Dict, List, Optional
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


# ============================================================
# 1. 数据准备：从原始标注到 DPO 格式
# ============================================================

def prepare_dpo_data(
    raw_data: List[Dict[str, str]],
    tokenizer: AutoTokenizer,
    max_length: int = 2048,
    max_prompt_length: int = 1024,
) -> Dataset:
    """
    将原始偏好数据转换为 DPO 训练格式。

    输入格式：
        [{"prompt": "...", "chosen": "...", "rejected": "..."}, ...]

    处理步骤：
        1. Tokenize prompt、chosen 和 rejected
        2. 拼接 prompt + chosen 和 prompt + rejected
        3. 为每个序列创建 labels（将 prompt 部分 mask 掉）
    """

    def tokenize_dpo_pair(example: Dict[str, str]) -> Dict[str, List[int]]:
        """对单个偏好对进行 tokenization"""

        # Tokenize prompt（不加回答）
        prompt_tokens = tokenizer(
            example["prompt"],
            truncation=True,
            max_length=max_prompt_length,
            add_special_tokens=False,
        )

        # Tokenize chosen 回答
        chosen_tokens = tokenizer(
            example["chosen"],
            truncation=True,
            max_length=max_length - len(prompt_tokens["input_ids"]),
            add_special_tokens=False,
        )

        # Tokenize rejected 回答
        rejected_tokens = tokenizer(
            example["rejected"],
            truncation=True,
            max_length=max_length - len(prompt_tokens["input_ids"]),
            add_special_tokens=False,
        )

        # 拼接 prompt + chosen
        chosen_input_ids = prompt_tokens["input_ids"] + chosen_tokens["input_ids"] + [tokenizer.eos_token_id]
        chosen_attention_mask = [1] * len(chosen_input_ids)
        chosen_labels = [-100] * len(prompt_tokens["input_ids"]) + chosen_tokens["input_ids"] + [tokenizer.eos_token_id]

        # 拼接 prompt + rejected
        rejected_input_ids = prompt_tokens["input_ids"] + rejected_tokens["input_ids"] + [tokenizer.eos_token_id]
        rejected_attention_mask = [1] * len(rejected_input_ids)
        rejected_labels = [-100] * len(prompt_tokens["input_ids"]) + rejected_tokens["input_ids"] + [tokenizer.eos_token_id]

        return {
            "prompt_ids": prompt_tokens["input_ids"],
            "prompt_attention_mask": prompt_tokens["attention_mask"],
            "chosen_input_ids": chosen_input_ids,
            "chosen_attention_mask": chosen_attention_mask,
            "chosen_labels": chosen_labels,
            "rejected_input_ids": rejected_input_ids,
            "rejected_attention_mask": rejected_attention_mask,
            "rejected_labels": rejected_labels,
        }

    # 转换为 HuggingFace Dataset 格式
    dataset = Dataset.from_list(raw_data)
    dataset = dataset.map(
        tokenize_dpo_pair,
        remove_columns=dataset.column_names,
        desc="Tokenizing DPO data",
    )

    return dataset


# ============================================================
# 2. 模型初始化（支持 LoRA）
# ============================================================

def initialize_models(
    base_model_name: str = "mistralai/Mistral-7B-v0.1",
    use_lora: bool = True,
    lora_r: int = 16,
    lora_alpha: int = 32,
    lora_dropout: float = 0.05,
    device: str = "auto",
):
    """初始化策略模型、参考模型和 Tokenizer"""

    logger.info(f"Loading base model: {base_model_name}")

    # 加载 tokenizer
    tokenizer = AutoTokenizer.from_pretrained(base_model_name)
    tokenizer.pad_token = tokenizer.eos_token
    tokenizer.padding_side = "left"  # DPO 需要 left padding

    # 加载策略模型（需要梯度）
    policy_model = AutoModelForCausalLM.from_pretrained(
        base_model_name,
        torch_dtype=torch.bfloat16,
        device_map=device,
        attn_implementation="flash_attention_2",  # 加速训练
    )

    # 加载参考模型（不需要梯度）
    reference_model = AutoModelForCausalLM.from_pretrained(
        base_model_name,
        torch_dtype=torch.bfloat16,
        device_map=device,
        attn_implementation="flash_attention_2",
    )

    # 冻结参考模型
    for param in reference_model.parameters():
        param.requires_grad = False

    # 可选：应用 LoRA
    if use_lora:
        lora_config = LoraConfig(
            r=lora_r,
            lora_alpha=lora_alpha,
            lora_dropout=lora_dropout,
            task_type="CAUSAL_LM",
            target_modules=["q_proj", "v_proj", "k_proj", "o_proj", "gate_proj", "up_proj", "down_proj"],
        )
        policy_model = get_peft_model(policy_model, lora_config)
        policy_model.print_trainable_parameters()

    return policy_model, reference_model, tokenizer


# ============================================================
# 3. 自定义 DPO 损失函数（可选，展示数学原理）
# ============================================================

def dpo_loss(
    policy_chosen_logps: torch.Tensor,   # log π_θ(y_w|x)
    policy_rejected_logps: torch.Tensor, # log π_θ(y_l|x)
    ref_chosen_logps: torch.Tensor,      # log π_ref(y_w|x)
    ref_rejected_logps: torch.Tensor,    # log π_ref(y_l|x)
    beta: float = 0.1,
) -> tuple:
    """
    计算 DPO 损失（数学验证用，DPOTrainer 已内置）。

    L = -E[ log σ(β * (log π_θ(y_w)/π_ref(y_w) - log π_θ(y_l)/π_ref(y_l))) ]

    返回：
        losses: 每个样本的损失
        chosen_rewards: 每个样本的偏好回答隐式奖励
        rejected_rewards: 每个样本的拒绝回答隐式奖励
        accuracy: 本批次的偏好准确率
    """
    # 计算隐式奖励差值
    pi_logratios = policy_chosen_logps - policy_rejected_logps
    ref_logratios = ref_chosen_logps - ref_rejected_logps
    logits = beta * (pi_logratios - ref_logratios)

    # DPO 损失
    losses = -torch.nn.functional.logsigmoid(logits)

    # 隐式奖励
    chosen_rewards = beta * (policy_chosen_logps - ref_chosen_logps).detach()
    rejected_rewards = beta * (policy_rejected_logps - ref_rejected_logps).detach()

    # 准确率
    accuracy = (logits > 0).float().mean()

    return losses, chosen_rewards, rejected_rewards, accuracy


# ============================================================
# 4. 主训练流程
# ============================================================

def train_dpo(
    # 数据参数
    train_data_path: str = "data/dpo_train.json",
    eval_data_path: Optional[str] = None,
    # 模型参数
    base_model_name: str = "mistralai/Mistral-7B-v0.1",
    use_lora: bool = True,
    # DPO 参数
    beta: float = 0.1,
    # 训练参数
    output_dir: str = "./dpo_model",
    num_train_epochs: int = 1,
    per_device_batch_size: int = 4,
    gradient_accumulation_steps: int = 8,
    learning_rate: float = 5e-6,
    warmup_steps: int = 100,
    logging_steps: int = 10,
    save_steps: int = 500,
    eval_steps: int = 500,
    max_length: int = 2048,
    max_prompt_length: int = 1024,
):
    """完整的 DPO 训练流程"""

    # 初始化模型
    policy_model, reference_model, tokenizer = initialize_models(
        base_model_name=base_model_name,
        use_lora=use_lora,
    )

    # 准备数据（从 JSON 或其他格式加载）
    import json

    with open(train_data_path, "r", encoding="utf-8") as f:
        raw_train_data = json.load(f)

    train_dataset = prepare_dpo_data(raw_train_data, tokenizer, max_length, max_prompt_length)

    if eval_data_path:
        with open(eval_data_path, "r", encoding="utf-8") as f:
            raw_eval_data = json.load(f)
        eval_dataset = prepare_dpo_data(raw_eval_data, tokenizer, max_length, max_prompt_length)
    else:
        eval_dataset = None

    # 配置训练参数
    training_args = DPOConfig(
        output_dir=output_dir,
        num_train_epochs=num_train_epochs,
        per_device_train_batch_size=per_device_batch_size,
        per_device_eval_batch_size=per_device_batch_size,
        gradient_accumulation_steps=gradient_accumulation_steps,
        learning_rate=learning_rate,
        warmup_steps=warmup_steps,
        logging_steps=logging_steps,
        save_steps=save_steps,
        eval_steps=eval_steps,
        evaluation_strategy="steps" if eval_dataset else "no",
        save_strategy="steps",
        bf16=True,
        tf32=True,
        gradient_checkpointing=True,
        gradient_checkpointing_kwargs={"use_reentrant": False},
        remove_unused_columns=False,
        report_to=["wandb"],
        run_name="dpo_training",
        # DPO 特定参数
        beta=beta,
        loss_type="sigmoid",  # 可选: sigmoid, ipo, kto, orpo
        label_smoothing=0.0,
    )

    # 初始化 DPOTrainer
    dpo_trainer = DPOTrainer(
        model=policy_model,
        ref_model=reference_model,
        args=training_args,
        train_dataset=train_dataset,
        eval_dataset=eval_dataset,
        tokenizer=tokenizer,
        max_length=max_length,
        max_prompt_length=max_prompt_length,
        # DPO 数据字段映射
        dataset_num_proc=4,
    )

    # 开始训练
    logger.info("Starting DPO training...")
    dpo_trainer.train()

    # 保存最终模型
    dpo_trainer.save_model(output_dir)
    tokenizer.save_pretrained(output_dir)

    logger.info(f"DPO training completed. Model saved to {output_dir}")

    return dpo_trainer


# ============================================================
# 5. 使用示例
# ============================================================

if __name__ == "__main__":
    # 示例数据
    example_data = [
        {
            "prompt": "如何学习编程？",
            "chosen": "学习编程最好的方法是从实践开始...（详细、结构化的建议）",
            "rejected": "去上网搜一下就行了。（简短、缺乏帮助）",
        },
        {
            "prompt": "解释一下什么是机器学习",
            "chosen": "机器学习是人工智能的一个分支...（清晰的解释，有例子）",
            "rejected": "机器学习就是 AI。（过于简略）",
        },
    ]

    # 保存示例数据
    import json
    import os

    os.makedirs("data", exist_ok=True)
    with open("data/dpo_train.json", "w", encoding="utf-8") as f:
        json.dump(example_data * 100, f, ensure_ascii=False, indent=2)  # 复制 100 条

    # 开始训练
    train_dpo(
        train_data_path="data/dpo_train.json",
        base_model_name="mistralai/Mistral-7B-v0.1",  # 或使用更小的模型测试
        use_lora=True,
        beta=0.1,
        num_train_epochs=1,
        output_dir="./dpo_model",
        per_device_batch_size=2,  # 小 batch 用于测试
        gradient_accumulation_steps=4,
    )
```

### 使用 DPOTrainer 的最简示例

```python
# 最简 DPO 训练（假设数据已准备好）
from datasets import load_dataset
from trl import DPOTrainer, DPOConfig
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("your-sft-model")
tokenizer = AutoTokenizer.from_pretrained("your-sft-model")
tokenizer.pad_token = tokenizer.eos_token

# 准备数据集（格式：prompt, chosen, rejected）
dataset = load_dataset("json", data_files="preference_data.jsonl")

training_args = DPOConfig(
    output_dir="./dpo-output",
    beta=0.1,
    learning_rate=5e-7,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=8,
    num_train_epochs=1,
    bf16=True,
)

trainer = DPOTrainer(
    model=model,
    ref_model=None,  # 如果不提供，自动使用 model 的副本
    args=training_args,
    train_dataset=dataset["train"],
    tokenizer=tokenizer,
    max_length=2048,
    max_prompt_length=1024,
)

trainer.train()
```

## Capability Boundaries: DPO needs high-quality preference data, sensitive to hyperparameters

### 已知边界

| 边界 | 说明 | 突破方向 |
|------|------|---------|
| **偏好数据质量要求高** | DPO 对单条偏好数据的质量极其敏感，一条错误数据即可扰乱训练 | 数据清洗 + 自动过滤 + IPO 鲁棒损失 |
| **β 调参敏感** | β 稍有不合适就导致欠对齐或过拟合 | β 调度策略 + 自适应 β 调整 |
| **缺乏在线探索** | 标准 DPO 无法探索策略空间，只能使用固定数据集 | 在线 DPO / 迭代 DPO |
| **多奖励融合困难** | 难以将多个奖励信号（安全性、有用性、风格等）同时纳入 | 奖励合并技术 + 多任务 DPO |
| **规模扩展挑战** | 70B+ 模型 DPO 训练需要特殊优化 | VERL / DeepSpeed + ZeRO-3 |
| **动态分布偏移** | 偏好数据分布与部署分布不一致时效果下降 | 持续 DPO + 分布匹配 |
| **难以联合优化** | 难以同时优化多个指标（如帮助性+安全性） | 多目标 DPO + 约束优化 |
| **跨语言泛化** | 英文偏好数据训练的 DPO 对其他语言泛化有限 | 多语言偏好数据 + 跨语言迁移 |

### 何时 DPO 不适用

1. **偏好数据质量差或数量不足**（< 500 对高质量偏好）
2. **需要细粒度的多维奖励控制**（如同时优化安全性、有用性、诚实性且各自有不同权重）
3. **需要在线探索来发现策略边界**（如新的、未见过的安全场景）
4. **奖励信号可以清晰定义**（如准确率、代码正确性等客观指标，此时用奖励模型更直接）
5. **计算资源极度受限**（连 2 个模型都加载不了 → 考虑 ORPO 或 SimPO）

## Comparison table: DPO vs IPO vs KTO vs ORPO vs RLHF

| 方法 | 发表年份 | 是否需要奖励模型 | 是否需要参考模型 | 是否需要成对数据 | 训练阶段数 | 训练稳定性 | 样本效率 | 偏好准确率 | 计算成本 | 代码复杂度 | 过拟合风险 |
|------|---------|----------------|----------------|----------------|-----------|-----------|---------|-----------|---------|-----------|-----------|
| **RLHF (PPO)** | 2022 | ✅ 需要 | ✅ 需要 | ✅ 需要 | 3 | ★★★★☆ 高 | ★★☆☆☆ 低 | ★★★★☆ 高 | ★★★★★ 极高 | ★★★★★ 极高 | ★★☆☆☆ 低 |
| **DPO** | 2023 | ❌ 不需要 | ✅ 需要 | ✅ 需要 | 2 | ★★★☆☆ 中 | ★★★★☆ 高 | ★★★★☆ 高 | ★★☆☆☆ 低 | ★★☆☆☆ 低 | ★★★★☆ 高 |
| **IPO** | 2023 | ❌ 不需要 | ✅ 需要 | ✅ 需要 | 2 | ★★★★☆ 高 | ★★★★☆ 高 | ★★★★☆ 高 | ★★☆☆☆ 低 | ★★☆☆☆ 低 | ★★★☆☆ 中 |
| **KTO** | 2024 | ❌ 不需要 | ✅ 需要 | ❌ 不需要（单侧） | 2 | ★★★☆☆ 中 | ★★★☆☆ 中 | ★★★☆☆ 中 | ★★☆☆☆ 低 | ★★☆☆☆ 低 | ★★★☆☆ 中 |
| **ORPO** | 2024 | ❌ 不需要 | ❌ 不需要 | ✅ 需要 | 1 | ★★★☆☆ 中 | ★★★★☆ 高 | ★★★☆☆ 中 | ★☆☆☆☆ 极低 | ★☆☆☆☆ 极低 | ★★★★☆ 高 |
| **SimPO** | 2024 | ❌ 不需要 | ❌ 不需要 | ✅ 需要 | 1 | ★★★★☆ 高 | ★★★★☆ 高 | ★★★★☆ 高 | ★☆☆☆☆ 极低 | ★☆☆☆☆ 极低 | ★★★☆☆ 中 |
| **CPO** | 2024 | ❌ 不需要 | ✅ 需要 | ✅ 需要 | 2 | ★★★☆☆ 中 | ★★★★☆ 高 | ★★★★☆ 高 | ★★☆☆☆ 低 | ★★☆☆☆ 低 | ★★★★☆ 高 |

### 详细对比维度

#### 训练成本

```
方法      GPU内存      训练时间       总成本
─────────────────────────────────────────────
RLHF   ~80GB (7B)   5-7 天        极高（4模型）
DPO    ~40GB (7B)   1-2 天        低（2模型）
IPO    ~40GB (7B)   1-2 天        低（2模型）
KTO    ~40GB (7B)   1-2 天        低（2模型）
ORPO   ~20GB (7B)   0.5-1 天      极低（1模型）
SimPO  ~20GB (7B)   0.5-1 天      极低（1模型，无参考）
```

#### 训练稳定性

```
方法        发散风险    超参数敏感度    损失可解释性
───────────────────────────────────────────────
RLHF   低      中（8+ 超参）     好（奖励分数可监控）
DPO    中-高   高（β 敏感）      一般（隐式奖励不可观测）
IPO    低-中   中（τ 较鲁棒）    好（回归损失可解释）
KTO    中      高（z_ref 敏感）  一般（效用函数）
ORPO   中-高   高（λ 敏感）      一般（SFT+OR 混合）
SimPO  低-中   中（γ 较鲁棒）    好（参考长度归一化）
```

#### 数据要求

```
方法        数据量需求    数据质量需求    数据格式
───────────────────────────────────────────
RLHF   高（10K+）   高             成对偏好 + 奖励标注
DPO    中（1K+）    极高            成对偏好
IPO    中（1K+）    中             成对偏好（可容忍噪声）
KTO    低（500+）   中             单侧好/坏标签
ORPO   中（1K+）    高             成对偏好（联合 SFT）
SimPO  中（1K+）    高             成对偏好 + 平均长度参考
```

## Engineering Optimization: beta scheduling, reference model update strategies, curriculum preference learning

### 1. Beta Scheduling（β 调度策略）

Beta 控制 DPO 中 KL 约束的强度。固定的 β 在训练全程使用单一约束强度，但不同训练阶段对约束的需求是不同的。

#### 策略对比

| 调度策略 | 方式 | 适用场景 |
|---------|------|---------|
| **固定 β** | β = 0.1 全程不变 | 简单场景，基线方法 |
| **β 退火** | β 从 0.5 线性/余弦退火到 0.01 | 训练初期强约束防止偏离，后期弱约束允许精细调优 |
| **β 升温** | β 从 0.01 升温到 0.5 | 训练初期允许大量探索，后期收紧约束保证稳定性 |
| **自适应 β** | 根据 KL 散度动态调整 β | 需要精确控制策略偏移量时 |
| **分阶段 β** | 第一阶段 β=0.3（粗对齐），第二阶段 β=0.05（细调） | 两阶段 DPO 训练 |

#### 自适应 β 调度算法

```python
class AdaptiveBetaScheduler:
    """
    基于 KL 散度的自适应 β 调度。

    核心逻辑：
    - 计算当前策略 π_θ 与参考策略 π_ref 之间的 KL 散度
    - 目标 KL 散度：D_target（如 0.1 nats/token）
    - 当 KL > D_target + margin → 增大 β（加强约束）
    - 当 KL < D_target - margin → 减小 β（放松约束）
    """
    def __init__(
        self,
        initial_beta: float = 0.1,
        target_kl: float = 0.1,
        margin: float = 0.02,
        adjustment_factor: float = 1.1,
        min_beta: float = 0.01,
        max_beta: float = 1.0,
    ):
        self.beta = initial_beta
        self.target_kl = target_kl
        self.margin = margin
        self.adjustment_factor = adjustment_factor
        self.min_beta = min_beta
        self.max_beta = max_beta
        self.kl_history = []

    def step(self, current_kl: float) -> float:
        """根据当前 KL 散度更新 β"""
        self.kl_history.append(current_kl)

        if current_kl > self.target_kl + self.margin:
            # KL 过高，需要加强约束
            self.beta = min(self.beta * self.adjustment_factor, self.max_beta)
        elif current_kl < self.target_kl - self.margin:
            # KL 过低，可以放松约束
            self.beta = max(self.beta / self.adjustment_factor, self.min_beta)

        return self.beta
```

### 2. Reference Model Update Strategies（参考模型更新策略）

标准 DPO 中 π_ref 是固定的，但在某些场景下需要更新 π_ref。

#### 策略对比

| 策略 | 更新频率 | 更新方式 | 优缺点 |
|------|---------|---------|--------|
| **固定参考模型** | 从不更新 | 使用训练初期的 π_ref | + 简单、稳定<br>- 限制探索空间 |
| **EMA 参考模型** | 每训练步 | θ_ref = α · θ_ref + (1-α) · θ_policy | + 平滑的参考分布<br>+ 训练稳定<br>- 需要额外的内存和维护 |
| **阶段更新** | 每 N 步 | θ_ref = θ_policy（完全替换） | + 周期性重置参考点<br>- 可能导致训练波动 |
| **混合策略** | 开始时固定，后期 EMA | 先固定 K 步，然后切换到 EMA | + 兼顾初期稳定和后期灵活 |

#### EMA 参考模型实现

```python
class EMAReferenceUpdater:
    """
    对参考模型进行指数移动平均更新。

    π_ref_new = α · π_ref_old + (1-α) · π_θ

    α 选择策略：
    - 大 α（>0.99）：更新极慢，接近固定参考模型
    - 小 α（0.9-0.99）：更新较快，参考模型更接近当前策略
    """
    def __init__(
        self,
        ref_model,
        policy_model,
        alpha: float = 0.999,
        warmup_steps: int = 1000,
    ):
        self.ref_model = ref_model
        self.policy_model = policy_model
        self.alpha = alpha
        self.warmup_steps = warmup_steps
        self.step_count = 0

        # 保存参考模型的初始参数副本
        self._init_ref_params = {
            name: param.data.clone()
            for name, param in ref_model.named_parameters()
        }

    def step(self):
        """执行一次 EMA 更新"""
        self.step_count += 1

        # warmup 阶段：不更新参考模型
        if self.step_count < self.warmup_steps:
            return

        # 有效 α：在 warmup 后逐渐过渡到目标 α
        effective_alpha = min(
            self.alpha,
            self.alpha + (1 - self.alpha) * (self.step_count - self.warmup_steps) / self.warmup_steps
        )

        # 对每层参数执行 EMA 更新
        with torch.no_grad():
            for ref_param, policy_param in zip(
                self.ref_model.parameters(),
                self.policy_model.parameters()
            ):
                ref_param.data = (
                    effective_alpha * ref_param.data
                    + (1 - effective_alpha) * policy_param.data
                )
```

### 3. Curriculum Preference Learning（课程偏好学习）

课程学习策略在 DPO 训练中逐步增加难度，从简单的偏好转到更细微的偏好。

#### 课程设计

```
层级         难度          偏好差异                       示例
──────────────────────────────────────────────────────────────
Level 1     简单          内容明显不同                   正确 vs 错误回答
Level 2     中等          质量明显不同                   详细完整 vs 简略回答
Level 3     较难          风格不同                       正式 vs 口语化
Level 4     困难          细微质量差异                   回答 A 略好于 B（都正确）
Level 5     专家          道德/安全权衡                   有用但危险 vs 安全但有损帮助
```

#### 课程学习实现

```python
class CurriculumDPOTrainer:
    """
    课程偏好学习 DPO 训练器。

    策略：
    1. 将偏好数据按难度分层
    2. 从简单层开始训练
    3. 当当前层达到准确率阈值后进入下一层
    4. 不同层使用不同的 β（简单层用大 β 保持稳定，困难层用小 β 精细调整）
    """

    def __init__(
        self,
        model,
        ref_model,
        tokenizer,
        curriculum_levels: List[Dict],
        beta_by_level: Dict[int, float] = None,
        accuracy_threshold: float = 0.85,
    ):
        self.model = model
        self.ref_model = ref_model
        self.tokenizer = tokenizer
        self.curriculum_levels = curriculum_levels
        self.beta_by_level = beta_by_level or {1: 0.3, 2: 0.2, 3: 0.15, 4: 0.1, 5: 0.05}
        self.accuracy_threshold = accuracy_threshold
        self.current_level = 1
        self.level_accuracies = {}

    def train_curriculum(self):
        """按课程顺序训练"""

        for level in sorted(self.curriculum_levels, key=lambda x: x["level"]):
            level_id = level["level"]
            level_data = level["data"]
            beta = self.beta_by_level.get(level_id, 0.1)

            print(f"Starting curriculum level {level_id} (beta={beta}, "
                  f"samples={len(level_data)})")

            # 在当前层级训练 DPO
            accuracy = self._train_level(level_data, beta)

            self.level_accuracies[level_id] = accuracy

            # 检查是否达到推进条件
            if accuracy < self.accuracy_threshold:
                print(f"Level {level_id} accuracy {accuracy:.3f} < threshold "
                      f"{self.accuracy_threshold}, retraining...")
                # 继续在当前层级训练直到达到阈值
                while accuracy < self.accuracy_threshold:
                    accuracy = self._train_level(level_data, beta)
                    self.level_accuracies[level_id] = accuracy

            print(f"Completed level {level_id} with accuracy {accuracy:.3f}")

        print("Curriculum training completed!")
        return self.level_accuracies

    def _train_level(self, data, beta):
        """在特定层级的数据上训练一个 epoch"""
        # DPO 训练逻辑（略）
        # 返回在该层级上的偏好准确率
        pass
```

### 4. 其他工程优化策略

#### 梯度累积与精度策略

| 策略 | 说明 | 收益 |
|------|------|------|
| **梯度检查点** | 在前向传播时丢弃中间激活，反向传播时重新计算 | 内存降低 50-70% |
| **混合精度训练** | bf16/fp16 + fp32 主权重 | 训练速度提升 2-3x |
| **梯度累积** | 累积多步梯度再更新 | 有效 batch size 扩展 |
| **ZeRO-3** | 优化器状态、梯度、参数分片 | 支持更大模型的单卡训练 |

#### 数据增强策略

1. **偏好对扩充**：对同一 prompt，使用不同模型/温度生成多种回答以增加多样性
2. **难例挖掘**：在训练过程中识别模型难以区分的偏好对，针对性补充训练
3. **偏好冲突检测**：当同一 prompt 的多个偏好对矛盾时，使用人类复审或多数投票
4. **伪偏好生成**：使用强模型（如 GPT-4）对弱模型的回答进行偏好标注

#### 多阶段 DPO 策略

```
Stage 1: 通用对齐
  - 数据：广泛领域的偏好数据
  - β：大（0.3-0.5）
  - 目标：建立基本的偏好方向

Stage 2: 领域对齐
  - 数据：特定领域的细粒度偏好
  - β：中（0.1-0.2）
  - 目标：在目标领域精细对齐

Stage 3: 安全对齐
  - 数据：安全/道德相关的红线数据
  - β：小（0.01-0.05）
  - 目标：注入安全约束，同时最小化对通用能力的损伤
```

#### 在线 DPO（Online DPO）

在线 DPO 从当前策略采样生成偏好对，而不是使用固定的离线数据集：

```
每一轮：
  1. 当前策略 π_θ 为 prompt x 生成多个回答 {y_1, ..., y_k}
  2. 使用自动评估（如奖励模型或 LLM-as-judge）对回答排序
  3. 构造偏好对 (y_w, y_l)
  4. 使用 DPO 损失更新策略
  5. 重复

优势：
  - 采样来自当前策略，分布匹配更好
  - 可以持续发现策略的弱点
  - 更接近 RLHF 的在线特性

代价：
  - 每次迭代需要采样和评估
  - 需要自动评估器（可能是另一个模型）
  - 训练周期更长
```

### 5. 训练监控最佳实践

```python
class DPOMonitor:
    """DPO 训练监控：记录关键指标并发出警报"""

    def __init__(self, log_dir: str = "./dpo_logs"):
        self.metrics = {
            "loss": [],
            "chosen_reward": [],
            "rejected_reward": [],
            "reward_margin": [],
            "accuracy": [],
            "kl_divergence": [],
            "avg_length": [],
            "beta": [],
        }
        self.alerts = []

    def log_step(
        self,
        loss: float,
        chosen_rewards: torch.Tensor,
        rejected_rewards: torch.Tensor,
        accuracy: float,
        kl_div: float,
        beta: float,
        outputs: List[str],
    ):
        """记录单步指标"""
        self.metrics["loss"].append(loss)
        self.metrics["chosen_reward"].append(chosen_rewards.mean().item())
        self.metrics["rejected_reward"].append(rejected_rewards.mean().item())
        self.metrics["reward_margin"].append(
            (chosen_rewards - rejected_rewards).mean().item()
        )
        self.metrics["accuracy"].append(accuracy)
        self.metrics["kl_divergence"].append(kl_div)
        self.metrics["avg_length"].append(sum(len(o) for o in outputs) / len(outputs))
        self.metrics["beta"].append(beta)

        # 检测异常
        self._check_alerts()

    def _check_alerts(self):
        """检测训练异常并发出警报"""

        recent_n = min(10, len(self.metrics["loss"]))

        # 警报 1：损失降到异常低
        if len(self.metrics["loss"]) >= 5:
            if self.metrics["loss"][-1] < 0.0001:
                self.alerts.append(
                    f"WARNING: Loss extremely low ({self.metrics['loss'][-1]:.6f}) "
                    f"— possible overfitting, consider increasing beta or stopping."
                )

        # 警报 2：偏好准确率过高
        if self.metrics["accuracy"][-1] > 0.95:
            self.alerts.append(
                f"WARNING: Accuracy {self.metrics['accuracy'][-1]:.3f} — "
                f"model may be overfitting to preference data."
            )

        # 警报 3：奖励差距过大
        if self.metrics["reward_margin"][-1] > 5.0:
            self.alerts.append(
                f"WARNING: Reward margin {self.metrics['reward_margin'][-1]:.2f} "
                f"— unusually large, check if model is exploiting length/shortcuts."
            )

        # 警报 4：KL 散度过大
        if self.metrics["kl_divergence"][-1] > 1.0:
            self.alerts.append(
                f"WARNING: KL divergence {self.metrics['kl_divergence'][-1]:.3f} "
                f"— policy is diverging from reference, consider increasing beta."
            )

        # 警报 5：输出长度急剧增加
        if len(self.metrics["avg_length"]) >= recent_n + 1:
            if self.metrics["avg_length"][-1] > self.metrics["avg_length"][-recent_n] * 1.2:
                self.alerts.append(
                    f"WARNING: Output length increased by >20% in last {recent_n} steps "
                    f"— possible length exploitation."
                )
```

### 6. 生产部署检查清单

```
□ 偏好数据质量审计
  ├─ chosen 是否真的比 rejected 好？
  ├─ 是否存在长度偏差（chosen 总是更长）？
  ├─ 是否覆盖了多样化的场景？
  └─ 是否有冲突的偏好标签？

□ 超参数验证
  ├─ β 是否通过小规模验证选择？
  ├─ 学习率是否比 SFT 小 1-2 个数量级？
  ├─ 训练轮数是否限制在 1-3 epoch？
  └─ 是否验证过几种不同的 β？

□ 训练监控
  ├─ 是否监控偏好准确率？（期望 70-85%）
  ├─ 是否监控输出长度变化？
  ├─ 是否监控 KL 散度？
  └─ 是否设置了早停条件？

□ 评估验证
  ├─ 在保留测试集上评估偏好准确率
  ├─ 使用自动评估（AlpacaEval, MT-Bench）
  ├─ 检查输出多样性（Self-BLEU, Distinct-n）
  ├─ 人工评估样本质量
  └─ 红队测试安全性

□ 部署准备
  ├─ 是否对比了 DPO 前后模型的生成质量？
  ├─ 是否有回滚机制？（保留 SFT 模型）
  ├─ 是否有 A/B 测试框架？
  └─ 是否对极端输入进行了压力测试？
```
