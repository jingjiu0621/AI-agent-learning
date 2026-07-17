# 12.6.2 compliance-frameworks — 合规框架

## 简单介绍

合规框架（Compliance Frameworks）是监管机构或行业组织制定的**强制性或推荐性规则集合**，规定了组织在数据处理、隐私保护、安全管理、AI 系统部署等方面的最低要求。对于 AI Agent 系统，合规不再是纯法务问题——Agent 自主决策的特性使得合规要求必须嵌入到系统架构、日志设计、数据流控制和模型行为约束中。从 GDPR 的"解释权"到 EU AI Act 的"高风险系统分类"，合规框架直接决定了 Agent 能做什么、必须记录什么、以及如何向用户和监管机构负责。

## 基本原理 — regulatory compliance is a minimum requirement for enterprise Agent deployment

**合规是 Agent 在企业环境落地的准入门槛，而非技术竞赛的终点。** 理解这个定位至关重要：

```
合规在 Agent 部署中的位置
───────────────────────────────

  理想态 (Differentiator)        伦理对齐、可解释 AI、社会责任
                                        ↑
  竞争优势 (Competitive Edge)    透明度报告、合规自动化、隐私增强技术
                                        ↑
  行业标准 (Industry Norm)       SOC 2 Type II、ISO 42001 认证
                                        ↑
  🟡 合规基线 (Compliance Floor)  GDPR/HIPAA/EU AI Act 硬性要求 ← 你是这里
                                        ↑
  非法 (Non-Compliant)           监管处罚、禁令、刑事责任
```

### 合规作为架构约束

对于 Agent 系统，合规不是"部署完成后加个 checkbox"，而是必须从架构层面设计的**系统性约束**：

```
传统软件合规                              Agent 系统合规
─────────────────                        ─────────────────────
• 数据存储方式合规                        • 数据采集 + 处理 + 推理全链路合规
• 访问控制列表合规                        • Agent 自主决策的权限边界合规
• 日志记录合规                            • Agent 推理过程的审计溯源合规
• 一次性合规审计                          • 持续合规监控（Agent 行为动态变化）
• 人工流程控制                            • 自动化合规检查（人工无法跟上 Agent 速度）
```

### 为什么合规对 Agent 更重要

| 维度 | 传统系统 | Agent 系统 | 合规压力 |
|------|---------|-----------|---------|
| 决策主体 | 人类用户 | Agent 自主决策 | Agent 的决策需要可解释、可审计 |
| 操作速度 | 人工节奏 | 毫秒级自动操作 | 合规检查必须自动化 |
| 数据接触面 | 有限的 API | 可访问多系统、多工具 | 数据最小化原则更难执行 |
| 行为变异性 | 确定的代码路径 | 非确定性 LLM 推理 | 合规不能依赖"同态审核" |
| 责任归属 | 人类操作者 | 模糊的开发者/部署者/用户分担 | 法律合规责任不清晰 |

## 背景 — GDPR (2018) → SOC 2 → HIPAA → EU AI Act (2025-2026) → global AI regulation

### 时间线：从数据保护到 AI 治理

```
合规框架演进时间线
═══════════════════════════════════════════════════════════════════════

1970s-1990s           2000s               2010s                2020s
│                     │                   │                    │
▼                     ▼                   ▼                    ▼
──┬──────────────────┬──────────────────┬────────────────────┬─────────►
  │                  │                  │                    │
  │ HIPAA (1996)     │                  │ GDPR (2018)        │ EU AI Act (2025)
  │ 美国医疗数据保护   │                  │ 欧盟数据保护       │ 全球首个 AI 立法
  │                  │                  │                    │
  │                  │ SOC 2 (2010)     │ CCPA (2020)        │ ISO 42001 (2023)
  │                  │ 服务组织控制      │ 加州消费者隐私     │ AI 管理体系
  │                  │                  │                    │
  │                  │                  │                    │ NIST AI RMF (2023)
  │                  │                  │ 中国个人信息保护法   │ AI 风险管理
  │                  │                  │  (PIPL) 2021       │
  │                  │                  │                    │ Brazil AI Bill
  │                  │                  │                    │ Canada AIDA
  │                  │                  │                    │ Japan AI Guidelines
```

### GDPR (2018) — 数据保护的里程碑

- **适用范围**：任何处理欧盟居民个人数据的组织，无论其所在地
- **关键原则**：数据最小化、目的限制、存储限制、完整性保密性
- **对 Agent 的直接影响**：Agent 必须支持"被遗忘权"——如果用户要求删除数据，Agent 必须能清除所有关联的推理记忆和训练数据痕迹
- **处罚力度**：最高全球年营收的 4% 或 2000 万欧元（取较高者）

### SOC 2 (2010, 持续更新) — 服务组织的安全标准

- **适用范围**：SaaS 提供商、云服务、数据处理服务商
- **五大信任服务准则**：安全性、可用性、处理完整性、机密性、隐私
- **对 Agent 的直接影响**：部署 Agent 的企业通常需要 SOC 2 Type II 报告来证明其 Agent 系统的安全控制有效
- **审计周期**：Type I（设计合理）+ Type II（6-12 个月运行有效）

### HIPAA (1996, 持续更新) — 美国医疗数据保护

- **适用范围**：医疗保健提供者、健康计划、医疗信息交换所及其业务伙伴
- **关键规则**：隐私规则（Privacy Rule）、安全规则（Security Rule）、违规通知规则（Breach Notification Rule）
- **对 Agent 的直接影响**：处理 PHI（受保护健康信息）的医疗 Agent 必须实施 BAA（业务伙伴协议）、加密传输与存储、严格的访问审计
- **处罚力度**：每年最高 170 万美元

### EU AI Act (2025-2026) — 全球首个 AI 综合立法

- **生效时间表**：2025 年 2 月（禁止条款）、2025 年 8 月（透明度义务）、2026 年 8 月（完整适用）
- **风险分类**：不可接受风险（禁止）→ 高风险（严格监管）→ 有限风险（透明度）→ 最小风险（无约束）
- **对 Agent 的直接影响**：自主决策型 Agent 很可能被归类为高风险系统，需要建立风险管理系统、技术文档、记录保持、透明度和人工监督机制
- **处罚力度**：最高全球年营收的 7% 或 3500 万欧元

### 全球 AI 监管格局 (2025-2026)

| 国家/地区 | 主要法规 | 状态 | Agent 影响度 |
|-----------|---------|------|-------------|
| 欧盟 | EU AI Act | 已通过，分阶段生效 | ★★★★★ |
| 美国 | NIST AI RMF, 州级法案 | 框架发布，联邦立法推进中 | ★★★★☆ |
| 中国 | 生成式 AI 管理办法、PIPL | 已生效 | ★★★★★ |
| 加拿大 | AIDA (Artificial Intelligence and Data Act) | 立法中 | ★★★☆☆ |
| 英国 | 基于原则的 AI 监管 | 白皮书阶段 | ★★★☆☆ |
| 巴西 | AI Bill PL 2338/2023 | 立法中 | ★★★☆☆ |
| 日本 | AI Guidelines | 指南已发布 | ★★☆☆☆ |
| 新加坡 | AI Verify | 测试框架已发布 | ★★☆☆☆ |

## 核心矛盾 — compliance frameworks were not designed for Agent systems

```
合规框架的假设                                        Agent 系统的现实
─────────────────────────                            ─────────────────────
┌─────────────────────────────────┐                  ┌─────────────────────────────┐
│  "系统行为是确定的"              │                  │  "Agent 行为是非确定的"      │
│  相同的输入 → 相同的输出          │    ←→           │  相同输入 → 可能不同输出     │
│  合规控制可以基于确定性预期       │                  │  合规需要基于概率范围       │
└─────────────────────────────────┘                  └─────────────────────────────┘

┌─────────────────────────────────┐                  ┌─────────────────────────────┐
│  "人类在决策链路中"             │                  │  "Agent 自主决策"           │
│  人类操作者作为责任主体          │    ←→           │  Agent 发起操作链           │
│  合规控制关注人类权限            │                  │  谁为 Agent 的决策负责？     │
└─────────────────────────────────┘                  └─────────────────────────────┘

┌─────────────────────────────────┐                  ┌─────────────────────────────┐
│  "静态数据处理"                 │                  │  "动态数据生命周期"         │
│  数据在已知系统中存储和处理      │    ←→           │  Agent 可能将数据传输        │
│  数据流可以被预先映射            │                  │  到未知的外部上下文          │
└─────────────────────────────────┘                  └─────────────────────────────┘

┌─────────────────────────────────┐                  ┌─────────────────────────────┐
│  "一次性合规审计"               │                  │  "持续合规监控"             │
│  年度审计 + 阶段性检查           │    ←→           │  Agent 的每次推理都不同     │
│  合规状态相对稳定               │                  │  合规状态动态变化           │
└─────────────────────────────────┘                  └─────────────────────────────┘
```

### 五大核心矛盾

| # | 矛盾 | 说明 |
|---|------|------|
| 1 | **确定性要求 vs 非确定性系统** | 合规框架假定行为可预测、可测试、可验证。Agent 的 LLM 推理本质是概率性的——同样的 prompt 在不同时间可能产生不同的决策。合规审计需要从"检查这个特定输出是否正确"转向"检查这个输出范围是否在合规边界内" |
| 2 | **人类责任模型 vs Agent 自主性** | GDPR 要求"数据控制者"负责，EU AI Act 要求"部署者"负责。但当 Agent 自主做出数据共享决策、工具调用决策时，责任归属变得模糊——是开发者的责任？部署者的责任？还是用户（通过提示引导）的责任？ |
| 3 | **静态数据地图 vs 动态数据流** | 传统合规要求组织维护数据地图（Data Mapping），记录数据在系统间的流向。但 Agent 自主调用外部工具、访问外部 API，数据流可能在运行时才确定。合规审计无法预先覆盖所有可能的数据路径 |
| 4 | **年度审计节奏 vs 毫秒级操作** | Agent 在几秒内可能完成数百次工具调用，在用户未见的情况下做出大量数据处理决策。传统的年度/季度审计周期在这个速度面前毫无意义——必须建立实时合规监控 |
| 5 | **边界清晰的系统 vs 开放的工具生态** | 传统合规控制假设系统边界是清晰的（如防火墙内的数据库）。Agent 可以调用任意外部 API、执行代码、访问网页——系统边界是动态扩展的。合规控制必须覆盖这些动态边界 |

## 详细内容

### 1. GDPR for Agents — 数据保护的核心约束

GDPR（General Data Protection Regulation）是 Agent 系统在全球范围内面临的最严格的隐私合规要求。它直接影响 Agent 如何采集、处理、存储和删除个人数据。

#### 核心原则的 Agent 映射

```
GDPR 原则 ──────────►  Agent 系统要求
────────────────────────────────────────────────

合法、公正、透明 ──►  Agent 必须告知用户"我是 Agent，正在处理你的数据"
目的限制       ──►  Agent 不能将收集的数据用于超出原始目的的其他任务
数据最小化     ──►  Agent 只收集完成特定任务所需的最少数据
准确性        ──►  Agent 必须有机制验证处理的数据准确，支持更正
存储限制      ──►  Agent 的对话历史、记忆数据必须有自动过期策略
完整保密性    ──►  Agent 工具调用链路必须加密，memory 必须加密存储
问责制        ──►  Agent 的每个数据操作必须可追溯、可审计
```

#### GDPR 对 Agent 的五项关键要求

**① 数据最小化（Data Minimization）**

```python
# GDPR 合规的 Agent 数据采集模式
# ❌ 不合规: Agent 默认捕获所有输入/输出数据
class NonCompliantAgent:
    def process(self, user_input: str) -> str:
        self.log_full_conversation(user_input)     # 默认记录所有对话
        self.extract_all_user_data(user_input)      # 提取所有可能的个人信息
        return self.llm.invoke(user_input)

# ✅ 合规: Agent 明确声明、最小化采集
class GDPRCompliantAgent:
    def process(self, user_input: str, purpose: str) -> str:
        # 明确数据使用目的
        self.assert_purpose_defined(purpose)
        # 最小化数据采集——只提取完成 purpose 所需的字段
        minimal_data = self.extract_minimal_data(user_input, purpose)
        # 自动过期（默认 30 天）
        ttl = timedelta(days=30)
        self.store_with_ttl(minimal_data, ttl)
        # 日志脱敏
        self.log_operation(minimal_data, purpose, mask_pii=True)
        return self.llm.invoke(minimal_data)
```

**② 解释权（Right to Explanation）**

GDPR Recital 71 指出数据主体有权获得基于自动化决策的解释。对于 Agent 系统：

- Agent 必须能够解释**为什么**做出了某个决策（基于哪些数据、哪些推理步骤）
- 解释必须是"有意义的"，而非标准化的免责声明
- 对于基于 LLM 的 Agent，这意味着需要记录推理链（Chain-of-Thought）、工具调用序列、关键输入特征

```python
# Agent 决策解释框架
class ExplainableAgent:
    def decide(self, user_input: str) -> tuple[str, dict]:
        decision_trace = {
            "timestamp": datetime.utcnow().isoformat(),
            "input_summary": self.mask_pii(user_input),
            "reasoning_steps": [],
            "data_sources_used": [],
            "confidence_score": None
        }

        # 记录推理链
        cot_result = self.llm.chain_of_thought(user_input)
        decision_trace["reasoning_steps"] = cot_result.steps

        # 记录数据源引用
        for tool_call in self.tool_calls:
            decision_trace["data_sources_used"].append({
                "tool": tool_call.name,
                "endpoint": tool_call.endpoint,
                "data_categories": tool_call.data_categories_accessed
            })

        final_decision = cot_result.conclusion
        return final_decision, decision_trace

    def explain_decision(self, user_id: str, session_id: str) -> dict:
        # 向用户提供人类可读的解释
        return self.audit_store.get_decision_trace(user_id, session_id)
```

**③ 删除权（Right to Erasure / Right to be Forgotten）**

Agent 系统的删除面临独特挑战：

```
传统系统删除                           Agent 系统删除
────────────────                      ────────────────────
• 删除数据库中的行                      • 删除数据库行+
• 删除文件系统中的文件                   • 删除向量数据库中的 embedding
                                       • 清除 Agent 长期记忆中的关联
                                       • 删除推理缓存中的相关内容
                                       • 从模型微调数据中移除（几乎不可能）
                                       • 清除所有衍生数据（Agent 生成的摘要等）
```

```python
class GDPRCompliantAgentStore:
    def erase_user_data(self, user_id: str) -> ErasureReport:
        report = ErasureReport(user_id)

        # 1. 删除结构化数据
        deleted_rows = self.sql_db.delete_user_data(user_id)
        report.add("structured_data", deleted_rows)

        # 2. 删除向量 embedding
        deleted_vectors = self.vector_db.delete_user_vectors(user_id)
        report.add("vector_embeddings", deleted_vectors)

        # 3. 清除长期记忆
        cleared_memories = self.memory_store.clear_user_memories(user_id)
        report.add("agent_memories", cleared_memories)

        # 4. 清除缓存中的用户数据
        cleared_cache = self.cache_store.invalidate_user_cache(user_id)
        report.add("cached_data", cleared_cache)

        # 5. 标记衍生数据（Agent 生成的摘要/报告）
        derived_count = self.derived_data_store.mark_for_deletion(user_id)
        report.add("derived_data", derived_count)

        # 6. 记录删除审计日志（保留删除操作记录本身）
        self.audit_log.record_deletion(user_id, report)
        return report
```

**④ 同意管理（Consent Management）**

Agent 系统在运行中可能频繁涉及数据处理，同意管理必须嵌入 Agent 工作流：

```python
class ConsentManagedAgent:
    def __init__(self):
        self.consent_store = ConsentStore()

    def process_with_consent(self, user_id: str, input_data: dict, purpose: str) -> Response:
        # 1. 检查是否已有有效同意
        consent = self.consent_store.get_consent(user_id, purpose)

        if not consent or consent.is_expired():
            # 2. 无有效同意 → 请求同意（阻塞式 或 降级服务）
            return Response.consent_required(purpose=purpose,
                                             data_categories=list(input_data.keys()))

        if not consent.covers_data(input_data.keys()):
            # 3. 当前操作超出了同意范围
            return Response.consent_insufficient(
                missing_categories=[k for k in input_data if k not in consent.data_categories]
            )

        # 4. 同意有效 → 正常处理
        return self._process_with_audit(user_id, input_data, purpose, consent.id)

    def _process_with_audit(self, user_id, input_data, purpose, consent_id):
        # Agent 实际处理逻辑
        result = self.agent.execute(input_data)
        # 记录审计：本次处理是基于哪个 consent
        self.audit_log.record(user_id, purpose, consent_id,
                              data_categories=list(input_data.keys()))
        return result
```

**⑤ 数据传输限制（Data Transfer Restrictions）**

Agent 调用外部工具时可能将数据传输到第三国。GDPR Chapter V 要求：

- Agent 工具调用链路必须识别数据流向的国家/地区
- 涉及第三国（非 EU 足够保护水平国家）数据传输必须有标准合同条款（SCC）或充分性决定
- Agent 不能默认将数据路由到不合规的第三方工具

```python
class GDPRDataTransferGuard:
    EU_ADEQUATE_COUNTRIES = {"JP", "KR", "UK", "CA", "IL", "NZ"}  # 示例

    def check_tool_call(self, tool: Tool, data_categories: set[str]) -> TransferDecision:
        if not self._contains_personal_data(data_categories):
            return TransferDecision.ALLOWED  # 不包含个人数据，允许

        tool_jurisdiction = tool.get_jurisdiction()
        if tool_jurisdiction == "EU":
            return TransferDecision.ALLOWED  # 欧盟境内，允许

        if tool_jurisdiction in self.EU_ADEQUATE_COUNTRIES:
            return TransferDecision.ALLOWED  # 充分性认定国家，允许

        if tool.has_scc():
            return TransferDecision.ALLOWED_WITH_SCC  # 有标准合同条款

        return TransferDecision.BLOCKED  # 不符合数据传输条件
```

---

### 2. SOC 2 for Agent Systems — 服务组织的信任准则

SOC 2（Service Organization Control 2）由美国注册会计师协会（AICPA）发布，是 SaaS 和云服务最广泛接受的安全合规标准之一。对于部署 Agent 的企业，SOC 2 是客户信任的基础。

#### 五大信任服务准则的 Agent 映射

| 准则 | 传统含义 | Agent 系统含义 |
|------|---------|---------------|
| **安全性** | 保护系统免受未授权访问 | Agent 工具调用的鉴权、防注入、权限边界控制 |
| **可用性** | 系统正常运行 | Agent 推理服务、工具 API 的可用性保障 |
| **处理完整性** | 系统处理经过授权、完整、准确 | Agent 决策的完整性——幻觉检测、推理链验证 |
| **机密性** | 保护机密信息 | Agent memory 加密、工具调用加密、数据脱敏 |
| **隐私** | 符合隐私声明 | 同 GDPR 原则，Agent 数据处理透明、可控制 |

#### SOC 2 对 Agent 系统的关键控制要求

**① 访问控制（CC6.1）**

```python
class SOC2AccessControl:
    """SOC 2 CC6.1: Logical and physical access controls"""

    def authorize_agent_action(self, agent: Agent, action: Action, context: Context) -> bool:
        # 身份验证
        if not self.authenticate_agent(agent):
            self.audit_log.fail(agent, action, "authentication_failed")
            return False

        # 授权检查——最小权限原则
        if not self.check_permissions(agent.role, action.required_permissions):
            self.audit_log.fail(agent, action, "insufficient_permissions")
            return False

        # 环境风险检查
        if self.is_high_risk_context(context):
            # 高风险操作需要额外批准
            return self.require_human_approval(agent, action)

        self.audit_log.success(agent, action)
        return True
```

**② 审计日志（CC6.2）**

- Agent 的所有工具调用、数据访问、配置变更必须记录
- 日志必须包含：时间戳、用户/Agent 身份、操作类型、资源、结果（成功/失败）
- 日志必须防篡改（详见 12.6.1 Audit Logging）

**③ 变更管理（CC6.3）**

- Agent 的 prompt 变更、工具定义变更、权限配置变更都需要受控的变更管理流程
- Agent 专属：prompt 版本控制、模型版本升级审批、工具注册/注销流程

```python
class SOC2ChangeManagement:
    """SOC 2 CC6.3: Change management for Agent systems"""

    def deploy_agent_change(self, change: AgentChange) -> DeployResult:
        # 1. 变更分类
        change_type = self.classify_change(change)
        #   - prompt_change: 系统提示修改
        #   - tool_change: 工具注册/注销/参数修改
        #   - model_change: 底层 LLM 模型升级
        #   - permission_change: 权限配置修改
        #   - config_change: 运行参数修改

        # 2. 自动化合规检查
        compliance_issues = self.run_compliance_checks(change)
        if compliance_issues:
            return DeployResult.rejected(compliance_issues)

        # 3. 审批流（高风险变更需人工审批）
        if change_type in self.HIGH_RISK_CHANGES:
            approval = self.request_approval(change, ["security_team", "compliance_team"])
            if not approval.granted:
                return DeployResult.rejected(["approval_denied"])

        # 4. 灰度部署
        canary_result = self.deploy_to_canary(change)
        if not canary_result.all_checks_passed:
            return DeployResult.rollback(canary_result.failures)

        # 5. 全量部署 + 变更记录
        deploy_result = self.full_deploy(change)
        self.change_log.record(change, deploy_result)
        return deploy_result
```

**④ 监控与检测（CC7.2）**

- 实时监控 Agent 异常行为：异常工具调用频率、数据访问模式突变、非工作时间活跃
- 设置告警阈值：单 Agent 每秒调用数、敏感数据访问频率、权限提升尝试

---

### 3. HIPAA Compliance — 医疗 Agent 的 PHI 保护

HIPAA（Health Insurance Portability and Accountability Act）为医疗保健领域的 Agent 系统设定了极为严格的数据保护要求。

#### PHI 在 Agent 系统中的特殊风险

Agent 系统在处理受保护健康信息（PHI）时面临传统系统没有的风险：

```
Agent 系统 PHI 暴露路径
═══════════════════════════════════════════════════════════════════

                   ┌──────────────────────┐
                   │   用户输入 PHI        │
                   │  "我最近被诊断出...   │
                   │   医生建议我..."     │
                   └──────────┬───────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │ LLM 推理      │   │ 外部 API 调用 │   │ 向量检索      │
  │              │   │              │   │              │
  │ PHI 可能     │   │ PHI 通过 URL/ │   │ PHI 被存为     │
  │ 出现在 prompt │   │ 参数传递给    │   │ embedding     │
  │ 和输出中     │   │ 第三方 API    │   │ 在向量库中    │
  └──────────────┘   └──────────────┘   └──────────────┘
          │                   │                   │
          ▼                   ▼                   ▼
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │ Agent 记忆    │   │ Agent 工具链  │   │ 训练数据      │
  │              │   │              │   │              │
  │ PHI 被持久化 │   │ PHI 在多个    │   │ PHI 可能被     │
  │ 在记忆中     │   │ 工具间传递    │   │ 用于后续训练  │
  └──────────────┘   └──────────────┘   └──────────────┘
```

#### HIPAA 对 Agent 系统的关键要求

**① PHI 识别与脱敏**

Agent 必须在运行时实时识别输入中的 PHI 并进行脱敏处理：

```python
class HIPAACompliantAgent:
    """HIPAA 合规的医疗 Agent"""

    PHI_PATTERNS = {
        "name": r"\b[A-Z][a-z]+ [A-Z][a-z]+\b",  # 简化示例
        "ssn": r"\b\d{3}-\d{2}-\d{4}\b",
        "mrn": r"\b(?:MRN|mrn)[:\s]*\d{6,10}\b",
        "phone": r"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b",
        "email": r"\b[\w.+-]+@[\w-]+\.[\w]{2,}\b",
        "dob": r"\b\d{1,2}/\d{1,2}/\d{4}\b",
        "address": r"\b\d{1,5}\s+[A-Za-z]+\s+(?:St|Ave|Rd|Dr|Blvd)\b",
    }

    def process_with_phi_protection(self, user_input: str) -> tuple[str, PHIActionReport]:
        # 1. PHI 检测
        phi_entities = self.detect_phi(user_input)
        report = PHIActionReport(detected_phi=phi_entities)

        if not phi_entities:
            # 不含 PHI → 正常处理
            result = self.agent.execute(user_input)
            return result, report

        # 2. PHI 脱敏（用令牌替换）
        deidentified_input = self.deidentify(user_input, phi_entities)
        report.add_action("deidentify", fields=list(phi_entities.keys()))

        # 3. 在脱敏后的输入上执行 Agent 推理
        result = self.agent.execute(deidentified_input)

        # 4. 如果有必要，将 PHI 重新注入到输出中
        #    （仅在明确授权的情况下）
        if self.context_authorizes_reidentification():
            final_result = self.reidentify(result, phi_entities)
            report.add_action("reidentify", authorized=True)
        else:
            final_result = result
            report.add_action("reidentify", authorized=False)

        return final_result, report
```

**② 业务伙伴协议（BAA — Business Associate Agreement）**

任何处理 PHI 的 Agent 系统底层服务都需要 BAA：

| 服务类型 | 需要 BAA | 常见服务商 |
|----------|---------|-----------|
| LLM API 提供商 | ✅ 必须 | Azure OpenAI（签 BAA）、AWS Bedrock（签 BAA） |
| 向量数据库 | ✅ 必须 | Pinecone（签 BAA）、MongoDB Atlas（签 BAA） |
| 云基础设施 | ✅ 必须 | AWS/Azure/GCP 企业版 |
| 监控/日志服务 | ✅ 必须 | Datadog（签 BAA）、Splunk（签 BAA） |
| 公共 LLM API | ❌ 不得使用 | OpenAI 公共 API（不签 BAA） |

```python
class BAAManager:
    """BAA 合规管理——确保 Agent 工具链中的服务都有 BAA"""

    BAA_PROVIDERS = {
        "azure-openai": BAAContract(provider="Microsoft", signed=True, expires="2026-12-31"),
        "aws-bedrock": BAAContract(provider="AWS", signed=True, expires="2027-06-30"),
        "pinecone": BAAContract(provider="Pinecone", signed=True, expires="2026-09-30"),
        "datadog": BAAContract(provider="Datadog", signed=True, expires="2026-08-15"),
    }

    def validate_agent_toolchain(self, agent: Agent) -> ComplianceReport:
        report = ComplianceReport()
        for tool in agent.get_toolchain():
            provider = tool.provider
            if tool.processes_phi:
                baa = self.BAA_PROVIDERS.get(provider)
                if not baa:
                    report.add_finding(
                        severity="CRITICAL",
                        message=f"Tool {tool.name} processes PHI but provider {provider} has no BAA"
                    )
                elif baa.is_expired():
                    report.add_finding(
                        severity="HIGH",
                        message=f"BAA for {provider} expired on {baa.expires}"
                    )
                else:
                    report.add_finding(
                        severity="INFO",
                        message=f"Tool {tool.name} has valid BAA through {baa.expires}"
                    )
        return report
```

**③ 访问控制（HIPAA Security Rule §164.312）**

- 唯一用户标识：每个 Agent 会话绑定到已验证的用户身份
- 紧急访问程序：在紧急情况下（如患者危急），允许打破常规权限但必须记录
- 自动登出：Agent 会话在设定时间内无活动后自动终止并清除会话数据
- 加密与解密：传输中（TLS 1.3）和存储中（AES-256）的 PHI 必须加密

**④ 审计控制（HIPAA Security Rule §164.312(b)）**

- Agent 系统必须记录所有涉及 PHI 的访问和操作
- 审计日志必须包含：谁（哪个用户/Agent 实例）、什么（操作类型）、什么时候（时间戳）、什么数据（PHI 标识符, 脱敏后）
- 审计日志保留期：至少 6 年（HIPAA 要求）

---

### 4. EU AI Act 2025 — AI 风险分类与 Agent 系统

EU AI Act 是**全球首个全面的 AI 立法**，采用基于风险的分级监管框架。它对 Agent 系统的影响将是深远的。

#### 风险分类体系

```
EU AI Act 风险金字塔
═══════════════════════════════════════════════════════════════════

                                        ┌──────────────────────┐
                                        │   不可接受风险         │
                                        │   (Prohibited)       │  ❌ 禁止
                                        │                      │
                                        │  • 社会评分系统       │
                                        │  • 实时生物特征监控   │
                                        │  • 操纵人类行为       │
                                        └──────────────────────┘
                               ┌──────────────────────────────────────┐
                               │     高风险 (High Risk)                │  🟡 严格监管
                               │                                      │
                               │  • 医疗设备 AI 组件                   │
                               │  • 关键基础设施管理                    │
                               │  • 教育/就业/信贷决策                  │
                               │  • 执法/移民/司法 AI                   │
                               │  • → 自主决策型 Agent 很可能在此      │
                               └──────────────────────────────────────┘
              ┌─────────────────────────────────────────────────────────────┐
              │     有限风险 (Limited Risk)                                  │  🔵 透明度
              │                                                             │
              │  • 聊天机器人（必须告知用户在与 AI 互动）                     │
              │  • 深度伪造（必须标注）                                      │
              │  • Agent 需要披露：你在与 AI Agent 交互                     │
              └─────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────────────────────────┐
│     最小风险 (Minimal Risk)                                                     │  ✅ 无约束
│                                                                                │
│  • AI 游戏、AI 垃圾邮件过滤器、AI 库存管理                                       │
│  • 非交互式、非决策型 Agent 可能归入此类                                        │
└────────────────────────────────────────────────────────────────────────────────┘
```

#### Agent 系统的高风险判定

一个 Agent 系统是否属于高风险，取决于其应用领域和功能：

```python
class EUAIActRiskClassifier:
    """EU AI Act 高风险分类器 for Agent 系统"""

    HIGH_RISK_DOMAINS = {
        "biometric_identification": False,
        "critical_infrastructure": {"energy", "transport", "water"},
        "education": {"exam_scoring", "admission_decisions", "learning_outcome_monitoring"},
        "employment": {"recruitment", "performance_evaluation", "promotion_decisions"},
        "credit": {"credit_scoring", "loan_approval", "insurance_pricing"},
        "law_enforcement": {"risk_assessment", "evidence_analysis", "profiling"},
        "migration": {"visa_processing", "asylum_decisions"},
        "justice": {"judicial_decisions", "legal_research"},
        "healthcare": {"triage", "diagnosis_support", "treatment_recommendation"},
    }

    def classify_agent(self, agent: Agent) -> RiskClassification:
        domain = agent.domain
        autonomy_level = agent.autonomy_level
        decision_impact = agent.decision_impact

        # 1. 禁止类检查
        if agent.capabilities & AgentCapability.SOCIAL_SCORING:
            return RiskClassification.PROHIBITED
        if agent.capabilities & AgentCapability.REAL_TIME_BIOMETRIC:
            return RiskClassification.PROHIBITED

        # 2. 高风险检查
        if domain in self.HIGH_RISK_DOMAINS:
            domain_uses = self.HIGH_RISK_DOMAINS[domain]
            if isinstance(domain_uses, set) and agent.use_case in domain_uses:
                return RiskClassification.HIGH_RISK
            if isinstance(domain_uses, bool) and domain_uses:
                return RiskClassification.HIGH_RISK

        # Agent 专属高风险信号
        high_risk_signals = 0
        if autonomy_level >= AutonomyLevel.FULLY_AUTONOMOUS:
            high_risk_signals += 1
        if decision_impact >= DecisionImpact.HIGH:
            high_risk_signals += 1
        if agent.processes_personal_data and agent.makes_automated_decisions:
            high_risk_signals += 1

        if high_risk_signals >= 2:
            return RiskClassification.HIGH_RISK

        # 3. 有限风险
        if agent.interacts_with_users:
            return RiskClassification.LIMITED_RISK

        # 4. 最小风险
        return RiskClassification.MINIMAL_RISK
```

#### 高风险 Agent 的合规义务

如果 Agent 被归类为高风险，必须满足以下义务：

| 义务 | 要求 | 对 Agent 系统的实施 |
|------|------|-------------------|
| **风险管理系统** | 建立持续的、迭代的风险管理流程 | Agent 行为监控 + 风险评分 + 自动降级 |
| **技术文档** | 详细的系统设计、用途、合规性证明 | Agent 架构文档、推理链路图、工具注册表 |
| **记录保持** | 自动记录高风险决策和事件 | Agent 审计日志 + 决策追溯 |
| **透明度和信息披露** | 告知用户在与 AI 系统交互 | Agent 身份声明 + 能力边界声明 |
| **人工监督** | 高风险决策必须有人类确认 | 人工-in-the-loop 审批，Agent 不能做最终决策 |
| **准确性、鲁棒性、网络安全** | 系统必须达到设定的准确率，对错误有弹性 | Agent 幻觉检测 + 注入防护 + 回退策略 |

```python
class HighRiskAgentOversight:
    """EU AI Act 高风险 Agent 的人工监督机制"""

    DECISION_THRESHOLDS = {
        "medical_diagnosis": {"confidence_min": 0.95, "human_approval_required": True},
        "loan_approval": {"confidence_min": 0.90, "human_approval_required": True},
        "recruitment_shortlist": {"confidence_min": 0.85, "human_approval_required": True},
        "customer_support": {"confidence_min": 0.70, "human_approval_required": False},
    }

    def execute_with_oversight(self, agent: Agent, task: Task) -> DecisionResult:
        config = self.DECISION_THRESHOLDS.get(task.domain, {})
        confidence_min = config.get("confidence_min", 0.80)
        human_approval = config.get("human_approval_required", False)

        # Agent 提出建议
        proposal = agent.propose_decision(task)
        confidence = proposal.confidence_score

        # 记录决策过程
        self.audit_log.record_decision(
            agent_id=agent.id,
            task=task,
            proposal=proposal,
            confidence=confidence,
        )

        # 自动决策（低风险 + 高置信度）
        if not human_approval and confidence >= confidence_min:
            agent.execute(proposal)
            return DecisionResult.AUTO_APPROVED

        # 需要人工审批
        approval = self.request_human_approval(
            agent=agent,
            proposal=proposal,
            confidence=confidence,
            reasoning=proposal.reasoning_chain,
        )

        if approval.granted:
            agent.execute(proposal)
            return DecisionResult.HUMAN_APPROVED
        else:
            return DecisionResult.HUMAN_REJECTED
```

---

### 5. NIST AI Risk Management Framework — 风险管理流程

NIST AI RMF（National Institute of Standards and Technology AI Risk Management Framework）由美国商务部发布，是一个**自愿性**的 AI 风险管理框架，分为四个核心功能：Govern、Map、Measure、Manage。

#### NIST AI RMF 的 Agent 映射

```
NIST AI RMF 框架
═══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────────┐
│  GOVERN（治理）                                                      │
│  • 建立 AI 风险管理文化                                               │
│  • 制定 AI 策略、政策、流程                                            │
│  • 明确角色与职责（AI 合规官、伦理委员会）                                │
│  • Agent 特化：Agent 行为准则、工具使用政策、模型准入标准                  │
└─────────────────────────────────────────────────────────────────────┘
                               │
           ┌───────────────────┼───────────────────┐
           ▼                   ▼                   ▼
┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
│  MAP（映射）       │ │  MEASURE（度量）   │ │  MANAGE（管理）    │
│                   │ │                   │ │                   │
│ • 理解 Agent 的    │ │ • 测试 Agent 的    │ │ • 处理已识别的     │
│   上下文和风险场景  │ │   行为和输出       │ │   风险             │
│ • 识别 Agent 的    │ │ • 使用红队测试     │ │ • 实施风险缓解      │
│   利益相关者       │ │ • 持续监控 Agent   │ │   措施              │
│ • Agent 能力映射   │ │ • 评估偏见/准确性  │ │ • 制定应急计划      │
│ • 数据流和工具链   │ │ • 合规差距分析     │ │ • 监控风险状态      │
│   映射             │ │                   │ │ • 持续改进          │
└───────────────────┘ └───────────────────┘ └───────────────────┘
                               │                   │
                               └───────────────────┘
                                      │
                                      ▼
                               ┌───────────────────┐
                               │ 反馈循环           │
                               │                   │
                               │ MAP → MEASURE →   │
                               │ MANAGE → 反馈到    │
                               │ GOVERN 和 MAP     │
                               └───────────────────┘
```

#### NIST AI RMF 的 Agent 系统实施

```python
class NISTAIRMFAgent:
    """基于 NIST AI RMF 的风险管理 Agent"""

    def __init__(self):
        self.risk_register = RiskRegister()
        self.metrics_store = MetricsStore()

    # ── MAP 阶段：风险识别 ──
    def map_agent_risks(self, agent: Agent) -> RiskMap:
        """Mapping phase: identify and characterize risks"""
        risks = []

        # Agent 能力映射
        risks.extend(self._map_capability_risks(agent))

        # 数据流风险
        risks.extend(self._map_data_flow_risks(agent))

        # 工具链风险
        risks.extend(self._map_toolchain_risks(agent))

        # 利益相关者影响
        risks.extend(self._map_stakeholder_impact(agent))

        # 对抗性威胁
        risks.extend(self._map_adversarial_risks(agent))

        return RiskMap(agent_id=agent.id, risks=risks)

    # ── MEASURE 阶段：风险评估 ──
    def measure_agent_risks(self, agent: Agent, risk_map: RiskMap) -> RiskAssessment:
        """Measure phase: assess, test, and score risks"""
        for risk in risk_map.risks:
            # 可能性评分（1-5）
            likelihood = self._assess_likelihood(risk, agent)

            # 影响程度评分（1-5）
            impact = self._assess_impact(risk, agent)

            # 风险等级 = 可能性 × 影响
            risk_score = likelihood * impact

            # 测试验证
            test_result = self._run_risk_test(risk, agent)

            self.risk_register.add(risk.id, risk_score, test_result)

        return self.risk_register.summarize()

    # ── MANAGE 阶段：风险处理 ──
    def manage_agent_risks(self, agent: Agent, assessment: RiskAssessment):
        """Manage phase: treat, mitigate, and monitor risks"""
        for risk_item in assessment.high_risks:
            # 为高风险制定缓解措施
            mitigation = self._design_mitigation(risk_item)

            # 实施缓解措施
            self._implement_mitigation(agent, risk_item.id, mitigation)

            # 设定持续监控指标
            self.metrics_store.add_metric(
                name=f"risk_{risk_item.id}_score",
                threshold=risk_item.acceptable_threshold,
                evaluation_fn=lambda a: self._reassess_risk(risk_item.id, a)
            )
```

---

### 6. ISO/IEC 42001: AI Management System — AI 管理体系

ISO/IEC 42001 是**全球首个 AI 管理体系标准**（2023 年 12 月发布），为组织提供建立、实施、维护和持续改进 AI 管理体系的框架。它不规定具体的合规要求，而是提供管理体系框架。

#### ISO 42001 的 Agent 系统映射

```
ISO 42001 管理体系架构
═══════════════════════════════════════════════════════════════════

组织上下文 (Context of the Organization)
├── 理解组织及其背景
├── 理解利益相关者的需求和期望
├── 确定 AI 管理体系的范围
└── AI 管理体系 (AIMS)

领导力 (Leadership)
├── 领导力和承诺
├── AI 政策
├── 角色、职责和权限
└── AI 伦理委员会

规划 (Planning)
├── 应对风险和机遇
├── AI 风险评估
├── AI 影响评估
├── AI 目标设定
└── 变更管理

支持 (Support)
├── 资源
├── 能力
├── 意识
├── 沟通
├── 文档化信息
└── AI 知识与数据管理

运行 (Operation)
├── 运行规划和控制
├── AI 系统生命周期管理
│   ├── 设计 → 开发 → 部署 → 运行 → 退役
│   └── Agent 特化：prompt 设计、工具开发、模型选择、运行时监控
└── 与外部提供者的接口

绩效评估 (Performance Evaluation)
├── 监控、测量、分析和评估
├── 内部审计
├── 管理评审
└── AI 系统绩效指标

改进 (Improvement)
├── 不符合项和纠正措施
└── 持续改进
```

#### Agent 系统生命周期管理的关键控制点

| 生命周期阶段 | ISO 42001 要求 | Agent 特化控制 |
|-------------|---------------|---------------|
| **设计** | AI 影响评估、风险识别 | Agent 能力边界定义、工具权限模型设计 |
| **开发** | 数据治理、模型选择 | Prompt 工程规范、工具开发标准、测试覆盖 |
| **部署** | 运行控制、变更管理 | 灰度部署、合规检查 gates、人工审批 |
| **运行** | 监控、事件响应 | Agent 行为监控、异常告警、错误恢复 |
| **退役** | 数据清理、知识迁移 | Agent 记忆清除、工具注销、训练数据销毁 |

```python
class ISO42001AIMS:
    """ISO 42001 AI Management System for Agent"""

    def __init__(self):
        self.policy = None
        self.risk_assessments = []
        self.audit_trail = AuditTrail()
        self.corrective_actions = []

    def lifecycle_gate_check(self, agent: Agent, phase: LifecyclePhase) -> GateResult:
        """Agent 生命周期门禁检查"""
        checks = []

        if phase == LifecyclePhase.DESIGN:
            # AI 影响评估
            impact = self._ai_impact_assessment(agent)
            checks.append(self._check("AI Impact Assessment", impact.is_acceptable))

            # 风险识别
            risks = self._identify_design_risks(agent)
            checks.append(self._check("Design Risk Identification", len(risks) > 0))

        elif phase == LifecyclePhase.DEPLOY:
            # 运行控制检查
            operational_readiness = self._check_operational_readiness(agent)
            checks.append(self._check("Operational Readiness", operational_readiness))

            # 合规检查
            compliance = self._check_deployment_compliance(agent)
            checks.append(self._check("Deployment Compliance", compliance.is_clean))

            # 监控就绪
            monitoring = self._check_monitoring_readiness(agent)
            checks.append(self._check("Monitoring Readiness", monitoring.is_ready))

        elif phase == LifecyclePhase.RETIREMENT:
            # 数据清理
            data_cleared = self._clear_agent_data(agent)
            checks.append(self._check("Data Clearance", data_cleared))

            # 审计记录归档
            audit_archived = self._archive_audit_trail(agent)
            checks.append(self._check("Audit Archival", audit_archived))

        # 记录本次门禁检查
        result = GateResult(phase=phase, checks=checks, passed=all(c.passed for c in checks))
        self.audit_trail.record_gate(agent.id, result)
        return result
```

---

### 7. Sector-Specific Compliance — 行业特定合规

不同行业的 Agent 系统面临不同的监管要求。

#### FINRA（金融业监管局）—— 金融 Agent

金融 Agent（如交易助手、投资顾问、客户服务）受 FINRA 严格监管：

| FINRA 规则 | 要求 | Agent 系统实施 |
|-----------|------|--------------|
| **4511 记录保留** | 所有与客户通信的记录必须保留 3-7 年 | Agent 对话日志、决策记录、工具调用审计 |
| **3110 监管** | 所有客户通信必须受监督 | Agent 输出合规检查、关键词标记、人工抽检 |
| **2210 通信内容** | 不得有误导性陈述、必须公平平衡 | Agent 输出真实性验证、免责声明自动添加 |
| **5310 最佳执行** | 交易必须为客户争取最佳条件 | Agent 交易决策的公平性审计 |

```python
class FINRACompliantTradingAgent:
    """FINRA 合规的金融交易 Agent"""

    SUPERVISED_KEYWORDS = [
        "guaranteed", "risk-free", "no risk", "guaranteed returns",
        "certain profit", "beat the market", "insider",
    ]

    def generate_advice(self, user_input: str) -> tuple[str, ComplianceCheck]:
        # Agent 生成投资建议
        advice = self.agent.generate(user_input)

        # 合规检查
        compliance_flags = []
        for word in self.SUPERVISED_KEYWORDS:
            if word.lower() in advice.lower():
                compliance_flags.append(f"Flagged term: '{word}' — requires disclaimer")

        # 自动添加免责声明
        if any(f.startswith("Flagged") for f in compliance_flags):
            advice += "\n\n[Required Disclaimer: This is not financial advice. Past performance does not guarantee future results.]"

        # 记录审计
        self.audit_log.record_communication(
            agent_id=self.agent.id,
            user_id=user_input.user_id,
            original_input=user_input.text,
            output=advice,
            compliance_flags=compliance_flags,
            requires_supervisory_review=len(compliance_flags) > 0,
        )

        # 高风险：需要人工监管审查
        if len(compliance_flags) > 0:
            self.supervisory_queue.add(
                agent_id=self.agent.id,
                message=advice,
                flags=compliance_flags,
                reviewer_assigned=None,
            )

        return advice, ComplianceCheck(flags=compliance_flags, passed=len(compliance_flags) == 0)
```

#### FDA / 医疗设备法规 — 医疗 Agent

作为医疗器械（SaMD — Software as a Medical Device）的 Agent 系统：

- **FDA 510(k) 预市场通知**：如果 Agent 作为医疗器械的软件组件，需要 510(k) 审批
- **IEC 62304**：医疗设备软件生命周期过程
- **21 CFR Part 11**：电子记录和电子签名
- **FDA AI/ML SaMD 行动计划**：AI/ML 作为医疗器械的监管框架

#### FCC（联邦通信委员会）—— 通信 Agent

- **TCPA（电话消费者保护法）**：Agent 自动拨号或发送消息受限制
- **CALEA（通信执法协助法案）**：通信 Agent 必须支持执法监听
- **Robocall 限制**：Agent 批量外呼受到严格限制

---

### 8. Compliance Automation — 合规自动化

Agent 系统的速度和数量使得人工合规检查完全不可行。合规自动化是唯一的可行路径。

#### 合规即代码（Compliance as Code）

```yaml
# compliance-policies/agent-gdpr.yaml
# 合规即代码：GDPR 合规策略定义
apiVersion: compliance.v1
kind: GDPRPolicy
metadata:
  name: agent-data-minimization
spec:
  scope:
    agentTypes: ["customer-support", "data-analysis"]
  rules:
    - id: "gdpr-5-1-c"
      title: "Data Minimization"
      description: "Agent must only collect data necessary for the stated purpose"
      check: |
        agent.collected_data_categories ⊆ purpose.data_categories
      remediation: |
        if check fails: block agent execution and log violation
    - id: "gdpr-17"
      title: "Right to Erasure"
      description: "Agent must support full user data erasure within 30 days"
      check: |
        agent.supports_erasure == true AND
        agent.erasure_timeout <= timedelta(days=30)
    - id: "gdpr-recital-71"
      title: "Right to Explanation"
      description: "Agent must provide meaningful explanation for automated decisions"
      check: |
        agent.supports_explanation == true AND
        agent.explanation_coverage >= 0.95
```

#### 自动化合规检查引擎

```python
class ComplianceChecker:
    """自动化合规检查引擎"""

    def __init__(self):
        self.policy_registry = PolicyRegistry()
        self.evidence_collector = EvidenceCollector()

    def check_agent_compliance(self, agent: Agent) -> ComplianceReport:
        """运行所有适用的合规检查"""
        report = ComplianceReport(agent_id=agent.id, timestamp=datetime.utcnow())

        # 收集合规证据
        evidence = self.evidence_collector.collect(agent)

        # 加载适用的策略
        applicable_policies = self.policy_registry.get_policies_for(agent)

        for policy in applicable_policies:
            for rule in policy.rules:
                try:
                    result = self._evaluate_rule(rule, evidence)
                    report.add_result(RuleResult(
                        rule_id=rule.id,
                        rule_title=rule.title,
                        framework=policy.framework,
                        passed=result.passed,
                        details=result.details,
                        evidence_refs=result.evidence_refs,
                    ))
                except Exception as e:
                    report.add_result(RuleResult(
                        rule_id=rule.id,
                        rule_title=rule.title,
                        framework=policy.framework,
                        passed=False,
                        details=f"Evaluation error: {str(e)}",
                    ))

        return report
```

#### 持续合规监控（Continuous Compliance Monitoring）

```python
class ContinuousComplianceMonitor:
    """Agent 持续合规监控"""

    def __init__(self):
        self.baselines = {}
        self.alerts = AlertManager()

    def start_monitoring(self, agent: Agent):
        baseline = self._establish_baseline(agent)
        self.baselines[agent.id] = baseline

        while agent.is_running:
            # 采样 Agent 行为
            window = self._get_recent_behavior_window(agent, minutes=5)

            # 偏差检测
            deviations = self._detect_compliance_deviations(window, baseline)

            for deviation in deviations:
                severity = self._classify_deviation(deviation)
                alert = ComplianceAlert(
                    agent_id=agent.id,
                    timestamp=datetime.utcnow(),
                    deviation=deviation,
                    severity=severity,
                )
                self.alerts.send(alert)

                # 自动缓解（如果可能）
                if severity == AlertSeverity.CRITICAL:
                    self._auto_mitigate(agent, deviation)

            # 更新基线（缓慢适应 Agent 正常行为漂移）
            if len(window) > 1000:
                baseline = self._update_baseline_smoothly(baseline, window)

            time.sleep(60)  # 每分钟检查一次
```

## Example Code: Python ComplianceChecker with GDPR/SOC2/HIPAA/EU AI Act rule evaluation

```python
"""
ComplianceChecker — 多框架合规检查引擎 for Agent Systems

支持 GDPR、SOC 2、HIPAA、EU AI Act 规则的自动化评估。
输出结构化合规报告，可集成到 CI/CD 流水线中。
"""

from enum import Enum, auto
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import Optional


# ═══════════════════════════════════════════════════════════════════
# 数据类型定义
# ═══════════════════════════════════════════════════════════════════

class Framework(Enum):
    GDPR = "GDPR"
    SOC2 = "SOC2"
    HIPAA = "HIPAA"
    EU_AI_ACT = "EU_AI_ACT"
    NIST_AI_RMF = "NIST_AI_RMF"
    ISO_42001 = "ISO_42001"


class Severity(Enum):
    CRITICAL = auto()
    HIGH = auto()
    MEDIUM = auto()
    LOW = auto()
    INFO = auto()


@dataclass
class RuleResult:
    rule_id: str
    rule_title: str
    framework: Framework
    passed: bool
    details: str = ""
    evidence_refs: list[str] = field(default_factory=list)


@dataclass
class ComplianceReport:
    agent_id: str
    timestamp: datetime
    results: list[RuleResult] = field(default_factory=list)
    summary: dict = field(default_factory=dict)

    def add_result(self, result: RuleResult):
        self.results.append(result)

    def generate_summary(self) -> dict:
        total = len(self.results)
        passed = sum(1 for r in self.results if r.passed)
        failed = total - passed
        by_framework = {}
        for r in self.results:
            by_framework.setdefault(r.framework, {"total": 0, "passed": 0})
            by_framework[r.framework]["total"] += 1
            if r.passed:
                by_framework[r.framework]["passed"] += 1
        self.summary = {
            "total_checks": total,
            "passed": passed,
            "failed": failed,
            "pass_rate": f"{passed/total*100:.1f}%" if total > 0 else "N/A",
            "by_framework": {
                k: {
                    "total": v["total"],
                    "passed": v["passed"],
                    "pass_rate": f"{v['passed']/v['total']*100:.1f}%"
                }
                for k, v in by_framework.items()
            },
            "verdict": "PASS" if failed == 0 else "FAIL",
        }
        return self.summary


# ═══════════════════════════════════════════════════════════════════
# Agent 系统模型（简化的示例）
# ═══════════════════════════════════════════════════════════════════

@dataclass
class AgentSystem:
    id: str
    name: str
    domain: str
    processes_personal_data: bool
    processes_phi: bool  # HIPAA
    makes_automated_decisions: bool
    interacts_with_users: bool
    supports_erasure: bool
    supports_explanation: bool
    has_consent_management: bool
    has_audit_logging: bool
    has_access_control: bool
    has_encryption_at_rest: bool
    has_encryption_in_transit: bool
    has_baa: bool  # HIPAA
    has_human_oversight: bool
    data_retention_days: int
    autonomy_level: str  # "full", "partial", "none"


# ═══════════════════════════════════════════════════════════════════
# 合规检查引擎
# ═══════════════════════════════════════════════════════════════════

class ComplianceChecker:
    """Multi-framework compliance checker for Agent systems"""

    def __init__(self):
        self.rules_registry = self._register_rules()

    def _register_rules(self) -> dict[str, callable]:
        return {
            # ── GDPR Rules ──
            "gdpr-5-1-c": self._check_gdpr_data_minimization,
            "gdpr-17": self._check_gdpr_right_to_erasure,
            "gdpr-recital-71": self._check_gdpr_right_to_explanation,
            "gdpr-7": self._check_gdpr_consent,
            "gdpr-32": self._check_gdpr_security,

            # ── SOC 2 Rules ──
            "soc2-cc6-1": self._check_soc2_access_control,
            "soc2-cc6-2": self._check_soc2_audit_logging,
            "soc2-cc7-2": self._check_soc2_monitoring,

            # ── HIPAA Rules ──
            "hipaa-164-312-a-1": self._check_hipaa_access_control,
            "hipaa-164-312-a-2": self._check_hipaa_audit_control,
            "hipaa-164-312-e-1": self._check_hipaa_encryption,
            "hipaa-baa": self._check_hipaa_baa,

            # ── EU AI Act Rules ──
            "eu-ai-act-article-6": self._check_eu_high_risk_classification,
            "eu-ai-act-article-14": self._check_eu_human_oversight,
            "eu-ai-act-article-52": self._check_eu_transparency,
        }

    def evaluate(self, agent: AgentSystem, frameworks: Optional[list[Framework]] = None) -> ComplianceReport:
        """Evaluate agent compliance against specified frameworks (or all)"""
        report = ComplianceReport(agent_id=agent.id, timestamp=datetime.utcnow())

        for rule_id, check_fn in self.rules_registry.items():
            framework = self._extract_framework(rule_id)
            if frameworks and framework not in frameworks:
                continue

            try:
                result = check_fn(agent)
                report.add_result(result)
            except Exception as e:
                report.add_result(RuleResult(
                    rule_id=rule_id,
                    rule_title=rule_id,
                    framework=framework,
                    passed=False,
                    details=f"Check evaluation error: {str(e)}",
                ))

        report.generate_summary()
        return report

    def _extract_framework(self, rule_id: str) -> Framework:
        mapping = {
            "gdpr": Framework.GDPR,
            "soc2": Framework.SOC2,
            "hipaa": Framework.HIPAA,
            "eu-ai": Framework.EU_AI_ACT,
        }
        for prefix, framework in mapping.items():
            if rule_id.startswith(prefix):
                return framework
        return Framework.GDPR

    # ────────────────────────────────────────────────────────────
    # GDPR Rule Checks
    # ────────────────────────────────────────────────────────────

    def _check_gdpr_data_minimization(self, agent: AgentSystem) -> RuleResult:
        passed = agent.processes_personal_data == False or agent.has_consent_management
        return RuleResult(
            rule_id="gdpr-5-1-c",
            rule_title="Data Minimization (Art. 5(1)(c))",
            framework=Framework.GDPR,
            passed=passed,
            details=(
                "Agent has consent management and collects minimal data."
                if passed else
                "Agent processes personal data without consent management — cannot enforce data minimization."
            ),
            evidence_refs=["consent_management_enabled", "data_collection_policy"],
        )

    def _check_gdpr_right_to_erasure(self, agent: AgentSystem) -> RuleResult:
        passed = agent.supports_erasure and agent.data_retention_days <= 365
        return RuleResult(
            rule_id="gdpr-17",
            rule_title="Right to Erasure (Art. 17)",
            framework=Framework.GDPR,
            passed=passed,
            details=(
                f"Agent supports erasure with {agent.data_retention_days}-day retention."
                if passed else
                f"Agent lacks erasure support or retention period ({agent.data_retention_days}d) exceeds limit."
            ),
            evidence_refs=["erasure_api", "retention_policy"],
        )

    def _check_gdpr_right_to_explanation(self, agent: AgentSystem) -> RuleResult:
        if not agent.makes_automated_decisions:
            return RuleResult(
                rule_id="gdpr-recital-71",
                rule_title="Right to Explanation (Recital 71)",
                framework=Framework.GDPR,
                passed=True,
                details="Agent does not make automated decisions — not applicable.",
            )
        passed = agent.supports_explanation
        return RuleResult(
            rule_id="gdpr-recital-71",
            rule_title="Right to Explanation (Recital 71)",
            framework=Framework.GDPR,
            passed=passed,
            details=(
                "Agent provides meaningful explanations for automated decisions."
                if passed else
                "Agent makes automated decisions but does not support explanation — HIGH RISK."
            ),
            evidence_refs=["explanation_api", "decision_trace_logs"],
        )

    def _check_gdpr_consent(self, agent: AgentSystem) -> RuleResult:
        if not agent.processes_personal_data:
            return RuleResult(
                rule_id="gdpr-7",
                rule_title="Consent (Art. 7)",
                framework=Framework.GDPR,
                passed=True,
                details="Agent does not process personal data — not applicable.",
            )
        passed = agent.has_consent_management
        return RuleResult(
            rule_id="gdpr-7",
            rule_title="Consent (Art. 7)",
            framework=Framework.GDPR,
            passed=passed,
            details=(
                "Consent management is implemented."
                if passed else
                "Agent processes personal data without consent management — VIOLATION."
            ),
        )

    def _check_gdpr_security(self, agent: AgentSystem) -> RuleResult:
        passed = agent.has_encryption_at_rest and agent.has_encryption_in_transit
        return RuleResult(
            rule_id="gdpr-32",
            rule_title="Security of Processing (Art. 32)",
            framework=Framework.GDPR,
            passed=passed,
            details=(
                f"Encryption at rest: {agent.has_encryption_at_rest}, in transit: {agent.has_encryption_in_transit}."
                if passed else
                "Missing encryption controls — GDPR Art. 32 violation."
            ),
            evidence_refs=["encryption_config"],
        )

    # ────────────────────────────────────────────────────────────
    # SOC 2 Rule Checks
    # ────────────────────────────────────────────────────────────

    def _check_soc2_access_control(self, agent: AgentSystem) -> RuleResult:
        passed = agent.has_access_control
        return RuleResult(
            rule_id="soc2-cc6-1",
            rule_title="Logical and Physical Access Controls (CC6.1)",
            framework=Framework.SOC2,
            passed=passed,
            details=(
                "Access controls are implemented for Agent operations."
                if passed else
                "No access controls — SOC 2 CC6.1 violation."
            ),
        )

    def _check_soc2_audit_logging(self, agent: AgentSystem) -> RuleResult:
        passed = agent.has_audit_logging
        return RuleResult(
            rule_id="soc2-cc6-2",
            rule_title="Audit Logging (CC6.2)",
            framework=Framework.SOC2,
            passed=passed,
            details=(
                "Agent operations are audited with tamper-proof logs."
                if passed else
                "No audit logging — SOC 2 CC6.2 violation."
            ),
        )

    def _check_soc2_monitoring(self, agent: AgentSystem) -> RuleResult:
        passed = agent.has_audit_logging  # 简化：有审计日志意味着有监控基础
        return RuleResult(
            rule_id="soc2-cc7-2",
            rule_title="Monitoring and Detection (CC7.2)",
            framework=Framework.SOC2,
            passed=passed,
            details=(
                "Agent behavior monitoring is enabled."
                if passed else
                "No monitoring — SOC 2 CC7.2 violation."
            ),
        )

    # ────────────────────────────────────────────────────────────
    # HIPAA Rule Checks
    # ────────────────────────────────────────────────────────────

    def _check_hipaa_access_control(self, agent: AgentSystem) -> RuleResult:
        if not agent.processes_phi:
            return RuleResult(
                rule_id="hipaa-164-312-a-1",
                rule_title="Access Control (§164.312(a)(1))",
                framework=Framework.HIPAA,
                passed=True,
                details="Agent does not process PHI — not applicable.",
            )
        passed = agent.has_access_control
        return RuleResult(
            rule_id="hipaa-164-312-a-1",
            rule_title="Access Control (§164.312(a)(1))",
            framework=Framework.HIPAA,
            passed=passed,
            details=(
                "Unique user identification and access controls in place."
                if passed else
                "No access controls for PHI — HIPAA violation."
            ),
        )

    def _check_hipaa_audit_control(self, agent: AgentSystem) -> RuleResult:
        if not agent.processes_phi:
            return RuleResult(
                rule_id="hipaa-164-312-a-2",
                rule_title="Audit Controls (§164.312(a)(2))",
                framework=Framework.HIPAA,
                passed=True,
                details="Agent does not process PHI — not applicable.",
            )
        passed = agent.has_audit_logging
        return RuleResult(
            rule_id="hipaa-164-312-a-2",
            rule_title="Audit Controls (§164.312(a)(2))",
            framework=Framework.HIPAA,
            passed=passed,
            details=(
                "PHI access is fully audited."
                if passed else
                "No audit controls for PHI — HIPAA violation."
            ),
        )

    def _check_hipaa_encryption(self, agent: AgentSystem) -> RuleResult:
        if not agent.processes_phi:
            return RuleResult(
                rule_id="hipaa-164-312-e-1",
                rule_title="Encryption (§164.312(e)(1))",
                framework=Framework.HIPAA,
                passed=True,
                details="Agent does not process PHI — not applicable.",
            )
        passed = agent.has_encryption_at_rest and agent.has_encryption_in_transit
        return RuleResult(
            rule_id="hipaa-164-312-e-1",
            rule_title="Encryption (§164.312(e)(1))",
            framework=Framework.HIPAA,
            passed=passed,
            details=(
                "PHI encrypted at rest and in transit."
                if passed else
                "PHI not fully encrypted — HIPAA violation."
            ),
        )

    def _check_hipaa_baa(self, agent: AgentSystem) -> RuleResult:
        if not agent.processes_phi:
            return RuleResult(
                rule_id="hipaa-baa",
                rule_title="Business Associate Agreement",
                framework=Framework.HIPAA,
                passed=True,
                details="Agent does not process PHI — not applicable.",
            )
        passed = agent.has_baa
        return RuleResult(
            rule_id="hipaa-baa",
            rule_title="Business Associate Agreement",
            framework=Framework.HIPAA,
            passed=passed,
            details=(
                "BAA in place for all PHI-processing services."
                if passed else
                "No BAA — HIPAA violation."
            ),
        )

    # ────────────────────────────────────────────────────────────
    # EU AI Act Rule Checks
    # ────────────────────────────────────────────────────────────

    def _check_eu_high_risk_classification(self, agent: AgentSystem) -> RuleResult:
        # 高风险信号判定
        high_risk_domains = {"healthcare", "finance", "education", "employment", "law_enforcement"}
        is_high_risk = (
            agent.domain in high_risk_domains
            and agent.makes_automated_decisions
        )
        if not is_high_risk:
            return RuleResult(
                rule_id="eu-ai-act-article-6",
                rule_title="High-Risk Classification Rules (Art. 6)",
                framework=Framework.EU_AI_ACT,
                passed=True,
                details="Agent is not classified as high-risk under EU AI Act.",
            )
        # 高风险 Agent 需要额外控制
        passed = agent.has_human_oversight and agent.has_audit_logging
        return RuleResult(
            rule_id="eu-ai-act-article-6",
            rule_title="High-Risk Classification Rules (Art. 6)",
            framework=Framework.EU_AI_ACT,
            passed=passed,
            details=(
                f"Agent operates in high-risk domain '{agent.domain}' with automated decisions. "
                + ("Has human oversight and audit logging — compliant."
                   if passed else
                   "Lacks required human oversight or audit logging — NON-COMPLIANT.")
            ),
        )

    def _check_eu_human_oversight(self, agent: AgentSystem) -> RuleResult:
        if agent.domain not in {"healthcare", "finance", "education", "employment", "law_enforcement"}:
            return RuleResult(
                rule_id="eu-ai-act-article-14",
                rule_title="Human Oversight (Art. 14)",
                framework=Framework.EU_AI_ACT,
                passed=True,
                details="Not a high-risk domain — Art. 14 does not apply.",
            )
        passed = agent.has_human_oversight
        return RuleResult(
            rule_id="eu-ai-act-article-14",
            rule_title="Human Oversight (Art. 14)",
            framework=Framework.EU_AI_ACT,
            passed=passed,
            details=(
                "Human oversight mechanisms are in place for high-risk decisions."
                if passed else
                "No human oversight for high-risk Agent — EU AI Act Art. 14 violation."
            ),
        )

    def _check_eu_transparency(self, agent: AgentSystem) -> RuleResult:
        if not agent.interacts_with_users:
            return RuleResult(
                rule_id="eu-ai-act-article-52",
                rule_title="Transparency Obligations (Art. 52)",
                framework=Framework.EU_AI_ACT,
                passed=True,
                details="Agent does not interact with users — not applicable.",
            )
        # 简化：交互式 Agent 必须声明自己是 AI
        passed = True  # 假设有透明度声明
        return RuleResult(
            rule_id="eu-ai-act-article-52",
            rule_title="Transparency Obligations (Art. 52)",
            framework=Framework.EU_AI_ACT,
            passed=passed,
            details="Agent discloses AI identity to users.",
            evidence_refs=["transparency_disclosure", "user_facing_agent_label"],
        )


# ═══════════════════════════════════════════════════════════════════
# 使用示例
# ═══════════════════════════════════════════════════════════════════

if __name__ == "__main__":
    # 定义一个医疗领域的 Agent 系统
    medical_agent = AgentSystem(
        id="agent-007",
        name="HealthAssist Pro",
        domain="healthcare",
        processes_personal_data=True,
        processes_phi=True,
        makes_automated_decisions=True,
        interacts_with_users=True,
        supports_erasure=True,
        supports_explanation=True,
        has_consent_management=True,
        has_audit_logging=True,
        has_access_control=True,
        has_encryption_at_rest=True,
        has_encryption_in_transit=True,
        has_baa=True,
        has_human_oversight=True,
        data_retention_days=90,
        autonomy_level="partial",
    )

    checker = ComplianceChecker()
    report = checker.evaluate(
        agent=medical_agent,
        frameworks=[Framework.GDPR, Framework.HIPAA, Framework.EU_AI_ACT],
    )

    print(f"Compliance Report for: {medical_agent.name} ({medical_agent.id})")
    print(f"Timestamp: {report.timestamp}")
    print(f"Total Checks: {report.summary['total_checks']}")
    print(f"Passed: {report.summary['passed']}")
    print(f"Failed: {report.summary['failed']}")
    print(f"Pass Rate: {report.summary['pass_rate']}")
    print(f"Verdict: {report.summary['verdict']}")
    print()

    print("Detailed Results by Framework:")
    for framework, data in report.summary["by_framework"].items():
        print(f"  {framework.value}: {data['passed']}/{data['total']} ({data['pass_rate']})")

    print()
    failed_rules = [r for r in report.results if not r.passed]
    if failed_rules:
        print("Failed Rules:")
        for r in failed_rules:
            print(f"  ❌ [{r.framework.value}] {r.rule_id}: {r.details}")
    else:
        print("All checks passed!")
```

**预期输出示例**：

```
Compliance Report for: HealthAssist Pro (agent-007)
Timestamp: 2025-12-01 14:30:00
Total Checks: 14
Passed: 14
Failed: 0
Pass Rate: 100.0%
Verdict: PASS

Detailed Results by Framework:
  GDPR: 5/5 (100.0%)
  HIPAA: 5/5 (100.0%)
  EU_AI_ACT: 4/4 (100.0%)

All checks passed!
```

## Capability Boundaries — compliance != security, compliance is a lagging indicator

理解合规的局限性，对于 Agent 系统的安全设计至关重要：

```
合规的局限性
═══════════════════════════════════════════════════════════════════

┌────────────────────────────────────────────────────────────────┐
│                         合规 (Compliance)                        │
│                                                            │
│  • 检查"你做了该做的事吗？"                                    │
│  • 关注已知风险、已有规则                                      │
│  • 以最低标准为基线                                            │
│  • 滞后指标——合规审计反映的是过去的状态                          │
│  • 满足合规 ≠ 系统安全                                        │
└────────────────────────────────────────────────────────────────┘
                          ↑ 不等于
                          ↓
┌────────────────────────────────────────────────────────────────┐
│                       安全 (Security)                           │
│                                                            │
│  • 回答"你的系统真的安全吗？"                                  │
│  • 关注未知风险、0-day、对抗性攻击                              │
│  • 以风险容忍度为基线                                          │
│  • 领先指标——主动发现和修复漏洞                                │
│  • 安全状态 > 合规状态                                        │
└────────────────────────────────────────────────────────────────┘
```

### Agent 系统的关键认知

1. **合规是最低门槛，不是安全目标**
   - 满足 GDPR 不意味着你的 Agent 不会被注入攻击
   - 通过 SOC 2 审计不意味着你的 Agent memory 是安全的
   - 合规覆盖了已知风险清单；安全必须覆盖已知 + 未知风险

2. **合规是滞后指标（Lagging Indicator）**
   - GDPR 罚款是基于过去已经发生的数据泄露
   - SOC 2 报告反映的是过去 6-12 个月的控制有效性
   - 对于 Agent 系统，一周前的合规状态不能保证今天的安全

3. **合规框架赶不上 Agent 技术演进**
   - GDPR 2018 年发布时，ChatGPT 和 LLM Agent 还不存在
   - HIPAA 1996 年发布，对 LLM 和向量数据库的 PHI 处理没有直接规定
   - EU AI Act 2025 才生效，可能在 2030 年前就需要大修以适应 Agent 技术

4. **框架重叠导致复杂性**
   - 一个处理欧洲用户医疗数据的 Agent 可能同时适用 GDPR + HIPAA + EU AI Act
   - 不同框架的要求存在冲突（如数据保留期限不同）
   - 统一合规策略需要抽象层，而非逐个框架实现

## Comparison: GDPR vs SOC2 vs HIPAA vs EU AI Act requirements for Agent systems

| 维度 | GDPR | SOC 2 | HIPAA | EU AI Act |
|------|------|-------|-------|-----------|
| **核心焦点** | 个人数据保护 | 服务组织控制 | 健康信息保护 | AI 风险治理 |
| **适用范围** | 欧盟居民个人数据 | 服务组织（全球） | 美国 PHI | AI 系统（欧盟市场） |
| **法律性质** | 法规（硬性） | 行业标准（合同驱动） | 法规（硬性） | 法规（硬性） |
| **Agent 数据最小化** | ✅ 明确要求 | ✅ 隐含要求 | ✅ PHI 最小化 | ✅ 要求 |
| **Agent 解释权** | ⚠️ Recital 71（有限制） | ❌ 无明确要求 | ❌ 无明确要求 | ✅ 高风险系统明确要求 |
| **删除权** | ✅ Art. 17 | ❌ 无明确要求 | ✅ §164.502 | ❌ 无明确要求 |
| **审计日志** | ✅ 隐含（问责制原则） | ✅ CC6.2 明确要求 | ✅ §164.312(b) | ✅ 高风险系统要求 |
| **人工监督** | ✅ Art. 22 自动化决策 | ❌ 无明确要求 | ❌ 无明确要求 | ✅ Art. 14 高风险系统 |
| **风险分类** | ❌ 无 | ❌ 无 | ❌ 无 | ✅ 四层风险分类 |
| **认证/审计** | ⚠️ 自评估为主 | ✅ Type I/II 第三方审计 | ⚠️ 自评估 + 合规调查 | ✅ 公告机构评估（高风险） |
| **处罚上限** | 2000万€ / 4% 营收 | 合同约定（通常无法定上限） | $170万/年 | 3500万€ / 7% 营收 |
| **Agent 特有条款** | ❌ 无（通用数据法） | ❌ 无（通用安全法） | ❌ 无（通用医疗法） | ✅ AI 系统定义明确 |
| **适合的 Agent 类型** | 处理个人数据的 Agent | 企业 SaaS Agent | 医疗保健 Agent | 所有 EU 市场 Agent |

## Engineering Optimization — compliance-as-code, automated evidence collection, continuous compliance monitoring

Agent 系统合规工程化的三个核心优化方向：

### 1. Compliance-as-Code（合规即代码）

将合规要求编码为可执行的策略，使合规检查成为 CI/CD 流水线的一部分：

```
传统合规流程                             合规即代码流程
─────────────────                       ─────────────────────
人工阅读法规 → 解释 → 制作检查表            法规 → 策略代码 → 自动化检查
┌──────────┐  ┌──────────┐  ┌────────┐    ┌──────────┐  ┌──────────┐
│ 法规文本 │─►│ 合规团队 │─►│ 检查表 │    │ YAML/Code│─►│ CI/CD    │
│ （PDF）  │  │ 解读     │  │ Excel  │    │ 策略定义  │  │ Pipeline │
└──────────┘  └──────────┘  └────────┘    └──────────┘  └──────────┘
                                                    │
                                                    ▼
                                              ┌──────────┐
                                              │ 自动检查  │
                                              │ → 合规报告│
                                              │ → 阻断部署│
                                              └──────────┘
```

**优化效果**：
- 检查时间：从数周（人工审计）缩短到数秒（自动化检查）
- 覆盖率：从抽样的 20-30% 提升到 100%
- 可靠性：消除人工检查的主观差异

### 2. Automated Evidence Collection（自动化证据收集）

传统合规审计的最大成本是证据收集——人工收集日志、截图、配置。Agent 系统可以自动收集和整理合规证据：

```python
class AutomatedEvidenceCollector:
    """自动化合规证据收集器 for Agent 系统"""

    def collect_for_audit(self, agent_id: str, framework: Framework) -> AuditEvidence:
        evidence = AuditEvidence(agent_id=agent_id, framework=framework)

        # 1. Agent 配置证据
        evidence.add("system_prompt", self._snapshot_system_prompt(agent_id))
        evidence.add("tool_registry", self._snapshot_tool_registry(agent_id))
        evidence.add("permissions_config", self._snapshot_permissions(agent_id))

        # 2. 运行期证据
        evidence.add("audit_logs", self._export_audit_logs(agent_id, days_range=90))
        evidence.add("data_flow_map", self._generate_data_flow_map(agent_id))
        evidence.add("decision_traces", self._sample_decision_traces(agent_id, sample_size=100))

        # 3. 安全控制证据
        evidence.add("access_control_reports", self._get_access_reports(agent_id))
        evidence.add("encryption_config", self._get_encryption_config(agent_id))
        evidence.add("vulnerability_scans", self._get_vuln_scans(agent_id))

        # 4. 合规声明
        evidence.add("consent_records", self._export_consent_records(agent_id))
        evidence.add("data_retention_policy", self._get_retention_policy(agent_id))
        evidence.add("baa_agreements", self._get_baa_list(agent_id))

        return evidence
```

### 3. Continuous Compliance Monitoring（持续合规监控）—— 架构

```
Agent 系统持续合规监控架构
═══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│                   Agent 运行时                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Agent 1  │  │ Agent 2  │  │ Agent 3  │  │ Agent N  │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │              │              │              │            │
└───────┼──────────────┼──────────────┼──────────────┼────────────┘
        │              │              │              │
        ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    合规事件流管道                                   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  事件采集 (Event Collection)                               │   │
│  │  • 工具调用事件 • 数据访问事件 • 权限判定事件 • 配置变更事件  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  合规规则引擎 (Rule Engine)                                │   │
│  │  • GDPR 规则 • SOC 2 规则 • HIPAA 规则 • EU AI Act 规则   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  合规评分 & 告警 (Scoring & Alerting)                      │   │
│  │  • 实时合规分数 • 偏差告警 • 自动缓解触发 • 合规报告生成    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Agent 专属的合规工程实践清单

| 实践 | 描述 | 对应框架 |
|------|------|---------|
| **Prompt 版本控制** | Agent 每次 prompt 修改都记录版本、审批、部署时间 | SOC 2 CC6.3, ISO 42001 |
| **工具变更审计** | 工具注册/注销/参数修改必须记录并审批 | SOC 2 CC6.3 |
| **数据流自动映射** | Agent 工具调用链路自动生成数据流图 | GDPR Art. 30 |
| **自动 PII/PHI 检测** | 输入输出实时检测敏感数据并脱敏 | GDPR, HIPAA |
| **决策追溯生成** | 每次 Agent 决策自动生成可审计的追溯报告 | GDPR Art. 22, EU AI Act Art. 14 |
| **合规门禁** | Agent 部署必须通过合规检查 gate | SOC 2, ISO 42001 |
| **持续基线对比** | Agent 行为与基线对比，检测合规偏差 | NIST AI RMF |
| **自动化证据包** | 合规审计时自动打包所有证据 | 所有框架 |

### 5. 合规工程化成熟度模型

```
Level 1: 人工合规                       Level 2: 工具辅助合规
┌──────────────────────┐              ┌──────────────────────┐
│ • Excel 检查表       │      ──►     │ • 合规 wiki/文档库   │
│ • 年度人工审计        │              │ • 半自动化检查脚本   │
│ • 合规是法务部的事    │              │ • 合规团队主导       │
└──────────────────────┘              └──────────────────────┘

Level 3: 合规即代码                      Level 4: 持续合规
┌──────────────────────┐              ┌──────────────────────┐
│ • YAML 策略定义      │      ──►     │ • 实时合规监控       │
│ • CI/CD 合规门禁     │              │ • 自动证据收集       │
│ • 自动化合规报告      │              │ • 自动缓解 + 修复   │
│ • 工程团队参与        │              │ • 全组织合规文化     │
└──────────────────────┘              └──────────────────────┘
```

**目标：从"合规是检查"到"合规是系统属性"**——合规不再是一次性的审计活动，而是 Agent 系统运行时内建的持续属性。
