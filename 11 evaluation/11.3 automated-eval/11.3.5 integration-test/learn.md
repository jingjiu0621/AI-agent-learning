# 11.3.5 integration-test — Agent 集成测试

## 简单介绍

集成测试（Integration Test）是 Agent 评估体系中"测真实交互"的关键手段。与单元测试不同，集成测试**不 mock（模拟）外部依赖**，而是用真实（或接近真实）的环境来验证 Agent 与各个组件的协作是否正确。

```
Agent 测试层级：

单元测试 (Unit Test)
  ┌─────────────────────────────────────┐
  │  测试单个函数/工具/LLM 调用           │
  │  所有外部依赖全部 mock               │
  │  速度：毫秒级                        │
  │  可靠性：高（确定性）                  │
  └──────────┬──────────────────────────┘
             │ 只测"组件自己"
             ▼
集成测试 (Integration Test)            ← 我们在这里
  ┌─────────────────────────────────────┐
  │  测试组件间的真实交互                 │
  │  外部依赖使用真实/沙箱实例            │
  │  速度：秒级到分钟级                    │
  │  可靠性：中（有非确定性因素）          │
  └──────────┬──────────────────────────┘
             │ 测"组件怎么配合"
             ▼
端到端测试 (E2E Test)
  ┌─────────────────────────────────────┐
  │  测试完整用户旅程                     │
  │  所有依赖真实部署                     │
  │  速度：分钟级到小时级                  │
  │  可靠性：低（环境波动）                │
  └──────────┬──────────────────────────┘
             │ 测"整个系统是否可用"
             ▼
生产环境 (Production)
  ┌─────────────────────────────────────┐
  │  真实用户使用                        │
  │  所有依赖真实且变化                  │
  │  持续监控                            │
  └─────────────────────────────────────┘
```

核心思想很简单：**Agent 的价值在于它能把 LLM、工具、记忆、推理整合在一起解决问题。如果只单独测试每个组件（单元测试），就永远不知道它们组合在一起能否正常工作。**

```
单元测试 vs 集成测试的对比：

        单元测试                        集成测试
  ┌──────────────┐              ┌──────────────────┐
  │ tool_call()  │              │  Agent 完整流程    │
  │     ↓        │              │      ↓            │
  │ mock API ←──│── 假的       │  real API ←──────│── 真的
  │     ↓        │              │      ↓            │
  │ 返回预设值   │              │  真实响应          │
  │     ↓        │              │      ↓            │
  │ 测函数逻辑   │              │  测决策+执行+适应  │
  └──────────────┘              └──────────────────┘

  例子：天气 Agent
    单元测试：mock get_weather("Beijing") → 返回 "25°C"
             测试输出格式化是否正确
    集成测试：实际调用天气 API → 拿到今天北京真实温度
             测试 Agent 能否正确解析并呈现
```

## 基本原理

### 集成测试光谱

```
测试完整度光谱：

  完全 Mock                        完全真实
  ◄═══════════════════════════════════════►

  ┌──────┬──────────┬──────────┬──────────┬──────────┐
  │ Unit │ 集成测试  │ 集成测试  │  集成测试 │   E2E    │
  │ Test │ (轻量)    │ (标准)    │ (重型)   │   Test   │
  ├──────┼──────────┼──────────┼──────────┼──────────┤
  │Mock  │Mock LLM  │Real LLM  │Real LLM  │全部真实  │
  │全部  │Real Tool  │Real Tool │Real Tool │          │
  │      │Sandbox   │Sandbox   │Sandbox   │          │
  │      │          │Mock 状态  │Real 状态 │          │
  ├──────┼──────────┼──────────┼──────────┼──────────┤
  │~10ms │ ~100ms   │  ~5s     │  ~30s    │  ~5min   │
  │稳定   │较稳定     │有波动     │有波动     │不稳定     │
  │0成本  │低成本     │中成本     │高成本     │高成本     │
  └──────┴──────────┴──────────┴──────────┴──────────┘

  关键决策：在成本和真实性之间找到平衡点。
```

### 集成测试回答的核心问题

```
┌─ Agent 集成测试要回答的问题 ─────────────────────────────┐
│                                                           │
│  Q1: Agent 能不能正确调用真实工具？                         │
│      → 工具存在吗？参数对吗？响应能解析吗？                   │
│                                                           │
│  Q2: Agent 能不能根据工具响应做下一步决策？                  │
│      → 拿到天气数据后，知道要带伞吗？                         │
│                                                           │
│  Q3: Agent 的多步流程能不能走通？                           │
│      → 搜索 → 阅读 → 总结，每一步都能正确执行吗？             │
│                                                           │
│  Q4: Agent 在真实环境下崩溃吗？                              │
│      → API 超时怎么办？返回错误怎么办？                       │
│                                                           │
│  Q5: Agent 的状态管理正确吗？                               │
│      → 跨步骤的记忆是否一致？状态是否能持久化？                │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

## 集成测试的核心维度

Agent 集成测试覆盖五个关键维度，每个维度测试不同的集成点：

```
                ┌──────────────────────────┐
                │     Agent 集成测试         │
                │     五大维度               │
                └──────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ Tool 集成      │ │  LLM 集成      │ │  Memory 集成   │
│               │ │               │ │               │
│ 真实 API 调用  │ │ 真实模型调用    │ │ 状态持久化      │
│ 沙箱环境       │ │ 受控 Prompt   │ │ 检索与存储      │
│ 参数验证       │ │ 输出格式验证   │ │ 跨会话一致性    │
└───────────────┘ └───────────────┘ └───────────────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                          ▼
                ┌──────────────────────────┐
                │    Multi-step Flow 集成    │  ← 最核心
                │    ┌──────────────────┐   │
                │    │ 完整的行动序列     │   │
                │    │ 工具 → 决策 → 工具 │   │
                │    │ → 决策 → 输出     │   │
                │    └──────────────────┘   │
                │                           │
                │    Cross-agent 集成        │
                │    ┌──────────────────┐   │
                │    │ 多 Agent 通信      │   │
                │    │ 消息传递           │   │
                │    │ 任务分发与协作      │   │
                │    └──────────────────┘   │
                └──────────────────────────┘
```

### 1. Tool Integration

测试 Agent 与真实工具的交互是否正确。

```python
# 工具集成测试的核心检查点
TOOL_INTEGRATION_CHECKS = {
    "discovery": {
        "问题": "工具是否可以被 Agent 发现和选择？",
        "验证": "Agent 在需要时是否调用了正确的工具"
    },
    "parameter_passing": {
        "问题": "参数传递是否正确？",
        "验证": "Agent 提取的参数与工具 schema 是否匹配"
    },
    "response_parsing": {
        "问题": "工具响应能否被正确解析？",
        "验证": "Agent 是否能从工具响应中提取所需信息"
    },
    "error_handling": {
        "问题": "工具返回错误时 Agent 如何应对？",
        "验证": "Agent 是否能优雅地处理工具错误并恢复"
    },
    "rate_limiting": {
        "问题": "Agent 是否尊重工具的速率限制？",
        "验证": "高频调用时 Agent 是否会等待或降速"
    }
}
```

### 2. LLM Integration

测试 Agent 与真正 LLM 的交互。

```python
LLM_INTEGRATION_CHECKS = {
    "prompt_delivery": {
        "问题": "构造的 Prompt 是否能被 LLM 正确理解？",
        "验证": "LLM 响应是否遵循了指令格式"
    },
    "tool_call_format": {
        "问题": "LLM 是否能生成正确的工具调用格式？",
        "验证": "工具调用 JSON 是否符合 schema"
    },
    "context_window": {
        "问题": "长上下文下 LLM 的表现是否稳定？",
        "验证": "历史累积后 LLM 是否会遗忘早期信息"
    },
    "output_consistency": {
        "问题": "同一输入下 LLM 输出是否足够一致？",
        "验证": "temperature=0 时多次调用输出的差异程度"
    }
}
```

### 3. Memory/State Integration

测试 Agent 状态管理的正确性。

```python
MEMORY_INTEGRATION_CHECKS = {
    "write_read": {
        "问题": "状态能否正确写入并读取？",
        "验证": "写入 → 读取 → 验证内容一致性"
    },
    "cross_step": {
        "问题": "跨步骤的状态传递是否正确？",
        "验证": "第 1 步保存的信息在第 3 步能否访问"
    },
    "persistence": {
        "问题": "状态持久化是否可靠？",
        "验证": "进程重启后状态是否能恢复"
    },
    "isolation": {
        "问题": "不同会话的状态是否隔离？",
        "验证": "会话 A 的修改不会影响会话 B"
    }
}
```

### 4. Multi-step Flow Integration

测试 Agent 完成多步骤任务的能力——这是集成测试最核心的部分。

```python
MULTI_STEP_FLOW_CHECKS = {
    "completion": {
        "问题": "Agent 能否完成任务从头到尾？",
        "验证": "执行完整流程并验证最终输出"
    },
    "branching": {
        "问题": "Agent 能否根据中间结果调整计划？",
        "验证": "当工具返回意外结果时 Agent 是否调整行为"
    },
    "recovery": {
        "问题": "Agent 能否从错误中恢复？",
        "验证": "工具失败后 Agent 是否尝试替代方案"
    },
    "termination": {
        "问题": "Agent 能否正确决定何时停止？",
        "验证": "任务完成后 Agent 是否停止调用更多工具"
    }
}
```

### 5. Cross-agent Integration

测试多 Agent 系统中的通信和协作。

```python
CROSS_AGENT_CHECKS = {
    "message_passing": {
        "问题": "Agent 间的消息能否正确传递？",
        "验证": "发送方消息被接收方正确解析"
    },
    "task_delegation": {
        "问题": "任务分发和委派是否正确？",
        "验证": "子任务被分配给正确的 Agent"
    },
    "result_aggregation": {
        "问题": "多 Agent 的结果能否正确汇总？",
        "验证": "最终输出包含了所有 Agent 的贡献"
    }
}
```

## 环境管理

集成测试最大的挑战之一是环境管理——测试必须可重复，但真实环境很难完全可重复。

```
环境管理策略金字塔：

                      ┌───────────┐
                      │  生产环境  │  真实数据、真实成本
                     ┌┼───────────┼┐
                    │ │  预发环境  │ │  接近生产但隔离
                   ┌┼┼───────────┼┼┐
                  │ │ │  Docker   │ │ │  可重复、可销毁
                 ┌┼┼┼───────────┼┼┼┐
                │ │ │ │ Sandbox  │ │ │ │  轻量隔离
               ┌┼┼┼┼───────────┼┼┼┼┐
              │ │ │ │ │  Fixture │ │ │ │ │  静态测试数据
             ┌┼┼┼┼┼───────────┼┼┼┼┼┐
            │ │ │ │ │ │  Mock   │ │ │ │ │ │  完全不依赖外部
            └┼┼┼┼┼┼───────────┼┼┼┼┼┼┘
             └┼┼┼┼┼───────────┼┼┼┼┼┘
              └┼┼┼┼───────────┼┼┼┼┘
               └┼┼┼───────────┼┼┼┘
                └┼┼───────────┼┼┘
                 └┼───────────┼┘
                  └───────────┘
```

### Docker/Sandbox 方案

使用容器为每个测试创建隔离环境：

```python
import docker
import pytest
from pathlib import Path


@pytest.fixture(scope="module")
def sandbox_container():
    """
    为 Agent 集成测试创建沙箱容器。
    每个测试模块一个容器，测试间共享。
    """
    client = docker.from_env()

    container = client.containers.run(
        image="agent-test-env:latest",
        command=["sleep", "infinity"],
        detach=True,
        remove=True,
        environment={
            "TEST_MODE": "1",
            "API_BASE_URL": "http://test-api:8080",
        },
        volumes={
            str(Path.cwd() / "test_fixtures"): {
                "bind": "/fixtures",
                "mode": "ro"
            }
        },
        network="test-network",
    )

    yield container

    # Cleanup
    container.stop(timeout=5)


@pytest.fixture
def api_endpoint(sandbox_container):
    """获取容器内 API 端点地址"""
    import requests
    ip = sandbox_container.attrs["NetworkSettings"]["Networks"]["test-network"]["IPAddress"]
    return f"http://{ip}:8080"
```

### Test Fixtures 设计

```python
# test_fixtures/weather_service.py
"""
测试用的天气 API Fixture。
用 Flask 模拟一个简单的天气服务，可控的响应。
"""
from flask import Flask, jsonify, request
import time

app = Flask(__name__)

# 预设的测试响应
TEST_RESPONSES = {
    "Beijing": {
        "temperature": 25,
        "condition": "sunny",
        "humidity": 45,
        "wind_speed": 10,
    },
    "Shanghai": {
        "temperature": 28,
        "condition": "rainy",
        "humidity": 75,
        "wind_speed": 15,
    },
    "ERROR_CITY": {  # 用于测试错误处理
        "error": "city_not_found",
        "message": "City not found in database"
    }
}


@app.route("/weather/<city>")
def get_weather(city):
    delay = float(request.args.get("delay", 0))
    if delay > 0:
        time.sleep(delay)  # 模拟网络延迟

    if city == "TIMEOUT":
        time.sleep(30)  # 模拟超时
        return jsonify({"error": "timeout"}), 504

    response = TEST_RESPONSES.get(city, TEST_RESPONSES["ERROR_CITY"])
    status = 200 if "error" not in response else 404
    return jsonify(response), status


@app.route("/health")
def health():
    return jsonify({"status": "ok"})


if __name__ == "__main__":
    app.run(port=8080)
```

### 环境配置管理

```python
from dataclasses import dataclass, field
from typing import Optional
import os
import json


@dataclass
class IntegrationTestConfig:
    """
    集成测试环境配置。
    支持通过环境变量覆盖，确保 CI 和本地使用不同配置。
    """
    # LLM 配置
    llm_model: str = field(
        default_factory=lambda: os.getenv("TEST_LLM_MODEL", "gpt-4o-mini")
    )
    llm_temperature: float = 0.0  # 集成测试应使用确定性温度
    llm_max_tokens: int = 1024

    # 外部服务端点
    weather_api_url: str = field(
        default_factory=lambda: os.getenv(
            "WEATHER_API_URL",
            "http://localhost:8080/weather"  # 默认指向 test fixture
        )
    )
    search_api_url: str = field(
        default_factory=lambda: os.getenv(
            "SEARCH_API_URL",
            "http://localhost:8081/search"
        )
    )

    # 测试行为控制
    record_mode: bool = False      # 是否录制交互（首次运行）
    replay_mode: bool = True       # 是否回放录制的交互（后续运行）
    cassette_dir: str = "./test_cassettes"

    # 超时控制
    tool_timeout: int = 10         # 工具调用超时（秒）
    agent_timeout: int = 60        # 整个 Agent 流程超时（秒）

    # 重试策略
    max_retries: int = 2
    retry_delay: float = 1.0

    @classmethod
    def from_json(cls, path: str) -> "IntegrationTestConfig":
        with open(path) as f:
            data = json.load(f)
        return cls(**data)


# CI 和本地使用不同配置的示例
# CI:  TEST_LLM_MODEL=gpt-4o-mini WEATHER_API_URL=http://docker-weather:8080/weather
# 本地: 默认使用 localhost test fixture
```

### 隔离策略

```
测试隔离的四个层次：

层次 1：进程级隔离
  ┌─────────────────────────────────┐
  │  每个测试文件运行在独立进程中      │
  │  使用 pytest-xdist 的 --forked   │
  │  防止共享状态污染                 │
  └─────────────────────────────────┘

层次 2：命名空间隔离
  ┌─────────────────────────────────┐
  │  每个测试用例使用唯一前缀/ID      │
  │  例：test_user_<uuid>            │
  │  测试结束后清理资源               │
  └─────────────────────────────────┘

层次 3：容器级隔离
  ┌─────────────────────────────────┐
  │  每个测试或套件启动独立容器       │
  │  容器销毁时自动清理               │
  │  适合有副作用（写文件、改状态）的  │
  │  工具测试                        │
  └─────────────────────────────────┘

层次 4：数据隔离
  ┌─────────────────────────────────┐
  │  每个测试使用独立测试数据集       │
  │  测试数据库：事务回滚             │
  │  fixture 数据：只读副本           │
  └─────────────────────────────────┘
```

## 典型集成测试模式

### 模式 1：工具执行集成测试

测试 Agent 的工具是否能真正与外部 API 协作。

```python
import pytest
import requests
from typing import Dict, Any


class WeatherTool:
    """被测试的工具——真实调用天气 API"""

    def __init__(self, api_url: str):
        self.api_url = api_url

    def get_weather(self, city: str, units: str = "celsius") -> Dict[str, Any]:
        """调用真实天气 API"""
        params = {"city": city, "units": units}

        try:
            response = requests.get(
                self.api_url,
                params=params,
                timeout=10
            )
            response.raise_for_status()

            data = response.json()
            return {
                "success": True,
                "city": city,
                "temperature": data["temperature"],
                "condition": data["condition"],
                "humidity": data.get("humidity"),
                "wind_speed": data.get("wind_speed"),
            }
        except requests.exceptions.Timeout:
            return {
                "success": False,
                "error": "timeout",
                "city": city,
                "message": f"Request to weather API timed out"
            }
        except requests.exceptions.HTTPError as e:
            return {
                "success": False,
                "error": "http_error",
                "city": city,
                "status_code": response.status_code,
                "message": str(e)
            }
        except Exception as e:
            return {
                "success": False,
                "error": "unknown",
                "city": city,
                "message": str(e)
            }


class TestWeatherToolIntegration:
    """
    天气工具的集成测试。
    测试工具与真实（或 fixture）API 的交互。
    """

    @pytest.fixture
    def weather_api(self, request):
        """提供天气 API URL 的 fixture"""
        return request.config.getoption(
            "--weather-api-url",
            default="http://localhost:8080/weather"
        )

    def test_successful_weather_lookup(self, weather_api):
        """测试正常天气查询"""
        tool = WeatherTool(weather_api)
        result = tool.get_weather("Beijing")

        assert result["success"] is True
        assert result["city"] == "Beijing"
        assert isinstance(result["temperature"], (int, float))
        assert -50 <= result["temperature"] <= 60  # 合理范围
        assert result["condition"] in ["sunny", "cloudy", "rainy", "snowy"]

    def test_city_not_found(self, weather_api):
        """测试城市不存在的情况"""
        tool = WeatherTool(weather_api)
        result = tool.get_weather("NonExistentCity123")

        assert result["success"] is False
        assert result["error"] == "http_error"
        assert result["status_code"] == 404

    def test_api_timeout(self, weather_api):
        """测试 API 超时处理"""
        tool = WeatherTool(weather_api)
        # 特殊的 city 名称触发 fixture 的延迟响应
        result = tool.get_weather("TIMEOUT")

        assert result["success"] is False
        assert result["error"] == "timeout"

    def test_invalid_parameters(self, weather_api):
        """测试无效参数传递"""
        tool = WeatherTool(weather_api)

        # 空的 city 名称
        result = tool.get_weather("")
        assert result["success"] is False

        # 超长 city 名称
        result = tool.get_weather("A" * 1000)
        assert result["success"] is False or isinstance(result["temperature"], (int, float))
```

### 模式 2：Agent Loop 集成测试

测试 Agent 的思考-行动-观察循环是否能在真实 LLM 下工作。

```python
import pytest
from typing import List, Dict, Any, Optional


class AgentLoop:
    """简化的 Agent 循环——用于集成测试"""

    def __init__(self, llm_client, tools: Dict[str, Any], max_steps: int = 5):
        self.llm = llm_client
        self.tools = tools
        self.max_steps = max_steps
        self.messages = []
        self.steps_taken = 0

    def run(self, task: str) -> Dict[str, Any]:
        """运行 Agent 循环"""
        self.messages = [{"role": "user", "content": task}]
        self.steps_taken = 0

        while self.steps_taken < self.max_steps:
            # 思考：调用 LLM 决定下一步
            response = self.llm.chat.completions.create(
                model="gpt-4o-mini",
                messages=self.messages,
                tools=list(self.tools.values()),
                temperature=0,
            )

            message = response.choices[0].message
            self.messages.append(message)

            # 检查是否有工具调用
            if not message.tool_calls:
                # 没有工具调用 → Agent 认为任务完成
                return {
                    "success": True,
                    "output": message.content,
                    "steps": self.steps_taken,
                    "messages": self.messages,
                }

            # 执行：调用工具
            for tool_call in message.tool_calls:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)

                tool_fn = self.tools.get(tool_name)
                if tool_fn:
                    result = tool_fn(**tool_args)
                else:
                    result = {"error": f"Unknown tool: {tool_name}"}

                self.messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": json.dumps(result),
                })

            self.steps_taken += 1

        # 达到最大步骤限制
        return {
            "success": False,
            "output": None,
            "steps": self.steps_taken,
            "messages": self.messages,
            "error": "max_steps_exceeded"
        }


class TestAgentLoopIntegration:
    """
    Agent Loop 集成测试。
    测试 Agent 能否在真实 LLM 下完成多步推理+工具调用。
    """

    @pytest.fixture
    def llm_client(self):
        """真实的 LLM 客户端（用最小/最便宜的模型以减少成本）"""
        from openai import OpenAI
        return OpenAI()

    @pytest.fixture
    def mock_tools(self):
        """轻量工具定义——为集成测试简化"""
        return {
            "get_weather": {
                "type": "function",
                "function": {
                    "name": "get_weather",
                    "description": "Get current weather for a city",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "city": {"type": "string", "description": "City name"}
                        },
                        "required": ["city"]
                    }
                }
            },
            "calculator": {
                "type": "function",
                "function": {
                    "name": "calculator",
                    "description": "Perform basic arithmetic",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "expr": {"type": "string", "description": "Math expression"}
                        },
                        "required": ["expr"]
                    }
                }
            }
        }

    @pytest.fixture
    def mock_tool_implementations(self):
        """工具的真实实现——使用 fixture 而非真实 API"""
        def get_weather(city: str) -> dict:
            weather_data = {
                "Beijing": {"temp": 25, "condition": "sunny"},
                "London": {"temp": 15, "condition": "cloudy"},
                "Tokyo": {"temp": 22, "condition": "rainy"},
            }
            return weather_data.get(city, {"error": "city not found"})

        def calculator(expr: str) -> dict:
            try:
                # 安全的 eval——仅限于数字运算
                allowed = set("0123456789+-*/() .")
                if not all(c in allowed for c in expr):
                    return {"error": "invalid expression"}
                result = eval(expr, {"__builtins__": {}}, {})
                return {"result": result}
            except Exception as e:
                return {"error": str(e)}

        return {"get_weather": get_weather, "calculator": calculator}

    def test_single_tool_call(self, llm_client, mock_tools, mock_tool_implementations):
        """测试简单的单步工具调用：Agent 调用 get_weather 并输出结果"""
        agent = AgentLoop(
            llm_client=llm_client,
            tools=mock_tool_implementations,
            tool_definitions=mock_tools,
            max_steps=3,
        )

        result = agent.run("What's the weather in Beijing?")

        assert result["success"] is True
        assert result["steps"] >= 1  # 至少调用了 1 次工具
        # 验证最终输出包含天气信息
        assert "25" in result["output"] or "sunny" in result["output"].lower()

    def test_multi_step_tool_sequence(self, llm_client, mock_tools, mock_tool_implementations):
        """测试多步工具序列：先查天气，再决定是否需要计算"""
        agent = AgentLoop(
            llm_client=llm_client,
            tools=mock_tool_implementations,
            tool_definitions=mock_tools,
            max_steps=5,
        )

        result = agent.run(
            "Check the weather in London and Tokyo. "
            "Which city is warmer and by how many degrees?"
        )

        assert result["success"] is True
        assert result["steps"] >= 2  # 至少调用了 2 次工具
        assert "Tokyo" in result["output"]
        assert "London" in result["output"]

    def test_agent_knows_when_to_stop(self, llm_client, mock_tools, mock_tool_implementations):
        """测试 Agent 知道何时停止：不需要工具的任务不应调用工具"""
        agent = AgentLoop(
            llm_client=llm_client,
            tools=mock_tool_implementations,
            tool_definitions=mock_tools,
            max_steps=5,
        )

        result = agent.run("What is the capital of France?")

        assert result["success"] is True
        assert result["steps"] == 0  # 没有工具调用
        assert "Paris" in result["output"]
```

### 模式 3：RAG Pipeline 集成测试

测试检索增强生成管道的端到端工作。

```python
@pytest.mark.integration
class TestRAGPipelineIntegration:
    """
    RAG 管道的集成测试。
    测试：文档嵌入 → 向量检索 → LLM 生成 的完整流程。
    """

    @pytest.fixture
    def vector_store(self):
        """真实的向量数据库（使用内存模式加速测试）"""
        import chromadb
        client = chromadb.Client(chromadb.Settings(
            anonymized_telemetry=False,
            is_persistent=False,  # 内存模式
        ))
        collection = client.create_collection("test_docs")
        return collection

    @pytest.fixture
    def embedding_model(self):
        """轻量 embedding 模型"""
        from sentence_transformers import SentenceTransformer
        return SentenceTransformer("all-MiniLM-L6-v2")

    @pytest.fixture
    def test_documents(self):
        return [
            {"id": "doc1", "text": "Paris is the capital of France. It is known for the Eiffel Tower."},
            {"id": "doc2", "text": "Tokyo is the capital of Japan. It is known for its technology."},
            {"id": "doc3", "text": "London is the capital of the UK. It is known for Big Ben."},
        ]

    def test_embed_and_retrieve(self, vector_store, embedding_model, test_documents):
        """测试：嵌入文档 → 检索相关片段"""
        # 嵌入并存储
        texts = [doc["text"] for doc in test_documents]
        embeddings = embedding_model.encode(texts).tolist()

        vector_store.add(
            ids=[doc["id"] for doc in test_documents],
            embeddings=embeddings,
            metadatas=test_documents,
            documents=texts,
        )

        # 检索
        query = "What is the capital of Japan?"
        query_embedding = embedding_model.encode(query).tolist()

        results = vector_store.query(
            query_embeddings=[query_embedding],
            n_results=1,
        )

        assert len(results["documents"][0]) > 0
        assert "Tokyo" in results["documents"][0][0]
        assert "Japan" in results["documents"][0][0]

    def test_full_rag_flow(self, vector_store, embedding_model, test_documents):
        """测试：检索 + LLM 生成的完整 RAG 流程"""
        # 第一步：索引文档
        texts = [doc["text"] for doc in test_documents]
        embeddings = embedding_model.encode(texts).tolist()
        vector_store.add(
            ids=[doc["id"] for doc in test_documents],
            embeddings=embeddings,
            documents=texts,
        )

        # 第二步：检索
        query = "Tell me about London"
        query_embedding = embedding_model.encode(query).tolist()
        results = vector_store.query(
            query_embeddings=[query_embedding],
            n_results=1,
        )
        retrieved = results["documents"][0][0]

        # 第三步：生成
        from openai import OpenAI
        llm = OpenAI()

        response = llm.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "Answer based on the retrieved context."},
                {"role": "user", "content": f"Context: {retrieved}\n\nQuestion: {query}"}
            ],
            temperature=0,
        )

        answer = response.choices[0].message.content
        assert "London" in answer
        assert "Big Ben" in answer or "capital" in answer.lower()
```

### 模式 4：多 Agent 通信集成测试

测试多个 Agent 之间的消息传递和协作。

```python
import pytest
from dataclasses import dataclass
from typing import List, Optional
from enum import Enum


class MessageType(Enum):
    TASK = "task"
    RESULT = "result"
    QUERY = "query"
    RESPONSE = "response"
    ERROR = "error"


@dataclass
class AgentMessage:
    """Agent 间传递的消息"""
    sender: str
    recipient: str
    msg_type: MessageType
    content: str
    metadata: Optional[dict] = None


class MessageBus:
    """简单的 Agent 消息总线"""

    def __init__(self):
        self.queues: dict = {}
        self.history: List[AgentMessage] = []

    def send(self, message: AgentMessage):
        if message.recipient not in self.queues:
            self.queues[message.recipient] = []
        self.queues[message.recipient].append(message)
        self.history.append(message)

    def receive(self, agent_name: str) -> Optional[AgentMessage]:
        if agent_name in self.queues and self.queues[agent_name]:
            return self.queues[agent_name].pop(0)
        return None

    def clear(self, agent_name: Optional[str] = None):
        if agent_name:
            self.queues[agent_name] = []
        else:
            self.queues = {}
            self.history = []


class TestMultiAgentCommunication:
    """
    多 Agent 通信的集成测试。
    测试 Agent 间消息能否正确路由和解析。
    """

    @pytest.fixture
    def message_bus(self):
        return MessageBus()

    def test_message_routing(self, message_bus):
        """测试消息能否正确路由到目标 Agent"""
        msg = AgentMessage(
            sender="coordinator",
            recipient="worker_1",
            msg_type=MessageType.TASK,
            content="Search the web for AI news",
        )
        message_bus.send(msg)

        received = message_bus.receive("worker_1")
        assert received is not None
        assert received.sender == "coordinator"
        assert received.content == "Search the web for AI news"

        # 其他 Agent 不应收到此消息
        assert message_bus.receive("worker_2") is None

    def test_message_history(self, message_bus):
        """测试消息历史记录"""
        messages = [
            AgentMessage("A", "B", MessageType.QUERY, "Hello"),
            AgentMessage("B", "A", MessageType.RESPONSE, "Hi"),
            AgentMessage("A", "C", MessageType.TASK, "Do work"),
        ]
        for msg in messages:
            message_bus.send(msg)

        assert len(message_bus.history) == 3
        assert message_bus.history[0].content == "Hello"
        assert message_bus.history[-1].recipient == "C"

    def test_result_aggregation(self, message_bus):
        """测试多 Agent 结果汇总"""
        # 多个 worker 返回结果
        workers = ["worker_search", "worker_calc", "worker_summary"]
        for w in workers:
            message_bus.send(AgentMessage(
                sender=w,
                recipient="coordinator",
                msg_type=MessageType.RESULT,
                content=f"result_from_{w}",
            ))

        # coordinator 收集所有结果
        collected = []
        for _ in workers:
            msg = message_bus.receive("coordinator")
            if msg:
                collected.append(msg)

        assert len(collected) == 3
        assert all(msg.msg_type == MessageType.RESULT for msg in collected)
```

### 模式 5：状态持久化集成测试

测试 Agent 状态的保存和加载。

```python
import pytest
import json
import tempfile
from pathlib import Path


class AgentState:
    """可持久化的 Agent 状态"""

    def __init__(self):
        self.memory: dict = {}
        self.conversation_history: list = []
        self.tool_results: dict = {}
        self.metadata: dict = {
            "step_count": 0,
            "total_tokens_used": 0,
        }

    def save(self, path: Path) -> None:
        """持久化状态到磁盘"""
        state = {
            "memory": self.memory,
            "conversation_history": self.conversation_history,
            "tool_results": self.tool_results,
            "metadata": self.metadata,
        }
        with open(path, "w", encoding="utf-8") as f:
            json.dump(state, f, ensure_ascii=False, indent=2)

    @classmethod
    def load(cls, path: Path) -> "AgentState":
        """从磁盘恢复状态"""
        with open(path, "r", encoding="utf-8") as f:
            state = json.load(f)

        instance = cls()
        instance.memory = state["memory"]
        instance.conversation_history = state["conversation_history"]
        instance.tool_results = state["tool_results"]
        instance.metadata = state["metadata"]
        return instance

    def add_step(self, action: str, result: dict):
        """记录一步执行"""
        step = self.metadata["step_count"]
        self.tool_results[f"step_{step}"] = {
            "action": action,
            "result": result,
        }
        self.metadata["step_count"] += 1


class TestStatePersistenceIntegration:
    """
    状态持久化的集成测试。
    测试：状态记录 → 写入磁盘 → 从磁盘恢复 → 状态一致性验证。
    """

    @pytest.fixture
    def temp_state_file(self):
        with tempfile.NamedTemporaryFile(
            suffix=".json", delete=False, mode="w"
        ) as f:
            path = Path(f.name)
        yield path
        # Cleanup
        if path.exists():
            path.unlink()

    def test_save_and_load_cycle(self, temp_state_file):
        """测试完整的保存加载周期"""
        # 创建并填充状态
        state = AgentState()
        state.memory["user_name"] = "Alice"
        state.memory["preferences"] = {"language": "Python", "theme": "dark"}
        state.conversation_history = [
            {"role": "user", "content": "Hello"},
            {"role": "assistant", "content": "Hi Alice!"},
        ]
        state.add_step("search_web", {"results": ["doc1", "doc2"]})
        state.add_step("summarize", {"summary": "..."})

        # 保存
        state.save(temp_state_file)
        assert temp_state_file.exists()
        assert temp_state_file.stat().st_size > 0

        # 加载
        loaded = AgentState.load(temp_state_file)

        # 验证一致性
        assert loaded.memory["user_name"] == "Alice"
        assert loaded.memory["preferences"]["language"] == "Python"
        assert len(loaded.conversation_history) == 2
        assert loaded.metadata["step_count"] == 2
        assert "step_0" in loaded.tool_results
        assert loaded.tool_results["step_0"]["action"] == "search_web"

    def test_state_isolation(self, temp_state_file):
        """测试状态隔离：不同 Agent 的状态不相互影响"""
        # Agent A 的状态
        state_a = AgentState()
        state_a.memory["agent_name"] = "AgentA"
        state_a.metadata["step_count"] = 5

        # Agent B 的状态
        state_b = AgentState()
        state_b.memory["agent_name"] = "AgentB"
        state_b.metadata["step_count"] = 10

        # 保存
        state_a.save(temp_state_file)
        loaded = AgentState.load(temp_state_file)

        assert loaded.memory["agent_name"] == "AgentA"
        assert loaded.metadata["step_count"] == 5
        assert loaded.memory["agent_name"] != "AgentB"

    def test_state_corruption_handling(self, temp_state_file):
        """测试状态文件损坏时的处理"""
        # 写入损坏的数据
        with open(temp_state_file, "w") as f:
            f.write("this is not valid json")

        with pytest.raises(json.JSONDecodeError):
            AgentState.load(temp_state_file)
```

## LangGraph 集成测试

LangGraph 是构建 Agent 状态的流行框架，集成测试需要覆盖其核心机制。

### StateGraph 端到端测试

```python
import pytest
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, END


class AgentState(TypedDict):
    """Agent 的状态类型"""
    messages: list
    next_action: str
    tool_results: list
    finished: bool


class TestLangGraphIntegration:
    """
    LangGraph StateGraph 的集成测试。
    测试完整的图执行流程，包括节点路由和状态转换。
    """

    @pytest.fixture
    def simple_agent_graph(self):
        """构建一个简单的 Agent 图"""

        def call_llm(state: AgentState) -> AgentState:
            """模拟 LLM 调用节点"""
            # 在集成测试中，这里会调用真实 LLM
            last_msg = state["messages"][-1]["content"]

            if "weather" in last_msg.lower():
                return {
                    **state,
                    "next_action": "get_weather",
                }
            elif "calculate" in last_msg.lower():
                return {
                    **state,
                    "next_action": "calculate",
                }
            else:
                return {
                    **state,
                    "next_action": "respond",
                }

        def get_weather(state: AgentState) -> AgentState:
            """模拟天气工具调用"""
            return {
                **state,
                "tool_results": [*state["tool_results"], "weather_data"],
                "next_action": "respond",
            }

        def calculate(state: AgentState) -> AgentState:
            """模拟计算工具调用"""
            return {
                **state,
                "tool_results": [*state["tool_results"], "calc_result"],
                "next_action": "respond",
            }

        def respond(state: AgentState) -> AgentState:
            """模拟最终响应生成"""
            return {
                **state,
                "messages": [
                    *state["messages"],
                    {"role": "assistant", "content": "Done!"}
                ],
                "next_action": "__end__",
            }

        def router(state: AgentState) -> Literal["get_weather", "calculate", "respond"]:
            """条件路由函数"""
            return state["next_action"]

        # 构建图
        builder = StateGraph(AgentState)

        builder.add_node("call_llm", call_llm)
        builder.add_node("get_weather", get_weather)
        builder.add_node("calculate", calculate)
        builder.add_node("respond", respond)

        builder.set_entry_point("call_llm")
        builder.add_conditional_edges(
            "call_llm",
            router,
            {
                "get_weather": "get_weather",
                "calculate": "calculate",
                "respond": "respond",
            }
        )
        builder.add_edge("get_weather", "respond")
        builder.add_edge("calculate", "respond")
        builder.add_edge("respond", END)

        return builder.compile()

    def test_weather_flow(self, simple_agent_graph):
        """测试天气查询流程"""
        initial_state = {
            "messages": [{"role": "user", "content": "What's the weather?"}],
            "next_action": "",
            "tool_results": [],
            "finished": False,
        }

        result = simple_agent_graph.invoke(initial_state)

        assert "weather_data" in result["tool_results"]
        assert result["messages"][-1]["content"] == "Done!"

    def test_calculate_flow(self, simple_agent_graph):
        """测试计算流程"""
        initial_state = {
            "messages": [{"role": "user", "content": "Calculate 2+2"}],
            "next_action": "",
            "tool_results": [],
            "finished": False,
        }

        result = simple_agent_graph.invoke(initial_state)

        assert "calc_result" in result["tool_results"]
        assert result["messages"][-1]["content"] == "Done!"

    def test_direct_response_flow(self, simple_agent_graph):
        """测试不需要工具的直接响应流程"""
        initial_state = {
            "messages": [{"role": "user", "content": "Hello!"}],
            "next_action": "",
            "tool_results": [],
            "finished": False,
        }

        result = simple_agent_graph.invoke(initial_state)

        assert len(result["tool_results"]) == 0  # 不需要工具
        assert result["messages"][-1]["content"] == "Done!"
```

### Checkpointer 持久化测试

```python
@pytest.mark.langgraph
class TestLangGraphCheckpointer:
    """
    LangGraph Checkpointer（检查点）的集成测试。
    测试 Agent 状态的保存、恢复和分支。
    """

    @pytest.fixture
    def graph_with_checkpointer(self):
        """带有检查点的图"""
        from langgraph.checkpoint import MemorySaver

        def process(state: AgentState) -> AgentState:
            return {
                **state,
                "messages": [
                    *state["messages"],
                    {"role": "assistant", "content": f"Step {len(state['messages'])}"}
                ],
                "next_action": "__end__",
            }

        builder = StateGraph(AgentState)
        builder.add_node("process", process)
        builder.set_entry_point("process")
        builder.add_edge("process", END)

        memory = MemorySaver()
        return builder.compile(checkpointer=memory)

    def test_state_persistence(self, graph_with_checkpointer):
        """测试状态持久化和恢复"""
        # 第一次执行
        config = {"configurable": {"thread_id": "test_thread_1"}}
        result1 = graph_with_checkpointer.invoke(
            {"messages": [{"role": "user", "content": "First message"}],
             "next_action": "", "tool_results": [], "finished": False},
            config=config,
        )

        assert len(result1["messages"]) == 2

        # 第二次执行（同一个 thread）
        result2 = graph_with_checkpointer.invoke(
            {"messages": [{"role": "user", "content": "Second message"}],
             "next_action": "", "tool_results": [], "finished": False},
            config=config,
        )

        # 历史消息被保留
        assert len(result2["messages"]) == 4

    def test_thread_isolation(self, graph_with_checkpointer):
        """测试不同 thread 的状态隔离"""
        # Thread A
        result_a = graph_with_checkpointer.invoke(
            {"messages": [{"role": "user", "content": "Thread A message"}],
             "next_action": "", "tool_results": [], "finished": False},
            config={"configurable": {"thread_id": "thread_a"}},
        )

        # Thread B（新的 thread，不应该有 A 的历史）
        result_b = graph_with_checkpointer.invoke(
            {"messages": [{"role": "user", "content": "Thread B message"}],
             "next_action": "", "tool_results": [], "finished": False},
            config={"configurable": {"thread_id": "thread_b"}},
        )

        assert len(result_a["messages"]) == 2
        assert len(result_b["messages"]) == 2

    def test_state_rollback(self, graph_with_checkpointer):
        """测试状态回滚"""
        config = {"configurable": {"thread_id": "test_rollback"}}

        graph_with_checkpointer.invoke(
            {"messages": [{"role": "user", "content": "Step 1"}],
             "next_action": "", "tool_results": [], "finished": False},
            config=config,
        )

        graph_with_checkpointer.invoke(
            {"messages": [{"role": "user", "content": "Step 2"}],
             "next_action": "", "tool_results": [], "finished": False},
            config=config,
        )

        # 获取执行历史中的所有状态
        state_history = list(graph_with_checkpointer.get_state_history(config))
        assert len(state_history) > 0

        # 回滚到第一个状态
        first_checkpoint = state_history[-1]  # 最早的 checkpoint
        rollback_config = {
            "configurable": {
                "thread_id": "test_rollback",
                "checkpoint_id": first_checkpoint.config["configurable"]["checkpoint_id"]
            }
        }

        rolled_back = graph_with_checkpointer.get_state(rollback_config)
        assert len(rolled_back.values["messages"]) == 2  # 只有 Step 1
```

### 条件边路由测试

```python
class TestLangGraphConditionalRouting:
    """
    条件边的路由逻辑测试。
    确保不同输入能路由到正确的处理路径。
    """

    def test_router_logic(self):
        """测试条件路由函数的逻辑正确性"""

        def route_by_intent(state: AgentState) -> Literal["search", "calculate", "chat"]:
            last_msg = state["messages"][-1]["content"].lower()

            if any(kw in last_msg for kw in ["search", "find", "look up"]):
                return "search"
            elif any(kw in last_msg for kw in ["calculate", "sum", "count"]):
                return "calculate"
            else:
                return "chat"

        # 测试各路由分支
        assert route_by_intent({
            "messages": [{"role": "user", "content": "Search for AI papers"}],
            "next_action": "", "tool_results": [], "finished": False
        }) == "search"

        assert route_by_intent({
            "messages": [{"role": "user", "content": "Calculate 5*10"}],
            "next_action": "", "tool_results": [], "finished": False
        }) == "calculate"

        assert route_by_intent({
            "messages": [{"role": "user", "content": "Hello, how are you?"}],
            "next_action": "", "tool_results": [], "finished": False
        }) == "chat"

    def test_unknown_intent_fallback(self):
        """测试未知意图的 fallback 行为"""

        def route_with_fallback(state: AgentState) -> str:
            last_msg = state["messages"][-1]["content"].lower()

            if "weather" in last_msg:
                return "weather_node"
            elif "search" in last_msg:
                return "search_node"
            else:
                # Fallback 到默认路径
                return "default_node"

        # 未知意图应该走 default
        assert route_with_fallback({
            "messages": [{"role": "user", "content": "Do something random"}],
            "next_action": "", "tool_results": [], "finished": False
        }) == "default_node"
```

### Streaming + Interrupt 测试

```python
class TestLangGraphStreaming:
    """
    流式输出和中断的集成测试。
    测试 Agent 在流式执行过程中的行为。
    """

    @pytest.mark.asyncio
    async def test_streaming_execution(self, simple_agent_graph):
        """测试流式执行是否产出中间状态"""
        initial_state = {
            "messages": [{"role": "user", "content": "What's the weather?"}],
            "next_action": "",
            "tool_results": [],
            "finished": False,
        }

        events = []
        async for event in simple_agent_graph.astream_events(initial_state, version="v1"):
            events.append(event)

        # 流式事件应该包含多个阶段
        assert len(events) > 1
        # 应该有节点的开始和结束事件
        node_events = [e for e in events if e["event"].startswith("on_node_")]
        assert len(node_events) >= 2  # 至少经过 call_llm + get_weather/respond

    def test_interrupt_and_resume(self):
        """测试中断和恢复"""
        from langgraph.graph import StateGraph, END
        from langgraph.checkpoint import MemorySaver
        from langgraph.constants import INTERRUPT

        def step1(state: AgentState) -> AgentState:
            return {
                **state,
                "messages": [*state["messages"], {"role": "assistant", "content": "Need user input"}],
            }

        def step2(state: AgentState) -> AgentState:
            return {
                **state,
                "messages": [*state["messages"], {"role": "assistant", "content": "Continuing after interrupt"}],
            }

        # 构建包含中断的图
        builder = StateGraph(AgentState)
        builder.add_node("step1", step1)
        builder.add_node("step2", step2)
        builder.set_entry_point("step1")
        builder.add_edge("step1", INTERRUPT)  # 这里会中断
        builder.add_edge("step2", END)

        memory = MemorySaver()
        graph = builder.compile(checkpointer=memory)

        config = {"configurable": {"thread_id": "interrupt_test"}}

        # 第一次 invoke → 在 step1 后中断
        result = graph.invoke(
            {"messages": [{"role": "user", "content": "Start"}],
             "next_action": "", "tool_results": [], "finished": False},
            config=config,
        )

        # 验证中断发生
        state = graph.get_state(config)
        assert state.next == ("step2",)  # 下一个节点是 step2

        # 恢复执行
        result = graph.invoke(None, config=config)
        assert result["messages"][-1]["content"] == "Continuing after interrupt"
```

## 测试基础设施

### pytest Fixtures 模式

```python
import pytest
from pathlib import Path
from typing import Generator
import json
import os


# ===========================================================
# conftest.py 中的共享 Fixture 模式
# ===========================================================

@pytest.fixture(scope="session")
def test_config() -> IntegrationTestConfig:
    """会话级别的共享配置"""
    config = IntegrationTestConfig()

    # CI 环境检测
    if os.getenv("CI"):
        config.llm_model = "gpt-4o-mini"  # CI 中使用更快的模型
        config.weather_api_url = "http://docker-weather:8080/weather"
        config.replay_mode = False  # CI 中不依赖回放缓存
    else:
        # 本地开发使用轻量 fixture
        config.weather_api_url = "http://localhost:8080/weather"

    return config


@pytest.fixture
def llm_client(test_config):
    """每个测试函数获得独立的 LLM 客户端"""
    from openai import OpenAI
    client = OpenAI()
    # 为每个测试设置唯一的请求标签
    client.default_headers = {
        "X-Test-Name": os.environ.get("PYTEST_CURRENT_TEST", "unknown")
    }
    yield client


@pytest.fixture
def agent_tools(test_config):
    """提供 Agent 可用的工具集合——在集成测试中指向 fixture 服务"""
    from myagent.tools import WeatherTool, SearchTool, CalculatorTool

    return {
        "weather": WeatherTool(api_url=test_config.weather_api_url),
        "search": SearchTool(api_url=test_config.search_api_url),
        "calculator": CalculatorTool(),
    }


@pytest.fixture(autouse=True)
def test_cleanup():
    """每个测试自动执行的清理"""
    yield
    # 测试结束后清理临时文件
    test_dir = Path("./test_temp")
    if test_dir.exists():
        import shutil
        shutil.rmtree(test_dir)


@pytest.fixture(scope="module")
def docker_services() -> Generator:
    """模块级别的 Docker 服务管理"""
    import subprocess

    # 启动测试服务
    subprocess.run(
        ["docker-compose", "-f", "docker-compose.test.yml", "up", "-d"],
        check=True,
    )

    # 等待服务就绪
    import requests
    import time
    for _ in range(30):
        try:
            r = requests.get("http://localhost:8080/health", timeout=2)
            if r.status_code == 200:
                break
        except requests.ConnectionError:
            time.sleep(1)

    yield

    # 清理
    subprocess.run(
        ["docker-compose", "-f", "docker-compose.test.yml", "down", "-v"],
        check=True,
    )
```

### 录制/回放外部交互

使用 VCR.py 录制和回放 HTTP 交互，让集成测试既真实又可重复。

```python
import pytest
import vcr


# VCR 配置
vcr_config = vcr.VCR(
    cassette_library_dir="./test_cassettes",
    record_mode="once",  # first run: record, subsequent: replay
    match_on=["uri", "method", "body"],
    filter_headers=["authorization"],  # 不录制敏感信息
    filter_query_parameters=["api_key"],
    decode_compressed_response=True,
)


class TestRecordedIntegration:
    """
    使用 VCR 录制/回放的集成测试。
    """

    @vcr_config.use_cassette("weather_beijing.yaml")
    def test_recorded_weather(self):
        """第一次运行录制真实调用，之后回放"""
        import requests
        response = requests.get(
            "https://api.weather.com/v1/Beijing",
            params={"units": "metric"},
            headers={"Authorization": "Bearer test-key"},
        )
        data = response.json()
        assert "temperature" in data

    @vcr_config.use_cassette("agent_loop_basic.yaml")
    def test_recorded_agent_loop(self, llm_client, mock_tools):
        """
        录制 Agent 循环的完整交互。
        包括多个 LLM 调用和工具调用的序列。
        """
        agent = AgentLoop(llm_client, mock_tools, max_steps=5)
        result = agent.run("Search for AI news and summarize")

        assert result["success"] is True
        # 可以从录制中回放，无需真实 API 调用


# 更精细的控制——按环境切换录制模式
@pytest.fixture(scope="session")
def vcr_mode(test_config):
    """
    根据配置选择 VCR 模式：
      - CI:  only_new (必须录制新交互)
      - 本地: once (录制一次后回放)
      - 调试: all (总是重新录制)
    """
    if os.getenv("CI"):
        return "none"  # CI 中不使用回放
    if os.getenv("RECORD_MODE") == "all":
        return "all"
    return "once"
```

### HTTP Test Servers

使用 `responses` 或 `wiremock` 模拟 HTTP 服务。

```python
# 使用 responses 库模拟 HTTP 服务
import responses
import pytest


class TestWithResponsesLibrary:
    """使用 responses 库的 HTTP 服务模拟"""

    @responses.activate
    def test_weather_api_mock(self):
        """模拟天气 API 响应"""
        responses.add(
            responses.GET,
            "https://api.weather.com/v1/Beijing",
            json={"temperature": 25, "condition": "sunny"},
            status=200,
        )

        tool = WeatherTool(api_url="https://api.weather.com/v1")
        result = tool.get_weather("Beijing")

        assert result["success"] is True
        assert result["temperature"] == 25

    @responses.activate
    def test_weather_api_error(self):
        """模拟 API 错误"""
        responses.add(
            responses.GET,
            "https://api.weather.com/v1/UnknownCity",
            json={"error": "not_found"},
            status=404,
        )

        tool = WeatherTool(api_url="https://api.weather.com/v1")
        result = tool.get_weather("UnknownCity")

        assert result["success"] is False
        assert result["status_code"] == 404

    @responses.activate
    def test_retry_on_server_error(self):
        """模拟服务器 500 错误后的重试"""
        import time

        # 第一次调用返回 500
        responses.add(
            responses.GET,
            "https://api.weather.com/v1/Beijing",
            body="Internal Server Error",
            status=500,
        )
        # 重试后成功
        responses.add(
            responses.GET,
            "https://api.weather.com/v1/Beijing",
            json={"temperature": 25, "condition": "sunny"},
            status=200,
        )

        tool = WeatherTool(api_url="https://api.weather.com/v1", max_retries=2)
        result = tool.get_weather("Beijing")

        assert result["success"] is True
        # 验证确实发生了重试
        assert len(responses.calls) == 2
```

### Docker Compose 测试环境

```yaml
# docker-compose.test.yml
# 集成测试的完整服务依赖

version: "3.8"

services:
  # 测试用的天气 API fixture
  weather-api:
    build:
      context: ./test_fixtures
      dockerfile: Dockerfile.weather
    ports:
      - "8080:8080"
    environment:
      - FLASK_ENV=testing
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 5s
      timeout: 3s
      retries: 3

  # 测试用的搜索 API fixture
  search-api:
    image: "test-search-api:latest"
    ports:
      - "8081:8081"
    environment:
      - INDEX_PATH=/fixtures/search_index.json
    volumes:
      - ./test_fixtures/search_data:/fixtures

  # 测试用的向量数据库
  vector-store:
    image: "chromadb/chroma:latest"
    ports:
      - "8082:8000"
    environment:
      - CHROMA_DB_IMPL=duckdb+parquet
      - PERSIST_DIR=/chroma-data
    volumes:
      - test-vector-data:/chroma-data

  # 测试专用的 Redis（用于状态缓存）
  redis:
    image: "redis:7-alpine"
    ports:
      - "6379:6379"
    command: ["redis-server", "--save", ""]  # 不持久化

volumes:
  test-vector-data:
```

## 集成测试策略

### 关键路径测试

识别 Agent 最核心的用户旅程，优先覆盖。

```
关键路径示例——一个客服 Agent：

路径 1: 正常查询
  用户提问 ──→ LLM 理解 ──→ 查询知识库 ──→ 生成回答
  ✅ 必须通过

路径 2: 需要人工转接
  用户提问 ──→ LLM 理解 ──→ 知识库无结果
              ──→ 升级到人工客服
  ✅ 必须通过

路径 3: 多轮对话
  用户提问 ──→ 回答 ──→ 追问细节 ──→ 深入回答
  ✅ 必须通过

路径 4: 工具链调用
  用户请求 ──→ 查天气 ──→ 计算温差 ──→ 给出建议
  ✅ 必须通过
```

### 错误路径测试

```python
class TestErrorPathIntegration:
    """
    错误路径集成测试。
    测试 Agent 在各种异常情况下的行为。
    """

    @pytest.fixture
    def flaky_api(self):
        """模拟不稳定 API"""
        import random

        class FlakyWeatherAPI:
            def __init__(self):
                self.call_count = 0

            def get_weather(self, city):
                self.call_count += 1
                if self.call_count % 3 == 0:
                    raise requests.exceptions.Timeout("Simulated timeout")
                return {"temperature": 25, "condition": "sunny"}

        return FlakyWeatherAPI()

    def test_api_timeout_recovery(self, llm_client, flaky_api):
        """
        测试 API 超时后的恢复：
        1. 第一次调用超时
        2. Agent 应重试
        3. 重试成功后继续
        """
        agent = AgentLoop(
            llm_client=llm_client,
            tools={"weather": flaky_api},
            max_steps=5,
        )

        result = agent.run("What's the weather in Beijing?")

        assert result["success"] is True
        assert flaky_api.call_count >= 2  # 至少发生了重试

    def test_all_apis_down(self):
        """
        测试所有 API 不可用时的优雅降级。
        Agent 应该给出合理的错误信息而不是崩溃。
        """
        def broken_api(**kwargs):
            raise ConnectionError("Service unavailable")

        agent = AgentLoop(
            llm_client=llm_client,
            tools={"weather": broken_api},
            max_steps=3,
        )

        result = agent.run("What's the weather?")

        # 不应该崩溃
        assert "error" in result or result["output"] is not None
        # 输出应该说明服务不可用
        if result.get("output"):
            assert any(
                kw in result["output"].lower()
                for kw in ["unavailable", "error", "sorry", "down"]
            )

    def test_rate_limit_handling(self):
        """
        测试速率限制处理。
        Agent 收到 429 后应该等待并重试。
        """
        import time

        call_timestamps = []

        def rate_limited_api(**kwargs):
            call_timestamps.append(time.time())
            if len(call_timestamps) < 3:
                return {"error": "rate_limited", "retry_after": 1}
            return {"temperature": 25}

        agent = AgentLoop(
            llm_client=llm_client,
            tools={"weather": rate_limited_api},
            max_steps=5,
        )

        result = agent.run("What's the weather?")

        assert result["success"] is True
        # 验证调用间隔（应该有等待）
        if len(call_timestamps) >= 3:
            intervals = [
                call_timestamps[i+1] - call_timestamps[i]
                for i in range(len(call_timestamps)-1)
            ]
            assert any(i > 0.5 for i in intervals)  # 有等待
```

### 确定性模式测试

```python
class TestDeterministicMode:
    """
    确定性模式集成测试。
    使用 temperature=0 和固定 seed 来减少非确定性。
    """

    N_RUNS = 3  # 每个测试运行 N 次

    @pytest.mark.parametrize("run", range(N_RUNS))
    def test_deterministic_llm_output(self, llm_client, run):
        """
        验证 temperature=0 下多次调用结果的一致性。
        虽然 LLM 仍有一定非确定性，但应基本一致。
        """
        response = llm_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "Answer with exactly one word."},
                {"role": "user", "content": "What is the capital of France?"}
            ],
            temperature=0,
            seed=42,  # OpenAI 支持 seed 参数
        )

        answer = response.choices[0].message.content.strip().lower()
        assert "paris" in answer

    @pytest.mark.parametrize("run", range(N_RUNS))
    def test_deterministic_tool_selection(self, llm_client, mock_tools, run):
        """
        验证同一输入下 Agent 选择相同的工具序列。
        """
        agent = AgentLoop(
            llm_client=llm_client,
            tools=mock_tools,
            max_steps=5,
        )

        result = agent.run("Calculate 15 * 7")

        # 应该始终选择 calculator 工具
        tool_names = [
            msg.function.name
            for msg in result["messages"]
            if hasattr(msg, "function") and msg.function
        ]
        assert "calculator" in tool_names
```

### 回归测试套件组织

```
tests/
├── integration/
│   ├── conftest.py                    # 共享 fixtures
│   ├── test_config.py                 # 配置管理测试
│   │
│   ├── tools/
│   │   ├── test_weather_tool.py       # 天气工具集成
│   │   ├── test_search_tool.py        # 搜索工具集成
│   │   └── test_database_tool.py      # 数据库工具集成
│   │
│   ├── agents/
│   │   ├── test_single_step.py        # 单步 Agent 测试
│   │   ├── test_multi_step.py         # 多步 Agent 测试
│   │   └── test_error_recovery.py     # 错误恢复测试
│   │
│   ├── langgraph/
│   │   ├── test_state_graph.py        # StateGraph 测试
│   │   ├── test_checkpointer.py       # 检查点测试
│   │   └── test_streaming.py          # 流式测试
│   │
│   ├── multi_agent/
│   │   ├── test_message_passing.py    # 消息传递
│   │   └── test_task_delegation.py    # 任务委派
│   │
│   └── regression/
│       ├── test_critical_paths.py     # 关键路径回归
│       └── test_known_issues.py       # 已知问题回归
│
├── cassettes/                         # VCR 录制文件
│   ├── weather_beijing.yaml
│   ├── weather_london.yaml
│   └── agent_loop_basic.yaml
│
├── docker-compose.test.yml           # 测试服务编排
└── pytest.ini                         # pytest 配置
```

## CI/CD 集成

### 集成测试在 Pipeline 中的位置

```
CI Pipeline 中的测试阶段：

                 ┌──────────┐
                 │  Lint    │  代码风格检查
                 └────┬─────┘
                      │
                 ┌────▼─────┐
                 │  Unit    │  单元测试（快速、无外部依赖）
                 │  Tests   │  ~2 分钟
                 └────┬─────┘
                      │ PASS
                      ▼
              ┌───────────────┐
              │  Integration  │  集成测试（需要外部服务）
              │  Tests         │  ~15 分钟
              │               │
              │  ┌─────────┐  │
              │  │ Provision│  │  启动 Docker 测试服务
              │  │ Env     │  │
              │  ├─────────┤  │
              │  │ Run     │  │  并行执行集成测试
              │  │ Tests   │  │
              │  ├─────────┤  │
              │  │ Cleanup │  │  关闭测试服务
              │  └─────────┘  │
              └───────┬───────┘
                      │ PASS
                      ▼
              ┌───────────────┐
              │  E2E Tests   │  全链路测试
              │               │  ~30 分钟
              └───────┬───────┘
                      │ PASS
                      ▼
                 ┌──────────┐
                 │  Deploy  │
                 └──────────┘
```

### 并行执行策略

```python
# pytest.ini
[pytest]
testpaths = tests/integration
markers =
    integration: Integration tests (real dependencies)
    slow: Tests that take more than 30 seconds
    langgraph: LangGraph-specific tests
    deterministic: Tests expected to be deterministic

# 并行执行配置
# pytest -n auto tests/integration/         ← pytest-xdist 自动分配
# pytest -n 4 --dist loadscope              ← 按测试文件分组
```

```yaml
# .github/workflows/integration-tests.yml
name: Agent Integration Tests
on:
  push:
    branches: [main, staging]
  pull_request:
    branches: [main]

jobs:
  integration-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        # 将测试分组以控制并行度
        test-group:
          - tools
          - agents
          - langgraph
          - multi-agent
          - regression

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Start Test Services
        run: docker-compose -f docker-compose.test.yml up -d

      - name: Wait for Services
        run: |
          for i in $(seq 1 30); do
            curl -s http://localhost:8080/health && break
            sleep 2
          done

      - name: Run Integration Tests (${{ matrix.test-group }})
        env:
          TEST_LLM_MODEL: gpt-4o-mini
          CI: "true"
        run: |
          pytest tests/integration/${{ matrix.test-group }} \
            -v \
            --timeout=120 \
            -x \
            --junitxml=reports/${{ matrix.test-group }}.xml

      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports-${{ matrix.test-group }}
          path: reports/

      - name: Cleanup
        if: always()
        run: docker-compose -f docker-compose.test.yml down -v
```

### 测试结果报告

```python
# 自定义测试报告格式
INTEGRATION_TEST_REPORT = """
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Agent 集成测试报告
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

时间戳: {timestamp}
环境:   {environment}
模型:   {llm_model}

─────────────────────────────────────────────────────
结果摘要
─────────────────────────────────────────────────────
总计:  {total}  |  通过:  {passed}  |  失败:  {failed}
跳过:  {skipped}  |  耗时:  {duration:.1f}s
通过率:  {pass_rate:.1f}%

─────────────────────────────────────────────────────
关键路径测试状态
─────────────────────────────────────────────────────
{critical_paths}

─────────────────────────────────────────────────────
失败的测试详情
─────────────────────────────────────────────────────
{failures}

─────────────────────────────────────────────────────
LLM 调用统计
─────────────────────────────────────────────────────
总 Token 消耗: {total_tokens}
预估成本:      ${estimated_cost:.4f}
平均每次测试:   {avg_tokens_per_test} tokens

─────────────────────────────────────────────────────
环境状态
─────────────────────────────────────────────────────
{service_status}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"""
```

## 挑战与局限

### 1. Flaky Tests（不稳定测试）

LLM 的非确定性输出是集成测试最大的痛点。

```
Flaky Test 的来源与缓解：

来源 1: LLM 输出的非确定性
  ┌─ 即使 temperature=0，模型输出仍可能有微小差异
  ├─ 缓解：断言时使用语义匹配而非精确匹配
  └─ 例：assert "good" in output 而非 assert output == "good"

来源 2: 外部服务不稳定
  ┌─ 天气 API 偶尔超时、搜索 API 返回不同结果
  ├─ 缓解：使用 VCR 回放、重试机制
  └─ 例：自动重试 2 次后才标记为失败

来源 3: 环境差异
  ┌─ 本地环境和 CI 环境的时间/网络/配置不同
  ├─ 缓解：容器化测试环境、环境变量统一管理
  └─ 例：使用 docker-compose 确保服务一致

来源 4: 时间敏感
  ┌─ 依赖当前时间的结果（如"今天的天气"）
  ├─ 缓解：mock 时间、使用固定测试数据
  └─ 例：freezegun 库固定测试时间
```

### 2. 环境一致性

```
跨机器一致性挑战：

不同机器：
  本地 Mac ──→ Docker Desktop ──→ 可能 OK
  CI Linux ──→ 原生 Docker ──→ 可能 OK
  同事 Windows ──→ WSL2 Docker ──→ 可能有问题

解决方案：
  1. 所有外部服务走 Docker，确保版本一致
  2. 测试配置文件（test_config.py）统一管理差异
  3. CI 中强制使用与本地相同的 Docker 镜像版本
  4. 使用 Makefile 封装"一条命令运行所有测试"
     $ make test-integration
```

### 3. Token 成本

```python
# 集成测试的 LLM 成本估算
LLM_COST_ESTIMATE = {
    "model": "gpt-4o-mini",
    "price_per_1k_input": 0.00015,   # $0.15/1M tokens
    "price_per_1k_output": 0.00060,  # $0.60/1M tokens

    # 单次集成测试的典型消耗
    "per_test": {
        "input_tokens": 2000,
        "output_tokens": 500,
        "cost_per_test": 2000 * 0.00015/1000 + 500 * 0.00060/1000,
        # = $0.0003 + $0.0003 = $0.0006
    }
}

# 成本分析
TOTAL_COST_ANALYSIS = """
集成测试数量               每月 LLM 成本（gpt-4o-mini）
─────────────────────  ──────────────────────────────
100 测试 × 3 次运行        100 × 3 × $0.0006 = $0.18
500 测试 × 3 次运行        500 × 3 × $0.0006 = $0.90
2000 测试 × 3 次运行       2000 × 3 × $0.0006 = $3.60

如果使用 gpt-4o (30x 价格):
2000 测试 × 3 次运行       2000 × 3 × $0.018 = $108.00

成本控制策略:
  1. 90% 测试使用最便宜的模型（gpt-4o-mini）
  2. 关键路径测试才用强模型
  3. 使用 VCR 录制回放 LLM 调用
  4. 测试失败时缓存 LLM 响应避免重复调用
"""
```

### 4. 测试执行时间 vs 覆盖率

```
时间-覆盖率权衡：

        覆盖率
          ↑
  100% ────● (不可能——成本无限)
          │ \
   80% ────●──● (集成测试最佳点)
          │    \
   50% ────●─────● (单元测试)
          │        \
    0% ──────────────●──→ 时间/成本
                     (完全不测)

经验法则：
  • 20% 的关键路径测试覆盖 80% 的用户场景
  • 集成测试覆盖核心交互，单元测试覆盖边缘情况
  • 运行时间超过 10 分钟的集成测试套件需要优化
  • 单测试超过 30 秒 → 考虑是否可以用轻量方式替代
```

### 5. 调试困难

LLM 参与的集成测试失败时，调试比传统软件更难。

```
集成测试调试流程：

传统软件测试：
  断言失败 ──→ 检查输入/输出 ──→ 找到 bug ──→ 修复

Agent 集成测试：
  断言失败 ──→ 哪一步出错了？
    ├─ LLM 理解错了？（Prompt 问题？）
    ├─ 工具调用错了？（参数提取问题？）
    ├─ 工具返回错了？（服务端问题？）
    └─ Agent 策略错了？（不该调这个工具？）
            ↓
   需要查看 Agent 的完整运行轨迹
            ↓
   轨迹可能涉及 5-10 步，每一步都有 LLM 调用
            ↓
   难以复现（非确定性）
            ↓
   调试手段：
     1. 记录完整轨迹到日志
     2. 复现时固定 temperature=0
     3. 逐步执行（interrupt 模式）
     4. 对比成功/失败运行轨迹的差异
```

```python
import logging

# 集成测试的日志配置
LOG_CONFIG = {
    "version": 1,
    "handlers": {
        "agent_trace": {
            "class": "logging.FileHandler",
            "filename": "test_traces/agent_trace.log",
            "formatter": "trace",
        }
    },
    "formatters": {
        "trace": {
            "format": (
                "\n{separator}\n"
                "Test: {test_name}\n"
                "Step: {step}\n"
                "Time: {asctime}\n"
                "Action: {action}\n"
                "LLM Input: {llm_input}\n"
                "LLM Output: {llm_output}\n"
                "Tool Results: {tool_results}\n"
                "{separator}\n"
            ),
            "style": "{",
        }
    },
    "loggers": {
        "agent_test": {
            "level": "DEBUG",
            "handlers": ["agent_trace"],
        }
    }
}


class IntegrationTestDebugger:
    """
    集成测试调试助手。
    记录 Agent 的完整执行轨迹用于失败分析。
    """

    def __init__(self, test_name: str):
        self.test_name = test_name
        self.trace = []
        self.logger = logging.getLogger("agent_test")

    def record_step(self, step: int, action: str,
                    llm_input: str, llm_output: str,
                    tool_results: list):
        """记录一步执行"""
        entry = {
            "step": step,
            "action": action,
            "llm_input": llm_input[:500],   # 截断防止日志过大
            "llm_output": llm_output[:500],
            "tool_results": tool_results,
        }
        self.trace.append(entry)

        self.logger.debug("Step record", extra={
            "test_name": self.test_name,
            "step": step,
            "action": action,
            "llm_input": llm_input[:200],
            "llm_output": llm_output[:200],
            "tool_results": str(tool_results)[:200],
            "separator": "─" * 60,
        })

    def save_trace(self, output_dir: str = "test_traces"):
        """保存完整轨迹到文件"""
        import json
        from pathlib import Path

        save_path = Path(output_dir)
        save_path.mkdir(parents=True, exist_ok=True)

        trace_file = save_path / f"{self.test_name}.json"
        with open(trace_file, "w") as f:
            json.dump({
                "test_name": self.test_name,
                "steps": self.trace,
                "step_count": len(self.trace),
            }, f, indent=2)

        return trace_file

    def compare_traces(self, trace_a_path: str, trace_b_path: str) -> dict:
        """比较两条轨迹，找出差异点（用于回归分析）"""
        import json

        with open(trace_a_path) as f:
            trace_a = json.load(f)
        with open(trace_b_path) as f:
            trace_b = json.load(f)

        differences = []
        max_steps = max(len(trace_a["steps"]), len(trace_b["steps"]))

        for i in range(max_steps):
            step_a = trace_a["steps"][i] if i < len(trace_a["steps"]) else None
            step_b = trace_b["steps"][i] if i < len(trace_b["steps"]) else None

            if step_a is None or step_b is None:
                differences.append({
                    "step": i,
                    "type": "step_count_differs",
                    "a": step_a,
                    "b": step_b,
                })
            elif step_a["action"] != step_b["action"]:
                differences.append({
                    "step": i,
                    "type": "action_differs",
                    "a_action": step_a["action"],
                    "b_action": step_b["action"],
                })

        return {
            "trace_a": trace_a_path,
            "trace_b": trace_b_path,
            "differences": differences,
            "diff_count": len(differences),
        }
```

## 最佳实践总结

```
集成测试的十大最佳实践：

1. 分层测试
   单元测试跑得快的问题，集成测试不重复测。
   集成测试聚焦"组件间的交互"。

2. 用最便宜的模型
   集成测试用 gpt-4o-mini，关键路径才用 gpt-4o。
   省下的钱可以跑更多测试。

3. 录制回放
   首次运行录制真实交互，之后回放。
   兼顾"真实"和"可重复"。

4. 环境容器化
   所有外部依赖走 Docker。
   "docker-compose up" 一键启动测试环境。

5. 测试隔离
   每个测试独立命名空间，独立数据。
   测试结束后自动清理。

6. 确定性优先
   使用 temperature=0，固定 seed。
   断言用语义匹配而非精确匹配。

7. 超时控制
   每个工具调用设置超时。
   整个 Agent 流程设置超时。
   防止测试"挂死"。

8. 错误路径覆盖
   API 超时、服务 500、格式错误……
   坏的情况比好的情况更需要测试。

9. 轨迹记录
   失败的测试自动保存执行轨迹。
   调试时直接用轨迹分析。

10. 成本透明
    在测试报告中显示 Token 消耗和成本。
    让团队感知每次测试的真实成本。
```
