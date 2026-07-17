# 13.1.1 docker-deploy — Docker 部署

**Docker 是 Agent 生产化部署的起点。** 它解决了"在我的机器上能跑"的问题，将 Agent 及其依赖打包成标准化的可运行单元。对于 AI Agent 来说，Docker 部署的挑战不仅在于 Python/Node.js 运行时，还包括：LLM 客户端的网络访问、工具执行环境隔离、记忆存储的卷挂载、以及镜像大小的控制。

## 背景与问题

### 之前是怎么做的？
早期的 Agent Demo 直接在宿主机上运行，依赖手动安装 Python 包、配置 API Key、管理虚拟环境。每次环境搭建都是"考古现场"——为了跑一个 Agent，需要在 README 里写 20 步安装指南。

### 这样做的问题
- **环境不一致**：开发者的 Mac 和生产的 Linux 可能存在微妙差异
- **依赖冲突**：Agent 框架（LangChain/CrewAI）与工具库（Playwright/Pandas）的依赖可能冲突
- **分发困难**：无法快速复制和扩展 Agent 实例
- **回滚复杂**：代码回滚了但依赖可能没回滚

## Docker 部署 Agent 的核心矛盾

```
需求                         现实
────                          ────
Agent 镜像要尽量小             Python + PyTorch + Playwright 轻松 2GB+
Agent 启动要快                 庞大镜像导致拉取和启动慢
Agent 需要网络访问 LLM API     容器网络配置复杂
Agent 需要持久化记忆           容器默认是 ephemeral 的
Agent 需要执行用户代码          需要安全的执行沙箱
```

## Dockerfile 最佳实践

### 多阶段构建

```dockerfile
# === Stage 1: 构建依赖 ===
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .

# 先安装依赖层（利用 Docker 缓存）
RUN pip install --user --no-cache-dir -r requirements.txt

# === Stage 2: 运行时 ===
FROM python:3.11-slim AS runtime

# 安全更新
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# 创建非 root 用户
RUN groupadd -r agent && useradd -r -g agent -m -d /home/agent agent

# 从 builder 复制预构建的依赖
COPY --from=builder /root/.local /home/agent/.local
ENV PATH=/home/agent/.local/bin:$PATH

WORKDIR /app
COPY --chown=agent:agent src/ ./src/
COPY --chown=agent:agent agent_config.yaml .

USER agent
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')"

ENTRYPOINT ["python", "-m", "src.main"]
```

### 依赖分层缓存

```dockerfile
# 依赖分三层缓存，加速 CI/CD 构建
# Layer 1: 系统依赖（极少变化）
RUN apt-get update && apt-get install -y \
    curl git && rm -rf /var/lib/apt/lists/*

# Layer 2: 核心框架依赖（中等频率变化）
COPY requirements.core.txt .
RUN pip install --no-cache-dir -r requirements.core.txt

# Layer 3: 应用依赖（频繁变化）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

### Node.js Agent 示例

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY src/ ./src/

RUN addgroup -S agent && adduser -S agent -G agent
USER agent

EXPOSE 3000
HEALTHCHECK --interval=30s --cmd="wget --no-verbose --tries=1 http://localhost:3000/health || exit 1"
CMD ["node", "src/server.js"]
```

## Docker Compose 多服务编排

Agent 系统通常需要多个依赖服务协同工作：

```yaml
# docker-compose.yml
version: "3.9"

services:
  # ─── Agent 服务 ───
  agent:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_URL=redis://redis:6379
      - DATABASE_URL=postgresql://agent:password@postgres:5432/agent_db
      - LOG_LEVEL=INFO
      - AGENT_MAX_STEPS=20
      - AGENT_CONTEXT_LIMIT=128000
    volumes:
      - agent_data:/app/data
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: "4G"
        reservations:
          cpus: "0.5"
          memory: "1G"
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')"]
      interval: 30s
      retries: 3
      timeout: 10s

  # ─── 记忆存储 ───
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      retries: 5

  # ─── 持久化存储 ───
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: agent_db
      POSTGRES_USER: agent
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U agent -d agent_db"]
      interval: 10s
      retries: 5

  # ─── 向量数据库（可选）───
  qdrant:
    image: qdrant/qdrant:v1.9
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage
    restart: unless-stopped

  # ─── 工具执行沙箱（可选）───
  sandbox:
    image: python:3.11-slim
    command: ["sleep", "infinity"]
    volumes:
      - sandbox_workspace:/workspace
    restart: unless-stopped
    # 隔离：只读根文件系统
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

volumes:
  agent_data:
  redis_data:
  postgres_data:
  qdrant_data:
  sandbox_workspace:
```

## 高级 Docker 配置

### .dockerignore

```dockerignore
__pycache__/
*.pyc
.git/
.env
.env.local
.vscode/
.idea/
*.md
tests/
docs/
data/
notebooks/
*.sqlite3
```

### 启动脚本 (entrypoint.sh)

```bash
#!/bin/bash
set -e

# 等待依赖服务就绪
echo "Waiting for Redis..."
while ! nc -z redis 6379; do sleep 1; done

echo "Waiting for PostgreSQL..."
while ! nc -z postgres 5432; do sleep 1; done

# 运行数据库迁移（如果有）
python -m src.migrations

# 启动 Agent 服务
exec python -m src.main
```

### 日志配置

确保日志输出到 stdout/stderr 以便 Docker 日志驱动收集：

```python
# logging_config.py
import logging
import sys

def setup_logging():
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(logging.Formatter(
        '{"time": "%(asctime)s", "level": "%(levelname)s", '
        '"name": "%(name)s", "message": "%(message)s"}'
    ))
    logging.basicConfig(level=logging.INFO, handlers=[handler])
```

### 优化镜像大小的关键技巧

```
初始镜像 (无优化)       ~2.5 GB    ❌ 不可接受
├── 使用 slim 基础镜像  ~1.2 GB    ↓
├── 多阶段构建          ~800 MB    ↓
├── 删除构建依赖        ~600 MB    ↓
├── 使用 --no-cache-dir ~500 MB    ↓
└── 最终优化            ~400 MB    ✅ 可接受
```

```dockerfile
# Agent 镜像大小对比 (以 Python Agent 为例)
# ──────────────────────────────────
# python:3.11                 → ~900 MB (基础镜像太大)
# python:3.11-slim           → ~120 MB (好多了)
# 安装 LangChain + OpenAI    → +200 MB
# 安装 Playwright + Chromium → +500 MB (浏览器是镜像膨胀的元凶)
# 安装 Pandas + NumPy        → +100 MB
# ──────────────────────────────────
# 不含浏览器的 Agent         → ~400 MB ✅
# 含浏览器的 Agent           → ~900 MB ⚠️ 考虑拆分
```

## Agent 特有的 Docker 考虑

### 1. API Key 安全
```yaml
# ❌ 不要：写在 Dockerfile 里
ENV OPENAI_API_KEY=sk-xxx

# ✅ 正确：运行时注入
environment:
  - OPENAI_API_KEY=${OPENAI_API_KEY}
# 或在 docker-compose 使用 .env 文件
```

### 2. 流式响应
Agent 的 SSE/WebSocket 流式响应需要容器配置正确的代理头：

```yaml
# nginx 反向代理配置
proxy_buffering off;
proxy_cache off;
proxy_read_timeout 300s;  # Agent 长思考需要超时
proxy_set_header Connection '';
chunked_transfer_encoding on;
```

### 3. 内存限制
Agent 的上下文窗口可能导致内存激增：

```python
# 在启动时设置内存限制
import resource

# 设置最大内存 (以字节为单位)
memory_limit = 4 * 1024 * 1024 * 1024  # 4GB
resource.setrlimit(resource.RLIMIT_AS, (memory_limit, memory_limit))
```

### 4. 健康检查端点

```python
# health.py
from fastapi import APIRouter
import time

router = APIRouter()
start_time = time.time()

@router.get("/health")
async def health():
    return {
        "status": "healthy",
        "uptime": time.time() - start_time,
        "version": "1.0.0",
        "dependencies": {
            "redis": await check_redis(),
            "postgres": await check_postgres(),
        }
    }

@router.get("/readiness")
async def readiness():
    # Agent 是否准备好处理请求
    llm_available = await check_llm_api()
    return {"ready": llm_available}
```

## 能力边界与结果

**Docker 部署能解决的问题：**
- ✅ 环境一致性：同一镜像在开发/测试/生产完全一致
- ✅ 依赖隔离：每个 Agent 独立打包，无冲突
- ✅ 快速部署：`docker pull` + `docker run` 即用
- ✅ 水平扩展：多容器 + 反向代理

**Docker 部署的边界：**
- ❌ 单机局限：无法自动跨多台机器
- ❌ 手动编排：没有自动伸缩/自愈/服务发现
- ❌ 状态管理：容器重启后内存中的对话丢失
- ❌ 监控薄弱：缺少内置的集群级监控

**何时使用 Docker：**
- MVP / 原型验证阶段
- 日均请求量 < 1000 的场景
- 内部工具 / 团队使用的 Agent
- 作为 K8s 部署的镜像构建层

**何时升级到 K8s / Serverless：**
- 需要自动伸缩应对流量波动
- 需要保证高可用 (99.9%+)
- 多服务协作的复杂 Agent 系统
- 需要滚动更新/金丝雀发布
