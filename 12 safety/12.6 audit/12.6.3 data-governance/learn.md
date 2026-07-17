# 12.6.3 data-governance — 数据治理

**Agent 是一台数据永动机——每轮推理、每次工具调用、每条记忆都在制造数据。** 没有数据治理，这些数据就会从资产变成负债：违规存储、隐私泄露、合规罚款。数据治理不是"管住数据"，而是在"利用数据改进 Agent"和"保护用户隐私"之间找到平衡。

## 基本原理

数据治理是对 Agent 系统中数据生命周期的系统性管理——从数据被创造的那一刻起，到它最终被安全销毁为止。它回答了四个核心问题：

```
Agent 数据治理四问
─────────────────────────────────────────────
① 我们收集了什么数据？          → 数据分类与打标
② 这些数据用来做什么？          → 目的限制与使用控制
③ 数据要存多久？               → 保留策略与自动过期
④ 数据怎么安全删除？           → 安全删除与删除验证
```

与传统系统不同，Agent 的数据治理面临独特挑战：
- **数据种类多**：对话日志、工具调用记录、中间推理过程、长期记忆、用户反馈
- **数据交织深**：用户数据与 Agent 推理深度纠缠，难以分离
- **不可预测性**：Agent 自主行为会产生意料之外的数据记录
- **合规叠加**：不同地区、不同数据类型可能适用不同法规

## 背景

Agent 系统的数据治理压力来自三个方向的汇合：

**第一，数据爆炸。** 一个 Agent 会话可能产生数千次 LLM 调用，每次调用都包含完整的提示词、生成结果和工具调用记录。单个 Agent 一天可以产生 GB 级别的日志数据。如果部署了数千个 Agent，数据量级将远超传统系统的处理能力。

**第二，监管压力。** GDPR 的实施开创了数据治理的严格监管时代，随后 CCPA、PIPL（中国个人信息保护法）、LGPD（巴西）等法规跟进。EU AI Act 明确要求高风险 AI 系统建立数据治理框架。不合规的罚款可达全球年营收的 4% 或 2000 万欧元（取较高者）。

**第三，自动化治理成为必需。** 当 Agent 系统达到一定规模后，手动管理数据存储、保留和删除变得完全不可行。治理必须是自动化的、策略驱动的、可审计的。

```
数据治理的演进
─────────────────────────────────────────────
无治理 →    被动治理 →      主动治理 →        策略驱动治理
(不管理数据)  (出问题才处理)   (定期清理)        (自动分类+自动策略执行)
    │            │               │                    │
    ▼            ▼               ▼                    ▼
  风险极高     合规滞后       成本可控           安全+合规+效率
                      Agent 系统出现 →
                      数据量爆炸 →
                      合规压力倍增 →
```

## 核心矛盾

**"保留数据以改进 Agent vs. 删除数据以保护隐私"**

这是 Agent 数据治理中最根本的张力：

| 要保留数据的理由 | 要删除数据的理由 |
|---|---|
| 调试和故障排查需要完整日志 | 存储的用户数据越少，隐私风险越低 |
| 模型微调需要真实对话数据 | 合规法规要求数据最小化 |
| 行为审计需要历史记录 | 数据泄露的损失随数据量线性增长 |
| 改进 Agent 需要分析失败案例 | 用户有权要求删除其数据 |
| 安全事件调查需要追溯痕迹 | 保留数据意味着保留责任 |

这个矛盾没有完美的解决方案，只有工程化的权衡。核心原则是：

```
数据治理的根本妥协
─────────────────────────────────────────────
保留：尽可能少   ← →   删除：尽可能彻底
     ↑                  ↑
  满足业务需求        满足隐私合规
     ↓                  ↓
     ┌─────────────────────┐
     │ 解决方案：分层治理      │
     │ • 结构化数据 → 长期保留  │
     │ • 原始对话 → 短期保留    │
     │ • PII 数据 → 即时脱敏    │
     │ • 聚合统计 → 无限期保留  │
     └─────────────────────┘
```

## 详细内容

### 1. Data Classification — 数据分类

数据分类是数据治理的基石——不知道有什么数据，就无法管理数据。Agent 系统中的数据可以按来源和敏感度两个维度分类。

#### 按数据来源分类

```
Agent 数据分类体系
─────────────────────────────────────────────

┌─────────────────────────────────────────────────────────┐
│  用户数据 (User Data)                                    │
│  • 用户身份信息：姓名、邮箱、用户 ID                       │
│  • 用户提供的上下文：文件、代码、业务数据                   │
│  • 用户偏好设置：语言、时区、个性化配置                     │
│  敏感度：★★★★★                                           │
├─────────────────────────────────────────────────────────┤
│  Agent 对话日志 (Conversation Logs)                       │
│  • 完整的提示词-回复对                                    │
│  • Agent 的推理链（Thought 轨迹）                        │
│  • 用户反馈（赞/踩、修正）                                │
│  敏感度：★★★★☆                                           │
├─────────────────────────────────────────────────────────┤
│  工具调用数据 (Tool Call Data)                             │
│  • 调用的工具名称和参数                                   │
│  • 工具返回的原始结果                                     │
│  • API 密钥、访问令牌（可能泄露在参数中）                  │
│  敏感度：★★★☆☆~★★★★★（取决于调用内容）                    │
├─────────────────────────────────────────────────────────┤
│  系统日志 (System Logs)                                   │
│  • 错误日志、性能指标                                     │
│  • 模型调用耗时、Token 用量                               │
│  • 系统配置变更记录                                       │
│  敏感度：★★☆☆☆                                           │
├─────────────────────────────────────────────────────────┤
│  Agent 记忆 (Agent Memory)                                │
│  • 长期记忆（跨会话持久化的事实）                          │
│  • 工作记忆（当前会话的上下文）                            │
│  • 用户画像（Agent 对用户的建模）                          │
│  敏感度：★★★★☆                                           │
├─────────────────────────────────────────────────────────┤
│  训练/微调数据 (Training Data)                             │
│  • 用于 RLHF 的偏好数据                                   │
│  • 用于微调的人工标注数据                                 │
│  • Agent 自我对弈生成的合成数据                           │
│  敏感度：★★★☆☆~★★★★★                                   │
└─────────────────────────────────────────────────────────┘
```

#### 按敏感度分级

| 级别 | 定义 | Agent 数据示例 | 处理要求 |
|------|------|---------------|---------|
| L0: 公开 | 无敏感内容 | 公开知识库、模型权重 | 无特殊处理 |
| L1: 内部 | 内部使用但非敏感 | 聚合统计、匿名指标 | 访问控制 |
| L2: 敏感 | 泄露有中等影响 | 对话日志（已脱敏）、工具调用 | 加密 + 访问审计 |
| L3: 机密 | 泄露有严重影响 | 含 PII 的对话、用户身份 | 强加密 + 最小访问 + 自动脱敏 |
| L4: 受限 | 法律保护的数据 | PHI、金融凭证、儿童数据 | 特殊合规 + 严格隔离 + 删除优先 |

### 2. Data Lifecycle Management — 数据生命周期管理

每个数据条目从创建到删除经历完整的生命周期。在每个阶段都需要不同的治理策略。

```
Agent 数据生命周期
─────────────────────────────────────────────

  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  创建     │────►│  存储     │────►│  使用     │
  │ Creation  │     │ Storage  │     │ Usage    │
  └──────────┘     └──────────┘     └──────────┘
       │                               │
       │                               │
       ▼                               ▼
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  删除     │◄────│  归档     │◄────│  检索     │
  │ Deletion │     │ Archival │     │ Retrieval│
  └──────────┘     └──────────┘     └──────────┘
```

#### 各阶段的治理要点

**① 创建阶段 (Creation)**
- **数据最小化**：只收集必要的数据，不要因为"以后可能有用"而收集
- **目的标注**：每条数据在创建时就标注用途（调试/审计/训练/改进）
- **默认脱敏**：尽可能在源头脱敏，而不是事后清理
- **知情同意**：涉及用户数据时，确保创建前已获得适当同意

```python
# 数据最小化的实践：只在必要时记录敏感信息
class AgentLogger:
    def log_conversation(self, conversation, purpose="debug"):
        # 调试目的：记录完整对话，但短期保留
        if purpose == "debug":
            self._write_to_fast_storage(conversation, ttl="7d")

        # 训练目的：只记录脱敏后的结构化数据
        if purpose == "training":
            sanitized = self._redact_pii(conversation)
            features = self._extract_training_features(sanitized)
            self._write_to_training_store(features)

        # 审计目的：只记录元数据，不记录内容
        if purpose == "audit":
            metadata = self._extract_metadata(conversation)
            self._write_to_audit_log(metadata)
```

**② 存储阶段 (Storage)**
- **分级存储**：热数据（SSD，快速访问）、温数据（对象存储）、冷数据（压缩归档）
- **加密要求**：传输加密（TLS）+ 存储加密（AES-256）
- **隔离存储**：不同敏感级别的数据存储在不同的存储区域
- **数据发现**：维护数据目录（Data Catalog），知道什么数据存在哪里

| 存储类型 | 适用数据 | 存储介质 | 访问速度 | 成本 |
|---------|---------|---------|---------|------|
| 热存储 | 当前会话、工作记忆 | Redis / Memcached | <1ms | 高 |
| 主存储 | 对话日志、系统日志 | PostgreSQL / ClickHouse | 10-100ms | 中 |
| 对象存储 | 归档日志、训练数据 | S3 / GCS / MinIO | 100ms-1s | 低 |
| 冷存储 | 合规保留的长期归档 | Glacier / 磁带 | 分钟-小时 | 极低 |

**③ 使用阶段 (Usage)**
- **目的限制**：仅为标注的目的使用数据（收集时不说是用于训练，就不能用于训练）
- **最小访问**：只提供完成任务所需的最少数据
- **使用审计**：记录谁在什么时间访问了什么数据
- **用途隔离**：生产数据和训练数据严格隔离

**④ 检索阶段 (Retrieval)**
- **查询审计**：所有数据查询操作都记录
- **批量限制**：单次查询的数据量上限
- **脱敏展示**：即使存储时是完整数据，展示时也需脱敏
- **结果缓存**：避免反复检索相同数据

**⑤ 归档阶段 (Archival)**
- **自动归档**：基于策略自动将冷数据转移到归档存储
- **压缩加密**：归档前压缩 + 加密
- **保留元数据**：归档后仍可通过元数据搜索
- **恢复测试**：定期测试归档数据的可恢复性

**⑥ 删除阶段 (Deletion)**
- **安全删除**：覆写或加密密钥销毁（不仅仅是标记删除）
- **级联删除**：关联数据一起删除
- **删除确认**：验证数据确实不可恢复
- **删除日志**：记录删除操作（但不记录已删除的内容）

### 3. Data Retention Policies — 数据保留策略

保留策略定义了不同类型的数据应该保存多久。政策由法规要求、业务需求和风险承受能力共同决定。

#### 保留期限矩阵

| 数据类型 | 默认保留期 | 法律依据 | 归档后保留 | 备注 |
|---------|-----------|---------|-----------|------|
| 用户身份信息 | 账户存续期间 + 90 天 | GDPR Art.5 | 不归档，到期删除 | 用户可随时请求删除 |
| 对话日志（含 PII） | 30 天 | GDPR 数据最小化 | 脱敏后归档 1 年 | 长期需要必须脱敏 |
| 对话日志（已脱敏） | 1 年 | SOC 2 审计要求 | 压缩归档 3 年 | 用于改进 Agent |
| 工具调用记录 | 90 天 | 安全审计要求 | 归档 2 年 | 含执行耗时和结果 |
| Agent 记忆 | 用户请求时删除 | GDPR 删除权 | 不归档 | 用户可导出后删除 |
| 系统错误日志 | 90 天 | 运维需要 | 归档 1 年 | 不含用户数据 |
| 训练数据 | 项目生命周期内 | 业务需要 | 匿名化后无限期 | 用于模型改进 |
| 审计日志 | 3-7 年 | 法规要求（各法域不同） | 防篡改存储 | 不可删除（在保留期内） |

#### 保留策略的实现

保留策略的核心机制是 **TTL (Time To Live)** 和 **自动过期处理**。

```yaml
# retention-policies.yaml — 策略即代码
retention_policies:
  conversation_logs:
    raw_with_pii:
      ttl: 30d
      action: redact_pii_then_archive
      archive_ttl: 365d
      final_action: secure_delete
    anonymized:
      ttl: 365d
      action: compress_archive
      archive_ttl: 3y
      final_action: secure_delete

  agent_memory:
    working_memory:
      ttl: 24h  # 会话级，会话结束即过期
      action: delete_on_session_end
    long_term_memory:
      ttl: null  # 由用户控制
      action: user_delete_request

  audit_logs:
    immutable:
      ttl: 7y  # 合规要求
      action: seal_and_store
      final_action: legal_review_before_delete
```

#### 合法保留 (Legal Hold)

当发生诉讼或监管调查时，即使数据已到保留期限，也必须暂时冻结：

- **触发条件**：法律通知、监管调查、诉讼预期
- **冻结范围**：指定用户/会话/时间段的数据
- **覆盖策略**：合法保留覆盖自动删除策略
- **解除条件**：法律程序结束，正式通知解除保留

```python
class LegalHoldManager:
    """合法保留管理器——防止在诉讼期间自动删除相关数据"""

    def __init__(self):
        self.active_holds: dict[str, LegalHold] = {}

    def place_hold(self, hold: LegalHold):
        """对特定用户或数据范围施加合法保留"""
        self.active_holds[hold.case_id] = hold
        logger.info(f"Legal hold placed: case={hold.case_id}, scope={hold.scope}")

    def should_preserve(self, data_entry: DataEntry) -> bool:
        """检查数据条目是否受合法保留保护"""
        for hold in self.active_holds.values():
            if hold.covers(data_entry):
                return True
        return False

    def release_hold(self, case_id: str):
        """法律程序结束后解除保留"""
        hold = self.active_holds.pop(case_id, None)
        if hold:
            logger.info(f"Legal hold released: case={case_id}")
            # 释放后立即处理积压的过期数据
            self._process_released_data(case_id)
```

### 4. Data Deletion — 数据删除

删除是数据治理中最容易被轻视的环节。"删除"在技术上远比听起来复杂。

#### 删除的层次

```
删除的深度层次
─────────────────────────────────────────────

软删除 (Soft Delete)         标记为已删除，数据仍在存储中
    │                          恢复容易，但合规性差
    ▼
逻辑删除 (Logical Delete)    从应用层不可见，但可能在备份中
    │                          通常足以满足多数合规要求
    ▼
安全删除 (Secure Delete)     覆写原始数据（DOD 5220.22-M 标准）
    │                          从存储介质上彻底抹除
    ▼
密码学删除 (Crypto Delete)   加密数据 + 销毁加密密钥
    │                          立即生效，成本极低
    ▼
物理销毁 (Physical Destroy)  销毁存储介质（硬盘粉碎/消磁）
    │                          最彻底，但成本最高
```

对于 Agent 系统，**推荐组合策略**：
- 常规删除用 **逻辑删除 + 自动过期**
- 敏感数据用 **安全删除**
- 大规模归档数据用 **密码学删除**（加密存储，到期销毁密钥）
- 最高级别机密用 **物理销毁**

#### 级联删除 (Cascade Deletion)

Agent 数据高度关联，删除一个用户的数据可能需要级联删除多个存储中的数据：

```
用户请求删除数据
        │
        ▼
┌─────────────────┐
│  主数据删除       │ ← 用户配置、账户信息
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│  对话数据删除     │────►│  关联工具调用     │
│ (所有会话)       │     │ (对应 call_id)   │
└────────┬────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│  Agent 记忆删除   │────►│  记忆嵌入向量     │
│ (长期记忆)       │     │ (向量数据库)     │
└────────┬────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│  训练数据排除     │────►│  反馈/标注数据    │
│ (排除出训练集)   │     │ (用户评价)       │
└────────┬────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐
│  删除验证         │ ← 确认所有副本已删除
└─────────────────┘
```

#### 删除验证 (Verification of Deletion)

仅仅执行删除操作是不够的——必须验证删除确实生效：

```python
class DeletionVerifier:
    """删除验证器——确认数据确实被删除"""

    async def verify_deletion(self, request: DeletionRequest) -> DeletionReport:
        report = DeletionReport(request_id=request.id)

        # 1. 检查主存储
        main_store_result = await self._check_store(
            self.main_storage, request.data_ids
        )
        report.add_result("main_storage", main_store_result)

        # 2. 检查备份存储
        for backup in self.backup_stores:
            backup_result = await self._check_store(backup, request.data_ids)
            report.add_result(f"backup:{backup.name}", backup_result)

        # 3. 检查缓存
        cache_result = await self._check_cache(request.data_ids)
        report.add_result("cache", cache_result)

        # 4. 检查数据仓库/分析管道
        warehouse_result = await self._check_warehouse(request.data_ids)
        report.add_result("warehouse", warehouse_result)

        # 5. 检查向量数据库
        vector_result = await self._check_vector_db(request.data_ids)
        report.add_result("vector_db", vector_result)

        # 6. 检查日志系统
        log_result = await self._check_log_system(request.data_ids)
        report.add_result("logs", log_result)

        report.all_verified = all(
            r.status == DeletionStatus.CONFIRMED
            for r in report.results.values()
        )

        return report
```

### 5. PII Detection & Redaction — 个人身份信息检测与脱敏

PII（Personally Identifiable Information）管理是数据治理的核心合规要求。Agent 系统因为处理大量自然语言对话，PII 检测尤其重要。

#### 常见的 PII 类型

| PII 类别 | 示例 | 检测难度 | 在 Agent 对话中出现频率 |
|---------|------|---------|----------------------|
| 姓名 | 张三, John Smith | ★☆☆☆☆ | 非常高 |
| 邮箱 | user@example.com | ★☆☆☆☆ | 高 |
| 电话号码 | 138-0000-0000 | ★☆☆☆☆ | 高 |
| 地址 | 北京市海淀区... | ★★★☆☆ | 中 |
| 身份证号 | 110101199001011234 | ★★☆☆☆ | 低 |
| 信用卡号 | 4111-1111-1111-1111 | ★★☆☆☆ | 低 |
| IP 地址 | 192.168.1.1 | ★★☆☆☆ | 中 |
| 医疗信息 | 诊断结果、病历 | ★★★★★ | 低（但风险极高） |
| 财务信息 | 薪资、账户余额 | ★★★★☆ | 低 |
| 生物特征 | 指纹、面部识别数据 | ★★★★★ | 极低 |

#### PII 检测策略

```python
class PIIDetector:
    """多层 PII 检测器——组合多种检测方法"""

    def __init__(self):
        # 1. 正则匹配——结构化 PII
        self.regex_detectors = {
            "email": re.compile(r"[\w.+-]+@[\w-]+\.[\w.]+"),
            "phone_cn": re.compile(r"1[3-9]\d{9}"),
            "id_card_cn": re.compile(r"\d{17}[\dXx]"),
            "credit_card": re.compile(r"\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}"),
            "ip_address": re.compile(r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"),
        }

        # 2. NER 模型——非结构化 PII
        self.ner_model = self._load_ner_model()

        # 3. 上下文检测——复杂 PII（如"我的生日是1990年1月1日"）
        self.context_detector = ContextAwareDetector()

        # 4. 校验和验证——防止假阳性
        self.validators = {
            "id_card_cn": self._validate_id_card_checksum,
            "credit_card": self._validate_luhn,
        }

    def detect(self, text: str) -> list[PIIDetection]:
        findings = []

        # 第一层：正则快速扫描
        for pii_type, pattern in self.regex_detectors.items():
            for match in pattern.finditer(text):
                detection = PIIDetection(
                    type=pii_type,
                    value=match.group(),
                    start=match.start(),
                    end=match.end(),
                    confidence=0.9,
                )
                # 校验和验证
                if pii_type in self.validators:
                    if self.validators[pii_type](match.group()):
                        detection.confidence = 0.99
                    else:
                        detection.confidence = 0.3  # 可能是假阳性
                findings.append(detection)

        # 第二层：NER 模型检测
        ner_findings = self.ner_model.extract(text)
        findings.extend(ner_findings)

        # 第三层：上下文检测
        ctx_findings = self.context_detector.detect(text)
        findings.extend(ctx_findings)

        # 去重和合并
        return self._deduplicate_and_merge(findings)
```

#### 脱敏策略

| 策略 | 方法 | 示例 | 适用场景 |
|------|------|------|---------|
| **掩码 (Masking)** | 只显示部分字符 | `138****0000` | 客服查看、调试 |
| **替换 (Replacement)** | 用占位符替换 | `[EMAIL_REDACTED]` | 训练数据、日志 |
| **泛化 (Generalization)** | 降低精度 | `北京市海淀区` → `北京市` | 分析报表 |
| **伪匿名化 (Pseudonymization)** | 用假名替换 | `张三` → `用户A` | 可逆脱敏 |
| **匿名化 (Anonymization)** | 不可逆去除 | 删除所有个人标识 | 研究数据、公开数据 |
| **差分隐私 (Differential Privacy)** | 添加噪声 | 统计结果加噪 | 聚合分析 |

#### 伪匿名化 vs 匿名化

```
伪匿名化 (Pseudonymization)          匿名化 (Anonymization)
─────────────────────────           ─────────────────────────
可逆                                  不可逆
│                                     │
▼                                     ▼
原始数据 → 伪标识符 → 可通过映射表还原   原始数据 → 匿名数据 → 无法还原
│                                     │
适用：分析、开发测试                    适用：研究、公开数据
GDPR Art.4(5)                          GDPR Recital 26
仍属于个人数据                          不再属于个人数据
```

**关键区分**：伪匿名化的数据从法律角度来看仍属于个人数据（因为可以还原），而匿名化数据则不再受数据保护法规管辖。在 Agent 系统中，通常的做法是：
- 内部调试用：伪匿名化（方便必要时溯源）
- 训练数据用：匿名化（降低合规风险）
- 公开数据集：匿名化 + 差分隐私

### 6. Data Access Control — 数据访问控制

知道数据存在、知道数据该存多久还不够——还必须控制谁能看到这些数据。

#### Agent 数据的访问主体

```
谁在访问 Agent 数据？
─────────────────────────────────────────────

访问者               权限范围                   典型场景
─────               ────────                   ────────
用户本人             自己的所有数据              查看历史对话
Agent 开发者         系统日志、脱敏后的聚合数据    调试和改进 Agent
合规审计员           审计日志、访问记录           合规审查
法务部门             合法保留范围内的数据          诉讼准备
数据保护官 (DPO)     所有数据（审计用）           隐私影响评估
监管机构             指定范围内的数据              合规检查
第三方审计           审计范围内的数据              SOC 2 审计
恶意攻击者           不应该有任何权限              （需要阻止）
```

#### 访问控制模型

```
Agent 数据访问控制
─────────────────────────────────────────────

┌──────────────────────────────────────────────┐
│              身份认证层                         │
│  • OAuth 2.0 / OIDC 用户认证                   │
│  • API Key / mTLS 服务间认证                   │
│  • 会话 Token + 刷新机制                       │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│              授权策略层                         │
│  ┌──────────────────────────────────────────┐ │
│  │ 基于属性的访问控制 (ABAC)                  │ │
│  │ 策略示例：                                 │ │
│  │  • 用户只能访问自己的对话数据                │ │
│  │  • 开发者只能在调试模式下访问脱敏数据          │ │
│  │  • 审计员只能读取审计日志（不可修改/删除）     │ │
│  │  • 训练管道只能访问已标记"训练用途"的数据      │ │
│  └──────────────────────────────────────────┘ │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│              数据脱敏层                         │
│  • 动态脱敏：根据访问者权限级别动态决定脱敏程度    │
│  • 行级安全：数据库层的强制访问控制               │
│  • 列级加密：敏感字段加密存储，按需解密            │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│              审计日志层                         │
│  • 记录每次数据访问：谁 + 什么 + 何时 + 为什么   │
│  • 异常访问实时告警                              │
│  • 定期访问权限审查                              │
└──────────────────────────────────────────────┘
```

#### 三种访问控制模型对比

| 模型 | 原理 | Agent 数据治理适用性 | 复杂度 |
|------|------|---------------------|--------|
| **RBAC** (基于角色) | 角色 → 权限 | 适合粗粒度控制：管理员/开发者/审计员 | 低 |
| **ABAC** (基于属性) | 用户属性 + 资源属性 + 环境 → 策略 | 适合细粒度控制：按用户、数据类型、用途、时间 | 高 |
| **ReBAC** (基于关系) | 用户与资源的关系 → 权限 | 适合社交/协作场景 | 中 |

推荐方案：**ABAC + RBAC 混合**——角色提供基础框架（粗粒度），属性策略进行精细调整（细粒度）。

```python
class DataAccessController:
    """数据访问控制器——基于 ABAC 的 Agent 数据访问管理"""

    async def check_access(
        self,
        user: User,
        data: DataEntry,
        action: str,
        context: AccessContext
    ) -> AccessDecision:
        # 构建属性集合
        attributes = {
            "user": {
                "id": user.id,
                "role": user.role,
                "department": user.department,
                "clearance": user.data_clearance_level,
            },
            "resource": {
                "type": data.type,
                "sensitivity": data.sensitivity_level,
                "owner": data.owner_id,
                "purpose_tag": data.purpose_tag,
            },
            "action": action,  # read / write / delete / export
            "environment": {
                "ip": context.ip_address,
                "time": context.access_time,
                "auth_method": context.auth_method,
            }
        }

        # 评估策略
        decision = await self.policy_engine.evaluate(attributes)

        # 记录访问
        await self.audit_logger.log_access(
            user_id=user.id,
            data_id=data.id,
            action=action,
            decision=decision,
            context=context
        )

        # 异常检测
        if decision == Decision.DENY:
            await self.anomaly_detector.record_denied_access(user, data, context)

        return decision
```

#### 目的限制 (Purpose Limitation)

Agent 数据的特殊之处在于——同一份数据可能被不同目的使用，而目的决定了访问权限：

| 数据用途 | 允许的访问者 | 脱敏要求 | 访问期限 |
|---------|------------|---------|---------|
| 实时对话 | 仅用户 + Agent | 无（用户自己的数据） | 会话期间 |
| 调试分析 | 开发团队 | 必须脱敏 | 项目期间 |
| 模型训练 | 训练管道 | 必须匿名化 | 训练期间 |
| 合规审计 | 审计员 | 不脱敏（需要完整数据） | 审计期间 |
| 法律诉讼 | 法务团队 | 不脱敏 | 合法保留期间 |

### 7. Agent Memory Governance — Agent 记忆治理

记忆是 Agent 区别于传统系统的核心能力，也是数据治理最特殊的领域。Agent 记忆不是简单的日志，而是经过处理的、结构化的、可被 Agent 主动检索和使用的信息。

#### 记忆类型及其治理要求

```
Agent 记忆体系
─────────────────────────────────────────────

短期记忆 (Working Memory)
  • 当前会话的上下文
  • 自然在会话结束后消失
  • 治理重点：确保会话结束即释放

─────────────────────────────────────────────

长期记忆 (Long-term Memory)
  • 跨会话持久化的事实和关系
  • 用户明确要求 Agent 记住的信息
  • 治理重点：用户控制权 + 可删除性

─────────────────────────────────────────────

隐式记忆 (Implicit Memory)
  • Agent 从对话中自动提取的用户画像
  • 用户偏好、行为模式、习惯
  • 治理重点：透明度 + 用户确认

─────────────────────────────────────────────

程序记忆 (Procedural Memory)
  • Agent 学会的工作流程和技能
  • 可能隐含用户特定行为
  • 治理重点：不泄露个体用户信息
```

#### 记忆治理的核心问题

```
Agent 记忆的两面性
─────────────────────────────────────────────

没有记忆的 Agent                 有记忆的 Agent
───────────────                 ───────────────
每个会话从零开始                  跨会话的连续性
每次都要重新介绍上下文            个性化体验
无法从过往交互中学习              持续改进

但：隐私安全                    但：数据治理风险
    体验差                          存储用户画像
    效率低                          难以彻底遗忘
```

#### 用户对记忆的控制权

用户应该对 Agent 的记忆拥有以下控制权：

```python
class MemoryGovernor:
    """Agent 记忆治理——用户对记忆的完全控制"""

    async def get_memories(self, user_id: str) -> list[Memory]:
        """用户查看 Agent 记住了什么"""
        return await self.memory_store.query(user_id=user_id)

    async def delete_memory(self, memory_id: str, user_id: str) -> bool:
        """用户删除单条记忆"""
        memory = await self.memory_store.get(memory_id)
        if memory.owner_id != user_id:
            raise PermissionError("无权删除他人的记忆")

        # 级联删除：删除记忆条目 + 对应的嵌入向量
        await self.memory_store.delete(memory_id)
        await self.vector_store.delete_embedding(memory.embedding_id)
        await self.prompt_cache.invalidate(user_id)  # 提示词缓存失效

        logger.info(f"Memory deleted: memory_id={memory_id}, user_id={user_id}")
        return True

    async def clear_all_memories(self, user_id: str) -> int:
        """用户清除所有记忆"""
        count = await self.memory_store.delete_by_user(user_id)
        await self.vector_store.delete_by_user(user_id)
        await self.prompt_cache.invalidate(user_id)

        logger.info(f"All memories cleared: user_id={user_id}, count={count}")
        return count

    async def export_memories(self, user_id: str) -> MemoryExport:
        """用户导出记忆（数据可移植性）"""
        memories = await self.memory_store.query(user_id=user_id)
        return MemoryExport(
            user_id=user_id,
            export_time=datetime.utcnow(),
            format="json",
            memories=[m.to_dict() for m in memories],
        )

    async def disable_memory(self, user_id: str) -> None:
        """用户选择不让 Agent 记住任何事情"""
        await self.memory_store.disable_for_user(user_id)
        await self.clear_all_memories(user_id)
        logger.info(f"Memory disabled for user: {user_id}")
```

#### 记忆自动遗忘机制

好的 Agent 记忆系统应该像人脑一样：该记住的记住，该忘记的自动忘记。

```python
class AutoForgetMechanism:
    """自动遗忘机制——时间衰减 + 重要性评估"""

    def __init__(self):
        self.decay_rates = {
            "fact": 0.05,      # 事实性记忆慢衰减
            "preference": 0.1, # 偏好适度衰减
            "context": 0.3,    # 上下文快速衰减
            "ephemeral": 0.5,  # 临时信息极快衰减
        }

    async def decay_memories(self, user_id: str):
        """运行记忆衰减——在每次会话开始时触发"""
        memories = await self.memory_store.query(user_id=user_id)

        for memory in memories:
            age_days = (datetime.utcnow() - memory.created_at).days
            access_days = (datetime.utcnow() - memory.last_accessed_at).days

            # 衰减公式：基于年龄 + 未访问时间 + 记忆类型
            decay_score = (
                age_days * self.decay_rates[memory.memory_type] +
                access_days * self.decay_rates[memory.memory_type] * 2
            )

            # 重要性提升——高频访问或显式保存的延缓衰减
            importance_boost = memory.access_count * 0.01 + \
                (10 if memory.explicitly_saved else 0)

            effective_decay = max(0, decay_score - importance_boost)

            # 衰减阈值触发删除
            if effective_decay > 100:
                await self.memory_store.delete(memory.id)
                await self.vector_store.delete_embedding(memory.embedding_id)
                logger.debug(f"Memory auto-forgotten: {memory.id}")
```

## Example Code

下面的 `DataGovernor` 类是一个完整的数据治理控制器，集成了分类、生命周期管理和 PII 脱敏：

```python
"""
DataGovernor — Agent 系统数据治理控制器

集成了数据分类、生命周期管理和 PII 脱敏三大核心功能，
提供统一的治理接口。
"""

import abc
import json
import logging
import re
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum, auto
from typing import Optional

logger = logging.getLogger(__name__)


# ═══════════════════════════════════════════════
#  数据模型
# ═══════════════════════════════════════════════

class DataCategory(Enum):
    USER_DATA = auto()
    CONVERSATION_LOG = auto()
    TOOL_CALL_DATA = auto()
    SYSTEM_LOG = auto()
    AGENT_MEMORY = auto()
    TRAINING_DATA = auto()


class SensitivityLevel(Enum):
    PUBLIC = 0       # L0: 公开
    INTERNAL = 1     # L1: 内部
    SENSITIVE = 2    # L2: 敏感
    CONFIDENTIAL = 3 # L3: 机密
    RESTRICTED = 4   # L4: 受限


@dataclass
class RetentionPolicy:
    """数据保留策略"""
    ttl: timedelta                    # 主存储保留期
    action_on_expiry: str             # 过期动作: archive / redact / delete
    archive_ttl: Optional[timedelta]  # 归档保留期 (None 表示不归档)
    legal_hold_override: bool = False # 是否可被合法保留覆盖


@dataclass
class PIIDetection:
    """PII 检测结果"""
    type: str
    value: str
    start: int
    end: int
    confidence: float
    suggested_redaction: str = ""


@dataclass
class DataEntry:
    """数据条目——治理的最小单元"""
    id: str
    category: DataCategory
    sensitivity: SensitivityLevel
    owner_id: str
    content: str
    created_at: datetime
    purpose_tag: str  # debug / audit / training / operation
    metadata: dict = field(default_factory=dict)
    retention_policy: Optional[RetentionPolicy] = None


# ═══════════════════════════════════════════════
#  接口定义
# ═══════════════════════════════════════════════

class Classifier(abc.ABC):
    """数据分类器接口"""
    @abc.abstractmethod
    async def classify(self, data: DataEntry) -> None: ...


class LifecycleManager(abc.ABC):
    """生命周期管理器接口"""
    @abc.abstractmethod
    async def apply_retention_policy(self, data: DataEntry) -> RetentionPolicy: ...
    @abc.abstractmethod
    async def process_expired_data(self) -> int: ...


class PIIRedactor(abc.ABC):
    """PII 脱敏器接口"""
    @abc.abstractmethod
    async def detect(self, content: str) -> list[PIIDetection]: ...
    @abc.abstractmethod
    async def redact(self, content: str, purpose: str) -> str: ...


# ═══════════════════════════════════════════════
#  实现
# ═══════════════════════════════════════════════

class RegexAndNERClassifier(Classifier):
    """基于规则和 NER 的数据分类器"""

    SENSITIVITY_RULES = {
        SensitivityLevel.CONFIDENTIAL: [
            re.compile(r"\d{17}[\dXx]"),          # 身份证号
            re.compile(r"\d{4}-\d{4}-\d{4}-\d{4}"), # 信用卡号
        ],
        SensitivityLevel.SENSITIVE: [
            re.compile(r"1[3-9]\d{9}"),            # 手机号
            re.compile(r"[\w.+-]+@[\w-]+\.[\w.]+"), # 邮箱
        ],
    }

    CATEGORY_KEYWORDS = {
        DataCategory.CONVERSATION_LOG: ["user said", "assistant:", "thought:"],
        DataCategory.TOOL_CALL_DATA: ["tool_call", "function:", "execute"],
        DataCategory.SYSTEM_LOG: ["error", "warn", "trace", "latency"],
    }

    async def classify(self, data: DataEntry) -> None:
        """自动分类数据条目的类别和敏感度"""
        # 敏感度分类
        detected_level = SensitivityLevel.PUBLIC
        for level, patterns in self.SENSITIVITY_RULES.items():
            for pattern in patterns:
                if pattern.search(data.content):
                    detected_level = level
                    break
            if detected_level != SensitivityLevel.PUBLIC:
                break

        # 类别自动识别（如果尚未设置）
        if data.category is None:
            for category, keywords in self.CATEGORY_KEYWORDS.items():
                if any(kw in data.content.lower() for kw in keywords):
                    data.category = category
                    break

        # 强化分类：如果内容含 PII，自动提升敏感度
        pii_count = sum(
            1 for level, patterns in self.SENSITIVITY_RULES.items()
            for p in patterns if p.search(data.content)
        )
        if pii_count >= 2 and detected_level.value < SensitivityLevel.CONFIDENTIAL.value:
            detected_level = SensitivityLevel.CONFIDENTIAL
        elif pii_count >= 1 and detected_level.value < SensitivityLevel.SENSITIVE.value:
            detected_level = SensitivityLevel.SENSITIVE

        data.sensitivity = detected_level
        data.metadata["classification_method"] = "auto_regex_ner"
        data.metadata["classified_at"] = datetime.utcnow().isoformat()


class PolicyBasedLifecycleManager(LifecycleManager):
    """基于策略的生命周期管理器——从政策文件加载规则"""

    # 默认保留策略
    DEFAULT_POLICIES = {
        DataCategory.USER_DATA: RetentionPolicy(
            ttl=timedelta(days=90),
            action_on_expiry="secure_delete",
            archive_ttl=None,
        ),
        DataCategory.CONVERSATION_LOG: RetentionPolicy(
            ttl=timedelta(days=30),
            action_on_expiry="redact_then_archive",
            archive_ttl=timedelta(days=365),
        ),
        DataCategory.TOOL_CALL_DATA: RetentionPolicy(
            ttl=timedelta(days=90),
            action_on_expiry="compress_archive",
            archive_ttl=timedelta(days=730),
        ),
        DataCategory.SYSTEM_LOG: RetentionPolicy(
            ttl=timedelta(days=90),
            action_on_expiry="archive",
            archive_ttl=timedelta(days=365),
        ),
        DataCategory.AGENT_MEMORY: RetentionPolicy(
            ttl=timedelta(days=0),  # 用户控制，无默认保留
            action_on_expiry="delete_on_request",
            archive_ttl=None,
        ),
        DataCategory.TRAINING_DATA: RetentionPolicy(
            ttl=timedelta(days=365),
            action_on_expiry="anonymize_and_keep",
            archive_ttl=None,
        ),
    }

    def __init__(self, redactor: PIIRedactor):
        self.redactor = redactor
        self.policies = dict(self.DEFAULT_POLICIES)

    async def apply_retention_policy(self, data: DataEntry) -> RetentionPolicy:
        """根据数据类别和敏感度确定保留策略并标注到数据条目"""
        policy = self.policies.get(data.category)
        if not policy:
            policy = RetentionPolicy(
                ttl=timedelta(days=30),
                action_on_expiry="delete",
                archive_ttl=None,
            )

        # 敏感数据保留期缩短
        if data.sensitivity == SensitivityLevel.CONFIDENTIAL:
            policy.ttl = min(policy.ttl, timedelta(days=7))
        elif data.sensitivity == SensitivityLevel.RESTRICTED:
            policy.ttl = min(policy.ttl, timedelta(days=1))

        data.retention_policy = policy
        return policy

    async def process_expired_data(self) -> int:
        """扫描并处理所有过期数据，返回处理数量"""
        processed = 0
        expired_entries = await self._find_expired_entries()

        for entry in expired_entries:
            try:
                if entry.retention_policy.action_on_expiry == "secure_delete":
                    await self._secure_delete(entry)
                elif entry.retention_policy.action_on_expiry == "redact_then_archive":
                    redacted_content = await self.redactor.redact(entry.content, "archive")
                    await self._archive(entry, redacted_content)
                elif entry.retention_policy.action_on_expiry == "archive":
                    await self._archive(entry, entry.content)
                elif entry.retention_policy.action_on_expiry == "delete":
                    await self._soft_delete(entry)
                processed += 1
            except Exception as e:
                logger.error(f"Failed to process expired entry {entry.id}: {e}")

        logger.info(f"Processed {processed} expired data entries")
        return processed

    async def _secure_delete(self, entry: DataEntry) -> None:
        """安全删除——覆写数据后再删除"""
        # 实际实现中会覆写存储介质上的数据
        # 这里用日志模拟
        logger.info(f"Secure deleted: {entry.id}")

    async def _archive(self, entry: DataEntry, content: str) -> None:
        """归档——压缩加密后移至冷存储"""
        logger.info(f"Archived: {entry.id} (size={len(content)} bytes)")

    async def _soft_delete(self, entry: DataEntry) -> None:
        """软删除——标记已删除"""
        logger.info(f"Soft deleted: {entry.id}")


class MultiLayerPIIRedactor(PIIRedactor):
    """多层 PII 脱敏器——正则 + NER + 上下文检测"""

    def __init__(self):
        # 第一层：正则检测器
        self.regex_detectors = {
            "email": re.compile(r"[\w.+-]+@[\w-]+\.[\w.]+"),
            "phone_cn": re.compile(r"1[3-9]\d{9}"),
            "id_card_cn": re.compile(r"\d{17}[\dXx]"),
            "credit_card": re.compile(r"\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}"),
            "ip_address": re.compile(r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"),
        }

        # 第二层：上下文检测模式
        self.context_patterns = [
            (re.compile(r"(?:我叫|我是|姓名[是为]?)\s*(\S+)"), "name"),
            (re.compile(r"(?:生日|出生日期)[是为]?(\S+)"), "birth_date"),
            (re.compile(r"(?:住址|地址)[是为]?(\S+)"), "address"),
        ]

    async def detect(self, content: str) -> list[PIIDetection]:
        """检测内容中的 PII"""
        findings = []

        # 正则匹配
        for pii_type, pattern in self.regex_detectors.items():
            for match in pattern.finditer(content):
                findings.append(PIIDetection(
                    type=pii_type,
                    value=match.group(),
                    start=match.start(),
                    end=match.end(),
                    confidence=0.9,
                    suggested_redaction=self._suggest_redaction(pii_type, match.group()),
                ))

        # 上下文匹配
        for pattern, pii_type in self.context_patterns:
            for match in pattern.finditer(content):
                if match.groups():
                    findings.append(PIIDetection(
                        type=pii_type,
                        value=match.group(1),
                        start=match.start(1),
                        end=match.end(1),
                        confidence=0.7,
                        suggested_redaction=f"[{pii_type.upper()}_REDACTED]",
                    ))

        return findings

    async def redact(self, content: str, purpose: str) -> str:
        """根据用途对内容进行脱敏

        不同目的决定了脱敏程度：
        - debug: 掩码 (保留部分信息)
        - training: 完全替换
        - archive: 完全替换
        - audit: 保留完整（审计需要）
        """
        if purpose == "audit":
            return content  # 审计需要完整数据

        detections = await self.detect(content)

        # 按位置从后往前替换（避免偏移问题）
        detections.sort(key=lambda d: d.start, reverse=True)

        redacted = content
        for detection in detections:
            if purpose == "debug":
                replacement = self._mask_value(detection.value)
            else:
                replacement = detection.suggested_redaction or \
                    f"[{detection.type.upper()}_REDACTED]"
            redacted = redacted[:detection.start] + replacement + redacted[detection.end:]

        return redacted

    def _suggest_redaction(self, pii_type: str, value: str) -> str:
        """生成 PII 替换文本"""
        redactions = {
            "email": "[EMAIL_REDACTED]",
            "phone_cn": "[PHONE_REDACTED]",
            "id_card_cn": "[ID_CARD_REDACTED]",
            "credit_card": "[CREDIT_CARD_REDACTED]",
            "ip_address": "[IP_REDACTED]",
        }
        return redactions.get(pii_type, f"[{pii_type.upper()}_REDACTED]")

    def _mask_value(self, value: str) -> str:
        """掩码脱敏——保留首尾，中间替换为 *"""
        if len(value) <= 4:
            return "****"
        return value[:2] + "*" * (len(value) - 4) + value[-2:]


# ═══════════════════════════════════════════════
#  主控制器
# ═══════════════════════════════════════════════

class DataGovernor:
    """数据治理控制器——统一的治理入口"""

    def __init__(
        self,
        classifier: Classifier,
        lifecycle_manager: LifecycleManager,
        redactor: PIIRedactor,
    ):
        self.classifier = classifier
        self.lifecycle = lifecycle_manager
        self.redactor = redactor
        self.legal_holds: dict[str, LegalHold] = {}

    async def ingest(self, data: DataEntry) -> DataEntry:
        """数据摄入——自动执行分类、打标和策略绑定"""
        # 第一步：自动分类
        await self.classifier.classify(data)
        logger.info(f"Classified: {data.id} → category={data.category}, "
                     f"sensitivity={data.sensitivity}")

        # 第二步：绑定保留策略
        await self.lifecycle.apply_retention_policy(data)
        logger.info(f"Retention policy applied: {data.id} → "
                     f"ttl={data.retention_policy.ttl}")

        # 第三步：如果敏感度高，立即脱敏后存储
        if data.sensitivity.value >= SensitivityLevel.CONFIDENTIAL.value:
            redacted_content = await self.redactor.redact(data.content, "store")
            data.metadata["original_content_hash"] = hash(data.content)
            data.content = redacted_content
            data.metadata["redacted_at_ingest"] = True

        return data

    async def handle_deletion_request(self, user_id: str, scope: DeletionScope):
        """处理用户删除请求——级联删除所有关联数据"""
        logger.info(f"Deletion request: user={user_id}, scope={scope}")

        entries = await self._find_user_entries(user_id, scope)

        deleted_count = 0
        for entry in entries:
            # 检查合法保留
            if self._is_under_legal_hold(entry):
                logger.warning(f"Skipping legal hold entry: {entry.id}")
                continue

            await self.lifecycle._secure_delete(entry)
            deleted_count += 1

        # 级联清理：向量数据库、缓存、备份索引
        await self._cascade_cleanup(user_id, scope)

        logger.info(f"Deletion complete: user={user_id}, deleted={deleted_count}")
        return DeletionReport(
            user_id=user_id,
            deleted_count=deleted_count,
            timestamp=datetime.utcnow(),
        )

    async def get_data_inventory(self, user_id: str) -> DataInventory:
        """为用户生成其数据的完整清单（数据可移植性支持）"""
        entries = await self._find_user_entries(user_id, DeletionScope.ALL)
        return DataInventory(
            user_id=user_id,
            total_entries=len(entries),
            by_category={
                category: [e for e in entries if e.category == category]
                for category in DataCategory
            },
            generated_at=datetime.utcnow(),
        )

    def place_legal_hold(self, case_id: str, user_ids: list[str]):
        """对指定用户施加合法保留"""
        self.legal_holds[case_id] = LegalHold(
            case_id=case_id,
            user_ids=user_ids,
            placed_at=datetime.utcnow(),
        )
        logger.info(f"Legal hold placed: case_id={case_id}, users={user_ids}")

    def _is_under_legal_hold(self, entry: DataEntry) -> bool:
        return any(
            entry.owner_id in hold.user_ids
            for hold in self.legal_holds.values()
        )


# ═══════════════════════════════════════════════
#  使用示例
# ═══════════════════════════════════════════════

async def example_usage():
    """DataGovernor 使用示例"""
    # 组装治理控制器
    classifier = RegexAndNERClassifier()
    redactor = MultiLayerPIIRedactor()
    lifecycle = PolicyBasedLifecycleManager(redactor)
    governor = DataGovernor(classifier, lifecycle, redactor)

    # 模拟数据摄入
    conversation_entry = DataEntry(
        id="conv_001",
        category=DataCategory.CONVERSATION_LOG,
        sensitivity=SensitivityLevel.PUBLIC,  # 将被自动提升
        owner_id="user_123",
        content="你好，我叫张三，我的邮箱是 zhangsan@example.com，电话是13800138000",
        created_at=datetime.utcnow(),
        purpose_tag="debug",
    )

    # 摄入过程自动完成分类 + 策略绑定 + 脱敏
    governed = await governor.ingest(conversation_entry)

    print(f"Sensitivity after auto-classify: {governed.sensitivity}")
    # → SensitivityLevel.CONFIDENTIAL (检测到多个 PII)

    print(f"Redacted content: {governed.content}")
    # → "你好，我叫[NAME_REDACTED]，我的邮箱是[EMAIL_REDACTED]，电话是[PHONE_REDACTED]"

    print(f"Retention TTL: {governed.retention_policy.ttl}")
    # → 7 days (因为敏感度提升到 CONFIDENTIAL，保留期缩短)

    # 模拟过期数据处理
    processed = await lifecycle.process_expired_data()
    print(f"Processed expired entries: {processed}")

    # 模拟用户删除请求
    report = await governor.handle_deletion_request(
        user_id="user_123",
        scope=DeletionScope.ALL,
    )
    print(f"Deletion report: {report.deleted_count} entries deleted")
```

## Capability Boundaries

数据治理虽必要但非万能，理解其边界才能合理设计系统：

**1. 治理引入开销**
- 每次数据写入前的分类、打标、脱敏增加延迟（通常 10-100ms）
- PII 检测的计算成本随内容长度线性增长
- 安全删除需要 I/O 操作，大量删除时可能影响系统性能
- 审计日志本身也在产生数据，需要被治理 —— 递归治理问题

**2. 不能替代设计阶段的数据考量**
- 数据治理无法修复根本性的过度收集问题
- 如果系统设计时就收集了不需要的数据，治理只能"管"但不能"收回"
- 数据最小化必须在架构设计阶段决定，事后治理是次优方案

**3. 脱敏不是万能的**
- 匿名化数据可能被重识别（re-identification），尤其是 Agent 对话中的独特模式
- 上下文关联攻击：多个脱敏数据集结合可能还原出 PII
- 差分隐私的隐私预算（ε）需要精心管理，耗尽后无法补充

**4. 删除的局限性**
- 备份中的数据可能数周后才被清理
- 日志聚合系统中可能残留历史统计（如平均值包含已删除用户的数据）
- 模型权重中可能隐含训练数据的信息（需要通过机器遗忘或模型重训练解决）
- 法律要求某些数据不可删除（如合规审计日志）

**5. 跨系统治理困难**
- Agent 系统通常依赖外部服务（向量数据库、对象存储、数据仓库）
- 外部服务的治理能力参差不齐
- 级联删除需要所有下游系统支持

**6. 合规不是一次性的**
- 法规不断变化（EU AI Act 的实施细则仍在制定中）
- 治理策略需要持续更新
- 合规审计是持续过程，不是一次检查

```
数据治理能做到 vs. 做不到
─────────────────────────────────────────────

能做到                                  做不到
──────                                  ──────
✓ 分类和标记所有数据条目                  ✗ 阻止系统设计时的过度收集
✓ 自动执行保留和删除策略                  ✗ 保证匿名化数据不被重识别
✓ 检测和脱敏常见 PII                     ✗ 覆盖所有供应链中的数据副本
✓ 审计数据访问和变更                     ✗ 从训练好的模型中完全擦除数据
✓ 支持用户的删除权和数据可移植性           ✗ 自动化所有合规决策
✓ 级联删除主存储和缓存                    ✗ 保证备份的实时一致性
```

## Comparison

不同数据类型需要不同的治理策略和优先级：

| 维度 | 对话日志 | Agent 记忆 | 训练数据 | 工具调用数据 | 系统日志 |
|------|---------|-----------|---------|------------|---------|
| **治理优先级** | 高（量最大） | 最高（隐私风险） | 中（已脱敏） | 高（可能含凭证） | 低（不含用户数据） |
| **敏感度** | L2-L4 | L2-L3 | L1-L2 | L2-L4 | L0-L1 |
| **保留期限** | 30天-1年 | 用户控制 | 项目生命周期 | 90天-2年 | 90天-1年 |
| **PII 密度** | 高 | 中 | 低（预处理后） | 中 | 低 |
| **删除复杂度** | 中 | 低 | 高（模型中的影响） | 中 | 低 |
| **合规要求** | GDPR Art.5,17 | GDPR Art.17,20 | GDPR Art.5(1)(e) | SOC 2 | SOC 2 |
| **脱敏策略** | 掩码/替换 | 匿名化 | 匿名化 | 替换 | 不需要 |
| **访问控制粒度** | 用户级 | 用户级 | 项目级 | 角色级 | 角色级 |
| **关联性** | 高（与用户/工具调用关联） | 低（独立条目） | 低 | 高（与对话关联） | 低 |
| **备份治理** | 需要 | 不需要 | 需要 | 需要 | 可以 |

**关键发现**：
- **Agent 记忆**需要最严格的治理，因为它是持续存在且可直接用于决策的用户数据
- **对话日志**治理最复杂，因为量大且 PII 密度最高
- **训练数据**虽然在收集时已脱敏，但删除的难度最大（因为数据已融入模型权重）
- **系统日志**优先级最低，但也不能完全忽视（可能意外记录敏感信息）

## Engineering Optimization

### 1. 自动化分类 (Automated Classification)

手动给每条数据打标不可扩展，需要自动化：

```python
class AutomatedClassifierPipeline:
    """自动分类管道——用规则 + 轻量模型实现分类自动化"""

    async def auto_classify_batch(self, entries: list[DataEntry]) -> list[DataEntry]:
        # 批量处理，利用并发提高吞吐
        async with asyncio.TaskGroup() as tg:
            tasks = [tg.create_task(self.classifier.classify(e)) for e in entries]

        # 对低置信度的条目进行人工抽样复核
        uncertain = [e for e in entries
                     if e.metadata.get("classification_confidence", 1.0) < 0.6]
        if uncertain:
            await self.queue_for_human_review(uncertain)

        return entries
```

优化要点：
- **分层检测**：先跑正则（低成本），命中再跑 NER（高成本）
- **缓存结果**：相同内容模式复用检测结果
- **渐进式分类**：先粗分类，空闲时再精细分类
- **反馈闭环**：人工修正的分类结果反哺自动分类模型

### 2. 策略即代码 (Policy-as-Code)

治理策略应该用代码/配置文件管理，而不是人工流程：

```yaml
# governance-policies.yaml
version: "2.0"
policies:
  - match:
      category: conversation_log
      sensitivity: ">= L3"
    actions:
      - on_ingest: redact_pii(mode="strict")
      - on_storage: encrypt(AES-256-GCM)
      - retention: ttl=7d → archive(1y) → secure_delete
      - access: deny_all_except(user=owner, role=auditor)

  - match:
      purpose: training
    actions:
      - on_ingest: verify_consent
      - on_export: anonymize
      - retention: ttl=project_lifetime
```

优势：
- **版本可控**：策略文件可以 Git 管理、Code Review、回滚
- **可测试性**：可以对策略编写单元测试
- **自动化执行**：治理引擎自动加载和执行策略
- **审计友好**：策略变更本身就是审计事件

### 3. 数据访问审计日志 (Audit Logging for Data Access)

每次数据访问都记录审计日志，但记录本身不能太昂贵：

```python
class DataAccessAuditor:
    """数据访问审计——结构化、可查询、低成本"""

    # 批量缓冲写入，减少写入次数
    _buffer: list[AccessLogEntry] = []
    _buffer_size = 100
    _flush_interval = 5  # 秒

    async def log_access(self, entry: AccessLogEntry):
        self._buffer.append(entry)
        if len(self._buffer) >= self._buffer_size:
            await self._flush()

    async def _flush(self):
        entries = self._buffer[:]
        self._buffer.clear()
        # 批量写入 ClickHouse / Elasticsearch
        await self.storage.batch_write(entries)

    async def query_access_log(self, user_id: str, time_range: tuple) -> list[AccessLogEntry]:
        """可查询的审计日志——支持按用户、时间、操作类型检索"""
        return await self.storage.query(
            filter={"user_id": user_id},
            time_range=time_range,
            order_by="timestamp DESC",
        )

    async def detect_anomalous_access(self) -> list[AnomalyAlert]:
        """异常访问检测——基于历史模式"""
        recent_logs = await self.storage.query(
            time_range=(datetime.utcnow() - timedelta(hours=1), datetime.utcnow())
        )

        alerts = []
        for log in recent_logs:
            # 异常：非工作时间访问大量数据
            if self._is_outside_business_hours(log.timestamp) and log.data_count > 100:
                alerts.append(AnomalyAlert(
                    type="off_hours_bulk_access",
                    severity="medium",
                    entry=log,
                ))

            # 异常：从未见过的 IP 访问
            if not await self._is_known_ip(log.user_id, log.ip_address):
                alerts.append(AnomalyAlert(
                    type="unrecognized_ip",
                    severity="high",
                    entry=log,
                ))

        return alerts
```

优化要点：
- **缓冲批量写入**：避免每次访问都写日志带来的性能开销
- **结构化日志**：预定义 schema，支持高效查询和聚合
- **日志数据自身治理**：审计日志也需要保留策略和删除策略
- **异常检测**：自动发现异常访问模式，实时告警

### 4. 完整的治理优化策略总结

| 优化方向 | 方法 | 效果 | 实现复杂度 |
|---------|------|------|-----------|
| **自动分类** | 规则 + 轻量 NER 模型 | 减少 90% 手动打标 | 中 |
| **分层检测** | 先正则后模型，成本递增 | PII 检测成本降低 70% | 中 |
| **策略即代码** | YAML/DSL 管理治理策略 | 策略变更可审计、可测试 | 低 |
| **缓冲写入** | 批量写入审计日志 | 写入性能提升 10x | 低 |
| **密码学删除** | 加密后销毁密钥 | 删除成本降低 99% | 中 |
| **冷热分层** | 热数据 SSD → 冷数据对象存储 | 存储成本降低 80% | 中 |
| **增量分类** | 先粗分类，空闲时精细分类 | 写入延迟降低 50% | 高 |
| **级联跟踪** | 数据关联图维护 | 确保完整的级联删除 | 高 |
| **记忆衰减** | 自动遗忘低频记忆 | 记忆存储量降低 60% | 中 |
| **自适应 TTL** | 根据访问频率动态调整保留期 | 热点数据保留更久，冷数据提前删除 | 高 |

## 关键认知

- **数据治理不是一次项目，而是一个持续的过程**：治理策略需要随着法规变化、系统演进而不断调整
- **数据最小化是最好的治理策略**：不收集就不需要管理，每一条额外收集的数据都是潜在负债
- **自动分类是治理自动化的前提**：没有自动分类，策略即代码就无从谈起
- **删除比存储更难**：真正的删除需要跨系统协调、验证和审计，远比"存起来"复杂
- **记忆治理是 Agent 特有的治理挑战**：传统系统没有"记忆"的概念，这是 Agent 数据治理需要创新的领域
- **合规是最低标准，不是最高目标**：好的数据治理超越合规要求，是对用户隐私的真正尊重
- **治理和安全是正相关关系**：好的数据治理降低数据泄露风险，也降低安全事件的损失面
