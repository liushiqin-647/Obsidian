# 沙盒

## `aenv start <target>`

启动一个 Sandbox（Sandbox）并附加交互式 shell。`<target>` 接受 Template（模板）或 Snapshot（快照）的 ID 或别名，或配合 `--cold` 的 OCI 镜像引用。

```bash
aenv start my-ubuntu
aenv start --cold ubuntu:24.04              # start directly from an OCI image
```

由 `aenv` 启动的 Sandbox 始终要求经过 token 认证的 envd 访问。CLI 会自动获取并管理访问令牌。

| 标志 | 描述 |
|------|-------------|
| `--cold` | 直接从外部 OCI 镜像启动，而不是从 Template 或 Snapshot 启动 |
| `--timeout <secs>` | Sandbox TTL（秒）（默认：300） |
| `--cpu <count>` | CPU 核心数；仅在与 `--cold` 一起使用时有效。默认使用服务器上的 `[machine].vcpu_count`。别名：`--cpu-count`。 |
| `--memory <MiB>` | 内存大小（MiB）；仅在与 `--cold` 一起使用时有效。默认使用服务器上的 `[machine].mem_size_mib`。别名：`--memory-mb`、`--mem`。 |
| `--disk-size-mb <MiB>` | 根文件系统大小；仅在与 `--cold` 一起使用时有效。默认使用源镜像的虚拟大小；显式值必须至少为 1024 且能被 1024 MiB 整除。别名：`--disk-mb`。 |
| `--volume <MOUNT_PATH=VOLUME>` | 通过 ID 或名称挂载持久卷。可用于热启动和冷启动；可重复。 |
| `--image-mount <MOUNT_PATH=IMAGE>` | 从镜像附加一个可写驱动器。可用于热启动和冷启动；可重复。参见[附加镜像驱动器](../../concepts/sandboxes/starting.md#attach-image-drives)。 |
| `--image-mount-ro <MOUNT_PATH=IMAGE>` | 从镜像附加一个只读驱动器。可用于热启动和冷启动；可重复。 |
| `-d, --detach` | 打印 Sandbox ID 并退出，不附加 shell |

CPU、内存和磁盘覆盖仅支持冷启动。

## `aenv pause <sandbox-id>`

暂停一个运行中的 Sandbox。Sandbox 状态会被保留，之后可以恢复。

```bash
aenv pause <sandbox-id>
```

## `aenv resume <sandbox-id>`

恢复一个已暂停的 Sandbox。

```bash
aenv resume <sandbox-id>
```

| 标志 | 描述 |
|------|-------------|
| `--timeout <secs>` | 从现在起的 TTL（秒）（默认：300）。必须长于 Sandbox 当前剩余的 TTL。 |

## `aenv timeout <sandbox-id> <seconds>`

将 Sandbox 过期时间设置或延长为从现在起的 `<seconds>` 秒。

```bash
aenv timeout <sandbox-id> 600
```

## `aenv connect <sandbox-id>`

向运行中或已暂停的 Sandbox 附加交互式 shell。别名：`aenv cn`。

```bash
aenv connect <sandbox-id>
```

如果 Sandbox 已暂停，附加前会先恢复它。

## `aenv exec <sandbox-id> <command> [args...]`

在 Sandbox 中运行一次性命令并流式输出其结果。

当远程命令的标志可能被解释为 `aenv` 标志时，请使用前导 `--`。

```bash
aenv exec <sandbox-id> ls -la /
aenv exec <sandbox-id> -- command-with-aenv-like-flags --timeout 10
```

## `aenv upload <sandbox-id> <local-path> <remote-path>`

通过 envd 将本地文件或目录上传到 Sandbox。文件会逐个流式传输，缺失的远程目录会自动创建。

```bash
aenv upload <sandbox-id> ./config.json /workspace/config.json
aenv upload <sandbox-id> ./config.json /workspace/
aenv upload <sandbox-id> ./project /workspace/
aenv upload <sandbox-id> ./project /workspace/app
aenv upload --user app <sandbox-id> ./config.json config.json
```

| 标志 | 描述 |
|------|-------------|
| `--user <user>` | 以该用户身份解析相对远程文件路径，并设置上传文件的所有者 |

对于目录上传，远程路径必须是绝对路径，且不支持 `--user`。如果远程目标以 `/` 结尾或已作为目录存在，则会追加本地目录名。否则，目标将用作新的目录根。会复制隐藏文件和空目录；符号链接和特殊文件会被拒绝。

上传仅复制文件内容和目录结构。它**不会**保留主机所有权或组、权限（包括可执行位）、时间戳、ACL、扩展属性或硬链接关系。目标元数据由 envd 和 Sandbox 文件系统分配。

## `aenv download <sandbox-id> <remote-path> [local-path]`

通过 envd 从 Sandbox 下载文件或目录。

```bash
aenv download <sandbox-id> /workspace/result.txt ./result.txt
aenv download <sandbox-id> /workspace/result.txt
aenv download <sandbox-id> /workspace/result.txt ./output/
aenv download <sandbox-id> /workspace/project ./backup/
aenv download --user app --force <sandbox-id> result.txt ./result.txt
```

| 标志 | 描述 |
|------|-------------|
| `--user <user>` | 从该用户的主目录解析相对远程文件路径 |
| `--force` | 替换冲突的本地文件 |

当省略本地路径时，将在当前目录中使用远程名称。当本地路径指定现有目录或以 `/` 结尾时，会自动追加远程名称。生成的本地父目录必须已存在。目录下载要求远程路径为绝对路径，且不支持 `--user`。现有目录会被合并；无关文件会保留，而冲突文件需要 `--force`。每个文件都通过临时文件写入，并在该文件成功后才会移动到目标位置。符号链接和特殊文件会被拒绝。下载**不会**保留远程所有权或组、权限（包括可执行位）、时间戳、ACL、扩展属性或硬链接关系。

## `aenv list`

列出所有 Sandbox。别名：`aenv ls`。

```bash
aenv list
```

| 标志 | 描述 |
|------|-------------|
| `--output <table\|json>` | 输出格式。在 TTY 上默认为 table，重定向时默认为 JSON。 |

## `aenv delete <sandbox-id>`

终止并删除一个 Sandbox。别名：`aenv rm`。

```bash
aenv delete <sandbox-id>
aenv rm <sandbox-id>
```
