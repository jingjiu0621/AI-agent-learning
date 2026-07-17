# 8.3.2 能力注册与发现 (Capability Registry)

## 1. 简单介绍

在多智能体系统（Multi-Agent System, MAS）中，当任务需要被分配给合适的 Agent 去执行时，一个根本性的问题随之浮现：**系统如何知道哪个 Agent 能做什么？**

能力注册与发现（Capability Registry）正是为解决这一问题而生的基础设施层。它本质上扮演着多智能体系统中"服务发现"（Service Discovery）的角色——Agent 将自身所能完成的任务类型、所需资源、性能指标等信息注册到注册中心；任务路由器（Task Router）或编排器（Orchestrator）则通过查询注册中心，找到最适合执行某项任务的 Agent。

如果把每个 Agent 看作一座拥有特定功能的微服务，那么 Capability Registry 就是这座微服务集群的 API 网关 + 服务目录。它让系统具备了"感知"能力——不再是硬编码地指定"任务A交给Agent B执行"，而是让系统在运行时动态发现"谁有能力完成任务A，且当前状态可用"。

在现代 AI Agent 系统中，Agent 的规模可能达到成百上千个，它们的能力各异、生命周期不同、负载状况动态变化。没有 Capability Registry，整个系统的可扩展性和灵活性将受到严重制约。

## 2. 基本原理

Capability Registry 的核心运作围绕三个关键概念展开：能力描述（Capability Descriptor）、注册协议（Registration Protocol）和发现协议（Discovery Protocol）。

**（1）能力描述符（Capability Descriptor）**

能力描述符是注册信息的核心单元，通常包含以下维度：

- **能力标识（Capability ID）**：能力的唯一标识，如 `text_summarization`、`code_generation`、`image_analysis`。
- **功能描述（Functional Description）**：结构化的能力定义，包括输入输出格式、依赖条件、约束等。
- **质量指标（Quality Metrics）**：该 Agent 执行此类任务的历史表现，如平均响应时间、成功率、准确率等。
- **当前状态（Current State）**：Agent 是否繁忙、是否在线、剩余容量等实时信息。
- **上下文范围（Context Scope）**：Agent 擅长处理的领域或上下文，如"中文法律文档摘要"而非泛泛的"文本摘要"。
- **版本与兼容性（Version & Compatibility）**：Agent 版本号、所支持的协议版本、运行环境要求等。

**（2）注册协议（Registration Protocol）**

注册协议定义了 Agent 如何向注册中心告知自身能力。典型的注册流程如下：

1. Agent 启动后，向注册中心发送注册请求，附带完整的能力描述符。
2. 注册中心验证能力描述符的合法性（格式检查、去重检查）。
3. 注册中心存储能力信息，分配租约（Lease），返回确认。
4. Agent 在租约有效期内定期发送心跳（Heartbeat）续约。
5. 如果 Agent 下线或能力发生变化，主动发送注销或更新请求。

**（3）发现协议（Discovery Protocol）**

发现协议定义了如何查询可用能力。典型的查询流程如下：

1. 任务编排器提交能力需求（Capability Requirement），描述所需的能力类型、质量最低要求等。
2. 注册中心执行匹配算法，在所有已注册的 Agent 中筛选符合条件的候选者。
3. 返回候选 Agent 列表及其能力匹配度评分。
4. 编排器根据评分和当前负载选择合适的 Agent 执行任务。

**（4）心跳与租约机制（Heartbeat / Lease）**

心跳机制是保证注册信息新鲜度的核心手段。Agent 每隔 T 秒发送心跳包；注册中心若在 N 个心跳周期内未收到心跳，则将该 Agent 标记为"可疑"（Suspect），再过 M 个周期仍未恢复则标记为"离线"（Offline）并从活跃列表中移除。租约（Lease）则是在注册时分配的一个有时间限制的"许可"，Agent 必须通过心跳持续续约。

**注册中心的架构选型**主要有三种：

| 架构 | 一致性 | 可用性 | 适用规模 |
|------|--------|--------|----------|
| **集中式（Centralized）** | 强一致 | 单点风险 | 中小规模 |
| **分布式（Distributed）** | 最终一致 | 高可用 | 大规模 |
| **混合式（Hybrid）** | 可调 | 高可用 | 超大规模 |

集中式注册中心实现简单、查询一致性强，适合几十到几百个 Agent 的系统。分布式注册中心（如基于 Raft 协议的共识集群）则适用于跨地域、超大规模的 Agent 网络。混合式通常采用分层架构——局部注册中心负责本域内 Agent，全局注册中心维护跨域摘要信息。

## 3. 背景与演进

能力注册与发现并非凭空诞生，它经历了从基础设施到智能系统的逐步演进。

**第一阶段：静态配置文件（Static Config）**

在早期的多智能体系统和分布式系统中，Agent 的能力通常通过静态配置文件（YAML、JSON、XML）来定义。系统启动时加载这些配置，运行时不再变更。这种方式的优点是简单直观，缺点同样明显——任何 Agent 的增减或能力变更都需要修改配置文件并重启系统，无法支持动态、弹性变化的 Agent 群体。

**第二阶段：命名服务（Naming Service / DNS）**

DNS 作为互联网最基础的命名服务，将"名称"解析为"地址"。在多智能体系统中，这一阶段对应的是通过名称查找 Agent：给定一个固定的 Agent 名称（如 `translator-agent-01`），系统通过命名服务找到其网络地址。但 DNS 级别的服务发现只能回答问题"这个名称对应的地址是什么"，无法回答"谁能做翻译任务"——它缺乏语义层面的能力描述。

**第三阶段：服务发现（Service Discovery / Consul, Eureka, etcd）**

随着微服务架构的兴起，Consul、Eureka、etcd、ZooKeeper 等服务发现工具成为标配。它们支持服务实例的动态注册与健康检查，但依然停留在"服务名 → 实例列表"的匹配模式。这一阶段比 DNS 的进步在于：支持心跳检测、负载感知和动态上下线。不过，对于多智能体系统而言，"服务名"这种粗粒度的标识仍然不够——两个都注册为 `translation-service` 的 Agent 可能一个擅长中文到英文的技术文档翻译，另一个擅长英文到法文的文学翻译，仅仅通过服务名无法区分。

**第四阶段：语义能力注册中心（Semantic Capability Registry）**

这是当前多智能体系统发展的前沿方向。不再依赖固定的名称或标签，而是通过**语义描述**来表征 Agent 的能力。能力注册信息可以是自然语言描述、结构化能力清单，甚至是经过嵌入（Embedding）后产生的向量表示。查询时也不再是精确匹配标签，而是通过语义相似度计算、推理引擎或向量检索来找到最合适的 Agent。

这一阶段的关键转变在于：从"查找一个我知道名字的服务"转变为"发现一个能解决我问题的服务"。这极大地提升了系统的灵活性，使得 Agent 的动态加入、能力漂移（Capability Drift）和热替换成为可能。

## 4. 之前做法与结果

在多智能体系统的实践中，业界和学术界尝试过多种能力管理方案，各自有其适用的场景和局限性。

**方案一：静态配置文件**

这是最朴素的做法。在系统启动时，通过一个集中的 `agents.yaml` 文件定义每个 Agent 的能力。

```
agents:
  - id: agent-alpha
    capabilities: [text_summarization, text_translation]
    endpoint: http://10.0.0.1:8080
  - id: agent-beta
    capabilities: [image_classification, object_detection]
    endpoint: http://10.0.0.2:8080
```

**结果**：在 Agent 数量少且变动不频繁的场景下足够使用。但一旦 Agent 规模超过几十个，或者需要频繁进行弹性伸缩，手动维护配置变成了运维噩梦。每次变更都需要走代码部署流程，无法做到运行时动态调整。此外，配置文件无法反映 Agent 的实时状态（是否繁忙、是否存活），导致任务分发到已经下线的 Agent。

**方案二：基于服务名的查找（Name-based Lookup）**

引入 Consul、Eureka 等服务注册与发现组件，Agent 启动时向注册中心注册自己的服务名和地址，注册中心通过心跳检测维护可用列表。

**结果**：解决了动态上下线和健康检查的问题，系统具备了一定的自愈能力。但核心缺陷在于——匹配粒度过粗。所有翻译 Agent 都注册为 `translation-service`，系统无法区分"哪个翻译 Agent 更擅长法律文档翻译"、"哪个翻译 Agent 当前负载更低"。对于同质化的微服务集群，名称查找已经足够；但对于能力差异化极大的多智能体系统，这种方案显得力不从心。

**方案三：基于关键字的匹配（Keyword-based Matching）**

在服务名的基础上增加标签（Tag）或关键字体系。Agent 注册时附带一组关键字描述其能力，例如 `domain:legal, language:zh-en, type:translation`。查询时通过关键字匹配筛选目标 Agent。

**结果**：相比纯名称查找有了质的提升，能够实现一定程度上的精细化匹配。但关键字体系存在天然局限：首先，关键字是离散的、二值的——匹配要么命中要么不命中，没有"部分匹配"或"相似程度"的概念，缺乏柔性的排序能力。其次，关键字体系需要人工维护和设计本体（Ontology），当系统引入新型 Agent 时，可能面临关键字体系不够用的问题。最后，关键字无法表达"这个 Agent 擅长处理短文本但长文本表现不佳"这类连续性的、边界模糊的能力特征。

**遗留的共性问题**：

- **描述粒度粗**：无法精细刻画 Agent 的差异化能力，导致匹配精度低。
- **注册信息过时**：Agent 的能力可能随时间变化（例如通过微调提升了某个领域的表现），但注册信息没有随之更新。
- **缺乏质量感知**：无法根据 Agent 的历史执行质量来筛选候选者，单纯靠能力标签匹配可能导致任务分配给实际表现较差的 Agent。
- **信任缺失**：Agent 声称的能力没有验证机制，可能存在夸大或谎报的情况。

## 5. 核心矛盾问题

Capability Registry 在设计与实践中面临着几组根本性的矛盾，这些矛盾构成了该领域的核心研究问题和工程挑战。

**矛盾一：能力描述的丰富度 vs 查询效率**

能力描述越详尽，查询时能做的匹配越精准。但如果每个 Agent 都附带数百维度的能力向量、数万字的能力描述文档，匹配时的计算开销将急剧上升。在需要实时路由决策的场景下（如毫秒级的任务分配），这种开销是不可接受的。

**矛盾二：信息新鲜度 vs 系统开销**

Agent 的状态和能力在不断变化：负载升高、网络波动、通过微调获得了新技能。为了让注册信息保持准确，Agent 需要频繁更新注册状态。但高频的心跳和更新意味着更多的网络通信和注册中心处理压力。如果每个 Agent 每秒发送一次心跳，1000 个 Agent 就意味着注册中心每秒需要处理 1000 次心跳请求。在新鲜度和系统开销之间寻找平衡点是一个持续的设计挑战。

**矛盾三：集中式的一致性 vs 分布式的可用性**

集中式注册中心实现简单、查询结果一致（每次查询都看到相同的数据），但存在单点故障风险和性能瓶颈。分布式注册中心通过共识算法（Raft、Paxos）或 Gossip 协议实现高可用，但引入了最终一致性问题——一个查询可能返回过时的注册信息，恰好错过了一个刚上线的 Agent，或者未感知到刚下线的 Agent。这正是分布式系统中经典的 CAP 理论在 Capability Registry 领域的映射。

**矛盾四：语义灵活性 vs 匹配确定性**

使用自然语言或向量嵌入描述能力带来了极大的灵活性，但语义匹配天然具有不确定性。一个任务描述为"帮我分析这张图片里的情感色彩"，不同的语义匹配引擎可能得出不同的结果，甚至将任务匹配给一个并不真正合适的 Agent。这种不确定性在关键任务场景中可能是致命的。相比之下，传统的标签匹配虽然缺乏灵活性，但行为是完全确定的。

## 6. 当前主流优化方向

面对上述矛盾，业界和学术界提出了多个优化方向。

**方向一：基于嵌入向量的语义能力匹配（Semantic Capability Matching with Embeddings）**

这是当前最受关注的方案之一。将 Agent 的能力描述通过语言模型转化为向量嵌入（Vector Embedding），存储在向量数据库中。当任务需求产生时，同样将任务要求转化为向量，通过向量相似度检索（如余弦相似度、点积）快速找到最匹配的 Agent。

这种方式突破了关键字的离散性限制，能够捕捉"相似但不同"的能力匹配。例如，查询"法律文书摘要"可能匹配到"司法判决书摘要"能力，即使两者没有共享关键字。Facebook 的 FAISS、Milvus、Pinecone 等向量检索库使得大规模向量匹配可以在毫秒级完成。

**方向二：层次化能力分类体系（Hierarchical Capability Taxonomies）**

构建标准化的能力本体，将 Agent 能力组织成树状或图状的分层结构。例如顶层是 `communication`，其下分为 `text_processing` 和 `voice_processing`，`text_processing` 下再细分为 `summarization`、`translation`、`sentiment_analysis` 等。

层次化分类的优势在于支持不同粒度的查询：粗粒度查询可以快速筛选大类，细粒度查询可以在子树内精确匹配。同时，层次结构天然支持能力的继承和泛化关系推理。

**方向三：动态能力推断（Dynamic Capability Inference）**

不再仅仅依赖 Agent 自我声明的能力，而是通过分析 Agent 的执行历史来自动推断其真实能力。例如，如果一个 Agent 在过去 100 次代码审查任务中的平均得分为 4.8/5.0，那么系统可以自动将其 `code_review` 能力评分提升。

动态推断的关键价值在于解决了"自夸"问题——实际能力 vs 宣称能力之间的差距。通过持续观察和机器学习模型，动态推断系统可以给出比 Agent 自我报告更客观的能力评估。

**方向四：预测性能力评分（Predictive Capability Scoring）**

不仅回答"这个 Agent 能不能做这个任务"，还预测"它做得多好"。预测模型基于 Agent 的历史表现、当前负载、任务的复杂度和领域特征，输出一个预期质量评分。路由决策不再基于二值化的能力匹配结果，而是基于连续的预测评分做排序。

**方向五：联邦式注册中心（Federated Registries）**

在超大规模或跨组织的多智能体系统中，单一的注册中心无法满足需求。联邦式架构下，每个域（Domain）维护自己的注册中心，域间通过统一的协议共享摘要信息。查询时，先在本域内查找，找不到或不够用时再向其他域的注册中心发起联合查询。这种方案在隐私保护和可扩展性之间取得了平衡。

**方向六：自描述 Agent（Self-describing Agents）**

Agent 本身携带一个完整的能力清单（Capability Manifest），类似于容器镜像的 Manifest 文件。注册中心负责索引 Manifest，而非存储全部细节。当系统需要深度了解某个 Agent 的能力时，可以从 Manifest URL 拉取完整的描述信息。这种方式降低了注册中心的存储压力，同时让 Agent 能力的定义和管理去中心化。

## 7. 核心优势与能力边界

**核心优势**

- **灵活的异构路由**：Capability Registry 使得系统可以将任务动态分配给任意具备相应能力的 Agent，无论这些 Agent 是由谁开发的、运行在什么环境中。这从根本上解耦了任务定义和任务执行者。
- **支持热插拔（Hot-swapping）**：Agent 可以在运行时动态加入或离开系统，注册中心自动感知并更新路由表，不影响正在运行的任务。这对于需要弹性伸缩或故障自愈的生产系统至关重要。
- **促进系统自组织（Self-organization）**：配合动态能力推断和预测性评分，系统可以自动发现"原来 Agent X 在处理 Y 类型任务上比声明的能力还要出色"，从而持续优化任务分配策略。
- **降低系统耦合**：任务编排器不再需要硬编码每个 Agent 的地址和能力，只需向注册中心表达需求即可。这为系统的进化提供了空间——新类型的 Agent 可以随时加入而不需要修改编排器代码。

**能力边界与局限**

- **短生命周期 Agent 的注册开销**：如果一个 Agent 只存活几秒钟（例如用于执行单个任务的临时 Agent），那么完整的注册-心跳-注销流程可能比任务本身的开销还大。对于此类场景，需要轻量级的注册协议或旁路机制。
- **语义鸿沟（Semantic Gap）**：能力描述与实际能力之间始终存在差距。一个 Agent 可能描述自己"擅长数据分析"，但在实际执行复杂的统计分析任务时表现不佳。这种语义鸿沟无法被注册中心本身消除，需要配合动态推断与反馈机制来弥合。
- **信任与验证问题**：恶意或设计不完善的 Agent 可能夸大声称自己的能力。Capability Registry 本质上基于 Agent 提供的信息做路由决策，如果信息本身不可信，路由质量就无从保证。引入验证机制（如沙盒测试、执行审核、声誉系统）是必要的补充。
- **匹配结果的解释性不足**：当使用深度语义匹配时，系统可能难以解释"为什么选择了 Agent A 而不是 Agent B"。在金融、医疗等需要可解释性的领域，这不被接受。

## 8. 与其他委派机制的区别

在多智能体系统的委派（Delegation）框架内，Capability Registry 与其他几个相关机制有着明确的职责边界和协作关系。

**与任务路由（Task Routing）的区别**

任务路由关注的是"给定一群候选 Agent，如何选择最优的那一个"。这是一个决策问题，考虑因素包括当前负载、网络延迟、历史成功率等。Capability Registry 则是路由的**上游输入来源**——它回答"有哪些候选者"，而路由回答"选哪一个"。

可以这样理解：Registry 提供候选集（Candidate Set），Router 执行选择（Selection）。没有 Registry，Router 无从知晓候选者有哪些；没有 Router，Registry 只知道谁有某项能力，却不知道如何分配任务。两者紧密配合但职责分离。

**与委派合约（Delegation Contract）的区别**

委派合约（或称任务合约、SLA）定义的是 Agent 对于某个特定任务的承诺——包括交付标准、截止时间、奖惩条款等。它回答的是"你愿意为这个任务承诺什么"。

Capability Registry 回答的是"你能做什么"，而 Delegation Contract 回答的是"对于这个具体的任务，你承诺做到什么程度"。Registry 是能力层面的**可能性空间**，Contract 是任务层面的**约束承诺**。在一个典型的委派流程中，先通过 Registry 发现候选 Agent，再通过 Contract 协商确定任务细节。

**与负载均衡（Load Balancing）的区别**

负载均衡关注的是如何在多个可用的工作节点之间合理分配请求，以优化资源利用率和响应时间。它的前提是"这些节点都能做这件事"，关心的是"谁更空闲"。

Capability Registry 关心的恰恰是"哪些节点能做这件事"——它处理的是能力的异构性。当所有 Agent 都具有相同能力时，Registry 的职责退化为简单的服务名发现，负载均衡开始主导。而当 Agent 能力高度异质时，Registry 的筛选作用远大于负载均衡。

**总结：Capability Registry 在多智能体委派架构中承担的是能力信息层（Knowledge Plane），为决策层（任务路由、委派合约协商）和执行层（负载均衡、任务调度）提供基础数据支持。**

## 9. 工程实现示例

以下是一个 Capability Registry 的核心实现骨架。为清晰起见，使用类似 Python 的伪代码，重点展示核心数据结构和流程。

```python
from dataclasses import dataclass, field
from typing import Optional
from enum import Enum
import time
import threading


class AgentStatus(Enum):
    ONLINE = "online"
    BUSY = "busy"
    SUSPECT = "suspect"  # 心跳超时但仍未移除
    OFFLINE = "offline"


@dataclass
class Capability:
    """能力描述"""
    name: str                                # 能力名，如 text_summarization
    domain: Optional[str] = None             # 领域，如 legal、medical
    description: str = ""                    # 自然语言描述
    quality_score: float = 1.0               # 历史质量评分 [0, 1]
    embedding: list[float] = field(default_factory=list)  # 语义向量


@dataclass
class CapabilityRequirement:
    """任务对能力的需求"""
    capability_name: str
    domain: Optional[str] = None
    min_quality: float = 0.5
    description: str = ""
    embedding: list[float] = field(default_factory=list)


@dataclass
class AgentInfo:
    """Agent 在注册中心的信息"""
    agent_id: str
    endpoint: str
    capabilities: list[Capability]
    status: AgentStatus = AgentStatus.ONLINE
    last_heartbeat: float = 0.0
    lease_ttl: float = 30.0  # 租约有效期（秒）
    current_load: float = 0.0  # 当前负载 [0, 1]


class CapabilityRegistry:
    """能力注册中心"""

    def __init__(self, heartbeat_timeout: float = 10.0):
        self._agents: dict[str, AgentInfo] = {}
        self._heartbeat_timeout = heartbeat_timeout
        self._lock = threading.RLock()
        # 启动后台心跳检查线程
        self._evictor = threading.Thread(target=self._evict_loop, daemon=True)
        self._evictor.start()

    def register(
        self,
        agent_id: str,
        endpoint: str,
        capabilities: list[Capability],
        lease_ttl: float = 30.0,
    ) -> bool:
        """Agent 注册自身能力"""
        with self._lock:
            if agent_id in self._agents:
                # 已存在则视为更新
                self._agents[agent_id].capabilities = capabilities
                self._agents[agent_id].endpoint = endpoint
                self._agents[agent_id].status = AgentStatus.ONLINE
            else:
                self._agents[agent_id] = AgentInfo(
                    agent_id=agent_id,
                    endpoint=endpoint,
                    capabilities=capabilities,
                    status=AgentStatus.ONLINE,
                    lease_ttl=lease_ttl,
                )
            self._agents[agent_id].last_heartbeat = time.time()
            return True

    def unregister(self, agent_id: str) -> bool:
        """Agent 主动注销"""
        with self._lock:
            if agent_id in self._agents:
                del self._agents[agent_id]
                return True
            return False

    def heartbeat(self, agent_id: str) -> bool:
        """Agent 发送心跳续约"""
        with self._lock:
            agent = self._agents.get(agent_id)
            if agent is None:
                return False
            agent.last_heartbeat = time.time()
            agent.status = AgentStatus.ONLINE
            return True

    def find_capable(
        self, requirement: CapabilityRequirement, top_k: int = 5
    ) -> list[tuple[str, float]]:
        """查找具备某项能力的 Agent，返回 (agent_id, score) 列表"""
        candidates: list[tuple[str, float]] = []
        with self._lock:
            for agent_id, info in self._agents.items():
                if info.status in (AgentStatus.OFFLINE, AgentStatus.SUSPECT):
                    continue
                for cap in info.capabilities:
                    match_score = self._match(cap, requirement)
                    if match_score > 0:
                        candidates.append((agent_id, match_score))
                        break  # 一个 Agent 只算一次
        # 按匹配度降序排列
        candidates.sort(key=lambda x: x[1], reverse=True)
        return candidates[:top_k]

    def _match(
        self, capability: Capability, requirement: CapabilityRequirement
    ) -> float:
        """计算能力与需求之间的匹配度 (0 ~ 1)"""
        score = 0.0

        # 1. 名称匹配合
        if capability.name == requirement.capability_name:
            score += 0.5

        # 2. 领域匹配（可选）
        if requirement.domain and capability.domain == requirement.domain:
            score += 0.3

        # 3. 质量门槛
        if capability.quality_score < requirement.min_quality:
            return 0.0

        # 4. 语义相似度（使用向量嵌入）
        if capability.embedding and requirement.embedding:
            semantic_sim = self._cosine_similarity(
                capability.embedding, requirement.embedding
            )
            score += 0.2 * semantic_sim

        return min(score, 1.0)

    @staticmethod
    def _cosine_similarity(a: list[float], b: list[float]) -> float:
        """余弦相似度"""
        dot = sum(x * y for x, y in zip(a, b))
        norm_a = sum(x * x for x in a) ** 0.5
        norm_b = sum(y * y for y in b) ** 0.5
        return dot / (norm_a * norm_b) if norm_a > 0 and norm_b > 0 else 0.0

    def _evict_loop(self):
        """后台线程：定期检查心跳超时，驱逐过期 Agent"""
        while True:
            time.sleep(self._heartbeat_timeout / 3)
            now = time.time()
            with self._lock:
                to_remove = []
                for agent_id, info in self._agents.items():
                    elapsed = now - info.last_heartbeat
                    if elapsed > info.lease_ttl * 2:
                        to_remove.append(agent_id)
                    elif elapsed > info.lease_ttl:
                        info.status = AgentStatus.SUSPECT
                for agent_id in to_remove:
                    print(f"[Registry] Evicting {agent_id}: heartbeat timeout")
                    del self._agents[agent_id]

    def get_status(self, agent_id: str) -> Optional[AgentStatus]:
        """查询 Agent 当前状态"""
        with self._lock:
            agent = self._agents.get(agent_id)
            return agent.status if agent else None
```

**核心流程说明**：

1. **注册（register）**：Agent 在启动时调用 `register`，提交自身的能力列表。注册中心为 Agent 建立记录并初始化心跳计时。
2. **心跳（heartbeat）**：Agent 定期调用 `heartbeat` 续约。后台的 `_evict_loop` 线程持续检查心跳超时，超时未续约的 Agent 先被标记为 SUSPECT，最终被驱逐。
3. **发现（find_capable）**：任务编排器提交 `CapabilityRequirement`，注册中心在在线 Agent 中匹配能力，返回按匹配度排序的候选列表。
4. **匹配（_match）**：示例中实现了名称匹配、领域匹配、质量门控和语义相似度匹配的组合评分策略，可以作为更复杂匹配逻辑的起点。

## 10. 适用场景

Capability Registry 并非所有多智能体系统的必需品。在以下场景中，它的价值最为突出：

**最适合的场景**

- **大规模异构 Agent 系统**：当系统包含数十个以上、能力各异的 Agent 时，手动管理路由规则变为不可能，Capability Registry 成为必需品。例如，一个企业自动化平台同时使用 NLP Agent、OCR Agent、数据分析 Agent、工作流编排 Agent 等。
- **动态环境**：Agent 频繁地加入、离开或升级。例如，一个开放市场的 Agent 集市（Agent Marketplace），第三方开发者可以随时发布新的 Agent 供系统调用。注册中心在此承担了 Agent 目录的角色。
- **弹性伸缩场景**：Agent 的副本数量根据负载动态调整。注册中心使得系统可以自动感知新增或缩容的 Agent 实例，并将任务路由到可用实例上。
- **需要灵活任务分配的系统**：同一个任务可能有多个 Agent 声称具备执行能力，系统需要根据质量、负载、领域等维度做出最优选择。Capability Registry 提供的丰富信息是理性选择的基础。

**不太适合的场景**

- **极少量固定 Agent**：如果系统中只有 2-3 个长期运行的 Agent，直接硬编码路由规则比引入注册中心更简单高效。
- **实时性要求极高的场景**：如果任务分发需要亚毫秒级的延迟，经过注册中心的查询步骤可能成为瓶颈。这类场景通常采用预计算的路由表或直连方式。
- **全同质 Agent 集群**：如果所有 Agent 能力完全相同（如一个由 100 个相同 LLM 实例组成的推理集群），Capability Registry 退化为基本的健康检查和负载均衡，使用简单的服务发现组件即可。

**总结**

Capability Registry 是多智能体系统从"玩具"走向"生产级"的关键基础设施之一。它让系统具备了感知和适应能力变化的灵活性，是构建大规模、可进化、自组织 Agent 系统的基石。理解其原理、局限和最佳实践，对于任何从事 AI Agent 系统开发的工程师来说都是必不可少的一课。
