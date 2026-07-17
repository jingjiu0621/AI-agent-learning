# 13.1.4 edge-deploy — Edge 部署

**Edge 部署将 Agent 的计算推到离用户最近的位置，将推理延迟从数百毫秒压缩到数十毫秒。** 对于延迟敏感的 Agent 场景（语音助手、实时决策、IoT 控制），从中心云的 100-300ms 降低到边缘的 10-50ms 可能意味着可用与不可用的区别。但边缘节点的资源限制（CPU/内存/存储）与 LLM 推理的算力需求之间存在根本张力——不是所有 Agent 类型都适合跑在边缘。

## 背景与问题

### Edge 部署的驱动力

中心云部署的 Agent 面临一个物理定律无法突破的延迟天花板：**光速和网络跳数**。一个从中国访问美国西海岸数据中心的请求，即使服务器端处理时间只有 50ms，网络往返也需要 150-300ms。对于需要多轮交互的 Agent（每轮都经历这一延迟），用户体验会显著劣化。

```
用户请求 ──→ DNS 解析 (10-50ms)
        ──→ SSL 握手 (20-80ms)
        ──→ 网络路由 (50-200ms)
        ──→ 网关/负载均衡 (5-20ms)
        ──→ Agent 推理 (500-5000ms)  ← 这才是真正的工作
        ──→ 响应回传 (50-200ms)
                                  总计: 600ms - 5.5s+
```

### 之前是怎么做的？

1. **单一中心区域部署**：所有 Agent 实例部署在一个或两个云区域，全球用户共享
2. **CDN 缓存静态内容**：只缓存前端资源，Agent API 请求仍然全部回到源站
3. **全局负载均衡（GLB）**：将用户路由到最近的数据中心，但云厂商的区域数量有限（AWS 约 30+ 个）

### 这样做的问题

- **多轮交互延迟叠加**：传统 Web 是"请求-响应"一次完成，Agent 可能需要 3-10 轮推理交互，延迟呈乘数效应
- **流式响应受限**：SSE/WebSocket 长连接跨洲传输，中间路由节点增加抖动和断连概率
- **语义缓存无法覆盖长尾**：边缘没有推理能力时，即使是语义相似的请求也必须回源
- **数据驻留合规**：某些行业要求数据不出特定地理边界，中心化部署难以满足

## Edge 部署的核心理念

### 什么是 Agent 的 Edge 部署

与 CDN 只缓存静态文件不同，Agent 的 Edge 部署是一个连续光谱：

```
完全中心化               完全边缘化
──────────────────────────────────────────────────
API 网关 + CDN    →     Edge 函数执行    →     On-device 推理
                              │                         │
                    轻量逻辑在边缘处理         全部推理在本地完成
                    推理仍回源站              不依赖网络
```

三个层次的边缘部署：

| 层次 | 示例 | 延迟 | 算力 | 适用场景 |
|------|------|------|------|----------|
| L1: CDN 边缘缓存 | Cloudflare Cache, Akamai | 10-30ms | 无 | 响应缓存、静态资源 |
| L2: Edge 函数 | Cloudflare Workers, Deno Deploy | 10-50ms | 有限 (CPU) | 预处理、路由、语义缓存命中 |
| L3: Edge 推理节点 | Fly.io, AWS Local Zones, On-device | 20-100ms | GPU/TPU | 小模型推理、Agent 决策 |

### Agent Edge 部署的核心挑战

```
Edge 环境约束                     Agent 需求
─────────────────                 ────────────────
CPU 有限 (通常 ≤ 2 vCPU)          LLM 推理需要 GPU/高算力
内存有限 (128MB - 1GB)             Agent 上下文窗口可能占用数百 MB
存储有限 (临时文件系统)              记忆/知识库需要持久存储
无持久化本地状态                      Agent 天然有状态（对话/记忆）
冷启动频繁 (<10ms 要求)              Python 框架加载 > 500ms
```

核心矛盾在于：**Agent 需要强大推理能力，而 Edge 天生轻量**。解决方案不是"在 Edge 上跑大模型"，而是"重新设计 Agent 架构以适应 Edge"。

## 主流 Edge 平台对比

| 维度 | Cloudflare Workers | Fly.io | AWS Local Zones | Deno Deploy | On-device (手机/PC) |
|------|--------------------|--------|-----------------|-------------|-------------------|
| **运行时** | V8 Isolate (JS/WASM/Pyodide) | Firecracker VM | 标准 VM | V8 Isolate | 本地 CPU/NPU |
| **冷启动** | ~0-5ms | ~200-500ms | ~5s+ | ~0-5ms | 0ms (常驻) |
| **内存** | 128MB (可增) | 按需分配 (VM) | 按需 | 128MB | 设备 RAM |
| **持久化** | Durable Objects, KV, R2 | 卷挂载 | EBS | KV | 本地存储 |
| **GPU 支持** | ❌ 无 | ✅ 可选 GPU | ✅ GPU 实例 | ❌ 无 | ✅ NPU/GPU |
| **全球节点** | 330+ 城市 | 30+ 区域 | 30+ Local Zones | 35+ 区域 | 设备本身 |
| **Python 支持** | 通过 Pyodide (受限) | 原生 ✅ | 原生 ✅ | ❌ | 原生 ✅ |
| **最大执行时间** | 30s (Unbound 15min) | 无限制 | 无限制 | 15min | 无限制 |
| **最佳场景** | 轻量预处理/缓存 | 完整 Agent 运行 | 低延迟推理 | 简单 FF 函数 | 隐私敏感/离线 |

### 平台选型决策流

```
Agent 需要边缘部署吗？
├── 是 ── 延迟要求?
│   ├── < 50ms P99 ── 完全在设备端可行?
│   │   ├── 是 → On-device (小模型 + 本地 RAG)
│   │   └── 否 → 需要 Edge 函数 (Cloudflare Workers)
│   ├── 50-200ms P99 ── Agent 复杂度?
│   │   ├── 低 (简单路由/分类) → Edge Function
│   │   ├── 中 (RAG 问答) → Fly.io / Local Zones
│   │   └── 高 (复杂推理) → 混合: 边缘预处理 + 云推理
│   └── > 200ms P99 ── 中心云即可
└── 否 ── 标准云部署
```

## 核心架构模式

### 模式一：Edge-Cloud 分层推理

最实用的边缘 Agent 架构：将推理链路拆分为边缘层和中心层。

```
用户请求
    │
    ▼
┌──────────────────────┐
│  Edge Layer          │  ← Cloudflare Workers / Fly.io
│                      │
│  ┌────────────────┐  │
│  │ Request        │  │  目的: 快速过滤和预处理
│  │ Classifier     │──│── 简单的 Agent 路由
│  │ (小模型 ~1B)   │  │  如: 分类查询类型
│  └───────┬────────┘  │
│          │           │
│  ┌───────▼────────┐  │
│  │ Semantic Cache │  │  命中 → 直接返回 (0-30ms)
│  │ (Edge KV)      │  │  未命中 → 回源
│  └───────┬────────┘  │
└──────────┼───────────┘
           │ 未命中
           ▼
┌──────────────────────┐
│  Cloud Layer         │  ← 中心云 GPU 集群
│  ┌────────────────┐  │
│  │ Full LLM Agent │  │  完整推理 + 工具调用 + 记忆
│  │ (70B-400B)     │  │  
│  └────────────────┘  │
└──────────────────────┘
```

```python
# edge_agent.py — 在 Cloudflare Workers 或边缘节点运行的 Agent 前端
import json
from typing import Optional

class EdgeAgentFrontend:
    """边缘 Agent 前端：请求分类 + 语义缓存命中 + 回源决策"""
    
    def __init__(self, kv_store, classifier_endpoint: str, origin_endpoint: str):
        self.cache = kv_store  # Edge KV / Durable Objects
        self.classifier = classifier_endpoint  # 小模型分类 API
        self.origin = origin_endpoint  # 中心 Agent 服务
        
    async def handle_request(self, user_input: str, context: dict) -> dict:
        # Step 1: 快速分类请求类型
        query_type = await self.classify(user_input)
        
        # Step 2: 语义缓存查询（使用缓存嵌入向量）
        cache_key = self._make_cache_key(user_input, context.get("user_id"))
        cached = await self.cache.get(cache_key)
        if cached and self._is_cache_valid(cached, query_type):
            return {"source": "cache", "data": json.loads(cached), "latency_ms": 5}
        
        # Step 3: 低复杂度请求尝试边缘推理
        if query_type == "simple_qa" and self._can_edge_infer(user_input):
            result = await self._edge_infer(user_input)  # 边缘小模型
            await self.cache.put(cache_key, json.dumps(result), ttl=3600)
            return {"source": "edge", "data": result, "latency_ms": 50}
        
        # Step 4: 复杂请求回源到中心云
        result = await self._origin_call(user_input, context)
        
        # Step 5: 写回缓存（异步，不阻塞响应）
        asyncio.create_task(self._async_cache_write(cache_key, result))
        return {"source": "origin", "data": result, "latency_ms": result["latency"]}
    
    async def classify(self, text: str) -> str:
        # 用轻量分类器（如 DistilBERT 或简单关键词匹配）
        # 实际可以调用 ONNX 运行时或 Cloudflare AI Gateway
        ...
    
    def _can_edge_infer(self, text: str) -> bool:
        # 判断是否可以在边缘完成推理
        # 如：不需要工具调用、不需要外部知识检索、不需要长上下文
        return len(text) < 200 and not any(kw in text for kw in ["搜索", "计算", "查询"])
```

### 模式二：On-device First Agent

隐私敏感的 Agent 场景（医疗、金融、个人助手）采用"设备优先，云端增强"策略。

```
┌─────────────────────────────────────────┐
│  用户设备 (手机/PC)                       │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ 本地 Agent 核心                    │  │
│  │  • Small LM (Phi-3 / Gemma 2B)   │  │  80%+ 的简单请求在此完成
│  │  • 本地向量数据库 (LanceDB)        │  │
│  │  • 本地工具执行 (系统 API)          │  │
│  └───────────────┬───────────────────┘  │
│                  │                      │
│  ⚠ 超出能力范围 ─┘                      │
└──────────────────┼──────────────────────┘
                   │ 加密通道
                   ▼
┌─────────────────────────────────────────┐
│  云增强层                                │
│  • 复杂推理 (GPT-4 / Claude)             │
│  • 全局知识库                            │
│  • 跨设备同步                            │
└─────────────────────────────────────────┘
```

```python
# on_device_agent.py — 设备端优先的 Agent 架构
class OnDeviceAgent:
    """设备优先 Agent：尽可能在本地完成，云端作为增强"""
    
    def __init__(self, local_model_path: str, cloud_endpoint: str):
        self.local_model = self._load_local_model(local_model_path)  # 2B-7B 参数
        self.local_knowledge = LocalVectorDB()  # 本地知识库
        self.cloud = CloudClient(cloud_endpoint)
        self.offline_queue = []  # 离线请求队列
        
    async def process(self, user_input: str, context: dict) -> dict:
        # 1. 本地优先级评估
        urgency = self._assess_urgency(user_input)
        complexity = self._assess_complexity(user_input)
        
        # 2. 本地能力范围决策
        if self._can_handle_locally(complexity):
            return await self._local_inference(user_input)
        
        # 3. 需要云端，但可以先做本地预处理
        local_ctx = await self._local_preprocess(user_input)
        
        # 4. 云端推理（网络可达时）
        if self._is_online():
            result = await self.cloud.infer(user_input, local_ctx)
        else:
            # 离线模式：排队或本地降级
            self.offline_queue.append((user_input, context))
            return self._fallback_response(user_input)
        
        # 5. 异步学习：云端结果更新本地模型
        asyncio.create_task(self._local_learn(user_input, result))
        return result
    
    def _can_handle_locally(self, complexity: float) -> bool:
        # 复杂度低、不依赖外部工具、不需要外部知识的请求在本地处理
        return complexity < 0.4
        
    def _fallback_response(self, user_input: str) -> dict:
        # 离线降级：关键词匹配 + 基于规则
        ...
```

### 模式三：Edge-Native Agent (Cloudflare Workers + Durable Objects)

将完整的 Agent 会话状态存储在边缘的 Durable Objects 中，实现全局低延迟的状态访问。

```javascript
// edge_agent.js — Cloudflare Workers Agent (JavaScript)
export class AgentSession extends DurableObject {
    constructor(state, env) {
        this.state = state;       // 持久化状态
        this.env = env;
        this.messages = [];
        this.tools = [];
    }
    
    async handleRequest(request) {
        // Durable Object 保证同一 session 在同一节点处理
        const { action, data } = await request.json();
        
        if (action === 'infer') {
            // 调用 AI Gateway 或回源 LLM API
            const result = await this.env.AI.run('@cf/meta/llama-3-8b', {
                messages: [...this.messages, { role: 'user', content: data.input }]
            });
            
            this.messages.push(
                { role: 'user', content: data.input },
                { role: 'assistant', content: result.response }
            );
            
            // 上下文窗口管理（边缘内存有限）
            if (this._tokenCount(this.messages) > 4000) {
                this.messages = this._summarizeAndTruncate(this.messages);
            }
            
            return new Response(JSON.stringify(result));
        }
        
        if (action === 'tool_call') {
            // 边缘执行简单工具（不需要 GPU）
            return await this._executeEdgeTool(data.tool, data.params);
        }
    }
    
    async _executeEdgeTool(tool, params) {
        switch(tool) {
            case 'calculator': return { result: eval(params.expression) };
            case 'lookup': return await this.env.KV.get(params.key);
            case 'time': return { result: new Date().toISOString() };
            default: throw new Error(`Tool ${tool} not supported on edge`);
        }
    }
}
```

## Edge Agent 的缓存策略

边缘部署最大的延迟优势来自缓存。Agent 场景有三种可缓存的层次：

```
缓存层级        缓存内容          命中延迟    缓存位置         典型 TTL
─────────────────────────────────────────────────────────────────────
L1: Response    LLM 响应全文      1-5ms       Edge KV        根据语义相似度
L2: Token       预计算的 KV Cache  5-20ms    Edge 内存       会话期间
L3: Knowledge   向量嵌入 + 文档    10-30ms    Edge KV/R2     知识库更新周期
```

### 语义缓存在 Edge 的实现差异

边缘的语义缓存与中心云最大的不同是：**边缘节点数量多，但每个节点的缓存空间小**。需要权衡缓存一致性与命中率：

```python
class EdgeSemanticCache:
    """分布在不同边缘节点的语义缓存"""
    
    # 权衡: 每个节点独立缓存（高命中但浪费）vs 分布式共享（一致但延迟）
    
    MODE_GEO_SPLIT = "geo_split"  # 按地理区域分片（推荐）
    MODE_LOCAL_ONLY = "local_only"  # 每个节点独立（简单但冗余）
    MODE_CENTRALIZED = "centralized"  # 统一缓存（低冗余但延迟高）
    
    def __init__(self, mode: str = "geo_split"):
        self.mode = mode
        self.local_cache = LRUCache(capacity=5000)  # 每个节点 5000 条
        self.embedding_model = self._load_lightweight_encoder()
```

## 关键挑战与应对

### 挑战 1：冷启动与 Agent 框架大小

Agent 框架（LangChain、LlamaIndex 等）的依赖体积庞大，在 Edge 环境中冷启动显著。

| 框架 | 压缩后大小 | 冷启动 (Workers) | 冷启动 (Fly.io) |
|------|-----------|-----------------|-----------------|
| LangChain Core | ~500KB | +200ms | +300ms |
| LangChain + 社区包 | ~5MB | 超出 Workers 限制 | +500ms |
| 手写 Agent (minimal) | ~50KB | +5ms | +100ms |
| ONNX 小模型 | ~10-100MB | N/A | +2-5s |

**对策**：
- Edge 场景使用**手写最小 Agent** 而非重型框架
- Fly.io/Firecracker 类 VM 隔离更适合完整框架
- 使用 WASM 编译的轻量推理引擎（如 llama.cpp WASM）

### 挑战 2：状态持久化

```
边缘环境的存储约束：
────────────────
• Cloudflare Workers: Durable Objects (128MB/DO), KV (25MB 免费)
• Fly.io: VM 卷挂载（持久化，但故障时重建）
• On-device: 本地文件系统（设备更换时丢失）

解决方案：Event Sourcing 模式
──────────────────────────────
Agent 在边缘的每次操作记录为事件
事件异步同步到中心持久化存储
边缘崩溃后从事件日志重建状态
```

### 挑战 3：模型大小 vs 延迟取舍

```
                    Edge 延迟目标
                    < 50ms    < 200ms   < 500ms   > 500ms
                    ───────  ────────  ────────  ────────
< 1B 参数 (Phi-3)     ✅        ✅        ✅        ✅
1B-7B 参数 (Gemma)    ❌        ⚠         ✅        ✅
7B-20B 参数            ❌        ❌        ⚠         ✅
> 70B 参数 (云)        ❌        ❌        ❌        ⚠
```

## 适用场景判断

### 适合 Edge 部署的 Agent

- **语音交互 Agent**：延迟敏感，50ms vs 500ms 体验差异巨大
- **IoT/嵌入式 Agent**：设备端运行，网络不稳定
- **离线 Agent**：飞机、偏远地区等无网络环境
- **隐私敏感 Agent**：医疗诊断、金融建议，数据不出设备
- **实时监控 Agent**：边缘告警，毫秒级响应需求
- **多语言翻译 Agent**：小模型在边缘即可完成

### 不适合 Edge 部署的 Agent

- **深度研究 Agent**：需要长上下文、多工具调用、大数据分析
- **复杂多 Agent 协作**：需要全局协调器和共享状态
- **大规模 RAG 系统**：知识库太大（>10GB），无法驻留边缘
- **高精度决策 Agent**：需要 70B+ 模型达到的推理质量

## 工程优化方向

1. **模型量化 + 蒸馏**：将 7B 模型量化为 INT4 在边缘运行，损失 1-3% 精度但延迟降低 4x
2. **推测解码 (Speculative Decoding)**：边缘小模型生成草稿 → 云端大模型验证
3. **渐进式上下文加载**：先传输入的前 100 tokens 给边缘分类，再决定传输方向
4. **局部 Agent 设计**：将全局 Agent 拆成"区域 Agent + 全局协调器"的地域化架构
5. **WebSocket 长连接保活**：减少边缘节点的连接重建开销
6. **自适应路由**：根据当前网络状况动态决定边缘缓存还是回源

## 与相关部署方案的对比

| 对比项 | Docker 部署 | K8s 部署 | Serverless | Edge 部署 |
|--------|-----------|---------|-----------|----------|
| 延迟 | 50-200ms | 50-200ms | 100-500ms | **5-50ms** |
| 全球覆盖 | 有限区域 | 有限区域 | 云区域 | **330+ 节点** |
| 算力 | 高 | 高 | 中 | **低** |
| 状态持久化 | ✅ 本地卷 | ✅ PVC | ⚠ 外部依赖 | ⚠ 受限 |
| 冷启动 | 无 (常驻) | 无 (常驻) | 100-500ms | **0-5ms (Workers)** |
| Python 生态 | ✅ 完整 | ✅ 完整 | ✅ 完整 | ⚠ 受限 |
| 隐私合规 | 手动 | 手动 | 自动区域 | **天然分布** |

Edge 部署不是 Docker/K8s/Serverless 的替代，而是**在最靠近用户的位置增加一层预处理和缓存**。在实际架构中，Edge Agent 通常作为前端守卫存在，处理简单请求、命中缓存，复杂请求仍然回源到中心云。
