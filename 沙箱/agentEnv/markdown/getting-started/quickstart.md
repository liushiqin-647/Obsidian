# 快速开始

## 前置条件

- **Linux 内核 6.8+**
- 用于运行 Firecracker microVM 的 `/dev/kvm` 访问权限

> 如果你的服务器不支持标准 KVM，请改用专门的 [PVM 部署](../deployment/pvm.md)指南。

安装脚本会尝试安装缺失的下载和校验和命令，配置 `/dev/kvm` 权限，加载 `ublk_drv` 内核模块，并下载所需的 AgentENV 运行时资源。

安装需要 root 权限，但安装后的服务不以 root 运行。它使用专用的 `aenv` 系统账户，具备 `CAP_NET_ADMIN` 和 `CAP_SYS_ADMIN`，并拥有对 `/dev/kvm` 和 ublk 设备的组访问权限。

## 安装

### 1. 安装并启动服务器

**选项 A — 安装脚本**

该脚本同时安装服务器和 `aenv` CLI。设置 `AENV_HOME_PATH` 可选择数据目录；如果未设置，AENV 会将运行时依赖和数据存储在 `/var/lib/aenv`。安装完成后，以 systemd 服务启动服务器：

```bash
curl -fsSL https://raw.githubusercontent.com/kvcache-ai/AgentENV/main/scripts/install.sh | sudo AENV_HOME_PATH=/path/to/aenv/data bash
sudo systemctl start aenv
```

要自定义服务器配置，请编辑 `config.toml` 并重启服务：

```bash
sudo vim /var/lib/aenv/config/config.toml  # Or <AENV_HOME_PATH>/config/config.toml if AENV_HOME_PATH is set.
sudo systemctl restart aenv
```

要更改端口，请编辑 `/etc/default/aenv` 中的 `API_ADDR` 并重启服务。

服务的持久状态默认由 `aenv` 用户拥有，位于 `/var/lib/aenv`。临时的命名空间和守护进程套接字状态位于 `/run/aenv`。

**选项 B — Docker**

```bash
curl -fsSL https://raw.githubusercontent.com/kvcache-ai/AgentENV/main/scripts/docker-setup.sh | sudo bash
docker pull ghcr.io/kvcache-ai/aenv-server:latest
docker run --rm -it \
  --name aenv-server \
  --device /dev/kvm --privileged -v /dev:/dev \
  -p 8000:8000 \
  ghcr.io/kvcache-ai/aenv-server:latest
```

要自定义服务器配置，请下载并编辑配置文件，然后将其挂载到容器使用的路径：

```bash
curl -fsSL https://raw.githubusercontent.com/kvcache-ai/AgentENV/main/config/default.toml -o config.toml
vim config.toml
docker run --rm -it \
  --name aenv-server \
  --device /dev/kvm --privileged -v /dev:/dev \
  -v "$PWD/config.toml:/workspace/config/default.toml:ro" \
  -p 8000:8000 \
  ghcr.io/kvcache-ai/aenv-server:latest
```

要更改端口，请添加 `-e API_ADDR` 和 `-p` 标志。

服务器默认可通过 `http://127.0.0.1:8000` 访问。验证其正在运行：

```bash
curl http://127.0.0.1:8000/health
```

### 2. 安装 aenv CLI

如果你使用了**选项 A**，请跳过此步骤——安装脚本已包含 `aenv`。

支持 x86_64 和 arm64 上的 Linux 与 macOS：

```bash
curl -fsSL https://raw.githubusercontent.com/kvcache-ai/AgentENV/main/scripts/install-cli.sh | bash
```

安装脚本会下载并验证包含 `aenv` 和 `aenv-buildctl` 的平台发布归档。设置 `INSTALL_DIR` 可进行用户本地安装。如需手动安装，请解压归档，并将 `aenv` 和 `aenv-buildctl` 保留在同一目录中。

### 3. 认证

服务器在首次正常启动时生成密钥。原生安装会复用托管的密钥；普通 Docker 容器将其保存在可写的容器层中：

```bash
# Native
sudo cat /var/lib/aenv/secrets/api-key
# Docker
docker exec aenv-server cat /workspace/env/secrets/api-key
```

```bash
aenv auth
# AENV server URL [http://localhost:8000]: http://127.0.0.1:8000
# API key: <paste the generated key>
```

### 4. 拉取 Template（模板）并运行 Sandbox（沙盒）

```bash
aenv pull ubuntu:22.04 --name ubuntu
aenv start ubuntu            # starts a sandbox and attaches an interactive shell
```

## 后续步骤

- [部署](../deployment/manual-compile.md) — 从源码构建、多节点选项
- [PVM 部署](../deployment/pvm.md) — 在标准 KVM 不可用时部署
- [核心概念](../concepts/overview.md) — Sandbox、Template 和 Snapshot 如何工作
- [E2B](../integration/e2b.md) — SDK 和 CLI 兼容性
- [API 参考](../api/index.md) — 完整的 HTTP API
