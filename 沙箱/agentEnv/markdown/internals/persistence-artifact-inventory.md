# 持久化 Artifact 清单

本文档列出 AgentENV 中可能在单次函数调用之后仍保留在磁盘或对象存储中的 artifact。它按拥有每个 artifact 生命周期的模块组织。

## 路径根

| 根 | 默认值 | 所有者 | 说明 |
| --- | --- | --- | --- |
| `home_path` | `/var/lib/aenv` | `src/cfg.rs` | 包含字面量 `$AENV_HOME` 占位符的路径的基础。`AENV_HOME_PATH` 在占位符展开之前覆盖它。 |
| `runtime_path` | `/run/aenv` | `src/cfg.rs`、`src/sandbox/network/*` | 瞬态命名空间挂载点和 daemon 套接字的基础。`AENV_RUNTIME_PATH` 覆盖它。 |
| `deps_path` | `$AENV_HOME/deps` | `src/cfg.rs`、`src/setup/*` | 下载的 runtime 依赖的基础。`AENV_DEPS_PATH` 可以将这些可重建资产放在 `home_path` 之外。 |
| 受管理的 Sandbox access-token seed | `$AENV_HOME/secrets/sandbox-access-token-hash-seed` | `src/sandbox/access.rs` | 当 `[sandbox].access_token_hash_seed` 未设置时，用于派生 envd 和流量 token 的节点本地 secret。它必须与持久化的 secure 或 private-ingress Sandbox 一起保留。 |
| Firecracker Sandbox 工作目录 | `$AENV_HOME/firecracker-work`，含 `agentenv-fc-` 子目录 | `src/sandbox/firecracker/*` | 每-Sandbox 的 runtime 目录，用于套接字、符号链接、ublk runtime 目录、本地日志和可写 OverlayBD upper layer 数据（`overlaybd/upper.data`、`overlaybd/upper.index`）。显式的 `[firecracker].work_dir` 会覆盖根。 |
| `firecracker.serial_dir` | `$AENV_HOME/logs/serial` | `src/sandbox/firecracker/*` | 持久化 Firecracker stdout/stderr 根，按 Sandbox ID 分组。显式的 `[firecracker].serial_dir` 会覆盖根。 |
| `managed_snapshot_root` | `<firecracker-work-base>/managed-snapshots` | `src/sandbox/firecracker/*` | 进程内 live Snapshot artifact 根，用于保持捕获的 Snapshot 存活直到发布或丢弃。 |
| `persisted_sandbox_store_path` | `$AENV_HOME/persisted-sandboxes` | `src/orchestrator/persistence/*` | 持久化的暂停 Sandbox 记录和 artifact。 |
| `snapshot_store` | `$AENV_HOME/snapshot-store` | `src/repository/*` | 持久化的已提交 Snapshot repository 根。已配置的后端使用 `<snapshot_store>/repository`。相对显式路径相对于配置文件目录解析。 |
| `snapshot.local_cache_path` | `$AENV_HOME/snapshot-local-cache` | `src/snapshot/artifact_cache.rs`、runtime resolver | 用于物化 runtime artifact 的节点本地缓存。相对显式路径相对于配置文件目录解析。 |
| `image.cache.root_dir` | `$AENV_HOME/image-cache` | `src/image/*`、overlaybd runtime | 节点本地镜像缓存根。包含 `configs/`、`indexes/`、`commits/` 和 `remote-blocks/`。离线 C++ 工具拥有隔离的兄弟缓存根：`convert-blocks/`（`overlaybd-apply`）和 `resize-blocks/`（`overlaybd-resize`）。 |
| `p2p.store_dir` | `$AENV_HOME/p2p/store` | `src/p2p/*`、`src/cfg.rs` | P2P artifact 传输后端的本地存储。相对显式路径相对于配置文件目录解析。 |
| `image.cache.remote_blocks` | `<image.cache.root_dir>/remote-blocks` | overlaybd runtime 配置 | 远程块缓存根。Overlaybd 还在该缓存目录下存储 `premerged-index/`。其大小限制来自 `image.cache.remote_blocks.max_size_gb`。 |
| `ublk.daemon_socket_path` | `$AENV_RUNTIME/ublk-daemon.sock` | `src/sandbox/ublk/*`、`storage/ublk-daemon/*` | 用于服务器到 daemon IPC 的 Unix 套接字。 |
| `ublk.daemon_log_path` | `$AENV_HOME/logs/ublk-daemon.log` | `storage/ublk-daemon/*` | 配置归一化期间提供的 daemon 日志文件。显式路径会覆盖默认值。 |

## 安装与配置

由 `src/setup/*` 和 `src/cfg.rs` 拥有。

| Artifact | 位置 | 内容 | 用途 | 生命周期 |
| --- | --- | --- | --- | --- |
| Firecracker 二进制 | `<deps_path>/firecracker/{version}/firecracker` | Firecracker 可执行文件 | VM 进程 runtime | 缺失时在安装期间创建。旧版本保留直到手动移除。 |
| CPU template 辅助工具 | `<deps_path>/firecracker/{version}/cpu-template-helper` | 可选的 Firecracker 辅助可执行文件 | 检测主机 CPU 配置以进行集群范围的 CPU 交集 | 存在时从 Firecracker 包中提取。 |
| 内核镜像 | `<deps_path>/kernel/{version}/vmlinux.bin` | Guest 内核 | VM 启动源 | 安装期间下载。旧版本保留直到手动移除。 |
| Tools 驱动器 | `<deps_path>/tools/{version}/tools.ext4` | 包含 envd/tools 的只读 ext4 镜像 | 由 Sandbox 共享、并通过其不可变 release 版本被 Snapshot 固定的 Firecracker root 驱动器 | 安装期间从 OCI 提取或从 `tools.drive_path` 导入。旧版本保留直到手动移除。 |
| Overlaybd 工具 | `<deps_path>/overlaybd/bin/*` | 静态链接的 `overlaybd-create`、`overlaybd-apply`、`overlaybd-commit` 和 `overlaybd-resize` | OCI 到 overlaybd 转换和打包 | 当 release 元数据不匹配时在安装期间安装。 |
| Overlaybd release 元数据 | `<deps_path>/overlaybd/tools-release.json` | 已安装的 overlaybd release 标识符 | 检测工具是否需要重新安装 | release 变化时在安装期间重写。 |
| Overlaybd 包下载 | `<deps_path>/overlaybd/downloads/*` | 临时下载的包归档 | overlaybd release 包的安装暂存 | 成功安装后移除。 |
| 生成的 overlaybd 配置 | `$AENV_HOME/overlaybd/overlaybd-global.json`、`$AENV_HOME/overlaybd/mem-overlaybd-global.json`、`$AENV_HOME/overlaybd/convert-overlaybd-global.json`、`$AENV_HOME/overlaybd/resize-overlaybd-global.json` | Runtime 全局配置、缓存路径、凭据配置 | 配置 overlaybd runtime、内存 Snapshot overlaybd 访问，以及离线 C++ 工具（`overlaybd-apply`、`overlaybd-resize`）；这些工具获得带隔离 cacheDirs（`convert-blocks`、`resize-blocks`）且禁用下载的专用配置 | 在安装/启动期间重写。 |
| Overlaybd runtime 日志 | `$AENV_HOME/overlaybd/overlaybd.log` | Overlaybd runtime 日志 | 调试 | 由 overlaybd runtime 追加；无自动 GC。 |
| 受管理的 Sandbox access-token seed | `$AENV_HOME/secrets/sandbox-access-token-hash-seed` | 编码为小写十六进制的 32 随机字节 | 当未配置显式 seed 时派生稳定的每-Sandbox envd 和流量 access token | 正常启动期间以 `0600` 模式原子创建，此后复用。secure 或 private-ingress Sandbox 持久化期间不得删除。 |

## Firecracker Sandbox

### Runtime

由 `src/sandbox/firecracker/*` 拥有。

| Artifact | 位置 | 内容 | 用途 | 生命周期 | 可重建 |
| ---------------------------- | ------------------------------------------------------------ | --------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 受管理的 live Snapshot 根 | `<firecracker-work-base>/managed-snapshots/{sandbox_id}/{uuid}/...` | 在进程内保持存活的 Firecracker Snapshot artifact | 持有捕获的运行中 Sandbox Snapshot 直到发布，并支持进程内暂停状态 | 由 `FirecrackerSandbox::pause()` 或 `snapshot()` 创建。在 `PersistentSnapshotRootGuard` drop 时移除。 | 否。 |
| Snapshot VM 状态 | Snapshot artifact 目录 `vm_state.bin` | Firecracker VM 状态 | 暂停/恢复和 Snapshot 发布输入 | 由 Firecracker 的仅状态 diff 快照创建。所有者取决于调用方：persister、managed root 或 repository 发布输入。 | 否。 |
| 内存 overlaybd layer | Snapshot artifact 目录 `mem_overlaybd/overlaybd.commit` | 内存页的已封存 OverlayBD layer | Runtime 内存恢复和发布输入 | 直接由使用 `process_vm_readv` 读取的 Firecracker dirty/present 内存范围创建。发布期间导入 repository 受管理 layer。 | 否。 |
| 内存镜像配置 | Snapshot artifact 目录 `mem_image.json` | 堆叠内存 layer 的 Overlaybd 镜像配置 | 恢复暂停的 Sandbox 并发布内存 layer | 内存转换后写入。稍后 repository resolver 从已提交 layer 重新生成 runtime 内存配置。 | 发布后可重建；暂停 Sandbox 除非已知 layer，否则不可重建。 |
| Rootfs Snapshot 配置 | Snapshot artifact 目录 `rootfs/image.json` | 捕获的 rootfs 状态的 Overlaybd 镜像配置 | 恢复暂停的 Sandbox 并发布 rootfs layer | restack/seal 后从 live runtime 配置暂存。 | 仅可从关联 layer 和元数据重建。 |
| Rootfs Snapshot layer | Snapshot artifact 目录 `rootfs/snapshot.commit` | 来自 live rootfs 的已封存可写 upper | 捕获自上一层 lower 堆栈以来的磁盘写入 | 由 ublk daemon restack 为可写 overlaybd rootfs 创建。 | 否。 |
| 继承的 runtime layer | Snapshot artifact 目录 `rootfs/inherited-layers/{index}/{source-file}` | 由 Snapshot 拥有的、继承的 runtime 创建 lower 后缀的硬链接或副本 | 消除对先前 managed Snapshot 或持久化 Sandbox artifact 根的依赖 | 当继承的 lower 来自 managed Snapshot 根或同一持久化 `artifacts` 根下的另一个 Sandbox/代际时，在暂停期间创建。 | 否。 |
| Attached-drive Snapshot 目录 | Snapshot artifact 目录 `drives/{drive_id}/...` | 每驱动镜像配置和 Snapshot layer | 捕获可写 attached-drive 状态 | 与每个驱动器的 rootfs Snapshot 一起创建。 | 否。 |
| Firecracker 工作目录 | 已配置的工作根，或消费的池临时文件目录 | API 套接字、符号链接、runtime 目录、本地日志 | Sandbox 的 Firecracker CWD | 在构建 Sandbox handle 时创建，或从 warm 池条目移动；由拥有的 `TempDir` 移除。 | 是，live 状态除外。 |
| Firecracker serial 日志 | `firecracker.serial_dir/{sandbox_id}/*`，或 warm 池工作目录日志 | Firecracker stdout/stderr | 调试 | 在 spawn 时打开，仅当 `firecracker.log_level` 非空或设置了显式 Rust 捕获目标时追加。已启用的 warm 日志在消费 warm 进程时被重新定位。已配置的 serial 输出不会自动 GC。 | 否，但可丢弃。 |
| Firecracker logger 输出 | 与 serial 日志相同的每-Sandbox 日志目录中的 `firecracker.log` | Firecracker 内部 logger（`PUT /logger`） | 调试 | 仅当 `firecracker.log_level` 设置为非空级别时创建。由 Firecracker 自身写入；不会自动 GC。 | 否，但可丢弃。 |

### Firecracker 池

由 `src/sandbox/firecracker/pool.rs` 拥有。

| Artifact | 位置 | 内容 | 用途 | 生命周期 |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------ | ----------------------------------- | ------------------------------------------------------------ |
| Warm 池工作目录 | 系统临时文件目录 | Firecracker 套接字和 warm 进程日志 | 预生成 Firecracker 进程的 CWD | 由池维护作为 `TempDir` 创建。在 warm 条目清理时移除，或移入消费的 Sandbox 并在那里稍后 drop。 |
| Warm 池日志 | warm 池工作目录 `firecracker-stdout.log`、`firecracker-stderr.log` | warm 进程被消费之前的 Firecracker 输出 | 调试 warm 启动 | 仅当 `firecracker.log_level` 非空时在 warm spawn 时创建。消费时如果没有显式 stdout/stderr 覆盖，则重新定位到 Sandbox 日志路径。 |

## 额外驱动器

由 `src/sandbox/extra_drive.rs` 和 Firecracker Snapshot 代码拥有。

| Artifact | 位置 | 内容 | 用途 | 生命周期 | 可重建 |
| ----------------------------- | -------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ----------------------------------------- |
| Extra-drive runtime 目录 | Sandbox 工作目录 `extra-drive-runtime-{drive_id}/` | Runtime `image.json`、upper 文件、result 文件 | attached drive 的 ublk runtime | 准备 extra drive 时创建。回滚/停止时释放；工作目录清理会移除文件。 | 运行期间不可重建；其他情况不必要。 |
| Extra-drive 符号链接 | Sandbox 工作目录 `extra-drive-{drive_id}` | 指向 `/dev/ublkbN` 设备路径的符号链接 | Firecracker 驱动器挂载路径 | 在 ublk runtime 设备创建后创建。回滚或工作目录清理时移除。 | 是。 |
| Extra-drive Snapshot artifact | Snapshot artifact 目录 `drives/{drive_id}/...` | 捕获的驱动 overlaybd 配置和 commit layer | 跨暂停/恢复/发布保留 attached-drive 可写状态 | 在 Sandbox Snapshot/暂停期间创建。之后由 persister、managed root 或 repository 发布流程拥有。 | 否。 |

## Snapshot 存储模型

Snapshot 数据经过三个存储层，具有不同的所有权和
生命周期规则：

1. **Builder 暂存**是 manager 拥有的临时工作区，位于
 `<snapshot.local_cache_path>/snapshots/<id>/`。它在构建或捕获进行期间
 持有本地 rootfs、memory、VM-state 和 attached-drive artifact。它不是持久化
 Snapshot 记录。
2. **已提交 Snapshot repository** 存储 Snapshot 目录、别名、
 `snapshot.json`、`firecracker-manifest.json`、`vm_state.bin` 以及
 引用的 managed layer。这是 Template 和 Snapshot API 暴露的持久化
 真相来源。
3. **节点本地 runtime 缓存**在启动之前将可运行的 rootfs、memory 和
 drive `image.json` 文件物化到
 `<snapshot.local_cache_path>/runtime/<id>/` 下。这些文件是派生的
 runtime 输入，可以从已提交状态重建。

已提交的 `snapshot.json` 记录捕获的 runtime 上下文、启动配置以及 rootfs、drive 和 memory layer 引用。Firecracker manifest 记录启动元数据，例如虚拟大小和 attached-drive 配置。临时 upper 文件和生成的 `image.json` 文件不是已提交 Snapshot 真相。

在 runtime 解析期间，AgentENV 将已提交 layer 引用转换为节点本地 rootfs、memory 和 attached-drive 配置，用节点本地路径填充 Firecracker manifest，并将已提交 `vm_state.bin` 解析为可运行路径。

## 镜像 Resolver

由 `src/image/*` 拥有。

| Artifact | 位置 | 内容 | 用途 | 生命周期 |
| --- | --- | --- | --- | --- |
| 镜像配置缓存 | `<image.cache.root_dir>/configs/*-image.json` | 已解析用户镜像的、按 digest 限定的 overlaybd 镜像配置 | 避免重复进行 OCI manifest 分类和配置生成 | 在镜像解析时写入。如果无效或引用的本地 lower 缺失则重新生成。目前没有统一 GC。 |
| 镜像元数据 sidecar | `<image.cache.root_dir>/configs/*.metadata.json` | 来自 OCI 镜像配置的 base env/workdir 元数据 | 在缓存镜像配置旁边保留镜像启动上下文 | 在镜像配置之后写入。在镜像配置可用时如果缺失则重建。 |
| Overlaybd commit 缓存 | `<image.cache.root_dir>/commits/{digest-slug}/overlaybd.commit` | 来自标准 OCI 转换的内容寻址 overlaybd commit layer，以及远程 overlaybd-native layer 的目标目录 | 用户镜像 layer 的节点本地可复用 layer 存储 | 标准 OCI 转换写入 commit。远程 overlaybd-native 配置将 `dir` 指向此处以供 runtime 填充。目前没有统一 GC。 |
| OCI 转换索引 | `<image.cache.root_dir>/indexes/{source-digest}/...json` | 从 OCI 源 layer/上下文到 overlaybd commit digest 和大小 的映射 | 当转换后的 commit 存在时跳过重复 layer 转换 | 成功标准 OCI 转换后写入。目前没有统一 GC。 |
| 转换后的 OCI layer P2P artifact | P2P 目录 key `oci-layer/v1/{context-hash}` | 已完成的 OverlayBD commit 及其转换上下文和输出 digest/size | 让另一个节点复用兼容的标准 OCI 转换 | 本地 commit 持久化后作为 Reference 发布；所有权持久化在 hard-commit 记录上，并由 image-cache GC 移除。 |
| 临时 OCI 拉取/转换工作 | 进程临时目录 | OCI layout 和每-layer 转换工作区 | 标准 OCI 转换的中间输入 | 由 `TempDir` 拥有；转换 scope 退出后移除。 |

## Snapshot Repository

由 `src/repository/*` 和 `src/snapshot/types/*` 拥有。

| Artifact | 位置 | 内容 | 用途 | 生命周期 | 可重建 |
| --- | --- | --- | --- | --- | --- |
| Snapshot 记录 | POSIX：`<snapshot_store>/repository/catalog/records/{id}.json`；OSS：`catalog/records/{id}.json` | Snapshot ID、别名、来源、资源、构建状态、已提交逻辑元数据 | 持久化用户可见的 Snapshot/Template 元数据 | 由 `src/snapshot/catalog.rs` 写入的共享文档。在 Template 构建之前或发布时创建。在提交或出错时更新。Snapshot 删除会先将其标记为 `deleting`，因此之后不会再附加启动 manifest，并最后删除它。 | 否。 |
| Snapshot 别名 | POSIX：`<snapshot_store>/repository/catalog/aliases/{alias}`；OSS：`catalog/aliases/{alias}.json` | 别名到 Snapshot 的绑定 | Template/Snapshot 的名称查找 | 共享文档。在记录写入后声明；在记录删除前释放。记录已消失的绑定是陈旧的，会被下次查找或声明移除。 | 部分可重建，如果别名仍被记录，则可从记录重建。 |
| 共享文档（Snapshot 记录和别名、Volume 记录和名称、构建日志、构建缓存状态） | `src/repository/layout.rs` 中命名的路径。POSIX：各一个文件，在锁文件下更新。OSS：各一个 `{name}.json` 对象。S3 仅在其 ETag 未变化时替换它。Alibaba OSS 不能，因此每次写入都在可追加锁对象 `locks/{path}` 上持有 lease（参见 `src/repository/backends/oss/lease.rs`）。 | Volume 状态、预留和名称绑定；Template 构建日志；共享构建缓存 seed | 由每个节点上的 Snapshot 和 Volume 目录以及 Template builder 原子更新 | 释放的 lease 会删除其锁对象。死掉的持有者会留下它，直到竞争者 30 秒后接管。 | 否。 |
| Firecracker manifest | POSIX：`<snapshot_store>/repository/snapshots/{id}/firecracker-manifest.json`；OSS：`artifacts/{id}/firecracker-manifest.json` | Firecracker Snapshot 形态、虚拟大小、attached-drive 元数据；路径字段在 runtime 填充 | 用于启动已提交 Snapshot 的 runtime manifest 模板 | 发布期间持久化。随每-Snapshot artifact 一起移除。 | 部分可重建，但保留为持久化 artifact。 |
| VM 状态 | POSIX：`<snapshot_store>/repository/snapshots/{id}/vm_state.bin`；OSS：`artifacts/{id}/vm_state.bin` | Firecracker VM 状态 Snapshot | 恢复已提交 Snapshot 所需 | 发布期间复制/上传。随每-Snapshot artifact 一起移除。 | 否。 |
| Managed Snapshot layer | POSIX：`<snapshot_store>/repository/managed-layers/{digest}.overlaybd.commit`；OSS：`managed-layers/{digest}` | 内容寻址的 rootfs、attached-drive 和 memory overlaybd commit layer | 已提交 Snapshot 的共享不可变 layer 存储 | 发布期间按描述符导入，或通过对没有描述符的本地 layer 进行哈希导入。通常不会随单个 Snapshot 移除。 | 仅凭元数据不可重建；只有在源仍存在时才能重新获取。 |
| 暂存文件 | POSIX：`<snapshot_store>/repository/staging/*`；OSS：进程临时目录 | 为存储准备的 dense 导出和重新容器化 layer | 在 POSIX 上，与 `managed-layers/` 是同一文件系统，因此准备好的 layer 可通过硬链接加入而无需复制 | 临时文件，在生成它们的发布完成后移除。发布中途崩溃的进程会留下其文件。 | 是。 |
| 源 registry 发布 | `SnapshotRecord.committed.disk_publications` 以及远程 OCI registry 对象 | 已发布的 rootfs/attached-drive 镜像引用和 manifest digest | 让兼容的 Snapshot delta 存在于其源 registry 中 | 启用镜像发布时在 OSS 发布期间创建。失败或删除时尽可能回滚。 | Registry 拥有；需要记录来定位。 |

Repository 记录是已提交 Snapshot 的逻辑真相来源。构建时和 runtime 的 `image.json` 文件不应成为已提交真相，除非它们包含无法从已提交元数据派生的数据。

## Snapshot Runtime 解析

由 `src/snapshot/artifact_cache.rs`、`src/snapshot/runtime_support.rs` 和后端 runtime resolver 拥有。

| Artifact | 位置 | 内容 | 用途 | 生命周期 | 可重建 |
| --- | --- | --- | --- | --- | --- |
| Runtime rootfs 配置 | `<snapshot_local_cache_path>/runtime/{snapshot_id}/rootfs/image.json` | 用于已提交 rootfs layer 的节点本地 overlaybd 配置 | 在当前节点启动已提交 Snapshot | 在 Snapshot 解析期间物化，并由 `RunnableSnapshot` lease 固定。解除固定后可被 LRU 驱逐。 | 是。 |
| Runtime memory 配置 | `<snapshot_local_cache_path>/runtime/{snapshot_id}/memory/image.json` | 用于已提交 memory layer 的节点本地 overlaybd 配置 | 向 Firecracker 恢复提供内存后端镜像 | 在 Snapshot 解析期间物化，并由 lease 固定。解除固定后可被 LRU 驱逐。 | 是。 |
| Runtime attached-drive 配置 | `<snapshot_local_cache_path>/runtime/{snapshot_id}/drives/{drive_id}/image.json` | 用于 attached drive 的节点本地 overlaybd 配置 | 使用 attached drive 启动已提交 Snapshot | 在 Snapshot 解析期间物化，并由 lease 固定。解除固定后可被 LRU 驱逐。 | 是。 |
| OSS 缓存 VM 状态 | `<snapshot_local_cache_path>/artifacts/{id}/vm_state.bin` | 已下载的 VM 状态对象 | 在固定/缓存期间避免重复对象存储下载 | 由 OSS resolver 通过 `LocalArtifactCache` 下载。解除固定后可被 LRU 驱逐。 | 是，通过重新下载。 |
| POSIX VM 状态引用 | Repository `vm_state.bin` 路径 | 指向 POSIX repository 的直接路径 | 避免将 VM 状态复制到本地 runtime 缓存 | 在解析期间检查。生命周期跟随 repository artifact。 | 否；repository artifact 是来源。 |

`LocalArtifactCache` 拥有对其管理文件的固定、进行中物化去重和 LRU 驱逐。它不拥有持久化 Snapshot repository。

## Orchestrator 持久化

由 `src/orchestrator/persistence/*` 拥有。

| Artifact | 位置 | 内容 | 用途 | 生命周期 |
| --- | --- | --- | --- | --- |
| 暂停 Sandbox 记录 DB | `<persisted_sandbox_store_path>/records.db` | 以 Sandbox ID 为键的 RocksDB；值是紧凑 JSON 记录，包含版本、生命周期、Sandbox 元数据、artifact 根和后端状态 | 在服务器重启后恢复暂停的 Sandbox | 暂停成功后写入记录。恢复前标记为 `resuming`。恢复失败时回滚。恢复/删除时删除。 |
| 暂停 Sandbox artifact 代际 | `<persisted_sandbox_store_path>/artifacts/{sandbox_id}/{uuid}/...` | 由 `pause_to_dir` 生成的 Firecracker Snapshot artifact | 一个暂停 Sandbox 代际的持久化 artifact | 暂停前分配，由后端填充，由暂停记录引用。在显式删除或 `load_all` 孤儿清理期间移除。 |

暂停 Sandbox artifact 不是已提交 Snapshot repository artifact。它们由 Sandbox persister 拥有，不应作为 Snapshot 真相共享。

## P2P Artifact 传输

由 `src/p2p/*` 拥有。

| Artifact | 位置 | 内容 | 用途 | 生命周期 |
| --- | --- | --- | --- | --- |
| Iroh blob 存储 | `<p2p.store_dir>/iroh` | `iroh-blobs` 内容寻址数据和内部元数据 | 已发布 artifact 和缓存获取的本地存储 | 在 P2P 传输初始化时创建，在发布和成功远程获取时增长。这些 blob 在 unpublish 移除那些标签后由 GC 收集。 |
| P2P 目录 DB | `<p2p.store_dir>/iroh/catalog.db` | 以 artifact key 为键的 RocksDB；值是紧凑 JSON artifact 描述符 | 让 peer 将稳定 key 解析为该节点的描述符 | 启动时加载。单个条目在发布时以及成功远程获取后 upsert，在 unpublish 时删除。 |

Snapshot 发布在 repository 提交后写入尽力而为的 P2P 目录条目：

- `snapshot/v1/artifacts/{snapshot_id}/vm_state.bin`
- `snapshot/v1/artifacts/{snapshot_id}/firecracker-manifest.json`
- `overlaybd-layer/v1/sha256:<digest>`，用于引用的 rootfs、memory 和 attached-drive overlaybd commit layer

这些 P2P 条目是可选的副本，不是已提交 Snapshot 真相。清除 `<p2p.store_dir>` 会降低点对点可用性，并迫使 peer 回退到 OSS 或 origin-registry 读取，但本身不得使已提交 Snapshot 无效。

## Ublk Daemon Runtime

由 `storage/ublk-daemon/*` 和 `src/sandbox/ublk/*` 拥有。

| Artifact | 位置 | 内容 | 用途 | 生命周期 | 可重建 |
| --- | --- | --- | --- | --- | --- |
| Overlaybd runtime 镜像配置 | 调用方提供的 runtime 目录 `image.json` | 用 runtime 相对 lower 和 upper 路径重写的 overlaybd 配置 | 打开 ublk overlaybd target | 在 `CreateOverlaybdRuntimeDevice` 期间物化。回滚或工作目录清理期间移除。 | 未运行时可从源镜像配置重建。 |
| Runtime upper 数据 | runtime 目录 `upper.data` | 可写 overlaybd upper 数据文件 | 存储可写设备的 live 写入 | 如果源配置没有现有 upper，则为可写 runtime 创建。暂停/Snapshot 时 restack 为 Snapshot layer。 | 运行时不可重建。 |
| Runtime upper 索引 | runtime 目录 `upper.index` | 日志结构 upper 索引 | 解析 `upper.data` 中的写入 | 随日志结构可写 upper 一起创建。Restack 为已封存 Snapshot layer。 | 运行时不可重建。 |
| Runtime result 文件 | runtime 目录 `result.txt` | Overlaybd result 文件路径 | Overlaybd runtime 约定 | 由 overlaybd runtime 创建/使用。清理期间移除。 | 是。 |
| Ublk daemon 套接字 | 默认 `$AENV_RUNTIME/ublk-daemon.sock` | Unix 套接字 | AgentENV 服务器与 ublk daemon 之间的 IPC | daemon 启动时创建；由进程生命周期移除/替换。 | 是。 |
| Ublk daemon 日志 | 已配置的 `ublk.daemon_log_path`，默认 `$AENV_HOME/logs/ublk-daemon.log` | Daemon 日志 | 调试 | 由 daemon 追加；部署负责轮转和保留。 | 否，但可丢弃。 |

## Overlaybd 存储

由 `storage/overlaybd/*` 拥有。

| Artifact | 位置 | 内容 | 用途 | 生命周期 | 可重建 |
| --- | --- | --- | --- | --- | --- |
| 远程块缓存 | 已配置的 `cacheConfig.cacheDir`，派生自 `<image.cache.root_dir>/remote-blocks` | 缓存的 registryfs_v2 块范围 | 加速远程 overlaybd-native layer 读取 | 由 overlaybd 缓存设置管理。 | 是。 |
| 离线 C++ 工具块缓存 | `<image.cache.root_dir>/convert-blocks`（`overlaybd-apply`）、`<image.cache.root_dir>/resize-blocks`（`overlaybd-resize`） | 离线工具的 C++ file-cache 条目 | 使 C++ 缓存驱逐（对扁平 cacheDir 文件执行 truncate+unlink）远离 Rust runtime 缓存的每-entry 目录 | 由 C++ 工具通过其专用生成的全局配置拥有和驱逐；这些配置中禁用下载。永不与 Rust runtime `remote-blocks` 缓存共享。 | 是。 |
| Premerged 索引缓存 | `cacheConfig.cacheDir/premerged-index/*.pmidx` | 序列化的合并只读 lower 索引 | 加速重复 lower 堆栈的打开 | 在只读打开时异步写入。按从缓存大小派生的大小限制进行修剪。 | 是。 |
| 已封存 overlaybd commit 文件 | 各种所有者路径：镜像缓存、Snapshot 目录、repository managed layer | Overlaybd layer 数据和索引 trailer | 块设备和内存镜像的不可变 lower layer | 生命周期由存储该文件的模块拥有。Overlaybd 仅定义格式和打开/合并行为。 | 取决于所有者。 |

## 所有权规则

- Snapshot repository artifact 是持久化的用户可见状态。不要从节点本地 GC 代码中删除它们。
- 暂停 Sandbox artifact 仅对拥有它们的暂停 Sandbox 持久化。不要将它们视为已提交 Snapshot。
- Snapshot 本地缓存或 ublk runtime 目录下的 runtime `image.json` 文件是派生 artifact。它们应可从已提交元数据或源镜像配置重建。
- 节点本地 `image-cache` artifact 是一次性缓存，但内容寻址的 commit 重新生成可能代价高昂。元数据支撑的 GC 仅回收不再被磁盘上源配置作为根、被 image-cache lease 持有、或被进程内运行集引用的 commit。已提交 Snapshot 是持久化 repository 状态，不会固定 ImageCache commit。调度请参见配置参考中的 `[image.cache.gc]`。
- P2P 存储内容是节点本地且可选的。Unpublish 移除本地目录条目和相关 blob；清除存储仅影响点对点共享，并可能需要重新发布或重新获取 artifact。
- 日志和依赖下载是运维 artifact。它们不是 Sandbox 或 Snapshot 正确性的一部分，但可能需要在 image-cache GC 之外单独制定保留策略。
