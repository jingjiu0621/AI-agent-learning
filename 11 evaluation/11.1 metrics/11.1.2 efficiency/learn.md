# 11.1.2 efficiency — 效率指标：步数 / Token 消耗 / 耗时

## 简单介绍

效率指标衡量 Agent 完成任务时消耗的资源和时间。一个 Agent 即使能 100% 完成任务，如果每一步都要想 30 秒、消耗 10 万 Token，在生产环境中也是不可接受的。效率评估关注三个核心维度：**计算成本（Token）**、**时间成本（延迟）**和**步骤成本（调用次数）**。

## 基本原理

### 效率三维度

```
效率 = f(资源消耗, 时间消耗, 步骤效率)

资源消耗 = 总 Token 数（输入 + 输出）
时间消耗 = 端到端延迟（P50/P95/P99）
步骤效率 = 完成任务所需步数
```

### 效率与质量的关系

```
高效率 + 高质量 ─── 理想状态
高效率 + 低质量 ─── Agent 在偷懒（跳过必要步骤）
低效率 + 高质量 ─── Agent 过度思考，成本不可控
低效率 + 低质量 ─── 系统性问题，需要重构
```

## 背景

传统的 API 性能评估很简单：延迟 P99 < 200ms，吞吐量 > 1000 QPS，就够了。但 Agent 的效率评估远比这复杂：

- 一次 Agent 请求可能包含 3-15 次 LLM 调用
- 每次 LLM 调用的延迟取决于输出长度和模型大小
- 工具调用的外部延迟不可控（API 慢、数据库查询慢）
- 不同的任务复杂度天然需要不同的步数和 Token 量

## 核心矛盾

**效率不能独立于任务复杂度衡量。** 一个 3 步完成的简单查询和一个 15 步完成的多步推理任务，步数相差 5 倍，但后者并不是"低效"的。效率指标必须结合任务难度来解读。

## 主流效率指标

### 1. 步数效率

```python
class StepEfficiency:
    """Agent 步数效率评估"""
    
    def __init__(self):
        self.step_data = []
    
    def record_step(self, step_type: str, is_necessary: bool = True):
        """记录每一步"""
        self.step_data.append({
            "type": step_type,  # "thought" | "tool_call" | "observation"
            "is_necessary": is_necessary,
            "timestamp": time.time()
        })
    
    def calculate_metrics(self, task_complexity: int) -> dict:
        """
        计算步数效率指标
        task_complexity: 预估最小步数（专家预估）
        """
        total_steps = len(self.step_data)
        necessary_steps = sum(1 for s in self.step_data if s["is_necessary"])
        unnecessary_steps = total_steps - necessary_steps
        
        return {
            "total_steps": total_steps,
            "necessary_steps": necessary_steps,
            "unnecessary_steps": unnecessary_steps,
            "step_ratio": total_steps / max(task_complexity, 1),
            # step_ratio < 1.5 → 高效
            # step_ratio 1.5-2.5 → 可接受
            # step_ratio > 2.5 → 效率需要优化
            "efficiency_grade": "high" if total_steps <= task_complexity * 1.5
                          else "medium" if total_steps <= task_complexity * 2.5
                          else "low",
            "tool_call_ratio": sum(1 for s in self.step_data 
                                   if s["type"] == "tool_call") / total_steps
        }
```

### 2. Token 消耗分析

```python
class TokenEfficiency:
    """Token 消耗效率评估"""
    
    def analyze(self, task_messages: list) -> dict:
        """
        分析消息历史的 Token 效率
        每个 message: {"role": str, "content": str, "tokens": int}
        """
        total_input = sum(m["tokens"] for m in task_messages 
                         if m["role"] in ("system", "user"))
        total_output = sum(m["tokens"] for m in task_messages 
                          if m["role"] == "assistant")
        tool_tokens = sum(m["tokens"] for m in task_messages 
                         if m["role"] == "tool")
        
        # Token 利用率
        # 有效 Token = 最终输出中直接贡献于任务完成的 Token
        useful_output = self._estimate_useful_tokens(task_messages)
        
        return {
            "total_tokens": total_input + total_output + tool_tokens,
            "input_tokens": total_input,
            "output_tokens": total_output,
            "tool_tokens": tool_tokens,
            "input_output_ratio": total_input / max(total_output, 1),
            # 正常范围 3:1 ~ 10:1
            # 过高 → Prompt 太长但输出太短
            # 过低 → Agent 在 generate 而不是 reason
            
            "token_efficiency": useful_output / max(total_output, 1),
            # > 0.7 → 高效
            # < 0.4 → 大量 Token 浪费在无关内容
            
            "estimated_cost": self._calculate_cost(
                total_input, total_output, model="gpt-4")
        }
    
    def _estimate_useful_tokens(self, messages: list) -> int:
        """估算有效 Token（简化版）"""
        # 实际实现可以用 LLM 评估或规则统计
        final_output = messages[-1].get("content", "")
        # 去除格式标记、重复内容等
        cleaned = self._remove_boilerplate(final_output)
        return len(cleaned.split()) * 1.3  # 粗略估算
```

### 3. 延迟指标

```python
class LatencyMetrics:
    """延迟指标追踪"""
    
    def __init__(self):
        self.latencies = []
        self.phase_latencies = {
            "thought": [],
            "tool_call": [],
            "observation_parse": [],
            "llm_inference": []
        }
    
    def record_phase(self, phase: str, duration_ms: float):
        """记录各阶段耗时"""
        self.phase_latencies[phase].append(duration_ms)
        self.latencies.append(duration_ms)
    
    def report(self) -> dict:
        """输出延迟报告"""
        def percentile(data, p):
            sorted_data = sorted(data)
            idx = int(len(sorted_data) * p / 100)
            return sorted_data[min(idx, len(sorted_data) - 1)]
        
        return {
            "end_to_end": {
                "p50": percentile(self.latencies, 50),
                "p95": percentile(self.latencies, 95),
                "p99": percentile(self.latencies, 99),
                "avg": sum(self.latencies) / len(self.latencies),
                "max": max(self.latencies)
            },
            "by_phase": {
                phase: {
                    "p50": percentile(lats, 50),
                    "p95": percentile(lats, 95),
                    "avg": sum(lats) / len(lats)
                }
                for phase, lats in self.phase_latencies.items()
                if lats
            },
            "llm_calls": len(self.phase_latencies["llm_inference"]),
            "tool_calls": len(self.phase_latencies["tool_call"])
        }
```

## 效率评估的挑战

### 1. Token 效率 vs 质量效率的冲突

```
详细的 Thought 过程   →  Token 消耗高，但质量可能更高
精简的 Thought 过程   →  Token 消耗低，但可能遗漏关键推理

例：
  Agent 写代码时，详细的思考过程：
  "需要先检查输入是否为空，然后用二分查找..." → 400 Tokens ✅ 质量高
  
  精简版本：
  "写二分查找" → 50 Tokens ❌ 可能漏边界处理
  
平衡策略：按任务复杂度动态调整 Thought 详细程度
```

### 2. 步骤数的"效率悖论"

```
场景：Agent A 用 2 步完成任务但结果错误，
      Agent B 用 5 步完成任务但结果正确。

Agent A 步数更少但失败，Agent B 步数更多但成功。
"效率"不能只看步数，要结合成功率一起看。

解决方案：效率调整后的成功率
adjusted_efficiency = success_rate / avg_steps
```

### 3. 外部依赖延迟不可控

```
Agent 调用外部 API：
  LLM 推理延迟:   2s  (可控，优化 KV Cache 等)
  工具调用延迟:   0.5s (可控，本地工具)
  外部 API 延迟:  3-10s (不可控，依赖第三方服务)
  知识库检索延迟: 1-5s (部分可控，取决于索引优化)
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 衡量 Token/步数/延迟的消耗量 | 判断哪些消耗是"必要的" |
| 比较不同 Agent 版本的效率 | 跨任务类型直接对比效率 |
| 识别异常的 Token/时间消耗 | 自动优化效率（需要人工决策） |
| 建立效率基线和预算 | 在不影响质量的前提下仅优化效率 |

## 与其他指标的关系

- **成功率**：效率必须结合成功率一起看，只追求效率 = 鼓励 Agent 偷懒
- **鲁棒性**：高鲁棒性（重试、验证）通常会降低效率
- **可靠性**：一致性检查会增加步骤，降低效率
- **综合评分**：效率是综合评分中的"成本优化"维度

## 工程优化方向

1. **模型路由**：简单任务用小模型（快且便宜），复杂任务用大模型
2. **缓存策略**：语义缓存减少重复 LLM 调用
3. **并行工具调用**：无依赖的工具同时执行
4. **动态步数限制**：根据任务复杂度自动设置最大步数
5. **成本预算**：每次请求设置 Token 预算上限，超预算回退

## 确定适用场景

效率指标对以下场景尤为重要：
- 生产环境（每毫秒和每 Token 都是成本）
- 高并发场景（延迟直接影响用户体验）
- 大规模部署（Token 成本线性增长）
- 实时交互系统（用户等待时间敏感）
