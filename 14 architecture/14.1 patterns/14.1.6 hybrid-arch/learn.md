# 混合架构 — Hybrid Agent Architecture

> **核心思想**：没有一种架构模式能解决所有问题。混合架构根据任务特征动态选择最合适的 Agent 架构模式，在单体、管道、事件驱动、微服务、网格之间自适应切换，实现"对的任务用对的模式"。

```
   ┌─────────────────────────────────────────────────────────┐
   │                混合架构的本质                            │
   │                                                         │
   │   "不是选择一种架构，而是构建一个能选择架构的系统"        │
   │                                                         │
   │   ┌──────────┐    ┌──────────────┐    ┌──────────────┐  │
   │   │ 任务到达   │───►│  路由决策器   │───►│  执行模式 N   │  │
   │   └──────────┘    └──────────────┘    └──────────────┘  │
   │                          │                              │
   │                    路由到合适的模式                        │
   │                          │                              │
   │   ┌──────────┬──────────┬──────────┬────────────────┐    │
   │   │ 单体      │ 管道      │ 事件驱动   │ 网格/微服务     │    │
   │   │ (简单)    │ (多步骤)  │ (实时)    │ (复杂协作)      │    │
   │   └──────────┴──────────┴──────────┴────────────────┘    │
   └─────────────────────────────────────────────────────────┘
```

---

## 1. 简介与背景

### 1.1 什么是混合 Agent 架构

混合架构（Hybrid Architecture）是一种元架构模式（Meta-Pattern）——它本身不是一种新的架构范式，而是一个**架构编排层**，用于在不同场景下选择和组合其他五种架构模式（单体、管道、事件驱动、微服务、网格）。

在传统软件工程中，混合架构早已被广泛使用：一个典型的后端系统可能同时包含 REST API（请求-响应）、消息队列（异步事件）、批处理任务（管道）和微服务组件。Agent 系统的混合架构思想与之类似，但引入了一个关键的新维度：**LLM 推理成本**。

```
传统混合架构 vs Agent 混合架构:

传统软件:
  [REST API] + [消息队列] + [批处理] = 技术栈的组合
  关注点: 延迟、吞吐、一致性

Agent 系统:
  [单体推理] + [管道协作] + [网格辩论] = 推理模式的组合
  关注点: Token 成本、推理质量、响应时间
```

### 1.2 为什么需要混合架构

现实世界的 Agent 任务具有极端的多样性。同一个系统可能同时需要处理：

| 任务类型 | 示例 | 最佳架构模式 | 推理成本 |
|---------|------|-------------|---------|
| 简单知识问答 | "今天天气怎么样" | 单体 Agent | ~1K tokens |
| 多步骤信息收集 | "帮我比较这三款产品的性价比" | 管道架构 | ~10K tokens |
| 实时监控响应 | "系统异常告警，请排查" | 事件驱动 | ~5K tokens |
| 复杂研究报告 | "分析2025年AI行业趋势" | 网格架构 | ~100K+ tokens |
| 多 Agent 协作 | "帮我安排团队的年度旅行" | 微服务 Agent | ~50K tokens |

如果只用一种模式：
- **只用单体**：复杂任务推理质量低，Token 利用率差
- **只用网格**：简单任务浪费大量 Token 和延迟
- **只用管道**：灵活度不够，无法处理动态依赖
- **只用事件驱动**：简单查询的响应路径过于复杂

**混合架构的核心动机**：用一个统一的入口，根据任务特征动态路由到最合适的处理模式，从而在**质量、成本、延迟**之间找到最优平衡。

### 1.3 历史演进

```
Agent 架构的演进路径:

Phase 1: 单一模式 (2022-2023)
  "ChatGPT 能解决一切" → 所有任务走同一个 Agent 循环
  问题: 简单任务浪费，复杂任务不够

Phase 2: 手动混合 (2023-2024)
  "根据不同场景硬编码不同 Agent 流程"
  如: LangChain 的 Router Chain、Semantic Kernel 的 Planner
  问题: 路由逻辑死板，模式间无状态共享

Phase 3: 自适应混合 (2024-2025)
  "LLM 本身作为路由器，自动选择执行模式"
  如: OpenAI 的 Structured Outputs 用于路由决策
  问题: 路由器本身的成本和延迟

Phase 4: 智能编排 (2025-Now)
  "多层次路由 + 动态模式组装 + 成本感知"
  路由器本身可以分析任务复杂度、历史成功率、当前系统负载
  模式可以在运行时动态组装（如临时创建一个 3-Agent 网格）
```

---

## 2. 基本原理

### 2.1 核心公式

混合架构的核心工作流程可以概括为三步：

```
Task → Classification → Mode Routing → Pattern Execution
  ↑                                          │
  └───────────────── Result ─────────────────┘
```

**详细流程**：

```
输入: 用户任务 T
步骤:
  1. Task Classification: 分析任务 T 的类型、复杂度、领域、所需协作程度
  2. Mode Routing: 根据分类结果选择最佳执行模式 M
  3. Pattern Execution: 使用模式 M 的完整架构执行任务
  4. Result Collection: 收集结果，可选性地做后处理（如跨模式结果融合）
输出: 最终响应
```

### 2.2 架构作为"模式开关"

混合架构本质上是一个**模式开关（Pattern Switch）**——它将 Agent 系统从"一种架构适配所有任务"转变为"为每个子任务选择最合适的架构"。

```
                    ┌──────────────────────┐
                    │    混合架构路由器      │
                    │  (Pattern Router)    │
                    └──────────────────────┘
                              │
          ┌───────────────────┼────────────────────┐
          ▼                   ▼                    ▼
   ┌──────────────┐   ┌──────────────┐    ┌──────────────┐
   │ "简单问答"    │   │ "多步骤分析"   │    │ "复杂研究"    │
   │  ↙ 单体模式   │   │  ↙ 管道模式    │    │  ↙ 网格模式   │
   │              │   │              │    │              │
   │  单一 LLM 调用  │   │  分阶段处理    │    │  多 Agent     │
   │  1 个工具     │   │  阶段间传递    │    │  辩论与共识    │
   │  ~1K tokens  │   │  ~10K tokens │    │  ~100K tokens │
   └──────────────┘   └──────────────┘    └──────────────┘
```

### 2.3 多范式 Agent 设计

混合架构的 Agent 是多范式的（Multi-Paradigm），这意味着 Agent 系统不再绑定到单一的交互模式，而是能根据上下文切换行为模式：

```
Agent 的多种"工作模式":

┌─────────────────────────────────────────────────────────────┐
│  模式             │ 工作方式                      │ 类比     │
├─────────────────────────────────────────────────────────────┤
│  反射模式          │ 快速响应，不经过深度推理        │ 人的直觉  │
│  (Reflexive)      │ query → template → respond    │          │
├─────────────────────────────────────────────────────────────┤
│  规划模式          │ 先拆解、再执行、分步骤完成      │ 人的计划  │
│  (Planning)       │ decompose → execute → verify  │          │
├─────────────────────────────────────────────────────────────┤
│  协作模式          │ 多个 Agent 交换观点、辩论       │ 团队讨论  │
│  (Collaborative)  │ agents debate → synthesize     │          │
├─────────────────────────────────────────────────────────────┤
│  学习模式          │ 使用工具获取新知识、反思         │ 人学习    │
│  (Learning)       │ tool-use → reflect → update     │          │
├─────────────────────────────────────────────────────────────┤
│  监控模式          │ 监听事件、被动触发、持续运行      │ 值班人员  │
│  (Monitoring)     │ event → assess → act            │          │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 核心矛盾

混合架构不是免费的午餐。引入模式切换能力的同时，也引入了一系列核心矛盾。

### 3.1 灵活性 vs 复杂性

| 灵活度 | 模式数量 | 系统复杂度 | 维护成本 |
|--------|---------|-----------|---------|
| 低 | 1 种模式 | 低 | 低 |
| 中 | 2-3 种模式 | 中 | 中 |
| 高 | 4-5 种模式 | 高 | 高 |
| 极高 | 6+ 种模式 | 极高 | 难以维护 |

```
复杂度爬坡曲线:

模式数  维护成本    路由复杂度    测试组合数
─────  ────────  ──────────  ──────────
  1      1x          0          1条路径
  2      2x          1x         2条路径
  3      4x          3x         6条路径
  4      8x          6x         24条路径
  5      16x         10x        120条路径
  n      2^(n-1)x    C(n,2)     n! 路径

结论: 模式数量不是越多越好。通常 3-4 种模式是最优平衡点。
```

**核心权衡**：每增加一种模式，系统获得了一部分新场景的处理能力，但同时增加了路由决策的复杂度、测试的组合数、以及开发维护成本。好的混合架构设计在于**识别出最具区分度的少数几种模式**，而不是追求模式数量的覆盖度。

### 3.2 最优路由 vs 路由开销

路由决策本身是有成本的——无论是 Token 成本（LLM 分类）、延迟成本（等待分类结果）、还是计算成本（特征提取）。

```
路由开销示例:

路由方式             Token 成本     延迟增加      准确率
────────────────  ────────────  ─────────  ─────────
规则路由              0 tokens      0-1ms      70-80%
轻量分类器            0 tokens      5-20ms     80-90%
LLM 路由            200-500 tokens  500ms-2s   90-95%
多层次路由            500-2000 tokens 1-5s     95-99%

矛盾:
  - 更复杂的路由 → 更高的准确率 → 更好的执行结果
  - 更复杂的路由 → 更高的成本 → 简单任务反而变慢变贵

示例: 一个简单查询 "现在几点？"
  - 如果完美路由 → 单体模式 → 100 tokens, 1s
  - 如果路由错误 → 网格模式 → 50000 tokens, 30s
  - 如果路由成本太高 → 光路由就花了 1000 tokens → 本末倒置
```

**解决思路**：采用渐进式路由（Progressive Routing），先试低成本路由，只有在不确定时才升级到高成本路由。

### 3.3 一致性 vs 专业化

| 维度 | 追求一致性 | 追求专业化 |
|------|-----------|-----------|
| 响应风格 | 统一的语气和格式 | 按模式风格定制 |
| 错误处理 | 统一的错误策略 | 模式专属错误处理 |
| 记忆系统 | 全局共享记忆 | 模式隔离的记忆 |
| 工具集 | 统一工具注册表 | 模式特定工具集 |
| 状态管理 | 统一状态机 | 模式独立状态 |

**矛盾本质**：不同的架构模式有不同的"工作方式"，但用户期望一个统一的体验。当模式切换时，如何保持以下方面的一致性：
- 对话上下文不丢失
- 用户信任不下降（模式切换不能让用户困惑）
- 数据格式统一的输出

### 3.4 开发成本 vs 运维收益

```
开发混合架构的时间分布:

┌─────────────────────────────────────────────────────────┐
│  阶段              │ 比例  │ 说明                       │
├─────────────────────────────────────────────────────────┤
│  路由系统            │ 25%   │ 分类器、路由逻辑、调度     │
│  每种模式实现         │ 30%   │ 4 种模式的 Agent 逻辑     │
│  模式间通信           │ 15%   │ 状态传递、上下文转换      │
│  测试覆盖             │ 20%   │ 每种路径的端到端测试      │
│  监控与可观测性       │ 10%   │ 跨模式追踪、指标          │
└─────────────────────────────────────────────────────────┘

对比: 单一模式开发只要 30% 的工作量

收益何时超过成本:
  - 系统需要处理 3 种以上差异较大的任务类型
  - 每天处理 1000+ 请求（Token 优化收益明显）
  - 团队规模 > 5 人（可以分工维护不同模式）
```

---

## 4. 主流设计模式

混合架构本身是元模式，但它内部有一些被反复使用的组织模式。

### 4.1 任务路由器模式 (Task Router Pattern)

这是最基础也最广泛的混合模式。一个中央路由器根据任务特征分派到不同的处理模式。

```
                      ┌──────────────────┐
                      │   用户请求         │
                      └────────┬─────────┘
                               │
                      ┌────────▼─────────┐
                      │   任务分类器        │
                      │   (Task Classifier)│
                      └────────┬─────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ Agent A      │   │ Agent B      │   │ Agent C      │
   │ (单体模式)    │   │ (管道模式)    │   │ (网格模式)    │
   │              │   │              │   │              │
   │ 简单查询      │   │ 多步骤任务    │   │ 复杂研究      │
   │ 快速响应      │   │ 分阶段处理    │   │ 多 Agent 协作  │
   └──────────────┘   └──────────────┘   └──────────────┘
           │                  │                  │
           └──────────────────┼──────────────────┘
                              ▼
                     ┌──────────────────┐
                     │   响应统一输出      │
                     └──────────────────┘
```

**适用场景**：请求类型差异明显、分类边界清晰、各模式相对独立。

**优缺点**：
- 优点：实现简单、路由逻辑直观、模式间隔离好
- 缺点：无法处理混合类型的任务、模式间不能协作

### 4.2 分层混合模式 (Layered Hybrid Pattern)

将任务按复杂度分层，不同层级使用不同的架构模式。这是生产系统中最常见的模式。

```
任务复杂度层级:

  层级 0: 缓存层
    缓存命中 → 直接返回 | 0 tokens | <100ms
    缓存未命中 → 进入下一层

  层级 1: 简单层 (单体模式)
    单步推理 + 0-1 个工具调用
    适用于: 知识问答、简单查询
    Token: ~1K | 延迟: ~1s

  层级 2: 标准层 (管道模式)
    2-5 步推理 + 多工具调用
    适用于: 信息收集、内容生成
    Token: ~10K | 延迟: ~5s

  层级 3: 复杂层 (事件驱动/微服务)
    异步处理 + 多 Agent 协作
    适用于: 长时间运行的任务
    Token: ~50K | 延迟: ~30s

  层级 4: 研究层 (网格模式)
    多 Agent 辩论 + 共识形成
    适用于: 深度分析、研究报告
    Token: ~200K | 延迟: ~2min
```

```
                          ┌──────────────┐
                          │   请求        │
                          └──────┬───────┘
                                 │
                     ┌───────────▼───────────┐
                     │   L0: 缓存检查         │
                     │   (Cache Layer)        │
                     └───────────┬───────────┘
                           命中/ │ 未命中
                       ┌─────────▼─────────┐
                       │  L1: 轻量级分类器    │
                       │  (Fast Classifier)  │
                       └─────────┬─────────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
            ┌────────────┐ ┌────────────┐ ┌────────────┐
            │ L1: 单体    │ │ L2: 管道    │ │ L3: 网格    │
            │ 简单模式    │ │ 标准模式    │ │ 研究模式    │
            │ ~1K tokens │ │ ~10K tokens│ │ ~200K toks│
            └────────────┘ └────────────┘ └────────────┘
                    │            │            │
                    └────────────┼────────────┘
                                 ▼
                        ┌──────────────────┐
                        │   格式化输出        │
                        └──────────────────┘
```

**适用场景**：请求复杂度差异极大、需要对不同用户提供差异化服务。

### 4.3 动态组装模式 (Dynamic Assembly Pattern)

在运行时根据任务需求动态组合微型架构（micro-architecture），而不是使用预定义的模式。

```
动态组装流程:

用户请求: "分析 A 公司和 B 公司的财报，给出投资建议"

Step 1: 任务分解
  ┌────────────────────────────────────────────────────┐
  │  子任务                                        │
  ├────────────────────────────────────────────────────┤
  │  [1] 获取 A 公司财报数据     → 工具调用 (单个动作)   │
  │  [2] 获取 B 公司财报数据     → 工具调用 (单个动作)   │
  │  [3] 对比分析               → 需要 2 个 Agent 辩论   │
  │  [4] 生成投资建议           → 单体推理               │
  └────────────────────────────────────────────────────┘

Step 2: 模式选择
  [1] → 单体模式 (简单工具调用)
  [2] → 单体模式 (简单工具调用)
  [3] → 创建临时网格 (2个分析Agent + 1个仲裁Agent)
  [4] → 单体模式 (单步推理)

Step 3: 执行编排
  ┌─────┐   ┌─────┐    ┌──────────────┐    ┌─────┐
  │ [1] │   │ [2] │    │ [3] 网格模式  │    │ [4] │
  │ 单体 │   │ 单体 │    │ Agent A↔Agent B│    │ 单体 │
  │数据  │   │数据  │    │    ↕         │    │建议 │
  │获取  │   │获取  │    │  仲裁者      │    │生成 │
  └─────┘   └─────┘    └──────────────┘    └─────┘
      │         │              │              │
      └─────────┴──────────────┴──────────────┘
                         │
                    ┌────▼────┐
                    │ 最终结果  │
                    └─────────┘
```

**适用场景**：任务结构多变、无法预定义固定模式路径。

### 4.4 渐进升级模式 (Progressive Escalation Pattern)

从最简单的模式开始，如果结果质量不够，逐步升级到更复杂的模式。

```
                          ┌──────────┐
                          │ 请求     │
                          └────┬─────┘
                               │
                    ┌──────────▼──────────┐
                    │ L1: 直接推理 (Zero-shot)│
                    │ LLM 直接回答          │
                    └──────────┬──────────┘
                         成功/ │ 不确定或失败
                    ┌──────────▼──────────┐
                    │ L2: 单工具调用       │
                    │ 使用搜索/计算等工具   │
                    └──────────┬──────────┘
                         成功/ │ 仍不够
                    ┌──────────▼──────────┐
                    │ L3: 多步管道         │
                    │ 分解任务，逐步执行    │
                    └──────────┬──────────┘
                         成功/ │ 还是不够
                    ┌──────────▼──────────┐
                    │ L4: 多 Agent 协作    │
                    │ 创建 Agent 团队讨论   │
                    └──────────┬──────────┘
                               │
                          ┌────▼────┐
                          │ 最终结果  │
                          └─────────┘

升级触发条件:
  - LLM 自身对结果置信度低 (token log probability 低)
  - 工具调用出现错误或缺少必要数据
  - 任务涉及的内容超出单步处理能力
  - 用户主动要求更深入的分析

降级策略:
  - Token 消耗超过预算阈值
  - 处理时间超过用户等待阈值
  - 复杂模式连续失败 (如网格无法达成共识)
```

**适用场景**：对结果质量要求高、但希望在简单场景下节省成本。

---

## 5. 路由策略

路由策略决定了"如何将任务映射到正确的模式"，是混合架构的核心决策点。

### 5.1 基于规则的路由 (Rule-based Routing)

使用预定义的规则集进行任务分类和模式选择。

```
规则路由示例:

RULE_SET = [
    # 规则优先级从上到下匹配
    {
        "condition": "task_length < 50 AND tools_needed == 0",
        "mode": "monolithic",
        "reason": "极简单的查询，不需要工具"
    },
    {
        "condition": "task_length < 200 AND tools_needed <= 2",
        "mode": "monolithic",
        "reason": "简单任务，1-2个工具"
    },
    {
        "condition": "task_type == 'analysis' AND depth == 'deep'",
        "mode": "mesh",
        "reason": "深度分析需要多Agent辩论"
    },
    {
        "condition": "task_type == 'research' AND sources >= 3",
        "mode": "pipeline",
        "reason": "多源研究需要分阶段处理"
    },
    {
        "condition": "duration == 'long_running'",
        "mode": "event_driven",
        "reason": "长时间运行需要异步事件驱动"
    },
    # 默认: 单体模式
    {"condition": "true", "mode": "monolithic"}
]
```

**适用场景**：
- 任务类型高度结构化（如客服工单分类）
- 特征维度少且边界清晰
- 需要极低的路由延迟（<1ms）

**优缺点**：
- 优点：零 Token 成本、低延迟、可解释性强
- 缺点：规则维护困难、无法处理边界情况、需要领域专家编写规则

### 5.2 基于 ML 的路由 (ML-based Routing)

训练一个轻量级分类器来预测最佳模式。

```
ML 路由架构:

输入特征:
  - 任务文本的 embedding 向量
  - 任务长度 (tokens)
  - 任务中提到的工具数量
  - 历史相似任务的最佳模式
  - 当前系统的负载情况

输出:
  - 每个模式的置信度分数
  - 推荐模式（最高分）
  - 不确定性标记（如果置信度低，降级到 LLM 路由）

classifier_input = {
    "embedding": text_embedding(task_text),  # 768-dim vector
    "task_length": len(tokens),
    "tool_mentions": extract_tool_mentions(task_text),
    "task_type_probs": task_type_classifier(task_text),
    "historical_success": get_best_mode_history(similar_tasks)
}
predicted_mode = classifier.predict(classifier_input)
```

**适用场景**：
- 有大量标注的历史数据
- 任务特征维度多、规则难以定义
- 对路由延迟有要求（<50ms）

**优缺点**：
- 优点：速度快、可处理复杂特征、可持续优化
- 缺点：需要训练数据、冷启动问题、黑盒不可解释

### 5.3 LLM 作为路由器 (LLM-as-Router)

直接使用 LLM 进行路由决策。这是目前最灵活也最流行的方法。

```
LLM 路由器提示词模板:

SYSTEM_PROMPT = """你是一个 Agent 架构路由专家。你的任务是为用户请求选择最合适的处理模式。

可选模式:
1. monolithic (单体模式): 简单查询，0-2个工具调用，单步推理
   - Token成本: 低 (~1K)
   - 延迟: 快速 (~1s)
2. pipeline (管道模式): 多步骤任务，需要分阶段处理
   - Token成本: 中 (~10K)
   - 延迟: 中等 (~5s)
3. mesh (网格模式): 复杂研究，需要多Agent辩论和达成共识
   - Token成本: 高 (~100K)
   - 延迟: 慢 (~30s)
4. fallback (降级模式): 以上模式都失败时的兜底策略

请输出 JSON 格式:
{"mode": "monolithic", "confidence": 0.95, "reason": "简短原因"}
"""

def llm_route(task_text: str) -> RouteDecision:
    response = llm.chat([
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": task_text}
    ])
    return parse_json(response)
```

**进阶技巧 —— 带上下文的路由**：

```
LLM 路由器 + 历史上下文:

def llm_route_with_context(task_text, conversation_history, mode_stats):
    context = {
        "conversation_turns": len(conversation_history),
        "previous_mode": get_last_mode(conversation_history),
        "mode_success_rates": {
            "monolithic": mode_stats["monolithic"]["success_rate"],
            "pipeline": mode_stats["pipeline"]["success_rate"],
            "mesh": mode_stats["mesh"]["success_rate"],
        },
        "token_budget_remaining": get_token_budget()
    }
    
    prompt = f"""
    当前上下文:
    - 对话轮次: {context['conversation_turns']}
    - 上次使用的模式: {context['previous_mode']}
    - 各模式历史成功率: {json.dumps(context['mode_success_rates'])}
    - Token预算剩余: {context['token_budget_remaining']}
    
    用户请求: {task_text}
    
    选择最佳模式并输出 JSON。
    """
    ...
```

**适用场景**：
- 任务类型多样化且不断变化
- 路由特征难以用规则或 ML 覆盖
- 对路由准确率要求高

### 5.4 成本感知路由 (Cost-aware Routing)

在路由决策中显式考虑 Token 成本和响应时间。

```
成本感知路由公式:

score(mode) = w1 * quality_score(mode) 
            - w2 * token_cost(mode) 
            - w3 * latency(mode)
            + w4 * success_rate(mode)

选择 score 最高的模式

其中:
  quality_score  = 预估推理质量 (0-1)
  token_cost     = 预估 Token 消耗 (归一化)
  latency        = 预估响应时间 (归一化)
  success_rate   = 历史成功率
  w1, w2, w3, w4 = 权重, 可根据当前系统状态动态调整

动态权重调整:
  - 系统负载高时: 增加 w2 (Token成本) 和 w3 (延迟) 的权重
  - 用户要求高质量时: 增加 w1 (质量) 的权重
  - 该模式近期失败率高时: 降低 w4 (成功率) 的权重
```

**适用场景**：
- Token 成本是核心指标（付费 API）
- 系统需要在高负载下保持可用
- 需要为不同类型用户提供差异化服务

### 5.5 混合路由 (Hybrid Routing)

将以上多种路由策略组合使用，取长补短。

```
多层次混合路由:

Step 1: 快速排除 (规则路由)
  用简单的规则快速识别明显简单的任务
  if 任务长度 < 20 tokens and 不含工具关键词:
      return monolithic (置信度: 0.9)

Step 2: 统计分类 (ML 路由)
  用轻量级分类器处理常见任务类型
  if classifier.confidence > 0.8:
      return classifier.predicted_mode

Step 3: 深度分析 (LLM 路由)
  对于不确定的情况，使用 LLM 分析
  if not resolved:
      return llm_route(task)

Step 4: 兜底 (默认模式)
  所有路由都失败时使用最安全的模式
  return monolithic (安全兜底)
```

```
                     ┌──────────┐
                     │ 请求     │
                     └────┬─────┘
                          │
               ┌──────────▼──────────┐
               │ L1: 规则路由          │
               │ (0 tokens, <1ms)    │
               └──────────┬──────────┘
                     确定/ │ 不确定
               ┌──────────▼──────────┐
               │ L2: ML 分类器        │
               │ (0 tokens, <50ms)   │
               └──────────┬──────────┘
                置信度高/ │ 置信度低
               ┌──────────▼──────────┐
               │ L3: LLM 路由         │
               │ (~300 tokens, ~1s)  │
               └──────────┬──────────┘
                          │ 仍不确定
               ┌──────────▼──────────┐
               │ 默认: 单体模式        │
               │ (安全兜底)           │
               └─────────────────────┘
```

---

## 6. 能产生什么结果

采用混合架构后，系统在多个维度上产生可量化的改善。

### 6.1 资源利用最优化

```
单一模式 vs 混合模式 的 Token 消耗对比:

场景: 一个 Agent 系统每天处理 10000 个请求
     其中 70% 简单查询, 20% 中等任务, 10% 复杂任务

单一网格模式 (只用最复杂的模式):
  每天消耗 = 10000 * 100K = 1,000,000,000 tokens
  每月成本 ≈ $5000 (GPT-4o 级别)

混合架构:
  7000 * 1K    (简单 → 单体)  =   7,000,000
  2000 * 10K   (中等 → 管道)  =  20,000,000
  1000 * 100K  (复杂 → 网格)  = 100,000,000
  每天总消耗                    = 127,000,000 tokens
  每月成本 ≈ $635

节省: 约 8 倍的 Token 成本降低
```

### 6.2 速度与能力的均衡

```
响应时间分布:

单一单体模式:
  - 简单任务: 1s   (很好)
  - 复杂任务: 失败或质量低 (不可接受)

单一网格模式:
  - 简单任务: 30s  (用户等不及)
  - 复杂任务: 30s  (质量好)

混合架构:
  - 简单任务: 1-2s  (快速)
  - 中等任务: 3-5s  (合理)
  - 复杂任务: 20-30s (可接受)

用户满意度:
  单体模式: 70% (简单OK, 复杂差)
  网格模式: 60% (简单太慢, 复杂OK)
  混合架构: 90%+ (各取所长)
```

### 6.3 生产级鲁棒性

```
混合架构的鲁棒性优势:

降级路径:
  正常: 网格模式 → 管道模式 → 单体模式 (逐步降级)
  某模式失败 → 自动降级到更简单但可用的模式
  所有模式失败 → 缓存或静态响应兜底

熔断机制:
  某模式连续失败 N 次 → 暂时禁用该模式
  冷却期后自动恢复
  管理员可以手动启停模式

监控指标:
  每个模式的: 调用量、成功率、平均延迟、Token消耗
  路由器的: 分类准确率、路由延迟、降级率
  全系统: 端到端满意度、成本/请求
```

---

## 7. 最大挑战

### 7.1 路由准确性 —— 选错模式的代价

路由错误是混合架构中代价最高的问题。

```
路由错误的代价矩阵:

                        实际应该用的模式
               monolithic    pipeline     mesh
路由到: ───  ───────────────────────────────────
monolithic     ✓ 完美        ✗ 质量不足    ✗ 完全不夠
pipeline       ✗ 浪费资源     ✓ 完美        ✗ 协作不够
mesh           ✗ 极大浪费    ✗ 较大浪费     ✓ 完美

代价量化:
  - 简单任务路由到网格: 100 倍的 Token 浪费 + 30 秒等待
  - 复杂任务路由到单体: 输出质量差、需要用户重试
  - 中等任务路由到单体: 可能成功但质量欠佳

案例: 一个电商客服 Agent
  - 用户问 "订单状态" → 路由到网格模式
  - 3 个 Agent 辩论了 20 秒 "这个订单应该怎么查"
  - 最后结论: 调用一次订单查询 API
  - 浪费: 只用了 0.1% 的处理能力, 但花了 100% 的资源
```

**缓解策略**：
- 带置信度的路由 + 人类兜底
- 路由结果可被后续步骤修正
- A/B 测试路由决策
- 持续收集路由反馈进行优化

### 7.2 模式切换时的状态连续性

当任务在一个模式中处理到一半需要切换到另一个模式时，状态如何传递？

```
状态传递问题:

场景: 任务在管道模式执行了 3 个阶段后，发现需要多 Agent 协作
      需要切换到网格模式继续处理

需要传递的状态:
  {
    "task_id": "task_12345",
    "original_input": "比较这三款产品的...",
    "phase_results": {
      "phase_1_search": ["产品A 数据", "产品B 数据", "产品C 数据"],
      "phase_2_analyze": {"A": "优劣分析", "B": "优劣分析", "C": "优劣分析"},
      "phase_3_verify": "分析完成 80%，缺少竞品对比深度"
    },
    "mode_history": ["monolithic → pipeline"],
    "token_used_so_far": 8500,
    "conversation_context": [...]
  }

难点:
  1. 不同模式的状态结构不同 (管道的阶段结果 ≠ 网格的辩论记录)
  2. 状态序列化/反序列化成本
  3. 某些中间状态在模式间不可重用
  4. 状态一致性问题 (部分状态已更新？)
```

**解决方案**：
- 定义统一的**任务状态协议** (Task State Protocol)
- 使用状态适配器 (State Adapter) 转换模式间的状态
- 采用事件溯源 (Event Sourcing) 记录所有状态变更

### 7.3 监控复杂度

```
单一模式监控:

单一 Agent 监控:
  [Agent] → 延迟 / Token / 成功率 / 工具调用
  └── 单一的追踪链路

混合架构监控:

  路由器:
    [Router] → 路由决策 / 置信度 / 路由延迟

  模式 1 (单体):
    [Agent_Mono] → 延迟 / Token / 成功率

  模式 2 (管道):
    [Phase1] → [Phase2] → [Phase3]
      ↓          ↓          ↓
    追踪   追踪   追踪

  模式 3 (网格):
    [Agent_A] ── [Agent_B]
        │            │
      追踪         追踪

  跨模式追踪:
    [Router] → [Mode_2_Phase_1] → [Mode_2_Phase_2] 
        → [Switch_to_Mode_3] → [Agent_A] → [Agent_B]

  问题: 如何在一个统一的视图中看到完整的请求链路？
```

**解决方案**：
- 统一的追踪 ID (Trace ID) 贯穿所有模式
- 每个模式输出标准化的遥测事件
- 使用 OpenTelemetry 或类似框架聚合
- 设计"模式切换"作为特殊的可视化节点

### 7.4 测试的组合爆炸

```
混合架构的测试覆盖问题:

模式数: 4 (单体、管道、网格、事件驱动)
路由方式: 3 (规则、ML、LLM)
状态组合: 2 (有状态、无状态)
错误场景: 3 (正常、部分失败、完全失败)

基础路径数: 4 × 3 × 2 × 3 = 72

模式间切换: C(4,2) = 6 种切换路径
切换条件: 3 (用户触发、系统触发、错误触发)

切换测试: 6 × 3 = 18

考虑执行顺序:
  单体 → 管道(3阶段): 不同阶段切换
  网格 → 管道(3阶段)
  ...

总测试组合 ≈ 200-500 个端到端场景

相比之下:
  单体 Agent: ~10 个测试场景
  混合架构: ~500 个测试场景
```

**解决方案**：
- 契约测试 (Contract Testing)：每个模式独立测试
- 路由测试与模式执行测试分离
- 使用 Property-based Testing 随机覆盖组合
- 生产环境的金丝雀发布 (Canary Release)

### 7.5 模式切换开销

```
冷启动和上下文传输成本:

模式切换的类型:
  1. 热切换: 同一进程中切换 (低开销, ~10ms)
  2. 温切换: 同进程但需要加载新资源 (~100ms)
  3. 冷切换: 启动新进程/容器 (~1-10s)
  4. 冻结切换: 从持久化状态恢复 (~10-30s)

上下文传输开销:
  - 序列化状态: ~1-5ms (取决于状态大小)
  - 传输: ~0.1-1ms (进程内) / ~10-100ms (跨进程)
  - 反序列化: ~1-5ms
  - LLM 预热: ~500ms (第一次推理有冷启动)

实战案例:
  一个任务从管道模式切换到网格模式
  总切换开销 = 500ms (状态传输) + 2s (新 Agent 初始化) + 1s (LLM 预热)
  = 3.5s 额外延迟

  如果这个任务总处理时间只有 10s
  切换开销占了 35%
```

---

## 8. 能力边界

### 8.1 什么时候混合架构是过度设计

```
不适合混合架构的场景:

场景 1: Todo List Agent
  - 任务: 增删改查待办事项
  - 复杂度: 极低
  - 只需要: 1 个 Agent + 1-2 个工具
  
  ✅ 单体 Agent 就够
  ❌ 混合架构: 增加 10 倍复杂度，0 收益

场景 2: 内部监控告警 Agent
  - 任务: 读取指标 → 判断是否告警 → 发送通知
  - 流程: 固定、结构化
  - 只需要: 简单的管道模式
  
  ✅ 管道架构就够
  ❌ 混合架构: 路由系统完全用不到

场景 3: 单人使用的个人助理
  - 任务: 日常问答、日程管理
  - 用户数: 1
  - 负载: 低
  
  ✅ 单体或管道就够
  ❌ 混合架构: 运维成本超出收益

判断标准: 是否满足以下至少 2 条?
  □ 系统需要处理 3 种以上差异明显的任务类型
  □ 每天的请求量 > 1000 (成本优化有明显收益)
  □ 任务复杂度范围覆盖"极简单"到"极复杂"
  □ 系统需要对外提供 SLA (需要降级/熔断)
  □ 多个团队/开发者在同一个平台上构建 Agent
```

### 8.2 什么时候混合架构是必要的

```
必须使用混合架构的场景:

场景 1: 企业级客户服务平台
  - 简单查询: "我的订单到哪了?" (占 60%)
  - 中等复杂: "帮我退掉订单 A, 换购订单 B" (占 30%)
  - 复杂: "分析我过去一年的消费模式, 给出省钱建议" (占 10%)
  
  → 混合架构的必要性: 高
  → Token 节省: ~80%
  → 用户满意度提升: ~30%

场景 2: AI 开发平台 (如 Dify、Coze)
  - 用户自己编排 Agent 工作流
  - 平台需要适配各种不同的编排模式
  - 有些用户做简单聊天机器人, 有些做复杂多 Agent 系统
  
  → 混合架构的必要性: 极高
  → 没有混合架构就无法满足多样化需求

场景 3: 研究助手系统
  - 简单: "解释什么是量子计算" → 单体
  - 中等: "对比 Transformer 和 Mamba 架构" → 管道
  - 复杂: "用最新论文数据, 全面分析 2025 年 AI 安全研究进展" → 网格
  
  → 混合架构的必要性: 高
  → 单一模式无法在所有请求上都表现良好
```

### 8.3 混合架构的能力上限

```
混合架构不能解决的问题:

1. LLM 本身的局限性
   - 无论用什么模式, 底层 LLM 的推理能力是天花板
   - 混合架构不能让 GPT-4 变出 GPT-5 的能力
   - 路由决策本身也受 LLM 能力限制

2. 跨模式任务的一致性
   - 如果一个任务需要在 3 种模式间来回切换
   - 状态的一致性和事务性难以保证
   - 目前没有成熟的分布式事务解决方案

3. 人类意图理解
   - 路由决策依赖对用户意图的准确理解
   - 如果用户自己都说不清楚想要什么
   - 再好的路由也无法保证

4. 实时性要求极高的场景
   - <100ms 响应要求的场景
   - 路由决策本身就会消耗太多时间
   - 这种场景应该用检索/规则系统而非 Agent

5. 资源极度受限的环境
   - 边缘设备、嵌入式系统
   - 无法承载多种模式的代码和依赖
   - 只能用最轻量的单体模式
```

---

## 9. 与其他模式的区别

混合架构与其他五种模式有本质的不同：它不是一种直接的处理模式，而是一个**元模式**。

### 9.1 元模式 vs 基础模式

```
                  基础模式 (单体/管道/事件/微服务/网格)
                  ──────────────────────────────────────
  定义:           直接处理任务的架构范式
  关注:           如何组织推理和工具调用
  输出:           任务的处理结果
  使用者:         直接与用户交互

                  元模式 (混合)
                  ──────────
  定义:           选择和切换基础模式的系统
  关注:           如何判断当前任务用哪种模式
  输出:           路由决策 / 模式切换指令
  使用者:         作为门面层, 用户不感知
```

### 9.2 对比来看

```
混合架构 vs 单体 Agent:
  单体: 一个强 Agent 处理一切
  混合: 单体会作为混合架构中"简单任务"的处理者
  关系: 混合架构包含单体模式

混合架构 vs 管道架构:
  管道: 固定的阶段序列
  混合: 管道作为其中一个"高确定性任务"的处理者
  关系: 混合架构可以路由到管道模式

混合架构 vs 事件驱动:
  事件驱动: 反应式、异步、松耦合
  混合: 事件驱动作为"实时/异步"场景的处理者
  关系: 混合架构可以包含事件驱动的能力

混合架构 vs 微服务 Agent:
  微服务: 独立部署、服务化
  混合: 混合不关心部署方式, 关注的是推理模式
  关系: 正交关系——混合架构中的每个模式都可以是微服务

混合架构 vs 网格架构:
  网格: 多 Agent 对等协作
  混合: 网格作为"复杂协作"场景的模式选择
  关系: 混合架构可以路由到网格模式
```

### 9.3 关键差异总结

```
混合架构与其他模式的本质差异:

1. 抽象层次不同
   - 其他 5 种模式: 具体如何处理一个任务
   - 混合架构: 如何选择处理方式

2. 关注点不同
   - 其他模式关注: Token 效率、推理质量、协作方式
   - 混合架构关注: 分类准确率、路由成本、模式间切换

3. 与被包含模式的关系
   - 混合架构中, 各模式是被路由调度的"执行器"
   - 各模式可以独立存在, 但混合架构依赖各模式的存在

4. 复杂性来源不同
   - 其他模式的复杂来自 Agent 逻辑
   - 混合架构的复杂来自"何时选择哪种模式"

5. 测试策略不同
   - 其他模式: 测试特定 Agent 行为
   - 混合架构: 测试路由决策 + 每种组合的端到端流
```

---

## 10. 核心优势

### 10.1 集所有模式之大成

```
混合架构 = 每种模式的最佳使用

  简单任务 → 单体的快速和低成本
  明确流程的任务 → 管道的结构化处理
  异步实时任务 → 事件驱动的反应式处理
  独立部署需要 → 微服务的可扩縮性
  复杂协作任务 → 网格的深度推理能力

"不是一种模式替代另一种, 而是让每种模式做它最擅长的事"
```

### 10.2 成本最优化

```
Token 成本优化矩阵:

任务类型       | 网格模式  | 管道模式  | 单体模式  | 混合(最优)
───────────────┼──────────┼──────────┼──────────┼──────────
简单问答       | 100K     | 10K      | 1K       | 1K ✓
文档翻译       | 100K     | 10K      | 5K       | 5K ✓
数据分析       | 100K     | 15K      | 30K      | 15K ✓
深度研究       | 100K     | 80K      | 50K(差)  | 100K ✓
多 Agent 协作  | 200K     | N/A      | N/A      | 200K ✓

月度成本 (100K 请求):
  只用网格: 100,000 × 100K = 10B tokens
  只用单体: 100,000 × 5K = 500M (质量差)
  混合架构: 约 1.5B → 成本为网格的 15%

节省比例:
  简单任务多的场景: 成本降低 80-95%
  复杂任务多的场景: 成本降低 20-40%
```

### 10.3 优雅降级

```
混合架构的降级链:

正常流程:
  [网格模式] → 高质量复杂推理

网格模式失败时:
  [网格模式] → [管道模式]
  原因: 网格的辩论 Agent 超时
  结果: 用管道完成基本分析, 质量降低但可用

管道模式也失败时:
  [网格模式] → [管道模式] → [单体模式]
  原因: 所需数据源部分不可用
  结果: 用单体给出基于已有知识的回答

所有模式都失败时:
  返回缓存中的相似问题答案
  或返回: "目前无法处理此问题, 请稍后重试"
  或转接人工客服

对比没有降级的系统:
  单一模式失败 → 直接报错 → 用户无法使用
  混合架构失败 → 逐步降低质量 → 用户至少有部分帮助
```

### 10.4 面向未来的扩展性

```
混合架构的演进路径:

时点 1: 初期 (只有 2 种模式)
  [单体模式] + [管道模式]
  覆盖: 简单任务 + 多步骤任务
  代码: ~500 行

时点 2: 增长 (3 种模式)
  [单体] + [管道] + [事件驱动]
  覆盖: + 实时异步任务
  代码: ~1500 行

时点 3: 成熟 (4-5 种模式)
  [单体] + [管道] + [事件驱动] + [网格]
  覆盖: + 复杂研究任务
  代码: ~3000 行

时点 4: 高级 (带动态组装)
  [动态模式组装] + [成本感知路由]
  覆盖: 任意类型的任务
  代码: ~5000 行

关键: 混合架构的架构设计允许逐步增加模式
       而不需要重构现有代码
```

---

## 11. 工程实现

### 11.1 模式注册表 (Pattern Registry)

模式注册表是混合架构的基础设施——它管理和发现所有可用的执行模式。

```
// Pattern Registry 接口设计

class PatternRegistry:
    """
    模式注册表: 管理所有可用的 Agent 执行模式
    
    职责:
    - 注册/注销模式
    - 发现可用模式
    - 获取模式元数据 (成本、能力、状态)
    """
    
    _patterns: dict[str, AgentPattern] = {}
    
    def register(self, name: str, pattern: AgentPattern) -> None:
        """注册一个新的执行模式"""
        ...
    
    def unregister(self, name: str) -> None:
        """注销一个执行模式"""
        ...
    
    def get(self, name: str) -> AgentPattern:
        """获取指定模式"""
        ...
    
    def list_patterns(self) -> list[PatternInfo]:
        """列出所有可用模式及其元数据"""
        ...
    
    def get_capabilities(self, name: str) -> list[str]:
        """获取模式的能力描述"""
        ...

class PatternInfo:
    """模式元数据"""
    name: str
    description: str
    capabilities: list[str]        # 能力标签
    estimated_cost: CostProfile    # 成本预估
    status: Literal["active", "degraded", "maintenance"]
    success_rate: float            # 历史成功率
    avg_latency_ms: float           # 平均延迟
```

### 11.2 路由器组件 (Router)

路由器是混合架构的决策核心。以下是路由器的完整接口设计：

```
class Router:
    """
    智能路由器: 根据任务特征选择最佳执行模式
    
    支持多种路由策略:
    - rule_based: 规则路由
    - ml_based: ML分类器路由
    - llm_based: LLM路由
    - cost_aware: 成本感知路由
    - hybrid: 组合路由
    """
    
    def __init__(self, registry: PatternRegistry, strategy: str = "hybrid"):
        self.registry = registry
        self.strategy = strategy
        self.routing_history: list[RoutingRecord] = []
    
    def route(self, task: Task) -> RouteResult:
        """主路由方法: 根据任务返回推荐模式"""
        if self.strategy == "rule_based":
            return self._route_by_rules(task)
        elif self.strategy == "ml_based":
            return self._route_by_ml(task)
        elif self.strategy == "llm_based":
            return self._route_by_llm(task)
        elif self.strategy == "cost_aware":
            return self._route_by_cost(task)
        else:  # hybrid
            return self._route_hybrid(task)
    
    def _route_hybrid(self, task: Task) -> RouteResult:
        """
        混合路由策略:
        1. 先试规则路由 (零成本)
        2. 不确定则用 ML 分类器
        3. 仍不确定则用 LLM
        4. 兜底返回默认模式
        """
        # L1: 规则路由
        result = self._route_by_rules(task)
        if result.confidence > 0.9:
            return result
        
        # L2: ML 分类器
        result = self._route_by_ml(task)
        if result.confidence > 0.85:
            return result
        
        # L3: LLM 路由
        result = self._route_by_llm(task)
        if result.confidence > 0.7:
            return result
        
        # 兜底: 单体模式
        return RouteResult(
            mode="monolithic",
            confidence=0.5,
            reason="路由不确定, 使用安全兜底模式",
            routing_path=["rule", "ml", "llm", "fallback"]
        )
    
    def _route_by_rules(self, task: Task) -> RouteResult:
        """规则路由实现"""
        ...
    
    def _route_by_ml(self, task: Task) -> RouteResult:
        """ML 路由实现"""
        ...
    
    def _route_by_llm(self, task: Task) -> RouteResult:
        """LLM 路由实现"""
        ...
    
    def _route_by_cost(self, task: Task) -> RouteResult:
        """成本感知路由实现"""
        ...

@dataclass
class RouteResult:
    mode: str
    confidence: float
    reason: str
    routing_path: list[str]
    estimated_cost: CostProfile | None = None
```

### 11.3 状态传输机制

当模式切换时，状态需要在不同模式间传递。

```
class StateTransferManager:
    """
    状态传输管理器: 协调模式切换时的状态传递
    
    职责:
    - 从当前模式提取状态
    - 转换为目标模式所需格式
    - 注入到目标模式
    - 验证状态完整性
    """
    
    def __init__(self):
        # 注册状态适配器: (from_mode, to_mode) -> adapter
        self.adapters: dict[tuple[str, str], StateAdapter] = {}
    
    def register_adapter(
        self, 
        from_mode: str, 
        to_mode: str, 
        adapter: StateAdapter
    ):
        """注册两个模式间的状态适配器"""
        self.adapters[(from_mode, to_mode)] = adapter
    
    def transfer(
        self,
        from_mode: str,
        to_mode: str,
        current_state: AgentState
    ) -> AgentState:
        """传输状态: 从当前模式到目标模式"""
        adapter = self.adapters.get((from_mode, to_mode))
        if not adapter:
            # 如果没有注册适配器, 尝试通用转换
            return self._default_transform(current_state, to_mode)
        return adapter.transform(current_state)
    
    def _default_transform(
        self,
        state: AgentState,
        target_mode: str
    ) -> AgentState:
        """默认转换: 只保留核心状态"""
        return AgentState(
            task_id=state.task_id,
            user_input=state.user_input,
            conversation_history=state.conversation_history,
            intermediate_results={},  # 丢弃中间结果
            mode_history=state.mode_history + [state.current_mode],
            current_mode=target_mode
        )

class StateAdapter(ABC):
    """状态适配器基类"""
    
    @abstractmethod
    def transform(self, state: AgentState) -> AgentState:
        """转换状态格式"""
        pass
```

### 11.4 跨模式监控

```
class HybridMonitor:
    """
    混合架构监控: 统一追踪跨模式的请求
    
    使用 OpenTelemetry 语义约定:
    - span: 每个模式的执行
    - event: 模式切换事件
    - metric: 各模式的性能指标
    """
    
    def __init__(self, service_name: str = "hybrid-agent"):
        self.tracer = trace.get_tracer(service_name)
        self.meter = metrics.get_meter(service_name)
        
        # 定义指标
        self.route_counter = self.meter.create_counter(
            "hybrid.route.count",
            description="路由决策计数",
            unit="1"
        )
        self.mode_switch_counter = self.meter.create_counter(
            "hybrid.mode.switch",
            description="模式切换计数"
        )
        self.latency_histogram = self.meter.create_histogram(
            "hybrid.mode.latency",
            description="各模式处理延迟",
            unit="ms"
        )
        self.token_counter = self.meter.create_counter(
            "hybrid.token.usage",
            description="各模式 Token 消耗"
        )
    
    @contextmanager
    def trace_execution(self, task_id: str, mode: str):
        """创建执行追踪上下文"""
        with self.tracer.start_as_current_span(
            f"mode.{mode}",
            attributes={"task_id": task_id, "mode": mode}
        ) as span:
            yield span
    
    def record_route(
        self,
        task_type: str,
        selected_mode: str,
        confidence: float,
        routing_path: str
    ):
        """记录路由决策"""
        self.route_counter.add(1, {
            "task_type": task_type,
            "selected_mode": selected_mode,
            "routing_strategy": routing_path
        })
    
    def record_mode_switch(
        self,
        from_mode: str,
        to_mode: str,
        reason: str
    ):
        """记录模式切换"""
        self.mode_switch_counter.add(1, {
            "from_mode": from_mode,
            "to_mode": to_mode,
            "reason": reason
        })
```

### 11.5 混合架构的测试策略

```
测试层级:

L1: 单元测试 —— 测试单个模式
  - 单体模式: 每个工具调用是否正确
  - 管道模式: 每个阶段是否正确传递数据
  - 网格模式: Agent 间通信是否正确
  - 路由器: 每种路由策略的决策是否正确

L2: 集成测试 —— 测试模式 + 路由
  - "简单查询"路由 → "单体模式" → 结果符合预期
  - "复杂研究"路由 → "网格模式" → 结果符合预期
  - 路由置信度低时 → 降级到默认模式

L3: 端到端测试 —— 测试完整的混合流程
  - 请求 → 路由 → 执行 → 可能切换 → 完成
  - 覆盖所有路由路径和模式组合

L4: 混沌测试 —— 模拟失败场景
  - 某模式超时 → 应自动切换
  - 路由器宕机 → 应使用缓存的路由规则
  - LLM 路由器返回异常 JSON → 应优雅降级

测试代码示例:

def test_hybrid_router_simple_queries():
    """简单查询应该路由到单体模式"""
    router = HybridRouter(use_mock_llm=True)
    
    task = Task(user_input="今天几号？")
    result = router.route(task)
    
    assert result.mode == "monolithic"
    assert result.confidence > 0.8

def test_hybrid_router_complex_research():
    """复杂研究应该路由到网格模式"""
    router = HybridRouter(use_mock_llm=True)
    
    task = Task(
        user_input="分析2025年全球AI监管政策变化趋势,"
                   "对比欧盟、美国、中国的法规差异"
    )
    result = router.route(task)
    
    assert result.mode == "mesh"
    assert result.confidence > 0.7

def test_mode_switch_state_transfer():
    """模式切换时状态应该正确传递"""
    agent = HybridAgent()
    
    # 先在管道模式执行
    state = agent.execute_with_mode("pipeline", "分析公司财报")
    
    # 切换到网格模式
    new_state = agent.switch_mode("mesh", state)
    
    assert new_state.task_id == state.task_id
    assert new_state.current_mode == "mesh"
    assert "pipeline" in new_state.mode_history

def test_graceful_degradation():
    """复杂模式失败时应降级"""
    agent = HybridAgent()
    # 模拟网格模式故障
    agent.disable_mode("mesh")
    
    result = agent.process("复杂的比较分析任务")
    
    assert result.mode_used == "pipeline"  # 应降级到管道
    assert result.quality_warning is True  # 提示质量可能下降
```

---

## 12. 适用场景

### 12.1 企业级生产系统

```
企业级 Agent 平台的标准需求:

┌────────────────────────────────────────────────────────┐
│  需求                     │ 混合架构如何满足            │
├────────────────────────────────────────────────────────┤
│  处理各种类型的用户请求      │ 路由到最合适的模式         │
│  保证 SLA (99.9% 可用性)    │ 模式降级 + 熔断           │
│  控制 API 成本             │ 简单任务用便宜模式          │
│  支持千万级用户            │ 不同模式独立扩缩容          │
│  快速迭代新功能            │ 添加新模式不影响现有系统      │
│  多租户隔离                │ 每个租户可配置路由策略       │
│  合规与审计                │ 统一追踪跨模式请求          │
└────────────────────────────────────────────────────────┘

典型企业案例:
  - 金融服务 Agent: 简单查询(余额)+ 复杂分析(投资建议)+ 实时告警
  - 医疗健康 Agent: 预约(单体)+ 症状分析(管道)+ 药物研究(网格)
  - 法律咨询 Agent: 法条查询(单体)+ 案例分析(管道)+ 文件审查(网格)
```

### 12.2 平台型服务

```
平台型 Agent 的挑战:

一个 Agent 平台 (如 Dify、Coze、扣子) 需要:
  1. 支持新手用户创建简单的聊天机器人
  2. 支持高级用户编排复杂的工作流
  3. 支持企业用户部署生产级多 Agent 系统
  4. 自己内置的 Agent 也需要高效运行

混合架构在平台中的应用:

  平台自身的 Agent:
    [路由器] → 判断用户意图
    
    简单: "我的工作流哪里出错了?" 
      → [单体] 直接回答
    中等: "帮我创建一个客服 Agent"
      → [管道] 分步引导用户创建
    复杂: "对比这些 Agent 的性能差异"
      → [网格] 测试多个 Agent 并分析

  用户创建的 Agent:
    根据用户配置生成不同的执行模式
    用户选择"简单模式" → 单体 Agent
    用户选择"工作流模式" → 管道 Agent
    用户选择"高级模式" → 自定义混合
```

### 12.3 多模态复杂系统

```
多模态 Agent 的混合架构需求:

输入可能是:
  ┌─ 文本: "分析这张图片里的内容"
  ├─ 图片: [用户上传的截图]
  ├─ 音频: [用户语音提问]
  ├─ 视频: [用户上传的视频文件]
  └─ 代码: [用户贴了一段代码]

每种输入类型可能需要不同的处理模式:

  文本 → 单体/管道 (根据复杂度)
  图片 → 管道模式 (图像识别 → 分析 → 回答)
  音频 → 管道模式 (语音转文字 → 理解 → 回答)
  视频 → 事件驱动 (异步处理 → 通知结果)
  代码 → 网格模式 (多角度代码审查)

混合架构可以统一处理所有这些情况:
  路由器首先判断输入类型
  然后根据类型和内容复杂度选择处理模式
  不同类型的结果统一格式输出
```

---

## 13. 完整代码示例

以下是一个生产级混合架构 Agent 的实现。包含 4 种执行模式、混合路由策略和状态管理。

```python
"""
hybrid_agent.py — Hybrid Architecture Agent Framework

A production-grade hybrid agent that routes tasks to the most appropriate
execution pattern (monolithic, pipeline, mesh, or fallback) based on
task characteristics and routing strategy.

运行: python hybrid_agent.py

依赖: openai>=1.0.0  (或任何兼容的 LLM API)
      pydantic>=2.0.0
      python-dotenv>=1.0.0
"""

from __future__ import annotations

import json
import time
import uuid
import logging
from abc import ABC, abstractmethod
from dataclasses import dataclass, field, asdict
from enum import Enum
from typing import Any, Callable, Optional

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")
logger = logging.getLogger(__name__)


# ============================================================================
# 1. 基础数据类型
# ============================================================================

class TaskType(str, Enum):
    """任务类型枚举"""
    SIMPLE_QA = "simple_qa"           # 简单问答
    MULTI_STEP = "multi_step"         # 多步骤任务
    DEEP_RESEARCH = "deep_research"   # 深度研究
    UNKNOWN = "unknown"               # 未知类型


@dataclass
class Task:
    """表示一个待处理的任务"""
    id: str = field(default_factory=lambda: f"task_{uuid.uuid4().hex[:8]}")
    user_input: str = ""
    task_type: TaskType = TaskType.UNKNOWN
    complexity: float = 0.0           # 0.0 ~ 1.0
    tools_needed: list[str] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)


@dataclass
class RouteDecision:
    """路由决策结果"""
    mode: str                        # 选择的模式名
    confidence: float                # 置信度 (0~1)
    reason: str                      # 决策理由
    routing_path: list[str] = field(default_factory=list)
    estimated_tokens: int = 0


@dataclass
class AgentState:
    """Agent 执行状态, 用于在模式间传递"""
    task_id: str
    user_input: str
    current_mode: str
    conversation_history: list[dict] = field(default_factory=list)
    intermediate_results: dict = field(default_factory=dict)
    mode_history: list[str] = field(default_factory=list)
    total_tokens_used: int = 0
    start_time: float = 0.0
    end_time: float = 0.0
    error: Optional[str] = None


@dataclass
class ExecutionResult:
    """执行结果"""
    success: bool
    output: str
    mode_used: str
    tokens_used: int
    latency_ms: float
    state: AgentState
    quality_warning: bool = False
    error: Optional[str] = None


# ============================================================================
# 2. 模式接口 (Pattern Interface)
# ============================================================================

class AgentPattern(ABC):
    """所有 Agent 执行模式的抽象基类"""
    
    @property
    @abstractmethod
    def name(self) -> str:
        """模式名称"""
        pass
    
    @property
    @abstractmethod
    def description(self) -> str:
        """模式描述"""
        pass
    
    @property
    @abstractmethod
    def estimated_cost_per_call(self) -> int:
        """每次调用的预估 Token 成本"""
        pass
    
    @abstractmethod
    def can_handle(self, task: Task) -> float:
        """
        评估此模式处理该任务的适合度 (0~1)
        路由器的关键调用点
        """
        pass
    
    @abstractmethod
    def execute(self, task: Task, state: Optional[AgentState] = None) -> ExecutionResult:
        """执行任务, 返回结果"""
        pass


# ============================================================================
# 3. 四种模式的具体实现
# ============================================================================

class MonolithicPattern(AgentPattern):
    """
    单体模式 (Monolithic Pattern)
    适用于: 简单问答、单个工具调用、快速响应
    特征: 一次 LLM 调用, 0-2 个工具
    """
    
    @property
    def name(self) -> str:
        return "monolithic"
    
    @property
    def description(self) -> str:
        return "单一 LLM 调用模式, 适合简单查询 (0-2 工具, 单步推理)"
    
    @property
    def estimated_cost_per_call(self) -> int:
        return 1000
    
    def can_handle(self, task: Task) -> float:
        """单体模式适合简单任务"""
        # 简单问答: 高适合度
        if task.task_type == TaskType.SIMPLE_QA:
            return 0.95
        # 短文本且不需要工具: 高适合度
        if len(task.user_input) < 100 and len(task.tools_needed) <= 1:
            return 0.85
        # 多步骤或复杂任务: 低适合度
        if task.task_type == TaskType.DEEP_RESEARCH:
            return 0.1
        if task.complexity > 0.7:
            return 0.2
        return 0.5
    
    def execute(self, task: Task, state: Optional[AgentState] = None) -> ExecutionResult:
        """单体模式执行"""
        start = time.time()
        
        if state is None:
            state = AgentState(
                task_id=task.id,
                user_input=task.user_input,
                current_mode=self.name
            )
        
        logger.info(f"[单体模式] 处理任务: {task.user_input[:50]}...")
        
        # 模拟单体模式处理
        # 真实场景中这里调用 LLM
        simulated_tokens = 800 + len(task.user_input) * 2
        output = f"[单体模式结果] 针对「{task.user_input}」的快速回答。\n"
        output += f"      (此模式使用单次 LLM 调用, 消耗 ~{simulated_tokens} tokens)"
        
        latency = (time.time() - start) * 1000
        
        state.current_mode = self.name
        state.total_tokens_used += simulated_tokens
        state.intermediate_results["monolithic_output"] = output
        state.mode_history.append(self.name)
        
        return ExecutionResult(
            success=True,
            output=output,
            mode_used=self.name,
            tokens_used=simulated_tokens,
            latency_ms=round(latency, 2),
            state=state
        )


class PipelinePattern(AgentPattern):
    """
    管道模式 (Pipeline Pattern)
    适用于: 需要分阶段处理的多步骤任务
    特征: 3-5 个阶段, 阶段间数据传递
    """
    
    @property
    def name(self) -> str:
        return "pipeline"
    
    @property
    def description(self) -> str:
        return "分阶段管道模式, 适合多步骤处理 (分析→规划→执行→验证)"
    
    @property
    def estimated_cost_per_call(self) -> int:
        return 10000
    
    def can_handle(self, task: Task) -> float:
        """管道模式适合结构化的多步骤任务"""
        # 多步骤任务: 高适合度
        if task.task_type == TaskType.MULTI_STEP:
            return 0.9
        # 任务包含多个工具: 高适合度
        if len(task.tools_needed) >= 3:
            return 0.8
        # 任务文本较长且包含步骤关键词
        step_keywords = ["首先", "然后", "接下来", "步骤", "分别", "比较", "分析"]
        if any(kw in task.user_input for kw in step_keywords):
            return 0.75
        # 简单任务: 低适合度
        if task.complexity < 0.3:
            return 0.2
        return 0.5
    
    def execute(self, task: Task, state: Optional[AgentState] = None) -> ExecutionResult:
        """管道模式执行: 分析 → 规划 → 执行 → 整合"""
        start = time.time()
        
        if state is None:
            state = AgentState(
                task_id=task.id,
                user_input=task.user_input,
                current_mode=self.name
            )
        
        logger.info(f"[管道模式] 处理任务: {task.user_input[:50]}...")
        
        # 阶段 1: 分析 (Analysis)
        logger.info("  └─ 阶段 1/4: 分析任务")
        analysis = f"任务分析: 这是一个需要分步骤处理的任务。\n"
        analysis += f"  任务长度: {len(task.user_input)} 字符\n"
        analysis += f"  涉及工具: {task.tools_needed}\n"
        phase1_tokens = 2000
        
        # 阶段 2: 规划 (Planning)
        logger.info("  └─ 阶段 2/4: 制定计划")
        plan = "执行计划:\n"
        plan += "  Step 1: 收集相关信息\n"
        plan += "  Step 2: 分析数据\n"
        plan += "  Step 3: 综合得出结论\n"
        phase2_tokens = 1500
        
        # 阶段 3: 执行 (Execution)
        logger.info("  └─ 阶段 3/4: 执行计划")
        execution_result = f"执行结果:\n"
        execution_result += f"  Step 1: 已获取相关数据\n"
        execution_result += f"  Step 2: 已分析数据, 发现关键模式\n"
        execution_result += f"  Step 3: 已综合信息\n"
        phase3_tokens = 3000
        
        # 阶段 4: 整合 (Synthesis)
        logger.info("  └─ 阶段 4/4: 整合结果")
        synthesis = f"[管道模式结果] 针对「{task.user_input}」的分阶段分析结果。\n"
        synthesis += f"\n  任务分析:\n{analysis}\n"
        synthesis += f"  执行计划:\n{plan}\n"
        synthesis += f"  执行结果:\n{execution_result}\n"
        synthesis += f"  (此模式使用 4 阶段管道, 总消耗 ~{phase1_tokens + phase2_tokens + phase3_tokens} tokens)"
        
        total_tokens = phase1_tokens + phase2_tokens + phase3_tokens + 500
        latency = (time.time() - start) * 1000
        
        state.current_mode = self.name
        state.total_tokens_used += total_tokens
        state.intermediate_results = {
            "analysis": analysis,
            "plan": plan,
            "execution": execution_result,
            "pipeline_output": synthesis
        }
        state.mode_history.append(self.name)
        
        return ExecutionResult(
            success=True,
            output=synthesis,
            mode_used=self.name,
            tokens_used=total_tokens,
            latency_ms=round(latency, 2),
            state=state
        )


class MeshPattern(AgentPattern):
    """
    网格模式 (Mesh Pattern)
    适用于: 需要多 Agent 辩论的复杂研究
    特征: 多个 Agent 独立分析后达成共识
    """
    
    @property
    def name(self) -> str:
        return "mesh"
    
    @property
    def description(self) -> str:
        return "多 Agent 网格模式, 适合复杂研究和辩论 (多角度分析→达成共识)"
    
    @property
    def estimated_cost_per_call(self) -> int:
        return 50000
    
    def can_handle(self, task: Task) -> float:
        """网格模式适合复杂研究任务"""
        # 深度研究: 极高适合度
        if task.task_type == TaskType.DEEP_RESEARCH:
            return 0.95
        # 高复杂度: 高适合度
        if task.complexity > 0.8:
            return 0.85
        # 文本包含研究、分析、对比等关键词
        research_keywords = ["研究", "分析", "对比", "评估", "趋势", "综述", "报告"]
        keyword_count = sum(1 for kw in research_keywords if kw in task.user_input)
        if keyword_count >= 3:
            return 0.8
        # 简单任务: 极低适合度
        if task.complexity < 0.4:
            return 0.05
        return 0.3
    
    def execute(self, task: Task, state: Optional[AgentState] = None) -> ExecutionResult:
        """网格模式执行: 3 个 Agent 独立分析后汇总"""
        start = time.time()
        
        if state is None:
            state = AgentState(
                task_id=task.id,
                user_input=task.user_input,
                current_mode=self.name
            )
        
        logger.info(f"[网格模式] 处理任务: {task.user_input[:50]}...")
        logger.info("  └─ 启动 3 个分析 Agent")
        
        # Agent A: 技术分析角度
        logger.info("    ├─ Agent A (技术角度): 分析中...")
        agent_a_analysis = f"[Agent A - 技术角度分析]\n"
        agent_a_analysis += f"  从技术层面分析: {task.user_input}\n"
        agent_a_analysis += f"  主要技术趋势: ...\n"
        agent_a_analysis += f"  技术创新点: ...\n"
        agent_a_tokens = 15000
        
        # Agent B: 商业/市场角度
        logger.info("    ├─ Agent B (市场角度): 分析中...")
        agent_b_analysis = f"[Agent B - 市场角度分析]\n"
        agent_b_analysis += f"  从市场角度分析: {task.user_input}\n"
        agent_b_analysis += f"  市场规模: ...\n"
        agent_b_analysis += f"  竞争格局: ...\n"
        agent_b_tokens = 15000
        
        # Agent C: 风险/伦理角度
        logger.info("    └─ Agent C (风险角度): 分析中...")
        agent_c_analysis = f"[Agent C - 风险角度分析]\n"
        agent_c_analysis += f"  从风险角度分析: {task.user_input}\n"
        agent_c_analysis += f"  主要风险: ...\n"
        agent_c_analysis += f"  缓解策略: ...\n"
        agent_c_tokens = 15000
        
        # 仲裁 Agent: 综合三个角度的分析
        logger.info("  └─ 仲裁 Agent: 综合各方观点...")
        summary = f"[网格模式结果] 针对「{task.user_input}」的多角度综合分析。\n"
        summary += f"\n  === 多 Agent 辩论结果 ===\n\n"
        summary += f"  {agent_a_analysis}\n\n"
        summary += f"  {agent_b_analysis}\n\n"
        summary += f"  {agent_c_analysis}\n\n"
        summary += f"  --- 共识结论 ---\n"
        summary += f"  综合三个角度的分析, 可以得出以下结论: ...\n"
        summary += f"  (此模式使用 4 个 Agent (3分析+1仲裁), "
        summary += f"总消耗 ~{agent_a_tokens + agent_b_tokens + agent_c_tokens + 5000} tokens)"
        
        total_tokens = agent_a_tokens + agent_b_tokens + agent_c_tokens + 5000
        latency = (time.time() - start) * 1000
        
        state.current_mode = self.name
        state.total_tokens_used += total_tokens
        state.intermediate_results = {
            "agent_a": agent_a_analysis,
            "agent_b": agent_b_analysis,
            "agent_c": agent_c_analysis,
            "mesh_output": summary
        }
        state.mode_history.append(self.name)
        
        return ExecutionResult(
            success=True,
            output=summary,
            mode_used=self.name,
            tokens_used=total_tokens,
            latency_ms=round(latency, 2),
            state=state
        )


class FallbackPattern(AgentPattern):
    """
    降级模式 (Fallback Pattern)
    当其他模式都失败时的兜底方案
    返回基于缓存的答案或友好的错误提示
    """
    
    @property
    def name(self) -> str:
        return "fallback"
    
    @property
    def description(self) -> str:
        return "降级兜底模式, 在所有其他模式失败时使用"
    
    @property
    def estimated_cost_per_call(self) -> int:
        return 100
    
    def can_handle(self, task: Task) -> float:
        """降级模式始终返回一个低置信度"""
        return 0.1  # 永远只有 0.1, 只有所有其他模式都拒绝时才使用
    
    def execute(self, task: Task, state: Optional[AgentState] = None) -> ExecutionResult:
        """降级模式执行"""
        start = time.time()
        
        if state is None:
            state = AgentState(
                task_id=task.id,
                user_input=task.user_input,
                current_mode=self.name
            )
        
        logger.info(f"[降级模式] 处理任务: {task.user_input[:50]}...")
        
        output = (
            f"[降级模式结果] 抱歉, 目前无法完整处理您的请求。\n"
            f"  请求内容: {task.user_input}\n\n"
            f"  建议:\n"
            f"  1. 简化您的请求后重试\n"
            f"  2. 联系技术支持\n"
            f"  3. 访问帮助中心查看更多信息\n"
        )
        
        total_tokens = 100
        latency = (time.time() - start) * 1000
        
        state.current_mode = self.name
        state.total_tokens_used += total_tokens
        state.mode_history.append(self.name)
        
        return ExecutionResult(
            success=True,
            output=output,
            mode_used=self.name,
            tokens_used=total_tokens,
            latency_ms=round(latency, 2),
            state=state,
            quality_warning=True
        )


# ============================================================================
# 4. 路由器 (Router)
# ============================================================================

class HybridRouter:
    """
    混合路由器: 支持多种路由策略
    
    路由层级:
      规则路由 (零成本, 快速排除) 
      → ML 分类器 (低成本, 常见任务) 
      → LLM 路由 (高质量, 复杂判断)
      → 兜底 (安全返回)
    """
    
    def __init__(self, patterns: dict[str, AgentPattern]):
        self.patterns = patterns
        self.routing_history: list[dict] = []
    
    def route(
        self, 
        task: Task, 
        strategy: str = "hybrid",
        prefer_cost_saving: bool = True
    ) -> RouteDecision:
        """
        路由决策主入口
        
        参数:
            task: 待处理任务
            strategy: 路由策略 (rule | ml | llm | cost_aware | hybrid)
            prefer_cost_saving: 是否优先考虑节省成本
        """
        if strategy == "hybrid":
            return self._hybrid_route(task, prefer_cost_saving)
        elif strategy == "rule":
            return self._rule_route(task)
        elif strategy == "cost_aware":
            return self._cost_aware_route(task)
        else:
            return self._llm_route(task)
    
    def _rule_route(self, task: Task) -> RouteDecision:
        """基于规则的路由"""
        logger.info(f"  [规则路由] 分析任务: {task.user_input[:40]}...")
        
        # 规则 1: 极短文本 → 单体模式
        if len(task.user_input) < 30 and len(task.tools_needed) == 0:
            return RouteDecision(
                mode="monolithic",
                confidence=0.9,
                reason="极短文本, 无需工具",
                routing_path=["rule"]
            )
        
        # 规则 2: 包含研究关键词 → 网格模式
        research_kw = ["研究", "分析", "对比", "趋势", "综述", "深度", "评估"]
        research_count = sum(1 for kw in research_kw if kw in task.user_input)
        if research_count >= 3 and len(task.user_input) > 100:
            return RouteDecision(
                mode="mesh",
                confidence=0.85,
                reason=f"包含 {research_count} 个研究关键词",
                routing_path=["rule"]
            )
        
        # 规则 3: 明确的多步骤 → 管道模式
        step_kw = ["首先", "然后", "接下来", "步骤1", "步骤一", "分别"]
        if any(kw in task.user_input for kw in step_kw):
            return RouteDecision(
                mode="pipeline",
                confidence=0.8,
                reason="检测到多步骤关键词",
                routing_path=["rule"]
            )
        
        # 规则 4: 需要多个工具 → 管道模式
        if len(task.tools_needed) >= 3:
            return RouteDecision(
                mode="pipeline",
                confidence=0.75,
                reason=f"需要 {len(task.tools_needed)} 个工具",
                routing_path=["rule"]
            )
        
        # 默认: 不确定, 交给下一级路由
        return RouteDecision(
            mode="monolithic",
            confidence=0.3,
            reason="规则路由无法确定",
            routing_path=["rule"]
        )
    
    def _llm_route(self, task: Task) -> RouteDecision:
        """基于 LLM 的路由 (模拟实现, 真实场景调用 LLM API)"""
        logger.info("  [LLM路由] LLM 分析任务特征...")
        
        # 模拟 LLM 路由分析
        # 真实场景: 调用 LLM API 进行任务分类
        
        # 任务复杂度分析 (模拟)
        complexity = min(1.0, len(task.user_input) / 500)
        has_tools = len(task.tools_needed) > 0
        has_research_keywords = any(
            kw in task.user_input 
            for kw in ["分析", "研究", "对比", "评估", "报告"]
        )
        
        # LLM 综合判断 (模拟)
        if complexity < 0.3 and not has_tools:
            return RouteDecision(
                mode="monolithic",
                confidence=0.92,
                reason="LLM判断: 简单任务, 无需工具, 适合单体模式",
                routing_path=["llm"]
            )
        elif complexity < 0.6 and has_tools:
            return RouteDecision(
                mode="pipeline",
                confidence=0.88,
                reason="LLM判断: 中等复杂度, 需要工具协作, 适合管道模式",
                routing_path=["llm"]
            )
        elif has_research_keywords and complexity > 0.6:
            return RouteDecision(
                mode="mesh",
                confidence=0.85,
                reason="LLM判断: 深度研究任务, 需要多角度分析, 适合网格模式",
                routing_path=["llm"]
            )
        else:
            return RouteDecision(
                mode="monolithic",
                confidence=0.6,
                reason="LLM判断: 特征不明显, 使用安全的单体模式",
                routing_path=["llm"]
            )
    
    def _cost_aware_route(self, task: Task) -> RouteDecision:
        """成本感知路由"""
        logger.info("  [成本路由] 计算各模式的成本效益比...")
        
        best_mode = "monolithic"
        best_score = -float("inf")
        
        for name, pattern in self.patterns.items():
            capability_score = pattern.can_handle(task)
            cost_penalty = pattern.estimated_cost_per_call / 1000  # 归一化
            
            # 成本效益比: 能力 / 成本
            efficiency = capability_score / max(cost_penalty, 0.1)
            
            logger.info(
                f"    {name}: 能力={capability_score:.2f}, "
                f"成本={cost_penalty:.1f}K, 效率={efficiency:.2f}"
            )
            
            if efficiency > best_score:
                best_score = efficiency
                best_mode = name
        
        confidence = min(0.9, best_score / 2)
        return RouteDecision(
            mode=best_mode,
            confidence=confidence,
            reason=f"成本效益优先: {best_mode} 模式效率最高",
            routing_path=["cost_aware"]
        )
    
    def _hybrid_route(self, task: Task, prefer_cost_saving: bool) -> RouteDecision:
        """
        混合路由: 多层次路由决策
        
        L1: 规则路由 (零成本, 快速)
        L2: LLM 路由 (高质量)
        L3: 兜底
        """
        logger.info("  [混合路由] 开始多层次路由决策")
        
        # L1: 规则路由
        rule_result = self._rule_route(task)
        if rule_result.confidence >= 0.85:
            logger.info(f"  └─ L1 规则路由确定: {rule_result.mode} (置信度: {rule_result.confidence})")
            rule_result.routing_path = ["rule_fast"]
            self.routing_history.append({
                "task_id": task.id,
                "final_mode": rule_result.mode,
                "confidence": rule_result.confidence,
                "path": rule_result.routing_path
            })
            return rule_result
        
        logger.info(f"  └─ L1 规则路由不确定 (置信度: {rule_result.confidence}), 升级到 LLM 路由")
        
        # L2: LLM 路由
        llm_result = self._llm_route(task)
        if llm_result.confidence >= 0.8:
            logger.info(f"  └─ L2 LLM路由确定: {llm_result.mode} (置信度: {llm_result.confidence})")
            llm_result.routing_path = ["rule_fast", "llm"]
            self.routing_history.append({
                "task_id": task.id,
                "final_mode": llm_result.mode,
                "confidence": llm_result.confidence,
                "path": llm_result.routing_path
            })
            return llm_result
        
        # L3: 成本感知兜底
        logger.info(f"  └─ L2 仍不确定, 使用成本感知兜底")
        cost_result = self._cost_aware_route(task)
        cost_result.routing_path = ["rule_fast", "llm", "cost_fallback"]
        self.routing_history.append({
            "task_id": task.id,
            "final_mode": cost_result.mode,
            "confidence": cost_result.confidence,
            "path": cost_result.routing_path
        })
        return cost_result


# ============================================================================
# 5. 混合架构 Agent 主类
# ============================================================================

class HybridAgent:
    """
    混合架构 Agent 主类
    
    核心能力:
    1. 多模式注册: 可动态添加/移除执行模式
    2. 智能路由: 根据任务特征选择最佳模式
    3. 模式切换: 在执行过程中切换模式
    4. 优雅降级: 模式失败时自动降级
    5. 成本追踪: 记录和分析 Token 消耗
    """
    
    def __init__(self, router_strategy: str = "hybrid"):
        # 注册所有默认模式
        self.patterns: dict[str, AgentPattern] = {}
        self._register_default_patterns()
        
        # 创建路由器
        self.router = HybridRouter(self.patterns)
        self.router_strategy = router_strategy
        
        # 统计信息
        self.stats = {
            "total_requests": 0,
            "mode_counts": {},
            "total_tokens": 0,
            "total_latency": 0,
            "switch_count": 0,
            "fallback_count": 0
        }
    
    def _register_default_patterns(self):
        """注册默认的 4 种执行模式"""
        self.register_pattern(MonolithicPattern())
        self.register_pattern(PipelinePattern())
        self.register_pattern(MeshPattern())
        self.register_pattern(FallbackPattern())
    
    def register_pattern(self, pattern: AgentPattern):
        """注册一个执行模式"""
        self.patterns[pattern.name] = pattern
        self.stats["mode_counts"][pattern.name] = 0
        logger.info(f"注册模式: {pattern.name} — {pattern.description}")
    
    def remove_pattern(self, name: str):
        """移除一个执行模式"""
        if name in self.patterns:
            del self.patterns[name]
            logger.info(f"移除模式: {name}")
    
    def process(self, user_input: str, **kwargs) -> ExecutionResult:
        """
        处理用户输入的完整流程:
        1. 任务分析 → 2. 路由决策 → 3. 模式执行 → 4. (可选) 模式切换
        """
        self.stats["total_requests"] += 1
        start_time = time.time()
        
        # Step 1: 任务分析
        task = self._analyze_task(user_input, **kwargs)
        logger.info(f"[混合Agent] 任务分析完成: 类型={task.task_type.value}, "
                     f"复杂度={task.complexity:.2f}, 工具={task.tools_needed}")
        
        # Step 2: 路由决策
        route_decision = self.router.route(
            task, 
            strategy=self.router_strategy,
            prefer_cost_saving=kwargs.get("prefer_cost_saving", True)
        )
        logger.info(f"[混合Agent] 路由决策: 模式={route_decision.mode}, "
                     f"置信度={route_decision.confidence:.2f}, 路径={route_decision.routing_path}")
        
        # Step 3: 模式执行
        selected_pattern = self.patterns.get(route_decision.mode)
        if not selected_pattern:
            logger.warning(f"模式 {route_decision.mode} 未注册, 使用 fallback")
            selected_pattern = self.patterns["fallback"]
        
        result = selected_pattern.execute(task)
        self.stats["mode_counts"][result.mode_used] = \
            self.stats["mode_counts"].get(result.mode_used, 0) + 1
        self.stats["total_tokens"] += result.tokens_used
        self.stats["total_latency"] += result.latency_ms
        
        logger.info(f"[混合Agent] 执行完成: 模式={result.mode_used}, "
                     f"Token={result.tokens_used}, 延迟={result.latency_ms:.0f}ms, "
                     f"质量警告={'是' if result.quality_warning else '否'}")
        
        # Step 4: 质量检查 & 可选模式切换
        if result.quality_warning and kwargs.get("allow_escalation", True):
            logger.info("[混合Agent] 质量不足, 尝试升级模式...")
            escalation_result = self._try_escalate(task, result.state)
            if escalation_result:
                self.stats["switch_count"] += 1
                return escalation_result
        
        return result
    
    def _analyze_task(self, user_input: str, **kwargs) -> Task:
        """分析用户输入, 提取任务特征"""
        task = Task(user_input=user_input)
        
        # 复杂度估计: 基于输入长度和关键词
        length_factor = min(1.0, len(user_input) / 1000)
        keyword_complexity = 0.0
        
        complex_keywords = [
            "分析", "研究", "对比", "评估", "趋势", "报告",
            "总结", "综述", "为什么", "如何", "影响", "原因"
        ]
        for kw in complex_keywords:
            if kw in user_input:
                keyword_complexity += 0.1
        
        task.complexity = min(1.0, length_factor * 0.5 + keyword_complexity * 0.5)
        
        # 任务类型判断
        research_keywords = ["研究", "分析", "对比", "评估", "趋势", "综述"]
        step_keywords = ["首先", "然后", "接下来", "步骤", "分别"]
        
        research_count = sum(1 for kw in research_keywords if kw in user_input)
        step_count = sum(1 for kw in step_keywords if kw in user_input)
        
        if research_count >= 2 and task.complexity > 0.5:
            task.task_type = TaskType.DEEP_RESEARCH
        elif step_count >= 1 or task.complexity > 0.4:
            task.task_type = TaskType.MULTI_STEP
        elif task.complexity < 0.3:
            task.task_type = TaskType.SIMPLE_QA
        else:
            task.task_type = TaskType.UNKNOWN
        
        # 工具需求分析 (模拟: 关键词匹配)
        tool_map = {
            "搜索": "web_search",
            "查询": "database_query",
            "计算": "calculator",
            "翻译": "translator",
            "图片": "image_generator",
            "代码": "code_interpreter",
            "文件": "file_handler",
            "邮件": "email_sender"
        }
        task.tools_needed = [
            tool for keyword, tool in tool_map.items() 
            if keyword in user_input
        ]
        
        # 额外 metadata
        task.metadata["length"] = len(user_input)
        task.metadata["has_question_mark"] = "?" in user_input or "？" in user_input
        
        return task
    
    def _try_escalate(
        self, 
        original_task: Task, 
        current_state: AgentState
    ) -> Optional[ExecutionResult]:
        """
        尝试升级模式: 在当前模式质量不够时,
        尝试切换到更强大的模式重新处理
        """
        # 升级路径: monolithic → pipeline → mesh
        escalation_path = {
            "monolithic": "pipeline",
            "pipeline": "mesh",
            "mesh": None  # 网格模式已经是最高级
        }
        
        current_mode = current_state.current_mode
        next_mode = escalation_path.get(current_mode)
        
        if next_mode is None or next_mode not in self.patterns:
            logger.info(f"  [升级] {current_mode} 已经是最高级模式, 无法升级")
            return None
        
        logger.info(f"  [升级] 从 {current_mode} 升级到 {next_mode}")
        next_pattern = self.patterns[next_mode]
        
        # 传递状态 (只传递核心信息, 丢弃当前模式的中间结果)
        transferred_state = AgentState(
            task_id=current_state.task_id,
            user_input=current_state.user_input,
            current_mode=next_mode,
            conversation_history=current_state.conversation_history,
            intermediate_results={
                "previous_mode": current_mode,
                "previous_output": current_state.intermediate_results.get(
                    f"{current_mode}_output", ""
                )
            },
            mode_history=current_state.mode_history + [current_mode],
            total_tokens_used=current_state.total_tokens_used
        )
        
        result = next_pattern.execute(original_task, transferred_state)
        result.output = (
            f"[质量升级] 初始使用 {current_mode} 模式质量不足, "
            f"已升级到 {next_mode} 模式重新处理。\n\n"
            f"{result.output}"
        )
        
        return result
    
    def get_stats(self) -> dict:
        """获取运行时统计信息"""
        stats = dict(self.stats)
        avg_latency = (stats["total_latency"] / stats["total_requests"] 
                       if stats["total_requests"] > 0 else 0)
        avg_tokens = (stats["total_tokens"] / stats["total_requests"]
                     if stats["total_requests"] > 0 else 0)
        
        stats["avg_latency_ms"] = round(avg_latency, 2)
        stats["avg_tokens_per_request"] = round(avg_tokens, 1)
        
        # 模式分布百分比
        mode_distribution = {}
        for mode, count in stats["mode_counts"].items():
            pct = (count / stats["total_requests"] * 100 
                   if stats["total_requests"] > 0 else 0)
            mode_distribution[mode] = f"{pct:.1f}%"
        stats["mode_distribution"] = mode_distribution
        
        return stats
    
    def print_stats(self):
        """打印统计信息"""
        stats = self.get_stats()
        print("\n" + "=" * 60)
        print("          混合架构 Agent 运行统计")
        print("=" * 60)
        print(f"  总请求数:          {stats['total_requests']}")
        print(f"  总 Token 消耗:     {stats['total_tokens']:,}")
        print(f"  平均 Token/请求:   {stats['avg_tokens_per_request']}")
        print(f"  平均延迟:          {stats['avg_latency_ms']}ms")
        print(f"  模式切换次数:      {stats['switch_count']}")
        print(f"  降级调用次数:      {stats['fallback_count']}")
        print(f"\n  模式分布:")
        for mode, pct in stats['mode_distribution'].items():
            count = stats['mode_counts'].get(mode, 0)
            print(f"    {mode:15s}: {pct:>6s} ({count} 次)")
        print("=" * 60)


# ============================================================================
# 6. 使用示例与测试
# ============================================================================

def demo_simple_usage():
    """演示混合 Agent 的基本使用"""
    print("\n" + "=" * 70)
    print("                 混合架构 Agent 使用演示")
    print("=" * 70)
    
    agent = HybridAgent(router_strategy="hybrid")
    
    # 测试不同复杂度的任务
    test_cases = [
        "今天天气怎么样",                          # 简单问答 → 单体模式
        "帮我搜索 AI Agent 的最新论文, 然后总结",    # 中等任务 → 管道模式
        "深度分析 2025 年全球大模型发展趋势",        # 复杂研究 → 网格模式
    ]
    
    for i, query in enumerate(test_cases, 1):
        print(f"\n{'─' * 70}")
        print(f"  测试 {i}: {query}")
        print(f"{'─' * 70}")
        
        result = agent.process(query, allow_escalation=True)
        
        print(f"  [结果]")
        print(f"  使用模式: {result.mode_used}")
        print(f"  Token消耗: {result.tokens_used}")
        print(f"  处理延迟: {result.latency_ms:.0f}ms")
        print(f"  成功: {'是' if result.success else '否'}")
        print(f"  输出预览: {result.output[:100]}...")
    
    agent.print_stats()


def demo_escalation():
    """演示模式升级 (降级模式 → 升级)"""
    print("\n" + "=" * 70)
    print("                 模式升级演示 (降级→升级)")
    print("=" * 70)
    
    agent = HybridAgent(router_strategy="rule")
    
    # 强制走降级模式, 然后观察升级
    result = agent.process(
        "深度分析 2025 年全球大模型发展趋势、技术突破和商业落地",
        allow_escalation=True
    )
    
    print(f"  最终使用模式: {result.mode_used}")
    print(f"  模式历史: {result.state.mode_history}")
    print(f"  Token消耗: {result.tokens_used}")
    print(f"  输出预览: {result.output[:150]}...")


def demo_all_tasks():
    """批量测试不同类型的任务"""
    print("\n" + "=" * 70)
    print("                 批量任务测试")
    print("=" * 70)
    
    agent = HybridAgent(router_strategy="hybrid")
    
    queries = [
        # (输入, 预期路由类型, 描述)
        ("现在几点?", "monolithic", "极简查询"),
        ("翻译 'Hello World' 成中文", "monolithic", "简单工具调用"),
        ("搜索Python的最新版本, 并写一段示例代码", "pipeline", "多步骤"),
        ("首先查一下最近的新闻, 然后总结要点", "pipeline", "多步骤带顺序"),
        ("帮我深度研究 AI Agent 在医疗领域的应用, 分析其优势和风险", "mesh", "深度研究"),
        ("对比分析 Transformer 和 Mamba 架构的技术差异, 评估各自优劣", "mesh", "研究对比"),
    ]
    
    results = []
    for query, expected, desc in queries:
        print(f"\n  [{desc}] {query[:40]}...")
        result = agent.process(query, allow_escalation=False)
        print(f"    → 路由到: {result.mode_used} (预期: {expected}) "
              f"{'✓' if result.mode_used == expected else '✗'}")
        results.append(result)
    
    # 统计路由准确率
    correct = sum(1 for r, (_, exp, _) in zip(results, queries) 
                  if r.mode_used == exp)
    total = len(queries)
    print(f"\n  路由准确率: {correct}/{total} = {correct/total*100:.0f}%")
    
    agent.print_stats()


def demo_hybrid_router_detail():
    """详细演示路由决策过程"""
    print("\n" + "=" * 70)
    print("                 路由决策过程详细演示")
    print("=" * 70)
    
    agent = HybridAgent()
    
    test_input = "深度分析 2025 年 AI Agent 的发展趋势"
    task = agent._analyze_task(test_input)
    
    print(f"\n  任务分析结果:")
    print(f"    输入: {test_input}")
    print(f"    类型: {task.task_type.value}")
    print(f"    复杂度: {task.complexity:.2f}")
    print(f"    工具需求: {task.tools_needed}")
    
    print(f"\n  执行路由决策:")
    decision = agent.router.route(task)
    print(f"    选择模式: {decision.mode}")
    print(f"    置信度: {decision.confidence:.2f}")
    print(f"    决策路径: {decision.routing_path}")
    print(f"    理由: {decision.reason}")
    
    # 展示每种模式的适合度
    print(f"\n  各模式适合度评估:")
    for name, pattern in agent.patterns.items():
        score = pattern.can_handle(task)
        print(f"    {name:15s}: {score:.2f}")


# ============================================================================
# 7. 主入口
# ============================================================================

if __name__ == "__main__":
    print("\n" + "█" * 70)
    print("  混合架构 Agent 演示系统")
    print("  Hybrid Architecture Agent — Pattern Switching Demo")
    print("█" * 70)
    
    demo_simple_usage()
    
    print("\n" + "▔" * 70)
    demo_escalation()
    
    print("\n" + "▔" * 70)
    demo_all_tasks()
    
    print("\n" + "▔" * 70)
    demo_hybrid_router_detail()
    
    print("\n" + "█" * 70)
    print("  演示结束")
    print("█" * 70)
```

---

## 14. 生产级架构图

### 14.1 完整架构总览

```
┌────────────────────────────────────────────────────────────────────────────┐
│                           混合架构 Agent 系统                               │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                        入口层 (Gateway)                            │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │ REST API │  │ WebSocket│  │ 消息队列  │  │ 批量导入/CLI     │  │   │
│  │  │  (HTTP)  │  │  (实时)  │  │  (异步)   │  │  (离线任务)      │  │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬──────────┘  │   │
│  └───────┼─────────────┼─────────────┼────────────────┼─────────────┘   │
│          │             │             │                │                  │
│  ┌───────▼─────────────▼─────────────▼────────────────▼─────────────┐   │
│  │                   路由器层 (Router Layer)                          │   │
│  │                                                                     │
│  │  ┌──────────────────────────────────────────────────────────┐     │   │
│  │  │             多层次混合路由器 (Hybrid Router)                │     │   │
│  │  │                                                          │     │   │
│  │  │  ┌────────────┐    ┌────────────┐    ┌────────────┐     │     │   │
│  │  │  │ L1: 规则    │───►│ L2: ML 分类 │───►│ L3: LLM    │     │     │   │
│  │  │  │ (零成本)    │    │ (低成本)    │    │ (高质量)    │     │     │   │
│  │  │  └────────────┘    └────────────┘    └────────────┘     │     │   │
│  │  │                          │                               │     │   │
│  │  │                    ┌─────▼─────┐                        │     │   │
│  │  │                    │ 兜底策略    │                        │     │   │
│  │  │                    │ (安全模式)  │                        │     │   │
│  │  │                    └───────────┘                        │     │   │
│  │  └──────────────────────────────────────────────────────────┘     │   │
│  │                                                                     │
│  │  ┌──────────────────────────────────────────────────────────┐     │   │
│  │  │  路由辅助服务                                              │     │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │     │   │
│  │  │  │ 历史统计  │  │ 成本计算  │  │ 负载感知  │  │ A/B测试  │ │     │   │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │     │   │
│  │  └──────────────────────────────────────────────────────────┘     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                             │                                               │
│         ┌──────────────────┼──────────────────┬──────────────────┐          │
│         ▼                  ▼                  ▼                  ▼          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  模式执行层    │  │  模式执行层    │  │  模式执行层    │  │  模式执行层    │  │
│  │  (Pattern Execution Layer)                              │              │
│  │              │  │              │  │              │  │              │  │
│  │  单体模式     │  │  管道模式     │  │  网格模式     │  │  降级模式     │  │
│  │  (快速/便宜)  │  │ (结构化/中等) │  │ (深度/昂贵)   │  │ (兜底/安全)   │  │
│  │              │  │              │  │              │  │              │  │
│  │  ┌─────────┐  │  │  ┌─────────┐  │  │  ┌─────────┐  │  │  ┌─────────┐  │  │
│  │  │ 单次推理  │  │  │ │ 阶段1   │  │  │  │Agent A  │  │  │  │ 缓存查询 │  │  │
│  │  │ + 0-2工具 │  │  │ │ (分析)  │  │  │  │ (技术)  │  │  │  │ 静态响应 │  │  │
│  │  └─────────┘  │  │  └─────────┘  │  │  └─────────┘  │  │  └─────────┘  │  │
│  │               │  │  ┌─────────┐  │  │  ┌─────────┐  │  │              │  │
│  │               │  │  │ 阶段2   │  │  │  │Agent B  │  │  │              │  │
│  │               │  │  │ (规划)  │  │  │  │ (市场)  │  │  │              │  │
│  │               │  │  └─────────┘  │  │  └─────────┘  │  │              │  │
│  │               │  │  ┌─────────┐  │  │  ┌─────────┐  │  │              │  │
│  │               │  │  │ 阶段3   │  │  │  │Agent C  │  │  │              │  │
│  │               │  │  │ (执行)  │  │  │  │ (风险)  │  │  │              │  │
│  │               │  │  └─────────┘  │  │  └─────────┘  │  │              │  │
│  │               │  │  ┌─────────┐  │  │  ┌─────────┐  │  │              │  │
│  │               │  │  │ 阶段4   │  │  │  │仲裁Agent│  │  │              │  │
│  │               │  │  │ (验证)  │  │  │  │ (汇总)  │  │  │              │  │
│  │               │  │  └─────────┘  │  │  └─────────┘  │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
│         │                  │                  │                  │          │
│         └──────────────────┼──────────────────┼──────────────────┘          │
│                            ▼                  ▼                             │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                      状态管理层 (State Management)                   │   │
│  │                                                                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │   │
│  │  │ 状态注册表│  │ 状态适配器 │  │ 状态还原  │  │ 一致性  │           │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                             │                                               │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                       可观测性层 (Observability)                     │   │
│  │                                                                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │   │
│  │  │ 分布式追踪│  │ 指标聚合  │  │ 日志采集  │  │ 告警系统  │           │   │
│  │  │ (Trace)  │  │ (Metrics)│  │ (Logs)   │  │ (Alerts) │           │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────┘
```

### 14.2 请求处理流程详细图

```
一个请求的完整生命周期:

                    ┌────────────────────────────┐
                    │ 用户: "分析 2025 AI 趋势"   │
                    └────────────┬───────────────┘
                                 │
                    ┌────────────▼───────────────┐
                    │ 1. 网关层                   │
                    │    - 身份验证                │
                    │    - 速率限制                │
                    │    - 请求规范化              │
                    │    - 分配 Trace ID          │
                    │    Trace ID: t_e7a1b2c3     │
                    └────────────┬───────────────┘
                                 │
                    ┌────────────▼───────────────┐
                    │ 2. 任务分析                  │
                    │    - 类型: DEEP_RESEARCH     │
                    │    - 复杂度: 0.82            │
                    │    - 工具: [搜索, 数据库]     │
                    │    - 关键词: 研究+分析+趋势   │
                    └────────────┬───────────────┘
                                 │
                    ┌────────────▼───────────────┐
                    │ 3. 路由决策                  │
                    │                              │
                    │    L1: 规则路由               │
                    │      "研究"关键词3个 → 置信度0.8│
                    │      但长度不够, 降级          │
                    │                              │
                    │    L2: LLM路由               │
                    │      "深度研究任务, 需要多角度  │
                    │       分析" → mesh, 置信度0.9 │
                    │                              │
                    │    → 选择: mesh (网格模式)    │
                    └────────────┬───────────────┘
                                 │
                    ┌────────────▼───────────────┐
                    │ 4. 模式执行 — 网格模式       │
                    │                              │
                    │    ┌──────────────────┐      │
                    │    │ 启动 3 个 Agent   │      │
                    │    │ (技术/市场/风险)   │      │
                    │    └──────────────────┘      │
                    │           │                   │
                    │    ┌──────┼───┬───┬──────┐   │
                    │    │      │   │   │      │   │
                    │    ▼      ▼   ▼   ▼      ▼   │
                    │  AgentA  AgentB AgentC       │
                    │  分析技术  市场  风险         │
                    │    │      │   │   │         │
                    │    └──────┼───┼───┘         │
                    │           │   │              │
                    │    ┌──────▼───▼──────┐      │
                    │    │ 仲裁 Agent      │      │
                    │    │ 综合各方观点     │      │
                    │    │ 生成最终结论     │      │
                    │    └─────────────────┘      │
                    └────────────┬───────────────┘
                                 │
                    ┌────────────▼───────────────┐
                    │ 5. 质量检查                  │
                    │    - 结果非空: ✓             │
                    │    - 置信度: 0.85            │
                    │    - Token预算: 未超限 ✓     │
                    │    - 无需升级/切换           │
                    └────────────┬───────────────┘
                                 │
                    ┌────────────▼───────────────┐
                    │ 6. 格式化输出                │
                    │    - 统一响应格式             │
                    │    - 添加模式元数据           │
                    │    - 设置缓存 (可选)          │
                    └────────────┬───────────────┘
                                 │
                    ┌────────────▼───────────────┐
                    │ 7. 返回给用户               │
                    │    "本次分析使用了网格模式    │
                    │     (3个分析Agent + 1仲裁)...│
                    └────────────────────────────┘
```

### 14.3 模式切换流程图

```
模式切换流程 (网格模式 → 管道模式):

  当前: 网格模式 (mesh)
  问题: Agent A 超时, 无法在三分钟内达成共识

  ┌─────────────────────────────────────────────────────────┐
  │  切换触发器                                              │
  │    ┌─ Agent 超时                                         │
  │    ├─ Token 超预算                                       │
  │    ├─ 用户要求更快响应                                    │
  │    └─ 系统负载过高                                       │
  └─────────────────────────────────────────────────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────┐
  │  切换决策                                                │
  │    "网格模式不可行, 应切换到什么模式?"                      │
  │                                                         │
  │    选项:                                                 │
  │      → 管道模式: 结构化执行, 较高质量                     │
  │      → 单体模式: 快速但质量低                              │
  │      → 事件驱动: 异步但延迟高                              │
  │                                                         │
  │    选择: pipeline (管道模式)                               │
  └───────────────────────────┬─────────────────────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────┐
  │  状态提取                                                │
  │    从当前网格模式提取可传递状态:                             │
  │    {                                                      │
  │      "task_id": "t_e7a1b2c3",                            │
  │      "user_input": "分析2025 AI趋势",                     │
  │      "completed_analyses": {                              │
  │        "agent_b_market": "...",    // 已完成的市场分析     │
  │        "agent_c_risk": "..."       // 已完成的风险分析     │
  │      },                                                   │
  │      "pending": ["agent_a_technical"], // 技术分析未完成    │
  │      "total_tokens_used": 45000,                          │
  │      "start_time": 1712345678                             │
  │    }                                                      │
  └───────────────────────────┬─────────────────────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────┐
  │  状态转换                                                │
  │    通过 StateAdapter(mesh → pipeline) 转换:               │
  │    {                                                      │
  │      "task_id": "t_e7a1b2c3",                            │
  │      "user_input": "分析2025 AI趋势",                     │
  │      "phase_input": {                                     │
  │        "已完成分析": "市场分析 + 风险分析数据",              │
  │        "待补充": "技术分析"                                │
  │      },                                                   │
  │      "pipeline_stage": "phase_1_analysis",                │
  │      "mode_history": ["mesh"],                            │
  │      "total_tokens_used": 45000                           │
  │    }                                                      │
  └───────────────────────────┬─────────────────────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────┐
  │  模式执行 (管道模式)                                      │
  │    Phase 1: 整合已有分析数据 (分析阶段)                    │
  │    Phase 2: 补充技术分析 (规划阶段)                        │
  │    Phase 3: 综合所有分析 (执行阶段)                        │
  │    Phase 4: 质量验证 (验证阶段)                            │
  └───────────────────────────┬─────────────────────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────┐
  │  切换完成                                                │
  │    返回结果带切换信息:                                     │
  │    "初始使用网格模式, 因技术Agent超时,                     │
  │     切换至管道模式完成分析。"                               │
  │    总 Token: 45000 (网格) + 15000 (管道) = 60000         │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. 案例分析

### 15.1 生产系统中的混合架构实践

**案例 1: OpenAI 的 GPT 系列服务**

OpenAI 的 API 服务本身就是混合架构的典范。虽然用户看到的是单一 API 入口，背后多种处理模式协作：

```
API 入口: api.openai.com/v1/chat/completions
  ↓
路由层 (内部):
  ├─ 缓存查询 (快速命中已有答案)
  │    模式: 纯缓存 → 直接返回
  │
  ├─ 简单查询 (query < 100 tokens, 无需工具)
  │    模式: 单模型推理 (类似单体)
  │    特点: 低延迟, 高吞吐
  │
  ├─ 复杂推理 (需要 chain-of-thought)
  │    模式: 内部推理链 (类似管道)
  │    特点: 多步推理, 逐步验证
  │
  ├─ Tool Use (需要调用函数/工具)
  │    模式: 函数调用循环 (类似事件驱动)
  │    特点: 模型自主决定何时调用工具
  │
  └─ 批量处理 (Batch API)
       模式: 异步队列 (类似微服务)
       特点: 低成本, 高延迟容忍

关键设计:
  - 每个模式共享底层的模型资源池
  - 路由决策由模型本身 + 请求参数共同决定
  - 成本在不同模式间独立核算
```

**案例 2: Anthropic 的 Claude 消息平台**

Anthropic 的 Claude.ai 和 API 在架构演进中也体现了混合架构思想：

```
Claude 系统的架构层级:

L0: 快速响应层
  - 简单的问候、确认
  - 模式: 预定义的模板响应 (非 LLM)
  - 延迟: <100ms

L1: 标准推理层
  - 日常对话、知识问答
  - 模式: 单模型推理 (单体)
  - 延迟: 1-3s

L2: 工具使用层
  - 搜索、计算、代码执行
  - 模式: ReAct 循环 (事件驱动/管道)
  - 延迟: 3-10s

L3: 长上下文层
  - 长文档分析、代码库理解
  - 模式: 分片+并行处理 (管道/网格)
  - 延迟: 10-60s

L4: 研究模式 (Claude Research 等)
  - 深度研究、多步分析
  - 模式: 多 Agent 协作 (网格)
  - 延迟: 2-15min

路由策略:
  - 用户不感知路由决策
  - 模型自身判断任务复杂度 (LLM-as-Router)
  - 根据上下文长度自动切换处理策略
```

**案例 3: LangChain / LangGraph 生态**

LangChain 生态系统提供了一个"可组合的 Agent 运行时"，本质上也是一种混合架构：

```
LangChain 的 Agent 执行器:

用户定义:
  - 工具列表
  - LLM 模型
  - 执行模式 (AgentType)

可用执行模式:
  ┌─ zero-shot-react: 单步推理+行动 (类似单体)
  ├─ structured-chat: 有状态对话 (类似事件驱动)
  ├─ openai-functions: 函数调用循环 (类似事件驱动)
  ├─ plan-and-execute: 先规划再执行 (类似管道)
  └─ multi-agent: 多 Agent 协作 (类似网格)

LangGraph 的混合增强:
  - 可定义有状态的计算图 (StateGraph)
  - 节点可以是不同模式的 Agent
  - 条件边实现模式切换
  - 编译成可执行的执行计划

示例 (LangGraph 的混合模式):
  def route_decision(state):
      if state["complexity"] < 0.3:
          return "simple_agent"      # 单体
      elif state["needs_tools"]:
          return "tool_agent"        # 事件驱动/管道
      else:
          return "research_agent"    # 网格
  
  graph = StateGraph(AgentState)
  graph.add_node("simple_agent", simple_chat_agent)
  graph.add_node("tool_agent", tool_using_agent)
  graph.add_node("research_agent", multi_agent_team)
  graph.add_conditional_edges("router", route_decision)
```

**案例 4: 微软 AutoGen**

AutoGen 是一个多 Agent 对话框架，它的架构天然支持混合模式：

```
AutoGen 的架构灵活性:

单 Agent 模式:
  agent = AssistantAgent(name="assistant", llm_config=config)
  # 单一 Agent 处理所有任务 (单体模式)

两 Agent 对话:
  agent_a = AssistantAgent(...)
  agent_b = UserProxyAgent(...)
  # 两个 Agent 对话解决问题 (管道/事件驱动)

多 Agent 团队:
  group_chat = GroupChat(agents=[...], messages=[])
  manager = GroupChatManager(groupchat=group_chat)
  # 多 Agent 辩论与协作 (网格模式)

动态组合:
  # 运行时根据任务动态创建 Agent 团队
  def create_team(task_type: str):
      if task_type == "code":
          return [CoderAgent, ReviewerAgent, TesterAgent]
      elif task_type == "research":
          return [SearcherAgent, AnalystAgent, WriterAgent]

关键经验:
  - 不同任务类型需要不同数量和角色的 Agent
  - Agent 团队的组合应该在运行时动态决定
  - 共享的上下文管理是所有模式的基础
```

**案例 5: 扣子 (Coze) / Dify 等 AI 应用平台**

这些"Agent 编排平台"本身就是混合架构理念的产品化：

```
Coze/Dify 的混合架构设计:

用户创建 Bot 时可以选择的模式:
  1. 简单对话 (单体模式)
     - 一个 Prompt + 知识库
     - 适用于: 客服机器人、FAQ

  2. 工作流模式 (管道模式)
     - 可视化编排 LLM 调用 + 工具
     - 适用于: 自动化流程、数据处理

  3. Agent 模式 (事件驱动)
     - Agent 自主规划 + 调用工具
     - 适用于: 复杂任务、研究分析

  4. 多 Agent 模式 (网格模式)
     - 多个 Bot 协作
     - 适用于: 复杂项目、团队协作

平台的路由层:
  - 用户创建的每个 Bot 都是一种"模式配置"
  - 平台本身不选择模式, 而是用户选择
  - 但平台在运行时可以根据输入自动切换
    (如: 配置了"工作流"和"Agent"两种模式,
     简单输入走工作流, 复杂输入走 Agent)
```

### 15.2 生产级混合架构的关键经验

```
从生产系统中学到的 7 条经验:

1. 路由决策必须有可观测性
   - 每次路由决策都要记录: 输入特征、决策路径、置信度
   - 定期分析路由准确率, 发现偏差
   - 设置路由质量仪表板

2. "默认单体"是最安全的选择
   - 当路由不确定时, 默认使用单体模式
   - 单体模式的容错性最好
   - 宁可简单回答, 不要出错

3. 渐进式路由优于一次性路由
   - 先试低成本路由 (规则、ML)
   - 不确定再升级到高成本路由 (LLM)
   - 避免"大炮打蚊子"

4. 模式间要有清晰的失败边界
   - 一个模式失败不应该影响其他模式
   - 使用舱壁模式 (Bulkhead Pattern) 隔离
   - 失败模式自动熔断

5. 状态传递保持最小化
   - 只传递核心上下文 (用户输入、历史、任务ID)
   - 丢弃中间处理细节
   - 使用统一的状态协议

6. 用 Token 预算控制复杂度
   - 为每个请求设置 Token 预算上限
   - 预算内路由只能选择"买得起"的模式
   - 超预算的请求自动降级

7. 混合架构需要混合测试
   - 单元测试: 每个模式独立
   - 集成测试: 路由 + 执行
   - 端到端测试: 完整链路
   - 混沌测试: 模拟模式失败
```

### 15.3 未来演进方向

```
混合架构的未来趋势:

1. 自适应路由 (Self-Adaptive Routing)
   - 系统自动学习哪些任务应该走哪些模式
   - 基于历史反馈持续优化路由策略
   - 不再需要人工编写规则或训练分类器

2. 微架构动态组装 (Dynamic Micro-Architecture)
   - 运行时根据任务动态生成专用 Agent 模式
   - 不是一个固定模式库, 而是一个模式生成器
   - 每个请求都可能获得一个"定制"的架构

3. 成本感知的实时优化 (Real-time Cost Optimization)
   - 路由时实时计算各模式的预期 Token 成本
   - 结合当前模型定价动态选择最优模式
   - 为不同用户层提供不同的成本策略

4. 跨会话模式学习 (Cross-session Pattern Learning)
   - 从所有用户的交互中学习模式选择
   - 模式选择逐渐从"人工定义"转向"数据驱动"
   - 各模式的性能数据反哺路由决策

5. 多模态路由 (Multi-modal Routing)
   - 根据输入类型 (文本/图像/音频/视频) 自动路由
   - 不同模态有不同的最佳模式
   - 多模态输入需要多模式组合
```

---

## 总结

混合架构不仅仅是一种技术选择，更是一种设计哲学：**没有银弹，但有选择**。

它承认每种架构模式都有其优势和局限，而真正的生产级系统需要根据不同场景做出不同的选择。混合架构提供的不是"另一种做事方式"，而是"选择做事方式的能力"。

在实际应用中，建议从一个简单的两层混合开始（单体 + 一种复杂模式），逐步演进到更丰富的模式组合。避免过度设计，让架构随着需求自然生长。

```
开始: 单体模式 (MVP)
  │
  ▼
第一阶段: 单体 + 管道 (应对多步骤任务)
  │
  ▼
第二阶段: 单体 + 管道 + 事件驱动 (应对异步实时任务)
  │
  ▼
第三阶段: 单体 + 管道 + 事件驱动 + 网格 (应对复杂研究)
  │
  ▼
成熟期: 动态路由 + 成本感知 + 模式切换 + 优雅降级
        (完整的混合架构)
```

记住：混合架构的最终目标不是"使用尽可能多的模式"，而是"让每次交互都使用最合适的模式"。
