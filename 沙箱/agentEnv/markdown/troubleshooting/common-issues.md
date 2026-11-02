# 常见问题

## `/dev/kvm` 缺失或无法访问

**现象**：服务器找不到或无法打开 `/dev/kvm`。

**解决方案**：首先检查标准 KVM 是否可用：

```bash
ls -l /dev/kvm
```

如果设备存在，请确保运行时用户对其具有读写权限。在大多数系统上：

```bash
sudo usermod -aG kvm $USER
# Log out and back in for the group change to take effect
```

如果云服务器不提供标准 KVM，请遵循 [PVM Deployment](../deployment/pvm.md)。该指南涵盖了在启动 AgentENV 之前所需的额外主机设置。

## 配置的虚拟化模式与主机不匹配

**现象**：启动时报告在加载 PVM 模块的情况下 KVM 无法运行，或报告 PVM 需要额外的主机设置。

**解决方案**：正常安装应使用默认的 KVM 模式。如果主机已为 PVM 做好准备，请遵循 [PVM Deployment](../deployment/pvm.md)，并确保服务环境中包含：

```bash
AENV_VIRTUALIZATION_MODE=pvm
```

## 网络操作权限被拒绝

**现象**：创建 sandbox 时因 network namespace 或 iptables 错误而失败。

**解决方案**：服务器在其 effective、permitted 和 inheritable 权限集中都需要 `CAP_NET_ADMIN` 和 `CAP_SYS_ADMIN`。安装的 systemd unit 会自动配置这些权限。对于源码检出，请使用 `make start-server` 或 `scripts/run-with-capabilities.sh <server-binary>`；不要以 root 身份运行整个服务器。同时请确认运行时账户属于 `kvm` 组、能够打开 `/dev/kvm`，并且能够打开 `/dev/ublk-control`。

## `ip netns list` 中缺少 Sandbox 命名空间

**现象**：Sandbox 正在运行，但 `ip netns list` 不显示其网络命名空间，且 `ip netns exec <name>` 找不到它们。

**解决方案**：AgentENV 将命名空间挂载点存储在 `$AENV_RUNTIME_PATH/netns` 下，而不是 `/var/run/netns`；安装的服务使用 `/run/aenv/netns`。请直接检查该目录，需要时按路径进入命名空间：

```bash
sudo nsenter --net=/run/aenv/netns/agentenv-ns-<slot> ip addr
```

## 找不到配置文件

**现象**：`Error: config file not found`

**解决方案**：服务器默认查找 `config/default.toml`。请从仓库根目录运行，或设置 `AENV_CONFIG_PATH`：

```bash
export AENV_CONFIG_PATH=/path/to/your/config.toml
```

## 端口已被占用

**现象**：启动服务器时出现 `Address already in use`。

**解决方案**：另一个进程正在使用 8000 端口。请停止该进程或更改监听地址：

```bash
API_ADDR=0.0.0.0:8001 make start-server
```

## Sandbox 创建超时

**现象**：`POST /sandboxes` 返回超时错误。

**解决方案**：检查运行时资产（Firecracker 二进制、kernel、rootfs）是否已下载。服务器会在首次启动时自动准备这些资产，但网络问题可能导致失败。运行 `cargo run --bin server -- --setup-only` 可手动准备并查看详细错误。

同时检查配置中的 `[envd].init_timeout_secs`。默认值为 60 秒。如果 rootfs 镜像较大，客户机内的 envd daemon 可能需要更多时间才能完成初始化。

> TODO：随着更多常见问题被报告，将在此处补充。
