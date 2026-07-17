# 代码执行引擎（Python / SQL 沙箱隔离）

## 简单介绍

代码执行工具允许 Agent 运行 Python、SQL 等代码。这是 Agent 最强大的能力之一——它使 Agent 能够进行数据分析、计算、数据库查询等复杂操作。但这也是最危险的能力——不安全的代码执行可能导致系统被攻破。

## 为什么需要代码执行

Agent 需要代码执行来补足 LLM 的天然弱点：

| LLM 弱点 | 代码执行弥补 |
|----------|-------------|
| 数学计算不准 | 用 Python 做精确计算 |
| 无法处理结构化数据 | 用 pandas 分析数据 |
| 没有实时数据 | 用 SQL 查询数据库 |
| 知识截止 | 用网络爬虫获取最新数据 |
| 无法生成图表 | 用 matplotlib 生成可视化 |

## 沙箱架构

```
Agent → Code 工具 → 沙箱环境
                      ├── 容器（Docker）
                      ├── 子进程（subprocess + seccomp）
                      ├── WebAssembly
                      └── 云函数（AWS Lambda / Cloudflare Workers）
```

### Docker 沙箱

```python
import docker

class DockerSandbox:
    def __init__(self):
        self.client = docker.from_env()
    
    async def execute_python(self, code: str, timeout: int = 30) -> dict:
        """在 Docker 容器中执行 Python 代码"""
        try:
            container = self.client.containers.run(
                image="python:3.11-slim",
                command=["python", "-c", code],
                mem_limit="256m",
                cpu_quota=50000,  # 0.5 CPU
                network_disabled=True,  # 禁止网络
                read_only=True,  # 只读文件系统
                remove=True,
                timeout=timeout,
            )
            output = container.decode("utf-8")
            return {"success": True, "output": output}
        except docker.errors.ContainerError as e:
            return {"success": False, "error": str(e.stderr)}
        except Exception as e:
            return {"success": False, "error": str(e)}
```

### 子进程沙箱（轻量）

```python
import subprocess
import tempfile

class SubprocessSandbox:
    async def execute_python(self, code: str, timeout: int = 30) -> dict:
        with tempfile.NamedTemporaryFile(suffix=".py", mode="w", delete=False) as f:
            f.write(code)
            f.flush()
            
            try:
                result = subprocess.run(
                    ["python", "-I", f.name],  # -I: 隔离模式
                    capture_output=True,
                    text=True,
                    timeout=timeout,
                    env={"PATH": "/usr/bin"},  # 最小环境
                )
                return {
                    "stdout": result.stdout,
                    "stderr": result.stderr,
                    "returncode": result.returncode
                }
            except subprocess.TimeoutExpired:
                return {"error": "代码执行超时"}
            finally:
                os.unlink(f.name)
```

## SQL 执行

```python
class SQLExecutor:
    def __init__(self, database_url: str):
        self.engine = create_engine(database_url)
    
    async def execute_sql(self, sql: str, max_rows: int = 100) -> dict:
        """执行 SQL 查询，返回结果"""
        
        # 安全检查：只允许 SELECT 查询
        if not sql.strip().upper().startswith("SELECT"):
            return {"error": "只允许执行 SELECT 查询"}
        
        try:
            with self.engine.connect() as conn:
                result = conn.execute(text(sql))
                rows = [dict(row) for row in result.fetchmany(max_rows)]
                return {
                    "rows": rows,
                    "row_count": len(rows),
                    "columns": list(rows[0].keys()) if rows else [],
                    "truncated": result.fetchone() is not None  # 还有更多数据
                }
        except Exception as e:
            return {"error": str(e)}
```

## 安全限制清单

| 限制维度 | 策略 | 说明 |
|---------|------|------|
| 系统调用 | 禁用 os.system, subprocess | 防止执行任意命令 |
| 文件系统 | 只读或临时文件系统 | 防止修改系统文件 |
| 网络 | 禁止出站网络连接 | 防止数据外泄 |
| 进程 | 限制 CPU/内存 | 防止资源耗尽 |
| 时间 | 强制超时 | 防止死循环 |
| 模块 | 白名单模块列表 | 只允许安全的模块 |
| 导入 | 禁用危险模块 | os, sys, shutil, requests |

## 核心挑战

1. **安全性 vs 功能性**：限制越多越安全，但 Agent 的能力也越受限——需要精细权衡
2. **状态持久化**：每次执行是独立的，但数据分析通常需要多步骤（导入 → 清洗 → 分析 → 可视化）
3. **结果大小**：代码执行可能产生大量输出（DataFrame、图片）——需要处理大结果
4. **环境一致性**：沙箱环境与生产环境可能不同，导致代码在沙箱中能跑但在生产中不能

## 最佳实践

- 代码执行沙箱必须有网络隔离，防止数据泄露
- 设置合理的资源限制（CPU 0.5 核、内存 256MB、超时 30s）
- SQL 工具默认限制为只读（SELECT），写入操作需要额外确认
- 对生成的图表，用 Base64 嵌入结果中返回给 LLM
- 记录所有执行过的代码，用于审计和安全审查
