# 8.6.1 CrewAI 编排流程源码分析 (CrewAI Orchestration Flow Source Code Analysis)

## 1. 简单介绍

CrewAI 是一个基于角色扮演（role-playing）的多 Agent 协作框架。其核心编排引擎围绕四个实体构建：**Agent**（角色）、**Task**（任务）、**Crew**（团队）和 **Process**（流程）。四个实体以"剧本式"（script-like）的方式协同工作：

- 开发者定义一组 `Agent`，每个 Agent 拥有角色、目标、背景故事和工具。
- 定义一组 `Task`，每个 Task 包含任务描述、期望输出和执行 Agent。
- 将这些 Agent 和 Task 组装到 `Crew` 中，指定 `Process` 类型（Sequential 或 Hierarchical）。
- 调用 `Crew.kickoff()` 启动执行，Process 负责编排任务在 Agent 间的流转。

CrewAI 的设计哲学是"声明式编排"——开发者声明"谁做什么"，框架负责"怎么协调"。这与 LangGraph 的"图编排"哲学形成鲜明对比：CrewAI 牺牲了底层控制力，换取了高层抽象和快速上手。

---

## 2. 整体架构

CrewAI 的架构分为五个层次，从底层 LLM 集成到顶层流程编排：

```
+------------------------------------------------------------------+
|                  Layer 5: Process  (编排层)                        |
|  +------------------+  +---------------------------+              |
|  | SequentialProcess |  | HierarchicalProcess       |             |
|  | 顺序执行,线性传递  |  |  Manager Agent 委派/监控   |             |
|  +------------------+  +---------------------------+              |
+------------------------------------------------------------------+
                          |
                          v
+------------------------------------------------------------------+
|                  Layer 4: Crew  (团队层)                           |
|  +------------------------------------------------------------+  |
|  | Crew { agents, tasks, process, kickoff() }                 |  |
|  | - validate_config() -> _create_agents() -> _create_tasks() |  |
|  | -> process.execute() -> collect_results()                  |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
                          |
                          v
+------------------------------------------------------------------+
|                  Layer 3: Task  (任务层)                           |
|  +------------------------------------------------------------+  |
|  | Task { description, expected_output, agent, tools }         |  |
|  | - execute(context) -> _build_prompt() -> agent.execute()    |  |
|  | - 生命周期: created -> assigned -> executed -> completed    |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
                          |
                          v
+------------------------------------------------------------------+
|                  Layer 2: Agent  (角色层)                          |
|  +------------------------------------------------------------+  |
|  | Agent { role, goal, backstory, llm, tools }                 |  |
|  | - execute_task(task) -> _build_prompt() -> _invoke_llm()    |  |
|  | - _handle_tool_call() -> context_window_management()        |  |
|  | - 支持 Tool 共享、角色扮演、委托（delegation）                |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
                          |
                          v
+------------------------------------------------------------------+
|                  Layer 1: LLM 集成层                               |
|  +------------+ +----------+ +----------+ +----------+          |
|  | OpenAI     | | Anthropic| | Google   | | Ollama   |  ...     |
|  | GPT-4/4o  | | Claude   | | Gemini   | | (本地)   |          |
|  +------------+ +----------+ +----------+ +----------+          |
|  所有 LLM 通过统一 BaseLLM 接口适配                                |
+------------------------------------------------------------------+
```

### 各层职责

| 层 | 核心类/组件 | 职责 |
|----|------------|------|
| **Layer 1** | `BaseLLM`, `LLMConnector` | 统一 LLM 调用接口，Provider 适配，Token 计数 |
| **Layer 2** | `Agent` | 角色扮演，Prompt 构建，工具调用，错误恢复 |
| **Layer 3** | `Task` | 任务定义，上下文注入，输出解析，回调触发 |
| **Layer 4** | `Crew` | 团队组装，配置校验，生命周期管理，结果聚合 |
| **Layer 5** | `Process` | 执行流程定义，任务编排，Agent 协调，状态追踪 |

### 依赖方向

```
LLM Layer  ←  Agent Layer  ←  Task Layer  ←  Crew Layer  ←  Process Layer
   (底层)                                              (高层编排)
```

每个上层依赖下层：Crew 依赖 Task 和 Agent 的定义，Agent 依赖 LLM 层进行推理。Process 是最高抽象，决定任务流动的拓扑结构。

---

## 3. 核心执行流程：Crew.kickoff()

`kickoff()` 是 CrewAI 的唯一入口方法。其执行流程分为五个阶段：

```
┌──────────────────────────────────────────────────────────────────┐
│                     Crew.kickoff()                                │
│                                                                   │
│  Phase 1: 校验配置                                                 │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ validate_config()                                         │    │
│  │  ├── agents 列表不为空                                    │    │
│  │  ├── tasks 列表不为空                                     │    │
│  │  ├── task 数量和 agent 数量匹配（Sequential）              │    │
│  │  ├── manager_llm 已设置（Hierarchical）                   │    │
│  │  └── 所有 task.agent 在 agents 列表中                     │    │
│  └──────────────────────────────────────────────────────────┘    │
│                           │                                       │
│                           v                                       │
│  Phase 2: 创建 Agent 实例                                         │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ _create_agents()                                         │    │
│  │  ├── 为每个 Agent 绑定 LLM                               │    │
│  │  ├── 注入共享工具（如果配置了 share_tools=True）          │    │
│  │  └── 设置 Agent 级别的回调（step/complete callback）      │    │
│  └──────────────────────────────────────────────────────────┘    │
│                           │                                       │
│                           v                                       │
│  Phase 3: 创建 Task 实例                                         │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ _create_tasks()                                          │    │
│  │  ├── 为每个 Task 绑定 Agent                               │    │
│  │  ├── 设置 Task 级别的工具（task.tools）                    │    │
│  │  └── 注入上下文（context 字段）                            │    │
│  └──────────────────────────────────────────────────────────┘    │
│                           │                                       │
│                           v                                       │
│  Phase 4: 执行 Process                                           │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ process.execute(tasks, agents, ...)                      │    │
│  │  ├── SequentialProcess: 逐一执行，输出传递给下一个         │    │
│  │  └── HierarchicalProcess: Manager 委派/监控/评估          │    │
│  └──────────────────────────────────────────────────────────┘    │
│                           │                                       │
│                           v                                       │
│  Phase 5: 收集结果                                                │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ collect_results()                                        │    │
│  │  ├── 汇总所有 Task 的 output                              │    │
│  │  ├── 按 task 顺序排列                                     │    │
│  │  └── 返回 CrewOutput（包含 raw, tasks_output, ...）       │    │
│  └──────────────────────────────────────────────────────────┘    │
│                           │                                       │
│                           v                                       │
│                   返回 CrewOutput                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 源码级伪代码

```python
class Crew:
    agents: List[Agent]
    tasks: List[Task]
    process: ProcessType  # sequential | hierarchical
    manager_llm: Optional[BaseLLM]
    share_tools: bool = False
    verbose: bool = False

    class CrewOutput:
        raw: str
        tasks_output: List[TaskOutput]

        def __str__(self):
            return self.raw

    def kickoff(self) -> CrewOutput:
        # Phase 1
        self._validate_config()

        # Phase 2 - 深拷贝 Agent 并绑定 LLM
        agent_instances = self._create_agents()

        # Phase 3 - 深拷贝 Task 并绑定 Agent
        task_instances = self._create_tasks(agent_instances)

        # Phase 4 - 根据 process 类型选择执行策略
        if self.process == ProcessType.sequential:
            results = self._run_sequential_process(task_instances, agent_instances)
        elif self.process == ProcessType.hierarchical:
            results = self._run_hierarchical_process(task_instances, agent_instances)

        # Phase 5
        return self._build_output(results)

    def _validate_config(self):
        assert len(self.agents) > 0, "At least one agent required"
        assert len(self.tasks) > 0, "At least one task required"
        if self.process == ProcessType.hierarchical:
            assert self.manager_llm is not None, "manager_llm required for hierarchical process"
        # 检查 task.agent 都在 agents 列表中
        for task in self.tasks:
            assert task.agent in self.agents, f"Agent for task '{task.description}' not in crew"

    def _create_agents(self) -> List[Agent]:
        instances = []
        for agent_config in self.agents:
            agent = Agent(
                role=agent_config.role,
                goal=agent_config.goal,
                backstory=agent_config.backstory,
                llm=self._bind_llm(agent_config.llm),
                tools=list(agent_config.tools)  # 深拷贝
            )
            # 如果设置了 share_tools，所有 Agent 共享工具集
            if self.share_tools:
                all_tools = self._collect_all_tools()
                agent.tools.extend(all_tools)
            instances.append(agent)
        return instances

    def _create_tasks(self, agents: List[Agent]) -> List[Task]:
        instances = []
        for task_config in self.tasks:
            # 找到对应的 Agent 实例
            matched_agent = next(a for a in agents if a.role == task_config.agent.role)
            task = Task(
                description=task_config.description,
                expected_output=task_config.expected_output,
                agent=matched_agent,
                tools=list(task_config.tools) if task_config.tools else [],
                context=task_config.context  # 可以手动注入前序上下文
            )
            instances.append(task)
        return instances
```

---

## 4. Sequential Process（顺序流程）

Sequential Process 是最简单的执行模式：任务按照定义的顺序逐一执行，前一个 Task 的输出作为后一个 Task 的上下文（context）。Agent 之间没有直接通信，所有信息通过 Task 输出传递。

### 流程图

```
                        Sequential Process
    =========================================================

    输入: tasks = [T1, T2, T3, ..., Tn]  (顺序排列)
          agents = [A1, A2, A3, ..., An]  (每个 task 绑定一个 agent)

    步骤:

    T1 (Agent A1)                          T2 (Agent A2)
    ┌─────────────────┐                   ┌─────────────────┐
    │ description: 研究 │                   │ description: 分析 │
    │ expected_output:  │                   │ expected_output:  │
    │   研究报告        │                   │   分析报告        │
    │                   │                   │                   │
    │ execute() ────────┼─── raw_output ───▶│ context =         │
    │                   │   传递给 T2       │   T1.output       │
    │ output = "..."    │                   │                   │
    └─────────────────┘                   │ execute()         │
                                          │                   │
                                          │ output = "..."    │
                                          └────────┬──────────┘
                                                   │
                                                   │ raw_output
                                                   │ 传递给 T3
                                                   v
                                          ┌─────────────────┐
                                          │      ...        │
                                          │  (后续 Task)    │
                                          └─────────────────┘
                                                   │
                                                   v
                                          Tn (Agent An)
                                          ┌─────────────────┐
                                          │ context =        │
                                          │   T(n-1).output  │
                                          │                   │
                                          │ execute()         │
                                          │                   │
                                          │ output = "最终结果"│
                                          └────────┬──────────┘
                                                   │
                                                   v
                                          最终 CrewOutput
                                          (所有 task 输出的聚合)
```

### 上下文传递机制

Sequential Process 的核心机制是 **task context 链式注入**：

```
Task 1: context = ""  (没有前序任务)
         prompt = system_prompt(A1) + task_prompt(T1)
         output = "原始研究数据"

Task 2: context = Task 1.output
         prompt = system_prompt(A2) + task_prompt(T2) + "Context from previous tasks:\n" + output_T1
         output = "数据分析结果"

Task 3: context = Task 1.output + Task 2.output
         prompt = system_prompt(A3) + task_prompt(T3) + "Context from previous tasks:\n" + output_T1 + "\n" + output_T2
         output = "最终报告"
```

这种机制确保了每个后续 Agent 都能感知到前面所有 Agent 的工作成果，形成信息流水线。

### 源码级伪代码

```python
class SequentialProcess(Process):
    def execute(
        self,
        tasks: List[Task],
        agents: List[Agent],
        crew: Crew
    ) -> List[TaskOutput]:
        results = []

        for i, task in enumerate(tasks):
            # 构建上下文：收集所有前序任务的输出
            context = self._build_context(results)

            # 分配 Agent 执行任务
            agent = task.agent
            task_output = agent.execute_task(
                task=task,
                context=context,
                tools=task.tools
            )

            # 存储当前 task 的输出
            task.output = task_output
            results.append(task_output)

            if crew.verbose:
                self._log_step(i, task, agent, task_output)

        return results

    def _build_context(self, previous_results: List[TaskOutput]) -> str:
        """将前序任务的输出汇聚成上下文字符串"""
        if not previous_results:
            return ""

        context_parts = []
        for i, result in enumerate(previous_results):
            context_parts.append(f"=== Task {i+1} Output ===\n{result.raw}")

        return "\n\n".join(context_parts)
```

### 特点

| 特性 | 说明 |
|------|------|
| **简单性** | 线性逻辑，容易理解和调试 |
| **可预测性** | 执行顺序确定，输出可预期 |
| **信息流自然** | 前序任务输出自动成为后序任务上下文 |
| **无 Agent 间通信** | Agent 之间不直接交互，全部通过 Task output 传递 |
| **串行瓶颈** | 总执行时间 = 所有 Task 执行时间之和 |
| **上下文膨胀** | 链越长，后续任务的上下文越大，Token 消耗线性增长 |

---

## 5. Hierarchical Process（层级流程）

Hierarchical Process 引入了一个 **Manager Agent** 负责协调。Manager Agent 不直接执行任务，而是扮演"项目经理"角色：接收所有 Task 和 Agent 信息，自主决定如何分配任务、监控执行、评估结果。

### 流程图

```
                        Hierarchical Process
    =============================================================

    +----------------------------------------------------------+
    |                  Manager Agent                            |
    |  role: "Crew Manager"                                     |
    |  goal: "Orchestrate tasks among agents"                   |
    |  backstory: "You are an expert project manager..."        |
    |  llm: manager_llm (可独立配置，通常用更强模型)              |
    |                                                           |
    |  职责:                                                    |
    |  1. 接收 Crew 级别的目标描述                                |
    |  2. 将大任务分解为子任务                                    |
    |  3. 选择合适的 Agent 执行每个子任务                         |
    |  4. 评估子任务输出，决定是否重做或继续                       |
    |  5. 在 token 预算内管理执行                                 |
    +----------------------------------------------------------+
                │                  │                │
       ┌────────┘    ┌────────────┘    ┌───────────┘
       v             v                  v
    ┌────────┐  ┌────────┐       ┌────────┐
    │ Agent A │  │ Agent B │  ... │ Agent N │
    │ 执行子任务│  │ 执行子任务│      │ 执行子任务│
    └────┬───┘  └────┬───┘       └────┬───┘
         │           │                │
         └───────┬───┘────────────────┘
                 │ (结果汇报给 Manager)
                 v
    +----------------------------------------------------------+
    |                  Manager Agent (评估阶段)                   |
    |                                                           |
    |  for each sub_task result:                                |
    |      if result meets quality threshold:                   |
    |          标记完成，进入下一子任务                            |
    |      else:                                                |
    |          重新分配（可能选择不同 Agent 或修改指令）            |
    |                                                           |
    |  当所有子任务完成:                                          |
    |      汇总结果，生成最终输出                                  |
    +----------------------------------------------------------+
```

### 执行细节

```
Step 1: Manager 接收 Crew 目标
        ─────────────────────────────────────
        Manager 接收整个 Crew 的 task 列表和 agent 列表
        Prompt:
            "You are managing a crew of agents.
             Available agents: [Agent A (researcher), Agent B (writer), ...]
             Goals: [Task 1 description, Task 2 description, ...]
             
             Your job is to:
             1. Decompose the work into subtasks
             2. Assign each subtask to the right agent
             3. Review results and decide next steps
             4. Continue until all objectives are met"

Step 2: Manager 委派子任务
        ─────────────────────────────────────
        Manager 选择一个 Agent 并发送指令:
            "Agent A, please perform the following task:
             [sub_task description]
             
             Use your available tools if needed.
             Report back with your findings."
             
        这通过 Agent 的 execute_task 方法实现，
        Agent 收到 prompt 后像正常任务一样执行。

Step 3: Agent 执行并汇报
        ─────────────────────────────────────
        Agent A 执行任务，返回结果给 Manager。
        Manager 收到结果后评估:
            - 如果质量合格 → 标记完成，进入下一步
            - 如果质量不合格 → 重新委派（可能给不同的 Agent）

Step 4: 迭代直到所有任务完成
        ─────────────────────────────────────
        Manager 持续委派、检查、评估，直到所有目标达成。
        最终 Manager 汇总所有结果。

Step 5: 返回最终结果
        ─────────────────────────────────────
        Manager 将汇总结果作为 Crew 的最终输出。
```

### 关键设计点

Manager Agent 的 prompt 中包含了所有可用 Agent 的信息和所有需要完成的任务。Manager 通过 LLM 的推理能力自主决策如何协调。这意味着：

1. **Manager 需要强模型**：协调决策需要较强的推理能力，通常建议使用 GPT-4、Claude 3.5+ 等高端模型。
2. **Token 消耗高**：Manager 的 prompt 包含所有 Agent 描述和任务描述，且每次委派都需要 LLM 调用。
3. **灵活性高**：Manager 可以自主调整策略，根据中间结果动态重规划。
4. **不可预测性**：不同次运行可能产生不同的协调策略，调试困难。

### 源码级伪代码

```python
class HierarchicalProcess(Process):
    def execute(
        self,
        tasks: List[Task],
        agents: List[Agent],
        crew: Crew,
        manager_llm: BaseLLM
    ) -> List[TaskOutput]:
        # 创建 Manager Agent
        manager = Agent(
            role="Crew Manager",
            goal=f"Manage execution of {len(tasks)} tasks using {len(agents)} agents",
            backstory="You are an expert project manager...",
            llm=manager_llm,
            allow_delegation=True  # 关键：允许委托
        )

        # 构建 Manager 的初始 prompt
        crew_goal = self._build_crew_goal(tasks, agents)

        # Manager 自主协调执行
        # 内部通过 Agent._handle_task_with_delegation() 实现
        final_result = manager.execute_task(
            task=Task(
                description=crew_goal,
                expected_output="All objectives completed successfully"
            ),
            context="",
            tools=[]
        )

        # 解析 Manager 的输出为结构化的 TaskOutput
        results = self._parse_manager_output(final_result, tasks, agents)

        return results

    def _build_crew_goal(self, tasks: List[Task], agents: List[Agent]) -> str:
        goal_parts = ["You are coordinating a team to complete the following tasks:\n"]
        for i, task in enumerate(tasks):
            goal_parts.append(
                f"Task {i+1}: {task.description}\n"
                f"  Expected output: {task.expected_output}\n"
                f"  Assigned to: {task.agent.role}\n"
            )

        goal_parts.append("\nAvailable agents:")
        for agent in agents:
            goal_parts.append(
                f"- {agent.role}: {agent.goal}\n"
                f"  Tools: {[t.name for t in agent.tools]}\n"
            )

        return "\n".join(goal_parts)
```

### Sequential vs Hierarchical 对比

| 维度 | Sequential | Hierarchical |
|------|-----------|--------------|
| **控制方式** | 静态编排（代码定义顺序） | 动态编排（Manager 自主决策） |
| **Agent 通信** | 通过 Task output 间接通信 | Manager 作为中枢协调 |
| **信息流向** | 单向流水线 | 星型拓扑（Manager 为中心） |
| **所需 LLM** | 普通模型即可 | Manager 需要强模型 |
| **Token 消耗** | 随任务数线性增长 | 额外 Manager 调用开销 |
| **灵活性** | 低（固定顺序） | 高（动态分配） |
| **可调试性** | 高（确定性执行） | 低（每次运行策略可能不同） |
| **适用场景** | 明确顺序的 pipeline | 需要动态规划的复杂场景 |
| **并行度** | 严格串行 | Manager 可并行委派 |

---

## 6. Agent 内部机制

Agent 是 CrewAI 的核心执行单元。每个 Agent 封装了一个 LLM 实例、一组工具和角色配置。执行任务时，Agent 经历六个阶段：

```
                    Agent 内部执行循环
    ========================================================

    输入: task (Task 对象), context (str), tools (List[Tool])
    输出: TaskOutput
    
    ┌─────────────────────────────────────────────────────────┐
    │  Phase 1: 构建系统 Prompt                                │
    │                                                         │
    │  system_prompt = f"""                                   │
    │  You are {agent.role}.                                  │
    │  Your goal: {agent.goal}                                │
    │  Your backstory: {agent.backstory}                      │
    │                                                         │
    │  Your available tools: {tool_descriptions}              │
    │                                                         │
    │  Guidelines:                                            │
    │  - Use tools when appropriate                           │
    │  - If you need to delegate a task, use the               │
    │    delegation tool (if allowed)                         │
    │  - Respond with the expected output format              │
    │  """                                                    │
    └──────────────────────┬──────────────────────────────────┘
                           │
                           v
    ┌─────────────────────────────────────────────────────────┐
    │  Phase 2: 构建用户 Prompt                                │
    │                                                         │
    │  user_prompt = f"""                                     │
    │  ## Task Description                                    │
    │  {task.description}                                     │
    │                                                         │
    │  ## Expected Output                                     │
    │  {task.expected_output}                                 │
    │                                                         │
    │  ## Context from Previous Tasks                         │
    │  {context}                                              │
    │  """                                                    │
    └──────────────────────┬──────────────────────────────────┘
                           │
                           v
    ┌─────────────────────────────────────────────────────────┐
    │  Phase 3: 管理上下文窗口                                  │
    │                                                         │
    │  total_prompt = system_prompt + user_prompt             │
    │  token_count = count_tokens(total_prompt)               │
    │                                                         │
    │  if token_count > model.context_window:                 │
    │      # 策略 1: 截断 context 部分                         │
    │      context = truncate_to_token_limit(context, max)    │
    │      # 策略 2: 压缩 context（摘要）                       │
    │      context = summarize_context(context)               │
    │      # 重新构建 user_prompt                             │
    │  endif                                                  │
    └──────────────────────┬──────────────────────────────────┘
                           │
                           v
    ┌─────────────────────────────────────────────────────────┐
    │  Phase 4: 调用 LLM                                       │
    │                                                         │
    │  messages = [                                            │
    │      {"role": "system", "content": system_prompt},      │
    │      {"role": "user", "content": user_prompt}           │
    │  ]                                                      │
    │                                                         │
    │  response = llm.invoke(messages, tools=tool_defs)       │
    │                                                         │
    │  # 处理流式/非流式输出                                   │
    │  if streaming:                                          │
    │      raw_output = collect_stream(response)              │
    │  else:                                                  │
    │      raw_output = response.content                      │
    └──────────────────────┬──────────────────────────────────┘
                           │
                           v
    ┌─────────────────────────────────────────────────────────┐
    │  Phase 5: 处理工具调用                                    │
    │                                                         │
    │  while response.has_tool_calls:                         │
    │      for tool_call in response.tool_calls:              │
    │          tool_name = tool_call.function.name            │
    │          args = parse_arguments(tool_call.function.args)│
    │          try:                                           │
    │              result = execute_tool(tool_name, args)     │
    │          except Exception as e:                         │
    │              result = f"Tool error: {str(e)}"           │
    │                                                         │
    │          messages.append(tool_call_result)              │
    │                                                         │
    │      response = llm.invoke(messages, tools=tool_defs)   │
    │                                                         │
    │  # 工具调用完成后，LLM 输出最终结果                        │
    │  final_output = response.content                         │
    └──────────────────────┬──────────────────────────────────┘
                           │
                           v
    ┌─────────────────────────────────────────────────────────┐
    │  Phase 6: 后处理与输出                                    │
    │                                                         │
    │  # 解析结构化输出（如果 expected_output 包含格式要求）     │
    │  parsed = parse_output(final_output, task.format)       │
    │                                                         │
    │  # 构建 TaskOutput                                      │
    │  task_output = TaskOutput(                              │
    │      raw=final_output,                                  │
    │      parsed=parsed,                                     │
    │      agent=agent.role,                                  │
    │      task=task.description                              │
    │  )                                                      │
    │                                                         │
    │  # 触发回调                                              │
    │  if task.callback:                                      │
    │      task.callback(task_output)                         │
    │                                                         │
    │  return task_output                                     │
    └─────────────────────────────────────────────────────────┘
```

### 上下文窗口管理（Context Window Management）

CrewAI 在处理长链任务时面临严重的上下文窗口压力。Agent 内部的上下文管理策略包括：

```python
def _manage_context(
    self,
    system_prompt: str,
    user_prompt: str,
    model_max_tokens: int,
    response_reserve: int = 2000  # 为响应预留的 token 数
) -> Tuple[str, str]:
    """
    管理上下文窗口，确保 prompt 不超过模型限制。
    策略优先级:
    1. 截断 context 部分（从 oldest 开始）
    2. 对 context 进行摘要压缩
    3. 截断 system_prompt 中的 backstory
    """
    total_tokens = self._count_tokens(system_prompt + user_prompt)
    max_prompt_tokens = model_max_tokens - response_reserve

    if total_tokens <= max_prompt_tokens:
        return system_prompt, user_prompt

    # 策略 1: 截断 context
    # 从 user_prompt 中分离出 context 部分
    system_part, context_part = self._split_prompt(user_prompt)

    context_tokens = self._count_tokens(context_part)
    overflow = total_tokens - max_prompt_tokens

    if context_tokens > overflow:
        # 截断 context 到 token 限制
        truncated_context = self._truncate_tokens(context_part, context_tokens - overflow)
        user_prompt = system_part + truncated_context
        return system_prompt, user_prompt

    # 策略 2: 对 context 做摘要
    summarized_context = self._summarize(context_part, max_tokens=context_tokens - overflow)
    user_prompt = system_part + summarized_context
    total_tokens = self._count_tokens(system_prompt + user_prompt)

    if total_tokens <= max_prompt_tokens:
        return system_prompt, user_prompt

    # 策略 3: 截断 system_prompt 中的 backstory
    system_prompt = self._shorten_backstory(system_prompt)
    return system_prompt, user_prompt
```

### 工具调用机制

Agent 的工具调用遵循标准的 LLM function calling 模式：

```python
def _handle_tool_calls(
    self,
    messages: List[dict],
    tool_defs: List[dict],
    max_tool_rounds: int = 10
) -> str:
    """
    处理工具调用的循环。
    每次 LLM 返回 tool_calls 就执行工具，将结果追加到 messages 中，
    然后继续调用 LLM，直到 LLM 返回纯文本结果或达到最大轮数。
    """
    for round_num in range(max_tool_rounds):
        response = self.llm.invoke(messages, tools=tool_defs)

        if not response.tool_calls:
            return response.content

        for tool_call in response.tool_calls:
            tool_name = tool_call.function.name
            tool_args = json.loads(tool_call.function.arguments)

            try:
                tool_result = self._execute_single_tool(tool_name, tool_args)
                result_content = str(tool_result)
            except Exception as e:
                result_content = f"Error executing {tool_name}: {str(e)}"

            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result_content
            })

    # 达到最大工具调用轮数，要求 LLM 基于已有信息生成最终输出
    messages.append({
        "role": "user",
        "content": "You have reached the maximum number of tool calls. "
                   "Please provide your best answer based on the information gathered so far."
    })
    final_response = self.llm.invoke(messages)
    return final_response.content
```

### 委托（Delegation）机制

当 Agent 设置了 `allow_delegation=True` 时，它可以调用特殊的 `delegate_task` 工具将子任务委托给其他 Agent：

```python
def _setup_delegation_tool(self, all_agents: List[Agent]):
    """为支持委托的 Agent 注入委托工具"""
    if not self.allow_delegation:
        return

    def delegate_task(agent_role: str, task_description: str) -> str:
        """将任务委托给指定角色的 Agent"""
        target_agent = next(
            (a for a in all_agents if a.role == agent_role),
            None
        )
        if not target_agent:
            return f"Error: No agent found with role '{agent_role}'"

        # 目标 Agent 执行任务
        result = target_agent.execute_task(
            task=Task(
                description=task_description,
                agent=target_agent
            ),
            context=""
        )
        return result.raw

    self.tools.append(
        Tool(
            name="delegate_task",
            func=delegate_task,
            description="Delegate a task to another agent by specifying their role"
        )
    )
```

---

## 7. Task 执行细节

Task 是工作单元的定义，其生命周期跨越多个阶段：

### Task 生命周期

```
                  Task 生命周期
    =====================================================

    ┌──────────┐      assign_agent()      ┌────────────┐
    │  CREATED  │ ───────────────────────▶ │  ASSIGNED   │
    │ task 定义  │     绑定执行 Agent       │ 已分配 Agent │
    │ 完成创建   │                          │ 准备就绪     │
    └──────────┘                           └──────┬─────┘
                                                   │
                                          execute(context)
                                                   │
                                                   v
                                          ┌────────────┐
                                          │ EXECUTING   │
                                          │ Agent 正在   │
                                          │ 执行任务     │
                                          └──────┬─────┘
                                                   │
                                          LLM 返回结果
                                                   │
                                                   v
                                          ┌────────────┐
                                          │  COMPLETED  │
                                          │ 任务完成     │
                                          │ output 可用  │
                                          └──────┬─────┘
                                                   │
                                          trigger callback
                                                   │
                                                   v
                                          ┌────────────┐
                                          │  OUTPUT     │
                                          │ 触发回调     │
                                          │ 返回结果     │
                                          └────────────┘
```

### Task 类的核心结构

```python
@dataclass
class Task:
    # --- 核心字段 ---
    description: str                    # 任务描述（对 Agent 的指令）
    expected_output: str                # 期望输出描述
    agent: Optional[Agent]              # 执行此任务的 Agent

    # --- 可选字段 ---
    tools: List[Tool] = field(default_factory=list)  # 任务特定工具
    context: Optional[str] = None       # 手动上下文注入
    async_execution: bool = False       # 是否异步执行

    # --- 回调 ---
    callback: Optional[Callable] = None # 完成回调
    step_callback: Optional[Callable] = None  # 步骤回调

    # --- 运行时状态 ---
    output: Optional[TaskOutput] = None # 任务输出（执行后填充）
    status: TaskStatus = TaskStatus.CREATED  # 当前状态

    def execute(
        self,
        context: Optional[str] = None,
        tools: Optional[List[Tool]] = None
    ) -> TaskOutput:
        """执行任务"""
        self.status = TaskStatus.EXECUTING

        # 合并任务级别的工具
        effective_tools = list(self.tools)
        if tools:
            effective_tools.extend(tools)

        # 使用绑定的 Agent 执行
        task_output = self.agent.execute_task(
            task=self,
            context=context or self.context or "",
            tools=effective_tools
        )

        self.output = task_output
        self.status = TaskStatus.COMPLETED

        # 触发回调
        if self.callback:
            self.callback(task_output)

        return task_output

    @property
    def summary(self) -> str:
        """任务摘要，用于日志和调试"""
        return f"Task({self.description[:50]}..., agent={self.agent.role if self.agent else 'None'})"
```

### 上下文注入机制

Task 的 context 可以从两个来源获取：

```
1. 显式 context（task.context）
   ──────────────────────────────
   开发者手动设置:
   task.context = "Previous analysis: market size is $10B"

2. 隐式 context（Process 自动注入）
   ──────────────────────────────
   SequentialProcess 自动将前序 task 的 output 注入:
   task.context = _build_context(previous_results)
   
   HierarchicalProcess 由 Manager 自主决定注入什么:
   manager 发送给 agent 的消息中自然包含了上下文

合并逻辑:
   final_context = task.context or ""
   if process_injected_context:
       final_context += "\n" + process_injected_context
```

### 输出解析

Task 的输出解析支持多种格式：

```python
def _parse_output(
    self,
    raw_output: str,
    expected_output: str
) -> Any:
    """
    根据 expected_output 的提示尝试解析输出。
    支持:
    - JSON 格式: 尝试 json.loads()
    - Markdown 代码块: 提取 ``` ... ``` 内容
    - 纯文本: 直接返回
    """
    # 尝试 JSON 解析
    if "json" in expected_output.lower():
        try:
            json_match = re.search(r'```json\n(.*?)\n```', raw_output, re.DOTALL)
            if json_match:
                return json.loads(json_match.group(1))
            return json.loads(raw_output)
        except json.JSONDecodeError:
            pass  # fall through to raw

    # 尝试提取 markdown 代码块
    code_match = re.search(r'```(.*?)\n(.*?)\n```', raw_output, re.DOTALL)
    if code_match:
        return code_match.group(2)

    return raw_output
```

---

## 8. 关键设计决策

### 8.1 为什么使用同步而非异步？

CrewAI 的核心执行模型是同步的，但支持 `async_execution=True` 的 Task 异步执行。

**设计理由：**

```python
def kickoff(self) -> CrewOutput:
    """
    同步执行是默认选择，原因：
    1. 简化上下文管理：同步链保证了上下文的一致性和确定性
    2. 避免竞态条件：同步执行天然避免了 Agent 间状态冲突
    3. 调试友好：调用栈清晰，出错容易定位
    4. 用户体验：大多数用户期望 kickoff() 是阻塞调用
    """
```

但 CrewAI 也支持在 Task 级别启用异步：

```python
# 异步 Task 会在后台执行，不阻塞其他 Task
task_1 = Task(description="...", async_execution=True)
task_2 = Task(description="...", async_execution=True)
task_3 = Task(description="...", depends_on=[task_1, task_2])

# Crew 内部会等待所有异步 Task 完成后再执行依赖 Task
```

### 8.2 Sequential vs Hierarchical 的选择依据

CrewAI 提供了两种流程，适用不同场景：

```python
def choose_process(workflow_type: str) -> ProcessType:
    """
    Sequential: 适合"流水线"场景
        - 任务有明确的先后顺序
        - 每个任务独立，前序输出即后续输入
        - 典型: 研究 -> 分析 -> 写作 -> 审核

    Hierarchical: 适合"动态规划"场景
        - 任务之间的关系不确定
        - 需要 Manager 自主决策如何分配
        - 典型: 复杂问题求解，需要多个 Agent 协作探索
        - 代价: 更高的 token 消耗和更长的执行时间
    """
    if workflow_type == "pipeline":
        return ProcessType.sequential
    elif workflow_type == "dynamic":
        return ProcessType.hierarchical
    else:
        raise ValueError(f"Unknown workflow type: {workflow_type}")
```

### 8.3 工具共享机制

CrewAI 提供两种工具共享策略：

```
策略 1: Agent 级别工具（默认）
    Agent A: tools = [web_search, calculator]
    Agent B: tools = [code_interpreter]
    → 每个 Agent 只能使用自己的工具

策略 2: 全局共享工具 (share_tools=True)
    Agent A: tools = [web_search, calculator]
    Agent B: tools = [code_interpreter]
    → 所有 Agent 共享全部工具
    → Agent A 可以使用 code_interpreter
    → Agent B 可以使用 web_search

实现:
    if crew.share_tools:
        all_tools = set()
        for agent in agents:
            all_tools.update(agent.tools)
        for agent in agents:
            agent.tools = list(all_tools)
```

### 8.4 LLM 绑定的灵活性

每个 Agent 可以绑定不同的 LLM，这允许在同一个 Crew 中混合使用不同模型：

```python
crew = Crew(
    agents=[
        Agent(
            role="Researcher",
            llm=LLM(model="gpt-4o"),          # 强模型做研究
            tools=[web_search_tool]
        ),
        Agent(
            role="Writer",
            llm=LLM(model="gpt-4o-mini"),      # 轻量模型做写作
            tools=[]
        ),
        Agent(
            role="Reviewer",
            llm=LLM(model="claude-sonnet-4"),   # 不同供应商
            tools=[]
        )
    ],
    tasks=[...],
    process=ProcessType.sequential
)
```

---

## 9. 局限性与改进

### 9.1 上下文窗口管理在长链中的问题

```
问题:
    Sequential Process 中，每个后续 Task 的上下文包含前面
    所有 Task 的输出。在 10+ 任务的链中，第 10 个 Task 的
    prompt 可能包含前 9 个 Task 的全部输出。

    假设每个 Task 输出 2000 tokens:
        Task 1:  prompt = system(1000) + user(500)
        Task 2:  prompt = system(1000) + user(500 + 2000)
        Task 3:  prompt = system(1000) + user(500 + 4000)
        ...
        Task 10: prompt = system(1000) + user(500 + 18000)
                  = 19500 tokens

    对于 32K 窗口模型，到 Task 16 就会达到限制。
    对于 8K 窗口模型，到 Task 4 就已经饱和。

影响:
    - 后续 Agent 收到的 context 中，早期信息被截断
    - 链越长，Agent 对早期任务的"记忆"越模糊
    - Token 消耗与任务数呈 O(n^2) 增长
```

**改进方向：**

| 方案 | 描述 | 代价 |
|------|------|------|
| Context 摘要 | 每次传递前对前序输出做摘要 | 信息丢失 |
| 选择性传递 | 只传递相关 Task 的输出 | 需要确定"相关性" |
| 外部记忆 | 使用 Vector Store 存储历史 | 增加系统复杂度 |
| 分片 Crew | 将长链拆分为多个短 Crew | 上下文断裂 |

### 9.2 有限的并行执行

CrewAI 的 Sequential Process 是严格串行的，不支持 Task 级并行。虽然有 `async_execution` 标记，但其实现较为简单：

```python
# 当前实现：顺序执行，不支持 DAG 依赖
def _run_sequential_process(self, tasks, agents):
    results = []
    for task in tasks:
        if task.async_execution:
            # 使用 ThreadPoolExecutor 异步执行
            # 但仍然等待结果后才进入下一个 Task
            ...
        else:
            result = task.execute(context)
        results.append(result)
    return results

# 理想实现：支持 DAG 依赖图
# 需要引入 Task 依赖关系和拓扑排序
```

### 9.3 工具错误处理脆弱性

```
问题:
    Agent 调用工具时如果出错，当前实现是将异常信息作为
    工具结果返回给 LLM，期望 LLM 能"自行处理"。

    依赖 LLM 自行处理错误的问题:
    1. LLM 可能不理解技术错误信息
    2. LLM 可能不断重试同样的错误调用
    3. 错误可能导致 Agent 产生幻觉

改进方向:
    - 工具调用的指数退避重试
    - 工具输出 schema 验证
    - 错误分类（可重试 vs 不可重试）
    - 工具超时控制
```

### 9.4 其他已知局限

| 局限 | 描述 | 影响程度 |
|------|------|----------|
| **无持久化状态** | Crew 执行完即销毁，不支持中间状态持久化 | 中 |
| **无流式输出** | kickoff() 是阻塞调用，不支持逐步输出 | 中 |
| **无条件分支** | Sequential Process 不支持条件跳转 | 高 |
| **Agent 数量上限** | Hierarchical Process 中 Agent 太多会撑爆 Manager prompt | 高 |
| **调试工具匮乏** | Agent 间通信链追踪困难 | 中 |
| **无缓存机制** | 相同输入重复执行会重新调用 LLM | 低 |

---

## 10. 代码示例

以下是一个简化但完整的 CrewAI 核心编排重实现，展示了四个核心类的协作关系。

### 完整可运行示例

```python
"""
CrewAI 核心编排机制的简化重实现。

包含:
- Agent: 角色执行单元
- Task: 工作单元定义
- Process: 执行流程基类
- SequentialProcess: 顺序执行
- HierarchicalProcess: 层级执行
- Crew: 团队编排入口

这不是 CrewAI 源码的精确拷贝，而是其核心编排逻辑的教学性重实现。
"""

import json
import time
from dataclasses import dataclass, field
from typing import List, Optional, Callable, Any
from enum import Enum


# ============================================================
# 输出类型定义
# ============================================================

@dataclass
class TaskOutput:
    """单个 Task 的输出"""
    raw: str
    agent_role: str
    task_description: str
    parsed: Any = None
    execution_time: float = 0.0

    def __str__(self) -> str:
        return self.raw


@dataclass
class CrewOutput:
    """Crew 的最终输出"""
    raw: str
    tasks_output: List[TaskOutput]

    def __str__(self) -> str:
        return self.raw

    @property
    def final_task_output(self) -> Optional[TaskOutput]:
        """获取最后一个 Task 的输出"""
        return self.tasks_output[-1] if self.tasks_output else None


# ============================================================
# 工具抽象
# ============================================================

class Tool:
    """Agent 可使用的工具"""
    def __init__(self, name: str, func: Callable, description: str):
        self.name = name
        self.func = func
        self.description = description

    def run(self, **kwargs) -> str:
        try:
            result = self.func(**kwargs)
            return str(result)
        except Exception as e:
            return f"Tool '{self.name}' error: {str(e)}"


# ============================================================
# LLM 抽象
# ============================================================

class BaseLLM:
    """LLM 接口抽象（真实 CrewAI 用 litellm 统一适配）"""
    def __init__(self, model: str, api_key: Optional[str] = None):
        self.model = model

    def invoke(self, messages: List[dict], tools: Optional[List[dict]] = None) -> Any:
        """调用 LLM（真实实现会调用 OpenAI/Anthropic 等 API）"""
        raise NotImplementedError

    def count_tokens(self, text: str) -> int:
        """估算 token 数"""
        return len(text) // 4  # 简化估算


# ============================================================
# Phase 2 & 3: Agent & Task
# ============================================================

class Agent:
    """
    CrewAI Agent 核心实现。
    
    职责:
    - 扮演一个特定角色（role + goal + backstory）
    - 使用 LLM 执行分配的任务
    - 调用工具完成任务
    - 可选：委托任务给其他 Agent
    """

    def __init__(
        self,
        role: str,
        goal: str,
        backstory: str,
        llm: BaseLLM,
        tools: Optional[List[Tool]] = None,
        allow_delegation: bool = False,
        max_retries: int = 3,
        max_tool_rounds: int = 10,
        verbose: bool = False
    ):
        self.role = role
        self.goal = goal
        self.backstory = backstory
        self.llm = llm
        self.tools = tools or []
        self.allow_delegation = allow_delegation
        self.max_retries = max_retries
        self.max_tool_rounds = max_tool_rounds
        self.verbose = verbose

    def execute_task(
        self,
        task: "Task",
        context: str = "",
        tools: Optional[List[Tool]] = None
    ) -> TaskOutput:
        """
        执行任务的完整流程。
        
        步骤:
        1. 构建 system prompt（角色设定）
        2. 构建 user prompt（任务描述 + 上下文）
        3. 管理上下文窗口
        4. 调用 LLM
        5. 处理工具调用循环
        6. 构建并返回 TaskOutput
        """
        start_time = time.time()

        # Step 1 & 2: 构建 prompts
        system_prompt = self._build_system_prompt()
        user_prompt = self._build_user_prompt(task, context)

        if self.verbose:
            print(f"[Agent:{self.role}] Executing task: {task.description[:60]}...")

        # Step 3: 管理上下文窗口
        system_prompt, user_prompt = self._manage_context(
            system_prompt, user_prompt
        )

        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ]

        # Step 4 & 5: LLM 调用 + 工具调用循环
        raw_output = self._invoke_with_tool_loop(messages)

        # Step 6: 构建输出
        execution_time = time.time() - start_time
        task_output = TaskOutput(
            raw=raw_output,
            agent_role=self.role,
            task_description=task.description,
            execution_time=execution_time
        )

        if self.verbose:
            print(f"[Agent:{self.role}] Completed in {execution_time:.2f}s")

        return task_output

    def _build_system_prompt(self) -> str:
        """构建 Agent 的系统提示词"""
        tool_descriptions = "\n".join([
            f"- {tool.name}: {tool.description}"
            for tool in self.tools
        ])

        delegation_note = ""
        if self.allow_delegation:
            delegation_note = (
                "\n- If a task requires specialized expertise, "
                "use the 'delegate_task' tool to assign it to another agent."
            )

        return f"""
You are {self.role}.

Your goal: {self.goal}

Your backstory: {self.backstory}

Your available tools:
{tool_descriptions}{delegation_note}

Guidelines:
- Always use the appropriate tools when needed
- Think step by step
- Provide comprehensive and accurate outputs
- If you encounter errors, try alternative approaches
"""

    def _build_user_prompt(self, task: "Task", context: str) -> str:
        """构建用户提示词"""
        context_section = ""
        if context:
            context_section = f"""
## Context from Previous Tasks
{context}
"""

        return f"""
## Task Description
{task.description}

## Expected Output
{task.expected_output}
{context_section}
"""

    def _manage_context(
        self,
        system_prompt: str,
        user_prompt: str,
        model_max_tokens: int = 128000,
        response_reserve: int = 4000
    ) -> tuple:
        """
        管理上下文窗口。
        如果 prompt 超过模型限制，截断 context 部分。
        """
        total = len(system_prompt + user_prompt) // 4
        max_allowed = model_max_tokens - response_reserve

        if total <= max_allowed:
            return system_prompt, user_prompt

        # 需要截断
        overflow = total - max_allowed
        overflow_chars = overflow * 4

        # 截断 user_prompt 的 context 部分（保留 task description）
        task_marker = "## Task Description"
        if task_marker in user_prompt:
            parts = user_prompt.split(task_marker)
            intro_part = parts[0]
            task_rest = task_marker + parts[1]

            # 截断 intro_part（context 部分）
            if len(intro_part) > overflow_chars:
                intro_part = "..." + intro_part[overflow_chars:]
                user_prompt = intro_part + task_rest
            else:
                user_prompt = task_rest

        return system_prompt, user_prompt

    def _invoke_with_tool_loop(self, messages: List[dict]) -> str:
        """
        LLM 调用 + 工具调用循环。
        
        循环直到:
        - LLM 返回纯文本（没有 tool_calls）
        - 达到 max_tool_rounds
        - 达到 max_retries
        """
        for attempt in range(self.max_retries):
            try:
                # 构建工具定义（OpenAI function calling 格式）
                tool_defs = self._build_tool_definitions()

                return self._tool_loop(messages, tool_defs)

            except Exception as e:
                if self.verbose:
                    print(f"[Agent:{self.role}] Attempt {attempt + 1} failed: {e}")
                if attempt == self.max_retries - 1:
                    return f"Error after {self.max_retries} attempts: {str(e)}"
                continue

        return "Max retries exceeded"

    def _tool_loop(self, messages: List[dict], tool_defs: List[dict]) -> str:
        """工具调用循环"""
        for round_num in range(self.max_tool_rounds):
            response = self.llm.invoke(messages, tools=tool_defs)

            # 纯文本响应 → 完成
            if not response.tool_calls:
                return response.content

            # 处理每个工具调用
            for tool_call in response.tool_calls:
                tool_name = tool_call.function.name

                # 检查是否是委托工具
                if tool_name == "delegate_task" and self.allow_delegation:
                    args = json.loads(tool_call.function.arguments)
                    delegate_result = self._handle_delegation(args)
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": delegate_result
                    })
                    continue

                # 常规工具调用
                tool = next(
                    (t for t in self.tools if t.name == tool_name),
                    None
                )
                if not tool:
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": f"Unknown tool: {tool_name}"
                    })
                    continue

                args = json.loads(tool_call.function.arguments)
                result = tool.run(**args)
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })

        return "Reached maximum tool call rounds"

    def _build_tool_definitions(self) -> List[dict]:
        """构建 OpenAI 格式的 tool definitions"""
        defs = []
        for tool in self.tools:
            defs.append({
                "type": "function",
                "function": {
                    "name": tool.name,
                    "description": tool.description,
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "input": {
                                "type": "string",
                                "description": "Input for the tool"
                            }
                        }
                    }
                }
            })

        if self.allow_delegation:
            defs.append({
                "type": "function",
                "function": {
                    "name": "delegate_task",
                    "description": "Delegate a task to another agent",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "agent_role": {
                                "type": "string",
                                "description": "Role of the agent to delegate to"
                            },
                            "task_description": {
                                "type": "string",
                                "description": "Description of the task to delegate"
                            }
                        }
                    }
                }
            })

        return defs

    def _handle_delegation(self, args: dict) -> str:
        """处理委托请求"""
        return f"Delegation requested to {args.get('agent_role')}: {args.get('task_description')}"


@dataclass
class Task:
    """
    CrewAI Task 核心实现。
    
    工作单元，包含:
    - 描述（对 Agent 的指令）
    - 期望输出（用于 Agent 理解输出格式）
    - 绑定的执行 Agent
    - 可选: 任务级别工具、上下文、回调
    """
    description: str
    expected_output: str
    agent: Agent
    tools: List[Tool] = field(default_factory=list)
    context: Optional[str] = None
    async_execution: bool = False
    callback: Optional[Callable] = None

    def __post_init__(self):
        self.output: Optional[TaskOutput] = None

    def execute(self, context: Optional[str] = None) -> TaskOutput:
        """执行本任务"""
        effective_context = context or self.context or ""
        task_output = self.agent.execute_task(
            task=self,
            context=effective_context,
            tools=self.tools
        )
        self.output = task_output

        if self.callback:
            self.callback(task_output)

        return task_output


# ============================================================
# Phase 5: Process 执行流程
# ============================================================

class ProcessType(Enum):
    """支持的流程类型"""
    sequential = "sequential"
    hierarchical = "hierarchical"


class Process:
    """流程基类"""
    def execute(
        self,
        tasks: List[Task],
        agents: List[Agent]
    ) -> List[TaskOutput]:
        raise NotImplementedError


class SequentialProcess(Process):
    """
    顺序执行流程。
    
    任务逐一执行，每个 Task 接收前序所有 Task 的输出作为 context。
    简单、可预测、适合流水线场景。
    """

    def __init__(self, verbose: bool = False):
        self.verbose = verbose

    def execute(
        self,
        tasks: List[Task],
        agents: List[Agent],
        max_context_tokens: int = 100000
    ) -> List[TaskOutput]:
        results = []

        for i, task in enumerate(tasks):
            # 构建 context
            context = self._build_context(results, max_context_tokens)

            if self.verbose:
                print(f"\n{'='*60}")
                print(f"Step {i+1}/{len(tasks)}")
                print(f"Task: {task.description[:80]}...")
                print(f"Agent: {task.agent.role}")
                print(f"{'='*60}")

            # 执行当前任务
            task_output = task.execute(context=context)
            results.append(task_output)

            if self.verbose:
                print(f"Output ({len(task_output.raw)} chars): "
                      f"{task_output.raw[:200]}...")

        return results

    def _build_context(
        self,
        previous_results: List[TaskOutput],
        max_tokens: int
    ) -> str:
        """构建上下文，限制 token 数量"""
        if not previous_results:
            return ""

        context_parts = []
        total_chars = 0

        for i, result in enumerate(previous_results):
            entry = f"=== Task {i+1} ({result.agent_role}) Output ===\n{result.raw}"
            total_chars += len(entry)
            context_parts.append(entry)

        # 检查是否超限
        estimated_tokens = total_chars // 4
        if estimated_tokens <= max_tokens:
            return "\n\n".join(context_parts)

        # 超限时：每个 entry 截断到平均分配
        avg_chars_per_entry = (max_tokens * 4) // len(context_parts)
        truncated_parts = []
        for i, part in enumerate(context_parts):
            if len(part) > avg_chars_per_entry:
                part = part[:avg_chars_per_entry] + "\n...[truncated]"
            truncated_parts.append(part)

        return "\n\n".join(truncated_parts)


class HierarchicalProcess(Process):
    """
    层级执行流程。
    
    Manager Agent 负责协调所有任务：
    - 分析任务列表和可用 Agent
    - 将大任务分解为子任务
    - 选择合适的 Agent 执行
    - 评估结果，必要时重做
    - 汇总最终输出
    """

    def __init__(self, manager_llm: BaseLLM, verbose: bool = False):
        self.manager_llm = manager_llm
        self.verbose = verbose

    def execute(
        self,
        tasks: List[Task],
        agents: List[Agent]
    ) -> List[TaskOutput]:
        # 创建 Manager Agent
        manager = Agent(
            role="Crew Manager",
            goal="Orchestrate tasks among specialized agents",
            backstory=(
                "You are an expert project manager with deep experience "
                "coordinating multi-agent teams. You break down complex "
                "objectives, assign tasks to the right specialists, "
                "review results, and ensure quality."
            ),
            llm=self.manager_llm,
            allow_delegation=True,
            verbose=self.verbose
        )

        # 构建 Crew 级别的目标描述
        crew_goal = self._build_crew_goal(tasks, agents)

        if self.verbose:
            print(f"\n{'='*60}")
            print("Hierarchical Process Started")
            print(f"Manager LLM: {self.manager_llm.model}")
            print(f"Agents: {[a.role for a in agents]}")
            print(f"Tasks: {len(tasks)}")
            print(f"{'='*60}")
            print(f"Crew Goal:\n{crew_goal[:300]}...")

        # Manager 执行协调（真实 CrewAI 中 Manager 会多次调用 LLM
        # 自主决定如何分配和协调任务，这里简化为一次协调输出）
        manager_result = manager.execute_task(
            task=Task(
                description=crew_goal,
                expected_output="Completed all tasks with final deliverables",
                agent=manager
            ),
            context=""
        )

        # 模拟 Task 级别的输出分配（真实 CrewAI 中 Manager
        # 会通过 delegation tool 实际调用各个 Agent）
        results = self._simulate_agent_executions(tasks, agents, manager_result)

        if self.verbose:
            print(f"\n{'='*60}")
            print("Hierarchical Process Completed")
            print(f"Tasks completed: {len(results)}")
            print(f"{'='*60}")

        return results

    def _build_crew_goal(self, tasks: List[Task], agents: List[Agent]) -> str:
        """构建 Crew 级别的目标描述"""
        parts = [
            "You are managing a team of specialized agents. "
            "Complete all of the following tasks by delegating to the appropriate agents.",
            "",
            "## Tasks to Complete",
        ]

        for i, task in enumerate(tasks):
            parts.append(f"Task {i+1}: {task.description}")
            parts.append(f"  Expected output: {task.expected_output}")
            parts.append(f"  Recommended agent: {task.agent.role}")
            parts.append("")

        parts.append("## Available Agents")
        for agent in agents:
            tool_names = [t.name for t in agent.tools]
            parts.append(f"- {agent.role}: {agent.goal}")
            parts.append(f"  Tools: {tool_names}")
            parts.append("")

        parts.append(
            "Instructions:\n"
            "1. Review all tasks and available agents\n"
            "2. For each task, delegate to the most suitable agent\n"
            "3. Review the results from each agent\n"
            "4. If a result is insufficient, reassign or provide additional guidance\n"
            "5. Provide a final summary of all completed work"
        )

        return "\n".join(parts)

    def _simulate_agent_executions(
        self,
        tasks: List[Task],
        agents: List[Agent],
        manager_result: TaskOutput
    ) -> List[TaskOutput]:
        """
        模拟 Manager 协调下的 Agent 执行。
        真实 CrewAI 中，Manager 会通过 delegation tool
        实际调用各个 Agent 的 execute_task 方法。
        """
        results = []

        for i, task in enumerate(tasks):
            # 收集前序结果作为 context
            context = self._build_context_for_task(results)

            # 让绑定的 Agent 执行任务
            task_output = task.execute(context=context)
            results.append(task_output)

        return results

    def _build_context_for_task(
        self,
        previous_results: List[TaskOutput]
    ) -> str:
        """为当前 Task 构建 context"""
        if not previous_results:
            return ""

        context_parts = []
        for i, result in enumerate(previous_results):
            context_parts.append(
                f"### Previous Task {i+1} (by {result.agent_role})\n"
                f"{result.raw}"
            )
        return "\n\n".join(context_parts)


# ============================================================
# Phase 1 & 4: Crew 编排入口
# ============================================================

class Crew:
    """
    CrewAI Crew 核心实现。
    
    编排入口，负责:
    1. 校验配置
    2. 创建 Agent 和 Task 实例
    3. 根据 Process 类型执行
    4. 收集和返回结果
    """

    def __init__(
        self,
        agents: List[Agent],
        tasks: List[Task],
        process: ProcessType = ProcessType.sequential,
        manager_llm: Optional[BaseLLM] = None,
        share_tools: bool = False,
        verbose: bool = False
    ):
        self.agents = agents
        self.tasks = tasks
        self.process_type = process_type
        self.manager_llm = manager_llm
        self.share_tools = share_tools
        self.verbose = verbose

    def kickoff(self) -> CrewOutput:
        """
        Crew 执行的唯一入口。
        
        五阶段流程:
        Phase 1: 校验配置
        Phase 2: 创建 Agent 实例
        Phase 3: 创建 Task 实例
        Phase 4: 执行 Process
        Phase 5: 收集结果
        """
        # Phase 1: 校验配置
        self._validate_config()

        # Phase 2: 创建 Agent 实例（绑定 LLM，注入工具）
        agents = self._setup_agents()

        # Phase 3: 创建 Task 实例（绑定 Agent）
        tasks = self._setup_tasks(agents)

        # Phase 4: 执行 Process
        process = self._create_process()
        results = process.execute(tasks, agents)

        # Phase 5: 收集结果
        return self._build_output(results)

    def kickoff_for_each(self, inputs: List[dict]) -> List[CrewOutput]:
        """
        对每个输入执行一次 kickoff。
        适用于需要对多个独立输入分别执行的场景。
        """
        outputs = []
        for input_data in inputs:
            # 用输入数据更新 task 上下文
            for task in self.tasks:
                task.context = json.dumps(input_data)
            output = self.kickoff()
            outputs.append(output)
        return outputs

    def _validate_config(self):
        """校验配置"""
        assert len(self.agents) > 0, "At least one agent is required"
        assert len(self.tasks) > 0, "At least one task is required"

        if self.process_type == ProcessType.hierarchical:
            assert self.manager_llm is not None, (
                "manager_llm is required for hierarchical process"
            )

        # 验证每个 task 的 agent 都在 crew 的 agents 列表中
        for task in self.tasks:
            assert task.agent in self.agents, (
                f"Agent '{task.agent.role}' assigned to task "
                f"but not in crew's agents list"
            )

    def _setup_agents(self) -> List[Agent]:
        """
        设置 Agent 实例:
        - 收集所有工具（如果 share_tools=True）
        - 确保每个 Agent 都有 LLM 绑定
        """
        if self.share_tools:
            all_tools = set()
            for agent in self.agents:
                all_tools.update(agent.tools)
            for agent in self.agents:
                agent.tools = list(all_tools)

        return self.agents

    def _setup_tasks(self, agents: List[Agent]) -> List[Task]:
        """
        设置 Task 实例:
        - 将 agent 配置绑定到实际的 agent 实例
        """
        return self.tasks

    def _create_process(self) -> Process:
        """根据 process_type 创建对应的 Process 实例"""
        if self.process_type == ProcessType.sequential:
            return SequentialProcess(verbose=self.verbose)
        elif self.process_type == ProcessType.hierarchical:
            return HierarchicalProcess(
                manager_llm=self.manager_llm,
                verbose=self.verbose
            )
        else:
            raise ValueError(f"Unknown process type: {self.process_type}")

    def _build_output(self, results: List[TaskOutput]) -> CrewOutput:
        """构建最终输出"""
        raw_parts = []
        for i, result in enumerate(results):
            raw_parts.append(
                f"## Task {i+1} ({result.agent_role})\n{result.raw}"
            )

        return CrewOutput(
            raw="\n\n".join(raw_parts),
            tasks_output=results
        )


# ============================================================
# 使用示例
# ============================================================

def example_sequential_crew():
    """Sequential Process 示例：内容创作 Pipeline"""

    # 定义工具
    def search_web(query: str) -> str:
        return f"Search results for '{query}': [mock data]..."

    def write_file(content: str) -> str:
        return f"File written: {len(content)} chars"

    search_tool = Tool(
        name="search_web",
        func=search_web,
        description="Search the web for information"
    )

    file_tool = Tool(
        name="write_file",
        func=write_file,
        description="Write content to a file"
    )

    # 创建 Agent
    researcher = Agent(
        role="Research Analyst",
        goal="Gather comprehensive information on given topics",
        backstory="Expert researcher with 10 years of experience...",
        llm=BaseLLM(model="gpt-4o"),
        tools=[search_tool]
    )

    writer = Agent(
        role="Content Writer",
        goal="Create engaging content based on research",
        backstory="Professional writer specializing in technical content...",
        llm=BaseLLM(model="gpt-4o-mini"),
        tools=[file_tool]
    )

    reviewer = Agent(
        role="Quality Reviewer",
        goal="Ensure content accuracy and quality",
        backstory="Detail-oriented editor with domain expertise...",
        llm=BaseLLM(model="gpt-4o-mini"),
        tools=[]
    )

    # 创建 Task
    research_task = Task(
        description="Research the latest trends in AI agent frameworks",
        expected_output="A comprehensive research report with key findings",
        agent=researcher
    )

    writing_task = Task(
        description="Write a blog post based on the research",
        expected_output="A well-structured blog post (1000 words)",
        agent=writer
    )

    review_task = Task(
        description="Review the blog post for accuracy and quality",
        expected_output="Review feedback with suggested improvements",
        agent=reviewer,
        callback=lambda output: print(f"Review completed: {len(output.raw)} chars")
    )

    # 创建 Crew
    crew = Crew(
        agents=[researcher, writer, reviewer],
        tasks=[research_task, writing_task, review_task],
        process=ProcessType.sequential,
        verbose=True
    )

    # 执行
    print("Starting Crew execution...")
    result = crew.kickoff()
    print(f"\nFinal output:\n{result}")

    return result


def example_hierarchical_crew():
    """Hierarchical Process 示例：复杂问题求解"""

    researcher = Agent(
        role="Senior Researcher",
        goal="Find and analyze relevant information",
        backstory="Experienced researcher with broad knowledge...",
        llm=BaseLLM(model="gpt-4o"),
        tools=[Tool(name="search", func=lambda q: f"results for {q}", description="Search tool")]
    )

    analyst = Agent(
        role="Data Analyst",
        goal="Analyze data and identify patterns",
        backstory="Expert analyst specializing in data interpretation...",
        llm=BaseLLM(model="gpt-4o"),
        tools=[Tool(name="analyze", func=lambda d: f"analysis of {d}", description="Analysis tool")]
    )

    task_1 = Task(
        description="Research market trends for 2025-2026",
        expected_output="Market trends report",
        agent=researcher
    )

    task_2 = Task(
        description="Analyze competitors in the AI space",
        expected_output="Competitive analysis report",
        agent=analyst
    )

    # Hierarchical Process 需要 manager_llm（通常更强）
    crew = Crew(
        agents=[researcher, analyst],
        tasks=[task_1, task_2],
        process=ProcessType.hierarchical,
        manager_llm=BaseLLM(model="gpt-4o"),  # Manager 用强模型
        verbose=True
    )

    result = crew.kickoff()
    return result


if __name__ == "__main__":
    print("=" * 60)
    print("CrewAI Orchestration Demo - Sequential Process")
    print("=" * 60)
    example_sequential_crew()

    print("\n" + "=" * 60)
    print("CrewAI Orchestration Demo - Hierarchical Process")
    print("=" * 60)
    example_hierarchical_crew()
```

---

## 总结

CrewAI 的核心编排引擎通过四个关键抽象（Agent、Task、Crew、Process）实现了声明式的多 Agent 协作框架。其设计哲学亮点包括：

1. **角色扮演驱动**：Agent 的 role/goal/backstory 三角配置为 LLM 提供了稳定的角色身份，使 Agent 行为可预测。
2. **流程与逻辑分离**：Process 独立于 Agent 和 Task，允许在不修改业务逻辑的情况下切换执行策略。
3. **Pipeline 友好**：Sequential Process 天然适合内容生产类的流水线作业。
4. **扩展性设计**：通过 Tool 抽象和 LLM 接口层，容易接入新工具和新模型。

核心局限也源于其设计取舍：
- 固定流程限制了复杂编排
- 链式上下文传递导致长链 Token 爆炸
- Manager 协调在 Hierarchical 模式下成为瓶颈

理解这些设计决策，有助于在 CrewAI、LangGraph、AutoGen 等框架中做出正确的技术选型。
