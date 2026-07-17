# 6.2.6 memory-serialization — 记忆序列化与反序列化

## 简单介绍

记忆序列化（Memory Serialization）是将 Agent 的内存中的记忆对象（Memory Object）转换为可以存储、传输的字节序列（Bytes）或文本格式（JSON/Protobuf）的过程，反序列化则是其逆过程。

它是"记忆"从运行时状态变为持久化数据的关键桥梁——没有它，短期记忆无法保存到长期存储，长期记忆也无法在需要时重新加载到推理上下文。

## 基本原理

### 序列化的核心链路

```
运行时记忆对象                 序列化格式                    持久化存储
┌──────────────┐     serialize()     ┌────────────────┐      ┌──────────┐
│ Memory Object │ ─────────────────→ │ JSON / Protobuf │ ───→ │  Disk    │
│              │                     │   / MessagePack │      │  Redis   │
│ .messages    │ ←───────────────── │                 │ ←─── │  S3      │
│ .embeddings  │   deserialize()     └────────────────┘      │  DB      │
│ .metadata    │                                            └──────────┘
│ .state       │
└──────────────┘
```

### 序列化什么

一个典型的 Agent 记忆对象包含以下需要序列化的部分：

```python
@dataclass
class AgentMemory:
    # 会话级
    session_id: str
    messages: list[Message]           # 对话消息历史
    state: WorkingMemoryState         # 工作记忆状态
    metadata: dict[str, Any]          # 元数据（时间戳、版本、标签）

    # 知识级（长期）
    facts: list[Fact]                 # 提取的事实
    preferences: list[Preference]     # 用户偏好
    reflections: list[Reflection]     # 反思记录

    # 索引级
    embeddings: dict[str, list[float]]  # 向量嵌入缓存（可选）
    indices: dict[str, int]            # 索引映射
```

## 核心序列化格式对比

### 1. JSON（最常用）

```python
import json
from datetime import datetime

class JSONSerializer:
    """JSON 序列化器——可读性好，通用性强"""

    def serialize(self, memory: AgentMemory) -> str:
        return json.dumps({
            "session_id": memory.session_id,
            "messages": [
                {
                    "role": m.role,
                    "content": m.content,
                    "timestamp": m.timestamp.isoformat(),
                    "metadata": m.metadata,
                }
                for m in memory.messages
            ],
            "state": {
                "current_task": memory.state.current_task,
                "step": memory.state.step,
                "status": memory.state.status,
            },
            "metadata": {
                "created_at": memory.metadata.get("created_at", datetime.now().isoformat()),
                "version": "1.0",
                "agent_id": memory.metadata.get("agent_id"),
            }
        }, ensure_ascii=False, indent=2)

    def deserialize(self, data: str) -> AgentMemory:
        raw = json.loads(data)
        # 重建对象
        messages = [
            Message(
                role=m["role"],
                content=m["content"],
                timestamp=datetime.fromisoformat(m["timestamp"]),
                metadata=m.get("metadata", {}),
            )
            for m in raw["messages"]
        ]
        return AgentMemory(
            session_id=raw["session_id"],
            messages=messages,
            state=WorkingMemoryState(**raw["state"]),
            metadata=raw["metadata"],
        )
```

### 2. MessagePack（高效二进制）

```python
import msgpack

class MessagePackSerializer:
    """MessagePack 序列化器——紧凑，解析快"""

    def serialize(self, memory: AgentMemory) -> bytes:
        return msgpack.packb({
            "s": memory.session_id,
            "m": [(msg.role, msg.content, msg.timestamp.timestamp())
                  for msg in memory.messages],
            "t": memory.state.current_task,
            "p": memory.state.step,
            "v": "1.0",
        }, use_bin_type=True)

    def deserialize(self, data: bytes) -> AgentMemory:
        raw = msgpack.unpackb(data)
        messages = [
            Message(role=r[0], content=r[1],
                    timestamp=datetime.fromtimestamp(r[2]))
            for r in raw[b"m"]
        ]
        return AgentMemory(
            session_id=raw[b"s"].decode(),
            messages=messages,
            state=WorkingMemoryState(current_task=raw.get(b"t"), step=raw.get(b"p")),
        )
```

### 3. Protocol Buffers（结构化强类型）

```protobuf
// memory.proto
syntax = "proto3";

message AgentMemory {
  string session_id = 1;
  repeated Message messages = 2;
  WorkingMemoryState state = 3;
  Metadata metadata = 4;
  float version = 5;
}

message Message {
  string role = 1;
  string content = 2;
  int64 timestamp = 3;
  map<string, string> metadata = 4;
}

message WorkingMemoryState {
  string current_task = 1;
  int32 step = 2;
  string status = 3;
}
```

### 格式对比

| 格式 | 空间效率 | 序列化速度 | 可读性 | Schema 强制 | 多语言支持 |
|------|---------|-----------|-------|------------|----------|
| JSON | 低（~1x） | 中 | 优 | 否 | 优 |
| MessagePack | 中（~0.7x） | 快 | 差 | 否 | 中 |
| Protobuf | 高（~0.3x） | 最快 | 差 | 是 | 优 |
| Pickle | 高（~0.4x） | 快 | 极差 | 否 | Python only |

## 高级序列化模式

### 模式 1：增量序列化（Differential Serialization）

只序列化变化的部分，而非全量：

```python
class DifferentialSerializer:
    """增量序列化：只存储变更"""

    def __init__(self):
        self.last_snapshot: dict | None = None

    def serialize_diff(self, current: AgentMemory) -> bytes | None:
        current_dict = self._to_dict(current)
        if self.last_snapshot is None:
            self.last_snapshot = current_dict
            return json.dumps(current_dict).encode()  # 全量

        diff = self._compute_diff(self.last_snapshot, current_dict)
        self.last_snapshot = current_dict

        if diff:
            return msgpack.packb(diff)  # 仅增量
        return None  # 无变化

    def _compute_diff(self, old: dict, new: dict) -> dict:
        """计算两个状态的差异（简化版）"""
        diff = {}
        for key in new:
            if key not in old:
                diff[key] = ("+", new[key])
            elif old[key] != new[key]:
                if isinstance(new[key], list):
                    # 列表增量：新增的元素
                    added = [x for x in new[key] if x not in old[key]]
                    if added:
                        diff[key] = ("+", added)
                else:
                    diff[key] = ("~", old[key], new[key])
        for key in old:
            if key not in new:
                diff[key] = ("-",)
        return diff
```

### 模式 2：分层序列化（Layered Serialization）

不同层级使用不同的序列化策略：

```python
class LayeredSerializer:
    """分层序列化——热数据快，冷数据省"""

    LEVELS = {
        "hot": {     # 频繁访问 → 最快序列化/反序列化
            "format": "pickle",
            "storage": "redis",
            "compression": None,
        },
        "warm": {    # 偶尔访问 → 平衡
            "format": "msgpack",
            "storage": "redis",
            "compression": "lz4",
        },
        "cold": {    # 很少访问 → 最小存储
            "format": "json",
            "storage": "s3",
            "compression": "gzip",
        },
    }

    def serialize(self, memory: AgentMemory, level: str = "warm") -> bytes:
        config = self.LEVELS[level]
        data = self._format(memory, config["format"])
        if config["compression"]:
            data = self._compress(data, config["compression"])
        return data
```

### 模式 3：版本化序列化（Versioned Serialization）

支持不同版本的数据格式，向前兼容：

```python
class VersionedSerializer:
    """版本化序列化——支持数据格式演进"""

    CURRENT_VERSION = 3

    def serialize(self, memory: AgentMemory) -> str:
        return json.dumps({
            "__version__": self.CURRENT_VERSION,
            "__schema__": "agent_memory_v3",
            "session_id": memory.session_id,
            "messages": [...],
            "state": {...},
        })

    def deserialize(self, data: str) -> AgentMemory:
        raw = json.loads(data)
        version = raw.get("__version__", 1)

        if version < self.CURRENT_VERSION:
            raw = self._upgrade(raw, version)
        return self._build_memory(raw)

    def _upgrade(self, raw: dict, from_version: int) -> dict:
        """数据格式升级"""
        upgrades = {
            1: self._v1_to_v2,
            2: self._v2_to_v3,
        }
        for v in range(from_version, self.CURRENT_VERSION):
            if v in upgrades:
                raw = upgrades[v](raw)
        return raw
```

## 记忆压缩

序列化过程中的重要优化——在保持信息可用的前提下减少数据体积。

### 压缩策略

| 策略 | 压缩比（近似） | 信息损失 | 恢复代价 |
|------|--------------|---------|---------|
| gzip 压缩 | 5-10x | 无 | 中 |
| 摘要替代 | 10-50x | 有 | 高 |
| 结构化精简 | 2-3x | 无 | 低 |
| 浮点量化 (f32→f16) | 2x | 有（精度可忽略） | 低 |
| 键名缩短 | 1.5-2x | 无 | 低 |

```python
def compress_memory(memory: AgentMemory, level: str = "standard") -> bytes:
    """多级记忆压缩"""
    serialized = json.dumps(_to_dict(memory))

    if level == "fast":
        return serialized.encode()

    # 键名缩短
    short_map = {
        "session_id": "s", "messages": "m", "role": "r",
        "content": "c", "timestamp": "ts", "metadata": "md",
    }
    for long_k, short_k in short_map.items():
        serialized = serialized.replace(f'"{long_k}"', f'"{short_k}"')

    if level == "standard":
        return gzip.compress(serialized.encode(), level=6)

    if level == "max":
        # 先摘要合并非关键消息
        memory = _summarize_old_messages(memory)
        serialized = json.dumps(_to_dict(memory))
        return gzip.compress(serialized.encode(), level=9)
```

## 背景

### 为什么记忆序列化是重要工程问题

早期 Agent 框架（2023）的记忆持久化很简单——把消息列表 dump 成 JSON 存到文件。但随着 Agent 系统复杂化，问题浮现：

1. **体积膨胀**：128K 上下文窗口的对话历史序列化后可达数 MB，频繁读写成本高昂
2. **格式碎片化**：每个模块用自己的格式序列化记忆，缺乏统一 Schema
3. **向前兼容**：代码升级后旧的序列化数据无法读取
4. **传输效率**：Agent 分布式部署后，记忆需要在服务间传输，序列化效率直接影响延迟

### 关键洞察

记忆序列化不只是一个"对象→字节"的转换——它决定了：

- 记忆可以**存活多久**（好的压缩让存储空间翻倍）
- 记忆可以**传多快**（网络传输中，序列化格式影响延迟）
- 记忆可以**怎么查**（结构化的序列化格式支持部分反序列化）
- 记忆可以**怎么演进**（版本化序列化支持系统升级）

## 核心矛盾

| 矛盾 | 说明 |
|------|------|
| **可读性 vs 效率** | JSON 可读性好但空间效率低；二进制格式效率高但难以调试 |
| **紧凑 vs 可扩展** | 字段名缩短节省空间但 Schema 演进困难；保留全名灵活但体积大 |
| **压缩 vs 访问速度** | 压缩比越高，序列化/反序列化越慢，影响实时读写 |
| **版本兼容 vs 代码复杂** | 向前兼容需要版本字段和升级逻辑，增加代码复杂度 |

## 当前主流优化方向

1. **Schema Registry**：集中管理记忆数据的 Schema，支持兼容性检查（向前/向后）
2. **部分反序列化**：只反序列化需要的字段（如只读取元数据就决定是否加载完整记忆）
3. **零拷贝序列化**：使用 Arrow/Flight 等格式，在序列化数据上直接计算，无需反序列化
4. **自适应压缩**：根据记忆的大小和访问频率动态选择压缩级别（小/高频→不压缩，大/低频→高压缩）
5. **流式序列化**：大型记忆分块序列化和反序列化，避免一次性加载到内存

## 工程优化方向

1. **异步序列化**：Agent 推理和记忆序列化分离，推理完成后台异步完成序列化
2. **序列化缓存**：最近使用的序列化结果缓存，避免同一记忆反复序列化
3. **校验和验证**：序列化数据带校验和，反序列化时验证完整性
4. **监控指标**：跟踪序列化大小、耗时、压缩比、版本分布

## 能力边界与结果边界

**能做的**：
- 将 Agent 的运行时记忆状态安全地持久化到各种存储后端
- 通过版本化机制支持系统升级后的数据兼容
- 通过压缩减少 80-90% 的存储空间占用

**不能做的**：
- ❌ 无损压缩不能无限压缩（有损压缩会丢失信息，不适合关键记忆）
- ❌ 不能解决底层存储的容量上限（序列化再高效也受磁盘空间限制）
- ❌ 无法保证序列化数据的安全性（需要额外的加密层）

**结果边界**：
- 合理的序列化方案可以让记忆存取延迟降低到 <1ms（内存级）或 <10ms（磁盘级）
- 版本化序列化是生产系统"必须做"而非"最好做"的事情
- 序列化格式的选择应该基于"谁会读这个数据"——只有机器读 → 二进制格式；人也要读 → JSON

## 与相关内容的区别

| 对比 | 序列化 | 压缩 | 存储引擎 |
|------|-------|------|---------|
| 关注点 | 对象↔字节转换 | 体积缩小 | 数据持久化 |
| 操作对象 | 内存对象 | 字节流 | 文件/记录 |
| 性能目标 | 快速转换 | 小体积 | 高效读写 |
| 副作用 | 无 | 可能有损 | 写入/读取延迟 |

## 核心优势

1. **格式自由**：不绑定特定编程语言或运行时（好的序列化方案跨语言互操作）
2. **Schema 演进**：版本化支持在不中断服务的情况下升级数据格式
3. **传输优化**：高效的序列化格式减少网络传输量，分布式部署的关键基础设施
4. **调试友好**：好的序列化方案支持在开发和调试时以可读格式查看记忆内容

## 适合场景的判断标准

- **数据量**：小记忆（<100KB）→ JSON 够用；大记忆（>1MB）→ 需要二进制格式+压缩
- **访问频率**：高频读写 → 快速序列化（pickle/msgpack）；低频访问 → 高压缩（gzip JSON）
- **跨语言需求**：单语言系统 → 语言特定方案（Python pickle）；多语言系统 → Protobuf/Arrow
- **Schema 稳定性**：稳定不变 → 简单序列化；频繁变更 → 版本化序列化
- **可调试性**：需要人工检查 → JSON；纯机器处理 → 任意二进制格式

**最适合**：生产环境中的长期记忆持久化、分布式 Agent 系统的记忆传输、需要数据版本管理的场景

**最不适合**：Agent 运行时的内存内操作（运行时不应反复序列化/反序列化）、极高性能要求的实时路径
