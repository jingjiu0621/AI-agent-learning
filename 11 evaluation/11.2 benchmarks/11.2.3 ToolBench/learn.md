# 11.2.3 ToolBench — 工具使用能力评估

## 简单介绍

ToolBench 是清华大学和面壁智能（OpenBMB）联合提出的工具使用评测基准，核心对应论文 ToolLLM: Facilitating Large Language Models to Master 16000+ Real-world APIs（ICML 2024）。ToolBench 不是一个单纯的评测数据集，而是一整套"训练-评估-规划"体系，包含三个核心组件：

```
ToolBench ─── 训练/评估数据集（16,464 个真实 API，126,686 条任务指令）
  │
ToolLLM ──── 基于 LLaMA 微调的工具使用模型
  │
DFSDT ────── Depth-First Search-based Decision Tree 规划算法
```

与其说 ToolBench 是一个评测基准，不如说它是工具学习领域的一个完整解决方案——既有数据，又有模型，还有规划算法。

## 基本原理

### 数据的构造方式

ToolBench 的数据全部来自 RapidAPI 生态——全球最大的 API 市场。RapidAPI 上有数千个分类、数万个真实 API，涵盖从天气查询到股票分析的各个领域。

```
RapidAPI 分类体系
    │
    ├── Travel ─── Amadeus, Skyscanner, Booking.com ...
    ├── Finance ── Alpha Vantage, Yahoo Finance, Twelvedata ...
    ├── Social ─── Twitter, Reddit, Instagram ...
    ├── Weather ── OpenWeatherMap, Weatherbit, AccuWeather ...
    ├── Sports ─── Sportsdata.io, ESPN, NBA ...
    └── ... 30+ 个分类
```

数据的构造流程：

```
1. 从 RapidAPI 收集 API 文档 → 16,464 个 API
2. 将 API 分类（30+ 个领域类别）
3. 基于 API 文档自动生成任务指令
4. 人工过滤 + 质量检验
```

### 三类指令

ToolBench 按照工具的复杂程度将指令分为三个层级：

```
指令类别    工具数量    示例                          复杂度
──────────  ────────  ───────────────────────────  ──────
I1           1 个      查询北京的当前天气              低
I2          2+ 个      比较北京和上海的机票价格         中
             同类别    （多个 Travel API）
I3          2+ 个      查纽约天气 → 推荐酒店 → 订机票  高
             跨类别    （Weather + Travel 交叉使用）
```

这个分层设计非常关键——它让评测可以精细地区分 Agent 在不同复杂度下的能力差异。

## 背景

在 ToolBench 出现之前，工具使用评测存在三个严重问题：

**1. 评测太少**：几乎没有专门针对工具使用的大规模评测基准。大多数研究只是在 10-20 个手工设计的 API 上做演示，缺乏统计意义。

**2. 评测太简单**：已有的工具使用评测只涉及单个工具的简单调用，没有多工具协作、没有复杂的参数组合、没有跨类别工具的规划需求。

**3. 人工标注太多**：工具使用的评测需要构造"指令 → API 调用序列"的配对数据。如果完全靠人工标注，成本极高，难以规模化。

```
ToolBench 之前的工具评测：
  ┌────────────────────────────────────────────┐
  │  10-20 个 API                               │
  │  人工构造指令和答案                           │
  │  只有单工具调用                               │
  │  ❌ 不反映真实工具使用场景                     │
  └────────────────────────────────────────────┘

ToolBench 的工具评测：
  ┌────────────────────────────────────────────┐
  │  16,464 个真实 API                          │
  │  自动生成 + 人工过滤                         │
  │  单工具 / 多工具同类别 / 多工具跨类别         │
  │  ✅ 接近真实世界的工具使用复杂度              │
  └────────────────────────────────────────────┘
```

## 核心矛盾

**工具使用的组合爆炸问题。** 当 Agent 面对 16,000+ 个 API 时，搜索空间巨大。Agent 需要：

1. 从数千个 API 中找到正确的那个
2. 理解这个 API 的输入输出格式
3. 构造正确的参数
4. 正确处理返回结果

每一步都有多种可能性，组合起来是指数级的搜索空间。传统的方法是让模型直接输出 API 调用序列（ReAct 风格），但在这么大的搜索空间下，一步错了后面全错。

```
API 选择    →    参数构造    →    结果处理
  │                │               │
 16000+ 种       平均 5+ 个       多种解析方式
 可能性          参数需要填入      可能性
  │                │               │
  └────────────────┴───────────────┘
         搜索空间 ≈ 16000 × 5 × N
```

这就是 DFSDT 想要解决的核心问题——通过搜索来弥补模型直接生成的不准确性。

## 三个核心组件

### ToolBench 数据集

```
规模统计：
  总 API 数：       16,464 个
  覆盖分类：         30+ 个
  总任务指令：       126,686 条
  I1（单工具）：     约 50,000 条
  I2（同类别多工具）：约 40,000 条
  I3（跨类别多工具）：约 36,000 条
  训练集/测试集划分： 约 9:1
```

### ToolLLM 模型

ToolLLM 是基于 LLaMA 微调的工具使用模型，核心思路是将 API 的调用方式教给模型：

```python
# ToolLLM 的训练数据格式
{
    "instruction": "查询北京的当前天气",
    "api_calls": [
        {
            "api_name": "OpenWeatherMap.current_weather",
            "parameters": {
                "city": "北京",
                "units": "metric"
            }
        }
    ],
    "response": "北京当前天气：晴，25°C，湿度 60%"
}
```

训练时将这种结构化的 API 调用序列作为模型的输出目标，使模型学会"看到用户指令 → 决定调用哪个 API → 构造参数 → 解析结果"的完整链路。

### DFSDT 规划算法

DFSDT（Depth-First Search-based Decision Tree）是 ToolLLM 的核心创新。它不是让模型一次生成完整的 API 调用序列，而是让模型在每一步生成多个候选行动，然后用搜索的方式探索最好的路径。

```
传统 ReAct 方法（一步生成）:
  ┌─────┐    ┌─────┐    ┌─────┐
  │  思考  │ → │  行动  │ → │  观察  │
  │       │    │       │    │       │
  │ 一条路 │    │ 错了  │    │ 无法  │
  │ 走到黑 │    │ 就崩  │    │ 回头  │
  └─────┘    └─────┘    └─────┘

DFSDT 方法（搜索式规划）:
  ┌──────────── 思考 ────────────┐
  │                              │
  ├── 候选行动 A ── 观察 A1 ── 继续
  │       │                       │
  │       └── 观察 A2 ── 回溯     │
  │                              │
  ├── 候选行动 B ── 观察 B1 ── 继续
  │       │                       │
  │       └── 观察 B2 ── 回溯     │
  │                              │
  └── 候选行动 C ── 观察 C1 ── 成功!
```

```python
# DFSDT 算法的核心逻辑
class DFSDTNode:
    """DFSDT 搜索树节点"""

    def __init__(self, state: str, parent: "DFSDTNode" = None,
                 action: str = None, depth: int = 0):
        self.state = state          # 当前 Agent 状态
        self.parent = parent        # 父节点（用于回溯）
        self.action = action        # 到达本节点的行动
        self.depth = depth          # 搜索深度
        self.children = []          # 子节点列表
        self.terminal = False       # 是否为终止节点

    def expand(self, model, api_pool: list, max_actions: int = 5):
        """从当前节点展开多个候选行动"""
        # 让模型生成多个候选 API 调用
        candidates = model.generate_candidates(
            self.state, api_pool, num_candidates=max_actions
        )
        for action in candidates:
            child = DFSDTNode(
                state=self.state + f"\n[Action]: {action}\n[Observation]: ...",
                parent=self,
                action=action,
                depth=self.depth + 1
            )
            self.children.append(child)
        return self.children

    def is_successful(self) -> bool:
        """判断当前状态是否完成任务"""
        return task_completed(self.state)


class DFSDTPlanner:
    """DFSDT 规划器"""

    def __init__(self, model, api_pool: list,
                 max_depth: int = 10, max_branches: int = 5):
        self.model = model
        self.api_pool = api_pool
        self.max_depth = max_depth
        self.max_branches = max_branches

    def search(self, instruction: str) -> list:
        """执行 DFSDT 搜索，返回最优行动序列"""
        root = DFSDTNode(state=instruction)

        # 深度优先搜索 + 剪枝
        best_path = None
        best_score = float("-inf")

        stack = [(root, 0)]  # (node, depth)

        while stack:
            node, depth = stack.pop()

            if depth >= self.max_depth:
                continue

            # 展开当前节点
            children = node.expand(
                self.model, self.api_pool, self.max_branches
            )

            for child in children:
                # 评估当前路径的得分
                score = self._evaluate_path(child)

                # 剪枝：得分明显低于当前最优路径的
                if score < best_score - THRESHOLD:
                    continue

                if child.is_successful():
                    # 找到成功路径
                    path = self._extract_path(child)
                    path_score = self._evaluate_path(child)
                    if path_score > best_score:
                        best_score = path_score
                        best_path = path
                else:
                    # 继续搜索
                    stack.append((child, depth + 1))

        return best_path or self._extract_path(root)

    def _evaluate_path(self, node: DFSDTNode) -> float:
        """评估从根到当前节点的路径质量"""
        # 考虑因素：API 调用正确性、参数准确度、步骤数
        return path_quality_score(node)

    def _extract_path(self, node: DFSDTNode) -> list:
        """从节点回溯提取完整行动序列"""
        path = []
        while node.parent is not None:
            path.append(node.action)
            node = node.parent
        path.reverse()
        return path
```

DFSDT 的关键优势在于，它允许 Agent 在探索过程中**回溯和纠错**——如果一条路径走不通，可以回到上一个分支点尝试其他选择。这对于工具使用场景尤为重要，因为 API 调用经常失败。

## 评估类别

ToolBench 从四个维度评估 Agent 的工具使用能力：

```
评估维度          测量内容                     测量方式
──────────────  ──────────────────────────  ─────────────────
工具选择准确率     Agent 是否选对了 API          比较 Agent 选择的
                                                API 与标准答案

参数预测正确率     Agent 填写的参数是否正确       逐个参数比对

规划质量           Agent 的 API 调用序列         评估路径的合理性
                 是否最优                       和步骤冗余度

回答质量           Agent 最终返回给用户的         评估回答是否准确、
                 回答是否正确                   完整、有用
```

## 评估指标

### Pass Rate

Pass Rate 是最核心的指标，表示 Agent 成功完成任务的比例。

```python
def calculate_pass_rate(agent_results: list,
                        ground_truth: list) -> float:
    """
    计算 Pass Rate

    Args:
        agent_results: Agent 的 API 调用序列和最终回答
        ground_truth: 标准答案（API 调用序列 + 正确回答）

    Returns:
        pass_rate: 0.0 ~ 1.0
    """
    successful = 0
    for result, truth in zip(agent_results, ground_truth):
        api_sequence_correct = compare_api_sequences(
            result["api_calls"], truth["api_calls"]
        )
        answer_correct = compare_answers(
            result["answer"], truth["answer"]
        )
        # 两个条件都满足才算通过
        if api_sequence_correct and answer_correct:
            successful += 1
    return successful / len(agent_results)
```

### Preference Rate

Preference Rate 是使用 GPT-4 作为评委，比较两个模型在相同任务上的表现好坏。

```python
class GPT4PreferenceEvaluator:
    """基于 GPT-4 的偏好评估器"""

    def __init__(self):
        self.eval_prompt = """
你是一个严格的评估员，负责比较两个 AI 助手在工
具使用任务上的表现。给定用户指令，比较两个回答。

## 用户指令
{instruction}

## 正确答案的 API 调用序列
{ground_truth_api}

## 助手 A 的回答
{answer_a}

## 助手 B 的回答
{answer_b}

## 评估标准
1. API 选择的正确性
2. 参数填写的准确性
3. 最终回答的完整性和准确度

请判断哪个助手的表现更好：
- "A": 助手 A 更好
- "B": 助手 B 更好
- "tie": 平局
"""

    def compare(self, instruction: str,
                answer_a: dict, answer_b: dict,
                ground_truth: dict) -> str:
        """比较两个模型的表现"""
        prompt = self.eval_prompt.format(
            instruction=instruction,
            ground_truth_api=json.dumps(ground_truth, indent=2),
            answer_a=json.dumps(answer_a, indent=2),
            answer_b=json.dumps(answer_b, indent=2)
        )
        # response = gpt4.invoke(prompt)
        return response.strip()
```

## 评估流程

### 完整的评估管线

```
用户指令（自然语言）
    │
    ▼
┌─────────────────────┐
│  步骤 1: API 检索     │
│  Agent 搜索可用 APIs  │
│  选择候选 API         │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  步骤 2: API 调用     │
│  Agent 填写参数      │
│  调用 API             │
│  获取返回结果         │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  步骤 3: 结果处理     │
│  Agent 解析返回数据  │
│  生成最终回答         │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  步骤 4: 结果评估     │
│  评估 Pass Rate      │
│  评估 Preference     │
│  评估各子维度         │
└─────────────────────┘
```

```python
# ToolBench 评估管线
class ToolBenchEvaluator:
    """ToolBench 评估器"""

    def __init__(self, api_pool: list, test_cases: list,
                 evaluator: str = "gpt-4"):
        self.api_pool = api_pool          # 所有可用 API
        self.test_cases = test_cases      # 测试用例
        self.evaluator = evaluator        # 评委模型

    def evaluate_agent(self, agent) -> dict:
        """评估一个 Agent 的工具使用能力"""
        results = {
            "pass_rate": {"I1": [], "I2": [], "I3": [], "overall": []},
            "tool_selection": [],
            "parameter_prediction": [],
            "plan_quality": [],
            "answer_quality": []
        }

        for case in self.test_cases:
            instruction = case["instruction"]
            difficulty = case["difficulty"]  # I1, I2, I3
            ground_truth = case["ground_truth"]

            # Agent 执行任务
            agent_output = agent.run(
                instruction=instruction,
                api_pool=self.api_pool
            )

            # 逐维度评估
            tool_ok = self._check_tool_selection(
                agent_output["api_calls"],
                ground_truth["api_calls"]
            )

            param_ok = self._check_parameter_prediction(
                agent_output["api_calls"],
                ground_truth["api_calls"]
            )

            plan_ok = self._check_plan_quality(
                agent_output["api_calls"],
                ground_truth["api_calls"]
            )

            answer_ok = self._check_answer_quality(
                agent_output["answer"],
                ground_truth["answer"]
            )

            # 记录结果
            results["tool_selection"].append(tool_ok)
            results["parameter_prediction"].append(param_ok)
            results["plan_quality"].append(plan_ok)
            results["answer_quality"].append(answer_ok)

            passed = tool_ok and param_ok and answer_ok
            results["pass_rate"][difficulty].append(passed)

        # 汇总
        return self._aggregate_results(results)

    def _check_tool_selection(self, agent_calls: list,
                              gt_calls: list) -> bool:
        """检查工具选择是否准确"""
        agent_apis = {call["api_name"] for call in agent_calls}
        gt_apis = {call["api_name"] for call in gt_calls}
        return agent_apis == gt_apis

    def _check_parameter_prediction(self, agent_calls: list,
                                    gt_calls: list) -> bool:
        """检查参数预测是否正确"""
        for agent_call, gt_call in zip(agent_calls, gt_calls):
            if agent_call["api_name"] != gt_call["api_name"]:
                continue
            for key, value in gt_call["parameters"].items():
                if agent_call["parameters"].get(key) != value:
                    return False
        return True

    def _check_plan_quality(self, agent_calls: list,
                            gt_calls: list) -> float:
        """评估规划质量（路径效率）"""
        if not gt_calls:
            return 1.0 if not agent_calls else 0.0

        # 路径越短越好
        optimal_steps = len(gt_calls)
        actual_steps = len(agent_calls)

        if actual_steps > optimal_steps * 1.5:
            return 0.5  # 步骤过多
        elif actual_steps < optimal_steps * 0.5:
            return 0.7  # 可能遗漏了步骤
        else:
            return 1.0

    def _check_answer_quality(self, agent_answer: str,
                              gt_answer: str) -> float:
        """评估回答质量"""
        # 使用 LLM-as-Judge
        return self._llm_judge(agent_answer, gt_answer)

    def _aggregate_results(self, raw: dict) -> dict:
        """汇总结果"""
        aggregated = {}
        for difficulty, scores in raw["pass_rate"].items():
            if scores:
                aggregated[f"pass_rate_{difficulty}"] = (
                    sum(scores) / len(scores)
                )

        for dim in ["tool_selection", "parameter_prediction",
                     "plan_quality", "answer_quality"]:
            scores = raw[dim]
            if scores:
                aggregated[dim] = sum(scores) / len(scores)

        return aggregated
```

## API 检索与选择

ToolBench 场景中 Agent 需要从 16,000+ API 中找到正确的那个。这需要一个高效的检索机制：

```python
class APIRetriever:
    """从 API 池中检索相关工具"""

    def __init__(self, api_pool: list):
        self.api_pool = api_pool
        # 构建 API 索引（名称 + 描述 + 参数名）
        self.index = self._build_index(api_pool)

    def _build_index(self, api_pool: list) -> dict:
        """构建 API 检索索引"""
        index = {}
        for api in api_pool:
            # 索引字段：API 名称、分类、描述、参数名
            text = f"{api['name']} {api['category']} "
            text += f"{api['description']} "
            text += " ".join(api.get("parameters", {}).keys())
            index[api["id"]] = {
                "text": text.lower(),
                "api": api
            }
        return index

    def retrieve(self, instruction: str, top_k: int = 10) -> list:
        """根据用户指令检索最相关的 API"""
        instruction_lower = instruction.lower()

        # 基于关键词匹配的简单检索
        scores = {}
        for api_id, entry in self.index.items():
            text = entry["text"]
            # 计算关键词重叠度
            common_words = set(instruction_lower.split()) & set(text.split())
            scores[api_id] = len(common_words) / max(len(text.split()), 1)

        # 排序取 Top-K
        ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        return [self.index[api_id]["api"]
                for api_id, _ in ranked[:top_k]]
```

## 当前 SOTA

### ToolBench 原始结果（ToolLLM 论文）

```
模型                          Pass Rate (Overall)    Preference Rate vs ChatGPT
───────────────────────────  ────────────────────  ─────────────────────────
ToolLLM-13B (LLaMA + DFSDT)      ~55%                      ~57%
ToolLLM-7B (LLaMA + DFSDT)       ~52%                      ~53%
GPT-4 (zero-shot)                ~45%                      ~48%
ChatGPT (GPT-3.5-turbo)          ~26%                     基准线
Text-Davinci-003                 ~16%                      ~32%
LLaMA-7B (base, zero-shot)       ~5%                       ~15%
```

按指令难度分类的结果：

```
模型                  I1 (单工具)    I2 (同类别多工具)    I3 (跨类别多工具)
───────────────────  ───────────  ─────────────────  ─────────────────
ToolLLM-13B            ~62%            ~52%               ~42%
GPT-4                  ~58%            ~42%               ~30%
ChatGPT                ~38%            ~24%               ~12%
```

### StableToolBench 结果

StableToolBench 是 ToolBench 的改进版本，解决了原始 ToolBench 中 API 调用不稳定（API 可能下线、返回结果变化等）的问题。StableToolBench 缓存了 API 调用的返回结果，确保评估的可复现性。

```
模型                          Pass Rate    Preference Rate
───────────────────────────  ───────────  ────────────────
GPT-4o                         ~52%          ~55%
GPT-4-Turbo                    ~48%          ~52%
Claude-3-Opus                  ~46%          ~50%
ToolLLM-13B + DFSDT            ~44%          ~48%
GPT-3.5-Turbo                  ~28%          ~31%
```

## 与 GAIA / SWE-bench 的比较

```
维度            ToolBench              GAIA                 SWE-bench
────────────  ────────────────────  ───────────────────  ───────────────────
评测焦点        工具使用能力            多步推理能力          代码修改能力

任务类型        API 调用 + 参数构造     多步推理 + 信息检索    GitHub Issue 修复

工具数量        16,000+ 个 API         无/少量工具           代码库文件

评估指标        Pass Rate,             任务成功率            Solve Rate
               Preference Rate

任务复杂度      1-3 步 API 调用        1-10+ 步推理          文件修改 + 测试

数据来源        RapidAPI 真实 API     人工设计的推理题       GitHub 真实 Issue

与语言的关系     无关                   相关                 强相关（代码）

对训练数据      低（API 是新的）        中（知识依赖）        低（代码库是新的）
的污染风险
```

## 实现挑战

### 1. 真实 API 调用不可靠

真实 API 经常变化——可能下线、限流、返回格式改变。这导致评估结果不稳定，同一次评测两次运行可能得到不同结果。

```python
# StableToolBench 的解决方案：缓存 API 响应
class CachedAPIExecutor:
    """缓存 API 调用结果，保证评估可复现"""

    def __init__(self, api_pool: list, cache_path: str = "api_cache.json"):
        self.api_pool = {api["id"]: api for api in api_pool}
        self.cache = self._load_cache(cache_path)

    def execute(self, api_id: str, parameters: dict) -> dict:
        """执行 API 调用（优先使用缓存）"""
        cache_key = f"{api_id}:{json.dumps(parameters, sort_keys=True)}"

        if cache_key in self.cache:
            return self.cache[cache_key]

        # 真实调用 API
        api = self.api_pool[api_id]
        result = self._real_api_call(api, parameters)

        # 缓存结果
        self.cache[cache_key] = result
        self._save_cache()

        return result
```

### 2. 评估成本高

使用 GPT-4 作为评委评估 Preference Rate 成本很高。对于 1,000 个测试用例，每个用例需要调用 GPT-4 做一次比较，成本在 $10-50 级别。

### 3. 指令多样性不足

虽然 TaskBench 有 126,686 条指令，但它们都是基于 RapidAPI 的 API 文档自动生成的。指令的模式相对固定，可能无法覆盖真实用户的所有使用方式。

### 4. 静态工具 vs 动态环境

ToolBench 中的 API 是预先定义好的，Agent 在使用前就已经知道所有 API 的规格。但真实场景中，API 是动态变化的——新的 API 不断出现，老的 API 被废弃。

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 评估工具选择准确率 | 评估 Agent 理解 API 文档的能力（API 是预定义的） |
| 评估多工具编排能力 | 反映真实 API 调用的全部复杂性（网络延迟、错误处理等） |
| 比较不同模型的工具使用能力 | 评估 Agent 发现和探索新 API 的能力 |
| 细粒度区分 I1/I2/I3 难度 | 覆盖工具使用中的长尾问题（如认证、限流、分页） |
| 提供可复现的评测环境（StableToolBench） | 评估工具使用的用户体验（API 调用 vs 真实用户感受） |

## 扩展方向

### ToolBench v2 / 后续版本

- **动态工具创建**：Agent 不仅使用工具，还能根据需求创建新工具（组合现有 API、编写新函数）
- **工具发现**：Agent 在开放的互联网上自主发现和了解新工具，而不是使用预定义的 API 池
- **多模态工具**：支持图像处理、音频处理等多模态 API
- **工具链自动化**：Agent 自动构建多步工具调用流水线

### 与其他评测的融合

- **ToolBench + GAIA**：在需要工具使用的推理任务上评测 Agent
- **ToolBench + SWE-bench**：评估 Agent 使用工具修改代码的能力
- **ToolBench + WebArena**：在真实网页环境中评测工具使用能力

## 工程实践建议

1. **使用 StableToolBench 而非原始 ToolBench**：StableToolBench 缓存了 API 响应，评估结果更稳定可复现
2. **分层报告**：分别报告 I1/I2/I3 的 Pass Rate，不要只看总分
3. **Preference Rate 辅助**：Pass Rate 不足以区分模型间的精细差异，结合 Preference Rate 更全面
4. **关注工具选择错误 vs 参数错误**：两种错误的根因不同，需要分别分析和优化
5. **定期更新 API 池**：RapidAPI 上的 API 在变化，定期更新评测集以保持时效性
