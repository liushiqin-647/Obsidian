# ublk 内核模块与 ublk daemon 在 AgentENV 中的作用

> 本文梳理 AgentENV 中 ublk 存储链路的两个核心角色：内核模块 `ublk_drv.ko` 与用户态守护进程
> `uvm-ublk-daemon`，以及它们与 Firecracker 微虚拟机的协作。
> 官方文档中相关内容分散在 `internals/architecture.md`、`getting-started/quickstart.md`、
> `troubleshooting/common-issues.md`、`configuration/reference.md`，缺少一条完整链路，故单独整理。

## 1. 两个角色定位

| | `ublk_drv.ko`（内核模块） | `uvm-ublk-daemon`（用户态进程） |
|---|---|---|
| 身份 | Linux ublk 框架，提供设备接口和 I/O 转发 | AgentENV 的**核心存储服务进程**（进程级单例） |
| 干的事 | 建设备节点、把块 I/O 转给用户态 | 实现真正的读写逻辑（overlaybd）、管理设备生命周期 |
| 来源 | Linux 内核（安装时 `modprobe ublk_drv`） | `storage/ublk/src/`（库）+ `storage/ublk-daemon/src/`（守护进程） |

## 2. 内核模块 `ublk_drv.ko` 的作用

1. 提供 **`/dev/ublk-control`** 控制入口；
2. 为每个设备创建：
   - **`/dev/ublkcN`**：用户态控制节点；
   - **`/dev/ublkbN`**：**标准块设备**（对 Firecracker/VM 来说就是一块"硬盘"）；
3. 把 VM 对 `ublkbN` 的块 I/O，经 **io_uring 队列**投递给用户态 daemon；
4. 内核里**不含任何存储逻辑**——它只是一个"桥/转发层"。

代码：`storage/ublk/src/ctrl.rs` 中 `CTRL_PATH = "/dev/ublk-control"`、`UringCmd80`，用 io_uring 发送 `ADD` / `DEL`。

## 3. `uvm-ublk-daemon` 的作用

它是**所有 ublk 设备的唯一管理者**，同时是 **overlaybd 的"前台"**。职责分四层：

### 3.1 设备生命周期管理

RPC 定义在 `storage/ublk-daemon/src/protocol.rs`（`enum DaemonRequest`）：

| 请求 | 作用 |
|---|---|
| `CreateOverlaybdRuntimeDevice` | 为 Sandbox 创建 rootfs / 数据盘设备 |
| `CreateOverlaybd` | 从已物化镜像创建原始 overlaybd 设备 |
| `Delete` | 销毁设备 |
| `RestackSnapshot` | Pause 时把可写层封存成新快照层 |
| `AcquireOverlaybd` / `ReleaseOverlaybd` | warm 设备池的取 / 还 |
| `QueryCapabilities` | 查询（如动态 resize）能力 |
| `ReleaseSharedMemory` | 上报内存快照设备使用结束（引用计数） |

### 3.2 提供三类块设备（`enum AccessMode`）

| 设备 | 用途 | 访问模式 |
|---|---|---|
| **rootfs**（→ VM 的 `/dev/vda`） | Sandbox 根文件系统（overlaybd 分层） | Exclusive |
| **extra drive**（→ `/dev/vdb`） | 用户附加数据盘 | Exclusive |
| **memory snapshot** | 快照恢复时给 Firecracker 当内存后端 | **Shared（引用计数共享）** |

### 3.3 I/O 服务（overlaybd 前台）

每个 ublk 设备背后是一个 overlaybd `ImageFile`。daemon 的 io_uring worker 线程处理块 I/O：

- 查 LSMT 索引 → 命中本地缓存直接读；否则按 **HTTP Range** 从远端拉块 → **zstd 局部解压** → 返回并缓存。

这就是"按块懒加载"在运行时真正执行的地方。

### 3.4 进程隔离

所有 ublk 设备所有权与 io_uring 控制都集中在 daemon 一个进程；节点（orchestrator）只通过 **Unix socket**（`UblkDaemonClient` → `UblkDeviceManager`）下发命令，不直接碰 io_uring。

## 4. 与 Firecracker 的协作场景

### 场景 A：启动 Sandbox（rootfs 变成 VM 硬盘）

```
orchestrator
  → UblkDeviceManager → daemon: CreateOverlaybdRuntimeDevice
  → daemon 向 /dev/ublk-control 发 ADD（io_uring）
  → 内核创建 /dev/ublkcN + /dev/ublkbN
  → Firecracker 把 /dev/ublkbN 当 /dev/vda 挂给 VM
  → VM 启动，开始读 rootfs
```

### 场景 B：VM 读一个块（I/O 全链路）

```
VM 读 /dev/vda
  → 内核 ublk_drv 捕获块 I/O
  → 经 io_uring 队列投给 daemon
  → daemon worker：overlaybd 查索引 → 本地缓存 / 远端 Range 拉块 → 解压
  → 写回 io_uring，内核完成本次 I/O → VM 拿到数据
```

### 场景 C：Pause（封存成快照）

```
orchestrator → daemon: RestackSnapshot
  → daemon 封存(seal) 当前可写层为只读层 → 接到层链 → 开新可写层
```

### 场景 D：内存快照恢复（多 Sandbox 共享）

```
daemon 创建只读 ublk 内存设备（Shared，引用计数）
  → Firecracker mmap 它作为 memory backend（COW）
  → 同一快照启动的多个 Sandbox 共享同一设备 → page cache 复用
```

## 5. 为什么把存储逻辑放用户态

| 原因 | 说明 |
|---|---|
| **逻辑复杂且演进快** | overlaybd 分层、远端按需拉取、zstd、P2P——放内核驱动难开发/调试 |
| **安全稳定** | 内核只做最小转发；用户态崩溃不拖垮内核 |
| **高性能** | ublk + io_uring 让用户态块设备接近内核态性能（传统 NBD 做不到） |
| **易共享/快照** | 引用计数、warm pool、restack 等策略在用户态灵活实现 |

## 6. 相关代码与配置

| 主题 | 位置 |
|---|---|
| ublk 用户态库（设备/队列/控制） | `storage/ublk/src/`（`ctrl.rs`、`dev.rs`、`queue.rs`、`io_buffer.rs`、`impls/overlaybd_target.rs`） |
| ublk 守护进程（RPC/协议） | `storage/ublk-daemon/src/`（`client.rs`、`server.rs`、`protocol.rs`） |
| 节点侧设备管理 | `src/sandbox/ublk/device.rs`（`UblkDeviceManager`）、`src/sandbox/ublk/overlaybd.rs` |
| daemon 二进制路径 | 配置 `ublk.daemon_binary_path`（默认 `$AENV_HOME/ublk/uvm-ublk-daemon`）；环境变量 `AENV_UBLK_DAEMON_BINARY_PATH` |
| daemon socket | 配置 `ublk.daemon_socket_path`（默认 `$AENV_RUNTIME/ublk-daemon.sock`） |
| 内核模块加载 | 安装脚本 `modprobe ublk_drv`；需能打开 `/dev/ublk-control` |

## 7. 一句话总结

**`ublk_drv.ko` 是内核里的"设备接口 + I/O 快递员"，`uvm-ublk-daemon` 是用户态真正"干活的人"（overlaybd 前台 + 设备总管家）。** Firecracker 看到的是一块普通硬盘（`ublkbN`），而背后所有分层镜像、按需拉块、快照封存、内存共享，都是 daemon 在用户态完成的——这正是 AgentENV "高性能 I/O + 按需加载 + 快照"的基础。
