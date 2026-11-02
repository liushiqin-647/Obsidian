# 将 Snapshot Rootfs 用作 OCI 镜像

捕获 Snapshot 并使用 `aenv start <snapshot>` 启动它会恢复完整的 Snapshot 状态。如果你只需要捕获的根文件系统，也可以将其发布为 OCI 镜像，并从该镜像冷启动 Sandbox。

## 在创建 Snapshot 时发布镜像

每当你创建 Snapshot 时，AgentENV 都可以自动创建并发布该 Snapshot rootfs 的 OCI 镜像。对于由 OSS 支持并从 OverlayBD 原生 OCI 镜像创建的 Snapshot，请在 AgentENV 配置文件（`config/default.toml`，或由 `AENV_CONFIG_PATH` 选定的文件）中启用自动发布：

```toml
[snapshot]
repository_backend = "oss"

[snapshot.image_publish]
enabled = true
```

创建 Snapshot：

```bash
aenv snapshot create <sandbox-id> --name my-checkpoint
```

发布成功时，命令会打印已发布的镜像引用。使用该引用冷启动 Sandbox：

```bash
aenv start --cold registry.example.com/team/app:agentenv-snapshot-<snapshot-id>
```

## 导出已有 Snapshot 的 Rootfs

你可以手动将已有 Snapshot 的 rootfs 导出为 OCI 镜像。这通过 `aenv-snapshot-image` 完成，它不包含在常规的 AgentENV 安装包中，必须先从仓库构建并安装：

```bash
git clone https://github.com/kvcache-ai/AgentENV.git
cd AgentENV
make build-snapshot-image
sudo install -m 0755 target/debug/aenv-snapshot-image /usr/local/bin/aenv-snapshot-image
```

导出 rootfs：

```bash
aenv-snapshot-image <snapshot-id-or-alias> \
  --target-repository registry.example.com/team/app \
  --tag release-1
```

| 参数或选项 | 默认值 | 描述 |
| --- | --- | --- |
| `<snapshot-id-or-alias>` | 必填 | 要导出其 rootfs 的 Snapshot。 |
| `--target-repository <registry/repository>` | 从 Snapshot 推断 | 目标 OCI 仓库。当无法推断出唯一源仓库时请指定它。 |
| `--tag <tag>` | 显式指定目标时为 `latest`；否则为 `snapshot-<snapshot-id>` | 导出镜像的标签。 |
| `--config <path>` | `AENV_CONFIG_PATH`，然后是默认配置路径 | 用于定位 Snapshot 仓库的 AgentENV 配置。 |

导出的镜像可以存储在 OCI 注册表中，独立于 AgentENV Snapshot 仓库进行共享，并用于冷启动新的 Sandbox。新 Sandbox 会继承导出的根文件系统和 OCI 运行时配置，但不会继承 Snapshot 的内存或运行中进程状态。

你可以捕获它并从生成的镜像启动 Sandbox：

```bash
aenv-snapshot-image <snapshot-id-or-alias> \
  --target-repository registry.example.com/team/app \
  --tag release-1

aenv start --cold <image_ref>
```
