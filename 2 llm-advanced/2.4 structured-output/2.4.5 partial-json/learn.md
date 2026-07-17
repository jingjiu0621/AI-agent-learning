# 部分 JSON 解析与流式结构化输出

## 简单介绍

在流式（Streaming）场景下，LLM 的输出是逐步到达的，而不是一次性完整返回。部分 JSON 解析（Partial JSON Parsing）处理的是"如何在不完整的、逐步到达的 Token 流中，提前解析出可用的结构化数据"的问题。这在需要低延迟的 Agent 系统中尤其重要。

## 问题定义

### 完整 JSON vs 流式 JSON

```
完整响应（非流式）:
  {"name": "张三", "age": 25, "city": "北京"}
  → 一次性拿到全部数据

流式响应:
  {"name" → 还不够
  : "张三" → 知道了 name!
  , "age" → 还能解析  
  : 25 → 知道了 age!
  , "city" → 继续
  : "北京" → 知道了 city!
  } → 完成
  → 在数据到达时逐步解析
```

## 部分 JSON 解析的策略

### 策略 1：容错解析器

使用专门的库（如 `partial-json-parser`）来渐进式解析：

```python
from partial_json_parser import loads as partial_loads

def process_stream(chunks):
    """处理流式 JSON 输出"""
    buffer = ""
    for chunk in chunks:
        buffer += chunk
        try:
            # 尝试解析部分 JSON
            data = partial_loads(buffer)
            
            # 即使不完整，也可以提取已有字段
            if "name" in data:
                print(f"已识别 name: {data['name']}")
            if "age" in data:
                print(f"已识别 age: {data['age']}")
                
        except:
            # 当前还不够解析出合法结构
            pass
```

### 策略 2：基于状态的增量更新

```python
class StreamingJSONParser:
    """基于状态机的流式 JSON 解析"""
    
    def __init__(self):
        self.stack = []  # 当前嵌套深度
        self.current_key = None
        self.result = {}
        self.buffer = ""
    
    def feed(self, token: str):
        self.buffer += token
        # 用有限状态机解析 JSON 结构
        for char in token:
            self._process_char(char)
    
    def _process_char(self, char):
        if char == '{':
            self.stack.append({})
        elif char == '}':
            # 结束当前对象
            ...
        elif char == '"':
            # 字符串开始/结束
            ...
    
    def get_partial_result(self) -> dict:
        """返回当前已解析的部分结果"""
        return self.result
```

### 策略 3：正则表达式提取

对于简单的键值对，可以用正则提前提取：

```python
import re

def extract_partial_json(buffer: str) -> dict:
    """用正则从部分 JSON 中提取键值对"""
    result = {}
    
    # 提取字符串值
    string_pattern = r'"(\w+)":\s*"([^"]*)"'
    for match in re.finditer(string_pattern, buffer):
        result[match.group(1)] = match.group(2)
    
    # 提取数值
    number_pattern = r'"(\w+)":\s*(\d+)'
    for match in re.finditer(number_pattern, buffer):
        result[match.group(1)] = int(match.group(2))
    
    return result
```

## 部分 JSON 解析的挑战

| 挑战 | 说明 | 解决方案 |
|------|------|----------|
| 截断值 | `"name": "张` 被截断了 | 忽略不完整的值 |
| 嵌套结构 | `{a: {b: {c: ...` 深层嵌套 | 限制解析深度 |
| 数组 | `[1, 2, 3` 不完整的数组 | 返回部分数组 |
| 转义字符 | `"text\": \"hel` 转义字符被截断 | 安全回退 |

## 工程实现

```python
# 综合使用
class StreamingStructuredOutput:
    def __init__(self):
        self.full_buffer = ""
        self.last_valid = {}
    
    async def process_tokens(self, stream):
        async for chunk in stream:
            self.full_buffer += chunk
            
            # 尝试提取 JSON
            partial_data = self._safe_parse(self.full_buffer)
            if partial_data is not None:
                # 通知监听者新数据
                await self.notify_update(partial_data)
                self.last_valid = partial_data
    
    def _safe_parse(self, text: str):
        """安全解析部分 JSON，失败时返回 None"""
        try:
            return json.loads(text)
        except json.JSONDecodeError:
            # 尝试部分解析
            try:
                return partial_json_parse(text)
            except:
                return None
```

## 实际应用场景

- **Agent 推理流式展示**：在工具调用参数生成时，逐步展示参数而非等待全部生成
- **搜索结果的增量展示**：搜索结果逐个出现，而非一次性全部展示
- **表单生成**：LLM 生成的表单字段逐步出现

## 工程启示

- 部分 JSON 解析在 Agent 系统的"流式 Agent"场景中非常有用
- 对于输出 Token 较长的场景（如代码生成、长文本分析），部分解析可以显著改善用户体验
- 但并非所有 Agent 场景都需要——简单工具调用通常等待完整输出即可
- Python 生态中 `partial-json-parser` 库是最常用的方案
- 注意：部分解析的结果可能和最终结果不一致（如值后续被修改），需要做好更新处理
