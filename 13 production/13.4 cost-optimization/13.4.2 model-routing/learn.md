# 模型路由 (Model Routing) —— 智能分发请求以优化成本与质量

**模型路由是 LLM Agent 系统中根据任务复杂度、能力需求和成本预算，将查询智能分发到不同模型的核心优化技术，通常可在保持 95% 以上输出质量的前提下降低 30%-60% 的推理成本。**

---

## 1. 基本原理

### 1.1 为什么需要路由

不同 NLP 任务对模型能力的需求差异巨大。将简单任务交给昂贵的大模型，等于用火箭筒打蚊子；将复杂推理交给廉价小模型，又会得到不可靠的结果。

```
任务复杂度光谱：

  简单                             复杂
  ────────────────────────────────────────────────>
  分类    抽取    格式化    总结    推理    编码    规划
  │       │       │        │       │       │      │
  │       │       │        │       │       │      │
  ┌───────┴───────┴────────┘       └───────┴──────┴──────┐
  │ 廉价模型足以胜任                    │ 需要高端模型       │
  │ GPT-4o-mini / Haiku               │ GPT-4o / Sonnet / Opus
  │ $0.15/M input tokens              │ $3.00/M input tokens
  └────────────────────────────────────┴────────────────────┘
```

### 1.2 成本差异量化

主流模型之间的价格差距可达 10-60 倍：

```
模型价格对比（每百万输入 Token）:

  GPT-4o-mini      $0.15   ─█──────
  Claude Haiku     $0.25   ──█─────
  Gemini Flash     $0.35   ───█────
  GPT-4o           $2.50   ──────█─
  Claude Sonnet    $3.00   ───────█
  Claude Opus      $15.00  ────────█═══════════════

  成本比（最贵/最便宜）≈ 100 倍
```

如果将所有流量不加区分地路由到最贵的模型，成本将呈线性增长；但如果全部路由到最便宜的模型，复杂任务的准确率可能从 90% 骤降至 60% 以下。

### 1.3 精度-成本帕累托前沿

模型路由的核心目标是在帕累托前沿上找到最优工作点：

```
精度 (%)
 100 │                                    ◆ Opus
     │                               ◆ Sonnet
  95 │                         ◆ GPT-4o
     │                    ◆─── 路由策略 Pareto 前沿
  90 │               ◆ GPT-4o-mini
     │          ◆ Haiku
  85 │     ◆ Flash
     │
     └─────────────────────────────────── 成本 ($)
          低                    高
```

良好的路由策略应该使系统尽可能靠近"高精度、低成本"的左上角区域。

---

## 2. 背景与演进

### 2.1 路由策略的演进历程

```
时间线 ─────────────────────────────────────────────────>

Gen 1         Gen 2              Gen 3               Gen 4
─────    ─────────────     ───────────────     ──────────────
手动      规则路由            ML 路由             动态自适应
路由                                                 路由
│           │                   │                    │
│  ┌────────▼────────┐  ┌──────▼───────┐   ┌─────────▼────────┐
│  │ if "代码"→GPT-4 │  │ Embedding+   │   │ 实时质量监控     │
│  │ if "摘要"→Mini  │  │ 逻辑回归      │   │ + 成本反馈       │
│  │ 规则硬编码       │  │ 概率阈值      │   │ + A/B 测试       │
│  │ 无自动适应       │  │ 可扩展       │   │ 完全自适应       │
│  └─────────────────┘  └──────────────┘   └──────────────────┘
│
v0.1                 v0.5                  v1.0                v2.0
```

- **Gen 1 (手动路由)**: 开发者在代码中硬编码 if-else 分支，完全依赖经验判断
- **Gen 2 (规则路由)**: 基于 Keyword 匹配、任务类型检测、Prompt 长度阈值的可配置规则系统
- **Gen 3 (ML 路由)**: 使用 Prompt 嵌入向量 + 分类器（逻辑回归、随机森林）自动学习路由规则
- **Gen 4 (动态自适应)**: 实时监控模型输出质量，引入反馈闭环，自动调整路由阈值和策略

### 2.2 工业界的实践演进

```
早期 (2023)            中期 (2024)              当前 (2025-2026)
─────────────      ────────────────          ────────────────────
所有请求→GPT-4     简单任务→GPT-3.5          多级路由 + 级联
成本爆炸            复杂任务→GPT-4            实时成本感知路由
                   节省约 40% 成本           节省 50-70% 成本
                   规则硬编码                 ML 驱动 + 自适应
```

---

## 3. 核心技术与实现

### 3.1 路由系统架构总览

```
                           ┌──────────────────────┐
                           │     用户请求          │
                           │  (User Request)       │
                           └──────────┬───────────┘
                                      │
                                      ▼
                           ┌──────────────────────┐
                           │   路由器 (Router)     │
                           │   ┌──────────────┐   │
                           │   │ 分类引擎      │   │
                           │   │ Classifier    │   │
                           │   └──────┬───────┘   │
                           └──────────┼───────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                    ▼                 ▼                 ▼
          ┌─────────────────┐ ┌──────────────┐ ┌──────────────┐
          │  简单任务        │ │  复杂任务     │ │ 不确定任务    │
          │  Simple         │ │  Complex     │ │ Unknown      │
          └────────┬────────┘ └──────┬───────┘ └──────┬───────┘
                   │                │                │
                   ▼                ▼                ▼
          ┌─────────────────┐ ┌──────────────┐ ┌──────────────┐
          │ 廉价模型         │ │ 高端模型      │ │ 降级+日志     │
          │ Haiku/Mini      │ │ Sonnet/Opus  │ │ Fallback +   │
          │ $0.25/M tok     │ │ $3-15/M tok  │ │ Logging      │
          └────────┬────────┘ └──────┬───────┘ └──────┬───────┘
                   │                │                │
                   └────────────────┼────────────────┘
                                    │
                                    ▼
                           ┌──────────────────────┐
                           │   响应 + 监控指标      │
                           │  成本 / 延迟 / 质量    │
                           └──────────────────────┘
```

### 3.2 规则路由 (Rule-based Routing)

最简单的实现方案，基于可配置的规则集做决策：

```python
import re
from typing import Literal

ModelRoute = Literal["cheap", "expensive", "unknown"]

class RuleBasedRouter:
    """
    基于规则的模型路由器
    规则优先级: 关键词匹配 > 任务类型检测 > 长度阈值
    """

    def __init__(self):
        # 规则优先级: 高优先级规则优先匹配
        self.rules = [
            # (匹配模式, 路由目标, 描述)
            ("keyword", ["翻译", "总结", "摘要", "分类"], "cheap",
             "简单NLP任务"),
            ("keyword", ["代码", "debug", "调试", "优化", "架构"], "expensive",
             "代码相关任务"),
            ("keyword", ["规划", "设计", "分析", "策略", "决策"], "expensive",
             "复杂推理任务"),
            ("length", 500, "cheap",
             "短 prompt → 简单任务"),
            ("length", 3000, "expensive",
             "长 prompt → 复杂上下文"),
        ]

    def classify(self, prompt: str) -> ModelRoute:
        """基于规则分类请求"""
        prompt_lower = prompt.lower()
        prompt_len = len(prompt)

        for rule_type, pattern, target, _ in self.rules:
            if rule_type == "keyword":
                for kw in pattern:
                    if kw in prompt_lower:
                        return target
            elif rule_type == "length":
                if prompt_len < pattern:
                    # 短 prompt → 简单 (但不在更高优先级匹配中)
                    continue
                elif prompt_len >= pattern:
                    # 长 prompt → 复杂
                    pass  # 由更具体的规则决定

        # 默认降级: 记录日志并走中等路线
        print(f"[WARN] 无法分类 prompt (len={prompt_len}), 走降级路径")
        return "unknown"

    def route(self, prompt: str):
        route = self.classify(prompt)
        print(f"路由决策: {route}")
        return route


# 使用示例
router = RuleBasedRouter()
router.route("请帮我总结这篇文章")    # cheap
router.route("请帮我 debug 这个代码")  # expensive
router.route("天气怎么样")            # unknown → fallback
```

### 3.3 ML 路由 (Embedding + 分类器)

利用 prompt 的语义嵌入向量训练一个轻量级分类器，比规则路由更准确且可泛化：

```python
import numpy as np
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score
from typing import List, Tuple
import json

class MLRouter:
    """
    基于嵌入向量 + 逻辑回归的模型路由器

    流程:
    prompt → Embedding → 分类器 → 路由决策

    训练数据格式:
    [
        {"prompt": "总结这篇文章", "label": "cheap", "embedding": [...]},
        {"prompt": "设计一个微服务架构", "label": "expensive", "embedding": [...]},
    ]
    """

    def __init__(self, embedding_dim: int = 1536):
        self.classifier = LogisticRegression(
            C=1.0,
            max_iter=1000,
            class_weight="balanced",
        )
        self.embedding_dim = embedding_dim
        self.confidence_threshold = 0.75  # 低于此阈值走 fallback
        self.is_trained = False

    def _get_embedding(self, text: str) -> np.ndarray:
        """
        获取文本嵌入向量
        实际使用中可调用 OpenAI / Cohere / 本地模型 API
        """
        # 这里使用随机向量作为演示
        # 生产环境应替换为真实 Embedding API 调用
        return np.random.randn(self.embedding_dim)

    def train(self, training_data: List[dict]):
        """训练分类器"""
        X = []
        y = []

        for item in training_data:
            if "embedding" in item:
                emb = item["embedding"]
            else:
                emb = self._get_embedding(item["prompt"])
            X.append(emb)
            y.append(item["label"])

        X = np.array(X)
        y = np.array(y)

        self.classifier.fit(X, y)
        self.is_trained = True

        # 训练集精度评估
        y_pred = self.classifier.predict(X)
        acc = accuracy_score(y, y_pred)
        print(f"[ML Router] 训练完成，训练精度: {acc:.3f}")

    def classify_with_confidence(self, prompt: str) -> Tuple[str, float]:
        """返回 (路由目标, 置信度)"""
        if not self.is_trained:
            return ("unknown", 0.0)

        emb = self._get_embedding(prompt).reshape(1, -1)
        probs = self.classifier.predict_proba(emb)[0]
        pred = self.classifier.predict(emb)[0]
        confidence = float(np.max(probs))

        return (pred, confidence)

    def route(self, prompt: str) -> str:
        """路由决策，含置信度判断"""
        target, confidence = self.classify_with_confidence(prompt)

        if target == "unknown" or confidence < self.confidence_threshold:
            print(
                f"[ML Router] 低置信度 ({confidence:.3f}), "
                f"走 fallback 路径"
            )
            return "fallback"

        print(
            f"[ML Router] 路由到 {target} "
            f"(置信度: {confidence:.3f})"
        )
        return target


# 模拟训练数据准备
def prepare_training_data() -> List[dict]:
    """准备训练数据（生产环境需人工标注）"""
    return [
        # 简单任务样例
        {"prompt": "将这段文本翻译成英文", "label": "cheap"},
        {"prompt": "提取文本中的关键实体", "label": "cheap"},
        {"prompt": "给这段话写一个标题", "label": "cheap"},
        {"prompt": "判断这段评论是正面还是负面", "label": "cheap"},
        {"prompt": "格式化这段 JSON 数据", "label": "cheap"},
        # 复杂任务样例
        {"prompt": "分析这个分布式系统的性能瓶颈", "label": "expensive"},
        {"prompt": "设计一个支持千万级用户的聊天系统架构", "label": "expensive"},
        {"prompt": "解释 Quantum Entanglement 的数学原理", "label": "expensive"},
        {"prompt": "比较六种排序算法的时间空间复杂度", "label": "expensive"},
        {"prompt": "重构这个模块并优化其可扩展性", "label": "expensive"},
    ]


# 使用示例
router = MLRouter()
data = prepare_training_data()
router.train(data)

test_prompts = [
    "帮我翻译这段话",
    "设计一个微服务架构的限流方案",
    "今天天气怎么样",
]

for p in test_prompts:
    result = router.route(p)
    print(f"  Prompt: {p}\n  路由: {result}\n")
```

### 3.4 级联路由 (Cascade Routing)

先尝试廉价模型，通过质量检测判断是否需要升级到昂贵模型：

```python
import random
import time
from dataclasses import dataclass
from typing import Optional, Callable

@dataclass
class CascadeConfig:
    """级联路由配置"""
    cheap_model: str = "claude-3-haiku"
    expensive_model: str = "claude-3-sonnet"
    max_retries: int = 1
    quality_threshold: float = 0.7

class CascadeRouter:
    """
    级联路由: 先廉价 → 质量检测 → 不够好则升级

    流程:
    ┌────────┐    ┌──────────┐    ┌───────────┐
    │ 请求   │───▶│ Haiku    │───▶│ 质量检测   │
    └────────┘    └──────────┘    └─────┬─────┘
                                       │
                          ┌────────────┴────────────┐
                          │                         │
                          ▼                         ▼
                   ┌──────────┐             ┌────────────┐
                   │ 通过检测  │             │ 未通过检测  │
                   │ 直接返回  │             │ 升级到 Sonnet
                   └──────────┘             └────────────┘
    """

    def __init__(
        self,
        config: CascadeConfig = None,
        quality_checker: Optional[Callable] = None,
    ):
        self.config = config or CascadeConfig()
        self.quality_checker = quality_checker or self._default_quality_check
        self.metrics = {
            "total_requests": 0,
            "cheap_only": 0,
            "escalated": 0,
            "total_cost": 0.0,
        }

    def _call_model(self, model: str, prompt: str) -> str:
        """
        模拟模型调用
        生产环境应替换为真实 API 调用
        """
        print(f"  调用模型: {model}")
        time.sleep(0.1)  # 模拟延迟
        # 模拟输出: 复杂模型输出更长、质量更高
        if model == self.config.expensive_model:
            return f"[高质量响应] 针对'{prompt[:20]}...'的详细分析..."
        else:
            return f"[响应] 对'{prompt[:20]}...'的回答"

    def _default_quality_check(self, prompt: str, response: str) -> float:
        """
        默认质量检测函数
        返回 0.0 ~ 1.0 的质量分数

        实际应用中可:
        1. 用另一个廉价模型评估
        2. 使用启发式规则 (响应长度、关键词覆盖率等)
        3. 使用专用质量评估模型
        """
        # 模拟质量评分
        score = random.uniform(0.3, 0.9)
        print(f"  质量评分: {score:.3f}")
        return score

    def route(self, prompt: str) -> tuple[str, str, float]:
        """
        执行级联路由

        返回:
            (最终响应, 使用的模型, 总成本)
        """
        self.metrics["total_requests"] += 1
        costs = {
            self.config.cheap_model: 0.25,
            self.config.expensive_model: 3.00,
        }

        # Step 1: 调用廉价模型
        cheap_response = self._call_model(
            self.config.cheap_model, prompt
        )

        # Step 2: 质量检测
        quality = self.quality_checker(prompt, cheap_response)

        if quality >= self.config.quality_threshold:
            # 通过检测，直接返回
            self.metrics["cheap_only"] += 1
            self.metrics["total_cost"] += costs[self.config.cheap_model]
            print(f"  => 通过检测，使用 {self.config.cheap_model}")
            return (cheap_response, self.config.cheap_model, quality)

        # Step 3: 升级到昂贵模型
        self.metrics["escalated"] += 1
        self.metrics["total_cost"] += (
            costs[self.config.cheap_model] + costs[self.config.expensive_model]
        )
        print(f"  => 未通过 ({quality:.3f}), 升级到 {self.config.expensive_model}")

        expensive_response = self._call_model(
            self.config.expensive_model, prompt
        )
        return (expensive_response, self.config.expensive_model, quality)

    def print_stats(self):
        """打印统计信息"""
        total = self.metrics["total_requests"]
        if total == 0:
            return

        cheap_pct = self.metrics["cheap_only"] / total * 100
        escalated_pct = self.metrics["escalated"] / total * 100
        avg_cost = self.metrics["total_cost"] / total

        print(f"\n{'='*50}")
        print(f"级联路由统计:")
        print(f"  总请求数:       {total}")
        print(f"  廉价直达:       {self.metrics['cheap_only']} ({cheap_pct:.1f}%)")
        print(f"  升级处理:       {self.metrics['escalated']} ({escalated_pct:.1f}%)")
        print(f"  总成本:         ${self.metrics['total_cost']:.2f}")
        print(f"  平均成本/请求:  ${avg_cost:.4f}")
        print(f"{'='*50}")


# 模拟运行
print("级联路由模拟:")
router = CascadeRouter()

test_cases = [
    "将这段文字翻译成英文",
    "设计一个分布式数据库的分片策略",
    "提取这封邮件中的会议时间和地点",
    "分析这段代码的内存泄漏问题",
    "给这篇文章写一个简短的摘要",
]

for prompt in test_cases:
    print(f"\n请求: {prompt}")
    response, model, quality = router.route(prompt)

# 计算如果全用昂贵模型的成本对比
all_expensive_cost = len(test_cases) * 3.00
print(f"\n如果全部使用昂贵模型:")
print(f"  总成本: ${all_expensive_cost:.2f}")

router.print_stats()
```

### 3.5 LLM-as-Judge 路由

使用廉价模型作为"裁判"，让它判断当前请求的复杂度并决定由哪个模型处理：

```python
import json
from typing import TypedDict, Optional

class RoutingDecision(TypedDict):
    """路由决策的结构化输出"""
    target_model: str      # "cheap" | "expensive" | "fallback"
    complexity: str        # "simple" | "moderate" | "complex"
    confidence: float       # 0.0 ~ 1.0
    reasoning: str          # 决策理由

class LLMAsJudgeRouter:
    """
    使用 LLM 作为路由判断器

    原理: 用一个廉价模型来分析请求复杂度，
    然后决定是否调用昂贵模型。

    优势:
    - 能理解上下文语义，比规则路由更灵活
    - 支持结构化输出，决策可追溯
    - 可以给出置信度和理由

    劣势:
    - 增加一次额外模型调用 (约 $0.0003-0.001/次)
    - 增加延迟 (约 200-500ms)
    """

    ROUTER_PROMPT = """你是一个智能请求分类器。分析以下用户请求并决定
应该由哪种能力的 LLM 来处理。

分类标准：
- cheap: 翻译、分类、提取、格式化、简单问答（单步推理）
- expensive: 代码生成、架构设计、数学推理、策略分析、多步推理
- fallback: 模糊、跨领域、需要最新信息

请以 JSON 格式返回决策：
{
    "target_model": "cheap",
    "complexity": "simple",
    "confidence": 0.95,
    "reasoning": "这是一个简单的翻译任务"
}

用户请求: {prompt}
"""

    def __init__(self, judge_model: str = "claude-3-haiku"):
        self.judge_model = judge_model
        self.metrics = {
            "judge_calls": 0,
            "model_routes": {"cheap": 0, "expensive": 0, "fallback": 0},
        }

    def _call_judge(self, prompt: str) -> RoutingDecision:
        """
        调用裁判模型进行路由判断

        生产环境中替换为真实 API 调用:
        response = client.chat.completions.create(
            model=self.judge_model,
            messages=[{"role": "user",
                       "content": self.ROUTER_PROMPT.format(prompt=prompt)}],
            response_format={"type": "json_object"},
        )
        """
        self.metrics["judge_calls"] += 1

        # 模拟判断逻辑（生产环境由 LLM 执行）
        simple_keywords = ["翻译", "总结", "提取", "分类", "格式化"]
        complex_keywords = ["设计", "架构", "分析", "优化", "推理"]

        prompt_lower = prompt.lower()
        if any(kw in prompt_lower for kw in simple_keywords):
            decision = RoutingDecision(
                target_model="cheap",
                complexity="simple",
                confidence=0.92,
                reasoning="检测到简单任务关键词",
            )
        elif any(kw in prompt_lower for kw in complex_keywords):
            decision = RoutingDecision(
                target_model="expensive",
                complexity="complex",
                confidence=0.88,
                reasoning="检测到复杂推理任务关键词",
            )
        else:
            decision = RoutingDecision(
                target_model="fallback",
                complexity="moderate",
                confidence=0.5,
                reasoning="无法明确分类，走降级路径",
            )

        self.metrics["model_routes"][decision["target_model"]] += 1
        return decision

    def route(self, prompt: str) -> tuple[RoutingDecision, Optional[str]]:
        """
        执行路由决策

        返回:
            (决策信息, 模型响应)
        """
        decision = self._call_judge(prompt)

        print(f"\n路由决策:")
        print(f"  目标模型:  {decision['target_model']}")
        print(f"  复杂度:    {decision['complexity']}")
        print(f"  置信度:    {decision['confidence']:.2f}")
        print(f"  理由:      {decision['reasoning']}")

        # 根据决策调用对应模型
        if decision["target_model"] == "cheap":
            response = f"[Haiku 响应] 对'{prompt}'的简单处理结果"
        elif decision["target_model"] == "expensive":
            response = f"[Sonnet 响应] 对'{prompt}'的深度分析..."
        else:
            response = f"[降级处理] 请求已记录待人工审核: {prompt}"

        return decision, response

    def print_stats(self):
        """输出统计"""
        print(f"\n{'='*50}")
        print(f"LLM-as-Judge 路由统计:")
        print(f"  总判断调用: {self.metrics['judge_calls']}")
        print(f"  路由分布:")
        for model, count in self.metrics["model_routes"].items():
            pct = count / max(self.metrics["judge_calls"], 1) * 100
            print(f"    {model}: {count} ({pct:.1f}%)")
        print(f"{'='*50}")


# 使用示例
print("LLM-as-Judge 路由演示:")
router = LLMAsJudgeRouter()

test_prompts = [
    "将这段中文翻译成日语",
    "为电商平台设计一个推荐系统架构",
    "今天天气怎么样",
]

for prompt in test_prompts:
    decision, response = router.route(prompt)

router.print_stats()
```

### 3.6 动态自适应路由

引入实时反馈闭环，使路由策略可以自适应地调整：

```python
import statistics
from collections import defaultdict
from datetime import datetime, timedelta
from enum import Enum

class RoutingStrategy(Enum):
    RULE_BASED = "rule_based"
    ML_BASED = "ml_based"
    LLM_JUDGE = "llm_judge"
    CASCADE = "cascade"

class AdaptiveRouter:
    """
    动态自适应路由器

    核心机制:
    1. 实时收集模型质量指标
    2. 周期性评估路由策略效率
    3. 自动调整路由阈值和策略权重

    指标收集:
    ┌─────────────┐    ┌──────────────┐    ┌──────────────┐
    │ 请求处理     │───▶│ 质量评估      │───▶│ 指标存储     │
    └─────────────┘    └──────────────┘    └──────┬───────┘
                                                  │
    ┌─────────────┐    ┌──────────────┐           │
    │ 策略调整     │◀───│ 性能分析      │◀──────────┘
    └─────────────┘    └──────────────┘
    """

    def __init__(self, adaptation_interval: int = 100):
        self.adaptation_interval = adaptation_interval
        self.request_count = 0

        self.metrics_history = defaultdict(list)
        self.current_strategy = RoutingStrategy.CASCADE
        self.thresholds = {
            "quality_min": 0.6,       # 最低可接受质量
            "cost_max_per_task": 1.0, # 每任务最高成本 ($)
            "latency_max_ms": 2000,   # 最大可接受延迟 (ms)
        }

        # 各策略的历史表现
        self.strategy_performance = {
            s: {"total_cost": 0, "avg_quality": 0, "count": 0}
            for s in RoutingStrategy
        }

    def _simulate_request(self, prompt: str, strategy: RoutingStrategy):
        """模拟一次请求处理（生产环境替换为真实逻辑）"""
        import random

        # 模拟不同策略的性能差异
        if strategy == RoutingStrategy.RULE_BASED:
            cost = random.uniform(0.5, 2.0)
            quality = random.uniform(0.5, 0.9)
            latency = random.uniform(100, 500)
        elif strategy == RoutingStrategy.ML_BASED:
            cost = random.uniform(0.3, 1.5)
            quality = random.uniform(0.6, 0.95)
            latency = random.uniform(150, 600)
        elif strategy == RoutingStrategy.LLM_JUDGE:
            cost = random.uniform(0.4, 2.0)
            quality = random.uniform(0.7, 0.98)
            latency = random.uniform(300, 800)
        else:  # CASCADE
            cost = random.uniform(0.3, 3.0)
            quality = random.uniform(0.75, 0.99)
            latency = random.uniform(200, 1200)

        return {"cost": cost, "quality": quality, "latency": latency}

    def _adapt_strategy(self):
        """根据历史指标自适应调整策略"""
        print(f"\n{'='*50}")
        print(f"自适应调整 (第 {self.request_count} 次请求后)")

        best_strategy = self.current_strategy
        best_score = -float("inf")

        for strategy, perf in self.strategy_performance.items():
            if perf["count"] == 0:
                continue

            avg_cost = perf["total_cost"] / perf["count"]
            avg_quality = perf["avg_quality"] / perf["count"]

            # 综合评分: 质量 × 成本效率
            cost_efficiency = 1.0 / max(avg_cost, 0.01)
            score = avg_quality * cost_efficiency

            print(f"  {strategy.value}: "
                  f"质量={avg_quality:.3f}, "
                  f"成本=${avg_cost:.3f}, "
                  f"评分={score:.3f}")

            if score > best_score:
                best_score = score
                best_strategy = strategy

        # 如果当前策略不是最优，切换策略
        if best_strategy != self.current_strategy:
            print(f"  策略切换: {self.current_strategy.value} → "
                  f"{best_strategy.value}")
            self.current_strategy = best_strategy
            # 重置性能统计，避免历史拖累
            self.strategy_performance = {
                s: {"total_cost": 0, "avg_quality": 0, "count": 0}
                for s in RoutingStrategy
            }

        print(f"{'='*50}")

    def route(self, prompts: list[str]):
        """批量处理并自适应"""
        for prompt in prompts:
            self.request_count += 1

            # 执行路由
            result = self._simulate_request(prompt, self.current_strategy)

            # 记录指标
            perf = self.strategy_performance[self.current_strategy]
            perf["total_cost"] += result["cost"]
            perf["avg_quality"] += result["quality"]
            perf["count"] += 1

            print(f"请求 #{self.request_count}: "
                  f"策略={self.current_strategy.value}, "
                  f"成本=${result['cost']:.2f}, "
                  f"质量={result['quality']:.2f}")

            # 定期自适应调整
            if self.request_count % self.adaptation_interval == 0:
                self._adapt_strategy()

        return self.current_strategy


# 模拟自适应路由
print("动态自适应路由演示 (每 5 次请求调整一次):")
router = AdaptiveRouter(adaptation_interval=5)

# 模拟 20 次请求
test_prompts = [f"请求 #{i}" for i in range(20)]
router.route(test_prompts)

print(f"\n最终策略: {router.current_strategy.value}")
```

---

## 4. 路由策略对比

### 4.1 主要路由策略一览

```
策略             实现成本    准确率     延迟开销    维护成本    可解释性    适用场景
──────────────────────────────────────────────────────────────────────────────
规则路由          低       60-75%     <1ms       中         高         固定场景
ML 路由           中       75-90%     5-20ms     低         中         通用场景
LLM-as-Judge      低       80-92%     200-500ms  低         高         需要上下文理解
级联路由          中       85-95%     可变        低         高         质量优先
动态自适应        高       90-97%     可变        高         低         大规模生产

成本对比 (每 1000 次请求):

  规则路由        $250  ────████████████████
  ML 路由         $200  ────███████████
  LLM-as-Judge    $350  ────█████████████████████
  级联路由        $300  ────█████████████████
  动态自适应      $220  ────████████████
  全部昂贵模型    $3000 ────████████████████████████████████████████
  全部廉价模型    $250  ────████
```

### 4.2 路由指标定义

```
路由性能核心指标:

                                    TP (正确分配到廉价)
                   ┌───────────────┐
                   │ 实际简单任务    │── TN: 正确分配到昂贵
                   └───────────────┘

  Accuracy  = (TP + TN) / (TP + TN + FP + FN)
             路由决策正确率

  Cost_Savings = 1 - (实际总成本 / 全昂贵成本)
                成本节省比例

  Fallback_Rate = 不可分类请求 / 总请求
                 降级率（需人工干预）

  Avg_Latency = 路由判断时间 + 模型推理时间
               平均响应延迟

  Quality_Score = 用户反馈满意度 / 评估模型评分
                 输出质量评分
```

### 4.3 路由决策矩阵

```
                    实际应该使用廉价模型        实际应该使用昂贵模型
                    ─────────────────────     ─────────────────────
路由到廉价模型          ✅ 最优决策                 ❌ 质量损失
                       (TP)                      (FN: 假阴性)

路由到昂贵模型          ❌ 成本浪费                 ✅ 最优决策
                       (FP: 假阳性)                (TN)

路由到 fallback        ⚠️ 轻微效率损失             ⚠️ 延迟增加但安全
```

---

## 5. 能力边界

### 5.1 已知限制

```
已知问题                     影响                     缓解方案
────────────────────────────────────────────────────────────
1. 误分类复杂任务             用户获得低质量输出        级联 + 质量检测
   → 廉价模型无法胜任
   → 用户不满意

2. 路由开销                   每请求增加延迟           内联决策 + 缓存
   → 分类器本身需要时间
   → 级联链增加延迟

3. 冷启动问题                 初始路由准确率低         规则兜底 + 主动学习
   → ML 路由需要标注数据
   → 初始阶段无训练数据

4. 模型版本漂移               路由阈值失效             定期重训练 + 漂移检测
   → 模型升级后行为变化
   → 之前的路由规则不准确

5. 成本估算偏差               实际节省低于预期         实时成本追踪
   → 输入/输出 Token 比例变化
   → Cache Hit Rate 影响
```

### 5.2 路由误判案例分析

```
案例 1: 假阴性 (FN) — 将复杂任务判为简单
─────────────────────────────────────────
用户请求: "用 Python 实现一个红黑树，包含插入和删除操作"
路由结果: → Haiku (廉价模型)

Haiku 输出:
  class RBTree:
      pass  # 不完整的实现

后果:
  - 用户得到不可用的代码
  - 需要二次请求昂贵模型
  - 总成本: $0.25 (Haiku) + $3.00 (Sonnet) = $3.25
  - 比直接使用 Sonnet 更贵、更慢

案例 2: 假阳性 (FP) — 将简单任务判为复杂
─────────────────────────────────────────
用户请求: "将"Hello"翻译成中文"
路由结果: → Sonnet (昂贵模型)

Sonnet 输出:
  你好

后果:
  - 成本: $3.00 而不是 $0.25
  - 12 倍的成本浪费
  - 但用户不会察觉问题
```

### 5.3 模型升级导致的路由漂移

```
模型版本变更对路由的影响:

               训练期                        生产期 (模型升级后)
               ──────                       ─────────────────────
原始分类边界:  ───●───●───●───│───●───●───
              简单任务        │        复杂任务
                              │ 分类边界

模型升级后:    ───●───●───●───●───│───●───●───
              简单任务的区域扩大了   │
                             新分类边界

问题: 原来路由到 "复杂" 的请求现在被分为 "简单"
结果: 廉价模型处理了超出其能力范围的任务
```

### 5.4 与其它优化策略的对比

```
优化策略         成本节省   质量影响   实现难度   延迟影响   组合效果
────────────────────────────────────────────────────────────────
模型路由         30-60%    轻微       中        低-中      ★ 基础策略
Token 压缩        5-15%    轻微       低        低        与路由正交
Prompt 缓存      20-40%    无         低        低        与路由互补
请求批处理        10-25%   无         中        高        与路由互补
模型量化          40-60%   中等       高        低        替代而非互补

策略组合收益示意:

  单独路由:       ████████████████░░░░░░░░  40% 成本节省
  单独缓存:       ██████████░░░░░░░░░░░░░░  25% 成本节省
  路由 + 缓存:    ████████████████████░░░░  55% 成本节省 (叠加)
  路由 + 压缩:    ██████████████████░░░░░░  50% 成本节省 (部分重叠)
```

---

## 6. 工程优化方向

### 6.1 生产级路由系统架构

```
                      ┌──────────────────────────┐
                      │     API Gateway           │
                      │    负载均衡 + 认证         │
                      └────────────┬─────────────┘
                                   │
                      ┌────────────▼─────────────┐
                      │     预处理器 (Preprocessor) │
                      │    - 输入清洗             │
                      │    - Cache Lookup         │
                      │    - 敏感内容检测          │
                      └────────────┬─────────────┘
                                   │
                      ┌────────────▼─────────────┐
                      │       路由器 (Router)      │
                      │                          │
                      │  ┌────────────────────┐  │
                      │  │ 多级分类引擎         │  │
                      │  │                    │  │
                      │  │ 1. 规则匹配 (ns级)  │  │
                      │  │ 2. Embedding 分类   │  │
                      │  │    (5-20ms)        │  │
                      │  │ 3. LLM Judge       │  │
                      │  │    (200-500ms)     │  │
                      │  └────────────────────┘  │
                      │                          │
                      │  ┌────────────────────┐  │
                      │  │ 决策融合器           │  │
                      │  │ 加权投票 + 保底策略   │  │
                      │  └────────────────────┘  │
                      └────────────┬─────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
         ▼                         ▼                         ▼
  ┌─────────────┐          ┌──────────────┐          ┌──────────────┐
  │ 模型池 Tier1 │          │ 模型池 Tier2  │          │ 模型池 Tier3  │
  │             │          │              │          │              │
  │ Haiku       │          │ Sonnet       │          │ Opus/Fable   │
  │ Mini        │          │ GPT-4o       │          │ 专有模型      │
  │ Flash       │          │ 开源 70B+    │          │              │
  │ 开源 7-8B   │          │              │          │              │
  └──────┬──────┘          └──────┬───────┘          └──────┬───────┘
         │                        │                         │
         └────────────────────────┼─────────────────────────┘
                                  │
                         ┌────────▼────────┐
                         │    响应后处理器    │
                         │  - 质量验证       │
                         │  - 格式标准化     │
                         │  - 指标采集      │
                         └────────┬────────┘
                                  │
                         ┌────────▼────────┐
                         │    监控 + 反馈    │
                         │                  │
                         │  实时 Dashboard  │
                         │  - 成本/请求      │
                         │  - 平均质量分     │
                         │  - 延迟 P50/P99  │
                         │  - Fallback 率   │
                         └─────────────────┘
```

### 6.2 关键优化方向

```
短期 (1-2 周)              中期 (1-2 月)             长期 (3-6 月)
────────────────────    ────────────────────    ─────────────────────
□ 规则路由上线            □ ML 路由训练             □ 全自动自适应路由
□ 基础质量监控            □ 级联路由 + 质量检测     □ 多目标 Pareto 优化
□ 成本追踪 Dashboard      □ A/B 测试框架            □ 强化学习路由策略
□ Fallback 机制           □ 模型漂移检测            □ 跨 Provider 路由
```

### 6.3 生产部署检查清单

```
[ ] 路由决策延迟是否在 SLA 范围内？
    └── 如果超过 50ms，考虑预计算 + 缓存

[ ] 是否有足够的监控指标？
    └── 至少覆盖: 路由准确率、成本/请求、延迟 P50/P99、Fallback 率

[ ] Fallback 策略是否安全？
    └── 所有 unknown 请求应记录 + 人工审核 + 后续纳入训练

[ ] 路由策略是否有 A/B 测试通道？
    └── 新策略应在 10% 流量上验证后再全量

[ ] 是否有成本预算控制？
    └── 设置月度成本上限，超限后自动降级到廉价模型

[ ] 是否考虑 Cache 预热？
    └── 高频请求的路由决策做本地缓存 (LRU Cache)

[ ] 模型更新是否有灰度流程？
    └── 新版本模型先在 5% 流量上验证路由效果
```

### 6.4 路由缓存优化

```python
from functools import lru_cache
import hashlib
import time
from typing import Optional

class CachedRouter:
    """
    带缓存的路由器
    - 对相同或相似的 prompt 复用路由决策
    - 减少路由判断开销
    """

    def __init__(self, cache_size: int = 1000, ttl_seconds: int = 300):
        self.cache: dict[str, tuple[str, float]] = {}
        self.cache_size = cache_size
        self.ttl = ttl_seconds
        self.hits = 0
        self.misses = 0

    def _hash_prompt(self, prompt: str) -> str:
        """对 prompt 做语义 hash"""
        return hashlib.md5(prompt.encode()).hexdigest()

    def route(self, prompt: str) -> str:
        """带缓存的路由决策"""
        prompt_hash = self._hash_prompt(prompt)

        # Cache 命中检查
        if prompt_hash in self.cache:
            cached_result, timestamp = self.cache[prompt_hash]
            if time.time() - timestamp < self.ttl:
                self.hits += 1
                return cached_result
            else:
                # 过期
                del self.cache[prompt_hash]

        self.misses += 1

        # 执行真实路由 (此处简化为规则路由)
        simple_keywords = ["翻译", "总结", "提取", "分类"]
        is_simple = any(kw in prompt for kw in simple_keywords)
        result = "cheap" if is_simple else "expensive"

        # 写入缓存
        if len(self.cache) >= self.cache_size:
            # LRU 淘汰: 删除最早的条目
            oldest_key = next(iter(self.cache))
            del self.cache[oldest_key]

        self.cache[prompt_hash] = (result, time.time())

        return result

    def stats(self):
        total = self.hits + self.misses
        hit_rate = self.hits / total if total > 0 else 0
        return {
            "cache_hits": self.hits,
            "cache_misses": self.misses,
            "hit_rate": f"{hit_rate:.1%}",
            "cache_size": len(self.cache),
        }


# 使用示例
router = CachedRouter()
prompts = [
    "翻译这段话",   # 新 → miss
    "翻译这段话",   # 命中 → hit
    "翻译这段话",   # 命中 → hit
    "设计个架构",   # 新 → miss
]

for p in prompts:
    result = router.route(p)
    print(f"  prompt='{p}' → {result}")

print(f"\n缓存统计: {router.stats()}")
```

### 6.5 路由指标监控

```python
import json
from collections import deque
from dataclasses import dataclass, field
from typing import List

@dataclass
class RoutingRecord:
    """单次路由记录"""
    prompt_hash: str
    routed_to: str
    cost: float
    latency_ms: float
    quality_score: float
    timestamp: float
    fallback_used: bool

class MetricsCollector:
    """
    路由指标采集器
    用于实时监控和事后分析
    """

    def __init__(self, window_size: int = 1000):
        self.records: deque = deque(maxlen=window_size)
        self.window_size = window_size

    def record(self, record: RoutingRecord):
        """记录一次路由"""
        self.records.append(record)

    def summarize(self) -> dict:
        """生成摘要统计"""
        if not self.records:
            return {}

        n = len(self.records)
        costs = [r.cost for r in self.records]
        latencies = [r.latency_ms for r in self.records]
        qualities = [r.quality_score for r in self.records]
        fallback_count = sum(1 for r in self.records if r.fallback_used)

        # 路由分布
        route_dist = {}
        for r in self.records:
            route_dist[r.routed_to] = route_dist.get(r.routed_to, 0) + 1

        return {
            "period": f"最近 {n} 次请求",
            "total_cost": sum(costs),
            "avg_cost": sum(costs) / n,
            "avg_latency_ms": sum(latencies) / n,
            "p95_latency_ms": sorted(latencies)[int(n * 0.95)],
            "avg_quality": sum(qualities) / n,
            "fallback_rate": fallback_count / n,
            "route_distribution": route_dist,
            "cost_saved_ratio": "计算中（需要全昂贵模型基线）",
        }


# 模拟指标采集
collector = MetricsCollector(window_size=100)
for i in range(10):
    collector.record(RoutingRecord(
        prompt_hash=f"hash_{i}",
        routed_to="cheap" if i % 3 != 0 else "expensive",
        cost=0.25 if i % 3 != 0 else 3.00,
        latency_ms=200 if i % 3 != 0 else 800,
        quality_score=0.85 if i % 3 != 0 else 0.97,
        timestamp=time.time(),
        fallback_used=(i % 7 == 0),
    ))

stats = collector.summarize()
print(json.dumps(stats, indent=2, ensure_ascii=False))
```

---

## 总结

模型路由是 LLM Agent 生产部署中**成本效益最高的优化手段之一**。通过智能地将请求分发到合适能力层级的模型，可以在保持输出质量的前提下节省 30-60% 的推理成本。

**关键要点:**

1. **不要一刀切** — 不同任务需要不同能力层级的模型，混合使用是常态
2. **从规则开始，向 ML 演进** — 规则路由成本最低，ML 路由效果最好，动态自适应是终极目标
3. **始终保留 Fallback** — 任何路由策略都会有误判，级联机制和人工审核是安全网
4. **监控是核心** — 没有监控就无法优化，成本、质量、延迟三指标缺一不可
5. **与其他策略组合** — 路由 + 缓存 + 压缩的组合效果远优于单一优化

模型路由不是银弹，但它是构建大规模、成本可控的 LLM Agent 系统不可或缺的核心基础设施。
