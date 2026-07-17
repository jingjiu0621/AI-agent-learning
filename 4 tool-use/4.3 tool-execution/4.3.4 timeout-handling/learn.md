# 超时处理机制

## 简单介绍

工具调用可能有各种耗时——简单的数据库查询 10ms 完成，复杂的文件分析可能需要 30 秒。超时处理确保长时间运行的工具不会阻塞整个 Agent 系统。好的超时机制决定了系统能否在生产环境中稳定运行。

## 超时类型

```
工具调用超时
├── 连接超时（connect_timeout）
│   └── 建立 TCP/HTTP 连接的最大等待时间
├── 读取超时（read_timeout）
│   └── 两次数据包之间的最大间隔
├── 执行超时（execution_timeout）
│   └── 整个工具执行的最大时间
└── 总超时（total_timeout）
    └── 含重试的总耗时上限
```

## 超时策略设计

### 分层超时

```python
class TimeoutConfig:
    def __init__(
        self,
        connect_timeout: float = 5.0,    # 建立连接超时 5s
        read_timeout: float = 10.0,      # 读取间隔超时 10s
        execution_timeout: float = 30.0, # 总执行超时 30s
        total_timeout: float = 120.0,    # 含重试总超时 2min
    ):
        self.connect_timeout = connect_timeout
        self.read_timeout = read_timeout
        self.execution_timeout = execution_timeout
        self.total_timeout = total_timeout
```

### 按工具类型配置

```python
# 不同的工具有不同的超时需求
timeout_configs = {
    "search_database": TimeoutConfig(execution_timeout=10.0),   # 快速查询
    "generate_report": TimeoutConfig(execution_timeout=120.0),  # 长任务
    "send_email": TimeoutConfig(execution_timeout=15.0),        # 适中
    "web_scrape": TimeoutConfig(execution_timeout=60.0),        # 网络不确定
}
```

### 超时实现

```python
import asyncio

async def execute_with_timeout(handler, args, timeout: float):
    try:
        result = await asyncio.wait_for(
            handler(**args),
            timeout=timeout
        )
        return ToolResult(success=True, data=result)
    except asyncio.TimeoutError:
        return ToolResult(
            success=False,
            error=ToolError(
                type="timeout",
                message=f"工具执行超过 {timeout}s 仍未返回",
                suggestion="工具暂时无响应，可稍后重试，或选择替代方案。"
            )
        )
```

## 超时后的策略

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| 返回超时错误 | 通知 LLM 工具超时了 | 大多数场景 |
| 异步切换 | 转为后台任务，返回任务 ID | 长时间数据处理 |
| 降级方案 | 尝试更快的替代工具 | 有备选方案时 |
| 部分结果 | 返回已获取的部分数据 | 流式请求场景 |
| 自动重试 | 超时后立即重试一次 | 网络抖动场景 |

## 核心挑战

1. **超时时间取值**：太短则正常请求也被中止，太长则系统被慢工具阻塞
2. **超时与流式冲突**：流式工具（SSE、WebSocket）的"连接"可以一直保持，但"数据"应该持续到达
3. **级联超时**：一个工具超时可能触发整个任务链的连锁超时

## 最佳实践

- 设置合理的默认超时（10-30s），对特殊工具单独配置
- 长任务（>30s）应考虑异步模式：返回 task_id，LLM 后续轮询
- 超时错误应包含"已耗时"信息，帮助 LLM 判断是否重试
- 分布式系统中使用"父超时"（parent timeout）防止级联
