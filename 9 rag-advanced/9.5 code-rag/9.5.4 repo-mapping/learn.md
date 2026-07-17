# 9.5.4 repo-mapping — 仓库映射：依赖图与文件关系

## 简单介绍

仓库映射（Repository Mapping / Repo Mapping）是指对整个代码仓库的结构、依赖关系、文件组织建立**全局视角**的技术。如果说代码分块关注"局部结构"（函数/类），代码索引关注"多维检索"，那么仓库映射关注的是**"全局拓扑"**——模块如何组织、文件如何依赖、数据如何流动。

## 基本原理

仓库映射的核心是构建三种图：

### 1. 模块依赖图（Module Dependency Graph）

```
src/
├── auth/              ← 被 user/、order/、api/ 依赖
│   ├── AuthService
│   └── TokenManager
├── payment/
│   ├── PaymentService  ← 被 order/ 依赖
│   └── StripeClient    ← 被 PaymentService 依赖
├── user/              ← 被 api/ 依赖
│   ├── UserService     ← 依赖 auth/
│   └── UserModel       ← 独立
├── order/             ← 被 api/ 依赖
│   ├── OrderService    ← 依赖 user/ + payment/
│   └── OrderModel
├── api/
│   ├── routes.py       ← 依赖所有 service
│   └── middleware.py   ← 依赖 auth/
└── shared/
    ├── types.py        ← 被所有模块依赖
    └── utils.py        ← 被大部分模块依赖
```

```python
# 构建模块依赖图
def build_module_dependency_graph(repo_path: str) -> nx.DiGraph:
    G = nx.DiGraph()
    
    for file_path in walk_python_files(repo_path):
        module = resolve_module_name(file_path, repo_path)
        G.add_node(module, file_path=file_path)
        
        for import_stmt in parse_imports(file_path):
            if not import_stmt.is_stdlib and not import_stmt.is_third_party:
                target_module = import_stmt.module
                G.add_edge(module, target_module, type='import')
    
    return G
```

### 2. 文件影响图（File Impact Graph）

记录了每个文件变更时可能影响的范围：

```python
def compute_file_impact(G: nx.DiGraph, file_path: str) -> set[str]:
    """计算修改 file_path 的直接影响范围"""
    changed_module = resolve_module(file_path)
    # 反向 BFS：所有直接或间接依赖此模块的文件
    impacted = set(nx.ancestors(G, changed_module))
    impacted.add(changed_module)
    return impacted
```

### 3. 数据流图（Data Flow Graph）

在类型系统层面追踪数据的产生、转换、消费：

```
数据流示例: "用户注册" 功能

register_user(email, password)
  │
  ├──→ validate_email(email)          # 验证
  │       └──→ return bool
  │
  ├──→ hash_password(password)        # 加密
  │       └──→ return hashed_pw
  │
  ├──→ User.create(email, hashed_pw)  # 持久化
  │       └──→ return User
  │
  └──→ send_welcome_email(user)       # 通知
          └──→ return bool
```

## 背景：为什么需要仓库映射

随着代码规模的增长，开发者面临"**只见树木不见森林**"的问题：

- 单个文件看懂了，但不知道它和整个系统的关系
- 修改一个基础类型（如 `User` 模型），不知道哪些文件会受影响
- 新功能开发不知道应该放在哪个模块、依赖哪些现有服务

## 之前是怎么做的

| 方法 | 工具/实践 | 问题 |
|------|----------|------|
| 人工文档 | README、架构图 | 容易过时 |
| 代码审查 | 人工评审 | 依赖 reviewers 的经验 |
| 静态分析 | SonarQube、CodeClimate | 只分析局部质量，不分析全局关系 |
| 文件搜索 | grep 查找引用 | 效率低，漏检多 |

**核心矛盾**：代码库的**实际架构**（代码依赖揭示的）和**理想架构**（文档描述的）之间存在差距（Architecture Gap），且差距随代码演进持续扩大。

## 主流优化方向

### 1. 自动仓库地图生成

```python
class RepoMapper:
    def __init__(self, repo_path: str):
        self.repo_path = repo_path
        self.modules = {}       # module_name → ModuleInfo
        self.dependencies = nx.DiGraph()
        self.entry_points = []  # 主入口文件
    
    def analyze(self):
        # Phase 1: 扫描文件，识别模块边界
        self._scan_modules()
        # Phase 2: 构建依赖图
        self._build_dependency_graph()
        # Phase 3: 计算度量指标
        self._compute_metrics()
        # Phase 4: 生成地图摘要
        return self._generate_map_summary()
    
    def _generate_map_summary(self):
        """生成供 LLM 使用的仓库地图摘要"""
        summary = []
        
        for module_name, info in sorted(self.modules.items()):
            depends_on = list(self.dependencies.successors(module_name))
            depended_by = list(self.dependencies.predecessors(module_name))
            
            summary.append(f"""
Module: {module_name}
  Files: {info['file_count']}
  Depends on: {', '.join(depends_on[:5]) or '(none)'}
  Used by: {', '.join(depended_by[:5]) or '(none)'}
  Entry points: {', '.join(info['entry_points'][:3])}
            """)
        
        return '\n'.join(summary)
```

### 2. 语义化的仓库摘要

为 LLM 生成可理解的仓库全局视图：

```
Repository: "E-Commerce Backend" (Python/FastAPI)
├── 📁 src/api — REST API 层 (5 文件)
│   ├── routes.py — 所有 API 路由注册
│   ├── middleware.py — 认证中间件 [auth] + 日志
│   └── schemas.py — Pydantic 请求/响应模型
├── 📁 src/service — 业务逻辑层 (8 文件)
│   ├── auth_service.py — 用户认证、JWT 管理 [critical]
│   ├── order_service.py — 订单创建/支付/退款
│   └── inventory_service.py — 库存锁/更新/回滚
├── 📁 src/models — 数据模型层 (4 文件)
│   ├── user.py — User ORM 模型
│   └── order.py — Order, OrderItem ORM 模型
├── 📁 src/integrations — 外部服务集成 (3 文件)
│   ├── stripe.py — Stripe 支付客户端
│   └── sendgrid.py — 邮件发送
└── 📁 tests — 测试目录 (12 文件)
    ├── conftest.py — 测试夹具配置
    └── test_*.py — 各模块测试
```

### 3. 变更影响范围预测

当开发者修改一个文件时，自动预测受影响的范围：

```python
def predict_impact_zone(changed_files: list[str], 
                         repo_map: RepoMap) -> ImpactZone:
    """预测变更影响范围"""
    impacted_files = set()
    impacted_tests = set()
    
    for f in changed_files:
        module = resolve_module(f)
        # 受影响的消费者
        consumers = nx.ancestors(repo_map.graph, module)
        impacted_files.update(consumers)
        
        # 相关测试
        for consumer in consumers:
            test_file = find_test_file(consumer)
            if test_file:
                impacted_tests.add(test_file)
        
        # 传递性依赖中的高内聚模块
        transitive = nx.descendants_at_distance(repo_map.graph, module, 2)
        impacted_files.update(transitive)
    
    return ImpactZone(
        primary=impacted_files,
        tests=list(impacted_tests),
        risk_score=compute_risk_score(changed_files, repo_map)
    )
```

## 与其他技术的区别

| 维度 | 普通代码搜索 | 仓库映射 | 代码索引 |
|------|------------|---------|---------|
| 关注点 | 找到具体的代码 | 理解全局结构 | 多维检索 |
| 输出 | 代码片段列表 | 模块关系图 + 摘要 | 索引条目 |
| 粒度 | 函数/行 | 模块/目录 | 函数/符号 |
| 使用场景 | 编码时的即时查询 | 规划/重构/评审 | 离线检索 |

## 最大挑战

1. **依赖循环检测**：大型仓库中模块间可能形成循环依赖，导致影响分析不准确
2. **条件依赖**：`if config.USE_REDIS: import redis_cache` 这类条件导入难以静态分析
3. **动态模块加载**：Python 的 `__import__()`、importlib.import_module() 等动态加载
4. **monorepo 规模**：数十万文件的仓库，依赖图达到百万节点级别

## 能力边界

- ✅ 生成模块级依赖图（import 关系）
- ✅ 预测单文件变更的影响范围
- ✅ 自动生成仓库结构化摘要
- ❌ 精确的运行时调用路径（需要实际执行）
- ❌ 配置驱动的影响分析（配置文件修改的影响难以预测）
- ❌ 外部 API 变更的影响（第三方服务变化不可静态分析）

## 工程优化

1. **增量图更新**：文件变更时只更新受影响的子图
2. **模块聚合**：将紧密耦合的文件聚合为"领域模块"，降低图复杂度
3. **可视化**：生成可交互的依赖图（D3.js、Cytoscape.js）
4. **缓存影响分析**：常见的变更模式的预计算结果缓存

## 推荐工具

- **dep-tree**: 命令行依赖树分析工具
- **pydeps**: Python 依赖可视化
- **Madge**: Node.js 模块依赖图
- **Bazel Query**: Bazel 构建系统的依赖查询
- **Sourcegraph**: 仓库级代码导航
- **github/github-code-search**: GitHub 的代码搜索基础设施
