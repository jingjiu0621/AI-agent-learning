# API → Tool Definition 自动生成

## 简单介绍

手动编写工具定义在工具数量少时尚可接受，但当系统有几十上百个 API 时，手动编写无法扩展。API 到工具定义的自动生成成为必备能力——它从已有的 API 描述（OpenAPI 规范、代码注释、类型定义等）自动生成 LLM 可消费的工具 Schema。

## 基本原理

```
[API 描述] → [解析器/转换器] → [工具定义 JSON]
     ↑                            ↓
  OpenAPI / TypeScript /       name, description, 
  Protobuf / 代码注释          parameters(JSON Schema)
```

## 几种自动生成方案

### 方案 1：从 OpenAPI 规范生成

最常见的方案。适用于已有 OpenAPI/Swagger 文档的系统。

```python
# 简化示例
def convert_openapi_to_tools(spec: dict) -> list[dict]:
    tools = []
    for path, path_item in spec["paths"].items():
        for method, operation in path_item.items():
            tool = {
                "name": operation.get("operationId", f"{method}_{path}"),
                "description": operation.get("summary", ""),
                "parameters": {
                    "type": "object",
                    "properties": {},
                    "required": []
                }
            }
            # 转换参数...
            tools.append(tool)
    return tools
```

### 方案 2：从 TypeScript/Java 类型生成

适用于前端/后端已经有类型定义的项目。

```typescript
// TypeScript 类型定义
interface SendEmailParams {
  to: string;      // 收件人地址
  subject: string; // 邮件主题
  body: string;    // 邮件正文
}

// 通过 TypeScript 编译器 API 提取类型信息
// → 转换为 JSON Schema
// → 生成 Tool Definition
```

### 方案 3：从代码注释/文档生成

适用于非标准化描述的旧系统。

```python
def send_email(to: str, subject: str, body: str):
    """发送电子邮件给指定收件人"""
    # 通过函数签名 + docstring 自动生成工具定义
    # 参数: to(str), subject(str), body(str)
    # 描述: "发送电子邮件给指定收件人"
```

## 核心挑战

| 挑战 | 说明 | 解决方案 |
|------|------|----------|
| 描述不足 | 自动生成的描述通常太技术化 | 描述优化后处理 |
| 过度暴露 | 所有 API 都可能被 Agent 调用 | 白名单/黑名单过滤 |
| 安全参数 | API Key、token 等参数可能暴露 | 自动移除敏感参数 |
| 命名不一致 | 生产 API 命名可能不统一 | 自动重命名/规范化 |
| 响应 Schema | 工具定义不包含响应格式 | 额外记录响应 Schema |

## 工程示例

```python
import ast
import inspect

def function_to_tool(func) -> dict:
    """从 Python 函数自动生成工具定义"""
    sig = inspect.signature(func)
    doc = inspect.getdoc(func) or ""
    
    tool = {
        "name": func.__name__,
        "description": doc.split("\n")[0],  # 第一行作描述
        "parameters": {"type": "object", "properties": {}, "required": []}
    }
    
    for name, param in sig.parameters.items():
        param_doc = extract_param_doc(doc, name)
        tool["parameters"]["properties"][name] = {
            "type": type_to_json_schema(param.annotation),
            "description": param_doc
        }
        if param.default is inspect.Parameter.empty:
            tool["parameters"]["required"].append(name)
    
    return tool
```

## 自动生成的局限

1. **描述质量有限**：自动生成的描述无法替代人工编写的精准描述
2. **缺少场景说明**：自动生成不知道 "这个工具什么时候该用"
3. **安全边界模糊**：自动生成无法判断哪些操作是危险的
4. **语义信息缺失**：类型信息有了，但"用途"信息不足

## 最佳实践

1. **自动生成 + 人工审核**：自动生成 80% 内容，人工优化 20% 关键工具的描述
2. **增量更新**：API 变更时自动更新对应工具定义，避免手动同步
3. **定期废弃检测**：标记不再使用的 API，从 Agent 工具集中移除
4. **安全过滤层**：自动生成后增加安全过滤器，移除敏感操作的参数
