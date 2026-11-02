# AgentENV 技术介绍 — PPT 内容大纲（v3 · 正文 10 页）

> 目标：听众不熟悉 AgentENV，讲者照着页面就能讲。每页都要有：结论、机制细节、关键数字、流程图（箭头表示流转方向）。
> 本版约束：**封面不计入正文，正文 10 页（P02–P11）**；快照生成与快照恢复是核心；性能与调优页由讲者后续补充，当前定稿只到 P11。
> 排版由 PPT 执行，本文件只负责内容与结构。

## 全局约定

- **数据口径**（Kimi K3 技术报告 §5.3.2）：checkpoint / resume **最低** 133 ms / 49 ms（as low as）、**大规模启动亚秒级（sub-second launch latency at large scale）**、数万沙盒需在数秒内创建、内存超卖最高 6.5×（写时复制内存 + page cache）、累计 51,219,741 个 Sandbox / 1,505,678 个镜像、单 Sandbox 生命周期最高 98% 在等推理。原文还说明 K3 同时使用容器 / GPU / AgentENV 三种运行时，AgentENV 是其中的 microVM 运行时。
- **事实**（以 `markdown/` 产品文档为准，K3 报告只提供性能口径）：8 态生命周期为 Creating / Running / Pausing / Paused / Resuming / Snapshotting / Forking / Killing；Pausing→Paused 可原地 Resume，Snapshotting / Forking 完成后回到 Running，Killing 为终止态。TTL：服务端 keep-alive 默认 15 s，CLI `--timeout` 默认 300 s，`resume` 从恢复时刻重算，`refresh` 续期。`aenv snapshot` 仅 create/list；删除模板用 `aenv template delete`。
- **默认规格**：`[machine]` 默认 2 vCPU / 1024 MiB（可配）；引用 2C4G 等其他规格时必须注明来源。
- **数据面走向**：Gateway 是 HTTP 反向代理，沙盒数据面经它转发到节点；Scheduler 只管选路与绑定，不参与数据转发。
- **warm 池**：`[pool.network]` 网络槽位（低 2 / 高 64）用于新建与恢复；`[pool.firecracker]` 预生成 Firecracker 进程只服务 Snapshot 恢复。
- **字号**：正文 20px、标题 36px、副标题 26px、注释 16px、脚注 14px、大数字 56px。
- **流程图规范**：卡片统一 **328×96**、3 列（x=80/476/872）、行高 180/320/460，行内蛇形、换行竖直箭头；末行可不铺满（8 步=3+3+2、7 步=3+3+1、6 步=3+3）；末步深色强调；页底一行注释。

## 页面结构（封面 + 正文 7 页 = 8 页）

| 页 | 内容 | 定位 |
|---|---|---|
| 1 | 封面 + 三个主数字 | 一句话定位 |
| 2 | 背景：K3 的沙盒需求 | Why |
| 3 | 系统架构（节点树） | What |
| 4 | 概念：三个抽象、8 态与存储 | What |
| 5 | overlaybd：镜像按块取 | How |
| 6 | 冷启动（9 步） | How |
| 7 | **快照：生成与恢复（8 步 + 7 步，一页两段）** | 核心 |
| 8 | 暂停：释放算力，共享换密度 | 补充 |

> 性能数据与调优：讲者后续补充（当前定稿不含性能页）。

---

## P01 封面

- 标题：**毫秒级启停的 Agent 沙盒运行时**
- 副标题：AgentENV · 为 Kimi K3 的 agentic RL 提供执行环境
- 三个数字：**133 ms** 检查点 ｜ **49 ms** 恢复 ｜ **5,121 万** 次 Sandbox 创建
- 一句定位：microVM + 分层镜像 + 原生快照
- 备注：当前封面只放了标题与副标题，三个数字待排版加入。

## P02 背景：K3 的沙盒需求

**结论**：Kimi K3 的 agentic RL 规模，靠 AgentENV 才跑得起来。
**页面顺序**：页标题「背景：K3 的沙盒需求」+ 小标题「四个硬约束」+ 四行（隔离 / 成本 / 速度 / 密度：需求 → 因此必须）→ 小标题「实测性能」+ 数据表（检查点 133 ms / 恢复 49 ms / 大规模启动 <1 s / 内存超卖 6.5×，说明列均为原文）→ 规模注释（页脚上方）。
**原文对应（按原文压缩成一行，不加推论）**：
- 隔离：Agent 会越界探索、甚至 reward hacking，还要能挂盘 / 跑容器 / 起 VM → 必须独立内核。
- 成本：等待推理的时间可占沙盒生命周期 98% → 必须能暂停，真正释放内存与 CPU。
- 速度：reward judging 要无副作用分叉、错误恢复要定期快照 → 保存与恢复必须足够快。
- 密度：数万个各自独立镜像的沙盒需在数秒内创建 → 镜像按需加载，内存可共享 / 超卖。
（以上四条是"需求 → 因此必须"，133 ms / 49 ms / <1 s / 6.5× 等结果数字只放在下方性能表。）
- **规模**：训练与评估期间累计 51,219,741 个 Sandbox、覆盖 1,505,678 个镜像；数万沙盒需秒级创建。
- **隔离**：Agent 会激进探索甚至越权（挂盘、跑容器、自起 VM）——共享内核容器做不到，必须独立内核 microVM。
- **成本**：单个 Sandbox 生命周期最高 98% 在等推理；必须能暂停（不占 CPU/内存）再恢复；默认 2 vCPU / 1 GiB 小规格 + 6.5× 内存超卖。
- **速度 / 密度**：百万 token 长轨迹要求随取随用（最低 133 ms / 49 ms）；数万沙盒秒级创建、大规模下亚秒级启动；150 万镜像不可能全放单机，只能分层按需加载 + 有界缓存。
- 结论条：**没有 AgentENV，Kimi K3 的 agentic RL 无法在成本与吞吐上成立。**

## P03 系统架构（照文档 mermaid 画节点树）

**结论**：节点内 API → Orchestrator → Firecracker VM；VM 下三条存储路径，每条四段（设备 → ublk → overlaybd → 末端节点）。
- **容器**：AgentENV Node（:8000，每节点一个 runtime 进程）。
- **主干**：API（Axum）→ Orchestrator（生命周期状态机）→ Firecracker VM（独立内核 · guest envd）。
- **rootfs**：`/dev/vda` → `ublk（/dev/ublkbN）` → `overlaybd` → `upper（r/w）` + `layer 0..N（r/o）`。
- **extra**：`/dev/vdb` → `ublk device` → `overlaybd（extra drive）`。
- **memory**：`VM memory` → `ublk（/dev/ublkbM，只读、引用计数共享）` → `overlaybd（mem layers）` → `snap N（r/o）` + `snap 0/1/2…（r/o）`。
- **数据来源**：块按需回源 OCI / OSS / 本地缓存 / P2P；ublk 把 overlaybd 镜像变成 `/dev/ublkbN`。
- **控制面**（Gateway / Scheduler）不在本图，见附录 A3。

## P04 概念：三个抽象、8 态与存储（独立概念页）

**版式**：与 P02「四个硬约束」一致——小标题 + 逐行（左标签、右说明、隔行浅灰底）。

**小标题「三个抽象」**
- Template 模板：静态、可复用的起点：OCI / Dockerfile 构建，产物本身就是一份已提交快照。
- Sandbox 沙盒：运行中的实例：独立内核、文件系统、网络栈；独占 CPU / 内存，有 8 态生命周期。
- Snapshot 快照：状态副本：内存增量层 + 封存磁盘层 + 上下文；可派生新沙盒、不替换源。

**小标题「8 态」**
- Orchestrator：Creating / Running / Pausing / Paused / Resuming / Snapshotting / Forking / Killing。

**小标题「ublk + overlaybd」**
- ublk：用户态块设备，把 overlaybd 镜像暴露为 `/dev/ublkbN`。
- overlaybd：LSMT 分层，按块拉取（OCI / OSS / 缓存 / P2P）；upper 可写 + layer 只读共享。
- 三类设备：rootfs / extra 独占；memory 只读共享（引用计数 + COW）。

## P05 overlaybd：镜像按块取，磁盘只做缓存

**结论**：读路径自上而下、写路径只追加；压缩让块可寻址；只读层共享 + 有界缓存撑起密度。
- **读路径**：VM 读块 → 自上而下查各层索引，命中即返（未映射向下穿透）→ HTTP Range 只取该段 → zstd 局部解压 → 写缓存。
- **写路径**：写入只追加到 upper，索引在内存更新、sync 时刷盘；暂停时 seal 成只读层，原地开新 upper。
- **压缩**：OCI 的 tar.gz 要整包解开才能读到中间；overlaybd 用 zstd 分段压缩，并用 LSMT 索引记录每段位置，读哪段只解哪段。
- **共享**：只读层不可变、可被多个 Sandbox 共用；本地只缓存热块（LRU 95% / 70% / 600 s），冷块回源 OCI / OSS / P2P。
- **分层图**：上层可写 upper（只追加，暂停时 seal 成只读层）→ 只读层 N … 0（不可变、可共享）。

## P06 冷启动：先存储，后网络，再启动 VM

**结论**：存储与网络必须先于 VM 启动完成；envd 就绪前状态停在 Creating。
1. 客户端请求：`aenv start --cold` / `POST /sandboxes-cold`；多节点先由 Gateway 选节点。
2. API 鉴权与校验：校验 API Key → Orchestrator；状态置 Creating。
3. 沙盒工厂：创建 FirecrackerSandbox；保护镜像引用，防止 GC 误删在途镜像。
4. 存储（先）：ublk daemon 建 rootfs 设备；可写 upper → `/dev/ublkbN` 交给 VM。
5. 网络（后）：分配 netns / tap / IP / 防火墙与出站策略；优先复用 warm 网络槽位。
6. VM 启动：spawn Firecracker，配 machine / boot drives / network → InstanceStart。
7. guest 启动：内核启动，init 拉起 envd；runsv 托管，崩溃后自动重启（TCP 49983）。
8. envd 就绪：健康检查通过；HTTP init（token / env）+ gRPC 进程与文件。
9. 沙盒可用 Running：发布代理路由；CLI 附加 shell，应用经 `/proxy` 或代理域名访问。
- 关键点：首次读盘才触发镜像按需加载（见 P05）。

## P07 快照：生成与恢复（一页两段）

**结论**：生成写三层产物且源不中断；恢复按引用计数共享、COW 换入、warm 池移交。
**版式**：同一页上下两段，均为 256×96 卡片、4 列；上段「快照生成（8 步 · 源不中断）」，下段「快照恢复（7 步）」；末步深色；页底一行对照。

**上段 · 快照生成（8 步）**
1. 触发：`aenv snapshot create <sandbox>`（可 `--name`）；源 Sandbox 全程不中断。
2. Firecracker 状态快照：开 track dirty pages，生成仅含状态的 diff，不含磁盘内容。
3. 读取脏页：查 dirty / present 范围，用 `process_vm_readv` 读出选中页。
4. 写内存层：生成 overlaybd 内存层，与父快照层堆叠成完整镜像，只含本次增量。
5. 封存磁盘：ublk daemon seal 可写 upper → 最新只读 lower；原地打开新 upper。
6. 记录上下文：卷大小 / 路径 / 挂载模式；CPU / 内存、环境变量、启动命令。
7. 提交仓库：SnapshotManager 内容寻址提交到 OSS（S3 兼容）/ POSIX。
8. P2P 通告：其他节点可发现；后续启动优先 P2P 拉取镜像层，减少回源。

**下段 · 快照恢复（7 步）**
1. 触发恢复：`POST resume` 或 `aenv start`；Orchestrator 置 Resuming 并校验。
2. 解析产物：先 P2P，未命中再 OSS / POSIX；得到 FC 状态 + overlaybd 内存层与磁盘层。
3. 恢复设备：建只读内存 ublk 设备（引用计数共享）；恢复 rootfs overlaybd 设备并挂载。
4. 移交或新建：warm 池移交网络槽位 + FC 进程；否则新分配槽位、spawn Firecracker。
5. 加载机器状态：SnapshotLoadParams；内存后端 = 只读 ublk，按需换入脏页。
6. mmap + COW：Firecracker mmap 内存设备；首次写入才复制到匿名内存，底层永不修改。
7. Running：envd 重启就绪；TTL 从恢复时刻重算，删除 paused 记录。

**页底对照**：生成 = 内存增量层 + 封存磁盘层 + 上下文；恢复 = 引用计数共享 + COW 换入 + warm 池移交。

## P08 暂停：释放算力，共享换密度

**结论**：暂停走同一条捕获链路，但停在本地、可原地恢复；省下来的算力与内存靠共享换密度。
- 暂停 6 步：触发 → Pausing（保护运行时产物、分配 paused 目录）→ 捕获内存 → 封存磁盘 → 停止 VM → Paused（持久化，TTL 到期 autoPause 同链路）。
- **暂停 ≠ 快照**：暂停停止 VM、状态留在本节点、可原地 Resume；快照提交共享仓库、可分发派生、源继续运行。
- **密度四机制**：只读层共享（镜像只存一份）/ 内存设备引用计数 + page cache 跨沙盒复用 / ballooning（6.5× 超卖）/ Fork（exclusive 卷按需 COW）。
- TTL：服务端 15 s、CLI `--timeout` 300 s；`resume` 重算、`refresh` 续期。

## 附录（不进正文，作为备份页/演讲备注）

### A1 组件清单
- 控制面：HTTP API（Axum，鉴权 / 校验 / 反代路由）、Orchestrator（8 态、TTL 驱逐、重启恢复）、元数据 / paused 存储、快照仓库（SnapshotManager，OSS / POSIX）、模板构建器（镜像 → 模板 → 发布快照）。
- 数据面：FirecrackerSandbox（节点控制器）、Firecracker（microVM）、NetworkManager / 网络槽位（netns、tap、IP、iptables、egress、warm 池）、ublk daemon（`uvm-ublk-daemon`）、ublk（内核框架 + 用户态库）、overlaybd（分层 / 按块 / 内存层 / 缓存）、envd（guest 内，TCP 49983）。
- 外部：OCI 仓库（HTTP Range）、OSS / POSIX、P2P（可选，iroh-blobs）。

### A2 调用通道表
| 操作 | 调用方 → 被调方 | 通道 |
|---|---|---|
| 解析模板 / 快照 | Orchestrator → 快照仓库（进程内）+ OSS / POSIX / P2P | 进程内 + 网络 |
| 记录状态 / TTL | Orchestrator → 元数据存储 | 进程内 / 本地文件 |
| 创建 rootfs 设备 | FirecrackerSandbox → ublk daemon | Unix socket RPC |
| 配机器 / 启动 VM | FirecrackerSandbox → Firecracker | Unix socket（HTTP API） |
| 分配网络槽位 | FirecrackerSandbox → NetworkManager | 进程内 |
| 初始化 / 执行命令 | 宿主 → envd | TCP 49983（HTTP + gRPC） |
| 读取镜像块 | ublk daemon → overlaybd → OCI / OSS | HTTP Range |
| 封存 upper / 提交快照 | Sandbox → ublk daemon；快照仓库 → OSS / POSIX | Unix socket；S3 API / POSIX |
| P2P 拉层 | 节点 ⇄ 节点 | iroh-blobs（可选） |

### A3 控制面与部署（已并入正文 P10，细节保留）
- Gateway（:8080）：从 `x-agentenv-sandbox-id` / `e2b-sandbox-id` 或 `{port}-{sandboxID}.{domain}` 提取路由；新 Sandbox 调 `Schedule()`、已有流量 `LookupNode()`、创建成功 `RecordAssignment()`；无路由头时聚合 `/sandboxes`、`/nodes`。
- Scheduler（:9090，gRPC）：`Schedule` / `LookupNode` / `RecordAssignment` / `Heartbeat` / `ListObservedNodes` / `ListP2pPeers` / `GetNode` / `UnregisterNode`；内存绑定 + 可插拔发现（static / K8s EndpointSlice）；绑定 TTL 只代表新鲜度，心跳名册为准；round_robin 默认。
- 数据面代理：`/proxy`、路由头、代理域名；HTTP / SSE / WebSocket 转发；域名要求 Sandbox ID 兼容 RFC 952/1123、标签 ≤63 字符。
- P2P（可选）：iroh-blobs 内容寻址；lookup / fetch / publish；快照固定产物与 overlaybd 层"先 P2P 后对象存储"；取到即通告；未启用零开销。
- 部署：单机 systemd / Docker；Compose 多节点模拟；K8s 特权 DaemonSet（每主机一个 runtime Pod，需 `/dev/kvm`、iptables、hostPath 缓存，ConfigMap 单一来源）；`AENV_API_KEY` 由 runtime 与 gateway 共享。

### A4 安全与隔离（已并入正文 P11，细节保留）
- 认证：API Key（`AENV_API_KEY`）由 runtime 与 gateway 共享；控制面请求鉴权。
- 代理：`/proxy` 与代理域名按 Sandbox ID 路由；DNS 标签约束（RFC 952/1123、≤63 字符）。
- 网络：每 Sandbox 独立 netns + tap + IP；防火墙顺序、egress 代理、策略替换（见 `internals/networking.md`）。
- 存储：rootfs / extra 独占设备；快照层不可变、可共享；卷按访问模式（ro / exclusive）处理。

### A5 模板构建细节
- `aenv pull <image> [--name] [--cpu] [--memory] [--start-cmd] [--ready-cmd] [--probe]`：导入 OCI 镜像为 Template。
- `aenv build . -f Dockerfile --name <name>`：BuildKit 在临时 microVM 中构建 → 转 OverlayBD → 捕获 Template；`--detach` 后台构建，`aenv template watch` 跟踪。
- `--start-cmd` / `--ready-cmd` / `--probe <port>` 控制快照捕获时机；删除用 `aenv template delete`。

### A6 可观测性
- 节点快照：`GET /nodes`、`GET /nodes/{id}`；主机 CPU / 内存 / 磁盘（`/proc/stat` 两次采样）。
- Orchestrator 增量计数器（运行中 / 启动中、分配 CPU / 内存、创建成功 / 失败）。
- Scheduler 心跳上报（gRPC `Heartbeat`，关闭时 `UnregisterNode`）；`ListObservedNodes`。
- TTL / autoPause 与驱逐策略；warm 池水位（low 2 / high 64）。
