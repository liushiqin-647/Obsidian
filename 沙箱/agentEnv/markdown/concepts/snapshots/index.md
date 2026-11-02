# Snapshots（快照）

Snapshot 是 Sandbox 的可复用检查点。它保留 Sandbox 的文件系统和运行时状态，以便你之后可以从同一点启动新的 Sandbox，而无需重新构建环境并重新执行设置工作。

- **Templates** 以 Snapshot 形式存储。Template 构建会提交一个 Snapshot；Template ID 是解析到该 Snapshot 的别名。
- **热启动的 Sandbox** 会恢复已提交 Template 或 Snapshot 的状态。
- **运行中的 Sandbox** 可以生成新的 Snapshot，捕获其当前状态以供之后重用或分支。

## Snapshot 保留的内容

Snapshot 保留：

- VM 状态和内存，使运行中的进程能够从捕获点继续。
- 根文件系统及其更改。
- 附加驱动器的内容和附加元数据。
- 已挂载 Volume 的内容、挂载路径、大小和访问模式。从该 Snapshot 启动会从捕获的内容创建新的 Volume，而不是重用源 Volume 的身份。
- CPU、内存和磁盘设置，以及命令上下文，例如环境变量、工作目录、用户和启动命令。

---

后续阅读：

- [创建 Snapshot](./create.md) — 从运行中的 Sandbox 捕获可复用检查点。
- [管理 Snapshot](./manage.md) — 列出、检查和删除 Snapshot。
- [将 Rootfs 作为 OCI 镜像](./use.md) — 仅发布或导出捕获的根文件系统。
- [可选的 P2P 可见性](./p2p.md) — 加速已提交 Snapshot 产物在节点之间的分发。
