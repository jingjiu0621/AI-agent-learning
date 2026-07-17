# 11.2.5 WebArena — 网页交互 Agent 基准

## 简单介绍

WebArena 是由 CMU、Google DeepMind 等机构联合提出的**网页交互式 Agent 基准测试**，论文发表于 ICLR 2024。它提供了一个包含 812 个真实世界任务的评测环境，任务范围涵盖电商购物、内容管理、论坛互动、代码协作和地图导航等场景。

与 GAIA 侧重于多步推理、SWE-bench 侧重于代码修改不同，WebArena 的核心设计理念是：**Agent 需要在真实的、功能完整的网站上完成端到端任务**，而不是通过 API 调用或简化模拟器。

```
WebArena 的核心信念：
  如果你能用自然语言指挥一个 Agent 完成网页上的任何操作，
  那它就是一个真正有用的通用 Agent。

  因为网页是"人类数字世界的事实标准接口"。
```

## 基本原理

### WebArena 的 5 个自建环境

WebArena 没有使用真实的互联网（不可控、不可复现），而是自建了 5 个功能完整的网站：

```
Environment        Base Software        #Tasks     #Templates     Skill Tested
─────────────────  ───────────────────  ────────  ─────────────  ─────────────────────
 1. Shopping       Magento + Luma           223        20,288     产品搜索、下订单、管理购物车
 2. CMS            WordPress + WooCommerce  136        12,496     创建/编辑内容、管理文章
 3. Forum          phpBB                     93        25,147     发帖、浏览、管理论坛内容
 4. GitLab         GitLab CE                105        15,800     仓库操作、Issue 管理、CI
 5. Map            OpenStreetMap (自制前端)   83         1,213     路线规划、地点搜索、信息查询
 ──────────────────────────────────────────────────────────────────────────────────
 Total: ──────────────────────────────────   640 (初始版)
                                              812 (扩展版)
```

每个网站都是功能完整的——你可以像操作真实网站一样在上面点击、滚动、搜索、填写表单。

### 任务类型

WebArena 的任务分为三类，每类考察不同的 Agent 能力：

```
任务类型                   占比    示例
────────────────────────  ────  ───────────────────────────────────────────
信息检索 (Information)     45%  "Shop 中评分最高的无线耳机是哪款？价格是多少？"
内容操作 (Content)         27%  "在 CMS 中创建一个新页面，标题为'About Us'，内容为..."
导航导航 (Navigation)      28%  "从 GitLab 的 Issue #42 页面跳转到合并请求 #15"

开发者根据模板自动生成多样化任务，避免模板固化。
```

### 评估指标

WebArena 使用两个主要指标：

```
Success Rate (SR):
  ──────────────────────────────────────────────────────────────────────
  所有任务中完全成功的比例。
  判定标准严格——Agent 必须完成所有目标条件才算成功。

Progress Rate (PR):
  ──────────────────────────────────────────────────────────────────────
  部分成功程度。每个任务分解成多个子目标，PR 衡量完成子目标的比例。
  例如一个 5 步的任务只完成了 3 步，PR = 60%。

  为什么要 PR？
  ── 有些任务很长（20+ 步），Agent 走了 19 步但最后 1 步错了，
     SR 给 0%，但 PR 能反映实际完成的工作量。
```

### WebArena 的自动评分机制

```
                         ┌──────────────────────┐
                         │  任务模板 + 参数实例    │
                         │  (Template + params)  │
                         └──────────┬───────────┘
                                    ▼
                         ┌──────────────────────┐
                         │   初始化目标状态        │
                         │   (意图 + 约束条件)     │
                         └──────────┬───────────┘
                                    ▼
        Agent 执行 ──────────► ┌──────────────────────┐
                               │  环境状态快照          │
                               │  (页面截图 + DOM)     │
                               └──────────┬───────────┘
                                          ▼
                               ┌──────────────────────┐
                               │  自动验证器            │
                               │  • 基于可执行 JS 脚本   │
                               │  • 检查 DOM 状态      │
                               │  • 检查页面 URL       │
                               │  • 检查特定元素是否存在 │
                               └──────────┬───────────┘
                                          ▼
                               ┌──────────────────────┐
                               │  Pass/Fail + Partial  │
                               └──────────────────────┘

    关键设计：验证器是脚本化的，不是 LLM 判定的。
    所以评分是确定性的、可复现的。
```

## 背景

### 为什么需要 WebArena？

在 WebArena 之前，网页 Agent 的评测方式存在严重问题：

```
🔴 使用真实网站（不可复现）
  网站结构每天在变 → 今天能通过的测试明天可能失败
  API 版本更新 → Agent 表现波动不是自身问题
  无法横向对比不同研究工作

🔴 使用简化模拟器（不真实）
  模拟器只有几个页面 → 无法衡量泛化能力
  交互模式过于简单 → Agent 在模拟器上表现好但在真实网站上不行
  模拟器不包含真实的 UI 元素 → Agent 学不会处理复杂页面

🔴 使用人工评估（不可扩展）
  人工评估成本高 → 只能评估少量场景
  评分标准不一致 → 不同评审者结果差异大
```

WebArena 的解决方案：**自建完整网站 + 自动化验证**。

### 与类似基准对比

```
Benchmark       环境类型       任务数    复现性    真实性    自动评分
──────────────  ───────────   ───────  ───────  ──────  ────────
WebArena        自建完整网站      812      ✅      ✅✅     ✅
MiniWoB++       简化网页模拟器    100      ✅      ❌       ✅
Mind2Web        真实网页截图     2350     ❌      ✅       ✅
StateFlow       简化表单        200      ✅      ❌       ✅
VisualWebArena  自建视觉网站     910      ✅      ✅✅     ✅
```

## 核心矛盾

**可控性与真实性的矛盾。**

```
要可控（可复现）：网站必须自建，内容和结构固定
    → 但自建网站再像真的，也不是真的
    → Agent 可能过拟合到自建网站的特定模式

要真实（有意义）：评测应该发生在真实网站上
    → 但真实网站不可控，评测结果不可复现
    → Agent 失败的原因可能是网站变了，不是 Agent 变了

WebArena 的妥协：
  自建功能完整的网站，但尽量模仿真实网站的复杂性。
```

## 当前 SOTA

### 主要模型在 WebArena 上的表现

```
模型/方法                           Success Rate     Progress Rate    发布时间
────────────────────────────────  ──────────────  ────────────────  ──────────
GPT-4V + Set-of-Marks (SoM)           28.1%           43.2%          2024-04
CogAgent                               27.4%           42.8%          2024-05
GPT-4V + SeeClick                      25.2%           40.1%          2024-05
GPT-4V                                 22.3%           36.7%          2024-03
Gemini Pro Vision                      18.5%           31.2%          2024-03
Claude-3 Opus                          17.1%           29.5%          2024-04
GPT-4                                  12.3%           22.1%          2024-01
人类基准                                78.2%           89.5%          2024-01
```

### 关键发现

```
1. 网页 Agent 还远未成熟
   GPT-4V + SoM 也才 28%，距离人类 78% 差距很大
   → 网页交互是 Agent 的硬骨头

2. 视觉能力至关重要
   GPT-4V (22.3%) > GPT-4 (12.3%)，视觉模型的优势非常明显
   原因：网页布局信息对任务完成至关重要

3. 结构化标注帮助大
   SoM (Set-of-Marks) 在网页元素上标注数字编号
   让 Agent 可以直接"点击 #7"而不是描述元素位置
   → 提升约 6 个百分点

4. 不同环境难度差异大
   Map 环境最难（~15%），Commerce 环境相对容易（~30%）
   原因：Map 交互密度高（拖拽、缩放）且元素更小
```

### 2025-2026 最新进展

```
模型/方法                           Success Rate    关键改进
────────────────────────────────  ──────────────  ──────────────────────
GPT-4o + Agent Workflow Memory        41.5%        记忆工作流重用 (>10% 绝对提升)
Claude-3.5 Sonnet + SFT UI Agent      38.2%        专用 UI Agent 微调
Gemini 2.0 + Grounding                36.8%        增强的视觉 grounding
WebVoyager (开源方案)                   35.5%        树搜索增强
Grounded Decoding                     33.1%        DOM 结构化重写
GPT-4-mini + AgentTrek               32.4%       BrowseRL 训练策略
```

注：WebArena 社区已经基本停止在原始 812 集上的显著突破（部分源于过拟合）。研究方向正向 VisualWebArena 转移，因为其视觉任务更丰富。

## 实现挑战

### 1. 环境维护成本高

5 个自建网站需要持续维护，每个网站基于真实开源软件构建：

```python
WEBARENA_ENVIRONMENTS = {
    "shopping": {
        "base": "Magento 2.4",
        "docker_image": "webarena_shopping:latest",
        "setup_time_seconds": 180,
        "data_seed_script": "seed_shopping.sh",
        "known_issues": [
            "Magento 缓存导致状态不一致",
            "Magento 价格计算依赖于复杂的规则引擎"
        ]
    },
    "cms": {
        "base": "WordPress 6.2 + WooCommerce",
        "docker_image": "webarena_cms:latest",
        "setup_time_seconds": 120,
        "data_seed_script": "seed_cms.sh",
        "known_issues": [
            "WordPress 插件兼容性问题",
            "多语言支持不完整"
        ]
    },
    "forum": {
        "base": "phpBB 3.3",
        "docker_image": "webarena_forum:latest",
        "setup_time_seconds": 90,
        "data_seed_script": "seed_forum.sh",
        "known_issues": [
            "phpBB 模板引擎限制",
            "会话管理复杂性"
        ]
    },
    "gitlab": {
        "base": "GitLab CE 16.0",
        "docker_image": "webarena_gitlab:latest",
        "setup_time_seconds": 240,
        "data_seed_script": "seed_gitlab.sh",
        "known_issues": [
            "GitLab 首次启动需要额外配置时间",
            "仓库克隆操作可能超时"
        ]
    },
    "map": {
        "base": "OpenStreetMap + custom frontend",
        "docker_image": "webarena_map:latest",
        "setup_time_seconds": 120,
        "data_seed_script": "seed_map.sh",
        "known_issues": [
            "地图渲染依赖 GPU（无 GPU 可能很慢）",
            "坐标精度偏差影响验证"
        ]
    }
}
```

### 2. Agent-Computer Interface (ACI) 设计的差异性

不同的网页操作接口严重影响性能：

```
接口类型              SR    优势              劣势
───────────────  ───────  ─────────────────  ─────────────────────
SoM (编号标记)     28.1%   清晰，减少定位错误   依赖视觉模型质量
SeeClick          25.2%   直观               点击坐标精度不足
纯 DOM 解析        22.0%   精确，适合文本模型   HTML 过长 token 不够
Accessibility Tree 26.4%   结构化，简洁         某些元素没有 AT 支持
多模态+DOM 混合     30.5%   兼有两种优势         架构复杂，延迟增加
```

### 3. 长任务中的累积误差

WebArena 某些任务需要 15-20 个步骤。每一步的微小错误会累积：

```python
def simulate_accumulated_error(steps: int, per_step_error: float) -> dict:
    """模拟累积误差对任务成功率的影响"""
    cumulative_success = 1.0

    for step in range(steps):
        # 每一步的误差概率
        step_success = 1.0 - per_step_error
        cumulative_success *= step_success

    return {
        "steps": steps,
        "per_step_error": per_step_error,
        "overall_success_rate": round(cumulative_success, 3),
        "variance_impact": (
            "高" if steps > 10 and cumulative_success < 0.3
            else "中" if steps > 5 and cumulative_success < 0.6
            else "低"
        )
    }

# 假设每一步 7% 的错误率
print(simulate_accumulated_error(15, 0.07))
# → overall_success_rate: 0.337
# 15 步的任务即使每一步只有 7% 的出错率，整体成功率也只有 34%

print(simulate_accumulated_error(5, 0.07))
# → overall_success_rate: 0.696
# 5 步的任务同样的错误率，整体成功率为 70%
```

### 4. 状态空间巨大

网页交互的状态空间几乎是无限的：
- 同一个页面可能有数千个可交互元素
- 每个元素可以有多种状态（可见/隐藏、启用/禁用、选中/未选中）
- 页面之间可以任意导航

## 与 VisualWebArena 的关系

VisualWebArena 是 WebArena 的视觉增强版本：

```
WebArena (2024) ────────► VisualWebArena (2024)
   812 tasks                    910 tasks
   5 environments               6 environments
   以 DOM 交互为主              以视觉交互为主
   文本密集型任务                视觉密集型任务

VisualWebArena 新增的维度：
  • 视觉理解（识别图片内容、图标含义）
  • 视觉定位（精确点击、拖拽）
  • 视觉推理（看图理解上下文）
  • 视觉反馈（页面截图作为反馈信号）

相同点：
  • 同样的 5 个平台（Shop/CMS/Forum/GitLab/Map）
  • 同样的 Docker 化环境管理
  • 同样的自动化验证体系
```

## 代码实现：WebArena Runner

```python
import json
import docker
import subprocess
from typing import Optional
from selenium import webdriver
from selenium.webdriver.common.by import By


class WebArenaEnvironment:
    """WebArena 单个环境的封装"""

    def __init__(self, env_name: str, config_path: str):
        self.env_name = env_name
        self.config = self._load_config(config_path, env_name)
        self.driver: Optional[webdriver.Remote] = None
        self.container_id: Optional[str] = None

    def _load_config(self, config_path: str, env_name: str) -> dict:
        """加载环境配置"""
        full_path = f"{config_path}/{env_name}_config.json"
        with open(full_path) as f:
            return json.load(f)

    def setup(self):
        """启动 Docker 容器浏览器驱动"""
        self.container_id = self._start_docker()
        self.driver = self._create_webdriver()

    def _start_docker(self) -> str:
        """启动环境 Docker 容器"""
        client = docker.from_env()
        container = client.containers.run(
            image=f"webarena_{self.env_name}:latest",
            detach=True,
            ports={self.config["port"]: self.config["port"]},
            environment=self.config.get("env_vars", {}),
            remove=True
        )
        return container.id

    def _create_webdriver(self) -> webdriver.Remote:
        """创建 Selenium WebDriver"""
        chrome_options = webdriver.ChromeOptions()
        chrome_options.add_argument("--no-sandbox")
        chrome_options.add_argument("--disable-dev-shm-usage")
        chrome_options.set_capability("browserName", "chrome")
        chrome_options.set_capability("goog:chromeOptions", {
            "args": ["--window-size=1280,720"]
        })

        return webdriver.Remote(
            command_executor="http://localhost:4444/wd/hub",
            options=chrome_options
        )

    def get_state(self) -> dict:
        """获取当前页面状态（DOM + 截图）"""
        dom = self.driver.page_source
        screenshot = self.driver.get_screenshot_as_base64()
        return {"dom": dom, "screenshot": screenshot}

    def execute_action(self, action: dict) -> str:
        """
        执行 Agent 的网页操作。

        action 格式：
            {"type": "click", "target": "element_id"}
            {"type": "type", "target": "input_id", "value": "hello"}
            {"type": "scroll", "direction": "down"}
            {"type": "navigate", "url": "..."}
        """
        action_type = action["type"]

        if action_type == "click":
            element = self.driver.find_element(By.ID, action["target"])
            element.click()
        elif action_type == "type":
            element = self.driver.find_element(By.ID, action["target"])
            element.clear()
            element.send_keys(action["value"])
        elif action_type == "scroll":
            js = f"window.scrollBy(0, {'100' if action['direction'] == 'down' else '-100'})"
            self.driver.execute_script(js)
        elif action_type == "navigate":
            self.driver.get(action["url"])
        else:
            raise ValueError(f"Unknown action: {action_type}")

        return f"Action {action_type} executed"

    def teardown(self):
        """清理环境"""
        if self.driver:
            self.driver.quit()
        if self.container_id:
            client = docker.from_env()
            container = client.containers.get(self.container_id)
            container.stop()


class WebArenaEvaluator:
    """WebArena 评估器"""

    def __init__(self, config_path: str, env_names: list = None):
        self.config_path = config_path
        self.envs = env_names or [
            "shopping", "cms", "forum", "gitlab", "map"
        ]
        self.tasks = self._load_all_tasks()
        self.results = []

    def _load_all_tasks(self) -> dict:
        """加载所有环境的任务"""
        all_tasks = {}
        for env in self.envs:
            with open(f"{self.config_path}/{env}_tasks.json") as f:
                all_tasks[env] = json.load(f)
        return all_tasks

    def evaluate_agent(self, agent) -> dict:
        """在 WebArena 上评估 Agent"""
        results = {}

        for env_name in self.envs:
            env = WebArenaEnvironment(env_name, self.config_path)
            env_results = []

            for task in self.tasks[env_name]:
                env.setup()

                # Agent 在环境中执行任务
                observation = env.get_state()
                trajectory = []

                for step in range(task.get("max_steps", 15)):
                    action = agent.act(observation, task["intent"])
                    feedback = env.execute_action(action)
                    observation = env.get_state()

                    trajectory.append({
                        "step": step,
                        "action": action,
                        "observation_snapshot": {
                            "url": observation.get("url", ""),
                            "title": observation.get("title", "")
                        }
                    })

                    if self._is_done(action, observation):
                        break

                # 验证结果
                task_result = self._validate_task(
                    task, env, trajectory
                )
                env.teardown()
                env_results.append(task_result)

            results[env_name] = self._aggregate(env_results)

        return results

    def _validate_task(self, task: dict, env, trajectory: list) -> dict:
        """执行验证脚本检查任务完成情况"""
        for checker in task["checkers"]:
            # 验证脚本是可执行的 JS，在页面上下文中运行
            result = env.driver.execute_script(checker["script"])
            if not result:
                return {
                    "task_id": task["id"],
                    "success": False,
                    "progress": self._calc_progress(task, env),
                    "steps": len(trajectory)
                }

        return {
            "task_id": task["id"],
            "success": True,
            "progress": 1.0,
            "steps": len(trajectory)
        }

    def _calc_progress(self, task: dict, env) -> float:
        """计算部分进度"""
        passed = 0
        for checker in task["checkers"]:
            if env.driver.execute_script(checker["script"]):
                passed += 1
        return passed / len(task["checkers"]) if task["checkers"] else 0

    def _aggregate(self, results: list) -> dict:
        """汇总评估结果"""
        n = len(results)
        successes = sum(1 for r in results if r["success"])
        return {
            "success_rate": successes / n,
            "progress_rate": sum(r["progress"] for r in results) / n,
            "total": n,
            "succeeded": successes
        }
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 评估网页交互 Agent 的端到端能力 | 评估非网页场景（移动 App、桌面应用） |
| 提供可复现的网页交互评测环境 | 完全模拟真实互联网的复杂性和不确定性 |
| 衡量 Agent 从视觉元素定位到任务规划的综合能力 | 评估需要长期记忆（跨越数天/数周）的任务 |
| 自动化验证减少人工成本 | 评估开放式创意任务（如"设计一个更好的网站"） |
| 支持多种交互模式（DOM/视觉/AT）的对比 | 衡量用户体验质量（加载速度、页面美观度） |
| 细分不同平台能力的强弱分析 | 预测 Agent 在全新网站类型上的表现 |

## 工程优化方向

1. **ACI 优化是当前最大杠杆**：SoM 标注、Accessibility Tree 结构化改写、DOM 压缩等接口层面的改进对性能影响最大（5-10%）

2. **多步任务错误恢复机制**：长任务中引入中间验证点和回退策略，当检测到当前行动偏离目标时自动回退到上一个稳定状态

3. **网站模板泛化训练**：使用不同的网站模板（不仅仅是 Magento/WordPress），训练 Agent 适应不同 UI 布局

4. **记忆与快捷操作缓存**：对于常见操作模式（如搜索商品、创建 Issue）建立快捷路径记忆，减少重复导航的步数

5. **对比式验证增强**：使用"对比状态快照"而非"目标状态匹配"来验证任务完成，提升验证的准确性

6. **视觉 + DOM 融合**：单一模态在 WebArena 上不够用，最佳方案是融合视觉理解+DOM 结构+Accessibility Tree

## 与 GAIA / ToolBench / SWE-bench 的对比

```
维度            WebArena                GAIA                 ToolBench             SWE-bench
────────────  ────────────────────  ──────────────────  ───────────────────  ───────────────────
评测焦点       网页端到端交互          多步推理与工具使用      API 工具调用           代码修改

环境类型       自建完整网站             无固定环境             RapidAPI 模拟          真实代码仓库

交互方式       网页点击+滚动+输入       API 调用+文本         API 调用               代码编辑

任务数量       812                    466 (165 公开)        >3000                 2294

主要指标       Success/Progress Rate  任务完成率            Pass/Preference Rate   % Resolved

当前 SOTA     ~42%                   ~83%                 ~64%                  ~77%

人类基线       78%                    ~92%                 ~80%                  ~90%+

难度评定       高（多步+视觉+交互）    中高（推理深度）       中（工具广度）          高（代码精度）
```

## 最佳实践建议

1. **评测时使用多种 ACI**：不要只测一种交互模式，DOM/AT/SoM 各跑一遍，取最优或平均
2. **关注 Progress Rate 而非 Success Rate 单一指标**：PR 在多步任务中更有区分度
3. **分环境报告**：5 个环境的难度差异很大，汇总 SR 会掩盖各环境表现
4. **最少运行 3 次取平均**：网页 Agent 的随机性较高，单次运行不具统计意义
5. **使用 VisualWebArena 补充**：如果 Agent 有视觉能力，VisualWebArena 是更全面的评测
