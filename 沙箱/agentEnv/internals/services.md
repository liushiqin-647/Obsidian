# 分布式控制平面

多节点控制平面位于 `services/`，作为一个单独的 Go 模块。它在多个 AgentENV 后端节点之间路由客户端流量。

## 组件

- **Gateway（网关）**（`services/gateway/`）：按 Sandbox（沙盒）ID 路由的 HTTP 反向代理
- **Scheduler（调度器）**（`services/scheduler/`）：gRPC 服务，用于节点选择、Sandbox 到节点绑定、已观测节点快照和 P2P peer 端点发现

## 构建与测试

前置条件：Go 1.21+

```bash
# From services/
make build      # builds both gateway and scheduler
make test       # tests both services
make tidy       # go mod tidy + formatting
make proto      # regenerate protobuf
```

## 本地运行

```bash
# Start scheduler (default: 127.0.0.1:9090)
make -C services run-scheduler

# Start gateway (use the same key on runtime nodes)
export AENV_API_KEY="e2b_$(openssl rand -hex 32)"
make -C services run-gateway
```

## 发现模式

Scheduler 支持两种节点发现模式：

- **static**（默认）：来自配置的显式节点列表
- **kubernetes**：watch headless Service 的 EndpointSlice，使用就绪的 Pod IP 作为后端

## 部署

### Docker Compose

```bash
# Run scripts/docker-setup.sh first for host prerequisites.
make deploy-up      # gateway + scheduler + 2 backend nodes
make deploy-ps      # status
make deploy-logs    # logs
make deploy-down    # teardown
```

### Kubernetes

```bash
make k8s-render     # render manifests
make k8s-apply      # apply to cluster
```

部署模型：
- `gateway`：Deployment + ClusterIP Service
- `scheduler`：单副本 Deployment + ClusterIP Service
- `agentenv-node`：特权 DaemonSet，具有 `/dev/kvm`、一种主机兼容的
 KVM/PVM 模式以及 hostPath
- `agentenv-nodes`：用于 Scheduler EndpointSlice 发现的 headless Service

## gRPC API

Proto 契约：`services/api/proto/scheduler.proto`

RPC：`Schedule`、`ListNodes`、`LookupNode`、`RecordAssignment`、`Heartbeat`、`ListObservedNodes`、`ListP2pPeers`、`GetNode`、`UnregisterNode`

Runtime 节点心跳可能包含一个不透明的 `P2pEndpoint`，其中含有后端名称和后端特定地址。Scheduler 将该端点与已观测节点记录一起存储，并通过 `ListP2pPeers(cluster_id, backend, exclude_node_id)` 返回就绪 peer。Scheduler 不查询 artifact 目录，也从不转发 artifact 数据。

完整配置细节（头部兼容性、超时、日志），请参见 [services README](https://github.com/kvcache-ai/AgentENV/blob/main/services/README.md)。
