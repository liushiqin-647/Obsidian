# Docker（单节点）

在 Docker 容器中运行单个 AgentENV 节点。这样可以避免在主机上安装 Rust 工具链，但仍然需要 `/dev/kvm`。

## 前置条件

- Linux 内核 6.8+
- 用于执行 Firecracker microVM 的 `/dev/kvm` 访问权限
- Docker

如果服务器不支持标准 KVM，请遵循 [PVM 部署](./pvm.md) 进行所需的主机设置并使用 PVM 镜像。

## 构建

**方式 A — 预构建镜像**

```bash
docker pull ghcr.io/kvcache-ai/aenv-server:latest

curl -fsSL https://raw.githubusercontent.com/kvcache-ai/AgentENV/main/scripts/docker-setup.sh | sudo bash
```

**方式 B — 从源码构建**

```bash
git clone https://github.com/kvcache-ai/AgentENV.git
cd AgentENV
sudo bash scripts/docker-setup.sh
docker build -f deploy/docker/Dockerfile.agentenv -t aenv:latest .
```

要为构建和运行时阶段都使用区域 apt 镜像，请传入包含 `debian`、`debian-security` 和 `ubuntu` 镜像路径的基础 URL：

```bash
docker build \
  --build-arg APT_MIRROR_BASE=https://mirrors.example.com \
  -f deploy/docker/Dockerfile.agentenv \
  -t aenv:latest .
```

## 运行

```bash
docker run --rm -it \
  --name aenv-server \
  --device /dev/kvm --privileged -v /dev:/dev \
  -p 8000:8000 \
  ghcr.io/kvcache-ai/aenv-server:latest   # or aenv:latest if built from source
```

`--privileged` 标志是 Firecracker 网络命名空间操作（veth 对、iptables）所必需的。服务器会在首次启动时自动下载运行时资产，就绪后可通过 `http://127.0.0.1:8000` 访问。

正常启动时，服务器会在容器内的 `/workspace/env/secrets/api-key` 生成 API key。容器运行时可通过以下方式读取：

```bash
docker exec aenv-server cat /workspace/env/secrets/api-key
```

删除容器也会删除此生成的 key。当 key 必须在容器替换之间保持稳定时，请提供显式的 `AENV_API_KEY` 或 `/run/secrets/api-key` 处的 secret。

## 验证

```bash
curl http://127.0.0.1:8000/health
```
