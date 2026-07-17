# 12.4.7 Human-in-the-Loop (HITL) — 人工介入

---

## 简单介绍

Human-in-the-Loop (HITL) 是一种安全设计模式，在 AI Agent 执行关键或高风险操作之前，引入人类审查与决策环节。其核心理念是：**当系统不确定、风险过高、或需要价值判断时，将控制权交还给人类**。HITL 不是对 Agent 能力的否定，而是对 Agent 自主性的审慎约束——它承认当前 AI 系统在歧义场景、道德判断、和安全关键决策中仍需要人类作为最终的裁决者。

在现代 Agent 系统中，HITL 已从简单的 "确认/取消" 二元选择发展为包含分级审查、异步审批、智能触发、疲劳管理在内的完整制度体系。

---

## 基本原理

HITL 作为终极安全机制，其基本原理建立在以下几个认知之上：

**1. 人机互补原则**

| 维度 | 人类优势 | Agent 优势 |
|------|----------|------------|
| 判断力 | 歧义处理、价值权衡、上下文理解 | 规则执行、大规模搜索、一致性 |
| 速度 | 慢（秒级到分钟级） | 快（毫秒级） |
| 可靠性 | 易疲劳、注意力波动 | 持续稳定、可预测 |
| 成本 | 高 | 低 |

HITL 让人类做人类擅长的事（判断、权衡、例外处理），让 Agent 做 Agent 擅长的事（执行、搜索、重复性工作）。

**2. 风险分层原理**

并非所有操作都需要人工介入。HITL 的价值在于按风险等级分配审查资源：

- **低风险**（读取操作、非敏感查询）→ 完全自主
- **中风险**（写入非关键数据、发送消息）→ 异步审查或可跳过确认
- **高风险**（删除数据、修改权限、财务操作）→ 同步人工确认
- **极高风险**（批量删除、系统配置变更）→ 多人审批 + 冷却期

**3. 不确定性触发**

Agent 应当在以下情形主动请求人工介入：
- 置信度低于阈值
- 指令存在歧义
- 操作影响超出预期范围
- 涉及安全、隐私、合规的边界情况
- 检测到异常状态或对抗性输入

**4. 最小必要干预原则**

HITL 设计应追求 "恰到好处的介入"——过多的人工干预会抵消 Agent 自动化的价值，过少则使安全机制形同虚设。良好的 HITL 系统应该让人类在**关键节点**介入，而非在每个步骤都打断。

---

## 背景

HITL 在软件系统中的应用经历了四个阶段的演进：

### 第一阶段: CLI Confirm（命令行确认）

最早的形态。CLI 工具在执行破坏性操作前打印 "Are you sure? [y/N]"，用户通过标准输入确认。

```
$ rm -rf /project
Are you sure? [y/N] y
```

- 优点: 实现简单，零依赖
- 缺点: 阻塞式，无上下文，无分级，不适用于 GUI/Agent 场景

### 第二阶段: GUI Approval（图形化审批）

随着 Web 应用和图形界面普及，HITL 进化为模态对话框、审批工作流。典型例子包括：

- GitLab/GitHub 的 Merge Request 审批
- AWS IAM 的权限变更审批流
- 支付系统的二次确认弹窗

- 优点: 丰富的上下文展示，支持多人审批
- 缺点: 模态阻断用户体验，审批流程僵化

### 第三阶段: Inline Agent HITL（内嵌 Agent 人工介入）

Agent 系统兴起后，HITL 被嵌入到 Agent 执行流程中。Agent 不再等待阻塞式用户输入，而是将需要确认的操作发送到审查队列，人类可以在方便时统一处理。

代表系统：
- Claude Code 的 permission prompt（权限提示）
- AutoGPT 的 human input mode
- LangChain 的 `HumanApprovalCallbackHandler`

- 优点: 非阻塞，与 Agent 执行流深度融合
- 缺点: 上下文断裂，人类可能不理解 Agent 请求的背景

### 第四阶段: Adaptive HITL（自适应人工介入）

当前演进方向。系统根据操作风险、人类当前状态、历史决策模式动态调整 HITL 触发策略：

- 学习用户偏好：哪些操作该问，哪些不需要问
- 上下文感知：同一操作在不同上下文中风险不同
- 状态感知：人类繁忙时自动降级为异步审查
- 预测式介入：在操作执行前预测是否需要人工介入

---

## 核心矛盾

HITL 系统的根本矛盾在于 **人类可用性与 Agent 自主性之间的张力**：

### 1. 可用性 vs 自主性

- Agent 追求快速、连续执行
- 人类响应存在延迟（几十秒到几小时不等）
- 每一次 HITL 等待都可能打断 Agent 的工作流

### 2. 延迟与吞吐量

- 同步 HITL：每个操作等待数秒到数分钟，Agent 吞吐量骤降
- 异步 HITL：延迟被吸收但反馈回路拉长，错误可能累积
- 批处理 HITL：效率最高但失去了实时干预的意义

### 3. 审查疲劳

- "确认疲劳"：过多的低价值确认提示导致人类机械式通过
- "警报疲劳"：过多的安全告警导致人类忽视真正的危险信号
- "认知负荷"：每次审查需要切换上下文，消耗认知资源

### 4. 技能退化

- 长期依赖 HITL 的人类审查者可能丧失独立判断能力
- 过度自动化使人类对系统行为的理解退化
- 当真正需要人类判断时，人类可能已经无法胜任

### 5. 责任归属模糊

- Agent 犯错时，是人没有审查到位还是 Agent 执行有误？
- 快速审批模式下，人类是否真正 "知情同意"？
- 法律合规场景中，HITL 能否真正转移责任？

### 权衡曲线

```
自主性
   ^
   |            · 完全自主
   |         ·  (高风险决策)
   |       ·
   |     ·  理想平衡点
   |   ·
   | ·            · 完全人工
   |               (手动操作)
   +-------------------------> 安全性
```

太偏向自主性的系统不安全，太偏向人工的系统不高效。HITL 设计的目标是在这条曲线上找到最适合业务场景的点。

---

## 详细内容

### 1. HITL Design Patterns（设计模式）

#### 1.1 Confirmation Gate（确认门）
最简单的模式。Agent 在执行操作前暂停，等待人类确认。

```
Agent: "准备删除用户 'john_doe' 及其所有数据 (3.2GB)"
Human: [Approve] [Reject] [Modify]
```

**适用场景**：高风险单步操作、不可逆操作、首次执行的操作。

**实现要点**：
- 展示操作的全部影响范围
- 提供 undo 信息（如果可能）
- 给出操作建议（如 "建议先备份"）

#### 1.2 Approval Queue（审批队列）
Agent 将需要审批的操作放入队列，人类审查者按优先级处理。

```
审批队列:
  [P0] 删除生产数据库表 `orders` — 等待 张三 审批
  [P1] 创建 500 个新用户账号 — 等待 李四 审批
  [P2] 修改 CI/CD 配置 — 等待 王五 审批
```

**适用场景**：批量操作、异步工作流、多审查者轮值。

**实现要点**：
- 支持优先级排序
- 显示队列长度和平均处理时间
- 支持批量审批（同类型低风险操作）

#### 1.3 Exception Handling（异常处理）
Agent 自主执行正常流程，仅在异常状况下请求人工介入。

```
正常流程: Agent 自主执行所有操作
异常触发: 操作结果超出预期范围 → 暂停并请求人类判断
异常恢复: 人类给出决策 → Agent 继续或回滚
```

**适用场景**：常规操作高度可预测、异常处理需要人类判断。

**实现要点**：
- 定义清晰的异常触发条件
- 提供完整的异常上下文
- 支持回滚或替代路径

#### 1.4 Escalation Path（升级路径）
当初级审查者无法决策或不在线时，逐级向上升级。

```
Level 1: 值班工程师 — 响应超时 5 分钟
    ↓
Level 2: 高级工程师 — 响应超时 15 分钟
    ↓
Level 3: 技术主管 — 响应超时 30 分钟
    ↓
Level 4: 安全负责人 — 响应超时 1 小时 → 自动阻断操作
```

**适用场景**：24/7 运行系统、事故响应、安全关键系统。

**实现要点**：
- 每一级都有明确的时间预算
- 升级时附带累积的上下文
- 最终兜底策略（自动拒绝/自动阻断）

---

### 2. Interruption Modes（中断模式）

#### 2.1 Synchronous（同步 — 等待人类）
Agent 完全暂停执行，等待人类做出决策后才继续。

```
Agent → 发送审批请求 → ⏸ 暂停 → 人类响应 → 继续/中止
```

| 特性 | 说明 |
|------|------|
| 延迟 | 取决于人类响应速度（秒~小时） |
| 安全性 | 最高 — 人类在操作前介入 |
| 吞吐量 | 低 — Agent 被阻塞 |
| 适用场景 | 高风险操作、不可逆操作 |

**实现**:
```python
result = await hitl.request_approval(
    action="DELETE_USER",
    context={"user_id": "u_12345", "data_size": "3.2GB"},
    mode=InterruptionMode.SYNCHRONOUS,
    timeout=timedelta(minutes=5)
)
# Agent 在此等待...
if result.decision == Decision.APPROVE:
    await execute_delete()
```

#### 2.2 Asynchronous（异步 — 排队审查）
Agent 继续执行其他任务，审批请求进入队列，人类在方便时审查。

```
Agent → 发送审批请求 → 继续执行其他任务
                             ↕ 异步
人类 → 查看队列 → 审批/拒绝 → Agent 收到结果后处理
```

| 特性 | 说明 |
|------|------|
| 延迟 | 不影响 Agent 主流程（反馈有延迟） |
| 安全性 | 中等 — 操作可能在审批完成前已执行 |
| 吞吐量 | 高 — Agent 不被阻塞 |
| 适用场景 | 低风险批量操作、可回滚操作 |

**设计要点**：
- 区分 "预先审批"（操作前）和 "事后审计"（操作后）
- 异步审批的操作应可回滚
- 提供清晰的状态追踪

#### 2.3 Deferred（延迟审查 — 事后审计）
Agent 完全自主执行，操作日志提交事后审查。

```
Agent → 执行操作 → 记录日志 → 审查者事后检查
                              ↕
                    发现问题 → 回滚/修复
```

| 特性 | 说明 |
|------|------|
| 延迟 | 无 — Agent 完全自由 |
| 安全性 | 最低 — 事后补救 |
| 吞吐量 | 最高 |
| 适用场景 | 低风险、可完全回滚、已有成熟审计机制 |

**适用条件**：
- 操作可逆
- 有完善的变更追踪
- 有自动化异常检测
- 审查周期短于风险暴露窗口

---

### 3. Context Presentation（上下文呈现）

人类审查者需要足够的上下文来做出知情决策，但过多的信息会导致认知过载。**最小充分上下文原则**：只展示做出当前决策所需的最少信息。

#### 3.1 必须呈现的信息

```
┌─────────────────────────────────────────────────────────────┐
│  审批请求 #A-20240715-003                                   │
│                                                             │
│  ⚠ 高风险操作                                               │
│                                                             │
│  操作: 删除 AWS S3 Bucket                                   │
│  目标: prod-data-archive (us-east-1)                        │
│  影响范围:                                                  │
│    • 对象数量: 1,247,892                                    │
│    • 总大小: 3.2 TB                                         │
│    • 最后修改时间: 2024-07-14 23:59:59 UTC                   │
│                                                             │
│  触发原因: 磁盘清理策略自动触发                              │
│  建议: 该 Bucket 已超过 90 天无读取访问                     │
│                                                             │
│  相似操作历史: 本月已执行 23 次同类操作，0 次回滚           │
│                                                             │
│  [批准]  [拒绝]  [修改参数]  [查看详情]  [升级]             │
└─────────────────────────────────────────────────────────────┘
```

**核心信息层级**:
1. **风险标识** — 一眼看清风险等级
2. **操作描述** — 什么操作 + 作用于什么目标
3. **影响范围** — 量化影响（数量、金额、用户数等）
4. **触发原因** — 为什么 Agent 要执行此操作
5. **决策支持** — 辅助人类决策的信息（建议、历史统计）
6. **操作选项** — 明确的行动按钮

#### 3.2 可选增强信息

- **Diff View**: 操作前后的状态对比
- **时间线**: 导致此操作的一系列事件
- **相似案例**: 历史上类似操作的后果
- **政策引用**: 相关策略规则的原文
- **风险评估**: 操作的自动风险评估分数

#### 3.3 注意力引导

使用视觉优先级引导人类注意力到最关键的信息：

```
[P0] ⚠️ 高危 — 将删除 3.2TB 生产数据
[P1] ⚠ 触发原因不明确
[P2] ℹ️ 上周有类似的成功操作
```

---

### 4. Human Response Options（人类响应选项）

人类审查者在收到审批请求时，应该有丰富的响应选项，而不是简单的 "是/否"。

#### 4.1 Standard Options

| 选项 | 含义 | 语义 |
|------|------|------|
| **Approve** | 批准执行 | "继续，我确认这是正确的" |
| **Reject** | 拒绝执行 | "停止，这个操作不应该执行" |
| **Modify** | 修改后执行 | "可以执行，但参数需要调整" |
| **Defer** | 暂时搁置 | "现在不确定，稍后重新审查" |
| **Escalate** | 升级决策 | "我无权/无法决定，找更合适的人" |

#### 4.2 Advanced Options

| 选项 | 含义 |
|------|------|
| **Approve Once** | 仅此一次批准，不改变未来行为 |
| **Approve Always** | 对此类操作永久授权（学习） |
| **Approve with Conditions** | 附加条件（如 "仅在非工作时间执行"）|
| **Approve for Session** | 当前会话内自动批准同类操作 |
| **Request More Info** | 需要更多上下文才能决策 |
| **Rollback** | 拒绝并回滚已执行的相关操作 |
| **Override with Command** | 直接输入替代指令 |

#### 4.3 决策记录

每次人类响应都应记录：
- 决策时间
- 决策者身份
- 决策结果
- 决策原因（可选输入）
- 审查时长
- 关联上下文快照

---

### 5. Escalation Chains（升级链）

#### 5.1 设计原则

- **明确的升级条件**: 超时、拒绝、请求升级
- **逐级递进**: 每次升级扩大范围和权限
- **熔断机制**: 达到升级链末端后自动触发兜底策略
- **上下文累积**: 每次升级附带完整的决策历史

#### 5.2 典型升级链

```
人类审查者 1（值班）→ 人类审查者 2（高级）→ 人类审查者 3（主管）→ 自动阻断
     ↓           ↓               ↓               ↓
   5min         15min           30min           60min
```

```
┌──────────────┐  超时    ┌──────────────┐  超时    ┌──────────────┐
│ 初级审查者    │ ───────→ │ 高级审查者    │ ───────→ │ 安全主管      │
│ 响应窗口: 5m │          │ 响应窗口: 15m │          │ 响应窗口: 30m │
└──────────────┘          └──────────────┘          └──────┬───────┘
                                                            │ 超时
                                                            ↓
                                                     ┌──────────────┐
                                                     │ 自动拒绝      │
                                                     │ + 阻断同类操作 │
                                                     └──────────────┘
```

#### 5.3 升级触发条件

1. **超时**: 当前审查者在窗口期内未响应
2. **拒绝授权**: 审查者认为自己无权批准
3. **上下文越界**: 操作涉及超出当前审查者权限范围
4. **冲突检测**: 同一操作收到矛盾决策
5. **风险升级**: 自动风险评分在审查期间升高

#### 5.4 紧急联系人

对于 P0 级事件，应配置：
- 主联系人（Primary）
- 备用联系人（Secondary）
- 紧急联系人（Emergency）
- 自动阻断策略（最终兜底）

支持多通道通知：邮件、短信、即时消息、电话。

---

### 6. HITL for Multi-Agent Systems（多 Agent 系统中的人工介入）

多 Agent 系统引入额外的 HITL 复杂性——不仅需要人与 Agent 之间的交互，还需要理解 Agent 之间的交互。

#### 6.1 监督者 Agent（Supervisor Agent）

引入一个专门的监督者 Agent 作为人类审查者的代理：

```
                 ┌──────────────────┐
                 │   人类审查者      │
                 └────────┬─────────┘
                          │ 审批/拒绝/修改
                          ↓
                 ┌──────────────────┐
                 │   Supervisor      │
                 │   Agent           │ ← 策略配置、风险规则
                 └──┬────┬────┬─────┘
                    │    │    │
              ┌─────┘    │    └─────┐
              ↓          ↓          ↓
         ┌────────┐ ┌────────┐ ┌────────┐
         │ Agent A│ │ Agent B│ │ Agent C│
         └────────┘ └────────┘ └────────┘
```

**Supervisor Agent 职责**:
- 监控子 Agent 的行为和决策
- 在风险操作前发起 HITL 请求
- 聚合多个 Agent 的上下文呈现给人类
- 在紧急情况下执行阻断

#### 6.2 分布式 HITL

在多 Agent 系统中，HITL 需要在多个层面实施：

| 层级 | HITL 策略 | 示例 |
|------|-----------|------|
| 单个 Agent | 操作级审批 | 每个 Agent 在执行写操作前请求审批 |
| Supervisor | 策略级审批 | 子 Agent 变更协作策略时需审批 |
| Orchestrator | 流程级审批 | 整个工作流的关键里程碑需审批 |
| 全局 | 熔断级审批 | 异常行为检测后全局暂停 |

#### 6.3 多 Agent 冲突时的 HITL

当两个 Agent 做出矛盾决策时：
1. Supervisor Agent 尝试自动仲裁
2. 自动仲裁失败 → 提交人类审查
3. 人类做出最终裁决
4. 裁决结果反馈为策略规则，防止再次冲突

#### 6.4 跨 Agent 上下文聚合

人类审查者面对多个 Agent 时，需要聚合的上下文视图：
- 谁发起了这个操作？哪个 Agent？
- 这个操作在整个工作流中的位置？
- 其他 Agent 对此操作的意见？
- 如果批准/拒绝，对其他 Agent 的影响？

---

### 7. HITL Fatigue Prevention（审查疲劳预防）

审查疲劳是 HITL 系统在实践中最大的挑战之一。

#### 7.1 疲劳的表现

- **自动化偏见**: 倾向于批准 Agent 的请求，逐渐丧失批判性思考
- **注意力下降**: 审查时间越来越短，忽略关键细节
- **决策质量波动**: 疲劳时更容易做出错误决策
- **规避行为**: 审查者开始忽略通知或延迟响应

#### 7.2 预防策略

**工作量平衡**:
- 轮值制：多人轮流担任审查者
- 限额定：每个审查者每日最大审查次数
- 强制休息：连续审查超过阈值后强制暂停
- 负载感知：根据当前审查队列长度动态调整

**自动审批（Auto-Approve）**:
- 低风险操作自动通过（不发起 HITL）
- 操作模式匹配时自动通过（学习人类决策模式）
- 特定时间窗口内自动通过（如夜间低风险维护）

**批量审查（Batch Review）**:
```
今日待审查操作 (15 项)
┌──────────────────────────────────────┐
│ ☑ 删除临时文件 tmp_*.log (5次同类)   │ → 全部批准
│ ☑ 更新 DNS 记录 (3次)               │ → 全部批准
│ ☐ 修改数据库索引 schema_v2          │ → 需要单独审查
│ ☐ 部署到生产环境                     │ → 需要单独审查
└──────────────────────────────────────┘
```

**智能优先级**:
- 高风险的请求排在队列前面
- 同类请求聚合展示
- 基于审查者专长分配

**反馈闭环**:
- 向审查者展示其决策的影响（正面和负面）
- "你上周批准的 23 个操作全部成功完成"
- "你昨天拒绝的操作导致了 X 问题"

#### 7.3 疲劳监测指标

| 指标 | 正常范围 | 警示范围 | 危险范围 |
|------|---------|---------|---------|
| 单次审查时长 | 5-60s | <3s 或 >5min | <1s |
| 批准率 | 70-95% | >98% 或 <50% | 100% |
| 连续审查数 | <20 | 20-50 | >50 |
| 决策修正率 | <5% | 5-15% | >15% |
| 响应时间趋势 | 稳定 | 逐渐加快/减慢 | 突变 |

---

### 8. The "Human Buffer"（人类缓冲区）

不让 HITL 成为瓶颈，而是将其设计为一个缓冲层。

#### 8.1 缓冲设计理念

```
无缓冲: Agent → [等待人类] → 执行
有缓冲: Agent → [队列/缓存] → [人类审查] → 执行反馈
```

缓冲层吸收人类响应延迟的波动，确保 Agent 端体验稳定。

#### 8.2 实现策略

**预执行缓冲**:
```
Agent 生成执行计划 → 放入缓冲队列 → 继续工作
                           ↓
人类审查 → 批准 → Agent 执行
         → 拒绝 → Agent 回滚/调整
         → 修改 → Agent 调整后执行
```

**后执行缓冲（审计模式）**:
```
Agent 执行 → 记录到审计日志 → 继续工作
                    ↓
人类审查 → 无问题 → 归档
         → 有问题 → 回滚/修复
```

**乐观执行 + 取消**:
```
Agent 执行低风险操作 → 人类异步审查 → 发现问题 → 取消/回滚
```

#### 8.3 缓冲容量管理

- 队列深度限制：缓冲队列最大深度
- 积压告警：队列长度超过阈值时告警
- 自动降级：积压严重时自动降低 Agent 自主性
- 弹性扩容：增加审查者以应对高峰

#### 8.4 避免成为瓶颈的关键实践

1. **不要等人类做 Agent 能做的事** — 只有需要人类判断时才发起 HITL
2. **预测式缓冲** — 预感可能需要审批时提前发起
3. **并行审批** — 多个独立操作可同时分配给不同审查者
4. **结果缓存** — 相同操作的历史审批结果可复用
5. **渐进式授权** — 从逐次审批到会话级授权到策略级授权

---

## Example Code: Python HITLManager

以下是一个完整的 HITL 管理器实现，包含同步/异步模式、升级链和上下文呈现。

```python
"""
HITL Manager — Human-in-the-Loop 管理系统
支持同步/异步审批、升级链、上下文呈现和疲劳监测
"""

from __future__ import annotations

import abc
import asyncio
import enum
import json
import logging
import time
import uuid
from dataclasses import dataclass, field, asdict
from datetime import datetime, timedelta
from enum import Enum
from typing import Any, Callable, Coroutine, Optional

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("hitl")


# ──────────────────────────────────────────────
# 数据类型定义
# ──────────────────────────────────────────────

class RiskLevel(Enum):
    """风险等级"""
    LOW = "low"          # 完全自主
    MEDIUM = "medium"    # 异步审查
    HIGH = "high"        # 同步审查
    CRITICAL = "critical"  # 多人审批 + 冷却期


class InterruptionMode(Enum):
    """中断模式"""
    SYNCHRONOUS = "synchronous"    # 等待人类
    ASYNCHRONOUS = "asynchronous"  # 排队审查
    DEFERRED = "deferred"          # 事后审计


class Decision(Enum):
    """人类决策"""
    APPROVE = "approve"
    REJECT = "reject"
    MODIFY = "modify"
    DEFER = "defer"
    ESCALATE = "escalate"


@dataclass
class ActionContext:
    """操作上下文 — 呈现给人类审查者的信息"""
    action_id: str = field(default_factory=lambda: uuid.uuid4().hex[:12])
    action_type: str = ""          # 操作类型，如 "DELETE_USER"
    target: str = ""                # 操作目标
    risk_level: RiskLevel = RiskLevel.MEDIUM
    description: str = ""           # 人类可读的描述
    impact: dict[str, Any] = field(default_factory=dict)  # 量化影响
    trigger_reason: str = ""        # 触发原因
    suggested_action: str = ""      # 建议操作
    related_history: list[dict] = field(default_factory=list)
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat())
    metadata: dict[str, Any] = field(default_factory=dict)

    def summarize(self) -> str:
        """生成人类可读的摘要"""
        lines = [
            f"⚠ 操作: {self.action_type}",
            f"  目标: {self.target}",
            f"  风险: {self.risk_level.value.upper()}",
            f"  描述: {self.description}",
        ]
        if self.impact:
            lines.append(f"  影响: {json.dumps(self.impact, ensure_ascii=False)}")
        if self.trigger_reason:
            lines.append(f"  触发: {self.trigger_reason}")
        return "\n".join(lines)


@dataclass
class ApprovalRequest:
    """审批请求"""
    id: str = field(default_factory=lambda: uuid.uuid4().hex[:12])
    context: ActionContext | None = None
    mode: InterruptionMode = InterruptionMode.SYNCHRONOUS
    created_at: float = field(default_factory=time.time)
    expires_at: float | None = None
    assigned_to: str | None = None
    escalation_level: int = 0  # 当前升级层级
    status: str = "pending"  # pending | approved | rejected | escalated | timed_out
    decision: Decision | None = None
    decision_reason: str = ""
    decided_by: str = ""
    decided_at: float | None = None


# ──────────────────────────────────────────────
# 审查者接口
# ──────────────────────────────────────────────

class Reviewer(abc.ABC):
    """审查者抽象基类"""

    @abc.abstractmethod
    async def review(self, request: ApprovalRequest) -> Decision:
        """审查一个审批请求"""
        ...

    @property
    @abc.abstractmethod
    def name(self) -> str:
        ...

    @property
    @abc.abstractmethod
    def level(self) -> int:
        """升级层级 (0, 1, 2, ...)"""
        ...


class HumanReviewer(Reviewer):
    """人类审查者 — 通过回调函数对接实际 UI"""

    def __init__(
        self,
        name: str,
        level: int = 0,
        send_fn: Callable[[ApprovalRequest], Coroutine[Any, Any, Decision]] | None = None,
    ):
        self._name = name
        self._level = level
        self._send_fn = send_fn
        self.review_count = 0
        self.fatigue_score = 0.0

    @property
    def name(self) -> str:
        return self._name

    @property
    def level(self) -> int:
        return self._level

    async def review(self, request: ApprovalRequest) -> Decision:
        """将审批请求发送给人类并等待响应"""
        self.review_count += 1
        if self._send_fn:
            logger.info(f"[HITL] 发送审批请求 {request.id} 给 {self._name}")
            decision = await self._send_fn(request)
            self._update_fatigue()
            return decision
        # 默认返回 DEFER 作为安全兜底
        logger.warning(f"[HITL] 审查者 {self._name} 未配置发送函数，自动 DEFER")
        return Decision.DEFER

    def _update_fatigue(self):
        """更新疲劳评分"""
        # 简单模型：连续审查超过阈值后疲劳度上升
        if self.review_count > 20:
            self.fatigue_score = min(1.0, (self.review_count - 20) / 30)

    def is_fatigued(self) -> bool:
        return self.fatigue_score > 0.7


class AutoReviewer(Reviewer):
    """自动审查者 — 基于规则自动决策，用于低风险操作"""

    def __init__(self, name: str = "auto-reviewer", level: int = -1):
        self._name = name
        self._level = level
        self.rules: list[tuple[str, Callable[[ActionContext], Decision | None]]] = []

    @property
    def name(self) -> str:
        return self._name

    @property
    def level(self) -> int:
        return self._level

    def add_rule(self, description: str, rule_fn: Callable[[ActionContext], Decision | None]):
        self.rules.append((description, rule_fn))

    async def review(self, request: ApprovalRequest) -> Decision:
        ctx = request.context
        if ctx is None:
            return Decision.DEFER

        for desc, rule_fn in self.rules:
            result = rule_fn(ctx)
            if result is not None:
                logger.info(f"[AutoReview] 规则 '{desc}' 命中: {result.value}")
                return result

        # 无规则匹配，不能自动决策
        return Decision.DEFER


# ──────────────────────────────────────────────
# 升级链配置
# ──────────────────────────────────────────────

@dataclass
class EscalationStep:
    """升级步骤"""
    reviewer: Reviewer
    timeout: timedelta  # 此步骤的超时时间
    on_timeout: Decision = Decision.ESCALATE  # 超时后的行为
    notify_channels: list[str] = field(default_factory=lambda: ["in_app"])


@dataclass
class EscalationPolicy:
    """升级策略"""
    steps: list[EscalationStep]
    final_action: Decision = Decision.REJECT  # 所有步骤都超时后的兜底决策
    final_action_reason: str = "所有审查者均超时未响应，自动拒绝"

    def get_step(self, level: int) -> EscalationStep | None:
        """获取指定层级的升级步骤"""
        if 0 <= level < len(self.steps):
            return self.steps[level]
        return None


# ──────────────────────────────────────────────
# HITL 管理器
# ──────────────────────────────────────────────

class HITLManager:
    """
    Human-in-the-Loop 管理器

    特性:
    - 同步/异步/延迟 三种中断模式
    - 可配置的升级链
    - 上下文自动呈现
    - 审查者疲劳监测
    - 决策历史记录
    """

    def __init__(self, escalation_policy: EscalationPolicy | None = None):
        self.escalation_policy = escalation_policy
        self._pending_requests: dict[str, ApprovalRequest] = {}
        self._decision_history: list[dict] = []
        self._auto_reviewer = AutoReviewer()
        self._setup_default_rules()

    def _setup_default_rules(self):
        """设置默认的自动审查规则"""
        # 只读操作自动批准
        self._auto_reviewer.add_rule(
            "只读操作自动批准",
            lambda ctx: Decision.APPROVE if ctx.action_type.startswith("READ_") else None
        )
        # 影响范围为 0 的低风险操作自动批准
        self._auto_reviewer.add_rule(
            "零影响操作自动批准",
            lambda ctx: (
                Decision.APPROVE
                if ctx.risk_level == RiskLevel.LOW
                and ctx.impact.get("count", 1) == 0
                else None
            )
        )

    async def request_approval(
        self,
        context: ActionContext,
        mode: InterruptionMode = InterruptionMode.SYNCHRONOUS,
        timeout: timedelta | None = None,
    ) -> ApprovalRequest:
        """
        请求人工审批

        Args:
            context: 操作上下文
            mode: 中断模式
            timeout: 超时时间 (同步模式默认 5 分钟，异步模式不超时)

        Returns:
            包含决策结果的 ApprovalRequest
        """
        request = ApprovalRequest(
            context=context,
            mode=mode,
        )

        # 设置超时
        if timeout:
            request.expires_at = time.time() + timeout.total_seconds()
        elif mode == InterruptionMode.SYNCHRONOUS:
            request.expires_at = time.time() + 300  # 默认 5 分钟

        # 尝试自动审查
        auto_decision = await self._auto_reviewer.review(request)
        if auto_decision != Decision.DEFER:
            request.status = "approved" if auto_decision == Decision.APPROVE else "rejected"
            request.decision = auto_decision
            request.decided_by = self._auto_reviewer.name
            request.decided_at = time.time()
            self._record_decision(request)
            logger.info(f"[HITL] 自动审查通过: {request.id}")
            return request

        # 根据模式处理
        if mode == InterruptionMode.DEFERRED:
            # 延迟审查：记录到审计日志，立即返回
            request.status = "deferred"
            self._pending_requests[request.id] = request
            logger.info(f"[HITL] 延迟审查已记录: {request.id}")
            return request

        if mode == InterruptionMode.ASYNCHRONOUS:
            # 异步审查：加入队列，立即返回
            request.status = "pending"
            self._pending_requests[request.id] = request
            logger.info(f"[HITL] 异步审批请求已入队: {request.id}")
            # 后台触发审批流程
            asyncio.create_task(self._run_approval_flow(request))
            return request

        # 同步模式：等待人类审批
        request.status = "pending"
        self._pending_requests[request.id] = request
        logger.info(f"[HITL] 同步审批请求: {request.id}")
        await self._run_approval_flow(request)
        return request

    async def _run_approval_flow(self, request: ApprovalRequest):
        """执行审批流程, 包含升级链"""
        policy = self.escalation_policy
        if policy is None:
            # 无升级策略则使用默认单步审批
            policy = EscalationPolicy(
                steps=[
                    EscalationStep(
                        reviewer=HumanReviewer(name="default-reviewer"),
                        timeout=timedelta(minutes=5),
                    )
                ]
            )

        current_level = request.escalation_level

        while True:
            step = policy.get_step(current_level)
            if step is None:
                # 所有层级已耗尽，执行兜底策略
                request.status = "timed_out"
                request.decision = policy.final_action
                request.decision_reason = policy.final_action_reason
                request.decided_at = time.time()
                self._record_decision(request)
                logger.warning(
                    f"[HITL] 升级链耗尽，兜底决策: {policy.final_action.value} | {request.id}"
                )
                break

            reviewer = step.reviewer

            # 检查审查者是否疲劳
            if hasattr(reviewer, "is_fatigued") and reviewer.is_fatigued():
                logger.warning(
                    f"[HITL] 审查者 {reviewer.name} 疲劳，自动升级"
                )
                current_level += 1
                request.escalation_level = current_level
                continue

            request.assigned_to = reviewer.name
            logger.info(
                f"[HITL] 升级层级 {current_level}: 分配给 {reviewer.name}"
            )

            try:
                decision = await asyncio.wait_for(
                    reviewer.review(request),
                    timeout=step.timeout.total_seconds(),
                )
            except asyncio.TimeoutError:
                logger.warning(
                    f"[HITL] 审查者 {reviewer.name} 超时 ({step.timeout})"
                )
                if step.on_timeout == Decision.ESCALATE:
                    current_level += 1
                    request.escalation_level = current_level
                    continue
                elif step.on_timeout == Decision.APPROVE:
                    decision = Decision.APPROVE
                else:
                    decision = Decision.REJECT

            # 处理决策结果
            if decision == Decision.ESCALATE:
                current_level += 1
                request.escalation_level = current_level
                continue

            request.status = "approved" if decision in (Decision.APPROVE, Decision.MODIFY) else "rejected"
            request.decision = decision
            request.decided_by = reviewer.name
            request.decided_at = time.time()
            self._record_decision(request)
            logger.info(
                f"[HITL] 决策完成: {request.id} → {decision.value} (by {reviewer.name})"
            )
            break

        return request

    def _record_decision(self, request: ApprovalRequest):
        """记录决策到历史"""
        self._decision_history.append({
            "request_id": request.id,
            "context": asdict(request.context) if request.context else None,
            "decision": request.decision.value if request.decision else None,
            "decided_by": request.decided_by,
            "decided_at": request.decided_at,
            "mode": request.mode.value,
            "escalation_level": request.escalation_level,
        })
        self._pending_requests.pop(request.id, None)

    def get_decision_history(
        self,
        limit: int = 50,
        action_type: str | None = None,
    ) -> list[dict]:
        """获取决策历史"""
        history = self._decision_history
        if action_type:
            history = [
                h for h in history
                if h.get("context", {}).get("action_type") == action_type
            ]
        return history[-limit:]

    def get_pending_requests(self) -> list[ApprovalRequest]:
        """获取待处理的审批请求"""
        return list(self._pending_requests.values())

    def get_fatigue_report(self) -> list[dict]:
        """获取审查者疲劳报告"""
        if not self.escalation_policy:
            return []
        report = []
        for step in self.escalation_policy.steps:
            reviewer = step.reviewer
            if hasattr(reviewer, "fatigue_score"):
                report.append({
                    "reviewer": reviewer.name,
                    "level": reviewer.level,
                    "review_count": reviewer.review_count,
                    "fatigue_score": reviewer.fatigue_score,
                    "is_fatigued": reviewer.is_fatigued(),
                })
        return report


# ──────────────────────────────────────────────
# 使用示例
# ──────────────────────────────────────────────

async def example_usage():
    """HITL 管理器使用示例"""

    # 1. 配置升级链
    primary = HumanReviewer(name="张三 (值班)", level=0)
    secondary = HumanReviewer(name="李四 (高级)", level=1)
    manager = HumanReviewer(name="王五 (主管)", level=2)

    escalation_policy = EscalationPolicy(
        steps=[
            EscalationStep(
                reviewer=primary,
                timeout=timedelta(seconds=30),  # 30 秒超时
            ),
            EscalationStep(
                reviewer=secondary,
                timeout=timedelta(seconds=30),
            ),
            EscalationStep(
                reviewer=manager,
                timeout=timedelta(seconds=60),
            ),
        ],
        final_action=Decision.REJECT,
        final_action_reason="所有审查者均超时，自动拒绝高风险操作",
    )

    # 2. 创建 HITL 管理器
    hitl = HITLManager(escalation_policy=escalation_policy)

    # 3. 模拟一个同步审批请求
    context = ActionContext(
        action_type="DELETE_BUCKET",
        target="prod-data-archive",
        risk_level=RiskLevel.CRITICAL,
        description="删除 S3 Bucket 'prod-data-archive'",
        impact={"objects": 1_247_892, "size_gb": 3200, "last_access": "90天前"},
        trigger_reason="磁盘清理策略自动触发",
        suggested_action="建议先完整备份后再删除",
        related_history=[
            {"action": "DELETE_BUCKET", "count": 23, "rollbacks": 0},
        ],
    )

    print("=" * 60)
    print("HITL 审批请求示例")
    print("=" * 60)
    print(context.summarize())
    print("-" * 60)

    # 由于是示例，我们 mock 人类审查者的决策
    # 在实际系统中，这里会连接到 UI/消息通知

    request = await hitl.request_approval(
        context=context,
        mode=InterruptionMode.ASYNCHRONOUS,  # 使用异步模式避免阻塞示例
        timeout=timedelta(minutes=5),
    )

    print(f"\n审批结果: {request.decision.value if request.decision else 'pending'}")
    print(f"决策者: {request.decided_by}")
    print(f"状态: {request.status}")

    # 4. 查看疲劳报告
    print("\n审查者疲劳报告:")
    for entry in hitl.get_fatigue_report():
        print(f"  {entry['reviewer']}: 审查 {entry['review_count']} 次, "
              f"疲劳度 {entry['fatigue_score']:.2f}, "
              f"{'疲劳' if entry['is_fatigued'] else '正常'}")

    # 5. 查看决策历史
    print(f"\n决策历史: {len(hitl.get_decision_history())} 条记录")


if __name__ == "__main__":
    asyncio.run(example_usage())
```

**输出示例**:

```
============================================================
HITL 审批请求示例
============================================================
⚠ 操作: DELETE_BUCKET
  目标: prod-data-archive
  风险: CRITICAL
  描述: 删除 S3 Bucket 'prod-data-archive'
  影响: {"objects": 1247892, "size_gb": 3200, "last_access": "90天前"}
  触发: 磁盘清理策略自动触发
------------------------------------------------------------
[AutoReview] 规则 '只读操作自动批准' 不命中
[AutoReview] 规则 '零影响操作自动批准' 不命中
[HITL] 同步审批请求: a1b2c3d4e5f6
[HITL] 升级层级 0: 分配给 张三 (值班)
...等待人类审查...
[Human] 张三 审查请求 a1b2c3d4e5f6

审批结果: approve
决策者: 张三 (值班)
```

---

## Capability Boundaries（能力边界）

理解 HITL 的局限性同样重要。以下是 HITL 的已知边界和风险：

### 1. 人类的速度瓶颈

| 操作 | 人类响应时间 | Agent 等效操作数 |
|------|-------------|-----------------|
| 简单确认 | 2-5 秒 | 10-50 次 API 调用 |
| 阅读上下文后决策 | 10-30 秒 | 100-300 次 API 调用 |
| 复杂决策（需分析） | 1-5 分钟 | 600-3000 次 API 调用 |
| 多人审批流程 | 数小时到数天 | 不可计数 |

**结论**: HITL 不适用于需要亚秒级响应的场景。

### 2. 人类决策的质量问题

- **疲劳影响**: 连续审查 30 分钟后，错误率上升约 30-50%
- **锚定效应**: 被 Agent 的请求措辞影响，倾向于批准
- **确认偏误**: 倾向于寻找支持 Agent 决策的证据
- **社会惰化**: 多人审批时，每个人都觉得别人会仔细审查
- **情绪波动**: 个人情绪影响决策一致性

### 3. 人类的成本

- 专职审查者的人力成本（工资、培训、轮班）
- 审查延迟造成的机会成本
- 为了配合 HITL 而设计的系统复杂度
- 审查工具的维护和开发成本

### 4. 不可适用场景

- 实时交易系统（毫秒级决策）
- 大规模批量操作（逐条审批不可行）
- 人类无法理解的技术决策（如优化算法的参数选择）
- 人类不具备相应专业知识的领域

### 5. HITL 不是万能药

- HITL 不能解决 Agent 的**系统性偏差**（人类可能共享同样的偏差）
- HITL 不能弥补**差的默认行为**（如果 Agent 的系统默认行为就是有害的）
- HITL 不能替代**安全设计**（安全应该设计进系统，而不是靠人工审查兜底）
- HITL 不能解决**责任归属问题**（当人机共同决策时，责任如何划分）

---

## Comparison: 自主性光谱

从完全自主到完全人工控制是一个连续光谱：

```
完全自主 <─────── 混合模式 ───────> 完全人工
   │         │          │           │
   0         1          2           3
```

| 模式 | 描述 | 适用场景 | 示例 |
|------|------|---------|------|
| **Level 0: 完全自主** | Agent 自主决策和执行，无人工介入 | 低风险、高度可预测的环境 | 文件整理 Agent、定时备份 |
| **Level 1: 审计追踪** | Agent 自主执行，操作日志供事后审计 | 中等风险，可回滚的操作 | 数据迁移、批量数据处理 |
| **Level 2: 异常上报** | Agent 自主执行，仅在异常/高风险时请求人工介入 | 有明确正常/异常边界的操作 | CI/CD 部署、配置变更 |
| **Level 3: 确认门** | Agent 建议，人类确认后执行 | 高风险、不可逆操作 | 删除数据、修改权限 |
| **Level 4: 建议模式** | Agent 生成计划和建议，人类决定是否执行 | 需要人类专业判断的场景 | 金融交易、医疗诊断建议 |
| **Level 5: 完全人工** | 人类手动执行所有操作，Agent 仅提供信息 | 极高风险、法律合规要求 | 核设施控制、军事行动 |

### 选择指南

```
问题 1: 操作是否可逆？
→ 不可逆 → Level 3+（确认门或更高）
→ 可逆 → Level 1-2（审计或异常上报）

问题 2: 操作风险等级？
→ 低 → Level 0-1（自主或审计）
→ 中 → Level 2（异常上报）
→ 高 → Level 3-4（确认门或建议模式）
→ 极高 → Level 4-5（建议或完全人工）

问题 3: 是否有成熟的规则可以自动化决策？
→ 有 → 可以降低一个 Level
→ 没有 → 保持或提高一个 Level

问题 4: 人类审查者是否具备足够的专业知识和上下文？
→ 是 → Level 3-4 有效
→ 否 → 需要更好的上下文呈现，或保持 Level 0
```

### 适应性调整

理想情况下，HITL 系统应该根据以下因素**动态调整**自主性等级：

```
追踪记录: Agent 过去 100 次操作 0 次回滚 → 自主性 +1
时间因素: 凌晨 3 点 → 更倾向于异步审查
用户偏好: 用户历史批准率 > 95% → 可减化确认流程
操作频次: 第 50 次同类操作 → 可考虑自动批准
审查者状态: 审查者疲劳 → 自动降级为异步
```

---

## Engineering Optimization（工程优化）

以下是在工程实践中优化 HITL 系统的关键策略。

### 1. 智能 HITL 触发（Smart Triggering）

避免在每个操作上都询问人类，而是在真正需要时触发。

**策略 1: 基于置信度的触发**
```python
def should_trigger_hitl(
    action: Action,
    confidence: float,
    risk_score: float,
) -> bool:
    # 低置信度 + 高风险 = 触发 HITL
    if confidence < 0.7 and risk_score > 0.3:
        return True
    # 极低置信度 = 任何风险都触发
    if confidence < 0.3:
        return True
    # 极高置信度 + 低风险 = 不触发
    if confidence > 0.95 and risk_score < 0.5:
        return False
    # 默认按风险阈值
    return risk_score > 0.7
```

**策略 2: 异常检测触发**
```
基线: Agent 通常的决策模式
偏离基线 > 2σ → 触发 HITL
检测到对抗性输入 → 触发 HITL
操作目标不在已知白名单 → 触发 HITL
```

**策略 3: 渐进式学习触发**
```
首次操作 → 强制 HITL
第 2-10 次 → 异常检测触发
10 次以后无问题 → 仅高风险触发
出现 1 次回滚 → 重置为首次状态
```

### 2. 减少误报（Reducing False Alarms）

误报是审查疲劳的主要来源。降低误报率的策略：

**误报分析**:
```python
class FalseAlarmAnalyzer:
    """误报分析器 — 追踪哪些 HITL 触发是不必要的"""

    def __init__(self):
        self.trigger_history: list[dict] = []

    def record_trigger(
        self, trigger_type: str, was_approved: bool,
        would_have_caused_issue: bool, reviewer: str
    ):
        self.trigger_history.append({
            "trigger_type": trigger_type,
            "was_approved": was_approved,
            "would_have_caused_issue": would_have_caused_issue,
            "reviewer": reviewer,
            "timestamp": time.time(),
        })

    def get_false_alarm_rate(self, trigger_type: str | None = None) -> float:
        """计算误报率"""
        relevant = self.trigger_history
        if trigger_type:
            relevant = [t for t in relevant if t["trigger_type"] == trigger_type]

        if not relevant:
            return 0.0

        # 误报: 人类批准了且不会造成问题（说明不需要 HITL）
        false_alarms = [
            t for t in relevant
            if t["was_approved"] and not t["would_have_caused_issue"]
        ]
        return len(false_alarms) / len(relevant)
```

**优化措施**:
1. 高频误报的触发条件自动放宽
2. 在不同上下文中使用不同的触发阈值
3. 季节性/周期性操作可跳过 HITL
4. 基于用户角色调整触发灵敏度

### 3. 从人类决策中学习（Learning from Human Decisions）

HITL 系统应该从每个决策中学习，逐步减少对人类介入的依赖。

**决策模式学习**:
```python
class DecisionLearner:
    """
    从人类审批历史中学习决策模式
    目标是逐渐用自动规则替代人工审查
    """

    def __init__(self):
        self.patterns: dict[str, list[dict]] = {}

    def record_decision(
        self, action_type: str, context: dict, decision: Decision
    ):
        key = self._build_pattern_key(action_type, context)
        if key not in self.patterns:
            self.patterns[key] = []
        self.patterns[key].append({
            "decision": decision.value,
            "context": context,
            "timestamp": time.time(),
        })

    def _build_pattern_key(self, action_type: str, context: dict) -> str:
        # 构建模式键: 只使用关键特征
        significant_features = {
            "action_type": action_type,
            "target_type": context.get("target_type"),
            "risk_level": context.get("risk_level"),
            "amount_if_financial": context.get("amount", 0) > 1000,
        }
        return json.dumps(significant_features, sort_keys=True)

    def predict_decision(self, action_type: str, context: dict) -> Decision | None:
        """基于历史预测人类的决策"""
        key = self._build_pattern_key(action_type, context)
        history = self.patterns.get(key, [])

        if len(history) < 5:
            return None  # 数据不足，无法预测

        approvals = sum(1 for h in history if h["decision"] == "approve")
        approval_rate = approvals / len(history)

        if approval_rate > 0.95:
            return Decision.APPROVE
        elif approval_rate < 0.1:
            return Decision.REJECT
        return None  # 不确定，继续使用 HITL

    def should_skip_hitl(self, action_type: str, context: dict) -> bool:
        """判断是否可以跳过 HITL"""
        prediction = self.predict_decision(action_type, context)
        if prediction is None:
            return False

        confidence = self._get_confidence(action_type, context)
        return confidence > 0.9

    def _get_confidence(self, action_type: str, context: dict) -> float:
        key = self._build_pattern_key(action_type, context)
        history = self.patterns.get(key, [])
        if len(history) < 5:
            return 0.0
        # 简单置信度模型: 样本量越大越自信
        return min(0.99, len(history) / 100)
```

**学习策略**:

| 阶段 | 行为 | 条件 |
|------|------|------|
| 冷启动 | 对所有操作触发 HITL | 无历史数据 |
| 学习期 | 触发 HITL + 记录决策 | 同类型操作 < 20 次 |
| 半自动 | 对高确定性操作跳过 HITL | 同类型 > 20 次且一致率 > 90% |
| 自动期 | 仅异常场景触发 HITL | 同类型 > 100 次且一致率 > 95% |
| 回退 | 重新强制 HITL | 出现误判或用户需求变化 |

### 4. 延迟优化（Latency Optimization）

**预取审批（Pre-fetch Approval）**:
```python
class PrefetchApproval:
    """预取审批 — 预测可能需要审批的操作，提前发起"""

    def __init__(self, hitl: HITLManager):
        self.hitl = hitl
        self.prefetch_cache: dict[str, Decision] = {}

    async def predict_and_prefetch(self, agent_plan: list[ActionContext]):
        """分析 Agent 计划，对可能需要的操作提前发起审批"""
        for ctx in agent_plan:
            if ctx.risk_level in (RiskLevel.HIGH, RiskLevel.CRITICAL):
                # 提前发起异步审批
                req = await self.hitl.request_approval(
                    context=ctx,
                    mode=InterruptionMode.ASYNCHRONOUS,
                )
                self.prefetch_cache[ctx.action_id] = req

    async def get_or_wait(self, ctx: ActionContext) -> Decision:
        """获取预取结果，或等待审批"""
        if ctx.action_id in self.prefetch_cache:
            # 结果应该已经在缓存中
            return self.prefetch_cache[ctx.action_id].decision or Decision.DEFER
        # 没有预取，需要实时等待
        req = await self.hitl.request_approval(
            context=ctx,
            mode=InterruptionMode.SYNCHRONOUS,
        )
        return req.decision or Decision.REJECT
```

**上下文压缩**: 对于大型上下文，使用 AI 摘要而非完整呈现：
```
完整上下文: 347 行日志 + 12 个关联事件 + 5 个配置文件差异
压缩上下文: "Agent 尝试修改数据库索引 schema_v2 以优化慢查询。
            涉及 3 张表，预计影响 500 万行数据。
            风险：修改期间写入性能可能下降 20%。"
```

**分级呈现**: 先展示摘要，人类可点击展开详情：
```
[初始视图] 高风险: 修改数据库索引 schema_v2
[展开详情] 影响表: users, orders, payments
[展开全部] DDL 语句 + 执行计划 + 回滚脚本
```

### 5. 审计与分析（Audit & Analytics）

建立一个 HITL 系统的仪表盘来监控运行状况：

```
HITL 仪表盘
─────────────────────────────────────
总审批请求:     1,247     今日: 42
平均响应时间:   12.3s     P95: 45.2s
批准率:         87.3%     拒绝率: 8.1%
升级率:         3.2%      超时率: 1.4%
审查者疲劳:     张三 72% ⚠️  李四 45%  王五 23%
误报率:         12.5%     ↓ 较上周下降 3.2%
自动替代率:     34.2%     ↑ 较上周上升 5.1%
─────────────────────────────────────
```

### 6. 最佳实践总结

| 实践 | 描述 | 优先级 |
|------|------|--------|
| 最小充分上下文 | 只展示人类决策所需的最少信息 | P0 |
| 渐进式授权 | 从逐次审批到策略级授权 | P0 |
| 自动降级 | 审查者疲劳时自动降低介入频率 | P0 |
| 误报追踪 | 追踪不必要的 HITL 触发并优化 | P1 |
| 决策学习 | 从历史决策中学习，减少未来干预 | P1 |
| 升级链 | 配置至少 3 级升级路径 | P1 |
| 疲劳监测 | 实时监测审查者的疲劳状态 | P1 |
| 批量审查 | 支持同类操作批量审批 | P1 |
| 预取批准 | 提前预测并获取审批 | P2 |
| A/B 测试 | 对比不同 HITL 策略的效果 | P2 |

---

## 总结

Human-in-the-Loop 是 AI Agent 安全体系中最重要的一道防线——它不是在技术上阻止 Agent 做错事，而是在 Agent 和现实世界之间设置一个由人类判断力构成的缓冲层。好的 HITL 系统不是让人类成为 Agent 的瓶颈，而是让人类的判断在最需要的地方发挥最大价值。

关键要点：
- **不是所有操作都需要 HITL** — 只对高风险、歧义、异常场景触发
- **HITL 是手段而非目的** — 目标是安全，不是让人类审批每个操作
- **好的 HITL 系统是自适应的** — 从人类决策中学习，逐步减少干预
- **审查者也需要保护** — 疲劳管理是 HITL 系统长期运行的关键
- **HITL 不是万能的** — 需要与权限控制、审计日志、沙箱等技术结合使用

---

*下一节: 12.4.8 权限控制的未来方向*
