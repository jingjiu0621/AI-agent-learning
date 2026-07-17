# 9.5.1 code-chunking — 代码分块：函数 / 类 / 文件

## 简单介绍

代码分块（Code Chunking）是 Code RAG 的第一步——将源代码按语法结构切分为有意义的独立单元。不同于自然语言文本的按段落或按 token 数切分，代码分块需要尊重语言的**语法边界**，确保每个块是一个完整的语法单元（完整的函数、类、或顶层语句）。

## 基本原理

代码分块的核心原则：**每个块应当是一个自包含的、语义完整的代码单元**。实现的底层机制是：

1. **解析源代码为 AST**：使用 Tree-sitter 或语言特定 Parser 将代码解析为抽象语法树
2. **按 AST 节点类型遍历**：识别函数定义（FunctionDef）、类定义（ClassDef）、导入语句（Import）等顶级节点
3. **提取完整子树**：以每个顶级节点为根，提取其完整的子树文本作为块
4. **关联元数据**：为每个块附加文件路径、起始行号、结束行号、节点类型、作用域层级等元数据

```python
# 示例：基于 Tree-sitter 的代码分块
from tree_sitter import Language, Parser

def chunk_python_file(source_code: str):
    parser = Parser(Language('python'))
    tree = parser.parse(bytes(source_code, 'utf8'))
    root = tree.root_node
    
    chunks = []
    # 遍历顶级语句（函数定义、类定义、赋值语句等）
    for node in root.children:
        if node.type in ('function_definition', 'class_definition', 
                         'decorated_definition', 'module'):
            chunk_text = source_code[node.start_byte:node.end_byte]
            chunks.append({
                'type': node.type,
                'text': chunk_text,
                'start_line': node.start_point[0] + 1,
                'end_line': node.end_point[0] + 1,
                'name': extract_name(node)  # 函数名/类名
            })
    return chunks
```

## 背景：为什么需要专门的代码分块

早期代码检索系统（如代码搜索引擎）将代码视为纯文本，使用全文检索（如 Elasticsearch 的 ngram 分词）。但这种方法的问题：

- **语义完整性被破坏**：一个函数可能跨越 50 行，固定 100 token 的分块可能在函数中间截断
- **上下文碎片化**：同一个函数的头尾在不同块中，检索到的片段不完整
- **结构信息丢失**：无法区分"这段代码属于哪个类/方法"

## 之前是怎么做的

| 时期 | 方法 | 典型代表 |
|------|------|---------|
| 2010s 前 | 全文检索 + 正则匹配 | SourceForge 搜索 |
| 2015-2020 | 基于行的代码搜索 | 简单的 grep/ripgrep |
| 2020-2023 | 函数级分块 + BM25 | 早期代码搜索引擎 |
| 2023-2024 | Tree-sitter AST 分块 + 向量嵌入 | Code RAG 初版 |
| 2024-2025 | 多粒度分块 + 依赖感知上下文 | 生产级 Code RAG |

## 之前方法的结果与核心矛盾

纯文本分块的结果：
- **检索命中但不可用**：检索引用了函数片段，但缺少函数签名或返回值类型
- **召回率虚高但精度低**：匹配了局部变量名，但函数实际逻辑不相关
- **无法处理多语言**：每种语言的分块策略完全不同

**核心矛盾**：代码既有**线性的文本序列**属性，又有**层次的树状结构**属性。线性方法（n-gram、固定窗口）破坏结构，纯结构方法（AST）可能过于刚性，丢失代码的文本语义。

## 目前主流优化方向

### 1. 多粒度分块

不同场景需要不同粒度的代码块：

| 粒度 | 块大小 | 适用场景 | 示例 |
|------|--------|---------|------|
| 文件级 | 整个文件 | 理解整体结构 | 导入模块 |
| 类级 | 单个类 | OOP 上下文 | 类继承关系 |
| 方法/函数级 | 单个函数 | 功能实现 | 具体算法 |
| 代码块级 | 逻辑代码段 | 局部逻辑 | for 循环体 |
| 语句级 | 单行/多行 | 精确引用 | 变量声明 |

**典型策略**：建立**层次化分块索引**——同时索引文件、类、函数三个级别的块，检索时根据查询类型自动选择合适粒度。

### 2. AST 感知的智能分块

```python
def smart_chunk(source_code: str, language: str):
    """AST 感知的智能分块，保持结构完整性"""
    parser = get_parser(language)
    tree = parser.parse(bytes(source_code, 'utf8'))
    
    chunks = []
    
    # 1. 提取所有顶级定义（函数、类）
    for node in tree.root_node.named_children:
        if node.type in TOP_LEVEL_NODES[language]:
            chunks.append(extract_chunk(node, source_code))
    
    # 2. 对大型类，进一步按方法分块，但保留类级上下文引用
    for chunk in chunks:
        if chunk['type'] == 'class_definition':
            child_methods = split_class_into_methods(chunk)
            for method in child_methods:
                method['parent_class'] = chunk['name']
                method['parent_start'] = chunk['start_line']
            chunks.extend(child_methods)
    
    # 3. 为每个块附加依赖信息（导入、继承）
    for chunk in chunks:
        chunk['dependencies'] = extract_dependencies(chunk)
    
    return chunks
```

### 3. 边界重叠分块

对于超长函数（>500行），采用带重叠的滑动窗口，确保检索时不会遗漏：

```python
def overlapping_chunk(text: str, chunk_size: int = 200, overlap: int = 50):
    """带行级重叠的分块，保证长函数不被截断丢失上下文"""
    lines = text.split('\n')
    chunks = []
    start = 0
    while start < len(lines):
        end = min(start + chunk_size, len(lines))
        chunks.append({
            'text': '\n'.join(lines[start:end]),
            'start_line': start + 1,
            'end_line': end
        })
        start += chunk_size - overlap  # 50行重叠
    return chunks
```

### 4. Docstring/注释增强分块

将函数/类的文档字符串作为单独元数据附加到块上，增强检索语义匹配：

```
块内容: [函数代码]
块元数据:
  - name: "calculate_interest"
  - docstring: "计算复利，支持按月/年计息"
  - params: ["principal", "rate", "period", "type"]
  - return_type: "float"
  - dependencies: ["math.pow", "decimal.Decimal"]
```

## 与其他方法对比

| 分块方法 | 结构完整性 | 检索精度 | 实现复杂度 | 适用场景 |
|---------|-----------|---------|-----------|---------|
| 固定行数 | ❌ 低 | 低 | 极低 | 不推荐用于代码 |
| AST 节点分块 | ✅ 高 | 中 | 中 | 通用 Code RAG |
| 多粒度分块 | ✅ 高 | 高 | 高 | 生产系统 |
| LLM 语义分块 | ✅ 高 | 最高 | 极高 | 高价值场景 |

## 最大挑战

1. **语言兼容性**：每种语言有不同语法结构（Python 的 def/class vs Go 的 func/struct vs Rust 的 fn/impl），需要语言特定的 AST 解析器
2. **超长函数处理**：某些遗留系统有上千行的函数，分块策略需要在完整性和粒度之间权衡
3. **装饰器/注解**：Python 装饰器、Java 注解等元数据属于哪个块？应当附加到被装饰的函数
4. **内联代码**：JSX/TSX 中 HTML 和 JavaScript 混合，单一的 AST 策略不够

## 能力边界

- ✅ 能正确处理 90%+ 的标准代码结构（函数、类、方法）
- ✅ 能附加丰富的元数据（名称、文档、依赖）
- ❌ 无法处理动态语言中的运行时类型推断
- ❌ 宏（Macro）展开后的代码无法用静态分块覆盖
- ❌ 代码生成（Codegen）产物通常不适合分块索引

## 工程优化方向

1. **增量分块**：文件修改后只更新受影响的分块，无需重建全部索引
2. **并行解析**：多文件并行 AST 解析，利用多核 CPU
3. **缓存 AST**：频繁访问的文件缓存其 AST 以减少解析开销
4. **自适应粒度选择**：根据文件大小动态选择分块粒度（小文件→文件级，大文件→函数级）

## 适用场景判断

适合使用代码分块技术：
- 需要对大型代码库进行语义搜索
- 构建代码辅助 Agent（自动补全、代码审查）
- 代码迁移或重构时的上下文理解

不适合的场景：
- 纯文本配置文件的检索（YAML/JSON/TOML）
- 二进制或编译产物的理解
- 高度依赖运行时动态特性的代码

## 推荐工具

- **Tree-sitter**: 多语言增量解析库，支持 40+ 编程语言
- **Pyright/Pylance**: Python 类型检查器内置的 AST 分析能力
- **rust-analyzer**: Rust 的语义分析引擎，提供精确的语法树
- **semgrep**: 模式匹配工具，内置 AST 解析能力
