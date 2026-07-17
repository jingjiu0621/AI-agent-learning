# Token Compression —— LLM Agent 系统的令牌压缩实践

> **核心挑战**：每次 LLM API 调用的输入令牌中，大量内容是历史上下文、系统指令和工具描述中冗余或低信息密度的部分，Token Compression 的目标是在最小化质量损失的前提下，通过选择性移除、语义压缩和自适应截断来降低每次请求的令牌消耗，从而直接削减 API 成本。

---

## 1. 基本原理

### 1.1 Token 成本构成

在 LLM Agent 系统中，一次请求的成本由输入与输出令牌共同决定。以当前主流定价模型（如 GPT-4o、Claude Sonnet）为例：

```
API 单次调用成本 = Input_Tokens × Input_Price + Output_Tokens × Output_Price
```

典型的令牌消耗分布：

```
| 组成部分          | 占比范围   | 说明                           |
|-------------------|-----------|--------------------------------|
| System Prompt     | 15% - 30% | 角色指令、行为约束、输出格式     |
| Message History   | 40% - 60% | 多轮对话、工具调用链、中间结果   |
| Tool Descriptions | 10% - 25% | 工具 Schema、参数说明、使用示例   |
| User Query        | 5% - 15%  | 本轮用户输入                     |

输入令牌通常占总成本的 **60%-70%**，且多轮交互中 Message History 会持续膨胀。Token Compression 本质上是在**输入侧**做减法。

### 1.2 压缩的质量曲线

压缩率与回答质量之间存在非线性关系：

```
回答质量
  ^
  |  ████████████████████░░ 无损区 (压缩 0-20%)
  |  ████████████████░░░░░░ 可接受区 (压缩 20-50%)
  |  ██████████░░░░░░░░░░░░ 风险区 (压缩 50-70%)
  |  ████░░░░░░░░░░░░░░░░░░ 衰减区 (压缩 70%+)
  +-------------------------------> 压缩率
```

关键观察：**存在一个"压缩悬崖"（Compression Cliff）—— 一旦压缩率超过某个任务特定的阈值，回答质量会急剧下降。** 这一阈值因任务类型而异（见第 4 节）。

### 1.3 压缩的经济效益模型

```
节省成本 = (原始输入令牌数 × 压缩率 × 输入单价 × 调用次数) - 压缩计算开销
```

当压缩计算开销（如使用小模型做摘要或过滤）显著低于节省的 API 成本时，压缩在经济上是净收益的。实践表明，对于上下文密集的 Agent 应用，压缩通常能带来 **30%-50% 的成本降低**，而质量损失可控制在 5% 以内。

---

## 2. 背景与演进

### 2.1 为什么需要 Token Compression

LLM 的上下文窗口虽然在快速增长（4K -> 8K -> 32K -> 100K -> 200K），但出现了新的问题：

- **成本与窗口成正比增长**：更大的窗口意味着更高的单次调用成本
- **"中间信息丢失"问题**：Lost in the Middle 研究表明，模型对长上下文中间位置的信息关注度显著降低
- **延迟增加**：长上下文导致首令牌时间（TTFT）增加
- **Agent 多轮累积**：工具调用链会产生大量中间输出，快速填满上下文

### 2.2 技术演进路线

```
时期           方法                          典型工具/技术
─────          ────                          ────────────
早期 (2022)    固定窗口截断                   直接保留最后 N 轮
中期 (2023)    摘要式压缩                      GPT-4 Summarization, Map-Reduce
后期 (2023)    选择性过滤                      LLMLingua, Selective Context
当前 (2024+)   自适应 + 多策略组合             动态压缩率、分层压缩、缓存感知压缩
```

### 2.3 与 Prompt Caching 的关系

Token Compression 与 Prompt Caching（13.3 节）是互补关系：

```
                      输入文本
                      /      \
                     /        \
                    /          \
         Prompt Caching       Token Compression
         (缓存重复前缀)        (移除冗余内容)
              |                     |
              v                     v
        减少重复计费           减少内容总量
        缓存命中时 ≈ 0 计费    每轮都降低基数
```

最佳实践：先压缩，再利用缓存的计费优惠，二者叠加效果最大。

---

## 3. 核心技术与实现

### 3.1 压缩的四个维度

```
┌─────────────────────────────────────────────────┐
│                 Token Compression                │
│                                                   │
│  System Prompt ──→ 精简指令 + 模板优化             │
│  Message Hist. ──→ 窗口截断 / 摘要 / 关键事件      │
│  Tool Desc.    ──→ 选择性传递 / 压缩描述 / 缓存     │
│  Output Token  ──→ max_tokens + 结构化约束         │
└─────────────────────────────────────────────────┘
```

#### 维度一：System Prompt 压缩

System Prompt 往往是最大的单次固定开销。压缩策略包括：

1. **去重与合并**：移除冗余的角色声明、重复的格式指令
2. **指令蒸馏**：将多条短指令合并为一条精炼指令
3. **模板优化**：移除不必要的空白、注释、示例

```
原始 System Prompt（约 1500 tokens）：
  "You are a helpful assistant. You must respond in JSON format.
   Always output valid JSON. The JSON must have 'action' and
   'parameters' fields. Please make sure to always use JSON..."

压缩后（约 800 tokens）：
  "You are a JSON-only assistant. Output: {"action": string, "parameters": object}"
```

#### 维度二：Message History 压缩

最常见的压缩发力点。三种主流策略：

```
策略 A：滑动窗口
  [Round 1] [Round 2] ... [Round N-5] [Round N-4] ... [Round N]
                                  └──── 保留最近 5 轮 ────┘

策略 B：摘要压缩
  [Round 1-10 摘要] [Round 11-20 摘要] [Round 21-30 原始]

策略 C：关键事件提取
  从完整历史中提取：{用户意图切换点, 工具调用结果, 错误信息}
  └── 舍弃中间推理步骤
```

#### 维度三：Tool Description 压缩

当注册的工具数量较多时，工具 Schema 会占用大量输入令牌。

- **选择性传递**：只传递与当前用户查询相关的工具
- **短名称优化**：使用简短但语义明确的工具名和参数名
- **描述缓存**：将完整的工具描述放在缓存中，只在上下文中传递工具名称

#### 维度四：Output Token 控制

虽然输出令牌单价通常低于输入令牌，但对成本也有显著影响：

- 使用 `max_tokens` 限制最大生成长度
- 使用结构化输出（JSON mode / Tool use）避免冗长的自由文本
- 使用 Stop Sequences 在不必要时提前终止生成

### 3.2 技术方案分类

```
技术                类型       压缩率    质量影响    计算开销
──────────────────  ────────   ──────   ────────   ──────
Token 计数 + 截断    lossless   10-20%   低         极小
Prompt 模板优化      lossless   10-15%   低         无
空白/格式标准化      lossless   5-10%    极低        无
语义重要性过滤       lossy      40-60%   中低        中
LLMLingua            lossy      40-70%   中          高
摘要式压缩           lossy      60-80%   中高        高
自适应压缩           adaptive   20-60%   低-中       中-高
```

**Lossless**：信息完全保留，质量零损失。适合固定指令和模板。
**Lossy**：信息有损失，但大幅降低令牌数。适合历史消息和上下文。
**Adaptive**：根据任务复杂度动态决定压缩率。适合通用场景。

### 3.3 代码实现

#### 示例 1：基于 Token 计数的消息截断器（Python）

```python
import tiktoken

class TokenTruncator:
    """基于 token 计数的最简截断器"""

    def __init__(self, model: str = "gpt-4"):
        self.encoder = tiktoken.encoding_for_model(model)

    def count_tokens(self, text: str) -> int:
        return len(self.encoder.encode(text))

    def truncate_messages(
        self,
        messages: list[dict],
        system_prompt: str,
        max_total_tokens: int,
        reserve_ratio: float = 0.3,
    ) -> list[dict]:
        """
        优先保留 System Prompt + 最新消息。

        Args:
            messages: 完整消息列表
            max_total_tokens: 允许的最大令牌数
            reserve_ratio: 为本轮输出预留的令牌比例
        """
        output_budget = int(max_total_tokens * reserve_ratio)
        input_budget = max_total_tokens - output_budget

        system_tokens = self.count_tokens(system_prompt)
        remaining = input_budget - system_tokens

        if remaining <= 0:
            raise ValueError("System prompt alone exceeds budget")

        # 从最新消息向前选择
        selected = []
        used = 0

        for msg in reversed(messages):
            content = msg.get("content", "")
            tokens = self.count_tokens(content)
            if used + tokens <= remaining:
                selected.insert(0, msg)
                used += tokens
            else:
                # 部分截断：保留消息的开头部分
                truncated = self._truncate_text(content, remaining - used)
                if truncated:
                    selected.insert(0, {**msg, "content": truncated})
                break

        return selected

    def _truncate_text(self, text: str, max_tokens: int) -> str:
        tokens = self.encoder.encode(text)
        if len(tokens) <= max_tokens:
            return text
        return self.encoder.decode(tokens[:max_tokens])
```

#### 示例 2：基于摘要的历史消息压缩器（Python）

```python
import asyncio
from dataclasses import dataclass

@dataclass
class CompressedHistory:
    summary: str
    recent_messages: list[dict]
    total_tokens_saved: int

class SummarizationCompressor:
    """
    使用小模型（或轻量 LLM）对历史消息做摘要压缩。
    将多轮对话压缩为一段摘要 + 保留最近 K 轮原始消息。
    """

    def __init__(self, summarize_client, keep_last_rounds: int = 3):
        self.client = summarize_client
        self.keep_last_rounds = keep_last_rounds

    async def compress(
        self,
        messages: list[dict],
        max_summary_tokens: int = 512,
    ) -> CompressedHistory:
        if len(messages) <= self.keep_last_rounds + 1:
            # 消息太少，无需压缩
            return CompressedHistory(
                summary="",
                recent_messages=messages,
                total_tokens_saved=0,
            )

        # 分为"可摘要部分"和"保留部分"
        summarizable = messages[:-(self.keep_last_rounds)]
        keep = messages[-(self.keep_last_rounds):]

        # 生成摘要
        summary = await self._summarize_rounds(summarizable, max_summary_tokens)

        tokens_before = sum(len(m.get("content", "").split())
                          for m in summarizable)
        tokens_after = len(summary.split())
        saved = tokens_before - tokens_after

        return CompressedHistory(
            summary=f"[Previous conversation summary]: {summary}",
            recent_messages=keep,
            total_tokens_saved=max(0, saved),
        )

    async def _summarize_rounds(
        self, messages: list[dict], max_tokens: int
    ) -> str:
        text = "\n".join(
            f"{m['role']}: {m.get('content', '')}"
            for m in messages
        )
        prompt = (
            "Compress the following conversation into a brief summary "
            f"(max {max_tokens} tokens). Keep key decisions, results, "
            "and user intent. Omit pleasantries and failed attempts:\n\n"
            f"{text}"
        )
        response = await self.client.complete(prompt)
        return response
```

#### 示例 3：基于关键词的工具选择器（TypeScript）

```typescript
interface ToolDefinition {
  name: string;
  description: string;
  parameters: Record<string, unknown>;
  keywords: string[];
  inputTokens: number; // 该工具描述的预估令牌数
}

class SelectiveToolPasser {
  private tools: ToolDefinition[];

  constructor(tools: ToolDefinition[]) {
    this.tools = tools;
  }

  /**
   * 根据用户查询的关键词匹配，只传递相关的工具。
   * 不匹配的工具完全不出现在请求中。
   */
  selectTools(userQuery: string): ToolDefinition[] {
    const queryLower = userQuery.toLowerCase();
    const queryTokens = new Set(queryLower.split(/\s+/));

    const scored = this.tools.map((tool) => {
      let score = 0;
      for (const kw of tool.keywords) {
        if (queryLower.includes(kw.toLowerCase())) {
          score += kw.length;
        }
      }
      // 即使没有关键词匹配，也保留名称包含查询词的工具
      if (queryTokens.has(tool.name.toLowerCase())) {
        score += 10;
      }
      return { tool, score };
    });

    // 按匹配度降序，只选择高于阈值的工具
    const THRESHOLD = 2;
    const selected = scored
      .filter((s) => s.score >= THRESHOLD)
      .sort((a, b) => b.score - a.score)
      .map((s) => s.tool);

    // 若完全无匹配，至少保留一个最通用的工具
    if (selected.length === 0 && this.tools.length > 0) {
      const sortedByTokens = [...this.tools].sort(
        (a, b) => a.inputTokens - b.inputTokens
      );
      return [sortedByTokens[0]];
    }

    return selected;
  }

  /**
   * 计算压缩节省的令牌数
   */
  estimatedTokenSavings(
    allTools: ToolDefinition[],
    selectedTools: ToolDefinition[]
  ): number {
    const allTokens = allTools.reduce((s, t) => s + t.inputTokens, 0);
    const selectedTokens = selectedTools.reduce((s, t) => s + t.inputTokens, 0);
    return allTokens - selectedTokens;
  }
}
```

---

## 4. 能力边界

### 4.1 Token Compression 做不到什么

- **无法保留所有语义细节**：Lossy 压缩必然丢失一部分信息，特别是跨上下文的隐式关联和细微情感色彩
- **无法保证零质量下降**：压缩后的上下文可能在边缘案例中导致模型遗漏关键信息
- **无法替代良好的 Prompt 设计**：压缩不能修复结构不良的指令，压缩应当在 Prompt 优化之后进行
- **无法完全自动化**：不同任务类型的最优压缩策略不同，需要针对性地调参

### 4.2 压缩悬崖（Compression Cliff）

不同任务类型的安全压缩阈值：

```
任务类型               安全压缩率   悬崖阈值    悬崖后的表现
────────────────────  ──────────  ──────────  ─────────────
简单信息提取            0-60%       70%+       字段遗漏
代码生成                0-40%       50%+       类型错误、语法错误
多步推理                0-35%       45%+       推理链断裂
创意写作                0-25%       35%+       风格连贯性丧失
工具调用编排            0-30%       40%+       参数选择错误
```

### 4.3 压缩的副作用

| 副作用 | 表现 | 缓解措施 |
|--------|------|----------|
| 上下文碎片化 | 模型丢失事件间因果关系 | 保留关键事件的时间戳和因果链标注 |
| 指令稀释 | 模型的角色一致性下降 | 仅压缩历史消息，不压缩 System Prompt |
| 循环压缩偏差 | 多次压缩后信息失真逐级放大 | 保留原始消息作为摘要的校验参考 |
| 延迟引入 | 压缩过程本身增加耗时 | 使用轻量模型做压缩，或预计算 |

---

## 5. 与其他策略对比

Token Compression 是成本优化工具箱中的一个组件。以下是各策略的定位对比：

```
策略              成本降低     质量影响    实现复杂度    适用场景
────────────────  ─────────    ────────   ──────────   ──────────
Token Compression  30-50%      低-中      中           所有场景，尤其长上下文
Model Routing      40-70%      低         高           简单 / 复杂任务混合
Prompt Caching     50-90%*     无         低           重复前缀固定的场景
Request Batching   50%**       无         中           高吞吐、异步场景
Output Truncation  10-20%      中         低           输出长度不受控的场景

* 缓存命中时节省比例，并非总成本的全局节省
** 指 batch API 的折扣价差
```

### 5.1 联合使用策略

```
优化的 Agent 请求生命周期：

  1. 用户查询到达
  2. 查询路由：Model Routing 选择适当的模型（简单 → 小模型，复杂 → 大模型）
  3. 上下文组装：
     a. 消息历史 → Token Compression（摘要 + 截断）
     b. 工具描述 → Selective Tool Passing（仅保留相关工具）
     c. System Prompt → 模板优化（lossless 压缩）
  4. 缓存检查：如果 System Prompt 相同，使用 Prompt Caching
  5. API 调用
  6. 输出收到 → 检查是否超过 max_tokens，截断输出
```

### 5.2 成本优化的 Pareto 优先级

基于投入产出比的推荐实施顺序：

```
优先级 1: Prompt Caching        ← 零实现成本，立竿见影
优先级 2: Message 窗口截断       ← 简单实现，效果显著
优先级 3: System Prompt 精简     ← 一次性工作，持续受益
优先级 4: 选择性工具传递          ← 需关键词工程，中投入
优先级 5: 摘要式压缩              ← 需引入小模型，高投入
优先级 6: 自适应压缩              ← 需监控和在线学习
```

---

## 6. 工程优化方向

### 6.1 监控先行

在实施任何压缩策略之前，先建立 Token 消耗的可观测性：

```python
# 需要监控的关键指标
METRICS = {
    "prompt_tokens_per_call": 0,      # 每次调用的输入令牌数
    "completion_tokens_per_call": 0,  # 每次调用的输出令牌数
    "total_cost_per_call": 0.0,       # 每次调用的成本
    "compression_ratio": 0.0,         # 实际压缩率
    "cost_saved": 0.0,               # 与未压缩基准相比节省的成本
    "quality_score": 0.0,             # 用户反馈 / eval 分数
    "compression_time_ms": 0,         # 压缩本身的延迟
}
```

### 6.2 分层压缩架构（推荐）

```
                    ┌──────────────┐
                    │  Compressor  │
                    │   Orchestrator│
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              v            v            v
        ┌─────────┐ ┌──────────┐ ┌──────────┐
        │ Lossless │ │  Lossy   │ │Adaptive  │
        │ Stage   │ │  Stage   │ │Controller│
        └─────────┘ └──────────┘ └──────────┘
              │            │            │
              v            v            v
        ┌─────────────────────────────────┐
        │       Quality Monitor           │
        │   (退火回退 + 压缩率调优)         │
        └─────────────────────────────────┘
```

### 6.3 安全回退策略

为了控制压缩带来的质量风险，建议实现以下安全机制：

1. **退火回退（Fallback）**：当模型在压缩后的上下文中无法完成任务时，自动降级为低压缩率或零压缩重试
2. **A/B 对比**：对同一任务以不同压缩率并行调用，选择置信度高的结果
3. **渐进式压缩**：从低压缩率开始，若质量指标保持稳定则逐步提高压缩率
4. **白名单机制**：对特定任务（如金融计算、医疗诊断）禁用 Lossy 压缩

### 6.4 实践清单

```
Token Compression 实施的十个检查项：

[ ] 1. 使用 tiktoken / tokenizers 库准确估算各部分的令牌数
[ ] 2. 对 System Prompt 做 lossless 压缩（去重、简化、合并指令）
[ ] 3. 对 Message History 实施滑动窗口（保留最近 N 轮）
[ ] 4. 评估摘要压缩的 ROI（引入小模型的开销 vs 节省的成本）
[ ] 5. 实现选择性工具传递（基于关键词匹配）
[ ] 6. 对输出设置合理的 max_tokens
[ ] 7. 建立压缩率 - 质量关系基线（每个任务类型单独测试）
[ ] 8. 实现压缩回退策略（质量下降时自动降低压缩率）
[ ] 9. 监控压缩引入的额外延迟（目标：< 100ms）
[ ] 10. 定期校准压缩策略（模型升级后重新测试阈值）
```

---

> **总结**：Token Compression 是 LLM Agent 成本优化中最直接有效的策略之一。最佳实践遵循"先无损后有损"的原则——先用低风险的 lossless 压缩收割 10-20% 的节省，再根据任务特点选择性引入 lossy 压缩以获得 40-60% 的深度节省。关键要在每个任务类型上建立"压缩率 vs 质量"的基线曲线，找到各自的压缩悬崖阈值，并在压缩管线中加入质量监控和自动回退机制。
