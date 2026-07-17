# 10.4.2 Replay Debugging — 回放调试

## 简单介绍

Replay Debugging（回放调试）是"事后调试"的利器。它记录 Agent 一次执行的全部轨迹（Thought、Action、Observation 的完整内容），然后允许开发者以回放的方式"重新体验"Agent 的执行过程——快进、暂停、检查每个步骤的状态。

## 基本原理

回放调试的核心架构：

```
记录阶段（Agent 正常执行）:
  Agent 执行 → 记录执行轨迹 → 写入回放文件

回放阶段（开发者调试）:
  读取回放文件 → Replay Engine → 逐步展示 Agent 的每步状态
                                    ↓
                              开发者检查/分析
```

```python
class ReplayRecorder:
    """记录 Agent 执行轨迹"""

    def record_turn(self, turn: int, thought: str,
                    action: dict, observation: dict,
                    context: dict) -> None:
        self.trajectory.append({
            "turn": turn,
            "timestamp": time.time(),
            "thought": {                # 记录完整的 Thought
                "content": thought,
                "token_count": count_tokens(thought),
            },
            "action": {
                "tool": action["tool"],
                "params": action["params"],
                "timestamp_call": action["_timestamp"],
            },
            "observation": {
                "content": observation,          # 完整的观察结果
                "truncated": len(str(observation)) > 10_000,
            },
            "context_snapshot": {       # Agent 的状态快照
                "message_history_length": len(context["messages"]),
                "turn_count": turn,
                "token_usage": context["token_usage"],
            },
        })

    def save(self, path: str):
        """保存回放文件"""
        with open(path, "w") as f:
            json.dump({
                "agent_config": self.config,
                "start_time": self.start_time,
                "trajectory": self.trajectory,
                "final_output": self.final_output,
            }, f, indent=2)


class ReplayEngine:
    """回放引擎"""

    def __init__(self, replay_data: dict):
        self.trajectory = replay_data["trajectory"]
        self.current_step = 0

    async def play(self):
        """按时间序回放 Agent 的执行过程"""
        for step in self.trajectory:
            await self.show_step(step)
            cmd = await self.wait_for_command()
            if cmd == "pause":
                await self.interactive_inspect(step)
            elif cmd == "skip":
                continue
            elif cmd == "speed_2x":
                pass  # 加速回放

    async def show_step(self, step: dict):
        """显示某一步的完整信息"""
        print(f"\n{'='*60}")
        print(f"Step {step['turn']} | "
              f"Tool: {step['action']['tool']} | "
              f"Thought Tokens: {step['thought']['token_count']}")
        print(f"{'='*60}")
        print(f"[THOUGHT]\n{step['thought']['content']}\n")
        print(f"[ACTION] {step['action']['tool']}({step['action']['params']})\n")
        print(f"[OBSERVATION] {truncate(str(step['observation']['content']), 500)}")
```

## 背景

Replay Debugging 的概念源自分布式系统调试（如 Twitter 的 Zipkin Replay、Cypress 的 Test Replay）。被引入 Agent 领域后，它解决了 Agent 调试中最痛苦的问题：**无法复现**。Agent 的非确定性意味着同一个 Prompt 两次运行可能产生不同的行为。有了完整轨迹记录 + 回放能力，开发者可以在"当时发生了什么"的确定记录上进行分析。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 手动检查日志 | 逐条查看日志 JSON | 效率极低，无法快速定位关键步骤 |
| 重新运行看问题 | 复制 Prompt 去 Playground 重测 | 非确定性导致"这次好了上次的错看不到了" |
| 录屏 | 录下 Agent 执行过程的屏幕 | 无法交互、无法深入检查内部状态 |
| 直接在线调试 | 把调试器附加到生产 Agent | 太危险，可能影响线上用户 |

## 当前主流优化方向

1. **双向回放**——不仅支持从开始到结束的"正向回放"，还支持"反向回放"：从失败的步骤回溯到最早出现异常迹象的步骤
2. **回放 + 评估**——在回放中自动注入评估点：每步结束后调用评估函数（如检测是否出现幻觉、是否重复调用工具），在回放过程中实时标注问题点
3. **修改回放**——在回放中修改某一步的 Observation（如"如果工具返回正确结果会怎样"），观察修改后 Agent 假设的行为变化
4. **批量回放分析**——自动回放大量失败的 Trace，识别共同的失败模式（如"第七步 Agent 开始陷入循环"，可能提示需要增加终止条件）

## 实现的最大挑战

1. **回放文件的大小**——Agent 的单次执行轨迹可能包含大量数据（完整的 Prompt 内容、工具返回的完整结果），一个复杂的 Agent 回放文件可能有几十 MB
2. **外部依赖的复活**——回放时，工具调用的结果来自记录文件而非实际调用，但如果回放环境中 Agent 尝试调用外部工具（如"再查一下"），记录文件中的结果不够用
3. **时间偏移**——回放时的时间戳与记录时不同，如果 Agent 的逻辑依赖于时间（如缓存时效），回放可能与原始行为不一致

## 能力边界

**能做什么：**
- 在不修改 Agent 代码的情况下，"重新体验"Agent 的执行过程
- 精确检查 Agent 在每一步的完整输入输出
- 识别 Agent 出现错误的"拐点"——哪一步开始走偏
- 保留"失败案例"用于测试和回归

**不能做什么：**
- 不能修改回放中 Agent 的行为——回放只是展示记录，不是重新执行
- 不能保证记录的过程是完整的——如果 Agent 框架没有正确记录关键数据，回放也无法还原
- 不能替代"活体调试"——对于 LLM 返回的即时变化（如随机性导致的不同输出），回放无法捕捉

## 最终工程优化

1. **流式回放文件格式**——使用 JSON Lines 格式而非一个巨大的 JSON 文件存储轨迹，每步独立一行，支持流式读取和部分加载
2. **差异回放**——只记录每步相对于上一步的变化（增量），而非完整的 Prompt 内容，将回放文件大小减少 60-80%
3. **回放索引**——为回放文件建立快速索引（步数、工具名称、关键词等），支持"跳转到 Agent 调用了 delete 工具的那一步"
4. **自动摘要生成**——回放完成后，自动生成"执行摘要"：共多少步、调用了哪些工具、每步耗时、首次出现错误的位置，帮助开发者快速了解回放内容
