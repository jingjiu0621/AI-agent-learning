# 6.5.4 记忆去重与冲突消解 (Memory Deduplication and Conflict Resolution)

## 简单介绍

记忆去重与冲突消解是记忆整合管线（Memory Consolidation Pipeline）中的第四步，紧接在语义抽象（6.5.2）与层次合并（6.5.3）之后。当 AI Agent 从多轮对话、多数据源、多次交互中积累了大量记忆条目后，其中必然存在三类冗余：

- **完全重复**：同一事实被完全相同的文字记录多次
- **近似重复**：同一事实被不同措辞记录（例如 "用户居住在深圳" 与 "用户常住地在深圳"）
- **语义冲突**：两条记忆对同一实体表达矛盾的信息（例如 "用户年龄 28 岁" 与 "用户年龄 30 岁"）

不做去重的后果是：记忆存储膨胀、检索结果被重复项污染、LLM 推理时看到矛盾信息导致困惑。不做冲突消解的后果是：Agent 可能同时相信两个矛盾的事实，导致行为不一致。

去重管线的目标是在**信息保留最大化**与**存储冗余最小化**之间取得平衡，同时对冲突信息给出可解释的消解策略。

---

## 基本原理

去重与冲突消解的核心流程可抽象为三步：

```
原始记忆池
     │
     ▼
┌──────────────────┐
│   Step 1: 检测    │  ← 计算两两相似度或哈希碰撞
│  (Detection)      │
└────────┬─────────┘
         ▼
┌──────────────────┐
│   Step 2: 判定    │  ← 根据阈值 / 规则决定是否重复或冲突
│  (Judgment)       │
└────────┬─────────┘
         ▼
┌──────────────────┐
│   Step 3: 动作    │  ← 合并、丢弃、标记冲突或保留双版本
│  (Action)         │
└──────────────────┘
```

**检测**依赖于一种"指纹"（fingerprint）机制，将记忆内容映射到一个可比较的表示上。表示越简洁，比较越快，但信息损失也越大。这是去重设计中固有的权衡。

**判定**依赖阈值：两个指纹的相似度超过某个阈值则视为重复。阈值设得太低会导致漏检（false negative），设得太高会导致误杀（false positive）。

**动作**包括：
- **合并**：将两条相似记忆融合为一条（通常涉及 6.5.2 的语义抽象）
- **丢弃**：删除冗余副本，保留质量更高的版本
- **标记冲突**：保留双方但打上 `conflict` 标签，留待 LLM 裁决或人工处理
- **保留双版本**：当无法判断哪一个正确时，保持两条记录并在检索时一同返回

---

## 背景与演进

去重技术的发展经历了三个主要阶段，每一阶段对记忆系统的适用场景不同。

```
阶段 1：精确去重 (Exact Dedup)
     │
     │  Hash-based (SHA-256, MD5)
     │  只能发现 100% 相同的文本
     │  无法处理措辞变化
     ▼

阶段 2：近似去重 (Near-Dedup)
     │
     │  MinHash, SimHash
     │  能发现词汇级别相似的文本
     │  对同义词替换、语序变化有一定容忍度
     ▼

阶段 3：语义去重 (Semantic Dedup)
     │
     │  Embedding + Cosine Similarity
     │  能发现语义相同但措辞完全不同的文本
     │  计算成本最高，依赖外部模型
     ▼
```

### 第一阶段：精确去重

最早的方案。对每条记忆计算哈希值（如 SHA-256），用哈希表判断是否存在完全相同的记录。

**优点**：
- 极快，O(1) 查找
- 零误报（false positive）

**缺点**：
- 对任何字符级差异敏感——"用户喜欢猫"和"用户喜欢猫。"被视为不同
- 无法发现任何形式的近似重复

**适用场景**：日志去重、缓存去重、系统消息去重。

### 第二阶段：近似去重

为了解决措辞变化的问题，研究者引入了 MinHash 和 SimHash 等局部敏感哈希（Locality-Sensitive Hashing, LSH）算法。这些算法将相似的内容以高概率映射到相同的桶中。

**MinHash**：基于 Jaccard 相似度的概率估计方法。将文本分词后，对每个文档的 shingle 集合计算多个哈希函数的最小值，形成签名向量。两个文档签名的相似度 ≈ 它们 Jaccard 相似度。

**SimHash**：基于向量空间的方法。将文本的每个特征（词）哈希为固定长度的位向量，加权求和后取符号位，生成一个指纹。两个 SimHash 指纹的汉明距离越小，文本越相似。

**优缺点**：比精确去重大幅提升了召回率，但对同义词替换（如"电脑"→"计算机"）依然无能为力。

### 第三阶段：语义去重

随着 embedding 模型（如 text-embedding-3-small, BGE, E5）的成熟，基于语义相似度的去重成为可能。每条记忆被编码为稠密向量，两两计算余弦相似度，超过阈值的视为重复。

**优点**：能发现"我喜欢编程"与"我对写代码很有热情"之间的语义相似性。

**缺点**：计算成本高；embedding 质量依赖模型和领域；阈值选择敏感。

---

## 核心矛盾

去重系统中存在一组无法完全消除的张力：

```
信息保留 (Information Preservation)
        ▲
        │    ╲ 理想曲线
        │     ╲
        │      ╲
        │       ╲████ 过度去重区
        │        ████
        └─────────────────────► 去重力度 (Dedup Aggressiveness)
```

- **去重力度过大**：高相似度阈值（如 >0.95）导致大量近似重复被保留，存储膨胀；低阈值（如 <0.70）导致语义相近但信息互补的记忆被误删。
- **信息密度 vs 信息完整性**：去重本质上是有损压缩。每次合并都可能丢失一些细节（如"用户喜欢红色"和"用户喜欢深红色"被合并为"用户喜欢红色"）。
- **时效性冲突**：旧记忆与更新后的记忆看似"重复"，但时间戳不同——直接丢弃旧版本会丢失历史演化信息。

**设计原则**：
1. 对事实性记忆（factual memories）使用高阈值，保守去重
2. 对情境性记忆（episodic memories）使用低阈值，激进去重
3. 重要记忆（高置信度、多源确认）优先保留

---

## 重复检测方法详解

### 精确去重 (Exact Dedup)

基于哈希的实现，简单高效。

```python
import hashlib
from typing import List, Dict, Set

class ExactDeduplicator:
    """基于 SHA-256 的精确去重器"""

    def __init__(self):
        self.seen_hashes: Set[str] = set()

    def _compute_hash(self, text: str) -> str:
        return hashlib.sha256(text.encode("utf-8")).hexdigest()

    def is_duplicate(self, text: str) -> bool:
        h = self._compute_hash(text)
        if h in self.seen_hashes:
            return True
        self.seen_hashes.add(h)
        return False

    def dedup(self, memories: List[str]) -> List[str]:
        results = []
        for m in memories:
            if not self.is_duplicate(m):
                results.append(m)
        return results
```

**局限**：仅适用于完全相同的内容。在真实 Agent 场景中，LLM 生成的记忆几乎不可能出现逐字重复，精确去重的作用非常有限。

---

### 近似去重 (Near-Dedup)

#### MinHash 实现

```python
import hashlib
from typing import List, Set, Tuple
import random

def shingle(text: str, k: int = 3) -> Set[str]:
    """生成 k-shingle 集合"""
    return {text[i:i+k] for i in range(len(text) - k + 1)}

def minhash_signature(shingles: Set[str], num_hashes: int = 128) -> List[int]:
    """计算 MinHash 签名"""
    # 为每个哈希函数生成随机种子
    seeds = [random.randint(0, 2**32 - 1) for _ in range(num_hashes)]
    signature = []

    for seed in seeds:
        min_hash = float("inf")
        for s in shingles:
            h = hashlib.md5((s + str(seed)).encode()).hexdigest()
            val = int(h, 16)
            if val < min_hash:
                min_hash = val
        signature.append(min_hash)

    return signature

def minhash_similarity(sig_a: List[int], sig_b: List[int]) -> float:
    """估计 Jaccard 相似度"""
    matches = sum(1 for a, b in zip(sig_a, sig_b) if a == b)
    return matches / len(sig_a)
```

#### SimHash 实现

```python
from typing import List, Dict
import hashlib

def simhash(text: str, bits: int = 64) -> int:
    """计算文本的 SimHash 指纹"""
    # 分词并加权
    tokens = text.split()
    v = [0] * bits

    for token in tokens:
        # 计算 token 的哈希
        h = int(hashlib.md5(token.encode()).hexdigest(), 16)
        # 简单权重：词频
        weight = 1.0

        for i in range(bits):
            bitmask = 1 << i
            if h & bitmask:
                v[i] += weight
            else:
                v[i] -= weight

    # 生成指纹
    fingerprint = 0
    for i in range(bits):
        if v[i] > 0:
            fingerprint |= (1 << i)

    return fingerprint

def hamming_distance(a: int, b: int) -> int:
    """计算汉明距离"""
    return bin(a ^ b).count("1")

def simhash_similarity(a: int, b: int, bits: int = 64) -> float:
    """SimHash 相似度：1 - (汉明距离 / 总位数)"""
    dist = hamming_distance(a, b)
    return 1.0 - (dist / bits)
```

---

### 语义去重 (Semantic Dedup)

使用 embedding 模型和余弦相似度进行去重。

```python
from typing import List, Tuple
import numpy as np

class SemanticDeduplicator:
    """基于 embedding 余弦相似度的语义去重器"""

    def __init__(self, similarity_threshold: float = 0.92):
        self.threshold = similarity_threshold
        self.stored_embeddings: List[Tuple[str, np.ndarray]] = []

    def _get_embedding(self, text: str) -> np.ndarray:
        """获取文本的 embedding 向量。
        实际使用中会调用外部模型如 OpenAI/text-embedding-3-small
        或本地模型如 BAAI/bge-small-zh-v1.5。
        """
        # 这里假设已有一个 embedding 函数
        # return embedding_model.encode(text)
        raise NotImplementedError("需要接入实际的 embedding 模型")

    def cosine_similarity(self, a: np.ndarray, b: np.ndarray) -> float:
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

    def find_similar(self, text: str) -> List[Tuple[str, float]]:
        """查找与 text 语义相似的已存储记忆"""
        emb = self._get_embedding(text)
        results = []

        for stored_text, stored_emb in self.stored_embeddings:
            sim = self.cosine_similarity(emb, stored_emb)
            if sim >= self.threshold:
                results.append((stored_text, sim))

        return sorted(results, key=lambda x: x[1], reverse=True)

    def add_memory(self, text: str) -> bool:
        """尝试添加记忆，如果与已有记忆相似度过高则拒绝"""
        similar = self.find_similar(text)
        if similar:
            return False  # 重复，拒绝添加
        self.stored_embeddings.append(
            (text, self._get_embedding(text))
        )
        return True

    def dedup_batch(self, memories: List[str]) -> List[str]:
        """批量去重，返回去重后的记忆列表"""
        self.stored_embeddings = []
        deduped = []

        for m in memories:
            if self.add_memory(m):
                deduped.append(m)

        return deduped
```

**批量聚类去重**（更高效的做法）：

```python
from sklearn.cluster import DBSCAN

def cluster_dedup(memories: List[str], embeddings: np.ndarray,
                  eps: float = 0.3, min_samples: int = 1) -> List[str]:
    """
    使用 DBSCAN 聚类进行去重。
    每个聚类中只保留一个代表性记忆（通常选择距离质心最近或置信度最高的）。
    """
    clustering = DBSCAN(metric="cosine", eps=eps,
                        min_samples=min_samples).fit(embeddings)

    # 每个聚类选取一个代表
    representatives = {}
    for i, label in enumerate(clustering.labels_):
        if label == -1:
            continue  # 噪声点，独立保留
        if label not in representatives:
            representatives[label] = memories[i]
        # 可在这里补充择优逻辑

    noise = [memories[i] for i, l in enumerate(clustering.labels_) if l == -1]
    return list(representatives.values()) + noise
```

---

## 冲突检测与消解策略

冲突检测比重复检测更困难——两条记忆看起来不同，甚至措辞完全不同，却在指称同一实体时表达了矛盾信息。

**冲突类型**：

| 类型 | 示例 | 检测难度 |
|------|------|----------|
| 数值矛盾 | "年龄 28" vs "年龄 30" | 中 |
| 关系矛盾 | "工作在 Google" vs "工作在 Meta" | 高 |
| 属性矛盾 | "喜欢猫" vs "对猫过敏" | 高 |
| 时间矛盾 | "2024 年住北京" vs "2024 年住上海" | 高（需时间推理） |

### 策略一：基于时间戳 (Timestamp-based)

最简单的消解策略。维护每条记忆的 `created_at` 和 `updated_at` 时间戳。

```python
class TimestampResolver:
    """时间戳冲突消解器"""

    def resolve(self, memory_a: dict, memory_b: dict) -> dict:
        """默认策略：保留更新的那条记忆"""
        if memory_a["updated_at"] > memory_b["updated_at"]:
            return memory_a
        return memory_b
```

**新版本优先 (Newer Wins)**：适用于用户个人信息更新、偏好变化。例如用户换了工作，新记忆应覆盖旧记忆。

**旧版本优先 (Older Wins)**：适用于基础性事实。例如用户的出生日期，首次记录通常更可靠。

### 策略二：基于置信度 (Confidence-based)

每条记忆附带一个置信度分数（0.0 ~ 1.0），来自信息来源的可信度评估。

```python
class ConfidenceResolver:
    """置信度冲突消解器"""

    def resolve(self, memory_a: dict, memory_b: dict) -> dict:
        if memory_a["confidence"] >= memory_b["confidence"]:
            return memory_a
        return memory_b
```

**置信度来源**：
- 用户直接陈述（confidence = 0.95）
- Agent 从对话中推断（confidence = 0.70）
- 多源交叉验证（confidence 提升至 0.90+）
- 单次观测（confidence = 0.50）
- 默认值/猜测（confidence = 0.30）

### 策略三：基于来源优先级 (Source Priority)

不同来源具有固有优先级层级：

```
用户明确陈述 (User-stated)
       ↑
Agent 观察推断 (Agent-inferred)
       ↑
跨会话聚合 (Cross-session)
       ↑
单次会话提取 (Single-session)
       ↑
系统默认值 (System default)
```

### 策略四：基于 LLM 裁决 (LLM-based Adjudication)

当规则无法确定时，将冲突记忆交给 LLM 判断。

```python
def llm_adjudicate(memory_a: str, memory_b: str,
                   context: str = "") -> dict:
    """
    使用 LLM 裁决冲突记忆。
    返回裁决结果，包含选择、理由和新记忆文本。
    """
    prompt = f"""你正在处理 AI Agent 记忆系统中的两条冲突记忆。
请分析它们是否真正冲突，以及哪一条更可信。

记忆 A：{memory_a}
记忆 B：{memory_b}
{% if context %}
额外上下文：{context}
{% endif %}

请按以下 JSON 格式输出裁决结果：
{{
    "is_conflict": true/false,
    "resolution": "keep_a" / "keep_b" / "merge" / "keep_both",
    "reasoning": "解释你的判断依据",
    "merged_text": "如果选择 merge，给出融合后的记忆文本"
}}
"""
    # 调用 LLM
    # response = llm_client.chat(prompt)
    # return json.loads(response)
    raise NotImplementedError
```

### 策略五：保留双版本 (Keep Both)

当无法判断时，保留两份记忆并标记冲突关系。在检索时同时返回，让 LLM 在问答时刻动态处理。

```python
def mark_conflict(memory_a_id: str, memory_b_id: str,
                  conflict_type: str) -> dict:
    """创建冲突记录"""
    return {
        "type": "conflict_edge",
        "memory_a": memory_a_id,
        "memory_b": memory_b_id,
        "conflict_type": conflict_type,  # "contradiction", "temporal", "source_mismatch"
        "resolved": False,
        "created_at": "2026-07-16T00:00:00Z"
    }
```

---

## 完整去重管线代码示例

以下是一个集成了多层去重策略的管线：

```python
"""
memory_dedup_pipeline.py
AI Agent 记忆去重与冲突消解管线
"""

import hashlib
from typing import List, Dict, Optional, Tuple
from dataclasses import dataclass, field
from enum import Enum
import numpy as np


class DedupLevel(Enum):
    EXACT = 1         # 精确哈希去重
    NEAR = 2          # 近似去重 (MinHash)
    SEMANTIC = 3      # 语义去重 (Embedding)


class ConflictStrategy(Enum):
    TIMESTAMP_NEWER = "timestamp_newer"
    TIMESTAMP_OLDER = "timestamp_older"
    CONFIDENCE = "confidence"
    SOURCE_PRIORITY = "source_priority"
    LLM_ADJUDICATE = "llm_adjudicate"
    KEEP_BOTH = "keep_both"


@dataclass
class Memory:
    id: str
    text: str
    entity: Optional[str] = None
    confidence: float = 0.5
    source_priority: int = 0  # 值越大优先级越高
    created_at: str = ""
    updated_at: str = ""
    embedding: Optional[np.ndarray] = None
    tags: List[str] = field(default_factory=list)


class MultiLayerDedupPipeline:
    """
    多层去重管线：
    第一关：精确哈希去重（快速过滤）
    第二关：SimHash 近似去重（中等粒度）
    第三关：Embedding 语义去重（精细粒度，仅在前两关通过后触发）
    """

    def __init__(self,
                 simhash_threshold: float = 0.85,
                 semantic_threshold: float = 0.92):

        self.exact_hashes: set = set()
        self.simhash_threshold = simhash_threshold
        self.semantic_threshold = semantic_threshold
        self.memories: List[Memory] = []

    def _exact_hash(self, text: str) -> str:
        return hashlib.sha256(text.encode()).hexdigest()

    def _simhash(self, text: str) -> int:
        tokens = text.split()
        v = [0] * 64
        for token in tokens:
            h = int(hashlib.md5(token.encode()).hexdigest(), 16)
            for i in range(64):
                if h & (1 << i):
                    v[i] += 1
                else:
                    v[i] -= 1
        fp = 0
        for i in range(64):
            if v[i] > 0:
                fp |= (1 << i)
        return fp

    def _simhash_similarity(self, a: int, b: int) -> float:
        return 1.0 - (bin(a ^ b).count("1") / 64)

    def _embed_and_compare(self, text: str) -> List[Tuple[Memory, float]]:
        """与已有记忆进行语义相似度比较"""
        # 实际实现需要调用 embedding 模型
        conflicts = []
        for mem in self.memories:
            if mem.embedding is not None:
                sim = self._cosine_similarity(
                    self._get_embedding(text), mem.embedding
                )
                if sim >= self.semantic_threshold:
                    conflicts.append((mem, sim))
        return conflicts

    def _cosine_similarity(self, a: np.ndarray, b: np.ndarray) -> float:
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

    def _get_embedding(self, text: str) -> np.ndarray:
        # 实际模型调用
        raise NotImplementedError

    def add_memory(self, memory: Memory) -> Tuple[bool, str]:
        """
        尝试添加一条记忆。返回 (是否成功, 状态信息)。
        状态信息说明是重复、冲突还是成功添加。
        """
        # 第一关：精确去重
        h = self._exact_hash(memory.text)
        if h in self.exact_hashes:
            return False, "exact_duplicate"

        # 第二关：SimHash 近似去重
        new_simhash = self._simhash(memory.text)
        for existing in self.memories:
            sim = self._simhash_similarity(
                new_simhash, self._simhash(existing.text)
            )
            if sim >= self.simhash_threshold:
                # 第三关：语义去重精检
                semantic_matches = self._embed_and_compare(memory.text)
                if semantic_matches:
                    return False, f"semantic_duplicate (sim={semantic_matches[0][1]:.3f})"

        # 去重通过，添加记忆
        self.exact_hashes.add(h)
        self.memories.append(memory)
        return True, "added"

    def detect_conflicts(self, entity: str) -> List[Tuple[Memory, Memory, float]]:
        """检测指定实体的所有冲突记忆"""
        entity_memories = [
            m for m in self.memories if m.entity == entity
        ]
        conflicts = []

        for i in range(len(entity_memories)):
            for j in range(i + 1, len(entity_memories)):
                a, b = entity_memories[i], entity_memories[j]
                # 使用 embedding 检测冲突
                sim = self._cosine_similarity(
                    a.embedding, b.embedding
                )
                if sim < 0.5 and a.entity == b.entity:
                    # 同实体但语义差异大 → 可能冲突
                    conflicts.append((a, b, sim))

        return conflicts


# 使用示例
def demo():
    pipeline = MultiLayerDedupPipeline()

    mem1 = Memory(
        id="1", text="用户张三年薪约 50 万", entity="张三",
        confidence=0.8, source_priority=2
    )
    mem2 = Memory(
        id="2", text="张三年薪大约 50 万人民币", entity="张三",
        confidence=0.7, source_priority=1
    )
    mem3 = Memory(
        id="3", text="用户张三年薪 80 万", entity="张三",
        confidence=0.9, source_priority=3
    )

    print(pipeline.add_memory(mem1))  # (True, "added")
    print(pipeline.add_memory(mem2))  # (False, "near_duplicate") 或 "semantic_duplicate"
    print(pipeline.add_memory(mem3))  # (True, "added") 但需要检测冲突

    conflicts = pipeline.detect_conflicts("张三")
    for a, b, sim in conflicts:
        print(f"冲突检测: [{a.text}] ↔ [{b.text}], 语义差异={sim:.3f}")


if __name__ == "__main__":
    demo()
```

---

## 去重评估指标

评估去重系统的质量需要一套多维度的指标：

### 核心指标

| 指标 | 公式 | 说明 |
|------|------|------|
| **压缩率** | `1 - (去重后数量 / 去重前数量)` | 衡量存储节省程度 |
| **去重召回率** | `被正确识别的重复数 / 实际重复总数` | 衡量去重系统发现重复的能力 |
| **去重精确率** | `被正确识别的重复数 / 被标记为重复的总数` | 衡量误报率，值越高说明冤枉的记忆越少 |
| **F1 分数** | `2 * (P * R) / (P + R)` | 综合精确率与召回率 |

### 实操指标

```python
def evaluate_dedup(gold_standard: List[Tuple[int, int]],
                   predicted_duplicates: List[Tuple[int, int]],
                   total_memories: int) -> Dict:
    """
    评估去重效果。

    gold_standard: 人工标注的重复对列表，每个元素是 (idx_a, idx_b)
    predicted_duplicates: 系统检测出的重复对列表
    total_memories: 记忆总数
    """
    gold_set = set(gold_standard)
    pred_set = set(predicted_duplicates)

    true_positives = len(gold_set & pred_set)
    false_positives = len(pred_set - gold_set)
    false_negatives = len(gold_set - pred_set)

    precision = true_positives / (true_positives + false_positives) if (true_positives + false_positives) > 0 else 0
    recall = true_positives / (true_positives + false_negatives) if (true_positives + false_negatives) > 0 else 0
    f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0

    # 压缩率
    n_after = total_memories - true_positives
    compression_rate = 1 - (n_after / total_memories)

    return {
        "precision": round(precision, 4),
        "recall": round(recall, 4),
        "f1": round(f1, 4),
        "compression_rate": round(compression_rate, 4),
        "true_positives": true_positives,
        "false_positives": false_positives,
        "false_negatives": false_negatives,
    }
```

---

## 工程优化

在实际的 Agent 记忆系统中，去重需要在性能和效果之间做出工程取舍。

### 批量去重 (Batch Dedup)

适用于定时触发的整合任务。将所有新记忆收集到一个批次中，统一进行聚类去重。

**优点**：可以利用矩阵运算加速（如 GPU 批量计算 embedding 相似度矩阵），聚类效果更好（能看到全局分布）。

**缺点**：延迟高，不适合实时场景。

```python
def batch_semantic_dedup(memories: List[str], batch_size: int = 1000):
    """批量语义去重"""
    all_embeddings = []
    for i in range(0, len(memories), batch_size):
        batch = memories[i:i+batch_size]
        # batch_emb = model.encode(batch)
        # all_embeddings.extend(batch_emb)
        pass

    # 用 FAISS 或 Scipy 做批量相似度搜索
    # ...
```

### 增量去重 (Incremental Dedup)

每次新记忆到达时立即去重，仅与新记忆进行比较。

**优点**：延迟低，适合在线 Agent 系统。

**缺点**：可能错过跨批次的重复（两条记忆在不同批次到达但互相重复），需要额外的维护机制。

**优化技巧**：为新记忆维护一个滑动窗口，不仅与全部历史比较，也优先与最近的 N 条记忆比较（因为现实场景中重复往往发生在时间上相邻的记忆之间）。

```python
class IncrementalDeduplicator:
    """增量去重器，结合缓存加速"""

    def __init__(self, cache_size: int = 10000):
        self.embedding_cache = {}      # memory_id → embedding
        self.simhash_cache = {}        # memory_id → simhash
        self.recent_window = []        # 滑动窗口
        self.window_size = 200

    def add_and_check(self, new_memory: Memory) -> bool:
        # 1. 先在滑动窗口中快速检查
        for recent_id in self.recent_window:
            if self._quick_compare(new_memory.id, recent_id):
                return False  # 重复

        # 2. 再在全局缓存中检查（抽样）
        # ...

        # 3. 添加成功后更新缓存和窗口
        self.recent_window.append(new_memory.id)
        if len(self.recent_window) > self.window_size:
            self.recent_window.pop(0)
        return True
```

### 分片级去重 (Shard-Level Dedup)

当记忆库规模达到百万级时，全量两两比较不可行。需要将记忆分片（shard），在 shard 内部进行去重，再在 shard 间进行有限比较。

**分片策略**：
- **实体分片**：按实体名称分片（如按用户 ID 分片）
- **时间分片**：按时间窗口分片（如按天分片）
- **哈希分片**：用 Locality-Sensitive Hashing 将相似记忆分到同一 shard

```
全局记忆库 (10M 条)
     │
     ├── Shard 0: 实体 A-C  (2.5M 条)
     ├── Shard 1: 实体 D-F  (2.5M 条)
     ├── Shard 2: 实体 G-K  (2.5M 条)
     └── Shard 3: 实体 L-Z  (2.5M 条)
          │
          ▼
    各 Shard 内独立去重
          │
     ┌────┴────┐
     ▼         ▼
   Shard 间跨边界去重（抽样比较 + LSH 加速）
```

### 向量索引加速

对于语义去重，使用近似最近邻（ANN）索引代替暴力搜索：

```python
# 使用 FAISS 加速语义去重
import faiss
import numpy as np

class FAISSDeduplicator:
    def __init__(self, dim: int = 768):
        self.index = faiss.IndexFlatIP(dim)  # 内积索引（配合归一化即为余弦相似度）
        self.memories = []

    def add(self, text: str, embedding: np.ndarray):
        self.index.add(embedding.reshape(1, -1))
        self.memories.append(text)

    def find_duplicates(self, embedding: np.ndarray,
                        threshold: float = 0.92) -> List[str]:
        similarities, indices = self.index.search(
            embedding.reshape(1, -1), k=10
        )
        results = []
        for sim, idx in zip(similarities[0], indices[0]):
            if sim >= threshold and idx != -1:
                results.append(self.memories[idx])
        return results
```

---

## 能力边界

即使是最先进的语义去重，也存在以下限制：

### 1. 语义去重的局限性

- **上下文敏感性**："用户喜欢苹果"可以指水果也可以指公司。embedding 模型可能无法区分这种多义性，导致错误地去重或错误地保留。
- **领域漂移**：在一个领域训练的 embedding 模型在另一个领域可能表现不佳。例如通用中文 embedding 在法律领域可能无法准确捕捉术语的细微差异。
- **长度依赖**：短文本（如 <10 个 token）的 embedding 质量往往不稳定，余弦相似度可能无法反映真实语义关系。

### 2. 对抗性重复 (Adversarial Duplicates)

- 有意构造的不同措辞表达同一事实（例如测试去重系统的鲁棒性时）
- 同义词轮换式改写可通过 SimHash 和语义去重，但极端的句式重组（如主动变被动、长句拆短句）可能导致 embedding 相似度意外降低

### 3. 交叉实体混淆

去重系统可能将分属不同实体的相似记忆错误合并：

```
记忆 A: "张三毕业于北京大学"
记忆 B: "李四毕业于北京大学"
→ 语义相似度很高，但实体不同，不应去重！
```

解决方案：去重必须在**同实体**范围内进行。在去重前先进行实体解析（entity resolution），确保只有同一实体的记忆参与比较。

### 4. 时效性盲区

语义去重不考虑时间维度。一条 2023 年和一条 2026 年的记忆即使语义相同，也不应该简单合并——用户的信息可能已经变化。

---

## 最大挑战

### 过度去重导致的信息丢失 (Over-Deduplication)

这是去重系统中最常见也最危险的问题。

**典型案例**：

```
记忆 A: "用户喜欢在周末打篮球"
记忆 B: "用户最近膝盖受伤，暂时不能打篮球"

去重阈值 = 0.90 时：
  cosine(A, B) = 0.82 < 0.90 → 保留两条 ✓

去重阈值 = 0.75 时：
  cosine(A, B) = 0.82 > 0.75 → 视为重复，丢弃 B ✗
  → 信息丢失！
```

两条记忆看似"重复"（都在谈论用户和篮球），但 B 包含了关键的新信息（膝盖受伤）。过度去重导致 Agent 丢失了"用户膝盖受伤"这一重要事实。

**缓解策略**：

1. **分级阈值**：事实性记忆使用高阈值（保守），情境性记忆使用低阈值（激进）
2. **增量信息检测**：在合并前计算"新信息量"——两条记忆的语义差异多大程度上是由于新事实引起的
3. **冲突标记**：当两条记忆"语义足够相似但存在事实矛盾"时，不丢弃任何一条，而是标记为冲突
4. **反事实保留**：对低置信度的去重决策，保留删除的记忆至回收站（soft-delete），允许回滚

```python
def safe_merge(memory_a: Memory, memory_b: Memory,
               semantic_similarity: float,
               info_gain_threshold: float = 0.15) -> Optional[Memory]:
    """
    安全的合并策略。
    - 计算两条记忆的"信息增益"
    - 如果增益超过阈值，视为互补而非重复，保留双方
    - 否则合并
    """
    # 计算信息增益（简化版：使用实体和谓词差异）
    entities_a = set(extract_entities(memory_a.text))
    entities_b = set(extract_entities(memory_b.text))
    new_entities = entities_b - entities_a

    info_gain = len(new_entities) / max(len(entities_a | entities_b), 1)

    if info_gain > info_gain_threshold:
        return None  # 不合并，保留双方

    # 合并逻辑...
    merged_text = f"{memory_a.text}（补充：{memory_b.text}）"
    return Memory(
        id=memory_a.id,
        text=merged_text,
        confidence=max(memory_a.confidence, memory_b.confidence),
    )
```

---

## 与其他模块的关系

去重与冲突消解不是孤立存在的，它与记忆整合管线中的其他步骤紧密互动：

```
6.5.2 语义抽象 (Semantic Abstraction)
    │
    │  提供标准化的事实表示（三元组、标准化实体名）
    │  降低去重的难度——"孙某"和"小孙"被统一为"孙XX"
    ▼
6.5.3 层次合并 (Hierarchical Merge)
    │
    │  将低层细节记忆合并为高层概括记忆
    │  去重在此步骤后进行，避免对概括前的重复内容重复计算
    ▼
6.5.4 去重与冲突消解 (Deduplication)  ← 当前模块
    │
    │  输出：精炼后的无冗余记忆集合 + 冲突记录
    ▼
6.5.5 结构化索引 (Structural Indexing)
    │
    │  接收去重后的干净数据，构建检索索引
    │  去重不彻底会导致索引膨胀和检索噪声
    ▼
6.5.6 记忆衰减 (Memory Decay)
    │
    │  对低频、低置信度的记忆进行衰减
    │  冲突双方中有一方衰减后可自动消解冲突
    ▼
(工作记忆 / 长期记忆存储)
```

**关键依赖关系**：

- **语义抽象在先**：如果不去抽象，同一实体的不同指称（"张总"、"张三"、"Zhang San"）将导致去重器无法正确关联两条记忆。抽象后的标准化表述大幅提升去重准确率。
- **层次合并在先**：层次合并本身也会产生冗余（父节点概括了子节点的内容），这些冗余需要在去重步骤中清理。如果在合并前去重，底层细节的去重要求与高层概括不同，容易误判。
- **结构化索引在后**：去重决定了索引质量。冗余记忆会导致索引中出现多条指涉同一事实的条目，影响检索排名和准确性。

### 与冲突消解的联动

冲突消解与去重共享检测阶段，但目标不同：

| 方面 | 去重 | 冲突消解 |
|------|------|----------|
| 目标 | 消除冗余 | 解决矛盾 |
| 相似度 | 高相似度（>0.85） | 低相似度 + 同实体（<0.50） |
| 动作 | 合并或丢弃 | 裁决或标记 |
| 风险 | 信息丢失 | 信息不一致 |

两者可以协同工作：先去重（合并高度相似的冗余），再检测冲突（在同实体低相似记忆中寻找矛盾）。

---

## 场景判断

不同的记忆类型和场景需要不同的去重策略组合。以下是推荐矩阵：

| 记忆类型 | 推荐去重策略 | 阈值建议 | 冲突策略 | 说明 |
|----------|-------------|----------|----------|------|
| **用户个人信息**（年龄、职业、住址） | Exact + Semantic | 0.95 | Timestamp Newer | 信息会更新，新版本覆盖旧版本 |
| **用户偏好**（喜欢、不喜欢、习惯） | Semantic | 0.90 | Confidence + Keep Both | 偏好可能矛盾（如既喜欢宁静又喜欢刺激），不宜武断覆盖 |
| **事实知识**（实体关系、属性） | Exact + Near | 0.85 | LLM Adjudicate | 事实矛盾需要推理判断 |
| **对话历史摘要** | Semantic + Cluster | 0.80 | Keep Both | 保留多角度摘要更安全 |
| **系统状态**（任务进度、完成状态） | Exact | 1.0 | Timestamp Newer | 状态必须精确，不允许近似匹配 |
| **工具调用记录**（API 响应、计算结果） | Exact | 1.0 | Timestamp Newer | 精确匹配，新结果替换旧结果 |
| **长期知识**（世界观、常识） | Semantic | 0.95 | Source Priority | 高优先级来源覆盖低优先级 |
| **情境记忆**（某次交互的详细记录） | Near + Semantic | 0.80 | Keep Both | 情境记忆天生有重叠，但应保留细节 |

### 决策树

```
新记忆到达
    │
    ├─ 实体是否已存在？
    │   ├─ 否 → 直接添加，无需去重
    │   └─ 是 → 进入去重检测
    │
    ├─ 与已有记忆是否精确匹配？
    │   ├─ 是 → 丢弃新记忆，更新旧记忆的元数据（时间戳、频率）
    │   └─ 否 → 进入近似匹配
    │
    ├─ 与已有记忆是否近似匹配 (SimHash > T_near)？
    │   ├─ 是 → 进入语义精检
    │   │       ├─ 语义重复 → 合并或丢弃
    │   │       └─ 语义不同 → 保留（可能为冲突，进入冲突检测）
    │   └─ 否 → 直接添加
    │
    └─ 冲突检测：同实体低相似记忆
        ├─ 确认冲突 → 按策略消解
        └─ 不冲突 → 保留
```

---

## 小结

记忆去重与冲突消解是 AI Agent 长期记忆系统中保证数据质量的关键步骤。从精确哈希到语义 embedding，每种方法都有自己的适用边界和权衡。工程实践中推荐构建**多层流水线**：先以低成本哈希快速过滤，再以 SimHash 做中等粒度检测，最后以 embedding 做精细语义判断。同时在冲突消解上采用**分级策略**：简单冲突用规则（时间戳、置信度），复杂冲突用 LLM 裁决。

最重要的原则是**安全第一**：宁可保留少量冗余，不要因为过度去重丢失关键信息。去重的目标不是"让记忆库最小化"，而是"让记忆库最有价值"。

---

## 延伸阅读

- 同一模块的 6.5.2 (semantic-abstraction) 和 6.5.3 (hierarchical-merge)
- Charikar, M. S. (2002). "Similarity Estimation Techniques from Rounding Algorithms" — SimHash 原始论文
- Broder, A. Z. (1997). "On the resemblance and containment of documents" — MinHash 原始论文
- NVIDIA NeMo Curator — 大规模语义去重的工业级实现
- FAISS (Facebook AI Similarity Search) — 向量索引加速库
