# 工具索引构建与 Embedding

## 简单介绍

工具索引是将每个工具定义转换为可搜索的向量表示的过程。通过将工具名、描述、参数等信息编码为 Embedding 向量，使 Agent 可以通过语义搜索快速找到与当前任务最相关的工具。这是海量工具场景（100+）的基础设施。

## 基本原理

```
工具定义
  ↓
工具文档构建（将工具定义转为可嵌入的文本）
  ↓
Embedding 模型（text-embedding-3, BGE, Cohere）
  ↓
向量存储（Chroma, Pinecone, Qdrant）
  ↓
运行时查询
```

## 工具文档构建

工具定义（JSON）不能直接嵌入，需要先转为自然语言描述：

```python
def build_tool_document(tool_def: dict) -> str:
    """将工具定义转换为适合嵌入的文本"""
    doc = f"Tool: {tool_def['name']}\n"
    doc += f"Description: {tool_def['description']}\n"
    
    if "parameters" in tool_def:
        doc += "Parameters:\n"
        for name, prop in tool_def["parameters"].get("properties", {}).items():
            doc += f"  - {name} ({prop.get('type', 'string')}): {prop.get('description', '')}\n"
            if "enum" in prop:
                doc += f"    Allowed values: {', '.join(prop['enum'])}\n"
    
    return doc

# 示例输出：
# Tool: get_weather
# Description: 获取指定城市的天气数据
# Parameters:
#   - location (string): 城市名称
#   - unit (string): 温度单位
#     Allowed values: celsius, fahrenheit
```

## Embedding 模型选择

| 模型 | 维度 | 适用场景 | 成本 |
|------|------|----------|------|
| text-embedding-3-small | 512/1536 | 通用场景 | 低 |
| text-embedding-3-large | 256/1024/3072 | 高精度需求 | 中 |
| BGE-base | 768 | 中文场景 | 低 |
| Cohere embed-multilingual | 1024 | 多语言 | 中 |

## 索引策略

### 1. 单向量索引

每个工具只生成一个向量（最简单）：

```
工具名 + 描述 → Embedding → 1 个向量
```

### 2. 多向量索引

一个工具生成多个向量（覆盖不同方面）：

```
工具的原始文档 → Embedding → 向量 1（通用表示）
工具的查询示例 → Embedding → 向量 2（查询场景表示）
工具的参数说明 → Embedding → 向量 3（参数细节表示）
```

### 3. 分层索引

```
Level 1: 工具类别（"数据查询类"、"通知发送类"）
Level 2: 具体工具
Level 3: 工具参数的组合方式
```

## 索引优化

| 技术 | 效果 | 复杂度 |
|------|------|--------|
| Prefix Caching | 相同前缀查询加速 | 低 |
| 工具分组索引 | 缩小搜索范围 | 中 |
| 混合索引（向量+关键词） | 提高召回率 | 中 |
| 索引分片 | 横向扩展 | 高 |
| 增量索引 | 工具变更时局部更新 | 中 |

## 工程实现示例

```python
import chromadb
from openai import OpenAI

class ToolIndex:
    def __init__(self):
        self.client = chromadb.Client()
        self.collection = self.client.create_collection("tools")
        self.embed_client = OpenAI()
    
    def add_tool(self, tool_def: dict):
        doc = build_tool_document(tool_def)
        self.collection.add(
            documents=[doc],
            ids=[tool_def["name"]],
            metadatas=[{"name": tool_def["name"], "type": "tool"}]
        )
    
    def search(self, query: str, top_k: int = 10):
        return self.collection.query(
            query_texts=[query],
            n_results=top_k
        )
```

## 核心挑战

1. **语义鸿沟**：用户说"发邮件"，但工具名是 send_email——需要语义理解而非关键词匹配
2. **参数细节丢失**：工具描述向量可能丢失参数级别的信息
3. **动态工具**：工具频繁增删改，索引需要实时更新
4. **冷启动**：新工具有 Embedding 但没有使用历史，检索质量不确定
