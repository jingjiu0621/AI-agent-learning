# 13.1.6 blue-green-canary — 蓝绿部署与金丝雀发布

**Agent 的版本更新比传统服务风险更高：新版本可能看起来"正常响应"，但回答质量下降、工具调用出错或行为出现有害偏差。** 蓝绿部署和金丝雀发布是两类互补的安全发布策略——前者通过瞬间切换降低部署风险，后者通过渐进式流量迁移验证新版本质量。对于 Agent 系统，这两种策略都需要加入**语义质量验证**这一传统部署策略中没有的维度。

## 背景与问题

### 传统部署策略的不足

传统部署方式（滚动更新）的核心问题：

```
滚动更新 (Rolling Update) 的问题：
────────────────────────────────────
1. 新旧版本共存期间，同一用户的请求可能落到不同版本
   → 用户感知到行为不一致："刚才还能用，现在怎么不行了？"

2. 回滚缓慢：需要逐个 Pod 恢复旧版本
   → 如果新版本有严重 Bug，数分钟的恢复时间影响巨大

3. 无法控制流量比例
   → 无法先让 5% 用户体验新版

4. Agent 的特殊问题：
   → 新版本"看起来"工作正常（API 返回 200），但回答质量下降
   → 传统健康检查（HTTP 探活）无法捕获语义退化
```

### Agent 版本更新的特有风险

与普通 Web 应用不同，Agent 的新版本可能在以下维度引入风险：

```
风险维度                 举例
─────────                ────
回答准确性退化            新 System Prompt 让 Agent 更爱捏造事实

工具选择错误              新版本误将"搜索"理解为"发送邮件"

行为偏移                  新版本回复更冗长或更简短

成本变化                  新版本多走了一步不必要的推理

安全漏洞                  新版本更容易被 Prompt 注入

记忆干扰                  新版本记忆读取逻辑变化导致检索到错误记忆
```

### 之前是怎么做的？

1. **直接替换部署**：停止旧版本 → 启动新版本（停机部署）
2. **滚动更新**：逐个替换实例（K8s 默认策略）
3. **手动金丝雀**：手动维护两个版本，手动切流量（运维负担重）

## 蓝绿部署 (Blue-Green Deployment)

### 核心原理

同时维护两个完全相同的生产环境（蓝环境 + 绿环境），任一时刻只有一个接收生产流量。发布新版本时，部署到空闲环境，验证通过后一键切换流量。

```
初始状态:             蓝 = v1.0 (活跃)   绿 = v1.0 (空闲)
                                   
发布 v2.0:            蓝 = v1.0 (活跃)   绿 = v2.0 (部署+验证)
                                   
切换:                 蓝 = v1.0 (空闲)   绿 = v2.0 (活跃) ← 流量切换
                                   
回滚 (如果需要):       蓝 = v1.0 (待命)   绿 = v2.0 (宕) ← 只需切回
```

### Agent 场景的实现

```python
# blue_green_deploy.py — Agent 蓝绿部署管理器
import asyncio
import json
import logging
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional

class Environment(Enum):
    BLUE = "blue"
    GREEN = "green"

@dataclass
class AgentVersion:
    version_id: str
    image_tag: str
    prompt_version: str
    model_name: str
    config_hash: str
    deployed_at: str
    status: str = "pending"  # pending → validating → active → deprecated

class BlueGreenManager:
    """Agent 蓝绿部署管理器"""
    
    def __init__(self, agent_service, eval_service):
        self.agent_service = agent_service  # Agent 服务实例
        self.eval_service = eval_service    # 评估服务
        self.active = Environment.BLUE
        self.versions = {
            Environment.BLUE: None,
            Environment.GREEN: None,
        }
        
    async def deploy(self, new_version: AgentVersion) -> bool:
        """执行蓝绿部署"""
        # 1. 确定目标环境（当前空闲的）
        target = Environment.GREEN if self.active == Environment.BLUE else Environment.BLUE
        
        # 2. 部署新版本到空闲环境
        logging.info(f"Deploying {new_version.version_id} to {target.value}")
        await self.agent_service.deploy(target, new_version)
        
        # 3. 运行验证套件
        eval_result = await self._validate(target, new_version)
        if not eval_result["passed"]:
            logging.error(f"Validation failed: {eval_result}")
            await self.agent_service.rollback(target)
            return False
        
        # 4. 切换流量
        logging.info(f"Switching traffic to {target.value}")
        await self._switch_traffic(target)
        
        # 5. 旧版本保持待命（快速回滚用）
        old_active = self.active
        self.active = target
        self.versions[target] = new_version
        new_version.status = "active"
        
        # 6. 旧版本标记为待命（保持 30 分钟）
        old_version = self.versions[old_active]
        if old_version:
            old_version.status = "standby"
            asyncio.create_task(self._auto_cleanup(old_active, old_version, 1800))
        
        return True
    
    async def _validate(self, env: Environment, version: AgentVersion) -> dict:
        """Agent 特有的验证：不仅是 HTTP 健康检查，还有语义质量验证"""
        # Layer 1: 基础设施健康检查
        health = await self.agent_service.health_check(env)
        if not health["healthy"]:
            return {"passed": False, "reason": f"Health check failed: {health}"}
        
        # Layer 2: 工具调用功能验证
        tool_test = await self.agent_service.run_tool_test(env)
        if tool_test["failure_rate"] > 0.05:
            return {"passed": False, "reason": f"Tool test failure: {tool_test}"}
        
        # Layer 3: 语义质量验证（Agent 特有的）
        eval_result = await self.eval_service.run_smoke_suite(env)
        if eval_result["score"] < 0.85:
            return {"passed": False, "reason": f"Semantic eval below threshold: {eval_result}"}
        
        # Layer 4: 成本与性能基准
        perf = await self.agent_service.benchmark(env)
        if perf["p95_latency_ms"] > 5000:
            return {"passed": False, "reason": "Latency exceeds threshold"}
        if perf["cost_per_request"] > 0.10:
            return {"passed": False, "reason": "Cost exceeds budget"}
        
        return {"passed": True, "details": {
            "health": health,
            "tool_test": tool_test,
            "eval": eval_result,
            "performance": perf,
        }}
    
    async def rollback(self) -> bool:
        """回滚到上一个活跃版本"""
        # 只需切换流量，旧版本还在
        old = Environment.GREEN if self.active == Environment.BLUE else Environment.BLUE
        if not self.versions[old]:
            return False
        await self._switch_traffic(old)
        self.active = old
        return True
    
    async def _switch_traffic(self, target: Environment):
        """流量切换：负载均衡器 / API 网关配置更新"""
        # 实现取决于基础设施：K8s Service selector / Nginx upstream / LB 配置
        await self.agent_service.update_routing(
            active_env=target,
            standby_env=Environment.GREEN if target == Environment.BLUE else Environment.BLUE
        )
```

### Agent 蓝绿部署的验证层次

```
验证层次              检查内容                    失败后果
────────────────      ───────────                ──────────
L1: 基础设施          Pod 健康、端口监听、资源     无法响应请求
                     使用率                       (立刻发现)

L2: 功能链路          LLM API 可达、工具调用         部分功能失效
                     正常、记忆读写正常              (几分钟内发现)

L3: 语义质量          ≈ 回答准确率、工具选择         用户不满意但
                     准确率、拒绝率                 不报错(最难发现)

L4: 行为安全          注入攻击防御、拒绝回答          安全事件
                     有害内容过滤                   (可能延迟发现)

L5: 成本性能          延迟、Token 消耗、成本         成本暴涨
                                                    (月末才发现)
```

## 金丝雀发布 (Canary Release)

### 核心原理

与蓝绿的二选一切换不同，金丝雀发布让新版本逐步接收流量，在早期发现问题的同时限制爆炸半径。

```
时间 →         
           v1.0 (稳定)    v2.0 (金丝雀)
────────────────────────────────────
T+0        100%           0%        部署金丝雀
T+10min    99%            1%        1% 流量验证
T+30min    95%            5%        扩大验证
T+1h       90%            10%       监控指标
T+2h       50%            50%       对半
T+4h       0%             100%      全量
          (可随时回滚到任意步骤)
```

### Agent 特有的金丝雀策略

Agent 的金丝雀不能只按流量比例切分，还需要考虑**用户群的质量反馈**：

```python
# canary_release.py — Agent 金丝雀发布
class AgentCanary:
    """Agent 金丝雀发布管理器"""
    
    def __init__(self, router_service, monitor_service):
        self.router = router_service    # 流量路由器
        self.monitor = monitor_service  # 监控服务
        self.phases = [
            CanaryPhase(traffic=0.01, duration=600,  eval_threshold=0.90),   # 1% / 10min
            CanaryPhase(traffic=0.05, duration=1800, eval_threshold=0.90),   # 5% / 30min
            CanaryPhase(traffic=0.20, duration=3600, eval_threshold=0.85),   # 20% / 1h
            CanaryPhase(traffic=0.50, duration=3600, eval_threshold=0.85),   # 50% / 1h
            CanaryPhase(traffic=1.00, duration=0,    eval_threshold=0.0),    # 100%
        ]
        
    async def execute_canary(self, new_version: AgentVersion) -> bool:
        """执行多阶段金丝雀发布"""
        
        for i, phase in enumerate(self.phases):
            logging.info(f"Canary phase {i+1}/{len(self.phases)}: {phase.traffic*100:.0f}%")
            
            # 设置流量比例
            await self.router.set_traffic_split(
                stable_version=self.current_version,
                canary_version=new_version,
                canary_percentage=phase.traffic
            )
            
            if phase.duration > 0:
                # 等待观察期
                await asyncio.sleep(phase.duration)
                
                # 评估新版本质量
                metrics = await self.monitor.get_comparison_metrics(
                    baseline=self.current_version.version_id,
                    candidate=new_version.version_id
                )
                
                # 自动化决策
                decision = self._evaluate_phase(metrics, phase)
                
                if decision == "rollback":
                    await self._rollback_canary()
                    return False
                elif decision == "hold":
                    logging.warning(f"Phase {i+1} needs manual review, extending...")
                    await asyncio.sleep(1800)  # 额外等 30 分钟
                    metrics = await self.monitor.get_comparison_metrics(...)
                    if self._evaluate_phase(metrics, phase) == "rollback":
                        await self._rollback_canary()
                        return False
        
        # 全量发布
        await self.router.set_traffic_split(
            stable_version=self.current_version,
            canary_version=new_version,
            canary_percentage=1.0
        )
        return True
    
    def _evaluate_phase(self, metrics: dict, phase: CanaryPhase) -> str:
        """评估金丝雀阶段，决定继续/持有/回滚"""
        
        # Agent 特有指标
        eval_score = metrics.get("eval_score", 1.0)
        tool_success_rate = metrics.get("tool_success_rate", 1.0)
        cost_per_request = metrics.get("cost_per_request", 0)
        hallucination_rate = metrics.get("hallucination_rate", 0)
        
        # 自动回滚条件
        if eval_score < phase.eval_threshold:
            return "rollback"  # 语义质量下降
        if tool_success_rate < 0.95:
            return "rollback"  # 工具调用可靠性下降
        if cost_per_request > metrics.get("baseline_cost", 0) * 1.5:
            return "rollback"  # 成本暴涨超过 50%
        if hallucination_rate > 0.05:
            return "rollback"  # 幻觉率超过 5%
        
        return "continue"
```

### Agent 金丝雀的特殊指标

传统金丝雀关注延迟、错误率、CPU 使用率。Agent 金丝雀额外需要关注：

```yaml
# agent_canary_metrics.yaml
metrics:
  # 传统指标
  http_error_rate:
    threshold: 0.01  # 1% 错误率
  p95_latency_ms:
    threshold: 5000  # 5 秒
  cpu_usage:
    threshold: 0.8   # 80%
  
  # Agent 特有指标
  semantic_eval_score:
    threshold: 0.85          # 语义评估分数
    comparison: "degradation" # 相比基线下降不超过 0.05
  tool_call_success_rate:
    threshold: 0.95          # 工具调用成功率
  hallucination_rate:
    threshold: 0.03          # 幻觉率
  avg_steps_per_task:
    threshold: "±1"          # 平均推理步数相比基线变化 ≤ 1
  cost_per_conversation:
    threshold: "1.2x"        # 成本不超过基线的 1.2 倍
  user_feedback_score:
    threshold: 0.80          # 用户反馈评分 (thumbs up/down)
  
  # 安全指标
  prompt_injection_success_rate:
    threshold: 0.01          # Prompt 注入成功率
  refusal_rate_legit:
    threshold: 0.05          # 对合法请求的错误拒绝率
```

## 蓝绿 vs 金丝雀：如何选择

```
                蓝绿部署                           金丝雀发布
                ────────                           ────────
切换速度        瞬间切换 (~1s)                       逐步迁移 (分钟到小时)

回滚速度        瞬间回滚 (~1s)                       逐步回滚 (需要调整流量比例)

流量控制        全有或全无                            精细控制 (1% → 5% → 20% → ...)

基础设施成本    ×2 (双倍资源)                         1 + ε (少量额外资源)

验证深度        部署后统一验证                          每个阶段持续验证

用户体验        切换瞬间可能中断                       平滑过渡

适用场景        重大版本升级                          渐进式功能发布
               安全补丁                              模型版本升级
               需要快速回滚的场景                      Prompt 微调验证
                                                     A/B 对比测试

Agent 推荐        ⭐ 高风险发布                     ⭐ 常规版本更新
                (模型切换、架构重写)               (Prompt 优化、工具更新)
```

### 混合策略：蓝绿 + 金丝雀

在实际生产中，两者常结合使用：

```
蓝绿部署提供基础设施级别的快速切换保障
↓
金丝雀在活跃环境中做流量比例控制
↓
两者叠加实现：双层安全保障
```

```python
# 蓝绿 + 金丝雀混合
class HybridDeployStrategy:
    """蓝绿 + 金丝雀混合部署"""
    
    async def deploy(self, new_version):
        # 1. 部署到空闲环境（蓝绿的基础设施隔离）
        standby_env = self._get_standby_env()
        await self.deploy_to_env(standby_env, new_version)
        
        # 2. 在空闲环境运行深度验证
        assert await self._deep_validation(standby_env), "深度验证失败"
        
        # 3. 金丝雀切流（通过负载均衡器控制流量比例）
        #    流量从 active_env 逐步迁移到 standby_env
        for percentage in [0.01, 0.05, 0.20, 0.50, 1.0]:
            await self._set_traffic_split(
                active_env, standby_env, percentage
            )
            await self._monitor_and_evaluate(percentage)
        
        # 4. 全部切完后，旧环境变为待命
        self._mark_standby(active_env)
```

## 关键挑战与应对

### 挑战 1：Agent 会话连续性

金丝雀发布期间，同一用户的多次请求可能落在不同版本上，导致体验不一致。

```python
# 会话粘滞 (Session Affinity) 实现
class SessionAwareRouter:
    """基于用户会话的路由，保证同一用户始终路由到同一版本"""
    
    def __init__(self):
        self.session_map = {}  # user_id -> version_id
        
    def get_route(self, user_id: str, canary_version: str, 
                  canary_percentage: float) -> str:
        # 如果用户已经在金丝雀中，保持
        if user_id in self.session_map:
            return self.session_map[user_id]
        
        # 新用户根据比例分配
        if hash(user_id) % 10000 < canary_percentage * 10000:
            self.session_map[user_id] = canary_version
            return canary_version
        
        return "stable"
```

### 挑战 2：语义评估的自动化

金丝雀的瓶颈不是基础设施，而是**能否自动化判断新版本的回答质量**。

```
自动语义评估的两个关键问题：
1. 评估数据集：需要一组覆盖所有功能的测试用例
2. 评估标准：需要确定"可接受的质量下降幅度"

常见的评估数据集设计：
├── 回归测试集: 50-100 个覆盖核心功能的测试用例
├── 边界测试集: 20-30 个边界条件测试
└── 对抗测试集: 10-20 个安全/注入测试
```

### 挑战 3：回滚后的状态清理

Agent 回滚后，金丝雀期间产生的记忆和对话需要处理：

```python
class RollbackStateCleaner:
    """金丝雀回滚后的状态清理"""
    
    async def clean_canary_state(self, canary_version_id: str):
        # 标记金丝雀期间产生的记忆为"不信任"
        await self.memory_store.tag_by_version(
            version_id=canary_version_id,
            tag="rollback_untrusted"
        )
        
        # 金丝雀期间开始的对话标记回滚
        await self.conversation_store.mark_rollback(
            version_id=canary_version_id
        )
        
        # 不删除数据，保留审计轨迹
        await self.audit_log.record(
            action="canary_rollback_state_clean",
            version_id=canary_version_id
        )
```

## 工程优化方向

1. **金丝雀自动化决策**：基于历史数据训练决策模型，减少人工干预
2. **多渠道金丝雀**：按用户属性（地域、会员等级、设备类型）分层金丝雀
3. **影子模式 (Shadow Mode)**：新版本接收请求但响应不返回给用户，只对比质量
4. **渐进式 Prompt 上线**：Prompt 变更也走金丝雀流程，不直接全量
5. **自动归因**：回滚时自动分析是 Prompt 变更、模型变更还是工具变更导致的问题
6. **跨版本基线数据库**：积累每次发布的评估数据，建立质量基线趋势

## 适用场景判断

### 需要蓝绿部署的 Agent
- **核心业务 Agent**：宕机直接导致收入损失的 Agent（客服、交易）
- **大规模 Agent 平台**：多租户场景，影响面大
- **模型版本切换**：底层 LLM 模型升级（如 GPT-4 → GPT-5），不可控因素多

### 需要金丝雀发布的 Agent
- **频繁更新 Prompt**：Prompt 优化每周多次发布
- **A/B 测试场景**：需要对比不同 Prompt/工具的效果
- **新增工具/能力**：验证新工具不会影响已有功能
- **模型配置调整**：temperature、top_p 等参数调优

### 不需要的简单场景
- 个人项目 / 单用户 Agent：直接发布
- 内部一次性工具：停机部署即可
- 无状态、无用户 Agent（后台批处理）：滚动更新即可

蓝绿部署和金丝雀发布不是银弹——对于大多数 Agent 项目，一个简单的"部署 → 冒烟测试 → 全量发布"流程在前 6 个月已经足够。只有当 Agent 开始服务真实用户且对**回答质量有 SLA 承诺**时，这些策略才真正必要。
