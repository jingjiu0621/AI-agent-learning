# Timeout Control —— LLM Agent 系统的超时控制

> **核心挑战**：Agent 系统的执行时间高度不可预测——一次 LLM 调用可能耗时 500ms 到 30s 不等，一个 Agent 任务可能 3 步完成也可能陷入 30 步的循环。没有严格的超时控制，一个失控的 Agent 可以持续执行数分钟甚至数小时，不仅消耗大量 Token 成本，还会阻塞系统资源、延迟其他请求。超时控制的核心任务是在"给 Agent 足够的执行时间"和"及时止损"之间找到精确的平衡。

---

## 1. 基本原理

### 1.1 为什么 Agent 需要超时控制

传统服务与 Agent 系统在超时问题上的本质差异：

```
传统 Web 服务的超时:
  ┌─ 单次 HTTP 请求: 30s 超时 → 超时后返回 504
  ├─ 数据库查询: 5s 超时 → 超时后报错
  └─ 大多数请求在 100ms 内完成, 超时是罕见的边界情况

Agent 系统的超时:
  ┌─ 单次 LLM 调用: 1-30s (高度可变)
  ├─ 单次工具调用: 100ms-10s (取决于外部 API)
  ├─ 一次 Thought: 500ms-5s (取决于模型和 Prompt 复杂度)
  ├─ 一次 Agent 循环: 2s-60s (取决于步数和工具)
  ├─ 整个 Agent 任务: 5s-300s+ (取决于任务复杂度)
  └─ 每次调用都有成本, 超时 = Token 损失 + 延迟惩罚
```

### 1.2 Agent 超时的层次结构

Agent 的超时不是单一维度，而是分层的体系：

```
超时层次:

  请求整体超时 ────────────── 整个 Agent 任务的最长允许时间
      │                          示例: "30s 内必须返回结果"
      │
      ├── Agent 循环超时 ──────── Agent 主循环的总执行时间
      │     │                      示例: "最多执行 10 步或 25s"
      │     │
      │     ├── 单步超时 ──────── 单次 Thought-Action-Observation 的时间
      │     │     │                 示例: 单步最多 10s
      │     │     │
      │     │     ├── LLM 调用超时 ── 单次 LLM API 调用
      │     │     │                     示例: 连接 5s + 读取 30s
      │     │     │
      │     │     └── 工具调用超时 ── 单次工具执行
      │     │                          示例: HTTP 5s + 处理 3s
      │     │
      │     └── 思考超时 ──────── LLM 生成 Thought 的时间
      │                            示例: stream 首 token 5s
      │
      └── 排队超时 ──────────── 请求在队列中的等待时间
                                   示例: 队列等待最长 10s

  层级原则:
  内层超时 < 外层超时 (每层预留缓冲)
  单步超时 < 循环超时 < 整体超时
```

---

## 2. 核心技术与实现

### 2.1 分层超时管理器

```python
import asyncio
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional, Callable

class TimeoutType(Enum):
    LLM_CALL = "llm_call"                    # LLM API 调用
    TOOL_CALL = "tool_call"                  # 工具调用
    THOUGHT = "thought"                      # Agent 思考
    SINGLE_STEP = "single_step"              # 单步总时间
    AGENT_LOOP = "agent_loop"                # Agent 循环总时间
    TOTAL_REQUEST = "total_request"          # 请求整体
    QUEUE_WAIT = "queue_wait"                # 队列等待

@dataclass
class TimeoutConfig:
    """超时配置"""
    timeouts: dict[TimeoutType, float] = field(default_factory=lambda: {
        TimeoutType.LLM_CALL: 30.0,
        TimeoutType.TOOL_CALL: 10.0,
        TimeoutType.THOUGHT: 15.0,
        TimeoutType.SINGLE_STEP: 45.0,
        TimeoutType.AGENT_LOOP: 120.0,
        TimeoutType.TOTAL_REQUEST: 180.0,
        TimeoutType.QUEUE_WAIT: 10.0,
    })

    def validate(self):
        """验证超时层级一致性"""
        assert self.timeouts[TimeoutType.SINGLE_STEP] <= self.timeouts[TimeoutType.AGENT_LOOP], \
            "Single step timeout must be <= loop timeout"
        assert self.timeouts[TimeoutType.LLM_CALL] < self.timeouts[TimeoutType.SINGLE_STEP], \
            "LLM call timeout must be < single step timeout"
        assert self.timeouts[TimeoutType.TOOL_CALL] < self.timeouts[TimeoutType.SINGLE_STEP], \
            "Tool call timeout must be < single step timeout"
        assert self.timeouts[TimeoutType.AGENT_LOOP] <= self.timeouts[TimeoutType.TOTAL_REQUEST], \
            "Agent loop timeout must be <= total request timeout"


class TimeoutManager:
    """
    分层超时管理器。

    管理 Agent 执行过程中各个级别的超时,
    并使用 deadline 传递 (而非嵌套超时) 来实现精确控制。
    """

    def __init__(self, config: TimeoutConfig):
        self.config = config
        self.metrics = {
            "timeouts_triggered": 0,
            "timeouts_by_type": {},
            "total_execution_time": 0.0,
        }

    def create_request_context(self) -> "RequestContext":
        """创建请求级超时上下文"""
        return RequestContext(self)

    def get_remaining_time(
        self, context: "RequestContext", timeout_type: TimeoutType
    ) -> float:
        """获取指定类型的剩余超时时间"""
        elapsed = context.elapsed()
        max_time = self.config.timeouts.get(timeout_type, float('inf'))

        # 检查父级限制
        if timeout_type == TimeoutType.LLM_CALL:
            parent_remaining = min(
                self.config.timeouts[TimeoutType.SINGLE_STEP] - context.step_elapsed(),
                self.config.timeouts[TimeoutType.AGENT_LOOP] - elapsed,
                self.config.timeouts[TimeoutType.TOTAL_REQUEST] - elapsed,
            )
        elif timeout_type == TimeoutType.TOOL_CALL:
            parent_remaining = min(
                self.config.timeouts[TimeoutType.SINGLE_STEP] - context.step_elapsed(),
                self.config.timeouts[TimeoutType.AGENT_LOOP] - elapsed,
                self.config.timeouts[TimeoutType.TOTAL_REQUEST] - elapsed,
            )
        elif timeout_type == TimeoutType.SINGLE_STEP:
            parent_remaining = min(
                self.config.timeouts[TimeoutType.AGENT_LOOP] - elapsed,
                self.config.timeouts[TimeoutType.TOTAL_REQUEST] - elapsed,
            )
        else:
            parent_remaining = self.config.timeouts[TimeoutType.TOTAL_REQUEST] - elapsed

        return min(max_time, parent_remaining, 0.1)  # 最少 100ms

    def record_timeout(self, timeout_type: TimeoutType):
        """记录超时事件"""
        self.metrics["timeouts_triggered"] += 1
        self.metrics["timeouts_by_type"][timeout_type.value] = \
            self.metrics["timeouts_by_type"].get(timeout_type.value, 0) + 1


class RequestContext:
    """请求级超时上下文"""

    def __init__(self, manager: TimeoutManager):
        self.manager = manager
        self.start_time = time.monotonic()
        self.step_start_time = time.monotonic()
        self.steps = 0

    def elapsed(self) -> float:
        return time.monotonic() - self.start_time

    def step_elapsed(self) -> float:
        return time.monotonic() - self.step_start_time

    def start_step(self):
        """开始新的一步"""
        self.steps += 1
        self.step_start_time = time.monotonic()

    def check_total_timeout(self):
        """检查总体超时"""
        if self.elapsed() >= self.manager.config.timeouts[TimeoutType.TOTAL_REQUEST]:
            self.manager.record_timeout(TimeoutType.TOTAL_REQUEST)
            raise TimeoutError(
                f"Total request timeout: {self.elapsed():.1f}s >= "
                f"{self.manager.config.timeouts[TimeoutType.TOTAL_REQUEST]}s"
            )

    def check_step_timeout(self):
        """检查单步超时"""
        step_elapsed = self.step_elapsed()
        step_timeout = self.manager.config.timeouts[TimeoutType.SINGLE_STEP]
        if step_elapsed >= step_timeout:
            self.manager.record_timeout(TimeoutType.SINGLE_STEP)
            raise TimeoutError(
                f"Step timeout: {step_elapsed:.1f}s >= {step_timeout}s"
            )

    def check_loop_timeout(self):
        """检查循环超时 (总步数时间)"""
        elapsed = self.elapsed()
        loop_timeout = self.manager.config.timeouts[TimeoutType.AGENT_LOOP]
        if elapsed >= loop_timeout:
            self.manager.record_timeout(TimeoutType.AGENT_LOOP)
            raise TimeoutError(
                f"Agent loop timeout: {elapsed:.1f}s >= {loop_timeout}s"
            )
```

### 2.2 带超时的 LLM 调用包装器

```python
class LLMTimeoutWrapper:
    """
    LLM 调用超时包装器。

    为 LLM API 调用提供多层超时保护:
    - 连接超时: 建立连接的最长等待
    - 读取超时: 收到首 token 的最长等待
    - 总超时: 完整响应的最长等待
    """

    def __init__(
        self,
        llm_client,
        connect_timeout: float = 5.0,
        read_timeout: float = 15.0,
        total_timeout: float = 30.0,
    ):
        self.client = llm_client
        self.connect_timeout = connect_timeout
        self.read_timeout = read_timeout
        self.total_timeout = total_timeout

    async def complete(
        self,
        messages: list[dict],
        tools: list[dict] = None,
        **kwargs,
    ) -> dict:
        """
        带多层超时的 LLM 调用。

        超时层级:
        1. connect_timeout: 建立连接
        2. read_timeout: 收首 token
        3. total_timeout: 完整响应
        """
        start = time.monotonic()

        try:
            # 创建带总超时的任务
            task = asyncio.create_task(
                self._do_complete(messages, tools, **kwargs)
            )

            # 总超时保护
            try:
                result = await asyncio.wait_for(
                    task, timeout=self.total_timeout
                )
            except asyncio.TimeoutError:
                task.cancel()
                raise LLMTimeoutError(
                    f"LLM call exceeded total timeout ({self.total_timeout}s)"
                )

            elapsed = time.monotonic() - start
            return {
                **result,
                "_timeout_metrics": {
                    "total_time": elapsed,
                    "connect_timeout": self.connect_timeout,
                    "total_timeout": self.total_timeout,
                },
            }

        except asyncio.TimeoutError:
            raise LLMTimeoutError(
                f"LLM call timed out after {time.monotonic() - start:.1f}s"
            )

    async def _do_complete(self, messages, tools, **kwargs):
        """实际的 LLM 调用"""
        # 不同 Provider 的超时设置不同, 这里以 OpenAI 为例
        return await self.client.chat.completions.create(
            messages=messages,
            tools=tools,
            timeout=self.connect_timeout,  # 连接超时
            **kwargs,
        )

    async def complete_with_stream(
        self,
        messages: list[dict],
        tools: list[dict] = None,
        first_token_timeout: float = 10.0,
        **kwargs,
    ) -> dict:
        """
        流式调用的超时控制。

        额外控制:
        - first_token_timeout: 首 token 超时
        - idle_timeout: 流式传输中断的超时
        """
        stream = await self.client.chat.completions.create(
            messages=messages,
            tools=tools,
            stream=True,
            **kwargs,
        )

        content = []
        first_token_received = False

        try:
            async for chunk in stream:
                if not first_token_received:
                    first_token_received = True

                if chunk.choices and chunk.choices[0].delta.content:
                    content.append(chunk.choices[0].delta.content)

                # 检查总超时
                if time.monotonic() - start > self.total_timeout:
                    stream.close()
                    raise LLMTimeoutError("Stream exceeded total timeout")

        except asyncio.TimeoutError:
            raise LLMTimeoutError("Stream chunk timeout")

        if not first_token_received:
            raise LLMTimeoutError(f"No first token within {first_token_timeout}s")

        return {"content": "".join(content)}


class LLMTimeoutError(Exception):
    """LLM 调用超时异常"""
    pass
```

### 2.3 Agent 循环超时

Agent 主循环的超时控制是核心——既要防止无限循环，又要给复杂任务足够的执行时间：

```python
class AgentLoopTimeout:
    """
    Agent 主循环超时控制器。

    负责:
    - 跟踪每一步的执行时间
    - 在超时阈值触发时优雅中断
    - 超时后返回"部分结果"而非直接报错
    """

    def __init__(
        self,
        timeout_mgr: TimeoutManager,
        context: RequestContext,
        max_steps: int = 15,
    ):
        self.timeout_mgr = timeout_mgr
        self.context = context
        self.max_steps = max_steps
        self.partial_results: list[dict] = []

    async def execute_step(self, step_fn) -> tuple[bool, Optional[dict]]:
        """
        执行一步 Agent 循环, 带超时控制。

        Returns:
            (是否继续, 步骤结果)
        """
        # 检查步数限制
        if self.context.steps >= self.max_steps:
            return False, {
                "status": "max_steps_reached",
                "steps": self.context.steps,
                "partial_results": self.partial_results,
            }

        # 检查总体超时
        try:
            self.context.check_total_timeout()
        except TimeoutError as e:
            return False, {
                "status": "total_timeout",
                "error": str(e),
                "partial_results": self.partial_results,
            }

        # 检查循环超时
        try:
            self.context.check_loop_timeout()
        except TimeoutError as e:
            return False, {
                "status": "loop_timeout",
                "error": str(e),
                "partial_results": self.partial_results,
            }

        self.context.start_step()

        # 单步执行 (带单步超时)
        try:
            result = await asyncio.wait_for(
                step_fn(),
                timeout=self.timeout_mgr.get_remaining_time(
                    self.context, TimeoutType.SINGLE_STEP
                ),
            )
            self.partial_results.append({
                "step": self.context.steps,
                "status": "success",
                "data": result,
            })
            return True, result

        except asyncio.TimeoutError:
            self.timeout_mgr.record_timeout(TimeoutType.SINGLE_STEP)
            self.partial_results.append({
                "step": self.context.steps,
                "status": "step_timeout",
            })
            # 单步超时不一定终止整个任务
            # 如果超时发生在非关键步骤, 可以继续
            return self._handle_step_timeout()

    def _handle_step_timeout(self) -> tuple[bool, dict]:
        """
        单步超时的恢复策略：
        根据超时发生的上下文决定终止还是继续。
        """
        step_timed_out = sum(
            1 for r in self.partial_results
            if r.get("status") == "step_timeout"
        )
        if step_timed_out >= 3:
            # 连续 3 步超时, 终止循环
            return False, {
                "status": "consecutive_step_timeout",
                "partial_results": self.partial_results,
            }

        # 单步超时, 跳过继续
        return True, {
            "status": "step_skipped",
            "reason": "timeout",
        }

    async def run(self, initial_fn, step_fn, finalize_fn) -> dict:
        """
        运行完整的 Agent 循环 (带超时)。

        Args:
            initial_fn: 初始化 (创建初始消息)
            step_fn: 单步执行 (Thought→Action→Observation)
            finalize_fn: 收尾 (生成最终回答)
        """
        # 初始化
        state = await initial_fn()

        # 主循环
        while True:
            should_continue, result = await self.execute_step(
                lambda: step_fn(state)
            )
            if not should_continue:
                # 超时或完成, 尝试收尾
                try:
                    final_result = await asyncio.wait_for(
                        finalize_fn(state, result),
                        timeout=10.0,  # 收尾超时较短
                    )
                    return {
                        **final_result,
                        "_timeout_info": {
                            "total_steps": self.context.steps,
                            "total_time": self.context.elapsed(),
                            "status": result.get("status", "completed"),
                        },
                    }
                except asyncio.TimeoutError:
                    # 收尾也超时, 返回部分结果
                    return {
                        "content": self._build_partial_response(state),
                        "_timeout_info": {
                            "total_steps": self.context.steps,
                            "total_time": self.context.elapsed(),
                            "status": "finalize_timeout",
                        },
                    }

            state = result

    def _build_partial_response(self, state) -> str:
        """超时后构建部分响应"""
        return (
            "I was unable to complete the full analysis within the time limit. "
            "Here's what I've found so far...\n\n"
            # 基于 state 和 partial_results 生成
        )
```

### 2.4 动态超时

固定超时不能适应 Agent 任务复杂度的变化。动态超时根据历史数据和任务特征自动调整：

```python
class DynamicTimeoutOptimizer:
    """
    动态超时优化器。

    基于历史执行数据, 为不同任务类型自动调整超时参数。
    """

    def __init__(self):
        # 按任务类型记录执行时间分布
        self.execution_history: dict[str, list[float]] = {}
        # 超时配置
        self.timeout_configs: dict[str, float] = {}

    def record_execution(self, task_type: str, duration: float, timed_out: bool):
        """记录一次执行时间"""
        if task_type not in self.execution_history:
            self.execution_history[task_type] = []
        self.execution_history[task_type].append({
            "duration": duration,
            "timed_out": timed_out,
            "timestamp": time.time(),
        })
        # 只保留最近 1000 条
        if len(self.execution_history[task_type]) > 1000:
            self.execution_history[task_type].pop(0)

    def suggested_timeout(self, task_type: str, percentile: float = 0.95) -> float:
        """
        推荐超时时间。

        使用历史数据的指定百分位作为超时建议。
        例如 P95 = 95% 的请求在该时间内完成。
        """
        records = self.execution_history.get(task_type, [])
        if len(records) < 10:
            return 30.0  # 默认值

        durations = sorted([
            r["duration"] for r in records if not r["timed_out"]
        ])
        if not durations:
            return 30.0

        idx = int(len(durations) * percentile)
        recommended = durations[min(idx, len(durations) - 1)]

        # 至少加 20% 缓冲
        return recommended * 1.2

    def timeout_efficiency(self, task_type: str) -> float:
        """
        超时效率: 超时的请求是不是真的该超时?

        如果大量请求在超时阈值附近才勉强完成,
        说明阈值设得太紧。如果大多数请求远低于阈值,
        阈值可以适当收紧。
        """
        records = self.execution_history.get(task_type, [])
        if len(records) < 20:
            return 1.0

        current_timeout = self.timeout_configs.get(task_type, 30.0)
        near_timeout = sum(
            1 for r in records
            if not r["timed_out"]
            and r["duration"] > current_timeout * 0.8
        )
        timed_out = sum(1 for r in records if r["timed_out"])

        total = len(records)
        return (near_timeout + timed_out) / total
```

### 2.5 超时后的优雅处理

超时不是终点——超时后的处理方式同样重要：

```python
class TimeoutRecoveryHandler:
    """
    超时恢复处理器。

    超时发生后, 根据超时类型和执行状态选择合适的恢复策略。
    """

    def __init__(self):
        self.recovery_strategies = {
            TimeoutType.LLM_CALL: self._recover_llm_timeout,
            TimeoutType.TOOL_CALL: self._recover_tool_timeout,
            TimeoutType.SINGLE_STEP: self._recover_step_timeout,
            TimeoutType.AGENT_LOOP: self._recover_loop_timeout,
            TimeoutType.TOTAL_REQUEST: self._recover_total_timeout,
        }

    async def handle_timeout(
        self,
        timeout_type: TimeoutType,
        context: dict,
    ) -> dict:
        """
        处理超时事件。

        Args:
            timeout_type: 超时类型
            context: 超时发生时的上下文 (状态、部分结果等)

        Returns:
            恢复后的响应
        """
        strategy = self.recovery_strategies.get(timeout_type)
        if strategy:
            return await strategy(context)
        return self._default_recovery(context)

    async def _recover_llm_timeout(self, context: dict) -> dict:
        """LLM 调用超时: 换更快模型重试"""
        return {
            "strategy": "retry_faster_model",
            "model": "gpt-4o-mini",  # 换小模型
            "max_tokens": context.get("max_tokens", 1024),
        }

    async def _recover_tool_timeout(self, context: dict) -> dict:
        """工具调用超时: 检查是否有缓存结果"""
        tool_name = context.get("tool_name")
        arguments = context.get("arguments")

        # 检查缓存
        cached = await self._check_cache(tool_name, arguments)
        if cached:
            return {
                "strategy": "cache_hit",
                "result": cached,
            }

        # 无缓存, 跳过该工具
        return {
            "strategy": "skip_tool",
            "message": f"Tool '{tool_name}' timed out, skipping",
        }

    async def _recover_step_timeout(self, context: dict) -> dict:
        """单步超时: 跳过当前步骤继续"""
        return {
            "strategy": "skip_step",
            "collected_results": context.get("partial_results", []),
        }

    async def _recover_loop_timeout(self, context: dict) -> dict:
        """循环超时: 尝试用已有结果生成回答"""
        return {
            "strategy": "partial_results",
            "content": self._summarize_partial(context.get("state", {})),
            "steps_completed": context.get("steps", 0),
        }

    async def _recover_total_timeout(self, context: dict) -> dict:
        """总超时: 返回错误 + 部分结果"""
        return {
            "strategy": "timeout_error",
            "error": "Request exceeded total time limit",
            "partial_content": context.get("partial_content", ""),
            "suggest_retry": True,
        }

    async def _default_recovery(self, context: dict) -> dict:
        """默认恢复"""
        return {
            "strategy": "error",
            "error": "Request timed out",
        }

    async def _check_cache(self, tool_name: str, arguments: dict):
        # 缓存查询逻辑
        return None

    def _summarize_partial(self, state: dict) -> str:
        return "I was interrupted. Here's what I found so far..."
```

---

## 3. 超时配置指南

### 3.1 不同场景的超时参考

| 场景 | LLM 调用超时 | 单步超时 | 循环超时 | 总超时 | 说明 |
|------|-------------|---------|---------|--------|------|
| 简单问答 | 10s | 15s | 30s | 60s | 一步完成, 快速响应 |
| 代码生成 | 30s | 45s | 120s | 180s | 代码生成可能需要多步推理 |
| 多工具 Agent | 20s | 30s | 120s | 180s | 工具调用增加延迟 |
| 研究型 Agent | 30s | 60s | 300s | 600s | 多步检索+分析 |
| 流式对话 | 30s | 30s | 60s | 120s | 首 token 需快 |

### 3.2 超时设计原则

```
1. 从外到内设置超时
   总超时 → 循环超时 → 单步超时 → LLM/工具超时
   每层预留 20-30% 的缓冲时间

2. 超时后要返回"部分结果"
   超时 ≠ 白屏。尽可能利用已完成的步骤

3. 超时是动态的
   基于历史 P95 执行时间 + 任务复杂度动态调整

4. 超时是可观测的
   记录超时率、超时分布、超时恢复后的满意度
```

---

## 4. 能力边界

### 4.1 超时做不到

```
超时控制做不到:
  ┌─ 加快 LLM 响应速度 (超时只是切断, 不是加速)
  ├─ 区分"快完成任务了"和"刚陷入循环"
  │  超时只能看时间, 不能看进度
  ├─ 恢复所有超时造成的损失
  │  超时浪费的 Token 和时间无法回收
  └─ 替代重试逻辑
      超时后的最佳策略通常是重试或降级, 而非等待
```

### 4.2 常见反模式

```
❌ "超时设长一点, 以防万一"
   超时太长 → 循环 Agent 燃烧更多 Token
   超时太短 → 复杂任务被频繁中断
   需要基于数据动态调整

❌ "所有场景用同一套超时"
   不同任务类型的执行时间差异可达 10 倍。
   需要按任务类型配置独立超时。

❌ "超时不通知用户"
   用户应在超时时获得反馈, 而不是等界面转圈到超时。
   建议使用流式输出 + 中间状态更新。

❌ "超时就是放弃"
   超时不等于失败。超时后应返回已有部分结果,
   或提供降级选项。
```

---

## 5. 与其他模式对比

| 模式 | 关系 | 配合方式 |
|------|------|---------|
| Retry | 互补 | 超时后可重试, 重试本身也要有超时 |
| Circuit Breaker | 互补 | CB 防止重试, 超时是 CB 的触发条件之一 |
| Bulkhead | 依赖 | 隔舱中的连接需要超时来释放 |
| Degradation | 互补 | 超时后触发降级, 返回部分结果 |
| Streaming | 辅助 | 流式输出需要 idle_timeout 检测断流 |

---

## 6. 工程优化方向

### 6.1 实施步骤

```
1. 建立基准: 收集各任务类型的执行时间分布
2. 设置初始超时: 基于 P95 + 20% 缓冲
3. 实现分层超时: 总超时 → 循环 → 单步 → 调用
4. 超时恢复: 每种超时类型的优雅处理
5. 动态优化: 基于实时数据自动调整超时参数
6. A/B 测试: 对比不同超时设置的成功率和延迟
```

### 6.2 监控指标

```
超时监控:
  ├── 超时率: 各层级超时发生的频率 (目标 < 1%)
  ├── 平均执行时间: 任务的成功完成时间 (P50/P95/P99)
  ├── 超时恢复率: 超时后用户继续使用/重试的比例
  ├── 超时位置分布: 超时发生在哪些步骤/工具/模型上
  └── 超时成本: 超时浪费的 Token 和延迟
```

### 6.3 实践检查清单

```
[ ] 1. 所有外部调用 (LLM/工具) 都有超时保护
[ ] 2. 超时是分层的: 内层 < 外层
[ ] 3. 超时后返回部分结果而不是空响应
[ ] 4. 超时参数按任务类型差异化配置
[ ] 5. 超时参数基于历史数据动态调整
[ ] 6. 流式传输有 idle_timeout
[ ] 7. 超时事件有详细的日志和 Trace
[ ] 8. 超时恢复策略已实现 (重试/降级/缓存)
[ ] 9. 超时对用户透明 (告知等待原因或超时原因)
[ ] 10. 定期审查超时配置 (模型升级后重新评估)
```

---

> **总结**：超时控制是 Agent 系统防止失控的最后一道防线。核心在于分层设计——总超时 > 循环超时 > 单步超时 > LLM/工具调用超时，每层预留缓冲。好的超时控制不只是"超时后报错"，而是超时后返回部分结果、触发降级、或为用户提供替代方案。关键原则：(1) 超时参数应基于历史数据动态调整；(2) 超时后必须优雅处理；(3) 超时设置需按任务类型差异化配置。
