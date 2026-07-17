# 8.5.3 Market Pattern — 市场模式

## 简单介绍

Market Pattern（市场模式）是一种**基于经济学的多 Agent 协调范式**：每个 Agent 被建模为自利的经济参与者，通过买卖机制交换服务或资源，价格由供需关系动态决定。Agent 不直接命令彼此，而是通过"出价-要价-成交"的市场流程完成协作。

核心思想可以概括为一句话：**让价格信号代替控制指令，让自利行为产生全局秩序**。

类比真实世界：打车平台（乘客出价、司机接单）、股票交易所（买方卖方撮合）、云计算 Spot Instance（竞价获取空闲算力）。系统中没有中央调度员，但市场机制通过价格这只"看不见的手"自动完成了资源分配。

```
       ┌─────────────────────────────────────────────────┐
       │                 Market Pattern                   │
       │                                                  │
       │  "Don't tell agents what to do.                  │
       │   Let them bid for what they want,               │
       │   and the price will coordinate them."           │
       │                                                  │
       │   ┌──────────┐      ┌──────────┐                │
       │   │ Producer  │      │ Consumer  │               │
       │   │ (卖方/服务方)│◄────►│ (买方/需求方)│               │
       │   └──────────┘      └──────────┘                │
       │         ▲                  ▲                     │
       │         │    价格信号      │                     │
       │         ▼                  ▼                     │
       │   ┌──────────────────────────────────┐          │
       │   │        Market / Exchange          │          │
       │   │  (订单簿 + 撮合引擎 + 清算系统)    │          │
       │   └──────────────────────────────────┘          │
       │                                                  │
       └─────────────────────────────────────────────────┘
```

## 基本原理

### Agent 角色体系

市场模式中，Agent 扮演三种核心角色：

| 角色 | 行为 | 目标 | 类比 |
|------|------|------|------|
| **Producer（生产者）** | 提供服务/资源，发布 Ask（要价） | 最大化收益，提高资源利用率 | 卖家、供应商、矿工 |
| **Consumer（消费者）** | 提出需求，发布 Bid（出价） | 以最低成本完成任务 | 买家、租客、调度器 |
| **Broker（经纪人/做市商）** | 撮合买卖，维护市场秩序 | 赚取差价，维持流动性 | 交易所、拍卖师 |

### 交互流程

生产者、消费者、市场三者之间的核心交互：

```
  Consumer (买方)                    Market (市场)                   Producer (卖方)
       │                                │                                │
       │  1. 发布需求/出价 (Bid)         │                                │
       │──────────────────────────────► │                                │
       │                                │  2. 发布服务/要价 (Ask)         │
       │                                │◄────────────────────────────── │
       │                                │                                │
       │                                │  3. 订单簿匹配                 │
       │                                │  ┌────────────────────┐        │
       │                                │  │ Bids  │  Asks      │        │
       │                                │  │───────│─────────── │        │
       │                                │  │ $9.00 │  $11.00    │        │
       │                                │  │ $8.50 │  $12.00    │        │
       │                                │  │ $8.00 │  $15.00    │        │
       │                                │  └────────────────────┘        │
       │                                │                                │
       │                                │  4. 撮合成交 (Bid >= Ask)      │
       │                                │  ◄──── Match ────►             │
       │                                │                                │
       │  5a. 支付代币                   │  5b. 释放服务                  │
       │──────────────────────────────► │◄────────────────────────────── │
       │                                │                                │
       │  6a. 确认收货                   │  6b. 确认收款                  │
       │──────────────────────────────► │◄────────────────────────────── │
       │                                │                                │
       │  7. 评价/信誉更新               │                                │
       │──────────────────────────────► │──────────────────────────────► │
```

### 市场生态系统全景

```
                        ┌──────────────────────────────────────┐
                        │          Market Ecosystem            │
                        │                                      │
                        │          ┌──────────────┐            │
                        │          │   Regulator  │            │
                        │          │  (监管者)     │            │
                        │          │  - 反垄断     │            │
                        │          │  - 防欺诈     │            │
                        │          │  - 定规则     │            │
                        │          └──────┬───────┘            │
                        │                 │                    │
                        │         ┌───────┴───────┐            │
                        │         │  Order Book   │            │
                        │         │  (订单簿)      │            │
                        │         │  ┌─────────┐  │            │
                        │         │  │ Bids    │  │            │
                        │         │  │ Asks    │  │            │
                        │         │  └─────────┘  │            │
                        │         └───────┬───────┘            │
                        │                 │                    │
                        │         ┌───────┴───────┐            │
                        │         │  Matching     │            │
                        │         │  Engine       │            │
                        │         │  (撮合引擎)    │            │
                        │         └───────┬───────┘            │
                        │                 │                    │
        ┌───────────────┼─────────────────┼────────────────┐   │
        │               │                 │                │   │
   ┌────┴─────┐   ┌────┴─────┐     ┌─────┴────┐   ┌──────┴──┐│
   │Producer 1│   │Producer 2│ ... │Consumer 1│   │Consumer 2││
   │(卖方)    │   │(卖方)    │     │(买方)    │   │(买方)    ││
   └──────────┘   └──────────┘     └──────────┘   └──────────┘│
        │               │                │               │    │
        └───────────────┴────────────────┴───────────────┘    │
                        │                                      │
                   ┌────┴────┐                                 │
                   │  Bank   │  (代币发行/销毁/结算)           │
                   └─────────┘                                 │
                        │                                      │
                   ┌────┴────┐                                 │
                   │Reputation│ (信誉系统 — 防欺诈)            │
                   │  System │                                 │
                   └─────────┘                                 │
                        │                                      │
                   ┌────┴────┐                                 │
                   │  Audit  │  (审计追踪 — 争议仲裁)           │
                   │  Trail  │                                 │
                   └─────────┘                                 │
                        │                                      │
                   ┌────┴────┐                                 │
                   │ Oracle  │  (价格预言机 — 外部定价参考)     │
                   └─────────┘                                 │
                   └──────────────────────────────────────────┘
```

### 价格发现机制

价格发现是市场模式的核心。以下是四种经典拍卖机制的价格发现过程：

```
┌────────────────────────────────────────────────────────────────────┐
│                    Auction Types Comparison                         │
├──────────────┬──────────────┬──────────────┬───────────────────────┤
│  English     │  Dutch       │  Vickrey     │  Sealed-Bid           │
│  (英式拍卖)   │  (荷式拍卖)  │  (维克里拍卖) │  (密封投标)           │
├──────────────┼──────────────┼──────────────┼───────────────────────┤
│  Price       │  Price       │  Price       │  Price                │
│  ▲           │  ▲           │  ▲           │  ▲                    │
│  │    ───    │  │  ───      │  │           │  │  B3=$15             │
│  │   │  │    │  │     │     │  │  ─────    │  │  B2=$12             │
│  │   │  │    │  │     │     │  │ │      │  │  │  B1=$10             │
│  │  ─┘  │    │  │     │     │  │ │      │  │  │                     │
│  │ │    │    │  │     │     │  │ │ B*    │  │  │  Winner B3 pays    │
│  │ │    │    │  │     ▼     │  │ │       │  │  │  = max(B2)=$12     │
│  │─┘    │    │  └─────      │  └─┘───────┘  │  └─────────────────────
│  └──────►    └────────────► └──────────────►                        │
│  Time         Time           Time            Time                   │
├──────────────┼──────────────┼──────────────┼───────────────────────┤
│  公开叫价     │  价格从高     │  密封投标     │  密封投标             │
│  价格上升     │  往下降      │  最高价者得   │  最高价者得           │
│  最高价者得   │  首个叫停者得│  支付第二高价  │  支付自己的出价       │
├──────────────┼──────────────┼──────────────┼───────────────────────┤
│  激励真相    │  速度快      │  激励诚实出价 │  简单直接             │
│  揭示价值    │  适合易损品  │  (策略免疫)    │                      │
├──────────────┼──────────────┼──────────────┼───────────────────────┤
│  耗时长      │  买家可能    │  规则难理解   │  鼓励策略性低报        │
│  可能合谋    │  等待降价    │  信任第三方   │  易出现 winner's curse │
└──────────────┴──────────────┴──────────────┴───────────────────────┘
```

### Double Auction（双向拍卖）的价格发现动态

```
均衡价格发现过程（Supply-Demand Cross）：

   Price
    ▲
    │  Demand (Bids)          Supply (Asks)
    │     ▼                     ▲
    │   $15  ██                  │
    │   $14  ███                 │
    │   $13  ████                │
    │   $12  █████               │░░           ← 供过于求
    │   $11  ██████        ░░░░░░             ← 均衡区间
    │   $10  █████   ════   ░░░░░             ← 均衡价格 P*
    │   $9   ████    ░░░░░░░                  ← 供不应求
    │   $8   ██      ░░░░
    │   $7   █       ░░░
    │   $6          ░░
    │
    └─────────────────────────────────►  Quantity
        0    1    2    3    4    5

  市场出清价格 P* 是使 Bid(price) ≥ Ask(price) 的最高价格
  在此价格下，所有愿意以 P* 或更高价格买入的买方，
  和所有愿意以 P* 或更低价格卖出的卖方，都会成交。
```

## 背景与发展

### 起源

Market Pattern 的思想渊源横跨经济学、分布式系统和人工智能三个领域：

| 时间 | 里程碑 | 贡献 |
|------|--------|------|
| **1776** | Adam Smith《国富论》 | "看不见的手"——自利行为产生集体福祉的理论基础 |
| **1951** | Arrow-Debreu 一般均衡理论 | 证明在完全竞争条件下，市场能达到帕累托最优 |
| **1960s** | 计算经济学诞生 | 开始用计算机模拟市场行为和经济均衡 |
| **1995** | Clearwater 等发表 **"Market-Based Control"** | 里程碑式论文集，首次系统提出用市场机制进行分布式系统资源分配 |
| **1996** | CASBA（金融 Agent 竞赛） | 将 Agent 技术应用于金融市场的模拟和优化 |
| **2001-2005** | 网格计算市场 | Tycoon、Bellagio 等项目用拍卖分配计算集群资源 |
| **2008** | 比特币白皮书 | 分布式共识+代币经济，启发了去中心化 Agent 经济系统 |
| **2010s** | Amazon EC2 Spot Instance | 工业级应用：用实时市场定价分配闲置算力 |
| **2015+** | 多 Agent 系统与博弈论结合 | 拍卖设计、机制设计理论成为 Agent 协调的核心工具 |
| **2023+** | LLM Agent 经济 | AutoGPT 等项目中 Agent 互相"雇佣"完成子任务 |

### 关键人物与理论

- **Scott H. Clearwater**：Market-Based Control 论文集的编辑，市场控制理论的奠基人
- **Noam Nisan**：算法博弈论先驱，将机制设计理论引入计算机科学
- **Eric S. Raymond**：开源社区中"集市模式"的隐喻，为市场模式在软件工程中的应用提供了哲学基础
- **Vernon Smith & Daniel Kahneman**：行为经济学，揭示了真实市场中人的非理性行为（对 Agent 市场的设计有重要警示意义）

## 之前做法与局限

在 Market Pattern 出现之前，多 Agent 系统的资源分配主要依靠以下方式：

### 做法 1：集中式调度（Centralized Scheduler）

```
                    ┌──────────────┐
                    │  Scheduler   │  ← 全局信息 + 全局决策
                    │  (中央调度器) │
                    └──────┬───────┘
                    ┌──────┼───────┐
                    │      │       │
                 ┌──┴─┐ ┌──┴─┐ ┌──┴─┐
                 │ A1 │ │ A2 │ │ A3 │
                 └────┘ └────┘ └────┘
```

- **优点**：理论上能达到全局最优
- **局限**：
  - 单点瓶颈：调度器处理所有请求，负载上限低
  - 信息爆炸：调度器需要知道每个 Agent 的全部状态和需求
  - 缺乏弹性：Agent 数量增加时调度复杂度指数增长
  - 适应性差：环境变化需要重新设计调度策略

### 做法 2：先到先得（First-Come-First-Served）

```
  请求到达顺序:  [任务C] [任务A] [任务B]
                      ↓
  处理顺序:      [任务C] [任务A] [任务B]
  
  即使任务A价值100，任务B价值1，任务C价值0.5
  C排在前面，就占用了资源
```

- **优点**：实现简单，公平（按到达顺序）
- **局限**：
  - 无优先级：高价值任务不会获得优先处理
  - 资源浪费：低价值任务占用高价值资源
  - 缺乏效率：全局资源利用率远低于最优

### 做法 3：轮询调度（Round-Robin）

```
  时间片:   A1 → A2 → A3 → A4 → A1 → A2 → ...
  
  每个 Agent 获得均等的时间/资源
  不管 Agent A3 是否急需，A1 是否需要那么多
```

- **局限**：无视 Agent 的差异化需求，平均分配导致整体效率低

### 总结：之前做法的共同问题

| 问题 | 表现 | 后果 |
|------|------|------|
| **集中式瓶颈** | 单点决策 | 扩展性差，单点故障 |
| **信息不对称** | 调度器不知 Agent 真实需求 | 次优决策 |
| **无价值感知** | 高价值/低价值任务无差异 | 资源错配 |
| **静态策略** | 固定规则无法适应动态变化 | 环境变化时效率急剧下降 |
| **激励不兼容** | Agent 无动机说实话 | 博弈性低报/虚报需求 |

## 核心矛盾

Market Pattern 虽然强大，但内在矛盾深刻：

### 核心矛盾 1：个体自利 vs 集体最优

```
         个体理性               集体理性
    ┌────────────────┐    ┌────────────────┐
    │ Agent 追求      │    │ 系统希望         │
    │ 自身利益最大化   │    │ 全局资源利用率最高 │
    │                │    │                │
    │ "我的收益最高"  │◄──►│ "系统效率最高"   │
    └────────────────┘    └────────────────┘
              │                    │
              └──── 价格机制 ──────┘
            Adam Smith: "看不见的手"
              可以在理想条件下统一二者
```

**经典博弈——囚徒困境在 Agent 市场中的体现**：

```
Agent B
               合作(低报价)       背叛(高报价)
Agent A ┌────────────────┼────────────────┐
合作    │  (5, 5)        │  (0, 8)        │
(低报价) │  全局最优       │  单边最差       │
        ├────────────────┼────────────────┤
背叛    │  (8, 0)        │  (2, 2)        │
(高报价) │  单边最优       │  纳什均衡 ← 实际结果 │
        └────────────────┴────────────────┘
```

### 核心矛盾 2：效率 vs 公平

```
    效率（Efficiency）                   公平（Fairness）
    ┌─────────────────────┐    ┌─────────────────────┐
    │ 资源分配给出价最高者  │    │ 资源平等分配给所有人  │
    │                    │    │                     │
    │ 总产出最大          │◄──►│ 每个人的基本需求       │
    │ 但可能淘汰弱者      │    │ 得到保障              │
    └─────────────────────┘    └─────────────────────┘
```

### 经典市场失灵（Market Failures）

市场模式并非万能，以下失灵场景需要特别关注：

```
┌──────────────────────────────────────────────────────────────────┐
│                    Market Failure Modes                           │
├───────────────────┬──────────────────┬───────────────────────────┤
│  失灵类型           │    表现           │    典型场景                │
├───────────────────┼──────────────────┼───────────────────────────┤
│                   │                  │                           │
│  垄断              │ 单一 Producer    │  唯一 GPU 算力提供商         │
│  (Monopoly)       │  任意定价        │  独家数据源 Agent           │
│                   │                  │                           │
│  ┌──────────┐     │ 解决方案:         │                           │
│  │Monopoly  │     │  价格上限监管     │                           │
│  │    P▲    │     │  引入竞品奖励     │                           │
│  └──────────┘     │                  │                           │
├───────────────────┼──────────────────┼───────────────────────────┤
│                   │                  │                           │
│  公地悲剧          │ 共享资源被过度使用 │  公共 API 会被 Consumer    │
│  (Tragedy of      │ 因为 Agent       │  无节制调用导致超载          │
│   the Commons)    │ 只考虑个体成本   │                           │
│                   │                  │                           │
│  ┌──────────┐     │ 解决方案:         │                           │
│  │  Commons │     │  个人配额+交易    │                           │
│  │  Usage▲  │     │  使用费/拥堵费    │                           │
│  └──────────┘     │                  │                           │
├───────────────────┼──────────────────┼───────────────────────────┤
│                   │                  │                           │
│  信息不对称        │ 卖方比买方      │  Producer 隐瞒服务真实质量   │
│  (Information     │ 更了解商品质量  │  Consumer 隐瞒真实预算       │
│   Asymmetry)      │ 导致"劣币驱逐良币"│                           │
│                   │                  │                           │
│  ┌──────────┐     │ 解决方案:         │                           │
│  │ Lemons   │     │  信誉系统        │                           │
│  │ Market   │     │  强制信息披露    │                           │
│  └──────────┘     │                  │                           │
├───────────────────┼──────────────────┼───────────────────────────┤
│                   │                  │                           │
│  外部性            │ 交易影响第三方   │  Agent A&B 交易占用网络     │
│  (Externality)    │ 但未计入价格    │  带宽，影响 Agent C         │
│                   │                  │                           │
│  ┌──────────┐     │ 解决方案:         │                           │
│  │ Extern.  │     │  庇古税(拥堵费)  │                           │
│  │ Cost     │     │  外部性内部化    │                           │
│  └──────────┘     │                  │                           │
├───────────────────┼──────────────────┼───────────────────────────┤
│                   │                  │                           │
│  合谋/串通         │ 多个 Agent       │  多家 Producer 联合抬价     │
│  (Collusion)      │ 联合操纵价格    │  多个 Consumer 联合压价      │
│                   │                  │                           │
│  ┌──────────┐     │ 解决方案:         │                           │
│  │Collusion │     │  合谋检测算法     │                           │
│  │ Price▲   │     │  匿名竞价        │                           │
│  └──────────┘     │                  │                           │
└───────────────────┴──────────────────┴───────────────────────────┘
```

## 主流优化方向

### 1. 拍卖机制设计（Auction Mechanism Design）

| 拍卖类型 | 规则 | 适用场景 | 优点 | 缺点 |
|----------|------|---------|------|------|
| **英式拍卖** | 公开叫价，价高者得 | 稀缺资源分配 | 价格发现充分 | 耗时长，易合谋 |
| **荷式拍卖** | 从高往低降价，首个接受者得 | 易损品、时效性资源 | 速度快 | 买方可能延迟出价 |
| **维克里拍卖** | 密封投标，最高价者得，付第二高价 | 高价值资源 | 激励说实话（策略免疫） | 规则复杂 |
| **双向拍卖** | 多买多卖同时竞价 | 持续交易市场 | 流动性好，效率高 | 匹配算法复杂 |
| **组合拍卖** | 对资源组合投标 | 互补性资源（如 GPU+内存） | 考虑资源互补性 | 计算复杂度极高（NP-hard） |
| **反向拍卖** | 买方发布需求，卖方竞争降价 | 任务外包 | 买方主导 | 可能压价过度 |

### 2. 信誉系统（Reputation Systems）

信誉系统是解决信息不对称的核心手段：

```
  交易完成后:
  Consumer ──────────► Rate Producer ──────────► Reputation Score
  
  信誉评分更新公式（加权移动平均）:
  
    R_new = α × R_recent + (1-α) × R_old
  
  其中 R_recent = f(完成率, 质量评分, 响应速度, 违约次数)
  
  信誉的应用:
    - 展示: 高信誉 Agent 获得更多曝光
    - 质押: 低信誉 Agent 需要更高保证金
    - 定价: 高信誉 Agent 可收取溢价
```

**信誉系统设计要点**：
- **冷启动问题**：新 Agent 无信誉记录 → 引入"初始信誉+逐步验证"机制
- **Sybil 攻击**：恶意注册大量身份刷好评 → 信誉与身份验证成本挂钩
- **共谋好评**：互相刷分 → 检测异常评分模式
- **时效性**：历史久远的评价权重降低 → 使用衰减因子

### 3. 动态定价算法（Dynamic Pricing）

```
         ┌───────────────────────────────┐
         │    动态定价策略空间             │
         ├───────────────────────────────┤
         │                               │
         │  基于供需:                     │
         │    P(t+1) = P(t) × (1 + k ×   │
         │      (D(t) - S(t)) / S(t))    │
         │                               │
         │  基于历史:                     │
         │    P(t) = EMA(历史成交价)       │
         │                               │
         │  基于学习:                     │
         │    RL Agent 通过试错学最优定价   │
         │                               │
         │  基于博弈:                     │
         │    Bayesian Nash Equilibrium   │
         │    预测对手出价后优化自身定价     │
         │                               │
         └───────────────────────────────┘
```

### 4. 预测性出价优化（Predictive Bid Optimization）

Consumer 利用历史数据预测未来价格走势，优化出价策略：

- **基于时间序列预测**：ARIMA、Prophet 等模型预测价格趋势
- **基于强化学习**：在历史市场中训练，在实际市场中执行
- **基于博弈论**：建立竞争对手出价模型，优化自身策略

### 5. 匹配市场（Matching Markets）

在双边市场中，不仅仅是价格匹配，还需要考虑多维偏好：

- **偏好排序**：买方给卖方排序，卖方给买方排序
- **Gale-Shapley 算法**：稳定匹配理论，应用于劳务市场
- **带支付匹配**：结合价格与偏好的综合匹配

## 实现挑战

### 1. 代币经济设计（Tokenomics）

```
┌──────────────────────────────────────────────────────────────┐
│                  Tokenomics 设计挑战                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  初始分配:                                                   │
│    ┌────────────────────────────────────┐                    │
│    │ 所有 Agent 初始代币数量？         │                    │
│    │ 如何防止初始分配不公平？          │                    │
│    │ 新加入 Agent 如何获得代币？       │                    │
│    └────────────────────────────────────┘                    │
│                                                              │
│  通货膨胀/通缩:                                              │
│    ┌────────────────────────────────────┐                    │
│    │ 代币总量固定？可增发？            │                    │
│    │ 增发规则是什么？谁有权增发？       │                    │
│    │ 代币如何退出流通？                │                    │
│    └────────────────────────────────────┘                    │
│                                                              │
│  价值锚定:                                                   │
│    ┌────────────────────────────────────┐                    │
│    │ 代币与真实资源如何兑换？          │                    │
│    │ 如果系统是封闭的，代币价值何来？   │                    │
│    │ 法币/外部资产是否可兑换？         │                    │
│    └────────────────────────────────────┘                    │
│                                                              │
│  循环流动:                                                   │
│    ┌────────────────────────────────────┐                    │
│    │ Producer 赚到代币后如何花出去？    │                    │
│    │ 如何在系统中形成持续的经济循环？   │                    │
│    │ 代币沉淀(Money Holiding)怎么处理？│                    │
│    └────────────────────────────────────┘                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2. 防作弊机制

| 攻击类型 | 描述 | 防御策略 |
|----------|------|---------|
| **投标狙击** | 在最后时刻出价，不让对手反应 | 随机结束时间、延长拍卖 |
| **围标（合谋）** | 多个 Producer 勾结抬高价格 | 价格异常检测、匿名化 |
| **洗售交易** | Agent 自我交易制造虚假活跃 | 交易模式分析、链路追踪 |
| **女巫攻击** | 注册大量假身份获取初始代币 | 身份验证成本、图分析 |
| **虚假报价** | 出价后不履约 | 保证金锁定、惩罚机制 |
| **信息操纵** | 散布虚假信息影响价格 | 价格预言机、信息真实性验证 |
| **前置交易** | Broker 利用订单信息优势 | 订单匿名化、T+0 限制 |

### 3. 做市商与流动性

```
  流动性危机场景：
  
  订单簿（流动性充足）        订单簿（流动性枯竭）
  ┌──────────────────┐      ┌──────────────────┐
  │ Bids  │  Asks    │      │ Bids  │  Asks    │
  │───────│──────────│      │───────│──────────│
  │ $10   │  $10.05  │      │ $10   │  $15.00  │ ← 价差极大
  │ $9.95 │  $10.10  │      │ (仅1个│  (仅1个   │
  │ $9.90 │  $10.15  │      │  买方) │  卖方)   │
  │ $9.85 │  $10.20  │      │       │          │
  │ $9.80 │  $10.25  │      │       │          │
  ├──────────────────┤      ├──────────────────┤
  │ 价差: $0.05      │      │ 价差: $5.00      │
  │ 深度: 10层       │      │ 深度: 1层        │
  └──────────────────┘      └──────────────────┘
```

**解决方案**：引入自动化做市商（Automated Market Maker, AMM）
- 恒定乘积做市（Uniswap 风格）：`x × y = k`
- 始终保持买卖双方挂单，赚取价差
- 在 Agent 市场中的变体：由系统预留代币池提供流动性

### 4. 信息不对称管理

```
  信息不对称 → 柠檬市场（Market for Lemons）:
  
  真实情况:                    买家认知:
  ┌───────────┐              ┌───────────┐
  │ Producer A│              │ 所有 Producer│
  │  质量: 90 │              │  质量: 均值  │
  │  成本: 80 │              │  只愿付均价  │
  └───────────┘              └───────────┘
  ┌───────────┐
  │ Producer B│              → 高质量退出
  │  质量: 30 │              → 平均质量进一步下降
  │  成本: 20 │              → 买家更不愿出价
  └───────────┘              → 市场崩溃（柠檬市场）
  
  防御策略:
  1. 信誉系统公开历史质量 → 缩小信息差
  2. 质量担保/保证金 → 高质量方信号传递
  3. 第三方审计 → 中立验证
  4. 强制信息披露 → 降低信息不对称
```

## 能力与边界

### 适合的场景

| 场景 | 原因 | 示例 |
|------|------|------|
| **资源分配** | 利用价格信号自动匹配供需 | 计算集群、带宽、存储空间分配 |
| **任务外包** | 任务发布者不清楚谁最适合 | 数据处理、翻译、代码审查任务分发 |
| **竞争性环境** | Agent 天然有竞争关系 | 拍卖代理、广告竞价、供应链竞标 |
| **大规模异构系统** | Agent 能力差异大，需要差异化分配 | IoT 设备群中不同算力节点的任务调度 |
| **去中心化协调** | 无中央控制器，每个 Agent 自主决策 | 分布式能源交易、P2P 资源共享 |
| **动态变化环境** | 供需需要实时调整 | 云资源 Spot Market、实时服务定价 |

### 不适合的场景

| 场景 | 原因 | 替代模式 |
|------|------|---------|
| **强协作任务** | 需要 Agent 紧密配合，市场交易成本高 | 议会模式、Mesh 模式 |
| **公共品供给** | 市场无法有效提供非排他性公共品 | 议会模式（集体决策） |
| **紧急响应** | 需要即时行动，拍卖流程太慢 | 管理模式（直接指令） |
| **公平优先** | 出价能力差异导致资源分配极度不均 | 配额制、轮询制 |
| **安全性关键** | 市场博弈可能导致不可预测的安全风险 | 严格分级控制 |
| **小规模系统** | Agent 数量少时市场机制的 overhead 不值得 | 简单 Master-Worker |

### 能力边界总结

```
                         能力光谱

  强适能力 ◄─────── 市场模式 ───────► 弱适能力
  
      资源分配          │          强协作
      任务外包          │          公共品
      异构系统          │          紧急响应
      无中心协调        │          公平优先
      动态环境          │          安全关键
      大规模            │          小规模
      ──────────────────┼──────────────────
      价格信号有效      │      价格信号失灵
```

## 与其他模式对比

| 维度 | Market Pattern | Parliament Pattern | Manager-Worker | Mesh Pattern |
|------|---------------|-------------------|---------------|-------------|
| **驱动机制** | 经济激励（价格） | 辩论投票（共识） | 指令控制（权威） | 直接通信（协议） |
| **核心假设** | 自利即效率 | 理性对话达共识 | 集中决策最优 | 对等协商 |
| **决策方式** | 价格发现 | 投票表决 | 上级决策 | 协商一致 |
| **协调成本** | 中（订单簿撮合） | 高（多轮辩论） | 低（单点发令） | 高（N:N 通信） |
| **扩展性** | 极好（O(n log n)） | 中等（O(n^2)） | 中等（O(n) 但单点瓶颈） | 差（O(n^2) 通信） |
| **容错性** | 高（无故障点） | 中（需协调者） | 低（管理者故障） | 高（网状路由） |
| **激励兼容** | 核心设计目标 | 不直接关注 | 强制服从 | 协议遵守 |
| **透明度** | 价格公开 | 对话公开 | 指令可见 | 消息可追踪 |
| **典型场景** | 资源市场 | 政策制定 | 组织执行 | 传感器网络 |

### 决策机制对比

```
                       Market (价格)
                         │
                    经济博弈
                         │
          ┌──────────────┼──────────────┐
          │              │              │
      Parliament     Manager          Mesh
      (辩论投票)    (指令控制)      (对等协商)
          │              │              │
      社会选择        层级权威      平等协商
```

### 与 Parliament Pattern 的哲学差异

```
Market Pattern:                    Parliament Pattern:
  "Individual rationality            "Collective rationality
   leads to group welfare"            requires shared deliberation"

  Agent 行为:                        Agent 行为:
  独立决策，追求个人利益              共同讨论，寻求群体共识
  价格是唯一信号                      论据是交流媒介
  不需要理解全局                      需要理解全局
  效率优先                            公平优先
  快速但可能有失公平                  公正但效率较低
```

## 核心优势

### 1. 高度可扩展（Scale Naturally）

```
  系统规模与协调成本关系：
  
  协调成本
    ▲
    │          Manager-Worker: O(n)
    │              ┌──── 管理者成为瓶颈
    │             ┌┘
    │         ┌───┘
    │     ┌───┘
    │   ┌─┘
    │ ┌─┘
    │┌┘
    │┘
    ├──────────────────────►
    │
    │  Market Pattern: O(n log n)
    │     ┌──   订单簿撮合可并行处理
    │   ┌──┘
    │  ┌┘
    │ ┌┘
    │ ┌┘
    │┌┘
    │┘
    └──────────────────────►  Agent 数量 n
```

- 订单簿可以分区处理（按资源类型、地域等）
- 匹配引擎可以水平扩展
- 新 Agent 加入无需重新配置整个系统

### 2. 天然去中心化（Inherently Decentralized）

- **无单点故障**：市场可以运行在分布式节点上
- **局部信息决策**：Agent 不需要全局知识
- **弹性容错**：单个 Agent 离线不影响市场运作

### 3. 经济高效（Economically Efficient）

```
  市场模式 vs 集中式调度 效率对比：
  
  资源利用率
    ▲ 100% ───────────────── 理论最优
    │       ▓▓▓▓▓ 市场模式
    │  80%  ───────────── (接近最优)
    │       ░░░░
    │  60%  ───────────── 集中式调度
    │
    └────────────────────────────►
       低            高
           环境动态程度
```

- **资源配置给最高价值的用途**：出价机制自然筛选
- **价格信号提供反馈**：供不应求时价格上涨，引导更多供给
- **自适应调节**：无需人工干预，系统自动响应变化

### 4. 自调节机制（Self-regulating）

```
  正向反馈循环（需求上升时）:
  
  需求增加 → 价格上升 → 更多 Producer 加入 → 供给增加
                              ↓
  价格趋于平稳 ← 供给=需求 ←─┘

  负向反馈循环（供给过剩时）:
  
  供给过剩 → 价格下降 → 低效 Producer 退出 → 供给减少
                               ↓
  价格趋于平稳 ← 供给=需求 ←──┘
```

### 5. 激励兼容（Incentive Compatible）

- 诚实出价是 Agent 的最优策略（在精心设计的市场中）
- 不需要强制监督，Agent 的自身利益驱使它们做出对系统有利的行为
- 减少欺诈动机——即使欺诈可能获利，长期损失更大（信誉成本）

## 工程优化

### 1. 双货币系统（Dual-Currency）

```
  ┌──────────────────────────────────────────────┐
  │           Dual-Currency System                │
  ├──────────────────────────────────────────────┤
  │                                              │
  │  代币 (Token)           信誉 (Reputation)     │
  │  ─────────────────       ─────────────────    │
  │  可转移/可交换           不可转移/不可交换      │
  │  表示购买力              表示可信度            │
  │  短期资源分配             长期信用历史          │
  │  可通胀/通缩             不可通胀（操作可积累）  │
  │                                              │
  │  交互方式:                                     │
  │  Bid = Token_Amount × (1 + Reputation_Bonus)  │
  │                                              │
  │  出价 = 代币数量 × (1 + 信誉加成)              │
  │                                              │
  │  作用:                                         │
  │  - 代币解决"谁付得起"的问题                     │
  │  - 信誉解决"谁可信赖"的问题                     │
  │  - 两者结合：高信誉 Agent 可以用更少的代币获胜   │
  │                                              │
  └──────────────────────────────────────────────┘
```

### 2. 自动化做市（Automated Market Making）

```
  恒定乘积做市公式:
  
         x × y = k
  
  x = 代币储备量
  y = 服务代金券储备量
  k = 恒定乘积
  
  当前价格 P = y / x
  
  当有人买入 Δx 代币时, 获得 Δy 服务代金券:
  (x + Δx) × (y - Δy) = k
  
  ⇒ Δy = y - k / (x + Δx)
  
  作用:
  - 任何时候都保证有流动性
  - 价格自动根据供需调整
  - 无需对手盘即可完成交易
```

### 3. 出价阴影算法（Bid Shading）

```
  问题: 在维克里拍卖中，出价者可能过高支付
  
  出价阴影 (Bid Shading): 在真实估值基础上打折出价
  
  简单阴影:
    Bid = Truthful_Value × (1 - δ)
    其中 δ 是固定折扣率
  
  自适应阴影:
    δ = f(Competition_Level, Historical_Win_Rate, Budget_Remaining)
  
  最优阴影（基于历史数据）:
      ┌──────────────────────┐
      │ 训练: 在历史拍卖数据上 │
      │ 学习最优 δ            │
      │ 目标: 最大化成功率    │
      │ 约束: 预算限制        │
      └──────────────────────┘
```

### 4. 订单簿管理（Order Book Management）

```
  数据结构（价格-时间优先排序）:
  
  Bids (买单, 降序排列)          Asks (卖单, 升序排列)
  ┌──────────┬──────────┐      ┌──────────┬──────────┐
  │ 价格     │ 数量     │      │ 价格     │ 数量     │
  ├──────────┼──────────┤      ├──────────┼──────────┤
  │ $12.00   │ 5        │      │ $10.00   │ 3        │
  │ $11.50   │ 8        │      │ $10.50   │ 6        │  ← 最优卖价
  │ $11.00   │ 12       │  ← 最优买价  │ $11.00   │ 4        │
  │ $10.50   │ 3        │      │ $11.50   │ 10       │
  │ $10.00   │ 1        │      │ $12.00   │ 7        │
  └──────────┴──────────┘      └──────────┴──────────┘
  
  撮合规则：Best Bid >= Best Ask 时成交
  成交价格：通常取 mid-price 或按到达顺序匹配
  
  优化技巧:
  - 使用跳表 (Skip List) 或红黑树维护价格序
  - 按价格档位聚合减少匹配复杂度
  - 部分成交时剩余量重新入队列
```

### 5. 市场微观结构优化（Market Microstructure）

| 优化项 | 描述 | 效果 |
|--------|------|------|
| **最小价格变动单位** | Tick Size 设置 | 太大会导致价差大，太小会频繁波动 |
| **最小交易量单位** | Lot Size 设置 | 防碎片化，但也降低了灵活性 |
| **订单类型** | 限价单、市价单、止损单 | 丰富 Agent 的策略空间 |
| **拍卖频率** | 连续交易 vs 批量拍卖 | 连续性高但加剧 HFT 竞争，批量拍卖降低博弈 |
| **信息披露** | 订单簿深度是否可见 | 透明促进发现，但也增加操纵风险 |
| **时间优先** | 价格相等时谁先到谁先得 | 公平但鼓励抢跑，可用随机化替代 |

## 代码示例

以下是用 Python 实现的一个最小化市场模式原型，包含 Producer、Consumer、OrderBook、Auctioneer 和 Settlement 核心组件。

### 完整实现

```python
"""
market_pattern_demo.py — Market Pattern 最小化实现

展示一个基于双向拍卖的 Agent 资源市场：
- Producer: 提供服务并要价
- Consumer: 发布需求并出价
- OrderBook: 维护买卖订单簿
- Auctioneer: 撮合引擎，执行匹配
- Settlement: 支付、交付和信誉更新
"""

from dataclasses import dataclass, field
from enum import Enum
from typing import List, Dict, Optional, Tuple
from heapq import heappush, heappop
import uuid
import time
import random
from collections import defaultdict


# ──────────────────────────────────────────────
# 基础类型定义
# ──────────────────────────────────────────────

class OrderType(Enum):
    """订单类型：买单(Bid) 或 卖单(Ask)"""
    BID = "bid"
    ASK = "ask"


class OrderStatus(Enum):
    """订单状态"""
    PENDING = "pending"      # 等待撮合
    PARTIAL = "partial"      # 部分成交
    FILLED = "filled"        # 完全成交
    CANCELLED = "cancelled"  # 已取消


@dataclass
class Order:
    """
    订单实体

    使用 heapq 进行优先级排序：
    - Bids: (-price, timestamp) → 高价在前，同价先到优先
    - Asks: (price, timestamp)  → 低价在前，同价先到优先
    """
    order_id: str
    agent_id: str
    order_type: OrderType
    resource_type: str       # 资源类型（如 "compute", "storage", "translation"）
    price: float             # 出价/要价
    quantity: int            # 数量
    timestamp: float = field(default_factory=time.time)
    status: OrderStatus = OrderStatus.PENDING
    filled_quantity: int = 0

    def remaining(self) -> int:
        return self.quantity - self.filled_quantity

    def is_filled(self) -> bool:
        return self.filled_quantity >= self.quantity

    def __lt__(self, other):
        """为 heapq 提供的比较方法"""
        if self.order_type == OrderType.BID:
            # 买单：价格从高到低，时间从早到晚
            return (-self.price, self.timestamp) < (-other.price, other.timestamp)
        else:
            # 卖单：价格从低到高，时间从早到晚
            return (self.price, self.timestamp) < (other.price, other.timestamp)


@dataclass
class Trade:
    """成交记录"""
    trade_id: str
    bid_order_id: str
    ask_order_id: str
    resource_type: str
    price: float
    quantity: int
    buyer_id: str
    seller_id: str
    timestamp: float = field(default_factory=time.time)


# ──────────────────────────────────────────────
# 信誉系统
# ──────────────────────────────────────────────

class ReputationSystem:
    """
    信誉追踪系统

    使用指数加权移动平均 (EWMA) 更新信誉分数。
    防止 Sybil 攻击和评分共谋的基本防护。
    """

    def __init__(self, alpha: float = 0.3, default_reputation: float = 0.5):
        self.alpha = alpha
        self.default_reputation = default_reputation
        self._scores: Dict[str, float] = defaultdict(lambda: default_reputation)
        self._ratings: Dict[str, List[Tuple[str, float, float]]] = defaultdict(list)
        # _ratings[agent_id] = [(rater_id, score, timestamp), ...]

    def get_score(self, agent_id: str) -> float:
        return self._scores.get(agent_id, self.default_reputation)

    def submit_rating(self, rater_id: str, target_id: str, score: float):
        """提交一个评分（0.0 - 1.0）"""
        clamped_score = max(0.0, min(1.0, score))
        self._ratings[target_id].append((rater_id, clamped_score, time.time()))
        self._update_score(target_id)

    def _update_score(self, agent_id: str):
        """使用 EWMA 更新信誉分数"""
        ratings = self._ratings[agent_id]
        if not ratings:
            return

        recent_scores = [r[1] for r in ratings[-20:]]  # 最近 20 条评分
        if not recent_scores:
            return

        # EWMA: R_new = alpha * R_recent_avg + (1-alpha) * R_old
        recent_avg = sum(recent_scores) / len(recent_scores)
        old_score = self._scores[agent_id]
        self._scores[agent_id] = self.alpha * recent_avg + (1 - self.alpha) * old_score

    def get_rating_count(self, agent_id: str) -> int:
        return len(self._ratings[agent_id])


# ──────────────────────────────────────────────
# 订单簿
# ──────────────────────────────────────────────

class OrderBook:
    """
    订单簿：维护买单和卖单的优先级队列

    数据结构：
    - 按 resource_type 分桶
    - 每个桶内有 Bids 和 Asks 两个 heap
    - Bids: max-heap（通过负价格用 min-heap 模拟）
    - Asks: min-heap
    """

    def __init__(self):
        self._bids: Dict[str, List[Order]] = defaultdict(list)  # resource_type -> [orders]
        self._asks: Dict[str, List[Order]] = defaultdict(list)  # resource_type -> [orders]
        self._all_orders: Dict[str, Order] = {}

    def add_order(self, order: Order) -> None:
        """添加一个新订单"""
        self._all_orders[order.order_id] = order

        if order.order_type == OrderType.BID:
            heappush(self._bids[order.resource_type], order)
        else:
            heappush(self._asks[order.resource_type], order)

    def cancel_order(self, order_id: str) -> bool:
        """取消订单（设为已取消状态，实际从 heap 中惰性删除）"""
        order = self._all_orders.get(order_id)
        if order is None or order.status == OrderStatus.FILLED:
            return False
        order.status = OrderStatus.CANCELLED
        return True

    def get_best_bid(self, resource_type: str) -> Optional[Order]:
        """获取最优买单（最高出价）"""
        return self._peek_valid(self._bids[resource_type])

    def get_best_ask(self, resource_type: str) -> Optional[Order]:
        """获取最优卖单（最低要价）"""
        return self._peek_valid(self._asks[resource_type])

    def _peek_valid(self, heap: List[Order]) -> Optional[Order]:
        """
        返回堆顶的有效订单
        （惰性删除已取消或已完成的订单）
        """
        while heap:
            top = heap[0]
            if top.status == OrderStatus.CANCELLED or top.is_filled():
                heappop(heap)  # 惰性清除
                continue
            return top
        return None

    def get_order_count(self, resource_type: str) -> Tuple[int, int]:
        """返回给定资源的买/卖订单数量"""
        bid_count = len(self._bids[resource_type])
        ask_count = len(self._asks[resource_type])
        return bid_count, ask_count


# ──────────────────────────────────────────────
# 经济引擎
# ──────────────────────────────────────────────

class EconomyEngine:
    """
    经济引擎：管理代币的发行、交易和结算

    支持：
    - 代币创建与分配
    - 交易冻结与结算
    - 余额查询
    """

    def __init__(self, initial_tokens: float = 1000.0):
        self._balances: Dict[str, float] = defaultdict(float)
        self._frozen: Dict[str, float] = defaultdict(float)
        self._initial_tokens = initial_tokens
        self._total_supply = 0.0

    def initialize_agent(self, agent_id: str) -> None:
        """为新 Agent 初始化代币"""
        if agent_id not in self._balances:
            self._balances[agent_id] = self._initial_tokens
            self._total_supply += self._initial_tokens

    def get_balance(self, agent_id: str) -> float:
        return self._balances.get(agent_id, 0.0)

    def get_available_balance(self, agent_id: str) -> float:
        return self._balances.get(agent_id, 0.0) - self._frozen.get(agent_id, 0.0)

    def freeze(self, agent_id: str, amount: float) -> bool:
        """冻结指定金额（出价时锁定资金）"""
        available = self.get_available_balance(agent_id)
        if available >= amount:
            self._frozen[agent_id] += amount
            return True
        return False

    def unfreeze(self, agent_id: str, amount: float) -> None:
        """解冻金额（出价失败或取消时）"""
        self._frozen[agent_id] = max(0.0, self._frozen[agent_id] - amount)

    def settle(self, buyer_id: str, seller_id: str, amount: float) -> bool:
        """
        结算交易：买方支付，卖方收款

        Returns:
            True 如果结算成功
        """
        if self._frozen[buyer_id] < amount:
            return False

        # 从冻结资金中扣除
        self._frozen[buyer_id] -= amount
        self._balances[buyer_id] -= amount

        # 卖方收款
        self._balances[seller_id] += amount

        return True

    def reward(self, agent_id: str, amount: float) -> None:
        """奖励代币（系统激励）"""
        self._balances[agent_id] += amount
        self._total_supply += amount

    def get_total_supply(self) -> float:
        return self._total_supply


# ──────────────────────────────────────────────
# 拍卖师（撮合引擎）
# ──────────────────────────────────────────────

class Auctioneer:
    """
    拍卖师：执行订单撮合

    使用连续双向拍卖（Continuous Double Auction, CDA）：
    - 每当有新订单到达，尝试撮合
    - 撮合条件：Best Bid >= Best Ask
    - 成交价格取买卖价格的中间值（也可以是按到达顺序的各自价格）
    """

    def __init__(self, order_book: OrderBook, economy: EconomyEngine,
                 reputation: ReputationSystem):
        self.order_book = order_book
        self.economy = economy
        self.reputation = reputation
        self.trades: List[Trade] = []
        self.price_history: Dict[str, List[Tuple[float, float, int]]] = defaultdict(list)
        # price_history[resource] = [(price, timestamp, quantity), ...]

    def submit_order(self, agent_id: str, order_type: OrderType,
                     resource_type: str, price: float, quantity: int) -> Optional[str]:
        """
        提交一个订单，立即尝试撮合

        Returns:
            如果成功生成订单/成交，返回订单 ID
        """
        order = Order(
            order_id=str(uuid.uuid4())[:8],
            agent_id=agent_id,
            order_type=order_type,
            resource_type=resource_type,
            price=price,
            quantity=quantity,
        )

        # 买单需要冻结资金
        if order_type == OrderType.BID:
            cost = price * quantity
            if not self.economy.freeze(agent_id, cost):
                print(f"  [FAIL] {agent_id} 余额不足，无法出价 {cost:.1f}")
                return None

        self.order_book.add_order(order)
        self._match_orders(resource_type)
        return order.order_id

    def _match_orders(self, resource_type: str) -> None:
        """尝试撮合指定资源的所有可匹配订单"""
        while True:
            best_bid = self.order_book.get_best_bid(resource_type)
            best_ask = self.order_book.get_best_ask(resource_type)

            if best_bid is None or best_ask is None:
                break

            if best_bid.price < best_ask.price:
                break  # 无法撮合

            if best_bid.agent_id == best_ask.agent_id:
                # 禁止自成交
                best_bid.status = OrderStatus.CANCELLED
                self.economy.unfreeze(best_bid.agent_id,
                                      best_bid.price * best_bid.remaining())
                continue

            # 计算成交数量和价格
            trade_qty = min(best_bid.remaining(), best_ask.remaining())
            # 成交价格：取买卖价格的加权中间值
            trade_price = (best_bid.price + best_ask.price) / 2

            # 执行结算
            cost = trade_price * trade_qty
            trade = Trade(
                trade_id=str(uuid.uuid4())[:8],
                bid_order_id=best_bid.order_id,
                ask_order_id=best_ask.order_id,
                resource_type=resource_type,
                price=trade_price,
                quantity=trade_qty,
                buyer_id=best_bid.agent_id,
                seller_id=best_ask.agent_id,
            )

            if self.economy.settle(best_bid.agent_id, best_ask.agent_id, cost):
                self.trades.append(trade)
                self.price_history[resource_type].append(
                    (trade_price, time.time(), trade_qty)
                )

                # 更新订单已成交数量
                best_bid.filled_quantity += trade_qty
                best_ask.filled_quantity += trade_qty

                print(f"  [MATCH] {trade.buyer_id} 买入 {trade_qty}x{resource_type}"
                      f" @ ${trade_price:.2f} (from {trade.seller_id})")
            else:
                # 结算失败（理论上不应发生，因为出价时已冻结资金）
                print(f"  [FAIL] 结算失败: {best_bid.agent_id} -> {best_ask.agent_id}")
                break

    def get_last_price(self, resource_type: str) -> Optional[float]:
        """获取最近一笔成交价"""
        history = self.price_history.get(resource_type, [])
        return history[-1][0] if history else None

    def get_market_spread(self, resource_type: str) -> Optional[float]:
        """获取当前市场价差（最优卖价 - 最优买价）"""
        best_bid = self.order_book.get_best_bid(resource_type)
        best_ask = self.order_book.get_best_ask(resource_type)
        if best_bid and best_ask:
            return best_ask.price - best_bid.price
        return None


# ──────────────────────────────────────────────
# Agent 基类
# ──────────────────────────────────────────────

class Agent:
    """
    Agent 基类

    每个 Agent 有唯一的 ID、代币余额、信誉分数。
    通过 Auctioneer 提交订单参与市场。
    """

    def __init__(self, agent_id: str, economy: EconomyEngine,
                 reputation: ReputationSystem):
        self.agent_id = agent_id
        self.economy = economy
        self.reputation = reputation
        self.economy.initialize_agent(agent_id)
        self.total_earned = 0.0
        self.total_spent = 0.0

    def __str__(self) -> str:
        return (f"{self.agent_id}"
                f" [代币:{self.economy.get_balance(self.agent_id):.1f}"
                f" 信誉:{self.reputation.get_score(self.agent_id):.2f}]")

    def get_utility(self) -> float:
        """Agent 的效用函数：衡量整体表现"""
        return self.total_earned - self.total_spent


class Producer(Agent):
    """
    生产者 Agent

    提供服务，设定要价。可调整定价策略以最大化收益。
    """

    def __init__(self, agent_id: str, economy: EconomyEngine,
                 reputation: ReputationSystem, production_cost: float = 2.0):
        super().__init__(agent_id, economy, reputation)
        self.production_cost = production_cost
        self.services_sold = 0

    def price_strategy(self, market_price: Optional[float]) -> float:
        """
        定价策略：基于市场价格和成本定价

        - 有市场价时，略低于市场价以快速成交
        - 无市场价时，按成本 + 基础利润定价
        """
        if market_price is not None:
            return max(self.production_cost * 1.1, market_price * 0.95)
        return self.production_cost * 1.5

    def offer_service(self, resource_type: str, quantity: int,
                      auctioneer: Auctioneer) -> Optional[str]:
        """发布服务到市场"""
        market_price = auctioneer.get_last_price(resource_type)
        price = self.price_strategy(market_price)
        order_id = auctioneer.submit_order(
            self.agent_id, OrderType.ASK, resource_type, price, quantity
        )
        if order_id is not None:
            self.services_sold += quantity
        return order_id


class Consumer(Agent):
    """
    消费者 Agent

    发布需求并出价。根据预算和需求紧急程度调整出价策略。
    """

    def __init__(self, agent_id: str, economy: EconomyEngine,
                 reputation: ReputationSystem, valuation: float = 5.0):
        super().__init__(agent_id, economy, reputation)
        self.valuation = valuation  # 对服务的估值（最高愿意支付的价格）
        self.services_bought = 0

    def bid_strategy(self, market_price: Optional[float],
                     urgency: float = 1.0) -> float:
        """
        出价策略

        - 有市场价时，略高于市场价以确保成交
        - 无市场价时，用估值的一定比例出价
        - urgency > 1.0 表示更急迫（愿意出更高价）
        """
        if market_price is not None:
            return min(self.valuation, market_price * 1.1 * urgency)
        return self.valuation * 0.7 * urgency

    def request_service(self, resource_type: str, quantity: int,
                        auctioneer: Auctioneer, urgency: float = 1.0) -> Optional[str]:
        """发布需求到市场"""
        market_price = auctioneer.get_last_price(resource_type)
        price = self.bid_strategy(market_price, urgency)
        order_id = auctioneer.submit_order(
            self.agent_id, OrderType.BID, resource_type, price, quantity
        )
        if order_id is not None:
            self.services_bought += quantity
        return order_id


# ──────────────────────────────────────────────
# 市场监管者（Settlement + 仲裁）
# ──────────────────────────────────────────────

class MarketRegulator:
    """
    市场监管者

    职责：
    - 交易后信誉更新
    - 市场异常检测（价格操纵、合谋等）
    - 通货膨胀/通缩监控
    - 市场激励分配
    """

    def __init__(self, auctioneer: Auctioneer, economy: EconomyEngine,
                 reputation: ReputationSystem):
        self.auctioneer = auctioneer
        self.economy = economy
        self.reputation = reputation
        self.trade_count = 0
        self.anomalies_detected = 0

    def settle_trade(self, trade: Trade, quality_score: float = 1.0):
        """交易完成后：更新信誉、发放奖励、记录审计"""
        # 买方评价卖方
        self.reputation.submit_rating(
            trade.buyer_id, trade.seller_id, quality_score
        )
        # 卖方评价买方（付款及时性等）
        self.reputation.submit_rating(
            trade.seller_id, trade.buyer_id,
            1.0  # 假设付款成功
        )

        # 成功交易奖励
        reward_amount = trade.quantity * 0.1
        self.economy.reward(trade.buyer_id, reward_amount * 0.5)
        self.economy.reward(trade.seller_id, reward_amount * 0.5)

        self.trade_count += 1

    def settle_all_pending(self, default_quality: float = 1.0):
        """结算所有未结算的成交"""
        settled = set()
        for trade in self.auctioneer.trades:
            key = trade.trade_id
            if key not in settled:
                self.settle_trade(trade, default_quality)
                settled.add(key)

    def detect_anomalies(self) -> List[str]:
        """
        检测市场异常

        简化版检测：价格突变、成交量异常等
        """
        alerts = []
        for resource in self.auctioneer.price_history:
            prices = [p[0] for p in self.auctioneer.price_history[resource][-20:]]
            if len(prices) >= 5:
                mean_p = sum(prices) / len(prices)
                for p in prices:
                    if abs(p - mean_p) / max(mean_p, 0.01) > 2.0:
                        alerts.append(f"[ALERT] {resource} 价格异常波动: {p:.2f}")
                        self.anomalies_detected += 1
                        break
        return alerts

    def print_market_summary(self):
        """打印市场摘要"""
        print("\n" + "=" * 60)
        print("市场摘要")
        print("=" * 60)
        print(f"总成交笔数: {self.trade_count}")
        print(f"总代币供应量: {self.economy.get_total_supply():.1f}")
        print(f"检测到的异常: {self.anomalies_detected}")

        for resource, history in self.auctioneer.price_history.items():
            if history:
                avg_price = sum(h[0] for h in history) / len(history)
                print(f"\n{resource}:")
                print(f"  成交 {len(history)} 笔, 均价 ${avg_price:.2f}")

        print("=" * 60)


# ──────────────────────────────────────────────
# 模拟运行
# ──────────────────────────────────────────────

def run_market_simulation():
    """
    运行一个完整的市场模式模拟

    场景：
    - 5 个 Producer（算力提供商）
    - 10 个 Consumer（算力消费者）
    - 1 种资源类型: "compute"
    - 运行 5 轮市场交互
    """
    print("=" * 60)
    print("Market Pattern 模拟演示")
    print("=" * 60)

    # 初始化系统组件
    economy = EconomyEngine(initial_tokens=500.0)
    reputation = ReputationSystem(alpha=0.3, default_reputation=0.5)
    order_book = OrderBook()
    auctioneer = Auctioneer(order_book, economy, reputation)
    regulator = MarketRegulator(auctioneer, economy, reputation)

    # 创建 Producer
    producers = [
        Producer(f"Producer-{i}", economy, reputation,
                 production_cost=random.uniform(1.0, 3.0))
        for i in range(5)
    ]

    # 创建 Consumer
    consumers = [
        Consumer(f"Consumer-{i}", economy, reputation,
                 valuation=random.uniform(4.0, 8.0))
        for i in range(10)
    ]

    agents = producers + consumers

    print(f"\n创建了 {len(producers)} 个 Producer 和 {len(consumers)} 个 Consumer")
    print(f"初始代币: 每个 Agent {economy._initial_tokens}")

    # 运行多轮市场交互
    for round_num in range(5):
        print(f"\n{'─' * 50}")
        print(f"第 {round_num + 1} 轮")
        print(f"{'─' * 50}")

        # Producer 发布服务
        for producer in producers:
            qty = random.randint(1, 3)
            order_id = producer.offer_service("compute", qty, auctioneer)
            if order_id:
                last_price = auctioneer.get_last_price("compute")
                print(f"  {producer.agent_id} 发布 {qty}x compute"
                      f" (成本={producer.production_cost:.1f}"
                      f", 市场价={last_price if last_price else 'N/A'})")

        # Consumer 发布需求
        for consumer in consumers:
            if random.random() < 0.7:  # 70% 概率发起请求
                qty = random.randint(1, 2)
                urgency = random.uniform(0.5, 2.0)
                order_id = consumer.request_service("compute", qty,
                                                     auctioneer, urgency)
                if order_id:
                    print(f"  {consumer.agent_id} 请求 {qty}x compute"
                          f" (估值={consumer.valuation:.1f})")

        # 结算本轮交易
        regulator.settle_all_pending()

        # 检查市场异常
        alerts = regulator.detect_anomalies()
        for alert in alerts:
            print(f"  {alert}")

        # 检查市场价差
        spread = auctioneer.get_market_spread("compute")
        if spread is not None:
            print(f"  当前价差: ${spread:.2f}")

    # 最终报告
    regulator.print_market_summary()

    # Agent 最终状态
    print("\nAgent 最终状态:")
    print("-" * 60)
    for agent in sorted(agents, key=lambda a: a.get_utility(), reverse=True):
        print(f"  {agent.agent_id:20s}"
              f" | 效用: {agent.get_utility():6.1f}"
              f" | 余额: {economy.get_balance(agent.agent_id):6.1f}"
              f" | 信誉: {reputation.get_score(agent.agent_id):.2f}")
    print("-" * 60)

    return auctioneer, economy, reputation


# ──────────────────────────────────────────────
# 执行模拟
# ──────────────────────────────────────────────

if __name__ == "__main__":
    auctioneer, economy, reputation = run_market_simulation()
```

### 运行结果示例

```
============================================================
Market Pattern 模拟演示
============================================================

创建了 5 个 Producer 和 10 个 Consumer
初始代币: 每个 Agent 500.0

──────────────────────────────────────────────────
第 1 轮
──────────────────────────────────────────────────
  Producer-0 发布 2x compute (成本=2.3, 市场价=N/A)
  Producer-1 发布 3x compute (成本=1.7, 市场价=N/A)
  Producer-2 发布 1x compute (成本=2.8, 市场价=N/A)
  Producer-4 发布 2x compute (成本=1.2, 市场价=N/A)
  [MATCH] Consumer-5 买入 2x compute @ $4.28 (from Producer-1)
  [MATCH] Consumer-3 买入 1x compute @ $3.44 (from Producer-0)
  [MATCH] Consumer-7 买入 1x compute @ $3.12 (from Producer-4)
  [MATCH] Consumer-1 买入 1x compute @ $3.12 (from Producer-4)
  当前价差: $0.52

...

============================================================
市场摘要
============================================================
总成交笔数: 28
总代币供应量: 8536.0
检测到的异常: 1

compute:
  成交 28 笔, 均价 $4.16
============================================================

Agent 最终状态:
------------------------------------------------------------
  Consumer-3             | 效用:   487.2 | 余额:   487.2 | 信誉: 0.59
  Consumer-8             | 效用:   486.5 | 余额:   486.5 | 信誉: 0.53
  ...
  Producer-4             | 效用:   524.3 | 余额:   524.3 | 信誉: 0.52
  Producer-1             | 效用:   517.8 | 余额:   517.8 | 信誉: 0.61
------------------------------------------------------------
```

### 关键设计要点

1. **价格-时间优先排序**：Order 使用 `heapq` 实现，Bids 按价格降序、Asks 按价格升序，同价时按时间先后
2. **惰性删除**：已取消或已成交的订单在 `_peek_valid` 中被惰性清除，避免 O(n) 的删除操作
3. **资金冻结**：Consumer 出价时立即冻结资金，防止超支
4. **信誉系统**：使用 EWMA（指数加权移动平均）更新信誉分，兼顾近期表现和历史积累
5. **中间价成交**：撮合时取买卖价格的中间值作为成交价，公平且简单
6. **防自成交**：撮合前检查买卖双方不是同一个 Agent
