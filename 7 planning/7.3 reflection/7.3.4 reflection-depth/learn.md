# 7.3.4 Reflection Depth — 反思深度控制：表层纠正 vs 深层重构

## 简单介绍

反思深度决定了 Agent 反思的"粒度"——是仅仅纠正当前步骤的错误（表层纠正），还是从根本上重构推理策略（深层重构）。不同的问题需要不同深度的反思，深度控制是反思效率的关键。

## 基本原理

反思可分为三个深度层次：

| 层次 | 关注点 | 行为变化 | 成本 |
|------|--------|---------|------|
| L1 表层纠正 | 当前步骤的参数错误 | 调整参数重试 | 低 |
| L2 策略调整 | 当前方法的有效性 | 更换工具或策略 | 中 |
| L3 深层重构 | 整个推理框架的正确性 | 重新思考任务理解 | 高 |

```
L1 反思: "我参数写错了，数据库名应该是 'prod' 不是 'dev'"
→ 修正参数，重试

L2 反思: "直接查询数据库不够，应该先检查数据新鲜度"
→ 增加前置步骤：先检查更新时间

L3 反思: "我完全理解错了任务——用户要的不是原始数据而是分析报告"
→ 回到任务起点，重新规划
```

## 背景

Reflexion 论文中的反思主要是 L2 级别的——Agent 回顾执行轨迹并思考"哪里做得不对，下次应该怎么做"。但实际工程中发现，很多简单的错误（如拼写参数名）不需要完整的 L3 深层反思，用 L1 级别快速修正即可。分层反思深度是工程优化的自然产物。

## 之前针对这个问题的做法与结果

1. **统一深度反思**：所有错误都用同一份反思 prompt。结果：简单错误过度反思（浪费 token），复杂错误反思深度不够。
2. **仅 L1 处理**：只重试错误操作，不做任何策略分析。结果：效率高但无法解决策略性问题。
3. **仅 L3 处理**：每次出错都从任务理解开始重新规划。结果：彻底但效率极低，简单错误也被复杂化。

## 核心矛盾

**反思的充分性 vs 效率**：深层反思更可能找到根本原因，但成本高；表层反思成本低，但可能治标不治本。关键在于用"足够深"的反思解决当前问题，而不是用"最深的"反思应对所有情况。

## 当前主流优化方向

1. **错误分类驱动的深度选择**：先对错误分类（参数错误/工具错误/逻辑错误/理解错误），再根据错误类型选择合适的反思深度。
2. **渐进式深度递增**：先尝试 L1 反思，如果问题仍然存在升级到 L2，再不行到 L3。这保证了"最小必要成本"。
3. **领域特定的深度预设**：在某些领域（如金融交易），默认需要 L3 深度反思，因为错误代价高；在其他领域（如简单问答），默认 L1 即可。
4. **深度-成功率历史**：记录每种错误类型的"反思深度→修复成功率"数据，用历史数据指导深度选择。

## 实现的最大挑战

```python
class DepthControlledReflection:
    def reflect(self, error_context: ErrorContext, 
                depth: str = "auto") -> Reflection:
        
        if depth == "auto":
            depth = self._select_depth(error_context)
        
        if depth == "L1":
            return self._surface_reflection(error_context)
        elif depth == "L2":
            return self._tactic_reflection(error_context)
        elif depth == "L3":
            return self._deep_reflection(error_context)
    
    def _select_depth(self, ctx: ErrorContext) -> str:
        """自动选择反思深度"""
        # 如果是简单参数错误 → L1
        if ctx.error_type in ["parameter_error", "format_error"]:
            return "L1"
        
        # 如果是工具调用失败但方法正确 → L2
        if ctx.error_type in ["tool_error", "timeout"]:
            return "L2"
        
        # 如果是逻辑错误、理解错误或反复失败 → L3
        if ctx.error_type in ["logic_error", "comprehension_error"]:
            return "L3"
        
        # 历史数据驱动
        depth_history = self._get_depth_history(ctx.error_type)
        if depth_history:
            return depth_history.best_depth
        
        return "L2"  # 默认中等深度
    
    def _surface_reflection(self, ctx: ErrorContext) -> Reflection:
        prompt = f"""
        工具调用返回了错误。
        错误: {ctx.error_message}
        参数: {ctx.parameters}
        
        请只检查参数是否正确，给出修正建议。
        不需要分析整个策略。
        """
        return self.llm.predict(prompt)
    
    def _deep_reflection(self, ctx: ErrorContext) -> Reflection:
        prompt = f"""
        任务目标: {ctx.task}
        完整轨迹: {ctx.trajectory}
        最终结果: {ctx.result}
        
        请从任务理解开始重新审视整个过程：
        1. 我对任务的理解是否正确？
        2. 我选择的方法是否适合这个任务？
        3. 有没有完全不同的方法可以尝试？
        4. 我的推理框架有什么根本性缺陷？
        """
        return self.llm.predict(prompt)
```

最大挑战是**错误类型的准确分类**：LLM 或规则分类器可能把"逻辑错误"错分类为"参数错误"，导致选择了过浅的反思深度，治标不治本。缓解方案：如果 L1 反思执行后问题仍然存在，自动升级到 L2/L3。

## 能力边界和结果边界

- **能力**：自适应深度选择可以在保持高修复率的同时减少 30-50% 的反思成本
- **边界**：某些错误的根本原因不在现场可见的信息中（如外部 API 变更导致的错误），再深的反思也无法发现
- **结果结构**：L1 输出"修正后的参数"，L2 输出"策略调整方案"，L3 输出"完整的任务重理解"

## 核心优势

深度控制将反思的"成本-收益"公式最优化——不再一刀切地用最重的方式处理所有问题，而是根据问题的实际复杂度匹配最合适的反思强度。

## 工程优化方向

- 构建"错误类型→反思深度"的映射知识库，随着使用经验积累自动优化
- 实施反思深度审计：定期回顾历史反思决策，检查是否存在"深度不足"导致的复发问题
- 对 L3 深层反思的结果做持久化，因为 L3 反思通常能产生可复用的通用经验
- 在 L3 反思后，自动更新 System Prompt 或行为规则，以永久消除同类错误

## 适合场景

- 错误类型多样的生产系统
- 需要在反思成本和质量之间优化的场景
- 作为 Reflexion 循环的智能控制组件
