# 10.5.1 Thought Map — 思维图谱

## 简单介绍

Thought Map（思维图谱）将 Agent 的推理过程可视化。在文本日志中，Thought 是逐行文字；在思维图谱中，Thought 变成了**节点（Thought Node）和边（Reasoning Edge）**，展现出 Agent 推理的完整结构——从原始观察到中间推导再到最终结论。

## 基本原理

```python
class ThoughtNode:
    """思维图谱中的节点"""
    thought_id: str
    content: str                    # 思维内容摘要
    type: str                       # observation / analysis / hypothesis / decision
    
    # 推理关系
    based_on: list[str]            # 基于哪些之前的 Thought/Observation
    leads_to: list[str]            # 导致了哪些后续 Thought/Action
    confidence: float               # 推理置信度
    
    # 元数据
    turn_number: int
    token_count: int
    model: str
    duration_ms: float

class ThoughtGraph:
    """思维图谱 —— DAG 结构"""
    nodes: list[ThoughtNode]
    edges: list[tuple[str, str, str]]  # (from_id, to_id, relation)
    # relation: "supports" / "contradicts" / "extends" / "questions"
```

思维图谱的可视化布局推荐

```
输入: "帮我分析下这家公司的财务状况"

[用户输入]
    │
    ▼
[分析请求解析] ──→ [识别需要的数据源]
    │                       │
    ▼                       ▼
[需要: 收入表/资产负债表]    [需要: 现金流表]
    │                       │
    ├──── ──── ──── ──── ───┤
    │                       │
    ▼                       ▼
[调用 financial_report]    [调用 cash_flow]
    │                       │
    ▼                       ▼
[收入 100M, 成本 70M]    [现金流 +10M]
    │                       │
    └──── ──── ─ ──── ─────┘
                │
                ▼
[综合分析: 公司健康]
    │
    ▼
[最终回答]
```

## 背景

人类理解复杂推理过程时，图形化的"思维导图"远优于逐行文本。Agent 的推理过程可能涉及多个分支（尝试不同工具）、回溯（发现某个工具结果无用）、并发推理（同时处理多个渠道的信息）。传统文本日志很难表达这种非线性的推理结构。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 日志文本展示 | 逐行输出 Thought 文本 | 无法直观看到推理路径的分支和依赖 |
| 缩进文本树 | 用缩进表示推理层级 | 可读性差，信息量有限 |
| 手动画图 | 把日志复制到思维导图工具 | 费时费力，无法实时 |
| 不可视化 | 完全靠读文本理解推理过程 | 低效，难以定位问题 |

## 当前主流优化方向

1. **自动推理树生成**——从日志/Trace 中自动提取 Thought 节点和推理关系，生成树/图结构
2. **置信度热力图**——在 Thought 节点上用颜色表示置信度（绿色=确定、红色=不确定、灰色=推测），一眼看出 Agent 的不确定区域
3. **分支折叠/展开**——Agent 的 Thought 可能有很长的分支（如尝试了 5 种搜索策略），支持折叠长分支只展示摘要
4. **推理路径对比**——同时展示"预期推理路径"和"实际推理路径"，对比发现偏差

## 实现的最大挑战

1. **Thought 关系提取**——从自然语言文本中自动提取"这个 Thought 基于哪个 Observation"的推理关系，需要 NLP 或 LLM 辅助
2. **大规模思维图谱的可读性**——50 步的 Agent 思维图谱可能有 200+ 节点，如何布局和交互才能不变成"一团乱麻"
3. **多语言 Thought 支持**——Thought 可能混合中文和英文（特别是思考中引用英文工具结果），布局需要适应

## 能力边界

**能做什么：**
- 直观展示 Agent 的推理路径和决策过程
- 快速定位"在哪一步 Agent 的推理出现了偏差"
- 比较不同 Agent/模型/Prompt 的推理模式差异
- 帮助非技术角色理解 Agent 的"思考过程"

**不能做什么：**
- 不能自动判断 Thought 的正确性——图谱展示结构，不验证内容
- 不能展示 LLM 的"潜意识"——只有明确 token 化的 Thought 才能被可视化
- 不能替代 Thought 的文本检查——对于关键的推理节点，仍然需要查看原文

## 最终工程优化

1. **DAG 布局优化**——使用 Sugiyama 分层布局算法（dagre.js）自动排列 Thought 节点，最小化边的交叉
2. **交互式探索**——支持缩放、拖拽、节点搜索、路径高亮（"从观察到结论的最短路径"）
3. **自动摘要节点**——当同一类型的连续 Thought 超过 3 个时，自动合并为"连续推理"摘要节点，可展开查看详情
4. **与 Trace 关联**——点击 Thought 节点跳转到对应的 Trace Span，查看完整的 LLM 输入输出
