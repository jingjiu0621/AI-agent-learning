# 13.2.1 concurrency-model — 并发模型

**Agent 的主循环本质上是 I/O 密集型的——绝大多数时间在等待 LLM API 响应、向量数据库查询和工具执行结果。** 这意味着 Agent 并发模型的选择标准与传统 CPU 密集型服务完全不同：核心指标不是"每秒能算多少"，而是"等待时能同时处理多少会话"。在 Agent 场景下，异步并发（asyncio）是默认选择，但需要特别注意避免阻塞事件循环。

## 背景与问题

### Agent 并发的工作负载特征

理解 Agent 并发的关键，是先理解一个 Agent 会话的典型时间分配：

```
一次 Agent 请求的完整生命周期 (~5s 总耗时):

时间线:
0ms    ┌─────────────────────────────────────────────
       │ Prompt 组装 & 上下文加载 (10ms, CPU)
500ms  │──────────────────────────────────────────────
       │ ◄─── LLM API 请求 (1500ms, I/O 等待) ────►
2000ms │──────────────────────────────────────────────
       │ 工具调用 & 结果解析 (200ms, CPU + I/O)
2200ms │──────────────────────────────────────────────
       │ ◄─── 第二轮 LLM API (1200ms, I/O 等待) ──►
3400ms │──────────────────────────────────────────────
       │ 响应组装 & 流式推送 (100ms, CPU + I/O)
3500ms │──────────────────────────────────────────────
       │ ◄─── 记忆存储 (400ms, I/O 等待) ────────►
3900ms │──────────────────────────────────────────────
       │ 日志 & 跟踪 (100ms, I/O)

CPU 使用率: ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██░░░░░░░░░░░░░░░░░░██
                        ≈ 8% CPU 利用率
```

**Agent 单线程处理单会话时，CPU 利用率只有 5-15%**，其余时间在等待 I/O。这意味着单线程异步可以轻松处理 50-200 个并发 Agent 会话——大多数会话都在等待各自的 LLM API 响应。

### 之前是怎么做的？

1. **同步单线程（逐请求处理）**：每个 Agent 请求串行执行，一个请求阻塞整个进程
2. **多进程/多线程**：每个 Agent 请求分配一个线程或进程
3. **使用同步框架（Flask/Gunicorn）**：Worker 进程处理请求，每个 Worker 一次只处理一个

### 这样做的问题

```
并发模型            Agent 场景的问题
─────────           ─────────────────
同步单线程          一次只能处理一个 Agent 请求，吞吐量极低
                   100 个用户需要 100 秒才能全部响应

多线程 (Thread)     GIL 让 Python 线程无法真正并行
                   线程切换开销 + 全局锁争用
                   每个线程独立占用内存 (栈空间 ~8MB)
                   需要线程安全锁，增加复杂度

多进程 (Process)    进程启动慢 (~500ms)，不适合突发流量
                   Agent 请求本身不占 CPU，多进程浪费资源
                   进程间共享状态困难 (需要 Redis/DB)
                   内存占用高 (每个进程加载全部依赖)

Gunicorn Worker     每个 Worker 一次处理一个 Agent 请求
(同步模式)          5 个 Worker = 同时只能处理 5 个 Agent
                   对 Agent 调用链长、I/O 密集的负载极其低效
```

## 核心并发模型对比

```python
# 对比三种并发模型处理 100 个 Agent 请求
# 每个请求平均耗时 5s (其中 I/O 等待 4.5s, CPU 0.5s)

MODEL_SYNC = """
同步模型 (每个请求一个线程/进程):
总耗时 = 100 × 5s = 500s
需要 100 个线程/进程同时运行才能 5s 内完成
瓶颈: 大多数时间花在等待 I/O，但线程/进程被白白占用
Agent 场景下: 极不推荐
"""

MODEL_ASYNC = """
异步模型 (asyncio):
总耗时 = max(所有请求的 I/O 等待) + 100 × CPU 时间
       = 4.5s + 100 × 0.5s = 54.5s
       ≈ 5s (如果 I/O 等待完全重叠)
实际上: 事件循环高效调度，约 5-10s 内完成 100 个请求
Agent 场景下: ✅ 默认推荐
"""

MODEL_HYBRID = """
混合模型 (async + 进程池):
asyncio 处理 I/O 密集型 Agent 编排
进程池处理 CPU 密集型工具执行 / 本地模型推理
Agent 场景下: 当 Agent 有本地推理或计算密集工具时推荐
"""
```

### 模型选择决策树

```
你的 Agent 主要瓶颈是?
│
├── I/O 等待 (LLM API、DB 查询、外部 API)
│   └── asyncio ← 95% 的 Agent 选这个
│
├── 本地推理 (运行本地 SLM/Embedding 模型)
│   └── asyncio + 进程池 (run_in_executor)
│
├── CPU 密集 (大量数据处理、代码执行)
│   ├── 轻量 → asyncio + 进程池
│   └── 重量 → 独立 Worker 进程 + 消息队列
│
└── 混合 (以上都有)
    └── 分层: asyncio 编排 + 进程池计算 + 独立进程推理
```

## asyncio Agent 架构

### 基础架构：一个事件循环处理数千 Agent

```python
# async_agent_server.py — 异步 Agent 服务器
import asyncio
import time
from typing import AsyncGenerator

class AsyncAgentWorker:
    """异步 Agent 工作器 - 单协程处理一个会话"""
    
    def __init__(self, session_id: str, llm_client, tools, memory):
        self.session_id = session_id
        self.llm = llm_client      # 异步 LLM 客户端
        self.tools = tools          # 工具注册表
        self.memory = memory        # 异步记忆客户端
        self.conversation = []      # 对话历史
        self.max_steps = 10
        
    async def run(self, user_input: str) -> AsyncGenerator[str, None]:
        """异步 Agent 主循环，支持流式输出"""
        self.conversation.append({"role": "user", "content": user_input})
        
        for step in range(self.max_steps):
            # 1. 构建 Prompt (CPU 操作，但很快)
            prompt = self._build_prompt()
            
            # 2. 调用 LLM API (I/O 密集 - 协程让出控制权)
            response = await self.llm.chat(prompt)
            
            # 3. 解析响应，检查是否需要调用工具
            tool_call = self._parse_tool_call(response)
            if not tool_call:
                # 最终回复
                self.conversation.append({
                    "role": "assistant", "content": response
                })
                yield response
                return
            
            # 4. 执行工具调用 (可能是 I/O + CPU)
            tool_result = await self._execute_tool(tool_call)
            
            # 5. 存储中间结果
            self.conversation.extend([
                {"role": "assistant", "content": response},
                {"role": "tool", "name": tool_call["name"], 
                 "content": tool_result}
            ])
            
            # 6. 异步持久化检查点
            asyncio.create_task(self._save_checkpoint(step))
    
    async def _execute_tool(self, tool_call: dict) -> str:
        """执行工具调用 - 可能是异步或同步"""
        tool_name = tool_call["name"]
        tool_args = tool_call["arguments"]
        
        if tool_name not in self.tools:
            return f"Error: unknown tool {tool_name}"
        
        tool_fn = self.tools[tool_name]
        
        if asyncio.iscoroutinefunction(tool_fn):
            # 异步工具：直接在事件循环中执行
            return await tool_fn(**tool_args)
        else:
            # 同步工具：在线程池中执行，避免阻塞事件循环
            loop = asyncio.get_event_loop()
            return await loop.run_in_executor(
                None, tool_fn, **tool_args
            )


class AsyncAgentServer:
    """异步 Agent 服务器 - 管理所有会话"""
    
    def __init__(self, llm_client, tools, memory):
        self.llm = llm_client
        self.tools = tools
        self.memory = memory
        self.sessions: dict[str, AsyncAgentWorker] = {}
        
    async def handle_request(self, session_id: str, 
                             user_input: str) -> AsyncGenerator[str, None]:
        """处理用户请求，自动创建或恢复会话"""
        if session_id not in self.sessions:
            self.sessions[session_id] = AsyncAgentWorker(
                session_id, self.llm, self.tools, self.memory
            )
        
        worker = self.sessions[session_id]
        async for chunk in worker.run(user_input):
            yield chunk
    
    async def run_server(self):
        """启动异步服务器"""
        # 示例：模拟并发请求
        tasks = []
        for i in range(100):  # 100 个并发 Agent 会话
            task = asyncio.create_task(
                self.handle_request(f"session_{i}", f"用户问题 {i}")
            )
            tasks.append(task)
        
        # 所有会话并发执行
        results = await asyncio.gather(*tasks, return_exceptions=True)
        print(f"完成 {len(results)} 个会话")


# 使用示例
async def main():
    server = AsyncAgentServer(llm_client, tools, memory)
    await server.run_server()
    # 100 个 Agent 并发运行，每个 5s 耗时
    # 总耗时 ≈ 5s (不是 500s)，因为 I/O 等待完全重叠

asyncio.run(main())
```

### 关键模式：asyncio.create_task 的两种用法

```python
# 模式 1: 并发执行 (多个 Agent 同时运行)
tasks = [asyncio.create_task(agent.run(input)) for agent in agents]
results = await asyncio.gather(*tasks)

# 模式 2: 后台任务 (不阻塞当前流程)
async def handle_request(self, ...):
    result = await self.llm.chat(prompt)
    # 异步写日志，不影响响应返回
    asyncio.create_task(self._log_async(result))
    asyncio.create_task(self._update_memory_async(result))
    return result
    # ⚠ 注意: create_task 创建的任务如果抛出异常会被静默吞掉
    # 需要全局异常处理器或 TaskGroup (Python 3.11+)
```

## 高级并发模式

### 模式一：Semaphore 控制并发度

不限制并发时，所有 Agent 同时调用 LLM API，可能导致 API 限流。

```python
class RateLimitedAgent:
    """使用信号量控制 LLM API 并发度的 Agent"""
    
    def __init__(self, max_concurrent: int = 10):
        # 限制同时最多 10 个 LLM API 请求
        self.semaphore = asyncio.Semaphore(max_concurrent)
        
    async def call_llm(self, prompt: str) -> str:
        async with self.semaphore:
            return await self.llm_client.chat(prompt)
    
    async def run_agent(self, input_text: str):
        # Agent 主循环中受控的 LLM 调用
        response = await self.call_llm(input_text)
        # ... 后续处理
```

### 模式二：优先级队列

高优先级请求（付费用户、实时交互）应该优先获得 LLM API 调用权。

```python
import asyncio
from dataclasses import dataclass
from enum import IntEnum

class Priority(IntEnum):
    CRITICAL = 0  # 付费用户 / 实时交互
    HIGH = 1       # 普通用户
    NORMAL = 2     # 后台任务
    LOW = 3        # 批量处理

@dataclass(order=True)
class AgentTask:
    priority: Priority
    timestamp: float
    session_id: str = ""
    input_text: str = ""

class PriorityAgentQueue:
    """优先级 Agent 队列"""
    
    def __init__(self, max_concurrent: int = 10):
        self.queue = asyncio.PriorityQueue()
        self.semaphore = asyncio.Semaphore(max_concurrent)
        
    async def submit(self, task: AgentTask):
        await self.queue.put(task)
        
    async def worker(self):
        while True:
            task = await self.queue.get()
            async with self.semaphore:
                await self._process_task(task)
            self.queue.task_done()
    
    async def _process_task(self, task: AgentTask):
        # Agent 处理逻辑
        ...
    
    async def start(self, num_workers: int = 5):
        workers = [asyncio.create_task(self.worker()) 
                   for _ in range(num_workers)]
        await asyncio.gather(*workers)
```

### 模式三：超时控制

Agent 请求可能卡住（LLM API 超时、工具死循环），需要主动超时。

```python
class TimeoutableAgent:
    """带超时控制的 Agent"""
    
    def __init__(self, llm_timeout: float = 30.0):
        self.llm_timeout = llm_timeout
        
    async def call_llm_with_timeout(self, prompt: str) -> str:
        """带超时的 LLM 调用"""
        try:
            return await asyncio.wait_for(
                self.llm_client.chat(prompt),
                timeout=self.llm_timeout
            )
        except asyncio.TimeoutError:
            # 超时后的降级策略
            return self._fallback_response(prompt)
    
    async def run_agent_with_timeout(self, input_text: str, 
                                      overall_timeout: float = 120.0):
        """整个 Agent 运行带超时"""
        try:
            return await asyncio.wait_for(
                self._agent_loop(input_text),
                timeout=overall_timeout
            )
        except asyncio.TimeoutError:
            await self._save_checkpoint()
            return {"status": "timeout", "partial_result": self._get_partial()}
```

## CPU 密集型工具处理

Agent 的工具执行可能是 CPU 密集的（文档解析、数据分析、代码执行）。这些不应该在事件循环中直接执行。

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor
import multiprocessing

class HybridExecutor:
    """异步编排 + CPU 密集型工具在线程/进程池执行"""
    
    def __init__(self):
        self.cpu_cores = multiprocessing.cpu_count()
        # 线程池: 适合 I/O 密集型工具
        self.thread_pool = ThreadPoolExecutor(max_workers=self.cpu_cores * 4)
        # 进程池: 适合 CPU 密集型工具
        self.process_pool = ProcessPoolExecutor(max_workers=self.cpu_cores)
        self.loop = asyncio.get_event_loop()
    
    async def run_io_tool(self, fn, *args):
        """I/O 密集型工具：线程池"""
        return await self.loop.run_in_executor(
            self.thread_pool, fn, *args
        )
    
    async def run_cpu_tool(self, fn, *args):
        """CPU 密集型工具：进程池"""
        return await self.loop.run_in_executor(
            self.process_pool, fn, *args
        )
    
    async def agent_tool_orchestration(self, tool_name: str, args: dict):
        """Agent 工具调用编排 - 自动选择执行器"""
        
        TOOL_EXECUTOR_MAP = {
            "search_web": "io",        # HTTP 请求 -> 线程池
            "read_file": "io",         # 文件读取 -> 线程池
            "analyze_csv": "cpu",      # 数据分析 -> 进程池
            "generate_image": "cpu",   # 图像生成 -> 进程池
            "execute_python": "cpu",   # 代码执行 -> 进程池
            "query_database": "io",    # 数据库查询 -> 线程池
            "embed_document": "cpu",   # Embedding -> 进程池
        }
        
        executor_type = TOOL_EXECUTOR_MAP.get(tool_name, "io")
        fn = self._get_tool_function(tool_name)
        
        if executor_type == "cpu":
            return await self.run_cpu_tool(fn, **args)
        else:
            return await self.run_io_tool(fn, **args)
```

## 并发模型的性能基准

### 基准对比：100 个并发 Agent 请求

测试条件：每个 Agent 完成一轮 Thought→Action→Observation (LLM API + 工具调用 + 记忆存储)

```
并发模型            总耗时    CPU 利用率    内存使用     推荐场景
─────────           ─────    ──────────    ──────      ────────
同步单线程           500s     5%           50MB        不可用
Gunicorn (5 worker)  100s     15%          250MB       不推荐
ThreadPool (20)      40s      25%          500MB       少量同步调用
asyncio              5-8s     40-60%       200MB       默认选择 ✅
asyncio + 进程池     5-10s    50-80%       400MB       有 CPU 密集工具 ✅
Multiprocessing (4)  30s      60-70%       800MB       本地模型推理专用
```

### Python 并发模型选择速查

```
                      asyncio          Thread          Process
────────────────      ───────          ──────          ───────
适合的场景            I/O 密集         I/O 密集         CPU 密集
                      大量连接          少量连接          隔离任务
                      长连接保持         同步代码集成        安全沙箱

并发上限            10,000+           200-500           CPU 核心数

内存/连接           低 (~1KB/协程)    中 (~8MB/线程)    高 (~50MB/进程)

GIL 问题            不阻塞             阻塞 (CPU 操作)    无 (独立进程)

状态共享            不需要 (协程安全)   需要锁             需要序列化

启动时间            ~0ms              ~1ms              ~500ms

Agent 场景推荐       ✅ 强烈推荐        ❌ 避免           ⚠ 特定场景使用
```

## 关键挑战与应对

### 挑战 1：事件循环阻塞

```
阻塞操作                             后果
─────────                            ────
time.sleep(1)                      事件循环暂停 1s，所有 Agent 卡住
requests.get(url) (同步 HTTP)       同上
大型文件读取 (同步 read)              同上
密集的正则/解析 (CPU 密集)             同上

✅ 正确做法:
await asyncio.sleep(1)
httpx.AsyncClient().get(url)
aiofiles.open().read()
loop.run_in_executor(None, heavy_parse, data)
```

### 挑战 2：任务取消与清理

```python
class GracefulAgentCancel:
    """优雅取消 Agent 任务"""
    
    async def run(self, input_text: str):
        try:
            result = await self._agent_loop(input_text)
            return result
        except asyncio.CancelledError:
            # 清理资源
            await self._save_checkpoint()
            await self._release_tools()
            # 重新抛出给调用方
            raise
```

### 挑战 3：异步代码中的调试

异步代码的堆栈跟踪不如同步直观，需要特殊工具：

```python
# 启用 asyncio 调试
import asyncio

# 查看阻塞的事件循环
loop = asyncio.get_event_loop()
loop.set_debug(True)

# 检测长期运行的协程
# 设置: PYTHONASYNCIODEBUG=1

# 或者用 aiomonitor 实时监控
# pip install aiomonitor
# import aiomonitor
# with aiomonitor.start_monitor(loop):
#     loop.run_forever()
```

### 挑战 4：Gunicorn/Uvicorn 集成

```bash
# 使用 Uvicorn 运行异步 Agent 服务
# workers=1 因为 asyncio 单进程足以处理数百并发
# 如果需要多进程，加 --workers=N 启动 N 个独立事件循环

uvicorn agent_server:app --host 0.0.0.0 --port 8000 --workers=1

# 生产环境推荐配置
# --workers=4: 每个进程独立的事件循环
# --limit-concurrency=1000: 最大并发连接数
# --timeout-keep-alive=30: 长连接超时

gunicorn agent_server:app \
    --worker-class uvicorn.workers.UvicornWorker \
    --workers=4 \
    --timeout=120 \
    --keep-alive=30 \
    --max-requests=10000 \
    --max-requests-jitter=1000
```

## 工程优化方向

1. **连接复用**：为 LLM API、数据库、Redis 使用连接池，避免每次请求都建连
2. **零拷贝数据共享**：Agent 间共享的只读数据（工具定义、Prompt 模板）使用共享内存
3. **请求合并**：相同上下文的多个 Agent 请求合并为一次 LLM API 调用
4. **批量推理**：后端处理 Embedding 时合并多个请求为 batch 推理
5. **自适应并发度**：根据 LLM API 的响应时间和错误率动态调整并发限制
6. **协程本地存储**：使用 `contextvars` 替代 threading.local 传递 Agent 上下文

```python
# contextvars 示例：在异步 Agent 中安全传递上下文
import contextvars

current_session_id = contextvars.ContextVar('session_id')

class AgentMiddleware:
    async def __call__(self, request):
        token = current_session_id.set(request.session_id)
        try:
            return await self.app(request)
        finally:
            current_session_id.reset(token)

# 在任意深度获取当前会话 ID
async def log_event(event: str):
    session_id = current_session_id.get()
    await logger.log(session_id=session_id, event=event)
```

## 能力边界

- **asyncio 不能加速 CPU 密集型操作**：纯计算任务需要进程池
- **单事件循环在多核上的局限**：即使有 32 核 CPU，单个事件循环也只能用满一个核，需要多进程
- **Python GIL 仍然存在**：事件循环释放 GIL 只在 `await` 时，CPU 操作期间 GIL 会被持有
- **协程的调试比线程更复杂**：堆栈跟踪可能丢失上下文
- **大规模进程池的内存开销**：每个进程加载一次 Agent 框架，内存消耗跟进程数成正比

对于超大规模（10K+ 并发 Agent），考虑使用 Golang/Rust 重写 Agent 编排层，或使用 Ray 等分布式计算框架。
