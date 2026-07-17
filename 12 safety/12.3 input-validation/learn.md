# 12.3 input-validation — 输入验证与清洗

**Agent 的输入决定 Agent 的行为。** 与传统 API 不同，Agent 的输入不仅有结构化参数，还有自然语言指令、文档、图片等多模态内容——这使得输入验证远不止检查类型和格式。

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|----------|------|
| 12.3.1 Schema Validation | 如何验证结构化输入的模式？ | ★★★☆☆ |
| 12.3.2 Type Checking | 如何做类型检查与转换？ | ★★★☆☆ |
| 12.3.3 Boundary Checking | 如何检查输入的边界条件？ | ★★★☆☆ |
| 12.3.4 Encoding Sanitize | 如何清洗编码混淆的输入？ | ★★★★☆ |
| 12.3.5 Rate Limit Input | 如何限制输入速率防滥用？ | ★★★☆☆ |
| 12.3.6 Allowlist/Blocklist | 如何设计白名单/黑名单策略？ | ★★★☆☆ |

## Agent 输入验证 vs 传统 API 输入验证

```
传统 API 输入验证                     Agent 输入验证
───────────────────                  ───────────────────
• JSON Schema 校验                    • JSON Schema + 语义验证
• 类型检查 (int, string, bool)        • 类型 + 内容安全 + 指令检测
• 值范围检查 (min, max)              • 边界 + 对抗性输入检测
• 必选/可选参数校验                   • 必选 + 上下文依赖校验
• CRLF/SQL 注入防护                  • Prompt 注入 + 工具欺骗防护
• 请求体大小限制                      • Token 预算 + 窗口限制

核心区别：Agent 的"指令输入"需要语义级别的理解，
         不仅仅是语法级别的校验。
```

## 输入验证四层模型

```
原始输入
    │
    ▼
┌──────────────┐
│ 语法层        │ ← Schema 验证、类型检查、格式校验
│ Syntax Check  │   JSON Schema / Pydantic / Zod
└──────┬───────┘
       ▼
┌──────────────┐
│ 边界层        │ ← 长度/范围/数量/频率限制
│ Boundary      │   输入 Token 预算、工具调用次数
└──────┬───────┘
       ▼
┌──────────────┐
│ 编码层        │ ← Unicode 归一化、编码检测、混淆识别
│ Encoding      │   Base64/Hex/混淆/多语言编码清洗
└──────┬───────┘
       ▼
┌──────────────┐
│ 内容层        │ ← 安全检测、注入检测、策略执行
│ Content       │   Prompt 注入/敏感内容/越狱检测
└──────┬───────┘
       ▼
  安全输入 → Agent 处理
```

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **验证深度 vs 延迟** | 深层语义验证需要 LLM 调用，大幅增加延迟 |
| **严格校验 vs 灵活性** | Agent 需要理解自然语言，过度校验限制表达能力 |
| **提前拒绝 vs 运行时检测** | 输入时拒绝可能误伤，运行时检测可能太晚 |
| **结构化 vs 非结构化** | 工具调用参数可结构化校验，但用户提示词本质自由文本 |

## 验证策略决策矩阵

| 输入类型 | 验证方法 | 严格度 | 延迟影响 |
|---------|---------|--------|---------|
| 工具参数 (JSON) | Schema 验证 + 类型检查 | 严格 | 极小 |
| 用户文本指令 | 内容安全 + 注入检测 | 中等 | 中等 |
| 上传文档 | 编码检测 + 大小限制 + 内容扫描 | 较严 | 大 |
| 图片/音频 | 格式检查 + 大小限制 + 内容审核 | 中等 | 大 |
| 系统配置参数 | Schema + 边界 + 类型 + 白名单 | 严格 | 极小 |
| 工具返回结果 | Schema 验证 + 大小限制 | 中等 | 小 |

## 验证通过/拒绝决策树

```python
class InputValidator:
    """分层输入验证器——从上往下依次过滤"""

    def validate(self, user_input: Input) -> ValidationResult:
        # 1. 语法层
        syntax_ok = self._check_syntax(user_input)
        if not syntax_ok:
            return ValidationResult.reject("语法校验不通过")

        # 2. 边界层
        bounds_ok = self._check_bounds(user_input)
        if not bounds_ok:
            return ValidationResult.reject("超出边界限制")

        # 3. 编码层
        encoding_ok = self._sanitize_encoding(user_input)
        if not encoding_ok:
            return ValidationResult.reject("编码异常")

        # 4. 内容层 (最耗时)
        content_ok = self._check_content(user_input)
        if content_ok.action == "block":
            return ValidationResult.reject("内容安全检测不通过")
        elif content_ok.action == "flag":
            return ValidationResult.flag(content_ok.reason)

        return ValidationResult.pass_()
```

## 关键认知

- **验证不等于拒绝**：很多时候需要"标记+放行"而不是直接拒绝
- **延迟预算分配**：语法+边界应该在 1ms 内完成，内容层可以分配 100-500ms
- **信任层级设计**：不同来源的输入应有不同的验证严格度
- **无法 100% 覆盖**：输入验证的目标是让攻击成本 > 攻击收益
- **分级验证**：高安全场景用多重验证，低风险场景用轻量验证
