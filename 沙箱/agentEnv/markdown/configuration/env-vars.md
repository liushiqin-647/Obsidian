# 环境变量

## 部署辅助变量

这些变量由仓库的 Docker Compose 和 Kubernetes 辅助脚本使用，然后传递给下文列出的服务器或 Gateway 专用变量。

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_PROXY_DOMAINS` | 空 | 用于基于主机的沙盒数据平面 URL 的逗号分隔 DNS 域名。在多节点部署中，此单一值会同时应用于 Gateway 路由和运行时 Sandbox 响应元数据。 |

## 服务器

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `AENV_API_KEY` | 生成于 `$AENV_HOME/secrets/api-key` | 可选的 API key 覆盖。运行时节点在创建托管 key 之前还会检查 `/run/secrets/api-key`。在多节点部署中使用一个共享值或 secret。 |
| `API_ADDR` | `0.0.0.0:8000` | API 服务器监听的地址和端口 |
| `AENV_CONFIG_PATH` | `config/default.toml` | TOML 配置文件路径 |
| `AENV_LOG_FORMAT` | `compact` | 服务器日志输出格式：`compact`、`pretty` 或 `json` |
| `AENV_LOG_SPAN_EVENTS` | `off` | 要发出的 tracing span 生命周期事件：`off`、`new`、`enter`、`exit`、`close`、`active` 或 `full` |
| `AENV_NODE_ID` | 由主机名派生 | 覆盖用于可观测性/管理快照的运行时节点标识符 |
| `AENV_CLUSTER_ID` | nil UUID | 覆盖用于 P2P 对等发现和 Scheduler 分组的集群 UUID |
| `AENV_SERVICE_INSTANCE_ID` | 随机 UUIDv7 | 覆盖包含在心跳中的每进程服务实例 UUID |
| `AENV_OBSERVABILITY_SCHEDULER_REPORT_ENABLED` | 来自配置 | 启用 Scheduler 心跳上报 |
| `AENV_OBSERVABILITY_SCHEDULER_ENDPOINT` | 未设置 | 覆盖 Scheduler 心跳上报端点 |
| `AENV_OBSERVABILITY_REPORT_INTERVAL_SECS` | `5` | 以秒为单位覆盖心跳上报间隔 |
| `AENV_CUSTOM_EXTENSION_URL` | 未设置 | 覆盖 `[custom_extension].url`，即自定义扩展服务的 HTTP 基础 URL |
| `AENV_SANDBOX_ACCESS_TOKEN_HASH_SEED` | 自动生成于 `$AENV_HOME/secrets` 下 | 可选的运行时覆盖，用于派生 Sandbox envd 和流量访问令牌的密钥。在集群部署中，需在每个运行时节点上配置相同的值。 |
| `AENV_SANDBOX_PROXY_DOMAINS` | 来自配置 | 逗号分隔的 DNS 域名，用于启用服务端基于主机的 Sandbox 代理 URL，如 `{port}-{sandboxID}.{domain}`，并填充 Sandbox 响应的 `domain` 字段。为空或未设置时保留 `[sandbox_proxy].domains`。 |
| `AENV_HOME_PATH` | `/var/lib/aenv` | 覆盖 AgentENV 派生本地状态、缓存、日志、生成的配置和下载依赖的基础目录。组件特定的路径设置仍可作为高级覆盖使用。 |
| `AENV_RUNTIME_PATH` | `/run/aenv` | 覆盖用于网络命名空间挂载点和默认 ublk 守护进程套接字的临时运行时目录。 |
| `AENV_DEPS_PATH` | `$AENV_HOME/deps` | 覆盖自动下载的运行时资产（Firecracker、内核、tools 驱动）的根目录。 |
| `AENV_VIRTUALIZATION_MODE` | `kvm` | 选择节点虚拟化模式。正常安装时保持未设置；仅在遵循 [PVM 部署](../deployment/pvm.md) 指南时设置为 `pvm`。 |
| `AENV_SNAPSHOT_LOCAL_CACHE_PATH` | `$AENV_HOME/snapshot-local-cache` | 覆盖 Snapshot 管理器的节点本地 artifact/缓存根目录 |
| `AENV_SNAPSHOT_STORE` | `$AENV_HOME/snapshot-store` | 覆盖 posix_fs Snapshot 仓库根目录 |
| `AENV_UBLK_DAEMON_BINARY_PATH` | `$AENV_HOME/ublk/uvm-ublk-daemon` | 覆盖 `uvm-ublk-daemon` 二进制路径 |
| `AENV_UBLK_DAEMON_METRICS_LISTEN_ADDR` | `0.0.0.0:9103` | 覆盖 ublk 守护进程 Prometheus 指标监听地址；空字符串会禁用它 |
| `AENV_FORCE_SYSCTL_TUNING` | 未设置 | 设置为 `1` 以在具有可写主机 sysctl 的特权容器中强制执行 sysctl 调优。在容器内通常会自动跳过。 |
| `AENV_FIRECRACKER_WORK_DIR` | `$AENV_HOME/firecracker-work` | 覆盖每 Sandbox Firecracker 工作目录的父目录。 |
| `AENV_FIRECRACKER_SERIAL_DIR` | `$AENV_HOME/logs/serial` | 覆盖持久化 Firecracker 串口输出的目录。文件按 `{serial_dir}/{sandbox_id}/` 分组存放。 |
| `AENV_PERSISTED_SANDBOX_STORE_PATH` | `$AENV_HOME/persisted-sandboxes` | 覆盖暂停 Sandbox 状态在服务器重启间持久化的目录。 |

## E2B SDK / CLI

这些变量配置 E2B SDK 和 CLI 指向 AgentENV 服务器。具体值取决于你的部署模式。

| 变量 | 说明 |
|----------|-------------|
| `E2B_API_URL` | AgentENV 服务器 API 基础 URL |
| `E2B_SANDBOX_URL` | Sandbox 代理 URL（用于 WebSocket 和进程交互） |
| `E2B_API_KEY` | 设置为部署的 `AENV_API_KEY` |

### 按部署模式取值

**手动编译（单节点）**：

```bash
export E2B_API_URL=http://127.0.0.1:8000
export E2B_SANDBOX_URL=${E2B_API_URL}
export E2B_API_KEY=${AENV_API_KEY}
```

**Docker Compose / Kubernetes（多节点）**：

```bash
export E2B_API_URL=http://127.0.0.1:8080
export E2B_SANDBOX_URL=${E2B_API_URL}
export E2B_API_KEY=${AENV_API_KEY}
```

> 在两种模式下，Sandbox 数据平面请求都可以使用路由头，并设置
> `E2B_SANDBOX_URL=${E2B_API_URL}`。显式的 `/proxy` 前缀
> （`${E2B_API_URL}/proxy`）出于向后兼容仍然被接受。

有关 key 的生成和存储，请参见 [API 认证](../concepts/authentication/api-authentication.md)。

## Gateway 与 Scheduler

这些变量适用于 Gateway 和 Scheduler 两个进程。

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `LOG_LEVEL` | `info` | 日志级别：`debug`、`info`、`warn` 或 `error` |
| `LOG_FORMAT` | `auto` | 日志输出格式：`auto`、`console` 或 `json` |

## Gateway

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `AENV_API_KEY` | 未设置 | 共享的单租户 API key。设置时 Gateway 使用环境变量值，否则读取 `/run/secrets/api-key`。 |
| `GATEWAY_HTTP_LISTEN_ADDR` | `:8080` | HTTP 监听地址 |
| `GATEWAY_METRICS_LISTEN_ADDR` | `:9102` | Prometheus 指标监听地址 |
| `GATEWAY_SCHEDULER_ADDR` | `127.0.0.1:9090` | 用于路由和节点查找的 Scheduler gRPC 地址 |
| `GATEWAY_QUERY_ONLY_SCHEDULER_ADDR` | 未设置 | 可选的辅助 Scheduler gRPC 地址，仅用于 Sandbox 数据平面 `LookupNode` 查询。设置后，创建和控制平面调用仍发往 `GATEWAY_SCHEDULER_ADDR`。 |
| `GATEWAY_REQUEST_TIMEOUT` | `30s` | 覆盖 Gateway 的 HTTP 请求超时时间（例如 `1m30s`） |
| `GATEWAY_SANDBOX_PROXY_DOMAINS` | 来自配置 | 逗号分隔的 DNS 域名，用于启用 Gateway 基于主机的 Sandbox 代理 URL，如 `{port}-{sandboxID}.{domain}`。为空或未设置时保留 `gateway.sandbox_proxy_domains`。 |
| `GATEWAY_DEBUG_MODE` | `false` | 启用 Gateway 调试模式 |

## Scheduler

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SCHEDULER_GRPC_LISTEN_ADDR` | `:9090` | gRPC 监听地址 |
| `SCHEDULER_METRICS_LISTEN_ADDR` | `:9101` | Prometheus 指标监听地址 |
| `SCHEDULER_STRATEGY` | `round_robin` | 新 Sandbox 的节点选择策略：`round_robin` 或 `random` |
| `SCHEDULER_REDIS_ADDR` | 未设置 | 用于持久化 Sandbox 到节点绑定的 Redis 地址（例如 `redis:6379`）。未设置 = 内存绑定，Scheduler 重启后丢失。 |
| `SCHEDULER_BINDING_TTL` | `30s` | 在没有确认心跳的情况下，Sandbox 到节点绑定的保留时长。接受 Go 时长字符串（例如 `1m`）。 |
| `SCHEDULER_ARTIFACT_STORE_CAPACITY` | `1000000` | Scheduler 内存索引中持有的 P2P artifact 条目最大数量 |
| `SCHEDULER_ARTIFACT_LOOKUP_NODE_LIMIT` | `0` | 每次 P2P artifact 查询检查的节点最大数量。`0` 表示不限制。 |
