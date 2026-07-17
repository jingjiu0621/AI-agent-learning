# 10.4.1 Step-Through Debug — 逐步调试

## 简单介绍

Step-Through Debug（逐步调试）是 Agent 调试中最基础的调试手段——让 Agent 在每一步（Thought、Action、Observation）之后暂停，等待开发者检查当前状态后决定是继续还是干预。类似于传统 IDE 的"断点 + 单步执行"，但在 Agent 上下文中是"Prompt 级"而非"代码行级"的。

## 基本原理

Agent 逐步调试的架构：

```python
class DebuggableAgent:
    """可逐步调试的 Agent"""

    def __init__(self, step_mode: bool = False):
        self.step_mode = step_mode
        self.debug_hooks = {
            "before_thought": [],
            "after_thought": [],
            "before_action": [],
            "after_action": [],
            "after_observation": [],
        }

    async def step(self, iteration: int):
        """单步执行 Agent 的一个 ReAct 循环"""

        # 1. 断点：思考前
        if self.step_mode:
            await self.wait_for_debugger("before_thought", iteration)

        thought = await self.generate_thought()

        # 2. 断点：思考后
        if self.step_mode:
            await self.wait_for_debugger("after_thought", iteration,
                                         thought_content=thought)

        action = self.parse_action(thought)

        # 3. 断点：行动前
        if self.step_mode:
            await self.wait_for_debugger("before_action", iteration,
                                         action=action)

        observation = await self.execute_action(action)

        # 4. 断点：观察后
        if self.step_mode:
            await self.wait_for_debugger("after_observation", iteration,
                                         observation=observation)

        return observation

    async def wait_for_debugger(self, hook_point: str, iteration: int, **context):
        """等待调试器交互"""
        display_state(iteration, hook_point, context)
        command = await input_async("debug> ")  # continue / step / inspect / modify
        if command == "inspect":
            show_full_context(context)
        elif command == "modify":
            context = await modify_context()
        elif command == "continue":
            self.step_mode = False
```

## 背景

传统 IDE 的逐步调试工作原理是：断点 → 暂停在代码行 → 检查变量值 → 继续。但在 Agent 中，"代码行"被替换为"Prompt 往返"（一次 LLM 调用 + 一次工具调用），"变量值"被替换为"完整的 Prompt 内容和 LLM 输出"。

逐步调试法在 Agent 开发的早期阶段尤为重要——当你在设计一个新的 Agent 行为时，你需要的不是代码 bug，而是微调 Prompt 提示语。逐步观察 Agent 在每步的表现是 Prompt 工程的最佳实践。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| print 调试 | 在框架关键函数中加 print | 每次都要改代码，输出杂乱 |
| 纯 Prompt 调试在线 | 直接在 LLM Playground 中测试 | 无法模拟工具调用和状态追踪 |
| 日志事后分析 | Agent 运行完再看日志 | 错过了"当时在想什么"的第一手感受 |
| 无调试直接上线 | 把 Agent 丢到线上靠监控 | 问题定位极慢，用户体验差 |

## 当前主流优化方向

1. **Jupyter Notebook 调试**——在 Jupyter Notebook 中运行 Agent，每个 Cell 代表一步，开发者可以自由检查和修改 Agent 的状态
2. **Web 界面步进器**——类似 LangGraph Studio 的可视化步进工具，在 Web 界面上展示 Agent 的每一步，支持查看/修改 Prompt 和状态
3. **自动驾驶调试**——Agent 运行时自动检测"异常迹象"（如重复调用同一工具、某步耗时异常），暂停并提示开发者检查
4. **条件断点**——设定条件（如 "当 tool = delete_file 时暂停"、"当 turn > 5 时暂停"），只在满足条件时才中断

## 实现的最大挑战

1. **流式 LLM 的暂停**——如果 LLM 使用流式返回，在步骤暂停意味着需要保持完整的流缓冲区，会占用更多内存
2. **状态不一致**——开发者在调试中修改了 Agent 的状态后，后续执行可能因为状态不一致而出现预料之外的行为
3. **超时问题**——调试器等待开发者输入时，外部工具调用可能超时，导致 Agent 执行环境与调试前不同

## 能力边界

**能做什么：**
- 观察 Agent 在每一步的完整状态
- 在运行时修改 Agent 的 Prompt 和参数
- 快速迭代 Agent 的行为设计
- 发现 Prompt 的语义问题（Agent 误解了指令）

**不能做什么：**
- 不能调试 LLM 的"黑盒内部"——只能看到输入输出，无法像传统调试器那样单步进入 LLM 的内部
- 不能保证调试时的修改在后续执行中完全兼容——Manual 修改可能破坏 Agent 的状态机
- 不能解决"不可复现"问题——同一个 Prompt 两次运行可能得到不同结果，逐步调试的结果可能无法复现

## 最终工程优化

1. **Event Loop 注入**——在 Agent 的事件循环中注入 `debug_hook` 回调，使外部调试器（CLI、Web UI、IDE 插件）可以无缝接入
2. **快照/恢复**——每个步骤暂停时自动创建 Agent 状态快照，开发者修改后可以从快照恢复，避免"改了一半不知道改了什么"
3. **调试命令 DSL**——设计简洁的调试命令集：`p`（print 状态）、`c`（continue）、`s`（step）、`m`（modify）、`q`（quit），降低调试认知负担
4. **非侵入式调试**——调试器的存在不应改变 Agent 的行为模式（不插入额外 Prompt、不改变 token 计数），确保调试环境与生产环境一致
