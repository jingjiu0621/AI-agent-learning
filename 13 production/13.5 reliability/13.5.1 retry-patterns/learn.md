# Retry Patterns —— LLM Agent 系统的重试模式

> **核心挑战**：Agent 系统中重试不再是简单的"再试一次"。LLM API 调用可能在 2% 的情况下返回格式错误、5% 的情况下输出偏差、偶尔因为限流返回 429——但重试策略稍有不慎就会导致成本倍增、延迟恶化、甚至放大故障。Retry Pattern 的核心任务是设计一套**可自适应**的重试策略，在"值得重试"和"应该放弃"之间找到精确的平衡点，同时对 LLM 特有的故障模式（输出格式错、语义偏差、工具选择错）做出针对性处理。

---

## 1. 基本原理

### 1.1 重试的必要性

Agent 系统的调用链比传统系统更长，每一步都可能失败：

```
传统系统: 请求 → 服务 → 响应
           失败率: ~0.1% (单一网络调用)

Agent 系统: 请求 → Thought → Tool Call → LLM → Tool Exec → LLM → ...
            每个环节都可能失败                 失败率: > 5%/步
```

### 1.2 Agent 重试决策树

Agent 系统中的重试决策比传统系统复杂得多——不仅要判断"重不重试"，还要判断"怎么重试"：

```
                                        收到错误/异常
                                             │
                                            ▼
                              ┌──────────────────────────┐
                              │  失败类型分类             │
                              └────────┬─────────────────┘
                                       │
          ┌──────────────┬──────────────┼──────────────┬──────────────┐
          ▼              ▼              ▼              ▼              ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
   │ 瞬态网络错误 │ │ LLM 限流   │ │ LLM 格式错  │ │ LLM 语义错  │ │ 工具执行失败│
   │ (TCP 超时)  │ │ (429/503) │ │ (JSON 解析  │ │ (幻觉/偏差) │ │ (API 返回   │
   └──────┬─────┘ └──────┬─────┘ │  失败)     │ └──────┬─────┘ │  错误)     │
          │              │       └──────┬─────┘        │       └──────┬─────┘
          ▼              ▼              ▼              ▼              ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
   │快速重试+    │ │退避重试+    │ │修复 Prompt  │ │换模型重试   │ │检查参数    │
   │Jitter      │ │排队等待    │ │重试         │ │或修正输出   │ │后重试      │
   │max=3      │ │max=5      │ │max=2       │ │max=2       │ │max=3      │
   └────────────┘ └────────────┘ └────────────┘ └────────────┘ └────────────┘
```

### 1.3 重试的成本模型

每次重试都是有代价的。理解成本才能做出合理的重试决策：

```
重试的显性成本:
  ┌─ LLM API 费用 (每次重试 = 一次完整的 API 调用)
  ├─ 延迟增加 (重试间隔 × 重试次数)
  ├─ Token 膨胀 (重试时历史消息更长 → 每次成本递增)
  └─ 用户等待时间延长

重试的隐性成本 (Agent 特有):
  ┌─ 上下文污染: 重试产生的错误信息可能进入上下文, 影响后续推理
  ├─ 状态不一致: 部分状态已变更 (如工具已执行), 重试可能导致重复操作
  └─ 级联延迟: 一个 Agent 的重试占用资源, 阻塞其他请求

成本-收益决策: 重试的期望收益 > 重试的总成本?
  E[收益] = 重试成功率 × 任务成功价值
  E[成本] = 重试次数 × 单次调用成本 + 延迟机会成本 + 风险成本
```

---

## 2. 背景与演进

### 2.1 从网络重试到 Agent 重试

```
时期         重试策略                       触发条件
─────        ──────                         ──────
Web 1.0      固定间隔重试                     HTTP 5xx
Web 2.0      指数退避 + jitter                HTTP 5xx / 超时
微服务        退避 + 熔断 + 幂等性              网络异常 + 业务异常
LLM API      退避 + 多 Provider              限流(429) + 超时
Agent 时代    退化感知 + 自适应 + 语义验证        LLM 输出格式/语义/工具选择等多维故障
```

### 2.2 为什么 Agent 重试不能照搬传统方案

| 维度 | 传统微服务重试 | Agent 重试 |
|------|--------------|-----------|
| 故障模式 | 网络抖动、超时、5xx | 格式错误、语义偏差、工具误选 + 网络问题 |
| 失败检测 | HTTP 状态码清晰 | 需要解析输出内容判断语义正确性 |
| 重试副作用 | 幂等性保证即可 | 工具可能已执行（非幂等）、上下文变长成本增加 |
| 重试策略 | 固定参数 | 依赖失败类型动态调整（换模型/换 Prompt/换参数） |
| 成功率上限 | ~99.9%（3次重试后） | 受 LLM 能力上限约束，不一定是 100% |

---

## 3. 核心技术

### 3.1 指数退避与 Jitter

最基础的退避策略。关键参数是初始间隔、乘数因子、最大间隔和随机扰动：

```python
import random
import time
import asyncio
from enum import Enum
from typing import Optional, Callable, Awaitable

class BackoffStrategy(Enum):
    EXPONENTIAL = "exponential"
    LINEAR = "linear"
    CONSTANT = "constant"
    DECORRELATED_JITTER = "decorrelated_jitter"

class RetryConfig:
    """重试配置"""

    def __init__(
        self,
        max_retries: int = 3,
        base_delay: float = 1.0,
        max_delay: float = 60.0,
        multiplier: float = 2.0,
        jitter: float = 0.1,
        strategy: BackoffStrategy = BackoffStrategy.EXPONENTIAL,
    ):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.multiplier = multiplier
        self.jitter = jitter
        self.strategy = strategy

    def get_delay(self, attempt: int) -> float:
        """计算第 attempt 次重试前的等待时间"""
        if self.strategy == BackoffStrategy.EXPONENTIAL:
            delay = min(
                self.base_delay * (self.multiplier ** attempt),
                self.max_delay
            )
        elif self.strategy == BackoffStrategy.LINEAR:
            delay = min(
                self.base_delay * (attempt + 1),
                self.max_delay
            )
        elif self.strategy == BackoffStrategy.DECORRELATED_JITTER:
            # 去相关抖动: delay = min(cap, random(base, delay * 3))
            prev_delay = self.base_delay * (self.multiplier ** max(0, attempt - 1))
            delay = min(
                self.max_delay,
                random.uniform(self.base_delay, prev_delay * 3)
            )
        else:  # CONSTANT
            delay = self.base_delay

        # 添加随机抖动以避免惊群效应
        jitter_amount = delay * self.jitter
        delay += random.uniform(-jitter_amount, jitter_amount)
        return max(0, delay)
```

### 3.2 Agent 专用重试器

相比通用的重试逻辑，Agent 重试需要理解不同的失败模式并做出针对性响应：

```python
from dataclasses import dataclass, field
from enum import auto

class FailureCategory(Enum):
    NETWORK = auto()           # TCP 超时、连接重置
    RATE_LIMIT = auto()        # 429 Too Many Requests
    SERVER_ERROR = auto()      # 500, 503
    FORMAT_ERROR = auto()      # JSON 解析失败、Schema 验证失败
    SEMANTIC_ERROR = auto()    # 输出内容事实错误
    TOOL_SELECTION_ERROR = auto()  # 选择了错误的工具
    CONTEXT_OVERSIZE = auto()  # 上下文超出窗口限制
    UNKNOWN = auto()

@dataclass
class RetryDecision:
    should_retry: bool
    strategy_override: Optional[dict] = None  # 重试策略参数覆盖
    prompt_modification: Optional[str] = None  # Prompt 修改建议
    model_override: Optional[str] = None       # 换模型
    fallback_action: Optional[str] = None      # 降级动作

class AgentRetryHandler:
    """
    Agent 专用重试处理器。
    根据失败类型选择不同的重试策略。
    """

    def __init__(self, default_config: RetryConfig = None):
        self.default_config = default_config or RetryConfig()
        self.failure_count: dict[str, int] = {}  # 按 endpoint 跟踪失败计数

    def classify_failure(
        self,
        error: Exception,
        response: Optional[str] = None,
    ) -> FailureCategory:
        """分类失败类型"""
        error_str = str(error).lower()

        if isinstance(error, (ConnectionError, TimeoutError)):
            return FailureCategory.NETWORK
        if "429" in error_str or "rate limit" in error_str:
            return FailureCategory.RATE_LIMIT
        if "500" in error_str or "503" in error_str:
            return FailureCategory.SERVER_ERROR
        if "json" in error_str and ("parse" in error_str or "decode" in error_str):
            return FailureCategory.FORMAT_ERROR
        if "context length" in error_str or "maximum context" in error_str:
            return FailureCategory.CONTEXT_OVERSIZE
        return FailureCategory.UNKNOWN

    def decide(self, category: FailureCategory, attempt: int) -> RetryDecision:
        """根据失败类型和重试次数做出决策"""

        max_retries_map = {
            FailureCategory.NETWORK: 3,
            FailureCategory.RATE_LIMIT: 5,
            FailureCategory.SERVER_ERROR: 3,
            FailureCategory.FORMAT_ERROR: 2,
            FailureCategory.SEMANTIC_ERROR: 2,
            FailureCategory.TOOL_SELECTION_ERROR: 2,
            FailureCategory.CONTEXT_OVERSIZE: 0,  # 不能通过重试解决
            FailureCategory.UNKNOWN: 1,
        }

        max_r = max_retries_map.get(category, 1)
        if attempt >= max_r:
            return RetryDecision(should_retry=False)

        # 根据不同失败类型定制策略
        if category == FailureCategory.RATE_LIMIT:
            return RetryDecision(
                should_retry=True,
                strategy_override={"base_delay": 5.0, "multiplier": 3.0},
            )
        elif category == FailureCategory.FORMAT_ERROR:
            return RetryDecision(
                should_retry=True,
                prompt_modification=(
                    "IMPORTANT: Your previous response had a format error. "
                    "Ensure your output is valid JSON."
                ),
            )
        elif category == FailureCategory.SEMANTIC_ERROR:
            return RetryDecision(
                should_retry=True,
                model_override="gpt-4-turbo",  # 换更可靠的模型
            )
        elif category == FailureCategory.CONTEXT_OVERSIZE:
            return RetryDecision(
                should_retry=False,
                fallback_action="compress_context",
            )
        else:
            # 瞬态故障: 标准退避
            return RetryDecision(should_retry=True)
```

### 3.3 语义验证重试

最复杂的场景：LLM 调用"成功"（状态码 200）但输出内容有问题。这需要额外的验证层来判断是否需要重试：

```python
import json
from typing import Any

class SemanticValidator:
    """
    验证 LLM 输出内容的语义正确性。
    如果输出不符合预期, 触发"语义级重试"。
    """

    def __init__(self, schema: dict):
        self.schema = schema

    def validate_tool_call(self, tool_name: str, arguments: dict) -> tuple[bool, str]:
        """
        验证工具调用的选择是否正确。
        返回: (是否通过, 失败原因)
        """
        # 1. 参数完整性检查
        required_params = self.schema.get("required", [])
        missing = [p for p in required_params if p not in arguments]
        if missing:
            return False, f"Missing required parameters: {missing}"

        # 2. 参数类型检查
        for param, value in arguments.items():
            param_schema = self.schema.get("properties", {}).get(param, {})
            expected_type = param_schema.get("type")
            if expected_type and not self._type_check(value, expected_type):
                return False, f"Parameter '{param}' expected {expected_type}, got {type(value).__name__}"

        # 3. 参数值范围检查 (如果定义了 enum 或 minimum/maximum)
        for param, value in arguments.items():
            param_schema = self.schema.get("properties", {}).get(param, {})
            enum_values = param_schema.get("enum")
            if enum_values and value not in enum_values:
                return False, f"Parameter '{param}' value '{value}' not in allowed: {enum_values}"

        return True, ""

    def validate_response_content(
        self,
        response: str,
        expected_keywords: list[str] = None,
        forbidden_patterns: list[str] = None,
    ) -> tuple[bool, str]:
        """
        验证 LLM 输出内容的质量。
        """
        if expected_keywords:
            missing = [kw for kw in expected_keywords if kw.lower() not in response.lower()]
            if missing:
                return False, f"Response missing expected content: {missing[:3]}"

        if forbidden_patterns:
            found = [p for p in forbidden_patterns if p.lower() in response.lower()]
            if found:
                return False, f"Response contains forbidden patterns: {found[:3]}"

        return True, ""

    def _type_check(self, value: Any, expected_type: str) -> bool:
        type_map = {
            "string": str,
            "integer": int,
            "number": (int, float),
            "boolean": bool,
            "array": list,
            "object": dict,
        }
        py_type = type_map.get(expected_type)
        if py_type is None:
            return True  # 未知类型跳过
        return isinstance(value, py_type)


class SemanticRetryWrapper:
    """
    包装 LLM 调用, 在语义验证失败时触发重试。
    """

    def __init__(
        self,
        llm_call_fn,
        validator: SemanticValidator,
        max_retries: int = 2,
    ):
        self.llm_call_fn = llm_call_fn
        self.validator = validator
        self.max_retries = max_retries

    async def execute_with_semantic_retry(
        self,
        messages: list[dict],
        tools: list[dict] = None,
    ) -> tuple[bool, str, dict]:
        """
        执行 LLM 调用并做语义验证, 失败时重试。
        返回: (是否在语义上通过, 响应内容, 元数据)
        """
        last_error = ""
        for attempt in range(self.max_retries + 1):
            # 选择模型
            model = "gpt-4-turbo" if attempt > 0 else "gpt-4o-mini"
            if attempt > 0:
                # 在消息中添加纠正指令
                correction_msg = {
                    "role": "user",
                    "content": (
                        f"Your previous response had an issue: {last_error}. "
                        "Please correct your response."
                    )
                }
                messages = messages + [correction_msg]

            # 调用 LLM
            response = await self.llm_call_fn(messages, tools=tools, model=model)

            # 验证工具调用 (如果有)
            if tools and response.get("tool_calls"):
                for tc in response["tool_calls"]:
                    try:
                        args = json.loads(tc["function"]["arguments"])
                    except json.JSONDecodeError as e:
                        last_error = f"JSON parse error: {e}"
                        continue

                    valid, reason = self.validator.validate_tool_call(
                        tc["function"]["name"], args
                    )
                    if not valid:
                        last_error = reason
                        break
                else:
                    return True, response, {"attempts": attempt + 1, "semantic_pass": True}
            else:
                # 验证文本响应
                valid, reason = self.validator.validate_response_content(
                    response.get("content", "")
                )
                if valid:
                    return True, response, {"attempts": attempt + 1, "semantic_pass": True}
                last_error = reason

        return False, response, {"attempts": self.max_retries + 1, "semantic_pass": False, "last_error": last_error}
```

### 3.4 混合重试器：完整实现

将上述模式整合为一个完整的重试编排器：

```python
class RetryOrchestrator:
    """
    完整的 Agent 重试编排器。
    整合了网络级重试、语义验证和退避策略。
    """

    def __init__(
        self,
        llm_client,
        semantic_validator: SemanticValidator = None,
        default_retry_config: RetryConfig = None,
    ):
        self.client = llm_client
        self.validator = semantic_validator
        self.retry_handler = AgentRetryHandler(default_retry_config)
        self.metrics = {
            "total_calls": 0,
            "network_retries": 0,
            "semantic_retries": 0,
            "total_failures": 0,
        }

    async def execute(
        self,
        messages: list[dict],
        tools: list[dict] = None,
        context: dict = None,
    ) -> dict:
        """
        执行带完整重试逻辑的 LLM 调用。

        Args:
            messages: 消息列表
            tools: 工具定义
            context: 上下文信息 (用于决策)

        Returns:
            调用结果
        """
        self.metrics["total_calls"] += 1
        last_error = None
        last_response = None

        for attempt in range(self.retry_handler.default_config.max_retries + 1):
            try:
                # 调用前检查熔断器状态
                if self._is_circuit_open():
                    return self._fallback_response("Circuit breaker open")

                # 实际的 LLM 调用
                response = await self._call_with_timeout(messages, tools)

                # 语义验证 (如果有 validator)
                if self.validator:
                    valid, reason = self._validate_response(response, tools)
                    if not valid:
                        self.metrics["semantic_retries"] += 1
                        decision = self.retry_handler.decide(
                            FailureCategory.SEMANTIC_ERROR, attempt
                        )
                        if decision.should_retry:
                            # 修改 Prompt 后重试
                            messages = self._inject_correction(messages, reason)
                            if decision.model_override:
                                response["_model"] = decision.model_override
                            continue
                        else:
                            last_error = reason
                            break

                return response

            except Exception as e:
                category = self.retry_handler.classify_failure(e, response)
                decision = self.retry_handler.decide(category, attempt)

                if not decision.should_retry:
                    last_error = str(e)
                    break

                if category == FailureCategory.NETWORK:
                    self.metrics["network_retries"] += 1

                # 计算等待时间
                config_override = decision.strategy_override or {}
                temp_config = RetryConfig(**{
                    **self.retry_handler.default_config.__dict__,
                    **config_override
                })
                delay = temp_config.get_delay(attempt)

                self._update_circuit_state(category)
                await asyncio.sleep(delay)

        self.metrics["total_failures"] += 1
        return {
            "error": last_error,
            "attempts": attempt + 1,
            "failed": True,
            "last_response": last_response,
        }

    def _call_with_timeout(self, messages, tools):
        """带超时的 LLM 调用"""
        ...

    def _validate_response(self, response, tools):
        """验证响应语义正确性"""
        ...

    def _inject_correction(self, messages, error_reason):
        """在消息历史中注入纠正指令"""
        ...

    def _is_circuit_open(self):
        """检查熔断器状态"""
        ...

    def _update_circuit_state(self, category):
        """更新熔断器状态"""
        ...

    def _fallback_response(self, reason):
        """返回降级响应"""
        return {"error": reason, "fallback": True}
```

### 3.5 Agent 特有的重试场景

#### 场景 1：工具调用的幂等性保证

Agent 重试最危险的问题：工具已经执行，重试会导致重复操作。

```python
class IdempotentToolWrapper:
    """
    为工具调用增加幂等性保证。
    使用去重键 + 结果缓存来防止重复执行。
    """

    def __init__(self):
        self.execution_cache: dict[str, dict] = {}

    def _make_idempotency_key(
        self, tool_name: str, arguments: dict, request_id: str
    ) -> str:
        """生成幂等键: 工具名 + 关键参数 + 请求 ID"""
        canonical_args = json.dumps(arguments, sort_keys=True)
        return f"{request_id}:{tool_name}:{canonical_args}"

    async def execute_with_idempotency(
        self,
        tool_name: str,
        arguments: dict,
        request_id: str,
        execute_fn,
    ) -> dict:
        """幂等执行: 已缓存的结果直接返回, 不重复执行"""
        key = self._make_idempotency_key(tool_name, arguments, request_id)

        if key in self.execution_cache:
            return self.execution_cache[key]

        result = await execute_fn(tool_name, arguments)
        self.execution_cache[key] = result
        return result

    def is_idempotent_action(self, tool_name: str) -> bool:
        """判断工具是否天然幂等 (GET/查询等)"""
        idempotent_tools = {
            "search", "lookup", "get_weather", "calculate",
            "read_file", "list_files", "get_info",
        }
        return tool_name in idempotent_tools
```

#### 场景 2：Provider 级重试与降级

调用 LLM API 时的跨 Provider 重试：

```python
class ProviderRetryRouter:
    """
    跨 Provider 重试路由。
    当一个 Provider 持续失败时，切换到备用 Provider。
    """

    def __init__(self, providers: list[dict]):
        self.providers = providers  # [{"name": "openai", "client": ..., "model": "gpt-4"}, ...]
        self.current_idx = 0
        self.failure_counts: dict[str, int] = {}

    async def execute_with_fallback(
        self, messages: list[dict], tools: list[dict] = None
    ) -> dict:
        """尝试当前 Provider, 失败后切换到下一个"""
        start_idx = self.current_idx
        attempts = []

        for i in range(len(self.providers)):
            idx = (start_idx + i) % len(self.providers)
            provider = self.providers[idx]

            try:
                response = await provider["client"].invoke(messages, tools)
                self.current_idx = idx
                self.failure_counts[provider["name"]] = 0
                return response
            except Exception as e:
                self.failure_counts[provider["name"]] = \
                    self.failure_counts.get(provider["name"], 0) + 1
                attempts.append({"provider": provider["name"], "error": str(e)})
                continue

        # 所有 Provider 都失败
        return {
            "error": "All providers failed",
            "attempts": attempts,
            "failed": True,
        }
```

---

## 4. 能力边界与风险

### 4.1 重试的局限

```
重试无法解决的问题:
  ┌─ LLM 能力边界: 如果模型本身不理解某个概念, 重试 100 次也没用
  ├─ 系统性故障: 整个 Provider 宕机时, 重试只会浪费时间和成本
  ├─ 设计缺陷: Prompt 有逻辑错误时, 重试次数再多也无法纠正
  ├─ 数据质量问题: 工具返回了错误数据, 重试 LLM 不会修复数据
  └─ 非瞬态错误: 认证失效、配额耗尽等持续性问题
```

### 4.2 不同失败类型的最佳重试策略

| 失败类型 | 重试策略 | 最大次数 | 等待时间 | 是否修改 Prompt |
|----------|---------|---------|----------|----------------|
| TCP 超时 | 快速重试 + jitter | 2-3 | 100ms-1s | 否 |
| 429 限流 | 长退避 + 排队 | 3-5 | 5s-60s | 否 |
| 500/503 | 退避重试 | 2-3 | 1s-10s | 否 |
| JSON 解析失败 | 重试 + 强调格式 | 1-2 | 0 | 是，加格式指令 |
| 语义偏差 | 换模型重试 | 1-2 | 0 | 是，加纠正 |
| 工具误选 | 重试 + 明确指令 | 1-2 | 0 | 是，加限制 |
| 上下文超长 | 不重试，压缩后重试 | 1 | — | 压缩后自动 |

### 4.3 重试风暴的风险

```
重试风暴 (Retry Storm):

  假设 100 个 Agent 实例同时遇到 Provider 限流:
    正常: 100 个请求 → 等待 → 恢复 → 继续
    重试风暴: 100 个请求 → 退避 N秒 → 100 个请求同时重试 → 再次限流 → 恶性循环

  解决方案:
    ┌─ Jitter (随机化等待时间): 避免惊群效应
    ├─ 熔断器 + 集体退避: 连续失败后集体停止重试
    ├─ 客户端限流: 限制单位时间内发往 Provider 的请求数
    └─ Provider 侧限制写入: 使用有上限的退避策略
```

---

## 5. 与其他模式对比

| 模式 | 解决的问题 | 重试的角色 |
|------|-----------|-----------|
| Retry + Timeout | 瞬态故障恢复 | 核心机制 |
| Retry + Circuit Breaker | 防止重试风暴 | 需要在 CB 闭合时才重试 |
| Retry + Bulkhead | 资源保护 | 重试使用隔舱内的配额 |
| Retry + Degradation | 最终兜底 | 重试耗尽后触发降级 |
| Retry + Idempotency | 防止副作用 | 重试的前提条件 |

---

## 6. 工程优化方向

### 6.1 自适应重试参数

最优的重试参数不是固定的，应基于实时数据动态调整：

```python
class AdaptiveRetryOptimizer:
    """
    基于历史数据动态调整重试参数。
    """

    def __init__(self, window_size: int = 100):
        self.window_size = window_size
        self.outcomes: list[dict] = []  # retry history

    def record_outcome(
        self,
        failure_type: str,
        retries: int,
        success: bool,
        latency: float,
    ):
        """记录一次重试的结果"""
        self.outcomes.append({
            "failure_type": failure_type,
            "retries": retries,
            "success": success,
            "latency": latency,
            "timestamp": time.time(),
        })
        if len(self.outcomes) > self.window_size:
            self.outcomes.pop(0)

    def suggested_max_retries(self, failure_type: str) -> int:
        """根据历史数据推荐最大重试次数"""
        relevant = [
            o for o in self.outcomes
            if o["failure_type"] == failure_type
        ]
        if len(relevant) < 10:
            return 3  # 数据不足时使用默认值

        # 分析每次重试的成功率曲线
        for n in range(1, 6):
            success_at_n = sum(
                1 for o in relevant
                if o["retries"] >= n and o["success"]
            )
            rate = success_at_n / len(relevant)
            if rate < 0.05:  # 第 N 次重试成功率低于 5%, 不值得
                return n - 1

        return 3

    def suggested_base_delay(self, failure_type: str) -> float:
        """推荐基础退避时间"""
        # 如果最近成功率低, 增加退避时间
        recent = [o for o in self.outcomes if o["success"] is False]
        if len(recent) > 20:
            return 2.0  # 增加退避
        return 1.0
```

### 6.2 可观测性

重试系统需要监控的关键指标：

```
重试监控指标:
  ├── 重试率: 重试次数 / 总调用次数 (基线 < 5%)
  ├── 重试成功率: 重试后成功的比例 (目标 > 80%)
  ├── 平均重试次数: 每次失败平均重试次数 (目标 < 2)
  ├── 重试成本占比: 因重试产生的额外 Token 占比
  ├── 无效重试率: 重试多次仍失败的请求比例
  └── Provider 切换率: 因失败切换 Provider 的频率
```

### 6.3 实践检查清单

```
重试策略实施的十个检查项:

[ ] 1. 根据失败类型分类制定不同的重试策略, 而不是一刀切
[ ] 2. 所有重试都带 jitter, 避免惊群效应
[ ] 3. 非幂等的工具调用使用去重键保护
[ ] 4. 语义验证层在重试前确认"失败不是由于语义偏差"
[ ] 5. 重试有上限 (最大次数 + 最大总耗时)
[ ] 6. 熔断器与重试协调: 熔断开启时不发起新重试
[ ] 7. 重试上下游均记录 Trace 可追溯
[ ] 8. 重试的 Token 成本单独追踪 (区别于首次调用)
[ ] 9. 跨 Provider 重试策略已实现并测试
[ ] 10. 定期分析重试数据, 优化退避参数和最大次数
```

---

> **总结**：Agent 系统的重试模式比传统系统复杂得多，核心差异在于失败类型的多样性——不仅包括网络级别的瞬态故障，还包括 LLM 特有的格式错误、语义偏差和工具选择错误。一套完善的重试策略必须：(1) 根据失败类型分类施策；(2) 集成语义验证来决定是否需要重试 LLM 调用；(3) 与熔断器、隔舱和降级等模式协调运作；(4) 包含幂等性保护机制防止重复执行副作用。好的重试策略不是"尽可能多试"，而是**在合适的时机、用合适的方式、试合适的次数**。
