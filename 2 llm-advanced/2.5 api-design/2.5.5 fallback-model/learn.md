# 模型降级回退策略

## 简单介绍

模型降级（Fallback Model）是指当主力模型不可用（如服务中断、限流、超时）或成本过高时，自动切换到备选模型的策略。这是生产级 Agent 系统提高可靠性的关键模式——不把系统可靠性绑定在单一模型上。

## 为什么需要降级

| 场景 | 影响 | 降级方案 |
|------|------|----------|
| 主力模型服务中断 | 全部请求失败 | 切换到备用模型 |
| API 限流满载 | 请求被延迟/拒绝 | 切换到低配额模型 |
| 高并发场景 | 成本飙升 | 切换到更便宜的模型 |
| 简单任务 | 大模型浪费 | 用快速小模型处理 |

## 降级策略

### 1. 硬降级（基于错误）

```python
class ModelRouter:
    """基于错误检测的模型降级"""
    
    def __init__(self):
        self.primary = "gpt-4o"
        self.fallback = "gpt-4o-mini"
        self.current = self.primary
    
    async def chat(self, messages, tools=None):
        try:
            return await self._call(self.current, messages, tools)
        except (ServerError, RateLimitError) as e:
            # 主力模型不可用，自动降级
            if self.current == self.primary:
                logger.warning(f"主力模型错误，降级到 {self.fallback}: {e}")
                return await self._call(self.fallback, messages, tools)
            raise
```

### 2. 策略路由（基于任务复杂度）

```python
class TaskModelRouter:
    """基于任务复杂度的智能路由"""
    
    def __init__(self):
        self.routes = {
            "simple":      "gpt-4o-mini",    # 简单任务：快速便宜
            "normal":      "gpt-4o-mini",    # 一般任务
            "complex":     "gpt-4o",         # 复杂任务：需要能力
            "critical":    "gpt-4o",         # 关键任务：最高质量
        }
    
    async def route(self, task_type: str, messages, tools=None):
        model = self.routes.get(task_type, self.routes["normal"])
        
        try:
            return await llm.chat(model, messages, tools)
        except Exception as e:
            # 如果当前模型失败，尝试更强的模型
            if model != self.routes["critical"]:
                fallback_model = self._get_stronger(model)
                return await llm.chat(fallback_model, messages, tools)
            raise
    
    def _get_stronger(self, current: str) -> str:
        """获取比当前更强的模型"""
        strength = ["gpt-4o-mini", "gpt-4o", "claude-3-5-sonnet"]
        try:
            idx = strength.index(current)
            if idx < len(strength) - 1:
                return strength[idx + 1]
        except ValueError:
            pass
        return "gpt-4o"
```

### 3. 成本感知路由

```python
class CostAwareRouter:
    """考虑 Token 成本的路由"""
    
    COST_PER_1K = {
        "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
        "gpt-4o": {"input": 0.005, "output": 0.015},
        "claude-3-haiku": {"input": 0.00025, "output": 0.00125},
        "claude-3-5-sonnet": {"input": 0.003, "output": 0.015},
    }
    
    BUDGET_CENTS = 10  # 每次请求预算 10 美分
    
    async def route_by_budget(self, input_tokens: int, complexity: str):
        """基于 Token 预算和复杂度路由"""
        # 估算所需输出 Token
        estimated_output = self._estimate_output(complexity)
        
        for model, costs in sorted(self.COST_PER_1K.items(), 
                                   key=lambda x: x[1]["input"]):
            total_cost = (input_tokens * costs["input"] + 
                         estimated_output * costs["output"]) / 1000 * 100
            
            if total_cost <= self.BUDGET_CENTS and self._model_capable(model, complexity):
                return model
        
        return "gpt-4o"  # 默认
```

## 降级时机

| 触发条件 | 策略 | 示例 |
|---------|------|------|
| API 返回 429/5xx | 立即切换到备用模型 | gpt-4o → gpt-4o-mini |
| 连续 N 次失败 | 切换并冷却主力模型 | 暂停使用主力 5 分钟 |
| 响应时间 > 阈值 | 切换到响应更快的模型 | 10s 无响应则切换 |
| 每日预算用完 | 切换到更经济的模型 | gpt-4o → gpt-4o-mini |

## 可用性设计

```python
class ModelHighAvailability:
    """多模型高可用设计"""
    def __init__(self):
        self.models = [
            {"name": "gpt-4o", "provider": "openai", "weight": 10},
            {"name": "gpt-4o-mini", "provider": "openai", "weight": 8},
            {"name": "claude-3-5-sonnet", "provider": "anthropic", "weight": 9},
        ]
        self.failures = {}
    
    async def get_best_model(self) -> str:
        """获取当前最佳可用模型"""
        available = []
        for m in self.models:
            # 检查最近失败率
            recent_failures = self.failures.get(m["name"], [])[-10:]
            failure_rate = sum(recent_failures) / max(len(recent_failures), 1)
            
            if failure_rate < 0.3:  # 失败率低于 30% 的模型可用
                available.append(m)
        
        # 按权重选模型
        total_weight = sum(m["weight"] for m in available)
        r = random.uniform(0, total_weight)
        for m in available:
            r -= m["weight"]
            if r <= 0:
                return m["name"]
        
        return available[0]["name"] if available else self.models[0]["name"]
```

## 核心挑战

1. **行为一致性**：不同模型的输出质量、风格不同——降级后用户体验可能变化
2. **状态同步**：切换到新模型后，对话上下文需要保持
3. **降级检测延迟**：检测到故障 + 切换的过程有延迟，影响用户体验
4. **成本控制**：切换到更强的模型（如 gpt-4o → claude-sonnet）可能导致成本超预期

## 工程启示

- 降级策略应该**渐进式**：小模型 → 同级别不同模型 → 更强模型
- 对交互式 Agent，首选"快速失败 + 降级"而非"长时间重试"
- 记录降级事件——用于分析主力模型的可信度
- 不同 Provider 的模型作为灾备可以防止"单点故障"
- 降级后应通知用户（"当前使用备用模型，某些能力可能受限"）
- 不要忽略降级模型的成本——便宜的模型可能更贵（因为需要更多 Token 或更多步骤）
