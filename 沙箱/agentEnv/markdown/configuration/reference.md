# 运行时节点配置

AgentENV 从 TOML 文件读取配置。默认路径为 `config/default.toml`。可通过以下方式覆盖：

```bash
export AENV_CONFIG_PATH=/path/to/config.toml
# or
cargo run --bin server -- --config /path/to/config.toml
```

## 全局设置

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `home_path` | string | `"/var/lib/aenv"` | 本地 AgentENV 状态的基础目录。由 `AENV_HOME_PATH` 覆盖 |
| `runtime_path` | string | `"/run/aenv"` | 临时命名空间和守护进程套接字状态的基础目录。由 `AENV_RUNTIME_PATH` 覆盖 |
| `deps_path` | string | `"$AENV_HOME/deps"` | 自动下载的运行时资产的根目录。由 `AENV_DEPS_PATH` 覆盖 |
| `virtualization_mode` | `"kvm"` 或 `"pvm"` | `"kvm"` | 此节点的虚拟化模式。除非遵循 [PVM 部署](../deployment/pvm.md) 指南，否则保持默认值。由 `AENV_VIRTUALIZATION_MODE` 覆盖 |

快照和暂停的沙盒只能在其创建时的模式下恢复。

`$AENV_HOME` 是状态路径值中的字面占位符，而非 shell 环境变量。AgentENV 在应用 `AENV_HOME_PATH` 后用解析后的 `home_path` 替换它；`ublk.daemon_socket_path` 还支持 `$AENV_RUNTIME`，它解析为 `runtime_path`。不含这些占位符的相对路径会相对于配置文件所在目录解析。

打包的运行时依赖版本和下载 URL 位于 `config/deps_manifest.toml`。只安装所选模式的依赖。用户配置应包含运行时行为和显式的本地路径覆盖，而非默认依赖目录。

## `[firecracker]`

Firecracker VM 二进制和启动配置。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `version` | string | manifest 值 | 用于自动下载的可选 Firecracker 发行版覆盖 |
| `url` | string | manifest 值 | 可选的下载 URL 模板覆盖，含 `{version}` 和 `{arch}` 占位符 |
| `binary_path` | string | 由 manifest/config 版本派生 | 指向本地 `firecracker` 二进制的显式路径。设置后跳过 Firecracker 下载，并要求其为可读、非空的可执行普通文件 |
| `boot_args` | string | `"console=ttyS0 reboot=k panic=1 pci=off init=/init …"` | 完整的内核命令行。随附的默认值还包含 DAMON 内存回收和 free-page-reporting 参数；完整值见 `config/default.toml`。 |
| `allowed_extra_boot_args_prefixes` | 字符串数组 | `[]` | 冷启动 Sandbox 上 `extraBootArgs` 的允许前缀。为空时不追加任何请求提供的额外启动参数 |
| `socket_timeout_secs` | integer | `3` | 等待 Firecracker API 套接字的最大秒数 |
| `socket_poll_ms` | integer | `1` | 检查套接字可用性的轮询间隔（ms） |
| `work_dir` | string | `"$AENV_HOME/firecracker-work"` | 每 Sandbox Firecracker 工作目录的父目录。这些目录包含运行时套接字、符号链接、本地日志以及可写的 OverlayBD 上层数据，如 `overlaybd/upper.data` 和 `overlaybd/upper.index` |
| `serial_dir` | string | `"$AENV_HOME/logs/serial"` | 启用时持久化 Firecracker 日志的目录（每 Sandbox 子目录）。仅设置此路径不会启用日志记录 |
| `log_level` | string | 未设置（禁用） | 可选的 Firecracker 日志级别（`Error`、`Warning`、`Info`、`Debug`、`Trace`，不区分大小写）。非空值会在每个 Sandbox 的日志目录中启用 `firecracker.log` 以及 stdout/stderr 捕获。为空/未设置时丢弃 stdout/stderr，不创建日志文件或每 Sandbox 日志目录。显式的 Rust stdout/stderr 目标仍会启用所请求的流 |

## `[kernel]`

用于 microVM 的 Linux 内核镜像。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `version` | string | manifest 值 | 用于自动下载的可选内核版本覆盖 |
| `url` | string | manifest 值 | 可选的下载 URL Template 覆盖，含 `{version}` 占位符 |
| `image_path` | string | 由 manifest/config 版本派生 | 指向本地 `vmlinux.bin` 的显式路径。设置后跳过内核下载，并要求其为可读、非空的普通文件 |

## `[overlaybd]`

用于覆盖 setup 下载的 OverlayBD 工具发行版的可选配置。省略该节时，使用 `config/deps_manifest.toml` 中的发行版。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `version` | string | manifest 值 | OverlayBD 工具发行版；设置该节时必填 |
| `package_url` | string | manifest 值 | 工具归档的下载 URL Template，含 `{version}` 和 `{arch}` 占位符 |
| `url` | string | 未设置 | `package_url` 的回退值 |

## `[tools]`

用于在每个 microVM 内启动 AgentENV 控制平面的 tools 驱动镜像。默认发行版会自动选择；大多数部署可以保持此节不变。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `version` | string | manifest 值 | 完整 tools 驱动的不可变 SemVer 发行版；自定义发行版应使用唯一的预发布版本，如 `0.1.0-custom.1` |
| `url` | string | manifest 值 | 可选的 OCI 镜像 URL Template 覆盖，含 `{version}` 占位符；设置时要求显式的 `version` |
| `drive_path` | string | 未设置 | 导入到版本化依赖目录的本地 tools ext4 源；要求显式的 `version` |
| `control_plane_port` | integer | `49983` | 客户机内 envd 使用的端口 |

Snapshot 和暂停的 Sandbox 只记录 `tools_drive_version`。启动和恢复时，如果本地缺少该版本，会从配置的 URL Template 获取。发行版必须不可变，并在其 Snapshot 的整个生命周期内保留；更改默认版本会影响新 Sandbox 和 Template。

OCI tools rootfs 镜像带有 `io.agentenv.tools-drive.format=oci-rootfs-v1`（tools Dockerfile 会自动添加它）。它们会在本地通过稳定、固定的转换契约下载并转换。转换后的层存放在版本化依赖目录中，独立于镜像缓存淘汰。显式的 `--setup-only` 会为依赖包准备默认发行版。

原生 OverlayBD 镜像保留惰性读取，并在 envd 就绪后自动在后台下载 tools 层，独立于内存和用户磁盘。不带格式标签的旧镜像提供 `tools.ext4`，它会被下载并以只读方式挂载。Tools 不会上传到 Snapshot 存储。本地 `drive_path` 仅标识其声明的版本。

## Template Rootfs 镜像

用户可见的 rootfs 镜像在 Template API 层选择。`POST /v2/templates/{templateID}/builds/{buildID}` 接受可选的 `fromImage` 字段：

- 省略：使用 `[image.resolver].default_image`
- 完整 OCI 引用：使用提供的镜像
- 短名称：规范化标准 Docker Hub 形式，如 `ubuntu:24.04` 和 `node:20`

## `[image.resolver]`

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `default_image` | string | `ubuntu:24.04` | Template 构建省略 `fromImage` 时使用的镜像 |
| `convert_standard_oci` | boolean | `true` | 将标准 OCI tar 层转换为本地 OverlayBD 提交。设置为 `false` 以要求 OverlayBD 原生镜像或 referrers，否则快速失败 |
| `search_registries` | 字符串数组 | `["docker.io", "ghcr.io"]` | 解析短镜像引用时尝试的 registry |
| `allowed_registries` | 字符串数组 | 未设置（无限制） | registry 主机白名单（例如 `docker.io`、`registry.example.com:5000`）。**省略该键**不施加限制；**显式空列表 `[]`** 拒绝所有 registry。设置为非空列表时，只有 registry 主机在列表中的引用才能解析；其他主机将以客户端（4xx、`ImageReferenceError`）错误被拒绝。参见下文*三个 registry 设置如何交互*。 |
| `try_referrers_overlaybd_prefixes` | 字符串数组 | `[]` | 镜像引用前缀，对于这些前缀，AgentENV 会在本地转换标准 OCI 镜像之前，通过 `regctl` 尝试 OCI Referrers API 以获取 overlaybd 原生 artifact。前缀使用简单的 `starts_with` 匹配；请自行包含末尾斜杠，例如 `registry.example.com/` 或 `registry.example.com/team/`。要求 `PATH` 中存在 `regctl`；查找失败会回退到源镜像。 |

### 三个 registry 设置如何交互

镜像解析分两个阶段运行，三个键在不同环节起作用：

1. **`search_registries` — 补全。** 仅用于*短/未限定*的引用（例如 `ubuntu`）。每个条目会作为前缀添加到名称前，以构建完全限定候选列表（`docker.io/library/ubuntu:latest`、`ghcr.io/ubuntu:latest` 等）。完全限定的引用会跳过此步骤。
2. **`allowed_registries` — 门控。** 在构建候选后立即应用，*同时*作用于完全限定的引用和由 `search_registries` 展开的候选。registry 主机不在白名单中的候选会被丢弃；如果没有剩余候选，该引用会以 4xx 错误被拒绝。实际上，可解析的短名称主机集合是 `search_registries` 与 `allowed_registries` 的**交集**——例如，搜索 `docker.io` 和 `ghcr.io` 但仅允许 `ghcr.io` 时，短名称只解析为 `ghcr.io`。
3. **`try_referrers_overlaybd_prefixes` — 逐候选优化。** 稍后运行，在解析*已被允许*的候选时：获取其 manifest 后，AgentENV 可能在**同一** registry/仓库上查询 OCI Referrers API 以获取 overlaybd 原生 artifact。由于 referrer 查找从不离开源镜像自身主机，它们隐式受 `allowed_registries` 覆盖——referrers 无需单独的白名单条目。

 按此顺序识别两种 referrer `artifactType`：

 | `artifactType` | 产生方 |
 |-----|-----|
 | `application/vnd.containerd.overlaybd.native.v1+json` | accelerated-container-image（`obdconv`） |
 | `application/vnd.azure.artifact.streaming.v1` | Azure Container Registry artifact streaming（`az acr artifact-streaming create`） |

 两者都指向 overlaybd 原生 manifest；只有发现标签不同。referrer manifest 在获取后会重新验证，因此实际上并非 overlaybd 原生的 referrer 会被拒绝而不是被使用。Turbo-OCI referrers（`application/vnd.containerd.overlaybd.turbo.v1+json`）永远不会被选中——AgentENV 的 overlaybd 运行时未实现 turbo 读取路径。

 要从 ACR 流式传输，请将你的 registry（带末尾斜杠）加入列表，例如 `try_referrers_overlaybd_prefixes = ["myregistry.azurecr.io/"]`。

## `[image.cache]`

已解析和已转换用户镜像的节点本地缓存根目录。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `root_dir` | string | `"$AENV_HOME/image-cache"` | AgentENV 镜像缓存 artifact 的根目录 |
| `capacity_gb` | integer | `100` | 用于容量驱动的本地提交字节淘汰的预算。仅在 `[image.cache.gc].enabled` 为 `true` 时生效：后台 GC 在使用量超过高水位后淘汰最近最少使用的源配置，直至低水位。未设置 = 无容量上限。 |

## `[image.cache.gc]`

镜像缓存的后台硬提交垃圾回收。启用后，每一轮会协调磁盘上源配置的元数据，然后删除不再被源配置根植、不再被镜像缓存租约持有或不再被进程内运行集引用的硬提交对象。已提交的 Snapshot 是持久仓库状态，不会固定 ImageCache 提交。设置 `capacity_gb` 后，GC 会先淘汰超过高水位的最近最少使用源配置，以便硬提交 GC 回收它们解除根植的对象。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `enabled` | bool | `true` | 启用后台镜像缓存 GC 任务 |
| `interval_secs` | integer | `1800` | GC 轮次之间的秒数（值 `<= 0` 会回退到默认值） |
| `min_age_secs` | integer | `600` | 源配置可被容量淘汰前自上次使用以来的最短时间（LRU 下限） |
| `high_watermark_ratio` | float | `0.95` | 本地提交字节超过 `capacity_gb` × 该比例后开始容量淘汰。限制在 `(0, 1]` |
| `low_watermark_ratio` | float | `0.70` | 高水位触发后淘汰至 `capacity_gb` × 该比例。限制在 `(0, high_watermark_ratio]` |

容量驱动的淘汰仅在设置了 `[image.cache].capacity_gb` 时运行；否则 GC 仍会回收不可达提交，但不执行水位淘汰。

## `[image.cache.remote_blocks]`

Overlaybd registryfs_v2 远程块缓存设置。目录始终为 `<image.cache.root_dir>/remote-blocks`。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `max_size_gb` | integer | `100` | overlaybd 远程块缓存的最大大小，以 GiB 计。此值会写入生成的 overlaybd `cacheConfig.cacheSizeGB` |
| `open_evict_idle_secs` | integer | `600` | 空闲时长（秒），超过后由运行中 Sandbox 持有的缓存条目的缓存块可能在容量压力下被回收。条目会保留并按需重新填充；`0` 禁用打开条目淘汰。写入生成的 overlaybd `cacheConfig.openEvictIdleSecs`，同时适用于 `remote-blocks` 和 `memory-blocks` 缓存 |

已解析的镜像数据缓存于：

```text
<image.cache.root_dir>/
  commits/
    <sha256-commit-digest>/
      overlaybd.commit
                  # full overlaybd commit store shared by OCI conversion and download
  indexes/        # OCI layer + conversion context -> overlaybd commit descriptor
  remote-blocks/  # overlaybd-native remote block cache
  configs/         # resolved image configs
    <slug>-<hash>-image.json
```

## `[sandbox_proxy]`

Sandbox 服务的可选基于主机的数据平面路由。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `domains` | 字符串数组 | `[]` | 服务器接受的 DNS 域名，用于形如 `{port}-{sandboxID}.{domain}` 的基于主机的代理 URL。第一个配置的域名会在 Sandbox 创建/详情响应中作为 `domain` 返回。 |

当 `domains` 为空时，服务器仍支持 `/proxy` 和路由头代理请求，但不按 `Host` 对请求分类。域名会规范化为小写、去重，且必须是合法的 DNS 名称。由于 `domains[0]` 是公布的 Sandbox 域名，配置顺序会被保留。

环境变量覆盖：

- `AENV_SANDBOX_PROXY_DOMAINS`

## `[network.egress]`

节点级 Sandbox 出口护栏。这些规则会在每 Sandbox 的 `allowOut` / `denyOut` 规则之前安装，因此 Sandbox API 请求无法覆盖它们。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `always_denied_cidrs` | IPv4 CIDR 字符串数组 | `["10.0.0.0/8", "100.64.0.0/10", "127.0.0.0/8", "169.254.0.0/16", "172.16.0.0/12", "192.168.0.0/16"]` | 在评估用户出口策略之前，始终从 Sandbox 拒绝的目标 CIDR。当 Sandbox 需要向这些目标出口时，部署可以移除选定的 RFC1918 范围。 |

## `[network.internal]`

AgentENV 内部 Sandbox 地址规划。仅当默认值与主机或部署网络范围重叠时才更改这些值。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `host_interaction_cidr` | IPv4 CIDR | `10.11.0.0/16` | 每槽位主机交互地址池。必须包含至少 32768 个地址。 |
| `veth_cidr` | IPv4 CIDR | `10.12.0.0/16` | 每槽位命名空间 veth 对地址池。必须包含至少 65536 个地址。 |

配置的两个 CIDR 不得相互重叠，也不得与 AgentENV 固定的 VM tap 链路 `169.254.0.20/30` 重叠。无论 `always_denied_cidrs` 如何，这些网络也会被视为保留的 Sandbox 出口目标。

## `[machine]`

Sandbox 的默认 VM 资源。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `mem_size_mib` | integer | `1024` | 客户机 RAM，以 MiB 计 |
| `vcpu_count` | integer | `2` | 虚拟 CPU 数量 |

## `[machine.disk_rate_limit]`

每 Sandbox 磁盘 I/O 限制，通过 Firecracker 的 virtio-blk 限速器应用。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `enabled` | boolean | `false` | 启用每 Sandbox 磁盘 I/O 限速 |
| `bandwidth_bytes_per_sec` | integer | `0` | 持续磁盘带宽限制，以字节/秒计；`0` 表示不限制 |
| `bandwidth_burst_bytes` | integer | `0` | 一次性带宽突发，以字节计，在 VM 启动时授予一次（Firecracker 的 `one_time_burst`）。它在持续桶之前被消耗且不会补充，因此只能吸收初始 I/O 峰值。要求 `bandwidth_bytes_per_sec` |
| `iops` | integer | `0` | 持续 IOPS 限制；`0` 表示不限制 |
| `iops_burst` | integer | `0` | 一次性 IOPS 突发，在 VM 启动时授予一次。在持续桶之前被消耗且不会补充。要求 `iops` |

## `[envd]`

客户机内 `envd` 守护进程设置。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `version` | string | `"0.5.15"` | 烘焙在 tools 驱动镜像中的预期 envd 版本 |
| `init_timeout_secs` | integer | `60` | VM 启动后等待 envd 就绪的最大秒数 |
| `poll_ms` | integer | `3` | envd 健康检查重试的轮询间隔（ms） |

## `[sandbox]`

Sandbox 控制通信设置。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `access_token_hash_seed` | string | 自动生成 | 可选覆盖，用于派生 Sandbox envd 和流量访问令牌的密钥。未设置时，正常服务器启动会创建并复用 `$AENV_HOME/secrets/sandbox-access-token-hash-seed`。对于集群部署，请配置显式的共享值。 |

托管的 seed 是节点本地持久状态，必须包含在 `$AENV_HOME` 的备份中。当存在持久化的安全或私有入口 Sandbox 时，AgentENV 拒绝生成替代值。显式环境变量或 TOML 值优先于托管文件；更改该有效值会使现有 Sandbox 访问令牌失效。

在集群部署中，需在每个运行时节点上将 `AENV_SANDBOX_ACCESS_TOKEN_HASH_SEED` 配置为相同的值。独立的运行时节点在未设置时使用其托管 seed。

## `[volume]`

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `max_size_mb` | integer | `262144` | 持久卷的最大大小，以 MiB 计（256 GiB）。 |
| `max_volume_count` | integer | `4` | 一个 Sandbox 可挂载的持久卷最大数量。必须介于 1 和 Firecracker 额外驱动限制之间。 |

## `[orchestrator]`

Sandbox 生命周期管理。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `auto_evict_interval_ms` | integer | `1000` | 后台超时淘汰的轮询间隔（ms） |
| `metrics_interval_secs` | integer | `15` | 客户机指标扫描间隔；0 禁用收集 |
| `metrics_retention_secs` | integer | `3600` | 节点本地内存中客户机样本的保留时长（秒） |
| `default_sandbox_timeout_secs` | integer | `15` | Sandbox 的默认 keep-alive 超时 |
| `auto_resume_min_sandbox_timeout_secs` | integer | `300` | 当数据平面请求指向非运行 Sandbox 时，自动恢复它（如启用了自动恢复）并将超时刷新为不少于此时长 |
| `persisted_sandbox_store_path` | string | `"$AENV_HOME/persisted-sandboxes"` | 持久化 Sandbox 状态的目录 |

## `[pool]`

由网络槽位、块设备和预生成 Firecracker 进程共享的进程级热池默认值。池会预热到低水位，然后当真实获取耗尽池时，将补充目标按几何级数增长至高水位。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `low_watermark` | integer | `2` | 所有已启用热资源池的初始下限 |
| `high_watermark` | integer | `64` | 所有已启用热资源池的最大空闲目标 |

组件节：

| 节 | 键 | 类型 | 默认值 | 说明 |
|---------|-----|------|---------|-------------|
| `[pool.network]` | `enabled` | boolean | `true` | 为 `false` 时禁用网络槽位维护 worker；槽位仍按需创建 |
| `[pool.network]` | `maintenance_enabled` | boolean | `true` | 启用后台网络槽位维护 worker |
| `[pool.network]` | `startup_prewarm` | boolean | `true` | 已接受但当前未使用 |
| `[pool.block]` | `enabled` | boolean | `true` | 启用 ublk overlaybd 热设备池 |
| `[pool.block]` | `maintenance_enabled` | boolean | `true` | 被忽略：块池始终在请求时补充，因为可复用设备的形态取决于镜像和大小 |
| `[pool.block]` | `startup_prewarm` | boolean | 基于能力 | 在已知第一个可复用镜像形态后预热块设备。省略时，仅当内核支持 `UBLK_F_UPDATE_SIZE` 时启用；显式值会覆盖检测结果 |
| `[pool.firecracker]` | `enabled` | boolean | `true` | 启用用于 Snapshot 恢复的预生成 Firecracker 进程 |
| `[pool.firecracker]` | `maintenance_enabled` | boolean | `true` | 启用后台 Firecracker 进程维护 worker |
| `[pool.firecracker]` | `startup_prewarm` | boolean | `true` | 在服务器启动期间生成热 Firecracker 条目至低水位 |
| `[pool.firecracker]` | `fill_concurrency` | integer | `4` | 一个维护补充批次同时创建的热 Firecracker 进程最大数量 |

验证规则：

- `low_watermark <= high_watermark`
- `[pool.firecracker].fill_concurrency > 0`

## `[node_identity]`

此节点的稳定标识字段。这些值会出现在节点 API 响应和 Scheduler 心跳中。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `node_id` | string | 由主机名派生 | 由 admin/node API 返回的稳定节点标识符 |
| `cluster_id` | string（UUID） | nil UUID | 包含在节点 Snapshot 中的逻辑集群标识符 |
| `service_instance_id` | string | 生成的 UUID | 当前节点运行时的唯一进程/服务实例标识符 |

## `[observability]`

节点级可观测性和主机指标收集。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `enabled` | boolean | `true` | 启用节点/admin 可观测性服务。禁用时，`/nodes` 返回空列表，`/nodes/{nodeID}` 返回 `404` |

启用可观测性后，会在请求时收集主机 CPU/内存/磁盘指标。CPU 百分比由两次采样计算；第一次节点指标请求会等待约 100ms 以返回测量值。

## `[observability.scheduler_report]`

用于多节点控制平面集成的可选 Scheduler 心跳上报。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `enabled` | boolean | `false` | 启用周期性 Scheduler 心跳上报。要求 `[cluster].scheduler_endpoint` |
| `interval_secs` | integer | `5` | 心跳上报间隔，以秒计 |

环境变量覆盖：

- `AENV_OBSERVABILITY_SCHEDULER_REPORT_ENABLED`
- `AENV_OBSERVABILITY_SCHEDULER_ENDPOINT`
- `AENV_OBSERVABILITY_REPORT_INTERVAL_SECS`

`AENV_OBSERVABILITY_SCHEDULER_ENDPOINT` 仅对上报器进程覆盖 `[cluster].scheduler_endpoint`。

## `[cluster]`

共享的集群级服务端点。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `scheduler_endpoint` | string | 未设置 | Scheduler 的 gRPC 端点，例如 `"http://127.0.0.1:9090"`。用于 Scheduler 心跳上报和 P2P 对等发现。 |

## `[p2p]`

> **实验性：** P2P 尚未在生产环境中测试。除非部署接受该运营风险，否则在生产中保持禁用。

项目范围的 artifact 传输配置。传输默认禁用。启用后，它会被 overlaybd P2P HTTP facade 以及 Snapshot 发布/运行时解析用作可选加速路径。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `enabled` | boolean | `false` | 启用 P2P artifact 传输。为 false 时，AgentENV 使用 `DisabledP2pTransport`，因此查找会未命中，发布为空操作。 |
| `transport` | string | `"iroh"` | 传输后端。支持的值为 `"disabled"` 和 `"iroh"`。`enabled = false` 时被忽略。 |
| `store_dir` | string | `"$AENV_HOME/p2p/store"` | 传输后端使用的本地存储。相对显式路径会相对于配置文件目录解析。 |
| `listen_addr` | string | `"0.0.0.0:0"` | 嵌入式传输端点的可选本地监听地址。端口 `0` 让 OS 选择空闲端口。 |
| `lookup_timeout_ms` | integer | `5000` | 针对一个对等进行一次 artifact 目录查找的超时时间。 |
| `fetch_timeout_ms` | integer | `30000` | 从一个对等获取一个 artifact 的超时时间。 |
| `peer_discovery_refresh_interval_secs` | integer | `5` | 从 Scheduler 刷新对等端点的间隔。小于一秒的值会被限制为一秒。 |

## `[custom_extension]`

自定义扩展服务配置。当 `url` 未设置时，集成完全禁用。参见 [自定义扩展](../concepts/custom-extension/index.md)。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `url` | string | 未设置 | 自定义扩展服务的 HTTP 基础 URL。设置后，AgentENV 会在 `POST {url}/sandbox-hook/*` 下调用 Sandbox 生命周期钩子。 |
| `timeout_ms` | integer | `5000` | 每次自定义扩展 HTTP 调用的超时时间，以毫秒计。 |

## `[snapshot]`

Snapshot 存储/构建配置。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `local_cache_path` | string | `"$AENV_HOME/snapshot-local-cache"` | 管理器拥有的节点本地 Snapshot artifact/缓存根目录。相对显式路径会相对于配置文件目录解析。 |
| `repository_backend` | string | `"posix_fs"` | Snapshot 仓库后端。支持的值：`"posix_fs"` 和 `"oss"` |
| `p2p_enabled` | boolean | `true` | 启用后，Snapshot 管理器会将已提交的 Snapshot 发布到 P2P 传输，并尝试从其中解析，然后回退到仓库后端。 |

环境变量覆盖：

- `AENV_SNAPSHOT_LOCAL_CACHE_PATH`

## `[snapshot.image_publish]`

源 registry 镜像发布。仅在 `snapshot.repository_backend = "oss"` 时生效。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `enabled` | boolean | `false` | 启用后，发布 Snapshot 时还会将其 rootfs 作为 OverlayBD 原生 OCI 镜像标签 `agentenv-snapshot-{snapshot_id}` 推送到原始源 registry。要求该 registry 中的源镜像是 OverlayBD 原生，并且 Docker 配置（`~/.docker/config.json`）中有推送凭证。现有远程层按 digest 引用；只上传新的增量层。发布的引用会作为 `imageRef` 暴露在 Snapshot API 中。内存和 VM 状态 artifact 始终保留在 Snapshot 仓库中。 |

## `[snapshot.publish_compression]`

上传到 OSS/ACR 的 Snapshot 层的发布时压缩。本地层始终保持原始格式，因此本地恢复无需承担解压开销；默认启用，内存层和增量读写层在上传时压缩一次，减少跨节点恢复的网络字节。这是唯一的压缩开关；`[memory_snapshot]` 和 `[template_build]` 下的旧式捕获时开关已从配置模式中移除。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `enabled` | boolean | `true` | 上传到 OSS/ACR 时压缩内存层和增量读写层。 |
| `algorithm` | string | `"lz4"` | 压缩算法。有效值仅为 `lz4` 和 `zstd`。 |
| `workers` | integer | `1` | 用于压缩层内 4 KiB 块的阻塞线程数。`1` 为顺序压缩；更高的值会并行运行且不改变输出布局。限制为 64。 |

已知影响：压缩层记录时不带层 uuid（ZFile 层不携带 LSMT uuid），因此按 uuid 键控的 P2P 加速不适用于它们。Snapshot P2P 发布也会跳过对本地原始层的按 digest 键控的广告，因为这些层的 digest 不在已提交记录中——记录命名的是压缩后的字节，因此原始 digest 键永远不会被消费者查到。


## `[snapshot.memory_startup_pack]`

针对 `oss` 和 `posix_fs` 仓库的可选启动内存轨迹记录和尽力预取。默认禁用。发布从不等待记录：完成的轨迹会编码为经 SHA-256 验证的 manifest，并附加到已提交的 Snapshot。没有描述符、manifest 缺失/损坏或预取失败的 Snapshot 始终通过普通按需路径恢复。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `enabled` | boolean | `false` | 捕获后记录首次触碰轨迹，并在记录成功时持久化 manifest。 |
| `record_min_window_ms` | integer | `200` | 记录器首次读取后的最小观测窗口。 |
| `record_quiet_ms` | integer | `300` | 在此静默间隔后停止记录。 |
| `record_max_window_ms` | integer | `2000` | 记录器观测窗口的硬上限。 |
| `record_budget_secs` | integer | `10` | 尽力记录的预算；它从不阻塞 Snapshot 发布。 |
| `max_pack_bytes` | integer | `1073741824` | 最大记录页数据预算和 POSIX 逻辑预取预算。 |
| `consume_enabled` | boolean | `false` | 启用恢复时预取；在推广和 A/B 对比期间保持禁用。 |
| `consume_timeout_secs` | integer | `30` | 排队和分派后续预取读取的截止时间；内核内读取可能超出该时间。 |

对于 POSIX，解析器会将常规 Sandbox 启动路径交给一个 `LocalPath` manifest。获取实际的共享内存 ublk 设备后，执行器会通过带缓冲的设备句柄读取其逻辑 manifest 范围，与恢复并发进行。这会预热 Firecracker 使用的设备页缓存，而非下层文件的独立缓存。每次运行使用一个读取器和 1 MiB 缓冲区；进程范围内最多运行四个读取器。进行中的共享包括 manifest digest 和活动设备代次。

取消和时间预算会停止后续读取的分派。已经在执行的内核读取无法被中断。停止和重试最多等待 Sandbox 套接字超时以进行预取清理；之后它会在后台继续，保留设备租约直至读取完成。`O_DIRECT`（`io_engine = 2`）仍被显式跳过，等待在设备预取下验证该下层 I/O 配置；这并非声称设备页缓存被绕过。配置的 I/O 引擎永远不会被更改。OSS 保留其现有的物理范围规划器和执行器。预取不是正确性依赖。

## `[backend.posix_fs]`

基于 POSIX 文件系统的 Snapshot 仓库配置。当 `snapshot.repository_backend = "posix_fs"` 时使用此节。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `snapshot_store` | string | `"$AENV_HOME/snapshot-store"` | 持久已提交 Snapshot 仓库状态的根目录。相对显式路径会相对于配置文件目录解析。 |

环境变量覆盖：

- `AENV_SNAPSHOT_STORE`

## `[backend.oss]`

基于 OSS 的 Snapshot 仓库配置。当 `snapshot.repository_backend = "oss"` 时，此节为必填。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `endpoint` | string | 无 | OSS 端点 URL，例如 `"https://oss-cn-hangzhou.aliyuncs.com"` |
| `bucket` | string | 无 | 用于已提交 Snapshot 状态的 OSS bucket 名称 |
| `prefix` | string | 空 | bucket 下可选的对象键前缀 |
| `credential_process` | string | 未设置 | 用于获取 OSS 凭证的外部命令。使用普通的可执行文件加参数形式，不含 shell 展开、管道或命令替换，以便在 AgentENV 和 overlaybd 凭证消费者之间行为一致 |
| `access_key_id` | string | 未设置 | 静态 OSS access key ID。未设置 `credential_process` 时必填 |
| `access_key_secret` | string | 未设置 | 静态 OSS access key secret。未设置 `credential_process` 时必填 |
| `security_token` | string | 未设置 | 与静态访问密钥凭证配对的可选会话令牌 |
| `region` | string | 无 | 传递给 S3 兼容对象存储客户端的区域；当前 OSS 后端必填 |
| `addressing_style` | string | 自动检测 | bucket 寻址样式，`"virtual"` 或 `"path"`。未设置时，后端会自动检测：阿里云 OSS 和端点内置 bucket 的主机使用虚拟主机样式，其他端点默认使用路径样式。当提供商要求或偏好的样式与检测到的默认值不同时，设置任一值 |
| `cache_max_size_gb` | integer | `10` | 节点本地 OSS artifact 缓存的最大大小，以 GiB 计 |

说明：

- `credential_process` 和静态访问密钥设置在实际上互斥；设置 `credential_process` 时，后端会忽略静态凭证字段。
- `credential_process` 应写为可移植的 argv 风格命令行。避免 `$VAR`、反引号、`$(...)`、管道和 shell 内建命令。
- 尽管配置节仍名为 `oss`，运行时路径是通过共享的 S3 兼容客户端实现的，因此必须配置 `region`。
- 当基于端点的检测正确时，保持 `addressing_style` 未设置。当提供商要求或偏好的样式与检测到的默认值不同时，将其设置为 `"virtual"` 或 `"path"`；例如，某些 Tigris 或 Cloudflare R2 部署使用虚拟主机寻址。
- 该设置涵盖数据路径的两部分：Snapshot 仓库客户端（元数据和 artifact 上传/下载）以及生成的 OverlayBD 运行时配置（`ossConfig.defaultAddressingStyle`），运行时在 Sandbox 恢复期间读取远程托管 Snapshot 层时使用后者。

对于要求或偏好虚拟主机寻址的 S3 兼容提供商——例如 [Tigris](https://www.tigrisdata.com/docs/)——请显式设置 `addressing_style`：

```toml
[backend.oss]
endpoint = "https://t3.storage.dev"
bucket = "agentenv-snapshots"
region = "auto"
addressing_style = "virtual"
```

其他路径覆盖：

- `AENV_DEPS_PATH`

Setup 的 sysctl 调优是主机级设置。当服务器检测到自身运行在容器内时，会在读取 `/proc/sys` 之前跳过它。仅对具有可写主机 sysctl 的特权容器设置 `AENV_FORCE_SYSCTL_TUNING=1`；否则请在主机上配置这些内核参数。

## `[protoc]`

代码生成的 Protobuf 编译器元数据位于 `config/deps_manifest.toml`，而非 `config.toml`。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `version` | string | `"33.4"` | protoc 发行版本 |
| `url` | string | GitHub release URL | 含 `{version}` 和 `{platform}` 占位符的下载 URL Template |

## `[ublk]`

用户空间块设备配置。Rootfs 通过由 `uvm-ublk-daemon` 管理的、基于 OverlayBD 的 ublk 设备提供。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `daemon_binary_path` | string | `"$AENV_HOME/ublk/uvm-ublk-daemon"` | `uvm-ublk-daemon` 二进制的路径 |
| `daemon_socket_path` | string | `"$AENV_RUNTIME/ublk-daemon.sock"` | 守护进程使用的 Unix 套接字路径 |
| `daemon_log_path` | string | `"$AENV_HOME/logs/ublk-daemon.log"` | 守护进程日志的文件路径；部署方负责轮转和保留 |
| `daemon_metrics_listen_addr` | string | `"0.0.0.0:9103"` | 守护进程 Prometheus 指标的 HTTP 监听地址；空字符串会禁用它 |

环境变量覆盖：

- `AENV_UBLK_DAEMON_BINARY_PATH`
- `AENV_UBLK_DAEMON_METRICS_LISTEN_ADDR`

## `[ublk.overlaybd]`

ublk 的 OverlayBD 配置。旧式 `enabled` 和 `device_type` 键会被忽略。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `global_config_path` | string | `"$AENV_HOME/overlaybd/overlaybd-global.json"` | overlaybd 全局配置 JSON 的路径（见下方说明）。相对显式路径会相对于配置文件目录解析。 |
| `read_only` | boolean | `false` | 设置为 `true` 时，在无可写上层的情况下物化 rootfs |
| `runtime_upper_mode` | string | `"hybridLogStructured"` | 新物化的可写 rootfs OverlayBD 镜像的运行时上层格式。支持的值为 `"logStructured"`、`"hybridLogStructured"` 和 `"sparse"`。现有源上层保持其自身模式 |
| `allow_shrink` | boolean | `false` | 允许显式冷启动 `diskSizeMB` 小于源 rootfs。显式大小使用 MiB，且必须能被 1024 整除。增长始终允许；Snapshot 恢复从不调整大小。 |
| `resize_timeout_secs` | integer | `120` | 冷启动 OverlayBD 调整大小工具的超时时间，以秒计。必须大于零。 |
| `remote_io_workers` | integer | `4` | overlaybd 运行时专用远程 I/O 运行时（OSS 和 registry 下载）的 worker 线程数，会作为 `remoteIoWorkers` 写入生成的 overlaybd 全局配置。必须大于零。 |
| `download_enable` | boolean | `false` | 启用远程层的 overlaybd 层级别后台下载 |
| `p2p_lookup_timeout_ms` | integer | `300` | 通过 localhost P2P HTTP facade 进行一次前台 Overlaybd 描述符查找的超时时间。超时被视为可缓存的未命中。 |
| `p2p_fetch_range_timeout_ms` | integer | `2000` | 在回退到源 registry 之前，通过 localhost P2P HTTP facade 进行一次前台 Overlaybd 范围获取的超时时间。 |

### `global_config_path` 与自动生成的配置

位于配置默认路径 `$AENV_HOME/overlaybd/overlaybd-global.json` 的文件由服务器在启动时**自动生成**。生成的 JSON 会将若干 TOML 设置——`[image.cache].root_dir`、`[image.cache.remote_blocks].max_size_gb`、`download_enable`、`[backend.oss]` 凭证以及从 `~/.docker/config.json` 检测到的 Docker registry 凭证——合并到一个 overlaybd 运行时配置文件中。

服务器在每次启动时重新生成 `global_config_path` 处的文件，因此这些 TOML 设置始终自动生效——对生成文件的任何手动编辑都会在下次启动时被覆盖。要保留自定义设置，请通过 TOML 设置进行，而非编辑生成的 JSON。

## `[memory_snapshot]`

内存 Snapshot overlaybd 配置。服务器在每次启动时自动生成默认路径处的文件。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `overlaybd_global_config_path` | string | `"$AENV_HOME/overlaybd/mem-overlaybd-global.json"` | 用于内存 Snapshot ublk 后端的 overlaybd 全局配置路径。启动时重新生成（手动编辑会被覆盖）；仅在需要迁移生成文件时更改。 |
| `track_dirty_pages` | bool | `true` | 为内存 Snapshot 启用 Firecracker KVM 脏页跟踪。PVM 会自动禁用它，因为此组合尚未测试。内存 Snapshot 打包始终使用直接 OverlayBD 路径。设置 `AGENTENV_MEMORY_SNAPSHOT_TRACK_DIRTY_PAGES=false` 以禁用它。 |

## `[memory_snapshot.background_download]`

专用于远程内存 Snapshot OverlayBD 层的后台下载设置。它们不会更改通用 rootfs 或附加驱动的默认值。所有字段都会序列化到生成的内存 OverlayBD 全局配置中。每个远程层会逐块填充到节点本地远程文件缓存：缓存拥有的后台下载调度器为每个远程层注册一个任务（按 blob 去重，跨 Sandbox 共享），并且只下载条目位图中仍然缺失的块——每个源请求获取 `block_size` 字节（对齐到整个缓存块），并在块一落地就发布该块的缓存块。提交在负载下永远不会被拒绝——任务在 Scheduler 容量允许时运行，每个文件缓存后端最多同时运行 `maxConcurrentFiles` 个层任务（来自生成的 overlaybd 下载配置，默认 8），每个层最多并行 `concurrency` 个块读取，受 Scheduler 的 `max_inflight_blocks` 上限约束。绑定到 Sandbox 的设备的下载仅在 envd 就绪（加上 `delay`）后开始，如果就绪信号丢失则有 20s 回退；当前台远程读取正在进行时，后台块读取会让位于一个小的保底份额，而非以全速竞争。生成的内存配置关闭限流（`maxMBps = 0`）；带有正 `maxMBps` 的镜像配置会在块任务之间保留其历史共享速率限制。完成的缓存块一旦提交到缓存位图即对前台读取可见；没有暂存文件、没有全文件 digest 检查、也没有切换到本地，因此失败或取消的块只会保持未缓存，并由前台读取或后续重试按需获取。缓存是有界工作集：块可能在容量压力下被淘汰，然后按需重新获取。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `enable` | boolean | `true` | 为远程内存 Snapshot 层启用后台下载。 |
| `delay` | integer | `0` | envd 就绪后开始后台下载前的延迟秒数（下载绝不会在 envd 就绪前开始；如果就绪信号丢失则应用 20s 回退）。 |
| `delay_extra` | integer | `1` | 随机额外延迟的排他上限。默认 `1` 确保 `delay = 0` 不添加抖动。 |
| `try_cnt` | integer | `5` | 重试次数，语义与 OverlayBD `DownloadConfig.tryCnt` 相同。 |
| `block_size` | integer | `16777216` | 后台下载块大小，以字节计（16 MiB）：一个源请求获取此大小的块，向下对齐到整个缓存块。缓存为前台读取保留自己更小的块大小，因此后台下载保持大请求吞吐，而前台保持细粒度按需读取。每个活动层下载的峰值暂存为 `block_size × concurrency`。 |
| `concurrency` | integer | `4` | 单个远程层内进行中块远程读取的最大数量。`1` 保持历史串行行为。必须大于零。 |
| `max_inflight_blocks` | integer | `16` | 每个文件缓存后端的下载 Scheduler 强制执行的并发下载块上限，由该后端上的每个并发层下载共享；将总暂存内存限制为 `max_inflight_blocks` × 下载块大小（`block_size`）。该值在从全局配置创建后端时固定；每镜像的 `download` 覆盖永远不会调整 Scheduler 拥有的上限（每个 Scheduler 的第一次不匹配会记录为 `max_inflight_blocks_override_ignored`）。必须大于零。 |

## `[template_build]`

托管的 Dockerfile 构建器资源。节点上的第一次 Dockerfile 构建会准备一个可复用的内部构建器 Template。每次构建会挂载仓库共享缓存种子的独立克隆；并发构建不会排队等待缓存所有权。最后成功发布的缓存会成为下一个种子，并尽力复用并发分支。构建器资源不会改变最终 Template 的 CPU 或内存。

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `max_concurrent_builds` | integer | `4` | 每节点托管构建的限制，包括准备、发布和清理。必须大于零。多余的构建器 PUT 请求返回 HTTP 429，并让构建等待重试。 |
| `builder_image` | string | `"docker.io/moby/buildkit:v0.33.0"` | 包含托管 BuildKit 守护进程、客户端和 OCI 运行时的镜像。 |
| `builder_cpu_count` | integer | `16` | 构建器 vCPU，从 1 到 255。 |
| `builder_memory_mb` | integer | `32768` | 构建器内存，以 MiB 计，从 256 到 2147483647。 |
| `cache_size_mb` | integer | `65536` | 新的持久 BuildKit 数据磁盘的容量，以 MiB 计。至少 1024 且至多 `volume.max_size_mb`；更改它不会调整现有缓存的大小。 |
