# OpenAPI / Swagger 规范

## 简单介绍

OpenAPI 规范（原名 Swagger）是描述 REST API 的标准格式。当你的系统已经有大量 OpenAPI 文档化的 API 时，最自然的方式就是将这些已有的 API 定义转化为 LLM 可以理解的工具定义。OpenAPI-to-Function-Call 转换是企业级 Agent 平台中最常见的需求之一。

## 基本原理

### 从 OpenAPI 到 Tool Definition 的映射

| OpenAPI 元素 | FC 工具定义 | 说明 |
|-------------|-------------|------|
| `operationId` | `name` | 唯一标识工具 |
| `summary`/`description` | `description` | 工具描述 |
| `parameters` | `properties` | 请求参数 |
| `requestBody` | `properties` | 请求体参数 |
| `responses` | （不直接使用） | 用于结果解析 |

### 转换示例

```yaml
# OpenAPI 3.0 定义
paths:
  /users/{userId}/orders:
    get:
      operationId: getUserOrders
      summary: 获取用户订单列表
      parameters:
        - name: userId
          in: path
          required: true
          schema:
            type: integer
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, completed, cancelled]
      responses:
        '200':
          description: 成功返回订单列表
```

```json
// 转换后的 FC 工具定义
{
  "name": "getUserOrders",
  "description": "获取用户订单列表",
  "parameters": {
    "type": "object",
    "properties": {
      "userId": { "type": "integer", "description": "用户 ID" },
      "status": { 
        "type": "string", 
        "enum": ["pending", "completed", "cancelled"],
        "description": "订单状态（可选）"
      }
    },
    "required": ["userId"]
  }
}
```

## 历史背景

传统 API 设计流程中，OpenAPI 是"先有定义、后有实现"的契约式设计工具。到了 Agent 时代，OpenAPI 文档可以被重复利用为 Agent 的"工具说明书"。

之前做 API+Agent 集成的方式：
1. 人工编写工具定义——重复劳动、易出错
2. 在 Agent 框架中手动注册工具——Scalability 差

## 核心优势

1. **存量资产复用**：已有 OpenAPI 规范的组织可以自动化转换，无需从零编写工具定义
2. **标准化**：OpenAPI 是业界标准，工具链丰富
3. **双向同步**：API 变更 → OpenAPI 更新 → 工具定义自动同步

## 挑战

| 挑战 | 说明 | 缓解方案 |
|------|------|----------|
| 过于复杂 | 生产级 OpenAPI 可能包含大量细节 | 精简为 FC 子集 |
| 认证信息 | OpenAPI 含 API Key、OAuth 等信息 | 分离认证与定义 |
| 路径参数 | REST 路径参数需要特殊处理 | URL 模板注入 |
| 响应处理 | OpenAPI 定义的是请求，响应的 Schema 也需要 | 部分平台忽略响应定义 |

## 自动转换工具

```python
import json
from openapi_python_client import OpenAPI

def openapi_to_tools(openapi_spec: dict) -> list:
    tools = []
    for path, methods in openapi_spec["paths"].items():
        for method, detail in methods.items():
            tool = {
                "name": detail.get("operationId", f"{method}_{path}"),
                "description": detail.get("summary", detail.get("description", "")),
                "parameters": {"type": "object", "properties": {}, "required": []}
            }
            # 转换路径参数 + 查询参数
            for param in detail.get("parameters", []):
                param_name = param["name"]
                param_schema = param.get("schema", {"type": "string"})
                tool["parameters"]["properties"][param_name] = param_schema
                tool["parameters"]["properties"][param_name]["description"] = param.get("description","")
                if param.get("required"):
                    tool["parameters"]["required"].append(param_name)
            
            # 转换 requestBody
            if "requestBody" in detail:
                content = detail["requestBody"]["content"]
                # 简化为只处理 JSON body
                if "application/json" in content:
                    body_schema = content["application/json"]["schema"]
                    # 展平 body 参数或整体作为一个参数
                    # ...
            
            tools.append(tool)
    return tools
```

## 最佳实践

- 不要自动转换所有 API，选择性暴露——不是每个 API 都需要给 Agent
- 将复杂的 OpenAPI 组合（多级嵌套、循环引用）展开为扁平结构
- 对关键 API 手动优化描述，弥补自动生成的不足
