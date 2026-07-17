# 12.3.3 Boundary Checking — 边界检查

## 简单介绍

边界检查（Boundary Checking）是 AI Agent 输入验证中的第二道防线——在确认输入的"语法"正确后（Schema Validation + Type Checking），检查输入是否在**可安全处理的范围内**。Agent 面临的核心约束是有限窗口（LLM context window）和有限算力（推理成本、延迟预算）。边界检查的任务是为这些有限资源设定安全阈值，拒绝那些超出 Agent 处理能力的极端输入。

与传统 API 的边界检查不同，Agent 需要关注七类边界：**Token 预算（Token Budget）**、**长度限制（Length Boundaries）**、**范围限制（Range Boundaries）**、**时间限制（Time Boundaries）**、**资源限制（Resource Boundaries）**、**层级限制（Hierarchical Boundaries）** 和**自适应边界（Adaptive Boundaries）**。

```
Agent 输入验证四层模型——边界层的位置
原始输入
    │
    ▼
┌──────────────┐
│ 语法层        │ ← Schema 验证、类型检查、格式校验
│ Syntax Check  │   此层关注"是否符合格式规范"
└──────┬───────┘
       ▼
┌──────────────┐
│ 边界层        │ ← 长度/范围/数量/频率限制  ← 我们在这里
│ Boundary     │   此层关注"是否在可处理范围内"
└──────┬───────┘
       ▼
┌──────────────┐
│ 编码层        │ ← Unicode 归一化、编码检测、混淆识别
│ Encoding     │
└──────┬───────┘
       ▼
┌──────────────┐
│ 内容层        │ ← 安全检测、注入检测、策略执行
│ Content      │
└──────┬───────┘
       ▼
  安全输入 → Agent 处理
```

边界检查的核心原则：**在输入进入 LLM 上下文窗口和工具执行环境之前，确定其是否在 Agent 可安全处理的资源预算范围内。**

---

## 基本原理 — Input Boundaries Prevent Resource Exhaustion, DoS, and Exploitation via Extreme Inputs

边界检查的防御目标有三层：

### 1. 防止资源耗尽（Resource Exhaustion）

LLM 推理的计算成本与输入长度大致呈线性关系（忽略 attention 的二次复杂度窗口内），但超出窗口的行为（截断、降级）会导致用户体验崩溃。边界检查确保 Agent 永远不会接收到超出其处理能力的输入：

```
资源耗尽链式反应：
极端输入
  → Token 超限 → 上下文截断/溢出
  → 推理成本飙升（按 Token 计费的 LLM API）
  → 响应延迟不可接受
  → Agent 线程阻塞/崩溃
  → 级联故障影响其他用户/服务
```

### 2. 防止拒绝服务攻击（DoS）

Agent 系统的 DoS 攻击面远大于传统 API：

| DoS 攻击向量 | 传统 API | AI Agent |
|-------------|----------|----------|
| 请求体大小 | 受 HTTP 限制 | Token 超长可能触发窗口溢出 |
| 请求频率 | 受 Rate Limiter 限制 | 每个请求都可能触发多次 LLM 推理（工具调用循环） |
| 并发工具调用 | 有限制 | Agent 可能并行调用数十个工具 |
| 文件上传 | 大小限制 | LLM 需处理文件内容，Token 成本急剧上升 |
| 复杂推理链 | 不适用 | 单个请求可能触发数十轮推理 |

Agent 的 DoS 独特风险：**一个精心构造的输入可以触发 Agent 进入"推理循环"——反复调用工具、递归分析结果、无限延伸对话**，消耗远超单次 API 调用的资源总量。

### 3. 防止通过极端值进行利用（Exploitation via Extreme Inputs）

极端输入不仅是"太多了"的问题——攻击者可以利用边界漏洞进行更精密的攻击：

- **Token 预算耗尽** → 系统提示被截断 → 安全边界声明丢失 → Agent 失去防御指令
- **上下文窗口溢出** → Agent 忘记之前的拒绝决策 → 攻击者重复提交绕过尝试
- **工具输出过度** → 注入内容在截断的边界处被"挤入"Agent 的指令处理区域
- **数值溢出** → 整数溢出导致权限检查通过（如 `max_retries = -1` 被解释为无限重试）

边界检查的原理本质上是**有限资源下的前置风险管理**——在资源分配之前，先确认请求在安全预算之内。

---

## 背景 — How Unbounded Inputs Cause Failures

### Context Overflow（上下文溢出）

LLM 上下文窗口是 Agent 最硬的约束。当输入超出窗口时，不同系统的行为不同：

```
模型           窗口大小    超限行为
─────────────────────────────────────────────────
GPT-4-Turbo    128K       静默截断最早的内容（滑动窗口）
Claude 3.5     200K       截断中间部分（保留头和尾）
Gemini 1.5     2M         仍可处理但质量下降，延迟飙升
开源模型       4K-32K     直接报错或截断
```

对 Agent 而言，上下文溢出导致的最严重问题不是"丢了一些早期对话"——而是**系统提示被截断**。系统提示通常位于上下文的最前端或最后端，取决于 LLM 的实现。如果系统提示中的安全边界声明被截断，Agent 就失去了防御层。

### Token Exhaustion（Token 耗尽）

商业 LLM 按 Token 计费。无边界控制的输入会导致：

```
一次失控的 Agent 交互产生的 Token 消耗：
正常交互:    ~5,000  tokens 输入 + ~500  tokens 输出 = ~$0.03 (GPT-4)
             单轮对话，1-2 次工具调用

失控交互:  ~200,000  tokens 输入 + ~50,000  tokens 输出 = ~$3.00 (GPT-4)
             多轮工具调用，Agent 陷入循环
             
攻击成本:  攻击者只需付出极少的构造代价
           Agent 端需要支付 100x 以上的 Token 费用

不对称性:  攻击成本 << 防御成本（但此处是攻击引发 Agent 消耗资源）
```

一种经典的"经济 DoS"攻击：攻击者构造一个看起来合理的复杂请求（"分析这 500 页文档中所有与 XX 相关的引用，调用搜索引擎验证每个引用"），Agent 每执行一步就消耗大量 Token，攻击者只需要支付一次请求的费用。

### Memory DoS（内存级拒绝服务）

当 Agent 处理超大输入时，不仅有 LLM API 成本，还有系统层面的内存压力：

```
无边界控制的 Agent 内存消耗线：
输入大小      内存消耗      典型问题
─────────────────────────────────────────
10KB           ~50MB       正常
1MB            ~500MB      可能触发 OOM（大文件解析）
10MB           ~5GB        几乎一定会 OOM（大图/长文档 OCR）
100MB          ~50GB       服务不可用
```

注意：这里的内存消耗不仅仅是 LLM 推理——还包括输入预处理、文本提取、Embedding 生成、向量检索等前置步骤。一个 100MB 的 PDF 在进入 LLM 之前，可能需要先 OCR（消耗额外内存）、分块（创建中间表示）、向量化（加载 Embedding 模型）。

## 真实案例分析

### Case 1：Sliding Window 截断导致的防御失效

某 Agent 使用滑动窗口截断来处理超长对话。系统提示在窗口的"最前面"，用户输入在"最后面"，中间是对话历史。当对话历史增长时，最早的内容被截断——但系统提示从未被截断。看起来安全。

问题：当单次用户输入就大到填满整个窗口时（例如用户粘贴了一整本书），滑动窗口会从"最旧的非系统内容"开始截断——**把系统提示之后的所有旧内容截掉，把用户的新内容放进来**。系统提示保留，但 Agent 的"记忆"被清空——Agent 忘记了自己之前做了什么、决定了什么。如果攻击者分多步诱导 Agent，上一步 Agent 已经同意执行某个操作，下一步 Agent 的输入导致窗口刷新——上一步的拒绝决策被清空。

### Case 2：工具输出注入 + Token 预算耗尽

攻击者构造一个包含大量合法内容和少量注入指令的文档，让 Agent 处理该文档：

1. Agent 开始处理文档（消耗大量 Token）
2. 文档中的注入指令在 Agent 窗口内出现，但由于窗口空间已经被文档内容占满，系统提示的安全边界被"挤出"窗口
3. Agent 丢失了安全约束，执行了注入指令中的操作

边界检查的关键洞察：**Token 不仅仅是成本问题——它是 Agent 的"安全生存空间"。超过预算的输入可能挤压安全声明所在的上下文区域。**

---

## 核心矛盾 — Over-Restrictive Boundaries Limit Legitimate Use Cases

边界检查面临的根本矛盾：**适当放开边界是 Agent 能力的来源，收紧边界是安全性的保障。**

| 矛盾方面 | 宽松边界的优势 | 严格边界的代价 |
|---------|--------------|--------------|
| **长文档处理** | 律师分析千页合同、研究者综述百篇论文 | 限制 10KB → 用户无法上传完整文档 |
| **复杂任务** | 多步推理解决复杂问题、跨工具协作 | 限制 5 步 → 无法完成任何有意义的任务 |
| **大数据分析** | 处理百万行 CSV、批量数据转换 | 限制 100 行 → 数据密集型任务不可用 |
| **长时间会话** | 持续上下文支持复杂项目 | 限制 10 轮 → 交互浅薄 |
| **高端用户需求** | 按需扩大 Token 预算（付费用户） | 统一定额 → 高端用户体验不佳 |

### 具体矛盾场景

**场景：法律文档分析**

```
律师上传一份 500 页的合同（~750K tokens）。
Agent 的 Token 预算：200K tokens

过度严格的选择：    拒绝处理（"文档超出长度限制"）
                   → 律师无法使用 Agent
适度放行的选择：    接受文档，但需要分块处理
                   → 代理需要多轮交互（逐段分析）
                   → 可能失去跨页面的全局上下文
风险敞口：          如果 Agent 的分块逻辑存在缺陷，
                   注入指令可能在被分块时逃逸检测
```

**场景：多工具协作任务**

```
"帮我比较这三份简历，然后搜索各公司的最新动态，
最后生成一封汇总邮件给 HR 总监"

宽松边界：Agent 可以自由执行所有步骤
          但可能陷入"搜索→阅读→搜索→阅读"的死循环
          （工具调用的 Token 消耗难以预估）

严格边界：限制工具调用次数 ≤ 5 次
          如果 5 次不够，Agent 可能在最后一封邮件
          中草草了事，输出不完整的分析
```

### 矛盾的本质

边界检查的"度"不是一个技术问题，而是一个**产品决策**：

```
边界严格度 = f(场景安全要求, 用户信任度, 任务复杂度, 成本预算)

同一个 Agent：
  - 对免费用户 → 严格边界（限制 5 次工具调用 + 10K tokens 输入）
  - 对付费用户 → 宽松边界（限制 50 次工具调用 + 100K tokens 输入）
  - 对内部管理员 → 更宽松的边界（需要额外的审计追踪）
```

核心矛盾无法通过技术手段消除，但可以通过**自适应边界**和**分级策略**来缓解——下文 7 种边界类型中的"自适应边界"专门针对此矛盾设计。

---

## 详细内容

### 1. Token Budget Management — Token 预算管理

Token 是 Agent 最核心的有限资源。Token 预算管理是边界检查的第一步，也是最重要的一步。

#### 1.1 输入 Token 限额计算

一个 Agent 请求的总 Token 消耗由多个部分组成：

```
Agent 请求总 Token = 
    系统提示 (System Prompt Token)  
    + 用户输入 (User Input Token)
    + 工具定义 (Tool Definitions Token)
    + 对话历史 (Conversation History Token)
    + 工具调用 (Tool Call Token)
    + 工具返回 (Tool Result Token)
     + 预留空间 (Reserved for Output Token)
    ────────────────────────────────────
    合计 ≤ 模型上下文窗口大小
```

实际计算中的分配策略：

```python
def calculate_token_budget(config: AgentConfig) -> TokenBudget:
    """
    计算每次 Agent 请求的 Token 分配方案

    分配原则：
    1. 系统提示不可压缩（必须完整保留）
    2. 工具定义不可压缩（缺少定义会导致调用失败）
    3. 输出预留必须充足（否则 Agent 在输出中途被截断）
    4. 用户输入和对话历史是可压缩的
    """
    window_size = config.model_context_window     # 模型上下文窗口总大小
    system_prompt = count_tokens(config.system_prompt)
    tool_definitions = sum(count_tokens(t) for t in config.tools)
    output_reserve = window_size * 0.2           # 预留 20% 给输出
                                                  # 过低 → 输出被截断
                                                  # 过高 → 输入空间不够

    available_for_input = (window_size
                           - system_prompt
                           - tool_definitions
                           - output_reserve)

    history_reserve = available_for_input * 0.3  # 预留 30% 给对话历史
    user_input_budget = available_for_input * 0.7 # 用户输入 + 工具返回

    return TokenBudget(
        total=window_size,
        system_prompt=system_prompt,
        tool_definitions=tool_definitions,
        output_reserve=output_reserve,
        conversation_history=history_reserve,
        user_input=user_input_budget,
    )
```

#### 1.2 按来源分配预算

不同来源的输入应共享 Token 预算池，但各有独立的软限制：

| 来源 | 预算分配 | 超限处理 |
|------|---------|---------|
| 系统提示 | 固定（不可调整） | 必须能放入窗口，否则系统配置有误 |
| 工具定义 | 固定（按注册工具计算） | 超限 → 禁止注册新工具 / 卸载不常用工具 |
| 用户输入 | 动态（按任务类型调整） | 超限 → 截断、分块或拒绝 |
| 对话历史 | 动态（LRU 淘汰） | 超限 → 丢弃最早的非关键历史 |
| 工具调用结果 | 动态（按次数累积） | 超限 → 截断结果、限制调用次数 |
| 输出预留 | 固定比例 | 不足 → Agent 生成被截断，回复不完整 |

#### 1.3 系统提示预留

系统提示是 Agent 的"宪法"——它的完整性比任何用户输入都重要。边界检查必须确保系统提示永远不会被截断：

```
安全预留规则：
1. Token 预算计算的第 0 步：确保系统提示总长度 < 窗口大小 × 50%
   （窗口一半以上被系统提示占用 → 说明系统提示过大）
2. 如果工具定义导致系统提示被挤压：应截断工具描述（而非系统提示）
3. 如果用户输入导致系统提示被挤压：应拒绝用户输入（而非系统提示）
4. 输出预留也不可挪用：输出被截断的 Agent 可能产出不完整/不安全的回复
```

---

### 2. Length Boundaries — 长度边界

长度边界是边界检查中最直观的约束——限制输入的长度属性。

#### 2.1 字段级长度限制

| 字段类型 | 检查维度 | 典型上限 | 检查时机 |
|---------|---------|---------|---------|
| 单行文本 | 字符数 | 500 chars | 输入时 |
| 多行文本 | 字符数 / 行数 | 5000 chars / 100 lines | 输入时 |
| 文档上传 | 字符数 / 页数 | 100K chars / 200 pages | 上传时 |
| 工具参数 | 序列化长度 | 10K chars | 工具调用前 |
| 工具返回 | 序列化长度 | 50K chars | 工具返回时 |
| 文件附件 | 原始字节 + 解码后字符 | 10MB / 100K tokens | 上传时 |
| 对话轮次 | 轮数 | 100 轮 | 每轮运行时 |

#### 2.2 总输入长度限制

```
总输入长度 = 用户当前输入 + 历史上下文中的用户消息

检查逻辑：
if total_input_tokens > user_input_budget:
    if can_truncate_history:
        history = truncate_to_fit(history, user_input_budget - current_input)
        if len(history) == 0 and total_input_tokens > budget:
            reject("输入超出 Agent 处理能力")
    else:
        reject("输入超出当前会话的 Token 预算")
```

#### 2.3 上下文窗口限制（Context Window Cap）

这是 Agent 的"硬天花板"——任何情况都不能突破。上下文窗口限制是所有其他边界计算的基础：

```
必选检查（所有 Agent 请求必须通过）：

assert total_tokens < model.context_window * 0.95
# 保留 5% 余量，防止 Token 计数误差导致溢出

if total_tokens >= model.context_window * 0.95:
    # 紧急截断：丢弃对话历史（从最旧的开始）
    while total_tokens >= model.context_window * 0.9 and history:
        history.pop(0)
        total_tokens = recalculate()

    # 如果连系统提示 + 当前用户输入就超限
    if total_tokens >= model.context_window * 0.95:
        reject("请求过大，请简化问题或减少附件")
```

---

### 3. Range Boundaries — 范围边界

范围边界适用于有数值/数量属性的输入维度。

#### 3.1 数值范围（Numeric Min/Max）

Agent 工具调用中数值参数的边界检查：

```python
class NumericRangeBoundary:
    """数值范围边界——检查工具调用中的数值参数"""

    BOUNDARIES = {
        "temperature": {"min": 0.0, "max": 2.0},
        "max_tokens": {"min": 1, "max": 8192},
        "top_p": {"min": 0.0, "max": 1.0},
        "frequency_penalty": {"min": -2.0, "max": 2.0},
        "presence_penalty": {"min": -2.0, "max": 2.0},
        "retry_count": {"min": 0, "max": 10},
        "timeout_seconds": {"min": 1, "max": 300},
        "batch_size": {"min": 1, "max": 100},
    }

    def check(self, param_name: str, value: float) -> RangeCheckResult:
        if param_name not in self.BOUNDARIES:
            return RangeCheckResult.pass_()  # 未注册参数默认通过

        bounds = self.BOUNDARIES[param_name]
        if value < bounds["min"] or value > bounds["max"]:
            return RangeCheckResult.reject(
                param=param_name,
                value=value,
                reason=f"超出范围 [{bounds['min']}, {bounds['max']}]"
            )
        return RangeCheckResult.pass_()
```

**特殊关注：整数的边界溢出**

```python
# Python 中整数无溢出问题，但和外部系统交互时尤其要注意
# C-like 系统的溢出可被利用：

def check_integer_safety(tool_input: dict, schema: dict) -> bool:
    """检查整数参数是否有边界安全问题"""
    for field, definition in schema.get("properties", {}).items():
        if definition.get("type") == "integer":
            value = tool_input.get(field)
            if value is None:
                continue

            # 最小边界检查（阻止负值导致的意外行为）
            if definition.get("minimum", 0) < 0:
                # 允许负值的参数（如 offset=-1 可能表示"全部"）
                # 需要特别审查业务逻辑含义
                pass

            # 最大边界检查（防止整数溢出或内存耗尽）
            max_val = definition.get("maximum", 2**31 - 1)
            if value > max_val:
                return False  # 拒绝

            # 零值检查（某些系统将 0 解释为"无限制"）
            if value == 0 and definition.get("minimum", 0) > 0:
                return False  # 最小值大于 0 时 0 一定非法

    return True
```

#### 3.2 数组长度（Array Length）

Agent 工具调用中数组参数（批量操作）的长度控制：

| 数组参数 | 典型长度上限 | 理由 |
|---------|------------|------|
| 批量删除 ID 列表 | 100 | 防止误删大量数据 |
| 批量发送消息目标 | 50 | 防止垃圾邮件 |
| 查询参数列表 | 20 | 防止复杂查询导致数据库压力 |
| 文件路径列表 | 10 | 防止文件系统过载 |
| Embedding 向量批量 | 100 | API 配额限制 |

```python
def check_array_length(value: list, schema: dict) -> bool:
    """检查数组参数的长度是否在安全范围内"""
    max_items = schema.get("maxItems", 100)  # 默认 100
    min_items = schema.get("minItems", 0)

    if len(value) < min_items:
        return False
    if len(value) > max_items:
        return False
    return True
```

#### 3.3 嵌套深度（Nesting Depth）

Agent 处理的结构化数据（JSON、XML、文档树）的嵌套深度限制：

```
嵌套深度攻击示例：

合法请求:
{"query": "find users", "filters": {"age": {">": 18}}}

嵌套攻击请求:
{"query": "find users", "filters": { ... 1000层嵌套 ... }}
→ 递归解析时栈溢出
→ JSON 解析器崩溃
→ Agent 服务不可用

防御：限制结构化输入的最大嵌套深度
典型深度上限：JSON → 20 层, XML → 32 层, 文档标题 → 6 级
```

```python
def check_nesting_depth(obj, max_depth=20, current_depth=0):
    """递归检查嵌套深度"""
    if current_depth > max_depth:
        raise BoundaryViolation(f"嵌套深度 {current_depth} 超过上限 {max_depth}")

    if isinstance(obj, dict):
        for v in obj.values():
            check_nesting_depth(v, max_depth, current_depth + 1)
    elif isinstance(obj, list):
        for item in obj:
            check_nesting_depth(item, max_depth, current_depth + 1)

    return True
```

---

### 4. Time Boundaries — 时间边界

时间边界控制 Agent 操作的频率和持续时间。

#### 4.1 请求频率（Request Frequency）

```
用户级频率限制：
  ┌────────────────────────────────────────┐
  │ 用户 A 过去 60s 内请求数: 127          │
  │ 限制: 60 req/min                       │
  │ 状态: ❌ 超出限制                       │
  │ 处理: 返回 429 + Retry-After: 23s       │
  └────────────────────────────────────────┘
```

Agent 特有的频率限制挑战：

| 频率指标 | 与传统 Rate Limit 的区别 | 典型限制 |
|---------|------------------------|---------|
| 用户请求频率 | 相同——按用户计费控制 | 60 req/min |
| Agent 推理频率 | 每个用户请求可能触发多次推理 | 10 次推理/请求 |
| 工具调用频率 | 单个请求内的工具调用节奏控制 | 30 calls/min |
| 外部 API 调用频率 | Agent 代表用户调用第三方 API | 需遵守第三方限额 |
| Token 消耗速率 | Token/min —— 突发大请求消耗大量 Token | 根据预算实时计算 |

Agent 多步推理中的频率控制尤为重要——一个请求可能触发 `1:N` 的推理调用：

```
用户 1 个请求
  → Agent 第 1 次推理（理解任务）
  → Agent 第 2 次推理（决定调用工具 A）
  → 工具 A 返回结果
  → Agent 第 3 次推理（分析工具 A 的结果）
  → Agent 第 4 次推理（决定调用工具 B）
  → 工具 B 返回结果
  → Agent 第 5 次推理（生成最终回复）
  → 1 个用户请求 = 5 次推理 + 2 次工具调用
```

#### 4.2 工具调用速率（Tool Call Rate）

单个 Agent 推理循环中的工具调用速率控制：

```python
class ToolCallRateController:
    """
    控制单个 Agent 请求内工具调用的速率和数量
    防止 Agent 在单次请求中发出过多的工具调用
    """

    def __init__(self, max_calls_per_request=20, cooldown_seconds=1.0):
        self.max_calls_per_request = max_calls_per_request
        self.cooldown_seconds = cooldown_seconds
        self.call_count = 0
        self.last_call_time = 0.0

    def can_call_tool(self, tool_name: str) -> bool:
        """检查是否可以执行工具调用"""
        # 1. 总调用次数限制
        if self.call_count >= self.max_calls_per_request:
            return False  # "当前请求的工具调用次数已达上限"

        # 2. 调用间隔限制（防止短时间内大量调用）
        now = time.time()
        if now - self.last_call_time < self.cooldown_seconds:
            return False  # "工具调用过于频繁，请稍后再试"

        return True

    def record_call(self, tool_name: str):
        """记录一次工具调用"""
        self.call_count += 1
        self.last_call_time = time.time()
```

#### 4.3 会话持续时间（Session Duration）

Agent 会话的生命周期管理：

```
会话类型     默认超时    超时后行为
──────────────────────────────────────────
单次请求     5-30 min    超时 → 返回已收集的部分结果 / 失败
多轮对话     1-24 hr     超时 → 保存对话摘要 / 提示用户重启
后台任务     1-72 hr     超时 → 发送状态通知 / 清理资源
长连接推送   持续         心跳检测 → 断连后清理上下文
```

---

### 5. Resource Boundaries — 资源边界

Agent 资源边界是传统 API 中没有的独特维度——Agent 可以调用工具、处理文件、分配算力。

#### 5.1 并发工具调用（Concurrent Tool Calls）

工具调用的并发控制防止 Agent 在同一时间发起过多外部请求：

```
无并发控制的 Agent 工具调用：
Agent 推理 → "同时调用搜索工具、数据库查询、文件读取"
         ↓
    ┌────┴────┐
    │ 搜索 API │ (耗时 2s)
    │ 数据库   │ (耗时 5s)  → 三者并行 → 最快 2s 完成
    │ 文件 I/O │ (耗时 0.1s) → 但系统同时承受 3 个外部压力
    └─────────┘

有并发控制的 Agent 工具调用：
Agent 推理 → "同时调用搜索、数据库、文件读取"
         ↓
    ┌────┴────┐
    │ 并发限制: 2 ↑ 只能同时发起 2 个外部调用
    │ 文件 I/O → 立即执行（瞬时操作）
    │ 搜索 API → 立即执行（较快）
    │ 数据库 → 排队等待，搜索完成后执行（较慢）
    └─────────┘
```

常见并发限制策略：

| 策略 | 说明 | 适用场景 |
|-----|------|---------|
| 全局并发限制 | 整个 Agent 系统最多 N 个并发工具调用 | 共享资源池 |
| 按用户限制 | 每个用户最多 N 个并发工具调用 | 多租户系统 |
| 按工具限制 | 每个工具最多 N 个并发调用 | 受外部 API 配额限制 |
| 优先级队列 | 高优先级任务插队 | 需要区分请求等级 |

#### 5.2 文件大小（File Size）

Agent 处理文件的边界检查包括多个维度：

```
上传文件检查流水线：

1. 原始大小（Raw Bytes）
   └── PDF, Word, Excel 等二进制格式
   限制: 10-50 MB（取决于解析能力）

2. 解压后大小（Decompressed Size）
   └── ZIP, TAR, GZ 等压缩格式
   限制: 压缩比 ≥ 10 倍时警惕"Zip Bomb"攻击
   检查: decompressed_size / compressed_size < 100

3. 提取后文本大小（Extracted Text）
   └── PDF 解析、文字提取、OCR 结果
   限制: 100K-500K tokens（取决于 Agent 的 Token 预算）

4. 图像分辨率（Image Resolution）
   └── 直接影响 LLM 视觉处理的计算成本
   限制: 2000x2000 px 或 4M 像素（超分辨率图片需降采样）
```

##### Zip Bomb 防护

```python
def check_file_boundary(file_path: str, limits: FileLimits) -> FileCheckResult:
    """
    文件边界检查——防止文件相关的资源耗尽攻击
    """
    # 1. 检查原始大小
    raw_size = os.path.getsize(file_path)
    if raw_size > limits.max_raw_size:
        return FileCheckResult.reject(f"文件大小 {raw_size} 超过 {limits.max_raw_size}")

    # 2. 检测压缩文件
    if is_archive(file_path):
        estimated_decompressed = estimate_decompressed_size(file_path)
        # Zip Bomb 检测：压缩比异常
        if estimated_decompressed > 0:
            ratio = estimated_decompressed / raw_size
            if ratio > limits.max_compression_ratio:
                return FileCheckResult.reject(
                    f"疑似 Zip Bomb：压缩比 {ratio:.1f}x 超过上限 {limits.max_compression_ratio}x"
                )

    return FileCheckResult.pass_()
```

#### 5.3 图像分辨率（Image Resolution）

多模态 Agent 处理图像时的分辨率边界：

```
分辨率             像素数          Token 成本（近似）
──────────────────────────────────────────────────
100x100            10,000         ~50 tokens
500x500            250,000        ~1,250 tokens
1000x1000          1,000,000      ~5,000 tokens
2000x2000          4,000,000      ~20,000 tokens  ← 典型上限
4000x4000          16,000,000     ~80,000 tokens  ← 分辨率的巨大影响

边界检查策略：
- 硬限制：拒绝超过 4000x4000 的图像
- 软限制：超过 2000x2000 的图像自动降采样后处理
- Token 预算模式：根据 Agent 剩余 Token 预算动态调整

# 动态降采样逻辑
def adaptive_resize(image, max_tokens_for_vision):
    current_tokens = estimate_image_tokens(image)
    if current_tokens > max_tokens_for_vision:
        scale = math.sqrt(max_tokens_for_vision / current_tokens)
        new_size = (int(image.width * scale), int(image.height * scale))
        return image.resize(new_size, Image.LANCZOS)
    return image
```

---

### 6. Hierarchical Boundaries — 层级边界

不同来源的输入应有不同的边界限制。这是 Agent 独有的边界维度——因为 Agent 从多个渠道接收输入，每个渠道的可信度和资源占用成本不同。

#### 6.1 按来源分级

```
输入来源            信任度     Token 预算    长度限制    说明
──────────────────────────────────────────────────────────────
系统提示 (system)   最高      固定占用        不适用     不可压缩
工具定义 (tool)     高        固定占用        不适用     不可压缩
内部数据源          高        宽松             宽松       DB 查询结果
已验证用户输入      中-高     标准             标准       已认证用户
未验证用户输入      中        较严格          较严格     游客/匿名
上传文档            低        严格             严格       UGC 内容
网络搜索结果        低        严格             严格       可能包含攻击
第三方 API 返回     低        严格             严格       不可信来源
```

#### 6.2 层级预算分配算法

```python
class HierarchicalBudgetManager:
    """
    层级 Token 预算分配器
    根据不同来源设置不同的预算上限和超限处理策略
    """

    # 来源层级定义
    SOURCE_TIERS = {
        "system": TierConfig(
            priority=100,            # 最高优先级——永远完整保留
            budget="fixed",          # 占用固定预算
            overflow_action="raise_error"  # 预算不足 → 系统配置错误
        ),
        "tool_definitions": TierConfig(
            priority=90,
            budget="fixed",
            overflow_action="raise_error"
        ),
        "verified_user": TierConfig(
            priority=50,
            budget="pooled",         # 共享用户输入预算池
            max_per_message=10000,   # 单条消息上限 10K tokens
            overflow_action="truncate_or_reject"
        ),
        "anonymous_user": TierConfig(
            priority=30,
            budget="pooled",
            max_per_message=2000,    # 匿名用户单条消息上限 2K tokens
            overflow_action="reject"
        ),
        "external_content": TierConfig(
            priority=20,
            budget="pooled",
            max_per_source=5000,     # 每个外部来源上限 5K tokens
            overflow_action="truncate"
        ),
        "tool_results": TierConfig(
            priority=40,
            budget="dynamic",        # 动态分配——基于剩余预算
            max_total=30000,         # 累计工具结果上限 30K tokens
            overflow_action="summarize_or_truncate"
        ),
    }

    def allocate(self, source: str, token_count: int) -> AllocationResult:
        """
        为指定来源分配 Token 预算
        返回是否分配成功及分配后的额度
        """
        config = self.SOURCE_TIERS.get(source)
        if not config:
            return AllocationResult.reject(f"未知来源: {source}")

        if config.budget == "fixed":
            # 固定预算——不能超过预设值
            if token_count > config.fixed_amount:
                return AllocationResult.reject(f"{source} 超出固定预算")
            return AllocationResult.approve(token_count)

        elif config.budget == "pooled":
            # 共享预算——从公共池中分配
            if self.remaining_pool < token_count:
                if config.overflow_action == "reject":
                    return AllocationResult.reject("共享预算不足")
                elif config.overflow_action == "truncate":
                    token_count = min(token_count, self.remaining_pool)
                    self.remaining_pool -= token_count
                    return AllocationResult.approve(token_count, truncated=True)
            self.remaining_pool -= token_count
            return AllocationResult.approve(token_count)

        elif config.budget == "dynamic":
            # 动态预算——按优先级从剩余空间中分配
            max_allowed = min(config.max_total, self.remaining_pool)
            if token_count > max_allowed:
                # 超限时尝试压缩
                return AllocationResult.reject(f"{source} 超出动态预算上限 {max_allowed}")
            self.remaining_pool -= token_count
            return AllocationResult.approve(token_count)
```

#### 6.3 层级间的耦合关系

不同层级的边界不是独立的——一个层级的超限可能影响其他层级的可用预算：

```
例子：用户上传了一篇 80K tokens 的文档

层级分配前：
  系统提示:   5K tokens  (固定)
  工具定义:   3K tokens  (固定)  
  输出预留:   20K tokens (固定比例 20%)
  对话历史:   30K tokens (动态)
  用户输入:   42K tokens (剩余空间)

如果文档 80K tokens > 用户输入 42K tokens 预算：
  → 拒绝文档（超限 38K tokens）
  → 或：压缩对话历史到 10K tokens，用户输入空间 = 62K tokens
      仍然不够（80K > 62K）
  → 或：压缩输出预留到 10K tokens
      用户输入空间 = 72K tokens
      仍然不够（80K > 72K）
  → 必须拒绝：无论如何都无法容纳

层级耦合规则：低优先级的预算可以被高优先级抢占
抢占顺序：输出预留 → 对话历史 → 用户输入 → 工具定义 → 系统提示
```

---

### 7. Adaptive Boundaries — 自适应边界

自适应边界是解决"核心矛盾"（严格 vs 宽松）的关键技术——边界不是静态的，而是根据 Agent 的当前状态和任务特点动态调整。

#### 7.1 基于 Agent 容量的动态调整

```
核心思想：Agent 在不同时间点有不同的"处理容量"
  高容量时段           低容量时段
  (系统空闲)            (系统高负载)
  ┌────────────┐       ┌────────────┐
  │ 容量: 80%  │       │ 容量: 30%  │
  │ Token 预算:│       │ Token 预算:│
  │ 50K tokens │       │ 20K tokens │
  │ 并发限制:  │       │ 并发限制:  │
  │ 10 调      │       │ 3 调       │
  │ 频率限制:  │       │ 频率限制:  │
  │ 100 req/m  │       │ 30 req/m   │
  └────────────┘       └────────────┘
```

Agent 容量指标：

```python
class AgentCapacity:
    """Agent 当前处理容量评估"""

    def __init__(self):
        self.current_requests = 0
        self.max_requests = 100
        self.avg_inference_time = 1.5  # 秒
        self.memory_usage_mb = 0
        self.max_memory_mb = 8000
        self.token_budget_remaining = 50000  # 当前分配剩余的 Token

    def get_bounds_multiplier(self) -> float:
        """
        返回当前容量因子（0.0-1.0）
        1.0 = 完全容量（边界可放宽）
        0.0 = 无剩余容量（边界应最严格）
        """
        load_factors = {
            "request_load": self.current_requests / self.max_requests,
            "memory_pressure": self.memory_usage_mb / self.max_memory_mb,
            "inference_slowdown": min(1.0, self.avg_inference_time / 5.0),
        }

        # 综合负载 = 各指标加权平均
        composite_load = (
            load_factors["request_load"] * 0.4
            + load_factors["memory_pressure"] * 0.3
            + load_factors["inference_slowdown"] * 0.3
        )

        # 容量因子: 负载低 → 因子高（更宽松）
        return max(0.2, 1.0 - composite_load)

    def scale_bound(self, base_limit: float) -> float:
        """根据当前容量调整边界值"""
        return base_limit * (0.2 + 0.8 * self.get_bounds_multiplier())
```

#### 7.2 基于任务复杂度的动态调整

不同任务对资源的需求不同，自适应边界根据任务类型分配预算：

```
任务复杂度分档：

简单任务（✩）
  例子: "今天天气怎么样？", "2+2=?"
  Token 预算: 5K
  工具调用限制: 1 次
  无需对话历史

常规任务（✩✩）
  例子: "帮我写一封邮件", "分析这个文档"
  Token 预算: 20K
  工具调用限制: 5 次
  需要最近 3 轮对话历史

复杂任务（✩✩✩）
  例子: "对比这三份报告并给出建议", "分析这个数据集"
  Token 预算: 50K
  工具调用限制: 20 次
  需要全量对话历史

极复杂任务（✩✩✩✩）
  例子: "审查这份 500 页合同中的所有法律风险"
  Token 预算: 100K（可能需要分块）
  工具调用限制: 50 次
  需要从对话历史中提取摘要
```

复杂度评估算法：

```python
def estimate_task_complexity(user_input: str, attachments: list) -> ComplexityLevel:
    """
    通过输入特征预估任务复杂度
    用于自适应边界调整
    """
    score = 0.0

    # 1. 输入长度
    token_count = count_tokens(user_input)
    score += min(2.0, token_count / 10000)  # 10K tokens = 1 分

    # 2. 工具调用关键词（隐含多步推理需求）
    tool_indicators = ["compare", "analyze", "research", "summarize",
                       "find", "search", "calculate", "generate report"]
    found_indicators = sum(1 for word in tool_indicators
                           if word in user_input.lower())
    score += min(1.0, found_indicators * 0.25)

    # 3. 附件数量和大小
    for att in attachments:
        score += min(1.0, estimate_document_complexity(att))

    # 4. 跨工具操作需求
    if has_multi_tool_request(user_input):
        score += 0.5

    # 5. 时间敏感度
    if is_long_running_task(user_input):
        score += 0.3

    # 映射到复杂度等级
    if score < 0.5:
        return ComplexityLevel.SIMPLE
    elif score < 1.5:
        return ComplexityLevel.REGULAR
    elif score < 3.0:
        return ComplexityLevel.COMPLEX
    else:
        return ComplexityLevel.VERY_COMPLEX
```

#### 7.3 自适应边界的风险

自适应边界是一把双刃剑——攻击者可能利用边界放宽机制：

```
攻击向量 1：伪装复杂度
  攻击者将恶意请求包装为"复杂任务"
  → Agent 分配高 Token 预算
  → 注入指令在大预算下更容易渗透
  
防御: 复杂度评估仅用于"资源分配"而非"安全检查"
      安全边界不会因复杂度评估而放宽

攻击向量 2：容量欺诈
  攻击者通过大量低负载请求耗尽 Agent 容量
  → 自适应边界收紧
  → 合法用户被严格限制
  
防御: 自适应边界的收紧不能低于硬性底线
      每个用户有独立的最低保障配额
```

---

## Example Code — Python BoundaryChecker

以下是一个完整的 Python 边界检查器，集成了 Token 预算管理、层级限制和自适应边界：

```python
"""
BoundaryChecker — AI Agent 输入边界检查器

功能：
1. Token 预算管理 —— 为不同来源分配 Token
2. 长度边界 —— 字符/Token/字段长度检查
3. 范围边界 —— 数值/数组/嵌套深度检查
4. 时间边界 —— 频率/速率/会话时长检查
5. 资源边界 —— 文件/图像/并发工具调用检查
6. 层级边界 —— 按来源分配不同预算
7. 自适应边界 —— 基于容量和复杂度动态调整
"""

import time
import math
import re
from typing import Any, Dict, List, Optional, Tuple, Callable
from dataclasses import dataclass, field
from enum import Enum, auto


# ============================================================
# 基础数据结构
# ============================================================

class BoundaryType(Enum):
    TOKEN_BUDGET = auto()
    LENGTH = auto()
    RANGE = auto()
    TIME = auto()
    RESOURCE = auto()
    HIERARCHICAL = auto()
    ADAPTIVE = auto()


class SourceTier(Enum):
    SYSTEM = "system"                # 最高优先级
    TOOL_DEFINITION = "tool_def"
    VERIFIED_USER = "verified_user"
    ANONYMOUS_USER = "anonymous_user"
    UPLOADED_DOCUMENT = "document"
    WEB_CONTENT = "web_content"
    TOOL_RESULT = "tool_result"


class ComplexityLevel(Enum):
    SIMPLE = 0
    REGULAR = 1
    COMPLEX = 2
    VERY_COMPLEX = 3


class Severity(Enum):
    PASS = "pass"
    WARN = "warn"           # 超限但可处理（截断/压缩）
    REJECT = "reject"       # 超限且必须拒绝
    FATAL = "fatal"         # 超限且系统状态异常


@dataclass
class BoundCheckResult:
    """单次边界检查的结果"""
    boundary: BoundaryType
    severity: Severity
    message: str
    details: Dict[str, Any] = field(default_factory=dict)
    suggested_action: Optional[str] = None  # 建议的修复措施


@dataclass 
class TokenBudget:
    """Token 预算分配方案"""
    total_window: int
    system_prompt: int
    tool_definitions: int
    output_reserve: int
    conversation_history: int
    user_input: int
    tool_results: int = 0
    reserved_buffer: int = 0  # 额外缓冲（防止 Token 计数误差）

    @property
    def allocated(self) -> int:
        return (self.system_prompt + self.tool_definitions
                + self.output_reserve + self.conversation_history
                + self.user_input + self.tool_results
                + self.reserved_buffer)

    @property
    def remaining(self) -> int:
        return self.total_window - self.allocated

    @property
    def utilization(self) -> float:
        return self.allocated / self.total_window


@dataclass
class TierConfig:
    """层级配置"""
    priority: int
    budget_type: str  # "fixed", "pooled", "dynamic"
    max_per_message: Optional[int] = None
    max_total: Optional[int] = None
    overflow_action: str = "reject"  # "reject", "truncate", "summarize"
    fixed_amount: Optional[int] = None


@dataclass
class CapacityMetrics:
    """Agent 当前容量指标"""
    request_load: float = 0.0           # 0.0-1.0
    memory_pressure: float = 0.0        # 0.0-1.0
    inference_slowdown: float = 0.0     # 0.0-1.0
    token_rate: float = 0.0             # tokens/second 消耗速率


# ============================================================
# Token 计数器（简化版）
# ============================================================

def count_tokens(text: str) -> int:
    """
    简化的 Token 计数——实际应调用 tiktoken 或对应模型的 Tokenizer
    这里使用字符数 / 4 作为近似估计（英文平均 1 token ≈ 4 字符）
    """
    if not text:
        return 0
    # 粗略估算：对中文每个字约 1.5 token，英文每 4 字符 1 token
    chinese_chars = len(re.findall(r'[一-鿿]', text))
    ascii_chars = len(text) - chinese_chars
    return math.ceil(chinese_chars * 1.5 + ascii_chars / 4)


def count_image_tokens(width: int, height: int) -> int:
    """图像 Token 估算（近似值，具体取决于模型的图像编码方式）"""
    # 简化的估算：每 256x256 块约 200 tokens
    blocks = math.ceil(width / 256) * math.ceil(height / 256)
    return blocks * 200


# ============================================================
# 核心边界检查器
# ============================================================

class BoundConfig:
    """边界配置——所有可调整的边界参数集中管理"""

    def __init__(self):
        # --- Token 预算 ---
        self.model_context_window = 200000          # Claude 3.5 Sonnet
        self.output_reserve_ratio = 0.2
        self.reserved_buffer_ratio = 0.02           # 2% 缓冲

        # --- 长度边界 ---
        self.max_message_chars = 50000              # 单条消息最大字符数
        self.max_message_lines = 1000               # 单条消息最大行数
        self.max_attachment_tokens = 100000         # 附件最大 Token 数
        self.max_conversation_turns = 200           # 最大对话轮次

        # --- 范围边界 ---
        self.max_nesting_depth = 32                 # 最大嵌套深度
        self.max_array_items = 200                  # 数组最大长度
        self.default_numeric_max = 2**31 - 1
        self.default_numeric_min = -(2**31)

        # --- 时间边界 ---
        self.rate_limit_per_minute = 60             # 每分钟请求数
        self.tool_call_rate_per_minute = 30         # 每分钟工具调用数
        self.session_timeout_minutes = 30           # 会话超时分钟数
        self.max_tool_calls_per_request = 30        # 单次请求最大工具调用
        self.tool_call_cooldown = 0.5               # 工具调用冷却秒数

        # --- 资源边界 ---
        self.max_concurrent_tool_calls = 10         # 最大并发工具调用
        self.max_file_size_mb = 50                  # 单个文件大小上限
        self.max_total_upload_mb = 200              # 总上传大小上限
        self.max_compression_ratio = 50             # 最大压缩比
        self.max_image_resolution = 4000            # 最大图像分辨率（像素）

        # --- 层级边界 ---
        self.tier_configs: Dict[SourceTier, TierConfig] = {
            SourceTier.SYSTEM: TierConfig(
                priority=100, budget_type="fixed",
                overflow_action="raise_error"
            ),
            SourceTier.TOOL_DEFINITION: TierConfig(
                priority=90, budget_type="fixed",
                overflow_action="raise_error"
            ),
            SourceTier.VERIFIED_USER: TierConfig(
                priority=50, budget_type="pooled",
                max_per_message=10000,
                overflow_action="truncate_or_reject"
            ),
            SourceTier.ANONYMOUS_USER: TierConfig(
                priority=40, budget_type="pooled",
                max_per_message=2000,
                overflow_action="reject"
            ),
            SourceTier.UPLOADED_DOCUMENT: TierConfig(
                priority=30, budget_type="pooled",
                max_per_message=50000,
                overflow_action="truncate"
            ),
            SourceTier.WEB_CONTENT: TierConfig(
                priority=20, budget_type="pooled",
                max_per_message=5000,
                overflow_action="truncate"
            ),
            SourceTier.TOOL_RESULT: TierConfig(
                priority=35, budget_type="dynamic",
                max_total=30000,
                overflow_action="summarize_or_truncate"
            ),
        }

        # --- 自适应边界 ---
        self.adaptive_enabled = True
        self.min_bounds_multiplier = 0.2
        self.max_bounds_multiplier = 1.5
        self.complexity_token_multipliers = {
            ComplexityLevel.SIMPLE: 0.5,
            ComplexityLevel.REGULAR: 1.0,
            ComplexityLevel.COMPLEX: 1.5,
            ComplexityLevel.VERY_COMPLEX: 2.0,
        }


class BoundaryChecker:
    """
    综合边界检查器
    按顺序执行所有边界检查：Token → 长度 → 范围 → 时间 → 资源 → 层级 → 自适应
    """

    def __init__(self, config: Optional[BoundConfig] = None):
        self.config = config or BoundConfig()
        self.capacity = CapacityMetrics()
        self._token_pool_remaining = 0.0

    # -------------------------------------------------------
    # 1. Token 预算管理
    # -------------------------------------------------------

    def check_token_budget(self, sources: Dict[SourceTier, str]) -> List[BoundCheckResult]:
        """
        检查所有来源的 Token 预算
        sources: 按来源分组的输入文本
        """
        results = []
        system_prompt_tokens = count_tokens(
            sources.get(SourceTier.SYSTEM, "")
        )
        tool_def_tokens = count_tokens(
            sources.get(SourceTier.TOOL_DEFINITION, "")
        )

        # 固定部分必须先放入窗口
        if system_prompt_tokens > self.config.model_context_window * 0.5:
            return [BoundCheckResult(
                boundary=BoundaryType.TOKEN_BUDGET,
                severity=Severity.FATAL,
                message="系统提示过大，无法在窗口中完整保留",
                details={"system_prompt_tokens": system_prompt_tokens}
            )]

        # 计算预算
        output_reserve = int(
            self.config.model_context_window * self.config.output_reserve_ratio
        )
        buffer = int(
            self.config.model_context_window * self.config.reserved_buffer_ratio
        )

        fixed_total = system_prompt_tokens + tool_def_tokens + output_reserve + buffer
        available = self.config.model_context_window - fixed_total

        # 分配各动态来源的预算
        total_input = 0
        for source in [SourceTier.VERIFIED_USER, SourceTier.ANONYMOUS_USER,
                       SourceTier.UPLOADED_DOCUMENT, SourceTier.WEB_CONTENT,
                       SourceTier.TOOL_RESULT]:
            text = sources.get(source, "")
            if not text:
                continue
            tokens = count_tokens(text)
            tier_config = self.config.tier_configs.get(source)

            if tier_config and tier_config.max_per_message:
                if tokens > tier_config.max_per_message:
                    results.append(BoundCheckResult(
                        boundary=BoundaryType.TOKEN_BUDGET,
                        severity=Severity.WARN,
                        message=f"{source.value} 单条消息超限：{tokens} > {tier_config.max_per_message}",
                        details={"source": source.value, "tokens": tokens,
                                 "limit": tier_config.max_per_message},
                        suggested_action="截断或分块处理"
                    ))
                    tokens = tier_config.max_per_message  # 截断至上限

            total_input += tokens

        if total_input > available:
            results.append(BoundCheckResult(
                boundary=BoundaryType.TOKEN_BUDGET,
                severity=Severity.REJECT,
                message=f"总输入 {total_input} tokens 超出可用预算 {available} tokens",
                details={"total_input": total_input, "available": available},
                suggested_action="减少附件大小或简化问题描述"
            ))
        else:
            self._token_pool_remaining = available - total_input
            results.append(BoundCheckResult(
                boundary=BoundaryType.TOKEN_BUDGET,
                severity=Severity.PASS,
                message=f"Token 预算检查通过，剩余 {self._token_pool_remaining} tokens",
                details={"total_input": total_input, "available": available,
                         "remaining": self._token_pool_remaining}
            ))

        return results

    # -------------------------------------------------------
    # 2. 长度边界
    # -------------------------------------------------------

    def check_length(self, text: str, source: SourceTier) -> BoundCheckResult:
        """检查输入文本的长度边界"""
        char_count = len(text)
        line_count = text.count("\n") + 1

        violations = []
        if char_count > self.config.max_message_chars:
            violations.append(f"字符数 {char_count} > {self.config.max_message_chars}")

        if line_count > self.config.max_message_lines:
            violations.append(f"行数 {line_count} > {self.config.max_message_lines}")

        if violations:
            return BoundCheckResult(
                boundary=BoundaryType.LENGTH,
                severity=Severity.WARN if source in (
                    SourceTier.VERIFIED_USER, SourceTier.TOOL_RESULT
                ) else Severity.REJECT,
                message="; ".join(violations),
                details={"char_count": char_count, "line_count": line_count},
                suggested_action="精简输入内容"
            )

        return BoundCheckResult(
            boundary=BoundaryType.LENGTH,
            severity=Severity.PASS,
            message="长度边界检查通过",
            details={"char_count": char_count, "line_count": line_count}
        )

    # -------------------------------------------------------
    # 3. 范围边界
    # -------------------------------------------------------

    def check_range(self, value: Any, schema: Optional[Dict] = None) -> BoundCheckResult:
        """
        检查数值/数组/嵌套深度范围
        schema 可选，提供字段级别的边界定义
        """
        # 数值范围
        if isinstance(value, (int, float)):
            min_val = (schema or {}).get("minimum", self.config.default_numeric_min)
            max_val = (schema or {}).get("maximum", self.config.default_numeric_max)

            if value < min_val:
                return BoundCheckResult(
                    boundary=BoundaryType.RANGE,
                    severity=Severity.REJECT,
                    message=f"值 {value} 低于最小值 {min_val}",
                    details={"value": value, "min": min_val, "max": max_val}
                )
            if value > max_val:
                return BoundCheckResult(
                    boundary=BoundaryType.RANGE,
                    severity=Severity.REJECT,
                    message=f"值 {value} 超出最大值 {max_val}",
                    details={"value": value, "min": min_val, "max": max_val}
                )

        # 数组长度
        elif isinstance(value, (list, tuple)):
            if len(value) > self.config.max_array_items:
                return BoundCheckResult(
                    boundary=BoundaryType.RANGE,
                    severity=Severity.REJECT,
                    message=f"数组长度 {len(value)} > {self.config.max_array_items}",
                    details={"length": len(value), "max": self.config.max_array_items}
                )

            # 检查数组元素的嵌套深度
            try:
                self._check_nesting_depth(value, self.config.max_nesting_depth)
            except ValueError as e:
                return BoundCheckResult(
                    boundary=BoundaryType.RANGE,
                    severity=Severity.REJECT,
                    message=str(e),
                    details={"nesting_depth": str(e)}
                )

        # 字符串长度（通过 Range 检查确认）
        elif isinstance(value, str):
            if len(value) > self.config.max_message_chars:
                return BoundCheckResult(
                    boundary=BoundaryType.RANGE,
                    severity=Severity.WARN,
                    message=f"字符串长度 {len(value)} 超出建议上限",
                    details={"length": len(value)}
                )

        return BoundCheckResult(
            boundary=BoundaryType.RANGE,
            severity=Severity.PASS,
            message="范围边界检查通过",
            details={"value_type": type(value).__name__}
        )

    def _check_nesting_depth(self, obj: Any, max_depth: int, current: int = 0):
        """递归检查嵌套深度"""
        if current > max_depth:
            raise ValueError(f"嵌套深度 {current} 超过上限 {max_depth}")

        if isinstance(obj, dict):
            for v in obj.values():
                self._check_nesting_depth(v, max_depth, current + 1)
        elif isinstance(obj, (list, tuple)):
            for item in obj:
                self._check_nesting_depth(item, max_depth, current + 1)

    # -------------------------------------------------------
    # 4. 时间边界
    # -------------------------------------------------------

    def __init__(self, config: Optional[BoundConfig] = None):
        self.config = config or BoundConfig()
        self.capacity = CapacityMetrics()
        self._token_pool_remaining = 0.0
        # 时间边界状态
        self._user_request_timestamps: Dict[str, List[float]] = {}
        self._tool_call_timestamps: List[float] = []
        self._session_start_time: Optional[float] = None
        self._tool_call_count = 0

    def check_time(self, user_id: str) -> List[BoundCheckResult]:
        """检查时间相关边界"""
        results = []
        now = time.time()

        # 4.1 请求频率
        user_ts = self._user_request_timestamps.get(user_id, [])
        # 清理 60 秒前的记录
        user_ts = [ts for ts in user_ts if now - ts < 60]
        self._user_request_timestamps[user_id] = user_ts

        if len(user_ts) >= self.config.rate_limit_per_minute:
            wait_time = 60 - (now - user_ts[0])
            results.append(BoundCheckResult(
                boundary=BoundaryType.TIME,
                severity=Severity.REJECT,
                message=f"请求频率超限，请 {wait_time:.0f} 秒后重试",
                details={
                    "requests_in_last_minute": len(user_ts),
                    "limit": self.config.rate_limit_per_minute,
                    "retry_after_seconds": wait_time
                },
                suggested_action=f"等待 {wait_time:.0f} 秒后重试"
            ))
        else:
            user_ts.append(now)
            self._user_request_timestamps[user_id] = user_ts

        # 4.2 工具调用速率
        self._tool_call_timestamps = [
            ts for ts in self._tool_call_timestamps if now - ts < 60
        ]
        if len(self._tool_call_timestamps) >= self.config.tool_call_rate_per_minute:
            results.append(BoundCheckResult(
                boundary=BoundaryType.TIME,
                severity=Severity.REJECT,
                message="工具调用频率超限",
                details={
                    "calls_in_last_minute": len(self._tool_call_timestamps),
                    "limit": self.config.tool_call_rate_per_minute
                }
            ))

        return results

    def check_tool_call_rate(self) -> BoundCheckResult:
        """检查工具调用速率"""
        now = time.time()
        self._tool_call_timestamps = [
            ts for ts in self._tool_call_timestamps if now - ts < 60
        ]

        if self._tool_call_count >= self.config.max_tool_calls_per_request:
            return BoundCheckResult(
                boundary=BoundaryType.TIME,
                severity=Severity.REJECT,
                message=f"单次请求工具调用次数 {self._tool_call_count} 超过上限 {self.config.max_tool_calls_per_request}",
                details={
                    "current_calls": self._tool_call_count,
                    "max_calls": self.config.max_tool_calls_per_request
                }
            )

        self._tool_call_timestamps.append(now)
        self._tool_call_count += 1

        return BoundCheckResult(
            boundary=BoundaryType.TIME,
            severity=Severity.PASS,
            message=f"工具调用 {self._tool_call_count}/{self.config.max_tool_calls_per_request}",
            details={"calls_used": self._tool_call_count}
        )

    def check_session_timeout(self) -> BoundCheckResult:
        """检查会话是否超时"""
        if self._session_start_time is None:
            self._session_start_time = time.time()
            return BoundCheckResult(
                boundary=BoundaryType.TIME,
                severity=Severity.PASS,
                message="会话已开始"
            )

        elapsed = time.time() - self._session_start_time
        max_seconds = self.config.session_timeout_minutes * 60

        if elapsed > max_seconds:
            return BoundCheckResult(
                boundary=BoundaryType.TIME,
                severity=Severity.REJECT,
                message=f"会话超时（已持续 {elapsed/60:.0f} 分钟）",
                details={"elapsed_minutes": elapsed / 60,
                         "timeout_minutes": self.config.session_timeout_minutes},
                suggested_action="重新开始新会话"
            )

        return BoundCheckResult(
            boundary=BoundaryType.TIME,
            severity=Severity.PASS,
            message=f"会话进行中（{elapsed/60:.0f}/{self.config.session_timeout_minutes} 分钟）"
        )

    # -------------------------------------------------------
    # 5. 资源边界
    # -------------------------------------------------------

    def check_file_resource(self, file_path: str, file_size_bytes: int) -> BoundCheckResult:
        """检查文件资源边界"""
        file_size_mb = file_size_bytes / (1024 * 1024)

        if file_size_mb > self.config.max_file_size_mb:
            return BoundCheckResult(
                boundary=BoundaryType.RESOURCE,
                severity=Severity.REJECT,
                message=f"文件大小 {file_size_mb:.1f}MB 超过上限 {self.config.max_file_size_mb}MB",
                details={
                    "file_size_mb": file_size_mb,
                    "max_mb": self.config.max_file_size_mb
                },
                suggested_action="压缩后上传或使用较小文件"
            )

        return BoundCheckResult(
            boundary=BoundaryType.RESOURCE,
            severity=Severity.PASS,
            message=f"文件大小检查通过 ({file_size_mb:.1f}MB)"
        )

    def check_image_resource(self, width: int, height: int) -> BoundCheckResult:
        """检查图像资源边界"""
        resolution = max(width, height)

        if resolution > self.config.max_image_resolution:
            return BoundCheckResult(
                boundary=BoundaryType.RESOURCE,
                severity=Severity.REJECT,
                message=f"图像分辨率 {width}x{height} 超过上限 {self.config.max_image_resolution}px",
                details={
                    "width": width, "height": height,
                    "max_resolution": self.config.max_image_resolution
                },
                suggested_action="降采样后上传"
            )

        vision_tokens = count_image_tokens(width, height)
        if vision_tokens > self._token_pool_remaining * 0.3:
            return BoundCheckResult(
                boundary=BoundaryType.RESOURCE,
                severity=Severity.WARN,
                message=f"图像所需 Token ({vision_tokens}) 接近可用预算上限",
                details={
                    "vision_tokens": vision_tokens,
                    "remaining_budget": self._token_pool_remaining
                }
            )

        return BoundCheckResult(
            boundary=BoundaryType.RESOURCE,
            severity=Severity.PASS,
            message=f"图像资源检查通过 ({width}x{height}, ~{vision_tokens} tokens)"
        )

    def check_concurrent_tool_calls(self, current_concurrent: int) -> BoundCheckResult:
        """检查并发工具调用数"""
        if current_concurrent >= self.config.max_concurrent_tool_calls:
            return BoundCheckResult(
                boundary=BoundaryType.RESOURCE,
                severity=Severity.REJECT,
                message=f"并发工具调用 {current_concurrent} 已达上限 {self.config.max_concurrent_tool_calls}",
                details={
                    "current": current_concurrent,
                    "max": self.config.max_concurrent_tool_calls
                },
                suggested_action="等待当前工具调用完成后再发起新的调用"
            )

        return BoundCheckResult(
            boundary=BoundaryType.RESOURCE,
            severity=Severity.PASS,
            message=f"并发工具调用 {current_concurrent}/{self.config.max_concurrent_tool_calls}"
        )

    # -------------------------------------------------------
    # 6. 层级边界
    # -------------------------------------------------------

    def check_hierarchical(self, source: SourceTier, text: str) -> BoundCheckResult:
        """
        层级边界检查——根据来源设置不同限制
        """
        tier_config = self.config.tier_configs.get(source)
        if not tier_config:
            return BoundCheckResult(
                boundary=BoundaryType.HIERARCHICAL,
                severity=Severity.WARN,
                message=f"来源 {source.value} 无层级配置，使用默认限制"
            )

        tokens = count_tokens(text)

        # 固定预算类型——检查是否超出预设固定值
        if tier_config.budget_type == "fixed":
            if tier_config.fixed_amount and tokens > tier_config.fixed_amount:
                return BoundCheckResult(
                    boundary=BoundaryType.HIERARCHICAL,
                    severity=Severity.FATAL,
                    message=f"来源 {source.value} 固定预算超限：{tokens} > {tier_config.fixed_amount}",
                    details={"source": source.value, "tokens": tokens}
                )
            return BoundCheckResult(
                boundary=BoundaryType.HIERARCHICAL,
                severity=Severity.PASS,
                message=f"来源 {source.value} 固定预算正常"
            )

        # 共享/动态预算类型
        if tier_config.max_per_message and tokens > tier_config.max_per_message:
            action = tier_config.overflow_action
            if action == "reject":
                return BoundCheckResult(
                    boundary=BoundaryType.HIERARCHICAL,
                    severity=Severity.REJECT,
                    message=f"来源 {source.value} 单条消息 {tokens} tokens > 上限 {tier_config.max_per_message}",
                    details={"source": source.value, "tokens": tokens,
                             "limit": tier_config.max_per_message},
                    suggested_action="缩短或拆分输入"
                )
            elif "truncate" in action:
                return BoundCheckResult(
                    boundary=BoundaryType.HIERARCHICAL,
                    severity=Severity.WARN,
                    message=f"来源 {source.value} 超限，将被截断",
                    details={"source": source.value, "tokens": tokens,
                             "limit": tier_config.max_per_message},
                    suggested_action="自动截断至限制大小"
                )

        # 累积总量检查
        if tier_config.max_total and tokens > tier_config.max_total:
            return BoundCheckResult(
                boundary=BoundaryType.HIERARCHICAL,
                severity=Severity.REJECT,
                message=f"来源 {source.value} 累积 {tokens} > 总上限 {tier_config.max_total}",
                details={"source": source.value, "tokens": tokens,
                         "max_total": tier_config.max_total},
                suggested_action="减少该来源的总输入量"
            )

        return BoundCheckResult(
            boundary=BoundaryType.HIERARCHICAL,
            severity=Severity.PASS,
            message=f"来源 {source.value} 层级检查通过",
            details={"source": source.value, "tokens": tokens}
        )

    # -------------------------------------------------------
    # 7. 自适应边界
    # -------------------------------------------------------

    def compute_adaptive_multiplier(self,
                                     user_input: str,
                                     attachments: Optional[List[Dict]] = None) -> float:
        """计算自适应边界乘数"""
        if not self.config.adaptive_enabled:
            return 1.0

        # 容量因子
        capacity_mult = self._compute_capacity_multiplier()

        # 复杂度因子
        complexity = self._estimate_complexity(user_input, attachments or [])
        complexity_mult = self.config.complexity_token_multipliers.get(complexity, 1.0)

        # 综合乘数（取平均，限制在 [min, max] 范围内）
        combined = (capacity_mult + complexity_mult) / 2
        combined = max(self.config.min_bounds_multiplier,
                       min(self.config.max_bounds_multiplier, combined))

        return combined

    def _compute_capacity_multiplier(self) -> float:
        """基于当前容量计算乘数"""
        composite_load = (
            self.capacity.request_load * 0.4
            + self.capacity.memory_pressure * 0.3
            + self.capacity.inference_slowdown * 0.3
        )
        # 负载低 → 因子高（更宽松）
        return max(self.config.min_bounds_multiplier, 1.0 - composite_load)

    def _estimate_complexity(self, text: str,
                              attachments: List[Dict]) -> ComplexityLevel:
        """预估任务复杂度"""
        token_count = count_tokens(text)
        score = 0.0

        # 输入长度
        score += min(2.0, token_count / 10000)

        # 多步推理关键词
        multi_step_keywords = [
            "compare", "analyze", "research", "investigate",
            "find", "search", "then", "after that", "finally",
            "对比", "分析", "研究", "搜索", "然后", "之后",
        ]
        text_lower = text.lower()
        found = sum(1 for kw in multi_step_keywords if kw in text_lower)
        score += min(1.0, found * 0.15)

        # 附件影响
        score += min(2.0, len(attachments) * 0.5)

        # 映射到等级
        if score < 0.5:
            return ComplexityLevel.SIMPLE
        elif score < 1.5:
            return ComplexityLevel.REGULAR
        elif score < 3.0:
            return ComplexityLevel.COMPLEX
        else:
            return ComplexityLevel.VERY_COMPLEX

    def apply_adaptive_bounds(self, base_limits: Dict[str, float],
                               multiplier: float) -> Dict[str, float]:
        """应用自适应边界乘数到各限制值"""
        return {
            key: int(value * multiplier) if key.endswith("_tokens")
                 else value * multiplier
            for key, value in base_limits.items()
        }

    # -------------------------------------------------------
    # 综合检查入口
    # -------------------------------------------------------

    def validate_request(self,
                          user_id: str,
                          user_input: str,
                          sources: Dict[SourceTier, str],
                          attachments: Optional[List[Dict]] = None,
                          tool_calls_concurrent: int = 0) -> List[BoundCheckResult]:
        """
        对一次 Agent 请求执行完整的边界检查
        按固定顺序执行所有检查，返回所有结果
        """
        all_results = []

        # 0. 重置单次请求的工具调用计数
        self._tool_call_count = 0

        # 1. Token 预算检查（最优先——超限则后续检查无意义）
        budget_results = self.check_token_budget(sources)
        all_results.extend(budget_results)

        # 如果有致命/拒绝级别的 Token 预算问题，提前返回
        if any(r.severity in (Severity.FATAL, Severity.REJECT) for r in budget_results):
            return all_results

        # 2. 长度边界检查
        for source, text in sources.items():
            if source == SourceTier.SYSTEM:
                continue  # 系统提示在启动时已检查
            all_results.append(self.check_length(text, source))

        # 3. 范围边界检查
        # 对用户输入和工具参数中嵌入的数组/数值进行检查
        for source, text in sources.items():
            parsed = self._try_parse_json(text)
            if parsed is not None:
                all_results.append(self.check_range(parsed))

        # 4. 时间边界检查
        all_results.extend(self.check_time(user_id))
        all_results.append(self.check_session_timeout())

        # 5. 资源边界检查
        if attachments:
            for att in attachments:
                if "file_size" in att:
                    all_results.append(
                        self.check_file_resource(att.get("path", ""), att["file_size"])
                    )
                if "image_width" in att and "image_height" in att:
                    all_results.append(
                        self.check_image_resource(att["image_width"], att["image_height"])
                    )

        all_results.append(
            self.check_concurrent_tool_calls(tool_calls_concurrent)
        )

        # 6. 层级边界检查
        for source, text in sources.items():
            all_results.append(self.check_hierarchical(source, text))

        # 7. 自适应边界（调整后续请求的默认限制）
        if self.config.adaptive_enabled:
            multiplier = self.compute_adaptive_multiplier(user_input, attachments)
            all_results.append(BoundCheckResult(
                boundary=BoundaryType.ADAPTIVE,
                severity=Severity.PASS,
                message=f"自适应边界调整：乘数 = {multiplier:.2f}",
                details={
                    "capacity_load": self._compute_capacity_multiplier(),
                    "complexity": self._estimate_complexity(user_input, attachments or []).name,
                    "final_multiplier": multiplier
                }
            ))

        return all_results

    def _try_parse_json(self, text: str) -> Optional[Any]:
        """尝试解析 JSON——如果输入是 JSON 格式则解析"""
        import json
        text = text.strip()
        if text.startswith("{") or text.startswith("["):
            try:
                return json.loads(text)
            except json.JSONDecodeError:
                return None
        return None


# ============================================================
# 使用示例
# ============================================================

if __name__ == "__main__":
    checker = BoundaryChecker()
    user_id = "user_123"
    user_input = "请帮我分析这三份 CSV 文件中的销售趋势，并搜索最新的市场报告进行对比"

    sources = {
        SourceTier.SYSTEM: "You are a helpful AI assistant with access to data analysis and web search tools.",
        SourceTier.TOOL_DEFINITION: "Tools: analyze_csv(file_path: str), search_web(query: str), summarize(text: str)",
        SourceTier.VERIFIED_USER: user_input,
        SourceTier.TOOL_RESULT: "",
    }

    attachments = [
        {"path": "/data/sales_q1.csv", "file_size": 2_500_000},     # 2.5 MB
        {"path": "/data/sales_q2.csv", "file_size": 3_100_000},     # 3.1 MB
        {"path": "/data/sales_q3.csv", "file_size": 2_800_000},     # 2.8 MB
    ]

    results = checker.validate_request(
        user_id=user_id,
        user_input=user_input,
        sources=sources,
        attachments=attachments,
        tool_calls_concurrent=3
    )

    print("=" * 60)
    print("Boundary Check Results")
    print("=" * 60)
    for r in results:
        icon = {
            Severity.PASS: "  OK",
            Severity.WARN: " WARN",
            Severity.REJECT: " BLOCK",
            Severity.FATAL: " FATAL",
        }[r.severity]
        print(f"[{icon}] [{r.boundary.name}] {r.message}")
        if r.suggested_action:
            print(f"      建议: {r.suggested_action}")

    rejected = [r for r in results if r.severity in (Severity.REJECT, Severity.FATAL)]
    print("\n" + "=" * 60)
    if rejected:
        print(f"请求被拒绝 — {len(rejected)} 项边界检查未通过")
    else:
        print("请求通过所有边界检查 ✓")
        multiplier = checker.compute_adaptive_multiplier(user_input, attachments)
        print(f"自适应边界乘数: {multiplier:.2f}")
```

---

## Capability Boundaries — Boundaries Prevent Resource Attacks but Not Semantic Attacks

边界检查的能力范围——它擅长什么，不擅长什么。

### 能做的（Resource-Level Protection）

| 保护类型 | 能力说明 | 效果 |
|---------|---------|------|
| **防止资源耗尽** | 阻止超长输入、海量文件、无限循环 | 强——精确量化，可硬限制 |
| **防止经济 DoS** | 控制 Token 消耗、API 调用次数 | 强——成本可预测 |
| **防止内存溢出** | 限制文件大小、解压比、图像分辨率 | 强——系统级保护 |
| **防止频率滥用** | Rate limiting、并发控制 | 强——成熟技术 |
| **防止上下文污染** | 分来源预算、截断策略 | 中——取决于截断算法的质量 |
| **提示边界违规** | 给用户清晰的限制说明和错误消息 | 强——可改善用户体验 |

### 不能做的（Semantic-Level Blind Spots）

边界检查的所有限制都源于同一个事实：**边界只关心"量"，不关心"质"**。

```
边界检查无法防御的攻击类型：

1. 语义攻击（Semantic Attacks）
   — 输入在"量"上完全正常（100 tokens），但在"语义"上包含注入指令
   — 边界检查：通过 ✓  Token 预算正常，长度正常
   — 实际结果：Agent 被诱导执行恶意操作

2. 慢速攻击（Slow Drip Attacks）
   — 攻击者长时间、低频率地发送微小请求
   — 每单个请求都在边界之内
   — 累积效果：逐步诱导 Agent 泄露信息/执行操作
   — 边界检查：每次通过 ✓（没有任何单次触发边界）

3. 分布式攻击（Distributed Attacks）
   — 攻击者从多个账户/多个 IP 同时发起请求
   — 单用户的边界检查无法发现多账户的协同攻击

4. 合法边界内的恶意内容
   — 1000 tokens 的恶意指令完全在 Token 预算内
   — 边界检查：通过 ✓  长度正常、频率正常
   — 实际危害：1000 tokens 足以包含一个完整的 Prompt Injection

5. 侧信道攻击（Side-Channel Leakage）
   — Agent 在合法边界内的行为可能泄露信息
   — 例如：对不同输入的不同响应时间泄露了内部策略
   — 边界检查无法防御信息泄露

6. 配置绕过（Configuration Bypass）
   — 攻击者通过某种方式修改或绕过边界配置
   — 例如：通过越权操作修改 max_tokens_per_request
   - 边界检查本身无法防御对其配置的篡改
```

### 边界检查的准确角色

```
安全防御体系中的边界检查：

┌─────────────────────────────────────────────┐
│ 资源层保护                                    │
│ ┌──────────────────────────────────┐         │
│ │ 边界检查 ← 我们在这里              │         │
│ │ 目标：拒绝超出资源预算的请求        │         │
│ │ 方法：量化阈值 + 前置检查          │         │
│ └──────────────────────────────────┘         │
│                                              │
│ 边界检查之上还需要：                          │
│ ┌──────────────────────────────────┐         │
│ │ 内容安全层                         │         │
│ │ 目标：检测恶意内容、注入指令        │         │
│ │ 方法：注入检测 + 语义分析          │         │
│ └──────────────────────────────────┘         │
│ ┌──────────────────────────────────┐         │
│ │ 行为监控层                         │         │
│ │ 目标：发现异常行为模式              │         │
│ │ 方法：审计日志 + 异常检测          │         │
│ └──────────────────────────────────┘         │
└─────────────────────────────────────────────┘

边界检查是资源安全的"守门人"，但不是内容安全的"裁判"。
它解决的是"能不能处理"的问题，不是"该不该处理"的问题。
```

### 结果边界

```
边界检查的最佳效果：
┌────────────────────────────────────┐
│ 非法资源消耗攻击: 100% 阻断         │
│ Token 超限:             100% 拒绝  │
│ 文件超限:               100% 拒绝  │
│ 频率超限:               100% 拒绝  │
│ 并发超限:               100% 拒绝  │
│ ───────────────────────────────── │
│ 语义攻击:                 0% 阻断  │
│ 注入攻击:                 0% 阻断  │
│ 慢速攻击:                 0% 阻断  │
└────────────────────────────────────┘
这就是为什么边界检查是"12.3 input-validation"中的一层，而不是全部。
```

---

## Comparison — Static vs Dynamic Boundaries

| 维度 | 静态边界（Static Boundaries） | 动态边界（Dynamic Boundaries） |
|------|---------------------------|---------------------------|
| **定义** | 固定不变的硬阈值 | 根据上下文实时调整的阈值 |
| **例子** | `max_tokens=4000`，所有请求统一 | `max_tokens=2000~8000`，根据用户等级和系统负载调整 |
| **确定性** | 高——行为完全可预测 | 低——相同输入在不同时刻可能得到不同结果 |
| **实现复杂度** | 低——常量配置即可 | 高——需要容量监控、复杂度评估、实时计算 |
| **安全性** | 强——最严格的边界总是适用 | 中——放宽的边界可能被利用 |
| **可用性** | 低——"一刀切"可能误伤合法用户 | 高——在安全前提下尽量满足用户需求 |
| **维护成本** | 低——配置一次，长期有效 | 高——需要持续调整优化 |
| **适用场景** | 资源预算紧、安全要求高的场景 | 用户体验优先、资源充足的场景 |
| **可审计性** | 强——拒绝原因清晰可重现 | 中——拒绝原因依赖当时的上下文状态 |
| **误报率** | 可能偏高（为了安全牺牲可用性） | 可控（可以在负载低时放宽限制） |

### 混合策略建议

生产环境中不应该是非此即彼的选择，而应是混合策略：

```
混合边界策略：

┌──────────────────────────────────────────────┐
│ 硬底线（Hard Floor）——始终生效的静态边界       │
│  • 任何时候不突破：                           │
│    - 模型上下文窗口的 95%                      │
│    - 单个文件 50MB                            │
│    - 单次请求 100 次工具调用                   │
│    - 每秒 10 个请求（系统级）                  │
│  这些是"系统生存底线"，不受任何动态调整影响      │
├──────────────────────────────────────────────┤
│ 弹性区间（Elastic Range）——动态调整的区域       │
│  • 在硬底线之内动态浮动：                      │
│    - 用户输入 Token 预算：4K-100K             │
│    - 工具调用次数：5-50                       │
│    - 会话长度：10-200 轮                      │
│    - 文件 Token：10K-100K                    │
│  根据用户等级、系统负载、任务复杂度在区间内浮动   │
├──────────────────────────────────────────────┤
│ 优先级抢占（Priority Preemption）              │
│  • 当系统高负载时：                            │
│    1. 付费用户 → 保持弹性上限                  │
│    2. 免费用户 → 降低到弹性下限                │
│    3. 高安全任务 → 使用硬底线                  │
│    4. 后台任务 → 延迟执行                      │
└──────────────────────────────────────────────┘
```

### 何时选择哪种策略

```
选择静态边界：
  - 系统资源极其有限（边缘设备部署）
  - 安全合规要求严格的场景（金融交易、医疗诊断）
  - 需要精确计费的场景（按 Token 计费的 API）
  - 系统启动初期（尚未积累足够的数据做动态调整）

选择动态边界：
  - 用户体验是首要指标
  - 资源充足且有弹性伸缩能力（云部署）
  - 存在用户等级差异化需求（免费 vs 付费）
  - 系统已运行一段时间，有负载数据支持动态调整

选择混合边界：
  - 所有生产环境推荐的默认选择
  - 任何需要平衡安全与体验的场景
```

---

## Engineering Optimization

边界检查的工程化实践——不只在代码层面实现边界检查，还要从系统架构层面优化。

### 1. Pre-Flight Checks — 预检机制

在输入进入 Agent 核心处理流程之前，先做"轻量预检"，快速拒绝明显不合法的请求。预检的核心价值是：**让高频边界检查在 1ms 内完成，避免将非法请求送入 LLM 推理管道浪费资源。**

```python
class PreFlightChecker:
    """
    预检器——在 Agent 主流程之前快速检查
    设计目标：所有预检合计 < 5ms，不调用外部服务
    """

    @staticmethod
    def pre_flight(user_input: str, files: Optional[List] = None) -> Optional[str]:
        """
        返回 None 表示通过预检
        返回 str 表示拒绝原因

        O(1) 或 O(n) 但 n 很小——不做任何复杂计算
        """
        # 1. 空输入检查（O(1)）
        if not user_input or not user_input.strip():
            return "输入不能为空"

        # 2. 极短检查（O(1)）
        if len(user_input.strip()) < 2:
            return "输入过短，请提供更详细的内容"

        # 3. 极端长度检查（O(1)——检查长度属性）
        if len(user_input) > 100_000:  # 100K 字符 ≈ 25K tokens 粗略估
            return "输入过长，请精简或分段处理"

        # 4. 文件检查（O(files)）
        if files:
            total_size = sum(f.get("size", 0) for f in files)
            if total_size > 200 * 1024 * 1024:  # 200MB
                return "附件总大小超过限制（200MB）"
            for f in files:
                if f.get("size", 0) > 50 * 1024 * 1024:  # 50MB
                    return f"单个文件 {f.get('name', '')} 超过限制（50MB）"

        # 5. 高频请求快速缓存检查
        # （使用 Redis/local cache 检查该用户是否被临时限流）
        # 此处省略具体实现

        return None  # 通过预检


# 在 Agent 入口处调用
def agent_endpoint(user_id: str, user_input: str, files: Optional[List] = None):
    # Step 0: Pre-flight (1-2ms)
    rejection = PreFlightChecker.pre_flight(user_input, files)
    if rejection:
        return {"error": rejection, "code": 400}

    # Step 1: 完整边界检查 (10-20ms)
    # ...
```

### 2. Progressive Loading — 渐进式加载

对于大输入（长文档、大文件），不一次性加载全部内容，而是按需、分批加载到边界检查器中。核心价值：**对于超大输入，如果前 10% 已经触发了边界拒绝，不需要加载剩余 90%。**

```python
class ProgressiveBoundaryLoader:
    """
    渐进式边界检查加载器
    按批次加载输入内容，逐批检查边界
    一旦某一批次触发拒绝，立即停止加载
    """

    def __init__(self, chunk_size: int = 4000):
        self.chunk_size = chunk_size   # 每批字符数（≈ 1000 tokens）
        self.loaded_chars = 0
        self.total_tokens = 0
        self.is_rejected = False

    def load_and_check(self, text: str, max_tokens: int) -> Tuple[bool, Optional[str]]:
        """
        逐批加载文本并检查边界
        返回 (accepted, rejection_reason)
        """
        for i in range(0, len(text), self.chunk_size):
            chunk = text[i:i + self.chunk_size]
            self.loaded_chars += len(chunk)
            self.total_tokens += count_tokens(chunk)

            # 每加载一批就检查一次
            if self.total_tokens > max_tokens:
                self.is_rejected = True
                return False, f"Token 预算超限（加载到 {self.total_tokens} tokens 时拒绝）"

            # 可以在这里插入其他渐进检查
            # 例如：检查到某处出现极端嵌套结构，即使只加载了 20% 也拒绝

        return True, None
```

### 3. Streaming Validation — 流式验证

对于流式输入（SSE、WebSocket 持续接收的输入），在数据流到达时实时验证边界。核心价值：**不等待输入完整接收就开始验证，减少端到端延迟。**

```python
class StreamingBoundaryValidator:
    """
    流式边界验证器
    对持续到达的输入数据进行实时边界检查
    """

    def __init__(self, config: BoundConfig):
        self.config = config
        self.accumulated_text = ""
        self.token_count = 0
        self.line_count = 0
        self.nesting_depth = 0
        self.max_depth_seen = 0

    def on_data_chunk(self, chunk: str) -> Optional[BoundCheckResult]:
        """
        每收到一个数据块时调用
        返回 None 表示继续；返回 BoundCheckResult 表示边界问题
        """
        self.accumulated_text += chunk
        self.token_count = count_tokens(self.accumulated_text)

        # 1. Token 预算检查（每次收到 chunk 时检查）
        if self.token_count > self.config.max_message_chars / 4:
            return BoundCheckResult(
                boundary=BoundaryType.TOKEN_BUDGET,
                severity=Severity.REJECT,
                message=f"流式输入已累计 {self.token_count} tokens，超出预算",
                suggested_action="停止发送剩余内容"
            )

        # 2. 嵌套深度检查（实时检测 JSON 流）
        for char in chunk:
            if char == '{' or char == '[':
                self.nesting_depth += 1
                self.max_depth_seen = max(self.max_depth_seen, self.nesting_depth)
            elif char == '}' or char == ']':
                self.nesting_depth -= 1

        if self.max_depth_seen > self.config.max_nesting_depth:
            return BoundCheckResult(
                boundary=BoundaryType.RANGE,
                severity=Severity.REJECT,
                message=f"嵌套深度 {self.max_depth_seen} 超过上限 {self.config.max_nesting_depth}",
                suggested_action="简化 JSON 结构"
            )

        return None  # 继续接收

    def finalize(self) -> List[BoundCheckResult]:
        """流结束后做最终检查"""
        results = []

        # 换行数检查
        self.line_count = self.accumulated_text.count("\n") + 1
        if self.line_count > self.config.max_message_lines:
            results.append(BoundCheckResult(
                boundary=BoundaryType.LENGTH,
                severity=Severity.WARN,
                message=f"行数 {self.line_count} > {self.config.max_message_lines}"
            ))

        return results
```

### 4. 边界检查缓存

避免对重复输入重复执行相同的边界检查：

```python
class BoundaryCache:
    """
    边界检查结果缓存
    对于同一用户的相同输入（hash 匹配），跳过重复的边界计算
    特别适合工具返回结果这类重复量大的内容
    """

    def __init__(self, ttl_seconds: int = 300):
        self.cache: Dict[str, Tuple[float, List[BoundCheckResult]]] = {}
        self.ttl = ttl_seconds

    def get_cached(self, content: str, source: SourceTier) -> Optional[List[BoundCheckResult]]:
        """获取缓存的边界检查结果"""
        key = f"{source.value}:{hash(content)}"
        if key in self.cache:
            timestamp, results = self.cache[key]
            if time.time() - timestamp < self.ttl:
                return results
            del self.cache[key]
        return None

    def set_cached(self, content: str, source: SourceTier, results: List[BoundCheckResult]):
        """缓存边界检查结果"""
        key = f"{source.value}:{hash(content)}"
        self.cache[key] = (time.time(), results)
```

### 5. 边界检查的 Observability

边界检查本身需要可观测性——知道它拒绝了什么、为什么拒绝、是否误伤：

```python
class BoundaryObservability:
    """
    边界检查的可观测性指标
    用于监控边界检查的运行状态和调试误报/漏报
    """

    def __init__(self):
        self.metrics = {
            "checks_total": 0,
            "checks_passed": 0,
            "checks_rejected": {bt.name: 0 for bt in BoundaryType},
            "checks_warned": {bt.name: 0 for bt in BoundaryType},
            "rejection_reasons": {},        # 拒绝原因分布
            "avg_check_time_ms": 0.0,       # 平均检查时间
            "adaptive_multipliers": [],      # 自适应乘数分布
        }

    def record_check(self, result: BoundCheckResult, duration_ms: float):
        """记录一次边界检查的结果和耗时"""
        self.metrics["checks_total"] += 1

        if result.severity == Severity.PASS:
            self.metrics["checks_passed"] += 1
        elif result.severity == Severity.REJECT or result.severity == Severity.FATAL:
            self.metrics["checks_rejected"][result.boundary.name] += 1
            reason = result.message[:50]
            self.metrics["rejection_reasons"][reason] = \
                self.metrics["rejection_reasons"].get(reason, 0) + 1
        elif result.severity == Severity.WARN:
            self.metrics["checks_warned"][result.boundary.name] += 1

        # 更新平均耗时（指数移动平均）
        alpha = 0.1
        self.metrics["avg_check_time_ms"] = (
            (1 - alpha) * self.metrics["avg_check_time_ms"]
            + alpha * duration_ms
        )

    def get_report(self) -> Dict:
        """生成边界检查可观测性报告"""
        total = self.metrics["checks_total"]
        rejection_count = sum(self.metrics["checks_rejected"].values())

        return {
            "total_checks": total,
            "pass_rate": f"{self.metrics['checks_passed'] / total * 100:.1f}%" if total else "N/A",
            "rejection_rate": f"{rejection_count / total * 100:.1f}%" if total else "N/A",
            "top_rejection_reasons": sorted(
                self.metrics["rejection_reasons"].items(),
                key=lambda x: x[1],
                reverse=True
            )[:5],
            "avg_check_time_ms": f"{self.metrics['avg_check_time_ms']:.2f}",
            "boundary_breakdown": {
                bt.name: {
                    "rejected": self.metrics["checks_rejected"][bt.name],
                    "warned": self.metrics["checks_warned"][bt.name],
                }
                for bt in BoundaryType
            },
        }
```

### 6. 边界检查在生产环境中的部署位置

```
           ┌─────────────┐
           │ 客户端        │
           └──────┬──────┘
                  │ 用户请求
                  ▼
           ┌─────────────┐
           │ API Gateway  │ ← Pre-flight 预检（最轻量）
           │              │   长度、大小、频率
           └──────┬──────┘
                  ▼
           ┌─────────────┐
           │ Agent 服务    │ ← 完整边界检查
           │              │   Token 预算、层级、资源
           │              │   自适应计算
           └──────┬──────┘
                  │
         ┌────────┴────────┐
         │                 │
         ▼                 ▼
   ┌────────────┐   ┌────────────┐
   │ LLM 推理    │   │ 工具执行    │
   │ (第三方 API) │   │ (内部服务)  │
   └────────────┘   └──────┬─────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ 工具返回检查    │ ← 对工具返回结果做边界检查
                    │              │   防止工具返回超大内容
                    └──────────────┘

关键部署原则：
1. 越靠近客户端 → 检查越轻量
2. 越靠近核心 → 检查越全面
3. 不要将边界检查部署在 LLM API 调用之后（太晚了）
4. 工具返回也需要边界检查（外部结果不可信）
```

### 7. 边界配置的管理

边界配置应该像代码一样被管理——版本化、可审计、可测试：

```python
"""
边界配置的最佳实践：

1. 配置即代码 (Config as Code)
   - 把边界配置放在版本控制中
   - 每次修改需要 code review
   - 配置变更需要回滚路径

2. 环境化配置
   - 开发环境: 宽松，方便调试
   - 测试环境: 和生产一致
   - 生产环境: 严格，有弹性余量

3. 运行时可调 (但不推荐)
   - 紧急情况下可以通过配置中心动态调整
   - 需要操作日志和审批流程
   - 调整后需要自动恢复（TTL 过期机制）

4. 定期复审
   - 每季度复审一次边界配置
   - 检查是否误伤了合法用户
   - 检查攻击者是否找到了绕过方式
"""
```

### 生产环境检查清单

```
边界检查生产准备检查清单：

基础检查:
  [ ] 所有用户输入是否都经过边界检查？（包括工具参数）
  [ ] 边界检查是否在 LLM 推理之前执行？
  [ ] 预检（Pre-flight）是否在 5ms 内完成？
  [ ] 完整边界检查是否在 50ms 内完成？
  [ ] 是否有边界检查失败的监控告警？

配置管理:
  [ ] 边界参数是否通过配置管理（而非硬编码）？
  [ ] 不同环境（dev/staging/prod）是否有单独的边界配置？
  [ ] 边界配置是否在版本控制中？
  [ ] 是否有边界配置变更的审计日志？

覆盖范围:
  [ ] Token 预算是否覆盖所有输入来源？
  [ ] 文件类型是否有单独的边界设置（压缩率、分辨率）？
  [ ] 工具返回结果是否也有边界检查？
  [ ] 流式输入是否支持渐进式边界验证？
  [ ] 多轮对话是否累积检查 Token 消耗？

安全加固:
  [ ] 边界配置本身是否有权限保护？
  [ ] 自适应边界的放宽上限是否有限制？
  [ ] 边界拒绝的日志是否包含足够信息用于事后审计？
  [ ] 是否有边界配置的"熔断"机制——当连续 N 次边界检查异常时自动收紧？
  [ ] 边界检查代码本身是否有 DoS 风险？（绕过边界检查本身）

测试验证:
  [ ] 是否有边界超限的自动化测试用例？
  [ ] 是否有边界附近的"临界值"测试？
  [ ] 是否有边界绕过的安全测试？
  [ ] 是否有大负载下的边界检查性能测试？
```
