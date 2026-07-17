# 12.6.4 Transparency Report — 透明度报告

## 简单介绍

Transparency Report（透明度报告）是 AI Agent 系统向其利益相关方（用户、监管者、开发者、受影响的第三方）披露自身行为、决策逻辑、能力边界、安全措施及局限性的结构化文档与机制。它不仅是静态的文档发布，更涵盖了运行时的可解释性、事件的事后披露，以及持续的合规报告。

对于 AI Agent 而言，透明度报告解决了三个核心问题：(1) Agent 能做什么、不能做什么；(2) Agent 为什么会做出某个决策；(3) 当 Agent 出错或造成损害时，如何追溯与归因。透明度报告是将黑箱式的 Agent 行为转化为可理解、可审计、可问责信息的主要手段。

---

## 基本原理 — Transparency Builds Trust and Enables Oversight

透明度报告的设计基于以下四项核心原理：

### 1. 知情同意 (Informed Consent)
用户与 Agent 交互前，有权了解 Agent 的能力边界、数据使用方式和潜在风险。没有透明度，用户的同意就不是"知情"的。这与医疗领域的知情同意书类似——患者需要了解手术的风险才能签字。

### 2. 可问责性 (Accountability)
透明度是问责的前提。当 Agent 造成损害时（例如错误地拒绝了一项贷款申请、泄露了敏感信息），只有留存了充分的决策日志和推理过程，才能进行事后调查和责任认定。没有透明度，问责无从谈起。

### 3. 信任建立 (Trust Calibration)
透明度帮助用户建立对 Agent 的**恰当信任**（appropriate trust）——既不盲目信任（over-trust），也不过度怀疑（under-trust）。当 Agent 清晰说明自己为何做出某项决策、对自己判断的信心水平如何，用户能够更准确地判断何时依赖 Agent。

### 4. 外部监督 (Oversight Enablement)
监管者、审计师、研究者和第三方评估机构需要透明度报告来验证 Agent 是否符合安全标准、法规要求和伦理准则。透明度将内部评估结果开放给外部审查，形成社会层面的监督机制。

---

## 背景 — Model Cards → System Cards → Agent Transparency Reports → Regulatory Requirement

### 模型卡 (Model Cards, 2019)
Google 的 Mitchell 等人于 2019 年提出了 **Model Cards for Model Reporting** 框架。模型卡是一页左右的标准化文档，记录模型的训练数据、评估结果、预期用途和已知局限性。其核心理念是：每个公开发布的 ML 模型都应附有一份"营养标签"式的透明度文档。

模型卡的标准字段包括：
- 模型详情（名称、版本、类型）
- 预期用途与超出范围的使用
- 训练数据描述
- 评估数据集与指标（含不同人群子组的结果）
- 伦理考量与已知局限性
- 使用建议与注意事项

### 系统卡 (System Cards, 2023-2024)
随着从单一模型向复杂 AI 系统的演进，业界意识到模型卡不足以描述一个完整 AI 系统的行为。系统卡扩展了模型卡的概念，涵盖了：

- 系统的整体架构与组件
- 安全机制与缓解措施
- 系统级评估结果（而非仅模型级）
- 部署配置与环境
- 监控与反馈机制

OpenAI 为 GPT-4 发布系统卡（GPT-4 System Card），Google DeepMind 也为 Gemini 系列发布系统卡。系统卡成为前沿模型部署的事实标准。

### Agent 透明度报告 (Agent Transparency Reports, 2024-2025)
AI Agent 的引入带来了新的透明度挑战：Agent 是自主行动的，其行为链可能很长且包含多个工具调用。单一的系统卡不足以捕捉 Agent 的动态行为。Agent 透明度报告因此出现，其新增维度包括：

- 工具与权限清单（Agent 可以调用哪些工具、访问哪些数据）
- 行为模式与策略（Agent 如何规划、如何分解任务）
- 失败模式目录（已知的 Agent 失败场景与频率）
- 人机交互设计（何时请求人类确认、如何回退）
- 数据流追溯（Agent 在何处收集、处理、存储了哪些数据）

### 法规要求 (EU AI Act, 2024生效-2026执行)
EU AI Act 将透明度从自愿行为提升为法律义务。核心条款包括：

- **Article 13** (高风险 AI 系统): 向部署者提供透明度信息，包括系统特性、能力与限制
- **Article 50** (通用 AI 的透明度义务): 标注 AI 生成内容、披露系统能力与局限性
- **Article 12** (记录保持): 高风险系统需自动记录运行事件(log)，确保可追溯性
- **Article 26** (基础模型提供者的义务): 提供技术文档、评估摘要，遵守版权法

EU AI Act 的处罚力度极高——违规可处以上一年度全球年营收的 7%（某些条款）或 3%（其他条款），使透明度合规成为企业的刚性需求。

---

## 核心矛盾 — Transparency vs Competitive Secrecy, Detail vs Readability

### 矛盾一：透明度 vs 商业秘密
最尖锐的矛盾。详细的透明度报告（尤其是涉及训练数据、模型架构、评估方法和安全缺陷的部分）可能泄露企业的核心知识产权。

- 企业立场："如果我们披露所有安全评估细节，竞争对手可以复制我们的方法，甚至利用我们公开的缺陷信息攻击我们的系统。"
- 监管立场："没有详细的透明度，外部审计师无法验证你们的安全声明是否可信。"
- **折中方案**: 分层披露（layered disclosure）——向公众提供摘要版，向经认证的审计师提供完整版；使用保密协议保护敏感细节。

### 矛盾二：细节 vs 可读性
- 全面的技术细节会使文档冗长难懂，对非专业用户几乎没有帮助。
- 过度简化的摘要可能隐藏关键风险信息，构成"透明洗白"（transparency washing）。
- **折中方案**: 多受众文档策略——为不同读者群体定制不同粒度的报告（详见"Comparison"章节）。

### 矛盾三：透明度的时效成本
- 编写和维护透明度报告需要大量人力。Agent 的快速迭代（月级甚至周级更新）意味着文档可能频繁过时。
- **折中方案**: 自动化报告生成、CI/CD 集成、结构化数据驱动文档。

### 矛盾四：透明度的虚假安全感
- 存在透明度文档并不等于系统是安全的。一份写得漂亮的报告可能使读者高估系统的可靠性。
- 解释可能不忠实于实际决策过程（详见"Capability Boundaries"章节）。

---

## 详细内容

### 1. Agent System Card: Capabilities, Limitations, Safety Measures, Evaluation Results

Agent System Card 是整个透明度报告的纲领性文档，回答"这个 Agent 是什么"的根本问题。它继承模型卡和系统卡的框架，并针对 Agent 特性进行扩展。

#### 标准字段

| 字段 | 说明 | 示例 |
|------|------|------|
| Agent 名称与版本 | 唯一标识 Agent 版本 | `CustomerSupportAgent-v2.3.1` |
| 基础模型 | Agent 依赖的 LLM 列表 | GPT-4o, Claude 3.5 Sonnet |
| 工具清单 | Agent 可调用的所有工具 | 搜索API、数据库查询、邮件发送 |
| 权限范围 | 工具的操作权限等级 | 只读、读写、无需人工确认的执行 |
| 预期用途 | 设计的使用场景 | 电商客服：退款、订单查询、退货处理 |
| 非预期用途 | 禁止或不适用的场景 | 处理医疗建议、法律咨询 |
| 安全措施 | 防护机制列表 | 输入过滤、输出审核、人工审批闸门 |
| 评估结果 | 关键指标评测结果 | 任务成功率 94.2%，幻觉率 1.8% |
| 已知局限 | 已识别的失败模式 | 对时间敏感查询可能使用过时数据 |
| 人机交互设计 | 何时需要人类介入 | 退款金额 > $500 需人工审批 |

#### Python 数据结构示例

```python
@dataclass
class AgentSystemCard:
    agent_name: str
    version: str
    base_models: list[str]
    tools: list[ToolSpecification]
    permissions: PermissionScope
    intended_use: str
    out_of_scope_uses: list[str]
    safety_measures: list[SafetyMeasure]
    evaluation_results: EvaluationReport
    known_limitations: list[Limitation]
    human_oversight_design: HumanOversightSpec
    last_updated: datetime
```

---

### 2. Decision Transparency: Explaining Agent Decisions, Rationale for Tool Calls, Confidence Levels

决策透明度关注的是 **运行时** 的可解释性——当 Agent 正在执行任务时，用户和开发者能否理解 Agent 为什么选择某个行动路径。

#### 工具调用解释 (Tool Call Rationale)
Agent 的每次工具调用都应附带一个可读的解释，说明：
- 为什么需要调用此工具（目标是什么）
- 期望从该工具获得什么信息
- 此次调用如何服务于整体任务

```
[Agent]: 我需要查询用户的订单状态。
→ 调用 get_order_status(order_id="ORD-2024-8891")
  原因: 用户询问退款进度，我需要当前订单状态来判断退款是否已处理。
  预期结果: 返回订单的当前状态、退款金额和处理时间线。
```

#### 置信度表达 (Confidence Communication)
Agent 应当表达对自己决策和输出的置信水平，帮助用户判断可靠程度：

```python
@dataclass
class AgentDecision:
    action: str
    reasoning: str  # 自然语言推理过程
    confidence: float  # 0.0 ~ 1.0
    confidence_calibration: str  # "high" | "medium" | "low"
    alternative_actions: list[str]  # 考虑过的其他选项
    uncertainty_factors: list[str]  # 导致不确定的因素
```

#### 推理过程摘要 (Chain-of-Thought Summary)
不应暴露原始推理（可能包含敏感信息或过长），而应提供一个经审核的推理摘要：

```json
{
  "decision_id": "dec_20240717_abc123",
  "task": "处理退款申请 ORD-2024-8891",
  "steps": [
    {"step": 1, "action": "验证用户身份", "status": "成功", "confidence": 0.99},
    {"step": 2, "action": "查询订单状态", "status": "成功", "confidence": 0.98, "result": "已发货"},
    {"step": 3, "action": "判断退款资格", "status": "不能自动退款", "confidence": 0.95,
     "reason": "订单已发货超过30天，超出自动退款政策范围"},
    {"step": 4, "action": "转接人工客服", "status": "已转接", "confidence": 0.99}
  ],
  "final_decision": "无法自动处理，已转接人工客服",
  "overall_confidence": 0.95
}
```

---

### 3. Limitation Disclosure: Known Failure Modes, Accuracy Bounds, Uncertainty Communication

透明度报告必须诚实地披露 Agent 的已知局限，而不是仅展示最佳表现。

#### 已知失败模式 (Known Failure Modes)

分类记录 Agent 的典型失败类型及频率：

| 失败模式 | 触发条件 | 频率 | 严重程度 | 缓解措施 |
|---------|---------|------|---------|---------|
| 幻觉 | 查询超出训练数据范围的信息 | 2.1% | 高 | 输出审核过滤 |
| 工具误选 | 任务需要工具 A，Agent 选择了工具 B | 0.8% | 中 | 人类确认关键调用 |
| 循环推理 | Agent 陷入重复调用同一工具的循环 | 0.3% | 高 | 最大步数限制 + 熔断机制 |
| 上下文溢出 | 长对话中 Agent 丢失早期上下文 | 1.5% | 中 | 上下文摘要 + 滑动窗口 |
| 越狱攻击 | 恶意提示注入绕过安全限制 | <0.1% | 极高 | 输入清洗 + 对抗训练 |

#### 准确性边界 (Accuracy Bounds)

不是在所有场景下 Agent 的表现都一致。应当按维度披露准确性：

```python
@dataclass
class AccuracyBound:
    dimension: str  # 例如 "language", "domain", "task_type"
    segment: str    # 例如 "中文", "医疗领域", "多步推理"
    accuracy: float
    confidence_interval: tuple[float, float]  # 95% CI
    sample_size: int
    notes: str
```

#### 不确定性沟通 (Uncertainty Communication)

Agent 应当能够识别并向用户表达不确定性：
- **认知不确定性** (Epistemic): "我不确定这个问题的答案，因为它涉及 2025 年之后的未公开事件。"
- **偶然不确定性** (Aleatoric): "我无法 100% 确认汇率，因为汇率实时波动。当前参考汇率为 7.24，仅供参考。"
- **模糊性** (Ambiguity): "您说的'那个项目'可能指 Project Alpha 或 Project Beta，请明确指定。"

---

### 4. Data Handling Disclosure: What Data is Collected, Stored, Shared

Agent 的数据处理透明度对应于应用程序的隐私政策，但粒度更细——因为 Agent 在交互过程中动态收集和使用数据。

#### 数据收集清单

| 数据类型 | 是否收集 | 用途 | 保留期限 | 是否共享第三方 |
|---------|---------|------|---------|--------------|
| 对话内容 | 是 | 改进服务 + 安全审计 | 90 天 | 否（经匿名化处理后用于模型训练） |
| 用户 ID | 是 | 身份验证 | 账户存续期间 | 否 |
| IP 地址 | 是 | 安全风控 | 30 天 | 否 |
| 工具调用日志 | 是 | 审计追踪 | 180 天 | 在合规要求下提供给监管者 |
| 位置信息 | 否 | — | — | — |
| 生物识别信息 | 否 | — | — | — |

#### 数据流图 (Data Flow Diagram)

透明度报告应包含 Agent 的数据流图，说明：

```
用户输入
  │
  ▼
输入过滤器 ──→ [记录到审计日志: 原始输入(加密)]
  │
  ▼
LLM 处理 ──→ [记录到审计日志: 推理摘要]
  │
  ▼
工具调用决策
  │
  ├──→ 数据库查询 ──→ [记录到审计日志: 查询内容 + 结果摘要]
  ├──→ 外部 API ──→ [记录到审计日志: API 调用 + 返回摘要]
  └──→ 文件读写 ──→ [记录到审计日志: 文件路径 + 操作类型]
  │
  ▼
输出过滤器 ──→ [记录到审计日志: 输出内容(审核后)]
  │
  ▼
用户响应
```

#### 数据最小化承诺

明确声明 Agent 遵循数据最小化原则：只收集完成任务所必需的数据，不收集无关信息。

---

### 5. Incident Reporting: When Agent Causes Harm, How Incidents Are Reported and Investigated

事故报告机制是透明度报告的"召回"功能——当 Agent 造成损害时，如何记录、上报、调查和补救。

#### 事故分级 (Incident Severity Levels)

| 等级 | 定义 | 示例 | 响应时限 |
|------|------|------|---------|
| P0 - 严重 | 造成实质性损害或违反法规 | Agent 泄露了用户的信用卡号 | 1小时内 |
| P1 - 高 | 造成不便或轻微损害 | Agent 错误地拒绝了一笔合法交易 | 4小时内 |
| P2 - 中 | 功能异常但无直接损害 | Agent 给出错误但无害的信息 | 24小时内 |
| P3 - 低 | 轻微偏差 | Agent 表达不够清晰但答案正确 | 7天内 |

#### 事故报告流程

```python
class IncidentReport:
    incident_id: str
    timestamp: datetime
    severity: SeverityLevel
    description: str
    affected_users: int  # 匿名计数
    root_cause: str      # 调查后填写
    containment_actions: list[str]
    remediation: str
    preventive_measures: list[str]
    disclosure_status: str  # "internal" | "regulatory" | "public"
    report_author: str

    def publish(self, audience: Audience):
        """根据受众级别生成不同详略的报告"""
        ...
```

#### 透明度承诺

事故报告机制的核心承诺：
- **及时披露**: 严重事故在 [X 小时] 内向监管机构报告，[X 天] 内向受影响用户通报
- **根本原因分析 (RCA)**: 每次事故都进行 RCA，不仅修复表象
- **改进闭环**: 每次事故都生成可操作的改进措施，并在下一版本透明度报告中跟踪落实情况
- **免于报复**: 鼓励内部举报（whistleblowing）安全事件的文化

---

### 6. Automated Transparency: Runtime Transparency — Agent Explaining Its Current Reasoning and Action

自动透明度是指 Agent 在运行时**实时**地向用户展示其当前状态、推理过程和下一步计划的能力。这是用户体验层面的透明度。

#### 运行状态指示器 (Status Indicators)

| 状态 | 显示 | 含义 |
|------|------|------|
| 思考中 | "Analyzing your request..." | Agent 正在理解用户输入 |
| 推理中 | "Planning the best approach..." | Agent 正在制定任务计划 |
| 使用工具 | "Searching knowledge base..." | Agent 正在调用工具 |
| 审核中 | "Reviewing response for safety..." | 输出正在通过安全过滤器 |
| 等待确认 | "I need your approval to proceed..." | Agent 正在等待人类确认 |
| 完成 | "Here is the result..." | Agent 已完成任务 |

#### 实时推理展示 (Live Reasoning)

可选功能：Agent 可以实时展示其推理过程的一部分（经安全过滤后），使用户理解 Agent 正在做什么以及为什么。

```
[Agent 实时状态]
├─ 步骤 1/4: 理解需求 ✓
│   └─ "用户询问订单 ORD-2024-8891 的退款状态"
├─ 步骤 2/4: 查询数据 ⟳ [正在查询: get_order_status]
│   └─ 原因: "需要当前状态判断退款进度"
├─ 步骤 3/4: 分析结果 □
└─ 步骤 4/4: 生成响应 □
```

#### 透明度开关 (Transparency Controls)

用户应能控制透明度的层级：
- **简洁模式**: 只显示最终结果
- **标准模式**: 显示步骤摘要和置信度
- **详细模式**: 显示完整推理过程、工具调用细节和时间线

---

### 7. Regulatory Transparency: EU AI Act Requirements, Explainability Obligations, Audit Trail Access

#### EU AI Act 的核心透明度义务

| 条款 | 要求 | 对 Agent 的影响 |
|------|------|----------------|
| Art. 13 | 向部署者提供系统特性、能力和限制的说明 | Agent 必须附带详细的 System Card 和用户文档 |
| Art. 14 | 人类 oversight——自然人在必要时能干预 | Agent 必须支持人工接管和停止机制 |
| Art. 50 | 标注 AI 生成内容，披露与 AI 交互的事实 | 用户必须知道他们在与 Agent 而非人类对话 |
| Art. 12 | 自动 log 记录功能，确保可追溯性 | Agent 的运行日志必须自动记录并保存 |
| Art. 10 | 数据治理——训练数据的透明度和质量 | Agent 使用的数据和知识来源需记录 |

#### 可解释性义务 (Explainability Obligations)

监管可解释性不同于用户可解释性——监管者需要的是 **可审计的决策链** (auditable decision chain)，而不仅仅是表面解释：

```python
@dataclass
class RegulatoryExplainabilityRecord:
    """满足监管审计要求的完整性解释记录"""
    agent_action_id: str
    timestamp: datetime
    input_hash: str              # 输入的哈希（保护隐私同时可验证）
    model_invocation_log: list[  # 每次 LLM 调用的关键信息
        ModelCallRecord
    ]
    tool_call_log: list[         # 每次工具调用的完整记录
        ToolCallRecord
    ]
    human_intervention_log: list[  # 人类介入记录
        HumanInterventionRecord
    ]
    output_hash: str
    decision_tree: str           # 决策路径的机器可读表示
```

#### 审计追踪访问 (Audit Trail Access)

- **监管者访问**: 获得完整的、未经过滤的审计日志
- **部署企业访问**: 获得操作层面的审计日志（可能省略商业秘密细节）
- **独立审计师访问**: 在 NDA 下获得详细但有限制的访问
- **用户访问**: 获得与自身交互相关的日志摘要

---

## Example Code: Python TransparencyReporter

以下是一个生成 Agent 透明度报告的 Python 框架，涵盖 System Card 和决策解释的自动生成。

```python
"""
TransparencyReporter: AI Agent Transparency Report Generator
"""
from dataclasses import dataclass, field, asdict
from datetime import datetime
from typing import Optional
import json


# ─────────────────── Data Models ───────────────────

@dataclass
class ToolSpecification:
    name: str
    description: str
    permission_level: str  # "read" | "write" | "execute" | "admin"
    requires_human_approval: bool
    data_domains: list[str]  # 工具操作的数据域


@dataclass
class SafetyMeasure:
    name: str
    description: str
    effectiveness: float  # 0-1 有效性评分
    false_positive_rate: Optional[float] = None
    false_negative_rate: Optional[float] = None


@dataclass
class EvaluationResult:
    metric: str
    value: float
    confidence_interval: tuple[float, float]
    eval_date: datetime
    eval_dataset: str


@dataclass
class Limitation:
    category: str          # "hallucination" | "tool_selection" | "context_limit" | ...
    description: str
    frequency: float       # 出现概率百分比 (0-100)
    severity: str          # "low" | "medium" | "high" | "critical"
    mitigation: str
    last_observed: Optional[datetime] = None


@dataclass
class DataPractice:
    data_type: str
    collected: bool
    purpose: str
    retention_days: int
    shared_with_third_party: bool
    anonymized: bool = False


@dataclass
class AgentSystemCard:
    # 基础信息
    agent_name: str
    version: str
    release_date: datetime
    base_models: list[str]
    organization: str

    # 能力
    intended_use: str
    out_of_scope_uses: list[str]
    supported_languages: list[str]
    tools: list[ToolSpecification]

    # 安全
    safety_measures: list[SafetyMeasure]
    safety_eval_results: list[EvaluationResult]
    red_teaming_summary: str

    # 表现
    evaluation_results: list[EvaluationResult]
    known_limitations: list[Limitation]
    failure_mode_catalog: dict[str, float]  # 失败模式 -> 频率

    # 数据
    data_practices: list[DataPractice]
    training_data_summary: str

    # 人机交互
    human_oversight_design: str
    escalation_path: str


# ─────────────────── Reporter ───────────────────

class TransparencyReporter:
    """
    自动生成 Agent 透明度报告。

    使用方式:
        1. 在 Agent 开发过程中填充 AgentSystemCard
        2. 调用 report.generate() 生成结构化报告
        3. 调用 report.export() 输出为 JSON/Markdown
        4. 将报告纳入 CI/CD 管线，每次发布自动更新
    """

    def __init__(self, system_card: AgentSystemCard):
        self.system_card = system_card
        self.incident_log: list[IncidentRecord] = []
        self.decision_log: list[DecisionRecord] = []
        self.generated_at = datetime.now()

    def generate_system_card_report(self) -> dict:
        """生成完整的 System Card 报告"""
        base = asdict(self.system_card)
        base["report_generated_at"] = self.generated_at.isoformat()
        base["report_version"] = "2.0"
        return base

    def explain_decision(
        self,
        agent_action_id: str,
        task: str,
        steps: list[dict],
        final_decision: str,
        overall_confidence: float,
        confidence_breakdown: Optional[dict] = None
    ) -> dict:
        """
        生成单次决策的解释记录。

        Args:
            agent_action_id: 唯一动作标识
            task: Agent 被要求执行的任务
            steps: 决策步骤列表
                每个步骤包含: step, action, status, confidence, reason
            final_decision: 最终决策的描述
            overall_confidence: 整体置信度 (0-1)
            confidence_breakdown: 各维度的置信度分解

        Returns:
            结构化的决策解释字典
        """
        record = DecisionRecord(
            agent_action_id=agent_action_id,
            timestamp=datetime.now(),
            task=task,
            steps=steps,
            final_decision=final_decision,
            overall_confidence=overall_confidence,
            confidence_breakdown=confidence_breakdown or {}
        )

        self.decision_log.append(record)
        return asdict(record)

    def log_incident(
        self,
        incident_id: str,
        severity: str,
        description: str,
        affected_users: int,
        containment_actions: list[str]
    ) -> IncidentRecord:
        """
        记录一起安全事件。

        Returns:
            IncidentRecord (root_cause 等字段调查后填写)
        """
        record = IncidentRecord(
            incident_id=incident_id,
            timestamp=datetime.now(),
            severity=severity,
            description=description,
            affected_users=affected_users,
            containment_actions=containment_actions,
            root_cause="",        # 待调查填写
            remediation="",       # 待调查填写
            preventive_measures=[],  # 待调查填写
            disclosure_status="internal",
            report_author=self.system_card.organization
        )
        self.incident_log.append(record)
        return record

    def update_incident(
        self,
        incident_id: str,
        root_cause: str,
        remediation: str,
        preventive_measures: list[str],
        disclosure_status: str
    ):
        """事故调查完成后，更新根本原因和补救措施"""
        for record in self.incident_log:
            if record.incident_id == incident_id:
                record.root_cause = root_cause
                record.remediation = remediation
                record.preventive_measures = preventive_measures
                record.disclosure_status = disclosure_status
                return record
        raise ValueError(f"Incident {incident_id} not found")

    def export_transparency_report(
        self,
        format: str = "json",  # "json" | "markdown"
        audience: str = "public"  # "public" | "regulator" | "developer"
    ) -> str:
        """
        导出完整的透明度报告。

        Args:
            format: 输出格式
            audience: 目标受众——控制信息披露的粒度

        Returns:
            格式化的报告字符串
        """
        base_report = {
            "system_card": self.generate_system_card_report(),
            "recent_incidents": [
                asdict(r) for r in self.incident_log[-10:]
            ],
            "recent_decisions": [
                asdict(r) for r in self.decision_log[-100:]
            ],
            "audience": audience,
            "exported_at": datetime.now().isoformat()
        }

        # 根据受众过滤敏感信息
        if audience == "public":
            base_report = self._redact_for_public(base_report)
        elif audience == "regulator":
            base_report = self._enrich_for_regulator(base_report)

        if format == "json":
            return json.dumps(base_report, indent=2, default=str)
        elif format == "markdown":
            return self._to_markdown(base_report)
        else:
            raise ValueError(f"Unsupported format: {format}")

    def _redact_for_public(self, report: dict) -> dict:
        """公众版本：移除商业秘密和安全细节"""
        # 移除具体的红队测试策略
        if "system_card" in report:
            sc = report["system_card"]
            sc["red_teaming_summary"] = "已完成内部红队测试（详细策略未公开）"
            # 模糊化工具权限细节
            for tool in sc.get("tools", []):
                if tool["permission_level"] == "admin":
                    tool["permission_level"] = "elevated"
        return report

    def _enrich_for_regulator(self, report: dict) -> dict:
        """监管版本：增加完整审计细节"""
        # 增加未经过滤的事故日志
        report["full_incident_log"] = [
            asdict(r) for r in self.incident_log
        ]
        report["compliance_declarations"] = {
            "eu_ai_act_article_13": "compliant",
            "eu_ai_act_article_50": "compliant",
            "eu_ai_act_article_12": "compliant"
        }
        return report

    def _to_markdown(self, report: dict) -> str:
        """将报告转换为 Markdown 格式"""
        lines = [
            "# AI Agent Transparency Report",
            f"**Agent**: {self.system_card.agent_name} v{self.system_card.version}",
            f"**Generated**: {self.generated_at.isoformat()}",
            f"**Audience**: {report.get('audience', 'public')}",
            "",
            "## System Card Summary",
            f"- **Intended Use**: {self.system_card.intended_use}",
            f"- **Base Models**: {', '.join(self.system_card.base_models)}",
            f"- **Tools**: {len(self.system_card.tools)} tools registered",
            f"- **Known Limitations**: {len(self.system_card.known_limitations)} documented",
            "",
            "### Evaluation Results",
        ]
        for result in self.system_card.evaluation_results:
            lines.append(
                f"- **{result.metric}**: {result.value:.2f} "
                f"(95% CI: {result.confidence_interval[0]:.2f} - "
                f"{result.confidence_interval[1]:.2f})"
            )

        lines.extend([
            "",
            "### Known Limitations",
        ])
        for lim in self.system_card.known_limitations:
            lines.append(
                f"- **{lim.category}** ({lim.severity}): {lim.description} "
                f"[频率: {lim.frequency:.1f}%]"
            )

        lines.extend([
            "",
            "## Recent Incidents",
        ])
        for record in self.incident_log[-5:]:
            lines.append(
                f"- `{record.incident_id}` | {record.severity} | "
                f"{record.description[:80]}..."
            )

        return "\n".join(lines)


@dataclass
class IncidentRecord:
    incident_id: str
    timestamp: datetime
    severity: str
    description: str
    affected_users: int
    containment_actions: list[str]
    root_cause: str
    remediation: str
    preventive_measures: list[str]
    disclosure_status: str
    report_author: str


@dataclass
class DecisionRecord:
    agent_action_id: str
    timestamp: datetime
    task: str
    steps: list[dict]
    final_decision: str
    overall_confidence: float
    confidence_breakdown: dict


# ─────────────────── 使用示例 ───────────────────

def create_sample_report():
    """创建示例透明度报告"""
    card = AgentSystemCard(
        agent_name="CustomerSupportAgent",
        version="2.3.1",
        release_date=datetime(2026, 6, 15),
        base_models=["Claude 4.0", "GPT-4o"],
        organization="AcmeCorp AI",
        intended_use=(
            "处理电商客服请求：订单查询、退款处理、退货管理、"
            "配送状态跟踪、常见问题解答"
        ),
        out_of_scope_uses=[
            "医疗建议", "法律咨询", "金融投资建议",
            "任何超出电商客服范围的专业领域"
        ],
        supported_languages=["zh-CN", "en-US", "ja-JP"],
        tools=[
            ToolSpecification("get_order", "查询订单详情", "read", False, ["orders"]),
            ToolSpecification("process_refund", "处理退款", "write", True, ["payments"]),
            ToolSpecification("send_email", "发送通知邮件", "execute", False, ["notifications"]),
        ],
        safety_measures=[
            SafetyMeasure("input_filter", "过滤提示注入攻击", 0.97, 0.02, 0.01),
            SafetyMeasure("output_guardrail", "审核输出内容", 0.95, 0.01, 0.03),
            SafetyMeasure("human_approval_gate", "高风险操作需人工审批", 1.0),
        ],
        safety_eval_results=[
            EvaluationResult("jailbreak_resistance", 0.94, (0.92, 0.96),
                             datetime(2026, 6, 1), "red_team_v3"),
        ],
        red_teaming_summary="...",
        evaluation_results=[
            EvaluationResult("task_success_rate", 0.942, (0.931, 0.953),
                             datetime(2026, 6, 10), "cust_svc_bench_v2"),
            EvaluationResult("hallucination_rate", 0.018, (0.012, 0.024),
                             datetime(2026, 6, 10), "cust_svc_bench_v2"),
        ],
        known_limitations=[
            Limitation("time_sensitivity", "对于实时库存信息可能延迟", 5.0,
                       "medium", "标注信息获取时间戳"),
            Limitation("complex_escalation", "涉及多部门协调的复杂问题", 2.0,
                       "medium", "自动转接人工坐席"),
        ],
        failure_mode_catalog={"hallucination": 0.018, "tool_mis_selection": 0.008},
        data_practices=[
            DataPractice("conversation_content", True,
                         "服务改进与安全审计", 90, False, True),
            DataPractice("user_id", True, "身份验证", 365, False),
        ],
        training_data_summary="基于 500 万条脱敏客服对话训练",
        human_oversight_design="退款 > $500 需人工审批；首次错误后转人工",
        escalation_path="level_1: Agent → level_2: Supervisor → level_3: Manager"
    )

    reporter = TransparencyReporter(card)

    # 记录一次决策
    reporter.explain_decision(
        agent_action_id="dec_20240717_abc123",
        task="处理退款申请 ORD-2024-8891",
        steps=[
            {"step": 1, "action": "验证用户身份", "status": "成功", "confidence": 0.99},
            {"step": 2, "action": "查询订单状态", "status": "成功", "confidence": 0.98},
            {"step": 3, "action": "判断退款资格", "status": "不能自动退款", "confidence": 0.95},
            {"step": 4, "action": "转接人工客服", "status": "已转接", "confidence": 0.99},
        ],
        final_decision="无法自动处理，已转接人工客服",
        overall_confidence=0.95,
        confidence_breakdown={"authentication": 0.99, "data_retrieval": 0.98,
                              "policy_judgment": 0.95, "routing": 0.99}
    )

    return reporter


# 生成示例报告
if __name__ == "__main__":
    reporter = create_sample_report()

    # 公众版（Markdown）
    public_report = reporter.export_transparency_report(
        format="markdown", audience="public"
    )
    print(public_report)

    # 监管版（JSON）
    regulator_report = reporter.export_transparency_report(
        format="json", audience="regulator"
    )
    print(regulator_report)
```

---

## Capability Boundaries: Transparency Can Be Gamed, Explanations May Not Be Faithful

### 1. 透明洗白 (Transparency Washing)
组织可能发布看起来详尽的透明度报告，但实际上隐藏了关键信息：
- 选择性地仅报告有利的评估结果
- 使用过于宽泛的语言描述安全问题
- 将局限埋藏在冗长文档的深处
- 缺乏具体的量化指标

**检测方法**: 独立审计、第三方验证、要求提供基础数据。

### 2. 解释的不忠实性 (Explanation Unfaithfulness)
AI Agent 的解释可能不忠实于其真实的决策过程。这对于基于 LLM 的 Agent 尤为重要——LLM 生成的"推理"可能是一种**事后合理化**（post-hoc rationalization），而非对实际决策路径的真实重建。

研究表明：
- LLM 可以为其实际未使用的策略生成令人信服的解释
- 思维链（Chain-of-Thought）推理过程可能被注入的偏见污染
- 当 Agent 使用多个模型和工具时，归因变得更加复杂

**缓解方法**:
- 使用可验证的决策追踪（如记录真实 token 级别的推理路径）
- 交叉验证：对比 Agent 的解释与实际的工具调用日志
- 采用因果可解释性方法（ mechanistic interpretability）

### 3. 透明度的对手利用 (Adversarial Exploitation)
详细披露安全措施和局限可能被攻击者利用：
- 公开的失败模式目录为攻击者提供了攻击目标
- 安全措施的细节可能帮助攻击者规避检测
- 已知的评估方法可能被逆向工程

**缓解方法**:
- 分层披露（公众版、审计师版、监管版）
- 延迟披露（在修复完成后才公开漏洞信息）
- 对评估方法进行模糊化处理

### 4. 透明度疲劳 (Transparency Fatigue)
当透明度报告变得过长、过于频繁或过于技术化时，读者可能停止阅读。结果：形式上存在透明度，但实际上无人理解。

**缓解方法**:
- 结构化摘要（executive summary + 分层详述）
- 可视化仪表盘替代纯文字报告
- 面向不同受众的差异化呈现

### 5. 时效性挑战 (Timeliness Challenge)
Agent 可能每周迭代更新。文档的更新速度往往跟不上 Agent 的变化速度，导致发布的报告与实际运行的系统之间存在差异。

**缓解方法**:
- 将透明度报告生成纳入 CI/CD 管线
- 使用结构化数据驱动的自动化报告生成
- 为 Agent 版本和报告版本建立强绑定

---

## Comparison: Transparency Approaches for Different Audiences

### 受众分层对比

| 维度 | 用户 (Users) | 监管者 (Regulators) | 开发者 (Developers) | 受影响的第三方 |
|------|-------------|-------------------|-------------------|--------------|
| **核心需求** | 知道 Agent 能做什么、数据怎么用 | 验证法规合规性 | 调试、改进、安全加固 | 了解自身权益是否受损 |
| **详略级别** | 高层的概要 | 完整的详细数据 | 技术细节 + 原始数据 | 与自身相关的具体信息 |
| **格式** | 自然语言 + 图标 | 结构化文档 + 审计日志 | API + 原始日志 + 代码 | 通知信 + 摘要 |
| **频率** | 首次使用时 + 重大更新 | 按法规要求（通常年度） | 每次发布 | 事件发生时 |
| **保密级别** | 公开 | 保密（经认证访问） | 内部 | 私密 |
| **关键内容** | 能力、限制、隐私 | 合规声明、审计日志、评估数据 | 失败模式、API 文档、性能数据 | 事件说明、补救措施 |
| **解释深度** | 决策结果 + 置信度 | 完整决策树 + 归因链 | 完整推理 + 工具调用 + 中间状态 | 与事件相关的特定解释 |

### 异构报告策略 (Multi-Report Strategy)

组织应维护一个层次化的透明度文档体系：

```
Layer 0: 产品内透明度 (In-Product Transparency)
  ├── Agent 身份标识（告知用户正在与 AI 交互）
  ├── 实时状态指示器
  └── 隐私政策链接
  受众: 所有用户

Layer 1: 公开透明度报告 (Public Transparency Report)
  ├── Agent System Card（简化版）
  ├── 已知局限摘要
  ├── 数据处理实践概述
  └── 历史事故摘要
  受众: 一般公众、用户、媒体

Layer 2: 开发者透明度报告 (Developer Transparency Report)
  ├── 完整 Agent System Card
  ├── 评估结果（完整数据）
  ├── 失败模式目录（含频率和触发条件）
  ├── API 文档与决策格式说明
  └── 性能基准
  受众: 集成开发者、企业客户

Layer 3: 监管与审计报告 (Regulatory & Audit Report)
  ├── 完整的合规声明
  ├── 未经过滤的审计日志（存取控制）
  ├── 红队测试完整报告
  ├── 安全措施详细评估
  └── 完整事故报告（含 RCA）
  受众: 监管机构、认证审计师（NDA 保护下）

Layer 4: 内部技术报告 (Internal Technical Report)
  ├── 完整架构文档
  ├── 模型权重和数据细节
  ├── 安全机制实现细节
  └── 内部实验和失败记录
  受众: 组织内部（信息安全控制下）
```

---

## Engineering Optimization: Automated Report Generation, Transparency Dashboards, Structured Transparency Data

### 1. CI/CD 集成自动化

将透明度报告生成集成到开发流程中，确保每次发布都自动附带更新后的报告：

```yaml
# .github/workflows/transparency-report.yml 示例
name: Generate Transparency Report

on:
  release:
    types: [published]

jobs:
  generate-report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Extract agent metadata
        run: python scripts/extract_agent_metadata.py
      - name: Run evaluations
        run: python scripts/run_safety_eval.py --output eval_results.json
      - name: Generate transparency report
        run: |
          python scripts/generate_report.py \
            --metadata agent_metadata.json \
            --eval eval_results.json \
            --output reports/transparency_report_v${{ github.ref_name }}.json
      - name: Publish report
        run: |
          python scripts/publish_report.py \
            --report reports/transparency_report_v${{ github.ref_name }}.json \
            --audience public --output docs/transparency/
```

### 2. 透明度仪表盘 (Transparency Dashboards)

可视化仪表盘使透明度数据可浏览、可交互，替代静态 PDF 报告：

```python
class TransparencyDashboard:
    """
    生成透明度仪表盘数据，供前端可视化框架（如 Grafana、Streamlit）渲染
    """

    def get_metrics_over_time(self) -> list[dict]:
        """返回关键指标时间序列"""
        return [
            {"date": "2026-01", "task_success_rate": 0.931, "hallucination_rate": 0.022},
            {"date": "2026-02", "task_success_rate": 0.938, "hallucination_rate": 0.020},
            {"date": "2026-03", "task_success_rate": 0.940, "hallucination_rate": 0.019},
            {"date": "2026-04", "task_success_rate": 0.942, "hallucination_rate": 0.018},
        ]

    def get_incident_summary(self) -> dict:
        """返回事故摘要"""
        return {
            "total_incidents": 23,
            "by_severity": {"P0": 0, "P1": 2, "P2": 8, "P3": 13},
            "mean_resolution_time_hours": 6.5,
            "open_incidents": 1,
        }

    def get_decision_confidence_distribution(self) -> list[dict]:
        """返回决策置信度分布"""
        return [
            {"range": "0.9-1.0", "count": 4521, "accuracy": 0.98},
            {"range": "0.7-0.9", "count": 834, "accuracy": 0.87},
            {"range": "0.5-0.7", "count": 124, "accuracy": 0.71},
            {"range": "<0.5", "count": 18, "accuracy": 0.55},
        ]

    def get_audit_trail_summary(self, date_from: datetime, date_to: datetime) -> dict:
        """返回指定时间范围内的审计日志摘要"""
        return {
            "period": f"{date_from.date()} - {date_to.date()}",
            "total_actions": 15234,
            "tool_calls": 8942,
            "human_interventions": 234,
            "flagged_actions": 12,
            "compliance_status": "compliant",
        }
```

### 3. 结构化透明度数据标准 (Structured Transparency Data)

使用机器可读的格式（JSON Schema、YAML、Protocol Buffers）定义透明度数据结构，实现标准化和互操作性：

```json
{
  "$schema": "https://example.org/agent-transparency-schema-v2.json",
  "agent": {
    "id": "acmecorp_cust_support_v2.3.1",
    "name": "CustomerSupportAgent",
    "version": "2.3.1"
  },
  "capabilities": {
    "task_types": ["order_query", "refund", "return", "tracking"],
    "max_tool_calls_per_task": 15,
    "supported_context_window": 128000,
    "languages": ["zh-CN", "en-US", "ja-JP"]
  },
  "safety": {
    "jailbreak_test_pass_rate": 0.94,
    "output_compliance_rate": 0.97,
    "human_oversight_gates": [
      {"trigger": "refund_amount > 500", "action": "require_approval"},
      {"trigger": "first_error_in_session", "action": "escalate_to_human"}
    ]
  },
  "data_practices": {
    "collected_types": ["conversation", "user_id", "ip_address"],
    "retention_policy": "90d_rolling",
    "third_party_sharing": false,
    "anonymization": "pii_redaction + k_anonymity"
  },
  "evaluations": [
    {
      "benchmark": "cust_svc_bench_v2",
      "date": "2026-06-10",
      "metrics": {
        "task_success_rate": {"value": 0.942, "ci_95": [0.931, 0.953]},
        "hallucination_rate": {"value": 0.018, "ci_95": [0.012, 0.024]}
      }
    }
  ],
  "limitations": [
    {
      "id": "LIM-001",
      "category": "time_sensitivity",
      "description": "实时库存信息可能延迟 1-5 分钟",
      "frequency": 0.05,
      "severity": "medium",
      "mitigation": "响应中标注信息获取时间戳"
    }
  ]
}
```

### 4. 与观测性平台的集成 (Observability Integration)

将透明度数据流式接入观测性平台（如 Datadog、Grafana、OpenTelemetry）：

```python
class ObservabilityBridge:
    """
    将透明度数据导出到标准观测性平台
    """

    def emit_decision_event(self, decision: DecisionRecord):
        """发送 OpenTelemetry span 记录决策"""
        span = {
            "name": "agent.decision",
            "attributes": {
                "agent_action_id": decision.agent_action_id,
                "task": decision.task,
                "confidence": decision.overall_confidence,
                "step_count": len(decision.steps),
                "final_decision": decision.final_decision,
            },
            "timestamp": decision.timestamp.isoformat()
        }
        # 发送到 OpenTelemetry collector
        # otlp_client.emit(span)

    def emit_metric(self, metric_name: str, value: float, tags: dict):
        """发送指标到观测性平台"""
        # datadog_client.gauge(metric_name, value, tags=tags)
        pass
```

### 5. 报告质量校验 (Report Quality Checks)

自动化检查透明度报告的完整性和质量：

```python
class ReportQualityChecker:
    """自动检查透明度报告是否满足质量标准"""

    def check_completeness(self, report: dict) -> list[str]:
        """检查报告是否覆盖所有必需字段"""
        missing = []
        required_fields = [
            "agent_name", "version", "intended_use",
            "evaluation_results", "known_limitations",
            "data_practices", "safety_measures"
        ]
        for field in required_fields:
            if field not in report.get("system_card", {}):
                missing.append(field)
        return missing

    def check_quantification(self, report: dict) -> list[str]:
        """检查评估结果是否包含量化指标和置信区间"""
        warnings = []
        for result in report.get("system_card", {}).get("evaluation_results", []):
            if "confidence_interval" not in result:
                warnings.append(
                    f"Missing confidence interval for metric: {result.get('metric')}"
                )
        return warnings

    def check_limitations(self, report: dict) -> list[str]:
        """检查局限披露是否包含必要的缓解措施"""
        warnings = []
        for lim in report.get("system_card", {}).get("known_limitations", []):
            if not lim.get("mitigation"):
                warnings.append(f"Missing mitigation for limitation: {lim.get('category')}")
        return warnings
```

---

## 总结

透明度报告是 AI Agent 安全审计体系中连接**开发者意图**与**公众信任**的关键桥梁。它从诞生于模型卡的静态文档，演进为涵盖 System Card、运行时解释、事故报告、监管合规的多层次、可自动化生成的报告体系。在实践中，透明度报告需要平衡商业秘密保护与公众知情权、技术细节与可读性、全面性与维护成本之间的张力。随着 EU AI Act 等行业法规的执行，透明度报告将从"最佳实践"转变为"法定义务"，成为 AI Agent 部署的必要前提条件。

### 最佳实践要点

1. **默认透明，按需保密**: 默认公开除商业秘密和个人隐私外的所有信息
2. **分层披露**: 同一内容面向不同受众以不同粒度呈现
3. **自动化集成**: 将报告生成纳入 CI/CD，确保文档与 Agent 版本同步
4. **量化优先**: 使用量化指标 + 置信区间，避免模糊表述
5. **诚实披露局限**: 比读者先发现并报告自己的不足
6. **可验证性**: 确保报告中的声明可以被独立验证
7. **持续更新**: 透明度不是一次性的发布，而是持续的过程
