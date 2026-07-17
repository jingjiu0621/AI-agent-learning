# 8.6.3 AutoGen 对话驱动模式源码分析

## 1. 简单介绍

AutoGen (Microsoft, 2023) 是微软推出的多 Agent 对话框架，其核心设计哲学是 **"对话即协调"**(Conversation as Coordination)。与 CrewAI 的 Process-based 显式编排不同，AutoGen 不预设任何任务管线或执行顺序——Agent 之间通过自由对话协商下一步该做什么。

**对话驱动模式**的核心洞察是：LLM 本质上是对话模型，最自然的 Agent 交互方式就是让 Agent 像人类团队一样互相发送消息。每个 Agent 有自己的角色和能力，收到消息后自主决定如何回应——是调用工具、执行代码、还是向另一个 Agent 提问。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AutoGen 对话驱动架构总览                          │
│                                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │ AssistantAgent│    │ UserProxyAgent│   │ CustomAgent │   ...     │
│  │  (LLM 驱动)  │    │ (人/代码执行)│    │ (扩展)      │           │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘           │
│         │                   │                   │                    │
│         └───────────────────┼───────────────────┘                    │
│                             │                                       │
│                    ┌────────▼────────┐                              │
│                    │   ConversableAgent                              │
│                    │   (基类: send/receive)                          │
│                    └────────┬────────┘                              │
│                             │                                       │
│                    ┌────────▼────────┐                              │
│                    │   Message Protocol                              │
│                    │   {content, role, name,                         │
│                    │    function_call}                               │
│                    └─────────────────┘                              │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                GroupChat / GroupChatManager                  │    │
│  │  多 Agent 群体对话: 轮次选择 → 广播 → 摘要                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

**与 CrewAI 的对比**：

| 维度 | AutoGen | CrewAI |
|------|---------|--------|
| 协调方式 | 对话协商 (chat) | 流程编排 (process) |
| 任务拓扑 | 动态图 (dynamic graph) | 静态 DAG (sequential/hierarchical) |
| 控制流 | 隐式 (Agent 自主决定) | 显式 (Manager 分配) |
| 扩展性 | 加入新 Agent 即插即聊 | 需要修改 Process 定义 |
| 调试难度 | 较高 (对话路径不确定) | 中等 (流程可预测) |

---

## 2. 核心架构

AutoGen 的核心类层次结构如下：

```
autogen
├── ConversableAgent          ← 所有 Agent 的基类
│   ├── AssistantAgent        ← LLM 驱动的 Agent (发消息、调函数)
│   └── UserProxyAgent        ← 人类代理 / 代码执行 Agent
├── GroupChat                 ← 群聊配置 (speaker 列表、选择策略)
├── GroupChatManager          ← 群聊管理器 (维护对话轮次)
├── Agent (Protocol)          ← Agent 接口协议 (鸭子类型)
└── ChatCompletion            ← LLM 调用封装 (流式 + 非流式)
```

### 2.1 ConversableAgent (基类)

`ConversableAgent` 是所有 Agent 的基类，定义了 Agent 通信的核心协议。它不仅是"可以对话的 Agent"，还包含了消息收发、自动回复链、函数注册等完整基础设施。

```
┌────────────────────────────────────────────────────────────┐
│                    ConversableAgent                        │
├────────────────────────────────────────────────────────────┤
│  Fields:                                                   │
│    name: str                                               │
│    system_message: str                                     │
│    llm_config: dict                                        │   LLM 配置
│    chat_messages: Dict[Agent, List[Dict]]                  │   按对话方维护的历史
│    _reply_func_list: List[Tuple[Callable, ...]]            │   自动回复函数链
│    _function_map: Dict[str, Callable]                      │   注册的工具函数
│    human_input_mode: str                                   │   NEVER/ALWAYS/TERMINATE
│    max_consecutive_auto_reply: int                         │
│    code_execution_config: dict                             │
├────────────────────────────────────────────────────────────┤
│  Core Methods:                                             │
│    send(msg, recipient, request_reply=True)                │   发送消息给指定 Agent
│    receive(msg, sender, request_reply=True)                │   接收消息 (触发自动回复)
│    generate_reply(messages, sender, **kwargs)              │   生成回复 (核心)
│    a_initiate_chat(recipient, message, ...)                   │   发起异步对话
│    initiate_chat(recipient, message, ...)                  │   发起同步对话
│    register_reply(reply_func, ...)                         │   注册回复生成函数
│    register_function(func_map)                             │   注册可调用工具函数
│    _append_oai_message(msg, ...)                           │   追加消息到历史
│    format_function_map(...)                                │   格式化函数描述给 LLM
├────────────────────────────────────────────────────────────┤
│  Auto-Reply Chain:                                         │
│    generate_reply() → 遍历 _reply_func_list                │
│    ┌─────────┐   ┌───────────┐   ┌──────────────┐        │
│    │ check    │ → │ generate_ │ → │ generate_    │        │
│    │ termination │ │ function_  │   │ oai_reply   │        │
│    │          │   │ call_reply │   │ (LLM call)  │        │
│    └─────────┘   └───────────┘   └──────────────┘        │
└────────────────────────────────────────────────────────────┘
```

**核心设计决策**：

1. **消息按对话方隔离**: `chat_messages` 是一个 `Dict[Agent, List[Dict]]`，不同对话方的消息历史彼此独立。这意味着同一个 Agent 可以同时参与多个对话而不会混淆上下文。

2. **自动回复链 (Reply Chain)**: `generate_reply()` 按优先级遍历注册的回复函数，第一个返回非 `None` 结果的函数胜出。典型链是: 检查终止条件 → 执行函数调用 → 调用 LLM 生成回复。

3. **send/receive 对称性**: `send()` 和 `receive()` 是配对操作。`send()` 调用 `recipient.receive()`，而 `receive()` 内部调用 `generate_reply()` 并将回复 `send()` 回去。这种对称设计使得 Agent 对间的通信是完全对等的。

### 2.2 AssistantAgent

`AssistantAgent` 是 ConversableAgent 最直接、最常见的子类。它的核心是注册了一个 `generate_oai_reply` 函数到回复链中，使得每次收到消息时自动调用 LLM。

```
┌────────────────────────────────────────────────────────────┐
│                    AssistantAgent                           │
├────────────────────────────────────────────────────────────┤
│  继承 ConversableAgent                                     │
│                                                             │
│  构造函数中自动注册:                                        │
│    _reply_func_list = [                                     │
│      (check_termination,    ...),  ← 检查是否应终止        │
│      (generate_function_call, ...), ← 检查 LLM 是否要调函数 │
│      (generate_oai_reply,   ...),  ← 调 LLM API 生成回复   │
│    ]                                                        │
│                                                             │
│  generate_oai_reply():                                     │
│    messages = self.chat_messages[sender]   ← 取历史         │
│    response = oai.ChatCompletion.create(   ← 调 LLM        │
│        messages=messages,                                   │
│        **self.llm_config                                    │
│    )                                                        │
│    reply = extract_text_or_function_call(response)          │
│    return reply                                             │
└────────────────────────────────────────────────────────────┘
```

关键点：
- AssistantAgent **不执行代码**，它只生成文本回复或函数调用请求。
- `generate_oai_reply` 是回复链的最后一环，前面的函数检查环节都不满足条件时才轮到它。
- LLM 调用支持流式 (`stream=True`) 和非流式两种模式。

### 2.3 UserProxyAgent

`UserProxyAgent` 是另一个关键子类，代表"人类用户"或"代码执行能力"。

```
┌────────────────────────────────────────────────────────────┐
│                    UserProxyAgent                           │
├────────────────────────────────────────────────────────────┤
│  继承 ConversableAgent                                     │
│                                                             │
│  额外能力:                                                  │
│    - 代码执行 (Python / Shell / Docker)                    │
│    - 人工输入 (human_input_mode)                            │
│    - 函数调用执行 (execute_function)                        │
│                                                             │
│  构造函数中注册:                                            │
│    _reply_func_list = [                                     │
│      (check_termination,   ...),                            │
│      (generate_human_input_reply, ...), ← 请求人工输入     │
│      (execute_function,    ...),  ← 执行 LLM 请求的函数    │
│      (execute_code_reply,  ...),  ← 执行代码块             │
│      (generate_code_execution_reply, ...),  ← 执行结果处理  │
│    ]                                                        │
│                                                             │
│  human_input_mode:                                          │
│    "NEVER"      → 全自动 (不询问用户)                       │
│    "ALWAYS"     → 每次回复前都询问用户                      │
│    "TERMINATE"  → 仅在对话终止时询问用户                    │
└────────────────────────────────────────────────────────────┘
```

`human_input_mode` 是 AutoGen 的一个重要设计——它允许 Agent 系统在**全自动**和**人工介入**之间平滑切换。这在需要人类监督的场景（如代码执行确认）中非常关键。

### 2.4 GroupChat 和 GroupChatManager

GroupChat 和 GroupChatManager 一起实现了多 Agent 群聊模式。

```
┌────────────────────────────────────────────────────────────┐
│  GroupChat  vs  GroupChatManager                            │
│                                                             │
│  ┌──────────────────────┐   ┌──────────────────────────┐   │
│  │     GroupChat        │   │    GroupChatManager       │   │
│  │  (数据/配置)         │   │  (逻辑/控制)             │   │
│  ├──────────────────────┤   ├──────────────────────────┤   │
│  │ agents: List[Agent]  │   │ 继承 UserProxyAgent       │   │
│  │ messages: List[Dict] │   │                            │   │
│  │ max_round: int       │   │ 核心方法:                  │   │
│  │ admin_name: str      │   │  run_chat()                │   │
│  │ speaker_selection_   │   │   ├─ select_speaker()     │   │
│  │   method: str/callable│  │   ├─ speaker.send()       │   │
│  │ func: callable       │   │   ├─ broadcast()          │   │
│  │                      │   │   └─ check_termination()  │   │
│  │ agent_names: set     │   │                            │   │
│  │ reset()              │   │ 作为 UserProxyAgent:      │   │
│  │ append()             │   │  可执行代码/可调用函数    │   │
│  │ search_serializable()│   │                            │   │
│  └──────────────────────┘   └──────────────────────────┘   │
│                                                             │
│  关系: GroupChatManager 内部持有 GroupChat 实例              │
│       GroupChatManager 作为 UserProxyAgent 参与对话         │
└────────────────────────────────────────────────────────────┘
```

设计意图：
- **GroupChat 是纯数据模型**：存储 Agent 列表、消息历史、对话轮次上限。
- **GroupChatManager 是控制逻辑**：它本身是一个 UserProxyAgent，拥有代码执行能力；它的工作是决定 "下一个谁说话"。

---

## 3. 对话机制 (Message Protocol & Flow)

### 3.1 消息格式

AutoGen 的消息格式直接继承自 OpenAI ChatCompletion 的 message 格式，但进行了扩展：

```python
# 标准消息格式
{
    "content": "Hello, how can I help you?",      # 文本内容 (str 或 None)
    "role": "assistant",                           # user / assistant / function
    "name": "Assistant_1",                         # 发送方 Agent 名称 (可选)
    "function_call": {                             # 函数调用请求 (由 LLM 生成)
        "name": "get_weather",
        "arguments": "{\"location\": \"Beijing\"}"
    },
}

# 函数执行结果消息
{
    "content": "The weather in Beijing is sunny.",
    "role": "function",
    "name": "get_weather",                         # 被调用的函数名
}

# 终止消息 (触发对话结束)
{
    "content": "TERMINATE",
    "role": "assistant",
    "name": "Assistant_1",
}
```

### 3.2 两 Agent 对话流程 (Assistant ↔ UserProxy)

这是 AutoGen 最基础的交互模式——一个 AssistantAgent 和一个 UserProxyAgent 来回对话，形成"思考-行动-观察"循环。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Assistant ↔ UserProxy 对话流程                        │
│                                                                         │
│  AssistantAgent                         UserProxyAgent                  │
│  (LLM Brain)                            (Hands & Eyes)                 │
│                                                                         │
│      │                                       │                          │
│      │  initiate_chat("Write a script...")   │                          │
│      │──────────────────────────────────────►│                          │
│      │                                       │                          │
│      │  receive(msg)                         │                          │
│      │    │                                  │                          │
│      │    ├─ generate_reply()                │                          │
│      │    │  ├─ (termination check → skip)   │                          │
│      │    │  ├─ (function call → skip)       │                          │
│      │    │  └─ generate_oai_reply()         │                          │
│      │    │      └─ LLM.generate() ←──┐      │                          │
│      │    │                            │      │                          │
│      │    └─ reply = "Let me write    │      │                          │
│      │         a Python script..."    │      │                          │
│      │                                       │                          │
│      │  send(reply)                          │                          │
│      │──────────────────────────────────────►│                          │
│      │                                       │  receive(msg)            │
│      │                                       │    │                     │
│      │                                       │    ├─ generate_reply()   │
│      │                                       │    │  ├─ term check      │
│      │                                       │    │  ├─ human input     │
│      │                                       │    │  ├─ execute_code()  │
│      │                                       │    │  │  └─ run Python   │
│      │                                       │    │  └─ generate_reply()│
│      │                                       │    └─ reply = result    │
│      │                                       │                          │
│      │  receive(result)                      │                          │
│      │◄──────────────────────────────────────│  send(result)            │
│      │    │                                  │                          │
│      │    ├─ "Error: indentation..."         │         (循环继续...)     │
│      │    └─ LLM.generate(try fix)           │                          │
│      │                                       │                          │
│      │  ... 循环直到 TERMINATE 或 max_round...│                          │
│      │                                       │                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**逐步说明**：

1. **发起**: `initiate_chat()` 发送初始消息给 UserProxyAgent。
2. **Assistant 回复**: AssistantAgent 调用 LLM 生成回复，通常包含代码建议。
3. **UserProxy 执行**: 收到消息 → 检测到代码块 → 执行 Python 代码 → 返回结果。
4. **Assistant 迭代**: 看到执行结果（可能成功或报错）→ LLM 决定是修正还是给出下一步。
5. **终止**: 当某条消息包含 `"TERMINATE"` 字符串时停止。

### 3.3 send/receive 源码级的调用链

```python
# 核心调用链 (伪码)
class ConversableAgent:
    def send(self, message, recipient, request_reply=True):
        # 1. 记录发出消息到自己的历史
        self._append_oai_message(message, self, recipient)
        
        # 2. 调用接收方的 receive
        if request_reply:
            recipient.receive(message, self)
    
    def receive(self, message, sender, request_reply=True):
        # 1. 记录收到消息到历史
        self._append_oai_message(message, sender, self)
        
        # 2. 如果需要回复，生成并发送回复
        if request_reply:
            reply = self.generate_reply(
                messages=self.chat_messages[sender],
                sender=sender
            )
            if reply is not None:
                self.send(reply, sender, request_reply=False)
                # request_reply=False 防止无限递归！
    
    def generate_reply(self, messages, sender, **kwargs):
        # 遍历 _reply_func_list，第一个非 None 返回就是回复
        for reply_func in self._reply_func_list:
            final, reply = reply_func(self, messages=sent, sender=sender, **kwargs)
            if reply is not None:
                return reply
        return None
```

这个设计最精巧的地方是 **`request_reply=False`** 防止了无限递归：A.send(B) → B.receive(A) → B.generate_reply() → B.send(A, request_reply=False) → A.receive(B, request_reply=False) → A 不 generate_reply。

---

## 4. AssistantAgent 内部

### 4.1 消息处理流水线

```
AssistantAgent.generate_reply()
    │
    ├── 0. check_termination()
    │      检查消息中是否包含 "TERMINATE" 或自定义终止词
    │      是 → 返回 None (停止继续回复)
    │
    ├── 1. generate_function_call_reply()
    │      检查 LLM 返回中是否包含 function_call
    │      是 → 提取 {"name": ..., "arguments": ...} 结构
    │          → 返回给 UserProxy 执行
    │          → UserProxy 执行后把结果注入对话
    │
    ├── 2. generate_oai_reply()  ← 大多数情况落在这里
    │      ├─ 构造 messages 列表:
    │      │   [
    │      │     {"role": "system", "content": system_message},
    │      │     ...chat_history...,
    │      │     {"role": "user", "content": sender_latest_message}
    │      │   ]
    │      │
    │      ├─ 注入 function_description:
    │      │   如果有注册的函数，将它们转换为 OpenAI tools 格式
    │      │   注入到 llm_config 的 tools 字段
    │      │
    │      ├─ oai.ChatCompletion.create(
    │      │     messages=messages,
    │      │     **llm_config  # model, tools, temperature, ...
    │      │   )
    │      │
    │      └─ 解析 response:
    │           如果 response 包含 function_call → 走 generate_function_call_reply
    │           如果 response 是 text → 返回 text
    │           如果 stream=True → 累积 chunks → 返回完整内容
    │
    └── 3. (如果 None) → return None
```

### 4.2 对话历史与上下文窗口管理

```python
# 关键问题: LLM 上下文窗口有限
# 当 chat_messages 积累过多时怎么办？

# AutoGen 的做法: 直接截断（不保留！）
# 这是一个已知的局限性
def _trim_messages(self, messages, max_tokens=None):
    """简单地从旧消息开始丢弃"""
    if max_tokens is None:
        max_tokens = self.llm_config.get("max_tokens", 4096)
    
    total = sum(count_tokens(m) for m in messages)
    while total > max_tokens and len(messages) > 1:
        messages.pop(0)          # 丢弃最早的消息
        total = sum(...)          # 重新计算
    return messages
```

### 4.3 流式输出

```python
# AssistantAgent 支持 OpenAI 流式输出
def generate_oai_reply(self, messages, sender, **kwargs):
    if self.llm_config.get("stream", False):
        response = ""
        for chunk in oai.ChatCompletion.create(
            messages=messages,
            stream=True,
            **self.llm_config
        ):
            if "content" in chunk["choices"][0]["delta"]:
                content = chunk["choices"][0]["delta"]["content"]
                response += content
                # 实时输出
                print(content, end="", flush=True)
        return response
    else:
        # 非流式: 一次性返回
        response = oai.ChatCompletion.create(
            messages=messages,
            **self.llm_config
        )
        return response["choices"][0]["message"]["content"]
```

---

## 5. UserProxyAgent 角色

UserProxyAgent 在 AutoGen 系统中扮演"执行者"角色，桥接 LLM 的"思考"和外部世界的"行动"。

### 5.1 Human Input Mode

```python
# 三种 human_input_mode:
HUMAN_INPUT_MODES = {
    "NEVER": """
        从不询问用户。全自动模式。
        适用于完全信任的自动化场景。
    """,
    "ALWAYS": """
        每次 Agent 要回复时都暂停并询问用户。
        适用于需要人类监督每一步的场景。
        用户输入 '' (空串) → 使用自动生成的回复
        用户输入自定义文字 → 覆盖自动回复
    """,
    "TERMINATE": """
        仅在检测到 TERMINATE 时询问用户确认。
        适用于半自动化场景——重要决策需要人批准。
    """,
}

def generate_human_input_reply(self, messages, sender, **kwargs):
    if self.human_input_mode == "NEVER":
        return None  # 跳过，继续下一个 reply function
    
    user_input = input("请输入你的回复 (直接回车使用自动回复): ")
    if user_input.strip() == "":
        return None  # 使用自动生成的回复
    else:
        return user_input  # 使用用户输入覆盖自动回复
```

### 5.2 代码执行

```python
def execute_code_reply(self, messages, sender, **kwargs):
    """从消息中提取代码块并执行"""
    
    last_message = messages[-1]["content"]
    
    # 提取 Python 代码块
    code_blocks = extract_code_blocks(last_message)
    # [("python", "print('hello')"), ("bash", "ls -la")]
    
    if not code_blocks:
        return None  # 没有代码块，跳过
    
    results = []
    for lang, code in code_blocks:
        if lang == "python":
            # 在隔离环境中执行 Python
            result = execute_python(
                code,
                docker=self.code_execution_config.get("use_docker", False)
            )
        elif lang == "bash" or lang == "sh":
            result = execute_bash(code)
        else:
            result = f"Unsupported language: {lang}"
        
        results.append(result)
    
    # 格式化为消息
    return "\n".join(results)

def execute_python(code, use_docker=False):
    """执行 Python 代码，支持 Docker 隔离"""
    if use_docker:
        # Docker 中执行 (安全隔离)
        return run_in_docker(code)
    else:
        # 本地执行 (方便调试)
        return subprocess.run(
            ["python", "-c", code],
            capture_output=True, text=True
        )
```

### 5.3 函数注册与执行

```python
def register_function(self, func_map):
    """注册可调用的工具函数"""
    self._function_map.update(func_map)

def execute_function(self, messages, sender, **kwargs):
    """执行 LLM 请求的函数调用"""
    
    last_message = messages[-1]
    if "function_call" not in last_message:
        return None
    
    func_call = last_message["function_call"]
    func_name = func_call["name"]
    arguments = json.loads(func_call["arguments"])
    
    if func_name not in self._function_map:
        return f"Error: Function '{func_name}' not registered."
    
    try:
        # 在 UserProxyAgent 的上下文中执行
        result = self._function_map[func_name](**arguments)
        return str(result)
    except Exception as e:
        return f"Error executing {func_name}: {str(e)}"
```

### 5.4 函数结果反馈循环

函数执行完成后，结果会被格式化为 "function" role 的消息注入回对话历史，LLM 在下一次回复中可以看到执行结果并决定后续行动。

---

## 6. GroupChat 实现

### 6.1 架构概览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     GroupChat 多 Agent 对话                             │
│                                                                         │
│  ┌──────────────────┐            ┌──────────────────┐                  │
│  │  AssistantAgent  │            │   AssistantAgent  │                 │
│  │  (Researcher)    │◄─────────►│   (Coder)         │                  │
│  └──────┬───────────┘            └────────┬─────────┘                  │
│         │                                 │                             │
│         │    ┌──────────────────────┐     │                             │
│         └────┤    GroupChatManager  ├─────┘                             │
│              │  (Orchestrator)      │                                   │
│              │                      │                                   │
│              │  1. 选择下一个发言人  │                                   │
│              │  2. 转发消息给选中的  │                                   │
│              │     发言人           │                                   │
│              │  3. 广播给所有 Agent  │                                   │
│              │  4. 检查终止条件      │                                   │
│              └──────────────────────┘                                   │
│                      │                                                  │
│         ┌────────────┼────────────┐                                    │
│         │            │            │                                     │
│  ┌──────▼─────┐ ┌───▼────┐ ┌────▼──────┐                              │
│  │ UserProxy  │ │ Custom │ │ UserProxy  │                              │
│  │ (Executor) │ │ Agent  │ │ (Reviewer)│                              │
│  └────────────┘ └────────┘ └───────────┘                              │
│                                                                         │
│  Agent 间不直接通信——所有消息都经过 GroupChatManager 路由               │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Speaker Selection 策略

```python
class GroupChat:
    def select_speaker(self, last_speaker, manager):
        """选择下一个发言的 Agent"""
        
        if self.speaker_selection_method == "round_robin":
            # 轮询: 按列表顺序轮流发言
            idx = self.agent_names.index(last_speaker.name)
            return self.agents[(idx + 1) % len(self.agents)]
        
        elif self.speaker_selection_method == "auto":
            # 自动: 让 LLM 决定谁应该发言
            prompt = f"""
            You are the group chat manager.
            
            Current conversation context:
            {self.format_messages()}
            
            Last speaker: {last_speaker.name}
            
            Who should speak next? Choose from:
            {', '.join(self.agent_names)}
            
            Reply with just the agent name.
            """
            response = manager.create_oai_reply(prompt)
            return self.name_to_agent[response.strip()]
        
        elif callable(self.speaker_selection_method):
            # 自定义函数: 用户完全控制选择逻辑
            return self.speaker_selection_method(
                last_speaker=last_speaker,
                groupchat=self,
                manager=manager
            )
        
        else:
            raise ValueError(f"Unknown method: {self.speaker_selection_method}")
```

三种模式对比：

| 模式 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| round_robin | 简单轮流对话 | 确定性强、开销小 | 死板、不适应对话动态 |
| auto | 通用场景 | 灵活、LLM 决定谁最适合 | 增加 token 消耗、可能选择错误 |
| 自定义函数 | 需要领域逻辑 | 完全可控 | 需要额外编码 |

### 6.3 GroupChatManager 运行循环

```python
class GroupChatManager(UserProxyAgent):
    """GroupChatManager 本身是一个 UserProxyAgent"""
    
    def __init__(self, groupchat, **kwargs):
        super().__init__(**kwargs)
        self._groupchat = groupchat
        # 注册 group_chat_reply 到回复链的最前端
        self.register_reply(self.group_chat_reply, ...)
    
    def group_chat_reply(self, messages, sender, **kwargs):
        """GroupChat 的核心回复逻辑"""
        groupchat = self._groupchat
        
        while True:
            # 1. 检查是否达到 max_round
            if len(groupchat.messages) >= groupchat.max_round * len(groupchat.agents):
                return "TERMINATE"
            
            # 2. 选择下一个发言人
            speaker = groupchat.select_speaker(
                last_speaker=sender,
                manager=self
            )
            
            # 3. 将历史消息发送给选中的 Agent (不带系统提示，保持角色)
            #    这里用 send 但禁止递归回复
            speaker.send(
                message=groupchat.messages,
                recipient=speaker,
                request_reply=True
            )
            
            # 4. speaker.receive() 被触发
            #    在 receive 中，speaker 有自己的 reply chain
            #    AssistantAgent → 调用 LLM → 生成回复 → send 回来
            
            # 5. 接收 speaker 的回复
            reply = self.last_message()  # 简化的表示
            
            # 6. 追加到 groupchat 消息列表
            groupchat.messages.append(reply)
            
            # 7. 检查终止
            if "TERMINATE" in reply.get("content", ""):
                return "TERMINATE"
            
            # 8. 设置 sender 为刚发言的 speaker，继续循环
            sender = speaker
```

### 6.4 消息路由细节

```
GroupChat 消息路由流程 (一次发言):

Step 1: Manager 选择下一个发言人 (speaker = Agent_B)

Step 2: Manager 将当前消息历史发送给 speaker
    
    Manager.send(history, speaker)
        → speaker.receive(history, Manager)
            → speaker.generate_reply(history)
                → LLM 基于完整对话历史生成回复
            → reply = "I think we should..."
            → speaker.send(reply, Manager, request_reply=False)
                → Manager.receive(reply, speaker)
    
Step 3: Manager 更新群聊状态
    
    groupchat.messages.append(reply)
    last_speaker = speaker
    
重要细节:
    - GroupChatManager.send(history, speaker) 发送完整历史
    - 但 ConversableAgent.receive() 中如果收到 List 而非 str,
      会将列表各元素逐个加入 chat_messages[sender]
    - 因此每个 Agent 看到的上下文包含了完整的群聊记录
    
    但这也导致一个问题:
      每个 Agent 都在自己的 chat_messages 中维护一份完整历史
      → 历史重复存储 (manager 存一份, 每个 agent 各存一份)
      → 内存消耗大
```

---

## 7. 函数调用 (Function Calling)

### 7.1 Function Calling 生命周期

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   AutoGen Function Calling 完整流程                      │
│                                                                         │
│  注册阶段                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ def get_weather(city: str) -> str:                              │    │
│  │     return weather_service.query(city)                          │    │
│  │                                                                  │    │
│  │ assistant = AssistantAgent(...)                                 │    │
│  │ user_proxy = UserProxyAgent(                                    │    │
│  │     function_map={"get_weather": get_weather}                   │    │
│  │ )                                                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  调用阶段                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ AssistantAgent      UserProxyAgent                              │    │
│  │      │                    │                                      │    │
│  │      │── text msg ──────►│                                      │    │
│  │      │                    │                                      │    │
│  │      │◄── LLM thinks ────│  (user_proxy 将函数描述注入 prompt)    │    │
│  │      │                    │                                      │    │
│  │      │    LLM decides to call get_weather("Beijing")            │    │
│  │      │                    │                                      │    │
│  │      │── function_call ─►│                                      │    │
│  │      │   {name, args}    │                                      │    │
│  │      │                    ├─ execute_function()                  │    │
│  │      │                    │  → get_weather("Beijing")           │    │
│  │      │                    │  → "Sunny, 25°C"                    │    │
│  │      │                    │                                      │    │
│  │      │◄─ function_result─│                                      │    │
│  │      │   {role:func,     │                                      │    │
│  │      │    content: result}│                                      │    │
│  │      │                    │                                      │    │
│  │      │  LLM sees result  │                                      │    │
│  │      │  generates text   │                                      │    │
│  │      │                    │                                      │    │
│  │      │── text reply ────►│                                      │    │
│  │      │   "Beijing is 25°C│                                      │    │
│  │      │    and sunny."    │                                      │    │
│  │      │                    │                                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  执行阶段 (UserProxyAgent.execute_function 内部)                        │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 1. parse function_call: name="get_weather", args={"city":"BJ"}   │    │
│  │ 2. lookup self._function_map["get_weather"] → <function>         │    │
│  │ 3. call get_weather(city="BJ")                                   │    │
│  │ 4. format result as message:                                     │    │
│  │    {                                                              │    │
│  │      "role": "function",                                         │    │
│  │      "content": "Sunny, 25°C",                                   │    │
│  │      "name": "get_weather"                                       │    │
│  │    }                                                              │    │
│  │ 5. append to chat_messages  ← 注入对话历史                        │    │
│  │ 6. send back to AssistantAgent                                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 register_function() 的源码实现

```python
# 现代 AutoGen (>=0.2) 推荐的方式
from autogen import register_function

def register_function(
    func: Callable,                      # 要注册的函数
    caller: ConversableAgent,            # 调用方 (通常是 AssistantAgent)
    executor: ConversableAgent,          # 执行方 (通常是 UserProxyAgent)
    name: Optional[str] = None,          # 函数名 (默认用 func.__name__)
    description: Optional[str] = None,   # 函数描述
):
    """注册一个函数，连接 caller 和 executor"""
    
    func_name = name or func.__name__
    func_desc = description or func.__doc__ or ""
    
    # 1. 将函数签名的类型注解转换为 JSON Schema
    func_schema = func_to_json_schema(func)
    # 例如: get_weather(city: str) → {"type": "function",
    #   "function": {"name": "get_weather", "parameters": {
    #     "type": "object",
    #     "properties": {"city": {"type": "string"}},
    #     "required": ["city"]
    #   }}}
    
    # 2. 向 caller (AssistantAgent) 注册函数描述
    #    → 注入到 llm_config 的 tools 字段
    caller.update_tool_schema(func_schema)
    # 这样 LLM 在生成回复时就知道可以调用这个函数
    
    # 3. 向 executor (UserProxyAgent) 注册可执行函数
    executor.register_function({func_name: func})
    # 这样 UserProxyAgent 收到 function_call 就能找到对应函数
    
    return func_schema
```

### 7.3 函数签名到 JSON Schema 的转换

```python
def func_to_json_schema(func: Callable) -> dict:
    """将 Python 函数签名转换为 OpenAI function calling schema"""
    
    sig = inspect.signature(func)
    schema = {
        "type": "function",
        "function": {
            "name": func.__name__,
            "description": func.__doc__ or "",
            "parameters": {
                "type": "object",
                "properties": {},
                "required": []
            }
        }
    }
    
    for name, param in sig.parameters.items():
        # 从类型注解推断 JSON Schema type
        param_schema = {
            "type": python_type_to_json_type(param.annotation)
        }
        if param.default is inspect.Parameter.empty:
            schema["function"]["parameters"]["required"].append(name)
        if param.default is not inspect.Parameter.empty:
            param_schema["default"] = param.default
        
        schema["function"]["parameters"]["properties"][name] = param_schema
    
    return schema
```

---

## 8. 关键源码实现 (Pseudo-code)

以下从零开始伪代码实现 AutoGen 的核心机制，帮助理解其内部工作原理。

### 8.1 ConversableAgent 基类

```python
class ConversableAgent:
    """所有 Agent 的基类——定义了 send/receive 通信协议"""
    
    def __init__(self, name, system_message="", 
                 human_input_mode="NEVER",
                 max_consecutive_auto_reply=10,
                 llm_config=None,
                 code_execution_config=None,
                 function_map=None):
        
        self.name = name
        self.system_message = system_message
        self.human_input_mode = human_input_mode
        self.max_consecutive_auto_reply = max_consecutive_auto_reply
        self.llm_config = llm_config or {}
        self.code_execution_config = code_execution_config or {}
        
        # 按对话方隔离的消息历史
        # chat_messages[sender_agent] = [msg1, msg2, ...]
        self.chat_messages: dict = {}
        
        # 已注册的函数映射
        self._function_map: dict = function_map or {}
        
        # 自动回复函数链 (按优先级降序)
        # 每个元素是 (func, ...config)
        self._reply_func_list = []
        
        # 默认注册回复链
        self._register_default_replies()
        
        # 连续自动回复计数
        self._consecutive_auto_reply_counter = defaultdict(int)
    
    def _register_default_replies(self):
        """注册默认回复函数链 (子类可覆盖)"""
        # 1. 检查终止条件
        self.register_reply(ReplyFunction(
            func=self._check_termination,
            config={"condition": lambda msg: "TERMINATE" in msg}
        ))
        # 2. 检查是否需要人类输入
        self.register_reply(ReplyFunction(
            func=self._generate_human_input_reply,
            config={"mode": self.human_input_mode}
        ))
        # 3. 生成 LLM 回复 (兜底)
        self.register_reply(ReplyFunction(
            func=self._generate_oai_reply,
            config=self.llm_config
        ))
    
    def register_reply(self, reply_func):
        """注册回复函数"""
        self._reply_func_list.append(reply_func)
    
    def send(self, message, recipient, request_reply=True):
        """向 recipient 发送消息"""
        # 记录发出的消息
        self._append_oai_message(message, self, recipient)
        
        if request_reply:
            # 触发接收方的 receive 方法
            recipient.receive(message, self, request_reply=True)
    
    def receive(self, message, sender, request_reply=True):
        """从 sender 接收消息"""
        # 记录收到的消息
        self._append_oai_message(message, sender, self)
        
        if request_reply:
            # 生成回复并发回
            reply = self.generate_reply(
                messages=self.chat_messages[sender],
                sender=sender
            )
            if reply is not None:
                # request_reply=False 防止无限递归！
                self.send(reply, sender, request_reply=False)
    
    def _append_oai_message(self, message, sender, recipient):
        """将消息追加到历史"""
        # 确保 sender 的历史列表存在
        if sender not in self.chat_messages:
            self.chat_messages[sender] = []
        
        # 标准化消息格式
        if isinstance(message, str):
            msg = {
                "content": message,
                "role": "user" if sender == self else "assistant",
                "name": sender.name
            }
        elif isinstance(message, dict):
            msg = message
        elif isinstance(message, list):
            # 如果是列表 (GroupChat 发送历史), 逐个添加
            for m in message:
                self.chat_messages[sender].append(m)
            return
        else:
            raise TypeError(f"Unsupported message type: {type(message)}")
        
        self.chat_messages[sender].append(msg)
    
    def generate_reply(self, messages, sender, **kwargs):
        """遍历回复链，第一个非空返回即为回复"""
        self._consecutive_auto_reply_counter[sender] += 1
        
        # 检查是否超过连续自动回复上限
        if (self._consecutive_auto_reply_counter[sender] 
            > self.max_consecutive_auto_reply):
            return None
        
        for reply_func in self._reply_func_list:
            is_final, reply = reply_func.func(
                self,
                messages=messages,
                sender=sender,
                config=reply_func.config
            )
            if reply is not None:
                # 如果是最终回复 (is_final=True), 不继续尝试
                # 否则继续遍历 (通常用于执行完函数后还要 LLM 总结)
                if is_final:
                    return reply
                # 非最终回复: 继续下一层
        return None
    
    def initiate_chat(self, recipient, message, **kwargs):
        """发起与 recipient 的对话"""
        # 清空连续回复计数
        self._consecutive_auto_reply_counter.clear()
        recipient._consecutive_auto_reply_counter.clear()
        
        # 发送初始消息
        self.send(message, recipient, request_reply=True)
    
    def register_function(self, function_map):
        """注册可执行的函数"""
        self._function_map.update(function_map)
```

### 8.2 AssistantAgent

```python
class AssistantAgent(ConversableAgent):
    """LLM 驱动的 Agent——收到消息时自动调用 LLM 生成回复"""
    
    def __init__(self, name, system_message="You are a helpful assistant.",
                 llm_config=None, **kwargs):
        
        super().__init__(
            name=name,
            system_message=system_message,
            llm_config=llm_config,
            **kwargs
        )
        
        # AssistantAgent 默认不使用人类输入
        self.human_input_mode = "NEVER"
    
    def _register_default_replies(self):
        """AssistantAgent 的默认回复链:
        1. 检查终止
        2. 执行函数调用 (如果 LLM 请求了)
        3. 调用 LLM 生成回复
        """
        self.register_reply(ReplyFunction(
            func=self._check_termination,
            config={}
        ))
        self.register_reply(ReplyFunction(
            func=self._generate_function_call_reply,
            config={}
        ))
        self.register_reply(ReplyFunction(
            func=self._generate_oai_reply,
            config=self.llm_config
        ))
    
    def _generate_oai_reply(self, messages, sender, config):
        """调用 OpenAI API 生成回复"""
        oai_config = config or self.llm_config
        
        # 构造 OpenAI API 的消息列表
        oai_messages = [
            {"role": "system", "content": self.system_message},
            *messages  # 历史消息
        ]
        
        # 如果注册了函数，注入 tools 参数
        if self._function_map:
            oai_messages[-1]["tools"] = [
                func_to_openai_tool(f) 
                for f in self._function_map
            ]
        
        # 调用 LLM
        response = openai.ChatCompletion.create(
            model=oai_config.get("model", "gpt-4"),
            messages=oai_messages,
            temperature=oai_config.get("temperature", 0.7),
            max_tokens=oai_config.get("max_tokens", 4096),
            stream=oai_config.get("stream", False)
        )
        
        # 提取回复内容
        if oai_config.get("stream", False):
            # 流式: 累积 chunks
            content = ""
            for chunk in response:
                delta = chunk["choices"][0].get("delta", {})
                content += delta.get("content", "")
            return content
        else:
            # 非流式: 直接取 content
            msg = response["choices"][0]["message"]
            return msg.get("content", "")
    
    def _generate_function_call_reply(self, messages, sender, config):
        """处理 LLM 返回的函数调用请求"""
        last_msg = messages[-1] if messages else {}
        
        # 检查最后一次消息是否包含 function_call
        if "function_call" not in last_msg:
            return False, None  # 不处理
        
        # 提取函数调用信息
        func_call = last_msg["function_call"]
        func_name = func_call.get("name", "")
        arguments = json.loads(func_call.get("arguments", "{}"))
        
        if func_name not in self._function_map:
            return False, None  # 不在本 Agent 注册的函数中
        
        # 执行函数
        try:
            result = self._function_map[func_name](**arguments)
            return True, str(result)
        except Exception as e:
            return True, f"Error: {str(e)}"
```

### 8.3 UserProxyAgent

```python
class UserProxyAgent(ConversableAgent):
    """人类代理 / 代码执行 Agent"""
    
    def __init__(self, name, human_input_mode="NEVER",
                 code_execution_config=None,
                 function_map=None, **kwargs):
        
        super().__init__(
            name=name,
            human_input_mode=human_input_mode,
            code_execution_config=code_execution_config,
            function_map=function_map,
            **kwargs
        )
    
    def _register_default_replies(self):
        """UserProxyAgent 的默认回复链:
        1. 检查终止
        2. 人类输入
        3. 执行函数调用
        4. 执行代码
        5. 生成回复 (兜底)
        """
        self.register_reply(ReplyFunction(
            func=self._check_termination, config={}
        ))
        self.register_reply(ReplyFunction(
            func=self._generate_human_input_reply,
            config={"mode": self.human_input_mode}
        ))
        self.register_reply(ReplyFunction(
            func=self._execute_function_reply, config={}
        ))
        self.register_reply(ReplyFunction(
            func=self._execute_code_reply,
            config=self.code_execution_config or {}
        ))
        self.register_reply(ReplyFunction(
            func=self._generate_oai_reply,
            config=self.llm_config or {}
        ))
    
    def _generate_human_input_reply(self, messages, sender, config):
        """生成人类输入回复"""
        mode = config.get("mode", "NEVER")
        
        if mode == "NEVER":
            return False, None
        
        if mode == "TERMINATE":
            last_msg = messages[-1] if messages else {}
            if "TERMINATE" not in last_msg.get("content", ""):
                return False, None  # 只在 TERMINATE 时询问
        
        # 请求用户输入
        print(f"\n==== {self.name} 需要你的输入 ====")
        print(f"对话上下文:\n{messages[-1].get('content', '')[:200]}")
        user_input = input("你的回复 (直接回车→自动回复): ")
        
        if user_input.strip() == "":
            return False, None  # 使用自动回复
        else:
            return True, user_input  # 使用用户输入
    
    def _execute_function_reply(self, messages, sender, config):
        """执行 LLM 请求的函数调用"""
        last_msg = messages[-1] if messages else {}
        
        if "function_call" not in last_msg:
            return False, None  # 没有函数调用
        
        func_call = last_msg["function_call"]
        func_name = func_call.get("name", "")
        arguments = json.loads(func_call.get("arguments", "{}"))
        
        if func_name not in self._function_map:
            return False, f"Error: Function '{func_name}' not registered."
        
        try:
            result = self._function_map[func_name](**arguments)
            # 将结果格式化为 "function" role 的消息
            return True, {
                "role": "function",
                "content": str(result),
                "name": func_name
            }
        except Exception as e:
            return True, f"Error executing {func_name}: {str(e)}"
    
    def _execute_code_reply(self, messages, sender, config):
        """从消息中提取代码块并执行"""
        last_msg = messages[-1] if messages else {}
        content = last_msg.get("content", "")
        
        # 提取代码块 (正则匹配 ```lang ... ```)
        code_blocks = self._extract_code_blocks(content)
        
        if not code_blocks:
            return False, None  # 没有代码块
        
        results = []
        for lang, code in code_blocks:
            if lang in ("python", "py"):
                result = self._run_python(code, config)
            elif lang in ("bash", "sh", "shell"):
                result = self._run_bash(code, config)
            else:
                result = f"Unsupported language: {lang}"
            results.append(result)
        
        return True, json.dumps({"results": results})
    
    def _extract_code_blocks(self, text):
        """提取 Markdown 代码块"""
        pattern = r"```(\w+)?\n(.*?)```"
        matches = re.findall(pattern, text, re.DOTALL)
        return [(lang or "text", code.strip()) for lang, code in matches]
    
    def _run_python(self, code, config):
        """执行 Python 代码"""
        use_docker = config.get("use_docker", False)
        
        if use_docker:
            return self._run_in_docker(code)
        else:
            try:
                result = subprocess.run(
                    [sys.executable, "-c", code],
                    capture_output=True, text=True,
                    timeout=config.get("timeout", 60)
                )
                output = result.stdout
                if result.stderr:
                    output += "\n" + result.stderr
                return output
            except subprocess.TimeoutExpired:
                return "Error: Code execution timed out"
            except Exception as e:
                return f"Error: {str(e)}"
```

### 8.4 GroupChat 和 GroupChatManager

```python
class GroupChat:
    """群聊配置——纯数据模型"""
    
    def __init__(self, agents, messages=None, max_round=10,
                 speaker_selection_method="round_robin",
                 admin_name="Admin"):
        
        self.agents = agents
        self.messages = messages or []
        self.max_round = max_round
        self.speaker_selection_method = speaker_selection_method
        self.admin_name = admin_name
        
        # 构建名称到 Agent 的映射
        self.agent_names = [a.name for a in agents]
        self.name_to_agent = {a.name: a for a in agents}
    
    def select_speaker(self, last_speaker, manager):
        """选择下一个发言人"""
        if self.speaker_selection_method == "round_robin":
            # 轮询
            idx = self.agent_names.index(last_speaker.name)
            next_idx = (idx + 1) % len(self.agents)
            return self.agents[next_idx]
        
        elif self.speaker_selection_method == "auto":
            # LLM 选择
            return self._select_speaker_by_llm(last_speaker, manager)
        
        elif callable(self.speaker_selection_method):
            # 自定义函数
            return self.speaker_selection_method(
                last_speaker=last_speaker,
                groupchat=self,
                manager=manager
            )
        
        else:
            raise ValueError(
                f"Unknown speaker_selection_method: "
                f"{self.speaker_selection_method}"
            )
    
    def _select_speaker_by_llm(self, last_speaker, manager):
        """让 LLM 决定下一个发言人"""
        prompt = f"""
        Given the following conversation context and the last speaker,
        determine who should speak next.
        
        Agents: {', '.join(self.agent_names)}
        Last speaker: {last_speaker.name}
        
        Conversation history:
        {self._format_messages_for_prompt()}
        
        Reply ONLY with the agent's name.
        """
        response = manager._generate_oai_reply(
            messages=[{"role": "user", "content": prompt}],
            sender=manager,
            config=manager.llm_config
        )
        
        # 解析响应中的 Agent 名称
        for name in self.agent_names:
            if name in response:
                return self.name_to_agent[name]
        
        # 兜底: 轮询
        idx = self.agent_names.index(last_speaker.name)
        return self.agents[(idx + 1) % len(self.agents)]
    
    def _format_messages_for_prompt(self):
        """格式化消息为可读文本"""
        lines = []
        for msg in self.messages[-10:]:  # 只取最近 10 条
            name = msg.get("name", "Unknown")
            content = msg.get("content", "")
            role = msg.get("role", "")
            lines.append(f"[{name}] ({role}): {content[:100]}")
        return "\n".join(lines)
    
    def append(self, message, sender):
        """追加消息到历史"""
        self.messages.append(message)
    
    def reset(self):
        """重置群聊"""
        self.messages = []


class GroupChatManager(UserProxyAgent):
    """群聊管理器——继承 UserProxyAgent，拥有代码执行能力"""
    
    def __init__(self, groupchat, **kwargs):
        super().__init__(
            name="chat_manager",
            system_message="Group chat manager.",
            **kwargs
        )
        self._groupchat = groupchat
        
        # 将 group_chat_manager_reply 注册到回复链最前
        self.register_reply(ReplyFunction(
            func=self._group_chat_manager_reply,
            config={}
        ), position=0)  # 插入到最前面
    
    def _group_chat_manager_reply(self, messages, sender, config):
        """GroupChat 的核心回复逻辑"""
        groupchat = self._groupchat
        
        # 传入的消息是整个群聊历史
        for msg in messages:
            groupchat.append(msg, sender)
        
        # 检查是否达到最大轮次
        total_messages = len(groupchat.messages)
        max_allowed = groupchat.max_round * len(groupchat.agents)
        
        if total_messages >= max_allowed:
            return True, "TERMINATE"
        
        # 选择下一个发言人
        speaker = groupchat.select_speaker(
            last_speaker=sender,
            manager=self
        )
        
        # 将当前群聊消息发送给 speaker
        # 注意: 发送整个消息列表，而不是单条消息
        speaker.send(
            groupchat.messages,  # 完整的对话历史
            self,                # 发送方是 GroupChatManager
            request_reply=True   # 期待回复
        )
        
        # sender.send() 触发 speaker.receive()
        # speaker.receive() 触发 speaker.generate_reply()
        # speaker.generate_reply() 使用完整历史调用 LLM
        # 生成回复后 send 回 GroupChatManager
        
        # 获取最新消息 (speaker 发回的回复)
        last_message = self.chat_messages[speaker][-1]
        groupchat.append(last_message, speaker)
        
        # 检查终止条件
        content = last_message.get("content", "")
        if "TERMINATE" in content:
            return True, "TERMINATE"
        
        # 继续循环——GroupChatManager 再次进入 speaker selection
        # 但是由于我们已经有了回复，这里返回消息给主循环
        return False, None  # 不是最终回复，继续
    
    def run_chat(self, initial_message=None):
        """启动群聊 (简化版)"""
        if initial_message:
            # 将初始消息注入群聊
            msg = {
                "content": initial_message,
                "role": "user",
                "name": self.admin_name or "Admin"
            }
            self._groupchat.append(msg, None)
            
            # 选择一个初始发言人
            speaker = self._groupchat.select_speaker(
                last_speaker=self,  # 假设 manager 是 "前一个说话人"
                manager=self
            )
            
            # 发送消息给初始发言人
            speaker.send(
                self._groupchat.messages,
                self,
                request_reply=True
            )
            
            last_message = self.chat_messages[speaker][-1]
            self._groupchat.append(last_message, speaker)
            
            # GroupChatManager 的 generate_reply 会继续循环
            # 直到达到 max_round 或收到 TERMINATE
```

### 8.5 函数注册和执行完整流程

```python
# ============================================================
# 完整的使用示例 + 内部流程
# ============================================================

# 1. 定义工具函数
def get_weather(city: str) -> str:
    """获取指定城市的天气"""
    # 假设调用外部 API
    weather_data = {
        "Beijing": "Sunny, 25°C",
        "Shanghai": "Rainy, 20°C",
        "Shenzhen": "Cloudy, 28°C"
    }
    return weather_data.get(city, f"No data for {city}")

def calculate(expression: str) -> str:
    """计算数学表达式"""
    try:
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"Error: {str(e)}"

# 2. 创建 Agent
assistant = AssistantAgent(
    name="Assistant",
    system_message="You are a helpful assistant. Use tools when needed.",
    llm_config={
        "model": "gpt-4",
        "temperature": 0.7,
        # 工具函数描述在 initiate_chat 时自动注入
    }
)

user_proxy = UserProxyAgent(
    name="UserProxy",
    human_input_mode="NEVER",      # 全自动
    code_execution_config={
        "use_docker": False,        # 本地执行
        "timeout": 30
    },
    function_map={
        "get_weather": get_weather,
        "calculate": calculate
    }
)

# 3. 通过 register_function 连接 caller 和 executor
#    内部等价于:
#    user_proxy.register_function({
#        "get_weather": get_weather,
#        "calculate": calculate
#    })
#    assistant.llm_config["tools"] += [get_weather_schema, calculate_schema]

register_function(
    get_weather,
    caller=assistant,
    executor=user_proxy,
    description="获取指定城市的天气信息"
)

register_function(
    calculate,
    caller=assistant,
    executor=user_proxy,
    description="计算数学表达式的结果"
)

# 4. 发起对话
#    内部流程:
#    user_proxy.initiate_chat(assistant, "北京天气怎么样？")
#    → user_proxy.send(msg, assistant)
#      → assistant.receive(msg, user_proxy)
#        → assistant.generate_reply(history)
#          1. check_termination → skip
#          2. generate_function_call_reply → skip (没有 function_call)
#          3. generate_oai_reply → LLM({"role":"user","content":"北京天气…"})
#            LLM 回复: {"content": None, "function_call": {"name": "get_weather", "arguments": "{\"city\": \"Beijing\"}"}}
#        → assistant.send(reply, user_proxy, request_reply=False)
#          → user_proxy.receive(reply, assistant)
#            → user_proxy.generate_reply(history)
#              1. check_termination → skip
#              2. human_input → skip (NEVER)
#              3. execute_function_reply
#                 → lookup get_weather in _function_map
#                 → get_weather(city="Beijing")
#                 → "Sunny, 25°C"
#                 → 格式化为 {"role": "function", "content": "Sunny, 25°C", "name": "get_weather"}
#              4. execute_code_reply → skip (不是最终回复)
#              5. generate_oai_reply → 作为最终回复
#        → user_proxy.send(result, assistant, request_reply=False)
#          → assistant.receive(result, user_proxy)
#            历史注入函数执行结果
#            LLM: "北京今天天气晴朗，25摄氏度。"
#        → ... 继续对话直到 TERMINATE

user_proxy.initiate_chat(
    assistant,
    message="北京天气怎么样？"
)
```

---

## 9. 局限性与改进

### 9.1 对话历史爆炸 (Conversation History Explosion)

```
问题: 每个 Agent 在 chat_messages[sender] 中保存完整历史
      GroupChat 中 GroupChatManager 也保存一份
      N 个 Agent 的群聊 → 同一份历史被复制 N+1 次

┌─────────────────────────────────────────────────────────┐
│  内存消耗分析 (M = 总消息数, N = Agent 数量)            │
│                                                          │
│  Two-Agent Chat:                                         │
│    Assistant.chat_messages[UserProxy] = [msg1..msgM]     │  M 条
│    UserProxy.chat_messages[Assistant] = [msg1..msgM]     │  M 条
│    总计: 2M 条                                            │
│                                                          │
│  GroupChat (N agents):                                   │
│    GroupChat.messages = [msg1..msgM]                     │  M 条
│    Agent_1.chat_messages[Manager] = [msg1..msgM]         │  M 条
│    Agent_2.chat_messages[Manager] = [msg1..msgM]         │  M 条
│    ...                                                    │
│    Agent_N.chat_messages[Manager] = [msg1..msgM]         │  M 条
│    Manager.chat_messages[each_agent] = subset            │  ~M 条
│    总计: ~(N+2)M 条                                       │
└─────────────────────────────────────────────────────────┘

改进方向:
  - 引用计数 + 共享历史 (copy-on-write)
  - 滑动窗口 + 摘要压缩 (类似 MemWalker)
  - 持久化到外部存储 (Redis/DB), 仅加载最近上下文
```

### 9.2 结构化编排不足

```python
# AutoGen 的问题: 一切对话
# 没有"并行执行"、"条件分支"、"循环"等原生概念

# 比如: 同时搜索三个不同来源, 然后汇总
# 在 CrewAI 中可以:
#   Process(
#     [SearchAgent(task1), SearchAgent(task2), SearchAgent(task3)],
#     aggregate=SummarizeAgent
#   )

# 在 AutoGen 中只能:
#   Agent 1: "我搜索了来源 A, 结果是..."
#   Agent 2: "我搜索了来源 B, 结果是..."  
#   Agent 3: "我搜索了来源 C, 结果是..."
#   然后 LLM 在对话中理解"哦, 我该汇总了"
#   → 依赖 LLM 的理解能力, 不稳定

改进方向:
  - 引入显式的 Task/Plan 原语
  - 支持 DAG 图执行 (参考 LangGraph)
  - 提供 Process-like 的 Flow 抽象
```

### 9.3 GroupChat 轮次管理挑战

```
问题 1: Speaker 选择可能重复
  - round_robin: 固定顺序, 无法处理"当前 Agent 无话可说"
  - auto: LLM 可能连续多次选择同一个 Agent
  
  改进: 引入"跳过"机制 + 反重复奖励

问题 2: 累计回复数 vs 有效轮次
  - 一次发言可能需要多个来回 (Assistant 思考 → UserProxy 执行 → 返回结果)
  - max_round=10 可能实际只完成了 3 个有效步骤
  
  改进: max_round 按"有效发言"计数, 跳过函数执行/代码执行的中间来回

问题 3: 发言权控制
  - 当前设计没有"抢答"或"优先级"机制
  - 所有 Agent 平等, 无法表达"这个 Agent 对当前话题更重要"
  
  改进: 支持说话人优先级权重
```

### 9.4 调试复杂性

```python
# AutoGen 的对话是异步触发的
# send → receive → generate_reply → send → receive → ...
# 调用栈很深, 调试困难

问题:
  - 无法单步跟踪"对话走向"
  - 日志分散在各个 Agent 的 generate_reply 中
  - LLM 调用返回的非确定性使得复现困难

改进方向:
  - 内置 Event Bus / 中间件机制
  - 对话轨迹可视化 (DAG 图)
  - 支持"确定性模式": 固定 seed + 记录所有 LLM 调用
```

### 9.5 其他已知问题

| 问题 | 描述 | 缓解方式 |
|------|------|----------|
| 上下文截断 | 超出 token 限制时简单丢弃旧消息 | 使用摘要 Agent 压缩历史 |
| 函数执行安全 | 本地代码执行有安全风险 | 使用 Docker 隔离 + 沙箱 |
| 终止检测脆弱 | 仅靠 "TERMINATE" 字符串匹配 | 自定义终止条件函数 |
| GroupChatManager 单点故障 | Manager 挂了整个群聊终止 | 冗余 Manager + 恢复机制 |
| 缺少消息验证 | 消息格式错误可能导致崩溃 | Pydantic 模型验证 |

---

## 附件: ASCII 图汇总

### A. AutoGen 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AutoGen Architecture                            │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     Application Layer                            │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │    │
│  │  │ Two-Agent    │  │ GroupChat    │  │ Custom Pattern       │  │    │
│  │  │ Chat Pattern │  │ Pattern      │  │ (Sequential, etc.)   │  │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │    │
│  └─────────┼─────────────────┼──────────────────────┼──────────────┘    │
│            │                 │                      │                    │
│  ┌─────────▼─────────────────▼──────────────────────▼──────────────┐    │
│  │                     Agent Layer                                  │    │
│  │  ┌──────────────────────────────────────────────────────────┐   │    │
│  │  │  ConversableAgent (Base)                                  │   │    │
│  │  │    ├─ send(message, recipient)                           │   │    │
│  │  │    ├─ receive(message, sender)                           │   │    │
│  │  │    ├─ generate_reply(messages)                           │   │    │
│  │  │    └─ _reply_func_list [...]                             │   │    │
│  │  └──────────────────────────────────────────────────────────┘   │    │
│  │         ▲                        ▲                               │    │
│  │         │                        │                               │    │
│  │  ┌──────┴──────────┐   ┌────────┴──────────┐                   │    │
│  │  │ AssistantAgent   │   │ UserProxyAgent     │                   │    │
│  │  │  (LLM-driven)    │   │ (Human/Code Exec)  │                   │    │
│  │  │  - generate_oai │   │  - execute_code    │                   │    │
│  │  │  - function_call│   │  - human_input     │                   │    │
│  │  └─────────────────┘   │  - execute_function│                   │    │
│  │                        └────────────────────┘                   │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                   Communication Layer                             │    │
│  │  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐   │    │
│  │  │ Message Bus  │  │ GroupChatManager │  │ Event / Callback │   │    │
│  │  │ (send/receive)│  │ (Speaker Select) │  │ (on_reply, ...)   │   │    │
│  │  └──────────────┘  └──────────────────┘  └──────────────────┘   │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                   Integration Layer                               │    │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌────────────────┐  │    │
│  │  │ OpenAI    │ │ Code Exec │ │ Docker    │ │ Custom Tools   │  │    │
│  │  │ (LLM API) │ │ (Python)  │ │ (Sandbox) │ │ (REST, DB, ...)│  │    │
│  │  └───────────┘ └───────────┘ └───────────┘ └────────────────┘  │    │
│  └──────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### B. Two-Agent 对话消息流

```
                            Message Flow: Assistant ↔ UserProxy

  UserProxy                              Assistant
     │                                       │
     │  1. initiate_chat(msg)                │
     │  ───────────────────────────────────►  │
     │                                       │
     │                                       │  2. receive(msg, sender)
     │                                       │     ├─ chat_messages[sender].append(msg)
     │                                       │     └─ generate_reply(messages)
     │                                       │          └─ _generate_oai_reply()
     │                                       │               └─ LLM.generate(messages)
     │                                       │
     │  3. send(reply, request_reply=False)  │
     │  ◄─────────────────────────────────── │
     │                                       │
     │  4. receive(reply, sender)            │
     │     ├─ chat_messages[sender].append() │
     │     └─ generate_reply()              │
     │          ├─ execute_code_reply()     │
     │          │    └─ run_python(code)    │
     │          └─ generate_oai_reply()     │
     │               └─ LLM.generate()      │
     │                                       │
     │  5. send(result, request_reply=False) │
     │  ───────────────────────────────────►  │
     │                                       │
     │                                       │  6. receive(result)
     │                                       │     └─ LLM.generate(修正)
     │                                       │
     │  (循环 2-6 直到 TERMINATE)            │
     │                                       │
     │  7. "TERMINATE"                       │
     │  ◄──────────────────────────────────► │
     │                                       │

    关键: request_reply=False 防止无限递归
          每个 generate_reply 遍历 _reply_func_list
    Agent A              Agent B
    send(msg, B) ──────► receive(msg, A)
                           generate_reply()
                           send(reply, A, request_reply=False)
                        ◄─ receive(reply, B)  ← 不再回复!
```

### C. GroupChat 多 Agent 对话

```
                        GroupChat 消息路由

            ┌─────────────────────────────────────┐
            │         GroupChatManager            │
            │                                     │
            │  ① select_speaker(last_speaker)     │
            │     → returns Agent_C               │
            │                                     │
            │  ② send(history, Agent_C)           │
            │     └─ Agent_C.receive(history)     │
            │        └─ LLM.generate(history)     │
            │           └─ send(reply, manager)   │
            │                                     │
            │  ③ receive(reply, Agent_C)          │
            │     ├─ groupchat.append(reply)       │
            │     ├─ check TERMINATE              │
            │     └─ → 回到 ①                     │
            └─────────────────────────────────────┘
                      │          │          │
          ┌───────────┘          │          └───────────┐
          ▼                      ▼                      ▼
   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
   │  Agent_A     │     │  Agent_B     │     │  Agent_C     │
   │ (Assistant)  │     │ (UserProxy)  │     │ (Assistant)  │
   ├──────────────┤     ├──────────────┤     ├──────────────┤
   │ chat_messages│     │ chat_messages│     │ chat_messages│
   │ [Manager]:   │     │ [Manager]:   │     │ [Manager]:   │
   │   msg_1      │     │   msg_1      │     │   msg_1      │
   │   msg_2      │     │   msg_2      │     │   msg_2      │
   │   ...        │     │   ...        │     │   ...        │
   └──────────────┘     └──────────────┘     └──────────────┘

   轮次顺序 (round_robin):
   Round 1: Agent_A → Agent_B → Agent_C
   Round 2: Agent_A → Agent_B → Agent_C
   ...

   注意: 所有 Agent 看到的是完全相同的对话历史
         GroupChatManager 只负责"路由", 不修改消息内容
```

### D. Function Call 完整生命周期

```
                     Function Call Lifecycle

                         ┌─────────────┐
                         │  User Task  │
                         │ "天气如何?" │
                         └──────┬──────┘
                                │
                                ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    AssistantAgent                            │
  │                                                              │
  │  1. receive(message)                                         │
  │     └─ generate_reply()                                      │
  │          └─ _generate_oai_reply()                            │
  │               └─ LLM(messages_with_tools_schema)             │
  │                                                              │
  │  2. LLM 返回: {                                              │
  │       content: None,                                         │
  │       function_call: {                                       │
  │         name: "get_weather",                                  │
  │         arguments: '{"city":"Beijing"}'                       │
  │       }                                                       │
  │     }                                                         │
  │                                                              │
  │  3. 将 function_call 打包成消息, send 给 UserProxy            │
  │     send({function_call: {...}}, user_proxy)                  │
  └──────────────────────────┬──────────────────────────────────┘
                             │
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    UserProxyAgent                            │
  │                                                              │
  │  4. receive(message)                                         │
  │     └─ generate_reply()                                      │
  │          ├─ check_termination()  → skip                      │
  │          ├─ human_input()       → skip (NEVER)              │
  │          ├─ execute_function_reply()  ← match!              │
  │          │   ├─ parse: name="get_weather", args={city:"BJ"} │
  │          │   ├─ lookup: _function_map["get_weather" ]       │
  │          │   ├─ call: get_weather(city="Beijing")           │
  │          │   └─ result: "Sunny, 25°C"                       │
  │          │                                                  │
  │          └─ 格式化返回:                                       │
  │              {role:"function", content:"Sunny, 25°C",        │
  │               name:"get_weather"}                            │
  │                                                              │
  │  5. send(result, assistant, request_reply=False)             │
  └──────────────────────────┬──────────────────────────────────┘
                             │
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    AssistantAgent                            │
  │                                                              │
  │  6. receive(function_result)                                 │
  │     └─ chat_messages 中已经有了:                              │
  │        [                                                    │
  │          {"role": "user", "content": "天气如何?"},           │
  │          {"role": "assistant", "function_call": {...}},      │
  │          {"role": "function", "content": "Sunny, 25°C",     │
  │           "name": "get_weather"}                             │
  │        ]                                                     │
  │                                                              │
  │  7. generate_reply() → LLM 看到 function result              │
  │     → "北京今天天气晴朗，25摄氏度。"                           │
  │                                                              │
  │  8. send(最终回复, user_proxy)                               │
  └─────────────────────────────────────────────────────────────┘

  LLM 内部视角 (发送给 LLM 的 messages 数组):
  
  [
    {"role": "system",    "content": "You are a helpful assistant..."},
    {"role": "user",      "content": "北京天气怎么样？"},
    {"role": "assistant", "content": None, 
     "function_call": {"name": "get_weather", "args": '{"city":"北京"}'}},
    {"role": "function",  "content": "Sunny, 25°C", 
     "name": "get_weather"},
    {"role": "assistant", "content": "北京今天天气晴朗，25°C。"}
  ]
  ↑ LLM 看到的是完整的"思考-行动-观察"轨迹
```

### E. GroupChat 消息路由细节

```
                     GroupChat 内部消息路由

  ┌──────────────────────────────────────────────────────────────────────┐
  │ 1. Manager 收到外部消息 (或 run_chat 初始消息)                      │
  │                                                                      │
  │    外部消息: {"content": "设计一个网页爬虫", "role": "user"}          │
  │    groupchat.messages = [ext_msg]                                    │
  │                                                                      │
  │ 2. Manager.select_speaker(last_speaker="Admin")                      │
  │    → round_robin: Agent_A (Researcher)                              │
  │                                                                      │
  │ 3. Manager 发送消息给 Agent_A                                        │
  │    Agent_A.send(groupchat.messages, Manager)                         │
  │    → Agent_A.receive(groupchat.messages, Manager)                   │
  │      ├─ chat_messages[Manager] += groupchat.messages (整个历史)      │
  │      ├─ generate_reply() → LLM                                      │
  │      └─ send(reply, Manager)                                         │
  │                                                                      │
  │ 4. Manager 收到 Agent_A 的回复                                       │
  │    groupchat.messages.append(Agent_A_reply)                          │
  │    last_speaker = Agent_A                                            │
  │                                                                      │
  │ 5. Manager.select_speaker(last_speaker=Agent_A)                      │
  │    → round_robin: Agent_B (Coder)                                   │
  │                                                                      │
  │    问题: Manager 发什么给 Agent_B?                                   │
  │    发 groupchat.messages (完整历史) 还是只发最后一条?                 │
  │                                                                      │
  │    答案: 发完整历史 (groupchat.messages)                              │
  │    因为每个 Agent 需要看到完整的对话上下文才能做出响应                │
  │                                                                      │
  │    但这也意味着: 每次发言都在发送整个历史                             │
  │    对于长对话, token 消耗和延迟都会线性增长                           │
  │                                                                      │
  │ 6. Agent_B.receive(完整历史, Manager)                                │
  │    → chat_messages[Manager] += 完整历史                              │
  │    → LLM.generate(完整历史)                                          │
  │    → send(reply, Manager)                                            │
  │                                                                      │
  │ 7. 重复直到 max_round 或 TERMINATE                                   │
  │                                                                      │
  │ 8. 终止条件:                                                         │
  │    a) len(groupchat.messages) >= max_round * len(agents)            │
  │    b) 某条消息包含 "TERMINATE"                                       │
  │    c) 自定义终止函数返回 True                                        │
  └──────────────────────────────────────────────────────────────────────┘

  关键理解:
    - GroupChatManager 是"交通警察", 不是"中央处理器"
    - 它不修改 Agent 的输出, 只决定谁拿到麦克风
    - 所有 Agent 共享完整对话历史, 因此能看到对方的输出
    - 每个 Agent 独立决定"如何回复当前上下文"
```

---

## 参考

- [AutoGen GitHub](https://github.com/microsoft/autogen)
- [AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation](https://arxiv.org/abs/2308.08155) (Wu et al., 2023)
- [ConversableAgent Source](https://github.com/microsoft/autogen/blob/main/autogen/agentchat/conversable_agent.py)
- [GroupChat Source](https://github.com/microsoft/autogen/blob/main/autogen/agentchat/groupchat.py)
- [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling)
