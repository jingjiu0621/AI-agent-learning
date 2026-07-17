# 手写 Agent 的适用场景与优势

## 简单介绍

"手写 Agent"指不依赖 LangChain/CrewAI/AutoGen 等框架，直接用 LLM SDK + Python/TypeScript 零基础构建的 Agent。手写不是"重新发明轮子"，而是在充分理解原理的基础上，按需构建最精简的解决方案。

## 核心优势

| 优势 | 说明 |
|------|------|
| 完全控制 | 每一行 Prompt、每一段逻辑都可精确控制 |
| 最简依赖 | 只有 LLM SDK，没有庞杂的框架依赖 |
| 调试简单 | 框架黑盒消失了，问题定位更快 |
| Prompt 透明 | 发送给 LLM 的内容完全可见可控 |
| 性能 | 没有框架的抽象开销，Token 和延迟更可控 |
| 学习效应 | 理解底层原理是掌握框架的基础 |

## 何时选择手写

### ✅ 适合手写的场景
- **学习阶段**：理解 Agent 工作原理
- **简单 Agent**：1-2 个工具，几轮交互
- **高度定制化**：框架无法满足的特定控制流
- **Prompt 敏感应用**：需要对 Prompt 有完全控制
- **最小化部署**：减少依赖和体积

### ❌ 不适合手写的场景
- 需要复杂的状态持久化和恢复
- 需要接入大量（100+）工具
- 需要框架特有的可观测性工具
- 团队多人协作开发 Agent 系统

## 从零实现 ReAct（核心代码）

```python
import json
from openai import OpenAI

client = OpenAI()

def agent_loop(user_input, tools, max_steps=5):
    messages = [
        {"role": "system", "content": "你是助手。调用工具获取信息，逐步完成任务。"},
        {"role": "user", "content": user_input}
    ]
    
    for _ in range(max_steps):
        response = client.chat.completions.create(
            model="gpt-4",
            messages=messages,
            tools=tools,
        )
        msg = response.choices[0].message
        
        if not msg.tool_calls:
            return msg.content
        
        # 处理工具调用
        messages.append(msg)
        for tc in msg.tool_calls:
            result = execute_tool(tc.function.name, json.loads(tc.function.arguments))
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": json.dumps(result)
            })
    
    return "达到最大步数限制"
```

## 手写 vs 框架的演进路径

```
学习路径：
手写（理解原理）→ 简单封装（提炼模式）→ 必要时引入框架（解决规模问题）

工程决策：
- 工具 < 5 个，步数 < 10 步 → 手写
- 需要复杂状态机 / 多人协作 → 框架
- 不确定 → 手写开始，必要时迁移到框架
```

## 关键原则

1. **手写从不用来"重写框架"**——只构建当前所需的
2. **保持核心循环最简**：Prompt + 消息列表 + 工具选择 够用就行
3. **渐进式增加复杂度**：需要时才加记忆/反思/规划，不需要不提前加
