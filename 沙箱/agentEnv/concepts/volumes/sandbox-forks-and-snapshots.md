# 沙盒分叉与快照中的 Volume

AgentENV 会在 Sandbox 分叉和 Snapshot 操作中自动携带已挂载的 Volume。你无需单独对每个已挂载的 Volume 进行分叉或 Snapshot。

## 分叉 Sandbox

当 Sandbox 被分叉时，AgentENV 会为每个子 Sandbox 处理每一个已挂载的 Volume：

- 每个已挂载的 `exclusive` Volume 都会获得一个独立的写时复制 Volume 分叉。子 Sandbox 会在相同的 guest 路径挂载新 Volume，并可在不改变源 Sandbox Volume 的情况下对其进行修改。
- 每个已挂载的 `ro` Volume 仍然从同一个只读 Volume 挂载。由于源 Sandbox 和子 Sandbox 都无法修改它，因此共享是安全的。

此行为与使用 `aenv volume create --from-volume` 手动创建可复用 Volume 分叉是不同的。

## 对 Sandbox 创建 Snapshot

当对 Sandbox 创建 Snapshot 时，每个已挂载的 Volume 都会被自动包含在内。Volume Snapshot 会记录其各层、大小、挂载路径和访问模式。从该 Snapshot 启动 Sandbox 时，会使用捕获到的内容创建新的 Volume，并将它们挂载到相同的路径。`exclusive` Volume 仍为 `exclusive`；只读 Volume 仍为只读。

SandboxSnapshot 中捕获的 Volume 挂载在启动时无法被替换。如果 Snapshot 包含 Volume 挂载，提供 `volumeMounts`（包括空数组）会返回 HTTP 400。省略该字段即可恢复捕获的 Volume。模板仍然接受显式挂载，不包含 Volume 挂载的 Snapshot 也是如此。

Volume Snapshot 独立于其源 Volume。删除源 Volume 不会移除已提交到 SandboxSnapshot 中的 Volume 数据。
