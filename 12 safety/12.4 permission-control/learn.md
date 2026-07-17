# 12.4 permission-control — 权限控制

**Agent 的权限越大，潜在危害就越大。** 权限控制是 Agent 安全的核心——限制了 Agent "能做什么"，即使 Prompt 注入成功，权限不足的攻击者也无法造成严重破坏。

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|----------|------|
| 12.4.1 Tool Authorization | 如何控制 Agent 对工具的授权访问？ | ★★★★☆ |
| 12.4.2 Least Privilege | 如何实现最小权限原则？ | ★★★★☆ |
| 12.4.3 Dynamic Permissions | 如何实现上下文感知的动态权限？ | ★★★★★ |
| 12.4.4 Approval Workflow | 高风险操作的审批流程怎么设计？ | ★★★☆☆ |
| 12.4.5 Audit Trail | 如何记录权限审计轨迹？ | ★★★☆☆ |
| 12.4.6 Isolation Boundary | 如何实现隔离边界（沙箱/容器）？ | ★★★★☆ |
| 12.4.7 Human-in-the-Loop | 如何在流程中嵌入人工确认环节？ | ★★★☆☆ |

## Agent 权限模型 vs 传统权限模型

```
传统权限（静态）                    Agent 权限（动态）
─────────────────                  ─────────────────
• 用户登录时分配角色                 • 每次对话动态评估权限
• 角色权限相对固定                   • 基于上下文、意图、风险动态调整
• 权限粒度：文件/API/DB             • 权限粒度：工具/参数/数据/执行
• 权限判定：静态 ACL               • 权限判定：策略 + 运行时评估
• 权限变更需要管理员                 • 权限可随对话上下文自动升级/降级
• 权限通常不过期                    • 权限随对话结束自动回收
```

## 权限控制架构

```
用户输入
   │
   ▼
┌─────────────────────┐
│  权限评估引擎          │
│                      │
│  ┌──────────────┐   │
│  │ 身份认证      │   │ ← 用户身份、会话 Token
│  └──────┬───────┘   │
│         ▼           │
│  ┌──────────────┐   │
│  │ 上下文分析    │   │ ← 对话历史、当前任务、风险等级
│  └──────┬───────┘   │
│         ▼           │
│  ┌──────────────┐   │
│  │ 策略决策      │   │ ← 权限策略 + 运行时规则
│  └──────┬───────┘   │
│         ▼           │
│  ┌──────────────┐   │
│  │ 权限分配      │   │ ← 授予/拒绝/限制/升级
│  └──────┬───────┘   │
└─────────┼───────────┘
          ▼
    ┌──────────┐
    │ Agent 执行│─── 在分配权限范围内操作
    └──────────┘
```

## 最小权限原则在 Agent 中的实现

```
传统最小权限                          Agent 最小权限
─────────────────                    ─────────────────
用户只需要读文件 → 给 read 权限        Agent 只需要查天气 → 只给 weather API 权限
                                      Agent 不需要发邮件 → 不给 email 权限
                                      需要临时发邮件 → 临时提升 + 自动回收
```

### 工具权限分级

| 级别 | 示例 | 权限控制 | 审批要求 |
|------|------|---------|---------|
| L0: 只读 | 搜索、查天气、读文件 | 自动授予 | 无需审批 |
| L1: 低风险写 | 写笔记、创建草稿 | 授权后执行 | 无需审批 |
| L2: 中风险 | 发邮件、修改文件 | 需上下文验证 | 可选确认 |
| L3: 高风险 | 删除文件、转账、修改配置 | 需明确授权 | 必须确认 |
| L4: 危险 | 执行 shell、修改系统 | 默认拒绝 | 管理员审批 |

## 权限控制核心策略

```python
class PermissionController:
    """权限控制器——基于策略的 Agent 权限管理"""

    def __init__(self):
        self.policy_engine = PolicyEngine()
        self.context_analyzer = ContextAnalyzer()
        self.approval_manager = ApprovalManager()

    async def check_permission(self, tool_call: ToolCall, context: SessionContext) -> Permission:
        # 1. 基础策略检查
        base_permission = self.policy_engine.evaluate(tool_call, context.user)

        # 2. 上下文风险评估
        risk_score = self.context_analyzer.assess_risk(tool_call, context)

        # 3. 动态调整
        effective_permission = self._apply_dynamic_rules(base_permission, risk_score)

        if effective_permission == Permission.DENY:
            return PermissionResult.DENY("策略禁止此操作")

        if effective_permission == Permission.REQUIRE_APPROVAL:
            approved = await self.approval_manager.request_approval(
                user=context.user,
                tool_call=tool_call,
                risk_score=risk_score
            )
            if not approved:
                return PermissionResult.DENY("审批未通过")

        return PermissionResult.ALLOW(effective_permission)
```

## 关键认知

- **权限是安全的最后防线**：所有其他防御（注入防护、输出护栏）失效时，权限限制仍然有效
- **最小权限是最有效的单一安全措施**：即使 Agent 被完全劫持，受限的权限也能限制损害范围
- **权限需要动态评估**：静态权限对 Agent 来说太僵化，需要根据上下文动态调整
- **权限粒度决定安全效果**：工具级权限不够，需要参数级、数据级权限
- **权限必须有审计**：每次权限判定都需要记录，用于事后分析和异常检测
