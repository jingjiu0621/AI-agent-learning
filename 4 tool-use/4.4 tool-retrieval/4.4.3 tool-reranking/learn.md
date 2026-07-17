# 工具检索重排序

## 简单介绍

语义搜索的第一步（初筛）通常返回 Top-K 候选（如 20-50 个工具），其中包含一些相关性不高但向量距离较近的结果。重排序（Reranking）使用更精确的模型对这些候选进行重新打分，将最相关的工具排在前面。

## 基本原理

```
初筛（向量检索） → 50 个候选（快但不够精确）
      ↓
重排序（Cross-Encoder） → 对每个候选计算精确相关性分数
      ↓
Top-N 结果（5-15 个） → 注入 LLM 上下文
```

## 为什么需要重排序

| 检索方案 | 召回率@10 | 精确率@10 | 延迟 |
|---------|-----------|----------|------|
| 仅关键词搜索 | ~50% | ~60% | 1ms |
| 仅向量搜索 | ~75% | ~65% | 10ms |
| 向量搜索 + 重排序 | ~85% | ~85% | 100ms |

初筛优先**召回率**，重排序优先**精确率**。

## 重排序模型

### Cross-Encoder 方案

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank(query: str, candidates: list[ToolDef]) -> list[ToolDef]:
    pairs = [(query, build_tool_document(tool)) for tool in candidates]
    scores = reranker.predict(pairs)
    
    scored = list(zip(candidates, scores))
    scored.sort(key=lambda x: x[1], reverse=True)
    return [tool for tool, score in scored[:top_n]]
```

### LLM 重排序

对于极高质量要求，可以用 LLM 作为重排序器：

```python
def llm_rerank(query: str, candidates: list[ToolDef]) -> list[ToolDef]:
    prompt = f"""
    用户请求: {query}
    
    候选工具列表:
    {format_tools(candidates)}
    
    请从候选工具中选择与用户请求最相关的 3 个工具，按相关性从高到低排列。
    只返回工具名称列表。
    """
    response = llm.chat(prompt)
    return parse_ranked_tools(response)
```

## 重排序策略对比

| 策略 | 精度 | 延迟 | 成本 | 适用 |
|------|------|------|------|------|
| TF-IDF/BM25 重排 | 低 | 1ms | 极低 | 快速过滤 |
| Cross-Encoder | 高 | 20-100ms | 低 | 通用 |
| Cohere Rerank API | 高 | 50-200ms | 中 | 不想自建 |
| LLM 重排 | 最高 | 1-5s | 高 | 高质量要求 |
| 规则重排（人工权重） | 中 | 0ms | 无 | 已知偏好 |

## 重排序的维度

| 维度 | 说明 | 计算方式 |
|------|------|----------|
| 语义相关性 | 工具功能与用户意图的匹配度 | Cross-Encoder 分数 |
| 历史使用率 | 该工具被选中的频率 | 统计计数 |
| 用户偏好 | 该用户经常使用哪些工具 | 个性化统计 |
| 工具时效性 | 工具是否在特定时间内更相关 | 时间衰减函数 |
| 权限过滤 | 用户是否有权使用该工具 | 权限检测 |

## 核心挑战

1. **额外延迟**：重排序增加了 20-100ms 的响应时间——需要权衡精度与速度
2. **模型选择**：通用 Cross-Encoder 可能不理解特定领域的工具语义
3. **动态特征**：用户的历史偏好、当前上下文等动态特征需要实时融合
4. **漏斗设计**：初筛→重排→最终选择，每个阶段吞吐量要匹配

## 最佳实践

- 初筛返回 Top-50，重排序取 Top-10 注入 LLM
- 给不同的工具设置"基础权重" —— 高频使用工具可通过权重得到额外加分
- 监控重排序的"提升率" —— 如果重排序后 Top-1 和初筛 Top-1 总是一样，可能重排没带来价值
