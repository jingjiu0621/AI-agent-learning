# 网络工具（HTTP 请求、爬虫、API 调用）

## 简单介绍

网络工具赋予 Agent 访问互联网的能力——调用外部 API、抓取网页内容、与 Web 服务交互。这是 Agent 从"封闭系统"走向"开放系统"的关键能力，也是最容易出现安全问题的工具类型。

## 基本工具集

```python
web_tools = [
    {
        "name": "http_request",
        "description": "发送 HTTP 请求到指定 URL。适用于调用 REST API、获取网页内容。",
        "parameters": {
            "properties": {
                "url": {"type": "string", "description": "请求 URL"},
                "method": {"type": "string", "enum": ["GET", "POST", "PUT", "DELETE"]},
                "headers": {"type": "object", "description": "请求头（可选）"},
                "body": {"type": "object", "description": "请求体（可选，用于 POST/PUT）"}
            },
            "required": ["url"]
        }
    },
    {
        "name": "web_search",
        "description": "搜索互联网信息，返回搜索结果列表。",
        "parameters": {
            "properties": {
                "query": {"type": "string", "description": "搜索关键词"}
            }
        }
    },
    {
        "name": "web_fetch",
        "description": "获取网页内容并转换为 Markdown 格式。",
        "parameters": {
            "properties": {
                "url": {"type": "string", "description": "网页 URL"}
            }
        }
    }
]
```

## 安全设计

网络工具面临最多的安全问题：

### 1. URL 过滤

```python
class URLFilter:
    BLOCKED_DOMAINS = [
        "localhost", "127.0.0.1", "0.0.0.0",
        "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",
        # ... 内网地址
    ]
    BLOCKED_SCHEMES = ["file://", "ftp://", "dict://"]
    
    def validate(self, url: str) -> bool:
        """验证 URL 是否可以访问"""
        parsed = urlparse(url)
        if parsed.scheme in self.BLOCKED_SCHEMES:
            return False
        if parsed.hostname in self.BLOCKED_DOMAINS:
            return False
        return True
```

### 2. SSRF 防护

SSRF（Server-Side Request Forgery）是网络工具最大的安全风险——Agent 可能被诱导访问内部服务。

### 3. 请求限制

```python
class WebToolConfig:
    max_response_size = 2 * 1024 * 1024  # 2MB
    max_redirects = 5
    timeout = 30.0  # 秒
    allowed_ports = [80, 443, 8080, 3000, 5000]  # 只允许特定端口
```

## 结果处理

### HTML 转 Markdown

```python
import html2text

async def fetch_as_markdown(url: str) -> str:
    """获取网页并转为 Markdown 格式"""
    html = await fetch_raw(url)
    converter = html2text.HTML2Text()
    converter.ignore_links = False
    converter.ignore_images = True
    markdown = converter.handle(html)
    
    # 限制输出长度
    if len(markdown) > 10000:
        markdown = markdown[:10000] + "\n\n... [内容截断]"
    return markdown
```

### API 响应处理

```python
async def call_api(url: str, method: str, headers: dict = None, body: dict = None) -> dict:
    """调用 API 并处理响应"""
    try:
        async with httpx.AsyncClient(timeout=30.0) as client:
            response = await client.request(method, url, headers=headers, json=body)
            
            result = {
                "status": response.status_code,
                "headers": dict(response.headers),
            }
            
            content_type = response.headers.get("content-type", "")
            if "application/json" in content_type:
                result["data"] = response.json()
            else:
                text = response.text
                result["data"] = text[:5000]  # 限制文本大小
            
            return result
    except httpx.TimeoutException:
        return {"error": "请求超时", "status": 0}
    except Exception as e:
        return {"error": str(e), "status": 0}
```

## 与文件工具的区别

| 维度 | 文件工具 | 网络工具 |
|------|---------|----------|
| 延迟 | 低（本地） | 高（网络不确定） |
| 可靠性 | 高 | 低（网络抖动） |
| 安全风险 | 路径遍历 | SSRF、数据泄露 |
| 大小限制 | 文件系统限制 | 带宽限制 |
| 认证 | 系统权限 | API Key / OAuth |

## 核心挑战

1. **网络延迟和不可靠**：请求可能超时、重定向、被限流——需要健壮的容错
2. **内容繁多**：网页包含大量无关内容（广告、导航），需要提取核心信息
3. **认证处理**：调用需要认证的 API 时，Token/Key 的管理复杂
4. **Cookie/Session**：需要模拟浏览器行为的场景下，会话管理复杂

## 最佳实践

- 对不可信 URL 执行 SSRF 防护（禁止内网地址）
- 设置响应大小限制，防止大响应消耗过多 Token
- 对 HTML 内容做 Markdown 转换，减少 Token 消耗
- 重要的 API 调用建议 Agent 以 "只读" 操作优先，写入操作让用户确认
