# 14.2.4 企业安全 — Enterprise Security

> **核心思想**：企业级 Agent 系统面临的安全威胁远超传统 Web 应用——Agent 的自主性、工具调用能力和 LLM 的非确定性引入了全新的攻击面。企业安全架构需要在**传输安全、存储安全、运行时安全、供应链安全和业务连续性**五个层面建立纵深防御体系。

---

## 1. 基本原理

### 1.1 Agent 系统的安全挑战

Agent 系统与传统企业系统的安全差异：

```
传统企业系统:                      Agent 系统:
─────────────                      ─────────────
固定 API 端点                    动态工具调用（不可预知）
确定的数据流                      Agent 自主决定数据流向
用户直接操作                     Agent 代理用户操作
攻击面可枚举                     攻击面动态变化
输入可验证的格式                 自然语言输入（不可枚举）
```

Agent 引入了**三类特有的安全风险**：

| 风险类型 | 描述 | 严重程度 |
|----------|------|----------|
| **间接提示注入** | Agent 读取的文档/网页内容中包含恶意指令 | 🔴 极高 |
| **工具劫持** | 攻击者诱导 Agent 调用未授权工具或危险参数 | 🔴 极高 |
| **数据泄露** | Agent 将敏感信息写入外部系统或返回给未授权用户 | 🟠 高 |
| **权限提升** | Agent 的自主操作链被用于绕过访问控制 | 🟠 高 |
| **供应链攻击** | Agent 使用的模型/工具/库被植入后门 | 🟠 高 |
| **拒绝服务** | 通过复杂提示词耗尽 Agent 的计算/Token 资源 | 🟡 中 |

### 1.2 纵深防御架构

```
┌─────────────────────────────────────────────────────────────┐
│                  纵深防御 (Defense in Depth)                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  层 1: 传输安全                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ TLS 1.3 / mTLS / 网络策略 / API 网关 / WAF           │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  层 2: 认证与授权                                           │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ SSO / MFA / RBAC / ABAC / 最小权限 / 工具级授权       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  层 3: 输入安全                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Prompt 注入检测 / 输入清洗 / 参数化 Prompt / 指令隔离    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  层 4: 运行时安全                                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 工具沙箱 / 代码隔离 / 执行审计 / 异常行为检测            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  层 5: 数据安全                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 静态加密 / 传输加密 / 脱敏 / DLP / 密钥管理            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  层 6: 输出安全                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 输出护栏 / 内容过滤 / 格式验证 / PII 检测               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  层 7: 监控与响应                                           │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 安全事件监控 / 告警 / 自动响应 / 事故响应流程            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 传输安全

### 2.1 网络层

```python
from dataclasses import dataclass

@dataclass
class NetworkSecurityConfig:
    """Agent 系统网络安全配置"""
    
    # TLS 配置
    tls_min_version: str = "1.3"
    tls_cipher_suites: list[str] = None
    
    # mTLS 配置
    mtls_enabled: bool = True
    mtls_ca_path: str = "/etc/certs/ca.pem"
    
    # 网络策略
    egress_rules: list[dict] = None
    ingress_rules: list[dict] = None
    
    # API 网关
    rate_limit_per_tenant: int = 1000  # RPM
    waf_enabled: bool = True
    ip_whitelist: list[str] = None
    
    def validate(self) -> bool:
        """验证安全配置的完整性"""
        if self.tls_min_version < "1.3":
            raise SecurityConfigError("TLS 1.3 minimum required")
        if self.mtls_enabled and not self.mtls_ca_path:
            raise SecurityConfigError("mTLS enabled but no CA path")
        return True

class NetworkSecurityManager:
    """Agent 系统网络安全管理"""
    
    async def validate_tool_endpoint(
        self, endpoint_url: str
    ) -> tuple[bool, str]:
        """验证工具端点的安全性"""
        
        parsed = urlparse(endpoint_url)
        
        # 1. 仅允许 HTTPS
        if parsed.scheme != "https":
            return False, "Only HTTPS allowed for tool endpoints"
        
        # 2. 检查是否在允许的域名列表中
        if not self._is_allowed_domain(parsed.hostname):
            return False, f"Domain {parsed.hostname} not in allowlist"
        
        # 3. 检查 IP 是否在允许范围
        resolved_ips = await self._resolve_ips(parsed.hostname)
        if not self._is_allowed_ip_range(resolved_ips):
            return False, f"IP {resolved_ips} not in allowed range"
        
        return True, "ok"
    
    def _is_allowed_domain(self, hostname: str) -> bool:
        """检查域名是否在白名单中"""
        allowed_domains = [
            "*.company-internal.com",
            "api.trusted-partner.com",
            "*.gov.cn",  # 政府服务
        ]
        return any(
            fnmatch(hostname, pattern)
            for pattern in allowed_domains
        )
```

### 2.2 Agent 特有传输风险

```
Agent 传输安全场景:

┌──── 用户 ────┐     TLS     ┌── Agent 平台 ──┐     TLS     ┌── LLM API ──┐
│  浏览器/App   │═══════════►│   API 网关      │═══════════►│  推理端点     │
└──────────────┘            └───────┬─────────┘            └─────────────┘
                                    │
                          ┌─────────┼─────────┐
                          │         │         │
                          ▼         ▼         ▼
                    ┌────────┐ ┌────────┐ ┌────────┐
                    │ 工具 A  │ │ 工具 B  │ │ 知识库  │
                    │ HTTPS   │ │ gRPC   │ │ HTTPS   │
                    │ mTLS    │ │ mTLS   │ │ mTLS    │
                    └────────┘ └────────┘ └────────┘

关键点:
  1. 用户 ↔ Agent 平台: TLS 1.3
  2. Agent 平台 ↔ LLM: TLS + API Key (可能需 BAA)
  3. Agent 平台 ↔ 工具: TLS + mTLS + Token Exchange
  4. Agent 平台 ↔ 知识库: 内部网络 + mTLS
```

---

## 3. 运行时安全

### 3.1 工具执行沙箱

Agent 调用工具时，需要确保工具执行在安全的沙箱环境中：

```python
import subprocess
import tempfile
import resource
from typing import Optional

class ToolSandbox:
    """工具执行沙箱——限制 Agent 调用的工具的执行权限"""
    
    def __init__(self, config: SandboxConfig):
        self.config = config
    
    async def execute_in_sandbox(
        self,
        tool_type: str,
        code: Optional[str] = None,
        command: Optional[list[str]] = None,
    ) -> SandboxResult:
        """在沙箱中执行工具"""
        
        if tool_type == "python":
            return await self._execute_python(code)
        elif tool_type == "shell":
            return await self._execute_shell(command)
        elif tool_type == "browser":
            return await self._execute_browser(command)
        else:
            raise ValueError(f"Unknown tool type: {tool_type}")
    
    async def _execute_python(self, code: str) -> SandboxResult:
        """在隔离的 Python 环境中执行"""
        
        # 1. 安全检查：不允许 import 危险模块
        blocked_modules = [
            "os", "subprocess", "shutil", "socket",
            "ctypes", "sys", "inspect",
        ]
        for mod in blocked_modules:
            if re.search(rf'\bimport\s+{mod}\b', code):
                return SandboxResult(
                    success=False,
                    error=f"Module '{mod}' is blocked",
                    stdout="",
                )
        
        # 2. 在受限的执行器中运行
        restricted_globals = {
            "__builtins__": {
                "print": print,
                "len": len,
                "range": range,
                "int": int,
                "float": float,
                "str": str,
                "list": list,
                "dict": dict,
                "tuple": tuple,
                "set": set,
                "bool": bool,
                "enumerate": enumerate,
                "zip": zip,
                "map": map,
                "filter": filter,
                "sum": sum,
                "min": min,
                "max": max,
                "abs": abs,
                "round": round,
                "sorted": sorted,
                "reversed": reversed,
                "any": any,
                "all": all,
                "True": True,
                "False": False,
                "None": None,
                "Exception": Exception,
                "ValueError": ValueError,
                "TypeError": TypeError,
                "KeyError": KeyError,
                "IndexError": IndexError,
            }
        }
        
        try:
            # 3. 设置资源限制
            resource.setrlimit(resource.RLIMIT_CPU, (5, 5))  # CPU 5 秒
            resource.setrlimit(resource.RLIMIT_AS, (256 * 1024 * 1024, 256 * 1024 * 1024))  # 256MB 内存
            
            # 4. 编译并执行
            compiled = compile(code, "<sandbox>", "exec")
            exec(compiled, restricted_globals)
            
            return SandboxResult(success=True, error="", stdout="")
            
        except Exception as e:
            return SandboxResult(success=False, error=str(e), stdout="")
    
    async def _execute_shell(self, command: list[str]) -> SandboxResult:
        """在隔离 shell 中执行命令"""
        
        # 仅允许白名单命令
        allowed_commands = {
            "ls", "cat", "head", "tail", "wc",
            "grep", "find", "sort", "uniq",
            "python3", "node",
        }
        if command[0] not in allowed_commands:
            return SandboxResult(
                success=False,
                error=f"Command '{command[0]}' not allowed",
                stdout="",
            )
        
        # 超时执行
        try:
            result = subprocess.run(
                command,
                capture_output=True,
                text=True,
                timeout=30,
                cwd=self.config.temp_dir,
            )
            return SandboxResult(
                success=result.returncode == 0,
                error=result.stderr[:1000],  # 限制错误输出长度
                stdout=result.stdout[:10000],  # 限制输出长度
            )
        except subprocess.TimeoutExpired:
            return SandboxResult(success=False, error="Command timed out", stdout="")
```

### 3.2 Agent 异常行为检测

```python
class AnomalyDetection:
    """Agent 行为异常检测"""
    
    def __init__(self):
        self.baselines = self._load_baselines()
    
    async def analyze_action(
        self, action: AgentAction, context: AgentContext
    ) -> AnomalyScore:
        """分析 Agent 行动是否异常"""
        
        anomalies = []
        
        # 1. 工具调用频率异常
        tool_freq = self._get_tool_frequency(
            context.session_id, action.tool_name
        )
        baseline_freq = self.baselines["tool_frequency"].get(
            action.tool_name, 1
        )
        if tool_freq > baseline_freq * 3:
            anomalies.append(Anomaly(
                type="tool_frequency_spike",
                severity="medium",
                detail=f"Tool {action.tool_name} called {tool_freq}x, "
                       f"baseline {baseline_freq}x",
            ))
        
        # 2. 敏感参数检测
        sensitive_params = self._detect_sensitive_params(action.params)
        if sensitive_params:
            anomalies.append(Anomaly(
                type="sensitive_data_in_tool_call",
                severity="high",
                detail=f"Sensitive data in tool params: {sensitive_params}",
            ))
        
        # 3. 异常工具组合（如 读数据库 + 发邮件 的组合）
        tool_sequence = self._get_tool_sequence(context.session_id)
        dangerous_pairs = [
            ("sql_query", "email_send"),
            ("read_file", "slack_message"),
            ("list_bucket", "file_upload"),
        ]
        for tool_a, tool_b in dangerous_pairs:
            if (action.tool_name == tool_b and tool_a in tool_sequence):
                # Agent 读取数据库后立即发邮件 → 可能的数据泄露
                anomalies.append(Anomaly(
                    type="dangerous_tool_sequence",
                    severity="high",
                    detail=f"Dangerous sequence: {tool_a} → {tool_b}",
                ))
        
        # 4. Token 消耗异常
        if context.token_usage > 50000:
            anomalies.append(Anomaly(
                type="excessive_token_usage",
                severity="medium",
                detail=f"Session token usage: {context.token_usage}",
            ))
        
        return AnomalyScore(
            score=len(anomalies),
            anomalies=anomalies,
            requires_intervention=any(
                a.severity == "high" for a in anomalies
            ),
        )
    
    async def intervene(self, action: AgentAction, score: AnomalyScore):
        """异常行为干预"""
        
        if score.requires_intervention:
            # 高风险：阻止当前操作 + 通知安全团队
            await self._block_action(action)
            await self._notify_security_team(action, score)
        elif score.score >= 2:
            # 中风险：需要人工确认
            await self._require_human_approval(action)
        else:
            # 低风险：记录日志
            await self._log_anomaly(action, score)
```

### 3.3 文件访问控制

```python
class FileAccessController:
    """Agent 文件系统访问控制"""
    
    def __init__(self):
        # 允许 Agent 访问的路径（白名单）
        self.allowed_paths = {
            "read": [
                "/data/knowledge_base/",
                "/data/templates/",
                "/data/reports/",
            ],
            "write": [
                "/data/output/",
                "/data/user_uploads/",
            ],
            "execute": [],  # 不允许执行任何文件
        }
    
    def check_path_access(
        self, path: str, operation: str
    ) -> bool:
        """检查 Agent 是否有路径访问权限"""
        
        normalized = os.path.normpath(path)
        
        # 1. 路径规范化，防止 ../ 遍历
        allowed_prefixes = self.allowed_paths.get(operation, [])
        
        # 2. 检查是否在允许路径下
        for prefix in allowed_prefixes:
            norm_prefix = os.path.normpath(prefix)
            if normalized.startswith(norm_prefix):
                return True
        
        return False
    
    def sanitize_path(self, user_provided_path: str) -> str:
        """清洗用户提供的路径（防止路径遍历攻击）"""
        
        # 1. 移除空字节
        sanitized = user_provided_path.replace('\0', '')
        
        # 2. 阻止路径遍历
        sanitized = sanitized.replace('..', '')
        
        # 3. 只允许安全字符
        sanitized = re.sub(r'[^\w/\-. ]', '', sanitized)
        
        return sanitized
```

---

## 4. 数据安全

### 4.1 加密方案

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC

class DataEncryption:
    """Agent 数据加密"""
    
    def __init__(self, master_key: bytes):
        self.fernet = Fernet(master_key)
    
    def encrypt_conversation(self, conversation: dict) -> dict:
        """加密对话内容（静态加密）"""
        serialized = json.dumps(conversation, ensure_ascii=False)
        encrypted = self.fernet.encrypt(serialized.encode())
        return {"encrypted_data": encrypted.decode(), "algorithm": "Fernet"}
    
    def decrypt_conversation(self, encrypted: dict) -> dict:
        """解密对话内容"""
        decrypted = self.fernet.decrypt(
            encrypted["encrypted_data"].encode()
        )
        return json.loads(decrypted.decode())
    
    @staticmethod
    def encrypt_tool_credentials(tool_config: dict) -> dict:
        """加密工具凭据"""
        safe_config = tool_config.copy()
        
        for key in ["api_key", "password", "token", "secret"]:
            if key in safe_config:
                if isinstance(safe_config[key], str):
                    safe_config[key] = f"encrypted:{safe_config[key][:8]}..."
        
        return safe_config
```

### 4.2 密钥管理

```python
class KeyManagement:
    """密钥管理体系"""
    
    def __init__(self, vault_address: str, vault_token: str):
        self.vault_client = hvac.Client(
            url=vault_address,
            token=vault_token,
        )
    
    def get_llm_api_key(self, tenant_id: str) -> str:
        """获取租户的 LLM API Key（从 Vault 读取）"""
        secret = self.vault_client.secrets.kv.v2.read_secret_version(
            path=f"tenants/{tenant_id}/llm_api_key",
        )
        return secret["data"]["data"]["api_key"]
    
    def rotate_key(self, path: str):
        """定期轮换密钥"""
        new_key = Fernet.generate_key()
        self.vault_client.secrets.kv.v2.create_or_update_secret(
            path=path,
            data={"key": new_key.decode()},
        )
```

---

## 5. DLP（数据防泄漏）

### 5.1 Agent 特有 DLP 规则

```python
class AgentDLP:
    """Agent 数据防泄漏"""
    
    DLP_RULES = [
        {
            "name": "no_credit_card_in_output",
            "pattern": r'\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b',
            "action": "block",
            "message": "Agent 输出包含信用卡号",
        },
        {
            "name": "no_api_key_in_tool_call",
            "pattern": r'(sk-[A-Za-z0-9]{20,}|api_key[=:]["\'][A-Za-z0-9]{16,})',
            "action": "mask",
            "message": "工具调用参数包含 API Key",
        },
        {
            "name": "no_internal_doc_forwarding",
            "pattern": r'机密|内部文件|CONFIDENTIAL',
            "context_required": True,
            "action": "alert",
            "message": "高风险内容被外部工具调用",
        },
        {
            "name": "no_batch_data_export",
            "pattern": None,  # 特殊规则：检测批量数据操作
            "action": "block",
            "condition": "batch_export",
            "message": "批量导出数据需要审批",
        },
    ]
    
    async def inspect_output(
        self, text: str, context: AgentContext
    ) -> DLPResult:
        """检查 Agent 输出是否包含敏感数据"""
        
        for rule in self.DLP_RULES:
            if rule.get("pattern"):
                matches = re.finditer(rule["pattern"], text)
                for match in matches:
                    return DLPResult(
                        triggered=True,
                        rule=rule["name"],
                        action=rule["action"],
                        message=rule["message"],
                        matched_text=match.group(),
                    )
            
            if rule.get("condition") == "batch_export":
                if await self._is_batch_export(context):
                    return DLPResult(
                        triggered=True,
                        rule=rule["name"],
                        action=rule["action"],
                        message=rule["message"],
                    )
        
        return DLPResult(triggered=False)
    
    async def _is_batch_export(self, context: AgentContext) -> bool:
        """检测是否在批量导出数据"""
        recent_actions = context.get_recent_actions(10)
        read_ops = sum(
            1 for a in recent_actions
            if a.type in ("sql_query", "read_file", "list_objects")
        )
        write_ops = sum(
            1 for a in recent_actions
            if a.type in ("email_send", "file_upload", "slack_message")
        )
        # 批量读取 + 批量写入 = 批量导出行为
        return read_ops >= 5 and write_ops >= 1
```

### 5.2 DLP 响应策略

```
DLP 检测到违规时的响应:

规则 "阻止" → 立即中断 Agent 操作
            → 记录完整上下文到安全日志
            → 通知安全管理员

规则 "告警" → 允许操作继续
            → 记录告警到安全日志
            → 如果频率超过阈值则升级为阻止

规则 "脱敏" → 自动替换敏感内容为 ***
            → 记录原始内容到审计日志
            → 继续执行

规则 "审批" → 暂停 Agent 操作
            → 发送审批请求
            → 审批通过则继续，拒绝则回滚
```

---

## 6. 供应链安全

### 6.1 模型供应链

```python
class ModelSupplyChainSecurity:
    """模型供应链安全管理"""
    
    REQUIRED_CHECKS = [
        "model_provenance",       # 模型来源可追溯
        "weight_integrity",       # 权重完整性校验
        "vulnerability_scan",     # 漏洞扫描
        "license_compliance",     # 许可证合规
        "red_team_evaluation",    # 红队评估报告
    ]
    
    async def validate_model(
        self, model_id: str, source: str
    ) -> ModelSecurityReport:
        """验证模型是否安全可用"""
        
        report = ModelSecurityReport(model_id=model_id)
        
        # 1. 来源验证
        if source not in self._trusted_sources():
            report.add_issue("untrusted_source", "Model from untrusted source")
        
        # 2. 完整性校验
        expected_hash = self._get_expected_hash(model_id)
        actual_hash = await self._compute_hash(model_id)
        if expected_hash and actual_hash != expected_hash:
            report.add_issue("integrity_mismatch", "Model hash mismatch")
        
        # 3. 漏洞扫描（如果有 SBOM）
        sbom = await self._get_sbom(model_id)
        if sbom:
            vulnerabilities = await self._scan_vulnerabilities(sbom)
            report.vulnerabilities = vulnerabilities
        
        return report
```

### 6.2 工具供应链

```python
class ToolSecurityRegistry:
    """工具安全注册——确保只有经过安全审核的工具才能注册"""
    
    def __init__(self):
        self.registered_tools: dict[str, ToolSecurityInfo] = {}
    
    async def register_tool(
        self,
        tool_def: ToolDefinition,
        security_review: SecurityReview,
    ) -> bool:
        """注册工具（需通过安全审核）"""
        
        # 安全审核清单
        checks = {
            "authentication": bool(tool_def.auth_method),
            "output_sanitization": self._has_output_sanitization(tool_def),
            "rate_limits": tool_def.rate_limit is not None,
            "timeout_configured": tool_def.timeout_seconds is not None,
            "input_validation": self._has_input_validation(tool_def),
            "audit_logging": security_review.audit_logging_reviewed,
            "data_classification": security_review.data_classification is not None,
        }
        
        failed_checks = [k for k, v in checks.items() if not v]
        
        if failed_checks:
            raise SecurityReviewError(
                f"Tool '{tool_def.name}' failed security checks: {failed_checks}"
            )
        
        self.registered_tools[tool_def.name] = ToolSecurityInfo(
            tool_def=tool_def,
            security_review=security_review,
            registered_at=datetime.utcnow(),
        )
        
        return True
```

---

## 7. 业务连续性与灾难恢复 (BCP/DR)

```python
class AgentBCP:
    """Agent 系统业务连续性计划"""
    
    DR_STRATEGIES = {
        "llm_failover": {
            "trigger": "primary_llm_unavailable",
            "action": "switch_to_fallback_provider",
            "rto": "30s",
            "impact": "可能使用不同模型，结果略差异",
        },
        "tool_degradation": {
            "trigger": "tool_timeout > 3x baseline",
            "action": "disable_tool_and_notify",
            "rto": "10s",
            "impact": "Agent 能力降级，但核心功能可用",
        },
        "region_failover": {
            "trigger": "primary_region_unavailable",
            "action": "route_to_dr_region",
            "rto": "5min",
            "impact": "延迟增加，数据可能滞后",
        },
        "knowledge_base_fallback": {
            "trigger": "vector_db_unavailable",
            "action": "use_cached_index",
            "rto": "1s",
            "impact": "检索可能不是最新的",
        },
    }
    
    async def execute_dr_plan(
        self, scenario: str, context: AgentContext
    ):
        """执行灾难恢复计划"""
        
        strategy = self.DR_STRATEGIES.get(scenario)
        if not strategy:
            raise ValueError(f"Unknown DR scenario: {scenario}")
        
        # 1. 记录灾难事件
        await self.audit_log.log_dr_event(scenario, context)
        
        # 2. 执行切换
        start_time = time.time()
        if scenario == "llm_failover":
            await self._failover_llm(context)
        elif scenario == "region_failover":
            await self._failover_region(context)
        
        # 3. 验证恢复
        recovery_time = time.time() - start_time
        if recovery_time > self._parse_rto(strategy["rto"]):
            await self._escalate(f"RTO exceeded for {scenario}")
        
        # 4. 通知相关方
        await self._notify_stakeholders(scenario, recovery_time)
```

---

## 8. 安全监控与响应

```python
class SecurityEvent:
    """安全事件"""
    type: str          # tool_anomaly / injection_attempt / data_leak
    severity: str      # critical / high / medium / low
    tenant_id: str
    user_id: str
    agent_action: dict
    timestamp: datetime
    context: dict

class SecurityIncidentResponse:
    """安全事件响应"""
    
    RESPONSE_PLAYBOOKS = {
        "injection_attempt": [
            "block_current_request",
            "invalidate_session",
            "notify_security_team",
            "review_recent_actions",
            "update_waf_rules",
        ],
        "data_leak_detected": [
            "stop_agent_execution",
            "revoke_tool_tokens",
            "capture_full_context",
            "notify_dpo",
            "assess_impact",
        ],
        "tool_abuse": [
            "disable_tool",
            "review_recent_calls",
            "update_permissions",
            "notify_tool_owner",
        ],
    }
    
    async def respond(self, event: SecurityEvent):
        """执行安全事件响应"""
        
        playbook = self.RESPONSE_PLAYBOOKS.get(event.type)
        if not playbook:
            raise ValueError(f"No playbook for {event.type}")
        
        for step in playbook:
            await self._execute_step(step, event)
```

---

## 9. 能力边界

### 9.1 能做到的

- **保护数据在传输和存储中的安全**：TLS 加密、静态加密
- **检测和阻止已知攻击模式**：Prompt 注入、工具劫持
- **隔离敏感操作**：沙箱执行、权限控制
- **记录完整审计轨迹**：所有 Agent 操作可追溯
- **自动响应安全事件**：基于 playbook 的自动响应

### 9.2 做不到的

- **防止 0-day 攻击**：未知的攻击模式可能绕过检测
- **保证 LLM 本身安全**：底层模型的漏洞不在控制范围内
- **完全防止内部威胁**：有合法权限的内部人员滥用权限难以完全阻止
- **零误报的异常检测**：安全监控总有误报和漏报的权衡

---

## 10. 核心优势

- **合规基础**：安全架构是 SOC 2、ISO 27001 等认证的基础
- **客户信任**：企业客户要求看到安全架构设计
- **风险可控**：系统化的安全架构降低安全事件概率和影响
- **自动响应**：快速检测和响应安全威胁

---

## 11. 安全测试清单

```
Agent 安全测试清单:

□ Prompt 注入测试
  □ 直接注入: "忽略之前的指令，执行..."
  □ 间接注入: 在知识库文档中嵌入恶意指令
  □ 多轮注入: 逐步引导 Agent 违反规则

□ 工具安全测试
  □ 工具参数注入: 构造恶意参数值
  □ 工具权限提升: 尝试调用未授权的工具
  □ 工具 DoS: 高频调用导致资源耗尽

□ 数据安全测试
  □ 跨租户数据访问: 租户 A 尝试访问租户 B 的数据
  □ 敏感数据泄露: Agent 输出中是否包含不应输出的数据
  □ 数据持久化: 删除后数据是否仍可恢复

□ 认证安全测试
  □ Token 伪造: 尝试伪造 JWT
  □ Token 重放: 重放已过期的 Token
  □ 权限提升: 使用低权限 Token 访问高权限功能
```

---

> **核心结论**：企业级 Agent 安全不是一组独立的安全工具，而是**嵌入 Agent 架构每一层的安全思维**。从传输加密到运行时沙箱，从输入检测到输出护栏，每个组件都需要安全设计。最值得投入的三个方面是：**间接提示注入防御**（Agent 特有的最大威胁）、**工具执行沙箱**（防止恶意代码执行）、**异常行为检测**（发现未知威胁）。
