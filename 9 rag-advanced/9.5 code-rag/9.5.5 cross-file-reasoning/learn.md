# 9.5.5 cross-file-reasoning — 跨文件推理与上下文聚合

## 简单介绍

跨文件推理（Cross-File Reasoning）是 Code RAG 中最具挑战性的问题——当回答一个问题或完成一个任务需要理解**分布在多个文件中的代码**时，如何将这些分散的信息片段正确地关联、聚合和组织，形成完整的上下文提供给 LLM。

## 基本原理

跨文件推理的底层逻辑：**代码库不是一个平面的文本集合，而是一个由引用关系连接的信息网络**。当需要理解一个函数时，不仅要看这个函数本身，还要看它调用/引用的其他文件中的定义。

```
跨文件推理链条示例:

问题: "用户注册接口是如何处理密码的？"

文件1: routes/auth.py           # 路由入口
  └── register_user() 调用了
      ├── auth_service.py       # 业务逻辑
      │   └── create_user() 调用了
      │       ├── validate_email() ← utils/validation.py
      │       ├── hash_password()  ← utils/crypto.py     ← 3层跨文件
      │       └── User.save()      ← models/user.py
      └── serializers.py         # 响应序列化
```

## 跨文件依赖的三种类型

### 1. 显式引用（Explicit References）

通过 import/require/include 明确声明的依赖：

```python
# user_service.py
from models.user import User                  # 显式导入
from utils.auth import hash_password         # 显式导入

class UserService:
    def create(self, data):
        user = User(**data)                    # 跨文件类型使用
        user.password = hash_password(pwd)     # 跨文件函数调用
        return user
```

### 2. 隐式引用（Implicit References）

通过框架约定、反射、依赖注入建立的引用：

```python
# 隐式引用示例
# 框架通过命名约定自动发现路由
class UserViewSet(ViewSet):                    # 继承自框架
    queryset = User.objects.all()              # ORM 映射（隐式引用数据库模式）
    serializer_class = UserSerializer           # 引用另一个文件中的类型

    # Django/Flask 的装饰器式路由
    @action(detail=True, methods=['post'])     
    def reset_password(self, request, pk=None):
        pass                                   # 路由注册是隐式的
```

### 3. 传递性引用（Transitive References）

A 引用 B，B 引用 C → 理解 A 可能需要 C：

```
传递性引用的典型传播链:

config.py → database.py → models/user.py → utils/encryption.py
    │            │              │                    │
    │            │              │                    └── AES 加密算法
    │            │              └── User 表结构定义
    │            └── 数据库连接池配置
    └── 环境变量读取

理解 user_service.py 中的 "User.save()" 需要:
  Level 1: user_service.py (调用方)
  Level 2: models/user.py → User.save() (方法实现)
  Level 3: database.py → session.commit() (事务提交)
  Level 4: config.py → DATABASE_URL (连接配置)
```

## 背景：为什么跨文件推理是核心难题

现代软件开发推崇"**关注点分离**"——一个功能往往分散在路由、业务逻辑、数据模型、外部集成等多个文件。但 LLM 的上下文窗口有限（即使有 200K token），也无法容纳整个仓库。

**核心矛盾**：**功能分散性**（一个功能跨 N 个文件）与 **上下文有限性**（LLM 只能看到 K 个 token）的矛盾。选择性聚合正确信息是关键。

## 之前是怎么做的

| 方法 | 描述 | 问题 |
|------|------|------|
| 展平整个仓库 | 将所有文件拼入 context | token 不够，噪声过多 |
| 只检索 Top-K 文件 | 按向量相似度取前 K 个 | 遗漏虽然不相似但必要的依赖文件 |
| 手动编写上下文 | 开发者自己找相关文件 | 不自动化，不可扩展 |
| RAG 单轮检索 | 查询→检索→回答 | 无法处理多跳依赖 |

## 主流优化方向

### 1. 图遍历上下文聚合（Graph-based Context Assembly）

```python
def assemble_context(seed_file: str, seed_line: int, 
                     repo: RepoIndex, max_tokens: int = 8000) -> Context:
    """从种子位置出发，沿依赖图逐步聚合上下文"""
    
    context = Context()
    visited = set()
    queue = [(seed_file, seed_line, 0)]  # (file, line, depth)
    
    while queue and context.token_count < max_tokens:
        file_path, line, depth = queue.pop(0)
        key = (file_path, line)
        if key in visited:
            continue
        visited.add(key)
        
        # 获取此位置的代码块
        block = repo.get_code_block(file_path, line)
        
        # 提取此块引用的外部符号
        refs = extract_references(block)
        
        # 将引用作为下一轮搜索的目标
        for ref in refs:
            target_file, target_line = resolve_reference(ref)
            if target_file and (target_file, target_line) not in visited:
                queue.append((target_file, target_line, depth + 1))
        
        # 按广度优先，深度控制（最多3层）
        if depth <= MAX_DEPTH:
            context.add_block(block, depth=depth)
        
        # 如果 token 预算不足，优先保留低 depth 的块
        context.prune_to_token_limit(max_tokens)
    
    return context
```

### 2. 分阶段上下文构建（Staged Context Building）

```python
def staged_context_assembly(query: str, repo: RepoIndex) -> str:
    """分阶段构建：先广后深，逐步聚焦"""
    
    # Stage 1: 种子检索（找到入口点）
    seeds = repo.semantic_search(query, top_k=5)
    
    # Stage 2: 广度展开（补充直接依赖）
    broad = set()
    for seed in seeds:
        direct_deps = repo.get_direct_dependencies(seed)
        broad.update(direct_deps)
    
    # Stage 3: 深度分析（评估哪些扩展文件真正重要）
    with LLM() as llm:
        relevance_scores = llm.judge_relevance(
            query=query,
            candidates=[*seeds, *broad],
            instruction="Rate each file's relevance to the query (0-10)"
        )
    
    # Stage 4: 最终上下文组装
    core_files = [f for f, s in zip(broad, relevance_scores) if s > 7]
    context = assemble_from_files(core_files, max_tokens=6000)
    
    return context
```

### 3. 依赖推理的查询改写

```python
def rewrite_query_with_dependencies(query: str) -> list[str]:
    """将用户查询改写为多步检索查询"""
    
    # 原始查询: "修复用户注册时的密码加密"
    decomposed = [
        "用户注册路由: routes/auth.py",           # 1. 找到注册入口
        "密码加密函数: 被注册接口调用",              # 2. 找到加密方法
        "用户模型: User model 定义",               # 3. 找到数据模型
        "数据库写入: 用户数据的存储",               # 4. 找到持久化逻辑
    ]
    return decomposed
```

### 4. 增量上下文注入

```python
class IncrementalCodeContext:
    """逐步注入，避免一次塞入过多无关上下文"""
    
    def __init__(self, max_tokens: int):
        self.max_tokens = max_tokens
        self.core_blocks = []    # 高优先级（种子文件）
        self.support_blocks = [] # 中优先级（一次引用）
        self.context_blocks = [] # 低优先级（二次引用）
    
    def finalize(self) -> str:
        tokens_used = 0
        final_parts = []
        
        for block in self.core_blocks:
            if tokens_used + block.tokens > self.max_tokens:
                break
            final_parts.append(block.text)
            tokens_used += block.tokens
        
        # 如果还有空间，补充 support 和 context
        for tier in [self.support_blocks, self.context_blocks]:
            for block in tier:
                if tokens_used + block.tokens > self.max_tokens:
                    break
                final_parts.append(block.text)
                tokens_used += block.tokens
        
        return '\n\n'.join(final_parts)
```

## 关键对比

| 策略 | 召回率 | 精度 | token 效率 | 延迟 |
|------|--------|------|-----------|------|
| 单文件检索 | 低 | 高 | 高 | 低 |
| 图遍历聚合 | 高 | 中 | 中 | 高 |
| 分阶段构建 | 高 | 高 | 高 | 中-高 |
| 展开全部依赖 | 最高 | 极低 | 极低 | 极高 |

## 最大挑战

1. **依赖爆炸**：一个核心文件可能被数十个文件引用，如果全部展开，token 预算迅速耗尽
2. **循环依赖**：A→B→C→A 的循环会在图遍历中造成死循环
3. **间接依赖的取舍**：第 3 层及以上的依赖是否重要？判断标准是什么？
4. **框架隐式依赖**：Django 的 ORM、Spring 的 DI——这些通过框架约定的依赖关系静态分析无法完全捕捉

## 能力边界

- ✅ 显式 import/require 引发的跨文件依赖追踪
- ✅ 广度优先 2-3 层内的依赖展开
- ✅ 基于符号解析的类型定义查找
- ❌ 框架约定的隐式路由/控制器映射
- ❌ 反射/动态加载的运行时依赖
- ❌ 编译期宏展开的跨文件引用

## 工程优化方向

1. **预计算文件亲密度**：计算文件间的"共同变更频率"（git log mining），发现隐式关联
2. **缓存聚合结果**：对同一文件/行的上下文需求缓存预计算结果
3. **优先级队列**：不再使用简单 BFS，而是使用基于重要性评分的优先级队列
4. **LLM 驱动的聚焦**：让 LLM 判断哪些额外文件是真需要的，而非机械展开

## 推荐工具

- **CodeGraph**: 代码图数据库，支持多跳查询
- **Sourcegraph Cody**: 跨文件上下文感知的代码 Agent
- **Tree-sitter + scope query**: 符号作用域和引用分析
- **git log mining**: 通过 git 历史发现文件间的隐式关联
