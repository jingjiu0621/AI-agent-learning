# 11.4.3 Stress Testing — 压力测试：高并发、长上下文与多工具

## 简单介绍

Agent 的压力测试与传统的系统压力测试有本质区别。传统压力测试关心的是"系统在负载下会不会挂"，而 Agent 压力测试关心三个层次的问题：

```
压力测试三层问题：

Layer 1 — 系统层（传统压力测试也关心）
  "高并发下 API 会不会超时？"
  "大量 Token 消耗会不会撑爆内存？"
  "多 Agent 同时运行会不会互相干扰？"

Layer 2 — LLM 层（Agent 特有的压力问题）
  "长上下文下推理质量会不会下降？"
  "大量工具选择会不会让 Agent 困惑？"
  "长时间运行会不会产生推理退化？"

Layer 3 — 行为层（Agent 特有的压力问题）
  "压力下 Agent 会不会做出在正常状态下不会做的危险操作？"
  "饥饿竞争下 Agent 会不会变得激进或短视？"
  "超时压力下 Agent 会不会产生幻觉来"完成"任务？"
```

```
                  Agent 压力测试三维度
                  ┌─────────────────────────────┐
                  │                             │
    传统压力测试 ←─→  系统资源压力                  │
      (QPS/CPU/内存)    │                          │
                       ├──────────────────────────┤
                       │  LLM 认知压力              │
                       │  (上下文长度/工具数量/      │
                       │   推理复杂度/运行时长)       │
                       ├──────────────────────────┤
                       │  行为压力                  │
                       │  (竞争/饥饿/超时/多任务)    │
                       │                          │
                       └──────────────────────────┘
```

## 基本原理

### Agent 压力测试的核心机制

Agent 在压力下的行为退化与人类在压力下的表现有惊人的相似之处：

```
人类在压力下的表现                   Agent 在压力下的表现
─────────────────────               ───────────────────────────
注意力范围缩小                        上下文利用率下降(只关注开头和结尾)
决策质量下降                          推理步骤简化，跳过关键验证
倾向于走捷径                          调用的工具变少，倾向于直接生成答案
更容易犯错                            Token 消耗压力下输出质量下降
情绪化决策                            产生幻觉来"完成"任务
```

### 压力源的分类

```
Agent 压力源分类：

┌─────────────────────────────────────────────────────────────┐
│                       Agent 压力源                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 输入压力                                                 │
│     ┌─────────────────────────────────────────────────┐     │
│     │ • 上下文长度：Agent 的推理窗口有多大              │     │
│     │ • 信息密度：输入中有多少需要处理的信息             │     │
│     │ • 噪声水平：无关信息的比例                        │     │
│     │ • 多模态：同时处理文本/图像/音频的复杂度           │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
│  2. 任务压力                                                 │
│     ┌─────────────────────────────────────────────────┐     │
│     │ • 工具数量：Agent 需要从多少工具中选择            │     │
│     │ • 步骤复杂度：完成任务需要多少步推理              │     │
│     │ • 并行度：同时跟踪多少个并行的子任务              │     │
│     │ • 时间约束：Agent 需要在多短时间内做出响应         │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
│  3. 环境压力                                                 │
│     ┌─────────────────────────────────────────────────┐     │
│     │ • 并发用户：同时服务多少个请求                    │     │
│     │ • 竞争资源：多个 Agent 竞争同一工具/知识库的权限   │     │
│     │ • 延迟冲击：工具 API 响应变慢                    │     │
│     │ • 错误率：工具调用失败的比例增加                   │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
│  4. 时间压力                                                 │
│     ┌─────────────────────────────────────────────────┐     │
│     │ • 响应截止时间：必须在 X 秒内返回结果              │     │
│     │ • Token 预算：总 Token 消耗上限                   │     │
│     │ • 步骤限制：最多允许多少轮思考-行动循环            │     │
│     └─────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

## 背景

### 从传统压力测试到 Agent 压力测试

```
传统压力测试工具和方法：
  • Apache Bench / wrk / k6（HTTP 负载测试）
  • JMeter / Gatling（多协议场景）
  • Chaos Monkey（故障注入）
  • 关注点：吞吐量、延迟、错误率、资源使用

Agent 压力测试的特殊性：
  • 不仅要测"系统会不会挂"，还要测"行为会不会变差"
  • 不仅要测响应速度，还要测响应质量
  • 不仅要测单次调用，还要测多轮交互的退化
  • 不仅要测正常负载，还要测认知负载（长上下文、复杂推理）
```

### ReliabilityBench（2025）的核心发现

ReliabilityBench（arXiv:2601.06112, 2025）是专门评估 Agent 在压力条件下可靠性的基准。关键发现：

```
测试设置：
  • 生产级 Agent 任务（代码生成、数据分析、客户支持）
  • 压力条件：工具延迟+30%、工具错误率+20%、并发请求

核心发现（Rank 1 — GPT-5 Agent）：
  ┌──────────────────────────────┬──────────────┐
  │ 指标                          │ 无压力  有压力 │
  ├──────────────────────────────┼──────────────┤
  │ 任务成功率                     │ 89.2%   63.5% │
  │ 平均推理步数                    │ 4.2     3.1  │
  │ 工具调用准确率                  │ 94.1%   78.3% │
  │ 幻觉率                         │ 3.2%    12.7% │
  │ 不必要的工具调用                │ 2.1%    8.9%  │
  │ API 调用数量（平均）            │ 5.3     7.8  │
  └──────────────────────────────┴──────────────┘

  解读：
  • 有压力环境下任务成功率下降 29%
  • Agent 在压力下倾向于"少思考多尝试"（推理更少、调用更多）
  • 压力下幻觉率飙升 4 倍
```

## 核心矛盾

**压力测试在"模拟真实压力"和"控制测试变量"之间存在根本矛盾。**

```
模拟 vs 控制的矛盾：

为了结果准确，需要：                   为了结果可复现，需要：
  ┌──────────────┐                     ┌──────────────┐
  │ 真实的负载模式 │                     │ 完全可控的环境 │
  │ 真实的数据分布 │                     │ 确定的压力参数 │
  │ 真实的环境波动 │                     │ 可重复的测试流 │
  │ 真实的延迟变化 │                     │ 一致的初始状态 │
  └──────────────┘                     └──────────────┘
         ↓                                     ↓
  结果更可信                            结果可复现
  但不可复现                            但可能不真实
  无法定位退化原因                      可能错过真实瓶颈

缓解方案：
  两阶段压力测试：
    Phase 1: 确定性压力（控制变量法）→ 定位问题
    Phase 2: 模拟真实压力（混沌工程）→ 验证鲁棒性
```

## 代码实现

```python
import time
import asyncio
import statistics
from typing import List, Dict, Optional, Callable, Any
from dataclasses import dataclass, field
from enum import Enum
from concurrent.futures import ThreadPoolExecutor
import random


# ============================================================
# 压力测试指标定义
# ============================================================

@dataclass
class StressMetrics:
    """压力测试指标"""
    # 系统指标
    qps: float = 0.0
    p50_latency: float = 0.0
    p95_latency: float = 0.0
    p99_latency: float = 0.0
    error_rate: float = 0.0
    token_usage: int = 0
    
    # Agent 行为指标
    task_success_rate: float = 0.0
    avg_reasoning_steps: float = 0.0
    tool_call_accuracy: float = 0.0
    hallucination_rate: float = 0.0
    unnecessary_tool_calls: float = 0.0
    
    # 资源指标
    memory_usage_mb: float = 0.0
    api_calls_count: int = 0
    context_utilization: float = 0.0


class StressLevel(Enum):
    """压力等级"""
    LIGHT = "light"       # 轻量: 低并发, 短上下文
    MODERATE = "moderate" # 中等: 正常生产负载
    HEAVY = "heavy"       # 重度: 峰值负载
    EXTREME = "extreme"   # 极限: 超出设计规格


# ============================================================
# 压力场景定义
# ============================================================

@dataclass
class StressScenario:
    """压力测试场景"""
    name: str
    description: str
    
    # 并发参数
    concurrency: int = 1
    total_requests: int = 10
    ramp_up_seconds: float = 0
    
    # 上下文压力
    context_length_tokens: int = 4000
    num_tools_available: int = 5
    task_complexity: str = "medium"  # easy / medium / hard
    
    # 环境压力
    tool_latency_ms: tuple = (100, 500)  # (min, max)
    tool_error_rate: float = 0.0
    timeout_seconds: float = 30
    
    # Token 预算
    max_tokens_per_call: int = 4096


# ============================================================
# 压力执行引擎
# ============================================================

class StressTestEngine:
    """
    Agent 压力测试引擎。
    支持并发、长上下文、多工具场景的压力测试。
    """

    def __init__(self, agent_func: Callable):
        """
        agent_func: Agent 调用函数
        """
        self.agent = agent_func
        self.results = []

    async def run_scenario(self, scenario: StressScenario) -> Dict:
        """运行一个压力测试场景"""
        print(f"\n=== 运行压力场景: {scenario.name} ===")
        print(f"并发: {scenario.concurrency}, "
              f"请求数: {scenario.total_requests}, "
              f"上下文: {scenario.context_length_tokens} tokens")

        # 构建压力任务
        tasks = []
        for i in range(scenario.total_requests):
            delay = self._calculate_delay(
                i, scenario.ramp_up_seconds
            )
            tasks.append(self._single_request(
                i, scenario, delay
            ))

        # 并发执行
        start_time = time.time()
        
        # 用信号量控制并发
        semaphore = asyncio.Semaphore(scenario.concurrency)
        
        async def throttled_request(task_id, scenario, delay):
            async with semaphore:
                if delay > 0:
                    await asyncio.sleep(delay)
                return await self._single_request(
                    task_id, scenario, 0
                )
        
        throttled_tasks = [
            throttled_request(i, scenario, delay)
            for i, delay in enumerate(
                self._calculate_all_delays(
                    scenario.total_requests,
                    scenario.ramp_up_seconds
                )
            )
        ]
        
        completed = await asyncio.gather(
            *throttled_tasks, return_exceptions=True
        )
        
        total_time = time.time() - start_time

        # 分析结果
        return self._analyze_results(
            completed, scenario, total_time
        )

    async def _single_request(self, task_id: int,
                               scenario: StressScenario,
                               delay: float) -> Dict:
        """单次压力请求"""
        if delay > 0:
            await asyncio.sleep(delay)

        # 构造压力输入
        input_data = self._build_stress_input(scenario)

        start_time = time.time()
        error = None
        response = None
        token_usage = 0

        # 注入环境压力（工具延迟、错误率）
        original_functions = self._inject_env_stress(scenario)

        try:
            # 执行 Agent
            response = await asyncio.wait_for(
                self.agent(input_data),
                timeout=scenario.timeout_seconds
            )
        except asyncio.TimeoutError:
            error = "timeout"
        except Exception as e:
            error = str(e)
        finally:
            # 恢复原始函数
            self._restore_functions(original_functions)

        latency = time.time() - start_time

        return {
            "task_id": task_id,
            "success": error is None,
            "latency": latency,
            "error": error,
            "response": response,
            "token_usage": token_usage
        }

    def _build_stress_input(self, scenario: StressScenario) -> str:
        """构建压力测试输入"""
        # 构造长上下文
        context_parts = []
        
        # 填充上下文到目标长度
        filler = "这是上下文中一段无关紧要的内容，" * 50
        context_length_chars = scenario.context_length_tokens * 4
        while len("".join(context_parts)) < context_length_chars:
            context_parts.append(filler)
        
        context = "".join(context_parts)[:context_length_chars]
        
        # 构造复杂任务
        tasks = {
            "easy": "请从以上文本中提取所有数字。",
            "medium": "请总结以上文本的主要内容，"
                      "并列出每个段落的关键点。",
            "hard": "请分析以上文本的逻辑结构，"
                    "识别其中的论证链条，"
                    "评估每个论点的支持证据强度，"
                    "并指出可能的逻辑谬误。"
        }
        
        return f"{context}\n\n任务: {tasks[scenario.task_complexity]}"

    def _inject_env_stress(self, scenario: StressScenario) -> Dict:
        """注入环境压力"""
        original = {}
        
        if scenario.tool_error_rate > 0:
            # 模拟工具错误
            original["tool_func"] = self._wrap_with_errors(
                scenario.tool_error_rate
            )
        
        if scenario.tool_latency_ms != (100, 500):
            # 模拟工具延迟
            original["latency_func"] = self._wrap_with_latency(
                scenario.tool_latency_ms
            )
        
        return original

    def _wrap_with_errors(self, error_rate: float):
        """包装函数使其按概率返回错误"""
        original = None  # 保存原始函数
        
        def wrapped(*args, **kwargs):
            if random.random() < error_rate:
                raise Exception(
                    f"Simulated tool error "
                    f"(rate: {error_rate})"
                )
            return original(*args, **kwargs) if original else None
        
        return {"original": original, "wrapped": wrapped}

    def _wrap_with_latency(self, latency_range: tuple):
        """包装函数使其有模拟延迟"""
        def wrapped(*args, **kwargs):
            delay = random.uniform(*latency_range) / 1000
            time.sleep(delay)
            return None
        return wrapped

    def _restore_functions(self, original: Dict):
        """恢复被替换的函数"""
        pass

    def _calculate_delay(self, index: int,
                          ramp_up: float) -> float:
        """计算请求的启动延迟（ramp-up）"""
        if ramp_up <= 0 or index == 0:
            return 0
        return (index / self.total_requests) * ramp_up

    def _calculate_all_delays(self, total: int,
                               ramp_up: float) -> List[float]:
        """计算所有请求的启动延迟"""
        if ramp_up <= 0:
            return [0.0] * total
        return [
            (i / max(total - 1, 1)) * ramp_up
            for i in range(total)
        ]

    def _analyze_results(self, results: List,
                          scenario: StressScenario,
                          total_time: float) -> Dict:
        """分析压力测试结果"""
        successful = [
            r for r in results
            if isinstance(r, dict) and r.get("success")
        ]
        failed = [
            r for r in results
            if isinstance(r, dict) and not r.get("success")
        ]
        exceptions = [
            r for r in results
            if not isinstance(r, dict)
        ]

        latencies = [
            r["latency"]
            for r in successful
            if isinstance(r, dict)
        ]

        # 计算指标
        total_requests = len(results)
        success_count = len(successful)
        error_count = len(failed) + len(exceptions)

        return {
            "scenario_name": scenario.name,
            "stress_level": scenario.concurrency,
            
            # 系统指标
            "total_requests": total_requests,
            "successful_requests": success_count,
            "failed_requests": error_count,
            "error_rate": error_count / total_requests
            if total_requests > 0 else 0,
            "actual_qps": success_count / total_time
            if total_time > 0 else 0,
            
            # 延迟指标
            "total_time_seconds": round(total_time, 2),
            "p50_latency": round(
                statistics.median(latencies), 3
            ) if latencies else 0,
            "p95_latency": round(
                sorted(latencies)[
                    int(len(latencies) * 0.95)
                ], 3
            ) if len(latencies) > 1 else 0,
            "p99_latency": round(
                sorted(latencies)[
                    int(len(latencies) * 0.99)
                ], 3
            ) if len(latencies) > 1 else 0,
            "max_latency": round(max(latencies), 3)
            if latencies else 0,
            
            # 错误详情
            "errors": [
                {"id": r.get("task_id"), "error": r.get("error")}
                for r in failed
            ],
            "exceptions_count": len(exceptions),
        }


# ============================================================
# 压力测试场景预设
# ============================================================

class StressScenarioPresets:
    """压力测试场景预设"""

    @staticmethod
    def get_standard_suite() -> List[StressScenario]:
        """标准压力测试套件"""
        return [
            # 并发压力
            StressScenario(
                name="并发-低",
                description="低并发测试",
                concurrency=5,
                total_requests=25,
            ),
            StressScenario(
                name="并发-中",
                description="中等并发测试",
                concurrency=20,
                total_requests=100,
            ),
            StressScenario(
                name="并发-高",
                description="高并发测试",
                concurrency=50,
                total_requests=250,
            ),
            
            # 上下文压力
            StressScenario(
                name="上下文-长",
                description="长上下文压力测试",
                context_length_tokens=32000,
                task_complexity="hard",
            ),
            StressScenario(
                name="上下文-极限",
                description="极限上下文压力测试",
                context_length_tokens=128000,
                task_complexity="hard",
            ),
            
            # 工具有限压力
            StressScenario(
                name="工具-多选项",
                description="大量工具选择压力",
                num_tools_available=30,
                task_complexity="medium",
            ),
            StressScenario(
                name="工具-海量",
                description="海量工具选择压力",
                num_tools_available=100,
                task_complexity="medium",
            ),
            
            # 环境退化压力
            StressScenario(
                name="退化-高延迟",
                description="工具高延迟环境",
                tool_latency_ms=(2000, 5000),
                timeout_seconds=60,
            ),
            StressScenario(
                name="退化-高错误率",
                description="工具高错误率环境",
                tool_error_rate=0.3,
            ),
            StressScenario(
                name="退化-双重打击",
                description="高延迟+高错误率",
                tool_latency_ms=(1000, 3000),
                tool_error_rate=0.2,
                timeout_seconds=60,
            ),
            
            # 组合压力
            StressScenario(
                name="组合-生产模拟",
                description="模拟生产环境峰值负载",
                concurrency=30,
                total_requests=150,
                context_length_tokens=16000,
                num_tools_available=20,
                tool_latency_ms=(200, 1000),
                tool_error_rate=0.05,
                task_complexity="medium",
            ),
        ]


# ============================================================
# 压力退化分析
# ============================================================

class StressDegradationAnalyzer:
    """
    压力退化分析器。
    对比无压力和压力下的 Agent 行为差异。
    """

    def __init__(self):
        self.baseline = None  # 无压力基线

    def set_baseline(self, metrics: Dict):
        """设置无压力基线"""
        self.baseline = metrics

    def analyze(self, stress_results: List[Dict]) -> Dict:
        """分析压力下的退化程度"""
        if not self.baseline:
            return {"warning": "无基线数据，无法进行退化分析"}

        analysis = {
            "degradation_found": False,
            "critical_degradations": [],
            "dimensions": {}
        }

        for result in stress_results:
            scenario = result["scenario_name"]
            
            # 计算关键指标的退化
            if self.baseline.get("error_rate", 0) > 0:
                error_rate_increase = (
                    result["error_rate"]
                    - self.baseline["error_rate"]
                )
            else:
                error_rate_increase = result["error_rate"]

            # 延迟退化
            if self.baseline.get("p50_latency", 0) > 0:
                latency_degradation = (
                    result["p50_latency"]
                    / self.baseline["p50_latency"]
                )
            else:
                latency_degradation = 1.0

            # 记录退化的维度
            degradations = {}
            if error_rate_increase > 0.1:
                degradations["error_rate"] = error_rate_increase
            if latency_degradation > 2.0:
                degradations["latency_p50"] = latency_degradation

            dimension = {
                "concurrency": result.get("stress_level"),
                "error_rate": result["error_rate"],
                "p50_latency": result["p50_latency"],
                "error_rate_increase": error_rate_increase,
                "latency_degradation_ratio": round(
                    latency_degradation, 2
                ),
                "degradations": degradations
            }

            analysis["dimensions"][scenario] = dimension

            if degradations:
                analysis["degradation_found"] = True
                if len(degradations) >= 2:
                    analysis["critical_degradations"].append(
                        scenario
                    )

        return analysis
```

## 压力测试的评估维度

```
Agent 压力测试评估矩阵

维度                 压力指标                      退化信号
────                 ──────                        ────────
任务成功率            请求完成比例                     < 80%
推理质量              步数/逻辑一致性                  步数减少 > 30%
工具选择              正确工具比例                     准确率下降 > 15%
输出质量              事实错误/幻觉率                  幻觉率 > 10%
响应延迟              端到端耗时                       P95 > 10s
Token 效率            完成任务所需 Token               增加 > 50%
错误恢复              错误后的行为                     放弃率 > 20%
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 测试 Agent 在预定压力下的行为退化 | 预测 Agent 在所有压力场景下的行为（无法穷举） |
| 定位导致退化的关键压力因素 | 完全隔离压力因素之间的交互效应 |
| 建立基线并与压力结果对比 | 在没有基础设施成本的情况下模拟真实负载 |
| 测试系统层的弹性（超时、并发） | 测试 LLM 的"认知极限"（无法量化"推理复杂度"） |
| 生成退化报告和优化方向 | 提供自动化修复方案 |
| 模拟工具延迟/错误环境 | 完全模拟真实用户的复杂行为模式 |

## 工程优化方向

1. **压力-质量-成本三维权衡**：压力测试不只是看"会不会挂"，更要看"质量如何退化"和"成本如何增长"。建立三维模型找到最优操作点。

2. **自适应限流与降级**：根据当前系统压力动态调整 Agent 行为——高负载时简化推理、减少工具调用、降低输出质量要求以维持响应。

3. **认知负载监控**：实时跟踪 Agent 当前推理的复杂度（上下文利用率、推理步数、工具调用密度），在负载过高时触发简化策略。

4. **压力测试即服务**：将压力测试作为 CI/CD 的常态化环节，每次 Agent 更新自动运行标准压力套件，对比基线检测退化。

5. **混沌工程 for Agents**：在生产环境（或类生产环境）中随机注入延迟、错误、竞争，观察 Agent 是否能在压力下维持合理行为。从"测压力"到"练抗压"。

6. **压力日志关联分析**：将压力测试指标与 Agent 行为日志关联，发现"压力→行为退化"的因果链，针对性地优化 Agent 在特定压力下的行为。
