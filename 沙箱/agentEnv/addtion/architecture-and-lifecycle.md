# AgentENV 架构、组件交互与生命周期

> 组件与分层、三条交互主线、冷启动/快照/暂停/恢复四流程、通信通道、依赖关系。
> 代码：`src/api/`、`src/orchestrator/`、`src/sandbox/`、`storage/`、`src/snapshot/`。

## 1. 组件与分层

| 组件 | 层 | 角色 |
|---|---|---|
| HTTP API | 控制面 | 对外接口、鉴权、路由 |
| Orchestrator | 控制面 | 沙盒生命周期总管（每节点一个） |
| 快照仓库 | 控制面 | 快照发布/解析/删除 |
| Firecracker | 数据面 | 微VM，承载 guest；HTTP API 供宿主控制 |
| envd | 数据面（guest 内） | 命令执行、文件操作、健康上报 |
| ublk daemon | 数据面 | 管理所有 ublk 设备，内嵌 overlaybd，处理块 I/O |
| ublk（库+内核框架） | 数据面 | 用户态块设备 `/dev/ublkbN` |
| overlaybd | 数据面 | 分层镜像、按块懒加载、内存层、缓存 |
| OCI 仓库 | 外部 | 镜像来源 |
| OSS（S3 兼容） | 外部 | 快照/对象存储 |

```
控制面: 客户端 → HTTP API → Orchestrator → {元数据存储, 快照仓库→OSS, 沙盒工厂}
数据面: FirecrackerSandbox
          ├─Unix socket→ Firecracker --virtio-blk(/dev/ublkbN)--> 内核ublk ⇄ ublk daemon ⇄ overlaybd ⇄ OCI/OSS/缓存
          └─TCP→ envd(guest:49983)
```

## 2. 三条交互主线

- **控制**：客户端 → API → Orchestrator → 元数据/快照仓库/沙盒工厂（进程内）。
- **数据**：VM 读块 → Firecracker virtio-blk → 内核 ublk → io_uring → ublk daemon → overlaybd（本地缓存命中或按 HTTP Range 回源 OCI/OSS，zstd 解压）→ 回写。
- **guest**：宿主 → TCP `guest_ip:49983` → envd（HTTP 做健康/初始化/指标/日志；gRPC 做进程执行与文件操作）。

## 3. 生命周期四流程

### 3.1 冷启动
```
POST /sandboxes → API → Orchestrator.create
  → 解析镜像/模板 → launch_sandbox：
    ① 造 FirecrackerSandbox
    ② 保护镜像引用（防 GC）
    ③ 启动：tools 盘(只读 ublk) + 用户 rootfs(overlaybd via ublk 可写) + 额外盘
             + 分配网络槽位(netns/tap/IP) + 组装 boot args(init=/init, ip=)
             + 在 netns 内启动 Firecracker
             + FC API 配置(machine/boot/drives/network/mmds/balloon) → InstanceStart
    ④ 记录过渡态 Creating
    ⑤ 等 envd 就绪（TCP 49983）
    ⑥ → Running，发布代理路由
```

### 3.2 快照创建（原创继续跑）
```
FirecrackerSandbox.snapshot：
  内存：FC 返回脏页范围 → process_vm_readv 读取 → overlaybd 生成内存层（父层堆叠）
  文件系统：ublk daemon 封存(seal)可写层 → 新只读层
  → 快照仓库 publish（OSS/POSIX）
```

### 3.3 暂停
```
POST .../pause → Orchestrator.pause
  ① 保护运行时产物 → ② 分配 paused 持久化目录
  ③ 捕获（同 3.2）→ paused 状态 → Firecracker 停止（不占 CPU/内存）
  ④ 持久化 paused 状态 → ⑤ Paused
```
> **Snapshot vs Pause**：Snapshot 发布可复用快照、原创继续跑；Pause 捕获并**停 VM**、转 Paused、可原地恢复。

### 3.4 恢复
```
POST .../resume → Orchestrator.resume（过渡态 Resuming）→ launch_sandbox：
  ① 从 paused 状态造 FirecrackerSandbox
  ② 恢复：检查 ublk daemon → 优先接管 warm pool(进程+网络槽位) → 恢复 rootfs(overlaybd)
           → 加载快照（内存后端=ublk 内存设备, mmap+COW）→ 重启 envd
  ③ 等 envd 就绪 → Running → ④ 删除 paused 记录
```

## 4. 通信通道

- **控制 Firecracker**：Unix domain socket（`--api-sock`），发 HTTP API（machine/boot/drives/network/mmds/balloon/snapshot/dirty-memory-ranges）。
- **访问 envd**：TCP `guest_ip:49983`（经 tap）——
  - HTTP：健康检查、初始化（access token/env/workdir/user）、指标、日志；
  - gRPC：进程执行（流式：启动事件→stdout/stderr 块→退出码；stdin/信号另发）、文件操作；
  - 鉴权：access token；执行时用 `authorization: Basic <base64(用户名:)>` 选账户。
- **vsock 澄清**：envd **不用 vsock**；vsock 仅用于 userfaultfd 内存恢复等。

## 5. 依赖关系

| 调用方 | 被调方 | 通道 |
|---|---|---|
| 客户端 | HTTP API | HTTP |
| HTTP API / Orchestrator / 沙盒工厂 | — | 进程内 |
| Orchestrator | 快照仓库 | 进程内 |
| 快照仓库 | OSS / POSIX | 对象存储 / 文件系统 |
| FirecrackerSandbox | Firecracker | Unix socket |
| FirecrackerSandbox | ublk daemon | Unix socket RPC |
| ublk daemon | ublk(库)+内核 | io_uring + `/dev/ublk-control` |
| ublk daemon | overlaybd | 进程内（内嵌） |
| overlaybd | OCI 仓库 / OSS / 本地缓存 / P2P | HTTP Range / S3 / 磁盘 / iroh |
| 内核 ublk | Firecracker | virtio-blk `/dev/ublkbN` |
| FirecrackerSandbox | envd | TCP（HTTP+gRPC） |

## 6. 结论

Orchestrator 在控制面"指挥"；FirecrackerSandbox 把指挥翻译成对 Firecracker、ublk daemon、envd 的操作；块数据由 ublk daemon + overlaybd 从 OCI/OSS/缓存搬运；快照经 Firecracker 脏页 + overlaybd 内存层 + ublk daemon 封存，落到快照仓库（OSS/POSIX）。
