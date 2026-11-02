# Sandbox 与测试流程指南

本文档描述公共 Sandbox（沙盒）API、`AgentENV/src/cfg.rs` 下的全局 `ConfigManager`，以及 `AgentENV/tests/integration/fc.rs`、
`AgentENV/tests/integration/process.rs`、`AgentENV/tests/integration/snapshot.rs` 和
`AgentENV/tests/integration/orchestrator.rs` 验证的内容。

## 1) Sandbox 公共接口

### 类型与职责

- `FirecrackerSandboxConfig`（`AgentENV/src/sandbox/firecracker/config.rs`）
  - **用途**：描述一次全新的 VM 启动（kernel + tools 驱动器 + 用户镜像）。
  - **关键字段**
    - `firecracker_binary`：`firecracker` 可执行文件的路径。
    - `kernel_image`：`vmlinux.bin` 的路径。
    - `tools_drive_version`：不可变的 tools 驱动器标识。节点在将其挂载为 `/dev/vda` 之前解析为
 `<deps_path>/tools/<version>/tools.ext4`。
    - `user_image_config`：可写用户镜像（`/dev/vdb`）的 overlaybd 配置。
    - `boot_args`：内核启动参数（例如 `console=ttyS0 ... init=/init`）。
    - `vcpu_count`、`mem_size_mib`：VM 大小。
    - `runtime_policy`：套接字/envd 超时和轮询间隔。
    - `common.stdout_path`、`common.stderr_path`：显式捕获目录。
 否则仅当 `common.firecracker_log_level` 非空时才启用 stdout/stderr 捕获，
 使用 serial 目录或 `work_dir/logs` 回退。
    - `envd_version`：guest 镜像期望的 envd 版本。
    - `env_vars`：在 envd 初始化后注入的可选默认环境变量。
    - `ublk_config`：可选的由 ublk 支撑的 rootfs 配置。
  - **便捷方法**
    - `from_global_config()` 使用内部默认 Template rootfs 镜像从 `ConfigManager` 构建配置。
 该镜像必须已在 `<image-cache-root>/configs` 下解析。
    - `from_global_config_with_user_image(...)` 从已解析的 overlaybd 镜像配置构建配置。
    - 从已提交 Snapshot 的 runtime 启动通过
 `FirecrackerSandbox::from_snapshot(&RunnableSnapshot, &SandboxLaunchConfig)` 构建。
    - `uid`/`gid`：文件所有权（默认 `0`）。

- `ProcessOpts`（`AgentENV/src/sandbox/process.rs`）
  - **用途**：在 Sandbox 内启动进程的选项。
  - **关键字段**
    - `envs`：进程环境变量的 `HashMap<String, String>`。
    - `cwd`：可选的工作目录。
    - `timeout`：等待完成的可选最大时间。
  - **Builder 方法**：`with_envs()`、`with_cwd()`、`with_timeout()`。

- `ProcessOutput`（`AgentENV/src/sandbox/process.rs`）
  - **用途**：已完成进程执行的结果。
  - **字段**：`stdout`、`stderr`、`exit_code`。

- `ProcessHandle`（`AgentENV/src/sandbox/process.rs`）
  - **用途**：Sandbox 内运行中进程的句柄。
  - **关键方法**
    - `pid()`：返回 guest VM 内的 PID。
    - `wait().await`：等待退出并收集所有输出。
    - `send_stdin(data).await`：向 stdin 发送字节。
    - `send_signal(signal).await`：发送信号（例如 SIGTERM）。
    - `kill().await`：使用 `SIGKILL` 杀死进程。

- `FirecrackerSnapshotConfig`（`AgentENV/src/sandbox/firecracker/config.rs`）
  - **用途**：描述如何从 Snapshot + memory + base disk 恢复 VM。
  - **关键字段**
    - `vm_state_path`：Firecracker VM 状态文件。
    - `mem_overlaybd_config`：runtime 内存 overlay 镜像配置。
    - `base_rootfs_path`：与该 Snapshot 配对的
 可写磁盘镜像或 overlaybd rootfs 镜像配置。
    - `firecracker_binary`：用于恢复的 FC 二进制。
    - `runtime_policy`：套接字/envd 超时和轮询间隔。
    - `envd_version`：guest envd 版本元数据。
    - `env_vars`：启动时恢复的可选默认环境变量。
  - **说明**
    - `pause()` 返回的 `FirecrackerSnapshotConfig` 拥有一个 tempdir（保持文件存活）。
    - `get_rootfs_path()` 解析 runtime 测试使用的 Snapshot rootfs 路径。

- `SandboxBackend` 和 `SandboxExecutor`（`AgentENV/src/sandbox/backend.rs`）
  - **用途**：Sandbox 生命周期和进程执行的公共 trait。
  - **关键方法**
    - `SandboxBackend`：`start`、`start_nowait`、`wait_for_ready`、`pause`、`resume`、`stop`。
    - `SandboxExecutor`：`run_command`、`run_command_with_opts`、`start_process`。

- `FirecrackerSandbox`（`AgentENV/src/sandbox/firecracker/sandbox.rs`）
  - **用途**：单个 Firecracker 实例的生命周期控制器。
  - **主要方法（详细）**
    - `FirecrackerSandbox::new(FirecrackerSandboxConfig)`
      - 为全新启动创建 Sandbox handle。
      - 不启动 Firecracker；仅准备对象。
      - 每个 Sandbox 都有自己的临时工作目录。
    - `FirecrackerSandbox::resume_from_snapshot_config(&FirecrackerSnapshotConfig)`
      - 创建新的 Sandbox handle 并立即从 Snapshot 配置启动它。
      - 使用节点当前配置的 Firecracker，无论是取用预热进程还是启动新进程。
 跨版本 Snapshot 兼容性由该 Firecracker runtime 提供。
      - 使用配置中的 `vm_state_path`、内存 overlay 镜像配置和
 `base_rootfs_path`。
      - 在 Firecracker 和 envd 就绪后返回运行中的 Sandbox。
    - `FirecrackerSandbox::from_snapshot(&RunnableSnapshot, &SandboxLaunchConfig)`
      - 从为节点解析的已提交 Snapshot 状态创建 Sandbox handle。
      - 使用 Snapshot `vm_state.bin` 加上节点本地物化的
 `memory/image.json` 和 `rootfs/image.json`。
      - 不启动 Firecracker；之后调用 `start().await`。
    - `start().await`
      - 生成 Firecracker 进程，等待 API 套接字，启动 VM，
 然后等待 guest 级 envd 就绪。
    - `pause().await`
      - 发送 `PATCH /vm` 暂停 VM。
      - 创建 Snapshot artifact，包括 `vm_state.bin`、`mem_image.json`、
 `mem_overlaybd/overlaybd.commit` 和 rootfs Snapshot 状态。
      - 返回一个拥有保存这些文件的 tempdir 的 `FirecrackerSnapshotConfig`。
    - `resume().await`
      - 通过发送带 `Resumed` 的 `PATCH /vm` 原地恢复暂停的 VM。
      - 当你希望保留同一 Sandbox 实例时使用。
    - `stop().await`
      - 停止 Firecracker，释放网络资源，并清理可选的 ublk 状态。
    - `run_command(cmd, args).await`
      - 通过 envd gRPC 在 Sandbox 内运行命令并等待其
 完成。返回包含 stdout、stderr 和退出码的 `ProcessOutput`。
    - `run_command_with_opts(cmd, args, opts).await`
      - 与 `run_command` 相同，但接受 `ProcessOpts` 以设置环境变量、
 工作目录和执行超时。
    - `start_process(cmd, args, opts).await`
      - 在 Sandbox 内启动长时间运行的进程。返回一个
 可流式输出、发送 stdin 或杀死进程的 `ProcessHandle`。
    - `firecracker_stdout_path()` / `firecracker_stderr_path()`
      - 此 Sandbox 使用的解析后日志路径（包括默认值）。
    - `work_rootfs_path()`
      - 返回工作目录内每实例可写 rootfs 的路径。
 用于在运行后检查或复制磁盘状态。

### 底层发生的事（对正确使用很重要）

- 每个 `FirecrackerSandbox` 使用自己的临时工作目录。
- 内核通过已配置的主机路径使用。
- 恢复时，可写 rootfs 和内存镜像会被物化到
 工作目录中，以便每个实例可以独立修改。对于由 overlaybd 支撑的
 Snapshot-backed Template 启动，这意味着从已提交 layer 堆栈重新创建 runtime 镜像配置和全新的
 可写 upper。
- Rootfs 副本在可用时使用 copy-on-write 辅助函数。
- Firecracker 以 `current_dir = work_dir` 运行，因此相对路径可用。
- Firecracker 日志默认禁用。设置 `firecracker.log_level` 以启用
 stdout/stderr 捕获和已配置 serial 目录下的 `firecracker.log`。
 直接 Rust 调用方也可以通过显式目标按流选择启用。
- `start()` 同时等待 API 套接字和 envd 就绪。
- 尽管 Firecracker 进程会在 drop 时被杀死，但建议显式调用 `stop()`。

### 最小用法示例

```rust
use agentenv::sandbox::{FirecrackerSandbox, FirecrackerSandboxConfig, SandboxExecutor};

let mut cfg = FirecrackerSandboxConfig::new(
    "/path/to/firecracker".into(),
    "/path/to/vmlinux.bin".into(),
    "/path/to/rootfs.ext4".into(),
);
cfg.boot_args = Some("console=ttyS0 reboot=k panic=1 pci=off init=/init".into());

let mut sandbox = FirecrackerSandbox::new(cfg)?;
sandbox.start().await?;

// Run a command inside the VM
let output = sandbox.run_command("echo", &["hello"]).await?;
assert_eq!(output.exit_code, 0);
println!("{}", output.stdout);

let snapshot = sandbox.pause().await?;
sandbox.stop().await?;

let mut resumed_sandbox = FirecrackerSandbox::resume_from_snapshot_config(&snapshot).await?;
resumed_sandbox.stop().await?;
```

### Runtime 前置条件

- Linux 主机
- 当前用户可访问 `/dev/kvm`，且主机模块与
 `virtualization_mode` 匹配（默认 KVM；PVM 需要 x86_64 和 `kvm_pvm`）
- 如果你使用 init 注入或磁盘检查辅助工具，则需要 `debugfs`（e2fsprogs）

## 2) 全局配置 Manager（`src/cfg.rs`）

测试和基准测试使用基于 `config/default.toml` 的共享全局配置 manager。

### 配置文件格式

`AgentENV/config/default.toml`（通过 `AENV_CONFIG_PATH` 或 `--config` 覆盖）

```toml
[firecracker]
boot_args = "your-kernel-boot-args"
binary_path = "/path/to/firecracker"
socket_timeout_secs = 3
socket_poll_ms = 20
# Optional override; defaults to $AENV_HOME/firecracker-work.
# work_dir = "/path/to/firecracker-work"
# Optional override; defaults to $AENV_HOME/logs/serial.
# serial_dir = "/path/to/serial"
# Optional; enables stdout/stderr capture and firecracker.log when non-empty.
# log_level = "Info"

[kernel]
image_path = "/path/to/vmlinux.bin"

[tools]
version = "0.1.0-custom.1"
drive_path = "/path/to/tools.ext4"

[machine]
mem_size_mib = 128
vcpu_count = 1

[envd]
version = "0.5.15"
init_timeout_secs = 30
poll_ms = 10

[orchestrator]
auto_evict_interval_ms = 1000
default_sandbox_timeout_secs = 15

[snapshot]
repository_backend = "posix_fs"
local_cache_path = "$AENV_HOME/snapshot-local-cache"

[backend.posix_fs]
snapshot_store = "$AENV_HOME/snapshot-store"

[ublk]
enabled = false
```

### TOML 参数参考

- `[firecracker]`
  - `boot_args`：内核命令行。如果尚未存在 `init=/init`，runtime 会追加它
 （tools 驱动器提供 init 脚本）。
  - `binary_path`：`firecracker` 的绝对或相对路径。
  - `socket_timeout_secs`：等待 Firecracker API 套接字的最大时间。
  - `socket_poll_ms`：检查套接字是否存在的轮询间隔。
  - `work_dir`：每-Sandbox Firecracker 工作目录的可选覆盖。
 这些目录包含 runtime 套接字、符号链接、本地
 日志和可写 OverlayBD upper layer 数据（`overlaybd/upper.data` 和
 `overlaybd/upper.index`）。默认 `$AENV_HOME/firecracker-work`。
  - `serial_dir`：持久化输出目录的可选覆盖。
 启用日志时，serial 输出写入每-Sandbox 子目录下，
 且不会被 Sandbox 移除。默认 `$AENV_HOME/logs/serial`。
 仅设置此路径不会启用日志。
  - `log_level`：可选的 Firecracker 日志级别（`Error`、`Warning`、`Info`、
 `Debug`、`Trace`，不区分大小写）。设置为非空值时，
 在 Sandbox 日志目录中启用 stdout/stderr 捕获和 `firecracker.log`。
 如果省略或为空，stdout/stderr 会被丢弃，且不会创建日志
 目录，除非设置了显式 Rust 捕获目标。

- `[kernel]`
  - `image_path`：可选的本地内核镜像路径（`vmlinux.bin`）。如果省略，
 AgentENV 从 `deps_path` 和捆绑的依赖 manifest 派生它。

- `[tools]`
  - `drive_path`：可选的本地 tools 驱动器源。设置时，需要显式 `version`，安装会将其导入不可变版本目录。

- Template rootfs 镜像
  - 用户可见的 rootfs 镜像在通过可选 `fromImage` 字段启动 Template 构建时选择。
  - 如果省略 `fromImage`，AgentENV 使用 `[image.resolver].default_image`。
  - 标准 Docker Hub 短名（如 `ubuntu:24.04` 和 `node:20`）
 在镜像解析之前会被归一化。
  - 解析后的镜像配置缓存在
 `<image-cache-root>/configs/<slug>-<hash>-image.json` 下；本地 layer
 产物缓存在 `<image-cache-root>/layers/` 下；overlaybd-native
 远程块读取缓存在 `<image-cache-root>/remote-blocks/` 下。
  - Registry 认证使用来自
 `~/.docker/config.json` 或 `$DOCKER_CONFIG/config.json` 的 docker 凭据。
- `[machine]`
  - `mem_size_mib`：Guest RAM 大小（默认：128）。
  - `vcpu_count`：vCPU 数量（默认：1）。

- `[envd]`
  - `version`：烘焙进 runtime 镜像的期望 envd 版本。
  - `init_timeout_secs`：VM 启动后等待 guest 内 envd daemon 就绪的最大时间
 （默认：30）。
  - `poll_ms`：envd 健康检查重试的轮询间隔（默认：10）。

- `[orchestrator]`
  - `auto_evict_interval_ms`：后台超时驱逐的轮询间隔。
  - `default_sandbox_timeout_secs`：Orchestrator 使用的默认 keep-alive 超时。
  - `auto_resume_min_sandbox_timeout_secs`：当数据平面请求针对非运行 Sandbox 时，自动恢复它（如果启用了自动恢复）并刷新其超时不少于该时长。

- `[pool]`
  - `low_watermark`：warm 资源的共享下界。默认 `2`。
  - `high_watermark`：warm 资源的共享上界目标。默认 `64`。
  - `[pool.network].maintenance_enabled`：启用后台网络槽位维护 worker。默认 `true`。
  - `[pool.block].enabled`：启用 ublk overlaybd warm 设备池。
  - `[pool.firecracker].enabled`：启用用于 Snapshot 恢复的预生成 Firecracker 进程。

- `[snapshot]`
  - `local_cache_path`：Manager 拥有的节点本地 Snapshot artifact/缓存根。默认 `$AENV_HOME/snapshot-local-cache`。
  - `repository_backend`：Snapshot repository 后端。默认 `posix_fs`。

- `[backend.posix_fs]`
  - `snapshot_store`：已提交 Snapshot 存储根。默认 `$AENV_HOME/snapshot-store`。

- `[ublk]`
  - `daemon_binary_path`：`uvm-ublk-daemon` 的可选路径。
  - `daemon_socket_path`：daemon RPC 的可选 unix 套接字路径。
  - `daemon_log_path`：daemon 进程的可选日志文件路径。
- `[ublk.overlaybd]`
  - `global_config_path`：生成的 overlaybd runtime 配置路径。
 ublk 需要。每镜像配置从
 Template 构建的 `fromImage` 值派生，位于
 `<image.cache.root_dir>/configs` 下。

### 默认值与可选字段

- `[firecracker].binary_path`：可选。如果省略，AgentENV 从 `deps_path` 和捆绑的依赖 manifest 派生它。
- `[firecracker].boot_args`：可选。默认 `console=ttyS0 reboot=k panic=1 pci=off`。
- `[firecracker].socket_timeout_secs`：可选。默认 3 秒。
- `[firecracker].socket_poll_ms`：可选。默认 20 ms。
- `[firecracker].work_dir`：可选覆盖。默认 `$AENV_HOME/firecracker-work`。
- `[firecracker].serial_dir`：可选覆盖。默认 `$AENV_HOME/logs/serial`。
- `[kernel].image_path`：可选。如果省略，AgentENV 从 `deps_path` 和捆绑的依赖 manifest 派生它。
- `[tools].drive_path`：可选的本地源覆盖；设置时需要显式 `[tools].version`。
- `[tools].url`：可选的 registry 覆盖；设置时需要显式 `[tools].version`。
- `[machine]`：可选。如果省略，默认 128 MiB RAM 和 1 vCPU。
- `[envd]`：可选，但 Template 和 Sandbox 流程期望有效的 `version`。
- `[envd].init_timeout_secs`：可选。默认 30 秒。
- `[envd].poll_ms`：可选。默认 10 ms。
- `[orchestrator]`：可选。如果省略，使用 Orchestrator runtime 默认值。
- `[pool]`：可选。如果省略，网络池维护启用，低水位 2、高水位 64；块和 Firecracker 进程池除非在其组件子节中启用，否则禁用。
- `[snapshot]`：可选。如果省略，`repository_backend` 默认 `posix_fs`，本地缓存根从 `AENV_HOME` 派生。
- `[backend.posix_fs]`：可选。如果省略，POSIX Snapshot 存储默认 `$AENV_HOME/snapshot-store`。
- `[ublk]`：可选。如果省略，ublk 禁用。

### 基于环境的配置

配置加载器从以下来源读取完整 TOML 文件：

- `AENV_CONFIG_PATH`
- `--config <path>`
- `config/default.toml`

Runtime 镜像和二进制路径在 TOML 文件本身中配置。

### 必需字段

- `[kernel].image_path`
- 当覆盖 `[tools].drive_path` 或 `[tools].url` 时，需要 `[tools].version`
- `[envd].version`

如果存在 `[orchestrator]` 节，它必须包含
`default_sandbox_timeout_secs`。

### ConfigManager（推荐入口点）

该辅助函数是严格的。如果不满足要求（文件缺失、配置无效），
它会返回错误且测试失败。

**路径解析顺序**
1) `AENV_CONFIG_PATH` 环境变量
2) `--config` CLI 标志
3) `config/default.toml`（默认）

### 辅助方法

- `ConfigManager::global()` -> `Result<&'static ConfigManager>`
  - 惰性初始化并返回进程范围单例。
- `ConfigManager::new()` -> `Result<ConfigManager>`
  - 使用默认解析顺序加载配置。
- `ConfigManager::new_from_path(path)` -> `Result<ConfigManager>`
  - 加载特定 TOML 文件。
- `ConfigManager::config()` -> `&AppConfig`
  - 返回解析后的应用配置。
- `ConfigManager::global_config()` -> `Result<&'static AppConfig>`
  - 直接返回全局解析配置。
- `ConfigManager::get_orchestrator_config()` -> `Option<OrchestratorConfig>`
  - 配置时返回 Orchestrator 设置。
- `AppConfig::resolved_snapshot_store()` -> `PathBuf`
  - 解析有效的已提交 Snapshot 存储根。
- `AppConfig::resolved_snapshot_local_cache_path()` -> `PathBuf`
  - 解析有效的 manager 拥有的节点本地 Snapshot 缓存根。

### 典型测试设置流程

```rust
use agentenv::sandbox::{FirecrackerSandbox, FirecrackerSandboxConfig, SandboxExecutor};

let sandbox_config = FirecrackerSandboxConfig::from_global_config()?;

let mut sandbox = FirecrackerSandbox::new(sandbox_config)?;
sandbox.start().await?;

// Run commands via envd gRPC
let output = sandbox.run_command("echo", &["hello"]).await?;
assert_eq!(output.exit_code, 0);

let snapshot = sandbox.pause().await?;
sandbox.stop().await?;

let mut resumed_sandbox = FirecrackerSandbox::resume_from_snapshot_config(&snapshot).await?;
resumed_sandbox.stop().await?;
```

## 3) 集成测试验证的内容

### `integration/fc.rs`

- `microvm_startup_moves_to_running`
 启动全新 VM 并验证：
  - Firecracker 启动并接受 API 调用。
  - 可以无错误地配置 kernel + rootfs。
  - VM 之后可以干净地关闭。

- `microvm_pause_transitions_to_paused`
 通过以下方式验证暂停 + Snapshot 创建：
  - 通过 FC API 暂停运行中的 VM。
  - 创建 `vm_state.bin`、`mem_image.json` 和
 `mem_overlaybd/overlaybd.commit`。
  - 确保这些文件存在于磁盘上。

- `microvm_resume_from_pause_transitions_to_running`
 通过以下方式验证原地恢复：
  - 暂停 VM。
  - 在同一 Sandbox 实例中恢复它。
  - 确认控制流无错误完成。

- `microvm_resume_transitions_to_running`
 通过以下方式验证恢复到新 Sandbox：
  - 暂停并快照 VM。
  - 关闭原始 Sandbox。
  - 在新 Sandbox 实例中从 Snapshot 恢复。

- `snapshot_preserves_disk_state_on_resume`
 通过以下方式确认磁盘状态被保留：
  - 通过 `run_command` 写入标记文件。
  - 暂停并从 Snapshot 恢复。
  - 恢复后读取标记文件并断言其持久存在。

- `snapshot_chain_survives_after_parent_snapshot_handle_is_dropped`
 通过以下方式验证多级 Snapshot 链：
  - 创建 Snapshot、恢复、创建第二个 Snapshot。
  - 丢弃第一个 Snapshot handle。
  - 从第二个 Snapshot 恢复并验证磁盘状态。
  - 还测试带 Snapshot 拥有的继承 layer 收养的 `pause_to_dir`。

- `microvm_can_access_internet`
 确认 guest 网络在 Snapshot 恢复前后都能工作。

- `multiple_resumes_have_independent_disk_state`
 通过以下方式确认多次恢复是隔离的：
  - 从单个 Snapshot 顺序恢复多个 VM。
  - 每个恢复的 VM 验证共享标记并写入自己的文件。
  - 验证每个 Snapshot 具有独立状态。

### `integration/process.rs`

- `run_command_captures_stdout`
 运行 `echo hello world` 并验证 stdout 包含预期字符串。

- `run_command_captures_stderr`
 运行一个写入 stderr 的 shell 命令并验证 stderr 捕获。

- `run_command_reports_exit_code`
 运行 `exit 42` 并验证退出码被正确报告。

- `run_command_with_opts_sets_env_and_cwd`
 验证环境变量和工作目录选项被遵守。

- `run_command_handles_timeout`
 验证长时间运行的命令在配置超时时失败。

- `run_command_with_unbounded_output`
 验证过量输出被拒绝，而不是无限缓冲。

- `start_process_interactive_stdin`
 启动 `cat`，通过 stdin 发送数据，杀死进程，并验证回显
 输出。

- `start_process_send_signal`
 启动 `sleep 300`，发送 SIGTERM，并验证进程以
 非零退出码终止。

- `run_command_after_snapshot_resume`
 暂停 Sandbox，从 Snapshot 配置恢复，并验证 `run_command` 在
 恢复的实例上工作。

### `integration/snapshot.rs`

- `built_snapshot_can_be_loaded_by_alias_and_launched`
 构建 Template 支撑的 Snapshot，通过别名加载它，从
 解析出的 runnable Snapshot 启动 Sandbox，并验证捕获的 runtime 状态。

- `listed_committed_snapshot_can_be_loaded_and_resolved`
 验证 `SnapshotManager::list_committed()` 返回可用的已提交 Snapshot，
 这些 Snapshot 可以加载并解析为可运行路径。

- `loaded_runnable_snapshot_by_alias_can_be_resolved`
 验证 `SnapshotManager::load_runnable()` 一步将基于别名的查找解析为
 可运行 Snapshot 状态。

- `delete_committed_snapshot_by_alias_removes_it_from_repository`
 验证通过别名删除已提交 Snapshot 会移除持久化记录。

- `snapshot_rebuild_from_committed_snapshot_preserves_base_state`
 验证从已提交 Snapshot 重建会保留基础文件系统
 状态，同时添加新更改。

- `rebuild_merges_env_metadata_and_list_filter_can_select_by_id`
 验证重建正确合并环境元数据，并且已提交
 Snapshot 列表可以按 Snapshot id 过滤。

### `integration/orchestrator.rs`

- `orchestrator_lifecycle`
 验证 Orchestrator 可以创建、获取、列出、过滤、保活和
 删除 Template 支撑的 Sandbox，同时保持代理查找状态同步。

## 4) 端到端执行清单

1. 提供 Firecracker 二进制、内核、tools 驱动器、overlaybd runtime 和 ublk daemon。主机依赖在服务器启动时下载，或使用 `cargo run --bin server -- --setup-only` 下载。工具在启动或恢复时使用记录的 tools 版本准备。
2. 以 root 身份一次性配置主机访问：`server --setup-host --runtime-user
 <user> --runtime-group <group>`。该组是 runtime 服务组：它
 拥有 AgentENV 状态并接收 ublk 设备访问权限。正常服务器启动
 仅执行验证，绝不调用 `sudo`。
3. 主机安装会为 `/dev/ublk-control`、`/dev/ublkc*` 和 `/dev/ublkb*` 安装 udev 规则，使 runtime 组可以访问控制节点和动态设备节点。同一规则将每个 `/dev/ublkb*` 的 I/O 调度器设置为 `none`。
4. 更新 `config/default.toml` 路径，或将 `AENV_CONFIG_PATH` 指向自定义配置文件。
5. 确保 runtime 用户可访问 `/dev/kvm`，且已配置的
 虚拟化模式与主机模块匹配。
6. 如果运行 Template 测试，确保主机可以运行 `regctl`（服务器安装会从 `[regclient]` manifest 条目自动安装它）并访问 registry 以进行 Template `fromImage` 解析。
7. 运行 `scripts/tests/e2e/run_e2e.sh` 进行 API 级 E2E 覆盖。运行器导出 `E2E_TEMPLATE_USER_IMAGE`。套件 `05_template_lifecycle.sh` 还使用 `E2E_SHORT_USER_IMAGE` 创建 Template 构建，以验证短名镜像解析。
8. 运行 `make test-agent-integration`，以非 root 用户和所需 capabilities 运行 `tests/integration/` 中的 `agentenv` 集成测试模块，以及 Docker/MinIO 支撑的 OSS Snapshot repository 测试（`crates/e2e-tests/tests/snapshot_oss_e2e_test.rs`）。
9. 运行 `make test-ublk` 运行 `uvm-ublk` 和 `overlaybd` 存储测试，包括 Docker/MinIO 支撑的 OSS 后端测试（`storage/overlaybd/tests/oss_backend_minio.rs`）。

## 4.1) Firecracker Client 与 Runtime 升级清单

当将 AgentENV 升级到打过补丁的 Firecracker runtime 时，同时更新生成的
client API 和安装时下载的二进制：

1. 将 `thirdparty/firecracker-client/firecracker.yaml` 替换为目标
 Firecracker OpenAPI spec。
2. 从 repository 根运行 `make firecracker-client` 并审查
 `thirdparty/firecracker-client/` 下生成的更改。
3. 在 Linux 主机上构建打过补丁的 Firecracker release 二进制。将其打包为
 名为 `firecracker-{version}-{arch}.tgz` 的 gzip tar 归档，其中包含
 `firecracker` 可执行文件。
4. 使用新版本和支持 `{version}` 与 `{arch}` 的下载 URL 模板
 更新 `config/deps_manifest.toml` 中的 `[firecracker]` 条目。
5. 在测试中依赖该 URL 之前，验证它是直接下载：

   ```bash
   curl -L "<url>" -o /tmp/firecracker.tgz
   tar -tzf /tmp/firecracker.tgz
   ```

 归档列表应包含 `firecracker`。
6. 对干净或临时 Firecracker 依赖目录运行安装，以便它
 真正下载归档：

   ```bash
   cargo run --bin server -- --config /path/to/config.toml --setup-only
   ```

7. 代码生成后验证 Rust 侧：

   ```bash
   cargo fmt --check -p firecracker_client -p agentenv
   cargo check -p firecracker_client
   cargo check -p agentenv
   cargo test -p agentenv --lib
   ```

## 4.2) 嵌套 KVM 验证

`scripts/tests/nestedvirt-smoke.py` 构建测试镜像，并在 x86_64 Sandbox 内运行 QEMU，
演练 Template 恢复、fork 和暂停/恢复。
在更改嵌套 KVM runtime、Snapshot 处理或 guest 内核时运行它。

1. 使用具有嵌套 KVM 的 x86_64 主机，以及内置 KVM 的外部 guest 内核。
 安装 Python 3.11+、QEMU x86 系统二进制、位于
 `/bin/busybox` 的静态 BusyBox、cpio 和 Bash。为内部 guest 提供可启动的 bzImage，
 以及至少有 5 GiB 空闲空间的干净工作目录。
2. 使用上述主机安装准备专用 AgentENV 节点。使用单独的
 `AENV_HOME_PATH`、`AENV_RUNTIME_PATH` 和网络池；使用 `AENV_DEPS_PATH` 选择测试资产。
 将 API 绑定到 loopback，并设置
 `[sandbox_proxy].domains`，例如设置为 `["nested.local"]`。
3. 保留完整的 `[firecracker].boot_args` 字符串并追加 `no-kvmapf`；
 运行器会检查该参数。设置 `[pool.block].enabled = true`。
 为运行器的 loopback registry 配置 regctl，使用
 `--host reg=127.0.0.1:18091,tls=disabled`；将端口与 `--port` 匹配。
4. 启动节点，然后使用 `python3 scripts/tests/nestedvirt-smoke.py run --help`
 查看 SDK 安装、参数和完整调用方式。通过 `--config` 传入节点的
 TOML 文件，包括其有效启动参数。
5. 要求成功退出，并在
 `<work>/evidence/api-results.json` 中为每个选定阶段和轮次生成一个结果（默认九个）。
 Guest 命令和输出位于 `api-commands.jsonl`。运行后停止测试节点。

运行器构建一个 Template，然后为每个用例创建一个源 Sandbox 和两个 fork
子 Sandbox。Guest payload `nestedvirt-guest.sh` 使用 `accel=kvm` 启动和停止 QEMU，并提供一个
内部 guest 计数器。

| 阶段 | 恢复检查 |
|-------|----------------|
| `unused` | fork 和暂停/恢复后，新的内部 VM 启动。 |
| `initialized` | 使用 `-S` 创建的 QEMU 保留启用 KVM 的 `prelaunch` 状态。 |
| `running` | fork 和暂停/恢复后，内部计数器继续递增。 |

每个用例还会在源 Sandbox 和两个子 Sandbox 中启动和停止一个新的内部 VM；
initialized 用例会替换其停止的内部 VM。现有 Snapshot 保留其捕获的内核和启动参数，因此使用新 Template 测试对任一者的更改。托管 CI 运行语法和 `check-fixture` 检查；生命周期矩阵在嵌套 KVM 硬件上运行。

## 4.3) 旧版 v9 Snapshot 恢复

`tests/integration/orchestrator.rs` 中的 `orchestrator_v9_snapshot_upgrade` 使用旧版和当前 Firecracker 进程运行
当前 AgentENV 测试二进制。它
保存 v9 状态，使记录的可执行文件路径无效，使用
配置的 runtime 恢复，检查 envd，并再次保存为 v1000。

1. 准备 `v9-node.toml` 和 `current-node.toml`，包含各自的 runtime
 和匹配的辅助工具。在两个配置中使用绝对路径以及相同的 state、storage、
 network、guest 内核和启动参数。
2. 完成上述集成主机设置，并使用
 `make build-ublk PROFILE=debug` 构建 daemon。
3. 显式运行该测试；默认忽略，因为它需要两个
 runtime 版本：

   ```bash
   AENV_HOME_PATH=/absolute/path/isolated-test/home \
   AENV_RUNTIME_PATH=/absolute/path/isolated-test/run \
   AENV_DEPS_PATH=/absolute/path/test-deps \
   AENV_UBLK_DAEMON_BINARY_PATH="$PWD/target/debug/uvm-ublk-daemon" \
   AENV_CONFIG_PATH=/absolute/path/current-node.toml \
   AENV_TEST_V9_CONFIG=/absolute/path/v9-node.toml \
   scripts/run-with-capabilities.sh cargo test -p agentenv \
     --test orchestrator_integration orchestrator::orchestrator_v9_snapshot_upgrade \
     -- --exact --ignored --nocapture
   ```

对于服务升级验证，使用旧 AgentENV 二进制创建 Template 和暂停记录，停止它，
针对相同状态启动新二进制，并验证恢复、fork 以及另一个暂停/恢复周期。

## 5) 运行自定义脚本

### 通过 envd 运行命令

Sandbox 启动后（rootfs 中包含 envd daemon），你可以
使用进程 API 在 VM 内执行命令：

```rust
// Simple command
let output = sandbox.run_command("ls", &["-la", "/tmp"]).await?;
println!("exit={} stdout={}", output.exit_code, output.stdout);

// Command with options
use agentenv::sandbox::ProcessOpts;
let opts = ProcessOpts::new()
    .with_cwd("/tmp")
    .with_envs([("KEY".into(), "value".into())].into());
let output = sandbox.run_command_with_opts("env", &[], opts).await?;

// Interactive / long-running process
let mut handle = sandbox.start_process("cat", &[], ProcessOpts::default()).await?;
handle.send_stdin(b"hello\n").await?;
handle.kill().await?;
let output = handle.wait().await?;
```

进程 API 底层使用 envd 的 gRPC `ProcessClient`。它要求
guest VM 内运行 envd daemon。

### 为多次 Snapshot 恢复运行不同脚本

恢复后 `/init` 不会重新执行，因此每实例行为必须来自
每实例磁盘内容。

典型模式：
1) 启动基础 VM 并 `pause()` 创建 Snapshot。
2) 对于每个实例，复制 Snapshot rootfs 并写入一个每实例文件
 （例如使用 `debugfs`）。
3) 创建一个指向该每实例 rootfs 的 `FirecrackerSnapshotConfig` 并恢复。

Guest 必须保留一个长期运行的 Scheduler（在初始
启动期间启动），它监视这些每实例文件并在恢复后执行它们。

## 6) 全新 Sandbox 吞吐量基准

`snapshot` 基准有两个基础镜像启动测量：

- `concurrent_fresh_start_throughput`：计时 `start()` 直到 guest 就绪；
 计时后停止 Sandbox。
- `concurrent_fresh_start_stop_throughput`：同时计时 `start()` 和 `stop()`。

每批使用单独的 Sandbox，并报告每秒完成的启动数。
镜像解析和 Sandbox 构造发生在计时之前。这测量
Sandbox 后端，而非 HTTP 或 Scheduler 请求。全新启动并不
一定意味着冷 Firecracker 进程池。

在具有 KVM 和 ublk 的 Linux 主机上，从 repository 根运行。在运行基准之前
一次性设置依赖：

```bash
export AENV_HOME_PATH="$HOME/.local/share/agentenv-fresh-bench/home"
export AENV_RUNTIME_PATH="$HOME/.local/share/agentenv-fresh-bench/run"
export AENV_FIRECRACKER_WORK_DIR="/tmp/aenv-fresh-fc-$(id -u)"
export AENV_DEPS_PATH="$HOME/.local/share/aenv/deps"
export AENV_UBLK_DAEMON_BINARY_PATH="$AENV_HOME_PATH/ublk/uvm-ublk-daemon"
make install-ublk PROFILE=release AENV_INSTALL_SUDO=
cargo run --bin server -- --setup-only
AENV_BENCH_CONCURRENCY=2 \
  CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_RUNNER="$PWD/scripts/run-with-capabilities.sh" \
  cargo bench -p agentenv-benchmarks --bench snapshot -- fresh_start_
```

过滤器会运行两个测量。默认运行器采用有界采样；
设置 `AENV_BENCH_FULL=1` 以使用 Criterion。`AENV_BENCH_CONCURRENCY` 默认是
可用 CPU 并行度的两倍，最多 50。失败的操作会使该批失败；
基准会尝试停止任何剩余的 Sandbox 并报告清理
错误。
