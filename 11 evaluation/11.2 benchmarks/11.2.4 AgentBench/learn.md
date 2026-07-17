# 11.2.4 AgentBench — 多维度综合评测

## 简单介绍

AgentBench 是由清华大学（THUDM / 智谱AI）、微软研究院和 Ohio State University 联合提出的多维度 Agent 评测基准，发表于 ICLR 2024。核心论文：AgentBench: Evaluating LLMs as Agents。

与 ToolBench 聚焦工具使用、GAIA 聚焦多步推理不同，AgentBench 的核心思路是：**一个真正的 Agent 需要在多个不同的环境中都能工作**。因此，AgentBench 设计了 8 个不同的交互环境，从操作系统到电商网站，从知识图谱到策略游戏，全面评估 LLM 作为 Agent 的能力。

```
AgentBench
    │
    ├── OS (操作系统)       ─── Linux 命令行操作
    ├── DB (数据库)         ─── SQL 查询执行
    ├── KG (知识图谱)       ─── 知识图谱推理
    ├── DCG (数字卡牌游戏)   ─── 策略对战
    ├── HH (家居任务)       ─── 任务规划
    ├── WS (电商购物)       ─── 网站导航
    ├── WB (网页浏览)       ─── 通用浏览
    └── Web Search          ─── 搜索问答
```

## 基本原理

### 为什么需要多维度评测？

AgentBench 发表的背景是：当时已有的 LLM 评测主要关注问答和对话能力（如 MMLU、Hellaswag），而 Agent 能力——即 LLM 在复杂环境中自主决策、使用工具、完成多步任务的能力——几乎没有系统性的评测。

```
传统 LLM 评测关注的维度：
  ┌────────────────────────────────────────────┐
  │  知识问答 (MMLU)                            │
  │  常识推理 (HellaSwag)                       │
  │  数学计算 (GSM8K)                           │
  │  代码生成 (HumanEval)                       │
  │  ...                                        │
  │  这些评测全部是"单轮问答"模式                │
  └────────────────────────────────────────────┘

Agent 实际需要的能力维度：
  ┌────────────────────────────────────────────┐
  │  规划能力     ─── 分解任务、安排步骤          │
  │  工具使用     ─── 调用 API/命令/函数          │
  │  环境理解     ─── 理解当前状态和约束           │
  │  错误恢复     ─── 遇到错误时调整策略           │
  │  多轮交互     ─── 连续对话中保持上下文         │
  │  策略思考     ─── 在不确定中做出决策           │
  └────────────────────────────────────────────┘
```

AgentBench 的核心洞察是：**单一环境的评测不能反映 Agent 的通用能力**。一个在电商网站上表现好的 Agent，在操作系统中可能完全不会用。真正的通用 Agent 需要在多个维度上都表现良好。

## 8 个评估环境

### 1. Operating System (OS) — Linux 命令行操作

Agent 需要通过 Linux Shell 完成指定任务，例如文件操作、进程管理、文本处理等。

```
任务示例：
  "找出 /data 目录下最大的 5 个文件，按大小排序输出"
  "将 access.log 中所有状态码为 500 的行提取到 error.log"

环境特征：
  - 反馈延迟低（命令执行结果即时返回）
  - 错误明确（命令失败有明确的报错信息）
  - 需要记住文件系统状态
  - 可以无限尝试（除非有破坏性操作）
```

### 2. Database (DB) — SQL 查询执行

Agent 需要根据自然语言描述编写 SQL 查询，在真实数据库中执行并返回结果。

```
任务示例：
  "查询 2023 年销售额最高的前 10 个产品及其销量"
  "找出所有在 30 天内连续登录超过 7 天的用户"

环境特征：
  - 精确度要求高（SQL 语法必须完全正确）
  - 反馈明确（语法错误 / 结果集）
  - 需要理解数据库 Schema
  - 一次错误查询成本低（但如果写了 DELETE 则成本高）
```

### 3. Knowledge Graph (KG) — 知识图谱推理

Agent 需要基于知识图谱进行多跳推理查询，找到实体之间的复杂关系。

```
任务示例：
  "找到与"爱因斯坦"在同一所大学任教过的所有诺贝尔奖得主"
  "列出"Python"编程语言的所有直接和间接依赖框架"

环境特征：
  - 需要多步推理（一跳可能不够）
  - 查询语言特殊（如 SPARQL）
  - 路径规划是关键
  - 不同的查询策略效率差异大
```

### 4. Digital Card Game (DCG) — 策略游戏

Agent 需要在一个简化的数字卡牌游戏中做出策略决策。这是一个回合制游戏，Agent 需要理解游戏规则、当前局面、对手策略，并做出最优决策。

```
环境特征：
  - 对抗性环境（有对手）
  - 需要长期策略规划
  - 信息不完全（不知道对手手牌）
  - 运气成分（卡牌随机性）
```

### 5. House-Holding (HH) — 家居任务规划

Agent 需要在一个虚拟家居环境中规划并执行任务，如整理房间、准备餐食等。

```
任务示例：
  "厨房的地板脏了，冰箱里有食材，请准备一顿晚餐"
  "客厅很乱，请整理好所有物品"

环境特征：
  - 开放式任务（没有唯一正确答案）
  - 需要常识推理
  - 多步骤依赖（先做 A 才能做 B）
  - 时间序列约束（某些操作需要等待）
```

### 6. Web Shopping (WS) — 电商网站导航

Agent 需要在模拟的电商网站上完成购物任务，如搜索商品、比较价格、下单购买。

```
任务示例：
  "在 50 美元预算内找到评论最好的蓝牙耳机"
  "买一件适合参加正式晚宴的衬衫，价格不超过 100 美元"

环境特征：
  - 需要理解网页结构
  - 信息搜索（浏览、筛选、排序）
  - 多目标权衡（价格、质量、评论）
  - 交互成本高（每一步都需要页面跳转）
```

### 7. Web Browsing (WB) — 通用网页浏览

Agent 需要在开放互联网上浏览网页以完成信息收集任务。

```
任务示例：
  "找到 2023 年图灵奖得主的研究方向和他的主要论文"
  "比较 3 款最新旗舰手机的规格参数"

环境特征：
  - 信息完全开放（但可能过时或不准确）
  - 需要导航策略（从哪里开始，链接怎么点）
  - 信息整合能力（从多个页面提取信息）
  - 需要判断信息可信度
```

### 8. Web Search — 搜索增强问答

Agent 需要使用搜索引擎收集信息来回答复杂问题。

```
任务示例：
  "俄罗斯世界杯冠军是哪一年？那一年总决赛的比分是多少？"
  "Python 3.12 有哪些新特性？相比 3.11 性能提升了多少？"

环境特征：
  - 信息检索 + 信息整合
  - 多步搜索（一个查询不够时需调整策略）
  - 需要判断搜索结果的时效性和权威性
  - 与直接问答不同——Agent 需要主动搜索
```

## 8 个环境的维度对比

```
环境       交互模式     反馈延迟   错误成本   开放度    核心能力
───────  ──────────  ────────  ────────  ──────  ─────────────
OS       命令行        低        中        低     命令掌握
DB       SQL 查询      低        高        低     精确执行
KG       查询语言      中        低        低     多跳推理
DCG      策略决策      中        中        中     策略思考
HH       自然语言      高        低        高     常识规划
WS       网页点击      中        中        中     信息搜寻
WB       网页浏览      高        低        高     自主探索
Web Search搜索查询     中        低        高     检索整合
```

## 背景

### 评测范围太窄的问题

AgentBench 之前，LLM 评测基本只做两件事：

1. **知识评测**（MMLU、BIG-bench）：模型能记住多少知识
2. **生成评测**（HumanEval、MT-Bench）：模型能生成多好的内容

但这两类评测都无法回答关键问题：**模型能不能在复杂环境中自主完成任务？**

```
AgentBench 之前的认知误区：
  "模型在 MMLU 上得 90% → 它很聪明 → 它能当 Agent"

事实：
  MMLU 是选择题，Agent 需要的是主动决策
  两者能力几乎不相关
```

AgentBench 的实验数据证明了这一点——某些在 MMLU 上得分很高的模型（如 LLaMA-2-70B），在 Agent 任务上的表现甚至不如小模型。

## 核心矛盾

**单一环境评估无法衡量通用 Agent 能力。** 但多环境评估又带来一个新问题：每个环境的难度不同、评价标准不同、任务数量不同。如何将 8 个环境的评估结果合并成一个有意义的综合评分？

```
综合评分 = Σ(环境 i 的得分 × 环境 i 的权重)

权重设计难题：
  - 按任务数量加权？→ OS 任务多，OS 能力就被放大
  - 等权平均？→ 有些环境简单，得分虚高
  - 按难度加权？→ 难度如何定义？
```

AgentBench 采用的方案是：**每个环境独立评分 + 综合排名**。不对权重做复杂设计，而是直接对每个环境的得分进行归一化和排名，然后用平均排名作为最终的比较依据。

## 评估实现

### Runner 架构

```python
class AgentBenchRunner:
    """AgentBench 多环境评测运行器"""

    def __init__(self, config_path: str):
        self.config = self._load_config(config_path)
        # 8 个环境
        self.environments = {
            "OS": OSEmulator(config_path),
            "DB": DBEmulator(config_path),
            "KG": KGEmulator(config_path),
            "DCG": DCGEmulator(config_path),
            "HH": HHEmulator(config_path),
            "WS": WSEmulator(config_path),
            "WB": WBEmulator(config_path),
            "WebSearch": WebSearchEmulator(config_path)
        }

    def evaluate(self, agent) -> dict:
        """在 8 个环境上评测 Agent"""
        results = {}

        for env_name, env in self.environments.items():
            print(f"Evaluating {env_name}...")
            env_results = []

            for task in env.get_tasks():
                # 初始化环境
                env.reset()
                task_description = task["description"]

                # Agent 在环境中执行任务
                trajectory = []
                max_steps = task.get("max_steps", 20)

                for step in range(max_steps):
                    # Agent 观察当前状态
                    observation = env.get_observation()

                    # Agent 做出决策
                    action = agent.act(observation, task_description)

                    # 执行动作
                    feedback, done, success = env.step(action)
                    trajectory.append({
                        "step": step,
                        "observation": observation,
                        "action": action,
                        "feedback": feedback
                    })

                    if done:
                        break

                # 记录结果
                env_results.append({
                    "task_id": task["id"],
                    "success": success,
                    "trajectory": trajectory,
                    "steps_taken": step + 1
                })

            # 汇总环境结果
            results[env_name] = self._aggregate_env_results(
                env_results, env
            )

        # 计算综合排名
        results["overall_ranking"] = self._compute_overall_ranking(results)

        return results

    def _aggregate_env_results(self, results: list,
                                env) -> dict:
        """汇总单个环境的评测结果"""
        success_count = sum(1 for r in results if r["success"])
        total = len(results)

        return {
            "success_rate": success_count / total if total > 0 else 0,
            "total_tasks": total,
            "successful_tasks": success_count,
            "avg_steps": sum(r["steps_taken"] for r in results) / total
        }

    def _compute_overall_ranking(self, env_results: dict) -> dict:
        """计算综合排名"""
        # 为每个环境打分并排名
        pass
```

### 各环境的评估器

```python
class OSEmulator:
    """操作系统环境模拟器"""

    def __init__(self, config_path: str):
        self.tasks = self._load_tasks(f"{config_path}/os_tasks.json")

    def reset(self):
        """重置环境到初始状态"""
        self.container = self._create_docker_container()
        self.state = {}

    def get_observation(self) -> str:
        """获取当前 Shell 状态"""
        return self.container.get_shell_output()

    def step(self, action: str) -> tuple:
        """
        执行命令

        Returns:
            (feedback, done, success)
        """
        try:
            output = self.container.execute(action)
            done = self._check_done(output)
            success = self._check_success(output)
            return output, done, success
        except Exception as e:
            return str(e), False, False

    def get_tasks(self) -> list:
        """获取所有任务"""
        return self.tasks

    def _check_done(self, output: str) -> bool:
        """检查是否达到终止条件"""
        return False

    def _check_success(self, output: str) -> bool:
        """检查任务是否成功完成"""
        return False

    def _create_docker_container(self):
        """创建隔离的 Docker 容器"""
        import subprocess
        container_id = subprocess.check_output(
            ["docker", "run", "-d", "--rm",
             "agentbench-os:latest", "sleep", "infinity"]
        ).strip().decode()
        return DockerShell(container_id)

    def _load_tasks(self, path: str) -> list:
        """加载任务配置"""
        import json
        with open(path, "r") as f:
            return json.load(f)
```

```python
class DBEmulator:
    """数据库环境模拟器"""

    def __init__(self, config_path: str):
        self.tasks = self._load_tasks(f"{config_path}/db_tasks.json")

    def reset(self):
        """重置数据库到初始状态"""
        self.conn = self._create_db_connection()
        self._init_schema()

    def get_observation(self) -> str:
        """返回当前数据库 Schema 或查询结果"""
        return self._get_schema_info()

    def step(self, action: str) -> tuple:
        """
        执行 SQL 查询

        Returns:
            (query_result, done, success)
        """
        try:
            cursor = self.conn.cursor()
            cursor.execute(action)
            result = cursor.fetchall()
            # 检查结果是否正确
            done = True
            success = self._verify_result(action, result)
            return str(result), done, success
        except Exception as e:
            return f"SQL Error: {str(e)}", False, False

    def _init_schema(self):
        """初始化数据库 Schema 和数据"""
        pass

    def _verify_result(self, query: str, result: list) -> bool:
        """验证查询结果是否正确"""
        pass
```

## 当前 SOTA

### AgentBench 原始结果（ICLR 2024 论文）

```
模型                         综合得分    OS     DB     KG     DCG    HH     WS     WB     WebSearch
───────────────────────────  ────────  ────  ─────  ─────  ─────  ─────  ─────  ─────  ─────────
GPT-4                         0.53     0.35   0.54   0.42   0.63   0.72   0.72   0.39    0.48
Claude-2                      0.29     0.23   0.22   0.19   0.31   0.44   0.62   0.13    0.22
GPT-3.5-Turbo                 0.25     0.16   0.22   0.14   0.30   0.35   0.54   0.15    0.18
Text-Davinci-003              0.21     0.12   0.17   0.12   0.27   0.28   0.49   0.11    0.15
LLaMA-2-70B                   0.12     0.04   0.07   0.05   0.08   0.16   0.34   0.06    0.14
LLaMA-2-13B                   0.08     0.03   0.05   0.03   0.05   0.11   0.26   0.04    0.09
LLaMA-2-7B                    0.05     0.02   0.03   0.02   0.04   0.08   0.18   0.03    0.06
Vicuna-13B                    0.07     0.03   0.04   0.03   0.05   0.09   0.22   0.05    0.08
```

### 关键发现

AgentBench 的实验得出了几个重要结论：

```
结论 1: 闭源模型全面优于开源模型
  GPT-4 综合得分 0.53 vs 最佳开源 LLaMA-2-70B 仅 0.12

结论 2: 所有模型在 Web Shopping 上表现都相对较好
  因为电商场景更接近聊天模型训练数据中的对话模式

结论 3: 所有模型在 OS 上表现都很差
  命令行操作与自然语言对话差异最大

结论 4: 模型大小与 Agent 能力不成正比
  LLaMA-2-70B (0.12) 比 GPT-3.5 Turbo (0.25) 参数量大得多
  但 Agent 能力弱得多 → 训练数据和指令微调的质量更重要
```

### 近期进展（AgentBench 后续）

```
模型                         综合得分（大致范围）   发布时间
───────────────────────────  ─────────────────  ──────────
GPT-4o                        0.62              2024-05
Claude-3.5 Sonnet             0.55              2024-06
GPT-4-Turbo                   0.58              2024-04
Gemini-1.5-Pro                0.50              2024-05
LLaMA-3-70B                   0.35              2024-04
Qwen2-72B                     0.32              2024-06
DeepSeek-V2                   0.30              2024-05
```

## 与 GAIA / ToolBench / SWE-bench 的比较

```
维度            AgentBench            GAIA                ToolBench          SWE-bench
────────────  ─────────────────  ──────────────────  ─────────────────  ──────────────────
评测焦点        多维度综合能力        多步推理能力         工具使用能力        代码修改能力

设计理念        "Agent 需要能在        "Agent 需要能        "Agent 需要能        "Agent 需要能
               多个环境工作"          解决复杂推理"        使用真实 API"        修改真实代码"

环境数量        8 个                 无特定环境           API 调用环境        代码仓库环境

任务类型        混合（命令/SQL/       推理 + 信息检索      API 调用 +          代码修改 +
               网页导航/游戏等）                          参数构造             测试通过

数据来源        人工设计的任务场景      人工设计的推理题      RapidAPI            GitHub Issue

评测方式        环境交互 +             最终答案正确性        Pass Rate +         Solve Rate
               任务完成度                                   Preference Rate

覆盖面          宽（多环境）            深（推理深度）        中（工具广度）        窄（代码专业）

与其他关系      互补                  互补                 互补                互补
               （通用能力）            （推理能力）           （工具使用）          （编程能力）
```

## 实现挑战

### 1. 多环境维护成本高

AgentBench 的 8 个环境各自需要独立的模拟器实现：

- OS 环境需要 Docker 容器，每次评测启动新容器
- DB 环境需要数据库 Schema 和数据初始化
- DCG 环境需要游戏引擎实现
- WS/WB 环境需要网页模拟器

维护 8 个不同环境的成本远高于维护单一环境。

```python
# 环境维护面临的典型问题
ENVIRONMENT_ISSUES = {
    "OS": [
        "Docker 镜像版本更新",
        "命令执行的安全隔离",
        "不同 Linux 发行版的命令差异"
    ],
    "DB": [
        "数据库 Schema 版本管理",
        "数据一致性",
        "SQL 方言差异（MySQL vs PostgreSQL）"
    ],
    "DCG": [
        "游戏规则平衡性调整",
        "随机种子控制",
        "AI 对手策略难度"
    ],
    "WS/WB": [
        "网页模拟器维护",
        "页面结构变化",
        "模拟与真实的差距"
    ]
}
```

### 2. 环境间难度不均衡

8 个环境的难度天然不同，这导致：

```
问题：简单环境对总分的贡献被放大

例：
  环境 A（简单）：所有模型得分都在 0.5-0.8
  环境 B（困难）：所有模型得分都在 0.1-0.3

在环境 B 上，最好模型和最差模型的差距可能只有 0.2
在环境 A 上，差距可能有 0.5

结论：简单的环境放大了模型间的差距
      困难的环境缩小了模型间的差距
```

### 3. 评测成本与效率

在多环境评测中，每个模型需要在 8 个环境上运行，每个环境有数十到上百个任务。总评测时间可能很长。

```python
def estimate_eval_cost(model_name: str,
                       tasks_per_env: int = 50,
                       avg_steps: int = 10,
                       seconds_per_step: float = 5.0) -> dict:
    """估算评测成本"""

    n_envs = 8
    total_tasks = tasks_per_env * n_envs
    total_steps = total_tasks * avg_steps
    total_time_seconds = total_steps * seconds_per_step

    # GPT-4 API 成本估算
    cost_per_step = 0.003  # 平均每个 step 的 API 成本
    total_api_cost = total_steps * cost_per_step

    return {
        "total_tasks": total_tasks,
        "total_steps": total_steps,
        "estimated_time_hours": total_time_seconds / 3600,
        "estimated_api_cost": total_api_cost
    }
```

### 4. 环境状态的序列化与复现

评测需要确保每次运行的结果是可复现的。但多步交互环境中，每一步的状态都依赖于前一步的决策。序列化 Agent 的整个交互轨迹并复现是一项挑战。

## AgentBench 排行榜

AgentBench 的排行榜按模型排名，综合 8 个环境的得分。

```
排名    模型              综合得分   强项环境          弱项环境
────  ────────────────  ────────  ────────────────  ───────────────
 1    GPT-4o              0.62    HH, WS            OS
 2    GPT-4-Turbo         0.58    WS, HH            KG, OS
 3    Claude-3.5 Sonnet   0.55    WS, HH            KG, OS
 4    GPT-4                0.53    HH, WS            OS, KG
 5    Gemini-1.5-Pro      0.50    WS, WebSearch     OS, DCG
 6    Claude-3 Opus       0.48    HH, WS            OS, KG
 7    LLaMA-3-70B          0.35    WS, WebSearch     OS, DCG
 8    Qwen2-72B            0.32    WS, WebSearch     OS, DCG
 9    DeepSeek-V2          0.30    WS, WebSearch     OS, KG
10    Claude-2             0.29    WS, HH            KG, WB
```

注：以上数据基于 AgentBench 论文及后续更新的排行榜结果。具体数值可能因评测版本不同而略有差异。

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 评估 LLM 在多环境中的 Agent 能力 | 评估 Agent 在真实生产环境中的表现（模拟器 vs 真实） |
| 比较不同模型作为 Agent 的通用性 | 预测模型在某个特定垂直领域的 Agent 能力 |
| 发现模型在不同任务类型上的强弱项 | 评估 Agent 的长期自主运行能力（8 个环境都是短期任务） |
| 为 Agent 能力的多维度分析提供数据 | 覆盖所有 Agent 应用场景（8 个环境远远不够） |
| 追踪模型 Agent 能力的进化趋势 | 评估 Agent 与人类协作的能力（所有任务都是独立完成） |

## 扩展方向

### AgentBench v2 及后续

- **更复杂的交互环境**：增加需要长期规划（数小时/数天）的任务
- **多 Agent 协作评测**：多个 Agent 之间的合作与竞争场景
- **动态环境**：环境规则在运行中可以变化，测试 Agent 的适应能力
- **安全与对齐**：在 Agent 评测中加入安全维度，测试 Agent 是否会做有害操作
- **工具使用融合**：将 ToolBench 式的工具使用能力整合进 AgentBench 的多环境框架

### 与其他评测的融合趋势

```
未来 Agent 评测的趋势是多维度 + 多层次的融合评测：

                      ┌──────────────────────────────┐
                      │     综合 Agent 评测框架         │
                      ├──────────────────────────────┤
                      │  基础能力层                     │
                      │  (MMLU / HumanEval / ...)     │
                      ├──────────────────────────────┤
                      │  推理能力层                     │
                      │  (GAIA / ...)                 │
                      ├──────────────────────────────┤
                      │  工具使用层                     │
                      │  (ToolBench / ...)            │
                      ├──────────────────────────────┤
                      │  综合交互层                     │
                      │  (AgentBench / WebArena / ...) │
                      ├──────────────────────────────┤
                      │  代码工程层                     │
                      │  (SWE-bench / ...)            │
                      └──────────────────────────────┘
```

## 工程实践建议

1. **关注 OS 和 KG 环境**：这是所有模型最薄弱的环境，也是区分模型 Agent 能力的关键
2. **分环境报告**：综合得分容易掩盖模型在不同环境上的能力差异，一定要分环境查看
3. **环境权重透明化**：在综合排名时明确每个环境的权重，避免评分偏差
4. **测试集隔离**：AgentBench 的测试集可能被包含在模型训练数据中，需要用新构造的任务做补充验证
5. **结合其他评测**：AgentBench + GAIA + ToolBench 三件套可以覆盖 Agent 综合能力、推理能力和工具使用能力
6. **持续追踪排行榜**：Agent 能力评测正在快速发展，AgentBench 的排行榜定期更新，需要持续关注
