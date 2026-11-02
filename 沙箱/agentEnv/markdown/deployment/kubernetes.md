# Kubernetes（多节点）

在 Kubernetes 集群中部署 AgentENV，包含 Gateway、Scheduler 以及每个 worker 上的运行时节点。

## 架构

| 工作负载 | 类型 | 说明 |
|----------|------|-------------|
| `agentenv-gateway` | Deployment + ClusterIP Service | 面向客户端流量的 HTTP 反向代理 |
| `agentenv-scheduler` | Deployment（单副本）+ ClusterIP Service | gRPC 节点选择和沙盒绑定 |
| `agentenv-node` | DaemonSet（特权） | 每个 Kubernetes 节点一个运行时 Pod |
| `agentenv-nodes` | Headless Service | 供 Scheduler 进行 EndpointSlice 发现 |

### 为什么运行时节点使用 DaemonSet

- 每个 Pod 需要宿主机本地访问 `/dev/kvm`
- Sandbox 网络使用主机 iptables 和网络命名空间
- 运行时资产和已提交的快照状态按主机缓存在 `/var/lib/aenv`

## 前置条件

- 具有 **Linux 内核 6.8+** 的 Kubernetes worker 节点
- 每个运行时 worker 上的 `/dev/kvm` 访问权限
- 运行时 Pod 以特权模式运行
- Docker
- `build-essential`（`sudo apt install -y build-essential`）
- 支持 Kustomize 的 `kubectl`
- 所有运行时节点间共享存储，使用 POSIXFS 或 OSS

提供的 manifest 使用标准 KVM。当标准 KVM 不可用时，要准备单独的 PVM 节点池，请参见 [PVM 部署](./pvm.md)。

## 克隆仓库
```bash
git clone https://github.com/kvcache-ai/AgentENV.git
cd AgentENV
```

## 构建容器镜像

```bash
make k8s-build
```

这会构建三个镜像：`agentenv-runtime:latest`、`agentenv-gateway:latest` 和 `agentenv-scheduler:latest`。

## 配置访问令牌种子（可选）

共享 Sandbox 访问令牌种子的配置请参见 [Sandbox 访问令牌种子](../concepts/authentication/access-token-seed.md)。

## 部署

```bash
# Run on each worker node before deploying
sudo bash scripts/docker-setup.sh

# Render manifests (preview)
make k8s-render

# Apply to cluster
make k8s-apply
```

`make k8s-apply` 会在首次部署时生成一个 256 位 API key 并存储在 `Secret/agentenv-auth` 中。后续 apply 会复用它。配置客户端时可在本地读取该 key：

```bash
kubectl -n agentenv-system get secret agentenv-auth \
  -o go-template='{{index .data "AENV_API_KEY" | base64decode}}{{"\n"}}'
```

在 apply 时设置 `AENV_API_KEY` 以提供你自己的值。单独运行 `make k8s-render` 会使用无效的 `REDACTED` 占位符，因此预览输出永远不会包含可部署的 API key。可选的运行时种子保留其现有的 `agentenv-runtime-secrets` 契约，详见 [Sandbox 访问令牌种子](../concepts/authentication/access-token-seed.md)。

要启用基于主机的 Sandbox 数据平面 URL，请在渲染或应用 manifest 时设置共享的 Sandbox 代理域名变量：

```bash
SANDBOX_PROXY_DOMAINS=sandbox.example.com make k8s-apply
```

辅助脚本会将此值写入生成的 ConfigMap，并同时应用于 Gateway 路由允许列表和运行时节点的 Sandbox 响应元数据。该域名必须解析到 Gateway 的 Ingress 或 LoadBalancer，通常通过 `*.sandbox.example.com` 的通配符 DNS 实现。

默认 overlay 为 `deploy/k8s/overlays/default`，目标命名空间为 `agentenv-system`。Gateway 默认以 ClusterIP 暴露。请添加你自己的 Ingress 或 LoadBalancer 以实现外部访问。

make 目标会构建一个临时 Kustomize 上下文，使运行时 Pod 挂载仓库的 `config/default.toml`，而非单独的已检入副本。

运行时 DaemonSet 会为每个节点 Pod 注入 Scheduler 上报接线：

- `AENV_UBLK_DAEMON_BINARY_PATH=/usr/local/bin/uvm-ublk-daemon`，使 Pod 使用运行时镜像中包含的 `uvm-ublk-daemon` 二进制
- `AENV_NODE_ID` 来自 Pod 元数据名称（`metadata.name`）
- `AENV_OBSERVABILITY_SCHEDULER_REPORT_ENABLED=true`
- `AENV_OBSERVABILITY_SCHEDULER_ENDPOINT=http://agentenv-scheduler:9090`
- `AENV_SANDBOX_PROXY_DOMAINS` 来自共享 Sandbox 代理 ConfigMap

P2P 监听地址必须 Pod 到 Pod 可达；如果你的集群策略不允许拨号临时端口，请使用具体的容器端口或 Pod 可达地址。

## 运维

```bash
# Rollout restart all workloads
make k8s-redeploy

# Delete all resources
make k8s-delete
```

## 本地开发（k3s）

专用的 `local-dev` overlay 会将仓库的 `env/` 目录直接挂载到 DaemonSet 的 `/workspace/env`，避免运行时资产复制：

apply 辅助脚本会提供与默认 overlay 相同的生成 API key。本地开发 overlay 保留其固定的仅测试运行时种子；请勿在本地开发之外复用该种子。

```bash
make k8s-build
make k8s-load-dev       # Import images into k3s/containerd
make k8s-render-dev     # Preview manifests
make k8s-apply-dev      # Apply to cluster
make k8s-refresh-dev    # Build + load + rollout restart (all-in-one)
```

## 配置

Scheduler 和 Gateway 分别使用 `deploy/k8s/base/config/scheduler.json` 和 `deploy/k8s/base/config/gateway.json`，通过 ConfigMap 挂载。有关 JSON 字段、发现设置、环境变量覆盖和 HTTPS 选项，请参见 [Gateway 与 Scheduler 配置](../configuration/control-plane-config.md)。

编辑这些文件后，使用 `make k8s-apply`（或对本地开发 overlay 使用 `make k8s-apply-dev`）应用 manifest，然后重启受影响的 Deployment，使进程加载更新的配置。对于默认命名空间：

```bash
kubectl -n agentenv-system rollout restart deployment/agentenv-scheduler deployment/agentenv-gateway
```

## 服务发现

Scheduler 会监视 headless `agentenv-nodes` Service 的 EndpointSlice，并监视 Pod 以执行可选的基于标签的发现策略。它只调度正在服务、未终止的 DaemonSet Pod。匹配 `scheduler.discovery.kubernetes.no_schedule_pod_selector` 的 Pod 仍可作为滞留/不调度节点被发现，而匹配 `scheduler.discovery.kubernetes.ignore_pod_selector` 的 Pod 会被排除。同时支持 IPv4 和 IPv6 端点地址。

> Sandbox 绑定保留在内存中，因此 Scheduler 应以单副本运行。绑定在重启后丢失。
