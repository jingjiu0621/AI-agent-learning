# 9.5.2 code-indexing — 代码索引：AST / 符号表 / 调用图

## 简单介绍

代码索引（Code Indexing）是 Code RAG 的核心基础设施——将源代码转换为**可高效检索的多维结构化索引**。不同于普通文档索引只做 Embedding + 倒排索引，代码索引需要同时构建三种视图：**AST 索引**（语法结构）、**符号表索引**（定义/引用关系）、**调用图/依赖图索引**（函数间/模块间关系）。

## 基本原理

代码索引的三大支柱：

### 1. AST 索引（Abstract Syntax Tree Index）

将代码解析为 AST 并将每个节点类型作为可检索维度：

```
AST 索引结构:

File: user_service.py
├── ImportFrom "flask" → "Flask"
├── ClassDef "UserService"
│   ├── FunctionDef "__init__" (line 5-8)
│   │   └── parameters: [self, db]
│   └── FunctionDef "get_user" (line 10-20)
│       ├── parameters: [self, user_id]
│       ├── Call "query_db" (line 15)
│       └── Return "User" (line 19)
└── FunctionDef "create_user" (line 22-35)
    └── ...
```

**索引内容**：每条 AST 节点记录节点类型、文本范围、父节点引用、子节点列表

```python
# AST 索引的 Schema
ASTIndexEntry = {
    "type": "function_definition",        # 节点类型
    "name": "get_user",                   # 节点名称
    "file": "user_service.py",            # 源文件
    "start_line": 10,
    "end_line": 20,
    "parent": "UserService",              # 父节点（类名）
    "signature": "(self, user_id: int)",  # 函数签名
    "docstring": "Get user by ID",        # 文档字符串
    "body_preview": "return self.db.query(User).get(user_id)",  # 正文摘要
}
```

### 2. 符号表索引（Symbol Table Index）

记录每个符号（变量、函数、类、模块）的**定义位置**和**所有引用位置**：

```
符号表索引:

符号 "User" (类)
  ├── 定义于: models.py:42
  ├── 引用:
  │   ├── user_service.py:15  (类型注解)
  │   ├── user_service.py:19  (return 类型)
  │   ├── auth_service.py:88  (import)
  │   └── test_user.py:55     (实例化)
  └── 继承自: "BaseModel" → "Base" → "object"

符号 "query_db" (函数)
  ├── 定义于: db.py:120
  ├── 引用:
  │   ├── user_service.py:15
  │   ├── order_service.py:45
  │   └── product_service.py:67
  └── 调用: "session.query" (db.py:121)
```

```python
# 符号表索引构建
def build_symbol_index(source_files: list[str]):
    symbol_index = {}  # symbol_name → {definitions, references}
    
    for file_path in source_files:
        ast = parse_to_ast(file_path)
        # 第一次遍历：收集所有定义
        for def_node in find_definitions(ast):
            symbol_index[def_node.name] = {
                'definition': {
                    'file': file_path,
                    'line': def_node.line,
                    'type': def_node.type_desc()
                },
                'references': []
            }
        # 第二次遍历：收集引用
        for ref_node in find_references(ast):
            name = ref_node.name
            if name in symbol_index:
                symbol_index[name]['references'].append({
                    'file': file_path,
                    'line': ref_node.line,
                    'context': ref_node.context_snippet()
                })
    
    return symbol_index
```

### 3. 调用图/依赖图索引（Call Graph & Dependency Graph Index）

记录函数之间的调用关系和模块之间的依赖关系：

```
调用图示例:

create_user() ──────────→ validate_email()
     │                        │
     ├──→ hash_password()     │
     │         │              │
     │         └──→ bcrypt.hash()
     │
     └──→ db.session.add()
              │
              └──→ db.session.commit()

依赖图（模块级）:

user_service.py ──→ models.py (import User)
     │                  │
     │                  ├──→ base.py (import Base)
     │                  │
     │                  └──→ db.py (import session)
     │
     └──→ utils.py (import validate_email, hash_password)
```

```python
# 调用图构建
def build_call_graph(asts: dict[str, AST]) -> nx.DiGraph:
    G = nx.DiGraph()
    
    for file_path, ast in asts.items():
        for func_def in ast.find_all(FunctionDef):
            caller = f"{file_path}::{func_def.name}"
            for call in func_def.find_all(Call):
                callee = resolve_call_target(call, file_path)
                if callee:
                    G.add_edge(caller, callee, 
                                type='call',
                                line=call.line)
    
    return G
```

## 背景：为什么需要多维度索引

早期代码检索只做单一维度的全文搜索。但代码的特殊性在于：

- **相同的标识符可能有完全不同的含义**（全局变量 vs 局部变量名相同）
- **代码的语义部分由结构决定**（`obj.method()` 的含义依赖于 `obj` 的类型）
- **跨文件关系无法通过文本匹配解决**（需要理解 import 和作用域解析）

## 之前是怎么做的

| 时代 | 索引方法 | 检索方式 | 局限 |
|------|---------|---------|------|
| grep 时代 | 无索引 | 逐行正则匹配 | 慢、无语义 |
| ctags/cscope | 符号表 | 跳转到定义 | 不支持语义搜索 |
| LSP 时代 | 语言服务器索引 | 精确查询 | 不支持模糊搜索 |
| 语义搜索 | Embedding 索引 | 向量相似度 | 丢失结构信息 |
| 多维索引 | AST+符号+调用图 | 混合检索 | 实现复杂 |

## 核心矛盾

**单一索引维度无法满足所有查询类型**：

| 查询示例 | 最适合的索引 |
|---------|------------|
| "这个函数做了什么？" | AST + 文档字符串 |
| "谁调用了这个函数？" | 调用图 |
| "这个变量在哪里定义的？" | 符号表 |
| "类似模式的代码在哪里？" | 语义向量 |
| "这个模块导入了什么？" | 依赖图 |

需要**多路检索 + 结果融合**来综合各维度的优势。

## 主流优化方向

### 1. 统一索引框架

将三种索引统一在同一个存储后端（如 PostgreSQL + pgvector + 关系表）：

```python
class UnifiedCodeIndex:
    def __init__(self):
        self.vector_store = PGVector()      # 语义向量索引
        self.symbol_table = SymbolTable()   # 符号表（关系型）
        self.graph_store = NetworkXGraph()  # 调用图/依赖图
    
    def query(self, query: str, intent: str) -> list[CodeContext]:
        # 根据查询意图选择索引组合
        if intent == 'explain':
            return self._vector_search(query, top_k=5)
        elif intent == 'find_callers':
            return self._graph_search(query, direction='incoming')
        elif intent == 'find_definition':
            return self._symbol_lookup(query)
        elif intent == 'find_similar_bugs':
            return self._hybrid_search(query, weight_vector=0.7, weight_graph=0.3)
```

### 2. 增量索引更新

监听文件变化，只重新索引变更的文件，使用 Tree-sitter 的增量解析能力：

```python
class IncrementalIndex:
    def __init__(self, root_dir: str):
        self.watcher = FileWatcher(root_dir)
        self.index = CodeIndex()
    
    def on_file_changed(self, file_path: str):
        # 只移除旧文件的索引
        self.index.remove_file(file_path)
        # 重新解析并索引
        ast = parse_file(file_path)
        new_entries = self.index_file(file_path, ast)
        # 更新受影响的跨文件引用
        self.update_cross_references(file_path, new_entries)
```

### 3. 层级摘要索引

为大型代码库构建层级摘要，支持"先概览后深入"：

```
Repository Level: "E-commerce backend — 50k lines, 200 files"
    ├── Module: auth/ — "用户认证模块，JWT + OAuth2"
    │   ├── File: auth_service.py — "认证逻辑入口"
    │   └── File: token_manager.py — "Token 生成与验证"
    ├── Module: payment/ — "支付处理，Stripe + PayPal"
    └── Module: inventory/ — "库存管理，实时同步"
```

## 与其他索引方法对比

| 维度 | 传统文本索引 | 代码多维索引 |
|------|------------|------------|
| 存储模型 | 倒排表 + 向量 | 向量 + 图 + 关系表 |
| 查询类型 | 关键词/语义 | 结构/关系/语义混合 |
| 精度 | 中 | 高 |
| 构建成本 | 低 | 中-高 |
| 维护成本 | 低 | 中 |

## 最大挑战

1. **跨语言引用解析**：monorepo 中 Python 调用 C++ 扩展，需要跨语言的符号解析
2. **动态语言符号解析**：Python 的 `getattr`、`setattr`、猴子补丁等动态特性使得静态符号表永远不完整
3. **索引一致性**：当文件发生变化时，如何保证所有索引（AST/符号/图）都同步更新
4. **存储规模**：大型 monorepo（百万文件级别）的图索引可能达到数亿节点

## 能力边界

- ✅ 精确解析标准语法（Java、Go、Rust、Python、TypeScript）
- ✅ 构建函数级/类级的调用图
- ✅ 支持增量索引和实时更新
- ❌ 宏展开（Rust macro、C++ template）后的符号无法完全解析
- ❌ 反射/动态派发的调用图不完整
- ❌ eval 执行生成的代码不可索引

## 工程优化方向

1. **懒加载索引**：只索引常用模块，大型历史代码按需加载
2. **索引压缩**：AST 节点使用 protobuf/FlatBuffers 编码减少存储
3. **预计算 transitive closure**：为常用查询预计算传递闭包（如"所有间接调用函数"）
4. **分片索引**：按模块/领域分片存储，支持并行查询

## 推荐工具

- **Tree-sitter**: AST 解析，支持增量解析和 40+ 语言
- **rust-analyzer**: Rust 符号索引引擎
- **Pyright**: Python 类型推断和符号解析
- **sourcegraph/scip**: 精确代码索引（SCIP 格式）
- **NetworkX / iGraph**: 图索引存储与遍历
- **Apache Age / Neo4j**: 图数据库存储调用图
