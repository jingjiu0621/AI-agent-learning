# 8.1.5 structured-dialogue — 结构化对话：角色标记 / 消息类型 / 路由

## 简单介绍

结构化对话是多 Agent 通信的"语义层"——它定义了消息的"言外之意"：这条消息是提问还是回答？是命令还是建议？是哪个角色在什么上下文中发出的？结构化对话让 Agent 不仅仅知道"对方说了什么"，还理解"对方在做什么"。

## 基本原理

### 结构化对话的三层模型

```
┌─────────────────────────────────────────┐
│  3. 路由层（Routing Layer）              │
│     消息该发给谁？谁需要知道？            │
├─────────────────────────────────────────┤
│  2. 对话层（Dialogue Layer）             │
│     消息类型：提问/回答/建议/命令/确认    │
│     角色标记：发言者角色与意图            │
├─────────────────────────────────────────┤
│  1. 传输层（Transport Layer）            │
│     消息格式、序列化、投递               │
└─────────────────────────────────────────┘
```

### 核心要素

**角色标记（Role Markers）**：标识消息来源的角色，帮助接收方理解消息的"立场"和"权威级别"。

| 角色 | 语义 | 示例 |
|------|------|------|
| orchestrator | 协调者/管理者 | 分配任务、查询状态 |
| researcher | 研究者 | 提交研究发现 |
| critic | 评审者 | 提供反馈、指出问题 |
| executor | 执行者 | 报告执行结果 |
| observer | 观察者 | 通报状态变化 |
| user_proxy | 用户代理 | 转达用户需求 |

**消息类型（Message Types）**：标识消息的"言外行为"（illocutionary force）。

| 类型 | 语义 | 期望响应 |
|------|------|----------|
| proposal | 提议/建议 | 接受/拒绝/修改建议 |
| request | 请求信息/行动 | 提供信息/执行行动 |
| command | 指令（必须执行） | 执行并报告 |
| response | 对请求的回应 | 无（对话结束） |
| confirmation | 确认收到/理解 | 无或确认回执 |
| objection | 反对/质疑 | 解释/修改/撤销 |
| query | 询问信息 | 信息回复 |
| inform | 主动告知信息 | 无必要 |
| error | 错误报告 | 处理/转交 |

**路由规则（Routing Rules）**：决定消息的传递路径。

| 路由策略 | 描述 | 适用场景 |
|----------|------|----------|
| Direct | 指定接收者 | 已知目标 |
| Role-based | 按角色路由 | 不知道具体 Agent，但知道想要谁做 |
| Topic-based | 按话题路由 | 发布-订阅模式 |
| Capability-based | 按能力路由 | 任务分配 |
| Hybrid | 组合多种策略 | 复杂场景 |

## 背景与演进

- **前期（2022-2023）**：Agent 间传递原始文本消息，没有角色/类型标记。Agent 需要在消息体中自己解析意图，经常误解——"帮我查一下资料"被理解成"帮我查一下"（查什么？）。非结构化消息导致频繁的澄清轮次。
- **对话协议引入（2023）**：AutoGen 提出了 AgentChat 协议，引入了消息类型（MessageType）和对话 ID。自此 Agent 明确知道"这是对之前问题的回复"还是"这是新话题"。
- **角色标记标准化（2024）**：CrewAI 推广了角色分配——每个 Agent 有明确的 role/goal/backstory。消息自动携带角色信息，接收方基于角色理解消息的权重和可靠性。
- **当前趋势**：结构化对话与工作流引擎结合（如 Temporal + Agent），对话状态由工作流管理，支持中断恢复、Saga 模式等企业级模式。

## 核心矛盾

**"表达的丰富性 vs 解析的确定性"——结构化对话的核心权衡**：

| 丰富性 | 确定性 |
|--------|--------|
| 自由文本，表达力强 | 类型有限，但解析可靠 |
| Agent 自主理解意图 | 意图显式标记 |
| 适应意外情况 | 可能遗漏特殊情况 |
| 需要 LLM 理解上下文 | 程序化路由，无需 LLM |
| 歧义性高，需多轮澄清 | 歧义性低，减少轮次 |

## 主流优化方向

1. **对话状态机**：定义对话的阶段（init → discuss → decide → confirm），每个阶段允许的消息类型不同
2. **角色层级**：角色有层级关系（orchestrator > researcher > executor），消息按角色权限路由
3. **对话上下文压缩**：长对话中只保留关键的结构化信息，丢弃冗余内容
4. **多轮对话匹配**：使用 conversation_id + turn_number 精确追踪多轮交互
5. **对话超时与恢复**：长时间无响应的对话自动超时，支持基于 checkpoint 恢复
6. **跨对话引用**：一条消息可以引用之前对话中的某条消息（类似 git cherry-pick）

## 能力边界与结果边界

- **对话长度**：建议单次对话不超过 20 轮，超过则归档开启新对话
- **角色数量**：建议 5-8 个不同角色，超过则角色认知模糊化
- **消息类型**：建议 8-12 种类型，过多则分类决策困难
- **路由延迟**：基于角色的路由增加 1-5ms，基于能力的路由增加 10-50ms
- **不适用场景**：高度创意性/开放性的讨论（结构化框架束缚创造力）、紧急情况（需要快速自由沟通）

## 与其他内容的区别

- **vs 消息格式（8.1.1）**：消息格式定义"语法"，结构化对话定义"语用"——消息在对话中的意义和用法。
- **vs 辩论协议（8.4）**：辩论是结构化对话的特例——有明确的论点、证据、反驳结构，以及评审 Agent 裁决。
- **vs 任务委派（8.3）**：委派关注"谁做什么"，结构化对话关注"怎么说"。

## 核心优势

- **减少歧义**：消息类型和角色标记让 LLM 准确理解意图
- **可路由**：结构化消息支持程序化路由，无需 LLM 决策
- **可审计**：对话结构清晰，适合日志分析和回放
- **状态可管理**：对话阶段和状态可以在工作流引擎中管理
- **容错性好**：结构化消息容易验证和修复

## 工程优化

1. **对话 ID 生成**：使用 ULID（时间有序的唯一 ID），便于按时间范围查询
2. **消息序列号**：同一对话中的消息按序编号，检测丢失和乱序
3. **角色权限缓存**：角色→权限映射缓存在本地，减少查找延迟
4. **对话归档**：完成的对话移出活跃存储，转入冷存储（S3/Blob）
5. **结构化日志**：记录每条消息的 type/role/conversation_id，便于分析对话模式

## 适用场景

| 场景 | 结构化程度 | 推荐角色数 | 推荐消息类型 |
|------|-----------|-----------|-------------|
| 多 Agent 研究协作 | 中 | 3-4（协调/研究/写作/评审） | 6-8 |
| 客服多 Agent 系统 | 高 | 4-5（接待/查询/升级/解决） | 8-10 |
| 代码审查 Agent | 高 | 3（提交/审查/批准） | 6 |
| 开放式讨论系统 | 低 | 2-3（主持人/参与者） | 4-5 |

## 示例代码

### 结构化对话消息

```python
from dataclasses import dataclass, field
from enum import Enum
import uuid
import time
from typing import Optional

class Role(Enum):
    ORCHESTRATOR = "orchestrator"
    RESEARCHER = "researcher"
    CRITIC = "critic"
    EXECUTOR = "executor"
    WRITER = "writer"

class MessageType(Enum):
    PROPOSAL = "proposal"
    REQUEST = "request"
    COMMAND = "command"
    RESPONSE = "response"
    CONFIRMATION = "confirmation"
    OBJECTION = "objection"
    QUERY = "query"
    INFORM = "inform"
    ERROR = "error"

@dataclass
class StructuredMessage:
    """结构化对话消息"""
    id: str = field(default_factory=lambda: f"msg_{uuid.uuid4().hex[:12]}")
    conversation_id: str = ""
    turn_number: int = 0
    sender_role: Role = Role.ORCHESTRATOR
    receiver_role: Optional[Role] = None
    msg_type: MessageType = MessageType.INFORM
    content: str = ""
    payload: dict = field(default_factory=dict)
    reply_to: Optional[str] = None  # 回复哪条消息
    timestamp: float = field(default_factory=time.time)

    def is_command(self) -> bool:
        return self.msg_type == MessageType.COMMAND

    def requires_response(self) -> bool:
        return self.msg_type in (
            MessageType.REQUEST,
            MessageType.QUERY,
            MessageType.PROPOSAL
        )


class StructuredDialogue:
    """结构化对话管理器"""

    def __init__(self, conversation_id: str = None):
        self.conversation_id = conversation_id or f"conv_{uuid.uuid4().hex[:8]}"
        self.turn = 0
        self.history: list[StructuredMessage] = []
        self.pending_requests: dict[str, StructuredMessage] = {}

    def send(self, msg: StructuredMessage) -> str:
        """发送消息"""
        msg.conversation_id = self.conversation_id
        msg.turn_number = self.turn
        self.turn += 1
        self.history.append(msg)

        if msg.requires_response():
            self.pending_requests[msg.id] = msg

        print(f"[{msg.sender_role.value}->{msg.receiver_role.value}] "
              f"[{msg.msg_type.value}] {msg.content[:50]}")
        return msg.id

    def reply(self, reply_to_id: str, sender: Role, receiver: Role,
              msg_type: MessageType, content: str) -> str:
        """回复之前的消息"""
        original = self.pending_requests.pop(reply_to_id, None)
        msg = StructuredMessage(
            sender_role=sender,
            receiver_role=receiver,
            msg_type=msg_type,
            content=content,
            reply_to=reply_to_id,
            conversation_id=self.conversation_id
        )
        return self.send(msg)

    def get_history(self, since_turn: int = 0) -> list[StructuredMessage]:
        """获取对话历史"""
        return [m for m in self.history if m.turn_number >= since_turn]


# 使用示例：多 Agent 研究协作
dialogue = StructuredDialogue("research_001")

# Orchestrator 指派研究任务
orchestrator_msg = StructuredMessage(
    sender_role=Role.ORCHESTRATOR,
    receiver_role=Role.RESEARCHER,
    msg_type=MessageType.REQUEST,
    content="请调查 2024 年多 Agent 系统的最新进展",
    payload={"depth": "comprehensive", "max_sources": 10}
)
req_id = dialogue.send(orchestrator_msg)

# Researcher 回复
dialogue.reply(
    req_id,
    sender=Role.RESEARCHER,
    receiver=Role.ORCHESTRATOR,
    msg_type=MessageType.RESPONSE,
    content="发现 3 篇关键论文和 2 个开源框架"
)

# Critic 提出异议（即使不是直接发给他的，他也可以关注）
critic_msg = StructuredMessage(
    sender_role=Role.CRITIC,
    receiver_role=Role.RESEARCHER,
    msg_type=MessageType.OBJECTION,
    content="我质疑其中一篇论文的方法论有效性",
    reply_to=req_id
)
dialogue.send(critic_msg)
```

### 基于角色的消息路由

```python
class RoleBasedRouter:
    """基于角色的消息路由器"""

    def __init__(self):
        # 角色 → Agent ID 映射
        self.role_registry: dict[Role, list[str]] = {
            Role.ORCHESTRATOR: [],
            Role.RESEARCHER: [],
            Role.CRITIC: [],
            Role.EXECUTOR: [],
        }

    def register(self, role: Role, agent_id: str):
        """注册 Agent 到角色"""
        self.role_registry[role].append(agent_id)
        print(f"注册 {agent_id} -> {role.value}")

    def route(self, msg: StructuredMessage) -> list[str]:
        """根据消息路由到目标 Agent"""
        if msg.receiver_role:
            # 按角色路由
            targets = self.role_registry.get(msg.receiver_role, [])
            if not targets:
                print(f"[Warn] 角色 {msg.receiver_role} 无可用 Agent")
            return targets
        # 没有指定角色，广播
        all_agents = []
        for agents in self.role_registry.values():
            all_agents.extend(agents)
        return all_agents
```
