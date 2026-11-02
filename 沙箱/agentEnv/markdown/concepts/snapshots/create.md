# 从运行中的 Sandbox 创建 Snapshot

捕获运行中 Sandbox 的当前状态：

```bash
aenv snapshot create <sandbox-id>
aenv snapshot create <sandbox-id> --name my-checkpoint
```

| 参数或选项 | 默认值 | 描述 |
| --- | --- | --- |
| `<sandbox-id>` | 必填 | 要捕获的运行中 Sandbox 的 ID。 |
| `--name <name>` | 无 | 分配一个人类可读的别名。如果省略，请使用命令返回的生成 Snapshot ID。 |

创建 Snapshot 后，源 Sandbox 会继续运行。继续阅读 [使用 Snapshot](./use.md) 以恢复它或将其 rootfs 发布为 OCI 镜像。

之后你可以将 Snapshot ID 或其别名传给 `aenv start`：

```bash
aenv start my-checkpoint
# Or:
aenv start <snapshot-id>
```

这会创建一个具有新 Sandbox ID 的独立 Sandbox。它继承捕获的文件系统、运行中的进程、内存状态、环境变量、运行时配置和资源设置。
