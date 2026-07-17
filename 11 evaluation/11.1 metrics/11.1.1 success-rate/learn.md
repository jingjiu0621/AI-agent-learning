# 11.1.1 success-rate — 任务成功率 / 目标达成率

## 简单介绍

任务成功率是 Agent 评估最直观、最核心的指标：给定一个任务，Agent 是否成功地完成了它？看似简单的问题，在 Agent 场景下却充满歧义——"成功"的定义取决于任务类型、评判标准和评估粒度。

## 基本原理

### 成功率的定义变体

```
原始成功率 = 成功完成的任务数 / 总任务数 × 100%

加权成功率 = Σ(任务 i 的权重 × 成功得分_i) / Σ 权重 × 100%

部分成功率 = Σ(子任务完成度_i) / 子任务总数 × 100%
```

### 三种成功率评估粒度

```
┌──────────────────────────────────────────────────────────┐
│  单步成功率                                                │
│  Agent 的每一步是否正确地执行了                             │
│  例：工具 A 被正确调用 → 1分，参数错误 → 0分               │
│  用途：诊断哪个环节最薄弱                                   │
├──────────────────────────────────────────────────────────┤
│  任务成功率                                                │
│  整个任务是否完成了目标                                     │
│  例：用户要订机票 → 最终订到了吗？                          │
│  用途：衡量端到端效果                                       │
├──────────────────────────────────────────────────────────┤
│  会话成功率                                                │
│  一次完整对话中所有请求的综合成功情况                         │
│  例：客服对话中，用户的所有问题是否都解决了                   │
│  用途：多轮交互场景                                         │
└──────────────────────────────────────────────────────────┘
```

## 背景

在传统软件中，成功率是二元结果：API 返回 200 就是成功，500 就是失败。但在 Agent 的世界里，成功是模糊的：

- 用户说"帮我安排下周的会议日程"，Agent 只安排了周一和周三的会，算成功吗？
- Agent 最终做对了，但中间调用错了工具然后自己纠正了，算成功还是失败？
- Agent 只完成了任务的 80%，但这 80% 已经给用户提供了价值，怎么算？

## 核心矛盾

**任务成功定义的主观性 vs 评估指标的客观性要求。** 同样的任务，不同的人对"成功"的标准不同。一个严谨的用户可能要求完全精确，另一个用户觉得"差不多就行"。

## 主流方案

### 1. 人工评分（Gold Standard）

最可靠的方式：由人工标注者对 Agent 的输出进行评分。

```python
# 人工评分流程示例
class HumanEvaluation:
    """人工评估任务完成情况"""
    
    def __init__(self, criteria: dict):
        self.criteria = criteria  # 评分标准
        """
        {
            "completeness": "任务是否100%完成",
            "accuracy": "结果是否正确",
            "quality": "输出质量是否满足要求"
        }
        """
    
    def rate_task(self, task_input: str, agent_output: str, 
                  expected_output: str = None) -> dict:
        """
        人工评分接口：返回评分结果
        实际使用时需要人工介入
        """
        # 在评估平台中展示给标注者
        return {
            "task_completed": True / False,
            "completeness_score": 0.85,  # 0-1
            "accuracy_score": 0.9,
            "quality_score": 0.8,
            "notes": "任务基本完成，但缺少对异常情况的处理"
        }
```

### 2. LLM-as-Judge 自动评分

让一个 LLM 作为评委，判断 Agent 是否成功完成了任务。

```python
import json

def llm_success_eval(task: str, agent_trajectory: list, 
                     final_output: str) -> dict:
    """使用 LLM 评估任务是否成功完成"""
    
    eval_prompt = f"""
你是一个严格的 Agent 任务评估员。请判断 Agent 是否成功完成了用户任务。

## 任务定义
{task}

## Agent 执行轨迹
{json.dumps(agent_trajectory, indent=2, ensure_ascii=False)}

## 最终输出
{final_output}

## 评估标准
1. 最终结果是否解决了用户的原始需求？
2. Agent 的执行路径是否合理？
3. 是否有不必要的多余步骤？
4. 输出质量是否达到专业水平？

请按以下 JSON 格式输出：
{{
    "success": true/false,
    "confidence": 0.0-1.0,
    "completion_percentage": 0-100,
    "reasoning": "简要解释评分理由",
    "issues": ["问题1", "问题2"]
}}
"""
    # 调用 LLM 获取评分
    # response = llm.invoke(eval_prompt)
    return json.loads(response)
```

### 3. 自动化验证（可程序化判断的任务）

对于有明确正确答案的任务，可以用自动化脚本验证。

```python
def auto_verify_success(task_type: str, input_data: dict, 
                        agent_result: dict) -> bool:
    """自动化验证任务是否成功"""
    
    if task_type == "calculator":
        # 数学计算：验证结果
        expected = eval(input_data["expression"])
        return abs(agent_result["value"] - expected) < 1e-6
    
    elif task_type == "data_query":
        # 数据查询：验证返回字段
        required_fields = input_data["required_fields"]
        returned_fields = set(agent_result["fields"])
        return required_fields.issubset(returned_fields)
    
    elif task_type == "code_generation":
        # 代码生成：验证编译/测试通过
        test_results = run_tests(agent_result["code"])
        return test_results.passed == test_results.total
    
    elif task_type == "information_extraction":
        # 信息抽取：验证关键实体
        expected_entities = set(input_data["entities"])
        extracted = set(agent_result["entities"])
        # 宽松匹配：F1 > 0.8 算成功
        precision = len(expected_entities & extracted) / len(extracted)
        recall = len(expected_entities & extracted) / len(expected_entities)
        f1 = 2 * precision * recall / (precision + recall)
        return f1 > 0.8
```

## 实现挑战

### 1. 任务成功的主观性

不同评判者对"成功"的判断一致性（Inter-rater Reliability）是核心挑战。

```
Cohen's Kappa 系数 < 0.6 → 评分标准需要重新定义
                 0.6-0.8 → 可接受的一致性
                 > 0.8   → 优秀的一致性
```

### 2. 部分成功的量化

```
Agent 做了 5 步：
  ✅ 步骤 1: 理解用户意图
  ✅ 步骤 2: 查询数据库
  ❌ 步骤 3: 数据格式转换（出错但被步骤4掩盖）
  ✅ 步骤 4: 生成回答（用了错误的数据但凑巧对了）
  ✅ 步骤 5: 输出最终结果

问题：最终结果正确，但中间有错误步骤被掩盖。
解决方案：过程评估 + 结果评估双通道。
```

### 3. 任务难度的差异

同样的成功率，简单任务和复杂任务的意义完全不同：

```python
def adjusted_success_rate(raw_rate: float, 
                          task_difficulty: float) -> float:
    """
    根据任务难度调整成功率
    task_difficulty: 0.0(最简单) ~ 1.0(最难)
    """
    # 难度越高，相同原始成功率的调整后分数越高
    adjusted = raw_rate * (1 + task_difficulty) / 2
    return min(adjusted, 1.0)
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 衡量任务级完成情况 | 区分"真正理解了"还是"凑巧对了" |
| 评估端到端效果 | 自动定位失败原因（需要 Trace 配合） |
| 跨任务对比 | 反映真实用户体验（满意度不等同于成功） |
| 趋势追踪 | 捕捉细微质量差异（85% 和 87% 可能无意义） |

## 与其他指标的关系

- **效率指标**：高成功率 + 低效率 = 不错的 Agent 但太慢
- **鲁棒性指标**：高成功率 + 低鲁棒性 = 只在理想环境下表现好
- **可靠性指标**：高成功率 + 低可靠性 = 偶尔表现极好但不稳定
- **综合评分**：成功率通常是加权综合评分中权重最大的维度

## 工程优化方向

1. **分层成功率追踪**：分别追踪单步、任务、会话三个层级，快速定位瓶颈
2. **难度分层报告**：将任务按难度分层统计，避免被简单任务拉高平均数
3. **失败模式分类**：每次失败时记录失败原因（LLM 错误 / 工具错误 / 规划错误）
4. **置信度校准**：让 Agent 对自己输出的置信度与真实成功率对齐
5. **用户确认闭环**：最终用户确认"这是你想要的吗？"作为真实反馈

## 确定适用场景

任务成功率最适合：
- 有明确目标的 Agent 任务（订票、查询、计算）
- 需要上线门禁的场景
- 长期效果追踪

不适合：
- 开放式创作任务（写诗、头脑风暴）
- 需要人类主观判断的任务
- Agent 行为探索性强的场景
