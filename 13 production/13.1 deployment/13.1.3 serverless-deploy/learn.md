# 13.1.3 serverless-deploy — Serverless 部署

**Serverless 是低流量、事件驱动型 Agent 最具成本效益的部署方式。** 它将基础设施管理完全抽象化，让开发者只关心 Agent 逻辑本身。对于调用频率不固定、需要快速原型验证或事件触发的 Agent 场景，Serverless 比维护一个常驻的 K8s 集群经济得多。但 Serverless 的冷启动、执行时长限制和有状态处理的短板，在 Agent 场景下会被显著放大。

## 背景与问题

### 之前是怎么做的？
维护一台 24/7 运行的服务器（或 K8s 集群）来托管 Agent 服务。即使 Agent 平均每小时只被调用几次，服务器资源也一直在消耗。

### 这样做的问题
- **资源浪费**：低流量时段服务器闲置，仍要付费
- **运维负担**：OS 补丁、安全更新、扩容规划都需要投入
- **弹性滞后**：流量突增时需要分钟级反应时间
- **成本不可控**：固定费用难以与真实使用量匹配

### Serverless 的承诺
- 按实际调用次数付费，零流量时零费用
- 自动从 0 到 N 的弹性伸缩
- 运维全托管，开发者只关心代码

## Serverless 与 Agent 的核心矛盾

```
Serverless 设计假设                    Agent 实际需求
─────────────────                     ────────────────
函数应轻量快速 (≤ 100ms 执行)          Agent 单次推理可能需要 10-30s
函数无状态 (每次调用全新环境)           Agent 天然有状态（对话、记忆）
函数执行时间受限 (通常 ≤ 15min)        复杂 Agent 任务可能长达数小时
函数间通信困难                         Agent 需要多工具协作、多步推理
流式输出受限 (API Gateway 超时)        Agent 需要 SSE/WebSocket 推送
冷启动 ~100ms-1s                      Agent 依赖大（框架、模型），冷启动 > 5s
```

这个矛盾并不意味着"Serverless 不适合 Agent"，而是说：**不是所有类型的 Agent 都适合 Serverless**。事件驱动的、短周期的、无状态的 Agent 子任务在 Serverless 上表现出色。

## 适合 Serverless 的 Agent 场景

```
适合 Serverless                               不适合 Serverless
─────────────────                              ─────────────────
• 单次问答 Agent（查询→响应）                    • 长时间对话 Agent（多轮交互）
• 事件触发 Agent（新邮件→自动回复）               • 连续监控 Agent（实时观察）
• 批处理 Agent（定期执行数据分析）                 • 延迟敏感型 Agent（毫秒级响应需求）
• 工具调用-Agent（单次工具调用）                   • 复杂推理 Agent（10+ 步推理链）
• Webhook Agent（外部系统触发）                   • 流式输出 Agent（持续 SSE 推送）
• 预处理/后处理 Agent（RAG 管道的某个环节）         • 有状态 Agent（长期记忆、会话保持）
```

## 主流 Serverless 平台对比

| 维度 | AWS Lambda | Vercel Functions | Cloudflare Workers | Google Cloud Functions |
|------|------------|------------------|-------------------|----------------------|
| 运行时 | Python/Node/Java/Go | Node/Python/Go | JS/WASM/Python | Node/Python/Go/Java |
| 超时上限 | 15min (900s) | 60s (Hobby), 300s (Pro), 900s (Enterprise) | 30s (Unbound: 15min) | 60min (Gen2) |
| 内存上限 | 10GB | 1GB (Pro) | 128MB (可增) | 32GB |
| 冷启动 | ~100-500ms | ~100-300ms | ~0ms (isolates) | ~200-500ms |
| 流式响应 | Lambda URL 支持 | Edge Functions 支持 | 原生 TransformStream | 有限支持 |
| Python 包大小 | 250MB (压缩) | 50MB | 1MB (Workers), 100MB (Python) | 无明确限制 |
| 并发 | 1000 (区域) | 1000 (Pro) | 无限 | 3000 |
| 成本 | 按请求+时长 | 按请求+时长 | 按请求 (极低) | 按请求+时长 |

**核心结论**：
- **AWS Lambda**：最成熟的 Agent Serverless 平台，但 Python 依赖管理复杂
- **Cloudflare Workers**：冷启动几乎为零，适合全球低延迟，但 Python 支持较弱
- **Vercel Functions**：前端+Agent 简单整合的最佳选择，但超时限制严格
- **GCP Cloud Functions**：长超时 + 大内存适合复杂 Agent，但生态较小

## AWS Lambda Agent 实战

### 基础 Lambda Handler

```python
# agent_handler.py
import json
import os
import logging
from typing import Any

import boto3

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# 全局初始化（热启动时复用）
import openai
from langchain_openai import ChatOpenAI

# 在全局作用域初始化 LLM 客户端（热启动复用）
def _init_llm() -> ChatOpenAI:
    return ChatOpenAI(
        model=os.environ.get("OPENAI_MODEL", "gpt-4o-mini"),
        temperature=0.1,
        max_tokens=4096,
    )

# 热启动时全局变量会缓存
llm = None

def lambda_handler(event: dict, context: Any) -> dict:
    global llm
    if llm is None:
        llm = _init_llm()  # 只在冷启动时执行

    try:
        # 解析请求
        body = json.loads(event.get("body", "{}"))
        user_message = body.get("message", "")

        if not user_message:
            return respond(400, {"error": "message is required"})

        # 简单的 Agent 调用（单轮对话）
        response = llm.invoke(user_message)

        logger.info(f"Agent response generated for input length {len(user_message)}")

        return respond(200, {
            "reply": response.content,
            "usage": {
                "prompt_tokens": response.response_metadata.get("token_usage", {}).get("prompt_tokens", 0),
                "completion_tokens": response.response_metadata.get("token_usage", {}).get("completion_tokens", 0),
            }
        })

    except Exception as e:
        logger.error(f"Agent error: {str(e)}", exc_info=True)
        return respond(500, {"error": "Internal agent error"})

def respond(status_code: int, body: dict) -> dict:
    return {
        "statusCode": status_code,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*",
        },
        "body": json.dumps(body, ensure_ascii=False),
    }
```

### 部署配置 (SAM)

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 300           # 5 分钟——Agent 可能需要多步推理
    MemorySize: 1024       # 1GB——LangChain + NumPy 需要
    Runtime: python3.12
    Environment:
      Variables:
        OPENAI_MODEL: gpt-4o-mini
        LOG_LEVEL: INFO

Resources:
  AgentFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: agent/
      Handler: agent_handler.lambda_handler
      Policies:
        - AWSLambdaBasicExecutionRole
        - SecretsManagerReadWrite  # 从 Secrets Manager 读取 API Key
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /agent
            Method: POST
      Environment:
        Variables:
          OPENAI_API_KEY: !Sub '{{resolve:secretsmanager:openai-key:SecretString}}'

  # 可选的：Step Functions 工作流编排复杂 Agent
  ComplexAgentWorkflow:
    Type: AWS::Serverless::StateMachine
    Properties:
      DefinitionUri: workflow.asl.json
      DefinitionSubstitutions:
        AgentFunctionArn: !GetAtt AgentFunction.Arn
      Policies:
        - LambdaInvokePolicy:
            FunctionName: !Ref AgentFunction
```

### Step Functions 编排（长时间运行 Agent）

对于需要多步推理的复杂 Agent，不能在一个 Lambda 内跑完（超时限制），需要拆成 Step Functions 工作流：

```json
{
  "Comment": "Multi-step Agent Workflow",
  "StartAt": "Analyze Query",
  "States": {
    "Analyze Query": {
      "Type": "Task",
      "Resource": "${AgentFunctionArn}",
      "ResultPath": "$.analysis",
      "Next": "Determine Tools"
    },
    "Determine Tools": {
      "Type": "Task",
      "Resource": "${AgentFunctionArn}",
      "ResultPath": "$.toolPlan",
      "Next": "Execute Tools"
    },
    "Execute Tools": {
      "Type": "Map",
      "ItemsPath": "$.toolPlan.tools",
      "Iterator": {
        "StartAt": "Call Tool",
        "States": {
          "Call Tool": {
            "Type": "Task",
            "Resource": "arn:aws:states:::lambda:invoke",
            "Parameters": {
              "FunctionName": "${AgentFunctionArn}",
              "Payload": {
                "action": "execute_tool",
                "tool_name.$": "$.name",
                "params.$": "$.params"
              }
            },
            "End": true
          }
        }
      },
      "ResultPath": "$.toolResults",
      "Next": "Synthesize Response"
    },
    "Synthesize Response": {
      "Type": "Task",
      "Resource": "${AgentFunctionArn}",
      "ResultPath": "$.finalResponse",
      "End": true
    }
  }
}
```

### Lambda Layer 策略

Agent 的依赖体积大（LangChain ~50MB, Pandas ~30MB），必须用 Layer：

```yaml
# 构建 Layer
# build_layer.sh
pip install -t layer/python -r requirements.txt \
    --platform manylinux2014_x86_64 \
    --only-binary=:all: \
    --no-cache-dir
cd layer && zip -r ../agent_layer.zip python/

# 部署后 Lambda 函数 + Layer = 总大小不超过 250MB (压缩后)
```

关键优化：
- 使用 `--only-binary=:all:` 避免在 Lambda 上编译
- 删除 `.pyc` 和 `__pycache__`：`find layer -name '*.pyc' -delete`
- 只安装生产依赖：`pip install --no-deps` 减少传递依赖

## Vercel 部署 Agent

### Serverless Function

```typescript
// api/agent.ts
import type { VercelRequest, VercelResponse } from '@vercel/node';
import OpenAI from 'openai';

// 热启动全局缓存
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

export default async function handler(req: VercelRequest, res: VercelResponse) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  const { message, conversationId } = req.body;

  try {
    // 从外部存储恢复对话上下文
    const context = await getConversationContext(conversationId);

    const response = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [
        { role: 'system', content: 'You are a helpful assistant.' },
        ...context.messages,
        { role: 'user', content: message },
      ],
    });

    await saveConversationContext(conversationId, message, response);

    return res.status(200).json({
      reply: response.choices[0].message.content,
      usage: response.usage,
    });
  } catch (error) {
    console.error('Agent error:', error);
    return res.status(500).json({ error: 'Agent processing failed' });
  }
}

// 使用 Upstash Redis (Vercel 推荐的外部状态存储)
async function getConversationContext(id: string) {
  const { Redis } = await import('@upstash/redis');
  const redis = new Redis({
    url: process.env.UPSTASH_REDIS_URL!,
    token: process.env.UPSTASH_REDIS_TOKEN!,
  });
  const data = await redis.get(`conversation:${id}`);
  return data || { messages: [] };
}
```

### Vercel Edge Function（流式响应）

```typescript
// api/agent-stream.ts
export const runtime = 'edge';

export default async function handler(req: Request): Promise<Response> {
  const { message } = await req.json();

  const stream = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
    },
    body: JSON.stringify({
      model: 'gpt-4o-mini',
      messages: [
        { role: 'system', content: 'You are a helpful assistant.' },
        { role: 'user', content: message },
      ],
      stream: true,
    }),
  });

  // 将 OpenAI 的 SSE 流透传给客户端
  return new Response(stream.body, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

## Cloudflare Workers Agent

### Worker 处理函数

```typescript
// src/agent.ts
export interface Env {
  OPENAI_API_KEY: string;
  AI: any; // Cloudflare AI Binding
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    const { message } = await request.json();

    // 方案 A：使用 Cloudflare AI（自带 Workers 优化）
    const response = await env.AI.run('@cf/meta/llama-3-8b-instruct', {
      messages: [
        { role: 'system', content: 'You are a helpful AI assistant.' },
        { role: 'user', content: message },
      ],
      stream: true,
    });

    // 方案 B：使用 OpenAI（需要 Worker 访问外部 API）
    // const response = await fetch('https://api.openai.com/v1/chat/completions', {
    //   method: 'POST',
    //   headers: {
    //     'Content-Type': 'application/json',
    //     'Authorization': `Bearer ${env.OPENAI_API_KEY}`,
    //   },
    //   body: JSON.stringify({ ... }),
    // });

    // Workers 的 TransformStream 非常适合流式推送
    return new Response(response, {
      headers: { 'Content-Type': 'text/event-stream' },
    });
  },
};
```

### wrangler.toml

```toml
name = "agent-worker"
main = "src/agent.ts"
compatibility_date = "2024-12-01"

[ai]
binding = "AI"

[env.production]
vars = { LOG_LEVEL = "info" }

[env.production.secrets]
OPENAI_API_KEY = "sk-..."
```

## 外部状态管理（Serverless Agent 的关键）

Serverless 要求无状态，而 Agent 需要状态。解决方案都是——**把状态移到外部**：

```python
# state_manager.py
import json
import os
from typing import Optional, List, Dict
from datetime import datetime, timedelta

import boto3
import redis.asyncio as redis
from pydantic import BaseModel

class SessionState(BaseModel):
    session_id: str
    messages: List[Dict]
    created_at: datetime
    expires_at: datetime
    metadata: Dict = {}

class StateManager:
    """跨 Serverless 调用共享 Agent 状态"""

    def __init__(self):
        # 根据环境选择后端
        self.backend = os.environ.get("STATE_BACKEND", "redis")

        if self.backend == "redis":
            self.redis = redis.from_url(os.environ["REDIS_URL"])
        elif self.backend == "dynamodb":
            self.dynamodb = boto3.resource("dynamodb")
            self.table = self.dynamodb.Table(os.environ["STATE_TABLE"])

    async def get_state(self, session_id: str) -> Optional[SessionState]:
        if self.backend == "redis":
            data = await self.redis.get(f"session:{session_id}")
            return SessionState(**json.loads(data)) if data else None

        elif self.backend == "dynamodb":
            resp = self.table.get_item(Key={"session_id": session_id})
            item = resp.get("Item")
            return SessionState(**item) if item else None

    async def save_state(self, state: SessionState):
        ttl = int((state.expires_at - datetime.utcnow()).total_seconds())

        if self.backend == "redis":
            await self.redis.setex(
                f"session:{state.session_id}",
                ttl,
                state.model_dump_json()
            )

        elif self.backend == "dynamodb":
            self.table.put_item(
                Item={**state.model_dump(), "ttl": ttl}
            )

    async def append_message(self, session_id: str, role: str, content: str):
        """追加消息到会话历史"""
        state = await self.get_state(session_id) or SessionState(
            session_id=session_id,
            messages=[],
            created_at=datetime.utcnow(),
            expires_at=datetime.utcnow() + timedelta(hours=1),
        )
        state.messages.append({"role": role, "content": content})
        await self.save_state(state)
```

## Serverless Agent 成本分析

### 成本模型对比

以日均 10,000 次 Agent 调用为例，每次调用平均耗时 5s：

| 部署方式 | 月成本估算 | 说明 |
|----------|-----------|------|
| Docker (单机) | ~$50-100 | 1 台云服务器 |
| K8s (3 节点) | ~$200-500 | 3 台云服务器 + 管理费 |
| Lambda (1GB, 5s) | ~$150-250 | 请求费 + 计算时长 |
| Vercel Pro | ~$20/月 + 超量费 | 包含 1k 小时/月 |
| Cloudflare Workers | ~$5-50 | 极低单价，高免费额度 |

**结论**：低流量时 Serverless 最便宜，高频调用时容器化/K8s 反而更经济。

### 成本优化技巧

```python
# 1. 控制上下文大小——减少 Token 意味着减少处理时间
MAX_CONTEXT_TOKENS = 4000  # 不是每轮对话都需要完整历史

# 2. 使用推理缓存 (Lambda 的全局变量)
_cached_llm = None

def get_llm():
    global _cached_llm
    if _cached_llm is None:  # 只在冷启动初始化一次
        _cached_llm = ChatOpenAI(model="gpt-4o-mini")
    return _cached_llm

# 3. 设置最大执行步数——防止 Agent 循环
def run_agent(query: str, max_steps: int = 5):
    for step in range(max_steps):  # 硬限制
        # agent loop...
        pass
```

## Serverless Agent 最佳实践

### 1. 冷启动优化
```python
# Lambda 冷启动优化清单
# ✅ 全局范围初始化客户端（热启动复用）
# ✅ 使用轻量化依赖：如果只需要 LLM 调用，用 openai 而不是 langchain
# ✅ 压缩依赖大小：Lambda Layer + 调试 treeshake
# ✅ 使用 Provisioned Concurrency（预留并发）
# ❌ 不要在全局做重量级初始化
```

### 2. 超时处理
```python
# 为 Agent 添加内部超时，防止 Lambda 超时导致丢数据
import signal

class TimeoutError(Exception):
    pass

def timeout_handler(signum, frame):
    raise TimeoutError("Agent execution timed out")

def run_with_timeout(func, timeout=240):  # Lambda 最多 900s
    signal.signal(signal.SIGALRM, timeout_handler)
    signal.alarm(timeout)
    try:
        result = func()
    finally:
        signal.alarm(0)
    return result
```

### 3. 事件驱动集成

```python
# Agent + SQS/SNS 事件驱动模式
# ┌────────┐   ┌──────┐   ┌────────┐   ┌────────┐
# │  S3    │──►│ SQS  │──►│ Lambda │──►│  Agent  │
# │ (新文档)│   │      │   │ (触发) │   │ (处理)  │
# └────────┘   └──────┘   └────────┘   └────────┘
#                                           │
#                                    ┌──────▼──────┐
#                                    │    SNS      │
#                                    │ (通知结果)  │
#                                    └─────────────┘

def lambda_handler(event, context):
    for record in event.get("Records", []):
        # SQS 消息
        if "body" in record:
            message = json.loads(record["body"])
            # 异步处理，无需立即返回
            process_agent_task(message)

    return {"status": "accepted"}
```

## 能力边界与结果

**Serverless 部署适合：**
- ✅ 低频率或调用间隔长的 Agent（如日报自动生成）
- ✅ 事件驱动型 Agent（文档上传后自动处理）
- ✅ Agent 管道的某个阶段（如 RAG 检索预处理）
- ✅ 快速原型验证和 MVP
- ✅ 对冷启动不敏感的场景

**Serverless 部署不适合：**
- ❌ 需要流式实时响应的对话 Agent
- ❌ 长时间运行的复杂推理 Agent（> 15min）
- ❌ 需要 GPU 推理的 Agent
- ❌ 高频低延迟 Agent（每次 <100ms 响应要求）
- ❌ 大型依赖的 Agent（> 250MB 压缩包）

**何时升级到 K8s：**
- 日均调用量稳定超过 5 万次
- Agent 推理步骤日益复杂，Lambda 超时限制成为瓶颈
- 需要 GPU/TPU 进行本地推理
- 需要精细的资源控制和质量保证
