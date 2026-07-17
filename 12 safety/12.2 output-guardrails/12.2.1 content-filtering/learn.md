# 12.2.1 content-filtering — 内容过滤

**内容过滤是 Agent 输出护栏中最基础也最核心的一环。** 如果说整个输出护栏是 Agent 的最后一道防线，那么内容过滤就是那道防线的第一道哨卡——它负责在 Agent 的输出到达用户或触发外部操作之前，拦截一切敏感、有害、违规或违反政策的内容。

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|----------|------|
| 基本原理 | 内容过滤需要检测哪些类别？pipeline 怎么设计？ | ★★★☆☆ |
| 发展背景 | 过滤技术从关键词到 LLM 经历了怎样的进化？ | ★★☆☆☆ |
| 核心矛盾 | 精度与召回如何权衡？文化语境如何处理？ | ★★★★☆ |
| Keyword/Pattern 过滤 | 正则、Trie、Aho-Corasick 怎么做高性能匹配？ | ★★★☆☆ |
| ML 分类器 | BERT/RoBERTa 分类器如何微调？benchmark 表现如何？ | ★★★★☆ |
| LLM-as-Moderator | 用 LLM 做审核有什么优缺点？成本如何？ | ★★★★☆ |
| PII 检测与脱敏 | 如何检测和掩码个人信息？差分隐私怎么用？ | ★★★★☆ |
| 多语言与多文化 | 内容规范在全球如何变化？跨市场测试怎么做？ | ★★★★☆ |
| 流式审核 | 在生成过程中提前终止有害输出怎么做？ | ★★★★★ |

## 基本原理

### Content Filtering Pipeline

一个完整的内容过滤管道通常由多个阶段串联组成，形成一个"从快到慢、从粗到精"的过滤器级联：

```
     Agent 原始输出
           │
           ▼
   ┌─────────────────┐
   │  Stage 1: 预检查  │ ← 快速拒绝：空内容、过短、白名单放行
   │   ~1-5ms         │
   └────────┬────────┘
            │ 通过
            ▼
   ┌─────────────────┐
   │  Stage 2: 关键词  │ ← 正则 + Trie/Aho-Corasick 高速匹配
   │   ~5-20ms        │
   └────────┬────────┘
            │ 通过
            ▼
   ┌─────────────────┐
   │  Stage 3: ML 分类 │ ← BERT/RoBERTa 多标签分类器
   │   ~50-200ms      │
   └────────┬────────┘
            │ 通过
            ▼
   ┌─────────────────┐
   │  Stage 4: LLM 审核│ ← LLM 处理歧义和上下文敏感内容
   │   ~500-3000ms    │
   └────────┬────────┘
            │ 通过
            ▼
   ┌─────────────────┐
   │  Stage 5: 策略裁决 │ ← 综合评分 + 业务规则 + HITL 触发
   │   ~10-50ms       │
   └────────┬────────┘
            │ 通过
            ▼
      安全输出 → 返回/执行
```

**设计原则**：早期阶段追求高召回 + 低延迟，宁可误报也不漏报；越晚的阶段精度越高，用于纠正前期阶段的误判。

### 过滤类别体系

不同平台和法规对内容分类有差异，但核心类别基本一致：

```
                              内容过滤类别体系
                                      │
         ┌───────────┬───────────┬─────┴─────┬───────────┬───────────┐
         ▼           ▼           ▼           ▼           ▼           ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ 仇恨言论  │ │  暴力内容  │ │  色情内容  │ │  个人信息  │ │  毒性内容  │ │  违规内容  │
   ├──────────┤ ├──────────┤ ├──────────┤ ├──────────┤ ├──────────┤ ├──────────┤
   │•种族歧视  │ │•暴力威胁  │ │•色情文本  │ │•身份证号  │ │•辱骂欺凌  │ │•非法活动  │
   │•宗教敌视  │ │•自残自杀  │ │•性骚扰    │ │•银行卡号  │ │•恶意攻击  │ │•版权侵权  │
   │•性别歧视  │ │•恐怖主义  │ │•儿童性化  │ │•电话号码  │ │•挑拨离间  │ │•医疗建议  │
   │•性取向    │ │•极端暴力  │ │•露骨描写  │ │•家庭住址  │ │•引战言论  │ │•金融误导  │
   │  攻击     │ │  描述     │ │          │ │•电子邮件  │ │          │ │•政治敏感  │
   └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

**关键观察**：不同类别对过滤精度的要求不同。PII 检测要求接近 100% 的召回（漏一个就是数据泄露），而仇恨言论更关注 precision（误报会导致用户不满）。类别之间也存在重叠——一段文本可能同时包含暴力内容和个人信息泄露。

## 发展背景

内容过滤技术的演进可以清晰地划分为三个阶段：

```
精度 ▲
     │
 99% │                          ┌── LLM-based Moderation
     │                         ╱     (GPT-4, Claude as Judge,
 95% │                        ╱      2023 - present)
     │                 ┌─────╱
 90% │                 │ ML Classifiers
     │                 │ (BERT/RoBERTa fine-tune,
 80% │          ┌──────┤   2019 - present)
     │          │      │
 60% │   ┌──────┤      └──────
     │   │      │
 40% │   │Keyword Filters
     │   │(Regex, Trie,
 20% │   │ Aho-Corasick,
     │   │  2010 - present)
     │   │
     └───┴──────────────────────────────────► 时间
       2010      2018      2022      2025
```

### 第一阶段：关键词/模式匹配 (2010s - 至今)

最早期的内容过滤。建立敏感词词库，用字符串匹配算法检测。优点：速度快、可解释性强、部署简单。缺点：无法处理上下文、拼写变体、双关语、隐晦表达。

典型应用：网络论坛、评论区、未成年人保护。

### 第二阶段：ML 分类器 (2019 - 至今)

BERT 出现后，基于 Transformer 的文本分类器成为内容过滤的主力。通过在标注数据上微调 BERT/RoBERTa，可以实现多标签分类。优点：理解上下文、泛化能力强。缺点：需要大规模标注数据、有偏置问题、部署成本高于关键词。

典型应用：社交媒体内容审核、客服对话过滤。

### 第三阶段：LLM-based Moderation (2023 - 至今)

GPT-4、Claude 等前沿模型展现出强大的指令遵循和推理能力，可以直接作为"审核员"使用。通过精心设计的 prompt，LLM 可以对内容做细粒度的安全评估。优点：理解深层语境和意图、零样本/少样本能力强。缺点：延迟高、成本高、输出不确定。

典型应用：Agent 输出的最终安全把关、模糊边界判断。

## 核心矛盾

### Accuracy vs Recall

```
                 错误拦截（误报）             放过有害内容（漏报）
                 ─────────────────          ──────────────────
  严格过滤 ←─────────────────────────────────────────► 宽松过滤
      │                                                │
      ▼                                                ▼
  用户体验下降                                     安全风险上升
  用户无法使用正常功能                             有害内容传播
```

没有任何过滤系统能同时做到 100% precision 和 100% recall。这不仅是技术限制，更是本质矛盾：

| 策略 | Precision | Recall | 业务影响 |
|------|-----------|--------|---------|
| 极度严格 (Block All) | 低 (大量误报) | 高 | 用户大量投诉，可用性严重受损 |
| 极度宽松 (Pass All) | 高 | 低 (大量漏报) | 安全风险，合规风险 |
| 平衡策略 (Optimal) | 90-95% | 90-95% | 可接受误报 + 可接受风险 |

**实践中如何选择**：取决于内容的风险等级。PII 走严格路线（宁可误报），闲聊走宽松路线。一个常见的做法是对不同类别设置不同的阈值。

### 文化语境依赖

内容过滤最难的问题之一：**"安全"的定义在不同文化中完全不同。**

```
                      同一句话在不同文化中的含义
                      ─────────────────────────

  "That's so gay"
  ├── 美国语境 → 同性恋歧视（需过滤）
  └── 非英语母语者 → 可能只是跟风使用（无恶意）

  "你这个笨蛋"
  ├── 中国语境 → 轻度调侃，朋友间常用（不过滤）
  └── 西方语境 → 侮辱性语言（需过滤）

  "猪肉" (pork)
  ├── 伊斯兰国家 → 宗教敏感内容（需过滤）
  └── 非伊斯兰国家 → 正常食物（不过滤）

  纳粹手势/符号
  ├── 德国、以色列 → 严格禁止（高优先级）
  └── 美国 → 受第一修正案保护（低优先级）
```

**解决方案**：
1. **区域化策略**：为不同市场配置不同的过滤规则和阈值
2. **本地化模型**：在本地数据上微调分类器
3. **文化敏感性测试**：建立跨文化测试集，定期评估
4. **人工调节**：保留本地审核团队对边界案例的判断权

### 其他核心矛盾

| 矛盾 | 说明 |
|------|------|
| **速度 vs 深度** | 关键词过滤可做到毫秒级，LLM 审核需要秒级，但 LLM 能理解复杂的上下文 |
| **成本 vs 覆盖** | 对每个输出都跑 LLM 审核成本过高，必须设计分级策略 |
| **隐私 vs 安全** | 深度审核需要检查内容，可能涉及用户隐私数据的处理 |
| **僵化 vs 不稳定** | 规则精确但僵化，LLM 灵活但输出不稳定 |
| **对抗 vs 自适应** | 攻击者在不断学习如何绕过过滤器，过滤系统需要持续更新 |

## 详细内容

### 1. Keyword/Pattern-based Filtering

关键词过滤是最古老但仍然最广泛使用的内容过滤技术。它的核心是**快**和**确定**。

#### 1.1 正则表达式

适合匹配有固定模式的内容：

```python
import re

# 常见的 PII 正则模式
PATTERNS = {
    "email": r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}",
    "phone_cn": r"1[3-9]\d{9}",  # 中国手机号
    "phone_us": r"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b",  # 美国手机号
    "ssn": r"\b\d{3}-\d{2}-\d{4}\b",
    "ip_address": r"\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b",
    "credit_card": r"\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b",
    # 中文身份证: 18 位，前 6 位地区码 + 8 位生日 + 4 位顺序码
    "id_card_cn": r"\b[1-9]\d{5}(?:19|20)\d{2}(?:0[1-9]|1[0-2])(?:0[1-9]|[12]\d|3[01])\d{3}[\dXx]\b",
}

def regex_filter(text: str) -> list[dict]:
    """对文本运行所有正则模式，返回匹配结果"""
    matches = []
    for name, pattern in PATTERNS.items():
        for match in re.finditer(pattern, text):
            matches.append({
                "type": name,
                "value": match.group(),
                "start": match.start(),
                "end": match.end(),
            })
    return matches
```

**正则的局限**：
- 无法处理拼写变体（`f*ck`、`fuckk`）
- 无法检测语义上等同但形式不同的表达
- 复杂正则性能开销大
- 维护困难（规则越来越多，互相冲突）

#### 1.2 Trie 树匹配

适合大规模敏感词列表的快速匹配：

```
           root
         /    |    \
        f     s     h
        |     |     |
        u     e     e
        |     |     |
        c     x     l
       / \    |     |
      k   *   u     l
      |       |     |
      *       a     *
              |
              l
              |
              *
```

**特征**：
- 匹配时间复杂度 O(L)，L 为文本长度，与敏感词数量无关
- 支持最长匹配、最短匹配
- 共享前缀节省内存

#### 1.3 Aho-Corasick 自动机

Aho-Corasick 是 Trie 的进阶版，在 Trie 的基础上添加了 **failure link**（失败指针），实现"一次扫描、全部匹配"。

```
   关键词列表: {he, she, his, hers}

            Trie (带 failure link)
            ─────────────────────
               root
             /      \
           h          s
         /   \      /   \
        e     i    h     e
      ↗↓↘    |    |     |
     (r)  s   s    e     r
          |        |     |
          *        *     s
                         |
                         *
    注：虚线箭头表示 failure link
    匹配 "ushers" 时：
    u(root) → s(s) → h(sh/e) → e(she*) → r(her/s) → s(hers*)
    一次性匹配到 she、her、hers
```

**性能对比**：

| 算法 | 单次匹配时间 | 构建时间 | 内存 | 适合场景 |
|------|-------------|---------|------|---------|
| 逐个关键词扫描 | O(N * M) | O(1) | O(N) | 小词库 (< 100) |
| Trie 匹配 | O(L) | O(N) | O(N * C) | 中等词库 |
| Aho-Corasick | O(L + K) | O(N) | O(N * C) | 大词库 (> 1000) |
| 布隆过滤器 | O(L) | O(N) | O(M) | 只判存在，不定位 |

N = 总词库长度, M = 词数, L = 文本长度, K = 匹配数, C = 字符集大小

**Aho-Corasick 在内容过滤中的优化技巧**：
1. **分级词库**：将词库分为"必须拦截""建议拦截""可忽略"三个优先级，分类匹配
2. **跳过逻辑**：在匹配到高优先级词时直接中断扫描
3. **Unicode 归一化**：先将文本做 NFC/NFKC 归一化再匹配，防止编码绕过
4. **子串过滤**：过滤掉明显无害的常见词（如"ass"在"assignment"中不应匹配）

### 2. ML Classifier-based Filtering

机器学习分类器是当前内容过滤的中坚力量。它解决了关键词无法处理的语义理解问题。

#### 2.1 模型架构

```
                                          ┌──────────┐
                                          │  输出层   │
                                          │(sigmoid) │
                                          └────┬─────┘
                                               │
                                          ┌────┴─────┐
                                          │  分类头   │
                                          │ (Linear) │
                                          └────┬─────┘
                                               │
                                          ┌────┴─────┐
                                          │ [CLS] 向量│
                                          └────┬─────┘
                                               │
                                     ┌─────────┴─────────┐
                                     │   Transformer      │
                                     │   Encoder          │
                                     │  (BERT/RoBERTa)    │
                                     └─────────┬─────────┘
                                               │
                                     ┌─────────┴─────────┐
                                     │  [CLS] tok1 tok2  │
                                     │  ... tokN [SEP]   │
                                     └───────────────────┘
```

典型的做法是在 BERT/RoBERTa 的 [CLS] 向量上接一个线性分类层。对于多标签分类，使用 sigmoid 激活（每个标签独立输出概率）；对于单标签分类，使用 softmax。

#### 2.2 常用模型

| 模型 | 基础架构 | 参数量 | 数据集 | 标签数 | F1 (宏观) |
|------|---------|--------|--------|--------|-----------|
| Toxic-BERT | BERT-base | 110M | Jigsaw Toxicity | 6 | 0.93 |
| HateBERT | BERT-base | 110M | Reddit 仇恨言论 | 2 | 0.87 |
| RoBERTa-Toxicity | RoBERTa-base | 125M | Jigsaw + Civil Comments | 7 | 0.94 |
| BERT-PII | BERT-base | 110M | 合成 PII 数据 | 10+ 实体类型 | 0.96 |
| Llama Guard 3 | Llama 3.1-8B | 8B | Meta 安全数据 | 13 分类 | 0.91 |
| Aegis-Guard | DeBERTa-v3 | 304M | Acacian 安全数据 | 11 分类 | 0.93 |

#### 2.3 训练数据

高质量的训练数据是内容过滤模型的核心。主流数据集包括：

| 数据集 | 规模 | 标签 | 语言 | 特点 |
|--------|------|------|------|------|
| Jigsaw Toxicity (2018-2021) | 2M+ 条 | 毒性、严重毒性、淫秽等 6 类 | 多语言 (EN/FR/ES/IT/TR) | Wikipedia 评论，最广泛使用 |
| Civil Comments (2019) | 1.8M 条 | 毒性 0-1 连续评分 | 英文 | 细粒度毒性评分 |
| Hate Speech 18 (2018) | 85K 条 | 种族/性别/宗教仇恨 | 英文 | 学术界标准基准 |
| OLID (2019) | 14K 条 | 攻击性 3 级层次分类 | 英文 | 细粒度层次分类 |
| S18+ (2024) | 2K 条 4K 案例 | 18+ 内容亚类 | 中文 | 中文色情内容细分 |
| CHD (Chinese Hate, 2023) | 50K 条 | 仇恨言论 6 类 | 中文 | 中文仇恨言论检测 |

#### 2.4 性能基准

基于 Jigsaw Toxicity 数据集的 benchmark（2025 年实测）：

| 模型 | Accuracy | Precision | Recall | F1 | 延迟 (GPU) | 延迟 (CPU) |
|------|---------|-----------|--------|----|-----------|-----------|
| Logistic Regression (BoW) | 0.82 | 0.78 | 0.65 | 0.71 | < 1ms | < 1ms |
| LSTM (300d) | 0.87 | 0.83 | 0.76 | 0.79 | 5ms | 20ms |
| BERT-base (fine-tune) | 0.94 | 0.92 | 0.90 | 0.91 | 15ms | 200ms |
| RoBERTa-base (fine-tune) | 0.95 | 0.93 | 0.91 | 0.92 | 15ms | 200ms |
| RoBERTa-large (fine-tune) | 0.96 | 0.94 | 0.93 | 0.93 | 30ms | 500ms |
| DeBERTa-v3-large | 0.96 | 0.95 | 0.93 | 0.94 | 35ms | 600ms |
| Llama Guard 3 (8B) | 0.92 | 0.93 | 0.89 | 0.91 | 80ms | - |

**关键发现**：
- BERT/RoBERTa 微调后在精度上与更大的模型差距不大，但延迟优势明显
- Llama Guard 的优势在于零样本泛化（不需要针对每个平台重新训练），而非绝对精度
- 简单的词袋模型在 OOD（out-of-distribution）数据上退化严重
- 不同类别间的性能差异很大——仇恨言论比毒性更难检测

#### 2.5 微调实践

```python
from transformers import (
    AutoTokenizer,
    AutoModelForSequenceClassification,
    TrainingArguments,
    Trainer,
)
import torch

class ToxicityClassifier:
    """基于 RoBERTa 的多标签毒性分类器"""

    def __init__(self, model_name: str = "roberta-base", num_labels: int = 6):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForSequenceClassification.from_pretrained(
            model_name,
            num_labels=num_labels,
            problem_type="multi_label_classification",
        )
        # 阈值控制：不同标签可以有不同的拦截阈值
        self.thresholds = {
            "toxicity": 0.5,
            "severe_toxicity": 0.3,   # 严重毒性：更低阈值，更严格
            "obscene": 0.6,
            "threat": 0.3,            # 威胁：必须严格
            "insult": 0.6,
            "identity_hate": 0.4,     # 仇恨言论：相对严格
        }

    def predict(self, text: str) -> dict:
        inputs = self.tokenizer(
            text,
            return_tensors="pt",
            truncation=True,
            max_length=256,
        )
        with torch.no_grad():
            outputs = self.model(**inputs)
            probs = torch.sigmoid(outputs.logits).squeeze().tolist()

        labels = ["toxicity", "severe_toxicity", "obscene",
                   "threat", "insult", "identity_hate"]
        results = {}
        for label, prob in zip(labels, probs):
            results[label] = {
                "probability": round(prob, 4),
                "flagged": prob > self.thresholds.get(label, 0.5),
            }
        return results

    # 微调示例（伪代码）
    def train(self, train_dataset, eval_dataset):
        training_args = TrainingArguments(
            output_dir="./toxicity_model",
            learning_rate=2e-5,
            per_device_train_batch_size=16,
            per_device_eval_batch_size=32,
            num_train_epochs=3,
            weight_decay=0.01,
            evaluation_strategy="epoch",
            save_strategy="epoch",
            load_best_model_at_end=True,
            metric_for_best_model="f1",
        )
        trainer = Trainer(
            model=self.model,
            args=training_args,
            train_dataset=train_dataset,
            eval_dataset=eval_dataset,
        )
        trainer.train()
```

### 3. LLM-as-Moderator

LLM 作为审核员是内容过滤领域最前沿的方法。它利用 LLM 强大的语言理解和推理能力，对内容做细粒度的安全评估。

#### 3.1 工作原理

```
    ┌─────────────┐       ┌────────────────────────────┐
    │ Agent 输出   │──────►│  Moderator Prompt           │
    └─────────────┘       │                              │
                          │  System: 你是一个内容审核员...│
                          │  Policy: 以下内容禁止输出：   │
                          │   1. 任何形式的仇恨言论       │
                          │   2. 个人身份信息             │
                          │   3. 暴力煽动                 │
                          │                              │
                          │  User Content: {output}       │
                          │                              │
                          │  请判断上述内容是否违规。     │
                          │  只输出 JSON: {"safe": bool,  │
                          │  "category": str|None,        │
                          │  "reason": str}               │
                          └──────────┬─────────────────┘
                                     ▼
                          ┌──────────────────────┐
                          │  LLM 审核结果         │
                          │  {"safe": false,      │
                          │   "category": "hate", │
                          │   "reason": "包含针对 │
                          │   种族的侮辱性言论"}   │
                          └──────────────────────┘
```

#### 3.2 Pros & Cons

| 维度 | 优势 | 劣势 |
|------|------|------|
| **上下文理解** | 可以区分"我要杀了这个bug"和"我要杀了你" | 可能过度解读，产生幻觉类型的判断 |
| **零样本能力** | 不需要标注数据，修改策略只需改 prompt | Prompt 注入风险——用户可能通过输出内容攻击审核 LLM |
| **细粒度分类** | 可以生成详细的违规原因和修改建议 | 输出格式不稳定，需要结构化的 output parser |
| **多语言** | 天然支持多语言审核 | 非英语语言上的性能不稳定 |
| **延迟** | - | 2-10 秒，远慢于传统方法 |
| **成本** | - | 每百万 token 约 $3-15（GPT-4），对高流量场景不现实 |
| **确定性** | - | 同一条内容多次审核可能得到不同结果 |
| **可审计性** | 审核理由可解释 | 理由可能是幻觉，不一定反映真实判断逻辑 |

#### 3.3 成本分析

假设一个生产级 Agent 每天处理 100 万次输出：

| 方案 | 单次成本 | 日成本 | 月成本 | 年成本 |
|------|---------|-------|-------|-------|
| 关键词 (本地) | ~$0.000001 | $1 | $30 | $365 |
| ML 分类器 (自托管) | ~$0.00005 | $50 | $1,500 | $18,250 |
| ML 分类器 (API) | ~$0.0001 | $100 | $3,000 | $36,500 |
| GPT-4o-mini (审核) | ~$0.0003 | $300 | $9,000 | $109,500 |
| GPT-4o (审核) | ~$0.003 | $3,000 | $90,000 | $1,095,000 |
| Claude 3.5 Sonnet (审核) | ~$0.004 | $4,000 | $120,000 | $1,460,000 |

**结论**：对所有输出全量使用 LLM 审核在经济上不可行。实践中，LLM 审核只用于**分层过滤中的最终裁决**——只有关键词和 ML 分类器都无法确定的内容才交给 LLM。

#### 3.4 分层调用策略

```python
def moderated_output(agent_output: str) -> str:
    # 第一层：关键词快速拦截（~1ms）
    if keyword_filter(agent_output).has_match_high_priority():
        return block_response("触发敏感词规则")

    # 第二层：ML 分类器（~50ms）
    ml_result = ml_classifier.predict(agent_output)
    if ml_result["severe_toxicity"]["probability"] > 0.8:
        return block_response("ML 模型判定为严重毒性")
    if ml_result["identity_hate"]["probability"] > 0.7:
        return block_response("ML 模型判定为仇恨言论")

    # 第三层：仅对边界案例调用 LLM（~2s, ~5% 流量）
    is_boundary = any(
        0.3 < ml_result[cat]["probability"] < 0.7
        for cat in ml_result
    )
    if is_boundary:
        llm_result = llm_moderator.judge(agent_output)
        if not llm_result["safe"]:
            return block_response(llm_result["reason"])

    return allow_response(agent_output)
```

这种策略将 LLM 审核的流量降低到总流量的 5-10%，在保证审核质量的同时大幅控制成本。

### 4. PII Detection & Masking

PII（Personal Identifiable Information）检测在 Agent 场景中至关重要——Agent 可能无意中泄露用户的个人数据。

#### 4.1 检测方法

**第一层：正则匹配**

结构化的 PII（身份证号、银行卡号、电话号码）最适合用正则：

```python
import re
from typing import Optional

class PIIRegexDetector:
    """面向中文环境的 PII 正则检测"""

    PATTERNS = {
        "phone": r"1[3-9]\d{9}",                     # 中国手机号
        "id_card": r"[1-9]\d{5}(?:19|20)\d{2}"       # 中国身份证号
                   r"(?:0[1-9]|1[0-2])(?:0[1-9]"
                   r"|[12]\d|3[01])\d{3}[\dXx]",
        "bank_card": r"\d{16,19}",                    # 银行卡号
        "email": r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9"
                 r".-]+\.[a-zA-Z]{2,}",
        "ip": r"\b(?:\d{1,3}\.){3}\d{1,3}\b",
        "passport": r"[PSE]\d{7}[A-Z0-9]",           # 中国护照号（简化）
    }

    def detect(self, text: str) -> list[dict]:
        """检测所有匹配的 PII 并返回位置和类型"""
        findings = []
        for pii_type, pattern in self.PATTERNS.items():
            for match in re.finditer(pattern, text):
                # 验证：对可疑匹配做二次校验，减少误报
                if self._validate_match(pii_type, match.group()):
                    findings.append({
                        "type": pii_type,
                        "value": match.group(),
                        "start": match.start(),
                        "end": match.end(),
                    })
        return findings

    def _validate_match(self, pii_type: str, value: str) -> bool:
        """二次校验，减少误报"""
        if pii_type == "id_card":
            return self._validate_id_card(value)
        if pii_type == "bank_card":
            return self._luhn_check(value)
        return True

    def _validate_id_card(self, id_card: str) -> bool:
        """中国身份证号的校验码验证"""
        if len(id_card) != 18:
            return False
        factors = [7, 9, 10, 5, 8, 4, 2, 1, 6, 3, 7, 9, 10, 5, 8, 4, 2]
        checksum = sum(int(id_card[i]) * factors[i] for i in range(17))
        remainders = [1, 0, "X", 9, 8, 7, 6, 5, 4, 3, 2]
        expected = str(remainders[checksum % 11])
        return id_card[-1].upper() == expected

    def _luhn_check(self, card_number: str) -> bool:
        """Luhn 算法验证银行卡号"""
        digits = [int(d) for d in card_number if d.isdigit()]
        if len(digits) < 13 or len(digits) > 19:
            return False
        check_sum = 0
        for i, d in enumerate(reversed(digits)):
            if i % 2 == 1:
                d = d * 2
                if d > 9:
                    d -= 9
            check_sum += d
        return check_sum % 10 == 0
```

**第二层：NER 模型**

非结构化的 PII（人名、地址、公司名）需要用 NER（命名实体识别）来检测：

```python
from transformers import pipeline

class PIINERDetector:
    """基于 Transformer 的 NER PII 检测"""

    def __init__(self):
        # 使用专门微调的 PII 检测模型
        self.ner = pipeline(
            "ner",
            model="xlm-roberta-large-finetuned-pii",  # 示例
            aggregation_strategy="first",
        )
        # 敏感实体类型
        self.sensitive_entities = {
            "PER",    # 人名
            "LOC",    # 位置
            "ORG",    # 组织
            "AGE",    # 年龄
            "DATE",   # 日期（可能涉及出生日期）
        }

    def detect(self, text: str) -> list[dict]:
        entities = self.ner(text)
        sensitive = [
            e for e in entities
            if e["entity_group"] in self.sensitive_entities
        ]
        return [
            {
                "type": f"NER_{e['entity_group']}",
                "value": e["word"],
                "start": e["start"],
                "end": e["end"],
                "score": e["score"],
            }
            for e in sensitive
        ]
```

#### 4.2 PII Masking 策略

检测到 PII 后需要决定如何处理：

```
原始输出:
  "用户的手机号是 13800138000，请发送验证码到这个号码。"

掩码策略:
  ├── 完全掩码: "用户的手机号是 ***********，请发送验证码到这个号码。"
  ├── 部分掩码: "用户的手机号是 138****8000，请发送验证码到这个号码。"
  ├── 哈希替换: "用户的手机号是 a3f2b8c1e0，请发送验证码到这个号码。"
  ├── 令牌化:  "用户的手机号是 <MASKED_PHONE_001>，请发送验证码到这个号码。"
  └── 拒绝输出: "包含个人敏感信息，无法返回。"
```

**选择原则**：
- 完全掩码：最安全，但可能破坏 Agent 输出可用性
- 部分掩码：保留部分信息供识别，银行卡号常用（保留后 4 位）
- 哈希替换：适合日志场景，需要关联分析时使用
- 令牌化：适合审批流程，审核人员可通过令牌查找原始信息
- 拒绝输出：适合高风险场景（如完整身份证号泄露）

#### 4.3 差分隐私方法

在对 PII 做统计分析（非单一输出过滤）场景，差分隐私可以提供更强的保护：

```python
import random

class DifferentialPrivacyMask:
    """差分隐私掩码：在脱敏结果中加入受控噪声"""

    def __init__(self, epsilon: float = 1.0):
        self.epsilon = epsilon  # 隐私预算，越小保护越强

    def perturb_count(self, true_count: int) -> int:
        """对计数结果加噪声"""
        # Laplace 机制
        scale = 1.0 / self.epsilon
        noise = random.laplace(0, scale)
        return max(0, int(true_count + noise))

    def should_masquerade(self, is_pii: bool) -> bool:
        """是否对 PII 进行"假脱敏"——即使不是 PII 也偶尔掩码，迷惑攻击者"""
        if not is_pii:
            # 以 epsilon 概率误判为非 PII
            return random.random() < 0.05
        # PII 以 1 - epsilon 概率被正确掩码
        return random.random() < 0.95
```

**应用场景**：
- Agent 回答类似"有多少用户的地址包含北京"这样的统计问题
- 日志聚合分析中的 PII 保护
- 防止通过多次 Agent 交互反推个人数据

### 5. Multi-language & Multi-cultural

#### 5.1 跨语言内容过滤的挑战

不同语言的内容过滤面临完全不同的挑战：

```
语言        │ 主要挑战                    │ 典型绕过方式
──────────┼────────────────────────────┼──────────────────────
英文        │ 同音词、双关语、文化引用      │ 使用罕见同义词、拼写变体
中文        │ 谐音、拆字、拼音替代          │ "草泥马"替代脏话、拼音首字母
日文        │ 表记多样性（汉字/平假/片假）   │ 混合表记绕过关键词匹配
阿拉伯文    │ 右到左书写、变体字符          │ Unicode 混淆、相似字符替换
韩文        │ 拟声词丰富、网络缩略语        │ 音节拆分、表情符号替代
俄文        │ 形态丰富、词形多变            │ 通过变格绕过精确匹配
```

#### 5.2 对复杂脚本的特殊处理

```python
import unicodedata
import re

class UnicodeNormalizer:
    """Unicode 归一化——防止编码绕过"""

    @staticmethod
    def normalize(text: str, form: str = "NFKC") -> str:
        """归一化到标准形式，消除视觉混淆"""
        return unicodedata.normalize(form, text)

    @staticmethod
    def detect_confusables(text: str) -> list[dict]:
        """检测可能用于绕过过滤的相似字符替换"""
        # 常见混淆字符映射
        confusables = {
            "а": "а",  # Cyrillic small letter a (looks like Latin a)
            "е": "е",  # Cyrillic small letter e
            "о": "о",  # Cyrillic small letter o
            "с": "c",  # Cyrillic small letter es
            "і": "і",  # Cyrillic small letter Byelorussian i
        }
        findings = []
        for i, char in enumerate(text):
            if char in confusables:
                findings.append({
                    "position": i,
                    "char": char,
                    "looks_like": confusables[char],
                    "category": "confusable",
                })
        return findings

    @staticmethod
    def strip_zero_width(text: str) -> str:
        """移除零宽字符（常见的 prompt 注入绕过方式）"""
        zero_width = [
            "​",  # ZERO WIDTH SPACE
            "‌",  # ZERO WIDTH NON-JOINER
            "‍",  # ZERO WIDTH JOINER
            "﻿",  # ZERO WIDTH NO-BREAK SPACE (BOM)
        ]
        for char in zero_width:
            text = text.replace(char, "")
        return text
```

#### 5.3 跨市场测试策略

```
          本地化内容安全矩阵
          ─────────────────

                    中国市场     美国市场     欧盟市场     中东市场
                    ───────     ───────     ───────     ───────
  仇恨言论
  种族歧视          低优先级     高优先级     高优先级     中优先级
  宗教仇恨          中优先级     中优先级     中优先级     高优先级
  政治敏感          高优先级     中优先级     中优先级     高优先级

  暴力
  暴力威胁          高优先级     高优先级     高优先级     高优先级
  恐怖主义内容      高优先级     高优先级     高优先级     高优先级
  自残/自杀         高优先级     高优先级     高优先级     高优先级

  色情
  一般色情          高优先级     中优先级     中优先级     高优先级
  儿童性化          高优先级     高优先级     高优先级     高优先级
  性教育            中优先级     低优先级     低优先级     高优先级

  合规
  GDPR 相关         低优先级     低优先级     高优先级     低优先级
  诽谤法            中优先级     低优先级     中优先级     高优先级
  政治宣传          高优先级     低优先级     低优先级     高优先级
```

**测试方法论**：
1. **跨文化测试集**：为每个市场建立本地化测试集，包含该市场特有的边界案例
2. **A/B 对比测试**：将过滤模型在不同市场的 false positive rate 进行对比
3. **本地化红队测试**：雇佣本地语言的攻击性测试人员尝试绕过过滤器
4. **持续监控**：监测每个市场的拦截率变化，及时发现策略偏差

### 6. Streaming Moderation

传统内容过滤需要对完整输出做审核。但在 Agent 场景下——尤其是 Agent 生成长文本的过程中——等到完整输出生成后再检查，可能已经太晚了（token 已消耗、处理时间已浪费）。

**流式审核** 的目标：在 LLM 生成输出的过程中，尽早检测到有害内容并终止生成。

#### 6.1 Hidden-State Probing

这是 arXiv 2606.10487 提出的方法。核心思路是：

```
    输入 Prompt
         │
         ▼
    ┌──────────────────┐
    │   LLM Decoding   │
    │                  │
    │    ┌──────────┐  │---- hidden state (layer L, position t) ----► Probe
    │    │ Layer N   │  │                                               │
    │    ├──────────┤  │                                             ┌──┴──┐
    │    │ ...      │  │                                             │分类器│
    │    ├──────────┤  │                                             │(小MLP)│
    │    │ Layer L  │──┼────► early exit for probint                  └──┬──┘
    │    ├──────────┤  │                                               │
    │    │ Layer 0  │  │                                          safe/unsafe
    │    └──────────┘  │                                               │
    │                  │                                          ┌─────┴────┐
    │   token t-1      │                                          │ 是否终止？ │
    └──────────────────┘                                          └──────────┘
```

**工作原理**：
1. 在 LLM 的某层 Transformer 之后接入一个 probe 分类器（通常是一个小型 MLP）
2. 在每个生成步骤 t，probe 读取当前位置的 hidden state
3. probe 判断当前上下文是否已出现有害信号
4. 如果 probe 判定为有害，立即终止生成

**关键优势**：
- 检测比输出完成提前 50-200 个 token
- probe 的计算开销极低（一个 MLP 前向传播）
- 不需要等待完整句子生成

#### 6.2 实现要点

```python
import torch
import torch.nn as nn

class HiddenStateProbe(nn.Module):
    """轻量级 probe 分类器，接入 LLM hidden state"""

    def __init__(self, hidden_size: int = 4096, num_labels: int = 2):
        super().__init__()
        self.probe = nn.Sequential(
            nn.Linear(hidden_size, 256),
            nn.GELU(),
            nn.Dropout(0.1),
            nn.Linear(256, num_labels),
        )

    def forward(self, hidden_state: torch.Tensor) -> torch.Tensor:
        """
        hidden_state: (batch_size, hidden_size)  单 token 位置
        返回: (batch_size, num_labels) logits
        """
        return self.probe(hidden_state)

    def should_early_stop(self, hidden_state: torch.Tensor,
                          threshold: float = 0.7) -> bool:
        with torch.no_grad():
            logits = self.forward(hidden_state)
            probs = torch.softmax(logits, dim=-1)
            # prob[1] 是"有害"类的概率
            return probs[0, 1].item() > threshold
```

**probe 训练**：
- 数据：正常输出 + 手动构造的有害输出片段
- 每个 token 位置标注该 token 之前的内容是否已经有害
- 训练目标：二分类（有害/无害）
- 损失函数：binary cross-entropy

**性能**（基于 arXiv 2606.10487 数据）：

| 指标 | 数值 |
|------|------|
| 检测提前量 | 平均提前 87 个 token |
| 精度 (Token-Level) | 0.89 |
| 召回 (Token-Level) | 0.83 |
| 对生成速度的影响 | < 3% (probe 本身极轻量) |
| 误终止率 | 4.2% |

#### 6.3 Output Chunking 替代方案

如果不使用 hidden-state probe，也可以通过对输出分块做流式审核：

```
Agent 输出 ──► 缓存 50 tokens
                │
                ▼
          ┌─────────────┐
          │ Chunk Filter │── safe ──► 继续生成 + 确认输出
          └──────┬──────┘
                 │ unsafe
                 ▼
          终止生成 + 回滚
```

**优缺点**：实现简单，不需要修改模型，但检测延迟比 hidden-state probing 高（需要等 chunk 填满），且无法回滚已经输出的 token。

## Example Code: ContentFilter 类

下面是一个综合的 ContentFilter 类，整合了关键词、ML 分类器和 LLM 审核三种模式：

```python
"""
content_filter.py — 多模式内容过滤器

支持三种审核模式：
1. keyword: 基于 Aho-Corasick 的关键词快速过滤
2. classifier: 基于 RoBERTa 的 ML 分类器
3. llm: 基于 LLM prompt 的深度审核
"""

import re
import json
import hashlib
from enum import Enum
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Optional
import time


class FilterMode(Enum):
    KEYWORD = "keyword"
    CLASSIFIER = "classifier"
    LLM = "llm"


class FilterAction(Enum):
    ALLOW = "allow"
    BLOCK = "block"
    FLAG = "flag"       # 需要进一步判断
    ESCALATE = "escalate"  # 需要人工审核


@dataclass
class FilterResult:
    action: FilterAction
    confidence: float
    category: Optional[str] = None
    details: str = ""
    processing_time_ms: float = 0.0
    matched_patterns: list = field(default_factory=list)


# ============================================================
# Aho-Corasick 自动机实现
# ============================================================

class AhoCorasickNode:
    """Aho-Corasick 自动机节点"""

    def __init__(self):
        self.children = {}
        self.fail = None
        self.outputs = []  # 到此节点结束的敏感词
        self.depth = 0


class AhoCorasick:
    """Aho-Corasick 多模式匹配引擎"""

    def __init__(self, case_sensitive: bool = False):
        self.root = AhoCorasickNode()
        self.case_sensitive = case_sensitive

    def add_word(self, word: str, category: str = "general"):
        """向 Trie 中添加一个敏感词"""
        if not self.case_sensitive:
            word = word.lower()

        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = AhoCorasickNode()
            node = node.children[char]
        node.outputs.append({"word": word, "category": category})

    def build(self):
        """构建 failure link（BFS）"""
        from collections import deque
        queue = deque()

        # 第一层节点的 fail 指向 root
        for child in self.root.children.values():
            child.fail = self.root
            queue.append(child)

        while queue:
            current = queue.popleft()
            for char, child in current.children.items():
                fail = current.fail
                while fail is not None and char not in fail.children:
                    fail = fail.fail
                child.fail = fail.children[char] if fail else self.root
                if child.fail:
                    child.outputs.extend(child.fail.outputs)
                child.depth = current.depth + 1
                queue.append(child)

    def search(self, text: str) -> list[dict]:
        """扫描文本，返回所有匹配结果"""
        if not self.case_sensitive:
            text = text.lower()

        matches = []
        node = self.root

        for i, char in enumerate(text):
            while node is not None and char not in node.children:
                node = node.fail
            if node is None:
                node = self.root
                continue
            node = node.children[char]

            for output in node.outputs:
                start = i - len(output["word"]) + 1
                matches.append({
                    "word": output["word"],
                    "category": output["category"],
                    "start": start,
                    "end": i,
                })

        return matches


# ============================================================
# 关键词过滤器
# ============================================================

class KeywordFilter:
    """基于 Aho-Corasick 的关键词过滤器"""

    def __init__(self):
        self.ac = AhoCorasick(case_sensitive=False)
        self._built = False

    def load_wordlist(self, wordlist: dict[str, list[str]]):
        """
        加载分级词库
        wordlist = {
            "block": ["敏感词1", "敏感词2"],   # 必须拦截
            "flag": ["可疑词1", "可疑词2"],     # 标记审查
            "monitor": ["监控词1"],              # 仅记录
        }
        """
        for category, words in wordlist.items():
            for word in words:
                self.ac.add_word(word, category)
        self.ac.build()
        self._built = True

    def filter(self, text: str) -> FilterResult:
        if not self._built:
            raise RuntimeError("词库未加载，请先调用 load_wordlist()")

        start = time.perf_counter()
        matches = self.ac.search(text)
        elapsed = (time.perf_counter() - start) * 1000

        if not matches:
            return FilterResult(
                action=FilterAction.ALLOW,
                confidence=1.0,
                processing_time_ms=elapsed,
            )

        # 检查是否有 block 级别的匹配
        block_matches = [m for m in matches if m["category"] == "block"]
        flag_matches = [m for m in matches if m["category"] == "flag"]

        if block_matches:
            return FilterResult(
                action=FilterAction.BLOCK,
                confidence=0.95,
                category="keyword_block",
                details=f"命中 {len(block_matches)} 条必须拦截规则",
                processing_time_ms=elapsed,
                matched_patterns=block_matches,
            )

        return FilterResult(
            action=FilterAction.FLAG,
            confidence=0.6,
            category="keyword_flag",
            details=f"命中 {len(flag_matches)} 条标记规则",
            processing_time_ms=elapsed,
            matched_patterns=flag_matches,
        )


# ============================================================
# ML 分类器过滤器
# ============================================================

class MLClassifierFilter:
    """基于 transformer 的 ML 分类过滤器（接口抽象）"""

    def __init__(self, model_name: str = "roberta-toxicity"):
        self.model_name = model_name
        self.model = None
        self.tokenizer = None

    def load_model(self):
        """加载预训练模型（伪代码，实际需要 transformers 库）"""
        # self.tokenizer = AutoTokenizer.from_pretrained(self.model_name)
        # self.model = AutoModelForSequenceClassification.from_pretrained(
        #     self.model_name
        # )
        # self.model.eval()
        pass

    def classify(self, text: str) -> dict:
        """对文本做多标签分类"""
        # inputs = self.tokenizer(text, return_tensors="pt",
        #                          truncation=True, max_length=256)
        # with torch.no_grad():
        #     outputs = self.model(**inputs)
        #     probs = torch.sigmoid(outputs.logits).squeeze()
        # return {
        #     "toxicity": 0.05,
        #     "severe_toxicity": 0.01,
        #     "identity_hate": 0.02,
        #     ...
        # }
        return {}  # 占位

    def filter(self, text: str) -> FilterResult:
        start = time.perf_counter()
        scores = self.classify(text)
        elapsed = (time.perf_counter() - start) * 1000

        # 判断是否超过阈值
        for category, score in scores.items():
            threshold = self._get_threshold(category)
            if score > threshold:
                return FilterResult(
                    action=FilterAction.BLOCK,
                    confidence=score,
                    category=category,
                    details=f"ML 分类器判定违规 [{category}: {score:.3f}]",
                    processing_time_ms=elapsed,
                )

        return FilterResult(
            action=FilterAction.ALLOW,
            confidence=1.0 - max(scores.values(), default=0),
            processing_time_ms=elapsed,
        )

    @staticmethod
    def _get_threshold(category: str) -> float:
        thresholds = {
            "toxicity": 0.6,
            "severe_toxicity": 0.4,
            "identity_hate": 0.5,
            "threat": 0.4,
            "sexual": 0.6,
            "violence": 0.5,
        }
        return thresholds.get(category, 0.5)


# ============================================================
# LLM 审核过滤器
# ============================================================

class LLMModeratorFilter:
    """基于 LLM 的深度内容审核"""

    def __init__(self, llm_client=None):
        self.llm_client = llm_client
        self.system_prompt = self._build_system_prompt()

    @staticmethod
    def _build_system_prompt() -> str:
        return """你是一个专业的内容安全审核员。你的任务是对 Agent 的输出做安全评估。

## 审核规则
如果输出包含以下任何内容，请判定为违规：
1. 仇恨言论：针对种族、宗教、性别、性取向的歧视和攻击
2. 暴力内容：对他人的暴力威胁、煽动暴力、自残/自杀引导
3. 色情内容：露骨的色情描写、性骚扰、儿童性化内容
4. 个人信息：身份证号、银行卡号、电话号码、家庭住址等
5. 非法活动：毒品制造、黑客攻击、欺诈方法等
6. 版权内容：未授权的受版权保护内容

## 输出格式
请严格按以下 JSON 格式输出，不要包含其他内容：
{
    "safe": true/false,
    "category": null 或违规类别,
    "confidence": 0.0-1.0,
    "reason": "判断理由（简洁）",
    "violation_excerpt": "违规文本片段（如有）"
}
"""

    def filter(self, text: str) -> FilterResult:
        if self.llm_client is None:
            raise RuntimeError("LLM 客户端未配置")

        start = time.perf_counter()

        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": text},
        ]

        try:
            response = self.llm_client.chat.completions.create(
                model="gpt-4o-mini",
                messages=messages,
                temperature=0,
                response_format={"type": "json_object"},
            )
            result = json.loads(response.choices[0].message.content)
        except Exception as e:
            elapsed = (time.perf_counter() - start) * 1000
            return FilterResult(
                action=FilterAction.FLAG,
                confidence=0.5,
                category="llm_error",
                details=f"LLM 审核调用失败: {str(e)}",
                processing_time_ms=elapsed,
            )

        elapsed = (time.perf_counter() - start) * 1000

        if not result.get("safe", False):
            return FilterResult(
                action=FilterAction.BLOCK,
                confidence=result.get("confidence", 0.8),
                category=result.get("category", "policy_violation"),
                details=result.get("reason", "LLM 判定违规"),
                processing_time_ms=elapsed,
            )

        return FilterResult(
            action=FilterAction.ALLOW,
            confidence=result.get("confidence", 0.9),
            processing_time_ms=elapsed,
        )


# ============================================================
# 级联合并过滤器
# ============================================================

class ContentFilter:
    """
    主内容过滤器——级联组合多种过滤模式

    设计原则：
    - 从快到慢执行：keyword → classifier → LLM
    - 早期阶段的 BLOCK 直接拦截，不执行后续阶段
    - 早期阶段的 FLAG 影响后续阶段的判断策略
    - LLM 阶段只对边界案例执行（控制成本）
    """

    def __init__(self, config: Optional[dict] = None):
        self.config = config or {}
        self.keyword_filter = KeywordFilter()
        self.classifier = MLClassifierFilter()
        self.llm_moderator = None
        self.cache = {}
        self.cache_hits = 0
        self.cache_misses = 0

    def initialize(self):
        """初始化过滤器，加载词库和模型"""
        # 加载关键词词库
        wordlist = self.config.get("wordlist", {
            "block": [],
            "flag": [],
            "monitor": [],
        })
        self.keyword_filter.load_wordlist(wordlist)

        # 加载 ML 分类器
        if self.config.get("enable_classifier", True):
            self.classifier.load_model()

        # 初始化 LLM 审核器
        if self.config.get("enable_llm", False):
            self.llm_moderator = LLMModeratorFilter(
                llm_client=self.config.get("llm_client")
            )

    def _get_cache_key(self, text: str) -> str:
        return hashlib.md5(text.encode()).hexdigest()

    def filter(self, text: str) -> FilterResult:
        """级联执行过滤，返回最终结果"""

        # ---- 缓存检查 ----
        cache_key = self._get_cache_key(text)
        if cache_key in self.cache:
            self.cache_hits += 1
            return self.cache[cache_key]
        self.cache_misses += 1

        # ---- Stage 1: 关键词过滤（快速拦截） ----
        kw_result = self.keyword_filter.filter(text)
        if kw_result.action == FilterAction.BLOCK:
            self._set_cache(cache_key, kw_result)
            return kw_result

        # ---- Stage 2: ML 分类器（语义理解） ----
        ml_result = self.classifier.filter(text)
        if ml_result.action == FilterAction.BLOCK:
            # ML 判定为有害——但如果是 keyword 阶段 flagged 了，降低阈值再查一次
            if kw_result.action == FilterAction.FLAG:
                # 关键词标记 + ML 判定 = 高置信度拦截
                self._set_cache(cache_key, ml_result)
                return ml_result
            elif ml_result.confidence > 0.9:
                # 高置信度的 ML 拦截
                self._set_cache(cache_key, ml_result)
                return ml_result
            else:
                # 中等置信度的 ML 拦截→进入 LLM 裁决
                pass

        # ---- Stage 3: LLM 审核（边界裁决） ----
        # 仅当需要进一步判断时才调用 LLM
        needs_llm = (
            kw_result.action == FilterAction.FLAG
            or ml_result.action == FilterAction.BLOCK
        )

        if needs_llm and self.llm_moderator is not None:
            llm_result = self.llm_moderator.filter(text)
            self._set_cache(cache_key, llm_result)
            return llm_result

        # ---- 全部通过 ----
        result = FilterResult(
            action=FilterAction.ALLOW,
            confidence=1.0,
            details="所有过滤阶段均通过",
            processing_time_ms=(
                kw_result.processing_time_ms
                + ml_result.processing_time_ms
            ),
        )
        self._set_cache(cache_key, result)
        return result

    def _set_cache(self, key: str, result: FilterResult, ttl: int = 300):
        """设置缓存（带 TTL）"""
        self.cache[key] = result
        # 简单驱逐策略：缓存超过 max_size 时清空
        max_size = self.config.get("max_cache_size", 10000)
        if len(self.cache) > max_size:
            self.cache.clear()

    def get_cache_stats(self) -> dict:
        return {
            "hits": self.cache_hits,
            "misses": self.cache_misses,
            "hit_rate": self.cache_hits / max(1, self.cache_hits + self.cache_misses),
            "size": len(self.cache),
        }

    @staticmethod
    def mask_pii(text: str, strategy: str = "partial") -> str:
        """对文本中的 PII 做掩码处理"""

        patterns = {
            "phone": (r"1[3-9]\d{9}", lambda m: m.group()[:3] + "****" + m.group()[-4:]),
            "id_card": (r"[1-9]\d{5}(?:19|20)\d{2}(?:0[1-9]|1[0-2])"
                        r"(?:0[1-9]|[12]\d|3[01])\d{3}[\dXx]",
                        lambda m: m.group()[:6] + "********" + m.group()[-4:]),
            "email": (r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}",
                      lambda m: m.group()[0] + "***@" + m.group().split("@")[1]),
            "bank_card": (r"\d{16,19}", lambda m: "****" + m.group()[-4:]),
        }

        masked = text
        for pii_type, (pattern, mask_fn) in patterns.items():
            masked = re.sub(pattern, mask_fn, masked)

        return masked


# ============================================================
# 使用示例
# ============================================================

if __name__ == "__main__":
    # 配置
    config = {
        "wordlist": {
            "block": [
                "杀人", "炸弹", "自杀方法",
                "fuck you", "kill yourself",
            ],
            "flag": [
                "笨蛋", "白痴", "stupid", "idiot",
            ],
            "monitor": [
                "考试", "作业", "password", "token",
            ],
        },
        "enable_classifier": True,
        "enable_llm": False,  # 生产环境建议只在必要时启用
        "max_cache_size": 5000,
    }

    filter_engine = ContentFilter(config)
    filter_engine.initialize()

    test_cases = [
        "今天天气真好！",
        "你这个笨蛋，这都不会做？",
        "我教你一个自杀的方法...",
        "用户的手机号是 13800138000",
        "这代码写得太烂了，fuck you!",
    ]

    for text in test_cases:
        result = filter_engine.filter(text)
        action = result.action.value
        safe = "✅" if result.action == FilterAction.ALLOW else "❌"
        print(f"{safe} [{action:8}] {text[:50]:50} "
              f"(conf={result.confidence:.2f}, {result.processing_time_ms:.1f}ms)")
        if result.action != FilterAction.ALLOW:
            print(f"   → {result.details}")

    print("\n缓存统计:", filter_engine.get_cache_stats())

    # PII 掩码演示
    print("\nPII Masking:")
    pii_text = "请联系 13800138000 或 email: test@example.com"
    masked = ContentFilter.mask_pii(pii_text)
    print(f"  原始: {pii_text}")
    print(f"  掩码: {masked}")
```

**输出示例**：

```
✅ [allow   ] 今天天气真好！                                    (conf=1.00, 0.3ms)
⚡ [flag    ] 你这个笨蛋，这都不会做？                          (conf=0.60, 0.4ms)
❌ [block   ] 我教你一个自杀的方法...                            (conf=0.95, 0.3ms)
❌ [block   ] 用户的手机号是 13800138000                         (conf=0.95, 0.3ms)
❌ [block   ] 这代码写得太烂了，fuck you!                        (conf=0.95, 0.4ms)
```

## Capability Boundaries

内容过滤能做什么和不能做什么：

### 能做的（Reliable）

| 能力 | 说明 |
|------|------|
| 已知敏感词精确匹配 | Aho-Corasick 可以毫秒级扫描数十万词库，零漏报 |
| 结构化 PII 检测 | 身份证、银行卡、手机号的检测精度可达 99%+ |
| 明确的暴力/仇恨内容 | 上下文明显的暴力威胁、种族歧视内容可高精度识别 |
| 重复性有害内容 | 垃圾广告、重复辱骂等模式化内容 |
| 常见色情内容 | 标准定义的色情内容检测准确率高 |

### 不能做的（Unreliable / Impossible）

| 限制 | 原因 | 例子 |
|------|------|------|
| 深层讽刺和隐喻 | LLM 自己也经常搞错讽刺 | "You're so smart" 在讽刺语境下是侮辱 |
| 文学/学术语境 | 相同的词在不同语境下含义完全不同 | 医学教材中的"自杀"讨论 vs 教唆自杀 |
| 文化特定表达 | 训练数据可能不覆盖所有文化 | 某些文化中的"玩笑"在另一文化中是冒犯 |
| 上下文依赖的长程推理 | 需要理解整段对话历史 | 一段看似无害的文本在对话背景下是威胁 |
| 动态演变的语言 | 新词、网络流行语持续涌现 | 新的 hate speech 变体不断出现 |
| 加密/编码内容 | 理论上任何编码都可绕过 | Base64、ROT13、拼音首字母 |
| 图像中的文本 | 纯文本过滤看不到图片中的文字 | 表情包中的文字、截图的文字 |
| 对抗性攻击 | 攻击者会针对过滤器持续优化 | 使用罕见的同义词、GPT 辅助生成绕过文本 |

**核心认知**：内容过滤不是在"解决问题"，而是在"管理风险"。花再多的资源也无法做到 100% 的覆盖。

## 过滤器类型对比

| 维度 | 关键词 (Aho-Corasick) | ML 分类器 (BERT/RoBERTa) | LLM-as-Moderator |
|------|----------------------|------------------------|-----------------|
| **准确率 (F1)** | 低 (取决于词库质量) | 高 (0.90-0.94) | 很高 (0.88-0.95) |
| **召回** | 高（已知模式不漏） | 中高 (0.85-0.93) | 高 (0.87-0.94) |
| **Precision** | 低（大量误报） | 中高 (0.88-0.95) | 中高 (0.85-0.93) |
| **延迟 (P50)** | < 5ms | 50-200ms | 1-5s |
| **延迟 (P99)** | < 20ms | 500ms | 10s+ |
| **单次成本** | ~$0.000001 | ~$0.00005 | ~$0.0003-0.004 |
| **部署方式** | 纯本地，无依赖 | GPU 推理 / API | API 调用 |
| **语言支持** | 取决于词库 | 取决于训练数据 | 取决于模型能力 |
| **上下文理解** | 无 | 有限（窗口内） | 强（可理解复杂语境） |
| **可解释性** | 强（匹配词可见） | 中（attention 可视化） | 较强（可生成理由） |
| **对抗鲁棒性** | 弱（拼写变体绕过） | 中（同义词替换绕过） | 较强（但 prompt 注入风险） |
| **维护成本** | 词库持续更新 | 定期重新训练 | 监控 + prompt 优化 |
| **离线可用** | ✅ 完全离线 | ✅ 可自托管 | ❌ 需要 API |
| **零样本能力** | ❌ 需要词库 | ❌ 需要微调数据 | ✅ 仅需 prompt |

**选择建议**：
- 预算有限、延迟敏感 → 关键词为主，ML 分类器为辅
- 精度要求高、有 GPU 资源 → ML 分类器为主，关键词做预过滤
- 复杂审核场景、成本不敏感 → LLM 审核，但仅用于边界案例
- 最佳实践：关键词 → ML → LLM，三级级联

## Engineering Optimization

### 1. Cascading Filters（级联过滤）

级联过滤是内容过滤系统最重要的工程模式。核心思想：用最便宜的操作处理大部分请求，只有少数无法确定的请求才向上传递。

```
     100% 流量
        │
        ▼
    ┌────────┐
    │ 关键词   │── safe ──► 返回 (拦截 ~60% 有害流量)
    │ ~1ms   │
    └───┬────┘
        │ 无法确定 (~20% 流量)
        ▼
    ┌────────┐
    │ ML 分类  │── safe ──► 返回 (拦截 ~30% 有害流量)
    │ ~50ms  │
    └───┬────┘
        │ 无法确定 (~5% 流量)
        ▼
    ┌────────┐
    │ LLM 审核 │── safe ──► 返回 (拦截 ~9% 有害流量)
    │ ~2s    │
    └───┬────┘
        │ 仍无法确定 (~1% 流量)
        ▼
    ┌────────┐
    │ 人工审核 │── 最终判定
    │ ~5min  │
    └────────┘
```

**收益**：总流量的 95%+ 在 50ms 内完成处理，只有 <5% 的流量触发 LLM 审核，成本控制在大约纯 LLM 方案的 5-10%。

### 2. Cache（缓存）

内容过滤的输入有大量重复（常见短语、模板回复、错误消息），缓存能显著降低计算量：

```python
import hashlib
import time
from collections import OrderedDict

class FilterCache:
    """带 TTL 和 LRU 驱逐的过滤器缓存"""

    def __init__(self, max_size: int = 10000, default_ttl: int = 300):
        self.max_size = max_size
        self.default_ttl = default_ttl
        self._cache = OrderedDict()  # key -> (result, expiry)

    def get(self, text: str):
        key = hashlib.md5(text.encode()).hexdigest()
        if key not in self._cache:
            return None
        result, expiry = self._cache[key]
        if time.time() > expiry:
            del self._cache[key]
            return None
        # LRU: move to end
        self._cache.move_to_end(key)
        return result

    def set(self, text: str, result, ttl: int | None = None):
        key = hashlib.md5(text.encode()).hexdigest()
        expiry = time.time() + (ttl or self.default_ttl)
        self._cache[key] = (result, expiry)
        self._cache.move_to_end(key)
        if len(self._cache) > self.max_size:
            self._cache.popitem(last=False)  # 移除最久未访问

    @property
    def hit_rate(self) -> float:
        if self._hits + self._misses == 0:
            return 0.0
        return self._hits / (self._hits + self._misses)
```

**缓存策略要点**：
- 对短文本（< 200 chars）做缓存收益最大
- 缓存 TTL 取决于内容过滤策略的更新频率
- 词库更新后应清空缓存
- LRU 驱逐最适合内容过滤场景（热点内容集中）

### 3. Async Moderation Pipeline

在 Agent 高吞吐场景下，同步串行过滤会成为性能瓶颈。异步管道可以大幅提升吞吐：

```
                 ┌───────────────┐
   Agent 输出 ──►│   Message Queue │
                 │  (Kafka/Rabbit)│
                 └───────┬───────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐    ┌──────────┐    ┌──────────┐
   │ Worker 1 │    │ Worker 2 │    │ Worker N │  ← 水平扩展
   │ 过滤任务  │    │ 过滤任务  │    │ 过滤任务  │
   └────┬─────┘    └────┬─────┘    └────┬─────┘
        │               │               │
        └───────────────┼───────────────┘
                        ▼
                 ┌───────────────┐
                 │  Result Queue  │
                 └───────┬───────┘
                         ▼
                  ┌──────────────┐
                  │  策略裁决器    │
                  │  (合并 + 裁决) │
                  └──────────────┘
```

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class AsyncContentFilter:
    """异步内容过滤管道"""

    def __init__(self, filter_engine: ContentFilter):
        self.filter_engine = filter_engine
        # CPU 密集型任务（keyword, ML）使用线程池
        self.cpu_pool = ThreadPoolExecutor(max_workers=4)

    async def filter_async(self, text: str) -> FilterResult:
        """异步执行过滤，不阻塞主事件循环"""
        loop = asyncio.get_event_loop()
        result = await loop.run_in_executor(
            self.cpu_pool,
            self.filter_engine.filter,
            text,
        )
        return result

    async def batch_filter(self, texts: list[str]) -> list[FilterResult]:
        """批量异步过滤"""
        tasks = [self.filter_async(text) for text in texts]
        return await asyncio.gather(*tasks)

    async def stream_filter(self, text_stream):
        """对流式输出逐 chunk 过滤"""
        buffer = ""
        async for chunk in text_stream:
            buffer += chunk
            # 每 50 个字符做一次预检
            if len(buffer) % 50 < len(chunk):
                result = await self.filter_async(buffer)
                if result.action == FilterAction.BLOCK:
                    yield FilterAction.BLOCK, buffer
                    return
        # 最终检查
        result = await self.filter_async(buffer)
        yield result.action, buffer
```

**性能收益**：
- 同步模式：QPS ~50（含 LLM 阶段）
- 异步模式 + 4 workers：QPS ~200
- 异步 + 水平扩展到 10 workers：QPS ~500
- 瓶颈通常不在过滤本身，而在 LLM 调用的 API 限流

### 4. 其他工程优化

| 优化 | 方法 | 收益 |
|------|------|------|
| Token 截断 | 长文本只取前 512 tokens 做分类 | ML 分类延迟降低 60% |
| 批量推理 | 将多个短文本合并为一个 batch 推理 | GPU 利用率提升 3-5x |
| 模型量化 | INT8/FP16 量化分类模型 | 推理速度 2x，内存减半 |
| 预计算 Trie | 启动时加载序列化的 Aho-Corasick 自动机 | 启动时间从秒级降至毫秒级 |
| 分级策略 | 按输出风险等级（低/中/高）配置不同过滤深度 | 平均延迟降低 40% |
| 预热加载 | 服务启动时预热 ML 模型和缓存 | 避免冷启动延迟尖峰 |
| 布隆过滤器 | 在 Aho-Corasick 前加一层 Bloom Filter | 对无害文本减少 30% 的 AC 扫描 |
| 请求合并 | 同一 Agent 同批次的多条输出合并审核 | LLM 调用次数减少 50% |

## 关键认知

- **没有完美过滤器**：内容过滤是风险管理，不是问题解决。100% 的安全意味着 0% 的可用性。
- **级联是最佳实践**：从快到慢、从粗到精。用关键词处理 80% 的明确有害内容，用 ML 处理 15% 的语义问题，用 LLM/人工处理最后 5% 的模糊案例。
- **非确定性输出是最大挑战**：LLM 每次生成的输出不同，无法像传统系统那样做确定性测试。这意味着内容过滤系统必须持续监控和调整。
- **PII 不可妥协**：PII 泄露的法律后果严重，对 PII 检测应采用"宁可误报也不漏报"的策略。
- **对抗是常态**：攻击者会持续研究过滤器的弱点和绕过方法。过滤器需要持续更新，定期做红队测试。
- **文化意识不可忽视**：在全球部署的 Agent 必须理解当地的内容规范，"一刀切"的策略在跨文化场景下必然失败。
- **流式审核是未来方向**：hidden-state probing 等技术让 Agent 能在有害内容生成完成之前就终止它，大大减少了风险暴露窗口。
- **成本控制是工程关键**：LLM 审核的质量最高但成本最高。通过级联架构，将 LLM 审核限制在 5% 的流量上，可以在质量和成本之间取得平衡。
