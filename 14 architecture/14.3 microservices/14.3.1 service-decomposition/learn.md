# 14.3.1 服务拆分原则与方法 — Service Decomposition for Agent Systems

> **核心问题**：如何将 Agent 系统拆分为合理的微服务集合？服务拆分是微服务化 Agent 的第一步，也是最关键、最不可逆的一步——一旦拆分错误，后续的服务间通信、部署、运维都会受到永久性影响。Agent 系统的服务拆分比传统微服务拆分更复杂，因为 Agent 的工作流带有**循环回溯**（推理→工具→推理→工具）和**大状态传递**（上下文/记忆），拆分会直接影响推理效率和 Token 成本。

---

## 1. 基本原理

### 1.1 什么是 Agent 系统的服务拆分

服务拆分（Service Decomposition）是将一个完整的 Agent 系统分解为多个独立部署、独立运行、独立维护的微服务的过程。每个微服务封装 Agent 的某一类能力或某一领域知识，通过定义良好的接口与其他服务通信。

```
服务拆分的输入与输出：

输入：
  一个完整的 Agent 系统（单体或模块化单体）
  ├── 功能清单 (推理 / 规划 / 工具 / 记忆 / 验证)
  ├── 领域模型 (业务实体和业务规则)
  ├── 非功能需求 (延迟 / 吞吐量 / 可用性)
  └── 组织架构 (团队结构 / 责任划分)

输出：
  一组微服务定义
  ├── 每个服务的职责说明
  ├── 每个服务的 API 契约 (接口 / 数据格式 / 协议)
  ├── 服务间的依赖关系图
  └── 数据所有权矩阵 (每个数据项属于哪个服务)
```

### 1.2 与传统的微服务拆分有何不同？

```
传统微服务拆分                          Agent 系统服务拆分
─────────────────                      ────────────────────

拆分依据：业务能力                     拆分依据：Agent 能力 + 资源 + 领域
  订单服务、用户服务、库存服务              推理服务、工具服务、记忆服务

数据边界清晰                           数据边界模糊
  每个服务有独立的数据库                    记忆数据可能被多个服务引用
  CRUD 式数据操作                       推理+工具+验证共享相同的中间数据

通信模式简单                           通信模式复杂
  请求-响应为主                           需要流式传输推理 Token
  偶尔异步事件                           需要传递大上下文窗口

拆分后的调用链                         拆分后的调用链
  A → B → C (线性，无环)                  编排器 → 推理 → 工具 → 推理 → 验证
                                         (有环，循环回溯)

状态管理                               状态管理
  无状态优先，状态外存                     推理无状态，编排有状态，记忆有状态
  Redis/DB 管理会话                      需要分布式上下文管理

核心差异总结：
  传统微服务拆分解决的是"业务能力"的边界问题
  Agent 微服务拆分需要同时解决三个边界问题：
    1. 能力边界 (什么功能)
    2. 资源边界 (用什么硬件)
    3. 数据边界 (谁拥有什么数据)
```

---

## 2. 背景与演进

### 2.1 微服务化前的单体 Agent

在微服务化之前，Agent 系统通常以单体形式运行。下面是单体 Agent 内部的典型结构：

```
单体 Agent 的内部结构:

┌────────────────────────────────────────────────────────────┐
│                    Agent 进程 (单进程)                       │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │             Agent 主循环 (Main Loop)                  │  │
│  │                                                      │  │
│  │  1. 接收输入                                         │  │
│  │  2. 调用 LLM 推理 (同步等待)                          │  │
│  │  3. 解析 LLM 输出 → 决定行动                          │  │
│  │  4. 执行工具调用 (同步等待)                            │  │
│  │  5. 更新记忆 (写入进程内存)                            │  │
│  │  6. 回到步骤 2                                       │  │
│  │  7. 生成最终响应                                      │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────┐   │
│  │ LLM Client  │  │ Tool Exec  │  │ Memory Manager     │   │
│  │ (OpenAI SDK)│  │ (函数调用)   │  │ (Python dict/List) │   │
│  └────────────┘  └────────────┘  └────────────────────┘   │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 共享内存状态:                                          │  │
│  │   conversation_history: List[Dict]                    │  │
│  │   tool_results: Dict[str, Any]                        │  │
│  │   agent_state: Dict[str, Any]  ← 所有组件都可读写      │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### 2.2 单体 Agent 的核心矛盾

```
随着系统增长，单体 Agent 暴露出 4 对核心矛盾：

矛盾 1: 资源异构 vs 统一部署
  ┌─ 推理需要 GPU 或 LLM API 配额 (吞吐量由 RPM/TPM 限制)
  ├─ 工具执行需要 CPU 和内存 (受 CPU 时钟和内存带宽限制)
  ├─ 代码执行需要强隔离沙箱 (Docker 容器或 Firecracker 微 VM)
  └─ 记忆检索需要 I/O 吞吐量 (向量数据库索引扫描)
  在单体中：所有组件共享同一进程资源 → 无法独立优化

矛盾 2: 不同生命周期 vs 统一发布
  ┌─ LLM 模型升级 (GPT-4 → GPT-5) → 需要更新推理代码
  ├─ 工具版本更新 (API 升级) → 需要更新工具适配器
  ├─ 记忆存储方案变更 (向量库切换) → 需要更新记忆模块
  └─ 在单体中：更新一个模块 = 重新部署整个 Agent → 发布频率受限于最不稳定的模块

矛盾 3: 不同扩展需求 vs 整体副本
  ┌─ 白天用户多 → 推理服务需要更多副本 (LLM API 并发)
  ├─ 夜间批处理 → 工具服务需要更多副本 (批量数据处理)
  ├─ 单体中：扩展推理能力 = 扩展整个 Agent → CPU/GPU/内存一起增加
  └─ 结果：要么过度配置 (浪费成本) 要么不足配置 (瓶颈)

矛盾 4: 不同故障容忍度 vs 统一故障域
  ┌─ 工具执行失败 → 可接受 (重试或降级)
  ├─ 记忆查询超时 → 可接受 (退回到短期上下文)
  ├─ 推理服务失败 → 严重 (但 LLM API 通常有重试机制)
  └─ 单体中：任何一个组件 OOM → 整个进程崩溃 → 所有用户连接断开
```

### 2.3 演进路径

```
单体 Agent 到微服务 Agent 的典型演进：

阶段 0: 纯单体
  [Agent 进程 ─── LLM API (外部)]
  推理/工具/记忆全部在进程内

阶段 1: 工具外置
  [Agent 进程] ──HTTP──► [Tool Execution Service]
  将执行耗时操作的工具 (代码执行、浏览器) 独立

阶段 2: 记忆外置
  [Agent 进程] ──gRPC──► [Memory Service] (Vector DB)
  将知识检索和记忆管理独立

阶段 3: 推理独立
  [Orchestrator] ──gRPC──► [Reasoning Service] (LLM Proxy)
  编排器协调推理和工具的交互

阶段 4: 全拆分
  [API Gateway] → [Orchestrator] → [Reasoner] → [Tool Exec]
                                         → [Planner] → [Validator]
                                         → [Memory]  → [Knowledge]

每个阶段的触发条件：
  阶段 0 → 1: 工具执行开始阻塞推理循环 (工具延迟 > LLM 延迟)
  阶段 1 → 2: 记忆数据量 > 内存限制，需要独立扩展
  阶段 2 → 3: 需要多模型支持 (不同任务用不同 LLM)
  阶段 3 → 4: 团队 > 5 人，需要自治开发
```

---

## 3. 服务拆分维度

### 3.1 按能力拆分 (Capability-Based Decomposition)

这是最直观的拆分方式，将 Agent 的核心能力函数拆分为独立服务。

```
Agent 能力 → 对应的微服务：

┌────────────────────────────────────────────────────────────────┐
│ Agent 能力           │ 对应服务           │ 职责               │
├────────────────────────────────────────────────────────────────┤
│ 推理 (Reasoning)      │ 推理服务           │ LLM 调用、思维链生成 │
│ 记忆 (Memory)         │ 记忆服务           │ 状态存储、向量检索   │
│ 工具 (Tool Use)       │ 工具执行服务        │ 工具注册、调用、结果  │
│ 规划 (Planning)       │ 规划服务           │ 任务分解、策略选择   │
│ 验证 (Validation)     │ 验证服务           │ 输出质量、安全检查   │
│ 知识 (Knowledge)      │ 知识服务           │ RAG 检索、文档理解   │
│ 编排 (Orchestration)  │ 编排服务           │ 工作流调度、状态追踪  │
│ 交互 (Interaction)    │ API 网关           │ 用户接口、协议转换   │
└────────────────────────────────────────────────────────────────┘

能力拆分的原则：
  1. 高内聚：每个服务包含完整的一个能力维度
  2. 低耦合：服务间通过数据契约通信，不共享实现
  3. 单一职责：一个服务改变的原因只有一个
```

### 3.2 按业务领域拆分 (Domain-Based Decomposition)

对于业务复杂的 Agent 系统（如企业级客服 Agent、金融 Agent），需要按业务领域拆分。

```
业务领域拆分示例 — 企业客服 Agent:

┌──────────────────────────────────────────────────────────┐
│                  客服 Agent 平台                           │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐  │
│  │ 订单客服领域   │  │ 售后客服领域   │  │ 技术客服领域       │  │
│  │             │  │             │  │                  │  │
│  │ • 订单查询    │  │ • 退换货处理   │  │ • 产品使用指南     │  │
│  │ • 物流追踪    │  │ • 退款流程    │  │ • 故障排查        │  │
│  │ • 支付问题    │  │ • 投诉处理    │  │ • API 文档        │  │
│  │ • 发票管理    │  │ • 赔偿评估    │  │ • 版本兼容性      │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                 │                    │           │
│         └─────────────────┼────────────────────┘           │
│                           │                                │
│                    ┌──────▼───────┐                        │
│                    │  共享能力层     │                        │
│                    │  • 推理(共享LLM)│                        │
│                    │  • 知识库      │                        │
│                    │  • 用户画像    │                        │
│                    └──────────────┘                        │
└──────────────────────────────────────────────────────────┘

领域拆分 vs 能力拆分：
  能力拆分关注"Agent 怎么工作"
  领域拆分关注"Agent 处理什么业务"
  
  实践中：先用能力拆分得到基础服务 (推理/工具/记忆)
           再按业务领域为每个领域部署一套独立的能力服务实例
```

### 3.3 按数据主权拆分 (Data Sovereignty Decomposition)

数据主权拆分关注的是"谁拥有什么数据"。在 Agent 系统中，不同服务管理不同类型的数据，这些数据的访问模式、一致性要求和安全级别都不同。

```
Agent 系统数据分类与对应的服务：

数据类型             拥有服务        存储             访问模式        一致性要求
──────────────────────────────────────────────────────────────────────────
会话上下文           编排服务        Redis             频繁读写       强一致
长期记忆             记忆服务        Vector DB         频繁读、低频写   最终一致
工具执行历史          工具服务        DB/对象存储        低频读、低频写   强一致
LLM 推理日志         推理服务        Elasticsearch     几乎不读         允许丢失
用户 PII             认证/用户服务   DB (加密)          低频读写         强一致
业务数据 (订单/库存)  业务领域服务    业务 DB            频繁读写         强一致

数据主权原则：
  每个数据项有且仅有一个"所有者"服务
  其他服务通过所有者服务的 API 访问数据
  不允许其他服务直接操作所有者服务的数据存储

示例：错误的数据主权设计
  [工具服务] ──直接SQL──► [记忆服务的 PostgreSQL]
  问题：工具服务跳过记忆服务的 API 直接操作数据库
        改变了数据结构，记忆服务不知情 → 数据不一致

示例：正确的数据主权设计
  [工具服务] ──POST /memory/store──► [记忆服务 API]
  记忆服务内部实现数据存储，对外屏蔽实现细节
```

### 3.4 按资源类型拆分 (Resource-Based Decomposition)

这是 Agent 系统特有的拆分维度。不同 Agent 能力对计算资源的需求差异巨大，按资源类型拆分可以实现精细化的资源分配和成本控制。

```
Agent 系统资源需求映射：

服务                计算需求           内存需求         I/O 需求        典型硬件
─────────────────────────────────────────────────────────────────────────────
推理服务             GPU 计算 / API     高 (大上下文)    低               GPU 实例
工具执行 (代码)      CPU 密集          中高            低               CPU 优化实例
工具执行 (API)       网络 I/O 密集     低              高              标准实例
记忆检索 (向量)       CPU 计算         高 (索引)        高 (磁盘 I/O)   高内存 + SSD
记忆存储 (KV)        低                中              高 (网络 I/O)   标准 + Redis
规划服务             低                低              低               标准实例
验证服务             API 调用          中              低               标准实例
知识图谱             CPU 密集          高              中               CPU 优化

资源隔离的实际效果：

  不隔离 (单体):
  [Agent 进程]
    LLM 推理: 消耗 GPU 内存 16GB ← 虚拟内存不足 → 进程 OOM
    代码执行: 消耗 CPU 100% ← 推理请求被抢占 → 推理延迟飙升
    向量检索: 消耗内存 8GB  ← 内存不足 → 系统开始 swap

  隔离后 (微服务):
  [推理服务]   GPU: 16GB, CPU: 2核  ← 独立资源保证
  [工具服务]   CPU: 8核,  Mem: 4GB   ← 独立资源保证
  [记忆服务]   CPU: 4核,  Mem: 32GB  ← 独立资源保证

资源拆分原则：
  1. 资源需求差异 > 2x 的服务应该拆分
  2. 资源瓶颈方向不同的服务应该拆分
  3. 共享资源的竞争会导致双方性能下降 → 拆分
```

---

## 4. 拆分策略详解

### 4.1 DDD (领域驱动设计) 在 Agent 系统的应用

领域驱动设计为服务拆分提供了系统化的方法论。在 Agent 系统中，DDD 的概念映射如下：

```
DDD 概念             在 Agent 系统中的映射
────────────────────────────────────────────
领域 (Domain)         Agent 所服务的业务领域 (客服/编程/金融)
子域 (Subdomain)      Agent 的某个能力域 (推理/工具/记忆)
限界上下文 (Bounded    服务的边界：一个限界上下文 = 一个微服务
  Context)
通用语言 (Ubiquitous   Agent 领域术语：Thought、Action、Observation
  Language)           Tool Call、Context Window、Token Budget
聚合 (Aggregate)       一组相关操作的组合：一个推理步骤、一个工具调用
实体 (Entity)          有唯一标识的对象：Agent 会话、任务实例
值对象 (Value Object)  无标识的对象：模型配置、工具参数
领域事件 (Domain        Agent 内部的重要事件：推理完成、工具执行完成、
  Event)              验证失败、任务完成

Agent 中的限界上下文识别：

  ┌──────────────────┐    ┌──────────────────┐
  │ 推理上下文          │    │ 工具执行上下文       │
  │                    │    │                   │
  │ 通用语言:          │    │ 通用语言:          │
  │ 模型、Prompt、Token │    │ 工具、参数、结果    │
  │ Chain、Thought     │    │ Timeout、Retry    │
  │ 聚合: 推理步骤      │    │ 聚合: 工具调用      │
  │ 实体: 会话          │    │ 实体: 工具实例      │
  └────────┬──────────┘    └────────┬──────────┘
           │                       │
           │    ┌──────────────────┘
           ▼    ▼
  ┌──────────────────┐
  │ 记忆上下文          │
  │                   │
  │ 通用语言:          │
  │ 短期、长期、向量    │
  │ 检索、存储、整合    │
  │ 聚合: 记忆条目      │
  │ 实体: 会话、知识    │
  └──────────────────┘
```

### 4.2 限界上下文 (Bounded Context) 的识别

在 Agent 系统中，限界上下文通过分析**组件间的交互频率**和**数据共享模式**来识别。

```
限界上下文识别矩阵：

                   推理      规划      工具      记忆      验证
    ──────────────────────────────────────────────────────────
    推理              -     高频      中频     高频      中频
    规划             高频    -        低频     低频      低频
    工具             中频   低频      -        中频      中频
    记忆             高频   低频      中频      -        低频
    验证             中频   低频      中频     低频      -

   注释：
     高频 (>10 次/任务) → 强关联
     中频 (2-10 次/任务) → 弱关联
     低频 (<2 次/任务) → 独立

   根据矩阵，可能的上下文边界：
     • 推理 + 规划 + 验证 → 考虑合并吗？ 不，它们是不同的能力维度
     • 推理 + 记忆 → 高频交互 → 需要优化通信，但不应合并 (资源异构)
     • 规划 + 工具 → 低频交互 → 可以作为独立上下文
```

### 4.3 六种启发式拆分策略

策略 1: 动词拆分 (Verb Decomposition) — 按能力动词拆分

```
识别 Agent 执行流程中的核心动词，每个动词对应一个微服务：

  [用户输入] → 理解 → 推理 → 规划 → 工具调用 → 验证 → 响应
  
  动词识别：
    理解 (Comprehend)  → 推理服务 (/reason)
    推理 (Reason)      → 推理服务 (/reason)
    规划 (Plan)        → 规划服务 (/plan)
    执行 (Execute)     → 工具服务 (/execute)
    验证 (Validate)    → 验证服务 (/validate)
    响应 (Respond)     → 推理服务 (/finalize)

  适用场景：Agent 能力边界清晰，动词明确
  优点：直观，容易理解
  缺点：可能忽略业务领域边界
```

策略 2: 名词拆分 (Noun Decomposition) — 按业务实体拆分

```
识别 Agent 处理的核心业务实体，围绕实体组织服务：

  客服 Agent 的名词：
    订单、用户、商品、退换货、投诉

  拆分后的服务：
    Order Agent Service — 处理订单相关查询和操作
    User Agent Service — 处理用户信息和个人化
    Return Agent Service — 处理退换货流程
    Complaint Agent Service — 处理投诉和升级

  适用场景：业务实体明确，不同实体处理流程差异大
  优点：与业务对齐，团队分工清晰
  缺点：可能忽略 Agent 的统一推理能力
```

策略 3: 资源隔离拆分 (Resource Isolation Decomposition)

```
计算资源的"热点"分析，将资源密集型操作独立：

  资源热点监控：
    服务           CPU%    内存%    I/O%    GPU%
    推理服务        20%     80%     5%      95%     ← GPU 瓶颈
    代码执行        95%     30%     10%     0%      ← CPU 瓶颈
    知识检索        40%     70%     90%     0%      ← I/O 瓶颈
    会话管理        10%     20%     30%     0%      ← 无明显瓶颈

  拆分决策：
    GPU 密集 → 推理服务独立 (GPU 实例)
    CPU 密集 → 代码执行独立 (CPU 优化实例)
    I/O 密集 → 知识检索独立 (高 IOPS 实例)
    无明显瓶颈 → 编排服务保持轻量 (共享或最小副本)

  适用场景：资源瓶颈明确，不同组件资源需求差异大
  优点：极致的成本优化
  缺点：需要精细的监控数据
```

策略 4: 变更频率拆分 (Change Frequency Decomposition)

```
分析不同组件的变更频率，将变更高频和低频的组件分离：

  变更频率分析：
    模块             变更频率        常见变更原因
    ──────────────────────────────────────────────
    LLM 模型适配器    每月 1-2 次    模型更新、供应商切换
    Prompt 模板      每周 3-5 次    提示词优化、A/B 测试
    工具定义          每周 2-3 次    新增/修改工具
    工具执行逻辑      每月 1 次      Bug 修复、性能优化
    记忆存储方案      每季度 1 次    存储层升级、迁移
    业务规则          每日多次       业务需求变更

  拆分建议：
    高频变更 → 独立服务 (工具服务、推理 Prompt 服务)
    低频变更 → 可以保持稳定 (记忆服务、基础架构)
    规则：变更频率差异 > 5x → 强烈建议拆分

  适用场景：业务需求变化快，团队需要频繁发布
  优点：减少部署冲突，降低回归测试范围
  缺点：可能增加服务数量
```

策略 5: 团队结构拆分 (Team Structure Decomposition — Conway 定律)

```
Conway 定律：系统架构复制组织的沟通结构

  组织架构：
    ┌─── 推理团队 (3人) ─── 负责 LLM 推理优化
    ├─── 工具团队 (5人) ─── 负责工具接入和维护
    ├─── 记忆团队 (2人) ─── 负责记忆和知识管理
    └─── 平台团队 (4人) ─── 负责编排、部署和可观测性

  对应的服务拆分：
    推理团队 → 推理服务
    工具团队 → 工具执行服务
    记忆团队 → 记忆服务 + 知识检索服务
    平台团队 → 编排服务 + API 网关 + 可观测性基础设施

  注意：团队结构拆分应该建立在能力/资源拆分的基础上
        不能"为了拆分而拆分"

  组织约束：
    • 两个团队间的沟通成本 = 服务间 API 的设计质量
    • 一个服务不要由多个团队共同维护 (所有权模糊)
    • 一个团队可以维护多个服务 (如果它们高度相关)
```

策略 6: 故障域拆分 (Failure Domain Decomposition)

```
分析不同组件的故障影响范围，将高风险组件隔离：

  故障风险评估：
    组件             故障概率   影响范围    恢复难度    隔离优先级
    ────────────────────────────────────────────────────────────
    代码执行器        高 (用户代码) 整个 Agent  中         最高
    LLM API 调用      中 (限流)   全部推理     高          高
    向量数据库查询     中 (超时)   知识检索     低          中
    会话状态管理       低          单个会话     低          低

  拆分规则：
    故障概率高 + 影响范围大 → 必须独立服务 (代码执行器)
    故障概率中 + 恢复困难 → 独立服务 + 降级策略 (推理服务)
    故障概率低 + 影响小 → 可以与其他服务合并

  适用场景：系统可靠性和可用性要求高 (金融/医疗)
  优点：精确的故障隔离
  缺点：可能增加服务数量和复杂度
```

### 4.4 常用拆分策略对比表

```
策略              核心依据       适用场景          优点               缺点
─────────────────────────────────────────────────────────────────────────
动词拆分          能力动词       能力边界清晰       直观易懂            忽略业务边界
名词拆分          业务实体       业务差异大         业务对齐		    忽略推理统一性
资源隔离拆分      资源类型       资源异构明显       极致成本优化        需要精细监控
变更频率拆分      变更速度       需求变化快         减少部署冲突        增加服务数
团队结构拆分      团队边界       大型团队           组织对齐            可能过度拆分
故障域拆分        故障影响       高可用要求         精确隔离            复杂度过高

使用建议：
  1. 首选动词拆分 + 资源隔离拆分的组合
  2. 在业务复杂时引入名词拆分
  3. 在团队扩大时考虑团队结构拆分
  4. 在高可用场景应用故障域拆分
  5. 在迭代频繁时使用变更频率拆分
  6. 所有拆分策略都要结合限界上下文分析
```

---

## 5. 代码示例

### 5.1 服务边界识别示例

```python
# service_boundary_analysis.py
"""
服务边界识别工具 — 分析组件间的交互模式，辅助服务拆分决策。

使用方法：
  python service_boundary_analysis.py
  
输入：组件依赖关系定义
输出：建议的服务拆分方案
"""

from dataclasses import dataclass, field
from typing import Dict, List, Set, Tuple
from enum import Enum

# ============ 数据模型 ============

class InteractionType(Enum):
    """组件间的交互类型"""
    SYNC_CALL = "sync_call"        # 同步调用 (强依赖)
    ASYNC_EVENT = "async_event"    # 异步事件 (松耦合)
    SHARED_DATA = "shared_data"    # 共享数据 (强耦合)
    NO_INTERACTION = "none"        # 无交互


@dataclass
class Component:
    """Agent 系统中的一个组件"""
    name: str
    description: str
    resource_type: str         # cpu | gpu | memory | io
    change_frequency: str      # high | medium | low
    failure_impact: str        # high | medium | low


@dataclass
class Interaction:
    """两个组件之间的交互关系"""
    source: str
    target: str
    interaction_type: InteractionType
    frequency: int             # 每次任务中的交互次数
    data_volume: str           # small | medium | large (KB/MB)
    latency_sensitive: bool    # 是否延迟敏感


class ServiceBoundaryAnalyzer:
    """
    服务边界分析器
    
    分析组件间的交互模式，根据以下规则推荐服务边界：
    1. 高频同步交互 (>5次/任务) → 考虑合并
    2. 大规模数据传输 → 考虑合并 (避免网络开销)
    3. 不同资源类型 → 考虑拆分
    4. 不同变更频率 → 考虑拆分
    5. 不同故障影响 → 考虑拆分
    """
    
    def __init__(self):
        self.components: Dict[str, Component] = {}
        self.interactions: List[Interaction] = []
    
    def add_component(self, component: Component):
        self.components[component.name] = component
    
    def add_interaction(self, interaction: Interaction):
        self.interactions.append(interaction)
    
    def calculate_coupling_score(self, comp_a: str, comp_b: str) -> float:
        """
        计算两个组件间的耦合度分数 (0-1)
        越高表示越应该合并为一个服务
        """
        score = 0.0
        factors = []
        
        for interaction in self.interactions:
            if {interaction.source, interaction.target} == {comp_a, comp_b}:
                # 交互类型权重
                type_weights = {
                    InteractionType.SYNC_CALL: 0.4,
                    InteractionType.SHARED_DATA: 0.5,
                    InteractionType.ASYNC_EVENT: 0.2,
                    InteractionType.NO_INTERACTION: 0.0,
                }
                score += type_weights[interaction.interaction_type]
                factors.append(f"交互类型: {type_weights[interaction.interaction_type]}")
                
                # 频率权重
                freq_weight = min(interaction.frequency / 10, 0.3)
                score += freq_weight
                factors.append(f"交互频率: {freq_weight}")
                
                # 数据量权重
                data_weights = {"small": 0.0, "medium": 0.1, "large": 0.2}
                score += data_weights[interaction.data_volume]
                factors.append(f"数据量: {data_weights[interaction.data_volume]}")
                
                # 延迟敏感度
                if interaction.latency_sensitive:
                    score += 0.1
                    factors.append("延迟敏感: 0.1")
        
        # 资源类型差异 (不同资源类型 → 降低耦合分数)
        comp_a_res = self.components[comp_a].resource_type
        comp_b_res = self.components[comp_b].resource_type
        if comp_a_res != comp_b_res:
            score *= 0.7  # 资源不同，减少30%的合并倾向
            factors.append(f"资源不同({comp_a_res} vs {comp_b_res}): ×0.7")
        
        # 变更频率差异
        freq_map = {"high": 3, "medium": 2, "low": 1}
        freq_diff = abs(freq_map[self.components[comp_a].change_frequency] -
                        freq_map[self.components[comp_b].change_frequency])
        if freq_diff >= 2:
            score *= 0.8  # 变更频率差异大，减少20%的合并倾向
            factors.append(f"变更频率差异大: ×0.8")
        
        return min(score, 1.0)
    
    def suggest_boundaries(self, threshold: float = 0.5):
        """
        根据耦合度分数建议服务边界
        threshold: 超过此阈值建议合并
        """
        print("=" * 70)
        print("服务边界分析报告")
        print("=" * 70)
        
        names = list(self.components.keys())
        print(f"\n分析组件: {', '.join(names)}")
        print(f"聚合阈值: {threshold}")
        print()
        
        print("耦合度矩阵:")
        print(f"{'':12s}", end="")
        for name in names:
            print(f"{name:12s}", end="")
        print()
        
        for name_a in names:
            print(f"{name_a:12s}", end="")
            for name_b in names:
                if name_a == name_b:
                    print(f"{'——':12s}", end="")
                else:
                    score = self.calculate_coupling_score(name_a, name_b)
                    marker = "*" if score >= threshold else " "
                    print(f"{score:.2f}{marker:>9s}", end="")
            print()
        
        print("\n建议:")
        merged = set()
        for i, name_a in enumerate(names):
            for name_b in names[i+1:]:
                score = self.calculate_coupling_score(name_a, name_b)
                action = "合并" if score >= threshold else "拆分"
                print(f"  {name_a} ↔ {name_b}: {score:.2f} → {action}")
        
        print("\n* = 超过阈值，建议合并到同一服务")
        return


# ============ 使用示例 ============

def analyze_example_agent():
    """分析一个典型的 Agent 系统"""
    analyzer = ServiceBoundaryAnalyzer()
    
    # 定义组件
    components = [
        Component("推理引擎", "LLM 推理和思维链生成", "gpu", "medium", "high"),
        Component("规划器", "任务分解和行动计划", "cpu", "high", "medium"),
        Component("工具执行", "调用外部 API 和工具", "cpu", "high", "medium"),
        Component("代码执行", "安全沙箱中运行代码", "cpu", "low", "high"),
        Component("记忆检索", "短期/长期记忆查询", "memory", "low", "medium"),
        Component("验证器", "检查输出质量和安全", "cpu", "medium", "low"),
    ]
    
    for comp in components:
        analyzer.add_component(comp)
    
    # 定义交互关系
    interactions = [
        Interaction("推理引擎", "规划器", InteractionType.SYNC_CALL, 3, "medium", True),
        Interaction("推理引擎", "工具执行", InteractionType.SYNC_CALL, 5, "medium", True),
        Interaction("推理引擎", "代码执行", InteractionType.SYNC_CALL, 2, "large", True),
        Interaction("推理引擎", "记忆检索", InteractionType.SYNC_CALL, 8, "large", True),
        Interaction("推理引擎", "验证器", InteractionType.SYNC_CALL, 3, "small", False),
        Interaction("规划器", "工具执行", InteractionType.ASYNC_EVENT, 1, "small", False),
        Interaction("工具执行", "记忆检索", InteractionType.SHARED_DATA, 2, "medium", False),
        Interaction("代码执行", "验证器", InteractionType.SYNC_CALL, 1, "large", True),
    ]
    
    for interaction in interactions:
        analyzer.add_interaction(interaction)
    
    analyzer.suggest_boundaries(threshold=0.5)


if __name__ == "__main__":
    analyze_example_agent()
```

### 5.2 服务接口定义 (FastAPI + Pydantic)

```python
# service_interfaces.py
"""
Agent 系统微服务的接口定义。

展示了服务拆分后，每个服务的 API 接口、数据模型和契约。
"""

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import Dict, List, Optional, Any
from enum import Enum
import uuid
from datetime import datetime


# ============ 1. 推理服务接口 ============

class ReasoningRequest(BaseModel):
    """推理服务请求"""
    session_id: str = Field(..., description="会话 ID，用于追踪")
    task_id: str = Field(default="", description="任务 ID")
    context: Dict[str, Any] = Field(..., description="推理上下文")
    conversation_history: List[Dict[str, Any]] = Field(
        default_factory=list,
        description="对话历史 (由编排器管理)"
    )
    model_config: Dict[str, Any] = Field(
        default_factory=lambda: {
            "model": "gpt-4",
            "temperature": 0.7,
            "max_tokens": 4096
        },
        description="模型配置"
    )
    available_tools: List[Dict[str, Any]] = Field(
        default_factory=list,
        description="当前可用的工具列表"
    )


class ReasoningResponse(BaseModel):
    """推理服务响应"""
    session_id: str
    thought: str = Field(..., description="LLM 生成的思考内容")
    needs_tool: bool = Field(default=False, description="是否需要调用工具")
    tool_name: Optional[str] = Field(default=None, description="需要调用的工具名")
    tool_params: Optional[Dict[str, Any]] = Field(
        default=None, description="工具调用参数"
    )
    tokens_used: int = Field(default=0, description="本次推理消耗的 Token 数")
    confidence: float = Field(default=0.0, description="置信度 (0-1)")
    finish_reason: str = Field(default="", description="完成原因")


class FinalizeRequest(BaseModel):
    """最终结果生成请求"""
    session_id: str
    task_id: str
    context: Dict[str, Any]
    conversation_history: List[Dict[str, Any]]


class FinalizeResponse(BaseModel):
    """最终结果响应"""
    session_id: str
    final_output: str
    tokens_used: int
    processing_time_ms: int


# ============ 2. 工具执行服务接口 ============

class ToolExecutionRequest(BaseModel):
    """工具执行请求"""
    tool_name: str
    params: Dict[str, Any]
    context: Dict[str, Any] = Field(
        default_factory=dict,
        description="执行的上下文信息"
    )
    timeout_seconds: int = Field(default=30, ge=1, le=300)
    retry_count: int = Field(default=0, ge=0, le=5)


class ToolExecutionResponse(BaseModel):
    """工具执行响应"""
    execution_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    tool_name: str
    success: bool
    result: Any = None
    error: Optional[str] = None
    execution_time_ms: int = 0
    metrics: Dict[str, Any] = Field(default_factory=dict)


class ToolDefinition(BaseModel):
    """工具定义"""
    name: str
    description: str
    schema: Dict[str, Any]
    enabled: bool = True
    timeout_default: int = 30
    rate_limit: Optional[int] = None  # RPM


# ============ 3. 记忆服务接口 ============

class StoreMemoryRequest(BaseModel):
    """记忆存储请求"""
    session_id: str
    content: Dict[str, Any]
    memory_type: str = Field(..., pattern="^(short_term|long_term|working)$")
    importance: float = Field(default=0.5, ge=0.0, le=1.0)
    metadata: Dict[str, Any] = Field(default_factory=dict)


class RetrieveMemoryRequest(BaseModel):
    """记忆检索请求"""
    session_id: str
    query: str
    memory_types: List[str] = Field(
        default=["short_term", "long_term"],
        description="检索的记忆类型"
    )
    top_k: int = Field(default=5, ge=1, le=50)
    min_importance: float = Field(default=0.0, ge=0.0, le=1.0)


class MemoryItem(BaseModel):
    """记忆条目"""
    memory_id: str
    content: Dict[str, Any]
    memory_type: str
    importance: float
    timestamp: datetime
    relevance_score: float = 0.0


class RetrieveMemoryResponse(BaseModel):
    """记忆检索响应"""
    session_id: str
    items: List[MemoryItem]
    total_found: int
    query_time_ms: int


# ============ 4. 规划服务接口 ============

class PlanRequest(BaseModel):
    """规划请求"""
    session_id: str
    goal: str = Field(..., description="总体目标")
    context: Dict[str, Any]
    available_tools: List[str]
    constraints: List[str] = Field(default_factory=list)
    max_steps: int = Field(default=10, ge=1, le=50)


class PlanStep(BaseModel):
    """规划中的一步"""
    step_id: int
    action: str
    description: str
    dependencies: List[int] = Field(default_factory=list)
    estimated_tools: List[str] = Field(default_factory=list)


class PlanResponse(BaseModel):
    """规划响应"""
    session_id: str
    plan_id: str
    steps: List[PlanStep]
    estimated_steps: int
    plan_summary: str


# ============ 5. 验证服务接口 ============

class ValidationRequest(BaseModel):
    """验证请求"""
    task_id: str
    step_id: int
    input_data: Dict[str, Any]
    output_data: Dict[str, Any]
    tool_results: Optional[List[Dict[str, Any]]] = None
    validation_rules: List[str] = Field(default_factory=list)


class ValidationResult(BaseModel):
    """验证结果项"""
    check_name: str
    passed: bool
    score: float = Field(default=0.0, ge=0.0, le=1.0)
    details: Optional[str] = None


class ValidationResponse(BaseModel):
    """验证响应"""
    task_id: str
    step_id: int
    overall_passed: bool
    results: List[ValidationResult]
    overall_score: float


# ============ 6. 编排服务接口 ============

class AgentTaskRequest(BaseModel):
    """Agent 任务请求 (面向外部)"""
    user_id: str = Field(..., description="用户 ID")
    input_text: str = Field(..., description="用户输入")
    task_type: str = Field(default="general", description="任务类型")
    config: Dict[str, Any] = Field(default_factory=dict)
    max_steps: int = Field(default=20, ge=1, le=100)


class AgentTaskResponse(BaseModel):
    """Agent 任务响应"""
    task_id: str
    status: str  # pending | in_progress | completed | failed
    created_at: datetime
    estimated_completion_ms: int = 0


class TaskStatusResponse(BaseModel):
    """任务状态查询响应"""
    task_id: str
    status: str
    current_step: int
    total_steps: int
    progress: float  # 0.0 - 1.0
    partial_output: Optional[str] = None
    error: Optional[str] = None
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None


# ============ FastAPI 应用定义 ============

# 这些路由定义展示每个服务的 API 端点
# 实际部署时每个服务是独立的 FastAPI 应用

reasoning_app = FastAPI(title="Reasoning Service")
tool_exec_app = FastAPI(title="Tool Execution Service")
memory_app = FastAPI(title="Memory Service")
planner_app = FastAPI(title="Planner Service")
validator_app = FastAPI(title="Validation Service")
orchestrator_app = FastAPI(title="Orchestrator Service")


# 推理服务路由示例
@reasoning_app.post("/reason", response_model=ReasoningResponse)
async def reason(request: ReasoningRequest):
    """
    LLM 推理端点。
    接收上下文和历史，返回推理结果和工具选择。
    """
    # 实际实现会调用 LLM API
    # 此处为接口定义示例
    return ReasoningResponse(
        session_id=request.session_id,
        thought="分析用户输入...",
        needs_tool=False,
        tokens_used=500,
        confidence=0.85,
        finish_reason="complete"
    )


@reasoning_app.post("/finalize", response_model=FinalizeResponse)
async def finalize(request: FinalizeRequest):
    """生成最终输出"""
    return FinalizeResponse(
        session_id=request.session_id,
        final_output="",
        tokens_used=1000,
        processing_time_ms=2000
    )


# 工具执行服务路由示例
@tool_exec_app.post("/execute", response_model=ToolExecutionResponse)
async def execute_tool(request: ToolExecutionRequest):
    """执行工具调用"""
    return ToolExecutionResponse(
        tool_name=request.tool_name,
        success=True,
        result={},
        execution_time_ms=150
    )


@tool_exec_app.get("/tools", response_model=List[ToolDefinition])
async def list_tools():
    """列出可用工具"""
    return []


# 记忆服务路由示例
@memory_app.post("/memory/store")
async def store_memory(request: StoreMemoryRequest):
    """存储记忆"""
    return {"status": "stored", "memory_id": str(uuid.uuid4())}


@memory_app.post("/memory/retrieve", response_model=RetrieveMemoryResponse)
async def retrieve_memory(request: RetrieveMemoryRequest):
    """检索记忆"""
    return RetrieveMemoryResponse(
        session_id=request.session_id,
        items=[],
        total_found=0,
        query_time_ms=10
    )


# 规划服务路由示例
@planner_app.post("/plan", response_model=PlanResponse)
async def create_plan(request: PlanRequest):
    """创建任务规划"""
    return PlanResponse(
        session_id=request.session_id,
        plan_id=str(uuid.uuid4()),
        steps=[],
        estimated_steps=0,
        plan_summary=""
    )


# 验证服务路由示例
@validator_app.post("/validate", response_model=ValidationResponse)
async def validate_output(request: ValidationRequest):
    """验证输出"""
    return ValidationResponse(
        task_id=request.task_id,
        step_id=request.step_id,
        overall_passed=True,
        results=[],
        overall_score=0.95
    )


# 编排服务路由示例
@orchestrator_app.post("/task", response_model=AgentTaskResponse)
async def create_task(request: AgentTaskRequest):
    """创建 Agent 任务"""
    return AgentTaskResponse(
        task_id=str(uuid.uuid4()),
        status="pending",
        created_at=datetime.now(),
        estimated_completion_ms=10000
    )


@orchestrator_app.get("/task/{task_id}", response_model=TaskStatusResponse)
async def get_task_status(task_id: str):
    """查询任务状态"""
    return TaskStatusResponse(
        task_id=task_id,
        status="in_progress",
        current_step=3,
        total_steps=10,
        progress=0.3
    )
```

### 5.3 服务模板代码

```python
# service_template.py
"""
微服务 Agent 的基础服务模板。

每个微服务应该：
  1. 继承 BaseAgentService 来获得基础功能
  2. 实现自己的业务逻辑
  3. 注册到服务发现中心
  4. 导出健康检查和指标端点
"""

from fastapi import FastAPI, Response
from pydantic import BaseModel
from typing import Dict, Any, Optional
import uvicorn
import logging
import time

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(name)s] %(levelname)s: %(message)s"
)
logger = logging.getLogger("AgentService")


class ServiceHealth(BaseModel):
    """健康检查响应"""
    service_name: str
    status: str  # healthy | degraded | unhealthy
    version: str
    uptime_seconds: float
    active_requests: int
    last_error: Optional[str] = None


class BaseAgentService:
    """
    微服务 Agent 基础服务类。
    
    所有 Agent 微服务都应该继承此类，获得统一的基础设施：
    - 健康检查端点
    - Metrics 端点
    - 优雅启动/关闭
    - 统一日志格式
    - 请求追踪
    """
    
    def __init__(
        self,
        service_name: str,
        version: str = "1.0.0",
        host: str = "0.0.0.0",
        port: int = 8000
    ):
        self.service_name = service_name
        self.version = version
        self.host = host
        self.port = port
        self.start_time = time.time()
        self.active_requests = 0
        
        self.app = FastAPI(
            title=service_name,
            version=version,
            docs_url="/docs" if version != "production" else None
        )
        
        self._setup_base_routes()
        logger.info(f"Service '{service_name}' initialized (v{version})")
    
    def _setup_base_routes(self):
        """设置基础路由 (健康检查、指标等)"""
        
        @self.app.get("/health", response_model=ServiceHealth)
        async def health_check():
            """健康检查端点"""
            return ServiceHealth(
                service_name=self.service_name,
                status="healthy",
                version=self.version,
                uptime_seconds=time.time() - self.start_time,
                active_requests=self.active_requests
            )
        
        @self.app.get("/ready")
        async def readiness():
            """就绪检查 (Kubernetes Readiness Probe)"""
            return {"status": "ready"}
        
        @self.app.get("/live")
        async def liveness():
            """存活检查 (Kubernetes Liveness Probe)"""
            return {"status": "alive"}
        
        @self.app.get("/metrics")
        async def metrics():
            """Prometheus 指标端点"""
            return {
                "service": self.service_name,
                "uptime_seconds": time.time() - self.start_time,
                "active_requests": self.active_requests,
                "version": self.version
            }
    
    def run(self):
        """启动服务"""
        logger.info(
            f"Starting '{self.service_name}' on {self.host}:{self.port}"
        )
        uvicorn.run(
            self.app,
            host=self.host,
            port=self.port,
            log_level="info"
        )


# ============ 服务实现示例 ============

class ReasoningService(BaseAgentService):
    """推理服务的具体实现"""
    
    def __init__(self):
        super().__init__(
            service_name="agent-reasoning-service",
            version="2.1.0",
            port=8001
        )
        self._setup_reasoning_routes()
    
    def _setup_reasoning_routes(self):
        @self.app.post("/v1/reason")
        async def reason(request: ReasoningRequest):
            self.active_requests += 1
            try:
                # 实际的 LLM 调用逻辑
                # ...
                return ReasoningResponse(...)
            finally:
                self.active_requests -= 1


class ToolService(BaseAgentService):
    """工具执行服务的具体实现"""
    
    def __init__(self):
        super().__init__(
            service_name="agent-tool-service",
            version="1.5.0",
            port=8002
        )
        self._setup_tool_routes()
    
    def _setup_tool_routes(self):
        @self.app.post("/v1/execute")
        async def execute(request: ToolExecutionRequest):
            self.active_requests += 1
            try:
                # 工具执行逻辑
                # ...
                pass
            finally:
                self.active_requests -= 1


class MemoryService(BaseAgentService):
    """记忆服务的具体实现"""
    
    def __init__(self):
        super().__init__(
            service_name="agent-memory-service",
            version="1.2.0",
            port=8003
        )
        self._setup_memory_routes()
    
    def _setup_memory_routes(self):
        @self.app.post("/v1/memory/store")
        async def store(request: StoreMemoryRequest):
            # 记忆存储逻辑
            pass
        
        @self.app.post("/v1/memory/retrieve")
        async def retrieve(request: RetrieveMemoryRequest):
            # 记忆检索逻辑
            pass


# 启动入口
if __name__ == "__main__":
    # 根据环境变量选择启动哪个服务
    import os
    service_type = os.getenv("AGENT_SERVICE_TYPE", "reasoning")
    
    services = {
        "reasoning": ReasoningService,
        "tool": ToolService,
        "memory": MemoryService,
    }
    
    service_cls = services.get(service_type)
    if service_cls:
        service = service_cls()
        service.run()
    else:
        logger.error(f"Unknown service type: {service_type}")
```

---

## 6. 最大挑战

### 挑战 1: 服务粒度 (Service Granularity)

服务粒度是服务拆分中最难把握的问题。

```
粒度过粗 (拆分不足):
  [Agent 核心服务]
    包含: 推理 + 规划 + 验证 + 部分工具
    问题:
      • 资源无法独立分配 (推理用 GPU，规划和验证用 CPU)
      • 变更互相影响 (改验证逻辑需要重新部署推理)
      • 扩缩不灵活 (想扩推理能力却连带扩展了规划)
    表现: 部署频率 < 1 次/周 → 需要进一步拆分

粒度过细 (过度拆分):
  [Token 计数服务] [Prompt 格式化服务] [温度参数调整服务] ...
    问题:
      • 服务间通信开销 > 实际处理时间
      • 一个推理步骤需要调用 5 个服务
      • 网络延迟累积：5 × 2ms RTT = 10ms 额外延迟
      • 部署和运维复杂度急剧上升
    表现: 单次 Agent 请求中服务间调用 > 50 次 → 需要合并

正确粒度的判断标准:
  1. 一个服务的变更是否要求同时变更另一个服务？ → 是 → 过细
  2. 一个服务的资源是否大部分被浪费？ → 是 → 过粗
  3. 一个服务是否有一半以上的 API 未被使用？ → 是 → 过粗
  4. 是否经常需要同时部署多个服务？ → 是 → 过细

经验法则:
  一个服务的合理大小 = 一个 2-4 人团队能独立维护的代码量
  服务数量 = (团队总人数 / 2) 到 (团队总人数 / 4)
```

### 挑战 2: 分布式数据一致性

Agent 系统对数据一致性的需求与传统系统不同。

```
Agent 系统中的一致性场景:

场景 A: 推理结果依赖记忆检索结果
  编排器：调用记忆服务 → 得到结果 → 传递给推理服务
  
  问题：
    记忆服务在写入后未完全同步，检索到旧数据
    → 推理基于错误上下文 → 生成错误响应

场景 B: 工具执行结果持久化
  编排器：工具服务执行完成 → 通知记忆服务存储结果
         → 推理服务基于结果继续推理
  
  问题：
    工具执行成功 → 记忆服务写入失败 (网络问题)
    → 推理服务不知道工具执行过 → 重复执行工具

场景 C: 多服务状态一致性
  用户：停止正在运行的 Agent 任务
  编排器：标记任务为 cancelled
  推理服务：还在处理上一个请求
  工具服务：正在执行工具调用
  
  问题：
    部分服务收到了取消信号，部分没有
    → 工具在 cancelled 状态后还在执行

解决方案：

1. 最终一致性 (推荐用于大多数 Agent 场景)
   原则：Agent 可以容忍短时间的不一致
   实现：异步消息 + 定期校对
   
2. Saga 事务模式 (用于关键操作)
   步骤 1: 工具执行 → 步骤 2: 记忆存储 → 步骤 3: 推理
   补偿: 如果记忆存储失败 → 撤销工具执行 (调用补偿 API)

3. 事件溯源 (用于审计和回溯)
   所有 Agent 状态变更作为事件存储
   可以从任意时间点重建状态
```

### 挑战 3: 网络开销与延迟

```
服务拆分带来的网络开销分析：

拆分前 (单体):
  Agent 处理一个请求的总时间:
    T_total = T_llm_1 + T_tool_1 + T_llm_2 + T_tool_2 + T_llm_3
            = 2s + 0.5s + 1s + 0.3s + 0.5s
            = 4.3s
  网络开销: 0 (所有调用在进程内)

拆分后 (微服务，假设 3 步循环):
  T_total = 
    T_llm_1 + RTT(R→T) + T_tool_1 + RTT(T→R) +   # 第一轮
    T_llm_2 + RTT(R→T) + T_tool_2 + RTT(T→R) +   # 第二轮
    T_llm_3                                        # 第三轮
            + 5 × RTT(编排器→各服务)                # 编排开销
  
  假设 RTT = 2ms (同数据中心):
    T_total = 4.3s + 4 × 2ms + 5 × 2ms
            = 4.3s + 18ms
            = 4.318s
  
  假设 RTT = 10ms (跨可用区):
    T_total = 4.3s + 90ms = 4.39s
  
  假设 10 步 + RTT=10ms:
    T_total = 4.3s + 10×2×10ms + 10×10ms
            = 4.3s + 300ms = 4.6s  ← 延迟增加 7%

减小网络开销的方法:
  1. 同机部署高频互调的服务 (K8s Pod 亲和性)
  2. 使用 gRPC 长连接 (避免 TCP 握手)
  3. 批量 RPC (合并多个小请求)
  4. 结果缓存 (减少重复调用)
  5. 预测性预取 (在推理期间预取可能需要的数据)
```

---

## 7. 能力边界：何时不应该拆分

服务拆分并非总是正确选择。以下情况应该保持单体或合并服务：

### 7.1 复杂度阈值

```
拆分决策的复杂度阈值:

Agent 系统复杂度 = f(Agent 数, 工具数, 推理步数, 用户数)

区域 1: 不需要拆分
  复杂度 < 100:
    • 1 个 Agent, <5 个工具, <10 步推理, <100 日活
    • 推荐：单体 Agent (14.1.5)
    • 原因：微服务化的成本 > 收益

区域 2: 可以考虑拆分
  复杂度 100-500:
    • 1-2 个 Agent, 5-15 个工具, 10-20 步, 100-1000 日活
    • 推荐：将工具服务独立 (部分微服务化)
    • 原因：工具执行的资源需求开始成为瓶颈

区域 3: 强烈建议拆分
  复杂度 > 500:
    • 2+ Agent, >15 工具, >20 步, >1000 日活
    • 推荐：全面微服务化
    • 原因：不拆分的运维成本 > 拆分后的架构成本

判断公式 (经验值):
  是否拆分 = 资源异构度 × 团队规模 × 变更频率 / 延迟敏感度
  如果 > 1: 考虑拆分
  如果 > 3: 必须拆分
```

### 7.2 不应该拆分的信号

```
以下信号表示应该保持单体：

1. 团队信号
   ├── 团队 < 3 人
   ├── 没有专职 DevOps/SRE
   ├── 没有容器化经验
   └── 当前单体代码尚未进行模块化重构

2. 产品信号
   ├── 产品尚未验证 PMF
   ├── 日活用户 < 100
   ├── 日均请求 < 1000
   └── 功能仍在快速迭代 (周更)

3. 技术信号
   ├── Agent 推理步数 < 5
   ├── 工具数量 < 5
   ├── 所有组件使用相同类型的资源
   └── 当前单体的 P99 延迟 < 5s (用户可接受)

4. 成本信号
   ├── 当前基础设施成本 < $1000/月
   ├── 微服务化后的预估成本 > 当前成本的 3 倍
   └── 团队时间成本：拆分需要 3 个月，这 3 个月无法开发新功能

5. 延迟信号
   ├── 端到端延迟要求 < 1s (实时交互)
   ├── 工具执行时间 < 50ms (所有工具在进程内即可完成)
   └── 不可接受额外 10ms 以上的网络开销

如果命中 3 个以上信号，请暂缓微服务化
```

### 7.3 合并的信号

有时拆分后需要合并：

```
需要合并回单体的信号：

1. 数据需频繁跨服务访问
   [服务 A] 每次推理需要 [服务 B] 的 10 个数据项
   → 10 次 RPC 调用 → 最佳方案：合并 A 和 B

2. 服务间事务边界模糊
   一个 "推理步骤" 涉及 4 个服务 → 无法原子化
   → 合并推理相关的服务

3. 部署耦合不减反增
   拆分后每次发布仍需同时部署 4 个服务
   → 说明拆分粒度错误，合并高频协同的服务

4. 团队协作成本上升
   修改一个功能需要跨 3 个团队的 Code Review
   → 组织结构不支持当前拆分粒度

合并策略：
  • 反向拆分：合并推理和规划 (如果它们总是同时变更)
  • 库化：将高频互调的服务代码提取为共享库
  • Sidecar：将辅助功能作为 Sidecar 进程运行
```

---

## 8. 与 14.1.4 微服务 Agent 的区别

```
14.1.4 微服务 Agent (模式篇)            14.3.1 服务拆分 (方法篇)
──────────────────────────             ────────────────────────

级别：架构模式                        级别：设计方法
  "这是一种架构风格"                     "这是如何拆分的具体方法"

回答：为什么用微服务？                  回答：怎么拆？
  • 独立扩缩、故障隔离、团队自治          • 6 种拆分策略
  • 适用场景和局限性                     • 限界上下文分析
                                       • 粒度控制

输出：架构蓝图                         输出：拆分方案
  • 服务组件列表                         • 服务边界定义
  • 通信模式                             • API 契约
  • 部署模型                             • 数据所有权

典型内容：
  微服务 Agent 的架构长什么样            典型内容：
  有 7 个主要服务                          如何用 DDD 识别限界上下文
  服务间用 gRPC 通信                       6 种启发式拆分策略
  推理无状态，编排有状态                    什么时候不拆分

关系：
  14.1.4 回答了 "what" → 微服务 Agent 是什么
  14.3.1 回答了 "how" → 如何将系统拆分为微服务

先学 14.1.4 理解微服务 Agent 的概念，
再学 14.3.1 学习如何实现拆分。
```

---

## 总结：服务拆分核心要点

```
服务拆分的 5 个关键步骤：

Step 1: 分析现状
  绘制组件依赖图 → 识别交互模式 → 评估资源需求

Step 2: 选择维度
  能力拆分 (默认) + 资源拆分 (必须) + 领域拆分 (业务复杂时)

Step 3: 应用策略
  6 种启发式策略 → 至少使用 2-3 种交叉验证 → 综合决策

Step 4: 定义边界
  为每个候选服务定义：
  • 职责说明 (一句话描述)
  • API 契约 (输入/输出/协议)
  • 数据所有权 (拥有什么数据)
  • 资源需求 (CPU/GPU/内存/存储)

Step 5: 验证合理性
  • 检查内聚性：服务内部功能是否高度相关？
  • 检查耦合度：服务间依赖是否最小化？
  • 检查粒度：服务数量是否在合理范围内？
  • 检查对齐度：服务边界是否与团队结构一致？

经验法则：
  • 从一个服务开始，拆分不如合并常见
  • 服务数量应该等于团队数量 × (1-2)
  • 如果不知道如何拆分，先模块化单体
  • 如果拆分后问题更多，考虑合并
```

---

## 参考

- Eric Evans: _Domain-Driven Design: Tackling Complexity in the Heart of Software_
- Sam Newman: _Building Microservices: Designing Fine-Grained Systems_
- Chris Richardson: _Microservices Patterns_
- Vaughn Vernon: _Implementing Domain-Driven Design_
- Martin Fowler: _Patterns of Enterprise Application Architecture_
- 14.1.4 微服务 Agent — 本节对应的架构模式
- 14.3.2 服务间通信 — 拆分后的下一章节
