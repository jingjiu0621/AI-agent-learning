# 10.2.3 LangSmith Integration — LangSmith 深度集成

## 简单介绍

LangSmith 是 LangChain 推出的 LLM 应用可观测性平台，是目前 Agent 链路追踪领域最成熟的产品之一。它为 LangChain/LangGraph 应用提供了开箱即用的追踪、监控、评估和调试能力。即使不直接使用 LangChain 框架，LangSmith 的 tracing API 也可以集成到手写的 Agent 中。

## 基本原理

LangSmith 的核心是 **Run Tree**——一种类似于 OpenTelemetry Span 的追踪结构。每次 LLM 调用、工具调用、链执行都被记录为一个 Run，Run 可以嵌套形成追踪树。

```python
# LangSmith 追踪示例（使用 LangChain）
from langsmith import traceable
from langchain_openai import ChatOpenAI

@traceable(name="my_agent", run_type="chain")
def my_agent(query: str):
    llm = ChatOpenAI(model="gpt-4o")
    # 这个 LLM 调用会自动被记录为 Run Tree 的子 Run
    result = llm.invoke(query)
    return result

# 追踪输出：
# Run Tree:
#   my_agent (chain) [trace_id: ...]
#     ├── ChatOpenAI (llm) [tokens: 150/32, model: gpt-4o, latency: 800ms]
#     └── result: "..."

# 手动创建追踪（不依赖 LangChain）
from langsmith import Client, run_trees

client = Client()
with run_trees.create_run(
    name="thought_step",
    run_type="chain",
    inputs={"thought": "用户需要查天气..."},
    project_name="agent-observability"
) as run:
    # 模拟工具调用
    with run_trees.create_run(
        name="get_weather",
        run_type="tool",
        inputs={"location": "Beijing"},
        parent_run_id=run.id,
    ) as tool_run:
        result = {"temp": 22, "condition": "sunny"}
        tool_run.end(outputs=result)

    run.end(outputs={"response": "北京晴，22°C"})
```

LangSmith 追踪的 Agent 特有信息：

| 追踪维度 | LangSmith 实现 |
|---------|---------------|
| LLM 调用 | 自动记录 model、tokens、temperature、latency |
| 工具调用 | 自动记录 tool_name、input、output、latency |
| 检索操作 | 自动记录 retriever、doc_ids、scores |
| 多步循环 | 通过 Run Tree 的嵌套关系表达 ReAct 步骤 |
| Token 成本 | 自动基于模型定价计算 |
| 错误追踪 | 自动捕获异常并关联到对应的 Run |

## 背景

LangSmith 于 2023 年由 LangChain 团队推出，填补了 LLM 应用可观测性工具的空白。在此之前，开发者只能使用通用 APM 工具（Datadog、New Relic）或自行实现追踪。LangSmith 的独特优势在于它**理解 LLM 应用的语义**——不是泛化的"请求延迟"，而是具体的"Prompt Token 消耗"、"工具调用错误"、"检索相关性"。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 通用 APM 工具 | 用 Datadog/NewRelic 追踪 Agent | 只能看到 HTTP 级别的调用，看不到 LLM 调用和 Token 消耗 |
| 自行实现追踪 | 基于 OpenTelemetry 自建追踪系统 | 需要大量工程投入，缺乏 LLM 特定的语义 |
| 无追踪 | 只用日志记录 Agent 行为 | 无法从整体视角分析延迟和 Token 消耗 |
| LangChain callback | 用 LangChain 的 CallbackHandler 手动记录 | 功能有限，缺乏可视化界面和评估功能 |

## 核心矛盾

| 矛盾维度 | 通用 APM | LangSmith | 自建 OTel |
|---------|---------|-----------|----------|
| LLM 语义理解 | ❌ | ✅ | 需自定义 |
| 开箱即用 | ✅ | ✅（LangChain 项目） | ❌ |
| 数据主权 | ✅（自建） | ❌（SaaS） | ✅ |
| 成本 | 按数据量 | 按 seat + trace | 自建成本 |
| 定制性 | 高 | 中 | 最高 |

**核心矛盾**：LangSmith 提供了最好的开箱即用体验和 LLM 语义理解，但它是 SaaS 服务，数据需要发送到 LangChain 的服务器。对于有数据驻留合规要求的企业，可能需要自建或使用开源替代（Langfuse、MLflow）。

## 当前主流优化方向

1. **LangSmith + OpenTelemetry 双通道**——将 LangSmith 用于开发调试阶段的详细追踪，OTel 用于生产监控的聚合指标，两者互补
2. **自定义 Run 类型**——不在局限于预设的 llm/chain/tool/retriever 类型，支持自定义的评估、规划、记忆等 Agent 专有操作
3. **评估集成**——LangSmith 的 Datasets + Evaluation 功能使追踪数据可以直接用于回归测试和模型评估
4. **反馈循环**——将追踪中识别的错误模式自动转化为测试用例，建立"发现问题 → 修复 → 验证"的闭环
5. **LangSmith Hub + Prompt 管理**——将追踪中表现好的 Prompt 版本上传到 Hub，实现 Prompt 版本管理与追踪的联动

## 实现的最大挑战

1. **LangChain 依赖**——如果不使用 LangChain 框架，LangSmith 的手动集成需要额外的适配工作，不如开箱即用流畅
2. **数据量控制**——LangSmith 的免费额度有限，生产环境中大量 Agent 调用产生的追踪数据可能导致高昂的成本
3. **数据隐私**——追踪数据会发送到 LangChain 的服务器，包含完整的 Prompt 和 Response（可能含敏感信息），需要配置数据脱敏
4. **Trace 可视化复杂度**——Agent 的 ReAct 循环在 Run Tree 中表现为深度嵌套，超过一定步数后可视化变得混乱

## 能力边界

**能做什么：**
- 开箱即用的 LLM 调用追踪（model, tokens, latency）
- 多步 Agent 执行的完整 Trace 可视化
- 基于追踪数据的评估和回归测试
- Prompt 版本管理与 A/B 测试
- 团队协作和分享

**不能做什么：**
- 不能替代全面的监控系统——LangSmith 聚焦追踪和评估，实时告警能力有限
- 不能在完全离线环境使用——虽然开源版本正在完善，但仍然需要服务端组件
- 不能追踪非 LangChain/非 HTTP 调用——需要额外的适配工作
- 不能保证数据 100% 不丢失——SaaS 服务有可用性 SLA 但不如本地部署可靠

## 与其他的区别

| 对比项 | LangSmith | Langfuse | OpenTelemetry | MLflow |
|--------|-----------|---------|---------------|--------|
| 开源 | 部分 | ✅ | ✅ | ✅ |
| 部署方式 | SaaS | 自建/SaaS | 自建 | 自建 |
| LLM 专有 | ✅ | ✅ | 需扩展 | 部分 |
| LangChain 集成 | 原生 | 好 | 中等 | 中等 |
| 评估功能 | ✅ | ✅ | ❌ | 部分 |
| 数据主权 | ❌ | ✅ | ✅ | ✅ |

## 最终工程优化

1. **选择性追踪**——使用 LangSmith 的 `@traceable` 装饰器的 `metadata` 参数添加自定义标签，实现按标签过滤追踪（如只追踪某些租户或某些类型的 Agent）
2. **本地 + LangSmith 双写**——开发环境同时写入本地文件系统和 LangSmith，生产环境只写入本地（或只写入 LangSmith 的采样数据），灵活切换
3. **自动脱敏中间件**——在 LangSmith Client 的上层封装一层自动脱敏层，自动替换追踪数据中的敏感字段（API Key、密码、PII）
4. **追踪数据导出流水线**——定期将 LangSmith 中的追踪数据导出到自有数据仓库（S3/BigQuery），用于长期分析和报表
5. **与 CI/CD 集成**——将 LangSmith 的评估结果作为 CI 门禁：如果新版本的 Prompt 或 Agent 配置导致评估分数下降，阻止部署
