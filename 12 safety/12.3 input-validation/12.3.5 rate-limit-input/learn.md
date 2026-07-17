# 12.3.5 rate-limit-input — 输入速率限制

**速率限制是 Agent 系统的第一道流量防线。** 与传统 API 速率限制不同，Agent 的速率限制不仅要考虑请求频率，还要考虑 LLM 调用成本、工具执行开销、上下文窗口压力——每一个 Agent 请求的背后都链接着昂贵的计算资源。

---

## 简单介绍

Rate Limiting（速率限制）通过控制单位时间内的请求数量或消耗量，防止 Agent 系统被滥用、过载或遭受暴力攻击。在 Agent 体系中，速率限制不仅是安全机制，也是成本控制和资源调度手段——一个精心设计的限速器可以在保障服务质量的同时，将 LLM API 成本控制在预算范围内。

Agent 速率限制的独特挑战在于：一次用户请求可能触发多次 LLM 调用和多轮工具执行，单一维度的限速远远不够。

---

## 基本原理

```
用户请求 ──→ 速率限制器 ──→ (通过) ──→ Agent 处理
                  │
                  ├── 请求频率 (RPS/RPM)
                  ├── Token 消耗 (TPM/TPD)
                  ├── 工具调用次数
                  ├── 成本预算 (USD/会话)
                  └── 并发请求数
```

速率限制的核心原理基于一个简单的观察：**合法用户的行为模式与攻击者不同。** 合法用户有思考间隙、操作间隔和自然的使用节奏；而攻击者会以机器速度发出大量请求，试图耗尽系统资源或暴力破解防护。

```
攻击者请求模式 (密集、规律):  ████████████████████████████████
合法用户请求模式 (稀疏、突发): █    ██   █    ███  █    █   █
```

速率限制要做的就是在两者之间画一条线——允许正常使用模式的波动，同时拦截异常流量。

---

## 背景 — why Agent rate limiting is different from API rate limiting

传统 API 速率限制处理的是**同质化请求**——每个 API 调用的资源消耗大致相同。Agent 速率限制面临的是**异质化请求**——每一次用户输入可能触发完全不同的资源消耗链。

```
传统 API 速率限制                      Agent 速率限制
────────────────────                  ────────────────────
• 请求数/秒 (RPS)                     • RPS + 会话级 Token 预算
• 用户维度限流                         • 用户 + 工具 + 模型维度
• 固定窗口算法即可                      • 多维度滑动窗口 + 自适应
• 资源消耗可预估                        • 资源消耗不确定 (LLM 动态输出)
• 限流后直接拒绝                        • 限流后排队或降级
• 成本不由请求频率直接决定                • 每个请求 = LLM 调用成本 + 工具成本
```

### Agent 速率限制的特殊维度

| 维度 | 说明 | 为什么 Agent 更复杂 |
|------|------|-------------------|
| **会话级限速** | 一个 Session 内总请求量 | Agent 会话可能持续数小时，需要累计预算 |
| **成本级限速** | LLM Token 消耗 + 工具调用费用 | 一次复杂 Agent 调用可能消耗数千 Token |
| **工具级限速** | 特定工具的调用频率 | 某些工具（如数据库查询、文件写入）有自身限流 |
| **模型级限速** | 不同模型有不同的 TPM 限制 | Agent 可能在不同模型间路由 |
| **思维链深度限速** | 限制 Agent 推理循环次数 | 防止 Agent 陷入无限反思循环 |
| **上下文窗口限速** | 限制累计上下文大小 | 防止超长对话耗尽窗口预算 |

### 典型 Agent 调用链中的资源消耗

```
用户: "分析这份销售数据并生成报告"
  ↓
├── LLM 调用 #1 (理解意图)  →  ~500 tokens 输入
├── 工具: 读取文件           →  1 次 API 调用
├── LLM 调用 #2 (分析数据)  →  ~4000 tokens 输入 + ~800 tokens 输出
├── 工具: 数据库查询         →  1 次数据库调用
├── LLM 调用 #3 (生成报告)  →  ~6000 tokens 输入 + ~2000 tokens 输出
├── 工具: 写入文件           →  1 次文件操作
├── LLM 调用 #4 (总结)      →  ~1000 tokens 输入 + ~300 tokens 输出
  ↓
总计: 1 次用户请求 → 4 次 LLM 调用 + 3 次工具调用 + ~15000 tokens
```

一个用户请求的"实际成本"是表面请求数的 5-10 倍。

---

## 核心矛盾

Rate Limiting 面临的根本矛盾是 **保护系统 vs 服务用户** 之间的张力。

| 矛盾 | 说明 |
|------|------|
| **严格限流 vs 用户体验** | 限流太严，合法用户的突发请求（如批量导入、赶截止日期）被误伤 |
| **成本控制 vs 服务质量** | 限制 Token 消耗控制成本，但可能导致 Agent "思考不足"而回答错误 |
| **对抗攻击 vs 误报** | 分布式攻击的请求模式和正常高并发用户很难区分 |
| **静态阈值 vs 动态负载** | 固定阈值在高峰期浪费容量，在低峰期不够用 |
| **提前拒绝 vs 事后降级** | 提前拒绝可能阻断重要请求，事后降级已造成资源浪费 |

```
风险平衡:

  过于宽松                       过于严格
  ────────                       ────────
  ✓ 用户体验好                    ✗ 用户体验差
  ✗ 成本失控                      ✓ 成本可控
  ✗ 容易遭受 DoS                  ✓ 防攻击能力强
  ✗ 资源被滥用                    ✓ 资源公平分配
  ✗ Token 预算超支                ✓ 预算不超支
```

**合理的速率限制不是在两个极端选择，而是对不同请求采用不同阈值。** 例如：高频用户降级到低成本模型，而不是直接拒绝。

---

## 详细内容

### 1. Rate Limit Dimensions: per-user, per-session, per-IP, per-tool, per cost budget

Agent 系统需要在多个维度同时施加限制，形成**多维限速矩阵**：

```
                     ┌──────┐
          ┌──────────┤ 用户 ├──────────┐
          │          └──────┘          │
          ▼                            ▼
     ┌────────┐                  ┌──────────┐
     │ 会话   │                  │  IP      │
     └───┬────┘                  └────┬─────┘
          │                           │
          ▼                           ▼
     ┌────────┐                  ┌──────────┐
     │ 工具   │                  │  模型    │
     └───┬────┘                  └────┬─────┘
          │                           │
          ▼                           ▼
     ┌────────┐                  ┌──────────┐
     │ 成本   │                  │ 全局     │
     └────────┘                  └──────────┘
```

#### 各维度详解

| 维度 | 例子 | 用途 | 粒度 |
|------|------|------|------|
| **Per-User** | 100 请求/分钟/用户 | 防止单一用户滥用 | 用户 ID |
| **Per-Session** | 10000 tokens/会话 | 控制单次对话资源消耗 | Session ID |
| **Per-IP** | 1000 请求/分钟/IP | 应对无认证攻击 | IP 地址 |
| **Per-Tool** | 10 次数据库查询/分钟 | 保护后端服务 | 工具名 |
| **Per-Model** | 100K tokens/分钟 (GPT-4) | 遵守模型 API 配额 | 模型名 |
| **Per-Cost** | $5/天/用户 | 控制成本预算 | 累计费用 |
| **Global** | 100K 请求/分钟/集群 | 保护系统整体容量 | 集群 |

#### 多维度联合判定

```python
class MultiDimensionRateLimit:
    """多维度综合限速判定"""

    def is_allowed(self, request: AgentRequest) -> RateLimitResult:
        checks = [
            self._check_user_rate(request.user_id),       # 用户级
            self._check_session_budget(request.session),   # 会话级
            self._check_ip_rate(request.ip),               # IP 级
            self._check_tool_rate(request.tool_name),      # 工具级
            self._check_cost_budget(request.user_id),      # 成本级
            self._check_global_rate(),                      # 全局级
        ]
        # 任意维度拒绝则整体拒绝
        denied = [c for c in checks if not c.allowed]
        if denied:
            return RateLimitResult.denied(dimensions=denied)
        return RateLimitResult.allowed()
```

---

### 2. Algorithm Comparison: Token Bucket, Leaky Bucket, Sliding Window, Fixed Window

| 算法 | 原理 | 突发处理 | 精度 | 内存开销 | Agent 适用场景 |
|------|------|---------|------|---------|-------------|
| **Fixed Window** | 固定时间窗口内计数，到上限则拒绝 | 差（窗口边界突发） | 低 | 极小 | 简单日/月配额 |
| **Sliding Window** | 按时间权重滑动统计，窗口内计数 | 好 | 高 | 中 | **推荐：Agent 通用限速** |
| **Token Bucket** | 桶中 Token 匀速补充，每次请求消耗一个 | 好（允许一定突发） | 中 | 低 | **推荐：突发请求平滑** |
| **Leaky Bucket** | 请求进桶，匀速流出，桶满则丢弃 | 差（不允许突发） | 中 | 低 | 严格均匀限速场景 |
| **Sliding Log** | 记录每个请求的时间戳，统计时间窗口内请求 | 好 | 最高 | 高 | 高精度审计场景 |

#### 算法对比图解

```
Fixed Window:         |←──── 1 min ────→|←──── 1 min ────→|
  窗口开始重置，         ████████████░░░░
  边界处突发流量通过     ← 边界突发 →

Sliding Window:
  按时间权重滑动        ░░░██████████░░░░
  无边界突发问题        ← 平滑过渡 →

Token Bucket:
  匀速补充 Token       ████████████████ ← 桶容量 10
  允许突发消耗          ░░░░░░░░░░░░░░░░ ← 补充速率 1/s
                      ↓ 突发可用

Leaky Bucket:
  请求排队匀速处理      ░░░█████░░░░░░░░░
  桶满丢弃             流出速率恒定
```

#### Agent 算法选择建议

| 场景 | 推荐算法 | 理由 |
|------|---------|------|
| 用户请求限速 (RPS) | Sliding Window | 精度高，无边界问题，实现复杂度适中 |
| Token 消耗控制 (TPM) | Token Bucket | 允许突发消耗（合法用户偶尔发长文本），长期平滑 |
| 成本预算控制 | Fixed Window | 按月/日重置，精度要求不高，实现简单 |
| 工具调用限速 | Leaky Bucket | 工具后端需要稳定流量，不允许突发 |
| 高精度审计 | Sliding Log | 需要精确记录每次请求用于计费/审计 |

---

### 3. Agent-Specific Rate Limits: LLM call limits, tool invocation limits, context window reset frequency

Agent 系统独有的限速维度，传统 API 限速不涉及。

#### LLM 调用限制

```
LLM 调用不是简单的"每秒次数"，而是多层约束：
  ┌─────────────────────────────────┐
  │ 输入 Token 速率 (TPM)            │ ← 模型 API 硬限制
  │ 输出 Token 速率 (TPM)            │ ← 模型 API 硬限制
  │ 每分钟请求数 (RPM)               │ ← API 配额限制
  │ 并发请求数                       │ ← 连接池限制
  │ 每次请求最大 Token               │ ← 上下文窗口限制
  │ 每日总 Token 预算                │ ← 成本控制
  └─────────────────────────────────┘
```

```python
class LLMRateLimits:
    """LLM 调用限速配置示例"""

    # GPT-4 典型限速 (API 级别)
    gpt4 = {
        "rpm": 500,                    # 每分钟请求数
        "tpm_input": 100000,           # 每分钟输入 tokens
        "tpm_output": 50000,           # 每分钟输出 tokens
        "max_concurrent": 10,          # 最大并发
        "max_context": 128000,         # 最大上下文 tokens
    }

    # Agent 内部更严格的限速 (成本控制)
    agent = {
        "rpm_per_user": 50,            # 每用户每分钟
        "tpm_per_session": 50000,      # 每会话总 tokens
        "cost_per_session": 0.50,      # 每会话成本上限 ($)
        "cost_per_day_per_user": 5.00, # 每用户每日成本上限 ($)
    }
```

#### Tool Invocation Limits

工具调用限速需要考虑工具的"伤害半径"：

| 工具类型 | 限速严格度 | 理由 | 示例限制 |
|---------|-----------|------|---------|
| **读取工具** (搜索、文件读取) | 中等 | 数据泄露风险，API 配额 | 30 次/分钟 |
| **写入工具** (数据库写入、文件创建) | 严格 | 数据完整性风险 | 5 次/分钟 |
| **执行工具** (代码执行、shell) | 最严格 | 安全风险最高 | 2 次/分钟 |
| **网络工具** (HTTP 请求) | 中等 | 对外部系统影响 | 10 次/分钟 |
| **高成本工具** (LLM 调用自身) | 严格 | LLM 再入调用产生循环 | 3 次/分钟 |
| **敏感工具** (删除、权限修改) | 极严格 | 不可逆操作 | 1 次/小时 |

#### Context Window Reset Frequency

Agent 会话的上下文窗口管理也涉及速率限制逻辑：

```
相关问题:
  • 多久需要重置/压缩一次上下文？
  • 上下文压缩本身的成本 (需要一次 LLM 调用)
  • 对话轮次限制 (防止上下文无限增长)
  • Token 累计速率 (每轮对话都会累加)

典型策略:
  轮次 ≤ 20  → 完整上下文 (原始)
  轮次 20-50 → 摘要上下文 (压缩)
  轮次 > 50  → 限流或强制新会话
```

---

### 4. Cost-Based Rate Limiting: token budget per session, per-user daily cost cap, model tier limits

Cost-based rate limiting 是 Agent 系统独有的限速维度——限制的不是请求次数，而是**金钱消耗**。

#### 限速层级

```
Layer 1: Per-Request 成本控制
  ├── 单次请求最大 Token (输入 + 输出)
  ├── 单次请求最大工具调用数
  └── 单次请求最大延迟时间

Layer 2: Per-Session 预算
  ├── 单次对话最大 Token 消耗
  ├── 单次对话最大工具调用数
  └── 单次对话最大耗时

Layer 3: Per-User 日/月预算
  ├── 用户每日成本上限
  ├── 用户每月成本上限
  └── 超出后降级到低成本模型

Layer 4: 全局预算
  ├── 团队/组织每日总预算
  ├── 模型间预算分配 (GPT-4 20%, GPT-3.5 80%)
  └── 紧急熔断阈值
```

#### 成本计算模型

```python
class CostCalculator:
    """实时计算 Agent 操作的成本"""

    # 模型定价 (每 1K tokens)
    MODEL_PRICING = {
        "gpt-4":       {"input": 0.03,  "output": 0.06},
        "gpt-4-turbo": {"input": 0.01,  "output": 0.03},
        "gpt-3.5":     {"input": 0.001, "output": 0.002},
        "claude-3-opus":   {"input": 0.015, "output": 0.075},
        "claude-3-sonnet": {"input": 0.003, "output": 0.015},
    }

    def estimate_cost(self, model: str, input_tokens: int,
                      output_tokens: int) -> float:
        pricing = self.MODEL_PRICING.get(model, self.MODEL_PRICING["gpt-3.5"])
        input_cost = (input_tokens / 1000) * pricing["input"]
        output_cost = (output_tokens / 1000) * pricing["output"]
        return round(input_cost + output_cost, 6)
```

#### Model Tier Limits (模型自动降级)

当用户超过某个预算阈值时，自动将其路由到低成本模型，而不是直接拒绝：

```
用户请求 ──→ 预算检查 ──→ 预算充足 → GPT-4 (高质量)
                    │
                    └──→ 预算超出 50% → GPT-3.5 (中等质量)
                    │
                    └──→ 预算超出 80% → 本地模型 (基本功能)
                    │
                    └──→ 预算超出 100% → 限流 (仅保留核心功能)
```

---

### 5. Queue-Based Smoothing: request queuing, priority queuing, backpressure mechanisms

当限流触发时，不是所有系统都选择直接拒绝。队列平滑允许请求在等待后处理，提高资源利用率。

#### 请求排队 (Queue)

```
无队列:              有队列:
请求 ─→ ✗ 拒绝       请求 ─→ [队列] ─→ 处理
                      ↑           ↑
                   排队等待    匀速出队
```

```python
class RequestQueue:
    """带限速的请求队列"""

    def __init__(self, max_size: int = 100, process_rate: float = 10.0):
        self.queue = deque()
        self.max_size = max_size
        self.process_rate = process_rate  # 每秒处理数

    def enqueue(self, request: AgentRequest) -> QueueResult:
        if len(self.queue) >= self.max_size:
            return QueueResult.rejected("队列已满，请稍后重试")

        estimated_wait = len(self.queue) / self.process_rate
        if estimated_wait > 30:  # 等待超过 30 秒则拒绝
            return QueueResult.rejected(
                f"预计等待 {estimated_wait:.0f}s，过长，请稍后再试")

        self.queue.append(request)
        return QueueResult.queued(estimated_wait)
```

#### 优先级队列

将请求按优先级分类，高优先级请求优先处理：

| 优先级 | 类型 | 处理策略 | 队列容量 |
|--------|------|---------|---------|
| **Critical** | 系统告警、紧急干预 | 立即处理，跳过队列 | 5 |
| **High** | 付费用户、管理员 | 有限等待 (最多 5s) | 50 |
| **Normal** | 普通用户请求 | 正常排队 | 200 |
| **Low** | 批量处理、非实时 | 空闲时处理 | 500 |
| **Background** | 预计算、缓存预热 | 仅系统空闲 | 1000 |

#### Backpressure Mechanism（背压机制）

背压是队列溢出的反向传播信号——当下游处理不过来时，向上游发出"减速"信号：

```
无背压:
  [用户] ──高速发送──→ [Agent] ──高速调用──→ [LLM API]
                    请求堆积 → OOM                      ← LLM 返回 429

有背压:
  [用户] ←──减速信号── [Agent] ←──减速信号── [LLM API]
                      ↓                           ↓
                    限速器感知压力               返回 429 Too Many
                    主动降低接受率                Requests
```

---

### 6. Adaptive Rate Limiting: dynamic limits based on system load, user trust score, request complexity

自适应限速根据**系统实时状态**和**请求特征**动态调整阈值，避免静态限速的"一刀切"问题。

#### 动态因子

```
动态限速阈值 = 基准阈值 × 负载系数 × 信任系数 × 复杂度系数

  基准阈值:    系统预设的默认值 (如 100 RPM)
  负载系数:    当前系统负载 (CPU/内存/延迟) 0.0 ~ 1.0
  信任系数:    用户历史行为评分 0.5 ~ 2.0
  复杂度系数:  请求复杂度因子 0.5 ~ 1.5
```

#### 自适应调节算法

```python
class AdaptiveRateLimiter:
    """自适应速率限制器"""

    def __init__(self, base_limit: int = 100):
        self.base_limit = base_limit
        self.system_load = 0.5       # 0.0 (空闲) ~ 1.0 (过载)
        self.user_scores = {}        # user_id -> trust_score

    def get_dynamic_limit(self, user_id: str,
                          request: AgentRequest) -> int:
        # 1. 系统负载系数
        if self.system_load < 0.3:
            load_factor = 1.5        # 低负载: 放大配额
        elif self.system_load < 0.7:
            load_factor = 1.0        # 正常负载: 基准配额
        else:
            load_factor = 0.5        # 高负载: 缩减配额

        # 2. 用户信任系数
        trust_score = self.user_scores.get(user_id, 1.0)
        trust_factor = 0.5 + trust_score  # 0.5 (不信任) ~ 2.0 (高信任)

        # 3. 请求复杂度系数
        complexity = self._estimate_complexity(request)
        complexity_factor = 1.0 / (1 + complexity)  # 越复杂越低配额

        # 4. 综合计算
        dynamic_limit = int(
            self.base_limit *
            load_factor *
            trust_factor *
            complexity_factor
        )

        return max(dynamic_limit, 5)  # 最低不少于 5
```

#### 用户信任评分模型

```
信任评分维度:
  • 历史请求成功率（高成功 = 高信任）
  • 历史限流触发次数（频繁触发 = 低信任）
  • 账户年龄/验证状态（新账户 = 低信任）
  • IP 声誉（已知代理 = 低信任）
  • 行为模式异常度（与历史模式偏差大 = 低信任）

评分更新:
  每次请求 → 更新统计 → 重新计算信任分 → 调整阈值
```

---

### 7. Rate Limit Headers & Feedback: X-RateLimit headers, user notifications, retry-after strategies

良好的反馈机制让限流从"黑盒拒绝"变成"透明通信"。

#### 标准 Rate Limit Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100              # 窗口内总配额
X-RateLimit-Remaining: 87           # 剩余配额
X-RateLimit-Reset: 1625097600       # 配额重置时间 (Unix Timestamp)
Retry-After: 5                       # 等待秒数 (429 时返回)
```

#### Agent 增强 Headers

Agent 限速需要更丰富的反馈信息：

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1625097600
X-RateLimit-Scope: user             # 限速维度
X-RateLimit-Tokens-Remaining: 45000  # Token 配额剩余
X-RateLimit-Cost-Remaining: 2.50    # 成本预算剩余 ($)
X-RateLimit-Session-Budget: 60%     # 会话预算消耗百分比
X-RateLimit-Tier: standard          # 当前服务等级
```

#### 用户通知策略

| 场景 | 通知方式 | 示例消息 |
|------|---------|---------|
| **接近限流** | 温和提醒 | "您今天的请求已使用 80%，请注意使用频率" |
| **已触发限流** | 明确告知 | "请求频率过快，请 30 秒后重试" |
| **成本预算耗尽** | 提供替代方案 | "今日配额已用尽，明日恢复。或升级到 Pro 计划获得更多配额" |
| **降级通知** | 透明告知 | "当前系统负载较高，已为您切换至高效模式" |

#### Retry-After 策略

```
策略选项:

立即重试 (不推荐):                 带有退避的重试 (推荐):
  请求 ─→ 429 ─→ 立即重试 ─→ 429      请求 ─→ 429 ─→ 等 1s ─→ 429 ─→ 等 2s ─→ 429 ─→ 等 4s ─→ 200
         ↑ 循环直到限流窗口重置                    ↑ 指数退避 (Exponential Backoff)

指数退避公式:
  retry_delay = min(base_delay × 2^attempt, max_delay)
  示例: base=1s, max=60s
  Attempt 1: 1s    Attempt 2: 2s    Attempt 3: 4s
  Attempt 4: 8s    Attempt 5: 16s   Attempt 6: 32s
  Attempt 7+: 60s (上限)
```

---

## Example Code: Python RateLimiter with sliding window + token bucket + cost-based limiting

下面实现一个在生产环境中可用的综合限速器，同时支持滑动窗口（请求频率）、令牌桶（突发平滑）和基于成本的限制。

```python
"""
comprehensive_rate_limiter.py

一个面向 Agent 系统的综合速率限制器。
支持：
  - 滑动窗口 (请求频率限制)
  - 令牌桶 (允许突发请求)
  - 成本预算限制 (Token 和费用)
  - 自适应调节
  - 分布式 (Redis 后端)
"""

import time
import json
import logging
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional
from collections import deque
import threading

logger = logging.getLogger(__name__)


# ─── 数据类型 ────────────────────────────────────────────

class LimitScope(Enum):
    USER = "user"
    SESSION = "session"
    IP = "ip"
    TOOL = "tool"
    GLOBAL = "global"


@dataclass
class RateLimitConfig:
    """限速配置"""
    # 滑动窗口
    window_size: int = 60            # 时间窗口 (秒)
    max_requests: int = 100          # 窗口内最大请求数

    # 令牌桶
    bucket_capacity: int = 20        # 桶容量 (允许的突发大小)
    refill_rate: float = 5.0         # 每秒补充令牌数

    # 成本限制
    max_tokens_per_session: int = 100000   # 每会话最大 Token
    max_cost_per_session: float = 1.0      # 每会话最大成本 ($)
    max_cost_per_day_per_user: float = 10.0  # 每用户每日最大成本 ($)

    # 模型定价 (每 1K tokens)
    model_pricing: dict = field(default_factory=lambda: {
        "gpt-4":       {"input": 0.03,  "output": 0.06},
        "gpt-4-turbo": {"input": 0.01,  "output": 0.03},
        "gpt-3.5":     {"input": 0.001, "output": 0.002},
    })


@dataclass
class RateLimitResult:
    allowed: bool
    scope: Optional[LimitScope] = None
    remaining: int = 0
    reset_at: float = 0.0
    retry_after: float = 0.0
    reason: str = ""


# ─── 滑动窗口实现 ────────────────────────────────────────

class SlidingWindowCounter:
    """
    滑动窗口计数器——按时间权重计算窗口内请求数。
    相比 Fixed Window，无边界突发问题。
    """

    def __init__(self, window_size: int, max_requests: int):
        self.window_size = window_size
        self.max_requests = max_requests
        self._current_window_start = 0.0
        self._current_count = 0
        self._previous_count = 0
        self._lock = threading.Lock()

    def _current_window(self) -> float:
        return time.time() // self.window_size

    def allow(self) -> tuple[bool, int]:
        with self._lock:
            now = time.time()
            window_start = self._current_window()

            if window_start != self._current_window_start:
                # 进入新窗口：当前窗口变成上一个窗口
                self._previous_count = self._current_count
                self._current_count = 0
                self._current_window_start = window_start

            # 计算滑动窗口内的近似请求数
            elapsed = now - (window_start * self.window_size)
            weight = (self.window_size - elapsed) / self.window_size
            estimated = self._previous_count * weight + self._current_count

            allowed = estimated < self.max_requests
            remaining = max(0, self.max_requests - int(estimated))

            if allowed:
                self._current_count += 1

            next_reset = (window_start + 1) * self.window_size
            return allowed, remaining, next_reset


# ─── 令牌桶实现 ──────────────────────────────────────────

class TokenBucket:
    """
    令牌桶——以固定速率补充令牌，允许一定程度的突发。
    适合 Agent 场景：合法用户有时会密集发送请求(突发)，然后休息。
    """

    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second
        self._tokens = float(capacity)
        self._last_refill = time.time()
        self._lock = threading.Lock()

    def _refill(self):
        """补充令牌"""
        now = time.time()
        elapsed = now - self._last_refill
        new_tokens = elapsed * self.refill_rate
        self._tokens = min(self.capacity, self._tokens + new_tokens)
        self._last_refill = now

    def consume(self, tokens: int = 1) -> bool:
        """消耗 tokens 个令牌，返回是否成功"""
        with self._lock:
            self._refill()
            if self._tokens >= tokens:
                self._tokens -= tokens
                return True
            return False

    def remaining(self) -> float:
        with self._lock:
            self._refill()
            return self._tokens

    def wait_time(self) -> float:
        """获取需要等待多久才能获得一个令牌"""
        with self._lock:
            self._refill()
            if self._tokens >= 1:
                return 0.0
            return (1 - self._tokens) / self.refill_rate


# ─── 成本追踪器 ──────────────────────────────────────────

class CostTracker:
    """
    实时追踪 Token 消耗和金钱成本。
    提供会话级和每日预算检查。
    """

    def __init__(self, config: RateLimitConfig):
        self.config = config
        self._session_tokens: dict[str, int] = {}
        self._daily_costs: dict[str, float] = {}
        self._lock = threading.Lock()

    def record_usage(self, session_id: str, user_id: str,
                     model: str, input_tokens: int,
                     output_tokens: int) -> None:
        """记录一次调用的消耗"""
        with self._lock:
            # 会话 Token
            self._session_tokens[session_id] = \
                self._session_tokens.get(session_id, 0) + \
                input_tokens + output_tokens

            # 计算成本
            pricing = self.config.model_pricing.get(model, {})
            cost = (input_tokens / 1000) * pricing.get("input", 0) + \
                   (output_tokens / 1000) * pricing.get("output", 0)

            # 每日成本 (简化：使用当天日期字符串)
            today = time.strftime("%Y-%m-%d")
            key = f"{user_id}:{today}"
            self._daily_costs[key] = self._daily_costs.get(key, 0) + cost

    def check_session_budget(self, session_id: str) -> RateLimitResult:
        """检查会话预算"""
        with self._lock:
            used = self._session_tokens.get(session_id, 0)
            if used >= self.config.max_tokens_per_session:
                return RateLimitResult(
                    allowed=False,
                    scope=LimitScope.SESSION,
                    reason=f"会话 Token 超限: {used}/{self.config.max_tokens_per_session}"
                )
            remaining = self.config.max_tokens_per_session - used
            return RateLimitResult(
                allowed=True,
                scope=LimitScope.SESSION,
                remaining=remaining
            )

    def check_daily_budget(self, user_id: str) -> RateLimitResult:
        """检查每日预算"""
        with self._lock:
            today = time.strftime("%Y-%m-%d")
            key = f"{user_id}:{today}"
            cost = self._daily_costs.get(key, 0.0)
            if cost >= self.config.max_cost_per_day_per_user:
                return RateLimitResult(
                    allowed=False,
                    scope=LimitScope.USER,
                    reason=f"每日成本超限: ${cost:.2f}/${self.config.max_cost_per_day_per_user:.2f}"
                )
            remaining = int((self.config.max_cost_per_day_per_user - cost) * 100)
            return RateLimitResult(
                allowed=True,
                scope=LimitScope.USER,
                remaining=remaining
            )


# ─── 综合限速器 ─────────────────────────────────────────

class ComprehensiveRateLimiter:
    """
    综合限速器。
    按顺序检查: 成本 → 频率(Sliding Window) → 突发(Token Bucket)
    """

    def __init__(self, config: Optional[RateLimitConfig] = None):
        self.config = config or RateLimitConfig()
        self._window = SlidingWindowCounter(
            self.config.window_size,
            self.config.max_requests
        )
        self._bucket = TokenBucket(
            self.config.bucket_capacity,
            self.config.refill_rate
        )
        self._cost_tracker = CostTracker(self.config)
        self._session_locks: dict[str, threading.Lock] = {}

    def _get_lock(self, key: str) -> threading.Lock:
        if key not in self._session_locks:
            self._session_locks[key] = threading.Lock()
        return self._session_locks[key]

    def check_request(self, user_id: str, session_id: str,
                      ip: str, tool_name: str = "default",
                      model: str = "gpt-3.5") -> RateLimitResult:
        """
        综合限速检查入口。
        按 成本 → 频率 → 突发 的顺序依次检查。
        """
        # Layer 1: 成本检查 (最硬限制)
        cost_check = self._cost_tracker.check_session_budget(session_id)
        if not cost_check.allowed:
            return cost_check

        daily_check = self._cost_tracker.check_daily_budget(user_id)
        if not daily_check.allowed:
            return daily_check

        # Layer 2: 滑动窗口频率检查
        allowed, remaining, reset_at = self._window.allow()
        if not allowed:
            retry_after = max(1, int(reset_at - time.time()))
            return RateLimitResult(
                allowed=False,
                scope=LimitScope.USER,
                remaining=remaining,
                reset_at=reset_at,
                retry_after=retry_after,
                reason=f"请求频率超限，请 {retry_after}s 后重试"
            )

        # Layer 3: 令牌桶突发控制
        if not self._bucket.consume():
            wait = self._bucket.wait_time()
            return RateLimitResult(
                allowed=False,
                scope=LimitScope.TOOL,
                reason=f"突发请求受限，需等待 {wait:.1f}s",
                retry_after=wait
            )

        # 全部通过
        return RateLimitResult(
            allowed=True,
            remaining=remaining,
            reset_at=reset_at
        )

    def record_llm_call(self, session_id: str, user_id: str,
                        model: str, input_tokens: int,
                        output_tokens: int) -> None:
        """记录一次 LLM 调用的消耗"""
        self._cost_tracker.record_usage(
            session_id, user_id, model, input_tokens, output_tokens
        )

    def get_headers(self, user_id: str, session_id: str) -> dict:
        """
        生成 Rate Limit HTTP Headers 用于反馈。
        """
        remaining, _, reset_at = self._window.allow()
        session_budget = self._session_budget_remaining(session_id)

        return {
            "X-RateLimit-Limit": str(self.config.max_requests),
            "X-RateLimit-Remaining": str(remaining),
            "X-RateLimit-Reset": str(int(reset_at)),
            "X-RateLimit-Scope": "user",
            "X-RateLimit-Tokens-Remaining": str(
                self.config.max_tokens_per_session - session_budget
            ),
            "X-RateLimit-Session-Budget": f"{session_budget}%",
        }

    def _session_budget_remaining(self, session_id: str) -> int:
        """计算会话预算剩余百分比"""
        used = self._cost_tracker._session_tokens.get(session_id, 0)
        return max(0, int((1 - used / self.config.max_tokens_per_session) * 100))


# ─── 使用示例 ────────────────────────────────────────────

if __name__ == "__main__":
    # 初始化限速器
    limiter = ComprehensiveRateLimiter(RateLimitConfig(
        window_size=60,           # 60 秒窗口
        max_requests=100,         # 每分钟最多 100 请求
        bucket_capacity=20,       # 允许突发 20 个请求
        refill_rate=5.0,          # 每秒恢复 5 个
        max_tokens_per_session=100000,
        max_cost_per_session=1.0,
        max_cost_per_day_per_user=10.0,
    ))

    # 模拟用户请求
    user_id = "user_123"
    session_id = "session_abc"

    for i in range(150):  # 发送 150 个请求触发限流
        result = limiter.check_request(
            user_id=user_id,
            session_id=session_id,
            ip="192.168.1.1",
        )

        if result.allowed:
            # 模拟 LLM 调用消耗
            limiter.record_llm_call(
                session_id=session_id,
                user_id=user_id,
                model="gpt-3.5",
                input_tokens=500,
                output_tokens=200,
            )
            print(f"[{i:03d}] ✓ 通过 | 剩余: {result.remaining}")
        else:
            print(f"[{i:03d}] ✗ 拒绝 | {result.reason}")
            if result.retry_after > 0:
                print(f"    请等待 {result.retry_after}s 后重试")

        time.sleep(0.1)  # 模拟请求间隔
```

---

## Capability Boundaries: rate limiting can't stop determined distributed attacks

速率限制是重要的防御手段，但不是银弹。理解其局限性对设计完整安全方案至关重要。

### 速率限制无法防御的攻击类型

```
攻击类型                     速率限制能否防御?      原因
──────────────────────      ────────────────      ───────────────────
单个 IP 暴力攻击             ✓ 有效               单个源被限速
分布式 DoS (DDoS)           ✗ 无效               多个 IP 分散请求，单个 IP 不超限
慢速攻击 (Slow Rate)         ✗ 无效               攻击者维持低速率长期渗透
凭证填充 (Credential Stuffing) △ 部分有效          分布在不同账户可能绕过
应用层逻辑攻击               ✗ 无效               单个请求正常，但组合起来破坏系统
社会工程攻击                 ✗ 无效               不依赖速率
合法账户的恶意使用            ✗ 无效               请求在正常速率范围内
```

### 本质局限

1. **无状态攻击者无法区分**：人类使用脚本发送请求 vs 攻击者使用脚本发送请求，从速率模式上无法完全区分。

2. **分布式合谋**：攻击者可以控制 1000 个 IP，每个 IP 每分钟只发 1 个请求，永远不触发限速。

```
传统 DDoS:    10 IP × 10000 RPS = 被限速 ✓
分布式慢速:  10000 IP × 1 RPS = 不被限速 ✗
```

3. **合法突发误伤**：合法用户的"突发工作模式"（如赶报告、批量处理）在速率上看起来和攻击非常相似。

4. **限速器本身是攻击面**：如果攻击者可以探测到限速阈值，就可以故意触发限速来制造 DoS（让其他用户被误限）。

### 需要与其他机制配合

```
速率限制 + 其他安全机制:
  ┌──────────────────────────────────────┐
  │ 速率限制    → 应对单源高频率攻击         │
  │ 验证码      → 区分人类和机器             │
  │ WAF         → 检测应用层攻击模式          │
  │ 异常检测    → 发现分布式慢速攻击           │
  │ 行为分析    → 检测账户滥用                │
  │ 熔断        → 保护后端服务                │
  │ 配额管理    → 长期资源控制                │
  └──────────────────────────────────────┘
```

---

## Comparison: rate limiting vs throttling vs backpressure

这三个概念经常混淆，它们在 Agent 系统中有不同的作用层次和实现方式。

| 维度 | Rate Limiting (限速) | Throttling (节流) | Backpressure (背压) |
|------|---------------------|------------------|---------------------|
| **核心目标** | 防止滥用，保护系统 | 平滑流量，保证公平 | 信号传递，防止级联故障 |
| **作用位置** | 入口 (Ingress) | 处理管道 (Pipeline) | 整条链路 (End-to-End) |
| **触发条件** | 超过预设阈值 | 资源使用率过高 | 下游处理能力不足 |
| **行为方式** | 直接拒绝请求 | 降低处理速率/降级服务 | 向上游发送减速信号 |
| **粒度** | 请求级别 | 处理级别 | 系统级别 |
| **反馈形式** | 429 Too Many Requests | 延迟增加，质量下降 | 背压信号 (如 TCP 窗口) |
| **在 Agent 中的例子** | 用户超过 100 RPM 被拒绝 | LLM 调用从 GPT-4 降级到 3.5 | LLM API 响应变慢，Agent 自动降低请求频率 |

### 关系示意图

```
                       Rate Limiting
                      (入口守卫)
                          │
                    ┌─────▼──────┐
                    │   Agent    │
                    │   Service  │
                    └─────┬──────┘
                          │
                    Throttling
                   (管道节流控制)
                          │
                    ┌─────▼──────┐
                    │  LLM API   │
                    │  调用层    │
                    └─────┬──────┘
                          │
                   Backpressure
                   (背压信号反馈)
                          │
                    ┌─────▼──────┐
                    │  LLM API   │
                    │ (外部服务) │
                    └────────────┘
```

### 何时使用哪个

```python
# 场景 1: 新用户请求到达
if rate_limiter.is_rate_limited(user):  # Rate Limiting
    return 429 "请求过于频繁"              # 直接拒绝

# 场景 2: 系统资源紧张
if system.cpu_usage > 80:
    throttler.reduce_quality(user)       # Throttling
    # 降低模型等级: GPT-4 → GPT-3.5
    # 减少上下文: 全部 → 只保留最后 5 轮
    # 限制工具调用: 允许读但不允许写

# 场景 3: LLM API 响应变慢
if llm_api_latency > 5_000:
    backpressure.signal(                 # Backpressure
        source="llm_api",
        suggested_rate=10,               # 建议降速到 10 RPM
        severity="high"
    )
    agent.adjust_speed(10)               # Agent 主动降速
```

---

## Engineering Optimization: distributed rate limiting (Redis), pre-calculation, batch rate checking

生产环境的速率限制需要面对分布式部署、高并发和低延迟的挑战。

### 分布式速率限制 (Redis)

单机限速无法在微服务架构下工作——用户可能被不同的 Agent 实例处理，需要共享限速状态。

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Agent Node 1 │   │ Agent Node 2 │   │ Agent Node 3 │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       └─────────┬────────┘──────────┐
                  ▼                    ▼
           ┌──────────────┐
           │    Redis     │
           │  (共享状态)   │
           │              │
           │ window:1:cnt │
           │ window:1:ts  │
           │ user:123:cost│
           └──────────────┘
```

#### Redis 实现滑动窗口

```python
import redis as redis_client  # 使用 import alias 避免与模块名冲突

class RedisSlidingWindowLimiter:
    """
    基于 Redis Sorted Set 的分布式滑动窗口限速器。
    每个请求的时间戳作为一个 member，窗口内 member 计数。
    """

    def __init__(self, redis_host: str = "localhost",
                 window_size: int = 60, max_requests: int = 100):
        self.redis = redis_client.Redis(host=redis_host, decode_responses=True)
        self.window_size = window_size
        self.max_requests = max_requests

    def is_allowed(self, key: str) -> tuple[bool, int]:
        """
        检查请求是否允许。
        key 通常为 "rate_limit:{user_id}:{scope}"。
        """
        now = time.time()
        window_start = now - self.window_size

        # 使用 Redis 管道保证原子性
        pipe = self.redis.pipeline()

        # 清除窗口外的旧记录
        pipe.zremrangebyscore(key, 0, window_start)

        # 统计窗口内请求数
        pipe.zcard(key)

        # 添加当前请求
        pipe.zadd(key, {str(now): now})

        # 设置过期时间 (自动清理)
        pipe.expire(key, self.window_size + 10)

        # 执行管道
        _, count, _, _ = pipe.execute()

        # 判断是否超限 (减去刚添加的本次请求)
        allowed = (count < self.max_requests)
        remaining = max(0, self.max_requests - count - 1)

        return allowed, remaining
```

#### Redis Token Bucket (Lua 脚本实现原子操作)

```lua
-- token_bucket.lua
-- KEYS[1] = bucket key
-- ARGV[1] = capacity
-- ARGV[2] = refill_rate (tokens per second)
-- ARGV[3] = tokens to consume
-- ARGV[4] = current timestamp

local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local consume = tonumber(ARGV[3])
local now = tonumber(ARGV[4])

-- 获取当前令牌数和上次补充时间
local bucket = redis.call("hmget", key, "tokens", "last_refill")
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- 补充令牌
local elapsed = now - last_refill
local new_tokens = elapsed * refill_rate
tokens = math.min(capacity, tokens + new_tokens)

-- 判断是否足够
if tokens >= consume then
    tokens = tokens - consume
    redis.call("hmset", key, "tokens", tokens, "last_refill", now)
    redis.call("expire", key, math.ceil(capacity / refill_rate) + 60)
    return {1, tokens}  -- allowed, remaining
else
    redis.call("hmset", key, "tokens", tokens, "last_refill", now)
    redis.call("expire", key, math.ceil(capacity / refill_rate) + 60)
    local wait_time = (consume - tokens) / refill_rate
    return {0, wait_time}  -- denied, wait_time
end
```

```python
class RedisTokenBucket:
    """基于 Redis Lua 脚本的分布式令牌桶"""

    def __init__(self, redis_host: str = "localhost"):
        self.redis = redis_client.Redis(host=redis_host)
        self._script = self._load_script()

    def _load_script(self):
        # 加载 Lua 脚本到 Redis (一次加载，多次使用)
        with open("token_bucket.lua", "r") as f:
            return self.redis.register_script(f.read())

    def consume(self, key: str, capacity: int,
                refill_rate: float, tokens: int = 1) -> tuple[bool, float]:
        """
        尝试消耗令牌。
        返回 (是否允许, 剩余令牌数或等待时间)
        """
        result = self._script(
            keys=[key],
            args=[capacity, refill_rate, tokens, time.time()]
        )
        allowed = result[0] == 1
        return allowed, result[1]
```

### 预计算

对于高频限速检查，每次请求都完整计算的开销不可忽视。预计算优化思路：

```python
class PrecomputedRateLimiter:
    """
    预计算限速器。
    将部分计算结果缓存，减少实时计算开销。
    """

    def __init__(self, config: RateLimitConfig):
        self.config = config
        # 预计算不同时间粒度的配额
        self._precomputed = {
            "per_second": config.max_requests / config.window_size,
            "per_minute": config.max_requests,
            "per_hour": config.max_requests * (3600 / config.window_size),
            "per_day": config.max_requests * (86400 / config.window_size),
        }

    def quick_estimate(self, current_rate: float) -> bool:
        """
        快速估计是否接近限流阈值。
        不需要精确计数时的预检查。
        """
        # 如果当前速率远低于阈值，直接放行
        if current_rate < self._precomputed["per_second"] * 0.5:
            return True
        # 如果需要精确判断，才走完整计算
        return self._exact_check()
```

### 批处理速率检查

当系统需要同时处理大量请求时，批量检查比逐个检查高效得多：

```python
class BatchRateChecker:
    """
    批量速率检查器。
    同时检查多个请求的限速状态，减少 Redis 往返次数。
    """

    def __init__(self, redis_host: str = "localhost"):
        self.redis = redis_client.Redis(host=redis_host)

    def batch_check(self, requests: list[AgentRequest]) -> list[RateLimitResult]:
        """
        批量检查一组请求的限速状态。
        使用 Redis 管道一次往返完成所有检查。
        """
        pipe = self.redis.pipeline()
        now = time.time()
        window_start = now - 60  # 60s 窗口

        # 为每个请求准备检查命令
        for req in requests:
            key = f"rate_limit:{req.user_id}"
            pipe.zremrangebyscore(key, 0, window_start)
            pipe.zcard(key)

        # 一次执行所有检查
        results = pipe.execute()

        # 解析结果
        check_results = []
        for i, req in enumerate(requests):
            zcard_idx = i * 2 + 1  # 每个请求有两个命令
            count = results[zcard_idx]
            allowed = count < 100  # 假设 100 RPM
            remaining = max(0, 100 - count)
            check_results.append(
                RateLimitResult(
                    allowed=allowed,
                    remaining=remaining
                )
            )

        return check_results
```

### 性能对比

| 实现方式 | 延迟 (P50) | 延迟 (P99) | 吞吐量 | 适用规模 |
|---------|-----------|-----------|--------|---------|
| 单机内存 (threading) | 0.01ms | 0.05ms | 100K+/s | 单节点 |
| 单机内存 (asyncio) | 0.005ms | 0.02ms | 500K+/s | 单节点 |
| Redis (简单命令) | 0.5ms | 2ms | 50K/s | 中小集群 |
| Redis (管道) | 0.3ms/批次 | 1ms/批次 | 100K+/s | 中集群 |
| Redis (Lua 脚本) | 0.8ms | 3ms | 30K/s | 大集群 |
| Redis Cluster | 1-5ms | 10ms | 100K+/s | 超大集群 |

---

## 最佳实践总结

### Do's

| 实践 | 说明 |
|------|------|
| **软限流 + 硬限流结合** | 软限流发出警告，硬限流才拒绝请求 |
| **多维限速** | 不同维度 (用户/会话/IP/工具/成本) 互相补充 |
| **优先降级，其次拒绝** | 预算超限后降级到低成本模型，而不是直接拒绝 |
| **合理的错误反馈** | 明确告知用户为什么被限、何时恢复 |
| **分布式共识** | 多节点共享限速状态，避免单点不一致 |
| **自适应阈值** | 根据系统负载动态调整，提高资源利用率 |

### Don'ts

| 陷阱 | 后果 | 解决方案 |
|------|------|---------|
| 仅使用 Fixed Window | 窗口边界突发流量绕过限速 | 使用 Sliding Window |
| 单维度限速 | 攻击者可切换维度绕过 | 多维度联合限速 |
| 限速策略不透明 | 用户困惑，支持压力大 | 提供清晰的 Headers + 通知 |
| 忽略成本维度 | LLM API 费用失控 | 实现 Cost-Based Rate Limiting |
| 静态阈值不调整 | 高峰期 / 低峰期都不合适 | 自适应动态阈值 |
| 同步 Redis 调用阻塞 | 限速器本身成为性能瓶颈 | 使用异步 + 管道 + 本地缓存 |
| 忘记设置 TTL | Redis 内存泄漏 | 所有 Key 设置过期时间 |

---

## 关键认知

- **Agent 限速是多维游戏**：RPS、TPM、成本、工具调用、上下文深度——每一个维度都可能成为瓶颈
- **Cost-based 限速是 Agent 独有的防御维度**：传统 API 限速不需要考虑每次调用的金钱成本，但 Agent 系统必须
- **限速不是越严格越好**：误伤合法用户比承受少量滥用更糟糕（从业务角度）
- **队列比拒绝更友好**：能排队就不要拒绝，除非队列已满
- **自适应是上限**：能够根据负载动态调整的限速器，比任何精心调参的静态限速器更有效
- **限速器是防御链的一环**：不能单独依赖限速，需要与 WAF、异常检测、行为分析配合
- **分布式限速的瓶颈在 Redis**：每个请求一次 Redis 往返会显著增加延迟，管道和本地缓存是必要的优化
- **"限速"的不同层次**：Rate Limiting (拒绝) / Throttling (降级) / Backpressure (反馈) 三者在不同层面共同工作
