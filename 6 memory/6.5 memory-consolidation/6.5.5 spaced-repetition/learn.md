# 6.5.5 间隔重复强化 (Spaced Repetition)

> 基于遗忘曲线的记忆巩固机制 —— 从艾宾浩斯遗忘曲线到 AI Agent 记忆系统的自动化复习调度

---

## 1. 简单介绍

**间隔重复 (Spaced Repetition)** 是一种在逐渐增加的时间间隔内反复激活记忆痕迹 (memory trace) 的技术，核心目标是在记忆即将遗忘的临界点进行复习，从而最大化每次复习的效益。

在 AI Agent 的记忆系统中，间隔重复承担以下角色：

- **记忆巩固的最终调度器**：决定"哪条记忆何时被再次激活"
- **遗忘曲线的对抗引擎**：通过数学建模预测每条记忆的遗忘进度，在最优时刻触发复习
- **计算资源的分配器**：将有限的检索/计算预算优先分配给最重要的、即将被遗忘的记忆

间隔重复位于 6.5 记忆巩固管线的第 5 步（最后一步），在 6.5.1 编码 (encoding)、6.5.2 重激活 (reactivation)、6.5.3 模式完成 (pattern completion)、6.5.4 时间衰减 (temporal decay) 之后，负责对记忆进行强化调度。

---

## 2. 基本原理

### 2.1 艾宾浩斯遗忘曲线 (Ebbinghaus Forgetting Curve)

Hermann Ebbinghaus 在 1885 年发现：**遗忘是指数级的**。刚学完的信息遗忘最快，之后遗忘速度逐渐放缓。

```
  记忆保留率 R(t)
  1.00 |▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
  0.90 |    ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
  0.80 |         ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
  0.70 |              ▓▓▓▓▓▓▓▓▓▓▓▓▓▓
  0.60 |                   ▓▓▓▓▓▓▓▓
  0.50 |                        ▓▓▓▓
  0.40 |                           ▓▓
  0.30 |                            ▓
  0.20 |                            ▓
  0.10 |                            ▓
  0.00 └─────────────────────────────────────▶ 时间 t
       0    1d    2d    7d    14d   30d
```

核心公式：

```
R = e^(-t / S)
```

其中：
- **R** (Retrievability)：记忆的可检索概率 (0~1)
- **t**：自上次复习以来的时间
- **S** (Stability / Strength)：记忆强度，即 R 衰减到约 37% 所需的时间

### 2.2 间隔效应 (Spacing Effect)

**间隔效应**是指：将复习分散到多个时间点，比在同一个时间点集中复习多次效果更好。这是因为每次复习时，记忆需要被"重新检索"——这种检索过程本身就是一种强化。

间隔重复的核心洞察在于：

```
一次复习带来的强度增益 ∝ 复习时记忆的检索难度
```

如果复习得太早（记忆还很新鲜），检索几乎不费力气，增益很小。
如果复习得太晚（记忆几乎遗忘），检索失败，没有增益。
**最优复习点**就在记忆即将跌破某个可检索阈值时——这是间隔重复要解决的核心优化问题。

### 2.3 检索强化效应 (Retrieval Practice Effect / Testing Effect)

每次成功检索记忆都会：

1. **增加强度 S**：使下一次遗忘曲线变得更加平缓
2. **增加间隔**：下次复习可以等更长时间
3. **多重记忆痕迹**：建立更多关联路径，提高冗余度

这与 6.4.2 时间衰减形成互补：衰减在消除噪音，而间隔重复在选择性地对抗信号衰减。

---

## 3. 背景与演进

### 3.1 时间线

```
1885 ─── Ebbinghaus 遗忘曲线研究
        │
1970s ── Leitner 卡片盒系统（物理纸牌）
        │
1987 ── SM-0 → SM-2 算法 (Piotr Woźniak, SuperMemo)
        │
1990s ── SM-4 → SM-5 → SM-6 → SM-8 迭代
        │
2006 ── Anki 发布（基于 SM-2 的改良版本）
        │
2011 ── Memrise, Duolingo 等应用采用间隔重复
        │
2019 ── FSRS (Free Spaced Repetition Scheduler) 提出
        │
2023 ── FSRS v4 成为 Anki 默认算法
        │
2025 ── FSRS 优化 + Agent 记忆系统采纳
```

### 3.2 Leitner 卡片盒系统 (Leitner System)

Sebastian Leitner 在 1970 年代提出的物理卡片系统：

- 多个盒子 (boxes)，每个盒子代表不同的复习间隔
- 答对的卡片升级到更高盒子（间隔更长）
- 答错的卡片降级到第一个盒子（立即复习）

```
 Box 1 (1天)    Box 2 (3天)    Box 3 (7天)    Box 4 (14天)   Box 5 (30天)
 ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
 │ Card A   │   │ Card C   │   │ Card E   │   │ Card G   │   │ Card I   │
 │ Card B   │──▶│ Card D   │──▶│ Card F   │──▶│ Card H   │──▶│ Card J   │
 │          │   │          │   │          │   │          │   │          │
 └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
      ▲                                                    │
      └───────────────── 答错降级 ──────────────────────────┘
```

### 3.3 SM-2 算法 (SuperMemo 2)

Piotr Woźniak 1987 年发表的里程碑算法，是计算机化间隔重复的起点。

**核心参数：**
- **n**：连续正确回答次数 (repetition count)
- **EF**：易度因子 (Easiness Factor, 初始值 2.5, 范围 1.3~∞)
- **I**：间隔天数 (interval)
- **q**：自我评分 (quality, 0~5)

**间隔计算：**

```
I(1) = 1 天        （第一次复习）
I(2) = 6 天        （第二次复习）
I(n) = I(n-1) × EF  （n ≥ 3 时）
```

**EF 更新公式：**

```
EF' = EF + (0.1 - (5 - q) × (0.08 + (5 - q) × 0.02))
EF' = max(1.3, EF')
```

**质量阈值：**
- q ≥ 3：保留递增间隔，n++
- q < 3：重置 n = 0，回到间隔 1 天

### 3.4 FSRS (Free Spaced Repetition Scheduler)

FSRS 由 Jarrett Ye 开发，是现代间隔重复算法的最新进展。与 SM-2 的根本区别在于：

- **SM-2**：基于规则，使用固定的 EF 公式
- **FSRS**：基于数据驱动，使用机器学习优化参数

**FSRS 的三维模型：**
- **D** (Difficulty)：记忆项的内在难度，范围 1~10
- **S** (Stability)：记忆强度，以天为单位
- **R** (Retrievability)：可检索概率

**核心思想：** 不再用单一的 EF 来驱动间隔增长，而是用算法拟合用户的历史复习数据，找到最优参数集，然后预测每条记忆的最佳复习时间。

---

## 4. 核心矛盾

间隔重复在 Agent 记忆系统中面临的根本性矛盾：

### 4.1 复习频率 vs. 计算成本

```
           高频复习（强记忆）                     低频复习（节省资源）
                │                                       │
                │   ◄── 这里是最优平衡点 ──►            │
                ▼                                       ▼
      [记忆牢固但计算开销大]                  [计算节省但可能遗忘]
```

- 每条记忆的复习都需要 CPU/时间开销
- 在 Agent 系统中，记忆可能成千上万甚至上百万
- 必须在"强化所有记忆"和"只强化关键记忆"之间做选择

### 4.2 强化什么 vs. 让什么遗忘

不是所有记忆都值得保留。Agent 面临的关键问题：

```
值得强化的记忆：
├── 高频访问的事实（用户偏好，常用知识）
├── 特定任务的关键上下文
├── 跨 session 的长期承诺
└── 重要的 learn 文件内容

可以衰减的记忆：
├── 一次性对话的临时上下文
├── 过时的状态信息
├── 低置信度的推测
└── 已经被替换的旧知识
```

### 4.3 固定调度 vs. 动态需求

- **固定调度**（如 Anki 式）：每天固定时刻复习一批卡片——简单但可能错过关键时间点
- **动态调度**：实时计算每条记忆的遗忘曲线，在最佳时刻触发——精确但计算代价高

---

## 5. 关键算法详解

### 5.1 Leitner 算法

最简单的间隔重复实现，适合 Agent 的轻量级优先级分类。

```python
import heapq
import time
from enum import Enum
from dataclasses import dataclass, field
from typing import Any, Optional

class LeitnerBox(Enum):
    """Leitner 盒子等级，对应不同的复习间隔（秒）"""
    BOX_1 = 86400       # 1 天
    BOX_2 = 259200      # 3 天
    BOX_3 = 604800      # 7 天
    BOX_4 = 1209600     # 14 天
    BOX_5 = 2592000     # 30 天

    @staticmethod
    def next(box: "LeitnerBox") -> "LeitnerBox":
        """升级到下一个盒子"""
        members = list(LeitnerBox)
        idx = members.index(box)
        return members[min(idx + 1, len(members) - 1)]

    @staticmethod
    def reset() -> "LeitnerBox":
        """答错时回到第一个盒子"""
        return LeitnerBox.BOX_1


@dataclass
class LeitnerItem:
    """Leitner 系统中的记忆项"""
    memory_id: str
    content: Any
    box: LeitnerBox = LeitnerBox.BOX_1
    last_reviewed: float = 0.0
    importance: float = 1.0  # 重要性权重 [0, 1]

    def is_due(self, now: float) -> bool:
        """判断是否到期复习"""
        return (now - self.last_reviewed) >= self.box.value

    def review_success(self):
        """回答正确：升级"""
        self.box = LeitnerBox.next(self.box)
        self.last_reviewed = time.time()

    def review_failure(self):
        """回答失败：降级到 box 1"""
        self.box = LeitnerBox.reset()
        self.last_reviewed = time.time()


class LeitnerScheduler:
    """Leitner 调度器——适合轻量级 Agent 记忆优先级管理"""

    def __init__(self):
        self.items: dict[str, LeitnerItem] = {}
        # 优先级队列：按 (应复习时间, 重要性) 逆序排列
        self._priority_queue: list[tuple[float, float, str]] = []

    def add_item(self, memory_id: str, content: Any,
                 importance: float = 1.0) -> None:
        """添加新记忆项"""
        item = LeitnerItem(
            memory_id=memory_id,
            content=content,
            importance=importance,
            last_reviewed=time.time()
        )
        self.items[memory_id] = item
        self._push_to_queue(item)

    def _push_to_queue(self, item: LeitnerItem) -> None:
        """将记忆项加入优先级队列"""
        now = time.time()
        due_time = item.last_reviewed + item.box.value
        # 使用负重要性实现最大堆效果
        heapq.heappush(
            self._priority_queue,
            (due_time, -item.importance, item.memory_id)
        )

    def get_due_items(self, now: float | None = None,
                      limit: int = 10) -> list[LeitnerItem]:
        """获取到期的、优先级最高的前 N 条记忆"""
        now = now or time.time()
        results = []
        temp_queue = []

        while self._priority_queue and len(results) < limit:
            due_time, neg_imp, mid = heapq.heappop(self._priority_queue)
            # 可能有过期的队列项（降级后旧记录）
            if mid not in self.items:
                continue
            item = self.items[mid]
            if item.is_due(now):
                results.append(item)
            # 未到期但很重要，保留待查
            temp_queue.append((due_time, neg_imp, mid))

        # 重建优先级队列（保留未获取的项）
        self._priority_queue = temp_queue + self._priority_queue
        heapq.heapify(self._priority_queue)

        return results

    def review(self, memory_id: str, success: bool) -> None:
        """执行一次复习"""
        if memory_id not in self.items:
            return
        if success:
            self.items[memory_id].review_success()
        else:
            self.items[memory_id].review_failure()
        # 重新入队
        self._push_to_queue(self.items[memory_id])
```

**优点：** 简单、零参数、适合快速原型
**缺点：** 硬编码间隔、没有个体差异化、无法区分"基本记住"和"非常熟悉"

---

### 5.2 SM-2 算法 (SuperMemo 2)

SM-2 是历史上最具影响力的间隔重复算法，也是 Anki 原始算法的基础。

```python
"""
SM-2 (SuperMemo 2) 算法完整实现

参考: Piotr Woźniak, 1987
适用于: Agent 记忆系统中需要差异化间隔计算的场景
"""

import time
import math
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class SM2Item:
    """
    SM-2 记忆项

    Attributes:
        memory_id: 唯一标识
        content: 记忆内容（可以是任何类型）
        repetition: 连续正确回答次数 (n)
        easiness_factor: 易度因子 (EF)，初始值 2.5
        interval: 当前间隔（天）
        last_review: 上次复习的时间戳
        next_review: 下次应复习的时间戳
        importance: 重要性权重 [0, 1]
    """
    memory_id: str
    content: str
    repetition: int = 0
    easiness_factor: float = 2.5
    interval: int = 0
    last_review: float = 0.0
    next_review: float = 0.0
    importance: float = 1.0
    access_count: int = 0

    def is_due(self, now: float | None = None) -> bool:
        now = now or time.time()
        return now >= self.next_review

    def days_overdue(self, now: float | None = None) -> int:
        now = now or time.time()
        return max(0, int((now - self.next_review) / 86400))


class SM2Scheduler:
    """
    SM-2 调度器

    质量评分 (quality) 说明:
        5 - 完美回忆，没有犹豫
        4 - 正确回忆，略有犹豫
        3 - 正确回忆，但有较大困难
        2 - 错误回忆，但正确答案感觉很熟悉
        1 - 错误回忆，记忆模糊
        0 - 完全遗忘
    """

    # 默认参数
    DEFAULT_EF = 2.5
    MIN_EF = 1.3
    MAX_EF = 10.0  # 防止 EF 过度增长
    INTERVAL_1 = 1   # 第一次复习间隔（天）
    INTERVAL_2 = 6   # 第二次复习间隔（天）

    def __init__(self):
        self.items: dict[str, SM2Item] = {}
        self._review_history: list[dict] = []  # 复习日志（可用于分析）

    def add_item(self, memory_id: str, content: str,
                 importance: float = 1.0) -> SM2Item:
        """
        添加新的记忆项

        新项的 next_review 设为 1 天后
        """
        now = time.time()
        item = SM2Item(
            memory_id=memory_id,
            content=content,
            last_review=now,
            next_review=now + 86400,  # 1 天后首次复习
            importance=importance,
        )
        self.items[memory_id] = item
        return item

    def review(self, memory_id: str, quality: int) -> dict:
        """
        执行一次复习，更新记忆参数

        Args:
            memory_id: 记忆项 ID
            quality: 自我评分 0-5

        Returns:
            更新后的记忆参数摘要
        """
        if memory_id not in self.items:
            raise KeyError(f"记忆项 {memory_id} 不存在")

        item = self.items[memory_id]
        quality = max(0, min(5, quality))  # 限制在 [0, 5]

        # 记录复习前的状态
        before = {
            "repetition": item.repetition,
            "ef": item.easiness_factor,
            "interval": item.interval,
        }

        # ----- SM-2 核心逻辑 -----

        # 1. 更新易度因子 EF
        item.easiness_factor = self._update_ef(item.easiness_factor, quality)

        # 2. 根据质量决定间隔策略
        if quality >= 3:
            # 回答正确：递增间隔
            if item.repetition == 0:
                item.interval = self.INTERVAL_1
            elif item.repetition == 1:
                item.interval = self.INTERVAL_2
            else:
                item.interval = round(item.interval * item.easiness_factor)
            item.repetition += 1
        else:
            # 回答错误：重置
            item.repetition = 0
            item.interval = self.INTERVAL_1

        # 3. 更新时间戳
        now = time.time()
        item.last_review = now
        item.next_review = now + item.interval * 86400
        item.access_count += 1

        # 记录日志
        log_entry = {
            "memory_id": memory_id,
            "time": now,
            "quality": quality,
            "before": before,
            "after": {
                "repetition": item.repetition,
                "ef": item.easiness_factor,
                "interval": item.interval,
            }
        }
        self._review_history.append(log_entry)

        return log_entry

    def _update_ef(self, old_ef: float, quality: int) -> float:
        """
        SM-2 易度因子更新公式

        EF' = EF + (0.1 - (5 - q) * (0.08 + (5 - q) * 0.02))

        约束: EF' >= 1.3
        """
        delta = 0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02)
        new_ef = old_ef + delta
        return max(self.MIN_EF, min(self.MAX_EF, new_ef))

    def get_due_items(self, now: float | None = None,
                      limit: int = 50,
                      min_importance: float = 0.0) -> list[SM2Item]:
        """
        获取到期待复习的记忆项，按优先级排序

        优先级 = 重要性 × (1 + 逾期天数 × 0.1)
        """
        now = now or time.time()
        due = []
        for item in self.items.values():
            if item.is_due(now) and item.importance >= min_importance:
                overdue = item.days_overdue(now)
                priority = item.importance * (1 + overdue * 0.1)
                due.append((priority, item))

        # 按优先级降序排列
        due.sort(key=lambda x: -x[0])
        return [item for _, item in due[:limit]]

    def predict_retrievability(self, memory_id: str) -> float:
        """
        预测当前记忆的可检索概率 R

        使用指数衰减模型: R = e^(-t/s)
        其中 s = interval × (1 + repetition × 0.5)
              t = 自上次复习以来的时间（天）
        """
        if memory_id not in self.items:
            return 0.0

        item = self.items[memory_id]
        now = time.time()
        days_since_review = (now - item.last_review) / 86400

        if days_since_review <= 0:
            return 1.0

        # 估计记忆强度: 间隔越长 / 重复次数越多，强度越大
        memory_strength = item.interval * (1 + item.repetition * 0.5)
        memory_strength = max(memory_strength, 0.1)

        # 遗忘曲线公式
        retrievability = math.exp(-days_since_review / memory_strength)
        return max(0.0, min(1.0, retrievability))

    def get_statistics(self) -> dict:
        """获取调度器统计信息"""
        now = time.time()
        total = len(self.items)
        due = sum(1 for item in self.items.values() if item.is_due(now))

        avg_ef = sum(
            item.easiness_factor for item in self.items.values()
        ) / max(total, 1)

        avg_r = sum(
            self.predict_retrievability(mid)
            for mid in self.items
        ) / max(total, 1)

        return {
            "total_items": total,
            "due_items": due,
            "average_ef": round(avg_ef, 2),
            "average_retrievability": round(avg_r, 3),
            "total_reviews": len(self._review_history),
        }


# ===== 使用示例 =====
def demo_sm2():
    """SM-2 调度器演示"""
    scheduler = SM2Scheduler()

    # 添加三条重要程度不同的记忆
    scheduler.add_item("agent:user_prefer_temp", "默认温度偏好: 0.7",
                       importance=0.9)
    scheduler.add_item("agent:tool_format", "工具调用格式说明",
                       importance=0.8)
    scheduler.add_item("agent:session_greeting",
                       "用户上次告别语: '明天继续聊'", importance=0.3)

    # 模拟第一轮复习
    print("=== 第一轮复习 ===")
    for mid in ["agent:user_prefer_temp", "agent:tool_format",
                 "agent:session_greeting"]:
        quality = 5 if scheduler.items[mid].importance > 0.5 else 3
        result = scheduler.review(mid, quality)
        print(f"  {mid}: 质量={quality}, "
              f"间隔={result['after']['interval']}天, "
              f"EF={result['after']['ef']:.2f}")

    # 模拟第二轮复习（仅高重要性项）
    print("\n=== 第二轮复习（7天后） ===")
    # 手动推进时间...
    for mid in ["agent:user_prefer_temp", "agent:tool_format"]:
        quality = 4
        result = scheduler.review(mid, quality)
        print(f"  {mid}: 质量={quality}, "
              f"间隔={result['after']['interval']}天, "
              f"EF={result['after']['ef']:.2f}")

    print(f"\n=== 统计 === {scheduler.get_statistics()}")


if __name__ == "__main__":
    demo_sm2()
```

**SM-2 的核心公式总结：**

```
EF_n+1 = EF_n + (0.1 - (5 - q) × (0.08 + (5 - q) × 0.02))
EF_n+1 = max(1.3, EF_n+1)

间隔策略:
  n=0: I₁ = 1
  n=1: I₂ = 6
  n≥2: Iₙ = Iₙ₋₁ × EF
```

---

### 5.3 FSRS (Free Spaced Repetition Scheduler)

FSRS 是当前最先进的间隔重复算法。它的核心思想是用**数据驱动**替代 SM-2 的**硬编码规则**。

#### 三维模型

```
┌─────────────────────────────────────────────────────────┐
│                FSRS 记忆状态空间                          │
│                                                         │
│   难度 D (Difficulty) ─────── 记忆的内在难度              │
│   [1.0 (极简单) ......... 10.0 (极困难)]                 │
│                                                         │
│   稳定性 S (Stability) ───── 记忆强度指标                │
│   [R 衰减到 90% 所需的天数]                              │
│                                                         │
│   可检索性 R (Retrievability) ── 当前的记忆可访问概率    │
│   [0.0 (完全遗忘) ......... 1.0 (完美保持)]              │
└─────────────────────────────────────────────────────────┘
```

#### 核心公式

**可检索性预测：**

```
R(t) = 2^(-t / S)
```

其中 t 是自上次复习以来的天数，S 是稳定性。

**稳定性更新（复习成功后）：**

FSRS 使用一组可训练的权重 w，通过以下函数更新稳定性：

```
S' = f(S, D, grade) = S × e^(w₁ × (grade - 3) + w₂ × (grade - 3)² + w₃)
```

具体来说，FSRS v4 使用 17 个可优化参数，根据用户的历史复习数据通过梯度下降进行优化。

**难度更新：**

```
D' = D + Δ(grade, D)
```

每次复习的评分 grade（同 SM-2 的 quality）会影响难度的微调。

#### FSRS vs. SM-2 对比

| 维度 | SM-2 | FSRS |
|------|------|------|
| 参数集 | 1 个 (EF) | 17+ 个可训练参数 |
| 间隔增长 | 规则驱动 (EF × I) | 数据驱动 (ML 优化) |
| 个体差异化 | 固定规则 | 根据用户历史学习 |
| 初始间隔 | 硬编码 (1, 6, ...) | 根据数据自动校准 |
| 适用场景 | 简单场景、冷启动 | 大规模、长周期 |

#### 在 Agent 记忆中的简化 FSRS

对于 Agent 系统，可以借鉴 FSRS 的思想但做简化：

```python
"""
简化的 FSRS 风格调度器——适用于 Agent 记忆系统

核心思想:
  1. 用 D (难度) 和 S (稳定性) 两个维度描述记忆状态
  2. 每次复习后根据评分更新 D 和 S
  3. 基于 R(t) 决定最佳复习时间
"""

import time
import math
from dataclasses import dataclass
from typing import Optional


@dataclass
class FSRSMemoryItem:
    """
    FSRS 风格记忆项

    状态空间: [D, S] → 预测 R(t)
    """
    memory_id: str
    content: str
    difficulty: float = 5.0       # D: 1~10, 默认 5.0
    stability: float = 1.0        # S: 记忆强度（天）
    retrievability: float = 1.0   # R: 当前可检索概率
    last_review: float = 0.0      # 上次复习时间戳
    importance: float = 1.0       # 外部重要性权重
    access_count: int = 0

    def compute_stability(self, grade: int) -> float:
        """
        简化的稳定性更新函数

        真实 FSRS 使用 17 参数 ML 模型。
        这里用一个近似公式模拟其行为：
          grade (0-4) 对应 Anki 中的 Again/Hard/Good/Easy

        核心观察：
          - grade 越高，稳定性增长越快
          - 难度 D 越大的项，稳定性增长越慢
        """
        # 从 grade (0-4) 映射到增益因子
        gain_map = {
            0: 0.5,    # Again  → 稳定性减半
            1: 0.8,    # Hard   → 略降
            2: 1.5,    # Good   → 适度增长
            3: 2.5,    # Easy   → 大幅增长
            4: 3.5,    # Perfect→ 最大增长
        }
        base_gain = gain_map.get(grade, 1.0)

        # 难度调节：高难度项增长更慢
        difficulty_penalty = 1.0 / (1.0 + 0.1 * (self.difficulty - 1.0))

        # 稳定性增长
        new_stability = self.stability * base_gain * difficulty_penalty
        return max(0.1, new_stability)

    def compute_difficulty(self, grade: int) -> float:
        """
        简化的难度更新

        连续高分 → 降低难度
        连续低分 → 增加难度
        """
        if grade >= 3:
            # 成功回忆 → 略微降低难度
            delta = -0.2 * (grade - 2)
        elif grade >= 1:
            # 部分回忆 → 微增难度
            delta = 0.3 * (3 - grade)
        else:
            # 完全遗忘 → 大幅增加难度
            delta = 1.0

        return max(1.0, min(10.0, self.difficulty + delta))

    def predict_retrievability(self, now: float | None = None) -> float:
        """预测当前的可检索概率 R = 2^(-t/S)"""
        now = now or time.time()
        days_since = (now - self.last_review) / 86400
        if days_since <= 0:
            return 1.0
        self.retrievability = math.pow(2, -days_since / max(self.stability, 0.1))
        return self.retrievability

    def optimal_review_time(self, threshold: float = 0.7) -> float:
        """
        计算最佳复习时间——当 R 降至 threshold 时

        R(t) = threshold
        → 2^(-t/S) = threshold
        → t = -S × log₂(threshold)
        """
        if threshold <= 0 or threshold >= 1:
            raise ValueError("threshold 应在 (0, 1) 区间")
        days_until = -self.stability * math.log2(threshold)
        return self.last_review + days_until * 86400

    def is_due(self, now: float | None = None,
               threshold: float = 0.7) -> bool:
        """是否到期复习"""
        now = now or time.time()
        return now >= self.optimal_review_time(threshold)


class FSRSMemoryScheduler:
    """
    基于 FSRS 思想的 Agent 记忆调度器
    """

    def __init__(self, default_threshold: float = 0.7):
        self.items: dict[str, FSRSMemoryItem] = {}
        self.default_threshold = default_threshold
        self.review_log: list[dict] = []

    def add_item(self, memory_id: str, content: str,
                 difficulty: float = 5.0,
                 importance: float = 1.0) -> FSRSMemoryItem:
        """添加新记忆"""
        now = time.time()
        item = FSRSMemoryItem(
            memory_id=memory_id,
            content=content,
            difficulty=difficulty,
            importance=importance,
            last_review=now,
        )
        self.items[memory_id] = item
        return item

    def review(self, memory_id: str, grade: int) -> dict:
        """
        执行复习

        对于 Agent 系统，grade 可以映射为:
          4: 完全匹配 / 完美检索
          3: 大部分匹配
          2: 部分匹配
          1: 模糊匹配
          0: 完全不匹配
        """
        if memory_id not in self.items:
            raise KeyError(f"未知记忆: {memory_id}")

        item = self.items[memory_id]
        grade = max(0, min(4, grade))

        # 记录复习前状态
        before_r = item.predict_retrievability()

        # 更新稳定性和难度
        old_s = item.stability
        old_d = item.difficulty

        item.stability = item.compute_stability(grade)
        item.difficulty = item.compute_difficulty(grade)
        item.last_review = time.time()
        item.access_count += 1

        # 计算新的最佳复习间隔
        next_interval = item.optimal_review_time(self.default_threshold)
        next_interval_days = (next_interval - item.last_review) / 86400

        log = {
            "memory_id": memory_id,
            "grade": grade,
            "before": {"stability": old_s, "difficulty": old_d,
                       "retrievability": before_r},
            "after": {"stability": item.stability,
                      "difficulty": item.difficulty,
                      "next_interval_days": round(next_interval_days, 1)},
        }
        self.review_log.append(log)
        return log

    def get_due_items(self, now: float | None = None,
                      limit: int = 50) -> list[FSRSMemoryItem]:
        """获取到期待复习项，按优先级排序"""
        now = now or time.time()
        candidates = []

        for item in self.items.values():
            if item.is_due(now, self.default_threshold):
                # 优先级 = 重要性 × (1 - R)  遗忘风险越高优先级越高
                item.predict_retrievability(now)
                urgency = item.importance * (1 - item.retrievability)
                candidates.append((urgency, item))

        candidates.sort(key=lambda x: -x[0])
        return [item for _, item in candidates[:limit]]

    def get_memory_stats(self) -> dict:
        """获取整体记忆状态统计"""
        items = list(self.items.values())
        n = len(items)
        if n == 0:
            return {"total": 0}

        avg_s = sum(item.stability for item in items) / n
        avg_d = sum(item.difficulty for item in items) / n
        now = time.time()
        due = sum(1 for item in items if item.is_due(now,
                  self.default_threshold))

        return {
            "total": n,
            "due": due,
            "avg_stability_days": round(avg_s, 1),
            "avg_difficulty": round(avg_d, 2),
            "total_reviews": len(self.review_log),
        }
```

---

## 6. Agent 记忆中的间隔重复应用

### 6.1 记忆检索频率的自动调度

Agent 不需要像人类那样"背诵"——间隔重复在 Agent 中的角色是**自动决定每条记忆何时应该被加载/激活**。

```
模拟流程：

  1. Agent 记录了一条新记忆（比如用户偏好）
     └→ 初始 interval = 1 hour (短间隔是因为新记忆尚不稳定)

  2. Agent 成功检索该记忆（用于推理）
     └→ 更新调度器: quality = 4 (完美匹配)
     └→ interval 更新为 3 hours

  3. Agent 再次成功检索
     └→ interval 更新为 12 hours

  4. 多次成功检索后
     └→ interval 达到 7 days
     └→ 这条记忆被认为是"长期稳固的"
```

### 6.2 重要性加权的复习优先级

在 Agent 中，不是所有记忆都同等重要。引入**重要性加权**：

```python
def compute_review_priority(memory) -> float:
    """
    计算复习优先级 = 重要性 × 遗忘紧急度

    遗忘紧急度 = max(0, 遗忘曲线阈值 - 当前 R)
    表示"这条记忆已经跌破了我设定的保留阈值"
    """
    retrieval_prob = memory.predict_retrievability()
    forget_urgency = max(0, 0.7 - retrieval_prob)  # 阈值 0.7
    return memory.importance * forget_urgency
```

### 6.3 主动回忆 (Active Recall) 模拟

人类学习中的"主动回忆"——合上书回想内容——在 Agent 中的等效操作是：

- 在检索记忆时**先遮盖部分信息**，尝试凭记忆补全
- 对于 learn 文件：定期触发"你能记住这条规则吗？"的内部核查
- 对于对话记忆：下次遇到相似上下文时，**优先尝试在记忆中检索而非重新编码**

```python
class ActiveRecallSimulator:
    """
    主动回忆模拟器

    当 Agent 遇到新的输入时，不是立即存储，
    而是先尝试从记忆中检索相关内容。
    这种"检索尝试"本身就是一种强化。
    """

    def __init__(self, scheduler):
        self.scheduler = scheduler
        self.recall_attempts = 0
        self.recall_successes = 0

    def attempt_recall(self, context: str, memory_pool: list) -> str:
        """
        模拟主动回忆：
        在给定上下文中，尝试从记忆池中 recall 相关内容

        成功 = quality ≥ 3 (SM-2 标准)
        """
        self.recall_attempts += 1
        # 模拟检索过程：简单地将 context 与记忆内容做相似度
        best_match = None
        best_score = 0

        for mem in memory_pool:
            score = self._similarity(context, mem.content)
            if score > best_score:
                best_score = score
                best_match = mem

        if best_match and best_score > 0.6:
            self.recall_successes += 1
            # 成功的主动回忆 = 复习成功
            self.scheduler.review(best_match.memory_id, quality=4)
            return best_match.content

        return ""  # 召回失败

    def _similarity(self, a: str, b: str) -> float:
        """简化的相似度度量（实际中可用 embedding 点积）"""
        set_a, set_b = set(a.split()), set(b.split())
        if not set_a or not set_b:
            return 0.0
        intersection = set_a & set_b
        return len(intersection) / max(len(set_a), len(set_b))
```

### 6.4 检索强化效应 (Retrieval Practice Effect)

**核心发现**：每一次成功的检索都会大幅强化记忆的稳定性。这与 Agent 记忆的关系：

- 频繁被检索的记忆 → 稳定性增长快 → 间隔越来越长 → 计算消耗越来越少
- 从未被检索的记忆 → 稳定性几乎不增长 → 自然衰减 → 最终被清理

这就是"用进废退"在记忆中的体现。间隔重复调度器要做的是：**确保每条值得保留的记忆都能在恰当的时间被检索到**。

---

## 7. 遗忘曲线的数学建模

### 7.1 单遗忘曲线

最基础的模型：

```
R(t) = e^(-t / s)
```

其中：
- R：记忆的可检索概率 (Retrievability)
- t：自上次复习以来的时间
- s：记忆强度 (Stability / Strength)

### 7.2 多因素遗忘模型

在 Agent 系统中，遗忘曲线的形状受多种因素影响：

```
R = exp(-t / S(m, a, f))
S(m, a, f) = S₀ × f_importance(m) × f_access(a) × f_recency(t_last)
```

其中：

```python
def memory_strength(
    base_strength: float,       # S₀：初始强度
    importance: float,           # m：记忆重要性
    access_count: int,           # a：访问次数
    hours_since_last_access: float  # t_last：距离上次访问
) -> float:
    """
    多因素记忆强度模型

    关键直觉：
    - 重要性越高，强度衰减越慢
    - 访问次数越多，基础强度越大
    - 距离上次访问越久，强度受到一定削弱
    """
    importance_factor = 0.5 + importance * 0.5  # [0.5, 1.0]
    access_factor = 1.0 + math.log1p(access_count)  # 对数增长
    recency_penalty = max(0.5, 1.0 - 0.01 * math.log1p(hours_since_last_access))

    return (base_strength
            * importance_factor
            * access_factor
            * recency_penalty)
```

### 7.3 遗忘曲线族

```
R(t)
1.0 │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
    │     ╱╲
0.8 │    ╱  ╲         高重要性 + 高频访问
    │   ╱    ╲        S = 30 天
0.6 │  ╱      ╲
    │ ╱        ╲
0.4 │╱          ╲────
    │             ╲   中等记忆
0.2 │              ╲── S = 7 天
    │                 ╲
    │                  ╲── 弱记忆 S = 1 天
0.0 └─────────────────────────▶ t (天)
    0    10    20    30    40
```

---

## 8. Python 综合示例：Agent 记忆间隔重复调度器

以下是一个完整的 Agent 记忆间隔重复系统，融合了 SM-2 和 FSRS 的核心理念：

```python
"""
AgentMemorySpacedRepetition —— 面向 AI Agent 的间隔重复强化模块

功能：
  1. 基于 SM-2 风格算法的记忆调度
  2. 重要性加权的复习优先级
  3. 多类型记忆的策略管理
  4. 批量调度优化
  5. 遗忘曲线预测
"""

import time
import math
import heapq
from enum import Enum
from dataclasses import dataclass, field
from typing import Any, Callable


class MemoryType(Enum):
    """记忆类型——不同类型的复习策略不同"""
    EPISODIC = "episodic"       # 对话历史（快速衰减）
    SEMANTIC = "semantic"       # 知识/事实（中等衰减）
    PROCEDURAL = "procedural"   # 工具/流程（慢速衰减）
    PREFERENCE = "preference"   # 用户偏好（最慢衰减）


@dataclass
class MemoryItem:
    """记忆项"""
    id: str
    content: str
    mem_type: MemoryType = MemoryType.EPISODIC
    importance: float = 1.0

    # SM-2 状态
    repetition: int = 0
    easiness: float = 2.5
    interval_days: int = 0

    # 时间戳
    created_at: float = 0.0
    last_review: float = 0.0
    next_review: float = 0.0

    # 统计
    access_count: int = 0

    def is_due(self, now: float) -> bool:
        return now >= self.next_review

    def days_overdue(self, now: float) -> float:
        return max(0, (now - self.next_review) / 86400)


# ============================================================
# 类型特定的默认策略
# ============================================================
TYPE_POLICIES: dict[MemoryType, dict] = {
    MemoryType.EPISODIC: {
        "review_threshold": 0.5,      # 低阈值 → 容易触发复习
        "interval_cap": 30,            # 最长间隔上限（天）
        "decay_multiplier": 0.7,       # 衰减加速
    },
    MemoryType.SEMANTIC: {
        "review_threshold": 0.6,
        "interval_cap": 90,
        "decay_multiplier": 1.0,
    },
    MemoryType.PROCEDURAL: {
        "review_threshold": 0.7,
        "interval_cap": 180,
        "decay_multiplier": 1.2,
    },
    MemoryType.PREFERENCE: {
        "review_threshold": 0.75,
        "interval_cap": 365,
        "decay_multiplier": 1.5,
    },
}


class AgentSpacedRepetitionScheduler:
    """
    Agent 间隔重复调度器

    核心管线：
      add() → schedule() → review() → predict()
    """

    def __init__(self):
        self._items: dict[str, MemoryItem] = {}
        self._priority_queue: list[tuple[float, float, str]] = []
        self._history: list[dict] = []

    # ---- 记忆管理 ----

    def add_memory(self, content: str,
                   mem_type: MemoryType = MemoryType.EPISODIC,
                   importance: float = 1.0,
                   memory_id: str | None = None) -> str:
        """添加新记忆"""
        mid = memory_id or f"mem:{len(self._items)}:{int(time.time())}"
        now = time.time()
        item = MemoryItem(
            id=mid,
            content=content,
            mem_type=mem_type,
            importance=importance,
            created_at=now,
            last_review=now,
            next_review=now + self._initial_interval(mem_type) * 86400,
        )
        self._items[mid] = item
        self._schedule(item)
        return mid

    def _initial_interval(self, mem_type: MemoryType) -> int:
        """初始间隔：根据记忆类型设置"""
        base = {
            MemoryType.EPISODIC: 1,
            MemoryType.SEMANTIC: 2,
            MemoryType.PROCEDURAL: 3,
            MemoryType.PREFERENCE: 5,
        }
        return base.get(mem_type, 1)

    # ---- SM-2 核心 ----

    def review(self, memory_id: str, quality: int) -> dict | None:
        """
        执行复习

        quality:
          5-4: 成功检索（匹配度好）
          3:  部分成功
          2-0: 检索失败
        """
        item = self._items.get(memory_id)
        if not item:
            return None

        policy = TYPE_POLICIES[item.mem_type]
        before_ef = item.easiness
        before_int = item.interval_days

        # --- SM-2 更新 ---
        item.easiness = self._update_easiness(item.easiness, quality)

        if quality >= 3:
            if item.repetition == 0:
                item.interval_days = 1
            elif item.repetition == 1:
                item.interval_days = 6
            else:
                item.interval_days = round(
                    item.interval_days * item.easiness
                )
            item.repetition += 1
        else:
            item.repetition = 0
            item.interval_days = 1

        # 应用类型策略的间隔上限
        item.interval_days = min(
            item.interval_days,
            policy["interval_cap"]
        )

        now = time.time()
        item.last_review = now
        item.next_review = now + item.interval_days * 86400
        item.access_count += 1

        # 记录
        entry = {
            "memory_id": memory_id,
            "time": now,
            "quality": quality,
            "before": {"ef": before_ef, "interval": before_int},
            "after": {"ef": item.easiness,
                      "interval": item.interval_days,
                      "repetition": item.repetition},
        }
        self._history.append(entry)
        self._schedule(item)
        return entry

    def _update_easiness(self, ef: float, q: int) -> float:
        """SM-2 EF 更新公式"""
        delta = 0.1 - (5 - q) * (0.08 + (5 - q) * 0.02)
        return max(1.3, ef + delta)

    # ---- 优先级队列 ----

    def _schedule(self, item: MemoryItem) -> None:
        """将记忆项加入优先级队列"""
        # 优先级 = 重要性的倒数 × 理论遗忘天数
        # 队列按此值排序，最小的先出（即最紧急的先出）
        priority = -item.importance  # 高重要性 → 优先复习
        heapq.heappush(
            self._priority_queue,
            (item.next_review, priority, item.id)
        )

    def get_due_memories(self, now: float | None = None,
                         batch_size: int = 20) -> list[MemoryItem]:
        """
        批量获取到期待复习的记忆（按紧急度排序）

        这是 Agent 的主要调用接口
        """
        now = now or time.time()
        results = []
        stale = []
        seen = set()

        while self._priority_queue and len(results) < batch_size:
            next_review, priority, mid = heapq.heappop(self._priority_queue)
            seen.add(mid)

            item = self._items.get(mid)
            if not item:
                stale.append((next_review, priority, mid))
                continue

            if now >= item.next_review:
                overdue = item.days_overdue(now)
                # 紧急度 = 逾期天数 × 重要性
                urgency = overdue * item.importance
                results.append((urgency, item))
            else:
                # 未到期，放回临时列表
                stale.append((next_review, priority, mid))

        # 按紧急度降序
        results.sort(key=lambda x: -x[0])

        # 重建队列
        for entry in stale:
            heapq.heappush(self._priority_queue, entry)

        return [item for _, item in results]

    # ---- 遗忘曲线预测 ----

    def predict_retrievability(self, memory_id: str) -> float:
        """
        预测记忆的当前可检索概率

        使用 R = e^(-t/S) 模型
        S = interval_days × (1 + repetition × 0.5) × type_multiplier
        """
        item = self._items.get(memory_id)
        if not item:
            return 0.0

        policy = TYPE_POLICIES[item.mem_type]
        now = time.time()
        t = (now - item.last_review) / 86400

        if t <= 0:
            return 1.0

        base_s = max(item.interval_days, 0.1) * (1 + item.repetition * 0.5)
        S = base_s * policy["decay_multiplier"]

        R = math.exp(-t / max(S, 0.1))
        return max(0.0, min(1.0, R))

    def get_forgetting_curve(self, memory_id: str,
                             days: int = 90) -> list[tuple[int, float]]:
        """
        生成某条记忆的遗忘曲线（用于可视化）
        返回 [(第几天, R值), ...]
        """
        item = self._items.get(memory_id)
        if not item:
            return []

        now = time.time()
        t0 = (now - item.last_review) / 86400
        policy = TYPE_POLICIES[item.mem_type]
        base_s = max(item.interval_days, 0.1) * (1 + item.repetition * 0.5)
        S = base_s * policy["decay_multiplier"]

        curve = []
        for d in range(0, days + 1):
            t = t0 + d
            if t <= 0:
                R = 1.0
            else:
                R = math.exp(-t / max(S, 0.1))
            curve.append((d, max(0.0, min(1.0, R))))
        return curve

    # ---- 统计与诊断 ----

    def get_statistics(self) -> dict:
        """获取调度器全局统计"""
        now = time.time()
        total = len(self._items)
        due = sum(1 for item in self._items.values() if item.is_due(now))

        type_dist = {}
        for item in self._items.values():
            t = item.mem_type.value
            type_dist[t] = type_dist.get(t, 0) + 1

        avg_r = [self.predict_retrievability(mid)
                 for mid in self._items]
        avg_r = sum(avg_r) / max(len(avg_r), 1)

        return {
            "total_memories": total,
            "due_now": due,
            "due_pct": round(due / max(total, 1) * 100, 1),
            "type_distribution": type_dist,
            "avg_retrievability": round(avg_r, 3),
            "total_reviews": len(self._history),
        }


# ===== 使用演示 =====
def demo():
    s = AgentSpacedRepetitionScheduler()

    # 添加不同类型的记忆
    s.add_memory("用户设置了默认温度 0.8",
                 MemoryType.PREFERENCE, importance=0.9)
    s.add_memory("Python 工具调用需要 json 格式",
                 MemoryType.PROCEDURAL, importance=0.8)
    s.add_memory("昨天讨论了关于微服务的架构",
                 MemoryType.EPISODIC, importance=0.3)
    s.add_memory("Agent 名称是 Claude，版本 4",
                 MemoryType.SEMANTIC, importance=0.7)

    print("=== 初始状态 ===")
    print(s.get_statistics())

    # 模拟一轮复习
    print("\n=== 模拟复习 ===")
    due = s.get_due_memories()
    for mem in due:
        q = 5 if mem.importance > 0.5 else 3
        result = s.review(mem.id, q)
        if result:
            print(f"  {mem.id[:30]:30s} quality={q} → "
                  f"interval={result['after']['interval']}d "
                  f"ef={result['after']['ef']:.2f}")

    print("\n=== 复习后统计 ===")
    print(s.get_statistics())

    # 查看一条记忆的遗忘曲线
    mid = "mem:0:..."
    # 查找实际的 memory_id
    for k in s._items:
        mid = k
        break
    curve = s.get_forgetting_curve(mid, days=14)
    print(f"\n=== 遗忘曲线 (前 14 天) === {mid}")
    for d, r in curve[::2]:  # 每 2 天取一点
        bar = "▓" * int(r * 20)
        print(f"  Day {d:2d}: R={r:.3f}  {bar}")


if __name__ == "__main__":
    demo()
```

---

## 9. 工程优化

### 9.1 批量调度 (Batch Scheduling)

间隔重复在单条记忆上以"秒"级精度调度没有意义。Agent 应该使用**批量窗口**：

```python
class BatchScheduler:
    """
    批量调度优化器

    核心思想：不要逐条检查每条记忆是否到期，
    而是将时间划分为批窗口，在窗口内一次性收集到期待复习项。
    """

    def __init__(self, window_hours: float = 6.0):
        self.window_hours = window_hours
        self.batch_cache: list = []

    def get_batch(self, scheduler, now: float) -> list:
        """获取当前批次中应复习的所有记忆"""
        window_start = now
        window_end = now + self.window_hours * 3600

        batch = []
        for item in scheduler._items.values():
            if window_start <= item.next_review <= window_end:
                batch.append(item)

        # 按紧急度排序
        batch.sort(
            key=lambda x: -x.importance * x.days_overdue(now)
        )
        return batch
```

### 9.2 优先级队列 (Priority Queue)

使用堆 (heap) 来管理百万级别的记忆调度：

```
                     优先级队列 (min-heap)
                     ┌──────────────────┐
  next_review 最早 ─▶│                  │◀─ 最早到期
                     │   (2025-06-01)   │
                     │   (2025-06-03)   │
                     │   (2025-06-07)   │
                     │   (2025-06-14)   │
                     │   (2025-06-30)   │
                     └──────────────────┘
```

- 插入：O(log n)
- 获取最近到期项：O(1)
- 批量获取：O(k log n)，k 为批大小

### 9.3 每记忆类型独立策略

不同类型的记忆有不同的遗忘特性，应当分别配置策略：

```python
MEMORY_TYPE_POLICIES = {
    "episodic": {
        "review_threshold": 0.5,    # 低阈值，容易触发复习
        "max_interval": 30,          # 最多 30 天
        "initial_interval": 1,       # 1 天
    },
    "semantic": {
        "review_threshold": 0.6,
        "max_interval": 90,
        "initial_interval": 2,
    },
    "procedural": {
        "review_threshold": 0.7,
        "max_interval": 365,
        "initial_interval": 3,
    },
    "preference": {
        "review_threshold": 0.75,
        "max_interval": 730,         # 2 年
        "initial_interval": 7,
    },
}
```

---

## 10. 与 6.4.2 时间衰减的关系

间隔重复和时间衰减是**互补的巩固机制**：

```
时间衰减 (Temporal Decay)         间隔重复 (Spaced Repetition)
    │                                      │
    ▼                                      ▼
被动过程                             主动过程
无条件减少记忆强度                   有选择地强化特定记忆
基于时间本身                         基于检索结果
消除噪音                             对抗信号衰减
不可配置                             高度可配置

两者共同作用：

  净记忆强度 = 初始强度 × 衰减因子 + 间隔重复增强因子

    衰减因子 = e^(-t/τ)        （τ 是衰减时间常数）
    增强因子 = Σ 每次复习的增益  （取决于质量和复习时间点）
```

在 Agent 记忆管线的流程中：

```
编码 (6.5.1)
    ↓
重激活 (6.5.2)
    ↓
模式完成 (6.5.3)
    ↓
时间衰减 → 间隔重复     ← 这两步并行/交替工作
    ↓
输出巩固结果
```

**关键区别：**

| 维度 | 时间衰减 (6.4.2) | 间隔重复 (6.5.5) |
|------|------------------|-------------------|
| 本质 | 被动遗忘过程 | 主动强化策略 |
| 控制 | 不可选择遗忘哪些 | 可选择强化哪些 |
| 参数 | τ（衰减常数） | EF, D, S, interval |
| 计算开销 | O(1) 每条记忆 | O(log n) 调度 |
| 依据 | 仅时间 | 重要性+检索质量+历史 |

---

## 11. 能力边界

间隔重复并非万能，以下场景下它的价值有限甚至为负：

### 11.1 一次性任务

对话 Agent 的**单次对话上下文**不需要间隔重复。这些记忆只在该轮对话中使用，对话结束后就可以自然衰减。

### 11.2 极短生命周期记忆

```
┌──────────────┐     ┌──────────────┐
│  临时缓存     │     │  长期知识     │
│  存活 < 1s   │     │  存活 > 1天  │
│  不需要复习   │     │  需要间隔重复  │
└──────────────┘     └──────────────┘
```

### 11.3 确定性事实

如果 Agent 的**知识库是完全确定性的**（如 SQL 数据库、向量搜索引擎），直接查询比记忆强化更可靠——间隔重复只适用于 Agent 的**工作记忆和内部状态**。

### 11.4 无法量化质量的场景

间隔重复依赖**质量评分** (quality/grade) 来更新参数。如果 Agent 无法判断一次检索是否成功（例如结果模棱两可），间隔重复会引入噪音：

```
质量评分不明确 → 参数漂移 → 复习策略劣化
```

### 11.5 零学习数据冷启动

在没有历史复习数据的情况下（全新的 Agent），间隔重复退化为规则驱动——本质上就是 Leitner 系统。此时它的优势无法充分发挥。

---

## 12. 最大挑战

### 12.1 无标签数据下的最优间隔确定

这是间隔重复在 Agent 记忆中最根本的挑战：

```
人类学习者：

  复习卡片 → 自我评分 (0-5) → 有标签数据 → 调参

AI Agent：

  检索记忆 → 如何评分？ → 无真实标签 → 难调参
```

Agent 没有"自我感觉"——它无法像人类一样判断"这次回忆花了多大力气"。

**可能的解决方案：**

1. **行为代理指标**：用检索延迟、embedding 余弦距离、下游任务成功率作为质量的代理
2. **隐式反馈**：如果 Agent 检索了一条记忆并成功用在了推理中，算作"高质量"
3. **探索性复习**：在低峰时段主动触发复习，观察记忆是否仍可检索

### 12.2 大规模下的调度效率

当记忆量达到百万级时，调度器本身成为瓶颈：

- 堆操作：O(log n)，n=10^6 时仍然可接受
- 内存占用：每条记忆 ~200 bytes → 10^6 条 ≈ 200 MB
- 更新风暴：如果大量记忆同时到期，批处理窗口需要动态调整

### 12.3 记忆类型异构性

不同类型的记忆共享同一套调度器会导致冲突：

- 对话记忆需要小时级别的调度
- 偏好记忆可能需要月级别的调度
- 在同一调度器中处理跨数量级的时间尺度需要分段策略

### 12.4 灾难性遗忘与过度巩固

```
过度巩固 (Over-consolidation)：

  一条记忆被过于频繁地复习 →
  稳定性极高 → 几乎无法更新/修正 →
  过时的信息变成了"顽固错误"

灾难性遗忘 (Catastrophic Forgetting)：

  新知识 → 强化新记忆 → 旧知识稳定性下降 →
  旧知识被自然遗忘 → Agent 行为前后不一致
```

---

## 13. 场景判断

### 13.1 哪些记忆需要间隔重复

```
需要间隔重复：
├── 跨 session 长期知识
│   ├── 用户偏好设置 (温度、风格、语气)
│   ├── 工具使用规则 (API 格式、参数约束)
│   └── 领域特定知识 (编码规范、业务逻辑)
│
├── 高频复用的上下文
│   ├── 常见问题的答案模式
│   ├── 用户常用的工作流
│   └── 频繁出现的错误及其修复
│
└── 需要强一致性的记忆
    ├── Agent 的行为准则
    ├── 安全约束和边界条件
    └── 多轮对话中的承诺
```

### 13.2 哪些记忆只需要简单衰减

```
只需简单衰减：
├── 单次对话上下文
│   ├── 用户刚才说的话
│   ├── 当前编辑的文件名
│   └── 这次对话的临时状态
│
├── 临时缓存
│   ├── 中间计算结果
│   ├── 临时 API 响应
│   └── 推理过程中的中间状态
│
└── 一次性的噪音信息
    ├── 用户偶然提到的无关事实
    ├── 过时的状态报告
    └── 低置信度的推测
```

### 13.3 决策矩阵

| 特征 | 间隔重复 | 简单衰减 |
|------|----------|----------|
| 生命周期 > 1 小时 | ✔ | ✘ |
| 跨 session 复用 | ✔ | ✘ |
| 重要性 > 0.5 | ✔ | ✘ |
| 允许小概率遗忘 | ✘ | ✔ |
| 需要精确一致性 | ✔ | ✘ |
| 被检索次数 > 3 | ✔ | ✘ |
| 存储成本敏感 | ✘ | ✔ |

---

## 14. 总结与未来方向

### 14.1 关键要点

1. **间隔重复的本质是"在遗忘的临界点复习"**——不是越频繁越好，而是时机要准
2. **记忆强度 S 是核心参数**——决定遗忘曲线的形状，决定下次复习时间
3. **Agent 的间隔重复与人类不同**——需要隐式质量信号，无法依赖自我评分
4. **重要性加权是 Agent 特有的优势**——人类很难给每条记忆标重要性，但 Agent 可以
5. **与时间衰减协作而不是对抗**——衰减清除噪音，重复强化信号

### 14.2 未来方向

- **自适应 FSRS**：根据 Agent 的行为模式自动调整遗忘曲线参数
- **上下文感知调度**：在 Agent 处于"空闲"或"低负载"时主动触发复习
- **跨 Agent 共享调度器**：多个 Agent 实例共享同一个间隔重复数据库
- **元学习最佳间隔**：让 Agent 自己学习不同记忆类型的最优复习策略

---

## 参考资料

- Ebbinghaus, H. (1885). *Memory: A Contribution to Experimental Psychology*
- Wozniak, P. (1987). *SM-2 Algorithm*. SuperMemo
- Leitner, S. (1975). *So lernt man lernen*
- Ye, J. (2019-2024). *FSRS: Free Spaced Repetition Scheduler*
- Anki Manual. *Spaced Repetition Algorithm*
- ZenBrain (2025). *A Neuroscience-Inspired 7-Layer Memory Architecture for Autonomous AI Systems*. Zenodo.
