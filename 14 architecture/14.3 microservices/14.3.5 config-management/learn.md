# 配置管理 — Configuration Management for Agent Microservices

> **核心思想**：配置管理是 Agent 系统的"中枢神经系统"——它决定了 Agent 使用哪个 LLM 模型、调用哪些工具、遵循哪些提示词模板、连接哪个向量数据库，以及如何控制成本和速率。在分布式 Agent 系统中，配置不仅包括传统的服务参数，还涉及 LLM 的模型选择策略、工具开关、Prompt 版本控制和权限策略，这使得配置管理比传统微服务复杂得多。

---

## 1. 基本原理

### 1.1 什么是分布式配置管理

在 Agent 微服务架构中，每个服务实例都需要配置来定义其行为。配置管理的核心目标是将**配置与代码分离**，使得配置可以独立于代码部署进行修改。

```
传统单体 Agent (配置在代码中):

  agent.py
  ├── LLM_API_KEY = "sk-xxx"          # 敏感信息硬编码
  ├── MODEL_NAME = "gpt-4"            # 模型选择硬编码
  ├── TOOL_LIST = ["search", "calc"]  # 工具列表硬编码
  ├── PROMPT_TEMPLATE = "你是..."      # 提示词硬编码
  └── MAX_TOKENS = 4096               # 参数硬编码

  → 修改任一配置需要: 改代码 → 提交 → CI/CD → 重启 → 全部实例

分布式配置管理 (配置与代码分离):

  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
  │ Agent 编排器     │     │ LLM 推理服务     │     │ 工具执行服务     │
  │                 │     │                 │     │                 │
  │ 启动时:          │     │ 启动时:          │     │ 启动时:          │
  │ GET /config     │     │ GET /config     │     │ GET /config     │
  │ → 获取完整的     │     │ → 获取 LLM 专    │     │ → 获取工具配     │
  │   Agent 配置     │     │   用配置         │     │   置             │
  └────────┬────────┘     └────────┬────────┘     └────────┬────────┘
           │                      │                       │
           └──────────────────────┼───────────────────────┘
                                  ▼
                        ┌─────────────────┐
                        │  配置中心         │
                        │  (Config Center) │
                        │                 │
                        │  • 配置存储       │
                        │  • 版本管理       │
                        │  • 动态推送       │
                        │  • 权限控制       │
                        │  • 配置校验       │
                        └─────────────────┘
                                  ▲
                                  │
                        ┌─────────────────┐
                        │  运维/开发人员    │
                        │  通过配置中心     │
                        │  管理配置         │
                        └─────────────────┘
```

### 1.2 Agent 配置的独特复杂性

与传统分布式系统不同，Agent 的配置管理面临更多维度的复杂性：

```
传统微服务配置 vs Agent 系统配置:

  传统微服务:
  ┌─────────────────────────────────────────┐
  │  数据库连接串: "postgres://user:pwd@..." │
  │  日志级别: "INFO"                        │
  │  服务端口: 8080                          │
  │  缓存大小: 256MB                         │
  │  超时时间: 30s                           │
  └─────────────────────────────────────────┘
  → 大多是静态的、确定的、不常变的值

  Agent 系统配置:
  ┌─────────────────────────────────────────┐
  │  LLM 提供方: openai/anthropic/local     │  ← 影响推理能力和成本
  │  模型名: gpt-4/claude-3-opus/llama-3   │  ← 影响质量
  │  Fallback 模型: gpt-4o-mini             │  ← 影响可用性
  │  温度参数: 0.7                           │  ← 影响创造力
  │  工具列表: [web_search, code_exec, ...]  │  ← 影响能力边界
  │  工具启用/禁用: {search: true, exec:true}│  ← 影响安全
  │  Prompt 模板 v3.2: "你是..."             │  ← 影响行为
  │  向量数据库: qdrant:6333                 │  ← 影响记忆
  │  速率限制: req/min = 100                 │  ← 影响成本控制
  │  API Key 轮换策略: 30天                  │  ← 影响安全性
  │  会话超时: 30min                         │  ← 影响用户体验
  │  推理预算: $0.05/次                       │  ← 影响成本
  └─────────────────────────────────────────┘
  → 动态变化频繁、多维度关联、且影响 Agent 行为
```

---

## 2. 背景与演进

配置管理经历了从"人工管理"到"声明式 GitOps"的演进：

```
演进路线:

  阶段 1                 阶段 2                 阶段 3                  阶段 4
  ┌──────────┐          ┌──────────┐          ┌──────────┐           ┌──────────┐
  │ 配置文件   │          │ 环境变量  │          │ 配置中心  │           │ GitOps   │
  │ (硬编码)  │────────►│ (环境)   │─────────►│ (APM/    │──────────►│ (声明式)  │
  │          │          │          │          │  Nacos)  │           │          │
  │          │          │          │          │          │           │          │
  └──────────┘          └──────────┘          └──────────┘           └──────────┘
       │                     │                     │                      │
       │ config.yaml         │ 12-Factor App       │ 动态热更新            │ ArgoCD /
       │ config.json         │ 环境隔离             │ 版本管理              │ Flux /
       │ 随代码发布           │ 敏感信息初步分离      │ 灰度发布              │ Kustomize
```

### 2.1 阶段一：配置文件

```
# config.yaml (随 Agent 代码一起发布)
agent:
  name: "customer-support-agent"
  llm:
    provider: "openai"
    model: "gpt-4"
    temperature: 0.7
    api_key: "sk-xxxxxxxx"       # 安全风险!
    max_tokens: 4096
  tools:
    - name: "web_search"
      enabled: true
      api_key: "search-xxxx"     # 安全风险!
    - name: "calculator"
      enabled: true
  prompts:
    system_prompt: "你是客服助手..."

问题:
  • 配置随代码发布, 修改配置需要完整 CI/CD 流程
  • 敏感信息 (API Key) 明文存储
  • 不同环境 (dev/staging/prod) 配置难以管理
  • 无法动态修改, 需要重启服务
  • 配置版本与代码版本耦合
```

### 2.2 阶段二：环境变量

```
# .env 文件 (从代码中分离)
LLM_PROVIDER=openai
LLM_MODEL=gpt-4
LLM_API_KEY=${LLM_API_KEY}    # 从 CI/CD Secret 注入
TOOL_SEARCH_ENABLED=true
TOOL_SEARCH_API_KEY=${SEARCH_API_KEY}
LOG_LEVEL=info

# docker-compose.yml
services:
  agent:
    image: agent:latest
    env_file:
      - .env.production         # 环境隔离
    environment:
      - LLM_MODEL=gpt-4o         # 可以覆盖

优势:
  • 配置与代码分离
  • 环境隔离 (不同 .env 文件)
  • 敏感信息通过 CI/CD Secret 注入

局限:
  • 修改配置需要重启服务
  • 大量的环境变量难以管理 (Agent 配置维度太多)
  • 没有版本管理
  • 无法动态热更新
```

### 2.3 阶段三：集中式配置中心

```
当前主流方案:

  运维人员修改配置
        │
        ▼
  ┌──────────────────┐     实时推送        ┌──────────────────┐
  │  配置中心          │──────────────────►│  Agent 实例 1     │
  │  (Apollo/Nacos)   │                   │                    │
  │                   │                    │  1. 监听配置变更    │
  │  ┌────────────┐  │                    │  2. 校验新配置      │
  │  │ 配置1: v3  │  │                    │  3. 加载新配置      │
  │  │ 配置2: v5  │  │                    │  4. 应用配置        │
  │  │ 配置3: v2  │  │                    │  5. 回执确认        │
  │  └────────────┘  │                    └──────────────────┘
  │  • 版本管理       │                        │
  │  • 灰度发布       │   Watch / Long Polling  │
  │  • 回滚           │◄───────────────────────┘
  │  • 权限控制       │
  │  • 变更审计       │
  └──────────────────┘

核心能力:
  • 动态修改: 不重启服务, 实时生效
  • 版本管理: 每次修改生成新版本, 支持回滚
  • 灰度发布: 先推送给 10% 实例验证
  • 配置校验: 修改时自动检查配置格式
  • 变更审计: 谁在什么时间修改了什么
```

### 2.4 阶段四：GitOps 声明式配置

```
GitOps 配置管理流程:

  开发人员提交 PR → Git 仓库 (配置即代码)
        │
        ▼
  ┌──────────────────────┐    自动同步      ┌──────────────────┐
  │  Git Repository      │────────────────►│  Agent 集群       │
  │                      │                 │                   │
  │  config/             │   ArgoCD / Flux  │  ┌─────────────┐ │
  │  ├── base/           │                 │  │ Agent 实例   │ │
  │  │   └── agent.yaml  │                 │  │ └─ 从 Git  │ │
  │  ├── overlays/       │                 │  │    读取配置  │ │
  │  │   ├── dev/        │                 │  └─────────────┘ │
  │  │   ├── staging/    │                 │  ┌─────────────┐ │
  │  │   └── prod/       │                 │  │ Agent 实例   │ │
  │  └── secrets/        │                 │  └─────────────┘ │
  │      └── .sops.yaml  │                 └──────────────────┘
  └──────────────────────┘

优势:
  • 所有配置变更通过 Git PR 审查
  • 完整的变更历史
  • 声明式: 期望状态与实际状态自动对齐
  • 天然支持环境隔离 (Kustomize Overlays)
  • 与 CI/CD 集成

局限:
  • 配置变更不是"实时"的 (取决于同步周期)
  • 不适合频繁变动的配置
  • Agent 配置的频繁变更可能造成 Git 仓库噪音
```

---

## 3. Agent 配置的特殊维度

Agent 系统的配置管理需要处理传统微服务没有的独特配置类型：

### 3.1 LLM 模型选型与降级策略

```yaml
# llm-config.yaml —— Agent 的 LLM 配置是立体化的
llm:
  # 主模型: 高能力, 高成本
  primary:
    provider: "anthropic"
    model: "claude-3-opus-20251022"
    endpoint: "https://api.anthropic.com/v1/messages"
    max_tokens: 8192
    temperature: 0.3
    timeout_seconds: 60

  # 快速模型: 低延迟, 低成本 (处理简单任务)
  fast:
    provider: "anthropic"
    model: "claude-3-haiku-20251022"
    endpoint: "https://api.anthropic.com/v1/messages"
    max_tokens: 2048
    temperature: 0.0

  # 降级策略: 当主模型不可用或超时时的自动降级链
  fallback_chain:
    - model: "claude-3-sonnet-20251022"      # 1st fallback: 降速但保留能力
      condition: "timeout > 30s || rate_limited"
    - model: "gpt-4o"                         # 2nd fallback: 跨提供商降级
      condition: "primary unavailable || 5xx"
    - model: "gpt-4o-mini"                    # 3rd fallback: 最终备选
      condition: "all above unavailable"
    - model: "local-llama-3-8b"              # 最后防线: 本地模型
      condition: "network isolated"

  # 路由规则: 什么任务用什么模型
  routing_rules:
    - task_type: "simple_qa"
      model: "fast"                # 简单问答用快速模型
    - task_type: "code_generation"
      model: "primary"             # 代码生成用最强模型
    - task_type: "summarization"
      model: "sonnet"              # 摘要用中等模型
    - task_type: "translation"
      model: "fast"                # 翻译用快速模型足够

  # 成本控制
  cost_control:
    max_daily_cost_usd: 100.0
    max_cost_per_session: 0.50
    budget_alert_threshold: 0.8   # 达到预算 80% 时告警
    hard_cap_enabled: true        # 达到上限时停止服务
```

### 3.2 工具注册与开关配置

```yaml
# tools-config.yaml —— Agent 的工具配置
tools:
  # 白名单模式: 只有列表中的工具可用
  enabled_tools:
    - "web_search"
    - "calculator"
    - "code_interpreter"
    - "file_reader"
  
  # 黑名单模式: 除列表外都可用 (用在受限场景)
  disabled_tools:
    - "shell_execution"    # 高危工具, 默认禁用
    - "file_deletion"      # 破坏性操作
  
  # 每个工具的具体配置
  tool_configs:
    web_search:
      provider: "bing"
      api_key_ref: "secret/tools/bing-api-key"   # 引用密钥管理
      max_results: 5
      timeout_seconds: 10
      rate_limit: 30/min
      enabled_for_roles: ["admin", "agent"]       # 角色权限

    code_interpreter:
      runtime: "docker"                           # 容器化执行
      image: "sandbox/python:3.11"
      memory_limit_mb: 512
      timeout_seconds: 30
      network_enabled: false                      # 隔离网络
      allowed_packages:                           # 允许的包列表
        - "pandas"
        - "numpy"
        - "matplotlib"
      enabled: true

    file_reader:
      allowed_paths:
        - "/data/shared/*"                        # 只允许读共享目录
        - "/tmp/agent-workspace/*"
      max_file_size_mb: 10
      max_files_per_session: 5
      enabled: true

  # 工具的动态发现配置
  dynamic_discovery:
    enabled: true
    registry_url: "http://tool-registry:8080"
    refresh_interval: 60                          # 秒
    auto_enable_new_tools: false                  # 新工具需要审批
```

### 3.3 Prompt 模板版本管理

Prompt 是 Agent 行为的核心定义，需要像代码一样进行版本管理：

```
Prompt 的配置化管理:

  ┌──────────────────────────────────────────────┐
  │  Prompt 模板仓库 (Git)                         │
  │                                               │
  │  prompts/                                     │
  │  ├── system/                                  │
  │  │   ├── v1.0_base.md                        │
  │  │   ├── v2.0_add_tools.md                    │
  │  │   └── v3.0_security.md                     │  ← 当前版本
  │  ├── task_specific/                           │
  │  │   ├── coding_assistant/                     │
  │  │   │   ├── v1.0.md                          │
  │  │   │   └── v1.1_fix_security.md             │
  │  │   └── customer_service/                     │
  │  │       └── v2.0.md                          │
  │  └── few_shot/                                │
  │      ├── v1.0_examples.json                    │
  │      └── v2.0_more_examples.json               │
  └──────────────────┬───────────────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────────────┐
  │  Prompt 版本元数据 (配置中心)                   │
  │                                               │
  │  prompt_version: {                            │
  │    "system": "v3.0",                          │
  │    "coding": "v1.1",                          │
  │    "customer": "v2.0",                        │
  │    "active_config": "prod-v12"                │
  │  }                                             │
  └──────────────────────────────────────────────┘
                     │
                     ▼
            ┌─────────────────┐
            │ Agent 实例       │
            │                 │
            │ 启动/运行时:      │
            │ 读取版本配置      │
            │ → 从仓库拉取     │
            │   对应版本的      │
            │   Prompt 内容    │
            └─────────────────┘
```

### 3.4 记忆系统与向量存储配置

```yaml
# memory-config.yaml
memory:
  # 短期记忆 (对话上下文)
  short_term:
    type: "in_memory"
    max_turns: 50                     # 保留最近 50 轮对话
    max_tokens: 16000                 # 控制上下文窗口

  # 长期记忆 (向量数据库)
  long_term:
    type: "vector_store"
    provider: "qdrant"
    connection:
      host: "qdrant.agent-system.svc.cluster.local"
      port: 6333
      grpc_port: 6334
      api_key_ref: "secret/vector/qdrant-key"
      tls_enabled: true
    
    collection: "agent-memories"
    embedding:
      model: "text-embedding-3-large"       # 嵌入模型
      dimensions: 3072
    indexing:
      strategy: "hnsw"                       # HNSW 索引
      m: 16                                   # HNSW 参数
      ef_construction: 200
    
    search:
      top_k: 10
      score_threshold: 0.75                  # 相似度阈值
      hybrid_search: true                    # 混合搜索: 向量 + BM25

  # 工作记忆 (中间推理状态)
  working:
    type: "redis"
    redis_url: "redis://redis.agent-system:6379/0"
    ttl_seconds: 3600                       # 1小时后自动过期
    max_sessions: 10000

  # 外部知识图谱
  knowledge_graph:
    type: "neo4j"
    uri: "bolt://neo4j.agent-system:7687"
    database: "agent-knowledge"
```

### 3.5 速率限制与成本配额

```yaml
# rate-limit-config.yaml
rate_limiting:
  # 全局限制
  global:
    requests_per_minute: 1000
    tokens_per_minute: 500000
    concurrent_sessions: 200

  # 按用户/租户限制
  per_tenant:
    default:
      requests_per_minute: 100
      tokens_per_day: 10000000                # 每日 Token 预算
      max_concurrent: 20
      cost_per_session_usd: 1.00              # 单次会话成本上限
    
    premium_tenant:
      requests_per_minute: 500
      tokens_per_day: 50000000
      max_concurrent: 100
      cost_per_session_usd: 5.00

  # 模型级别限制
  per_model:
    claude-3-opus:
      requests_per_minute: 100
      rpm_burst: 200                          # 突发限额
    gpt-4o:
      requests_per_minute: 500
    local-model:
      requests_per_minute: 2000               # 本地模型无限制

  # 成本配额
  cost_quotas:
    enabled: true
    daily_budget_usd: 500.00
    monthly_budget_usd: 10000.00
    
    # 触发策略
    actions_on_exceed:
      - threshold: 0.8                         # 80% 预算
        action: "log_warning"
      - threshold: 0.95                        # 95% 预算
        action: "switch_to_cheap_model"
      - threshold: 1.0                         # 100%
        action: "reject_non_essential"
    
    cost_tracking:
      per_session: true
      per_user: true
      per_tool: true                           # 按工具跟踪成本
```

---

## 4. 配置管理方案对比

| 特性 | etcd | Consul KV | ZooKeeper | K8s ConfigMap | Apollo | Nacos | AWS AppConfig |
|------|------|-----------|-----------|---------------|--------|-------|---------------|
| **存储模型** | 分层 KV | 分层 KV | 树形 ZNode | KV (扁平) | Namespace+Key | Namespace+Group | Application+Profile |
| **一致性** | Raft 强一致 | Raft 强一致 | ZAB 强一致 | 最终一致 (etcd 后端) | 最终一致 | Raft v3 | 最终一致 |
| **热更新** | Watch API | Watch API | Watch API | Volume 挂载/滚动 | 长轮询推送 | HTTP 长轮询 | 托管轮询 |
| **版本管理** | 修订号 | 无 | 无 | 无 | 全量版本 | 版本记录 | 版本 + 部署策略 |
| **回滚** | 手动 | 手动 | 手动 | 手动 | 一键回滚 | 一键回滚 | 自动回滚 |
| **权限管理** | RBAC 基础 | ACL | ACL | K8s RBAC | 应用+角色 | RBAC | IAM |
| **配置加密** | 需扩展 | 需扩展 | 需扩展 | Secret 资源 | 内建加密 | AES 加密 | KMS 集成 |
| **灰度发布** | 无 | 无 | 无 | 无 | 标签灰度 | 多场景 | 百分比部署 |
| **Agent 适配** | 低层 API | 低层 API | 低层 API | 仅 K8s | 企业级 | 阿里生态 | AWS 生态 |
| **运维复杂度** | 中 | 中 | 高 | 低 (K8s) | 中高 | 中 | 低 (托管) |
| **适合场景** | 基础设施 | 基础设施 | 分布式协调 | 纯 K8s 部署 | 中大型企业 | 阿里云 | AWS 云 |

### 方案选型速查

```
配置中心选型决策树:

你的 Agent 部署在哪里?
│
├── 纯 K8s 环境?
│     ├── 配置不敏感 → K8s ConfigMap (最简单)
│     ├── 配置敏感 → K8s Secret + SealedSecrets (加密)
│     ├── 需要热更新 → Reloader + ConfigMap (K8s 原生)
│     └── 复杂微服务 → K8s + 外挂 Apollo (独立配置中心)
│
├── 云原生 (AWS)?
│     ├── 简单需求 → AWS Parameter Store (免费)
│     ├── 复杂配置 → AWS AppConfig (托管配置)
│     └── 密钥管理 → AWS Secrets Manager + KMS
│
├── 阿里云?
│     └── → Nacos (阿里全家桶集成)
│
├── 自建 IDC?
│     ├── 团队熟悉 Java → Apollo (成熟、功能丰富)
│     ├── 团队熟悉 Go → Consul KV + Vault (轻量)
│     └── 需要共识 → etcd (基础组件)
│
└── 混合云?
      └── → Apollo (多数据中心支持) / 自定义 GitOps
```

---

## 5. 动态配置热更新

### 5.1 热更新架构

```
配置变更 → 生效的完整路径:

  运维修改配置
        │
        ▼
  ┌─────────────────┐     推送变更          ┌─────────────────┐
  │  配置中心         │─────────────────────►│  Watch 监听器    │
  │  (Apollo/Nacos)  │  1. 发布新版本         │                  │
  │                  │  2. 通知所有监听者      │  收到变更通知    │
  │                  │                      └────────┬────────┘
  └─────────────────┘                               │
                                                     ▼
                                            ┌─────────────────┐
                                            │  Config Loader   │
                                            │                  │
                                            │  1. 拉取新配置    │
                                            │  2. Schema 校验   │
                                            │  3. 对比差异      │
                                            └────────┬────────┘
                                                     │
                                                     ▼
                                            ┌─────────────────┐
                                            │  配置生效决策     │
                                            │                  │
                                            │  此配置支持热更新? │
                                            │  ├─ 是 → 应用新值 │
                                            │  └─ 否 → 记录等  │
                                            │       待重启     │
                                            └────────┬────────┘
                                                     │
                                                     ▼
                                            ┌─────────────────┐
                                            │  Agent 运行时     │
                                            │  使用新配置继续    │
                                            │  处理推理          │
                                            └─────────────────┘
```

### 5.2 Python 动态配置加载器

```python
# dynamic_config_loader.py
import json
import os
import asyncio
import logging
from typing import Any, Dict, Optional, Callable
from dataclasses import dataclass
from enum import Enum
import jsonschema  # 配置 Schema 校验

logger = logging.getLogger(__name__)


class ConfigReloadStrategy(Enum):
    """配置变更时的处理策略"""
    IMMEDIATE = "immediate"              # 立即生效
    NEXT_SESSION = "next_session"        # 下一个会话开始生效
    DRAIN_FIRST = "drain_first"          # 等待当前推理完成
    RESTART_REQUIRED = "restart"         # 需要重启


@dataclass
class ConfigEntry:
    """单个配置项的定义"""
    key: str
    value: Any
    version: int
    reload_strategy: ConfigReloadStrategy
    schema: Optional[Dict] = None
    validator: Optional[Callable] = None


class AgentConfigManager:
    """
    Agent 系统的动态配置管理器
    支持: 热更新、Schema 校验、版本追踪、变更回调
    """

    def __init__(self, config_center_url: str, app_id: str):
        self.config_center_url = config_center_url
        self.app_id = app_id
        self._configs: Dict[str, ConfigEntry] = {}
        self._listeners: Dict[str, list] = {}
        self._watch_tasks: list = []
        self._reload_strategies: Dict[str, ConfigReloadStrategy] = {}

    async def load_initial_config(self):
        """启动时加载全部配置"""
        logger.info(f"[Config] 加载初始配置, app_id={self.app_id}")

        # 模拟从配置中心加载
        initial_config = {
            "llm.model": "claude-3-opus-20251022",
            "llm.temperature": 0.3,
            "llm.max_tokens": 8192,
            "tools.enabled": json.dumps(["web_search", "calculator"]),
            "memory.type": "vector_store",
            "memory.top_k": 10,
            "rate_limit.rpm": 100,
            "system.prompt_version": "v3.0",
        }

        for key, value in initial_config.items():
            # 确定每种配置的热更新策略
            strategy = self._infer_reload_strategy(key)
            self._configs[key] = ConfigEntry(
                key=key,
                value=value,
                version=1,
                reload_strategy=strategy,
            )

        logger.info(f"[Config] 已加载 {len(self._configs)} 项配置")

    def _infer_reload_strategy(self, key: str) -> ConfigReloadStrategy:
        """根据配置 key 推断热更新策略"""
        # 立即生效的配置
        immediate_keys = [
            "llm.temperature",
            "rate_limit",
            "log_level",
            "tools.enabled",
        ]
        # 需要等下个会话的配置
        session_keys = [
            "llm.model",
            "system.prompt_version",
            "memory.type",
        ]
        # 需要重启的配置
        restart_keys = [
            "llm.provider",
            "memory.connection",
            "server.port",
        ]

        if any(k in key for k in immediate_keys):
            return ConfigReloadStrategy.IMMEDIATE
        elif any(k in key for k in session_keys):
            return ConfigReloadStrategy.NEXT_SESSION
        else:
            return ConfigReloadStrategy.RESTART_REQUIRED

    async def get(self, key: str, default: Any = None) -> Any:
        """获取配置值"""
        entry = self._configs.get(key)
        if entry is None:
            return default
        return self._parse_value(entry.value)

    async def get_model_config(self) -> Dict:
        """获取 LLM 模型配置组合"""
        return {
            "model": await self.get("llm.model"),
            "temperature": await self.get("llm.temperature"),
            "max_tokens": await self.get("llm.max_tokens"),
        }

    def _parse_value(self, value: str) -> Any:
        """尝试解析 JSON, 失败返回原字符串"""
        try:
            return json.loads(value)
        except (json.JSONDecodeError, TypeError):
            return value

    async def update_config(self, key: str, new_value: str, version: int):
        """配置更新入口 (由 Watch 回调触发)"""
        old_entry = self._configs.get(key)
        if old_entry and old_entry.version >= version:
            return  # 旧的版本, 忽略

        strategy = self._infer_reload_strategy(key)

        # 更新存储
        self._configs[key] = ConfigEntry(
            key=key,
            value=new_value,
            version=version,
            reload_strategy=strategy,
        )

        # 根据策略执行
        if strategy == ConfigReloadStrategy.IMMEDIATE:
            logger.info(f"[Config] 热更新: {key} = {new_value}")
            await self._notify_listeners(key, new_value)

        elif strategy == ConfigReloadStrategy.NEXT_SESSION:
            logger.info(f"[Config] 配置 {key} 将在下个会话生效")
            # 标记需要在下次会话开始时重新加载

        elif strategy == ConfigReloadStrategy.DRAIN_FIRST:
            logger.info(f"[Config] 等待当前推理完成, 然后应用 {key}")
            # 注册到"待应用队列"

        else:
            logger.info(f"[Config] 配置 {key} 需要重启应用")

        # 记录变更
        if old_entry:
            logger.info(f"[Config] 变更: {key} {old_entry.value} → {new_value}")

    async def watch_config(self, key: str, callback: Callable):
        """注册配置变更监听器"""
        if key not in self._listeners:
            self._listeners[key] = []
        self._listeners[key].append(callback)

    async def _notify_listeners(self, key: str, new_value: Any):
        """通知配置变更"""
        parsed = self._parse_value(new_value)
        for callback in self._listeners.get(key, []):
            try:
                await callback(key, parsed)
            except Exception as e:
                logger.error(f"[Config] 监听器执行失败: {key}: {e}")

    async def start_watch_loop(self, poll_interval: int = 10):
        """启动配置变更监听循环 (模拟长轮询)"""
        logger.info(f"[Config] 启动监听循环, 间隔={poll_interval}s")

        while True:
            try:
                # 模拟从配置中心获取变更
                # 实际实现: Apollo HTTP 长轮询 / Consul Watch / etcd Watch
                # changed = await self.config_center.fetch_changes(self.app_id)
                # for key, value, version in changed:
                #     await self.update_config(key, value, version)
                pass
            except Exception as e:
                logger.error(f"[Config] 监听出错: {e}")

            await asyncio.sleep(poll_interval)

    async def validate_config_schema(self, config: Dict) -> bool:
        """使用 JSON Schema 验证配置"""
        schema = {
            "type": "object",
            "properties": {
                "llm.model": {"type": "string"},
                "llm.temperature": {"type": "number", "minimum": 0, "maximum": 2},
                "llm.max_tokens": {"type": "integer", "minimum": 1, "maximum": 128000},
                "rate_limit.rpm": {
                    "type": "integer", "minimum": 1, "maximum": 10000
                },
                "system.prompt_version": {"type": "string", "pattern": "^v\\d+\\.\\d+$"},
            },
        }
        try:
            jsonschema.validate(config, schema)
            return True
        except jsonschema.ValidationError as e:
            logger.error(f"[Config] Schema 校验失败: {e}")
            return False
```

### 5.3 版本化配置与回滚

```python
class VersionedConfigManager:
    """
    带版本管理和回滚功能的配置管理器
    Agent 配置错误可能导致严重的功能异常, 版本管理至关重要
    """

    def __init__(self, config_manager: AgentConfigManager):
        self.config_mgr = config_manager
        self._history: Dict[str, list] = {}     # key → [(version, value, timestamp)]
        self._snapshots: Dict[int, Dict] = {}    # version → 全量快照
        self._current_version = 0

    async def apply_config_change(
        self,
        changes: Dict[str, str],
        reason: str = "",
        author: str = "",
    ) -> int:
        """
        应用一批配置变更, 生成新版本
        返回新版本号
        """
        self._current_version += 1
        version = self._current_version

        # 生成快照
        snapshot = {}
        for key, new_value in changes.items():
            old_entry = self.config_mgr._configs.get(key)
            old_value = old_entry.value if old_entry else None

            # 记录历史
            if key not in self._history:
                self._history[key] = []
            self._history[key].append({
                "version": version,
                "old_value": old_value,
                "new_value": new_value,
                "timestamp": asyncio.get_event_loop().time(),
                "reason": reason,
                "author": author,
            })

            # 更新配置
            await self.config_mgr.update_config(key, new_value, version)
            snapshot[key] = new_value

        # 保存全量快照
        self._snapshots[version] = snapshot
        logger.info(f"[Config] 版本 {version} 已应用: {len(changes)} 项变更")
        return version

    async def rollback_to_version(self, target_version: int) -> bool:
        """
        回滚到指定版本
        把当前版本和回滚版本之间的所有变更反向应用
        """
        if target_version not in self._snapshots:
            logger.error(f"[Config] 版本 {target_version} 不存在")
            return False

        target_config = self._snapshots[target_version]
        current_config = {}

        # 收集当前配置
        for key, entry in self.config_mgr._configs.items():
            current_config[key] = entry.value

        # 计算需要回滚的变更
        rollback_changes = {}
        for key, target_value in target_config.items():
            if current_config.get(key) != target_value:
                rollback_changes[key] = target_value

        if not rollback_changes:
            logger.info("[Config] 已是指定版本, 无需回滚")
            return True

        # 应用回滚
        await self.apply_config_change(
            rollback_changes,
            reason=f"回滚到版本 {target_version}",
            author="system",
        )

        logger.info(
            f"[Config] 已从版本 {self._current_version - 1} 回滚到 {target_version}"
        )
        return True

    async def get_change_history(self, key: str = None) -> list:
        """获取变更历史"""
        if key:
            return self._history.get(key, [])
        return self._history

    async def diff_versions(self, v1: int, v2: int) -> Dict:
        """比较两个版本的配置差异"""
        s1 = self._snapshots.get(v1, {})
        s2 = self._snapshots.get(v2, {})

        diff = {}
        all_keys = set(s1.keys()) | set(s2.keys())
        for key in all_keys:
            if s1.get(key) != s2.get(key):
                diff[key] = {
                    "from": s1.get(key, "<未定义>"),
                    "to": s2.get(key, "<未定义>"),
                }
        return diff
```

### 5.4 配置热更新在 Agent 中的安全策略

```
Agent 配置变更的安全边界:

  配置变更类型                       安全策略
  ┌─────────────────────────┐──────────────────────────┐
  │  LLM 模型切换            │  → 不中断当前推理          │
  │  (gpt-4 → claude-3)     │  → 下个会话生效            │
  │                         │  → 预热新模型再切换         │
  ├─────────────────────────┼──────────────────────────┤
  │  工具启用/禁用            │  → 当前推理继续使用旧工具集  │
  │  (禁用 web_search)       │  → 下个推理步骤使用新配置   │
  │                         │  → 检查没有正在使用的工具    │
  ├─────────────────────────┼──────────────────────────┤
  │  Prompt 版本切换          │  → 缓存旧 Prompt 直至会话结束 │
  │  (v3.0 → v3.1)          │  → 新会话使用新 Prompt     │
  │                         │  → A/B 测试验证效果         │
  ├─────────────────────────┼──────────────────────────┤
  │  API Key 轮换            │  → 双 Buffer 模式          │
  │  (更换 LLM Key)          │  → 旧 Key 保留 TTL 内有效  │
  │                         │  → 逐步切换到新 Key         │
  ├─────────────────────────┼──────────────────────────┤
  │  限流策略调整             │  → 立即生效 (需同步更新)    │
  │  (100 → 50 RPM)         │  → 通知所有实例             │
  │                         │  → 避免限流窗口内的不一致    │
  └─────────────────────────┴──────────────────────────┘
```

---

## 6. 敏感信息管理

### 6.1 API Key / Secret 加密存储

```python
# secret_manager.py
import os
import base64
import json
from typing import Optional, Dict
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC


class AgentSecretManager:
    """
    Agent 系统的敏感信息管理
    支持: 加密存储、自动轮换、访问审计
    """

    def __init__(self, master_key: Optional[str] = None):
        # 主密钥: 从环境变量或 Vault 获取
        key_material = master_key or os.environ.get("AGENT_MASTER_KEY")
        if not key_material:
            raise ValueError("需要 AGENT_MASTER_KEY 环境变量")

        # 派生加密密钥
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=b"agent-secret-salt",
            iterations=100000,
        )
        self.cipher = Fernet(base64.urlsafe_b64encode(kdf.derive(key_material.encode())))
        self._secret_cache: Dict[str, str] = {}

    def encrypt_secret(self, plain_text: str) -> str:
        """加密敏感信息"""
        return self.cipher.encrypt(plain_text.encode()).decode()

    def decrypt_secret(self, encrypted: str) -> str:
        """解密敏感信息"""
        # 检查缓存
        cache_key = encrypted[:32]  # 用密文前缀做缓存键
        if cache_key in self._secret_cache:
            return self._secret_cache[cache_key]

        decrypted = self.cipher.decrypt(encrypted.encode()).decode()
        self._secret_cache[cache_key] = decrypted
        return decrypted

    def clear_cache(self):
        """清理密钥缓存 (用于轮换后)"""
        self._secret_cache.clear()


# =====================
# HashiCorp Vault 集成
# =====================
import hvac

class VaultIntegration:
    """
    HashiCorp Vault 集成 — 生产级 Secret 管理
    """

    def __init__(self, vault_addr: str, role_id: str, secret_id: str):
        self.client = hvac.Client(url=vault_addr)
        # AppRole 认证
        self.client.auth.approle.login(
            role_id=role_id,
            secret_id=secret_id,
        )

    def get_llm_api_key(self, provider: str) -> str:
        """从 Vault 获取 LLM API Key"""
        secret = self.client.secrets.kv.v2.read_secret_version(
            path=f"agent/llm/{provider}/api-key",
            mount_point="secret",
        )
        return secret["data"]["data"]["value"]

    def rotate_llm_api_key(self, provider: str, new_key: str) -> bool:
        """轮换 API Key"""
        # 1. 写入新密钥 (保留历史版本)
        self.client.secrets.kv.v2.create_or_update_secret(
            path=f"agent/llm/{provider}/api-key",
            secret={"value": new_key},
            mount_point="secret",
        )
        # 2. 通知所有 Agent 实例加载新 Key
        # (通过配置中心的动态更新)
        return True

    def get_database_credentials(self, db_name: str) -> dict:
        """获取动态数据库凭证 (Vault 动态生成)"""
        creds = self.client.secrets.database.generate_credentials(
            name=f"agent-{db_name}-role",
        )
        return {
            "username": creds["data"]["username"],
            "password": creds["data"]["password"],
            "ttl": creds["data"]["ttl"],
        }
```

### 6.2 密钥轮换策略

```
两阶段轮换 (Zero-Downtime Rotation):

  阶段 1: 发布新密钥                  阶段 2: 回收旧密钥
  ┌────────────────────┐             ┌────────────────────┐
  │                    │             │                    │
  │  Vault:            │             │  Vault:            │
  │  key-v1 (active)   │             │  key-v1 (过期)      │
  │  key-v2 (staged)   │             │  key-v2 (active)   │
  │                    │             │                    │
  │  Agent 实例们:      │             │  Agent 实例们:      │
  │  ┌─ 实例 A: v1     │             │  ┌─ 实例 A: v2     │
  │  ├─ 实例 B: v1     │  TTL 过后   │  ├─ 实例 B: v2     │
  │  └─ 实例 C: v1     │  ────────►  │  └─ 实例 C: v2     │
  │                    │             │                    │
  │  通知实例加载 v2    │             │  安全:              │
  │  但 v1 仍然有效     │             │  v1 已全部替换      │
  └────────────────────┘             └────────────────────┘

  窗口期: 所有实例从 v1 切换到 v2 的时间
  - 如果实例离线, v1 保留 24h
  - 如果 v2 有问题, 回滚到 v1
```

```python
class KeyRotationManager:
    """自动密钥轮换管理器"""

    def __init__(self, vault: VaultIntegration, config_mgr: AgentConfigManager):
        self.vault = vault
        self.config_mgr = config_mgr
        self.rotation_intervals = {
            "llm-api-key": 30,       # 30 天
            "tool-api-key": 45,      # 45 天
            "db-password": 90,       # 90 天
            "jwt-secret": 180,       # 180 天
        }

    async def rotate_if_needed(self, secret_type: str, provider: str):
        """检查是否需要轮换密钥"""
        days_since_last = await self._days_since_last_rotation(secret_type, provider)
        interval = self.rotation_intervals.get(secret_type, 30)

        if days_since_last >= interval:
            logger.info(f"[Rotation] 需要轮换 {secret_type}/{provider}")
            # 生成新密钥
            new_key = self._generate_key()
            # 写入 Vault 的新版本
            self.vault.rotate_llm_api_key(provider, new_key)
            # 通知配置中心
            await self.config_mgr.update_config(
                f"secret.{provider}.api_key",
                f"vault:agent/llm/{provider}/api-key#v2",
                version=int(asyncio.get_event_loop().time()),
            )
            logger.info(f"[Rotation] {secret_type}/{provider} 轮换完成")

    def _generate_key(self) -> str:
        """生成安全的随机密钥"""
        import secrets
        return f"sk-{secrets.token_urlsafe(32)}"

    async def _days_since_last_rotation(self, secret_type: str, provider: str) -> int:
        """查询距离上次轮换的天数"""
        # 从 Vault 的 metadata 中读取
        return 0  # 简化实现
```

---

## 7. 代码示例

### 7.1 完整的配置加载与热更新系统

```python
# agent_config_system.py
"""
Agent 系统的完整配置管理示例
集成: 配置加载 + Schema 校验 + 热更新 + 敏感信息解密
"""

import asyncio
import json
import yaml
from pathlib import Path
from typing import Any, Dict, Optional, List
from pydantic import BaseModel, Field, validator


# =====================
# 配置 Schema 定义
# =====================

class LLMConfig(BaseModel):
    """LLM 配置 Schema"""
    provider: str = Field(..., pattern="^(openai|anthropic|local|azure)$")
    model: str = Field(..., min_length=3)
    temperature: float = Field(default=0.3, ge=0.0, le=2.0)
    max_tokens: int = Field(default=4096, ge=1, le=128000)
    api_key_ref: str = Field(..., pattern="^vault:|secret:|env:")
    timeout_seconds: int = Field(default=60, ge=1, le=300)

    @validator("api_key_ref")
    def validate_key_ref(cls, v):
        if not any(v.startswith(p) for p in ["vault:", "secret:", "env:"]):
            raise ValueError("api_key_ref 必须以 vault:/secret:/env: 开头")
        return v


class ToolConfig(BaseModel):
    """工具配置 Schema"""
    name: str
    enabled: bool = True
    timeout_seconds: int = Field(default=30, ge=1, le=120)
    rate_limit: str = Field(default="60/min", pattern=r"^\d+/(min|sec|hour)$")
    allowed_roles: List[str] = Field(default=["admin"])


class AgentConfig(BaseModel):
    """Agent 系统总配置 Schema"""
    app_id: str
    environment: str = Field(..., pattern="^(dev|staging|prod)$")
    version: str = Field(..., pattern=r"^\d+\.\d+\.\d+$")

    llm: Dict[str, LLMConfig]          # primary, fast, fallback...
    tools: List[ToolConfig]
    memory: Dict[str, Any] = Field(default_factory=dict)
    rate_limits: Dict[str, Any] = Field(default_factory=dict)

    prompt_version: str = Field(default="v1.0", pattern=r"^v\d+\.\d+$")
    log_level: str = Field(default="INFO", pattern="^(DEBUG|INFO|WARN|ERROR)$")

    class Config:
        extra = "forbid"  # 禁止未定义的字段


# =====================
# 配置加载器
# =====================

class AgentConfigLoader:
    """
    Agent 配置加载器
    支持: 多层配置合并 (Base + Environment + Override)
    """

    def __init__(
        self,
        secret_manager: AgentSecretManager,
        config_dir: str = "/etc/agent/config",
    ):
        self.secret_manager = secret_manager
        self.config_dir = Path(config_dir)
        self.active_config: Optional[AgentConfig] = None

    async def load(self, environment: str = "prod") -> AgentConfig:
        """加载配置: base.yaml + environment overlay"""
        # 1. 加载基础配置
        base_config = await self._load_yaml("base.yaml")

        # 2. 加载环境特定配置 (覆盖)
        env_config = await self._load_yaml(f"overlays/{environment}.yaml")

        # 3. 合并配置 (深度合并)
        merged = self._deep_merge(base_config, env_config)

        # 4. 解析敏感信息引用
        merged = await self._resolve_secrets(merged)

        # 5. Schema 校验
        config = AgentConfig(**merged)

        self.active_config = config
        logger.info(f"[Config] 加载完成: env={environment}, version={config.version}")
        return config

    async def reload(self, environment: str = "prod") -> AgentConfig:
        """重新加载配置"""
        logger.info("[Config] 重新加载配置...")
        return await self.load(environment)

    async def _load_yaml(self, path: str) -> Dict:
        """加载 YAML 配置文件"""
        full_path = self.config_dir / path
        if not full_path.exists():
            logger.warning(f"[Config] 文件不存在: {full_path}")
            return {}

        content = full_path.read_text(encoding="utf-8")
        return yaml.safe_load(content) or {}

    def _deep_merge(self, base: Dict, override: Dict) -> Dict:
        """深度合并两个配置字典"""
        result = base.copy()
        for key, value in override.items():
            if key in result and isinstance(result[key], dict) and isinstance(value, dict):
                result[key] = self._deep_merge(result[key], value)
            else:
                result[key] = value
        return result

    async def _resolve_secrets(self, config: Dict) -> Dict:
        """递归解析配置中的敏感信息引用"""
        if isinstance(config, dict):
            return {
                key: await self._resolve_secrets(value)
                for key, value in config.items()
            }
        elif isinstance(config, str):
            # 解析 vault:/secret:/env: 引用
            if config.startswith("env:"):
                env_var = config[4:]
                return os.environ.get(env_var, "")
            elif config.startswith("secret:"):
                encrypted = config[7:]
                return self.secret_manager.decrypt_secret(encrypted)
            elif config.startswith("vault:"):
                # 从 Vault 读取
                vault_path = config[6:]
                return f"<vault:{vault_path}>"  # 实际从 Vault 客户端读取
            return config
        elif isinstance(config, list):
            return [await self._resolve_secrets(item) for item in config]
        return config
```

### 7.2 配置 Schema 验证

```python
# config_validator.py
"""
配置 Schema 验证器
确保错误的配置不会导致 Agent 行为异常
"""

class ConfigValidator:
    """
    多层配置验证: 语法 → Schema → 业务规则 → 灰度验证
    """

    def __init__(self, schema: Dict):
        self.schema = schema

    def validate_syntax(self, config: Dict) -> List[str]:
        """语法层校验: YAML/JSON 格式正确性"""
        errors = []
        # 由 YAML/JSON 解析器负责, 这里只做结构化检查
        if not isinstance(config, dict):
            errors.append("配置必须是字典类型")
        return errors

    def validate_schema(self, config: Dict) -> List[str]:
        """Schema 层校验: 类型/范围/必填"""
        errors = []
        try:
            AgentConfig(**config)
        except Exception as e:
            errors.append(str(e))
        return errors

    def validate_business_rules(self, config: Dict) -> List[str]:
        """业务规则校验: Agent 特有的约束"""
        errors = []

        llm_configs = config.get("llm", {})
        if "primary" not in llm_configs:
            errors.append("必须配置 primary LLM 模型")

        # 检查 fallback 链中不能有重复模型
        tools = config.get("tools", [])
        tool_names = [t.get("name") for t in tools if isinstance(t, dict)]
        if len(tool_names) != len(set(tool_names)):
            errors.append("工具名称不能重复")

        # 检查记忆配置的一致性
        mem_config = config.get("memory", {})
        if mem_config.get("type") == "vector_store":
            if "provider" not in mem_config:
                errors.append("向量存储需要指定 provider")

        # 检查速率限制
        rate_limits = config.get("rate_limits", {})
        rpm = rate_limits.get("requests_per_minute", 0)
        if rpm < 1:
            errors.append("请求速率限制必须 >= 1")

        return errors

    def validate_all(self, config: Dict) -> Dict:
        """执行全部验证"""
        return {
            "syntax_errors": self.validate_syntax(config),
            "schema_errors": self.validate_schema(config),
            "business_errors": self.validate_business_rules(config),
            "is_valid": all([
                not self.validate_syntax(config),
                not self.validate_schema(config),
                not self.validate_business_rules(config),
            ])
        }
```

### 7.3 配置变更的灰度发布

```python
class ConfigCanaryRelease:
    """
    配置的灰度发布 — 先在小范围验证, 再全量推送
    """

    def __init__(self, config_mgr: AgentConfigManager):
        self.config_mgr = config_mgr
        self.canary_groups = {
            "internal": 0.05,     # 5% 内部员工
            "beta": 0.20,        # 20% Beta 用户
            "stable": 1.0,       # 100% 稳定
        }

    async def publish_with_canary(
        self,
        changes: Dict[str, str],
        target_group: str = "beta",
        observation_period: int = 300,  # 观察 5 分钟
    ) -> bool:
        """
        灰度发布配置变更
        """
        ratio = self.canary_groups.get(target_group, 0.05)
        logger.info(
            f"[Config] 灰度发布开始: group={target_group}, ratio={ratio}"
        )

        # 1. 推送给灰度实例
        await self._apply_to_instances(changes, ratio)

        # 2. 观察期: 检查错误率和响应
        await asyncio.sleep(observation_period)

        # 3. 检查效果
        metrics = await self._collect_canary_metrics()
        if metrics.get("error_rate", 0) > 0.01:  # 错误率 > 1%
            logger.error("[Config] 灰度发布发现错误, 自动回滚")
            await self.rollback(changes)
            return False

        # 4. 全量推送
        for key, value in changes.items():
            await self.config_mgr.update_config(key, value, version=999)
        logger.info("[Config] 灰度验证通过, 全量推送完成")
        return True

    async def _apply_to_instances(self, changes: Dict, ratio: float):
        """按比例推送给部分实例"""
        # 简化实现: 利用配置中心的灰度能力
        # Apollo 支持按 IP / 标签灰度
        # Nacos 支持按集群/分组灰度
        pass

    async def _collect_canary_metrics(self) -> Dict:
        """收集灰度实例的运行指标"""
        # 检查: LLM 调用成功率、延迟变化、错误类型分布
        return {"error_rate": 0.0}

    async def rollback(self, changes: Dict):
        """回滚灰度配置"""
        logger.warning("[Config] 回滚灰度配置")
        # 恢复旧值
        pass
```

---

## 8. 最大挑战

### 8.1 配置一致性 (Config Consistency)

在分布式 Agent 系统中, 所有实例看到相同的配置是正确行为的前提。

```
配置不一致导致的问题:

  场景: 更新工具白名单 (禁用了 "web_search")

  时间 →
  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
  │ 实例 A  │  │ 实例 B  │  │ 实例 C  │  │ 实例 D  │
  │(已更新)  │  │(已更新)  │  │(未更新)  │  │(未更新)  │
  │         │  │         │  │         │  │         │
  │ web_    │  │ web_    │  │ web_    │  │ web_    │
  │ search  │  │ search  │  │ search  │  │ search  │
  │ ❌ 禁用  │  │ ❌ 禁用  │  │ ✅ 可用  │  │ ✅ 可用  │
  └────────┘  └────────┘  └────────┘  └────────┘

  问题:
  • 部分 Agent 实例仍然使用 web_search
  • 用户可以"挑选"连接未更新的实例来规避限制
  • 运维团队不知道哪个实例是"正确的"
  • 调试时困惑: 相同输入, 不同输出

  根源:
  1. 配置推送不是原子操作 (先推送 A, 再推 B...)
  2. 实例可能离线 (更新期间宕机)
  3. 网络分区导致部分实例无法接收更新
  4. 不同实例的配置版本散落在不同版本

  解决方案:
  • 配置版本号: 每次变更附带全局版本号
  • 版本一致性检查: 实例定期上报当前版本
  • 配置仲裁: 如果版本差异过大, 强制同步
  • 语义锁定: Agent 在执行关键推理前检查配置版本
```

**配置一致性检查器**：

```python
class ConfigConsistencyChecker:
    """配置一致性检查与修复"""

    def __init__(self, config_mgr: AgentConfigManager, registry):
        self.config_mgr = config_mgr
        self.registry = registry    # 实例注册中心

    async def check_consistency(self) -> Dict:
        """检查集群中所有实例的配置版本一致性"""
        instances = await self.registry.get_all_instances()
        versions = {}

        for inst in instances:
            inst_version = await self._get_instance_version(inst)
            versions[inst["id"]] = inst_version

        # 分析版本分布
        version_counts = {}
        for v in versions.values():
            version_counts[v] = version_counts.get(v, 0) + 1

        most_common = max(version_counts, key=version_counts.get)
        outliers = [k for k, v in versions.items() if v != most_common]

        return {
            "total_instances": len(instances),
            "version_distribution": version_counts,
            "majority_version": most_common,
            "outdated_instances": outliers,
            "consistent": len(outliers) == 0,
        }

    async def _get_instance_version(self, instance: Dict) -> int:
        """查询实例的配置版本"""
        # 从实例的健康检查端点获取
        # GET /config/version → {"version": 12}
        return instance.get("config_version", 0)

    async def enforce_consistency(self):
        """强制所有实例使用最新配置"""
        result = await self.check_consistency()
        if not result["consistent"]:
            for inst_id in result["outdated_instances"]:
                logger.warning(f"[Config] 实例 {inst_id} 配置落后, 触发同步")
                # 通知实例强制重新加载
                await self._trigger_reload(inst_id)
```

### 8.2 部分更新 (Partial Updates)

在 Agent 系统中, 配置项之间往往存在依赖关系, 部分更新可能导致不一致状态。

```
部分更新的陷阱:

  正常配置 (v5):
  ┌────────────────────────────────┐
  │ llm.model = "claude-3-opus"    │
  │ llm.fallback = "claude-sonnet" │
  │ prompt.system = "v3.0"         │
  │ tools.enabled = ["search","code"] │
  └────────────────────────────────┘
            所有配置一致

  错误的部分更新 (只更新了模型, 没有更新 fallback):
  ┌────────────────────────────────┐
  │ llm.model = "gpt-4o"           │  ← 更新了
  │ llm.fallback = "claude-sonnet" │  ← 没更新! 跨 Provider 不一致
  │ prompt.system = "v3.0"         │
  │ tools.enabled = ["search","code"] │
  └────────────────────────────────┘
            Fallback 模型可能是被 OpenAI 禁用的

  更隐蔽的问题 (多配置关联):
  修改 tools.enabled 移除 "code_interpreter" →
    但 memory.config 仍然记录着 code_interpreter 的执行结果格式 →
    Agent 尝试解析不再支持的执行结果 → 解析失败

  解决方案:
  1. 事务性更新: 一组相关变更要么全部成功, 要么全部回滚
  2. 配置依赖图: 声明配置项之间的依赖关系
  3. 预检查: 在应用前验证配置组的完整性
  4. 版本化配置组: 将相关配置捆绑为一个版本单元
```

**配置依赖与事务更新**：

```python
class ConfigDependencyManager:
    """
    配置依赖管理与事务性更新
    确保相关配置项之间的依赖关系不会断裂
    """

    def __init__(self):
        # 配置依赖图: 父配置 → [子配置列表]
        self.dependency_graph = {
            "llm.model": ["llm.fallback", "llm.max_tokens", "llm.temperature"],
            "llm.fallback": [],  # fallback 不依赖其他配置
            "tools.enabled": ["tools.{name}.enabled"],
            "memory.type": ["memory.connection", "memory.indexing"],
            "system.prompt_version": ["prompts.{version}"],
        }

        # 互斥配置组: 同一组内的配置不能指向不同 Provider
        self.mutex_groups = [
            {"keys": ["llm.model", "llm.fallback"], "rule": "same_provider"},
        ]

    async def transactional_update(
        self,
        changes: Dict[str, str],
        config_mgr: AgentConfigManager,
    ) -> bool:
        """
        事务性配置更新:
        1. 验证变更集的完整性
        2. 检查受影响的相关配置
        3. 原子化提交
        """
        # Step 1: 扩展变更集 (添加依赖项)
        expanded = await self._expand_dependencies(changes)
        logger.info(f"[Config] 扩展后变更集: {expanded}")

        # Step 2: 验证完整性
        validation = await self._validate_change_set(expanded)
        if not validation["valid"]:
            logger.error(f"[Config] 事务验证失败: {validation['errors']}")
            return False

        # Step 3: 原子应用
        try:
            for key, value in expanded.items():
                await config_mgr.update_config(key, value, version=100)
            logger.info(f"[Config] 事务完成: {len(expanded)} 项")
            return True
        except Exception as e:
            logger.error(f"[Config] 事务失败, 回滚: {e}")
            # 回滚已应用的变更
            return False

    async def _expand_dependencies(self, changes: Dict) -> Dict:
        """扩展变更集: 如果更新了父配置, 确保子配置也更新"""
        expanded = dict(changes)

        for key, value in changes.items():
            deps = self.dependency_graph.get(key, [])
            for dep_template in deps:
                dep_key = dep_template.replace("{name}", key.split(".")[-1])
                if dep_key not in changes:
                    logger.warning(
                        f"[Config] 依赖配置 {dep_key} 未被包含在变更中"
                    )
        return expanded

    async def _validate_change_set(self, changes: Dict) -> Dict:
        """验证变更集的约束"""
        errors = []

        # 检查互斥组
        for group in self.mutex_groups:
            keys_in_change = [k for k in group["keys"] if k in changes]
            if len(keys_in_change) > 1:
                # 检查是否为同一 Provider
                providers = set()
                for key in keys_in_change:
                    value = changes[key]
                    if "gpt" in value:
                        providers.add("openai")
                    elif "claude" in value:
                        providers.add("anthropic")
                if len(providers) > 1:
                    errors.append(
                        f"互斥配置冲突: {keys_in_change} 不能跨 Provider"
                    )

        return {"valid": len(errors) == 0, "errors": errors}
```

### 8.3 配置漂移 (Config Drift)

随着时间的推移, 实际运行的配置与期望配置之间逐渐产生差异。

```
配置漂移的成因:

  1. 手动临时修改:
     运维直接在服务器上改了配置 → 没有走配置中心

  2. 滚动更新时序:
     新版本 v2 部署时, old 实例用 v1 配置, new 实例用 v2 配置

  3. 灰度残留:
     灰度发布后, 部分实例没有更新到全量配置

  4. 配置回滚遗漏:
     回滚了配置中心, 但部分实例的本地缓存没有清除

  5. 环境泄漏:
     测试环境的配置意外推到了生产实例

  检测与修复:

  期望配置 (Git)         实际配置 (运行时)
  ┌──────────────┐       ┌──────────────┐
  │ model: gpt-4 │       │ model: gpt-4 │  ✅ 一致
  │ tools: [...] │       │ tools: [...] │  ✅
  │ version: 12  │       │ version: 12  │  ✅
  └──────────────┘       └──────────────┘

  ┌──────────────┐       ┌──────────────┐
  │ model: gpt-4 │       │ model: gpt-4o│  ❌ 漂移!
  │ version: 12  │       │ version: 11  │  ❌ 版本落后
  └──────────────┘       └──────────────┘

  修复: 自动同步 / 告警人工介入
```

```python
class ConfigDriftDetector:
    """配置漂移检测器"""

    def __init__(self, config_mgr: AgentConfigManager, git_repo: str = ""):
        self.config_mgr = config_mgr
        self.git_repo = git_repo

    async def detect_drift(self) -> List[Dict]:
        """检测所有 Agent 实例的配置漂移"""
        drifts = []

        # 1. 获取期望配置 (来自 Git)
        desired = await self._get_desired_config()

        # 2. 获取所有实例的实际配置
        instances = await self._get_all_instance_configs()

        # 3. 逐一对比
        for inst in instances:
            diff = self._diff_config(desired, inst["config"])
            if diff:
                drifts.append({
                    "instance_id": inst["id"],
                    "drifted_keys": diff,
                    "severity": self._assess_severity(diff),
                })

        return drifts

    async def auto_remediate(self, drift: Dict) -> bool:
        """自动修复配置漂移"""
        if drift["severity"] == "critical":
            logger.warning(
                f"[Config] 严重漂移 {drift['instance_id']}, "
                f"强制同步配置"
            )
            # 强制实例重新加载配置
            await self._force_reload(drift["instance_id"])
            return True
        elif drift["severity"] == "warning":
            logger.info(
                f"[Config] 轻度漂移 {drift['instance_id']}, "
                f"发送告警"
            )
            # 发送告警, 等待人工处理
            return False
        return False  # info 级别, 仅记录

    def _diff_config(self, desired: Dict, actual: Dict) -> Dict:
        """对比期望配置和实际配置的差异"""
        diff = {}
        for key, expected_value in desired.items():
            actual_value = actual.get(key)
            if actual_value != expected_value:
                diff[key] = {
                    "expected": expected_value,
                    "actual": actual_value,
                }
        return diff

    def _assess_severity(self, diff: Dict) -> str:
        """评估漂移的严重程度"""
        critical_keys = ["llm.model", "tools.enabled", "security"]
        for key in diff:
            if any(c in key for c in critical_keys):
                return "critical"
        return "warning"

    async def _get_desired_config(self) -> Dict:
        """从 Git 获取期望配置"""
        # 实现: 读取 Git 仓库中 config/ 目录下的期望配置
        return {}

    async def _get_all_instance_configs(self) -> List[Dict]:
        """获取所有实例的当前配置"""
        # 实现: 通过 Agent 的 /config 端点获取
        return []
```

---

## 9. 总结与最佳实践

### 最佳实践清单

1. **配置即代码**: 所有配置变更通过 Git PR 审查, 保留完整历史
2. **环境隔离**: 不同环境 (dev/staging/prod) 使用独立的配置命名空间
3. **最小权限配置**: 每个 Agent 服务只访问其必需的配置项
4. **配置版本管理**: 每个配置变更附带版本号, 支持一键回滚
5. **Schema 校验**: 所有配置在加载和变更时进行 Schema 校验, 拒绝无效配置
6. **敏感信息加密**: API Key / Secret 使用 Vault 或加密存储, 绝不明文
7. **灰度发布**: 重要配置变更先推送给 5-20% 实例验证
8. **配置一致性监控**: 持续检查所有实例的配置版本是否一致
9. **优雅变更**: 根据配置类型选择不同的热更新策略 (立即/下个会话/重启)
10. **配置依赖管理**: 避免部分更新导致配置组内部不一致

### 参考与延伸

- **Apollo 配置中心**: 携程开源的分布式配置中心, 功能丰富
- **Nacos**: 阿里开源的动态服务发现、配置管理平台
- **HashiCorp Consul KV + Vault**: 轻量级配置 + 密钥管理
- **Kubernetes ConfigMap & Secret**: K8s 原生配置管理
- **AWS AppConfig**: AWS 应用配置管理服务
- **etcd**: 分布式键值存储, 常用于配置管理后端
- **ArgoCD**: 声明式 GitOps 部署工具
- **SOPS**: 加密 YAML/JSON 文件的 CLI 工具, 适合 GitOps 密钥管理
