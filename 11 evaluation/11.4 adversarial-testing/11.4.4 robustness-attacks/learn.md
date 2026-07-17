# 11.4.4 Robustness Attacks — 鲁棒性攻击：输入扰动与噪声

## 简单介绍

鲁棒性攻击（Robustness Attacks）测试的是 Agent 在输入被轻微扰动时的稳定性。与 Prompt 注入的"恶意攻击"不同，鲁棒性攻击更多关注的是**非恶意的输入变异**——拼写错误、格式变化、同义词替换、轻微的不一致性——这些在日常使用中不可避免。

```
核心问题：
  Agent 对输入的敏感度有多高？
  一个字的改动会不会导致完全不同的输出？
  同一个意思的不同表达方式会不会得到一致的结果？

"鲁棒性"的定义：
  输入 → 微小变化 → 输出应该基本不变
  如果输入微小变化导致输出大幅变化 → 鲁棒性差
```

```
鲁棒性测试与注入测试的区别：

注入测试：                        鲁棒性测试：
─────────────────                 ─────────────────
主动构造恶意输入                   模拟日常使用变异
攻击者故意让 Agent 出错           用户无意中产生的变化
目标是突破安全边界                 目标是保持稳定性
成功＝防御被绕过                   成功＝输出一致
```

## 基本原理

### 输入扰动空间

Agent 的输入可以在多个维度被扰动：

```
扰动维度全景：

1. 词汇层扰动
   ┌─────────────────────────────────────────────┐
   │ 原文: "请帮我查询北京明天的天气"              │
   │                                             │
   │ 拼写错误:  "请帮我查洵北京明天的天气"          │
   │ 同义词替换: "请帮我查看北京明天的天气"          │
   │ 语序变化:  "北京明天的天气，请帮我查询"        │
   │ 省略:      "北京明天天气"                    │
   │ 重复:      "请请帮我帮我查查北京天气天气"      │
   │ 语气词插入: "嗯...请帮我查询北京明天的天气"     │
   └─────────────────────────────────────────────┘

2. 语义层扰动
   ┌─────────────────────────────────────────────┐
   │ 歧义:     "查询北京明天的天气和后天"          │
   │          （是查询两天还是查询一天？）           │
   │ 否定:     "不要查询北京天气"                  │
   │          （双重否定绕晕 Agent）              │
   │ 指代模糊: "查一下那里的天气"                  │
   │          （"那里"指哪里？）                    │
   │ 隐含条件: "我明天去北京"                      │
   │          （隐含：需要知道明天的北京天气）        │
   └─────────────────────────────────────────────┘

3. 格式层扰动
   ┌─────────────────────────────────────────────┐
   │ 大小写:    "HELP ME CHECK WEATHER"           │
   │ 标点变化:  "请帮我查询北京明天的天气!!!!!!!!"   │
   │ 空格变异:  "请  帮我  查询  天气"              │
   │ 换行插入:  "请\n\n帮我\n查询\n天气"             │
   │ 混合语言:  "Please 帮我 query 天气"           │
   │ 表情符号:  "请帮我查询北京明天的天气☀️🌧️"      │
   └─────────────────────────────────────────────┘

4. 结构层扰动（Agent 场景特有）
   ┌─────────────────────────────────────────────┐
   │ 工具结果噪声:    API 返回了多余字段             │
   │ 记忆检索噪声:    检索到相关但部分错误的内容       │
   │ 对话历史干扰:    历史消息中包含矛盾信息           │
   │ 多轮上下文偏移:  多轮后话题逐渐漂移              │
   └─────────────────────────────────────────────┘
```

### 鲁棒性为什么会下降

Agent 鲁棒性下降的根本原因在于 LLM 的**概率性本质**：

```
LLM 生成的本质：
  给定输入序列 X，生成输出 Y
  P(Y|X) 是条件概率分布
  微小变化 δ 使得 P(Y|X+δ) ≠ P(Y|X)

为什么微小变化就能导致输出大变？
  1. Token 级别的敏感性
     "查询" vs "查洵" → Token 化结果不同 → 语义表示不同
     
  2. 注意力分布变化
     输入变化 → 注意力权重重新分布 → 关注不同的上下文
     
  3. 推理链的蝴蝶效应
     第一步的微小偏移 → 后续推理沿着不同路径
     → 最终输出完全不同
```

## 背景

### Agent 鲁棒性测试的演进

```
2023 年：基础鲁棒性测试
  • 文本分类/情感分析的对抗样本测试
  • 主要关注文本分类的鲁棒性
  • 工具：TextAttack、AdvGLUE

2024 年：LLM 鲁棒性测试
  • 关注 LLM 对输入扰动的敏感度
  • 发现 LLM 对同义句的响应不一致（语义不变性差）
  • 方法：同义词替换、语法变异、角色互换

2025-2026 年：Agent 鲁棒性测试体系化
  • 关注全链路的鲁棒性（输入→推理→工具→输出）
  • Adaptive Stress Testing：自适应搜索最坏扰动
  • Impatient Users / Noisy Environments 仿真
  • Robustness Benchmark 标准化
```

### 关键研究发现

```
输入扰动对 Agent 性能的影响（2025 年研究综合）：

扰动类型             性能下降    影响最大环节
──────────────────  ────────  ─────────────────
拼写错误 (<10%)       3-8%    工具名称/参数识别
同义词替换            5-12%    任务意图理解
语序变化             2-5%    多步推理连贯性
格式变化（大小写）    1-3%    影响较小
语气词插入           2-4%    工具调用选择
否定结构             15-25%  任务目标理解（最脆弱）
歧义指代             20-35%  多步执行路径
混合语言            10-18%    工具选择准确率
工具返回噪声         8-15%    后续推理质量

核心发现：
  • Agent 对"否定"和"歧义"的鲁棒性最差
  • 工具命名和参数识别对拼写错误敏感
  • 语义等价但表面形式不同的输入→Agent 行为不一致
  • 多轮交互中鲁棒性问题会累积放大
```

## 核心矛盾

**鲁棒性测试追求"等价输入应有等价输出"，但自然语言中"等价"本身就没有绝对定义。**

```
等价性困境：

"帮我查天气"和"查询一下天气情况"
  人类认为：意思相同 ✓
  LLM 认为：Token 序列完全不同
  Agent 行为：可能调用相同工具 ✓ 也可能不同工具 ✗

什么是"等价"？
  语义等价？—— 语义相似度是连续的，没有明确边界
  意图等价？—— 同一意图的不同表述方式
  功能等价？—— 应该触发相同的工具调用链
  结果等价？—— 最终输出应该一致

问题：没有客观的等价性标准 → 无法定义"鲁棒"的确切含义
```

## 代码实现

```python
import random
import string
from typing import List, Callable, Dict, Any
from dataclasses import dataclass, field
from enum import Enum


# ============================================================
# 扰动类型定义
# ============================================================

class PerturbationType(Enum):
    TYPO = "typo"
    SYNONYM = "synonym"
    WORD_ORDER = "word_order"
    OMISSION = "omission"
    REPETITION = "repetition"
    PUNCTUATION = "punctuation"
    WHITESPACE = "whitespace"
    CASING = "casing"
    LANGUAGE_MIX = "language_mix"
    NEGATION = "negation"
    AMBIGUITY = "ambiguity"
    NOISE_INSERTION = "noise_insertion"


@dataclass
class PerturbationResult:
    """扰动测试结果"""
    original_input: str
    perturbed_input: str
    perturbation_type: PerturbationType
    original_output: str
    perturbed_output: str
    output_similarity: float  # 0~1 输出相似度
    behavior_consistent: bool  # 行为是否一致
    degradation_level: str  # none/slight/moderate/severe


# ============================================================
# 输入扰动生成器
# ============================================================

class InputPerturbationGenerator:
    """
    输入扰动生成器 — 生成各种输入变体用于鲁棒性测试。
    """

    def __init__(self, seed: int = 42):
        random.seed(seed)
        
        # 常见中文同义词映射（简化示例）
        self.synonyms = {
            "查询": ["查询", "查一下", "看看", "搜索", "查找"],
            "天气": ["天气", "天气预报", "天气情况", "气温"],
            "帮助": ["帮助", "协助", "帮我", "请"],
            "北京": ["北京", "北京市"],
            "明天": ["明天", "明日", "第二天"],
            "今天": ["今天", "今日"],
            "设置": ["设置", "设定", "配置", "设定"],
            "删除": ["删除", "移除", "去掉", "清除"],
            "创建": ["创建", "新建", "建立", "生成"],
            "修改": ["修改", "更改", "编辑", "更新"],
            "列表": ["列表", "清单", "目录", "所有"],
            "搜索": ["搜索", "搜寻", "查找", "寻找"],
        }

    def apply_typo(self, text: str, rate: float = 0.1) -> str:
        """
        模拟拼写错误：替换、删除、插入、交换字符
        rate: 每个字符的出错概率
        """
        chars = list(text)
        result = []
        
        for char in chars:
            if random.random() >= rate:
                result.append(char)
                continue
            
            if char.strip() == '':
                result.append(char)
                continue
            
            action = random.choice(['replace', 'delete', 'insert', 'swap'])
            
            if action == 'replace':
                # 替换为相邻键盘字符（简化版）
                replacement = random.choice(
                    "qwertyuiopasdfghjklzxcvbnm"
                )
                result.append(replacement)
            elif action == 'delete':
                pass  # 跳过该字符
            elif action == 'insert':
                result.append(char)
                result.append(random.choice(
                    "abcdefghijklmnopqrstuvwxyz"
                ))
            elif action == 'swap' and len(result) > 0:
                # 与前一个字符交换
                prev = result.pop()
                result.append(char)
                result.append(prev)
        
        return ''.join(result)

    def apply_synonym_replace(self, text: str,
                               rate: float = 0.3) -> str:
        """
        同义词替换。
        rate: 可替换词被替换的概率
        """
        result = text
        
        for word, syns in self.synonyms.items():
            if word in result and random.random() < rate:
                replacement = random.choice(syns)
                result = result.replace(word, replacement, 1)
        
        return result

    def apply_word_order_change(self, text: str) -> str:
        """
        改变语序（中文语序灵活性高）。
        将动词短语移到句首/句尾。
        """
        patterns = [
            # "请帮我查询天气" → "查询天气，请帮我"
            (r"请帮我(\w+)(\w+)", r"\1\2，请帮我"),
            # "北京明天的天气" → "明天北京的天气"
            (r"(\w{2,4})明天的(\w{2,4})", r"明天\1的\2"),
        ]
        
        import re
        for pattern, replacement in patterns:
            if re.search(pattern, text):
                if random.random() < 0.5:
                    text = re.sub(pattern, replacement, text)
        
        return text

    def apply_omission(self, text: str,
                        rate: float = 0.2) -> str:
        """
        省略非关键词（助词、语气词等）。
        """
        omit_words = ["的", "了", "吗", "呢", "吧", "啊",
                       "请", "帮我", "一下"]
        
        result = text
        for word in omit_words:
            if word in result and random.random() < rate:
                result = result.replace(word, "", 1)
        
        return result

    def apply_repetition(self, text: str,
                          rate: float = 0.15) -> str:
        """重复部分字符/词语"""
        chars = list(text)
        result = []
        
        for char in chars:
            result.append(char)
            if char.strip() and random.random() < rate:
                result.append(char)  # 重复该字符
        
        return ''.join(result)

    def apply_punctuation_change(self, text: str) -> str:
        """标点符号变化"""
        # 增加感叹号
        if random.random() < 0.3:
            text += "!" * random.randint(1, 5)
        
        # 删除所有标点
        if random.random() < 0.2:
            for p in "，。！？、；：":
                text = text.replace(p, " ")
        
        return text

    def apply_casing_change(self, text: str) -> str:
        """大小写变化（主要影响英文部分）"""
        if any(c.isalpha() for c in text):
            actions = [
                lambda s: s.upper(),
                lambda s: s.lower(),
                lambda s: s.swapcase(),
                lambda s: ''.join(
                    c.upper() if random.random() < 0.5
                    else c.lower() for c in s
                ),
            ]
            return random.choice(actions)(text)
        return text

    def apply_noise_insertion(self, text: str) -> str:
        """插入无关噪声内容"""
        noises = [
            "（对了顺便问一下）",
            "【忽略这个】",
            "<extra_info>无关内容</extra_info>",
            "\n\nP.S. 我上周去了趟上海\n\n",
            "！！！",
        ]
        
        if random.random() < 0.3:
            noise = random.choice(noises)
            insert_pos = random.randint(0, len(text))
            return text[:insert_pos] + noise + text[insert_pos:]
        
        return text

    def apply_negation(self, text: str) -> str:
        """构造否定变体（测试 Agent 对否定的理解）"""
        negation_patterns = [
            ("请", "请不要"),
            ("查询", "不要查询"),
            ("帮我", "不要帮我"),
            ("我想", "我不想"),
        ]
        
        for positive, negative in negation_patterns:
            if positive in text:
                if random.random() < 0.3:
                    text = text.replace(positive, negative, 1)
                    break
        
        return text

    def generate_variants(self, text: str,
                           num_variants: int = 5) -> List[Dict]:
        """生成多个扰动变体"""
        generators = [
            ("typo", self.apply_typo),
            ("synonym", self.apply_synonym_replace),
            ("word_order", self.apply_word_order_change),
            ("omission", self.apply_omission),
            ("repetition", self.apply_repetition),
            ("punctuation", self.apply_punctuation_change),
            ("noise", self.apply_noise_insertion),
            ("negation", self.apply_negation),
        ]
        
        variants = []
        for _ in range(num_variants):
            variant_text = text
            applied_types = []
            
            # 随机选择 1-3 种扰动叠加
            num_perturbations = random.randint(1, 3)
            selected = random.sample(generators,
                                     num_perturbations)
            
            for ptype, generator in selected:
                variant_text = generator(variant_text)
                applied_types.append(ptype)
            
            variants.append({
                "perturbed": variant_text,
                "perturbations": applied_types,
            })
        
        return variants


# ============================================================
# 鲁棒性测试框架
# ============================================================

class RobustnessTester:
    """
    Agent 鲁棒性测试框架。
    对同一语义的多种输入变体，测试输出一致性。
    """

    def __init__(self, agent_func: Callable):
        self.agent = agent_func
        self.generator = InputPerturbationGenerator()

    def test_semantic_consistency(self, test_input: str,
                                   num_variants: int = 5) -> Dict:
        """
        测试语义一致性：同一意思的不同表达 → 输出是否一致
        """
        # 原始输出
        original_output = self.agent(test_input)
        
        # 生成扰动变体
        variants = self.generator.generate_variants(
            test_input, num_variants
        )
        
        results = []
        for variant in variants:
            perturbed_output = self.agent(variant["perturbed"])
            
            # 计算输出相似度
            similarity = self._compute_output_similarity(
                original_output, perturbed_output
            )
            
            # 判断行为一致性
            behavior_consistent = (
                self._check_behavior_consistency(
                    original_output, perturbed_output
                )
            )
            
            results.append({
                "perturbed_input": variant["perturbed"],
                "perturbations": variant["perturbations"],
                "original_output": original_output[:200],
                "perturbed_output": perturbed_output[:200],
                "similarity": round(similarity, 3),
                "behavior_consistent": behavior_consistent,
            })
        
        # 统计
        similarities = [r["similarity"] for r in results]
        consistency_rate = sum(
            1 for r in results if r["behavior_consistent"]
        ) / len(results) if results else 0

        return {
            "original_input": test_input,
            "num_variants": len(results),
            "original_output": original_output[:200],
            "avg_similarity": round(
                sum(similarities) / len(similarities), 3
            ) if similarities else 0,
            "min_similarity": round(min(similarities), 3)
            if similarities else 0,
            "max_similarity": round(max(similarities), 3)
            if similarities else 0,
            "consistency_rate": round(consistency_rate, 3),
            "variant_results": results,
            "overall_verdict": (
                "ROBUST" if consistency_rate >= 0.8
                else "MODERATELY_ROBUST" if consistency_rate >= 0.5
                else "NOT_ROBUST"
            )
        }

    def test_tool_call_robustness(self,
                                   tool_call_description: str,
                                   param_variants: List[str]) -> Dict:
        """
        测试工具调用鲁棒性：同一工具调用的不同参数表达
        → Agent 是否能正确识别工具和参数
        """
        results = []
        
        for variant in param_variants:
            output = self.agent(variant)
            
            # 判断是否调用了正确的工具
            correct_tool = self._check_correct_tool(
                output, tool_call_description
            )
            
            # 判断参数是否正确提取
            correct_params = self._check_correct_params(
                output, variant
            )
            
            results.append({
                "input": variant,
                "correct_tool": correct_tool,
                "correct_params": correct_params,
                "output": output[:200]
            })
        
        correct_tool_rate = sum(
            1 for r in results if r["correct_tool"]
        ) / len(results) if results else 0
        
        correct_param_rate = sum(
            1 for r in results if r["correct_params"]
        ) / len(results) if results else 0

        return {
            "test_description": tool_call_description,
            "num_variants": len(results),
            "tool_accuracy": round(correct_tool_rate, 3),
            "param_accuracy": round(correct_param_rate, 3),
            "results": results
        }

    def test_noise_tolerance(self, clean_input: str,
                              noise_levels: List[float]) -> Dict:
        """
        测试噪声容忍度：逐渐增加噪声水平，观察性能退化曲线。
        noise_levels: 噪声水平列表 [0.05, 0.1, 0.2, 0.5]
        """
        degradation_curve = []
        baseline_output = self.agent(clean_input)
        
        for noise_level in noise_levels:
            noisy_input = self.generator.apply_typo(
                clean_input, rate=noise_level
            )
            noisy_output = self.agent(noisy_input)
            
            similarity = self._compute_output_similarity(
                baseline_output, noisy_output
            )
            
            degradation_curve.append({
                "noise_level": noise_level,
                "similarity": round(similarity, 3),
                "noisy_input": noisy_input,
            })

        return {
            "clean_input": clean_input,
            "noise_levels_tested": noise_levels,
            "degradation_curve": degradation_curve,
            "breakpoint": self._find_breakpoint(
                degradation_curve
            )
        }

    def _compute_output_similarity(self, output_a: str,
                                     output_b: str) -> float:
        """
        计算两个输出的语义相似度。
        简化实现：基于 BLEU 风格的重叠 n-gram 比率。
        生产环境建议使用语义嵌入相似度。
        """
        # 简单 token 重叠率
        tokens_a = set(output_a.lower().split())
        tokens_b = set(output_b.lower().split())
        
        if not tokens_a or not tokens_b:
            return 0.0
        
        intersection = tokens_a & tokens_b
        union = tokens_a | tokens_b
        
        return len(intersection) / len(union)

    def _check_behavior_consistency(self, output_a: str,
                                     output_b: str) -> bool:
        """
        检查行为一致性：两个输出是否代表相同的 Agent 行为。
        简化实现：检查关键动作词是否一致。
        """
        action_keywords = ["调用", "查询", "创建", "删除",
                           "修改", "设置", "搜索", "分析",
                           "总结", "回复", "call", "search",
                           "create", "delete", "update"]
        
        actions_a = set()
        actions_b = set()
        
        for kw in action_keywords:
            if kw in output_a:
                actions_a.add(kw)
            if kw in output_b:
                actions_b.add(kw)
        
        if not actions_a and not actions_b:
            return True  # 没有明确动作 → 认为一致
        
        # 检查动作集是否有显著重叠
        if actions_a and actions_b:
            overlap = actions_a & actions_b
            return len(overlap) / max(len(actions_a),
                                      len(actions_b)) >= 0.5
        
        return False

    def _check_correct_tool(self, output: str,
                             expected_tool: str) -> bool:
        """检查是否调用了正确的工具"""
        return expected_tool.lower() in output.lower()

    def _check_correct_params(self, output: str,
                               input_text: str) -> bool:
        """检查参数是否正确提取（简化版）"""
        import re
        number_pattern = r'\d+'
        numbers_in_input = re.findall(number_pattern, input_text)
        numbers_in_output = re.findall(number_pattern, output)
        
        # 检查输入中的数字是否出现在输出中
        if numbers_in_input:
            return any(
                n in " ".join(numbers_in_output)
                for n in numbers_in_input
            )
        return True

    def _find_breakpoint(self, curve: List[Dict]) -> Dict:
        """找到性能退化曲线的拐点"""
        if len(curve) < 2:
            return {"breakpoint": None, "note": "数据不足"}
        
        max_drop = 0
        breakpoint = None
        
        for i in range(1, len(curve)):
            drop = curve[i-1]["similarity"] - curve[i]["similarity"]
            if drop > max_drop:
                max_drop = drop
                breakpoint = curve[i]
        
        return {
            "breakpoint": breakpoint,
            "max_drop": round(max_drop, 3),
            "interpretation": (
                f"最大退化发生在噪声水平 "
                f"{breakpoint['noise_level']}，"
                f"相似度降至 {breakpoint['similarity']}"
                if breakpoint else "无明显拐点"
            )
        }


# ============================================================
# 鲁棒性报告生成
# ============================================================

class RobustnessReport:
    """鲁棒性测试报告生成"""

    @staticmethod
    def generate(test_results: List[Dict]) -> Dict:
        """生成综合鲁棒性报告"""
        reports = []
        
        for result in test_results:
            report = {
                "input": result.get("original_input", "")[:50],
                "consistency_rate": result.get("consistency_rate", 0),
                "verdict": result.get("overall_verdict", "UNKNOWN"),
                "weaknesses": []
            }
            
            # 找出最不相似的变体
            for variant in result.get("variant_results", []):
                if variant.get("similarity", 1) < 0.5:
                    report["weaknesses"].append({
                        "perturbation": variant.get(
                            "perturbations", []
                        ),
                        "perturbed_input": variant.get(
                            "perturbed_input", ""
                        )[:50],
                        "similarity": variant.get(
                            "similarity", 0
                        )
                    })
            
            reports.append(report)
        
        # 汇总统计
        verdicts = [
            r["verdict"] for r in reports
        ]
        robust_count = verdicts.count("ROBUST")
        moderate_count = verdicts.count("MODERATELY_ROBUST")
        fragile_count = verdicts.count("NOT_ROBUST")
        
        return {
            "total_tests": len(reports),
            "robust": robust_count,
            "moderately_robust": moderate_count,
            "not_robust": fragile_count,
            "robustness_score": round(
                (robust_count + moderate_count * 0.5)
                / max(len(reports), 1), 3
            ),
            "overall_assessment": (
                "GOOD" if robust_count / max(len(reports), 1) > 0.7
                else "NEEDS_IMPROVEMENT"
            ),
            "top_weaknesses": [
                w for r in reports for w in r["weaknesses"]
            ][:10],  # 前 10 个最严重的鲁棒性问题
            "detail_reports": reports
        }
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 测试拼写/同义/格式变化的鲁棒性 | 覆盖所有自然语言变体（空间无限） |
| 量化输出一致性（相似度评分） | 完美度量"语义等价" |
| 找到导致性能退化的扰动类型 | 修复所有发现的鲁棒性问题 |
| 建立鲁棒性基线并跟踪变化 | 预测未测试扰动的效果 |
| 对比不同 Agent 版本的鲁棒性 | 确保 100% 的语义一致性（LLM 的固有特性） |
| 定位 Agent 最脆弱的输入维度 | 消除噪声和扰动对 Agent 的所有影响 |

## 工程优化方向

1. **输入标准化**：在 Agent 处理输入前进行标准化——修正拼写、统一术语、规范化格式。虽然不能消除所有扰动，但可以大幅降低常见的变异。

2. **多视角集成**：同一输入用轻微不同的表达方式调用 Agent 多次，聚合结果。虽然成本高，但对关键任务可以显著提高鲁棒性。

3. **对抗训练数据增强**：在微调 Agent 使用的 LLM 时，加入扰动训练数据——拼写错误版本、同义词替换版本等，让模型学会对表面形式变化不敏感。

4. **鲁棒性监控仪表盘**：在生产环境中持续采样检测鲁棒性——将实际用户输入与标准输入对比，检测因输入变异导致的输出不一致。

5. **输入-输出嵌入跟踪**：维护输入嵌入和输出嵌入的映射关系，检测异常偏离——当语义相似的输入产生嵌入空间中距离很远的输出时触发告警。

6. **自适应输入预处理**：根据 Agent 在不同扰动类型上的表现，动态调整预处理策略——如果 Agent 对拼写错误敏感度低于同义替换，则优先处理同义替换问题。
