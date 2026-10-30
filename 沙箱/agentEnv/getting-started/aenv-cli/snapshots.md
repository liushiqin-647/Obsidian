# 快照

`aenv snap` 是 `aenv snapshot` 的别名，包括其 `create` 和 `list` 子命令。

## `aenv snapshot create <sandbox-id>`

从运行中的 Sandbox（沙盒）捕获一个持久 Snapshot（Snapshot）。该 Snapshot 可作为 Template（模板）用于通过 `aenv start` 启动新的 Sandbox。

```bash
aenv snapshot create <sandbox-id>
aenv snapshot create <sandbox-id> --name my-base
```

| 标志 | 描述 |
|------|-------------|
| `--name <name>` | Snapshot 名称或别名。如果省略，生成的 Snapshot ID 用于标识该 Snapshot。 |

当服务器启用了源注册表镜像发布时，该命令还会在 `Image:` 行打印已发布的 OverlayBD 原生镜像引用；该引用可传递给 `aenv start --cold <image-reference>`。

## `aenv snapshot list`

列出持久 Snapshot。别名：`aenv snapshot ls`、`aenv snap ls`。

```bash
aenv snapshot list
aenv snapshot list --sandbox-id <sandbox-id>
```

| 标志 | 描述 |
|------|-------------|
| `--sandbox-id <id>` | 按源 Sandbox ID 过滤 Snapshot |
| `--output <table\|json>` | 输出格式。在 TTY 上默认为 table，重定向时默认为 JSON。 |

table 输出包含 `IMAGE REF` 列（未发布镜像时为 `-`）；JSON 输出包含可选的 `imageRef` 字段。

要删除 Snapshot，请使用 `aenv template delete <snapshot-id>` 或 `aenv template delete <name>`——Snapshot 与 Template 共享相同的底层存储，并通过同一命令删除。
