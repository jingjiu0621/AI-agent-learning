# 自动化 Agent 架构：从定时任务到智能工作流引擎

## 1. 业务背景

### 1.1 什么是自动化 Agent

自动化 Agent 是一类专门用于执行预定义或动态生成的工作流任务的智能体系统。与对话式 Agent 不同，自动化 Agent 的核心目标是**可靠地完成一系列有序的操作步骤**，而不是与用户进行自由对话。这类 Agent 广泛应用于工作流自动化、数据处理管道（ETL）、CI/CD 流水线、以及企业业务流程自动化。

```
传统自动化                        AI Agent 自动化
┌─────────────────┐              ┌─────────────────────┐
│ 硬编码脚本       │              │  LLM 驱动的智能流程   │
│ 固定执行路径     │     ──→      │  动态步骤生成         │
│ 无异常处理       │              │  自适应错误恢复       │
│ 单机执行         │              │  分布式执行器池       │
│ Cron 触发        │              │  多触发器 + 事件驱动  │
└─────────────────┘              └─────────────────────┘
```

### 1.2 核心驱动力

| 驱动力 | 说明 |
|--------|------|
| **流程复杂性爆炸** | 现代业务逻辑涉及数十甚至上百个步骤，手动运维不可行 |
| **跨系统集成需求** | 需要协调多个 SaaS 系统、数据库、API 之间的数据流转 |
| **异常处理智能化** | 非确定性错误需要 AI 级别的判断能力，而非简单的重试 |
| **成本与效率压力** | 自动化每减少 1 分钟的人工操作，大型企业可节省数百万/年 |
| **合规与审计** | 每个操作步骤需要可追溯、可审计、可复现 |

---

## 2. 系统架构

### 2.1 整体架构总览

```
                             ┌────────────────────────────────────────────┐
                             │               API Gateway                   │
                             │     REST / gRPC / WebSocket / GraphQL       │
                             └────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────────┐
                    ▼                   ▼                        ▼
          ┌─────────────────┐ ┌─────────────────┐ ┌──────────────────────┐
          │   触发器层        │ │   工作流引擎      │ │    LLM 编排层         │
          │                  │ │                  │ │                      │
          │ • 时间触发(Cron) │ │ • DAG 定义器      │ │ • 自然语言→工作流转换  │
          │ • 事件触发(Kafka)│ │ • 状态机管理器     │ │ • 动态步骤生成         │
          │ • Webhook 触发   │ │ • 步骤执行器       │ │ • 异常分析 & 恢复策略  │
          │ • API 调用触发   │ │ • 上下文传递       │ │ • 审批节点建议         │
          └─────────────────┘ └─────────────────┘ └──────────────────────┘
                    │                  │                      │
                    │                  ▼                      │
                    │         ┌─────────────────┐            │
                    └────────►│   任务调度器      │◄───────────┘
                              │                  │
                              │ • 优先级队列      │
                              │ • 速率限制        │
                              │ • 分片分发        │
                              │ • 死信队列        │
                              └─────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                   ▼
          ┌─────────────────┐ ┌─────────────────┐ ┌──────────────────┐
          │  执行器池        │ │   Approval Gate  │ │   监控告警        │
          │                  │ │                  │ │                  │
          │ • K8s Job Pod    │ │ • 人工审批界面    │ │ • 指标采集(Prom)  │
          │ • 沙箱容器       │ │ • 多级审批链      │ │ • 日志聚合(ELK)   │
          │ • Serverless函数  │ │ • 超时自动策略    │ │ • 告警通知        │
          │ • 远程执行节点    │ │ • 审批人路由      │ │ • Dashboard      │
          └─────────────────┘ └─────────────────┘ └──────────────────┘
```

### 2.2 触发器层（Trigger Layer）

触发器层是自动化 Agent 的入口，负责将外部事件转化为工作流实例。

```
触发源分类:

时间触发                   事件触发                   Webhook 触发
┌──────────────┐   ┌──────────────────┐   ┌─────────────────────┐
│ Cron 表达式   │   │ Kafka 消息       │   │ GitHub push event   │
│ 每天 02:00   │   │ 订单创建事件      │   │ JIRA 状态变更       │
│ 每小时一次   │   │ 文件上传事件      │   │ Slack 命令         │
│ 每月第一天   │   │ 数据库 binlog    │   │ 自定义 HTTP 回调   │
│              │   │                  │   │                     │
│ 可预测性强   │   │ 实时性要求高      │   │ 集成外部系统        │
└──────────────┘   └──────────────────┘   └─────────────────────┘
```

**触发器的关键设计考量：**

```python
# 触发器注册与匹配逻辑示例
from abc import ABC, abstractmethod
from datetime import datetime
import json
import hashlib

class Trigger(ABC):
    """触发器基类"""
    
    def __init__(self, trigger_id: str, workflow_id: str):
        self.trigger_id = trigger_id
        self.workflow_id = workflow_id
        self.created_at = datetime.utcnow()
    
    @abstractmethod
    def evaluate(self, event: dict) -> bool:
        """判断事件是否匹配此触发器"""
        pass
    
    @abstractmethod
    def build_context(self, event: dict) -> dict:
        """从原始事件构建工作流初始上下文"""
        pass


class CronTrigger(Trigger):
    """时间触发器 - 基于 cron 表达式"""
    
    def __init__(self, trigger_id: str, workflow_id: str, 
                 cron_expr: str, timezone: str = "UTC"):
        super().__init__(trigger_id, workflow_id)
        self.cron_expr = cron_expr
        self.timezone = timezone
    
    def evaluate(self, event: dict) -> bool:
        # 在调度器层面已匹配，这里直接返回 True
        return event.get("type") == "cron" and \
               event.get("cron_expr") == self.cron_expr
    
    def build_context(self, event: dict) -> dict:
        return {
            "trigger_type": "cron",
            "trigger_id": self.trigger_id,
            "scheduled_time": event.get("timestamp"),
            "workflow_id": self.workflow_id
        }


class KafkaEventTrigger(Trigger):
    """事件触发器 - 基于 Kafka 消息"""
    
    def __init__(self, trigger_id: str, workflow_id: str,
                 topic: str, filter_expr: dict = None):
        super().__init__(trigger_id, workflow_id)
        self.topic = topic
        self.filter_expr = filter_expr or {}
    
    def evaluate(self, event: dict) -> bool:
        if event.get("type") != "kafka":
            return False
        if event.get("topic") != self.topic:
            return False
        # 应用过滤条件
        for key, value in self.filter_expr.items():
            if event.get("payload", {}).get(key) != value:
                return False
        return True
    
    def build_context(self, event: dict) -> dict:
        return {
            "trigger_type": "kafka_event",
            "trigger_id": self.trigger_id,
            "topic": self.topic,
            "offset": event.get("offset"),
            "partition": event.get("partition"),
            "payload": event.get("payload", {}),
            "received_at": datetime.utcnow().isoformat()
        }


class WebhookTrigger(Trigger):
    """Webhook 触发器 - 支持签名验证"""
    
    def __init__(self, trigger_id: str, workflow_id: str,
                 secret_key: str = None):
        super().__init__(trigger_id, workflow_id)
        self.secret_key = secret_key
    
    def verify_signature(self, payload: bytes, signature: str) -> bool:
        if not self.secret_key:
            return True
        expected = hashlib.sha256(
            self.secret_key.encode() + payload
        ).hexdigest()
        return hmac.compare_digest(expected, signature)
    
    def evaluate(self, event: dict) -> bool:
        return event.get("type") == "webhook" and \
               event.get("trigger_id") == self.trigger_id
    
    def build_context(self, event: dict) -> dict:
        return {
            "trigger_type": "webhook",
            "trigger_id": self.trigger_id,
            "headers": event.get("headers", {}),
            "body": event.get("body", {}),
            "received_at": datetime.utcnow().isoformat()
        }
```

### 2.3 工作流引擎（Workflow Engine）

工作流引擎是整个自动化 Agent 的核心——它定义了"做什么"和"按什么顺序做"。

```
DAG 工作流模型示例:

          ┌──────────────┐
          │   Start Node  │
          └──────┬───────┘
                 │
          ┌──────▼───────┐
          │  Validate     │──────── 失败 ──► ┌──────────────┐
          │  Input        │                  │  Notify      │
          └──────┬───────┘                  │  Admin       │
                 │ 通过                      └──────────────┘
          ┌──────▼───────┐
          │  Transform   │
          │  Data        │
          └──────┬───────┘
                 │
         ┌───────┴────────┐
         ▼                ▼
  ┌──────────────┐  ┌──────────────┐
  │  Enrich From  │  │  Enrich From │
  │  Database A   │  │  API B       │
  └──────┬───────┘  └──────┬───────┘
         │                 │
         └───────┬─────────┘
                 │
          ┌──────▼───────┐
          │  Human       │
          │  Approval    │◄──────── 审批人收到通知
          └──────┬───────┘
                 │
          ┌──────▼───────┐      ┌──────────────┐
          │  Write to    │      │  If rejected │
          │  Warehouse   │      │  → Rollback  │
          └──────┬───────┘      └──────────────┘
                 │
          ┌──────▼───────┐
          │  Send        │
          │  Notification│
          └──────┬───────┘
                 │
          ┌──────▼───────┐
          │   End Node   │
          └──────────────┘
```

**工作流引擎的核心实现：**

```python
from enum import Enum
from typing import Dict, List, Optional, Callable, Any
from datetime import datetime
import asyncio
import uuid
import logging

logger = logging.getLogger(__name__)


class StepStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    SKIPPED = "skipped"
    WAITING_APPROVAL = "waiting_approval"
    CANCELLED = "cancelled"


class WorkflowStatus(Enum):
    CREATED = "created"
    RUNNING = "running"
    PAUSED = "paused"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"


class StepResult:
    """单个步骤的执行结果"""
    
    def __init__(self, step_name: str, status: StepStatus,
                 output: dict = None, error: str = None,
                 duration_ms: int = 0):
        self.step_name = step_name
        self.status = status
        self.output = output or {}
        self.error = error
        self.duration_ms = duration_ms
        self.timestamp = datetime.utcnow().isoformat()


class WorkflowStep:
    """工作流中的单个步骤"""
    
    def __init__(self, name: str, 
                 execute_fn: Callable,
                 retry_count: int = 0,
                 retry_delay_sec: int = 5,
                 timeout_sec: int = 300,
                 requires_approval: bool = False,
                 depends_on: List[str] = None,
                 max_parallel: int = 1):
        self.name = name
        self.execute_fn = execute_fn          # 实际执行逻辑
        self.retry_count = retry_count        # 最大重试次数
        self.retry_delay_sec = retry_delay_sec  # 重试间隔
        self.timeout_sec = timeout_sec         # 超时时间
        self.requires_approval = requires_approval  # 需要人工审批
        self.depends_on = depends_on or []     # 依赖的前置步骤
        self.max_parallel = max_parallel       # 最大并行度


class WorkflowDefinition:
    """工作流定义 - DAG 结构"""
    
    def __init__(self, workflow_id: str, name: str,
                 steps: Dict[str, WorkflowStep],
                 start_step: str,
                 error_handlers: Dict[str, str] = None,
                 max_duration_min: int = 60):
        self.workflow_id = workflow_id
        self.name = name
        self.steps = steps                     # step_name -> WorkflowStep
        self.start_step = start_step           # 入口步骤
        self.error_handlers = error_handlers or {}  # 错误 -> 处理策略
        self.max_duration_min = max_duration_min     # 最大执行时间
        self._validate_dag()
    
    def _validate_dag(self):
        """验证 DAG 无循环依赖"""
        visited = set()
        path = []
        
        def dfs(step_name):
            if step_name in path:
                cycle = " -> ".join(path[path.index(step_name):] + [step_name])
                raise ValueError(f"检测到循环依赖: {cycle}")
            if step_name in visited:
                return
            visited.add(step_name)
            path.append(step_name)
            step = self.steps.get(step_name)
            if step:
                for dep in step.depends_on:
                    if dep in self.steps:
                        dfs(dep)
            path.pop()
        
        dfs(self.start_step)


class WorkflowInstance:
    """工作流运行时实例"""
    
    def __init__(self, definition: WorkflowDefinition,
                 context: dict = None):
        self.instance_id = str(uuid.uuid4())
        self.definition = definition
        self.context = context or {}
        self.status = WorkflowStatus.CREATED
        self.step_results: Dict[str, StepResult] = {}
        self.created_at = datetime.utcnow()
        self.updated_at = datetime.utcnow()
    
    async def execute_step(self, step_name: str) -> StepResult:
        """执行单个步骤（含重试逻辑）"""
        step = self.definition.steps[step_name]
        logger.info(f"执行步骤: {step_name}, 实例: {self.instance_id}")
        
        # 检查前置依赖
        for dep in step.depends_on:
            dep_result = self.step_results.get(dep)
            if not dep_result or dep_result.status != StepStatus.SUCCESS:
                raise ValueError(f"前置步骤 {dep} 未完成或失败")
        
        # 检查是否需要审批
        if step.requires_approval:
            return await self._handle_approval_gate(step_name, step)
        
        # 执行步骤（含重试）
        last_error = None
        for attempt in range(step.retry_count + 1):
            try:
                start_time = datetime.utcnow()
                result = await asyncio.wait_for(
                    step.execute_fn(self.context),
                    timeout=step.timeout_sec
                )
                duration = int(
                    (datetime.utcnow() - start_time).total_seconds() * 1000
                )
                
                step_result = StepResult(
                    step_name=step_name,
                    status=StepStatus.SUCCESS,
                    output=result,
                    duration_ms=duration
                )
                self.step_results[step_name] = step_result
                self.context[step_name] = result
                return step_result
                
            except Exception as e:
                last_error = e
                logger.warning(
                    f"步骤 {step_name} 第 {attempt+1} 次尝试失败: {str(e)}"
                )
                if attempt < step.retry_count:
                    await asyncio.sleep(step.retry_delay_sec)
        
        # 所有重试均失败
        error_step = StepResult(
            step_name=step_name,
            status=StepStatus.FAILED,
            error=str(last_error)
        )
        self.step_results[step_name] = error_step
        
        # 调用错误处理策略
        await self._handle_step_failure(step_name, last_error)
        return error_step
    
    async def _handle_approval_gate(self, step_name: str, 
                                     step: WorkflowStep) -> StepResult:
        """审批门禁处理"""
        logger.info(f"步骤 {step_name} 等待人工审批")
        
        # 通知审批人
        approval_request = {
            "instance_id": self.instance_id,
            "step_name": step_name,
            "context_snapshot": self.context,
            "requested_at": datetime.utcnow().isoformat()
        }
        
        # 发布审批事件（由外部审批服务处理）
        await self._publish_approval_event(approval_request)
        
        # 设置审批等待状态
        step_result = StepResult(
            step_name=step_name,
            status=StepStatus.WAITING_APPROVAL,
            output={"approval_request": approval_request}
        )
        self.step_results[step_name] = step_result
        self.status = WorkflowStatus.PAUSED
        return step_result
    
    async def resolve_approval(self, step_name: str, 
                                approved: bool, 
                                reviewer: str) -> StepResult:
        """处理审批结果"""
        current = self.step_results.get(step_name)
        if not current or current.status != StepStatus.WAITING_APPROVAL:
            raise ValueError(f"步骤 {step_name} 不在审批等待状态")
        
        if approved:
            # 审批通过，执行步骤
            return await self.execute_step(step_name)
        else:
            # 审批拒绝，标记为跳过
            step_result = StepResult(
                step_name=step_name,
                status=StepStatus.SKIPPED,
                output={"rejected_by": reviewer,
                        "rejected_at": datetime.utcnow().isoformat()}
            )
            self.step_results[step_name] = step_result
            self.status = WorkflowStatus.RUNNING
            return step_result
    
    async def _handle_step_failure(self, step_name: str, 
                                    error: Exception):
        """步骤失败后的处理策略"""
        error_type = type(error).__name__
        strategy = self.definition.error_handlers.get(
            error_type, "fail"
        )
        
        if strategy == "skip":
            logger.info(f"步骤 {step_name} 失败，策略：跳过")
            # 继续执行下一个步骤
        elif strategy == "notify":
            logger.info(f"步骤 {step_name} 失败，策略：通知管理员")
            await self._notify_admin(step_name, str(error))
        elif strategy == "retry_later":
            logger.info(f"步骤 {step_name} 失败，策略：稍后重试")
            # 将任务发送到死信队列
            await self._send_to_dead_letter(step_name, str(error))
        else:
            # fail 策略：终止整个工作流
            logger.error(f"步骤 {step_name} 失败，终止工作流")
            self.status = WorkflowStatus.FAILED
    
    async def run(self):
        """启动工作流执行（BFS 拓扑排序）"""
        self.status = WorkflowStatus.RUNNING
        logger.info(f"工作流启动: {self.definition.name} [{self.instance_id}]")
        
        # 拓扑排序确定执行顺序
        execution_order = self._topological_sort()
        
        for step_name in execution_order:
            if self.status == WorkflowStatus.FAILED:
                break
            if self.status == WorkflowStatus.PAUSED:
                # 等待审批恢复
                break
            await self.execute_step(step_name)
        
        # 检查所有步骤是否完成
        all_success = all(
            r.status == StepStatus.SUCCESS 
            for r in self.step_results.values()
        )
        self.status = WorkflowStatus.COMPLETED if all_success else self.status
    
    def _topological_sort(self) -> List[str]:
        """基于 BFS 的拓扑排序"""
        in_degree = {}
        for name, step in self.definition.steps.items():
            in_degree[name] = len(step.depends_on)
        
        queue = [n for n, d in in_degree.items() if d == 0]
        result = []
        
        while queue:
            node = queue.pop(0)
            result.append(node)
            for name, step in self.definition.steps.items():
                if node in step.depends_on:
                    in_degree[name] -= 1
                    if in_degree[name] == 0:
                        queue.append(name)
        
        return result
    
    async def _publish_approval_event(self, request: dict):
        """发布审批事件到消息队列"""
        # 实际实现中会发送到 Kafka / Redis PubSub
        pass
    
    async def _notify_admin(self, step_name: str, error: str):
        """通知管理员"""
        # 发送邮件 / Slack 消息 / PagerDuty
        pass
    
    async def _send_to_dead_letter(self, step_name: str, error: str):
        """发送到死信队列"""
        # 存储失败任务以便后续分析和重试
        pass
```

### 2.4 任务调度器（Task Scheduler）

任务调度器负责将工作流步骤分发到执行器池，管理优先级和并发。

```
调度器内部架构:

                    ┌─────────────────────────────────────┐
                    │           Task Scheduler             │
                    │                                      │
                    │  ┌──────────┐  ┌──────────┐         │
                    │  │ 优先级队列  │  │ 普通队列   │         │
                    │  │ (Redis)   │  │ (Redis)   │         │
                    │  └─────┬────┘  └─────┬────┘         │
                    │        │              │              │
                    │  ┌─────▼──────────────▼────┐         │
                    │  │     Dispatch Worker      │         │
                    │  │   (速率限制 + 分片策略)   │         │
                    │  └─────┬───────────────────┘         │
                    │        │                              │
                    │  ┌─────▼───────────────────┐         │
                    │  │    Dead Letter Queue     │         │
                    │  │   (失败 + 超时 + 重试耗尽) │         │
                    │  └─────────────────────────┘         │
                    └─────────────────────────────────────┘
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
            ┌──────────────┐   ┌──────────────┐
            │ Executor K8s  │   │ Executor K8s  │
            │ Pod - high    │   │ Pod - normal  │
            │ PriorityClass │   │ PriorityClass │
            └──────────────┘   └──────────────┘
```

### 2.5 执行器池（Executor Pool）

执行器池管理实际的步骤执行环境，支持容器化隔离和资源控制。

```
执行器生命周期:

Idle ──► Acquired ──► Initialized ──► Executing ──► Completed ──► Released ──► Idle
                         │               │               │
                         ▼               ▼               ▼
                      Init Error     Execution Error   Cleanup Error
                         │               │               │
                         ▼               ▼               ▼
                      Failed ───────────────────────────► Released ──► Idle
```

**执行器管理核心代码：**

```python
class ExecutorPool:
    """执行器池管理器"""
    
    def __init__(self, min_size: int = 5, max_size: int = 50,
                 container_image: str = "workflow-executor:latest"):
        self.min_size = min_size
        self.max_size = max_size
        self.container_image = container_image
        self.idle_executors: asyncio.Queue = asyncio.Queue()
        self.active_executors: Dict[str, Executor] = {}
        self._lock = asyncio.Lock()
    
    async def acquire(self, task: dict) -> 'Executor':
        """获取可用的执行器"""
        # 尝试从空闲池获取
        executor = await self._try_get_idle()
        if executor:
            return executor
        
        # 如果没有空闲且未达上限，创建新的
        async with self._lock:
            if len(self.active_executors) < self.max_size:
                executor = await self._create_executor()
                self.active_executors[executor.id] = executor
                return executor
        
        # 达到上限，等待
        return await self.idle_executors.get()
    
    async def release(self, executor: 'Executor'):
        """释放执行器回池"""
        await executor.reset()
        self.active_executors.pop(executor.id, None)
        await self.idle_executors.put(executor)
    
    async def _create_executor(self) -> 'Executor':
        """创建新的执行器（启动容器）"""
        # 实际实现中调用 K8s API / Docker SDK 创建容器
        executor = Executor(
            executor_id=str(uuid.uuid4()),
            container_image=self.container_image
        )
        await executor.initialize()
        return executor
    
    async def scale_to(self, target: int):
        """扩容/缩容到指定数量"""
        current = len(self.active_executors) + self.idle_executors.qsize()
        if target > current:
            for _ in range(target - current):
                executor = await self._create_executor()
                await self.idle_executors.put(executor)
        elif target < current:
            to_remove = current - target
            for _ in range(to_remove):
                executor = await self.idle_executors.get()
                await executor.shutdown()
```

---

## 3. 核心设计决策

### 3.1 静态 DAG vs 动态 LLM 生成工作流

这是自动化 Agent 架构中最关键的设计分水岭。

```
静态 DAG 工作流                        动态 LLM 生成工作流
┌──────────────────────┐              ┌──────────────────────────┐
│ 预定义步骤和依赖关系    │              │ 根据用户意图实时生成步骤    │
│ 确定性强               │              │ 灵活适应新场景             │
│ 容易测试和验证          │              │ 需要校验生成结果的有效性    │
│ 适合固定业务流程        │              │ 适合非结构化任务            │
│                      │              │                          │
│ 优点:                 │              │ 优点:                     │
│ • 可靠性高             │              │ • 零编码 = 零维护成本      │
│ • 延迟可预测           │              │ • 可处理意外场景           │
│ • 审计简单             │              │ • 快速适应业务变化         │
│                      │              │                          │
│ 缺点:                 │              │ 缺点:                     │
│ • 维护工作量大          │              │ • LLM 幻觉风险            │
│ • 业务变化需改代码      │              │ • 执行路径不可预测         │
│ • 无法处理未知场景      │              │ • 每次执行成本较高         │
└──────────────────────┘              └──────────────────────────┘
```

**混合方案**：核心步骤使用静态 DAG + 非关键路径使用 LLM 生成。

```python
class HybridWorkflowEngine:
    """混合工作流引擎：静态 DAG + LLM 动态扩展"""
    
    def __init__(self, llm_client, static_workflows: Dict[str, WorkflowDefinition]):
        self.llm = llm_client
        self.static_workflows = static_workflows
        
    async def build_workflow(self, user_request: str) -> WorkflowDefinition:
        """根据用户请求构建工作流"""
        
        # 阶段 1：意图识别 - 匹配预定义工作流
        intent = await self._classify_intent(user_request)
        if intent in self.static_workflows:
            return self.static_workflows[intent]
        
        # 阶段 2：LLM 动态生成
        generated = await self._llm_generate_workflow(user_request)
        
        # 阶段 3：验证生成结果
        validated = await self._validate_workflow(generated)
        if validated:
            return validated
        
        # 阶段 4：验证失败，回退到通用模式
        return self._fallback_workflow(user_request)
    
    async def _llm_generate_workflow(self, request: str) -> dict:
        """使用 LLM 生成工作流定义"""
        prompt = f"""
        根据以下用户请求，生成一个工作流 DAG 定义。
        请求: {request}
        
        请以 JSON 格式返回，包含:
        - steps: 步骤列表，每个步骤有 name, description, depends_on, retry_count
        - 用拓扑顺序排列步骤
        - 只返回 JSON，不要其他文字
        
        示例格式:
        {{
            "steps": [
                {{"name": "validate_input", "description": "验证输入数据", 
                  "depends_on": [], "retry_count": 0}},
                {{"name": "process_data", "description": "处理数据",
                  "depends_on": ["validate_input"], "retry_count": 2}}
            ]
        }}
        """
        
        response = await self.llm.complete(prompt)
        return json.loads(self._extract_json(response))
    
    async def _validate_workflow(self, definition: dict) -> bool:
        """验证 LLM 生成的工作流定义"""
        if "steps" not in definition or len(definition["steps"]) == 0:
            return False
        
        step_names = set()
        for step in definition["steps"]:
            if step["name"] in step_names:
                return False  # 步骤名重复
            step_names.add(step["name"])
            
            # 验证 depends_on 引用的步骤都存在
            for dep in step.get("depends_on", []):
                if dep not in step_names and dep != "":
                    return False
        
        return True
```

### 3.2 人工审批门禁（Approval Gate）

某些关键操作必须在自动化流程中插入人工确认环节。

```
审批门禁模式:

              ┌──────────────────────────┐
              │   Automation Step        │
              │   (e.g., 数据预处理完成)   │
              └──────────┬───────────────┘
                         │
              ┌──────────▼───────────────┐
              │   Approval Gate          │
              │                          │
              │   通知方式:                │
              │   • Slack 消息 + 按钮     │
              │   • 邮件 + 审批链接       │
              │   • 企业微信 / 钉钉通知   │
              │                          │
              │   策略配置:                │
              │   • 超时自动策略           │
              │     - auto_approve: 是    │
              │     - auto_reject: 否     │
              │     - escalate: 升级通知   │
              │   • 多级审批链             │
              │     - 金额 > 10万 → 经理  │
              │     - 金额 > 100万 → VP   │
              └──────────┬───────────────┘
                         │
           ┌─────────────┴─────────────┐
           ▼                           ▼
    ┌──────────────┐          ┌──────────────┐
    │  Approved    │          │  Rejected    │
    │  → Continue  │          │  → Rollback  │
    └──────────────┘          └──────────────┘
```

```python
class ApprovalGate:
    """审批门禁服务"""
    
    def __init__(self, notification_service, 
                 policy_store, 
                 escalation_service):
        self.notifier = notification_service
        self.policy_store = policy_store
        self.escalator = escalation_service
    
    async def request_approval(self, 
                                step_name: str,
                                workflow_instance_id: str,
                                context: dict) -> str:
        """发起审批请求，返回审批票据 ID"""
        
        # 确定审批人
        approver = await self._resolve_approver(step_name, context)
        
        # 创建审批票据
        ticket = {
            "ticket_id": str(uuid.uuid4()),
            "workflow_instance_id": workflow_instance_id,
            "step_name": step_name,
            "approver": approver,
            "context": context,
            "status": "pending",
            "created_at": datetime.utcnow().isoformat(),
            "timeout_policy": await self._get_timeout_policy(step_name)
        }
        
        # 发送审批通知
        await self.notifier.send_approval_request(
            approver=approver,
            ticket_id=ticket["ticket_id"],
            context_snapshot=context
        )
        
        # 启动超时计时器
        asyncio.create_task(
            self._approval_timeout_monitor(ticket)
        )
        
        return ticket["ticket_id"]
    
    async def resolve(self, ticket_id: str, 
                       approved: bool, 
                       reviewer: str,
                       comment: str = ""):
        """处理审批结果"""
        if approved:
            await self._on_approved(ticket_id, reviewer)
        else:
            await self._on_rejected(ticket_id, reviewer, comment)
    
    async def _resolve_approver(self, step_name: str, 
                                 context: dict) -> str:
        """根据审批策略确定审批人"""
        policies = await self.policy_store.get_policies(step_name)
        
        for policy in policies:
            if policy.condition_matches(context):
                return policy.approver
        
        return "default-approver@company.com"  # 兜底审批人
    
    async def _approval_timeout_monitor(self, ticket: dict):
        """审批超时监控"""
        timeout_sec = ticket["timeout_policy"].get("timeout_sec", 3600)
        await asyncio.sleep(timeout_sec)
        
        # 检查是否仍未处理
        if ticket["status"] == "pending":
            timeout_action = ticket["timeout_policy"].get("action", "escalate")
            
            if timeout_action == "auto_approve":
                await self.resolve(ticket["ticket_id"], 
                                   True, "system[timeout]")
            elif timeout_action == "auto_reject":
                await self.resolve(ticket["ticket_id"], 
                                   False, "system[timeout]")
            elif timeout_action == "escalate":
                await self.escalator.escalate(ticket["ticket_id"])
```

### 3.3 错误恢复策略

| 策略 | 适用场景 | 风险 |
|------|----------|------|
| **Retry** | 临时性错误（网络超时、限流） | 可能导致重复执行 |
| **Skip** | 非关键步骤失败 | 数据可能不完整 |
| **Notify** | 需要人工判断的错误 | 延迟增加 |
| **Rollback** | 关键步骤失败 | 实现复杂，需要补偿事务 |
| **Dead Letter** | 多次重试仍失败 | 需要定期处理死信 |

### 3.4 并行执行策略

```
并行度控制:

顺序执行：                  并行执行：
Step1 ──► Step2 ──► Step3   Step1 ──► ┌──► Step2 ──┐
                                      │            │
                                      ├──► Step3 ──┤──► Step4
                                      │            │
                                      └──► Step5 ──┘

扇出/汇聚模式:              DAG 并行:
   ┌──► Step2 ──┐             Step1 ──┬──► Step2 ──► Step4
   │            │                     │
Step1 ──► Step3 ──► Step4            └──► Step3 ──┘
   │            │
   └──► Step4 ──┘
```

---

## 4. 与其他 Agent 类型的对比

| 维度 | 自动化 Agent | 对话 Agent | 研究 Agent |
|------|-------------|-----------|-----------|
| **核心目标** | 可靠完成预定义工作流 | 自然语言交互 | 信息收集与综合 |
| **输出稳定性** | 极高（确定性步骤） | 中等（对话多样性） | 低（结果依赖搜索） |
| **执行方式** | DAG 拓扑排序执行 | 流式生成 | 递归搜索 |
| **错误处理** | 重试 / 跳过 / 回滚 | 重新生成响应 | 更换搜索策略 |
| **状态管理** | 持久化状态机 | 对话历史 | 搜索图遍历 |
| **延迟要求** | 分钟级 ~ 小时级 | 秒级响应 | 秒级 ~ 分钟级 |
| **典型场景** | ETL, CI/CD, 审批流 | 客服, 助手 | 竞品分析, 调研 |
| **LLM 使用** | 仅动态生成/异常处理 | 每一步 | 搜索策略决策 |
| **人工介入** | 审批门禁（必要） | 可选 | 无需 |

---

## 5. 代码示例

### 5.1 完整工作流定义与执行

```python
import asyncio
import json
from datetime import datetime

# ==================== 定义步骤函数 ====================

async def validate_input(ctx: dict) -> dict:
    """验证输入数据的完整性"""
    data = ctx.get("input_data", {})
    if not data:
        raise ValueError("输入数据为空")
    if "customer_id" not in data:
        raise ValueError("缺少 customer_id 字段")
    if "amount" not in data:
        raise ValueError("缺少 amount 字段")
    
    return {
        "validated": True,
        "record_count": 1,
        "data_schema": list(data.keys())
    }


async def transform_data(ctx: dict) -> dict:
    """数据转换与清洗"""
    raw = ctx.get("input_data", {})
    
    # 转换操作：格式化金额、标准化日期、脱敏处理
    transformed = {
        "customer_id": raw["customer_id"],
        "amount": round(float(raw["amount"]), 2),
        "currency": raw.get("currency", "CNY").upper(),
        "processed_at": datetime.utcnow().isoformat(),
        "masked_name": raw.get("name", "N/A")[:1] + "**" 
                       if raw.get("name") else "N/A"
    }
    
    return {"transformed_data": transformed}


async def enrich_from_crm(ctx: dict) -> dict:
    """从 CRM 系统获取客户信息"""
    customer_id = ctx.get("input_data", {}).get("customer_id")
    
    # 模拟 CRM API 调用
    await asyncio.sleep(0.5)
    
    crm_data = {
        "customer_tier": "gold",
        "lifetime_value": 50000,
        "risk_score": 0.12,
        "credit_limit": 100000
    }
    
    return {"crm_info": crm_data}


async def calculate_risk(ctx: dict) -> dict:
    """风险评估"""
    crm_info = ctx.get("enrich_from_crm", {}).get("crm_info", {})
    amount = ctx.get("input_data", {}).get("amount", 0)
    
    risk_score = crm_info.get("risk_score", 0.5)
    amount_ratio = float(amount) / max(crm_info.get("credit_limit", 1), 1)
    
    final_risk = risk_score * 0.7 + amount_ratio * 0.3
    
    return {
        "risk_score": final_risk,
        "risk_level": "low" if final_risk < 0.3 
                      else "medium" if final_risk < 0.7 
                      else "high"
    }


async def execute_write(ctx: dict) -> dict:
    """写入数据仓库"""
    transformed = ctx.get("transform_data", {}).get("transformed_data", {})
    risk_info = ctx.get("calculate_risk", {}).get("risk_score", 0)
    
    # 模拟数据库写入
    await asyncio.sleep(0.3)
    
    return {
        "record_id": f"REC-{datetime.utcnow().strftime('%Y%m%d%H%M%S')}",
        "status": "written",
        "target_table": "transactions"
    }


async def send_notification(ctx: dict) -> dict:
    """发送执行结果通知"""
    result = ctx.get("execute_write", {})
    
    notification = {
        "channel": "slack",
        "channel_id": "#data-pipeline",
        "message": f"数据处理完成: {result.get('record_id', 'N/A')}",
        "sent_at": datetime.utcnow().isoformat()
    }
    
    return {"notification": notification}


# ==================== 定义工作流 ====================

def build_payment_workflow():
    """构建支付处理工作流"""
    
    steps = {
        "validate_input": WorkflowStep(
            name="validate_input",
            execute_fn=validate_input,
            retry_count=0,           # 输入验证无需重试
            timeout_sec=30
        ),
        "transform_data": WorkflowStep(
            name="transform_data",
            execute_fn=transform_data,
            retry_count=1,           
            retry_delay_sec=3,
            timeout_sec=60,
            depends_on=["validate_input"]
        ),
        "enrich_from_crm": WorkflowStep(
            name="enrich_from_crm",
            execute_fn=enrich_from_crm,
            retry_count=2,           
            retry_delay_sec=5,
            timeout_sec=30,
            depends_on=["validate_input"]
        ),
        "calculate_risk": WorkflowStep(
            name="calculate_risk",
            execute_fn=calculate_risk,
            retry_count=1,
            timeout_sec=30,
            depends_on=["enrich_from_crm"],
            requires_approval=True   # 风险计算需要人工审批
        ),
        "execute_write": WorkflowStep(
            name="execute_write",
            execute_fn=execute_write,
            retry_count=2,
            retry_delay_sec=10,
            timeout_sec=120,
            depends_on=["transform_data", "calculate_risk"]
        ),
        "send_notification": WorkflowStep(
            name="send_notification",
            execute_fn=send_notification,
            retry_count=2,
            timeout_sec=30,
            depends_on=["execute_write"]
        )
    }
    
    # 错误处理策略: 不同类型错误采用不同策略
    error_handlers = {
        "ValueError": "skip",        # 数据验证错误 -> 跳过
        "TimeoutError": "retry",     # 超时 -> 重试
        "ConnectionError": "retry",  # 连接错误 -> 重试
        "RateLimitError": "retry_later"  # 限流 -> 稍后重试
    }
    
    return WorkflowDefinition(
        workflow_id="payment-workflow-v1",
        name="支付数据处理工作流",
        steps=steps,
        start_step="validate_input",
        error_handlers=error_handlers,
        max_duration_min=30
    )


# ==================== 执行工作流 ====================

async def main():
    workflow = build_payment_workflow()
    
    context = {
        "input_data": {
            "customer_id": "CUST-2024-001",
            "amount": "1500.00",
            "currency": "CNY",
            "name": "张三"
        },
        "source": "webhook_payment_system"
    }
    
    instance = WorkflowInstance(workflow, context)
    
    print(f"启动工作流: {instance.instance_id}")
    print(f"工作流名称: {workflow.name}")
    print(f"步骤数量: {len(workflow.steps)}")
    print("-" * 50)
    
    await instance.run()
    
    print(f"\n工作流状态: {instance.status.value}")
    print("\n步骤执行结果:")
    for step_name, result in instance.step_results.items():
        status_icon = "✓" if result.status == StepStatus.SUCCESS else "✗"
        print(f"  {status_icon} {step_name}: {result.status.value} "
              f"({result.duration_ms}ms)")
        if result.error:
            print(f"    错误: {result.error}")


if __name__ == "__main__":
    asyncio.run(main())
```

### 5.2 LLM 动态步骤生成

```python
class LLMWorkflowGenerator:
    """基于 LLM 的工作流动态生成器"""
    
    def __init__(self, llm_client, 
                 tool_registry: Dict[str, Callable]):
        self.llm = llm_client
        self.tools = tool_registry  # 注册可用的执行工具
    
    async def generate_step(self, 
                             goal: str, 
                             previous_results: List[dict]) -> dict:
        """LLM 动态决定下一步要做什么"""
        
        # 构建可用工具的描述
        tool_descriptions = "\n".join([
            f"- {name}: {tool.__doc__ or '无描述'}"
            for name, tool in self.tools.items()
        ])
        
        prompt = f"""
        你是一个工作流编排 Agent。根据当前目标和已完成的步骤，决定下一步操作。
        
        目标: {goal}
        
        已完成步骤:
        {json.dumps(previous_results, indent=2, ensure_ascii=False)}
        
        可用工具:
        {tool_descriptions}
        
        请返回 JSON 格式的决策:
        {{
            "action": "工具名称 或 'complete' 或 'request_human_input'",
            "parameters": {{ "参数1": "值1" }},
            "reasoning": "选择此步骤的原因",
            "requires_approval": false
        }}
        """
        
        response = await self.llm.complete(prompt)
        decision = json.loads(self._extract_json(response))
        
        if decision["action"] == "complete":
            return {"type": "complete"}
        
        if decision["action"] == "request_human_input":
            return {"type": "human_input", 
                    "question": decision["parameters"].get("question", "")}
        
        if decision["action"] in self.tools:
            return {
                "type": "execute",
                "tool": decision["action"],
                "parameters": decision["parameters"],
                "requires_approval": decision.get("requires_approval", False)
            }
        
        raise ValueError(f"未知的操作: {decision['action']}")
```

---

## 6. 关键挑战

### 6.1 可靠性保证

| 挑战 | 解决方案 |
|------|---------|
| **执行幂等性** | 每个步骤生成唯一 ID，操作前检查是否已执行 |
| **Exactly-Once 语义** | 分布式事务 + 幂等性键 + 去重队列 |
| **状态持久化** | 每步执行结果写入持久化存储（PostgreSQL / DynamoDB） |
| **崩溃恢复** | Checkpoint + 断点续传，从失败步骤恢复执行 |

### 6.2 幂等性实现

```python
class IdempotentExecutor:
    """幂等执行器包装"""
    
    def __init__(self, state_store, ttl_seconds: int = 86400):
        self.store = state_store  # Redis / DynamoDB
        self.ttl = ttl_seconds
    
    async def execute_once(self, 
                            workflow_id: str,
                            step_name: str,
                            execution_key: str,
                            execute_fn: Callable) -> dict:
        """保证每个 execution_key 只执行一次"""
        
        full_key = f"idempotent:{workflow_id}:{step_name}:{execution_key}"
        
        # 1. 检查是否已经执行过
        existing = await self.store.get(full_key)
        if existing:
            logger.info(f"步骤 {step_name} 已执行过，返回缓存结果")
            return json.loads(existing)
        
        # 2. 通过分布式锁防止并发执行
        lock_key = f"lock:{full_key}"
        lock_acquired = await self.store.setnx(lock_key, "locked", ttl=60)
        if not lock_acquired:
            # 等待锁释放
            await asyncio.sleep(1)
            return await self.execute_once(workflow_id, step_name, 
                                            execution_key, execute_fn)
        
        try:
            # 3. 再次检查（双重检测）
            cached = await self.store.get(full_key)
            if cached:
                return json.loads(cached)
            
            # 4. 实际执行
            result = await execute_fn()
            
            # 5. 缓存结果（含 TTL）
            await self.store.setex(
                full_key, 
                json.dumps(result, ensure_ascii=False),
                self.ttl
            )
            
            return result
        finally:
            # 6. 释放锁
            await self.store.delete(lock_key)
```

### 6.3 长时间运行的工作流管理

长时间运行的工作流（小时级~天级）面临额外的挑战：

1. **工作流暂停与恢复**：支持暂停、恢复、取消操作
2. **版本兼容性**：工作流定义可能在执行期间更新
3. **资源泄漏**：确保超时或取消时正确清理资源
4. **进度可视化**：向用户展示当前执行位置和预计完成时间
5. **心跳检测**：检测挂起的执行器并重新调度

### 6.4 LLM 调用成本控制

```python
class CostAwareLLMOrchestrator:
    """成本感知的 LLM 编排器"""
    
    def __init__(self, llm_client, budget_per_workflow: float = 0.50):
        self.llm = llm_client
        self.budget = budget_per_workflow
        self.total_cost = 0.0
        self.llm_calls = 0
    
    async def should_use_llm(self, step_name: str) -> bool:
        """判断当前步骤是否需要使用 LLM"""
        
        # 策略 1: 预算控制
        if self.total_cost >= self.budget:
            return False  # 超出预算，使用规则引擎
        
        # 策略 2: 步骤类型判断
        non_llm_steps = {"validate", "write", "notify", "transform"}
        if step_name in non_llm_steps:
            return False  # 确定性步骤无需 LLM
        
        # 策略 3: 缓存命中
        cache_key = f"llm_cache:{step_name}:{self._get_context_hash()}"
        cached = await self._check_cache(cache_key)
        if cached:
            return False  # 命中缓存
        
        return True
    
    async def llm_call_with_cost(self, prompt: str) -> str:
        """执行 LLM 调用并记录成本"""
        self.llm_calls += 1
        
        response = await self.llm.complete(prompt)
        
        # 估算成本
        input_tokens = len(prompt) / 4  # 粗略估计
        output_tokens = len(str(response)) / 4
        cost = (input_tokens * 3 + output_tokens * 15) / 1_000_000  # $/token
        self.total_cost += cost
        
        logger.info(
            f"LLM 调用 #{self.llm_calls}: cost=${cost:.4f}, "
            f"total=${self.total_cost:.4f}"
        )
        
        return response
```

---

## 7. 应用场景

### 7.1 CI/CD 自动化

```
代码提交 ──► 静态检查 ──► 单元测试 ──► 构建 ──► 集成测试 ──► 部署
  │            │            │           │         │           │
  ▼            ▼            ▼           ▼         ▼           ▼
GitHub    SonarQube    pytest       Docker    Playwright    K8s Rollout
Webhook   Code Scan    UT + Cov    Build     E2E Test     Canary %
```

### 7.2 数据管道自动化（ETL）

```
数据源系统                             目标系统
┌─────────┐                           ┌─────────┐
│ MySQL   │──┐    ┌──────────────┐    │ 数据仓库 │
└─────────┘  ├───►│              │    └─────────┘
┌─────────┐  │    │  自动化 ETL   │    ┌─────────┐
│ Kafka   │──┤    │  Agent       │───►│ 报表系统  │
└─────────┘  │    │              │    └─────────┘
┌─────────┐  │    │  审批门禁     │    ┌─────────┐
│ S3/FTP  │──┘    └──────────────┘    │ AI 模型   │
└─────────┘                           └─────────┘

自动化 Agent 处理:
1. 增量/全量数据抽取
2. 数据质量检查（规则引擎 + LLM 异常检测）
3. 数据转换与清洗
4. 人工审批：schema 变更、数据量突变
5. 写入目标 + 数据血缘记录
```

### 7.3 业务流程自动化

| 场景 | 自动化前 | 自动化后 |
|------|---------|---------|
| **发票处理** | 人工录入 - 5min/张 | OCR + LLM 提取 - 30s/张 |
| **员工入职** | 跨 8 个系统手动开通权限 | 工作流自动编排 - 5min |
| **订单审核** | 人工检查 20+ 项规则 | 规则引擎 + LLM 异常标记 |
| **合规报告** | 每月 3 天人工整理数据 | Agent 自动采集 + 生成报告 |

---

## 8. 架构演进

### 8.1 演进路线图

```
阶段 1: 定时任务                      阶段 2: 事件驱动
┌─────────────────────┐              ┌─────────────────────┐
│ Cron + Shell Script │              │ 消息队列 + Worker   │
│ • 单机执行           │    ──→      │ • 事件触发           │
│ • 无状态             │              │ • 简单重试           │
│ • 无依赖管理         │              │ • 水平扩展           │
│ • 无监控             │              │ • 基础监控           │
└─────────────────────┘              └─────────────────────┘

阶段 3: DAG 工作流                     阶段 4: 智能自动化
┌─────────────────────┐              ┌─────────────────────┐
│ 专用工作流引擎       │              │ AI Agent 驱动       │
│ • DAG 定义 + 状态机  │    ──→      │ • LLM 动态编排       │
│ • 分布式执行器        │              │ • 自适应异常处理     │
│ • 审批门禁           │              │ • 自然语言定义工作流  │
│ • 完整监控告警        │              │ • 自我优化执行策略   │
└─────────────────────┘              └─────────────────────┘
```

### 8.2 从简单到智能的关键转变

1. **触发方式**：Cron --> 事件驱动 --> 预测性触发（ML 预测需求高峰）
2. **步骤定义**：硬编码 --> 配置化 DAG --> LLM 动态生成
3. **错误处理**：崩溃 --> 重试 --> 自适应策略 --> AI 诊断恢复
4. **资源分配**：固定 --> 弹性伸缩 --> 成本优化调度
5. **人工介入**：每一步 --> 关键审批点 --> 异常时介入
6. **监控维度**：Up/Down --> 性能指标 --> 业务 KPI --> 成本效益

### 8.3 未来趋势

- **自主修复工作流**：当步骤失败时，Agent 自动排查原因并修复
- **多 Agent 协作**：不同专业 Agent 各自负责工作流中的特定领域
- **联邦自动化**：跨组织边界的自动化工作流协作
- **自优化调度**：基于历史执行数据自动优化并行度和资源分配

---

## 参考与扩展阅读

- [Temporal.io](https://temporal.io) - 持久化工作流引擎参考
- [Prefect](https://www.prefect.io) - 数据工作流编排平台
- [Apache Airflow](https://airflow.apache.org) - 批处理工作流引擎
- [AWS Step Functions](https://aws.amazon.com/step-functions/) - 云工作流服务
- **ReAct 模式**：用于 LLM 动态决策的协同框架
