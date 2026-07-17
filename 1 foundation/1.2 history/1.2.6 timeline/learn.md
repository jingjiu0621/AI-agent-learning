# 关键历史节点与里程碑论文

## 简单介绍

AI Agent 从理论概念到工程实践，经历了近 70 年的演进。本文按时间线梳理关键节点和对应的里程碑论文/工作，帮助建立 Agent 发展的全局历史观——了解"从哪来"才能更好地判断"往哪去"。

## 完整时间线

### 1950s-1960s：理论的诞生期

| 年份 | 事件/论文 | 重要性 |
|------|----------|--------|
| 1950 | Turing《Computing Machinery and Intelligence》 | 提出图灵测试，首次将"智能行为"作为可操作概念 |
| 1957 | Bellman 提出动态规划 | MDP 的理论基础，序列决策的数学框架 |
| 1959 | Samuel 的跳棋程序 | 最早的自主学习程序之一，强化学习的前身 |
| 1965 | Feigenbaum 开始 DENDRAL 项目 | 第一个专家系统，知识工程的开始 |
| 1969 | Minsky & Papert《Perceptrons》 | 揭示单层感知器的局限，间接推动符号 AI 研究 |

### 1970s：专家系统的黄金期

| 年份 | 事件/论文 | 重要性 |
|------|----------|--------|
| 1976 | Shortliffe《MYCIN》 | 医疗诊断专家系统，600+规则，准确率超过部分医生 |
| 1977 | Newell & Simon 获图灵奖 | SOAR 认知架构的起点 |
| 1979 | R1/XCON 系统 | DEC 计算机配置系统，第一个商业成功的专家系统 |

### 1980s：Agent 理论的形成

| 年份 | 事件/论文 | 重要性 |
|------|----------|--------|
| 1986 | Minsky《The Society of Mind》 | 多智能体思想的起源 |
| 1986 | Rumelhart 等提出反向传播 | 神经网络重获新生，数据驱动的起点 |
| 1987 | Bratman《Intention, Plans, and Practical Reason》 | BDI 模型的哲学基础 |
| 1988 | Sutton 提出 TD 学习 | 强化学习的核心算法突破 |
| 1989 | Watkins 提出 Q-Learning | 无模型 RL 的突破性算法 |
| 1989 | Brooks 的 Subsumption Architecture | 反应式 Agent 的经典工作 |

### 1990s：Agent 理论系统化与分布式 AI

| 年份 | 事件/论文 | 重要性 |
|------|----------|--------|
| 1995 | Wooldridge & Jennings《Intelligent Agents: Theory and Practice》 | Agent 特性的经典定义（自主/反应/主动/社交） |
| 1995 | Rao & Georgeff 形式化 BDI 模型 | BDI 的计算模型 |
| 1997 | 深蓝战胜卡斯帕罗夫 | 搜索+评估的经典 AI 路线 |
| 1998 | Laird 等 SOAR-8 | SOAR 认知架构的成熟版本 |
| 1999 | Russell & Norvig《A Modern Approach》第一版 | AI 教材的标准化，Agent 成为中心概念 |

### 2000s：数据驱动的兴起

| 年份 | 事件/论文 | 重要性 |
|------|----------|--------|
| 2006 | Hinton 提出深度信念网络 | 深度学习复兴的开端 |
| 2007 | Kaelbling 等 POMDP 综述 | 部分可观测 MDP 的全面理论 |
| 2009 | 李飞飞发布 ImageNet | 大规模数据驱动的催化剂 |

### 2010s：深度学习的黄金期

| 年份 | 事件/论文 | 重要性 |
|------|----------|--------|
| 2013 | Mnih 等 DQN | 深度强化学习的突破，从 Atari 原始像素学习 |
| 2014 | Sutskever 等 Seq2Seq + Attention | Transformer 的前奏 |
| 2015 | 松尾丰等深度强化学习综述 | RL+DL 的系统化 |
| 2016 | Silver 等 AlphaGo | RL + MCTS + DNN 击败围棋世界冠军 |
| 2017 | Vaswani 等《Attention Is All You Need》 | Transformer 架构，LLM 时代的基础 |
| 2018 | Devlin 等 BERT | 预训练语言模型的范式确立 |
| 2019 | Brown 等 GPT-2 | 展示语言模型规模扩展的潜力 |

### 2020s：LLM Agent 时代

| 年份 | 事件/论文 | 重要性 |
|------|----------|--------|
| 2020 | Brown 等 GPT-3 | 涌现 In-Context Learning，零样本任务执行 |
| 2021 | Chen 等 Codex | 代码生成模型，Agent 编程能力的基础 |
| 2022 | OpenAI 发布 ChatGPT | 指令遵循能力爆发，Agent 的控制变得可靠 |
| 2022 | Wei 等 Chain-of-Thought | 多步推理的提示方法，Agent 推理的基础 |
| 2022 | Yao 等 ReAct | 推理+行动协同，最广泛使用的 Agent 模式 |
| 2023 | Shinn 等 Reflexion | 带自我反思的 Agent，引入错误学习机制 |
| 2023 | OpenAI 发布 GPT-4 Function Calling | 工具调用标准化，Agent 工程爆发 |
| 2023 | Long 等 Tree of Thoughts | 多路径搜索推理 |
| 2023 | AutoGPT / BabyAGI 发布 | 开源 Agent 运动，社区级 Agent 实验 |
| 2023 | Park 等 Generative Agents | 多 Agent 社会模拟，Smallville 小镇 |
| 2023 | LangChain / LangGraph 发布 | 第一个广泛采用的 Agent 开发框架 |
| 2024 | Anthropic Claude Tool Use + Computer Use | Agent 从文本扩展到电脑操作 |
| 2024 | Yang 等 SWE-Agent | 编程 Agent 达到 SWE-bench 实用水平 |
| 2024 | Multiple Agents 框架成熟 | CrewAI, AutoGen 进入生产级稳定 |
| 2025 | 长上下文 Agent / Agentic AI | Agent 处理任务复杂度达到新高度 |

## 关键转折点

### 从符号到统计（1990s-2010s）
- 规则驱动 → 数据驱动的范式迁移
- 核心原因：专家系统的知识获取瓶颈不可突破
- 标志：深度学习的成功

### 从单独学习到预训练+微调（2018-2020）
- 从"为每个任务训练模型"到"一个模型适应所有任务"
- 核心原因：Transformer + 大规模预训练的有效性
- 标志：BERT → GPT-3 的演进

### 从模型到 Agent（2022-2023）
- 从"一个模型直接输出答案"到"模型驱动工具调用"
- 核心原因：指令遵循 + 工具使用能力的涌现
- 标志：ChatGPT Plugins, GPT-4 Function Calling

## 历史启示

1. **能力不是计划出来的，是涌现出来的**：Agent 的真正突破来自 LLM 的涌现能力，而非自上而下的设计
2. **工程推动大于理论推动**：ReAct 和 Function Calling 的实际效果远超过同期更"高级"的理论框架
3.  **范式迁移不可预测**：2019 年没有人预测到 2023 年 Agent 会以 LLM 的形式爆发
4. **旧思想在新瓶子中重生**：BDI、SOAR、专家系统等"老"概念在 LLM 时代以新形式回归
5. **基础设施决定应用**：Transformer → LLM → API → Agent 框架 → Agent 应用，每一层都是下一层的基础

## 时间线洞察

回顾历史可以发现几个清晰的模式：
- Agent 的能力爆发间隔在缩短（从几十年 → 几年 → 几个月）
- 从理论到工程落地的周期在加速（BDI 理论到 Agent 框架用了 30 年 → ReAct 论文到广泛采用用了不到 1 年）
- 开源和社区的作用越来越大（AutoGPT → LangChain → 各种 Agent 框架的快速迭代）
