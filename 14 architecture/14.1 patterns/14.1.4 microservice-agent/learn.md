# 微服务 Agent — Microservice Agent Architecture

> **核心思想**：将 Agent 系统拆分为多个独立部署、独立扩展、独立运维的微服务，每个微服务封装特定的 Agent 能力或领域知识，通过轻量级通信协议 (HTTP/gRPC/消息队列) 协作。这是微服务架构理念在 Agent 领域的直接应用。

---

## 1. 基本原理

### 1.1 什么是微服务 Agent

微服务 Agent 将单体 Agent 的各个组件（推理引擎、工具执行器、记忆系统、规划器、验证器等）拆分为独立的服务，每个服务有自己独立的生命周期、扩展策略和故障域。

```
单体 Agent:                         微服务 Agent:
┌──────────────────┐                ┌──────────┐  ┌──────────┐
│  推理 (LLM)       │                │ 推理服务   │  │ 规划服务   │
│  规划            │                │ (独立扩缩) │  │ (独立扩缩) │
│  工具执行         │                └──────────┘  └──────────┘
│  记忆             │                      │              │
│  工具注册         │                ┌──────────┐  ┌──────────┐
│  错误处理         │                │ 工具服务   │  │ 记忆服务   │
│  ... (All-in-1)  │                │ (独立扩缩) │  │ (独立扩缩) │
└──────────────────┘                └──────────┘  └──────────┘
                                          │              │
                                    ┌──────────┐  ┌──────────┐
                                    │ 验证服务   │  │ 编排服务   │
                                    │ (独立扩缩) │  │ (API网关) │
                                    └──────────┘  └──────────┘
```

### 1.2 背景与演进

**之前怎么做**：随着 Agent 系统的复杂度增长，单体 Agent 逐渐暴露出各种问题：

```
单体 Agent 的扩展困境:

场景: 一个提供编程帮助的 Agent
功能: 代码理解 → 搜索 → 推理 → 代码生成 → 代码执行 → 测试

问题 1: 扩展不均衡
  • 代码执行非常消耗 CPU (沙箱隔离)
  • LLM 推理消耗 GPU/API 配额
  • 代码搜索需要内存/索引资源
  在单体中，要扩展代码执行能力，必须一起扩展推理能力 → 资源浪费

问题 2: 部署耦合
  • 更新代码执行器的沙箱版本 → 必须重新部署整个 Agent
  • 升级 LLM 模型 → 必须重建整个进程
  • 修改记忆存储逻辑 → 影响所有功能

问题 3: 故障放大
  • 代码执行器 OOM → 整个 Agent 进程崩溃
  • 记忆系统超时 → 阻塞推理循环
  • LLM API 限流 → 工具调用也无法进行
```

**核心矛盾**：单体 Agent 将异构的计算资源需求（CPU 密集型、I/O 密集型、GPU 密集型、内存密集型）捆绑在同一个进程中，导致资源利用率低、部署耦合度高、故障域过大。在传统 Web 系统中，这个问题通过微服务架构解决；同样，Agent 系统也需要微服务化。

### 1.3 微服务 Agent 如何解决

通过将不同资源需求的组件拆分为独立服务，每个服务可以：
- 独立选择最合适的硬件 (CPU/GPU/内存)
- 独立扩缩 (高频服务多副本，低频服务少副本)
- 独立部署和升级 (不影响其他服务)
- 独立故障隔离 (一个服务崩溃不影响其他服务)

```
微服务 Agent 的资源分配:

服务              计算需求        典型副本数         硬件
──────────────────────────────────────────────────────────
推理服务          GPU/API 延迟     3-10               GPU/高带宽
工具执行器        CPU 密集型       5-20               CPU 优化
代码执行器        CPU+内存         10-50              高内存
记忆检索          I/O + 内存       3-5                内存优化
向量搜索          I/O              3-5                SSD/NVMe
规划服务          API 延迟         2-3                标准
验证服务          API 延迟         2-3                标准
API 网关          网络 I/O         2-5 (+自动扩缩)    标准
```

---

## 2. 架构详解

### 2.1 服务拆分

微服务 Agent 的服务拆分不可能是一次完成的，需要根据实际需求逐步演进。下面是典型的拆分策略：

```
第一阶段: 2 个服务
  [Agent Service]  ←→  [Tool Execution Service]
   推理 + 规划 + 记忆     工具执行 + 代码沙箱

第二阶段: 4 个服务
  [Orchestrator] → [Reasoning] → [Tool Exec] → [Memory]
     编排            推理         工具执行       记忆管理

第三阶段: 7+ 个服务 (全拆分)
  用户请求 → [API Gateway]
              │
              ├──→ [Orchestrator Service]    — 任务编排与状态管理
              ├──→ [Reasoning Service]       — LLM 推理 (多模型)
              ├──→ [Planner Service]         — 任务分解与规划
              ├──→ [Tool Executor Service]   — 工具调用与结果处理
              ├──→ [Code Executor Service]   — 安全代码执行
              ├──→ [Memory Service]          — 短期/长期记忆
              ├──→ [Knowledge Service]       — RAG/知识检索
              └──→ [Validation Service]      — 输出验证与质量控制
```

### 2.2 完整代码实现

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Any, Dict, List, Optional
import httpx
import asyncio
import json
import uuid
from enum import Enum

# ============ 共享 Schema ============

class AgentTask(BaseModel):
    task_id: str = ""
    task_type: str = ""
    input: Any
    context: Dict[str, Any] = {}
    config: Dict[str, Any] = {}
    max_steps: int = 10

class TaskResult(BaseModel):
    task_id: str
    status: str = "pending"  # "completed" | "failed" | "in_progress"
    output: Any = None
    error: Optional[str] = None
    metrics: Dict[str, Any] = {}

# ============ 1. API 网关服务 ============

class APIGateway:
    """API 网关 — 统一入口"""
    def __init__(self):
        self.app = FastAPI(title="Agent API Gateway")
        self.setup_routes()
    
    def setup_routes(self):
        @self.app.post("/agent/process")
        async def process_request(request: Dict[str, Any]):
            # 1. 认证与授权
            # 2. 速率限制
            # 3. 请求验证
            # 4. 路由到编排服务
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    "http://orchestrator:8001/task",
                    json=request,
                    timeout=300.0
                )
                return response.json()
        
        @self.app.get("/agent/task/{task_id}")
        async def get_task_status(task_id: str):
            async with httpx.AsyncClient() as client:
                response = await client.get(
                    f"http://orchestrator:8001/task/{task_id}"
                )
                return response.json()
        
        @self.app.post("/agent/stream/{task_id}")
        async def stream_task(task_id: str):
            """流式结果接口"""
            # 使用 SSE 推送
            pass

# ============ 2. 编排服务 (Orchestrator) ============

class OrchestratorService:
    """编排服务 — Agent 主循环管理"""
    def __init__(self):
        self.app = FastAPI(title="Orchestrator Service")
        self.task_store: Dict[str, AgentTask] = {}
        self.result_store: Dict[str, TaskResult] = {}
        self.setup_routes()
    
    def setup_routes(self):
        @self.app.post("/task")
        async def create_task(task: AgentTask):
            task.task_id = task.task_id or str(uuid.uuid4())
            self.task_store[task.task_id] = task
            self.result_store[task.task_id] = TaskResult(
                task_id=task.task_id, status="in_progress"
            )
            
            # 异步启动处理
            asyncio.create_task(self.process_task(task))
            
            return {"task_id": task.task_id, "status": "in_progress"}
        
        @self.app.get("/task/{task_id}")
        async def get_task(task_id: str):
            result = self.result_store.get(task_id)
            if not result:
                raise HTTPException(404, "Task not found")
            return result
    
    async def process_task(self, task: AgentTask):
        """Agent 主循环 — 跨服务编排"""
        context = {
            "input": task.input,
            "history": [],
            "intermediate_results": []
        }
        
        step = 0
        while step < task.max_steps:
            # Step 1: 调用规划服务
            async with httpx.AsyncClient() as client:
                plan_response = await client.post(
                    "http://planner:8002/plan",
                    json={"context": context, "task": task.dict()},
                    timeout=30.0
                )
                plan = plan_response.json()
            
            if plan.get("action") == "complete":
                break
            
            # Step 2: 调用推理服务
            async with httpx.AsyncClient() as client:
                thought_response = await client.post(
                    "http://reasoning:8003/reason",
                    json={
                        "context": context,
                        "plan": plan,
                        "task_type": task.task_type
                    },
                    timeout=60.0
                )
                thought = thought_response.json()
            
            # Step 3: 执行工具 (如果需要)
            if thought.get("needs_tool"):
                async with httpx.AsyncClient() as client:
                    tool_response = await client.post(
                        "http://tool-executor:8004/execute",
                        json={
                            "tool": thought["tool_name"],
                            "params": thought["tool_params"],
                            "context": context
                        },
                        timeout=30.0
                    )
                    tool_result = tool_response.json()
                
                # Step 4: 验证结果
                async with httpx.AsyncClient() as client:
                    validation = await client.post(
                        "http://validator:8005/validate",
                        json={
                            "step_output": thought,
                            "tool_result": tool_result,
                            "task": task.dict()
                        },
                        timeout=30.0
                    )
                    validation_result = validation.json()
                
                context["intermediate_results"].append({
                    "step": step,
                    "thought": thought,
                    "tool_result": tool_result,
                    "validation": validation_result
                })
                
                if not validation_result.get("valid", True):
                    # 验证失败，尝试修正
                    continue
            
            step += 1
        
        # 完成：生成最终输出
        async with httpx.AsyncClient() as client:
            final_response = await client.post(
                "http://reasoning:8003/finalize",
                json={"context": context, "task": task.dict()},
                timeout=60.0
            )
            final_result = final_response.json()
        
        self.result_store[task.task_id] = TaskResult(
            task_id=task.task_id,
            status="completed",
            output=final_result,
            metrics={"steps": step, "latency": 0}
        )

# ============ 3. 推理服务 (Reasoning Service) ============

class ReasoningService:
    """推理服务 — 封装 LLM 调用"""
    def __init__(self, api_key: str, model: str = "gpt-4"):
        self.app = FastAPI(title="Reasoning Service")
        self.api_key = api_key
        self.model = model
        self.setup_routes()
    
    def setup_routes(self):
        @self.app.post("/reason")
        async def reason(request: Dict):
            context = request["context"]
            plan = request["plan"]
            
            # 组装 Prompt
            system_prompt = self.build_system_prompt(request.get("task_type", ""))
            messages = [
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": json.dumps({
                    "context": context,
                    "current_plan": plan
                })}
            ]
            
            # 调用 LLM (可替换为不同后端)
            response = await self.call_llm(messages)
            
            return {
                "thought": response["content"],
                "needs_tool": self.detect_tool_need(response["content"]),
                "tool_name": self.extract_tool_name(response["content"]),
                "tool_params": self.extract_tool_params(response["content"]),
                "confidence": response.get("confidence", 0.8)
            }
    
    async def call_llm(self, messages: List) -> Dict:
        """LLM 调用 (可切换模型/Provider)"""
        # 这里可以是 OpenAI / Anthropic / 本地模型
        # 实际实现需要集成具体的 LLM SDK
        return {"content": "...", "confidence": 0.9}

# ============ 4. 工具执行服务 ============

class ToolExecutorService:
    """工具执行服务 — 封装所有工具调用"""
    def __init__(self):
        self.app = FastAPI(title="Tool Executor Service")
        self.tools: Dict[str, Any] = self.register_tools()
        self.setup_routes()
    
    def register_tools(self) -> Dict:
        return {
            "search": self.search_web,
            "read_file": self.read_file,
            "execute_python": self.execute_python,
            "query_database": self.query_database,
            "call_api": self.call_external_api,
        }
    
    def setup_routes(self):
        @self.app.post("/execute")
        async def execute_tool(request: Dict):
            tool_name = request["tool"]
            params = request["params"]
            
            if tool_name not in self.tools:
                return {"error": f"Unknown tool: {tool_name}"}
            
            try:
                result = await self.tools[tool_name](**params)
                return {"success": True, "result": result}
            except Exception as e:
                return {"success": False, "error": str(e)}
        
        @self.app.get("/tools")
        async def list_tools():
            """返回可用工具列表 (用于工具发现)"""
            return {
                "tools": [
                    {
                        "name": name,
                        "schema": self.get_tool_schema(name)
                    }
                    for name in self.tools
                ]
            }
    
    async def execute_python(self, code: str, timeout: int = 30):
        """安全的 Python 代码执行 (沙箱隔离)"""
        # 在隔离的 Docker 容器或 pyodide 沙箱中执行
        pass

# ============ 5. 记忆服务 ============

class MemoryService:
    """记忆服务 — 统一管理 Agent 记忆"""
    def __init__(self):
        self.app = FastAPI(title="Memory Service")
        self.setup_routes()
    
    def setup_routes(self):
        @self.app.post("/memory/store")
        async def store_memory(request: Dict):
            # 存储到短期记忆 (Redis) + 长期记忆 (Vector DB)
            pass
        
        @self.app.post("/memory/retrieve")
        async def retrieve_memory(request: Dict):
            # 语义检索 + 最近检索 + 重要性检索融合
            pass
        
        @self.app.post("/memory/consolidate")
        async def consolidate_memory():
            # 记忆整合: 短期 → 长期
            pass
        
        @self.app.delete("/memory/clear/{session_id}")
        async def clear_session(session_id: str):
            # 清除会话记忆
            pass

# ============ 6. 验证服务 ============

class ValidationService:
    """验证服务 — 输出质量和安全验证"""
    def __init__(self):
        self.app = FastAPI(title="Validation Service")
        self.setup_routes()
    
    def setup_routes(self):
        @self.app.post("/validate")
        async def validate_step(request: Dict):
            """验证 Agent 的每一步输出"""
            step_output = request["step_output"]
            tool_result = request.get("tool_result")
            
            validations = {
                "format_valid": self.check_format(step_output),
                "tool_result_valid": self.check_tool_result(tool_result) if tool_result else True,
                "no_hallucination": self.check_hallucination(step_output),
                "safe_content": self.check_safety(step_output),
                "task_relevant": self.check_relevance(step_output, request["task"])
            }
            
            return {
                "valid": all(validations.values()),
                "details": validations
            }
```

### 2.3 端到端调用流程

```
用户请求流程:

时间  |  客户端          API 网关          编排服务         推理服务         工具服务         记忆服务
     |    │               │                │               │               │               │
     |    │───HTTP POST───►│                │               │               │               │
     |    │               │──创建任务──────►│               │               │               │
     |    │               │  返回 task_id  │               │               │               │
     |    │◄─────task_id──│                │               │               │               │
     |    │               │                │──规划───────►│               │               │
     |    │               │                │◄────plan──────│               │               │
     |    │               │                │               │               │               │
     |    │               │                │──推理───────►│               │               │
     |    │               │                │◄────thought───│               │               │
     |    │               │                │               │               │               │
     |    │               │                │──执行工具───────────────────►│               │
     |    │               │                │◄────result────────────────────│               │
     |    │               │                │               │               │               │
     |    │               │                │──查询记忆──────────────────────────────────►│
     |    │               │                │◄────memories─────────────────────────────────│
     |    │               │                │               │               │               │
  (重复推理-行动循环 2-5 次)               │               │               │               │
     |    │               │                │               │               │               │
     |    │               │                │──最终生成────►│               │               │
     |    │               │                │◄────response──│               │               │
     |    │               │                │               │               │               │
     |    │◄───最终响应────│◄──完成────────│               │               │               │
     ▼    ▼               ▼                ▼               ▼               ▼               ▼
```

### 2.4 服务间通信

```python
# 通信方式选择:
#
# ┌────────────────────────────────────────────────────────────────┐
# │ 场景                    推荐协议       原因                   │
# ├────────────────────────────────────────────────────────────────┤
# │ Agent 推理 (LLM 调用)     HTTP/2        请求-响应模式          │
# │ 工具执行结果返回          gRPC          高性能，双向流          │
# │ 事件通知 (任务状态变更)   消息队列       解耦，广播              │
# │ 流式 Token 输出           gRPC Server-  低延迟，增量推送        │
# │                          Side Streaming                       │
# │ 服务状态 / 健康检查        HTTP/2        轻量，频繁              │
# │ 大规模状态同步            Kafka         持久化，顺序保证         │
# └────────────────────────────────────────────────────────────────┘

# gRPC 服务定义示例 (proto/agent_service.proto):
"""
service ReasoningService {
    rpc Reason (ReasonRequest) returns (ReasonResponse);
    rpc StreamReason (ReasonRequest) returns (stream TokenChunk);
    rpc Finalize (FinalizeRequest) returns (FinalizeResponse);
}

service ToolService {
    rpc Execute (ToolRequest) returns (ToolResponse);
    rpc StreamExecute (ToolRequest) returns (stream ToolChunk);
    rpc ListTools (Empty) returns (ToolList);
}

service MemoryService {
    rpc Store (StoreRequest) returns (StoreResponse);
    rpc Retrieve (RetrieveRequest) returns (RetrieveResponse);
    rpc Consolidate (Empty) returns (ConsolidateResponse);
}
"""

# 消息队列事件示例:
"""
事件: agent.task.completed
{
    "task_id": "task-123",
    "agent_id": "agent-a",
    "result_summary": "...",
    "metrics": {"steps": 5, "tokens": 2500, "latency_ms": 8500},
    "timestamp": "..."
}

事件: agent.tool.timeout
{
    "task_id": "task-123",
    "tool": "code_executor",
    "timeout_seconds": 30,
    "retry_count": 2
}

事件: agent.memory.consolidated
{
    "agent_id": "agent-a",
    "episodes_consolidated": 15,
    "new_long_term_memories": 3
}
"""
```

---

## 3. 高级设计

### 3.1 无状态推理服务

推理服务设计为无状态是微服务 Agent 扩展的关键：

```python
class StatelessReasoningService:
    """无状态推理服务 — 可在多副本间负载均衡"""
    
    # 所有状态由编排器维护，推理服务只负责 LLM 调用
    # 编排器在每次推理请求中附加上下文和历史
    
    @app.post("/reason")
    async def reason(request: Dict):
        """
        请求必须包含完整上下文:
        {
            "session_id": "...",
            "user_input": "...",
            "conversation_history": [...],  # 由编排器管理
            "current_thought": "...",
            "tool_results": [...],
            "agent_state": {...},           # 由编排器管理
            "config": {"model": "gpt-4", "temperature": 0.7}
        }
        """
        # 推理服务无状态地处理请求
        # 可以安全地负载均衡到任何副本
        result = await llm_call(request)
        return result
    
    # 优势:
    # 1. 轻松水平扩展 (kubectl scale deploy reasoning --replicas=10)
    # 2. 滚动更新无影响
    # 3. 故障实例自动被负载均衡移除
```

### 3.2 断路器与服务降级

```python
class ServiceCircuitBreaker:
    """跨微服务的断路器"""
    
    def __init__(self, service_name: str, failure_threshold: int = 5):
        self.service_name = service_name
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.state = "closed"  # closed → open → half-open
        self.last_failure_time = 0
        self.recovery_timeout = 30  # 30 秒后尝试恢复
    
    async def call_with_circuit_breaker(self, func, fallback=None):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "half-open"  # 尝试恢复
            else:
                return await fallback() if fallback else None
        
        try:
            result = await func()
            if self.state == "half-open":
                self.state = "closed"  # 恢复成功
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "open"
            return await fallback() if fallback else None

# 降级策略示例:
#
# 推理服务不可用 → 使用缓存的结果 / 使用小模型替代
# 工具服务不可用 → 告知用户工具暂时不可用 / 排队等待
# 记忆服务不可用 → 退化到仅使用短期上下文
# 验证服务不可用 → 跳过验证步骤 (接受一定风险)
```

### 3.3 服务的独立扩展策略

```
推理服务 (GPU/API 密集):
  │ 扩缩依据: Pending LLM Request Queue Depth
  │ 指标: LLM API 延迟 P95 > 5s → scale up
  │       LLM API 延迟 P95 < 1s → scale down (如果有闲置)
  │ 约束: API 速率限制 (RPM/TPM)
  │ 策略: 每个副本保持 3-5 个并发请求
  ├── HPA 配置:
  │   apiVersion: autoscaling/v2
  │   kind: HorizontalPodAutoscaler
  │   spec:
  │     metrics:
  │     - type: External
  │       external:
  │         metric: pending_llm_requests
  │         target:
  │           type: AverageValue
  │           averageValue: 5

工具执行服务 (CPU 密集):
  │ 扩缩依据: CPU 使用率 + 请求队列深度
  │ HPA: CPU > 70% → scale up
  │       请求队列 > 100 → scale up
  │ 策略: 代码执行器独占 CPU，不共享

记忆服务 (I/O 密集):
  │ 扩缩依据: QPS + 连接数
  │ HPA: QPS > 1000/副本 → scale up
  │       连接数 > 200/副本 → scale up
  │ 注意: 向量数据库通常有独立扩缩 (不受 Agent 服务控制)

API 网关 (网络 I/O):
  │ 扩缩依据: 并发连接数
  │ HPA: 连接数 > 1000/副本 → scale up
  │ 通常保持 2-3 个副本 + 自动扩缩
```

---

## 4. 适用场景与能力边界

### 4.1 最佳适用场景

| 场景 | 原因 | 案例 |
|------|------|------|
| **生产级系统** | 需要独立扩缩、滚动更新 | SaaS Agent 平台 |
| **异构计算需求** | 不同组件需要不同硬件 | 代码沙箱 + LLM 推理 |
| **大型团队** | 不同团队负责不同服务 | 企业级 Agent 产品 |
| **高可用要求** | 故障隔离、多 AZ 部署 | 金融/医疗领域 |
| **多模型支持** | 不同任务用不同模型 | 简单任务用小模型 |

### 4.2 不适用场景

| 场景 | 原因 | 替代方案 |
|------|------|----------|
| **MVP/原型** | 微服务架构带来高开发成本 | 单体 Agent |
| **小团队 (1-3人)** | 运维负担 > 收益 | 单体/管道架构 |
| **简单 Agent** | 拆分反而增加延迟 | 单体 Agent |
| **边缘部署** | 资源受限 | 单体 Agent |
| **低延迟场景** | 服务间网络调用增加延迟 | 单体/管道架构 |

### 4.3 能力边界

```
微服务 Agent 的核心挑战:

┌──────────────────────────────────────────────────────────────┐
│ 挑战                      影响                               │
├──────────────────────────────────────────────────────────────┤
│ 网络延迟                  服务间调用的累积延迟               │
│ 数据一致性                跨服务的数据一致性问题             │
│ 分布式追踪                需要端到端的链路追踪                │
│ 部署复杂度                多服务部署和版本管理                │
│ 测试难度                  集成测试和端到端测试复杂             │
│ Token 重复消耗             跨服务传递上下文导致 Token 重复     │
│ 调试困难                  分布式系统中的问题定位              │
│ 运维成本                  更多服务 = 更多监控/告警/日志       │
└──────────────────────────────────────────────────────────────┘

微服务 Agent 的 Token 效率问题:
  在单体 Agent 中，LLM 调用的上下文在所有步骤间共享
  在微服务 Agent 中，每次服务间调用都需要传递上下文
  → Token 消耗可能增加 20-40%

  优化方案:
  • 共享上下文服务 (Shared Context Service) 
  • 引用式上下文传递 (传递 ID 而不是内容)
  • 上下文摘要 (每个服务只传递需要的信息)
```

---

## 5. 工程优化方向

| 方向 | 方法 | 效果 |
|------|------|------|
| **上下文压缩** | 引用式传递 + 摘要 | Token 消耗减少 30-50% |
| **结果缓存** | 服务级缓存 + 分布式缓存 | 延迟降低 40-60% |
| **连接池** | 服务间 gRPC 连接复用 | 减少连接建立开销 |
| **批量推理** | 合并多个推理请求 | LLM API 成本降低 30% |
| **异步处理** | 工具执行异步回调 | 资源利用率提升 |
| **服务网格** | 使用 Istio/Linkerd 管理 | 可观测性 + 流量管理 |
| **单元化部署** | 按租户或任务类型隔离部署 | 故障隔离 + 性能隔离 |
