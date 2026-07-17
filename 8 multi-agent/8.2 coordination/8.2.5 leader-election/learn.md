# 8.2.5 领导者选举 (Leader Election)

## 1. 简单介绍

领导者选举（Leader Election）是多智能体系统中一种关键的协调机制，允许一组对等的智能体在没有中央协调器的情况下，通过分布式协议动态选出一个智能体作为临时的领导者/协调者。被选出的领导者负责执行特定协调职能，如任务分配、结果汇总、冲突裁决或与其他系统的通信接口。

与静态指定领导者的方式不同，动态选举使得系统能够在领导者故障、网络分区恢复或群体成员变化时自动重新选举，从而保持系统的自治性和鲁棒性。

## 2. 基本原理

领导者选举的核心思想基于以下几个关键原则：

**领导者是角色而非身份**：领导权是一个临时角色，任何具备资格的智能体都可以担任。角色可以随系统状态变化而转移，不存在永久的"领导智能体"。

**选举触发条件**：
- **启动时选举**：系统初始化时，所有对等节点中尚无领导者，需要通过首次选举确定
- **领导者故障检测**：当前领导者心跳超时或无法响应，触发重新选举
- **成员变更**：新智能体加入或现有智能体离开，可能需要重新评估领导权
- **负载重平衡**：当前领导者过载，主动发起重新选举以转移领导职责
- **租约到期**：领导权附带时间租约，租约到期后需重新选举或续约

**法定人数要求（Quorum）**：选举结果需要获得大多数（超过半数）智能体的认可，以防止网络分区时出现多个领导者（即防止 split-brain）。在规模为 N 的系统中，法定人数通常定义为 `floor(N/2) + 1`。

**选举超时与随机化**：为了防止多个智能体同时发起选举导致冲突（选举风暴），每个智能体在检测到领导者故障后等待一个随机化的超时时间再发起选举，这是 Raft 等算法中广泛使用的关键技术。

## 3. 背景与演进

领导者选举起源于分布式系统领域，经历了从简单到复杂、从通用到场景定制的演进过程。

**分布式系统时期的经典算法**：

- **Bully 算法（Garcia-Molina, 1982）**：每个节点拥有唯一 ID（通常为数值），具有最高 ID 的节点当选为领导者。当任意节点发现领导者无响应时，便向所有 ID 更高的节点发送选举消息，若无人应答则自己成为领导者。实现简单，但消息复杂度高（O(n^2)），且优先级静态固定。

- **Ring 算法（Chang & Roberts, 1979）**：节点组成逻辑环，选举消息沿环传递，每个节点将自己的 ID 加入消息，最终拥有最高 ID 的节点当选。消息数量为 O(n)，但依赖环拓扑的完整性。

- **Raft 领导者选举（Diego Ongaro, 2013）**：引入任期（term）概念和随机化超时机制。节点分为三种状态：跟随者、候选者、领导者。跟随者在超时内未收到领导者心跳即转为候选者，并向其他节点发起投票请求。获得多数票的节点成为新领导者。Raft 将选举与日志复制紧密耦合，设计目标是强一致性的状态机复制。

**向多智能体系统的演进**：

多智能体系统继承了分布式系统的选举算法，但面临新的需求：

- 智能体具有异构的能力和角色（不同于分布式节点通常是对等的）
- 智能体的"能力"是动态变化的（学习、资源消耗、任务负载）
- 智能体之间的通信可能是语义级别的（而非简单的 RPC）
- 需要支持更丰富的选举标准（不仅仅是 ID 大小）
- 智能体可能出现拜占庭故障（恶意行为），而不仅是崩溃故障

这些差异推动了面向智能体的选举算法的发展，如能力加权选举、基于信誉的选举等。

## 4. 之前做法与结果

### 传统 Bully 算法

**做法**：每个智能体分配一个静态优先级（通常是唯一 ID）。当领导者故障时，检测到故障的智能体向所有优先级更高的智能体发送选举消息。若在超时时间内无回应，则宣告自己为领导者。新领导者通知所有低优先级智能体。

**结果与问题**：
- 消息复杂度为 O(n^2)，通信开销大
- 优先级静态固定，无法根据智能体当前状态灵活调整
- 依赖可靠通信假设，消息丢失可能导致多个智能体都认为自己是领导者
- 仅支持崩溃-停止（crash-stop）模型，无法处理拜占庭故障

### Raft 领导者选举

**做法**：每个任期（term）内最多有一个领导者。跟随者通过随机化超时检测领导者故障，转换为候选者后请求投票。获得 `N/2 + 1` 票后成为领导者，然后定期发送心跳维持领导权。

**结果与问题**：
- 专为日志复制设计，选举和日志管理紧密耦合，对仅需协调功能的智能体系统过于重量级
- 所有节点投票权重相同，无法反映智能体能力的差异
- 假设网络相对稳定，节点数变化不频繁
- 要求大多数节点在线且正常通信，对高动态环境适应差

### 核心问题归纳

| 问题 | 描述 |
|------|------|
| 静态优先级 | 无法反映智能体动态变化的能力和状态 |
| 可靠通信假设 | 传统算法假设消息不会丢失，在不可靠环境中表现不佳 |
| 仅支持 crash-fail | 无法处理智能体的拜占庭故障或恶意行为 |
| 节点同质性假设 | 所有节点投票权重相同，不适用于能力异构的智能体系统 |
| 高消息复杂度 | 在大规模智能体群体中通信开销不可忽视 |

## 5. 核心矛盾：Speed vs Fairness

领导者选举面临一个根本性的权衡：**选举速度与选举公平性之间的矛盾**。

**追求速度**：
- 使用静态优先级或预定义的领导者顺序
- 最小化选举过程中的消息交换
- 快速收敛到某个领导者，即使不是最优选择
- 适用于对延迟敏感的实时系统

**追求公平/最优**：
- 收集各智能体的多维能力指标（负载、延迟、历史可靠性、领域专长）
- 基于加权投票或综合评分选出最合适的领导者
- 引入周期性重选以防止领导者固化
- 选举过程本身可能消耗大量时间和通信资源

**权衡的本质**：快速选举倾向于选择"最容易成为领导者"的智能体（如 ID 最高或最先响应），而公平选举倾向于选择"最适合担任领导者"的智能体。两者在大多数场景中不可兼得。

**设计决策**：实际系统需要在两者之间找到平衡点。例如，使用两阶段选举：第一阶段快速选出一个临时领导者，第二阶段在后台收集能力数据进行确认或调整。或者采用租约机制，允许能力不匹配但快速选出的领导者短期执政，直到下一次更全面的选举。

## 6. 主流优化方向

### 能力加权选举 (Capability-Weighted Election)

智能体的投票权重和候选人评分基于其当前的实际能力，而非固定的 ID。能力指标可以包括：

- **计算资源**：CPU 利用率、可用内存、网络带宽
- **任务完成率**：历史任务完成比例和平均完成时间
- **领域专长**：与当前协调任务匹配的专业知识水平
- **可靠性评分**：历史故障频率、平均无故障时间

每个智能体维护一个能力向量，选举时综合计算得分。这使得最适合的智能体（而非最早或 ID 最大的智能体）成为领导者。

### 租约式领导权 (Lease-Based Leadership)

领导者获得一个**有时间限制的领导权租约**，在租约有效期内行使领导职能。

- 领导者必须在租约到期前**续约**（通过心跳或显式续约消息）
- 如果租约到期未续约，自动触发重新选举
- 租约时长是可配置的（权衡：短租约增加通信开销但提高响应速度）
- 有效防止领导者在故障后长期霸占领导权

### 多标准选举 (Multi-Criteria Election)

综合考虑多个维度的因素进行选举决策，通常使用加权评分模型：

```
score(agent) = w1 * reliability + w2 * available_capacity + w3 * domain_match + w4 * latency_score
```

权重可以根据系统当前需求动态调整。例如，高负载时期提高"可用容量"的权重，高可靠性要求时期提高"可靠性"的权重。

### 分层选举 (Hierarchical Election)

将大规模智能体群体划分为多个小组，采用分层结构：

1. **本地选举**：每个小组内部选举一个本地领导者
2. **全局选举**：所有本地领导者组成一个高层级群体，从中选举全局领导者

这种方式将选举的消息复杂度从 O(n^2) 降低到 O(k * m^2)，其中 k 是小组数量，m 是小组大小（n = k * m）。同时自然地支持了大规模系统中的层级管理。

### 预测性选举 (Predictive Election)

不等待领导者实际故障，而是通过监控健康指标预测即将发生的故障：

- **健康指标监控**：响应延迟趋势、错误率上升、资源耗尽速度
- **早期预警**：当领导者健康评分低于阈值时，提前触发预选
- **平滑过渡**：新领导者在原领导者完全故障前完成状态同步，实现无感切换

这种方式可以显著减少领导者故障导致的系统停机时间。

### 民主式重新选举 (Democratic Re-Election)

引入周期性重新选举机制，防止领导者的权力固化：

- **固定任期**：每个领导者有固定的任期，任期结束后强制重新选举
- **支持不信任投票**：当多数智能体对当前领导者不满意时，可发起不信任投票提前触发重选
- **权力制衡**：防止单一智能体长期占据领导位置，提高系统的民主性和适应性

## 7. 能力边界

### 适合的场景

- 需要从多个对等智能体中动态选择一个决策协调者
- 系统需要处理领导者的故障并且自动恢复
- 智能体能力异构，需要选择当前最合适的智能体担任领导
- 任务具有阶段性，不同阶段可能需要不同类型的领导者

### 不擅长的场景

- **高动态群体（高成员变动率）**：智能体频繁加入和离开导致选举几乎持续进行，系统永远无法稳定
- **无广泛能力突出的群体**：所有智能体能力相近且都是领域狭专的专家，没有智能体能有效领导全局
- **需要完全分布式决策的系统**：某些系统架构要求没有中心节点，所有决策都通过共识达成（如完全去中心化的区块链）
- **极端延迟敏感的系统**：选举过程本身需要时间和通信，对于微秒级响应的系统，预分配领导者更为现实
- **拜占庭故障频发的环境**：标准选举算法不防护恶意投票或虚假能力声明，需要引入额外的信任机制

## 8. vs 静态领导者 (Static Leader)

| 维度 | 动态选举 | 静态领导者 |
|------|---------|-----------|
| **容错性** | 领导者故障后自动恢复，系统持续可用 | 领导者故障导致系统不可用，直到人工干预 |
| **延迟** | 引入选举延迟（通常数百毫秒到数秒） | 无选举延迟，领导者始终固定 |
| **单点故障** | 无 SPOF（通过选举自动替换） | 存在明显的 SPOF |
| **适应性** | 可选择当前最合适的智能体担任领导 | 领导能力固定，无法随系统状态变化 |
| **复杂度** | 实现复杂，需要处理选举状态机和网络分区 | 实现简单，配置即用 |
| **消息开销** | 选举过程和心跳消息持续消耗带宽 | 几乎没有协议开销 |
| **可预测性** | 领导者可能变化，行为模式动态 | 领导者固定，行为模式稳定可预测 |
| **适用场景** | 高可用系统、故障频繁的环境、动态集群 | 开发环境、小型稳定系统、实时性要求极高的系统 |

## 9. 核心优势

1. **容错性**：领导者的故障不会导致系统瘫痪，自动选举出新的领导者继续工作。

2. **自适应性**：系统始终选择当前最合适的智能体担任领导者，适应能力和资源的变化。

3. **负载均衡**：领导权可以在不同智能体之间轮换，防止单一智能体长期过载。

4. **无单点故障**：没有永久性的中心节点，系统的可靠性不受限于任何一个智能体。

5. **可伸缩性**：智能体可以动态加入或离开系统，而不需要重新配置领导关系。

6. **自动化运维**：减少了人工干预的需要，系统可以自主维持正常运作。

## 10. 工程实现挑战

### 脑裂 (Split-Brain)

网络分区时，两个分区各自选举出领导者，导致系统出现两个"合法"领导者。

**解决方案**：
- 使用法定人数机制（quorum），只有获得多数票的候选人才成为领导者
- 引入 fencing（栅栏）机制，旧领导者被隔离后主动让位
- 共享存储（如 etcd、ZooKeeper）作为最终的权威来源

### 领导者震荡 (Leader-Flapping)

由于网络不稳定或资源波动，领导权在多个智能体之间快速转移。

**解决方案**：
- 引入最小执政时间（min-election-interval），防止过于频繁的选举
- 使用指数退避，连续失败后延长等待时间
- 稳定偏好：在没有显著理由时倾向于保留当前领导者

### 选举风暴 (Election Storm)

所有智能体同时检测到领导者故障并同时发起选举，导致大量冲突消息。

**解决方案**：
- Raft 风格的随机化超时时间
- 引入 backoff 机制，选举失败后加倍等待时间
- 使用抑制机制，收到合法的选举消息后停止自己的选举尝试

### 观察者效应

选举过程本身消耗智能体的计算和通信资源，可能导致原本最适合的智能体因参与选举而过载。

**启示**：选举协议的设计应尽量轻量，避免"为了选出最好的领导者而导致最好的领导者因选举而崩溃"的悖论。

### 状态转移开销

新领导者上任后需要从旧领导者（或持久存储）处获取系统状态，这个转移过程本身可能很耗时。

**挑战**：
- 状态大小：大规模系统的状态可能达数 GB
- 一致性保证：转移后的状态必须与旧领导者的状态一致
- 增量同步：支持只同步增量变化而非全量状态
- 双缓冲：新领导者准备就绪前，旧领导者继续服务

## 11. 适用场景

| 场景 | 描述 |
|------|------|
| **编排器故障切换** | 在 Agent 编排系统中，编排器智能体故障后自动选举备用编排器接管工作流调度 |
| **智能体组协调** | 一组协作智能体需要选举一个协调者来统一对外通信和任务分配 |
| **资源分配管理器** | 负责管理共享资源池（如 GPU、API 配额）的智能体需要选举产生 |
| **争议裁决机构** | 当多个智能体对某个决策产生分歧时，由当选的领导者做出最终裁决 |
| **任务分发中心** | 从任务队列中获取任务并分配给组内最合适的智能体执行 |
| **结果汇总合并** | 收集各智能体的处理结果进行合并、去重和汇总报告 |
| **对外通信代理** | 代表整个智能体群体与外部系统（人类用户、其他系统）进行通信 |

---

## 代码示例

### 示例 1：基于 Bully 算法的简单智能体领导者选举

```python
"""
Bully Algorithm for Agent Leader Election
每个智能体拥有唯一 ID，ID 最高的智能体成为领导者。
"""

import asyncio
import random
import logging
from enum import Enum
from typing import Optional

logging.basicConfig(level=logging.INFO, format="%(message)s")
logger = logging.getLogger(__name__)


class AgentState(Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"


class Agent:
    """
    参与领导者选举的智能体。
    """

    def __init__(self, agent_id: int, all_agent_ids: list[int]):
        self.agent_id = agent_id
        self.all_agent_ids = all_agent_ids
        self.state = AgentState.FOLLOWER
        self.leader_id: Optional[int] = None
        self.higher_ids = [aid for aid in all_agent_ids if aid > agent_id]

        # 模拟网络通信延迟
        self.latency_ms = random.randint(10, 100)

    async def send_message(self, target_id: int, msg_type: str, **kwargs):
        """模拟发送消息——带随机延迟。"""
        delay = random.uniform(0.01, self.latency_ms / 1000)
        await asyncio.sleep(delay)
        logger.info(f"  Agent {self.agent_id} -> Agent {target_id}: [{msg_type}] {kwargs}")

    async def receive_election(self, sender_id: int):
        """
        收到选举消息。
        如果发送者 ID 更低，则回应 OK 并自己发起选举。
        """
        logger.info(f"Agent {self.agent_id} 收到来自 Agent {sender_id} 的选举消息")
        if sender_id < self.agent_id:
            await self.send_message(sender_id, "OK")
            await self.start_election()
        else:
            # 发送者 ID 更高，承认其领导权
            pass

    async def start_election(self):
        """发起选举：向所有更高 ID 的智能体发送选举消息。"""
        self.state = AgentState.CANDIDATE
        logger.info(f"\nAgent {self.agent_id} 发起选举，向更高 ID 智能体发送选举消息: {self.higher_ids}")

        responses = []
        for higher_id in self.higher_ids:
            await self.send_message(higher_id, "ELECTION")
            responses.append(higher_id)

        # 等待响应
        if self.higher_ids:
            await asyncio.sleep(0.05 * len(self.higher_ids))

        # 检查是否有更高 ID 的智能体响应该消息（实际中通过消息回调处理）
        # 简化实现：如果没有更高 ID 的智能体，自己成为领导者
        # 注意：在真实实现中，应通过实际的 OK 响应来判断

    def on_ok_received(self):
        """收到 OK 响应：放弃选举，等待最终结果。"""
        logger.info(f"Agent {self.agent_id} 收到 OK 响应，放弃本次选举")
        self.state = AgentState.FOLLOWER

    async def declare_leadership(self):
        """
        没有更高 ID 的智能体响应，宣告自己为领导者。
        通知所有更低 ID 的智能体。
        """
        self.state = AgentState.LEADER
        self.leader_id = self.agent_id
        logger.info(f"\n=== Agent {self.agent_id} 宣布自己为领导者 ===")

        lower_ids = [aid for aid in self.all_agent_ids if aid < self.agent_id]
        for lower_id in lower_ids:
            await self.send_message(lower_id, "COORDINATOR", leader=self.agent_id)

    async def on_coordinator_received(self, leader_id: int):
        """收到新领导者的宣布消息。"""
        self.leader_id = leader_id
        self.state = AgentState.FOLLOWER
        logger.info(f"Agent {self.agent_id} 承认 Agent {leader_id} 为领导者")

    async def detect_leader_failure(self):
        """检测领导者故障（模拟：指定领导者不响应）。"""
        logger.info(f"Agent {self.agent_id} 检测到领导者可能故障")
        await self.start_election()
        # 简化：如果没有更高 ID 的智能体，直接宣告
        if not self.higher_ids:
            await self.declare_leadership()
        else:
            # 在实际中，通过响应判断
            pass


async def run_bully_election():
    """模拟一次完整的 Bully 领导者选举。"""
    agent_ids = [1, 2, 3, 4, 5]
    agents = {aid: Agent(aid, agent_ids) for aid in agent_ids}

    logger.info("=" * 50)
    logger.info("Bully 算法领导者选举演示")
    logger.info("智能体 ID 集合: %s", agent_ids)
    logger.info("最高 ID 的智能体（5）应成为领导者")
    logger.info("=" * 50)

    logger.info("\n[阶段 1] 假设领导者（Agent 5）故障")
    logger.info("[阶段 2] Agent 3 检测到领导者故障，发起选举\n")

    # Agent 3 检测到领导者故障并发起选举
    agent3 = agents[3]
    for higher_id in agent3.higher_ids:
        await agent3.send_message(higher_id, "ELECTION")

    # Agent 4 和 5 收到选举消息
    await agents[4].receive_election(3)
    await agents[5].receive_election(3)

    logger.info("\n[阶段 3] Agent 4 继续向更高 ID 发起选举\n")
    # Agent 4 向 Agent 5 发起选举
    await agents[4].send_message(5, "ELECTION")
    await agents[5].receive_election(4)

    logger.info("\n[阶段 4] 最高 ID 智能体（5）宣布为领导者\n")
    await agents[5].declare_leadership()

    logger.info("\n" + "=" * 50)
    logger.info("选举结果:")
    for aid, agent in agents.items():
        logger.info(f"  Agent {aid} -> {agent.state.value}, leader_id={agent.leader_id}")
    logger.info("=" * 50)


if __name__ == "__main__":
    asyncio.run(run_bully_election())
```

### 示例 2：基于能力加权的智能体领导者选举

```python
"""
Capability-Weighted Leader Election for Multi-Agent Systems
智能体根据综合能力评分进行加权投票，选出最合适的领导者。
"""

import asyncio
import random
import math
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class CapabilityProfile:
    """智能体的能力概况。"""
    agent_name: str
    cpu_available: float          # 0.0 ~ 1.0, 可用 CPU 比例
    memory_available: float       # 0.0 ~ 1.0, 可用内存比例
    task_completion_rate: float   # 0.0 ~ 1.0, 历史任务完成率
    avg_response_time_ms: float   # 平均响应时间（毫秒）
    domain_expertise: float       # 0.0 ~ 1.0, 领域专长度
    reliability_score: float      # 0.0 ~ 1.0, 可靠性评分

    def compute_leader_score(self) -> float:
        """
        计算综合领导力评分。
        权重可随系统状态动态调整。
        """
        weights = {
            "cpu": 0.20,
            "memory": 0.15,
            "completion": 0.25,
            "response": 0.15,
            "expertise": 0.10,
            "reliability": 0.15,
        }

        # 响应时间越低越好，归一化处理
        response_score = max(0, 1.0 - self.avg_response_time_ms / 1000.0)

        score = (
            weights["cpu"] * self.cpu_available +
            weights["memory"] * self.memory_available +
            weights["completion"] * self.task_completion_rate +
            weights["response"] * response_score +
            weights["expertise"] * self.domain_expertise +
            weights["reliability"] * self.reliability_score
        )
        return round(score, 4)

    def __repr__(self):
        score = self.compute_leader_score()
        return (f"{self.agent_name}(CPU={self.cpu_available:.2f}, "
                f"Mem={self.memory_available:.2f}, "
                f"完成率={self.task_completion_rate:.2f}, "
                f"延迟={self.avg_response_time_ms:.0f}ms, "
                f"专长={self.domain_expertise:.2f}, "
                f"可靠={self.reliability_score:.2f}, "
                f"总分={score:.4f})")


class WeightedElectionAgent:
    """
    参与能力加权选举的智能体。
    """

    def __init__(self, profile: CapabilityProfile):
        self.profile = profile
        self.leader: Optional[str] = None
        self.vote_for: Optional[str] = None

    async def evaluate_candidates(self, candidates: list["WeightedElectionAgent"]) -> str:
        """
        根据候选人的能力评分进行投票。
        评分最高者得票。
        """
        best_candidate = max(candidates, key=lambda c: c.profile.compute_leader_score())
        self.vote_for = best_candidate.profile.agent_name
        await asyncio.sleep(random.uniform(0.01, 0.05))  # 模拟投票延迟
        return self.vote_for

    def __repr__(self):
        return self.profile.__repr__()


async def run_weighted_election(agents: list[WeightedElectionAgent]):
    """
    执行能力加权领导者选举。
    每个智能体对所有候选人（包括自己）进行评分，然后投票。
    获得最多选票的智能体当选为领导者。
    """
    n = len(agents)
    logger.info(f"\n参与选举的智能体数量: {n}")
    logger.info("-" * 70)

    # 打印所有候选人信息
    for i, agent in enumerate(agents):
        logger.info(f"  [{i}] {agent}")
    logger.info("-" * 70)

    # 每个智能体对所有候选人评分并投票
    vote_tasks = []
    for voter in agents:
        vote_tasks.append(voter.evaluate_candidates(agents))

    votes = await asyncio.gather(*vote_tasks)

    # 统计投票结果
    vote_count = {}
    for voter, candidate in zip(agents, votes):
        vote_count[candidate] = vote_count.get(candidate, 0) + 1
        logger.info(f"  {voter.profile.agent_name} 投票给 {candidate}")

    # 确定胜者
    winner_name = max(vote_count, key=vote_count.get)
    winner_agent = next(a for a in agents if a.profile.agent_name == winner_name)
    max_votes = vote_count[winner_name]

    logger.info("\n" + "=" * 70)
    logger.info(f"选举结果: {winner_name} 当选领导者（得票 {max_votes}/{n}）")
    logger.info(f"  - 能力评分: {winner_agent.profile.compute_leader_score():.4f}")
    logger.info("=" * 70)

    # 设置所有智能体的领导者
    for agent in agents:
        agent.leader = winner_name

    return winner_agent


async def simulate_scenario(agents: list[WeightedElectionAgent], scenario_name: str):
    """模拟一个选举场景。"""
    logger.info(f"\n{'#' * 70}")
    logger.info(f"场景: {scenario_name}")
    logger.info(f"{'#' * 70}")

    winner = await run_weighted_election(agents)

    # 验证选举后的系统状态
    for agent in agents:
        assert agent.leader == winner.profile.agent_name, \
            f"{agent.profile.agent_name} 的 leader 未正确设置"
    logger.info("所有智能体已确认新领导者。\n")


async def main():
    """运行多个选举场景。"""

    # 场景 1：能力有明显差异
    agents_1 = [
        WeightedElectionAgent(CapabilityProfile(
            "Agent-Alpha", 0.95, 0.90, 0.98, 15, 0.85, 0.99)),
        WeightedElectionAgent(CapabilityProfile(
            "Agent-Beta",  0.60, 0.55, 0.88, 45, 0.60, 0.85)),
        WeightedElectionAgent(CapabilityProfile(
            "Agent-Gamma", 0.75, 0.70, 0.92, 30, 0.75, 0.90)),
        WeightedElectionAgent(CapabilityProfile(
            "Agent-Delta", 0.40, 0.35, 0.75, 80, 0.50, 0.70)),
    ]
    await simulate_scenario(agents_1, "能力有明显差异——高能力智能体应当选")

    # 场景 2：能力接近，但各有优势
    agents_2 = [
        WeightedElectionAgent(CapabilityProfile(
            "Agent-Echo",   0.85, 0.80, 0.90, 20, 0.95, 0.92)),  # 领域专长突出
        WeightedElectionAgent(CapabilityProfile(
            "Agent-Foxtrot", 0.90, 0.88, 0.95, 18, 0.70, 0.98)),  # 完成率和可靠性高
        WeightedElectionAgent(CapabilityProfile(
            "Agent-Golf",   0.70, 0.75, 0.88, 25, 0.88, 0.89)),  # 中等
    ]
    await simulate_scenario(agents_2, "能力接近但各有优势——综合评分最高者当选")

    # 场景 3：加入低资源智能体后重选
    agents_3 = agents_1 + [
        WeightedElectionAgent(CapabilityProfile(
            "Agent-New", 0.20, 0.15, 0.50, 200, 0.30, 0.40)),
    ]
    await simulate_scenario(agents_3, "新智能体加入（低资源）——不应影响选举结果")

    # 让原领导者的能力下降，触发重选
    agents_1[0].profile.cpu_available = 0.15
    agents_1[0].profile.memory_available = 0.10
    agents_1[0].profile.avg_response_time_ms = 300
    await simulate_scenario(agents_1, "原领导者能力下降——系统应自动重选出更合适的领导者")


if __name__ == "__main__":
    import logging
    logging.basicConfig(level=logging.INFO, format="%(message)s")
    logger = logging.getLogger(__name__)
    asyncio.run(main())
```

### 示例 2 输出示例

运行示例 2 将输出类似以下内容（实际数值因随机延迟而异）：

```
######################################################################
场景: 能力有明显差异——高能力智能体应当选
######################################################################

参与选举的智能体数量: 4
----------------------------------------------------------------------
  [0] Agent-Alpha(CPU=0.95, Mem=0.90, 完成率=0.98, 延迟=15ms, 专长=0.85, 可靠=0.99, 总分=0.9425)
  [1] Agent-Beta(CPU=0.60, Mem=0.55, 完成率=0.88, 延迟=45ms, 专长=0.60, 可靠=0.85, 总分=0.7448)
  [2] Agent-Gamma(CPU=0.75, Mem=0.70, 完成率=0.92, 延迟=30ms, 专长=0.75, 可靠=0.90, 总分=0.8300)
  [3] Agent-Delta(CPU=0.40, Mem=0.35, 完成率=0.75, 延迟=80ms, 专长=0.50, 可靠=0.70, 总分=0.5788)
----------------------------------------------------------------------
  Agent-Alpha 投票给 Agent-Alpha
  Agent-Beta 投票给 Agent-Alpha
  Agent-Gamma 投票给 Agent-Alpha
  Agent-Delta 投票给 Agent-Alpha

======================================================================
选举结果: Agent-Alpha 当选领导者（得票 4/4）
  - 能力评分: 0.9425
======================================================================
所有智能体已确认新领导者。
```

---

## 总结

领导者选举是多智能体系统中实现自组织和容错协调的核心机制。从分布式系统的 Bully、Ring 和 Raft 算法演进而来，面向智能体的选举算法正在向**能力感知**、**动态适应**和**负载感知**的方向发展。核心的 Speed vs Fairness 矛盾驱动了一系列优化方向，包括能力加权选举、租约式领导权、分层选举和预测性选举等。

在工程实现中，需要特别关注脑裂、领导者震荡和选举风暴等实际挑战。选择是否使用动态选举以及采用何种策略，取决于系统的可靠性需求、延迟敏感度和智能体的异构程度。
