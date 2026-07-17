# 10.5.6 Post-Hoc Analysis — 事后分析可视化

## 简单介绍

Post-Hoc Analysis Visualization（事后分析可视化）在 Agent 执行完成后，从完整轨迹数据中提取洞察——**这次的执行表现如何？相比之前是进步还是退步？有哪些重复出现的问题？** 如果说实时可视化关注"当下"，事后分析关注"历史"和"模式"。

## 基本原理

事后分析的三个层次：

```
Level 1: 单次执行分析
  ┌─ 执行摘要
  │   ├── 总步数: 8
  │   ├── 总耗时: 12.3s
  │   ├── Token 消耗: 8,500 (成本: $0.052)
  │   ├── 工具调用: 5 次 (1 次失败)
  │   └── 质量评分: 0.85 (LLM-as-Judge)
  │
  ├─ 延迟分解瀑布图
  │   LLM:  ████████████████ 5.2s (42%)
  │   Tool: █████████████   4.8s (39%)
  │   Proc: ████            2.3s (19%)
  │
  └─ 步骤时间线
      Step 1 [██] Step 2 [████] Step 3 [██] ...

Level 2: 多执行对比分析
  ┌─ 延迟趋势图（近 24 小时 P50/P95/P99）
  ├─ 成功率趋势图（每小时的任务成功率）
  ├─ Token 消耗趋势图（每小时的 Token 消耗总量）
  └─ 错误类型分布饼图

Level 3: 回归/优化分析
  ┌─ 版本对比（V1 vs V2 Agent 的性能和效果对比）
  ├─ A/B 测试结果（Prompt A vs Prompt B 的对比）
  ├─ Agent 行为变更检测（新版本与旧版本的行为差异）
  └─ 慢执行聚类分析（自动识别"最慢的 10% 请求"的共同特征）
```

```python
class PostHocAnalyzer:
    """事后分析引擎"""

    def analyze_execution(self, trace_data: dict) -> ExecutionReport:
        return ExecutionReport(
            summary=self._generate_summary(trace_data),
            latency_waterfall=self._build_waterfall(trace_data),
            step_timeline=self._build_timeline(trace_data),
            token_breakdown=self._analyze_tokens(trace_data),
            error_clusters=self._find_error_patterns(trace_data),
            quality_score=self._assess_quality(trace_data),
        )

    def compare_runs(self, run_a: dict, run_b: dict) -> ComparisonReport:
        return ComparisonReport(
            latency_diff=run_b.latency - run_a.latency,
            token_diff=run_b.tokens - run_a.tokens,
            step_diff=run_b.steps - run_a.steps,
            tool_call_diff=self._compare_tool_calls(run_a, run_b),
        )

    def find_regression_patterns(self, recent_runs: list[dict]) -> list[Pattern]:
        """从最近的运行记录中寻找回归趋势"""
        # 检测延迟是否逐渐增加
        # 检测错误率是否上升
        # 检测某类工具的调用次数是否有异常变化
```

## 背景

事后分析在传统软件中对应"性能分析"和"APM"。但在 Agent 系统中，事后分析的范围更广——不仅要分析"快慢"，还要分析"对错"。"对错"的分析需要结合评估系统（Module 11）。

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 不分析 | Agent 执行完就不管了 | 每次都从零开始优化，没有积累和改进 |
| 凭印象评估 | 根据"感觉"判断 Agent 好坏 | 不客观、不精确，无法量化改进 |
| 手动分析 | 手动从日志中提取数据到 Excel | 费时、容易出错、难以持续 |
| 只看平均值 | 只关注平均延迟和总 Token | 丢失了分布和长尾信息 |

## 当前主流优化方向

1. **自动归因分析**——当某次执行"特别慢"或"特别差"时，自动分析原因：是 LLM 调用慢、工具调用多、还是 Agent 陷入了循环？
2. **执行聚类**——将大量执行记录自动聚类（按任务类型、Agent 版本、输入特征等），发现不同"族群"的共性和差异
3. **行为漂移检测**——检测 Agent 行为是否随 LLM 模型版本更新而"漂移"（同一 Prompt 在今天和一个月前的表现差异）
4. **成本效益分析（ROI）**——将 Token 消耗、执行延迟、任务成功率等指标关联到"业务价值"（如用户满意度），回答"Agent 系统到底值不值"
5. **自动报告生成**——每次重要变更（Prompt 更新、模型切换）后自动生成 A/B 报告，量化变更影响

## 实现的最大挑战

1. **数据量大**——生产环境的 Agent 每天可能产生数百万条执行记录，事后分析需要高效的数据仓库和查询引擎
2. **对比的公平性**——A/B 测试中，两个版本的 Agent 可能处理了不同的请求（复杂度不同），直接对比指标不公平，需要分层对比
3. **原因的归因**——"这次延迟高了"的原因可能是 LLM 慢、工具慢、Agent 步骤多、或者输入复杂，多重因素叠加时难以精确归因
4. **可视化效果的呈现**——太多信息挤在一个图表里反而看不清，设计有效的数据可视化是一个持续优化的过程

## 能力边界

**能做什么：**
- 从历史执行数据中提取趋势和模式
- 量化 Agent 系统改进的效果（"新 Prompt 让成功率提升了 5%"）
- 识别系统性问题（"每天下午 3 点延迟最高"）
- 为容量规划提供数据（"本月的 Token 消耗增长了 30%"，需关注）

**不能做什么：**
- 不能预测未来——历史趋势不代表未来的表现，LLM 模型更新可能打破所有规律
- 不能替代实时监控——事后分析是"慢查询"，不能用于及时发现和响应问题
- 不能完全自动化优化——分析结果需要人来判断和采取行动

## 最终工程优化

1. **数据流水线自动化**——使用 ELK / ClickHouse / BigQuery 构建自动化的事后分析数据流水线，Agent 执行完成后自动写入，查询延迟秒级
2. **内置对比基线**——分析工具自动为每个指标计算"过去 7 天平均值"作为基线，每次分析自动与基线对比，不需要手动选参照
3. **自定义分析模板**——允许开发团队定义自己的分析模板（如"每次 Prompt 更新后的自动化分析报告"），分析完成后自动推送到 Slack/邮件
4. **可交互的分析报告**——分析报告不是静态 PDF，而是可交互的 Web 页面（展开/折叠、下钻、切换时间范围）
