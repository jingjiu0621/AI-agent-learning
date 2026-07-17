# 11.2.2 SWE-bench — 软件工程任务评估

## 简单介绍

SWE-bench 是评估 LLM/Agent 解决真实软件工程问题能力的基准测试。它的核心理念非常直接：**从真实 GitHub 仓库中提取一个 Issue（Bug 报告/功能请求），让 Agent 读懂 Issue 描述，找到需要修改的代码，做出正确的修改，然后通过项目的测试套件验证修改是否正确。**

SWE-bench 由普林斯顿大学的研究团队于 2023 年提出，是第一个将"修改真实代码库"作为评估任务的基准。它回答了 Agent 领域的一个关键问题：**AI 能不能真的帮助程序员修 Bug？**

## 基本原理

### 什么是 SWE-bench 实例

每个 SWE-bench 实例包含三个核心元素：

```
┌──────────────────────────────────────────────────────┐
│                  SWE-bench 实例                       │
│                                                      │
│  ┌─────────────┐   ┌─────────────┐   ┌────────────┐ │
│  │  GitHub Issue │   │  对应 PR    │   │  测试套件   │ │
│  │             │   │             │   │           │ │
│  │ Bug 描述    │   │ 人类开发者  │   │ 单元测试   │ │
│  │ 复现步骤    │   │ 提交的代码  │   │ 集成测试   │ │
│  │ 预期行为    │   │ 修复方案    │   │ 回归测试   │ │
│  │ 实际行为    │   │ 变更的文件  │   │           │ │
│  └──────┬──────┘   └──────┬──────┘   └─────┬──────┘ │
│         │                │                │         │
│         └────────────────┼────────────────┘         │
│                          │                          │
│                     ┌────┴────┐                     │
│                     │ Agent   │                     │
│                     │ 需要:   │                     │
│                     │ 1. 理解  │                     │
│                     │ 2. 定位  │                     │
│                     │ 3. 修改  │                     │
│                     │ 4. 验证  │                     │
│                     └─────────┘                     │
└──────────────────────────────────────────────────────┘
```

### 评估流程

SWE-bench 的评估遵循严格的六步流程：

```
Step 1: 环境准备
  ┌─────────────────────────────────────┐
  │ 创建该仓库对应版本的 Docker 镜像      │
  │ 安装所有依赖和测试框架                 │
  └────────────────┬────────────────────┘
                   ▼
Step 2: 任务输入
  ┌─────────────────────────────────────┐
  │ 向 Agent 提供:                       │
  │  • Issue 标题和描述                   │
  │  • 仓库的代码库（完整 clone）          │
  │  • 允许修改的文件范围                   │
  └────────────────┬────────────────────┘
                   ▼
Step 3: Agent 执行
  ┌─────────────────────────────────────┐
  │ Agent 可以:                          │
  │  • 读取代码文件                       │
  │  • 搜索和 grep                       │
  │  • 编辑文件                          │
  │  • 运行测试                          │
  │  • 提交 patch                        │
  │  超时限制: 通常 45-60 分钟            │
  └────────────────┬────────────────────┘
                   ▼
Step 4: 生成 Patch
  ┌─────────────────────────────────────┐
  │ Agent 输出: git diff 格式的 patch    │
  │ 只包含对代码文件的修改                 │
  └────────────────┬────────────────────┘
                   ▼
Step 5: 测试验证
  ┌─────────────────────────────────────┐
  │ 将 patch 应用到 Docker 环境           │
  │ 运行 FAIL_TO_PASS 测试               │
  │ （原 Issue 对应的测试用例）            │
  │ 检查测试是否通过                      │
  └────────────────┬────────────────────┘
                   ▼
Step 6: 结果判定
  ┌─────────────────────────────────────┐
  │ % Resolved = 通过的测试 / 总测试     │
  │ 同时记录:                            │
  │  • 运行时间                          │
  │  • Token 消耗                       │
  │  • API 成本                         │
  └─────────────────────────────────────┘
```

## 数据集规模

### SWE-bench 家族

```
SWE-bench 数据集家族概览

                          SWE-bench (Full)
                           2294 实例
                        12 个 Python 仓库
                    ┌──────────┼──────────┐
                    │          │          │
               SWE-bench   SWE-bench   SWE-bench
               Lite        Verified    Multimodal
               300 实例    500 实例    517 实例
               精选子集    人工校验    含视觉元素
               ┌──────────┐
               │ SWE-bench │
               │ Multilingual │
               │ 300 实例  │
               │ 9 种语言  │
               └──────────┘
```

### 各数据集详情

| 数据集 | 实例数 | 说明 | 成本评估 | 典型用途 |
|--------|-------|------|---------|---------|
| SWE-bench (Full) | 2294 | 完整数据集 | 高 | 完整评测 |
| SWE-bench Verified | 500 | 人工筛选以确保质量 | 中高 | 标准评测 |
| SWE-bench Lite | 300 | 低成本的精选子集 | 低 | 快速迭代 |
| SWE-bench Multilingual | 300 | 9 种编程语言 | 中 | 跨语言评测 |
| SWE-bench Multimodal | 517 | 含 UI 截图的 Issue | 中高 | 多模态评测 |

### SWE-bench Verified 的诞生

SWE-bench Verified 是 OpenAI 与 SWE-bench 团队合作推出的子集。原始的 SWE-bench Full 存在一些问题：部分 Issue 描述不够清晰、测试用例有缺陷、或者修复方案不唯一。Verified 子集通过人工审查，剔除了这些问题实例，从 2294 个中筛选出 500 个高质量实例。这个子集被社区广泛接受为标准评估集合。

## 评估指标

### 核心指标: % Resolved

```
% Resolved = 通过的 "FAIL_TO_PASS" 测试数 / 总 "FAIL_TO_PASS" 测试数
```

这里的关键概念是 **FAIL_TO_PASS** 测试：

- 在应用 Agent 的 patch **之前**: 这些测试是**失败**的（代表 Bug 存在）
- 在应用 Agent 的 patch **之后**: 这些测试应该是**通过**的（代表 Bug 被修复）

此外还追踪：

- **PASS_TO_PASS**: 原本通过的测试在应用 patch 后不应被破坏（回归检测）
- **严格模式**: 同时要求 FAIL_TO_PASS 全部通过 + PASS_TO_PASS 没有新增失败

### 辅助指标

| 指标 | 含义 | 重要性 |
|------|------|--------|
| Avg. Cost ($) | 每次评估的平均 API 成本 | 中 |
| Execution Time | 平均执行时间 | 低 |
| Token Usage | 消耗的 Token 数量 | 中 |
| Trajectory Length | Agent 执行的步数 | 低 |

## 背景：从"写函数"到"修 Bug"

在 SWE-bench 之前，代码生成评测主要评估的是"写函数"的能力：

```
HumanEval (2021):    "写一个函数计算斐波那契数列"     → 几十行代码
MBPP (2021):         "写一个函数检查回文字符串"       → 几十行代码
APPS (2022):         "写算法题的解"                   → 几百行代码
```

这些基准测试评估的都是**从零编写代码**的能力，而不是**在现有代码库中修改代码**的能力。但现实中，软件工程师的大部分时间花在后者上——阅读和理解现有代码，定位 Bug 所在，然后做出精确的修改。

SWE-bench 填补了这个空白。它不是问"你能写出这个函数吗"，而是问"你能找到这个库里哪里出错了并修好它吗"。

### 难度对比

```
HumanEval: "写一个 Python 函数，对列表排序"
  代码量: ~10 行, 上下文: 自己写, 环境: 空文件

SWE-bench: "修复 django 的 bug #12345"
  代码量: ~50,000 行仓库, 上下文: 理解整个项目结构
  需要: 定位 → 理解 → 修改 → 验证 → 不破坏已有功能

难度差距: 约 2-3 个数量级
```

## 核心矛盾

**真实软件开发需要理解整个仓库 vs 只修改局部代码。**

Agent 在 SWE-bench 上面临的挑战：

```
人类的修复过程:                         Agent 面临的困难:
────────────────                      ────────────────
已经熟悉项目结构                       需要从头理解项目
了解代码惯例和模式                      不知道代码风格和约定
能通过运行和调试定位 Bug               Agent 的调试能力有限
知道哪些文件是相关的                    可能搜索到无关文件
能理解 Issue 的深层意图                 可能误解 Issue 描述
可以利用团队知识（谁写的某段代码）       没有团队协作上下文
```

另一个矛盾是：**测试覆盖率的局限性**。SWE-bench 依赖测试用例来判断修复是否正确，但：

- 测试用例可能不完整（通过了测试不代表 Bug 真的修好了）
- 测试用例可能本身有 Bug
- 一个正确的补丁可能因为测试环境问题而"未通过"

## 当前 SOTA

### SWE-bench Verified 排行榜 (截至 2026-02)

使用 mini-SWE-agent v2 统一评估框架，所有模型使用相同的 Agent 流程。

| 排名 | 模型 | % Resolved | Avg. Cost | 日期 |
|------|------|-----------|-----------|------|
| 1 | Claude 4.5 Opus (high reasoning) | **76.80%** | $0.75 | 2026-02 |
| 2 | Gemini 3 Flash (high reasoning) | **75.80%** | $0.36 | 2026-02 |
| 3 | MiniMax M2.5 (high reasoning) | **75.80%** | $0.07 | 2026-02 |
| 4 | Claude Opus 4.6 | **75.60%** | $0.55 | 2026-02 |
| 5 | GPT-5-2 Codex | **72.80%** | $0.45 | 2026-02 |
| 6 | GLM-5 (high reasoning) | **72.80%** | $0.53 | 2026-02 |
| 7 | GPT-5-2 (high reasoning) | **72.80%** | $0.47 | 2026-02 |
| 8 | Claude 4.5 Sonnet (high reasoning) | **71.40%** | $0.66 | 2026-02 |
| 9 | Kimi K2.5 (high reasoning) | **70.80%** | $0.15 | 2026-02 |
| 10 | DeepSeek V3.2 (high reasoning) | **70.00%** | $0.45 | 2026-02 |
| 11 | Gemini 3 Pro | **69.60%** | $0.96 | 2026-02 |
| 12 | Claude 4.5 Haiku (high reasoning) | **66.60%** | $0.33 | 2026-02 |
| 13 | GPT-5 Mini | **56.20%** | $0.05 | 2026-02 |

### 排行榜趋势

```
80% ┤                                    ● Claude 4.5 Opus (76.8%)
75% ┤                          ● Claude Opus 4.5 (49%)     ● MiniMax (75.8%)
70% ┤                    ● Devin (13.8%)
65% ┤
60% ┤
55% ┤
50% ┤
45% ┤
40% ┤
35% ┤
30% ┤             ● SWE-Agent (12.5%)
25% ┤
20% ┤
15% ┤   ● GPT-4 (1.7%)
10% ┤
 5% ┤
   └──────────────────────────────────────────────────
      2023            2024            2025     2026

从 1.7% 到 76.8%，SWE-bench 上的进步展示了
AI 编程能力的飞速提升——年化增长超过 100%。
```

### 成本与性能的关系

```
Cost vs. Performance (SWE-bench Verified)
  ┌────────┬────────┬────────┬────────┬────┐
  │        │        │        │        │    │
  │        │        │ Gemini3P│        │    │
  │        │        │ ($0.96)│        │    │
  │        │   │    │        │        │    │
  │        │   │    │        │ Claude │    │
  │        │   │    │        │ 4.5Opus│    │
  │        │   │    │        │($0.75) │    │
  │        │   │    │  GPT5-2│        │    │
  │        │   │    │($0.47) │        │    │
  │    │   │   │    │    │   │        │    │
  │ MiniMax│   │    │    │   │        │    │
  │($0.07) │   │    │    │   │        │    │
  └────────┴────────┴────────┴────────┴────┘
    56%     66%     70%     73%      77%
              % Resolved

Claude 4.5 Opus 在性能上领先，但成本也较高。
MiniMax M2.5 以极低的成本 ($0.07) 达到了 75.8%，
显示了极高的性价比。
```

## 著名 Agent 系统

### SWE-Agent

由普林斯顿大学团队开发（也是 SWE-bench 的创建团队），是最早也是最著名的 SWE-bench Agent。使用"Agent-Computer Interface (ACI)"设计，使 LLM 更容易与代码库进行交互。2024 年首次在 SWE-bench 上达到 12.5%。

### Devin

由 Cognition AI 开发，2024 年推出时引起轰动。号称"第一个 AI 软件工程师"。在 SWE-bench 上达到 13.8%。虽然这个成绩很快被超越，但 Devin 展示了 AI 编程 Agent 的商业化潜力。

### OpenHands (原 OpenDevin)

开源社区驱动的最成功的 SWE-bench Agent 项目。提供了完整的 Agent 框架，包括代码编辑、Shell 执行、网络搜索等功能。社区持续贡献评估结果。

### Aider

基于编辑器的代码辅助 Agent，特别擅长在现有代码库中进行精确修改。以其高效的代码编辑策略和良好的开发体验著称。

### Mini-SWE-agent

SWE-bench 团队开发的轻量级 Agent，仅用 100 行 Python 代码就达到了 65% 的 Verified 分数。是社区的标准评估框架。

```python
# mini-SWE-agent 的核心逻辑（简化）
class MiniSWEAgent:
    """极简的 SWE-bench Agent 框架"""

    def __init__(self, model, repo_path: str):
        self.model = model
        self.repo_path = repo_path
        self.patch = ""

    def run(self, issue: str) -> str:
        """给定 Issue 描述，返回 git diff patch"""

        # Step 1: 理解 Issue
        analysis = self.analyze_issue(issue)

        # Step 2: 定位相关文件
        relevant_files = self.search_relevant_files(analysis)

        # Step 3: 读取关键代码
        code_context = ""
        for file_path in relevant_files[:5]:
            code_context += self.read_file(file_path) + "\n"

        # Step 4: 生成修复方案
        fix_plan = self.generate_fix(issue, code_context)

        # Step 5: 应用修改
        for file_path, edit in fix_plan["edits"]:
            self.apply_edit(file_path, edit)

        # Step 6: 输出 patch
        return self.get_git_diff()

    def analyze_issue(self, issue: str) -> str:
        prompt = f"分析以下 Issue，找出 Bug 的类型和可能的原因:\n{issue}"
        return self.model.generate(prompt)

    def search_relevant_files(self, query: str) -> list:
        import subprocess
        result = subprocess.run(
            ["grep", "-rl", query, self.repo_path, "--include=*.py"],
            capture_output=True, text=True, timeout=30
        )
        return result.stdout.strip().split("\n")[:10]

    def read_file(self, file_path: str) -> str:
        with open(file_path, 'r') as f:
            return f.read()

    def generate_fix(self, issue: str, code: str) -> dict:
        prompt = f"""
Issue: {issue}
Code: {code[:3000]}
Output a fix plan as JSON with 'edits' list.
"""
        response = self.model.generate(prompt)
        return eval(response)

    def apply_edit(self, file_path: str, edit: dict):
        full_path = f"{self.repo_path}/{file_path}"
        with open(full_path, 'r') as f:
            content = f.read()
        content = content.replace(edit["old"], edit["new"])
        with open(full_path, 'w') as f:
            f.write(content)

    def get_git_diff(self) -> str:
        import subprocess
        result = subprocess.run(
            ["git", "diff"], cwd=self.repo_path,
            capture_output=True, text=True
        )
        return result.stdout
```

## Code Examples

### SWE-bench 评估 Harness

```python
import docker
import json
import tempfile
import os
from pathlib import Path


class SWEBenchHarness:
    """SWE-bench 评估框架的核心"""

    def __init__(self, swe_bench_data: list):
        self.data = swe_bench_data
        self.client = docker.from_env()

    def evaluate_instance(self, instance: dict,
                          agent_output: str) -> dict:
        """评估单个 SWE-bench 实例"""

        instance_id = instance["instance_id"]
        repo = instance["repo"]
        base_commit = instance["base_commit"]
        test_patch = instance["test_patch"]

        result = {
            "instance_id": instance_id,
            "repo": repo,
            "patch_applied": None,
            "tests_passed": 0,
            "tests_failed": 0,
            "tests_skipped": 0,
            "resolved": False,
            "error": None
        }

        try:
            # Step 1: 应用 Agent 的 patch
            patch_status = self.apply_patch(
                instance_id, agent_output
            )
            if not patch_status["success"]:
                result["error"] = patch_status["error"]
                return result

            result["patch_applied"] = True

            # Step 2: 运行测试
            test_result = self.run_tests(
                instance_id, test_patch
            )

            result["tests_passed"] = test_result.get("passed", 0)
            result["tests_failed"] = test_result.get("failed", 0)
            result["tests_skipped"] = test_result.get("skipped", 0)
            result["resolved"] = test_result.get("resolved", False)

        except Exception as e:
            result["error"] = str(e)

        return result

    def apply_patch(self, instance_id: str,
                    patch: str) -> dict:
        """在 Docker 容器中应用 patch"""
        container_name = f"swe-bench-{instance_id}"

        try:
            container = self.client.containers.get(container_name)
        except docker.errors.NotFound:
            return {"success": False, "error": "container_not_found"}

        # 将 patch 写入容器内的临时文件
        patch_file = "/tmp/agent_patch.diff"
        container.exec_run(
            f"bash -c 'cat > {patch_file}'",
            stdin=patch.encode()
        )

        # 应用 patch
        exit_code, output = container.exec_run(
            f"git apply --check {patch_file}"
        )
        if exit_code != 0:
            return {
                "success": False,
                "error": f"patch_apply_failed: {output.decode()}"
            }

        container.exec_run(f"git apply {patch_file}")
        return {"success": True}

    def run_tests(self, instance_id: str,
                  test_patch: str) -> dict:
        """运行测试套件"""
        container_name = f"swe-bench-{instance_id}"

        try:
            container = self.client.containers.get(container_name)
        except docker.errors.NotFound:
            return {"resolved": False, "error": "container_not_found"}

        # 写入测试 patch 并运行
        container.exec_run(
            f"bash -c 'cat > /tmp/test_patch.diff'",
            stdin=test_patch.encode()
        )
        container.exec_run("git apply /tmp/test_patch.diff")

        # 运行 FAIL_TO_PASS 测试
        exit_code, output = container.exec_run(
            "python -m pytest tests/test_fix.py -v --json-report",
            timeout=600
        )

        report = json.loads(output) if output else {}
        failed = len(report.get("failures", []))
        passed = len(report.get("passed", []))

        return {
            "resolved": failed == 0,
            "passed": passed,
            "failed": failed,
            "output": output.decode()[:500]
        }

    def evaluate_all(self, num_instances: int = None) -> dict:
        """运行完整评估"""
        instances = self.data[:num_instances] if num_instances else self.data

        results = []
        passed = 0
        for i, instance in enumerate(instances):
            print(f"Evaluating {i+1}/{len(instances)}: {instance['instance_id']}")
            agent_patch = self.agent_generate_patch(instance)
            result = self.evaluate_instance(instance, agent_patch)
            results.append(result)
            if result["resolved"]:
                passed += 1

        return {
            "total": len(instances),
            "resolved": passed,
            "percentage": round(passed / len(instances) * 100, 2),
            "results": results
        }

    def agent_generate_patch(self, instance: dict) -> str:
        """这里集成实际的 Agent 来生成 patch"""
        _ = instance
        return ""
```

### 日志分析工具

```python
class SWEBenchAnalyzer:
    """SWE-bench 评估结果分析器"""

    def __init__(self, results: list):
        self.results = results

    def summary(self) -> dict:
        """生成汇总报告"""
        resolved = [r for r in self.results if r["resolved"]]
        failed = [r for r in self.results if not r["resolved"]]

        return {
            "total": len(self.results),
            "resolved": len(resolved),
            "rate": round(len(resolved) / len(self.results) * 100, 2),
            "by_repo": self._group_by_repo(),
            "by_difficulty": self._group_by_difficulty(),
            "common_failures": self._analyze_failures(failed),
            "avg_tests_per_instance": self._avg_tests()
        }

    def _group_by_repo(self) -> dict:
        repos = {}
        for r in self.results:
            repo = r["repo"]
            if repo not in repos:
                repos[repo] = {"total": 0, "resolved": 0}
            repos[repo]["total"] += 1
            if r["resolved"]:
                repos[repo]["resolved"] += 1
        for repo in repos:
            repos[repo]["rate"] = round(
                repos[repo]["resolved"] / repos[repo]["total"] * 100, 1
            )
        return repos

    def _group_by_difficulty(self) -> dict:
        import re
        difficulties = {"easy": 0, "medium": 0, "hard": 0}
        for r in self.results:
            pass

    def _analyze_failures(self, failed_results: list) -> dict:
        reasons = {}
        for r in failed_results:
            error = r.get("error", "unknown")
            reasons[error] = reasons.get(error, 0) + 1
        return dict(sorted(reasons.items(),
                          key=lambda x: -x[1])[:5])

    def _avg_tests(self) -> float:
        total = sum(
            r["tests_passed"] + r["tests_failed"]
            for r in self.results
        )
        return round(total / len(self.results), 1)
```

## 实现挑战

### 1. Docker 环境隔离

每个 SWE-bench 实例都需要一个特定版本的 Docker 镜像，包括精确的依赖版本和测试框架。

```
挑战: 2294 个实例 → 2294 个 Docker 镜像
  每个镜像: ~1-5 GB
  总存储: ~2-10 TB 磁盘空间
  构建时间: 数小时到数天

解决方案:
  - 分层构建、共享基础层
  - 按需下载，而不是全部预构建
  - 使用缓存加速重复构建
```

### 2. 测试运行可靠性

测试运行可能因为环境问题而非代码问题而失败：

```python
class TestRunnerRobust:
    """健壮性测试运行器"""

    RETRY_LIMIT = 3
    FLAKY_TEST_THRESHOLD = 0.3

    def __init__(self):
        self.flaky_tests = set()

    def run_with_retry(self, test_command: str,
                        max_retries: int = 3) -> dict:
        """带重试的测试运行（降低 flaky test 影响）"""

        for attempt in range(max_retries):
            exit_code, output = self._exec(test_command)

            if exit_code == 0:
                return {"passed": True, "attempts": attempt + 1}

            # 检查是否是已知的 flaky test
            if self._is_flaky(output):
                continue

            return {"passed": False, "output": output,
                    "attempts": attempt + 1}

        return {"passed": False,
                "error": "max_retries_exceeded",
                "attempts": max_retries}

    def _is_flaky(self, test_output: str) -> bool:
        for flaky in self.flaky_tests:
            if flaky in test_output:
                return True
        return False

    def _exec(self, command: str):
        import subprocess
        result = subprocess.run(
            command, shell=True,
            capture_output=True, timeout=300
        )
        return result.returncode, result.stdout.decode()
```

### 3. 长时间运行成本

一次完整的 SWE-bench Full 评估（2294 个实例）的成本估算：

```
计算成本:
  Docker 构建: ~50-100 美元（云计算资源）
  API 调用: ~500-2000 美元（取决于模型价格）
  运行时间: ~24-72 小时（并行化后）

时间成本:
  串行评估: 2000+ 小时（不现实）
  并行 50 个容器: ~40 小时
  并行 200 个容器: ~10 小时

典型做法: 先跑 SWE-bench Lite (300 实例) 快速迭代
         最终评估用 SWE-bench Verified (500 实例)
```

## 扩展

### SWE-bench Verified (Anthropic 精选子集)

由 Anthropic 和 SWE-bench 团队合作，在 2024 年 8 月推出的子集。通过人工审查原始 SWE-bench 数据，剔除了描述模糊、测试有缺陷、或多个修复方案的问题。最终得到 500 个高质量的实例，是目前社区最广泛使用的评估标准。

### SWE-bench Multilingual

将 SWE-bench 扩展到 Python 之外的其他编程语言，支持 JavaScript、TypeScript、Java、Go、Rust、C、C++、Ruby、Jupyter Notebook 等 9 种语言/环境。包含 300 个实例。

### SWE-bench Multimodal

包含需要进行 UI 修改的任务（如前端 Bug 修复），相关 Issue 通常附带截图。评估 Agent 理解和处理视觉信息的能力。包含 517 个实例。

### SWE-bench-Live

持续更新的 SWE-bench 变体，从 GitHub 上实时收集新的 Issue。解决了静态基准被过拟合的问题。

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 评估代码修改能力 | 评估架构设计和系统设计能力 |
| 衡量 Bug 修复能力 | 衡量从零构建新项目的能力 |
| 提供标准化的编程 Agent 对比 | 评估代码审查和团队协作能力 |
| 追踪 AI 编程能力的进步 | 反映真实开发中的沟通和需求澄清 |
| 发现代码理解能力的短板 | 衡量 Agent 的代码质量意识（可读性/性能） |

## 与 GAIA / ToolBench 的比较

| 维度 | SWE-bench | GAIA | ToolBench |
|------|-----------|------|-----------|
| 核心能力 | 代码修改+调试 | 通用推理+工具 | API 调用 |
| 任务来源 | 真实 GitHub Issue | 专家设计 | 真实 API 文档 |
| 环境 | Docker 隔离 | 搜索+执行 | API 沙箱 |
| 评分方式 | 测试通过率 | 答案匹配 | 结果验证 |
| 人类基线 | ~90% | 92% | -- |
| SOTA (2026) | 76.8% (Claude 4.5 Opus) | 82.7% (MiroThinker) | ~70%+ |
| 规模 | 2294 (Full) | 466 | 3468 |
| 最像真实场景 | 非常接近 | 部分接近 | 中等 |
| 主要瓶颈 | Docker 开销 | 多模态处理 | API 模拟 |

**SWE-bench** 最适合评估"AI 能否作为软件工程师的助手"——它的场景最接近真实的编程工作。

**GAIA** 最适合评估"AI 能否作为通用助理"——它覆盖的能力维度最广。

**ToolBench** 最适合评估"AI 能否理解和调用陌生 API"——它的工具多样性最高。

三者的关系是互补而非替代：一个优秀的通用 Agent 应该在三个基准上都有不错的表现。
