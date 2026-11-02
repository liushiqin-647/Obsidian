# 启动 Sandbox

启动 Sandbox 有两种方式：从可复用的 Template 或 Snapshot 热启动，或直接从 OCI 镜像冷启动。

## 从 Template 或 Snapshot 启动

使用热启动之前，请先创建一个 [Template](../templates/index.md) 或一个 [Snapshot](../snapshots/index.md)。之后你就可以从其别名或 ID 启动任意数量的 Sandbox：

用法：

```bash
aenv start <template-or-snapshot> [options]
```

示例：

```bash
# Start by alias
aenv start my-python-template

# Start by ID
aenv start 018f0d93-aaaa-bbbb-cccc-0123456789ab
```

热启动选项：

| 参数或选项 | 默认值 | 描述 |
|---|---|---|
| `<template-or-snapshot>` | 必填 | Template 或 Snapshot 的 ID 或别名。 |
| `--timeout <seconds>` | `300` | 设置 Sandbox TTL。Sandbox 达到 TTL 时会自动暂停；参见 [自动驱逐](./auto-eviction.md)。 |
| `--volume <mount-path>=<volume>` | 无 | 按 ID 或名称挂载持久化 Volume（卷）。重复该选项可挂载多个 Volume。当 Snapshot 已存在 Volume 挂载时不允许使用。 |
| `--image-mount <mount-path>=<image>` | 无 | 从镜像附加一个可写驱动器；参见 [附加镜像驱动器](#attach-image-drives)。可重复。当 Snapshot 已存在 Volume 挂载或已附加驱动器时不允许使用。 |
| `--image-mount-ro <mount-path>=<image>` | 无 | 从镜像附加一个只读驱动器。可重复。当 Snapshot 已存在 Volume 挂载或已附加驱动器时不允许使用。 |
| `-d`, `--detach` | 关闭 | 打印 Sandbox ID 后退出，而不附加交互式 shell。 |

不带 `--detach` 时，`aenv start` 会等待 Sandbox 就绪，然后附加一个交互式 shell。CPU、内存和磁盘设置继承自 Template 或 Snapshot，热启动时无法覆盖。CLI 始终启用安全 Sandbox 认证，并自动管理 envd 访问令牌；参见 [安全 Sandbox 认证](../authentication/secure-sandbox.md)。

要获取某个 Sandbox 的当前状态和配置，请使用 HTTP API：

```bash
curl -H 'X-API-Key: test-key' \
  http://127.0.0.1:8000/sandboxes/<sandbox-id>
```

## 从 OCI 镜像冷启动

冷启动会直接解析 OCI 镜像，并在运行时准备一个全新的可写根文件系统：

用法：

```bash
aenv start --cold <image> [options]
```

示例：

```bash
aenv start --cold ubuntu:24.04
aenv start --cold ubuntu:24.04 --cpu 4 --memory 4096 --disk-size-mb 65536
```

冷启动选项：

| 参数或选项 | 默认值 | 描述 |
|---|---|---|
| `<image>` | 必填 | 外部 OCI 镜像引用、已发布的镜像摘要（`sha256:...`）或镜像标签（`aenv/name:tag`）。 |
| `--cold` | 从 OCI 镜像启动时必填 | 直接从 `<image>` 冷启动。 |
| `--timeout <seconds>` | `300` | 设置 Sandbox TTL。Sandbox 达到 TTL 时会自动暂停；参见 [自动驱逐](./auto-eviction.md)。 |
| `--cpu <count>` | 你的 AgentENV 配置文件中的 `[machine].vcpu_count` | 设置 Sandbox 的 vCPU 数量。别名：`--cpu-count`。 |
| `--memory <MiB>` | 你的配置文件中的 `[machine].mem_size_mib` | 设置 Sandbox 内存。别名：`--memory-mb`、`--mem`。 |
| `--disk-size-mb <MiB>` | 源镜像的虚拟大小 | 设置根文件系统大小。该值必须大于零且能被 1024 MiB 整除。别名：`--disk-mb`。 |
| `--volume <mount-path>=<volume>` | 无 | 按 ID 或名称挂载持久化 Volume。重复该选项可挂载多个 Volume。 |
| `--image-mount <mount-path>=<image>` | 无 | 从镜像附加一个可写驱动器；参见 [附加镜像驱动器](#attach-image-drives)。可重复。 |
| `--image-mount-ro <mount-path>=<image>` | 无 | 从镜像附加一个只读驱动器。可重复。 |
| `-d`, `--detach` | 关闭 | 打印 Sandbox ID 后退出，而不附加交互式 shell。 |

冷启动的 Sandbox 默认也使用安全 Sandbox 认证。

AgentENV 配置文件默认为 `config/default.toml`，或由 `AENV_CONFIG_PATH` 指定的文件。

OverlayBD 原生镜像无需先下载完整镜像即可启动；其文件系统数据按需从注册表加载。参见 [按需加载](../../getting-started/on-demand-loading.md)。

默认允许增大磁盘大小。若要缩小到小于源镜像大小，需要在 AgentENV 配置文件中设置 `ublk.overlaybd.allow_shrink = true`。调整大小仅在创建全新的可写根文件系统时生效，不适用于只读镜像、已有上层的镜像或 Snapshot 恢复。Sandbox 响应会以 `diskSizeMB` 报告有效大小。

## 附加镜像驱动器

热启动和冷启动都可以将镜像文件系统作为驱动器附加到客户机路径：

```bash
aenv start my-template \
  --image-mount /workspace=aenv/repo:main \
  --image-mount-ro /mnt/models=aenv/models:v1
```

CLI 会将这些作为 `attachedDrives` 条目发送，热启动时通过 `POST /v2/sandboxes`，冷启动时通过 `POST /sandboxes-cold`。每个条目包含一个 `driveID`、一个 `source.image`，以及可选的 `mountPath`、`subPath`、`readOnly` 和 `diskSizeMB`。API 默认将 `readOnly` 设为 `true`；CLI 对 `--image-mount` 将其设为 `false`，并根据挂载路径推导 `driveID`，例如 `/mnt/models` 对应 `image_mnt_models`。镜像接受与冷启动 `<image>` 相同的引用。

附加驱动器属于其 Sandbox，并随其一起移除。写入内容不会保留到其他位置；对于必须在 Sandbox 之后继续存在的数据，请使用 [Volume](../volumes/index.md)。

| 操作 | 附加驱动器 |
|---|---|
| Snapshot | 将每个驱动器的当前状态捕获到 Snapshot 中。 |
| 从 Snapshot 启动 | 恢复已捕获的驱动器。当 Snapshot 存在 Volume 挂载或附加驱动器时，提供 `attachedDrives`（包括空数组）会被拒绝。 |
| Fork | 为每个子 Sandbox 提供驱动器状态的独立副本。 |
| 暂停和恢复 | 保留驱动器及其写入内容。 |

热启动时，附加驱动器和新 Volume 共享 Snapshot 预留的驱动器槽位，因此它们的合计数量不能超过其空闲槽位。在预留槽位存在之前捕获的 Snapshot 没有预留槽位。

## 构建可复用镜像

要构建独立镜像，请运行 `aenv build --image --name my-app ./app`。该命令会将已发布的 `sha256:...` 引用打印到 stdout；构建进度输出到 stderr。在 Sandbox 的 `image` 字段中使用该引用，或使用诸如 `aenv/my-app:latest` 的标签，或用 `aenv start --cold sha256:...` 冷启动它。Dockerfile、构建参数、密钥、缓存、进度和超时标志的用法与 Template 构建相同。启动覆盖项和 Sandbox 资源标志不适用于镜像构建；worker 资源来自服务器配置。

镜像通过其规范 OverlayBD 镜像描述（`sha256:...`）的 SHA256 来引用，而不是其原始 OCI 清单摘要。该描述绑定了有序的层摘要、平台和 OCI 运行时配置，包括 entrypoint、command、environment 和 healthcheck。描述存放在 `images/descriptions/{digest}.json`；层数据使用与 Snapshot 相同的内容寻址托管层存储。节点本地路径在解析时物化，不属于镜像标识的一部分。

构建器和运行时节点必须使用相同的仓库。每个拥有仓库访问权限的兼容节点都可以独立于构建节点的缓存解析镜像。本地缓存驱逐不会移除已发布的镜像。使用 POSIX 后端时，跨节点使用需要共享文件系统。发布在状态变为 `ready` 之前完成；发布失败不会暴露可用的构建结果。已上传的不可变对象可能会在构建失败后保留以供重用。

镜像构建使用 Template 构建 API。带有 `"imageOnly": true` 的 `POST /v3/templates` 会创建构建；`name`（可选 `name:tag`）和 `tags` 为镜像命名，且不会创建 Template。随后 `/templates/{templateID}/builds/{buildID}` 下的 builder、status 和日志端点会像 Dockerfile Template 构建一样工作，就绪状态会报告 `imageDigest`。构建记录存放在 `images/builds/{buildID}.json`，因此 worker 退出后任意节点都能报告状态。`DELETE /templates/{templateID}` 会移除构建记录和日志，并保留已发布的镜像及其标签。要使用镜像构建，请同时升级服务器、Gateway 和 CLI。

### 标记镜像

镜像标签是可变 `name:tag` 指针，指向在该名称下构建的某个镜像。在可接受 `sha256:...` 镜像引用的任何位置都可以将其引用为 `aenv/name:tag`，例如 `aenv start --cold aenv/my-app:v1`。省略 `:tag` 表示 `latest`。名称使用小写字母和数字，并以单个 `.`、`_` 或 `-` 连接；标签使用字母、数字、`.`、`_` 和 `-`，且不能以 `.` 或 `-` 开头。每项最多 128 个字符。`aenv/` 下的短名称始终指代标签；若要引用注册表中 `aenv` 命名空间的 OCI 镜像，请使用其完整名称，例如 `docker.io/aenv/app:v1`。

标签属于其镜像名称，与 Template 标签类似。构建会将 `--name NAME:TAG` 和每个 `--tag TAG` 指向其镜像；如果两者都未给出，则指向 `latest`。标签存放在共享仓库中的 `images/tags/{name}/{tag}.json`。

当 Sandbox、Volume 或 Template 构建使用标签时，标签会解析为其摘要。Sandbox、Snapshot 和节点缓存仅记录摘要，因此之后移动标签不会改变现有工作负载。与摘要引用相比，每次标签解析都会多读取一个仓库对象。
