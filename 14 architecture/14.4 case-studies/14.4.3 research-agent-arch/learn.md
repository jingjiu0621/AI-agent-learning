# 研究 Agent 架构 (Research Agent Architecture)

## 1. 业务背景

### 1.1 研究 Agent 的兴起

研究 Agent 是一类专门用于自动进行信息收集、文献分析、数据综合和报告生成的 AI 系统。与传统搜索工具不同，研究 Agent 不是简单地返回链接列表，而是主动规划研究路径、跨源交叉验证、并生成结构化报告。

```
  传统搜索                  研究助手                   研究 Agent
+-------------+        +-------------+          +-------------------+
| Google      |        | Perplexity  |          | Deep Research     |
| 返回链接列表 |        | 摘要+引用    |          | 完整研究报告      |
| 用户负责筛选 |        | 单源回答    |          | 多源交叉验证      |
| 无深度分析   |        | 有限推理    |          | 结构化分析        |
+-------------+        +-------------+          +-------------------+
      |                      |                          |
  人找信息                机器整理信息              机器研究信息
```

### 1.2 核心场景

| 场景 | 描述 | 典型用户 |
|------|------|---------|
| 文献综述 | 自动搜索并总结特定领域的学术论文 | 研究生、研究员 |
| 竞品分析 | 收集竞品信息，生成对比报告 | 产品经理、战略分析师 |
| 技术调研 | 评估技术方案，列出优缺点和适用场景 | 技术负责人、架构师 |
| 市场研究 | 分析市场规模、趋势、用户反馈 | 市场分析师、创业者 |
| 事实核查 | 多源验证特定主张的真伪 | 记者、事实核查员 |
| 代码审计 | 分析开源库的安全性、维护状态 | 安全工程师、技术选型者 |

### 1.3 研究 Agent 与通用 Chat Agent 的区别

```
  通用 Chat Agent                  研究 Agent
+-------------------------+    +---------------------------+
| 目标: 回答问题           |    | 目标: 产出结构化研究报告  |
| 单轮或多轮对话          |    | 多步骤研究流程            |
| 依赖模型内部知识        |    | 依赖外部搜索和阅读        |
| 偶尔搜索验证            |    | 系统化多源信息收集        |
| 引用非必须              |    | 引用和溯源是核心要求      |
| 无明确质量指标          |    | 有可验证的质量标准        |
| 实时交互                |    | 批处理为主，异步汇报      |
+-------------------------+    +---------------------------+
```

---

## 2. 系统架构

### 2.1 整体架构概览

```
                          +---------------------------+
                          |    User Request           |
                          |  "研究 RAG 系统在金融    |
                          |   领域的应用现状"        |
                          +-----------+---------------+
                                      |
                          研究意图解析 + 参数提取
                                      |
                                      v
+--------------------------------------------------------------------------+
|                         Research Orchestrator                            |
|  +--------------------------------------------------------------------+  |
|  |  Plan -> Search -> Read -> Verify -> Synthesize -> ... -> Report   |  |
|  +--------------------------------------------------------------------+  |
+--------------------------------------------------------------------------+
        |           |            |              |               |
        v           v            v              v               v
+----------+ +----------+ +----------+ +-----------+ +--------------+
| 搜索规划  | | 网络爬取 | | 阅读理解 | | 交叉验证  | | 报告生成     |
| Search    | | Crawling | | Reading  | | Cross-    | | Report       |
| Planning  | |          | |          | | verify    | | Generation   |
+----------+ +----------+ +----------+ +-----------+ +--------------+
    |             |             |             |              |
    v             v             v             v              v
+----------+ +----------+ +----------+ +-----------+ +--------------+
| 查询分解  | | 页面抓取 | | 长文档   | | 事实抽取  | | 大纲生成     |
| 源选择    | | 内容提取 | | 处理     | | 矛盾检测  | | 逐节撰写     |
| 查询重写  | | 动态渲染 | | 分段     | | 源评分    | | 引用注入     |
| 优先级    | | 反爬规避 | | 摘要     | | 置信度    | | 格式排版     |
+----------+ +----------+ +----------+ +-----------+ +--------------+
    |             |             |             |              |
    +-------------+-------------+-------------+--------------+
                              |
                              v
                    +-------------------+
                    |   Knowledge Store |
                    |   (搜索结果缓存)   |
                    |   (阅读摘要缓存)   |
                    |   (引用源缓存)     |
                    +-------------------+
```

### 2.2 搜索规划模块 (Search Planning)

搜索规划是研究 Agent 的核心智能之一——如何将用户的问题转化为有效的搜索策略。

```
                    +-----------------------------+
                    |     Search Planner          |
                    |                             |
                    |  输入: "微服务架构的优缺点"  |
                    |                             |
                    |  1. 问题分解:               |
                    |     - 微服务定义是什么      |
                    |     - 主要优点有哪些        |
                    |     - 主要缺点有哪些        |
                    |     - 适用场景是什么        |
                    |     - 业界实践案例          |
                    |                             |
                    |  2. 源选择:                 |
                    |     - 技术博客 (高)         |
                    |     - 学术论文 (中)         |
                    |     - 官方文档 (高)         |
                    |     - 社区讨论 (中)         |
                    |     - 视频 (低)             |
                    |                             |
                    |  3. 查询生成:               |
                    |     - "微服务架构 优点 缺点" |
                    |     - "microservices pros   |
                    |        cons"               |
                    |     - "微服务 适用场景"     |
                    |     - "微服务 案例研究"     |
                    |                             |
                    |  4. 优先级排序:             |
                    |     按信息价值/成本排序     |
                    +-----------------------------+
```

**搜索规划器实现:**

```python
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class SearchQuery:
    """搜索查询"""
    query_text: str
    sub_question: str          # 对应的子问题
    target_sources: list[str]  # 目标源类型
    priority: int              # 优先级 1-5 (5最高)
    language: str = 'zh'       # 搜索语言
    max_results: int = 10


@dataclass
class ResearchPlan:
    """研究计划"""
    main_question: str
    sub_questions: list[str]
    queries: list[SearchQuery]
    depth: int                 # 研究深度 1-5
    breadth: int               # 研究广度 1-5
    required_sections: list[str]
    

class SearchPlanner:
    """
    搜索规划器: 将研究问题转化为搜索策略
    
    核心方法:
    - query_decomposition: 问题分解
    - source_selection: 源选择
    - query_generation: 查询生成
    - priority_ranking: 优先级排序
    """
    
    def __init__(self, llm_client):
        self.llm = llm_client
        
        # 源类型评分
        self.source_scores = {
            'academic_paper': {'credibility': 0.9, 'freshness': 0.3, 'coverage': 0.7},
            'official_doc':   {'credibility': 0.8, 'freshness': 0.6, 'coverage': 0.6},
            'tech_blog':      {'credibility': 0.6, 'freshness': 0.7, 'coverage': 0.5},
            'news_article':   {'credibility': 0.5, 'freshness': 0.8, 'coverage': 0.4},
            'forum_discussion': {'credibility': 0.3, 'freshness': 0.5, 'coverage': 0.3},
            'github_repo':    {'credibility': 0.7, 'freshness': 0.8, 'coverage': 0.5},
        }
    
    async def create_plan(self, question: str, 
                           depth: int = 3, breadth: int = 3) -> ResearchPlan:
        """创建完整研究计划"""
        
        # Step 1: 问题分解
        sub_questions = await self._decompose_question(question)
        
        # Step 2: 为每个子问题生成搜索查询
        all_queries = []
        for sq in sub_questions:
            queries = await self._generate_queries(sq, breadth)
            all_queries.extend(queries)
        
        # Step 3: 优先级排序
        ranked_queries = self._rank_by_priority(all_queries, depth)
        
        # Step 4: 生成报告章节
        sections = await self._generate_sections(question, sub_questions)
        
        return ResearchPlan(
            main_question=question,
            sub_questions=sub_questions,
            queries=ranked_queries[:self._budget_for_depth(depth)],
            depth=depth,
            breadth=breadth,
            required_sections=sections,
        )
    
    async def _decompose_question(self, question: str) -> list[str]:
        """使用 LLM 将问题分解为子问题"""
        prompt = f"""请将以下研究问题分解为 3-6 个独立的子问题。
每个子问题应该能从不同角度回答主问题。

主问题: {question}

以 JSON 数组格式返回子问题列表:
["子问题1", "子问题2", ...]"""
        
        response = await self.llm.generate(prompt)
        # 解析 JSON 响应
        import json, re
        match = re.search(r'\[.*?\]', response, re.DOTALL)
        if match:
            return json.loads(match.group())
        return [question]  # 回退: 原始问题
    
    async def _generate_queries(self, sub_question: str, 
                                 breadth: int) -> list[SearchQuery]:
        """为子问题生成搜索查询"""
        queries = []
        
        # 生成不同角度和语言的查询
        prompts = [
            f"搜索中文资料: {sub_question}",
            f"搜索英文资料: {sub_question}",
            f"搜索最新进展: {sub_question}",
        ]
        
        for i, query_text in enumerate(prompts[:breadth]):
            queries.append(SearchQuery(
                query_text=query_text,
                sub_question=sub_question,
                target_sources=['tech_blog', 'academic_paper', 'official_doc'],
                priority=5 - i,  # 第一个查询优先级最高
            ))
        
        return queries
    
    def _rank_by_priority(self, queries: list[SearchQuery],
                           depth: int) -> list[SearchQuery]:
        """按优先级排序查询"""
        # 按 priority 降序排列
        return sorted(queries, key=lambda q: -q.priority)
    
    def _budget_for_depth(self, depth: int) -> int:
        """根据深度计算搜索预算"""
        return depth * 5  # 深度 1->5次, 深度 5->25次搜索
    
    async def _generate_sections(self, question: str,
                                   sub_questions: list[str]) -> list[str]:
        """生成报告章节结构"""
        prompt = f"""基于以下研究问题和子问题，生成报告章节列表。

主问题: {question}
子问题: {sub_questions}

返回 JSON 数组: ["章节1", "章节2", ...]"""
        
        response = await self.llm.generate(prompt)
        import json, re
        match = re.search(r'\[.*?\]', response, re.DOTALL)
        if match:
            return json.loads(match.group())
        return ["概述", "详细分析", "结论"]
```

### 2.3 网络爬取模块 (Web Crawling)

```
                    +-----------------------------+
                    |      Web Crawler            |
                    +-----------------------------+
                    |  +-----------------------+  |
                    |  | URL 队列管理器        |  |
                    |  | 去重 -> 优先级 -> 限速 |  |
                    |  +-----------------------+  |
                    |  +-----------------------+  |
                    |  | 获取层                |  |
                    |  | HTTP 请求 + 动态渲染   |  |
                    |  | 重试 + 代理 + Cookie   |  |
                    |  +-----------------------+  |
                    |  +-----------------------+  |
                    |  | 解析层                |  |
                    |  | HTML 解析              |  |
                    |  | 正文提取 (Readability) |  |
                    |  | Markdown 转换          |  |
                    |  +-----------------------+  |
                    |  +-----------------------+  |
                    |  | 后处理                |  |
                    |  | 去噪                   |  |
                    |  | 链接提取               |  |
                    |  | 元数据提取             |  |
                    |  +-----------------------+  |
                    +-----------------------------+
```

**网页抓取与解析:**

```python
import hashlib
import asyncio
from dataclasses import dataclass, field
from typing import Optional
from urllib.parse import urlparse


@dataclass
class CrawledDocument:
    """爬取到的文档"""
    url: str
    title: str
    content: str                # 纯文本内容
    markdown: str               # Markdown 格式
    metadata: dict = field(default_factory=dict)
    source_type: str = 'web'    # 'web' | 'pdf' | 'video_transcript'
    credibility_score: float = 0.5
    word_count: int = 0
    
    def __post_init__(self):
        self.word_count = len(self.content)
    
    @property
    def doc_id(self) -> str:
        return hashlib.md5(self.url.encode()).hexdigest()[:12]


class WebCrawler:
    """
    网络爬取器
    
    核心功能:
    - 从搜索查询到 URL 收集
    - 智能去重 (URL 规范化 + 内容指纹)
    - 自适应速率限制
    - 重试和错误处理
    """
    
    def __init__(self, user_agent: str = "ResearchAgent/1.0"):
        self.user_agent = user_agent
        self.seen_urls: set[str] = set()
        self.seen_content_hashes: set[str] = set()
        self.domain_delays: dict[str, float] = {}
        self.max_concurrent = 5
        self._semaphore = asyncio.Semaphore(self.max_concurrent)
    
    async def crawl(self, query: SearchQuery, 
                     search_api) -> list[CrawledDocument]:
        """从搜索查询开始爬取"""
        
        # 1. 调用搜索 API 获取 URL
        search_results = await search_api.search(query.query_text, 
                                                  max_results=10)
        
        # 2. 过滤已访问的 URL
        new_urls = []
        for result in search_results:
            url = self._normalize_url(result['url'])
            if url not in self.seen_urls:
                self.seen_urls.add(url)
                new_urls.append({
                    'url': url,
                    'title': result.get('title', ''),
                    'snippet': result.get('snippet', ''),
                })
        
        # 3. 并发抓取
        tasks = [self._fetch_single(item) for item in new_urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # 4. 过滤重复内容 (基于内容指纹)
        documents = []
        for r in results:
            if isinstance(r, CrawledDocument):
                content_hash = hashlib.md5(r.content[:500].encode()).hexdigest()
                if content_hash not in self.seen_content_hashes:
                    self.seen_content_hashes.add(content_hash)
                    documents.append(r)
        
        return documents
    
    async def _fetch_single(self, item: dict) -> Optional[CrawledDocument]:
        """获取单个 URL """
        url = item['url']
        domain = urlparse(url).netloc
        
        async with self._semaphore:
            # 域级速率限制
            delay = self.domain_delays.get(domain, 1.0)
            await asyncio.sleep(delay)
            
            try:
                # 实际使用 httpx 或 aiohttp
                # 这里简化为模拟获取
                html_content = await self._http_get(url)
                
                # 解析 HTML -> 正文
                doc = self._parse_html(html_content, url)
                doc.credibility_score = self._score_source_credibility(domain)
                
                # 更新域延迟（如果服务器响应慢，增加延迟）
                self.domain_delays[domain] = min(delay * 1.5, 5.0)
                
                return doc
                
            except Exception as e:
                # 抓取失败，降低域优先级
                self.domain_delays[domain] = delay * 2
                return None
    
    def _normalize_url(self, url: str) -> str:
        """URL 规范化: 去重关键"""
        parsed = urlparse(url)
        # 移除 fragment, 排序 query params
        return f"{parsed.scheme}://{parsed.netloc}{parsed.path}"
    
    async def _http_get(self, url: str) -> str:
        """HTTP GET 请求 (简化)"""
        # 实际使用 httpx.AsyncClient
        return "<html>...</html>"
    
    def _parse_html(self, html: str, url: str) -> CrawledDocument:
        """将 HTML 解析为结构化文档"""
        # 实际使用 readability-lxml 或 trafilatura
        # 这里做简化处理
        from bs4 import BeautifulSoup
        soup = BeautifulSoup(html, 'html.parser')
        
        title = soup.title.string if soup.title else url
        
        # 正文提取算法
        # 1. 移除 script/style/nav/footer
        # 2. 提取 <article> 或 <main>
        # 3. 计算文本密度，选取最密集区域
        # 4. 转换为 Markdown
        
        return CrawledDocument(
            url=url,
            title=title,
            content="extracted text...",
            markdown="# Title\n\nContent...",
            metadata={'fetch_time': asyncio.get_event_loop().time()},
        )
    
    def _score_source_credibility(self, domain: str) -> float:
        """评估源可信度"""
        credible_domains = {
            'arxiv.org': 0.9, 'scholar.google.com': 0.9,
            'github.com': 0.8, 'stackoverflow.com': 0.7,
            'medium.com': 0.5, 'zhihu.com': 0.5,
            'blog.csdn.net': 0.4, 'juejin.cn': 0.5,
        }
        return credible_domains.get(domain, 0.5)
```

### 2.4 阅读理解模块 (Reading)

```
                    +-----------------------------+
                    |     Reading Pipeline        |
                    |                             |
                    |  文档 -> 分段 -> 摘要 -> 提取 |
                    |                             |
                    |  +-----------------------+  |
                    |  | 文档分段              |  |
                    |  | - 按标题分段          |  |
                    |  | - 按段落分段          |  |
                    |  | - 按 Token 数分段     |  |
                    |  +-----------------------+  |
                    |  +-----------------------+  |
                    |  | 并行摘要              |  |
                    |  | - 每段独立摘要         |  |
                    |  | - 提取关键数据点       |  |
                    |  +-----------------------+  |
                    |  +-----------------------+  |
                    |  | 信息聚合              |  |
                    |  | - 合并重叠信息         |  |
                    |  | - 识别矛盾             |  |
                    |  | - 结构化存储           |  |
                    |  +-----------------------+  |
                    +-----------------------------+
```

**长文档处理分段策略:**

```python
@dataclass
class DocumentChunk:
    """文档片段"""
    chunk_id: str
    doc_id: str
    content: str
    heading: str
    chunk_index: int
    token_count: int
    key_facts: list[str] = field(default_factory=list)


class DocumentReader:
    """
    文档阅读器: 处理长文档并提取关键信息
    
    核心策略:
    - 分层分段: 按文档结构 (标题 -> 段落) 分段
    - 并行处理: 各段独立摘要
    - 增量理解: 先读标题 -> 再读首段 -> 必要时读全文
    """
    
    def __init__(self, llm_client, max_chunk_tokens: int = 3000):
        self.llm = llm_client
        self.max_chunk_tokens = max_chunk_tokens
    
    async def read_document(self, doc: CrawledDocument) -> list[DocumentChunk]:
        """阅读文档，返回结构化片段"""
        
        # 1. 文档分段
        chunks = self._chunk_document(doc)
        
        # 2. 并行提取关键事实
        tasks = [self._extract_facts(chunk) for chunk in chunks]
        fact_lists = await asyncio.gather(*tasks)
        
        for chunk, facts in zip(chunks, fact_lists):
            chunk.key_facts = facts
        
        return chunks
    
    def _chunk_document(self, doc: CrawledDocument) -> list[DocumentChunk]:
        """将文档按结构分段"""
        import re
        
        # 按标题分段
        sections = re.split(r'\n#{1,3}\s+', doc.markdown)
        
        chunks = []
        for i, section in enumerate(sections):
            lines = section.split('\n')
            heading = lines[0].strip() if lines else "无标题"
            content = '\n'.join(lines[1:]).strip() if len(lines) > 1 else ""
            
            # 粗略 token 计数 (中文: ~1.5 字/token)
            token_count = len(content) * 1.5
            
            if token_count > self.max_chunk_tokens:
                # 过长的段再按段落切分
                sub_chunks = self._split_long_section(
                    content, heading, i, doc.doc_id
                )
                chunks.extend(sub_chunks)
            else:
                chunks.append(DocumentChunk(
                    chunk_id=f"{doc.doc_id}_{i}",
                    doc_id=doc.doc_id,
                    content=content[:500],  # 仅保留前 500 字符用于预览
                    heading=heading,
                    chunk_index=i,
                    token_count=int(token_count),
                ))
        
        return chunks
    
    def _split_long_section(self, content: str, heading: str,
                             section_idx: int, doc_id: str) -> list[DocumentChunk]:
        """拆分过长的段落"""
        paragraphs = content.split('\n\n')
        chunks = []
        current_chunk = []
        current_tokens = 0
        
        for i, para in enumerate(paragraphs):
            para_tokens = len(para) * 1.5
            if current_tokens + para_tokens > self.max_chunk_tokens and current_chunk:
                # 保存当前 chunk
                chunk_text = '\n\n'.join(current_chunk)
                chunks.append(DocumentChunk(
                    chunk_id=f"{doc_id}_{section_idx}_{len(chunks)}",
                    doc_id=doc_id,
                    content=chunk_text[:500],
                    heading=f"{heading} (续)",
                    chunk_index=section_idx,
                    token_count=int(current_tokens),
                ))
                current_chunk = [para]
                current_tokens = para_tokens
            else:
                current_chunk.append(para)
                current_tokens += para_tokens
        
        # 处理最后一个 chunk
        if current_chunk:
            chunk_text = '\n\n'.join(current_chunk)
            chunks.append(DocumentChunk(
                chunk_id=f"{doc_id}_{section_idx}_{len(chunks)}",
                doc_id=doc_id,
                content=chunk_text[:500],
                heading=f"{heading} (续)",
                chunk_index=section_idx,
                token_count=int(current_tokens),
            ))
        
        return chunks
    
    async def _extract_facts(self, chunk: DocumentChunk) -> list[str]:
        """从片段中提取关键事实"""
        prompt = f"""从以下文本中提取 3-5 个关键事实。
只提取客观的可验证信息，不要添加你的知识。
用简洁的中文列出，每行一个事实。

文本: {chunk.content[:2000]}"""
        
        response = await self.llm.generate(prompt)
        facts = [line.strip().lstrip('- ') 
                 for line in response.split('\n')
                 if line.strip().startswith('-')]
        return facts[:5]
```

### 2.5 交叉验证模块 (Cross-Verification)

```
                    +-----------------------------+
                    |   Cross-Verification Engine |
                    |                             |
                    |  事实一致性矩阵              |
                    |                             |
                    |        Source A  Source B   |
                    | 事实1   ✓         ✓        |  一致 -> 高置信度
                    | 事实2   ✓         ✗        |  矛盾 -> 标记 + 调查
                    | 事实3   ✗         -        |  缺失 -> 低置信度
                    |                             |
                    +-----------------------------+
                              |
                    +---------+---------+
                    |                   |
                    v                   v
            +-------------+    +--------------+
            | 矛盾解决策略  |    | 置信度评估   |
            | 1. 查证更多源 |    | 1. 源权威性  |
            | 2. 评估源权威 |    | 2. 一致数量  |
            | 3. 记录双方   |    | 3. 时效性    |
            +-------------+    +--------------+
```

**交叉验证实现:**

```python
from dataclasses import dataclass, field
from enum import Enum


class FactStatus(Enum):
    CONFIRMED = "confirmed"       # 多源一致确认
    CONTRADICTED = "contradicted" # 存在矛盾
    SINGLE_SOURCE = "single_source" # 仅单源
    UNCERTAIN = "uncertain"       # 信息不足


@dataclass
class Fact:
    """事实单元"""
    statement: str
    sources: list[str]          # 来源 URL 列表
    status: FactStatus = FactStatus.UNCERTAIN
    confidence: float = 0.0      # 0.0 - 1.0
    contradictory_evidence: list[str] = field(default_factory=list)
    
    def add_source(self, url: str, is_supporting: bool = True):
        if is_supporting:
            self.sources.append(url)
        else:
            self.contradictory_evidence.append(url)
        self._recompute_confidence()
    
    def _recompute_confidence(self):
        """重新计算置信度"""
        num_supporting = len(self.sources)
        num_contradicting = len(self.contradictory_evidence)
        
        if num_supporting == 0:
            self.confidence = 0.0
            self.status = FactStatus.UNCERTAIN
        elif num_contradicting > 0:
            # 有矛盾时，置信度 = 支持率
            ratio = num_supporting / (num_supporting + num_contradicting)
            self.confidence = ratio * 0.7  # 有矛盾则最高 0.7
            self.status = FactStatus.CONTRADICTED
        elif num_supporting == 1:
            self.confidence = 0.5
            self.status = FactStatus.SINGLE_SOURCE
        else:
            # 多源一致，置信度递增
            self.confidence = min(0.5 + num_supporting * 0.1, 0.95)
            self.status = FactStatus.CONFIRMED


class CrossVerifier:
    """
    交叉验证器: 多源事实校验
    
    核心流程:
    1. 从各文档中提取事实
    2. 事实对齐 (同一事实的不同表述)
    3. 跨源一致性检查
    4. 矛盾检测与处理
    5. 置信度评分
    """
    
    def __init__(self, llm_client):
        self.llm = llm_client
        self.facts: dict[str, Fact] = {}  # 规范化的事实 -> Fact
    
    async def verify(self, documents: list[CrawledDocument],
                      existing_facts: Optional[list[Fact]] = None) -> list[Fact]:
        """对文档集合进行交叉验证"""
        
        # 1. 从每个文档提取事实
        all_new_facts = []
        for doc in documents:
            doc_facts = await self._extract_facts_from_doc(doc)
            all_new_facts.extend(doc_facts)
        
        # 2. 事实归一化和对齐
        for new_fact in all_new_facts:
            normalized = await self._normalize_fact(new_fact)
            
            if normalized in self.facts:
                # 已有相同事实，添加来源
                self.facts[normalized].add_source(new_fact.sources[0])
            else:
                # 新事实
                self.facts[normalized] = Fact(
                    statement=new_fact.statement,
                    sources=new_fact.sources
                )
        
        return list(self.facts.values())
    
    async def _extract_facts_from_doc(self, doc: CrawledDocument) -> list[Fact]:
        """从文档中提取事实"""
        prompt = f"""从以下文档中提取客观的事实陈述。
只提取文档中明确陈述的内容，不要推断或添加。
对每个事实，判断它是"支持性陈述"还是"否定性陈述"。

文档标题: {doc.title}
文档内容:
{doc.content[:3000]}

以 JSON 格式返回:
[
  {{"statement": "事实描述", "is_supporting": true}},
  ...
]"""
        
        response = await self.llm.generate(prompt)
        import json, re
        match = re.search(r'\[.*?\]', response, re.DOTALL)
        if match:
            items = json.loads(match.group())
            return [
                Fact(statement=item['statement'], 
                     sources=[doc.url])
                for item in items
            ]
        return []
    
    async def _normalize_fact(self, fact: Fact) -> str:
        """事实归一化: 不同表述映射到同一规范形式"""
        prompt = f"""将以下事实陈述改写为标准陈述形式。
保持核心含义不变，只做规范化。

原陈述: {fact.statement}

改写:"""
        
        response = await self.llm.generate(prompt)
        return response.strip()
    
    def get_confirmed_facts(self, min_confidence: float = 0.7) -> list[Fact]:
        """获取高置信度的事实"""
        return [
            f for f in self.facts.values()
            if f.confidence >= min_confidence
        ]
    
    def get_contradictions(self) -> list[tuple[Fact, Fact]]:
        """获取矛盾的事实对"""
        # 寻找对同一主题持相反观点的事实
        contradictions = []
        facts_list = list(self.facts.values())
        
        for i in range(len(facts_list)):
            for j in range(i + 1, len(facts_list)):
                if self._is_contradictory(facts_list[i], facts_list[j]):
                    contradictions.append((facts_list[i], facts_list[j]))
        
        return contradictions
    
    def _is_contradictory(self, fact_a: Fact, fact_b: Fact) -> bool:
        """判断两个事实是否矛盾"""
        # 简化实现: 检查是否在同一话题上持相反立场
        # 实际需要使用 NLP 语义对比
        return bool(fact_a.contradictory_evidence and 
                    fact_b.contradictory_evidence)
```

### 2.6 报告生成模块 (Report Generation)

```
                    +-----------------------------+
                    |     Report Generator        |
                    +-----------------------------+
                    |                             |
                    |  输入: Facts + Documents    |
                    |                             |
                    |  1. 大纲生成                |
                    |     LLM 根据研究问题        |
                    |     和收集的事实生成大纲     |
                    |                             |
                    |  2. 分节撰写                |
                    |     每节独立生成            |
                    |     并行 LLM 调用           |
                    |                             |
                    |  3. 引用注入                |
                    |     [1][2][3] -> 超链接     |
                    |     引用去重                |
                    |                             |
                    |  4. 交叉检查                |
                    |     确保无矛盾              |
                    |     确保引用准确            |
                    |                             |
                    |  5. 格式排版                |
                    |     Markdown -> 最终格式    |
                    |                             |
                    +-----------------------------+
```

**报告生成器:**

```python
from dataclasses import dataclass, field


@dataclass
class ReportSection:
    """报告章节"""
    title: str
    content: str
    citations: list[str] = field(default_factory=list)
    subsections: list['ReportSection'] = field(default_factory=list)


@dataclass
class ResearchReport:
    """完整研究报告"""
    title: str
    summary: str
    sections: list[ReportSection]
    citations: list[dict]  # [{'id': '1', 'url': '...', 'title': '...'}, ...]
    metadata: dict = field(default_factory=dict)
    
    def to_markdown(self) -> str:
        """转为 Markdown 格式"""
        lines = [f"# {self.title}\n", f"\n{self.summary}\n"]
        
        for section in self.sections:
            lines.append(f"\n## {section.title}\n")
            lines.append(section.content)
            
            for sub in section.subsections:
                lines.append(f"\n### {sub.title}\n")
                lines.append(sub.content)
        
        lines.append("\n## 参考文献\n")
        for cite in self.citations:
            lines.append(
                f"- [{cite['id']}] {cite['title']} - {cite['url']}\n"
            )
        
        return ''.join(lines)


class ReportGenerator:
    """
    报告生成器: 将研究结果综合为结构化报告
    
    设计特点:
    - 分节独立生成，可并行
    - 每节底部附加来源引用
    - 最终进行交叉一致性检查
    """
    
    def __init__(self, llm_client):
        self.llm = llm_client
        self.citation_registry: dict[str, dict] = {}  # url -> citation info
    
    async def generate(self, plan: ResearchPlan,
                        facts: list[Fact],
                        documents: list[CrawledDocument]) -> ResearchReport:
        """生成完整研究报告"""
        
        # 1. 构建引用注册表
        self._build_citation_registry(documents)
        
        # 2. 生成执行摘要
        summary = await self._generate_summary(plan, facts)
        
        # 3. 分节生成
        section_tasks = [
            self._generate_section(section_title, plan, facts, documents)
            for section_title in plan.required_sections
        ]
        sections = await asyncio.gather(*section_tasks)
        
        # 4. 交叉一致性检查
        sections = await self._cross_check_sections(sections)
        
        # 5. 组装最终报告
        citations = [
            {'id': info['id'], 'url': url, 'title': info['title']}
            for url, info in self.citation_registry.items()
        ]
        
        report = ResearchReport(
            title=plan.main_question,
            summary=summary,
            sections=sections,
            citations=citations,
            metadata={
                'num_sources': len(documents),
                'num_facts': len(facts),
                'depth': plan.depth,
            }
        )
        
        return report
    
    def _build_citation_registry(self, documents: list[CrawledDocument]):
        """构建引用注册表"""
        for i, doc in enumerate(documents, 1):
            if doc.url not in self.citation_registry:
                self.citation_registry[doc.url] = {
                    'id': str(i),
                    'title': doc.title,
                    'url': doc.url,
                    'used': False,
                }
    
    def _cite(self, url: str) -> str:
        """生成引用标记"""
        info = self.citation_registry.get(url)
        if info:
            info['used'] = True
            return f"[{info['id']}]"
        return "[?]"
    
    async def _generate_summary(self, plan: ResearchPlan,
                                  facts: list[Fact]) -> str:
        """生成执行摘要"""
        confirmed = [f for f in facts if f.status == FactStatus.CONFIRMED]
        
        prompt = f"""基于以下研究问题和高置信度事实，生成 200 字以内的执行摘要。

研究问题: {plan.main_question}

关键发现:
{chr(10).join(f'- {f.statement}' for f in confirmed[:10])}

执行摘要:"""
        
        response = await self.llm.generate(prompt)
        return response.strip()
    
    async def _generate_section(self, title: str, plan: ResearchPlan,
                                 facts: list[Fact],
                                 docs: list[CrawledDocument]) -> ReportSection:
        """生成单个章节"""
        # 过滤与该章节相关的事实
        related_facts = await self._filter_related_facts(title, facts)
        
        # 找到相关文档
        related_docs = [d for d in docs 
                        if any(d.url in f.sources for f in related_facts)]
        
        # 构建上下文
        doc_excerpts = []
        for doc in related_docs[:5]:
            excerpt = doc.content[:500]
            doc_excerpts.append(
                f"来源 [{self.citation_registry[doc.url]['id']}]: {doc.title}\n"
                f"{excerpt}\n"
            )
        
        context = '\n'.join(doc_excerpts)
        facts_text = '\n'.join(
            f"- {f.statement} {self._cite(f.sources[0])}" 
            for f in related_facts[:10]
        )
        
        prompt = f"""基于以下信息和事实，撰写研究报告章节。

章节标题: {title}
研究问题: {plan.main_question}

相关事实:
{facts_text}

相关文档摘录:
{context}

请撰写这一章节，包含:
1. 客观描述各个方面
2. 标注引用 [1][2]...
3. 如果存在矛盾观点，明确指出
4. 给出平衡的总结

章节内容:"""
        
        response = await self.llm.generate(prompt)
        
        return ReportSection(
            title=title,
            content=response.strip(),
            citations=list(self.citation_registry.keys()),
        )
    
    async def _filter_related_facts(self, section_title: str,
                                      facts: list[Fact]) -> list[Fact]:
        """过滤与章节相关的事实"""
        prompt = f"""以下事实列表中，哪些与"{section_title}"这个章节直接相关？
返回相关事实的序号 (从 0 开始)，用逗号分隔。

事实列表:
{chr(10).join(f'{i}: {f.statement}' for i, f in enumerate(facts))}"""
        
        response = await self.llm.generate(prompt)
        import re
        indices = re.findall(r'\d+', response)
        related = [facts[int(i)] for i in indices if int(i) < len(facts)]
        return related or facts[:5]  # 容错: 至少返回 5 条
    
    async def _cross_check_sections(self, 
                                      sections: list[ReportSection]) -> list[ReportSection]:
        """跨章节一致性检查"""
        # 检查不同章节中对同一事实的描述是否一致
        # 简化实现: 直接返回
        return sections
```

---

## 3. 核心设计决策

### 3.1 Planning-First vs Iterative Refinement

```
     Planning-First                      Iterative Refinement
+---------------------------+      +-----------------------------+
|  Step 1: 制定完整计划      |      |  Step 1: 初始搜索           |
|  - 问题分解               |      |  Step 2: 阅读 + 发现新方向  |
|  - 搜索策略               |      |  Step 3: 调整搜索策略       |
|  - 章节结构               |      |  Step 4: 深入研究           |
|                           |      |  Step 5: 如果不够 -> Step 3 |
|  Step 2: 执行搜索         |      |  Step 6: 综合报告           |
|                           |      |                             |
|  Step 3: 阅读与分析       |      |  优点: 灵活, 能发现意外     |
|                           |      |  优点: 适合探索性研究       |
|  Step 4: 生成报告         |      |  缺点: 可能无限循环         |
|                           |      |  缺点: 成本不可预测         |
|  优点: 可预测, 成本可控   |      |                             |
|  优点: 结构清晰           |      |                             |
|  缺点: 不够灵活           |      |                             |
+---------------------------+      +-----------------------------+
```

**混合策略——建议方案:**

```
  混合研究策略

  阶段 1: 快速侦察 (Fast Reconnaissance)
  ├── 1-2 次宽泛搜索 -> 全面了解领域
  ├── 阅读搜索结果摘要 -> 识别关键方向
  └── 输出: 领域地图 + 关键问题

  阶段 2: 计划制定 (Planning)
  ├── 基于侦察结果制定详细计划
  ├── 问题分解 + 源选择 + 优先级
  └── 输出: ResearchPlan

  阶段 3: 深度研究 (Deep Research)
  ├── 按计划执行搜索和阅读
  ├── 每个子问题独立研究
  ├── 支持中途调整方向 (限幅度)
  └── 输出: 每个子问题的研究发现

  阶段 4: 综合生成 (Synthesis)
  ├── 汇总所有研究发现
  ├── 矛盾检测 + 置信度评估
  └── 输出: 最终报告
```

### 3.2 矛盾信息处理

研究 Agent 必须处理真实世界中常见的矛盾信息。

```
  矛盾类型:
  
  类型 A: 事实性矛盾
  源1: "GPT-4 有 1.7 万亿参数"
  源2: "GPT-4 有 1 万亿参数"
  处理: 查官方文档 + 标注不确定
  
  类型 B: 观点性矛盾
  源1: "微服务适合创业公司"
  源2: "微服务不适合创业公司"
  处理: 呈现双方观点 + 分析前提和语境
  
  类型 C: 时效性矛盾
  源1 (2022): "React 是前端首选框架"
  源2 (2024): "React 市场份额在下降"
  处理: 按时间线呈现
```

**矛盾处理策略:**

```python
class ContradictionHandler:
    """
    矛盾处理策略
    
    四级处理流程:
    Level 1: 自动决策 (高置信度)
    Level 2: 额外搜索 (中等置信度)
    Level 3: 人工介入 (存疑)
    Level 4: 如实呈现 (无法解决)
    """
    
    async def handle(self, fact_a: Fact, fact_b: Fact) -> dict:
        """处理矛盾"""
        
        # Step 1: 检查时效性
        temporal = self._check_temporal_conflict(fact_a, fact_b)
        if temporal:
            return {
                'resolution': 'temporal',
                'message': f"因时间不同：{temporal['old']} vs {temporal['new']}",
                'facts': [fact_a, fact_b],
            }
        
        # Step 2: 检查源权威性
        credibility_gap = self._check_credibility_gap(fact_a, fact_b)
        if credibility_gap > 0.3:
            winner = fact_a if credibility_gap > 0 else fact_b
            return {
                'resolution': 'credibility',
                'message': f"采用权威来源：{winner.sources[0]}",
                'facts': [winner],
            }
        
        # Step 3: 尝试额外搜索验证
        extra_evidence = await self._search_for_verification(fact_a, fact_b)
        
        if extra_evidence:
            return {
                'resolution': 'extra_evidence',
                'message': f"额外搜索找到 {len(extra_evidence)} 个支持源",
                'facts': extra_evidence,
            }
        
        # Step 4: 如实呈现矛盾
        return {
            'resolution': 'present_both',
            'message': "存在矛盾观点，置信度不足以判断",
            'facts': [fact_a, fact_b],
            'note': "建议读者自行核实",
        }
    
    def _check_temporal_conflict(self, a: Fact, b: Fact) -> Optional[dict]:
        """检查是否因时间不同导致矛盾"""
        # 从元数据中提取时间信息
        # 简化实现
        return None
    
    def _check_credibility_gap(self, a: Fact, b: Fact) -> float:
        """计算源权威性差距"""
        # 正数表示 a 更可信，负数表示 b 更可信
        return a.confidence - b.confidence
    
    async def _search_for_verification(self, a: Fact, b: Fact) -> list[Fact]:
        """额外搜索寻找第三方验证"""
        # 实际实现中: 用矛盾的话题进行额外搜索
        return []
```

### 3.3 引用准确性与溯源追踪

引用是研究 Agent 的生命线——没有准确引用的研究毫无价值。

```
  引用追踪链:
  
  原始网页
    └── 爬取 + 解析
         └── 段落全文 (保留原始上下文)
              └── LLM 提取事实
                   └── 事实 -> 源 URL + 段落位置
                        └── 报告引用 [1]
                             └── 参考文献表
  
  关键设计:
  - 每个事实都绑定到具体 URL
  - 引用 ID 在报告生成时分配
  - 最终校验: 每个引用都有对应的参考文献
  - 无引用的事实标记为 "待验证"
```

**引用管理器:**

```python
@dataclass
class Citation:
    """引用条目"""
    id: str
    url: str
    title: str
    author: Optional[str] = None
    publish_date: Optional[str] = None
    access_date: str = ""
    snippet: str = ""       # 引用原文摘要
    fact_statements: list[str] = field(default_factory=list)


class CitationManager:
    """
    引用管理器: 追踪所有引用
    
    核心职责:
    1. URL 去重: 同一 URL 分配相同 ID
    2. 引用完整性: 确保每个 [N] 都有对应条目
    3. 引用格式: 统一格式 (APA/MLA/GB/T)
    4. 来源追溯: 从报告中的引用追溯到原始网页
    """
    
    def __init__(self):
        self.citations: dict[str, Citation] = {}  # url -> Citation
        self._next_id = 1
    
    def register_source(self, url: str, title: str) -> str:
        """注册来源，返回引用 ID"""
        if url not in self.citations:
            citation_id = str(self._next_id)
            self._next_id += 1
            self.citations[url] = Citation(
                id=citation_id,
                url=url,
                title=title,
                access_date=asyncio.get_event_loop().time(),
            )
        return self.citations[url].id
    
    def verify_integrity(self, report_text: str) -> list[str]:
        """验证报告中的引用完整性"""
        import re
        issues = []
        
        # 提取所有引用标记 [1], [2,3], [1-3]
        cited_ids = set()
        for match in re.finditer(r'\[([\d,\-\s]+)\]', report_text):
            parts = match.group(1).split(',')
            for part in parts:
                part = part.strip()
                if '-' in part:
                    start, end = part.split('-')
                    for i in range(int(start), int(end) + 1):
                        cited_ids.add(str(i))
                else:
                    cited_ids.add(part)
        
        # 检查是否有未注册的引用
        registered_ids = {c.id for c in self.citations.values()}
        
        for cid in cited_ids:
            if cid not in registered_ids:
                issues.append(f"引用 [{cid}] 在报告中出现但无对应文献")
        
        # 检查是否有未使用的文献
        for c in self.citations.values():
            if c.id not in cited_ids:
                issues.append(f"文献 [{c.id}] 已注册但未在报告中引用")
        
        return issues
```

### 3.4 深度 vs 广度控制

```
  研究策略图谱:
  
  广度优先 (BFS)                    深度优先 (DFS)
+--------------------------+    +--------------------------+
|  覆盖更多子主题           |    | 深入单个子主题           |
|  每个主题浅层了解         |    | 包含全面细节             |
|  更多搜索查询             |    | 更少但更精准的查询       |
|  更少的阅读深度           |    | 每个源完整阅读           |
|  适合: 技术选型           |    | 适合: 文献综述           |
|  调研、竞品概览           |    | 学术研究、根因分析       |
+--------------------------+    +--------------------------+

  参数调节:
  
  研究深度 (depth: 1-5):
  1 -> 仅搜索摘要, 不深入阅读
  3 -> 搜索 + 前 5 个结果全文阅读
  5 -> 搜索 + 所有结果全文 + 跟踪引用链接
  
  研究广度 (breadth: 1-5):
  1 -> 仅 1 个搜索角度
  3 -> 3 个不同角度的搜索
  5 -> 5+ 个角度 + 多语言搜索
```

**自适应的深度控制:**

```python
class AdaptiveDepthController:
    """
    自适应深度控制器
    
    根据搜索结果质量动态决定是否深入:
    - 如果前 3 个结果已充分覆盖 -> 停止
    - 如果结果存在矛盾 -> 需要更深入
    - 如果结果质量低 -> 换查询词
    """
    
    def __init__(self, llm_client):
        self.llm = llm_client
    
    async def should_deepen(self, sub_question: str,
                             documents: list[CrawledDocument],
                             current_depth: int) -> tuple[bool, str]:
        """判断是否需要加深研究"""
        
        if current_depth >= 5:
            return False, "已达最大深度"
        
        # 评估已收集信息的充分性
        prompt = f"""基于已经收集到以下资料，判断是否需要继续深入研究。

子问题: {sub_question}

已收集资料的摘要:
{chr(10).join(f'- {d.title}: {d.content[:200]}' for d in documents[:5])}

评估标准:
1. 是否已有足够信息回答该子问题? (是/否)
2. 是否存在矛盾或不确定的信息? (是/否)
3. 是否需要更多来源来验证? (是/否)

只需返回 JSON: {{"sufficient": true/false, "reason": "..."}}"""
        
        response = await self.llm.generate(prompt)
        import json, re
        match = re.search(r'\{.*\}', response, re.DOTALL)
        if match:
            result = json.loads(match.group())
            if not result.get('sufficient', True):
                return True, result.get('reason', '信息不足')
        
        return False, "信息已充分"
```

---

## 4. 技术栈

| 组件 | 技术选型 | 选型理由 |
|------|---------|---------|
| **LLM** | Claude 4 / GPT-4o | 长上下文窗口, 强推理能力, 支持工具调用 |
| **搜索 API** | SerpAPI / Bing Search API / Tavily | 结构化搜索结果, 支持新闻/学术过滤 |
| **网页抓取** | httpx + trafilatura / readability | 异步高性能, 精准正文提取 |
| **动态渲染** | Playwright / Puppeteer | 支持 SPA 和 JavaScript 渲染页面 |
| **长文档处理** | LangChain Document Loaders | 支持 PDF/HTML/JSON 等格式 |
| **向量存储** | Chroma / LanceDB | 本地部署, 用于去重和相似度匹配 |
| **缓存** | Redis / SQLite | 搜索结果缓存, 避免重复请求 |
| **速率限制** | asyncio semaphore + 令牌桶 | 控制 API 调用频率 |
| **质量验证** | 自洽性检查 + 交叉验证器 | 内置事实核查管道 |

---

## 5. 完整研究循环代码示例

```python
import asyncio
import time
from dataclasses import dataclass, field
from typing import Optional


class ResearchAgent:
    """
    研究 Agent 主循环
    
    完整流程:
    plan -> search -> read -> verify -> synthesize -> report
    
    对比通用 Agent 的特点:
    1. 更长的规划周期 (不是每步都规划)
    2. 更强的引用要求 (不能无中生有)
    3. 更多的并行操作 (多源同时搜索)
    4. 更严格的质量控制 (交叉验证)
    """
    
    def __init__(self, llm_client, search_api):
        self.planner = SearchPlanner(llm_client)
        self.crawler = WebCrawler()
        self.reader = DocumentReader(llm_client)
        self.verifier = CrossVerifier(llm_client)
        self.reporter = ReportGenerator(llm_client)
        self.citation_mgr = CitationManager()
        self.depth_controller = AdaptiveDepthController(llm_client)
        
        self.llm = llm_client
        self.search_api = search_api
        
        # 统计
        self.stats = {
            'searches': 0,
            'pages_fetched': 0,
            'facts_extracted': 0,
            'llm_calls': 0,
            'start_time': 0,
            'total_cost': 0.0,
        }
    
    async def research(self, question: str, 
                        depth: int = 3, breadth: int = 3) -> ResearchReport:
        """执行完整研究"""
        
        self.stats['start_time'] = time.time()
        print(f"[ResearchAgent] 开始研究: {question}")
        
        # ============ Phase 1: 规划 ============
        print("[Phase 1] 制定研究计划...")
        plan = await self.planner.create_plan(question, depth, breadth)
        self.stats['llm_calls'] += 2
        
        # ============ Phase 2: 搜索 ============
        print(f"[Phase 2] 执行 {len(plan.queries)} 个搜索查询...")
        all_documents = []
        
        for query in plan.queries:
            docs = await self.crawler.crawl(query, self.search_api)
            all_documents.extend(docs)
            self.stats['searches'] += 1
            self.stats['pages_fetched'] += len(docs)
            
            # 注册引用
            for doc in docs:
                self.citation_mgr.register_source(doc.url, doc.title)
        
        # ============ Phase 3: 阅读 ============
        print(f"[Phase 3] 阅读 {len(all_documents)} 个文档...")
        all_chunks = []
        for doc in all_documents:
            chunks = await self.reader.read_document(doc)
            all_chunks.extend(chunks)
        
        self.stats['llm_calls'] += len(all_documents)
        
        # ============ Phase 4: 交叉验证 ============
        print(f"[Phase 4] 交叉验证...")
        facts = await self.verifier.verify(all_documents)
        self.stats['facts_extracted'] = len(facts)
        self.stats['llm_calls'] += len(all_documents)
        
        # ============ Phase 5: 深度控制 ============
        # 检查是否需要进一步研究
        for sub_q in plan.sub_questions:
            need_deeper, reason = await self.depth_controller.should_deepen(
                sub_q, all_documents, depth
            )
            if need_deeper and depth < 5:
                print(f"  [{sub_q}] 需要加深: {reason}")
                # 执行额外搜索...
        
        # ============ Phase 6: 报告生成 ============
        print(f"[Phase 5] 生成报告...")
        report = await self.reporter.generate(plan, facts, all_documents)
        self.stats['llm_calls'] += len(plan.required_sections) + 1
        
        # ============ Phase 7: 质量检查 ============
        print("[Quality] 执行质量检查...")
        citation_issues = self.citation_mgr.verify_integrity(
            report.to_markdown()
        )
        contradictions = self.verifier.get_contradictions()
        
        if citation_issues:
            print(f"  ⚠ 引用问题: {len(citation_issues)} 个")
        if contradictions:
            print(f"  ⚠ 矛盾事实: {len(contradictions)} 对")
        
        # 记录统计
        elapsed = time.time() - self.stats['start_time']
        self.stats['total_cost'] = self._estimate_cost()
        
        print(f"\n[ResearchAgent] 研究完成!")
        print(f"  - 耗时: {elapsed:.1f}s")
        print(f"  - 搜索次数: {self.stats['searches']}")
        print(f"  - 页面读取: {self.stats['pages_fetched']}")
        print(f"  - 提纯事实: {self.stats['facts_extracted']}")
        print(f"  - LLM 调用: {self.stats['llm_calls']}")
        print(f"  - 估计成本: ${self.stats['total_cost']:.4f}")
        
        return report
    
    def _estimate_cost(self) -> float:
        """估算 API 成本"""
        # 简化: 每次 LLM 调用约 $0.01
        # 搜索 API: 每次 $0.001
        search_cost = self.stats['searches'] * 0.001
        llm_cost = self.stats['llm_calls'] * 0.01
        return search_cost + llm_cost
```

---

## 6. 关键挑战

### 6.1 信息准确性

研究 Agent 面临的最大挑战是确保信息的准确性。

| 挑战 | 场景 | 缓解方案 |
|------|------|---------|
| LLM 幻觉 | Agent 编造不存在的研究结果 | 强制引用 + 引用完整性检查 |
| 源偏差 | 只搜索到某一种观点 | 多角度搜索 + 矛盾检测 |
| 过时信息 | 搜索到已过时的数据 | 按时间排序 + "截至日期"标注 |
| 断章取义 | 摘取脱离上下文的引文 | 保留原始上下文片段 |
| 翻译误差 | 跨语言信息理解偏差 | 首选用原文引用 + 翻译标注 |

### 6.2 源质量评估

```
  源质量评分体系:
  
  权威性 (40%):
  - 官方文档/学术论文: 10分
  - 知名技术博客: 7分
  - 个人博客: 4分
  - 论坛/问答: 2分
  
  时效性 (20%):
  - 3个月内: 10分
  - 1年内: 8分
  - 2年内: 5分
  - 3年以上: 2分
  
  相关性 (25%):
  - 直接回答子问题: 10分
  - 部分相关: 5分
  - 间接相关: 2分
  
  可验证性 (15%):
  - 含有可验证的数据/引用: 10分
  - 含有具体细节: 7分
  - 仅有观点无数据: 3分
```

### 6.3 处理时间

研究 Agent 需要平衡质量与速度：

```
  速度 vs 质量权衡:
  
  快速模式 (depth=1, breadth=2):
  时间: ~30秒
  质量: 可接受的概览
  成本: ~$0.05
  
  标准模式 (depth=3, breadth=3):
  时间: ~3分钟
  质量: 可靠的研究报告
  成本: ~$0.30
  
  深度模式 (depth=5, breadth=5):
  时间: ~15分钟
  质量: 详尽的文献综述级别
  成本: ~$1.50
```

**并行化策略:**

- 子问题之间的搜索和阅读可以完全并行
- 同一子问题内部的不同查询可以并行
- 报告各章节可以并行生成
- 交叉验证可以分批进行

### 6.4 成本管理

```
  成本构成分析:
  
  ┌─────────────────────────────────────┐
  │  成本项             占比            │
  ├─────────────────────────────────────┤
  │  LLM 调用            65%            │
  │    ├─ 阅读摘要       35%            │
  │    ├─ 报告生成       20%            │
  │    ├─ 验证分析       10%            │
  │  Web Search API      20%            │
  │  网页抓取 (带宽)      10%            │
  │  向量存储             5%            │
  └─────────────────────────────────────┘
  
  成本优化策略:
  1. 缓存: 相同的搜索查询不重复执行
  2. 选择性阅读: 先读摘要，再决定是否读全文
  3. 批处理: 合并多个小 LLM 请求为一个大请求
  4. 模型分级: 简单任务用小模型，复杂任务用大模型
  5. 渐进式深度: 先浅层调研，用户确认后再深入
```

---

## 7. 与通用 Agent 的区别

### 7.1 核心差异对比

| 维度 | 通用 Agent | 研究 Agent |
|------|-----------|-----------|
| **目标** | 完成任务 (Task Completion) | 产出知识 (Knowledge Production) |
| **输出** | 操作结果 (代码/文件/回复) | 结构化报告 (Report) |
| **规划周期** | 短 (每步 ~1 次 LLM 调用) | 长 (全流程 ~50+ 次 LLM 调用) |
| **外部依赖** | 中 (文件系统/API/数据库) | 高 (搜索/爬虫/阅读/验证) |
| **内部知识依赖** | 高 (常靠模型知识) | 低 (必须靠外部信息) |
| **容错性** | 高 (可重试/用户修正) | 低 (信息错误代价大) |
| **引用要求** | 松散 (可有可无) | 严格 (必须准确标注来源) |
| **并行度** | 低 (通常是顺序执行) | 高 (大量并行搜索和阅读) |
| **记忆结构** | 对话历史 + 短期记忆 | 事实库 + 引用库 + 文档缓存 |
| **质量评估** | 功能正确性 | 信息准确性 + 完整性 + 可溯源 |

### 7.2 专用记忆结构

```
  通用 Agent 记忆                研究 Agent 记忆
+-------------------+        +----------------------+
| 对话历史 (List)   |        | 事实库 (Dict)        |
|                   |        | fact_id -> {         |
| [msg1, msg2, ...] |        |   statement,         |
|                   |        |   sources[],         |
|                   |        |   confidence,        |
|                   |        |   status             |
|                   |        | }                    |
+-------------------+        +----------------------+
                              | 引用库 (Dict)        |
                              | url -> {            |
                              |   id, title,        |
                              |   access_date,      |
                              |   used_in_report    |
                              | }                   |
                              +----------------------+
                              | 文档缓存 (Dict)      |
                              | doc_id -> {         |
                              |   url, content,     |
                              |   summary,          |
                              |   facts[]           |
                              | }                   |
                              +----------------------+
```

### 7.3 更长的规划周期

```
  通用 Agent 循环:
  观察 -> 思考 -> 行动 -> 观察 -> 思考 -> 行动 ...
  每次循环 1-3 步, 计划深度很浅
  
  研究 Agent 循环:
  [规划阶段] -> [执行阶段] -> [综合阶段] -> [质量阶段]
     |              |              |              |
     v              v              v              v
  制定计划       批量执行       汇总结果       检查质量
  
  每个阶段内部可能有多步, 但规划是一个独立的阶段,
  而不是边做边规划。这是因为:
  1. 搜索需要批量化 (单个搜索意义不大)
  2. 并行执行需要预先知道依赖关系
  3. 质量检查需要全局视角而非局部
```

### 7.4 更强的质量要求

```
  质量保证体系:
  
  1. 事实层
     - 每个事实必须有来源
     - 每个来源必须有 URL
     - 单一来源的事实标记为"待验证"
  
  2. 源层
     - 源权威性评分 (>0.6 才算可靠)
     - 多源一致性检查 (至少 2 个独立源)
     - 时效性检查 (3 年以上标记为"历史数据")
  
  3. 报告层
     - 引用完整性 (每个 [N] 有对应文献)
     - 引用未使用检查 (每个文献在报告中至少被引用一次)
     - 矛盾检测 (报告中不能包含自相矛盾的陈述)
     - 完整性检查 (是否回答了所有子问题)
  
  4. 输出层
     - 格式规范 (Markdown 格式正确)
     - 长度要求 (符合用户指定的字数范围)
     - 语言一致性 (全文中英文术语统一)
```

---

## 参考架构

- OpenAI Deep Research: https://openai.com/index/introducing-deep-research/
- Google Mariner: https://blog.google/products/gemini/mariner-agent/
- Perplexity Deep Research: https://www.perplexity.ai/hub/blog/introducing-perplexity-deep-research
- STORM (Stanford): https://github.com/stanford-oval/storm
- AI Scientist (Sakana AI): https://github.com/SakanaAI/AI-Scientist
- PaperQA: https://github.com/white-lab/PaperQA
- AutoResearch: https://github.com/assafelovic/gpt-researcher
