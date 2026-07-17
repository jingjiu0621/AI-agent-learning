# 参数设计：必选/可选/默认值/枚举约束

## 简单介绍

工具的参数设计直接影响 LLM 调用工具的准确率和可靠性。好的参数设计让 LLM"很难用错"，差的参数设计则让 LLM 频繁出错。关键在于遵循**最少参数原则**——只暴露 LLM 真正需要决策的参数，其余全部硬编码或使用默认值。

## 核心原则

### 1. 最少参数原则（Minimum Viable Parameters）

工具定义中每个额外的参数都增加了 LLM 的决策负担：

```
❌ 差：8 个参数，其中 5 个可选
✅ 好：3 个必选参数，2 个常用可选参数
```

**每个参数都是一种认知负担。如果参数可以被隐含推理出来，就不应该成为参数。**

### 2. 默认值优于可选参数

```json
{
  "type": "object",
  "properties": {
    "language": {
      "type": "string",
      "enum": ["zh", "en"],
      "default": "zh",  // 明确默认值
      "description": "翻译的目标语言，默认为中文"
    }
  }
}
```

### 3. Enum 约束优于自由文本

```
❌ 参数: "sort": {"type": "string"}
    → LLM 可能输出 "price_low_to_high" / "price_asc" / "by price" 等各种变体

✅ 参数: "sort": {"type": "string", "enum": ["price_asc", "price_desc", "rating"]}
    → LLM 限制在三个选项内
```

## 参数设计策略

| 策略 | 说明 | 示例 |
|------|------|------|
| 硬编码 | 参数值不影响核心逻辑 | API 版本号、超时时间 |
| 默认值 | 大部分场景通用 | 分页大小（默认 20） |
| 自动推导 | 从上下文推断 | 当前时间（如果不传就取当前时间） |
| LLM 决策 | 需要 LLM 理解意图 | 搜索关键词、筛选条件 |
| 用户确认 | 高风险操作 | 删除确认、支付确认 |

## 典型参数陷阱

### 陷阱 1：布尔参数语义不清

```
❌ 差: {"send_notification": {"type": "boolean"}}
    → LLM 不理解 true/false 的含义

✅ 好: {"send_notification": {"type": "boolean", "description": "是否发送通知给用户。true=发送, false=不发送"}}
```

### 陷阱 2：缺少类型约束

```
❌ 差: {"user_ids": {"type": "array", "items": {"type": "string"}}}
    → LLM 可能传空数组或者单字符串而非数组

✅ 好: {"user_ids": {"type": "array", "items": {"type": "string"}, "minItems": 1, "description": "用户 ID 列表，至少传一个"}}
```

### 陷阱 3：参数过多

一个工具有 12 个参数，其中 8 个可选 → LLM 通常只会填前 3-4 个。

## 工程示例

```python
# 反模式：把所有配置都作为参数
def search_database(
    query,           # LLM 决策 ✓
    table,           # LLM 决策 ✓  
    limit=20,        # 默认值 ✓
    offset=0,        # 默认值 ✓
    use_cache=True,  # 硬编码 ✓（不暴露给 LLM）
    timeout=30,      # 硬编码 ✓
    retry=3,         # 硬编码 ✓
):
    ...

# 更好的做法：包装器只暴露 LLM 需要的参数
def search_database_for_agent(query: str, table: str, limit: int = 20):
    """只暴露 3 个参数，其余由包装器内部处理"""
    return _search_database(
        query=query, table=table, limit=limit,
        use_cache=True, timeout=30
    )
```

## 最佳实践

1. **首选硬编码，其次默认值，再是可选参数，最后才是必选**
2. **必选参数的数量不超过 4 个**——超过了说明粒度太粗或太细
3. **对 enum 类型的每个值都写 description**——帮助 LLM 区分
4. **避免自由文本参数**——用 enum / pattern / format 约束
5. **对象参数（object）不要嵌套超过 2 层**——太深的结构 LLM 容易出错
