# 9.4.2 Graph Construction — 知识图谱自动构建：NER + 关系抽取

> **核心问题**：如何从非结构化文本中自动识别实体及其关系，构建可用于 RAG 的知识图谱？

---

## 1. 背景与问题

### 1.1 手工构建 KG 的困境

传统知识图谱（如 Freebase、Wikidata）依赖大量领域专家进行人工标注和录入。这种方式存在三个不可持续的问题：

```
手工构建成本模型:
┌─────────────────────────────────────────────────┐
│  环节             时间占比    质量风险           │
├─────────────────────────────────────────────────┤
│  本体设计          15%       架构缺陷传播       │
│  实体标注          35%       标注不一致         │
│  关系标注          30%       关系遗漏           │
│  质量控制          20%       人工复核瓶颈       │
└─────────────────────────────────────────────────┘

结论: 在 RAG 场景下，需要构建**领域特定**图谱，依赖人工完全不现实。
```

### 1.2 RAG 场景下的特殊需求

| 需求维度 | 说明 | 对构建管线的约束 |
|---------|------|----------------|
| **时效性** | 知识需要快速纳入 | 构建流程必须自动化，延迟在分钟级 |
| **覆盖度** | 覆盖所有文档提及的实体 | 不能依赖预定义词典 |
| **噪声容忍** | RAG 允许一定程度的不精确 | 可以接受召回率优先，LLM 后续裁决 |
| **可更新** | 知识持续变化 | 需要增量更新而非全量重建 |
| **成本敏感** | Token 消耗直接关联成本 | 需要权衡 LLM 调用 vs 传统方法 |

### 1.3 核心挑战

从非结构化文本构建知识图谱，本质上需要解决以下问题：

```
输入: "苹果公司CEO蒂姆·库克在2023年WWDC上发布了Vision Pro，该设备搭载了M2芯片。"

需要抽取:
  实体1: 苹果公司 (ORG)
  实体2: 蒂姆·库克 (PERSON)
  实体3: 2023年WWDC (EVENT)
  实体4: Vision Pro (PRODUCT)
  实体5: M2芯片 (PRODUCT)

  关系1: (苹果公司, 首席执行官, 蒂姆·库克)
  关系2: (蒂姆·库克, 发布, Vision Pro)
  关系3: (Vision Pro, 搭载, M2芯片)
  关系4: (蒂姆·库克, 参与, 2023年WWDC)
```

看似简单的例子，实际上包含：嵌套实体识别、多重关系抽取、实体消歧等核心难题。

---

## 2. 命名实体识别 (NER)

### 2.1 问题定义

NER 的目标是从文本中识别出具有特定意义的实体，并将其分类到预定义类型中（如人名、组织、地点、时间等）。

```
形式化定义:
  给定文本序列 X = [x₁, x₂, ..., xₙ]
  输出实体集合 E = {(e₁, t₁, s₁, e₁), ..., (eₖ, tₖ, sₖ, eₖ)}
  其中 eᵢ 是实体文本, tᵢ 是类型, sᵢ 是起始位置, eᵢ 是结束位置
```

### 2.2 方法演进

#### 2.2.1 传统方法：CRF + 特征工程

Conditional Random Fields (CRF) 是序列标注的经典方法。它建模整个标注序列的联合概率 P(Y|X)，而不是每个 token 独立预测。

```
CRF 模型结构:
         y₁ ── y₂ ── y₃ ── ... ── yₙ
         │      │      │            │
         │      │      │            │
         x₁    x₂    x₃    ...    xₙ

  特征: 词本身 + 词性 + 大小写 + 前后缀 + 上下文窗口词
  标注: B-PER, I-PER, B-ORG, I-ORG, O (BIO 标注模式)
```

**优缺点**：
- 优点：可解释性强，小数据量表现好，全局最优解码
- 缺点：严重依赖特征工程，泛化能力差，无法处理未登录词

#### 2.2.2 深度学习方法：BiLSTM-CRF

BiLSTM-CRF 将双向 LSTM 的上下文建模能力与 CRF 的序列约束相结合：

```
BiLSTM-CRF 架构:
                     ┌─────────────────────┐
                     │    CRF Layer         │ ← 序列约束
                     └────────┬────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
          ┌───────┐      ┌───────┐      ┌───────┐
          │ LSTM  │      │ LSTM  │      │ LSTM  │  → 正向
          └───────┘      └───────┘      └───────┘
              ↑               ↑               ↑
          ┌───────┐      ┌───────┐      ┌───────┐
          │ LSTM  │      │ LSTM  │      │ LSTM  │  → 反向
          └───────┘      └───────┘      └───────┘
              │               │               │
          ┌───────┐      ┌───────┐      ┌───────┐
          │ w₁    │      │ w₂    │      │ w₃    │  → 词嵌入
          └───────┘      └───────┘      └───────┘
```

```python
# 简化的 BiLSTM-CRF 伪代码
import torch
import torch.nn as nn
from torchcrf import CRF

class BiLSTM_CRF(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_dim, num_tags):
        super().__init__()
        self.embeddings = nn.Embedding(vocab_size, embedding_dim)
        self.bilstm = nn.LSTM(
            embedding_dim, hidden_dim // 2,
            num_layers=1, bidirectional=True, batch_first=True
        )
        self.fc = nn.Linear(hidden_dim, num_tags)
        self.crf = CRF(num_tags, batch_first=True)

    def forward(self, x, mask=None):
        embeds = self.embeddings(x)                    # (B, L, E)
        lstm_out, _ = self.bilstm(embeds)              # (B, L, H)
        emissions = self.fc(lstm_out)                  # (B, L, T)
        return emissions

    def loss(self, x, tags, mask=None):
        emissions = self.forward(x)
        return -self.crf(emissions, tags, mask=mask)   # 负对数似然

    def decode(self, x, mask=None):
        emissions = self.forward(x)
        return self.crf.decode(emissions, mask=mask)   # Viterbi 解码
```

**BERT-NER 的关键改进**：使用预训练 Transformer 替代 BiLSTM 作为编码器，大幅提升上下文理解能力。

```python
# 使用 transformers 库构建 BERT-NER
from transformers import AutoModelForTokenClassification, AutoTokenizer
import torch

model_name = "dslim/bert-base-NER"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForTokenClassification.from_pretrained(model_name)

text = "Apple Inc. was founded by Steve Jobs in Cupertino."
inputs = tokenizer(text, return_tensors="pt")

with torch.no_grad():
    outputs = model(**inputs)
    predictions = torch.argmax(outputs.logits, dim=-1)

# 解码预测结果
tokens = tokenizer.convert_ids_to_tokens(inputs["input_ids"][0])
pred_labels = [model.config.id2label[p.item()] for p in predictions[0]]

for token, label in zip(tokens, pred_labels):
    if label != "O":
        print(f"{token:15s} -> {label}")

# 输出: Apple -> B-ORG, Inc -> I-ORG, Steve -> B-PER, Jobs -> I-PER, Cupertino -> B-LOC
```

#### 2.2.3 LLM 方法：Few-shot NER 与 Schema-based 抽取

LLM 将 NER 从序列标注问题转化为文本生成问题，极大降低了领域适配成本。

```python
"""
LLM-based NER: 通过 Prompt 从文本中抽取实体
"""

NER_PROMPT_TEMPLATE = """从以下文本中识别所有实体，并按类型分类。

实体类型: {entity_types}
- PERSON: 人名
- ORG: 组织/公司
- LOC: 地点
- PRODUCT: 产品
- EVENT: 事件
- DATE: 日期/时间

输出格式为 JSON:
{{
    "entities": [
        {{"text": "实体文本", "type": "实体类型", "mention_start": 起始字符位置, "mention_end": 结束字符位置}}
    ]
}}

文本: {text}

只输出 JSON，不要额外解释。
"""


def extract_entities_with_llm(text: str, entity_types: list, llm_client) -> list:
    """
    使用 LLM 从文本中抽取实体。

    Args:
        text: 输入文本
        entity_types: 需要抽取的实体类型列表
        llm_client: LLM API 客户端

    Returns:
        实体列表 [{"text": str, "type": str, "start": int, "end": int}, ...]
    """
    prompt = NER_PROMPT_TEMPLATE.format(
        entity_types=entity_types,
        text=text
    )

    response = llm_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0,        # NER 需要确定性
        response_format={"type": "json_object"}
    )

    import json
    result = json.loads(response.choices[0].message.content)
    return result["entities"]


# ============================================================
# 进阶：Schema-based NER —— 使用函数调用（Tool Calling）
# 让 LLM 按预定义 Schema 输出，更稳定
# ============================================================

NER_TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "extract_entities",
            "description": "从文本中抽取预定义类型的实体",
            "parameters": {
                "type": "object",
                "properties": {
                    "entities": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "text": {"type": "string", "description": "实体文本"},
                                "type": {
                                    "type": "string",
                                    "enum": ["PERSON", "ORG", "LOC", "PRODUCT", "EVENT", "DATE"]
                                },
                                "start": {"type": "integer", "description": "起始字符偏移"},
                                "end": {"type": "integer", "description": "结束字符偏移"}
                            },
                            "required": ["text", "type", "start", "end"]
                        }
                    }
                },
                "required": ["entities"]
            }
        }
    }
]


def extract_entities_tool_call(text: str, llm_client) -> list:
    """
    使用 Tool Calling 方式实现结构化 NER 抽取。
    这种方式比 Free-text JSON 更稳定，输出格式有保证。
    """
    response = llm_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": f"从以下文本中抽取所有实体:\n\n{text}"}],
        tools=NER_TOOLS,
        tool_choice={"type": "function", "function": {"name": "extract_entities"}},
        temperature=0.0
    )

    import json
    tool_call = response.choices[0].message.tool_calls[0]
    return json.loads(tool_call.function.arguments)["entities"]
```

### 2.3 三种 NER 方法对比

| 维度 | CRF | BiLSTM-CRF / BERT-NER | LLM NER |
|------|-----|----------------------|---------|
| **标注数据需求** | 大量（千级标注句） | 中等（百级标注句） | 极少（0-shot 可行） |
| **推理速度** | 极快（ms 级） | 快（10-100ms） | 慢（1-5s/句） |
| **领域迁移成本** | 高（重做特征） | 中（Fine-tune） | 低（改 Prompt） |
| **实体类型灵活性** | 固定类型集 | 固定类型集 | 动态指定 |
| **长尾实体召回** | 低（OOV 问题） | 中（依赖预训练） | 高（世界知识） |
| **计算成本** | 极低 | 低-中 | 高（Token 计费） |
| **可解释性** | 高（特征权重可见） | 中（Attention 热图） | 低（黑盒） |
| **适合场景** | 高吞吐、固定类型的生产管线 | 中等规模、精度优先 | 灵活抽取、快速原型 |

### 2.4 工程最佳实践：分层 NER 管线

在实际 RAG 系统中，推荐的策略是**分层 NER**：

```python
class HierarchicalNERPipeline:
    """
    分层 NER 管线：
    第一层：spaCy 快速抽取（高精度类型，如人名、组织）
    第二层：LLM 补充抽取（长尾、非标准类型）
    第三层：实体合并与消歧
    """

    def __init__(self, llm_client, spacy_model="en_core_web_trf"):
        import spacy
        self.nlp = spacy.load(spacy_model)
        self.llm = llm_client

    def extract_fast(self, text: str) -> list:
        """spaCy 高速抽取基础类型实体"""
        doc = self.nlp(text)
        return [
            {"text": ent.text, "type": ent.label_, "start": ent.start_char, "end": ent.end_char}
            for ent in doc.ents
        ]

    def extract_deep(self, text: str, custom_types: list) -> list:
        """LLM 抽取更灵活或长尾的实体类型"""
        # 只让 LLM 抽取 spaCy 不支持的类型，
        # 不重复抽取基础类型，节省 Token
        return extract_entities_tool_call(
            text + "\n重点识别以下类型: " + ", ".join(custom_types),
            self.llm
        )

    def merge_entities(self, fast_entities: list, deep_entities: list) -> list:
        """
        合并并去重两个来源的实体。
        策略：相同文本 + 相同类型 → 保留一个；
        相同文本 + 不同类型 → 保留 LLM 结果（置信度更高）。
        """
        seen = {}
        for ent in fast_entities + deep_entities:
            key = (ent["text"].lower(), ent["type"])
            if key not in seen or ent in deep_entities:  # LLM 结果优先
                seen[key] = ent
        return list(seen.values())

    def extract(self, text: str, custom_types: list = None) -> list:
        fast = self.extract_fast(text)
        if custom_types:
            deep = self.extract_deep(text, custom_types)
            return self.merge_entities(fast, deep)
        return fast
```

---

## 3. 关系抽取 (Relation Extraction)

### 3.1 问题定义

关系抽取的目标是识别文本中实体之间的语义关系，输出三元组：

```
给定: 文本 T, 实体集合 E₁, E₂, ..., Eₙ
输出: 关系三元组 R = {(h, r, t) | h ∈ E, t ∈ E, r ∈ RelationType}
  其中 h = head_entity (头实体), t = tail_entity (尾实体), r = relation (关系)
```

### 3.2 三种主要范式

#### 3.2.1 管道式 (Pipeline): NER → RE 两阶段

先识别实体，再枚举实体对进行关系分类：

```
管道式流程:
  文本 → [NER] → 实体列表 → [枚举实体对] → [关系分类] → 三元组集合

  实体对枚举:
    (苹果公司, 蒂姆·库克) → 类别: 首席执行官 ✓
    (苹果公司, Vision Pro) → 类别: 发布产品    ✗
    (蒂姆·库克, Vision Pro) → 类别: 发布       ✓
    (苹果公司, M2芯片) → 类别: 使用技术        ✓
```

```python
"""
管道式关系抽取示例：使用预训练关系分类模型
"""

from transformers import pipeline

# 加载预训练关系分类器
classifier = pipeline(
    "text-classification",
    model="bert-base-uncased",  # 实际需要 fine-tune 的关系分类模型
)

# 预定义的关系类型
RELATIONS = [
    "CEO_OF", "FOUNDED_BY", "LOCATED_IN", "PRODUCES",
    "DEVELOPED", "PART_OF", "ACQUIRED", "EMPLOYS"
]


def pipe_re(entity_pairs: list, text: str, llm_client=None) -> list:
    """
    管道式关系抽取：枚举实体对，分类关系。

    Args:
        entity_pairs: [(e1, e2), ...] 包含实体文本及其上下文
        text: 原始文本
        llm_client: 可选 LLM 客户端

    Returns:
        关系三元组列表 [(头实体, 关系, 尾实体), ...]
    """
    triples = []
    for e1, e2 in entity_pairs:
        # 构建关系分类输入
        # 策略: 找到包含两个实体的最近句子作为上下文
        context = extract_mention_context(text, e1, e2)

        prompt = f"""判断 "{e1}" 和 "{e2}" 之间的关系。
上下文: {context}
可能的关系类型: {RELATIONS}
如果不存在明确关系，输出 "NO_RELATION"。
输出格式: RELATION

关系: """

        response = llm_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.0,
            max_tokens=10
        )

        relation = response.choices[0].message.content.strip()
        if relation != "NO_RELATION":
            triples.append((e1, relation, e2))

    return triples


def extract_mention_context(text: str, e1_text: str, e2_text: str) -> str:
    """提取同时包含两个实体的最小文本窗口"""
    import re

    # 找到两个实体的位置
    e1_match = re.search(re.escape(e1_text), text)
    e2_match = re.search(re.escape(e2_text), text)

    if not e1_match or not e2_match:
        return text[:512]  # 回退策略

    start = min(e1_match.start(), e2_match.start())
    end = max(e1_match.end(), e2_match.end())

    # 扩展到最近的句子边界
    sent_start = max(0, text.rfind(".", 0, start) + 1)
    sent_end = text.find(".", end)
    if sent_end == -1:
        sent_end = len(text)

    return text[sent_start:sent_end + 1].strip()
```

**管道式的两大问题**：
1. **误差传播**：NER 的错误会直接传递给 RE，导致级联放大
2. **效率问题**：n 个实体会产生 O(n²) 个候选对，大部分是无效的

#### 3.2.2 联合式：实体关系联合抽取

同时建模实体和关系，共享底层表示，避免误差传播。

```
联合抽取模型（以 TPLinker 为例）:

输入: "Steve Jobs founded Apple."
     ┌─────────────────────────────────┐
     │  BERT / RoBERTa Encoder          │
     └─────────────────────────────────┘
                ↓
     ┌─────────────────────────────────┐
     │  Handshaking Tagging 机制        │
     │  (实体起始-起始矩阵 + 关系标记)     │
     └─────────────────────────────────┘
                ↓
     ┌─────────────────────────────────┐
     │  解码: 实体识别 + 关系判定同时输出  │
     │  输出: (Steve Jobs, founder, Apple)│
     └─────────────────────────────────┘

  TPLinker 的核心思想:
  用矩阵标记 (entity_head, entity_tail, relation_type) 的关系
  而不是先抽实体再分类关系
```

```python
"""
简化的联合抽取思路（非完整实现，用于说明结构）
"""

def joint_extract_simulated(text: str, entities: list, llm_client) -> list:
    """
    模拟联合抽取：在一次 LLM 调用中同时输出实体和关系。
    实际联合抽取模型（如 TPLinker、PRGC）使用专用网络结构，
    这里用 LLM 模拟其"一次性输出"的理念。
    """
    prompt = f"""从以下文本中抽取所有(头实体, 关系, 尾实体)三元组。

文本: {text}

已知实体: {entities}

输出格式为 JSON 数组:
[
    {{"head": "头实体文本", "relation": "关系名称", "tail": "尾实体文本"}}
]

确保:
1. 头实体和尾实体都来自已知实体列表（或你可以补充新实体）
2. 关系名称使用英文动词/介词形式
3. 只输出明确由文本支持的三元组
"""

    response = llm_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0,
        response_format={"type": "json_object"}
    )

    import json
    result = json.loads(response.choices[0].message.content)
    return [(t["head"], t["relation"], t["tail"]) for t in result]
```

#### 3.2.3 LLM-based：直接 Prompt 输出三元组

最直接的方式——让 LLM 一步完成 NER + RE，输出结构化三元组：

```python
"""
基于 LLM 的端到端三元组抽取
"""

TRIPLE_EXTRACTION_PROMPT = """你是一个知识图谱构建专家。从以下文本中抽取所有(头实体, 关系, 尾实体)三元组。

抽取规则:
1. 只抽取文本中明确陈述的事实（不推理、不臆测）
2. 实体使用文本中的原词
3. 关系使用标准化的英文关系词
4. 为每个三元组标注置信度（0.0-1.0）
5. 如果一句话包含多个三元组，全部抽取

输出格式:
{{
    "triples": [
        {{"head": "实体A", "relation": "关系名", "tail": "实体B", "confidence": 0.95, "source_sentence": "原文句子"}}
    ]
}}

文本:
{text}
"""


def extract_triples(text: str, llm_client, model="gpt-4o-mini") -> list:
    """
    使用 LLM 从文本中直接抽取三元组（端到端）。
    """
    # 将长文本分成段落分别抽取，避免上下文过长
    # 同时保留句子级别的 Source 信息
    paragraphs = [p for p in text.split("\n\n") if p.strip()]

    all_triples = []
    for para in paragraphs:
        response = llm_client.chat.completions.create(
            model=model,
            messages=[
                {"role": "system", "content": "你是一个精确的知识抽取助手，只抽取文本中明确陈述的事实。"},
                {"role": "user", "content": TRIPLE_EXTRACTION_PROMPT.format(text=para)}
            ],
            temperature=0.0,
            response_format={"type": "json_object"}
        )

        import json
        try:
            result = json.loads(response.choices[0].message.content)
            all_triples.extend(result["triples"])
        except (json.JSONDecodeError, KeyError) as e:
            print(f"解析失败: {e}")
            continue

    # 去重：相同 (head, relation, tail) 保留置信度最高的
    seen = {}
    for t in all_triples:
        key = (t["head"], t["relation"], t["tail"])
        if key not in seen or t["confidence"] > seen[key]["confidence"]:
            seen[key] = t

    return list(seen.values())


# ============================================================
# 高阶优化：Schema-guided 三元组抽取
# 通过预定义 Schema 约束关系类型，提高一致性
# ============================================================

SCHEMA_GUIDED_PROMPT = """从文本中抽取符合以下 Schema 的三元组:

预定义实体类型和关系:
{schema}

只抽取 Schema 中明确定义的关系类型。
如果两个实体之间存在多种关系，分别抽取为多个三元组。

文本: {text}

输出 JSON 数组 [{{"head", "relation", "tail", "confidence"}}]:
"""

# 示例 Schema
EXAMPLE_SCHEMA = """
实体类型: [PERSON, ORG, PRODUCT, LOC, EVENT, DATE]
关系类型:
  - PERSON → ORG: ["CEO_OF", "FOUNDED_BY", "EMPLOYEE_OF", "INVESTOR_IN"]
  - PERSON → PRODUCT: ["INVENTED", "DEVELOPED", "DESIGNED"]
  - ORG → PRODUCT: ["PRODUCES", "DEVELOPS", "OWNS"]
  - ORG → PERSON: ["HIRES", "FIRED"]
  - ORG → LOC: ["HEADQUARTERED_IN", "OPERATES_IN"]
  - PRODUCT → PRODUCT: ["POWERED_BY", "COMPATIBLE_WITH", "UPGRADE_FROM"]
"""
```

### 3.3 三种关系抽取方法对比

| 维度 | 管道式 | 联合抽取模型 | LLM-based |
|------|-------|------------|-----------|
| **误差传播** | 严重 | 轻微（共享编码） | N/A（端到端） |
| **效率/成本** | O(n²) 枚举 | O(n) 或 O(n log n) | O(1) 每次 Token 计费 |
| **关系类型灵活性** | 固定类型 | 固定类型 | 动态指定 |
| **关系重叠处理** | 困难 | 可处理 | 天然支持 |
| **需要标注数据** | NER + RE 分开标注 | 联合标注（更复杂） | 几乎为零 |
| **精度上限** | 高（有监督上限高） | 高 | 中高（取决于模型） |
| **生产部署成熟度** | 最成熟 | 中等 | 快速提升中 |

---

## 4. 端到端构建管线

### 4.1 完整流程 ASCII 图

```
                    ┌─────────────────────────────────────┐
                    │          文档库 (Document Store)       │
                    │  (.pdf, .docx, .html, .md, .txt...)  │
                    └────────────────┬────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │    阶段1: 文档预处理 (Preprocessing)   │
                    │  - 文本提取 (OCR / Parser)            │
                    │  - 段落分割 (Paragraph Segmentation)   │
                    │  - 句子边界检测 (Sentence Splitting)    │
                    └────────────────┬────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │    阶段2: 命名实体识别 (NER)           │
                    │  ┌─────────────────────────────┐     │
                    │  │  分层 NER 管线                │     │
                    │  │  ├─ spaCy 基础实体(快速)      │     │
                    │  │  ├─ LLM 补充实体(深度)        │     │
                    │  │  └─ 实体合并与去重             │     │
                    │  └─────────────────────────────┘     │
                    └────────────────┬────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │    阶段3: 关系抽取 (RE)                │
                    │  ┌─────────────────────────────┐     │
                    │  │  不同策略可选:                │     │
                    │  │  ├─ 句子级共现+LLM分类       │     │
                    │  │  ├─ 跨句关系推理             │     │
                    │  │  └─ Schema-constrained 抽取  │     │
                    │  └─────────────────────────────┘     │
                    └────────────────┬────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │    阶段4: 实体链接 (Entity Linking)    │
                    │  ┌─────────────────────────────┐     │
                    │  │  将文本实体映射到 KG 节点      │     │
                    │  │  ├─ 同义合并 ("Apple"="苹果")  │     │
                    │  │  ├─ 消歧 ("苹果(水果)" vs     │     │
                    │  │  │    "苹果(公司)")            │     │
                    │  │  └─ 共指消解 ("它"→"Vision Pro")│     │
                    │  └─────────────────────────────┘     │
                    └────────────────┬────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │    阶段5: 图谱存储 (Graph Storage)     │
                    │  ┌─────────────────────────────┐     │
                    │  │  节点: 实体 (带类型属性)       │     │
                    │  │  边: 关系 (带置信度、来源)     │     │
                    │  │  存储: Neo4j / NebulaGraph /  │     │
                    │  │        NetworkX + 序列化       │     │
                    │  └─────────────────────────────┘     │
                    └────────────────┬────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │    阶段6: 质量校验 (Quality Gate)      │
                    │  ┌─────────────────────────────┐     │
                    │  │  - 置信度过滤 (confidence     │     │
                    │  │    threshold)                │     │
                    │  │  - 冲突检测 (矛盾三元组)      │     │
                    │  │  - 人工审核采样              │     │
                    │  └─────────────────────────────┘     │
                    └────────────────┬────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────────┐
                    │         知识图谱 (Knowledge Graph)     │
                    │   供 Graph RAG 检索使用               │
                    └─────────────────────────────────────┘
```

### 4.2 端到端管线代码实现

```python
"""
端到端知识图谱构建管线
"""

import json
from typing import List, Dict, Any


class KnowledgeGraph:
    """简化版知识图谱存储"""

    def __init__(self):
        self.entities = {}      # entity_id -> {"name": str, "type": str, ...}
        self.relations = []     # [{"head_id": str, "relation": str, "tail_id": str, ...}]
        self._next_entity_id = 0

    def add_entity(self, name: str, entity_type: str, metadata: dict = None) -> str:
        """添加实体，返回 entity_id"""
        entity_id = f"E{self._next_entity_id}"
        self._next_entity_id += 1
        self.entities[entity_id] = {
            "name": name,
            "type": entity_type,
            "metadata": metadata or {}
        }
        return entity_id

    def add_relation(self, head_id: str, relation: str, tail_id: str,
                     confidence: float = 1.0, source: str = None):
        self.relations.append({
            "head_id": head_id,
            "relation": relation,
            "tail_id": tail_id,
            "confidence": confidence,
            "source": source
        })

    def get_entity_id(self, name: str, entity_type: str = None) -> str:
        """查找已有实体 ID（简化版：精确匹配）"""
        for eid, entity in self.entities.items():
            if entity["name"].lower() == name.lower():
                if entity_type and entity["type"] != entity_type:
                    continue
                return eid
        return None

    def find_or_create(self, name: str, entity_type: str) -> str:
        eid = self.get_entity_id(name, entity_type)
        if eid:
            return eid
        return self.add_entity(name, entity_type)


class GraphConstructionPipeline:
    """
    知识图谱构建管线
    """

    def __init__(self, llm_client, spacy_model="en_core_web_trf"):
        self.llm = llm_client
        import spacy
        self.nlp = spacy.load(spacy_model)
        self.graph = KnowledgeGraph()

    def preprocess(self, document_path: str) -> List[Dict]:
        """
        文档预处理：分句 + 分段。
        返回段落列表，每段包含段落文本和句子列表。
        """
        with open(document_path, "r", encoding="utf-8") as f:
            text = f.read()

        # 分段落
        paragraphs = text.split("\n\n")

        processed = []
        for i, para in enumerate(paragraphs):
            if not para.strip():
                continue

            # 用 spaCy 分句
            doc = self.nlp(para)
            sentences = [sent.text.strip() for sent in doc.sents if sent.text.strip()]

            processed.append({
                "paragraph_id": i,
                "text": para.strip(),
                "sentences": sentences
            })

        return processed

    def extract_triples_from_paragraph(self, paragraph: Dict) -> List[Dict]:
        """
        从单个段落抽取三元组。
        策略：用 LLM 端到端抽取。
        """
        # 先做快速 NER 为 LLM 提供上下文
        doc = self.nlp(paragraph["text"])
        fast_entities = [
            {"text": ent.text, "type": ent.label_}
            for ent in doc.ents
        ]

        # LLM 抽取完整三元组
        context_info = f"段落中已知的实体: {fast_entities}"
        triples = extract_triples(
            context_info + "\n\n" + paragraph["text"],
            self.llm
        )

        return triples

    def build_graph(self, document_path: str) -> KnowledgeGraph:
        """
        执行完整构建流程。
        """
        # 阶段1: 预处理
        paragraphs = self.preprocess(document_path)

        # 阶段2-3: NER + RE
        all_triples = []
        for para in paragraphs:
            triples = self.extract_triples_from_paragraph(para)
            # 过滤低置信度
            triples = [t for t in triples if t.get("confidence", 1.0) >= 0.7]
            all_triples.extend(triples)

        # 阶段4-5: 实体链接 + 图谱存储
        for t in all_triples:
            head_id = self.graph.find_or_create(t["head"], t.get("head_type", "ENTITY"))
            tail_id = self.graph.find_or_create(t["tail"], t.get("tail_type", "ENTITY"))
            self.graph.add_relation(
                head_id, t["relation"], tail_id,
                confidence=t.get("confidence", 1.0),
                source=t.get("source_sentence", "")
            )

        return self.graph

    def build_incremental(self, new_document_path: str, existing_graph: KnowledgeGraph) -> KnowledgeGraph:
        """
        增量构建：在已有图谱基础上添加新文档的知识。
        - 新实体添加到图谱
        - 已有实体复用 ID
        - 新关系加入（如有冲突，保留置信度高的）
        """
        self.graph = existing_graph
        paragraphs = self.preprocess(new_document_path)

        for para in paragraphs:
            triples = self.extract_triples_from_paragraph(para)

            for t in triples:
                # 尝试链接到已有实体
                head_id = self.graph.get_entity_id(t["head"])
                if not head_id:
                    head_id = self.graph.add_entity(t["head"], t.get("head_type", "ENTITY"))
                    print(f"  新增实体: {t['head']} ({head_id})")

                tail_id = self.graph.get_entity_id(t["tail"])
                if not tail_id:
                    tail_id = self.graph.add_entity(t["tail"], t.get("tail_type", "ENTITY"))
                    print(f"  新增实体: {t['tail']} ({tail_id})")

                self.graph.add_relation(
                    head_id, t["relation"], tail_id,
                    confidence=t.get("confidence", 1.0),
                    source=t.get("source_sentence", "")
                )

        return self.graph
```

### 4.3 管线关键决策点

```
在构建管线中，每个阶段都有权衡：

阶段1: 文档分块粒度
  句子级 ←───────────→ 段落级 ←───────────→ 文档级
  精度高，召回低        平衡                召回高，精度低
  适合关系抽取          默认选择              适合主题抽取

阶段2: NER 策略
  spaCy-only → 速度最快，类型有限
  LLM-only → 最灵活，成本最高
  分层策略 → 推荐方案，兼顾成本和质量

阶段3: 关系抽取范围
  句内关系 → 精度高，丢失跨句关系
  段落内跨句 → 推荐，适度召回
  文档级 → 精度下降，但能发现长距离关系

阶段4: 实体链接策略
  精确匹配 → 保守，避免错误链接
  模糊匹配 → 提高覆盖，但有误链接风险
  LLM判断 → 灵活但成本高
```

---

## 5. 核心挑战与方案

### 5.1 实体消歧 (Entity Disambiguation)

同一文本可能指向不同实体，取决于上下文。

```
"苹果"的消歧:
                      ┌──────────────────┐
  输入: "苹果发布了   │  上下文分析        │
        iPhone 15"   │  → 科技、产品发布   │
                      └────────┬─────────┘
                               │ 苹果公司 (ORG)
                               │
                      ┌──────────────────┐
  输入: "苹果是一种   │  上下文分析        │
        富含维生素    │  → 水果、营养      │
        的水果"       │                   │
                      └────────┬─────────┘
                               │ 苹果 (FRUIT)
```

```python
def disambiguate_entity(entity_text: str, context: str, candidates: list) -> str:
    """
    基于上下文对实体进行消歧。

    Args:
        entity_text: 实体文本
        context: 上下文（完整句子或段落）
        candidates: 候选实体列表 [{"id": str, "name": str, "description": str}, ...]

    Returns:
        最可能的候选实体 ID
    """
    if len(candidates) == 1:
        return candidates[0]["id"]

    prompt = f"""实体消歧: "{entity_text}" 在以下上下文中指代哪个实体？

上下文: {context}

候选实体:
{json.dumps(candidates, ensure_ascii=False, indent=2)}

只输出最匹配的候选实体 ID:
"""

    response = llm_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0
    )

    return response.choices[0].message.content.strip()
```

### 5.2 共指消解 (Coreference Resolution)

"他"、"它"、"该公司"等代词需要解析到实际的实体。

```
共指消解示例:

输入: "苹果公司在1984年发布了Macintosh。它彻底改变了个人电脑行业。"

       ┌─────────────────────────────────────────────┐
       │  共指消解模型                                │
       │  (e2e-coref / LLM with coref prompt)         │
       └─────────────────────────────────────────────┘
                        ↓
       识别: "它" → "苹果公司"

输出: "苹果公司在1984年发布了Macintosh。苹果公司彻底改变了个人电脑行业。"
      (替换后的文本，便于后续 NER/RE 准确抽取)
```

```python
def resolve_coreferences(text: str, llm_client) -> str:
    """
    使用 LLM 解析共指关系，将代词替换为实际实体名。
    这是简化方案 — 生产环境推荐专用模型如 e2e-coref。
    """
    prompt = f"""将以下文本中的所有代词（他、她、它、他们、该公司、该产品等）
替换为它们所指代的实际实体名称。只输出替换后的文本，不要额外解释。

如果代词指代不明确，保留原文不变。

文本: {text}

替换后的文本:
"""

    response = llm_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0
    )

    return response.choices[0].message.content.strip()
```

### 5.3 关系重叠 (Relation Overlap)

一个句子可能包含多个三元组，且实体和关系之间存在多种重叠模式：

```
关系重叠模式:

1. Single Entity Overlap (SEO) — 实体参与多个关系:
   "Steve Jobs founded Apple and also founded NeXT."
   → (Steve Jobs, founded, Apple)
   → (Steve Jobs, founded, NeXT)

2. Entity Pair Overlap (EPO) — 同一对实体有多种关系:
   "Tim Cook is CEO of Apple and joined Apple in 1998."
   → (Tim Cook, CEO_OF, Apple)
   → (Tim Cook, JOINED, Apple)

3. 嵌套:
   "The CEO of Apple, Tim Cook, announced Vision Pro."
   → (Tim Cook, CEO_OF, Apple)
   → (Tim Cook, announced, Vision Pro)
```

**应对策略**：
- LLM-based 方法天然能处理关系重叠（因为生成是并行解码）
- 传统模型需要特殊设计（如 TPLinker 的 Handshaking Tagging）
- 确保 Prompt 明确要求输出所有三元组，不要只输出第一个

### 5.4 长尾关系识别

常见关系（如`首席执行官`）识别效果好，少见关系（如`战略合作伙伴`、`交叉授权`）则差。

```python
"""
长尾关系处理策略：
1. 使用 LLM 的开放关系抽取（不限制关系类型）
2. 事后标准化（把相似关系映射到标准类型）
"""

OPEN_RE_PROMPT = """从以下文本中抽取所有(头实体, 关系, 尾实体)三元组。

注意: 关系类型不限于预定义集合。
请你根据文本语义自由定义关系名称（使用英文）。
但要保持一致性——相同语义关系使用相同关系名称。

文本:
{text}

输出 JSON 格式:
[{{"head": "...", "relation": "...", "tail": "..."}}]
"""


def normalize_relation(relation: str, standard_relations: dict) -> str:
    """
    将开放抽取的关系名标准化到预定义类型。

    Args:
        relation: 开放抽取的关系名，如 "is the CEO of"
        standard_relations: 标准关系类型映射
            {"ceo_of": ["is the CEO of", "serves as CEO of", "CEO of", ...]}

    Returns:
        标准关系名或原始关系名（如果无匹配）
    """
    rel_lower = relation.lower().strip()
    best_match = relation
    best_score = 0

    for standard, variants in standard_relations.items():
        for variant in variants:
            # 简单的包含匹配
            if variant.lower() in rel_lower or rel_lower in variant.lower():
                score = len(variant) / max(len(rel_lower), len(variant))
                if score > best_score:
                    best_score = score
                    best_match = standard

    return best_match if best_score > 0.6 else relation


# 示例关系标准化映射表
STANDARD_RELATIONS = {
    "CEO_OF": ["CEO of", "chief executive officer of", "serves as CEO of", "is the CEO of"],
    "FOUNDED_BY": ["founded by", "co-founded by", "was founded by", "established by"],
    "LOCATED_IN": ["located in", "based in", "headquartered in", "situated in"],
    "SUBSIDIARY_OF": ["subsidiary of", "owned by", "acquired by", "a division of"],
    "PRODUCES": ["produces", "manufactures", "makes", "develops", "builds"],
}
```

### 5.5 抽取质量控制

```python
class QualityGate:
    """
    质量控制门：过滤低质量和冲突的三元组。
    """

    MIN_CONFIDENCE = 0.5    # 最低置信度
    MAX_CONFLICT_RATIO = 0.3  # 冲突比例上限

    def __init__(self, min_confidence: float = 0.5):
        self.min_confidence = min_confidence

    def filter_confidence(self, triples: list) -> list:
        """过滤低置信度三元组"""
        return [t for t in triples if t.get("confidence", 1.0) >= self.min_confidence]

    def detect_conflicts(self, triples: list) -> list:
        """
        检测冲突三元组。
        冲突定义为：(A, relation, B) 和 (A, relation_neg, B) 矛盾的关系。
        """
        inverse_relations = {
            "CEO_OF": ["EMPLOYEE_OF"],
            "FOUNDED_BY": ["EMPLOYEE_OF"],
            "PARENT_OF": ["SUBSIDIARY_OF"],
        }

        conflicts = []
        rel_map = {}
        for t in triples:
            key = (t["head"], t["tail"])
            if key not in rel_map:
                rel_map[key] = []
            rel_map[key].append(t)

        for key, rels in rel_map.items():
            for i, r1 in enumerate(rels):
                for r2 in rels[i+1:]:
                    if r2["relation"] in inverse_relations.get(r1["relation"], []):
                        conflicts.append({
                            "head": key[0],
                            "tail": key[1],
                            "relation_a": r1["relation"],
                            "relation_b": r2["relation"],
                            "confidence_a": r1["confidence"],
                            "confidence_b": r2["confidence"]
                        })

        return conflicts

    def resolve_conflicts(self, triples: list) -> list:
        """
        解决冲突：保留置信度更高的，或标记待人工审核。
        """
        conflicts = self.detect_conflicts(triples)
        if not conflicts:
            return triples

        # 保留高置信度，删除低置信度
        to_remove = set()
        for c in conflicts:
            if c["confidence_a"] >= c["confidence_b"]:
                to_remove.add((c["head"], c["relation_b"], c["tail"]))
            else:
                to_remove.add((c["head"], c["relation_a"], c["tail"]))

        return [
            t for t in triples
            if (t["head"], t["relation"], t["tail"]) not in to_remove
        ]

    def validate(self, triples: list) -> tuple:
        """
        执行完整质量控制流程。
        返回 (有效三元组, 警告列表)
        """
        warnings = []

        # 1. 置信度过滤
        valid = self.filter_confidence(triples)
        filtered_count = len(triples) - len(valid)
        if filtered_count > 0:
            warnings.append(f"过滤了 {filtered_count} 个低置信度三元组")

        # 2. 冲突检测与解决
        conflicts = self.detect_conflicts(valid)
        if conflicts:
            warnings.append(f"检测到 {len(conflicts)} 组冲突，已自动解决")
            valid = self.resolve_conflicts(valid)

        # 3. 结构验证
        schema_warnings = self.validate_schema(valid)
        warnings.extend(schema_warnings)

        return valid, warnings

    def validate_schema(self, triples: list) -> list:
        """验证三元组是否符合基本结构规则"""
        issues = []
        for t in triples:
            if not t.get("head") or not t.get("tail"):
                issues.append(f"三元组缺少头或尾实体: {t}")
            if not t.get("relation"):
                issues.append(f"三元组缺少关系: {t}")
        return issues
```

---

## 6. 增量更新 (Incremental Update)

知识图谱的一大挑战是**持续更新**。典型的 RAG 场景中，每天可能有新文档加入，全量重建成本过高。

### 6.1 增量构建策略

```
全量重建 vs 增量更新:
┌──────────────────────────────┬──────────────────────────────┐
│  全量重建                      │  增量更新                    │
├──────────────────────────────┼──────────────────────────────┤
│  重新处理所有文档              │  只处理新增/修改的文档         │
│  O(N) 成本，N=文档总数         │  O(M) 成本，M=新增文档数      │
│  M << N 时严重浪费            │  M << N 时显著节省            │
│  保证图一致性                  │  需处理版本控制               │
│  停机时间不可避免              │  在线更新，无停机             │
│  适合：首次构建/重大Schema变更  │  适合：日常增量文档流入        │
└──────────────────────────────┴──────────────────────────────┘
```

### 6.2 增量更新的核心设计

```python
class IncrementalGraphUpdater:
    """
    增量图谱更新器。

    核心策略:
    1. 对新文档运行完整 NER + RE
    2. 与已有图谱做实体链接（而不是重建）
    3. 处理关系冲突（新知识 vs 旧知识）
    4. 记录变更日志用于回滚
    """

    def __init__(self, graph: KnowledgeGraph, llm_client):
        self.graph = graph
        self.llm = llm_client
        self.change_log = []

    def update_with_document(self, document_text: str, source: str) -> dict:
        """
        用新文档增量更新图谱。
        返回本次更新的统计信息。
        """
        stats = {"new_entities": 0, "new_relations": 0, "updated_relations": 0}

        # 1. 抽取新三元组
        triples = extract_triples(document_text, self.llm)

        for t in triples:
            head_id = self._link_entity(t["head"], t.get("head_type", "ENTITY"))
            tail_id = self._link_entity(t["tail"], t.get("tail_type", "ENTITY"))

            existing = self._find_existing_relation(head_id, t["relation"], tail_id)

            if existing:
                # 关系已存在 — 更新置信度（取平均或取高值）
                old_conf = existing["confidence"]
                new_conf = (old_conf + t.get("confidence", 0.8)) / 2
                existing["confidence"] = new_conf
                stats["updated_relations"] += 1
            else:
                # 新关系
                self.graph.add_relation(
                    head_id, t["relation"], tail_id,
                    confidence=t.get("confidence", 0.8),
                    source=source
                )
                stats["new_relations"] += 1

        # 记录变更
        self.change_log.append({
            "source": source,
            "timestamp": datetime.now().isoformat(),
            "triples_added": len(triples),
            "new_entities": stats["new_entities"]
        })

        return stats

    def _link_entity(self, name: str, entity_type: str) -> str:
        """
        实体链接：查找已有实体或创建新实体。
        使用多策略匹配：精确 → 别名 → 模糊。
        """
        # 策略1: 精确匹配
        eid = self.graph.get_entity_id(name, entity_type)
        if eid:
            return eid

        # 策略2: 别名匹配（如 "Apple" 匹配 "Apple Inc."）
        eid = self._alias_match(name)
        if eid:
            return eid

        # 策略3: 创建新实体
        new_id = self.graph.add_entity(name, entity_type)
        stats["new_entities"] += 1
        return new_id

    def rollback_latest(self) -> bool:
        """回滚最近的增量更新"""
        if not self.change_log:
            return False

        last_change = self.change_log.pop()
        print(f"回滚: {last_change['source']} ({last_change['timestamp']})")
        # 实际实现需要维护实体的版本历史
        # 这里简化处理
        return True

    def _alias_match(self, name: str) -> str:
        """基于别名和缩写的实体匹配"""
        alias_map = {
            "Apple": "Apple Inc.",
            "Apple Computer": "Apple Inc.",
            "Google": "Google LLC",
            "Facebook": "Meta Platforms Inc.",
            "Microsoft": "Microsoft Corporation",
        }
        canonical = alias_map.get(name)
        if canonical:
            return self.graph.get_entity_id(canonical)
        return None

    def _find_existing_relation(self, head_id: str, relation: str, tail_id: str) -> dict:
        """查找已有关系"""
        for r in self.graph.relations:
            if r["head_id"] == head_id and r["relation"] == relation and r["tail_id"] == tail_id:
                return r
        return None
```

### 6.3 增量更新实战注意事项

1. **时间戳管理**：每个三元组带更新时间戳，支持时间线回溯
2. **知识衰减**：旧知识可能过时，可引入折扣因子降低其置信度
3. **版本化存储**：保留历史版本，支持时间旅行查询（"2024年1月的知识状态"）
4. **冲突处理策略**：
   - 新覆盖旧（适用于新闻等时效性强的场景）
   - 置信度高的覆盖低的
   - 人工审核裁决（高价值场景）

---

## 7. 质量评估指标

### 7.1 NER 评估指标

| 指标 | 公式 | 说明 |
|------|------|------|
| **Precision** | TP / (TP + FP) | 所有识别出的实体中，正确的比例 |
| **Recall** | TP / (TP + FN) | 所有真实实体中，被识别出的比例 |
| **F1-score** | 2 * P * R / (P + R) | 调和平均 |
| **Strict 匹配** | 边界和类型完全一致 | 最严格的评估方式 |
| **Relaxed 匹配** | 部分重叠或类型接近 | 更实用的评估方式（如 B-PER 和 I-PER 部分匹配） |

### 7.2 关系抽取评估指标

```
RE 评估的特殊性:

正确三元组需要三个要素都匹配:
  - 头实体文本 ✓
  - 关系类型 ✓
  - 尾实体文本 ✓

部分正确的三元组:
  (Steve Jobs, CEO_OF, Apple)     ← 错误（他是创始人，不是CEO）
  (Steve Jobs, FOUNDED_BY, Apple) ← 错误（方向反了，应该是 Jobs FOUNDED Apple）
  (Steve, FOUNDED, Apple Inc.)    ← 模糊匹配（实体边界不完全一致）

评价标准:
  - 精确匹配: 头实体、关系、尾实体都完全正确
  - 模糊匹配: 允许实体边界轻微偏移
  - 方向敏感: (A, R, B) 和 (B, R, A) 是不同的
```

### 7.3 图级评估指标

除了逐点评估，还需要图级评估：

| 指标 | 说明 | 评估方式 |
|------|------|----------|
| **Graph Density** | 边的数量 / 可能边的数量 | 稀疏图(0.01-0.1) vs 稠密图(>0.3) |
| **Connectivity** | 连通分量数量 | 越低越好（孤立实体少） |
| **Schema Conformance** | 三元组符合本体约束的比例 | 高约束性场景下的关键指标 |
| **Human Evaluation** | 人工抽查图质量 | 实际效果最终标准 |

### 7.4 实际评估代码

```python
def evaluate_extraction(
    predicted_triples: list,
    ground_truth_triples: list,
    mode: str = "exact"
) -> dict:
    """
    评估抽取效果。

    Args:
        predicted_triples: 模型抽取的三元组
        ground_truth_triples: 人工标注的标准答案
        mode: "exact" (精确) 或 "relaxed" (模糊)

    Returns:
        {"precision": float, "recall": float, "f1": float}
    """
    pred_set = set()
    for t in predicted_triples:
        if mode == "exact":
            key = (t["head"], t["relation"], t["tail"])
        else:
            # 模糊：忽略实体边界差异，只看核心内容
            key = (t["head"].strip().lower(), t["relation"].lower(), t["tail"].strip().lower())
        pred_set.add(key)

    gt_set = set()
    for t in ground_truth_triples:
        if mode == "exact":
            key = (t["head"], t["relation"], t["tail"])
        else:
            key = (t["head"].strip().lower(), t["relation"].lower(), t["tail"].strip().lower())
        gt_set.add(key)

    tp = len(pred_set & gt_set)
    fp = len(pred_set - gt_set)
    fn = len(gt_set - pred_set)

    precision = tp / (tp + fp) if (tp + fp) > 0 else 0
    recall = tp / (tp + fn) if (tp + fn) > 0 else 0
    f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0

    return {
        "precision": round(precision, 4),
        "recall": round(recall, 4),
        "f1": round(f1, 4),
        "tp": tp,
        "fp": fp,
        "fn": fn
    }
```

---

## 8. 工程优化方向

### 8.1 LLM 调用成本优化

LLM-based NER/RE 的 Token 消耗是最大成本。以下是分层优化策略：

```
成本优化策略金字塔:
                    ┌──────────────────────┐
                    │  Cache 热文本复用      │  ← 成本最低
                   ┌┴──────────────────────┴┐
                  │  小模型预过滤+LLM精调    │
                 ┌┴────────────────────────┴┐
                │  批处理与 Prompt 压缩       │
               ┌┴──────────────────────────┴┐
              │  Schema 约束减少输出长度      │
             ┌┴────────────────────────────┴┐
            │  智能跳过（重复文本/无实体段落） │  ← 成本最高
            └────────────────────────────────┘
```

```python
class CostOptimizedExtractor:
    """
    成本优化的抽取器。

    策略:
    1. 用小模型做初筛（spaCy + 规则），只有疑似含实体的段落才调 LLM
    2. LLM 调用结果缓存
    3. 批量处理段落
    """

    def __init__(self, llm_client, spacy_model="en_core_web_sm"):
        self.llm = llm_client
        self.nlp = spacy.load(spacy_model)
        self.cache = {}  # text_hash -> result

    def should_call_llm(self, text: str) -> bool:
        """
        判断是否需要调 LLM 抽取。
        规则：如果 spaCy 没有识别到任何实体，且文本很短，跳过。
        """
        doc = self.nlp(text)
        # 如果有命名实体，调 LLM
        if len(doc.ents) >= 2:
            return True
        # 如果文本包含明显的关系指示词
        relation_indicators = [
            "is the", "was founded", "CEO of", "located in",
            "works for", "part of", "acquired", "developed"
        ]
        if any(indicator in text.lower() for indicator in relation_indicators):
            return True
        return False

    def extract_batch(self, texts: list[str]) -> list[list]:
        """
        批量抽取（带缓存）。
        """
        results = []
        uncached_texts = []
        uncached_indices = []

        for i, text in enumerate(texts):
            text_hash = hash(text)
            if text_hash in self.cache:
                results.append(self.cache[text_hash])
            else:
                results.append(None)
                uncached_texts.append(text)
                uncached_indices.append(i)

        # 批量调用 LLM（发送多个段落在一个请求中）
        if uncached_texts:
            batch_prompt = "批量抽取以下段落的三元组:\n\n"
            for idx, text in enumerate(uncached_texts):
                batch_prompt += f"---段落 {idx}---\n{text}\n\n"

            # 实际实现中需要调整 batch_prompt 长度避免超限
            batch_results = extract_triples(batch_prompt, self.llm)

            # 分发结果到各段落（简化实现）
            for idx, text in zip(uncached_indices, uncached_texts):
                result = batch_results  # 实际需要按段落分割
                self.cache[hash(text)] = result
                results[idx] = result

        return results
```

### 8.2 并行分片构建

对于大规模文档，分片并行处理是关键优化：

```python
from concurrent.futures import ThreadPoolExecutor, as_completed


def parallel_build(document_paths: list, num_workers: int = 4) -> KnowledgeGraph:
    """
    并行构建知识图谱。

    策略:
    1. 文档分片 → 每个 worker 处理一个分片
    2. 子图构建 → 每个分片独立构建子图
    3. 子图合并 → 实体链接合并各子图
    """
    pipeline = GraphConstructionPipeline(llm_client)

    all_subgraphs = []

    with ThreadPoolExecutor(max_workers=num_workers) as executor:
        futures = {
            executor.submit(pipeline.build_graph, doc_path): doc_path
            for doc_path in document_paths
        }

        for future in as_completed(futures):
            doc_path = futures[future]
            try:
                subgraph = future.result()
                all_subgraphs.append(subgraph)
                print(f"完成: {doc_path} ({len(subgraph.entities)} entities)")
            except Exception as e:
                print(f"失败: {doc_path}: {e}")

    # 合并子图
    master_graph = KnowledgeGraph()
    for subgraph in all_subgraphs:
        # 实体合并
        entity_mapping = {}
        for eid, entity in subgraph.entities.items():
            existing = master_graph.get_entity_id(entity["name"], entity["type"])
            if existing:
                entity_mapping[eid] = existing
            else:
                new_id = master_graph.add_entity(entity["name"], entity["type"])
                entity_mapping[eid] = new_id

        # 关系迁移
        for rel in subgraph.relations:
            master_graph.add_relation(
                entity_mapping[rel["head_id"]],
                rel["relation"],
                entity_mapping[rel["tail_id"]],
                confidence=rel["confidence"],
                source=rel.get("source")
            )

    return master_graph
```

### 8.3 人工审核闭环

对于高价值场景（如金融、医疗），需要引入人工审核：

```
人工审核闭环流程:

  自动抽取
      │
      ▼
  置信度排序 → 低置信度三元组 → 人工审核队列
      │                              │
      ▼                              ▼
  高置信度三元组              审核员确认/修正/删除
      │                              │
      ▼                              ▼
  直接入库 → [知识图谱] ← 人工审核通过后入库
  
  审核反馈 → 优化 Prompt / 调整置信度阈值
```

---

## 思考与总结

### 核心洞察

1. **没有银弹**：NER 和 RE 的方法选择取决于具体需求。高吞吐场景选传统模型，灵活性优先场景选 LLM。

2. **LLM 正在改变游戏规则**：传统 NER/RE 需要大量标注数据，LLM 实现了 0-shot 或 few-shot，极大降低了 KG 构建门槛。但成本是硬约束。

3. **分层策略是最佳实践**：单一方法无法解决所有问题。spaCy 做基础 NER + LLM 处理长尾和复杂关系，是目前成本效益最优的方案。

4. **质量 > 规模**：一个较小但高质量的知识图谱比大规模但充满噪声的图谱更有价值。质量控制门不是可选项，而是必需组件。

5. **Graph RAG 不是全量 KG**：为 RAG 构建的图谱不需要像维基数据那样全面。重点识别与问答最相关的实体和关系类型。

### 未来方向

- **多模态实体抽取**：从图片、表格、PDF 中抽取实体
- **动态图谱**：实体和关系随时间演化，需要时间感知的图谱
- **Self-supervised 优化**：LLM 自我评估抽取质量，持续改进
- **Agentic 构建**：AI Agent 主动搜索缺失知识，补充图谱覆盖空白
