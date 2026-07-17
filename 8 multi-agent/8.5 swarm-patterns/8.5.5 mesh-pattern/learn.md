# Mesh Pattern — 网状模式：完全去中心化的多智能体协作

## 简单介绍

**网状模式（Mesh Pattern）** 是多智能体系统中最彻底的去中心化协作模式。在该模式中，所有智能体（Agent）彼此对等（Peer），没有任何中心协调节点或层次结构，每个智能体都可以直接与系统中的任何其他智能体通信。这种模式的设计灵感来源于 **P2P 网络（Peer-to-Peer Networking）**、**无线网状网络（Mesh Networking）** 和 **去中心化分布式系统** 中的成熟实践。

在 Mesh Pattern 下，系统是**全连接、去中心化、自组织**的：智能体之间通过直接消息传递进行通信、协商和协作，每个智能体既是服务的消费者也是服务的提供者。这种模式天然支持**涌现行为（Emergent Behavior）**——复杂的全局协作模式从局部交互中自然产生，无需任何外部控制。

核心思想可以用一句话概括：**让所有智能体能直接对话，让协作从对话中自然涌现。**

## 基本原理

Mesh Pattern 的工作原理建立在三个核心机制之上：**对等连接（Peer Connectivity）**、**发现与注册（Discovery & Registration）**、以及**直接消息传递（Direct Message Passing）**。

### 全连接网状架构

在理想的 Mesh 中，N 个智能体之间形成 N(N-1)/2 条通信链路：

```
                    ┌─────────────────────────────────────┐
                    │   ┌───────────┐                     │
                    │   │ Agent A   │                     │
                    │   └───────────┘                     │
                    │      │  │  │  │                     │
          ┌─────────┼──────┘  │  │  └─────────┐           │
          │         │   ┌─────┘  │           │           │
          │         │   │        │           │           │
          ▼         │   ▼        ▼           ▼           │
    ┌───────────┐   │ ┌───────────┐     ┌───────────┐    │
    │ Agent B   │◄──┼─┤ Agent C   │     │ Agent D   │    │
    └───────────┘   │ └───────────┘     └───────────┘    │
          │         │      │  │               │          │
          │         │   ┌──┘  └──┐            │          │
          │         │   │       │            │          │
          ▼         │   ▼       ▼            ▼          │
    ┌───────────┐   │ ┌───────────────────────────┐      │
    │ Agent E   │   │ │        Agent F            │      │
    └───────────┘   │ └───────────────────────────┘      │
                    └─────────────────────────────────────┘
                          Fully Connected Mesh (6 Agents)
                          每条线 = 直接通信通道
                          总连接数 = N(N-1)/2 = 15
```

### 工作循环

每个 Mesh Agent 的核心工作循环：

```
  ┌─────────────────────────────────────────────────────────┐
  │                    Mesh Agent Lifecycle                  │
  │                                                         │
  │   ┌──────────┐    ┌──────────┐    ┌────────────────┐   │
  │   │ Peer     │───→│ Heart-   │───→│ Message        │   │
  │   │ Discovery│    │ beat     │    │ Routing        │   │
  │   └──────────┘    └──────────┘    └────────────────┘   │
  │        │               │                   │           │
  │        ▼               ▼                   ▼           │
  │   ┌──────────┐    ┌──────────┐    ┌────────────────┐   │
  │   │ Register │    │ Health   │    │ Process &      │   │
  │   │ & Advert │    │ Check    │    │ Forward        │   │
  │   └──────────┘    └──────────┘    └────────────────┘   │
  │        │               │                   │           │
  │        ▼               ▼                   ▼           │
  │   ┌──────────┐    ┌──────────┐    ┌────────────────┐   │
  │   │ Neighbor │    │ Failure  │    │ State          │   │
  │   │ Table    │    │ Recovery │    │ Sync           │   │
  │   └──────────┘    └──────────┘    └────────────────┘   │
  └─────────────────────────────────────────────────────────┘
```

### 三种核心通信模式

1. **点对点直接通信（Unicast）**：已知目标 Agent ID，直接发送消息
2. **广播（Broadcast）**：消息发送给所有已知在线 Peer
3. **泛洪传播（Flooding）**：每个收到消息的 Peer 转发给所有邻居（带 TTL 控制）

## 背景与发展

### 起源脉络

网状模式的工程思想路线可以追溯到以下关键技术的发展脉络：

| 时间 | 技术 | 核心贡献 |
|------|------|---------|
| 1970s | TCP/IP 协议族 | 去中心化分组交换网络的基本通信模型 |
| 1980s | 分布式系统理论 | CAP 定理、逻辑时钟、分布式共识基础 |
| 1999 | Napster | 第一个大规模 P2P 文件共享——中央索引 + P2P 传输（混合式） |
| 2000 | Gnutella | 纯 P2P 协议，完全去中心化，基于泛洪的搜索 |
| 2001 | Kademlia DHT | 分布式哈希表，结构化 P2P 覆盖网络 |
| 2002 | BitTorrent | P2P 下载中的分块并行传输与激励机制 |
| 2000s | Gossip 协议 | 流行病传播模型应用于信息分发和故障检测 |
| 2010s | 区块链 | 完全去中心化共识的工程实践 |
| 2020s | Swarm Robotics | 多机器人系统的自组织协作 |
| 2023+ | Multi-Agent AI | LLM 驱动的多智能体系统中引入 Mesh 架构 |

### 在多智能体系统中的应用演进

1. **早期多 Agent 系统（1990s-2000s）**：以 FIPA 标准为核心，基于 Agent 通信语言（ACL）和中间件（如 JADE）构建对等 Agent 平台
2. **集中式多 Agent 框架（2023-2024）**：AutoGen、CrewAI 等框架以 Manager 编排 Worker 为主
3. **去中心化多 Agent 探索（2024-至今）**：研究者开始在 LLM Agent 领域中实验 Mesh 模式，用于辩论、协作推理、分布式问题求解

## 之前做法与局限

### 做法 1：中央编排器（Central Orchestrator）

系统由一个中央控制器统一调度所有 Agent 的行动和通信。

```
           ┌───────────────────┐
           │    Orchestrator   │
           └───────────────────┘
             ↕   ↕   ↕   ↕   ↕
           ┌───────────────────┐
           │ A  B  C  D  E  F  │
           └───────────────────┘
```

**局限**：
- **单点故障（SPOF）**：编排器一旦崩溃，整个系统瘫痪
- **通信瓶颈**：所有信息流经中央节点，成为吞吐量上限
- **僵化的通信路径**：Agent 不能直接对话，必须经过编排器中转
- **扩展性差**：Agent 数量增加时，编排器负载线性增长
- **不自然的交互模式**：Agent A 给 Agent B 的消息需要编排器转发

### 做法 2：层次式编排（Hierarchical Architecture）

```
            ┌──────────┐
            │  Top-Tier │
            └──────────┘
              ↕      ↕
          ┌──────┐  ┌──────┐
          │ Mid1 │  │ Mid2 │
          └──────┘  └──────┘
           ↕  ↕       ↕  ↕
          A  B       C  D
```

**局限**：
- **通信效率衰减**：底层 Agent 需要经过多层转发才能通信
- **层次间的信息失真**：信息每经过一层过滤/转发可能丢失细节
- **层次绑定限制**：Agent 的通信能力受限于它在层级中的位置
- **重新配置成本高**：添加/移除 Agent 需要更新层次结构

### 做法 3：黑板模式（Blackboard Pattern）

Agent 通过共享的"黑板"（共享内存/数据库）间接通信。

**局限**：
- **共享状态冲突**：多个 Agent 同时写入产生竞争
- **无直接交互**：Agent 无法针对特定合作者发送定向消息
- **中央存储瓶颈**：所有信息必须写入/读取中央存储

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **通信完备性 vs 通信开销** | 全连接网状结构保证了任意 Agent 间的最短通信路径，但 N 个节点产生 O(N^2) 条连接，网络开销呈平方级增长 |
| **信息传播速度 vs 网络拥塞** | Gossip/泛洪能快速传播信息到全网，但冗余消息风暴可能淹没网络带宽和 Agent 处理能力 |
| **去中心化韧性 vs 共识效率** | 无中心节点避免了单点故障，但在需要全局一致决策时，缺乏中心的系统达成共识的代价远超集中式方案 |
| **动态自组织 vs 拓扑不稳定** | Agent 可以随时加入/离开网络，但频繁的拓扑变更导致路由表不断失效，消息丢失和重传开销增加 |
| **隐私自治 vs 可见性** | 对等架构中每个 Agent 掌握自己的数据，但分布式协调需要一定程度的信息共享，两者存在张力 |

**核心矛盾公式化**：

```
全连接网状结构的通信复杂度：C = O(N²)  [连接数]
全连接网状结构的信息扩散步数：S = O(1)  [任意两点一跳直达]
```

这意味着：你从 Mesh 中获得的速度优势（任意两点一步直达），是以平方级增长的连接维护成本为代价的。

## 主流优化方向

### 1. 部分网状连接（Partial Mesh）

放弃"所有点全连接"，转而建立战略性连接，形成稀疏但高效的拓扑：

```
                    ┌───────────┐
                    │ Agent A   │
                    └───────────┘
                       │     │
                       │     │
    ┌───────────┐     │     │     ┌───────────┐
    │ Agent B   │─────┘     └─────│ Agent C   │
    └───────────┘                 └───────────┘
         │                             │
         │                             │
         │         ┌───────────┐       │
         │         │ Agent D   │       │
         │         └───────────┘       │
         │              │              │
         └──────────────┴──────────────┘

                    ┌───────────┐
                    │ Agent E   │
                    └───────────┘
           Partial Mesh (Strategic Connections)
           连接数 = 6 (vs 全连接 10)
           但任意两点仍然可达（多跳）
```

**策略**：基于业务相关性、地理/逻辑邻近度、通信频率等指标建立连接，不常用的路径通过多跳路由完成。

### 2. Gossip 协议传播信息

模仿流行病传播模型，每个 Agent 随机选择 k 个邻居转发信息：

```
  时间 T=0            T=1              T=2               T=3
   ┌─A┐             ┌─A┐             ┌─A┐              ┌─A┐
   │█ │             │█ │             │█ │              │█ │
   └─┘              └─┘              └─┘               └─┘
    │                │↘               │↘ ↘              │↘ ↘
   ┌─B┐             ┌─B┐             ┌─B┐              ┌─B┐
   │  │             │█ │             │█ │              │█ │
   └─┘              └─┘              └─┘               └─┘
    │                │               │  ↘              │  ↘
   ┌─C┐             ┌─C┐             ┌─C┐              ┌─C┐
   │  │             │  │             │█ │              │█ │
   └─┘              └─┘              └─┘               └─┘
                      ↘                               │↓
                     ┌─D┐             ┌─D┐             ┌─D┐
                     │  │             │  │             │█ │
                     └─┘              └─┘              └─┘

  █ = 已收到信息的 Agent
```

- **Push Gossip**：收到信息的节点主动推送给 k 个随机邻居
- **Pull Gossip**：节点定期轮询随机邻居获取新信息
- **混合模式**：Push + Pull 结合

**收敛保证**：在 O(log N) 轮内，信息以高概率传播到所有节点。

### 3. 小世界网络拓扑（Small-World Network）

结合**高聚类系数**和**短平均路径长度**：

```
              紧密社区 1              紧密社区 2
          ┌─────────────────┐   ┌─────────────────┐
          │ A ─ B ─ C       │   │ G ─ H ─ I       │
          │ │   │   │       │   │ │   │   │       │
          │ D ─ E ─ F       │   │ J ─ K ─ L       │
          └─────────────────┘   └─────────────────┘
                 │   ↑               ↓   │
                 │   └── 长程连接 ─────┘   │
                 │         (shortcut)      │
                 └──────────────────────────┘
                               │
                     ┌─────────────────┐
                     │ M ─ N ─ O       │
                     │ │   │   │       │
                     │ P ─ Q ─ R       │
                     └─────────────────┘
                       紧密社区 3
```

**特性**：大多数连接在本地社区内（保证局部通信效率），少量长程连接跨越社区（保证全局可达性）。

### 4. 发布-订阅覆盖（Pub-Sub Overlay）

引入主题（Topic）作为逻辑通信通道：

```
  Agent A (天气预报)            Agent E (天气预报订阅者)
        │                              ↑
        │ 发布: topic="weather"         │ 订阅: topic="weather"
        │                              │
        └──────────→ [Message Bus] ←───┘
                     /        \
              发布: topic=    发布: topic=
              "stock"         "news"
                    ↓              ↓
              Agent B (股票)    Agent C (新闻订阅者)
              Agent D (股票)    Agent F (新闻订阅者)
```

- 发布者发送到 Topic，不关心谁在收听
- 订阅者接收 Topic，不关心谁在发布
- 网络负责 Topic 路由和消息投递

### 5. 分级网状（Super-Peer Hierarchy）

将 Agent 分为普通 Peer 和超级 Peer，超级 Peer 之间构成 Mesh，各自管理一组普通 Peer：

```
              ┌─────────── Mesh Backbone ───────────┐
              │                                      │
        ┌─────Super-Peer A─────┐         ┌─────Super-Peer B─────┐
        │                      │         │                      │
      ┌─┴─┐ ┌─┴─┐ ┌─┴─┐    ┌─┴─┐     ┌─┴─┐ ┌─┴─┐ ┌─┴─┐    ┌─┴─┐
      │ P1│ │ P2│ │ P3│ ...│ Pk│     │ Q1│ │ Q2│ │ Q3│ ...│ Qm│
      └───┘ └───┘ └───┘    └───┘     └───┘ └───┘ └───┘    └───┘
              Cluster A                   Cluster B
```

- 超级 Peer 之间保持全连接或半连接 Mesh
- 普通 Peer 只连接自己的超级 Peer
- 跨集群通信通过超级 Peer 转发

## 实现挑战

### 1. Agent 发现与注册（Discovery & Registration）

在去中心化系统中，新 Agent 如何发现已有网络并注册？

**挑战**：
- 无中央注册中心，新节点需要找到至少一个"入口点"
- Agent 动态加入/离开，节点列表持续变化
- 网络分区后重新合并时，节点列表需要合并去重

**常用方案**：
- **种子节点列表（Seed Nodes）**：预配置一组稳定地址作为启动入口
- **DHT 索引**：基于分布式哈希表维护节点信息（如 Kademlia）
- **组播发现（Multicast Discovery）**：本地网络通过 UDP 组播宣告存在
- **引荐机制（Referral）**：A 认识 B，B 引荐给 C

### 2. 动态拓扑中的消息路由

Agent 加入/离开频繁，路由表需要持续更新。

**挑战**：
- 发送消息时，目标 Agent 可能已离线
- 多跳路由可能因为中间节点离开而中断
- 路由收敛延迟导致消息暂时丢失或重复

| 路由策略 | 优点 | 缺点 |
|---------|------|------|
| 直接连接 | 一跳直达，延迟最低 | 连接数 O(N²)，不扩展 |
| 泛洪（Flooding） | 只要存在路径就能送达 | 消息风暴，带宽浪费 |
| 随机转发（Random Walk） | 低开销，实现简单 | 送达无保证，延迟不可控 |
| 结构化路由（DHT） | 确定性路由 O(log N) | 需要维护 DHT 结构，实现复杂 |
| 基于内容的路由 | 语义精确匹配 | 需要内容索引，计算开销大 |

### 3. 网络分区检测与恢复

当网络因故障被切割成多个互不相连的片段时，系统如何感知和恢复？

```
  初始拓扑:               网络分区后:
  A ─ B ─ C              A ─ B ─ C
  │   │   │              │   │
  D ─ E ─ F              D ─ E     F (孤立)
  │   │   │              
  G ─ H ─ I              G ─ H ─ I

                     中间链路 C-F, E-F 中断
                     集群 1: {A,B,C,D,E,G,H,I}
                     集群 2: {F}
```

**检测机制**：
- **心跳超时（Heartbeat Timeout）**：如果连续 N 次未收到某邻居的心跳，标记为可疑
- **间接健康检查**：Agent A 向 Agent B 询问是否仍能联系到 Agent C
- **集合交换（Membership Exchange）**：定期交换可到达的节点集合，发现不一致性

**恢复策略**：
- **定期尝试重连**：被标记为离线的节点，周期性尝试重新连接
- **备份路径缓存**：保存曾经可达的替代路径信息
- **合并检测**：分区恢复后检测到分裂集合的合并

### 4. 信息洪泛抑制

Gossip 或 Flooding 可能导致消息无限扩散。

**控制手段**：

```
  消息结构:
  ┌──────────────────────────────────────┐
  │ Message ID: UUID   (全局唯一)         │
  │ TTL: 3             (剩余跳数)         │
  │ Origin: Agent_A    (源节点)           │
  │ Content: ...                         │
  │ Sequence: 42       (发送方序列号)      │
  └──────────────────────────────────────┘

  去重机制: 每个 Agent 维护已见消息的 Bloom Filter
  扩散控制: TTL 每转发一次减 1，TTL=0 停止转发
  扇出限制: 每个节点只转发给最多 k 个随机邻居
```

### 5. 跨 Agent 状态一致性

多个对等 Agent 维护共享状态时的一致性挑战。

- **最终一致性（Eventual Consistency）**：通过 Gossip 传播状态变更，最终所有节点达成一致
- **向量时钟（Vector Clock）**：检测和解决并发更新冲突
- **CRDT（Conflict-free Replicated Data Types）**：数据类型本身保证无冲突合并
- **分布式锁**：但在去中心化环境中实现分布式锁本身就需要共识算法（如 Paxos/Raft），而共识算法通常依赖 Leader 选举，这与 Mesh 的去中心化理念存在内在矛盾

## 能力与边界

### 能力

| 能力 | 说明 |
|------|------|
| **去中心化决策** | 多个对等 Agent 可以独立推理并交换意见，通过辩论、投票形成决策，无需中央协调 |
| **容错性** | 任意 Agent 故障不影响整体系统功能，其他 Agent 可接管任务或绕过故障节点 |
| **动态环境适应** | 新 Agent 可随时加入增强系统能力，离线 Agent 自动被绕过——拓扑自适应 |
| **自然涌现行为** | 没有全局控制，但局部交互可以涌现出复杂的全局协作模式 |
| **低延迟局部通信** | 经常协作的 Agent 间建立直接通道，通信一步直达 |
| **隐私保护** | 信息仅在与相关的节点间传递，无需全部透露给中央协调器 |

### 边界

| 边界 | 说明 |
|------|------|
| **大规模扩展受限** | Agent 数量超过 50-100 后，全连接 Mesh 的 O(N^2) 连接维护成本变得不可承受 |
| **全局协调困难** | 需要全局最优解或全局一致决策的场景（如全局资源分配）在 Mesh 中实现代价极高 |
| **调试与可观测性差** | 信息流散布在所有节点间，没有中心日志，追踪一条消息的完整路径非常困难 |
| **消息交付无强保证** | 在动态拓扑中，无法提供像队列系统那样的"精确一次交付"保证 |
| **计算资源消耗高** | 每个 Agent 需要维护邻居表、处理心跳、运行发现协议等，额外开销不容忽视 |
| **安全威胁面大** | 无中心意味着无统一的身份认证和访问控制，恶意 Agent 更容易混入系统 |

**不适合的场景**：
- 需要严格全局一致性的系统（如金融结算）
- Agent 数量极大（1000+）的大规模系统
- 对消息投递有"精确一次"保证要求的系统
- 安全敏感且需要统一访问控制的系统

## 与其他模式对比

| 维度 | Mesh Pattern | Manager-Worker | Market Pattern | Pipeline Pattern |
|------|-------------|---------------|---------------|-----------------|
| **控制方式** | 完全去中心化 | 集中控制 | 市场驱动 | 流程驱动 |
| **通信拓扑** | 全连接/部分连接 | 星形（Manager 为中心） | 点对点协商 | 线性链 |
| **协调机制** | 直接对话 | 任务分配 | 价格/竞标 | 上下游传递 |
| **故障影响** | 仅故障节点 | Manager 故障 = 全局瘫痪 | 市场出清失败 | 单点中断 = 全线停摆 |
| **扩展性** | 差（O(N²) 连接） | 中（Manager 瓶颈） | 好（市场天然可扩展） | 好（新节点串联即可） |
| **通信延迟** | 最快（一步直达） | 中（经过 Manager） | 慢（招标/竞标/签约） | 快（但受限于最慢环节） |
| **涌现能力** | 最强 | 弱（受 Manager 控制） | 中（价格信号传导） | 弱（固定流水线） |
| **实现复杂度** | 最高 | 低 | 中 | 低 |
| **典型场景** | 分布式推理、辩论 | 任务分解与分配 | 资源分配、竞拍 | 数据处理流水线 |

### 与 Market Pattern 的深入对比

Mesh 和 Market 都是去中心化的，但核心哲学不同：

- **Mesh**：基于**直接协作**——Agent A 直接告诉 Agent B 需要什么
- **Market**：基于**间接信号**——Agent A 发布任务和价格，Agent B 通过竞标表达兴趣

市场模式的"好处"是引入了经济抽象层来解耦供需双方，但这也意味着失去了直接沟通的丰富性。Mesh 的信息密度更高（可以传输复杂的推理结果），但耦合度也更高。

## 核心优势

### 1. 无单点故障（No Single Point of Failure）

这是 Mesh 模式最具说服力的优势。任意 Agent 离线，其他 Agent 不受影响：

```
  传统集中式:                     Mesh 模式:
     [Coordinator]                 A ─ B
      /    |    \                  │ ╲ │
     A     B     C                 C ─ D

  Coordinator 下线 →              任意 Agent 下线 →
  全部功能不可用                   其余 Agent 重新布线后继续
```

### 2. 通信灵活性最大化

任意 Agent 之间可以直接传递任何信息——不需要走固定路径，不需要中转节点理解消息内容。这使 Mesh 模式能承载的信息类型最丰富：不仅限于任务/响应，还可以传输中间推理过程、置信度、不确定性估计、反事实论证等复杂信息。

### 3. 有机适应变化

没有预定义的拓扑结构，Agent 根据交互模式自然形成"社交网络"——频繁协作的 Agent 间建立强连接，不常协作的通过弱连接保持可达。这种**自组织特性**使系统能够有机适应任务负载的变化。

### 4. 涌现行为

由于信息可以在任意 Agent 间自由流动，系统最容易产生**涌现智能**——局部交互产生超出任何单个 Agent 能力的全局行为。例如：

- **辩论涌现真理**：多个 Agent 自由辩论中，正确观点自然胜出
- **分工涌现专业**：Agent 间自由协商中，自然形成专业分工
- **竞争涌现最优**：多个 Agent 并行求解同一问题，通过互相验证涌现最佳方案

### 5. 增量扩展

添加新 Agent 不需要修改现有系统的任何配置——新 Agent 通过发现协议自动接入网络。这使 Mesh 模式非常适合需要持续扩展的系统。

## 工程优化

### 1. 连接池管理（Connection Pooling）

每个 Mesh Agent 维护到其他 Agent 的复用连接池，避免每次通信都建立新连接：

```
  Agent A 到 Agent B 的连接池:
  ┌──────────────────────────────┐
  │  ConnectionPool (max=5)      │
  │  ├── Conn #1 [in_use]        │
  │  ├── Conn #2 [in_use]        │
  │  ├── Conn #3 [idle]          │
  │  ├── Conn #4 [idle]          │
  │  └── Conn #5 [closed]        │
  │  WaitQueue: 0 pending        │
  │  TotalCreated: 23 / lifetime │
  └──────────────────────────────┘
```

### 2. 消息压缩与批处理

- **消息压缩**：对超过阈值（如 1KB）的消息体进行 gzip/snappy 压缩
- **消息批处理**：将短时间内的多条小消息合并成一个大消息包发送
- **差量同步**：只传输变更部分而非全量状态

### 3. 拓扑感知路由（Topology-Aware Routing）

考虑网络拓扑信息来优化路由选择：

```
  Agent A 到 Agent X 的路由表:
  ┌──────┬──────────┬─────────┬─────────┐
  │ Dest │ Next Hop │ Latency │ Hops    │
  ├──────┼──────────┼─────────┼─────────┤
  │ X    │ B        │ 12ms    │ 2       │
  │ X    │ C        │ 35ms    │ 3       │
  │ X    │ D        │ 8ms     │ 1 (直连) │
  └──────┴──────────┴─────────┴─────────┘
  选择策略: 最低延迟路径 (D → 直连)
```

### 4. 惰性连接建立（Lazy Connection Establishment）

只在需要通信时才建立连接，而非在发现时就全量连接：

```
  发现阶段: 记录 Agent 地址和元数据，但不建立 TCP 连接
  首次通信: 按需建立连接，缓存连接句柄
  空闲释放: 长时间无通信的连接自动释放
  重连策略: 下次需要时重新建立
```

### 5. 缓存邻居列表与心跳健康检查

```
  ┌─────────────────────────────────────────────┐
  │ Neighbor Table (Agent A)                    │
  ├──────┬──────────┬──────────┬───────┬────────┤
  │ Peer │ Address  │ LastSeen │ Status│ Latency│
  ├──────┼──────────┼──────────┼───────┼────────┤
  │ B    │ tcp://...│ 12:34:56 │ Alive │ 5ms    │
  │ C    │ tcp://...│ 12:34:50 │ Alive │ 8ms    │
  │ D    │ tcp://...│ 12:30:00 │ Suspect│ -     │
  │ E    │ tcp://...│  -       │ Dead  │ -     │
  └──────┴──────────┴──────────┴───────┴────────┘

  心跳参数:
  - Interval: 5s
  - Timeout:  15s (3 次未响应 → Suspect)
  - Death:    30s (6 次未响应 → Dead)
  - Cleanup:  5min (Dead 状态持续 → 移除)
```

### 6. 自适应扇出控制

根据网络规模动态调整 Gossip 的扇出数量（fanout）：

```
  fanout = min(k, max(3, log2(N) * 2))

  N=10   → fanout = 3
  N=50   → fanout = 6
  N=100  → fanout = 7
  N=1000 → fanout = 10
```

小网络少转发（减少冗余），大网络适度增加（保证覆盖）。

### 7. 消息去重与去环

每个消息携带全局唯一 ID（UUID），每个 Agent 维护已处理消息的布隆过滤器（Bloom Filter）来高效检测和丢弃重复消息、防止消息在网络中无限循环。

## 代码示例

以下是一个完整的 Mesh Pattern 多智能体系统 Python 伪代码实现，包含核心组件：

```python
"""
Mesh Pattern 多智能体系统 — 完整实现示例

核心组件:
  - MeshAgent: 网状网络中的单个对等智能体
  - DiscoveryProtocol: 基于 Gossip 的 Peer 发现
  - MessageRouter: 消息路由与转发
  - HealthMonitor: 对等节点健康监控
  - InformationPropagator: 带 TTL 的信息泛洪传播
"""

import asyncio
import time
import uuid
import random
import logging
from enum import Enum
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Set, Callable, Any
from collections import defaultdict

logger = logging.getLogger(__name__)


# ──────────────────────────────────────────────────────────
# 基础数据结构
# ──────────────────────────────────────────────────────────

class PeerStatus(Enum):
    """对等节点状态"""
    UNKNOWN = "unknown"
    ALIVE = "alive"
    SUSPECT = "suspect"
    DEAD = "dead"


@dataclass
class PeerInfo:
    """对等节点信息"""
    agent_id: str
    address: str
    capabilities: List[str] = field(default_factory=list)
    status: PeerStatus = PeerStatus.UNKNOWN
    last_seen: float = 0.0
    latency_ms: float = 0.0
    metadata: Dict[str, Any] = field(default_factory=dict)


@dataclass
class Message:
    """Mesh 网络中传输的消息"""
    msg_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    sender_id: str = ""
    receiver_id: str = ""           # 空字符串 = 广播/泛洪
    content: Any = None
    msg_type: str = "direct"        # direct / broadcast / gossip
    ttl: int = 5                    # Time-To-Live
    timestamp: float = field(default_factory=time.time)
    sequence: int = 0
    requires_ack: bool = False


@dataclass
class GossipMessage:
    """Gossip 协议传播的消息"""
    msg_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    origin_id: str = ""
    payload: Any = None
    ttl: int = 5
    timestamp: float = field(default_factory=time.time)


# ──────────────────────────────────────────────────────────
# 核心 MeshAgent
# ──────────────────────────────────────────────────────────

class MeshAgent:
    """
    网状模式中的对等智能体。

    每个 MeshAgent 是一个完全自治的节点，能:
    - 发现并连接其他 Agent
    - 直接发送和接收消息
    - 转发消息给其他节点（泛洪/Gossip）
    - 监控邻居健康状况
    - 动态加入/离开网络
    """

    def __init__(
        self,
        agent_id: str,
        address: str,
        capabilities: List[str] = None,
        seed_peers: List[str] = None,
    ):
        self.agent_id = agent_id
        self.address = address
        self.capabilities = capabilities or []

        # 邻居表: agent_id → PeerInfo
        self.neighbors: Dict[str, PeerInfo] = {}

        # 种子节点列表（启动入口）
        self.seed_peers: List[str] = seed_peers or []

        # 已处理消息的 Bloom Filter（去重用）
        self.seen_messages: Set[str] = set()
        self.max_seen_cache: int = 10000

        # 消息回调处理器
        self.message_handlers: Dict[str, Callable] = defaultdict(
            lambda: self._default_handler
        )

        # 状态
        self.is_running: bool = False
        self.sequence_counter: int = 0
        self.message_queue: asyncio.Queue = asyncio.Queue()

        # 配置
        self.heartbeat_interval: float = 5.0   # 心跳间隔(秒)
        self.heartbeat_timeout: float = 15.0   # 心跳超时(秒)
        self.gossip_fanout: int = 3            # Gossip 扇出数量
        self.gossip_interval: float = 10.0     # Gossip 交换间隔
        self.cleanup_interval: float = 60.0    # 清理离线节点间隔

        logger.info(f"MeshAgent [{agent_id}] initialized at {address}")

    # ──────────── 生命周期管理 ────────────

    async def start(self):
        """启动 Agent，加入 Mesh 网络"""
        self.is_running = True

        # 连接到种子节点
        for seed_addr in self.seed_peers:
            await self._connect_to_peer(seed_addr)

        # 启动后台任务
        asyncio.create_task(self._heartbeat_loop())
        asyncio.create_task(self._gossip_loop())
        asyncio.create_task(self._health_check_loop())
        asyncio.create_task(self._cleanup_loop())
        asyncio.create_task(self._message_processing_loop())

        # 广播自己已上线
        await self.broadcast_online()

        logger.info(f"MeshAgent [{self.agent_id}] started and joined mesh")

    async def stop(self):
        """停止 Agent，离开 Mesh 网络"""
        self.is_running = False

        # 广播离线消息
        await self.broadcast_offline()

        # 关闭所有连接
        for peer_id in list(self.neighbors.keys()):
            self.neighbors[peer_id].status = PeerStatus.DEAD

        logger.info(f"MeshAgent [{self.agent_id}] left mesh network")

    # ──────────── Peer 发现 ────────────

    async def _connect_to_peer(self, peer_address: str) -> Optional[str]:
        """
        尝试连接到对等节点。
        在实际实现中，这涉及 TCP/TLS/WebSocket 连接建立。

        简化实现：假设连接成功并交换身份信息。
        """
        try:
            # 模拟网络连接
            # 在实际系统中: 建立 TCP/WebSocket/gRPC 连接
            remote_id = f"peer_{hash(peer_address) % 10000}"

            peer_info = PeerInfo(
                agent_id=remote_id,
                address=peer_address,
                status=PeerStatus.ALIVE,
                last_seen=time.time(),
            )
            self.neighbors[remote_id] = peer_info

            # 发送身份介绍
            intro_msg = Message(
                sender_id=self.agent_id,
                receiver_id=remote_id,
                content={
                    "type": "introduction",
                    "agent_id": self.agent_id,
                    "address": self.address,
                    "capabilities": self.capabilities,
                },
                msg_type="direct",
                ttl=1,
            )
            await self._send_direct(remote_id, intro_msg)

            logger.info(
                f"Connected to peer [{remote_id}] at {peer_address}"
            )
            return remote_id

        except Exception as e:
            logger.warning(f"Failed to connect to {peer_address}: {e}")
            return None

    async def handle_introduction(self, peer_info: dict, from_id: str):
        """
        处理其他 Agent 的介绍消息。

        当收到新 Agent 的介绍时:
        1. 记录到邻居表
        2. 回复自己的身份信息
        3. 如果对方有引荐列表，尝试连接被引荐的节点
        """
        new_agent_id = peer_info.get("agent_id")
        if not new_agent_id:
            return

        if new_agent_id not in self.neighbors:
            self.neighbors[new_agent_id] = PeerInfo(
                agent_id=new_agent_id,
                address=peer_info.get("address", ""),
                capabilities=peer_info.get("capabilities", []),
                status=PeerStatus.ALIVE,
                last_seen=time.time(),
            )
            logger.info(
                f"Discovered new peer [{new_agent_id}] "
                f"via introduction from [{from_id}]"
            )

        # 回复自我介绍
        reply = Message(
            sender_id=self.agent_id,
            receiver_id=new_agent_id,
            content={
                "type": "introduction_reply",
                "agent_id": self.agent_id,
                "address": self.address,
                "capabilities": self.capabilities,
                "known_peers": list(self.neighbors.keys())[:10],
            },
            msg_type="direct",
            ttl=1,
        )
        await self._send_direct(new_agent_id, reply)

    # ──────────── 消息发送 ────────────

    async def send_message(
        self, receiver_id: str, content: Any,
        msg_type: str = "direct", ttl: int = 5,
        require_ack: bool = False,
    ) -> bool:
        """
        发送消息给指定 Agent。

        支持三种模式:
        - direct: 直接发送给已知邻居
        - broadcast: 发送给所有邻居，邻居继续转发
        - gossip: 使用 Gossip 协议传播
        """
        self.sequence_counter += 1
        message = Message(
            sender_id=self.agent_id,
            receiver_id=receiver_id,
            content=content,
            msg_type=msg_type,
            ttl=ttl,
            sequence=self.sequence_counter,
            requires_ack=require_ack,
        )

        if msg_type == "direct":
            return await self._send_direct(receiver_id, message)
        elif msg_type == "broadcast":
            return await self._broadcast_message(message)
        elif msg_type == "gossip":
            return await self._gossip_message(message)
        else:
            logger.warning(f"Unknown message type: {msg_type}")
            return False

    async def _send_direct(
        self, receiver_id: str, message: Message
    ) -> bool:
        """直接发送消息给指定邻居"""
        if receiver_id not in self.neighbors:
            # 尝试路由查找 — 查看是不是非直连节点
            logger.warning(
                f"Cannot send directly to [{receiver_id}]: "
                f"not in neighbor table"
            )
            return False

        peer = self.neighbors[receiver_id]
        if peer.status != PeerStatus.ALIVE:
            logger.warning(
                f"Peer [{receiver_id}] is {peer.status.value}, "
                f"cannot send"
            )
            return False

        # 实际实现: 通过 TCP/TLS 连接发送序列化消息
        # 这里简化为入队到消息队列
        await self.message_queue.put(
            ("send", receiver_id, message)
        )

        logger.debug(
            f"Message [{message.msg_id[:8]}...] sent "
            f"to [{receiver_id}]"
        )
        return True

    async def _broadcast_message(self, message: Message) -> bool:
        """广播消息给所有活跃邻居"""
        if message.ttl <= 0:
            return False

        # 去重检查
        if message.msg_id in self.seen_messages:
            return False
        self._mark_seen(message.msg_id)

        # TTL 递减
        message.ttl -= 1

        sent_count = 0
        for peer_id, peer in self.neighbors.items():
            if peer.status == PeerStatus.ALIVE and peer_id != message.sender_id:
                await self._send_direct(peer_id, message)
                sent_count += 1

        logger.debug(
            f"Broadcast [{message.msg_id[:8]}...] "
            f"forwarded to {sent_count} peers"
        )
        return sent_count > 0

    async def _gossip_message(self, message: Message) -> bool:
        """
        使用 Gossip 协议传播消息。

        选择随机的 k 个邻居转发，不用发给所有人。
        这样可以显著降低网络负载同时保证高概率覆盖。
        """
        if message.ttl <= 0:
            return False

        if message.msg_id in self.seen_messages:
            return False
        self._mark_seen(message.msg_id)

        message.ttl -= 1

        # 随机选择 fanout 个邻居
        alive_peers = [
            pid for pid, p in self.neighbors.items()
            if p.status == PeerStatus.ALIVE
            and pid != message.sender_id
        ]

        if not alive_peers:
            return False

        fanout = min(self.gossip_fanout, len(alive_peers))
        targets = random.sample(alive_peers, fanout)

        for target_id in targets:
            await self._send_direct(target_id, message)

        logger.debug(
            f"Gossip [{message.msg_id[:8]}...] "
            f"sent to {fanout} random peers"
        )
        return True

    def _mark_seen(self, msg_id: str):
        """记录已处理消息 ID 用于去重"""
        self.seen_messages.add(msg_id)
        # 限制缓存大小
        if len(self.seen_messages) > self.max_seen_cache:
            # 移除最旧的 50%
            cutoff = len(self.seen_messages) // 2
            self.seen_messages = set(
                list(self.seen_messages)[cutoff:]
            )

    # ──────────── 消息接收 ────────────

    async def on_message(
        self, message: Message, from_id: str
    ):
        """
        收到来自其他 Agent 的消息。

        消息处理流程:
        1. 去重检查
        2. 如果是 Direct 消息: 处理内容
        3. 如果是 Broadcast/Gossip: 先处理，然后继续转发
        """
        # 1. 去重
        if message.msg_id in self.seen_messages:
            return
        self._mark_seen(message.msg_id)

        # 2. 更新邻居状态
        if from_id in self.neighbors:
            self.neighbors[from_id].last_seen = time.time()
            self.neighbors[from_id].status = PeerStatus.ALIVE

        # 3. 处理消息内容
        handler = self.message_handlers.get(
            message.msg_type, self._default_handler
        )
        await handler(message, from_id)

        # 4. 如果是广播/Gossip 且 TTL > 0，继续转发
        if message.msg_type in ("broadcast", "gossip") and message.ttl > 0:
            if message.msg_type == "broadcast":
                await self._broadcast_message(message)
            else:
                await self._gossip_message(message)

    async def _default_handler(
        self, message: Message, from_id: str
    ):
        """默认消息处理器"""
        logger.info(
            f"Unhandled message [{message.msg_type}] "
            f"from [{from_id}]: {message.content}"
        )

    def register_handler(
        self, msg_type: str, handler: Callable
    ):
        """注册特定类型的消息处理器"""
        self.message_handlers[msg_type] = handler

    # ──────────── 广播生命周期事件 ────────────

    async def broadcast_online(self):
        """广播自己已上线"""
        await self.send_message(
            receiver_id="",
            content={
                "type": "peer_online",
                "agent_id": self.agent_id,
                "address": self.address,
                "capabilities": self.capabilities,
                "timestamp": time.time(),
            },
            msg_type="broadcast",
            ttl=3,
        )

    async def broadcast_offline(self):
        """广播自己即将离线"""
        await self.send_message(
            receiver_id="",
            content={
                "type": "peer_offline",
                "agent_id": self.agent_id,
                "timestamp": time.time(),
            },
            msg_type="broadcast",
            ttl=3,
        )

    # ──────────── 信息泛洪 ────────────

    async def flood_information(
        self,
        content: Any,
        ttl: int = 5,
        fanout: Optional[int] = None,
    ):
        """
        将信息通过泛洪方式传播到整个 Mesh 网络。

        Gossip 风格的传播:
        - 每个节点收到后转发给随机 k 个邻居
        - TTL 控制传播范围
        - 通过 Message ID 去重防止环路

        适用于: 通知全局事件、发布 Agent 能力更新、状态同步
        """
        gossip_msg = GossipMessage(
            origin_id=self.agent_id,
            payload=content,
            ttl=ttl,
        )

        # 转换为普通消息通过 gossip 协议发送
        msg = Message(
            sender_id=self.agent_id,
            content=gossip_msg.__dict__,
            msg_type="gossip",
            ttl=ttl,
        )

        await self._gossip_message(msg)
        logger.info(
            f"Flood initiated: {str(content)[:50]}... "
            f"TTL={ttl}"
        )

    # ──────────── 心跳与健康监控 ────────────

    async def _heartbeat_loop(self):
        """
        定期发送心跳给所有邻居。

        心跳内容包含:
        - agent_id: 身份
        - timestamp: 当前时间
        - known_peers: 已知节点列表（用于发现扩散）
        """
        while self.is_running:
            await asyncio.sleep(self.heartbeat_interval)

            for peer_id in list(self.neighbors.keys()):
                if self.neighbors[peer_id].status != PeerStatus.DEAD:
                    heartbeat = Message(
                        sender_id=self.agent_id,
                        receiver_id=peer_id,
                        content={
                            "type": "heartbeat",
                            "agent_id": self.agent_id,
                            "timestamp": time.time(),
                            "known_peers": list(self.neighbors.keys()),
                        },
                        msg_type="direct",
                        ttl=1,
                    )
                    await self._send_direct(peer_id, heartbeat)

    async def _health_check_loop(self):
        """
        定期检查邻居健康状况。

        健康检查策略:
        1. 检查每个邻居的 last_seen 时间
        2. 超过 timeout 未响应 → SUSPECT
        3. 超过 2*timeout 未响应 → DEAD
        4. DEAD 节点等待清理
        """
        while self.is_running:
            await asyncio.sleep(self.heartbeat_interval)

            now = time.time()
            for peer_id, peer in self.neighbors.items():
                time_since_last = now - peer.last_seen

                if time_since_last > self.heartbeat_timeout * 2:
                    peer.status = PeerStatus.DEAD
                    logger.warning(
                        f"Peer [{peer_id}] declared DEAD "
                        f"(no contact for {time_since_last:.0f}s)"
                    )
                elif time_since_last > self.heartbeat_timeout:
                    if peer.status == PeerStatus.ALIVE:
                        peer.status = PeerStatus.SUSPECT
                        logger.warning(
                            f"Peer [{peer_id}] marked SUSPECT "
                            f"(last seen {time_since_last:.0f}s ago)"
                        )

    async def _cleanup_loop(self):
        """
        定期清理离线节点和过期的已处理消息记录。
        """
        while self.is_running:
            await asyncio.sleep(self.cleanup_interval)

            # 移除 DEAD 超过 5 分钟的节点
            now = time.time()
            dead_peers = [
                pid for pid, p in self.neighbors.items()
                if p.status == PeerStatus.DEAD
                and (now - p.last_seen) > 300
            ]
            for pid in dead_peers:
                del self.neighbors[pid]
                logger.info(f"Cleaned up dead peer [{pid}]")

    # ──────────── Gossip 信息交换 ────────────

    async def _gossip_loop(self):
        """
        定期与随机邻居交换成员信息。

        这是 Peer 发现的核心机制:
        1. 选择 1-2 个随机邻居
        2. 交换已知节点列表
        3. 合并去重后更新邻居表
        4. 连接新发现的节点

        通过这种方式，新 Agent 的信息会在 O(log N) 轮内
        传播到网络中的所有节点。
        """
        while self.is_running:
            await asyncio.sleep(self.gossip_interval)

            alive_peers = [
                pid for pid, p in self.neighbors.items()
                if p.status == PeerStatus.ALIVE
            ]

            if not alive_peers:
                continue

            # 随机选一个邻居交换信息
            peer_id = random.choice(alive_peers)
            membership_msg = Message(
                sender_id=self.agent_id,
                receiver_id=peer_id,
                content={
                    "type": "membership_exchange",
                    "known_peers": [
                        {
                            "id": pid,
                            "addr": p.address,
                            "caps": p.capabilities,
                            "status": p.status.value,
                        }
                        for pid, p in self.neighbors.items()
                    ],
                    "agent_id": self.agent_id,
                },
                msg_type="direct",
                ttl=1,
            )
            await self._send_direct(peer_id, membership_msg)

    async def handle_membership_exchange(
        self, peer_list: list, from_id: str
    ):
        """
        处理来自邻居的成员信息交换。

        合并策略:
        - 发现新节点 → 添加到邻居表（状态 UNKNOWN）
        - 已知节点信息更新 → 合并更新
        - 获得新地址 → 尝试连接
        """
        new_peers_discovered = 0
        for peer_data in peer_list:
            pid = peer_data.get("id")
            if not pid or pid == self.agent_id:
                continue

            if pid not in self.neighbors:
                # 发现新节点，尝试连接
                addr = peer_data.get("addr", "")
                if addr:
                    self.neighbors[pid] = PeerInfo(
                        agent_id=pid,
                        address=addr,
                        capabilities=peer_data.get("caps", []),
                        status=PeerStatus.UNKNOWN,
                        last_seen=time.time(),
                    )
                    new_peers_discovered += 1
                    # 尝试连接新节点
                    asyncio.create_task(
                        self._connect_to_peer(addr)
                    )

        if new_peers_discovered > 0:
            logger.info(
                f"Discovered {new_peers_discovered} new peers "
                f"via gossip with [{from_id}]"
            )

    # ──────────── 网络分区恢复 ────────────

    async def detect_and_heal_partition(self):
        """
        检测并尝试修复网络分区。

        分区检测:
        1. 通过成员交换发现自己的节点集合与其他节点的集合不一致
        2. 如果存在某节点无法通过任何已知路径到达，发生分区

        分区修复:
        1. 尝试重新连接已知但标记为 DEAD 的节点
        2. 请求其他节点帮助建立与孤立节点的连接
        3. 如果节点长期不可达，触发网络重配置
        """
        logger.info("Checking for network partitions...")

        # 收集已知但不可达的节点
        unreachable = [
            pid for pid, p in self.neighbors.items()
            if p.status == PeerStatus.DEAD
        ]

        for peer_id in unreachable:
            logger.info(
                f"Attempting to reconnect to dead peer [{peer_id}]"
            )
            # 重新尝试建立连接
            addr = self.neighbors[peer_id].address
            if addr:
                new_id = await self._connect_to_peer(addr)
                if new_id:
                    logger.info(
                        f"Partition healed: reconnected to [{peer_id}]"
                    )

        # 如果仍有节点不可达，请求其他节点帮忙
        still_dead = [
            pid for pid, p in self.neighbors.items()
            if p.status == PeerStatus.DEAD
        ]
        if still_dead:
            logger.warning(
                f"Still {len(still_dead)} peers unreachable "
                f"after reconnection attempt"
            )

    # ──────────── 消息处理循环 ────────────

    async def _message_processing_loop(self):
        """后台消息处理循环"""
        while self.is_running:
            try:
                action, target, message = (
                    await asyncio.wait_for(
                        self.message_queue.get(),
                        timeout=1.0,
                    )
                )
                if action == "send":
                    # 实际系统中: 序列化并通过网络连接发送
                    pass
            except asyncio.TimeoutError:
                continue
            except Exception as e:
                logger.error(f"Message processing error: {e}")


# ──────────────────────────────────────────────────────────
# 高级功能: 信息传播（泛洪带优先级）
# ──────────────────────────────────────────────────────────

class InformationPropagator:
    """
    信息传播器 — 提供可靠的 Mesh 网络信息扩散能力。

    功能:
    - 带优先级的消息传播（紧急消息优先扩散）
    - 自适应 TTL 控制
    - 传播确认和重试（有损网络中的可靠性）
    """

    def __init__(self, agent: MeshAgent):
        self.agent = agent
        self.pending_acks: Dict[str, float] = {}
        self.critical_messages: asyncio.Queue = asyncio.Queue()

    async def propagate(
        self,
        content: Any,
        priority: str = "normal",
        ttl: int = 5,
        require_confirmation: bool = False,
    ):
        """
        在 Mesh 中传播信息。

        Args:
            content: 要传播的信息
            priority: "critical" | "normal" | "low"
            ttl: Time-To-Live
            require_confirmation: 是否需要全局确认
        """
        msg_id = str(uuid.uuid4())

        if priority == "critical":
            await self._critical_propagation(content, ttl)
        else:
            await self._normal_propagation(content, ttl)

        if require_confirmation:
            self.pending_acks[msg_id] = time.time()

    async def _critical_propagation(
        self, content: Any, ttl: int
    ):
        """
        紧急信息传播:
        - 采用 Push-Pull 混合 Gossip
        - 更高的扇出（fanout = 2 * normal）
        - 更频繁的传播尝试
        """
        original_fanout = self.agent.gossip_fanout
        self.agent.gossip_fanout = min(
            original_fanout * 2, 10
        )

        try:
            await self.agent.flood_information(
                content=content, ttl=ttl
            )

            # Push-Pull: 主动推送 + 请求邻居主动获取
            for peer_id in list(self.agent.neighbors.keys()):
                if self.agent.neighbors[peer_id].status == PeerStatus.ALIVE:
                    await self.agent.send_message(
                        receiver_id=peer_id,
                        content={
                            "type": "pull_request",
                            "critical_update": True,
                            "since": time.time() - 30,
                        },
                        msg_type="direct",
                        ttl=1,
                    )
        finally:
            self.agent.gossip_fanout = original_fanout


# ──────────────────────────────────────────────────────────
# 使用示例
# ──────────────────────────────────────────────────────────

async def mesh_demo():
    """
    Mesh Pattern 使用示例。

    场景: 5 个 Agent 组成去中心化系统，
    其中一个 Agent 发布信息，其他 Agent 通过 Gossip 接收。
    """

    # 创建 5 个 MeshAgent
    agents = {
        f"Agent_{i}": MeshAgent(
            agent_id=f"Agent_{i}",
            address=f"tcp://localhost:900{i}",
            capabilities=["research", "reasoning", "coding"],
            seed_peers=[f"tcp://localhost:9000"],
        )
        for i in range(5)
    }

    # 注册消息处理器
    async def handle_question(message, from_id):
        print(
            f"[{message.receiver_id}] Received question "
            f"from [{from_id}]: {message.content}"
        )
        # 回复
        await agents[message.receiver_id].send_message(
            receiver_id=from_id,
            content={
                "answer": f"Answer from {message.receiver_id}",
                "confidence": 0.85,
            },
            msg_type="direct",
        )

    # 所有 Agent 注册消息处理器
    for agent_id, agent in agents.items():
        agent.register_handler("question", handle_question)

    # 启动所有 Agent
    for agent in agents.values():
        await agent.start()

    # Agent_0 发送问题给 Agent_2
    await agents["Agent_0"].send_message(
        receiver_id="Agent_2",
        content={"question": "What is the meaning of life?"},
        msg_type="direct",
    )

    # Agent_3 通过 Gossip 广播信息
    propagator = InformationPropagator(agents["Agent_3"])
    await propagator.propagate(
        content={"event": "system_update", "version": "2.0.0"},
        priority="critical",
        ttl=3,
    )

    # 模拟 Agent_4 离线
    await agents["Agent_4"].stop()
    print("Agent_4 left the network")

    # 其他 Agent 继续正常工作
    await agents["Agent_0"].send_message(
        receiver_id="Agent_1",
        content={"task": "continue_processing"},
        msg_type="direct",
    )

    # 等待消息处理
    await asyncio.sleep(2)

    # 停止所有 Agent
    for agent in agents.values():
        if agent.is_running:
            await agent.stop()

    print("Mesh demo completed")


# ──────────────────────────────────────────────────────────
# 拓扑适配器: 从全连接到部分连接
# ──────────────────────────────────────────────────────────

class TopologyManager:
    """
    拓扑管理器 — 根据通信模式自适应调整网络拓扑。

    核心策略:
    1. 监控 Agent 间的通信频率
    2. 高频通信的对 → 保持直连
    3. 低频通信的对 → 降级为通过中间节点路由
    4. 定期重新评估连接必要性
    """

    def __init__(self, agent: MeshAgent):
        self.agent = agent
        self.communication_counts: Dict[str, int] = defaultdict(int)
        self.max_direct_connections: int = 10
        self.evaluation_interval: float = 30.0

    async def record_communication(self, peer_id: str):
        """记录与某个 Peer 的通信"""
        self.communication_counts[peer_id] += 1

    async def evaluate_topology(self):
        """
        评估并调整拓扑结构。

        高频率通信: 保持或建立直连
        低频率通信: 考虑断开直连，通过路由通信
        新增高频通信: 建立新直连
        """
        # 按通信频率排序
        ranked = sorted(
            self.communication_counts.items(),
            key=lambda x: x[1],
            reverse=True,
        )

        # 高频通信: 确保直连
        high_frequency = ranked[:self.max_direct_connections]

        # 低频通信: 考虑断开
        low_frequency = ranked[self.max_direct_connections:]

        for peer_id, _ in low_frequency:
            if peer_id in self.agent.neighbors:
                # 不是完全断开，只是降级
                logger.info(
                    f"Demoting connection to [{peer_id}]: "
                    f"low communication frequency"
                )

        # 确保高频通信的 Peer 都在邻居表中
        for peer_id, _ in high_frequency:
            if peer_id not in self.agent.neighbors:
                logger.info(
                    f"Promoting connection to [{peer_id}]: "
                    f"high communication frequency"
                )


# ──────────────────────────────────────────────────────────
# 运行
# ──────────────────────────────────────────────────────────

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    asyncio.run(mesh_demo())
```

### 代码结构说明

| 组件 | 类/函数 | 职责 |
|------|---------|------|
| 对等节点 | `MeshAgent` | 每个 Agent 的核心实现，包含邻居表、消息发送/接收、心跳维护 |
| 消息模型 | `Message` | 网络传输的消息结构，包含 ID、发送方、TTL、类型等 |
| 邻居信息 | `PeerInfo` | 对等节点的元数据：地址、能力、状态、延迟等 |
| 发现协议 | `_gossip_loop()` | 通过周期性与随机邻居交换成员信息来发现新节点 |
| 信息传播 | `InformationPropagator` | 支持优先级和确认的高层信息扩散接口 |
| 拓扑管理 | `TopologyManager` | 根据通信模式自适应调整拓扑（降级/升级连接） |
| 健康监控 | `_health_check_loop()` | 通过心跳超时检测节点故障，标记 SUSPECT/DEAD |
| 分区恢复 | `detect_and_heal_partition()` | 检测网络分区并尝试重新连接孤立节点 |

### 代码中的关键设计决策

1. **去重机制**：`seen_messages` 集合配合 Bloom Filter 心智模型，防止消息在网络中无限循环
2. **TTL 控制**：每转发一次 TTL 减 1，TTL=0 停止转发，精确控制扩散半径
3. **自适应扇出**：`gossip_fanout` 参数控制传播速度和负载的平衡
4. **状态机健康模型**：`ALIVE → SUSPECT → DEAD → 清理`，逐级降级而非立即断开
5. **Gossip 成员交换**：信息通过"我认识 X，X 认识 Y"的链条逐步传播到全网
6. **Pull 机制**：不仅 Push，还支持主动 Pull 请求，形成 Push-Pull 混合 Gossip

## 推荐场景

### 最适合使用 Mesh 的场景

1. **多 Agent 辩论/讨论系统**：多个具有不同视角的 Agent 自由交换论点，通过辩论收敛到最佳答案——Mesh 的自然涌现特性完美匹配
2. **分布式研究助手**：多个专业化研究 Agent 各自负责不同领域，自由交换研究发现和交叉引用
3. **去中心化决策系统**：没有单一决策者，多个 Agent 各自分析后交换意见形成集体决策
4. **动态传感器/监控网络**：传感器 Agent 随时加入/离开，需要自组织的通信拓扑
5. **P2P 协作计算**：多个计算 Agent 协作求解复杂问题，均分任务和结果

### 应避免使用 Mesh 的场景

1. **严格分层管理的组织架构**：需要明确指挥链和责任归属的系统
2. **Agent 数量极大（100+）**：连接维护和消息风暴将不可控
3. **需要强一致性保证**：Mesh 天然倾向最终一致性
4. **对审计追踪有严格要求的系统**：分布式拓扑导致消息路径追踪困难
5. **计算资源受限的节点**：每个节点需要承担路由和转发职责

## 延伸阅读

- Kademlia: A Peer-to-Peer Information System Based on the XOR Metric (Petar Maymounkov, David Mazieres, 2002)
- Epidemic Algorithms for Replicated Database Maintenance (Alan Demers et al., 1987) — Gossip 协议的奠基性论文
- SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol (Das, Gupta, Motivala, 2002) — 实用的 Gossip 成员管理协议
- Chord: A Scalable Peer-to-Peer Lookup Service for Internet Applications (Stoica et al., 2001) — 结构化 P2P 路由
- Small-World Networks: Evidence and an Algorithm (Watts & Strogatz, 1998) — 小世界网络模型
- Conflict-free Replicated Data Types (Shapiro et al., 2011) — CRDT 无冲突合并数据类型
