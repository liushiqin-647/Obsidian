# Template Builder 与测试

本文档解释当前面向 Template（模板）的构建 API 如何映射到
Snapshot 优先的内部实现、已提交 Snapshot 如何穿过
repository/runtime 边界，以及相关测试验证的内容。

## 1) 公共表面与内部模型

### 面向用户的 builder API

- `TemplateBuildSpec`（`src/template/build_spec.rs`）
  - 描述一个声明式构建请求。
  - Builder 辅助方法包括：
    - `from_existing_rootfs(path)`
    - `from_overlaybd_configs(global_config_path, image_config_path)`
    - `run(cmd)`
    - `env(key, value)`
    - `workdir(path)`
    - `apt(packages)`
    - `alias(alias)`
    - `resources(cpu_count, memory_mib)`
  - 当前 builder 实现接受由 overlaybd 支撑的全新构建和
 从已提交 Snapshot 的重建。全新 ext4-rootfs 构建尚未通过 `TemplateBuilder`
 接通。

- `TemplateBuilder`（`src/template/builder.rs`）
  - 保留构建/重建流程的外部 Template 语义，而
 已提交 Snapshot 生命周期操作位于 `SnapshotManager`。
  - 主要方法：
    - `new()`
    - `with_local_store_root(path)`
    - `build_and_publish(snapshot_manager, config).await`
    - `rebuild_and_publish(snapshot_manager, config, snapshot_id, base_snapshot).await`

### Snapshot 优先的内部实现

- `StoredSnapshot` / `RunnableSnapshot`（`src/snapshot/types/`）
  - `StoredSnapshot` 是持久化的已提交 manifest 状态。
  - 它存储 Snapshot 元数据以及逻辑 rootfs / attached-drive layer
 描述和 memory layer。
  - Snapshot 发布还会持久化 `firecracker-manifest.json`，用于
 启动时元数据，例如 rootfs/memory 虚拟大小和 attached-drive
 `read_only` 标志。
  - 它不存储固定的本地 artifact 位置，例如 `mem_image.json`
 或 runtime 派生的 `rootfs/image.json`。
  - `RunnableSnapshot` 是节点本地的已解析视图，具有具体 artifact
 路径和 runtime 就绪的 overlaybd 镜像配置。

- `Repository`（`src/repository/interfaces.rs`）/
 `SnapshotRuntimeResolver`（`src/snapshot/resolver.rs`）
  - `Repository` 拥有已提交的持久化状态。
  - `SnapshotRuntimeResolver` 将已提交状态转换为节点本地可运行
 路径。

## 2) 构建与发布流程

`TemplateBuilder::build_and_publish(...)` 执行以下操作：

1. 从以下之一准备构建基础：
   - 已提交 Snapshot（`rebuild_and_publish`）
   - overlaybd 配置（`from_overlaybd_configs`）
2. 使用请求的 CPU/内存启动一个临时 Firecracker Sandbox（沙盒）。
3. 按顺序执行 Template 步骤。
4. 从运行中的 guest 和可执行文件探测 `envd`、kernel 和 firecracker 版本。
5. 暂停 Sandbox 并导出一个携带 repository 发布元数据的 `FirecrackerSnapshotManifest`
6. 将这些本地 artifact 发布到已配置的 Snapshot repository。

重要的边界是：

- builder API 描述要构建什么
- 本地构建 artifact 是 manager 拥有的临时输出
- 已提交 Snapshot 记录存储逻辑持久化状态
- 后端布局约定决定已提交文件和共享 layer 的位置
- runtime 解析将已提交 Snapshot 状态转换为节点本地可运行路径

## 3) Repository 布局

默认后端是 POSIX 文件系统后端。给定一个后端根，如
`/path/to/store/repository`：

- 已提交 Snapshot manifest：
  - `/path/to/store/repository/snapshots/<id>/snapshot.json`
- 已提交固定布局文件：
  - `/path/to/store/repository/snapshots/<id>/vm_state.bin`
  - `/path/to/store/repository/snapshots/<id>/firecracker-manifest.json`
- 共享 managed layer：
  - `/path/to/store/repository/managed-layers/<digest>.overlaybd.commit`
- 别名绑定：
  - `/path/to/store/repository/catalog/aliases/<alias>`
  - `/path/to/store/repository/catalog/aliases/<alias>.lock`

别名锁定是按别名的，而非单个全局 `aliases.lock` 文件。

`snapshot.json` 有意不重复本地构建 artifact 路径，例如 `mem_image.json` 或 `rootfs/image.json`。Runtime 代码从以下派生节点本地镜像配置：

- repository 根
- Snapshot id
- attached drive id
- managed layer digest

`firecracker-manifest.json` 通过在不嵌入节点本地 runtime
路径的情况下持久化启动时虚拟大小和 attached-drive 模式元数据来补充这一点。

## 4) 加载与启动流程

典型 runtime 用法是：

1. `SnapshotManager::load_committed(id_or_alias).await`
2. `SnapshotManager::resolve_runnable(stored).await`
3. `FirecrackerSandbox::from_snapshot(&runnable, &SandboxLaunchConfig::default())`
4. `sandbox.start().await`

如果希望合并两次 snapshot-manager 调用，请使用
`SnapshotManager::load_runnable(id_or_alias).await`。

`resolve_runnable(...)` 是后端中立的已提交状态变为
节点本地启动输入的地方：

- `memory_layers` -> runtime `memory/image.json`
- `rootfs.layers` -> runtime `rootfs/image.json`
- `attached_drives[].layers` -> runtime `drives/<id>/image.json`
- `attached_drives[].read_only` -> `drives/<id>` 的 runtime 挂载模式
- `firecracker-manifest.json` -> 用节点本地
 artifact 路径填充的 runtime manifest
- 已提交 `vm_state.bin` -> 可运行 vm-state 路径

## 5) 最小示例

```rust
use agentenv::cfg::ConfigManager;
use agentenv::image::ImageResolver;
use agentenv::sandbox::{FirecrackerSandbox, SandboxExecutor, SandboxLaunchConfig};
use agentenv::snapshot::SnapshotManager;
use agentenv::template::{TemplateBuildSpec, TemplateBuilder};

let builder = TemplateBuilder::new();
let snapshot_manager = SnapshotManager::new()?;
let config = ConfigManager::global()?.config();
let image_resolver = ImageResolver::new(config);
let image_config = image_resolver
  .resolve(image_resolver.default_image())
  .await?
  .overlaybd_config_path;

let alias = "my-template-v1";

builder
    .build_and_publish(
        &snapshot_manager,
        TemplateBuildSpec::new()
            .from_overlaybd_config(image_config)
            .alias(alias)
            .resources(1, 128)
            .run("mkdir -p /workspace")
            .workdir("/workspace")
            .env("MARK", "ready")
            .run("printf '%s' \"$MARK\" > mark.txt"),
    )
    .await?;

let runnable = snapshot_manager
    .load_runnable(alias)
    .await?
    .expect("template should exist");

let mut sandbox =
    FirecrackerSandbox::from_snapshot(&runnable, &SandboxLaunchConfig::default())?;
sandbox.start().await?;

let out = sandbox.run_command("cat", &["/workspace/mark.txt"]).await?;
assert_eq!(out.exit_code, 0);
assert_eq!(out.stdout.trim(), "ready");

sandbox.stop().await?;
```

## 6) 状态转换

面向 Template 的状态转换是：

1. 本地构建 artifact
2. 已提交 `snapshot.json` + `firecracker-manifest.json` + 固定文件 + managed layer
3. 节点本地 runtime 派生的镜像配置
4. Firecracker 恢复输入

## 7) 测试验证的内容

`tests/integration/snapshot.rs` 覆盖：

- 从 overlaybd 配置构建和发布
- 通过 Snapshot manager 按别名/id 加载已提交 Snapshot
- 将已提交 Snapshot 解析为可运行 Snapshot
- 从解析后的 Snapshot 启动 Firecracker Sandbox
- 从已提交 Snapshot 重建并保留基础状态
- 通过 Template API 表面列出和删除已提交 Snapshot

`tests/integration/snapshot_attached_drive.rs` 覆盖 overlaybd attached-drive 处理
跨 Sandbox 捕获、发布、解析、启动和重建流程，包括：

- 已提交元数据保留 `readOnly`
- 只读和可写驱动器的 runtime 挂载模式
- 重建保持 attached-drive 元数据稳定

`crates/e2e-tests/tests/snapshot_oss_e2e_test.rs` 覆盖 OSS 支撑的 Snapshot 发布、
解析、别名清理、缺失 managed-layer 的失败模式以及删除。

`src/snapshot/backends/` 和 `src/snapshot/store/` 下的 Snapshot 存储单元测试
针对 POSIX repository 覆盖：

- artifact 导入
- 别名处理
- 已提交 Snapshot 删除
- runtime 缓存行为

## 8) 常见失败区域

- overlaybd 基础配置路径缺失或无效
- 使用 overlaybd 构建基础时 `ublk` 被禁用
- Firecracker、所选 KVM/PVM 模式或网络命名空间的 Linux 主机前置条件
 缺失
- 发布期间 repository 别名冲突
- runtime resolver 无法从已提交 artifact 物化本地路径
