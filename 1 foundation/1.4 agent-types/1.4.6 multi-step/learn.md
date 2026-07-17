# Multi-step Agent：多步工具调用与状态跟踪

## 简单介绍

Multi-step Agent 是 ReAct 模式的工程化产物——Agent 在多次工具调用中维护状态，每个工具调用的结果影响后续的推理和决策。大多数生产级 Agent 本质上是 Multi-step Agent：它们处理复杂的多步骤工作流，跟踪中间状态，并根据新信息动态调整行动。

## 基本原理

### 多步 vs 单步

```
单步 Agent:
用户输入 → [LLM 推理 + 工具调用] → 输出

多步 Agent:
用户输入 → [Step1: 推理 → 工具A → 结果] 
         → [Step2: 推理 → 工具B → 结果] 
         → [Step3: 推理 → 综合所有结果 → 输出]
```

### 状态跟踪的内容

| 状态 | 内容 | 目的 |
|------|------|------|
| 任务进度 | 已完成的步骤、当前步骤 | 知道"进行到哪了" |
| 中间结果 | 每步工具调用结果 | 后续推理的输入 |
| 上下文 | 完整的对话历史 | 理解任务的来龙去脉 |
| 约束 | Token 预算、可用工具 | 控制 Agent 行为 |

## 核心挑战

| 挑战 | 表现 | 解决方案 |
|------|------|---------|
| 上下文膨胀 | 多步后历史过长 | 摘要 + 裁剪 |
| 错误积累 | 早期错误影响后续所有步骤 | 检查点 + 回溯 |
| 状态混淆 | Agent 忘记前面做了什么 | 结构化工作记忆 |
| 工具依赖 | 后一步依赖前一步结果 | 依赖图调度 |

## 工程实现

```python
class MultiStepAgent:
    def __init__(self, tools, max_steps=10):
        self.tools = tools
        self.max_steps = max_steps
        self.history = []
        self.workspace = {}  # 工作记忆
    
    def run(self, task):
        for step in range(self.max_steps):
            # 检查停止条件
            if self.should_stop():
                break
                
            # LLM 决定下一步
            thought, action = self.llm_think(
                task=task, 
                history=self.history,
                workspace=self.workspace
            )
            
            # 执行行动
            if action["type"] == "finish":
                return action["output"]
                
            result = self.execute_tool(action)
            
            # 保存状态
            self.history.append({"thought": thought, "action": action, "result": result})
            self.workspace[action["id"]] = result
```

## 优化方向

1. **状态压缩**：多步后，将早期步骤自动摘要，释放上下文空间
2. **并行执行**：识别独立的工具调用并批量执行
3. **分支处理**：不同路径探索不同方案
4. **恢复机制**：从检查点恢复失败的任务
