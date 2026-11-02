# AgentENV 存储子系统：overlaybd 与 ublk

> overlaybd（分层镜像、按块懒加载、缓存）与 ublk / ublk daemon（用户态块设备）。
> 代码：`storage/overlaybd/`、`storage/ublk/`、`storage/ublk-daemon/`。

## 1. 全貌

```
OCI 镜像(tar.gz) ──转换──► overlaybd 层(索引 + zstd 可寻址块)
                              │
                        ublk daemon（内嵌 overlaybd，处理 I/O）
                              │ io_uring + /dev/ublk-control
                        内核 ublk → /dev/ublkbN ──virtio-blk──► Firecracker VM
                              │
                     OCI 仓库 / OSS / 本地缓存 / P2P（数据来源）
```

## 2. overlaybd

### 2.1 问题与转换
原始 OCI 层是 **tar.gz**（顺序流 + 顺序压缩），无法按块随机取。overlaybd 把它**转换成块可寻址格式**（`convert_standard_oci`，默认开启）：zstd 压缩 + jump table + LSMT 段映射索引。

- **zstd**：快且可调的压缩格式；配合 **jump table**（每固定大小设跳转入口）可只解压目标段，不像 gzip 必须从头解。
- **LSMT**：overlaybd 的分层格式——只追加写（底层只读不变）、多只读层 + 一个可写层堆叠、每层带段映射索引。**封存(seal)** 即可写层冻结为只读层，是暂停/快照的核心动作。

### 2.2 按块拉取的三支柱
| 支柱 | 作用 |
|---|---|
| 段映射索引 | 定位块在层文件中的字节偏移 |
| zstd + jump table | 只解压目标块所在压缩段 |
| HTTP Range | 只从 registry 拉需要的字节范围 |

**块可寻址由 overlaybd 格式提供，不是 registry/OSS 原生能力**——它们只是"支持按字节 Range 读取的 blob 存储"。

### 2.3 数据来源与缓存
| 远端 | 存什么 |
|---|---|
| OCI registry | 原始/转换后镜像层（主要来源；不用 Turbo-OCI referrers） |
| OSS / POSIX 共享存储 | 快照持久化产物 |
| 集群其它节点（P2P） | 已缓存层/转换产物，仅加速 |

本地是**有界缓存**（容量上限 + LRU 淘汰，配置在 `[image.cache]`、`[image.cache.gc]`、`[image.cache.remote_blocks]`）：只存热块，冷块淘汰后按需回源，因此聚合镜像/快照总量可远超本地磁盘。

## 3. ublk 与 ublk daemon

### 3.1 内核模块 `ublk_drv.ko`
提供 `/dev/ublk-control`；为每个设备创建 `/dev/ublkcN`（控制）+ `/dev/ublkbN`（块设备）；把 VM 的块 I/O 经 io_uring 队列投给用户态。内核不含存储逻辑，只做转发。

### 3.2 ublk daemon（uvm-ublk-daemon）
进程级单例，职责：
1. **设备生命周期**：创建/删除/快照/池化（rootfs、extra、memory snapshot）；
2. **overlaybd 前台**：每个设备背后是 overlaybd 镜像，worker 处理 I/O（查索引→缓存/远端→解压）；
3. **进程隔离**：所有设备与 io_uring 集中在 daemon，节点经 Unix socket 下命令。

> **归属**：daemon 与 `storage/ublk` 是 **AgentENV 自定义**；底层依赖开源 **Linux ublk 内核框架** + **libublk-rs**（"怎么和内核通信"由内核+开源库提供，"数据是什么、设备怎么管"由 AgentENV 定义）。

### 3.3 三类设备
| 设备 | 用途 | 访问 |
|---|---|---|
| rootfs（`/dev/vda`） | 根文件系统 | Exclusive |
| extra drive（`/dev/vdb`） | 附加数据盘 | Exclusive |
| memory snapshot | 快照恢复的内存后端 | Shared（引用计数） |

### 3.4 与 Firecracker 协作
启动建 rootfs 设备 → `/dev/ublkbN` 当盘挂给 VM；读块走 daemon→overlaybd；暂停时 daemon 封存可写层；恢复时建只读内存设备供 Firecracker mmap+COW。

### 3.5 为什么放用户态
逻辑复杂且演进快、用户态崩溃不拖垮内核、ublk+io_uring 性能接近内核态、共享/快照策略易实现。

## 4. 关系与代码位置

**ublk daemon 内嵌 overlaybd**：daemon 管"块设备框架与管理"，overlaybd 管"块里是什么数据、怎么取"。

| 主题 | 位置 |
|---|---|
| overlaybd（格式/压缩/快照/后端） | `storage/overlaybd/src/`（`lsmt/`、`image/`、`compression/zfile.rs`、`backend/`、`tools/oci.rs`） |
| ublk 库 | `storage/ublk/src/`（`ctrl.rs`、`dev.rs`、`queue.rs`、`impls/overlaybd_target.rs`） |
| ublk daemon | `storage/ublk-daemon/src/`（`client.rs`、`server.rs`、`protocol.rs`） |
| 节点侧设备管理 | `src/sandbox/ublk/device.rs`、`src/sandbox/ublk/overlaybd.rs` |
