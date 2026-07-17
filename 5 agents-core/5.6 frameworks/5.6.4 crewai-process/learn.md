# 5.6.4 crewai-process — CrewAI 流程源码

## 简单介绍

CrewAI 是一个多角色 Agent 协作框架。核心模型是"Agent 扮演特定角色，执行分配的任务，组成 Crew 按 Process 协同工作"。其源码的核心是 Agent、Task、Crew、Process 四个实体及其编排逻辑。

## 基本原理

### 核心实体

```
Crew (团队)
  ├── Agents (角色分工)
  │     ├── Agent A: 研究员（扮演研究者角色）
  │     ├── Agent B: 写作者（扮演写作者角色）  
  │     └── Agent C: 审核员（扮演审核者角色）
  ├── Tasks (任务分配)
  │     ├── Task 1: 收集资料 → Agent A
  │     ├── Task 2: 撰写报告 → Agent B
  │     └── Task 3: 审核报告 → Agent C
  └── Process (协作模式)
        ├── Sequential: 顺序执行
        └── Hierarchical: 层级管理
```

### 顺序执行流程

```
Task 1 (Agent A 收集资料)
    │ 输出 → 输入
    ↓
Task 2 (Agent B 撰写报告)
    │ 输出 → 输入
    ↓
Task 3 (Agent C 审核报告)
    │ 输出 → 最终结果
```

### 关键源码结构

```python
class Agent:
    role: str          # 角色定位
    goal: str          # 目标
    backstory: str     # 背景故事
    llm: BaseLLM       # LLM 配置
    tools: List[Tool]  # 可用工具

class Task:
    description: str   # 任务描述
    agent: Agent       # 执行 Agent
    tools: List[Tool]  # 任务特定工具
    output: str        # 任务输出

class Crew:
    agents: List[Agent]
    tasks: List[Task]
    process: Process   # 顺序或层级
    
    def kickoff(self):
        """启动 Crew 执行"""
        # 根据 process 类型编排任务执行
```

## 背景与演进

CrewAI 的设计哲学来源于"角色扮演"——让每个 Agent 扮演一个专业角色，通过角色分工实现协作。这与 LangGraph 的"图编排"哲学不同。

## 核心矛盾

**角色灵活性 vs 角色固化**：
- 角色分工使 Agent 职责清晰，行为可预测
- 但角色也限制了 Agent 的灵活性，跨角色行动困难

## 核心优势

相比 LangGraph，CrewAI 的优势：
- **更上层的抽象**：无需了解图、节点、边等底层概念
- **角色分工自然**：适合模拟人类团队协作
- **快速上手**：几个核心概念即可构建多 Agent 系统

## 局限性

- 执行模型相对固定（顺序/层级），不够灵活
- 复杂条件逻辑需要 hack
- 状态管理不如 LangGraph 精细

## 工程优化

1. Task 描述要清晰包含输出格式要求
2. Agent 的 role/goal/backstory 一致性直接影响协作质量
3. 任务间的上下文传递依赖输出格式设计
4. 混合使用：CrewAI 做高层编排，LangGraph 做底层 Agent

## 场景判断

适合 CrewAI 的场景：
- 角色分明的多 Agent 协作（研究→写作→审核）
- 需要模拟人类团队工作流
- 快速原型开发
