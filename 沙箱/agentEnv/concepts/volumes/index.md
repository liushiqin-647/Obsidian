# Volume

> **Beta：**Volume 功能仍处于 beta 阶段。对于可以使用仅在冷启动沙盒时提供的驱动器的负载，`POST /sandboxes-cold` 上的 `attachedDrives` 功能经过了更充分的测试。额外的驱动器请求 schema 请参阅 [API 参考](../../api/index.md)。

Volume 是与 Sandbox 独立管理的持久块文件系统。Volume 具有稳定的 ID、唯一名称、固定大小和访问模式。你可以在创建 Sandbox 时将其挂载到绝对 guest 路径。

删除 Sandbox 不会删除其 Volume。在停止运行中的 Sandbox 之前，AgentENV 会提交其可写 Volume 的最新写入，以便之后可由另一个 Sandbox 挂载。如果无法安全完成此提交，AgentENV 会阻止挂载不完整的 Volume 数据。

暂停则不同：它会保留完整的 Sandbox 状态（包括内存和根文件系统），以便之后恢复。

默认 Volume 大小为 65536 MiB（64 GiB）。默认情况下，一个 Sandbox 最多可挂载四个 Volume，每个 Volume 最大为 262144 MiB（256 GiB）。管理员可以在 [`[volume]` 配置](../../configuration/reference.md#volume)中更改这些限制。

---

接下来：

- [创建与挂载 Volume](./create-and-mount.md) — 创建空 Volume、分叉 Volume 或由镜像支撑的 Volume，并在 Sandbox 中挂载它们。
- [管理 Volume](./manage.md) — 列出、检查并删除 Volume。
- [Sandbox 分叉与快照](./sandbox-forks-and-snapshots.md) — 了解已挂载 Volume 在 Sandbox 分叉和 Snapshot 操作中的行为。
