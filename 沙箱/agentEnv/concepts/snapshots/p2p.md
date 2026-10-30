# 可选的 P2P 可见性

P2P 可见性使节点能够从对等节点发现并获取已提交的 Snapshot 产物。Snapshot 仓库仍是持久的真实来源，因此 P2P 是一种可选的分发路径，而不是 Snapshot 存储的替代品。

在 AgentENV 配置文件（`config/default.toml`，或由 `AENV_CONFIG_PATH` 选定的文件）中同时启用节点级 P2P 传输和 Snapshot 发布：

```toml
[p2p]
enabled = true

[snapshot]
p2p_enabled = true
```

有关可选的传输、存储目录、地址和超时设置，请参见 [运行时节点配置](../../configuration/reference.md#p2p)。
