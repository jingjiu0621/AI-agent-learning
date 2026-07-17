# 并行调用与依赖图管理

## 简单介绍

当 LLM 一次输出多个工具调用时，执行引擎需要判断哪些可以并行执行、哪些有依赖关系必须串行。依赖图管理（Dependency Graph Management）解决了这个问题——它自动分析工具间的数据依赖，构建有向无环图（DAG），并按拓扑顺序执行。

## 基本原理

### 依赖关系类型

```python
# 类型 1：完全独立 → 并行执行
get_weather("北京")     ← 不需要任何前置结果
get_stock_price("AAPL") ← 不需要任何前置结果
# 这两个可以并行

# 类型 2：数据依赖 → 串行执行
result1 = get_user_id("张三")     # 步骤 1
result2 = get_orders(result1.id)  # 步骤 2（依赖步骤 1 的 id）
# 这两个必须串行

# 类型 3：部分依赖 → 混合执行
result1 = get_user_info("张三")     # 步骤 1
result2 = get_weather("北京")       # 步骤 2.1（与步骤 1 并行）
result3 = send_notification(result1.email, result2.weather)  # 步骤 3（依赖步骤 1 和 2）
```

### DAG 调度器

```python
import asyncio
from collections import defaultdict

class DAGScheduler:
    def __init__(self):
        self.dependencies = {}  # tool_call_id -> set of dependency ids
        self.results = {}
    
    def build_dag(self, tool_calls: list):
        """根据参数分析依赖关系，构建 DAG"""
        for tc in tool_calls:
            deps = set()
            for arg_value in extract_values(tc.function.arguments):
                # 检查参数值是否引用了之前结果中的字段
                if matches_result_pattern(arg_value):  # e.g., "${step_1.id}"
                    deps.add(extract_step_id(arg_value))
            self.dependencies[tc.id] = deps
        return self._topological_sort()
    
    async def execute(self, tool_calls: list, registry: dict):
        dag = self.build_dag(tool_calls)
        ready = [tc for tc in tool_calls if not self.dependencies[tc.id]]
        pending = [tc for tc in tool_calls if self.dependencies[tc.id]]
        
        while ready:
            batch = [self._run_one(tc, registry) for tc in ready]
            results = await asyncio.gather(*batch)
            for tc, result in zip(ready, results):
                self.results[tc.id] = result
            
            # 检查哪些 pending 的依赖已满足
            ready = []
            still_pending = []
            for tc in pending:
                self.dependencies[tc.id] -= set(self.results.keys())
                if not self.dependencies[tc.id]:
                    # 注入依赖值
                    tc.function.arguments = inject_values(tc.function.arguments, self.results)
                    ready.append(tc)
                else:
                    still_pending.append(tc)
            pending = still_pending
        
        return self.results
```

## 依赖检测策略

| 策略 | 方法 | 精度 | 复杂度 |
|------|------|------|--------|
| 参数引用 | 检查参数中是否包含 ${step_x.field} 模式 | 高 | 低 |
| 语义匹配 | 用 LLM 判断两个工具是否有依赖 | 中 | 高 |
| 静态规则 | 预定义的依赖关系规则 | 中 | 中 |
| 运行检测 | 实际执行后再决定 | 低 | 低 |

## 核心挑战

1. **LLM 不总是明确标记依赖**：LLM 可能在一个工具的参数中隐含地依赖另一个工具的结果，但没有显式引用
2. **循环依赖**：工具 A 需要 B 的结果，B 也需要 A 的结果——这在现实系统中很少见但可能发生
3. **部分失败**：DAG 中的某个节点失败后，依赖它的下游节点怎么办？
4. **动态 DAG**：某些依赖关系在执行过程中才显现（如根据搜索结果决定下一步）

## 工程实践

- 对大部分场景，"显式参数引用"策略已经够用
- 设置 max_concurrency 防止资源耗尽
- DAG 中某个节点失败后，下游节点的处理策略应该是可配置的（跳过/降级/终止全部）
