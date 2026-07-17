# 11.4.5 Simulation Env — 仿真测试环境构建

## 简单介绍

仿真测试环境（Simulation Environment）是对抗测试的基础设施。没有好的仿真环境，对抗测试要么不可重复（依赖真实环境的不确定性），要么不真实（在隔离环境中测试出来的结果与生产表现脱节）。

```
仿真环境的三个核心需求：

可重复性                        真实性                        安全性
┌──────────────┐           ┌──────────────┐           ┌──────────────┐
│ 同一个测试     │           │ 环境行为与     │           │ 对抗测试不    │
│ 每次运行结果   │           │ 生产环境一致   │           │ 会影响真实    │
│ 相同          │           │              │           │ 用户和数据    │
└──────────────┘           └──────────────┘           └──────────────┘
       ↕                          ↕                          ↕
    必须三者兼顾——缺少任何一个维度，仿真环境的价值都大打折扣
```

```
Agent 仿真环境 vs 传统仿真环境：

传统仿真环境（游戏/Autonomous Driving）：
  • 关注物理世界的模拟
  • 确定性状态转换
  • 可枚举的动作空间
  • 成熟的模拟引擎（Unreal、CARLA）

Agent 仿真环境：
  • 关注信息世界的模拟
  • 概率性状态转换（LLM 的不确定性）
  • 开放的动作空间（自然语言）
  • 需要模拟：用户、工具、知识库、多 Agent
```

## 基本原理

### 仿真环境的架构

Agent 仿真环境由多个子环境组成，每个子环境模拟 Agent 交互的不同方面：

```
┌────────────────────────────────────────────────────────────┐
│                    Agent 仿真测试环境                         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐ │
│  │   用户模拟器     │  │   工具模拟器     │  │  知识库模拟器  │ │
│  ├────────────────┤  ├────────────────┤  ├──────────────┤ │
│  │ 生成用户输入    │  │ 模拟 API 响应    │  │ 创建/管理知识  │ │
│  │ 模拟用户行为    │  │ 注入延迟/错误    │  │ 控制检索结果  │ │
│  │ 提供用户反馈    │  │ 记录调用日志    │  │ 模拟检索噪声  │ │
│  └────────────────┘  └────────────────┘  └──────────────┘ │
│                                                            │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐ │
│  │   环境控制器    │  │   监控记录器    │  │  场景管理器   │ │
│  ├────────────────┤  ├────────────────┤  ├──────────────┤ │
│  │ 控制模拟参数   │  │ 记录所有交互    │  │ 加载测试场景  │ │
│  │ 管理时间推进   │  │ 追踪状态变化   │  │ 管理场景参数  │ │
│  │ 控制随机性    │  │ 导出跟踪数据   │  │ 切换测试条件  │ │
│  └────────────────┘  └────────────────┘  └──────────────┘ │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 仿真层级

仿真环境可以在不同层级上进行模拟，精度和成本各不相同：

```
仿真层级金字塔：

         ┌─────────┐
         │  完全真实 │  精度: 100%  成本: 最高
         │  (生产)  │  用途: 最终验证
        ┌┴─────────┴┐
        │  高保真模拟 │  精度: 80-90%  成本: 高
        │ (真实API) │  用途: 关键场景测试
       ┌┴──────────┴┐
       │  中保真模拟  │  精度: 60-80%  成本: 中
       │  (模拟API) │  用途: 常规对抗测试
      ┌┴───────────┴┐
      │  低保真模拟   │  精度: 40-60%  成本: 低
      │ (规则引擎)  │  用途: 快速迭代
     ┌┴────────────┴┐
     │  纯合成模拟    │  精度: 20-40%  成本: 最低
     │ (LLM模拟)   │  用途: 大规模探索
     
选择原则：
  大多数日常测试 → 低保真/中保真（快速+便宜）
  关键安全测试   → 高保真（准确但贵）
  最终上线前     → 完全真实（验证）
```

## 背景

### 为什么需要专门的 Agent 仿真环境

```
问题：为什么不能直接用生产环境做对抗测试？

1. 安全风险
   对抗测试可能破坏真实数据
   注入攻击可能影响真实用户
   越狱测试可能产生有害内容

2. 不可重复
   生产环境的状态时刻在变
   同一次攻击在不同时间结果不同
   无法做 A/B 对比测试

3. 覆盖不全
   生产环境中很难遇到所有边界情况
   某些攻击需要特定环境条件
   长尾场景在真实流量中很少出现

4. 效率问题
   在生产环境测试需要等待真实事件
   难以并行大量测试
   测试速度受限于真实时间
```

### 仿真工具的演进

```
2023 年：简单模拟
  • Mock API 响应（手动构造）
  • 简单的用户模拟（基于模板）
  • 工具：unittest.mock, responses

2024 年：半自动化仿真
  • LLM 驱动的用户模拟（让 LLM 扮演用户）
  • 工具沙箱（Docker 隔离）
  • 录制-回放（VCR-like）
  • 工具：ToolFuzz 前身

2025-2026 年：全栈仿真环境
  • 完整的 Agent 沙箱（LangGraph + Docker）
  • LLM 模拟用户 + LLM 模拟工具
  • 自动化场景生成
  • 对抗测试专用仿真
  • 工具：FLARE（多 Agent 覆盖引导模糊测试）
  •        Simulate-Agent（高保真用户模拟）
  •        ReliabilityBench（压力仿真环境）
```

## 核心矛盾

**仿真环境的"真实度"和"可控度"是一对根本矛盾。**

```
真实度 vs 可控度：

完全真实（生产环境）                   完全仿真（Mock环境）
  ├── 最真实                          ├── 最可控
  ├── 但不可控                         ├── 但不真实
  ├── 不可重复                         ├── 完全可重复
  ├── 有风险                           ├── 无风险
  └── 结果可信                         └── 结果可能不反映真实

折中方案：
  多级仿真 + 校准
  1. 先在低保真仿真中大量测试
  2. 找到问题后在中等保真仿真中复现
  3. 关键场景在高保真仿真中验证
  4. 最后在生产环境中用少量真实流量确认
  每一级都校准下一级的参数
```

## 代码实现

```python
import json
import time
import random
import asyncio
from typing import List, Dict, Optional, Callable, Any
from dataclasses import dataclass, field
from enum import Enum
from abc import ABC, abstractmethod


# ============================================================
# 环境组件基类
# ============================================================

class SimulationComponent(ABC):
    """仿真组件基类"""

    @abstractmethod
    async def reset(self):
        """重置组件状态"""
        pass

    @abstractmethod
    async def step(self, action: Any) -> Any:
        """执行一步模拟"""
        pass


# ============================================================
# 用户模拟器
# ============================================================

@dataclass
class SimulatedUser:
    """模拟用户配置"""
    name: str
    personality: str  # "patient" / "impatient" / "confused" / "expert"
    domain_knowledge: str  # "none" / "basic" / "expert"
    language: str = "zh"
    error_tolerance: float = 0.3  # 对 Agent 错误的容忍度


class UserSimulator(SimulationComponent):
    """
    用户模拟器 — 用 LLM 模拟真实用户行为。
    参考 2025 年研究：Impatient Users Confuse AI Agents
    """

    def __init__(self, user: SimulatedUser, llm_client=None):
        self.user = user
        self.llm = llm_client  # 用于驱动用户行为的 LLM
        self.conversation_history = []
        self.state = {
            "satisfaction": 1.0,
            "patience": 1.0,
            "current_goal": None,
        }

    async def reset(self):
        self.conversation_history = []
        self.state = {
            "satisfaction": 1.0,
            "patience": 1.0,
            "current_goal": None,
        }

    async def set_goal(self, goal: str):
        """设置用户的目标"""
        self.state["current_goal"] = goal

    async def generate_turn(self, agent_response: str = None) -> str:
        """
        根据 Agent 的回复生成用户的下一轮输入。
        模拟不同类型用户的行为模式。
        """
        if agent_response:
            self.conversation_history.append(
                ("agent", agent_response)
            )
            # 更新满意度
            self._update_satisfaction(agent_response)

        # 根据用户类型生成不同的行为
        if self.user.personality == "impatient":
            return await self._impatient_behavior()
        elif self.user.personality == "confused":
            return await self._confused_behavior()
        elif self.user.personality == "expert":
            return await self._expert_behavior()
        else:  # patient
            return await self._patient_behavior()

    def _update_satisfaction(self, agent_response: str):
        """模拟满意度变化"""
        # 简化模型：长时间回复降低满意度
        if len(agent_response) > 500:
            self.state["satisfaction"] -= 0.1
            self.state["patience"] -= 0.15

        # Agent 说"我不知道"降低满意度
        if "不知道" in agent_response or "无法" in agent_response:
            self.state["satisfaction"] -= 0.2

        # 边界检查
        self.state["satisfaction"] = max(0, min(1,
            self.state["satisfaction"]))
        self.state["patience"] = max(0, min(1,
            self.state["patience"]))

    async def _patient_behavior(self) -> str:
        """耐心用户：给出完整、明确的指令"""
        templates = [
            f"请帮我完成这个任务: {self.state['current_goal']}",
            f"我需要 {self.state['current_goal']}，请逐步解决",
            f"谢谢，请继续: {self.state['current_goal']}",
        ]
        return random.choice(templates)

    async def _impatient_behavior(self) -> str:
        """不耐心用户：催促、简略、情绪化"""
        if self.state["patience"] < 0.5:
            templates = [
                "快点！我还要等多久？",
                "简单点说，别绕弯子",
                "直接给结果，不要解释",
                f"还没好吗？{self.state['current_goal']}",
                "你就说能不能做",
            ]
        else:
            templates = [
                f"帮我{self.state['current_goal']}",
                "快点",
                "开始吧",
            ]
        return random.choice(templates)

    async def _confused_behavior(self) -> str:
        """困惑用户：表达不清、反复修改要求"""
        templates = [
            f"嗯...我想{self.state['current_goal']}，不对，"
            f"应该是另一个...算了你帮我看看吧",
            f"我有个问题，但是不太确定怎么说。"
            f"就是关于{self.state['current_goal']}的",
            f"刚才说的你忘了吧？我是说{self.state['current_goal']}，"
            f"不对，不是这个...",
        ]
        return random.choice(templates)

    async def _expert_behavior(self) -> str:
        """专家用户：使用专业术语、具体参数"""
        templates = [
            f"执行 {self.state['current_goal']}，"
            f"参数按默认配置",
            f"需要处理 {self.state['current_goal']}，"
            f"注意数据格式和边界情况",
            f"{self.state['current_goal']}，"
            f"完成后给我结构化报告",
        ]
        return random.choice(templates)


# ============================================================
# 工具模拟器
# ============================================================

@dataclass
class SimulatedToolConfig:
    """模拟工具的配置"""
    name: str
    description: str
    parameters: Dict
    response_template: str  # 响应模板
    latency_ms: tuple = (100, 500)  # (min, max)
    error_rate: float = 0.0  # 错误率
    error_types: List[str] = field(default_factory=lambda:
        ["timeout", "invalid_params", "server_error", "not_found"])


class ToolSimulator(SimulationComponent):
    """
    工具模拟器 — 模拟外部工具/API 的行为。
    支持延迟注入、错误注入、结果控制。
    """

    def __init__(self, configs: List[SimulatedToolConfig]):
        self.tools = {c.name: c for c in configs}
        self.call_log = []

    async def reset(self):
        self.call_log = []

    async def step(self, action: Dict) -> Dict:
        """
        执行工具调用模拟。
        action: {"tool": "tool_name", "params": {...}}
        """
        tool_name = action.get("tool")
        params = action.get("params", {})

        if tool_name not in self.tools:
            return {
                "success": False,
                "error": f"Unknown tool: {tool_name}"
            }

        config = self.tools[tool_name]
        self.call_log.append({
            "tool": tool_name,
            "params": params,
            "timestamp": time.time()
        })

        # 模拟延迟
        delay = random.uniform(*config.latency_ms) / 1000
        await asyncio.sleep(delay)

        # 模拟错误
        if random.random() < config.error_rate:
            error_type = random.choice(config.error_types)
            return {
                "success": False,
                "error": error_type,
                "message": f"Simulated {error_type} for {tool_name}"
            }

        # 生成响应
        response = self._generate_response(config, params)
        return {
            "success": True,
            "result": response
        }

    def _generate_response(self, config: SimulatedToolConfig,
                            params: Dict) -> str:
        """根据模板生成工具响应"""
        try:
            return config.response_template.format(**params)
        except KeyError:
            return config.response_template

    def get_call_stats(self) -> Dict:
        """获取工具调用统计"""
        if not self.call_log:
            return {}

        stats = {}
        for tool_name in self.tools:
            calls = [
                c for c in self.call_log
                if c["tool"] == tool_name
            ]
            stats[tool_name] = {
                "total_calls": len(calls),
                "last_call": calls[-1]["timestamp"]
                if calls else None
            }
        return stats


# ============================================================
# 仿真环境控制器
# ============================================================

@dataclass
class SimulationConfig:
    """仿真环境配置"""
    seed: int = 42
    time_scale: float = 1.0  # 时间加速倍数
    max_steps: int = 50
    record_all: bool = True


class SimulationEnvironment:
    """
    完整的 Agent 仿真测试环境。
    集成了用户模拟器、工具模拟器、场景管理和记录功能。
    """

    def __init__(self, config: SimulationConfig):
        self.config = config
        self.user_simulator: Optional[UserSimulator] = None
        self.tool_simulator: Optional[ToolSimulator] = None
        self.records = []
        self.current_step = 0
        random.seed(config.seed)

    def set_user_simulator(self, simulator: UserSimulator):
        """设置用户模拟器"""
        self.user_simulator = simulator

    def set_tool_simulator(self, simulator: ToolSimulator):
        """设置工具模拟器"""
        self.tool_simulator = simulator

    async def reset(self):
        """重置整个仿真环境"""
        self.current_step = 0
        self.records = []

        if self.user_simulator:
            await self.user_simulator.reset()
        if self.tool_simulator:
            await self.tool_simulator.reset()

        random.seed(self.config.seed)

    async def load_scenario(self, scenario: Dict):
        """
        加载测试场景。
        scenario: {
            "user": SimulatedUser,
            "goal": "用户目标",
            "tools": [ToolConfig, ...],
            "environment": {"error_rate": 0.1, ...}
        }
        """
        await self.reset()

        # 设置用户
        user_config = scenario.get("user", {})
        user = SimulatedUser(**user_config)
        self.user_simulator = UserSimulator(user)

        # 设置用户目标
        if "goal" in scenario:
            await self.user_simulator.set_goal(scenario["goal"])

        # 设置工具
        tool_configs = scenario.get("tools", [])
        self.tool_simulator = ToolSimulator(tool_configs)

    async def run_episode(self, agent_func: Callable) -> Dict:
        """
        运行一个完整的仿真回合。

        agent_func: 接收用户消息，返回 Agent 响应
        """
        episode_record = {
            "steps": [],
            "summary": {}
        }

        # 用户发送初始消息
        user_message = await self.user_simulator.generate_turn()
        episode_record["steps"].append({
            "step": 0,
            "type": "user",
            "content": user_message
        })

        while self.current_step < self.config.max_steps:
            # Agent 处理
            agent_response = await agent_func(user_message)

            episode_record["steps"].append({
                "step": self.current_step,
                "type": "agent",
                "content": agent_response
            })

            # 检查 Agent 是否完成了任务
            if self._is_task_complete(agent_response):
                episode_record["summary"] = {
                    "completed": True,
                    "steps_used": self.current_step + 1,
                    "result": agent_response
                }
                break

            # 用户下一轮输入
            user_message = await self.user_simulator.generate_turn(
                agent_response
            )

            episode_record["steps"].append({
                "step": self.current_step + 1,
                "type": "user",
                "content": user_message
            })

            self.current_step += 1

        # 记录结果
        if "summary" not in episode_record:
            episode_record["summary"] = {
                "completed": False,
                "steps_used": self.config.max_steps,
                "reason": "max_steps_reached"
            }

        if self.config.record_all:
            self.records.append(episode_record)

        return episode_record

    def _is_task_complete(self, response: str) -> bool:
        """判断任务是否完成（简化版）"""
        completion_signals = [
            "完成", "done", "DONE", "已完成",
            "任务完成", "全部完成"
        ]
        return any(s in response for s in completion_signals)

    def get_statistics(self) -> Dict:
        """获取仿真运行统计"""
        if not self.records:
            return {"no_records": True}

        completed = sum(
            1 for r in self.records
            if r["summary"].get("completed")
        )

        return {
            "total_episodes": len(self.records),
            "completed": completed,
            "completion_rate": completed / len(self.records),
            "avg_steps": sum(
                r["summary"].get("steps_used", 0)
                for r in self.records
            ) / len(self.records),
            "total_steps": sum(
                len(r["steps"]) for r in self.records
            ),
        }

    def export_traces(self, format: str = "json") -> str:
        """导出仿真轨迹"""
        if format == "json":
            return json.dumps(self.records, ensure_ascii=False,
                              indent=2)
        return str(self.records)


# ============================================================
# 场景库
# ============================================================

class AdversarialScenarioLibrary:
    """对抗测试场景库"""

    @staticmethod
    def get_standard_scenarios() -> List[Dict]:
        """获取标准对抗测试场景"""
        return [
            {
                "name": "基础用户-简单查询",
                "user": {
                    "name": "user1",
                    "personality": "patient",
                    "domain_knowledge": "basic"
                },
                "goal": "查询北京的天气",
                "tools": [
                    SimulatedToolConfig(
                        name="get_weather",
                        description="获取天气信息",
                        parameters={"city": "string"},
                        response_template="北京今天25°C，晴"
                    )
                ],
                "environment": {}
            },
            {
                "name": "不耐心用户-复杂分析",
                "user": {
                    "name": "user2",
                    "personality": "impatient",
                    "domain_knowledge": "basic"
                },
                "goal": "分析这份销售数据并给出建议",
                "tools": [
                    SimulatedToolConfig(
                        name="analyze_data",
                        description="分析数据",
                        parameters={"dataset": "string"},
                        response_template="数据已分析完成",
                        latency_ms=(2000, 5000)
                    ),
                    SimulatedToolConfig(
                        name="generate_report",
                        description="生成报告",
                        parameters={"format": "string"},
                        response_template="报告已生成",
                        error_rate=0.2
                    )
                ],
                "environment": {"error_rate": 0.1}
            },
            {
                "name": "困惑用户-高压力",
                "user": {
                    "name": "user3",
                    "personality": "confused",
                    "domain_knowledge": "none"
                },
                "goal": "帮我设置一个提醒",
                "tools": [
                    SimulatedToolConfig(
                        name="set_reminder",
                        description="设置提醒",
                        parameters={
                            "time": "string",
                            "content": "string"
                        },
                        response_template="提醒已设置: {content}"
                    ),
                    SimulatedToolConfig(
                        name="list_reminders",
                        description="列出所有提醒",
                        parameters={},
                        response_template="当前没有提醒"
                    )
                ],
                "environment": {}
            },
            {
                "name": "工具全面退化",
                "user": {
                    "name": "user4",
                    "personality": "patient",
                    "domain_knowledge": "expert"
                },
                "goal": "获取并分析服务器状态报告",
                "tools": [
                    SimulatedToolConfig(
                        name="get_server_status",
                        description="获取服务器状态",
                        parameters={"server_id": "string"},
                        response_template="服务器状态: 正常",
                        error_rate=0.4,
                        latency_ms=(3000, 8000)
                    ),
                    SimulatedToolConfig(
                        name="analyze_logs",
                        description="分析日志",
                        parameters={"time_range": "string"},
                        response_template="日志分析完成",
                        error_rate=0.3
                    )
                ],
                "environment": {"global_error_rate": 0.2}
            }
        ]
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 模拟多种用户行为模式 | 完美模拟真实用户的随机性和创造力 |
| 控制工具响应（延迟、错误、结果） | 预测所有工具的实际生产行为 |
| 创建可重复的测试场景 | 覆盖所有生产环境的状态组合 |
| 记录完整的交互轨迹 | 自动判断测试结果的正确性（依赖断言/人工） |
| 加速时间（快速运行多轮交互） | 模拟真实时间对 Agent 行为的影响 |
| 组合不同的压力和对抗条件 | 补偿仿真与真实环境之间的所有差异 |

## 工程优化方向

1. **仿真-生产差异校准**：定期运行校准测试——同一测试用例在仿真环境和生产环境中各运行一次，对比结果差异，调整仿真参数缩小偏差。

2. **场景自动生成**：使用 LLM 从生产日志自动提取和泛化测试场景。生产中的真实用户行为是最好的场景来源。

3. **渐进式保真度**：单个测试场景支持多级保真度配置。快速迭代用低配，关键验证用高配，同一个场景在不同阶段复用。

4. **录制-回放管道**：在生产环境中录制真实交互，转换为仿真场景。这样可以持续扩充场景库，同时保证场景的真实性。

5. **对抗场景进化**：定期在仿真环境中运行自动对抗测试，发现新的漏洞后自动添加到场景库——让仿真环境从静态变成自进化。

6. **分布式仿真集群**：将仿真环境部署为分布式服务，支持大规模并行测试。每天自动运行数万场景，生产回归检测。
