# 6.1.1 context-budget — 上下文预算跟踪与分配

## 简单介绍

上下文预算（Context Budget）是一个动态跟踪和分配 Token 空间的管理机制。它的核心任务是在有限的上下文窗口中，决定哪些内容应该保留、占多大空间、以及什么时候需要释放空间。如同操作系统管理物理内存，上下文预算管理 Agent 的"推理内存"。

## 基本原理

### 预算模型

上下文预算的核心是一个 Token 账本：

```
总上下文窗口: C_total (如 128K tokens)
系统预留:     C_reserved (系统指令 + 工具定义，约 5K-50K)
响应预留:     C_response (模型回复需要的空间，约 2K-8K)
可用预算:     C_available = C_total - C_reserved - C_response

实际使用:     C_used (当前对话 + 检索内容)
剩余空间:     C_remaining = C_available - C_used
```

### 按角色/优先级分配策略

```
Token 预算分配层次:

Layer 0 — 不可裁剪的系统指令（System Prompt）
  ├── 身份/行为定义 (高优先级)
  ├── 安全约束 (最高优先级, 不可移除)
  └── 核心能力声明

Layer 1 — 当前交互上下文
  ├── 最近的用户消息 (高优先级)
  ├── 最近的助手回复
  ├── 最近的工具调用结果
  └── 待处理的推理链

Layer 2 — 检索增强内容
  ├── 与当前查询直接相关的文档 (中优先级)
  ├── 用户历史偏好
  └── 参考示例

Layer 3 — 历史对话记录
  ├── 对话摘要 (比完整对话高效)
  ├── 重要决策节点
  └── 用户长期偏好
```

### 动态预算分配算法（伪代码）

```
class ContextBudget:
    def __init__(self, total_window, system_prompt_tokens, tools_tokens):
        self.total_budget = total_window
        self.reserved = system_prompt_tokens + tools_tokens + RESPONSE_BUFFER
        self.available = self.total_budget - self.reserved
        self.allocations = {
            'system': system_prompt_tokens + tools_tokens,
            'recent_history': 0,
            'retrieved_context': 0,
            'user_input': 0,
        }

    def can_fit(self, tokens: int) -> bool:
        """检查是否能容纳额外内容"""
        return self.used() + tokens <= self.total_budget - self.reserved

    def allocate(self, content_type: str, tokens: int) -> bool:
        """尝试分配空间，返回是否成功"""
        if not self.can_fit(tokens):
            self._evict(content_type)
        if self.can_fit(tokens):
            self.allocations[content_type] += tokens
            return True
        return False

    def _evict(self, incoming_type: str):
        """按优先级驱逐已有内容"""
        priorities = ['retrieved_context', 'history_summary', 'old_history']
        for target in priorities:
            if self.allocations.get(target, 0) > 0:
                self.allocations[target] = 0
                if self.can_fit(0):
                    break

    def utilization(self) -> float:
        return self.used() / self.total_budget
```

## 背景与演进

### 早期做法：静态窗口

最早的 Agent 实现通常不做预算管理——直接把所有内容塞进上下文，超出就报错或粗暴截断。

**问题：**
- 无法预测什么时候会超限
- 截断时不知道丢了什么关键信息
- 没有优先级概念，同等对待所有内容

### 演进路径

```
静态窗口 (hard limit) → 简单计数器 (累计 Token) → 分层预算 (按角色) → 动态预算 (自适应分配)
       v                      v                         v                        v
  超限即报错             截断尾部            重要内容优先保留           按场景动态调整
```

### 当前状态

业界主流方案是**分层+动态混合模式**：用固定配额保证系统指令等关键内容，用动态分配处理可变部分（如检索内容、历史记录）。

## 核心矛盾

**精度 vs 利用率**：预算划分越精细，空间利用率越高，但管理开销也越大。反之，粗粒度的划分虽然简单，但容易浪费空间或在关键内容上分配不足。

具体表现为三个权衡：
1. **预留 vs 灵活**：为响应预留的空间越多，模型越安全，但可用空间越少
2. **优先级粒度**：分的层级越多，保留下来的内容越合理，但调度算法越复杂
3. **分配时机**：预先分配（前置决定）vs 动态分配（运行时决定）

## 主流优化方向

| 方向 | 描述 | 效果 |
|------|------|------|
| **比例预算** | 按固定比例划分各类型内容的配额（如系统:30%, 历史:40%, 检索:30%） | 简单可靠，但不够灵活 |
| **优先级队列** | 每段内容标记优先级，低优先级内容在空间不足时被驱逐 | 灵活但有额外标记成本 |
| **压力感知分配** | 监控当前预算压力（使用率），压力高时自动降低低优先级配额 | 自适应，实现复杂 |
| **机会主义注入** | 只有在有"空闲"空间时才注入非关键内容（如检索结果） | 最大化利用，但可能随机失效 |
| **Token 货币化** | 每个内容段赋予"价值/Token"比，保留性价比最高的内容 | 理论最优，但价值评估困难 |

## 产生的结果

- **可预测性**：Agent 不会意外超出上下文窗口
- **空间效率**：同等等级下可用交互空间提升 20%-50%
- **内容质量**：高优先级内容保留率显著提高
- **管理开销**：预算管理本身消耗约 0.1%-1% 的总 Token（预算信息自身的存储）

## 最大挑战

**预算的"自指"问题**：预算管理自身的描述（它要管理哪些区域、当前使用量等）也需要占用上下文空间。当上下文极度紧张时，管理者自身的信息也可能被截断，导致预算管理失效。

解决方案通常是将预算元数据放在不可裁剪的 Layer 0 区域，或用外部存储跟踪预算状态。

## 能力边界与结果边界

| 维度 | 能做的 | 不能做的 |
|------|--------|----------|
| 分配 | 按优先级分配 Token 配额 | 无法创造额外的 Token 空间 |
| 预测 | 可以预测当前会话能否容纳新内容 | 无法预测未来用户消息的长度 |
| 保护 | 可以保护关键系统指令不被截断 | 无法保证所有高优先级内容共存 |
| 优化 | 空间利用率提升 20-50% | 无法解决本质上的窗口限制 |

## 与相关概念的区别

| 概念 | 区别 |
|------|------|
| **Token 计数 (6.1.3)** | 预算管理关注"分配"，Token 计数关注"测量"——先有精确计数，才有合理分配 |
| **截断策略 (6.1.4)** | 预算管理在上游做预防性分配，截断在下游做被动处理。好预算可以减少截断 |
| **内存管理 (OS概念)** | 虚拟内存可以换页（page out / swap），Agent 上下文无法——窗口外的内容完全不可见 |

## 核心优势

1. **主动管理而非被动应付**：在内容注入前就判断是否放得下，而不是满了再处理
2. **优先级感知**：确保关键内容（安全指令、核心配置）不被非关键内容挤占
3. **场景适应**：根据当前任务类型调整预算分配策略（如编码任务 vs 对话任务）

## 工程优化建议

### 实现建议

1. **分层计数器**：维护一个四层（Layer 0-3）的 Token 计数器，每层独立跟踪
2. **预留缓冲**：总窗口的 10%-15% 作为不可动用的响应缓冲
3. **预算水位线**：设置 70%（警告）、85%（紧急）、95%（临界）三个水位线，触发不同级别的清理
4. **异步预算审计**：每 N 轮对话执行一次完整的预算使用分析，识别浪费模式

### 伪代码实现要点

```python
# 分布式预算监控
class BudgetMonitor:
    WARNING_LEVEL = 0.70
    CRITICAL_LEVEL = 0.85
    EMERGENCY_LEVEL = 0.95

    def get_budget_status(self) -> str:
        ratio = self.current / self.total
        if ratio >= self.EMERGENCY_LEVEL:
            return "EMERGENCY"
        elif ratio >= self.CRITICAL_LEVEL:
            return "CRITICAL"
        elif ratio >= self.WARNING_LEVEL:
            return "WARNING"
        return "NORMAL"

    # EMERGENCY 时：立即触发截断，丢弃所有非关键检索内容
    # CRITICAL 时：不再接受新的检索注入，开始压缩历史
    # WARNING 时：对新的非关键内容进行选择性接受
```

## 适合场景判断

| 场景 | 适用性 | 说明 |
|------|--------|------|
| 长对话 Agent | 强制 | 没有预算管理就无法运行多轮对话 |
| 工具密集型 Agent（50+ 工具） | 强制 | 工具定义本身就会占用大量空间 |
| 单轮快速问答 | 可选 | 简单场景不超限就不需要 |
| 流式响应 | 推荐 | 分段输出需要预留足够的响应空间 |
| RAG Agent | 强制 | 检索内容的不确定性需要动态预算调节 |
| 多 Agent 协作 | 强制 | 每个 Agent 的上下文都需要独立预算跟踪 |
