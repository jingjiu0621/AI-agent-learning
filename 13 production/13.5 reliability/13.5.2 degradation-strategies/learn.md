# Degradation Strategies —— LLM Agent 系统的降级策略

> **核心挑战**：Agent 系统不可能在所有场景下都完美运行。当 LLM API 不可用、工具调用持续失败、或系统负载超过阈值时，降级策略决定了系统"失败"的方式——是优雅地提示用户并提供替代方案，还是直接崩溃报错。降级设计的核心是在**功能完备度**和**可用性**之间找到平衡，确保在最坏情况下用户仍能获得有价值的结果。

---

## 1. 基本原理

### 1.1 降级 vs 重试 vs 熔断

三种模式构成故障处理的完整链路：

```
故障处理层级:
                                        故障发生
                                            │
                                           ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Level 1: Retry                                              │
  │  尝试修复: 重试 1-3 次                                       │
  │  适用: 瞬态故障 (网络抖动、限流)                               │
  └──────────────────────────┬───────────────────────────────────┘
                             │ 重试失败
                             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Level 2: Circuit Breaker                                    │
  │  快速失败: 防止重试风暴, 保护下游                              │
  │  适用: 持续故障 (Provider 宕机、工具不可用)                     │
  └──────────────────────────┬───────────────────────────────────┘
                             │ 熔断开启
                             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Level 3: Degradation                                        │
  │  降级: 提供替代方案, 维持基本服务                              │
  │  适用: 降级策略, 保持可用                                     │
  └──────────────────────────────────────────────────────────────┘

关键顺序: Retry → Circuit Breaker → Degradation
         先尝试修复 → 避免无效重试 → 最后降级兜底
```

### 1.2 降级的维度

Agent 系统的降级可以在多个维度上进行，不同维度的降级对用户体验的影响不同：

```
降级维度光谱:

  功能完整          ████████████████████░░░░░░░░░░  完全不可用
  高精度            ████████████████░░░░░░░░░░░░░░  低精度/近似
  实时              ████████████░░░░░░░░░░░░░░░░░░  延迟/异步
  个性化            ██████████░░░░░░░░░░░░░░░░░░░░  通用模板
  自主性            ████████░░░░░░░░░░░░░░░░░░░░░░  人工介入
  丰富形式          ██████░░░░░░░░░░░░░░░░░░░░░░░░  纯文本

  ──── 正常模式 ────→  ──── 逐步降级 ────→  ──── 最低可用 ────→
```

### 1.3 降级触发条件

降级不应是随意的，需要明确的触发条件和退出条件：

```
触发降级:
  ┌─ 熔断器开启 (Circuit Breaker Open)
  ├─ 重试耗尽 (Max Retries Exceeded)
  ├─ LLM API 返回持续错误
  ├─ 工具调用持续失败
  ├─ 超时 (Timeout)
  ├─ 成本预算超限 (Cost Budget Exceeded)
  ├─ 系统负载过高 (High Load)
  └─ 依赖服务不可用 (Dependency Unavailable)

退出降级:
  ┌─ 触发条件消失 (如熔断器半开 → 关闭)
  ├─ 手动恢复 (运维干预)
  └─ 定时检查 (健康检查通过后自动恢复)
```

---

## 2. 背景与演进

### 2.1 降级策略的技术演进

```
时期            降级策略                    典型场景
─────           ──────                      ──────
单体应用        功能开关 (Feature Flag)      手动禁用非核心功能
微服务          熔断+降级, 默认返回值         服务依赖不可用时, 返回缓存/默认值
LLM 应用        静态回退 (Fallback)          Prompt 降级、模型降级
Agent 系统      多维渐进式降级                知识降级 → 工具降级 → 能力降级 → 人工接管
```

### 2.2 为什么 Agent 降级比传统系统复杂

| 维度 | 传统系统 | Agent 系统 |
|------|---------|-----------|
| 降级粒度 | 功能级 (如 "推荐功能不可用") | 能力级 (如 "不能查询实时数据, 但可以查缓存") |
| 降级表现 | 明确的错误信息 | 需要 LLM 理解当前能力范围并调整回答 |
| 恢复方式 | 无需告知用户, 静默恢复 | LLM 需要知道"能力已恢复" |
| 降级链 | 单维度降级 | 多维度串联/并联降级 |

---

## 3. 核心技术

### 3.1 降级级别定义

Agent 系统建议定义 5 级降级级别，从全功能到完全不可用：

```python
from enum import IntEnum
from dataclasses import dataclass
from typing import Optional

class DegradationLevel(IntEnum):
    """降级级别: 数值越大, 降级越严重"""
    FULL = 0          # 全功能: 一切正常
    DEGRADE_1 = 1     # 轻度降级: 缓存替代实时, 小模型替代大模型
    DEGRADE_2 = 2     # 中度降级: 禁用部分工具, 简化推理
    DEGRADE_3 = 3     # 重度降级: 仅支持基础问答, 禁用工具调用
    MINIMAL = 4       # 最低可用: 返回静态响应或转人工
    UNAVAILABLE = 5   # 完全不可用: 返回明确的不可用信息

    def description(self) -> str:
        descs = {
            0: "全功能模式: 所有能力正常可用",
            1: "轻度降级: 部分优化能力受限, 核心功能正常",
            2: "中度降级: 工具和推理能力受限, 保持基础问答",
            3: "重度降级: 仅支持基础问答, 无工具调用",
            4: "最低可用: 返回静态响应或转人工",
            5: "完全不可用: 系统无法提供服务",
        }
        return descs.get(self.value, "未知级别")


@dataclass
class DegradationPolicy:
    """降级策略配置"""
    enabled_tools: list[str]               # 当前可用的工具列表
    llm_model: str                          # 当前使用的模型
    use_cache_only: bool = False            # 是否仅使用缓存
    max_tokens: int = 2048                  # 输出 Token 限制
    enable_reasoning: bool = True           # 是否启用推理
    enable_memory: bool = True              # 是否启用记忆
    request_human: bool = False             # 是否需要人工介入


class DegradationManager:
    """
    降级管理器: 根据系统状态动态调整降级级别。
    """

    def __init__(self):
        self.current_level: DegradationLevel = DegradationLevel.FULL
        self.level_history: list[tuple[float, DegradationLevel, str]] = []
        self.policy_cache: dict[DegradationLevel, DegradationPolicy] = {
            DegradationLevel.FULL: DegradationPolicy(
                enabled_tools=["*"],  # 所有工具
                llm_model="gpt-4-turbo",
                use_cache_only=False,
                max_tokens=4096,
                enable_reasoning=True,
                enable_memory=True,
                request_human=False,
            ),
            DegradationLevel.DEGRADE_1: DegradationPolicy(
                enabled_tools=["*"],
                llm_model="gpt-4o-mini",  # 换成小模型
                use_cache_only=False,
                max_tokens=2048,
                enable_reasoning=True,
                enable_memory=True,
                request_human=False,
            ),
            DegradationLevel.DEGRADE_2: DegradationPolicy(
                enabled_tools=[
                    "search", "read", "calculate"  # 只保留读工具
                ],
                llm_model="gpt-4o-mini",
                use_cache_only=False,
                max_tokens=1024,
                enable_reasoning=True,
                enable_memory=True,
                request_human=False,
            ),
            DegradationLevel.DEGRADE_3: DegradationPolicy(
                enabled_tools=[],  # 禁用所有工具
                llm_model="gpt-4o-mini",
                use_cache_only=True,  # 仅使用缓存
                max_tokens=512,
                enable_reasoning=False,  # 禁用推理
                enable_memory=False,
                request_human=False,
            ),
            DegradationLevel.MINIMAL: DegradationPolicy(
                enabled_tools=[],
                llm_model="gpt-4o-mini",
                use_cache_only=True,
                max_tokens=256,
                enable_reasoning=False,
                enable_memory=False,
                request_human=True,  # 请求人工介入
            ),
        }

    def get_current_policy(self) -> DegradationPolicy:
        """获取当前级别的策略"""
        return self.policy_cache.get(self.current_level, self.policy_cache[DegradationLevel.MINIMAL])

    def degrade(self, reason: str, target_level: Optional[DegradationLevel] = None):
        """降级到指定级别或自动降一级"""
        if target_level:
            new_level = target_level
        else:
            new_level = DegradationLevel(min(self.current_level + 1, 5))

        if new_level > self.current_level:
            self.current_level = new_level
            self.level_history.append((time.time(), new_level, reason))

    def recover(self, target_level: Optional[DegradationLevel] = None):
        """恢复到指定级别或升一级"""
        if target_level:
            new_level = target_level
        else:
            new_level = DegradationLevel(max(self.current_level - 1, 0))

        if new_level < self.current_level:
            self.current_level = new_level
            self.level_history.append((time.time(), new_level, "recovery"))
```

### 3.2 能力感知的 Agent 降级

降级不只是"禁用功能"，更需要让 Agent 意识到自己当前的能力范围，并据此调整行为：

```python
class DegradationAwareAgent:
    """
    降级感知的 Agent。
    能够根据当前降级级别调整 Prompt 和行为。
    """

    def __init__(self, degradation_mgr: DegradationManager):
        self.degradation = degradation_mgr

    def build_system_prompt(self, base_prompt: str) -> str:
        """根据当前降级级别, 注入能力限制指令"""
        policy = self.degradation.get_current_policy()

        degradation_instruction = self._get_degradation_prompt(policy)
        return f"{base_prompt}\n\n{degradation_instruction}"

    def _get_degradation_prompt(self, policy: DegradationPolicy) -> str:
        """生成降级指令注入到 System Prompt"""
        parts = [
            "[SYSTEM CAPABILITY NOTICE]",
            f"Current mode: {self.degradation.current_level.name}",
        ]

        if not policy.enable_reasoning:
            parts.append("- Complex reasoning is disabled. Provide direct answers.")

        if policy.use_cache_only:
            parts.append(
                "- You can ONLY use cached/pre-existing knowledge. "
                "You CANNOT make new tool calls."
            )

        if not policy.enabled_tools:
            parts.append(
                "- Tool calling is currently disabled. "
                "Answer based on your existing knowledge only."
            )
        else:
            active_tools = policy.enabled_tools
            if active_tools != ["*"]:
                tools_str = ", ".join(active_tools)
                parts.append(f"- Only the following tools are available: {tools_str}")

        if policy.request_human:
            parts.append(
                "- Your capabilities are limited. "
                "If the request requires actions beyond your current scope, "
                "clearly state your limitations and suggest the user contact support."
            )

        parts.append(
            "ACKNOWLEDGE this notice. Do not attempt actions outside your current capabilities."
        )

        return "\n".join(parts)

    def validate_action(self, tool_name: str, arguments: dict) -> tuple[bool, str]:
        """验证工具调用是否在当前降级策略允许范围内"""
        policy = self.degradation.get_current_policy()

        if policy.enabled_tools == ["*"]:
            return True, ""

        if not policy.enabled_tools:
            return False, f"Tool '{tool_name}' is disabled in current degradation mode"

        if tool_name not in policy.enabled_tools:
            return False, (
                f"Tool '{tool_name}' is not available in current mode. "
                f"Available: {policy.enabled_tools}"
            )

        return True, ""
```

### 3.3 多级降级链

预定义的降级链决定了降级发生时如何逐步退化：

```python
class DegradationChain:
    """
    降级链: 定义当某个能力不可用时的逐级回退路径。
    """

    def __init__(self, llm_client, cache_client):
        self.llm = llm_client
        self.cache = cache_client

    async def execute_with_degradation(
        self,
        request: str,
        context: dict,
    ) -> dict:
        """
        执行带多级降级的 Agent 请求。
        逐步尝试从最高级降级到最低级。
        """

        # Level 0: 全功能模式
        result = await self._try_full_capability(request, context)
        if result["success"]:
            return result

        # Level 1: 换小模型 + 缓存优先
        result = await self._try_small_model_with_cache(request, context)
        if result["success"]:
            return {**result, "degradation_level": 1}

        # Level 2: 禁用工具, 仅知识回答
        result = await self._try_knowledge_only(request, context)
        if result["success"]:
            return {**result, "degradation_level": 2}

        # Level 3: 返回缓存/模板响应
        result = await self._try_cached_response(request)
        if result["success"]:
            return {**result, "degradation_level": 3}

        # Level 4: 返回静态提示 + 转人工
        return {
            "success": True,
            "content": self._minimal_response(request),
            "degradation_level": 4,
            "human_escalation": True,
        }

    async def _try_full_capability(
        self, request: str, context: dict
    ) -> dict:
        """尝试全功能模式"""
        try:
            response = await self.llm.complete(
                messages=self._build_messages(request, context),
                tools=self._all_tools(),
                model="gpt-4-turbo",
            )
            return {"success": True, "content": response, "mode": "full"}
        except Exception as e:
            return {"success": False, "error": str(e)}

    async def _try_small_model_with_cache(
        self, request: str, context: dict
    ) -> dict:
        """降级: 小模型 + 优先查缓存"""
        # 先查缓存
        cached = await self.cache.get(request)
        if cached:
            return {"success": True, "content": cached, "mode": "cache"}

        try:
            response = await self.llm.complete(
                messages=self._build_messages(request, context),
                tools=self._non_mutating_tools(),  # 只允许读工具
                model="gpt-4o-mini",
            )
            return {"success": True, "content": response, "mode": "small_model"}
        except Exception as e:
            return {"success": False, "error": str(e)}

    async def _try_knowledge_only(self, request: str, context: dict) -> dict:
        """降级: 禁用工具, 仅靠模型知识"""
        try:
            response = await self.llm.complete(
                messages=self._build_messages(request, context, no_tools=True),
                tools=[],  # 不传工具
                model="gpt-4o-mini",
                max_tokens=1024,
            )
            return {"success": True, "content": response, "mode": "knowledge_only"}
        except Exception as e:
            return {"success": False, "error": str(e)}

    async def _try_cached_response(self, request: str) -> dict:
        """重度降级: 返回缓存或模板响应"""
        # 语义缓存搜索
        similar = await self.cache.semantic_search(request, threshold=0.85)
        if similar:
            return {"success": True, "content": similar, "mode": "semantic_cache"}

        return {"success": False}

    def _minimal_response(self, request: str) -> str:
        """最低可用: 返回标准的不可用响应"""
        return (
            "I apologize, but I'm currently experiencing limited availability. "
            "I could not process your request in real-time. "
            "Please try again later, or contact our support team for assistance."
        )

    def _all_tools(self) -> list[dict]:
        return [...]

    def _non_mutating_tools(self) -> list[dict]:
        """只返回读操作工具"""
        return [t for t in self._all_tools() if t.get("type") == "read"]

    def _build_messages(self, request, context, no_tools=False):
        return [...]
```

### 3.4 降级场景实战

#### 场景 1: LLM API 限流/不可用

```
触发: API 返回 429 或 5xx, 重试耗尽后熔断开启

降级链:
  1. 语义缓存查询 (可能最近有人问过类似问题)
  2. 切到备用 Provider (如 OpenAI → Anthropic)
  3. 降级到小模型 (减小负载)
  4. 返回知识库缓存结果 (可能不是最新的)
  5. 返回"暂时不可用" + 转人工
```

#### 场景 2: 工具调用持续失败

```
触发: 外部工具 API 不可用 (如天气 API、数据库)

降级链:
  1. 重试 2-3 次 (可能是瞬态故障)
  2. 使用缓存的结果 (如果有)
  3. 跳过该工具, 使用模型自身知识回答
  4. 告知用户 "我无法获取实时数据, 以下是基于已有知识的回答"
  5. 如果任务是写操作 (发邮件/创建工单), 排入重试队列稍后处理
```

#### 场景 3: 系统负载过高

```
触发: CPU > 80%, 队列深度 > 阈值, 延迟 P99 > 5s

降级链:
  1. 新请求直接使用小模型处理
  2. 禁用非核心工具 (分析/建议类工具)
  3. 启用请求排队 + 异步响应
  4. 返回缓存结果 (不启动新的 LLM 调用)
  5. 返回 "系统繁忙" + 建议稍后重试
```

### 3.5 降级状态持久化

降级状态需要在实例间共享，确保同一用户的多个请求体验一致：

```python
import json
import aioredis

class PersistentDegradationState:
    """
    持久化降级状态 (Redis 后端)。
    确保整个集群的降级状态一致。
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self.key = "agent:degradation:global"

    async def get_global_level(self) -> DegradationLevel:
        data = await self.redis.get(self.key)
        if data:
            return DegradationLevel(int(data))
        return DegradationLevel.FULL

    async def set_global_level(
        self, level: DegradationLevel, reason: str, ttl: int = 300
    ):
        await self.redis.set(self.key, level.value, ex=ttl)
        # 同时记录降级事件
        event = {
            "level": level.value,
            "reason": reason,
            "timestamp": time.time(),
        }
        await self.redis.rpush("agent:degradation:events", json.dumps(event))

    async def get_degradation_history(
        self, count: int = 50
    ) -> list[dict]:
        events = await self.redis.lrange(
            "agent:degradation:events", -count, -1
        )
        return [json.loads(e) for e in events]
```

### 3.6 降级指标与监控

```python
class DegradationMetrics:
    """
    降级指标收集。
    """

    def __init__(self):
        self.metrics = {
            "total_requests": 0,
            "degraded_responses": 0,
            "level_counts": {level: 0 for level in DegradationLevel},
            "degradation_reasons": {},
            "recovery_count": 0,
        }

    def record_decision(self, level: DegradationLevel, reason: str):
        self.metrics["total_requests"] += 1
        if level > DegradationLevel.FULL:
            self.metrics["degraded_responses"] += 1
            self.metrics["level_counts"][level] = \
                self.metrics["level_counts"].get(level, 0) + 1
            self.metrics["degradation_reasons"][reason] = \
                self.metrics["degradation_reasons"].get(reason, 0) + 1
        else:
            self.metrics["recovery_count"] += 1

    def degradation_rate(self) -> float:
        """降级率: 降级响应占比"""
        if self.metrics["total_requests"] == 0:
            return 0.0
        return self.metrics["degraded_responses"] / self.metrics["total_requests"]

    def report(self) -> dict:
        return {
            "degradation_rate": self.degradation_rate(),
            "current_level": self.metrics["level_counts"],
            "top_reasons": sorted(
                self.metrics["degradation_reasons"].items(),
                key=lambda x: x[1],
                reverse=True,
            )[:5],
            "recovery_count": self.metrics["recovery_count"],
        }
```

---

## 4. 能力边界

### 4.1 降级的局限

```
降级策略做不到:
  ┌─ 无法为写操作 (转账、发邮件) 提供有意义的降级
  │  缓存或近似回答对写操作毫无意义
  ├─ 无法解决所有"降级体验"问题
  │  降级必然降低用户体验, 好的降级只是让体验下降得更平缓
  ├─ 无法避免用户感知
  │  严格的降级可能被用户察觉 (回答变短、工具功能消失)
  └─ 无法处理所有边缘案例
      某些任务在降级模式下可能给出错误但看起来合理的回答
```

### 4.2 降级场景对照

| 降级类型 | 适用场景 | 不适用场景 | 用户体验影响 |
|---------|---------|-----------|------------|
| 缓存替代 | 知识查询、静态信息 | 实时数据、个性化内容 | 数据可能过时 |
| 小模型替代 | 简单问答、文本处理 | 复杂推理、代码生成 | 质量下降 5-15% |
| 禁用写工具 | 分析 Agent | 自动化 Agent | 功能受限但安全 |
| 仅知识回答 | 事实性问答 | 依赖外部数据的任务 | 回答范围受限 |
| 异步延迟处理 | 非实时任务 | 实时交互 | 延迟增加 |
| 人工接管 | 复杂/关键任务 | 简单高频任务 | 成本增加 |

### 4.3 降级的安全风险

```
降级引入的安全问题:

  ┌─ 绕过安全控制: 降级可能禁用安全验证工具
  │  解决方案: 安全相关的工具永不降级
  │
  ├─ 过度降级: 持续处于降级状态, "新常态"
  │  解决方案: 降级自动恢复机制 + 人工干预触发
  │
  └─ 用户欺骗: Agent 降级后仍声称"已处理"
      解决方案: 降级时明确告知用户当前能力范围
```

---

## 5. 与其他模式对比

| 模式 | 解决的问题 | 降级的关系 |
|------|-----------|-----------|
| Retry | 尝试修复瞬态故障 | 降级前置: 重试失败后才触发降级 |
| Circuit Breaker | 快速拒绝防止级联 | 降级触发源: 熔断开启时降级 |
| Bulkhead | 隔离故障避免影响全局 | 降级辅助: 隔舱耗尽时触发该舱降级 |
| Timeout | 防止无限等待 | 降级触发源: 超时后降级 |
| Fallback | 提供替代方案 | 降级的一种具体实现 |

---

## 6. 工程优化方向

### 6.1 渐进式降级策略

推荐从最低级别的降级开始，逐步上升：

```
实施步骤:

  1. 建立基准: 确定哪些功能是"核心" (不可降级) 和"非核心" (可降级)
  2. 定义降级链: 每类故障的逐级回退路径
  3. 实现 Level 1: 缓存 + 小模型 (ROI 最高的降级)
  4. 实现 Level 2: 禁用非核心工具
  5. 实现 Level 3: 仅知识回答 + 禁用推理
  6. 实现 Level 4-5: 人工接管 + 不可用响应
  7. 添加自动升降级: 基于监控指标自动调整降级级别
```

### 6.2 降级测试

```
降级策略测试清单:

  [ ] 1. 每个降级级别是否产生有意义的结果?
  [ ] 2. 降级是否在触发条件满足时自动生效?
  [ ] 3. 降级是否在故障恢复后自动回退?
  [ ] 4. 降级状态是否在集群中一致?
  [ ] 5. 所有安全校验是否在降级模式下依然生效?
  [ ] 6. 用户是否能感知到降级 (信息透明)?
  [ ] 7. 降级路径是否存在死循环 (降级→恢复→再降级→再恢复)?
  [ ] 8. 人工干预的升级路径是否畅通?
```

### 6.3 降级策略设计原则

```
四条黄金原则:

  1. 核心功能永不降级
     安全校验、数据保护、身份认证在任何降级级别下都必须正常运行

  2. 降级必须透明
     用户应当知道服务正在降级运行, 而不是"看起来正常但实际降级"

  3. 降级必须有恢复路径
     没有自动恢复机制的降级最终会成为永久降级

  4. 降级是业务决策, 不只是技术决策
     哪些功能可降级、降级到什么程度, 需要产品和业务团队的输入
```

---

> **总结**：降级策略是 Agent 系统可靠性的最后一道防线。当其他机制（重试、熔断）都失效时，降级决定了系统"失败的方式"。好的降级策略应定义 4-5 个降级级别，涵盖缓存替代、小模型降级、禁用工具、仅知识回答、转人工等不同粒度的降级方案，并确保 Agent 能够感知自身的能力边界并据此调整行为。核心原则是"核心功能永不降级、降级必须透明、必须有恢复路径"。
