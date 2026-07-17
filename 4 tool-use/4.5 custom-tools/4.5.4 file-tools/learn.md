# 文件系统工具（读/写/搜索/结构化）

## 简单介绍

文件系统工具是 Agent 与本地文件交互的基础能力。包括读取文件内容、写入文件、搜索文件、获取文件结构等。文件工具看似简单，但安全设计（路径遍历、权限）和经验设计（大文件处理、编码）是真正的挑战。

## 基本工具集

```python
# 文件工具的标准接口
file_tools = [
    {
        "name": "read_file",
        "description": "读取指定路径的文件内容。适用于文本文件、代码文件、配置文件等。",
        "parameters": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "文件路径"},
                "max_length": {"type": "integer", "description": "最大读取字符数，默认 5000"}
            }
        }
    },
    {
        "name": "write_file",
        "description": "写入内容到指定文件。如果文件存在则覆盖，不存在则创建。",
        "parameters": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "文件路径"},
                "content": {"type": "string", "description": "文件内容"}
            }
        }
    },
    {
        "name": "search_files",
        "description": "在指定目录中搜索匹配的文件。",
        "parameters": {
            "type": "object",
            "properties": {
                "pattern": {"type": "string", "description": "通配符模式，如 *.py, **/*.md"},
                "root_dir": {"type": "string", "description": "搜索根目录"}
            }
        }
    },
    {
        "name": "list_directory",
        "description": "列出指定目录的内容。",
        "parameters": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "目录路径"}
            }
        }
    }
]
```

## 安全设计

文件工具中最关键的是**安全**——Agent 不能读写不应该访问的文件。

### 路径遍历防护

```python
import os

class SecureFileTool:
    def __init__(self, allowed_root: str):
        self.allowed_root = os.path.abspath(allowed_root)
    
    def _resolve_path(self, user_path: str) -> str:
        """安全解析路径，防止路径遍历攻击"""
        # 1. 解析用户路径
        resolved = os.path.abspath(
            os.path.join(self.allowed_root, user_path)
        )
        # 2. 检查是否在允许范围内
        if not resolved.startswith(self.allowed_root):
            raise PermissionError(f"无权访问此路径: {user_path}")
        # 3. 检查是否存在
        # 4. 返回安全路径
        return resolved
```

### 路径遍历攻击示例

```
用户输入: ../../etc/passwd
解析后: /etc/passwd (如果在 Linux 系统)
安全防护: resolved 必须以 allowed_root 开头
```

## 大文件处理

```python
async def read_file_safe(path: str, max_length: int = 5000) -> str:
    """安全读取文件，限制最大读取量"""
    file_size = os.path.getsize(path)
    if file_size > max_length * 4:  # 粗略估计
        # 文件太大，只读取前 max_length 个字符
        with open(path, 'r', encoding='utf-8', errors='replace') as f:
            content = f.read(max_length)
        return content + f"\n\n... [文件总大小 {file_size} 字节，仅显示前 {max_length} 字符]"
    else:
        with open(path, 'r', encoding='utf-8', errors='replace') as f:
            return f.read()
```

## 编码处理

```python
def read_with_fallback_encoding(path: str) -> str:
    """尝试多种编码读取文件"""
    encodings = ['utf-8', 'gbk', 'latin-1', 'shift-jis']
    for enc in encodings:
        try:
            with open(path, 'r', encoding=enc) as f:
                return f.read()
        except UnicodeDecodeError:
            continue
    return f"[无法解码文件: {path}]"
```

## 结构化文件支持

除了纯文本，文件工具还应支持常见结构化格式：

| 格式 | 读取策略 | 写入策略 |
|------|---------|----------|
| JSON | 解析为可读字符串 | 序列化为 JSON |
| YAML | 解析后呈现 | 序列化为 YAML |
| CSV | 表格形式展示 | 按行写入 |
| XML | 解析为树形结构 | 序列化为 XML |

## 核心挑战

1. **大文件**：Agent 不能读取上 GB 的文件——需要分块读取或摘要
2. **二进制文件**：图片、压缩包等不能直接读——需要特殊处理
3. **并发写入**：多个 Agent 同时写同一个文件——需要锁或版本控制
4. **文件编码**：各种编码的文件都可能遇到，需要自动检测

## 最佳实践

- 文件工具的根目录限定在项目目录内，禁止访问系统文件
- 对大文件提供"截断预览"而非全量读取，并在结果中提示截断信息
- 写操作前先让 LLM 确认内容；覆盖已有文件前询问用户确认
