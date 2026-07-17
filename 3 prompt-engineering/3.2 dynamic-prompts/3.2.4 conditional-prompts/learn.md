# 3.2.4 Conditional Prompts — 条件分支 Prompt 策略

## 简单介绍

Conditional Prompts（条件分支 Prompt）是一种根据输入条件动态选择或组合不同 Prompt 片段的技术。它是传统编程中 "if-else" / "switch-case" 在 Prompt 工程中的对应物。在 AI Agent 开发中，用户请求的多样性意味着没有任何单一 Prompt 能高效处理所有场景。条件分支策略允许 Agent 系统根据当前上下文（用户意图、输入模式、角色、安全要求等）选择最优的 Prompt 配置。

## 基本原理

条件分支的核心是将 Prompt 选择逻辑从 Prompt 内容中分离出来。系统首先评估一组条件（基于用户输入、系统状态、外部信号等），然后根据评估结果选择执行路径。每条路径对应一组不同的 Prompt 片段配置，这些片段可以是独立模板，也可以是同一模板的不同参数化版本。

```python
class ConditionalPromptRouter:
    def __init__(self):
        self.routes = []

    def add_route(self, condition_fn, prompt_template, route_name=""):
        """condition_fn: (context) -> bool 判断是否走这条路由"""
        self.routes.append({
            "condition": condition_fn,
            "template": prompt_template,
            "name": route_name,
        })

    def resolve(self, context):
        """返回第一个条件匹配的路由"""
        for route in self.routes:
            if route["condition"](context):
                return route["template"].render(**context)
        # fallback 路由
        return self.fallback_template.render(**context)


# 使用示例：根据用户意图路由不同 Prompt
router = ConditionalPromptRouter()
router.add_route(
    condition_fn=lambda ctx: ctx.get("intent") == "code_review",
    prompt_template=code_review_prompt,
    route_name="code_review",
)
router.add_route(
    condition_fn=lambda ctx: ctx.get("intent") == "debugging",
    prompt_template=debug_prompt,
    route_name="debugging",
)
router.add_route(
    condition_fn=lambda ctx: ctx.get("intent") == "refactoring",
    prompt_template=refactor_prompt,
    route_name="refactoring",
)
```

## 背景

在早期的 Prompt 工程实践中，开发者倾向于构建一个"全能 Prompt"，试图在一个 Prompt 中覆盖所有可能的用户请求类型。这种做法的典型产物就是包含大量 if-else 式自然语言指令的 System Prompt——"如果用户问 A，则做 X；如果用户问 B，则做 Y"。但 LLM 对这种"自指"指令的处理并不稳定，尤其是当条件增多、指令复杂度上升时，模型很容易遗漏或误解某些分支。

## 之前针对这个问题的做法与结果

**做法一：大型全能 System Prompt** — 在一个固定的 System Prompt 中写入所有分支逻辑。缺点是 Prompt 长度飙升（影响性能和成本）；模型在长 Prompt 中迷失；修改一个分支需要更新整个 Prompt，回滚困难。

**做法二：规则引擎前置分类** — 在调用 LLM 之前，使用规则或传统 NLP 方法判断用户意图，然后选择对应的 Prompt 模板。比全能 Prompt 更可控，但规则维护成本高，且规则之外的边缘情况容易被遗漏。

**做法三：意图分类 + LLM 混合** — 先用一个轻量 Prompt 或小模型做意图分类，再根据分类结果选择对应的 Prompt 模板。这是当前的主流方案，但由于分类本身由 LLM 完成，存在分类错误导致误入错误分支的风险。

## 核心矛盾

**分类精度与分支粒度之间的矛盾**。分支越多、分类越细，对用户请求的处理越精准，但意图分类的准确率越低——因为类别之间的边界变窄，分类器（无论是规则还是 LLM）更容易出错。一旦分类错误，LLM 会使用完全错误的 Prompt，输出质量远不如"用万能 Prompt 给出一般性回答"。因此条件分支策略的核心问题不是"能不能分"，而是"分错后果是否可接受"。

## 当前主流优化方向

1. **置信度门控** — 不直接使用分类器的 top-1 结果，而是设置置信度阈值。高于阈值走精确分支，低于阈值走通用 fallback Prompt。这大幅降低了"分错后结果极差"的风险。

2. **层次化分支** — 先分大类（如"代码相关"vs"非代码"），再在大类内细分（"代码相关"内再分"审查/调试/编写"）。每层的分类任务更简单、准确率更高，且错误的影响范围更局限。

3. **提示内联条件** — 在模板中嵌入 Jinja2/Mustache 条件语句，实现"模板级别的条件分支"。这种方式不需要在代码中显式路由，由渲染引擎在模板内部完成分支选择。

4. **混合分支** — 将确定性规则（正则匹配关键词、正则表达式）与 LLM 分类结合。对高频、明确的请求用规则快速路由，对模糊、复杂的请求用 LLM 判断。规则的零延迟和零错误让简单场景得到极速处理。

```python
def hybrid_classifier(user_input):
    # 确定性规则：快速路由
    if re.search(r"(fix|bug|error|crash)", user_input, re.I):
        return "debugging", 0.99  # 高置信度
    if re.search(r"^(write|create|implement|add)", user_input, re.I):
        return "code_gen", 0.99

    # LLM 分类：模糊请求
    intent = classify_with_llm(user_input)  # 返回 (category, confidence)
    if intent.confidence < 0.6:
        return "general", 1.0  # 走通用分支
    return intent.category, intent.confidence
```

## 实现的最大挑战

**分支衰减（Branch Decay）** 是最大挑战。随着系统演进，分支数量会不可控地增长。每个新场景增加一个新分支，导致：（1）分类器需要处理的类别越来越多，准确率持续下降；（2）多个分支之间可能覆盖彼此，产生冲突；（3）测试覆盖所有分支路径的成本指数级增长。解决方案包括定期合并相似分支、弃用低使用率分支、使用分层结构而非平面结构。

## 能力边界与结果边界

Conditional Prompts 能显著提升"已知场景"的处理质量，但对"未知场景"的处理能力取决于 fallback 分支的质量。条件分支系统本质上是一个"封闭集合"系统——它只能处理预先定义好的分支，无法自动发现和适应全新的场景。此外，分支之间的"灰色地带"（某些请求同时匹配多个分支）是永远的难题。

## 与其他技术的区别

与 Prompt Chaining 的区别：Chaining 是"按顺序执行多个步骤"，Conditional 是"按条件选择不同路径"。Chaining 中也可以包含 Conditional 步骤（如"根据上一步结果决定走哪条分支"）。与 Template Engine 的区别：Template Engine 在模板内部做条件判断（"如果变量 X 存在，则显示 Y"），Conditional 在代码层面做全局分支决策（"选择哪个 Prompt 模板"）。两者可以嵌套使用。

## 核心优势

- **精准匹配**：每个场景使用为它量身定制的 Prompt，输出质量远高于通用 Prompt
- **隔离性**：修改一个分支的 Prompt 不影响其他分支，降低变更风险
- **效率**：分支级别使用不同模型（简单分支用小模型、复杂分支用大模型），优化成本和延迟
- **可扩展**：新场景可以添加新分支，无需重写现有逻辑

## 工程优化方向

1. **分支监控面板**：实时追踪各分支的调用量、成功率、平均延迟，识别冷门和故障分支
2. **分支合并检测**：自动检测两个分支的 Prompt 内容和输入特征是否高度相似，提示合并
3. **自动 fallback 增强**：当 fallback 分支被频繁触发时，自动分析这些请求的共性，建议创建新分支
4. **分支权重调优**：根据历史数据自动调整分支选择参数（如置信度阈值、路由优先级）

## 适合场景的判断标准

以下情况适合引入条件分支：（1）用户请求的类型差异巨大，单一 Prompt 难以兼顾（如同时有"代码审查"和"产品咨询"）；（2）不同场景需要不同的系统约束和安全规则；（3）需要对不同类型请求使用不同的模型或参数配置；（4）需要精确控制某些敏感请求的处理方式。如果用户请求类型高度一致，条件分支反而增加了不必要的复杂度。
