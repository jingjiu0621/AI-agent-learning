# 13.3.1 response-cache -- Response Cache: 精确命中与 TTL

**Response Cache 是 Agent 生产化中最直接有效的缓存策略 -- 当完全相同或高度相似的 LLM 请求重复出现时，直接返回缓存结果可以节省 30-60% 的 Token 消耗。核心矛盾在于：LLM 的生成式特性使"完全相同"的定义变得困难，而 TTL 的设定直接决定了缓存效率与数据新鲜度之间的 trade-off。**

---

## 1. 基本原理

### 1.1 Cache Key 设计

Response Cache 的核心是将 LLM 请求转化为唯一的缓存键（Cache Key）。一个典型的 LLM 请求包含以下维度：

```
Cache Key = f(system_prompt, messages, model, temperature, max_tokens, ...)
```

关键维度拆解：

| 维度 | 对 Key 的影响 | 建议策略 |
|------|--------------|---------|
| System Prompt | 高 -- 决定行为基准 | 必须包含，先做 normalization |
| User Messages | 高 -- 核心查询内容 | 必须包含，需 normalization |
| Model ID | 中 -- 不同模型输出不同 | 建议包含 |
| Temperature | 中 -- 影响随机性 | temp=0 时可忽略，>0 时缓存需谨慎 |
| Max Tokens | 低 -- 仅影响长度 | 可选择忽略或做范围归一化 |
| Tools/Function Calling | 高 -- 影响响应结构 | 必须包含 tool definitions |

### 1.2 精确匹配 vs 模糊匹配

**精确匹配（Exact Match）**：对 Cache Key 做哈希（如 SHA256），直接比对。优点是零误判，缺点是过于脆弱。

```
"帮我查一下北京的天气" != "帮我查一下 北京的天气"  // 空格差异导致失效
"查询北京的天气" != "帮我查一下北京的天气"          // 措辞不同导致失效
```

**模糊匹配（Fuzzy Match）**：使用编辑距离（Levenshtein）、N-gram 重叠度等指标。命中率提升但引入误判风险。实践中，模糊匹配通常在语义缓存层面解决（见 13.3.2），Response Cache 侧重精确匹配 + 规范化。

### 1.3 TTL 设定原则

TTL（Time-To-Live）是缓存有效期的核心参数。不同任务类型需要差异化的 TTL：

```
任务类型            TTL 范围          理由
--------           --------          ----
事实性问答          1h - 24h         答案不会频繁变化
代码生成            5min - 1h        上下文相关性强
客服 FAQ            6h - 48h         知识库更新周期长
实时数据分析        < 1min           数据时效性要求高
个性化推荐          不缓存或不透明    每个人都有不同偏好
```

---

## 2. 背景与演进

### 2.1 早期做法：直接缓存

最早的实现方式非常简单 -- 直接将 LLM 返回的 response 原文作为缓存值，用完整 prompt 文本的哈希作为 key：

```python
# 早期 naive 实现
cache_key = hashlib.md5(prompt_text.encode()).hexdigest()
```

**问题**：prompt 中任何微小的变化都会导致缓存完全失效。换行符、多余空格、同义词替换，甚至 timestamp 类动态内容都会使缓存命中率趋近于零。

### 2.2 规范化（Normalization）预处理

现代 Response Cache 在生成 key 之前会做规范化处理：

```
原始 prompt              经规范化后               是否匹配
"你好, 请问北京天气?"      "你好请问北京天气"        -
"你好，请问北京天气？"      "你好请问北京天气"        匹配 (全角统一)
"你好! 请问北京天气?"       "你好请问北京天气"        匹配 (标点去除)
"你好 请问 北京 天气"       "你好请问北京天气"        匹配 (空格压缩)
```

规范化步骤包括：
1. 去除不可见字符（零宽空格、多余空白）
2. 统一全角/半角标点
3. 统一大小写
4. 剔除时间戳、会话 ID 等动态字段
5. 对 Messages 数组按角色重组为标准格式

### 2.3 演进路线

```
naive hash --> normalization --> structured key --> multi-level key
  2019-2021      2021-2023         2023-2024          2024-2025
```

---

## 3. 主流实现方案

### 3.1 OpenAI Prompt Caching

OpenAI 提供服务端自动缓存。当请求的 prefix（前 1024 tokens）与已有缓存匹配时，自动应用 50% 折扣。

```
实现方式：Server-side prefix cache
缓存单位：Token 级别的 prefix
触发条件：prompt 长度 > 1024 tokens
折扣力度：50% input token cost 减免
TTL：5-10 分钟无访问后过期
```

使用示例：

```python
from openai import OpenAI

client = OpenAI()
# OpenAI 自动检测 cacheable prefix
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": LONG_SYSTEM_PROMPT},
        {"role": "user", "content": user_query}
    ]
)
# response.usage.prompt_tokens_details.cached_tokens 可获取缓存命中数
```

### 3.2 Anthropic Prompt Caching

Anthropic 的缓存机制支持显式的 cache breakpoint，允许开发者手动标记缓存前缀：

```
实现方式：Explicit cache breakpoints
缓存单位：指定消息之前的全部内容
触发条件：手动设置 cache_control
折扣力度：第一层 cache 折扣 90%，深层递减
TTL：5 分钟无访问后过期（2025），1h（2026 扩展）
```

使用示例：

```python
import anthropic

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": LONG_SYSTEM_PROMPT,
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=[{"role": "user", "content": user_message}]
)
```

### 3.3 Provider Cache 的局限性

1. 只缓存 prefix，不缓存完整 response -- 对完全相同的重复请求仍需 LLM 推理
2. 缓存策略不可控（OpenAI）或需要手动标记（Anthropic）
3. 跨会话缓存不保证
4. 不解决 tool call 结果的复用

自建 Response Cache 可以弥补这些不足。

---

## 4. Python 代码示例：完整 ResponseCache

以下实现了一个生产可用的 ResponseCache 类：

```python
"""
response_cache.py -- 生产级 Response Cache 实现
后端: Redis
特性: 规范化 key、TTL 管理、命中统计、分级匹配策略
"""

import json
import hashlib
import re
import time
from typing import Optional, Dict, Any, List
from dataclasses import dataclass, field
from enum import Enum

import redis.asyncio as aioredis


class MatchStrategy(Enum):
    """分级匹配策略"""
    EXACT = "exact"           # 原始精确匹配
    NORMALIZED = "normalized" # 规范化后匹配
    PATTERN = "pattern"       # 模式匹配（正则提取关键语义）


@dataclass
class CacheConfig:
    """缓存配置"""
    default_ttl_seconds: int = 3600
    task_ttl_map: Dict[str, int] = field(default_factory=lambda: {
        "factual": 86400,      # 事实性问答 24h
        "code": 600,            # 代码生成 10min
        "faq": 43200,           # 客服 FAQ 12h
        "realtime": 30,         # 实时数据 30s
    })
    enable_stats: bool = True
    redis_url: str = "redis://localhost:6379/0"


class ResponseCache:
    """Response Cache 核心类"""

    def __init__(self, config: CacheConfig):
        self.config = config
        self.redis = None
        self.stats = {
            "hits": 0,
            "misses": 0,
            "total_saved_tokens": 0,
        }

    async def initialize(self):
        """初始化 Redis 连接"""
        self.redis = await aioredis.from_url(
            self.config.redis_url,
            decode_responses=True
        )

    # ---------------------------------------------------------------
    # Key 生成与规范化
    # ---------------------------------------------------------------

    @staticmethod
    def _normalize_text(text: str) -> str:
        """规范化文本用于 key 生成"""
        # 去除零宽字符
        text = re.sub(r'[​‌‍⁠﻿]', '', text)
        # 统一全角半角
        text = text.replace('　', ' ')        # 全角空格
        text = text.replace('！', '!')        # 全角感叹号
        text = text.replace('，', ',')        # 全角逗号
        text = text.replace('？', '?')        # 全角问号
        text = text.replace('“', '"')         # 左双引号
        text = text.replace('”', '"')         # 右双引号
        text = text.replace('‘', "'")         # 左单引号
        text = text.replace('’', "'")         # 右单引号
        # 压缩空白
        text = re.sub(r'\s+', ' ', text).strip()
        return text.lower()

    def _build_cache_key(
        self,
        system_prompt: str,
        messages: List[Dict[str, str]],
        model: str,
        temperature: float = 0.0,
        strategy: MatchStrategy = MatchStrategy.NORMALIZED,
        task_type: str = "factual"
    ) -> str:
        """构建结构化的缓存 key"""
        key_parts = {
            "strategy": strategy.value,
            "model": model,
            "temperature": temperature if strategy == MatchStrategy.EXACT else 0.0,
            "task_type": task_type,
        }

        if strategy == MatchStrategy.EXACT:
            # 精确匹配：保留所有原始信息
            key_parts["system"] = system_prompt
            key_parts["messages"] = json.dumps(messages, sort_keys=True)
        elif strategy == MatchStrategy.NORMALIZED:
            # 规范化匹配
            key_parts["system"] = self._normalize_text(system_prompt)
            normalized_msgs = [
                {"role": m["role"], "content": self._normalize_text(m.get("content", ""))}
                for m in messages
            ]
            key_parts["messages"] = json.dumps(normalized_msgs, sort_keys=True)
        elif strategy == MatchStrategy.PATTERN:
            # 模式匹配：仅保留关键语义片段
            key_parts["system"] = self._normalize_text(system_prompt)[:200]
            content = " ".join(
                self._normalize_text(m.get("content", ""))
                for m in messages
            )
            # 用关键词提取代替完整内容（简化版：仅截取前 500 字符）
            key_parts["content_sig"] = content[:500]

        serialized = json.dumps(key_parts, sort_keys=True)
        return f"llm:response:{hashlib.sha256(serialized.encode()).hexdigest()}"

    # ---------------------------------------------------------------
    # TTL 管理
    # ---------------------------------------------------------------

    def _get_ttl(self, task_type: str) -> int:
        """根据任务类型获取 TTL"""
        return self.config.task_ttl_map.get(
            task_type, self.config.default_ttl_seconds
        )

    # ---------------------------------------------------------------
    # 核心操作
    # ---------------------------------------------------------------

    async def get(
        self,
        system_prompt: str,
        messages: List[Dict[str, str]],
        model: str,
        temperature: float = 0.0,
        task_type: str = "factual",
    ) -> Optional[Dict[str, Any]]:
        """查询缓存"""
        # 尝试分级匹配：精确 -> 规范化 -> 模式
        strategies = [
            MatchStrategy.EXACT,
            MatchStrategy.NORMALIZED,
            MatchStrategy.PATTERN,
        ]
        for strategy in strategies:
            cache_key = self._build_cache_key(
                system_prompt, messages, model, temperature,
                strategy=strategy, task_type=task_type
            )
            result = await self.redis.get(cache_key)
            if result is not None:
                if self.config.enable_stats:
                    self.stats["hits"] += 1
                    self.stats["total_saved_tokens"] += len(result)
                return json.loads(result)

        if self.config.enable_stats:
            self.stats["misses"] += 1
        return None

    async def set(
        self,
        system_prompt: str,
        messages: List[Dict[str, str]],
        model: str,
        response: Dict[str, Any],
        temperature: float = 0.0,
        task_type: str = "factual",
    ):
        """写入缓存（使用规范化 key）"""
        cache_key = self._build_cache_key(
            system_prompt, messages, model, temperature,
            strategy=MatchStrategy.NORMALIZED,
            task_type=task_type
        )
        ttl = self._get_ttl(task_type)
        await self.redis.setex(
            cache_key, ttl, json.dumps(response)
        )

    async def invalidate(self, pattern: str = "llm:response:*"):
        """按模式失效缓存"""
        cursor = 0
        while True:
            cursor, keys = await self.redis.scan(
                cursor, match=pattern, count=100
            )
            if keys:
                await self.redis.delete(*keys)
            if cursor == 0:
                break

    # ---------------------------------------------------------------
    # 统计信息
    # ---------------------------------------------------------------

    def get_stats(self) -> Dict[str, Any]:
        """获取缓存命中统计"""
        total = self.stats["hits"] + self.stats["misses"]
        hit_rate = self.stats["hits"] / total if total > 0 else 0
        return {
            **self.stats,
            "hit_rate": round(hit_rate, 4),
            "estimated_cost_saved": self._estimate_cost_saved(),
        }

    def _estimate_cost_saved(self) -> float:
        """估算节省成本（美元）"""
        tokens = self.stats["total_saved_tokens"]
        # 按 GPT-4o 价格估算：$2.50 / 1M input tokens
        return round(tokens / 1_000_000 * 2.50, 4)

    async def close(self):
        if self.redis:
            await self.redis.close()
```

使用示例：

```python
async def main():
    config = CacheConfig(redis_url="redis://localhost:6379/0")
    cache = ResponseCache(config)
    await cache.initialize()

    system = "你是一个天气预报助手。请用中文回答。"
    msgs = [
        {"role": "user", "content": "北京今天天气怎么样？"}
    ]

    # 查询缓存
    result = await cache.get(system, msgs, "gpt-4o")
    if result:
        print(f"[CACHE HIT] {result}")
    else:
        # 调用 LLM ...
        llm_response = {
            "content": "北京今天晴，25-32度，东南风2级。",
            "usage": {"prompt_tokens": 50, "completion_tokens": 20}
        }
        await cache.set(system, msgs, "gpt-4o", llm_response)
        print(f"[MISS] LLM called, response cached")

    # 统计
    print(cache.get_stats())
    await cache.close()
```

---

## 5. 挑战与边界

### 5.1 流式响应（Streaming）的缓存处理

Streaming 场景下，LLM 逐个 token 返回，传统缓存需要在第一个 token 到达前决定是否命中。

**解决方案**：延迟流式决策
```
用户请求 -> 生成 cache key
         -> 异步检查缓存
         -> 命中: 从 Redis 读取完整响应并以流式格式逐块返回
         -> 未命中: 转发到 LLM stream，同时将完整响应存入缓存
```

实现要点：
- Streaming 缓存需额外存储"完整响应"用于后续命中
- 返回时需要模拟 SSE（Server-Sent Events）格式逐块输出
- 首个 token 的延迟会增加，因为需要等缓存查询完成

### 5.2 Tool Call 结果的缓存一致性

Agent 场景中，Tool Call 结果往往作为后续 LLM 调用的输入。缓存工具调用结果需要格外谨慎：

```
风险场景：
1. Agent 查询"用户余额"
2. 缓存返回 5 分钟前的余额（已过期）
3. Agent 基于过时余额做决策
4. 用户实际余额已变更 -> 决策错误
```

**最佳实践**：
- 只缓存幂等工具（getIdempotent）的调用结果
- 非幂等操作（写入、修改、删除）不应缓存
- 工具结果缓存加上数据版本号或时间戳

### 5.3 多轮对话中的上下文依赖

多轮对话中，每个新回合都包含完整历史，导致 cache key 几乎总是唯一：

```
回合 1: [system] -> [user: Q1] -> [assistant: A1]
回合 2: [system] -> [user: Q1] -> [assistant: A1] -> [user: Q2] -> [assistant: A2]
回合 3: [system] -> [user: Q1] -> [assistant: A1] -> [user: Q2] -> [assistant: A2] -> [user: Q3]
```

每个回合的 cache key 都不同，因为历史消息在增长。

**应对**：
- 只缓存最后的 user question 与对应的 response，剥离历史上下文
- 或使用上下文窗口截断（仅用最近 N 轮对话构建 key）
- 独立的长期记忆层（如 RAG）与 response cache 互补使用

### 5.4 缓存污染与错误传播

缓存一旦写入错误内容，所有后续命中都会返回同样的错误。

```
写入错误 -> 缓存 -> 命中 -> 错误传播 -> 更多命中 -> 错误放大
```

**防御措施**：
- Response 写入前的质量校验（内容长度、格式检查、敏感词过滤）
- 渐进式 TTL：新入缓存用短 TTL，多次验证后延长
- 手动失效机制：提供 API 按 pattern 或 tag 批量失效

---

## 6. 与其他策略的对比

```
策略               命中率   延迟节省  成本节省   误判风险   适用场景
----               ----    ----     ----      ----       ----
Response Cache     低-中   高        30-60%    极低       重复查询、固定模板
Semantic Cache    中-高    中-高     40-80%    中         意图相同但表述不同
Prompt Cache      中      中        20-50%    极低       长 system prompt 复用
Tool Result Cache  高      高        依赖工具   低-中      幂等工具结果复用
模型内部 KV Cache  自动    高        0        无         所有请求（provider 内置）
```

**Response Cache 的核心定位**：
- 最适合高重复率、低实时性要求的场景
- 最常见于：FAQ 机器人、固定格式报告生成、批量数据标注、定时任务触发
- 不适合：高度动态、个性化要求高、需要最新数据的场景

---

## 7. 工程优化建议

### 7.1 分级 Key 匹配策略

实践中，从低到高逐级尝试，平衡命中率与误判风险：

```
第一级: 精确匹配 (EXACT)
  - 直接用完整 prompt hash
  - 优点: 零误判
  - 命中率: 约 5-15%

第二级: 规范匹配 (NORMALIZED)
  - 规范化后 hash
  - 优点: 消除格式干扰
  - 命中率提升至: 15-30%

第三级: 模式匹配 (PATTERN)
  - 提取语义骨架后 hash
  - 优点: 容忍同义表达
  - 命中率提升至: 25-45%
  - 注: 此级有误判风险，需要搭配验证
```

### 7.2 自适应 TTL

根据缓存的实际使用模式动态调整 TTL：

```python
class AdaptiveTTL:
    """自适应 TTL 管理器"""

    def __init__(self, base_ttl: int = 3600):
        self.base_ttl = base_ttl
        self.access_counts: Dict[str, int] = {}
        self.min_ttl = 60
        self.max_ttl = 86400 * 7  # 最长一周

    def adjust(self, cache_key: str) -> int:
        """根据访问频率调整 TTL"""
        count = self.access_counts.get(cache_key, 0)
        if count > 100:
            # 高频访问 -> 延长 TTL
            multiplier = min(10, count // 10)
            return min(self.base_ttl * multiplier, self.max_ttl)
        elif count < 5:
            # 低频访问 -> 缩短 TTL
            return max(self.base_ttl // 2, self.min_ttl)
        return self.base_ttl
```

### 7.3 缓存预热策略

在预期流量高峰前，提前生成并填充关键查询的缓存：

```python
async def warmup_cache(cache: ResponseCache, expected_queries: List[dict]):
    """缓存预热：预计算并填充常见查询的响应"""
    print(f"Pre-warming cache with {len(expected_queries)} queries...")
    for query in expected_queries:
        # 检查是否已缓存
        existing = await cache.get(
            query["system"], query["messages"],
            query.get("model", "gpt-4o")
        )
        if existing is None:
            # 预调用 LLM 并缓存
            response = await call_llm(query)
            await cache.set(
                query["system"], query["messages"],
                query.get("model", "gpt-4o"), response
            )
    print("Cache warmup complete.")
```

### 7.4 监控指标

生产环境中必须监控以下指标：

```
- 命中率 (Hit Rate): < 20% 需要检查 cache key 设计
- 平均缓存大小: 大响应占用 Redis 内存，考虑压缩
- TTL 到期率: 大量到期说明 TTL 设置偏短
- 缓存写入延迟: 写入耗时 > 50ms 需排查
- 误判率: 缓存返回后用户不满意或纠错的比例
```

---

## 总结

Response Cache 是 Agent 成本优化的第一道防线。它实现简单、风险可控（精确匹配的误判率几乎为零），但命中率受限于 prompt 的重复程度。在典型生产环境中，它为 15-30% 的请求提供零推理成本的响应，配合规范化预处理可将这一比例提升至 30-45%。

关键在于：不要把它当作唯一的缓存策略。Response Cache 最适合做"第一层缓存" -- 用最低的成本覆盖最高的确定性重复。对于"意思相同但表述不同"的场景，请交给语义缓存（13.3.2）。
