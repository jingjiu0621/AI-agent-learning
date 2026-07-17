# 9.1.1 chunking-strategies — 分块策略：固定大小 / 语义 / 递归

## 简单介绍

分块（Chunking）是将原始文档切分成可索引、可检索的文本片段的过程。这是 RAG 系统索引构建的第一步——决定了后续检索的最细粒度。分块策略的核心矛盾是：**块太大则噪声多且浪费上下文预算，块太小则语义不完整且丢失上下文关联**。

## 基本原理

### 分块的核心维度

```
原始文档
    │
    ├── 按字符/Token 切分（Fixed-size）── 简单快速，但可能切碎语义
    │
    ├── 按语义边界切分（Semantic）  ── 尽量保持语义完整
    │   ├── 句子边界 (sentence_splitter)
    │   ├── 段落边界 (paragraph_split)
    │   └── 文档结构 (Markdown header / LaTeX section)
    │
    └── 递归切分（Recursive）      ── 从粗到细，适配内容结构
        ├── 先按文档结构 → 再按段落 → 最后按 token 预算
        └── 如果块太大，递归切分直到满足大小要求
```

### 三种核心策略对比

| 策略 | 实现难度 | 语义完整性 | 一致性 | 适用场景 |
|------|---------|-----------|--------|---------|
| 固定大小分块 | ⭐ 低 | ⭐ 低 | ⭐⭐⭐ 高 | 通用兜底方案 |
| 语义分块 | ⭐⭐⭐ 中 | ⭐⭐⭐ 高 | ⭐⭐ 中 | 文档结构清晰的场景 |
| 递归分块 | ⭐⭐⭐⭐ 高 | ⭐⭐⭐⭐ 很高 | ⭐⭐⭐ 高 | 异构内容混合的场景 |

### 固定大小分块

```python
# 基于 Token 的固定大小分块（使用 tiktoken）
import tiktoken

def fixed_size_chunk(text: str, chunk_size: int = 512, overlap: int = 50):
    """固定 token 数量分块，带重叠"""
    enc = tiktoken.get_encoding("cl100k_base")
    tokens = enc.encode(text)
    chunks = []
    
    start = 0
    while start < len(tokens):
        end = start + chunk_size
        chunk_tokens = tokens[start:end]
        chunk_text = enc.decode(chunk_tokens)
        chunks.append({
            "text": chunk_text,
            "start_token": start,
            "end_token": end,
            "token_count": len(chunk_tokens)
        })
        start += chunk_size - overlap  # 滑动窗口
    
    return chunks
```

### 递归分块（LangChain RecursiveCharacterTextSplitter 核心逻辑）

```python
def recursive_chunk(text: str, chunk_size: int = 1000, chunk_overlap: int = 200):
    """按分隔符优先级递归分块"""
    separators = ["\n\n", "\n", "。", ".", " ", ""]  # 从粗到细
    
    def split_at(text: str, separator: str, size: int) -> list:
        """在分隔符处切分"""
        parts = []
        current = []
        current_len = 0
        
        for segment in text.split(separator):
            seg_len = len(segment)
            if current_len + seg_len > size and current:
                parts.append(separator.join(current))
                # 保持 overlap：保留末尾内容
                overlap_content = current[-max(1, len(current)//4):]
                current = overlap_content
                current_len = sum(len(s) for s in overlap_content)
            current.append(segment)
            current_len += seg_len
        
        if current:
            parts.append(separator.join(current))
        return parts
    
    result = [text]
    for sep in separators:
        new_result = []
        for chunk in result:
            if len(chunk) > chunk_size:
                new_result.extend(split_at(chunk, sep, chunk_size))
            else:
                new_result.append(chunk)
        result = new_result
        if all(len(c) <= chunk_size for c in result):
            break
    
    return result
```

### 语义分块（Semantic Chunking）

```python
def semantic_chunk(text: str, model, threshold: float = 0.8):
    """基于语义相似度的分块：边界处语义变化大 → 切分"""
    sentences = split_into_sentences(text)
    chunks = []
    current_chunk = [sentences[0]]
    
    for i in range(1, len(sentences)):
        # 计算当前句子与当前块的语义相似度
        chunk_text = " ".join(current_chunk)
        emb_chunk = model.encode(chunk_text)
        emb_sentence = model.encode(sentences[i])
        
        similarity = cosine_similarity(emb_chunk, emb_sentence)
        
        if similarity < threshold:  # 语义变化大 → 新块
            chunks.append(" ".join(current_chunk))
            current_chunk = [sentences[i]]
        else:
            current_chunk.append(sentences[i])
    
    if current_chunk:
        chunks.append(" ".join(current_chunk))
    
    return chunks
```

## 背景与演进

- **固定大小分块（2020-2022）**：早期 RAG 系统（如原始 LangChain）最简单直接的方式就是按 token 数量切分。实现简单但问题明显——经常在句子中间切碎，导致检索到的片段语义不完整。

- **递归字符分块（2023 初）**：LangChain 引入 `RecursiveCharacterTextSplitter`，按照 `\n\n → \n → 。 → 空格` 的优先级递归切分，在保持语义完整和控制块大小之间取得平衡，成为当时事实标准。

- **语义分块（2023 中）**：使用 embedding 相似度检测语义边界（如 LlamaIndex 的 `SemanticSplitterNodeParser`）。当相邻句子语义变化大时切开，更能保持语义一致性。但计算成本高（需要多次调用 embedding 模型）。

- **LLM 辅助分块（2024）**：让 LLM 直接决定分块边界，如 "请将此文档分成语义完整的段落"。质量更高但成本更高。适合高价值文档的高质量索引构建。

- **当前趋势**：没有"万能"分块策略，最佳实践是混合策略——根据文档类型（Markdown、代码、PDF）自动选择分块器，用固定大小作为保底，用语义分块提升质量。

## 核心矛盾

**语义完整性 vs 检索精度**：

```
大块（完整语义）    ───  检索命中精度低（噪声多）
小块（精确命中）    ───  检索上下文不完整（碎片化）
```

详细权衡：

| 块大小 | 优势 | 劣势 | 适合场景 |
|--------|------|------|---------|
| 128 tokens | 精确命中，高精度 | 上下文碎片化严重 | 事实性 QA（"XX 的出生年份"） |
| 256 tokens | 精度和完整性的折中 | 两者都不够极致 | 通用问答 |
| 512 tokens | 大部分场景的 sweet spot | 中等噪声 | 最常用大小 |
| 1024 tokens | 上下文丰富 | 噪声多，精度下降 | 摘要、分析类任务 |
| 2048+ tokens | 完整段落/章节 | 检索精度很低 | 学术论文、长文档分析 |

## 主流优化方向

1. **自适应分块（Adaptive Chunking）**：根据文档内容和类型动态选择块大小，而非固定值。如代码文件用函数边界，Markdown 用标题边界。

2. **分块重叠（Chunk Overlap）**：相邻块之间保留 10-20% 的重叠内容。避免正好在关键信息处切分导致两边都丢失上下文。

   ```python
   # 重叠策略：回溯保留前一块尾部
   chunks = []
   start = 0
   while start < len(doc):
       end = start + chunk_size
       chunk = doc[start:end]
       chunks.append(chunk)
       start = end - overlap_size  # overlap 让切分不那么"硬"
   ```

3. **复合分块（Compound Chunking）**：同时保留粗粒度和细粒度两个版本的块，粗块提供上下文，细块提供精确信息。检索时先找细块再取粗块补充上下文。

4. **LLM 辅助边界判定**：对高价值文档，用 LLM 判断自然段落边界，而非机械按字符切分。

5. **结构感知分块**：利用文档的结构信息（标题层级、列表、表格、代码块）作为分块的天然边界，而非纯文本处理。

## 能力边界与结果边界

| 维度 | 固定大小分块 | 递归分块 | 语义分块 |
|------|-------------|---------|---------|
| 处理速度 | ⭐⭐⭐⭐⭐ 毫秒级 | ⭐⭐⭐⭐ 毫秒级 | ⭐⭐ 秒级（需 embedding） |
| 语义完整性 | ⭐⭐ 常被切碎 | ⭐⭐⭐ 基本完整 | ⭐⭐⭐⭐ 较完整 |
| 实现复杂度 | ⭐⭐⭐⭐⭐ 极简单 | ⭐⭐⭐⭐ 简单 | ⭐⭐ 需要 embedding 模型 |
| 文档类型适应性 | ⭐⭐ 所有文档一样 | ⭐⭐⭐ 部分适配 | ⭐⭐⭐⭐ 内容感知 |
| 确定性 | ⭐⭐⭐⭐⭐ 完全确定 | ⭐⭐⭐⭐⭐ 完全确定 | ⭐⭐ 依赖 embedding |
| 存储效率 | ⭐⭐⭐⭐ 无额外存储 | ⭐⭐⭐⭐ 无额外存储 | ⭐⭐⭐ 需要计算相似度 |

## 与其他分块策略的区别

- **固定大小 vs 递归**：固定大小是盲切，递归是"尝试保持结构完整后再按大小限制切"
- **语义分块 vs 固定大小**：语义分块需要 embedding，质量更高但成本更高
- **语义 vs 递归**：递归基于显式分隔符，语义基于隐式语义变化。递归确定性高，语义灵活性高
- **递归分块 vs LLM 分块**：递归是规则驱动，LLM 是模型驱动，LLM 质量最高但成本和延迟最高

## 核心优势

递归分块在大多数场景下是"最不坏的选择"：它比固定大小更智能，比语义分块更高效，实现复杂度适中。**递归分块 + 适当重叠**是目前生产 RAG 系统最广泛采用的方案。

## 工程优化

1. **选择 chunk_size 的黄金法则**：从 512 tokens 开始，根据评估结果（Recall@K + Faithfulness）上下调整
2. **分块与检索对齐**：chunk_size 应该与检索 Top-K 的上下文预算适配（如 Top-5 × 512 = 2560 tokens ≈ 2K 上下文）
3. **分块 + 引用溯源**：每个分块保留源文档 ID 和位置，便于引用溯源和去重
4. **预计算文档类型**：在预处理阶段检测文档类型，路由到对应分块策略（Markdown / PDF / Code / LaTeX）
5. **分块持久化缓存**：如果文档未变化，缓存分块结果避免重复计算
6. **不同层级不同策略**：摘要索引用粗块（1024+），内容索引用细块（256-512）

## 场景适配指南

| 场景 | 推荐分块策略 | chunk_size | 重叠 |
|------|-------------|-----------|------|
| 客服文档 QA | 递归分块 | 512 | 10% |
| 学术论文检索 | 语义分块 | 1024 | 5% |
| 代码库检索 | 结构感知（函数级别） | 按函数 | 0% |
| 新闻摘要分析 | 固定大小 | 1024 | 20% |
| 法律合同审查 | 递归分块 | 256 | 15% |
| 对话历史检索 | 按轮次分块 | 按消息 | 0% |

## 示例代码：生产级分块管线

```python
from typing import List, Dict, Optional
import tiktoken
import re

class ProductionChunker:
    """生产级分块管线：类型感知 + 递归 + 重叠"""
    
    def __init__(self, default_chunk_size: int = 512):
        self.default_chunk_size = default_chunk_size
        self.encoder = tiktoken.get_encoding("cl100k_base")
    
    def detect_doc_type(self, text: str) -> str:
        """检测文档类型"""
        if re.search(r'^(def |class |import |#!|//)', text, re.MULTILINE):
            return 'code'
        if re.search(r'^#{1,6}\s', text, re.MULTILINE):
            return 'markdown'
        return 'plain'
    
    def chunk(self, text: str, doc_type: Optional[str] = None) -> List[Dict]:
        """根据文档类型选择分块策略"""
        if doc_type is None:
            doc_type = self.detect_doc_type(text)
        
        if doc_type == 'code':
            return self._chunk_code(text)
        elif doc_type == 'markdown':
            return self._chunk_markdown(text)
        else:
            return self._chunk_recursive(text)
    
    def _chunk_code(self, text: str) -> List[Dict]:
        """按函数/类边界分块（保留 import 作为上下文）"""
        chunks = []
        # 正则匹配函数和类定义
        pattern = r'(^(?:def |class |async def ).*?(?=\n(?:def |class |async def )|\Z))'
        matches = re.findall(pattern, text, re.MULTILINE | re.DOTALL)
        
        for i, match in enumerate(matches):
            tokens = self.encoder.encode(match)
            chunks.append({
                "text": match,
                "token_count": len(tokens),
                "type": "code_unit"
            })
        
        # 如果代码块太大（单函数超长），回退到递归分块
        for c in chunks:
            if c["token_count"] > self.default_chunk_size:
                sub_chunks = self._chunk_recursive(c["text"])
                c["sub_chunks"] = sub_chunks
        
        return chunks
    
    def _chunk_markdown(self, text: str) -> List[Dict]:
        """按 Markdown 标题层级分块"""
        chunks = []
        sections = re.split(r'(^#+\s.*?$)', text, flags=re.MULTILINE)
        
        current_chunk = ""
        for part in sections:
            if re.match(r'^#+\s', part):
                if current_chunk:
                    chunks.append(current_chunk)
                current_chunk = part
            else:
                current_chunk += part
        
        if current_chunk:
            chunks.append(current_chunk)
        
        # 检查是否有超大的 section（没有二级标题的长文本）
        result = []
        for chunk in chunks:
            tokens = self.encoder.encode(chunk)
            if len(tokens) > self.default_chunk_size * 2:
                result.extend(self._chunk_recursive(chunk))
            else:
                result.append(chunk)
        
        return [{"text": c, "token_count": len(self.encoder.encode(c)), "type": "md_section"} for c in result]
    
    def _chunk_recursive(self, text: str) -> List[Dict]:
        """递归字符分块（保底方案）"""
        separators = ["\n\n", "\n", "。", ". "]
        
        chunks = [text]
        for sep in separators:
            new_chunks = []
            for chunk in chunks:
                tokens = self.encoder.encode(chunk)
                if len(tokens) <= self.default_chunk_size:
                    new_chunks.append(chunk)
                else:
                    new_chunks.extend(self._split_at(text=chunk, sep=sep))
            chunks = new_chunks
            # 如果所有块都满足大小了，停止
            if all(len(self.encoder.encode(c)) <= self.default_chunk_size for c in chunks):
                break
        
        return [{"text": c, "token_count": len(self.encoder.encode(c)), "type": "recursive"} for c in chunks]
    
    def _split_at(self, text: str, sep: str) -> List[str]:
        """在分隔符处切分，维持大小限制"""
        parts = text.split(sep)
        result = []
        current = []
        current_len = 0
        
        for part in parts:
            part_len = len(self.encoder.encode(part))
            if current_len + part_len > self.default_chunk_size and current:
                result.append(sep.join(current))
                current = current[-1:]  # 保留最后一个元素作为 overlap
                current_len = len(self.encoder.encode(current[0])) if current else 0
            current.append(part)
            current_len += part_len
        
        if current:
            result.append(sep.join(current))
        
        return result
```
