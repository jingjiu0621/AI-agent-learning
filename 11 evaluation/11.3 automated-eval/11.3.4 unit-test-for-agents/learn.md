# 11.3.4 unit-test-for-agents — Agent 单元测试

## 简单介绍

Agent 单元测试指的是对 Agent 系统中的**单个组件或单一决策步骤**进行隔离验证，而不是对完整的 Agent 运行循环做端到端测试。核心思想是：**把 Agent 拆解成可独立测试的小单元——工具选择逻辑、参数提取、LLM 响应解析、状态更新——然后逐个验证它们的正确性。**

```
传统软件单元测试:
   函数 ──输入──→ 执行逻辑 ──→ 断言输出
   确定性、可重复、无外部依赖

Agent 单元测试:
   Agent 组件 ──输入──→ 模拟依赖 ──→ 断言组件行为
   需要 Mock LLM、Mock 工具、控制状态
   核心挑战：非确定性输出 + 外部依赖 + 状态耦合
```

与传统单元测试的关键区别在于——Agent 的核心决策引擎是 LLM，而 LLM 的输出本质上是**非确定性的（non-deterministic）**。这使得传统的"输入-断言输出"模式不能直接套用。Agent 单元测试的应对策略是：**把非确定性部分 Mock 掉，只测试外围确定性逻辑。**

```
Agent 系统中的哪些可以单元测试？

  可以测试（确定性逻辑）:
    ✓ 工具选择逻辑：给定状态，Agent 选择了哪个工具
    ✓ 参数提取：从 LLM 响应中解析出的工具参数
    ✓ 状态更新：执行工具后状态如何变化
    ✓ 错误处理：工具返回错误时 Agent 的反应
    ✓ Prompt 模板：模板是否正确渲染了变量
    ✓ 输出解析器：LLM 的原始输出是否正确解析成结构化数据

  不可直接测试（非确定性逻辑）:
    ✗ LLM 生成的具体文本内容
    ✗ LLM 的"思考"过程是否最优
    ✗ 开放式回答的语义质量
    ✗ 多步 Agent 的路径选择（这属于集成测试范围）
```

## 基本原理

### Agent 单元测试的四根支柱

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     Agent 单元测试框架                                     │
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ LLM Mocking  │  │Tool Mocking  │  │State Testing │  │Cycle Testing │  │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤  ├──────────────┤  │
│  │  Mock LLM    │  │  Mock 外部    │  │  给定初始     │  │  Thought→    │  │
│  │  响应为确定   │  │  API/工具     │  │  状态 → 执    │  │  Action→     │  │
│  │  性输出      │  │  返回值      │  │  行动作 →     │  │  Observation │  │
│  │             │  │             │  │  验证状态变    │  │  每一步隔离   │  │
│  │             │  │             │  │  化是否符      │  │  验证        │  │
│  │             │  │             │  │  合预期       │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                                          │
│  核心原则：隔离 (Isolation)                                                │
│  "测试一个组件时，其他所有组件都应该被 Mock"                                │
└──────────────────────────────────────────────────────────────────────────┘
```

### 支柱一：LLM Mocking

LLM Mocking 是 Agent 单元测试的基石。其核心思路是：**不实际调用 LLM，而是用预设的响应替换 LLM 输出。** 这样就能在确定性环境中验证 Agent 的外围逻辑。

```
LLM Mocking 的本质：

  在不调用真实 LLM 的前提下，控制"Agent 的下一步决策"。
  
  真实场景：
    LLM ──→ "需要查询天气" ──→ 调用天气工具
  
  Mock 场景：
    MockLLM ──→ 返回 "需要查询天气" ──→ 验证 Agent 调用了天气工具
  
  通过 Mock，我们把"LLM 是否做出了正确的决定"这个问题，
  转换为"给定 LLM 做出了某个决定，Agent 是否正确地执行了这个决定"。
```

### 支柱二：Tool Mocking

Agent 的工具调用涉及外部系统——数据库、API、文件系统、其他服务。Tool Mocking 让测试不依赖这些外部系统的可用性和状态。

```
Tool Mocking 层次：

  Level 1: 返回值 Mock
    工具函数根本不执行，直接返回预设值
    例：search_database() 直接返回 [mock_result]
  
  Level 2: 行为 Mock
    模拟工具的副作用（超时、错误、限流）
    例：search_database() 第一次调用抛超时，第二次返回正常结果
  
  Level 3: 调用验证 Mock
    不关心返回值，只关心 Agent 是否按预期调用了工具
    例：验证 Agent 确实用正确参数调用了 search_database()
```

### 支柱三：State Testing

Agent 是有状态的——它维护当前的执行上下文、已收集的信息、决策历史。State Testing 验证状态在每次操作后是否正确更新。

```
Agent 状态变化示例：

  初始状态:
    { "messages": [user_query], "collected_data": {}, "step": 0 }
  
  工具调用后:
    { "messages": [user_query, tool_result], 
      "collected_data": {"weather": "sunny"}, 
      "step": 1 }
  
  LLM 响应后:
    { "messages": [user_query, tool_result, llm_response],
      "collected_data": {"weather": "sunny"},
      "step": 1, "final_answer": "今天是晴天" }
```

### 支柱四：Thought/Action/Observation 循环测试

ReAct 模式的 Agent 遵循"思考-行动-观察"的循环。单元测试可以隔离验证每个循环步骤。

```
┌──────────────────────────────────────────────────────────────────────┐
│  ReAct 循环的单元测试                                                  │
│                                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐                   │
│  │ Thought  │───→│  Action  │───→│ Observation  │                   │
│  │  步骤    │    │  步骤    │    │   步骤       │                   │
│  └──────────┘    └──────────┘    └──────────────┘                   │
│       │               │               │                             │
│       ▼               ▼               ▼                             │
│  测试:            测试:            测试:                            │
│  LLM 输出是否     Action 结构      Observation 是                     │
│  包含有效的       是否正确？       否被正确记录                        │
│  Action 字段？   参数是否完整？    到状态中？                          │
│                                                                      │
│  每个步骤单独测试，用 Mock 切断对其他步骤的依赖                        │
└──────────────────────────────────────────────────────────────────────┘
```

## Agent 单元测试的核心挑战与应对

### 挑战一：非确定性输出

LLM 的输出本质上是非确定性的——同样的 Prompt 每次调用可能给出不同回答。

```
问题：
  Agent 接收相同的输入，两次运行可能选择不同的工具、
  使用不同的参数、给出不同的回答。
  
  传统的 Assert 模式：assert output == expected
  在 Agent 单元测试中不成立。

解决策略：
  1. Mock LLM 响应，让输出完全确定
  2. 只断言"结构"而非"内容"——检查 JSON 结构而非具体值
  3. 使用模糊匹配——检查预期字段是否包含在输出中
  4. 使用分类验证——断言输出属于某一类别而非具体值
```

```python
# 策略 1: Mock LLM 响应
def test_agent_with_mock_llm():
    mock_llm = MockLLM(responses=[
        {"role": "assistant", "content": "我需要查询天气，使用 get_weather 工具"}
    ])
    agent = Agent(llm=mock_llm, tools=[get_weather])
    result = agent.run("今天天气怎么样？")
    # 断言 Agent 没有调用真实 LLM，但正确处理了 Mock 响应
    assert result.tool_called == "get_weather"

# 策略 2: 只断言结构
def test_agent_output_structure():
    mock_llm = MockLLM(responses=[
        {"role": "assistant", "content": json.dumps({
            "thought": "用户想知道天气，我需要查询",
            "action": "get_weather",
            "action_input": {"city": "北京"}
        })}
    ])
    agent = Agent(llm=mock_llm, tools=[get_weather])
    action = agent.extract_action()
    assert "action" in action  # 只断言字段存在
    assert "action_input" in action  # 不断言具体值
```

### 挑战二：状态依赖

Agent 的行为高度依赖当前状态——历史消息、已收集的数据、执行步骤数。

```
问题：
  一个工具调用是否成功，取决于之前的工具调用结果。
  没有正确的状态上下文，单元测试就不可靠。

解决策略：
  1. 显式构造初始状态，测试时完全控制输入
  2. 状态作为参数传入，而非从全局获取
  3. 每个测试用例构造最小但完整的上下文
```

```python
def test_agent_state_mutation():
    # 显式构造完整初始状态
    initial_state = AgentState(
        messages=[HumanMessage(content="北京的天气怎么样？")],
        collected_data={},
        step=0,
        available_tools=["get_weather", "get_time"]
    )
    
    mock_llm = MockLLM(responses=[
        {"action": "get_weather", "action_input": {"city": "北京"}}
    ])
    
    agent = Agent(llm=mock_llm)
    
    # 测试在给定状态下，Agent 的状态更新逻辑
    new_state = agent.step(initial_state)
    
    assert new_state.step == 1
    assert "get_weather" in new_state.tool_history
```

### 挑战三：环境耦合

Agent 经常调用外部环境——API、数据库、文件系统、网络请求。这些环境在测试中不可控。

```
解决策略：
  1. 依赖注入：工具作为参数传入 Agent，而非硬编码
  2. Mock 替换：在测试中传入 Mock 工具替换真实工具
  3. 接口抽象：定义工具接口，生产和测试使用不同实现

依赖注入模式：

  # 不好的做法：工具在 Agent 内部硬编码
  class WeatherAgent:
      def __init__(self):
          self.tools = [RealWeatherAPI()]  # ❌ 无法替换
  
  # 好的做法：工具从外部注入
  class WeatherAgent:
      def __init__(self, tools):  # ✅ 可注入 Mock
          self.tools = tools
```

### 挑战四：异步工具调用

现代 Agent 框架通常使用异步调用（asyncio），这给测试带来额外复杂性。

```python
# 异步 Agent 的单元测试模式

import pytest
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_async_agent():
    # AsyncMock 可以模拟异步工具
    mock_search = AsyncMock(return_value={"results": ["item1"]})
    
    agent = AsyncAgent(tools=[mock_search])
    result = await agent.run("查找商品")
    
    mock_search.assert_awaited_once_with(query="查找商品")
    assert result.success is True

@pytest.mark.asyncio
async def test_async_tool_timeout():
    # 模拟超时场景
    mock_slow_tool = AsyncMock(side_effect=asyncio.TimeoutError)
    
    agent = AsyncAgent(tools=[mock_slow_tool])
    result = await agent.run("查询数据")
    
    assert result.error_type == "timeout"
    assert result.recovery_action == "retry"
```

## Mock 策略

### LLM 响应 Mock 的层次

```
LLM Mock 可以分为三个层次，每个层次的模拟程度不同：

  Layer 1: 简单字符串 Mock
    直接返回预设的字符串作为 LLM 输出
    适用场景：测试 Prompt 渲染、输出解析器
    优点：最简单，最可控
    缺点：无法模拟复杂的响应结构

  Layer 2: 结构化 Mock  
    返回包含多个字段的字典/对象（模拟真实 LLM 的结构化输出）
    适用场景：测试工具调用解析、参数提取
    优点：接近真实 LLM 的输出格式
    缺点：需要手动构造响应

  Layer 3: 行为 Mock
    根据输入动态决定返回什么响应
    适用场景：测试条件逻辑、多轮对话
    优点：能模拟 LLM 的动态行为
    缺点：复杂度高，需要写条件逻辑
```

### 各层次 Mock 代码示例

```python
# ============================================================
# Layer 1: 简单字符串 Mock
# ============================================================
class SimpleMockLLM:
    """最简单的 Mock：总是返回预设字符串"""
    
    def __init__(self, response: str):
        self.response = response
        self.call_count = 0
        self.call_history = []
    
    def chat(self, messages: list[dict]) -> dict:
        self.call_count += 1
        self.call_history.append(messages)
        return {
            "role": "assistant",
            "content": self.response
        }

# 使用场景：测试 Prompt 是否被正确构造
def test_prompt_rendering():
    mock_llm = SimpleMockLLM("我需要查询数据库")
    
    agent = Agent(llm=mock_llm, system_prompt="你是{role}助手")
    agent.run("你好")
    
    # 验证传给 LLM 的 messages 中是否正确渲染了 Prompt
    sent_prompt = mock_llm.call_history[0][0]["content"]
    assert "{role}" not in sent_prompt  # 变量应已被替换
    assert "助手" in sent_prompt  # 验证渲染结果

# ============================================================
# Layer 2: 结构化 Mock
# ============================================================
class StructuredMockLLM:
    """返回结构化输出（如 JSON），模拟真实 Agent LLM 的响应"""
    
    def __init__(self, responses: list[dict]):
        self.responses = responses
        self.index = 0
    
    def chat(self, messages: list[dict]) -> dict:
        response = self.responses[self.index % len(self.responses)]
        self.index += 1
        # 模拟 LLM 返回的 JSON 格式
        return {
            "role": "assistant",
            "content": json.dumps(response)
        }

# 使用场景：测试工具选择和参数提取
def test_tool_selection_with_structured_mock():
    mock_responses = [
        {
            "thought": "用户要查天气",
            "action": "get_weather",
            "action_input": {"city": "上海", "unit": "celsius"}
        },
        {
            "thought": "用户要查时间",
            "action": "get_time",
            "action_input": {"timezone": "Asia/Shanghai"}
        }
    ]
    mock_llm = StructuredMockLLM(mock_responses)
    
    agent = Agent(llm=mock_llm, tools=[get_weather, get_time])
    
    # 第一轮：验证工具选择
    action = agent.extract_action()
    assert action["name"] == "get_weather"
    assert action["params"]["city"] == "上海"
    assert action["params"]["unit"] == "celsius"
    
    # 执行工具调用（这里 mock_weather_tool 也被 Mock 了）
    agent.execute_tool(action)
    
    # 第二轮：验证下一个工具选择
    next_action = agent.extract_action()
    assert next_action["name"] == "get_time"

# ============================================================
# Layer 3: 行为 Mock（条件响应 Mock）
# ============================================================
class ConditionalMockLLM:
    """根据输入内容动态返回不同响应"""
    
    def __init__(self, response_map: list[tuple]):
        """
        response_map: [(condition_fn, response), ...]
        按顺序匹配，第一个条件为 True 的响应被返回
        """
        self.response_map = response_map
    
    def chat(self, messages: list[dict]) -> dict:
        last_message = messages[-1]["content"] if messages else ""
        
        for condition_fn, response in self.response_map:
            if condition_fn(last_message):
                return {
                    "role": "assistant",
                    "content": json.dumps(response)
                }
        
        # 默认响应
        return {"role": "assistant", "content": "我无法处理这个请求"}

# 使用场景：测试 Agent 对不同输入的条件行为
def test_conditional_behavior():
    def has_weather_keyword(msg: str) -> bool:
        return "天气" in msg or "weather" in msg.lower()
    
    def has_time_keyword(msg: str) -> bool:
        return "时间" in msg or "time" in msg.lower()
    
    mock_llm = ConditionalMockLLM([
        (has_weather_keyword, {"action": "get_weather", "params": {}}),
        (has_time_keyword, {"action": "get_time", "params": {}}),
    ])
    
    agent = Agent(llm=mock_llm)
    
    # 测试天气查询
    weather_action = agent.get_action("今天天气好吗？")
    assert weather_action["action"] == "get_weather"
    
    # 测试时间查询
    time_action = agent.get_action("现在几点了？")
    assert time_action["action"] == "get_time"
```

### 工具调用 Mock

```python
from unittest.mock import MagicMock, patch

# ============================================================
# 基础工具 Mock
# ============================================================
def create_mock_tool(name: str, return_value=None):
    """创建 Mock 工具的统一工厂函数"""
    mock_tool = MagicMock()
    mock_tool.name = name
    mock_tool.description = f"Mock {name} tool"
    mock_tool.__call__ = MagicMock(return_value=return_value)
    return mock_tool

# 使用
def test_agent_executes_tool():
    mock_calc = create_mock_tool("calculator", return_value=42)
    
    agent = Agent(tools=[mock_calc])
    result = agent.execute_tool("calculator", {"expression": "6*7"})
    
    assert result == 42
    mock_calc.assert_called_once_with(expression="6*7")

# ============================================================
# 带副作用的 Mock（模拟错误场景）
# ============================================================
def test_agent_error_handling():
    """模拟工具抛出异常，验证 Agent 的错误处理"""
    
    # 第一次调用抛错，第二次正常返回
    mock_db = MagicMock()
    mock_db.__call__ = MagicMock(
        side_effect=[ConnectionError("DB down"), {"data": "ok"}]
    )
    
    agent = Agent(tools=[mock_db])
    
    # 第一次执行应该触发错误处理
    result1 = agent.execute_tool("query_db", {"sql": "SELECT *"})
    assert result1.status == "error"
    assert agent.consecutive_errors == 1
    
    # 第二次应该正常返回
    result2 = agent.execute_tool("query_db", {"sql": "SELECT *"})
    assert result2.status == "success"
    assert result2.data == {"data": "ok"}

# ============================================================
# 调用验证（Spy 模式）
# ============================================================
def test_agent_tool_call_parameters():
    """验证 Agent 传给工具的参数是否正确"""
    
    mock_search = MagicMock(return_value=[])
    
    agent = Agent(tools=[mock_search])
    agent.execute_tool("search", {
        "query": "机器学习",
        "limit": 10,
        "filters": {"lang": "zh"}
    })
    
    # 验证参数正确传递
    mock_search.assert_called_once_with(
        query="机器学习",
        limit=10,
        filters={"lang": "zh"}
    )

# ============================================================
# 模拟多个工具返回不同结果
# ============================================================
def test_multi_tool_scenario():
    """模拟多工具场景中每个工具的不同行为"""
    
    tools = {
        "search": MagicMock(return_value=["result1", "result2"]),
        "summarize": MagicMock(return_value="这是摘要"),
        "translate": MagicMock(side_effect=RuntimeError("翻译服务不可用"))
    }
    
    agent = Agent(tools=list(tools.values()))
    
    # 验证每个工具的行为
    search_result = agent.execute_tool("search", {"query": "AI"})
    assert len(search_result) == 2
    
    summarize_result = agent.execute_tool("summarize", {"text": "long text..."})
    assert "摘要" in summarize_result
    
    # 验证翻译工具的异常被正确捕获
    translate_result = agent.execute_tool("translate", {"text": "hello"})
    assert "不可用" in translate_result  # Agent 应返回友好的错误消息
```

### 流式响应 Mock

现代 Agent 框架中，LLM 响应可能是流式的（streaming）。测试流式处理逻辑需要专门的 Mock。

```python
# ============================================================
# 流式 LLM Mock
# ============================================================
class StreamingMockLLM:
    """模拟 LLM 的流式响应"""
    
    def __init__(self, chunks: list[str]):
        self.chunks = chunks
    
    async def chat_stream(self, messages: list[dict]):
        """异步生成器，逐块返回预设的流式内容"""
        for chunk in self.chunks:
            yield {"type": "content", "delta": chunk}
            await asyncio.sleep(0.01)  # 模拟网络延迟
        yield {"type": "done"}

@pytest.mark.asyncio
async def test_streaming_response_handling():
    chunks = [
        "我", "需要", "查询", "天气",
        "，", "使", "用", "工", "具"
    ]
    mock_llm = StreamingMockLLM(chunks)
    
    agent = StreamingAgent(llm=mock_llm)
    
    full_response = ""
    async for chunk in agent.run_stream("天气"):
        if chunk["type"] == "content":
            full_response += chunk["delta"]
    
    assert "查询天气" in full_response
    assert "使用工具" in full_response

# ============================================================
# 流式工具结果 Mock
# ============================================================
class MockStreamingTool:
    """模拟返回流式数据的工具"""
    
    async def __call__(self, **params):
        for i in range(3):
            yield {"progress": i * 33, "partial": f"chunk_{i}"}
            await asyncio.sleep(0.01)
        yield {"progress": 100, "done": True}

@pytest.mark.asyncio
async def test_streaming_tool_consumption():
    mock_tool = MockStreamingTool()
    agent = StreamingAgent(tools=[mock_tool])
    
    results = []
    async for chunk in agent.execute_tool_stream("mock_tool", {}):
        results.append(chunk)
    
    assert results[-1]["done"] is True
    assert results[-1]["progress"] == 100
```

## 测试框架与工具

### 核心框架选型

```
Agent 单元测试的常用工具栈：

  pytest              —— 基础测试框架
  pytest-asyncio      —— 异步测试支持（async Agent 必备）
  unittest.mock       —— 内置 Mock 工具（MagicMock, AsyncMock, patch）
  pytest-mock         —— pytest 风格的 Mock（mocker fixture）
  syrupy              —— 快照测试（snapshot testing）
  vcrpy               —— HTTP 请求录制/回放
  betch               —— LLM 调用录制/回放
  ward                —— 现代化测试框架（比 pytest 更严格的隔离）
```

### pytest 配置与最佳实践

```python
# conftest.py - Agent 测试的共享配置

import pytest
from unittest.mock import MagicMock

# ============================================================
# Fixture: 共享 Mock LLM
# ============================================================
@pytest.fixture
def mock_llm():
    """为所有测试提供统一的 Mock LLM"""
    return MagicMock(
        chat=MagicMock(return_value={
            "role": "assistant",
            "content": '{"action": "noop", "reason": "test"}'
        })
    )

# ============================================================
# Fixture: 共享 Mock 工具集
# ============================================================
@pytest.fixture
def mock_tools():
    """为测试提供一组预配置的 Mock 工具"""
    return [
        MagicMock(name="search", return_value=["doc1", "doc2"]),
        MagicMock(name="calculate", return_value=42),
        MagicMock(name="translate", return_value="hello"),
    ]

# ============================================================
# Fixture: 预构造的 Agent 实例
# ============================================================
@pytest.fixture
def agent(mock_llm, mock_tools):
    """创建一个注入 Mock 依赖的 Agent 实例"""
    return Agent(
        llm=mock_llm,
        tools=mock_tools,
        system_prompt="你是测试助手",
        max_steps=3
    )

# ============================================================
# Fixture: 各种测试用的状态
# ============================================================
@pytest.fixture
def empty_state():
    """空的初始状态"""
    return AgentState(messages=[], step=0)

@pytest.fixture
def mid_conversation_state():
    """对话中间状态（已有一轮交互）"""
    return AgentState(
        messages=[
            {"role": "user", "content": "北京的天气"},
            {"role": "assistant", "content": "已查询天气"},
            {"role": "tool", "content": "晴，22°C"},
        ],
        step=1,
        collected_data={"weather": "sunny", "temperature": 22}
    )
```

### VCR 录制模式（Recording LLM Calls）

VCR（Video Cassette Recording）模式是指**第一次运行测试时记录真实的 LLM 调用，后续测试重放记录**。这种方式结合了真实调用和 Mock 的优点。

```python
# ============================================================
# 使用 vcrpy 录制 LLM API 调用
# ============================================================
import vcr

# 配置 VCR：录制真实的 LLM API 调用
llm_vcr = vcr.VCR(
    cassette_library_dir="tests/cassettes/",  # 录制文件存放位置
    record_mode="once",        # once: 第一次录制，之后重放
    # new_episodes: 有新的请求就追加录制
    # all: 每次都重新录制（覆盖）
    # none: 只重放，不录制
    match_on=["method", "scheme", "host", "port", "path", "query", "body"],
    filter_headers=["authorization"],  # 过滤掉 API key
    filter_post_data_parameters=["api_key"],  # 过滤敏感参数
)

@llm_vcr.use_cassette("tests/cassettes/weather_query.yaml")
def test_agent_with_recorded_llm():
    """使用录制的 LLM 响应测试 Agent"""
    agent = Agent(llm=OpenAILLM(api_key="test-key"))  # VCR 会拦截真实调用
    result = agent.run("北京的天气怎么样？")
    
    # 第一次运行：真实调用 LLM 并录制
    # 之后运行：重放录制的响应，不调用真实 LLM
    assert result.success

# ============================================================
# betch: 专为 LLM 调用设计的录制/回放工具
# ============================================================
# betch 是更专用的 LLM VCR 工具，支持更细粒度的匹配策略

"""
betch 的核心概念：

  Cassette: 存储一组录制的 LLM 调用
  Matcher: 决定如何匹配请求
    精确匹配：相同的 Prompt → 返回相同的响应
    语义匹配：相似的 Prompt → 返回相似的响应
    模板匹配：符合模板的 Prompt → 返回预设响应

  使用示例：
    with betch.use_cassette("test_cassette"):
        # 这里的 LLM 调用会被录制/回放
        response = llm.chat(messages)
    
    # 第一次执行：录制
    # 后面的执行：匹配并回放
"""
```

### LangGraph 自带的测试工具

LangGraph 等 Agent 框架提供了内置的测试实用工具，简化了图结构的单元测试。

```python
# ============================================================
# LangGraph 的测试工具
# ============================================================

from langgraph.graph import StateGraph

# LangGraph 的测试模式允许你：
# 1. 在图的任意节点注入 Mock 值
# 2. 验证节点间的状态传递
# 3. 测试单个节点的输入输出

def test_langgraph_node_isolation():
    """测试 LangGraph 中单个节点的行为"""
    
    def my_node(state: dict) -> dict:
        """示例节点：累加 count"""
        return {"count": state.get("count", 0) + 1}
    
    # 在隔离环境中测试节点
    # LangGraph 的测试工具允许直接调用节点函数
    result = my_node({"count": 5})
    assert result["count"] == 6
    
    # 测试空状态
    result = my_node({})
    assert result["count"] == 1

def test_langgraph_edge_condition():
    """测试条件边的判断逻辑"""
    
    def should_continue(state: dict) -> str:
        """条件边：根据状态决定下一个节点"""
        if state["step"] >= state.get("max_steps", 5):
            return "end"
        return "continue"
    
    # 测试"继续"条件
    assert should_continue({"step": 3, "max_steps": 5}) == "continue"
    
    # 测试"结束"条件
    assert should_continue({"step": 5, "max_steps": 5}) == "end"
    
    # 测试默认值
    assert should_continue({"step": 3}) == "continue"  # max_steps 默认 5
    assert should_continue({"step": 5}) == "end"
```

### 自定义测试工具集

```python
# ============================================================
# 自定义 Agent 测试 Harness
# ============================================================

class AgentTestHarness:
    """Agent 单元测试的统一脚手架"""
    
    def __init__(self, agent_class, mock_llm=None, mock_tools=None):
        self.agent_class = agent_class
        self.mock_llm = mock_llm or MagicMock()
        self.mock_tools = mock_tools or []
        self.agent = None
    
    def create_agent(self, **overrides):
        """创建注入 Mock 依赖的 Agent 实例"""
        self.agent = self.agent_class(
            llm=overrides.pop("llm", self.mock_llm),
            tools=overrides.pop("tools", self.mock_tools),
            **overrides
        )
        return self.agent
    
    def given_mock_llm_response(self, response: str):
        """设置 Mock LLM 的下一次响应"""
        self.mock_llm.chat.return_value = {
            "role": "assistant",
            "content": response
        }
    
    def given_mock_llm_responses(self, responses: list[str]):
        """设置 Mock LLM 的多次响应"""
        self.mock_llm.chat.side_effect = [
            {"role": "assistant", "content": r} for r in responses
        ]
    
    def given_mock_tool_result(self, tool_name: str, result):
        """设置特定工具的 Mock 返回值"""
        for tool in self.mock_tools:
            if tool.name == tool_name:
                tool.__call__.return_value = result
                return
        raise ValueError(f"Tool {tool_name} not found")
    
    def assert_tool_called(self, tool_name: str, **expected_params):
        """断言某个工具被调用，且参数符合预期"""
        for tool in self.mock_tools:
            if tool.name == tool_name:
                tool.assert_called_with(**expected_params)
                return
        raise AssertionError(f"Tool {tool_name} was never called")
    
    def assert_state_contains(self, state: dict, **expected_fields):
        """断言状态包含指定字段"""
        for key, value in expected_fields.items():
            assert key in state, f"State missing key: {key}"
            assert state[key] == value, \
                f"State[{key}] = {state[key]}, expected {value}"

# 使用示例
def test_harness_usage():
    harness = AgentTestHarness(MyAgent)
    harness.given_mock_llm_response('{"action": "search", "params": {"q": "AI"}}')
    harness.given_mock_tool_result("search", ["result1"])
    
    agent = harness.create_agent()
    result = agent.run("搜索AI相关")
    
    harness.assert_tool_called("search", q="AI")
    harness.assert_state_contains(
        result.state,
        step=1,
        last_action="search"
    )
```

## 具体测试模式

### 模式一：工具选择测试（Tool Selection Test）

验证 Agent 在给定场景下是否选择了正确的工具。

```python
def test_tool_selection_for_weather_query():
    """
    场景：用户问天气
    预期：Agent 选择 get_weather 工具
    """
    mock_llm = MagicMock()
    mock_llm.chat.return_value = {
        "role": "assistant",
        "content": json.dumps({
            "thought": "用户想查询天气",
            "action": "get_weather",
            "action_input": {"city": "北京"}
        })
    }
    
    agent = Agent(
        llm=mock_llm,
        tools=[get_weather, get_news, calculator]
    )
    
    # 模拟用户输入
    action = agent.plan("北京今天天气怎么样？")
    
    assert action["name"] == "get_weather", \
        f"Expected get_weather, got {action['name']}"
    assert "city" in action["params"], "Missing city parameter"
    
    # 验证 LLM 收到了包含工具描述的 Prompt
    sent_messages = mock_llm.chat.call_args[0][0]
    tool_descriptions = " ".join(sent_messages[-1]["content"])
    assert "get_weather" in tool_descriptions

def test_tool_selection_with_ambiguous_input():
    """
    场景：用户输入模糊
    预期：Agent 遵循"不确定时询问"的策略
    """
    mock_llm = MagicMock()
    mock_llm.chat.return_value = {
        "role": "assistant",
        "content": json.dumps({
            "thought": "用户意图不明确，需要澄清",
            "action": "ask_clarification",
            "action_input": {
                "question": "您想查询什么信息？天气、新闻还是其他？"
            }
        })
    }
    
    agent = Agent(
        llm=mock_llm,
        tools=[get_weather, get_news, ask_clarification]
    )
    
    action = agent.plan("帮我查点东西")  # 模糊输入
    
    assert action["name"] == "ask_clarification", \
        "Agent should ask for clarification on ambiguous input"
```

### 模式二：参数提取测试（Parameter Extraction Test）

验证 Agent 从用户输入中提取的工具参数是否正确。

```python
def test_parameter_extraction_complete():
    """
    场景：用户提供了完整的参数信息
    预期：Agent 正确提取所有参数
    """
    mock_llm = MagicMock()
    mock_llm.chat.return_value = {
        "role": "assistant",
        "content": json.dumps({
            "action": "book_flight",
            "action_input": {
                "from_city": "北京",
                "to_city": "上海",
                "date": "2025-12-25",
                "passengers": 2,
                "class": "economy"
            }
        })
    }
    
    agent = Agent(llm=mock_llm, tools=[book_flight])
    action = agent.plan(
        "帮我订12月25号从北京到上海的机票，两个人，经济舱"
    )
    
    assert action["params"]["from_city"] == "北京"
    assert action["params"]["to_city"] == "上海"
    assert action["params"]["date"] == "2025-12-25"
    assert action["params"]["passengers"] == 2

def test_parameter_extraction_partial():
    """
    场景：用户提供了部分参数
    预期：Agent 只提取已有参数，不捏造不存在的信息
    """
    mock_llm = MagicMock()
    mock_llm.chat.return_value = {
        "role": "assistant",
        "content": json.dumps({
            "action": "book_flight",
            "action_input": {
                "from_city": "北京",
                "to_city": None,  # 用户没提供
                "date": "2025-12-25",
                "passengers": 1
            }
        })
    }
    
    agent = Agent(llm=mock_llm, tools=[book_flight])
    action = agent.plan("12月25号从北京出发的机票")
    
    # 验证：Agent 没有捏造目的地
    assert action["params"]["to_city"] is None, \
        "Should not fabricate missing parameters"
    
    # 验证：已知参数被正确提取
    assert action["params"]["from_city"] == "北京"
```

### 模式三：错误处理测试（Error Handling Test）

验证 Agent 在工具出错时是否能优雅地恢复。

```python
def test_error_handling_tool_unavailable():
    """
    场景：工具临时不可用（网络问题、服务宕机）
    预期：Agent 捕获错误，给出友好提示，提供替代方案
    """
    # 模拟调用数据库工具时抛出异常
    mock_db = MagicMock()
    mock_db.name = "query_database"
    mock_db.__call__ = MagicMock(
        side_effect=ConnectionError("数据库连接超时")
    )
    
    agent = Agent(llm=MagicMock(), tools=[mock_db])
    
    result = agent.execute_tool("query_database", {"sql": "SELECT *"})
    
    assert result.status == "error"
    assert "connection" in result.error_message.lower() or \
           "超时" in result.error_message

def test_error_handling_invalid_params():
    """
    场景：工具参数验证失败（格式错误、值超出范围）
    预期：Agent 验证参数，返回明确的错误信息
    """
    mock_calc = MagicMock()
    mock_calc.name = "calculator"
    
    agent = Agent(llm=MagicMock(), tools=[mock_calc])
    
    # 测试空参数
    result = agent.execute_tool("calculator", {})
    assert result.status == "error"
    assert "参数" in result.error_message or "parameter" in result.error_message.lower()
    
    # 测试非法参数
    result = agent.execute_tool("calculator", {"expression": "1/0"})
    assert result.status == "error"

def test_error_handling_retry_logic():
    """
    场景：调用失败后自动重试
    预期：Agent 按策略进行重试，在达到重试上限后返回错误
    """
    mock_api = MagicMock()
    mock_api.name = "call_api"
    # 前两次失败，第三次成功
    mock_api.__call__ = MagicMock(
        side_effect=[
            TimeoutError("timeout"),
            TimeoutError("timeout again"),
            {"status": 200, "data": "success"}
        ]
    )
    
    agent = Agent(
        llm=MagicMock(),
        tools=[mock_api],
        max_retries=3
    )
    
    result = agent.execute_tool_with_retry("call_api", {"endpoint": "/data"})
    
    assert result.status == "success"
    assert mock_api.__call__.call_count == 3  # 重试了两次
```

### 模式四：状态变更测试（State Mutation Test）

验证 Agent 在执行操作后状态是否按预期更新。

```python
def test_state_mutation_after_tool_call():
    """
    场景：工具调用后状态更新
    预期：tool_history 增加记录，step 递增
    """
    mock_weather = MagicMock(
        name="get_weather",
        return_value={"temperature": 25, "condition": "晴"}
    )
    
    initial_state = AgentState(
        step=0,
        messages=[],
        tool_history=[],
        collected_data={}
    )
    
    agent = Agent(tools=[mock_weather])
    
    new_state = agent.step(
        initial_state,
        action={"name": "get_weather", "params": {"city": "北京"}}
    )
    
    assert new_state.step == 1, "Step should increment"
    assert len(new_state.tool_history) == 1, "Tool history should grow"
    assert new_state.tool_history[0]["tool"] == "get_weather"
    assert new_state.tool_history[0]["status"] == "success"
    assert new_state.collected_data.get("weather") is not None

def test_state_mutation_error_scenario():
    """
    场景：工具调用失败时状态更新
    预期：错误被记录，状态标记为错误
    """
    mock_broken = MagicMock(
        name="fragile_tool",
        side_effect=RuntimeError("unexpected error")
    )
    
    initial_state = AgentState(
        step=0,
        messages=[],
        tool_history=[],
        errors=[]
    )
    
    agent = Agent(tools=[mock_broken])
    
    new_state = agent.step(
        initial_state,
        action={"name": "fragile_tool", "params": {}}
    )
    
    assert new_state.tool_history[0]["status"] == "error"
    assert len(new_state.errors) == 1, "Error should be recorded"
    assert "unexpected error" in new_state.errors[0]
```

### 模式五：边界情况测试（Edge Case Test）

```python
def test_empty_user_input():
    """空输入：Agent 不应崩溃"""
    agent = Agent(llm=MagicMock(), tools=[default_tool])
    
    result = agent.run("")
    
    assert result.status != "crash"
    # 应该请求用户提供更多信息
    assert result.action == "ask_clarification" or result.status == "invalid_input"

def test_very_long_input():
    """超长输入：Agent 应截断或优雅处理"""
    mock_llm = MagicMock()
    agent = Agent(llm=mock_llm, tools=[], max_input_length=1000)
    
    long_text = "a" * 10000  # 10 倍于限制
    result = agent.run(long_text)
    
    # 验证传给 LLM 的输入被截断了
    sent_messages = mock_llm.chat.call_args[0][0]
    last_content = sent_messages[-1]["content"]
    assert len(last_content) <= 1000, "Input should be truncated"

def test_missing_required_params():
    """缺少必需参数：Agent 应能检测并请求补充"""
    mock_llm = MagicMock()
    mock_tool = MagicMock(name="send_email")
    # 工具声明了必需的 to 和 subject 参数
    mock_tool.required_params = ["to", "subject"]
    
    agent = Agent(llm=mock_llm, tools=[mock_tool])
    
    # Agent 尝试调用但缺少参数
    result = agent.validate_and_execute(
        "send_email", 
        {"to": "user@example.com"}  # 缺少 subject
    )
    
    assert result.status == "validation_error"
    assert "subject" in result.missing_params
    # Agent 应请求补充缺失参数

def test_special_characters_in_input():
    """特殊字符：Agent 应正确处理"""
    mock_llm = MagicMock()
    mock_llm.chat.return_value = {
        "role": "assistant",
        "content": json.dumps({
            "action": "search",
            "action_input": {"query": "SQL injection: DROP TABLE"}
        })
    }
    
    agent = Agent(llm=mock_llm, tools=[search_tool])
    
    # 输入中包含 SQL 注入尝试
    action = agent.plan("搜索 '; DROP TABLE users; --")
    
    # 验证参数被正确编码/转义
    assert action["params"]["query"] is not None
    
def test_concurrent_tool_calls():
    """并发调用：验证状态隔离"""
    mock_tool = MagicMock(
        name="slow_tool",
        return_value="done",
        side_effect=lambda **kw: asyncio.sleep(0.1) or "done"
    )
    
    agent = Agent(tools=[mock_tool])
    
    # 模拟快速连续调用
    import asyncio
    async def test():
        results = await asyncio.gather(
            agent.execute_tool("slow_tool", {"id": 1}),
            agent.execute_tool("slow_tool", {"id": 2}),
        )
        return results
    
    results = asyncio.run(test())
    assert len(results) == 2
    # 验证每次调用的参数隔离
```

## 单元测试 vs 集成测试边界

Agent 测试应该遵循**测试金字塔**原则，但在 Agent 场景下金字塔的形状与传统软件不同。

```
传统测试金字塔：
        /\          UI / E2E 测试（少）
       /  \         集成测试（中）
      /────\        单元测试（多）
     /──────\
    /────────\

Agent 测试金字塔：
        /\          端到端评估（少）
       /  \         集成测试（中多）
      /────\        单元测试（中）
     /──────\       Prompt/Eval 测试（多）
    /────────\

解释：Agent 的单元测试覆盖范围比传统软件窄，
因为核心决策逻辑（LLM）无法被单元测试覆盖。
所以需要更多的"集成测试"和"评估测试"来补足。
```

### 何时使用单元测试

```
✅ 适合单元测试的场景：
  1. 参数解析器：LLM 输出 → 结构化参数
  2. 状态管理：状态读取、更新、验证
  3. 工具调用包装：参数验证、返回值规范化
  4. 错误处理逻辑：重试、降级、错误格式化
  5. Prompt 模板：模板渲染、变量注入
  6. 条件分支：路由逻辑、停止条件判断
  7. 数据转换：工具输出 → 内部表示

  特征：不依赖 LLM 的真实输出，可以完全用 Mock 验证
```

### 何时使用集成测试

```
✅ 适合集成测试的场景：
  1. 工具与 LLM 的交互：LLM 是否选择了正确的工具
  2. 多轮对话流：状态在轮次之间是否正确传递
  3. 错误恢复链：LLM 在工具失败后是否调整策略
  4. 工具链执行：A 工具的输出是否是 B 工具的输入
  
  特征：涉及 LLM 的真实输出，但可以控制测试范围

❌ 不适合集成测试的场景：
  1. 开放式回答质量（应使用 LLM-as-Judge 评估）
  2. 长序列 Agent 行为（应使用端到端评估）
  3. 对抗性用例（应使用 Adversarial Testing）
```

```python
# 边界判定示例：同一功能的不同测试层次

# 单元测试版本：Mock LLM，只测参数解析逻辑
def test_email_parsing_unit():
    """单元测试：只测参数解析，不测 LLM 是否生成了正确的参数"""
    mock_llm = MagicMock()
    mock_llm.chat.return_value = {
        "content": json.dumps({
            "action": "send_email",
            "params": {
                "to": "test@example.com",
                "subject": "Hello",
                "body": "Test message"
            }
        })
    }
    agent = Agent(llm=mock_llm, tools=[])
    action = agent.parse_llm_output(mock_llm.chat.return_value)
    assert action["params"]["to"].count("@") == 1
    assert "subject" in action["params"]

# 集成测试版本：使用真实的 LLM（但控制模型和温度）
def test_email_extraction_integration():
    """集成测试：测 LLM 是否能从自然语言中正确提取参数"""
    real_llm = OpenAILight(model="gpt-4o-mini", temperature=0)
    agent = Agent(llm=real_llm, tools=[send_email_tool])
    
    # 多次运行取统计结果（因为 LLM 有随机性）
    results = []
    for _ in range(5):
        action = agent.plan("给张三发邮件，主题是项目进度，内容说一切正常")
        results.append(action["params"]["to"])
    
    # 大多数情况下应该提取出收件人
    valid_results = [r for r in results if r is not None]
    assert len(valid_results) >= 3, \
        "Most trials should extract the recipient"
```

## 工程最佳实践

### 测试隔离原则

```
Agent 单元测试的隔离原则：

  原则 1：不调真实 LLM
    所有测试都必须 Mock LLM 响应
    即使是"简单查询"也要 Mock
  
  原则 2：不调真实外部服务
    数据库、API、文件系统全部 Mock
    测试失败只应该因为代码有 bug，而不是因为外部服务挂了
  
  原则 3：状态完全可控
    测试开始时构造确切的状态
    不依赖前一个测试的遗留状态
  
  原则 4：时间无关性
    不依赖系统时间（用 freezegun 冻结时间）
    不依赖随机数（用固定 seed）
    不依赖并发执行顺序
```

### Fixture 设计模式

```python
# ============================================================
# Fixture 分层设计
# ============================================================

# conftest.py 中的 fixture 分层：

# Layer 1: 基础 Mock（全局可用）
@pytest.fixture(scope="session")
def base_mock_llm():
    """会话级 fixture：所有测试共享同一个基础 Mock"""
    return MagicMock()

@pytest.fixture(scope="session")
def base_tools():
    """会话级 fixture：基础工具集合"""
    return [
        MagicMock(name="search"),
        MagicMock(name="lookup"),
    ]

# Layer 2: 预配置 Agent（按需定制）
@pytest.fixture
def basic_agent(base_mock_llm, base_tools):
    """函数级 fixture：每个测试获得一个新的 Agent 实例"""
    mock_llm = copy.deepcopy(base_mock_llm)
    tools = copy.deepcopy(base_tools)
    return Agent(llm=mock_llm, tools=tools)

@pytest.fixture
def weather_agent(base_mock_llm):
    """专门的 weather 测试 Agent"""
    mock_llm = copy.deepcopy(base_mock_llm)
    weather_tool = MagicMock(name="get_weather")
    return Agent(llm=mock_llm, tools=[weather_tool])

# Layer 3: 场景状态（测试数据）
@pytest.fixture
def state_with_weather_data():
    """已经收集了天气数据的状态"""
    return AgentState(
        collected_data={
            "weather": {"city": "北京", "temp": 22, "condition": "晴"}
        },
        step=3,
        messages=[
            {"role": "user", "content": "北京天气"},
            {"role": "assistant", "content": "已查询"},
            {"role": "tool", "content": "晴，22°C"},
        ]
    )
```

### 测试数据管理

```python
# ============================================================
# 测试数据的管理策略
# ============================================================

# 策略 1: 使用数据类组织测试数据
from dataclasses import dataclass

@dataclass
class MockLLMResponse:
    """Mock LLM 响应的数据类"""
    action: str
    params: dict = None
    thought: str = ""
    response_type: str = "tool_call"
    
    def to_dict(self) -> dict:
        return {
            "thought": self.thought,
            "action": self.action,
            "action_input": self.params or {}
        }
    
    def to_llm_output(self) -> dict:
        return {
            "role": "assistant",
            "content": json.dumps(self.to_dict())
        }

# 策略 2: 使用工厂函数
def make_weather_response(city="北京", temp=22, condition="晴"):
    return MockLLMResponse(
        action="get_weather",
        params={"city": city, "temp": temp, "condition": condition}
    )

def make_error_response(error_msg="服务不可用"):
    return MockLLMResponse(
        action="_error",
        params={"error": error_msg},
        response_type="error"
    )

# 策略 3: 使用参数化测试生成多组测试数据
@pytest.mark.parametrize("user_input,expected_action,expected_params", [
    ("北京的天气", "get_weather", {"city": "北京"}),
    ("上海的天气", "get_weather", {"city": "上海"}),
    ("现在几点", "get_time", {"timezone": "Asia/Shanghai"}),
    ("今天的新闻", "get_news", {"category": "general"}),
    ("1+1等于几", "calculator", {"expression": "1+1"}),
])
def test_tool_selection_parametrized(
    user_input, expected_action, expected_params
):
    """参数化测试：一组输入覆盖多种工具选择场景"""
    mock_llm = MagicMock()
    mock_llm.chat.return_value = {
        "content": json.dumps({
            "action": expected_action,
            "action_input": expected_params
        })
    }
    
    agent = Agent(llm=mock_llm, tools=[
        get_weather, get_time, get_news, calculator
    ])
    
    action = agent.plan(user_input)
    assert action["name"] == expected_action
```

### 覆盖率测量

```python
# ============================================================
# Agent 代码的覆盖率配置
# ============================================================

# pytest 配置 (pyproject.toml)
"""
[tool.coverage.run]
source = ["my_agent"]
omit = [
    "*/tests/*",
    "*/migrations/*",
    "*/__init__.py"
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "if __name__ == .__main__.:",
    "raise NotImplementedError",
    "if TYPE_CHECKING:",
]

# 针对 Agent 的特殊覆盖率规则
[tool.coverage.agent_specific]
# Agent 的核心逻辑路径必须覆盖
required_branches = [
    "tool_selection",      # 工具选择分支
    "error_handling",      # 错误处理分支
    "state_update",        # 状态更新路径
    "max_steps_reached",   # 最大步数限制
]
"""

# Agent 覆盖率测量的关注点：

AGENT_COVERAGE_CHECKLIST = """
✓ 每个工具选择分支都被测试覆盖
  - 正常场景：正常选择每个工具
  - 边界场景：没有匹配的工具时
  - 模糊场景：匹配多个工具时

✓ 每个错误处理路径都被测试覆盖
  - 工具抛出异常
  - 超时
  - 参数验证失败
  - 重试耗尽

✓ 每个状态转换都被测试覆盖
  - 初始 → 运行中
  - 运行中 → 完成
  - 运行中 → 错误
  - 完成 → 重置

✓ 停止条件都被测试覆盖
  - max_steps 限制
  - max_tokens 限制
  - 用户取消
  - 致命错误
"""
```

### CI 集成建议

```yaml
# .github/workflows/agent-tests.yml
# Agent 单元测试的 CI 配置

name: Agent Unit Tests
on: [push, pull_request]

jobs:
  agent-unit-tests:
    runs-on: ubuntu-latest
    
    # Agent 测试的特殊环境变量
    env:
      # 强制使用 Mock，防止意外调用真实 LLM
      AGENT_TEST_MODE: "1"
      # 如果测试意外尝试调用真实 LLM，立即失败
      LLM_API_KEY: "sk-test-blocked"
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install dependencies
        run: pip install -e ".[dev,test]"
      
      - name: Run agent unit tests
        # Mock 测试在隔离环境中运行，不需要 GPU 或 API key
        run: |
          pytest tests/unit/agent/ \
            --cov=my_agent \
            --cov-report=xml \
            --cov-report=term \
            -v \
            -m "not integration and not e2e"
      
      - name: Run agent integration tests
        # 集成测试可能需要 API key，使用 secrets
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
        run: |
          pytest tests/integration/agent/ \
            -v \
            -m "integration"
      
      - name: Check agent-specific coverage
        run: |
          python scripts/check_agent_coverage.py \
            --min-tool-coverage 90 \
            --min-error-handling 80 \
            --report coverage.xml
```

```python
# scripts/check_agent_coverage.py
# Agent 专用覆盖率检查脚本

def check_agent_coverage(coverage_report: dict) -> bool:
    """
    检查 Agent 特有的覆盖率指标：
    - 工具选择分支覆盖率
    - 错误处理路径覆盖率
    - 状态转换路径覆盖率
    """
    required_paths = [
        "agent.planner.tool_selection",
        "agent.executor.error_handler",
        "agent.state_manager.state_transition",
        "agent.orchestrator.stop_condition",
    ]
    
    all_covered = True
    for path in required_paths:
        coverage = coverage_report.get(path, 0)
        if coverage < 80:
            print(f"WARNING: {path} coverage is only {coverage}%")
            all_covered = False
    
    return all_covered
```

## 挑战与局限

### 挑战一：Mocking 降低测试有效性

这是 Agent 单元测试最根本的矛盾——**Mock 越多，测试越可靠（不 flaky），但测试的有效性越低（真实场景可能完全不同）**。

```
单元测试的"可靠性-有效性"权衡：

  完全 Mock:
    测试可靠性：★★★★★（从不 flaky）
    测试有效性：★★☆☆☆（和真实场景差距大）
    问题：Mock LLM 总是返回正确的结构化输出，
          但真实 LLM 可能返回乱七八糟的内容。
          "Mock 通过，上线就挂"

  部分 Mock:
    测试可靠性：★★★☆☆（偶尔 flaky）
    测试有效性：★★★★☆（接近真实场景）
    问题：使用了真实 LLM 但限制了范围和温度，
          结果仍然有随机性，测试偶尔失败。

  真实环境:
    测试可靠性：★☆☆☆☆（经常 flaky）
    测试有效性：★★★★★（完全真实）
    问题：LLM 的每一次响应都不同，
          测试结果不可重复。
```

**缓解策略**：

```
1. 多层测试策略
   ┌─ 单元测试（完全 Mock）───── 验证外围逻辑 ── 快速、可靠
   ├─ 集成测试（真实 LLM）────── 验证核心交互 ── 中等速度
   └─ 评估测试（端到端）──────── 验证整体质量 ── 慢、统计性

2. Mock 匹配度审计
   定期检查 Mock 响应是否还匹配真实 LLM 的输出格式
   如果 LLM 升级/换模型，Mock 数据需要同步更新

3. 契约测试 (Contract Testing)
   定义 LLM 输出的"契约"（格式、字段、类型）
   Mock 和真实 LLM 都遵循同一个契约
   当真实 LLM 输出不符合契约时，测试失败
```

### 挑战二：确定性 vs 非确定性的张力

```
Agent 天生是非确定性的，但测试要求确定性。
这对矛盾无法完全解决，只能管理。

  非确定性的来源：
    LLM 采样温度 > 0：每次输出不同
    LLM 版本迭代：同样 Prompt 在不同版本上结果不同
    工具响应时间：网络延迟影响 Agent 行为
    外部数据变化：数据库内容变化影响 Agent 决策

  管理策略：
    1. 测试环境冻结：固定 LLM 版本、固定模型温度 = 0
    2. 输出结构的统计测试：运行 N 次，检查成功率 > 阈值
    3. 非确定性部分的降级：对于 LLM 输出内容做"模糊断言"
    4. 分离确定和非确定层：非确定性部分用集成测试覆盖
```

### 挑战三：高维护负担

Agent 系统的变化速度远快于传统软件——Prompt 在频繁调整、工具在增删、LLM 模型在升级——这些变化都可能导致 Mock 数据失效。

```
维护负担的具体来源：

  1. Prompt 变化 → Mock 响应的 Prompt 匹配条件需要更新
  2. 工具增加 → 测试需要覆盖新工具的路径
  3. 模型升级 → LLM 输出格式变化 → Mock 数据结构需要更新
  4. 框架升级 → Agent 框架 API 变化 → 测试代码需要重写
  5. 行为变化 → Agent 决策逻辑变化 → 期望的测试结果变化

  经验数据：
    一个中等复杂的 Agent（5-8 个工具，3-5 轮对话）
    大约需要 30-50 个单元测试
    每次 Prompt 大改需要更新 10-20% 的 Mock 数据
    每次模型升级需要验证 100% 的 Mock 数据是否仍然有效
```

**缓解策略**：

```
  1. 将 Mock 数据与测试逻辑分离
     把 Mock 响应放在独立的 JSON/YAML 文件中
     测试逻辑不变，只改数据文件
  
  2. 使用"录制-重放"工具（VCR）
     自动录制真实 LLM 调用 → 减少手动维护 Mock
  
  3. 定期清理失效测试
     标记长时间不维护的测试
     建立测试健康度仪表盘
  
  4. 优先测试稳定层
     把资源投入到不太变化的部分（工具调用包装、状态管理）
     快速变化的部分（Prompt、Agent 行为）用更少的测试覆盖
```

### 挑战四：社区标准不足

与传统软件开发相比，Agent 测试的社区标准和最佳实践仍在形成中。

```
Agent 测试当前的"混乱"状态：

  缺乏共识：
    ✗ 没有统一的 Agent 测试框架
    ✗ 没有公认的 Mock 标准
    ✗ 覆盖率定义不统一
    ✗ "好的 Agent 测试"的标准不明确

  各说各话：
    LangGraph 推荐自己的图测试方案
    LangChain 推荐 Chain 级别的测试
    AutoGen 推荐多 Agent 测试
    CrewAI 推荐任务级别的测试
    每个框架的测试哲学都不同

  正在形成的共识（2025-2026）：
    ✓ Mock LLM 是单元测试的必要条件
    ✓ 测试应该分层（单元/集成/评估）
    ✓ 工具调用是 Agent 测试的核心关注点
    ✓ 确定性逻辑优先测试，非确定性逻辑用统计方法
```

```
未来趋势（预测）：

  2026-2027:
    更多 Agent 框架内置测试工具
    LLM Mock 标准化（类似 HTTP Mock 的标准化程度）
    社区测试模式库出现
  
  2027-2028:
    Agent 测试覆盖率定义标准化
    自动生成 Mock 响应的工具成熟
    Agent 的 TDD 方法论形成
```

## 总结

```
Agent 单元测试的核心要点：

  DO（应该做的）:
    ✓ Mock LLM 响应，让测试确定可重复
    ✓ Mock 所有外部工具和 API
    ✓ 测试工具选择、参数提取、状态更新等确定性逻辑
    ✓ 使用分层测试策略（单元 → 集成 → 评估）
    ✓ 维护 Mock 数据与真实 LLM 输出的一致性
    ✓ 关注边界条件和错误处理路径

  DON'T（不应做的）:
    ✗ 试图测试 LLM 的"思考质量"
    ✗ 测试中真实调用 LLM（这是集成测试的事）
    ✗ 断言 LLM 生成的具体文本内容
    ✗ 写 flaky 的测试然后 ignore
    ✗ 过度测试——Agent 代码变化快，测试太多维护不过来
    ✗ 把单元测试当成评估——它们在验证不同的事

  黄金法则：
    "单元测试验证 Agent 的代码是否正确实现了设计逻辑，
    评估测试验证 LLM 是否做出了正确的决策，
    两者互相补充，不可替代。"
```
