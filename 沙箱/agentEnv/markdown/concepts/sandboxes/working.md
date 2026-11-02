# 使用 Sandbox

本页提供与 Sandbox 交互和管理 Sandbox 的基本命令。

## 连接到 Sandbox

`aenv connect` 会在 Sandbox 内打开一个交互式 shell 并附加你的终端。`aenv cn` 是其简短别名：

```bash
aenv connect <sandbox-id>
aenv cn <sandbox-id>
```

如果 Sandbox 已暂停，`aenv connect` 会自动恢复它。

## 执行命令

`aenv exec` 运行一条非交互式命令，将其输出流式传输到你的本地终端，并以远程命令的退出码退出。它不会附加交互式 shell。如果为远程命令准备的标志与 aenv 自身的标志冲突，可以在前面加上 `--` 进行转义。

```bash
aenv exec <sandbox-id> ls -la /
aenv exec <sandbox-id> -- command-with-aenv-like-flags --timeout 10
```

## 上传文件

`aenv upload` 将本地文件或目录复制到运行中的 Sandbox：

用法：

```bash
aenv upload <sandbox-id> <local-path> <remote-path> [options]
```

示例：

```bash
aenv upload 018f0d93-aaaa-bbbb-cccc-0123456789ab ./config.json /workspace/config.json
```

| 参数或选项 | 默认值 | 描述 |
| --- | --- | --- |
| `<sandbox-id>` | 必填 | 目标 Sandbox 的 ID。 |
| `<local-path>` | 必填 | 要上传的本地文件或目录。 |
| `<remote-path>` | 必填 | Sandbox 内的目标位置。目录路径必须为绝对路径。 |
| `--user <user>` | 无 | 从该用户的主目录解析相对远程文件路径，并设置上传文件的所有者。不支持目录上传。 |

## 下载文件

`aenv download` 将文件或目录从运行中的 Sandbox 复制到你的本地机器：

用法：

```bash
aenv download <sandbox-id> <remote-path> [local-path] [options]
```

示例：

```bash
aenv download 018f0d93-aaaa-bbbb-cccc-0123456789ab /workspace/result.txt ./result.txt
```

| 参数或选项 | 默认值 | 描述 |
| --- | --- | --- |
| `<sandbox-id>` | 必填 | 要从中下载的 Sandbox 的 ID。 |
| `<remote-path>` | 必填 | Sandbox 内的文件或目录。目录路径必须为绝对路径。 |
| `[local-path]` | 当前目录 | 本地目标文件或目录。 |
| `--user <user>` | 无 | 从该用户的主目录解析相对远程文件路径。不支持目录下载。 |
| `--force` | 禁用 | 替换冲突的本地文件。不带它时，下载会停止而不会覆盖它们。 |

## 暂停和恢复

暂停会保存 Sandbox 的当前运行时状态并停止其 microVM。暂停期间，其中的程序不会运行，服务不会处理请求，Sandbox 会释放其 CPU 和内存资源。其保存的状态保留在存储中，以便之后恢复同一个 Sandbox。

恢复后，文件系统、运行中的进程、环境变量和内存中的数据都会恢复到暂停时捕获的状态。程序会从该保存状态继续，而不是从头开始。

```bash
aenv pause <sandbox-id>
aenv resume <sandbox-id>
aenv resume <sandbox-id> --timeout 600
```

`aenv resume` 接受 `--timeout <seconds>`，默认值为 300 秒，并从恢复时刻起设置新的 TTL。

默认情况下，Sandbox 在达到其 TTL 时会自动暂停。有关如何设置截止时间以及如何改为删除的信息，请参见 [自动驱逐](./auto-eviction.md)。

## 持久化快照

Snapshot 是运行中 Sandbox 的持久、可复用检查点。创建 Snapshot 不会替换 Sandbox：捕获后源 Sandbox 返回 Running，而 Snapshot 之后可以启动一个或多个新的 Sandbox。

```bash
aenv snapshot create <sandbox-id>
aenv snapshot create <sandbox-id> --name my-base
```

生成的 Snapshot 会出现在 `aenv snapshot list` 中，并可通过 `aenv start <snapshot-id-or-name>` 启动。有关其参数和生命周期，请参见 [Snapshots](../snapshots/index.md)。

## Fork

Fork 会将运行中的 Sandbox 克隆为同一节点上的独立子 Sandbox。捕获其状态时源 Sandbox 会短暂暂停，然后返回 Running。子 Sandbox 继承源的文件系统、内存、网络策略、安全模式以及 CPU/内存/磁盘配置。所有子 Sandbox 使用一个捕获的状态，但每个子 Sandbox 可以独立成功或失败。

```bash
curl -X POST \
  -H 'X-API-Key: test-key' \
  -H 'Content-Type: application/json' \
  -d '{"count": 3, "timeout": 600}' \
  http://127.0.0.1:8000/sandboxes/<sandbox-id>/fork
```

| 字段 | 默认值 | 描述 |
|---|---|---|
| `count` | `1` | 要创建的子 Sandbox 数量；最小 1，最大 100。 |
| `timeout` | 源 Sandbox 的 TTL 时长 | 每个子 Sandbox 的 TTL，从 fork 时刻开始计算。 |

成功的请求会返回一个数组，为每个请求的子 Sandbox 包含一个结果。每个条目要么包含一个 `sandbox` 对象——包括其 `sandboxID`——要么包含一个解释该单个子 Sandbox 为何失败的 `error`。它不是单纯的 ID 列表。非 201 响应表示请求在尝试任何子 Sandbox 之前就已失败。完整的 fork 请求和响应模式请参见 [API 参考](../../api/index.md)。

## 查看资源指标

获取某个 Sandbox 保留的指标：

```bash
curl -H "X-API-Key: $AENV_API_KEY" \
  "$AENV_URL/sandboxes/$SANDBOX_ID/metrics?start=1700000000&end=1700003600"
```

`start` 和 `end` 是可选的 Unix 秒级时间戳，定义包含两端的时间范围。省略时，该端点会按时间戳顺序返回该 Sandbox 所有保留的样本。

获取多个 Sandbox 的最新指标：

```bash
curl -H "X-API-Key: $AENV_API_KEY" \
  "$AENV_URL/sandboxes/metrics?sandbox_ids=$SANDBOX_ID,$OTHER_SANDBOX_ID"
```

`sandbox_ids` 为必填，接受逗号分隔的最多 100 个唯一 Sandbox ID 列表。响应包含每个请求的运行中 Sandbox 的最新可用指标。没有可用指标的 Sandbox 会被省略。

每个指标包含：

| 字段 | 描述 |
| --- | --- |
| `timestampUnix` | 样本时间，Unix 秒。 |
| `cpuCount` | CPU 核心数。 |
| `cpuUsedPct` | CPU 使用百分比。 |
| `memUsed` | 已用内存，字节。 |
| `memTotal` | 总内存，字节。 |
| `memCache` | 缓存内存，字节。 |
| `diskUsed` | 已用磁盘空间，字节。 |
| `diskTotal` | 总磁盘空间，字节。 |

完整的请求、响应和错误模式请参见 [API 参考](../../api/index.md)。

在配置文件的 `[orchestrator]` 下配置采集：

```toml
[orchestrator]
metrics_interval_secs = 15
metrics_retention_secs = 3600
```

`metrics_interval_secs` 设置采集间隔（秒）；`0` 禁用采集。`metrics_retention_secs` 设置样本的保留时长。

## 管理 Sandbox

列出 Sandbox：

```bash
aenv list  # alias: aenv ls
aenv list --output json
```

`--output` 接受 `table` 或 `json`。在交互式终端中默认为表格，当输出被管道传输或重定向时默认为 JSON。

删除 Sandbox：

```bash
aenv delete <sandbox-id>  # alias: aenv rm
```

删除是永久性的，但之前从该 Sandbox 创建的 Snapshot 不受影响。
