# 11.2.1 GAIA — 通用 AI Agent 评估基准

## 简单介绍

GAIA（General AI Assistants）是由 Meta FAIR 于 2023 年提出的 Agent 基准测试。如果用一个比喻来理解 GAIA：传统的 NLP 基准测试的是"学生能答对多少题"，而 GAIA 测试的是"助理能办好多少事"。它不关心模型是否"知道"某个知识点，而是关心模型是否能像人类助理一样，通过推理、搜索、使用工具来**完成一个真实世界任务**。

GAIA 的 466 个问题对普通人来说相对简单（人类基线 92%），但对 AI 系统极具挑战性——最初 GPT-4 也只能做到约 15%。到 2026 年，最优系统（如 MiroThinker-1.7）在验证集上达到了 82.7%，但仍然离人类水平有差距。

## 基本原理

### 数据集的构成

GAIA 数据集包含 466 个精心设计的问题，这些问题具有以下特征：

```
┌─────────────────────────────────────────────────────────┐
│                    GAIA 数据集概览                        │
│                                                         │
│  总问题数: 466  (训练 61 + 验证 165 + 测试 240)          │
│  难度分布: Level 1 ~165, Level 2 ~165, Level 3 ~136     │
│  人类基线: 92% (众包测试，不限时间)                      │
│  任务类型: 信息检索、数据分析、文件处理、代码生成等        │
│  输出格式: 简短答案（数字、字符串、列表、文件）           │
│  工具需求: 网络搜索、代码执行、文件解析、多模态理解       │
│  多项模态: 文本、图片、音频、视频、电子表格、PDF         │
└─────────────────────────────────────────────────────────┘
```

注意 GAIA 的验证和测试集的答案是保密的（held-out），只在 HuggingFace 平台上通过特殊 API 进行评分，这有效防止了数据污染。

### GAIA-Val-165 和 GAIA-Text-103

由于完整测试集的答案不公开，社区常用两个子集进行评估：

- **GAIA-Val-165**: 165 个验证集问题（含多模态任务），是目前社区最常用的评估子集
- **GAIA-Text-103**: GAIA-Val-165 中去掉图片/音频/视频任务后的纯文本子集，共 103 题

## 三个难度级别

GAIA 的任务按所需的推理步数分为三个级别：

### Level 1: 基础任务（2-5 步）

```
示例: "2022年诺贝尔物理学奖得主的出生年份是多少？"
所需步骤:
  1. 搜索"2022 Nobel Prize Physics winner"
  2. 提取获奖者姓名（Alain Aspect, John Clauser, Anton Zeilinger）
  3. 搜索其中一位的出生年份
  4. 返回答案
```

**特点**: 单一信息源，基本工具使用，少量推理

### Level 2: 中级任务（5-10 步）

```
示例: "下载附件中的 Excel 表格，计算第三季度销售额大于
      平均值的产品数量，并以 JSON 格式输出结果。"
所需步骤:
  1. 读取附件 Excel 文件
  2. 解析表格结构，定位销售额列
  3. 筛选"第三季度"数据
  4. 计算平均值
  5. 逐行比较销售额与平均值
  6. 计数器累加
  7. 格式化为 JSON 输出
```

**特点**: 多信息源组合，工具协同（文件处理+代码执行），中等推理

### Level 3: 高级任务（10+ 步，最高可达 50 步）

```
示例: "从这篇 PDF 论文中提取所有实验数据，将其与作者
      提供的开源数据集进行比较，找出不一致之处并生成报告。"
所需步骤: (简化)
  1. 下载 PDF → 2. OCR/解析 → 3. 提取表格 → 4. 搜索数据集
  5. 下载数据集 → 6. 写代码对比 → 7. 分析差异
  8. 判断是否显著 → 9. 格式化报告 → 10. 输出
```

**特点**: 长链推理，深度工具编排，外部知识整合，需要自我纠错

```
Level 1 (165题)    ┌─────┬─────┬─────┬─────┐
                   │ 搜索 │ 提取 │ 推理 │ 输出 │
                   └─────┴─────┴─────┴─────┘
                     2-5 步，单一工具

Level 2 (165题)    ┌─────┬─────┬─────┬─────┬─────┬─────┐
                   │ 解析 │ 代码 │ 计算 │ 搜索 │ 整合 │ 输出 │
                   └─────┴─────┴─────┴─────┴─────┴─────┘
                     5-10 步，多工具协同

Level 3 (136题)    ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
                   │多步推理│多轮搜索│代码│分析│整合│纠错│输出│
                   └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
                     10+ 步，复杂工具编排
```

## 任务类型

GAIA 的任务可以按照所需能力进行分类：

### 信息检索类 (约占 40%)

需要从互联网或给定文档中查找、筛选和整合信息。

```python
gaia_retrieval_examples = [
    "某部电影的导演还导演过哪些获得奥斯卡最佳影片的电影？",
    "2023 年全球 GDP 排名前 10 的国家中，有几个是欧洲国家？",
    "某篇学术论文中提到的实验方法最早是谁提出的？"
]
```

### 数据分析类 (约占 25%)

需要对给定的数据（表格、CSV、数据库）进行计算和分析。

### 文件处理类 (约占 15%)

需要读取和处理附件中的文件（PDF、Excel、图片、音视频）。

### 代码生成类 (约占 10%)

需要编写代码来完成特定的计算或数据处理任务。

### 多模态理解类 (约占 10%)

需要理解图片、图表、音频或视频中的信息。

## 评估指标

### 严格成功率 (Strict Accuracy)

```
严格成功率 = Agent 输出与标准答案完全匹配的数量 / 总任务数

匹配规则:
  - 字符串: 精确匹配（忽略大小写和标点）
  - 数字: 数值相等（允许 ±0.1% 误差）
  - 列表: 元素相同（顺序无关）
  - 文件: MD5 哈希匹配
```

### 宽松成功率 (Relaxed Accuracy)

```
宽松成功率 = 人类评估认为"基本正确"的数量 / 总任务数

宽松条件:
  - 同义但不同的表述
  - 数值的近似匹配
  - 部分正确的答案（得分 0-1 之间）
```

### 评估代码示例

```python
class GAIAScorer:
    """GAIA 任务的评分器"""

    def __init__(self, strict: bool = True):
        self.strict = strict

    def score(self, agent_answer: str,
              expected_answer: str, answer_type: str) -> float:
        """计算单个任务的得分"""

        if answer_type == "string":
            return self._score_string(agent_answer, expected_answer)
        elif answer_type == "number":
            return self._score_number(agent_answer, expected_answer)
        elif answer_type == "list":
            return self._score_list(agent_answer, expected_answer)
        elif answer_type == "file":
            return self._score_file(agent_answer, expected_answer)
        else:
            return 0.0

    def _score_string(self, agent: str, expected: str) -> float:
        """字符串匹配"""
        agent = agent.strip().lower()
        expected = expected.strip().lower()
        return 1.0 if agent == expected else 0.0

    def _score_number(self, agent: str, expected: str) -> float:
        """数值匹配，允许微小误差"""
        try:
            agent_val = float(agent.strip())
            expected_val = float(expected.strip())
            if abs(expected_val) < 1e-10:
                return 1.0 if abs(agent_val) < 1e-10 else 0.0
            relative_error = abs(agent_val - expected_val) / abs(expected_val)
            return 1.0 if relative_error < 0.001 else 0.0
        except ValueError:
            return 0.0

    def _score_list(self, agent: str, expected: str) -> float:
        """列表匹配，顺序无关"""
        import ast
        try:
            agent_list = set(ast.literal_eval(agent.strip()))
            expected_list = set(ast.literal_eval(expected.strip()))
            if len(expected_list) == 0:
                return 1.0 if len(agent_list) == 0 else 0.0
            intersection = agent_list & expected_list
            return len(intersection) / len(expected_list)
        except Exception:
            return 0.0

    def _score_file(self, agent_path: str, expected_hash: str) -> float:
        """文件匹配（MD5 哈希）"""
        import hashlib
        try:
            with open(agent_path, 'rb') as f:
                content = f.read()
            agent_hash = hashlib.md5(content).hexdigest()
            return 1.0 if agent_hash == expected_hash.strip() else 0.0
        except Exception:
            return 0.0
```

## 背景：为什么需要 GAIA

在 GAIA 之前，基准测试存在一个根本性问题：**它们测试的是"模型能力"而非"Agent 能力"**。

```
传统基准测试                     GAIA
───────────────                 ────
"模型能答对吗？"                 "Agent 能办成事吗？"
单一知识问答                     多步任务完成
不需要工具                       必须使用工具
封闭环境                         开放环境
单轮交互                         多步推理链
答案在训练数据中                  答案不在训练数据中
```

GAIA 的论文标题《GAIA: a benchmark for General AI Assistants》直接点出了定位——它要测试的是"通用 AI 助理"的能力，而不仅仅是"语言模型"的能力。论文中最令人印象深刻的数字是：人类得分 92%，当时最好的 GPT-4 + 工具也只有 15%。这个巨大的差距（77 个百分点）清楚地表明：**模型知道的多，不等于能办成事**。

这个发现直接推动了整个 Agent 领域的研究方向——大家都在思考如何缩小这个"知道-做到"的差距。

## 核心矛盾

**真实世界任务的开放性 vs 可重复评估的封闭性**

```
开放的真实任务                             封闭的基准评估
─────────────────                         ────────────────
问题可能有无穷多种表达                     问题格式标准化
答案可能不唯一                            有且仅有一个正确答案
工具和环境随时变化                         工具和环境固定
成功标准因人而异                          成功标准统一制定
```

GAIA 的设计选择了"封闭"这一侧来保证评估的可重复性。但这也意味着：一个在 GAIA 上得分高的 Agent，未必能在完全开放的场景中同样出色。

另一个重要的矛盾是 **"可评分" vs "有价值"**：GAIA 只能选择那些答案可以自动评分的问题，这天然排除了很多开放式的、创造性的任务。GAIA 能评估的是"助理能否完成指定任务"，而不是"助理能否主动发现并解决未明确的问题"。

## Code Examples

### GAIA 任务运行器

```python
import json
import time
from typing import Optional


class GAIAEnvironment:
    """GAIA 评估环境"""

    def __init__(self, data_dir: str):
        self.data_dir = data_dir
        self.tasks = self._load_tasks()

    def _load_tasks(self) -> list:
        """加载 GAIA 任务"""
        import glob
        tasks = []
        for task_file in sorted(
            glob.glob(f"{self.data_dir}/**/*.json", recursive=True)
        ):
            with open(task_file, 'r') as f:
                task_data = json.load(f)
            tasks.append(task_data)
        return tasks

    def get_task(self, task_id: str) -> Optional[dict]:
        for task in self.tasks:
            if task["task_id"] == task_id:
                return task
        return None

    def get_tasks_by_level(self, level: int) -> list:
        return [t for t in self.tasks if t.get("level") == level]

    def get_file_path(self, task_id: str, filename: str) -> str:
        """获取任务附件的路径"""
        import os
        task_dir = os.path.join(self.data_dir, task_id)
        return os.path.join(task_dir, filename)


class GAIAEvaluator:
    """GAIA 评估器"""

    def __init__(self, agent, environment: GAIAEnvironment,
                 scorer: GAIAScorer):
        self.agent = agent
        self.env = environment
        self.scorer = scorer
        self.results = []

    def evaluate_level(self, level: int, num_samples: int = None) -> dict:
        """评估指定难度的所有（或部分）任务"""

        tasks = self.env.get_tasks_by_level(level)
        if num_samples:
            import random
            tasks = random.sample(tasks, min(num_samples, len(tasks)))

        level_results = []
        for task in tasks:
            result = self._evaluate_single(task)
            level_results.append(result)
            self.results.append(result)

        scores = [r["score"] for r in level_results]
        avg = sum(scores) / len(scores) if scores else 0.0

        return {
            "level": level,
            "num_tasks": len(level_results),
            "avg_score": round(avg, 4),
            "passed": sum(1 for s in scores if s >= 1.0),
            "details": level_results
        }

    def _evaluate_single(self, task: dict) -> dict:
        """评估单个 GAIA 任务"""

        task_id = task["task_id"]
        question = task["Question"]
        answer_type = task.get("answer_type", "string")
        expected_answer = task.get("answer", "")

        start_time = time.time()

        try:
            # 让 Agent 执行任务
            agent_output = self.agent.run(question)

            # 评分
            score = self.scorer.score(
                str(agent_output), expected_answer, answer_type
            )
            passed = score >= 1.0
            error = None

        except Exception as e:
            agent_output = ""
            score = 0.0
            passed = False
            error = str(e)

        execution_time = time.time() - start_time

        return {
            "task_id": task_id,
            "level": task.get("level"),
            "question": question[:80],
            "expected": expected_answer,
            "agent_output": str(agent_output)[:100],
            "score": score,
            "passed": passed,
            "execution_time": round(execution_time, 2),
            "error": error
        }
```

### Agent 接入示例

```python
import requests


class WebSearchTool:
    """网络搜索工具"""

    def __init__(self, api_key: str):
        self.api_key = api_key

    def search(self, query: str, num_results: int = 5) -> list:
        """执行网络搜索"""
        response = requests.get(
            "https://api.serper.dev/search",
            headers={"X-API-KEY": self.api_key},
            json={"q": query, "num": num_results}
        )
        results = response.json().get("organic", [])
        return [r["snippet"] for r in results[:num_results]]

    def visit_page(self, url: str) -> str:
        """访问网页获取内容"""
        response = requests.get(url, timeout=10)
        return response.text[:5000]


class PythonExecutor:
    """Python 代码执行工具（沙箱）"""

    def execute(self, code: str) -> str:
        """在隔离环境中执行 Python 代码"""
        sanitized_code = self._sanitize(code)
        local_vars = {}
        try:
            exec(sanitized_code, {"__builtins__": __builtins__}, local_vars)
            return str(local_vars.get("result", "No result variable"))
        except Exception as e:
            return f"Error: {str(e)}"

    def _sanitize(self, code: str) -> str:
        import re
        dangerous = ["__import__", "open(", "os.", "subprocess"]
        for item in dangerous:
            code = code.replace(item, f"_{item}")
        return code


class SimpleGAIAgent:
    """简化的 GAIA Agent"""

    def __init__(self, llm, tools: dict):
        self.llm = llm
        self.tools = tools

    def run(self, task: str) -> str:
        """执行任务"""
        prompt = f"""
你是一个 AI 助理，需要完成以下任务。

任务: {task}

可用工具:
{list(self.tools.keys())}

请一步步思考并完成任务。
输出最终答案（简短格式）。
"""
        # 实际使用时需要调用 LLM + 工具循环
        return self.llm.generate(prompt)
```

## 当前 State-of-the-Art

根据 2026 年的公开数据，GAIA 基准上的领先成绩如下：

### GAIA-Val-165 排行榜 (2026)

| 排名 | 模型/系统 | 分数 | 发布时间 | 备注 |
|------|----------|------|---------|------|
| 1 | MiroThinker-H1 (proprietary) | 88.4% | 2026-03 | 商业系统 |
| 2 | MiroThinker-1.7 (235B) | 82.7% | 2026-03 | 开源 |
| 3 | MiroThinker-v1.5 (235B) | 80.8% | 2026-01 | 开源 |
| 4 | H2O.ai h2oGPTe | ~65% | 2025 | 商业系统 |
| 5 | GPT-4 + tools (早期基线) | ~15% | 2023 | 原始论文基线 |
| 6 | Human baseline | ~92% | 2023 | 众包测试 |

### 发展趋势

```
100% ┤                                 Human (92%)
 90% ┤                                ●
 80% ┤                     ● MiroThinker-1.7 (82.7%)
 70% ┤              ● H2O.ai (65%)
 60% ┤
 50% ┤
 40% ┤
 30% ┤
 20% ┤   ● GPT-4+Tools (15%)
 10% ┤
    └──────────────────────────────────────────
       2023         2024         2025    2026

从 15% 到 82.7%，Agent 系统在 GAIA 上的表现取得了巨大进步。
但距离人类基线（92%）仍有约 10 个百分点的差距。
```

### 关键进展

- **2023 年底**: GAIA 论文发布，GPT-4 + 工具仅 15%，震惊学界
- **2024 年**: AutoGPT、BabyAGI 等早期 Agent 开始在 GAIA 上验证
- **2025 年**: H2O.ai h2oGPTe 达到 65%，首次达到"C 级"水平
- **2026 年初**: MiroThinker 系列达到 80%+，但人类基线仍领先约 10%

## 实现挑战

### 1. 任务多样性 (Task Diversity)

GAIA 的 466 个问题涉及极其广泛的知识领域和工具类型。设计一个对所有任务都有效的通用 Agent 架构并不容易。

### 2. 答案验证 (Answer Validation)

由于 GAIA 的答案格式多样（字符串、数字、列表、文件），验证逻辑需要为每种类型定制。多模态任务的验证更是复杂。

```python
class GAIAAnswerValidator:
    """GAIA 答案验证器"""

    def __init__(self, validator_api_url: str):
        self.validator_api_url = validator_api_url

    def submit_validation(self, task_id: str,
                          agent_answer: str) -> dict:
        """通过 HuggingFace API 提交答案验证"""
        response = requests.post(
            f"{self.validator_api_url}/validate",
            json={
                "task_id": task_id,
                "answer": agent_answer
            }
        )
        return response.json()

    def batch_validate(self, results: list) -> list:
        """批量验证"""
        validated = []
        for result in results:
            v_result = self.submit_validation(
                result["task_id"],
                result["agent_answer"]
            )
            validated.append({
                **result,
                "validated_score": v_result.get("score", 0)
            })
        return validated
```

### 3. 人类基线校准 (Human Baseline)

GAIA 的人类基线是 92%，但这个数字是"不限时间、可以使用任何工具、多人协作"的结果。单个人类在有限时间内使用有限工具的得分可能低于这个数字。

### 4. 多模态处理成本

GAIA 包含图片、音频和视频任务。处理这些多模态输入需要额外的模型调用（如 GPT-4o 进行模态转换），增加了评估成本。

```python
class MultimodalProcessor:
    """多模态输入处理器"""

    def process_image(self, image_path: str, query: str) -> str:
        """使用多模态模型处理图片"""
        prompt = f"请描述这张图片中与以下问题相关的信息: {query}"
        return self.vlm_call(prompt, image_path)

    def process_audio(self, audio_path: str, query: str) -> str:
        """先语音转文字，再回答"""
        transcript = self.speech_to_text(audio_path)
        prompt = f"以下是一段音频的文字转录，请回答: {query}\n\n{transcript}"
        return self.llm_call(prompt)

    def vlm_call(self, prompt: str, image_path: str) -> str:
        """调用视觉语言模型"""
        _ = image_path
        return f"[VLM response to: {prompt[:50]}]"

    def speech_to_text(self, audio_path: str) -> str:
        """语音转文字"""
        _ = audio_path
        return "[transcribed text]"

    def llm_call(self, prompt: str) -> str:
        """调用语言模型"""
        _ = prompt
        return "[text response]"
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 评估通用推理+工具使用能力 | 评估深度编程能力（那需要 SWE-bench） |
| 测量多步推理链的质量 | 测量实时交互的质量 |
| 提供标准化的 Agent 能力对比 | 覆盖所有真实场景（466 题有限） |
| 追踪 Agent 技术的年度进展 | 预测在新任务类型上的表现 |
| 发现工具使用能力短板 | 衡量 Agent 的创造性和主动性 |

## 与 SWE-bench / ToolBench 的比较

### 三维能力对比

```
                     GAIA (通用型)
                     /            \
                    /              \
                   /                \
                  /                  \
          SWE-bench ──────────────── ToolBench
          (代码修改)                   (API 调用)
```

| 维度 | GAIA | SWE-bench | ToolBench |
|------|------|-----------|-----------|
| 核心能力 | 通用推理+工具 | 代码调试+修改 | API 调用+组合 |
| 任务数量 | 466 | 2294 | 3468 |
| 任务来源 | 专家手工设计 | 真实 GitHub Issue | 真实 API 文档 |
| 评估方式 | 答案匹配 | 测试用例通过 | 执行结果验证 |
| 环境依赖 | 搜索+执行环境 | Docker 容器 | API 模拟器 |
| 人类基线 | 92% | ~90% | -- |
| 当前 SOTA | 82.7% | 76.8% | ~70%+ |
| 贴近真实 | 中（问题设计但场景真实） | 高（真实 Issue） | 中（模拟 API） |
| 评估成本 | 中 | 高（Docker 开销） | 中 |
| 数据污染风险 | 低（答案未公开） | 中（Issue 在 GitHub） | 中 |

**GAIA 的优势**: 能力维度最全面——一个好的 GAIA Agent 需要同时具备搜索、推理、代码执行、文件处理、多模态理解等多种能力。

**SWE-bench 的优势**: 最贴近真实软件工程场景——直接使用真实 GitHub Issue 和 PR，评估的是"在完整的代码库中修复 Bug"的能力。

**ToolBench 的优势**: 工具使用广度最大——覆盖超过 3000 个真实 API，评估 Agent 理解和调用陌生 API 的能力。

三者互为补充：GAIA 看"通用性"，SWE-bench 看"代码能力"，ToolBench 看"工具使用能力"。全面的 Agent 评估应该三管齐下。
