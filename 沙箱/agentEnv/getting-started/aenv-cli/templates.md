# 模板

`aenv templates` 是 `aenv template` 的别名，包括其 `list`、`watch` 和 `delete` 子命令。

## `aenv pull <image>`

从 OCI 镜像创建 Template（Template）。默认等待构建完成。

```bash
aenv pull ubuntu:22.04
aenv pull ubuntu:22.04 --name my-ubuntu
```

| 标志 | 描述 |
|------|-------------|
| `--name <name>` | 覆盖 Template 名称。默认为镜像的仓库段。 |
| `--cpu <count>` | Template 的 CPU 核心数。默认使用服务器上的 `[machine].vcpu_count`。别名：`--cpu-count`。 |
| `--memory <MiB>` | Template 的内存。默认使用服务器上的 `[machine].mem_size_mib`。别名：`--memory-mb`、`--mem`。 |
| `--start-cmd <cmd>` | 在捕获 Template 快照之前，在 Sandbox 内运行的 shell 命令 |
| `--ready-cmd <cmd>` | 轮询执行的 shell 命令，直到其以 0 退出。当设置了 `--start-cmd` 时默认为 `/agentenv/bin/busybox sleep 20`；否则未设置。 |
| `--probe <PORT>` | 等待直到 `localhost:<PORT>` 接受 TCP 连接。与 `--ready-cmd` 冲突。 |
| `-d, --detach` | 提交构建并立即返回，不等待 |
| `--timeout <SECS>` | 等待构建完成的最长秒数。默认无超时。与 `--detach` 冲突。 |

## `aenv build <context> --name <name>`

使用 BuildKit 在隔离的 microVM 中从本地 Dockerfile 创建 Template。该命令会等待 Template 就绪。

```bash
aenv build . --name my-app
aenv build . -f deploy/docker/Dockerfile.agentenv --name aenv
```

| 标志 | 描述 |
|------|-------------|
| `--name <name>` | 必需的 Template 名称。 |
| `--cpu <count>` | Template 的 CPU 核心数。默认使用服务器上的 `[machine].vcpu_count`。别名：`--cpu-count`。 |
| `--memory <MiB>` | Template 的内存。默认使用服务器上的 `[machine].mem_size_mib`。别名：`--memory-mb`、`--mem`。 |
| `-f, --file <path>` | Dockerfile 路径。默认为 `<context>/Dockerfile`；显式相对路径从当前目录解析。 |
| `--start-cmd <command>` | 覆盖镜像的 `ENTRYPOINT`/`CMD`；空字符串禁用启动。 |
| `--ready-cmd <command>` | 用一条必须在捕获前成功的命令覆盖镜像的 `HEALTHCHECK`。 |
| `--build-arg KEY=VALUE` | 构建参数；可重复。 |
| `--secret <spec>` | BuildKit secret 挂载；可重复。 |
| `--no-cache` | 不使用缓存的指令或其缓存挂载进行重建。 |
| `--buildctl <path>` | 选择本地 BuildKit 客户端可执行文件。 |
| `--progress <auto\|plain\|tty>` | 构建进度格式。默认为 `auto`。 |
| `--timeout <seconds>` | 构建截止时间，从 1 到 86400 秒。默认为 3600；CLI 会额外允许 10 分钟用于供应和发布。 |

## `aenv template list`

列出所有 Template。别名：`aenv template ls`、`aenv templates list`。

```bash
aenv template list
aenv template list --output json
```

| 标志 | 描述 |
|------|-------------|
| `--output <table\|json>` | 输出格式。在 TTY 上默认为 table，重定向时默认为 JSON。 |

## `aenv template watch <template>`

监视 Template 构建直到成功或失败。接受 Template 名称/别名或 Template UUID。

```bash
aenv template watch my-ubuntu
aenv template watch <template-id>
```

## `aenv template delete <template>`

按名称或 ID 删除 Template。别名：`aenv template rm`。

```bash
aenv template delete my-ubuntu
aenv template delete <template-id>
```
