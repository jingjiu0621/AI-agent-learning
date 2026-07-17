# 8.6 frameworks -- 多 Agent 框架源码分析

## 概述

多 Agent 框架是多智能体系统从理论走向工程的桥梁。在 8.1-8.5 中，我们学习了通信协议、协调机制、委派策略、辩论规则和群体组织模式——这些是构成多 Agent 系统的"设计模式"。而 8.6 的目标是回答一个更实际的问题：**这些模式在主流框架中是如何实现的？**

分析框架源码的意义在于：

- **打破黑箱**：框架封装了大量复杂性，但调试和定制需要理解其内部机制。知其然更要知其所以然。
- **理解设计取舍**：每个框架在灵活性、抽象层级、开发体验、运行时性能之间做出了不同的权衡。这些取舍反映了其设计者对多 Agent 系统核心挑战的理解。
- **提升工程判断力**：知道框架的边界在哪里——什么场景下框架帮到你，什么场景下框架反而成为束缚。
- **汲取工程智慧**：LangGraph 的状态管理、CrewAI 的任务编排、AutoGen 的对话管理——每个框架都有值得借鉴的工程模式。

本模块分析三个代表性框架：

| 框架 | 定位 | 核心抽象 | 分析深度 |
|------|------|---------|---------|
| **CrewAI** | 多 Agent 协作框架 | Crew / Agent / Task / Tool | 编排逻辑与工具系统源码（8.6.1-8.6.2） |
| **AutoGen** | 多 Agent 对话框架 | Agent / Conversation / GroupChat | 对话管理与 AgentChat 源码（8.6.3-8.6.4） |
| **LangGraph** | Agent 图状态机框架 | StateGraph / Node / Edge / Channel | 多 Agent 图结构源码（8.6.5） |
| **横向对比** | 跨框架分析 | 架构 / 选型 / 适用场景 | 综合比较（8.6.6） |

三个框架代表了三种不同的多 Agent 实现哲学：

```
  图状态机 (LangGraph)         对话驱动 (AutoGen)            任务编排 (CrewAI)
  ┌──────────────────┐     ┌──────────────────────┐     ┌───────────────────┐
  │ Agent = 图节点    │     │ Agent = 对话参与者    │     │ Agent = 角色+工具  │
  │ 状态 = 全局图状态  │     │ 状态 = 对话历史       │     │ 状态 = 任务上下文  │
  │ 控制 = 边+条件边   │     │ 控制 = 对话流程       │     │ 控制 = 过程定义    │
  │ 哲学: 你控制流程   │     │ 哲学: 对话产生结果    │     │ 哲学: 我组织任务   │
  │ 灵活但自由度过高   │     │ 自然但控制力弱       │     │ 明确但灵活性低    │
  └──────────────────┘     └──────────────────────┘     └───────────────────┘
```

## 知识图谱

```
                                     ┌──────────────────────────────────────────────────────────────┐
                                     │              多 Agent 框架源码分析全景图                       │
                                     │      从抽象层到实现层的完整映射                                  │
                                     └──────────────────────────────────────────────────────────────┘
                                                    │
      ┌─────────────────────────────────────────────┼─────────────────────────────────────────────┐
      │                                             │                                             │
      │                                             │                                             │
┌─────▼──────────────────────┐     ┌────────────────▼──────────────┐     ┌──────────────────────▼──┐
│      CrewAI                 │     │        AutoGen               │     │      LangGraph          │
│      (CrewAI)               │     │        (AutoGen)             │     │      (LangGraph)        │
│                            │     │                              │     │                         │
│  ┌──────────────────┐      │     │  ┌────────────────────┐      │     │  ┌─────────────────┐    │
│  │ 8.6.1            │      │     │  │ 8.6.3              │      │     │  │ 8.6.5           │    │
│  │ crewai-orch      │      │     │  │ autogen-conv       │      │     │  │ langgraph-multi │    │
│  │ 编排引擎源码      │      │     │  │ 对话管理源码        │      │     │  │ 多Agent图源码    │    │
│  └────────┬─────────┘      │     │  └────────┬───────────┘      │     │  └────────┬────────┘    │
│           │                │     │           │                  │     │           │             │
│  ┌────────▼─────────┐      │     │  ┌────────▼───────────┐      │     │  ┌────────▼────────┐    │
│  │ Crew 类           │      │     │  │ Conversation       │      │     │  │ StateGraph       │    │
│  │ Agent 定义        │      │     │  │ GroupChat          │      │     │  │ Node / Edge      │    │
│  │ Task 生命周期     │      │     │  │ Agent 注册与路由    │      │     │  │ Channel / Reducer │    │
│  │ Process 执行器    │      │     │  │ 发言顺序轮转        │      │     │  │ Checkpointer     │    │
│  │ 顺序/层级流程      │      │     │  │ 消息广播/定向      │      │     │  │ 条件边路由       │    │
│  └──────────────────┘      │     │  └────────────────────┘      │     │  └──────────────────┘    │
│                            │     │                              │     │                         │
│  ┌──────────────────┐      │     │  ┌────────────────────┐      │     │  ┌─────────────────┐    │
│  │ 8.6.2            │      │     │  │ 8.6.4              │      │     │  │                 │    │
│  │ crewai-tools     │      │     │  │ autogen-agentchat  │      │     │  │                 │    │
│  │ 工具系统源码      │      │     │  │ AgentChat 源码     │      │     │  │                 │    │
│  └────────┬─────────┘      │     │  └────────┬───────────┘      │     │  └─────────────────┘    │
│           │                │     │           │                  │     │                         │
│  ┌────────▼─────────┐      │     │  ┌────────▼───────────┐      │     │                         │
│  │ BaseTool          │      │     │  │ AgentChat 类        │      │     │                         │
│  │ Tool 注册机制      │      │     │  │ Team / Agent       │      │     │                         │
│  │ 工具错误处理       │      │     │  │ Handoff / ToolUse   │      │     │                         │
│  │ 工具结果格式化     │      │     │  │ 流式响应            │      │     │                         │
│  │ 工具委托模式       │      │     │  │ 终止条件            │      │     │                         │
│  └──────────────────┘      │     │  └────────────────────┘      │     │                         │
│                            │     │                              │     │                         │
└──────────┬─────────────────┘     └────────────────┬─────────────┘     └──────────────────────┬───┘
           │                                         │                                         │
           └─────────────────────────────────────────┼─────────────────────────────────────────┘
                                                     │
                                                     ▼
                              ┌──────────────────────────────────────────────────────┐
                              │                    8.6.6 framework-compare              │
                              │             跨框架综合对比与选型分析                     │
                              │                                                       │
                              │  ┌──────────┐  ┌──────────┐  ┌────────────────────┐   │
                              │  │ 架构对比  │  │ 性能对比  │  │ 适用场景匹配       │   │
                              │  │ 抽象层级  │  │ 流式支持  │  │ 生产就绪度评估     │   │
                              │  │ 状态管理  │  │ 持久化   │  │ 社区生态对比       │   │
                              │  └──────────┘  └──────────┘  └────────────────────┘   │
                              └──────────────────────────────────────────────────────┘
                                                     │
                                                     ▼
                              ┌──────────────────────────────────────────────────────┐
                              │           选型输出：适用于你的项目的框架推荐           │
                              └──────────────────────────────────────────────────────┘
```

## 内容结构

| 子主题 | 核心问题 | 难度 | 框架 |
|--------|----------|------|------|
| 8.6.1 crewai-orch | CrewAI 是如何编排多 Agent 协作流程的？ | ★★★★☆ | CrewAI |
| 8.6.2 crewai-tools | CrewAI 的工具系统如何设计？工具如何发现和执行？ | ★★★★☆ | CrewAI |
| 8.6.3 autogen-conversation | AutoGen 的对话管理层如何实现 Agent 间通信？ | ★★★★☆ | AutoGen |
| 8.6.4 autogen-agentchat | AutoGen AgentChat 的 Team/Agent/Handoff 机制源码分析 | ★★★★★ | AutoGen |
| 8.6.5 langgraph-multi | LangGraph 如何用 StateGraph 实现多 Agent 协作？ | ★★★★★ | LangGraph |
| 8.6.6 framework-compare | 三个框架在同场景下的源码实现差异和选型建议 | ★★★☆☆ | 横向 |

## 框架核心抽象对比

每个框架都围绕一组核心抽象构建。理解这些抽象是阅读源码的起点：

```
抽象层级          CrewAI                     AutoGen                    LangGraph
────────────────────────────────────────────────────────────────────────────────────
最高层          Crew                       Team                        Graph
                 │                          │                           │
中层            Agent / Task               Agent / GroupChat           Node
                 │                          │                           │
基础层          Tool / Process             ConversationMessage         State / Channel
                 │                          │                           │
最底层          LLM 调用                     LLM 调用                    LLM 调用
```

```python
# === 三个框架的核心抽象 ===

# CrewAI: Agent + Task + Tool -> Process -> Crew -> Kickoff
# Crew 管理一组 Agent 和 Task，通过 Process(顺序/层级) 编排执行

# AutoGen (conversation): Agent + Message -> GroupChat -> Manager
# Agent 们通过 GroupChat 对话，由 Manager 协调发言顺序

# AutoGen (agentchat): Agent + Tool + Handoff -> Team -> Run
# Team 管理 Agent，通过 Handoff 在 Agent 间转交控制权

# LangGraph: State + Node + Edge -> StateGraph -> Compile -> Invoke
# 将多 Agent 协作建模为图，节点是 Agent 逻辑，边是数据流
```

## 三框架全景对比

| 维度 | CrewAI | AutoGen | LangGraph |
|------|--------|---------|-----------|
| **核心抽象** | Crew Agent Task Tool | Agent Conversation Handoff | StateGraph Node Edge |
| **架构哲学** | 任务编排驱动 | 对话驱动 | 状态图驱动 |
| **状态管理** | Crew-level 上下文传递，Task 输出链式传递 | 对话历史累加，消息列表 | StateGraph 全局状态，Reducer 更新 |
| **Agent 定义** | Role + Goal + Backstory + Tools | system_message + LLM 配置 | 自定义函数，接收 State 返回 State |
| **任务分配** | Process 定义执行顺序，Task 指定 Agent | 对话自然流转，Manager 调度发言 | 图拓扑定义，边决定数据流向 |
| **工具执行** | BaseTool 子类化，参数 Schema 自动生成 | Tool 函数注册，FuncTool 封装 | 节点内部调用，无内置工具抽象 |
| **消息路由** | Task 输出到 Task | GroupChat 广播 + 定向 | Edge / ConditionalEdge |
| **通信模式** | 隐式（Task 输出传递） | 显式（消息列表交换） | 显式（State 读写） |
| **流式支持** | 有限（Task 级回调） | 事件驱动（AgentChat Team） | 内置（Node / Channel 流式输出） |
| **持久化** | 可选 PostgreSQL / SQLite | Conversation History | Checkpointer (MemorySaver, SqliteSaver, PostgresSaver) |
| **并行执行** | 层级 Process 支持部分并行 | 非原生（同步对话） | 节点可异步执行 |
| **错误处理** | Tool 级异常捕获 + 重试 | Agent 级异常处理 | 节点级 try/except，Fallback 边 |
| **调试能力** | 日志回调，中间结果打印 | 调试日志，事件发布 | LangSmith 深度集成，逐步执行 |
| **扩展性** | 插件式 Tool，自定义 Process | Agent 子类化，自定义 Manager | 完全自定义，高度灵活 |
| **学习曲线** | 中低（API 清晰直观） | 中（conversation 简单，agentchat 复杂） | 高（需要理解图计算概念） |
| **生产就绪度** | 中（早期版本） | 高（微软维护，企业级） | 高（LangChain 生态，LangSmith） |
| **GitHub Stars** | 较多 | 较多 | 很多 |
| **源码语言** | Python | Python | Python (Pydantic 驱动) |
| **版本** | >= 0.11 | >= 0.4 (核心), >= 0.1 (agentchat) | >= 0.2 |
| **核心依赖** | Instructor (结构化输出) | 轻量（核心依赖少） | LangChain Core, Pydantic |

## 源码架构深度分析

### 8.6.1 CrewAI 编排引擎 (crewai-orch)

CrewAI 的编排引擎围绕四个核心类构建，其交互关系如下：

```
                        ┌─────────────┐
                        │    Crew     │
                        │  (类/实例)   │
                        │             │
                        │ 管理 Agent   │
                        │ 管理 Task   │
                        │ 定义 Process│
                        │ 执行 Kickoff│
                        └──────┬──────┘
                               │
                ┌──────────────┼────────────────┐
                │              │                 │
         ┌──────▼──────┐ ┌────▼─────┐  ┌────────▼────────┐
         │   Agent     │ │   Task   │  │   Process       │
         │  (实体类)    │ │ (实体类)  │  │  (策略模式)      │
         │             │ │          │  │                 │
         │ role        │ │ description│ │ SequentialProcess │
         │ goal        │ │ agent    │  │ HierarchicalProcess│
         │ backstory   │ │ tools    │  │                 │
         │ tools       │ │ context  │  │ 控制 Task 执行顺序 │
         │ allow_deleg│ │ callback │  │ 处理结果传递       │
         └──────┬──────┘ └────┬─────┘  └────────┬────────┘
                │             │                  │
                └─────────────┼──────────────────┘
                              │
                        ┌─────▼─────┐
                        │   Tool    │
                        │ (插件式)   │
                        │           │
                        │ BaseTool  │
                        │ tool_decorator │
                        │ 参数绑定  │
                        └───────────┘
```

**核心源码流程**（简化的 Kickoff 调用链）：

```
Crew.kickoff()
  ├── _setup_for_kickoff()        # 验证配置，初始化
  ├── _run_sequential_process()   # 顺序流程
  │   ├── for each task:
  │   │   ├── 构造 Agent 上下文（前序 Task 输出）
  │   │   ├── Agent.execute_task(task, context)
  │   │   │   ├── 构造 system_prompt (role+goal+backstory)
  │   │   │   ├── Agent 内部 __call__ → LLM 调用
  │   │   │   └── 如果有工具 → Tool 调用 → 结果回填
  │   │   └── task.output = result
  │   └── return final_output
  ├── _run_hierarchical_process() # 层级流程
  │   ├── Manager Agent 接收所有 Task 描述
  │   ├── Manager Agent 决定 Task 分配和执行顺序
  │   ├── 动态指派 Agent 执行 Task
  │   └── Manager 汇总结果
  └── return CrewOutput
```

**关键源码洞察**：

1. **Process 策略模式**：`SequentialProcess` 和 `HierarchicalProcess` 都继承自 `BaseProcess`。新增自定义流程只需实现 `_execute` 方法。

2. **Agent 调用机制**：Agent 本质上是一个 LLM 调用封装器。`Agent.execute_task()` 将 Task 描述、Agent 配置、工具定义、上下文拼装为一个完整的 Prompt，调用 LLM 后解析回复。

3. **Task 上下文传递**：`Task.context` 可以是其他 Task 的输出列表。框架在每次 Task 执行前自动拉取前序 Task 的输出并注入 Prompt，形成隐式的上下文传递链。

4. **委托机制**：当 `Agent.allow_delegation=True` 时，Agent 如果认为自己无法完成任务，可以在回复中包含委托指令，框架解析后转发到目标 Agent。

### 8.6.2 CrewAI 工具系统 (crewai-tools)

CrewAI 的工具系统是一个插件式的工具注册和执行框架：

```
                    ┌─────────────────────────────────────┐
                    │          CrewAI Tool System          │
                    │                                      │
                    │  所有工具继承自 BaseTool 基类         │
                    │  工具通过 @tool 装饰器或子类化注册    │
                    │  自动生成 OpenAI Tool Schema          │
                    └──────────────────────────────────────┘
                                      │
            ┌─────────────────────────┼─────────────────────────────┐
            │                         │                             │
    ┌───────▼──────────┐   ┌─────────▼────────┐   ┌───────────────▼───┐
    │  BaseTool       │   │  @tool 装饰器     │   │  ToolRouter     │
    │  (抽象基类)      │   │  (快速定义工具)    │   │  (工具路由分发)  │
    │                 │   │                  │   │                  │
    │ name            │   │ def my_tool():   │   │ 根据 Tool 名称   │
    │ description     │   │   "..."          │   │ 路由到对应实现    │
    │ args_schema     │   │   return ...     │   │ 统一错误处理      │
    │ _run()          │   │                  │   │ 结果格式化        │
    │ _arun()         │   └──────────────────┘   └──────────────────┘
    └───────┬──────────┘
            │
    ┌───────┴───────────────────────────────────────────────┐
    │  工具注册生命周期                                       │
    │                                                        │
    │  1. 工具定义 → 2. Schema 生成 → 3. Agent 绑定          │
    │  4. LLM 选择工具 → 5. 参数解析 → 6. 工具执行            │
    │  7. 结果格式化 → 8. 回填到 LLM 上下文 → 9. 继续推理      │
    └─────────────────────────────────────────────────────────┘
```

**关键源码洞察**：

1. **BaseTool 设计**：继承自 LangChain 的 `BaseTool`，主要扩展了参数 Schema 的自动生成能力。`args_schema` 由参数类型注解自动推导为 Pydantic Model。

2. **@tool 装饰器**：一个语法糖，将普通函数转换为 Tool 对象。函数文档字符串成为工具 description，类型注解成为 args_schema。底层调用 `Tool.from_function()`。

3. **工具执行流程**：Agent 通过 OpenAI 的 `tool_calls` 机制选择工具。框架解析 `tool_calls` 参数，路由到具体工具，执行后结果以 `tool` role 消息回填到对话上下文。

4. **错误处理**：`BaseTool._run()` 包裹在 try/except 中，异常信息格式化为工具结果的一部分（不中断 Agent 主流程），Agent 可以选择重试或改用其他工具。

5. **工具缓存**：可选的缓存机制，对相同参数的工具调用直接返回缓存结果，减少 Token 消耗。

### 8.6.3 AutoGen 对话管理层 (autogen-conversation)

AutoGen 的对话管理层实现了多 Agent 之间结构化消息交换的核心机制：

```
                     ┌─────────────────────────────────┐
                     │      GroupChat 对话管理层         │
                     │                                  │
                     │ 管理一组 Agent 的结构化对话        │
                     │ 维护对话历史（消息列表）            │
                     │ 控制发言顺序（轮转/选择/自定义）    │
                     └───────────────┬─────────────────┘
                                     │
            ┌────────────────────────┼────────────────────────┐
            │                        │                        │
    ┌───────▼──────────┐   ┌────────▼────────┐   ┌───────────▼────────┐
    │  GroupChat       │   │  GroupChatManager│   │  Conversationable  │
    │  (数据结构)       │   │  (控制逻辑)      │   │  (协议接口)         │
    │                  │   │                  │   │                    │
    │ agents: list     │   │ 管理对话轮次      │   │ send()             │
    │ messages: list   │   │ 决定谁发言        │   │ receive()          │
    │ max_round: int   │   │ 广播消息          │   │ generate_reply()   │
    │ speaker: str     │   │ 检测终止条件      │   │ a_send() (async)   │
    │ admin_name: str  │   │ 调用 Agent 生成   │   │                    │
    └──────────────────┘   └──────────────────┘   └────────────────────┘
```

**核心源码流程**：

```
GroupChatManager.run_chat()
  ├── GroupChat 初始化（Agent 注册，消息历史加载）
  ├── 对话主循环
  │   ├── select_speaker()
  │   │   ├── 轮转模式: 按顺序选择下一个
  │   │   ├── 自动模式: LLM 根据上下文选择
  │   │   ├── 手动模式: 人工指定
  │   │   └── 随机模式: 从可用 Agent 随机
  │   │
  │   ├── speaker.send(message)
  │   │   ├── Agent.generate_reply()  # 内部 LLM 调用
  │   │   └── return reply
  │   │
  │   ├── GroupChat.broadcast(reply)
  │   │   └── 所有 Agent.receive(reply)  # 更新内部上下文
  │   │
  │   ├── 添加到消息历史
  │   ├── 检查终止条件
  │   │   ├── max_round 达到
  │   │   ├── Agent 发送 "TERMINATE" 消息
  │   │   └── 管理员手动终止
  │   └── 如果未终止 → 继续循环
  └── 返回完整对话历史和最终结果
```

**关键源码洞察**：

1. **Conversationable 协议**：这是一个 Python 协议类（PEP 544），定义了 Agent 通信的标准接口（`send`, `receive`, `generate_reply`）。任何实现了该协议的类都可以作为 GroupChat 的参与者，包括非 LLM 的 Agent（如人类用户代理）。

2. **消息格式**：消息是字典列表，格式兼容 OpenAI API：`{"role": "user"/"assistant"/"function", "content": ..., "name": agent_name}`。这种设计降低了与 LLM API 之间的序列化开销。

3. **select_speaker 模式**：这是 GroupChat 的核心差异点。"自动"模式使用一个内置的 LLM 裁判（Select Speaker Agent）来分析对话上下文并选择最合适的下一个发言者，比其他固定轮转模式更灵活但也更昂贵。

4. **终止检测**：Manager 在每轮对话后检查终止条件。关键机制是 Agent 可以在消息内容中包含隐藏的终止标记（`"TERMINATE"`），Manager 解析消息时检测到此标记即终止对话。

5. **广播机制**：`GroupChat.broadcast()` 不是真正的网络广播，而是遍历所有 Agent 调用其 `receive()` 方法，让每个 Agent 内部更新自己的上下文窗口，确保所有 Agent 感知对话进展。

### 8.6.4 AutoGen AgentChat (autogen-agentchat)

AutoGen 0.4+ 引入的 AgentChat 是对对话管理的更高层抽象，引入了 Team、Handoff 等新概念：

```
                    ┌────────────────────────────────────┐
                    │         AgentChat 新架构             │
                    │   (0.4+ 引入，替代旧 Conversation)  │
                    │                                     │
                    │  Team : 管理一组 Agent 的执行         │
                    │  Agent : 增强的 Agent 抽象           │
                    │  Handoff : Agent 间控制权转交         │
                    │  ToolUse : 统一工具调用机制           │
                    └──────────────┬──────────────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
┌───────▼────────┐       ┌────────▼────────┐       ┌─────────▼────────┐
│  BaseTeam      │       │  BaseAgent      │       │  Handoff         │
│                │       │                 │       │                  │
│ run()          │       │ on_messages()   │       │ source: Agent    │
│ run_stream()   │       │ on_reset()      │       │ target: Agent    │
│                │       │ tool_use: bool  │       │ message: str     │
│ RoundRobinGroup│       │ handoffs: list  │       │ 控制权转交        │
│ SelectGroup    │       │                 │       │ 上下文传递        │
│ SwarmGroup     │       │ ToolUseAgent    │       │                  │
│ AgentRuntime   │       │ CodeExecution   │       │                  │
└────────────────┘       └─────────────────┘       └──────────────────┘
```

**核心源码流程**：

```
Team.run(task)
  ├── Team 初始化（Agent 注册，初始化运行上下文）
  ├── 根据 Team 类型启动执行策略
  │   ├── RoundRobinGroup
  │   │   └── 轮流向每个 Agent 发送消息
  │   ├── SelectGroup
  │   │   └── LLM 选择下一个发言的 Agent
  │   └── SwarmGroup
  │       └── Handoff 驱动：Agent 主动转交控制权
  │
  ├── 消息传递循环
  │   ├── Agent.on_messages(messages)
  │   │   ├── Agent 内部处理消息
  │   │   ├── 如果有 ToolUse → 执行工具
  │   │   ├── 如果有 Handoff → 准备转交
  │   │   └── 返回 AgentResponse (消息 + 状态)
  │   │
  │   ├── 检查 AgentResponse 中的手off 请求
  │   │   └── Handoff: 暂停当前 Agent，转交控制权
  │   │       └── target Agent.on_messages(context)
  │   │
  │   ├── 收集消息 → 更新对话上下文
  │   └── 检查终止条件
  │       └── max_turns / 所有 Agent 完成 / StopMessage
  │
  └── 返回 TaskResult (最终消息 + 完整历史)
```

**关键源码洞察**：

1. **Team 分层架构**：`BaseTeam` 定义了通用的 `run()` 契约，具体策略由子类实现。`RoundRobinGroup` 是简单的轮转，`SelectGroup` 使用 LLM 选择，`SwarmGroup` 通过 Handoff 实现控制权转交。

2. **Handoff 机制**：这是 AgentChat 最独特的设计。Agent 在回复中可以附带 `HandoffMessage`，框架解析后将控制权转交给目标 Agent。目标 Agent 接收当前上下文继续执行。这模拟了人类团队中"这个你更专业，你来处理"的场景。

3. **AgentResponse**：Agent 的 `on_messages()` 返回一个结构化结果，包含：`response_messages`（实际回复）、`inner_messages`（工具调用/内部思考）、`agent_id`、`handoff` 指示。这种结构化输出使 Team 可以做出精确的后续决策。

4. **流式执行**：`Team.run_stream()` 使用 Python 异步生成器，每个 Agent 的回复通过事件流式输出。观察者可以订阅不同粒度的事件（消息级、Token 级、状态变更级）。

5. **终止条件**：比 Conversation 更丰富。除了 max_turns，还支持 `StopMessage` 类型的消息（Agent 显式停止）、`TextMentionTermination`（检测到关键词）、`MaxMessageTermination`（消息数上限）、`OrTermination` / `AndTermination`（组合条件）。

### 8.6.5 LangGraph 多 Agent 图状态机 (langgraph-multi)

LangGraph 将多 Agent 协作建模为有向图，节点是 Agent 逻辑，边是数据流路径：

```
                    ┌─────────────────────────────────────┐
                    │     LangGraph Multi-Agent Graph      │
                    │                                      │
                    │  StateGraph : 定义图拓扑和状态        │
                    │  Node       : Agent 处理逻辑         │
                    │  Edge       : 节点间数据流路径        │
                    │  ConditionalEdge : 基于状态的条件路由  │
                    │  Checkpointer: 持久化快照             │
                    └──────────────────────────────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
 ┌───────▼─────────┐    ┌─────────▼─────────┐    ┌──────────▼──────────┐
 │  StateGraph     │    │  MessageGraph     │    │  Checkpointer      │
 │  (通用图)       │    │  (对话专用图)      │    │  (状态持久化)       │
 │                 │    │                   │    │                    │
 │ class StateGraph│    │ class MessageGraph│    │ MemorySaver        │
 │   <State>       │    │   extends         │    │ SqliteSaver        │
 │                 │    │   StateGraph       │    │ PostgresSaver      │
 │ state: TypedDict│    │   <MessagesState> │    │ 快照 + 恢复        │
 │ add_node()      │    │                   │    │ 分支 + 回放        │
 │ add_edge()      │    │ messages: list    │    │ 时间旅行调试       │
 │ add_conditional │    │ add_messages      │    │                    │
 │   _edge()       │    │   reducer         │    │                    │
 │ set_entry_point │    │                   │    │                    │
 │ set_finish_point│    │                   │    │                    │
 │ compile()       │    │                   │    │                    │
 │ invoke()        │    │                   │    │                    │
 └─────────────────┘    └───────────────────┘    └────────────────────┘
```

**多 Agent 协作的图拓扑示例**：

```python
# 典型的多 Agent Supervisor 图模式 (源码层面的结构)

#     ┌──────────────────┐
#     │     __start__     │  ← 入口点
#     └────────┬─────────┘
#              │
#     ┌────────▼─────────┐
#     │   supervisor      │  ← 主管 Agent: 决定下一步
#     │   (LLM Router)    │     根据当前状态选择执行节点
#     └──┬────┬────┬────┬─┘
#        │    │    │    │
#   ┌────▼┐ ┌▼───┐ ┌▼──┐ │  ← 多个 Worker Agent 节点
#   │ res │ │web │ │code│ │     (每种能力一个节点)
#   │earch│ │    │ │    │ │
#   └──┬──┘ └┬───┘ └┬──┘ │
#      │      │      │    │
#     ┌▼──────▼──────▼────▼─┐
#     │    aggregate        │  ← 聚合节点: 收集所有结果
#     └──────────┬──────────┘
#                │
#     ┌──────────▼──────────┐
#     │      __end__        │  ← 结束点
#     └─────────────────────┘
```

**核心源码流程**：

```
graph = StateGraph(AgentState)      # 定义状态 Schema

# 添加节点
graph.add_node("supervisor", supervisor_node)    # 主管节点
graph.add_node("researcher", research_node)      # 研究员节点
graph.add_node("coder", code_node)               # 编码节点
graph.add_node("aggregator", aggregate_node)     # 聚合节点

# 添加边
graph.set_entry_point("supervisor")              # 入口 → supervisor

# 条件边: supervisor 根据状态路由到不同 Worker
graph.add_conditional_edges(
    "supervisor",
    router_function,        # 接收 State，返回下一个节点名称
    {
        "researcher": "researcher",
        "coder": "coder",
        "aggregate": "aggregator",
        "FINISH": END,
    }
)

# 固定边: Worker 完成后回到 supervisor
graph.add_edge("researcher", "supervisor")
graph.add_edge("coder", "supervisor")

# 结束
graph.add_edge("aggregator", END)

# 编译（添加 Checkpointer）
app = graph.compile(checkpointer=MemorySaver())

# 执行
result = app.invoke(
    {"messages": [HumanMessage(content="分析这个 API 并生成调用代码")]},
    config={"configurable": {"thread_id": "thread-1"}}
)
```

**关键源码洞察**：

1. **StateGraph 泛型设计**：`StateGraph[S]` 是参数化类型，`S` 是状态 Schema（TypedDict 或 Pydantic Model）。这提供了编译时类型检查和 IDE 自动补全。每个节点接收 `S` 返回 `S` 的部分更新。

2. **Reducer 机制**：这是 LangGraph 最精巧的设计之一。状态字段可以关联 reducer 函数（如 `operator.add` 或自定义函数），用于处理同一字段的并发更新。`add_messages` 是内置的 Message 列表 reducer，自动处理消息追加、ID 去重和更新。源码中每个 Node 的返回通过 reducer 合并到全局状态。

3. **ConditionalEdge 路由函数**：路由函数接收完整 State，返回字符串（指向目标节点名称）。这使得路由逻辑完全自定义——可以基于消息内容、工具调用结果、计数器等任意状态信息做决策。源码中路由函数在 Supervisor 模式中通常是一个 LLM 调用，让 LLM 决定下一步。

4. **Checkpointer 状态持久化**：Checkpointer 在每次 Node 执行后保存状态快照。这支持了：时间旅行调试（回退到任意历史状态）、人工介入（暂停执行等待输入）、故障恢复（从最近快照恢复）。`MemorySaver` 是内存实现，`SqliteSaver` / `PostgresSaver` 是持久化实现。源码中 Checkpointer 的核心是 `put()`（写入快照）、`get()`（读取快照）、`list()`（列出所有快照）。

5. **流式输出**：LangGraph 的流式支持分为多个层级：`stream_mode="values"`（节点输出）、`stream_mode="updates"`（状态增量）、`stream_mode="custom"`（自定义事件）。底层通过 Channel 机制实现——每个 Node 的输出通过 Channel 传递给下游。

6. **节点函数签名**：每个 Node 是一个 Callable `(State) -> Partial<State>`。典型实现：
   ```python
   def agent_node(state: AgentState, name: str, llm: ChatOpenAI) -> dict:
       # 1. 读取当前状态
       messages = state["messages"]
       # 2. 调用 LLM
       response = llm.invoke(messages)
       # 3. 更新状态（只返回变更部分）
       return {"messages": [response]}
   ```

### 8.6.6 框架对比 (framework-compare)

#### 代码结构对比（同一任务，三个框架的实现差异）

以"多 Agent 研究 + 写作"任务为例：

```
任务：Agent A 搜索资料 → Agent B 编写大纲 → Agent C 撰写报告
```

**CrewAI 实现**（任务编排风格）：

```python
# CrewAI: 定义 Agent → 定义 Task → 组成 Crew → 启动
researcher = Agent(role="研究员", goal="搜索资料", ...)
outliner = Agent(role="大纲编写", goal="撰写大纲", ...)
writer = Agent(role="报告撰写", goal="完成报告", ...)

task1 = Task(description="搜索相关资料", agent=researcher)
task2 = Task(description="撰写大纲", agent=outliner)
task3 = Task(description="完成报告", agent=writer)

crew = Crew(agents=[researcher, outliner, writer],
            tasks=[task1, task2, task3],
            process=Process.sequential)
crew.kickoff()
```

**AutoGen 实现**（对话驱动风格）：

```python
# AutoGen: Agent 注册到 GroupChat → Manager 协调对话
researcher = AssistantAgent(name="researcher", ...)
outliner = AssistantAgent(name="outliner", ...)
writer = AssistantAgent(name="writer", ...)
user_proxy = UserProxyAgent(name="user", ...)

groupchat = GroupChat(
    agents=[user_proxy, researcher, outliner, writer],
    messages=[],
    max_round=10
)
manager = GroupChatManager(groupchat=groupchat)
user_proxy.initiate_chat(manager, message="写一份关于 AI 的报告")
# 对话自然流转，Manager 决定发言顺序
```

**LangGraph 实现**（图状态机风格）：

```python
# LangGraph: 定义 State → 定义 Node → 定义边 → 编译执行
class ReportState(TypedDict):
    topic: str
    research: str
    outline: str
    report: str
    messages: list

def research_node(state: ReportState) -> dict:
    # 调用 LLM 搜索，返回 {"research": ..., "messages": [...]}
def outline_node(state: ReportState) -> dict:
    # 读取 research，生成 outline
def writer_node(state: ReportState) -> dict:
    # 读取 research + outline，生成 report

graph = StateGraph(ReportState)
graph.add_node("research", research_node)
graph.add_node("outline", outline_node)
graph.add_node("writer", writer_node)
graph.add_edge("research", "outline")
graph.add_edge("outline", "writer")
graph.set_entry_point("research")
graph.set_finish_point("writer")

app = graph.compile()
app.invoke({"topic": "AI 的未来发展"})
```

#### 框架适用场景矩阵

| 场景特征 | CrewAI | AutoGen | LangGraph |
|----------|--------|---------|-----------|
| **简单顺序流水线** | 最合适，代码最少 | 可以，但对话模型显得重 | 可以，但图模型显得重 |
| **需要人类参与** | 有限支持 | 强（UserProxyAgent） | 强（Interrupt + 断点） |
| **Agent 层级管理** | 原生支持（层级 Process） | 需要自定义 | 图拓扑可模拟 |
| **动态 Agent 发现** | 不原生支持 | GroupChat 可动态加入 | 图拓扑需预定义 |
| **条件路由/决策** | 有限（Task 条件） | 通过 Manager 选择 | 最强（ConditionalEdge） |
| **复杂循环/迭代** | 不原生 | 通过对话循环实现 | 强（图可以有环） |
| **高性能/低延迟** | 中等 | 中等 | 依赖图复杂度 |
| **调试和可观察性** | 日志 | 日志+事件 | LangSmith 集成 |
| **状态持久化** | 可选 DB | 对话历史 | Checkpointer 原生 |
| **流式输出** | 有限 | AgentChat 支持 | 原生支持 |
| **工具集成** | 强（BaseTool 生态） | 中等 | 自定义 |
| **快速原型** | 最合适 | 合适 | 需要理解图概念 |
| **生产部署** | 发展中 | 较成熟 | 较成熟 |

#### 源码架构风格对比

| 维度 | CrewAI | AutoGen | LangGraph |
|------|--------|---------|-----------|
| **代码库规模** | 中等（~5K 行核心） | 中等（~8K 行核心） | 较小（~4K 行核心，依赖 LangChain） |
| **设计模式** | 策略模式、装饰器模式 | 协议模式、策略模式 | 图模式、Reducer 模式 |
| **依赖注入** | 显式构造 | 构造 + 协议 | 参数传递 |
| **类型系统** | Pydantic | Pydantic + Protocol | TypedDict + Pydantic |
| **异步支持** | 同步为主 | 同步 + 异步 | 同步 + 异步 |
| **配置方式** | YAML / 代码 | 代码 | 代码 |
| **扩展点** | 自定义 Tool、Process | 自定义 Agent、Manager | 自定义 Node、Edge、Checkpointer |

## 核心架构差异深度分析

### 1. Agent 定义方式

| 框架 | 定义方式 | 灵活性 | 可配置性 |
|------|---------|--------|---------|
| CrewAI | `Agent(role, goal, backstory, tools, llm)` | 中（面向角色） | 高（角色/目标/背景故事） |
| AutoGen | `AssistantAgent(name, system_message, llm_config)` | 中高（协议接口） | 中（System Message 驱动） |
| LangGraph | 自定义函数 `(State) -> State` | 最高（任意逻辑） | 最低（无预定义 Agent 类） |

**工程观察**：CrewAI 的 Agent 定义最"结构化"，适合角色明确的场景；AutoGen 的 Agent 通过 `system_message` 实现角色定义，灵活性居中；LangGraph 没有独立的 Agent 概念——一个 Node 可以是一个 Agent、一个函数、或者任意处理逻辑，自由度最高但结构也最弱。

### 2. 任务分配方式

| 框架 | 机制 | 静态/动态 | 代码体现 |
|------|------|----------|---------|
| CrewAI | Process 预定义顺序/层级 | 静态（Task 列表固定） | Task.agent 指定 |
| AutoGen | Manager 运行时选择 | 动态（Manager 决策） | GroupChatManager.select_speaker() |
| LangGraph | 图拓扑 + 条件边 | 混合（节点固定，路由动态） | add_conditional_edges() |

**工程观察**：CrewAI 最适合确定性流程（Task 列表在启动时已知）；AutoGen 适合探索式对话（谁发言由对话进展决定）；LangGraph 位于两者之间——节点拓扑固定，但路径由运行时状态决定。

### 3. 工具执行机制

| 方面 | CrewAI | AutoGen | LangGraph |
|------|--------|---------|-----------|
| 工具定义 | BaseTool 子类 / @tool | Tool 函数 | 无内置抽象 |
| Schema 生成 | 自动（参数注解 → JSON Schema） | 手动 / 自动 | 节点内自行处理 |
| 调用方式 | Agent 内部自动调用 | Agent 内部自动调用 | 节点内显式调用 |
| 结果回填 | 自动（tool 消息回填） | 自动 | 手动处理 |
| 错误传播 | 捕获后格式化 | 捕获后重试 | 节点内自定义 |

**工程观察**：CrewAI 和 AutoGen 提供了完善的工具抽象层，开发体验好但"黑箱"程度高。LangGraph 不提供工具抽象——工具调用是节点内部逻辑，灵活性最高但需要开发者自己实现调用循环。

### 4. 消息路由和通信机制

```
CrewAI: 隐式路由
  Task1 输出 ──→ 注入 Task2 上下文 ──→ Task3
  （Agent 之间不直接通信，通过 Task 输出传递数据）

AutoGen Conversation: 广播路由
  Agent A ──→ GroupChat ──→ broadcast ──→ Agent B, C, D...
  （消息通过 Manager 广播到所有 Agent）

AutoGen AgentChat: Handoff 路由
  Agent A ──→ Handoff ──→ Agent B ──→ Handoff ──→ Agent C
  （控制权通过 Handoff 消息在 Agent 间转交）

LangGraph: 图路由
  Node A ──→ Edge ──→ Node B
  （通过 StateGraph 的边定义数据流路径）
```

### 5. 错误处理与容错

| 框架 | 层级 | 机制 | 粒度 |
|------|------|------|------|
| CrewAI | Tool / Agent / Crew | try/except + 重试 + 跳过 | 工具级重试，任务级跳过 |
| AutoGen | Agent / Conversation | 异常捕获 + 恢复 + Manager 干预 | Agent 级降级 |
| LangGraph | Node / Edge / Graph | try/except + Fallback Edge + Checkpoint 恢复 | 节点级回退 + 图级重放 |

**工程观察**：

- **CrewAI** 在 Tool 层最完善——自动重试、异常格式化、缓存降级。但在 Process 层较简单——一个 Task 失败可能导致整链中断。
- **AutoGen** 通过对话 Manager 实现容错——某个 Agent 失败时，Manager 可以选择跳过或通知其他 Agent 处理。但异常处理分散在各 Agent 实现中，一致性挑战较大。
- **LangGraph** 依靠 Checkpointer 实现最强大的容错——可以在任意节点暂停、检查状态、修复后恢复。`Fallback Edge` 允许为主路径定义备用路径。

### 6. 状态持久化机制

```
CrewAI:
  ┌──────────┐     ┌───────────┐     ┌─────────────┐
  │ Crew     │────▶│ 可选 DB   │────▶│ Task 结果     │
  │ 执行完成  │     │ PostgreSQL│     │ SQLite 存储  │
  └──────────┘     │ SQLite    │     │ 最终结果      │
                   └───────────┘     └─────────────┘
  说明: 仅持久化最终结果，不支持中间状态保存

AutoGen:
  ┌────────────┐    ┌──────────────┐   ┌──────────────┐
  │ Conversation│───▶│ 消息列表      │──▶│ 对话历史文件   │
  │ 运行时      │    │ (内存中维护)  │   │ JSON / DB    │
  └────────────┘    │ 可导出到文件   │   │              │
                    └──────────────┘   └──────────────┘
  说明: 持久化全部对话历史，支持对话回放，但不支持状态分支

LangGraph:
  ┌──────────┐    ┌──────────────┐   ┌────────────────┐
  │ StateGraph│───▶│ Checkpointer │──▶│ 状态快照序列     │
  │ 执行中    │    │              │   │                │
  │ 每步保存  │    │ MemorySaver  │   │ 快照1 → 快照2  │
  └──────────┘    │ SqliteSaver  │   │ → 快照3 ...    │
                   │ PostgresSaver│   │ 形成状态链       │
                   └──────────────┘   └────────────────┘
  说明: 持久化每一步的状态快照，支持时间旅行、分支、回放
```

## 关键工程洞察

### CrewAI 做得好的地方

1. **开发体验优秀**：Agent/Task/Crew 三层抽象直观易懂，即使没有多 Agent 背景的开发者也能快速上手。YAML 配置支持让非技术人员也能参与定义。

2. **角色扮演深度**：`role` + `goal` + `backstory` 三件套提供了比简单 System Prompt 更丰富的 Agent 人格化能力。源码中这三者被精心拼接为结构化的系统提示词。

3. **工具生态**：`@tool` 装饰器和 `BaseTool` 子类化机制让工具定义非常简洁。社区提供的 50+ 内置工具覆盖了常见需求。

4. **Process 可扩展**：策略模式使得添加新的执行流程（如并行 Process、DAG Process）不需要修改核心代码。

### CrewAI 的局限

1. **Task 是静态的**：Task 列表在 Kickoff 前就固定了，无法在运行时动态创建或修改 Task。这对于需要"基于上一步结果动态决定下一步"的场景不够灵活。

2. **状态管理简单**：上下文纯粹通过 Task 输出链式传递，没有全局状态的概念。复杂场景下信息容易丢失或重复。

3. **层级流程的黑箱**：`HierarchicalProcess` 中 Manager Agent 的任务分配逻辑完全不可见，开发者无法控制或干预分配决策。

4. **流式支持弱**：没有原生流式输出支持，所有结果在 Task 完成后才返回，不适合长任务。

### AutoGen 做得好的地方

1. **对话模型自然**：多 Agent 协作建模为对话，符合人类协作的直觉。`GroupChat` 的"多人在线聊天室"隐喻降低了理解成本。

2. **人机协作出色**：`UserProxyAgent` 使得人类可以自然地参与 Agent 对话，`input()` 和确认机制提供了良好的人机交互体验。

3. **选择性发言灵活**：`select_speaker` 的多种模式（轮转/自动/手动/随机）覆盖了从确定到探索的完整光谱。

4. **AgentChat 架构清晰**：0.4+ 的 Team/Handoff/ToolUse 抽象在保持灵活性的同时提供了更好的结构。

### AutoGen 的局限

1. **对话效率问题**：所有消息广播到所有 Agent，不论是否需要。在大规模 Agent 群中 Token 浪费严重。

2. **发言选择的开销**：自动模式下，每轮都需要额外调用 LLM 来选择下一个发言者，增加了成本。

3. **长对话上下文膨胀**：对话历史线性增长，没有内置的摘要或压缩机制。长对话中 Token 消耗呈 O(n) 增长。

4. **两套 API 并存**：0.4+ 的 AgentChat 尚未完全替代旧的 Conversation API，出现两套体系共存的情况，增加了学习成本。

### LangGraph 做得好的地方

1. **状态管理的 sophistication**：Reducer 机制优雅地解决了图执行中的状态合并问题。时间旅行调试用 Checkpointer 实现，是一个工程杰作。

2. **表达能力最强**：条件边、循环边、分支边可以表达任意复杂的多 Agent 协作拓扑。没有其他框架能与之匹敌。

3. **流式架构一流**：多层级流式输出（values / updates / custom）覆盖了从简单到复杂的所有用例。

4. **生产级调试**：LangSmith 集成提供了从前端 UI 对图执行的完整可观测性，包括状态快照浏览、节点级延迟分析、Token 消耗追踪。

### LangGraph 的局限

1. **学习曲线陡峭**：需要理解图计算概念（状态、reducer、通道）才能有效使用。对于简单场景显得过度工程。

2. **缺乏高层抽象**：没有 Agent 类、没有 Task 类、没有工具管理——一切都需要开发者自己构建。这在带来灵活性的同时也带来了从零开始搭建的成本。

3. **Debug 困难（无 LangSmith 时）**：图执行是"黑盒推进"——当节点数量增多时，没有 LangSmith 纯凭日志很难跟踪问题。

4. **状态 Schema 维护成本**：随着图复杂度增加，状态 TypedDict 不断膨胀。Reducer 冲突（多个节点修改同一字段）可能产生难以排查的 Bug。

## 框架选型决策树

```
                               ┌─────────────────────────────────────┐
                               │ 选择多 Agent 框架：从哪里开始？       │
                               │                                     │
                               │ 你需要多 Agent 系统解决什么问题？     │
                               └──────────────────┬──────────────────┘
                                                  │
                         ┌────────────────────────┼────────────────────────┐
                         │                        │                        │
                    ┌────▼────┐             ┌─────▼─────┐           ┌─────▼─────┐
                    │ 简单的   │             │ 需要复杂   │           │ 想要最大   │
                    │ 顺序流水线│             │ 条件路由   │           │ 灵活性     │
                    │ 确定性高  │             │ 动态决策   │           │ 自定义    │
                    │ 快速交付  │             │ 分支/循环  │           │ 控制流    │
                    └────┬────┘             └─────┬─────┘           └─────┬─────┘
                         │                        │                        │
                         ▼                        ▼                        ▼
                   ┌──────────┐            ┌─────────────┐          ┌────────────┐
                   │  CrewAI  │            │  LangGraph  │          │  LangGraph │
                   │          │            │             │          │            │
                   │ 最佳入门  │            │ 灵活的图拓扑 │          │ 完全控制   │
                   │          │            │ Conditional │          │ 任何模式   │
                   │ 代码最简  │            │ Edge 路由   │          │ 但工作量   │
                   └──────────┘            └─────────────┘          │ 最大      │
                         │                                           └────────────┘
                         │
              ┌──────────┼──────────┐
              │                     │
         ┌────▼────┐          ┌─────▼─────┐
         │ 需要     │          │ 需要      │
         │ 人类参与  │          │ 对话式    │
         │ 交互式   │          │ 协作      │
         └────┬────┘          └─────┬─────┘
              │                     │
              ▼                     ▼
        ┌──────────────┐     ┌──────────────┐
        │  AutoGen     │     │  AutoGen     │
        │              │     │              │
        │ UserProxy    │     │ GroupChat    │
        │ 人机协作     │     │ 对话流       │
        └──────────────┘     └──────────────┘

  附加考量：

  团队背景:
  ├─ 熟悉 LangChain 生态 → LangGraph 学习成本最低
  ├─ 习惯面向对象设计   → CrewAI / AutoGen 更舒适
  └─ 函数式编程偏好     → LangGraph 的 Reducer 风格更自然

  生产环境要求:
  ├─ 需要状态持久化 / 故障恢复  → LangGraph (Checkpointer)
  ├─ 需要人机协作审批流         → AutoGen (UserProxyAgent)
  └─ 快速 MVP / PoC            → CrewAI (开发速度最快)

  性能与成本:
  ├─ Token 预算敏感            → LangGraph (精确控制每次 LLM 调用)
  ├─ 需要流式响应               → LangGraph / AutoGen AgentChat
  └─ 简单任务避免框架 overhead   → 考虑不使用框架，直接用 LLM API + 代码
```

## 与 8.1-8.5 的关系

本节是 8 Multi-Agent 模块的"工程实现篇"。之前五节从设计模式层面讨论了多 Agent 系统"应该怎么做"，本节从源码层面展示主流框架"实际上是怎么做的"。

### 向上追溯：模式在框架中的体现

```
8.1 Communication ───→ AutoGen Conversation 的消息路由
(通信模式)               GroupChat.broadcast() 实现广播通信
                        AutoGen 的消息格式复用 OpenAI API 格式

8.2 Coordination ────→ CrewAI Process 的 Task 执行顺序
(协调机制)               LangGraph ConditionalEdge 实现协调决策
                        AutoGen GroupChatManager 的发言协调

8.3 Delegation ──────→ CrewAI Agent.allow_delegation 委托
(委派机制)               AutoGen Handoff 控制权转交
                        LangGraph Supervisor → Worker 委派图

8.4 Debate & Negot. ─→ AutoGen GroupChat 的多 Agent 对话
(辩论协商)               select_speaker 自动模式 = Manager 裁决
                        辩论轮次 = groupchat.max_round

8.5 Swarm Patterns ───→ LangGraph 可以表达所有 7 种群体模式
(群体组织)               Manager-Worker = Supervisor + Worker Nodes
                        Pipeline = 线性 Edge 链
                        Parliament = Mes sageGraph 广播讨论
                        Market = 自定义路由函数实现竞标
```

### 具体的模式-框架映射表

| 8.x 模式 | CrewAI 实现 | AutoGen 实现 | LangGraph 实现 |
|-----------|-------------|--------------|----------------|
| **8.1 广播通信** | Task 间传递（非广播） | GroupChat.broadcast() | State 共享（所有 Node 可见） |
| **8.1 定向通信** | 不原生支持 | Agent.send(receiver, msg) | Edge 定义数据流方向 |
| **8.2 中心化协调** | HierarchicalProcess (Manager) | GroupChatManager | Supervisor Node |
| **8.2 去中心化协调** | SequentialProcess (对等) | GroupChat 自动模式 | Mesh 拓扑 (自定义) |
| **8.3 直接委派** | Task.agent = Agent | Agent.send() | Node A → Node B |
| **8.3 中介委派** | 不原生支持 | GroupChatManager | Supervisor Node 路由 |
| **8.4 结构化辩论** | 不原生支持 | RoundRobinGroup (轮转发言) | 自定义图拓扑 + 辩论 Node |
| **8.4 评审裁决** | 不原生支持 | select_speaker 自动模式 | Supervisor 作为评审 |
| **8.5 Manager-Worker** | Process.hierarchical + Agents | Team + Handoff | Supervisor + Worker 子图 |
| **8.5 Pipeline** | Process.sequential | 不原生 | 线性 Edge 链 |
| **8.5 Parliament** | 不原生支持 | GroupChat + 投票 | 广播通信 + 聚合 |
| **8.5 Market** | 不原生支持 | 不原生支持 | 自定义竞标 Node + 路由 |

### 跨章节连接

- **7.1 Task Decomposition**：CrewAI 的 HierarchicalProcess 中 Manager 做任务分解的方法论基础；LangGraph 中任务分解可以建模为 Agent Node 的输出多路分发。

- **7.2 ReAct Pattern**：三个框架的 Agent 内部都运行 ReAct 循环。CrewAI Agent 和 AutoGen AssistantAgent 的 `generate_reply()` 内部是标准的 ReAct 实现。

- **7.3 Reflection**：AutoGen 的 GroupChat 对话中，Agent 收到其他 Agent 的回复后可以进行自我反思。LangGraph 的循环边天然支持 Reflection 模式（A → B → A 形成反思环）。

- **7.4 Plan-and-Execute**：CrewAI 的 HierarchicalProcess 是 Plan-and-Execute 的直接体现。LangGraph 中 Plan Node → Execute Node → Verify Node 是更灵活的 Plan-and-Execute。

- **7.6 Uncertainty**：AutoGen GroupChat 中 Agent 表达不确定时，Manager 可以选择引入更多 Agent 参与讨论。LangGraph 的条件边可以根据置信度路由到不同的处理路径。

- **7.9 Failover**：LangGraph 的 Checkpointer + Fallback Edge 是最完善的故障转移机制。CrewAI 的 Task 重试是基本的故障转移。

- **11 Evaluation**：多 Agent 系统的评估指标——CrewAI 注重 Task 完成率，AutoGen 关注对话轮次/质量，LangGraph 更灵活但需要自定义评估。

## 框架学习路径建议

```
初学者（1-2 周快速入门）
  │
  ├── 1. CrewAI (最快上手)
  │      ├── 理解 Agent / Task / Crew 三件套
  │      ├── 掌握 @tool 装饰器
  │      └── 实现一个 3-Agent 顺序流水线
  │
  └── 2. AutoGen Conversation (对话模式)
         ├── 理解 GroupChat / GroupChatManager
         ├── 掌握 UserProxyAgent 人机协作
         └── 实现一个 2-Agent 对话解决问题

中级（2-4 周深入）
  │
  ├── 1. AutoGen AgentChat (新架构)
  │      ├── 理解 Team / Handoff / ToolUse
  │      ├── 掌握流式执行和终止条件
  │      └── 实现 Handoff 驱动的 Agent 团队
  │
  └── 2. LangGraph 基础 (图思维)
         ├── 理解 StateGraph / Node / Edge / ConditionalEdge
         ├── 掌握 Reducer 和 Checkpointer
         └── 实现 Supervisor + Worker 模式

高级（4-8 周融会贯通）
  │
  └── 1. LangGraph 深度
         ├── 复杂图拓扑设计（子图、递归）
         ├── 自定义 Checkpointer 和 Channel
         ├── 与 LangSmith 集成调试
         └── 实现生产级多 Agent 系统
  │
  └── 2. 框架源码贡献
         ├── 阅读核心源码（三个框架）
         ├── 理解设计决策的权衡逻辑
         └── 尝试为社区提交 PR
```

## 选型的元原则

最终，选择哪个框架不取决于技术参数，而取决于：

```
  框架选择 = f(问题复杂度, 团队技能, 时间约束, 生产要求, 生态偏好)

  原则 1: 最简单的满足需求 = 最佳选择
  原则 2: 预计未来复杂度增长 → 选择更能扩展的框架
  原则 3: 团队熟悉的技术栈 → 选择学习成本最低的框架
  原则 4: 生产环境部署 → 优先考虑调试和监控能力
  原则 5: 没有最好的框架，只有最合适的框架
```

对于大多数项目：**从 CrewAI 开始原型，复杂度增长时迁移到 LangGraph**，是一条经过验证的稳妥路径。
