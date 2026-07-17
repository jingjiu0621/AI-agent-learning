# 8.6.5 LangGraph 多 Agent 状态管理与路由

## 1. 简单介绍

LangGraph 是 LangChain 团队推出的低层级编排框架，用于构建有状态、多步骤的 Agent 应用。其核心思想是将多 Agent 协作建模为**有向图**：图中的每个节点（Node）是一个 Agent 或工具函数，边（Edge）定义了执行顺序和路由逻辑，而一个共享的**状态对象（State）**在节点之间传递和流转。

与简单的链式调用不同，LangGraph 提供了：

- **图状态机**：基于 StateGraph 定义节点和边，支持条件分支、循环和并行。
- **内置持久化**：通过 Checkpointer 在每个步骤保存状态快照，支持故障恢复、暂停/恢复和流式重放。
- **人机协同**：通过 interrupt 机制在任意节点暂停执行，等待人类输入后恢复。
- **并行执行**：通过 Send API 实现动态 fan-out / fan-in 模式。

LangGraph 不限定 Agent 的组织方式——你可以用它实现 Supervisor（监督者）模式、Round-Robin（轮询）模式、Network（网络）模式，或是任意自定义拓扑。

---

## 2. 核心概念

### 2.1 StateGraph

`StateGraph` 是 LangGraph 的核心类，代表一个有向图结构。它接收一个状态 schema 定义图中所有节点共享的状态类型。

```
                 +-------------------+
                 |     StateGraph    |
                 |  (状态图定义)      |
                 +--------+----------+
                          |
            +-------------+-------------+
            |             |             |
       add_node     add_edge    add_conditional_edges
            |             |             |
     +------+------+    +--+--+    +----+----+
     |  Agent/工具  |    | 固定  |    | 条件路由  |
     |    节点      |    | 边   |    |   函数    |
     +-------------+    +-----+    +---------+
```

### 2.2 Node（节点）

Node 是图中的一个执行单元。它可以是一个 Agent 函数、一个工具调用、一个数据转换函数，甚至是另一个被编译的子图（Subgraph）。

```python
def research_agent(state: AgentState) -> dict:
    """一个 Agent 节点：接收状态，返回状态更新。"""
    query = state["messages"][-1].content
    result = llm.invoke(f"Research: {query}")
    return {"messages": [result]}
```

### 2.3 Edge（边）

边定义了节点之间的执行流。LangGraph 支持两类边：

- **固定边（Fixed Edge）**：`add_edge(node_a, node_b)` — node_a 执行完后无条件进入 node_b。
- **条件边（Conditional Edge）**：`add_conditional_edges(node_a, router_func, routing_map)` — 根据 router_func 的返回值决定下一个节点。

### 2.4 State（状态）

状态是一个在图的整个生命周期内传递的共享对象。它通过 `TypedDict` 或 Pydantic `BaseModel` 定义 schema，并且每个字段可以附加一个 **reducer** 函数来定义该字段在多路写入时如何合并。

```python
from typing import Annotated, Sequence, TypedDict
from langgraph.graph.message import add_messages
from langchain_core.messages import BaseMessage

class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]  # 追加模式
    agent_output: str                                          # 覆盖模式
    documents: Annotated[list, operator.add]                   # 列表拼接
    steps: Annotated[list, operator.add]                       # 步骤跟踪
```

### 2.5 Checkpointer（检查点）

Checkpointer 是持久化层，在每个节点执行完毕后保存状态的完整快照。这使得：

- 执行可以在任意点暂停并恢复
- 可以回放历史步骤
- 支持多线程/多会话隔离

```
+-----+     +-----+     +-----+     +-----+
| N1  | --> | N2  | --> | N3  | --> | N4  |
+--+--+     +--+--+     +--+--+     +--+--+
   |           |           |           |
   v           v           v           v
[CP1]       [CP2]       [CP3]       [CP4]
           Checkpoint Timeline
```

### 2.6 完整 ASCII 架构图

```
                          +===============================+
                          |      LangGraph 应用运行时       |
                          +===============================+
                                     |
                   +-----------------+------------------+
                   |                                    |
          +--------v--------+              +-------------v----------+
          |   StateGraph     |              |    Checkpointer       |
          |  (图定义)        |              |  (持久化引擎)          |
          +--------+--------+              |  MemorySaver /         |
                   |                       |  SqliteSaver /         |
          +--------v--------+              |  PostgresSaver         |
          |   CompiledGraph  |              +------------------------+
          |  (编译后的图)     |
          +--------+--------+
                   |
     +-------------+------------------+
     |             |                  |
+----v----+  +----v----+        +----v----+
| Agent A  |  | Agent B  |  ...  | Tool Z  |
| (LLM)   |  | (LLM)   |        | (函数)  |
+----+----+  +----+----+        +----+----+
     |             |                  |
     +------+------+                  |
            |                         |
     +------v------+                  |
     | Router 函数  |------------------+
     | (条件判断)   |
     +-------------+
            |
  +---------+---------+
  |         |         |
  v         v         v
END      Agent C    Tool Y

      +----- State 对象贯穿所有节点 -----+
      | messages: Annotated[Sequence,   |
      |          add_messages]          |
      | documents: Annotated[list,      |
      |            operator.add]        |
      | agent_output: str               |
      +----------------------------------+
```

---

## 3. 多 Agent 状态管理

### 3.1 共享状态 Schema

LangGraph 使用 `TypedDict` 或 Pydantic `BaseModel` 定义状态结构。所有节点读写同一个状态对象。

```python
from typing import Annotated, List, Optional
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages
from langchain_core.messages import BaseMessage
import operator

# TypedDict 方式
class ResearchState(TypedDict):
    messages: Annotated[List[BaseMessage], add_messages]  # reducer: append
    research_notes: Annotated[List[str], operator.add]    # reducer: list concat
    final_report: str                                     # reducer: replace (default)
    current_agent: str                                    # reducer: replace
    search_results: Annotated[List[dict], operator.add]  # reducer: list concat
    retry_count: int                                      # reducer: replace
```

### 3.2 Reducer 函数

Reducer 是 LangGraph 状态管理的核心机制。当多个节点并行写入同一个状态字段时，reducer 定义如何合并这些写入。

| Reducer | 行为 | 适用场景 |
|---|---|---|
| 默认（无注解） | 后写覆盖前写 | 单一路径的字段 |
| `add_messages` | 追加到消息列表，去重 | Agent 对话消息 |
| `operator.add` | 列表拼接 | 收集结果、步骤记录 |
| 自定义函数 | 任意合并逻辑 | 计数器、投票聚合 |

**自定义 Reducer 示例：**

```python
from typing import Annotated

def vote_aggregator(current: dict, updates: dict) -> dict:
    """聚合多个 Agent 的投票结果。"""
    merged = dict(current)
    for agent, vote in updates.items():
        if agent not in merged:
            merged[agent] = []
        merged[agent].append(vote)
    return merged

class CouncilState(TypedDict):
    votes: Annotated[dict, vote_aggregator]    # 自定义聚合
    consensus: str                              # 覆盖模式
```

### 3.3 状态分支与并行

当多个节点并行执行时，每个节点写入自己那份状态，LangGraph 的运行时负责在 fan-in 点合并。

```
时间线 -->
        START
          |
    +-----+------+
    |            |
 Agent A     Agent B       <- 两个节点并行执行
    |            |
    |   Agent C  |          <- Agent C 也是并行的
    |            |
    +-----+------+
          |
      Aggregator           <- fan-in: 所有写入在此合并
          |
```
在合并点，同一个字段会收到来自多个节点的更新。Reducer 决定了这些更新如何合并：

- `add_messages`：按 `id` 去重追加（如果消息有相同 id 则覆盖）。
- `operator.add`：列表直接拼接。
- 自定义 reducer：由你的函数定义。

### 3.4 状态同步机制

LangGraph 的状态同步通过**通道（Channel）**系统实现：

```
每个状态字段 = 一个 Channel
     |
     +-- 节点 A 写入 --> Channel.update(values_A)
     +-- 节点 B 写入 --> Channel.update(values_B)
     |
     +-- Reducer(values_A + values_B) --> 新状态
```

- 每个字段是独立的 Channel。
- Channel 记录版本号（channel_versions），检测冲突。
- 当所有前置节点完成后，Channel 调用 reducer 合并写入。
- 合并结果写入 Checkpointer。

---

## 4. 路由机制

路由是 LangGraph 多 Agent 系统的核心——它决定消息/状态从一个节点执行完后流向哪个节点。

### 4.1 固定边（Fixed Edge）

最简单的路由：当前节点执行完，无条件进入下一个节点。

```python
workflow.add_edge("agent_a", "agent_b")       # 串行
workflow.add_edge(START, "entry_agent")       # 起始
workflow.add_edge("final_agent", END)          # 结束
workflow.add_edge(["a", "b"], "aggregator")    # fan-in：等待 a 和 b 都完成
```

### 4.2 条件边（Conditional Edge）

条件边通过一个**路由函数**决定流向。这是多 Agent 路由最常用的方式。

```python
def supervisor_router(state: AgentState) -> str:
    """Supervisor Agent 决定下一个执行的 Agent。"""
    messages = state["messages"]
    last_msg = messages[-1].content

    # 通过 LLM 判断下一步
    decision = llm.invoke(
        f"Based on: {last_msg}\nChoose: research | code | chart | done",
        response_format={"type": "text"}
    )
    return decision.content.strip()

workflow.add_conditional_edges(
    "supervisor",              # 起始节点
    supervisor_router,         # 路由函数
    {                          # 路由映射表
        "research": "researcher",
        "code": "coder",
        "chart": "chart_generator",
        "done": END,
    }
)
```

### 4.3 Agent 路由（通过 LLM 判断）

Agent 路由是条件边的特例，路由函数本身是一个 LLM 调用。这是 Supervisor 模式的典型实现：

```python
# Supervisor Agent 节点：分析状态并决定下一节点
def supervisor_node(state: AgentState) -> dict:
    messages = state["messages"]
    # LLM 分析当前情况
    response = llm.invoke([
        SystemMessage("你是 Supervisor。分析当前进展并选择下一工作者。选项: researcher, coder, chart_gen, finish"),
        HumanMessage(f"当前消息: {messages[-1].content}"),
        HumanMessage(f"已完成步骤: {state['steps']}")
    ])
    # 将决策写入状态
    return {"next_agent": response.content.strip()}

# 路由函数读取状态中的决策
def routing_function(state: AgentState) -> str:
    return state["next_agent"]

workflow.add_conditional_edges(
    "supervisor",
    routing_function,
    {
        "researcher": "research_agent",
        "coder": "code_agent",
        "chart_gen": "chart_agent",
        "finish": "report_writer",
    }
)
```

### 4.4 工具路由（Tool Routing）

当 Agent 调用工具时，LangGraph 自动将执行流转到工具执行节点。这是通过检测 LLM 输出的 `tool_calls` 字段实现的。

```python
def should_continue(state: AgentState) -> str:
    messages = state["messages"]
    last_message = messages[-1]
    # 如果 LLM 输出包含工具调用，则进入工具节点
    if last_message.tool_calls:
        return "tool"
    # 否则结束或进入下一个 Agent
    return "next_agent"

workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "tool": "tool_executor",
        "next_agent": "supervisor",
        "end": END,
    }
)
```

### 4.5 通过 Command 类动态路由

LangGraph 的 `Command` 类允许节点在返回时动态指定下一个目标节点，比条件边更灵活。

```python
from langgraph.types import Command

def transfer_to_bob(state: AgentState) -> Command:
    """当前 Agent 主动将控制权转移给 Bob。"""
    return Command(
        goto="bob",                              # 下一个节点
        update={                                 # 同时更新状态
            "messages": [AIMessage(content="Transferring to Bob...")]
        },
        graph=Command.PARENT,                    # 路由到父图（用于子图）
    )
```

### 4.6 路由决策树 ASCII 图

```
                         +------------------+
                         |   当前节点执行完   |
                         +--------+---------+
                                  |
                    +-------------+-------------+
                    |                           |
          +---------v----------+        +--------v---------+
          | 是否有条件边？       |        | 是否是固定边？     |
          +---------+----------+        +--------+---------+
                    |                            |
               Yes  |  No                   Yes  |
          +---------v----------+                 |
          | 调用 Router 函数   |                 |
          +---------+----------+                 |
                    |                            |
                    v                            v
          +---------+----------+        +--------+---------+
          | Router 返回字符串   |        | 直接进入下一节点 |
          +---------+----------+        +------------------+
                    |
        +-----------+------------+
        |                        |
  匹配路由映射表              不匹配映射表
        |                        |
        v                        v
+-------+--------+     +---------+---------+
| 进入对应节点    |     | 默认分支/报错     |
+----------------+     +-------------------+
        |
        v
+-------+--------+
| 特殊路由方式    |
+-------+--------+
        |
+-------v--------+     +--------v---------+
| Command.goto   |     | Send             |
| (动态指定节点)  |     | (并行分发)       |
+----------------+     +------------------+
```

---

## 5. 并行 Agent 执行

LangGraph 通过 `Send` API 实现动态并行，这是其最强大的特性之一。

### 5.1 Fan-Out 模式（一对多）

一个节点将工作分发到多个并行执行的节点。

```python
from langgraph.types import Send

def research_router(state: ResearchState) -> list[Send]:
    """将研究任务分发到多个并行 Agent。"""
    queries = state["research_questions"]
    return [
        Send(
            "research_worker",          # 目标节点
            {"query": q}                # 发送给该节点的状态子集
        )
        for q in queries
    ]

builder = StateGraph(ResearchState)
builder.add_node("planner", plan_research)
builder.add_node("research_worker", execute_research)  # 并行 Worker
builder.add_node("aggregator", aggregate_results)

builder.add_edge(START, "planner")
builder.add_conditional_edges(
    "planner",
    research_router,                 # 返回 Send 对象列表
)
builder.add_edge(["research_worker"], "aggregator")  # fan-in
```

```
Fan-Out 拓扑：

     +-----------+
     |  Planner   |
     +-----+-----+
           |
   +-------+-------+-------+-------+
   |       |       |       |       |
   v       v       v       v       v
 Worker1  Worker2  Worker3  Worker4 ...  <- 并行执行
   |       |       |       |       |
   +-------+-------+-------+-------+
           |
     +-----v-----+
     | Aggregator |  <- 等待所有 Worker 完成
     +-----------+
```

### 5.2 Fan-In 模式（多对一）

fan-in 通过 `add_edge` 指定多个源节点来实现。

```python
# 两个检索器并行运行，都完成后进入 QA 节点
workflow.add_node("retriever_one", retrieve_from_db)
workflow.add_node("retriever_two", retrieve_from_web)
workflow.add_node("qa", answer_question)

workflow.add_edge("rewrite_query", "retriever_one")
workflow.add_edge("rewrite_query", "retriever_two")
workflow.add_edge(
    ["retriever_one", "retriever_two"],  # 两个源节点
    "qa"                                  # 目标节点
)
```

### 5.3 Map-Reduce 模式

Map-Reduce 是 fan-out + fan-in 的组合：将输入切分，并行处理，然后合并结果。

```python
from langgraph.types import Send
from typing import Annotated
import operator

# ---------- 状态定义 ----------
class MapReduceState(TypedDict):
    input_data: list[str]
    chunked_data: Annotated[list, operator.add]    # 收集并行结果
    final_output: str

# ---------- Map 阶段 ----------
def chunker(state: MapReduceState) -> list[Send]:
    """将输入数据分片并分发。"""
    chunks = split_into_chunks(state["input_data"], chunk_size=5)
    return [Send("processor", {"chunk": chunk}) for chunk in chunks]

def process_chunk(state: dict) -> dict:
    """处理一个数据分片（并行执行）。"""
    chunk = state["chunk"]
    result = llm.invoke(f"Process this chunk: {chunk}")
    return {"chunked_data": [result.content]}

# ---------- Reduce 阶段 ----------
def reducer(state: MapReduceState) -> str:
    """合并所有并行结果。"""
    all_results = state["chunked_data"]
    summary = llm.invoke(f"Summarize: {all_results}")
    return {"final_output": summary.content}

# ---------- 构建图 ----------
builder = StateGraph(MapReduceState)
builder.add_node("chunker", chunker)
builder.add_node("processor", process_chunk)   # 多个并行实例
builder.add_node("reducer", reducer)

builder.add_edge(START, "chunker")
builder.add_conditional_edges("chunker", chunker)     # 路由到 Send
builder.add_edge("processor", "reducer")               # fan-in (等待所有 processor)
builder.add_edge("reducer", END)
```

```
Map-Reduce 拓扑：

     +---------+
     |  Input   |
     +----+----+
          |
     +----v----+       +------------------+
     | Chunker | ----> | Send(chunk_1)    |
     +----+----+       | Send(chunk_2)    |
          |            | Send(chunk_3)    |
          |            | ...              |
          |            +------------------+
          |                     |
    +-----+-----+     +---------+---------+
    | chunk_1   |     |     chunk_2       |  ...  <- Map 阶段（并行）
    +-----+-----+     +---------+---------+
          |                     |
    +-----+---------------------+---------+
          |                               |
    +-----v-----+                 +-------v------+
    | chunk_1   |                 |  chunk_2     |
    | result    |                 |  result      |
    +-----+-----+                 +-------+------+
          |                               |
          +---------+---------------------+
                    |
              +-----v-----+
              |  Reducer   |  <- Reduce 阶段
              +-----+------+
                    |
              +-----v------+
              |  Final     |
              |  Output    |
              +------------+
```

### 5.4 并行执行的核心机制

LangGraph 的并行通过**任务 ID（task_id）**和**通道版本号**实现：

1. `Send` 创建多个任务，每个任务有唯一的 `task_id`。
2. 所有任务分配到执行线程池，并发运行。
3. 每个任务写入状态时，通过 `task_id` 隔离在独立的 pending write 队列中。
4. 在 fan-in 点（所有前置节点完成），所有 pending write 被提交给 reducer。
5. reducer 生成最终的状态更新。

```
Send API 内部流程：

Send("node_x", input_1) ----+
                            |
Send("node_x", input_2) ----+--> 线程池调度
                            |       |
Send("node_x", input_3) ----+       v
                            |   task_id: uuid_1 -> 执行 -> 暂存写入
                            |   task_id: uuid_2 -> 执行 -> 暂存写入
                            |   task_id: uuid_3 -> 执行 -> 暂存写入
                                    |
                            所有完成？ --> Yes --> Reducer 合并
                                                    |
                                             Checkpointer 保存
```

---

## 6. 人机协同模式（Human-in-the-Loop）

LangGraph 提供了一流的 Human-in-the-Loop (HITL) 支持，通过 `interrupt()` 函数在任意节点暂停执行。

### 6.1 interrupt() 函数

在节点函数内部调用 `interrupt()` 会暂停图执行，等待外部输入。

```python
from langgraph.types import interrupt

def approval_node(state: AgentState) -> dict:
    """需要用户批准的节点。"""
    proposal = state["agent_output"]

    # 暂停执行，等待人类反馈
    response = interrupt(
        {
            "question": "是否批准以下输出？",
            "proposal": proposal,
            "options": ["approve", "reject", "modify"]
        }
    )

    if response == "approve":
        return {"approved": True, "feedback": None}
    elif response == "reject":
        return {"approved": False, "feedback": "用户拒绝"}
    elif response == "modify":
        modified = input("请输入修改建议: ")  # 由外部控制器提供
        return {"approved": True, "modified_output": modified}
```

### 6.2 interrupt_before / interrupt_after

在编译图时指定在哪些节点之前或之后自动中断。

```python
# 在 approval 节点之前中断
graph = workflow.compile(
    interrupt_before=["approval_node"],
    checkpointer=checkpointer,    # 需要 checkpointer 才能中断
)

# 在 approval 节点之后中断
graph = workflow.compile(
    interrupt_after=["approval_node"],
    checkpointer=checkpointer,
)

# 多个中断点
graph = workflow.compile(
    interrupt_before=["research_agent", "code_agent", "deploy_agent"],
    interrupt_after=["approval_node"],
    checkpointer=checkpointer,
)
```

### 6.3 恢复执行

中断后通过 `invoke` 或 `Command` 恢复。

```python
# 第一次调用：会在 interrupt_before 节点前暂停
result = graph.invoke(
    {"messages": [HumanMessage("帮我写一个 Python 脚本")]},
    config={"configurable": {"thread_id": "session_1"}}
)

# 此时图暂停，检查状态
state = graph.get_state({"configurable": {"thread_id": "session_1"}})
print("暂停于:", state.next)  # 显示下一个要执行的节点

# 恢复执行：通过 Command 传递人类输入
result = graph.invoke(
    Command(resume="approve"),    # 传递恢复值
    config={"configurable": {"thread_id": "session_1"}}
)
```

### 6.4 检查点恢复

中断时，Checkpointer 保存了完整的执行状态。可以随时查询和恢复。

### 6.5 Human-in-the-Loop 流程图

```
+------------------+
|  用户发送请求     |
+--------+---------+
         |
+--------v---------+       +--------------------------+
|  Agent 开始执行   |       | Checkpointer:            |
+--------+---------+       | 保存 checkpoint_1        |
         |                 +--------------------------+
+--------v---------+
|  interrup_before |
|  = approval_node |
+--------+---------+
         |
+--------v---------+       +--------------------------+
| 审批节点暂停执行   |       | Checkpointer:            |
| (interrupt 调用)  |------>| 保存 checkpoint_2        |
+--------+---------+       | state.next = [approve]   |
         |                 +--------------------------+
         |
+--------v---------+
|  等待人类输入     |
|  (暂停状态)       |
+--------+---------+
         |
    +----+----+
    |         |
 批准       拒绝
    |         |
    v         v
继续执行   终止/修改
    |
+--------v---------+       +--------------------------+
|  Command(resume  |       | Checkpointer:            |
|  = "approve")    |------>| 保存 checkpoint_3        |
+--------+---------+       +--------------------------+
         |
+--------v---------+
|  审批节点恢复执行  |
|  interrupt()      |
|  返回 "approve"   |
+--------+---------+
         |
+--------v---------+
|  后续节点继续执行  |
+--------+---------+
         |
+--------v---------+
|  最终输出         |
+------------------+
```

---

## 7. 流式与持久化

### 7.1 流式（Streaming）

LangGraph 支持多种流式模式，适合构建实时响应式 UI。

#### Node 级别事件

```python
# 逐节点流式输出
for event in graph.stream(
    {"messages": [HumanMessage("分析这份报告")]},
    config={"configurable": {"thread_id": "s1"}},
    stream_mode="updates"        # 每个节点的完整状态更新
):
    for node_name, update in event.items():
        print(f"[{node_name}]: {update}")
```

| stream_mode | 输出内容 | 适用场景 |
|---|---|---|
| `"values"` | 每个步骤的完整状态 | 调试、回放 |
| `"updates"` | 每个节点的增量更新 | UI 逐步更新 |
| `"messages"` | LLM token 级别流式输出 | 流式显示 LLM 回复 |
| `"debug"` | 完整调试信息 | 调试 |
| `"custom"` | 自定义流式事件 | 自定义进度报告 |

#### Token 级别流式

```python
# Token 级别的 LLM 流式输出
async for event in graph.astream_events(
    {"messages": [HumanMessage("讲个故事")]},
    version="v2",
):
    kind = event["event"]
    if kind == "on_chat_model_stream":
        content = event["data"]["chunk"].content
        if content:
            print(content, end="", flush=True)
```

#### 自定义流式事件

```python
from langgraph.config import get_stream_writer

def progress_reporting_node(state: AgentState) -> dict:
    """在节点执行过程中发送自定义流式事件。"""
    writer = get_stream_writer()
    steps = ["加载数据", "分析", "生成报告", "格式化"]

    for i, step in enumerate(steps):
        writer({"progress": f"{step}...", "percent": (i + 1) * 25})
        time.sleep(0.5)  # 模拟耗时操作

    return {"report": "最终报告"}

# 消费自定义事件（需要 v3 或注册 transformer）
for event in graph.stream(
    input,
    stream_mode="custom"
):
    print(event)  # {"progress": "加载数据...", "percent": 25}
```

### 7.2 Checkpointer（检查点）

Checkpointer 在图的每个超步（super-step）结束后保存完整状态。

| Checkpointer | 存储方式 | 适用场景 |
|---|---|---|
| `MemorySaver` | 内存 | 开发调试、单会话 |
| `SqliteSaver` | SQLite 文件 | 单机持久化 |
| `AsyncSqliteSaver` | SQLite（异步） | 异步应用 |
| `PostgresSaver` | PostgreSQL | 生产环境、多实例 |
| `MongoDBSaver` | MongoDB | 文档数据库 |

**使用示例：**

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.postgres import PostgresSaver

# MemorySaver（开发用）
checkpointer = MemorySaver()

# SqliteSaver（单机持久化）
checkpointer = SqliteSaver.from_conn_string("checkpoints.db")

# PostgresSaver（生产环境）
checkpointer = PostgresSaver.from_conn_string(
    "postgres://user:pass@localhost:5432/langgraph"
)
await checkpointer.setup()  # 创建表（首次使用）

graph = workflow.compile(checkpointer=checkpointer)
```

### 7.3 检查点内部结构

```python
# Checkpoint 的内部结构
checkpoint = {
    "v": 4,                         # 版本号
    "ts": "2024-07-31T20:14:19Z",   # 时间戳
    "id": "1ef4f797-...",           # 唯一 ID
    "channel_values": {              # 当前状态值
        "messages": [...],
        "research_notes": [...],
        "final_report": "..."
    },
    "channel_versions": {            # 每个通道的版本（冲突检测）
        "__start__": 2,
        "messages": 5,
        "research_notes": 3
    },
    "versions_seen": {               # 每个节点看到的版本
        "agent_a": {"messages": 5},
        "agent_b": {"messages": 5}
    }
}
```

### 7.4 线程隔离

每个会话（Thread）有独立的 checkpoint 命名空间，通过 `thread_id` 隔离。

```python
# 三个独立会话，互不干扰
config_1 = {"configurable": {"thread_id": "user_sarah"}}
config_2 = {"configurable": {"thread_id": "user_bob"}}
config_3 = {"configurable": {"thread_id": "batch_job_42"}}

# 每个会话有自己的状态链
result_1 = graph.invoke(input1, config=config_1)
result_2 = graph.invoke(input2, config=config_2)
result_3 = graph.invoke(input3, config=config_3)

# 可以随时恢复任意会话
state_sarah = graph.get_state(config_1)
state_bob = graph.get_state(config_2)

# 列出会话的所有历史 checkpoint
for cp in checkpointer.list(config_1):
    print(cp)
```

### 7.5 Checkpoint Save 和 Resume 生命周期

```
+=================+
| 图执行开始       |
+=================+
        |
+-------v--------+
| channel_values  |
| = 初始输入      |
+-------+--------+
        |
+=======v========+    +------------------------+
| 等待可执行节点   |    | LangGraph 运行时引擎    |
+=======+========+    +------------------------+
        |
+-------v--------+
| 选择下一个节点   |
| (根据拓扑顺序)  |
+-------+--------+
        |
+-------v--------+    +------------------------+
| 执行节点        |    | 节点函数返回状态更新    |
+-------+--------+    | {"key": new_value}    |
        |             +------------------------+
+-------v--------+
| 应用状态更新    |
| (通过 Reducer) |
+-------+--------+
        |
+-------v--------+    +------------------------+
| Checkpointer   |    | 序列化: msgpack/json   |
| .put(config,   |    | 存储: SQL/PG/Mongo     |
|  checkpoint)   |    | 索引: thread_id +      |
+-------+--------+    |       checkpoint_ns    |
        |             +------------------------+
+-------v--------+
| 检查是否有      |
| 中断点          |
+-------+--------+
        |
   +----+----+
   |         |
 有中断     无中断
   |         |
   v         v
暂停，    继续下一
等待输入  个节点
```

---

## 8. 关键源码实现

### 8.1 多 Agent StateGraph 设置

```python
from typing import Annotated, Literal, Sequence
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage, SystemMessage
from langchain_anthropic import ChatAnthropic
import operator

# ===== 1. 状态定义 =====
class MultiAgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    research_results: Annotated[list[str], operator.add]
    code_output: str
    current_task: str
    step_count: int

# ===== 2. Agent 节点 =====
llm = ChatAnthropic(model="claude-sonnet-4-20250514")

def supervisor_agent(state: MultiAgentState) -> dict:
    """Supervisor Agent：分析状态，决定下一步。"""
    prompt = """You are a supervisor. Analyze the conversation and decide next step.
    Options: research | code | review | finalize

    Current step count: {step_count}
    Research results so far: {results}
    """

    response = llm.invoke([
        SystemMessage(prompt.format(
            step_count=state.get("step_count", 0),
            results=state.get("research_results", [])
        )),
        *state["messages"]
    ])

    return {
        "messages": [response],
        "current_task": response.content.strip(),
        "step_count": state.get("step_count", 0) + 1
    }

def research_agent(state: MultiAgentState) -> dict:
    """Research Agent：执行研究任务。"""
    query = state["messages"][-1].content
    result = llm.invoke(f"Research: {query}")
    return {
        "messages": [result],
        "research_results": [result.content]
    }

def code_agent(state: MultiAgentState) -> dict:
    """Code Agent：编写代码。"""
    spec = state["messages"][-1].content
    code = llm.invoke(f"Write code for: {spec}")
    return {"messages": [code], "code_output": code.content}

def review_agent(state: MultiAgentState) -> dict:
    """Review Agent：审阅代码。"""
    code = state.get("code_output", "")
    review = llm.invoke(f"Review this code:\n{code}")
    return {"messages": [review]}

# ===== 3. 路由函数 =====
def supervisor_router(state: MultiAgentState) -> str:
    """根据 Supervisor 的决策路由到对应 Agent。"""
    task = state.get("current_task", "").strip().lower()
    if "research" in task:
        return "research"
    elif "code" in task:
        return "code"
    elif "review" in task:
        return "review"
    elif "finalize" in task:
        return "finalize"
    return "research"  # default

# ===== 4. 构建图 =====
builder = StateGraph(MultiAgentState)

# 添加节点
builder.add_node("supervisor", supervisor_agent)
builder.add_node("research", research_agent)
builder.add_node("code", code_agent)
builder.add_node("review", review_agent)

# 起始边
builder.add_edge(START, "supervisor")

# 条件边：从 supervisor 到各 Agent
builder.add_conditional_edges(
    "supervisor",
    supervisor_router,
    {
        "research": "research",
        "code": "code",
        "review": "review",
        "finalize": END,
    }
)

# Agent 执行完后返回 supervisor 做下一次决策
builder.add_edge(["research", "code", "review"], "supervisor")

# ===== 5. 编译 =====
checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)
```

### 8.2 条件路由实现

```python
# 带工具调用的 Agent 路由模式
from langgraph.prebuilt import ToolNode, tools_condition

# 定义工具
@tool
def web_search(query: str) -> str:
    """搜索网络。"""
    return f"网络搜索结果: {query}"

@tool
def calculate(expression: str) -> str:
    """数学计算。"""
    return str(eval(expression))

# 创建工具节点
tools = ToolNode([web_search, calculate])

# Agent 节点：绑定工具
def agent(state: AgentState) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

# 路由逻辑：内置的 tools_condition 检查 tool_calls
# 如果有工具调用 -> "tools"，否则 -> END 或下一节点

builder = StateGraph(AgentState)
builder.add_node("agent", agent)
builder.add_node("tools", tools)

builder.add_edge(START, "agent")
builder.add_conditional_edges(
    "agent",
    tools_condition,      # 内置路由：检测 tool_calls
)

# 工具执行完后回到 agent
builder.add_edge("tools", "agent")
```

### 8.3 并行 Agent Fan-Out / Fan-In

```python
from langgraph.types import Send

# ===== 并行分析器模式 =====
class AnalysisState(TypedDict):
    input_text: str
    parallel_analyses: Annotated[list[str], operator.add]
    synthesis: str

# Fan-out：分发到多个分析 Agent
def analysis_dispatcher(state: AnalysisState) -> list[Send]:
    text = state["input_text"]
    # 为每个分析维度创建一个并行任务
    dimensions = [
        ("sentiment", {"text": text, "dimension": "情感分析"}),
        ("entities", {"text": text, "dimension": "实体提取"}),
        ("summary", {"text": text, "dimension": "摘要生成"}),
        ("topics", {"text": text, "dimension": "主题分类"}),
    ]
    return [Send("analyzer", d[1]) for d in dimensions]

# 并行 Worker
def analyzer(state: dict) -> dict:
    """单个分析维度处理（并行执行）。"""
    text = state["text"]
    dimension = state["dimension"]
    result = llm.invoke(f"执行{dimension}:\n{text}")
    return {"parallel_analyses": [f"[{dimension}]: {result.content}"]}

# Fan-in：合并结果
def synthesis_node(state: AnalysisState) -> dict:
    """合并所有并行分析结果。"""
    all_analyses = "\n".join(state["parallel_analyses"])
    final = llm.invoke(f"综合以下分析:\n{all_analyses}")
    return {"synthesis": final.content}

# 构建图
builder = StateGraph(AnalysisState)
builder.add_node("dispatcher", analysis_dispatcher)
builder.add_node("analyzer", analyzer)
builder.add_node("synthesis", synthesis_node)

builder.add_edge(START, "dispatcher")
builder.add_conditional_edges("dispatcher", analysis_dispatcher)
builder.add_edge(["analyzer"], "synthesis")    # fan-in: 等待所有 analyzer
builder.add_edge("synthesis", END)
```

### 8.4 Human-in-the-Loop 中断模式

```python
from langgraph.types import interrupt, Command

# ===== 需要人工批准的代码生成 =====
class CodeReviewState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    generated_code: str
    code_review: str
    approved: bool
    feedback: str

def code_generator(state: CodeReviewState) -> dict:
    """生成代码。"""
    spec = state["messages"][0].content
    code = llm.invoke(f"Generate Python code: {spec}")
    return {"generated_code": code.content, "messages": [code]}

def code_reviewer(state: CodeReviewState) -> dict:
    """自动审阅代码。"""
    code = state["generated_code"]
    review = llm.invoke(f"Review:\n{code}")
    return {"code_review": review.content}

def human_approval(state: CodeReviewState) -> dict:
    """人类审批节点。"""
    code = state["generated_code"]
    review = state["code_review"]

    # 暂停执行，等待人类决策
    decision = interrupt({
        "type": "approval_request",
        "code": code,
        "review": review,
        "options": ["approve", "reject", "revise"]
    })

    if decision == "approve":
        return {"approved": True}
    elif decision == "reject":
        return {"approved": False, "feedback": "用户拒绝"}
    elif decision == "revise":
        return {"approved": False, "feedback": "需要修改"}

def should_deploy(state: CodeReviewState) -> str:
    """根据人类决定路由。"""
    if state["approved"]:
        return "deploy"
    else:
        return END

# 构建图
builder = StateGraph(CodeReviewState)
builder.add_node("generator", code_generator)
builder.add_node("reviewer", code_reviewer)
builder.add_node("approval", human_approval)

builder.add_edge(START, "generator")
builder.add_edge("generator", "reviewer")
builder.add_edge("reviewer", "approval")

builder.add_conditional_edges(
    "approval",
    should_deploy,
    {"deploy": END}
)

# 编译时指定中断点
graph = builder.compile(
    interrupt_before=["approval"],      # approval 节点前暂停
    checkpointer=MemorySaver(),
)

# ===== 使用示例 =====
# 第 1 步：发起请求，执行到审批节点前暂停
thread_config = {"configurable": {"thread_id": "code_session_1"}}
result = graph.invoke(
    {"messages": [HumanMessage("写一个快速排序")]},
    config=thread_config
)

# 第 2 步：查看当前状态
state = graph.get_state(thread_config)
print(f"暂停于: {state.next}")     # ['approval']
print(f"生成代码: {state.values['generated_code']}")

# 第 3 步：恢复执行，传递人类决策
result = graph.invoke(
    Command(resume="approve"),    # 人类说"批准"
    config=thread_config
)

print(f"最终状态: {result}")
```

### 8.5 Checkpoint Save 和 Resume

```python
from langgraph.checkpoint.sqlite import SqliteSaver

# ===== 持久化配置 =====
checkpointer = SqliteSaver.from_conn_string("sessions.db")
graph = workflow.compile(checkpointer=checkpointer)

# ===== 保存执行（自动） =====
config = {"configurable": {"thread_id": "session_42"}}
result = graph.invoke(input, config=config)
# 每个步骤后自动保存 checkpoint

# ===== 查询历史 =====
# 获取当前状态
state = graph.get_state(config)
print(f"当前状态: {state.values}")
print(f"下一节点: {state.next}")

# 获取所有历史 checkpoint
for cp in graph.get_state_history(config):
    print(f"Checkpoint: {cp.metadata['checkpoint_id']} at {cp.metadata['timestamp']}")

# ===== 从历史 checkpoint 恢复 =====
# 回退到特定 checkpoint
checkpoint_id = "1ef4f797-8335-6428-8001-8a1503f9b875"
revert_config = {
    "configurable": {
        "thread_id": "session_42",
        "checkpoint_id": checkpoint_id,   # 指定历史 checkpoint
    }
}

# 从该 checkpoint 的状态继续执行
result = graph.invoke(
    Command(resume="continue"),
    config=revert_config
)

# ===== 多会话隔离 =====
# 两个完全隔离的会话
config_alice = {"configurable": {"thread_id": "alice_chat"}}
config_bob = {"configurable": {"thread_id": "bob_chat"}}

result_alice = graph.invoke({"messages": [HumanMessage("我的名字是 Alice")]}, config=config_alice)
result_bob = graph.invoke({"messages": [HumanMessage("你是谁？")]}, config=config_bob)

alice_state = graph.get_state(config_alice)
bob_state = graph.get_state(config_bob)

# Alice 的状态知道她的名字，Bob 的不知道
print(alice_state.values["messages"][-1].content)  # 记得 Alice
print(bob_state.values["messages"][-1].content)     # 不记得 Alice
```

---

## 9. 局限性与改进

### 9.1 陡峭的学习曲线

- **问题**：LangGraph 的概念层较多（StateGraph, Channel, Reducer, Checkpointer, Send, Command, interrupt），新手很难快速上手。
- **改进**：从简单的链式 agent 开始，逐步引入条件边、并行、中断。官方提供的预构建组件（`create_react_agent`、`ToolNode`）可以降低入门门槛。

### 9.2 调试复杂度

- **问题**：图的执行是隐式的。节点间的数据流动不直观，中断和恢复的调试困难。
- **改进**：
  - 使用 `stream_mode="debug"` 输出完整的执行过程。
  - 使用 `graph.get_state_history()` 回溯 checkpoint 链。
  - 在开发阶段使用 `MemorySaver` 并频繁检查状态。
  - LangSmith 提供了图的执行可视化工具。

### 9.3 状态 Schema 的刚性

- **问题**：TypedDict 的状态定义是静态的。在运行时动态添加字段需要修改 schema 定义。
- **改进**：
  - 使用 `dict` 作为状态类型（失去类型检查）。
  - 使用 `Annotated` 配合 `Optional` 标记可选字段。
  - 考虑使用 Pydantic `BaseModel`（支持继承和动态字段）。

```python
# 更灵活的状态设计（但牺牲了一些类型安全）
class FlexibleState(TypedDict, total=False):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    runtime_data: dict                    # 通用数据容器
    custom_fields: Annotated[dict, merge_dicts]  # 自定义合并
```

### 9.4 高级模式的文档缺口

- **问题**：LangGraph 的官方文档偏基础。高级模式（subgraph 嵌套通信、动态图拓扑、跨图 checkpoint 共享）缺乏系统性的文档和示例。
- **改进**：
  - 阅读 LangGraph 的测试用例（`test_pregel.py`）获得高级模式灵感。
  - 参考 LangGraph 的 examples 目录中的 Notebook。
  - 社区实践通过 LangSmith Hub 分享。

### 9.5 性能考量

- **问题**：
  - 每个超步后都进行 checkpoint 写入，在高频场景下 I/O 开销显著。
  - 状态对象在节点间序列化/反序列化，大型状态（如长文档、图像）影响性能。
- **改进**：
  - 对于高吞吐场景，使用 `PostgresSaver` 而不是 `SqliteSaver`。
  - 考虑将大型二进制数据存储在外部存储中，状态中只保留引用（URL/路径）。

### 9.6 缺乏原生循环检测

- **问题**：LangGraph 允许图中有循环（Agent 回到 Supervisor），但运行时不会自动检测无限循环。
- **改进**：在节点和路由函数中手动添加步骤计数器或最大迭代限制。

```python
def supervisor_with_guard(state: MultiAgentState) -> str:
    """带有循环上限的 Supervisor。"""
    if state.get("step_count", 0) > 10:
        return "finalize"      # 超过上限，强制结束
    # ... 正常路由逻辑 ...
```
