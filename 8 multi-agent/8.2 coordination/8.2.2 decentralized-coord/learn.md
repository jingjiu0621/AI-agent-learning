# 8.2.2 去中心化协调：市场 / 拍卖 / 共识 (Decentralized Coordination)

## 1. 简单介绍

去中心化协调（Decentralized Coordination）指多个智能体在没有中央控制器或单一决策节点的情况下，通过局部交互、市场机制或共识协议达成全局一致的行为协调。每个智能体自主决策，仅依赖局部信息或有限通信，但整体系统却能够展现出有序、高效的协作行为。

与集中式协调不同，去中心化协调中不存在一个"知道一切、决定一切"的中心节点。协调结果从智能体之间的竞争、协商、投票或局部规则中涌现出来。这种范式在规模大、环境动态、或信任关系复杂的多智能体系统中尤其重要。

典型代表包括：基于拍卖的任务分配、基于市场价格的资源调度、基于共识协议的区块链式协调，以及基于局部规则的涌现式协调（如蚁群算法、鱼群行为）。

---

## 2. 基本原理

去中心化协调的核心机制可以从三个维度理解：

### (1) 市场机制 (Market-Based Coordination)

类比经济学中的"看不见的手"——每个智能体作为理性经济人，追求自身效用最大化。通过价格信号调节供需：

- **定价机制**：资源或任务具有价格，智能体根据自身需求出价
- **供需平衡**：价格随需求上升、随供给下降，最终达到均衡
- **资源配置效率**：在理想假设下，市场均衡能够实现 Pareto 最优
- **典型应用**：云计算资源竞价、机器人任务市场、智能电网调度

基本原理形式化：每个智能体 $i$ 拥有效用函数 $U_i(x)$，在预算约束 $p \cdot x \leq B_i$ 下最大化效用。市场价格 $p$ 由供需关系 $D(p) = S(p)$ 决定。

### (2) 拍卖机制 (Auction-Based Coordination)

拍卖是市场机制的一种特殊形式，用于在有限时间内完成单轮或多轮资源分配：

- **任务委托方**发布任务（商品），附带任务描述和约束
- **候选执行方**根据自身能力报价
- **拍卖机制**决定中标者和最终价格
- **关键设计维度**：密封 vs 公开、单轮 vs 多轮、统一价格 vs 歧视价格

### (3) 共识机制 (Consensus-Based Coordination)

智能体通过信息交换和迭代更新，在没有中央聚合器的情况下就某个值或状态达成一致：

- **一致性协议**：所有智能体最终对同一值达成一致
- **拜占庭容错**：即使部分智能体存在故障或恶意行为，仍能达成共识
- **顾拜协议 (Gossip)**：智能体随机选择邻居交换信息，信息最终扩散至全体
- **典型应用**：区块链共识、分布式状态估计、多机器人共同认知

---

## 3. 背景与演进

去中心化协调的根源可以追溯到分布式计算系统，但其在 MAS 中的演进经历了多个阶段：

### 分布式系统基础（1970s -- 1990s）

- **Gossip 协议**（1987）：最初用于 Usenet 和分布式数据库，节点随机与邻居交换信息，保证信息最终扩散至全网
- **分布式哈希表 DHT**（2001）：Chord、Pastry、Kademlia 等协议，为 P2P 网络提供无中心的路由和查找
- **拜占庭将军问题**（1982）：奠定了容错共识的理论基础

### 多智能体系统兴起（1990s -- 2000s）

- **Contract Net Protocol**（1980, Smith）：首次将拍卖概念引入 agent 任务分配
- **市场导向编程 (Market-Oriented Programming)**（1993, Wellman）：用计算经济学模型设计 agent 交互
- **Swarm Intelligence**（1990s）：从昆虫/鱼群行为中抽象出基于局部规则的涌现协调

### 现代发展（2010s -- 至今）

- **区块链与去中心化金融**：为 MAS 提供了天然的不可信环境下协调基础设施
- **大规模多机器人系统**：100+ 机器人需要完全去中心化的任务分配和路径协调
- **边缘计算与 IoT**：低延迟、高动态环境的去中心化资源调度
- **Federated Learning**：去中心化的模型训练与协调

---

## 4. 之前做法与结果

### 集中式协调的问题

在多智能体系统发展的早期，研究者主要依赖集中式协调器：

| 方面 | 集中式做法 | 出现的问题 |
|------|-----------|-----------|
| 任务分配 | 中心规划器统一分配 | 单点瓶颈，O(n) 或更差的计算复杂度 |
| 资源调度 | 全局视图统一调度 | 通信集中的 C10k 问题（高连接数下性能急剧下降） |
| 冲突解决 | 中心仲裁 | 延迟随 agent 数量线性增长 |
| 知识融合 | 全局黑板（Blackboard）架构 | 写冲突、信息过载 |

### 关键发现

1. **可扩展性断崖（Scalability Cliff）**：当 agent 数量超过 30-50 时，集中式协调器的响应时间呈非线性增长
2. **脆弱性（Fragility）**：中心节点失效导致整个系统瘫痪
3. **环境敏感度（Brittleness）**：在动态环境中，集中式规划的全局最优解因环境变化而迅速失效
4. **通信成本**：所有 agent 向中心汇报——通信量是 O(n)，中心决策再下发

这些局限性驱使研究者探索去中心化方案，并在 2000 年代后逐渐成为多智能体协调的主流范式。

---

## 5. 核心矛盾：自主性 vs 一致性 (Autonomy vs Coherence)

去中心化协调面临的核心张力：

- **自主性 (Autonomy)**：每个 agent 独立决策，拥有自己的局部目标、局部信息和局部利益
- **一致性 (Coherence)**：系统需要全局一致的行为，避免冲突，避免资源浪费，达成共同目标

| 维度 | 高度自主性 | 高度一致性 |
|------|-----------|-----------|
| 决策速度 | 快（无需等待协调） | 慢（需要达成共识） |
| 全局效率 | 可能局部最优、全局次优 | 接近全局最优 |
| 鲁棒性 | 单点故障不影响 | 协调协议本身可能脆弱 |
| 适应性 | 快速适应局部变化 | 响应全局变化慢 |
| 实现复杂度 | 简单 | 需要复杂协调协议 |

### 缓解策略

- **分层协调**：高自主性局部决策 + 低频率全局协调
- **市场约束**：价格信号引导自主决策方向
- **社会规范**：预设规则约束自主行为的空间
- **声誉机制**：长期博弈中自我约束

---

## 6. 主流优化方向

### 6.1 Contract Net Protocol (CNP)

**Contract Net Protocol** 是 MAS 中最经典的基于拍卖的任务分配协议，1980 年由 Reid G. Smith 提出。

**流程：**

```
Task Manager Agent                 Contractor Agents
      |                                   |
      |--- 1. 任务公告 (announcement) ---->|
      |                                   |
      |<--- 2. 投标 (bid) ----------------|
      |                                   |
      |--- 3. 中标通知 (award) ---------->|
      |                                   |
      |<--- 4. 结果汇报 (result) ---------|
```

**核心特性：**
- 任务公告包含：任务描述、截止时间、资质要求
- 投标包含：能力声明、预计成本、完成时间
- 中标后承包商可再分解任务进行子委托
- 支持多轮谈判（迭代 CNP）

### 6.2 拍卖机制 (Auction Mechanisms)

不同拍卖机制适用于不同的多智能体协调场景：

| 拍卖类型 | 出价规则 | 价格确定 | MAS 典型应用 |
|----------|---------|---------|-------------|
| **英式拍卖 (English)** | 公开递增出价 | 最高出价 | 公共资源竞争 |
| **荷式拍卖 (Dutch)** | 公开递减报价 | 首个接受的报价 | 时效性强的任务（如物流） |
| **维克里拍卖 (Vickrey)** | 密封投标 | 次高价成交 | 鼓励诚实报价，防止市场操纵 |
| **组合拍卖 (Combinatorial)** | 对资源组合投标 | 最优组合 | 多资源复杂任务 |
| **双向拍卖 (Double)** | 买卖双方出价 | 匹配价格 | 电力市场、金融交易 |

**Vickrey 拍卖 (次高价拍卖)** 的核心优势：激励兼容——诚实报价是每个 agent 的占优策略（dominant strategy）。

### 6.3 基于市场的协调 (Market-Based Coordination)

将经济学的 Walrasian 均衡应用于多智能体系统：

1. **定价机制**：计算资源、通信带宽、任务 = 商品
2. **初始分配**：每个 agent 拥有初始资源票赋
3. **迭代交易**：agent 根据当前价格计算需求，报价买卖
4. **价格更新**：若超额需求 > 0，涨价；若超额供给 > 0，降价
5. **均衡到达**：所有市场出清

**"计算经济学" 方法**：使用一般均衡理论（General Equilibrium Theory）中的 Tatonnement 过程逐步收敛到均衡价格。

### 6.4 Gossip 协议 (Gossip / Epidemic Protocols)

用于去中心化系统中信息传播和同步的轻量级协议：

- **反熵 (Anti-Entropy)**：节点定期随机选择另一个节点，交换所有更新
- **谣言传播 (Rumor-Mongering)**：只交换新信息，收到即传播
- **推模式 (Push)**：主动向邻居发送信息
- **拉模式 (Pull)**：主动向邻居请求更新
- **混合模式 (Push-Pull)**：同时推和拉

**关键性质**：O(log n) 轮次即可将信息传播到所有节点，对节点故障具有天然容错性。

### 6.5 分布式约束优化 (DCOP)

DCOP 是一种形式化框架，用于分布式而非集中式地解决约束满足问题：

- **变量**：每个 agent 控制一个或多个变量
- **约束**：变量之间必须满足局部约束
- **目标**：分布式找到最大化总体效用的变量赋值

**经典算法**：

| 算法 | 机制 | 特点 |
|------|------|------|
| **ADOPT (2002)** | 深度优先搜索 + 回溯 | 完备最优，但通信量大 |
| **DPOP (2005)** | 动态规划 + 伪树 | 高效，但消息体积指数增长 |
| **Max-Sum (2007)** | 因子图 + 信念传播 | 近似最优，可扩展性好 |
| **DSA (2002)** | 分布式随机算法 | 快速近似，简单易实现 |

```
Agent A ─── constraint: A ≠ B ─── Agent B
    │                                 │
    │          constraint:            │
    │         A + C = value           │
    │                                 │
    └──────── constraint: B > C ──────┘
                  │
            Agent C
```

### 6.6 涌现式协调与群体智能 (Emergent Coordination / Swarm Intelligence)

通过设计局部交互规则，让全局协调行为从个体行为中自然涌现：

- **Boids 模型**（1986, Reynolds）：分离、对齐、聚集三条简单规则产生鱼群/鸟群行为
- **蚁群优化 (ACO)**：蚁群通过信息素轨迹实现去中心化路径优化
- **粒子群优化 (PSO)**：群体通过个体最优和全局最优引导搜索
- **反应式 agent 编队**：机器人根据邻居相对位置调整自身

**核心思想**：全局有序 = 局部规则 + 随机性 + 正反馈循环。

---

## 7. 能力边界

### 优势区间

- **大规模系统**（> 100 agents）：去中心化的优势明显，通信和计算负载在节点间均匀分布
- **高动态环境**：快速响应局部变化，无需等待中心重新规划
- **异构系统**：不同机构、不同利益主体的 agent 可在统一市场框架下协作
- **不可信环境**：无需信任中心节点，共识机制提供安全保障

### 局限性

| 限制 | 描述 | 影响程度 |
|------|------|---------|
| **收敛时间** | 尤其是共识机制和迭代拍卖，到达稳定解需要多轮通信 | 随 agent 数量和问题复杂度非线性增长 |
| **通信爆炸** | 完全连接拓扑下的广播和 all-to-all 通信导致 O(n²) 消息量 | 网络带宽成为瓶颈 |
| **次优性** | 去中心化决策通常无法保证全局最优 | 在某些问题上性能差距可达 30-50% |
| **自由骑手问题 (Free-Rider)** | 不承担成本但享受协调收益的 agent 破坏系统 | 需要惩罚机制或声誉系统 |
| **博弈操纵** | agent 可能策略性虚报成本和需求 | 需要 incentive-compatible 的机制设计 |
| **状态同步滞后** | 不同 agent 持有不同的全局状态视图 | 在高动态系统中导致决策冲突 |

---

## 8. vs 集中式协调 (Centralized vs Decentralized)

| 对比维度 | 集中式 (Centralized) | 去中心化 (Decentralized) |
|---------|---------------------|------------------------|
| **架构** | 一个主控 + 多个执行 | 对等节点 (Peer-to-Peer) |
| **决策速度** | 快（O(1) 决策） | 慢（需要协商/多轮通信） |
| **通信模式** | 星形拓扑（中心-边缘） | 复杂图拓扑 |
| **全局最优性** | 可达全局最优 | 通常仅能近似最优 |
| **容错性** | 中心节点单点故障 | 天然容错，无单点故障 |
| **扩展性** | 中心节点成为瓶颈 | 随规模线性或亚线性扩展 |
| **实现难度** | 较简单 | 较复杂（协议设计、一致性） |
| **可调试性** | 中心日志可追踪全局 | 涌现行为难以复现和调试 |
| **安全性** | 中心节点是攻击目标 | 攻击面更广但无致命弱点 |
| **预测性** | 行为可预测 | 涌现行为难以预测 |
| **典型例** | 机器人集中规划、SWARM 中心调度 | 区块链、CNP、Gossip |

### 混合架构 (Hybrid)

实践中，多数系统采用混合架构——在关键协调点使用集中式决策，在规模化和灵活性区域使用去中心化协调：

- **层级市场**：局部市场自主协调 + 全局市场处理跨域资源
- **动态角色切换**：正常时去中心化，异常时选举临时协调者（如 Raft 协议的 Leader）
- **分片 (Sharding)**：组内集中协调 + 组间去中心化协调

---

## 9. 核心优势

### 9.1 无单点故障

系统中不存在任何一个节点，其失效会导致全局瘫痪。即使 30% 的节点离线，基于 Gossip 和共识的系统仍可正常运行。这在以下几个场景中尤为关键：

- **太空 / 深海 / 灾难响应**：通信不稳定，不能依赖固定中心
- **军事系统**：中心节点是首要攻击目标
- **大规模 IoT**：接入节点可能掉线、电池耗尽或网络断开

### 9.2 自然可扩展

- 计算负载、通信负载、存储负载均在节点间均匀分布
- 增加 agent 通常不会导致系统性能断崖式下降
- 理论上系统可扩展到数万甚至数十万节点

### 9.3 涌现智能 (Emergent Intelligence)

- 简单的局部规则 + 局部交互 = 复杂的全局协调行为
- 系统展现出设计者未曾明确编码的适应性和创造性
- 类似于蚁群能够找到最短路径、鱼群能够躲避捕食者

### 9.4 对 Agent 变更的鲁棒性

- agent 可随时加入/离开系统而不需要全局重新配置
- 适应 agent 能力的变化（变强/变弱/故障）
- 无需停止整个系统进行维护或升级

---

## 10. 工程挑战

### 10.1 消息爆炸问题

在完全连接拓扑或全广播协议中，消息量级可达 O(n²)：

```
n = 10   → 45  条双向消息
n = 100  → 4950 条消息
n = 1000 → 499500 条消息
```

**缓解方法**：
- 不是所有 agent 都需要相互通信——使用局部邻居图
- Gossip 中限制每次随机选择的邻居数（如 fanout = 3-5）
- 消息聚合——将多个决策打包成单个消息
- 通信拓扑优化：树 / 环 / 小世界网络

### 10.2 市场操纵与博弈

如果 agent 是策略性的（strategic），市场机制可能被操纵：

- **合谋 (Collusion)**：多个 agent 联合操纵价格
- **虚假报价 (Shill Bidding)**：卖家安排虚假买家推高价格
- **需求隐藏 (Demand Hiding)**：买家低报需求以压低价格
- **空头投标 (Sniping)**：最后时刻低价中标

**应对方案**：
- Vickrey 拍卖（次高价 = 激励兼容）
- 隐式投标（同态加密、安全多方计算）
- 声誉系统 + 长期博弈惩罚（回顾式惩罚机制）
- 零知识证明验证 agent 报价的合理性

### 10.3 共识收敛慢

在分布式共识中，收敛到一致意见的时间与系统的规模和网络延迟密切相关：

- **理论下限**：在存在拜占庭故障时，至少需要 3f+1 个节点容忍 f 个故障
- **消息轮次**：Paxos / Raft 需要至少 2 轮 RTT 完成一次共识
- **信息不对称**：agent 持有的信息不同，需要多轮交换才能对齐

**加速方法**：
- 推测执行（Optimistic Execution）
- 流水线共识（Pipeline Consensus）
- 基于信誉的权重投票（减少投票者数量）
- 概率性共识（最终一致性，非严格一致性）

### 10.4 涌现行为难以调试

这是去中心化系统最大的工程挑战之一：

- **复现困难**：相同输入在不同运行中可能产生不同行为（随机性 + 时序）
- **全局状态不可知**：没有节点能看到完整系统状态
- **因果链不清晰**：局部决策如何影响全局行为难以追踪
- **相变行为**：参数微小变化可能导致系统行为模式突变

**调试策略**：
- 确定性重放日志（Deterministic Replay Logging）
- 分布式追踪（如 OpenTelemetry + 因果标签）
- 形式化验证（TLA+、Alloy 建模）
- 模拟环境先行验证，再逐步扩展到真机
- 引入可观察性的"元 agent"——观察但不干预

### 10.5 激励兼容的机制设计

要让自私的 agent 真实报告信息、遵守协议，必须设计 incentive-compatible 的机制：

```
核心问题：
  真实报告收益 > 策略性误报收益

设计原则：
  1. 揭示原理 (Revelation Principle)
  2. 占优策略激励兼容 (DSIC)
  3. 个体理性 (Individual Rationality)
  4. 预算平衡 (Budget Balance)
```

**Myerson-Satterthwaite 不可能定理**表明：在信息不对称下，不存在一种机制同时满足效率、预算平衡和个体理性。工程中必须在三者之间选择取舍。

---

## 11. 适用场景

### 11.1 大规模资源分配

- **云计算 Spot 实例定价**：AWS/Azure/Google Cloud 使用市场机制定价，用户竞价获取闲置资源
- **智能电网**：分布式能源（太阳能、风电）的 local energy market，住户之间交易余电
- **频谱分配**：无线通信中认知无线电的分布式频谱接入

### 11.2 分布式感知 (Distributed Sensing)

- **环境监测**：大量传感器自主协调采样频率和覆盖区域
- **无人机群感知**：无人机群去中心化分配巡查区域，最大化覆盖
- **卫星任务规划**：多卫星自主协调拍摄任务

### 11.3 多机器人协调

- **仓储物流**：Kiva / Amazon Robotics 系统的去中心化任务分配
- **搜索救援**：多机器人去中心化搜索目标区域
- **协同建图**：多机器人分布式 SLAM，局部图合并
- **自动驾驶车队**：车辆之间通过协商协调换道、交叉口通行

### 11.4 众包 (Crowdsourcing)

- **Waze 路线推荐**：用户上传路况信息，去中心化路由推荐
- **OpenStreetMap 编辑**：众包地图的冲突解决和版本协调
- **知识图谱协作编辑**：去中心化的事实一致性校核

### 11.5 区块链与 Web3

- **智能合约自动化 agent**：在 DeFi 中自动做市、套利的 agent 通过链上拍卖交互
- **去中心化自治组织 (DAO)**：agent 投票、提案的共识协调
- **跨链桥的验证者协调**

---

## 代码示例

### 示例 1：Contract Net Protocol 实现

```python
"""
Contract Net Protocol (CNP) Implementation
A task manager announces a task, receives bids from contractor agents,
evaluates them, and awards the contract.
"""
from __future__ import annotations
import uuid
import time
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional


class TaskStatus(Enum):
    ANNOUNCED = "announced"
    BIDDING = "bidding"
    AWARDED = "awarded"
    COMPLETED = "completed"
    FAILED = "failed"


@dataclass
class Task:
    """A task to be allocated via CNP."""
    task_id: str
    description: str
    deadline: float          # latest completion time
    required_skill: str
    estimated_effort: float  # in abstract units

    def __post_init__(self):
        if not self.task_id:
            self.task_id = str(uuid.uuid4())


@dataclass
class Bid:
    """A bid submitted by a contractor agent."""
    bidder_id: str
    task_id: str
    offered_price: float          # cost to perform
    estimated_completion_time: float
    skill_match_score: float      # 0.0 to 1.0
    timestamp: float = field(default_factory=time.time)


@dataclass
class ContractAward:
    """The award decision sent to the winning contractor."""
    task_id: str
    awarded_to: str
    award_price: float
    terms: str = "Standard CNP terms apply."


class ContractorAgent:
    """An agent that can bid on and execute tasks."""

    def __init__(self, agent_id: str, skills: dict[str, float], cost_per_unit: float):
        self.agent_id = agent_id
        self.skills = skills                # skill -> proficiency (0.0-1.0)
        self.cost_per_unit = cost_per_unit
        self.current_tasks: list[Task] = []
        self.max_concurrent = 3

    def can_handle(self, task: Task) -> bool:
        """Check if this agent can take on the task."""
        if len(self.current_tasks) >= self.max_concurrent:
            return False
        if task.required_skill not in self.skills:
            return False
        return True

    def prepare_bid(self, task: Task) -> Optional[Bid]:
        """Prepare a bid for an announced task."""
        if not self.can_handle(task):
            return None

        proficiency = self.skills[task.required_skill]
        effective_cost = task.estimated_effort * self.cost_per_unit / max(proficiency, 0.1)
        estimated_time = task.estimated_effort / max(proficiency, 0.1)

        return Bid(
            bidder_id=self.agent_id,
            task_id=task.task_id,
            offered_price=effective_cost,
            estimated_completion_time=estimated_time,
            skill_match_score=proficiency,
        )

    def execute_task(self, task: Task) -> str:
        """Simulate executing the task."""
        self.current_tasks.append(task)
        # Simulate work
        time.sleep(0.1)
        self.current_tasks.remove(task)
        return f"Task {task.task_id} completed by {self.agent_id}"


class TaskManagerAgent:
    """
    Central coordinator in CNP: announces tasks, collects bids,
    evaluates, and awards contracts.
    """

    def __init__(self, manager_id: str = "Manager-1"):
        self.manager_id = manager_id
        self.contractors: dict[str, ContractorAgent] = {}
        self.active_tasks: dict[str, Task] = {}
        self.awards: dict[str, ContractAward] = {}

    def register_contractor(self, contractor: ContractorAgent):
        """Register a potential contractor agent."""
        self.contractors[contractor.agent_id] = contractor
        print(f"[{self.manager_id}] Registered contractor: {contractor.agent_id}")

    def announce_task(self, task: Task) -> str:
        """Phase 1: Announce a task to all registered contractors."""
        self.active_tasks[task.task_id] = task
        print(f"\n[{self.manager_id}] ANNOUNCING TASK: {task.description}")
        print(f"  Required skill: {task.required_skill}, "
              f"Est. effort: {task.estimated_effort}")
        return task.task_id

    def collect_bids(self, task_id: str) -> list[Bid]:
        """Phase 2: Collect bids from all eligible contractors."""
        task = self.active_tasks.get(task_id)
        if not task:
            raise ValueError(f"Task {task_id} not found")

        bids: list[Bid] = []
        print(f"\n[{self.manager_id}] COLLECTING BIDS for task {task_id}...")

        for cid, contractor in self.contractors.items():
            bid = contractor.prepare_bid(task)
            if bid:
                bids.append(bid)
                print(f"  Bid from {cid}: price={bid.offered_price:.2f}, "
                      f"time={bid.estimated_completion_time:.2f}, "
                      f"skill_match={bid.skill_match_score:.2f}")
            else:
                print(f"  {cid}: cannot bid (busy or unqualified)")

        return bids

    def evaluate_bids(self, bids: list[Bid], weight_price: float = 0.4,
                      weight_time: float = 0.3,
                      weight_skill: float = 0.3) -> Optional[Bid]:
        """Phase 3: Evaluate bids using weighted scoring."""
        if not bids:
            return None

        # Normalize scores
        max_price = max(b.offered_price for b in bids) if bids else 1
        max_time = max(b.estimated_completion_time for b in bids) if bids else 1

        def score(bid: Bid) -> float:
            # Lower price and time are better => invert
            price_score = 1.0 - (bid.offered_price / max_price)
            time_score = 1.0 - (bid.estimated_completion_time / max_time)
            skill_score = bid.skill_match_score
            return (weight_price * price_score +
                    weight_time * time_score +
                    weight_skill * skill_score)

        scored = [(score(b), b) for b in bids]
        scored.sort(key=lambda x: x[0], reverse=True)

        best_score, best_bid = scored[0]
        print(f"\n  Best bid: {best_bid.bidder_id} "
              f"(score={best_score:.4f}, price={best_bid.offered_price:.2f})")
        for s, b in scored:
            print(f"    {b.bidder_id}: score={s:.4f}")

        return best_bid

    def award_contract(self, task: Task, winning_bid: Bid) -> ContractAward:
        """Phase 4: Award the contract to the winning bidder."""
        award = ContractAward(
            task_id=task.task_id,
            awarded_to=winning_bid.bidder_id,
            award_price=winning_bid.offered_price,
        )
        self.awards[task.task_id] = award
        print(f"\n[{self.manager_id}] AWARDING contract: "
              f"task={task.task_id} -> {winning_bid.bidder_id} "
              f"at price={winning_bid.offered_price:.2f}")
        return award

    def run_cnp_cycle(self, task: Task) -> Optional[str]:
        """
        Run a full CNP cycle: announce -> collect bids -> evaluate -> award.
        Returns the winning agent ID, or None if no bids received.
        """
        self.announce_task(task)
        bids = self.collect_bids(task.task_id)

        if not bids:
            print(f"[{self.manager_id}] No bids received for task {task.task_id}. "
                  f"Task remains unassigned.")
            return None

        winner = self.evaluate_bids(
            bids, weight_price=0.4, weight_time=0.3, weight_skill=0.3
        )
        award = self.award_contract(task, winner)

        # The winning contractor executes
        contractor = self.contractors[winner.bidder_id]
        result = contractor.execute_task(task)
        print(f"  Result: {result}")
        return winner.bidder_id


# ========== Usage Example ==========

if __name__ == "__main__":
    # Create a task manager
    manager = TaskManagerAgent("CNP-Manager")

    # Create contractor agents with different skills and costs
    contractors = [
        ContractorAgent("Robot-Alpha", {"navigation": 0.9, "manipulation": 0.7}, 10.0),
        ContractorAgent("Robot-Beta",  {"navigation": 0.6, "manipulation": 0.9}, 8.0),
        ContractorAgent("Robot-Gamma", {"navigation": 0.8, "manipulation": 0.5}, 6.0),
        ContractorAgent("Robot-Delta", {"navigation": 0.9, "manipulation": 0.9}, 15.0),
    ]

    for c in contractors:
        manager.register_contractor(c)

    # Create tasks and run CNP cycles
    tasks = [
        Task("", "Transport cargo to warehouse", 10.0, "navigation", 5.0),
        Task("", "Assemble component", 15.0, "manipulation", 8.0),
        Task("", "Navigate to zone B and pick up item", 8.0, "navigation", 3.0),
    ]

    for task in tasks:
        winner = manager.run_cnp_cycle(task)
        print(f"  Task '{task.description}' -> awarded to {winner}\n")
        print("=" * 60)
```

### 示例 2：基于拍卖的资源分配

```python
"""
Auction-based Resource Allocation among multiple agents.
Implements both English Auction (ascending) and Vickrey Auction (sealed-bid second-price).
"""
from __future__ import annotations
import uuid
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class Resource:
    """A resource to be allocated via auction."""
    resource_id: str
    name: str
    base_value: float  # intrinsic value


@dataclass
class Bid:
    """A bid in an auction."""
    agent_id: str
    resource_id: str
    amount: float
    round_number: int


class Agent:
    """An agent participating in resource auctions."""

    def __init__(self, agent_id: str, budget: float,
                 valuation_fn: callable = None):
        self.agent_id = agent_id
        self.budget = budget
        self.allocated_resources: list[Resource] = []
        self.total_spent = 0.0
        # valuation_fn maps resource -> willingness-to-pay
        self.valuation_fn = valuation_fn or (lambda r: r.base_value * 1.0)

    def value_resource(self, resource: Resource) -> float:
        """How much this agent values a resource."""
        return self.valuation_fn(resource)

    def can_afford(self, amount: float) -> bool:
        return self.total_spent + amount <= self.budget


class EnglishAuction:
    """
    English (ascending-open) Auction.
    Price increases until only one bidder remains.
    """

    def __init__(self, resource: Resource, reserve_price: float = 0.0,
                 bid_increment: float = 1.0):
        self.resource = resource
        self.reserve_price = reserve_price
        self.bid_increment = bid_increment
        self.current_price = reserve_price
        self.current_winner: Optional[str] = None
        self.bids: list[Bid] = []
        self.round = 0
        self.is_active = True

    def run(self, agents: list[Agent]) -> tuple[Optional[Agent], float]:
        """
        Run the English auction.
        Returns (winning_agent, winning_price).
        """
        print(f"\n=== English Auction: {self.resource.name} ===")
        print(f"  Reserve price: {self.reserve_price}")

        active_bidders = set(
            a.agent_id for a in agents
            if a.can_afford(self.reserve_price) and
               a.value_resource(self.resource) >= self.reserve_price
        )

        if len(active_bidders) == 0:
            print("  No bidders. Resource unsold.")
            return None, 0.0

        print(f"  Initial active bidders: {active_bidders}")

        self.current_price = self.reserve_price
        self.current_winner = next(iter(active_bidders))

        while len(active_bidders) > 1 and self.is_active:
            self.round += 1
            self.current_price += self.bid_increment
            print(f"\n  Round {self.round}: price = {self.current_price:.1f}")

            # Bidders drop out if price exceeds valuation
            still_active = set()
            for agent_id in active_bidders:
                agent = next(a for a in agents if a.agent_id == agent_id)
                if agent.value_resource(self.resource) >= self.current_price:
                    still_active.add(agent_id)
                    bid = Bid(agent_id, self.resource.resource_id,
                              self.current_price, self.round)
                    self.bids.append(bid)
                    print(f"    {agent_id} bids {self.current_price:.1f}")
                else:
                    agent_ = next(a for a in agents if a.agent_id == agent_id)
                    max_val = agent_.value_resource(self.resource)
                    print(f"    {agent_id} drops out (valuation={max_val:.1f} < {self.current_price:.1f})")

            active_bidders = still_active
            if active_bidders:
                self.current_winner = next(iter(active_bidders))

        if len(active_bidders) == 1:
            final_price = self.current_price
            winner_id = next(iter(active_bidders))
            winner = next(a for a in agents if a.agent_id == winner_id)
            print(f"\n  WINNER: {winner_id} at price {final_price:.1f}")
            return winner, final_price
        else:
            print(f"\n  No winner (no bidders remained)")
            return None, 0.0


class VickreyAuction:
    """
    Vickrey (sealed-bid second-price) Auction.
    Highest bidder wins but pays the second-highest price.
    Incentive-compatible: truthful bidding is a dominant strategy.
    """

    def __init__(self, resource: Resource, reserve_price: float = 0.0):
        self.resource = resource
        self.reserve_price = reserve_price

    def run(self, agents: list[Agent]) -> tuple[Optional[Agent], float]:
        """
        Run the Vickrey auction.
        Returns (winning_agent, second_price).
        """
        print(f"\n=== Vickrey Auction: {self.resource.name} ===")
        print(f"  Reserve price: {self.reserve_price}")

        # Each agent submits a sealed bid (their true valuation)
        bids: list[Bid] = []
        for agent in agents:
            valuation = agent.value_resource(self.resource)
            if valuation >= self.reserve_price and agent.can_afford(valuation):
                # In Vickrey, rational agents bid their true valuation
                sealed_bid = Bid(
                    agent_id=agent.agent_id,
                    resource_id=self.resource.resource_id,
                    amount=valuation,
                    round_number=1
                )
                bids.append(sealed_bid)
                print(f"  {agent.agent_id} bids {valuation:.1f} "
                      f"(true valuation={valuation:.1f})")

        if len(bids) == 0:
            print("  No valid bids. Resource unsold.")
            return None, 0.0

        # Sort bids descending
        bids.sort(key=lambda b: b.amount, reverse=True)

        highest_bid = bids[0]
        second_price = bids[1].amount if len(bids) > 1 else self.reserve_price

        winner = next(a for a in agents if a.agent_id == highest_bid.agent_id)
        print(f"  WINNER: {winner.agent_id} "
              f"(bid={highest_bid.amount:.1f}, pays={second_price:.1f})")

        return winner, second_price


class AuctionHouse:
    """
    Centralized auction house that conducts auctions over multiple resources.
    """

    def __init__(self):
        self.auctions = []
        self.results: dict[str, tuple[Optional[Agent], float]] = {}

    def run_english_auction(self, resource: Resource,
                            agents: list[Agent],
                            reserve_price: float,
                            bid_increment: float = 1.0) -> tuple[Optional[Agent], float]:
        auction = EnglishAuction(resource, reserve_price, bid_increment)
        winner, price = auction.run(agents)
        self.results[resource.resource_id] = (winner, price)
        return winner, price

    def run_vickrey_auction(self, resource: Resource,
                            agents: list[Agent],
                            reserve_price: float) -> tuple[Optional[Agent], float]:
        auction = VickreyAuction(resource, reserve_price)
        winner, price = auction.run(agents)
        self.results[resource.resource_id] = (winner, price)
        return winner, price


# ========== Usage Example ==========

if __name__ == "__main__":
    house = AuctionHouse()

    # Create resources
    resources = [
        Resource("r1", "GPU-Cluster-A", base_value=100.0),
        Resource("r2", "GPU-Cluster-B", base_value=80.0),
        Resource("r3", "Storage-Node-1", base_value=50.0),
    ]

    # Create agents with different valuations
    agents = [
        Agent(
            "Trainer-AI",
            budget=200.0,
            valuation_fn=lambda r: r.base_value * 1.5 if "GPU" in r.name else r.base_value * 0.5
        ),
        Agent(
            "Inference-Engine",
            budget=150.0,
            valuation_fn=lambda r: r.base_value * 1.2 if "GPU" in r.name else r.base_value * 1.1
        ),
        Agent(
            "Data-Pipeline",
            budget=100.0,
            valuation_fn=lambda r: r.base_value * 1.3 if "Storage" in r.name else r.base_value * 0.3
        ),
    ]

    # Run English auction for GPU-Cluster-A
    winner, price = house.run_english_auction(
        resources[0], agents,
        reserve_price=50.0, bid_increment=5.0
    )
    if winner:
        winner.allocated_resources.append(resources[0])
        winner.total_spent += price
        print(f"  Final allocation: {winner.agent_id} gets {resources[0].name} "
              f"(spent {price:.1f}, remaining budget {winner.budget - winner.total_spent:.1f})")

    # Run Vickrey auction for Storage-Node-1
    winner2, price2 = house.run_vickrey_auction(
        resources[2], agents, reserve_price=20.0
    )
    if winner2:
        winner2.allocated_resources.append(resources[2])
        winner2.total_spent += price2
        print(f"  Final allocation: {winner2.agent_id} gets {resources[2].name} "
              f"(spent {price2:.1f}, remaining budget {winner2.budget - winner2.total_spent:.1f})")

    print("\n=== Final Allocation Summary ===")
    for agent in agents:
        if agent.allocated_resources:
            res_names = [r.name for r in agent.allocated_resources]
            print(f"  {agent.agent_id}: {res_names}, spent={agent.total_spent:.1f}, "
                  f"remaining budget={agent.budget - agent.total_spent:.1f}")
        else:
            print(f"  {agent.agent_id}: no resources allocated")
```

---

## 延伸阅读与资源

- Smith, R. G. (1980). *The Contract Net Protocol: High-Level Communication and Control in a Distributed Problem Solver*. IEEE Transactions on Computers.
- Wellman, M. P. (1993). *A Market-Oriented Programming Environment and its Application to Distributed Multicommodity Flow Problems*. JAIR.
- Vickrey, W. (1961). *Counterspeculation, Auctions, and Competitive Sealed Tenders*. Journal of Finance.
- Shoham, Y. & Leyton-Brown, K. (2008). *Multiagent Systems: Algorithmic, Game-Theoretic, and Logical Foundations*. Cambridge University Press.
- Dempe, S. et al. — DCOP 算法综述（ADOPT / DPOP / Max-Sum）
- Lynch, N. A. (1996). *Distributed Algorithms*. Morgan Kaufmann.
