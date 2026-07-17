# 编程 Agent 架构 (Coding Agent Architecture)

## 1. 业务背景

### 1.1 从代码补全到自主编程 Agent 的演进

编程辅助工具经历了三个关键发展阶段：

```
  2018                   2021                  2023-2024               2025+
   |                      |                      |                       |
   v                      v                      v                       v
+--------+          +-----------+          +------------+          +-----------+
| 智能补全 |  -->    | 代码生成   |  -->     | 编程 Agent |   -->    | 自主开发  |
| TabNine |         | GitHub    |          | Cursor Tab |          | Agent    |
| Kite    |         | Copilot   |          | Devin      |          | (未来)   |
+--------+          +-----------+          +------------+          +-----------+
   |                      |                      |                       |
   行级补全              函数级补全            多文件编辑              全流程自主
   无上下文理解          token 预测            理解项目结构             需求到部署
```

| 阶段 | 代表产品 | 交互模式 | 上下文范围 | 用户角色 |
|------|---------|---------|-----------|---------|
| 智能补全 | TabNine, Kite | 被动触发 (tab 接受) | 当前文件 ~10 行 | 打字员 |
| 代码生成 | Copilot, Codex | 写注释 -> 生成代码 | 当前文件 + 邻近文件 | 审阅者 |
| 编程 Agent | Cursor Tab, Copilot Agent | 自然语言指令 -> 多步操作 | 全项目 + Git 历史 | 管理者 |
| 自主 Agent | Devin, SWE-agent | 任务描述 -> 完整 PR | 全仓库 + 终端 + 浏览器 | 需求方 |

### 1.2 核心能力的跃迁

编程 Agent 相比传统代码补全的核心能力变化：

1. **上下文感知范围**: 从单文件数行扩展到整个代码库 + Git 历史 + 依赖库
2. **操作边界**: 从插入文本扩展到读取文件、搜索符号、运行测试、执行命令、创建 PR
3. **自主程度**: 从被动响应补全请求到主动规划编辑步骤、验证结果、自我修正
4. **协作模式**: 从"人写机器补"到"人描述机器做"

---

## 2. 系统架构

### 2.1 整体架构概览

```
                          +--------------------------+
                          |      User Interface      |
                          |  (VSCode Extension /     |
                          |   Web IDE / CLI)         |
                          +-----------+--------------+
                                      |
                            Prompt + Intent
                                      |
                                      v
+-------------------------------------------------------------------------+
|                     Orchestrator / Agent Loop                           |
|  +-------------------------------------------------------------------+  |
|  |  Plan -> Act -> Observe -> Reflect -> Repeat                       |  |
|  +-------------------------------------------------------------------+  |
|         |           |            |            |            |            |
|         v           v            v            v            v            |
|  +----------+ +----------+ +---------+ +----------+ +-----------+      |
|  | 代码理解 | | 上下文管理| | LSP 集成| | 编辑引擎 | | 执行沙箱  |      |
|  | Module   | | Module   | | Module  | | Module   | | Sandbox   |      |
|  +----------+ +----------+ +---------+ +----------+ +-----------+      |
|         |           |            |            |            |            |
|         v           v            v            v            v            |
|  +----------+ +----------+ +---------+ +----------+ +-----------+      |
|  | AST 解析 | | 文件缓存 | | 诊断    | | Diff     | | 容器化    |      |
|  | 符号索引 | | 项目结构 | | Hover   | | 生成     | | 执行环境  |      |
|  | 引用分析 | | Git 历史 | | 跳转定义 | | 多文件   | | 网络隔离  |      |
|  | 类型推导 | | 依赖图   | | 补全    | | 重构    | | 超时保护  |      |
|  +----------+ +----------+ +---------+ +----------+ +-----------+      |
+-------------------------------------------------------------------------+
                        |                |
                        v                v
               +----------------+ +------------------+
               | Vector Store   | |  LLM Inference   |
               | (代码嵌入索引)  | |  (Claude/GPT-4)  |
               +----------------+ +------------------+
```

### 2.2 代码理解模块 (Code Understanding)

代码理解是编程 Agent 的"眼睛"——它决定了 Agent 能多准确地感知代码库。

```
                    +-----------------------------+
                    |   Code Understanding Stack  |
                    +-----------------------------+
                    |  Layer 4: 语义理解          |
                    |  - 函数意图分类              |
                    |  - API 使用模式识别           |
                    |  - 业务逻辑提取              |
                    +-----------------------------+
                    |  Layer 3: 关系分析          |
                    |  - 调用图 (Call Graph)       |
                    |  - 继承层次 (Inheritance)    |
                    |  - 数据流 (Data Flow)        |
                    +-----------------------------+
                    |  Layer 2: 符号索引          |
                    |  - 定义/引用关系              |
                    |  - 跨文件符号表              |
                    |  - 类型信息                  |
                    +-----------------------------+
                    |  Layer 1: 语法分析          |
                    |  - AST 解析                  |
                    |  - 语法错误检测              |
                    |  - 结构提取 (类/函数/变量)   |
                    +-----------------------------+
```

**核心数据结构 —— 符号索引 (Symbol Index):**

```python
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional


@dataclass
class SymbolDefinition:
    """代码中符号的定义信息"""
    name: str
    kind: str  # 'function' | 'class' | 'variable' | 'interface' | 'type'
    file_path: Path
    start_line: int
    end_line: int
    signature: str
    docstring: Optional[str] = None
    visibility: str = 'public'  # 'public' | 'private' | 'protected'


@dataclass
class SymbolReference:
    """符号的引用位置"""
    symbol_name: str
    file_path: Path
    line: int
    column: int
    reference_kind: str  # 'call' | 'read' | 'write' | 'import'


@dataclass
class CrossFileRelation:
    """跨文件关系"""
    source_file: Path
    target_file: Path
    relation_type: str  # 'import' | 'extend' | 'implement' | 'call'
    symbols: list[str] = field(default_factory=list)
```

**符号索引构建过程:**

```python
import ast
import tree_sitter  # 更精确的 AST 解析
from concurrent.futures import ThreadPoolExecutor


class CodeIndexer:
    """
    代码索引器：构建整个项目的符号索引
    
    使用 tree-sitter 进行多语言 AST 解析，
    支持 Python/TypeScript/Java/Go 等主流语言。
    """
    
    def __init__(self, project_root: Path):
        self.project_root = project_root
        self.symbols: dict[str, list[SymbolDefinition]] = {}
        self.references: dict[str, list[SymbolReference]] = {}
        self.cross_file_relations: list[CrossFileRelation] = []
        
    def build_index(self) -> None:
        """构建全量代码索引"""
        files = list(self.project_root.rglob("*.py")) + \
                list(self.project_root.rglob("*.ts")) + \
                list(self.project_root.rglob("*.tsx"))
        
        with ThreadPoolExecutor(max_workers=8) as executor:
            results = executor.map(self._index_single_file, files)
        
        for symbols, refs, relations in results:
            for sym in symbols:
                self.symbols.setdefault(sym.name, []).append(sym)
            for ref in refs:
                self.references.setdefault(ref.symbol_name, []).append(ref)
            self.cross_file_relations.extend(relations)
    
    def _index_single_file(self, file_path: Path):
        """解析单个文件，提取符号和引用"""
        with open(file_path, 'r', encoding='utf-8', errors='replace') as f:
            source = f.read()
        
        # 使用 tree-sitter 解析 AST
        # 这里用 Python ast 模块做简化演示
        try:
            tree = ast.parse(source, filename=str(file_path))
        except SyntaxError:
            return [], [], []
        
        symbols = []
        references = []
        relations = []
        
        for node in ast.walk(tree):
            if isinstance(node, ast.FunctionDef):
                symbols.append(SymbolDefinition(
                    name=node.name,
                    kind='function',
                    file_path=file_path,
                    start_line=node.lineno,
                    end_line=getattr(node, 'end_lineno', node.lineno),
                    signature=f"def {node.name}(...)"
                ))
            elif isinstance(node, ast.ClassDef):
                symbols.append(SymbolDefinition(
                    name=node.name,
                    kind='class',
                    file_path=file_path,
                    start_line=node.lineno,
                    end_line=getattr(node, 'end_lineno', node.lineno),
                    signature=f"class {node.name}(...)"
                ))
            elif isinstance(node, ast.Call):
                if isinstance(node.func, ast.Name):
                    references.append(SymbolReference(
                        symbol_name=node.func.id,
                        file_path=file_path,
                        line=node.lineno,
                        column=node.col_offset,
                        reference_kind='call'
                    ))
        
        return symbols, references, relations
```

### 2.3 上下文管理模块 (Context Management)

上下文管理是编程 Agent 的核心挑战——如何在有限的上下文窗口中塞入最相关的信息。

```
                        +-----------------------------+
                        |    Context Manager           |
                        |                              |
                        |  总预算: ~100K tokens        |
                        +-----------------------------+
                               |            |
               +---------------+            +---------------+
               | 静态上下文 (30%)           | 动态上下文 (70%) |
               |                           |                |
               +---------+---------+       +--------+-------+
                         |                          |
               +---------+-------+        +---------+--------+
               | 项目配置文件     |        | 用户当前打开文件  |
               | package.json    |        | 光标附近代码     |
               | tsconfig.json   |        | 最近编辑区域     |
               | .gitignore      |        | 选中文本         |
               +-----------------+        +-----------------+
               +---------+-------+        +---------+--------+
               | 项目结构摘要    |        | 搜索结果片段     |
               | 目录树          |        | grep 匹配行     |
               | 文件列表        |        | AST 查询结果    |
               +-----------------+        +-----------------+
               +---------+-------+        +---------+--------+
               | 关键类型定义    |        | Git 变更        |
               | 接口/类型签名   |        | diff 上下文     |
               | 导出 API        |        | commit 历史     |
               +-----------------+        +-----------------+
```

**上下文优先级排序算法:**

```python
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional


@dataclass
class ContextItem:
    """上下文条目"""
    content: str
    source: str          # 'file' | 'search' | 'git' | 'symbol'
    file_path: Path
    start_line: int
    end_line: int
    priority_score: float = 0.0
    token_count: int = 0
    relevance_reason: str = ""


class ContextRanker:
    """
    上下文排序器：对候选上下文片段进行优先级排序
    
    排序因素 (加权评分):
    - 编辑距离: 距离光标位置越近，分数越高 (weight: 0.35)
    - 引用频率: 被当前编辑文件引用越多，分数越高 (weight: 0.20)
    - 修改热区: 近期修改越频繁，分数越高 (weight: 0.15)
    - 符号关联: 与当前编辑符号直接相关，分数越高 (weight: 0.20)
    - 文件类型: 核心业务代码 > 配置文件 > 测试代码 (weight: 0.10)
    """
    
    def __init__(self, cursor_position: tuple[int, int],
                 current_file: Path, open_files: list[Path]):
        self.cursor_line, self.cursor_col = cursor_position
        self.current_file = current_file
        self.open_files = open_files
        
        # 权重配置
        self.weights = {
            'edit_distance': 0.35,
            'ref_count': 0.20,
            'edit_frequency': 0.15,
            'symbol_relevance': 0.20,
            'file_type': 0.10,
        }
    
    def rank_contexts(self, candidates: list[ContextItem],
                      budget_tokens: int = 64000) -> list[ContextItem]:
        """对候选上下文排序，确保总 token 不超过预算"""
        
        for item in candidates:
            score = self._compute_score(item)
            item.priority_score = score
        
        # 按分数降序排列
        ranked = sorted(candidates, key=lambda x: -x.priority_score)
        
        # 贪心选择：在 token 预算内选择最高优先级的上下文
        selected = []
        used_tokens = 0
        for item in ranked:
            if used_tokens + item.token_count <= budget_tokens:
                selected.append(item)
                used_tokens += item.token_count
            else:
                break
        
        return selected
    
    def _compute_score(self, item: ContextItem) -> float:
        """计算单个上下文项的优先级分数"""
        score = 0.0
        
        # 1. 编辑距离分数: 离光标越近越高
        if item.file_path == self.current_file:
            line_distance = abs(item.start_line - self.cursor_line)
            # 指数衰减: 同文件内距离 50 行以内的代码
            distance_score = max(0, 1.0 - line_distance / 50.0)
            score += self.weights['edit_distance'] * distance_score
        
        # 2. 引用频率分数
        if item.file_path in self.open_files:
            score += self.weights['edit_distance'] * 0.5  # 打开的文件有加分
        
        # 3. 符号关联（简化版）
        # 实际实现中需要检查 item 中是否包含当前文件的 import 或引用
        if item.source == 'symbol':
            score += self.weights['symbol_relevance'] * 0.8
        
        # 4. 文件类型优先级
        file_type_scores = {
            '.py': 0.9, '.ts': 0.9, '.js': 0.9,
            '.tsx': 0.85, '.jsx': 0.85,
            '.json': 0.5, '.yaml': 0.4, '.toml': 0.4,
            '.test.py': 0.3, '.spec.ts': 0.3,
        }
        ext = item.file_path.suffix
        # 特殊处理测试文件
        if '.test' in item.file_path.name or '.spec' in item.file_path.name:
            ext = '.test' + ext
        score += self.weights['file_type'] * file_type_scores.get(ext, 0.3)
        
        return score
```

### 2.4 LSP 集成 (Language Server Protocol)

LSP 是编程 Agent 连接语言智能的关键通道。

```
                    +------------------+
                    |   Coding Agent   |
                    +--------+---------+
                             |
                    LSP 请求 / 响应
                             |
                    +--------+---------+
                    |  LSP Proxy Layer |
                    |  (缓存 + 批处理) |
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
              v              v              v
     +---------+      +---------+     +---------+
     | pyright |      | tsserver |     | 其他 LSP|
     | (Python)|      | (TS/JS)  |     | 服务器  |
     +---------+      +---------+     +---------+
```

**LSP 代理层封装:**

```python
import json
import subprocess
from typing import Optional


class LSPClient:
    """
    LSP 客户端封装
    
    封装 LSP 协议的核心操作：
    - textDocument/definition: 跳转到定义
    - textDocument/references: 查找引用
    - textDocument/hover: 悬停信息
    - textDocument/completion: 自动补全
    - textDocument/diagnostic: 诊断信息
    - textDocument/codeAction: 代码操作
    """
    
    def __init__(self, language: str, workspace_root: str):
        self.language = language
        self.workspace_root = workspace_root
        self.request_id = 0
        self._proc = None
        self._capabilities = {}
    
    def _send_request(self, method: str, params: dict) -> dict:
        """发送 LSP 请求"""
        self.request_id += 1
        request = {
            'jsonrpc': '2.0',
            'id': self.request_id,
            'method': method,
            'params': params,
        }
        # 发送和接收 LSP 消息（通过 stdin/stdout）
        payload = json.dumps(request)
        # ... LSP 传输层实现 ...
        return {'result': {}}  # 简化返回
    
    def go_to_definition(self, file_path: str, line: int, col: int) -> Optional[dict]:
        """跳转到定义"""
        response = self._send_request('textDocument/definition', {
            'textDocument': {'uri': f'file://{file_path}'},
            'position': {'line': line, 'character': col},
        })
        return response.get('result')
    
    def find_references(self, file_path: str, line: int, col: int) -> list[dict]:
        """查找所有引用"""
        response = self._send_request('textDocument/references', {
            'textDocument': {'uri': f'file://{file_path}'},
            'position': {'line': line, 'character': col},
            'context': {'includeDeclaration': True},
        })
        return response.get('result', [])
    
    def get_diagnostics(self, file_path: str) -> list[dict]:
        """获取文件诊断信息（错误、警告）"""
        response = self._send_request('textDocument/diagnostic', {
            'textDocument': {'uri': f'file://{file_path}'},
        })
        return response.get('result', {}).get('diagnostics', [])
    
    def get_hover_info(self, file_path: str, line: int, col: int) -> Optional[str]:
        """获取悬停信息"""
        response = self._send_request('textDocument/hover', {
            'textDocument': {'uri': f'file://{file_path}'},
            'position': {'line': line, 'character': col},
        })
        result = response.get('result')
        if result and 'contents' in result:
            contents = result['contents']
            if isinstance(contents, dict):
                return contents.get('value', '')
            return str(contents)
        return None
```

### 2.5 编辑引擎 (Edit Engine)

编辑引擎是将 Agent 的计划转化为实际代码修改的核心组件。

```
                    +---------------------------+
                    |     Edit Engine           |
                    |                           |
                    |   LLM 输出 -> 精确代码修改 |
                    +---------------------------+
                               |
                +--------------+--------------+
                |              |              |
                v              v              v
        +-----------+  +-----------+  +-----------+
        | Diff 格式 |  | 全文件重写 |  | 搜索替换  |
        | Unified   |  | Whole-file |  | Search &  |
        | Diff      |  | Replace   |  | Replace   |
        +-----------+  +-----------+  +-----------+

        Cursor Tab 的典型编辑流程:

        1. 用户输入意图描述
        2. Agent 读取相关上下文
        3. LLM 生成 <-{编辑指令}->
        4. Edit Engine 解析指令
        5. 应用修改到文件
        6. LSP 验证修改后无错误
        7. 显示 diff 给用户确认
```

**Diff 生成与合并:**

```python
import difflib
from pathlib import Path
from typing import Optional


@dataclass
class FileEdit:
    """文件编辑操作"""
    file_path: Path
    original_content: str
    new_content: str
    edit_type: str  # 'create' | 'modify' | 'delete'
    
    @property
    def diff_text(self) -> str:
        """生成 unified diff 格式"""
        original_lines = self.original_content.splitlines(keepends=True)
        new_lines = self.new_content.splitlines(keepends=True)
        
        diff = difflib.unified_diff(
            original_lines, new_lines,
            fromfile=str(self.file_path),
            tofile=str(self.file_path),
            n=3  # 上下文行数
        )
        return ''.join(diff)


class EditEngine:
    """
    编辑引擎：将 LLM 输出转换为精确的文件修改
    
    三种编辑模式:
    1. diff 模式: LLM 输出 unified diff，直接 apply
    2. search-replace 模式: 搜索目标代码块并替换
    3. whole-file 模式: LLM 输出完整文件内容，直接覆盖
    """
    
    def __init__(self, workspace_root: Path):
        self.workspace_root = workspace_root
        self.edit_history: list[FileEdit] = []
    
    def apply_edit(self, edit: FileEdit) -> bool:
        """应用文件编辑"""
        file_path = self.workspace_root / edit.file_path
        
        if edit.edit_type == 'create':
            file_path.parent.mkdir(parents=True, exist_ok=True)
            file_path.write_text(edit.new_content, encoding='utf-8')
            
        elif edit.edit_type == 'modify':
            # 先验证原始内容匹配
            current = file_path.read_text(encoding='utf-8')
            if current != edit.original_content:
                # 内容已变更，尝试三路合并
                return self._three_way_merge(file_path, edit)
            file_path.write_text(edit.new_content, encoding='utf-8')
            
        elif edit.edit_type == 'delete':
            file_path.unlink()
        
        self.edit_history.append(edit)
        return True
    
    def _three_way_merge(self, file_path: Path, edit: FileEdit) -> bool:
        """
        三路合并：当文件在 Agent 读取后被修改时使用
        
        基础: edit.original_content (Agent 读取时的版本)
        本地: file_path 当前内容 (用户可能手动修改了)
        他们: edit.new_content (Agent 想要写入的版本)
        """
        base_lines = edit.original_content.splitlines(keepends=True)
        local_lines = file_path.read_text(encoding='utf-8').splitlines(keepends=True)
        their_lines = edit.new_content.splitlines(keepends=True)
        
        # 使用 difflib 的三路合并
        # 实际实现中需要更复杂的冲突检测
        from difflib import Differ
        differ = Differ()
        
        # 简化策略：如果在 local 中 base 的部分未改变，安全应用 their 的修改
        # 如果冲突了，回退到 search-replace 精准匹配
        has_conflict = self._detect_conflict(base_lines, local_lines, their_lines)
        
        if not has_conflict:
            # 安全合并
            merged = self._safe_merge(base_lines, local_lines, their_lines)
            file_path.write_text(''.join(merged), encoding='utf-8')
            return True
        
        return False  # 冲突，需要用户介入
    
    def _detect_conflict(self, base, local, their) -> bool:
        """检测是否有编辑冲突"""
        # 对于 base 中的每一段，检查在 local 中是否有修改
        # 同时在 their 中也有修改
        # 简单实现：如果 base != local 且 base != their，则可能有冲突
        # 实际需要更精细的行级对比
        return base != local and base != their
    
    def _safe_merge(self, base, local, their):
        """安全合并: 保留 local 的修改，应用 their 中不冲突的部分"""
        # 简化实现：使用 difflib.HtmlDiff 或类似工具
        # 生成行级 patch 并检查冲突
        return their  # 简化: 实际需要更复杂的合并策略


class SearchReplaceEditEngine(EditEngine):
    """
    搜索-替换编辑引擎
    
    Cursor Tab 使用的编辑格式:
    找到目标代码块的确切位置，然后替换。
    比 diff 更精确，不容易因为格式差异失败。
    """
    
    def parse_and_apply(self, file_path: Path, 
                         search_block: str, 
                         replace_block: str) -> bool:
        """解析搜索替换块并应用"""
        full_path = self.workspace_root / file_path
        content = full_path.read_text(encoding='utf-8')
        
        if search_block not in content:
            # 搜索块未找到，尝试模糊匹配
            match = self._fuzzy_search(content, search_block)
            if match is None:
                return False
            search_block = match
        
        new_content = content.replace(search_block, replace_block, 1)
        edit = FileEdit(
            file_path=file_path,
            original_content=content,
            new_content=new_content,
            edit_type='modify'
        )
        return self.apply_edit(edit)
    
    def _fuzzy_search(self, content: str, target: str, threshold: float = 0.85) -> Optional[str]:
        """
        模糊搜索: 当确切搜索块匹配失败时使用
        
        例如缩进变化、空行差异等情况。
        使用编辑距离或最长公共子序列进行匹配。
        """
        import difflib
        lines = content.splitlines()
        target_lines = target.splitlines()
        
        # 滑动窗口 + 相似度计算
        for i in range(len(lines) - len(target_lines) + 1):
            window = '\n'.join(lines[i:i + len(target_lines)])
            similarity = difflib.SequenceMatcher(
                None, target, window
            ).ratio()
            
            if similarity >= threshold:
                return window
        
        return None
```

### 2.6 执行沙箱 (Sandbox)

沙箱是编程 Agent 安全执行代码的关键保障。

```
                    +------------------------------+
                    |        Execution Sandbox      |
                    |                              |
                    |  +--------+  +-----------+   |
                    |  | Docker |  | 网络隔离   |   |
                    |  | 容器   |  | no-ethernet|   |
                    |  +--------+  +-----------+   |
                    |  +--------+  +-----------+   |
                    |  | 资源限制 |  | 文件系统   |   |
                    |  | CPU/Mem |  | 快照/回滚  |   |
                    |  +--------+  +-----------+   |
                    |  +--------+  +-----------+   |
                    |  | 超时保护 |  | 安全策略   |   |
                    |  | Timeout |  | Policy     |   |
                    |  +--------+  +-----------+   |
                    +------------------------------+
                               |
                    +----------+----------+
                    |                     |
                    v                     v
              +----------+          +----------+
              | 编译执行  |          | 测试运行  |
              | Compile   |          | Test Run |
              +----------+          +----------+
```

**沙箱执行器示例:**

```python
import asyncio
import subprocess
import tempfile
from pathlib import Path
from typing import Optional


@dataclass
class ExecutionResult:
    """执行结果"""
    success: bool
    stdout: str
    stderr: str
    exit_code: int
    execution_time_ms: int
    timeout_occurred: bool = False


class SandboxExecutor:
    """
    代码执行沙箱
    
    核心安全策略:
    - 网络隔离: 禁止所有出站连接
    - 资源限制: CPU 虚拟时间 30s, 内存 2GB
    - 文件隔离: 只读挂载项目目录, 写入到临时目录
    - 超时保护: 硬性超时 60s
    - 命令白名单: 仅允许安全的 shell 命令
    """
    
    def __init__(self, project_root: Path, sandbox_root: Optional[Path] = None):
        self.project_root = project_root
        self.sandbox_root = sandbox_root or Path(tempfile.mkdtemp())
        
        # 允许的安全命令
        self.allowed_commands = {
            'python', 'pytest', 'node', 'npm', 'npx',
            'tsc', 'go', 'rustc', 'cargo',
            'git', 'make', 'cmake',
        }
    
    async def run_code(self, code: str, language: str = 'python',
                        timeout_ms: int = 30000) -> ExecutionResult:
        """在沙箱中运行代码"""
        start_time = asyncio.get_event_loop().time()
        
        # 1. 写入临时文件
        ext_map = {'python': '.py', 'node': '.js', 'go': '.go'}
        ext = ext_map.get(language, '.py')
        
        tmp_file = self.sandbox_root / f"script_{hash(code)}{ext}"
        tmp_file.write_text(code, encoding='utf-8')
        
        # 2. 构建沙箱命令
        cmd_map = {
            'python': ['python', str(tmp_file)],
            'node': ['node', str(tmp_file)],
            'go': ['go', 'run', str(tmp_file)],
        }
        cmd = cmd_map.get(language, ['python', str(tmp_file)])
        
        # 3. 执行（实际环境中使用 Docker 隔离）
        try:
            proc = await asyncio.create_subprocess_exec(
                *cmd,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
                cwd=self.project_root,
                limit=1024 * 1024,  # 输出限制 1MB
            )
            
            try:
                stdout, stderr = await asyncio.wait_for(
                    proc.communicate(), timeout=timeout_ms / 1000
                )
                elapsed = (asyncio.get_event_loop().time() - start_time) * 1000
                
                return ExecutionResult(
                    success=proc.returncode == 0,
                    stdout=stdout.decode('utf-8', errors='replace')[:10000],
                    stderr=stderr.decode('utf-8', errors='replace')[:5000],
                    exit_code=proc.returncode or 0,
                    execution_time_ms=int(elapsed),
                )
            except asyncio.TimeoutError:
                proc.kill()
                return ExecutionResult(
                    success=False,
                    stdout='',
                    stderr='Timeout: execution exceeded {}ms'.format(timeout_ms),
                    exit_code=-1,
                    execution_time_ms=timeout_ms,
                    timeout_occurred=True,
                )
        finally:
            # 清理临时文件
            tmp_file.unlink(missing_ok=True)
    
    async def run_test(self, test_path: str, timeout_ms: int = 60000) -> ExecutionResult:
        """运行测试文件"""
        test_file = self.project_root / test_path
        if not test_file.exists():
            return ExecutionResult(False, '', f'Test file not found: {test_path}',
                                    -1, 0)
        
        # 使用 pytest 运行测试
        cmd = ['python', '-m', 'pytest', str(test_file), '-v', '--tb=short']
        
        proc = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            cwd=self.project_root,
        )
        
        try:
            stdout, stderr = await asyncio.wait_for(
                proc.communicate(), timeout=timeout_ms / 1000
            )
            return ExecutionResult(
                success=proc.returncode == 0,
                stdout=stdout.decode('utf-8', errors='replace')[:20000],
                stderr=stderr.decode('utf-8', errors='replace')[:5000],
                exit_code=proc.returncode or 0,
                execution_time_ms=0,  # 简化实现
            )
        except asyncio.TimeoutError:
            proc.kill()
            return ExecutionResult(False, '', 'Test timeout', -1, timeout_ms, True)
```

---

## 3. 核心设计决策

### 3.1 上下文窗口分配策略

编程 Agent 面临的核心约束：**上下文窗口有限，但代码库无限**。

```
                    +-------------------------------+
                    |  100K Token Budget            |
                    +-------------------------------+
                    | System Prompt:         ~4K    |  Agent 角色定义、工具描述
                    | Conversation History:  ~8K    |  最近 5-10 轮交互
                    | User Intent:           ~2K    |  当前用户请求
                    +-------------------------------+
                    | 动态上下文分配 (剩余 ~86K)     |
                    +-------------------------------+
                    |   高优先级 (50% ~ 43K):        |
                    |   - 当前文件光标附近           |
                    |   - 错误/诊断相关代码          |
                    |   - 当前编辑目标符号定义       |
                    |   中优先级 (30% ~ 26K):        |
                    |   - 打开的其他文件             |
                    |   - 搜索结果片段               |
                    |   - import 的模块签名          |
                    |   低优先级 (20% ~ 17K):        |
                    |   - Git diff 上下文            |
                    |   - 项目结构摘要               |
                    |   - 相关测试文件               |
                    +-------------------------------+
```

**Cursor 的上下文管理策略:**

| 策略 | 描述 | 适用场景 |
|------|------|---------|
| FIFO Eviction | 最早添加的上下文最先被移除 | 长对话历史 |
| Relevance Scoring | 基于语义相关性打分，保留高分 | 代码库搜索 |
| Token Budget | 为每种上下文分配固定预算 | 稳定可预期的行为 |
| Hierarchical Summarization | 先摘要再包含完整内容 | 大型函数/文件 |

### 3.2 编辑格式选择

三种主流编辑格式的对比：

```
   Diff 格式              Search-Replace 格式       Whole-file 格式
+------------------+    +---------------------+   +------------------+
| --- a/foo.py     |    | SEARCH:             |   | (完整文件内容)   |
| +++ b/foo.py     |    | def old_func(x):    |   |                  |
| @@ -10,5 +10,7 @@|    |     return x + 1    |   | import os        |
|  def foo():      |    |                     |   | import sys       |
| -    return 1    |    | REPLACE:            |   |                  |
| +    return 2    |    | def new_func(x):    |   | def main():      |
| +    # new line  |    |     return x + 2    |   |     ...          |
+------------------+    +---------------------+   +------------------+

  优点: 精确简洁          优点: 人类可读、          优点: 简单直接
  缺点: 行号敏感           不易出错                 缺点: token 消耗大
  适用: 自动化工具         缺点: 唯一性要求         适用: 小文件/新文件
  适用: Cursor/Copilot     适用: 任何大小
```

**决策建议:**

| 场景 | 推荐格式 | 原因 |
|------|---------|------|
| 新增文件 | Whole-file | 无冲突、简单 |
| 修改 1-5 行 | Search-Replace | 精确、容错性强 |
| 大规模重构 | Diff | 高效、适合批量操作 |
| 修正编译错误 | Search-Replace | 目标明确、不易出错 |

### 3.3 工具设计

编程 Agent 的工具集设计直接影响其能力边界和用户体验。

```
+------------------------+----------------------------------------+
|      读取工具           |      写入工具                           |
+------------------------+----------------------------------------+
| read(file_path)        | edit(file_path, old, new)              |
|   - 读取文件内容       |   - 搜索替换编辑                        |
|   - 支持行范围         |   - 自动处理缩进                        |
|                        |   - 语法验证后应用                      |
| grep(pattern)          | write(file_path, content)               |
|   - 全项目正则搜索     |   - 创建新文件/完全覆盖                  |
|   - 支持 glob 过滤     |   - 自动创建目录                        |
|   - 返回文件名+行号    |                                        |
|                        | run(command)                            |
| ls(path)               |   - 在沙箱中执行命令                    |
|   - 列出目录内容       |   - 输出截断 + 超时                     |
|   - 显示文件大小       |                                        |
|                        | create_pr(title, body)                 |
| codebase_search(query) |   - 创建 Pull Request                   |
|   - 语义搜索           |   - 包含变更摘要                        |
|   - 返回代码片段       |                                        |
+------------------------+----------------------------------------+
```

**工具设计原则:**

1. **原子性**: 每个工具只做一件事，做好一件事
2. **可观测性**: 工具执行结果必须包含足够的信息让 Agent 判断下一步
3. **安全性**: 写操作需要用户确认，读操作自动执行
4. **幂等性**: 多次调用相同工具应产生相同结果

---

## 4. 架构模式分析

### 4.1 Cursor Tab 架构特点

Cursor Tab 是目前最成功的编程 Agent 之一，其架构特点：

```
                  +-----------------------------------+
                  |      Cursor Tab 架构              |
                  +-----------------------------------+
                  |                                   |
                  |  +-----------------------------+  |
                  |  |  快速路径 (Fast Path)        |  |
                  |  |  - 直接使用当前文件上下文     |  |
                  |  |  - 单次 LLM 调用              |  |
                  |  |  - < 500ms 响应时间           |  |
                  |  |  适用: 简单补全、行内编辑     |  |
                  |  +-----------------------------+  |
                  |              |                     |
                  |              v                     |
                  |  +-----------------------------+  |
                  |  |  慢速路径 (Slow Path)        |  |
                  |  |  - 全项目上下文收集           |  |
                  |  |  - 多步 Agent 循环            |  |
                  |  |  - 2-10s 响应时间             |  |
                  |  |  适用: 复杂重构、跨文件修改   |  |
                  |  +-----------------------------+  |
                  |                                   |
                  |  +-----------------------------+  |
                  |  |  Composer (计划模式)         |  |
                  |  |  - 用户打开专用面板           |  |
                  |  |  - 多次迭代编辑               |  |
                  |  |  - 手动触发执行               |  |
                  |  |  适用: 大型功能开发           |  |
                  |  +-----------------------------+  |
                  +-----------------------------------+
```

**双路径设计的原因:**

| 特性 | Fast Path | Slow Path |
|------|-----------|-----------|
| 延迟要求 | < 500ms | < 10s |
| 上下文大小 | < 4K tokens | < 100K tokens |
| LLM 调用次数 | 1 | 1-10 |
| 编辑范围 | 单文件, 单位置 | 多文件, 多位置 |
| 触发方式 | 自动 | 按 Tab 或 Ctrl+K |
| 回退策略 | 失败 -> 降级到传统补全 | 失败 -> 用户修正描述 |

### 4.2 Agentic vs Non-Agentic 模式

```
   Non-Agentic (传统 Copilot)          Agentic (Cursor Tab / Copilot Agent)
+------------------------------+    +----------------------------------+
|  用户输入 -> 生成 -> 结束    |    |  用户输入 -> 规划 -> 执行 -> 观察 |
|                              |    |    -> 反思 -> 再执行 -> 结束     |
|  单次 LLM 调用              |    |  多轮 LLM 调用                   |
|  无工具使用                  |    |  使用工具 (读/写/搜索/运行)      |
|  无状态                      |    |  有状态 (跟踪已完成步骤)         |
|  无错误恢复                  |    |  自动错误检测和修正              |
|  无验证                      |    |  执行后验证 (编译/测试)          |
+------------------------------+    +----------------------------------+

  Copilot 内联补全                 Cursor Tab / Copilot Agent Mode
```

**何时使用 Agentic 模式:**

| 场景 | 模式 | 原因 |
|------|------|------|
| 行内补全 (typing 时) | Non-Agentic | 延迟敏感, 简单任务 |
| 函数实现 | Non-Agentic 或 Agentic | 上下文单一的可非 Agentic |
| 跨文件重构 | Agentic | 需要搜索和理解多个文件 |
| Bug 修复 | Agentic | 需要定位根因 + 验证 |
| 新功能开发 | Agentic | 需要规划和多步执行 |
| 代码审查 | Agentic | 需要全局理解和分析 |

### 4.3 微服务角色划分

如果将编程 Agent 拆分为微服务:

```
                    +-----------------------------+
                    |   Gateway / Orchestrator     |
                    |   - 会话管理                 |
                    |   - 请求路由                 |
                    |   - 速率限制                 |
                    +-----------------------------+
                    |              |               |
          +---------+              |              +---------+
          v                        v                        v
+-------------------+  +-------------------+  +-------------------+
| Context Service   |  | Intent Service    |  | Execution Service |
| - 文件缓存        |  | - 意图分类        |  | - 沙箱管理       |
| - 索引管理        |  | - 任务分解        |  | - 编译执行       |
| - 优先级排序      |  | - 步骤规划        |  | - 测试运行       |
+-------------------+  +-------------------+  +-------------------+
          |                      |                      |
          v                      v                      v
+-------------------+  +-------------------+  +-------------------+
| Code Intelligence |  | Generation Service|  | Validation Service|
| - AST 解析        |  | - LLM 调用        |  | - 语法检查       |
| - LSP 代理        |  | - 提示构建        |  | - 类型检查       |
| - 符号索引        |  | - 结果解析        |  | - Lint 检查      |
+-------------------+  +-------------------+  +-------------------+
```

---

## 5. 完整工作流示例

```python
import asyncio
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional


@dataclass
class AgentStep:
    """Agent 执行的单个步骤"""
    action: str           # 'read' | 'search' | 'edit' | 'run' | 'think'
    input: str
    output: str
    duration_ms: int


class CodingAgent:
    """
    编程 Agent 主循环
    
    工作流程:
    1. 接收用户意图
    2. 收集上下文
    3. 规划步骤
    4. 逐步执行（每步包含: 思考 -> 行动 -> 观察）
    5. 验证结果
    6. 展示给用户
    """
    
    def __init__(self, project_root: Path, llm_client):
        self.project_root = project_root
        self.llm = llm_client
        self.context_manager = ContextRanker(
            cursor_position=(0, 0),
            current_file=Path(""),
            open_files=[]
        )
        self.edit_engine = SearchReplaceEditEngine(project_root)
        self.sandbox = SandboxExecutor(project_root)
        self.indexer = CodeIndexer(project_root)
        
        # 执行历史
        self.steps: list[AgentStep] = []
        self.max_steps = 25
    
    async def run(self, user_intent: str) -> list[FileEdit]:
        """执行用户意图，返回所有编辑操作"""
        
        print(f"[Agent] 开始处理: {user_intent}")
        
        # Step 1: 收集上下文
        context = await self._gather_context(user_intent)
        print(f"[Agent] 上下文收集完成: {sum(c.token_count for c in context)} tokens")
        
        # Step 2: 生成计划
        plan = await self._generate_plan(user_intent, context)
        print(f"[Agent] 计划生成: {len(plan)} 步骤")
        
        # Step 3: 执行计划
        edits = []
        for i, step in enumerate(plan):
            step_start = asyncio.get_event_loop().time()
            
            result = await self._execute_step(step, context)
            
            elapsed = (asyncio.get_event_loop().time() - step_start) * 1000
            self.steps.append(AgentStep(
                action=step['action'],
                input=step.get('params', ''),
                output=str(result),
                duration_ms=int(elapsed)
            ))
            
            if isinstance(result, FileEdit):
                edits.append(result)
            
            # Step 4: 验证
            if step.get('verify', False):
                verified = await self._verify_step(result)
                if not verified:
                    print(f"[Agent] 步骤 {i} 验证失败，尝试修正...")
                    # 重试逻辑
                    break
        
        print(f"[Agent] 完成! 共 {len(self.steps)} 步, {len(edits)} 个文件修改")
        return edits
    
    async def _gather_context(self, intent: str) -> list[ContextItem]:
        """收集相关的代码上下文"""
        # 1. 解析意图中的关键词
        keywords = self._extract_keywords(intent)
        
        # 2. 搜索相关文件
        candidates = []
        for keyword in keywords:
            results = await self._search_code(keyword)
            candidates.extend(results)
        
        # 3. 排序和裁剪
        ranked = self.context_manager.rank_contexts(
            candidates, budget_tokens=64000
        )
        
        return ranked
    
    async def _generate_plan(self, intent: str, 
                              context: list[ContextItem]) -> list[dict]:
        """使用 LLM 生成执行计划"""
        # 构建提示
        prompt = self._build_plan_prompt(intent, context)
        
        # 调用 LLM
        response = await self.llm.generate(prompt)
        
        # 解析计划
        plan = self._parse_plan(response)
        return plan
    
    async def _execute_step(self, step: dict, 
                             context: list[ContextItem]) -> any:
        """执行单个计划步骤"""
        action = step['action']
        params = step.get('params', {})
        
        if action == 'read':
            return self._read_file(params['file_path'], 
                                     params.get('lines'))
        elif action == 'search':
            return await self._search_code(params['pattern'])
        elif action == 'edit':
            return self.edit_engine.parse_and_apply(
                params['file_path'],
                params['search'],
                params['replace']
            )
        elif action == 'run':
            return await self.sandbox.run_code(
                params['code'],
                params.get('language', 'python')
            )
        elif action == 'think':
            # Agent 内部思考步骤（无外部副作用）
            return {'thought': params.get('reasoning', '')}
        
        return None
    
    def _extract_keywords(self, text: str) -> list[str]:
        """从用户输入中提取搜索关键词"""
        # 简化实现: 提取函数名、类名、文件名模式
        import re
        # 匹配可能的标识符
        identifiers = re.findall(r'\b[a-zA-Z_][a-zA-Z0-9_]*\b', text)
        # 过滤掉常见停用词
        stopwords = {'the', 'is', 'at', 'which', 'on', 'in', 'to', 'for',
                     'add', 'fix', 'update', 'remove', 'change', 'make'}
        return [w for w in identifiers if w.lower() not in stopwords]
    
    def _build_plan_prompt(self, intent: str, context: list[ContextItem]) -> str:
        """构建规划提示"""
        context_text = "\n\n".join([
            f"=== {c.file_path}:{c.start_line}-{c.end_line} ===\n{c.content}"
            for c in context[:20]  # 限制上下文数量
        ])
        
        return f"""你是一个编程 Agent。用户要求: {intent}

以下是相关的代码上下文:
{context_text}

请制定一个执行计划，用 JSON 数组格式返回每一步的操作。
每一步包含: action (read/search/edit/run), params, reasoning。
只返回 JSON 数组，不要其他内容。"""
    
    def _parse_plan(self, llm_response: str) -> list[dict]:
        """解析 LLM 返回的计划"""
        import json
        import re
        
        # 提取 JSON
        json_match = re.search(r'\[.*\]', llm_response, re.DOTALL)
        if json_match:
            try:
                return json.loads(json_match.group())
            except json.JSONDecodeError:
                pass
        
        # 回退: 单步计划
        return [{'action': 'think', 'params': {'reasoning': '直接执行'}}]
    
    def _read_file(self, file_path: str, lines: Optional[tuple] = None) -> str:
        """读取文件内容"""
        full_path = self.project_root / file_path
        if not full_path.exists():
            return f"File not found: {file_path}"
        
        content = full_path.read_text(encoding='utf-8')
        if lines:
            start, end = lines
            lines_list = content.splitlines()[start:end]
            return '\n'.join(lines_list)
        return content
    
    async def _search_code(self, pattern: str) -> list[ContextItem]:
        """搜索代码"""
        import subprocess
        # 使用 ripgrep 搜索
        result = subprocess.run(
            ['rg', '-n', pattern, '--type-add', 'code:*.py,*.ts,*.tsx,*.js,*.jsx',
             '--type', 'code', str(self.project_root)],
            capture_output=True, text=True, timeout=10
        )
        
        items = []
        for line in result.stdout.splitlines()[:50]:
            parts = line.split(':', 2)
            if len(parts) == 3:
                file_path, line_num, content = parts
                items.append(ContextItem(
                    content=content.strip(),
                    source='search',
                    file_path=Path(file_path),
                    start_line=int(line_num),
                    end_line=int(line_num),
                    priority_score=0.5,
                    token_count=len(content) // 4,
                ))
        return items
    
    async def _verify_step(self, result: any) -> bool:
        """验证步骤执行结果"""
        if isinstance(result, FileEdit):
            # 检查文件是否有语法错误
            # 通过 LSP 获取诊断信息
            return True  # 简化
        return True
```

---

## 6. 关键挑战

### 6.1 上下文窗口限制

| 挑战 | 影响 | 缓解方案 |
|------|------|---------|
| 大型代码库无法全部放入上下文 | Agent 遗漏关键信息 | 分层摘要 + 按需加载 |
| 长对话历史消耗大量 token | Agent 忘记早期上下文 | 对话摘要 + 关键信息提取 |
| 多文件同时编辑需要广泛上下文 | 上下文碎片化 | 优先级排序 + 缓存策略 |
| 第三方库代码不可用 | 无法理解库行为 | 类型签名 + 文档嵌入 |

### 6.2 编辑准确性

```
  编辑失败模式:
  
  1. 搜索块不唯一
     SEARCH: "def process(data):"
     -> 文件中有多个 def process(data)
     -> 需要添加周围上下文使其唯一
  
  2. 搜索块已变更
     SEARCH 的内容在 Agent 读取后被用户编辑
     -> 搜索替换失败
     -> 需要模糊匹配或三路合并
  
  3. 编辑引入语法错误
     替换后产生不完整/不合法的代码
     -> LSP 诊断检测
     -> 自动回退或二次修正
  
  4. 缩进/格式不匹配
     LLM 生成的代码缩进与项目配置不一致
     -> 使用项目的 formatter 自动格式化
```

**Cursor 的解决方案:**

1. **LSP 预检**: 在应用编辑前使用 LSP 验证语法正确性
2. **自动格式化**: 编辑后自动运行 `prettier` 或 `black`
3. **用户确认**: 显示 diff 让用户确认后才写入
4. **Undo 支持**: 所有编辑都可撤销

### 6.3 跨文件理解

编程任务经常需要同时理解多个相关文件：

```
  用户: "添加用户注册功能"
  
  涉及的文件:
  
  routes/auth.py       # 需要添加新路由
  models/user.py       # 需要添加 User 模型
  schemas/auth.py      # 需要添加 Pydantic schema
  services/auth.py     # 需要添加业务逻辑
  tests/test_auth.py   # 需要添加测试
  alembic/versions/    # 需要数据库迁移
```

**解决方案:**

1. **依赖图遍历**: 从当前文件出发，沿 import 图展开
2. **关联文件推荐**: 基于历史编辑模式，推荐相关文件
3. **分步上下文加载**: 先加载核心文件，再按需加载次要文件
4. **索引优先**: 先建立全项目索引，再针对性地读取内容

### 6.4 延迟要求

编程 Agent 面临严格延迟约束：

| 模式 | 目标延迟 | 实现方式 |
|------|---------|---------|
| 行内补全 | < 300ms | 专用小模型 + 预计算 |
| Tab 建议 | < 1s | 投机解码 + 缓存 |
| Ctrl+K 编辑 | < 5s | 流式输出 + 增量显示 |
| 多文件重构 | < 30s | 异步上下文加载 + 并行 LLM 调用 |
| Agent 模式 | < 2min | 进度反馈 + 可中断 |

**延迟优化策略:**

1. **投机解码 (Speculative Decoding)**: 用小模型生成草稿，大模型验证
2. **提示缓存 (Prompt Cache)**: 复用系统提示和项目上下文
3. **预加载**: 用户开始输入时预加载可能的上下文
4. **流式输出**: 边生成边显示，减少感知延迟
5. **快速回退路径**: 慢路径超时时降级到简单补全

---

## 7. 与 SWE-agent 对比

### 7.1 SWE-agent 设计概述

SWE-agent (2024, Princeton) 是专为 SWE-bench 设计的编程 Agent：

```
   SWE-agent 架构:
   
   +---------------------------------------------------+
   |   Agent 循环 (预测 -> 行动 -> 观察)               |
   +---------------------------------------------------+
              |              |               |
              v              v               v
   +----------------+ +------------+ +----------------+
   | 文件查看器      | | 目录编辑器 | | 命令执行器     |
   | (File Viewer)  | | (Editor)   | | (Jupyter)      |
   +----------------+ +------------+ +----------------+
              |              |               |
              v              v               v
   +---------------------------------------------------+
   |  临时文件系统 (Temp Files)                        |
   |  - 修改存储在临时文件                             |
   |  - 完成后 apply 到原始仓库                        |
   +---------------------------------------------------+
```

### 7.2 关键差异对比

| 维度 | Cursor/Copilot Agent | SWE-agent |
|------|---------------------|-----------|
| **目标场景** | 辅助日常编程 | 自动化修复 GitHub Issue |
| **交互方式** | IDE 内嵌, 实时交互 | 命令行, 批量处理 |
| **上下文范围** | 当前项目 + 用户意图 | 单 Issue + 完整仓库 |
| **编辑方式** | Search-Replace / Diff | 逐行编辑 (文件查看器) |
| **执行环境** | 沙箱 (Docker) | Jupyter 内核 + shell |
| **工具设计** | 高粒度 (read/grep/ls) | 低粒度 (文件查看器 ↑↓) |
| **状态管理** | 持久化会话 (持续使用) | 单次任务 (用完即弃) |
| **成本优化** | 缓存 + 小模型优先 | 单次任务, 精度优先 |
| **失败恢复** | 自动重试 + 用户介入 | 自动修正 (最长 3 次) |

### 7.3 SWE-agent 的特殊设计

```
   SWE-agent 核心设计:
   
   1. 文件查看器 (File Viewer)
      不是传统读取, 而是分页查看:
      [File: src/main.py (0/100 lines)]
      window: [0-30]
      ...
      ─── window ───
      [scroll up] [scroll down]
      
      优点: 精确控制上下文
      缺点: 需要多次调用才能了解文件全貌
   
   2. 编辑方式
      不是一次写入, 是逐行操作:
      -> edit 10:13          # 编辑 10-13 行
      -> def new_func():     # 新内容
      ->     pass
      -> \end                
      
      优点: 精确控制修改位置
      缺点: 编辑大块代码效率低
   
   3. 提交策略
      不直接修改仓库, 而是:
      1. 在临时目录工作
      2. 生成 patch 文件
      3. 任务结束时 apply patch
```

### 7.4 对 Coding Agent 架构的启示

| 从 SWE-agent 学到的 | 应用到产品 Agent |
|---------------------|-----------------|
| 细粒度文件查看器 | 用于大型文件的分页阅读 |
| 明确的编辑格式 | Search-Replace + 行号 |
| 命令执行沙箱 | Docker 隔离 + 资源限制 |
| 自动提交/PR | 一键创建 PR 功能 |
| 错误修正循环 | 自动检测和修复编译错误 |
| Patch 生成 | 生成干净的 diff 给代码审查 |

```
  融合架构 (理想态):
  
  +----------------+  +--------------+  +-----------------+
  | Cursor 的 UX   |  | SWE-agent 的 |  | Copilot 的      |
  | IDE 集成体验   | +| 精确编辑引擎 | +| 代码理解能力    |
  +----------------+  +--------------+  +-----------------+
           |                |                   |
           v                v                   v
  +------------------------------------------------------+
  |             下一代编程 Agent 架构                     |
  |                                                      |
  |  - 编辑器内实时协作 (Cursor)                          |
  |  - 精确可靠的编辑执行 (SWE-agent)                    |
  |  - 深层代码理解 (Copilot)                            |
  |  - 自主问题解决 (Devin)                              |
  +------------------------------------------------------+
```

---

## 参考架构

- Cursor: https://cursor.sh
- SWE-agent: https://github.com/princeton-nlp/SWE-agent
- GitHub Copilot: https://github.com/features/copilot
- Devin: https://cognition.ai
- Aider: https://github.com/paul-gauthier/aider
- Continue: https://github.com/continuedev/continue
