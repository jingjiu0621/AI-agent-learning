# 3.2.6 Multi-lang Prompts — 多语言 / 多角色 Prompt 切换

## 简单介绍

Multi-lang Prompts（多语言/多角色 Prompt 切换）是指 Prompt 系统根据用户的语言偏好、角色身份或使用场景，动态切换 Prompt 的语言、风格和人格设定。这不仅是"翻译"问题——不同语言和文化背景的用户对同一功能的期望不同，不同角色的用户需要不同风格和深度的交互。在 AI Agent 的全球化部署中，多语言/多角色支持是基本要求而非增值功能。

## 基本原理

核心原理是"配置驱动的 Prompt 选择"：系统维护一个配置表，每个条目定义了在特定条件下（语言 / 角色 / 场景）应使用的 Prompt 模板和参数。当条件变化时，系统根据配置表找到匹配的模板，重新渲染即可。关键在于，切换不仅是语言的翻译，还包括风格、习惯用语、知识库引用源、示例数据等多维度的适配。

```python
class MultiLangPromptManager:
    def __init__(self):
        self.locales = {}  # "zh-CN" / "en-US" / "ja-JP" -> prompt config

    def register_locale(self, locale_code, config):
        """
        config = {
            "system_prompt": template,
            "greeting_template": template,
            "error_template": template,
            "tools": [tool_configs],
            "examples": [example_configs],
            "cultural_notes": str,
            "max_tokens": int,
        }
        """
        self.locales[locale_code] = config

    def build_prompt(self, user_locale, user_role, context):
        locale_config = self._resolve_locale(user_locale)
        role_config = self._resolve_role(user_role)

        system = locale_config["system_prompt"].render(
            role=role_config["role_name"],
            tone=role_config["tone"],
            **context
        )
        examples = self._select_examples(locale_config, role_config)
        tools = locale_config["tools"]

        return {
            "system": system,
            "examples": examples,
            "tools": tools,
            "language": locale_config["language_code"],
        }

    def _resolve_locale(self, locale):
        # "zh" -> "zh-CN", "en" -> "en-US", fallback to "en-US"
        if locale in self.locales:
            return self.locales[locale]
        base = locale.split("-")[0]
        for code in self.locales:
            if code.startswith(base):
                return self.locales[code]
        return self.locales["en-US"]
```

## 背景

早期 AI Agent 主要面向英语用户，Prompt 和示例数据都是英文的。随着 AI 应用全球化，越来越多非英语用户使用 Agent。起初开发者的应对方式很简单——"LLM 本身就懂多语言，用英语写 Prompt 它也能理解中文输入"。但实践证明这种方式存在多个问题：英语 Prompt 对非英语用户的响应质量明显低于英语用户（推理一致性下降）；文化特定的示例和格式不适用于其他文化圈；不同语言中的敏感词和合规要求不同。

## 之前针对这个问题的做法与结果

**做法一：单一英语 Prompt + 依赖 LLM 翻译** — 所有 Prompt 用英语编写，靠 LLM 理解用户的其他语言输入并用对应语言回答。优点是实现简单、维护成本低。缺点是：非英语场景的推理质量不稳定；英语文化中的假设（如时间格式 MM/DD、数字格式）在非英语环境中造成困惑；无法针对不同语言做特定优化。

**做法二：翻译全部 Prompt** — 将英语 Prompt 人工翻译为所有目标语言。优点是质量高、可控。缺点是维护成本巨大——每次修改 Prompt 都需要重新翻译所有语言版本；不同语言版本之间的差异难以追踪；翻译质量难以保证一致性。

**做法三：模板参数化 + 本地化** — 将 Prompt 中需要本地化的部分提取为变量（问候语、示例、格式说明），为每种语言提供一套本地化变量值。这种方法在"保留 Prompt 框架同时切换语言细节"之间取得了较好的平衡。

```python
# 模板参数化本地化示例
SYSTEM_PROMPT_TEMPLATE = """
You are a helpful assistant that responds in {{ lang_name }}.

Date format: {{ date_format }}
Number format: {{ number_format }}

{{ greeting }}

{{ cultural_context }}

Examples:
{% for example in examples %}
Q: {{ example.question }}
A: {{ example.answer }}
{% endfor %}
"""

LOCALE_DATA = {
    "zh-CN": {
        "lang_name": "简体中文",
        "date_format": "YYYY-MM-DD",
        "number_format": "1,234.56",
        "greeting": "请用友好而专业的方式回答用户的问题。",
        "cultural_context": "中国用户习惯使用微信，沟通方式偏正式但有礼貌。",
        "examples": [...],
    },
    "en-US": {
        "lang_name": "English",
        "date_format": "MM/DD/YYYY",
        "number_format": "1,234.56",
        "greeting": "Please respond in a friendly and professional manner.",
        "cultural_context": "US users value directness and efficiency.",
        "examples": [...],
    },
}
```

## 核心矛盾

**本地化深度与维护成本之间的矛盾**。浅层本地化（翻译 Prompt 文本）成本低但体验差——不同语言的用户得到的体验"翻译腔"很重。深层本地化（适配语言习惯、文化规范、法律法规）体验好但成本极高——每种语言都需要独立的领域适配和文化审核。特别对于 Agent 这类交互式系统，不同文化的用户对 Agent 的"人格期待"不同（有些文化偏好权威专业型，有些偏好平等朋友型），这种人格层面的深度适配超出了简单翻译的范畴。

## 当前主流优化方向

1. **语言+角色矩阵管理** — 将"语言"和"角色"作为两个独立维度，交叉组合为 Prompt 配置矩阵。每种语言定义了语言特定配置（日期格式、示例语言），每种角色定义了角色特定配置（语气正式度、专业深度）。系统根据用户的语言和角色从矩阵中选取对应配置。

2. **文化适配层** — 在 Prompt 渲染链中增加一块"文化适配"逻辑，自动检测和处理文化敏感点。例如某些语言中不能直接要求 LLM 批评或否定用户；某些文化偏好结构化的列表而非段落式描述。

3. **动态示例选择** — 不同语言/角色的用户需要不同类型的示例。系统根据用户画像从示例库中动态选取最相关的示例，而非静态绑定一组示例。

4. **按需自动翻译 + 人工审核** — 使用 LLM 自动翻译 Prompt 内容（减少人力成本），但关键内容经人工审核后才能上线（保证质量）。自动翻译的 Prompt 版本标记为"auto-translate"，需要人工确认后才能 promote 到正式版本。

5. **回退链** — 当某种语言的配置不完整时，系统自动回退到默认语言（通常是英语）的对应配置而非崩溃。回退行为记录日志，便于后续补全该语言的配置。

## 实现的最大挑战

**多语言下 Prompt 行为一致性**是最大挑战。在理想情况下，切换语言应只改变"表达方式"而不改变"功能行为"。但实际上，同一 Prompt 的不同语言版本可能产生不同的推理结果——尤其是在涉及文化特定概念、复杂推理和安全约束时。例如，英语版本能正确拒绝一个有害请求，但日语翻译版本可能因语言差异导致拒绝规则被误解或忽略。确保所有语言版本在关键行为上的一致性，需要全面的跨语言测试体系。

## 能力边界与结果边界

多语言 Prompt 系统能确保用户以自己熟悉的语言与 Agent 交互，但无法保证"所有语言获得完全同等的体验"。某些语言（如英语）的训练数据覆盖更广，LLM 对该语言的理解和生成能力更强。对于小语种，即使 Prompt 翻译正确，LLM 的生成质量也受限于训练数据。这是模型层面的限制，Prompt 工程能做的是"在模型能力范围内最大化各语言体验"。

## 与其他技术的区别

与 Condition Prompts 的关系：Multi-lang 是一种特殊的条件分支——条件变量是"语言"和"角色"。与 Version Control 的关系：每个语言版本的 Prompt 可能有独立的版本轨迹（不同语言的不同团队维护）。与 Template Engine 的关系：模板引擎是实现多语言切换的基础设施——变量替换机制天然支持语言参数化。

## 核心优势

- **全球覆盖**：单一 Agent 系统可以服务多种语言和文化的用户群体
- **体验优化**：用户使用母语交互，理解更准确、感受更自然
- **合规性**：不同地区的法律法规（如 GDPR 中文版要求、中国生成式 AI 管理办法）可在本地化层处理

## 工程优化方向

1. **语言覆盖分析**：追踪各语言的用户活跃度和满意度，确定本地化优先级
2. **跨语言一致性测试**：对同一功能的所有语言版本进行相同的测试用例验证，标记行为不一致的语言
3. **语言热部署**：修改某个语言的 Prompt 不影响其他语言，且可独立发布
4. **翻译记忆库**：建立 Prompt 术语和常用表达的翻译记忆库，确保跨语言的术语一致性
5. **角色模板市场**：建立可复用的角色模板库（"技术支持"、"导师"、"顾问"），每种语言下可选用相同的角色模板

## 适合场景的判断标准

以下情况需要多语言/多角色 Prompt 切换：（1）Agent 面向全球多语言用户；（2）Agent 需要根据不同用户类型（如"开发者"vs"管理者"）提供不同风格的交互；（3）不同地区的法律合规要求不同；（4）同一产品需要同时面向 C 端普通用户和 B 端专业用户。如果 Agent 只服务单一语言群和单一用户角色，则不需要此功能。
