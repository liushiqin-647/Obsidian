# Docker Compose（多节点模拟）

使用 Docker Compose 在单台主机上运行完整的多节点栈。这会模拟一个类生产拓扑，包含 Gateway、Scheduler 和多个 AgentENV 后端节点。

关于不使用 Kubernetes 的真实多机部署，请参见 [静态多节点](./static-multi-node.md)。

## 前置条件

- Linux 内核 6.8+
- `/dev/kvm` 访问权限（传入运行时容器）
- 带 Docker Compose v2 的 Docker Engine（`docker compose version`）
- 用于验证命令的 `curl`

仓库中已检入的 Compose 配置使用标准 KVM。如果主机不支持它，请在调整运行时镜像和主机配置之前阅读 [PVM 部署](./pvm.md)。

## 克隆仓库

```bash
git clone https://github.com/kvcache-ai/AgentENV.git
cd AgentENV
```

## 启动集群

```bash
sudo bash scripts/docker-setup.sh
make deploy-up
```

`make deploy-up` 会在启动栈之前使用 Docker Compose 构建运行时、Gateway 和 Scheduler 镜像。Rust 和 Go 工具链安装在镜像构建阶段，因此主机上不需要它们。

运行时节点会以只读方式挂载主机的上游 DNS 配置，用于客户机网络（包括模板构建器）。`make deploy-*` 目标在可用时选择 `/run/systemd/resolve/resolv.conf`，否则选择 `/etc/resolv.conf`。用绝对路径覆盖 `HOST_RESOLV_CONF`，指向包含客户机可访问 DNS 服务器的解析器文件；客户机无法使用 `127.0.0.53` 等回环地址和 Docker 的 `127.0.0.11`。直接使用 `docker compose` 命令时默认使用 `/run/systemd/resolve/resolv.conf`；在没有该文件的主机上请显式设置 `HOST_RESOLV_CONF`。容器保留 Docker DNS 用于服务发现。

仅构建而不启动，请运行 `make deploy-build`。启动已构建的镜像，请运行 `make deploy-up-no-build`：

```bash
# Build only
make deploy-build

# Start previously built images
make deploy-up-no-build
```

首次启动时，运行时节点会在共享的 `agentenv-auth` 卷中原子地生成一个 API key 和一个沙盒访问令牌种子。Gateway 以只读方式挂载该卷并读取 API key；Sandbox 令牌由运行时节点验证。正常的 `make deploy-down` 调用会保留这两个值。

要启用基于主机的 Sandbox 数据平面 URL，请在启动栈时设置共享的 Sandbox 代理域名变量：

```bash
SANDBOX_PROXY_DOMAINS=sandbox.example.com \
make deploy-up
```

Compose 会将此值同时传递给 Gateway 路由允许列表和运行时节点的 Sandbox 响应元数据。该域名必须解析到 Gateway，通常通过 `*.sandbox.example.com` 的通配符 DNS 实现。

## 验证

```bash
# Health check via gateway
curl http://127.0.0.1:8000/health

# Authenticated cluster node snapshots via gateway
export AENV_API_KEY="$(docker compose -f deploy/docker-compose.yml exec -T agentenv-a \
  cat /workspace/env/secrets/api-key)"
curl -H "X-API-Key: ${AENV_API_KEY}" http://127.0.0.1:8000/nodes

# Health check from inside a backend container
docker compose -f deploy/docker-compose.yml exec -T agentenv-a \
  curl -fsS http://127.0.0.1:8000/health
```

## 管理命令

```bash
make deploy-ps      # Show container status
make deploy-logs    # Stream logs from all services
make deploy-down    # Tear down the cluster
```

使用 `docker compose down -v` 删除 Compose 卷也会删除两个 secret。下次启动会生成新值，因此现有客户端和 Sandbox 访问令牌会失效。

要使用已有的 API key 而非生成值，请在启动栈之前导出 `AENV_API_KEY`。Compose 会将其传递给 Gateway 和两个运行时节点：

```bash
export AENV_API_KEY="e2b_..."
make deploy-up
```

## 配置

Scheduler 和 Gateway 使用 `deploy/docker/config/default.json`。有关 JSON 字段、环境变量覆盖、HTTPS 设置和示例，请参见 [Gateway 与 Scheduler 配置](../configuration/control-plane-config.md)。

仅更改 JSON 后，请重启受影响的服务以重新加载：

```bash
docker compose -f deploy/docker-compose.yml restart scheduler gateway
```

更改 Compose 环境变量、端口或挂载后，请使用现有镜像应用新的容器配置：

```bash
make deploy-up-no-build
```
