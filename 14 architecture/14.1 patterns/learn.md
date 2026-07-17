# 14.1 架构模式 — Agent Architecture Patterns

> Agent 架构模式描述了 Agent 系统内部的组件组织方式、交互流程和控制流结构。与传统软件架构模式类似，每种 Agent 架构模式都有其特定的适用场景、权衡取舍和演进路径。理解这 6 种模式，能帮助架构师在不同需求下做出合理的设计决策。

---

## 内容结构

| 子主题 | 核心思想 | 适用场景 | 难度 |
|--------|----------|----------|------|
| 14.1.1 事件驱动 Agent | 基于事件触发推理与行动 | 实时响应、异步处理 | ★★★☆☆ |
| 14.1.2 管道架构 | 链式分阶段处理 | 处理流程明确的场景 | ★★★☆☆ |
| 14.1.3 网格架构 | 多 Agent 对等通信 | 复杂协作、研究型系统 | ★★★★★ |
| 14.1.4 微服务 Agent | 独立部署的 Agent 服务 | 大规模生产系统 | ★★★★☆ |
| 14.1.5 单体 Agent | 内聚的单一 Agent 进程 | MVP、快速验证 | ★★☆☆☆ |
| 14.1.6 混合架构 | 场景自适应的组合模式 | 复杂真实系统 | ★★★★★ |

---

## 架构模式的演变图谱

```
传统软件架构演进                          Agent 架构模式演进
─────────────────                         ────────────────────

单体架构 (Monolithic)                     单体 Agent (Simple I/O)
    │                                            │
    ▼                                            ▼
分层架构 (Layered)                         管道架构 (Pipeline)
    │                                            │
    ▼                                            ▼
事件驱动架构 (Event-Driven)                事件驱动 Agent (Reactive)
    │                                            │
    ▼                                            ▼
微服务架构 (Microservices)                 微服务 Agent (Independent)
    │                                            │
    ▼                                            ▼
网格架构 (Service Mesh)                    网格架构 (Agent Mesh)
    │                                            │
    ▼                                            ▼
混合架构 (Hybrid)                          混合架构 (Hybrid)
```

Agent 架构模式并非凭空产生——它们大多是对传统软件架构模式的适应性重构，核心变化在于引入了 **LLM 推理节点** 作为架构的核心处理单元。

---

## 6 种模式总览

### 对比大表

| 维度 | 单体 Agent | 管道架构 | 事件驱动 | 微服务 Agent | 网格架构 | 混合架构 |
|------|-----------|----------|----------|-------------|----------|----------|
| **控制流** | 内部循环 | 线性链 | 事件触发 | 独立循环 | 对等通信 | 自适应切换 |
| **状态管理** | 进程内 | 管道间传递 | 事件中有状态 | 服务内独立 | 分布式共识 | 按模式不同 |
| **Agent 数** | 1 | 1 个逻辑/多阶段 | 1 或多个 | 多个独立 | 多个互连 | 动态变化 |
| **通信模式** | N/A (内部) | 阶段间传递 | Pub/Sub | API/gRPC | 自定义协议 | 混合 |
| **扩展方式** | 垂直扩展 | 阶段级扩缩 | 订阅者扩展 | 独立水平扩展 | 全互联 | 按需扩展 |
| **故障影响** | 整体宕机 | 单阶段阻塞 | 事件丢失 | 服务级隔离 | 局部传播 | 模式依赖 |
| **调试难度** | 容易 | 中等 | 困难 | 中等 | 很困难 | 困难 |
| **适用规模** | 原型/小规模 | 中规模 | 中大规模 | 大规模 | 研究/复杂 | 生产级 |
| **典型 Token 消耗** | 低 | 中 | 中 | 高 (跨服务) | 很高 | 视场景 |
| **冷启动速度** | 秒级 | 秒级 | 秒级 | 分级 (依赖预热) | 分级 | 分级 |

### 架构模式选择热力图

```
                    简单 Agent   多工具 Agent   多 Agent   生产级
                    ─────────   ──────────   ─────────   ──────
单体 Agent             █████       ████░       ██░░░      ██░░░
管道架构               ████░       █████       ███░░      ███░░
事件驱动               ██░░░       ████░       ████░      ████░
微服务 Agent           ██░░░       ████░       █████      █████
网格架构               █░░░░       ██░░░       █████      ████░
混合架构               ██░░░       ████░       █████      █████

█=强烈推荐  █=推荐  █=可用  ░=不推荐
```

---

## 架构模式的本质差异：控制流

6 种模式的核心差异在于 **控制流** 的组织方式，这是架构选型的首要考量。

```
单体 Agent:          ┌──────────────────────┐
                     │  Thought → Action →   │
                     │  ←─ Observation ──    │   (内部循环)
                     └──────────────────────┘

管道架构:            [阶段1] → [阶段2] → [阶段3] → [阶段4]
                     (数据流驱动，阶段间无环)

事件驱动:            ┌── 事件总线 ──┐
                     │              │
                     ▼              ▼
                   [Agent A]    [Agent B]
                   (事件 → 推理 → 行动 → 新事件)

微服务 Agent:        [Agent A] ── API ──► [Agent B]
                     (独立进程，同步/异步通信)

网格架构:            [Agent A] ◄──► [Agent B]
                     [Agent C] ◄──► [Agent D]
                     (全互联/部分互联，对等通信)

混合架构:            根据场景在以上模式间动态切换
```

---

## 架构模式中的公共组件

所有架构模式都共享一些基础组件，区别在于这些组件的组织方式和交互协议。

```
每个 Agent 架构都包含:

┌─────────────────────────────────────────────────────────┐
│  核心组件              │  职责                        │
├─────────────────────────────────────────────────────────┤
│  LLM 推理引擎          │  Token 生成、推理、决策      │
│  Prompt 管理器         │  动态组装 System/User/Tool   │
│  工具注册表            │  工具定义、Schema、发现      │
│  工具执行器            │  调用外部工具、处理结果       │
│  记忆系统              │  短期/长期/工作记忆          │
│  状态管理器            │  会话/任务/Agent 状态        │
│  错误处理器            │  重试/降级/熔断/回退         │
│  可观测性 SDK          │  日志/追踪/指标              │
└─────────────────────────────────────────────────────────┘

不同模式下这些组件如何组合:
  单体: 全部在同一个进程内，方法调用
  管道: 组件分布在管道阶段，通过数据传递
  事件: 组件通过事件总线异步通信
  微服务: 组件分布在独立服务中，通过网络通信
  网格: 每个 Agent 拥有完整组件，Agent 间对等通信
  混合: 按子任务动态组合
```

---

## 伪代码：架构模式的本质差异

下面从**控制流**的角度展示 6 种模式的本质差异：

```
=== 单体 Agent ===
def agent_loop(user_input):
    state = init_state(user_input)
    while not done:
        thought = llm_reason(state)      # 推理
        action = select_action(thought)   # 选择行动
        observation = execute(action)     # 执行
        state = update(state, observation) # 更新状态
    return state.final_answer()

=== 管道架构 ===
def pipeline_agent(user_input):
    parsed = parse_stage(user_input)        # 阶段1: 解析
    planned = plan_stage(parsed)            # 阶段2: 规划
    executed = execute_stage(planned)       # 阶段3: 执行
    verified = verify_stage(executed)       # 阶段4: 验证
    return format_stage(verified)           # 阶段5: 格式化

=== 事件驱动 ===
class EventDrivenAgent:
    def handle_event(self, event):
        thought = self.reason(event)        # 推理事件
        actions = self.decide(thought)      # 决定行动
        for action in actions:
            result = self.execute(action)   # 执行
            if result.should_emit():
                self.emit(Event(result))    # 发射新事件
        return thought.response

=== 微服务 Agent ===
# Service A
@app.post("/agent-a/process")
def agent_a_process(request):
    result_a = agent_a_loop(request.input)
    result_b = call_service_b(result_a)    # 调用服务 B
    return merge_results(result_a, result_b)

# Service B (独立部署, 独立扩缩)
@app.post("/agent-b/process")
def agent_b_process(input_data):
    return agent_b_loop(input_data)

=== 网格架构 ===
class MeshAgent:
    peers: List[MeshAgent]
    
    def process(self, task):
        thought = self.reason(task)
        # 与邻居 Agent 交换中间结果
        peer_results = self.exchange_with_peers(thought)
        # 综合自身推理和邻居信息
        merged = self.merge(thought, peer_results)
        return self.act(merged)
    
    def exchange_with_peers(self, data):
        results = []
        for peer in self.peers:
            result = peer.receive_message(self.id, data)
            results.append(result)
        return self.aggregate(results)

=== 混合架构 ===
class HybridAgent:
    def process(self, task):
        mode = self.route_mode(task)    # 根据任务选择模式
        if mode == 'simple':
            return self.monolithic_process(task)
        elif mode == 'multi_step':
            return self.pipeline_process(task)
        elif mode == 'needs_collaboration':
            return self.mesh_process(task)
        else:
            return self.event_driven_process(task)
```

---

## 架构模式的反模式

### 1. 过早分布式

**症状**: 在只有 1 个 Agent、3 个工具的 MVP 阶段就引入消息队列、服务发现、分布式追踪。

**后果**: 开发效率降低 3-5x，调试复杂度急剧上升。

**正确做法**: 从单体 Agent 开始，当出现以下信号时再进行拆分：
- 单一 Agent 的工具数量 > 15
- 需要独立扩缩不同 Agent 组件
- 团队 > 5 人需要并行开发
- 单一进程的资源瓶颈显现

### 2. 架构冻结

**症状**: 选型后完全不考虑演进，在 1 年后仍然使用最初的架构。

**后果**: 架构与需求日益脱节，技术债累积。

**正确做法**: 每 2-3 个月评估一次架构是否仍然适合当前需求，逐步演进。

### 3. 过度抽象

**症状**: 为了"未来可能需要的灵活性"，引入过多的抽象层 (抽象工厂、策略模式、插件系统)。

**后果**: 代码复杂度增加 2-3x，实际收益为 0。

**正确做法**: YAGNI (You Aren't Gonna Need It)。当确实需要扩展时再增加抽象。

---

## 后续学习

| 子主题 | 核心内容 |
|--------|----------|
| 14.1.1 事件驱动 | 事件总线架构、异步推理、事件溯源 |
| 14.1.2 管道架构 | 分阶段流水线、中间产物传递、并行管道 |
| 14.1.3 网格架构 | 对等通信、信息交换协议、共识机制 |
| 14.1.4 微服务 Agent | 独立部署、服务通信、独立状态 |
| 14.1.5 单体 Agent | 内聚结构、简单循环、适用 MVP |
| 14.1.6 混合架构 | 模式切换、场景路由、自适应组合 |
