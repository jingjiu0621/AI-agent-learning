# 12.2 output-guardrails — 输出护栏

**Agent 的输出不止是"答案"，它可能触发 API 调用、写入数据库、发送邮件、执行命令。** 输出护栏是最后一道防线——在 Agent 的输出到达外部世界之前，拦截一切有害、违规、不安全的输出。

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|----------|------|
| 12.2.1 Content Filtering | 如何过滤敏感/违规/有害内容？ | ★★★☆☆ |
| 12.2.2 Format Validation | 如何验证输出格式正确性？ | ★★★☆☆ |
| 12.2.3 Semantic Guardrails | 如何做语义级别的安全审核？ | ★★★★☆ |
| 12.2.4 Guardrails Libraries | 有哪些现成的护栏库可用？ | ★★★☆☆ |
| 12.2.5 Output Sampling | 如何通过采样参数控制输出安全性？ | ★★★★☆ |
| 12.2.6 Bypass Resistance | 护栏被绕过怎么办？如何评估？ | ★★★★☆ |

## 为什么输出护栏对 Agent 尤为重要

```
传统 LLM 使用                               Agent 使用
─────────────────                          ─────────────────
用户 ← LLM (纯文本输出)                     用户 ← Agent ← LLM → 工具(写文件/发邮件/操作DB)
                                                │
                                          输出护栏至关重要！
                                          因为输出会触发真实操作
```

Agent 的输出不只是"说说而已"——它会调工具、写文件、发请求。一个有害的文本输出在纯聊天场景只是一个冒犯，但在 Agent 场景可能意味着：删除用户数据、向客户发送侮辱性邮件、执行未经授权的转账。

## 输出护栏分层架构

```
     LLM 输出 (原始 token 流)
           │
           ▼
    ┌──────────────┐
    │ 格式层        │ ← JSON Schema 验证、类型检查、结构完整性
    └──────┬───────┘
           ▼
    ┌──────────────┐
    │ 内容层        │ ← 敏感词过滤、PII 检测、有害内容分类
    └──────┬───────┘
           ▼
    ┌──────────────┐
    │ 语义层        │ ← 事实性核查、安全性评估、行为合规性
    └──────┬───────┘
           ▼
    ┌──────────────┐
    │ 策略层        │ ← 权限策略、HITL 审批、执行前拦截
    └──────┬───────┘
           ▼
     安全输出 → 执行/返回
```

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **安全 vs 可用** | 越严格的护栏产生越多的误报，影响正常使用 |
| **速度 vs 深度** | 语义审核更准确但延迟高，简单过滤快但精度低 |
| **成本 vs 覆盖** | 每层护栏都增加 token 和 API 成本 |
| **确定性 vs 灵活性** | 硬规则拦截精确但僵化，LLM 审核灵活但不可靠 |

## 输出审核评估指标

| 指标 | 含义 | 目标 |
|------|------|------|
| True Positive Rate (TPR) | 正确拦截的有害输出比例 | > 95% |
| False Positive Rate (FPR) | 无害输出被误拦的比例 | < 3% |
| Precision | 拦截结果中有害的真实比例 | > 90% |
| Recall | 所有有害输出中被拦截的比例 | > 95% |
| Reject Rate | 输出被拦截的整体比例 | 取决于场景 |
| Human Escalation Rate | 需要人工判断的比例 | < 5% |
| Latency Overhead | 护栏引入的额外延迟 | < 500ms |

## 护栏部署模式

```python
class OutputGuardrail:
    """输出护栏基类——所有护栏类型的统一接口"""

    def __init__(self, config: GuardrailConfig):
        self.config = config
        self.metrics = GuardrailMetrics()

    def check(self, agent_output: Output) -> GuardrailResult:
        """执行输出检查，返回通过/拦截/标记"""
        ...

class GuardrailPipeline:
    """多级护栏管道——依次执行各层检查"""

    def __init__(self):
        self.guardrails = [
            FormatValidator(),         # 格式层
            ContentFilter(),           # 内容层
            SemanticGuardrail(),       # 语义层
            PolicyEnforcer(),          # 策略层
        ]

    def process(self, output: Output) -> ProcessedOutput:
        for guardrail in self.guardrails:
            result = guardrail.check(output)
            if result.action == "block":
                return self._handle_block(output, result)
            elif result.action == "flag":
                output = self._handle_flag(output, result)
        return ProcessedOutput(output, status="safe")
```

## 主流防护库对比 (2025-2026)

| 工具/库 | 类型 | 核心能力 | 适用场景 | 开源 |
|--------|------|---------|---------|------|
| NVIDIA NeMo Guardrails | 完整框架 | 对话护栏、行为护栏、检索护栏 | 生产级 Agent | ✅ |
| Guardrails AI | 完整框架 | XML 策略定义、多模型支持 | 通用 Agent | ✅ |
| Llama Guard (Meta) | 分类模型 | 安全分类、风险等级 | 内容过滤 | ✅ |
| Aegis (Acacian) | Auto-instrument | 零代码接入 10+ 框架 | 快速部署 | ✅ |
| SingGuard-NSFA | 分类 + 推理 | 生成式推理 + 实时分类 | Agent 安全 | ✅ |
| OpenAI Moderation API | API 服务 | 内容安全分类 | OpenAI 用户 | ❌ |
| Azure Content Safety | API 服务 | 多模态安全 | Azure 用户 | ❌ |
| AWS Bedrock Guardrails | API 服务 | 策略配置 + 敏感词 | AWS 用户 | ❌ |

## 关键认知

- **输出护栏不是可选的**：Agent 场景下，输出直接驱动操作，必须审核
- **多层优于单层**：格式 + 内容 + 语义 + 策略组合远强于任何单层
- **规则 + ML 混合**：精确的规则处理已知风险，ML/LLM 处理未知风险
- **护栏本身也是攻击面**：攻击者可能尝试逆向工程、绕过或欺骗护栏
- **人工永远是最后防线**：高风险操作必须保留 HITL 能力
