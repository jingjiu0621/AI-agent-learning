# Advanced AI Agent Development — 完整学习目录

阅读 @MyAgentLearning/overview.md 和 @MyAgentLearning/readme.md 明确任务，读取进度 @MyAgentLearning/current.md ，开始下一步学习。


## 1 目录概览

```
 1  foundation/               AI Agent 基础理论与范式
 2  llm-advanced/             大语言模型原理与推理进阶
 3  prompt-engineering/       提示工程高级技术与系统设计
 4  tool-use/                 工具调用与外部世界交互
 5  agents-core/              Agent 核心架构与工程实现
 6  memory/                   记忆系统与知识管理
 7  planning/                 思考、规划与推理
 8  multi-agent/              多智能体协作与组织
 9  rag-advanced/             高级 RAG 与知识检索
10  observability/            可观测性、监控与调试
11  evaluation/               评估体系与测试
12  safety/                   安全、对齐与合规
13  production/               生产化部署与运维
14  architecture/             系统架构设计
15  projects/                 实战项目
16  papers/                   前沿论文阅读
17  tools/                    常用工具与框架
18  references/               参考资料
```

---

## 2 学习路径图

```
                                                    难度
                                                     ★~★★★★★
                                                    ─────────
PHASE 1 ── 入门筑基
                                    ┌──────────────────┐
  1. foundation  ───────────────────│ Agent 基础理论与认知 │  ★★☆☆☆
                                    └──────────────────┘
                                             │
                           ┌─────────────────┼─────────────────┐
                           ▼                 ▼                  ▼
                  ┌───────────────┐  ┌───────────────┐  ┌──────────────┐
  3. prompt-eng   │ Prompt 系统工程 │  │  Function Call  │  │ LLM 原理进阶 │  2. llm-adv
                  │     ★★☆☆☆     │  │     ★★★☆☆     │  │   ★★★☆☆    │
                  └───────┬───────┘  └───────┬───────┘  └──────┬───────┘
                          │                  │                  │
                          └─────────┬────────┘                 │
                                    ▼                          │
PHASE 2 ── 核心能力                  │                          │
                                    │                          │
                           ┌──────────────────┐               │
  5. agents-core  ─────────│  Agent 核心架构实现 │◄──────────────┘
                           └──────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
           ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
  6. memory │    记忆系统    │ │    规划推理    │ │  高级 RAG    │  9. rag-adv
           │   ★★★★☆      │ │   ★★★★☆      │ │   ★★★★☆     │
           └──────────────┘ └──────────────┘ └──────────────┘
                    │               │               │
                    └───────┬───────┘               │
                            ▼                       │
PHASE 3 ── 进阶协作          │                       │
                            │                       │
                   ┌──────────────────┐             │
  8. multi-agent ──│ 多智能体协作与组织  │◄────────────┘
                   └──────────────────┘
                  ★★★★★
                            │
                            ▼
              ┌─────────────────────────┐
  10. obs  ──│ 可观测性 │ 评估体系 │ 安全 │── 12. safety
              │   ★★★☆☆    ★★★☆☆   ★★★★☆ │
  11. eval ──│ 监控 · 调试 · 测试 · 对齐 │
              └─────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
  13. prod  ──│   生产部署 · 扩展 · 运维   │
              │        ★★★★☆            │
  14. arch  ──│   系统架构 · 企业级设计    │
              │        ★★★★☆            │
              └─────────────────────────┘
                            │
                            ▼
PHASE 4 ── 实战与研究
              ┌─────────────────────────┐
  15. projects│  L1 入门 → L2 进阶       │
              │  → L3 高级 → L4 生产     │
              │   → Capstone 毕业项目     │
              ├─────────────────────────┤
  16. papers  │  论文精读：ReAct → SWE   │
              │  ★★★★★                  │
              ├─────────────────────────┤
  17. tools   │  框架工具链学习           │
  18. refs    │  参考资料索引             │
              └─────────────────────────┘
```

### 难度说明

| 星级 | 含义 | 前置要求 |
|------|------|---------|
| ★★☆☆☆ | 入门 | 有编程基础即可 |
| ★★★☆☆ | 基础 | 理解 LLM 基本概念 |
| ★★★★☆ | 进阶 | 掌握 Agent 核心原理 |
| ★★★★★ | 高级 | 综合工程与研究能力 |

### 推荐学习路线

```
ROUTE A ── 全栈精通（推荐）
  1 → 3 → 4 → 2 → 5 → 6 → 7 → 9 → 8 → 10+11+12 → 13+14 → 15 → 16

ROUTE B ── 快速上手
  1 → 3 → 4 → 5 → 15(L1-L2) → 17 → 后续按需深入

ROUTE C ── 架构与生产
  1 → 2 → 5 → 6 → 7 → 10 → 13 → 14

ROUTE D ── 多 Agent 与高级 RAG
  1 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 15(L3-L4)
```

---

## 3 完整目录树

---

### 1 foundation/ — AI Agent 基础理论与范式

```
1.1  what-is-agent             Agent 定义、认知基础与设计范式
      1.1.1  agent-definition      Agent 核心定义：感知环境、自主推理、采取行动
      1.1.2  key-properties        四大特性：自主性 / 反应性 / 主动性 / 社交能力
      1.1.3  vs-traditional        与传统程序的区别：状态驱动 vs 意图驱动
      1.1.4  weak-vs-strong        Weak Agent vs Strong Agent 划分
      1.1.5  agent-environment     Agent 与环境的关系：完全/部分可观测、确定性/随机性
      1.1.6  traditional-arch      经典 Agent 架构分类：反应式 / 模型式 / 目标驱动 / 效用驱动 / 学习型
      1.1.7  bdi-model             BDI 模型：信念-愿望-意图框架

1.2  history                   从符号主义到 LLM Agent 的演进
      1.2.1  rule-based           早期基于规则的专家系统（MYCIN, DENDRAL）
      1.2.2  rl-agents           强化学习 Agent（DQN, PPO, AlphaGo）
      1.2.3  mdp-basics          马尔可夫决策过程与规划算法入门
      1.2.4  llm-emergence       LLM 涌现能力如何催生新一代 Agent
      1.2.5  paradigm-shift       范式迁移：规则驱动 → 数据驱动 → 意图驱动
      1.2.6  timeline             关键历史节点与里程碑论文
      1.2.7  trends               当前趋势：Agentic AI、多模态、具身智能

1.3  cognitive-arch            认知架构：感知 → 思考 → 行动 循环
      1.3.1  perceive             感知模块：多源信息接收、解析与表示
      1.3.2  think                思考模块：推理、规划、决策与反思
      1.3.3  act                  行动模块：工具调用、输出生成与环境交互
      1.3.4  feedback-loop        反馈循环：观察结果驱动下一轮思考
      1.3.5  SOAR-model           SOAR 统一认知理论对 Agent 架构的影响
      1.3.6  ACT-R-model          ACT-R 认知架构与工作记忆模型

1.4  agent-types               主流 Agent 类型与模式
      1.4.1  simple-io            Simple I/O Agent：单次输入输出
      1.4.2  chain-thought        Chain-of-Thought Agent：逐步推理链
      1.4.3  react-agent          ReAct Agent：推理与行动交织
      1.4.4  reflexion-agent      Reflexion Agent：带自我反思的迭代改进
      1.4.5  plan-execute         Plan-and-Execute Agent：先规划后执行
      1.4.6  multi-step           Multi-step Agent：多步工具调用与状态跟踪

1.5  comparison                主流框架对比与选型
      1.5.1  langchain-profile    LangChain / LangGraph：灵活图编排
      1.5.2  crewai-profile       CrewAI：角色分工与层级协作
      1.5.3  autogen-profile      AutoGen：对话驱动多 Agent 与代码执行
      1.5.4  handcrafted          手写 Agent 的适用场景与优势
      1.5.5  selection-guide      框架选择决策矩阵
      1.5.6  benchmark-comp       框架性能与开发效率基准对比
```

---

### 2 llm-advanced/ — 大语言模型原理与推理进阶

```
2.1  tokenization              Tokenizer 原理与优化
      2.1.1  bpe                   BPE 算法详解：从字节到子词的合并策略
      2.1.2  wordpiece             WordPiece / SentencePiece / Unigram LM
      2.1.3  tiktoken              tiktoken 库与 Token 精确计数
      2.1.4  special-tokens        Special Tokens 的作用与设计
      2.1.5  token-efficiency      Token 效率优化：减少 Token 数的工程技巧
      2.1.6  multilingual-token    多语言 Tokenizer 的挑战与方案

2.2  inference                 推理机制与性能优化
      2.2.1  auto-regressive       自回归生成原理：逐 Token 预测
      2.2.2  kv-cache              KV Cache 机制与显存优化
      2.2.3  speculative-decode    Speculative Decoding：草稿模型加速
      2.2.4  quantization          量化推理：INT4/INT8/FP16 精度与性能权衡
      2.2.5  flash-attention       Flash Attention：IO 感知的注意力算法
      2.2.6  batch-inference       批量推理与动态 batching
      2.2.7  llm-limitations       LLM 固有局限：幻觉、知识截止、推理谬误与缓解

2.3  context-window            上下文窗口技术
      2.3.1  positional-embed      Positional Embedding 演进：绝对 → 相对 → RoPE
      2.3.2  rope                  RoPE 旋转位置编码详解
      2.3.3  context-extension     窗口扩展技术（YaRN, NTK-aware, 线性缩放）
      2.3.4  context-compression    上下文压缩（LLMLingua, Selective Context）
      2.3.5  sliding-window        Sliding Window Attention
      2.3.6  long-context-bench    长上下文评估（Needle In A Haystack）

2.4  structured-output         结构化输出
      2.4.1  json-mode             OpenAI / Claude JSON Mode 原理
      2.4.2  constrained-decoding  约束解码：Grammar-constrained 生成
      2.4.3  json-schema-enforce   JSON Schema 强制输出
      2.4.4  tool-call-structure   Function Calling 的结构化输出本质
      2.4.5  partial-json          部分 JSON 解析与流式结构化输出
      2.4.6  open-source-libs      开源结构化输出库（Outlines, JsonFormer, lm-format-enforcer）

2.5  api-design                LLM API 设计模式
      2.5.1  streaming-arch        流式架构：SSE / WebSocket / Chunked Transfer
      2.5.2  retry-strategy        重试策略：指数退避、jitter、幂等性保证
      2.5.3  rate-limit            速率限制处理与请求排队
      2.5.4  batch-api             Batch API 与异步结果轮询
      2.5.5  fallback-model        模型降级回退策略
      2.5.6  cost-tracking         API 调用成本追踪与预算控制
```

---

### 3 prompt-engineering/ — 提示工程高级技术与系统设计

```
3.1  advanced-techniques       高级推理提示技术
      3.1.1  chain-of-thought      Chain-of-Thought：零样本 / 少样本思维链
      3.1.2  tree-of-thoughts      Tree of Thoughts：多路径搜索与评估
      3.1.3  graph-of-thoughts     Graph of Thoughts：非线性推理网络
      3.1.4  self-consistency      Self-Consistency：多路径投票聚合
      3.1.5  automatic-cot         Auto-CoT：自动生成推理链
      3.1.6  least-to-most         Least-to-Most Prompting：从简到繁分解

3.2  dynamic-prompts           动态 Prompt 组装与管理
      3.2.1  template-engine       Prompt Template Engine（Jinja2, Mustache）
      3.2.2  context-injection     动态上下文注入技术
      3.2.3  prompt-chaining       Prompt 链式组装与管道
      3.2.4  conditional-prompts   条件分支 Prompt 策略
      3.2.5  version-control       Prompt 版本管理与 diff
      3.2.6  multi-lang-prompts    多语言 / 多角色 Prompt 切换

3.3  prompt-optimization       Prompt 优化
      3.3.1  manual-optimization   人工优化方法论与最佳实践
      3.3.2  automatic-optimize    自动优化（DSPy, OPRO, APE）
      3.3.3  ab-testing            A/B 测试：指标设计、对照组、统计分析
      3.3.4  prompt-evolution      进化算法优化 Prompt
      3.3.5  meta-prompts          元 Prompt：让 LLM 自己优化 Prompt
      3.3.6  regression-test       Prompt 回归测试套件

3.4  system-design             System Prompt 设计模式
      3.4.1  persona-pattern       角色扮演：身份设定与行为边界
      3.4.2  instruction-hierarchy  指令层级：系统级 → 用户级 → 工具级
      3.4.3  constraint-listing    约束列举：明确禁止项与限制边界
      3.4.4  format-specification  格式规范：输出结构与格式要求
      3.4.5  few-shot-embedding    少样本示例嵌入策略
      3.4.6  security-boundary     安全边界：拒绝策略与注入防护声明
```

---

### 4 tool-use/ — 工具调用与外部世界交互

```
4.1  function-calling          Function Calling 原理深度剖析，执行层，文件层，信息层，编排层，扩展层
      4.1.1  openai-fc            OpenAI Function Calling 协议详解
      4.1.2  claude-tool-use      Claude Tool Use 协议详解
      4.1.3  FC-lifecycle         函数调用生命周期：声明 → 触发 → 执行 → 结果注入
      4.1.4  parallel-fc          并行函数调用机制
      4.1.5  FC-limitations       函数调用局限性与边界情况
      4.1.6  multi-turn-fc        多轮函数调用与状态跟踪

4.2  tool-definition           工具定义规范
      4.2.1  json-schema          JSON Schema 工具定义语法
      4.2.2  openapi-spec         OpenAPI / Swagger 规范
      4.2.3  description-engineering 工具描述优化：让 LLM 准确选择工具
      4.2.4  parameter-design     参数设计：必选/可选/默认值/枚举约束
      4.2.5  naming-convention    工具命名规范与语义
      4.2.6  auto-generation      API → Tool Definition 自动生成

4.3  tool-execution            工具执行引擎
      4.3.1  execution-pipeline   执行管道架构设计
      4.3.2  parallel-execution   并行调用与依赖图管理
      4.3.3  error-propagation    错误传播与优雅降级
      4.3.4  timeout-handling     超时处理机制
      4.3.5  retry-per-tool       单工具级别重试策略
      4.3.6  execution-context    执行上下文：状态共享与隔离

4.4  tool-retrieval            工具检索（海量工具场景）
      4.4.1  tool-indexing        工具索引构建与 embedding
      4.4.2  semantic-tool-search 语义工具搜索
      4.4.3  tool-reranking       工具检索重排序
      4.4.4  dynamic-tool-registry 动态注册与发现
      4.4.5  tool-fusion          多路检索结果融合
      4.4.6  few-shot-tool-demo   工具使用示例动态选取

4.5  custom-tools              自定义工具开发
      4.5.1  tool-wrapper         工具封装模式（Wrapper 层设计）
      4.5.2  auth-integration     工具认证与授权集成
      4.5.3  stateful-tools       有状态工具：会话、事务、游标
      4.5.4  file-tools           文件系统工具（读/写/搜索/结构化）
      4.5.5  web-tools            网络工具（HTTP 请求、爬虫、API 调用）
      4.5.6  code-execution       代码执行引擎（Python / SQL 沙箱隔离）
      4.5.7  browser-automation   浏览器自动化与视觉操作（Playwright、截图理解）

4.6  advanced-capabilities     高级 Agent 交互能力
      4.6.1  embodied-agent       具身智能与机器人 Agent（视觉-语言-动作闭环）
      4.6.2  multimodal-agent     多模态 Agent（图像 / 音频 / 视频 多通道输入输出）
```

---

### 5 agents-core/ — Agent 核心架构与工程实现

```
5.1  loop-engine               Agent 主循环：Thought-Action-Observation
      5.1.1  loop-architecture    主循环架构：单轮 vs 多轮迭代
      5.1.2  thought-generation   思考生成与内部推理轨迹
      5.1.3  action-selection     行动选择：贪婪 / 加权 / MCTS 策略
      5.1.4  observation-parse    观察结果解析与归因
      5.1.5  loop-termination     终止条件：任务完成 / 最大步数 / 异常
      5.1.6  loop-variants        循环变体：ReAct Loop / Reflexion Loop / Plan-Execute Loop

5.2  state-management          Agent 状态管理
      5.2.1  state-definition     状态构成：会话状态、任务状态、环境状态
      5.2.2  state-transition     状态机与状态转移图
      5.2.3  state-persistence    状态持久化（Redis / Database / 文件）
      5.2.4  state-recovery       状态恢复与故障转移
      5.2.5  state-conflict       状态冲突检测与解决
      5.2.6  immutable-state      不可变状态与事件溯源模式

5.3  message-history           消息历史管理与压缩
      5.3.1  window-strategy      滑动窗口策略：固定窗口 / Token 预算窗口
      5.3.2  summarization        历史摘要：自动摘要 / 分层摘要 / 增量摘要
      5.3.3  hierarchical-arch    分层消息架构：核心 + 压缩 + 长期记忆
      5.3.4  lossy-compression    有损压缩：关键信息抽取
      5.3.5  priority-pruning     按重要性裁剪消息
      5.3.6  multi-modal-history  多模态消息历史管理

5.4  error-handling            错误处理与恢复策略
      5.4.1  error-taxonomy       Agent 错误分类（LLM / 工具 / 逻辑）
      5.4.2  graceful-degradation 优雅降级策略
      5.4.3  retry-logic          智能重试：自适应退避、替代路径
      5.4.4  human-escalation     人工介入与升级机制
      5.4.5  checkpoint-rollback  检查点与回滚
      5.4.6  circuit-breaker      熔断机制：防止级联故障

5.5  streaming                 流式输出与中间状态
      5.5.1  token-streaming      Token 流推送（SSE / WebSocket）
      5.5.2  intermediate-state   中间状态暴露：思考过程 / 工具调用进度
      5.5.3  partial-output       部分输出渲染与增量更新
      5.5.4  cancellation         流式取消与中断
      5.5.5  buffer-flush         Buffer 与 Flush 策略
      5.5.6  streaming-error      流式传输中的错误处理

5.6  frameworks                框架源码深度分析
      5.6.1  langgraph-src        LangGraph 源码：StateGraph / MessageGraph / 节点与边
      5.6.2  langgraph-checkpoint LangGraph Checkpointer 与持久化机制
      5.6.3  langgraph-stream     LangGraph 流式与中断机制
      5.6.4  crewai-process       CrewAI 源码：Agent / Task / Process / Crew
      5.6.5  autogen-orchestra    AutoGen 源码：Orchestration 与对话管理
      5.6.6  handcrafted-patterns 手写 Agent 参考实现（最小 ReAct → 完整框架）
```

---

### 6 memory/ — 记忆系统与知识管理

```
6.1  short-term                短期记忆：上下文窗口管理
      6.1.1  context-budget       上下文预算跟踪与分配
      6.1.2  recency-bias         近因偏差与注意力衰减
      6.1.3  token-accounting     Token 精确计算、预留与回收
      6.1.4  truncation-strategies 截断策略：头部 / 尾部 / 关键信息保留
      6.1.5  short-term-expiry    短期记忆过期与刷新
      6.1.6  working-memory-link  短期↔工作记忆的桥接

6.2  long-term                 长期记忆：向量存储与结构化存储
      6.2.1  vector-stores        向量数据库选型（Chroma / Pinecone / Weaviate / Qdrant / Milvus）
      6.2.2  embedding-models     嵌入模型选型（text-embedding-3 / Cohere / BGE）
      6.2.3  structured-memory    结构化记忆（SQLite / PostgreSQL）
      6.2.4  key-value-memory     KV 存储（Redis / Memcached）
      6.2.5  hybrid-storage       混合存储架构：向量 + 结构化 + 全文
      6.2.6  memory-serialization 记忆序列化与反序列化

6.3  working-memory            工作记忆：当前任务状态
      6.3.1  scratchpad           Scratchpad：Agent 的推理草稿纸
      6.3.2  task-stack           任务栈：子任务嵌套与优先级
      6.3.3  progress-tracking    进度跟踪：已完成 / 进行中 / 待办
      6.3.4  intermediate-results 中间结果缓存与引用
      6.3.5  focus-management     注意力焦点管理
      6.3.6  working-memory-cap   容量限制与溢出处理

6.4  memory-retrieval          记忆检索策略
      6.4.1  semantic-search      语义检索：余弦相似度 / MIP 搜索
      6.4.2  temporal-decay       时间衰减：艾宾浩斯遗忘曲线应用
      6.4.3  importance-scoring   重要性评分：频率 / 最近 / 相关性 / 显著性
      6.4.4  multi-modal-retrieve 多模态记忆检索
      6.4.5  retrieval-fusion     多路检索结果融合（RRF, CC）
      6.4.6  retrieval-threshold  检索阈值自适应调整

6.5  memory-consolidation      记忆整合与巩固
      6.5.1  episodic-summary     情节记忆摘要
      6.5.2  semantic-abstraction 语义抽象：具体事实 → 通用知识
      6.5.3  hierarchical-merge   分层合并：短期 → 中期 → 长期
      6.5.4  deduplication        记忆去重与冲突消解
      6.5.5  spaced-repetition    间隔重复：基于遗忘曲线的巩固
      6.5.6  sleep-consolidation  "休眠"整合：空闲时批量处理

6.6  external-memory           外部记忆源
      6.6.1  knowledge-graph      知识图谱构建（Neo4j, NetworkX）
      6.6.2  graph-traversal      图谱遍历：实体 / 关系 / 路径
      6.6.3  database-memory      关系型数据库作为记忆后端
      6.6.4  document-store       文档存储（MongoDB, Elasticsearch）
      6.6.5  external-api-memory  外部 API 作为记忆源
      6.6.6  memory-federation    多源记忆联合查询
```

---

### 7 planning/ — 思考、规划与推理

```
7.1  task-decomposition        任务分解策略
      7.1.1  hierarchical-split  层次分解：顶层目标 → 子任务 → 原子步骤
      7.1.2  sequential-plan     顺序规划：步骤依赖与约束
      7.1.3  DAG-plan            DAG 规划：并行任务识别
      7.1.4  decomposition-strategies 分解策略（基于功能 / 数据流 / 依赖）
      7.1.5  recursion-depth     递归深度与终止条件
      7.1.6  decomposition-eval  分解质量评估

7.2  reAct-pattern             ReAct 模式深度解析
      7.2.1  reasoning-tracing   推理轨迹：Thought 生成与记录
      7.2.2  action-observation  行动-观察循环详解
      7.2.3  ReAct-variants      ReAct 变体（ReAct+CoT / ReAct+Reflexion）
      7.2.4  prompting-ReAct     ReAct 的 Prompt 工程设计
      7.2.5  ReAct-limitations   局限性：循环陷阱 / 效率问题
      7.2.6  ReAct-implementation 从零实现 ReAct（Python / TypeScript）

7.3  reflection                反思机制与自我改进
      7.3.1  self-reflection     自我反思：回顾行为与改进
      7.3.2  external-critic     外部评审（Critic Agent）
      7.3.3  reflection-triggers 反思触发条件：错误 / 低置信度 / 周期性
      7.3.4  reflection-depth    反思深度控制：表层纠正 vs 深层重构
      7.3.5  reflection-storage  反思结果存储与复用
      7.3.6  reflexion-loop      Reflexion 完整循环实现

7.4  plan-execute              Plan-and-Execute 模式
      7.4.1  plan-creation       计划生成：LLM 规划器设计
      7.4.2  plan-validation     计划验证：可行性 / 一致性 / 完整性
      7.4.3  plan-execution      计划执行：按序 / 并行调度
      7.4.4  plan-adaptation     执行中动态调整
      7.4.5  plan-repair         失败重规划
      7.4.6  hierarchical-plan-exe 分层 Plan-and-Execute

7.5  hierarchical-planning     分层规划
      7.5.1  abstraction-levels  抽象层次：战略 → 战术 → 执行
      7.5.2  top-down-planning   自顶向下逐层细化
      7.5.3  bottom-up-planning  自底向上从原子合成
      7.5.4  HTN-planning        HTN（Hierarchical Task Network）
      7.5.5  subgoal-dependency  子目标依赖管理
      7.5.6  cross-level-comm    跨层级通信协调

7.6  uncertainty               不确定性处理与回退
      7.6.1  uncertainty-sources 不确定性来源（模型 / 工具 / 环境）
      7.6.2  confidence-scoring  置信度评分：token 概率 / 一致性
      7.6.3  fallback-strategies 回退：降级 / 替代方案 / 求助
      7.6.4  confirmation-loop   确认循环：不确定时主动询问
      7.6.5  probabilistic-planning 概率规划：期望效用最大化
      7.6.6  risk-aware-planning 风险感知规划
```

---

### 8 multi-agent/ — 多智能体协作与组织

```
8.1  communication             Agent 间通信协议
      8.1.1  message-format      消息格式（JSON-RPC / Protobuf / 自定义）
      8.1.2  synchronous-comm    同步通信：请求-响应
      8.1.3  async-comm          异步通信：消息队列 / Pub-Sub
      8.1.4  broadcast           广播模式：信息分发
      8.1.5  structured-dialogue 结构化对话：角色标记 / 消息类型 / 路由
      8.1.6  comm-overhead       通信开销优化

8.2  coordination              协调机制
      8.2.1  centralized-coord   中心化协调：Orchestrator 模式
      8.2.2  decentralized-coord 去中心化协调：市场 / 拍卖 / 共识
      8.2.3  voting-mechanisms   投票机制：多数决 / 加权 / 排序
      8.2.4  consensus-building  共识算法：Paxos / Raft 在 Agent 中的应用
      8.2.5  leader-election     领导者选举
      8.2.6  conflict-resolution 冲突解决机制

8.3  delegation                任务分配与委派
      8.3.1  task-routing        任务路由：基于能力 / 负载 / 优先级
      8.3.2  capability-registry 能力注册与发现
      8.3.3  workload-balancing  动态负载均衡
      8.3.4  delegation-contract 委派契约：职责 / 期限 / 质量标准
      8.3.5  result-verification 结果验证与反馈
      8.3.6  re-delegation       重新委派：失败转移

8.4  debate-negotiation        辩论与协商机制
      8.4.1  debate-protocol     辩论协议：轮流发言 / 举证 / 反驳
      8.4.2  argument-structure  论证结构：主张 → 证据 → 推理
      8.4.3  negotiation-strategies 协商策略：让步 / 妥协 / 利益交换
      8.4.4  multi-round-debate  多轮辩论深入分析
      8.4.5  judge-agent         评审 Agent 裁决
      8.4.6  debate-limitations  辩论机制的局限性

8.5  swarm-patterns            群体组织模式
      8.5.1  manager-worker      Manager-Worker：分发-汇报
      8.5.2  parliament-pattern  议会模式：讨论 → 提案 → 投票 → 执行
      8.5.3  market-pattern      市场模式：供需匹配 / 竞价 / 交易
      8.5.4  pipeline-pattern    流水线模式：链式处理
      8.5.5  mesh-pattern        网状模式：全互联对等通信
      8.5.6  game-theory         博弈论：纳什均衡 / 合作博弈 / 机制设计
      8.5.7  hybrid-patterns     混合模式：多层架构

8.6  frameworks                多 Agent 框架源码分析
      8.6.1  crewai-orch         CrewAI 编排流程源码
      8.6.2  crewai-tools        CrewAI 工具与任务协作
      8.6.3  autogen-conversation AutoGen 对话驱动模式源码
      8.6.4  autogen-agentchat   AutoGen AgentChat 协议分析
      8.6.5  langgraph-multi     LangGraph 多 Agent 状态管理与路由
      8.6.6  framework-compare   多 Agent 框架核心机制对比
```

---

### 9 rag-advanced/ — 高级 RAG 与知识检索

```
9.1  indexing                  索引策略
      9.1.1  chunking-strategies  分块策略：固定大小 / 语义 / 递归
      9.1.2  metadata-annotation  元数据标注：来源 / 时间 / 类型 / 权限
      9.1.3  multi-level-index    多级索引：摘要 + 全文 + 向量
      9.1.4  hierarchical-index   层次索引：文档 → 章节 → 段落
      9.1.5  index-update         增量索引更新
      9.1.6  index-optimization   索引优化：压缩 / 分片 / 缓存

9.2  retrieval                 检索策略
      9.2.1  dense-retrieval      稠密检索：向量相似度搜索
      9.2.2  sparse-retrieval     稀疏检索：BM25 / TF-IDF
      9.2.3  hybrid-search        Hybrid Search：稠密 + 稀疏融合
      9.2.4  re-ranking           重排序（Cross-Encoder / Cohere Rerank）
      9.2.5  multi-stage-retrieval 多级检索：粗排 → 精排
      9.2.6  retrieval-latency    检索延迟优化（HNSW / IVF / 量化）

9.3  advanced-rag              高级 RAG 模式
      9.3.1  agentic-rag          Agentic RAG：Agent 自主决策检索
      9.3.2  self-rag             Self-RAG：自我反思式检索
      9.3.3  corrective-rag       Corrective RAG：检索结果自动修正
      9.3.4  adaptive-rag         Adaptive RAG：自适应检索策略
      9.3.5  speculative-rag      推测性 RAG：预检索与缓存
      9.3.6  fusion-rag           Fusion RAG：多源生成融合

9.4  graph-rag                 Graph RAG：知识图谱增强检索
      9.4.1  knowledge-graph-basics 知识图谱基础（实体 / 关系 / 三元组）
      9.4.2  graph-construction   自动构建：NER + 关系抽取
      9.4.3  graph-rag-flow       Graph RAG 流程：检索 → 子图 → 推理
      9.4.4  graph-query          图谱查询（Cypher / SPARQL / GraphQL）
      9.4.5  community-detection  社区发现（Louvain / Leiden）
      9.4.6  microsoft-graphrag   Microsoft GraphRAG 方案深度解析

9.5  code-rag                  Code RAG：代码库理解
      9.5.1  code-chunking        代码分块：函数 / 类 / 文件
      9.5.2  code-indexing        代码索引：AST / 符号表 / 调用图
      9.5.3  code-retrieval       代码检索：语义 + 结构混合搜索
      9.5.4  repo-mapping         仓库映射：依赖图与文件关系
      9.5.5  cross-file-reasoning 跨文件推理与上下文聚合
      9.5.6  code-gen-with-rag    RAG 增强代码生成

9.6  query-understanding       查询理解与改写
      9.6.1  query-classification 查询分类：事实性 / 推理性 / 指令性
      9.6.2  query-rewriting      查询改写：扩展 / 分解 / 规范化
      9.6.3  query-decomposition  子问题拆分
      9.6.4  hypothetical-questions HyDE 假设性文档生成
      9.6.5  query-intent-detection 查询意图检测
      9.6.6  multi-turn-query     多轮查询上下文管理

9.7  evaluation                RAG 评估体系
      9.7.1  retrieval-metrics    检索指标（Recall / Precision / MRR / NDCG）
      9.7.2  generation-metrics   生成指标（Faithfulness / Relevance / Completeness）
      9.7.3  end-to-end-eval      端到端总体质量评分
      9.7.4  eval-datasets        评估数据集（RGB / RECALL / KILT / CRUD）
      9.7.5  automated-eval-tools 自动评估工具（RAGAS / TruLens / DeepEval）
      9.7.6  human-eval-framework 人工评估框架
```

---

### 10 observability/ — 可观测性、监控与调试

```
10.1  logging                  Agent 结构化日志
      10.1.1  structured-logging   结构化日志（JSON / ECS / OpenTelemetry）
      10.1.2  thought-logging      思考过程轨迹记录
      10.1.3  action-logging       工具调用日志：入参 / 出参 / 耗时
      10.1.4  observation-logging  观察结果与异常标记
      10.1.5  log-levels           日志级别设计（DEBUG → INFO → WARN → ERROR）
      10.1.6  log-aggregation      日志聚合（ELK / Loki / CloudWatch）

10.2  tracing                  链路追踪
      10.2.1  span-design          Span 设计：Root / Child / Event
      10.2.2  trace-context        链路上下文传播（Baggage / Headers）
      10.2.3  langsmith-integration LangSmith 深度集成
      10.2.4  open-telemetry       OpenTelemetry（Tracing SDK / Collector / Exporter）
      10.2.5  custom-tracer        自定义追踪器实现
      10.2.6  distributed-tracing  分布式 Agent 链路追踪

10.3  monitoring               实时监控
      10.3.1  token-monitoring     Token 消耗监控（输入 / 输出 / 总量 / 成本）
      10.3.2  latency-monitoring   延迟监控（P50 / P95 / P99）
      10.3.3  error-rate-tracking  错误率追踪（LLM / 工具 / 超时）
      10.3.4  dashboard-building   监控仪表盘（Grafana / Datadog）
      10.3.5  alerting             告警规则设计
      10.3.6  SLO-budgeting        SLO 定义与错误预算管理

10.4  debugging                Agent 调试技术
      10.4.1  step-through-debug   逐步调试：暂停 → 检查状态 → 继续
      10.4.2  replay-debugging     回放调试：重放记录轨迹
      10.4.3  prompt-inspection    Prompt 检查：实际发出的完整 Prompt
      10.4.4  tool-call-inspection 工具调用检查：参数 / 结果 / 耗时
      10.4.5  state-inspection     Agent 状态快照检查
      10.4.6  common-patterns      常见 Bug 模式与调试策略

10.5  visualization            Agent 行为可视化
      10.5.1  thought-map          思维图谱：推理路径可视化
      10.5.2  action-graph         行动图谱：工具调用 DAG
      10.5.3  memory-visualization 记忆检索路径与关联强度
      10.5.4  agent-interaction    多 Agent 交互可视化
      10.5.5  real-time-vis        实时流式可视化
      10.5.6  post-hoc-analysis    事后分析可视化
```

---

### 11 evaluation/ — 评估体系与测试

```
11.1  metrics                  评估指标
      11.1.1  success-rate        任务成功率 / 目标达成率
      11.1.2  efficiency          效率指标：步数 / Token 消耗 / 耗时
      11.1.3  robustness          鲁棒性：干扰 / 变异 / 边界条件
      11.1.4  adaptability        适应性：新场景 / 跨领域迁移
      11.1.5  reliability         可靠性：一致性 / 可复现性
      11.1.6  composite-score     综合评分：加权多指标融合

11.2  benchmarks               基准测试
      11.2.1  GAIA                GAIA：通用 AI Agent 评估
      11.2.2  SWE-bench           SWE-bench：软件工程任务评估
      11.2.3  ToolBench           ToolBench：工具使用能力评估
      11.2.4  AgentBench          AgentBench：多维度综合评测
      11.2.5  WebArena            WebArena：Web 环境 Agent 评估
      11.2.6  custom-benchmark    自定义基准构建

11.3  automated-eval           自动评估
      11.3.1  LLM-as-Judge        LLM 评审：评分标准与偏差控制
      11.3.2  pairwise-comparison  成对比较：A/B 测试
      11.3.3  rubric-based-eval    评分准则：维度 / 等级 / 示例
      11.3.4  unit-test-for-agents Agent 单元测试（工具 Mock / 环境模拟）
      11.3.5  integration-test     集成测试：多步骤 / 多工具流程
      11.3.6  eval-pipeline        自动化评估管道

11.4  adversarial-testing      对抗测试与压力测试
      11.4.1  prompt-attacks       Prompt 攻击：注入 / 越狱 / 混淆
      11.4.2  edge-cases           边界情况：极端输入 / 空值 / 冲突指令
      11.4.3  stress-testing       压力测试：高并发 / 长上下文 / 多工具
      11.4.4  robustness-attacks   鲁棒性攻击：输入扰动 / 噪声
      11.4.5  simulation-env       仿真测试环境构建
      11.4.6  failure-mode-fuzzing 失败模式模糊测试
      11.4.7  adversarial-training 对抗训练提升防御

11.5  regression               回测与持续评估
      11.5.1  regression-suite     回归测试套件维护
      11.5.2  version-compare      版本间性能差异对比
      11.5.3  continuous-eval      持续评估（CI/CD 集成）
      11.5.4  drift-detection      性能漂移检测
      11.5.5  historical-analysis  历史趋势分析
      11.5.6  regression-triage    回归问题分类与优先级
```

---

### 12 safety/ — 安全、对齐与合规

```
12.1  prompt-injection         Prompt 注入防护
      12.1.1  injection-types      注入类型：直接 / 间接 / 越狱
      12.1.2  input-sanitization   输入清洗：特殊字符 / 指令标记
      12.1.3  instruction-defense  指令防御：分隔 / 隔离 / 权限分离
      12.1.4  parameterized-prompts 参数化 Prompt：模板与用户输入分离
      12.1.5  detection-strategies 注入检测（分类器 / 困惑度 / 规则）
      12.1.6  red-teaming          红队测试：模拟攻击

12.2  output-guardrails        输出护栏
      12.2.1  content-filtering    内容过滤：敏感词 / PII / 仇恨言论
      12.2.2  format-validation    格式验证（JSON Schema / 正则 / 类型）
      12.2.3  semantic-guardrails  语义护栏：事实性 / 安全性 / 有用性
      12.2.4  guardrails-libraries 护栏库（NVIDIA NeMo / Guardrails AI）
      12.2.5  output-sampling      输出采样控制（temperature / top_p / logit_bias）
      12.2.6  bypass-resistance    绕过防护评估

12.3  input-validation         输入验证与清洗
      12.3.1  schema-validation    Schema 验证（JSON Schema / Pydantic）
      12.3.2  type-checking        类型检查与转换
      12.3.3  boundary-checking    边界检查：长度 / 范围 / 格式
      12.3.4  encoding-sanitize    编码清洗（Unicode / Base64 / 混淆）
      12.3.5  rate-limit-input     输入速率限制
      12.3.6  allowlist-blocklist  白名单 / 黑名单策略

12.4  permission-control       权限控制
      12.4.1  tool-authorization   工具授权（Scope / Role / Resource）
      12.4.2  least-privilege      最小权限原则
      12.4.3  dynamic-permissions  上下文感知动态权限
      12.4.4  approval-workflow    高风险操作审批流程
      12.4.5  audit-trail          权限审计轨迹
      12.4.6  isolation-boundary   隔离边界：容器 / 沙箱 / 虚拟环境
      12.4.7  human-in-the-loop    Human-in-the-Loop：人工确认 / 审批网关 / 升级

12.5  alignment                对齐技术
      12.5.1  RLHF                RLHF：基于人类反馈的强化学习
      12.5.2  DPO                 DPO：直接偏好优化
      12.5.3  constitutional-ai    Constitutional AI：原则驱动自我改进
      12.5.4  preference-collection 偏好数据收集
      12.5.5  reward-modeling      奖励模型设计训练
      12.5.6  alignment-evaluation 对齐评估框架

12.6  audit                    审计与合规
      12.6.1  audit-logging        不可篡改的审计日志
      12.6.2  compliance-frameworks 合规框架（GDPR / SOC2 / HIPAA）
      12.6.3  data-governance      数据治理：存储 / 保留 / 删除
      12.6.4  transparency-report  透明度与模型行为说明
      12.6.5  bias-detection       偏见检测与公平性
      12.6.6  ethics-review        伦理审查流程
```

---

### 13 production/ — 生产化部署与运维

```
13.1  deployment              部署方案
      13.1.1  docker-deploy        Docker 多阶段构建与镜像优化
      13.1.2  k8s-deploy           Kubernetes：Pod / Service / HPA
      13.1.3  serverless-deploy    Serverless（AWS Lambda / Vercel / Cloudflare Workers）
      13.1.4  edge-deploy          Edge 部署：延迟优化
      13.1.5  env-management       环境管理：开发 / 测试 / 预发布 / 生产
      13.1.6  blue-green-canary    蓝绿部署与金丝雀发布

13.2  scaling                 扩展策略
      13.2.1  concurrency-model    并发模型：进程 / 线程 / 协程
      13.2.2  load-balancing       负载均衡（Round Robin / Least Connections / Consistent Hashing）
      13.2.3  rate-limiting        限流（Token Bucket / Leaky Bucket / 滑动窗口）
      13.2.4  auto-scaling         自动扩缩容：基于 CPU / QPS / 队列深度
      13.2.5  queue-based-load     基于队列的负载平滑
      13.2.6  database-scaling     数据库扩展：读写分离 / 分片

13.3  caching                 缓存策略
      13.3.1  response-cache       Response Cache：精确命中与 TTL
      13.3.2  semantic-cache       语义缓存：相似输入命中
      13.3.3  cache-layer-design   缓存分层：L1(LRU) + L2(Redis) + L3(DB)
      13.3.4  cache-invalidation   失效策略：主动 / 被动 / 事件驱动
      13.3.5  cache-warming        缓存预热
      13.3.6  distributed-cache    分布式缓存（Redis Cluster / Memcached）

13.4  cost-optimization       成本优化
      13.4.1  token-compression    Token 压缩：精简 Prompt / 摘要替代
      13.4.2  model-routing        模型路由：简单任务小模型 / 复杂任务大模型
      13.4.3  cache-hit-optimize   缓存命中率优化
      13.4.4  batch-optimization   请求合并批处理
      13.4.5  cost-budgeting       预算配额 / 告警 / 可视化
      13.4.6  model-distillation   模型蒸馏：专有小模型训练

13.5  reliability             可靠性
      13.5.1  retry-patterns       重试：指数退避 / jitter / 最大次数
      13.5.2  degradation-strategies 降级：功能裁剪 / 数据降级
      13.5.3  circuit-breaker      熔断器：状态机 / 半开 / 恢复
      13.5.4  bulkhead-pattern     隔舱模式：资源隔离
      13.5.5  timeout-control      超时控制（连接 / 读取 / 总超时）
      13.5.6  chaos-engineering    混沌工程：故障注入

13.6  ci-cd                   CI/CD 管道
      13.6.1  pipeline-design      Pipeline：触发 → 构建 → 测试 → 部署
      13.6.2  test-automation      自动化测试（单元 / 集成 / E2E / Agent Eval）
      13.6.3  linting-formatting   代码检查：Lint / Format / Type Check
      13.6.4  artifact-management  制品管理（Docker Image / Model Weight）
      13.6.5  env-promotion        环境晋升：dev → staging → prod
      13.6.6  rollback-strategy    快速回滚 / 渐进回滚
```

---

### 14 architecture/ — 系统架构设计

```
14.1  patterns                架构模式
      14.1.1  event-driven        Event-driven Agent：事件源 / 订阅 / 响应
      14.1.2  pipeline-arch       管道架构：链式处理
      14.1.3  mesh-architecture   网格架构：Agent Mesh
      14.1.4  microservice-agent  微服务 Agent：独立部署与缩放
      14.1.5  monolithic-agent    单体 Agent：MVP 与验证场景
      14.1.6  hybrid-arch         混合架构：场景适配

14.2  enterprise              企业级架构
      14.2.1  multi-tenant        多租户：隔离 / 配额 / 计费
      14.2.2  sso-integration     SSO 集成（OAuth / SAML / LDAP）
      14.2.3  data-residency      数据驻留合规
      14.2.4  enterprise-security 企业安全：加密 / 审计 / DLP
      14.2.5  sla-design          SLA 设计：可用性 / 响应时间 / 准确性
      14.2.6  integration-patterns 企业系统集成（ERP / CRM）

14.3  microservices           微服务化 Agent
      14.3.1  service-decomposition 服务拆分：按能力 / 领域 / 数据
      14.3.2  inter-service-comm   服务间通信（gRPC / Message Queue / REST）
      14.3.3  api-gateway          API 网关：路由 / 限流 / 认证
      14.3.4  service-discovery    服务发现（Consul / K8s DNS）
      14.3.5  config-management    配置管理：集中式 vs 分布式
      14.3.6  observability-stack  微服务可观测性栈

14.4  case-studies            真实系统案例分析
      14.4.1  chatbot-arch        客服 Agent 架构
      14.4.2  coding-agent-arch   编程 Agent（Cursor / Copilot）架构
      14.4.3  research-agent-arch 研究 Agent 架构
      14.4.4  automation-agent    自动化 Agent 架构
      14.4.5  enterprise-agent    企业 Agent 平台架构
      14.4.6  lessons-learned     架构设计经验教训
```

---

### 15 projects/ — 实战项目

```
15.1  level-1-basic           入门项目
      15.1.1  chat-agent          基础对话 Agent：LLM API 封装与流式响应
      15.1.2  weather-agent       天气查询 Agent：单一工具调用
      15.1.3  search-agent        搜索 Agent：网络搜索工具整合
      15.1.4  calculator-agent    计算 Agent：数学工具调用
      15.1.5  todo-agent          Todo List Agent：简单 CRUD
      15.1.6  tech-stack          L1 技术栈：OpenAI API + Python + FastAPI

15.2  level-2-intermediate    进阶项目
      15.2.1  research-assistant  研究助手：多工具 + 长上下文
      15.2.2  email-agent         邮件 Agent：读写 / 分类 / 回复
      15.2.3  memory-chatbot      带持久化记忆的聊天 Agent
      15.2.4  data-analysis-agent 数据分析 Agent：代码执行 + 可视化
      15.2.5  personal-assistant  个人助手：日历 + 邮件 + 提醒
      15.2.6  tech-stack          L2 技术栈：LangChain + Chroma + FastAPI

15.3  level-3-advanced        高级项目
      15.3.1  multi-agent-research 多 Agent 研究系统
      15.3.2  customer-support     多角色客服系统
      15.3.3  document-qa          全流程 RAG 文档问答系统
      15.3.4  code-review-agent    代码评审 Agent
      15.3.5  meeting-notes-agent  会议纪要 Agent
      15.3.6  tech-stack           L3 技术栈：LangGraph + CrewAI + Qdrant

15.4  level-4-production      生产级项目
      15.4.1  production-rag       生产级 RAG + A/B 测试
      15.4.2  agent-platform       多租户 Agent 平台
      15.4.3  autonomous-coding    SWE-bench 级自主编程 Agent
      15.4.4  enterprise-knowledge 企业知识库 Agent
      15.4.5  multi-agent-orch     生产级多 Agent 编排
      15.4.6  tech-stack           L4 技术栈：K8s + LangGraph + OpenTelemetry

15.5  capstone                 Capstone 毕业项目
      15.5.1  topic-selection      选题方向（全能助理 / 代码审查 / 多 Agent 调研 / 客服）
      15.5.2  arch-design          架构设计文档
      15.5.3  core-implementation  核心代码实现
      15.5.4  eval-report          评估报告
      15.5.5  presentation         演示与评审（任务完成度 / 架构合理性 / 安全性）
```

---

### 16 papers/ — 前沿论文阅读

```
16.1  reAct                   ReAct: Synergizing Reasoning and Acting
      16.1.1  core-idea           推理与行动的协同
      16.1.2  architecture        Thought-Action-Observation 循环
      16.1.3  experiments         HotpotQA / AlfWorld
      16.1.4  impact              ReAct 模式的广泛采纳
      16.1.5  code-implementation 最小复现
      16.1.6  extensions          后续扩展工作

16.2  reflexion               Reflexion: Language Agents with Verbal Reinforcement
      16.2.1  core-idea           语言强化学习
      16.2.2  architecture        Actor + Evaluator + Memory
      16.2.3  self-reflection     自我反思机制
      16.2.4  experiments         决策与推理任务
      16.2.5  limitations         反思深度与效率
      16.2.6  improvements        改进方向

16.3  tree-of-thoughts        Tree of Thoughts
      16.3.1  core-idea           多路径树状搜索
      16.3.2  search-algorithms   BFS / DFS / 剪枝
      16.3.3  state-evaluation    启发式评分
      16.3.4  experiments         24 点游戏 / 创意写作
      16.3.5  vs-chain-thought    ToT vs CoT 对比
      16.3.6  computational-cost  计算成本分析

16.4  graph-of-thoughts       Graph of Thoughts
      16.4.1  core-idea           非线性推理图
      16.4.2  graph-operations    分解 / 聚合 / 精炼 / 生成
      16.4.3  reasoning-topologies 链 / 树 / 图 / 环
      16.4.4  experiments         排序 / 文档合并 / 关键词计数
      16.4.5  flexibility-analysis 相比 ToT 的优势
      16.4.6  implementation      图推理引擎实现

16.5  agent-ln                AgentLN: Agentic Language Networks
      16.5.1  core-idea           Agent 间语言网络
      16.5.2  network-topology    层级 / 图谱 / 动态
      16.5.3  communication-protocol 通信协议设计
      16.5.4  emergent-behavior   网络级涌现智能
      16.5.5  experiments         协作问题求解
      16.5.6  scalability         可扩展性分析

16.6  swe-agent               SWE-Agent: Agent-Computer Interfaces
      16.6.1  core-idea           Agent-计算机接口设计
      16.6.2  architecture        感知 / 行动 / 记忆
      16.6.3  agent-computer-iface ACI 设计原则
      16.6.4  SWE-bench-results   评测结果分析
      16.6.5  key-design-choices  关键设计决策
      16.6.6  real-world-impact   实际应用影响

16.7  generative-agents       Generative Agents: Interactive Simulacra
      16.7.1  core-idea           可信人类行为模拟
      16.7.2  architecture        记忆流 + 反思 + 计划
      16.7.3  memory-stream       体验 → 反思 → 高层次推断
      16.7.4  social-emergence    信息传播 / 关系形成
      16.7.5  sandbox-simulation  Smallville 小镇
      16.7.6  applications        游戏 / 社交模拟 / 角色扮演

16.8  tool-former             Toolformer / ToolLLM
      16.8.1  toolformer-core     Toolformer：自监督工具学习
      16.8.2  toolformer-arch     API 调用集成
      16.8.3  toolllm-core        ToolLLM：指令调优工具使用
      16.8.4  toolllm-framework   ToolBench + DFSDT
      16.8.5  comparison          Toolformer vs ToolLLM vs GPT-4 FC
      16.8.6  future-directions   工具学习未来方向
```

---

### 17 tools/ — 常用工具与框架

```
17.1  langchain               LangChain / LangGraph
      17.1.1  LCEL                LangChain Expression Language
      17.1.2  chains              Chain：LLMChain / Sequential / Router
      17.1.3  agents              AgentExecutor / AgentType
      17.1.4  tools               内置与自定义工具
      17.1.5  langgraph-basics    StateGraph / MessageGraph / 节点
      17.1.6  langgraph-advanced  Checkpointer / Streaming / 中断恢复

17.2  crewai                  CrewAI
      17.2.1  core-concepts       核心：Agent / Task / Crew / Process
      17.2.2  agent-definition    角色 / 目标 / 能力 / LLM 配置
      17.2.3  task-assignment     任务分配与依赖
      17.2.4  process-types       Sequential / Hierarchical
      17.2.5  tool-integration    内置 / 自定义 / LangChain 桥接
      17.2.6  crew-output         结果聚合与汇报

17.3  autogen                 AutoGen
      17.3.1  core-concepts       核心：Agent / Conversation / Orchestration
      17.3.2  assistant-agent     AssistantAgent：LLM 驱动
      17.3.3  user-proxy          UserProxyAgent：人机交互
      17.3.4  group-chat          GroupChat：多 Agent 群聊
      17.3.5  tool-registration   工具注册与执行
      17.3.6  extended-agents     GPTAssistantAgent / MultimodalAgent

17.4  openai                  OpenAI SDK
      17.4.1  chat-completion     Chat Completion：消息格式与参数
      17.4.2  streaming           Stream 事件处理
      17.4.3  function-calling    定义 / 调用 / 解析
      17.4.4  structured-outputs  JSON Schema 约束
      17.4.5  assistants-api      Assistants：Thread / Run / Message
      17.4.6  embeddings          text-embedding-3 系列

17.5  anthropic               Anthropic SDK
      17.5.1  messages-api        Messages：消息格式与 System Prompt
      17.5.2  streaming           Stream Events / SSE
      17.5.3  tool-use            工具定义 / 调用 / 结果注入
      17.5.4  extended-thinking   思维链展示
      17.5.5  prompt-caching      缓存策略与成本节省
      17.5.6  token-counting      Token 精确计数

17.6  llamaindex              LlamaIndex
      17.6.1  core-concepts       核心：Documents / Nodes / Index / Retriever
      17.6.2  indexing            VectorStoreIndex / SummaryIndex
      17.6.3  retrieval           检索器 / 融合 / 重排序
      17.6.4  query-engine        查询构造 / 响应合成
      17.6.5  agent-capabilities  工具调用 / 记忆
      17.6.6  advanced-patterns   Agentic RAG / Router

17.7  other                   其他工具与框架
      17.7.1  Vercel-AI-SDK      Vercel AI SDK：多 Provider 流式工具调用
      17.7.2  DSPy               自动优化 LM 程序
      17.7.3  Haystack           Pipeline 检索架构
      17.7.4  Semantic-Kernel    Microsoft AI 编排框架
      17.7.5  MetaGPT            多角色元编程 Agent
      17.7.6  Agno               轻量多模态 Agent 框架
      17.7.7  Atomic-Agents      原子化可组合 Agent
      17.7.8  Flowise-Langflow   低代码 Agent 构建
      17.7.9  Ollama-vLLM        本地 LLM 部署推理
```

---

### 18 references/ — 参考资料

```
18.1  books                   推荐书籍
      18.1.1  building-llm-apps    Building LLM Applications（Valentina Alto）
      18.1.2  llm-engineering      LLM Engineering（Karpathy 系列）
      18.1.3  deep-learning        Deep Learning（Goodfellow / Bengio / Courville）
      18.1.4  speech-patterns      Speech & Language Processing（Jurafsky & Martin）
      18.1.5  ai-modern-approach   AI: A Modern Approach（Russell & Norvig）
      18.1.6  hands-on-llm         Hands-On LLM（LLMOps 实践）

18.2  courses                 在线课程
      18.2.1  deeplearning-ai     DeepLearning.AI：Prompt Engineering / LangChain / RAG
      18.2.2  stanford-cs224n     Stanford CS224N：NLP with Deep Learning
      18.2.3  stanford-cs229      Stanford CS229：Machine Learning
      18.2.4  huggingface-course  Hugging Face Course：Transformers / Diffusers
      18.2.5  fastai              Fast.ai：Practical Deep Learning
      18.2.6  mit-6s191           MIT 6.S191：Introduction to Deep Learning

18.3  blogs                   精选博客
      18.3.1  openai-blog         OpenAI Blog：GPT-4 / Function Calling
      18.3.2  anthropic-blog      Anthropic Blog：Claude / Tool Use / Safety
      18.3.3  langchain-blog      LangChain Blog：Agent 设计模式
      18.3.4  lilianweng          Lilian Weng：Agent 综述 / Prompt Engineering
      18.3.5  simonw              Simon Willison：LLM 工程实践
      18.3.6  arxiv-sanity        Arxiv Sanity：最新论文追踪

18.4  videos                  视频教程
      18.4.1  andrej-karpathy     Andrej Karpathy：Let's Build GPT / Tokenizer
      18.4.2  stanford-lectures   Stanford CS224N 录播
      18.4.3  yannic-kilcher      Yannic Kilcher：论文精读
      18.4.4  llm-university      LLM University（Cohere）：Embedding / RAG
      18.4.5  conference-talks    NeurIPS / ICML / ICLR Agent 讲座
      18.4.6  practical-tutorials 实践教程：Agent 搭建与框架使用

18.5  glossary                术语表
      18.5.1  core-terms          核心术语：Agent / LLM / RAG / CoT / Fine-tuning
      18.5.2  architecture-terms  架构术语：ReAct / Reflexion / MoE / LoRA
      18.5.3  evaluation-terms    评估术语：Faithfulness / Perplexity / BLEU / ROUGE
      18.5.4  safety-terms        安全术语：Alignment / RLHF / DPO / Red-teaming
      18.5.5  infrastructure-terms 基础设施：Orchestration / Serving / Quantization
      18.5.6  chinese-english     中英对照索引表
```

---

## 4 统计

| 层级 | 数量 |
|------|------|
| 顶层模块 | 18 |
| 中层节点 (x.y) | 96 |
| 叶子节点 (x.y.z) | 554 |
| **总计** | **668** |

> —— 完整分层知识体系，共 668 个知识点 ——
