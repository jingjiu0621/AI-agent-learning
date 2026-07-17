# 11.1.4 adaptability — 适应性：新场景 / 跨领域迁移

## 简单介绍

适应性衡量 Agent 在面对未见过的新场景、新领域或新任务时的表现。一个在测试集上表现优异的 Agent，换到真实场景就失灵——这是最常见的"实验室效果 vs 实际效果"落差。适应性评估回答一个关键问题：**Agent 学到的是真正的能力，还是只是在熟悉的数据集上过拟合了？**

## 基本原理

### 适应性的三个层次

```
领域适应性 ─── 从未见过的知识领域能否正常工作？
  ├── 跨领域：从"科技新闻问答"迁移到"医疗咨询"
  ├── 跨语言：从中文迁移到英文、日文
  └── 跨文化：不同地区的行为规范和表达习惯

任务适应性 ─── 从未见过的任务类型能否处理？
  ├── 新工具：Agent 从未用过的工具能否正确调用
  ├── 新流程：从未见过的多步工作流
  └── 新约束：从未见过的规则和限制条件

环境适应性 ─── 环境变化时能否快速调整？
  ├── 知识库内容更新
  ├── 工具接口版本变更
  ├── 模型底层升级（GPT-4 → GPT-4.1）
  └── 系统配置变化
```

## 背景

传统 ML 模型的适应性通过"领域适应"（Domain Adaptation）和"迁移学习"（Transfer Learning）来评估——在源域训练、目标域测试，看性能衰减了多少。但在 Agent 场景下，适应性的维度更丰富：

- Agent 不仅要适应新的数据分布，还要适应新的工具、新的交互模式
- Agent 可以通过 Prompt 调整快速适配，不需要重新训练
- Agent 的适应性不仅取决于模型能力，还取决于 Prompt 设计和工具编排

**适应性差的核心表现**：在开发集上效果很好，但一遇到真实用户的新问题就表现不佳。这是因为测试集和真实场景之间存在"分布偏移"（Distribution Shift）。

## 核心矛盾

**泛化能力 vs 专项优化的权衡。** 一个高度专精于特定场景的 Agent（比如专门处理退款流程的客服 Agent）在垂直场景上表现最佳，但换个场景就完全失效。反之，一个高度通用的 Agent 什么都能做，但每个场景都做不到最好。适应性的本质是在广度和深度之间寻找平衡。

## 适应性评估方法

### 1. 跨领域泛化测试

```python
class CrossDomainTest:
    """跨领域适应性测试"""
    
    DOMAINS = [
        {
            "name": "科技",
            "train_tasks": [
                "解释什么是微服务架构",
                "比较 React 和 Vue 的异同"
            ],
            "test_tasks": [
                "解释什么是量子纠缠（从未出现过的子领域）",
                "比较 TCP 和 UDP 的优劣"
            ]
        },
        {
            "name": "医疗",
            "train_tasks": [],  # 零样本：从未训练过医疗数据
            "test_tasks": [
                "感冒和流感有什么区别？",
                "BMI 大于30属于什么级别？"
            ]
        },
        {
            "name": "法律",
            "train_tasks": [],  # 零样本
            "test_tasks": [
                "合同中的不可抗力条款通常包含哪些内容？",
                "著作权保护的期限是多久？"
            ]
        }
    ]
    
    def evaluate_domain_adaptation(self, agent, 
                                    train_domain: str,
                                    test_domain: str) -> dict:
        """评估从 train_domain 到 test_domain 的迁移能力"""
        
        # 训练领域效果基准
        train_tasks = self._get_tasks(train_domain, "train")
        train_scores = []
        for task in train_tasks:
            result = agent.run(task)
            train_scores.append(self._score_task(task, result))
        
        # 测试领域效果
        test_tasks = self._get_tasks(test_domain, "test")
        test_scores = []
        for task in test_tasks:
            result = agent.run(task)
            test_scores.append(self._score_task(task, result))
        
        # 计算适应系数
        train_avg = sum(train_scores) / len(train_scores)
        test_avg = sum(test_scores) / len(test_scores)
        
        adaptation_coefficient = test_avg / max(train_avg, 0.01)
        
        return {
            "source_domain": train_domain,
            "target_domain": test_domain,
            "source_performance": train_avg,
            "target_performance": test_avg,
            "adaptation_coefficient": adaptation_coefficient,
            # > 0.8 → 迁移良好
            # 0.5-0.8 → 部分迁移
            # < 0.5 → 需要领域特定优化
            "evaluation": "good" if adaptation_coefficient > 0.8
                     else "partial" if adaptation_coefficient > 0.5
                     else "poor"
        }
```

### 2. 新工具适应测试

```python
class NewToolAdaptationTest:
    """新工具适应能力测试"""
    
    def test_new_tool_adoption(self, agent, 
                                familiar_tools: list,
                                new_tool_spec: dict) -> dict:
        """
        测试 Agent 能否在给定新工具规格的情况下
        正确理解并使用从未用过的新工具
        
        new_tool_spec = {
            "name": "weather_alert_api",
            "description": "获取天气预警信息",
            "parameters": {
                "region": "string, 地区编码",
                "alert_type": "enum: storm/flood/drought",
                "severity": "integer 1-5"
            }
        }
        """
        # 给 Agent 配置新工具
        agent.add_tools(familiar_tools + [new_tool_spec])
        
        test_cases = [
            {
                "task": "查询广东省的台风预警",
                "expected_tool": "weather_alert_api",
                "expected_params": {"region": "广东", "alert_type": "storm"}
            },
            {
                "task": "哪些地区有红色洪水预警？",
                "expected_tool": "weather_alert_api",
                "expected_params": {"alert_type": "flood", "severity": 5}
            }
        ]
        
        results = []
        for case in test_cases:
            agent.run(case["task"])
            used_tool = self._get_last_tool_call(agent)
            params_match = self._check_params(
                used_tool.get("parameters", {}),
                case["expected_params"]
            )
            
            results.append({
                "task": case["task"],
                "tool_selected_correctly": 
                    used_tool.get("name") == case["expected_tool"],
                "parameters_correct": params_match,
                "overall_success": 
                    used_tool.get("name") == case["expected_tool"] 
                    and params_match
            })
        
        tool_adaptation = sum(1 for r in results if r["overall_success"])
        
        return {
            "new_tool_name": new_tool_spec["name"],
            "tool_adaptation_rate": tool_adaptation / len(test_cases),
            "detail": results
        }
```

### 3. 零样本 / 少样本适应测试

```python
class FewShotAdaptationTest:
    """零样本和少样本适应能力测试"""
    
    def __init__(self):
        self.shot_levels = [0, 1, 3, 5]  # 0-shot, 1-shot, 3-shot, 5-shot
    
    def test_adaptation_curve(self, agent, 
                               novel_task: str,
                               examples: list) -> dict:
        """
        测试在不同样本量下的适应能力
        examples: [{"input": ..., "output": ...}, ...]
        """
        results = {}
        for n_shots in self.shot_levels:
            # 构建包含 n 个示例的 Prompt
            prompt = self._build_prompt(
                novel_task, examples[:n_shots])
            
            # 运行多次取平均
            scores = []
            for _ in range(3):  # 3 次运行取稳定结果
                output = agent.run(prompt)
                score = self._score_output(novel_task, output)
                scores.append(score)
            
            results[f"{n_shots}_shot"] = {
                "avg_score": sum(scores) / len(scores),
                "scores": scores
            }
        
        # 计算"样本效率"
        zero_shot = results["0_shot"]["avg_score"]
        five_shot = results["5_shot"]["avg_score"]
        
        sample_efficiency = (five_shot - zero_shot) / 5
        # 每增加一个样本，性能提升多少
        
        return {
            "results": results,
            "zero_shot_performance": zero_shot,
            "five_shot_performance": five_shot,
            "adaptation_gap": five_shot - zero_shot,
            # gap < 0.1 → 零样本已经很好，Few-shot 帮助不大
            # gap > 0.3 → Agent 严重依赖示例，零样本能力弱
            "sample_efficiency": sample_efficiency,
            "is_zero_shot_capable": zero_shot > 0.7
        }
```

## 影响适应性的关键因素

```
适应性强 ←──────────────────────────────────→ 适应性弱

模型能力强 (GPT-4)                       模型能力弱 (小模型)
Prompt 工程设计好                        Prompt 工程粗糙
任务描述清晰                             任务定义模糊
与训练数据分布接近                        与训练数据分布差异大
工具接口规范                            工具接口不标准
有清晰示例                              没有示例
```

## 实现挑战

### 1. 适应性的量化困难

"适应性"不像成功率那样有自然定义。一个 Agent 在医疗领域表现差，是因为不适应还是因为医疗知识本身就有难度？区分"领域不适应"和"任务太难"需要精心设计的对照组实验。

### 2. 测试数据的污染

LLM 的训练数据可能已经包含了所谓的"新领域"信息。实际上没有真正"零样本"的测试——因为模型可能已经在训练数据中见过无数的医疗问答对。

```python
# 应对数据污染的策略
def build_contamination_free_test(agent, domain: str) -> list:
    """
    构建相对"干净"的适应性测试集
    策略：使用模型训练截止日期后出现的新概念
    """
    # 1. 使用最新知识（模型训练后的事件）
    # 2. 使用专有数据（内部文档，不在公网）
    # 3. 使用组合型问题（需要推理而非记忆）
    # 4. 使用自定义工具和 API
    return contamination_free_cases
```

### 3. 适应的"代价"测量

Agent 适应新场景可能需要 Prompt 调整、工具重新配置、Few-shot 示例准备。适应性评估不仅要看最终效果，还要看适应过程的**代价**：

```
完全适应 = 修改 Prompt + 添加工具 + 准备示例 + 测试验证
         = 投入 2-3 小时人工

部分适应 = 只改 Prompt + 用现成工具
         = 投入 30 分钟

零适应 = 直接用原来的 Agent 跑新任务
       = 0 投入
```

## 能力边界

| 能做 | 不能做 |
|------|--------|
| 衡量 Agent 在未见过场景的基准表现 | 预测在完全陌生领域的表现 |
| 评估 Few-shot 对适应的提升效果 | 自动确定需要多少样本才能适应 |
| 比较不同 Agent 的适应能力 | 让 Agent 自己意识到"我不适应这个场景" |
| 发现跨领域的性能衰减 | 无需人工干预的零成本适应 |

## 与其他指标的关系

- **鲁棒性**：鲁棒性关注已知场景下的抗干扰能力，适应性关注新场景的泛化能力
- **可靠性**：可靠性和适应性通常成反比——高度专门化的 Agent 更可靠但适应性差
- **成功率**：适应性是"跨场景的成功率"，是成功率的泛化版本
- **综合评分**：适应性好的 Agent 在综合评分中表现出更稳定的跨场景性能

## 工程优化方向

1. **Prompt 模板化**：将场景特定部分参数化，新场景只需更换参数
2. **工具发现机制**：Agent 自动扫描可用工具并理解其用途
3. **Few-shot 示例库**：预置多场景示例，新场景自动检索最相关的
4. **自适应 Prompt 生成**：让 Agent 自己生成适应新场景的 Prompt
5. **迁移学习基准**：定期在未见过的领域上测试，追踪适应性的变化趋势

## 确定适用场景

适应性评估对以下场景尤为重要：
- 多租户/多客户 Agent（每个客户场景不同）
- 通用型 Agent（非垂直专用）
- 面向长尾需求的 Agent
- 快速迭代中的 Agent（每次迭代都可能引入新场景）
- 跨地域/跨文化部署的 Agent
