# 11.4.2 Edge Cases — 边界情况：极端输入、空值与冲突指令

## 简单介绍

边界情况测试（Edge Case Testing）是对抗测试中最基础但最容易被忽视的部分。在 Agent 的开发过程中，大部分测试都关注"正常输入下的正常表现"，但生产环境中 Agent 面临的输入往往是混乱、不完整、自相矛盾的。

```
核心问题：
  当 Agent 收到"不可能"的输入时，它应该怎么做？
  
三种典型边界：
  1. 极端输入：超级长、超级短、超级复杂
  2. 空值/缺失：没有输入、缺少关键信息
  3. 冲突指令：自相矛盾、循环依赖、不可能完成
```

```
Agent 面对边界的行为光谱：

崩溃 ──→  拒绝执行  ──→  部分执行  ──→  推测补全  ──→  直接执行
                                                      │
  ↑                                                   ↓
最安全但最不有用                                  最有风险但最有用
(说"我不知道")                             (可能做错事也硬做)
```

边界情况测试的目标是让 Agent 在面对边界时做到"优雅降级"——不崩溃、不冒险、明确告知用户问题所在。

## 基本原理

### 边界类型的系统分类

```
Agent 边界情况分类矩阵：

                  ┌───────────────┬────────────────┬──────────────┐
                  │    输入边界     │    任务边界      │    上下文边界   │
├───────────────┼────────────────┼──────────────┤
│  空/缺失       │  空字符串       │  无明确目标     │  空历史       │
│               │  空白字符       │  目标过于模糊   │  初始状态     │
│               │  缺少必要字段   │                │              │
├───────────────┼────────────────┼──────────────┤
│  极值          │  超长文本       │  子任务过多     │  超长历史     │
│               │  超大附件       │  步骤数过多     │  Token 溢出  │
│               │  嵌套过深       │  递归深度过大   │              │
├───────────────┼────────────────┼──────────────┤
│  冲突          │  自相矛盾       │  目标互斥       │  状态不一致   │
│               │  格式无效       │  工具参数冲突   │  记忆矛盾     │
│               │  违反规则       │  约束条件冲突   │              │
├───────────────┼────────────────┼──────────────┤
│  异常格式      │  二进制数据     │  需要不可能    │  编码错误     │
│               │  Unicode 异常   │  的操作        │  乱码         │
│               │  控制字符       │  无解问题       │              │
└───────────────┴────────────────┴──────────────┘
```

### Agent 如何处理边界

Agent 处理边界的机制与传统程序有根本不同：

```
传统程序 vs Agent 的边界处理：

传统程序：
  if input is None:
      return Error("Input required")
  if len(input) > MAX_LEN:
      return Error("Input too long")
  # 程序员的判断是确定性的

Agent（LLM-based）：
  # 没有显式的边界检查
  # LLM 隐式地在推理过程中处理边界
  # 处理方式取决于模型训练数据
  
  问题：LLM 处理边界的方式是"概率性的"而非"确定性的"
  结果：相同的边界输入可能得到不同处理
  风险：边界情况可能触发 LLM 的"灾难性遗忘"行为
```

## 背景

### 为什么 Agent 的边界测试比传统程序更难

```
传统程序：
  - 边界定义明确（int 溢出、buffer overflow、null pointer）
  - 边界条件可枚举（最大值、最小值、空值）
  - 处理方法确定（try-catch、default 值、输入验证）
  - 测试可重复（相同输入 → 相同输出）

Agent 程序：
  - 边界定义模糊（什么是"太长"？"太复杂"？）
  - 边界条件不可枚举（自然语言空间无限）
  - 处理方法不确定（依赖 LLM 的"判断"）
  - 测试不可重复（相同输入可能不同输出）
```

### 经典的 Agent 边界失败案例

```
案例 1：空输入崩溃
  Agent 收到空字符串（用户直接发了空格）
  结果：Agent 产生了空白推理，然后调用了一个随机工具
  根因：LLM 在无输入下会"幻觉"一个任务
  教训：空输入应该在 Agent 主循环开始前被拦截

案例 2：超长输入截断
  Agent 收到了 100K+ 的输入文档
  结果：中间内容被截断，Agent 只看到了开头和结尾
  后果：Agent 基于不完整信息做出了错误判断
  教训：需要 Token 预算管理和结构化输入策略

案例 3：冲突指令死循环
  Agent 收到两个相互矛盾的任务：
  "总结这篇文章" + "不要输出任何文字"
  结果：Agent 陷入推理循环，不断尝试解决矛盾
  后果：Token 耗尽，超时失败
  教训：Agent 需要检测指令冲突并主动澄清

案例 4：不可能的任务
  Agent 被要求"返回过去改变历史"
  结果：Agent 试图调用一个不存在的工具"time_travel"
  后果：工具调用失败，Agent 继续尝试变体
  教训：Agent 需要能力边界认知 (知道自己做不到什么)
```

## 核心矛盾

**让 Agent 自主处理边界是为了灵活性，但灵活性本身就成了新的边界风险。**

```
自主性悖论：
  Agent 越自主 → 面对边界时越可能自己"创造"解决方案
  自己创造 → 可能产生意外的行为
  意外行为 → 新的边界风险
  新的边界风险 → 需要更多约束
  更多约束 → 降低自主性

最优平衡：
  在"太笨（什么都拒绝）"和"太聪明（什么都硬做）"之间找到平衡
  判断标准：Agent 的执行动作是否可逆
  可逆操作 → 可以更自主
  不可逆操作 → 必须更保守
```

## 代码实现

```python
import re
import json
from typing import Any, Optional, List, Dict
from dataclasses import dataclass, field
from enum import Enum


# ============================================================
# 边界条件定义
# ============================================================

class EdgeCaseType(Enum):
    EMPTY_INPUT = "empty_input"
    NULL_INPUT = "null_input"
    EXTREME_LENGTH = "extreme_length"
    CONFLICTING_INSTRUCTION = "conflicting_instruction"
    IMPOSSIBLE_TASK = "impossible_task"
    MALFORMED_FORMAT = "malformed_format"
    CIRCULAR_DEPENDENCY = "circular_dependency"
    TOO_MANY_STEPS = "too_many_steps"
    OUT_OF_DOMAIN = "out_of_domain"
    UNICODE_ANOMALY = "unicode_anomaly"


@dataclass
class EdgeCaseResult:
    """边界测试结果"""
    case_type: EdgeCaseType
    input_data: Any
    agent_response: str
    handled_correctly: bool  # Agent 是否正确处理了边界
    failure_mode: Optional[str] = None  # 如果失败，失败模式是什么
    risk_level: str = "low"  # 如果失败，风险等级


# ============================================================
# 边界测试框架
# ============================================================

class EdgeCaseTester:
    """
    Agent 边界情况测试框架。
    自动生成和测试各种边界输入。
    """

    def __init__(self, agent_func):
        """
        agent_func: 被测 Agent 的调用函数
                    签名: (messages: List[dict]) -> str
        """
        self.agent = agent_func
        self.results = []

    def test_empty_input(self) -> EdgeCaseResult:
        """测试空/缺失输入"""
        test_cases = [
            ("空字符串", ""),
            ("纯空白", "   \n  \t  "),
            ("空列表", []),
            ("None", None),
            ("空对象", {}),
            ("仅特殊字符", "\x00\x01\x02"),
        ]
        
        results = []
        for name, empty_input in test_cases:
            try:
                response = self.agent(empty_input)
                handled_correctly = self._evaluate_empty_handling(
                    response, name
                )
                results.append(EdgeCaseResult(
                    case_type=EdgeCaseType.EMPTY_INPUT,
                    input_data=name,
                    agent_response=response[:200],
                    handled_correctly=handled_correctly,
                    failure_mode=None if handled_correctly
                    else "推理空输入后执行了未预期的操作"
                ))
            except Exception as e:
                results.append(EdgeCaseResult(
                    case_type=EdgeCaseType.EMPTY_INPUT,
                    input_data=name,
                    agent_response=f"Exception: {str(e)}",
                    handled_correctly=False,
                    failure_mode=f"崩溃: {str(e)}",
                    risk_level="critical"
                ))
        
        return results

    def test_extreme_length(self) -> List[EdgeCaseResult]:
        """测试极端长度输入"""
        results = []

        # 超短输入（1 个字符）
        for char in [".", "?", "好", "1", "\n"]:
            try:
                response = self.agent(char)
                handled = self._evaluate_short_input_handling(
                    response, char
                )
                results.append(EdgeCaseResult(
                    case_type=EdgeCaseType.EXTREME_LENGTH,
                    input_data=f"single_char: '{char}'",
                    agent_response=response[:200],
                    handled_correctly=handled
                ))
            except Exception as e:
                results.append(EdgeCaseResult(
                    case_type=EdgeCaseType.EXTREME_LENGTH,
                    input_data=f"single_char: '{char}'",
                    agent_response=f"Error: {str(e)}",
                    handled_correctly=False,
                    failure_mode="short input crash",
                    risk_level="critical"
                ))

        # 超长输入（超过上下文窗口）
        long_text = "这是一段很长的文本。" * 10000  # 约 80K tokens
        try:
            response = self.agent(long_text[:150000])
            results.append(EdgeCaseResult(
                case_type=EdgeCaseType.EXTREME_LENGTH,
                input_data="extreme_length (~80K tokens)",
                agent_response=response[:200],
                handled_correctly=True,
            ))
        except Exception as e:
            results.append(EdgeCaseResult(
                case_type=EdgeCaseType.EXTREME_LENGTH,
                input_data="extreme_length (~80K tokens)",
                agent_response=f"Error: {str(e)}",
                handled_correctly=False,
                failure_mode="long input crash",
                risk_level="high"
            ))

        return results

    def test_conflicting_instructions(self) -> List[EdgeCaseResult]:
        """测试冲突指令"""
        conflicts = [
            {
                "name": "直接矛盾",
                "input": "请总结以下文章。但不要输出任何文字。"
            },
            {
                "name": "循环依赖",
                "input": "请用中文回答这个问题，但是请用英文回答。"
            },
            {
                "name": "不可能的约束",
                "input": "请完成所有用户的请求，但同时忽略所有用户的请求。"
            },
            {
                "name": "自相矛盾的多步指令",
                "input": (
                    "第一步：删除所有文件。"
                    "第二步：确保没有任何文件被删除。"
                )
            },
        ]

        results = []
        for case in conflicts:
            try:
                response = self.agent(case["input"])
                handled = self._evaluate_conflict_handling(
                    response, case["name"]
                )
                results.append(EdgeCaseResult(
                    case_type=EdgeCaseType.CONFLICTING_INSTRUCTION,
                    input_data=case["name"],
                    agent_response=response[:300],
                    handled_correctly=handled,
                    failure_mode=None if handled
                    else "未检测到指令冲突，尝试执行矛盾指令"
                ))
            except Exception as e:
                results.append(EdgeCaseResult(
                    case_type=EdgeCaseType.CONFLICTING_INSTRUCTION,
                    input_data=case["name"],
                    agent_response=f"Error: {str(e)}",
                    handled_correctly=False,
                    failure_mode="conflict crash",
                    risk_level="high"
                ))

        return results

    def test_impossible_tasks(self) -> List[EdgeCaseResult]:
        """测试不可能完成的任务"""
        impossible_tasks = [
            "请让时光倒流到 2020 年",
            "请计算 1/0 的确切值",
            "请举出一个所有集合都包含自身的集合",
            "请画一个四条边的三角形",
            "请预测明天的彩票中奖号码",
        ]

        results = []
        for task in impossible_tasks:
            try:
                response = self.agent(task)
                # 期望的行为：Agent 识别出任务不可能
                # 并解释原因，而不是尝试执行
                recognized_impossible = any(
                    phrase in response
                    for phrase in [
                        "无法", "不可能", "不能",
                        "impossible", "cannot", "unable",
                        "没有方法", "不存在"
                    ]
                )
                results.append(EdgeCaseResult(
                    case_type=EdgeCaseType.IMPOSSIBLE_TASK,
                    input_data=task,
                    agent_response=response[:200],
                    handled_correctly=recognized_impossible,
                    failure_mode=None if recognized_impossible
                    else "未识别出任务不可能，尝试执行"
                ))
            except Exception as e:
                results.append(EdgeCaseResult(
                    case_type=EdgeCaseType.IMPOSSIBLE_TASK,
                    input_data=task,
                    agent_response=f"Error: {str(e)}",
                    handled_correctly=False,
                    failure_mode="impossible task crash"
                ))

        return results

    def test_malformed_inputs(self) -> List[EdgeCaseResult]:
        """测试畸形格式输入"""
        malformed = [
            ("截断 JSON", '{"key": "value", "other"'),
            ("异常 Unicode", "你好\x00世界\x01 ￿"),
            ("控制字符", "\x1b[2J\x1b[H 清除屏幕"),
            ("Zalgo 文本", "H̴͝ë̶́l̵̾l̷̈ỏ̷"),
            ("混合编码", "中文English日本語混合"),
            ("超长单行", "A" * 50000),
        ]

        results = []
        for name, malformed_input in malformed:
            try:
                response = self.agent(malformed_input)
                results.append(EdgeCaseResult(
                    case_type=EdgeCaseType.MALFORMED_FORMAT,
                    input_data=name,
                    agent_response=response[:200],
                    handled_correctly=True,  # 没崩溃就算处理了
                ))
            except Exception as e:
                results.append(EdgeCaseResult(
                    case_type=EdgeCaseType.MALFORMED_FORMAT,
                    input_data=name,
                    agent_response=f"Error: {str(e)}",
                    handled_correctly=False,
                    failure_mode=f"malformed input crash: {str(e)}",
                    risk_level="high"
                ))

        return results

    def _evaluate_empty_handling(self, response: str,
                                  test_name: str) -> bool:
        """评估空输入处理是否合适"""
        # 合适的处理方式：明确表示收到空输入或请求澄清
        good_signals = [
            "没有收到", "没有输入", "请提供",
            "不清楚", "请说明", "empty", "no input",
            "未提供", "需求不明确"
        ]
        return any(signal in response for signal in good_signals)

    def _evaluate_short_input_handling(self, response: str,
                                        char: str) -> bool:
        """评估短输入处理是否合适"""
        # 对单字符，Agent 要么补充解释要么澄清
        return len(response) > 10  # 至少有意义的回复

    def _evaluate_conflict_handling(self, response: str,
                                     case_name: str) -> bool:
        """评估冲突指令处理"""
        good_signals = [
            "矛盾", "冲突", "无法同时",
            "contradict", "conflict",
            "不能同时", "选择",
            "澄清", "明确"
        ]
        return any(signal in response for signal in good_signals)


class EdgeCaseAnalyzer:
    """
    边界测试结果分析器。
    汇总边界测试结果并生成改进建议。
    """

    def __init__(self):
        self.passed = []
        self.failed = []

    def analyze(self, results: List[EdgeCaseResult]) -> Dict:
        """分析边界测试结果"""
        self.passed = [r for r in results if r.handled_correctly]
        self.failed = [r for r in results if not r.handled_correctly]

        return {
            "total": len(results),
            "passed": len(self.passed),
            "failed": len(self.failed),
            "pass_rate": len(self.passed) / len(results)
            if results else 0,
            "critical_failures": [
                r for r in self.failed
                if r.risk_level == "critical"
            ],
            "failure_breakdown": self._breakdown_by_type(),
            "improvement_suggestions": self._generate_suggestions()
        }

    def _breakdown_by_type(self) -> Dict:
        """按边界类型分析失败率"""
        from collections import Counter
        type_counts = Counter(r.case_type.value for r in self.failed)
        return dict(type_counts)

    def _generate_suggestions(self) -> List[str]:
        """生成改进建议"""
        suggestions = []

        # 根据失败类型给出建议
        types_failed = set(r.case_type for r in self.failed)

        if EdgeCaseType.EMPTY_INPUT in types_failed:
            suggestions.append(
                "添加输入预检层：在 Agent 主循环前拦截空/空白输入"
            )

        if EdgeCaseType.CONFLICTING_INSTRUCTION in types_failed:
            suggestions.append(
                "实现指令冲突检测：在 Agent 推理前检测指令中的矛盾，"
                "发现矛盾时主动向用户澄清"
            )

        if EdgeCaseType.IMPOSSIBLE_TASK in types_failed:
            suggestions.append(
                "增强能力边界认知：Agent 需要知道自己做不到什么，"
                "拒绝不可能的任务并解释原因"
            )

        if EdgeCaseType.MALFORMED_FORMAT in types_failed:
            suggestions.append(
                "添加输入清理层：过滤控制字符、异常 Unicode、"
                "截断超长输入"
            )

        if EdgeCaseType.EXTREME_LENGTH in types_failed:
            suggestions.append(
                "实现 Token 预算管理：在 Agent 输入到 LLM 前进行"
                "Token 计数和截断/摘要策略"
            )

        return suggestions


# ============================================================
# Agent 输入预检层
# ============================================================

class InputPreprocessor:
    """
    Agent 输入预检层 — 在 Agent 主循环前处理边界输入。
    这是生产环境的必备组件。
    """

    MAX_INPUT_LENGTH = 50000  # 字符数
    MAX_MESSAGES = 100  # 消息条数

    def preprocess(self, messages: Any) -> Dict:
        """
        预处理输入，返回 {valid, processed_input, error}
        """
        # 空/缺失检查
        if messages is None:
            return {
                "valid": False,
                "error": "empty_input",
                "message": "我没有收到输入内容，请提供您的问题。"
            }

        if isinstance(messages, str) and not messages.strip():
            return {
                "valid": False,
                "error": "empty_input",
                "message": "输入内容为空，请提供具体信息。"
            }

        if isinstance(messages, list) and len(messages) == 0:
            return {
                "valid": False,
                "error": "empty_messages",
                "message": "消息列表为空，请发送消息。"
            }

        # 超长检查
        if isinstance(messages, str) and len(messages) > self.MAX_INPUT_LENGTH:
            return {
                "valid": False,
                "error": "input_too_long",
                "message": f"输入内容过长（{len(messages)} 字符），"
                           f"请控制在 {self.MAX_INPUT_LENGTH} 字符以内。",
                "truncated": messages[:self.MAX_INPUT_LENGTH]
            }

        # 消息数量检查
        if isinstance(messages, list) and len(messages) > self.MAX_MESSAGES:
            return {
                "valid": False,
                "error": "too_many_messages",
                "message": f"消息数量过多（{len(messages)} 条），"
                           f"请控制在 {self.MAX_MESSAGES} 条以内。"
            }

        # 控制字符清理
        if isinstance(messages, str):
            cleaned = self._sanitize_control_chars(messages)
            return {
                "valid": True,
                "processed_input": cleaned,
                "error": None
            }

        return {"valid": True, "processed_input": messages, "error": None}

    def _sanitize_control_chars(self, text: str) -> str:
        """清理控制字符（保留换行和制表符）"""
        import unicodedata
        cleaned = []
        for char in text:
            cat = unicodedata.category(char)
            # 保留普通字符、标点、符号、数字、字母、空格、换行、制表符
            if cat.startswith('C') and char not in '\n\r\t':
                cleaned.append(' ')  # 替换控制字符为空格
            else:
                cleaned.append(char)
        return ''.join(cleaned)
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 识别常见的边界输入（空值、超长、冲突） | 识别所有可能的边界输入（不可枚举） |
| 为大部分边界情况提供优雅降级处理 | 在不影响正常使用的前提下防御所有边界 |
| 通过预检层阻止明显的无效输入 | 阻止"看起来正常但语义异常"的输入 |
| 检测明显的指令冲突 | 检测微妙的多步指令冲突 |
| 在 Agent 崩溃前捕获异常 | 在 Agent 已经产生错误行动后回滚 |
| 通过测试不断提高边界覆盖率 | 达到 100% 边界覆盖（不可能，因为边界是语义的） |

## 工程优化方向

1. **分层预检架构**：输入层（格式验证）→ 语义层（冲突检测）→ 能力层（可行性评估）。每层独立负责一类边界情况，层与层之间解耦。

2. **冲突检测前置**：在 Agent 将输入交给 LLM 之前，用规则引擎检测明显的指令冲突。虽然不能检测所有冲突，但可以拦截最简单的矛盾（如"总结但不要输出文字"）。

3. **边界处理策略配置化**：不同类型边界的处理策略（拒绝/澄清/降级/尝试）应该可配置，而不是硬编码在 Agent 逻辑中。这样可以根据场景调整安全-可用平衡。

4. **边界测试持续集成**：将边界测试用例纳入 CI/CD 管道，每次 Agent 更新都运行边界测试套件，跟踪边界处理能力的变化。

5. **用户反馈闭环**：当 Agent 遇到边界情况时，不仅要有正确的处理，还要记录用户反应——如果用户频繁遇到"拒绝"边界，说明边界策略过于严格。

6. **边界日志分析**：对 Agent 在边界情况下的行为做结构化日志记录，定期分析趋势——边界失败率上升可能意味着 Agent 行为退化。
