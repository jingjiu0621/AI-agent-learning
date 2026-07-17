# 3.2.1 Prompt Template Engine — 提示模板引擎（Jinja2, Mustache）

## 简单介绍

Prompt Template Engine 是动态 Prompt 系统的基石。它提供了一种将静态模板与运行时变量结合生成最终 Prompt 的机制。常用的模板引擎包括 Jinja2（Python 生态，功能强大）、Mustache（逻辑较少，跨语言）、Handlebars（JavaScript 生态）和 Liquid（产品级渲染）。在 AI Agent 开发中，模板引擎负责将结构化数据（工具返回结果、对话历史、用户意图）渲染为 LLM 可理解的自然语言指令。

## 基本原理

模板引擎的核心操作是"变量替换 + 控制结构"。开发者编写包含占位符和控制逻辑的模板文本，引擎在运行时将变量值填入占位符，并根据条件/循环逻辑动态生成最终输出。

```python
# Jinja2 示例：Agent System Prompt 模板
from jinja2 import Environment, BaseLoader

TEMPLATE = """
You are a {{ agent_role }} assistant.

Available tools:
{% for tool in tools %}
- {{ tool.name }}: {{ tool.description }}
{% endfor %}

Current user context:
- Language: {{ user_lang }}
- Expertise level: {{ expertise_level }}

{% if conversation_history %}
Previous interactions:
{{ conversation_history }}
{% endif %}

Always respond in {{ output_lang }}.
"""

env = Environment(loader=BaseLoader())
template = env.from_string(TEMPLATE)
prompt = template.render(
    agent_role="customer support",
    tools=[{"name": "search_kb", "description": "Search knowledge base"}],
    user_lang="zh-CN",
    expertise_level="beginner",
    output_lang="Chinese",
    conversation_history=""
)
```

## 背景

在早期 LLM 应用中，Prompt 通常是手写的静态字符串。开发者直接在代码中编写完整的 Prompt 文本。这种做法在原型阶段工作良好，但一旦 Agent 需要支持多种角色、多个工具和不同用户场景，手写 Prompt 的重复性和维护成本急剧上升。每次修改 Prompt 都需要搜索整个代码库，且容易在复制粘贴时引入不一致。

## 之前针对这个问题的做法与结果

**做法一：字符串拼接** — 使用 f-string 或 `str.format()` 拼接 Prompt。优点是简单直接，缺点是逻辑与模板混杂，嵌套条件时代码急剧膨胀，难以测试和复用。

**做法二：函数封装** — 将 Prompt 生成封装为函数，参数化输入。改进了复用性，但每个新场景需要新函数，参数爆炸，且非技术人员无法参与 Prompt 优化。

**做法三：配置文件** — 将 Prompt 存储在 YAML/JSON 文件中，代码读取后替换变量。解决了硬编码问题，但缺失循环和条件能力，无法处理复杂场景。

## 核心矛盾

模板的灵活性与简单性之间的矛盾。功能强大的模板引擎（如 Jinja2）支持循环、条件、宏、过滤器，但学习和调试成本高。简单模板引擎（如 Mustache）遵循"无逻辑"哲学，易于理解和安全，但面对复杂渲染需求时力不从心。Agent 开发需要在两者之间找到平衡点——既要支持复杂的 Prompt 组装逻辑，又要保持模板的可读性和可调试性。

## 当前主流优化方向

1. **模板片段（Partial/Template Inheritance）** — 将 Prompt 拆分为可复用的片段，通过继承或包含机制组装。例如基础系统模板定义角色，工具模板修改工具列表，用户模板处理具体请求。

2. **安全沙箱渲染** — 在 LLM 应用场景中，模板变量可能包含用户输入，需要防范模板注入攻击。主流方案包括限制可访问的变量范围、拒绝危险的 Python 内置函数、对输出做转义处理。

3. **类型安全的模板编译** — 将模板编译为类型安全的函数，在编译期而非运行时捕获变量缺失、类型不匹配等问题。例如基于 JinjaX 或自定义类型系统。

4. **模板缓存与预编译** — 高频调用的 Prompt 模板应当预编译为 AST，避免重复解析。对需要频繁切换的模板片段做部分缓存。

5. **模板版本元数据** — 在模板中嵌入版本号、变更描述、依赖的上下文字段列表，使得每个生成的 Prompt 都可追踪其来源。

## 实现的最大挑战

**调试困难**是最大挑战。动态模板生成的 Prompt 难以在开发阶段完全预览——因为渲染结果依赖运行时上下文。一个变量未传递可能导致"undefined"出现在给 LLM 的指令中，但 LLM 仍会尝试响应该指令，导致难以定位的隐式错误。解决方案包括：（1）模板渲染结果的自动日志；（2）模板变量的 Schema 校验（在每个模板中声明它需要哪些变量）；（3）渲染前后的 diff 对比工具。

```python
# 变量校验示例
class PromptTemplate:
    def __init__(self, template_str, required_vars):
        self.template = env.from_string(template_str)
        self.required_vars = set(required_vars)

    def render(self, **kwargs):
        missing = self.required_vars - set(kwargs.keys())
        if missing:
            raise ValueError(f"Missing variables: {missing}")
        return self.template.render(**kwargs)
```

## 能力边界与结果边界

模板引擎擅长处理"内容结构与变量替换"，但不擅长处理"内容质量的动态调整"。模板可以控制说什么，但无法保证说得好。例如，模板可以插入工具调用结果，但无法保证 LLM 正确理解该结果。模板引擎是 Prompt 生成的语法层，语义层的质量仍需要其他技术来保证。

## 与其他技术的区别

与 Context Injection 的区别：模板引擎解决"格式和结构"问题，Context Injection 解决"哪些数据需要注入"的问题。模板引擎更像是 HTML 模板，Context Injection 更像是数据流管道。在实际系统中，它们是互补的——模板引擎决定了变量的位置，Context Injection 决定了变量的来源。

## 核心优势

- **关注点分离**：Prompt 内容与生成逻辑解耦，非技术人员可以参与 Prompt 编写
- **复用性**：模板片段可在不同 Agent 场景间共享，避免重复
- **可测试性**：模板渲染是纯函数，给定相同输入必然产生相同输出，易于单元测试
- **一致性**：所有生成的 Prompt 遵循相同的结构模板，减少人为不一致

## 工程优化方向

1. **模板 Schema 化**：为每个模板定义其所需的变量 schema，实现自动校验
2. **渲染观测**：记录每次模板渲染的输入变量和输出结果，支持回放调试
3. **A/B 测试支持**：对同一场景使用不同模板版本，对比 LLM 输出质量
4. **模板热加载**：开发环境下支持模板文件修改后自动重载，加速迭代

## 适合场景的判断标准

以下情况适合引入模板引擎：（1）同一 Agent 需要在多个角色/模式下输出不同风格的内容；（2）Prompt 中包含动态的数据（工具列表、搜索结果、用户信息）；（3）有多个开发者或非技术人员参与 Prompt 编写；（4）需要对不同场景生成结构相似但细节不同的 Prompt。如果 Agent 只有一个固定角色、处理单一任务，直接写静态 Prompt 可能更高效。
