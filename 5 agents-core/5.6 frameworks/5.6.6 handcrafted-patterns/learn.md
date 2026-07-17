# 5.6.6 handcrafted-patterns — 手写 Agent 参考实现

## 简单介绍

手写 Agent（Handcrafted Agent）是不依赖框架、从零实现的 Agent。理解手写实现的模式是深入理解 Agent 架构的最佳方式——框架抽象掉了复杂性，但也隐藏了关键设计决策。

## 基本原理

### 最小 ReAct Agent

```python
import json
from openai import OpenAI

class MinimalReActAgent:
    def __init__(self, tools, max_steps=10):
        self.client = OpenAI()
        self.tools = tools  # [{name, description, parameters}]
        self.max_steps = max_steps
    
    def run(self, user_input):
        messages = [{"role": "user", "content": user_input}]
        
        for step in range(self.max_steps):
            # Step 1: 思考
            response = self.client.chat.completions.create(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": self.SYSTEM_PROMPT},
                    *messages
                ],
                tools=self.tools,
                tool_choice="auto"
            )
            
            msg = response.choices[0].message
            
            # Step 2: 检查是否应该结束
            if msg.content and not msg.tool_calls:
                return msg.content  # 最终答案
            
            # Step 3: 执行工具
            if msg.tool_calls:
                messages.append(msg)
                for tc in msg.tool_calls:
                    result = self.execute_tool(tc)
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tc.id,
                        "content": json.dumps(result)
                    })
        
        return "Max steps reached"
    
    def execute_tool(self, tool_call):
        # 根据 tool_call 调用对应的工具函数
        name = tool_call.function.name
        args = json.loads(tool_call.function.arguments)
        for tool in self.tools:
            if tool["name"] == name:
                return tool["fn"](**args)
        return {"error": f"Tool {name} not found"}
    
    SYSTEM_PROMPT = """You are a helpful assistant with access to tools.
Use the tools to help the user. When you have enough information,
provide a final answer without calling any tools."""
```

### 进化路径

```
v1: 最小 ReAct（30 行）
    ↓ 增加：
v2: 带状态管理的 Agent（结构化的 State 对象）
    ↓ 增加：
v3: 带记忆的 Agent（消息压缩 + 摘要）
    ↓ 增加：
v4: 带错误处理的 Agent（重试 + 降级）
    ↓ 增加：
v5: 带流式的 Agent（SSE 推送）
    ↓ 增加：
v6: 多 Agent 协作
```

## 背景与演进

手写 Agent 经历了从"简单的 LLM 循环"到"完整的 Agent 系统"的演进。每个框架功能都可以逐步手动添加，每次添加都加深对 Agent 架构的理解。

## 核心矛盾

**手写灵活性 vs 框架便利性**：
- 手写：完全控制、无依赖包袱、理解深入
- 框架：开箱即用、生态丰富、但受限于框架设计

## 何时选择手写

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 学习理解 | 手写 | 深入理解 Agent 核心机制 |
| 简单原型 | 手写 | 30 行代码可跑 |
| 生产系统 | 框架 | 状态管理、错误处理、流式等开箱即用 |
| 特殊需求 | 手写 | 框架无法满足的特定需求 |
| 性能优化 | 手写 | 去掉框架开销 |

## 核心优势

手写实现的最大价值是**教育意义**——让你真正理解 Agent 的每一个环节在做什么。

## 工程注意事项（手写生产级）

1. **安全性**：工具执行需要沙箱隔离
2. **可观测性**：每一步的日志和追踪
3. **限流**：LLM API 调用的速率限制
4. **幂等性**：重试时的幂等保证
5. **超时控制**：每步和整体超时
6. **序列化**：状态的可序列化

## 从手写到框架的迁移

```python
# 当手写 Agent 出现以下信号时，考虑迁移到框架：
# 1. 状态管理代码超过 Agent 逻辑代码
# 2. 错误处理散布在各个环节
# 3. 需要检查点/恢复功能
# 4. 需要流式输出
# 5. 需要多 Agent 协作
```

## 场景判断

- **学习路径**：从手写 ReAct 开始 → 逐步添加功能 → 理解框架的必要性
- **生产选择**：默认选框架，只有在框架成为瓶颈时手写
- **教学用途**：手写是理解 Agent 架构的最佳方式
