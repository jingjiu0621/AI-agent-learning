# 8.2.3 投票机制 (Voting Mechanisms)

## 1. 简单介绍

投票机制是一种将多个智能体的偏好或决策聚合为单一群体输出的协调方法。在多智能体系统中，每个智能体可能对同一问题持有不同看法，投票通过某种聚合规则（如多数决、加权求和、排序计分）将这些分散的意见合并为一个最终决策。其核心思想源于社会选择理论（Social Choice Theory）：给定一组个体偏好，如何"公平"地产生一个群体决策。

在 AI Agent 语境下，投票机制常用于：
- **LLM-as-Judge**：多个 LLM 对同一输出打分，聚合后得到更可靠的评估
- **响应选择**：多个 Agent 生成候选回复，投票选出最佳者
- **路由决策**：多个 Agent 对下一步动作投票，避免单一 Agent 的路径偏差
- **集成方法**：多个模型/Agent 的预测通过投票集成，提升鲁棒性

---

## 2. 基本原理

### 社会选择理论核心

投票机制的理论基础是社会选择理论（Social Choice Theory），它研究如何将个体的偏好集合转化为群体决策。

**阿罗不可能定理 (Arrow's Impossibility Theorem)**：
> 当候选选项大于等于 3 个时，不存在一种投票规则能同时满足以下四个条件：
> 1. **帕累托最优 (Pareto Efficiency)**：如果所有个体都偏好 A 胜过 B，则群体偏好 A 胜过 B
> 2. **无关选项独立性 (Independence of Irrelevant Alternatives, IIA)**：对 A 和 B 的群体偏好不应受 C 的影响
> 3. **非独裁性 (Non-dictatorship)**：不存在单个个体总能决定群体偏好
> 4. **全域性 (Universal Domain)**：任何个体偏好组合都能被处理

这一定理从根本上揭示了投票机制的局限性：没有任何投票规则是"完美"的。

**孔多塞悖论 (Condorcet Paradox)**：
> 当三个或以上投票者对三个或以上选项进行偏好排序时，可能出现循环多数（cyclical majority）：A > B、B > C、C > A 同时成立。这使得"多数决"无法产生一个传递性的群体偏好。

### 孔多塞陪审团定理 (Condorcet Jury Theorem)

这是支持投票机制最重要的理论保证：

> 如果每个投票者独立地做出正确判断的概率 p > 0.5，那么随着投票者数量 N 的增加，多数决得出正确结果的概率趋近于 1。反之，如果 p < 0.5，则趋近于 0。

数学表达：
$$P_{correct} = \sum_{k=\lceil N/2 \rceil}^{N} \binom{N}{k} p^k (1-p)^{N-k}$$

这个定理成立的关键前提是：
- 投票者的判断相互独立
- 每个投票者的准确率高于随机水平

### 投票聚合函数形式化

给定 N 个 Agent，每个 Agent $i$ 对选项集合 $O = \{o_1, o_2, ..., o_m\}$ 有偏好 $P_i$，投票机制是一个函数：

$$F: P_1 \times P_2 \times ... \times P_N \rightarrow O$$

即：将 N 个偏好向量映射为一个群体决策输出。

---

## 3. 背景与演进

```
18世纪 ──────────────────────────────→ 20世纪中期 ───────────→ 21世纪 ──────────────→ 现在
    │                                      │                      │                    │
    ▼                                      ▼                      ▼                    ▼
社会选择理论                          计算社会选择              机器学习集成           LLM Agent 投票
(孔多塞 1785)                         (偏好聚合算法)            (Bagging/Bosting)     (LLM-as-Judge)
(Borda 1770)                          (投票协议设计)            (随机森林)             (多 Agent 协商)
(Arrow 1951)                          (操纵性分析)             (模型集成投票)          (基于置信度的投票)
```

### 关键里程碑

| 时期 | 发展 | 影响 |
|------|------|------|
| 1770 | Borda 提出 Borda 计数法 | 最早的排序投票系统之一 |
| 1785 | Condorcet 提出陪审团定理 | 为多数决提供了数学保证 |
| 1785 | Condorcet 发现投票悖论 | 揭示了投票的固有矛盾 |
| 1951 | Arrow 提出不可能定理 | 奠定了现代社会选择理论基础 |
| 1970s-80s | Gibbard-Satterthwaite 定理 | 证明所有非平凡投票规则都可被操纵 |
| 1990s-2000s | 计算社会选择理论兴起 | 从算法角度研究投票的复杂性 |
| 2000s | Bagging / 随机森林 | 将投票引入机器学习集成方法 |
| 2020s | LLM Agent 投票 | 多个 LLM Agent 通过投票协作决策 |

---

## 4. 之前做法与结果

### 机器学习中的投票

**Majority Vote (硬投票)**：
```python
# 多个分类器的预测，取多数
predictions = [clf.predict(x) for clf in ensemble]
final_pred = mode(predictions)  # 多数决
```

**Bagging (Bootstrap Aggregating)**：
- 对训练数据进行 Bootstrap 采样，训练多个模型
- 通过投票聚合预测结果
- 代表算法：随机森林（Random Forest）

**软投票 (Soft Voting / Averaging)**：
```python
# 平均预测概率
probas = [clf.predict_proba(x) for clf in ensemble]
final_proba = mean(probas, axis=0)
```

### 主要失败模式

**1. 多数偏差 (Majority Bias)**：
当大多数 Agent 共享相同的系统性偏差时，多数决会放大而非抵消偏差。

```
场景：审核一段代码是否有 bug
Agent₁ (有偏差): "有 bug"  ← 因训练数据过拟合，总是过度报告
Agent₂ (有偏差): "有 bug"  ← 同上
Agent₃ (准确):   "无 bug"
结果：多数决判定"有 bug" ← 错误的结论
```

**2. 知识少数派困境**：
当少数 Agent 拥有专业知识但被多数平庸 Agent 否决时，投票反而降低性能。

```
场景：诊断罕见疾病
5 个全科医生投票 → 多数诊断为普通感冒（正确率各 60%）
1 个专科医生   → 诊断为罕见病（正确率 95%）
多数决结果：感冒 ← 错误！
```

**3. 独立性缺失**：
LLM Agent 训练数据高度重叠，导致判断之间高度相关，严重削弱 Condorcet 陪审团定理的前提条件。

当 Agent 相关性为 $\rho$ 时，多数决准确率随 N 增加的提升幅度大幅降低：
- $\rho = 0$（完全独立）：$P_{correct} \xrightarrow{N \to \infty} 1$
- $\rho > 0$：$P_{correct}$ 的上界受限于相关性，不会趋近于 1

---

## 5. 核心矛盾：公平性 vs 效率

### 核心张力

```
公平性（每 Agent 一票） ←——→ 效率（按能力/置信度加权）

            群体智慧需要独立意见
                    ↕
          LLM Agent 缺乏真正的独立性
```

| 方面 | 公平性优先 | 效率优先 |
|------|-----------|---------|
| 投票权重 | 每 Agent 一票 | 按置信度/准确率加权 |
| 适用场景 | Agent 能力均匀 | Agent 能力差异大 |
| 风险 | 被多数平庸意见淹没 | 少数 Agent 垄断决策 |
| 对独立性的依赖 | 强 | 弱 |

### 加权投票的公平性权衡

加权投票本质上承认某些声音"更重要"，但这带来了新的问题：
- 权重如何确定？谁来决定权重？
- 权重是否公开？Agent 是否会为了获得更高权重而策略性行为？
- 动态权重还是静态权重？

### 群体智慧的前提条件

"Wisdom of Crowds" 效应的成立需要：
1. 观点多样性 (Diversity of opinion)
2. 独立性 (Independence)
3. 去中心化 (Decentralization)
4. 聚合机制 (Aggregation)

—— Surowiecki, *The Wisdom of Crowds*, 2004

LLM Agent 投票在"独立性"和"多样性"方面天然存在短板。

---

## 6. 主流优化方向

### 6.1 加权投票 (Weighted Voting)

每个 Agent 的投票按其置信度、历史准确率或领域专长度加权。

**置信度加权**：Agent 输出预测的同时输出置信度分数，投票时按置信度加权。

```python
def weighted_vote(agents_votes: list[dict], method: str = "confidence"):
    """
    agents_votes: [{"option": "A", "weight": 0.85}, ...]
    返回最终选择的 option
    """
    scores = defaultdict(float)
    for vote in agents_votes:
        scores[vote["option"]] += vote["weight"]
    return max(scores, key=scores.get)
```

**历史准确率加权**：基于 Agent 在验证集上的表现动态调整权重。

$$
w_i = \log\left(\frac{acc_i}{1 - acc_i}\right)
$$

其中 $acc_i$ 是 Agent $i$ 的历史准确率。

### 6.2 Borda 计数 / 排序投票 (Ranked Voting)

Agent 不只投给一个选项，而是对所有选项给出排序。Borda 计数为每个排名分配分数：

- 在有 m 个选项时，排名第 k 的选项获得 m - k 分
- 累加所有 Agent 的分数，总分最高者胜出

**优势**：
- 比单一投票信息量更大
- 天然支持多个选项
- 对极端偏好的鲁棒性更好

### 6.3 二次方投票 (Quadratic Voting)

Agent 购买"票数"，成本与票数的平方成正比：

$$
\text{Cost}(v) = v^2
$$

其中 $v$ 是为某个选项投的票数（可以投多张票给同一选项）。

**优势**：允许 Agent 表达偏好强度，同时通过非线性成本防止主导。

**适用场景**：当不同 Agent 对决策结果的利益相关程度差异很大时。

### 6.4 液态民主 (Liquid Democracy)

每个 Agent 可以选择：
1. 直接投票
2. 将投票权委托给其他 Agent

委托可以是可传递的（A 委托给 B，B 委托给 C，最终 C 投票）。

**优势**：
- 专业问题由专业 Agent 决定
- 保持灵活性和参与度
- 减少低质量 Agent 的噪声投票

```python
class LiquidDemocracy:
    """
    液态民主投票系统
    Agent 可以投票给选项，或委托给其他 Agent
    """
    def __init__(self):
        self.votes = {}     # agent -> option or delegate_target
        self.weights = {}   # agent -> accumulated weight

    def resolve_delegation(self, agent, visited=None):
        """递归解析委托链，检测循环"""
        if visited is None:
            visited = set()
        if agent in visited:
            raise ValueError(f"Cyclic delegation detected: {visited}")
        visited.add(agent)

        vote = self.votes.get(agent)
        if vote is None:
            return None
        if vote in ["A", "B", "C"]:  # direct vote to an option
            return vote
        return self.resolve_delegation(vote, visited)  # delegate

    def compute_result(self):
        """递归解析所有委托，计算最终投票结果"""
        tally = defaultdict(float)
        for agent in self.votes:
            final_vote = self.resolve_delegation(agent)
            if final_vote:
                tally[final_vote] += self.weights.get(agent, 1.0)
        return max(tally, key=tally.get)
```

### 6.5 多轮投票+协商 (Multi-round Voting with Deliberation)

Agent 在每轮之间交换信息（理由、证据），然后重新投票。

```
Round 1: Agent 各自投票，不交流
    ↓ （结果公布 + 各方给出理由）
Discourse: Agent 讨论，质询，提供证据
    ↓
Round 2: Agent 根据新信息重新投票
    ↓ （可能重复多轮）
Final: 聚合最终投票
```

**优势**：
- 信息共享帮助纠正错误认知
- 促进趋近更优解
- 模拟人类委员会的讨论机制

### 6.6 置信度阈值投票 (Confidence-threshold Voting)

Agent 只在置信度超过某个阈值时才投票，否则弃权。

```python
def confidence_threshold_vote(votes, threshold=0.7):
    """只有置信度 >= threshold 的 Agent 参与投票"""
    eligible = [v for v in votes if v["confidence"] >= threshold]
    if not eligible:
        return None  # 无人达到置信度阈值，需要降级处理
    # 对合格票进行等权或加权投票
    tally = defaultdict(int)
    for v in eligible:
        tally[v["option"]] += 1
    return max(tally, key=tally.get)
```

**优势**：
- 减少低质量投票的噪声
- 自动处理"不知道"的情况
- 提高投票结果的信噪比

---

## 7. 能力边界

### 投票机制擅长的场景

| 场景 | 说明 | 典型效果 |
|------|------|---------|
| 评估 (Evaluation) | 多个 Judge 打分+聚合 | 比单个 Judge 更稳定 |
| 路由 (Routing) | Agent 投票决定下一步动作 | 减少单 Agent 的路径偏差 |
| 选择 (Selection) | 从多个候选中选最优 | Bagging/集成方法的核心理念 |
| 分类 (Classification) | 多分类器投票 | 随机森林的成功验证 |
| 二元决策 (Binary Decision) | 是/否投票 | Condorcet 陪审团定理保证 |

### 投票机制不擅长的场景

**1. 创造性共识 (Creative Consensus)**：
- 投票擅长"选择"而非"创造"
- 多个 Agent 对同一篇文本评价后投票选择最佳版本可以
- 但投票无法直接生成一个融合多个创意的合成方案

**2. 盲点共享 (Shared Blind Spots)**：
- 当所有 Agent 都缺失同一信息或共享同一偏见时，投票毫无帮助
- 因为所有投票产生了相同的系统性偏移
- 例：所有 LLM 都未在训练数据中见过某新领域的知识，投票无法弥补

**3. 主观价值判断 (Subjective Value Judgment)**：
- 投票偏好方向本身存在争议（如：应优先考虑安全性还是创造性？）
- 投票结果反映的是多数"价值观"而非"正确性"
- 在价值观分歧的场景下，多数决可能压制少数群体

**4. 信息层级不对称 (Asymmetric Information)**：
- 当某个 Agent 拥有其他 Agent 完全不知道的关键信息时
- 等权投票会浪费这个信息优势
- 加权投票可以部分缓解，但权重校准本身就是一个难题

---

## 8. vs 共识 (Consensus)

| 维度 | 投票 (Voting) | 共识 (Consensus) |
|------|-------------|------------------|
| 决策规则 | 多数决（或加权聚合） | 全体一致 |
| 对分歧的态度 | 容忍分歧，多数拍板 | 必须消弭分歧 |
| 效率 | 高：一轮投票即可出结果 | 低：可能需要多轮协商 |
| 公平感知 | 少数可能感到被忽视 | 所有人都同意（或"足够满意"） |
| 可扩展性 | 好：可以处理大规模群体 | 差：群体越大越难达成共识 |
| 对极端观点 | 可能被多数压制的正确观点 | 必须被纳入考虑 |
| 信息聚合 | 仅聚合偏好/选择 | 聚合偏好+理由+论据 |
| 并行性 | 完全可并行 | 本质上是串行的 |

### 何时选择哪种

- **投票优先**：时间敏感、规模大、存在客观正确标准、需要快速决策
- **共识优先**：决策影响深远、需要全员承诺执行、价值判断核心、安全关键场景

在实践中，**投票+讨论**（多轮投票结合协商）可以结合两者的优点。

---

## 9. 核心优势

### 1. 简单性 (Simplicity)

- 大多数投票规则（多数决、Borda）容易理解和实现
- 不需要复杂的 Agent 间通信协议
- 调试和解释成本低

### 2. 可并行化 (Parallelizability)

- 投票过程天然可并行：所有 Agent 同时投票
- 不依赖 Agent 之间的顺序依赖
- 适合大规模分布式部署

```python
# 并行投票的核心模式 —— 所有 Agent 独立运行
import asyncio

async def parallel_vote(agents, query):
    """所有 Agent 并行投票"""
    tasks = [agent.vote(query) for agent in agents]
    votes = await asyncio.gather(*tasks)
    return aggregate(votes)
```

### 3. 可证明的理论保证 (Provable Guarantees)

- **Condorcet 陪审团定理**：独立性 + p > 0.5 时，多数决收敛到正确
- **单调性**：某选项获得更多支持不会降低其获胜概率
- **参与性**：新增投票者不会使原有群体偏好逆转（对某些规则成立）

### 4. 良好的可扩展性 (Scalability)

- 增加 Agent 数量通常提升结果质量（受独立性条件限制）
- 计算复杂度为 O(N)，可水平扩展
- 不需要全连接通信拓扑

### 5. 鲁棒性 (Robustness)

- 对单个 Agent 的随机故障不敏感
- 对异常的极端投票（outliers）有天然免疫力
- 可以通过冗余投票检测不一致

---

## 10. 工程实现挑战

### 10.1 策略性投票 / 投票操纵 (Strategic Voting / Manipulation)

Agent 可能不按真实偏好投票，而是策略性地操纵结果。

**Gibbard-Satterthwaite 定理**：
> 对于三个或更多选项，任何非独裁的投票规则都是可操纵的 —— 即存在某些情境下，某个投票者通过虚假投票可以得到比诚实投票更好的结果。

**防御策略**：
- 使用操纵难度高的投票规则（如 Borda 比 Plurality 更难操纵）
- 引入随机性（随机化投票规则）
- 将投票理由与投票结果一起审计
- 对历史投票行为进行一致性检测

**LLM Agent 的独特风险**：
- LLM Agent 可能被提示词注入影响投票行为
- 多个 Agent 如果被同一个系统提示词引导，可能产生共谋
- 需要区分"真诚偏见"和"被操纵的投票"

### 10.2 平局处理 (Tie-breaking)

当两个或多个选项获得相同的总票数时，需要有效的破平策略：

```python
def break_tie(options: list, method: str = "random") -> str:
    """多种破平策略"""
    if method == "random":
        return random.choice(options)
    elif method == "preference":
        return options[0]  # 按预定义优先级
    elif method == "confidence":
        # 选择支持者平均置信度最高的选项
        return max(options, key=lambda o: avg_confidence(o))
    elif method == "revote":
        # 仅对平局选项重新投票
        return revote_only(options)
    elif method == "status_quo":
        return "none"  # 保持现状，不做决策
    else:
        raise ValueError(f"Unknown tie-breaking method: {method}")
```

**关键选择**：
- **确定性破平**（固定顺序、循环轮换）：可预测但可能系统性偏向
- **随机破平**（随机选择）：公平但不可预测
- **再投**（只在平局选项上再投一轮）：增加通信成本
- **外部仲裁**：引入更高级 Agent 做最终决定

### 10.3 多轮投票的通信成本 (Communication Cost)

多轮投票 + 协商可以提升质量，但通信成本不可忽视：

| 轮次 | 消息数 | 延迟 | 信息量 |
|------|--------|------|--------|
| 1 轮 (纯投票) | O(N) | 低 | 仅偏好 |
| 2 轮 (投票+理由) | O(N) | 中 | 偏好+理由 |
| 3+ 轮 (多轮协商) | O(N × R) | 高 | 偏好+理由+反驳 |

**优化策略**：
- 限制最大轮数
- 使用增量信息披露（Agent 按需提供更多信息）
- 异步投票（不要求所有 Agent 同时参与）

### 10.4 Agent 共谋检测 (Collusion Detection)

多个 Agent 可能串通影响投票结果：

```python
def detect_collusion(vote_history: list[dict], threshold: float = 0.95):
    """
    检测异常投票模式
    - 完全一致的投票序列
    - 违反独立性的投票模式
    """
    suspicious_pairs = []
    n = len(vote_history)

    for i in range(n):
        for j in range(i + 1, n):
            agreement = vote_agreement_rate(vote_history[i], vote_history[j])
            if agreement > threshold:
                suspicious_pairs.append((i, j, agreement))

    return suspicious_pairs
```

**检测指标**：
- **一致性异常**：两个 Agent 在所有投票中 100% 一致（超出随机期望）
- **互信息**：Agent 投票之间的互信息过高
- **投票时间相关性**：投票时间戳高度相关

### 10.5 置信度校准 (Confidence Calibration)

加权投票的效果高度依赖置信度分数的质量，而 LLM 的置信度校准是一个已知难题。

**常见问题**：
- LLM 倾向于过度自信（输出高置信度但实际不准确）
- 不同 LLM 的置信度尺度不可比
- 置信度受提示词措辞影响

**校准方法**：

```python
def calibrate_confidence(raw_confidence: float, historical_acc: float) -> float:
    """
    使用 Platt Scaling 或 Isotonic Regression 校准
    简单版本：基于历史准确率调整
    """
    # 如果 Agent 的历史准确率远低于其声称的置信度，则降权
    calibration_factor = min(1.0, historical_acc / max(raw_confidence, 0.01))
    return raw_confidence * calibration_factor
```

---

## 11. 适用场景

### LLM-as-Judge 评估

多个 LLM Judge 对同一系统输出打分，通过投票（均值/中位数/多数）聚合为最终分数。

```python
# 多个 LLM 作为 Judge 投票评估输出质量
judges = [gpt4_judge, claude_judge, gemini_judge]
scores = [judge.evaluate(response) for judge in judges]
final_score = statistics.median(scores)  # 中位数对异常值更鲁棒
```

### 内容审核 (Content Moderation)

多个审核 Agent 对同一内容独立判断，投票决定是否违规。加权投票中，专家 Agent（如法律领域微调）可以拥有更高权重。

### 响应选择 (Response Selection)

在 Agent 工作流中，多个 Agent 生成候选响应，投票选择最优：

| 方法 | 说明 | 效果 |
|------|------|------|
| 简单多数 | 每个 Agent 投给其认为最好的响应 | 对噪声敏感 |
| Borda 计数 | 每个 Agent 对所有响应排序 | 信息更丰富 |
| 加权投票 | 按历史表现加权 | 对能力不均更鲁棒 |

### 工具选择 (Tool Selection)

Agent 决定下一步调用哪个工具时，多个 Agent 投票决策：
- 每个 Agent 给出候选工具及其置信度
- 聚合后选择总置信度最高的工具
- 降低单 Agent 错误调用工具的风险

### Agent 集成 (Agent Ensemble)

与模型集成的思路一致，多个异构 Agent 的投票集成：

```python
class AgentEnsemble:
    """异构 Agent 投票集成"""
    def __init__(self, agents: list):
        self.agents = agents

    def predict(self, input_data, voting="soft"):
        if voting == "hard":
            # 多数决
            votes = [a.predict(input_data) for a in self.agents]
            return Counter(votes).most_common(1)[0][0]
        elif voting == "soft":
            # 概率平均
            probas = [a.predict_proba(input_data) for a in self.agents]
            avg_proba = np.mean(probas, axis=0)
            return np.argmax(avg_proba)
```

### 路由决策 (Routing Decisions)

在 Multi-Agent 系统中决定任务如何路由：
- 投票决定任务的优先级
- 投票决定哪个 Agent 子团队最适合处理当前任务
- 加权投票中，空闲容量大的 Agent 获得更高路由权重

---

## 代码示例

### 示例 1：加权投票系统 (Weighted Voting with Confidence Calibration)

```python
"""
加权投票系统示例
- 多个 Agent 对选项投票，附带置信度分数
- 基于历史准确率校准置信度
- 支持多种权重计算策略
"""
from collections import defaultdict
import numpy as np


class WeightedVotingSystem:
    """
    加权投票系统，支持置信度校准和多种聚合策略
    """

    def __init__(self, calibration_mode: str = "historical"):
        """
        calibration_mode:
            "historical" - 基于历史准确率校准
            "raw"        - 使用原始置信度
            "softmax"    - 对同一 Agent 的置信度做 softmax 归一化
        """
        self.calibration_mode = calibration_mode
        self.history = {}  # agent_id -> list of (confidence, was_correct)

    def record_outcome(self, agent_id: str, confidence: float, was_correct: bool):
        """记录一次投票结果，用于后续校准"""
        if agent_id not in self.history:
            self.history[agent_id] = []
        self.history[agent_id].append((confidence, was_correct))

    def get_agent_reliability(self, agent_id: str) -> float:
        """获取 Agent 的历史准确率（用于权重计算）"""
        if agent_id not in self.history or len(self.history[agent_id]) < 5:
            return 0.5  # 数据不足时假设为随机水平
        corrects = [1 for _, c in self.history[agent_id] if c]
        return sum(corrects) / len(self.history[agent_id])

    def calibrate_confidence(self, agent_id: str, raw_confidence: float) -> float:
        """基于历史表现校准置信度"""
        if self.calibration_mode == "raw":
            return raw_confidence

        historical_acc = self.get_agent_reliability(agent_id)

        if self.calibration_mode == "historical":
            # Agent 声称的置信度 vs 实际准确率的差距
            avg_conf = np.mean([c for c, _ in self.history.get(agent_id, [(0.5,)])])
            calibration_factor = min(1.0, historical_acc / max(avg_conf, 0.01))
            return raw_confidence * calibration_factor

        if self.calibration_mode == "softmax":
            return raw_confidence  # softmax 在全局归一化时做

        return raw_confidence

    def aggregate(self, votes: list[dict], strategy: str = "weighted") -> str:
        """
        聚合投票结果

        votes: [{"agent_id": "A1", "option": "X", "confidence": 0.9}, ...]
        strategy:
            "weighted"    - 按校准后置信度加权
            "majority"    - 等权多数决
            "threshold"   - 仅统计置信度 > 阈值的投票
        """
        if strategy == "majority":
            tally = defaultdict(int)
            for v in votes:
                tally[v["option"]] += 1
            return max(tally, key=tally.get)

        if strategy == "threshold":
            threshold = 0.7
            eligible = [v for v in votes
                        if self.calibrate_confidence(v["agent_id"], v["confidence"]) >= threshold]
            if not eligible:
                return "ABSTAIN"
            tally = defaultdict(int)
            for v in eligible:
                tally[v["option"]] += 1
            return max(tally, key=tally.get)

        # strategy == "weighted": 置信度加权投票
        scores = defaultdict(float)
        for v in votes:
            weight = self.calibrate_confidence(v["agent_id"], v["confidence"])
            scores[v["option"]] += weight

        return max(scores, key=scores.get)


# ========== 使用示例 ==========

if __name__ == "__main__":
    wvs = WeightedVotingSystem(calibration_mode="historical")

    # 模拟 Agent 投票（假设问题是：这段代码有 bug 吗？）
    votes = [
        {"agent_id": "gpt4",        "option": "有 bug", "confidence": 0.85},
        {"agent_id": "claude3",     "option": "无 bug", "confidence": 0.92},
        {"agent_id": "gemini",      "option": "有 bug", "confidence": 0.70},
        {"agent_id": "llama3",      "option": "有 bug", "confidence": 0.60},
    ]

    # 模拟历史表现（训练校准数据）
    for aid in ["gpt4", "claude3", "gemini", "llama3"]:
        # 假设 claude3 历史准确率最高
        import random
        for _ in range(20):
            conf = random.uniform(0.5, 0.95)
            correct = random.random() < {
                "gpt4": 0.70, "claude3": 0.88, "gemini": 0.65, "llama3": 0.55
            }.get(aid, 0.5)
            wvs.record_outcome(aid, conf, correct)

    # 对比不同策略
    print(f"Majority vote:        {wvs.aggregate(votes, 'majority')}")
    print(f"Weighted vote:        {wvs.aggregate(votes, 'weighted')}")
    print(f"Threshold vote:       {wvs.aggregate(votes, 'threshold')}")

    # 输出每个 Agent 的校准后权重
    print("\nCalibrated weights:")
    for v in votes:
        cal = wvs.calibrate_confidence(v["agent_id"], v["confidence"])
        print(f"  {v['agent_id']}: {v['confidence']:.2f} -> {cal:.3f}")
```

### 示例 2：Borda 计数排序投票 (Ranked Voting)

```python
"""
Borda 计数投票系统
Agent 对所有选项给出排序，而不是单一投票
"""
from collections import defaultdict


class BordaCountVoting:
    """
    Borda 计数排序投票

    每个 Agent 对 m 个选项给出完整排序，
    排名第 k 的选项获得 (m - k) 分
    """

    def __init__(self, options: list[str]):
        self.options = options
        self.m = len(options)

    def borda_score(self, rank: int) -> int:
        """
        计算某排名的 Borda 分数
        排名从 0 开始（0 = 最高排名）
        """
        return self.m - 1 - rank

    def aggregate(self, rankings: list[list[str]]) -> dict:
        """
        聚合多个 Agent 的排序投票

        rankings: [
            ["C", "A", "B"],     # Agent 1: C > A > B
            ["A", "B", "C"],     # Agent 2: A > B > C
            ["A", "C", "B"],     # Agent 3: A > C > B
        ]
        返回每个选项的总分
        """
        scores = defaultdict(int)

        for ranking in rankings:
            for rank, option in enumerate(ranking):
                scores[option] += self.borda_score(rank)

        return dict(scores)

    def get_winner(self, rankings: list[list[str]]) -> str:
        """返回总分最高的选项"""
        scores = self.aggregate(rankings)
        return max(scores, key=scores.get)

    def get_full_ranking(self, rankings: list[list[str]]) -> list[str]:
        """返回所有选项的完整排序（Borda 总分降序）"""
        scores = self.aggregate(rankings)
        return sorted(scores.keys(), key=lambda o: scores[o], reverse=True)


# ========== 使用示例 ==========

if __name__ == "__main__":
    # 场景：4 个 LLM 生成的回复，由 5 个 Judge Agent 排序投票
    options = ["Response_A", "Response_B", "Response_C", "Response_D"]

    borda = BordaCountVoting(options)

    # 5 个 Judge 给出的排序
    judges_rankings = [
        ["Response_A", "Response_B", "Response_C", "Response_D"],
        ["Response_A", "Response_C", "Response_B", "Response_D"],
        ["Response_B", "Response_A", "Response_C", "Response_D"],
        ["Response_A", "Response_B", "Response_D", "Response_C"],
        ["Response_C", "Response_A", "Response_B", "Response_D"],
    ]

    scores = borda.aggregate(judges_rankings)
    winner = borda.get_winner(judges_rankings)
    full_ranking = borda.get_full_ranking(judges_rankings)

    print("Borda Count Results:")
    print(f"  Raw scores: {scores}")
    print(f"  Winner:     {winner}")
    print(f"  Ranking:    {full_ranking}")

    # 对比简单的多数决
    print("\nComparison with Plurality (first-preference only):")
    plurality = defaultdict(int)
    for r in judges_rankings:
        plurality[r[0]] += 1
    print(f"  Plurality scores: {dict(plurality)}")
    print(f"  Plurality winner: {max(plurality, key=plurality.get)}")
```

### 示例 3：多轮投票 + 协商 (Bonus)

```python
"""
多轮投票 + 协商示例
Agent 在多轮之间交换信息和理由，逐步收敛
"""
import random
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class Agent:
    """模拟 Agent，有"真实倾向"和可说服性"""
    name: str
    bias: float          # 对选项 A 的初始偏好（0~1），>0.5 倾向 A
    persuadability: float = 0.3  # 受他人影响的程度
    current_opinion: float = 0.5  # 当前对 A 的偏好
    arguments: list = field(default_factory=list)  # 收集的论据
    vote_history: list = field(default_factory=list)

    def initialize(self):
        self.current_opinion = self.bias

    def vote(self) -> str:
        """基于当前意见投票"""
        vote = "A" if self.current_opinion > 0.5 else "B"
        self.vote_history.append(vote)
        return vote

    def get_confidence(self) -> float:
        """置信度 = 意见的极端程度"""
        return abs(self.current_opinion - 0.5) * 2

    def deliberate(self, other_votes: list[tuple]):
        """听取其他 Agent 的投票和理由，更新自己的意见"""
        avg_other = sum(
            (1.0 if v == "A" else 0.0) for v, _ in other_votes
        ) / len(other_votes) if other_votes else 0.5

        # 混合自己的意见和他人的意见
        self.current_opinion = (
            (1 - self.persuadability) * self.current_opinion
            + self.persuadability * avg_other
        )

        # 记录论据（简化模拟）
        for _, args in other_votes:
            self.arguments.extend(args[:2])  # 每个 Agent 最多吸收 2 个理由


def multi_round_vote(agents: list, max_rounds: int = 5, convergence_threshold: float = 0.9):
    """
    多轮投票 + 协商

    每轮：
    1. 所有 Agent 投票
    2. 如果达到收敛（同一选项票数占比 >= threshold），提前结束
    3. 否则交换理由，Agent 更新意见，进入下一轮
    """
    n = len(agents)
    for agent in agents:
        agent.initialize()

    for round_num in range(1, max_rounds + 1):
        print(f"\n{'='*50}")
        print(f"Round {round_num}")

        # 投票阶段
        votes_with_args = []
        for agent in agents:
            vote = agent.vote()
            # 模拟生成投票理由
            args = [f"{agent.name}: Evidence supporting {vote}"]
            votes_with_args.append((vote, args))
            print(f"  {agent.name}: votes for {vote} (confidence={agent.get_confidence():.2f})")

        # 检查收敛
        vote_counts = defaultdict(int)
        for v, _ in votes_with_args:
            vote_counts[v] += 1

        dominant = max(vote_counts.values())
        ratio = dominant / n

        print(f"  A: {vote_counts.get('A', 0)}, B: {vote_counts.get('B', 0)}, "
              f"dominant ratio={ratio:.2f}")

        if ratio >= convergence_threshold:
            winner = max(vote_counts, key=vote_counts.get)
            print(f"\n>>> Converged! Winner: {winner} after {round_num} rounds")
            return winner

        # 协商阶段：交换信息和理由
        if round_num < max_rounds:
            print(f"\n  --- Deliberation phase ---")
            for agent in agents:
                others = [(v, a) for a_agent, (v, a) in zip(agents, votes_with_args)
                          if a_agent is not agent]
                agent.deliberate(others)
                print(f"  {agent.name} updated opinion to {agent.current_opinion:.3f}")

    # 最终投票
    winner = max(vote_counts, key=vote_counts.get)
    print(f"\n>>> Max rounds reached. Winner: {winner}")
    return winner


# ========== 使用示例 ==========

if __name__ == "__main__":
    # 创建一组 Agent
    agents = [
        Agent("Expert_A", bias=0.85, persuadability=0.15),  # 强偏好 A，不易说服
        Agent("Expert_B", bias=0.20, persuadability=0.15),  # 强偏好 B，不易说服
        Agent("Neutral_1", bias=0.55, persuadability=0.4),  # 略偏 A，较易被说服
        Agent("Neutral_2", bias=0.60, persuadability=0.3),  # 偏 A，中等可说服
        Agent("Novice", bias=0.50, persuadability=0.7),     # 中立，极易被说服
    ]

    print("="*50)
    print("Multi-Round Voting Simulation")
    print(f"Agents: {[a.name for a in agents]}")
    print(f"Initial biases: {[a.bias for a in agents]}")

    result = multi_round_vote(agents, max_rounds=4)

    # 输出投票历史
    print("\n\nVote history per agent:")
    for agent in agents:
        print(f"  {agent.name}: {agent.vote_history}")
```

---

## 总结

投票机制是一种简洁、高效、可扩展的多 Agent 协调方法。其核心价值在于利用"群体智慧"降低单 Agent 决策的方差和偏差。然而，投票并非万能灵药：

- **有独立性假设前提**：LLM Agent 的共享训练数据破坏了独立性，削弱了理论保证
- **不擅长创造**：投票擅长"选择"而非"生成"
- **需要合理的权重设计**：等权投票在 Agent 能力不均时表现不佳
- **可能被操纵**：策略性投票、提示注入、共谋等问题需要工程化防御

投票在不同多 Agent 协作场景中，常作为**最终决策层**与其他协调机制（如共识、层级路由、市场机制）组合使用，形成一个完整的协作决策管线。
