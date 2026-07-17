# 13.1.5 env-management — 环境管理

**多环境管理是 Agent 从开发到生产的安全通道。** 没有环境隔离，开发者可能在开发环境中意外消耗大量生产 LLM API 配额，或者在调试时向真实用户发送测试消息。Agent 的"环境"比传统应用多了三层复杂度：LLM API 密钥和模型版本、Prompt 版本、记忆/知识库的独立实例。一个成熟的环境管理方案需要同时覆盖代码、配置、数据、模型和 Prompt 五个维度。

## 背景与问题

### 传统应用的环境管理

传统 Web 服务通常有清晰的环境分层：开发者在本地开发 -> 推送到测试环境 -> 预发布验证 -> 生产发布。环境的区别主要是：数据库连接、API 密钥、配置参数。

```
开发环境          测试环境           预发布             生产环境
─ ─ ─ ─ ─      ─ ─ ─ ─ ─        ─ ─ ─ ─ ─        ─ ─ ─ ─ ─
本地 DB         测试 DB           类生产 DB          生产 DB
.dev 密钥        .test 密钥        .staging 密钥      .prod 密钥
调试模式开启      模拟外部服务        接近真实            完全真实
```

### Agent 带来的新维度

Agent 的环境管理比传统应用多了三个独特的维度：

```
传统应用的环境维度:           Agent 新增的环境维度:
代码版本                        🔸 LLM 模型版本 + 参数配置
数据库连接                      🔸 Prompt 版本 (System Prompt + 工具定义)
API 密钥                        🔸 知识库 / 向量数据库实例
第三方服务凭证                    🔸 记忆存储实例
基础设施配置                    🔸 工具执行环境
                               🔸 评估数据集 + 黄金测试集
```

这意味着 Agent 的环境不能简单地复制传统方案——**每个环境不仅是"配置不同"，而是"认知能力不同"**。

### 之前是怎么做的？

1. **单一环境开发**：所有人在同一个环境开发调试，共享一个 LLM API Key
2. **环境变量驱动**：用 `.env` 文件 + 环境变量切换 API Key 和模型
3. **分支对应环境**：`git branch` 对应环境（dev/staging/prod），手动切换

### 这样做的问题

```
问题                         后果
─────────────────            ────────────────
所有环境共用一个 LLM API Key   开发环境误操作消耗生产配额，成本失控
开发环境也用 GPT-4 生产模型     每次调试消耗 $0.10+，日积月累惊人
Prompt 无版本管理              预发布验证通过的 Prompt 上线后不一致
知识库不隔离                   开发环境写入测试数据污染生产检索
记忆存储共享                   开发调试制造的记忆影响真实用户体验
环境切换不自动化               人为配置错误导致事故频发
```

## 核心原则

### 环境分层的核心模型

Agent 系统推荐四层环境模型，每一层有不同的 LLM 策略、数据隔离度和成本预算：

```
                 开发 (Dev)         测试 (Test)        预发布 (Staging)     生产 (Prod)
                ──────────        ────────────       ───────────────      ─────────────
LLM 模型         小模型优先          混合                生产模型 (70B+)     生产模型
                 (GPT-4o-mini,      (大部分小模型，      (GPT-4, Claude)    (最新稳定版)
                 Claude Haiku)      关键测试用生产模型)

Prompt 版本     最新开发版          PR 对应版本         即将发布的版本        稳定版本
                 (可随时修改)        (冻结+评审)          (完全冻结)

知识库           裁剪版/模拟         测试数据集            生产数据子集         完整生产数据
                 (100 条文档)        (1K 条)             (10K 条)

记忆             临时/可清理          隔离实例              隔离实例             生产实例
                  (每次部署重置)       (保留测试用)

API Key         沙箱 Key/有限预算   隔离 Key+配额         隔离 Key+配额       生产 Key+预算告警

工具             模拟 (Mock)         部分真实              完全真实             完全真实
                  + 沙箱环境          + 隔离

成本限制          $50/月              $200/月              $500/月             按预算
```

### Agent 特有的环境隔离策略

理解 Agent 环境管理的核心在选择合适的 LLM 模型和 Prompt 版本策略：

```
开发环境                       测试环境
── ─ ─ ─ ─                   ── ─ ─ ─ ─
不要求结果质量                要求结果质量达标
F1 < 0.7 可接受               F1 ≥ 0.85 才通过
可使用快速迭代模型             需使用最终模型或接近
Prompt 可大幅修改              Prompt 已冻结
可使用 mock 工具              tool 调用需完整验证

预发布环境                     生产环境
── ─ ─ ─ ─                   ── ─ ─ ─ ─
与生产完全一致(A/B)           稳定版本运行
唯一区别是流量                 自动回滚/金丝雀
全量评估跑一遍                 监控+告警
验证通过才切换流量             人工审批上线
```

## Agent 环境配置管理架构

### 分层配置模型

```
┌──────────────────────────────────────────────┐
│  第1层: 环境类型 (env_type)                    │
│  dev / test / staging / prod                  │
│  通过环境变量 AGENT_ENV 注入                    │
├──────────────────────────────────────────────┤
│  第2层: LLM 配置                              │
│  model_name, api_key, max_tokens, temperature │
│  ─ 不同环境用不同模型和预算                      │
├──────────────────────────────────────────────┤
│  第3层: 服务连接                               │
│  vector_store_url, memory_store_url, db_url   │
│  ─ 严格隔离的存储实例                            │
├──────────────────────────────────────────────┤
│  第4层: Prompt 配置                            │
│  system_prompt_version, tool_def_version      │
│  ─ 版本化、环境相关                             │
├──────────────────────────────────────────────┤
│  第5层: Agent 行为配置                         │
│  max_steps, enable_reflection, allow_destructive│
│  ─ 开发环境允许破坏性操作，生产环境需要审批         │
└──────────────────────────────────────────────┘
```

### 配置加载实现

```python
# agent_config.py — 分层配置管理
import os
from functools import lru_cache
from typing import Optional
from pydantic import BaseModel, Field, SecretStr

class LLMConfig(BaseModel):
    model: str
    api_key: SecretStr
    max_tokens: int = 4096
    temperature: float = 0.7
    cost_limit_per_request: float = 0.05  # 单次请求成本上限

class MemoryConfig(BaseModel):
    vector_store_url: str
    kv_store_url: str
    namespace: str  # 环境命名空间隔离

class AgentBehavior(BaseModel):
    max_steps: int = 20
    enable_reflection: bool = True
    allow_destructive_tools: bool = False  # 生产环境默认禁止删除操作
    require_human_approval: list[str] = ["delete", "write_file", "execute_code"]

class EnvironmentConfig(BaseModel):
    env_type: str  # dev/test/staging/prod
    llm: LLMConfig
    memory: MemoryConfig
    behavior: AgentBehavior
    prompt_version: str
    log_level: str = "INFO"

class ConfigManager:
    """分层配置管理器，根据环境类型加载不同配置"""
    
    ENV_SPECS = {
        "dev": {
            "llm": {"model": "gpt-4o-mini", "max_tokens": 2048, "cost_limit": 0.01},
            "behavior": {"allow_destructive_tools": True, "max_steps": 50},
            "log_level": "DEBUG",
        },
        "test": {
            "llm": {"model": "gpt-4o-mini", "max_tokens": 4096, "cost_limit": 0.02},
            "behavior": {"allow_destructive_tools": False},
            "log_level": "INFO",
        },
        "staging": {
            "llm": {"model": "gpt-4o", "max_tokens": 8192, "cost_limit": 0.05},
            "behavior": {"allow_destructive_tools": False},
            "log_level": "INFO",
        },
        "prod": {
            "llm": {"model": "gpt-4o", "max_tokens": 16384, "cost_limit": 0.10},
            "behavior": {"allow_destructive_tools": False},
            "log_level": "WARNING",
        },
    }
    
    def __init__(self, env_type: Optional[str] = None):
        self.env_type = env_type or os.getenv("AGENT_ENV", "dev")
        if self.env_type not in self.ENV_SPECS:
            raise ValueError(f"Unknown environment: {self.env_type}")
    
    @lru_cache()
    def load(self) -> EnvironmentConfig:
        spec = self.ENV_SPECS[self.env_type]
        return EnvironmentConfig(
            env_type=self.env_type,
            llm=LLMConfig(
                model=spec["llm"]["model"],
                api_key=SecretStr(os.getenv(f"LLM_API_KEY_{self.env_type.upper()}")),
                max_tokens=spec["llm"]["max_tokens"],
                cost_limit_per_request=spec["llm"]["cost_limit"],
            ),
            memory=MemoryConfig(
                vector_store_url=os.getenv(f"VECTOR_STORE_URL_{self.env_type.upper()}"),
                kv_store_url=os.getenv(f"KV_STORE_URL_{self.env_type.upper()}"),
                namespace=f"agent_{self.env_type}",
            ),
            behavior=AgentBehavior(**spec["behavior"]),
            prompt_version=os.getenv(f"PROMPT_VERSION_{self.env_type.upper()}", "latest"),
            log_level=spec["log_level"],
        )

# 使用示例
config = ConfigManager(env_type="prod").load()
print(f"Using model: {config.llm.model}")  # gpt-4o
print(f"Destructive tools: {config.behavior.allow_destructive_tools}")  # False
```

## Prompt 版本管理

Agent 的核心能力由 Prompt 定义，Prompt 版本管理是 Agent 环境管理中最独特的挑战。

### Prompt 版本与代码版本的对应

```
PR #123: 新增代码审查功能
├── src/agent/          ← 代码变更
├── prompts/            ← Prompt 变更
│   ├── system/v2.1.md    系统 Prompt 更新
│   ├── tools/code-review.md  新增工具 Prompt
│   └── examples/      新增示例
└── tests/prompts/      ← Prompt 测试
    └── test_code_review.yaml
```

### Prompt 环境晋升流程

```
开发者编写/修改 Prompt
        │
        ▼
   Dev 环境验证
   ├── 用 dev 模型测试新 Prompt
   ├── 开发者自测通过
   └── Prompt 写入 Git (feature branch)
        │
        ▼
   PR 提交 → 自动测试
   ├── 用 staging 模型跑回归
   ├── 与基线版本对比质量
   └── 自动评估通过后合并
        │
        ▼
   Staging 预发布验证
   ├── 用生产模型跑完整评估套件
   ├── A/B 对比 (v2.0 vs v2.1)
   └── 人工评审通过
        │
        ▼
   Prod 金丝雀发布
   ├── 5% 流量先用新 Prompt
   ├── 监控质量指标
   └── 逐步扩到 100%
```

```python
# prompt_version_manager.py
class PromptVersionManager:
    """Prompt 版本管理与环境绑定"""
    
    def __init__(self, storage_backend: str = "git"):
        self.versions = {}  # version_id -> PromptBundle
        self.env_binding = {}  # env -> version_id
        
    def promote(self, version_id: str, from_env: str, to_env: str):
        """将 Prompt 版本从源环境晋升到目标环境"""
        # 验证
        if from_env == "dev" and to_env == "staging":
            assert self._regression_test_passed(version_id), "Regression test required"
        if from_env == "staging" and to_env == "prod":
            assert self._eval_suite_passed(version_id), "Full eval suite required"
            assert self._human_reviewed(version_id), "Human review required"
        
        self.env_binding[to_env] = version_id
        self._log_promotion(version_id, from_env, to_env)
```

## 记忆与知识库的环境隔离

Agent 的生产化中最容易被忽视的是：**记忆和知识库需要在环境间严格隔离**。

```yaml
# environment-isolation.yaml
environments:
  dev:
    vector_store:
      url: ${DEV_VECTOR_DB_URL}
      collection_prefix: "agent_dev_"     # 前缀隔离
      refresh_on_deploy: true             # 每次部署重建
    kv_store:
      url: ${DEV_KV_URL}
      namespace: "agent-dev"
  
  test:
    vector_store:
      url: ${TEST_VECTOR_DB_URL}
      collection_prefix: "agent_test_"
      snapshot_from: "prod"               # 从生产快照创建
    kv_store:
      url: ${TEST_KV_URL}
      namespace: "agent-test"
  
  staging:
    vector_store:
      url: ${STAGING_VECTOR_DB_URL}
      collection_prefix: "agent_staging_"
      sync_from_prod: true                # 定期同步生产数据
    kv_store:
      url: ${STAGING_KV_URL}
      namespace: "agent-staging"
  
  prod:
    vector_store:
      url: ${PROD_VECTOR_DB_URL}
      collection_prefix: "agent_prod_"
      backup_schedule: "0 */6 * * *"      # 每 6 小时备份
    kv_store:
      url: ${PROD_KV_URL}
      namespace: "agent-prod"
```

### 隔离策略选项

| 策略 | 实现方式 | 优点 | 缺点 | 推荐场景 |
|------|---------|------|------|---------|
| Prefix 隔离 | 同一 DB + 不同前缀 | 管理简单 | 共享资源，性能可能互相影响 | Dev/Test |
| 独立实例 | 完全独立的 DB 实例 | 完全隔离 | 成本高 | Staging/Prod |
| 快照恢复 | 从生产定期快照恢复 | 数据真实 | 快照延迟 | Test |
| 代理缓存 | 生产数据的子集 | 快速启动 | 数据不完整 | Dev |

## 关键挑战与应对

### 挑战 1：LLM API Key 泄露

开发环境中 API Key 泄露的风险最高（代码提交到 Git、日志输出、错误消息暴露）。

```python
# 安全配置加载
import os
from pathlib import Path

def load_api_key(env: str) -> str:
    """分层密钥加载，避免密钥泄露"""
    # 1. 优先环境变量 (CI/CD 注入)
    key = os.getenv(f"LLM_API_KEY_{env.upper()}")
    if key:
        return key
    
    # 2. 本地密钥管理服务
    key = _from_secret_manager(env)
    if key:
        return key
    
    # 3. 本地 .env 文件 (仅开发环境)
    if env == "dev":
        env_file = Path(".env.local")
        if env_file.exists():
            return _parse_env_file(env_file)[f"LLM_API_KEY_DEV"]
    
    raise ValueError(f"No API key found for environment: {env}")
```

### 挑战 2：环境间配置漂移

随着时间推移，不同环境的配置会逐渐不一致，导致"在 staging 能运行，上生产就出问题"。

**对策**：
1. **基础设施即代码**：所有环境用 Terraform/Pulumi 声明式定义
2. **配置 lint 检查**：CI 中对比 staging 和 prod 的配置差异
3. **金丝雀的配置一致性检查**：发布前自动验证
4. **定期环境同步**：每月对齐所有环境的非敏感配置

### 挑战 3：Agent 行为的环境特异

不同环境下 Agent 的行为应该有显著差异：

| 行为 | Dev | Test | Staging | Prod |
|------|-----|------|---------|------|
| 最大推理步数 | 50 (允许探索) | 20 | 20 | 15 |
| 自我反思 | ✅ | ✅ | ✅ | ⚠ 仅对低置信度 |
| 工具 Mock | ✅ | ⚠ 部分 | ❌ 真实 | ❌ 真实 |
| 破坏性操作 | ✅ 允许 | ❌ 禁止 | ❌ 禁止 | ❌ 禁止+审批 |
| 日志级别 | DEBUG | INFO | INFO | WARNING |
| Token 限制 | 100K/日 | 500K/日 | 2M/日 | 按 SLA |
| 人工审批 | ❌ 不需要 | ❌ | ⚠ 高风险 | ✅ 全部 |

## 工程优化方向

1. **环境配置模板 + 差异覆盖**：基础配置通用，每个环境只写差异部分
2. **自动环境一致性检查**：CI 中检查 staging 和 prod 配置的差异告警
3. **环境启动脚本自动化**：`make setup-dev` 一键搭建完整开发环境
4. **开发者门户 (Backstage)**：统一管理环境配置、API Key、版本信息
5. **环境使用量监控**：跟踪每个环境的 Token 消耗、API 调用量、成本
6. **临时环境 (Ephemeral Environment)**：每个 PR 自动创建临时环境，合并后自动销毁

## 能力边界与适用场景

### 适合多层环境管理的团队

- 团队规模 **3 人以上** 共同开发 Agent
- Agent 需要 **对外提供 SLA** 的服务
- 涉及 **用户数据** 或 **金融/医疗等合规场景**
- 需要 **Prompt 变更可追溯、可回滚**

### 不适合过度环境管理的场景

- 个人项目 / 原型验证：Dev + Prod 两层就够了
- 非关键 Agent（内部工具、一次性任务）：Dev + Prod 两层
- 完全离线 Agent：一层环境 + 版本控制即可

### 最小化环境模型

对于小型项目，两层环境已经足够：

```
开发环境                         生产环境
── ─ ─ ─ ─                     ── ─ ─ ─ ─
小模型 (Haiku / mini)           标准模型 (Sonnet / GPT-4o)
.mock 环境变量                   真实服务
.可频繁重置                       持久化数据
.无监控告警                       完整监控
.COST_LIMIT=低                    COST_LIMIT=按预算
```

层数不是越多越好——每增加一层，维护成本翻倍。关键不是"有多少环境"，而是"环境之间有没有清晰的隔离和晋升标准"。
