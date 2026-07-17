# 9.1.2 metadata-annotation — 元数据标注：来源 / 时间 / 类型 / 权限

## 简单介绍

元数据标注是为每个索引块附加结构化标签信息的过程。如果说分块决定了"什么能被检索"，元数据标注决定了"检索结果能否被有效过滤、组织和呈现"。元数据是连接检索结果和业务逻辑的桥梁——通过时间范围过滤最新知识、通过来源过滤可信度、通过文档类型过滤格式差异。

## 基本原理

### 元数据的核心分类

```
文档块元数据
    │
    ├── 溯源元数据（Provenance）
    │   ├── source_doc_id       源文档 ID
    │   ├── source_url          来源 URL
    │   ├── chunk_position      块在文档中的位置（第 N 段）
    │   ├── parent_doc_title    父级文档标题
    │   └── doc_path            文档路径（文件系统或 URL）
    │
    ├── 时间元数据（Temporal）
    │   ├── created_at          创建时间
    │   ├── updated_at          最后更新时间
    │   ├── indexed_at          索引时间
    │   └── effective_date      内容生效日期
    │
    ├── 类型元数据（Typological）
    │   ├── doc_type            文档类型（PDF / Markdown / Code / Email）
    │   ├── mime_type           MIME 类型
    │   ├── language            语言（zh / en / ja）
    │   └── content_category    内容类别（技术文档 / 法律 / 营销）
    │
    ├── 结构元数据（Structural）
    │   ├── heading_level       标题层级（H1 / H2 / H3）
    │   ├── section_name        所在章节名称
    │   ├── chunk_index         块序号
    │   └── total_chunks        总块数
    │
    └── 权限元数据（Access Control）
        ├── visibility          可见性（public / internal / confidential）
        ├── allowed_roles       允许的角色列表
        ├── owner               创建者/部门
        └── access_level        访问级别（1-5）
```

### 元数据结构定义

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional, List

@dataclass
class ChunkMetadata:
    """索引块元数据标准结构"""
    # 溯源
    source_doc_id: str
    source_url: Optional[str] = None
    chunk_index: int = 0
    total_chunks: int = 1
    parent_doc_title: Optional[str] = None
    
    # 时间
    created_at: Optional[datetime] = None
    updated_at: Optional[datetime] = None
    indexed_at: datetime = field(default_factory=datetime.utcnow)
    effective_date: Optional[datetime] = None
    
    # 类型
    doc_type: str = "plain"      # md, pdf, code, email, csv
    mime_type: str = "text/plain"
    language: str = "en"
    content_category: Optional[str] = None
    
    # 结构
    heading_level: int = 0       # 0 = 无标题
    section_name: Optional[str] = None
    heading_path: List[str] = field(default_factory=list)  # ["Ch1", "Sec2", "Subsec3"]
    
    # 权限
    visibility: str = "public"   # public, internal, confidential, secret
    allowed_roles: List[str] = field(default_factory=list)
    owner: Optional[str] = None
    
    # 统计
    token_count: int = 0
    char_count: int = 0
    embedding_model: Optional[str] = None
```

## 背景与演进

- **无元数据时代（2022）**：早期的 RAG 系统（如原始 LangChain 的 VectorStore）仅存储文本和向量，没有结构化元数据。检索结果纯靠语义相似度，无法过滤。用户问"去年的财报"时，检索到的是所有年份的财报混在一起。

- **简单 KV 元数据（2023 初）**：Chroma、Pinecone 等向量数据库开始支持 metadata filtering，但通常是简单的 key-value 对（如 `{"source": "wiki"}`），只支持等值过滤。

- **丰富元数据 + 复杂过滤（2023-2024）**：Qdrant、Weaviate 等支持复杂过滤条件（`AND/OR`、范围查询、全文过滤）。系统开始标注来源、时间、类型、权限等丰富元数据，检索时可以用结构化过滤大幅缩小搜索范围。

- **自动元数据提取（2024）**：用 LLM 自动从文档中提取元数据（标题、作者、摘要、关键词），减少人工标注成本。如 LlamaIndex 的 `MetadataExtractor`。

- **当前趋势**：元数据不再只是"可选的过滤条件"，而是检索策略的核心组成部分。HyDE、RAPTOR 等高级技术利用元数据构建多级索引结构。

## 核心矛盾

**标注丰富度 vs 标注成本**：

```
丰富标注（过滤强）  ───  标注成本高（手动或 LLM 调用）
稀疏标注（成本低）  ───  过滤能力弱（检索精度下降）
```

详细权衡：

| 标注策略 | 过滤能力 | 标注成本 | 检索精度提升 | 维护成本 |
|---------|---------|---------|-------------|---------|
| 无元数据 | ⭐ 无 | ⭐⭐⭐⭐⭐ 零 | 基线 | ⭐⭐⭐⭐⭐ 零 |
| 基础 KV（来源+时间） | ⭐⭐ 弱 | ⭐⭐⭐⭐ 低 | +10-20% | ⭐⭐⭐⭐ 低 |
| 丰富元数据 | ⭐⭐⭐⭐ 强 | ⭐⭐ 中高 | +30-50% | ⭐⭐ 中 |
| LLM 自动标注 | ⭐⭐⭐ 中 | ⭐⭐⭐ 中 | +20-40% | ⭐⭐⭐ 中 |
| 手工精细标注 | ⭐⭐⭐⭐⭐ 最强 | ⭐ 高 | +50%+ | ⭐ 高 |

## 主流优化方向

1. **自动元数据提取**：用 LLM 或 NLP 模型自动从文档中提取关键元数据，结合 Schema 验证确保格式正确。

   ```python
   async def auto_extract_metadata(text: str, llm) -> dict:
       """用 LLM 从文档中提取元数据"""
       prompt = f"""从以下文档中提取元数据，返回 JSON：
       {{
           "title": "文档标题",
           "author": "作者",
           "summary": "一句话摘要 (20字内)",
           "keywords": ["关键词1", "关键词2", ...],
           "doc_type": "文档类型",
           "language": "语言"
       }}
       
       文档内容：
       {text[:2000]}
       """
       response = await llm.complete(prompt)
       return json.loads(response.text)
   ```

2. **元数据过滤优先策略**：将过滤条件分为"硬过滤"（权限、时间范围）和"软过滤"（类型、来源）。先执行硬过滤缩小候选集，再做语义搜索。

   ```python
   def search_with_hard_filters(
       query: str,
       time_range: tuple[datetime, datetime],
       allowed_roles: List[str],
       top_k: int = 10
   ):
       """先硬过滤再语义搜索"""
       # Step 1: 硬过滤（元数据层）
       candidate_ids = metadata_store.query("""
           updated_at BETWEEN ? AND ?
           AND visibility IN ('public', 'internal')
           AND allowed_roles CONTAINS ?
       """, time_range[0], time_range[1], allowed_roles)
       
       # Step 2: 语义搜索（向量层）
       results = vector_store.search(query, filter={"doc_id": candidate_ids})
       return results[:top_k]
   ```

3. **层次化元数据继承**：文档级元数据被子块继承，子块可以覆盖特定字段。避免重复存储公共元数据。

   ```python
   class HierarchicalMetadata:
       def __init__(self):
           self.doc_meta = {}    # 文档级元数据
           self.chunk_overrides = {}  # 块级覆盖
       
       def get_metadata(self, chunk_id: str) -> dict:
           """合并文档级和块级元数据"""
           base = self.doc_meta.copy()
           override = self.chunk_overrides.get(chunk_id, {})
           base.update(override)  # 块级覆盖文档级
           return base
   ```

4. **元数据索引加速**：对高频过滤的元数据字段（权限、部门、时间）建立倒排索引，避免扫描所有记录。

5. **Schema 验证**：使用 JSON Schema / Pydantic 确保元数据格式统一、完整性。

## 能力边界与结果边界

| 维度 | 无元数据 | KV 元数据 | 丰富元数据 |
|------|---------|-----------|-----------|
| 过滤精度 | ⭐ 无过滤 | ⭐⭐ 等值过滤 | ⭐⭐⭐⭐⭐ 复杂过滤 |
| 检索召回提升 | 基线 | +10-20% | +30-50% |
| 存储开销 | 最小 | 少量 | 中等 |
| 查询延迟影响 | 无 | +5-10ms | +10-50ms（复杂过滤） |
| 维护复杂度 | 最低 | 低 | 中 |
| 权限控制 | 不支持 | 支持基础 | 支持精细 |
| 多租户隔离 | 不支持 | 手动 | 原生支持 |

## 与其他内容的区别

- **元数据 vs Embedding**：元数据是离散的结构化标签，embedding 是连续的语义向量。两者互补——元数据做精确过滤，embedding 做语义匹配。**永远先用元数据缩小范围，再用 embedding 精排**。
- **元数据 vs 全文索引**：元数据是高层次的文档属性过滤，全文索引是细粒度的文本关键词匹配。元数据更适合权限、时间、类型等结构化过滤。
- **自动标注 vs 手工标注**：自动标注成本低但准确率有限（80-90%），手工标注准确率高但不可扩展。实践中常用"LLM 自动标注 + 人工抽检"的混合策略。

## 核心优势

元数据注解是 RAG 系统中**性价比最高的优化手段**之一。相比于更换 embedding 模型或重排序器（价格高、收益不确定），添加适当的元数据过滤经常能以极小成本显著提升检索精度。特别是在多租户、权限敏感或时效性要求高的场景中，元数据过滤是保证系统正确性的必要手段。

## 工程优化

1. **过滤下推**：利用向量数据库的内置过滤机制（filter pushdown），让过滤在数据库层执行而非应用层，减少数据传输

2. **元数据缓存**：将高频访问的元数据字段缓存在 Redis，避免每次查询都扫描元数据表

3. **批量标注管线**：在文档入库时建立自动化的元数据标注管线，而非在检索时临时标注

4. **增量更新**：当文档内容不变但权限变化时（如文档从 internal 变为 public），只更新元数据而非重新 embedding

5. **元数据版本化**：当元数据 Schema 演进时保留版本号，支持旧格式兼容

## 场景适配指南

| 场景 | 必要元数据 | 可选元数据 | 过滤策略 |
|------|-----------|-----------|---------|
| 企业内部知识库 | 权限 / 部门 / 文档类型 | 作者 / 评审状态 | 权限硬过滤 → 语义搜索 |
| 学术论文检索 | 发表年份 / 期刊 / 作者 | 引用数 / 关键词 | 时间范围 + 期刊 → 语义搜索 |
| 客服文档 | 产品线 / 版本 / 分类 | 语言 / 标签 | 产品过滤 → 语义搜索 |
| 法律合同审查 | 合同类型 / 生效日期 / 保密等级 | 签署方 / 金额 | 保密等级硬过滤 → 日期范围 → 语义 |
| 代码库检索 | 语言 / 文件路径 / 函数名 | 贡献者 / 最后修改 | 路径过滤 → 语言过滤 → 语义 |

## 代码示例：生产级元数据管线

```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional, List

# Pydantic 模型定义（自带 Schema 验证）
class DocMetadata(BaseModel):
    doc_id: str
    title: str
    source_type: str = Field(pattern="^(pdf|md|code|email|csv)$")
    author: Optional[str] = None
    created_at: datetime
    updated_at: datetime
    visibility: str = Field(default="internal", pattern="^(public|internal|confidential)$")
    department: Optional[str] = None
    tags: List[str] = Field(default_factory=list)
    version: str = "1.0"

# 元数据标注管线
class MetadataPipeline:
    def __init__(self, llm=None):
        self.llm = llm  # LLM 用于自动提取
    
    def process_document(self, doc_path: str, content: str) -> DocMetadata:
        """完整的元数据处理流程"""
        # 1. 自动提取（从文件路径和内容推断）
        auto_meta = self._auto_extract(doc_path, content)
        
        # 2. LLM 增强（对高价值文档）
        if self.llm and len(content) > 100:
            llm_meta = self._llm_extract(content)
            auto_meta.update(llm_meta)
        
        # 3. Schema 验证
        validated = DocMetadata(**auto_meta)
        
        return validated
    
    def _auto_extract(self, path: str, content: str) -> dict:
        """从文件名和路径自动推断"""
        import re, os
        
        meta = {
            "doc_id": hashlib.md5(content.encode()).hexdigest(),
            "created_at": datetime.fromtimestamp(os.path.getctime(path)),
            "updated_at": datetime.fromtimestamp(os.path.getmtime(path)),
        }
        
        # 从路径推断文档类型
        ext = os.path.splitext(path)[1].lower()
        type_map = {".md": "md", ".pdf": "pdf", ".py": "code", ".js": "code"}
        meta["source_type"] = type_map.get(ext, "md")
        
        # 从文件名推断标题
        meta["title"] = os.path.splitext(os.path.basename(path))[0]
        
        return meta
    
    def _llm_extract(self, content: str) -> dict:
        """用 LLM 提取高级元数据"""
        if not self.llm:
            return {}
        # 调用 LLM 提取
        return self.llm.extract_metadata(content[:2000])
```
