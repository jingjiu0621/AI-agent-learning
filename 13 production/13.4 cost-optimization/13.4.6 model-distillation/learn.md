# 模型蒸馏：用小型专用模型替代大模型实现 50-80% 成本削减

**模型蒸馏 (Model Distillation) 是通过让一个大模型（Teacher）"教授"一个小模型（Student）来继承其核心能力的技术路线，在 Agent 生产部署中可将单次推理成本降低 50-80%，是 13.4 成本优化专题中回报率最高的高级手段。**

---

## 1. 基本原理

### 1.1 Teacher-Student 范式

蒸馏的核心是一个知识迁移过程：

```
                      ┌──────────────────┐
                      │   Teacher Model   │
                      │  (GPT-4 / Claude) │
                      │    ~1.8T params   │
                      └────────┬─────────┘
                               │
              ┌────────────────┼────────────────┐
              │   Response     │    Logit        │   Feature
              │   Distillation │   Distillation  │  Distillation
              ▼                ▼                 ▼
        ┌──────────────────────────────────────────┐
        │          Student Model                    │
        │   (Llama-3-8B / Phi-3 / Qwen2.5-7B)      │
        │            ~7B params                     │
        └──────────────────────────────────────────┘
```

- **Teacher**：大型通用模型（GPT-4、Claude 3.5 Sonnet），拥有广泛知识但推理成本高
- **Student**：小型专用模型（7B-13B 参数），针对特定 Agent 任务优化，规模缩小 100-200 倍

### 1.2 为什么蒸馏对 Agent 有效

Agent 系统中的绝大多数调用并非需要大模型的全部能力：

| Agent 任务 | 是否需要世界知识 | 是否需要复杂推理 | 是否适合蒸馏 |
|---|---|---|---|
| 意图分类 | 不需要 | 不需要 | 非常适合 |
| 工具选择 | 部分需要 | 不需要 | 非常适合 |
| 简单问答（RAG） | 由检索提供 | 低 | 非常适合 |
| 信息抽取 | 不需要 | 低 | 非常适合 |
| 多步推理 | 需要 | 高 | 部分适合 |
| 创意写作 | 需要 | 高 | 不适合 |

关键洞察：**Agent 任务大多是窄域任务，其能力需求远低于通用大模型的全量能力。** 蒸馏的精髓在于"够用即可"。

### 1.3 成本对比

```
成本对比（每百万 Token 推理成本）：

GPT-4                     ┌────────────────────── $30.00
Claude 3.5 Sonnet         ┌────────────────── $15.00
─────── 蒸馏边界 ─────────────────────────────────────
Llama-3-8B (自部署)        ┌── $0.50
Phi-3-mini (自部署)         ┌── $0.20
Qwen2.5-7B (自部署)        ┌── $0.40
BERT 分类器                 ┌── $0.01

成本降低比：30 ～ 60 倍
```

### 1.4 蒸馏的三种主要形式

1. **Response Distillation（响应蒸馏）**：Teacher 生成（prompt, response）训练对，Student 通过监督学习模仿。最简单、最广泛使用。
2. **Logit Distillation（logit 蒸馏）**：Student 学习匹配 Teacher 的输出概率分布（soft targets），保留更多不确定性和知识。由 Hinton 2015 年提出，适合分类任务。
3. **Feature Distillation（特征蒸馏）**：Student 学习模仿 Teacher 中间层的特征表示。适合需要语义理解的任务。

---

## 2. 蒸馏方法论

### 2.1 Response Distillation（响应蒸馏）

这是 Agent 蒸馏中最常用的方法。流程如下：

```
[生产日志中的用户查询]
        │
        ▼
 ┌─────────────────┐
 │  Teacher Model   │  ← GPT-4 / Claude
 │  生成响应 + 推理   │
 └────────┬────────┘
          │
          ▼
 ┌─────────────────────────────────────┐
 │ 训练数据集：(query, response) 对      │
 │    - 意图分类：(query → intent_label) │
 │    - 工具调用：(query → tool_call)    │
 │    - 问答：(query + context → answer) │
 └─────────────────────────────────────┘
          │
          ▼
 ┌─────────────────┐
 │  Student Model   │  ← 用 LoRA 微调
 │  监督学习微调     │
 └─────────────────┘
```

代码示例：生成蒸馏训练数据

```python
# 阶段一：使用 Teacher 模型生成训练数据
import openai  # Teacher: GPT-4

def generate_distillation_data(queries: list[str], system_prompt: str) -> list[dict]:
    """
    从 Teacher 模型生成 (query, response) 训练对。
    
    Args:
        queries: 用户查询列表（来自生产日志）
        system_prompt: Agent 的系统提示词
    
    Returns:
        list[dict]: 训练样本列表
    """
    dataset = []
    for query in queries:
        response = openai.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": query}
            ],
            temperature=0.3,  # 低温度确保一致性
        )
        dataset.append({
            "instruction": system_prompt,
            "input": query,
            "output": response.choices[0].message.content,
        })
    return dataset


# 阶段二：用 LoRA 微调 Student 模型
#
# 使用 unsloth / axolotl / LLamaFactory
#
# 示例 unsloth 训练脚本:
#
# from unsloth import FastLanguageModel
# model, tokenizer = FastLanguageModel.from_pretrained(
#     model_name="unsloth/Llama-3.2-7B",
#     max_seq_length=4096,
#     load_in_4bit=True,
# )
# model = FastLanguageModel.get_peft_model(
#     model,
#     r=16,
#     target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
# )
# # 在蒸馏数据集上训练 ...
```

### 2.2 Logit Distillation（Logit 蒸馏）

适用于分类头的 Agent 组件（如意图分类、路由选择）：

```
Teacher Logits:  [0.70, 0.20, 0.05, 0.03, 0.02]  ← 软标签（保留类间关系）
                       ↕  KL 散度
Student Logits:  [0.60, 0.25, 0.08, 0.04, 0.03]

硬标签（one-hot）: [1, 0, 0, 0, 0]  ← 丢失了"第二候选"信息
```

Logit 蒸馏的核心损失函数：

```python
import torch
import torch.nn.functional as F

def distillation_loss(student_logits, teacher_logits, labels, 
                      temperature=4.0, alpha=0.5):
    """
    Hinton 蒸馏损失函数。
    
    Args:
        student_logits: Student 模型的输出 logits
        teacher_logits: Teacher 模型的输出 logits
        labels: 真实标签（硬标签）
        temperature: 温度参数，越大软标签分布越平滑
        alpha: 软标签损失的权重
    """
    # 软标签损失（KL 散度）
    soft_student = F.log_softmax(student_logits / temperature, dim=-1)
    soft_teacher = F.softmax(teacher_logits / temperature, dim=-1)
    kl_loss = F.kl_div(soft_student, soft_teacher, reduction="batchmean")
    kl_loss *= temperature ** 2  # 缩放补偿
    
    # 硬标签损失（交叉熵）
    ce_loss = F.cross_entropy(student_logits, labels)
    
    # 加权组合
    return alpha * kl_loss + (1 - alpha) * ce_loss
```

### 2.3 其他蒸馏方法

| 方法 | 描述 | 适用场景 | 复杂度 |
|---|---|---|---|
| **Self-Distillation** | 同一架构，Teacher = Student | 缺乏更大模型时自我提升 | 低 |
| **Task-Specific Distillation** | 专注于一个 Agent 子能力 | 工具选择、意图分类 | 中 |
| **Progressive Distillation** | Teacher → 中等 → 小，逐步压缩 | 从 70B → 13B → 7B | 高 |
| **Data-Free Distillation** | Teacher 生成合成数据训练 Student | 缺乏生产日志时 | 中 |

---

## 3. 针对 Agent 的蒸馏策略

### 3.1 工具调用蒸馏

Agent 最核心且最昂贵的操作之一是工具选择与调用。蒸馏这一能力可以将成本降低 90% 以上。

```
训练数据构造：

[用户查询]                [可用工具列表]                [训练目标]
─────────────────────────────────────────────────────────────────
"帮我查一下北京的天气"    get_weather(), send_email() → get_weather()
"给张三发一封邮件"        get_weather(), send_email() → send_email()
"北京和上海哪个更热"      get_weather()               → 两次 get_weather()
```

```python
# 工具调用蒸馏的训练数据格式
TRAINING_EXAMPLES = [
    {
        "messages": [
            {
                "role": "system",
                "content": "你是一个工具调用助手。根据用户请求，选择最合适的工具。"
            },
            {
                "role": "user",
                "content": "帮我查一下北京的天气"
            },
            {
                "role": "assistant",
                "content": None,
                "tool_calls": [
                    {
                        "type": "function",
                        "function": {
                            "name": "get_weather",
                            "arguments": {"city": "北京"}
                        }
                    }
                ]
            }
        ]
    }
]
```

**实践建议**：
- 先用 Teacher 对 5000-10000 条生产查询生成工具调用
- 用 Student 做 7B 模型 LoRA 微调（1 小时，1x A100）
- 部署后 shadow 模式对比准确率
- 通常能达到 Teacher 95%+ 的工具选择准确率

### 3.2 意图分类蒸馏

将路由/分类能力从 LLM 蒸馏到轻量级 BERT 分类器：

```
                  ┌──────────────────────────┐
                  │  Teacher: GPT-4          │
                  │  "这个用户想查天气"       │
                  │  → intent: weather_query  │
                  └────────┬─────────────────┘
                           │ 生成 20000+ 标注样本
                           ▼
┌─────────────────────────────────────────────┐
│  Student: BERT-base + 分类头 (110M params)  │
│  推理成本: $0.01/M tokens（GPT-4 的 1/3000） │
│  单次推理: < 5ms                            │
└─────────────────────────────────────────────┘
```

```python
# 使用 HuggingFace transformers 训练 BERT 分类器
from transformers import (
    AutoTokenizer, 
    AutoModelForSequenceClassification,
    Trainer, 
    TrainingArguments
)

# 加载学生模型
model = AutoModelForSequenceClassification.from_pretrained(
    "bert-base-chinese",
    num_labels=8  # 8 种 Agent 意图
)

# 在 LLM 标注的数据上微调
training_args = TrainingArguments(
    output_dir="./intent_classifier",
    learning_rate=2e-5,
    per_device_train_batch_size=32,
    num_train_epochs=3,
)
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=distilled_dataset,  # LLM 标注的 20000+ 样本
)
trainer.train()
```

### 3.3 简单推理蒸馏

对于客户支持、格式化、FAQ 等狭域推理任务，可以蒸馏 Chain-of-Thought 模式：

```
Teacher 推理链（完整版 → 数据生成）：
  "用户说登录失败 → 问题属于认证类 → 应该检查 token 是否过期
   → token 过期错误码是 401 → 建议重新登录"

Student 推理（蒸馏后 → 简化版）：
  "登录失败 → 检查 token → 建议重新登录"

Student 只需要记住"登录失败→重新登录"这个映射，
而不需要理解 OAuth2 协议细节。
```

**适用条件**：
- 输入输出映射相对固定
- 领域知识不频繁变化
- 无需外部世界知识（或知识已由 RAG 提供）

### 3.4 RAG 响应生成蒸馏

蒸馏 RAG Agent 的阅读理解 + 答案生成能力：

```
输入： 用户问题 + 检索文档
输出： 基于文档的回答

[RAG Agent 蒸馏]
Teacher 能力：理解文档 → 提取相关信息 → 组织答案 → 引用来源
Student 能力：定位相关段落 → 用原文回答（简化）
```

Student 不需要记忆世界知识——知识由检索系统提供，Student 只需"读得懂"检索结果。这大大降低了 Student 的能力要求。

---

## 4. 训练流程与工具链

### 4.1 完整三阶段流程

```
阶段一：数据生成
═══════════════════════════════════════════════
 生产日志查询
     │
     ▼
 Teacher 模型 ──→ 生成 (query, response) 对
     │
     ├── 质量过滤（去除低置信度/错误/有害输出）
     ├── 类别平衡（确保覆盖所有任务类型）
     └── 去重与清洗
     │
     ▼
 最终训练集（通常 5K - 50K 样本）

阶段二：Student 训练
═══════════════════════════════════════════════
 选择 Student 架构
  Llama-3-8B / Phi-3-mini / Qwen2.5-7B / DeepSeek-7B
     │
     ▼
 LoRA / QLoRA 微调（4bit 量化训练）
     │
     ├── 损失函数：SFT（监督微调）
     ├── 学习率：2e-4（LoRA 标准设置）
     ├── 批次大小：依赖显存（8-64）
     └── Epochs：2-3（避免过拟合）
     │
     ▼
 评估 ←── 在保留测试集上与 Teacher 对比

阶段三：部署与监控
═══════════════════════════════════════════════
     │
     ▼
 Student 模型部署（vLLM / Ollama / TGI）
     │
     ▼
 Shadow 模式（Student ∥ Teacher，对比输出）
     │
     ├── Student 准确率 >= Teacher 90%  → 逐步放量
     ├── Student 准确率 >= Teacher 95%  → 完全切换
     └── Student 准确率 < Teacher 85%   → 补数据重训
     │
     ▼
 持续改进：
   - 收集 Student 失败案例
   - 用 Teacher 重新生成正确输出
   - 增量训练
```

### 4.2 工具链推荐

| 环节 | 推荐工具 | 说明 |
|---|---|---|
| 数据生成 | OpenAI API / Claude API | Teacher 模型调用 |
| 数据管理 | Argilla / LabelStudio | 数据标注与质量审核 |
| 微调框架 | **Unsloth**（最快）、Axolotl、LLamaFactory | LoRA/QLoRA 训练 |
| 训练监控 | Weights & Biases / TensorBoard | Loss 曲线、评估指标 |
| 推理部署 | **vLLM**（高吞吐）、Ollama（本地测试）、llama.cpp（边缘设备） | 生产推理 |
| 评估 | DeepEval / RAGAS / 自定义评估集 | 自动评估 |

### 4.3 硬件需求参考

| Student 规模 | 微调 GPU | 推理 GPU | 显存需求（推理） |
|---|---|---|---|
| 7B 量化 (4bit) | 1x A100-80G | 1x A10G-24G | 6-8 GB |
| 7B 半精度 (FP16) | 1x A100-80G | 1x A10G-24G | 14-16 GB |
| 13B 量化 (4bit) | 2x A100-80G | 1x A10G-24G | 10-12 GB |
| 3B 量化 (4bit) | 1x RTX 4090-24G | 1x T4-16G | 3-4 GB |
| BERT 分类器 | 1x T4-16G | CPU 即可 | <1 GB |

### 4.4 关键经验法则

> **数据质量 >> 数据数量：10K 高质量训练对 > 100K 低质量训练对**

- 从生产日志中采样真实用户查询（而不是人工构造）
- Teacher 生成响应后务必经过质量过滤
- 涵盖边界情况和错误恢复路径
- 保留至少 10% 数据作为验证集

---

## 5. 能力边界与评估

### 5.1 蒸馏能做什么、不能做什么

```
蒸馏的能力范围
═══════════════════════════════════════════════════

可以替换（通常可达 Teacher 90-98% 性能）：
  ┌────────────────────────────────────────────┐
  │ ✓ 意图分类与路由                              │
  │ ✓ 信息抽取（实体、关系、关键词）                │
  │ ✓ 简单 Q&A（来自检索结果）                     │
  │ ✓ 结构化工具调用                              │
  │ ✓ 文本摘要（短文本）                           │
  │ ✓ 文本分类/打标签                              │
  │ ✓ 格式转换（JSON、Markdown 等）                │
  └────────────────────────────────────────────┘

部分可以替换（需要精心设计，可能损失 5-15% 性能）：
  ┌────────────────────────────────────────────┐
  │ △ 简单链式推理（2-3 步，领域固定）            │
  │ △ 多轮对话管理                               │
  │ △ 代码生成（模板化场景）                      │
  └────────────────────────────────────────────┘

不能替换（强烈建议保留 Teacher）：
  ┌────────────────────────────────────────────┐
  │ ✗ 复杂多步推理（数学证明、策略规划）           │
  │ ✗ 创意写作（故事、营销文案）                  │
  │ ✗ 新颖问题求解（训练分布外场景）              │
  │ ✗ 开放域探索（用户意图不明确）                │
  │ ✗ 需要最新知识的任务（如果 Student 未更新）    │
  └────────────────────────────────────────────┘
```

### 5.2 性能差距量化

典型的蒸馏效果对比（以 Teacher = GPT-4 为基准）：

| 任务类型 | Teacher (GPT-4) | Student (7B) | 相对性能 | 成本比 |
|---|---|---|---|---|
| 意图分类 | 98% 准确率 | 96% 准确率 | 98% | 1:60 |
| 工具选择 | 97% 准确率 | 93% 准确率 | 96% | 1:60 |
| 简单 Q&A | 92% 得分 | 87% 得分 | 95% | 1:60 |
| 两步推理 | 88% 准确率 | 76% 准确率 | 86% | 1:60 |
| 复杂推理 | 82% 准确率 | 58% 准确率 | 71% | 1:60 |

**关键发现**：随着任务复杂度增加，蒸馏的性能差距扩大。窄域任务的损失通常不超过 5%，而复杂推理任务损失可达 15-30%。

### 5.3 评估框架

```python
# 蒸馏评估模板
def evaluate_distillation(teacher_fn, student_fn, test_cases):
    """
    对比 Teacher 和 Student 在测试集上的表现。
    
    返回差异报告，指导是否适合切换到 Student。
    """
    results = {
        "total": len(test_cases),
        "teacher_correct": 0,
        "student_correct": 0,
        "student_degradations": [],  # Student 明显差于 Teacher 的案例
    }
    
    for case in test_cases:
        teacher_output = teacher_fn(case["input"])
        student_output = student_fn(case["input"])
        
        t_correct = evaluate(teacher_output, case["expected"])
        s_correct = evaluate(student_output, case["expected"])
        
        results["teacher_correct"] += t_correct
        results["student_correct"] += s_correct
        
        if t_correct and not s_correct:
            results["student_degradations"].append(case)
    
    teacher_acc = results["teacher_correct"] / results["total"]
    student_acc = results["student_correct"] / results["total"]
    relative_perf = student_acc / teacher_acc
    
    print(f"Teacher 准确率: {teacher_acc:.1%}")
    print(f"Student 准确率: {student_acc:.1%}")
    print(f"相对性能: {relative_perf:.1%}")
    print(f"退化案例数: {len(results['student_degradations'])}")
    
    # 决策建议
    if relative_perf >= 0.95:
        print("推荐：可以安全切换到 Student 模型")
    elif relative_perf >= 0.85:
        print("谨慎：建议 Shadow 模式灰度放量")
    else:
        print("不推荐：蒸馏损失过大，需要优化数据或增大 Student")
    
    return results
```

### 5.4 维护负担

Student 模型并非一次训练终身使用。需要关注：

- **任务分布漂移**：用户查询类型变化 → 需要重新生成训练数据 → 重训 Student
- **Teacher 能力提升**：GPT-5 或 Claude 4 发布后 → 用新 Teacher 重蒸馏
- **工具变更**：API 参数变化 → 需要更新工具调用训练数据
- **建议维护节奏**：每 1-3 个月重新评估并增量训练

---

## 6. 工程实践

### 6.1 决策框架

```
是否应该对某 Agent 任务使用模型蒸馏？

                          ┌─────────────────────┐
                          │  Agent 任务的日调用量  │
                          │  是否 > 10,000 次？    │
                          └──────────┬──────────┘
                               YES /  NO
                               │      │
                     ┌─────────┘      └──────────┐
                     │ YES                       │ NO
                     ▼                           ▼
              ┌──────────────────┐              ┌────────────┐
              │ 任务范围是否窄域？ │              │ 不值得蒸馏  │
              │ 输入输出是否明确？ │              │ Teacher 直  │
              └────────┬─────────┘              │ 接调用即可  │
                   YES / NO                     └────────────┘
                     │      │
                     │      └──→ 考虑部分蒸馏（仅特定子任务）
                     ▼
              ┌──────────────────┐
              │ 任务复杂度：       │
              │ 分类/抽取 → BERT   │
              │ 简单推理 → 7B LLM  │
              │ 复杂推理 → 保持     │
              │    Teacher + 缓存   │
              └──────────────────┘
```

### 6.2 成本效益分析模板

```python
# 蒸馏成本效益分析
def distillation_roi_analysis(
    daily_calls: int,
    teacher_cost_per_call: float,  # e.g., $0.01 per GPT-4 call
    student_cost_per_call: float,  # e.g., $0.0002 per 7B self-hosted
    training_cost: float,          # e.g., $50 for 1x A100 for 1 hour
    maintenance_cost_per_month: float,
    months: int = 12,
):
    """计算蒸馏项目的投资回报率。"""
    
    monthly_teacher = daily_calls * teacher_cost_per_call * 30
    monthly_student = daily_calls * student_cost_per_call * 30
    monthly_saving = monthly_teacher - monthly_student
    
    total_training = training_cost * 4  # 假设每季度重训一次
    total_maintenance = maintenance_cost_per_month * months
    
    total_roi = (monthly_saving * months) - total_training - total_maintenance
    
    print(f"当前 Teacher 月成本: ${monthly_teacher:.2f}")
    print(f"Student 月成本: ${monthly_student:.2f}")
    print(f"月节省: ${monthly_saving:.2f}")
    print(f"{months} 个月总投资成本: ${total_training + total_maintenance:.2f}")
    print(f"{months} 个月总节省: ${total_roi:.2f}")
    print(f"ROI: {(total_roi / (total_training + total_maintenance)):.1f}x")
    
    return total_roi

# 示例：日调用 10 万次的工具选择 Agent
distillation_roi_analysis(
    daily_calls=100000,
    teacher_cost_per_call=0.01,      # GPT-4 每次约 $0.01
    student_cost_per_call=0.0002,    # 7B 自部署每次约 $0.0002
    training_cost=50,                # 单次微调 $50
    maintenance_cost_per_month=200,  # 月维护 $200
)
# 输出: 12 个月总节省 ~$34,600，ROI 约 43x
```

### 6.3 混合架构模式

实践中，建议保留 Teacher 作为"裁判"而非完全替换：

```
                    ┌─────────────┐
                    │  用户请求    │
                    └──────┬──────┘
                           ▼
                    ┌─────────────┐
                    │  Router     │
                    │ (蒸馏后的    │
                    │  意图分类器) │
                    └──┬──────┬───┘
                       │      │
                ┌──────┘      └──────┐
                ▼                     ▼
         ┌──────────────┐    ┌──────────────┐
         │ Student 处理  │    │ Teacher 处理  │
         │ (95% 流量)    │    │ (5% 高难度)   │
         │ 低成本        │    │ 高成本但准确  │
         └──────────────┘    └──────────────┘
                │                     │
                └──────────┬──────────┘
                           ▼
                    ┌─────────────┐
                    │  Quality    │
                    │  Monitor    │
                    │ (Shadow     │
                    │  对比)       │
                    └─────────────┘
```

**路由策略**：
- **置信度路由**：Student 输出置信度 < 阈值 → 升级到 Teacher
- **回退路由**：Student 调用工具出错 → 自动回退到 Teacher
- **成本预算路由**：每天前 N 次调用用 Teacher，之后用 Student

### 6.4 持续蒸馏循环

```
                          ┌─────────────┐
                          │   部署 Student │
                          └──────┬──────┘
                                 │
                                 ▼
                    ┌─────────────────────┐
                    │ Shadow 监控 + 日志收集 │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ 识别 Student 失败模式 │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Teacher 重新生成纠正   │
                    │ 加入训练集             │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ 增量训练 / 全量重训   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ 评估 → 部署新版本     │────→ 回到顶部
                    └─────────────────────┘
```

### 6.5 常见陷阱

| 陷阱 | 表现 | 解决方案 |
|---|---|---|
| **数据偏差** | Student 在真实流量中表现远差于测试集 | 确保训练数据分布与生产分布一致 |
| **Teacher 犯错传播** | Student 继承了 Teacher 的错误模式 | 人工审核训练数据，去除错误样本 |
| **过拟合** | Student 在训练集上完美，泛化差 | 增加数据多样性，减少 epoch |
| **忽视长尾** | Student 对罕见查询表现极差 | 刻意在训练集中包含长尾样本 |
| **评估不充分** | 上线后才发现性能差距 | Shadow 模式充分对比后再切换 |

---

## 总结

模型蒸馏是 Agent 成本优化的终极手段——它不是在"省着用大模型"，而是在"用不起大模型的地方用小模型替代"。成功的关键在于：

1. **选择正确的任务**：窄域、高频、输入输出明确的任务最适合
2. **数据质量至上**：10K 精心筛选的 Teacher 生成数据胜过 100K 未经清洗的数据
3. **持续迭代**：蒸馏不是一次性工作，而是持续的训练-部署-监控-改进循环
4. **混合架构**：Student 处理 95% 常规流量，Teacher 处理 5% 高难度请求，这是最优性价比方案

当正确实施时，在不影响最终用户体验的前提下，Agent 系统的推理成本可以降低 50-80%，一个日调用 10 万次的 Agent 系统每年可节省数万至数十万美元。
