# 多路检索结果融合

## 简单介绍

多路检索结果融合（Tool Fusion）是将多种检索方式的结果合并为统一的排序列表。不同检索方式各有优劣——向量搜索长于语义、关键词搜索长于精确匹配、历史统计长于热门偏好。融合它们可以得到比单一检索更好的结果。

## 为什么要多路融合

| 检索方式 | 擅长 | 不擅长 |
|---------|------|--------|
| 向量语义搜索 | 同义理解、泛化 | 精确关键词匹配 |
| BM25 关键词搜索 | 精确匹配、稀有词 | 语义理解 |
| 历史使用统计 | 热门工具、用户偏好 | 新工具、冷门请求 |
| LLM 重排序 | 深度理解 | 速度慢、成本高 |

## 融合方法

### 1. 加权线性融合

```python
def fuse_linear(
    vector_results: list[ScoredTool],
    keyword_results: list[ScoredTool],
    weights: dict[str, float] = None
) -> list[ScoredTool]:
    """加权线性融合"""
    weights = weights or {"vector": 0.5, "keyword": 0.5}
    score_map = {}
    
    for tool in vector_results:
        score_map[tool.name] = weights["vector"] * tool.score
    
    for tool in keyword_results:
        score_map[tool.name] = score_map.get(tool.name, 0) + weights["keyword"] * tool.score
    
    # 排序返回
    ranked = sorted(score_map.items(), key=lambda x: x[1], reverse=True)
    return [name for name, _ in ranked]
```

### 2. Reciprocal Rank Fusion (RRF)

RRF 不需要分数归一化，基于排名位置融合：

```python
def rrf(results: list[list[str]], k: int = 60) -> list[str]:
    """Reciprocal Rank Fusion"""
    scores = {}
    for rank_list in results:
        for rank, tool_name in enumerate(rank_list, 1):
            scores[tool_name] = scores.get(tool_name, 0) + 1 / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)
```

RRF 的核心优势：不需要分数归一化，不同检索方式的排名可以直接融合。

### 3. 学习排序（Learning to Rank）

```python
# 用历史数据训练一个排序模型
# 特征：向量相似度、BM25 分数、历史点击率、工具类别匹配度
# 目标：用户最终是否选择了该工具

def ltr_rank(query_features: dict, candidates: list[ToolDef]) -> list[ToolDef]:
    scores = model.predict([
        extract_features(query_features, tool) for tool in candidates
    ])
    return [tool for _, tool in sorted(zip(scores, candidates), reverse=True)]
```

## 融合策略比较

| 方法 | 实现成本 | 效果 | 适用场景 |
|------|---------|------|----------|
| 加权融合 | 低 | 中 | 快速实现 |
| RRF | 低 | 高 | 通用推荐 |
| 学习排序 | 高 | 最高 | 有历史数据 |
| 级联融合 | 中 | 高 | 分阶段过滤 |

## 多路检索的常见组合

```
组合 1：向量搜索 + BM25 关键词
  → 最常用组合，兼顾语义和精确匹配

组合 2：向量搜索 + BM25 + 历史统计
  → 加入用户个性化信息

组合 3：向量搜索 + LLM 生成的检索词
  → 用 LLM 扩展查询词再检索，提高召回

组合 4：类别过滤 + 向量搜索 + 重排序
  → 先按类别缩减范围，再语义搜索，再精确排序
```

## 挑战

1. **分数归一化**：不同检索方式的分数范围不同（向量 0-1, BM25 0-10+），归一化不当影响融合效果
2. **性能开销**：多路检索意味着多倍的查询耗时
3. **权重调优**：权重参数需要根据实际效果反复调整
4. **冗余结果**：多路检索可能返回相同的工具名——需要去重

## 最佳实践

- RRF 是性价比最高的融合方案——实现简单、效果稳定、无需调参
- 如果用户有明确个性化需求（某些工具被特定团队频繁使用），加入历史统计权重
- 多路检索结果中设置"保底策略"——如果所有检索都没有结果，回退到类别匹配
