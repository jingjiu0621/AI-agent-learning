# 14.1.5 单体 Agent (Monolithic Agent)

## 1. 简介与背景

### 什么是单体 Agent

**单体 Agent (Monolithic Agent)** 是一种将智能体所有功能组件打包在**单一进程、单一地址空间**中运行的架构模式。在这种模式下，Agent 的感知（Perception）、推理（Reasoning）、行动（Action）、记忆（Memory）和工具调用（Tool Use）全部在同一个进程内完成，不存在跨进程通信（IPC, Inter-Process Communication）或网络调用（除 LLM API 本身外）。

单体 Agent 的核心特征是：

- **单一进程**：整个 Agent 运行在一个操作系统进程中
- **统一状态空间**：所有状态（对话历史、工具结果、中间变量）保存在进程内存中
- **线性控制流**：执行路径为 `Thought -> Action -> Observation -> Thought...` 的紧密循环
- **直接函数调用**：工具调用是本地函数调用，不涉及网络 RPC（Remote Procedure Call）

### 历史背景

单体架构在软件工程中并非新鲜事物。在微服务流行之前，绝大多数应用都采用单体架构（Monolithic Architecture）。Agent 领域的单体模式与传统单体架构在哲学上一脉相承：

```
传统单体 Web 应用                    单体 Agent
┌─────────────────────┐      ┌─────────────────────┐
│  UI Layer           │      │  LLM Interface      │
│  Business Logic     │      │  Reasoning Engine   │
│  Data Access        │  →   │  Tool Registry      │
│  Auth               │      │  Memory Manager     │
│  All in one process │      │  All in one process │
└─────────────────────┘      └─────────────────────┘
```

早期的 AI Agent 实现（如 2023 年的 AutoGPT、BabyAGI 早期版本）几乎全部采用单体架构——一个 Python 脚本、一个 `while` 循环、一个 LLM 调用，就构成了一个 Agent。

---

## 2. 基本原理

### 单进程架构

单体 Agent 的进程模型极为简单：

```
┌──────────────────────────────────────────┐
│           操作系统进程 (PID: 1234)          │
│                                          │
│  ┌─────────┐  ┌──────────┐  ┌────────┐  │
│  │ LLM     │  │ Tool     │  │ Memory │  │
│  │ Client  │──│ Registry │──│ Store  │  │
│  └────┬────┘  └────┬─────┘  └────┬───┘  │
│       │            │             │       │
│       └────────────┴─────────────┘       │
│                    │                     │
│          ┌─────────┴──────────┐          │
│          │   Agent Loop       │          │
│          │  (while True)      │          │
│          └────────────────────┘          │
└──────────────────────────────────────────┘
```

### 进程内状态管理

所有状态直接存储在进程内存中，无需序列化/反序列化：

```python
# 单体 Agent 的状态管理 — 全部在内存中
class MonolithicAgentState:
    def __init__(self):
        self.conversation_history: list[dict] = []   # 对话历史
        self.tool_results: dict[str, Any] = {}       # 工具执行结果缓存
        self.current_step: int = 0                    # 当前步骤计数
        self.max_steps: int = 20                      # 最大步数限制
        self.intermediate_thoughts: list[str] = []    # 思维链中间结果
```

### 线性控制流

单体 Agent 的控制流是严格顺序的，每一步依赖上一步的结果：

```
Step 1: LLM 接收用户输入 → 生成 Thought + Action
Step 2: 执行 Action (工具调用) → 得到 Observation
Step 3: LLM 接收 Observation → 生成下一个 Thought + Action (或 Final Answer)
Step N: ...
```

这种控制流的数学表示：

```
S₀ = 初始状态 (用户输入 + 系统提示词)
S₁ = LLM(S₀) → Thought₁ + Action₁
S₂ = Execute(Action₁) → Tool₁ output
S₃ = LLM(S₂) → Thought₂ + Action₂ (或 Final Answer)
...
S₂ₙ₊₁ = Final Answer
```

---

## 3. 之前怎么做 vs 现在

### 传统规则型单体 Agent (Before LLM)

在大型语言模型出现之前，单体 Agent 通常基于规则引擎或专家系统实现：

```python
# 传统规则型单体 Agent (伪代码)
class RuleBasedMonolithicAgent:
    def __init__(self):
        self.knowledge_base = load_knowledge_base()
        self.rule_engine = RuleEngine()
        self.rules = [
            Rule(condition="user asks about weather", action="call_weather_api"),
            Rule(condition="user asks about time", action="get_current_time"),
            # ... 数百条硬编码规则
        ]

    def run(self, user_input: str) -> str:
        for rule in self.rules:
            if rule.matches(user_input):  # 基于关键词/模板匹配
                return rule.execute()
        return "I don't understand"  # 超出规则范围即失败
```

**局限性**：
- 规则覆盖范围有限，无法处理未预见的情况
- 自然语言理解能力极差，依赖关键词匹配或正则表达式
- 状态管理僵硬，难以维护多轮对话上下文
- 每增加一个功能都需要手动添加规则

### 现代 LLM 驱动型单体 Agent (Now)

```python
# 现代 LLM 驱动单体 Agent (伪代码)
class LLMMonolithicAgent:
    def __init__(self):
        self.llm = LLMClient(model="gpt-4")  # LLM 取代规则引擎
        self.tools = ToolRegistry()
        self.memory = ConversationMemory()

    def run(self, user_input: str) -> str:
        messages = self.memory.get_context() + [{"role": "user", "content": user_input}]
        response = self.llm.chat(messages, tools=self.tools.get_schemas())
        # LLM 自主决定调用哪个工具，无需硬编码规则
        return self.execute_response(response)
```

**根本性变化**：

| 维度 | 传统规则型 | 现代 LLM 型 |
|------|-----------|------------|
| 推理引擎 | if-else 规则链 | 大语言模型 |
| 语言理解 | 关键词/正则 | 语义理解 |
| 工具调用 | 硬编码触发条件 | LLM 自主选择 |
| 状态维护 | 显式状态机 | 隐式上下文窗口 |
| 扩展能力 | 需手动加规则 | 加工具描述即可 |
| 容错能力 | 规则外即失败 | 可推理未知情况 |

### 关键洞察

现代单体 Agent 的"单体"体现在**架构上仍然是一个进程**，但内部的智能决策从规则引擎升级为 LLM。这带来了质的飞跃：不再需要为每个可能的输入编写规则，LLM 通过自然语言理解就能动态决定行为。

---

## 4. 核心矛盾

### 4.1 简单性 vs 可扩展性 (Simplicity vs Scalability)

```
简单性                 可扩展性
  ●─────────────────────●
  单体 Agent             分布式 Agent

  优势:                   优势:
  - 3 小时可搭建原型      - 支持 1000+ 并发
  - 1 个文件全部逻辑      - 组件独立扩缩容
  - 全局调试方便          - 可单独优化瓶颈

  代价:                   代价:
  - 单进程性能天花板      - 3 周才能搭建
  - 无法处理高并发        - DevOps 复杂度剧增
  - 组件耦合紧密          - 调试困难
```

**现实困境**：项目初期单体模式开发效率极高，但当系统增长到一定规模后，单体架构会成为瓶颈。问题在于**何时拆分**——拆分太早增加不必要的复杂度，拆分太晚重构成本巨大。

### 4.2 单体一致性 vs 团队并行开发

```
场景: 5 人团队开发同一个单体 Agent

                    ┌───────────────┐
                    │  agent.py     │ ← Git 冲突热点
                    ├───────────────┤
  Alice ───→ 添加工具  │  def run():    │
  Bob   ───→ 改记忆逻辑 │  def think():  │
  Carol ───→ 修异常处理 │  def act():    │
                    │  def remember()│
                    └───────────────┘
```

单体 Agent 的代码耦合度天然很高。核心的 agent loop、prompt 模板、状态管理往往集中在少数文件中。当团队超过 2-3 人时，频繁的合并冲突和相互阻塞成为常态。

### 4.3 全合一进程 vs 资源隔离

```
单体进程崩溃级联效应:

        用户请求
            │
            ▼
    ┌───────────────────┐
    │   Agent 主进程     │
    ├───────────────────┤
    │   1. LLM 调用     │──── 超时卡住整个 Agent
    │   2. 代码执行器    │──── 恶意代码导致进程崩溃
    │   3. 浏览器工具    │──── 浏览器 OOM 拖垮进程
    │   4. 文件操作      │──── 权限错误导致状态丢失
    │   5. 内存缓存      │──── 无限增长导致 OOM
    └───────────────────┘
            │
            ▼
    整个 Agent 不可用 ❌
```

在单体架构中，任何一个子组件的故障都可能导致整个 Agent 进程崩溃。工具执行中的内存泄漏、LLM 调用的长时间阻塞、第三方库的崩溃——这些问题在单体中都无法隔离。

### 4.4 紧耦合 vs 灵活性

```python
# 单体 Agent 中的紧耦合示例
class MonolithicAgent:
    def __init__(self):
        # 直接依赖具体实现，难以替换
        self.llm = OpenAILLM(model="gpt-4")      # 想换 Claude 需要改代码
        self.memory = InMemoryMemory()            # 想换 Redis 需要改代码
        self.tools = LocalToolRegistry()          # 想换远程工具需要改代码
        self.embedder = OpenAIEmbedder()           # 想换本地模型需要改代码
```

在单体中，各个组件往往通过直接实例化绑在一起。虽然可以通过接口抽象来解耦，但在快速的 MVP 开发中，开发人员通常跳过这一步骤，导致后期替换成本高昂。

---

## 5. 主流优化方向

### 5.1 模块化单体 (Modular Monolithic)

即使在一个进程中运行，也可以通过清晰的内部接口实现模块化：

```
┌──────────────────────────────────────────────┐
│               模块化单体 Agent                  │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │  Interface │  │  Interface │  │  Interface  │  │
│  │  LLM       │  │  Memory   │  │  Tool       │  │
│  │  Provider  │  │  Provider │  │  Provider   │  │
│  ├──────────┤  ├──────────┤  ├────────────┤  │
│  │ OpenAI   │  │ InMemory │  │ Local     │  │
│  │ Claude   │  │ Redis    │  │ Remote    │  │
│  │ Local    │  │ SQLite   │  │ Sandboxed │  │
│  └──────────┘  └──────────┘  └────────────┘  │
│                                              │
│  ┌──────────────────────────────────────────┐ │
│  │          Agent Loop (Orchestrator)       │ │
│  │  只依赖接口，不依赖具体实现                   │ │
│  └──────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

### 5.2 异步非阻塞 (Async Non-Blocking)

在单体中使用异步 IO 避免 LLM 调用阻塞整个进程：

```python
import asyncio

class AsyncMonolithicAgent:
    async def run(self, user_input: str) -> str:
        # LLM 调用不阻塞事件循环
        thought = await self.llm.athink(user_input)

        if thought.has_tool_call:
            # 并发执行多个独立工具
            results = await asyncio.gather(
                self.execute_tool(thought.tool_calls[0]),
                self.execute_tool(thought.tool_calls[1]),
                return_exceptions=True
            )
            # 超时保护
            final = await asyncio.wait_for(
                self.llm.athink(self.build_prompt(results)),
                timeout=30.0
            )
            return final

    async def execute_tool(self, call):
        try:
            return await asyncio.wait_for(
                self.tools.execute(call), timeout=10.0
            )
        except asyncio.TimeoutError:
            return {"error": "Tool timeout"}
```

### 5.3 优雅降级与熔断 (Graceful Degradation)

```python
class CircuitBreaker:
    def __init__(self, threshold=3, recovery_time=30):
        self.failures = 0
        self.threshold = threshold
        self.recovery_time = recovery_time
        self.last_failure_time = 0
        self.state = "closed"  # closed / open / half-open

    async def call(self, func, fallback=None):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.recovery_time:
                self.state = "half-open"
            else:
                return fallback() if fallback else {"error": "circuit_breaker_open"}

        try:
            result = await func()
            self.failures = 0
            self.state = "closed"
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure_time = time.time()
            if self.failures >= self.threshold:
                self.state = "open"
            return fallback() if fallback else {"error": str(e)}
```

### 5.4 内嵌小模型处理子任务

在单体中嵌入轻量级模型处理特定子任务，减少对主 LLM 的依赖：

```python
class HybridMonolithicAgent:
    def __init__(self):
        self.main_llm = LLMClient("gpt-4")      # 主推理引擎
        self.classifier = SmallModel("bert")    # 意图分类（本地）
        self.extractor = SmallModel("ner")      # 实体提取（本地）
        self.embedder = SmallModel("embed")     # 向量嵌入（本地）

    async def run(self, user_input: str) -> str:
        # 1. 本地小模型做意图分类（毫秒级）
        intent = self.classifier.predict(user_input)

        # 2. 简单意图直接处理，无需 LLM
        if intent == "greeting":
            return "Hello! How can I help you today?"

        if intent == "calculate":
            result = self.fast_calculator(user_input)
            return f"The result is {result}"

        # 3. 复杂推理才调用 LLM
        entities = self.extractor.extract(user_input)
        context = self.embedder.search(entities)
        return await self.main_llm.achat(context + user_input)
```

---

## 6. 能产生什么结果

### 快速原型验证

采用单体 Agent 架构的项目，从想法到可演示原型通常只需 1-3 天：

```python
# 一个完整的单体 Agent 原型可以在 100 行内实现
# 这使得快速验证"Agent 能否解决这个问题"成为可能

def demo():
    agent = MonolithicAgent(
        llm="gpt-4",
        tools=[web_search, calculator, file_reader]
    )
    # 几分钟内即可测试
    print(agent.run("Search the web for AI news and summarize"))
```

### 易于调试

单体架构的调试优势是革命性的：

- **单步调试**：可以在任何地方设置断点，跟踪每一步的变量状态
- **日志集中**：所有组件输出到同一个日志文件，无需跨服务追踪
- **可复现**：由于状态在本地内存，可以轻松保存和重放完整会话

### 运维成本极低

| 运维活动 | 单体成本 | 微服务成本 |
|---------|---------|-----------|
| 部署 | `python agent.py` | Docker + K8s + CI/CD |
| 监控 | 单一进程监控 | 分布式追踪 + 多服务仪表盘 |
| 备份 | 保存内存状态 | 同步多个服务状态 |
| 扩容 | 垂直扩容 | 水平扩容 + 服务发现 |

### 适合的场景输出

- **MVP 产品**：快速推向市场验证需求
- **个人助手工具**：满足个人或小团队使用
- **内部自动化脚本**：替代复杂的自动化流程
- **教学演示**：让学生快速理解 Agent 工作原理

---

## 7. 最大挑战

### 7.1 资源争用 (Resource Contention)

```python
# 在同步单体中，LLM 调用会阻塞所有操作
class SyncMonolithicAgent:
    def handle_user_message(self, user_input):
        # 这个调用可能耗时 3-10 秒
        # 在此期间，整个 Agent 无法处理任何其他请求
        response = self.llm.chat(user_input)
        return response

# 在异步单体中，虽然不阻塞事件循环，
# 但 CPU 密集型工具仍然会占用 GIL (Python)
class AsyncMonolithicAgent:
    async def handle_user_message(self, user_input):
        # asyncio 不解决 CPU 密集型工具的阻塞问题
        result = await self.run_cpu_intensive_tool()  # 仍然卡住事件循环
        return result
```

### 7.2 无法独立扩缩子组件

```
流量增长时的困境:

用户请求 50 QPS
      │
      ▼
┌──────────────────────┐
│    单体 Agent         │ ← 所有组件必须一起扩容
│                      │
│  LLM 调用: 10 req/s  │ ← 只需要扩容这个组件
│  向量搜索: 100 req/s │ ← 并不需要扩容
│  代码执行: 5 req/s   │ ← 并不需要扩容
│  缓存查询: 500 req/s │ ← 并不需要扩容
└──────────────────────┘
      │
      ▼
被迫整体扩容: 浪费资源
```

在单体中，无法单独为瓶颈组件扩容。如果 LLM 调用是瓶颈，你不能只加 LLM 调用进程——你必须复制整个 Agent 实例，包括不需要扩容的内存和向量搜索组件。

### 7.3 技术锁定 (Technology Lock-in)

单体 Agent 架构容易导致隐式的技术锁定：

```python
# 开始时使用的简单方案，后来成为迁移障碍
class MonolithicAgent:
    def __init__(self):
        # 刚开始无意识地选择了 OpenAI
        self.llm = OpenAILLM()
        self.embedder = OpenAIEmbedding()

        # 随着开发深入，大量 prompt 和逻辑
        # 都是针对 GPT 的格式和风格优化的
        # 日后再想切换到 Claude 或本地模型
        # 需要重写大量 prompt 模板和解析逻辑

    def parse_llm_response(self, response):
        # 这是针对 GPT 输出格式写的解析逻辑
        # 换模型后输出格式变化，这段代码需要重写
        return extract_tool_calls_from_gpt_format(response)
```

### 7.4 增长复杂度失控

随着功能增加，单体 Agent 的复杂度呈非线性增长：

```
功能数量 vs 复杂度增长

复杂度 ↑
        │                          /
        │                     ╱──
        │                 ╱─
        │            ╱──         复杂度的非线性增长
        │        ╱─               (耦合导致的连锁效应)
        │    ╱──
        │ ╱─
        │─
        └──────────────────────────→ 功能数量
```

**典型失控表现**：
- `agent.py` 从 200 行膨胀到 5000 行
- Prompt 模板互相冲突，修改一个影响其他
- 工具注册逻辑复杂到需要专门的框架来管理
- 状态管理分支覆盖不全，出现难以排查的竞态条件

---

## 8. 能力边界

### 何时保持单体 vs 何时拆分

```
决策矩阵: 保持单体还是拆分?

┌─────────────────────────────────────────────────────────────┐
│                     │  团队 ≤ 3 人  │  团队 > 3 人           │
├─────────────────────┼──────────────┼───────────────────────┤
│ 工具 ≤ 10          │  ✅ 单体      │  ⚠️ 模块化单体         │
│ 工具 10-30         │  ⚠️ 模块化单体 │  ❌ 考虑拆分           │
│ 工具 > 30          │  ❌ 考虑拆分   │  ❌ 必须拆分           │
├─────────────────────┼──────────────┼───────────────────────┤
│ 单用户工具          │  ✅ 单体      │  ⚠️ 模块化单体         │
│ 10-100 并发        │  ⚠️ 异步单体   │  ❌ 考虑微服务         │
│ >100 并发          │  ❌ 不可行     │  ❌ 必须分布式         │
├─────────────────────┼──────────────┼───────────────────────┤
│ 单一 LLM            │  ✅ 单体      │  ✅ 单体               │
│ 多 LLM 组合         │  ⚠️ 模块化单体 │  ⚠️ 模块化单体         │
│ 混合本地+云端模型   │  ❌ 考虑拆分   │  ❌ 必须拆分           │
├─────────────────────┼──────────────┼───────────────────────┤
│ 纯文本处理          │  ✅ 单体      │  ✅ 单体               │
│ 含代码执行          │  ⚠️ 需沙箱隔离 │  ❌ 必须拆分           │
│ 含物理设备控制      │  ❌ 必须拆分   │  ❌ 必须拆分           │
└─────────────────────┴──────────────┴───────────────────────┘
```

### 明确的拆分信号

当你观察到以下**任一**信号时，应考虑拆分：

1. **部署频率下降**：每次部署需要协调多个人的修改，部署周期从每天变成每周
2. **回归 bug 增多**：修改一个功能导致其他功能出现意外行为
3. **启动时间过长**：Agent 进程启动时间超过 30 秒
4. **内存持续增长**：单体进程的内存无法通过优化控制在合理范围
5. **团队阻塞**：多人同时修改 `agent.py` 的合并冲突每周超过 2 次
6. **工具执行安全风险**：工具执行可能导致整个进程崩溃（如运行用户代码）
7. **不同工具的 SLA 要求不同**：核心工具需要 99.99% 可用性，辅助工具可容忍偶尔失败

---

## 9. 与其他模式的区别

### 9.1 vs 管道模式 (Pipeline)

```
管道模式 (Pipeline)                   单体模式 (Monolithic)
┌──────┐   ┌──────┐   ┌──────┐      ┌─────────────────┐
│ Node1 │ → │ Node2 │ → │ Node3 │      │  Agent 进程     │
│       │   │       │   │       │      │ ┌────┐ ┌────┐  │
│ 感知  │   │ 推理  │   │ 行动  │      │ │感知│ │推理│  │
└──────┘   └──────┘   └──────┘      │ │行动│ │记忆│  │
  进程1      进程2      进程3         │ └────┘ └────┘  │
 可以独立部署/扩容                   └─────────────────┘
                                      只能整体部署/扩容

- 管道: 数据流驱动, 阶段间解耦       - 单体: 控制流驱动, 状态共享
- 管道: 每阶段可独立扩缩容           - 单体: 所有组件绑定在一起
```

### 9.2 vs 事件驱动模式 (Event-Driven)

```
事件驱动模式                         单体模式

         ┌──────────┐
 事件 ──→│ Event Bus │ ──→ Handler1  LLM 输出 → 直接调用工具
         │          │ ──→ Handler2
         │ (Redis/  │ ──→ Handler3  Agent.run(user_input)
         │  Kafka)  │ ──→ HandlerN    ↓
         └──────────┘             思考 → 行动 → 观察
                                    线性顺序, 无事件总线
- 事件驱动: 异步, 松耦合, 可扩展
- 事件驱动: 调试困难 (不知道事件何时被消费)
                                     - 单体: 同步, 紧耦合, 简单
                                     - 单体: 调试方便 (一步接一步)
```

### 9.3 vs 微服务 Agent (Microservice)

```
微服务 Agent                          单体 Agent

┌───────────┐  ┌───────────┐        ┌─────────────────────┐
│ LLM Service│  │Memory     │        │  All-in-One Process │
│(独立部署)   │  │Service    │        │                     │
└─────┬─────┘  └─────┬─────┘        │  LLM + Memory       │
      │               │              │  + Tools + Loop     │
┌─────▼─────┐  ┌─────▼─────┐        │                     │
│Tool Exec  │  │Orchestrator│        │ 单体部署, 单体运行   │
│Service    │  │Service    │        │                     │
└───────────┘  └───────────┘        └─────────────────────┘

每个服务:                          一个进程:
- 独立进程                          - 单一进程
- 独立扩缩容                        - 整体扩缩容
- 独立技术栈                        - 统一技术栈
- 网络通信                          - 函数调用
- 高运维成本                        - 低运维成本
```

### 9.4 vs 网状模式 (Mesh)

```
网状模式 (Multi-Agent Mesh)           单体模式 (Single Agent)

┌──────────┐       ┌──────────┐     ┌────────────────────┐
│  Agent A │◄─────►│  Agent B │     │                    │
│  (专家1)  │       │  (专家2)  │     │   Single Agent    │
└──────────┘       └──────────┘     │   (通用能力)        │
      ▲                 ▲           │                    │
      │   消息路由       │           └────────────────────┘
      ▼                 ▼
┌──────────┐       ┌──────────┐
│  Agent C │◄─────►│  Agent D │
│  (专家3)  │       │  (专家4)  │

- 多 Agent 协作                      - 单 Agent 独立完成
- 复杂通信协议                        - 简单函数调用
- 需要协调器                           - 不需要协调
- 模拟团队协作                         - 模拟个人执行
```

### 对比总结表

| 维度 | 单体 | 管道 | 事件驱动 | 微服务 | 网状 |
|------|------|------|---------|-------|------|
| 进程数 | 1 | N (线性) | N (广播) | N (独立) | N (对等) |
| 状态共享 | 直接内存 | 数据传递 | 事件负载 | 网络调用 | 消息传递 |
| 控制流 | 顺序 | 顺序 | 异步 | 编排 | 协商 |
| 调试 | 极简单 | 简单 | 困难 | 困难 | 极困难 |
| 扩容 | 整体 | 按阶段 | 按处理器 | 按服务 | 按 Agent |
| 最适合 | 原型/小团队 | 确定性流程 | 复杂事件处理 | 中大型团队 | 多专家系统 |
| 复杂度 | 低 | 中 | 中高 | 高 | 极高 |

---

## 10. 核心优势

### 10.1 开发速度极快

单体 Agent 的开发速度优势明显：

```python
# 15 分钟内可启动一个功能完整的单体 Agent

from openai import OpenAI

class QuickMonolithicAgent:
    """极简单体 Agent — 代码量 < 50 行即可运行"""

    def __init__(self, system_prompt: str, tools: list):
        self.client = OpenAI()
        self.system_prompt = system_prompt
        self.tools = tools
        self.messages = [{"role": "system", "content": system_prompt}]

    def run(self, user_input: str) -> str:
        self.messages.append({"role": "user", "content": user_input})

        for _ in range(10):  # 最多 10 步
            response = self.client.chat.completions.create(
                model="gpt-4",
                messages=self.messages,
                tools=self.tools
            )
            msg = response.choices[0].message
            self.messages.append(msg)

            if not msg.tool_calls:
                return msg.content

            for tc in msg.tool_calls:
                result = execute_local_tool(tc.function.name, tc.function.arguments)
                self.messages.append({
                    "role": "tool",
                    "tool_call_id": tc.id,
                    "content": str(result)
                })

        return "Max steps reached"
```

### 10.2 调试极简单

单体架构的调试体验是分布式架构无法比拟的：

| 调试场景 | 单体 | 分布式 |
|---------|------|--------|
| 设置断点 | 全局生效 | 需要每个服务设置 |
| 跟踪变量 | 同一进程空间 | 需要分布式追踪 |
| 复现 Bug | 保存会话即可 | 需要重现多个服务状态 |
| 日志查看 | 一个文件 | 需要日志聚合系统 |
| 性能分析 | profile 单进程 | 需要 APM 工具 |

### 10.3 部署极简单

```bash
# 单体部署
pip install -r requirements.txt
python agent.py

# 分布式部署
docker-compose up -d   # 至少需要 docker-compose.yml
# 或者
kubectl apply -f k8s/   # 需要编写多个 YAML 文件
# 还需要配置:
# - 服务发现 (Consul/K8s Service)
# - 负载均衡 (Nginx/Envoy)
# - 消息队列 (Kafka/RabbitMQ)
# - 分布式存储 (Redis Cluster/S3)
# - CI/CD 管道 (Jenkins/GitLab CI)
```

### 10.4 延迟最低 (无 IPC)

在单体中，工具调用是本地函数调用，延迟为微秒级。在分布式架构中，同样的调用需要网络传输：

```
延迟对比:

本地函数调用:       ~0.001 ms
同机进程间通信:     ~0.1 ms   (Unix Domain Socket)
本地网络调用:       ~0.5 ms   (localhost TCP)
同数据中心网络:     ~1-5 ms   (真实网络)
跨区域网络:        ~50-200 ms (跨地域)

对于 Agent 来说，每次工具调用节省 1-5 ms 的网络延迟
在 20 步的 Agent 循环中，累计节省 20-100 ms
虽然与 LLM 调用的秒级延迟相比微不足道，
但在高频场景下差异显著
```

---

## 11. 工程优化方向

### 11.1 清晰的内部模块边界

即使是在单体中，也应该强制定义清晰的接口边界：

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Protocol

# 定义接口协议 (即使只有一个实现)

class LLMProvider(Protocol):
    """LLM 提供者接口 — 可随时替换实现"""
    async def chat(self, messages: list, tools: list | None = None) -> dict: ...

class MemoryProvider(Protocol):
    """记忆提供者接口 — 可随时替换存储后端"""
    async def add(self, message: dict) -> None: ...
    async def get_context(self, limit: int = 10) -> list[dict]: ...
    async def clear(self) -> None: ...

class ToolProvider(Protocol):
    """工具提供者接口 — 可统一本地和远程工具"""
    @property
    def schemas(self) -> list[dict]: ...
    async def execute(self, name: str, args: dict) -> dict: ...

# 模块化单体：Agent 只依赖接口
class ModularMonolithicAgent:
    def __init__(
        self,
        llm: LLMProvider,
        memory: MemoryProvider,
        tools: ToolProvider,
    ):
        self._llm = llm          # 依赖注入，而非直接实例化
        self._memory = memory
        self._tools = tools
```

### 11.2 配置驱动的工具注册

```python
import json
import yaml
from pathlib import Path
from typing import Callable

class ConfigDrivenToolRegistry:
    """配置驱动的工具注册中心"""

    def __init__(self, config_path: str):
        self.tools: dict[str, dict] = {}
        self.functions: dict[str, Callable] = {}
        self._load_config(config_path)

    def _load_config(self, config_path: str):
        path = Path(config_path)
        if path.suffix in (".yaml", ".yml"):
            config = yaml.safe_load(path.read_text(encoding="utf-8"))
        else:
            config = json.loads(path.read_text(encoding="utf-8"))

        for tool_def in config["tools"]:
            self.register(tool_def)

    def register(self, tool_def: dict):
        """动态注册工具 — 无需修改代码即可添加工具"""
        name = tool_def["name"]
        self.tools[name] = tool_def
        # 动态导入并绑定实现函数
        module_path = tool_def.get("module", f"tools.{name}")
        function_name = tool_def.get("function", "execute")
        module = __import__(module_path, fromlist=[function_name])
        self.functions[name] = getattr(module, function_name)

    def get_schemas(self) -> list[dict]:
        """返回 OpenAI 格式的工具定义"""
        return [
            {
                "type": "function",
                "function": {
                    "name": name,
                    "description": t.get("description", ""),
                    "parameters": t.get("parameters", {}),
                }
            }
            for name, t in self.tools.items()
        ]
```

对应的 YAML 配置文件 `tools.yaml`：

```yaml
tools:
  - name: web_search
    description: "Search the web for current information"
    module: tools.web_search
    function: execute
    parameters:
      type: object
      properties:
        query:
          type: string
          description: "Search query"

  - name: calculator
    description: "Evaluate mathematical expressions"
    module: tools.calculator
    function: calculate
    parameters:
      type: object
      properties:
        expression:
          type: string
          description: "Math expression"
```

### 11.3 插件式内部架构

```python
class PluginBase(ABC):
    """插件基类 — 所有 Agent 功能扩展的基类"""

    @abstractmethod
    def name(self) -> str: ...

    @abstractmethod
    async def on_agent_start(self, context: dict) -> None: ...

    @abstractmethod
    async def on_before_llm_call(self, messages: list) -> list: ...

    @abstractmethod
    async def on_after_llm_call(self, response: dict) -> dict: ...

    @abstractmethod
    async def on_agent_end(self, context: dict) -> None: ...


class PluginManager:
    """插件管理器 — 管理 Agent 生命周期钩子"""

    def __init__(self):
        self.plugins: list[PluginBase] = []

    def load_plugins(self, plugin_paths: list[str]):
        for path in plugin_paths:
            module = __import__(path)
            for attr in dir(module):
                cls = getattr(module, attr)
                if isinstance(cls, type) and issubclass(cls, PluginBase) and cls != PluginBase:
                    self.plugins.append(cls())

    def apply_before_hooks(self, messages: list) -> list:
        for plugin in self.plugins:
            messages = plugin.on_before_llm_call(messages)
        return messages
```

### 11.4 工作线程处理 CPU 密集型工具

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

class WorkerThreadManager:
    """工作线程管理器 — 避免 CPU 密集型工具阻塞 Agent"""

    def __init__(self, max_workers: int = 2):
        self.executor = ProcessPoolExecutor(max_workers=max_workers)

    async def run_in_process(self, func, *args, **kwargs) -> dict:
        """在独立进程中执行 CPU 密集型任务"""
        loop = asyncio.get_running_loop()
        try:
            result = await asyncio.wait_for(
                loop.run_in_executor(self.executor, func, *args, **kwargs),
                timeout=30.0
            )
            return {"success": True, "data": result}
        except asyncio.TimeoutError:
            return {"success": False, "error": "Timeout after 30s"}
        except Exception as e:
            return {"success": False, "error": str(e)}

    async def __aenter__(self):
        return self

    async def __aexit__(self, *args):
        self.executor.shutdown(wait=False)
```

### 11.5 异步 LLM 调用带超时

```python
class ResilientLLMClient:
    """弹性 LLM 客户端 — 超时、重试、降级"""

    def __init__(self, primary: str, fallback: str | None = None):
        self.primary = primary    # 主模型 (如 gpt-4)
        self.fallback = fallback  # 降级模型 (如 gpt-3.5-turbo)

    async def chat_with_timeout(
        self,
        messages: list,
        tools: list | None = None,
        timeout: float = 30.0,
    ) -> dict:
        """带超时的 LLM 调用"""

        try:
            result = await asyncio.wait_for(
                self._call_llm(self.primary, messages, tools),
                timeout=timeout
            )
            return result

        except asyncio.TimeoutError:
            # 超时触发降级
            if self.fallback:
                return await self._call_llm(self.fallback, messages, tools)
            return {"error": "timeout", "role": "assistant", "content": "Request timed out."}

        except RateLimitError:
            # 限流时等待后重试
            await asyncio.sleep(2)
            return await self.chat_with_timeout(messages, tools, timeout)

    async def _call_llm(self, model: str, messages: list, tools: list | None) -> dict:
        # 实际的 LLM API 调用
        ...
```

---

## 12. 适用场景

### 最佳场景

1. **MVP 原型开发**：快速验证 Agent 方案是否可行
   - 示例：2 天搭建一个客服 Agent 原型
   - 推荐：单体 + OpenAI Assistants API

2. **个人效率工具**：满足个人或小团队使用
   - 示例：个人代码审查助手、笔记整理 Agent
   - 推荐：单体 + 本地模型 (如 Ollama)

3. **简单自动化**：替代传统的 if-else 自动化脚本
   - 示例：邮件分类回复、文件批量处理
   - 推荐：单体 + Python 脚本

4. **教学与学习**：理解 Agent 工作原理
   - 示例：学习 ReAct 模式的实现
   - 推荐：单体 + 100 行以内的核心循环

5. **小团队探索**：团队不超过 3 人
   - 示例：初创公司内部工具
   - 推荐：模块化单体

6. **工具数量有限**：集成的工具不超过 10 个
   - 示例：搜索 + 计算 + 文件读写 + API 调用
   - 推荐：单体 + 配置驱动注册

7. **单一模型**：只使用一种 LLM
   - 示例：只使用 GPT-4 或只使用 Claude
   - 推荐：单体 + 直接 API 调用

### 不适合场景

| 不适合的场景 | 原因 | 推荐的替代架构 |
|-------------|------|--------------|
| 高并发服务 (100+ QPS) | 单进程无法承载 | 微服务 Agent |
| 多模型混合编排 | 切换成本高 | 模块化单体/微服务 |
| 跨团队协作开发 | 代码冲突频繁 | 微服务 |
| 不同工具不同 SLA | 无法独立保障 | 微服务/事件驱动 |
| 安全敏感工具 (代码执行) | 进程隔离不足 | 沙箱化微服务 |
| 长时间运行 (7x24) | 内存泄漏风险 | 有状态微服务 |
| 多租户隔离 | 无法隔离租户 | 多实例微服务 |

---

## 13. 代码示例

以下是一个完整的、生产质量的单体 Agent 实现，包含所有核心组件。

### 完整实现

```python
"""
monolithic_agent.py — 完整的单体 Agent 实现

功能:
- LLM 集成 (支持 OpenAI / Anthropic / 本地模型)
- 工具注册与执行
- 完整的 ReAct 循环
- 状态管理 (对话历史 + 中间状态)
- 错误处理与优雅降级
- 超时控制与步数限制

依赖: pip install openai httpx
"""

import json
import logging
import time
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Callable, Optional

import httpx

# ============================================================
# 日志配置
# ============================================================

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(name)s] %(levelname)s: %(message)s"
)
logger = logging.getLogger("MonolithicAgent")


# ============================================================
# 类型定义
# ============================================================

class AgentStatus(Enum):
    """Agent 运行状态"""
    IDLE = "idle"
    THINKING = "thinking"
    EXECUTING = "executing"
    OBSERVING = "observing"
    COMPLETED = "completed"
    ERROR = "error"
    MAX_STEPS_REACHED = "max_steps_reached"


@dataclass
class AgentConfig:
    """Agent 配置"""
    system_prompt: str = "You are a helpful AI assistant with access to tools."
    model: str = "gpt-4o-mini"
    api_base: str = "https://api.openai.com/v1"
    api_key: str = ""
    max_steps: int = 15
    llm_timeout: float = 30.0
    tool_timeout: float = 15.0
    temperature: float = 0.7
    verbose: bool = True


@dataclass
class AgentState:
    """Agent 运行时状态"""
    messages: list[dict] = field(default_factory=list)
    tool_results: dict[str, Any] = field(default_factory=dict)
    current_step: int = 0
    status: AgentStatus = AgentStatus.IDLE
    error: Optional[str] = None
    start_time: float = 0.0
    total_llm_time: float = 0.0
    total_tool_time: float = 0.0


# ============================================================
# 工具定义
# ============================================================

class ToolSpec:
    """工具规格定义"""

    def __init__(
        self,
        name: str,
        description: str,
        parameters: dict,
        function: Callable,
    ):
        self.name = name
        self.description = description
        self.parameters = parameters
        self.function = function

    def to_openai_schema(self) -> dict:
        """转为 OpenAI 工具调用格式"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            }
        }

    def execute(self, **kwargs) -> Any:
        """执行工具函数"""
        logger.info(f"  [Tool] Executing: {self.name}({kwargs})")
        try:
            result = self.function(**kwargs)
            logger.info(f"  [Tool] Result: {str(result)[:200]}")
            return result
        except Exception as e:
            logger.error(f"  [Tool] Error: {self.name}: {e}")
            return {"error": str(e)}


# 内置工具函数

def web_search(query: str) -> str:
    """模拟网络搜索 (实际应调用搜索 API)"""
    return f"Search results for '{query}': [模拟结果] Found 3 relevant articles."


def calculator(expression: str) -> str:
    """安全计算器 — 仅允许数学表达式"""
    allowed = set("0123456789+-*/()., ")
    if not all(c in allowed for c in expression):
        return "Error: Invalid characters in expression"
    try:
        result = eval(expression, {"__builtins__": {}}, {})
        return f"Result: {result}"
    except Exception as e:
        return f"Error: {e}"


def get_current_time(format: str = "%Y-%m-%d %H:%M:%S") -> str:
    """获取当前时间"""
    return time.strftime(format)


# ============================================================
# LLM 客户端
# ============================================================

class LLMClient:
    """LLM API 客户端 (支持 OpenAI API 兼容接口)"""

    def __init__(self, config: AgentConfig):
        self.config = config
        self.client = httpx.AsyncClient(
            base_url=config.api_base,
            headers={
                "Authorization": f"Bearer {config.api_key}",
                "Content-Type": "application/json",
            },
            timeout=httpx.Timeout(config.llm_timeout + 5.0),
        )

    async def chat(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
    ) -> dict:
        """调用 LLM API"""

        payload = {
            "model": self.config.model,
            "messages": messages,
            "temperature": self.config.temperature,
        }

        if tools:
            payload["tools"] = tools

        if self.config.verbose:
            logger.debug(f"  [LLM] Request: model={self.config.model}, messages={len(messages)}")

        try:
            response = await self.client.post("/chat/completions", json=payload)
            response.raise_for_status()
            data = response.json()
            choice = data["choices"][0]["message"]

            if self.config.verbose:
                if "content" in choice and choice["content"]:
                    logger.debug(f"  [LLM] Response content: {choice['content'][:100]}")
                if "tool_calls" in choice:
                    logger.debug(f"  [LLM] Tool calls: {len(choice['tool_calls'])}")

            return choice

        except httpx.TimeoutException:
            logger.warning("  [LLM] Request timed out")
            return {
                "role": "assistant",
                "content": json.dumps({"error": "LLM request timed out"})
            }
        except Exception as e:
            logger.error(f"  [LLM] Request failed: {e}")
            return {
                "role": "assistant",
                "content": json.dumps({"error": f"LLM request failed: {e}"})
            }

    async def close(self):
        await self.client.aclose()


# ============================================================
# 核心 Agent 实现
# ============================================================

class MonolithicAgent:
    """
    单体 Agent — 完整实现

    所有组件在单一进程中运行:
    - LLM 客户端
    - 工具注册表
    - 状态管理
    - Agent 循环
    - 错误处理
    """

    def __init__(self, config: AgentConfig):
        self.config = config
        self.state = AgentState()
        self.llm = LLMClient(config)
        self.tools: dict[str, ToolSpec] = {}
        self.state.status = AgentStatus.IDLE
        logger.info(f"MonolithicAgent initialized (model={config.model}, max_steps={config.max_steps})")

    # ----------------------------------------------------------
    # 工具管理
    # ----------------------------------------------------------

    def register_tool(self, tool: ToolSpec):
        """注册一个工具"""
        self.tools[tool.name] = tool
        logger.info(f"  Registered tool: {tool.name}")

    def register_tools(self, *tools: ToolSpec):
        """批量注册工具"""
        for tool in tools:
            self.register_tool(tool)

    def get_tool_schemas(self) -> list[dict]:
        """获取所有工具的 OpenAI 格式定义"""
        return [t.to_openai_schema() for t in self.tools.values()]

    # ----------------------------------------------------------
    # 核心循环
    # ----------------------------------------------------------

    async def run(self, user_input: str) -> str:
        """
        运行 Agent 的主入口

        执行流程:
        1. 将用户输入加入消息列表
        2. 循环: LLM 思考 → 执行工具 → 观察结果
        3. 直到 LLM 返回最终答案或达到最大步数
        """
        self.state = AgentState(
            messages=[
                {"role": "system", "content": self.config.system_prompt},
                {"role": "user", "content": user_input},
            ],
            status=AgentStatus.THINKING,
            start_time=time.time(),
        )

        logger.info(f"Agent run started (input: {user_input[:80]})")

        try:
            final_answer = await self._run_loop()
            self.state.status = AgentStatus.COMPLETED
            elapsed = time.time() - self.state.start_time
            logger.info(f"Agent run completed in {elapsed:.2f}s ({self.state.current_step} steps)")
            return final_answer

        except Exception as e:
            self.state.status = AgentStatus.ERROR
            self.state.error = str(e)
            logger.error(f"Agent run failed: {e}")
            return f"Error: {e}"

    async def _run_loop(self) -> str:
        """Agent 主循环"""

        tools = self.get_tool_schemas() if self.tools else None

        while self.state.current_step < self.config.max_steps:
            self.state.current_step += 1
            self.state.status = AgentStatus.THINKING

            if self.config.verbose:
                logger.info(f"\n{'='*50}")
                logger.info(f"Step {self.state.current_step}/{self.config.max_steps}")

            # ---- 1. LLM 思考 ----
            t0 = time.time()
            response = await self.llm.chat(self.state.messages, tools)
            self.state.total_llm_time += time.time() - t0

            # 处理 LLM 错误
            if "error" in response.get("content", ""):
                self.state.messages.append({
                    "role": "assistant",
                    "content": response["content"]
                })
                return response["content"]

            # 将 LLM 回复加入消息列表
            self.state.messages.append(response)

            # ---- 2. 检查是否完成 ----
            if not response.get("tool_calls"):
                # LLM 没有请求工具调用 → 返回最终答案
                final_content = response.get("content", "")
                if self.config.verbose:
                    logger.info(f"  [Final] {final_content[:200]}")
                return final_content

            # ---- 3. 执行工具 ----
            self.state.status = AgentStatus.EXECUTING

            for tool_call in response["tool_calls"]:
                func_name = tool_call["function"]["name"]
                func_args_str = tool_call["function"]["arguments"]

                # 解析参数
                try:
                    func_args = json.loads(func_args_str) if isinstance(func_args_str, str) else func_args_str
                except json.JSONDecodeError:
                    func_args = {}

                # 查找工具
                tool = self.tools.get(func_name)
                if not tool:
                    result = {"error": f"Unknown tool: {func_name}"}
                else:
                    # 执行工具
                    t0 = time.time()
                    result = tool.execute(**func_args)
                    self.state.total_tool_time += time.time() - t0

                # 将工具结果加入消息列表
                self.state.messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call["id"],
                    "content": json.dumps(result, ensure_ascii=False),
                })

                # 缓存结果
                self.state.tool_results[func_name] = result

                self.state.status = AgentStatus.OBSERVING

        # ---- 达到最大步数 ----
        self.state.status = AgentStatus.MAX_STEPS_REACHED
        return f"Reached maximum steps ({self.config.max_steps}). Here is the current state."

    # ----------------------------------------------------------
    # 状态管理
    # ----------------------------------------------------------

    def get_conversation_history(self) -> list[dict]:
        """获取完整的对话历史"""
        return self.state.messages

    def clear_conversation(self):
        """清空对话历史"""
        self.state.messages = []
        self.state.tool_results = {}
        logger.info("Conversation cleared")

    def get_stats(self) -> dict:
        """获取运行统计"""
        return {
            "steps": self.state.current_step,
            "status": self.state.status.value,
            "total_llm_time": round(self.state.total_llm_time, 2),
            "total_tool_time": round(self.state.total_tool_time, 2),
            "total_time": round(time.time() - self.state.start_time, 2) if self.state.start_time else 0,
            "messages_count": len(self.state.messages),
            "tools_count": len(self.tools),
        }

    # ----------------------------------------------------------
    # 生命周期
    # ----------------------------------------------------------

    async def shutdown(self):
        """优雅关闭 Agent"""
        logger.info("Shutting down MonolithicAgent...")
        await self.llm.close()
        self.state.status = AgentStatus.IDLE
        logger.info("Shutdown complete.")


# ============================================================
# 使用示例
# ============================================================

async def main():
    """演示如何使用 MonolithicAgent"""

    # 配置
    config = AgentConfig(
        system_prompt=(
            "You are a helpful assistant. You have access to tools to help users. "
            "Use tools when needed, and respond in Chinese."
        ),
        model="gpt-4o-mini",
        api_key="your-api-key-here",  # 替换为实际 API Key
        max_steps=10,
        verbose=True,
    )

    # 创建 Agent
    agent = MonolithicAgent(config)

    # 注册工具
    agent.register_tools(
        ToolSpec(
            name="web_search",
            description="Search the web for information",
            parameters={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "The search query"
                    }
                },
                "required": ["query"]
            },
            function=web_search,
        ),
        ToolSpec(
            name="calculator",
            description="Evaluate a mathematical expression",
            parameters={
                "type": "object",
                "properties": {
                    "expression": {
                        "type": "string",
                        "description": "The mathematical expression to evaluate"
                    }
                },
                "required": ["expression"]
            },
            function=calculator,
        ),
        ToolSpec(
            name="get_current_time",
            description="Get the current date and time",
            parameters={
                "type": "object",
                "properties": {
                    "format": {
                        "type": "string",
                        "description": "Time format string",
                        "default": "%Y-%m-%d %H:%M:%S"
                    }
                }
            },
            function=get_current_time,
        ),
    )

    # 运行 Agent
    try:
        result = await agent.run("What time is it now? Also calculate 123 * 456.")
        print(f"\n{'='*50}")
        print(f"Final Answer: {result}")
        print(f"Stats: {agent.get_stats()}")
    finally:
        await agent.shutdown()


if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### 代码结构说明

```
MonolithicAgent 代码结构:

monolithic_agent.py
│
├── AgentConfig          ← 配置管理 (模型、超时、步数限制)
├── AgentState           ← 状态管理 (消息、步骤计数、统计)
├── AgentStatus(Enum)    ← 状态枚举 (生命周期)
│
├── ToolSpec             ← 工具规格定义
│   ├── to_openai_schema()  ← 转为 OpenAI 格式
│   └── execute()           ← 执行工具函数
│
├── LLMClient            ← LLM API 客户端
│   └── chat()              ← 带超时和错误处理的 API 调用
│
└── MonolithicAgent      ← 核心 Agent (主类)
    ├── register_tool()      ← 注册工具
    ├── run()                ← 主入口
    ├── _run_loop()          ← Agent 循环 (核心)
    ├── get_conversation_history()
    ├── clear_conversation()
    ├── get_stats()
    └── shutdown()
```

---

## 14. 生产级架构图

### 14.1 内部组件架构

```
┌══════════════════════════════════════════════════════════════┐
║                  单体 Agent 生产架构                           ║
║                   (Monolithic Agent)                         ║
║                                                              ║
║  ┌────────────────────────────────────────────────────────┐  ║
║  │                   用户接口层 (Interface)                  │  ║
║  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐      │  ║
║  │  │  CLI     │  │  HTTP    │  │  WebSocket       │      │  ║
║  │  │  Interface│  │  API     │  │  (实时流)         │      │  ║
║  │  └────┬─────┘  └────┬─────┘  └────────┬─────────┘      │  ║
║  └───────┼─────────────┼─────────────────┼────────────────┘  ║
║          │             │                 │                    ║
║  ┌───────▼─────────────▼─────────────────▼────────────────┐  ║
║  │                编排层 (Orchestration)                    │  ║
║  │                                                         │  ║
║  │  ┌─────────────────────────────────────────────────┐   │  ║
║  │  │           Agent 主循环 (Agent Loop)               │   │  ║
║  │  │                                                 │   │  ║
║  │  │   ┌──────────┐   ┌──────────┐   ┌──────────┐   │   │  ║
║  │  │   │  Thought  │ → │  Action  │ → │  Observe │   │   │  ║
║  │  │   │  (LLM)   │   │  (Tool)  │   │ (Result) │   │   │  ║
║  │  │   └──────────┘   └──────────┘   └──────────┘   │   │  ║
║  │  │         ↑              │              │         │   │  ║
║  │  │         └──────────────┴──────────────┘         │   │  ║
║  │  │                 循环直到 Final Answer            │   │  ║
║  │  └─────────────────────────────────────────────────┘   │  ║
║  └────────────────────────────────────────────────────────┘  ║
║                              │                                ║
║  ┌────────────────────────────────────────────────────────┐  ║
║  │                   核心服务层 (Core Services)              │  ║
║  │                                                         │  ║
║  │  ┌────────────┐  ┌────────────┐  ┌──────────────────┐  │  ║
║  │  │  LLM       │  │  工具注册表  │  │  记忆管理器      │  │  ║
║  │  │  客户端    │  │  Registry  │  │  Memory         │  │  ║
║  │  │           │  │            │  │                  │  │  ║
║  │  │  ┌─────┐  │  │  ┌──────┐  │  │  ┌────────────┐  │  │  ║
║  │  │  │OpenAI│  │  │  │Tool1 │  │  │  │对话历史     │  │  │  ║
║  │  │  ├─────┤  │  │  ├──────┤  │  │  ├────────────┤  │  │  ║
║  │  │  │Claude│  │  │  │Tool2 │  │  │  │向量嵌入     │  │  │  ║
║  │  │  ├─────┤  │  │  ├──────┤  │  │  ├────────────┤  │  │  ║
║  │  │  │Local│  │  │  │ToolN │  │  │  │摘要缓存     │  │  │  ║
║  │  │  └─────┘  │  │  └──────┘  │  │  └────────────┘  │  │  ║
║  │  └────────────┘  └────────────┘  └──────────────────┘  │  ║
║  └────────────────────────────────────────────────────────┘  ║
║                              │                                ║
║  ┌────────────────────────────────────────────────────────┐  ║
║  │                    基础设施层 (Infrastructure)            │  ║
║  │                                                         │  ║
║  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │  ║
║  │  │ 日志     │  │ 监控     │  │ 配置管理  │  │错误处理 │  │  ║
║  │  │ Logger  │  │ Metrics │  │ Config  │  │ Handler│  │  ║
║  │  └──────────┘  └──────────┘  └──────────┘  └────────┘  │  ║
║  │                                                         │  ║
║  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │  ║
║  │  │ 熔断器   │  │ 重试机制  │  │ 工作线程池           │  │  ║
║  │  │ Breaker │  │ Retry   │  │ Worker Pool         │  │  ║
║  │  └──────────┘  └──────────┘  └──────────────────────┘  │  ║
║  └────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════╝
```

### 14.2 请求生命周期时序图

```
用户                        Agent 循环                     LLM API              工具函数
 │                            │                            │                    │
 │  用户输入                   │                            │                    │
 │───────────────────────────►│                            │                    │
 │                            │                            │                    │
 │                            │  Step 1: 构建 messages      │                    │
 │                            │─────────────────────────────│                    │
 │                            │─────────────────────────────┤                    │
 │                            │  Step 2: LLM 调用           │                    │
 │                            │────────────────────────────►│                    │
 │                            │                            │  LLM 返回思考       │
 │                            │◄────────────────────────────│                    │
 │                            │                            │                    │
 │                            │  Step 3: 解析响应            │                    │
 │                            │  tool_calls? ── 否 ──→ 返回最终答案              │
 │                            │    │                                              │
 │                            │    │ 是                                           │
 │                            │    ▼                                              │
 │                            │  Step 4: 执行工具                                  │
 │                            │─────────────────────────────────────────────────►│
 │                            │                工具执行结果                       │
 │                            │◄─────────────────────────────────────────────────│
 │                            │                                                    │
 │                            │  Step 5: 工具结果加入 messages                     │
 │                            │                                                    │
 │                            │  ──→ 回到 Step 2 (下一轮迭代)                      │
 │                            │                                                    │
 │  Final Answer              │                                                    │
 │◄───────────────────────────│                                                    │
```

---

## 15. 演进路径

### 从单体到分布式: 优雅的迁移策略

当单体 Agent 达到能力边界时，不要重写，而要逐步提取。以下是一个经过验证的演进路径。

### 阶段一: 模块化单体 (0→1 月)

不改变架构，先做好内部解耦：

```
单体 (Monolith)         →    模块化单体 (Modular Monolith)
┌──────────────────┐        ┌──────────────────────────┐
│  LLM 直接调用     │        │  LLMProvider (接口)       │
│  工具散落各处     │        │  MemoryProvider (接口)    │
│  状态全局变量     │   →    │  ToolProvider (接口)      │
│  无错误处理       │        │  熔断器 + 重试            │
│  紧耦合           │        │  依赖注入                  │
└──────────────────┘        └──────────────────────────┘

目标: 每一层都有清晰的接口，为拆分做准备
风险: 低 (不改架构，只改代码组织)
收益: 模块可测试、可替换
```

### 阶段二: 提取独立进程 (1→2 月)

将最容易成为瓶颈的组件提取为独立进程：

```
模块化单体             →    部分提取
┌──────────────────┐        ┌────────────┐  ┌────────────┐
│  Agent Loop      │        │ Agent Loop │  │ Memory     │
│  LLMProvider     │        │ (主进程)    │  │ Service    │
│  MemoryProvider  │   →    │            │  │ (独立进程)  │
│  ToolProvider    │        │ LLM调用    │  └────────────┘
│  核心逻辑        │        │ 工具执行    │        ↑
└──────────────────┘        │ 核心逻辑    │        │ gRPC/HTTP
                            └────────────┘        │
                                    │        ┌────────────┐
                                    │        │ Tool Exec  │
                                    └───────►│  Service   │
                                             │ (沙箱进程)  │
                                             └────────────┘

提取优先级:
1. 内存记忆 → 独立 Memory Service (最易提取, 收益最大)
2. 沙箱工具执行 → 独立 Tool Execution Service (安全隔离)
3. 向量搜索 → 独立 Embedding Service (计算密集型)
```

### 阶段三: 完全微服务化 (2→3 月)

```
┌─────────────┐     ┌──────────────┐     ┌────────────────┐
│  API Gateway │ ──► │  Orchestrator│ ──► │  LLM Router    │
└─────────────┘     │  Service     │     │  Service       │
                    └──────────────┘     └────────────────┘
                           │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
     ┌────────────┐ ┌──────────┐ ┌──────────┐
     │  Memory    │ │  Tool    │ │  Context  │
     │  Service   │ │  Service │ │  Service  │
     └────────────┘ └──────────┘ └──────────┘
```

### 迁移检查清单

```
□ [ ] 阶段一: 模块化单体
   □ 定义 LLMProvider 接口
   □ 定义 MemoryProvider 接口
   □ 定义 ToolProvider 接口
   □ 实现依赖注入 (不再直接 new)
   □ 添加熔断器/重试/超时
   □ 单元测试覆盖率达到 80%

□ [ ] 阶段二: 部分提取
   □ 提取 Memory 为独立服务 (HTTP/gRPC)
   □ 提取高风险工具为独立沙箱进程
   □ 添加健康检查端点
   □ 添加分布式追踪 (OpenTelemetry)
   □ 集成测试通过

□ [ ] 阶段三: 完全微服务
   □ 拆分 Orchestrator 服务
   □ 添加消息队列 (异步解耦)
   □ 配置 Kubernetes 部署
   □ 添加服务网格 (Istio/Linkerd)
   □ 性能测试达标
```

### 关键原则

1. **不要提前拆分**：只有在明确感受到单体痛点后才启动迁移
2. **一次只提取一个组件**：每次提取后稳定运行一段时间
3. **保持 API 兼容**：提取后内部 API 与之前单体接口保持一致
4. **用 Strangler Fig 模式**：新功能用新架构实现，旧功能保持单体，逐步替换

---

## 总结

| 维度 | 评价 |
|------|------|
| **核心理念** | 所有组件在单一进程中运行，简单直接 |
| **最大优势** | 开发速度快、调试简单、部署容易、延迟最低 |
| **最大劣势** | 无法独立扩缩容、资源争用、技术锁定 |
| **适用团队** | 1-3 人小团队 |
| **适用阶段** | MVP 原型、个人工具、教学演示 |
| **适用规模** | 工具 < 10 个、单模型、低并发 |
| **迁移路径** | 模块化单体 → 部分提取 → 完全微服务 |
| **代表项目** | OpenAI Assistants API, 简单的 LangGraph Agent, AutoGPT 早期版本 |

### 一句话总结

> **单体 Agent 是最简单的 Agent 架构——用 100 行代码就能运行，但当功能膨胀到需要 5000 行时，就是该拆分的时候了。**

在实际项目中，建议**从单体开始，保持模块化，在感受到明确的痛点后再逐步拆分**。大多数 Agent 应用永远不需要超越单体——不是每个项目都需要成为微服务。
