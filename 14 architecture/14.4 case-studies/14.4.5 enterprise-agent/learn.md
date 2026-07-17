# 企业 Agent 平台架构：多租户、安全合规与大规模部署

## 1. 业务背景

### 1.1 什么是企业 Agent 平台

企业 Agent 平台是为大型组织内部的多个团队、部门或业务单元提供 AI Agent 服务的综合性基础设施平台。与面向个人用户的 Agent 产品不同，企业 Agent 平台需要解决多租户隔离、企业级安全合规、与现有 IT 系统深度集成、以及运营管理等复杂问题。典型的企业 Agent 场景包括：内部知识库问答、IT 技术支持、HR 自助服务、法务合同审查、财务报销流程等。

```
个人 Agent 产品                       企业 Agent 平台
┌─────────────────┐              ┌─────────────────────────┐
│ 单用户           │              │ 多租户（部门/团队隔离）    │
│ 无需 SSO         │              │ SSO + RBAC + 审计       │
│ 通用知识         │     ──→      │ 企业私有知识库            │
│ 不可定制         │              │ 可定制 Agent 模板        │
│ 无 SLA 承诺      │              │ SLA 99.9% + 7x24 支持   │
│ 公有云部署       │              │ 混合云 / 私有化部署       │
└─────────────────┘              └─────────────────────────┘
```

### 1.2 企业需求驱动力

| 需求 | 说明 |
|------|------|
| **降本增效** | 通过 AI 自动化减少重复性人力投入，每 Agent 可节省 3-5 FTE |
| **知识管理** | 沉淀组织知识资产，消除信息孤岛，新员工 onboarding 周期缩短 60% |
| **合规要求** | 满足 GDPR、等保、SOX 等法规对数据处理和存储的要求 |
| **快速响应** | IT/HR/财务等内部服务的 7x24 自助服务能力 |
| **标准化与复用** | 一次开发的 Agent 能力可在多个部门复用 |

---

## 2. 系统架构

### 2.1 总体架构

```
                                   ┌─────────────────────────────────────┐
                                   │          企业统一身份认证            │
                                   │   SSO (SAML/OIDC) + LDAP + MFA     │
                                   └──────────────┬──────────────────────┘
                                                  │
┌──────────────────────────────────────────────────┼──────────────────────────────────────────┐
│                  全局入口层                       │                                          │
│  ┌───────────────────────────────────────────────┴────────────────────────────────────┐     │
│  │                      API Gateway / Load Balancer                                   │     │
│  │          请求路由 | 限流 | 认证委托 | CORS | 请求/响应日志                          │     │
│  └───────────────────────────────────────────────┬────────────────────────────────────┘     │
└──────────────────────────────────────────────────┼──────────────────────────────────────────┘
                                                   │
┌──────────────────────────────────────────────────┼──────────────────────────────────────────┐
│                  多租户隔离层                     │                                          │
│  ┌───────────────────────────────────────────────▼────────────────────────────────────┐     │
│  │                   Tenant Context Middleware                                         │     │
│  │  x-tenant-id 提取 → 租户配置加载 → 数据源路由 → 资源隔离 → 速率限制(per tenant)    │     │
│  └───────────────────────────────────────────────┬────────────────────────────────────┘     │
└──────────────────────────────────────────────────┼──────────────────────────────────────────┘
                                                   │
┌──────────────────────────────────────────────────┼──────────────────────────────────────────┐
│                 Agent 管理平面                    │                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────▼───────┐ ┌──────────┐                    │
│  │ Agent     │ │ Agent    │ │ 模板     │ │ 版本管理      │ │ 部署管理  │                    │
│  │ 目录/商店  │ │ 生命周期  │ │ 引擎     │ │ 灰度发布      │ │ 扩缩容    │                    │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────┘ └──────────┘                    │
└────────────────────────────────────────────────────────────────────────────────────────────┘
                                                   │
┌──────────────────────────────────────────────────┼──────────────────────────────────────────┐
│              核心 Agent 运行时                    │                                          │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────▼───────────┐ ┌──────────────────┐       │
│  │ LLM 网关         │ │ RAG 引擎         │ │ 工具执行引擎      │ │ 对话管理         │       │
│  │ • 模型路由        │ │ • 向量检索        │ │ • 企业连接器      │ │ • 上下文管理      │       │
│  │ • 模型负载均衡    │ │ • 混合搜索        │ │ • API 编排       │ │ • 会话持久化     │       │
│  │ • 成本核算        │ │ • 租户知识隔离    │ │ • 审批流集成     │ │ • 历史管理       │       │
│  │ • 内容审核        │ │ • 多模态索引      │ │ • 数据脱敏       │ │                  │       │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘ └──────────────────┘       │
└──────────────────────────────────────────────────┬──────────────────────────────────────────┘
                                                   │
┌──────────────────────────────────────────────────┼──────────────────────────────────────────┐
│              集成网关                            │                                          │
│  ┌───────────────────────────────────────────────▼────────────────────────────────────┐     │
│  │          Enterprise Connector 适配器阵列                                            │     │
│  │   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐       │     │
│  │   │ SAP  │ │Sales │ │Service│ │Work  │ │JIRA  │ │Share │ │Slack │ │Custom│       │     │
│  │   │      │ │force │ │ Now   │ │day   │ │      │ │Point │ │      │ │API   │       │     │
│  │   └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘       │     │
│  └────────────────────────────────────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────────────────────────────────┘
                                                   │
┌──────────────────────────────────────────────────┼──────────────────────────────────────────┐
│              安全合规层                          │                                          │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐                  │
│  │ 审计日志      │ │ DLP           │ │ 审批工作流    │ │ 数据脱敏引擎     │                  │
│  │ 全量操作记录   │ │ 敏感数据检测   │ │ 多级审批      │ │ PII / 财务数据   │                  │
│  │ 不可篡改存储   │ │ 数据泄露防护   │ │ 自动升级      │ │ 差分隐私         │                  │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────────┘                  │
└────────────────────────────────────────────────────────────────────────────────────────────┘
                                                   │
┌──────────────────────────────────────────────────┼──────────────────────────────────────────┐
│              运营平台                            │                                          │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐                  │
│  │ 租户监控      │ │ 成本追踪      │ │ 使用分析      │ │ 健康检查          │                  │
│  │ • 租户级别    │ │ • Token 消耗  │ │ • 热门查询    │ │ • 租户级存活检测   │                  │
│  │ • Agent 级别  │ │ • API 调用量  │ │ • 满意度评分  │ │ • 依赖服务可用性   │                  │
│  │ • 响应延迟    │ │ • 成本分摊     │ │ • 用户反馈    │ │ • 自动告警        │                  │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────────┘                  │
└────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 多租户隔离层（Multi-Tenant Layer）

多租户是企业平台与个人产品最本质的区别。租户可能是一个部门、一个子公司、或一个独立的业务单元。

**租户隔离策略对比：**

```
隔离级别                     隔离强度       共享度        运维成本       适用场景
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. 数据库级隔离 (Database-per-Tenant)                                       │
│    Tenant A ──► DB_A                                                        │
│    Tenant B ──► DB_B        ★★★★★       低           高          金融/合规 │
│    Tenant C ──► DB_C                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│ 2. Schema 级隔离 (Schema-per-Tenant)                                        │
│    DB ──► Schema_A                                                          │
│    DB ──► Schema_B       ★★★★☆      中           中           中型企业   │
│    DB ──► Schema_C                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│ 3. 表级隔离 (Shared Table with Tenant ID)                                   │
│    DB ──► Table (tenant_id列)                                               │
│           tenant_id=A  ★★★☆☆       高           低           标准服务   │
│           tenant_id=B                                                        │
│           tenant_id=C                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

**多租户中间件实现：**

```python
import functools
import logging
from typing import Dict, Optional, Callable, Any
from contextvars import ContextVar
from fastapi import Request, HTTPException
from sqlalchemy import create_engine, event
from sqlalchemy.orm import Session, sessionmaker

logger = logging.getLogger(__name__)

# 使用 ContextVar 实现线程/协程安全的租户上下文
current_tenant: ContextVar[Optional[str]] = ContextVar("current_tenant", default=None)


class TenantContext:
    """租户上下文管理器 - 线程/协程安全"""
    
    @classmethod
    def get_tenant_id(cls) -> Optional[str]:
        return current_tenant.get()
    
    @classmethod
    def set_tenant_id(cls, tenant_id: str):
        current_tenant.set(tenant_id)
    
    @classmethod
    def clear(cls):
        current_tenant.set(None)


class TenantAwareRouter:
    """租户感知的路由中间件"""
    
    def __init__(self, tenant_configs: Dict[str, dict]):
        """
        tenant_configs: {
            "tenant_a": {
                "db_url": "postgresql://...",
                "vector_store": "pinecone_index_a",
                "llm_model": "claude-sonnet",
                "rate_limit": 100,
                "features": ["rag", "tools", "approval"]
            }
        }
        """
        self.tenant_configs = tenant_configs
        self._engines: Dict[str, any] = {}
    
    async def __call__(self, request: Request, call_next):
        # 1. 从请求中提取租户 ID
        tenant_id = self._extract_tenant(request)
        
        if not tenant_id:
            raise HTTPException(status_code=401, 
                                detail="缺少租户标识")
        
        if tenant_id not in self.tenant_configs:
            raise HTTPException(status_code=403, 
                                detail="无效的租户")
        
        # 2. 验证租户访问权限
        self._validate_tenant_access(tenant_id, request)
        
        # 3. 设置租户上下文
        TenantContext.set_tenant_id(tenant_id)
        
        # 4. 针对该租户的速率限制
        await self._check_rate_limit(tenant_id, request)
        
        try:
            # 5. 请求处理（后续所有代码可通过 TenantContext 获取租户）
            response = await call_next(request)
            
            # 6. 添加租户标识到响应头
            response.headers["X-Tenant-Id"] = tenant_id
            return response
        finally:
            # 7. 清理租户上下文
            TenantContext.clear()
    
    def _extract_tenant(self, request: Request) -> Optional[str]:
        """从请求中提取租户 ID"""
        # 优先从 header 获取
        tenant_id = request.headers.get("X-Tenant-Id")
        if tenant_id:
            return tenant_id
        
        # 从 JWT token 中的 claim 获取
        token = request.headers.get("Authorization", "")
        if token.startswith("Bearer "):
            try:
                # 解码 JWT 获取租户信息
                payload = self._decode_jwt(token[7:])
                return payload.get("tenant_id")
            except Exception:
                pass
        
        # 从子域名获取
        host = request.headers.get("Host", "")
        if "." in host:
            subdomain = host.split(".")[0]
            if subdomain in self.tenant_configs:
                return subdomain
        
        return None
    
    def get_tenant_engine(self, tenant_id: str = None):
        """获取租户专属的数据库引擎"""
        tid = tenant_id or TenantContext.get_tenant_id()
        if not tid:
            raise ValueError("无法确定租户")
        
        if tid not in self._engines:
            config = self.tenant_configs[tid]
            self._engines[tid] = create_engine(
                config["db_url"],
                pool_size=10,
                max_overflow=20,
                # 连接池按租户隔离
                pool_pre_ping=True
            )
        
        return self._engines[tid]


class TenantAwareSession:
    """租户感知的数据库会话"""
    
    def __init__(self, router: TenantAwareRouter):
        self.router = router
    
    def __enter__(self) -> Session:
        tenant_id = TenantContext.get_tenant_id()
        engine = self.router.get_tenant_engine(tenant_id)
        session = sessionmaker(bind=engine)()
        
        # 设置数据库会话级别的租户上下文
        session.execute(
            "SET app.current_tenant = :tenant_id",
            {"tenant_id": tenant_id}
        )
        
        return session
    
    def __exit__(self, *args):
        pass


# 装饰器：标记需要特定租户权限的端点
def require_tenant_feature(feature: str):
    """要求租户具有特定功能权限"""
    def decorator(func: Callable):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            tenant_id = TenantContext.get_tenant_id()
            config = getattr(func, "_tenant_configs", {}).get(tenant_id, {})
            features = config.get("features", [])
            
            if feature not in features:
                raise HTTPException(
                    status_code=403,
                    detail=f"租户 {tenant_id} 未开通 {feature} 功能"
                )
            
            return await func(*args, **kwargs)
        return wrapper
    return decorator
```

### 2.3 Agent 管理平面（Management Plane）

管理平面是企业平台的核心控制面，负责 Agent 的全生命周期管理。

```
Agent 生命周期:

                     ┌──────────────┐
                     │   创建 Agent  │
                     │  (模板/自定义) │
                     └──────┬───────┘
                            ▼
                     ┌──────────────┐
                     │   配置阶段    │
                     │ • 知识库绑定  │
                     │ • 工具授权    │
                     │ • 权限设置   │◄──── 审批流
                     │ • 模型选择   │
                     └──────┬───────┘
                            ▼
                     ┌──────────────┐
                     │   测试阶段    │
                     │ • 沙箱环境    │
                     │ • 回归测试    │
                     │ • 安全扫描    │
                     └──────┬───────┘
                            ▼
               ┌────────────┴────────────┐
               ▼                         ▼
        ┌──────────────┐        ┌──────────────┐
        │   灰度发布    │        │   直接部署    │
        │ • 5% → 20%   │        │              │
        │ → 50% → 100% │        │              │
        └──────┬───────┘        └──────┬───────┘
               │                       │
               └───────┬───────────────┘
                       ▼
                ┌──────────────┐
                │   运行监控    │
                │ • 性能指标    │
                │ • 异常检测    │
                │ • 用户反馈    │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │   版本迭代    │
                │ • A/B 测试   │
                │ • 升级/回滚  │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │   下线归档    │
                │ • 数据导出    │
                │ • 审计归档    │
                └──────────────┘
```

```python
from enum import Enum
from typing import Dict, List, Optional
from datetime import datetime
import uuid


class AgentStatus(Enum):
    DRAFT = "draft"
    TESTING = "testing"
    APPROVED = "approved"
    DEPLOYED = "deployed"
    DEPRECATED = "deprecated"
    ARCHIVED = "archived"


class AgentTemplate:
    """Agent 模板定义"""
    
    def __init__(self, template_id: str, name: str,
                 category: str, description: str,
                 system_prompt_template: str,
                 required_tools: List[str],
                 default_config: dict):
        self.template_id = template_id
        self.name = name
        self.category = category          # hr, it, legal, finance...
        self.description = description
        self.system_prompt_template = system_prompt_template
        self.required_tools = required_tools
        self.default_config = default_config


class AgentDefinition:
    """Agent 实例定义"""
    
    def __init__(self, agent_id: str, tenant_id: str,
                 template_id: str, name: str,
                 config: dict, status: AgentStatus = AgentStatus.DRAFT):
        self.agent_id = agent_id
        self.tenant_id = tenant_id
        self.template_id = template_id
        self.name = name
        self.config = config
        self.status = status
        self.version = 1
        self.created_at = datetime.utcnow()
        self.updated_at = datetime.utcnow()
        self.deployed_at: Optional[datetime] = None


class AgentManager:
    """Agent 管理器 - 管理平面核心"""
    
    def __init__(self, storage, template_registry):
        self.storage = storage          # Agent 元数据存储
        self.templates = template_registry  # 模板注册表
    
    async def create_from_template(self, tenant_id: str,
                                    template_id: str,
                                    name: str,
                                    overrides: dict = None) -> AgentDefinition:
        """从模板创建 Agent"""
        template = self.templates.get(template_id)
        if not template:
            raise ValueError(f"模板 {template_id} 不存在")
        
        config = template.default_config.copy()
        if overrides:
            config.update(overrides)
        
        agent = AgentDefinition(
            agent_id=str(uuid.uuid4()),
            tenant_id=tenant_id,
            template_id=template_id,
            name=name,
            config=config
        )
        
        await self.storage.save(agent)
        return agent
    
    async def deploy_agent(self, agent_id: str) -> dict:
        """部署 Agent（含灰度策略）"""
        agent = await self.storage.get(agent_id)
        if not agent:
            raise ValueError(f"Agent {agent_id} 不存在")
        
        if agent.status != AgentStatus.APPROVED:
            raise ValueError(f"Agent 状态为 {agent.status.value}，无法部署")
        
        canary_config = agent.config.get("deployment", {}).get("canary", {})
        
        deployment = {
            "agent_id": agent_id,
            "version": agent.version,
            "canary_percentage": canary_config.get("percentage", 100),
            "canary_segments": canary_config.get("segments", []),
            "rollback_version": agent.version - 1,
            "status": "deploying",
            "deployed_at": datetime.utcnow().isoformat()
        }
        
        agent.status = AgentStatus.DEPLOYED
        agent.deployed_at = datetime.utcnow()
        await self.storage.save(agent)
        
        return deployment
    
    async def rollback_agent(self, agent_id: str) -> dict:
        """回滚 Agent 到上一版本"""
        agent = await self.storage.get(agent_id)
        if agent.version <= 1:
            raise ValueError("没有可回滚的版本")
        
        agent.version -= 1
        agent.status = AgentStatus.DEPLOYED
        await self.storage.save(agent)
        
        return {
            "agent_id": agent_id,
            "rollback_to_version": agent.version
        }
    
    async def list_tenant_agents(self, tenant_id: str) -> List[AgentDefinition]:
        """列出租户下的 Agent"""
        return await self.storage.list_by_tenant(tenant_id)
```

### 2.4 集成网关（Integration Gateway）

企业 Agent 需要与数十个内部系统交互，集成网关提供统一的连接器框架。

```
集成网关架构:

                    ┌─────────────────────────────────────┐
                    │         Integration Gateway          │
                    │                                      │
                    │  ┌──────────────────────────────┐   │
                    │  │    Connector Registry          │   │
                    │  │  ┌──────┐ ┌──────┐ ┌──────┐  │   │
                    │  │  │SAP   │ │Sales │ │Service│  │   │
                    │  │  │      │ │force │ │ Now   │  │   │
                    │  │  └──┬───┘ └──┬───┘ └──┬───┘  │   │
                    │  │  ┌──────┐ ┌──────┐ ┌──────┐  │   │
                    │  │  │AD    │ │JIRA  │ │Slack │  │   │
                    │  │  │      │ │      │ │      │  │   │
                    │  │  └──┬───┘ └──┬───┘ └──┬───┘  │   │
                    │  └──────┼────────┼────────┼─────┘   │
                    │         │        │        │         │
                    │  ┌──────▼────────▼────────▼─────┐   │
                    │  │    Connector Runtime           │   │
                    │  │  • 认证管理 (Basic/OAuth/JWT)  │   │
                    │  │  • 重试与断路                   │   │
                    │  │  • 速率限制                     │   │
                    │  │  • 请求/响应转换                │   │
                    │  │  • 数据脱敏                     │   │
                    │  │  • 审计日志                     │   │
                    │  └──────────────────────────────┘   │
                    └─────────────────────────────────────┘
```

**企业连接器模式实现：**

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, Optional
from datetime import datetime
import asyncio
import hashlib
import hmac
import json
import logging

logger = logging.getLogger(__name__)


class ConnectorAuth(ABC):
    """连接器认证基类"""
    
    @abstractmethod
    async def get_headers(self) -> Dict[str, str]:
        pass
    
    @abstractmethod
    async def refresh_if_expired(self):
        pass


class OAuth2Auth(ConnectorAuth):
    """OAuth2 认证"""
    
    def __init__(self, client_id: str, client_secret: str,
                 token_url: str, scopes: List[str],
                 tenant_id: str):
        self.client_id = client_id
        self.client_secret = client_secret
        self.token_url = token_url
        self.scopes = scopes
        self.tenant_id = tenant_id
        self.access_token: Optional[str] = None
        self.expires_at: Optional[datetime] = None
    
    async def get_headers(self) -> Dict[str, str]:
        await self.refresh_if_expired()
        return {"Authorization": f"Bearer {self.access_token}"}
    
    async def refresh_if_expired(self):
        if not self.access_token or datetime.utcnow() >= self.expires_at:
            await self._refresh_token()
    
    async def _refresh_token(self):
        """刷新 OAuth token（实际中调用 OAuth 服务）"""
        logger.info(f"刷新租户 {self.tenant_id} 的 OAuth token")
        # 模拟 token 刷新
        self.access_token = f"token_{uuid.uuid4().hex}"
        self.expires_at = datetime.utcnow().replace(hour=23, minute=59)
        await self._store_token_securely()
    
    async def _store_token_securely(self):
        """安全存储 token（加密后存储到 vault）"""
        pass


class EnterpriseConnector(ABC):
    """企业连接器基类"""
    
    def __init__(self, connector_id: str, 
                 tenant_id: str,
                 config: dict,
                 auth: ConnectorAuth):
        self.connector_id = connector_id
        self.tenant_id = tenant_id
        self.config = config
        self.auth = auth
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=config.get("circuit_breaker_threshold", 5),
            recovery_timeout=config.get("recovery_timeout_sec", 60)
        )
    
    @abstractmethod
    async def execute(self, action: str, 
                      params: dict) -> dict:
        """执行企业系统操作"""
        pass
    
    @abstractmethod
    async def search(self, query: str, 
                      limit: int = 10) -> List[dict]:
        """搜索企业系统数据"""
        pass
    
    async def call_with_fault_tolerance(self, 
                                         method: str,
                                         url: str,
                                         headers: dict,
                                         body: dict = None,
                                         timeout_sec: int = 30) -> dict:
        """带容错的 API 调用"""
        
        if not self.circuit_breaker.allow_request():
            raise CircuitBreakerOpenError(
                f"断路器已打开，连接器 {self.connector_id} 暂时不可用"
            )
        
        auth_headers = await self.auth.get_headers()
        headers.update(auth_headers)
        
        last_error = None
        max_retries = self.config.get("max_retries", 3)
        
        for attempt in range(max_retries + 1):
            try:
                async with aiohttp.ClientSession() as session:
                    async with session.request(
                        method, url, headers=headers,
                        json=body,
                        timeout=aiohttp.ClientTimeout(total=timeout_sec)
                    ) as response:
                        if response.status == 429:  # Rate limited
                            retry_after = int(
                                response.headers.get("Retry-After", "5")
                            )
                            await asyncio.sleep(retry_after)
                            continue
                        
                        response.raise_for_status()
                        result = await response.json()
                        
                        # 记录审计日志
                        await self._log_audit(method, url, result)
                        
                        # 成功，重置断路器
                        self.circuit_breaker.record_success()
                        return result
                        
            except Exception as e:
                last_error = e
                logger.warning(
                    f"连接器 {self.connector_id} 调用失败 "
                    f"(尝试 {attempt+1}/{max_retries+1}): {str(e)}"
                )
                
                if attempt < max_retries:
                    wait_time = min(2 ** attempt * 2, 30)
                    await asyncio.sleep(wait_time)
                else:
                    self.circuit_breaker.record_failure()
        
        raise last_error
    
    async def _log_audit(self, method: str, url: str, result: dict):
        """记录审计日志"""
        audit_entry = {
            "connector_id": self.connector_id,
            "tenant_id": self.tenant_id,
            "method": method,
            "url": url,
            "timestamp": datetime.utcnow().isoformat(),
            "result_summary": str(result)[:200]
        }
        # 发送到审计日志服务
        await AuditService.log(audit_entry)


class SAPConnector(EnterpriseConnector):
    """SAP 系统连接器"""
    
    async def execute(self, action: str, params: dict) -> dict:
        """
        SAP 操作:
        - create_purchase_order
        - get_material_info
        - approve_requisition
        """
        endpoint = self.config["base_url"] + f"/api/sap/{action}"
        return await self.call_with_fault_tolerance(
            "POST", endpoint, {}, params
        )
    
    async def search(self, query: str, limit: int = 10) -> List[dict]:
        url = self.config["base_url"] + "/api/sap/search"
        body = {"query": query, "max_results": limit}
        result = await self.call_with_fault_tolerance("POST", url, {}, body)
        return result.get("results", [])


class ServiceNowConnector(EnterpriseConnector):
    """ServiceNow 连接器"""
    
    async def execute(self, action: str, params: dict) -> dict:
        """ServiceNow 操作: create_ticket, update_ticket, resolve_ticket"""
        endpoint = self.config["base_url"] + f"/api/now/{action}"
        return await self.call_with_fault_tolerance("POST", endpoint, {}, params)
    
    async def search(self, query: str, limit: int = 10) -> List[dict]:
        url = self.config["base_url"] + "/api/now/table/incident"
        params = {"sysparm_query": f"short_descriptionLIKE{query}",
                  "sysparm_limit": limit}
        result = await self.call_with_fault_tolerance("GET", url, {}, params)
        return result.get("result", [])


class CircuitBreaker:
    """断路器模式实现"""
    
    def __init__(self, failure_threshold: int = 5,
                 recovery_timeout: int = 60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time: Optional[datetime] = None
        self.state = "closed"  # closed, open, half-open
    
    def allow_request(self) -> bool:
        if self.state == "closed":
            return True
        
        if self.state == "open":
            if (datetime.utcnow() - self.last_failure_time).seconds > \
                    self.recovery_timeout:
                self.state = "half-open"
                return True
            return False
        
        # half-open 状态允许一个请求探测
        return True
    
    def record_success(self):
        self.failure_count = 0
        self.state = "closed"
    
    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.utcnow()
        
        if self.failure_count >= self.failure_threshold:
            self.state = "open"
            logger.warning(
                f"断路器打开: {self.failure_count} 次连续失败"
            )
```

### 2.5 安全合规层

```
企业安全架构纵深防御:

外层: 网络边界
  ┌─ WAF ──► DDoS防护 ──► IP白名单 ──► TLS 1.3 ─┐

中层: 认证授权
  ┌─ SSO(SAML/OIDC) ──► RBAC ──► 细粒度权限 ──►  just-in-time权限 ─┐

内层: 数据安全
  ┌─ 传输加密 ──► 存储加密(AES-256) ──► 字段级加密 ──► TEE ─┐

核心: 审计与合规
  ┌─ 不可篡改审计日志 ──► DLP ──► 合规报告 ──► 数据保留策略 ─┐
```

**审计日志系统实现：**

```python
class AuditService:
    """企业级审计日志服务"""
    
    def __init__(self, log_storage, 
                 crypto_service,
                 dlp_engine=None):
        self.storage = log_storage      # 持久化存储（不可变）
        self.crypto = crypto_service    # 签名/加密服务
        self.dlp = dlp_engine           # 数据防泄漏引擎
    
    @classmethod
    async def log(cls, entry: dict):
        """便捷日志记录"""
        # 实际使用中需从全局获取 audit service 实例
        service = get_audit_service()
        await service.record(entry)
    
    async def record(self, event: dict):
        """记录审计事件"""
        # 1. 标准化审计事件
        audit_entry = self._normalize(event)
        
        # 2. DLP 检查 - 确保审计日志本身不泄露敏感信息
        if self.dlp:
            audit_entry = await self.dlp.sanitize(audit_entry)
        
        # 3. 计算哈希链（防篡改）
        previous_hash = await self._get_last_hash()
        audit_entry["previous_hash"] = previous_hash
        audit_entry["hash"] = self._compute_hash(audit_entry)
        
        # 4. 数字签名
        audit_entry["signature"] = self.crypto.sign(
            audit_entry["hash"]
        )
        
        # 5. 持久化存储
        await self.storage.append(audit_entry)
        
        # 6. 实时告警（如果检测到安全事件）
        await self._check_alerts(audit_entry)
        
        return audit_entry["audit_id"]
    
    def _normalize(self, event: dict) -> dict:
        """标准化审计事件格式"""
        return {
            "audit_id": str(uuid.uuid4()),
            "timestamp": datetime.utcnow().isoformat(),
            "tenant_id": event.get("tenant_id") or \
                         TenantContext.get_tenant_id(),
            "actor_id": event.get("actor_id", "system"),
            "actor_type": event.get("actor_type", "user"),  # user/agent/system
            "action": event.get("action"),  # create/read/update/delete/execute
            "resource_type": event.get("resource_type"),  # knowledge/tool/agent
            "resource_id": event.get("resource_id"),
            "details": event.get("details", {}),
            "ip_address": event.get("ip_address", ""),
            "user_agent": event.get("user_agent", ""),
            "correlation_id": event.get("correlation_id", ""),
            "risk_level": event.get("risk_level", "info")
        }
    
    def _compute_hash(self, entry: dict) -> str:
        """计算审计条目的哈希"""
        content = json.dumps(entry, sort_keys=True, ensure_ascii=False)
        return hashlib.sha256(content.encode()).hexdigest()
    
    async def verify_chain(self, from_time: datetime = None) -> bool:
        """验证审计日志链的完整性"""
        entries = await self.storage.get_since(from_time) if from_time \
                  else await self.storage.get_all()
        
        previous_hash = ""
        for entry in entries:
            expected_hash = entry.pop("hash", "")
            entry_hash = self._compute_hash(entry)
            
            if expected_hash != entry_hash:
                logger.error(f"审计日志被篡改: {entry['audit_id']}")
                return False
            
            if entry.get("previous_hash", "") != previous_hash:
                logger.error(f"审计链断裂: {entry['audit_id']}")
                return False
            
            previous_hash = expected_hash
        
        return True
    
    async def _check_alerts(self, entry: dict):
        """实时安全告警检查"""
        # 高风险操作
        high_risk_actions = {"delete", "export", "grant_permission"}
        if entry["action"] in high_risk_actions:
            await self._send_security_alert(entry)
        
        # 异常时段操作
        hour = datetime.utcnow().hour
        if hour < 6 or hour > 22:
            await self._send_anomaly_alert(entry)
        
        # 敏感数据访问
        sensitive_resources = {"hr_salary", "legal_contract", "financial_report"}
        if entry["resource_type"] in sensitive_resources:
            await self._send_sensitive_access_alert(entry)
```

### 2.6 运营平台

**成本追踪实现：**

```python
class CostTracker:
    """多租户成本追踪"""
    
    def __init__(self, pricing_service, billing_storage):
        self.pricing = pricing_service
        self.storage = billing_storage
    
    async def track_llm_call(self, tenant_id: str, agent_id: str,
                               model: str, prompt_tokens: int,
                               completion_tokens: int, duration_ms: int):
        """记录 LLM 调用成本"""
        token_cost = self.pricing.get_token_cost(model)
        total_tokens = prompt_tokens + completion_tokens
        
        cost_entry = {
            "tenant_id": tenant_id,
            "agent_id": agent_id,
            "model": model,
            "prompt_tokens": prompt_tokens,
            "completion_tokens": completion_tokens,
            "total_tokens": total_tokens,
            "estimated_cost": round(total_tokens * token_cost, 6),
            "duration_ms": duration_ms,
            "timestamp": datetime.utcnow().isoformat()
        }
        
        await self.storage.store(cost_entry)
        
        # 实时预算检查
        await self._check_budget(tenant_id, cost_entry["estimated_cost"])
    
    async def _check_budget(self, tenant_id: str, cost: float):
        """检查租户预算"""
        monthly_usage = await self.storage.get_monthly_cost(tenant_id)
        monthly_budget = await self.pricing.get_tenant_budget(tenant_id)
        
        usage_ratio = monthly_usage / monthly_budget
        if usage_ratio > 0.9:
            await self._send_budget_warning(tenant_id, usage_ratio)
        elif usage_ratio > 1.0:
            await self._enforce_budget(tenant_id)
    
    async def get_tenant_report(self, tenant_id: str, 
                                 start_date: str,
                                 end_date: str) -> dict:
        """生成租户使用报告"""
        costs = await self.storage.query(
            tenant_id=tenant_id,
            start_date=start_date,
            end_date=end_date
        )
        
        report = {
            "tenant_id": tenant_id,
            "period": {"start": start_date, "end": end_date},
            "total_cost": round(sum(c["estimated_cost"] for c in costs), 2),
            "total_tokens": sum(c["total_tokens"] for c in costs),
            "total_calls": len(costs),
            "model_breakdown": self._breakdown_by_model(costs),
            "agent_breakdown": self._breakdown_by_agent(costs),
            "daily_trend": self._daily_trend(costs)
        }
        
        return report
```

---

## 3. 核心设计决策

### 3.1 租户隔离策略选择

| 策略 | 数据隔离 | 性能 | 运维复杂度 | 成本 | 推荐场景 |
|------|---------|------|-----------|------|---------|
| Database-per-Tenant | 最强 | 独立资源 | 高 | 高 | 金融、医疗等强合规 |
| Schema-per-Tenant | 强 | 共享实例 | 中 | 中 | 中型企业 |
| Shared Table | 弱 | 共享资源 | 低 | 低 | 标准化服务、SaaS |
| Hybrid | 可配置 | 灵活 | 高 | 中 | 大型平台 |

**企业平台通常采用 Hybrid 策略**：敏感数据使用独立数据库，普通数据使用共享表 + 行级隔离。

### 3.2 Agent 模板 vs 自定义 Agent

```
模板化 Agent                            自定义 Agent
┌────────────────────┐              ┌──────────────────────────┐
│ 优点:               │              │ 优点:                     │
│ • 开箱即用           │              │ • 完全灵活               │
│ • 经过充分测试       │              │ • 满足特殊需求            │
│ • 维护成本低         │              │ • 深度集成企业系统         │
│ • 一致的用户体验      │              │                          │
│                      │              │ 缺点:                     │
│ 缺点:                │              │ • 开发周期长              │
│ • 灵活性受限         │              │ • 需要专业技能            │
│ • 可能不满足特殊需求   │              │ • 维护成本高              │
│                      │              │ • 质量参差不齐            │
└────────────────────┘              └──────────────────────────┘

平衡策略: 80% 模板 + 20% 自定义
┌──────────────────────────────────────────────────────────────┐
│  80% 标准场景: HR FAQ, IT Helpdesk, 报销流程                  │
│     └── 使用预构建模板，租户只需配置知识库和权限                │
│                                                              │
│  20% 特殊场景: 法务合同审查, 合规报告生成, 供应链优化           │
│     └── 使用模板 + 自定义扩展，或完全自定义                    │
└──────────────────────────────────────────────────────────────┘
```

### 3.3 企业 SSO/OAuth 集成架构

```
                    ┌──────────────────┐
                    │   身份提供者(IdP) │
                    │  Azure AD / Okta │
                    │  / PingIdentity  │
                    └────────┬─────────┘
                             │ SAML/OIDC
                    ┌────────▼─────────┐
                    │  企业 Agent 平台  │
                    │                  │
                    │  ┌────────────┐  │
                    │  │ SAML SP    │  │
                    │  │ 断言消费服务 │  │
                    │  └────────────┘  │
                    │  ┌────────────┐  │
                    │  │ OIDC RP    │  │
                    │  │ 授权码流程   │  │
                    │  └────────────┘  │
                    │  ┌────────────┐  │
                    │  │ JWT 验证    │  │
                    │  │ 签名 + 过期  │  │
                    │  └────────────┘  │
                    │  ┌────────────┐  │
                    │  │ 租户映射    │  │
                    │  │ email→租户  │  │
                    │  └────────────┘  │
                    └──────────────────┘
```

### 3.4 合规感知的审计日志

企业 Agent 平台必须满足多种合规标准：

| 标准 | 要求 | 实现 |
|------|------|------|
| **GDPR** | 数据可删除、可导出 | 用户数据物理删除 + 数据可移植 API |
| **SOX** | 财务相关操作不可篡改 | 审计日志哈希链 + 写后即封 |
| **HIPAA** | PHI 数据加密 + 访问审计 | 字段级加密 + 每次访问记录 |
| **等保 2.0** | 日志留存 180 天+ | 冷热分层存储 + 合规转储 |
| **PCI-DSS** | 信用卡数据处理限制 | DLP 自动屏蔽 + tokenization |

---

## 4. 代码示例

### 4.1 租户感知的向量存储

```python
class TenantAwareVectorStore:
    """租户隔离的向量存储"""
    
    def __init__(self, embedding_service,
                 index_registry: Dict[str, any],
                 tenant_index_map: Dict[str, str]):
        self.embedder = embedding_service
        self.index_registry = index_registry  # 实际向量索引
        self.tenant_index_map = tenant_index_map  # 租户 -> 索引名称
    
    def _get_tenant_index(self, tenant_id: str = None):
        """获取租户专属的向量索引"""
        tid = tenant_id or TenantContext.get_tenant_id()
        index_name = self.tenant_index_map.get(tid, f"index_default")
        return self.index_registry[index_name]
    
    async def add_document(self, document: str, metadata: dict,
                            tenant_id: str = None):
        """添加文档到租户隔离的索引"""
        tid = tenant_id or TenantContext.get_tenant_id()
        
        # 确保 metadata 中包含租户标识
        metadata["tenant_id"] = tid
        metadata["indexed_at"] = datetime.utcnow().isoformat()
        
        # 生成向量
        embedding = await self.embedder.embed(document)
        
        # 写入租户专属索引
        index = self._get_tenant_index(tid)
        await index.add_vector(
            vector=embedding,
            payload={"text": document, "metadata": metadata}
        )
    
    async def search(self, query: str, top_k: int = 5,
                      filter_conditions: dict = None,
                      tenant_id: str = None) -> List[dict]:
        """在租户隔离的索引中搜索"""
        tid = tenant_id or TenantContext.get_tenant_id()
        
        query_vector = await self.embedder.embed(query)
        index = self._get_tenant_index(tid)
        
        # 自动添加租户过滤条件
        filters = {"tenant_id": tid}
        if filter_conditions:
            filters.update(filter_conditions)
        
        results = await index.search(
            vector=query_vector,
            top_k=top_k,
            filter=filters
        )
        
        return [
            {
                "score": r.score,
                "text": r.payload["text"],
                "metadata": r.payload["metadata"]
            }
            for r in results
        ]
    
    async def delete_tenant_data(self, tenant_id: str):
        """删除租户的所有数据（GDPR 合规）"""
        index = self._get_tenant_index(tenant_id)
        await index.delete_by_filter({"tenant_id": tenant_id})
        logger.info(f"已删除租户 {tenant_id} 的所有向量数据")
```

### 4.2 审批工作流集成

```python
class EnterpriseApprovalWorkflow:
    """企业审批工作流 - 与 ServiceNow / JIRA 集成"""
    
    def __init__(self, connector_registry, notification_service):
        self.connectors = connector_registry
        self.notifier = notification_service
    
    async def create_approval_request(self, 
                                       request_type: str,  # "agent_publish", "knowledge_upload", "tool_access"
                                       requester: str,
                                       tenant_id: str,
                                       details: dict) -> str:
        """创建审批请求"""
        
        # 确定审批流程
        workflow = await self._resolve_workflow(request_type, details)
        
        # 创建审批票据（在 ServiceNow 或 JIRA 中）
        ticket = await self._create_ticket(
            workflow["system"],
            request_type, requester, tenant_id, details
        )
        
        # 通知第一级审批人
        first_approver = workflow["levels"][0]["approvers"]
        await self.notifier.notify_approval_required(
            approvers=first_approver,
            ticket_id=ticket["id"],
            request_type=request_type,
            summary=details.get("summary", ""),
            deadline=workflow["levels"][0].get("deadline_hours", 24)
        )
        
        return ticket["id"]
    
    async def _resolve_workflow(self, request_type: str, 
                                  details: dict) -> dict:
        """根据请求类型和详情确定审批流程"""
        workflows = {
            "agent_publish": {
                "system": "servicenow",
                "levels": [
                    {"approvers": ["team_lead"], "deadline_hours": 24},
                    {"approvers": ["dept_head"], "deadline_hours": 48},
                ]
            },
            "knowledge_upload": {
                "system": "jira",
                "levels": [
                    {"approvers": ["content_reviewer"], "deadline_hours": 12},
                ]
            },
            "tool_access": {
                "system": "servicenow",
                "levels": [
                    {"approvers": ["system_owner"], "deadline_hours": 24},
                ]
            }
        }
        
        base = workflows.get(request_type, workflows["agent_publish"])
        
        # 根据金额/风险级别动态增加审批层级
        amount = details.get("amount", 0)
        if amount > 1000000:
            base["levels"].append({
                "approvers": ["vp_finance"],
                "deadline_hours": 72
            })
        
        return base
    
    async def handle_approval_response(self, 
                                         ticket_id: str,
                                         approved: bool,
                                         approver: str,
                                         comment: str = ""):
        """处理审批响应"""
        if approved:
            await self._execute_approved_action(ticket_id)
        else:
            await self._handle_rejection(ticket_id, approver, comment)
    
    async def _execute_approved_action(self, ticket_id: str):
        """审批通过后执行授权操作"""
        ticket = await self._get_ticket(ticket_id)
        
        if ticket["request_type"] == "agent_publish":
            # 发布 Agent
            await agent_manager.deploy_agent(ticket["details"]["agent_id"])
        elif ticket["request_type"] == "knowledge_upload":
            # 发布知识库文档
            await knowledge_manager.publish_document(
                ticket["details"]["document_id"]
            )
        elif ticket["request_type"] == "tool_access":
            # 授权工具访问
            await tool_manager.grant_access(
                tenant_id=ticket["tenant_id"],
                tool_id=ticket["details"]["tool_id"],
                user_id=ticket["details"]["user_id"]
            )
```

---

## 5. 与 SaaS Agent 的区别

| 维度 | SaaS Agent (面向个人) | 企业 Agent 平台 |
|------|---------------------|----------------|
| **部署方式** | 公有云多租户 | 私有化 / 混合云 / VPC |
| **身份认证** | Email + Password | SSO + SAML/OIDC + MFA |
| **数据存储** | 共享存储 | 租户隔离 + 数据主权 |
| **安全合规** | 基本加密 | 等保 / GDPR / SOC2 / HIPAA |
| **定制能力** | 无 / 有限 | 模板 + 自定义 + API 扩展 |
| **集成能力** | 公开 API | 企业连接器（SAP, Salesforce 等） |
| **SLA** | 99.5% | 99.9% ~ 99.99% |
| **支持模式** | 在线文档 | 专属 CSM + 技术客户经理 |
| **计费模式** | 按用户/月 | 按用量 + 基础费用 + 定制费用 |
| **审核要求** | 无 | 全量操作审计 + 合规报告 |
| **网络部署** | 公网 | 私有网络 / 专线 / 空气间隙 |

---

## 6. 关键挑战

### 6.1 企业数据隐私

```
数据隐私防护层级:

L1: 传输层
  └─ 所有 API 通信强制 TLS 1.3 + 证书固定

L2: 存储层
  └─ AES-256 静态加密 + HSM 密钥管理 + 字段级加密

L3: 处理层
  └─ LLM 调用前脱敏 → 调用后还原 → 日志不落盘
  └─ 敏感字段自动检测 (正则 + NER 模型)

L4: 隔离层
  └─ 租户级 VPC / Kubernetes Namespace
  └─ 数据永不跨租户边界

L5: 审计层
  └─ 每个数据访问操作记录审计日志
  └─ 数据库行级安全策略 (RLS)
```

### 6.2 集成复杂度

企业通常有 100+ 个内部系统，每个系统有不同的 API 风格、认证方式和数据模型。关键策略：

1. **连接器标准化**：统一认证、错误处理、重试、审计接口
2. **适配器模式**：每个系统一个适配器，注册到连接器注册表
3. **Mock 系统**：开发和测试环境使用模拟系统，避免影响生产
4. **连接健康监控**：每个连接器的实时健康检查 + 自动切换

### 6.3 SLA 保证

```
企业 SLA 框架:

响应时间:
  99% 的请求 < 3s (P99)
  99.9% 的请求 < 10s (P99.9)

可用性:
  平台整体: 99.9% (月宕机 < 43min)
  关键路径: 99.99% (月宕机 < 4.3min)

吞吐量:
  单租户: 100 RPM (每分钟请求数)
  平台峰值: 100,000 RPM

多活部署:
  主 - 备 跨 AZ 部署
  RTO < 5min, RPO < 1min

容量规划:
  提前 2 周预测 + 自动扩缩容
  每个租户独立资源池
```

### 6.4 定制化 vs 标准化

这是企业平台最棘手的平衡问题：完全标准化会失去客户，过度定制化会拖垮工程团队。

```
标准化                         定制化
  ◄────────────────────────────►
     │          │          │
     │    80%   │   15%    │  5%
     │  内置功能  │ 配置化   │ 代码扩展
     │          │          │
     ▼          ▼          ▼
  ┌────────┐ ┌────────┐ ┌────────┐
  │ 零配置  │ │ 可选配置 │ │ 插件/API │
  │ 即开即用 │ │ 通过 UI  │ │ 自定义   │
  │         │ │ 调整行为 │ │ 开发     │
  └────────┘ └────────┘ └────────┘

管理策略:
  - 80% 标准化: 平台团队负责，所有租户收益
  - 15% 配置化: 通过管理控制台配置，无需开发
  - 5% 定制化: 作为增值服务，收取额外费用
```

---

## 7. 真实案例参考

### 7.1 Salesforce Einstein

Salesforce Einstein 是嵌入在 Salesforce CRM 中的 AI Agent 平台。

```
Salesforce Einstein 架构要点:
┌─────────────────────────────────────────────────────────┐
│ 多租户:                                                  │
│ • 每个 Salesforce Org 是一个租户                          │
│ • 数据完全隔离，模型训练不跨租户                           │
│                                                         │
│ 集成方式:                                                │
│ • 原生嵌入 Salesforce 对象 (Lead, Opportunity, Case)      │
│ • 通过 MuleSoft 连接外部系统                              │
│ • Flow Builder 可视化工作流                               │
│                                                         │
│ Agent 能力:                                              │
│ • Einstein Bot: 对话式 AI 客服                           │
│ • Einstein Prediction: 销售预测、评分                     │
│ • Einstein Discovery: 自动发现数据模式                     │
│                                                         │
│ 安全合规:                                                │
│ • Shield Platform Encryption                             │
│ • Field-Level Security                                   │
│ • Event Monitoring                                       │
└─────────────────────────────────────────────────────────┘
```

### 7.2 ServiceNow AI

ServiceNow 将其 AI 能力嵌入 ITSM（IT 服务管理）和 ITOM（IT 运维管理）流程中。

```
ServiceNow AI 关键特性:
┌─────────────────────────────────────────────────────────┐
│ 流程原生 AI:                                             │
│ • 事件管理: AI 自动分类、分配、优先级判定                    │
│ • 变更管理: AI 评估变更风险、推荐审批人                     │
│ • 知识管理: AI 自动生成知识库文章                            │
│                                                         │
│ 虚拟 Agent:                                              │
│ • 多语言对话式 Self-Service                               │
│ • 与 ITSM 流程深度集成                                    │
│ • 可配置的对话流 + 上下文感知                              │
│                                                         │
│ Now Platform:                                            │
│ • Low-Code 开发平台                                       │
│ • 集成 Hub: 100+ 预建连接器                              │
│ • 统一的权限和审计模型                                    │
└─────────────────────────────────────────────────────────┘
```

### 7.3 Microsoft Copilot Studio

Microsoft 的企业 AI Agent 平台，构建于 Microsoft 365 生态之上。

```
Microsoft Copilot Studio 架构:
┌─────────────────────────────────────────────────────────┐
│ 基础能力:                                                │
│ • 自定义 Copilot: 基于 GPT-4 构建自定义 AI Agent         │
│ • 知识源: SharePoint, OneDrive, 自定义数据, 第三方       │
│ • 插件: 1000+ Microsoft 连接器 + Power Automate         │
│                                                         │
│ 企业合规:                                                │
│ • Microsoft Purview 集成                                 │
│ • 数据主权: 区域数据驻留                                  │
│ • 合规认证: ISO 27001, SOC 2, HIPAA, GDPR              │
│                                                         │
│ 管理功能:                                                │
│ • Admin Center: 集中管理 Agent 发布                      │
│ • Analytics: 使用分析 + 满意度评分                        │
│ • DLP 策略: 防止敏感数据泄露                              │
│                                                         │
│ 部署方式:                                                │
│ • SaaS (M365 内置)                                       │
│ • 专属数据中心 (用于数据主权要求)                          │
└─────────────────────────────────────────────────────────┘
```

---

## 8. 架构演进路线

### 8.1 阶段建设路线

```
阶段 1: MVP 验证（1-3 个月）
┌────────────────────────────────────────────────────┐
│ • 单租户支持                                         │
│ • 1-2 个 Agent 模板                                  │
│ • 基本 RAG + 对话                                     │
│ • 简单 SSO 集成                                      │
│ • 基础审计日志                                        │
└────────────────────────────────────────────────────┘

阶段 2: 多租户平台（3-6 个月）
┌────────────────────────────────────────────────────┐
│ • 多租户隔离 (Schema-per-Tenant)                    │
│ • Agent 目录与商店                                   │
│ • 企业连接器 SDK                                    │
│ • 租户管理控制台                                     │
│ • 使用统计与计费                                     │
└────────────────────────────────────────────────────┘

阶段 3: 企业级能力（6-12 个月）
┌────────────────────────────────────────────────────┐
│ • 高级租户隔离 (Hybrid)                             │
│ • 审批工作流集成                                     │
│ • DLP + 敏感数据检测                                 │
│ • 灰度发布 + A/B 测试                                │
│ • 自定义 Agent 插件框架                              │
│ • SLA 监控 + 自动故障转移                             │
└────────────────────────────────────────────────────┘

阶段 4: 智能平台（12-24 个月）
┌────────────────────────────────────────────────────┐
│ • Auto Agent: 自动生成 Agent 配置                   │
│ • 联邦查询: 跨租户数据匿名聚合分析                      │
│ • 自适应模型路由: 按查询复杂度选择模型                   │
│ • Self-Healing: 自动检测和修复 Agent 问题              │
│ • 市场: Agent 模板 / 技能交易市场                      │
└────────────────────────────────────────────────────┘
```

### 8.2 构建还是购买

| 场景 | 推荐 |
|------|------|
| 少于 3 个部门使用 | 购买 SaaS Agent 产品 |
| 3-10 个部门，标准化需求 | 使用开源 + 定制开发 |
| 10+ 部门，深度集成需求 | 自建企业 Agent 平台 |
| 强合规行业（金融/医疗） | 私有化部署 + 自定义开发 |

---

## 9. 总结

企业 Agent 平台架构的核心挑战不在于 AI 能力本身，而在于如何将 AI 能力安全、可靠、合规地嵌入到复杂的企业 IT 生态中。设计时需要重点关注：

1. **多租户隔离是基础**：选择适合业务场景的隔离策略，平衡安全、性能和成本
2. **集成能力是关键**：企业系统的价值取决于能连接多少内部系统
3. **安全合规是底线**：审计日志、DLP、审批流是交付企业的必要条件
4. **管理平面是核心**：让非技术用户也能管理和监控他们的 Agent
5. **标准化与定制化的平衡**：保证平台可维护性的同时满足不同部门的个性化需求

## 参考与扩展阅读

- [Salesforce Einstein](https://www.salesforce.com/products/einstein/) - CRM AI 平台
- [ServiceNow AI](https://www.servicenow.com/products/ai/) - ITSM AI 平台
- [Microsoft Copilot Studio](https://www.microsoft.com/en-us/microsoft-copilot/microsoft-copilot-studio) - 自定义 Copilot 构建器
- [AWS SaaS Architecture Fundamentals](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/) - 多租户最佳实践
- [12-Factor App](https://12factor.net/) - SaaS 应用设计原则（多租户部分）
- **OWASP ASVS** - 企业应用安全验证标准
