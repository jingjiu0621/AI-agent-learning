# 12.4.6 isolation-boundary — 隔离边界

**隔离边界是 Agent 安全失效时的最后一道物理防线。** 当 Prompt 注入绕过了指令层级、权限控制未能限制工具调用、护栏放行了恶意输出——隔离边界仍然是阻挡实际损害的最后屏障。它通过限制 Agent 执行环境能"触及"的范围，将攻击的物理影响限制在一个可控的沙箱内，使其无法触及宿主机、网络中的其他服务、持久化存储或敏感数据。

---

## 简单介绍

隔离边界（Isolation Boundary）指通过操作系统级、虚拟化级或运行时级的隔离技术，限制 AI Agent 执行环境的资源访问范围，使其在遭到攻击或出现故障时无法影响外部系统。对于具备工具调用能力的 Agent，隔离边界定义了 Agent "能做什么"的**物理极限**——权限控制说"你不能调用这个工具"，隔离边界说"即使你调用了工具，你仍无法触及这些资源"。

Agent 的隔离边界涉及多个层次：

```
Agent 执行环境隔离层次
───────────────────────────────────────────────────────────
Agent 进程
  │
  ├── 进程级隔离: 独立进程、seccomp、namespaces、cgroups
  ├── 容器级隔离: Docker / Podman 容器、资源限制
  ├── 微 VM 隔离: Firecracker / gVisor / Kata Containers
  ├── 沙箱库隔离: nsjail / Bubblewrap / Landlock
  ├── 代码执行隔离: WASM 沙箱、受限 Python 运行时
  ├── 网络隔离: egress 过滤、DNS 策略、网络命名空间
  └── 文件系统隔离: overlayfs / tmpfs / read-only root
```

**关键区别**：权限控制是"逻辑的"（策略决定能否调用某工具），隔离边界是"物理的"（即使 Agent 设法绕过权限检查，隔离环境也阻止其访问资源）。

---

## 基本原理 — isolation contains damage when other defenses fail

隔离边界的核心原则是**纵深防御的最后一道物理防线**。其基本假设是：**所有上层安全机制都可能被攻破**，因此必须在执行环境层面提供与上层机制无关的、不可绕过的资源访问限制。

```
安全防御纵深
───────────────────────────────────────────────────────────────
                                       攻击者渗透深度
Layer 1:  Prompt 注入防护               ──────────►
Layer 2:  输入验证 / 内容过滤                ──────►
Layer 3:  输出护栏 / 语义审核                    ───►
Layer 4:  权限控制 / 最小权限                     ──►
Layer 5:  审批流程 / HITL                          ─►
Layer 6:  隔离边界 ★                                  ► 止步于此
                                                      │
                                                      ▼
                                                  系统受损
                                           (隔离边界阻止)
```

隔离边界基于三个基本原理：

**1. 最小环境原则（Principle of Minimal Environment）**

Agent 的执行环境只包含完成任务所需的最少资源——没有网络访问权限就不提供网络栈，不需要写文件就不提供可写文件系统，不需要 GPU 就不暴露 GPU 设备。

```
Agent 任务: "帮我翻译这份文档"
所需最小环境:
  ✅ 文件系统: read-only 挂载文档目录
  ✅ 网络:    不需要（使用 MCP 协议而非直接网络调用）
  ❌ 网络:    不需要外部 API 调用（非必要不授予）
  ❌ Shell:   不需要执行任意命令
  ❌ 写权限:  不需要修改文件系统
```

**2. 失败隔离（Failure Isolation）**

一个 Agent 的崩溃、卡死或被攻破不影响其他 Agent 或宿主系统。每个执行环境是独立的故障域——如果一个 Agent 沙箱被完全攻破，攻击者也只能看到这个沙箱内的资源。

```
┌──────────────────────────────┐
│ 宿主机                        │
│                              │
│  ┌─────────┐  ┌─────────┐   │
│  │ Agent A │  │ Agent B │   │  ← A 被攻破不影响 B
│  │ 沙箱    │  │ 沙箱    │   │
│  └────┬────┘  └─────────┘   │
│       │                     │
│  ┌────▼─────────────────┐   │
│  │ 攻击者视角（仅在A沙箱内） │   │
│  │ /tmp (空)             │   │
│  │ /app (read-only)      │   │
│  │ 无网络                │   │
│  │ 无敏感数据            │   │
│  └───────────────────────┘   │
└──────────────────────────────┘
```

**3. 无特权逃逸（No Privilege Escalation）**

执行环境内部没有任何机制可以突破隔离边界——沙箱内的进程以非特权用户运行，没有能力加载内核模块、修改系统调用表、访问宿主机命名空间或操作 cgroup 配置。

```
沙箱内部                           沙箱外部
─────────                         ─────────
uid=1000 (nobody)                 root 操作 cgroup
no CAP_SYS_ADMIN                  管理网络命名空间
no CAP_NET_RAW                    挂载文件系统
no /dev devices                   管理 Seccomp 策略
无法加载内核模块                   审计和监控
```

---

## 背景 — chroot jails → containers → micro-VMs → WASM sandboxes → AI sandboxes

隔离技术的发展历史是一部攻防博弈史：每一次更强大的隔离技术的出现，都伴随着对更轻量、更灵活的执行环境的需求。

```
隔离技术演进时间线
───────────────────────────────────────────────────────────────────

1979:  chroot (V7 Unix)
       │ 第一个文件系统隔离机制。将进程的根目录锁定在指定目录。
       │ 问题：chroot 不是安全机制——有 root 权限可以轻易逃逸。
       │
2000:  FreeBSD Jail
       │ 真正的操作系统级 jail。每个 jail 有独立的网络栈、文件系统、进程视图。
       │ 奠定了现代容器隔离的概念基础。
       │
2001:  Linux VServer / Solaris Zones
       │ 类似 FreeBSD Jail 的 Linux 实现。引入安全上下文概念。
       │
2005:  SELinux / AppArmor
       │ 强制访问控制（MAC）。通过 LSM 钩子实现细粒度的资源访问控制。
       │
2006:  cgroups (Google 工程师 Paul Menage 和 Rohit Seth)
       │ 进程组资源限制——CPU、内存、磁盘 I/O、网络带宽。
       │ 与 namespace 一起构成容器的基石。
       │
2008:  LXC (Linux Containers)
       │ 第一个完整的 Linux 容器管理工具。使用 cgroups + namespace。
       │
2013:  Docker
       │ 容器技术爆发点。引入镜像、分层文件系统（overlayfs）、
       │ 容器编排的标准化格式。让容器成为开发者友好的工具。
       │
2015:  OCI Standard + Kubernetes
       │ Open Container Initiative 标准化容器运行时接口。
       │ Kubernetes 成为容器编排的事实标准。
       │
2018:  gVisor (Google)
       │ 在用户空间实现 Linux 内核接口。每个沙箱有独立的"内核"。
       │ 提供比共享内核容器更强的隔离性。
       │
2018:  Firecracker (AWS)
       │ 为无服务器计算设计的微虚拟机。每个函数一个 VM。
       │ AWS Lambda + Fargate 的底层技术。启动时间 < 125ms。
       │
2019:  WASM (WebAssembly) 用于服务端
       │ WebAssembly System Interface (WASI) 让 WASM 可以安全地
       │ 访问操作系统资源。提供内存安全的代码执行沙箱。
       │
2020:  Kata Containers
       │ 融合容器体验和 VM 安全。每个容器运行在轻量级 VM 中。
       │ 兼具容器的快启动和 VM 的强隔离。
       │
2022:  nsjail / Bubblewrap 流行
       │ 轻量级进程沙箱工具。用于 CI/CD、代码编译执行、CTF 比赛。
       │ 在开发者工具领域广泛采用。
       │
2023:  AI Agent 沙箱需求爆发
       │ AutoGPT / ChatGPT Plugins / Claude Tool Use 等 Agent 框架
       │ 需要安全执行 LLM 生成的操作。隔离边界成为 Agent 安全核心话题。
       │
2024:  MCP (Model Context Protocol)
       │ Anthropic 推出 MCP 协议，标准化 Agent 与外部系统的接口。
       │ 隔离边界从"进程/容器级"延伸到"协议级"。
       │
2025:  MCP-Secure Agent Kernel (ETH Zurich)
       │ 基于能力的安全（capability-based security）用于 Agent 系统。
       │ 每个工具调用在隔离环境中执行,具有独立的安全策略。
```

**Agent 时代对隔离技术的特殊需求**：

```
传统场景 vs AI Agent 场景的隔离需求差异
────────────────────────────────────────────────────
维度              传统场景                 AI Agent 场景
──               ──────                  ──────────
工作负载类型      静态服务（如 Web Server）  动态生成的操作序列
执行模式          长时间运行                  短暂、高频、突发
安全威胁          外部攻击                    内部攻击（Agent 被注入）
隔离粒度          服务级                      单次工具调用级
策略动态性        部署时固定                  运行时动态生成
与 LLM 的交互     无                          Agent 做决策 → 沙箱执行
审计要求          操作审计                    完整决策链审计
```

---

## 核心矛盾 — strong isolation vs performance, developer experience

隔离边界的核心矛盾是**隔离强度与执行效率、开发者体验之间的三元权衡**。

```
三元权衡
───────────────────────────────────────────────────────────
                  隔离强度
                     ▲
                    / \
                   /   \
                  /     \
                 /       \
                /         \
               /           \
              ▼─────────────▼
           性能              开发者体验

不可能三角：三者无法同时最优
  • 强隔离 + 高性能 = 牺牲开发者体验（Firecracker 微 VM）
  • 强隔离 + 好体验 = 牺牲性能（完整 VM，启动慢）
  • 高性能 + 好体验 = 牺牲隔离强度（共享内核容器）
```

### 具体矛盾表现

| 矛盾 | 描述 | 实际影响 |
|------|------|---------|
| **隔离强度 vs 启动延迟** | 微 VM 提供强隔离但启动需要 100ms+；容器 10ms；进程 < 1ms | Agent 高频调用时累积延迟不可接受 |
| **隔离强度 vs 资源开销** | 每个隔离环境都有内存和 CPU 开销。千级并发 Agent 下开销显著 | 微 VM 每个约 50MB 内存起始；容器约 5MB |
| **隔离强度 vs 开发者体验** | 强隔离环境调试困难。无法 attach 调试器，无法查看日志文件 | 开发期频繁遇到"在沙箱外能跑，沙箱内不行" |
| **隔离强度 vs 网络延迟** | 网络隔离增加代理跳数，egress 过滤引入额外延迟 | 每次工具调用的网络延迟增加 5-50ms |
| **隔离强度 vs 文件 I/O** | overlayfs 层叠引入额外 I/O 开销 | 大量文件操作时性能下降 10-30% |
| **通用隔离 vs 定制需求** | 通用沙箱配置无法满足所有 Agent 的特定需求 | 每个 Agent 类型需要定制隔离策略 |

### 实际场景中的权衡

```
场景一：个人助手 Agent（单用户，低风险）
  隔离策略：进程级隔离 + seccomp
  启动时间：< 5ms
  额外开销：< 2% CPU
  安全等级：低（足够个人使用）
  理由：用户本身就是信任边界，隔离主要防注入后的横向移动

场景二：企业数据分析 Agent（多租户，数据敏感）
  隔离策略：容器级隔离 + 网络隔离 + 文件系统隔离
  启动时间：~50ms
  额外开销：~5% CPU
  安全等级：中（防止数据泄露）
  理由：多租户数据隔离是刚需，但性能不能太差

场景三：代码执行 Agent（高风险，第三方代码）
  隔离策略：微 VM + WASM 沙箱 + 严格网络隔离
  启动时间：~200ms
  额外开销：~15% CPU
  安全等级：高（防止逃逸攻击）
  理由：执行未知代码需要最强隔离，启动时间可接受

场景四：金融交易 Agent（极高风险，监管合规）
  隔离策略：专用 VM + 硬件安全模块 + 完整审计
  启动时间：~5s
  额外开销：~30% CPU
  安全等级：最高（合规要求）
  理由：监管要求物理隔离级别，性能是次要考虑
```

---

## 详细内容

### 1. Container-Level Isolation: Docker/K8s for tool execution, per-session containers, resource limits

容器是 Agent 隔离最广泛使用的技术——它提供了合理的隔离强度、可接受的性能和成熟的生态。容器通过 Linux 内核的 **namespaces**（隔离进程视图）和 **cgroups**（限制资源使用）实现。

**容器隔离的关键机制**：

```
Docker 容器隔离层
───────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────┐
│  PID namespace    │  独立的进程树                    │
│  (pid)            │  容器内看不到宿主机进程           │
├─────────────────────────────────────────────────────┤
│  Network namespace│  独立的网络栈                    │
│  (net)            │  容器有自己的 IP、路由、端口       │
├─────────────────────────────────────────────────────┤
│  Mount namespace  │  独立的文件系统挂载              │
│  (mnt)            │  overlayfs 层叠文件系统           │
├─────────────────────────────────────────────────────┤
│  User namespace   │  独立的 UID/GID 映射             │
│  (user)           │  容器内 root ≠ 宿主机 root        │
├─────────────────────────────────────────────────────┤
│  IPC namespace    │  独立的进程间通信                │
│  (ipc)            │  System V IPC / POSIX 消息队列   │
├─────────────────────────────────────────────────────┤
│  UTS namespace    │  独立的主机名和域名              │
│  (uts)            │                                    │
├─────────────────────────────────────────────────────┤
│  Cgroups          │  CPU / 内存 / I/O 资源限制         │
│                   │  防止 DoS 和资源抢占               │
├─────────────────────────────────────────────────────┤
│  Seccomp          │  系统调用过滤                    │
│                   │  只允许必要的 syscall              │
├─────────────────────────────────────────────────────┤
│  Capabilities     │  移除所有危险 capability           │
│                   │  --cap-drop=ALL                   │
└─────────────────────────────────────────────────────┘
```

**容器隔离在 Agent 中的典型模式**：

```
模式一：Per-Session 容器
─────────────────────────────────────────
  每次用户对话启动一个专用 Agent 容器
  对话结束后销毁容器
  优势：完全隔离，用户间无数据泄漏
  劣势：启动开销累计

  用户 A ──► Session A ──► Container A ──► 工具调用
  用户 B ──► Session B ──► Container B ──► 工具调用
  用户 C ──► Session C ──► Container C ──► 工具调用

模式二：Per-Tool 容器
─────────────────────────────────────────
  每次工具调用启动一个专用容器
  调用完成后立即销毁
  优势：最细粒度隔离，工具间互相不影响
  劣势：高频调用时启动开销巨大

  Agent ──► search_web ──► Container (搜索) ──► 销毁
       ──► execute_code ──► Container (代码执行) ──► 销毁
       ──► read_file ──► Container (只读挂载) ──► 销毁

模式三：容器池（Container Pool）
─────────────────────────────────────────
  预创建一组容器，Agent 执行时从池中获取
  执行完成后清理并归还到池中
  优势：消除启动延迟，复用环境
  劣势：池中环境的残留数据需要彻底清理

  ┌─────────────┐
  │ 容器池       │   Agent A ──► 获取容器 ──► 执行 ──► 归还
  │ Container A  │   Agent B ──► 获取容器 ──► 执行 ──► 归还
  │ Container B  │
  │ Container C  │
  │ Container D  │
  └─────────────┘
```

**Dockerfile 示例——Agent 工具执行容器**：

```dockerfile
FROM alpine:3.19 AS agent-tool-runner

# 最小化基础系统
RUN apk add --no-cache \
    curl=8.5.0-r0 \
    ca-certificates=20230506-r0 \
    python3=3.12.1-r0 \
    # 只安装必要的运行时
    && rm -rf /var/cache/apk/*

# 创建非 root 用户
RUN addgroup -S agent && adduser -S agent -G agent

# 设置工作目录
WORKDIR /workspace

# 复制只读的工具定义
COPY --chown=agent:agent tools/ /tools/

# 安全配置
# 1. 只读根文件系统（容器启动时）
# 2. 无特权模式
# 3. 移除所有 capability
# 4. seccomp 默认配置
# 5. 内存限制（运行时 Docker CPU 参数）
USER agent

# 入口
ENTRYPOINT ["/tools/entrypoint.sh"]
```

**Kubernetes 安全策略——Pod Security Standards**：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: agent-sandbox
  labels:
    app: agent-tool-executor
spec:
  # 安全上下文——容器级
  containers:
  - name: tool-runner
    image: agent-tool-runner:latest
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
      seccompProfile:
        type: RuntimeDefault
    resources:
      limits:
        cpu: "0.5"
        memory: "256Mi"
      requests:
        cpu: "0.1"
        memory: "64Mi"
    # 临时文件系统
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: workspace
      mountPath: /workspace
  volumes:
  - name: tmp
    emptyDir:
      medium: Memory   # tmpfs 内存文件系统
  - name: workspace
    emptyDir:
      medium: Memory
  # Pod 级安全策略
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
```

**资源限制最佳实践**：

| 资源 | 典型限制 | 理由 |
|------|---------|------|
| CPU | 0.5-2 核 | Agent 工具调用通常是 CPU 轻量级的 |
| 内存 | 128-512 MB | 限制内存泄漏和 DoS |
| 磁盘 | 100 MB tmpfs | 临时工作空间，用完即焚 |
| 网络 | 仅允许出站到白名单 | 阻止数据泄露到未授权端点 |
| 文件描述符 | 1024 | 防止句柄耗尽 |
| 进程数 | 64 | 防止 fork bomb |
| 写入速率 | 1 MB/s | 防止磁盘 I/O DoS |

### 2. Micro-VM Isolation: Firecracker/gVisor for stronger isolation, Kata Containers

当共享内核容器的隔离强度不够（容器逃逸漏洞 CVE-2019-5736、CVE-2022-0492 等），且完整 VM 的开销不可接受时，微 VM（Micro-VM）提供了中间选择。微 VM 为每个工作负载运行独立的、最小化的虚拟机，但启动速度和资源开销远低于传统 VM。

**微 VM 技术对比**：

```
技术栈对比
───────────────────────────────────────────────────────────────────────
            启动时间    内存开销    隔离强度    兼容性    典型场景
            ───────    ────────    ────────    ──────    ────────
Docker       10-50ms    ~5 MB       中         高        通用 Agent
gVisor       20-80ms    ~15 MB      中高       中上      多租户 Agent
Firecracker  125ms+     ~50 MB      高         低        代码执行/高风险
Kata          1-3s      ~128 MB     很高       高        合规/金融场景
传统 VM       30-60s    ~1 GB+      最高       高        很少用于 Agent
```

#### Firecracker

AWS 开发的微虚拟机管理程序，是 AWS Lambda 和 AWS Fargate 的底层技术。Firecracker 运行在 KVM（Kernel-based Virtual Machine）之上，每个微 VM 运行独立的 Linux 内核。

```
Firecracker 架构
───────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────┐
│ 宿主机                                              │
│  ┌───────────────────────────────────────────────┐  │
│  │ Firecracker 进程 (VMM)                        │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │ 微 VM #1 (Agent A)                      │  │  │
│  │  │  ┌──────────────┐  ┌────────────────┐   │  │  │
│  │  │  │ Linux Kernel │  │ Agent 进程      │   │  │  │
│  │  │  │ (5.10)       │  │ 工具执行         │   │  │  │
│  │  │  └──────────────┘  └────────────────┘   │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │ 微 VM #2 (Agent B)                      │  │  │
│  │  │  ┌──────────────┐  ┌────────────────┐   │  │  │
│  │  │  │ Linux Kernel │  │ Agent 进程      │   │  │  │
│  │  │  │ (5.10)       │  │ 工具执行         │   │  │  │
│  │  │  └──────────────┘  └────────────────┘   │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  │                                    KVM        │  │
│  └───────────────────────────────────────────────┘  │
│  Hardware (CPU with VT-x / AMD-V)                  │
└─────────────────────────────────────────────────────┘
```

**Firecracker 用于 Agent 的优势**：
- 真正的硬件级隔离（独立内核，无共享内核攻击面）
- 快速启动（~125ms 到 shell，优化后可更低）
- 安全多租户（一个微 VM 的逃逸不影响其他微 VM）
- 精简设备模型（仅虚拟必要的设备，减少攻击面）

**Firecracker 用于 Agent 的挑战**：
- 需要 KVM 支持（无法在容器内再嵌套 Firecracker）
- 启动延迟比容器高一个数量级
- 资源开销更大（每个微 VM 有独立内核）
- 调试困难（没有 SSH，需要串口控制台）

#### gVisor

Google 开发的用户空间内核（Sandbox Kernel）。gVisor 拦截 Agent 进程的系统调用，在用户空间实现 Linux 内核接口，而不是让系统调用直接通过宿主机内核执行。

```
gVisor 架构
───────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────┐
│ 宿主机内核                                           │
│  ┌───────────────────────────────────────────────┐  │
│  │ gVisor Sentry (用户空间内核)                    │  │
│  │                                                │  │
│  │ 实现 Linux syscall 接口:                        │  │
│  │  • read/write/open/close → 文件系统代理         │  │
│  │  • socket/bind/connect → 网络栈（用户空间）      │  │
│  │  • mmap/brk → 内存管理                         │  │
│  │  • clone/fork → 进程管理                       │  │
│  │                                                │  │
│  │ 每个 syscall 不在宿主机内核执行                   │  │
│  │ → 缩小内核攻击面                                │  │
│  └───────────────────────────────────────────────┘  │
│       │                                              │
│       ├── 文件系统: gofer (9p 协议代理)               │
│       ├── 网络:    netstack (用户空间 TCP/IP 栈)       │
│       └── 内存:    host page table 映射              │
└─────────────────────────────────────────────────────┘
```

**gVisor 隔离特性**：

| 特性 | 实现 | 安全效果 |
|------|------|---------|
| 系统调用拦截 | ptrace / KVM 拦截所有 syscall | 减少内核攻击面 |
| 用户空间网络栈 | 纯 Go 实现的 TCP/IP 栈 | 网络 syscall 不经过宿主机内核 |
| 用户空间文件系统 | Gofer 协议代理文件访问 | 文件 syscall 被过滤 |
| seccomp 加固 | 限制 sentry 自身的 syscall | 防止 sentry 本身的逃逸 |
| 兼容性 | 支持 ~70% Linux syscall | 部分应用不兼容 |

**gVisor vs 原生性能**（典型场景）：

```
操作             原生 Linux    gVisor      性能比
────             ──────────   ──────       ────
文件读 (4KB)      ~0.5 us      ~5 us       10%
网络吞吐量         ~40 Gbps     ~10 Gbps    25%
内存分配          ~50 ns       ~200 ns      25%
fork+exec         ~200 us      ~5 ms         4%
系统调用吞吐量     ~100M ops/s  ~5M ops/s    5%
```

#### Kata Containers

Kata 融合了容器的开发者体验和 VM 的安全等级。每个"容器"实际上运行在一个轻量级 VM 中，但通过兼容 OCI 标准，用户使用 Docker/K8s 命令操作。

```
Kata Containers 架构
───────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────┐
│ Kubernetes / Docker                                 │
│       │                                              │
│       ▼                                              │
│  containerd ──► kata-runtime ──► QEMU / Cloud Hypervisor
│                                      │                │
│                                      ▼                │
│                              ┌──────────────┐        │
│                              │ Guest Kernel  │        │
│                              │ (轻量级 Linux) │        │
│                              │              │        │
│                              │ Agent 进程    │        │
│                              │ 容器内执行     │        │
│                              └──────────────┘        │
│                                      │                │
│                              Hardware-assisted        │
│                              virtualization           │
└─────────────────────────────────────────────────────┘
```

**Kata 用于 Agent 的优劣**：

| 优势 | 劣势 |
|------|------|
| 完全兼容 Docker/OCI 工具链 | 启动时间 1-3s |
| 真正的 VM 级隔离 | 内存开销大（~128MB+） |
| 支持 SEV/SNP 硬件加密 | 嵌套虚拟化问题 |
| 与 K8s 原生集成 | 资源密集，不适合高频轻量调用 |

### 3. Sandbox Libraries: gVisor, nsjail, Bubblewrap — comparing isolation levels

沙箱库提供了比完整容器更轻量的进程级隔离方案，适合在单个进程内运行不可信代码。它们在 CI/CD、在线评测系统（OJ）、代码执行 Agent 中广泛使用。

#### nsjail

Google 开源的轻量级沙箱工具，广泛用于 CTF 比赛和在线代码执行。使用 Linux 命名空间、cgroups、seccomp-bpf 实现进程隔离。

```bash
# nsjail 基本用法 —— 隔离 Python 代码执行
nsjail \
  --chroot /sandbox/root \           # chroot 到沙箱根目录
  --user 65534 \                     # nobody 用户
  --group 65534 \                    # nogroup 组
  --hostname sandbox \               # 隔离 UTS namespace
  --cwd /tmp \                       # 工作目录
  --tmpfs /tmp:size=64M \            # tmpfs 内存文件系统
  --seccomp_string 'POLICY apps\
    ALLOW { read, write, openat, close, mmap, munmap, exit_group }\
    DENY { execveat, mount, umount }' \
  --rlimit_as 256 \                  # 地址空间限制
  --rlimit_nofile 32 \               # 文件描述符限制
  --time_limit 10 \                  # CPU 时间限制
  --max_cpus 1 \                     # 单 CPU
  -- \                               # 执行命令
  python3 /tmp/agent_code.py
```

**nsjail 能力特性**：

| 特性 | 支持 | 说明 |
|------|------|------|
| 挂载命名空间 | ✅ | 独立的文件系统视图 |
| PID 命名空间 | ✅ | 独立进程树 |
| 网络命名空间 | ✅ | 可完全禁用网络 |
| UTS 命名空间 | ✅ | 独立主机名 |
| IPC 命名空间 | ✅ | 独立 IPC 资源 |
| 用户命名空间 | ✅ | UID/GID 映射 |
| cgroups | ✅ | CPU/内存限制 |
| seccomp-bpf | ✅ | 细粒度系统调用过滤 |
| rlimit | ✅ | 资源软/硬限制 |
| 时间限制 | ✅ | CPU 时间和挂钟时间 |

#### Bubblewrap

Flatpak 项目使用的轻量级沙箱工具，由同一个开发者维护。Bubblewrap 的设计哲学是最小特权——沙箱没有任何默认权限，需要显式授予。

```bash
# Bubblewrap 基本用法
bwrap \
  --ro-bind /usr /usr \              # 只读挂载 /usr
  --ro-bind /lib /lib \              # 只读挂载 /lib
  --ro-bind /bin /bin \              # 只读挂载 /bin
  --tmpfs /tmp \                     # 临时文件系统
  --tmpfs /home \                    # 隔离 home 目录
  --proc /proc \                     # 挂载 /proc
  --dev /dev \                       # 最小化 /dev
  --unshare-net \                    # 隔离网络
  --unshare-ipc \                    # 隔离 IPC
  --unshare-pid \                    # 隔离 PID
  --unshare-uts \                    # 隔离 UTS
  --hostname sandbox \               # 设置主机名
  --die-with-parent \                # 父进程退出时子进程自动终止
  -- \                               # 执行命令
  python3 /tmp/agent_code.py
```

**Bubblewrap 核心特性**：
- 无 SUID 二进制文件，通过 setuid 或 user namespace 工作
- 所有权限默认拒绝，显式授予
- 支持 bind mount、tmpfs、ro-bind、symlink
- 不支持 seccomp-bpf（nsjail 的优势）
- 不支持 cgroups（需要手动配置）

#### 隔离级别对比

```
nsjail vs Bubblewrap vs Docker 隔离级别对比
─────────────────────────────────────────────────────────────────────────
特性               nsjail         Bubblewrap      Docker (默认)
────               ──────         ──────────      ─────────────
进程级隔离          ✅ PID ns      ✅ PID ns       ✅ PID ns
文件系统隔离        ✅ chroot+ns   ✅ bind mount   ✅ overlayfs
网络隔离            ✅ net ns      ✅ net ns       ✅ net ns
用户隔离            ✅ user ns     ✅ user ns      ✅ user ns
seccomp             ✅ 可编程      ❌ 不支持       ✅ 默认配置
cgroups             ✅ 原生支持    ❌ 外部配置     ✅ 原生支持
capabilities        ❌ 不适用      ❌ 不适用       ✅ drop all
启动时间            < 1ms         < 1ms           ~10ms
资源开销            基本为0        基本为0         ~5MB
配置复杂度          中             中              低
安全性              中高           中              中
典型场景            代码执行/CICD   Flatpak/桌面    通用容器
```

**选择建议**：

```python
# 选择沙箱库的决策逻辑
def choose_sandbox(requirements: dict) -> str:
    if requirements.get("needs_network"):
        return "Docker"  # 容器提供完整网络栈
    if requirements.get("needs_seccomp"):
        return "nsjail"  # 唯一支持可编程 seccomp
    if requirements.get("min_overhead"):
        return "nsjail"  # 最低开销
    if requirements.get("no_root"):
        return "Bubblewrap"  # 无需 setuid（user ns 模式）
    if requirements.get("needs_cgroups"):
        return "nsjail"  # 原生 cgroups 支持
    return "Docker"  # 默认选择
```

### 4. Code Execution Sandboxing: Python sandbox (PyPy sandbox, restricted Python, subinterpreter), WASM sandbox

代码执行是 Agent 场景中风险最高的操作之一——它允许 Agent（或被注入的攻击者）在宿主机上运行任意代码。代码执行沙箱需要提供比通用隔离更强的**运行时安全**保证。

#### Python 代码执行沙箱

**方案一：RestrictedPython**

Zope 项目的受限 Python 执行环境。通过 AST 转换（Compile-time）限制 Python 语法，移除危险的操作。

```python
"""
RestrictedPython — 限制 Python 执行

通过 AST 转换限制:
  - 禁止 import 语句
  - 禁止文件 I/O 操作
  - 禁止属性访问的 __ 魔术方法
  - 限制内置函数集合
"""

from RestrictedPython import compile_restricted, safe_globals

def execute_restricted_python(code: str, timeout: int = 5) -> str:
    """
    在 RestrictedPython 沙箱中执行用户代码
    """
    # 安全内置函数白名单
    restricted_builtins = {
        'abs': abs, 'bool': bool, 'chr': chr, 'complex': complex,
        'divmod': divmod, 'float': float, 'hash': hash, 'hex': hex,
        'id': id, 'int': int, 'isinstance': isinstance,
        'len': len, 'list': list, 'map': map, 'max': max,
        'min': min, 'ord': ord, 'pow': pow, 'range': range,
        'repr': repr, 'round': round, 'set': set, 'slice': slice,
        'sorted': sorted, 'str': str, 'sum': sum, 'tuple': tuple,
        'type': type, 'zip': zip, 'True': True, 'False': False,
        'None': None,
        # 明确移除: open, eval, exec, __import__, compile
    }

    # 构建安全全局上下文
    restricted_globals = {
        '__builtins__': restricted_builtins,
        '_getattr_': safe_globals['_getattr_'],
        '_print_': safe_globals['_print_'],
        '_write_': safe_globals['_write_'],
    }

    try:
        # 编译受限代码
        byte_code = compile_restricted(code, '<string>', 'exec')

        # 在受限命名空间中执行
        local_namespace = {}
        exec(byte_code, restricted_globals, local_namespace)

        # 只允许返回基本类型
        result = local_namespace.get('result', None)
        if isinstance(result, (int, float, str, bool, list, dict, tuple, type(None))):
            return str(result)
        return "Error: Result type not allowed"

    except SyntaxError as e:
        return f"Syntax Error: {e}"
    except Exception as e:
        return f"Execution Error: {e}"
```

**方案一的问题**：RestrictedPython 只能限制 Python 层级的操作，无法限制 C 扩展模块中的原生代码。即使不允许 os 模块，攻击者仍然可以通过 `ctypes`（如果在白名单中）间接调用 libc。

**方案二：PyPy Sandbox**

PyPy 提供了一个真正的沙箱模式——在 PyPy 解释器级别拦截所有系统调用，任何文件 I/O、网络操作、进程创建都需要通过沙箱代理。

```
PyPy 沙箱架构
───────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────┐
│ 宿主机                                              │
│  ┌───────────────────────────────────────────────┐  │
│  │ PyPy 沙箱                                     │  │
│  │  ┌──────────────────┐   ┌────────────────┐   │  │
│  │  │ 用户 Python 代码  │   │ 沙箱代理        │   │  │
│  │  │ (受限执行)        │──►│ (管理外部资源)  │   │  │
│  │  │                   │   │                │   │  │
│  │  │ open("file")      │   │ → 检查权限      │   │  │
│  │  │ os.system("cmd")  │   │ → 拒绝          │   │  │
│  │  │ socket.connect()  │   │ → 检查白名单    │   │  │
│  │  └──────────────────┘   └────────────────┘   │  │
│  └───────────────────────────────────────────────┘  │
│                                                      │
│  PyPy 沙箱在解释器层拦截所有系统调用                   │
│  → 即使使用 ctypes 或 C 扩展也无法绕过                │
└─────────────────────────────────────────────────────┘
```

**方案三：Python Subinterpreter (PEP 554)**

Python 3.12+ 引入的子解释器（Subinterpreter）提供了同一个进程内的隔离执行环境。每个子解释器有独立的 GIL、独立的命名空间、独立的模块导入缓存。

```python
"""
Python Subinterpreter 沙箱（Python 3.12+）

子解释器提供进程内隔离：
  - 独立 GIL（真正的并行执行）
  - 独立模块导入系统
  - 独立全局变量
  - 独立异常传播
"""

import _xxsubinterpreters as interpreters
import interpreters

def execute_in_subinterpreter(code: str) -> str:
    """
    在子解释器中执行不可信代码
    """
    # 创建子解释器
    interp_id = interpreters.create()

    try:
        # 在子解释器中设置受限环境
        setup_code = """
import builtins
# 移除危险内置函数
_safe_builtins = {k: v for k, v in builtins.__dict__.items()
                  if k not in ('exec', 'eval', 'compile', '__import__',
                               'open', 'input', 'breakpoint')}
builtins.__dict__.clear()
builtins.__dict__.update(_safe_builtins)
"""

        interpreters.run_string(interp_id, setup_code)
        result = interpreters.run_string(interp_id, code)

        return str(result) if result else ""

    except interpreters.RunFailedError as e:
        return f"Execution Error in subinterpreter: {e}"
    finally:
        # 销毁子解释器
        interpreters.destroy(interp_id)
```

**限制**：子解释器在同一个进程内，没有内存隔离。C 扩展模块（如 numpy、pandas）可能修改共享状态。子解释器无法完全防御基于内存损坏的逃逸。

**方案四：容器 + Python Sanbox 组合**

生产环境中推荐的方案——容器提供进程/文件系统/网络隔离，Python 沙箱提供运行时的代码级限制：

```
┌─────────────────────────────────────┐
│ Docker 容器                          │
│  ┌───────────────────────────────┐  │
│  │ Python 沙箱层                  │  │
│  │  • RestrictedPython           │  │
│  │  • 白名单内置函数              │  │
│  │  • AST 编译时检查              │  │
│  └──────────┬────────────────────┘  │
│             │                       │
│  ┌──────────▼────────────────────┐  │
│  │ 隔离的进程执行                  │  │
│  │  • subprocess 带 seccomp      │  │
│  │  • nsjail 包装                 │  │
│  └───────────────────────────────┘  │
│                                      │
│  容器安全配置:                        │
│  • read-only rootfs                 │
│  • no capabilities                  │
│  • network isolation                │
│  • resource limits                  │
└─────────────────────────────────────┘
```

#### WASM (WebAssembly) 沙箱

WASM 提供了真正的**内存安全**的代码执行沙箱——WASM 模块在沙箱化的线性内存空间中执行，无法直接访问宿主系统内存。WASI（WebAssembly System Interface）将文件系统、网络等系统调用也纳入沙箱管控。

```
WASM 沙箱架构
───────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────┐
│ 宿主进程                                             │
│  ┌───────────────────────────────────────────────┐  │
│  │ WASM 运行时 (Wasmtime / WasmEdge / Wasmer)    │  │
│  │                                                │  │
│  │  ┌──────────────────┐  ┌──────────────────┐   │  │
│  │  │ WASM 模块 #1     │  │ WASM 模块 #2     │   │  │
│  │  │ 线性内存 (64MB)  │  │ 线性内存 (64MB)  │   │  │
│  │  │ [独立内存空间]   │  │ [独立内存空间]   │   │  │
│  │  │                  │  │                  │   │  │
│  │  │ 函数调用         │  │ 函数调用         │   │  │
│  │  │ 无原生 syscall   │  │ 无原生 syscall   │   │  │
│  │  └──────────────────┘  └──────────────────┘   │  │
│  │                                                │  │
│  │  ┌────────────────────────────────────────┐    │  │
│  │  │ WASI 接口 (sandboxed system access)    │    │  │
│  │  │  • fd_read/fd_write (文件操作)          │    │  │
│  │  │  • path_open (路径过滤)                 │    │  │
│  │  │  • sock_accept (网络 + 白名单)          │    │  │
│  │  │  • clock_time_get (只读时间)            │    │  │
│  │  └────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**WASM 沙箱在 Agent 中的优势**：

| 特性 | WASM | 容器 | 微 VM |
|------|------|------|-------|
| 启动时间 | < 1ms | ~10ms | ~125ms |
| 内存开销 | ~1-5 MB | ~5-15 MB | ~50 MB+ |
| 内存安全 | ✅ 天生安全 | ❌ 无保护 | ❌ 依赖内核 |
| 系统调用隔离 | ✅ WASI 管控 | ❌ 共享内核 | ✅ 独立内核 |
| 语言支持 | 40+ 编译目标 | 任意 | 任意 |
| 性能 | ~95% 原生 | ~100% 原生 | ~90-98% 原生 |
| 可嵌入性 | ✅ 库级嵌入 | ❌ 进程级 | ❌ VM 级 |
| 细粒度权限 | ✅ WASI 策略 | ❌ 粗粒度 | ❌ 粗粒度 |

**WASM 沙箱示例**（使用 Wasmtime Rust Runtime）：

```python
# 通过 wasmtime-py 在 Python Agent 中嵌入 WASM 沙箱
import wasmtime

# 初始化 WASM 引擎（配置沙箱权限）
engine = wasmtime.Engine()
linker = wasmtime.Linker(engine)
store = wasmtime.Store(engine)

# 配置 WASI（系统接口的沙箱限制）
wasi_config = wasmtime.WasiConfig()
wasi_config.inherit_stdout()
wasi_config.inherit_stderr()
# 限制文件系统访问
wasi_config.preopen_dir("/sandbox/data", "/data")  # 只映射 /sandbox/data
# 不提供网络访问
# wasi_config.inherit_network()  # 不调用 = 无网络

store.set_wasi(wasi_config)

# 加载 WASM 模块
module = wasmtime.Module.from_file(engine, "agent_tool.wasm")

# 实例化（在沙箱中）
instance = linker.instantiate(store, module)

# 调用模块中的函数
tool_func = instance.exports(store)["execute_tool"]
result = tool_func(store, "analyze_data")
```

### 5. Browser Automation Isolation: isolated browser contexts, Playwright browser contexts, profile isolation

Agent 经常需要执行浏览器操作——网页爬取、无头浏览器渲染、Web 自动化测试。浏览器是一个巨大的攻击面：JavaScript 执行、DOM API、扩展系统、网络请求、本地存储。浏览器自动化隔离防止 Agent 的浏览器操作影响到宿主机或其他会话。

**浏览器隔离的层次**：

```
浏览器隔离层次
───────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────┐
│ 浏览器进程 (Chromium / Firefox)                      │
│                                                      │
│  ┌───────────────────────────────────────────────┐  │
│  │ 浏览器上下文 1 (Agent A)                       │  │
│  │  ┌──────────┐  ┌──────────┐                   │  │
│  │  │ 页面 1   │  │ 页面 2   │  隔离存储/Cookie    │  │
│  │  └──────────┘  └──────────┘  隔离 localStorage │  │
│  │                              隔离 IndexedDB    │  │
│  └───────────────────────────────────────────────┘  │
│                                                      │
│  ┌───────────────────────────────────────────────┐  │
│  │ 浏览器上下文 2 (Agent B)                       │  │
│  │  ┌──────────┐                                 │  │
│  │  │ 页面 1   │   完全隔离的存储和状态             │  │
│  │  └──────────┘                                 │  │
│  └───────────────────────────────────────────────┘  │
│                                                      │
│  沙箱约束:                                           │
│  • --no-sandbox 禁止（使用浏览器沙箱）                │
│  • --disable-extensions                             │
│  • --disable-gpu                                    │
│  • --disable-bleuetooth / --disable-usb              │
│  • 下载目录限制                                     │
└─────────────────────────────────────────────────────┘
```

**Playwright 隔离示例**：

```python
"""
Playwright 浏览器自动化隔离

为每个 Agent 会话创建隔离的浏览器上下文，确保：
  - 独立的 Cookie/存储/缓存
  - 独立的会话状态
  - 受限的文件下载
  - 受限的网络权限
"""

from playwright.sync_api import sync_playwright, BrowserContext, Page
from typing import Optional


class AgentBrowserIsolation:
    """
    为 Agent 提供隔离的浏览器环境
    """

    def __init__(self, download_dir: str = "/tmp/agent_downloads"):
        self.playwright = sync_playwright().start()
        self.download_dir = download_dir
        self._contexts: dict[str, BrowserContext] = {}

    def create_isolated_context(self, session_id: str) -> BrowserContext:
        """
        为 Agent 会话创建隔离的浏览器上下文
        """
        context = self.playwright.chromium.launch_persistent_context(
            # 隔离的用户数据目录
            user_data_dir=f"/tmp/browser_profiles/{session_id}",

            # 沙箱安全配置
            headless=True,
            no_sandbox=False,  # 启用 Chromium 沙箱
            ignore_default_args=[
                "--enable-automation",  # 不暴露自动化标记
            ],

            # 禁用潜在危险特性
            args=[
                "--disable-extensions",
                "--disable-sync",
                "--disable-background-networking",
                "--disable-default-apps",
                "--disable-device-discovery-notifications",
                "--disable-features=Bluetooth,Usb,WebUsb",
                "--disable-component-update",
                "--no-first-run",
                "--no-default-browser-check",
            ],

            # 地理位置权限（默认拒绝）
            permissions=[],

            # 限制下载路径
            downloads_path=self.download_dir,

            # 视窗设置
            viewport={"width": 1280, "height": 720},

            # 区域设置（防止 fingerprint 泄露）
            locale="en-US",
            timezone_id="UTC",
        )

        # 设置请求拦截（网络隔离）
        context.route("**/*", self._handle_route)

        self._contexts[session_id] = context
        return context

    def _handle_route(self, route):
        """
        路由拦截——实现网络隔离策略
        """
        url = route.request.url

        # 拒绝内部地址
        if any(prefix in url for prefix in [
            "localhost", "127.0.0.1", "10.", "172.16.", "192.168."
        ]):
            route.abort("blockedbyclient")
            return

        # 拒绝 file:// 协议
        if url.startswith("file://"):
            route.abort("blockedbyclient")
            return

        # 放行
        route.continue_()

    def destroy_context(self, session_id: str):
        """
        销毁隔离的浏览器上下文
        """
        if session_id in self._contexts:
            self._contexts[session_id].close()
            del self._contexts[session_id]

    def cleanup(self):
        """
        清理所有浏览器上下文和 Playwright 实例
        """
        for session_id in list(self._contexts.keys()):
            self.destroy_context(session_id)
        self.playwright.stop()


# ─── 使用示例 ─────────────────────────────────────────

def agent_browse_web(agent_session_id: str, url: str) -> str:
    """
    Agent 在隔离的浏览器环境中浏览网页
    """
    isolation = AgentBrowserIsolation()
    try:
        context = isolation.create_isolated_context(agent_session_id)
        page = context.new_page()
        page.goto(url, wait_until="networkidle")

        # 提取可见文本（不包含隐藏指令）
        visible_text = page.evaluate("""
            () => document.body.innerText
        """)

        return visible_text

    finally:
        isolation.destroy_context(agent_session_id)
```

**浏览器隔离的安全威胁**：

| 威胁 | 描述 | 防御措施 |
|------|------|---------|
| **浏览器沙箱逃逸** | 利用 Chromium 漏洞逃逸到宿主机 | 容器套嵌浏览器（Docker + Chromium） |
| **跨上下文数据污染** | 一个 Agent 的 Cookie 被另一个 Agent 访问 | 每个 Agent 独立 BrowserContext |
| **下载的恶意文件** | Agent 下载的恶意文件影响宿主 | 文件存储在隔离 tmpfs，执行后销毁 |
| **DNS 泄露** | 浏览器 DNS 查询泄露目标 | 使用隔离的 DNS 解析器 |
| **Canvas fingerprint** | 浏览器 fingerprint 泄露宿主机信息 | 禁用 canvas / webgl |
| **Service Worker 持久化** | 恶意 SW 在上下文中持久化 | 每个上下文新创建，用完即毁 |

### 6. Network Isolation: egress filtering, DNS filtering, network policy enforcement for tool execution

网络隔离是 Agent 隔离中最关键也最容易泄漏的一环。Agent 的工具调用（API 请求、数据获取、文件上传）本质上是网络操作，但 Agent 不应该能访问所有网络端点。

**网络隔离的目标**：
1. **阻止数据泄露** — Agent 不应能向未授权端点发送数据
2. **阻止横向移动** — Agent 不应能访问内部服务（数据库、K8s API、云元数据端点）
3. **限制攻击面** — Agent 只应访问其工作需要的外部服务

#### 网络隔离层次

```
网络隔离层次
───────────────────────────────────────────────────────────
Layer 1: 网络命名空间隔离（容器级）
  ┌──────────────┐    ┌──────────────┐
  │ 容器 A        │    │ 容器 B        │
  │ eth0: 10.0.1.2│    │ eth0: 10.0.2.2│
  │ 独立网络栈    │    │ 独立网络栈    │
  └──────┬───────┘    └──────┬───────┘
         │                   │
  ┌──────┴───────────────────┴───────┐
  │ 虚拟网络 (bridge / overlay)       │
  │ ┌────────────────────────────┐  │
  │ │ eBPF / iptables 规则       │  │
  │ │ • egress 白名单            │  │
  │ │ • 内部 IP 段丢弃           │  │
  │ │ • DNS 过滤                 │  │
  │ └────────────────────────────┘  │
  └──────────────┬──────────────────┘
                 │
         ┌───────▼───────┐
         │ 物理网络/Internet│
         └───────────────┘

Layer 2: egress 代理过滤（应用级）
  Agent ──► 代理服务器 ──► 目标服务
              │
              │ 过滤规则:
              │ • 域名白名单: api.weather.com, *.googleapis.com
              │ • 拒绝 internal.*, *.local, 10.0.0.0/8
              │ • 请求内容检查（检测数据泄露）

Layer 3: TLS 终止和检查（内容级）
  Agent ──► MITM 代理 ──► 目标服务
              │
              │ 检查加密内容:
              │ • 检测敏感数据（PII、密钥、Token）
              │ • 检测异常 payload
              │ • 速率限制
```

#### egress 过滤实现

```python
"""
Agent 网络隔离 — egress 过滤和策略执行

三种模式：
  Mode 1: 完全隔离（无网络）
  Mode 2: 白名单模式（仅允许特定域名）
  Mode 3: 代理模式（所有流量通过代理过滤）
"""

import ipaddress
import re
from dataclasses import dataclass
from typing import Optional
from urllib.parse import urlparse


@dataclass
class NetworkPolicy:
    """网络策略定义"""
    allowed_domains: set[str]           # 允许的域名白名单
    blocked_domains: set[str]           # 明确拒绝的域名
    blocked_ips: list[str]              # 拒绝的 IP 段（CIDR）
    allowed_protocols: set[str]         # 允许的协议
    max_request_size: int               # 最大请求大小（bytes）
    allow_internal: bool = False        # 是否允许访问内部网络
    allow_private_ips: bool = False     # 是否允许私有 IP
    dns_whitelist_enabled: bool = True  # 是否启用 DNS 白名单


class NetworkIsolationEngine:
    """
    网络隔离引擎——在工具调用前检查目标网络策略
    """

    # 敏感内部地址
    METADATA_IPS = [
        "169.254.169.254",  # AWS/GCP/Azure 元数据端点
        "100.100.100.200",  # 阿里云元数据端点
    ]
    PRIVATE_RANGES = [
        "10.0.0.0/8",
        "172.16.0.0/12",
        "192.168.0.0/16",
        "127.0.0.0/8",
        "::1/128",
    ]

    def __init__(self, policy: NetworkPolicy):
        self.policy = policy
        self._dns_cache: dict[str, list[str]] = {}

    def check_url(self, url: str) -> tuple[bool, str]:
        """
        检查 URL 是否允许访问

        Returns:
            (allowed, reason)
        """
        parsed = urlparse(url)

        # 1. 协议检查
        if parsed.scheme not in self.policy.allowed_protocols:
            return False, f"Protocol '{parsed.scheme}' not allowed"

        # 2. 域名检查
        hostname = parsed.hostname

        if self.policy.blocked_domains:
            for blocked in self.policy.blocked_domains:
                if hostname.endswith(blocked) or hostname == blocked:
                    return False, f"Domain '{hostname}' is blocked"

        if self.policy.allowed_domains:
            allowed = False
            for allowed_domain in self.policy.allowed_domains:
                if hostname.endswith(allowed_domain) or hostname == allowed_domain:
                    allowed = True
                    break
            if not allowed:
                return False, f"Domain '{hostname}' not in whitelist"

        # 3. IP 检查
        try:
            ip = ipaddress.ip_address(hostname)
        except ValueError:
            ip = self._resolve_dns(hostname)
            if ip is None:
                return False, f"DNS resolution failed for '{hostname}'"

        # 元数据端点检查
        if str(ip) in self.METADATA_IPS:
            return False, f"Metadata endpoint '{ip}' is blocked"

        # 私有 IP 检查
        if not self.policy.allow_private_ips:
            for private_range in self.PRIVATE_RANGES:
                if ipaddress.ip_address(ip) in ipaddress.ip_network(private_range):
                    return False, f"Private IP '{ip}' is blocked"

        # 自定义 IP 段检查
        for blocked_ip in self.policy.blocked_ips:
            if ipaddress.ip_address(ip) in ipaddress.ip_network(blocked_ip):
                return False, f"IP '{ip}' is in blocked range '{blocked_ip}'"

        return True, "OK"

    def _resolve_dns(self, hostname: str) -> Optional[str]:
        """解析 DNS 并检查白名单"""
        import socket
        try:
            ips = socket.getaddrinfo(hostname, 80)
            ip = ips[0][4][0]
            self._dns_cache[hostname] = [ip]
            return ip
        except socket.gaierror:
            return None


# ─── 使用示例 ─────────────────────────────────────────

policy = NetworkPolicy(
    allowed_domains={"api.openweathermap.org", "*.googleapis.com"},
    blocked_domains={"malicious.com", "pastebin.com"},
    blocked_ips=["0.0.0.0/0"],  # 默认拒绝所有未在 allowed_domains 中的 IP
    allowed_protocols={"https", "wss"},
    max_request_size=10 * 1024 * 1024,  # 10 MB
    allow_private_ips=False,
)

engine = NetworkIsolationEngine(policy)

# 测试
test_urls = [
    "https://api.openweathermap.org/data/2.5/weather",    # ✅
    "http://169.254.169.254/latest/meta-data/",           # ❌ metadata
    "https://malicious.com/steal",                         # ❌ blocked domain
    "http://192.168.1.1/admin",                             # ❌ private IP
    "ftp://files.example.com/data",                         # ❌ protocol
]

for url in test_urls:
    allowed, reason = engine.check_url(url)
    status = "✅" if allowed else "❌"
    print(f"{status} {url}: {reason}")
```

#### DNS 过滤

DNS 过滤是网络隔离的第一道关卡——在 DNS 解析阶段就阻止 Agent 访问恶意或未授权域名：

```bash
# CoreDNS 配置——Agent 网络隔离
# 部署为 Agent 命名空间内的 DNS 解析器

.:53 {
    # 白名单模式：只允许以下域名
    rewrite {
        # 内部服务拒绝
        (.*\.internal\.local|.*\.svc\.cluster\.local) -> HOST 0.0.0.0 {
            answer 0.0.0.0
        }
    }

    # 域名白名单
    prometheus  # 监控用

    # 拒绝列表
    template IN A blocked.domains {
        match "^(.*\.)?(pastebin\.com|malicious\.com|evil\.net)$"
        answer "{{ .Name }} 60 IN A 0.0.0.0"
    }

    # 转发到上游 DNS（过滤后）
    forward . 8.8.8.8 1.1.1.1 {
        except internal.local
    }
}
```

### 7. Filesystem Isolation: overlayfs, tmpfs, read-only root, per-session workspaces

文件系统隔离定义了 Agent 能"看到"和"写入"的文件路径。Agent 的文件系统隔离需要解决两个问题：
1. **读取隔离** — Agent 不能读取其权限范围外的文件
2. **写入隔离** — Agent 的写入不会污染宿主系统或其他 Agent 的文件

#### Linux overlayfs

Overlayfs 是容器文件系统的核心技术。它将一个只读的"下层"（lowerdir）和一个可写的"上层"（upperdir）叠加为一个统一的文件视图。

```
overlayfs 架构
───────────────────────────────────────────────────────────
只读下层 (lowerdir)              可写上层 (upperdir)
┌──────────────────────┐         ┌──────────────────────┐
│ /usr/bin             │         │ (初始为空)            │
│ /lib                 │         │                      │
│ /etc (默认配置)      │         │ Agent 运行时写入:     │
│ /tools               │         │  • /tmp/agent_*.py   │
│                      │         │  • /workspace/out.*  │
│ (多个下层可叠加)      │         │  • /etc/config.custom│
└──────────┬───────────┘         └──────────┬────────────┘
           │                                │
           └────────────┬───────────────────┘
                        │
               ┌────────▼────────┐
               │ 合并视图 (merged)│
               │                │
               │ /usr/bin       │ ← 来自下层（只读）
               │ /tmp           │ ← 来自上层（可写，但容器删除即消失）
               │ /workspace     │ ← 来自上层（整个容器生命周期内）
               │                │
               │ 写入 /etc/hosts│ → overlayfs 在上层创建副本
               │ → 实际上修改的是上层的副本，下层不变
               └────────────────┘
```

**Agent 文件系统隔离配置**：

```python
"""
Agent 文件系统隔离配置
"""

FILESYSTEM_CONFIG = {
    # 容器运行时文件系统挂载
    "mounts": {
        # 只读系统文件
        "readonly_system": [
            "/usr",         # 系统工具和库
            "/lib",         # 共享库
            "/bin",         # 核心命令
            "/etc",         # 配置（容器级默认配置）
        ],

        # 临时可写空间（tmpfs，不持久化）
        "tmpfs": [
            "/tmp",         # 临时文件
            "/var/tmp",
            "/run",
        ],

        # 只读数据挂载（Agent 需要访问的数据集）
        "readonly_data": [
            # ("/host/path/to/data", "/container/data"),
        ],

        # 工作区（可写，session 级别）
        "workspace": [
            ("/workspace", {
                "type": "tmpfs",
                "size": "100MB",
                "mode": "0700",     # 只有 Agent 用户可访问
            }),
        ],

        # 日志输出（只写，不可读）
        "log_output": [
            ("/var/log/agent", {
                "type": "none",
                "bind": {
                    "create": "file",
                    "mode": "0220",  # 只写：owner 和 group 只能写，不能读
                }
            }),
        ],
    },

    # 禁止挂载的设备
    "deny_devices": [
        "/dev/sda",       # 磁盘
        "/dev/mem",       # 物理内存
        "/dev/kmem",      # 内核内存
        "/dev/port",      # I/O 端口
        "/dev/core",      # 内核 core dump
    ],

    # 文件系统操作限制
    "fs_limits": {
        "max_file_size": "10MB",     # 单文件大小上限
        "max_files": 1000,           # 最大文件数
        "max_dir_depth": 8,          # 目录深度限制
    },
}
```

#### Per-Session 工作区管理

```python
"""
Per-Session 工作区管理

每个 Agent 会话获得一个独立、隔离的工作区。
会话结束后工作区自动销毁。
"""

import os
import shutil
import tempfile
from pathlib import Path


class SessionWorkspace:
    """
    会话级隔离的工作区管理
    """

    def __init__(self, session_id: str, base_dir: str = "/tmp/agent_workspaces"):
        self.session_id = session_id
        self.base_dir = Path(base_dir)
        self.workspace_path: Path | None = None
        self._created = False

    def create(self) -> Path:
        """
        创建隔离的工作区

        目录结构:
        /tmp/agent_workspaces/
          └── {session_id}/
              ├── input/          ← Agent 可读（只读挂载）
              ├── output/         ← Agent 可写（结果存放）
              ├── temp/           ← Agent 可写（临时文件，限制大小）
              └── metadata/      ← 系统使用（Agent 不可见）
        """
        self.workspace_path = self.base_dir / self.session_id
        self.workspace_path.mkdir(parents=True, exist_ok=True)

        # 创建子目录
        for subdir in ["input", "output", "temp"]:
            (self.workspace_path / subdir).mkdir(exist_ok=True)

        # TODO: 在实际容器中 overlay mount 到隔离的文件系统
        # 这里示意文件创建

        self._created = True
        return self.workspace_path

    def cleanup(self) -> None:
        """
        销毁工作区——清除所有 Agent 写入的数据
        """
        if self._created and self.workspace_path:
            shutil.rmtree(self.workspace_path, ignore_errors=True)
            self._created = False

    def get_input_path(self) -> Path:
        """Agent 可读的输入数据路径"""
        return self.workspace_path / "input"

    def get_output_path(self) -> Path:
        """Agent 存放结果的路径"""
        return self.workspace_path / "output"

    def __enter__(self):
        return self.create()

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.cleanup()
```

### 8. MCP-Secure Agent Kernel: capability-based authorization with policy enforcement (2025 ETH Zurich)

MCP-Secure Agent Kernel 是 2025 年 ETH Zurich 提出的研究框架，将**基于能力的安全（Capability-based Security）**引入 Agent 系统的隔离边界设计。其核心思想颠覆了传统的访问控制模型：不是"主体是否可以访问这个资源"，而是"主体持有这个资源的能力令牌吗？"。

**传统 ACL vs 基于能力的安全**：

```
传统 ACL 模型                         基于能力的模型
─────────────                         ────────────────
Agent 说: "我要读 /etc/passwd"        Agent 说: "我要读 /etc/passwd"
         ↓                                      ↓
系统检查: "Agent 对 /etc/passwd       系统检查: "Agent 持有 /etc/passwd
          有读权限吗?"                           的读能力令牌吗?"
         ↓                                      ↓
ACL 列表: Agent -> read /etc/passwd   能力: Agent 持有 capability#0x3A
         ↓                                      ↓
权限在对象侧（被访问资源）              权限在主体侧（Agent 持有的令牌）

问题:                                      优势:
  • 权限提升攻击（一个 Agent 被               • 能力不可伪造（加密哈希）
    攻破，ACL 允许访问的资源全暴露）             • 能力可传递但可撤回
  •  confused deputy 问题                      • 最小权限天然实现
  • 权限检查是"查询"而非"持有"                  • 无 confused deputy
```

#### MCP-Secure Kernel 架构

```
MCP-Secure Agent Kernel 架构
───────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────┐
│ Agent 进程                                           │
│                                                      │
│  工具调用: send_email(to="user@...")                 │
│       │                                              │
│       ▼                                              │
│  ┌───────────────────────────────────────────────┐  │
│  │ MCP-Secure Kernel                            │  │
│  │                                               │  │
│  │  1. 接收工具调用请求                            │  │
│  │  2. 提取所需能力: {cap: "email:send", ...}    │  │
│  │  3. 检查能力持有: Agent 持有 email:send#0x7F? │  │
│  │     ├── 持有 ✅ → 进入隔离执行环境              │  │
│  │     └── 未持有 ❌ → 拒绝 + 审计                │  │
│  │                                               │  │
│  │  ┌───────────────────────────────────────┐    │  │
│  │  │ 能力存储 (Capability Store)            │    │  │
│  │  │                                        │    │  │
│  │  │  ┌────────────────────────────────┐   │    │  │
│  │  │  │ Agent A 持有的能力:            │   │    │  │
│  │  │  │  • weather:read#0xA3          │   │    │  │
│  │  │  │  • search:web#0xB7            │   │    │  │
│  │  │  │  • file:read:/data/reports#0xC2│   │    │  │
│  │  │  └────────────────────────────────┘   │    │  │
│  │  └───────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────┘  │
│              │                                       │
│              ▼                                       │
│  ┌───────────────────────────────────────────────┐  │
│  │ 隔离执行环境                                   │  │
│  │                                               │  │
│  │  • 容器 / 微 VM / WASM 沙箱                    │  │
│  │  • 能力映射到文件系统/网络/系统调用权限          │  │
│  │  • 执行完成后能力自动回收                        │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

#### 能力模型在 Agent 隔离中的应用

```python
"""
MCP-Secure Agent Kernel — 基于能力的隔离边界

核心概念:
  - Capability: 不可伪造的资源访问令牌
  - Capability Store: 管理 Agent 持有的能力
  - Isolated Execution: 能力在隔离沙箱中兑现为实际权限
"""

import hashlib
import json
import secrets
from dataclasses import dataclass
from enum import Enum
from typing import Optional


class ResourceType(Enum):
    FILE = "file"
    NETWORK = "network"
    TOOL = "tool"
    DATABASE = "database"
    SHELL = "shell"


@dataclass
class Capability:
    """
    能力令牌——不可伪造的资源访问权限

    Capability 的不可伪造性通过加密哈希保证：
      token = HMAC(resource_id + action + params, kernel_secret)
    """
    resource_type: ResourceType
    resource_id: str           # 资源标识符
    action: str                # 允许的操作 (read/write/execute/...)
    params: dict               # 操作参数约束
    token: str                 # 加密令牌
    expires_at: Optional[int] = None  # 过期时间戳

    def is_valid(self, kernel_secret: str) -> bool:
        """验证能力令牌的真实性"""
        expected = self._compute_token(kernel_secret)
        return secrets.compare_digest(self.token, expected)

    def is_expired(self) -> bool:
        """检查是否过期"""
        if self.expires_at is None:
            return False
        import time
        return time.time() > self.expires_at

    def _compute_token(self, kernel_secret: str) -> str:
        """计算能力令牌的 HMAC"""
        payload = f"{self.resource_type.value}:{self.resource_id}:{self.action}:{json.dumps(self.params, sort_keys=True)}"
        return hashlib.sha256(f"{payload}:{kernel_secret}".encode()).hexdigest()[:16]


class CapabilityStore:
    """
    能力存储——管理 Agent 持有的能力集合

    能力在以下时机被赋予：
      - Agent 初始化时（默认能力）
      - 用户授权时（临时能力提升）
      - 父 Agent 委派时（能力传递）

    能力在以下时机被回收：
      - 使用完毕
      - 超过过期时间
      - 用户撤回授权
    """

    def __init__(self, kernel_secret: str):
        self.kernel_secret = kernel_secret
        self._capabilities: dict[str, list[Capability]] = {}

    def grant(self, agent_id: str, cap: Capability):
        """授予 Agent 一个能力"""
        if agent_id not in self._capabilities:
            self._capabilities[agent_id] = []
        self._capabilities[agent_id].append(cap)

    def revoke(self, agent_id: str, resource_type: ResourceType, resource_id: str):
        """撤回 Agent 的某个能力"""
        if agent_id in self._capabilities:
            self._capabilities[agent_id] = [
                c for c in self._capabilities[agent_id]
                if not (c.resource_type == resource_type and c.resource_id == resource_id)
            ]

    def check(self, agent_id: str, resource_type: ResourceType,
              resource_id: str, action: str) -> bool:
        """
        检查 Agent 是否持有满足要求的能力

        检查通过的条件:
          1. 存在匹配资源类型和 ID 的能力
          2. 能力允许请求的操作
          3. 能力的参数约束满足
          4. 令牌有效
          5. 未过期
        """
        if agent_id not in self._capabilities:
            return False

        for cap in self._capabilities[agent_id]:
            if (cap.resource_type == resource_type
                    and cap.resource_id == resource_id
                    and cap.action == action
                    and cap.is_valid(self.kernel_secret)
                    and not cap.is_expired()):
                return True

        return False

    def get_effective_permissions(self, agent_id: str) -> list[dict]:
        """获取 Agent 当前的有效权限（用于审计和展示）"""
        if agent_id not in self._capabilities:
            return []
        return [
            {
                "resource": f"{c.resource_type.value}:{c.resource_id}",
                "action": c.action,
                "expires": c.expires_at,
            }
            for c in self._capabilities[agent_id]
            if c.is_valid(self.kernel_secret) and not c.is_expired()
        ]


class SecureAgentKernel:
    """
    安全 Agent 内核——将能力模型映射到隔离执行环境

    流程:
      1. Agent 发起工具调用
      2. Kernel 提取所需能力
      3. Kernel 验证能力持有
      4. Kernel 创建隔离执行环境
      5. Kernel 将能力逐项映射为沙箱权限
      6. 执行工具
      7. 回收临时能力
    """

    def __init__(self):
        self.kernel_secret = secrets.token_hex(32)
        self.capability_store = CapabilityStore(self.kernel_secret)
        self._audit_log: list[dict] = []

    def execute_tool(self, agent_id: str, tool_name: str,
                     params: dict) -> tuple[bool, str]:
        """
        在隔离环境中执行工具调用
        """
        # 1. 解析工具调用所需能力
        required_caps = self._resolve_required_capabilities(tool_name, params)
        audit_entry = {
            "agent_id": agent_id,
            "tool": tool_name,
            "params": params,
            "required_caps": required_caps,
            "timestamp": __import__('time').time(),
        }

        # 2. 验证能力
        for cap in required_caps:
            if not self.capability_store.check(
                agent_id, cap["type"], cap["resource"], cap["action"]
            ):
                audit_entry["result"] = "denied"
                audit_entry["reason"] = f"Missing capability: {cap}"
                self._audit_log.append(audit_entry)
                return False, f"Access denied: missing {cap}"

        # 3. 创建隔离执行环境
        #    这里将能力映射为沙箱的具体隔离策略
        isolation_config = self._capabilities_to_isolation_config(required_caps)

        # 4. 在隔离环境中执行
        result = self._execute_in_isolation(
            agent_id, tool_name, params, isolation_config
        )

        # 5. 审计记录
        audit_entry["result"] = "allowed"
        audit_entry["isolation_config"] = isolation_config
        self._audit_log.append(audit_entry)

        return True, result

    def _resolve_required_capabilities(self, tool_name: str,
                                       params: dict) -> list[dict]:
        """
        工具调用 → 所需能力映射

        每个工具注册时声明其所需的能力列表。
        这里做简化的映射示意。
        """
        tool_cap_map = {
            "read_file": [{"type": ResourceType.FILE, "resource": params.get("path", ""), "action": "read"}],
            "write_file": [{"type": ResourceType.FILE, "resource": params.get("path", ""), "action": "write"}],
            "make_http_request": [{"type": ResourceType.NETWORK, "resource": params.get("url", ""), "action": "connect"}],
            "execute_python": [{"type": ResourceType.SHELL, "resource": "python", "action": "execute"}],
            "query_database": [{"type": ResourceType.DATABASE, "resource": params.get("db", ""), "action": "query"}],
        }
        return tool_cap_map.get(tool_name, [])

    def _capabilities_to_isolation_config(self, caps: list[dict]) -> dict:
        """
        能力 → 沙箱隔离配置映射

        每个能力对应到隔离环境中的一个具体权限配置
        """
        config = {
            "filesystem": {"readonly": [], "writable": []},
            "network": {"allowed_domains": [], "blocked_ips": []},
            "syscalls": {"allowed": [], "blocked": []},
        }

        for cap in caps:
            if cap["type"] == ResourceType.FILE:
                if cap["action"] == "read":
                    config["filesystem"]["readonly"].append(cap["resource"])
                elif cap["action"] == "write":
                    config["filesystem"]["writable"].append(cap["resource"])

            elif cap["type"] == ResourceType.NETWORK:
                from urllib.parse import urlparse
                domain = urlparse(cap["resource"]).hostname
                if domain:
                    config["network"]["allowed_domains"].append(domain)

            elif cap["type"] == ResourceType.SHELL:
                if cap["action"] == "execute":
                    config["syscalls"]["allowed"].extend([
                        "read", "write", "openat", "close",
                        "mmap", "munmap", "exit_group"
                    ])
                    config["syscalls"]["blocked"].extend([
                        "clone", "fork", "vfork", "execveat",
                        "mount", "umount", "keyctl"
                    ])

        return config

    def _execute_in_isolation(self, agent_id: str, tool_name: str,
                              params: dict, isolation_config: dict) -> str:
        """
        在配置的隔离环境中执行工具

        实际实现中，这里会：
          - 如果配置的 fs readonly 为空，使用完全只读文件系统
          - 如果配置的网络域名为空，禁用网络
          - 根据 syscalls 配置 seccomp 策略
          - 在 nsjail / Docker / Firecracker 中执行
        """
        # 模拟执行
        return f"Executed {tool_name} in isolated environment (config: {isolation_config})"

    def get_audit_log(self) -> list[dict]:
        """获取审计日志"""
        return self._audit_log


# ─── 使用示例 ─────────────────────────────────────────

kernel = SecureAgentKernel()
agent_id = "agent-session-001"

# 授予 Agent 读文件的权限
cap = Capability(
    resource_type=ResourceType.FILE,
    resource_id="/data/reports",
    action="read",
    params={"max_size": 10485760},
    token="",  # 将由 grant 计算
)
cap.token = cap._compute_token(kernel.kernel_secret)
kernel.capability_store.grant(agent_id, cap)

# Agent 尝试执行工具
success, result = kernel.execute_tool(agent_id, "read_file", {
    "path": "/data/reports/2025-Q1.pdf"
})
print(f"Result: {result}")  # ✅ 有权限

# Agent 尝试未授权的工具
success, result = kernel.execute_tool(agent_id, "execute_python", {
    "code": "import os; os.system('rm -rf /')"
})
print(f"Result: {result}")  # ❌ 无能力
```

**MCP-Secure Kernel 的核心创新**：

| 创新点 | 传统方法 | MCP-Secure Kernel |
|--------|---------|-------------------|
| **权限模型** | ACL/RBAC（访问控制列表） | Capability-based（基于能力） |
| **权限传递** | 需要中心化策略引擎 | 能力令牌可直接传递 |
| **权限撤回** | 修改 ACL 列表 | 设置能力过期 + 种子密钥轮换 |
| **最小权限** | 手动配置 | 能力精确对应必要操作 |
| **隔离粒度** | 用户/角色级 | 单次工具调用级 |
| **审计粒度** | 操作级 | 能力持有 + 使用全链路 |
| **跨系统** | 依赖统一身份系统 | 能力令牌自包含，无中心依赖 |

---

## Example Code: Python AgentSandbox with container lifecycle, network policy, filesystem isolation

以下是一个完整的 Agent 沙箱实现，整合了容器生命周期管理、网络隔离、文件系统隔离和资源限制。

```python
"""
AgentSandbox — Python Agent 隔离沙箱

整合：
  - 容器生命周期管理（Docker SDK）
  - 网络隔离（egress 白名单）
  - 文件系统隔离（per-session workspace）
  - 资源限制（CPU/Memory）
  - 能力管理和审计
"""

import docker
import os
import shutil
import tempfile
import time
import json
from dataclasses import dataclass
from pathlib import Path
from typing import Optional


@dataclass
class SandboxConfig:
    """沙箱配置"""
    image: str = "agent-tool-runner:latest"
    memory_limit: str = "256m"
    cpu_limit: float = 0.5
    network_enabled: bool = False
    allowed_domains: list[str] | None = None
    allowed_paths: list[str] | None = None
    timeout: int = 30          # 单次工具执行超时（秒）
    max_output_size: int = 1 * 1024 * 1024  # 1 MB
    read_only_rootfs: bool = True
    enable_seccomp: bool = True


class AgentSandbox:
    """
    Agent 执行沙箱

    生命周期:
      ┌─────────┐     ┌──────────┐     ┌──────────┐
      │ 创建     │────►│ 执行工具  │────►│ 清理销毁  │
      │ 容器     │     │ 调用     │     │ 容器     │
      └─────────┘     └──────────┘     └──────────┘
           │                │                │
           ▼                ▼                ▼
      预配工作区        网络隔离限制       销毁工作区
      挂载文件系统      资源限制控制       移除容器
      配置网络          stdout/stderr      清理日志
    """

    def __init__(self, config: SandboxConfig):
        self.config = config
        self.docker_client = docker.from_env()
        self.container: Optional[docker.models.Container] = None
        self.workspace: Optional[Path] = None
        self.container_id: Optional[str] = None

    def create(self, session_id: str) -> str:
        """
        创建隔离的 Agent 执行环境

        Args:
            session_id: 会话标识

        Returns:
            容器 ID
        """
        # 1. 创建隔离工作区
        self.workspace = Path(f"/tmp/agent_sandbox/{session_id}")
        self.workspace.mkdir(parents=True, exist_ok=True)
        (self.workspace / "input").mkdir(exist_ok=True)
        (self.workspace / "output").mkdir(exist_ok=True)
        (self.workspace / "temp").mkdir(exist_ok=True)

        # 2. 构建安全挂载
        volumes = {}
        if self.config.read_only_rootfs:
            # 只读系统文件
            volumes = {
                str(self.workspace / "input"): {
                    "bind": "/workspace/input",
                    "mode": "ro",
                },
                str(self.workspace / "output"): {
                    "bind": "/workspace/output",
                    "mode": "rw",
                },
                str(self.workspace / "temp"): {
                    "bind": "/workspace/temp",
                    "mode": "rw",
                },
            }

        # 3. 网络配置
        network_config = {}
        if not self.config.network_enabled:
            network_config = {"NetworkMode": "none"}
        else:
            # DNS 和白名单网络配置
            network_config = {
                "NetworkMode": "agent_isolated_net",
                "Dns": ["8.8.8.8"],
            }

        # 4. 安全配置
        security_config = {}
        if self.config.enable_seccomp:
            security_config = {
                "SecurityOpt": [
                    "no-new-privileges:true",
                    "seccomp=profiles/agent_seccomp.json",
                ]
            }

        # 5. 创建容器
        container_config = {
            "image": self.config.image,
            "command": ["/bin/sh", "-c", "sleep infinity"],
            "detach": True,
            "network_mode": "none" if not self.config.network_enabled else None,
            "volumes": volumes,
            "mem_limit": self.config.memory_limit,
            "cpu_quota": int(self.config.cpu_limit * 100000),
            "cpu_period": 100000,
            "read_only": self.config.read_only_rootfs,
            "security_opt": security_config.get("SecurityOpt", []),
            "cap_drop": ["ALL"],
            "cap_add": [],  # 不要添加任何 capability
            "tmpfs": {
                "/tmp": "size=64M,noexec,nosuid,nodeuid=1000",
                "/run": "size=32M,noexec,nosuid",
            },
            "user": "1000:1000",  # 非 root 用户
            "working_dir": "/workspace",
            "labels": {
                "agent_session": session_id,
                "sandbox_type": "tool_execution",
            },
        }

        self.container = self.docker_client.containers.create(**container_config)
        self.container.start()
        self.container_id = self.container.id

        return self.container.id

    def execute_tool(self, tool_script: str, timeout: Optional[int] = None) -> dict:
        """
        在隔离环境中执行工具脚本

        Args:
            tool_script: 要执行的 Python/Shell 代码
            timeout: 执行超时（秒）

        Returns:
            {"stdout": str, "stderr": str, "exit_code": int, "error": str | None}
        """
        if not self.container:
            raise RuntimeError("Sandbox not created. Call create() first.")

        timeout = timeout or self.config.timeout

        # 将工具脚本写入工作区
        script_path = self.workspace / "temp" / "agent_tool.py"
        script_path.write_text(tool_script)

        try:
            # 在容器中执行
            exec_result = self.container.exec_run(
                cmd=["python3", "/workspace/temp/agent_tool.py"],
                timeout=timeout,
                workdir="/workspace",
                user="1000:1000",
            )

            output = exec_result.output.decode("utf-8", errors="replace")

            # 限制输出大小
            if len(output) > self.config.max_output_size:
                output = output[:self.config.max_output_size] + "\n... [truncated]"

            return {
                "stdout": output,
                "stderr": "",
                "exit_code": exec_result.exit_code,
                "error": None,
            }

        except docker.errors.APIError as e:
            return {
                "stdout": "",
                "stderr": "",
                "exit_code": -1,
                "error": f"Docker API error: {e}",
            }
        except Exception as e:
            return {
                "stdout": "",
                "stderr": "",
                "exit_code": -1,
                "error": str(e),
            }
        finally:
            # 清理脚本文件
            if script_path.exists():
                script_path.unlink()

    def destroy(self) -> None:
        """
        销毁沙箱——停止、移除容器，清理工作区
        """
        errors = []

        # 1. 停止并移除容器
        if self.container:
            try:
                self.container.stop(timeout=5)
                self.container.remove(force=True)
            except Exception as e:
                errors.append(f"Container cleanup error: {e}")
            finally:
                self.container = None

        # 2. 清理工作区
        if self.workspace and self.workspace.exists():
            try:
                shutil.rmtree(self.workspace, ignore_errors=True)
            except Exception as e:
                errors.append(f"Workspace cleanup error: {e}")
            finally:
                self.workspace = None

        return errors if errors else []

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.destroy()


class SandboxPool:
    """
    沙箱池——预创建沙箱，减少启动延迟

    策略：
      - 根据负载动态调整池大小
      - 空闲沙箱超过 TTL 后自动销毁
      - 沙箱使用后清理为全新状态
    """

    def __init__(self, config: SandboxConfig, pool_size: int = 5):
        self.config = config
        self.pool_size = pool_size
        self._pool: list[AgentSandbox] = []
        self._in_use: set[str] = set()

    def acquire(self) -> AgentSandbox:
        """从池中获取一个沙箱"""
        # 尝试从池中获取空闲沙箱
        for sandbox in self._pool:
            if sandbox.container_id and sandbox.container_id not in self._in_use:
                self._in_use.add(sandbox.container_id)
                return sandbox

        # 池已空，创建新沙箱
        sandbox = AgentSandbox(self.config)
        session_id = f"pool-{int(time.time())}-{len(self._pool)}"
        sandbox.create(session_id)
        self._pool.append(sandbox)
        self._in_use.add(sandbox.container_id)
        return sandbox

    def release(self, sandbox: AgentSandbox) -> None:
        """释放沙箱回池中"""
        if sandbox.container_id:
            self._in_use.discard(sandbox.container_id)

    def warm_up(self) -> None:
        """预热池——提前创建沙箱"""
        for i in range(self.pool_size):
            sandbox = AgentSandbox(self.config)
            sandbox.create(f"warmup-{i}")
            self._pool.append(sandbox)

    def shutdown(self) -> None:
        """关闭沙箱池——销毁所有沙箱"""
        for sandbox in self._pool:
            sandbox.destroy()
        self._pool.clear()
        self._in_use.clear()


# ─── 使用示例 ─────────────────────────────────────────

if __name__ == "__main__":
    # 创建配置
    config = SandboxConfig(
        image="python:3.12-slim",
        memory_limit="128m",
        cpu_limit=0.25,
        network_enabled=False,
        timeout=10,
    )

    # 创建沙箱池并预热
    pool = SandboxPool(config, pool_size=3)
    pool.warm_up()

    # Agent 执行工具
    with pool.acquire() as sandbox:
        result = sandbox.execute_tool(
            tool_script="""
import json

# 这个代码在完全隔离的环境中执行
# 没有网络，只有受限的文件系统

result = {
    "status": "ok",
    "message": "Hello from sandbox!",
    "cwd": __import__('os').getcwd(),
    "user": __import__('os').getenv('USER', 'unknown'),
}

# 尝试写文件（应该成功——output 是可写的）
with open('/workspace/output/result.json', 'w') as f:
    json.dump(result, f)

# 尝试读系统文件（应该失败——read-only）
try:
    with open('/etc/shadow', 'r') as f:
        print("SECURITY BREACH: read /etc/shadow")
except PermissionError:
    print("OK: Cannot read /etc/shadow (read-only filesystem)")

# 尝试创建网络连接（应该失败——无网络）
try:
    import urllib.request
    urllib.request.urlopen('http://example.com')
    print("SECURITY BREACH: network access")
except Exception:
    print("OK: Network access blocked")

print(json.dumps(result))
"""
        )

        print(f"Exit code: {result['exit_code']}")
        print(f"Output:\n{result['stdout']}")

    pool.shutdown()
    print("Sandbox pool shut down.")
```

---

## Capability Boundaries: no perfect isolation, escape vulnerabilities always exist

没有完美的隔离。所有隔离技术都有攻击面——区分"理论上的隔离强度"和"工程实现中的实际安全等级"至关重要。

### 已知的容器/沙箱逃逸技术

```
容器/沙箱逃逸攻击分类
──────────────────────────────────────────────────────────────────────

CVE-2019-5736 (runC 容器逃逸)
  • 原理: 利用 runC 的 /proc/self/exe 竞态条件
  • 影响: Docker、containerd、CRI-O
  • 条件: 容器内拥有 root 权限（--privileged 或 CAP_SYS_ADMIN）
  • 防御: 升级 runC、--cap-drop=ALL、使用 user namespace

CVE-2022-0492 (cgroup 逃逸)
  • 原理: 利用 cgroup v1 release_agent 实现逃逸
  • 条件: 容器内拥有 CAP_SYS_ADMIN 或 SYS_ADMIN in cgroup
  • 防御: 升级内核、移除 CAP_SYS_ADMIN、使用 cgroup v2

CVE-2024-21626 (runC 工作目录逃逸)
  • 原理: runC exec 时工作目录设置为 /proc/self/fd/... 可逃逸到宿主机
  • 影响: Docker 25.0.0+, containerd 1.6.28-
  • 防御: 升级 runC >= 1.1.12

/proc/sysrq-trigger 攻击
  • 原理: 容器内写入 /proc/sysrq-trigger 可触发内核操作
  • 防御: 限制 /proc 挂载为只读或 masked

内核漏洞利用
  • Dirty Pipe (CVE-2022-0847): 内核管道内存损坏
  • Dirty COW (CVE-2016-5195): 写时复制竞态条件
  • 影响: 所有共享宿主机内核的隔离方案（容器、nsjail、gVisor）
  • 防御: 内核升级、使用微 VM（独立内核）

nsjail/bubblewrap 逃逸
  • 原理: 利用系统调用中遗漏的白名单
  • 常见向量: keyctl(CAP_SYS_ADMIN)、io_uring 绕过 seccomp
  • 防御: 持续跟进 seccomp 过滤规则更新

WASM 沙箱逃逸
  • 原理: WASM 运行时本身可能有内存安全漏洞
  • CVE-2023-30699 (Wasmtime): 类型混淆导致任意代码执行
  • 防御: 更新 WASM 运行时、启用所有安全特性
```

### 隔离边界的实际安全限制

```
安全限制
──────────────────────────────────────────────────────────────────────

能做到的:
  ✅ 防止普通攻击者逃逸到宿主机（99%+ 场景）
  ✅ 阻止被注入的 Agent 访问未授权的网络/文件
  ✅ 限制数据泄露的速率和总量
  ✅ 提供完整审计轨迹（谁在什么时间执行了什么操作）
  ✅ 多租户隔离（用户 A 的 Agent 不能访问用户 B 的数据）
  ✅ 防止 DoS（资源限制防止一个 Agent 耗尽所有资源）

做不到的:
  ❌ 防止国家级攻击者的"零日内核 exploit"逃逸
  ❌ 防止硬件级攻击（Rowhammer、Spectre-type side channel）
  ❌ 防止旁路攻击（通过时序、功耗、缓存推断隔离环境内信息）
  ❌ 完全防止通过合法 API 泄露数据（egress 分析是另一道防线）
  ❌ 在保证性能的同时达到最高隔离等级

永远存在的风险:
  • 内核漏洞: 共享内核的方案永远有内核级逃逸风险
  • 配置错误: 隔离方案配置不当比没有隔离更危险（虚假的安全感）
  • 供应链攻击: 隔离工具本身可能包含恶意代码
  • 侧信道: 同一硬件上的不同隔离环境可能互相影响
```

### 隔离强度评估框架

```
隔离强度评估维度
──────────────────────────────────────────────────────────────────────

1. 逃逸难度
  简单: 普通开发者水平                              [进程级隔离]
  中等: 需要操作系统漏洞知识                         [容器级隔离]
  困难: 需要内核 exploit                            [微 VM]
  极难: 需要硬件级攻击                              [完整 VM + 硬件安全]

2. 数据泄露防御
  低: Agent 进程内存可被其他进程读取                   [无隔离]
  中: OS 隔离阻止跨进程读内存                         [容器]
  高: 内存加密 + 硬件隔离                            [SEV/SNP VM]

3. 持久化能力
  低: 沙箱销毁后所有状态消失                         [tmpfs]
  中: 会话数据在销毁时清理                           [per-session workspace]
  高: 数据加密存储 + 访问审计                        [加密 + 审计]

4. 影响范围
  单工具: 一次工具调用失败不影响其他调用              [per-tool sandbox]
  单会话: 一个 Agent 崩溃不影响其他 Agent            [per-session sandbox]
  全局: 一个 Agent 逃逸影响整个系统                  [无隔离]
```

---

## Comparison: process vs container vs micro-VM vs WASM isolation levels

```
四种隔离方案的全面对比
─────────────────────────────────────────────────────────────────────────────────────────────────

维度                   进程级隔离            容器级隔离            微 VM 隔离          WASM 沙箱
                      (nsjail/bwrap)       (Docker/K8s)        (Firecracker/Kata)   (Wasmtime)
────                   ─────────────        ────────────        ─────────────────    ──────────

【安全】

内核攻击面             共享宿主机内核         共享宿主机内核       独立内核             无内核（WASI）
                      所有 syscall 经过     所有 syscall 经过     syscall 不经过      所有操作通过
                      宿主机内核            宿主机内核            宿主机内核           宿主 API 调用
内核逃逸风险           高                   中高                 低                   不适用

系统调用拦截           可编程 seccomp        默认 seccomp         硬件虚拟化           不适用
                      白名单模式            RuntimeDefault       拦截所有 syscall

内存隔离              同进程内存           cgroup 内存限制       硬件级内存隔离       线性内存沙箱
                      无硬件保护           无硬件保护            EPT 页表            内存安全天生

文件系统隔离           chroot/bind mount    overlayfs           独立 rootfs          WASI 预开放
                      需要手动配置          层叠文件系统          完整文件系统          路径白名单

网络隔离               net ns 隔离           net ns + 策略        独立网络栈            WASI 网络
                      或无网络               iptables/eBPF        Tap 设备            需宿主转发
                      

【性能】

启动时间               < 1ms                ~10-50ms             ~125ms-3s            < 1ms
                       (fork+exec)          (容器创建)            (微 VM 启动)          (模块加载)

CPU 开销               < 1%                 ~1-3%                ~3-10%               ~2-5%

内存开销               ~0 (进程自身)         ~5-15 MB             ~50-128 MB           ~1-5 MB

文件 I/O 性能          ~100% 原生            ~100% 原生           ~90-98%              通过 WASI 代理

网络吞吐量             原生                  原生                 虚拟化 ~90%           WASI 代理 ~70%

系统调用延迟           原生                  seccomp 检查         VM trap 到 VMM        WASI 函数调用
                      (nsjail 加 seccomp)     ~1-5 us 延迟        ~100-500 us 延迟       ~1 us


【开发者体验】

易用性                手动配置              声明式 Dockerfile     需要定制镜像         多语言编译
                      命令行参数多            丰富的生态            工具链较新           工具链成熟度不一

调试便利性            直接 attach            docker exec         串口控制台            宿主级调试
                      简单                  容易                  不方便                中等

生态成熟度            小众                  非常成熟              中                   增长中

语言支持              任意                  任意                  任意                 40+ 编译目标
                                                                                      需编译到 WASM

可编排性              手动                   Kubernetes            Firecracker/K8s      可嵌入宿主
                      适合单机              大规模集群             无服务器计算           适合函数粒度


【场景适配】

个人助手 Agent         ✅ 推荐               ✅ 可用              ❌ 过度              ✅ 轻量可选
  (低风险, 单用户)

多租户服务 Agent       ❌ 隔离不足           ✅ 推荐              ✅ 高安全选项        ❌ 语言限制
  (中风险, 隔离要求)

代码执行 Agent         ✅ 可选              ✅ 推荐              ✅ 强隔离选项        ✅ 推荐
  (高风险, 运行不可信代码)

金融/合规 Agent        ❌ 不满足             ❌ 不够              ✅ 推荐              ❌ 不够
  (极高风险, 监管要求)

CI/CD 流水线           ✅ 常用              ✅ 常用              ❌ 太慢              ✅ 快速编译
  (执行测试, 编译)

浏览器自动化           ❌ 不支持             ✅ 推荐              ✅ 可选              ❌ 不适用
  (无头浏览器)

数据处理               ❌ 不够               ✅ 推荐              ❌ 开销大            ❌ 不适用
  (大数据量, GPU)


【经济性】

单次执行成本           ~$0                  ~$0.0001              ~$0.001              ~$0
                       (极低)                (按容器)              (按 VM·秒)           (极低)

基础设施成本           Linux 主机            Docker/K8s 集群       KVM 主机             宿主进程
                                           containerd           Firecracker           WASM 运行时

运维复杂度            低                    中                    高                   低

规模化成本            线性                  线性                  线性（单 VM 成本高）   线性（极低）


【选择建议】

适用条件:
  • 工具调用频率高 (1000+次/秒)
  • 工具内不执行不可信代码
  • 同用户同信任域
  → 进程隔离足够安全

适用条件:
  • 多租户或多 Agent 隔离
  • 需要标准化的执行环境
  • 需要网络隔离和资源限制
  → 容器隔离性价比最优

适用条件:
  • 执行不可信用户代码
  • 高合规要求行业
  • 数据极度敏感
  → 微 VM 隔离提供最强安全保障

适用条件:
  • 函数级执行粒度
  • 语言可编译到 WASM
  • 需要极速启动和大规模并发
  → WASM 沙箱兼顾安全与性能
```

**决策矩阵**：

```python
def recommend_isolation_level(requirements: dict) -> str:
    """
    根据需求推荐隔离方案
    """
    risk_level = requirements.get("risk_level", "low")
    exec_frequency = requirements.get("exec_frequency", "medium")
    untrusted_code = requirements.get("untrusted_code", False)
    compliance = requirements.get("compliance", "none")
    language_flexibility = requirements.get("language_flexibility", True)

    if compliance in ("hipaa", "sox", "pci-dss", "finra"):
        return "Micro-VM (Kata/Firecracker)"

    if untrusted_code and risk_level == "high":
        return "Container + WASM dual sandbox"

    if untrusted_code and risk_level == "medium":
        return "Container + nsjail"

    if exec_frequency == "high" and not untrusted_code:
        return "Process sandbox (nsjail/Bubblewrap)"

    if not language_flexibility and exec_frequency == "high":
        return "WASM sandbox"

    # 默认
    return "Container (Docker)"
```

---

## Engineering Optimization: sandbox reuse, lazy initialization, pre-warmed pools

将隔离边界部署到生产环境需要解决三个问题：**启动延迟**、**资源消耗**、**吞吐量**。

### 1. Sandbox Pooling（沙箱池）

预创建沙箱，消除每次工具调用的启动延迟。

```python
"""
沙箱池优化——减少启动延迟
─────────────────────────────────

无池化:
  工具调用 ──► 创建沙箱 (50ms) ──► 执行 (100ms) ──► 销毁 (20ms)
                                  总延迟: ~170ms

有池化:
  工具调用 ──► 从池获取 (0.1ms) ──► 执行 (100ms) ──► 归还 (1ms)
                                  总延迟: ~101ms

预热池 + 懒加载:
  池大小: 10
  预热: 应用启动时创建 5 个沙箱
  懒加载: 负载超过 5 时动态创建
  回收: 空闲超过 5min 的沙箱自动销毁
"""

class AdaptiveSandboxPool:
    """
    自适应沙箱池——动态调整池大小

    策略:
      - 最小池大小: 2（始终保留）
      - 最大池大小: 20（资源上限）
      - 扩容: 当使用率 > 80% 时增加 2 个
      - 缩容: 当使用率 < 30% 且空闲超过 TTL 时回收
    """

    def __init__(self, config: SandboxConfig, min_size: int = 2, max_size: int = 20):
        self.config = config
        self.min_size = min_size
        self.max_size = max_size
        self._pool: list[AgentSandbox] = []
        self._in_use: set[str] = set()
        self._last_access: dict[str, float] = {}
        self._idle_ttl = 300  # 5 分钟空闲 TTL

    def acquire(self) -> AgentSandbox:
        """获取沙箱"""
        # 尝试获取空闲沙箱
        for sandbox in self._pool:
            cid = sandbox.container_id
            if cid and cid not in self._in_use:
                self._in_use.add(cid)
                self._last_access[cid] = time.time()
                return sandbox

        # 池未满，创建新沙箱
        if len(self._pool) < self.max_size:
            sandbox = AgentSandbox(self.config)
            sandbox.create(f"pool-{len(self._pool)}")
            self._pool.append(sandbox)
            self._in_use.add(sandbox.container_id)
            return sandbox

        # 池满，等待释放
        # 生产环境应使用队列 + 等待机制
        raise RuntimeError("Sandbox pool exhausted")

    def release(self, sandbox: AgentSandbox) -> None:
        """释放沙箱"""
        if sandbox.container_id:
            self._in_use.discard(sandbox.container_id)
            self._last_access[sandbox.container_id] = time.time()

    def adjust_pool_size(self) -> None:
        """
        动态调整池大小

        池大小 = max(min_size, min(current_usage * 1.5, max_size))
        """
        if not self._pool:
            return

        usage_rate = len(self._in_use) / len(self._pool)

        # 扩容
        if usage_rate > 0.8 and len(self._pool) < self.max_size:
            needed = min(self.max_size - len(self._pool), 2)
            for _ in range(needed):
                sandbox = AgentSandbox(self.config)
                sandbox.create(f"pool-auto-{len(self._pool)}")
                self._pool.append(sandbox)

        # 缩容（只回收长期空闲的）
        elif usage_rate < 0.3:
            now = time.time()
            to_remove = []
            for sandbox in self._pool:
                cid = sandbox.container_id
                if cid and cid not in self._in_use:
                    last = self._last_access.get(cid, 0)
                    if now - last > self._idle_ttl:
                        to_remove.append(sandbox)

            # 保留至少 min_size 个沙箱
            keep_count = max(self.min_size, len(self._pool) - len(to_remove))
            for sandbox in to_remove:
                if len(self._pool) <= keep_count:
                    break
                sandbox.destroy()
                self._pool.remove(sandbox)
```

### 2. Lazy Initialization（懒加载）

懒加载策略——不预先初始化所有组件，只在首次使用时创建，减少空闲资源占用。

```
懒加载优化
───────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────┐
│ AgentSandbox (懒加载)                                    │
│                                                          │
│  创建时:                                                  │
│    • 存储配置（几乎无开销）                                │
│    • 不创建 Docker 客户端                                 │
│    • 不分配工作区                                         │
│                                                          │
│  首次 execute_tool() 调用时:                               │
│    • 创建 Docker 客户端 (lazy)  ← 只在第一次执行时创建      │
│    • 拉取镜像 (lazy + cached) ← 只在本地无缓存时拉取       │
│    • 分配工作区 (lazy)          ← 只在需要时创建           │
│    • 创建容器 (lazy)            ← 只在需要时创建           │
│                                                          │
│  优势:                                                    │
│    • Agent 可能在整个会话中只需执行 1-2 次工具调用          │
│    • 预先创建沙箱可能浪费资源（如果 Agent 根本不用工具）     │
│    • 懒加载让没用到沙箱的场景零开销                         │
└─────────────────────────────────────────────────────────┘
```

```python
class LazyAgentSandbox(AgentSandbox):
    """
    懒加载 Agent 沙箱——只有真正执行时才创建隔离环境
    """

    def __init__(self, config: SandboxConfig):
        # 父类 __init__ 没有开销——不创建 Docker 客户端
        super().__init__(config)
        self._docker_client: Optional[docker.DockerClient] = None
        self._initialized = False

    def _ensure_initialized(self) -> None:
        """懒加载——首次使用时初始化"""
        if self._initialized:
            return

        # 创建 Docker 客户端
        self._docker_client = docker.from_env()

        # 检查镜像是否存在
        try:
            self._docker_client.images.get(self.config.image)
        except docker.errors.ImageNotFound:
            self._docker_client.images.pull(self.config.image)

        self._initialized = True

    def create(self, session_id: str) -> str:
        """覆写 create——懒加载中实际创建容器"""
        self._ensure_initialized()
        return super().create(session_id)

    def destroy(self) -> None:
        """销毁——置回懒加载状态"""
        super().destroy()
        self._initialized = False
```

### 3. CoW（Copy-on-Write）容器快照

利用 overlayfs 的 CoW 特性——所有沙箱共享同一个只读基础层，每个沙箱只维护自己的写入层。

```
CoW 容器快照
───────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────┐
│ 基础镜像 (只读下层)                                      │
│ python:3.12-slim                                        │
│ + agent-tools.tar.gz                                    │
│                                         共享给所有沙箱    │
│ 大小: ~150MB                                            │
│ 磁盘占用: 1份                                           │
└──────────────────────────┬──────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ 沙箱 A 写入层   │  │ 沙箱 B 写入层   │  │ 沙箱 C 写入层   │
│ 大小: ~2MB     │  │ 大小: ~1MB     │  │ 大小: ~3MB     │
│ /workspace/*   │  │ /workspace/*   │  │ /workspace/*   │
│ /tmp/*         │  │ /tmp/*         │  │ /tmp/*         │
└───────────────┘  └───────────────┘  └───────────────┘

总磁盘占用: ~150MB + 2MB + 1MB + 3MB = ~156MB
如果不使用 CoW: 150MB × 3 = 450MB

优化比: ~65% 空间节省
```

```python
# 在 Docker/Podman 中的 CoW 配置
# 使用 docker commit 或 Podman 的 --rootfs 实现快速快照

def create_cow_snapshot(base_container: str, session_id: str) -> str:
    """
    基于基础容器创建 CoW 快照容器
    
    原理：
      1. 基础容器创建完毕后，docker commit 保存为镜像
      2. 每个会话从该镜像创建新容器（共享只读层）
      3. 每个容器只写入自己的 overlay 层
    """
    import docker
    
    client = docker.from_env()
    
    # 如果快照镜像不存在，从基础容器创建
    snapshot_image = f"agent-snapshot:{base_container}"
    try:
        client.images.get(snapshot_image)
    except docker.errors.ImageNotFound:
        container = client.containers.get(base_container)
        container.commit(repository="agent-snapshot", tag=base_container)
    
    # 从快照镜像创建新容器
    container = client.containers.create(
        image=snapshot_image,
        command=["sleep", "infinity"],
        detach=True,
        read_only=True,
        tmpfs={"/tmp": "size=64M"},
        user="1000:1000",
    )
    container.start()
    return container.id
```

### 4. Sandbox Reuse with State Reset（可重用沙箱的状态重置）

每次使用后将沙箱重置为初始状态，避免重新创建的开销。

```python
def reset_sandbox(sandbox: AgentSandbox) -> bool:
    """
    将沙箱重置为初始状态

    重置策略:
      1. 清除工作区中 Agent 写入的文件
      2. 清除 /tmp 中的临时文件
      3. 终止残留进程
      4. 重置网络连接

    Returns:
        是否重置成功（失败时需要重建容器）
    """
    container = sandbox.container
    if not container:
        return False

    try:
        # 1. 清除工作区
        container.exec_run(
            "rm -rf /workspace/output/* /workspace/temp/*",
            user="1000:1000",
        )

        # 2. 清除临时文件
        container.exec_run(
            "rm -rf /tmp/* /var/tmp/*",
            user="1000:1000",
        )

        # 3. 终止所有用户进程
        container.exec_run(
            "pkill -u 1000 2>/dev/null || true",
        )

        # 4. 验证容器仍在运行
        container.reload()
        return container.status == "running"

    except Exception:
        return False
```

### 5. Optimized seccomp Profile

自定义 seccomp 配置——只允许 Agent 工具执行真正需要的系统调用。

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "read", "write", "close", "lseek",
        "openat", "newfstatat",
        "mmap", "munmap", "mprotect", "brk",
        "exit_group", "gettid",
        "clock_gettime", "nanosleep",
        "getrandom", "sched_yield",
        "futex"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

### 6. 工程优化效果

```python
# 各优化措施的效果评估
optimization_impact = {
    "sandbox_pooling": {
        "latency_reduction": "40-70%",
        "memory_overhead": "+10-20%",
        "complexity": "low",
        "best_for": "高频工具调用场景",
    },
    "lazy_initialization": {
        "latency_reduction": "首次调用不变，空闲场景 0 开销",
        "memory_overhead": "-100%（空闲时不占用资源）",
        "complexity": "low",
        "best_for": "低频工具调用场景",
    },
    "cow_snapshots": {
        "latency_reduction": "创建时间 -50%",
        "memory_overhead": "-65% 磁盘",
        "complexity": "medium",
        "best_for": "大规模部署",
    },
    "state_reset": {
        "latency_reduction": "避免重复创建，减少 ~80% 容器生命周期开销",
        "memory_overhead": "无额外开销",
        "complexity": "medium",
        "best_for": "长会话 + 频繁工具调用",
    },
    "custom_seccomp": {
        "latency_reduction": "syscall 检查减少 ~30%",
        "security_improvement": "攻击面减少 ~70%",
        "complexity": "high",
        "best_for": "高风险环境",
    },
}
```

---

## 关键认知

1. **隔离是最后一道防线**：当 Prompt 注入绕过了指令防御、权限控制未能阻止调用、护栏放行了输出——隔离边界是阻止实际损害的最后屏障。它不依赖上层安全的正确性。

2. **没有完美的隔离**：所有隔离方案都有攻击面。共享内核的方案面临内核漏洞风险；微 VM 方案面临 VMM 漏洞风险；硬件隔离方案面临侧信道攻击风险。隔离是"足够好"而非"绝对安全"。

3. **隔离需要分层**：进程隔离 + 容器隔离 + 文件系统隔离 + 网络隔离，多层叠加的效果远好于任何单层隔离。每增加一层，攻击者就需要多突破一层。

4. **最小环境原则**：不要给 Agent 它不需要的东西。不需要网络就不配网络栈，不需要写文件就不给写权限，不需要 GPU 就不暴露设备。

5. **开发者体验与安全强度需要平衡**：过度隔离导致开发调试困难，团队可能绕过隔离机制。设计隔离方案时要考虑开发者的日常工作流程。

6. **审计与隔离互补**：隔离限制损害范围，审计追踪损害来源。两者结合才能构成完整的 Agent 安全体系。

7. **MCP-Secure Kernel 代表未来方向**：将基于能力的安全模型引入 Agent 隔离，使得权限控制更细粒度和更可审计。能力令牌可以作为跨系统的统一授权机制。

8. **WASM 是值得关注的新方向**：提供内存安全、极速启动、低开销的代码执行沙箱。随着 WASI 生态成熟，WASM 可能在 Agent 安全中扮演重要角色。

9. **池化和懒加载是生产化必备**：没有池化，每次工具调用 50-200ms 的沙箱创建开销在高频场景下不可接受。懒加载则在低频场景下节省资源。

10. **配置错误比没有隔离更危险**：错误的隔离配置（如错误的 seccomp 规则、不正确的路径映射）会创造虚假的安全感。隔离配置必须经过严格的审计和测试。
