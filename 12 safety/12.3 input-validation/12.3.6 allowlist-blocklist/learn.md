# 12.3.6 Allowlist / Blocklist -- 白名单/黑名单策略

---

## 简单介绍

Allowlist（白名单）和 Blocklist（黑名单）是两种基础的输入验证与访问控制策略，在 AI Agent 系统中用于约束 Agent 可以调用的工具、访问的域名、处理的参数以及执行的操作。

- **Allowlist（白名单）**：明确列出允许的项目，未列出的全部拒绝。其核心原则是"默认拒绝，显式允许"（Default Deny, Explicit Allow）。
- **Blocklist（黑名单）**：明确列出禁止的项目，未列出的全部允许。其核心原则是"默认允许，显式拒绝"（Default Allow, Explicit Deny）。

在 Agent 安全体系中，这两种策略是构建"输入验证"防线的基础手段，通常组合使用形成纵深防御。

---

## 基本原理

### Allowlist（白名单） -- Permit Known Good

允许明确已知安全/合法的输入，拒绝一切未明确允许的输入。

```
允许集合 = {item1, item2, item3, ...}
判断逻辑：input in 允许集合 ? 通过 : 拒绝
```

**优势**：安全边界清晰，误放风险低，策略可穷举验证。
**劣势**：维护成本高，灵活性差，容易遗漏合法用例。

### Blocklist（黑名单） -- Deny Known Bad

拒绝明确已知危险/非法的输入，允许一切未明确禁止的输入。

```
禁止集合 = {bad1, bad2, bad3, ...}
判断逻辑：input not in 禁止集合 ? 通过 : 拒绝
```

**优势**：灵活性高，新增功能不需要频繁修改策略，用户自由度大。
**劣势**：安全边界模糊，总有未知攻击方式绕过，策略无法被证明是完备的。

### 核心公式对比

| 维度 | Allowlist | Blocklist |
|------|-----------|-----------|
| 安全模型 | 正向匹配（Positive Matching） | 负向匹配（Negative Matching） |
| 完备性 | 可证明完备（封闭集合） | 不可证明完备（开放集合） |
| 误报率 | 高（合法请求被拒） | 低 |
| 漏报率 | 低（恶意请求难绕过） | 高（新型攻击可绕过） |
| 维护成本 | 随功能增长线性增加 | 随攻击面增长持续增加 |

---

## 背景

### 传统安全中的 Allowlist vs Blocklist

在传统网络安全领域，allowlist/blocklist 的争论由来已久：

- **Web 应用防火墙（WAF）**：早期 WAF 依赖签名库（blocklist）检测已知攻击（SQL 注入、XSS），但 0-day 攻击总是能绕过。现代 WAF 逐步转向 allowlist 模式，只允许符合 OpenAPI/Swagger 规范的请求。
- **端点安全**：传统杀毒软件基于病毒特征库（blocklist）工作，永远落后于新变种。现代 EDR 方案引入应用白名单（allowlist），只允许经过签名的可执行文件运行。
- **网络安全**：防火墙从早期的端口 blocklist 演进到默认 deny 的 allowlist 策略（只开放特定端口和服务）。

这一演进路径揭示了一个共识：allowlist 是更安全的长期方向，但 blocklist 作为补充手段不可或缺。

### Agent 系统中的特殊挑战

AI Agent 的输入验证面临传统系统没有的新挑战：

1. **自然语言输入的无限性**：Agent 接收的是自然语言指令，不是结构化 API 调用。合法指令的表达方式几乎是无限的（"帮我查天气" vs "今天会下雨吗" vs "看看外面用不用带伞"），无法用有限的 allowlist 覆盖。

2. **工具调用的组合爆炸**：Agent 可以链式调用多个工具，单个工具的参数组合呈指数级增长。即使每个工具单独验证通过，参数组合后可能产生危险行为（如同时调用"读取文件"和"发送网络请求"）。

3. **语义理解的模糊地带**：某些输入单独看是安全的，但在上下文中是恶意的。"删除用户 data.txt" 和 "删除系统 config.txt" 在语法上没有区别，语义风险完全不同。

4. **对抗性提示（Prompt Injection）**：攻击者可以通过精心构造的 prompt，使 Agent 在"合理"的请求中执行危险操作。传统的关键词 blocklist 难以抵御语义级别的攻击。

### 演进路径：从简单列表到语义 Allowlisting

Agent 系统的 allowlist/blocklist 策略经历了三个发展阶段：

| 阶段 | 描述 | 示例 |
|------|------|------|
| **V1: 静态列表** | 基于固定字符串匹配的简单白/黑名单 | 禁止 `rm -rf`，允许 `/api/v1/*` |
| **V2: 模式匹配** | 基于 glob/regex 的模式化匹配 | 允许 `file:///data/*.txt`，禁止 `DELETE /users/\d+` |
| **V3: 语义匹配** | 理解输入意图的语义级匹配 | 识别"删除系统文件"意图（即使使用同义词、变形） |

当前最先进的 Agent 安全系统处于 V2-V3 过渡阶段：核心工具调用使用 allowlist + 参数模式匹配，自然语言输入层使用基于 LLM 的语义安全分类器（本质上是"语义 allowlist"）。

---

## 核心矛盾

Allowlist 与 Blocklist 之间的根本矛盾可以概括为：**安全性与灵活性的永恒对抗**。

### Allowlist 的矛盾

> "Allowlist 是安全的，但安全到无法使用。"

- **过于严格**：一个合法的新 API 端点如果没有及时加入 allowlist，Agent 会拒绝执行，用户体验断崖式下降。
- **维护负载**：每次 Agent 能力升级（新增工具、扩大操作范围），都必须同步更新 allowlist，形成运维瓶颈。
- **覆盖盲区**：设计 allowlist 时无法预见所有合法用例，导致"能做的太少"。

### Blocklist 的矛盾

> "Blocklist 是灵活的，但灵活到形同虚设。"

- **永远不完备**：攻击者总能用新方式绕过已知的黑名单模式。`rm -rf /` 被禁止了，但 `rm -rf $ROOT_DIR` 呢？`rm --recursive --force /` 呢？
- **维护无底洞**：每发现一种新的攻击变体就要追加一条规则，规则库持续膨胀，性能和维护成本不断上升。
- **误放风险高**：只要有一条攻击路径没有被覆盖，系统就可能被攻破。

### 现实中的平衡策略

在实际 Agent 系统中，极少纯用一种策略。典型的做法是：

```
安全策略 = Allowlist（核心）+ Blocklist（补充）+ Semantic Filter（兜底）
```

- **Allowlist 层**：对 Agent 可调用的工具进行白名单管理（第一道防线，最严格）
- **Blocklist 层**：对工具参数进行黑名单模式过滤（第二道防线，捕获已知攻击）
- **Semantic Filter 层**：基于 LLM 对输入意图进行安全分类（第三道防线，处理未知威胁）

---

## 详细内容

### 1. Allowlist Strategy（白名单策略）

#### 1.1 工具级别 Allowlisting（Tool Allowlisting）

允许 Agent 调用哪些工具，是所有策略中最有效的一层。

```python
tool_allowlist = {
    "read_file",
    "write_file",
    "search_web",
    "calculate_math",
    "send_email_draft",
}
# Agent 只能调用以上工具，不能调用 deploy_service、execute_shell 等
```

**实施要点**：
- 按最小权限原则（Principle of Least Privilege）设计工具集
- 不同 Agent 角色可以有不同工具 allowlist
- 工具 allowlist 应在 Agent 启动时静态确定，运行时不可修改

#### 1.2 域名 Allowlisting（Domain Allowlisting）

Agent 的网络请求只能访问允许的域名/IP。

```python
domain_allowlist = {
    "api.openai.com",
    "*.internal.company.com",
    "data.example.org",
}
```

**实施要点**：
- 优先使用后缀匹配（`*.example.com`），而非精确匹配
- 注意子域名劫持风险——`*.example.com` 允许 `evil.example.com` 如果 DNS 被篡改
- 内部服务域名需要特别管控，避免 SSRF 攻击

#### 1.3 操作 Allowlisting（Operation Allowlisting）

对工具的具体操作进行控制，比如文件系统的读写权限分离。

```python
file_operation_allowlist = {
    "read": ["/home/user/data/*", "/tmp/*.csv"],
    "write": ["/home/user/output/*"],
    "delete": [],  # 不允许任何删除操作
    "execute": [], # 不允许任何执行操作
}
```

#### 1.4 参数 Allowlisting（Parameter Allowlisting）

对工具的特定参数值进行白名单约束。

```python
tool_param_allowlist = {
    "send_email": {
        "recipient_domain": ["company.com", "client.org"],
        "max_recipients": 10,
        "allowed_attachments": [".pdf", ".docx", ".xlsx"],
    }
}
```

---

### 2. Blocklist Strategy（黑名单策略）

#### 2.1 敏感指令 Blocklisting（Blocked Patterns）

禁止 Agent 执行包含特定模式的操作。

```python
blocked_patterns = [
    "rm -rf /",
    "DROP TABLE",
    "shutdown -h now",
    "format(",
]
```

**问题**：这些模式非常容易被绕过。攻击者可以用 `rm --recursive --force /`、`rm -rf $MOUNT_POINT`、使用十六进制编码等方式绕过。

#### 2.2 禁止关键词 Blocklisting（Forbidden Keywords）

在 Agent 的输入或输出中检测敏感关键词。

```python
forbidden_keywords = [
    "password",
    "credit_card",
    "ssn",
    "secret_key",
    "token",
]
```

**问题**：关键词检测会产生大量误报（"请重置我的密码"被误判为密码泄露），同时攻击者可以用同义词/编码绕过（"passwd"、"p@ssword"、"pwd"）。

#### 2.3 禁止参数组合（Prohibited Combinations）

某些工具单独使用是安全的，但组合使用存在风险。Blocklist 可以禁止特定参数组合。

```python
dangerous_combinations = [
    {"tool": "read_file", "path": "/etc/shadow"},
    {"tool": "execute_shell", "command": "sudo *"},
    {"tool": "network_request", "url": "*.secret.internal*"},
]
```

#### 2.4 危险操作组合（Dangerous Operation Chains）

某些操作单独无害，但链式组合构成攻击路径。

```python
dangerous_chains = [
    ["read_database", "send_email"],   # 读取数据库并外发
    ["read_file", "network_request"],  # 读取文件并网络外传
    ["execute_shell", "delete_file"],  # 执行并清理痕迹
]
```

---

### 3. Allowlist vs Blocklist in Agent Context

在 Agent 上下文中，allowlist 和 blocklist 各有其最佳适用场景。

#### 3.1 工具 Allowlisting（最有效）

**结论：工具级 Allowlist 是 Agent 安全中最有效的单一控制点。**

原因：工具是 Agent 能力的原子单元，数量有限（通常 10-50 个），边界清晰。对工具做 allowlist 意味着：

- 即使 prompt injection 成功，攻击者也只能在允许的工具集内活动
- 新增工具需要明确审批，安全评审点清晰
- 工具数量可控，allowlist 维护成本低

```python
# 推荐：用 allowlist 控制 Agent 可用的工具集
allowed_tools = {
    "search_knowledge_base",  # 安全
    "read_document",          # 安全（需配合路径验证）
    "draft_email",            # 安全（需配合收件人验证）
    "calculate",              # 安全
    "get_weather",            # 安全
}
# 绝对不放在 allowlist 中的工具：
# "execute_command", "delete_resource", "modify_system_config"
```

#### 3.2 域名 Blocklisting（效果最差）

**结论：域名级 Blocklist 是最不可靠的控制手段，应仅作为辅助。**

原因：域名无限且攻击者可以注册任意域名，blocklist 永远无法跟上攻击者的步伐。

```
# 不推荐：仅依赖域名黑名单
blocked_domains = ["evil.com", "malware.net", "phishing.org"]
# 攻击者只需注册一个新域名即可绕过
```

**更好的做法**：对网络请求使用 allowlist + 内容审查的组合。

```
# 推荐做法
network_access = DomainAllowlist(allowed=["*.trusted.com", "api.safe.org"])
                        + ContentInspection(block_sensitive_data_exfiltration=True)
```

#### 3.3 参数限制（混合策略）

参数级别的安全控制需要混合使用 allowlist 和 blocklist：

| 参数类型 | 推荐策略 | 原因 |
|----------|---------|------|
| 枚举值 | Allowlist | 值是有限的已知集合（如日志级别：debug/info/error） |
| 文件路径 | Allowlist + 正则 | 路径空间大但有模式可循（如 `/data/*.csv`） |
| 字符串内容 | Blocklist | 内容无限，只能禁止已知危险模式 |
| 数值范围 | Allowlist | 范围是明确的（如 `1 <= page_size <= 100`） |
| URL | Allowlist | 对域名做 allowlist，对路径做 blocklist 补充 |

---

### 4. Pattern-Based Matching（基于模式的匹配）

实际的 allowlist/blocklist 很少使用精确字符串匹配，而是基于各种模式匹配技术。

#### 4.1 精确匹配（Exact Match）

最简单的匹配方式，适用于枚举值。

```
允许: "read_only" / "read_write" / "admin"
拒绝: "read_only" (如果不在允许列表中)
```

**适用场景**：角色名称、状态枚举、操作类型等有限集合。

#### 4.2 Glob 模式匹配（Glob Patterns）

使用通配符进行路径/域名匹配。

```
允许: /data/**/*.txt
禁止: /etc/shadow
匹配: /data/projects/report.txt -> 允许
匹配: /etc/shadow -> 禁止
匹配: /data/.config/secret.json -> 拒绝（不在允许列表中）
```

**常用通配符**：
- `*`：匹配任意字符（除路径分隔符）
- `**`：匹配任意路径层级
- `?`：匹配单个任意字符
- `[abc]`：匹配字符集中的任意一个

#### 4.3 正则表达式匹配（Regex Patterns）

最灵活的模式匹配方式，适用于复杂条件。

```python
# 正则 allowlist：只允许特定格式的文件路径
path_pattern = r"^/data/(projects|reports|exports)/[\w\-\.]+\.(csv|json|txt)$"

# 正则 blocklist：禁止包含敏感模式的 SQL 查询
sql_block_pattern = r"(?i)(DROP|TRUNCATE|ALTER|DELETE\s+FROM\s+\w+\s+WHERE)"
```

**注意事项**：
- 正则表达式本身可能被 ReDoS（ReDoS，Regular Expression Denial of Service）攻击利用
- 复杂正则的性能开销不可忽略
- 正则的覆盖范围可能不精确，导致过宽或过窄

#### 4.4 语义模式匹配（Semantic Patterns）

基于 LLM 或 NLP 模型理解输入语义，是最新的防线。

```python
# 语义安全分类器
def semantic_safety_check(user_input: str, context: dict) -> SafetyVerdict:
    """
    使用 LLM 判断用户输入的语义是否安全。
    不是匹配关键词，而是理解意图。
    """
    prompt = f"""
    用户输入: {user_input}
    当前上下文: {context}
    
    请判断这是否是恶意操作：
    - JAILBREAK: 试图越狱或绕过安全限制
    - DATA_EXFIL: 试图窃取敏感数据
    - UNAUTHORIZED: 试图执行未授权的操作
    - SAFE: 正常请求
    
    只需要返回分类标签。
    """
    return llm_classify(prompt)
```

**优势**：
- 能检测未见过的攻击模式（0-day 防护）
- 理解上下文语义，减少误报
- 能识别 paraphrased 的攻击指令

**劣势**：
- 延迟较高（LLM 推理时间）
- 成本较高（每次 API 调用）
- LLM 本身可能被攻击（如 prompt injection）

---

### 5. Dynamic Allowlist/Blocklist（动态列表）

静态列表难以适应动态变化的 Agent 使用场景，动态列表机制提供了适应性。

#### 5.1 基于用户行为学习（Behavioral Learning）

系统自动识别用户的使用模式，动态调整允许/禁止列表。

```python
class BehavioralAllowlist:
    def __init__(self):
        self.base_allowlist = set()
        self.usage_frequency = defaultdict(int)
        self.learning_mode = True
        
    def record_usage(self, item: str, approved: bool):
        if approved:
            self.usage_frequency[item] += 1
            # 高频使用的合法项自动加入候选 allowlist
            if self.usage_frequency[item] > THRESHOLD:
                self.base_allowlist.add(item)
    
    def get_suggested_additions(self):
        """返回建议加入 allowlist 的项目（需要人工审核）"""
        return {item for item, count in self.usage_frequency.items()
                if count > THRESHOLD and item not in self.base_allowlist}
```

**注意事项**：自动学习必须有"人工审核"环节（Human-in-the-Loop），防止攻击者通过刻意的高频访问污染 allowlist。

#### 5.2 自适应更新（Adaptive Updating）

根据威胁情报、攻击检测结果自动更新 blocklist。

```python
class AdaptiveBlocklist:
    def __init__(self):
        self.blocklist = set()
        self.attack_signatures = []
        
    def on_attack_detected(self, attack_pattern: str):
        """检测到攻击后自动提取特征并加入 blocklist"""
        signature = self._extract_signature(attack_pattern)
        self.blocklist.add(signature)
        self._propagate_to_other_agents(signature)  # 跨 Agent 同步
    
    def _extract_signature(self, attack: str) -> str:
        """从攻击 payload 中提取通用特征"""
        # 去除非核心部分，保留关键模式
        return normalize_pattern(attack)
```

#### 5.3 基于时间规则（Time-Based Rules）

允许/禁止规则可以根据时间上下文动态生效。

```python
time_based_rules = [
    {
        "time": "09:00-18:00",
        "allow": ["read", "write", "send_email"],
        "deny": ["delete", "deploy"],
    },
    {
        "time": "18:00-09:00",
        "allow": ["read"],
        "deny": ["write", "delete", "send_email", "deploy"],
    },
    {
        "special": "incident_response",
        "allow": ["read", "write", "delete", "deploy"],
        "require": "break_glass_approval",
    },
]
```

**典型场景**：
- 工作时间外禁止危险操作
- 部署窗口期内才允许发布操作
- 紧急事件响应模式下可临时突破限制（Break Glass）

---

### 6. Hierarchical Lists（层次化列表）

企业级 Agent 系统中，allowlist/blocklist 不是平面结构，而是多层级的层次化体系。

#### 6.1 层级结构

```
Global Level（全局）
  ├── Tenant Level（租户）
  │   ├── User Level（用户）
  │   │   └── Session Level（会话）
  │   └── User Level
  └── Tenant Level
       └── User Level
```

#### 6.2 各层级职责

| 层级 | 范围 | 示例规则 | 管理者 |
|------|------|---------|--------|
| **Global** | 所有 Agent 实例 | 禁止 `rm -rf /`，禁止调用 `shell_execute` | 安全团队 |
| **Tenant** | 特定租户/组织 | 允许访问 `*.tenant-specific.internal` | 组织管理员 |
| **User** | 特定用户 | 用户 A 可以 `deploy_dev`，用户 B 可以 `deploy_prod` | 用户管理员 |
| **Session** | 单次会话 | 本次会话中只能读取文件 A（临时授权） | 用户自己 |

#### 6.3 覆盖优先级

```
Session > User > Tenant > Global
```

也就是说，低层级（更具体）的规则覆盖高层级（更通用）的规则。

```
Global:   Allow {read, write}, Deny {delete, execute}
Tenant:   Allow {delete} on /tenant-backup/
User:     Allow {execute} on /user-scripts/*.sh
Session:  (本次会话临时禁止 write)

结果：User 可以 read, write(但本次会话不行), delete(仅 /tenant-backup/), execute(仅 /user-scripts/*.sh)
```

#### 6.4 实现中的关键设计

```python
class HierarchicalAllowlist:
    def __init__(self):
        self.layers = {
            "global": AllowlistLayer(),
            "tenant": AllowlistLayer(),
            "user": AllowlistLayer(),
            "session": AllowlistLayer(),
        }
        
    def check(self, operation: str, context: Context) -> bool:
        """
        从最具体层级到最通用层级依次检查。
        任何一层明确禁止（DENY）则直接拒绝。
        任何一层明确允许则继续向通用层检查。
        """
        layers = ["session", "user", "tenant", "global"]
        
        for layer in layers:
            decision = self.layers[layer].evaluate(operation, context)
            if decision == Decision.DENY:
                return False
            if decision == Decision.GRANT:
                continue  # 此层通过，继续向上检查
        
        # 所有层都通过
        return True
```

**关键原则**：
- **Deny takes precedence**（拒绝优先）：任何层级的明确拒绝都阻断操作
- **Explicit beats implicit**（显式覆盖隐式）：显式规则优先于默认策略
- **Least privilege at every layer**（层层最小权限）：每层都应尽可能收缩权限

---

### 7. Testing & Validation（测试与验证）

Allowlist/Blocklist 策略的正确性需要通过系统化的测试来保障。

#### 7.1 覆盖率测试（Coverage Testing）

确保 allowlist 覆盖了所有已知的合法用例。

```python
def test_allowlist_coverage():
    # 场景：所有合法工具调用应被 allowlist 允许
    known_legitimate_operations = [
        ("read_file", {"path": "/data/report.csv"}),
        ("search_web", {"query": "weather today"}),
        ("send_email", {"to": "user@company.com", "body": "Hello"}),
        ("calculate", {"expression": "2 + 2"}),
    ]
    
    for tool, params in known_legitimate_operations:
        assert allowlist.check(tool, params), f"合法操作被拒绝: {tool}({params})"
```

#### 7.2 绕过检测测试（Bypass Detection）

系统化地测试 blocklist 能否被绕过。

```python
def test_blocklist_bypass():
    # 场景：所有已知的攻击变体应被 blocklist 阻止
    attack_variants = [
        # rm -rf / 的各种变体
        "rm -rf /",
        "rm --recursive --force /",
        "rm -rf $ROOT",
        "rm -rf /var/empty/../",
        "`rm -rf /`",
        "$(rm -rf /)",
        "rm -rf '/'",
        # SQL 注入变体
        "1; DROP TABLE users",
        "1 UNION SELECT * FROM passwords",
        "1; DROP/**/TABLE users",
    ]
    
    for attack in attack_variants:
        assert blocklist.check(attack), f"绕过成功: {attack}"
```

#### 7.3 回归测试（Regression Testing）

每次更新 allowlist/blocklist 后，确保之前能通过的合法请求仍然通过，之前能拒绝的恶意请求仍然被拒绝。

```python
class RegressionTestSuite:
    def __init__(self):
        self.positive_tests = []   # 应该通过的合法请求
        self.negative_tests = []   # 应该被拒绝的恶意请求
    
    def add_positive(self, name: str, tool: str, params: dict):
        self.positive_tests.append((name, tool, params))
    
    def add_negative(self, name: str, tool: str, params: dict):
        self.negative_tests.append((name, tool, params))
    
    def run(self, checker):
        for name, tool, params in self.positive_tests:
            assert checker.check(tool, params), f"回归失败: 正例 {name} 被拒绝"
        
        for name, tool, params in self.negative_tests:
            assert not checker.check(tool, params), f"回归失败: 负例 {name} 被允许"
```

---

## Example Code

以下是一个完整的 Python 实现，包含 AllowlistManager 和 BlocklistManager，支持层次化规则、模式匹配和动态更新机制。

```python
"""
Allowlist / Blocklist Manager for AI Agent Systems
"""
import re
import fnmatch
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional, Pattern


class Decision(Enum):
    """检查决策结果"""
    ALLOW = "allow"
    DENY = "deny"
    ABSTAIN = "abstain"  # 不做判断，交由上层决策


class MatchType(Enum):
    """匹配模式类型"""
    EXACT = "exact"
    GLOB = "glob"
    REGEX = "regex"
    PREFIX = "prefix"
    SUFFIX = "suffix"


@dataclass
class Rule:
    """单条规则"""
    id: str
    match_type: MatchType
    pattern: str
    compiled: Optional[Pattern] = field(default=None, init=False)
    metadata: dict = field(default_factory=dict)
    priority: int = 0

    def __post_init__(self):
        if self.match_type == MatchType.REGEX:
            self.compiled = re.compile(self.pattern)

    def matches(self, value: str) -> bool:
        """判断给定值是否匹配当前规则"""
        if self.match_type == MatchType.EXACT:
            return value == self.pattern
        elif self.match_type == MatchType.GLOB:
            return fnmatch.fnmatch(value, self.pattern)
        elif self.match_type == MatchType.REGEX:
            return bool(self.compiled.search(value))
        elif self.match_type == MatchType.PREFIX:
            return value.startswith(self.pattern)
        elif self.match_type == MatchType.SUFFIX:
            return value.endswith(self.pattern)
        return False


# ============================================================
# AllowlistManager
# ============================================================

class AllowlistManager:
    """
    白名单管理器：明确允许已知安全的操作，拒绝所有未明确允许的请求。
    
    设计原则：Default Deny - 只有显式列入 allowlist 的操作才被允许。
    """

    def __init__(self, name: str = "default"):
        self.name = name
        self._tool_allowlist: dict[str, list[Rule]] = {}     # tool -> rules[]
        self._domain_allowlist: list[Rule] = []
        self._param_allowlist: dict[str, dict[str, list[Rule]]] = {}  # tool -> param -> rules[]
        self._audit_log: list[dict] = []
        self._enabled: bool = True

    # ---- Tool Level Allowlist ----

    def allow_tool(self, tool_name: str, match_type: MatchType = MatchType.EXACT,
                   pattern: str = None, metadata: dict = None):
        """允许指定的工具（及其变体模式）"""
        if pattern is None:
            pattern = tool_name
        rule = Rule(
            id=f"tool_allow_{tool_name}_{len(self._tool_allowlist)}",
            match_type=match_type,
            pattern=pattern,
            metadata=metadata or {},
        )
        self._tool_allowlist.setdefault(tool_name, []).append(rule)
        return self

    def remove_tool_allow(self, tool_name: str):
        """移除工具的 allow 规则"""
        self._tool_allowlist.pop(tool_name, None)
        return self

    def is_tool_allowed(self, tool_name: str) -> bool:
        """检查工具是否在白名单中"""
        if not self._enabled:
            return True
        if tool_name in self._tool_allowlist:
            return True
        # 检查通配符规则
        for allowed_tool, rules in self._tool_allowlist.items():
            for rule in rules:
                if rule.matches(tool_name):
                    return True
        return False

    # ---- Domain Level Allowlist ----

    def allow_domain(self, domain_pattern: str, match_type: MatchType = MatchType.GLOB):
        """允许访问指定域名模式"""
        rule = Rule(
            id=f"domain_allow_{domain_pattern}",
            match_type=match_type,
            pattern=domain_pattern,
        )
        self._domain_allowlist.append(rule)
        return self

    def is_domain_allowed(self, domain: str) -> bool:
        """检查域名是否在白名单中"""
        if not self._enabled:
            return True
        return any(rule.matches(domain) for rule in self._domain_allowlist)

    # ---- Parameter Level Allowlist ----

    def allow_param(self, tool_name: str, param_name: str,
                    pattern: str, match_type: MatchType = MatchType.EXACT):
        """允许工具的特定参数值"""
        rule = Rule(
            id=f"param_allow_{tool_name}_{param_name}_{pattern}",
            match_type=match_type,
            pattern=pattern,
        )
        self._param_allowlist.setdefault(tool_name, {}).setdefault(param_name, []).append(rule)
        return self

    def is_param_allowed(self, tool_name: str, param_name: str, param_value: str) -> bool:
        """检查参数值是否在白名单中"""
        if not self._enabled:
            return True
        tool_params = self._param_allowlist.get(tool_name, {})
        param_rules = tool_params.get(param_name, [])
        if not param_rules:
            # 没有为该参数定义 allowlist 规则，按照工具级别的默认策略
            # 如果工具在白名单中但没有参数规则，视为允许所有参数
            return True
        return any(rule.matches(param_value) for rule in param_rules)

    # ---- Composite Check ----

    def check(self, tool_name: str, params: dict = None) -> tuple[bool, str]:
        """
        综合检查：工具 + 域名 + 参数是否全部通过 allowlist。
        
        返回 (通过, 拒绝原因)
        """
        if not self._enabled:
            return True, "allowlist disabled"

        # 1. 检查工具
        if not self.is_tool_allowed(tool_name):
            self._audit(tool_name, params, False, f"Tool '{tool_name}' not in allowlist")
            return False, f"Tool '{tool_name}' is not allowed"

        # 2. 检查参数
        if params:
            for param_name, param_value in params.items():
                if isinstance(param_value, str):
                    # 检查域名参数（如 url, domain, host）
                    if param_name in ("url", "domain", "host", "endpoint"):
                        domain = self._extract_domain(param_value)
                        if domain and not self.is_domain_allowed(domain):
                            self._audit(tool_name, params, False,
                                        f"Domain '{domain}' not in allowlist")
                            return False, f"Domain '{domain}' is not allowed"

                    # 检查参数级别的 allowlist
                    if not self.is_param_allowed(tool_name, param_name, param_value):
                        self._audit(tool_name, params, False,
                                    f"Param '{param_name}={param_value}' not in allowlist")
                        return False, f"Parameter '{param_name}' value is not allowed"

        self._audit(tool_name, params, True, "Allowed")
        return True, "Allowed"

    def _extract_domain(self, url: str) -> Optional[str]:
        """从 URL 中提取域名"""
        import re
        match = re.search(r'https?://([^/]+)', url)
        return match.group(1) if match else None

    def _audit(self, tool: str, params: dict, allowed: bool, reason: str):
        """记录审计日志"""
        self._audit_log.append({
            "tool": tool,
            "params": params,
            "allowed": allowed,
            "reason": reason,
        })

    def get_audit_log(self) -> list[dict]:
        return self._audit_log.copy()

    def enable(self):
        self._enabled = True

    def disable(self):
        self._enabled = False


# ============================================================
# BlocklistManager
# ============================================================

class BlocklistManager:
    """
    黑名单管理器：明确禁止已知危险的操作，允许所有未明确禁止的请求。
    
    设计原则：Default Allow - 只有显式列入 blocklist 的操作才被拒绝。
    注意：这本身就是一个不安全的设计原则，BlocklistManager 应作为
    AllowlistManager 的补充，而不是替代。
    """

    def __init__(self, name: str = "default"):
        self.name = name
        self._keyword_blocklist: list[Rule] = []
        self._pattern_blocklist: dict[str, list[Rule]] = {}  # tool -> rules[]
        self._combination_blocklist: list[dict] = []
        self._audit_log: list[dict] = []
        self._enabled: bool = True

    # ---- Keyword Level Blocklist ----

    def block_keyword(self, keyword: str, match_type: MatchType = MatchType.SUBSTRING):
        """禁止包含特定关键词的输入"""
        rule = Rule(
            id=f"keyword_block_{keyword}",
            match_type=match_type,
            pattern=keyword,
        )
        self._keyword_blocklist.append(rule)
        return self

    # ---- Tool Pattern Blocklist ----

    def block_pattern(self, tool_name: str, pattern: str,
                      match_type: MatchType = MatchType.REGEX):
        """禁止工具的特定参数模式"""
        rule = Rule(
            id=f"pattern_block_{tool_name}_{pattern[:20]}",
            match_type=match_type,
            pattern=pattern,
        )
        self._pattern_blocklist.setdefault(tool_name, []).append(rule)
        return self

    # ---- Dangerous Combination Blocklist ----

    def block_combination(self, tool_sequence: list[str], reason: str = ""):
        """禁止特定的工具调用组合/链"""
        self._combination_blocklist.append({
            "sequence": tool_sequence,
            "reason": reason or f"Dangerous combination: {' -> '.join(tool_sequence)}",
        })
        return self

    # ---- Check Methods ----

    def check_tool_params(self, tool_name: str, params: dict) -> tuple[bool, str]:
        """检查工具的特定参数是否存在黑名单模式"""
        if not self._enabled:
            return True, "blocklist disabled"

        # 检查关键词（对所有输入）
        for rule in self._keyword_blocklist:
            for param_name, param_value in params.items():
                if isinstance(param_value, str) and rule.matches(param_value):
                    reason = f"Blocked keyword '{rule.pattern}' in param '{param_name}'"
                    self._audit(tool_name, params, False, reason)
                    return False, reason

        # 检查工具特定的模式
        for rule in self._pattern_blocklist.get(tool_name, []):
            for param_name, param_value in params.items():
                if isinstance(param_value, str) and rule.matches(param_value):
                    reason = f"Blocked pattern in param '{param_name}'"
                    self._audit(tool_name, params, False, reason)
                    return False, reason

        self._audit(tool_name, params, True, "Passed")
        return True, "Passed"

    def check_combination(self, tool_history: list[str]) -> tuple[bool, str]:
        """检查历史工具调用序列是否匹配禁止组合"""
        for combo in self._combination_blocklist:
            seq = combo["sequence"]
            if len(tool_history) >= len(seq):
                # 检查工具序列末尾是否匹配禁止组合
                if tool_history[-len(seq):] == seq:
                    return False, combo["reason"]
        return True, "No dangerous combination detected"

    def _audit(self, tool: str, params: dict, allowed: bool, reason: str):
        self._audit_log.append({
            "tool": tool,
            "params": params,
            "allowed": allowed,
            "reason": reason,
        })


# ============================================================
# CompositeValidator: Allowlist + Blocklist 组合验证器
# ============================================================

class CompositeValidator:
    """
    组合验证器：同时使用 allowlist 和 blocklist 进行多层验证。
    
    验证流程：
    1. Allowlist 检查（先检查是否明确允许）
       - 如果工具不在 allowlist 中 -> 拒绝
       - 如果域名不在 allowlist 中 -> 拒绝  
       - 如果参数值不在 allowlist 中 -> 拒绝
    2. Blocklist 检查（后检查是否明确禁止）
       - 如果参数值匹配 blocklist 模式 -> 拒绝
       - 如果工具序列匹配禁止组合 -> 拒绝
    3. 全部通过 -> 允许
    """

    def __init__(self):
        self.allowlist = AllowlistManager("primary")
        self.blocklist = BlocklistManager("secondary")
        self._tool_history: list[str] = []

    def validate(self, tool_name: str, params: dict = None) -> tuple[bool, str, str]:
        """
        三级验证。
        
        Returns:
            (是否通过, 阶段信息, 具体原因)
        """
        params = params or {}

        # Stage 1: Allowlist Check
        allowed, reason = self.allowlist.check(tool_name, params)
        if not allowed:
            return False, "allowlist", reason

        # Stage 2: Blocklist Check - Parameters
        passed, reason = self.blocklist.check_tool_params(tool_name, params)
        if not passed:
            return False, "blocklist", reason

        # Stage 3: Blocklist Check - Tool Sequence
        self._tool_history.append(tool_name)
        passed, reason = self.blocklist.check_combination(self._tool_history)
        if not passed:
            return False, "blocklist_combination", reason

        return True, "all", "Validated"

    def reset_history(self):
        self._tool_history.clear()


# ============================================================
# 使用示例
# ============================================================

if __name__ == "__main__":
    validator = CompositeValidator()

    # 配置 Allowlist
    validator.allowlist.allow_tool("read_file")
    validator.allowlist.allow_tool("search_web")
    validator.allowlist.allow_tool("send_email")
    validator.allowlist.allow_tool("calculate")

    # 配置域名 Allowlist
    validator.allowlist.allow_domain("*.company.com")
    validator.allowlist.allow_domain("api.openai.com")

    # 配置参数 Allowlist
    validator.allowlist.allow_param("read_file", "path", "/data/**/*.csv", MatchType.GLOB)
    validator.allowlist.allow_param("send_email", "to", "*@company.com", MatchType.GLOB)

    # 配置 Blocklist
    validator.blocklist.block_keyword("password", MatchType.SUBSTRING)
    validator.blocklist.block_keyword("DROP TABLE", MatchType.SUBSTRING)
    validator.blocklist.block_pattern("read_file", r"(/etc/passwd|/etc/shadow)")
    validator.blocklist.block_combination(
        ["read_file", "send_email"],
        reason="Data exfiltration via file read + email send"
    )

    # 测试用例
    test_cases = [
        ("read_file", {"path": "/data/report.csv"}),             # 应通过
        ("read_file", {"path": "/etc/passwd"}),                  # 应被 blocklist 拒绝
        ("send_email", {"to": "user@company.com", "body": "Hi"}),# 应通过
        ("execute_shell", {"command": "ls"}),                    # 应被 allowlist 拒绝
        ("read_file", {"path": "/data/secret.txt"}),             # 应被 allowlist 拒绝（不匹配 .csv）
        ("search_web", {"query": "my password is 123"}),         # 应被 blocklist 拒绝（含 keywords）
    ]

    print("=" * 60)
    print("Allowlist / Blocklist Validation Tests")
    print("=" * 60)

    for tool, params in test_cases:
        passed, stage, reason = validator.validate(tool, params)
        status = "PASS" if passed else "BLOCK"
        print(f"[{status}] {stage}: {tool}({params}) -> {reason}")
```

---

## Capability Boundaries（能力边界）

Allowlist 和 Blocklist 各有其根本性的能力边界，理解这些边界是正确设计安全策略的前提。

### Blocklist 的根本局限性

1. **不完备性定理**（Incompleteness）：不存在一个 blocklist 能覆盖所有可能的攻击。这是由攻击面的无限性和攻击手段的创造性决定的。任何 blocklist 都可以被一个足够有创造力的攻击者绕过。

2. **规则膨胀**：随着攻击变体的增多，blocklist 的规则数量线性增长。当规则数量超过一定阈值后：
   - 性能下降（每次请求需要匹配数万条规则）
   - 管理成本上升（规则之间的冲突和冗余）
   - 误报率上升（规则覆盖了意料之外的合法输入）

3. **检测窗口**（Detection Gap）：从新型攻击出现到 blocklist 规则更新之间存在时间窗口。对于 0-day 攻击，blocklist 完全无效。

4. **适应性问题**：在 Agent 场景中，用户可以通过同义词替换、句式重组、编码变换等方式轻松绕过关键词级别的 blocklist。

### Allowlist 的根本局限性

1. **覆盖完整性**（Completeness）：设计 allowlist 时，需要预知所有合法的用例。对于 Agent 系统，自然语言输入的无限表达方式使得完整覆盖变得不可能。

   ```
   问题举例：
   合法请求可以表达为：
   - "帮我查一下北京的天气"
   - "北京今天多少度？"
   - "看看北京用不用穿外套"
   - "北京天气怎么样？"
   
   用静态 allowlist 不可能穷举所有合法表达。
   ```

2. **创新抑制**：Allowlist 本质上会抑制 Agent 能力的"意外发现"。用户可能发现 Agent 能以非预期的方式完成有用任务，但 allowlist 会阻止这些非预期的能力发挥。

3. **运维滞后**：Agent 能力升级（新增工具、扩大操作范围）与 allowlist 更新之间存在时间差。这个滞后时间成为业务效率的瓶颈。

4. **粒度假象**（Granularity Illusion）：过细粒度的 allowlist 给人一种"安全"的假象，但微观上的精确控制可能在宏观上留下巨大的攻击面。

### 实际限制总结

| 维度 | Allowlist 限制 | Blocklist 限制 |
|------|---------------|---------------|
| 覆盖性 | 无法覆盖所有合法输入 | 无法覆盖所有恶意输入 |
| 可证明性 | 对静态工具集可证明 | 永远不可证明 |
| 适应性 | 不适应动态变化的业务需求 | 不适应持续演变的攻击技术 |
| 误报 | 合法请求被拒（业务损失） | 攻击被放过（安全损失） |
| 维护成本 | 随功能增长线性增加 | 随攻击面增长指数级增加 |
| 用户感知 | 限制感强，体验差 | 体验好，但信任度低 |

### 认识到"不能依赖单一策略"

这些根本局限性意味着：**没有任何一种单一的 allowlist 或 blocklist 策略是足够的**。Agent 安全需要纵深防御：

```
输入层（自然语言）: Semantic Safety Classifier (LLM-based)
工具层（工具选择）: Tool Allowlist (最严格)
参数层（参数验证）: Param Allowlist + Pattern Blocklist
执行层（运行时）  : Resource Quota + Audit Logging
监控层（事后）    : Anomaly Detection + Human Review
```

---

## Comparison（对比决策矩阵）

以下矩阵帮助为不同的 Agent 场景选择合适的策略。

### 决策矩阵

| 场景 | 推荐策略 | 原因 |
|------|---------|------|
| **Agent 工具集管理** | Allowlist | 工具数量有限（10-50个），边界清晰，适合正向列表 |
| **URL/域名访问** | Allowlist | 可访问的服务是确定的，blocklist 永远防不住新域名 |
| **文件路径访问** | Allowlist + Blocklist | Allowlist 控制可访问的根目录，Blocklist 阻止敏感路径 |
| **SQL 查询** | Blocklist（辅助）+ Parametric Query（主要） | 依赖参数化查询（安全），blocklist 只做补充过滤 |
| **用户自然语言输入** | Semantic Filter（LLM）+ Blocklist（关键词） | 自然语言表达无限，LLM 语义过滤 + 关键词兜底 |
| **API 参数枚举值** | Allowlist | 值是有限集合（如 status: active/inactive/suspended） |
| **文件上传类型** | Allowlist | 只允许已知安全的 MIME 类型 |
| **邮件收件人** | Allowlist | 只允许向已知安全域名发送邮件 |
| **代码执行/Shell** | Allowlist | 永远只允许预先定义的命令列表（最好完全禁止） |
| **内部网络访问** | Allowlist | 明确定义 Agent 可以访问的内部服务列表 |
| **第三方集成** | Allowlist | 只允许预先审核的第三方 API |
| **用户自定义脚本** | Blocklist + Sandbox | 用 blocklist 过滤已知危险模式，沙箱兜底 |
| **日志输出** | Blocklist | 阻止敏感信息（密码、token）出现在日志中 |
| **Agent 间通信** | Allowlist | 明确哪些 Agent 可以互相通信 |
| **数据库写操作** | Allowlist | 只允许预先定义的数据写入点 |
| **数据库读操作** | Allowlist + Row-Level Security | 控制可读取的表和行 |

### 策略选择流程图

```
输入需要保护的对象
        │
        ▼
对象集合是否有限且已知？
   ├── 是 ──► Allowlist（最优选择）
   │
   └── 否 ──► 对象集合是否可模式化？
        │
        ├── 是 ──► Allowlist with Pattern Matching（如路径模式）
        │
        └── 否 ──► 是否需要高安全性？
             │
             ├── 是 ──► 限制操作范围 + Semantic Filter
             │
             └── 否 ──► Blocklist + 监控（理解风险可接受）
```

### 场景权重评分表

为每个场景的四个维度打分（1-5），辅助决策：

| 场景 | 安全性要求 | 灵活性要求 | 可预见性 | 变更频率 | 推荐策略 |
|------|-----------|-----------|---------|---------|---------|
| 工具调用 | 5 | 2 | 5 | 1 | Allowlist |
| 网络请求 | 4 | 3 | 4 | 2 | Allowlist |
| 文件访问 | 5 | 3 | 3 | 2 | Allowlist + Blocklist |
| 用户输入 | 3 | 5 | 1 | 5 | Semantic Filter + Blocklist |
| SQL 查询 | 5 | 4 | 2 | 3 | 参数化 + Blocklist |

---

## Engineering Optimization（工程优化）

在 Agent 生产环境中，allowlist/blocklist 的查询性能至关重要，因为每次 Agent 工具调用都需要经过安全验证，而 Agent 可能每秒处理数十到数百次工具调用。

### 1. Trie-Based Pattern Matching（基于 Trie 的模式匹配）

对于大量前缀/精确匹配规则，使用 Trie（前缀树）可以大幅提升匹配性能。

```python
class TrieNode:
    """Trie 树节点"""
    __slots__ = ('children', 'is_end', 'metadata')
    
    def __init__(self):
        self.children = {}
        self.is_end = False
        self.metadata = None


class AllowlistTrie:
    """
    基于 Trie 树的 allowlist。
    特别适合域名白名单（如 *.company.com）和路径白名单。
    时间复杂度：O(L) 其中 L 是输入字符串长度。
    空间复杂度：O(N*K) 其中 N 是规则数，K 是平均模式长度。
    """

    def __init__(self):
        self.root = TrieNode()
        self._rule_count = 0

    def insert(self, pattern: str, metadata: dict = None):
        """插入一条 allowlist 规则"""
        # 反向插入以支持后缀匹配（如 *.company.com）
        parts = pattern.strip('*').split('.')
        node = self.root
        for part in reversed(parts):
            if part not in node.children:
                node.children[part] = TrieNode()
            node = node.children[part]
        node.is_end = True
        node.metadata = metadata
        self._rule_count += 1

    def search(self, value: str) -> bool:
        """搜索值是否在 allowlist 中"""
        parts = value.split('.')
        node = self.root
        
        # 尝试完整匹配
        for part in reversed(parts):
            if part in node.children:
                node = node.children[part]
                if node.is_end:
                    return True
            else:
                break
        
        # 检查是否有通配符匹配（允许部分后缀匹配）
        node = self.root
        if node.is_end:  # 如果 root 被标记为结束，意味着 "*" 通配所有
            return True
        
        return False

    @property
    def rule_count(self) -> int:
        return self._rule_count


# ---- 性能演示 ----
def trie_vs_linear_benchmark():
    """对比 Trie 和线性扫描的性能"""
    import time
    
    # 生成 10000 条域名规则
    domains = [f"sub{i}.example.com" for i in range(10000)]
    
    # Trie 方法
    trie = AllowlistTrie()
    for d in domains:
        trie.insert(d)
    
    # 线性扫描方法
    domain_set = set(domains)
    
    # 测试查找
    test_domains = ["sub5000.example.com", "notfound.other.com"]
    
    for td in test_domains:
        # Trie 查询
        t0 = time.perf_counter()
        r1 = trie.search(td)
        t1 = time.perf_counter()
        
        # Set 查找（Python 的 set 也是哈希表，理论上 O(1)）
        t2 = time.perf_counter()
        r2 = td in domain_set
        t3 = time.perf_counter()
        
        trie_time = (t1 - t0) * 1_000_000  # microseconds
        set_time = (t3 - t2) * 1_000_000
        
        print(f"Trie: {trie_time:.2f} us, Set: {set_time:.2f} us")
```

**Trie 的优势**：
- 支持通配符匹配（如 `*.example.com` 匹配 `sub.sub.example.com`）
- 匹配时间只与输入长度有关，与规则数量无关
- 天然支持最长公共前缀压缩

### 2. Bloom Filters for Large Lists（布隆过滤器处理大列表）

当 blocklist 包含数百万条规则（如恶意域名列表、已知攻击签名），使用 Bloom Filter 可以大幅降低内存和查询时间。

```python
import hashlib
import math
from bitarray import bitarray


class BloomFilter:
    """
    布隆过滤器用于大规模 blocklist。
    
    特点：
    - 空间效率极高（百万级规则只需要几 MB 内存）
    - 查询时间固定 O(k)，k 为哈希函数数量
    - 可能有误报（False Positive），但不会有漏报（False Negative）
    - 适用于"大概率是安全的，快速过滤已知恶意"的场景
    """

    def __init__(self, expected_items: int, false_positive_rate: float = 0.01):
        """
        初始化 Bloom Filter。
        
        Args:
            expected_items: 预期的元素数量
            false_positive_rate: 目标误报率（0-1）
        """
        self.size = self._optimal_size(expected_items, false_positive_rate)
        self.hash_count = self._optimal_hash_count(self.size, expected_items)
        self.bit_array = bitarray(self.size)
        self.bit_array.setall(0)
        self._inserted = 0

    @staticmethod
    def _optimal_size(n: int, p: float) -> int:
        """计算最优位数组大小"""
        return int(-n * math.log(p) / (math.log(2) ** 2))

    @staticmethod
    def _optimal_hash_count(m: int, n: int) -> int:
        """计算最优哈希函数数量"""
        return int((m / n) * math.log(2))

    def _hashes(self, item: str) -> list[int]:
        """生成 k 个不同的哈希值"""
        result = []
        # 使用双重哈希技术生成多个哈希值
        h1 = int(hashlib.md5(item.encode()).hexdigest(), 16)
        h2 = int(hashlib.sha1(item.encode()).hexdigest(), 16)
        for i in range(self.hash_count):
            combined = (h1 + i * h2) % self.size
            result.append(combined)
        return result

    def add(self, item: str):
        """向过滤器中添加元素"""
        for h in self._hashes(item):
            self.bit_array[h] = 1
        self._inserted += 1

    def might_contain(self, item: str) -> bool:
        """检查元素可能是否在集合中（可能有误报）"""
        return all(self.bit_array[h] for h in self._hashes(item))

    @property
    def current_false_positive_rate(self) -> float:
        """估算当前的误报率"""
        if self.size == 0:
            return 1.0
        p = (1 - math.exp(-self.hash_count * self._inserted / self.size)) ** self.hash_count
        return p


# ---- Bloom Filter 应用示例 ----
class LargeScaleBlocklist:
    """
    大规模黑名单系统。
    
    使用 Bloom Filter 作为快速预过滤 + 精确集合作为确认。
    两级过滤设计：BF 快速排除，精确 Set 消除误报。
    """

    def __init__(self, name: str):
        self.name = name
        # 布隆过滤器：快速预过滤（可能有误报）
        # 预期 100 万条恶意域名，1% 误报率
        self.bloom = BloomFilter(expected_items=1_000_000, false_positive_rate=0.01)
        # 精确集合：消除误报（只存储确实被标记的条目）
        self.exact_set: set[str] = set()
        # 积极缓存：缓存最近通过的请求，避免重复查询
        self._pass_cache: dict[str, bool] = {}

    def add_blocked_item(self, item: str):
        """添加被禁止的项目"""
        self.bloom.add(item)
        self.exact_set.add(item)

    def batch_add(self, items: list[str]):
        """批量添加"""
        for item in items:
            self.add_blocked_item(item)

    def is_blocked(self, item: str) -> bool:
        """
        三级检查：
        1. 先查积极缓存（最快）
        2. 布隆过滤器预过滤（快速排除）
        3. 精确集合确认（消除误报）
        """
        # Level 1: 积极缓存
        if item in self._pass_cache:
            return self._pass_cache[item]

        # Level 2: Bloom Filter（快速路径：大概率不在集合中的元素被快速放过）
        if not self.bloom.might_contain(item):
            self._pass_cache[item] = False
            return False

        # Level 3: 精确集合（消除 Bloom Filter 的误报）
        blocked = item in self.exact_set
        self._pass_cache[item] = blocked
        return blocked

    def clear_cache(self):
        """定期清理积极缓存"""
        self._pass_cache.clear()

    @property
    def stats(self) -> dict:
        return {
            "bloom_size_bits": self.bloom.size,
            "bloom_hash_count": self.bloom.hash_count,
            "bloom_fpr": self.bloom.current_false_positive_rate,
            "exact_set_size": len(self.exact_set),
            "cache_size": len(self._pass_cache),
        }


# ---- 性能对比 ----
def benchmark_blocklist_strategies():
    """对比不同 blocklist 实现的性能"""
    import time
    
    N = 100_000
    
    # 准备测试数据：100,000 条恶意域名
    malicious = [f"evil{i}.malware.com" for i in range(N)]
    benign = [f"safe{i}.example.com" for i in range(N)]
    
    # 1. Python Set（基线）
    block_set = set(malicious)
    
    # 2. Bloom Filter + Set
    bf_blocklist = LargeScaleBlocklist("bloom")
    bf_blocklist.batch_add(malicious)
    
    # 测试查询（混合恶意和良性）
    queries = malicious[:1000] + benign[:1000]
    
    # Set 查询
    t0 = time.perf_counter()
    for q in queries:
        _ = q in block_set
    t1 = time.perf_counter()
    set_time = (t1 - t0) * 1000
    
    # Bloom Filter + Set 查询
    t2 = time.perf_counter()
    for q in queries:
        _ = bf_blocklist.is_blocked(q)
    t3 = time.perf_counter()
    bf_time = (t3 - t2) * 1000
    
    print(f"Set query: {set_time:.2f} ms for {len(queries)} queries")
    print(f"Bloom+Set query: {bf_time:.2f} ms for {len(queries)} queries")
    print(f"Memory (Set): ~{N * 50 / 1024 / 1024:.2f} MB (estimated)")
    print(f"Memory (Bloom): ~{bf_blocklist.bloom.size / 8 / 1024 / 1024:.2f} MB + set")
```

**布隆过滤器适用场景**：
- 大规模恶意域名/URL 列表（数百万级别）
- 已知的攻击 payload 签名库
- 作为第一层快速过滤器，减少精确匹配压力

### 3. Cache-Compiled Patterns（缓存编译后的模式）

正则表达式的编译是昂贵的操作。在频繁调用的场景下，预编译并缓存模式可以大幅提升性能。

```python
import re
from functools import lru_cache
from typing import Pattern


class CompiledPatternCache:
    """
    正则表达式编译缓存。
    
    在 Agent 安全验证中，同一批模式会被重复用于每次工具调用。
    预编译可以节省大量的重复编译开销。
    """

    def __init__(self, max_size: int = 1000):
        self.max_size = max_size
        self._cache: dict[str, Pattern] = {}

    def compile(self, pattern: str, flags: int = 0) -> Pattern:
        """获取或编译正则表达式"""
        key = f"{pattern}:{flags}"
        if key not in self._cache:
            if len(self._cache) >= self.max_size:
                # LRU：移除最旧的条目
                self._cache.pop(next(iter(self._cache)))
            self._cache[key] = re.compile(pattern, flags)
        return self._cache[key]

    def match_any(self, patterns: list[str], value: str) -> tuple[bool, str]:
        """
        检查值是否匹配任意一个模式。
        返回 (是否匹配, 匹配到的模式)
        """
        for pattern in patterns:
            compiled = self.compile(pattern)
            if compiled.search(value):
                return True, pattern
        return False, ""

    def clear(self):
        self._cache.clear()


class OptimizedValidator:
    """
    工程优化后的验证器。
    
    优化要点：
    1. 预编译所有正则模式
    2. 使用 LRU 缓存热门模式
    3. 优先使用简单的字符串匹配（先精确后正则）
    4. 短路评估（一旦命中拒绝规则立刻返回）
    """

    def __init__(self):
        self.pattern_cache = CompiledPatternCache(max_size=5000)
        
        # 规则按类型分组，优先检查计算简单的规则
        self.exact_block_set: set[str] = set()
        self.prefix_block_list: list[str] = []
        self.regex_block_patterns: list[str] = []
        
        # 统计
        self._stats = {"checks": 0, "exact_hits": 0, "prefix_hits": 0, "regex_hits": 0}

    def add_exact_block(self, item: str):
        """添加精确匹配规则（O(1) 检查）"""
        self.exact_block_set.add(item)

    def add_prefix_block(self, prefix: str):
        """添加前缀匹配规则"""
        self.prefix_block_list.append(prefix)

    def add_regex_block(self, pattern: str):
        """添加正则匹配规则"""
        self.regex_block_patterns.append(pattern)

    def quick_check(self, value: str) -> bool:
        """
        优化的多层检查。
        
        检查顺序（按计算成本递增）：
        1. 精确匹配（Set lookup, O(1)）
        2. 前缀匹配（startswith, O(L)）
        3. 正则匹配（编译后 regex, O(L) 但常量更大）
        """
        self._stats["checks"] += 1
        
        # Level 1: Exact match (cheapest)
        if value in self.exact_block_set:
            self._stats["exact_hits"] += 1
            return False
        
        # Level 2: Prefix match (cheap)
        for prefix in self.prefix_block_list:
            if value.startswith(prefix):
                self._stats["prefix_hits"] += 1
                return False
        
        # Level 3: Regex match (more expensive)
        for pattern in self.regex_block_patterns:
            compiled = self.pattern_cache.compile(pattern)
            if compiled.search(value):
                self._stats["regex_hits"] += 1
                return False
        
        return True

    def get_stats(self) -> dict:
        return {**self._stats, "cache_size": len(self.pattern_cache._cache)}
```

### 4. 更多工程优化策略

| 优化技术 | 适用场景 | 效果 |
|---------|---------|------|
| **AHO-CORASICK 自动机** | 大量关键词匹配（关键词黑名单） | O(N + M + K)，N 为文本长度，M 为所有模式长度和 |
| **位图索引（BitMap）** | 枚举值 allowlist，值空间连续 | 每个值 1 bit，内存极小 |
| **分片并行检查** | 超大规模规则集 | 规则分片到多个 CPU 核心并行匹配 |
| **冷热分离** | 高频率的规则查询 | 热规则在内存，冷规则在 SSD |
| **SIMD 加速** | 需要极致性能的场景 | 单指令多数据流并行匹配 |
| **Rust/C 扩展** | Python 中性能瓶颈的模式匹配 | 比纯 Python 快 10-100 倍 |

### 5. 性能基准（Benchmark）

| 规则数量 | 匹配方式 | 单次查询时间 | 内存占用 |
|---------|---------|------------|---------|
| 1,000 | Python Set (Exact) | ~0.05 us | ~50 KB |
| 10,000 | Trie (Glob/Prefix) | ~0.5 us | ~500 KB |
| 100,000 | Set (Exact) | ~0.05 us | ~5 MB |
| 100,000 | Bloom Filter (Approx) | ~2 us | ~1 MB |
| 100,000 | Aho-Corasick (Keywords) | ~1 us | ~10 MB |
| 1,000,000 | Bloom Filter | ~2 us | ~10 MB |
| 1,000,000 | Set (Exact) | ~0.05 us | ~50 MB |
| 10,000 | Regex (compiled) | ~1-10 us | ~1 MB |
| 10,000 | Regex (uncompiled=每次编译) | ~100-1000 us | N/A |

> **关键结论**：对于 Agent 安全场景，大多数情况下 allowlist 规则集在 100-10,000 级别。此时 Python Set（精确匹配）和 Trie（模式匹配）是最优选择。只有在处理超过 100,000 条规则的 blocklist 时，才有必要引入 Bloom Filter 等高级数据结构。

---

*本文件所属模块：12.3 Input Validation / 12 Safety / AI Agent Safety*
