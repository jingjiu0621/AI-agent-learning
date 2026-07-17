# 7.2.6 ReAct Implementation — 从零实现 ReAct（Python）

## 简单介绍

从零实现一个最小但完整的 ReAct Agent，是理解 Agent 内部机理最有效的方式。本节提供一个生产级的 Python 参考实现，覆盖核心循环、工具管理、上下文维护和错误处理。

## 基本原理

一个最小 ReAct Agent 只需要以下组件：

1. **LLM 客户端**：与大模型交互的接口
2. **工具注册表**：可用工具的注册和调用
3. **主循环引擎**：驱动 Thought→Action→Observation 循环
4. **消息历史**：维护循环上下文
5. **解析器**：从 LLM 输出中提取 Action

## 核心实现

```python
"""
ReAct Agent 最小实现
依赖: openai (>=1.0)
运行: pip install openai
"""

import json
import re
from typing import Callable, Any

# ─── 1. 工具定义 ───────────────────────────────────────

class Tool:
    """通用工具封装"""
    def __init__(self, name: str, description: str, 
                 parameters: dict, fn: Callable):
        self.name = name
        self.description = description
        self.parameters = parameters  # JSON Schema
        self.fn = fn
    
    def to_openai_tool(self) -> dict:
        """转换为 OpenAI Function Calling 格式"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters
            }
        }
    
    async def execute(self, **kwargs) -> Any:
        """执行工具函数"""
        try:
            result = await self.fn(**kwargs) if asyncio.iscoroutinefunction(self.fn) \
                     else self.fn(**kwargs)
            return result
        except Exception as e:
            return f"Error: {str(e)}"


# ─── 2. Agent 实现 ──────────────────────────────────────

import asyncio
from openai import AsyncOpenAI

class ReActAgent:
    """最小但完整的 ReAct Agent 实现"""
    
    def __init__(self, 
                 model: str = "gpt-4o",
                 max_steps: int = 10,
                 system_prompt: str = None):
        self.client = AsyncOpenAI()
        self.model = model
        self.max_steps = max_steps
        self.tools: dict[str, Tool] = {}
        
        self.system_prompt = system_prompt or self._default_prompt()
    
    def register_tool(self, tool: Tool):
        """注册可用工具"""
        self.tools[tool.name] = tool
    
    def _default_prompt(self) -> str:
        return """你是 ReAct Agent——通过"思考→行动→观察"循环完成任务。
每次回复必须包含 Thought 和 Action。
任务完成时，输出 Final Answer。"""
    
    async def run(self, user_input: str) -> str:
        """运行 ReAct Agent"""
        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": user_input}
        ]
        
        openai_tools = [t.to_openai_tool() for t in self.tools.values()]
        
        for step in range(self.max_steps):
            # Step A: LLM 推理 → 生成 Thought + Action
            response = await self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                tools=openai_tools if openai_tools else None,
                tool_choice="auto" if openai_tools else None,
            )
            
            message = response.choices[0].message
            
            # Step B: 检查是否完成
            if message.content and "Final Answer" in message.content:
                # 提取最终答案
                final = message.content.split("Final Answer:")[-1].strip()
                return final
            
            # Step C: 处理工具调用
            if message.tool_calls:
                for tc in message.tool_calls:
                    # 解析 Action
                    tool_name = tc.function.name
                    tool_args = json.loads(tc.function.arguments)
                    
                    # 执行 Action → 获取 Observation
                    tool = self.tools.get(tool_name)
                    if not tool:
                        observation = f"Error: 未找到工具 {tool_name}"
                    else:
                        observation = await tool.execute(**tool_args)
                    
                    # 注入 Observation 到上下文
                    messages.append({
                        "role": "assistant",
                        "content": message.content,
                        "tool_calls": [tc]
                    })
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tc.id,
                        "content": str(observation)
                    })
            else:
                # 无工具调用，继续对话
                messages.append({"role": "assistant", "content": message.content or ""})
        
        # 达到最大步数，返回当前进展
        return f"已达到最大步数 ({self.max_steps})。当前进展:\n{messages[-1].get('content', '')}"
    
    async def stream_run(self, user_input: str):
        """流式版本——实时输出每个步骤"""
        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": user_input}
        ]
        openai_tools = [t.to_openai_tool() for t in self.tools.values()]
        
        for step in range(self.max_steps):
            yield f"\n{'='*40}\nStep {step + 1}\n{'='*40}\n"
            
            response = await self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                tools=openai_tools,
                stream=True
            )
            
            # 处理流式响应（略——需要在 stream 模式下处理 tool_calls）
            # 这里简化处理
            ...


# ─── 3. 使用示例 ────────────────────────────────────────

async def main():
    agent = ReActAgent(model="gpt-4o")
    
    # 注册工具
    agent.register_tool(Tool(
        name="search",
        description="搜索网络信息",
        parameters={
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "搜索关键词"}
            },
            "required": ["query"]
        },
        fn=lambda query: f"模拟搜索结果: {query} 的相关信息..."
    ))
    
    # 运行
    result = await agent.run("2024年诺贝尔物理学奖得主是谁？")
    print(result)

if __name__ == "__main__":
    asyncio.run(main())


# ─── 4. 进阶：带结构化 Thought 的版本 ──────────────────

class StructuredReActAgent(ReActAgent):
    """增强版：显式分离 Thought 和 Action，且 Action 由 Function Calling 驱动"""
    
    def _parse_thought(self, content: str) -> str:
        """从 LLM 回复中提取 Thought"""
        # 当 Function Calling 驱动 Action 时，content 部分自然就是 Thought
        return content or ""
    
    async def run(self, user_input: str) -> str:
        # 与基础版基本相同，但增加了：
        # 1. Thought 验证：确保 Agent 真的思考了才行动
        # 2. 步骤计时：记录每步耗时
        # 3. 自动截断：当 token 接近窗口限制时，压缩历史
        # 4. 安全检查：检测循环和偏离
        ...
```

## 实现的最大挑战

**Function Calling 与自由格式 Thought 的平衡**：使用 OpenAI 的 Function Calling 时，Tool Call 是结构化的，但 Thought 作为 content 字段是自由文本。需要确保 LLM 在 content 中表达了充分的思考，但又不能过长导致 token 浪费。实践中通过提示词控制 Thought 长度（"Keep your Thought concise, 2-3 sentences"）。

## 关键设计决策

- **使用 Function Calling 驱动 Action 而非文本解析**：大幅提升可靠性，避免 Action 格式解析错误
- **tool_call_id 匹配**：正确关联 Tool Call 和 Tool Result，支持多工具并行调用
- **消息结构完整性**：每轮循环追加一对 assistant message + tool message，确保 LLM 看到完整上下文

## 工程优化方向

- 为 `_default_prompt()` 引入模板引擎，支持动态注入工具列表和行为规则
- 在 `run()` 中增加重试逻辑（LLM API 调用失败时自动重试）
- 实现上下文压缩：当消息历史超过 token 阈值时，自动对早期步骤做摘要
- 添加 checkpoint 机制：定期保存执行状态，支持失败恢复

## 适合场景

- 学习 Agent 原理的最直接途径
- 需要完全控制 Agent 行为的定制场景
- 作为更复杂框架（LangGraph、AutoGen）的基础理解
