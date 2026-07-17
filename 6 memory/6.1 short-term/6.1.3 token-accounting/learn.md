# 6.1.3 token-accounting — Token 精确计算、预留与回收

## 简单介绍

Token Accounting 是对上下文窗口中每个消息、每个工具调用、每个检索结果进行精确 Token 计数和管理的机制。核心任务是在构造每一次 LLM 调用时，知道"已经用了多少、还剩多少、新内容占多少"——同时为模型的输出预留足够的空间。

## 基本原理

### 核心等式

```
总上下文窗口 = C (由模型 API 定义，如 8192, 32768, 131072, 200000)

一次 API 调用的 Token 构成：
  C_total = C_input + C_output + C_overhead

其中：
  C_input   = 输入 Token（用户消息 + 系统指令 + 工具定义 + 历史 + 检索内容）
  C_output  = 输出 Token（模型回复 + 工具调用声明）
  C_overhead = API 内部开销（消息格式、角色标记等）

核心约束：
  C_total <= 模型的 max_tokens 限制
  C_output 必须预留足够的空间（否则回复会被截断）
```

### Token 预留机制

预留的关键在于：给模型输出分配的空间不够时，回复会被强行截断，导致：
- 工具调用 JSON 被截断 → 解析失败
- 推理链被截断 → 结论不完整
- 对话结束语被截断 → 用户体验差

```
┌──────────────────────────────────────────────┐
│                上下文窗口                      │
├──────────────┬──────────────────┬────────────┤
│  输入内容     │  预留输出空间     │  安全缓冲  │
│  (消息+工具)  │  (max_tokens)    │  (5-10%)   │
├──────────────┴──────────────────┴────────────┤
│ ← 已占用                                       │
│                               ← 可用空间 →     │
└──────────────────────────────────────────────┘

预留量决策：
  C_reserved = min(C_available * 0.2, fixed_max)
  其中 C_available = C_window - C_input
  fixed_max 通常设为 2048 或 4096
```

### Token 精确计算流程

```python
class TokenAccountant:
    def __init__(self, model: str, max_context: int):
        self.encoder = tiktoken.encoding_for_model(model)
        self.max_context = max_context
        self.max_output = min(4096, max_context // 4)  # 默认预留
        self.running_total = 0

    def count_tokens(self, text: str) -> int:
        """精确计算文本的 Token 数（不是字符数）"""
        return len(self.encoder.encode(text))

    def count_message(self, msg: dict) -> int:
        """计算一条消息的 Token 数（含格式开销）"""
        tokens = 0
        # 消息本身的角色标记、格式开销
        tokens += 4  # 消息格式基准开销
        tokens += self.count_tokens(msg.get("content", ""))
        tokens += self.count_tokens(msg.get("name", ""))
        # tool_calls 额外计算
        for tc in msg.get("tool_calls", []):
            tokens += self.count_tokens(tc["function"]["name"])
            tokens += self.count_tokens(tc["function"]["arguments"])
            tokens += 2  # id 和 type 字段开销
        return tokens

    def get_available(self) -> int:
        """计算可用输入空间"""
        return self.max_context - self.max_output - self.running_total

    def can_add(self, content: str) -> bool:
        """检查是否可以添加新内容"""
        needed = self.count_tokens(content)
        return self.running_total + needed + self.max_output <= self.max_context

    def reserve_output(self, tokens: int):
        """动态调整输出预留"""
        self.max_output = min(tokens, self.max_context // 2)  # 不超过一半
```

## 背景与演进

### 早期做法：字符计数

最早的 Agent 实现使用字符长度（len(text)）来估算上下文使用情况。

**问题**：
- 中文字符 1 字 ≈ 2-3 Tokens，代码中 1 Token ≈ 3-4 字符
- 字符计数误差高达 200%-300%
- 工具调用、特殊格式的开销完全被忽略

### 演进路径

```
字符计数 → 简单单词计数 → 模型特定 Tokenizer → 精确消息级核算 → 动态预留
   误差高      不够精确        tiktoken/            含格式开销      自适应调整
                            transformers
```

### current state

主流方案（OpenAI、Anthropic SDK）都提供了内置的 Token 计数方法。但 Agent 框架层面的精确核算仍需要自行实现，因为 SDK 不自动管理"多个消息组合后还剩多少空间"。

## 核心矛盾

**精度 vs 计算开销**：每次调用 Tokenizer 编码都有计算成本。在长对话中，每次新增消息都重新编码整个历史来统计 Token 会显著增加延迟。但如果不精确计数，超限风险就会增加。

具体表现为：
1. **增量计数 vs 全量重算**：增量累加有累积误差，全量重算开销大
2. **预留量动态调整**：预留太多浪费空间，预留太少截断风险
3. **输出长度不确定性**：模型实际输出长度无法提前精确预测

## 主流优化方向

| 方向 | 方法 | 效果 |
|------|------|------|
| **增量 Token 计数** | 只对新消息编码，累加到缓存总量 | 减少重复计算 90%+ |
| **自适应预留** | 根据历史输出长度动态调整预留量 | 空间利用率提升 10-20% |
| **Token 预算窗口** | 用滑动窗口跟踪最近 N 轮的 Token 消耗模式 | 更精确地预测未来消耗 |
| **格式开销预知** | 预先计算消息模板的固定格式开销 | 提高计数精度约 5-10% |
| **异步 Token 审计** | 后台线程定期重新统计全量，纠正累积误差 | 保持精度同时不阻塞主流程 |

### 增量计数实现

```python
class IncrementalTokenCounter:
    def __init__(self, model):
        self.encoder = tiktoken.encoding_for_model(model)
        self.total = self._count_system_overhead()  # 初始系统开销
        self.message_cache = {}  # msg_id -> token_count

    def add_message(self, msg_id: str, msg: dict) -> int:
        count = self._count_message(msg)
        self.message_cache[msg_id] = count
        self.total += count
        return count

    def remove_message(self, msg_id: str):
        if msg_id in self.message_cache:
            self.total -= self.message_cache[msg_id]
            del self.message_cache[msg_id]

    def audit(self, messages: list) -> int:
        """全量审计，纠正累积误差"""
        actual = sum(self._count_message(m) for m in messages)
        self.total = actual
        return actual
```

## 产生的结果

- **零超限事故**：精确计数确保永远不会意外超出上下文窗口
- **空间利用率提升**：精确知道还剩多少空间，可以"塞满"而不是"保守预留"
- **截断率降低**：输出预留精确后，模型回复被截断的概率降低 50%+
- **计算开销增加**：每次 API 调用前增加约 5-50ms 的 Token 计数时间（取决于历史长度）

## 最大挑战

**跨模型的 Tokenizer 兼容性**：不同模型使用不同的 Tokenizer（甚至同一提供商的不同模型也不同）。一个针对 GPT-4 优化的计数方案切换到 Claude 后可能需要完全重写。Agent 框架如果支持多模型，需要维护一个 Tokenizer 适配层。

## 能力边界与结果边界

| 维度 | 能做的 | 不能做的 |
|------|--------|----------|
| 精确计数 | 精确计算已有内容的 Token 数 | 无法预测模型会输出多少 Token |
| 空间预留 | 为模型输出预留可配置的空间 | 无法保证预留空间刚好够用 |
| 误差校正 | 全量审计消除累积误差 | 审计本身有计算成本 |
| 跨模型 | 支持主流模型的 Tokenizer | 每个新模型需要单独适配 |

## 与相关概念的区别

| 概念 | 区别 |
|------|------|
| **上下文预算 (6.1.1)** | Token 会计提供"数据"（用了多少），预算做"决策"（怎么分配）。会计在先，预算在后 |
| **截断策略 (6.1.4)** | Token 会计判断是否超限，截断策略决定超限后怎么办 |
| **Tokenizer** | Tokenizer 是编码工具，Token 会计是管理机制——会计调用 Tokenizer 来获取度量值 |
| **max_tokens 参数** | API 参数，限制单次输出的最大 Token 数——会计用它做预留但不等同于此 |

## 核心优势

1. **消除不确定性**：Agent 的行为不再因为"不知道还剩多少空间"而出现随机失败
2. **空间利用率最大化**：精确知道剩余空间，可以"恰到好处"地填充内容
3. **故障预防**：在 API 调用前就发现潜在的超限问题，而非在调用失败后才处理

## 工程优化建议

### 实现要点

1. **离线 Tokenizer 缓存**：常用文本（系统指令、工具定义）的 Token 计数提前计算并缓存，避免重复编码
2. **消息级索引**：给每条消息分配 ID，维护 Token 计数索引，支持快速增删
3. **周期性全量审计**：每 50 条消息或每次截断后，执行一次全量重算以纠正累积误差
4. **输出长度预测模型**：基于历史数据训练一个简单的输出长度预测器（如"历史平均输出 × 1.2"），比固定预留更高效

### 具体代码模式

```python
class TokenAccountingSystem:
    def __init__(self, model: str, context_limit: int):
        self.counter = IncrementalTokenCounter(model)
        self.context_limit = context_limit
        self.output_reserve = 2048  # 默认 2K 预留

    def prepare_request(self, messages: list, new_content: str = "") -> dict:
        """准备 API 请求，确保不会超限"""
        # 1. 检查可用空间
        available = self.context_limit - self.output_reserve - self.counter.total
        new_tokens = self.counter.encoder.encode(new_content)

        if len(new_tokens) > available:
            # 2. 空间不够，触发截断
            self._trigger_truncation(messages)

        # 3. 动态调整输出预留
        estimated_output = self._estimate_output_length()
        self.output_reserve = min(estimated_output, self.context_limit // 3)

        return {
            "max_tokens": self.output_reserve,
            "messages": messages
        }
```

## 适合场景判断

| 场景 | 适用性 | 说明 |
|------|--------|------|
| 所有 Agent 应用 | 强制 | 没有精确计数的 Agent 就像没有油表的车 |
| 长对话场景 | 强制 | 误差会随对话轮次累积，必须定期审计 |
| 工具密集型 Agent | 强制 | 工具定义和调用结果的 Token 消耗大且变化多 |
| 多模型部署 | 强烈推荐 | 不同 Tokenizer 导致切换模型时必须重新适配 |
| 资源受限环境 | 推荐 | 每个 Token 都宝贵，需要精确管理 |
| 单次简单调用 | 可选 | 消息量小、窗口大的场景不计数也不会出问题 |
