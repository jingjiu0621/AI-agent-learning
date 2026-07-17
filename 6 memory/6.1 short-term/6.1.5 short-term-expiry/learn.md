# 6.1.5 short-term-expiry — 短期记忆过期与刷新

## 简单介绍

短期记忆过期（Short-term Memory Expiry）是一个**时间感知**的上下文管理机制——不是所有进入上下文的信息都永久有效。有些信息有时效性（如"当前时间""临时授权令牌"），有些信息会随着上下文变化而变得无关（如"已完成的子任务状态"），需要主动从短期记忆中移除或刷新。

没有过期机制的短期记忆会积累"信息垃圾"——过时的时间戳、已执行的指令副本、失效的工具结果，既浪费 Token 空间，又可能误导 Agent 的决策。

## 基本原理

### 为什么短期记忆需要过期

短期记忆面临三类"过时"问题：

```
时间轴 ────────────────────────────────────────────────→

消息1      消息2     消息3    消息4      消息5      当前时刻
[时间:T0]  [T1]     [T2]    [T3]      [T4]
   │         │        │       │         │
   │         │        │       │         └── 有效：刚发生的交互
   │         │        │       └──────────── 有效：正在进行的步骤
   │         │        └─────────────────── ❓ 半有效：已部分过时
   │         └──────────────────────────── ❌ 过时：步骤已执行完毕
   └────────────────────────────────────── ❌ 过时：初始规划已迭代
```

过期机制的目标是识别和清理那些**不再服务于当前目标的信息**。

### 过期类型

| 过期类型 | 例子 | 判断依据 |
|---------|------|---------|
| **时间过期** | "现在是 14:30" → 现在是 15:00 | 绝对时间或相对时间超限 |
| **状态过期** | "任务 A 进行中" → "任务 A 已完成" | 状态已经变迁 |
| **相关性过期** | "用户想查天气" → "用户已经问到天气结果" | 目标已经达成 |
| **层次过期** | 子步骤的详细信息 → 已进入下一步 | 粒度不再匹配 |
| **环境过期** | 工具 A 的认证令牌 → 已刷新 | 外部状态变更 |

## 过期策略

### 1. TTL 过期（Time-To-Live）

每条消息进入短期记忆时带有一个 TTL（存活时间），超时后自动标记为可回收。

```python
class TTLMessage:
    def __init__(self, content: dict, ttl_seconds: int = 300):
        self.content = content
        self.created_at = time.time()
        self.ttl = ttl_seconds

    def is_expired(self) -> bool:
        return (time.time() - self.created_at) > self.ttl

    def remaining(self) -> float:
        return max(0, self.ttl - (time.time() - self.created_at))

class TTLBasedExpiry:
    """基于 TTL 的过期管理"""

    def __init__(self):
        self.messages: list[TTLMessage] = []

    def add(self, content: dict, ttl: int | None = None):
        """添加消息，ttl=None 表示永不过期"""
        if ttl is None:
            ttl = {"system": None, "user": 600, "assistant": 600,
                   "tool": 300}.get(content.get("role", ""), 600)
        self.messages.append(TTLMessage(content, ttl))

    def purge_expired(self) -> list[dict]:
        """清理过期消息，返回被清理的内容"""
        valid, expired = [], []
        for msg in self.messages:
            if msg.is_expired():
                expired.append(msg.content)
            else:
                valid.append(msg)
        self.messages = valid
        return expired

    def get_valid_context(self) -> list[dict]:
        self.purge_expired()
        return [m.content for m in self.messages]
```

**TTL 建议值**：
| 消息类型 | 默认 TTL | 理由 |
|---------|---------|------|
| 系统指令 | 永久 | 定义 Agent 行为边界 |
| 工具定义 | 永久 | 工具能力描述不随时间变化 |
| 用户消息 | 10-15 min | 用户意图可能随时间变化 |
| 助手回复 | 10-15 min | 已执行的回复可以过期 |
| 工具结果 | 5-10 min | 结果会随系统状态变更 |
| 时间戳 | 1-5 min | 时间信息快速过时 |
| 临时令牌 | 与实际 TTL 一致 | 令牌过期后必然失效 |

### 2. 访问计数过期（Access-Count Expiry）

跟踪每条消息被访问/引用的次数，低访问频率的消息优先过期。

```python
class AccessCountExpiry:
    """基于访问频率的过期"""

    def __init__(self, min_access_count: int = 2):
        self.min_access_count = min_access_count
        self.messages: dict[str, dict] = {}  # id -> message
        self.access_count: dict[str, int] = {}

    def record_access(self, msg_id: str):
        """记录一次访问"""
        self.access_count[msg_id] = self.access_count.get(msg_id, 0) + 1

    def get_candidates_for_expiry(self) -> list[str]:
        """获取低频且非关键的过期候选"""
        return [
            msg_id for msg_id, count in self.access_count.items()
            if count < self.min_access_count
            and not self.messages[msg_id].get("critical", False)
        ]
```

### 3. 重要性阈值过期（Importance-Threshold Expiry）

每条消息带有重要性评分，当上下文空间不足时，移除低重要性消息。

```python
def importance_score(message: dict, current_task: str) -> float:
    """评估消息重要性 (0~1)"""
    score = 0.5  # 基准分

    # 角色权重
    role_weights = {"system": 1.0, "user": 0.8, "tool": 0.6}
    score += role_weights.get(message.get("role", ""), 0.5)

    # 任务相关性（关键词匹配）
    if current_task:
        text = str(message.get("content", ""))
        keywords = extract_keywords(current_task)
        relevance = sum(1 for kw in keywords if kw in text) / len(keywords)
        score += relevance * 0.3

    # 时间衰减
    age_hours = (time.time() - message.get("timestamp", time.time())) / 3600
    score *= max(0.5, 1.0 - age_hours * 0.1)

    return min(1.0, score)
```

### 4. 状态变更驱动过期（State-Change Expiry）

当 Agent 的任务状态发生显式变更（如任务完成、步骤推进），相关消息自动过期。

```python
class StateChangeExpiry:
    """状态变更驱动的过期"""

    def __init__(self):
        self.contexts: dict[str, set[str]] = {}  # task_id -> set of msg_ids

    def on_task_complete(self, task_id: str):
        """任务完成时，过期相关上下文"""
        expired_ids = self.contexts.pop(task_id, set())
        return expired_ids

    def on_state_transition(self, old_state: str, new_state: str):
        """状态迁移时，过期旧状态的上下文"""
        state_msg_map = {
            "planning": {"system_prompt_v1", "initial_requirements"},
            "executing": {"task_plan_v1"},
            "verifying": {"execution_details"},
        }
        expired = state_msg_map.get(old_state, set())
        return expired
```

## 刷新机制

过期不是终点——有时"过期"的信息需要被**刷新**后重新进入短期记忆。

### 刷新触发条件

```python
class RefreshTrigger:
    """刷新触发策略"""

    def should_refresh(self, item: ContextItem, current_state: dict) -> bool:
        """判断是否应该刷新某个过期项"""
        reasons = []

        # 1. 用户主动提及
        if current_state.get("user_query") and item.matches(current_state["user_query"]):
            reasons.append("user_reference")

        # 2. 任务依赖
        if item.id in current_state.get("task_dependencies", set()):
            reasons.append("task_dependency")

        # 3. 周期性刷新（如每 N 轮刷新一次时间信息）
        if item.periodic and item.rounds_since_refresh >= item.refresh_interval:
            reasons.append("periodic")

        # 4. 环境变更通知
        if item.monitors_external and item.external_changed():
            reasons.append("external_change")

        return len(reasons) > 0, reasons
```

### 刷新策略

| 策略 | 操作 | 开销 | 适用场景 |
|------|------|------|---------|
| **完全刷新** | 重新获取最新版本替换旧版本 | 高 | 认证令牌、时间信息 |
| **增量刷新** | 只更新变化的部分 | 中 | 进度跟踪、状态更新 |
| **续期（延期）** | 不刷新内容，仅延长 TTL | 低 | 持续相关的历史上下文 |
| **重新激活** | 从长期记忆重新加载 | 高 | 用户重新提及旧话题 |

```python
def refresh_item(item: ContextItem, strategy: str = "extend") -> ContextItem:
    if strategy == "renew":
        item.ttl_remaining = item.default_ttl  # 续期
        item.access_count += 1
    elif strategy == "refresh":
        new_data = fetch_latest(item.source)
        item.update_content(new_data)
        item.ttl_remaining = item.default_ttl
    elif strategy == "reload":
        item.content = long_term_memory.retrieve(item.memory_key)
        item.ttl_remaining = item.default_ttl
    return item
```

## 背景

### 从"永不遗忘"到"智能遗忘"

早期 Agent 设计倾向于尽可能保留所有上下文——认为"记住更多 = 表现更好"。但实践中发现：

- **信息噪声稀释注意力**：大量过时信息降低了 LLM 对真正重要信息的关注
- **过期信息误导决策**：Agent 可能基于已过时的工具结果或任务状态做决策
- **Token 浪费**：无效信息占用宝贵的上下文空间，降低有效交互长度

"选择性遗忘"逐渐被认为比"无差别记忆"更重要。

### 与人类记忆的类比

人类短期记忆的遗忘机制是经过进化优化的——遗忘不是为了丢弃，而是为了**更好地记住重要的事情**。Agent 的短期记忆过期机制借鉴了这一点：
- **衰退理论**：不使用就会遗忘 → 访问计数过期
- **干扰理论**：新信息干扰旧信息 → TTL 过期
- **线索依赖遗忘**：没有合适的检索线索就无法提取 → 刷新机制

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **过早过期 vs 过晚过期** | 过期太早丢失关键信息，过期太晚浪费 Token 空间 |
| **自动过期 vs 用户控制** | 自动过期省心但可能误删，用户控制安全但增加交互负担 |
| **静态 TTL vs 动态调整** | 固定 TTL 简单但不够灵活，动态 TTL 效果好但实现复杂 |
| **本地过期 vs 全局一致性** | 单条消息过期很简单，但多条相关消息的一致过期需要细粒度跟踪 |

## 当前主流优化方向

1. **多因素过期决策**：结合 TTL + 访问频率 + 重要性 + 状态变更，综合判断是否过期
2. **预测性过期**：基于任务类型和 Agent 行为模式，预测哪些消息将不再需要，提前标记
3. **分层 TTL**：不同类型消息使用不同的 TTL 和过期策略，形成过期优先级
4. **过期审计与回滚**：记录过期决策日志，支持在 Agent 行为异常时回滚过期操作
5. **协同过期**：多个相关消息作为一个组过期（如"一条工具调用及其结果"一起过期）

## 工程优化方向

1. **消息标记体系**：每条消息进入短期记忆时即携带过期元数据（类型、TTL、重要性、依赖关系）
2. **惰性过期**：不实时扫描过期消息，在每次需要截断时才评估和清理
3. **过期回调**：消息过期时触发回调函数，允许将关键信息摘要后存入长期记忆
4. **预热刷新**：在消息即将过期但 Agent 可能还会用到时（如基于预测），提前刷新
5. **用户显式过期**：提供接口让用户可以标记"这条信息已经不需要了"

## 能力边界与结果边界

**能做的**：
- 减少过期/无效信息对 Agent 注意力的干扰
- 释放 Token 空间给更重要的新信息
- 提高 Agent 在长时间会话中的相关性保持能力

**不能做的**：
- ❌ 准确预测哪些信息未来会被用到（只能估计）
- ❌ 完全消除"过期关键信息"的风险（总是存在误判可能）
- ❌ 替代良好的长期记忆设计（过期只解决短期，长期信息需要持久化）

**结果边界**：
- 好的过期策略可以提升 Agent 长会话的相关性 20-40%
- 过期机制需要配合截断策略一起工作——过期是"标记"，截断是"执行"
- 过期策略的效果高度依赖消息类型标记的准确性

## 与其他内容的区别

| 对比 | 过期机制 | 截断策略 | 记忆整合 |
|------|---------|---------|---------|
| 触发原因 | 时间/状态/相关性 | Token 超限 | 空闲时间 |
| 操作 | 标记为可移除 | 物理移除 | 重写为长期记忆 |
| 是否恢复 | 可刷新恢复 | 不可恢复 | 已存入长期 |
| 粒度 | 单条消息 | 批量消息 | 批量 |
| 时间视角 | 面向未来（何时失效） | 面向现在（空间不足） | 面向过去（总结历史） |

## 核心优势

1. **选择性遗忘**：不是简单丢弃，而是基于信息价值和时效性的智能管理
2. **刷新机制**：过期的信息可以在需要时重新激活，保留恢复能力
3. **自然对齐**：与人类认知中的"遗忘曲线"一致，交互更自然
4. **可组合性**：TTL + 重要性 + 状态变更多种策略可组合

## 适合场景的判断标准

- **会话时长**：短会话（<5 轮）→ 不需要过期；长会话（>20 轮）→ 需要 TTL 策略；极长会话 → 需要综合策略
- **信息时效性**：涉及实时数据（时间、价格、状态）→ 需要严格 TTL；静态知识 → 不需要过期
- **状态变更频率**：任务状态频繁变化 → 需要状态变更过期；稳定对话 → 基本过期就够
- **用户行为**：用户频繁回退到旧话题 → 需要延长 TTL + 刷新机制

**最适合**：长时间运行的 Agent 会话、涉及实时信息的 Agent（天气/股票/监控）、多步骤任务执行流程

**最不适合**：单轮问答、离线批处理任务、每次对话完全独立的场景
