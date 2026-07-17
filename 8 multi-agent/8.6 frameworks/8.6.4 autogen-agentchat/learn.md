# 8.6.4 AutoGen AgentChat 协议分析

## 1. 简单介绍

AgentChat 是 AutoGen 的高层 API，提供了 agent teams、handoff 机制和结构化多智能体对话能力。相比 AutoGen 早期的 ConversableAgent API，AgentChat 引入了更清晰的角色抽象和编排模式。

核心定位：AgentChat 是一组构建在 `autogen_core` 底层运行时之上的高级抽象层，它封装了消息路由、对话管理和团队协作的复杂性，让开发者可以用声明式的方式定义多智能体系统。

```
AutoGen 架构分层:

        +-----------------------------------------+
        |          AgentChat (高层 API)             |
        |  AssistantAgent, ToolAgent, Team, Handoff |
        +-----------------------------------------+
        |        autogen_agentchat 包              |
        +-----------------------------------------+
        |          Core (底层 API)                  |
        |  AgentRuntime, Messaging, CoreAgent       |
        +-----------------------------------------+
        |        autogen_core 包                   |
        +-----------------------------------------+
```

AgentChat 解决了原始 AutoGen 中的几个核心痛点：
- 繁琐的手动消息路由配置
- 缺乏团队/编排结构
- agent 间跳转需要手动编码
- 会话管理不透明

---

## 2. AgentChat 架构演进

### 从 ConversableAgent 到 AgentChat

AutoGen v0.2 (原始):
```
ConversableAgent (基类)
  |-- AssistantAgent
  |-- UserProxyAgent
  |-- GroupChat (手动管理)
```

AutoGen v0.4+ (AgentChat):
```
ChatAgent (协议接口)
  |-- BaseChatAgent (抽象基类)
       |-- AssistantAgent
       |-- ToolAgent
       |-- CodeExecutorAgent
       |-- SocietyOfMindAgent
```

### 核心改进

| 维度 | ConversableAgent (v0.2) | AgentChat (v0.4+) |
|------|------------------------|--------------------|
| 消息路由 | 手动 `send()/receive()` | 自动由 Team 管理 |
| Agent 类型 | 通用 ConversableAgent | 专用类型 (Tool, Code, Assistant) |
| 团队编排 | GroupChat + 手动轮转 | RoundRobin/Selector/Swarm Team |
| Agent 跳转 | 手动编码 | 内置 Handoff 协议 |
| 终止条件 | 手动检查 | 声明式 TerminationCondition |
| 状态管理 | 无标准接口 | save_state/load_state |

### AgentChat 解决的问题

1. **配置简洁化**: Agent v0.2 需要大量 `send/receive` 调用来编排对话; AgentChat 只需声明 Team 类型即可。
2. **类型明确化**: ToolAgent 只执行 tools, CodeExecutorAgent 只执行代码, 职责清晰。
3. **团队结构化**: Team 封装了对话管理循环, 开发者只需关注 agent 定义和 termination 条件。
4. **跳转标准化**: Handoff 协议提供了规范的 agent 间切换, 支持上下文传递。

---

## 3. 核心概念

### 3.1 Agent 类型层次

```
ChatAgent (Protocol / ABC)
    |
    |-- run(): 主入口, 接收消息列表, 返回 TaskResult
    |-- on_messages(): 同步处理消息
    |-- on_messages_stream(): 流式处理消息
    |-- save_state() / load_state(): 状态持久化
    |
    BaseChatAgent (abstract)
    |
    |-- name: str           -- agent 名称
    |-- description: str    -- agent 描述
    |-- on_messages()       -- 具体实现
    |-- on_messages_stream()-- 流式实现
    |
    +---> AssistantAgent
    |     LLM 驱动的对话 agent, 支持 tools 和 handoffs
    |     - model_client: 模型客户端
    |     - tools: 可用工具列表
    |     - handoffs: handoff 目标 agent 列表
    |     - system_message: 系统提示词
    |
    +---> ToolAgent
    |     专门执行工具的 agent, 不调用 LLM
    |     - tools: 工具列表
    |     - 接收 ToolCallRequest, 返回 ToolCallResult
    |
    +---> CodeExecutorAgent
    |     在沙箱环境中执行代码
    |     - code_executor: 代码执行器 (local/docker/jupyter)
    |     - 接收代码块, 返回执行结果
    |
    +---> SocietyOfMindAgent
           内部维护一组子 agent, 聚合它们的回复
           - 用于 inner monologue 模式
```

### 3.2 Team 类型

```
Team (BaseGroupChat)
    |
    |-- participants: list[ChatAgent]    -- 参与 agent
    |-- termination_condition            -- 终止条件
    |-- run()                            -- 执行对话循环
    |-- run_stream()                     -- 流式执行
    |
    +---> RoundRobinGroupChat
    |     简单的轮转调度, agent 按顺序轮流发言
    |
    +---> SelectorGroupChat
    |     LLM 或函数驱动的 speaker 选择
    |     支持 selector_func / selector_llm
    |
    +---> MagnifyGroupChat
    |     分层团队, 有 meta-agent 管理子 agent
    |
    +---> Swarm
           Handoff 驱动的 agent 切换
           支持 context agents 扩展
```

### 3.3 架构总览

```
                    +---------------------------+
                    |       Termination          |
                    |    (max_turn / stop_msg    |
                    |     / external / text)     |
                    +---------------------------+
                               |
                    +---------------------------+
                    |         Team.run()         |
                    |  (对话调度主循环)           |
                    +---------------------------+
                    |                           |
         +------------------+         +------------------+
         |  Message Queue   |         |  Speaker/Turn    |
         |  (历史消息队列)   |         |  Selection Logic |
         +------------------+         +------------------+
                    |                           |
         +----------+-----------+   +-----------+----------+
         |                      |   |                      |
    +---------+           +---------+               +----------+
    | Agent A |           | Agent B |               | Agent C  |
    | ChatAgent|          | ChatAgent|               | ChatAgent|
    +---------+           +---------+               +----------+
         |                      |                      |
    +---------+           +---------+               +----------+
    | LLM /   |           | Tools / |               | Code     |
    | Handoff |           | Funcs   |               | Executor |
    +---------+           +---------+               +----------+
```

---

## 4. Team 机制

### 4.1 RoundRobinGroupChat -- 轮转调度

最简单的团队模式。Agent 按预设顺序轮流发言, 从头到尾, 循环往复。

```
RoundRobinGroupChat 执行流程:

步骤 1: Agent A 发言
  Team.run() --> Agent A.on_messages(history)
  Agent A 返回 Response --> 加入消息队列
         |
步骤 2: Agent B 发言
  Team.run() --> Agent B.on_messages(history)
  Agent B 返回 Response --> 加入消息队列
         |
步骤 3: Agent C 发言
  Team.run() --> Agent C.on_messages(history)
  Agent C 返回 Response --> 加入消息队列
         |
步骤 4: 检查 TerminationCondition
  --> 未终止: 回到步骤 1 (Agent A 再次发言)
  --> 已终止: 返回 TaskResult
```

特点:
- 确定性: 发言顺序完全可预测
- 公平性: 每个 agent 获得相同的发言机会
- 适用场景: 结构化对话, 如评审流程、多角度分析

使用示例:
```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.conditions import TextMentionTermination

agent_a = AssistantAgent("Alice", model_client=client, ...)
agent_b = AssistantAgent("Bob", model_client=client, ...)

team = RoundRobinGroupChat(
    [agent_a, agent_b],
    termination_condition=TextMentionTermination("DONE")
)

result = await team.run()  # 或 team.run_stream()
```

### 4.2 SelectorGroupChat -- LLM 选择发言者

使用 LLM 或自定义函数动态选择下一个发言者。SelectorGroupChat 在每轮结束后, 通过 selector 决定哪个 agent 最适合继续对话。

```
SelectorGroupChat 执行流程:

初始轮: 选择第一个发言者
  Selector 评估所有 agent 的描述和当前上下文
  --> 选择最佳匹配的 agent
         |
选定 Agent 发言
  Agent.on_messages(history)
  --> 加入消息队列
         |
下一轮: Selector 再次选择
  Selector 收到更新后的对话历史
  --> 分析哪个 agent 最适合处理当前状态
  --> 选择下一个 agent
         |
重复直到 TerminationCondition 触发
```

Selector 机制:
- **selector_llm**: 使用 LLM 来做选择 (默认方式)
  - 接收当前对话历史和所有 agent 的描述
  - 输出应发言的 agent name
  - selector_prompt 可自定义
- **selector_func**: 使用 Python 函数做选择
  - 接收 messages 列表和 agent 上下文
  - 返回选中的 agent 名称

使用示例:
```python
from autogen_agentchat.teams import SelectorGroupChat
from autogen_agentchat.agents import AssistantAgent

researcher = AssistantAgent("Researcher", model_client=client,
    description="负责调研和收集信息")
writer = AssistantAgent("Writer", model_client=client,
    description="负责撰写和润色文本")

team = SelectorGroupChat(
    [researcher, writer],
    model_client=client,       # selector 使用的 LLM
    termination_condition=TextMentionTermination("DONE")
)
# 每次选择时, SelectorGroupChat 会构建一个包含所有 agent 描述
# 和当前对话历史的 prompt, 让 LLM 决定谁适合发言
```

### 4.3 Swarm -- Handoff 驱动的 Agent 切换

Swarm 模式受 OpenAI Swarm 启发, 使用 handoff 消息进行 agent 间的显式切换。Agent 自主决定何时将控制权移交给其他 agent。

```
Swarm 团队模式执行流程:

                  +---------------------+
                  |  初始 Agent 启动      |
                  +---------------------+
                            |
                            v
                  +---------------------+
                  | Agent A 处理消息      |
                  +---------------------+
                            |
                    +-------+-------+
                    |               |
                    v               v
            +-----------+    +-----------+
            | 正常回复    |    | Handoff   |
            | + tools 等 |    | 到 Agent B|
            +-----------+    +-----------+
                    |               |
                    v               v
            继续当前 Agent    切换到 Agent B
                    |               |
                    +-------+-------+
                            |
                            v
                  +---------------------+
                  | 检查 Termination    |
                  | max_turns / stop    |
                  +---------------------+
                            |
                    +-------+-------+
                    |               |
                    v               v
                继续循环         返回结果
```

Swarm 的核心特点:
- Agent 自主决策: Agent 决定何时 handoff
- 上下文传递: Handoff 时携带上下文信息
- Context Agents: 额外 agent 可被引入提供上下文但不直接发言
- 强制 Handoff: 可配置避免 agent 陷入循环

使用示例:
```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import Swarm
from autogen_agentchat.base import Handoff

# 定义 handoff 目标
handoff_to_triage = Handoff(target="TriageAgent",
    message="将用户转接到分类 agent")

triage_agent = AssistantAgent(
    "TriageAgent",
    model_client=client,
    handoffs=[handoff_to_sales, handoff_to_support],
    description="负责初步分类用户请求"
)

sales_agent = AssistantAgent(
    "SalesAgent",
    model_client=client,
    handoffs=[handoff_to_triage],
    description="负责销售相关咨询"
)

team = Swarm(
    participants=[triage_agent, sales_agent, support_agent],
    termination_condition=TextMentionTermination("DONE")
)
```

Swarm 对比其他模式的独特之处:
- 不是外部调度器选择发言者, 而是 agent 自己决定
- Handoff 是 agent 回复的一部分 (作为 tool call)
- 支持多级跳转 (A -> B -> C -> A)

### 4.4 Team 类型对比

```
Team 类型对比:

RoundRobinGroupChat:
  发言顺序: [A] -> [B] -> [C] -> [A] -> [B] -> [C] -> ...
  选择方式: 固定轮转
  确定性: 完全确定
  适用: 结构化评审、全参与对话

SelectorGroupChat:
  发言顺序: [A] -> [C] -> [B] -> [A] -> [C] -> ...
  选择方式: LLM/函数动态选择
  确定性: 不确定 (依赖 LLM)
  适用: 动态协作、按需参与

Swarm:
  发言顺序: [A] --handoff--> [B] --handoff--> [C]
  选择方式: Agent 自主 handoff
  确定性: 由 agent 决策
  适用: 客服流转、任务分配、工作流
```

---

## 5. Handoff 协议

Handoff 是 AgentChat 中最核心的协议之一, 提供了 agent 间结构化的上下文转移机制。

### 5.1 HandoffMessage 类型

```python
from autogen_agentchat.messages import HandoffMessage
from autogen_agentchat.base import Handoff

# Handoff 定义
class Handoff:
    target: str          # 目标 agent 名称
    message: str         # handoff 时发送的消息
    context: dict | None # 可选, 要传递的上下文

# HandoffMessage 是 AgentChat 中的一种特殊消息类型
class HandoffMessage(ChatMessage):
    content: str          # handoff 消息内容
    source: str           # 发送 handoff 的 agent
    target: str           # 目标 agent 名称
```

### 5.2 Handoff 注册

Agent 通过 `handoffs` 参数注册可跳转的目标:

```python
handoff_support = Handoff(
    target="SupportAgent",
    message="用户需要技术支持, 转接到 SupportAgent"
)

handoff_billing = Handoff(
    target="BillingAgent",
    message="用户需要账单帮助, 转接到 BillingAgent"
)

triage_agent = AssistantAgent(
    "TriageAgent",
    model_client=client,
    handoffs=[handoff_support, handoff_billing]
)
```

### 5.3 Handoff 流程

```
Handoff 协议详细流程:

Agent A (当前发言)
    |
    |--- LLM 生成回复
    |--- 回复中包含 Handoff tool call
    |
    v
Team 检测到 HandoffMessage
    |
    |--- 解析 target = "AgentB"
    |--- 验证 Agent B 存在
    |
    +---> 上下文打包:
    |     - 当前对话历史
    |     - HandoffMessage 内容
    |     - 可选的 context 字典
    |
    v
消息分发:
    - HandoffMessage 加入消息队列
    - Team 将控制权切换到 Agent B
    |
    v
Agent B 接收:
    - 完整对话历史 (包括 Agent A 的所有消息)
    - 感知到自己是被 handoff 的目标
    - 基于完整上下文继续对话
```

### 5.4 上下文传递

AgentChat 的 handoff 保证:

1. **历史完整性**: 所有对话消息都传递给目标 agent
2. **来源可追溯**: `HandoffMessage.source` 标明转出来源
3. **目标明确**: `HandoffMessage.target` 标明接收者
4. **上下文扩展**: 可通过 `Handoff.context` 传递额外数据

```
上下文传递示意图:

对话历史:
  [User]  "我需要退款"
  [Triage] "我来帮您转接到账单部门"
  [Handoff] source=TriageAgent, target=BillingAgent

Agent B (BillingAgent) 接收到的上下文:
  messages = [
    UserMessage("我需要退款"),
    TextMessage("我来帮您转接到账单部门", source="TriageAgent"),
    HandoffMessage("用户需要账单帮助", source="TriageAgent", target="BillingAgent")
  ]
  --> BillingAgent 完整看到之前的对话, 并知道自己是 handoff 目标
```

### 5.5 注意事项

- Handoff 是单向的: agent A handoff 到 agent B 后, agent B 可以继续 handoff 到 agent C
- Agent 不能 handoff 给自己
- Handoff 目标必须存在于 participant list 中
- 在 Swarm 团队中, handoff 是主要的 agent 切换机制
- 在非 Swarm 团队中, handoff 消息仍然可以被处理, 但 team 的调度逻辑覆盖 handoff

---

## 6. ToolAgent 与 CodeExecutorAgent

### 6.1 ToolAgent

ToolAgent 是专门执行工具的 agent。它不调用 LLM, 而是接收 tool call 请求并执行相应的工具函数。

```
ToolAgent 架构:

            ToolCallRequest
                  |
                  v
         +------------------+
         |    ToolAgent     |
         |                  |
         | tool_registry =  |
         |  - tool_a        |
         |  - tool_b        |
         |  - tool_c        |
         +------------------+
                  |
       +----------+----------+
       |          |          |
       v          v          v
    tool_a     tool_b     tool_c
       |          |          |
       +----------+----------+
                  |
                  v
            ToolCallResult
```

使用示例:
```python
from autogen_agentchat.agents import ToolAgent
from autogen_ext.tools import tool

@tool
def search_web(query: str) -> str:
    """搜索网络"""
    return f"搜索结果: {query}"

@tool
def calculate(expression: str) -> str:
    """计算数学表达式"""
    return str(eval(expression))

tool_agent = ToolAgent(
    "ToolExecutor",
    tools=[search_web, calculate],
    description="执行工具调用"
)
```

### 6.2 CodeExecutorAgent

CodeExecutorAgent 在沙箱化环境中执行代码。它支持多种执行后端。

```
CodeExecutorAgent 架构:

    Code 文本消息
         |
         v
    +-------------------+
    | CodeExecutorAgent |
    |                   |
    | code_executor =   |
    |  +- LocalCLI      |
    |  +- Docker        |
    |  +- Jupyter       |
    +-------------------+
         |
    +----+-----+
    | 执行代码  |
    +----+-----+
         |
         v
    执行结果 (stdout + stderr)
```

使用示例:
```python
from autogen_agentchat.agents import CodeExecutorAgent
from autogen_ext.code_executors.local import LocalCommandLineCodeExecutor

code_agent = CodeExecutorAgent(
    "CodeRunner",
    code_executor=LocalCommandLineCodeExecutor(
        timeout=30,
        work_dir="./code_workspace"
    )
)

# CodeExecutorAgent 自动识别代码块并执行
# 支持 Python、Shell 等多种语言
```

支持的后端:
- **LocalCommandLineCodeExecutor**: 本地命令行执行
- **DockerCommandLineCodeExecutor**: Docker 容器执行
- **JupyterCodeExecutor**: Jupyter notebook 内核执行

### 6.3 组合使用

在实际系统中, Agent 链通常组合使用:

```
对话示例:

AssistantAgent (编写代码)
    |
    |--- 生成 Python 代码
    v
CodeExecutorAgent (执行代码)
    |
    |--- 返回执行结果
    v
AssistantAgent (分析结果)
    |
    |--- 总结并回复
    v
用户
```

---

## 7. 与低层 API 对比

### 7.1 对比表格

| 维度 | Core API (低层) | AgentChat API (高层) |
|------|-----------------|---------------------|
| Agent 定义 | `CoreAgent` / 自定义 | `AssistantAgent` / `ToolAgent` |
| 消息传递 | 手动 Agent.send/receive | Team 自动管理 |
| 对话编排 | 手动实现 loop | 内置 RoundRobin/Selector/Swarm |
| Agent 切换 | 手动消息路由 | 内置 Handoff 协议 |
| 终止检测 | 手动检查 | 声明式 TerminationCondition |
| 状态管理 | 自行实现 | save_state/load_state 接口 |
| 复杂度 | 灵活但需大量样板代码 | 开箱即用 |
| 自定义程度 | 完全控制 | 受 Team 抽象约束 |

### 7.2 何时使用 AgentChat vs Core API

**使用 AgentChat 当**:
- 需要快速搭建多 agent 协作系统
- 对话模式符合 RoundRobin/Selector/Swarm
- 需要 handoff 机制进行 agent 切换
- 希望减少样板代码

**使用 Core API 当**:
- 需要自定义消息路由逻辑
- 对话模式不符合 AgentChat 提供的 Team 类型
- 需要底层控制 agent 生命周期
- 构建自定义运行时或框架

### 7.3 代码对比

低层 API 手动实现 RoundRobin:
```python
# Core API: 手动管理
agents = [agent_a, agent_b, agent_c]
messages = []
turn = 0
while not done:
    current = agents[turn % len(agents)]
    response = await current.on_messages(messages)
    messages.append(response)
    turn += 1
    if check_termination(messages):
        done = True
```

AgentChat 声明式:
```python
# AgentChat: 声明式
team = RoundRobinGroupChat(
    [agent_a, agent_b, agent_c],
    termination_condition=MaxTurnTermination(10)
)
result = await team.run()
```

---

## 8. 源码分析关键点

### 8.1 ChatAgent 协议接口

```python
# 伪代码: ChatAgent 协议 / 抽象基类
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import AsyncGenerator, Sequence

class Response:
    """Agent 单次调用的回复"""
    chat_message: ChatMessage
    inner_messages: list[ChatMessage] | None = None

class TaskResult:
    """Team.run() 的最终结果"""
    messages: list[ChatMessage]
    stop_reason: str | None

class ChatAgent(ABC):
    """AgentChat 中所有 agent 的协议接口"""

    @property
    @abstractmethod
    def name(self) -> str: ...

    @property
    @abstractmethod
    def description(self) -> str: ...

    @abstractmethod
    async def on_messages(
        self, messages: Sequence[ChatMessage],
        cancellation_token: CancellationToken
    ) -> Response:
        """同步处理消息, 返回单条回复"""
        ...

    @abstractmethod
    def on_messages_stream(
        self, messages: Sequence[ChatMessage],
        cancellation_token: CancellationToken
    ) -> AsyncGenerator[AgentEvent | ChatMessage, None]:
        """流式处理消息, yield 中间事件和最终消息"""
        ...

    # on_messages_stream 的最后一个 yield 必须等价于 on_messages 的返回值

    @abstractmethod
    async def run(
        self, messages: Sequence[ChatMessage] | None = None,
        cancellation_token: CancellationToken | None = None
    ) -> TaskResult:
        """运行 agent, 处理消息直到主动停止"""
        ...

    @abstractmethod
    async def save_state(self) -> dict: ...

    @abstractmethod
    async def load_state(self, state: dict) -> None: ...
```

### 8.2 BaseChatAgent 抽象实现

```python
# 伪代码: BaseChatAgent -- ChatAgent 的抽象基类实现
class BaseChatAgent(ChatAgent):
    """提供 ChatAgent 的通用实现骨架"""

    def __init__(
        self,
        name: str,
        description: str = ""
    ):
        self._name = name
        self._description = description

    @property
    def name(self) -> str:
        return self._name

    @property
    def description(self) -> str:
        return self._description

    async def run(
        self,
        messages: Sequence[ChatMessage] | None = None,
        cancellation_token: CancellationToken | None = None
    ) -> TaskResult:
        messages = list(messages) if messages else []
        response = await self.on_messages(messages, cancellation_token)
        messages.append(response.chat_message)
        return TaskResult(
            messages=messages,
            stop_reason="agent stopped"
        )

    async def save_state(self) -> dict:
        return {"name": self._name}

    async def load_state(self, state: dict) -> None:
        self._name = state.get("name", self._name)
```

### 8.3 RoundRobinGroupChat 实现

```python
# 伪代码: RoundRobinGroupChat 核心逻辑
class RoundRobinGroupChat(BaseGroupChat):
    """轮转调度团队"""

    def __init__(
        self,
        participants: list[ChatAgent],
        termination_condition: TerminationCondition | None = None,
        max_turns: int | None = None
    ):
        self._participants = participants
        self._termination_condition = termination_condition
        self._current_turn = 0

    async def run(self) -> TaskResult:
        messages = []
        while not await self._should_terminate(messages):
            # 选择当前发言的 agent
            agent_idx = self._current_turn % len(self._participants)
            current_agent = self._participants[agent_idx]

            # agent 处理对话历史
            response = await current_agent.on_messages(messages)

            # 将回复加入消息队列
            messages.append(response.chat_message)
            if response.inner_messages:
                messages.extend(response.inner_messages)

            # 轮转到下一个 agent
            self._current_turn += 1

        return TaskResult(messages=messages, ...)

    # 消息始终保留: 所有 agent 看到完整历史
    # 无 speaker selection, 纯粹轮转
```

### 8.4 SelectorGroupChat 实现

```python
# 伪代码: SelectorGroupChat 的 LLM speaker selection
class SelectorGroupChat(BaseGroupChat):
    """基于 LLM 或函数的 speaker 选择"""

    def __init__(
        self,
        participants: list[ChatAgent],
        model_client: ModelClient | None = None,
        selector_func: Callable | None = None,
        selector_prompt: str | None = None,
        termination_condition: TerminationCondition | None = None
    ):
        self._participants = participants
        self._model_client = model_client
        self._selector_func = selector_func
        self._selector_prompt = selector_prompt or DEFAULT_SELECTOR_PROMPT

    async def _select_speaker(
        self, messages: list[ChatMessage]
    ) -> ChatAgent:
        if self._selector_func:
            # 使用自定义函数选择
            agent_name = await self._selector_func(messages, ...)
            return self._find_agent(agent_name)

        # 使用 LLM 选择
        participant_descriptions = "\n".join(
            f"- {a.name}: {a.description}" for a in self._participants
        )

        prompt = self._selector_prompt.format(
            roles=participant_descriptions,
            history=format_messages(messages)
        )

        # LLM 分析对话并选择下一个 speaker
        response = await self._model_client.complete(prompt)
        selected_name = self._parse_selector_response(response)

        return self._find_agent(selected_name)

    async def run(self) -> TaskResult:
        # 初始选择第一个 speaker
        messages = []
        current_agent = await self._select_speaker([])

        while not await self._should_terminate(messages):
            response = await current_agent.on_messages(messages)
            messages.append(response.chat_message)

            # 为下一轮选择新的 speaker
            if not await self._should_terminate(messages):
                current_agent = await self._select_speaker(messages)

        return TaskResult(messages=messages, ...)
```

### 8.5 Handoff 协议实现

```python
# 伪代码: Handoff 协议的核心机制

# 消息类型
@dataclass
class HandoffMessage(ChatMessage):
    content: str       # 转接消息
    source: str        # 来源 agent
    target: str        # 目标 agent

# Handoff 注册
@dataclass
class Handoff:
    target: str
    message: str
    context: dict | None = None

    def to_tool(self) -> Tool:
        """将 Handoff 转换为 LLM tool definition"""
        return Tool(
            name=f"handoff_to_{self.target}",
            description=self.message,
            parameters={
                "type": "object",
                "properties": {
                    "reason": {"type": "string"}
                },
                "required": ["reason"]
            }
        )

# AssistantAgent 中 handoff 的处理
class AssistantAgent(BaseChatAgent):
    def __init__(self, ..., handoffs: list[Handoff] | None = None):
        self._handoffs = handoffs or []
        # 将 handoffs 注册为 LLM tools
        self._tools.extend([
            h.to_tool() for h in self._handoffs
        ])

    async def on_messages(self, messages, ...) -> Response:
        llm_response = await self._model_client.complete(
            messages=messages,
            tools=self._tools  # 包括 handoff tools
        )

        if llm_response.is_handoff:
            # LLM 调用了一个 handoff tool
            target_name = llm_response.tool_call.arguments["target"]
            return Response(
                chat_message=HandoffMessage(
                    content=self._get_handoff(target_name).message,
                    source=self.name,
                    target=target_name
                )
            )
        else:
            # 正常回复
            return Response(chat_message=llm_response.text_message)

# Team 中 handoff 的处理
class BaseGroupChat:
    async def _process_handoff(
        self, message: HandoffMessage
    ) -> ChatAgent:
        # 验证目标存在
        target_agent = self._get_agent_by_name(message.target)
        if target_agent is None:
            raise ValueError(f"handoff target {message.target} not found")
        return target_agent
```

### 8.6 Swarm 执行循环

```python
# 伪代码: Swarm 团队的执行循环
class Swarm(BaseGroupChat):
    """Handoff 驱动的多 agent 团队"""

    def __init__(
        self,
        participants: list[ChatAgent],
        termination_condition: TerminationCondition | None = None,
        max_turns: int | None = 100,
        handoff_message_filter: Callable | None = None
    ):
        self._participants = participants
        self._max_turns = max_turns
        self._handoff_message_filter = handoff_message_filter

    async def run(self) -> TaskResult:
        messages = []
        # 初始 agent: 第一个 participant
        current_agent = self._participants[0]
        turn_count = 0

        while not await self._should_terminate(messages):
            if self._max_turns and turn_count >= self._max_turns:
                break

            # 当前 agent 处理完整历史
            response = await current_agent.on_messages(messages)

            # 添加中间消息 (tool calls, code execution)
            if response.inner_messages:
                for inner_msg in response.inner_messages:
                    if inner_msg not in messages:
                        messages.append(inner_msg)

            # 添加 agent 的主回复
            messages.append(response.chat_message)

            # 检查是否是 handoff 消息
            if isinstance(response.chat_message, HandoffMessage):
                # 切换到目标 agent
                target_name = response.chat_message.target
                new_agent = self._get_agent(target_name)

                if new_agent is None:
                    raise RuntimeError(
                        f"Handoff target {target_name} not in participants"
                    )

                current_agent = new_agent

                # 可选: 应用 handoff message filter
                if self._handoff_message_filter:
                    messages = self._handoff_message_filter(messages)

            # 如果没有 handoff, 当前 agent 继续
            # (与 SelectorGroupChat 不同, Swarm 不自动切换发言者)

            turn_count += 1

        return TaskResult(messages=messages, ...)

    def _get_agent(self, name: str) -> ChatAgent | None:
        for agent in self._participants:
            if agent.name == name:
                return agent
        return None
```

### 8.7 消息类型系统

```python
# 伪代码: AgentChat 消息类型层次
@dataclass
class BaseMessage:
    """所有消息的基类"""
    source: str

@dataclass
class ChatMessage(BaseMessage):
    """agent 间通信的主要消息"""
    content: str

@dataclass
class TextMessage(ChatMessage):
    """普通文本消息"""
    pass

@dataclass
class HandoffMessage(ChatMessage):
    """handoff 协议消息"""
    target: str

@dataclass
class ToolCallMessage(ChatMessage):
    """工具调用请求"""
    tool_calls: list[ToolCall]

@dataclass
class ToolCallResultMessage(ChatMessage):
    """工具调用结果"""
    tool_call_results: list[ToolCallResult]

@dataclass
class StopMessage(ChatMessage):
    """终止信号"""
    pass

@dataclass
class AgentEvent(BaseMessage):
    """agent 内部事件 (不直接进入对话历史)"""
    pass
```

---

## 9. 局限性与改进

### 9.1 API 成熟度

- **持续演进**: AgentChat API 仍在快速迭代 (v0.4.x -> v0.6.x), 接口和语义在不同版本间可能存在不兼容
- **文档缺口**: 部分高级功能 (MagnifyGroupChat, SocietyOfMindAgent) 文档较少, 需要阅读源码理解
- **迁移成本**: 从 v0.2 ConversableAgent 迁移到 v0.4+ AgentChat 需要较大改动

### 9.2 Handoff 性能

- **上下文重复**: Handoff 时传递完整对话历史可能导致 token 浪费
- **无细粒度控制**: 默认全量上下文传递, 无法只选择性地传递部分上下文
- **循环风险**: Agent 可能在 agent 间无限 handoff (可通过 max_turns 缓解)
- **上下文丢失**: 某些情况下 tool call 的中间结果在 handoff 中丢失 (GitHub issue #5045)

### 9.3 Team 扩展性

- **参与数量限制**: 大量 agent 在一个 team 中可能导致:
  - Selector 的 prompt 变得非常长 (所有 agent 描述)
  - RoundRobin 的轮次效率低下
  - LLM 选择混淆
- **嵌套团队**: 缺乏成熟的嵌套团队支持
- **动态参与**: 无法在运行时动态添加/移除 agent

### 9.4 已知问题

| 问题 | 影响 | 状态 |
|------|------|------|
| Tool call 输出在 handoff 中丢失 | Swarm 模式下 tool 结果不传递给下个 agent | 已知 issue (#5045) |
| Selector prompt 不可自定义 | 默认 prompt 可能不适合某些场景 | 部分版本已改进 |
| Agent 在 Swarm 中卡住 | agent 无法触发 handoff 导致无限循环 | 强制 handoff 特性已添加 (#5611) |
| MagnifyGroupChat 文档缺乏 | 高级团队模式难以使用 | 待完善 |
| on_messages/on_messages_stream 关系模糊 | BaseChatAgent 的抽象接口让人困惑 | 讨论中 (#5125) |

### 9.5 改进方向

1. **选择性上下文传递**: 允许 agent 指定 handoff 时传递哪些历史消息
2. **嵌套团队**: 支持 team 作为 participant, 实现分层编排
3. **动态 Agent 注册**: 运行时动态添加/移除 agent
4. **分布式 Swarm**: 支持跨进程/跨机器的 agent handoff
5. **改进的终止条件**: 更丰富的 TerminationCondition 组合
6. **Selector 优化**: 缓存 selector 结果, 减少不必要的 LLM 调用

---

## 参考

- AutoGen AgentChat API: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/
- AutoGen GitHub: https://github.com/microsoft/autogen
- OpenAI Swarm (inspired Swarm team): https://github.com/openai/swarm
- AgentChat tutorial agents: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/agents.html
- AgentChat teams: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/teams.html
- Source code: autogen_agentchat.agents -- https://github.com/microsoft/autogen/tree/main/python/packages/autogen-agentchat/src/autogen_agentchat/agents
- Source code: autogen_agentchat.teams -- https://github.com/microsoft/autogen/tree/main/python/packages/autogen-agentchat/src/autogen_agentchat/teams
