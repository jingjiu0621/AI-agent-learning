# 9.1.4 hierarchical-index — 层次索引：文档 → 章节 → 段落

## 简单介绍

层次索引（Hierarchical Index）保留了文档的原始结构层级——书籍有"章→节→段"，技术文档有"模块→类→方法"，网页有"站点→页面→区域"。与多级索引不同（多级是为内容创建不同粒度的"摘要"），层次索引是**保留并利用文档本身的结构关系**来组织索引，使得检索系统能够沿结构路径从粗到精地定位信息。

## 基本原理

### 层次索引的数据结构

```
文档树（Document Tree）
    │
    ├── Document "A" (根节点)
    │   ├── Section "1" (H1)
    │   │   ├── Subsection "1.1" (H2)
    │   │   │   ├── Paragraph 1  ← 叶子节点（实际检索单元）
    │   │   │   ├── Paragraph 2
    │   │   │   └── Paragraph 3
    │   │   │
    │   │   └── Subsection "1.2" (H2)
    │   │       ├── Paragraph 4
    │   │       ├── Paragraph 5
    │   │       └── Paragraph 6
    │   │
    │   └── Section "2" (H1)
    │       ├── Paragraph 7
    │       └── Paragraph 8
    │
    └── Document "B" (根节点)...
```

### 核心索引策略：父节点 embedding + 子节点引用

```python
class HierarchicalIndex:
    """层次索引"""
    
    class Node:
        def __init__(self, id: str, text: str, level: int, parent=None):
            self.id = id
            self.text = text
            self.level = level     # 0=document, 1=section, 2=paragraph
            self.parent = parent
            self.children = []
            self.embedding = None
            self.heading_path = []  # ["Chapter 1", "Section 2.1"]
    
    def __init__(self, embed_model):
        self.embed_model = embed_model
        self.root = None
        self.nodes_by_id = {}
    
    async def build_from_markdown(self, md_text: str):
        """从 Markdown 构建层次索引"""
        lines = md_text.split("\n")
        stack = []  # 维护节点栈，用于确定父子关系
        
        current_doc = self.Node(id="doc_1", text="", level=0)
        self.root = current_doc
        self.nodes_by_id[current_doc.id] = current_doc
        stack.append(current_doc)
        
        current_text = []
        for line in lines:
            if line.startswith("#"):
                # 保存前面的段落文本
                if current_text:
                    self._add_leaf_paragraph(stack[-1], current_text)
                    current_text = []
                
                # 确定标题层级并更新栈
                heading_level = len(line) - len(line.lstrip("#"))
                level = min(heading_level, 6)  # H1-H6
                
                # 出栈到当前层级的父级
                while len(stack) > 1 and stack[-1].level >= level:
                    stack.pop()
                
                # 创建章节节点
                node = self.Node(
                    id=f"sec_{len(self.nodes_by_id)}",
                    text=line.strip("# "),
                    level=level,
                    parent=stack[-1]
                )
                node.heading_path = stack[-1].heading_path + [line.strip("# ")]
                stack[-1].children.append(node)
                self.nodes_by_id[node.id] = node
                stack.append(node)
            else:
                current_text.append(line)
        
        if current_text:
            self._add_leaf_paragraph(stack[-1], current_text)
    
    def _add_leaf_paragraph(self, parent: Node, text_lines: list):
        text = "\n".join(text_lines).strip()
        if not text:
            return
        node = self.Node(
            id=f"para_{len(self.nodes_by_id)}",
            text=text,
            level=parent.level + 1,
            parent=parent
        )
        node.heading_path = parent.heading_path
        parent.children.append(node)
        self.nodes_by_id[node.id] = node
    
    async def index_all(self):
        """为所有节点生成 embedding"""
        for node in self.nodes_by_id.values():
            if node.text:
                node.embedding = await self.embed_model.embed(node.text)
```

### 层次检索：自顶向下 vs 自底向上

```
检索策略对比

自顶向下（Top-Down）                 自底向上（Bottom-Up）
    │                                     │
    ▼                                     ▼
搜索文档节点                              搜索段落节点
    │                                     │
    ▼                                     ▼
选择匹配的文档                            获取 Top-K 段落
    │                                     │
    ▼                                     ▼
在该文档内搜索章节                        从段落向上回溯到文档
    │                                     │
    ▼                                     ▼
在匹配章节内搜索段落                      返回段落 + 上下文（标题、章节路径）
    │                                     │
    ▼                                     ▼
返回精确段落                              ✔ 更快，直接定位内容
✔ 更精确，逐级筛选                        ❌ 缺失结构的整体上下文
❌ 延迟较高（多次查询）
```

## 背景与演进

- **扁平索引（2022）**：所有文档被切块后 flatten 成一个平面列表，检索时完全忽略文档的结构关系。结果是：跨段落的信息（如"上文中提到的方案"）无法被理解，因为检索到的段落丢失了结构上下文。

- **父子分块（Parent-Child Chunking, 2023）**：LlamaIndex 引入的 `ParentDocumentRetriever`——检索子块但返回父块作为上下文。如检索段落级别的 embedding，但返回整个章节作为 LLM 的上下文输入。这是最早的层次索引实践，但仍是"父子两层"，不支持多级深度。

- **层次检索器（2023-2024）**：LlamaIndex 的 `RecursiveRetriever` 和 LangChain 的 `ParentDocumentRetriever` 进化出真正的多层次检索能力——检索可以从任意层级开始，沿树向上/向下导航。特别是 `RecursiveRetriever` 支持在节点树中递归搜索。

- **AI-native 层次索引（2024）**：Notion AI、Mem 等产品利用文档的块级 ID 和引用关系，构建了支持细粒度引用和上下文回溯的层次索引系统。同时，层次索引与向量索引开始融合——每个节点既有 embedding 又有结构化路径。

- **当前趋势**：层次索引 + 动态上下文聚合——检索到叶子节点后，根据 LLM 的上下文窗口，动态向上聚合多级父节点内容，实现"检索一个点，返回一条路径"的上下文补充。

## 核心矛盾

**结构深度 vs 检索效率**：

```
深层次（完整结构）  ───  检索需多次查询，延迟高
浅层次（扁平）     ───  检索快，但丢失结构上下文
```

详细权衡：

| 层次深度 | 结构化上下文 | 检索延迟 | 存储开销 | 维护复杂度 |
|---------|------------|---------|---------|-----------|
| 扁平（1级） | ⭐ 无 | 50ms | 1x | 最低 |
| 文档→段落（2级） | ⭐⭐⭐ 部分 | 60-80ms | 1.1x | 低 |
| 文档→章节→段落（3级） | ⭐⭐⭐⭐ 丰富 | 80-120ms | 1.3x | 中 |
| 全结构（N级） | ⭐⭐⭐⭐⭐ 完整 | 150ms+ | 2x+ | 高 |

## 主流优化方向

### 1. 混合检索：层次 + 扁平

在层次索引上同时保留扁平搜索的能力——快速搜索叶子节点，然后通过父指针回溯获取结构上下文。

```python
class HybridHierarchicalRetriever:
    def __init__(self, hierarchical_index, embed_model):
        self.index = hierarchical_index
        self.embed_model = embed_model
    
    async def retrieve(self, query: str, top_k: int = 5, context_levels: int = 2):
        """
        检索 + 上下文聚合
        
        1. 扁平搜索所有叶子节点
        2. 对 Top-K 结果，回溯 context_levels 层父节点
        3. 聚合完整层次上下文
        """
        query_emb = await self.embed_model.embed(query)
        
        # Step 1: 扁平搜索所有叶子节点
        all_leaf_nodes = [
            n for n in self.index.nodes_by_id.values()
            if not n.children and n.embedding is not None
        ]
        
        scored = [
            (n, cosine_similarity(query_emb, n.embedding))
            for n in all_leaf_nodes
        ]
        scored.sort(key=lambda x: -x[1])
        
        # Step 2: 回溯获取上下文
        results = []
        for node, score in scored[:top_k]:
            # 向上回溯获取父级上下文
            context_path = []
            current = node
            for _ in range(context_levels):
                if current.parent:
                    context_path.append({
                        "level": current.parent.level,
                        "heading": current.parent.heading_path[-1] if current.parent.heading_path else "",
                        "text": current.parent.text[:200]  # 父节点摘要
                    })
                    current = current.parent
                else:
                    break
            
            results.append({
                "node": node,
                "context": context_path[::-1],  # 从顶层到当前
                "score": score,
                "full_context_text": self._build_context(node, context_levels)
            })
        
        return results
    
    def _build_context(self, node, levels: int) -> str:
        """构建完整上下文文本（用于 LLM 输入）"""
        parts = []
        ancestors = []
        current = node
        for _ in range(levels):
            if current.parent:
                ancestors.append(current.parent)
                current = current.parent
        
        for anc in reversed(ancestors):
            if anc.heading_path:
                parts.append(f"# {' > '.join(anc.heading_path)}")
            if anc.text:
                parts.append(anc.text)
        
        parts.append("---")
        parts.append(node.text)
        
        return "\n\n".join(parts)
```

### 2. 缓存热点路径

统计检索的访问模式，将高频访问的树路径缓存起来。

```python
class CachedHierarchicalRetriever:
    def __init__(self, retriever, cache_size=100):
        self.retriever = retriever
        self.cache = LRUCache(cache_size)
        self.path_stats = defaultdict(int)  # 统计热点路径
    
    async def retrieve(self, query):
        # 扁平搜索叶子节点
        results = await self.retriever.retrieve(query)
        
        for r in results:
            # 统计路径频率
            path_key = tuple(r["node"].heading_path)
            self.path_stats[path_key] += 1
            
            # 如果该路径是热点，缓存上下文
            if self.path_stats[path_key] >= 5:
                cache_key = f"ctx:{hash(path_key)}"
                if cache_key not in self.cache:
                    self.cache[cache_key] = r["full_context_text"]
        
        return results
```

### 3. 动态上下文预算

根据 LLM 的可用上下文窗口动态调整向上聚合的层级数。

```python
def adaptive_context_aggregation(
    node, 
    available_tokens: int, 
    max_levels: int = 3
) -> str:
    """
    在 token 预算内动态聚合上下文
    
    从叶子节点开始，向上聚合父节点内容
    直到 token 预算用尽
    """
    import tiktoken
    enc = tiktoken.get_encoding("cl100k_base")
    
    parts = [node.text]
    tokens_used = len(enc.encode(node.text))
    
    current = node.parent
    level = 0
    while current and level < max_levels and tokens_used < available_tokens:
        ancestor_text = ""
        if current.heading_path:
            ancestor_text = f"[{' > '.join(current.heading_path)}]\n{current.text[:500]}"
        
        ancestor_tokens = len(enc.encode(ancestor_text))
        if tokens_used + ancestor_tokens <= available_tokens:
            parts.append(ancestor_text)
            tokens_used += ancestor_tokens
        else:
            break
        
        current = current.parent
        level += 1
    
    return "\n\n".join(reversed(parts))
```

## 能力边界与结果边界

| 维度 | 扁平索引 | 父子索引（2级） | 全层次索引（N级） |
|------|---------|---------------|----------------|
| 上下文完整性 | ⭐⭐ 仅段落级 | ⭐⭐⭐⭐ 含父级 | ⭐⭐⭐⭐⭐ 完整路径 |
| 检索精度 | ⭐⭐⭐⭐ 直接命中 | ⭐⭐⭐⭐ 同精度 | ⭐⭐⭐ 层级间噪声 |
| 存储开销 | 低 | 中 | 高 |
| 实现复杂度 | 低 | 中 | 高 |
| 可解释性 | ⭐⭐ "黑盒段落" | ⭐⭐⭐ "段落+标题" | ⭐⭐⭐⭐⭐ "完整路径" |
| 跨段落关联 | ⭐ 无 | ⭐⭐⭐ 父级关联 | ⭐⭐⭐⭐⭐ 全路径关联 |

## 与多级索引的区别

- **层次索引 ≠ 多级索引**：这是新人最容易混淆的概念对
  - **层次索引**：保留文档本身的**结构**（章→节→段），利用树形组织关系来检索
  - **多级索引**：对同一份内容创建不同**粒度**的展示（摘要版 vs 完整版 vs 关键词版）
  - **类比**：层次索引像"地图的行政区划（省→市→县）"，多级索引像"不同比例尺的地图（1:100万、1:10万、1:1万）"

- **层次索引 vs 文档结构解析**：层次索引需要结构解析作为输入，但不是每个文档都有清晰的结构。对无结构文本（聊天记录、推文），层次索引不适用。
- **层次索引 vs RAPTOR**：RAPTOR 是动态聚类生成"合成层次"，层次索引是保留"自然层次"。文档结构清晰时层次索引更好，结构模糊时 RAPTOR 更好。

## 核心优势

层次索引的最大价值是**"检索一个点，带回一条路径"**。当 LLM 需要对检索结果进行推理时，仅靠孤立的段落是不够的——它需要知道这个段落在文档中的位置、前后的上下文、所属的章节主题。层次索引天然提供了这种结构感知能力。在技术文档 QA、学术论文分析、法律合同审查等场景中，这种结构上下文至关重要。

## 工程优化

1. **懒加载 embedding**：只为叶子节点生成 embedding，非叶子节点用其子节点 embedding 的平均值或聚合向量
2. **路径编码**：将 heading_path 编码为向量的一部分，使得"相似的章节结构"有相似的 embedding
3. **引用完整性**：当文档更新时，仅重建受影响子树，不改动全树
4. **层级融合搜索**：将不同层级的搜索结果按权重融合（叶子节点 1.0，段落 0.8，章节 0.5，文档 0.3）
5. **树剪枝**：对过深的树（超过 6 层）进行压缩——合并小型叶子节点
6. **分页检索**：树的宽度过大时，水平分片到不同索引分片

## 场景适配指南

| 场景 | 层次深度 | 理由 |
|------|---------|------|
| 技术文档（MD Book） | 文档→章节→段落 | 结构清晰，需要完整路径 |
| 网页/博客 | 文档→段落 | 结构浅，不需要多层 |
| 学术论文 | 章节→段落 | 章节边界清晰 |
| 法律合同 | 条款→子条款→句子 | 需要精确引用条款编号 |
| 代码文档 | 模块→类→方法→行 | 代码结构天然层次 |
| 对话历史 | 会话→轮次 | 简单两级即可 |
