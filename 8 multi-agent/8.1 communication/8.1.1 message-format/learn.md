# 8.1.1 message-format — 消息格式（JSON-RPC / Protobuf / 自定义）

## 简单介绍

消息格式是多 Agent 通信的"语言"——定义了 Agent 之间传递信息的结构、语法和语义。选择正确的消息格式直接影响系统的可读性、性能、可扩展性和跨语言互操作性。

## 基本原理

### 消息的核心构成

无论使用何种格式，Agent 间消息通常包含以下字段：

```
{
  "id": "msg_001",              // 消息唯一标识
  "from": "agent/researcher",   // 发送者
  "to": "agent/writer",         // 接收者（或路由目标）
  "type": "request/response/event/broadcast",
  "timestamp": 1700000000,      // 时间戳
  "ttl": 60000,                 // 生存时间（毫秒）
  "payload": { ... },           // 消息体
  "metadata": {                 // 元数据
    "correlation_id": "req_001", // 关联 ID（用于请求-响应配对）
    "priority": "high",
    "version": "1.0"
  }
}
```

### 三种主流格式

| 格式 | 序列化 | 可读性 | 性能 | Schema 支持 | 跨语言 |
|------|--------|--------|------|-------------|--------|
| JSON-RPC | JSON | ⭐⭐⭐ | ⭐⭐ | JSON Schema | ⭐⭐⭐ |
| Protocol Buffers | 二进制 | ⭐ | ⭐⭐⭐⭐⭐ | .proto 强制 | ⭐⭐⭐⭐⭐ |
| 自定义文本 | XML/YAML | ⭐⭐⭐⭐ | ⭐ | DTD/XSD | ⭐⭐ |

## 背景与演进

- **早期（2022-2023）**：多 Agent 系统刚兴起时，大多直接传递原始 JSON 字典，没有统一格式。结果是消息结构混乱，解析逻辑耦合严重。
- **JSON-RPC 规范化（2023）**：借鉴分布式系统的 JSON-RPC 2.0 规范，定义了标准化的请求/响应/错误结构。AutoGen 的对话消息格式就是基于此。
- **强类型需求（2024）**：随着 Agent 系统规模扩大，JSON 的动态类型导致了大量运行时错误。Protocol Buffers 开始被引入，特别是在需要高性能跨语言通信的场景。
- **当前趋势**：JSON 用于灵活性优先的场景（快速原型、调试），Protobuf 用于性能优先的场景（高频通信、大规模部署）。混合方案也越来越常见——内部用 Protobuf，外部暴露 JSON 接口。

## 核心矛盾

**灵活性与严格性的权衡**：

| JSON 倾向 | Protobuf 倾向 |
|-----------|---------------|
| 动态类型，快速迭代 | 强类型，编译期检查 |
| 人类可读，调试友好 | 二进制，需工具解析 |
| 无编译步骤 | 需要编译生成代码 |
| Schema 可选 | Schema 强制 |
| 解析性能较低 | 解析性能极高 |
| 消息体积大 | 消息体积小（1/10） |

## 主流优化方向

1. **Schema-first 设计**：无论是 JSON Schema 还是 .proto 文件，先定义接口再实现
2. **版本化消息**：消息格式添加版本号，支持向后兼容的演进
3. **可选字段模式**：使用 optional/default 字段避免破坏性变更
4. **Codec 抽象层**：在应用层和传输层之间插入编解码层，支持格式切换
5. **消息压缩**：对大型消息启用 gzip / snappy 压缩
6. **差分更新**：只传递变化部分而非完整消息（类似 JSON Patch）

## 能力边界与结果边界

| 维度 | JSON-RPC | Protocol Buffers |
|------|----------|-----------------|
| 最大消息体 | 受内存限制，通常 <10MB | 同 JSON，但更节省空间 |
| 字段数量 | 无硬限制，但大对象影响性能 | 2^29 个字段 |
| 嵌套深度 | 受解析器限制 | 支持嵌套，但深度影响性能 |
| 可选字段 | 天然支持 | proto3 支持 (optional) |
| 枚举类型 | 字符串模拟 | 原生支持 |
| OneOf 联合 | 需手动实现 | 原生支持 |
| Map 类型 | 原生支持 | map<K,V> 支持 |
| 默认值 | undefined | 零值（可能造成歧义） |

## 与其他通信要素的区别

- **vs 传输协议（8.1.2/8.1.3）**：消息格式是"说什么"，传输协议是"怎么说"。同一格式可跑在不同传输层上。
- **vs 结构化对话（8.1.5）**：消息格式定义单条消息的结构，结构化对话定义多轮消息的交互模式。
- **vs API Schema**：Agent 消息比传统 API 更灵活，需要承载元数据（意图、上下文、角色标记）。

## 核心优势

- **JSON-RPC**：零依赖、快速原型、生态丰富、调试友好
- **Protobuf**：极致性能、强类型安全、跨语言无缝、向后兼容
- **混合方案**：取两者之长——内部高速通道用 Protobuf，外部集成用 JSON

## 工程优化

1. **消息 ID 生成**：使用 UUID v7（时间有序）或 Snowflake ID，便于排序和追踪
2. **Schema 注册表**：集中管理所有消息 Schema，实现运行时验证
3. **lint + 格式化**：protolint / json-schema-lint 保证消息格式一致性
4. **代码生成**：从 Schema 自动生成客户端/服务端代码，消除手动解析
5. **测试替身**：基于 Schema 生成 mock 消息用于测试

## 适用场景

| 场景 | 推荐格式 | 原因 |
|------|----------|------|
| 快速原型 / Hackathon | JSON | 零配置，直接调试 |
| 生产级多 Agent 系统 | Protobuf | 性能、类型安全 |
| 异构系统集成 | JSON（+Schema） | 跨语言兼容性好 |
| 高频低延迟通信 | Protobuf | 序列化速度 <1μs |
| Browser ↔ Server | JSON | 浏览器原生支持 |
| IoT Agent | Protobuf | 消息体积小，带宽敏感 |

## 示例代码

### JSON-RPC 风格消息

```python
import json
import uuid
import time
from dataclasses import dataclass, asdict
from typing import Any, Optional

@dataclass
class AgentMessage:
    id: str
    from_agent: str
    to_agent: str
    msg_type: str  # request / response / event / error
    payload: dict
    correlation_id: Optional[str] = None
    timestamp: Optional[float] = None
    priority: int = 0
    ttl_ms: int = 30000

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = time.time()

    def serialize(self) -> str:
        data = asdict(self)
        data["from"] = data.pop("from_agent")
        data["to"] = data.pop("to_agent")
        return json.dumps(data)

    @classmethod
    def deserialize(cls, raw: str) -> "AgentMessage":
        data = json.loads(raw)
        data["from_agent"] = data.pop("from")
        data["to_agent"] = data.pop("to")
        return cls(**data)

    def is_expired(self) -> bool:
        return (time.time() - self.timestamp) * 1000 > self.ttl_ms

# 使用示例
msg = AgentMessage(
    id=uuid.uuid4().hex,
    from_agent="agent/orchestrator",
    to_agent="agent/researcher",
    msg_type="request",
    payload={"task": "search_papers", "query": "multi-agent systems"},
    correlation_id="task_001",
    priority=1
)
raw = msg.serialize()
# 网络传输...
parsed = AgentMessage.deserialize(raw)
print(f"{parsed.from_agent} -> {parsed.to_agent}: {parsed.payload}")
```

### Protocol Buffers 定义示例

```protobuf
syntax = "proto3";

package multiagent.comm;

message AgentMessage {
  string id = 1;
  string from_agent = 2;
  string to_agent = 3;
  MessageType type = 4;
  bytes payload = 5;
  string correlation_id = 6;
  double timestamp = 7;
  int32 priority = 8;
  int32 ttl_ms = 9;
  map<string, string> metadata = 10;

  enum MessageType {
    UNKNOWN = 0;
    REQUEST = 1;
    RESPONSE = 2;
    EVENT = 3;
    ERROR = 4;
    BROADCAST = 5;
  }
}

message SearchTask {
  string query = 1;
  int32 max_results = 2;
  repeated string sources = 3;
  bool require_fulltext = 4;
}
```

### Schema 验证（JSON Schema）

```python
from jsonschema import validate, ValidationError

AGENT_MESSAGE_SCHEMA = {
    "type": "object",
    "required": ["id", "from", "to", "type", "payload"],
    "properties": {
        "id": {"type": "string", "pattern": "^msg_"},
        "from": {"type": "string"},
        "to": {"type": "string"},
        "type": {"enum": ["request", "response", "event", "error"]},
        "payload": {"type": "object"},
        "correlation_id": {"type": "string"},
        "priority": {"type": "integer", "minimum": 0, "maximum": 10}
    }
}

def validate_message(raw: dict) -> bool:
    try:
        validate(instance=raw, schema=AGENT_MESSAGE_SCHEMA)
        return True
    except ValidationError as e:
        print(f"Schema 验证失败: {e.message}")
        return False
```
