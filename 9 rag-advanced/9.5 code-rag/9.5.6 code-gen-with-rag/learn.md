# 9.5.6 code-gen-with-rag — RAG 增强代码生成

## 简单介绍

RAG 增强代码生成（RAG-Augmented Code Generation）是指**通过检索相关代码片段来辅助 LLM 生成更准确、更符合项目风格的代码**的技术。不同于纯 LLM 代码生成（仅靠模型参数中的知识），RAG 增强的代码生成能结合项目自身的代码库作为参考，生成更贴合现有架构的代码。

## 基本原理

### RAG 增强代码生成的工作流

```
用户需求: "添加一个用户注销API端点"
    │
    ▼
┌──────────────────┐
│  1. 查询理解      │  → 提取意图: "创建新的 REST 端点"
│  需求 → 检索查询   │  → 关键要素: "User"、"DELETE"、"API"
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  2. 代码检索      │  → 检索已有路由: routes/users.py
│  检索相关代码片段   │  → 检索相似端点: delete_product, delete_order
└──────┬───────────┘  → 检索数据模型: User model
       │              → 检索测试模式: test_users.py
       ▼
┌──────────────────┐
│  3. 上下文组装    │  → 将检索结果构造成 LLM 上下文
│  代码片段 + 结构   │  → 包含: 路由模式 + 模型定义 + 测试示例
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  4. LLM 代码生成  │  → 参考项目风格生成符合架构的代码
│  生成最终代码      │  → 保持命名约定、错误处理模式一致
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  5. 验证与迭代    │  → 语法检查、导入补全
│  编译检查 + 测试   │  → 验证生成的代码可编译/运行
└──────────────────┘
```

### 为什么需要 RAG 增强

纯 LLM 代码生成的典型问题：

| 问题 | 表现 | RAG 如何解决 |
|------|------|-------------|
| **架构不一致** | 生成代码风格与项目不符 | 检索项目中的类似代码作为风格参考 |
| **API 幻觉** | 使用了项目中不存在的 API | 检索实际 API 定义和 import |
| **过时知识** | 生成已废弃的 API 用法 | 检索当前代码库中的实际用法 |
| **忽视约定** | 不遵循项目的错误处理/日志规范 | 检索已有代码作为 few-shot 示例 |
| **遗漏依赖** | 生成的代码缺少 import | 检索相关 import 并自动补全 |

## 背景：代码生成范式的演进

```
2021-2022: 零样本代码生成
  Prompt: "Write a Python function to..."
  LLM 完全依赖训练数据中的知识

2022-2023: 仓库上下文窗口
  Prompt: [仓库文件拼接] + "Add..."
  人工选择相关文件，拼接到 context 中

2023-2024: 检索增强生成
  Prompt: [RAG 检索的代码片段] + "Add..."
  自动检索相关代码，不依赖人工选择

2024-2025: Agentic 代码生成
  → Agent 自主决定: 需要什么信息 → 检索 → 生成 → 验证 → 修复
  → 多轮交互: 生成后自动运行测试，失败则重新生成
```

## 主流优化方向

### 1. Few-shot 代码示例检索

```python
def few_shot_code_generation(query: str, repo: RepoIndex) -> str:
    """检索最相似的代码示例作为 few-shot 参考"""
    
    # 检索与查询语义最相似的已有代码片段
    similar_code = repo.semantic_search(query, top_k=3)
    
    # 构建 few-shot prompt
    prompt = """You are generating code for this project. 
    Follow the exact style and patterns shown in these existing code examples:
    
    === EXISTING CODE EXAMPLES (参考已有代码风格) ===
    """
    
    for i, example in enumerate(similar_code, 1):
        prompt += f"""
    Example {i} (from {example['file']}:{example['start_line']}):
    ```{example['language']}
    {example['text']}
    ```
    """
    
    prompt += f"""
    === YOUR TASK ===
    Generate code that follows the same patterns as above:
    {query}
    """
    
    return llm_generate(prompt)
```

### 2. 上下文补全（Fill-in-the-Middle + RAG）

```python
def rag_infill_code(prefix: str, suffix: str, cursor_line: int, 
                    file_path: str, repo: RepoIndex) -> str:
    """RAG 增强的代码中间补全"""
    
    # 检索与当前文件相关的上下文
    related = repo.retrieve_by_proximity(file_path, cursor_line)
    
    # 检索当前文件的 import
    imports = repo.get_file_imports(file_path)
    
    # 检索附近函数的签名
    nearby_funcs = repo.get_nearby_functions(file_path, cursor_line)
    
    prompt = f"""
    Project context:
    - Imports: {imports}
    - Nearby functions: {nearby_funcs}
    - Related files: {[r['file'] for r in related[:3]]}
    
    Complete the code:
    ```python
    {prefix}⏸{suffix}
    ```
    Generate only the middle part (replace ⏸).
    """
    
    return llm_generate(prompt)
```

### 3. 测试感知的代码生成

```python
def test_aware_code_generation(test_file: str, repo: RepoIndex) -> str:
    """通过测试用例推断需要生成的代码"""
    
    # 从测试文件提取测试意图
    test_cases = repo.parse_test_cases(test_file)
    
    # 检索被测模块的已有代码
    module_under_test = resolve_module_under_test(test_file)
    existing_code = repo.get_file_content(module_under_test)
    
    # 检索项目中类似测试对应的实现作为参考
    similar_implementations = repo.search_similar_implementations(test_cases)
    
    prompt = f"""
    Implement the functions to make these tests pass:
    
    Tests: {test_file}
    {test_cases}
    
    Existing module code:
    {existing_code}
    
    Similar implementations in the project:
    {similar_implementations}
    """
    
    return llm_generate(prompt)
```

### 4. 迭代式生成+验证

```python
def iterative_code_generation(query: str, repo: RepoIndex, max_iter: int = 3):
    """生成 → 验证 → 修复 的迭代循环"""
    
    for i in range(max_iter):
        # Step 1: 检索
        context = repo.retrieve_for_query(query)
        
        # Step 2: 生成
        code = generate_code(query, context)
        
        # Step 3: 静态验证
        syntax_ok, errors = validate_syntax(code)
        if syntax_ok:
            # Step 4: 尝试导入/运行
            import_ok, import_errors = validate_imports(code, repo)
            if import_ok:
                return code
        
        # Step 5: 如果失败，收集错误信息
        error_context = {
            'generated_code': code,
            'errors': errors or import_errors,
            'fix_attempt': i + 1
        }
        
        # Step 6: 检索修复参考
        fix_examples = repo.search_similar_fixes(error_context)
        query += f"\n(Previous attempt failed: {errors}. Fix it.)"
        if fix_examples:
            query += f"\nReference fixes: {fix_examples}"
    
    return None  # 所有迭代都失败
```

### 5. 仓库级代码生成策略选择

```python
def select_generation_strategy(query: str, repo: RepoIndex):
    """根据任务类型选择最合适的代码生成策略"""
    
    task_type = classify_code_task(query)
    
    strategies = {
        'new_feature': {
            'strategy': few_shot_code_generation,
            'context': ['similar_features', 'arch_patterns', 'models']
        },
        'bug_fix': {
            'strategy': iterative_code_generation,
            'context': ['bug_context', 'test_failures', 'git_history']
        },
        'refactoring': {
            'strategy': structural_code_generation,
            'context': ['current_impl', 'tests', 'call_graph']
        },
        'code_explanation': {
            'strategy': explain_with_context,
            'context': ['function_def', 'callers', 'callees']
        },
        'test_generation': {
            'strategy': test_aware_code_generation,
            'context': ['module_under_test', 'similar_tests']
        }
    }
    
    return strategies.get(task_type, few_shot_code_generation)
```

## CodeRAG-Bench 基准

CodeRAG-Bench 是专门评估 RAG 增强代码生成的基准，包含三个核心维度：

| 维度 | 评测内容 | 关键指标 |
|------|---------|---------|
| **Exact Match** | 精确匹配已有 API/函数 | 编译通过率、API 使用正确率 |
| **Semantic Match** | 语义功能正确性 | 功能测试通过率 |
| **Style Match** | 代码风格一致性 | 命名/结构/模式的符合度 |

## 最大挑战

1. **上下文窗口与检索精度平衡**：对比普通 RAG 需要更多 context（因为代码依赖链长），但 token 预算有限
2. **生成代码的编译正确性**：检索到的代码片段可能已过时（被重构过），导致生成使用废弃 API
3. **样式模仿 vs 创造性**：太像现有代码可能只是复制粘贴，太少参考又可能脱离项目风格
4. **多语言混合生成**：一个 Web 项目中可能同时涉及 Python/JS/SQL/HTML，跨语言 RAG 难度大

## 能力边界

- ✅ 在已有架构上生成符合项目风格的增量代码
- ✅ 利用相似功能作为 few-shot 参考
- ✅ 生成时自动补充缺失的 import
- ❌ 从零生成全新架构的系统（缺乏参考代码）
- ❌ 需要深度领域知识（缺乏项目内相关知识）
- ❌ 多步骤重构（涉及大量文件修改的跨文件变更）

## 工程优化

1. **渐进式上下文注入**：先注入最重要的 few-shot，若 LLM 需要更多，再提供扩展信息
2. **模板预填充**：项目中常见的代码结构预编译为模板变量，减少检索频率
3. **生成的代码自动测试**：在 CI 中自动运行生成的代码的单元测试
4. **反馈循环**：将生成代码的后续修改（开发者的手动编辑）作为正反馈信号，改进检索

## 推荐工具与项目

- **CodeRAG-Bench**: 代码生成 RAG 评估基准
- **Sourcegraph Cody**: 代码感知的 AI 编码助手
- **GitHub Copilot**: 内联代码补全（隐式使用 RAG 风格）
- **CodiumAI**: 测试生成和代码审查
- **Continue.dev**: 开源 IDE 扩展，支持自定义代码 RAG
- **Aider**: 基于 git 的 AI 编码助手，自动检索仓库上下文
