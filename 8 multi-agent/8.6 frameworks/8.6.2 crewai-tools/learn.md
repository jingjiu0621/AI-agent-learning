# 8.6.2 CrewAI 工具与任务协作源码分析

> CrewAI Tool Integration and Task Collaboration Source Code Analysis

---

## 1. 简单介绍

CrewAI 是一个轻量级多智能体编排框架，其核心设计理念是让多个 LLM-based Agent 通过**工具（Tools）**与外部世界交互，并通过**任务（Tasks）**组织协作流程。本文件深入 CrewAI 源码层面，剖析其工具系统架构、任务-工具绑定机制、上下文传播链路以及关键实现模式。

CrewAI 的工具体系围绕三个核心问题设计：

- **工具如何定义**：通过 BaseTool 抽象类和 `@tool` 装饰器提供两层抽象，兼顾类型安全与快速原型。
- **工具如何被 Agent 调用**：Agent 持有工具列表，工具描述被注入系统提示（System Prompt），LLM 通过 ReAct 模式（Thought/Action/Action Input）自主选择并调用工具。
- **任务如何协作**：任务通过 `context` 参数声明依赖关系，前序任务的输出自动注入后续任务的提示上下文，形成数据流管道。

---

## 2. 工具系统架构

### 2.1 核心类层次

```
┌─────────────────────────────────────────────────────────────────┐
│                    BaseTool (Abstract Class)                      │
├─────────────────────────────────────────────────────────────────┤
│  + name: str                                                     │
│  + description: str                                              │
│  + args_schema: Type[BaseModel] | None                           │
│  + cache_function: Callable | None                               │
│  + result_as_answer: bool = False                                │
│  + _cache: CacheHandler (via Crew)                               │
│  + _run(*args, **kwargs) -> str          [ABSTRACT]              │
│  + _generate_description() -> str                                │
│  + validate_input(params: dict) -> dict                          │
│  + run(params: dict) -> str              [PUBLIC ENTRY]          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
   Built-in Tools    LangChain Tools   Custom Tools
   (SerperDevTool,   (via Bridge)     (@tool decorator
    ScrapeWebsite,                    or subclassing)
    FileReadTool)
```

### 2.2 BaseTool 核心实现解析

BaseTool 继承自 Pydantic 的 `BaseModel`，利用 Pydantic 的字段声明能力完成工具元数据定义：

```python
class BaseTool(BaseModel):
    # ── 工具标识 ──
    name: str                                            # 工具名称，LLM 通过此名称引用工具
    description: str                                     # 工具描述，注入 prompt 供 LLM 理解用途
    args_schema: Optional[Type[BaseModel]] = None        # Pydantic 输入校验模型
    cache_function: Optional[Callable] = None             # 自定义缓存判断函数
    result_as_answer: bool = False                       # 是否直接作为最终答案返回

    # ── 内部字段（PrivateAttr 不会被 Pydantic 序列化） ──
    _cache: CacheHandler = PrivateAttr(default_factory=CacheHandler)

    # ── 工具描述生成 ──
    def _generate_description(self) -> str:
        """生成注入到 Agent prompt 中的工具描述字符串。"""
        schema = self._get_schema()
        return json.dumps({
            "name": self.name,
            "description": self.description,
            "parameters": schema
        })

    # ── 参数校验 ──
    def validate_input(self, params: dict) -> dict:
        """如果定义了 args_schema，用 Pydantic 模型校验输入参数。"""
        if self.args_schema:
            return self.args_schema(**params).model_dump()
        return params

    # ── 公共执行入口 ──
    def run(self, params: dict) -> str:
        """公开的执行入口，包含缓存逻辑。"""
        # 1. 校验输入
        validated = self.validate_input(params)

        # 2. 检查缓存
        cache_key = f"{self.name}-{json.dumps(validated, sort_keys=True)}"
        cached = self._cache.read(self.name, cache_key)
        if cached is not None and self._should_cache(validated, cached):
            return cached

        # 3. 执行子类实现的 _run
        result = self._run(**validated)

        # 4. 写入缓存
        if self._should_cache(validated, result):
            self._cache.add(self.name, cache_key, result)

        return result

    # ── 子类必须实现 ──
    @abstractmethod
    def _run(self, **kwargs) -> str:
        """工具的核心逻辑，由子类实现。"""
        ...
```

### 2.3 工具描述注入 Agent Prompt 流程

Agent 在构建 system prompt 时，将工具列表序列化为 LLM 可读的格式，注入到 ReAct 指令模板中：

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent System Prompt                        │
├─────────────────────────────────────────────────────────────┤
│  You are {role}. {goal} {backstory}                          │
│                                                              │
│  [TOOL LIST]                                                 │
│  You have access to the following tools:                     │
│                                                              │
│  Tool Name: search_tool                                      │
│  Description: Search the internet for information            │
│  Parameters:                                                 │
│    - query: string (required) - The search query             │
│                                                              │
│  Tool Name: scrape_website                                   │
│  Description: Scrape and extract content from a URL          │
│  Parameters:                                                 │
│    - url: string (required) - The website URL                │
│                                                              │
│  [REACT INSTRUCTIONS]                                        │
│  Use the following format:                                   │
│  Thought: you should always think about what to do           │
│  Action: the action to take [tool_names]                     │
│  Action Input: the input to the action, as JSON              │
│  ...                                                         │
└─────────────────────────────────────────────────────────────┘
```

关键源码位置：Agent 在构建提示时调用 `_get_tools_description()` 方法，遍历 `self.tools` 列表，对每个工具调用 `_generate_description()`，然后拼接成上述格式。CrewAI 的 i18n 系统通过 `I18N_DEFAULT.tools("tools")` 模板控制最终格式。

### 2.4 工具验证机制

CrewAI 使用 Pydantic v2 的模型校验作为工具输入验证层：

```python
# 内部验证链路
def _get_schema(self) -> dict:
    """获取 JSON Schema 用于 prompt 注入和 LLM 参考。"""
    if self.args_schema:
        return self.args_schema.model_json_schema()
    # 如果没有显式定义 args_schema，从 _run 签名推断
    return self._infer_schema_from_signature()

def _infer_schema_from_signature(self) -> dict:
    """从 _run 方法签名自动生成 JSON Schema。"""
    sig = inspect.signature(self._run)
    properties = {}
    for name, param in sig.parameters.items():
        if name == 'self':
            continue
        properties[name] = {
            "type": self._type_to_json_type(param.annotation),
            "description": name
        }
    return {
        "type": "object",
        "properties": properties,
        "required": [p for p, param in sig.parameters.items()
                     if p != 'self' and param.default is inspect.Parameter.empty]
    }
```

**验证流程：**

1. Agent 的 LLM 输出 Action Input（JSON 字符串）
2. Agent 解析 JSON 为 dict
3. 调用 `tool.run(params)` -> `tool.validate_input(params)`
4. 如果定义了 `args_schema`，Pydantic 进行类型校验 + 默认值填充
5. 校验通过后调用 `_run(**validated_params)`
6. 校验失败抛出 `ValidationError`，Agent 捕获后重试

---

## 3. 工具类型

### 3.1 内置工具（crewai-tools 包）

| 工具类 | 功能 | 关键依赖 |
|--------|------|----------|
| `SerperDevTool` | Google 搜索 API | serper API key |
| `ScrapeWebsiteTool` | 网页内容抓取 | 无认证要求 |
| `FileReadTool` | 读取本地文件 | 文件系统访问 |
| `DirectoryReadTool` | 列出目录结构 | 文件系统访问 |
| `DirectorySearchTool` | 目录内全文搜索 | 文件系统访问 |
| `CodeDocsSearchTool` | 代码文档搜索 | 可配置文档源 |
| `PDFSearchTool` | PDF 内容搜索 | RAG 管道 |
| `TXTSearchTool` | 文本文件搜索 | RAG 管道 |
| `JSONSearchTool` | JSON 文件搜索 | RAG 管道 |
| `MDXSearchTool` | MDX 文档搜索 | RAG 管道 |
| `YoutubeVideoSearchTool` | YouTube 视频搜索 | YouTube API |
| `YoutubeChannelSearchTool` | YouTube 频道搜索 | YouTube API |
| `MySQLSearchTool` | MySQL 数据库查询 | 数据库连接 |
| `PGSearchTool` | PostgreSQL 查询 | 数据库连接 |
| `BrowserbaseTool` | 浏览器自动化 | Browserbase API |
| `SpiderTool` | 全网爬虫 | Spider API |

### 3.2 LangChain 工具桥接

CrewAI 可以直接包装 LangChain 工具，通过 `LangChainToolAdapter` 适配器：

```python
from crewai.tools import LangChainTool
from langchain_community.tools import WikipediaQueryRun

# LangChain 工具 → CrewAI 工具
langchain_tool = WikipediaQueryRun(...)
crewai_tool = LangChainTool(langchain_tool)
agent = Agent(tools=[crewai_tool])
```

适配器内部将 LangChain 的 `_run` 映射到 CrewAI 的 `_run`，并处理两者的参数规范差异。

### 3.3 自定义工具

#### 方式 A：子类化 BaseTool（类型安全）

```python
from crewai.tools import BaseTool
from pydantic import BaseModel, Field
from typing import Type

class SearchInput(BaseModel):
    query: str = Field(..., description="Search query string")
    max_results: int = Field(default=5, description="Max results to return")

class CustomSearchTool(BaseTool):
    name: str = "custom_search"
    description: str = "Search internal knowledge base"
    args_schema: Type[BaseModel] = SearchInput

    def _run(self, query: str, max_results: int = 5) -> str:
        # 实际搜索逻辑
        return f"Search results for {query} (limit: {max_results})"
```

#### 方式 B：@tool 装饰器（快速原型）

```python
from crewai.tools import tool

@tool("CustomSearch")
def custom_search(query: str, max_results: int = 5) -> str:
    """Search internal knowledge base."""
    return f"Search results for {query} (limit: {max_results})"

# 装饰器自动完成：
# 1. 从函数签名推断 args_schema
# 2. 从 docstring 提取 description
# 3. 用函数名或自定义名称作为工具 name
# 4. 将函数包装为 BaseTool 子类实例
```

`@tool` 装饰器的内部实现大致如下：

```python
TOOL_REGISTRY: dict[str, type[BaseTool]] = {}

def tool(name_or_func: str | Callable) -> Callable | BaseTool:
    """将普通函数转换为 BaseTool 子类的装饰器。"""
    def decorator(func: Callable) -> BaseTool:
        tool_name = name if isinstance(name_or_func, str) else func.__name__
        tool_doc = func.__doc__ or ""

        # 动态创建 BaseTool 子类
        class DynamicTool(BaseTool):
            name: str = tool_name
            description: str = tool_doc

            def _run(self, **kwargs) -> str:
                return func(**kwargs)

        # 注册到全局注册表（主要用于发布到 CrewAI 市场）
        instance = DynamicTool()
        TOOL_REGISTRY[tool_name] = instance
        return instance

    if callable(name_or_func):
        name = None
        return decorator(name_or_func)
    return decorator
```

---

## 4. Task-Tool 绑定机制

### 4.1 绑定关系总览

```
┌──────────────────────────────────────────────────────────────┐
│                        Crew                                   │
│  ┌─────────────┐          ┌─────────────┐                    │
│  │   Agent A    │          │   Agent B    │                   │
│  │  (Researcher)│          │   (Writer)   │                   │
│  │              │          │              │                   │
│  │  tools = [   │          │  tools = [   │                   │
│  │   search_t,  │          │   file_t,    │                   │
│  │   scrape_t   │          │   format_t   │                   │
│  │  ]           │          │  ]           │                   │
│  └──────┬───────┘          └──────┬───────┘                   │
│         │                         │                           │
│         ▼                         ▼                           │
│  ┌─────────────┐          ┌─────────────┐                    │
│  │  Task A      │          │  Task B      │                   │
│  │  agent=AgentA│          │  agent=AgentB│                   │
│  │  context=[]  │ ──context──▶  context=[TaskA]               │
│  │  tools=None  │          │  tools=None  │                   │
│  └─────────────┘          └─────────────┘                    │
│                                                              │
│  关键规则：Task 不直接持有 tools，而是通过 agent 引用获得      │
│  Task 执行时：agent.execute_task(task)                        │
│  → agent 将自己的 tools 注入 prompt                          │
│  → agent 用 LLM + tools 完成任务                             │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 绑定实现机制

Task 定义中的 `tools` 字段是可选的。当 Task 的 `tools` 为 `None` 时，实际执行时使用 Agent 的 `tools`：

```python
class Task(BaseModel):
    description: str
    expected_output: str
    agent: Optional[BaseAgent] = None
    tools: Optional[List[BaseTool]] = None   # 可选，覆盖 agent 的工具
    context: Optional[List[Task]] = None      # 依赖的前序任务
    async_execution: bool = False

    def execute(self, crew_context: dict = None) -> str:
        """执行任务。"""
        # 确定使用的工具集
        # 优先级：task.tools > agent.tools
        active_tools = self.tools or (self.agent.tools if self.agent else [])

        # Agent 实际执行时使用 active_tools
        result = self.agent.execute_task(
            task=self,
            tools=active_tools,       # ← 工具在这里传入
            context=crew_context
        )
        self.output = result
        return result
```

### 4.3 Agent 执行任务时的工具调用循环

Agent 的 `execute_task` 方法是工具调用的核心循环，遵循 ReAct（推理-行动-观察）模式：

```
                    ┌─────────────────────────┐
                    │  Agent.execute_task()    │
                    └──────────┬──────────────┘
                               │
                               ▼
                    ┌─────────────────────────┐
                    │  Build prompt with:      │
                    │   - System prompt (tools) │
                    │   - Task description      │
                    │   - Context (from prev)   │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │  LLM generates response  │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │  Parse response:         │
                    │  Contains Action?        │
                    │  (tool_name + input)     │
                    └──────────┬──────────────┘
                               │
              ┌────────────────┼────────────────┐
              │ YES            │ NO              │
              ▼                ▼                 │
    ┌─────────────────┐  ┌──────────────┐       │
    │ Find tool by    │  │ Return final  │       │
    │ name in active  │  │ answer text   │       │
    │ tools list      │  └──────────────┘       │
    └────────┬────────┘                          │
             │                                   │
             ▼                                   │
    ┌─────────────────┐                          │
    │ Validate input   │                          │
    │ via args_schema  │                          │
    └────────┬────────┘                          │
             │                                   │
             ▼                                   │
    ┌─────────────────┐                          │
    │ Execute _run()   │                          │
    │ → Get result     │                          │
    └────────┬────────┘                          │
             │                                   │
             ▼                                   │
    ┌─────────────────┐                          │
    │ Append result to │─────────────────────────┘
    │ conversation     │
    │ (Observation)    │
    └─────────────────┘
             │
             ▼
    (back to LLM generation loop)
```

### 4.4 工具选择与执行的源码关键流程

```python
class Agent(BaseModel):
    tools: List[BaseTool] = []
    _tool_mapping: Dict[str, BaseTool] = {}   # name → tool 的查找表

    def execute_task(self, task: Task, tools: List[BaseTool],
                     context: dict = None) -> str:
        # 1. 构建工具查找表
        self._tool_mapping = {tool.name: tool for tool in tools}

        # 2. 构建包含工具描述的 system prompt
        system_prompt = self._build_system_prompt(tools)

        # 3. ReAct 循环
        messages = [{"role": "system", "content": system_prompt}]
        if context:
            messages.append({"role": "user", "content": str(context)})

        messages.append({"role": "user", "content": task.description})

        max_iterations = 15  # 防止无限循环
        for iteration in range(max_iterations):
            response = self.llm.generate(messages)

            # 3a. 解析 LLM 输出
            action = self._parse_action(response)

            if action is None:
                # 没有工具调用 → 返回最终答案
                return response

            # 3b. 执行工具
            tool = self._tool_mapping.get(action.name)
            if tool is None:
                result = f"Error: Unknown tool '{action.name}'"
            else:
                try:
                    result = tool.run(action.input)
                except Exception as e:
                    result = f"Error executing tool: {str(e)}"

            # 3c. 将工具结果作为 Observation 追加
            messages.append({
                "role": "user",
                "content": f"Observation: {result}"
            })

        return "Max iterations reached."

    def _parse_action(self, response: str) -> Action | None:
        """从 LLM 输出解析工具调用。
        匹配 ReAct 格式：
        Action: tool_name
        Action Input: {"key": "value"}
        """
        pattern = r"Action:\s*(\w+)\s*Action Input:\s*(\{.*\})"
        match = re.search(pattern, response, re.DOTALL)
        if match:
            return Action(
                name=match.group(1),
                input=json.loads(match.group(2))
            )
        return None
```

---

## 5. 任务协作流程

### 5.1 Sequential Process（顺序执行）

```
时间线 ──────────────────────────────────────────────────────────────▶

Task A ──▶ Agent A ──▶ [Tool Call Loop] ──▶ Output A
                                                  │
                                                  ▼ (context)
                                             Task B ──▶ Agent B ──▶ [Tool Call Loop] ──▶ Output B
                                                                                               │
                                                                                               ▼ (context)
                                                                                          Task C ──▶ Agent C ──▶ ...
```

源码实现（Crew 类的核心逻辑）：

```python
class Crew(BaseModel):
    agents: List[Agent]
    tasks: List[Task]
    process: Process = Process.sequential

    def kickoff(self, inputs: dict = None) -> str:
        if self.process == Process.sequential:
            return self._run_sequential(inputs)
        elif self.process == Process.hierarchical:
            return self._run_hierarchical(inputs)

    def _run_sequential(self, inputs: dict = None) -> str:
        """顺序执行所有任务，前序输出自动传递为后续上下文。"""
        task_outputs: Dict[str, str] = {}

        for i, task in enumerate(self.tasks):
            # 1. 收集依赖任务的输出作为上下文
            context = {}
            if task.context:
                for dep_task in task.context:
                    if dep_task in task_outputs:
                        context[dep_task.description] = task_outputs[dep_task]

            # 2. 注入全局 inputs
            if inputs:
                context.update(inputs)

            # 3. 获取任务绑定的 agent
            agent = task.agent

            # 4. Agent 执行任务（含工具调用循环）
            result = agent.execute_task(
                task=task,
                tools=task.tools or agent.tools,
                context=context
            )

            # 5. 缓存输出
            task_outputs[task] = result

            # 6. 触发回调
            if self.task_callback:
                self.task_callback(task, result)

        # 最后一个任务的输出作为 Crew 的最终输出
        return task_outputs[self.tasks[-1]]
```

### 5.2 Hierarchical Process（层级执行）

```
┌─────────────────────────────────────────────────────────┐
│                    Manager Agent                          │
│  Role: "Expert Task Manager"                             │
│  Goal: "Manage and delegate tasks to workers"            │
│  Tools: [DelegateWorkTool, AskQuestionTool]              │
│                                                          │
│  Process:                                                │
│  1. Receive full task list                               │
│  2. For each task, decide to:                            │
│     a. Execute directly                                  │
│     b. Delegate to a worker agent via DelegateWorkTool   │
│     c. Ask a question via AskQuestionTool                │
│  3. Aggregate results                                    │
└──────────────────────────┬──────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Worker A    │  │  Worker B    │  │  Worker C    │
│ (Researcher) │  │  (Analyst)   │  │  (Writer)    │
│              │  │              │  │              │
│ tools=[      │  │ tools=[      │  │ tools=[      │
│  search,     │  │  database,   │  │  format,     │
│  scrape      │  │  calc        │  │  style       │
│ ]            │  │ ]            │  │ ]            │
└──────────────┘  └──────────────┘  └──────────────┘
```

Manager Agent 的 `DelegateWorkTool` 关键实现：

```python
class DelegateWorkTool(BaseTool):
    """Manager Agent 用来委派任务的工具。"""
    name: str = "Delegate work to coworker"
    description: str = (
        "Use this tool to delegate a task to one of your coworkers. "
        "The coworker will execute the task and return the result. "
        "Available coworkers: {coworkers}"
    )
    agents: List[BaseAgent] = []

    def _run(self, coworker: str, task: str,
             context: str = "", expected_output: str = "") -> str:
        """将任务委派给指定的 coworker agent。"""
        target_agent = next(a for a in self.agents if a.role == coworker)

        # 创建临时任务
        delegated_task = Task(
            description=task,
            expected_output=expected_output or "Task result",
            agent=target_agent,
            context_input=context
        )

        # 目标 agent 使用自己的工具执行
        result = target_agent.execute_task(
            task=delegated_task,
            tools=target_agent.tools,
            context={"context": context}
        )
        return result
```

层级模式的 Manager Agent 创建：

```python
def _create_manager_agent(self) -> Agent:
    """创建层级模式下的 Manager Agent。"""
    manager = Agent(
        role=i18n.retrieve("hierarchical_manager_agent", "role"),
        goal=i18n.retrieve("hierarchical_manager_agent", "goal"),
        backstory=i18n.retrieve("hierarchical_manager_agent", "backstory"),
        # Manager 默认拥有委托和提问工具
        tools=AgentTools(agents=self.agents).tools(),
        allow_delegation=True,
        llm=self.manager_llm or self._select_best_llm(),
    )
    return manager
```

### 5.3 异步任务执行与等待机制

当任务标记为 `async_execution=True` 时，Crew 使用 `asyncio.gather` 并发执行多个独立任务，然后等待所有前置任务完成再执行依赖任务：

```python
async def _run_sequential_async(self, inputs: dict = None) -> str:
    """异步版本：支持 async_execution=True 的并发执行。"""
    task_outputs: Dict[str, str] = {}
    pending: List[Task] = []
    task_queue = list(self.tasks)

    while task_queue:
        # 找出所有可执行的任务（依赖已满足）
        ready = []
        for task in task_queue:
            deps = task.context or []
            if all(dep in task_outputs for dep in deps):
                ready.append(task)

        if not ready:
            raise RuntimeError("Circular dependency detected.")

        for task in ready:
            task_queue.remove(task)

            if task.async_execution:
                pending.append(self._execute_single_task(task, inputs,
                                                         task_outputs))
            else:
                result = await self._execute_single_task(task, inputs,
                                                         task_outputs)
                task_outputs[task] = result

        # 等待所有异步任务完成
        if pending:
            results = await asyncio.gather(*pending)
            for task, result in zip(
                [t for t in self.tasks if t.async_execution and t not in task_outputs],
                results
            ):
                task_outputs[task] = result
            pending = []

    return task_outputs[self.tasks[-1]]
```

---

## 6. 上下文传递

### 6.1 上下文传播详解

上下文传播是 CrewAI 任务协作的基石，涉及三个层面的数据流动：

```
┌────────────────────────────────────────────────────────────────────┐
│                     上下文传播的三层架构                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  第1层：Crew 级别（kickoff inputs）                                 │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  crew.kickoff(inputs={"topic": "AI", "style": "formal"})   │    │
│  └────────────────────────┬───────────────────────────────────┘    │
│                           │                                        │
│                           ▼                                        │
│  第2层：Task 级别（context 依赖声明）                                │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Task B: context=[Task_A] → Task B 获得 Task_A 的输出       │    │
│  └────────────────────────┬───────────────────────────────────┘    │
│                           │                                        │
│                           ▼                                        │
│  第3层：Agent 级别（prompt 注入）                                   │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Agent._build_context_prompt(context) → 注入到 user message │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 6.2 Sequential 模式的上下文注入

```
Task A Output: "量子计算在药物发现中的应用..."
     │
     ▼
─────────────────────────────────────────────────────────────
Agent B 收到的 Prompt（简化）：

System: You are a Science Writer. Your goal is to write engaging articles...
Tools: [file_tool, format_tool]
...（工具描述和 ReAct 指令）...

User (Context from Task A):
  Here is the context you are working with:
  ┌─────────────────────────────────────┐
  │ Task: Research quantum computing    │
  │ Output: 量子计算在药物发现中的应用...  │
  └─────────────────────────────────────┘

User (Task B description):
  Based on the research, write a 1000-word article...
─────────────────────────────────────────────────────────────
```

### 6.3 上下文窗口管理策略

CrewAI 目前**没有内置的上下文窗口压缩机制**。在长链任务中，上下文会线性累积：

```python
# 实际行为：上下文无限制累积
Task 1 output: 2K tokens
Task 2 output: 2K tokens  +  Task 1 context (2K)  =  4K tokens in prompt
Task 3 output: 2K tokens  +  Task 2 context (4K)  =  6K tokens in prompt
...线性增长...
```

这可能导致后续任务的 LLM 调用超出上下文窗口限制。CrewAI 依赖于底层 LLM 的上下文窗口处理（如 GPT-4 的 128K 窗口），但未主动做摘要或裁剪。

**社区实践（非官方支持）：**

- 在 Task 的 `description` 中要求 Agent 输出摘要而非原始数据
- 使用外部流程（CrewAI Flow）在任务间插入摘要步骤

---

## 7. 关键源码实现（伪代码）

### 7.1 BaseTool 完整实现

```python
import inspect
import json
from abc import abstractmethod
from typing import Any, Callable, Dict, List, Optional, Type

from pydantic import BaseModel, Field, PrivateAttr


class CacheHandler:
    """线程安全的工具结果缓存。"""

    def __init__(self):
        self._cache: Dict[str, str] = {}

    def add(self, tool_name: str, input_key: str, output: str) -> None:
        cache_key = f"{tool_name}-{input_key}"
        self._cache[cache_key] = output

    def read(self, tool_name: str, input_key: str) -> Optional[str]:
        cache_key = f"{tool_name}-{input_key}"
        return self._cache.get(cache_key)

    def clear(self) -> None:
        self._cache.clear()


class BaseTool(BaseModel):
    """所有 CrewAI 工具的抽象基类。"""

    # ── 公共字段（Pydantic 序列化） ──
    name: str
    description: str = ""
    args_schema: Optional[Type[BaseModel]] = None
    cache_function: Optional[Callable[[Dict[str, Any], str], bool]] = None
    result_as_answer: bool = False

    # ── 私有字段（不序列化） ──
    _cache: CacheHandler = PrivateAttr(default_factory=CacheHandler)

    # ── 公共执行入口 ──

    def run(self, params: Dict[str, Any]) -> str:
        """执行工具：校验 → 缓存检查 → _run → 缓存写入。"""
        validated = self._validate_params(params)

        cache_key = self._make_cache_key(validated)

        # 检查缓存
        cached = self._cache.read(self.name, cache_key)
        if cached is not None:
            if self._should_cache(validated, cached):
                return cached

        # 执行核心逻辑
        result = self._run(**validated)

        # 写入缓存
        if self._should_cache(validated, result):
            self._cache.add(self.name, cache_key, result)

        return result

    # ── 子类必须实现 ──

    @abstractmethod
    def _run(self, **kwargs) -> str:
        """工具核心逻辑。"""
        ...

    # ── 内部方法 ──

    def _validate_params(self, params: Dict[str, Any]) -> Dict[str, Any]:
        """如果定义了 args_schema，使用 Pydantic 校验。"""
        if self.args_schema is not None:
            validated = self.args_schema(**params)
            return validated.model_dump()
        return params

    def _make_cache_key(self, params: Dict[str, Any]) -> str:
        """生成缓存键。"""
        return json.dumps(params, sort_keys=True, default=str)

    def _should_cache(self, params: Dict[str, Any], result: str) -> bool:
        """判断结果是否应该缓存。"""
        if self.cache_function is not None:
            return self.cache_function(params, result)
        return True  # 默认缓存所有结果

    def _generate_description(self) -> str:
        """生成注入到 Agent prompt 的描述。"""
        if self.args_schema:
            schema = self.args_schema.model_json_schema()
        else:
            schema = self._infer_schema()

        return json.dumps({
            "name": self.name,
            "description": self.description,
            "parameters": schema
        }, indent=2)

    def _infer_schema(self) -> Dict[str, Any]:
        """从 _run 方法签名推断参数 Schema。"""
        sig = inspect.signature(self._run)
        properties = {}
        required = []

        for param_name, param in sig.parameters.items():
            if param_name == "self":
                continue

            properties[param_name] = {
                "type": self._resolve_type(param.annotation),
                "description": param_name,
            }

            if param.default is inspect.Parameter.empty:
                required.append(param_name)

        return {
            "type": "object",
            "properties": properties,
            "required": required,
        }

    @staticmethod
    def _resolve_type(annotation: type) -> str:
        """将 Python 类型映射为 JSON Schema 类型。"""
        type_map = {
            str: "string",
            int: "integer",
            float: "number",
            bool: "boolean",
            list: "array",
            dict: "object",
        }
        return type_map.get(annotation, "string")
```

### 7.2 @tool 装饰器实现

```python
from functools import wraps

# 全局工具注册表
TOOL_REGISTRY: Dict[str, BaseTool] = {}


def tool(name_or_func: Optional[str | Callable] = None) -> Callable | BaseTool:
    """将函数转换为 CrewAI BaseTool 的装饰器。

    用法:
        @tool
        def my_tool(param: str) -> str: ...

        @tool("CustomName")
        def my_tool(param: str) -> str: ...

        @tool(cache_function=my_cache_func)
        def my_tool(param: str) -> str: ...
    """
    cache_func = None

    if callable(name_or_func):
        # 直接用作装饰器: @tool
        return _make_tool_from_func(name_or_func)

    if name_or_func is not None and not callable(name_or_func):
        # 带参数: @tool("Name") 或 @tool(cache_function=...)
        if isinstance(name_or_func, str):
            tool_name = name_or_func
            return lambda f: _make_tool_from_func(f, name=tool_name)
        elif isinstance(name_or_func, dict):
            cache_func = name_or_func.get("cache_function")

    def decorator(func: Callable) -> BaseTool:
        return _make_tool_from_func(func, name=name_or_func if isinstance(name_or_func, str) else None, cache_function=cache_func)

    return decorator


def _make_tool_from_func(
    func: Callable,
    name: Optional[str] = None,
    cache_function: Optional[Callable] = None,
) -> BaseTool:
    """从函数动态创建 BaseTool 实例。"""
    tool_name = name or func.__name__
    doc = func.__doc__ or ""

    # 动态构造 args_schema
    sig = inspect.signature(func)
    fields = {}
    for param_name, param in sig.parameters.items():
        annotation = param.annotation if param.annotation != inspect.Parameter.empty else str
        default = ... if param.default is inspect.Parameter.empty else param.default
        fields[param_name] = (annotation, Field(default=default, description=param_name))

    if fields:
        DynamicInputSchema = type(
            f"{tool_name}Input",
            (BaseModel,),
            {"__doc__": f"Input schema for {tool_name}", **fields},
        )
    else:
        DynamicInputSchema = None

    # 创建工具实例
    tool_instance = _create_tool_instance(
        name=tool_name,
        description=doc.strip(),
        args_schema=DynamicInputSchema,
        func=func,
        cache_function=cache_function,
    )

    # 注册到全局注册表
    TOOL_REGISTRY[tool_name] = tool_instance
    return tool_instance


def _create_tool_instance(
    name: str,
    description: str,
    args_schema: Optional[Type[BaseModel]],
    func: Callable,
    cache_function: Optional[Callable],
) -> BaseTool:
    """通过动态子类化创建 BaseTool 实例。"""
    class DecoratorTool(BaseTool):
        name: str = name
        description: str = description
        args_schema: Optional[Type[BaseModel]] = args_schema
        cache_function: Optional[Callable] = cache_function

        def _run(self, **kwargs) -> str:
            return str(func(**kwargs))

    return DecoratorTool()
```

### 7.3 Agent 任务执行与工具调用循环

```python
class Agent(BaseModel):
    role: str
    goal: str
    backstory: str
    tools: List[BaseTool] = []
    llm: Any  # LLM 客户端
    allow_delegation: bool = False
    max_iterations: int = 15
    max_execution_time: Optional[int] = None

    # ── 内部状态 ──
    _tool_mapping: Dict[str, BaseTool] = PrivateAttr(default_factory=dict)

    def execute_task(
        self,
        task: "Task",
        tools: List[BaseTool],
        context: Optional[Dict[str, Any]] = None,
    ) -> str:
        """执行任务的主体循环。"""
        # 1. 建立工具索引
        self._tool_mapping = {tool.name: tool for tool in tools}

        # 2. 构建系统提示
        system_prompt = self._build_system_prompt(tools)

        # 3. 构建消息列表
        messages = [
            {"role": "system", "content": system_prompt},
        ]

        # 4. 注入上下文
        if context:
            context_block = self._format_context(context)
            messages.append({"role": "user", "content": context_block})

        # 5. 注入任务描述
        messages.append({"role": "user", "content": task.description})

        # 6. ReAct 循环
        for iteration in range(self.max_iterations):
            # 6a. LLM 调用
            response = self.llm.invoke(messages)
            content = self._extract_content(response)

            # 6b. 解析工具调用
            action = self._parse_react_action(content)

            if action is None:
                # 没有 Action → 最终答案
                return content

            # 6c. 查找并执行工具
            tool = self._tool_mapping.get(action["name"])
            if tool is None:
                observation = (
                    f"Error: Tool '{action['name']}' not found. "
                    f"Available: {list(self._tool_mapping.keys())}"
                )
            else:
                try:
                    observation = tool.run(action["input"])
                except Exception as e:
                    observation = f"Tool execution error: {type(e).__name__}: {str(e)}"

            # 6d. 将 Observation 追加回对话
            messages.append({
                "role": "user",
                "content": f"Observation: {observation}"
            })

        return "Max iterations reached without final answer."

    def _build_system_prompt(self, tools: List[BaseTool]) -> str:
        """构建包含工具描述的 System Prompt。"""
        tool_descriptions = "\n\n".join(
            t._generate_description() for t in tools
        )

        tool_names = ", ".join(t.name for t in tools)

        prompt = (
            f"You are {self.role}.\n"
            f"Goal: {self.goal}\n"
            f"Backstory: {self.backstory}\n\n"
            f"You have access to the following tools:\n"
            f"{tool_descriptions}\n\n"
            f"IMPORTANT: Use the following format:\n"
            f"Thought: your reasoning\n"
            f"Action: one of [{tool_names}]\n"
            f"Action Input: JSON object with parameters\n"
            f"Observation: the result (provided by the system)\n"
            f"... (repeat Thought/Action/Observation as needed)\n"
            f"Thought: I have the final answer\n"
            f"Final Answer: your response to the user"
        )
        return prompt

    def _parse_react_action(self, content: str) -> Optional[Dict[str, Any]]:
        """从 LLM 输出中解析 ReAct Action。"""
        # 匹配 Action 和 Action Input
        action_match = re.search(
            r"Action:\s*(\w+)\s*\nAction Input:\s*(\{.*\})",
            content,
            re.DOTALL,
        )
        if action_match:
            try:
                return {
                    "name": action_match.group(1),
                    "input": json.loads(action_match.group(2)),
                }
            except json.JSONDecodeError:
                return None

        # 检查是否有 Final Answer
        if "Final Answer:" in content:
            return None  # 最终答案，没有工具调用

        return None

    def _format_context(self, context: Dict[str, Any]) -> str:
        """将上下文格式化为提示文本。"""
        parts = ["Here is the context from previous tasks:"]
        for key, value in context.items():
            parts.append(f"\n---\nSource: {key}\n{value}\n---")
        return "\n".join(parts)
```

### 7.4 Task 执行与上下文传播

```python
class Task(BaseModel):
    description: str
    expected_output: str
    agent: Optional[Agent] = None
    tools: Optional[List[BaseTool]] = None
    context: Optional[List["Task"]] = None
    async_execution: bool = False
    output: Optional[str] = None

    def execute(
        self,
        task_outputs: Dict["Task", str],
        global_inputs: Dict[str, Any],
    ) -> str:
        """执行任务并返回结果。"""
        # 1. 收集上下文
        context = self._collect_context(task_outputs, global_inputs)

        # 2. 确定使用的工具
        active_tools = self.tools if self.tools is not None else (
            self.agent.tools if self.agent else []
        )

        # 3. Agent 执行
        result = self.agent.execute_task(
            task=self,
            tools=active_tools,
            context=context,
        )

        self.output = result
        return result

    def _collect_context(
        self,
        task_outputs: Dict["Task", str],
        global_inputs: Dict[str, Any],
    ) -> Dict[str, Any]:
        """收集所有上下文来源。"""
        context = dict(global_inputs) if global_inputs else {}

        # 添加依赖任务的输出
        if self.context:
            for dep_task in self.context:
                if dep_task in task_outputs:
                    context[dep_task.description] = task_outputs[dep_task]

        return context
```

### 7.5 Crew 编排器完整流程

```python
class Crew(BaseModel):
    agents: List[Agent]
    tasks: List[Task]
    process: Process = Process.sequential
    manager_agent: Optional[Agent] = None
    manager_llm: Optional[Any] = None
    verbose: bool = False
    step_callback: Optional[Callable] = None
    task_callback: Optional[Callable] = None
    cache: bool = True

    def kickoff(self, inputs: Optional[Dict[str, Any]] = None) -> str:
        """编排启动入口。"""
        self._setup_cache()

        if self.process == Process.sequential:
            return self._run_sequential(inputs)
        elif self.process == Process.hierarchical:
            return self._run_hierarchical(inputs)
        else:
            raise ValueError(f"Unknown process: {self.process}")

    def _setup_cache(self) -> None:
        """在所有 agent 间共享缓存实例。"""
        if self.cache:
            shared_cache = CacheHandler()
            for agent in self.agents:
                for tool in agent.tools:
                    tool._cache = shared_cache

    def _run_sequential(
        self, inputs: Optional[Dict[str, Any]] = None
    ) -> str:
        """顺序执行所有任务。"""
        task_outputs: Dict[Task, str] = {}
        inputs = inputs or {}

        for task in self.tasks:
            # 1. 收集前序任务输出作为上下文
            context = dict(inputs)
            if task.context:
                for dep in task.context:
                    if dep in task_outputs:
                        context[dep.description] = task_outputs[dep]

            # 2. 获取 agent 和工具
            agent = task.agent
            tools = task.tools or agent.tools

            # 3. 执行
            result = agent.execute_task(
                task=task,
                tools=tools,
                context=context,
            )

            # 4. 缓存输出
            task_outputs[task] = result

            # 5. 回调
            if self.task_callback:
                self.task_callback(task, result)

            if self.verbose:
                print(f"[{task.description[:50]}...] completed")

        return task_outputs[self.tasks[-1]]

    def _run_hierarchical(
        self, inputs: Optional[Dict[str, Any]] = None
    ) -> str:
        """层级模式：Manager Agent 委派工作。"""
        # 1. 创建或使用 manager agent
        manager = self.manager_agent or self._create_manager_agent()

        # 2. 构建 Manager 的任务：管理所有子任务
        task_descriptions = "\n".join(
            f"  {i+1}. {t.description} (assigned to: {t.agent.role})"
            for i, t in enumerate(self.tasks)
        )

        manager_task = Task(
            description=(
                f"Manage the following tasks and delegate them "
                f"to the appropriate workers:\n{task_descriptions}\n\n"
                f"Use the DelegateWorkTool to assign each task "
                f"to the correct coworker."
            ),
            expected_output="All tasks completed and results aggregated.",
            agent=manager,
        )

        # 3. 为 manager 提供 AgentTools（delegate + ask）
        manager_tools = AgentTools(agents=self.agents).tools()
        manager.tools.extend(manager_tools)

        # 4. 执行
        result = manager.execute_task(
            task=manager_task,
            tools=manager.tools,
            context=inputs or {},
        )

        return result
```

---

## 8. 局限性与改进

### 8.1 工具错误处理局限

| 问题 | 描述 | 影响 |
|------|------|------|
| 原始异常泄露 | 工具 `_run` 抛出的异常被直接转换为 `Observation: Error ...` 字符串 | LLM 可能被技术性错误信息迷惑，无法正确恢复 |
| 无重试机制 | 工具调用失败后直接返回错误信息，没有自动重试 | 瞬时故障（如网络超时）导致整个任务失败 |
| 错误信息不结构化 | 所有错误被 flatten 为纯文本字符串，丢失原始错误类型和堆栈 | 难以在回调中区分错误类型做不同处理 |

### 8.2 上下文窗口溢出

```
问题：长链任务中的上下文累积

Task 1: 3,000 tokens output
Task 2: 3,000 tokens + Task 1 context (3,000) = 6,000 tokens
Task 3: 3,000 tokens + Task 1+2 context (6,000) = 9,000 tokens
...
Task 10: 3,000 tokens + previous 9 tasks (27,000) = 30,000 tokens in prompt
```

- CrewAI 不主动压缩或摘要历史上下文
- 依赖底层 LLM 的上下文窗口（Claude 3.5 Sonnet: 200K, GPT-4o: 128K）
- 超出窗口时，LLM 静默截断早期上下文，导致任务遗忘

### 8.3 工具缓存粒度粗糙

- 默认缓存所有工具调用结果（除非显式设置 `cache_function` 返回 False）
- 缓存键基于完整的参数 JSON 序列化，对包含时间戳、随机数的参数无效
- 缓存是进程级内存缓存，多进程/分布式场景下不共享
- 无缓存过期机制（TTL），长时间运行的 Crew 可能使用过期数据

### 8.4 异步工具支持不成熟

- `BaseTool._run` 是同步方法，`_arun`（异步版本）在早期版本中存在但文档不完善
- 工具内部的 IO 操作（HTTP 请求、文件读写）会阻塞事件循环
- Agent 的工具调用循环本质上是同步的，不支持并发工具调用
- 即使 Task 支持 `async_execution`，每个工具调用本身还是串行的

---

## 9. 生产优化建议

### 9.1 自定义工具错误包装

```python
class ToolExecutionError(Exception):
    """结构化工具执行错误。"""
    def __init__(self, original: Exception, tool_name: str,
                 params: dict):
        self.original = original
        self.tool_name = tool_name
        self.params = params
        self.error_type = type(original).__name__
        super().__init__(self._format_message())

    def _format_message(self) -> str:
        return (
            f"[Tool Error - {self.tool_name}]\n"
            f"Error Type: {self.error_type}\n"
            f"Message: {str(self.original)}\n"
            f"Suggested action: Try rephrasing your request "
            f"or using different parameters."
        )


class SafeBaseTool(BaseTool):
    """带错误包装的基础工具。"""

    def run(self, params: Dict[str, Any]) -> str:
        try:
            return super().run(params)
        except ToolExecutionError:
            raise  # 已经是包装过的，直接传递
        except Exception as e:
            raise ToolExecutionError(e, self.name, params) from e


class RetryToolWrapper:
    """为工具添加重试能力。"""

    def __init__(self, tool: BaseTool, max_retries: int = 3,
                 retry_delay: float = 1.0):
        self.tool = tool
        self.max_retries = max_retries
        self.retry_delay = retry_delay

    def run(self, params: Dict[str, Any]) -> str:
        last_error = None
        for attempt in range(self.max_retries):
            try:
                return self.tool.run(params)
            except ToolExecutionError as e:
                last_error = e
                if attempt < self.max_retries - 1:
                    time.sleep(self.retry_delay * (attempt + 1))
        return f"Failed after {self.max_retries} attempts: {last_error}"
```

### 9.2 工具结果缓存层

```python
import hashlib
import time
from typing import Any, Dict, Optional, Tuple

class TTLToolCache(CacheHandler):
    """带 TTL 的缓存层。"""

    def __init__(self, default_ttl: int = 300):
        super().__init__()
        self.default_ttl = default_ttl
        self._timestamps: Dict[str, float] = {}

    def add(self, tool_name: str, input_key: str,
            output: str, ttl: Optional[int] = None) -> None:
        ttl = ttl or self.default_ttl
        cache_key = f"{tool_name}-{input_key}"
        self._cache[cache_key] = output
        self._timestamps[cache_key] = time.time() + ttl

    def read(self, tool_name: str, input_key: str) -> Optional[str]:
        cache_key = f"{tool_name}-{input_key}"
        if cache_key in self._cache:
            if time.time() > self._timestamps.get(cache_key, 0):
                del self._cache[cache_key]
                del self._timestamps[cache_key]
                return None
            return self._cache[cache_key]
        return None

    def make_cache_key(self, params: Dict[str, Any]) -> str:
        """过滤掉时间戳等易变参数。"""
        stable = {
            k: v for k, v in params.items()
            if k not in ("timestamp", "time", "random", "seed")
        }
        raw = json.dumps(stable, sort_keys=True, default=str)
        return hashlib.sha256(raw.encode()).hexdigest()
```

### 9.3 上下文摘要中间件

```python
class ContextSummarizer:
    """在任务间自动摘要上下文，防止窗口溢出。"""

    def __init__(self, llm: Any, max_context_tokens: int = 8000):
        self.llm = llm
        self.max_context_tokens = max_context_tokens

    def compress(self, context: Dict[str, str]) -> Dict[str, str]:
        """如果上下文超过阈值，对最旧的内容做摘要。"""
        total_tokens = sum(self._count_tokens(v) for v in context.values())

        if total_tokens <= self.max_context_tokens:
            return context

        compressed = {}
        accumulated = 0

        # 保留最新（最后）的上下文原样，压缩较早的
        items = list(context.items())
        for key, value in reversed(items):
            tokens = self._count_tokens(value)
            if accumulated + tokens <= self.max_context_tokens * 0.6:
                compressed[key] = value
                accumulated += tokens
            else:
                compressed[f"{key} (summarized)"] = self._summarize(value)
                break  # 之前的全部摘要

        # 对剩余的较旧内容做合并摘要
        remaining = items[len(compressed):-1]
        if remaining:
            old_content = "\n\n".join(v for _, v in remaining)
            compressed["Previous tasks (summarized)"] = self._summarize(old_content)
            if len(compressed) > 1:
                # 只保留 summary + 最新上下文
                latest = compressed.popitem()
                compressed = dict([("Previous tasks (summarized)",
                                    next(v for k, v in compressed.items()
                                         if "summarized" in k)),
                                   latest])

        return compressed

    def _summarize(self, text: str) -> str:
        """调用 LLM 生成摘要。"""
        prompt = (
            f"Summarize the following content concisely, "
            f"preserving all key facts and data:\n\n{text}"
        )
        return self.llm.invoke(prompt)

    @staticmethod
    def _count_tokens(text: str) -> int:
        """粗略估算 token 数。"""
        return len(text) // 4  # 约 4 字符/token
```

### 9.4 并行工具执行支持

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor


class ParallelToolExecutor:
    """支持 Agent 同时调用多个独立工具。"""

    def __init__(self, max_workers: int = 5):
        self.executor = ThreadPoolExecutor(max_workers=max_workers)

    def execute_parallel(
        self, tool_calls: List[Tuple[BaseTool, Dict[str, Any]]]
    ) -> List[str]:
        """并行执行多个工具调用。"""
        futures = []
        for tool, params in tool_calls:
            future = self.executor.submit(tool.run, params)
            futures.append(future)

        results = []
        for future in futures:
            try:
                results.append(future.result(timeout=30))
            except Exception as e:
                results.append(f"Error: {str(e)}")

        return results


class ParallelAwareAgent(Agent):
    """支持并行工具调用的 Agent。"""

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self._parallel_executor = ParallelToolExecutor()

    def execute_task(self, task, tools, context=None):
        # 在 prompting 中增加并行调用提示
        system = self._build_system_prompt(tools)
        system += (
            "\n\nYou can call multiple tools at once when they are "
            "independent. Use the following format:\n"
            "Actions:\n"
            "  - Action: tool_name_1\n"
            "    Action Input: {...}\n"
            "  - Action: tool_name_2\n"
            "    Action Input: {...}\n"
            "Observations will be returned in order."
        )
        # ... 执行逻辑中的解析需要支持多 Action 解析
```

---

## 附录：关键 ASCII 架构图

### 图1：工具定义 → 注册 → 执行 → 结果流

```
  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐
  │ 定义工具   │    │ 注册到 Agent  │    │ Agent 执行时注入    │
  │           │    │              │    │ 到 System Prompt  │
  │ @tool     │───▶│ agent =       │───▶│                   │
  │ def func():│    │   Agent(      │    │ "You have access  │
  │   ...     │    │     tools=[t] │    │  to these tools:  │
  │           │    │   )          │    │  {name, desc,     │
  │ 或        │    │              │    │   parameters}"    │
  │ class T   │    │              │    │                   │
  │ (BaseTool)│    │              │    └────────┬──────────┘
  └──────────┘    └──────────────┘             │
                                                │
                                                ▼
  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐
  │ LLM 返回  │    │ Agent 执行    │    │ LLM 输出 Action +  │
  │ 最终答案   │◀───│ ReAct 循环    │◀───│ Action Input      │
  │           │    │              │    │                   │
  │ "Final    │    │ while not    │    │ Thought: ...      │
  │ Answer:"  │    │   final:     │    │ Action: search    │
  │           │    │   LLM →      │    │ Action Input:     │
  │           │    │   Tool →     │    │   {"q": "AI"}     │
  │           │    │   Observe    │    │                   │
  └──────────┘    └──────────────┘    └──────────────────┘
```

### 图2：Task-Tool-Agent 绑定关系

```
┌──────────────────────────────────────────────────────────────────┐
│                         TASK                                       │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Task(                                                     │    │
│  │    description="Research...",                             │    │
│  │    agent=researcher,          ──────┐                     │    │
│  │    tools=None,                       │ (继承 Agent 的工具)   │    │
│  │    context=[prev_task]               │                     │    │
│  │  )                                  │                     │    │
│  └──────────────────────────────────────┼─────────────────────┘    │
│                                         │                          │
└─────────────────────────────────────────┼──────────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
                    ▼                     ▼                     ▼
        ┌────────────────────┐  ┌────────────────────┐  ┌──────────────┐
        │      AGENT         │  │      TOOLS         │  │   CONTEXT    │
        │                    │  │                    │  │              │
        │  Agent(            │  │  search_tool ──────┤──▶ prev_task    │
        │    role="researcher"│  │  scrape_tool ──────┤──▶ output      │
        │    goal="...",     │  │  calc_tool   ──────┤──▶             │
        │    tools=[...],    │  │                    │  │              │
        │    llm=...         │  │  (工具被注入        │  │              │
        │  )                 │  │   system prompt)   │  │              │
        └────────────────────┘  └────────────────────┘  └──────────────┘
                                          │
                                          ▼
                               ┌────────────────────┐
                               │ TOOL EXECUTION      │
                               │                    │
                               │  tool.run(         │
                               │    params          │
                               │  )                 │
                               │    → validate      │
                               │    → cache check   │
                               │    → _run()        │
                               │    → cache store   │
                               │    → return result │
                               └────────────────────┘
```

### 图3：上下文在 Sequential Process 中的传播

```
Time ──────────────────────────────────────────────────────────▶

Crew.kickoff(inputs={"topic": "AI"})
  │
  ▼
┌──────────────────────────────────────────────────────────────┐
│ Task 1: Research                                              │
│                                                               │
│  Prompt:                                                      │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ System: You are a researcher...                        │   │
│  │ User: Research the topic: AI                           │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                               │
│  Output: "AI research results: transformers, LLMs, ..."       │
└──────────────────────────────────┬───────────────────────────┘
                                   │
                                   ▼ context={Task1: output}
                                   │
┌──────────────────────────────────────────────────────────────┐
│ Task 2: Analysis                                              │
│                                                               │
│  Prompt:                                                      │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ System: You are an analyst...                          │   │
│  │ User (Context):                                        │   │
│  │   Here is the context from previous tasks:             │   │
│  │   ┌────────────────────────────────────────────────┐   │   │
│  │   │ Source: Research the topic                      │   │   │
│  │   │ AI research results: transformers, LLMs, ...    │   │   │
│  │   └────────────────────────────────────────────────┘   │   │
│  │ User (Task): Analyze the research findings...          │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                               │
│  Output: "Analysis: key trends identified..."                  │
└──────────────────────────────────┬───────────────────────────┘
                                   │
                                   ▼ context={Task1: output,
                                   │          Task2: output}
                                   │
┌──────────────────────────────────────────────────────────────┐
│ Task 3: Writing                                              │
│                                                               │
│  Prompt:                                                      │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ System: You are a writer...                           │   │
│  │ User (Context):                                        │   │
│  │   Source: Research → AI research results: ...          │   │
│  │   Source: Analysis → Analysis: key trends ...          │   │
│  │ User (Task): Write an article based on the context...  │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                               │
│  Output: "Final article..." ← Crew 最终输出                    │
└──────────────────────────────────────────────────────────────┘
```

### 图4：Hierarchical Process 任务委派与工具使用

```
                  ┌─────────────────────────────────────────┐
                  │          Manager Agent                    │
                  │  LLM + DelegateWorkTool + AskQuestionTool │
                  └──────────────┬──────────────────────────┘
                                 │
           ┌─────────────────────┼─────────────────────┐
           │  Delegate Task 1    │  Delegate Task 2    │  Delegate Task 3
           ▼                     ▼                     ▼
  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
  │  Researcher      │  │  Analyst         │  │  Writer          │
  │                   │  │                  │  │                  │
  │  Tools:           │  │  Tools:           │  │  Tools:          │
  │  ┌───────────┐   │  │  ┌───────────┐    │  │  ┌───────────┐   │
  │  │search_tool│   │  │  │sql_tool   │    │  │  │fmt_tool   │   │
  │  └───────────┘   │  │  └───────────┘    │  │  └───────────┘   │
  │  ┌───────────┐   │  │  ┌───────────┐    │  │  ┌───────────┐   │
  │  │scrape_tool│   │  │  │stat_tool  │    │  │  │md_tool    │   │
  │  └───────────┘   │  │  └───────────┘    │  │  └───────────┘   │
  │                   │  │                  │  │                  │
  │  ReAct Loop:      │  │  ReAct Loop:     │  │  ReAct Loop:     │
  │  think → search   │  │  think → query   │  │  think → format  │
  │  → observe → ...  │  │  → analyze → ... │  │  → write → ...   │
  │                   │  │                  │  │                  │
  │  Output: research │  │  Output: analysis │  │  Output: article │
  └─────────┬─────────┘  └─────────┬─────────┘  └─────────┬─────────┘
            │                      │                      │
            └──────────────────────┼──────────────────────┘
                                   │
                                   ▼
                  ┌─────────────────────────────────────────┐
                  │     Manager Agent aggregates results     │
                  │     → Final output                       │
                  └─────────────────────────────────────────┘
```

---

## 参考资源

- [CrewAI 官方文档 - Tools](https://docs.crewai.com/v1.15.1/en/concepts/tools)
- [CrewAI 官方文档 - Tasks](https://docs.crewai.com/v1.15.1/en/concepts/tasks)
- [CrewAI 官方文档 - 自定义工具](https://docs.crewai.com/v1.15.1/en/learn/create-custom-tools)
- [CrewAI 官方文档 - Processes](https://docs.crewai.com/edge/en/concepts/processes)
- [CrewAI GitHub](https://github.com/crewAIInc/crewAI)
- [CrewAI Tools GitHub](https://github.com/crewAIInc/crewAI-tools)
- [DeepWiki: crewAI Architecture](https://deepwiki.com/crewAIInc/crewAI-tools/1.2-architecture-and-design)
