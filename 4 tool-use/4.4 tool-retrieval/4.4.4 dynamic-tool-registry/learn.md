# 动态注册与发现

## 简单介绍

动态工具注册与发现允许在运行时向 Agent 系统添加、更新或移除工具，而无需重启服务。这类似于微服务中的服务发现机制——工具像是"微服务"，Agent 框架是"消费者"，注册中心是"服务目录"。

## 基本原理

```
工具实现                                            Agent 框架
    │                                                  │
    ├── 启动时注册 ───────────────────────────────→     │
    │    (name, schema, handler, 元数据)                │
    │                                                  │
    ├── 运行时心跳 ───────────────────────────────→     │
    │    (确认工具仍然可用)                              │
    │                                                  │
    ├── 更新注册 ───────────────────────────────→      │
    │    (变更 Schema / handler 版本)                   │
    │                                                  │
    └── 取消注册 ───────────────────────────────→      │
         (工具下线通知)                                  │
```

## 注册中心模式

```python
class DynamicToolRegistry:
    def __init__(self):
        self._tools: dict[str, ToolRegistration] = {}
        self._watchers: list[Callable] = []
    
    def register(self, registration: ToolRegistration):
        """注册一个新工具"""
        self._tools[registration.name] = registration
        self._notify_watchers("add", registration)
    
    def unregister(self, name: str):
        """移除一个工具"""
        if name in self._tools:
            tool = self._tools.pop(name)
            self._notify_watchers("remove", tool)
    
    def update(self, name: str, updates: dict):
        """更新工具定义"""
        if name in self._tools:
            self._tools[name].apply_updates(updates)
            self._notify_watchers("update", self._tools[name])
    
    def get_available(self, context: ExecutionContext) -> list[ToolDef]:
        """获取当前可用的工具列表（已过权限过滤）"""
        return [
            t.to_tool_def() for t in self._tools.values()
            if self._check_permission(t, context)
        ]
    
    def watch(self, callback: Callable):
        """监听工具变更事件（用于刷新索引）"""
        self._watchers.append(callback)
```

## 注册信息

```python
@dataclass
class ToolRegistration:
    name: str
    description: str
    parameters: dict
    handler: Callable
    version: str
    
    # 元数据
    category: str          # "查询", "写入", "通知" ...
    permissions: list[str] # 需要的权限
    owner: str             # 维护团队
    created_at: datetime
    updated_at: datetime
    
    # 运行时配置
    timeout: float = 30.0
    max_retries: int = 2
    rate_limit: int = 100  # 每分钟调用次数上限
```

## 动态注册的方式

### 1. 装饰器注册

```python
@tool_registry.register(
    name="get_weather",
    description="获取天气",
    category="查询",
    permissions=["weather:read"]
)
async def get_weather(location: str, unit: str = "celsius"):
    # 实现...
    pass
```

### 2. 配置文件注册

```yaml
# tools.yaml
tools:
  - name: get_weather
    description: "获取天气"
    handler: weather_service.get_weather
    parameters: { ... }
    category: 查询
```

### 3. API 注册

```python
# 外部服务通过 API 注册
POST /api/v1/tools/register
{
    "name": "external_service.some_api",
    "description": "...",
    "parameters": {...},
    "endpoint": "https://external.service/api"
}
```

## 核心挑战

1. **安全性**：动态注册可能被恶意工具利用——需要验证注册来源
2. **版本兼容**：工具更新可能破坏现有 Agent 任务——需要版本管理和灰度
3. **索引同步**：工具变更需要同步更新向量索引
4. **资源管理**：每个注册的工具都消耗内存和索引资源——需要配额管理

## 最佳实践

- 注册时进行 Schema 校验（确保工具定义符合规范）
- 工具每隔一段时间发送心跳，超时未心跳的工具自动标记为不可用
- 工具变更时自动触发向量索引增量更新
- 提供工具沙箱环境，注册的工具只能在沙箱中执行
