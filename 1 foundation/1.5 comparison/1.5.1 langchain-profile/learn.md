# LangChain / LangGraph：灵活图编排

## 简单介绍

LangChain（2022）是最早广泛采用的 LLM 应用开发框架，LangGraph（2024）是其图编排扩展。LangChain 提供了一整套工具链（Prompt 管理、模型调用、工具集成、记忆、RAG），LangGraph 在此基础上增加了基于**有状态图（StateGraph）** 的 Agent 编排能力。

## 核心设计

```
LangChain 核心抽象：
- Model I/O: ChatModel, PromptTemplate, OutputParser
- Retrieval: DocumentLoader, TextSplitter, VectorStore, Retriever
- Tools: Tool, tool decorator, Toolkits
- Memory: ConversationBufferMemory, SummaryMemory
  
LangGraph 新增：
- StateGraph: 节点(Node) + 边(Edge) 构建的执行图
- Checkpointer: 状态持久化与中断恢复
- 循环支持: 图中可以有环（ReAct 循环的图表示）
```

## 核心优势

| 优势 | 说明 |
|------|------|
| 生态丰富 | 最多的集成（数百种工具、向量库、模型） |
| 灵活性高 | LCEL 表达式可组合各种组件 |
| LangGraph 强大 | 图编排支持复杂控制流 |
| 社区活跃 | 最活跃的 LLM 框架社区 |
| 可观测性 | LangSmith 深度集成 |

## 主要不足

| 不足 | 说明 |
|------|------|
| 学习曲线陡 | 抽象层次多，概念多 |
| 过度封装 | 简单任务也被复杂化 |
| 版本不稳定 | API 变化频繁，升级成本高 |
| Debug 困难 | 多层抽象隐藏了实际执行的细节 |
| Prompt 控制有限 | 框架的 Prompt 模板可能覆盖用户的精细控制 |

## 适用场景

- **复杂 Agent 工作流**：需要精细控制执行流程（条件分支、循环、并行）
- **需要复杂状态管理**：多步任务需要持久化和恢复
- **RAG 系统**：丰富的文档加载和检索工具
- **生产级部署**：LangSmith 可观测性、LangServe 部署

## 不适用场景

- **简单 API 封装**：直接调用 SDK 更轻量
- **需要完全控制**：框架抽象限制了 Prompt 和执行的精细控制
- **最小依赖项目**：LangChain 依赖链重，增加部署体积
