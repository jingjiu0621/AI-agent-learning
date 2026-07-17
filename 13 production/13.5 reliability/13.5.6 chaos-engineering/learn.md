# Chaos Engineering —— LLM Agent 系统的混沌工程

> **核心挑战**：Agent 系统的故障模式比传统系统更复杂、更多样，且不确定性使得"测试环境一切正常"不等于"生产环境也会正常"。传统的单元测试和集成测试只能覆盖已知的故障路径，但 Agent 系统在真实环境中会遇到大量无法预测的故障组合——LLM 输出突变、工具 API 间歇性故障、上下文窗口被异常填满、多 Agent 之间的死锁。混沌工程的核心思想是**主动注入故障**，在受控环境中验证系统的弹性设计是否真正有效，并在生产环境发现测试覆盖不到的薄弱环节。

---

## 1. 基本原理

### 1.1 什么是混沌工程

混沌工程不是"搞破坏"，而是**系统化地验证系统弹性**的工程实践：

```
传统测试 vs 混沌工程:

  传统测试:
    问: "系统在正常情况下能否正常工作?"
    方法: 预设输入 → 验证预设输出
    覆盖: 已知场景, 确定性的
    局限: 只能验证"知道要测试什么"

  混沌工程:
    问: "系统在异常情况下会如何表现?"
    方法: 主动注入故障 → 观察系统行为
    覆盖: 未知场景, 概率性的
    价值: 发现"没想到要测试什么"的问题

  混沌工程 ≠ 随机破坏
  混沌工程 = 有假设的实验 + 受控的变量 + 可观测的结果
```

### 1.2 为什么 Agent 需要混沌工程

Agent 系统的"故障空间"比传统系统大得多：

```
传统系统的故障空间:
  ┌─ 服务宕机 (进程崩溃)
  ├─ 网络分区 (延迟 / 丢包)
  ├─ 资源耗尽 (CPU / 内存 / 磁盘)
  └─ 超时 (慢响应)

Agent 系统的额外故障空间:
  ┌─ LLM 输出偏差 (语义错误, 非确定性)
  ├─ LLM 输出格式错误 (JSON 解析失败)
  ├─ LLM 拒绝回答 (安全策略触发)
  ├─ 工具选择错误 (LLM 选错了工具)
  ├─ Agent 循环 (Thought → Action 死循环)
  ├─ 上下文溢出 (Token 超限)
  ├─ 记忆污染 (错误记忆进入长期存储)
  ├─ 幻觉传播 (一个 Agent 的幻觉传递给另一个 Agent)
  ├─ 成本爆炸 (某步 Token 消耗突然激增)
  └─ 级联质量降级 (多步偏差累积)

关键点: LLM 输出的非确定性使"测试覆盖"成为不可能。
混沌工程是唯一能系统性发现 Agent 薄弱环节的方法。
```

### 1.3 混沌工程成熟度模型

```
Level 0: 无混沌
  └── 只在正常条件下测试, 从未验证过故障场景

Level 1: 人工混沌
  └── 手动注入故障 (关掉数据库, 模拟网络延迟)
  └── 不定期, 没有系统性

Level 2: 自动化混沌
  └── 自动化故障注入管道
  └── 在 Staging 环境定期运行
  └── 有基本的监控验证

Level 3: 生产混沌
  └── 在生产环境中注入受控故障
  └── 有完善的爆炸半径控制
  └── 自动回滚和终止机制

Level 4: 持续混沌
  └── 混沌实验作为 CI/CD 的一部分持续运行
  └── 自适应实验 (基于生产数据自动生成实验)
  └── 自动根因分析和修复建议
```

---

## 2. Agent 特有的混沌实验设计

### 2.1 故障注入类型

Agent 系统的混沌实验需要覆盖六个维度的故障：

```
1. 基础设施层
   ┌─ LLM API 延迟 (500ms → 5000ms)
   ├─ LLM API 错误 (429 / 500 / 503)
   ├─ LLM API 断连 (TCP 连接中断)
   ├─ 工具 API 延迟 / 错误
   ├─ 数据库延迟 / 不可用
   └─ 缓存服务不可用

2. LLM 输出层 (Agent 特有!)
   ┌─ 空输出 (LLM 返回空字符串)
   ├─ 格式错误 (无效 JSON, 错误 Schema)
   ├─ 拒绝回答 ("I cannot answer this")
   ├─ 幻觉输出 (事实错误但看起来合理)
   ├─ 语言混用 (输出与输入语言不一致)
   ├─ 过长的输出 (Token 激增)
   └─ 过短的输出 (回答不完整)

3. Agent 行为层 (Agent 特有!)
   ┌─ 工具选择错误 (选错/不选)
   ├─ 参数提取错误 (参数缺失/错误)
   ├─ 重复相同 Action (循环)
   ├─ 忽略上下文 (忘记之前的对话)
   └─ 违反指令 (不遵守 System Prompt)

4. 状态层
   ┌─ 状态损坏 (部分字段丢失)
   ├─ 状态不一致 (多 Agent 状态不同步)
   ├─ 记忆检索为空 (查不到相关内容)
   └─ 记忆检索到错误内容 (不相关的记忆)

5. 编排层
   ┌─ 多 Agent 通信超时
   ├─ 任务分配死锁
   ├─ 协调者故障
   └─ Agent 间消息丢失

6. 资源层
   ┌─ Token 配额耗尽
   ├─ 上下文窗口溢出
   ├─ 连接池耗尽
   └─ 并发超限
```

### 2.2 实验框架

```python
import random
import time
import asyncio
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional, Callable

class ExperimentType(Enum):
    LATENCY = "latency"                    # 注入延迟
    ERROR = "error"                        # 注入错误
    NULL_RESPONSE = "null_response"        # 空响应 (Agent 特有)
    FORMAT_ERROR = "format_error"          # 格式错误 (Agent 特有)
    TOOL_MISSELECTION = "tool_misselection"  # 工具误选 (Agent 特有)
    LOOP = "loop"                          # 循环注入 (Agent 特有)
    CONTEXT_OVERFLOW = "context_overflow"  # 上下文溢出
    MEMORY_CORRUPTION = "memory_corruption"  # 记忆污染
    RESOURCE_EXHAUSTION = "resource_exhaustion" # 资源耗尽

@dataclass
class ExperimentConfig:
    """混沌实验配置"""
    experiment_type: ExperimentType
    probability: float = 0.1       # 注入概率
    duration_seconds: int = 60     # 实验持续时间
    target_service: str = "llm"    # 目标服务
    intensity: float = 0.5         # 强度 (0.0-1.0)

class ChaosExperiment:
    """
    混沌实验基类。

    定义实验的假设、注入方法、验证方法。
    """

    def __init__(self, name: str, config: ExperimentConfig):
        self.name = name
        self.config = config
        self.start_time: Optional[float] = None
        self.metrics_before: dict = {}
        self.metrics_after: dict = {}
        self.hypothesis: str = ""
        self.verified: Optional[bool] = None

    def set_hypothesis(self, hypothesis: str):
        """设定实验假设"""
        self.hypothesis = hypothesis

    async def run(self, environment: "ChaosEnvironment"):
        """执行实验"""
        self.start_time = time.time()

        # 1. 收集实验前指标
        self.metrics_before = await environment.collect_metrics()

        # 2. 注入故障
        await self.inject(environment)

        # 3. 等待故障生效
        await asyncio.sleep(5)

        # 4. 观察系统行为
        self.metrics_after = await environment.collect_metrics()

        # 5. 验证假设
        self.verified = self.verify_hypothesis()

        # 6. 回滚
        await self.rollback(environment)

        return self.report()

    async def inject(self, environment: "ChaosEnvironment"):
        """注入故障 - 子类重写"""
        raise NotImplementedError

    async def rollback(self, environment: "ChaosEnvironment"):
        """回滚故障注入"""
        pass

    def verify_hypothesis(self) -> bool:
        """验证实验假设"""
        return True

    def report(self) -> dict:
        return {
            "name": self.name,
            "type": self.config.experiment_type.value,
            "hypothesis": self.hypothesis,
            "verified": self.verified,
            "duration": time.time() - self.start_time,
            "metrics_before": self.metrics_before,
            "metrics_after": self.metrics_after,
        }
```

### 2.3 Agent 专用混沌实验

#### 实验 1: LLM 格式错误注入

```python
class LLMFormatErrorExperiment(ChaosExperiment):
    """
    实验: LLM 返回格式错误的 JSON 时, Agent 能否正确处理?

    假设:
    - Agent 应能检测到 JSON 解析失败
    - Agent 应重试并要求 LLM 生成正确格式
    - 反复失败 3 次后, Agent 应优雅降级而非崩溃
    """

    def __init__(self, target_model: str = "gpt-4"):
        super().__init__(
            "LLM 格式错误注入",
            ExperimentConfig(
                experiment_type=ExperimentType.FORMAT_ERROR,
                probability=0.5,
                target_service=target_model,
            ),
        )
        self.set_hypothesis(
            "当 LLM 返回格式错误的 JSON 时, Agent 会检测到错误并重试, "
            "重试 3 次失败后降级, 不会崩溃"
        )
        self.original_complete = None
        self.injection_active = False

    async def inject(self, environment: "ChaosEnvironment"):
        """注入格式错误故障"""
        llm_client = environment.get_llm_client(self.config.target_service)

        # 保存原始方法
        self.original_complete = llm_client.complete

        # 替换为故障版本
        self.injection_active = True

        async def faulty_complete(messages, **kwargs):
            if self.injection_active and random.random() < self.config.probability:
                # 返回格式错误的 JSON
                return {
                    "content": 'Here is the result: {"key": "value", }',  # 无效 JSON
                    "tool_calls": [
                        {
                            "type": "function",
                            "function": {
                                "name": "search",
                                "arguments": "{invalid json",  # JSON 解析失败
                            }
                        }
                    ]
                }
            return await self.original_complete(messages, **kwargs)

        llm_client.complete = faulty_complete

    async def rollback(self, environment: "ChaosEnvironment"):
        """恢复原始方法"""
        self.injection_active = False
        llm_client = environment.get_llm_client(self.config.target_service)
        if self.original_complete:
            llm_client.complete = self.original_complete

    def verify_hypothesis(self) -> bool:
        """验证假设"""
        after = self.metrics_after
        # 检查是否出现了预期的行为模式
        format_errors = after.get("format_errors", 0)
        retries = after.get("retries", 0)
        degradations = after.get("degradations", 0)
        crashes = after.get("crashes", 0)

        # 假设验证: 有错误但无崩溃
        return format_errors > 0 and crashes == 0
```

#### 实验 2: Agent 循环检测实验

```python
class AgentLoopExperiment(ChaosExperiment):
    """
    实验: 当 LLM 反复选择相同的工具/参数时, Agent 能否检测到循环?

    假设:
    - Agent 应有重复 Action 检测机制
    - 检测到循环后应跳出循环并尝试其他路径
    - 应防止无限循环消耗 Token
    """

    def __init__(self):
        super().__init__(
            "Agent 循环注入",
            ExperimentConfig(
                experiment_type=ExperimentType.LOOP,
                probability=0.3,
                duration_seconds=30,
            ),
        )
        self.set_hypothesis(
            "当 LLM 反复生成相同的工具调用时, "
            "Agent 的循环检测机制会在 5 步内识别并终止循环"
        )
        self.repeated_action = {
            "name": "search",
            "arguments": '{"query": "weather in Tokyo"}',
        }

    async def inject(self, environment: "ChaosEnvironment"):
        """注入循环行为: 让 LLM 每次都返回相同工具调用"""
        llm_client = environment.get_llm_client()
        self.original_complete = llm_client.complete

        self.loop_count = 0

        async def looping_complete(messages, **kwargs):
            self.loop_count += 1
            if self.loop_count <= 8 and random.random() < self.config.probability:
                # 返回相同的工具调用
                return {
                    "content": "I need to search for weather information.",
                    "tool_calls": [
                        {
                            "type": "function",
                            "function": self.repeated_action,
                        }
                    ]
                }
            return await self.original_complete(messages, **kwargs)

        llm_client.complete = looping_complete

    async def rollback(self, environment: "ChaosEnvironment"):
        llm_client = environment.get_llm_client()
        if self.original_complete:
            llm_client.complete = self.original_complete

    def verify_hypothesis(self) -> bool:
        after = self.metrics_after
        steps = after.get("agent_steps", 0)
        loop_detected = after.get("loop_detections", 0)
        return loop_detected >= 1 and steps <= 10
```

#### 实验 3: 工具超时组合实验

```python
class ToolTimeoutCascadeExperiment(ChaosExperiment):
    """
    实验: 多个工具同时超时时, Agent 能否优雅降级?

    假设:
    - Agent 应跳过超时的工具而非阻塞等待
    - 部分工具超时不应对其他工具造成影响
    - 所有工具都失败时, Agent 应返回降级响应
    """

    def __init__(self, tool_names: list[str]):
        super().__init__(
            "工具超时级联实验",
            ExperimentConfig(
                experiment_type=ExperimentType.LATENCY,
                probability=1.0,  # 所有指定工具都注入故障
                duration_seconds=30,
                intensity=0.8,     # 高延迟: 5-15s
            ),
        )
        self.tool_names = tool_names
        self.set_hypothesis(
            f"当工具 {tool_names} 全部超时时, Agent 应跳过这些工具 "
            "并基于已有信息继续执行"
        )
        self.original_executors = {}

    async def inject(self, environment: "ChaosEnvironment"):
        """给指定工具注入高延迟"""
        for tool_name in self.tool_names:
            executor = environment.get_tool_executor(tool_name)
            self.original_executors[tool_name] = executor.execute

            async def slow_execute(*args, tool=tool_name, **kwargs):
                delay = random.uniform(8.0, 15.0)  # 8-15s 延迟
                await asyncio.sleep(delay)
                raise asyncio.TimeoutError(f"Tool '{tool}' timed out")

            executor.execute = slow_execute

    async def rollback(self, environment: "ChaosEnvironment"):
        for tool_name, original_fn in self.original_executors.items():
            executor = environment.get_tool_executor(tool_name)
            executor.execute = original_fn

    def verify_hypothesis(self) -> bool:
        after = self.metrics_after
        timed_out_tools = after.get("timed_out_tools", 0)
        completed = after.get("completed", False)
        degraded = after.get("degradation_used", False)

        return timed_out_tools >= len(self.tool_names) and (completed or degraded)
```

---

## 3. 混沌工程平台

### 3.1 实验编排器

```python
class ChaosOrchestrator:
    """
    混沌实验编排器。

    管理实验的生命周期:
    - 实验调度
    - 爆炸半径控制
    - 自动回滚
    - 结果分析
    """

    def __init__(self, environment: "ChaosEnvironment"):
        self.environment = environment
        self.active_experiments: list[ChaosExperiment] = []
        self.completed_experiments: list[dict] = []
        self.is_running = False
        self.explosion_radius = {
            "max_affected_tenants": 1,     # 同时最多影响 1 个 Tenant
            "max_affected_requests": 0.1,  # 最多影响 10% 的请求
            "max_duration_seconds": 120,   # 实验最长 2 分钟
            "blocked_hours": [9, 17],      # 工作时间自动暂停
        }

    async def run_experiment(self, experiment: ChaosExperiment) -> dict:
        """运行单个实验"""
        # 安全检查
        self._validate_experiment(experiment)

        # 记录实验开始
        self.active_experiments.append(experiment)
        self.is_running = True

        try:
            # 注入探针 (监控实验影响)
            monitor = ExperimentMonitor(self.environment)
            await monitor.start()

            # 执行实验
            result = await experiment.run(self.environment)

            # 分析结果
            analysis = self._analyze_result(result)

            self.completed_experiments.append({
                "experiment": experiment.name,
                "result": result,
                "analysis": analysis,
                "timestamp": time.time(),
            })

            return {"result": result, "analysis": analysis}

        except Exception as e:
            # 实验异常: 紧急回滚
            await self._emergency_rollback(experiment)
            raise

        finally:
            self.active_experiments.remove(experiment)
            self.is_running = False
            await monitor.stop()

    def _validate_experiment(self, experiment: ChaosExperiment):
        """验证实验是否在安全范围内"""
        # 检查时间窗口
        current_hour = time.localtime().tm_hour
        blocked_start, blocked_end = self.explosion_radius["blocked_hours"]
        if blocked_start <= current_hour < blocked_end:
            raise ChaosExperimentBlocked(
                "Experiments blocked during working hours"
            )

    def _analyze_result(self, result: dict) -> dict:
        """分析实验结果"""
        return {
            "hypothesis_verified": result.get("verified"),
            "weaknesses_found": self._extract_weaknesses(result),
            "recommendations": self._generate_recommendations(result),
        }

    def _extract_weaknesses(self, result: dict) -> list[str]:
        """提取发现的薄弱环节"""
        weaknesses = []
        after = result.get("metrics_after", {})

        if after.get("crashes", 0) > 0:
            weaknesses.append("System crashed under injected fault")
        if after.get("error_rate", 0) > 0.3:
            weaknesses.append("High error rate under fault conditions")
        if after.get("recovery_time", 0) > 30:
            weaknesses.append("Slow recovery from faults (>30s)")

        return weaknesses

    def _generate_recommendations(self, result: dict) -> list[str]:
        """生成改进建议"""
        recs = []
        weaknesses = self._extract_weaknesses(result)

        if "crashes" in str(weaknesses):
            recs.append("Add try-catch around LLM calls")
        if "recovery_time" in str(weaknesses):
            recs.append("Improve circuit breaker recovery time")

        return recs

    async def _emergency_rollback(self, experiment: ChaosExperiment):
        """紧急回滚"""
        await experiment.rollback(self.environment)
        # 通知运维
        print(f"EMERGENCY ROLLBACK: {experiment.name}")


class ChaosExperimentBlocked(Exception):
    pass


class ExperimentMonitor:
    """实验监控器: 实时跟踪实验影响"""

    def __init__(self, environment: "ChaosEnvironment"):
        self.environment = environment
        self.baseline_metrics = {}
        self.thresholds = {
            "error_rate_increase": 0.2,     # 错误率增长 < 20%
            "latency_increase": 0.5,        # 延迟增长 < 50%
            "success_rate_drop": 0.1,       # 成功率下降 < 10%
        }

    async def start(self):
        self.baseline_metrics = await self.environment.collect_metrics()

    async def stop(self):
        pass

    def check_impact(self, current_metrics: dict) -> bool:
        """检查实验影响是否在安全范围内"""
        for key, threshold in self.thresholds.items():
            baseline = self.baseline_metrics.get(key, 0)
            current = current_metrics.get(key, 0)
            if current > baseline * (1 + threshold):
                return False  # 超过安全阈值
        return True
```

### 3.2 实验场景库

预定义的 Agent 混沌实验场景：

```python
class ExperimentLibrary:
    """预定义的混沌实验场景库"""

    @staticmethod
    def all_tools_timeout() -> list[ChaosExperiment]:
        """场景: 所有工具超时"""
        return [
            ToolTimeoutCascadeExperiment(["search", "read_file", "calculate"]),
        ]

    @staticmethod
    def llm_degradation() -> list[ChaosExperiment]:
        """场景: LLM 质量降级"""
        return [
            LLMFormatErrorExperiment("gpt-4"),
        ]

    @staticmethod
    def agent_loop_scenario() -> list[ChaosExperiment]:
        """场景: Agent 循环"""
        return [
            AgentLoopExperiment(),
        ]

    @staticmethod
    def full_suite() -> list[ChaosExperiment]:
        """完整的实验套件"""
        return [
            LLMFormatErrorExperiment("gpt-4"),
            AgentLoopExperiment(),
            ToolTimeoutCascadeExperiment(["search"]),
            ToolTimeoutCascadeExperiment(["read_file", "write_file"]),
        ]

    @staticmethod
    def smoke_test() -> list[ChaosExperiment]:
        """冒烟测试 (快速)"""
        return [
            LLMFormatErrorExperiment("gpt-4o-mini"),
        ]
```

---

## 4. 安全与爆炸半径控制

混沌工程的核心安全原则：**实验造成的最大伤害必须在可控范围内**。

### 4.1 爆炸半径控制策略

```
Agent 系统的爆炸半径控制:

  租户隔离:
    └── 实验只影响测试 Tenant, 不影响生产 Tenant

  流量比例:
    └── 最多影响 X% 的请求 (从 1% 开始)

  时间窗口:
    └── 只在低峰期执行 (如凌晨 2-4 点)

  功能范围:
    └── 实验只针对非核心功能
    └── 安全校验、支付、数据写入永不参与实验

  自动终止条件:
    └── 错误率超过阈值 → 自动终止
    └── 延迟超过阈值 → 自动终止
    └── 成本超过阈值 → 自动终止
    └── 人工审批 → 关键实验需人工确认

  最小化影响:
    └── 先模拟故障 (不注入真实故障)
    └── 再沙箱故障 (隔离环境)
    └── 最后生产故障 (低流量)
```

### 4.2 Agent 特有的安全考量

```
Agent 混沌工程的额外安全规则:

  1. 不注入会导致数据损坏的故障
     模拟读操作的故障, 不模拟写操作的故障

  2. 不注入会污染长期记忆的故障
     实验中的错误信息不应写入持久化记忆

  3. 不注入影响用户信任的故障
     实验中的 Agent 错误行为不应被真实用户看到

  4. 所有实验都有 Kill Switch
     一键终止所有正在运行的实验

  5. 实验后有自动恢复验证
     确认系统回到正常状态, 没有残留影响
```

---

## 5. 能力边界

### 5.1 混沌工程的局限

```
混沌工程做不到:
  ┌─ 覆盖所有故障场景 (故障空间无限, 实验数量有限)
  ├─ 精确预测每次故障的影响 (LLM 非确定性无法预测)
  ├─ 代替单元测试和集成测试 (混沌工程是补充, 不是替代)
  └─ 保证 100% 的可靠性 (混沌工程只是"提高信心", 不是"保证正确")
```

### 5.2 实施建议

```
实施路线图:

  Phase 1: 基础 (第 1-2 周)
    └── 建立混沌实验框架
    └── 实现最基础的故障注入 (LLM 超时)
    └── 在 Staging 环境运行

  Phase 2: 扩展 (第 3-4 周)
    └── 增加故障类型 (格式错误、循环、工具超时)
    └── 建立实验场景库
    └── 自动化实验执行

  Phase 3: 生产化 (第 5-8 周)
    └── 生产环境实验 (低流量, 1% 请求)
    └── 爆炸半径控制
    └── 自动终止和回滚

  Phase 4: 持续化 (第 9 周+)
    └── 实验作为 CI/CD 环节
    └── 自适应实验生成
    └── 自动根因分析
```

---

## 6. 实践检查清单

```
[ ] 1. 定义了明确的实验假设 (不是随机破坏)
[ ] 2. 有完善的爆炸半径控制 (隔离 + 限流 + 终止)
[ ] 3. 实验覆盖 Agent 特有的故障 (格式错误/循环/语义偏差)
[ ] 4. 有基线指标用于对比 (实验前 vs 实验后)
[ ] 5. 实验不涉及写操作和数据持久化
[ ] 6. 有自动终止条件 (错误率/延迟/成本阈值)
[ ] 7. 实验后有自动恢复验证
[ ] 8. 实验结果有记录和分析
[ ] 9. 缺陷修复后验证实验 (确认修复有效)
[ ] 10. 实验不会影响真实用户体验
```

---

> **总结**：混沌工程是验证 Agent 系统可靠性的终极手段。Agent 系统的非确定性和多样化的故障模式，使得传统测试无法覆盖所有场景——混沌工程通过主动注入故障来发现薄弱环节。关键在于：(1) Agent 特有的故障（格式错误、循环、语义偏差）必须纳入实验范围；(2) 爆炸半径控制比传统系统更严格（避免污染记忆和用户信任）；(3) 每个实验必须有明确的假设和验证方法；(4) 从 Staging 环境开始，逐步过渡到生产环境。最终目标是建立一套持续运行、自动发现薄弱环节并驱动改进的混沌工程体系。
