# AI Agent 沙箱 + 文件系统共享整体方案设计

> 调研整理：2026-05-22  
> 范围：面向"长会话型 AI Agent"（Claude / ChatGPT / Manus / DeerFlow / 国内大模型 PaaS 等）的沙箱执行环境与跨实例文件共享设计

---

## 一、问题框架：把"沙箱+文件共享"拆成五个独立维度

很多团队把"沙箱方案"当成一个原子选型，其实它是 5 个独立子问题的组合，先把维度拆清楚，后面的对比才不会乱。

| 维度 | 决定的事 | 典型选项 |
| --- | --- | --- |
| **D1. 隔离技术** | 一个沙箱进程对宿主和其他沙箱的隔离强度 | 容器 (runc) / gVisor / Kata / Firecracker microVM / WASM / 进程级 chroot+seccomp |
| **D2. 沙箱生命周期** | 沙箱实例什么时候创建、销毁、复用 | 即用即销 / 会话绑定 / idle 超时回收 / 显式 pause-resume 快照 / 长驻 workspace |
| **D3. 文件持久化层** | 文件物理存放在哪里 | 容器本地磁盘 / NAS-NFS / 对象存储 / 块存储 / 元数据分离的分布式 FS |
| **D4. 文件共享机制** | 沙箱进程怎么"看到"持久化层 | 启动期下载-退出期上传 / NAS 直挂 / FUSE over 对象存储 / Volume API 显式 commit / Block snapshot |
| **D5. 命名空间与多租户** | 不同 conversation / 不同用户的文件如何隔离 | 按目录 / 按 bucket prefix / 按 FS namespace / 按 microVM 镜像 |

任何一家产品的"沙箱方案"都是这 5 维上的一个组合点。下面先穷举每个维度的主流方案，再回到整体架构上做对比。

---

## 二、Agent Loop / 沙箱 / Session 的关系与部署拓扑

> 在讨论"沙箱怎么选、文件系统怎么共享"之前，必须先回答一个更上层的问题：
>
> **沙箱在 Agent 系统里到底处于什么位置？它和 Agent Loop、Session 是合住一个进程，还是各自分布在不同的机房？**
>
> 这个拓扑决定了后面的所有讨论：文件需不需要跨机器共享、状态丢失会不会丢话、谁来负责重启、信任边界画在哪。所以本章先把这件事讲清楚——先有整体，再有局部。

### 2.1 三个核心概念的边界

很多文档把这三个词混着用，先用一句话锁定每个概念的本职工作：

| 概念 | 本职工作 | 关键资产 | 失败时的影响 |
| --- | --- | --- | --- |
| **Agent Loop** | "想—做—看"循环：调 LLM、解析 tool_use、分发到工具、把 tool_result 喂回模型 | 模型上下文窗口、工具 schema、决策状态机 | 单步崩溃 → 当前轮次失败，可重试 |
| **Sandbox**（沙箱） | 工具执行环境：跑 bash / python / browser / 文件读写 / shell session | 进程、文件系统、网络栈、可能有持久化 workspace | 崩溃 → 当前 tool_call 失败，但会话上下文未丢 |
| **Session**（会话） | 用户对话的逻辑单元：消息流、记忆、文件命名空间、用户身份、状态恢复点 | messages 序列、user_id/conv_id、attach 文件清单 | 丢失 → 用户看到对话从头开始 |

这三个概念的关系可以画成一个"洋葱"：

```
                  ┌─────────────────────────────────────────────┐
                  │                Session                       │
                  │  (谁在对话 / 上下文 / 文件命名空间)              │
                  │                                              │
                  │   ┌──────────────────────────────────────┐  │
                  │   │           Agent Loop                  │  │
                  │   │  (LLM 调用 + 工具调度 + 状态机)          │  │
                  │   │                                        │  │
                  │   │   ┌──────────────────────────────┐   │  │
                  │   │   │        Sandbox(es)            │   │  │
                  │   │   │  (bash/python/browser/...)   │   │  │
                  │   │   └──────────────────────────────┘   │  │
                  │   └──────────────────────────────────────┘  │
                  └─────────────────────────────────────────────┘
```

**注意：这只是"逻辑包含关系"，不代表三者一定运行在同一台机器上。** 真实生产系统里，它们可能：

- 全部在同一进程（CLI 工具）；
- Agent Loop 和 Sandbox 同住一个容器，Session 落在另一个 KV 存储（早期 Manus）；
- Agent Loop 在 Worker 机房，Sandbox 在另一个 Pool，Session 走第三方 DB（Claude.ai / ChatGPT 这种 SaaS）；
- Agent Loop 在云端，Sandbox 在用户本机（Cursor Agent / Claude Cowork desktop）。

下面把这些"住在哪儿"的组合穷举成 5 种部署拓扑。

### 2.2 五种主流部署拓扑

#### 拓扑 ①：All-in-One Local（全本机一体）

```
┌─────────────────── User Machine ───────────────────┐
│                                                     │
│   ┌────────────────────────────────────────────┐   │
│   │  CLI / Desktop App Process                  │   │
│   │   ├── Agent Loop                            │   │
│   │   ├── Session State (内存 + .json 文件)      │   │
│   │   └── Tool Dispatch                         │   │
│   └─────────────┬──────────────────────────────┘   │
│                 │ exec / IPC                        │
│                 ▼                                   │
│   ┌────────────────────────────────────────────┐   │
│   │  Sandbox (bubblewrap / sandbox-exec / 子进程)│   │
│   │   workspace = 用户当前项目目录                │   │
│   └────────────────────────────────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

- **代表**：Claude Code (CLI)、Cursor、Aider、Continue、Qoder、Codex CLI
- **特点**：一切在用户机器内完成；模型 API 走出口请求，但状态、文件、工具都在本地
- **沙箱形态**：进程级 sandbox（macOS sandbox-exec、Linux bubblewrap），轻量、启动毫秒级
- **文件共享需求**：极弱——沙箱直接看用户 workspace 目录，不需要远程 FS
- **优点**：延迟最低、隐私最好、用户对文件 100% 控制
- **缺点**：算力受用户机器限制；不适合需要重型计算（如训练 / 大模型本地推理）的场景；"会话连续性"靠本地 .json 文件，跨设备难

#### 拓扑 ②：Co-located / Cohabitation（Agent Loop 和沙箱同住）

```
┌──────────── Worker Container / VM ────────────┐
│                                                 │
│   ┌──────────────────────────────────────┐     │
│   │  Agent Loop Process (长驻)            │     │
│   │   - 拉取 LLM API                      │     │
│   │   - 内嵌工具直接 fork/exec            │     │
│   └──────────┬───────────────────────────┘     │
│              │  本地 IPC / 直接 syscall          │
│              ▼                                   │
│   ┌──────────────────────────────────────┐     │
│   │  Tools 在同一个文件系统命名空间运行     │     │
│   │  bash / python / browser ...          │     │
│   │  workspace = /workspace（容器内目录）   │     │
│   └──────────────────────────────────────┘     │
│                                                 │
│   Session State: 写到容器内的 SQLite/文件       │
│                                                 │
└─────────────────────────────────────────────────┘
              ▲
              │ HTTPS API
              │
        ┌─────┴──────┐
        │  Frontend  │  (用户浏览器，仅做转发)
        └────────────┘
```

- **代表**：**Manus**（Docker 沙箱内跑 analyze→plan→execute→observe 循环）、**OpenCode**（LangChain 称之为 "agent in sandbox" 模式）、DeerFlow All-in-One Sandbox、早期 AutoGPT 长驻容器
- **特点**：把整个 agent runtime 装进沙箱，沙箱本身就是 agent 的"身体"
- **沙箱形态**：通常是 Docker 容器或 microVM，长生命周期（小时—天）
- **文件共享需求**：中等——单个会话内不需要跨机共享，但当容器重启 / 扩容时需要持久化卷
- **优点**：开发体验最像"远程开发机"；工具间共享文件零拷贝；agent 可以自己写自己用的脚本
- **缺点**：沙箱崩溃 = agent 也死；横向扩展困难（一个会话只能跑在一个容器里）；隔离边界混在一起，安全审计复杂；多租户必须严格按容器隔离

#### 拓扑 ③：Decoupled SaaS（三层解耦的云服务）

```
                ┌────────────────────────────────┐
                │     Frontend (Web / App)       │
                └──────────────┬─────────────────┘
                               │ HTTPS
                               ▼
                ┌────────────────────────────────┐
                │   API Gateway / Session Store  │
                │   - conv_id → state (DynamoDB) │
                │   - 路由 + 鉴权 + 限流           │
                └──────────────┬─────────────────┘
                               │
                ┌──────────────┴─────────────────┐
                ▼                                ▼
   ┌──────────────────────┐          ┌──────────────────────┐
   │  Agent Loop Worker   │  RPC →   │   Sandbox Pool        │
   │  (无状态 worker pool) │ ◀── RPC  │  (gVisor / Firecracker)│
   │  - 取对话上下文        │          │  - 独立调度器           │
   │  - 调 LLM             │          │  - warm pool          │
   │  - 把 tool_call 派出去 │          │  - 按 conv_id 亲和    │
   └──────────────────────┘          └──────────────────────┘
                               │
                               ▼
                ┌────────────────────────────────┐
                │  Shared FS (NAS / OSS+FUSE)    │
                │  按 namespace 隔离              │
                └────────────────────────────────┘
```

- **代表**：**Claude.ai / Claude Cowork Web**、**ChatGPT Code Interpreter**、**阿里云函数计算 AI Sandbox**、Bedrock Agents、Agentuity Runtime
- **特点**：Session、Agent Loop、Sandbox 是三个独立可扩缩的层；它们通过 RPC + 共享存储联通
- **沙箱形态**：gVisor 容器或 microVM；按 conv_id 做亲和路由（保证同一会话尽量回到同一沙箱）；idle 5–15min 回收
- **文件共享需求**：**强**——这是文件系统设计的主战场。沙箱可能销毁重启、可能跨节点漂移、可能被多个 worker 同时访问，所以必须有持久化的共享 FS（路径 ②/③/④/⑤）
- **优点**：每层独立扩缩；故障域清晰；多租户天然隔离；适合海量轻量会话
- **缺点**：链路最长，每次工具调用要跨两次 RPC；session 一致性要靠 sticky routing；崩溃恢复模型最复杂

#### 拓扑 ④：Sandbox-as-a-Service / BYO Agent（自带 Agent，平台只提供沙箱）

```
        你（开发者）                  服务商
   ┌──────────────────┐         ┌──────────────────────┐
   │  你的 Agent Loop  │         │   Sandbox API        │
   │  在你的服务器     │ ──RPC─▶│   /sandboxes/xxx     │
   │  (Python/JS SDK) │         │   /run /read /write  │
   └──────────────────┘         │   /pause /resume     │
            │                    └──────────────────────┘
            │                              │
            │                              ▼
            │                    ┌──────────────────────┐
            │                    │  Firecracker microVM │
            │                    │  / gVisor Container  │
            │                    └──────────────────────┘
            │
            ▼
   你的 Session 存储（你的 DB）
```

- **代表**：**E2B**、**Modal Sandbox**、**Daytona**、**Cloudflare Sandbox SDK**、Northflank Sandbox、CodeSandbox
- **特点**：服务商**只**提供沙箱（隔离 + 文件 + 进程），Agent Loop 由开发者自己写
- **沙箱形态**：microVM / 容器；提供 SDK 调用 `sandbox.create() / sandbox.run() / sandbox.write() / sandbox.pause()`
- **文件共享需求**：由 SDK 暴露的 API 决定（多数走 Volume API + 快照）
- **优点**：组件解耦，开发者可以选最适合自己业务的 agent 框架；服务商只需要把"安全跑代码"这件事做到极致
- **缺点**：开发者要承担 session、记忆、对话编排的全部复杂度；调用沙箱的延迟通常比 Co-located 高 50–200ms
- **观察**：这个生态正在成为新的"基础设施层"，类似当年 AWS Lambda / S3 的位置

#### 拓扑 ⑤：Federated（云端 Agent + 本地 Sandbox）

```
   ┌───────────────────────────┐
   │       Cloud (服务商)       │
   │                           │
   │   Agent Loop              │
   │   Session Store           │
   │   LLM Inference           │
   │                           │
   └─────────────┬─────────────┘
                 │ Tunnel / WebSocket
                 │ (远程工具调用 RPC)
                 ▼
   ┌───────────────────────────┐
   │       User Machine         │
   │                            │
   │   Local Bridge / Daemon    │
   │   ├── 本地 Sandbox         │
   │   │   (操作用户文件系统)    │
   │   ├── Computer Use Driver  │
   │   │   (操作鼠标/键盘/截图)  │
   │   └── 凭证 (本机已登录的 IDE / 浏览器) │
   │                            │
   └────────────────────────────┘
```

- **代表**：**Cursor Agent Mode**（云端 LLM + 本地工作区）、**Devin**（部分模式）、**Claude Cowork Desktop**（Pluto Security 逆向：Agent Loop 在沙箱 VM 内，Computer Use 在宿主 macOS 上），Goose 远程模式、Dust+本地连接器
- **特点**：模型推理放在云上（保护权重 / 用户共享算力），但工具执行必须落在用户机器上才有意义（操作用户的 git 仓库、本地文件、本机已登录的 SaaS）
- **沙箱形态**：本地进程级 sandbox 或 desktop VM；通过 tunnel 暴露给云端 agent
- **文件共享需求**：**几乎为零**——文件本来就在用户机器上，沙箱直接读写本地 FS；难点反而是网络层（NAT 穿透、断线续传）
- **优点**：兼顾"模型在云、数据在本机"的合规需求；用户身份与本机权限天然一致
- **缺点**：链路最长，且强依赖用户网络稳定性；调试难（云端 stack trace + 本地 stack trace）；安全模型最微妙——云端 Agent 一旦被劫持，可以远程控制用户机器

### 2.3 五种拓扑的横向对比

| 维度 | ① All-in-One Local | ② Co-located | ③ Decoupled SaaS | ④ Sandbox-as-a-Service | ⑤ Federated |
| --- | --- | --- | --- | --- | --- |
| Agent Loop 在哪 | 用户机器 | 沙箱内 | 服务商 worker pool | 开发者自己机器 | 服务商云端 |
| 沙箱在哪 | 用户机器 | 同 Agent Loop 容器 | 服务商沙箱 pool | 服务商沙箱 pool | 用户机器 |
| Session State 在哪 | 用户本地文件 | 沙箱内 / 外挂 KV | 独立 Session DB | 开发者自己 DB | 服务商云端 |
| 沙箱生命周期 | 进程级，秒级 | 长驻（小时—天） | idle 5–15min | API 显式控制 | 进程级 / VM |
| 文件共享需求 | 无 | 仅持久化卷 | **强**（文件系统设计核心战场） | 由 SDK 决定 | 几乎为零 |
| 跨机器共享文件 | 不需要 | 不需要（单容器） | 需要（NAS/FUSE） | 视服务而定 | 不需要 |
| 多租户隔离 | 单用户单机器 | 容器级 | namespace + IAM | sandbox_id | 用户机器内 |
| 算力上限 | 用户硬件 | 单容器配额 | 弹性扩缩 | 弹性扩缩 | 用户硬件 |
| 链路延迟 | 极低 | 极低 | 中（2 跳 RPC） | 中（1 跳 RPC） | 高（云↔本地） |
| 隐私 | 最高 | 看部署方 | 看服务商 | 看服务商 | 中（模型在云、数据在本机） |
| 信任边界 | 用户↔进程 | 容器↔宿主机 | 网关↔worker↔sandbox | API 边界 | 云↔用户机器 |
| 适合场景 | 编码助手、IDE 集成 | 长任务自治 agent | C 端 SaaS | 平台型基础设施 | 受合规约束的企业场景 |

### 2.4 选这套拓扑前必答的 6 个问题

不同拓扑会逼出不同的工程决策。下面这些问题没有"标准答案"，但**必须在开工前明确**，否则做到后面会被反复推翻：

1. **Session State 落在哪？**
   - 落在 Agent Loop 进程内存：拓扑 ② 常见，崩溃即丢
   - 落在沙箱内文件系统：拓扑 ② / ④ 常见，沙箱销毁即丢
   - 落在独立 KV / DB：拓扑 ③ / ⑤ 标配，可重启可漂移
   - **决策准则**：你能接受多大粒度的状态丢失？秒级（重新生成上一个 token）还是分钟级（重发上一条消息）？

2. **沙箱的生命周期由谁决定？**
   - Agent 自己决定（拓扑 ②，agent 写代码 → 不需要外部触发）
   - Gateway 根据用户请求触发（拓扑 ③，"用户来了再起"）
   - SDK 显式控制（拓扑 ④，开发者主动 create/destroy）
   - **决策准则**：成本敏感 → 让 Gateway 控制；体验敏感 → 让 Agent 自己控制

3. **同步还是异步工具调用？**
   - 同步：Agent Loop 阻塞等沙箱返回（拓扑 ① / ② 默认）
   - 异步：Agent 把任务丢进队列，沙箱跑完回调（拓扑 ③ / ④ 中长任务）
   - **决策准则**：90 分位 tool 耗时 > 30s 就该考虑异步，否则会话级超时不好处理

4. **会话亲和性怎么做？**
   - 同 conv_id 必须回同一节点（拓扑 ②）
   - 同 conv_id 尽量回同一节点，回不到也能恢复（拓扑 ③，gateway 路由 + shared FS 兜底）
   - 不需要亲和（拓扑 ④，每次 SDK 调用都是新会话）
   - **决策准则**：状态全在共享层 → 不需要亲和；状态在内存 → 必须亲和

5. **崩溃恢复模型？**
   - At-most-once（崩了就丢，等用户重发）—— ① / ② 常见
   - At-least-once（重启时把上一条 tool_call 重新跑一次）—— ③ / ④ 部分
   - Exactly-once（沙箱里 + Gateway 都做幂等）—— 高端 SaaS 才舍得做
   - **决策准则**：tool_call 是否幂等决定了是否能简单重放

6. **信任边界画在哪？**
   - 拓扑 ① / ⑤：模型决策完全可信（在用户机器上跑），但要防 prompt injection 让模型乱删东西
   - 拓扑 ② / ③：用户代码在沙箱里，必须强隔离（gVisor / Firecracker）
   - 拓扑 ④：服务商对沙箱内代码完全不可信，必须假设客户跑的是恶意代码
   - **决策准则**：你的"敌人模型"是谁？是用户故意搞破坏，还是模型被 prompt 注入？

### 2.5 主流产品拓扑速查

| 产品 | 拓扑 | 备注 |
| --- | --- | --- |
| **Claude Code** (CLI) | ① All-in-One | bubblewrap 沙箱，操作用户当前 git 仓库 |
| **Cursor** (本机) | ① + ⑤ | 默认本机；Agent Mode 部分调云端 |
| **Aider / Continue / Qoder / Codex CLI** | ① All-in-One | IDE / CLI 本机执行 |
| **Manus** | ② Co-located | Docker 沙箱内长驻 agent loop |
| **OpenCode** | ② Co-located | LangChain 博文里被列为"agent in sandbox"代表 |
| **DeerFlow** (字节) | ② Co-located | All-in-One Sandbox 容器 |
| **Claude.ai / Cowork (Web)** | ③ Decoupled SaaS | Anthropic rclone-filestore + sandbox pool |
| **ChatGPT Code Interpreter** | ③ Decoupled SaaS | OpenAI 容器 pool + 后端持久化 |
| **阿里云 函数计算 AI Sandbox** | ③ Decoupled SaaS | NAS / OSS 二选一持久化 |
| **Bedrock Agents** | ③ Decoupled SaaS | AWS 托管 |
| **E2B** | ④ Sandbox-as-a-Service | Firecracker microVM + Snapshot |
| **Modal Sandbox** | ④ Sandbox-as-a-Service | gVisor + Volume API + 内存快照 |
| **Daytona** | ④（也支持 ②） | Dev container 长驻模式 |
| **Cloudflare Sandbox SDK** | ④ Sandbox-as-a-Service | Container on Worker + R2 mountBucket |
| **Cursor Agent Mode** | ⑤ Federated | 云端 agent + 本地 workspace |
| **Devin** | ⑤（部分场景 ③） | 远程 IDE + 自有 sandbox |
| **Claude Cowork Desktop** | ⑤ Federated | Pluto Security 逆向：Agent 在沙箱 VM，Computer Use 在 host macOS |

### 2.6 拓扑如何决定文件共享的需求

把上面 5 个拓扑按"是否需要跨机器共享文件"画到一条轴上，正好对应本文档后面要展开讨论的 D3+D4：

```
不需要跨机文件共享                                        必须跨机文件共享
        ◀──────────────────────────────────────────────▶
   ① All-in-Local      ⑤ Federated      ② Co-located      ④ SaaS      ③ Decoupled SaaS
   (本地 FS 即可)       (本地 FS 即可)    (单容器持久化卷)   (Volume API) (NAS/FUSE 主战场)
```

- **拓扑 ① / ⑤** 沙箱直接看本地文件系统 → §三~§六的"6 种文件共享路径"几乎都不需要，POSIX 本地盘就够
- **拓扑 ②** 单容器内文件零拷贝，但容器重启需要外挂卷（路径 ② NAS / ⑥ MicroVM 快照）
- **拓扑 ④** 由平台 SDK 决定（Modal Volume / E2B Snapshot），开发者按 API 用就行
- **拓扑 ③** 是文件共享设计的真正主战场——下面 §三~§八的 6 种路径、3 种原型、推荐架构，都是为这种拓扑而生

**所以：如果你做的是拓扑 ③ 的产品（典型国内 SaaS 类 AI Agent），后面所有讨论都直接相关；如果是 ① / ⑤，可以跳到 §七 直接看陷阱清单。**

---

## 三、维度详解

### 3.1 D1：隔离技术的层级

| 隔离级别 | 代表 | 启动时间 | 资源开销 | 安全等级 | AI Agent 场景适用度 |
| --- | --- | --- | --- | --- | --- |
| **进程级**（chroot+seccomp+namespace） | bubblewrap、Apple sandbox-exec | 毫秒 | 极低 | 低 | 仅适合本地工具，不适合云端跑用户代码 |
| **容器（runc）** | Docker 默认 | 50-200 ms | 低 | 中（共享内核，CVE 多） | 内部可信代码可，外部代码不安全 |
| **用户态内核（gVisor）** | OpenAI 推测、Modal | 100-300 ms | 中 | 较高 | 主流 AI Agent 选择 |
| **轻量 VM（Kata Containers）** | 部分云厂商 | 500 ms-1s | 中高 | 高 | 折中方案 |
| **MicroVM (Firecracker / Cloud Hypervisor)** | E2B、AWS Lambda | 125-250 ms | 中 | **极高** | 跑不可信代码的首选 |
| **WASM 沙箱** | Cloudflare Workers、Wasmtime | 微秒 | 极低 | 高（受语言能力约束） | 适合纯计算工具，不适合 shell |

**关键结论**：

- 商业 AI Agent 沙箱基本都在 **gVisor / microVM** 这两档；进程级隔离只适合"我自己写的工具调用"，不能给模型直接 `bash`。
- microVM 启动时间已经压到亚秒级，启动开销不再是不选它的理由，剩下的差距在生态（rootfs、网络、监控）。
- **AI Agent 真正的"沙箱安全"不是只看 D1**，还要看：网络是否禁出（防 SSRF）、文件系统挂载是否最小化（防越权读宿主机）、是否禁掉 CAP_SYS_ADMIN 等。Anthropic 直接禁外网 + 限定 workspace 目录，比单纯依赖隔离层稳得多。

### 3.2 D2：沙箱生命周期策略

按"何时回收"分四类，每一类又决定了 D4 的可选项：

#### Pattern A：单次即销（Stateless / Per-call）
- **行为**：每次工具调用拉起一个新沙箱，调用结束立刻销毁
- **代表**：早期 OpenAI Code Interpreter v1 模型、Lambda + Bedrock Code Execution
- **优点**：最简单、安全边界最清晰
- **缺点**：无中间状态，每次都要重装依赖、重读上下文，成本高
- **必须配合的 D4**：启动期下载文件、退出期上传文件（路径 ①）

#### Pattern B：会话绑定 + 空闲回收（Session-scoped, Idle-Reaped）
- **行为**：会话首次工具调用时拉起沙箱，绑定到 conversation；空闲 N 分钟后回收（典型 5-15 min）；下次再用时重启
- **代表**：**Claude.ai (Cowork)**、**ChatGPT Code Interpreter**、阿里云函数计算 Sandbox
- **优点**：常用场景下免重启，但闲置不烧钱
- **缺点**：销毁瞬间内存态丢失，下次启动需要恢复文件
- **必须配合的 D4**：要么"销毁前 dump、启动后 load"（路径 ①），要么挂载共享存储让文件天然不丢（路径 ②③）

#### Pattern C：显式快照 (Pause/Resume)
- **行为**：客户端调用 `pause()` 把沙箱整体快照（内存+磁盘+进程）持久化，下次 `resume()` 复活
- **代表**：**E2B**、**Modal Sandbox**、**Daytona** 部分模式
- **优点**：连内存里的 Python REPL 状态都能保留，开发体验最好
- **缺点**：快照大、存储成本高；快照含敏感数据需要加密；resume 不能跨硬件代际
- **必须配合的 D4**：快照本身就是"文件共享层"——但和"工具结果对外可见"是两回事，仍然需要单独的 outputs 通道

#### Pattern D：长驻 Workspace
- **行为**：一个 workspace 长时间存在，跨 conversation、跨 day、跨重启都常驻
- **代表**：**Daytona**（dev container 思路）、**Cloudflare Sandbox SDK** 默认模式
- **优点**：等同于一台远程开发机，最自然
- **缺点**：占用资源时间最长、安全审计最难、退出策略要专门设计
- **必须配合的 D4**：通常 D3 就是直接用容器层文件系统，外加一个 Volume 让用户能下载

#### Pattern E：混合模式（生产中最常见）
真实生产环境往往是 **B 主体 + C 兜底 + D 给企业版用户**。比如：
- 普通用户：B（idle 5min 回收，靠 OSS 持久化）
- Pro 用户：C（pause/resume，会话状态完整保留）
- 企业用户：D（长驻 workspace，按月计费）

### 3.3 D3 + D4：文件持久化层 与 共享机制（核心问题）

这是整篇文档的重头戏。AI Agent 文件共享有 **6 种主流路径**，按"显式程度"从下往上排：

#### 路径 ①：启动期下载 / 退出期上传（"打包搬运"模式）
```
沙箱启动 → 从 OSS 拉 tarball 解压到本地 → 工作 → 退出前打包上传 OSS
```

- **物理介质**：对象存储（OSS / S3 / R2）
- **共享语义**：弱（只在销毁/启动这两个时刻同步）
- **优点**：实现极简单，所有云厂商都支持，沙箱里就是普通本地磁盘 → POSIX 完美
- **缺点**：销毁瞬间未上传成功就丢；启动慢（GB 级依赖解压几十秒）；并发会话需要锁
- **代表**：早期 OpenAI Code Interpreter、阿里云函数计算"工作目录持久化"、很多自研系统
- **适合场景**：单线程会话、文件总量 < 数百 MB、对启动速度不极致敏感

#### 路径 ②：NAS / NFS 直挂（"网络盘"模式）
```
沙箱启动时 mount nfs://nas-server/<user>/<conv> → /mnt/user-data
```

- **物理介质**：AWS EFS / Azure Files / 阿里云 NAS / 自建 NFS / Ceph
- **共享语义**：完整 POSIX，强一致（EFS）或 close-to-open（NFSv3）
- **优点**：写后立即可读、支持 flock、沙箱看起来就是普通文件夹；多沙箱并发协作天然支持
- **缺点**：成本高（EFS Standard 约 $0.30/GB/月，是 S3 的 13 倍）；横向扩展受限于 NAS 集群；按目录隔离需要 quota；NFS 挂载启动有 200-500ms 开销
- **代表**：**阿里云函数计算 Sandbox（NAS 动态挂载）**、企业自建 AI 平台、HPC 改造
- **适合场景**：企业内部、强一致性需求、多 worker 协同写、文件大小变化巨大需要弹性

#### 路径 ③：对象存储 + FUSE 挂载（"伪装成文件系统"模式）
```
沙箱启动 → rclone/s3fs/JuiceFS mount → /mnt/user-data (背后是 OSS)
```

- **物理介质**：对象存储 + FUSE 客户端
- **共享语义**：因 FUSE 实现而异（s3fs 弱、JuiceFS 强、Mountpoint 只读）
- **优点**：成本接近对象存储原价、namespace 天然多租户、可跨 Region
- **缺点**：POSIX 语义靠模拟、写一致性窗口靠 cache 配置、随机写需要先全文下载（vfs_cache_mode=full）
- **代表**：**Anthropic Claude.ai / Cowork（rclone-filestore）**、Cloudflare Sandbox SDK `mountBucket()`、阿里云 OSSFS、AWS Mountpoint S3
- **适合场景**：海量轻量会话、读多写少、成本敏感、跨地域

#### 路径 ④：元数据分离的分布式文件系统（"两层存储"模式）
```
沙箱启动 → JuiceFS mount → 元数据走 Redis/TiKV，数据走 OSS
```

- **物理介质**：元数据库（Redis Cluster / TiKV / FoundationDB） + 对象存储数据层
- **共享语义**：完整 POSIX、强一致、原子 rename
- **优点**：同时拿到对象存储的弹性 + NAS 的语义；小文件元数据飞快；一致性窗口由元数据库保证
- **缺点**：要多维护一套元数据库（Redis 集群是新的故障源）；冷启动比直挂 NAS 慢
- **代表**：**JuiceFS**（Juicedata）、阿里云 CPFS、Tencent CFS Turbo、Lustre on object
- **适合场景**：大规模 AI 训练 + Agent 共用一套存储、对一致性和性能都有高要求

#### 路径 ⑤：Volume API（"显式 commit"模式）
```
sandbox.write("/data/x.csv", ...)
sandbox.volume.commit()   # 显式触发持久化
sandbox2 = create_sandbox(volume="vol_abc")
sandbox2.read("/data/x.csv")
```

- **物理介质**：平台托管（底层通常是对象存储 + 元数据）
- **共享语义**：commit 后强一致；commit 前只在当前沙箱可见
- **优点**：边界最清晰，不会出现 cache 旧读 bug；适合无人值守 batch
- **缺点**：用户/Agent 必须显式调 commit，体验不够透明
- **代表**：**Modal Volume**、**E2B Filesystem API**（部分）、**Cloudflare Durable Object Storage**
- **适合场景**：函数即服务模型、明确划分"输入-计算-输出"三段的批处理任务

#### 路径 ⑥：MicroVM 块存储快照（"虚拟机磁盘"模式）
```
pause() → 把 microVM 的整块磁盘 + 内存 dump 成 image
resume() → 从 image 恢复
```

- **物理介质**：块存储（EBS / 阿里云云盘 / 本地 NVMe + 后台同步）
- **共享语义**：单实例独占，不支持多沙箱并发共享
- **优点**：恢复速度快（亚秒级 resume）；支持 REPL 等纯内存态
- **缺点**：不能跨实例共享文件；快照大；按容量计费贵
- **代表**：**E2B Snapshot**、Firecracker 原生快照、**Modal Sandbox memory snapshot**
- **适合场景**：单会话内频繁中断恢复、需要保留 Python 解释器状态

### 3.4 D5：命名空间与多租户隔离

实际生产中常见的三层 ID 结构：

```
user_id ──┐
          ├──→ namespace = f"{user_id}/{workspace_id}/{conversation_id}"
workspace_id ──┘
          │
          └──→ 物理映射：
                  - 路径 ①: oss://bucket/{namespace}/state.tar.gz
                  - 路径 ②: nfs://server/exports/{namespace}/
                  - 路径 ③: rclone-filestore://{namespace}/
                  - 路径 ④: juicefs:/{namespace}/
                  - 路径 ⑤: modal_volume_{namespace}
                  - 路径 ⑥: snapshot_{namespace}.img
```

几个一定要做对的细节：

- **不要把 conversation_id 直接当文件名**：要 hash 后做 prefix sharding（避免对象存储的 1k QPS/前缀限流）。
- **删除策略要写死**：会话过期 N 天后自动 GC，否则存储费用会失控。
- **跨 namespace 严禁穿透**：用 IAM/STS 临时凭证 + path prefix 限制，单次只能 list 自己的目录。
- **审计日志独立**：transcripts 这种"只读记录"必须沙箱不可写——Anthropic 在 rclone 配置里直接 readonly:true，是好实践。

---

## 四、各家产品方案剖析（横向对比）

### 4.1 概览矩阵

> **表格中的 D2 / D4 列均为代号简写，下方对照表与单元格内括号注释配合阅读**
>
> **D2 生命周期模式**（详见 §3.2）：A=单次即销 / B=会话+idle 回收 / C=显式快照 pause-resume / D=长驻 workspace / E=混合分级
>
> **D4 文件共享路径**（详见 §3.3）：①=打包搬运 / ②=NAS 直挂 / ③=FUSE+对象存储 / ④=JuiceFS 元数据分离 / ⑤=Volume API / ⑥=MicroVM 快照

| 产品 | D1 隔离 | D2 生命周期 | D3 持久化 | D4 共享路径 | D5 隔离粒度 |
| --- | --- | --- | --- | --- | --- |
| **Claude.ai (Cowork)** | 容器 (gVisor 风格) | **B(会话+idle 5min 回收)** | 自研对象存储 `rclone-filestore` | **③(FUSE+对象存储)** | `claude_chat_<conv_id>` namespace |
| **Claude API code_execution** | 容器 | **C-lite(显式 `container_id` 复用，30 天过期)** | 容器本地 5GB + Files API | **⑤(Volume API/Files API)** | `workspace_id` |
| **Claude Code (CLI)** | 进程级 sandbox-exec / bindfs | **D(长驻本机)** | 本机磁盘 | 无（直接本地，不需共享） | 无 |
| **ChatGPT Code Interpreter** | 容器（推测 gVisor） | **B(会话+idle ~10min 回收)** | 容器内 `/mnt/data` + 后端持久化 | **①(打包搬运，推测)** | `conversation_id` |
| **E2B** | Firecracker microVM | **A(默认即销)/C(可显式 pause/resume)** | microVM 磁盘 + 快照 | **⑥(MicroVM 快照)为主，可挂 Volume** | `sandbox_id`，最长 24h |
| **Modal Sandbox** | gVisor + memory snapshot | **C(快照 resume)** | Modal Volume + Image | **⑤(Volume API 显式 commit)** | `app_name + volume_name` |
| **Daytona** | Linux 容器 | **D(长驻 workspace)** | Docker volume / 本地盘 | 直接本地 + Git 同步 | `workspace_id` |
| **Cloudflare Sandbox SDK** | 容器（Container on Worker） | **B/D(可配置)** | Durable Object 元数据 + R2/S3 mountBucket | **③(FUSE/mountBucket)+⑤(DO storage)** | `sandbox_id` |
| **阿里云 函数计算 AI Sandbox** | 函数计算实例 | **B(实例级 idle 回收)** | OSS / NAS 二选一 | **②(NAS 直挂)或 ③(OSSFS)** | 按实例 + 子目录 |
| **字节 DeerFlow** | 容器（"All-in-One Sandbox"） | **B(会话级 idle 回收)** | 本地 + skill loader | **①(打包搬运)+skill 静态资源** | `conversation_id` |
| **Manus（公开资料推测）** | microVM/容器 | **B+C(会话+可 pause)** | 对象存储 | **③(FUSE)/①(打包搬运) 混合** | `task_id` |
| **OpenAI Responses API + Code Interpreter** | 容器 | **B(containers 资源对象 idle 回收)** | 容器关联存储 | **⑤(Volume API/containers API)** | `container_id` |

### 4.2 三种典型架构原型

把上面的产品归纳为三种"原型设计"，分别对应你提到的三种方案：

#### 原型 ⓐ：你说的方案 1 ——"打包搬运型"（路径 ①）

```
沙箱启动
  └─→ 从 OSS 下载 state.tar.gz
        └─→ 解压到 /workspace
              └─→ Agent 工作
                    └─→ idle 5min
                          └─→ 销毁前 tar 整个 /workspace 上传 OSS
                                └─→ 容器销毁
```

**真实采用者**：早期 OpenAI Code Interpreter、字节 DeerFlow、大量国内自研 Agent 平台

**为什么会选这个**：
- 沙箱内部不需要任何 FUSE/NFS，POSIX 语义最干净
- 对象存储成本最低，归档周期长
- 容易加密：tarball 整个加密就行，不需要文件级 KMS

**核心问题**：
- 启动延迟与状态大小线性相关（10GB 的 node_modules 解压要 20-30s）
- 销毁时如果上传失败 → 数据丢失（必须有 retry+落盘 fallback）
- 不支持沙箱内并发协作（多 worker 同时跑会冲突）

**优化套路**：
- 状态分层：**热数据**（小，每次都打包）+ **冷数据**（大，依赖镜像层）+ **不打包**（node_modules 用 lockfile 重装）
- 增量上传：只上传 diff 而非全量 tarball（zstd + content-addressed）
- 双写：销毁前先写本地 NVMe，再异步上传 OSS，宿主机有定时哨兵兜底

#### 原型 ⓑ：你说的方案 2 ——"NAS 直挂型"（路径 ②）

```
沙箱启动
  └─→ mount nfs://nas/<conv_id> → /mnt/user-data
        └─→ Agent 工作（写 NAS 即写持久化）
              └─→ idle 5min
                    └─→ umount + 销毁，NAS 数据天然保留
```

**真实采用者**：阿里云函数计算 Sandbox（NAS 模式）、企业内部自建 AI Agent 平台、HPC + AI 融合场景

**为什么会选这个**：
- 状态对沙箱透明，启动只多 200ms 挂载时间
- 完整 POSIX、强一致、支持 flock —— 写出来的代码跟跑在物理机上一样
- 多 worker 并发、跨实例共享、人工干预（直接 SSH 到 NAS 查看）都很方便

**核心问题**：
- **成本**：EFS / NAS 单价是对象存储的 5-15 倍，海量小会话会非常贵
- **横向扩展上限**：单 NAS 集群通常 IOPS/吞吐有顶（EFS Provisioned 也只能开到 GB 级吞吐）
- **多租户隔离**：要靠子目录 + ACL，做错了一行就是数据穿透事故
- **跨 Region**：NAS 不能跨 Region，多地部署需要副本同步方案

**优化套路**：
- 冷热分层：热的最近 7 天放 NAS，冷的归档到 OSS
- 配额：每个 conv 给 5GB 硬限，超出转到对象存储
- 双 NAS：把"读密集"和"写密集"切到不同 NAS 集群

#### 原型 ⓒ：Anthropic 方案 —— "对象存储 + FUSE 透明挂载型"（路径 ③）

```
沙箱启动
  └─→ rclone-filestore multimount --config /tmp/rclone-mount-config.json
        └─→ 多个挂载点出现（uploads/outputs/transcripts/tool_results）
              └─→ Agent 工作（写 FUSE 即写远端对象存储）
                    └─→ idle 5min 销毁，下次重启重新挂载
                          └─→ 文件全程在远端，本地 cache 仅性能优化
```

**真实采用者**：Anthropic Claude.ai/Cowork、Cloudflare Sandbox SDK、Modal（部分场景）、阿里云 OSSFS 模式

**为什么会选这个**：
- 拿到对象存储的弹性（11 个 9 持久性、无限扩展、跨 Region）和 NAS 的"看起来像文件夹"的体验
- 多挂载点可以独立配权限：`uploads/` readonly、`outputs/` readwrite、`transcripts/` readonly
- 多租户极简：一个 namespace 一个挂载，无需 ACL 大表

**核心问题**：
- **写一致性窗口**：FUSE VFS cache + 对象存储最终一致 → 写后立即读可能拿到旧版（这就是 Anthropic outputs cache 3600s 的坑）
- **随机写**：对象存储不支持范围写，需要 vfs_cache_mode=full 把整个文件下到本地改完再回写 → 大文件慢
- **元数据操作慢**：list 一个有 10 万小文件的目录要发 100+ 次 LIST API → 几十秒
- **rename 非原子**：对象存储 rename 是 copy+delete，中间崩溃就是文件双份

**优化套路**：
- 给"会被频繁修改"的目录关掉 cache（cache_duration_s=1）
- 给"只读历史"的目录开长 cache（3600s+）
- 把 FUSE 路径作为唯一写入路径（不允许工具绕开 FUSE 直接写后端）→ 缓存自然一致
- 元数据走独立 KV（这就是 JuiceFS 路径 ④ 的设计）

### 4.3 三种原型的本质区别

| 维度 | ⓐ 打包搬运 | ⓑ NAS 直挂 | ⓒ FUSE+对象存储 |
| --- | --- | --- | --- |
| 沙箱内是否有 FUSE/NFS | ❌ 纯本地 FS | ✅ NFS | ✅ FUSE |
| 状态同步时机 | 启动+销毁两点 | 实时 | 实时（带 cache） |
| 启动延迟 | O(state size) | 200ms 量级 | 50-200ms |
| 写后立即可读 | ✅（本地） | ✅（强一致） | ⚠️ 取决于 cache |
| 多沙箱并发协作 | ❌ | ✅ | ⚠️ 取决于 FUSE 实现 |
| 单 GB·月成本（毛估） | $0.023（OSS） | $0.30（EFS） | $0.023+$少量缓存 |
| 多租户隔离 | bucket prefix | 子目录+ACL | namespace |
| 大依赖（如 node_modules） | 必须缓存策略 | 直接放 NAS | 可放但 list 慢 |
| 跨 Region | ✅ | ❌ | ✅ |
| 实现复杂度 | 低 | 中 | 高（要 fork rclone/写 FUSE） |

---

## 五、生命周期策略的工程细节

下面整理"生产级 idle 5min 销毁 + 重启恢复"这条最常见的链路，把容易踩坑的点列清楚。

### 5.1 沙箱状态机

```
        ┌────────────┐
        │  COLD      │ (无沙箱)
        └─────┬──────┘
              │ 用户触发工具调用
              ▼
        ┌────────────┐
        │ STARTING   │ (拉镜像、挂载、初始化)
        └─────┬──────┘
              │ 启动完成
              ▼
        ┌────────────┐    工具调用
        │ ACTIVE     │ ◀──────────┐
        └─────┬──────┘            │
              │                    │
              │ 5min 无活动        │
              ▼                    │
        ┌────────────┐    新调用    │
        │ IDLE       │ ────────────┘ (优雅复活)
        └─────┬──────┘
              │ 超过保留窗口
              ▼
        ┌────────────┐
        │ STOPPING   │ (持久化、解挂载)
        └─────┬──────┘
              │
              ▼
        ┌────────────┐
        │ TERMINATED │
        └────────────┘
```

### 5.2 关键时序点的设计

> **本节涉及的 D4 共享路径代号回顾**：①=打包搬运 / ②=NAS 直挂 / ③=FUSE+对象存储 / ④=JuiceFS 元数据分离 / ⑤=Volume API / ⑥=MicroVM 快照

#### 启动期（COLD → ACTIVE）
- **目标**：< 2 秒
- **要素**：基础镜像预热（机器池常驻 N 个 warm pool）+ 快速挂载（NFS 200ms / FUSE 100ms / OSS 拉取看大小）
- **失败处理**：若挂载失败，沙箱必须 fail fast 而非降级到本地 FS（防止"以为有持久化其实没有"）

#### 销毁期（ACTIVE → TERMINATED）
- **目标**：所有 dirty 数据落盘
- **顺序**：
  1. 停止接受新工具调用（GW 层先把路由摘掉）
  2. 等当前命令结束（带超时，超时 SIGTERM）
  3. flush FUSE / 上传 tar / commit volume（按 D4 路径）
  4. umount
  5. 销毁容器
- **关键陷阱**：
  - 销毁触发器是"idle 5min"，但 *人* 此刻可能正在打字 → 必须有"用户活跃信号"提前续约
  - FUSE umount 失败会让对象存储中残留写未提交的数据 → 加 retry + 哨兵 sweeper 兜底
  - 销毁时机不能是"沙箱进程崩溃" → 那时 dirty 数据无法 flush，需要 D4 选项支持崩溃恢复（**路径 ②(NAS 直挂)、④(JuiceFS 元数据分离)、⑤(Volume API)** 天然支持，**路径 ①(打包搬运)** 必须有定期 checkpoint）

#### 复活期（TERMINATED → ACTIVE，下次问问题时）
- **目标**：< 5 秒，让用户感觉沙箱"一直都在"
- **路径 ①(打包搬运) 的优化**：
  - 不解压所有文件，只 lazy load（FUSE-on-tar 或自实现 overlay）
  - 上次 active 时记录"最近用过的文件 top-K"，启动时优先解压
- **路径 ②(NAS 直挂)、③(FUSE+对象存储) 几乎免开销**：mount 即可见
- **路径 ⑥(MicroVM 快照) 的开销主要是镜像下载**：本地保留最近 N 个快照命中即秒级

### 5.3 多副本竞争的设计

如果一个 conversation 同时被两个客户端打开（例如用户网页端和移动端），可能并发请求同一个 namespace：

| D4 路径 | 并发友好度 | 推荐策略 |
| --- | --- | --- |
| **①(打包搬运)** | ❌ 危险，覆盖写 | 加分布式锁（Redis SETNX），只允许一个 active |
| **②(NAS 直挂)** | ✅ 文件锁原生支持 | 应用层用 flock |
| **③(FUSE+对象存储)** | ⚠️ rename 非原子 | 写入用 content-addressed + 后写赢 |
| **④(JuiceFS 元数据分离)** | ✅ 元数据强一致 | 同 ②(NAS 直挂) |
| **⑤(Volume API)** | ⚠️ commit 冲突 | 平台层做 OCC（version check） |
| **⑥(MicroVM 快照)** | ❌ 单实例独占 | 强制 active 沙箱 ≤ 1 |

---

## 六、文件共享机制选型决策树

```
          需要支持多沙箱并发写同一个文件？
                 │
        ┌────────┴────────┐
       Yes                No
        │                 │
        ▼                 ▼
   需要强 POSIX        总状态大小？
   (rename/flock)?         │
        │           ┌──────┴──────┐
   ┌────┴────┐    < 500MB     > 500MB
   Yes      No      │              │
    │       │       ▼              ▼
    │       │   路径 ①         需要写后立即一致读？
    │       │   (打包搬运)         │
    ▼       ▼                ┌────┴────┐
路径 ②    路径 ③            Yes        No
(NAS)    (s3fs/rclone)       │          │
                             ▼          ▼
                         路径 ②      路径 ③
                         或 路径 ④   (FUSE+对象)
                         (JuiceFS)
                             │
        预算允许多维护一套元数据库？
                ┌──────┴──────┐
              Yes              No
               │                │
               ▼                ▼
          路径 ④           路径 ② (NAS)
          (JuiceFS)
```

### 选型 Cheatsheet

| 你的场景 | 推荐路径 | 核心理由 |
| --- | --- | --- |
| 个人 demo / POC | ① 打包搬运 | 最简单，能用就行 |
| 国内 SaaS、海量短会话 | ③ OSSFS / rclone | 成本最低 |
| 企业内部、协作多 worker | ② NAS 直挂 | POSIX 语义最干净 |
| AI 训练 + Agent 共用 | ④ JuiceFS | 元数据强一致 + 弹性 |
| 函数即服务 / 批处理 | ⑤ Volume API（Modal 风格） | 边界清晰 |
| 需要保留 REPL 状态 | ⑥ MicroVM 快照（E2B） | 内存态可恢复 |
| 大模型 PaaS 公网产品 | ③ + ① 混合 | uploads/outputs FUSE，依赖打包 |

---

## 七、常见问题与陷阱清单

### 7.1 共享层陷阱

1. **写后立即读 stale**：FUSE cache > 1s 的几乎都会踩；解决：写路径强一致 / cache 设短 / 显式 fsync+invalidate
2. **list 大目录慢**：对象存储 LIST API 单次 1000 keys，10 万文件要 100 次调用；解决：限制目录深度 / 元数据走 KV / 分页缓存
3. **rename 非原子**：对象存储和大部分 FUSE 都做不到；解决：用 content-addressed 命名 + 业务层处理"覆盖"语义
4. **小文件爆炸**：node_modules / .git 这种动辄 10 万文件的目录在对象存储上是噩梦；解决：tar 起来当一个对象存
5. **跨 Region 一致性**：对象存储跨 Region 通常异步复制，几秒延迟；解决：把 active conversation 锁在一个 Region

### 7.2 生命周期陷阱

6. **idle 判定不准**：只看 stdin/stdout 不够，模型输出 stream 中间也是 idle；解决：用业务事件（tool_call_start/end）而非 IO 事件
7. **销毁瞬间数据丢**：用户最后一个工具调用刚写完文件就触发 idle 销毁，flush 来不及；解决：销毁前先冻结 5s 等所有写入返回
8. **同时复活两次**：用户关掉重新打开，两个 STARTING 同时发起；解决：分布式锁 + 单一 owner
9. **快照膨胀**：pause/resume 默认会把所有内存 dump，包括缓存的大文件；解决：定期 drop_caches 再快照

### 7.3 多租户陷阱

10. **prefix 穷举**：用 conversation_id 直接当路径，攻击者可遍历；解决：HMAC 之后再做路径
11. **临时凭证泄漏**：把 OSS AK 直接放沙箱里 → 模型可读；解决：STS Token + 单 prefix 限制 + 短 TTL
12. **审计日志可写**：模型可以改 transcripts → 可被用来抹除痕迹；解决：transcripts 必须 readonly 挂载 + WORM 后端
13. **跨租户软链接**：FUSE 实现里 symlink 指向其他 namespace；解决：禁掉 symlink 或 chroot 隔离

### 7.4 隔离与安全陷阱

14. **网络出站未禁**：模型可以 curl 内网；解决：默认禁出，需要时白名单
15. **/proc 暴露**：从 /proc 读到宿主机信息或其他容器 PID；解决：gVisor / microVM 必选
16. **资源 fork 炸弹**：模型生成的代码 fork 几万进程；解决：cgroups pids.max + memory.limit
17. **挂载点被覆盖**：模型 mount --bind 把 outputs 改指向其他目录；解决：禁 CAP_SYS_ADMIN

---

## 八、推荐架构（以国内云原生场景为参考）

如果今天要新设计一套生产级 AI Agent 沙箱+文件共享系统，下面是一个工程上比较稳健、各家方案优点都吸纳的架构：

```
┌────────────────────────────────────────────────────────────────────┐
│                        Gateway (Java)                               │
│  - 鉴权、限流、对话路由、沙箱生命周期编排                             │
│  - 维护 conversation → sandbox_instance 映射                         │
│  - 颁发短 TTL STS Token（限定 namespace prefix）                     │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│                  Sandbox Pool (gVisor 容器 + warm pool)              │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │ Sandbox Container (per conversation, idle 5min reaped)    │      │
│   │   ┌──────────────────────────────────────┐               │      │
│   │   │ /home/agent      ← tmpfs，本地工作区   │               │      │
│   │   ├──────────────────────────────────────┤               │      │
│   │   │ /mnt/uploads     ← FUSE, RO, cache 1s │ 用户上传       │      │
│   │   │ /mnt/outputs     ← FUSE, RW, cache 1s │ AI 产物       │      │
│   │   │ /mnt/workspace   ← NFS, RW            │ 大文件协作    │      │
│   │   │ /mnt/transcripts ← FUSE, RO, cache 60s│ 历史记录      │      │
│   │   └──────────────────────────────────────┘               │      │
│   └─────────────────────────────────────────────────────────┘      │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┬─────────────────┐
        ▼              ▼              ▼                 ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐      ┌─────────┐
   │ OSS     │    │ JuiceFS │    │ NAS     │      │ Redis   │
   │(冷归档)  │    │(热共享) │    │(协作)   │      │(元数据) │
   └─────────┘    └─────────┘    └─────────┘      └─────────┘
```

### 设计要点

> **本节涉及的代号回顾**：D2 生命周期 B=会话+idle 回收 / C=显式快照 / D=长驻 workspace；D4 共享路径 ①=打包搬运 / ②=NAS 直挂 / ③=FUSE+对象存储 / ④=JuiceFS 元数据分离

1. **隔离用 gVisor 起步，敏感企业版上 Firecracker microVM**——同一套控制面，按 SKU 切。
2. **生命周期：B(会话+idle 回收) 模式为主**——会话级 + idle 5min 回收，但启动用 warm pool 把首次延迟压到 < 1s。
3. **文件层分三类挂载**：
   - **uploads / outputs**：路径 **③(FUSE over OSS)**，cache 设短，所有挂载点只读或可写明确分离
   - **workspace**：路径 **②(NAS)** 或 **④(JuiceFS)**，给那些"需要在沙箱里跑 npm install / git"的高频写场景
   - **transcripts**：路径 **③(FUSE+对象存储)** readonly + 长 cache，沙箱不可写入，单独走审计后端
4. **退出兜底**：销毁前 flush FUSE + 同步 NFS + tar 冷数据归档 OSS（路径 **①(打包搬运)** 作为最后的备份）。
5. **复活恢复**：下次重启时 mount 即可见，同时异步从 OSS 还原冷数据到 NFS（lazy load）。
6. **多租户**：每个 conversation 一个 namespace，STS Token 限制 prefix；沙箱启动时 inject 临时凭证，TTL 30 分钟自动续约。
7. **可观测性**：每个挂载点的 cache hit/miss、上传字节、list 次数都打到独立 metric；超阈值告警。

### 与你提到的三种方案的关系

- **你的方案 1 = 路径 ①(OSS 打包搬运)**：合并到本架构的"workspace 冷归档"角色，作为崩溃恢复兜底
- **你的方案 2 = 路径 ②(NAS 直挂)**：作为 `workspace` 挂载点的主力（高一致性、多 worker 协作场景）
- **Anthropic 方案 3 = 路径 ③(FUSE+对象存储)**：作为 `uploads/outputs/transcripts` 这些"对外可见、按会话隔离"挂载点的主力

也就是说——**生产级方案不是三选一，而是按文件用途分三层挂载**。这才是把"成本/一致性/弹性"这个三角形优化到位的真实工程做法。

---

## 九、核心结论

1. **"沙箱+文件共享"不是单一选型，而是 5 个独立维度的组合**：隔离技术、生命周期、持久化层、共享机制、命名空间。
2. **6 种文件共享路径**（打包搬运 / NAS 直挂 / FUSE+对象存储 / 元数据分离 FS / Volume API / MicroVM 快照）各有适用场景，没有银弹。
3. **生产级架构通常是混合**——按文件用途分三层挂载（用户输入只读 / AI 产物可写 / 协作工作区），每一层独立选型。
4. **Anthropic 的 rclone+FUSE 方案在"海量轻量会话+成本敏感"赛道是最优解**；阿里云 NAS 方案在"企业内部强一致+多协作"赛道更合适；E2B/Modal 的快照方案在"需要保留 REPL 状态"场景独占。
5. **idle 5min 销毁的真正难点不在"销毁"，而在"复活时让用户感觉沙箱一直都在"**——这要求共享层在销毁时已经持久化、复活时秒级可见。
6. **多租户和审计往往是系统真正暴雷的地方**——挂载点权限、STS Token、transcripts readonly、prefix HMAC 这四件事任何一件做错都是大事故。

---

## 参考资料

- [Anthropic Code Execution Tool 文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)
- [Anthropic Claude Code Sandboxing 博客](https://www.anthropic.com/engineering/claude-code-sandboxing)
- GitHub Issue [anthropics/claude-code#55877](https://github.com/anthropics/claude-code/issues/55877)
- [E2B Sandbox Persistence](https://e2b.dev/docs/sandbox/persistence)
- [Modal Volumes](https://modal.com/docs/guide/volumes)
- [Daytona Architecture](https://www.daytona.io/)
- [Cloudflare Sandbox SDK](https://developers.cloudflare.com/sandbox/llms-full.txt)
- [阿里云 函数计算 AI Sandbox NAS 挂载](https://help.aliyun.com/zh/functioncompute/fc/sandbox-supports-instance-level-dynamic-attachment-of-nas-test-invitation)
- [阿里云 函数计算 AI Sandbox OSS 挂载](https://help.aliyun.com/zh/functioncompute/fc/sandbox-supports-instance-level-dynamic-mount-of-oss-test-invitation)
- [JuiceFS vs S3FS 官方对比](https://juicefs.com/docs/community/comparison/juicefs_vs_s3fs/)
- [AWS Mountpoint for S3 设计博客](https://aws.amazon.com/blogs/storage/the-inside-story-on-mountpoint-for-amazon-s3-a-high-performance-open-source-file-client/)
- [Best Cloud Sandboxes for AI Agents 2026 (Blaxel)](https://blaxel.ai/blog/best-cloud-sandboxes-ai-agents-2026)
- [Daytona vs E2B AI Sandbox 对比](https://northflank.com/blog/daytona-vs-e2b-ai-code-execution-sandboxes)

