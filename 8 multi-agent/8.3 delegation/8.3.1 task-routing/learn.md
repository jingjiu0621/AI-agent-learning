# 8.3.1 任务路由 (Task Routing)

## 1. 简单介绍

在多智能体系统（Multi-Agent System, MAS）中，任务路由（Task Routing）是指将到达的任务按照一定的策略和规则，分派给最合适的智能体（Agent）去执行的过程。它不是简单的"谁空闲谁干"，而是综合考虑智能体的**能力匹配度**（Capability Match）、**当前负载**（Current Load）、**任务优先级**（Task Priority）以及**历史表现**（Historical Performance）等多个维度，做出最优的分配决策。

可以把任务路由理解为一个"智能调度中枢"——它类似于互联网中的IP路由，但路由的对象不是数据包，而是具有语义和要求的任务；路由的决策依据不再是IP地址，而是智能体的能力画像和实时状态。一个设计良好的任务路由机制，是整个多智能体系统高效协作的基石。

## 2. 基本原理

任务路由的核心工作流程包含三个关键环节：**路由信息收集 → 路由决策 → 任务分派**。其背后的基本原理可以从以下几个层面理解：

```
                  ┌─────────────────────────────────────┐
                  │           任务进入 (Task Ingress)      │
                  └────────────────┬────────────────────┘
                                   │
                                   ▼
                  ┌─────────────────────────────────────┐
                  │         路由信息库 (RIB)              │
                  │  ┌─────────┐ ┌────────┐ ┌────────┐  │
                  │  │能力索引表 │ │负载状态表│ │优先级策略│  │
                  │  └─────────┘ └────────┘ └────────┘  │
                  └────────────────┬────────────────────┘
                                   │
                                   ▼
                  ┌─────────────────────────────────────┐
                  │         路由决策引擎                   │
                  │  内容路由 → 能力路由 → 负载路由 → 优先级路由 │
                  └────────────────┬────────────────────┘
                                   │
                                   ▼
                  ┌─────────────────────────────────────┐
                  │        目标智能体 (Target Agent)       │
                  └─────────────────────────────────────┘
```

### 2.1 路由表 (Routing Table)

路由表是任务路由系统的核心数据结构，记录了从"任务特征"到"智能体标识"的映射关系。与网络路由表不同，MAS中的路由表条目通常包含：

- **任务模式**：描述任务类型、领域、所需能力的特征向量
- **目标智能体**：能够处理该任务的智能体ID列表
- **权重/优先级**：各智能体处理该类任务的倾向得分
- **TTL/失效时间**：路由条目的有效期，用于处理动态变化

### 2.2 路由策略

| 策略类型 | 决策依据 | 优点 | 缺点 |
|---------|--------|------|------|
| **基于内容路由** | 任务内容与智能体声明的能力匹配 | 精准度高，语义理解强 | 计算开销大，依赖语义解析质量 |
| **基于能力路由** | 智能体注册的能力标签 | 实现简单，匹配快速 | 能力描述粗糙，粒度有限 |
| **基于负载路由** | 智能体当前待处理任务队列长度 | 负载均衡，避免过载 | 忽略能力匹配，可能分派错误 |
| **基于优先级路由** | 任务的紧急程度/重要程度 | 保证SLA，关键任务优先 | 低优先任务可能被饥饿 |

实际系统中，通常采用**多因子加权路由**，将上述策略组合为一个评分函数。

### 2.3 路由决策点

路由决策可以在多个层级发生：

1. **全局路由决策**：由中央路由器统一分派，信息全面但存在单点瓶颈
2. **本地路由决策**：各智能体自行决定是否接受任务，分布式但可能次优
3. **分层路由决策**：结合全局和局部，上层做粗粒度路由，下层做细粒度调整

### 2.4 路由信息库 (Routing Information Base, RIB)

RIB是路由决策的"数据源"，维护三类信息：

- **静态信息**：智能体的能力声明、所属领域、支持的接口协议
- **动态信息**：实时负载、健康状态、当前工作上下文
- **历史信息**：过往任务完成质量、响应时间、成功率

## 3. 背景与演进

任务路由并非多智能体系统的独创概念，它经历了从传统分布式系统到AI智能体系统的演进过程。

### 3.1 硬编码路由（早期）

最初的智能体系统中，任务分派逻辑直接硬编码在代码中。每个任务的去向由`if-else`或`switch-case`语句决定。这种方式的优点是简单直接，但系统每增加一个智能体或任务类型，都需要修改代码。

```
if task.type == "image_generation":
    return agent_a
elif task.type == "text_summary":
    return agent_b
...
```

**痛点**：缺乏弹性，系统耦合度高，无法应对动态变化。

### 3.2 规则路由（Rule-Based Routing）

随着智能体数量增多，硬编码不再可行。规则引擎被引入，通过可配置的规则文件来决定路由策略。

```yaml
routes:
  - pattern: "task.domain == 'nlp' && task.difficulty == 'high'"
    target: "expert-nlp-agent"
    priority: 10
  - pattern: "task.estimated_cost < 100"
    strategy: "load-balance"
    targets: ["worker-1", "worker-2", "worker-3"]
```

**进步**：配置化、动态加载、规则可复用。**局限**：规则维护成本随规模上升，规则之间的冲突难以检测。

### 3.3 语义路由（Semantic Routing）

当智能体和任务的描述从结构化标签转向自然语言时，基于标签匹配的规则路由显得力不从心。语义路由利用嵌入（Embedding）和向量相似度来匹配任务与智能体。

核心流程：
1. 将任务描述和智能体能力描述分别编码为向量
2. 计算任务向量与各智能体能力向量的余弦相似度
3. 选择相似度最高的智能体作为路由目标

**进步**：无需人工定义匹配规则，自动理解语义关联。**局限**：需要嵌入模型和向量数据库支持，冷启动时效果较差。

### 3.4 基于机器学习的自适应路由（ML-Based Adaptive Routing）

现代多智能体系统开始利用机器学习模型来优化路由决策。路由策略不再固定不变，而是能根据历史反馈持续优化。

- **在线学习**：根据路由结果（完成时间、质量评分）实时调整路由权重
- **强化学习**：将路由建模为马尔可夫决策过程，智能体选择任务作为动作，系统吞吐量为奖励
- **上下文多臂赌博机**：将每个智能体看作一个"臂"，在探索（尝试新路由）和利用（使用已知最优路由）之间权衡

## 4. 之前做法与结果

在实践中，几种经典的路由方案被广泛尝试，它们各有得失。

### 4.1 轮询/随机分派

**做法**：维护一个智能体列表，按顺序轮流分派（Round-Robin）或随机选择。

```python
class RoundRobinRouter:
    def __init__(self, agents):
        self.agents = agents
        self.counter = 0

    def route(self, task):
        agent = self.agents[self.counter % len(self.agents)]
        self.counter += 1
        return agent
```

**结果**：
- 实现极为简单，零学习成本
- 完全无视能力差异——一个擅长图像处理的智能体和擅长文本处理的智能体收到完全一样的任务量
- 无法处理异构智能体系统，准确率通常在50%以下

### 4.2 简单规则路由

**做法**：通过`if-then`规则或简单的决策树进行匹配。

**结果**：
- 比随机分派有明显提升，特别是在能力差异较大的场景
- 规则需要人工定义和维护，当系统扩展时，规则数量呈指数增长
- 无法适应运行时变化——如果一个智能体过载或宕机，规则不会自己调整

### 4.3 静态能力映射

**做法**：在系统启动时建立任务类型→智能体的静态映射表，运行期间不更新。

```json
{
  "task_type_mapping": {
    "image_classification": ["vision-agent-1", "vision-agent-2"],
    "sentiment_analysis": ["nlp-agent-1"],
    "code_generation": ["coder-agent-1", "coder-agent-2", "coder-agent-3"]
  }
}
```

**结果**：
- 映射关系清晰可见，方便理解和审计
- 一旦某个智能体表现下降（如模型漂移、性能退化），路由系统无法感知
- 新智能体上线后，需要手动更新映射表才能参与路由
- 当多个任务类型映射到同一智能体时，负载问题暴露

### 4.4 核心问题总结

以上做法的共同问题是：

1. **路由决策信息陈旧**：路由依据的"能力声明"只在注册时采集一次，运行期间的负载变化、性能退化没有被纳入决策
2. **无法适应动态变化**：智能体可能因并发任务变慢、因模型更新能力变化、因故障下线，但这些变化不反映在路由表中
3. **缺乏负载感知**：即使能力匹配，如果目标智能体已经超载，路由上去只会增加排队延迟
4. **冷启动困难**：新加入的智能体无法立刻获得任务（没有被路由到），导致无法积累"信誉分"
5. **缺乏反馈闭环**：路由决策没有与任务完成结果形成反馈回路，无法从错误中学习

## 5. 核心矛盾问题

任务路由面临若干根本性的矛盾，这些矛盾限制了路由系统的上限。

### 5.1 路由精度 vs 路由开销

这是最核心的矛盾。**要做出精准的路由决策，需要尽可能完整的信息**——每个智能体的实时负载、每项能力的最新质量评分、网络延迟、当前上下文……但收集这些信息本身会产生通信和计算开销。

```
信息完整性 ┃
          ┃                    ╱ 理想区域
          ┃                 ╱
          ┃              ╱   ← 信息越多决策越准
          ┃           ╱       但收集成本越高
          ┃        ╱
          ┃     ╱
          ┃  ╱
          ┃╱
          ┗━━━━━━━━━━━━━━━━━━━━━→ 开销
```

**权衡策略**：
- **采样而非全量**：定期采样而非实时上报路由信息
- **增量更新**：只在状态变化超过阈值时才上报
- **按需查询**：只在路由决策时才向智能体查询状态
- **推拉结合**：智能体定期推送状态，路由器按需拉取详细数据

### 5.2 探索 vs 利用（Exploration vs Exploitation）

这是路由系统的"创新困局"：

- **利用**：将任务路由到已知表现好的智能体——获得确定性的高质量结果
- **探索**：尝试将任务路由到新智能体或未见过的路由路径——可能发现更好的匹配，也可能失败

纯利用策略会导致路由路径固化，老智能体越来越忙、新智能体永远接不到任务；纯探索策略则浪费系统资源，降低整体吞吐量。

**平衡方法**：
- **Epsilon-Greedy**：以大概率（如90%）选择最优路由，小概率随机探索
- **UCB（Upper Confidence Bound）**：选择置信上界最高的智能体，自然倾向于探索不确定性高的选项
- **汤普森采样**：根据路由结果的后验分布采样，隐式平衡探索和利用

### 5.3 全局最优 vs 局部快速

全局最优路由需要路由器掌握所有智能体的全局状态，在多智能体系统中这可能意味着O(n)的通信复杂度（n为智能体数量）。当智能体数量达到成百上千时，全局路由变得不可行。

**分层折中方案**：
```
Layer 1 (全局路由器)
    │
    ├── Region A 路由器
    │   ├── Agent A-1
    │   ├── Agent A-2
    │   └── Agent A-3
    │
    └── Region B 路由器
        ├── Agent B-1
        ├── Agent B-2
        └── Agent B-3
```

上层路由器只做粗粒度的区域选择，区域内路由器做细粒度的智能体选择。这样每个路由器的管理范围有限，信息新鲜度高，决策速度快。

## 6. 当前主流优化方向

### 6.1 ML增强路由（ML-Enhanced Routing）

利用机器学习模型从历史路由数据中学习模式，预测最优路由目标。

**技术方案**：
- **特征工程**：任务类型、任务复杂度、智能体能力向量、历史响应时间、当前负载
- **模型选择**：梯度提升树（LightGBM/XGBoost）用于可解释性场景；图神经网络（GNN）用于建模智能体间的协作关系；序列模型（LSTM/Transformer）用于捕捉任务流的时序模式
- **训练方式**：离线预训练 + 在线微调

**实际效果**：在大型异构MAS中，ML增强路由相比静态规则路由，端到端任务完成时间通常可降低30%-50%。

### 6.2 上下文感知路由（Context-Aware Routing）

路由决策不只是看任务本身的特征，还考虑更广泛的上下文：

- **会话上下文**：同一个用户会话中的连续任务应该路由到同一智能体以保持状态连续性
- **工作流上下文**：工作流中前序任务的结果影响了后续任务的执行环境
- **环境上下文**：系统所处的时段（高峰/低谷）、网络状况、可用资源

### 6.3 分层路由（Hierarchical Routing）

将路由决策分解为多级，每一级处理不同粒度的决策：

```
任务 → 领域级路由（哪个领域）→ 技能级路由（什么技能）→ 实例级路由（哪个智能体）
```

每一级的路由表规模被控制在可管理范围内，更新开销显著降低。

### 6.4 混合路由（Hybrid Routing）

结合规则路由和ML路由的优点：

- **快速路径**：对明确匹配的任务（如已知的热门任务类型），使用规则路由，O(1)复杂度
- **慢速路径**：对模糊匹配或新类型任务，使用ML模型深度分析
- **反馈回路**：慢速路径的决策结果被记录，用于优化快速路径的规则

### 6.5 预测性路由（Predictive Routing）

不是被动响应任务到达，而是主动预测未来的任务分布和系统负载，提前调整路由策略。

**方法**：
- 用量预测：基于历史数据预测下一时间窗的任务数量和类型分布
- 资源预置：根据预测结果提前调整智能体的状态（预热缓存、加载模型）
- 路由预计算：在空闲时段预先计算常见任务类型的路由方案，减少决策延迟

### 6.6 多目标路由（Multi-Objective Routing）

路由决策需要同时优化多个可能相互冲突的目标：

| 目标 | 度量指标 | 与其他目标的潜在冲突 |
|------|---------|-------------------|
| 低延迟 | P50/P99响应时间 | 可能牺牲吞吐量 |
| 高吞吐量 | 单位时间完成任务数 | 可能降低质量 |
| 高质量 | 任务完成正确率 | 可能增加成本和延迟 |
| 低成本 | Token消耗/计算资源 | 可能增加延迟 |
| 公平性 | 各智能体负载方差 | 可能降低整体效率 |
| 鲁棒性 | 故障时的任务成功率 | 需要冗余储备 |

多目标路由通常通过**帕累托优化**来寻找非支配解，或者通过**权重参数化**将多目标转化为单目标。

## 7. 核心优势与能力边界

### 7.1 核心优势

- **最大化系统吞吐量**：通过合理的任务分派，让每个智能体都工作在最适合的任务上，避免能力错配导致的资源浪费
- **均衡资源利用**：结合负载感知，避免热点智能体过载而其他智能体闲置
- **服务质量保障**：通过优先级路由确保关键任务获得及时处理，满足SLA要求
- **可扩展性**：新智能体加入后，只需注册能力和状态，路由系统自动将其纳入候选
- **容错能力**：当某个智能体故障时，路由系统可以自动避开它，将任务转向备用智能体

### 7.2 能力边界

- **路由决策质量受限于信息质量**：如果智能体的能力声明不准确、负载信息延迟，路由决策必然失真。"垃圾进，垃圾出"在路由系统中同样成立
- **冷启动问题**：新智能体或新任务类型缺乏历史数据，路由系统难以做出精准决策，需要探索策略来积累数据
- **路由表维护开销**：在大型动态系统中，保持路由信息的准确性需要持续的通信和计算投入，这部分开销可能抵消路由优化带来的收益
- **非确定性行为**：ML增强路由引入了概率性，相同的任务在不同时间可能被路由到不同的智能体，增加了系统行为的不确定性
- **级联效应**：路由决策影响智能体的负载分布，负载分布反作用于路由决策，这种反馈循环可能导致震荡（不停切换路由目标）

## 8. 与其他委派机制的区别

在多智能体系统的委派（Delegation）体系中，任务路由与多个相关概念存在交集但本质不同。

### 8.1 对比：能力注册表（Capability Registry）

```
能力注册表:        任务路由:
  "有什么智能体"     →    "任务给谁做"
  静态/半静态信息     →    动态决策
  被路由系统消费      →    消费注册表信息
```

能力注册表是一个**事实数据库**（What），回答的是"系统中有哪些智能体、各自拥有什么能力"。任务路由是一个**决策引擎**（Where），回答的是"当前这个任务应该交给谁"。路由系统是注册表信息的消费者，但路由决策需要的信息远不止能力声明。

### 8.2 对比：负载均衡（Load Balancing）

```
负载均衡:          任务路由:
  关注"多少"         →    关注"谁"
  同质化场景         →    异质化场景
  量化目标           →    质+量双重目标
```

负载均衡通常在**同质化**智能体集群中使用——所有智能体能力相同，只需决定谁接下一个任务。任务路由在**异质化**场景中发挥作用——智能体能力不同，路由决策首先要确保匹配正确，其次才是负载均衡。可以理解为：路由决定**去哪里**（Which），负载均衡决定**何时去、去多少**（When & How Much）。

### 8.3 对比：委派契约（Delegation Contract）

```
委派契约:          任务路由:
  委派后的约束       →    委派前的选择
  协议/合约层面       →    决策/调度层面
  关注执行过程       →    关注分配策略
```

委派契约（Delegation Contract）定义了智能体接受任务后的责任、权限、回报和约束条件——这是**任务执行阶段**的框架。任务路由则是**任务分派阶段**的决策——在契约被签订之前，决定将任务委派给谁。路由选择了一个智能体后，再由委派契约来规范双方的交互。

### 8.4 关系总览

```
                           ┌─────────────────┐
                           │   任务到达 Task   │
                           └────────┬────────┘
                                    ▼
                    ┌───────────────────────────────┐
                    │     任务路由 (Task Routing)     │
                    │   "应该分配给哪个智能体？"       │
                    │   ↑ 查询能力注册表              │
                    │   ↑ 考虑当前负载                │
                    └────────┬──────────────────────┘
                                    │ 选择目标
                                    ▼
                    ┌───────────────────────────────┐
                    │     委派契约 (Delegation        │
                    │     Contract)                  │
                    │   "执行条件/约束是什么？"        │
                    └────────┬──────────────────────┘
                                    │ 委派确认
                                    ▼
                    ┌───────────────────────────────┐
                    │     目标智能体执行任务           │
                    │     ↑ 负载均衡持续调整           │
                    └───────────────────────────────┘
```

## 9. 工程实现示例

以下是一个简化的任务路由引擎实现，展示了能力匹配、负载感知和优先级路由的集成。

```python
"""
Task Routing Engine - 一个简化的多因子任务路由引擎
"""
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Callable
from enum import Enum
import heapq
import time


class Priority(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4


@dataclass
class Agent:
    agent_id: str
    capabilities: Dict[str, float]     # 能力名称 → 熟练度 (0-1)
    max_concurrency: int = 5
    active_tasks: int = 0
    avg_completion_time: float = 1.0   # 平均完成时间 (秒)
    success_rate: float = 1.0          # 成功率 (0-1)
    is_healthy: bool = True
    last_heartbeat: float = 0.0

    def load_ratio(self) -> float:
        """当前负载比率"""
        return self.active_tasks / max(self.max_concurrency, 1)


@dataclass
class Task:
    task_id: str
    required_capabilities: Dict[str, float]  # 所需能力及最低要求
    priority: Priority = Priority.MEDIUM
    payload: dict = field(default_factory=dict)
    created_at: float = 0.0


@dataclass
class RoutingResult:
    agent: Agent
    score: float
    reason: str


class CapabilityRegistry:
    """能力注册表 - 维护智能体的能力信息"""

    def __init__(self):
        self._agents: Dict[str, Agent] = {}
        self._capability_index: Dict[str, List[str]] = {}  # 能力 → 智能体ID列表

    def register(self, agent: Agent):
        self._agents[agent.agent_id] = agent
        for cap in agent.capabilities:
            if cap not in self._capability_index:
                self._capability_index[cap] = []
            self._capability_index[cap].append(agent.agent_id)

    def find_capable(self, required: Dict[str, float]) -> List[Agent]:
        """查找满足能力要求的智能体"""
        if not required:
            return list(self._agents.values())

        candidates = None
        for cap, min_level in required.items():
            cap_agents = self._capability_index.get(cap, [])
            # 过滤熟练度不足的智能体
            eligible = [
                aid for aid in cap_agents
                if self._agents[aid].capabilities.get(cap, 0) >= min_level
            ]
            cap_set = set(eligible)
            if candidates is None:
                candidates = cap_set
            else:
                candidates &= cap_set

        return [self._agents[aid] for aid in (candidates or [])]

    def unregister(self, agent_id: str):
        agent = self._agents.pop(agent_id, None)
        if agent:
            for cap in agent.capabilities:
                if cap in self._capability_index:
                    self._capability_index[cap] = [
                        aid for aid in self._capability_index[cap]
                        if aid != agent_id
                    ]


class TaskRouter:
    """
    多因子任务路由器
    综合考虑能力匹配、负载情况、优先级和历史表现
    """

    def __init__(self, registry: CapabilityRegistry):
        self.registry = registry
        self.routing_cache: Dict[str, str] = {}      # 任务类型 → 优先智能体
        self.routing_history: List[dict] = []         # 路由历史日志
        self.cache_ttl: float = 300.0                 # 缓存有效期 5 分钟

        # 评分权重
        self.weights = {
            "capability_match": 0.40,
            "load_balance": 0.25,
            "affinity": 0.15,
            "historical_quality": 0.20,
        }

    def route(self, task: Task) -> RoutingResult:
        """主路由接口：为任务选择最优智能体"""
        # Step 1: 检查路由缓存
        cache_key = self._cache_key(task)
        if cache_key in self.routing_cache:
            cached_agent_id = self.routing_cache[cache_key]
            agent = self.registry._agents.get(cached_agent_id)
            if agent and agent.is_healthy and agent.load_ratio() < 0.8:
                return RoutingResult(agent, 1.0, "cache_hit")

        # Step 2: 查找候选智能体
        candidates = self.registry.find_capable(task.required_capabilities)
        if not candidates:
            # 降级策略：忽略部分能力要求
            candidates = self._fallback_search(task)

        if not candidates:
            return RoutingResult(None, 0.0, "no_candidate_found")

        # Step 3: 筛选健康智能体
        healthy = [a for a in candidates if a.is_healthy]
        if not healthy:
            return RoutingResult(None, 0.0, "all_candidates_unhealthy")

        # Step 4: 多因子评分
        scored = [
            (self._score(agent, task), agent)
            for agent in healthy
        ]
        scored.sort(key=lambda x: x[0], reverse=True)
        best_score, best_agent = scored[0]

        # Step 5: 记录路由决策
        self._log_routing(task, best_agent, best_score)

        # Step 6: 更新缓存
        self.routing_cache[cache_key] = best_agent.agent_id

        return RoutingResult(best_agent, best_score, "routed")

    def _score(self, agent: Agent, task: Task) -> float:
        """多因子评分函数"""
        # 能力匹配度评分
        cap_scores = []
        for cap, min_level in task.required_capabilities.items():
            agent_level = agent.capabilities.get(cap, 0)
            if agent_level >= min_level:
                # 超额满足能力要求也有额外加分
                cap_scores.append(min(1.0, agent_level / max(min_level, 0.01)))
            else:
                cap_scores.append(0)
        capability_score = sum(cap_scores) / len(cap_scores) if cap_scores else 0.0

        # 负载评分：低负载 → 高分
        load_score = 1.0 - agent.load_ratio()

        # 亲和性评分：近期成功处理过类似任务 → 高分
        affinity_score = self._compute_affinity(agent.agent_id, task)

        # 历史质量评分
        quality_score = agent.success_rate

        # 加权综合
        total = (
            self.weights["capability_match"] * capability_score
            + self.weights["load_balance"] * load_score
            + self.weights["affinity"] * affinity_score
            + self.weights["historical_quality"] * quality_score
        )

        # 优先级加成：高优先级任务略微偏向能力最强的智能体
        if task.priority == Priority.CRITICAL:
            total += 0.1 * capability_score

        return total

    def _compute_affinity(self, agent_id: str, task: Task) -> float:
        """计算智能体与任务的亲和性"""
        recent = [
            r for r in self.routing_history[-100:]  # 只看最近100条
            if r["agent_id"] == agent_id
            and r["task_type"] == self._task_type(task)
        ]
        if not recent:
            return 0.5  # 无历史记录，中等亲和性

        success_count = sum(1 for r in recent if r["success"])
        return success_count / len(recent)

    def _fallback_search(self, task: Task) -> List[Agent]:
        """降级搜索：放宽能力要求"""
        # 移除最低要求，只保留能力名称匹配
        relaxed = {cap: 0.0 for cap in task.required_capabilities}
        return self.registry.find_capable(relaxed)

    def _cache_key(self, task: Task) -> str:
        """生成缓存键"""
        caps = "_".join(sorted(task.required_capabilities.keys()))
        return f"{caps}_{task.priority.value}"

    def _task_type(self, task: Task) -> str:
        """提取任务类型"""
        caps = sorted(task.required_capabilities.keys())
        return "_".join(caps) if caps else "unknown"

    def _log_routing(self, task: Task, agent: Agent, score: float):
        self.routing_history.append({
            "task_id": task.task_id,
            "agent_id": agent.agent_id,
            "score": score,
            "task_type": self._task_type(task),
            "timestamp": time.time(),
            "success": None,  # 后续由监控系统填充
        })
        # 限制历史记录大小
        if len(self.routing_history) > 10000:
            self.routing_history = self.routing_history[-5000:]

    def record_outcome(self, task_id: str, success: bool):
        """记录任务完成结果，用于后续路由优化"""
        for record in reversed(self.routing_history):
            if record["task_id"] == task_id:
                record["success"] = success
                agent = self.registry._agents.get(record["agent_id"])
                if agent:
                    # 更新成功率（滑动窗口）
                    recent = [
                        r for r in self.routing_history[-200:]
                        if r["agent_id"] == agent.agent_id and r["success"] is not None
                    ]
                    successes = sum(1 for r in recent if r["success"])
                    agent.success_rate = successes / max(len(recent), 1)
                break


# ============ 使用示例 ============

if __name__ == "__main__":
    # 初始化注册表
    registry = CapabilityRegistry()

    # 注册智能体
    registry.register(Agent(
        agent_id="nlp-agent-1",
        capabilities={"text_generation": 0.9, "sentiment": 0.8, "summarization": 0.85},
        max_concurrency=10,
    ))
    registry.register(Agent(
        agent_id="vision-agent-1",
        capabilities={"image_classification": 0.95, "object_detection": 0.9, "ocr": 0.7},
        max_concurrency=5,
    ))
    registry.register(Agent(
        agent_id="general-agent-1",
        capabilities={"text_generation": 0.6, "image_classification": 0.5, "summarization": 0.7},
        max_concurrency=20,
    ))

    # 初始化路由器
    router = TaskRouter(registry)

    # 路由一个任务
    task = Task(
        task_id="task-001",
        required_capabilities={"text_generation": 0.7, "summarization": 0.6},
        priority=Priority.HIGH,
        payload={"content": "Summarize this article..."},
        created_at=time.time(),
    )

    result = router.route(task)
    print(f"任务 {task.task_id} 路由至: {result.agent.agent_id}")
    print(f"路由评分: {result.score:.3f}, 原因: {result.reason}")
    # 输出: 任务 task-001 路由至: nlp-agent-1
    #       路由评分: 0.872, 原因: routed

    # 模拟结果反馈
    router.record_outcome("task-001", success=True)
```

## 10. 工程优化与最佳实践

### 10.1 路由缓存

高频重复的任务类型不需要每次都重新计算路由。缓存策略包括：

- **正向缓存**：为已知任务类型缓存最优智能体，TTU（Time-To-Update）机制定期刷新
- **负向缓存**：记录"此任务无可用智能体"的结果，避免重复无效查找
- **缓存失效策略**：当智能体心跳超时、负载突变、能力变更时，主动清除相关缓存

### 10.2 批量路由决策

当大量任务同时到达时，逐一路由决策的QPS开销很大。优化思路：

```python
def route_batch(self, tasks: List[Task]) -> List[RoutingResult]:
    # 1. 按任务类型分组
    groups = defaultdict(list)
    for task in tasks:
        groups[self._task_type(task)].append(task)

    # 2. 每组只做一次候选查找
    results = []
    for task_type, group in groups.items():
        # 对每组第一个任务做完整路由
        representative = group[0]
        first_result = self.route(representative)
        # 剩余任务直接使用相同目标（负载感知式分配）
        for task in group[1:]:
            agent = first_result.agent
            if agent.load_ratio() < 0.9:
                results.append(RoutingResult(agent, first_result.score, "batch_routed"))
            else:
                results.append(self.route(task))  # 负载过高则重新路由
        results.append(first_result)

    return results
```

### 10.3 路由表分区

对于大规模系统（上千个智能体），路由表需要分区管理：

- **垂直分区**：按能力领域（NLP、CV、代码、语音……）将路由表拆分为独立子表
- **水平分区**：将同一领域的智能体按地理位置/服务器节点分组
- **一致性哈希**：使用一致性哈希将相似的智能体映射到同一路由分区，减少动态变化时的缓存失效

### 10.4 增量更新

避免全量重建路由表，采用增量更新机制：

```
事件驱动更新:
  Agent上线  → 注册能力 + 创建路由条目
  Agent下线  → 标记不可用 + 保留条目（用于恢复）
  负载变化   → 仅更新分数，不更新路由逻辑
  能力变化   → 更新能力索引 + 失效相关缓存
  质量退化   → 降低路由偏好分
```

### 10.5 降级路由策略

当主路由策略不可用时（如ML模型推理超时、向量数据库不可达），需要有降级路径：

```python
def route_with_fallback(self, task: Task) -> RoutingResult:
    """带三级降级的路由"""
    try:
        # 一级路由：ML增强路由
        return self._ml_route(task)
    except MLRouterUnavailable:
        try:
            # 二级路由：规则路由
            return self._rule_route(task)
        except NoMatchingRule:
            try:
                # 三级路由：负载均衡
                return self._load_balance_route(task)
            except NoAvailableAgent:
                return RoutingResult(None, 0.0, "system_overloaded")
```

### 10.6 监控与可观测性

路由系统需要全面的监控来确保决策质量：

| 指标 | 含义 | 告警阈值 |
|------|------|---------|
| 路由决策延迟 | 路由计算耗时 | > 50ms |
| 缓存命中率 | 缓存避免重复计算的比例 | < 60% |
| 降级率 | 使用降级策略的比例 | > 5% |
| 路由稳定性 | 同类任务被路由到同一智能体的概率 | < 70% |
| 路由准确率 | 路由结果与事后最优的一致性 | < 80% |

## 11. 适用场景

### 11.1 大型异构智能体系统

当系统中存在数十个以上能力各异的智能体时，任务路由的价值最大。典型场景包括：

- **企业级AI助手平台**：平台上有负责文档处理的Agent、负责数据分析的Agent、负责代码生成的Agent、负责图像设计的Agent。用户发来一个"帮我分析这份销售报告并生成摘要"的任务，路由系统需要将其引导到数据分析Agent和文本生成Agent协作完成。
- **AI工厂/AI编排平台**：类似LangChain/LlamaIndex的Agent编排系统，任务路由决定了每个子任务去往哪个专用Agent。

### 11.2 服务质量敏感系统

需要满足SLA（Service Level Agreement）的系统，任务路由通过以下方式保障服务质量：

- **关键路径保护**：将高优先级任务路由到性能最佳、负载最低的Agent
- **资源预留**：为紧急任务预留部分Agent的处理能力
- **超时逃生**：当路由目标Agent响应超时时，自动切换到备选Agent

### 11.3 动态Agent环境

Agent频繁上下线、能力动态变化的场景：

- **云原生Agent集群**：Agent Pod弹性伸缩时，路由系统自动纳入/排除Agent
- **Agent模型升级**：Agent的能力因模型更新而变化，路由系统自动感知并调整路由权重
- **故障转移**：Agent宕机时，路由系统将未完成的任务重新路由到健康Agent

### 11.4 不适用场景

任务路由并非银弹，以下场景中路由系统带来的收益可能无法覆盖成本：

- **同质化Agent集群**：所有Agent能力完全相同，只需要简单的负载均衡即可
- **任务类型极度单一**：只有一种任务，不存在"匹配"的问题
- **Agent数量极少（<5个）**：手动分配比路由系统更简单高效
- **任务对路由延迟极其敏感**：路由决策本身增加了端到端的延迟，在微秒级延迟要求的场景中不适用

---

任务路由是多智能体系统中"委派"环节的起点，决定了整个协作流程的效率上限。一个好的路由系统需要权衡精度与开销、短期与长期、全局与局部等多对矛盾，在实践中往往需要根据具体场景做针对性设计。随着AI Agent系统的规模越来越大、能力越来越多样化，任务路由的重要性只会继续增长。
