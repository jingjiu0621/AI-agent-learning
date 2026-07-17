# 9.3.1 Agentic RAG — 智能体自主决策检索

## 背景与问题

传统 RAG（Naive RAG）的运作模式是"每轮必检"：无论用户问什么，系统都先检索知识库，再将检索结果拼入 Prompt 交给 LLM 生成。这种模式在以下场景中暴露出严重问题：

1. **不必要的检索浪费 Token**：当用户问"今天天气怎么样"这类不需要外部知识的问题时，系统仍然执行检索，浪费了检索延迟和上下文窗口的 Token 配额。

2. **检索噪声污染生成**：检索结果中混入无关或低相关信息时，LLM 可能被误导，产生幻觉或偏离用户意图。

3. **缺乏多步推理能力**：复杂问题需要拆解为多个子问题，每个子问题可能需要不同的检索策略和检索源，传统 RAG 无法实现这种"思考-检索-再思考"的迭代过程。

4. **检索与生成脱节**：检索结果直接拼入上下文，LLM 被动接收——模型无法主动选择"我要检索什么"。

**核心矛盾**：RAG 的价值在于提供外部知识，但并非所有查询都需要外部知识；即使需要，检索的时机、精度、范围也需要智能判断，而非机械执行。

Agentic RAG 的解决思路是：**将检索封装为 Agent 的一项工具，让 LLM 自主决策是否检索、何时检索、检索什么、是否继续检索**。

---

## 核心原理

### Agent 主循环中注入检索工具

Agentic RAG 的核心不是简单的"检索+生成"两步，而是完整的 Agent 循环：

```
Agent 主循环（Thought-Action-Observation）

   ┌─────────────────────────────────────┐
   │  1. Thought（思考）                   │
   │     "用户问的是...我需要知道...才能回答" │
   └──────────────┬──────────────────────┘
                  │
                  ▼
   ┌─────────────────────────────────────┐
   │  2. Decision（决策）                  │
   │     "需要检索吗？ → 是/否"            │
   └──────────────┬──────────────────────┘
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
   ┌──────────┐     ┌──────────────┐
   │ 需要检索  │     │ 直接回答      │
   │ 调用工具  │     │ 无需外部知识   │
   └─────┬────┘     └──────┬───────┘
         │                 │
         ▼                 │
   ┌──────────┐            │
   │ 3. Action │            │
   │ (检索)    │            │
   └─────┬────┘            │
         │                 │
         ▼                 │
   ┌──────────┐            │
   │ 4. Observe│            │
   │ (结果观察)│            │
   └─────┬────┘            │
         │                 │
         └──────┬──────────┘
                ▼
   ┌──────────────────────┐
   │ 5. Generate /        │
   │    Continue Loop     │
   └──────────────────────┘
```

### 核心决策点

Agentic RAG 在每个循环中需要回答三个关键问题：

| 决策问题 | 决策结果 | 触发条件 |
|----------|---------|---------|
| **是否需要检索？** | need_search / no_search | 知识截止日期后的问题、领域专业问题、事实性查询 |
| **检索什么？** | 检索查询词、检索范围、检索策略 | 子问题分解结果、当前推理状态 |
| **检索结果是否足够？** | enough / not_enough → 继续检索 | 结果置信度低、信息不完整、存在矛盾 |

---

## 架构设计

### 1. Tool-based RAG Agent

这是最直接的实现方式：将检索器封装为 Agent 可调用的工具函数。

```
                    ┌─────────────────────────┐
                    │       Orchestrator       │
                    │    Agent 主循环 (LLM)     │
                    └────┬──────┬──────┬──────┘
                         │      │      │
              ┌──────────┘      │      └──────────┐
              ▼                 ▼                 ▼
       ┌────────────┐   ┌────────────┐   ┌────────────┐
       │ Retriever  │   │ Calculator │   │ Web Search │
       │ (知识库)    │   │ (计算器)    │   │ (网络搜索)  │
       └────────────┘   └────────────┘   └────────────┘
```

**关键设计**：
- 检索器作为标准工具函数，与 Calculator、Web Search 等同级
- Agent 可多次调用检索器，每次调用不同的查询词
- 检索结果存入 Agent 的上下文记忆

### 2. Memory-augmented RAG

在 Tool-based 基础上，增加记忆系统来存储和引用历史检索结果。

```
Agent 循环
   │
   ├── 短期记忆（当前会话上下文）
   │     └── 本轮检索结果列表
   │
   ├── 长期记忆（跨会话知识）
   │     └── 持久化向量存储
   │
   └── 工作记忆（当前推理状态）
         └── 子问题队列、未完成检索
```

**检索决策与记忆的关系**：
- Agent 在决定是否需要检索时，优先检查短期记忆中是否已有相关信息
- 重复查询避免重复检索——通过语义缓存实现
- 长期记忆中的高频知识可以预加载到 Agent 上下文中

### 3. Multi-step RAG

复杂查询需要多步推理 + 多步检索的协同：

```
用户查询: "2024年诺奖得主在量子计算领域的主要贡献是什么？"

Step 1: 思考 + 检索
   Thought: "我需要先找到2024年诺奖得主是谁"
   Action: retrieve("2024 Nobel Prize winners")
   Observation: [Physics: John Hopfield, Geoffrey Hinton; ...]

Step 2: 推理 + 聚焦检索
   Thought: "诺奖得主在量子计算领域...让我聚焦Hopfield和Hinton的工作"
   Action: retrieve("John Hopfield contribution quantum computing")
   Action: retrieve("Geoffrey Hinton contribution quantum computing")
   Observation: [Hopfield网络... Hinton对玻尔兹曼机的贡献...]

Step 3: 综合生成
   Thought: "现在我有足够的信息，可以综合回答了"
   Generate: "2024年诺贝尔物理学奖得主..."
```

---

## 实现示例

以下 Python 伪代码展示了一个简化的 Agentic RAG 主循环实现：

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class AgentState:
    """Agent 运行时状态"""
    messages: list = field(default_factory=list)      # 对话历史
    context_docs: list = field(default_factory=list)   # 已检索文档
    max_turns: int = 5                                 # 最大循环轮次
    current_turn: int = 0
    done: bool = False

class AgenticRAG:
    """
    Agentic RAG 核心实现
    将检索作为 Agent 工具，自主决策检索时机
    """
    
    def __init__(self, llm, retriever, tools=None):
        self.llm = llm
        self.retriever = retriever
        self.tools = tools or []
        # 注册检索工具
        self.tools.append({
            "name": "retrieve",
            "description": "从知识库中检索与查询相关的文档",
            "function": self.retrieve_docs
        })
    
    def retrieve_docs(self, query: str, top_k: int = 3) -> list[str]:
        """检索工具的函数实现"""
        docs = self.retriever.retrieve(query, top_k=top_k)
        return docs
    
    def think(self, state: AgentState) -> dict:
        """
        思考阶段：LLM 判断是否需要检索，或可以直接生成答案
        
        返回决策结果：
        - needs_search: True/False
        - search_query: 检索查询词（如果 need_search）
        - reasoning: 推理过程
        """
        system_prompt = """你是一个智能 RAG Agent。你的任务是判断是否需要检索外部知识。
        
        需要检索的情况：
        - 用户询问事实性信息（日期、人物、事件、数据）
        - 用户询问你的知识截止日期之后的信息
        - 用户询问领域专业知识，而你的训练数据可能不充分
        
        不需要检索的情况：
        - 用户进行日常对话
        - 用户询问你已知的常识
        - 用户已经提供了足够上下文
        
        请以 JSON 格式返回决策结果：
        {"needs_search": bool, "search_query": str, "reasoning": str}
        """
        
        response = self.llm.chat([
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": "当前状态:\n" + 
             f"对话历史: {state.messages}\n" +
             f"已检索文档: {state.context_docs}\n" +
             "请判断下一步行动"}
        ])
        
        decision = self.parse_decision(response)
        return decision

    def run(self, query: str) -> str:
        """
        Agentic RAG 主循环
        
        流程：
        1. 接收用户查询
        2. Agent 思考 → 判断是否需要检索
        3. 如果需要检索 → 执行检索 → 注入上下文 → 回到 2
        4. 如果不需要 → 直接从上下文生成最终答案
        5. 返回最终答案
        """
        state = AgentState()
        state.messages.append({"role": "user", "content": query})
        
        while not state.done and state.current_turn < state.max_turns:
            state.current_turn += 1
            
            # 1. 思考阶段：Agent 判断下一步
            decision = self.think(state)
            
            # 2. 决策执行
            if decision["needs_search"]:
                # 2a. 执行检索
                search_query = decision.get("search_query", query)
                docs = self.retrieve_docs(search_query)
                
                # 2b. 将检索结果注入上下文
                state.context_docs.extend(docs)
                state.messages.append({
                    "role": "system", 
                    "content": f"检索结果（查询: {search_query}）:\n" + 
                               "\n---\n".join(docs)
                })
                
                # 2c. 记录检索行为
                state.messages.append({
                    "role": "assistant",
                    "content": f"我检索了关于「{search_query}」的信息，获得 {len(docs)} 篇相关文档。"
                })
            else:
                # 3. 直接生成最终答案
                if decision.get("direct_answer"):
                    state.done = True
                    final_answer = self.llm.chat([
                        {"role": "system", "content": self._build_generation_prompt(state)},
                        *state.messages
                    ])
                    return final_answer
        
        # 达到最大轮次后的兜底生成
        final_answer = self.llm.chat([
            {"role": "system", "content": "基于已有信息生成回答"},
            *state.messages
        ])
        return final_answer
    
    def _build_generation_prompt(self, state: AgentState) -> str:
        """构建生成阶段的系统提示"""
        return f"""你是一个知识丰富的 AI 助手。基于以下检索到的文档回答问题。
        
        如果文档中包含答案，请引用具体内容。
        如果文档不包含答案，请坦诚说明。
        
        已检索 {len(state.context_docs)} 篇文档。
        当前轮次: {state.current_turn}/{state.max_turns}
        """
    
    def parse_decision(self, response: str) -> dict:
        """解析 LLM 的决策响应"""
        import json
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return {"needs_search": False, "direct_answer": True}

# 使用示例
if __name__ == "__main__":
    llm = YourLLM()          # 你的 LLM 实例
    retriever = YourRetriever()  # 你的检索器实例
    
    agent = AgenticRAG(llm=llm, retriever=retriever)
    
    # 不需要检索的查询
    answer = agent.run("你好，今天过得怎么样？")
    # → 直接回答，不需要检索
    
    # 需要检索的查询
    answer = agent.run("2025年诺贝尔化学奖得主是谁？")
    # → 思考 → 检索 → 回答
```

---

## 关键挑战

### 1. 检索时机判断准确率

Agent 需要在"过度检索"和"检索不足"之间找到平衡。这是一个二分类问题，但实际边界非常模糊：

```
检索不足                           过度检索
◄────────────── Balance ──────────────►

用户: "什么是量子纠缠？"
  → Agent 可能:
      - 判断为常识 → 直接回答 ✓
      - 判断为专业术语 → 检索 ✓
      - 判断错误 → 检索无关文档 ✗
      
用户: "2025年最新的量子纠错进展"
  → Agent 必须检索 → 这是知识截止后信息
```

**优化方向**：
- 使用分类模型（BERT 微调）预处理查询，辅助 Agent 判断检索必要性
- 设置"默认检索"的阈值策略：对不确定性高的查询，优先检索再判断
- 通过 RLHF 优化 LLM 的检索决策能力

### 2. 多轮检索的 Token 消耗

Agentic RAG 的 Token 消耗是 Naive RAG 的 2-3 倍以上：

| 阶段 | Token 消耗 |
|------|-----------|
| 用户查询 | 基础 |
| Think 阶段（每次循环） | 200-500 tokens |
| Retrieve 结果注入 | 500-2000 tokens |
| 多轮对话累积 | 逐轮增长 |
| 最终生成 | 200-1000 tokens |

**优化策略**：
- 使用更小的模型做检索决策（如 GPT-4o-mini judge）
- 对检索结果进行摘要压缩后再注入上下文
- 设置最大循环轮次上限（通常 3-5 轮）
- 使用滑动窗口管理已检索的上下文

### 3. 工具调用延迟

Agentic RAG 的每次循环包含多次 LLM 调用 + 检索调用，端到端延迟显著：

```
Agentic RAG 延迟分解:
┌──────────────────────┬──────────────┐
│ 阶段                 │ 延迟         │
├──────────────────────┼──────────────┤
│ LLM Think (1次)      │ 300-800ms   │
│ LLM 检索决策 (1次)   │ 200-500ms   │
│ 向量检索 (1次)       │ 50-200ms    │
│ 重排序 (1次)         │ 100-300ms   │
│ LLM 生成 (1次)       │ 500-2000ms  │
├──────────────────────┼──────────────┤
│ 单轮合计              │ 1150-3800ms │
│ 3轮合计              │ 3.5-11.4s   │
└──────────────────────┴──────────────┘
```

**优化策略**：
- 并行执行多个检索查询（多路检索）
- 使用流式生成减少首 token 延迟
- 预缓存常见检索结果
- 对简单查询直接使用非 Agentic 模式

---

## 能力边界

### 不适合的场景

1. **高频实时场景**：需要亚秒级响应的系统（如客服机器人），Agent 的推理开销不可接受
2. **确定性要求高的场景**：Agent 的决策有随机性，同样的查询可能触发不同的检索行为
3. **成本极度敏感的场景**：Agentic RAG 的 API 调用成本是传统 RAG 的 2-5 倍
4. **Agent 缺乏领域知识**：如果 Agent 本身对领域不熟悉，无法做出正确的检索决策

### 级联错误风险

Agentic RAG 的核心风险在于决策错误的级联放大：

```
查询 → 错误判断"不需要检索" → 生成错误答案    ✗
查询 → 错误判断"需要检索"   → 生成正确但浪费   △
查询 → 检索到错误文档       → 基于错误信息推理  ✗
查询 → 错误判断"结果足够"   → 错过关键信息     ✗
```

**缓解策略**：
- 在关键决策点设置人类审核（Human-in-the-loop）
- 对低置信度答案设置"不回答"策略
- 引入 Self-RAG 的反思机制作为补救

---

## 主流框架支持

### LangGraph Agent

LangGraph 提供了构建 Agentic RAG 最成熟的框架：

```python
from langgraph.graph import StateGraph
from langchain_core.messages import HumanMessage, AIMessage
from typing import TypedDict, Literal

class GraphState(TypedDict):
    question: str
    generation: str
    documents: list[str]
    needs_search: bool

def decide_to_generate(state: GraphState) -> Literal["generate", "retrieve"]:
    """Agent 决策节点"""
    decision = llm.predict_need_search(state["question"])
    return "retrieve" if decision else "generate"

# LangGraph 构建 Agentic RAG
graph = StateGraph(GraphState)
graph.add_node("agent_decision", decide_to_generate)
graph.add_node("retrieve", retrieve_docs)
graph.add_node("generate", generate_answer)

graph.set_conditional_entry_point(
    "agent_decision",
    {"retrieve": "retrieve", "generate": "generate"}
)
graph.add_edge("retrieve", "generate")
```

### CrewAI

在 CrewAI 中，Agentic RAG 通过知识源（Knowledge Source）集成：

```python
from crewai import Agent, Task, Crew, Process
from crewai.knowledge.source.string_knowledge_source import StringKnowledgeSource

# 创建 RAG Agent
rag_agent = Agent(
    role="Research Analyst",
    goal="自主决定是否需要检索知识库来回答问题",
    backstory="你是一个高级研究分析师，知道何时需要查找信息",
    knowledge_sources=[knowledge_source],
    verbose=True
)
```

### AutoGen

AutoGen 中可以用 AssistantAgent 结合 RetrieveUserProxyAgent 实现：

```python
import autogen
from autogen.agentchat.contrib.retrieve_assistant_agent import RetrieveAssistantAgent
from autogen.agentchat.contrib.retrieve_user_proxy_agent import RetrieveUserProxyAgent

assistant = RetrieveAssistantAgent(
    name="assistant",
    system_message="你需要时自动检索知识库",
    llm_config=llm_config,
)

ragproxyagent = RetrieveUserProxyAgent(
    name="ragproxyagent",
    retrieve_config={
        "task": "qa",
        "docs_path": "./knowledge_base",
        "vector_db": "chroma"
    }
)
```

---

## 2025-2026 前沿进展

### SPARKLE 结构化检索策略

SPARKLE（Structured Planning and Adaptive Retrieval with Knowledge-guided Learning）是 2025 年提出的前沿框架，它让 Agent 不仅仅做"检不检索"的二元决策，而是：

1. **结构化检索规划**：Agent 生成多步检索计划（类似思维链），每一步指定检索目标、方法和评估标准
2. **自适应的策略调整**：根据中间检索结果动态调整后续策略，类似于自我纠错的搜索
3. **知识引导的决策**：利用知识图谱中的结构信息来指导检索决策，比纯语义搜索更精确

### Agent-Orchestrated Adaptive RAG

多智能体协同成为趋势——不同 Agent 分管不同类型的检索决策：

```
                      ┌─────────────────────┐
                      │    Orchestrator      │
                      │    编排 Agent        │
                      └────────┬────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
   │ 简单查询 Agent  │  │ 复杂推理 Agent  │  │ 验证 Agent     │
   │ 直接检索+生成   │  │ 多步检索+推理   │  │ 答案质量验证    │
   │ 低延迟          │  │ 高精度          │  │ 需要时重做      │
   └────────────────┘  └────────────────┘  └────────────────┘
```

### 关键趋势总结

| 趋势 | 描述 | 影响 |
|------|------|------|
| 决策模型专门化 | 用小型专用模型替代大模型做检索决策 | 降低成本，提高决策速度 |
| 检索策略结构化 | 从二元决策到多步检索计划规划 | 处理复杂查询能力飞跃 |
| 多 Agent 协同 | 不同 Agent 分工负责检索的不同环节 | 系统鲁棒性和可维护性提升 |
| 决策可解释性 | Agent 的检索决策需要可追溯、可审计 | 生产环境部署的必要条件 |
| 端到端训练 | 检索决策与生成联合优化 | 整体效果超越两阶段分离的系统 |
