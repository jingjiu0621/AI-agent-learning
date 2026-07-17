# 13.3.4 cache-invalidation -- 失效策略：主动 / 被动 / 事件驱动

**核心论点：缓存失效是计算机科学中最难的两件事之一 -- 对于 Agent 系统，这个问题更加棘手：LLM 的知识不是静态的，工具的返回状态是实时变化的，用户的上下文是动态演进的。一个过时的缓存响应可能导致 Agent 做出灾难性决策，而过于激进的失效策略又会让缓存形同虚设。**

在多层缓存架构（13.3.3）中，我们讨论了如何高效地存储和读取缓存数据。但"如何让缓存变 stale"同样重要。对于 AI Agent，缓存失效的决策直接影响响应质量：用过期数据做出的推理可能产生错误的工具调用、遗漏关键信息、甚至输出幻觉内容。本节深入探讨 Agent 系统中的三种缓存失效模式及其工程实现。

---

## 1. 三种失效模式详解

### 1.1 主动失效（TTL-based）

主动失效是最简单、最广泛使用的策略。每条缓存数据设定一个生存时间（TTL），到期后自动失效。

**工作原理**：

```
写入缓存时的决策：
  set(key, value, ttl=3600)
  // 缓存条目将在 3600 秒后自动失效

读取缓存时的检查：
  get(key):
    if entry.expires_at < now:
      delete(key)
      return None
    return entry.value
```

**TTL 的选择依据**：

| 数据类型 | 推荐 TTL | 依据 |
|----------|----------|------|
| LLM 补全结果（通用知识）| 24h ~ 7d | 知识变化慢 |
| 工具调用结果（股票价格）| 30s ~ 5min | 实时性要求高 |
| 用户会话上下文 | 15min ~ 1h | 会话生命周期 |
| Embedding 向量 | 7d ~ 30d | 计算成本高，变化慢 |
| Prompt 模板 | 永久（显式失效） | 仅代码部署时变化 |

**固定 TTL 的问题**：

固定 TTL 无法适应数据访问模式和底层数据变化频率的动态性。典型问题包括：

1. **窗口期不一致**：一个 TTL=3600s 的缓存条目前 59 分钟都是最新的，但第 59 分 30 秒时底层数据变更了，剩下的 30 秒内 Agent 将使用过期数据
2. **缓存雪崩**：大量缓存条目在同一时间过期，导致 LLM 瞬间被大量请求淹没
3. **资源浪费**：低频访问的数据也在缓存中占用空间直到 TTL 到期

**自适应 TTL 的实现**：

```python
import time
from collections import defaultdict

class AdaptiveTTLStrategy:
    """自适应 TTL：基于访问频率、错误率和数据变化率动态调整 TTL"""

    def __init__(self, base_ttl: float = 3600.0):
        self.base_ttl = base_ttl
        self.access_count: dict[str, int] = defaultdict(int)
        self.error_count: dict[str, int] = defaultdict(int)
        self.last_access: dict[str, float] = {}
        self.ttl_overrides: dict[str, float] = {}

    def record_access(self, key: str):
        self.access_count[key] += 1
        self.last_access[key] = time.time()

    def record_error(self, key: str):
        """记录由该缓存条目导致的错误"""
        self.error_count[key] += 1

    def compute_ttl(self, key: str) -> float:
        # 基于访问频率：高频访问的数据 TTL 更长（更有价值）
        freq = self.access_count.get(key, 0)
        freq_bonus = min(freq * 0.1, 2.0)  # 最多延长 2 倍

        # 基于错误率：产生错误的缓存应该更快过期
        total = self.access_count.get(key, 1)
        err_rate = self.error_count.get(key, 0) / max(total, 1)
        err_penalty = 1.0 - min(err_rate * 10, 0.9)  # 错误率越高，TTL 缩短越多

        # 基于时间衰减：长时间未访问的数据 TTL 缩短
        last = self.last_access.get(key, 0)
        time_since_access = time.time() - last
        decay = 1.0 / max(1.0, time_since_access / self.base_ttl)

        # 组合计算
        ttl = self.base_ttl * freq_bonus * err_penalty * decay
        return max(ttl, 60.0)  # 最小 TTL 60 秒

    def get_ttl(self, key: str) -> float:
        return self.ttl_overrides.get(key, self.compute_ttl(key))
```

**自适应 TTL 的演进：Probabilistic TTL**

更高级的做法是引入概率性过期。每条缓存不设固定的过期时间，而是在访问时根据概率决定是否将其视为过期：

```python
import random
import math

class ProbabilisticTTL:
    """概率性过期：每个缓存条目有概率提前过期，防止缓存雪崩"""

    def __init__(self, base_ttl: float, beta: float = 1.0):
        self.base_ttl = base_ttl
        self.beta = beta  # 控制概率曲线的陡峭程度

    def should_expire(self, age: float) -> bool:
        """根据缓存已存在的时间，决定是否概率性过期

        概率公式: P(expire) = 1 - exp(-beta * age / base_ttl)
        当 age == base_ttl 时，过期概率约为 63%
        当 age == 2 * base_ttl 时，过期概率约为 86%
        """
        if age < self.base_ttl * 0.5:
            return False  # 半衰期内永不概率过期

        probability = 1.0 - math.exp(-self.beta * age / self.base_ttl)
        return random.random() < probability
```

这种策略的关键优势：大量 key 不在同一时刻过期，而是分散在时间轴上，天然防止缓存雪崩。

---

### 1.2 被动失效（Event-driven）

被动失效是主动失效的补充 -- 它不是在时间维度上猜测何时数据过期，而是当底层数据真正变化时立即通知缓存层。

**架构**：

```
  ┌─────────────┐     ┌──────────────┐     ┌──────────────┐
  │ Data Source │────►│  Event Bus   │────►│Cache Updater │
  │ (DB Change) │     │  (Kafka/Rabbit)   │ (Subscriber) │
  └─────────────┘     └──────────────┘     └──────┬───────┘
                                                   │
                                                   ▼
                                           ┌──────────────┐
                                           │  L1 / L2 / L3│
                                           │  Invalidate  │
                                           └──────────────┘
```

**Agent 中的典型事件源**：

| 事件源 | 事件类型 | 影响的缓存 |
|--------|----------|------------|
| 知识库更新 | `document.updated` | RAG 检索结果、Embedding 缓存 |
| 外部 API 变更 | `api.schema_changed` | 工具调用 schema 缓存 |
| 模型版本升级 | `model.deployed` | 所有 LLM 补全缓存（全量失效）|
| Prompt 策略更新 | `prompt.updated` | 特定 namespace 缓存 |
| 用户数据变更 | `user.profile_updated` | 用户上下文缓存 |

**事件驱动失效的实现**：

```python
import asyncio
import json
from typing import Protocol

class CacheInvalidationEvent:
    """缓存失效事件的统一格式"""

    def __init__(
        self,
        event_type: str,
        namespace: str,
        keys: list[str] | None = None,
        pattern: str | None = None,
        cascade: bool = False,
    ):
        self.event_type = event_type        # e.g., "document.updated"
        self.namespace = namespace           # e.g., "rag_embeddings"
        self.keys = keys                     # 指定失效的具体 key
        self.pattern = pattern               # glob 模式，如 "session:user_*"
        self.cascade = cascade               # 是否级联失效
        self.timestamp = asyncio.get_event_loop().time()


class EventBus:
    """简单的事件总线实现（生产环境推荐 Kafka / RabbitMQ）"""

    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = defaultdict(list)

    def subscribe(self, event_type: str, callback: Callable):
        self._subscribers[event_type].append(callback)

    async def publish(self, event: CacheInvalidationEvent):
        callbacks = self._subscribers.get(event.event_type, [])
        results = await asyncio.gather(
            *[cb(event) for cb in callbacks],
            return_exceptions=True,
        )
        for result in results:
            if isinstance(result, Exception):
                logger.error(f"Event handler failed: {result}")


class EventDrivenInvalidator:
    """事件驱动的缓存失效器"""

    def __init__(
        self,
        cache: "ProductionMultiLayerCache",
        event_bus: EventBus,
    ):
        self._cache = cache
        self._event_bus = event_bus
        self._register_handlers()

    def _register_handlers(self):
        self._event_bus.subscribe("document.updated", self._handle_doc_update)
        self._event_bus.subscribe("api.schema_changed", self._handle_api_schema)
        self._event_bus.subscribe("model.deployed", self._handle_model_deploy)
        self._event_bus.subscribe("prompt.updated", self._handle_prompt_update)

    async def _handle_doc_update(self, event: CacheInvalidationEvent):
        """知识库文档更新：使相关的 RAG 缓存失效"""
        if event.keys:
            tasks = [
                self._cache.invalidate(event.namespace, key)
                for key in event.keys
            ]
            await asyncio.gather(*tasks)
        elif event.pattern:
            await self._cache.invalidate_by_pattern(event.namespace, event.pattern)

    async def _handle_model_deploy(self, event: CacheInvalidationEvent):
        """模型版本升级：全量缓存失效（谨慎使用！）"""
        logger.warning("Model version upgraded: performing full cache invalidation")
        # 全量失效不是简单 delete all，而是版本号递增
        await self._cache.bump_version(event.namespace)

    async def invalidate_on_event(
        self,
        namespace: str,
        keys: list[str],
    ):
        event = CacheInvalidationEvent(
            event_type="custom.invalidate",
            namespace=namespace,
            keys=keys,
        )
        await self._event_bus.publish(event)
```

---

### 1.3 主动验证（Lazy Validation）

对于无法预测变化时间、也无法接入事件通知的数据源，主动验证是一种轻量级的妥协方案。

**核心思想**：缓存命中后不直接返回，而是异步验证其有效性。如果验证失败，重新获取并更新缓存。

```
  Request ──► Cache Hit ──► 返回结果（立即）
                              │
                              ▼ (异步)
                    ┌──────────────────┐
                    │  Validate Cache  │
                    └────────┬─────────┘
                             │
                    ┌────────┴────────┐
                    │  Still valid?   │
                    └────────┬────────┘
                        Yes /    \ No
                         │       │
                    无操作     触发刷新
                              ┌────────┐
                              │ 异步回填│
                              └────────┘
```

**验证方式对比**：

| 验证方式 | 开销 | 保证强度 | 适用场景 |
|----------|------|----------|----------|
| Lightweight checksum | 低 | 中 | 检测数据是否被修改 |
| HEAD request (HTTP) | 低 | 高 | 外部 API 资源 |
| Conditional GET (ETag) | 中 | 高 | REST API 响应缓存 |
| Full content comparison | 高 | 最高 | 关键数据（如金融信息）|
| TTL + probabilistic check | 极低 | 低 | 通用场景 |

**实现示例**：

```python
import asyncio
import hashlib

class LazyValidationCache:
    """主动验证缓存：命中后异步验证"""

    def __init__(self, cache: MultiLayerCache):
        self._cache = cache
        self._validation_tasks: dict[str, asyncio.Task] = {}

    async def get_with_validation(
        self,
        namespace: str,
        key: str,
        validator: Callable[[dict], bool],
        refresher: Callable[[], dict],
    ) -> dict:
        # 同步返回缓存结果（即使可能过期）
        cached = await self._cache.get(namespace, key)

        if cached is None:
            # 完全未命中：同步获取新数据
            value = await refresher()
            await self._cache.set(namespace, key, value)
            return value

        # 缓存命中：立即返回，同时异步验证
        validation_key = f"{namespace}:{key}"

        # 避免对同一 key 重复验证
        if validation_key not in self._validation_tasks:
            task = asyncio.create_task(
                self._validate_and_refresh(
                    namespace, key, cached, validator, refresher
                )
            )
            self._validation_tasks[validation_key] = task
            task.add_done_callback(lambda _: self._validation_tasks.pop(validation_key, None))

        return cached

    async def _validate_and_refresh(
        self,
        namespace: str,
        key: str,
        cached_value: dict,
        validator: Callable[[dict], bool],
        refresher: Callable[[], dict],
    ):
        try:
            if not validator(cached_value):
                logger.info(f"Cache validation failed for {namespace}:{key}, refreshing...")
                fresh_value = await refresher()
                await self._cache.set(namespace, key, fresh_value)
        except Exception as e:
            logger.error(f"Validation error for {namespace}:{key}: {e}")

    async def lightweight_checksum_validator(self, value: dict) -> bool:
        """轻量级校验和验证示例"""
        checksum_endpoint = value.get("_checksum_url")
        if not checksum_endpoint:
            return True  # 无校验和，视为有效

        # 发起轻量 HTTP HEAD 请求获取最新 ETag
        async with aiohttp.ClientSession() as session:
            async with session.head(checksum_endpoint) as response:
                current_etag = response.headers.get("ETag")
                stored_etag = value.get("_etag")
                return current_etag == stored_etag
```

---

## 2. Agent 特有失效场景

AI Agent 的缓存失效比传统 Web 应用复杂得多，因为缓存的依赖维度更多、变化速度更快。

### 2.1 上下文失效

Agent 在多轮对话中，用户可能随时提供新的上下文信息，导致之前缓存的推理结果失效。

```
轮次 1: 用户 "帮我查一下北京的天气"
        缓存: get_weather("北京") = "晴天 25°C"

轮次 2: 用户 "等一下，我说的是上海"
        上下文发生变化 -> 轮次 1 的缓存结果不再适用
        需要: 基于新上下文重新推理
```

**实现**：缓存 key 必须包含上下文指纹（Context Fingerprint）

```python
class ContextAwareCacheKey:
    """上下文感知的缓存 key 生成器"""

    def __init__(self):
        pass

    def make_key(self, base_key: str, context: dict) -> str:
        """将上下文的敏感部分编码到 key 中

        例如: "get_weather" + context_hash(["北京", "2026-07-17"])
        -> "get_weather:loc=beijing:date=2026-07-17:ctx=a1b2c3"
        """
        # 提取上下文中与当前任务相关的部分
        relevant_context = self._extract_relevant_context(base_key, context)

        # 序列化上下文指纹
        ctx_fingerprint = hashlib.sha256(
            json.dumps(relevant_context, sort_keys=True).encode()
        ).hexdigest()[:12]

        return f"{base_key}:ctx={ctx_fingerprint}"

    def _extract_relevant_context(self, task: str, context: dict) -> dict:
        """基于任务类型提取相关上下文"""
        task_type = self._classify_task(task)

        if task_type == "weather":
            return {"location": context.get("location")}
        elif task_type == "recommendation":
            return {
                "user_preferences": context.get("preferences"),
                "history": context.get("recent_items", [])[-5:],
            }
        # 其他任务类型...
        return context

    def _classify_task(self, task: str) -> str:
        if "weather" in task.lower():
            return "weather"
        if "recommend" in task.lower():
            return "recommendation"
        return "generic"
```

### 2.2 工具状态失效

Agent 调用的外部工具（API、数据库、搜索引擎）的返回值可能实时变化。缓存必须在工具状态变更时及时失效。

```python
class ToolStateTracker:
    """跟踪工具调用的状态变更"""

    def __init__(self, event_bus: EventBus):
        self._tool_versions: dict[str, str] = {}
        self._event_bus = event_bus

    async def on_tool_call(self, tool_name: str, params: dict, result: dict):
        """记录工具调用结果和版本信息"""
        version_key = f"{tool_name}:{json.dumps(params, sort_keys=True)}"

        # 从结果中提取版本信息（如果 API 返回了版本号）
        result_version = result.get("_version", result.get("etag", ""))

        if version_key in self._tool_versions:
            previous_version = self._tool_versions[version_key]
            if previous_version != result_version:
                # 工具返回了不同版本 -> 旧的缓存失效
                await self._event_bus.publish(CacheInvalidationEvent(
                    event_type="tool.state_changed",
                    namespace="tool_results",
                    keys=[version_key],
                ))

        self._tool_versions[version_key] = result_version
```

### 2.3 时效性失效

某些数据具有天然的时间敏感性，与 TTL 不同，这种失效发生在 Agent 感知到"时间维度变化"时。

```python
class TemporalInvalidation:
    """时间敏感数据的失效策略"""

    def __init__(self, cache: MultiLayerCache):
        self._cache = cache

    async def get_time_sensitive(
        self,
        data_type: str,
        key: str,
        query_time: float,
    ):
        """时间敏感数据：检查缓存的时间戳是否满足查询需求"""
        cached = await self._cache.get(data_type, key)
        if cached is None:
            return None

        cached_time = cached.get("_timestamp", 0)
        max_age = self._max_age_for(data_type)

        if query_time - cached_time > max_age:
            # 数据虽然未到 TTL，但已经不满足查询的时间要求
            logger.debug(f"Temporal invalidation: {data_type}:{key} is too old")
            await self._cache.invalidate(data_type, key)
            return None

        return cached

    def _max_age_for(self, data_type: str) -> float:
        ages = {
            "stock_price": 60,         # 60 秒
            "weather_forecast": 1800,   # 30 分钟
            "news_headlines": 300,      # 5 分钟
            "exchange_rate": 120,       # 2 分钟
        }
        return ages.get(data_type, 3600)
```

### 2.4 版本失效

当 Agent 系统的配置发生变化时（Prompt 更新、模型切换、策略调整），所有基于旧版本的缓存可能需要批量失效。

```python
class VersionedCache:
    """版本感知的缓存：版本号编码在缓存 key 中"""

    def __init__(self, cache: MultiLayerCache, namespace: str):
        self._cache = cache
        self._namespace = namespace
        # 版本号可以存储在 Redis / 配置中心
        self._current_version: str = "v1"
        self._version_subscribers: list[Callable] = []

    def _versioned_key(self, base_key: str) -> str:
        return f"{self._current_version}:{base_key}"

    async def get(self, base_key: str):
        return await self._cache.get(self._namespace, self._versioned_key(base_key))

    async def set(self, base_key: str, value: dict, ttl: int | None = None):
        return await self._cache.set(
            self._namespace, self._versioned_key(base_key), value, ttl
        )

    async def bump_version(self):
        """版本升级：将当前版本的过期时间缩短，切换到新版本"""
        old_version = self._current_version
        self._current_version = f"v{int(old_version[1:]) + 1}"
        logger.info(f"Cache version bumped: {old_version} -> {self._current_version}")

        # 通知订阅者
        for cb in self._version_subscribers:
            await cb(old_version, self._current_version)

        # 旧版本的缓存不会被立即删除，而是在访问时自然过期
        # 这样可以避免全量失效导致缓存雪崩
```

---

## 3. 失效策略决策树

选择缓存失效策略时，可以从以下三个维度进行决策：

```
                   ┌────────────────────────────────────┐
                   │       开始：这是什么数据？          │
                   └────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
              数据变化可预测？             数据变化不可预测？
                    │                               │
              ┌─────┴─────┐                  ┌──────┴──────┐
              │           │                  │              │
         变化频率固定？ 变化频率波动？    有事件通知？   无事件通知？
              │           │                  │              │
          ┌───┴───┐   ┌──┴───┐          ┌───┴───┐      ┌──┴───┐
          │       │   │      │          │       │      │      │
       固定TTL  自适应 事件驱动  事件驱动  事件驱动  主动验证
       (简单)   TTL   + TTL    + 自适应  + TTL    (懒验证)
                         (动态)            (兜底)     + TTL
                                                    (兜底)

  ┌──────────────────────────────────────────────────────────────┐
  │                     成本 / 收益评估                          │
  │                                                              │
  │  更新成本高（如 LLM 结果）-> 倾向长 TTL + 被动失效           │
  │  更新成本低（如简单 API）  -> 倾向短 TTL + 懒验证            │
  │  Stale 数据危害大           -> 倾向事件驱动 + 实时校验        │
  │  Stale 数据危害小           -> 倾向长 TTL + 概率性过期        │
  └──────────────────────────────────────────────────────────────┘
```

---

## 4. 级联失效

当一个缓存条目失效时，依赖于它的其他缓存条目也需要相应地失效。这在 Agent 系统中尤为常见。

### 依赖图示例

```
  tool_schema("search_web")
         │
         ├── tool_result("search_web", query="AI trends")
         │         │
         │         └── llm_completion("总结搜索: AI trends")
         │
         └── tool_result("search_web", query="ML news")
                   │
                   └── llm_completion("总结搜索: ML news")

  如果 tool_schema 变化 -> 所有依赖它的缓存条目都需要级联失效
```

### 实现

```python
class CascadingInvalidation:
    """级联缓存失效管理器"""

    def __init__(self):
        # 依赖图: parent_key -> set[child_key]
        self._dependency_graph: dict[str, set[str]] = {}
        # 反向依赖: child_key -> set[parent_key]
        self._reverse_deps: dict[str, set[str]] = {}

    def register_dependency(self, parent_key: str, child_key: str):
        """注册依赖关系：child 依赖于 parent"""
        if parent_key not in self._dependency_graph:
            self._dependency_graph[parent_key] = set()
        self._dependency_graph[parent_key].add(child_key)

        if child_key not in self._reverse_deps:
            self._reverse_deps[child_key] = set()
        self._reverse_deps[child_key].add(parent_key)

    async def invalidate_cascade(
        self,
        cache: MultiLayerCache,
        namespace: str,
        root_key: str,
        depth: int = 0,
        max_depth: int = 10,
    ):
        """级联失效：失效 root_key 及其所有下游依赖"""
        if depth > max_depth:
            logger.warning(f"Cascade invalidation exceeded max depth at {root_key}")
            return

        # 1. 失效当前 key
        await cache.invalidate(namespace, root_key)
        logger.debug(f"[depth={depth}] Invalidated {namespace}:{root_key}")

        # 2. 找到所有子依赖，递归失效
        children = self._dependency_graph.get(root_key, set())
        tasks = [
            self.invalidate_cascade(cache, namespace, child, depth + 1, max_depth)
            for child in children
        ]
        await asyncio.gather(*tasks)

    async def on_parent_invalidated(self, parent_key: str):
        """当外部通知某个 parent 失效时：触发级联"""
        logger.info(f"Parent {parent_key} invalidated, cascading...")
        # 这个函数由事件驱动失效器调用
        pass
```

---

## 5. Stale Cache 的优雅处理

在某些场景中，完全避免 stale 数据是不可能的或成本过高的。这种情况下，"优雅降级"比"强一致"更务实。

### 5.1 Stale-while-revalidate 模式

允许返回 stale 数据的同时，在后台异步刷新缓存：

```python
class StaleWhileRevalidateCache:
    """Stale-while-revalidate 缓存"""

    def __init__(self, cache: MultiLayerCache, max_stale_ratio: float = 2.0):
        self._cache = cache
        self._max_stale_ratio = max_stale_ratio
        self._revalidating: dict[str, asyncio.Event] = {}

    async def get(self, namespace: str, key: str, refresher: Callable) -> dict:
        result = await self._cache.get(namespace, key)

        if result is None:
            # 完全未命中：同步获取
            value = await refresher()
            await self._cache.set(namespace, key, value)
            return value

        # 缓存命中：检查是否已过原始 TTL
        age = time.time() - result.get("_cached_at", time.time())
        original_ttl = result.get("_ttl", 3600)

        if age < original_ttl:
            # 数据仍然是新鲜的：直接返回
            return result

        if age < original_ttl * self._max_stale_ratio:
            # 数据已过原始 TTL，但在 stale 容忍窗口内
            # 返回 stale 数据 + 异步刷新
            if key not in self._revalidating:
                self._revalidating[key] = asyncio.Event()
                asyncio.create_task(self._revalidate(namespace, key, refresher))
            return result  # 返回 stale 数据

        # 超过最大 stale 容忍度：同步等待刷新
        if key in self._revalidating:
            await self._revalidating[key].wait()
            result = await self._cache.get(namespace, key)
            if result is not None:
                return result

        # 同步刷新
        value = await refresher()
        await self._cache.set(namespace, key, value)
        return value

    async def _revalidate(self, namespace: str, key: str, refresher: Callable):
        try:
            fresh_value = await refresher()
            await self._cache.set(namespace, key, fresh_value)
        except Exception as e:
            logger.error(f"Revalidation failed for {namespace}:{key}: {e}")
        finally:
            if key in self._revalidating:
                self._revalidating[key].set()
                del self._revalidating[key]
```

### 5.2 缓存屏障（Cache Shield）

防止惊群效应（Thundering Herd）：当大量并发请求同时发现缓存过期时，只有第一个请求穿透到后端，其余请求等待第一个请求回填缓存。

```python
class CacheShield:
    """缓存屏障：防止缓存惊群效应"""

    def __init__(self, cache: MultiLayerCache, redis_client):
        self._cache = cache
        self._redis = redis_client

    async def get_or_compute(
        self,
        namespace: str,
        key: str,
        compute_fn: Callable,
        lock_ttl: int = 30,
    ):
        # 1. 尝试缓存读取
        result = await self._cache.get(namespace, key)
        if result is not None:
            return result

        # 2. 分布式锁：只有第一个请求获得锁
        lock_key = f"shield:lock:{namespace}:{key}"
        acquired = await self._redis.set(lock_key, "locked", nx=True, ex=lock_ttl)

        if acquired:
            try:
                # Double-check
                result = await self._cache.get(namespace, key)
                if result is not None:
                    return result

                # 实际计算
                result = await compute_fn()
                await self._cache.set(namespace, key, result)
                return result
            finally:
                await self._redis.delete(lock_key)
        else:
            # 3. 未获得锁：等待回填
            # 轮询等待其他请求将数据写入缓存
            for _ in range(lock_ttl * 10):  # 最多等待 lock_ttl 秒
                await asyncio.sleep(0.1)
                result = await self._cache.get(namespace, key)
                if result is not None:
                    return result

            # 超时：可能持有锁的请求失败了，尝试获取锁重新计算
            # 这里需要处理死锁检测
            raise TimeoutError("Cache shield timeout: compute did not complete")
```

### 5.3 概率性过期（Probabilistic Expiry）

前文提到的概率性过期可以作为一个独立的优雅降级策略。结合 TTL 使用，可以在数据完全过期前逐步降低其"被信任"的概率，从而平滑地过渡到新数据。

```python
class ProbabilisticExpiryCache:
    """概率性过期缓存：逐步降低信任度"""

    def __init__(self, cache: MultiLayerCache, base_ttl: float):
        self._cache = cache
        self._base_ttl = base_ttl

    async def get(self, namespace: str, key: str, compute_fn: Callable) -> dict:
        result = await self._cache.get(namespace, key)

        if result is None:
            value = await compute_fn()
            await self._cache.set(namespace, key, value)
            return value

        # 计算缓存年龄
        age = time.time() - result.get("_cached_at", time.time())
        freshness = 1.0 - (age / self._base_ttl)

        if freshness > 0:
            # 数据仍然"新鲜"
            result["_freshness"] = freshness
            return result
        else:
            # 数据已过 base TTL
            # 概率性决定是否返回 stale 数据还是重新计算
            # 随着 age 增加，返回 stale 数据的概率趋近于 0
            stale_probability = max(0, 1.0 - (self._base_ttl / max(age, 1)))
            if random.random() < stale_probability:
                # 重新计算
                value = await compute_fn()
                await self._cache.set(namespace, key, value)
                return value
            else:
                # 继续使用 stale 数据，但标记低新鲜度
                result["_freshness"] = max(0, freshness)
                result["_stale_warning"] = True
                return result
```

---

## 6. 完整实现：CacheInvalidationManager

综合以上所有模式，实现一个统一的缓存失效管理器：

```python
import asyncio
import logging
from enum import Enum
from typing import Callable

logger = logging.getLogger(__name__)


class InvalidationStrategy(Enum):
    TTL = "ttl"
    ADAPTIVE_TTL = "adaptive_ttl"
    EVENT_DRIVEN = "event_driven"
    LAZY_VALIDATION = "lazy_validation"
    STALE_WHILE_REVALIDATE = "stale_while_revalidate"
    PROBABILISTIC = "probabilistic"


@dataclass
class CachePolicy:
    """缓存策略定义"""
    strategy: InvalidationStrategy
    base_ttl: float = 3600.0
    max_stale_ratio: float = 2.0
    probabilistic_beta: float = 1.0
    enable_cascade: bool = False
    max_cascade_depth: int = 5


class CacheInvalidationManager:
    """统一的缓存失效管理器：支持多种失效策略"""

    def __init__(
        self,
        cache: MultiLayerCache,
        redis_client=None,
        event_bus: EventBus | None = None,
    ):
        self._cache = cache
        self._redis = redis_client
        self._event_bus = event_bus
        self._policies: dict[str, CachePolicy] = {}
        self._alert_tracker = AdaptiveTTLStrategy()
        self._cascade = CascadingInvalidation()
        self._shield = CacheShield(cache, redis_client) if redis_client else None

    def register_policy(self, namespace: str, policy: CachePolicy):
        """为特定 namespace 注册缓存策略"""
        self._policies[namespace] = policy
        logger.info(f"Registered policy {policy.strategy.value} for namespace {namespace}")

    async def get(
        self,
        namespace: str,
        key: str,
        compute_fn: Callable | None = None,
        context: dict | None = None,
    ) -> dict:
        policy = self._policies.get(namespace, CachePolicy(InvalidationStrategy.TTL))

        # 上下文感知 key 生成
        if context:
            key = ContextAwareCacheKey().make_key(key, context)

        # 根据策略选择执行路径
        if policy.strategy == InvalidationStrategy.STALE_WHILE_REVALIDATE:
            swr = StaleWhileRevalidateCache(self._cache, policy.max_stale_ratio)
            return await swr.get(namespace, key, compute_fn)

        elif policy.strategy == InvalidationStrategy.PROBABILISTIC:
            prob = ProbabilisticExpiryCache(self._cache, policy.base_ttl)
            return await prob.get(namespace, key, compute_fn)

        elif policy.strategy == InvalidationStrategy.LAZY_VALIDATION:
            lazy = LazyValidationCache(self._cache)
            return await lazy.get_with_validation(
                namespace, key,
                validator=lambda v: self._lightweight_validate(namespace, v),
                refresher=compute_fn,
            )

        elif policy.strategy in (InvalidationStrategy.ADAPTIVE_TTL, InvalidationStrategy.TTL):
            # 标准路径 + 自适应 TTL
            if self._shield:
                return await self._shield.get_or_compute(namespace, key, compute_fn)
            else:
                result = await self._cache.get(namespace, key)
                if result is None and compute_fn:
                    result = await compute_fn()
                    ttl = self._alert_tracker.get_ttl(key) if policy.strategy == InvalidationStrategy.ADAPTIVE_TTL else policy.base_ttl
                    await self._cache.set(namespace, key, result, ttl=ttl)
                return result

        else:
            # 默认 TTL 策略
            result = await self._cache.get(namespace, key)
            if result is None and compute_fn:
                result = await compute_fn()
                await self._cache.set(namespace, key, result, ttl=policy.base_ttl)
            return result

    async def invalidate(
        self,
        namespace: str,
        key: str,
        reason: str = "manual",
    ):
        """手动失效 + 级联传播"""
        policy = self._policies.get(namespace, CachePolicy(InvalidationStrategy.TTL))

        await self._cache.invalidate(namespace, key)
        self._alert_tracker.record_error(key)  # 记录失效事件
        logger.debug(f"Invalidated {namespace}:{key} (reason: {reason})")

        if policy.enable_cascade:
            await self._cascade.invalidate_cascade(
                self._cache, namespace, key,
                max_depth=policy.max_cascade_depth,
            )

    async def handle_event(self, event: CacheInvalidationEvent):
        """处理外部事件驱动的缓存失效"""
        policy = self._policies.get(event.namespace)
        if policy is None:
            logger.warning(f"No policy registered for namespace {event.namespace}")
            return

        logger.info(
            f"Event-driven invalidation: {event.event_type} "
            f"on {event.namespace}"
        )

        if event.keys:
            tasks = [
                self.invalidate(event.namespace, k, reason=event.event_type)
                for k in event.keys
            ]
            await asyncio.gather(*tasks)

    def _lightweight_validate(self, namespace: str, value: dict) -> bool:
        """轻量级验证：检查缓存条目的元数据完整性"""
        required_fields = ["_cached_at", "_ttl"]
        for field in required_fields:
            if field not in value:
                return False

        # 检查是否在有效期内
        age = time.time() - value["_cached_at"]
        if age > value["_ttl"] * 3:  # 超过 TTL 的 3 倍视为无效
            return False

        return True
```

---

## 7. 失效策略推荐矩阵

基于数据类型和系统特性，推荐的失效策略组合：

| 数据类型 | 推荐策略 | TTL | 级联 | 事件驱动 | 备注 |
|----------|----------|-----|------|----------|------|
| LLM 通用知识补全 | TTL + Stale-while-revalidate | 24h | 否 | 模型升级时 | 成本高，容忍一定延迟 |
| 工具调用结果（稳定）| TTL + 自适应 | 30min | 否 | API 变更时 | 视 API 稳定性调整 |
| 工具调用结果（实时）| 事件驱动 + 短 TTL | 60s | 否 | 需要 | 如天气、股票 |
| RAG 检索结果 | 事件驱动 + 懒验证 | 1h | 是 | 知识库变更时 | 需要级联失效 |
| 用户上下文 | 自适应 TTL + 概率性 | 15min | 否 | 用户操作时 | 会话级别隔离 |
| Embedding 向量 | 版本化 TTL + 事件驱动 | 7d | 是 | 文档变更时 | 计算成本高 |
| Prompt 模板 | 版本化 + 手动失效 | 永久 | 是 | 部署事件 | 代码版本控制 |
| 会话中间结果 | 短 TTL + 上下文感知 | 5min | 否 | 不需要 | 天然短期有效 |

**选择原则总结**：

1. **先考虑危害**：如果 stale 数据的危害大（金融交易、医疗诊断），首选事件驱动 + 主动验证；如果危害小（天气信息、闲聊回复），TTL + stale-while-revalidate 足够
2. **后考虑成本**：LLM 调用成本高的场景（长上下文、复杂推理），倾向长 TTL + 懒失效；成本低的场景（短 prompt），可以接受更频繁的失效
3. **再考虑复杂度**：简单的固定 TTL 能解决问题时，不要引入事件总线和级联失效 -- 分布式系统的每一点复杂度都有运维代价
4. **兜底策略永远存在**：即使有事件驱动机制，也必须设置合理 TTL 作为兜底 -- 事件可能丢失，消息可能延迟，系统可能故障

---

## 总结

缓存失效在 Agent 系统中不是一个技术选项，而是一个持续的设计决策。三种失效模式各有适用场景：

- **主动（TTL）失效**是基线 -- 简单、可靠、可预测，适合大部分通用场景
- **被动（事件驱动）失效**是精确打击 -- 实时、高效，适合底层数据源可观测的场景
- **主动验证（Lazy Validation）失效**是安全网 -- 灵活、容错，适合变化模式不可预测的场景

最佳实践是将这三种模式组合使用：以 TTL 为兜底，事件驱动保证实时性，懒验证处理边缘情况。配合级联失效和优雅降级策略（Stale-while-revalidate、缓存屏障、概率性过期），可以在不影响 Agent 响应质量的前提下，最大化缓存收益。

记住：缓存失效的目标不是"永远不返回 stale 数据"，而是"在可接受的代价范围内，返回足够新鲜的数据"。这个"足够新鲜"的标准，取决于 Agent 的任务类型、用户的期望和系统的成本约束。
