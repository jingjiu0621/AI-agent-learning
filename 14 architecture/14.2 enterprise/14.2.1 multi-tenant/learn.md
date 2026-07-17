# 14.2.1 多租户隔离 — Multi-Tenant Isolation

> **核心思想**：多租户隔离是指在同一个 Agent 平台上，不同租户（客户/组织/部门）的数据、配置、权限和资源在逻辑或物理上完全隔离，确保租户之间互不可见、互不影响。这不仅是数据库加一个 `tenant_id` 字段的问题，而是需要在**身份认证、数据存储、模型调用、工具注册、缓存、日志、监控**全链路强化租户边界。

---

## 1. 基本原理

### 1.1 什么是多租户 Agent 平台

多租户 Agent 平台允许**一个部署实例同时服务多个独立客户**。每个租户看到的是自己的 Agent 实例，但实际上共享底层基础设施。

```
┌─────────────────────────────────────────────────────────────┐
│                   多租户 Agent 平台                           │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ 租户 A    │  │ 租户 B    │  │ 租户 C    │  │ 租户 D    │   │
│  │ 金融行业   │  │ 医疗行业   │  │ 零售行业   │  │ 教育行业   │   │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘   │
│        │              │              │              │        │
│        └──────────────┼──────────────┼──────────────┘        │
│                       │              │                       │
│              ┌────────▼──────────────▼────────┐              │
│              │       隔离层 (Isolation Layer)    │              │
│              │  Tenant Context → 路由 → 过滤    │              │
│              └─────────────────────────────────┘              │
│                       │              │                       │
│        ┌──────────────┼──────────────┼──────────────┐        │
│        ▼              ▼              ▼              ▼        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ LLM 推理  │  │ 向量存储  │  │ 工具执行  │  │ 知识库    │   │
│  │ (共享池)  │  │ (隔离)    │  │ (隔离)    │  │ (隔离)    │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 背景与演进

**之前怎么做**：
- **SaaS 1.0（无隔离）**：所有用户共享同一个 Prompt、同一套工具、同一个知识库
- **SaaS 2.0（数据库级隔离）**：在数据表加 tenant_id 字段，代码中通过 WHERE tenant_id = ? 过滤
- **SaaS 3.0（应用级隔离）**：独立的应用实例（Docker），共享数据库但 Schema 隔离
- **SaaS 4.0（全链路隔离）**：身份、数据、模型配额、工具权限、配置、缓存、日志全维度隔离

**产生的问题**：
- 只加 tenant_id 的隔离在 Agent 场景下完全不够——Agent 的自主决策可能导致跨租户访问
- LLM 在没有明确约束时可能"借用"其他租户的数据
- 缓存命中错误租户的数据
- 日志中泄漏跨租户信息

**核心矛盾**：SaaS 的经济性（共享基础设施降低成本）与安全性（完全隔离防止数据泄露）之间的根本冲突。

### 1.3 七维隔离模型

多租户隔离需要覆盖 7 个维度：

```
维度 1: 身份 (Identity)
  └── 租户 ID 从认证 Token 解析，贯穿请求全链路
  
维度 2: 数据 (Data)  
  ├── 结构化: 数据库表中 tenant_id 分区
  ├── 向量: Metadata 过滤 / 独立 Collection / 独立实例
  └── 文件: 租户级存储桶 / 前缀隔离

维度 3: 模型配额 (Model Quota)
  ├── 租户级 Token 预算 (每日/每小时)
  ├── 租户级 Rate Limit (RPM/TPM)
  └── 模型访问白名单

维度 4: 工具权限 (Tool Permission)
  ├── 租户可用的工具列表
  ├── 工具参数级权限控制
  └── 工具调用频次限制

维度 5: 配置 (Configuration)
  ├── System Prompt / 角色设定
  ├── 知识库 / RAG 数据源
  └── 安全策略 / 输出护栏

维度 6: 缓存 (Cache)
  ├── Cache Key 必须包含 tenant_id
  └── 共享缓存仅限公共数据

维度 7: 日志 (Logging)
  ├── 日志按租户可检索
  ├── 敏感字段自动脱敏
  └── 运维人员受权限控制
```

---

## 2. 隔离策略深度分析

### 2.1 隔离模式对比

| 隔离模式 | 实现方式 | 安全等级 | 成本 | 适用场景 |
|----------|----------|----------|------|----------|
| 软隔离 (Soft) | 字段过滤 + 代码约束 | ★★★☆☆ | 低 | 低安全要求、内部多部门 |
| 硬隔离 (Hard) | 独立 Collection/Schema | ★★★★☆ | 中 | 中等安全要求、SaaS |
| 物理隔离 (Physical) | 独立实例/集群 | ★★★★★ | 高 | 高安全要求、金融/医疗 |
| 混合隔离 (Hybrid) | 小租户软隔离+大租户物理 | ★★★★☆ | 中 | 多层级客户 |

#### 软隔离（推荐起步）

```
优点: 基础设施共享，运维成本低，快速添加租户
缺点: 隔离强度依赖代码质量，过滤遗漏可能泄露数据
风险: 必须有"二次校验"机制兜底

适用: 早期 SaaS、内部工具、低风险场景
```

#### 硬隔离（推荐生产）

```
优点: 数据库层面天然隔离，过滤遗漏不会泄露
缺点: 管理多个 Collection/Schema，备份恢复复杂
风险: 需要统一管理多个索引，跨租户查询困难

适用: 生产 SaaS、中等风险场景
```

#### 物理隔离（推荐合规）

```
优点: 最强的隔离保证，独立扩缩容，独立版本
缺点: 资源利用率低，运维复杂度 x10
风险: 更新需要逐个实例，增加人力成本

适用: 金融核心、医疗 PHI、政府数据
```

### 2.2 身份隔离

身份隔离是**第一道防线**。租户 ID 必须在请求的最早阶段确定，并贯穿始终。

```python
import uuid
from dataclasses import dataclass
from typing import Optional

@dataclass
class TenantContext:
    """租户上下文——所有请求都必须携带"""
    tenant_id: str
    user_id: str
    user_roles: list[str]
    tenant_plan: str  # basic / pro / enterprise

class TenantContextMiddleware:
    """从请求中提取并验证租户上下文"""
    
    async def resolve(self, request_headers: dict) -> TenantContext:
        # 1. 从 Authorization header 解析 JWT
        token = request_headers.get("Authorization", "").replace("Bearer ", "")
        
        # 2. 验证 Token 签名
        payload = self.verify_jwt(token)
        
        # 3. 提取租户 ID（服务端注入，而非客户端提供）
        tenant_id = payload.get("tenant_id")
        if not tenant_id:
            raise PermissionError("Missing tenant context")
        
        # 4. 验证租户状态（是否活跃、是否欠费）
        tenant = await self.tenant_service.get_tenant(tenant_id)
        if not tenant or tenant.status != "active":
            raise PermissionError("Tenant not active")
        
        # 5. 返回租户上下文
        return TenantContext(
            tenant_id=tenant_id,
            user_id=payload["sub"],
            user_roles=payload.get("roles", []),
            tenant_plan=tenant.plan,
        )
    
    def verify_jwt(self, token: str) -> dict:
        """JWT 签名验证（使用公钥）"""
        try:
            return jwt.decode(
                token,
                self.public_key,
                algorithms=["RS256"],
                audience=self.expected_audience,
            )
        except jwt.ExpiredSignatureError:
            raise PermissionError("Token expired")
        except jwt.InvalidTokenError:
            raise PermissionError("Invalid token")
```

### 2.3 数据隔离

#### 向量数据库隔离

不同规模下的向量数据隔离策略：

```
小规模（< 100 万向量/租户）: Metadata 过滤
  └── 所有租户共享 Collection，查询时 WHERE tenant_id = ?
  └── 优点: 管理简单
  └── 风险: 过滤遗漏 = 数据泄露
  └── 防御: 查询结果二次校验

中规模（100 万-1000 万向量/租户）: 独立 Collection
  └── 每个租户独立 Collection
  └── 优点: 天然隔离，查询效率高
  └── 风险: 管理多个 Collection，跨租户操作复杂
  └── 防御: Collection 命名规范 + 访问控制

大规模（> 1000 万向量/租户）: 独立索引实例
  └── 大租户独立索引/集群
  └── 优点: 独立扩缩容，性能隔离
  └── 风险: 资源碎片化，运维复杂
  └── 防御: 统一管理面 + 自动化运维
```

#### 向量隔离代码实现

```python
from typing import Optional
import chromadb
from chromadb.config import Settings

class TenantVectorStore:
    """多租户向量存储管理器"""
    
    def __init__(self, isolation_mode: str = "metadata"):
        """
        isolation_mode:
          - "metadata": 共享 Collection + tenant_id 过滤
          - "collection": 独立 Collection
          - "instance": 独立客户端实例
        """
        self.isolation_mode = isolation_mode
        self.client = chromadb.Client(Settings(persist_directory="./vector_db"))
        self._collections: dict[str, chromadb.Collection] = {}
    
    def _collection_name(self, tenant_id: str) -> str:
        if self.isolation_mode == "collection":
            return f"tenant_{tenant_id}"
        return "shared"
    
    def get_collection(self, tenant_id: str) -> chromadb.Collection:
        name = self._collection_name(tenant_id)
        if name not in self._collections:
            self._collections[name] = self.client.get_or_create_collection(name=name)
        return self._collections[name]
    
    async def query(
        self,
        tenant_id: str,
        query_embedding: list[float],
        top_k: int = 10,
    ) -> list[dict]:
        """带租户隔离的向量检索"""
        collection = self.get_collection(tenant_id)
        
        # 构建查询
        where = None
        if self.isolation_mode == "metadata":
            where = {"tenant_id": tenant_id}
        
        # 执行检索
        results = collection.query(
            query_embeddings=[query_embedding],
            n_results=top_k,
            where=where,
        )
        
        # ★ 二次校验：即使 metadata 过滤失败，也能兜底
        items = []
        for i, metadata in enumerate(results["metadatas"][0]):
            if metadata.get("tenant_id") != tenant_id:
                continue  # 跳过不属于本租户的结果
            items.append({
                "id": results["ids"][0][i],
                "text": results["documents"][0][i],
                "metadata": metadata,
                "score": results["distances"][0][i],
            })
        
        return items
    
    async def add_documents(
        self,
        tenant_id: str,
        documents: list[str],
        embeddings: list[list[float]],
        metadata: list[dict],
        ids: list[str],
    ):
        """带租户标签的文档写入"""
        collection = self.get_collection(tenant_id)
        
        # 强制注入 tenant_id 元数据
        enriched_metadata = []
        for m in metadata:
            m["tenant_id"] = tenant_id  # 服务端强制注入，客户端不可伪造
            enriched_metadata.append(m)
        
        collection.add(
            documents=documents,
            embeddings=embeddings,
            metadatas=enriched_metadata,
            ids=ids,
        )
```

### 2.4 模型配额隔离

租户级配额管理是防止"吵闹邻居"问题的关键。

```python
import time
from collections import defaultdict
from dataclasses import dataclass

@dataclass
class QuotaConfig:
    """租户配额配置"""
    rpm_limit: int = 60        # 每分钟请求数
    tpm_limit: int = 100000    # 每分钟 Token 数
    concurrent_limit: int = 5  # 最大并发
    daily_token_limit: int = 1000000  # 每日 Token 上限

class TenantQuotaManager:
    """多租户配额管理器"""
    
    def __init__(self):
        # tenant_id -> QuotaConfig
        self.configs: dict[str, QuotaConfig] = {}
        # 滑动窗口计数器
        self._rpm_counters: dict[str, list[float]] = defaultdict(list)
        self._tpm_counters: dict[str, list[int]] = defaultdict(list)
        self._daily_token_counters: dict[str, int] = defaultdict(int)
        self._last_daily_reset: dict[str, float] = {}
    
    async def check_quota(
        self, tenant_id: str, estimated_tokens: int = 0
    ) -> tuple[bool, str]:
        """
        检查租户配额是否超限
        返回 (是否允许, 拒绝原因)
        """
        config = self.configs.get(tenant_id, QuotaConfig())
        now = time.time()
        
        # 1. RPM 检查（滑动窗口 60s）
        self._clean_old_counts(tenant_id, now)
        rpm_count = len(self._rpm_counters[tenant_id])
        if rpm_count >= config.rpm_limit:
            return False, "rate_limit_exceeded"
        
        # 2. TPM 检查
        tpm_sum = sum(self._tpm_counters[tenant_id])
        if tpm_sum + estimated_tokens > config.tpm_limit:
            return False, "token_quota_exceeded"
        
        # 3. 日配额检查
        self._reset_daily_if_needed(tenant_id)
        if self._daily_token_counters[tenant_id] + estimated_tokens > config.daily_token_limit:
            return False, "daily_quota_exceeded"
        
        return True, "ok"
    
    def record_usage(self, tenant_id: str, tokens_used: int):
        """记录实际用量"""
        now = time.time()
        self._rpm_counters[tenant_id].append(now)
        self._tpm_counters[tenant_id].append(tokens_used)
        self._daily_token_counters[tenant_id] += tokens_used
    
    def _clean_old_counts(self, tenant_id: str, now: float):
        window = now - 60
        self._rpm_counters[tenant_id] = [
            t for t in self._rpm_counters[tenant_id] if t > window
        ]
        # TPM 用最近 60s 的 Token
        idx = 0
        while idx < len(self._rpm_counters[tenant_id]):
            if self._rpm_counters[tenant_id][idx] > window:
                break
            idx += 1
        self._tpm_counters[tenant_id] = self._tpm_counters[tenant_id][idx:]
    
    def _reset_daily_if_needed(self, tenant_id: str):
        last = self._last_daily_reset.get(tenant_id, 0)
        if time.time() - last > 86400:
            self._daily_token_counters[tenant_id] = 0
            self._last_daily_reset[tenant_id] = time.time()
```

### 2.5 工具权限隔离

不同租户需要不同的工具权限——

```python
from dataclasses import dataclass, field

@dataclass
class ToolPermission:
    tool_name: str
    allowed_actions: list[str] = field(default_factory=lambda: ["read"])
    allowed_params: dict[str, list[str]] = field(default_factory=dict)
    rate_limit: int = 100  # 每小时

# 租户级别的工具权限配置
TENANT_TOOL_PERMISSIONS = {
    "tenant_finance_001": {
        "sql_query": ToolPermission(
            tool_name="sql_query",
            allowed_actions=["read"],
            allowed_params={"database": ["finance_db", "analytics_db"]},
            rate_limit=200,
        ),
        "email_send": ToolPermission(
            tool_name="email_send",
            allowed_actions=["write"],
            allowed_params={"recipient_domain": ["@company.com"]},
            rate_limit=50,
        ),
        "file_delete": ToolPermission(
            tool_name="file_delete",
            allowed_actions=[],  # 不允许删除操作
        ),
    },
    "tenant_retail_001": {
        "sql_query": ToolPermission(
            tool_name="sql_query",
            allowed_actions=["read"],
            allowed_params={"database": ["retail_db"]},
            rate_limit=100,
        ),
        "email_send": ToolPermission(
            tool_name="email_send",
            allowed_actions=["read", "write"],
            rate_limit=20,
        ),
    },
}

class TenantToolAuthorizer:
    """租户工具授权器"""
    
    def __init__(self, permissions: dict):
        self.permissions = permissions
    
    def authorize(
        self, tenant_id: str, tool_name: str, action: str, params: dict
    ) -> tuple[bool, str]:
        """检查租户是否有权限执行此工具调用"""
        
        tenant_perms = self.permissions.get(tenant_id, {})
        tool_perm = tenant_perms.get(tool_name)
        
        if not tool_perm:
            return False, f"Tool '{tool_name}' not allowed for tenant"
        
        # 检查动作权限
        if action not in tool_perm.allowed_actions:
            return False, f"Action '{action}' not allowed for tool '{tool_name}'"
        
        # 检查参数级权限
        for param_name, allowed_values in tool_perm.allowed_params.items():
            if param_name in params:
                param_value = params[param_name]
                if param_value not in allowed_values:
                    return False, (
                        f"Parameter '{param_name}={param_value}' not allowed. "
                        f"Allowed: {allowed_values}"
                    )
        
        return True, "ok"
```

### 2.6 缓存隔离

```python
class TenantCacheKeyBuilder:
    """租户感知的缓存 Key 构建器"""
    
    @staticmethod
    def build_key(tenant_id: str, key_type: str, *parts) -> str:
        """
        构建包含租户上下文的缓存 Key
        
        格式: {tenant_id}:{key_type}:{part1}:{part2}:...
        示例: tenant_finance_001:response:weather:beijing
        """
        return f"{tenant_id}:{key_type}:" + ":".join(parts)
    
    @staticmethod
    def build_semantic_cache_key(tenant_id: str, query: str, model: str) -> str:
        """语义缓存 Key（含租户）"""
        return f"semantic:{tenant_id}:{model}:{query}"
    
    @staticmethod
    def build_tool_cache_key(tenant_id: str, tool_name: str, params_hash: str) -> str:
        """工具结果缓存 Key（含租户）"""
        return f"tool:{tenant_id}:{tool_name}:{params_hash}"
```

### 2.7 日志隔离

```python
import json
import hashlib
from datetime import datetime

class TenantAwareLogger:
    """租户感知的结构化日志器"""
    
    def __init__(self, es_client):
        self.es = es_client
    
    async def log_agent_action(
        self,
        tenant_id: str,
        log_entry: dict,
    ):
        """记录 Agent 操作日志（自动注入租户上下文）"""
        
        # 强制注入租户 ID（防止客户端伪造）
        log_entry["@tenant_id"] = tenant_id
        log_entry["@timestamp"] = datetime.utcnow().isoformat()
        
        # 敏感字段脱敏
        if "user_input" in log_entry:
            log_entry["user_input"] = self.mask_pii(log_entry["user_input"])
        if "tool_result" in log_entry:
            log_entry["tool_result"] = self.mask_pii(log_entry["tool_result"])
        
        # 写入 ES（按租户分区）
        index = f"agent-logs-{tenant_id}-{datetime.utcnow():%Y%m%d}"
        await self.es.index(index=index, document=log_entry)
    
    async def query_logs(self, tenant_id: str, query: dict) -> list[dict]:
        """查询租户的日志（强制作用域）"""
        # 强制添加租户过滤
        query.setdefault("bool", {}).setdefault("filter", [])
        query["bool"]["filter"].append({"term": {"@tenant_id": tenant_id}})
        
        result = await self.es.search(index=f"agent-logs-{tenant_id}-*", body={"query": query})
        return result["hits"]["hits"]
    
    def mask_pii(self, text: str) -> str:
        """脱敏 PII 信息"""
        import re
        # 邮箱脱敏
        text = re.sub(r'[\w.]+@[\w.]+', '***@***', text)
        # 手机号脱敏
        text = re.sub(r'1[3-9]\d{9}', '***********', text)
        # 身份证脱敏
        text = re.sub(r'\d{18}|\d{17}X', '******************', text)
        return text
```

---

## 3. 隔离的挑战与反模式

### 3.1 常见失败模式

```
失败模式 1: 过滤遗漏
  场景: 向量检索时 WHERE 条件拼接错误，遗漏 tenant_id 过滤
  结果: 租户 A 检索到租户 B 的文档
  防御: 二次校验 + 自动化测试 + 红队演练

失败模式 2: 上下文泄漏
  场景: 异步任务处理时丢失租户上下文
  结果: 回调处理时使用了错误租户的配置
  防御: 消息体携带租户上下文，不从线程变量获取

失败模式 3: 缓存污染
  场景: 语义缓存 Key 不包含租户 ID
  结果: 租户 B 看到了租户 A 缓存的回答
  防御: Cache Key 强制包含 tenant_id

失败模式 4: 配置互串
  场景: 共享的 Prompt 模板被一个租户修改，影响所有租户
  结果: 所有租户的 Agent 行为异常
  防御: 配置版本化 + 租户作用域 + 变更审批
```

### 3.2 隔离反模式

| 反模式 | 表现 | 正确做法 |
|--------|------|----------|
| Prompt 层隔离 | "你只能访问 A 的数据" | 基础设施层强制隔离 |
| 租户 ID 来自客户端 | 前端传 tenant_id | 服务端从 JWT 解析 |
| 字段级隔离一刀切 | 所有字段同一隔离策略 | 按敏感度分级隔离 |
| 隔离测试只测正向 | 只测自己能看到什么 | 更要测不能看到什么 |
| 物理隔离才安全 | 直接上独立实例 | 按实际需求选型 |

---

## 4. 能力边界

### 4.1 隔离能做到什么

- **数据安全**：确保租户 A 无法访问租户 B 的数据
- **性能隔离**：防止一个租户的突发流量影响其他租户
- **配置独立**：每个租户拥有独立的 Agent 行为和风格
- **合规满足**：满足 GDPR/SOC 2 等合规要求

### 4.2 隔离做不到什么

- **绝对安全**：没有 100% 的安全隔离，只有可接受的剩余风险
- **零成本隔离**：隔离越强，基础设施和运维成本越高
- **完全性能独立**：共享层（如 LLM API 提供商）仍可能被其他租户影响
- **零延迟隔离**：硬隔离增加网络跳数和延迟

### 4.3 何时应该拆分

当出现以下信号时，考虑物理隔离/独立部署：

```
1. 合规要求: 租户的数据必须物理驻留在特定区域
2. 性能需求: 租户的负载足以独立使用资源
3. 版本分歧: 租户需要不同的 Agent 版本
4. 故障隔离: 一个租户的故障不应影响其他租户
5. 合同要求: 客户合同明确要求物理隔离
```

---

## 5. 核心优势

- **SaaS 经济性**：一套基础设施服务多个客户
- **统一运维**：版本升级、安全补丁一次部署所有租户受益
- **快速入驻**：新租户无需部署，配置即可开通
- **集中监控**：跨租户的可观测性视图

---

## 6. 与其他内容的区别

| 对比 | 多租户隔离 | SSO 集成 | 企业安全 |
|------|-----------|----------|----------|
| 关注点 | 租户间边界 | 用户身份认证 | 整体安全防护 |
| 核心问题 | 如何隔离 | 如何认证 | 如何保护 |
| 技术手段 | 过滤/Collection/实例 | JWT/OAuth/SAML | 加密/审计/隔离 |
| 失败后果 | 跨租户数据泄露 | 未授权访问 | 安全事件 |

---

## 7. 工程优化方向

### 7.1 自动化隔离测试

```python
# 隔离测试的核心原则：测试不能看到什么
def test_tenant_isolation():
    # 租户 A 写入数据
    tenant_a = TenantContext(tenant_id="A", user_id="user_a", user_roles=["admin"], tenant_plan="pro")
    add_document(tenant_a, "租户 A 的机密文档")
    
    # 租户 B 尝试检索
    tenant_b = TenantContext(tenant_id="B", user_id="user_b", user_roles=["user"], tenant_plan="basic")
    results = search_documents(tenant_b, "机密文档")
    
    # 租户 B 不应看到 A 的数据
    assert len(results) == 0, "隔离失败！租户 B 看到了租户 A 的数据"
```

### 7.2 混合隔离策略

```
小租户(L1): Metadata 过滤 → 低成本
中租户(L2): 独立 Collection → 中等成本
大租户(L3): 独立实例 → 高性能

自动升降级: 基于租户的数据量和负载自动调整隔离级别
```

### 7.3 隔离监控

```python
# 需要监控的关键指标
ISOLATION_METRICS = [
    "跨租户访问尝试次数",          # 安全事件
    "隔离过滤命中率",              # 正常隔离工作
    "元数据过滤遗漏次数",          # 二次校验发现的问题
    "租户配额使用率",              # 配额管理健康度
    "租户级 P50/P95 延迟",         # 性能隔离效果
]
```

---

## 8. 完整参考方案：生产级多租户架构

```
                                    ┌──────────────────────────┐
                                    │     负载均衡器 (LB)        │
                                    │   Tenant-aware 路由       │
                                    └────────────┬─────────────┘
                                                 │
                                    ┌────────────▼─────────────┐
                                    │     API 网关 (Gateway)     │
                                    │  ├─ JWT 解析 → Tenant ID │
                                    │  ├─ Rate Limit (租户级)   │
                                    │  └─ 请求日志 (含租户)     │
                                    └────────────┬─────────────┘
                                                 │
                                    ┌────────────▼─────────────┐
                                    │    Agent 编排层            │
                                    │  ├─ 租户配置注入           │
                                    │  ├─ 工具权限校验           │
                                    │  └─ 上下文传播             │
                                    └────────────┬─────────────┘
                                                 │
                    ┌────────────────────────────┼────────────────────────────┐
                    │                            │                            │
            ┌───────▼───────┐          ┌─────────▼─────────┐       ┌─────────▼─────────┐
            │  向量数据库     │          │   LLM API 代理     │       │   工具执行器        │
            │  Metadata过滤  │          │   租户级配额       │       │   租户权限校验      │
            │  二次校验      │          │   模型路由         │       │   参数级授权        │
            └───────────────┘          └───────────────────┘       └───────────────────┘
                    │                            │                            │
                    └────────────────────────────┼────────────────────────────┘
                                                 │
                                    ┌────────────▼─────────────┐
                                    │     数据层                  │
                                    │  ├─ Postgres (tenant_id 分区)│
                                    │  ├─ Redis (Key=tenant_id:*) │
                                    │  └─ S3 (bucket/tenant_id/)  │
                                    └────────────────────────────┘
```

---

> **核心结论**：多租户隔离是 Agent 平台进入企业市场的**准入门槛**。隔离不是"加一个字段"的事，而是需要在架构的每一层（身份→数据→模型→工具→缓存→日志）都建立清晰的租户边界。"测试隔离是否有效"的最佳方法是：让一个租户尝试访问另一个租户的资源，看能否成功——**不能成功才是正确**。
