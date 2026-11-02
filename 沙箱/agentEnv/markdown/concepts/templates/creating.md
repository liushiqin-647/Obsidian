# 创建 Template

创建 Template 有两种方式：`aenv pull` 直接导入已有的 OCI 镜像，而 `aenv build` 使用 BuildKit 运行 Dockerfile。

下文中的一些默认值来自 `config/default.toml`，或由 `AENV_CONFIG_PATH` 选定的文件。

## aenv pull

将已有 OCI 镜像拉取为 Template，并可选择为其赋予一个易记的名称：

```bash
aenv pull <image> [options]
```

示例：

```bash
aenv pull ubuntu:24.04
aenv pull ubuntu:24.04 --name my-base
```

`--name` 是可选的。不带它时，AgentENV 使用镜像仓库名称。Template 名称可以在任何接受 Template ID 的地方使用。

| 参数或选项 | 默认值 | 描述 |
|---|---|---|
| `<image>` | 必填 | OCI 镜像引用。支持诸如 `ubuntu:24.04` 的短名称和完整引用。 |
| `--name <name>` | 镜像仓库名称 | 分配一个人类可读的 Template 名称。 |
| `--cpu <count>` | 你的配置文件中的 `[machine].vcpu_count` | 设置 Template 的 vCPU 数量。别名：`--cpu-count`。 |
| `--memory <MiB>` | 你的配置文件中的 `[machine].mem_size_mib` | 设置 Template 的内存。别名：`--memory-mb`、`--mem`。 |
| `--start-cmd <cmd>` | 无 | 在捕获 Template Snapshot 之前运行一条命令。 |
| `--ready-cmd <cmd>` | 设置 `--start-cmd` 时为 `/agentenv/bin/busybox sleep 20`；否则无 | 每两秒轮询一次 shell 命令，直到其成功退出。 |
| `--probe <port>` | 无 | 等待 `localhost:<port>` 上的 TCP。不能与 `--ready-cmd` 组合使用。 |
| `-d, --detach` | 关闭 | 提交构建并立即返回，而不等待。 |
| `--timeout <seconds>` | 无超时 | 限制 CLI 等待构建的时长。不能与 `--detach` 组合使用。 |

使用 `--detach` 时，请用 `aenv template watch <template>` 监控构建。

## aenv build

在临时 microVM 中使用 BuildKit 构建 Dockerfile，然后将结果转换为 OverlayBD 并捕获 Template。CLI 机器上不需要 Docker 和暂存注册表。

```bash
aenv build <context> --name <name> [options]
# From the repository root:
aenv build . -f deploy/docker/Dockerfile.agentenv --name aenv
```

| 参数或选项 | 默认值 | 描述 |
|---|---|---|
| `<context>` | 必填 | 本地上下文目录，与 `docker build` 相同。 |
| `-f, --file <path>` | `<context>/Dockerfile` | Dockerfile 路径；显式相对路径从当前目录解析。 |
| `--name <name>` | 必填 | 分配 Template 名称。 |
| `--cpu <count>` | 你的配置文件中的 `[machine].vcpu_count` | 设置 Template 的 vCPU 数量。别名：`--cpu-count`。 |
| `--memory <MiB>` | 你的配置文件中的 `[machine].mem_size_mib` | 设置 Template 的内存。别名：`--memory-mb`、`--mem`。 |
| `--start-cmd <command>` | 镜像 `ENTRYPOINT`/`CMD` | 覆盖 Template 启动；空字符串会禁用启动。 |
| `--ready-cmd <command>` | 镜像 `HEALTHCHECK`，或正常的启动延迟 | 覆盖在 Snapshot 捕获之前必须成功的命令。 |
| `--build-arg KEY=VALUE` | 无 | 构建参数；可重复。 |
| `--secret <spec>` | 无 | 原生 BuildKit 密钥挂载；可重复。 |
| `--no-cache` | False | 不使用缓存指令重新构建。BuildKit 还会重置这些指令使用的缓存挂载。 |
| `--buildctl <path>` | `aenv` 旁边的 `aenv-buildctl` | 本地客户端可执行文件。 |
| `--progress <format>` | `auto` | 终端中显示三阶段进度条，重定向时显示纯日志。`plain` 选择纯日志；`tty` 选择 BuildKit 的原生显示。 |
| `--timeout <seconds>` | 3600 | 构建器准备和 Dockerfile 构建的截止时间；CLI 额外允许 10 分钟用于发布。 |

### 构建上下文和 Dockerfile

`COPY` 和 `ADD` 从上下文目录解析，与 Dockerfile 的位置无关。支持本地目录；不支持 URL 和 stdin 上下文。BuildKit 支持多阶段构建、`.dockerignore`、缓存挂载和标准 Dockerfile 语法。最终 Dockerfile 阶段始终会被发布。

构建 CLI 中没有镜像、阶段选择、SSH 或构建器资源覆盖选项。请使用 `FROM` 选择基础镜像，包括由 `FROM` 使用的 `ARG`。

### 启动和就绪

默认情况下，镜像的 `ENTRYPOINT` 和 `CMD` 决定 Template 的启动方式，Dockerfile 的 `HEALTHCHECK` 提供 Snapshot 捕获之前的就绪命令。Shell 健康检查遵循 Dockerfile 的 `SHELL`，而 `HEALTHCHECK NONE` 会禁用该检查。没有健康检查时，启动会使用正常的 Template 就绪延迟。

`--start-cmd` 和 `--ready-cmd` 可以独立覆盖启动和就绪，而不修改镜像配置。发布会启动镜像并运行选定的启动命令。因此，镜像可能编译成功，但如果其 entrypoint 需要客户机中不可用的设备，则可能在发布期间失败。若要在不启动其应用或运行其健康检查的情况下捕获镜像，请使用：

```bash
aenv build <context> --name <name> --start-cmd "" --ready-cmd true
```

### 构建容量

托管构建器设置属于服务器配置：

```toml
[template_build]
max_concurrent_builds = 4
builder_image = "docker.io/moby/buildkit:v0.33.0"
builder_cpu_count = 16
builder_memory_mb = 32768
cache_size_mb = 65536
```

每个节点最多接纳 `max_concurrent_builds` 个托管构建，包括准备、镜像发布和清理。达到每节点上限时，构建器准备会返回 HTTP 429。请在容量可用时重试构建。

`cache_size_mb` 控制持久化 BuildKit 缓存 Volume 的大小。其容量在创建缓存时设定，因此更改该设置不会调整现有缓存的大小。它必须至少为 1024 MiB，且不能超过 `volume.max_size_mb`。如果配置的缓存容量超过该 Volume 限制，Dockerfile 构建会被拒绝；设置更小的 Volume 限制不会阻止服务器启动或提供其他 API 服务。构建器的 CPU、内存和缓存容量与分配给结果 Template 的 CPU 和内存相互独立。

## 监控构建

`aenv pull` 默认会等待 Template 就绪。如果你使用 `--detach` 提交，请按 Template 名称或 ID 监控构建：

```bash
aenv template watch my-template
```

`aenv template watch` 没有超时选项。使用 `Ctrl-C` 停止本地监视；远程构建会继续。它会报告以下状态：

| 状态 | 含义 |
| --- | --- |
| `waiting` | 构建已提交，正在等待开始。 |
| `building` | Template 当前正在构建或发布。 |
| `ready` | 构建成功，Template 可以启动 Sandbox。 |
| `error` | 构建失败。可用时，`aenv template watch` 会报告失败原因。 |

## 运行时配置

镜像配置会影响从该 Template 创建的每个 Sandbox。`aenv pull` 保留源镜像配置，而 `aenv build` 从最终 Dockerfile 阶段生成它。下表区分了会自动影响 Sandbox 运行时的字段和需要显式 CLI 选项的字段。

| 镜像配置字段 | Dockerfile 指令 | 运行时效果 |
|-----------|------------------------|----------------|
| `Env` | `ENV` | Sandbox 进程可用的环境变量。 |
| `WorkingDir` | `WORKDIR` | 默认工作目录。 |
| `User` | `USER` | 默认用户。 |
| `Entrypoint` / `Cmd` | `ENTRYPOINT` / `CMD` | 决定 `aenv build` 的启动方式；`aenv pull` 需要 `--start-cmd` 来显式配置启动。 |
| `Healthcheck` | `HEALTHCHECK` | 决定 `aenv build` 的就绪状态；`--ready-cmd` 会覆盖它。 |

`EXPOSE`、`VOLUME` 和 `LABEL` 会作为镜像元数据保留，但不会创建端口、挂载 Volume 或以其他方式配置 Sandbox 运行时。
