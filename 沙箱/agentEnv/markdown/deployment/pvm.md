# PVM 部署

> [!NOTE]
> **在标准 KVM 不可用时使用本指南**，这通常发生在未暴露嵌套虚拟化的云 VM 上。如果标准 KVM 已经可用，请改用 [快速开始](../getting-started/quickstart.md)。

> [!WARNING]
> 此功能为**实验性**。PVM 功能尚未合并到 Linux 主线内核，分叉内核可能不会获得与主线内核相同级别的测试和安全更新。

PVM 最初在论文 [*PVM: Efficient Shadow Paging for Deploying Secure Containers in Cloud-native Environment*](https://dl.acm.org/doi/10.1145/3600006.3613158) 中提出，是一种替代虚拟化模式，可以在不依赖传统嵌套虚拟化的情况下提供 AgentENV 所需的 KVM 兼容接口。安装 PVM 主机环境后，AgentENV 仍使用 `/dev/kvm` 创建 Firecracker microVM。

与普通 KVM 部署相比，PVM 部署增加两个主机级步骤：

1. 安装并启动支持 PVM 的主机内核。
2. 在启动 AgentENV 之前加载 PVM 虚拟化模块。

随后 AgentENV 会使用其 PVM 专用的 Firecracker 和客户机内核 artifact。

## 开始之前

PVM 不能仅通过更改 AgentENV 配置值来启用。必须先使用兼容的 PVM 内核准备好主机。

你需要：

- 一台 x86_64 Linux 服务器。
- Root 访问权限。
- 安装主机内核并重启服务器的权限。
- 由已发布 PVM 主机内核包支持的基于 DEB 或基于 RPM 的 Linux 发行版，或从源码构建内核的能力。
- 满足 AgentENV 其余要求的 Linux 内核 6.8 或更新版本。

AgentENV **不会**自动替换正在运行的主机内核。预构建的 PVM 主机内核包单独发布在 [`kvcache-ai/linux` releases](https://github.com/kvcache-ai/linux/releases) 中。在安装 AgentENV 之前，你必须安装合适的包、重启进入该内核并验证 PVM 模块。

> [!WARNING]
> 在生产服务器上更改内核之前，请确认你具有控制台访问权限或其他恢复路径，以防新内核无法启动。

## 主机与客户机内核兼容性

PVM 主机内核和 AgentENV microVM 内运行的内核必须使用兼容的 PVM ABI。不兼容的组合可能会阻止客户机启动或导致意外的运行时故障。

为获得最可预测的设置，请使用由同一 PVM 内核版本构建的主机和客户机内核。AgentENV 打包的 PVM 客户机内核基于 [`virt-pvm/linux` 的 `pvm-612` 分支](https://github.com/virt-pvm/linux/tree/pvm-612)，Linux 版本为 **6.12.33**。匹配的预构建主机包发布在 [`kvcache-ai/linux` release `pvm-kernel-6.12.33`](https://github.com/kvcache-ai/linux/releases/tag/pvm-kernel-6.12.33)。

主机和客户机需要不同的内核配置选项：

- **主机内核：** 启用 `CONFIG_KVM_PVM=m`。
- **客户机内核：** 启用 `CONFIG_PVM_GUEST`。

## PVM 如何融入 AgentENV

一个 AgentENV 节点只会以一种虚拟化模式运行：

| 模式 | AgentENV 设置 | 主机状态 |
|------|------------------|------------|
| 标准 KVM | `virtualization_mode = "kvm"` | 标准 KVM 模块；未加载 `kvm_pvm` |
| PVM | `virtualization_mode = "pvm"` | 支持 PVM 的主机内核，已加载 `kvm_pvm` |

这些模式被有意隔离：

- 依赖供应只安装所选模式的 Firecracker 和客户机内核。
- 快照记录其捕获时的模式。
- 持久化的暂停沙盒记录其模式。
- 节点拒绝恢复在另一模式下创建的状态。

不要将 KVM 和 PVM 节点指向同一个持久化 Sandbox 目录。如果节点共享 Snapshot 仓库，请确保工作负载只在创建 Snapshot 时所用模式的节点上恢复。

## 步骤 1：安装支持 PVM 的主机内核

使用适合你的发行版的包格式。AgentENV 只需要内核镜像和模块包。单独发布的 headers/开发包不是必需的，除非你需要在主机上构建外部内核模块。

### Debian 和 Ubuntu

下载内核镜像：

```bash
curl -fLO \
  https://github.com/kvcache-ai/linux/releases/download/pvm-kernel-6.12.33/linux-image-6.12.33_6.12.33-7_amd64.deb
```

安装并刷新引导加载程序：

```bash
sudo dpkg -i linux-image-6.12.33_6.12.33-7_amd64.deb
sudo update-grub
```

如果另一个已安装内核具有更高版本，请在重启前从引导加载程序的 advanced options 中选择 Linux `6.12.33`，或将其配置为默认启动项。

### 基于 RPM 的发行版（Fedora、RHEL、CentOS、TencentOS）

下载内核包：

```bash
curl -fLO \
  https://github.com/kvcache-ai/linux/releases/download/pvm-kernel-6.12.33/kernel-6.12.33_g91e9c9be4472-2.x86_64.rpm
```

安装它：

```bash
sudo rpm -ivh --oldpackage kernel-6.12.33_g91e9c9be4472-2.x86_64.rpm
```

在使用 `grubby` 的系统上，选择已安装的 PVM 内核：

```bash
sudo grubby --set-default /boot/vmlinuz-6.12.33-g91e9c9be4472
sudo grubby --default-kernel
```

### 从源码构建

如果已发布的包与发行版不兼容，请从 [`pvm-612` 分支](https://github.com/kvcache-ai/linux/tree/pvm-612) 构建主机内核。启用 `CONFIG_KVM_PVM=m`，安装内核和模块，并根据发行版的内核构建文档配置引导加载程序。

### 重启并验证

重启主机：

```bash
sudo reboot
```

重新连接后，确认预期内核处于活动状态：

```bash
uname -r
```

预期输出：

```text
# DEB package
6.12.33

# RPM package
6.12.33-g91e9c9be4472
```

如果 `uname -r` 报告的是之前的内核，请更新引导加载程序选择并再次重启，然后继续。

## 步骤 2：加载并验证 PVM 模块

加载模块：

```bash
sudo modprobe kvm_pvm
```

验证其已加载：

```bash
lsmod | grep kvm_pvm
test -d /sys/module/kvm_pvm
```

验证 KVM 兼容设备现已可用：

```bash
ls -l /dev/kvm
```

主机设置的重要结果是：

- `/sys/module/kvm_pvm` 存在。
- `/dev/kvm` 存在。
- AgentENV 运行时账户可以读写打开 `/dev/kvm`。

确认模块成功加载后，配置内核在启动时自动加载它（不同发行版可能使用不同目录）：

```bash
echo kvm_pvm | sudo tee /etc/modules-load.d/kvm-pvm.conf
```

## 步骤 3：以 PVM 模式安装 AgentENV

### 方式 A：安装脚本

在 Ubuntu 24.04 上：

```bash
curl -fsSL https://raw.githubusercontent.com/kvcache-ai/AgentENV/main/scripts/install.sh \
  | sudo AENV_VIRTUALIZATION_MODE=pvm bash

sudo systemctl start aenv
```

安装程序会：

- 下载 `aenv-server-linux-x86_64-pvm.tar.gz`。
- 安装 PVM Firecracker 和客户机内核 artifact。
- 将 `AENV_VIRTUALIZATION_MODE="pvm"` 写入 `/etc/default/aenv`。
- 配置服务账户对 `/dev/kvm` 的访问。

它不会安装或加载 PVM 主机内核或 `kvm_pvm`。

### 方式 B：Docker

使用专用 PVM 镜像：

```bash
docker pull ghcr.io/kvcache-ai/aenv-server:latest-pvm

docker run --rm -it --name aenv-server \
  --device /dev/kvm \
  --privileged \
  -v /dev:/dev \
  -p 8000:8000 \
  ghcr.io/kvcache-ai/aenv-server:latest-pvm
```

该镜像默认设置 `AENV_VIRTUALIZATION_MODE=pvm`，且仅包含 PVM 运行时 artifact。

### 方式 C：从源码构建

为依赖供应和服务器启动都选择 PVM：

```bash
export AENV_VIRTUALIZATION_MODE=pvm

cargo run --bin server -- --setup-only
make start-server
```

服务器会在首次正常启动时在 `$AENV_HOME/secrets/api-key` 下生成并持久化 API key。

你也可以在 TOML 配置中设置模式：

```toml
virtualization_mode = "pvm"
```

环境变量优先于 TOML 值。

内存 Snapshot 脏页跟踪对 KVM 默认启用，并在 PVM 模式下自动禁用，因为此组合尚未测试。

要构建 PVM Docker 镜像：

```bash
docker build \
  --build-arg AENV_VIRTUALIZATION_MODE=pvm \
  -f deploy/docker/Dockerfile.agentenv \
  -t aenv:pvm .
```

## 步骤 4：验证 AgentENV

对于安装脚本部署，验证持久化的模式：

```bash
grep AENV_VIRTUALIZATION_MODE /etc/default/aenv
```

预期输出：

```text
AENV_VIRTUALIZATION_MODE="pvm"
```

检查启动状态和日志：

```bash
sudo systemctl status aenv
sudo journalctl -u aenv -f
```

验证 API：

```bash
curl http://127.0.0.1:8000/health
```

服务器健康后，模板创建和 Sandbox 操作与标准 [快速开始](../getting-started/quickstart.md) 相同。

## 多节点部署

在运行时池中使用一致的虚拟化模式。多节点部署中的 Gateway 和 Scheduler JSON 设置，请参见 [Gateway 与 Scheduler 配置](../configuration/control-plane-config.md)。

对于 Docker Compose：

- 使用 PVM 运行时镜像。
- 导出 `AENV_VIRTUALIZATION_MODE=pvm`。
- 在启动 Compose 栈之前准备主机。

对于 Kubernetes：

- 为启动 PVM 内核的 x86_64 worker 节点打标签。
- 在每个选定的节点上加载 `kvm_pvm`。
- 使用 PVM AgentENV 镜像。
- 在运行时 DaemonSet 中设置 `AENV_VIRTUALIZATION_MODE=pvm`。
- 添加节点选择器或亲和性规则，使 PVM Pod 无法运行在 KVM 节点上。

避免在从共享 Snapshot 集调度的池中混合 KVM 和 PVM 节点，除非 Scheduler 也强制执行虚拟化模式亲和性。

## 故障排查

### `PVM virtualization mode is only supported on x86_64 hosts`

当前机器架构不受支持。请在 x86_64 服务器上部署 PVM 节点或使用标准 KVM。

### `PVM mode requires the kvm_pvm host module to be loaded`

AgentENV 模式设置为 PVM，但主机模块未激活。

检查正在运行的内核并尝试加载模块：

```bash
uname -r
sudo modprobe kvm_pvm
lsmod | grep kvm_pvm
```

如果 `modprobe` 报告找不到模块，则服务器未运行兼容的 PVM 主机内核。

### 加载 `kvm_pvm` 后缺少 `/dev/kvm`

确认 `kvm_pvm` 成功加载并查看内核日志：

```bash
lsmod | grep kvm_pvm
sudo dmesg | tail -n 100
```

如果模块已加载但 `/dev/kvm` 仍不存在，请与内核提供商一起验证 PVM 内核安装和启动参数。

### `/dev/kvm` 不可访问

检查所有权和服务账户组：

```bash
ls -l /dev/kvm
id aenv
```

更改组成员身份后，请重启服务或用户会话。

### AgentENV 下载或使用标准 KVM artifact

确认服务环境中存在该模式：

```bash
grep AENV_VIRTUALIZATION_MODE /etc/default/aenv
```

然后重新运行供应：

```bash
sudo AENV_VIRTUALIZATION_MODE=pvm \
  AENV_CONFIG_PATH=/var/lib/aenv/config/config.toml \
  AENV_HOME_PATH=/var/lib/aenv \
  /usr/local/bin/server --setup-only
```

### Snapshot 或暂停 Sandbox 模式不匹配

持久化状态是在另一种虚拟化模式下创建的。请在其原始模式的节点上恢复它，或在目标模式下重建工作负载并捕获新 Snapshot。
