# 10.1.6 Log Aggregation — 日志聚合

## 简单介绍

日志聚合（Log Aggregation）是将分散在多个 Agent 实例、多个服务、多个节点的日志收集到统一平台的过程。在分布式 Agent 系统中，单台机器上的日志文件是"盲人摸象"——只看一个 Agent 的日志无法理解全局发生了什么。日志聚合使得 **"搜索所有 Agent 的所有日志"** 成为可能。

## 基本原理

日志聚合管道的标准架构：

```
Agent 实例 ──→ 日志文件 / stdout
    │
    ├── Filebeat / Fluentd / Vector ──→ 日志收集器
    │                                       │
    │                                       ▼
    │                                  Kafka / Redis
    │                                  （缓冲层）
    │                                       │
    │                                       ▼
    │                               Log Processor
    │                          解析/结构化/过滤/富化
    │                                       │
    │                                       ▼
    │                         ┌──────────────┴──────────────┐
    │                         ▼                             ▼
    │                   Elasticsearch / Loki             S3 / GCS
    │                  （索引 + 存储）                  （冷存储归档）
    │                         │
    │                         ▼
    │                   Kibana / Grafana
    │                   （搜索 + 可视化）
```

```python
# 一个典型的日志聚合架构配置（伪代码）
class LogAggregator:
    def __init__(self):
        self.buffer = AsyncQueue(max_size=10000)
        self.collector = FluentdCollector(host="fluentd.example.com")
        self.processor = LogProcessor(schema=AGENT_LOG_SCHEMA)

    async def emit(self, log_entry: dict):
        # 1. 本地缓冲（防崩溃丢失）
        await self.buffer.put(log_entry)

        # 2. 批量发送（减少网络开销）
        if self.buffer.qsize() >= BATCH_SIZE:
            batch = await self.buffer.drain()
            enriched = [self.processor.enrich(e) for e in batch]
            await self.collector.send_batch(enriched)

    def enrich(self, entry: dict) -> dict:
        # 日志富化：添加全局上下文
        entry["@environment"] = self.environment
        entry["@cluster"] = self.cluster_name
        entry["@agent_version"] = AGENT_VERSION
        return entry
```

## 背景

传统软件的日志聚合（ELK Stack）已经有成熟的实践。但 Agent 系统的日志聚合面临新的挑战：

1. **Agent 实例是动态的**——K8s 上的 Agent Pod 可能随时扩缩容，日志源地址不断变化
2. **Agent 日志与业务日志混合**——Agent 日志输出到 stdout，与后端服务的业务日志混在一起
3. **Agent 日志量巨大**——一个 Agent 的单次任务可能产生 100+ 步的日志，100 个并发 Agent 每秒产生数千条日志
4. **跨 Agent 关联**——在多 Agent 系统中，一个任务的日志分布在多个 Agent 实例上，需要按 trace_id 关联

## 之前的做法与结果

| 做法 | 描述 | 结果 |
|------|------|------|
| 本地文件存储 | 每个 Agent 实例写本地日志文件 | 无法搜索、无法聚合，实例销毁日志即丢失 |
| NFS 共享存储 | 所有 Agent 写同一个 NFS 共享 | 写冲突、IO 瓶颈、不可扩展 |
| K8s `kubectl logs` | 直接通过 K8s API 查看日志 | 只能看单 Pod、默认只保留最近几小时、无法检索历史 |
| 简单的 HTTP 上报 | Agent 每次打日志就 HTTP POST 到中心服务器 | 阻塞 Agent 主流程，高并发下日志服务成为瓶颈 |

## 核心矛盾

| 矛盾维度 | 不聚合 | 强实时聚合 | 准实时聚合 |
|---------|-------|-----------|-----------|
| 延迟 | - | 秒级 | 分钟级 |
| 系统复杂度 | 低 | 高（需要 Kafka 等中间件） | 中 |
| 运维成本 | 低 | 高 | 中 |
| 数据完整性 | 高（本地存储） | 中（网络问题可能丢数据） | 中 |
| 搜索能力 | 无 | 好 | 好 |

**核心矛盾**：Agent 系统需要准实时的日志聚合来支持调试和监控，但聚合管道的每一层都在增加延迟和复杂度。解决方案是**分层管道**：本地快速写入 + 异步批量传输 + 最终的索引和存储，各层之间的延迟可接受分钟级。

## 当前主流优化方向

1. **OTel 日志协议**——OpenTelemetry 的日志协议正在成为日志聚合的标准，与已有的 OTel 追踪基础设施统一
2. **基于向量（Vector）的下一代管道**——Vector（Rust 实现）比传统的 Filebeat/Fluentd 性能高 10x，且支持丰富的处理 transform
3. **日志即事件流**——将日志视为事件流而非文件集合，使用 Kafka/Pulsar 等流处理平台进行实时处理，而非传统的"提文件-解析-索引"批处理模式
4. **自动 Schema 推理**——聚合系统自动从日志 JSON 中推断字段类型和结构，无需手动配置 mapping
5. **成本感知的保留策略**——智能管理日志的存储分层：热存储（SSD，7天）、温存储（HDD，30天）、冷存储（S3/Glacier，1年）、归档（Glacier Deep Archive，长期）

## 实现的最大挑战

1. **日志格式的统一**——不同 Agent 框架（LangChain、CrewAI、AutoGen）、不同模型提供商（OpenAI、Anthropic、本地模型）的日志格式各不相同，聚合管道需要适配多种格式
2. **Agent 日志的关联性**——在分布式环境中，分散在多个 Agent 实例上的同一次任务日志需要按 trace_id 准确关联
3. **日志量的预估和容量规划**——Agent 的日志量与任务复杂度强相关（一个复杂任务可能产生 1000+ 步日志），难以准确预测
4. **敏感信息泄露风险**——聚合后的日志可被多人搜索查看，如果脱敏不彻底会形成数据泄露渠道

## 能力边界

**能做什么：**
- 跨所有 Agent 实例搜索和过滤日志
- 按 trace_id/session_id/agent_id 关联相关日志
- 长期存储历史日志以供事后分析
- 自动计算日志量、错误率、延迟趋势等聚合指标

**不能做什么：**
- 不能实时保证日志的完整性——网络故障或 Log Processor 崩溃可能导致数据丢失
- 不能自动识别日志中的异常模式——聚合只负责收集和存储，异常检测需要额外的分析层
- 不能降低 Agent 的日志产生量——聚合是"管道"，两端分别是 Agent 和搜索界面

## 与其他的区别

| 对比项 | 传统 ELK 日志聚合 | Agent 日志聚合 |
|--------|------------------|--------------|
| 日志量 | 请求数 × 服务数 | 请求数 × 步数 × Agent 数 |
| 索引策略 | 按时间索引 | 按时间 + trace_id 索引 |
| 关键字段 | endpoint, status_code, duration | trace_id, turn_number, tool_name |
| 关联查询 | 按 request_id 关联 | 按 trace_id + session_id 关联 |
| 存储策略 | 短期热 + 长期冷 | 调试期全量 + 稳定期采样 |

## 最终工程优化

1. **双通道聚合**——Agent 日志走两条路径：快速通道（实时写入本地 Redis，供最近 1 小时搜索）和完整通道（批量写入 ES/Loki，供长期查询）
2. **日志缓冲与背压**——当日志聚合管道拥堵时，Agent 端自动降采样（从 100% → 10%），防止日志积压导致 Agent OOM
3. **自动化日志 schema 注册**——Agent 启动时向 Schema Registry 注册其日志格式，聚合系统自动识别和适配，避免手动更新配置
4. **跨集群日志联合搜索**——多地域部署的 Agent 系统需要日志联合搜索能力，通过查询路由实现跨集群的 trace_id 关联
5. **与 APM 集成**——Agent 日志聚合与 APM（Application Performance Monitoring）系统打通，实现"从用户请求到 Agent 推理到工具调用的全链路观察"
