# 9.5.3 code-retrieval — 代码检索：语义 + 结构混合搜索

## 简单介绍

代码检索（Code Retrieval）是 Code RAG 中从索引中召回相关代码片段的过程。与普通文本检索不同，代码检索需要融合**语义相似度**（理解代码功能）和**结构匹配**（理解代码结构特征），才能在不同查询场景下都取得良好效果。

## 基本原理

代码检索有三种基本模式：

### 1. 语义检索（Semantic Code Search）

将代码片段和自然语言查询映射到同一向量空间：

```python
# 代码语义检索
def semantic_code_search(query: str, code_chunks: list[dict], top_k: int = 10):
    query_emb = embed_model.encode(query)
    
    scores = []
    for chunk in code_chunks:
        # 代码嵌入（可用函数名+文档+代码体的组合嵌入）
        code_emb = chunk['embedding']
        similarity = cosine_similarity(query_emb, code_emb)
        scores.append((similarity, chunk))
    
    return sorted(scores, key=lambda x: x[0], reverse=True)[:top_k]
```

**适合**：自然语言→代码的查询（"Find the user authentication function"）

### 2. 结构检索（Structural Code Search）

利用 AST 结构特征进行匹配：

```python
# 基于 AST 模式的结构检索
def structural_code_search(pattern_ast: ASTNode, code_ast: ASTNode) -> float:
    """计算两个 AST 的结构相似度"""
    # 方法1：树编辑距离（Tree Edit Distance）
    ted = tree_edit_distance(pattern_ast, code_ast)
    
    # 方法2：子树匹配
    subtree_match = count_matching_subtrees(pattern_ast, code_ast)
    
    # 方法3：结构指纹（Structure Fingerprint）
    fingerprint = extract_structure_fingerprint(code_ast)
    
    return normalized_score(ted, subtree_match, fingerprint)
```

**适合**：模式匹配（"查找所有 try-except 块中只有 pass 语句的错误处理"）

### 3. 图检索（Graph-based Code Retrieval）

利用调用图/依赖图进行拓扑检索：

```python
# 图遍历检索
def graph_code_search(function_name: str, call_graph: nx.DiGraph, 
                       depth: int = 2):
    """查找与目标函数相关的所有函数（调用者 + 被调用者）"""
    results = set()
    
    # 前向：被调用者（callees）
    for _, callee in nx.bfs_edges(call_graph, function_name, 
                                    direction='forward', depth_limit=depth):
        results.add(callee)
    
    # 反向：调用者（callers）
    for caller, _ in nx.bfs_edges(call_graph, function_name, 
                                    direction='reverse', depth_limit=depth):
        results.add(caller)
    
    return results
```

**适合**：影响分析（"修改这个函数会影响哪些其他函数？"）

## 背景：为什么需要混合检索

单一检索模式在代码领域都有明显缺陷：

| 检索模式 | 优势 | 缺陷 |
|---------|------|------|
| 语义检索 | 理解意图，容错性强 | 忽略精确结构；无法区分"a=b"和"b=a" |
| 结构检索 | 精确匹配，无歧义 | 对变量名变化敏感；不理解语义 |
| 图检索 | 关系感知，全局视野 | 只捕获连接关系，不捕获功能语义 |

**核心洞察**：三种检索模式是互补的，**混合检索**（Hybrid Search）才是代码检索的最优解。

## 之前是怎么做的

传统代码检索系统采用单一策略：

- **Sourcegraph**: 早期以正则 + 符号索引为主，2024 年起加入语义搜索
- **GitHub Code Search**: 基于 Elasticsearch 的倒排索引，2023 年后加入向量搜索
- **LiveSearch / ripgrep**: 纯正则匹配，速度快但无语义
- **AskGit / Bloop**: 早期尝试语义代码搜索，但精度不如人意

这些方法的共同问题：**单一策略无法在所有代码查询场景中都表现良好**。

## 当前主流优化方向

### 1. 多路检索融合架构

```python
class HybridCodeRetriever:
    def __init__(self):
        self.semantic = SemanticRetriever()
        self.structural = StructuralRetriever()
        self.graph = GraphRetriever()
    
    def retrieve(self, query: str, k: int = 10) -> list[CodeChunk]:
        # Step 1: 检测查询意图
        intent = detect_query_intent(query)
        
        # Step 2: 根据意图选择检索策略和权重
        weights = self._get_weights(intent)
        
        # Step 3: 多路并行检索
        semantic_results = self.semantic.retrieve(query, k * 2)
        structural_results = self.structural.retrieve(query, k)
        graph_results = self.graph.retrieve(query, k)
        
        # Step 4: 分数归一化
        semantic_results = normalize_scores(semantic_results)
        structural_results = normalize_scores(structural_results)
        graph_results = normalize_scores(graph_results)
        
        # Step 5: RRF 融合
        fused = reciprocal_rank_fusion(
            [semantic_results, structural_results, graph_results],
            weights=weights
        )
        
        return fused[:k]
    
    def _get_weights(self, intent: str):
        """根据查询意图动态分配权重"""
        intent_weights = {
            'find_by_description':   {'semantic': 0.7, 'structural': 0.2, 'graph': 0.1},
            'find_by_pattern':       {'semantic': 0.1, 'structural': 0.8, 'graph': 0.1},
            'find_impact_analysis':  {'semantic': 0.1, 'structural': 0.2, 'graph': 0.7},
            'find_similar_code':     {'semantic': 0.5, 'structural': 0.4, 'graph': 0.1},
            'find_bug_fix':          {'semantic': 0.4, 'structural': 0.3, 'graph': 0.3},
        }
        return intent_weights.get(intent, {'semantic': 0.4, 'structural': 0.3, 'graph': 0.3})
```

### 2. 查询意图分类

```python
def detect_query_intent(query: str) -> str:
    """分类用户代码查询的意图"""
    prompt = f"""Classify this code-related query into one of:
    - find_by_description: "find the function that validates emails"
    - find_by_pattern: "find all try-catch blocks that swallow exceptions"
    - find_impact_analysis: "what depends on this function?"
    - find_similar_code: "find code similar to this snippet"
    - find_bug_fix: "find similar bug patterns"
    
    Query: {query}
    Intent:"""
    
    return llm_classify(prompt)
```

### 3. 检索增强的代码嵌入

专门针对代码优化的 Embedding 模型（如 CodeBERT、GraphCodeBERT、CodeT5+、UniXCoder）：

| 模型 | 特点 | 适用场景 |
|------|------|---------|
| CodeBERT | 基于 Transformer 的双向编码 | NL↔PL 检索 |
| GraphCodeBERT | 引入数据流图 | 含结构信息检索 |
| UniXCoder | 跨语言统一编码 | 多语言代码库 |
| code-embedding-ada-2 | OpenAI 通用嵌入 | 快速原型 |
| Voyage Code | 专为代码优化的嵌入 | 生产级代码检索 |

### 4. 上下文感知检索

不单检索代码块，还包括其依赖上下文：

```python
def contextual_code_retrieval(query: str, top_k: int = 5):
    """检索代码块并自动补充依赖上下文"""
    chunks = hybrid_retriever.retrieve(query, top_k)
    
    enriched = []
    for chunk in chunks:
        # 补充导入语句
        imports = get_imports(chunk['file'])
        # 补充父类/包含类定义
        parent = get_enclosing_class(chunk)
        # 补充被调用的函数签名
        callees = get_callee_signatures(chunk)
        
        enriched.append({
            **chunk,
            'context': {
                'imports': imports,
                'parent_class': parent,
                'callee_signatures': callees
            }
        })
    
    return enriched
```

## 核心矛盾与方法对比

| 方法 | 语义理解 | 精确匹配 | 关系感知 | 延迟 |
|------|---------|---------|---------|------|
| 纯语义检索 | ✅ 高 | ❌ 低 | ❌ 低 | 中 |
| 纯结构检索 | ❌ 低 | ✅ 高 | ❌ 低 | 低 |
| 图检索 | ❌ 低 | ❌ 低 | ✅ 高 | 高 |
| 混合检索 | ✅ 高 | ✅ 高 | ✅ 高 | 中-高 |

## 最大挑战

1. **检索延迟**：多路检索 + 重排序的端到端延迟可能达到 500ms+，对实时编码辅助来说过长
2. **分数归一化**：不同检索模式返回的分数尺度不同，如何公平融合是一个工程难题
3. **大规模仓库的分片**：百万文件级仓库中，图检索的遍历深度需要严格控制
4. **查询歧义**："find user" 可能指用户界面、用户数据库表、用户 API 路由——需要消歧

## 能力边界

- ✅ 自然语言描述的代码功能检索（"find the payment processing function"）
- ✅ 模式匹配检索（"find all singleton patterns"）
- ✅ 影响范围分析（"what breaks if I change this API?"）
- ❌ 需要运行时信息才能理解的语义（反射调用、动态代理）
- ❌ 极度模糊的查询（"fix the code"——没有具体问题描述）

## 工程优化

1. **检索结果缓存**：对常见查询缓存检索结果，LRU 淘汰
2. **索引分片并行检索**：按模块分片，多线程并行检索后聚合
3. **近似图遍历**：使用 PageRank 近似代替 BFS 全遍历
4. **渐进式检索**：先语义检索 Top-100，再对候选做结构过滤

## 推荐工具

- **Pyterrier**: 灵活的检索管道框架，支持多路融合
- **Cohere Rerank**: 代码重排序 API
- **FAISS / pgvector**: 向量检索后端
- **NetworkX / CuGraph**: 图遍历引擎
- **Elasticsearch**: 倒排索引 + 向量检索双模式
