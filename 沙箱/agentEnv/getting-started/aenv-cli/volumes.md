# 卷

## `aenv volume create <name>`

创建一个独立的持久 Volume（卷）。Volume 默认大小为 65536 MiB（64 GiB），模式为 `exclusive`。

```bash
aenv volume create workspace
aenv volume create models --mode ro --image ghcr.io/example/models:latest
aenv volume create job-workspace --from-volume workspace
```

| 标志 | 描述 |
|------|-------------|
| `--size-mb <MiB>` | Volume 大小（默认：65536）。写时复制 fork 必须与其源使用相同的大小。 |
| `--mode <exclusive\|ro>` | 访问模式（默认：`exclusive`）。独占 Volume 可由一个 Sandbox 写入；只读 Volume 可共享。 |
| `--from-volume <volume>` | 从现有 Volume ID 或名称创建写时复制 fork。与 `--image` 冲突。 |
| `--image <image>` | 从 OCI 镜像初始化 Volume。与 `--from-volume` 冲突。 |

我们建议为每个 Sandbox 创建一个独占 fork，而不是直接挂载共享的可写 Volume：

```bash
aenv volume create job-data --mode exclusive --from-volume dataset-base
aenv start ubuntu --volume /workspace/data=job-data
```

有关访问模式语义、生命周期行为、自动 Sandbox fork 与 Snapshot 处理以及完整的 CLI 示例，请参见 [Volume](../../concepts/volumes/index.md)。

## `aenv volume list`

列出持久 Volume。别名：`aenv volume ls`。

```bash
aenv volume list
aenv volume list --output json
```

| 标志 | 描述 |
|------|-------------|
| `--output <table\|json>` | 输出格式。在 TTY 上默认为 table，重定向时默认为 JSON。 |

## `aenv volume inspect <volume>`

按 ID 或名称检查 Volume。该命令以格式化 JSON 打印完整的 Volume 记录。

```bash
aenv volume inspect job-data
```

## `aenv volume delete <volume>`

按 ID 或名称删除 Volume。

```bash
aenv volume delete job-data
```

已挂载的 Volume 无法删除。
