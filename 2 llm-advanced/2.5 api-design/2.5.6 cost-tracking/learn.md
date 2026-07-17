# API 调用成本追踪与预算控制

## 简单介绍

LLM API 按 Token 计费，Agent 系统可能发起大量 API 调用。成本追踪与预算控制是生产级 Agent 系统必须考虑的问题——不仅要监控花了多少钱，还要控制花钱的速度，避免成本失控。

## 成本计算基础

### 计费模型

| 模型 | 输入价格（每 1M Token） | 输出价格（每 1M Token） |
|------|----------------------|-----------------------|
| GPT-4o | $5 | $15 |
| GPT-4o-mini | $0.15 | $0.60 |
| Claude 3.5 Sonnet | $3 | $15 |
| Claude 3 Haiku | $0.25 | $1.25 |

### Agent 系统的成本构成

```python
def estimate_agent_call_cost(model: str, system_prompt: str, 
                              history: list, tool_defs: list,
                              output_tokens: int) -> float:
    """估算一次 Agent 调用的成本"""
    input_tokens = (
        count_tokens(system_prompt) +
        sum(count_tokens(str(m)) for m in history) +
        sum(count_tokens(str(t)) for t in tool_defs)
    )
    
    price = MODEL_PRICES[model]
    input_cost = input_tokens / 1_000_000 * price["input"]
    output_cost = output_tokens / 1_000_000 * price["output"]
    
    return input_cost + output_cost
```

## 成本追踪实现

### 1. Token 计数器

```python
class TokenCounter:
    """追踪 Token 消耗的装饰器"""
    def __init__(self):
        self.total_input_tokens = 0
        self.total_output_tokens = 0
        self.total_cost = 0.0
        self.call_count = 0
    
    def track(self, model: str):
        """返回一个用于追踪 API 调用的装饰器"""
        def decorator(fn):
            async def wrapper(*args, **kwargs):
                result = await fn(*args, **kwargs)
                
                # 从 API 响应中提取 Token 消耗
                input_tokens = result.usage.prompt_tokens
                output_tokens = result.usage.completion_tokens
                
                self.total_input_tokens += input_tokens
                self.total_output_tokens += output_tokens
                self.call_count += 1
                
                cost = calculate_cost(model, input_tokens, output_tokens)
                self.total_cost += cost
                
                # 记录日志
                logger.info(
                    f"[Cost] {model}: {input_tokens}in + {output_tokens}out "
                    f"= ${cost:.4f} | Total: ${self.total_cost:.2f}"
                )
                
                return result
            return wrapper
        return decorator
```

### 2. 预算控制器

```python
class BudgetController:
    """预算控制——防止成本失控"""
    
    def __init__(self, daily_budget: float = 10.0):
        self.daily_budget = daily_budget
        self.daily_spent = 0.0
        self.last_reset = date.today()
    
    async def check_budget(self, estimated_cost: float) -> bool:
        """检查是否还有预算"""
        self._maybe_reset()
        
        if self.daily_spent + estimated_cost > self.daily_budget:
            # 预算不足时触发降级
            logger.warning(
                f"Daily budget ${self.daily_budget:.2f} exceeded "
                f"(spent: ${self.daily_spent:.2f})"
            )
            return False  # 告诉调用者需要降级
        return True
    
    def record_cost(self, cost: float):
        self.daily_spent += cost
    
    def _maybe_reset(self):
        if date.today() != self.last_reset:
            self.daily_spent = 0.0
            self.last_reset = date.today()
```

### 3. 完整成本追踪系统

```python
class CostTracker:
    """完整的 API 成本追踪系统"""
    
    def __init__(self):
        self.counters = defaultdict(TokenCounter)
        self.budget = BudgetController(daily_budget=50.0)
        
        # 持久化存储
        self.storage = JSONStorage("cost_logs.json")
        self._load_history()
    
    def record(self, model: str, input_tokens: int, output_tokens: int, tags: dict = None):
        cost = calculate_cost(model, input_tokens, output_tokens)
        
        record = {
            "timestamp": time.time(),
            "model": model,
            "input_tokens": input_tokens,
            "output_tokens": output_tokens,
            "cost": cost,
            "tags": tags or {}
        }
        
        self.budget.record_cost(cost)
        self.history.append(record)
        self._save()  # 每 10 条写一次磁盘
    
    def get_stats(self) -> dict:
        """获取成本统计摘要"""
        total_cost = sum(r["cost"] for r in self.history)
        today_cost = sum(
            r["cost"] for r in self.history
            if datetime.fromtimestamp(r["timestamp"]).date() == date.today()
        )
        
        model_breakdown = defaultdict(float)
        for r in self.history:
            model_breakdown[r["model"]] += r["cost"]
        
        return {
            "total_cost": total_cost,
            "today_cost": today_cost,
            "total_calls": len(self.history),
            "model_breakdown": dict(model_breakdown),
            "budget_remaining": self.budget.daily_budget - today_cost
        }
```

## 成本优化策略

| 策略 | 节省 | 影响 | 说明 |
|------|------|------|------|
| Prompt 精简 | 10-30% | 无 | 删除冗余指令 |
| 模型路由 | 30-70% | 简单任务用小模型 | 复杂度感知路由 |
| 缓存 | 20-50% | 完全无影响 | 语义缓存常见问题 |
| 历史压缩 | 20-40% | 部分信息丢失 | 摘要压缩历史 |
| Batch API | 50% | 延迟增加 | 离线任务 |

## Agent 成本可视化

追踪"每个任务的成本"有时比"总成本"更有价值：

```python
class TaskCostTracker:
    """按任务追踪成本"""
    
    def __init__(self):
        self.tasks = {}
    
    def start_task(self, task_id: str, task_type: str):
        self.tasks[task_id] = {
            "type": task_type,
            "start": time.time(),
            "calls": [],
            "total_cost": 0
        }
    
    def record_call(self, task_id: str, call_info: dict):
        task = self.tasks[task_id]
        task["calls"].append(call_info)
        task["total_cost"] += call_info["cost"]
    
    def get_expensive_tasks(self, top_n: int = 5) -> list:
        """找出最昂贵的任务"""
        sorted_tasks = sorted(
            self.tasks.values(),
            key=lambda t: t["total_cost"],
            reverse=True
        )
        return sorted_tasks[:top_n]
```

## 核心挑战

| 挑战 | 说明 | 解决方案 |
|------|------|----------|
| 成本不可预测 | 每次调用的 Token 消耗不同 | 实时追踪 + 预算告警 |
| 多模型成本混合 | 不同模型价格不同 | 按模型细分追踪 |
| 缓存节省难衡量 | 缓存命中节省了多少成本 | 记录缓存 hit/miss |
| 长流程成本累积 | 一个 Agent 任务可能涉及很多调用 | 按任务 ID 聚合 |
| Token 膨胀 | 工具结果/历史对话的持续增长 | 定期压缩和清理 |

## 工程启示

- **成本追踪应该在开发阶段就引入**——后期再加成本控制非常痛苦
- 设置**硬预算限制**（每日/每月上限）和**软告警**（到达 80% 预算时告警）
- 对非关键路径启用"成本模式"——自动使用更便宜的模型
- 缓存是最划算的优化——相同/相似输入的 80% 都命中缓存
- 成本追踪数据也可用于容量规划——了解系统对 API 的真实消耗模式
