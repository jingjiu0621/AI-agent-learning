# 管道架构 — Pipeline Architecture for AI Agents

> **核心思想**：将 Agent 的处理过程分解为多个有序的阶段 (Stage)，每个阶段负责一个特定的处理步骤，数据在阶段之间单向流动。这种架构借鉴了 Unix 管道和传统 ETL 管道的设计哲学——每个阶段做好一件事，阶段之间通过明确的接口通信。

---

## 1. 基本原理

### 1.1 什么是管道架构

管道架构 (Pipeline Architecture) 将 Agent 的处理流程拆解为串行的阶段序列，每个阶段完成特定的子任务，输出作为下一阶段的输入：

```
传统 Agent (ReAct 循环):
  输入 → [Thought → Action → Observation] × N → 输出
         ↑                                      │
         └────────── 每次迭代完整循环 ────────────┘

管道架构 Agent:
  输入 → [理解阶段] → [规划阶段] → [执行阶段] → [验证阶段] → 输出
  
  每个阶段专注一个职责，阶段间单向数据流
  可以并行处理不同输入的相同阶段
```

### 1.2 背景与演进

**之前怎么做**：早期的 Agent 大多采用统一的 ReAct 循环，所有逻辑混合在同一个循环中。这种做法的好处是灵活——Agent 可以自由地在推理和行动之间切换。但问题也随之而来：

```
单一 ReAct 循环的问题:
  ┌─ 所有逻辑混合：推理、工具选择、错误处理混杂在一起
  ├─ 难以优化：无法独立优化特定阶段（如检索、推理）
  ├─ 难以调试：漫长的事件链中定位问题困难
  ├─ 无法并行：流水线各阶段串行执行
  └─ 不可复用：Agent 间的处理逻辑难以共享
```

**核心矛盾**：Agent 的复杂度增长后，统一的 ReAct 循环变成了一个"大泥球"——所有逻辑都在同一个循环中交织。特别是当 Agent 需要执行多种不同类型的操作（检索知识、调用 API、代码执行、多步推理）时，这些操作的处理逻辑、错误处理和优化策略都不同，放在同一个循环中导致代码复杂度指数级增长。

### 1.3 管道架构如何解决

管道架构通过 **责任分离** 来解决上述问题：

```
Stage 1           Stage 2           Stage 3           Stage 4
[输入解析] ────► [推理规划] ────► [工具执行] ────► [输出生成]
     │                │                │                │
  职责:            职责:            职责:            职责:
  • 理解意图      • 制定步骤        • 调用工具        • 格式化结果
  • 提取参数      • 依赖分析        • 执行代码        • 添加引用
  • 上下文整理    • 任务分解        • 处理错误        • 摘要生成
  • 过滤无关      • 策略选择        • 重试逻辑        • 多轮整合
```

**管道架构的关键决策**：
1. **阶段粒度**：每个阶段应该多细？太粗则退化到单体循环，太细则管理成本过高
2. **数据契约**：阶段间传递什么数据结构？必须清晰定义接口
3. **错误传播**：某个阶段失败时是整个管道失败，还是可以降级通过？
4. **并行策略**：哪些阶段可以并行运行？哪些必须串行？

---

## 2. 架构详解

### 2.1 核心架构

```
管道架构的完整内部结构:

┌─────────────────────────────────────────────────────────────────────────┐
│                     管道编排器 (Pipeline Orchestrator)                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ 阶段 1   │─►│ 阶段 2   │─►│ 阶段 3   │─►│ 阶段 4   │─►│ 阶段 5   │      │
│  │ Stage   │  │ Stage   │  │ Stage   │  │ Stage   │  │ Stage   │      │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │            │            │            │            │           │
│  ┌────▼────┐  ┌────▼────┐  ┌────▼────┐  ┌────▼────┐  ┌────▼────┐      │
│  │ Pipeline│  │ Pipeline│  │ Pipeline│  │ Pipeline│  │ Pipeline│      │
│  │ Context │  │ Context │  │ Context │  │ Context │  │ Context │      │
│  │ (共享)  │  │ (共享)  │  │ (共享)  │  │ (共享)  │  │ (共享)  │      │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘      │
│                                                                        │
│  管道上下文 (Pipeline Context): 所有阶段共享的可变状态                   │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ {                                                                 │ │
│  │   "input": "原始用户输入",                                         │ │
│  │   "parsed_intent": "查询销售数据",                                  │ │
│  │   "plan": [{"step": 1, "tool": "query_db", ...}],                 │ │
│  │   "intermediate_results": {"query_db": [...], "analyze": [...]},  │ │
│  │   "errors": [],                                                    │ │
│  │   "final_output": "...",                                          │ │
│  │   "metrics": {"total_tokens": 1500, "steps": 5, "latency": 3.2s} │ │
│  │ }                                                                  │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 代码实现

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional, Callable
from enum import Enum
import asyncio
import time

class StageStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    SKIPPED = "skipped"

@dataclass
class PipelineContext:
    """管道上下文：所有阶段共享的状态"""
    input: Any
    stage_results: Dict[str, Any] = field(default_factory=dict)
    errors: List[Dict] = field(default_factory=list)
    metadata: Dict[str, Any] = field(default_factory=dict)
    config: Dict[str, Any] = field(default_factory=dict)
    
    # 跟踪执行状态
    current_stage: str = ""
    stage_statuses: Dict[str, StageStatus] = field(default_factory=dict)
    start_time: float = field(default_factory=time.time)
    
    def get(self, key: str, default=None):
        """从任意阶段结果或 metadata 中获取值"""
        for result in self.stage_results.values():
            if isinstance(result, dict) and key in result:
                return result[key]
        return self.metadata.get(key, default)
    
    def set(self, key: str, value: Any):
        self.metadata[key] = value

class PipelineStage(ABC):
    """管道阶段抽象基类"""
    def __init__(self, name: str, config: Optional[Dict] = None):
        self.name = name
        self.config = config or {}
    
    @abstractmethod
    async def process(self, ctx: PipelineContext) -> Any:
        """处理管道上下文，返回该阶段的输出"""
        pass
    
    async def __call__(self, ctx: PipelineContext) -> Any:
        ctx.current_stage = self.name
        ctx.stage_statuses[self.name] = StageStatus.RUNNING
        try:
            result = await self.process(ctx)
            ctx.stage_results[self.name] = result
            ctx.stage_statuses[self.name] = StageStatus.SUCCESS
            return result
        except Exception as e:
            ctx.stage_statuses[self.name] = StageStatus.FAILED
            ctx.errors.append({
                "stage": self.name,
                "error": str(e),
                "time": time.time()
            })
            if not self.config.get("optional", False):
                raise  # 非可选阶段失败传播
            return None

class AgentPipeline:
    """Agent 管道编排器"""
    def __init__(self, stages: List[PipelineStage], 
                 error_policy: str = "stop"):
        self.stages = stages
        self.error_policy = error_policy  # "stop" | "skip" | "degrade"
    
    async def run(self, user_input: Any) -> PipelineContext:
        ctx = PipelineContext(input=user_input)
        
        for stage in self.stages:
            try:
                await stage(ctx)
            except Exception as e:
                if self.error_policy == "stop":
                    raise
                elif self.error_policy == "skip":
                    continue  # 跳过失败的阶段
                elif self.error_policy == "degrade":
                    # 降级模式：用默认值代替
                    ctx.stage_results[stage.name] = stage.config.get("fallback", None)
        
        ctx.metadata["total_time"] = time.time() - ctx.start_time
        return ctx

# ============ 具体阶段实现 ============

class InputAnalysisStage(PipelineStage):
    """阶段1: 输入分析"""
    def __init__(self, llm_client):
        super().__init__("input_analysis")
        self.llm = llm_client
    
    async def process(self, ctx: PipelineContext) -> Dict:
        prompt = f"""分析用户输入并提取关键信息：
输入: {ctx.input}

请返回 JSON:
{{
  "intent": "用户意图分类",
  "entities": ["关键实体列表"],
  "complexity": "simple|medium|complex",
  "required_tools": ["需要的工具列表"],
  "context_summary": "上下文摘要"
}}"""
        result = await self.llm.complete(prompt)
        return result

class PlanningStage(PipelineStage):
    """阶段2: 规划"""
    def __init__(self, llm_client):
        super().__init__("planning")
        self.llm = llm_client
    
    async def process(self, ctx: PipelineContext) -> List[Dict]:
        analysis = ctx.stage_results.get("input_analysis", {})
        prompt = f"""基于以下分析制定执行计划：
意图: {analysis.get('intent')}
复杂度: {analysis.get('complexity')}
可用工具: {analysis.get('required_tools')}

请制定详细的步骤计划，每步指定使用什么工具和参数。"""
        
        plan = await self.llm.complete(prompt)
        return plan.get("steps", [])

class ToolExecutionStage(PipelineStage):
    """阶段3: 工具执行"""
    def __init__(self, tool_registry: Dict[str, Callable]):
        super().__init__("tool_execution")
        self.tools = tool_registry
    
    async def process(self, ctx: PipelineContext) -> Dict:
        plan = ctx.stage_results.get("planning", [])
        results = {}
        
        for step in plan:
            tool_name = step.get("tool")
            if tool_name not in self.tools:
                results[tool_name] = {"error": f"工具 {tool_name} 不可用"}
                continue
            
            try:
                result = await self.tools[tool_name](**step.get("params", {}))
                results[tool_name] = {"success": True, "data": result}
            except Exception as e:
                results[tool_name] = {"error": str(e)}
        
        return {"tool_results": results}

class OutputGenerationStage(PipelineStage):
    """阶段4: 输出生成"""
    def __init__(self, llm_client):
        super().__init__("output_generation")
        self.llm = llm_client
    
    async def process(self, ctx: PipelineContext) -> str:
        tool_results = ctx.stage_results.get("tool_execution", {})
        prompt = f"""基于以下工具执行结果生成最终回答：
{json.dumps(tool_results, ensure_ascii=False)}

要求：准确、简洁、完整。"""
        
        final = await self.llm.complete(prompt)
        return final.get("response", "")

# ============ 使用示例 ============

async def main():
    # 构建管道
    pipeline = AgentPipeline([
        InputAnalysisStage(llm_client),
        PlanningStage(llm_client),
        ToolExecutionStage(tool_registry),
        OutputGenerationStage(llm_client)
    ], error_policy="degrade")
    
    # 执行
    ctx = await pipeline.run("查询上季度华东区销售数据并生成分析报告")
    
    # 输出结果
    print(f"最终输出: {ctx.stage_results.get('output_generation')}")
    print(f"执行耗时: {ctx.metadata['total_time']:.2f}s")
    print(f"各阶段状态: {ctx.stage_statuses}")
```

---

## 3. 高级模式

### 3.1 并行管道 (Parallel Pipeline)

某些阶段可以并行执行，提高吞吐量：

```
串行管道:                         并行管道:
                                    ┌─────────────┐
                                    │ 数据分析     │
[输入] ──► [分析] ──► [检索]        │ (gather)    │
    ──► [推理] ──► [输出]          │             │
                                    │ 文档检索 ──► │
                        [输入] ──► │             │──► [合成] ──► [输出]
                                    │ 知识库查询   │
                                    │             │
                                    │ 外部引用     │
                                    └─────────────┘
```

```python
class ParallelPipelineStage(PipelineStage):
    """并行执行多个子阶段，结果合并"""
    def __init__(self, name: str, sub_stages: List[PipelineStage]):
        super().__init__(name)
        self.sub_stages = sub_stages
    
    async def process(self, ctx: PipelineContext) -> Dict:
        # 并行执行所有子阶段
        results = await asyncio.gather(
            *[stage(ctx) for stage in self.sub_stages],
            return_exceptions=True
        )
        
        # 处理结果和异常
        merged = {}
        for stage, result in zip(self.sub_stages, results):
            if isinstance(result, Exception):
                merged[stage.name] = {"error": str(result)}
            else:
                merged[stage.name] = result
        return merged
```

### 3.2 条件分支管道 (Conditional Pipeline)

根据阶段结果决定后续流程：

```
                     ┌─── 简单 ──► [快速回复]
                     │
[输入分析] ──► 复杂度─┼─── 中等 ──► [检索] ──► [推理] ──► [输出]
                     │
                     └─── 复杂 ──► [分解] ──► [多步执行] ──► [验证] ──► [输出]
```

```python
class ConditionalPipelineStage(PipelineStage):
    """根据条件路由到不同的子阶段"""
    def __init__(self, name: str, condition_fn: Callable,
                 if_true: PipelineStage, if_false: PipelineStage):
        super().__init__(name)
        self.condition_fn = condition_fn
        self.if_true = if_true
        self.if_false = if_false
    
    async def process(self, ctx: PipelineContext) -> Any:
        if self.condition_fn(ctx):
            return await self.if_true(ctx)
        else:
            return await self.if_false(ctx)
```

### 3.3 管道中间产物缓存

管道架构天然支持缓存——每个阶段的输出都可以被缓存和复用：

```python
class CachedPipelineStage(PipelineStage):
    """带缓存的管道阶段"""
    def __init__(self, stage: PipelineStage, cache_ttl: int = 3600):
        super().__init__(stage.name)
        self.inner = stage
        self.cache = {}
        self.cache_ttl = cache_ttl
    
    async def process(self, ctx: PipelineContext) -> Any:
        # 根据输入生成缓存键
        cache_key = self._make_cache_key(ctx)
        
        if cache_key in self.cache:
            entry = self.cache[cache_key]
            if time.time() - entry["time"] < self.cache_ttl:
                return entry["value"]
        
        result = await self.inner(ctx)
        self.cache[cache_key] = {
            "value": result,
            "time": time.time()
        }
        return result
    
    def _make_cache_key(self, ctx: PipelineContext) -> str:
        # 简化的缓存键生成
        return f"{self.name}:{hash(str(ctx.input))}"
```

---

## 4. 适用场景与能力边界

### 4.1 最佳适用场景

| 场景 | 原因 | 案例 |
|------|------|------|
| **文档处理** | 明确的阶段：解析→理解→摘要→翻译 | 批量文档摘要系统 |
| **数据 ETL** | 传统 ETL 思维的天然延伸 | 数据清洗和转换 Agent |
| **内容审核** | 审核→过滤→分类→报告 | 内容安全审核流水线 |
| **RAG 流程** | 检索→重排序→合成→验证 | RAG 问答系统 |
| **批处理任务** | 固定流程，高吞吐 | 批量邮件处理 |

### 4.2 不适用场景

| 场景 | 原因 | 替代方案 |
|------|------|----------|
| **自由对话** | 对话流程不可预测，不适合固定阶段 | 事件驱动/单体 Agent |
| **复杂决策** | 需要上下文回溯和动态调整 | ReAct/网格架构 |
| **不确定流程** | 阶段数量和顺序不确定 | 事件驱动 Agent |
| **低延迟交互** | 阶段间传递增加延迟 | 单体 Agent |

### 4.3 能力边界

```
管道架构的核心限制:

┌─────────────────────────────────────────────────────────────┐
│ 限制                       影响                             │
├─────────────────────────────────────────────────────────────┤
│ 固定流程                   难以处理动态变化的流程            │
│ 阶段间耦合                 上游阶段的输出决定下游输入        │
│ 错误恢复                   某阶段失败后重新执行成本高        │
│ 上下文规模                  所有阶段共享的上下文不断增长      │
│ 调试可观测性                需要追踪跨阶段的数据流            │
│ 灵活性不足                  难以实现非线性的推理路径          │
└─────────────────────────────────────────────────────────────┘

特别需要注意的陷阱:
  1. "阶段膨胀": 每个阶段太复杂 → 退化到单体循环
  2. "上下文泄漏": 下游阶段依赖了上游不应该暴露的内部细节  
  3. "错误沉默": 某个阶段静默失败，下游在错误基础上继续
  4. "过度拆分": 阶段太多，管理开销 > 收益
```

---

## 5. 工程优化方向

| 方向 | 方法 | 效果 |
|------|------|------|
| **阶段级缓存** | 缓存每个阶段的输出 | 相同输入重复处理加速 10-100x |
| **动态阶段路由** | 根据输入动态选择阶段的组合 | 处理简单请求降级 50% Token |
| **并行执行** | 独立阶段并行执行 | 总延迟降低 30-60% |
| **阶段级重试** | 每个阶段独立的重试策略 | 整体成功率提升 |
| **流式输出** | 前阶段输出立即传递到下阶段 | 首 token 延迟降低 |
| **阶段池化** | 预初始化阶段实例 | 减少冷启动延迟 |
