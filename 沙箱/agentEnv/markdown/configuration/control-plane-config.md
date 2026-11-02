# Gateway 与 Scheduler 配置

Gateway 和 Scheduler 通过各自的 `-config` 命令行参数加载 JSON 配置。以下多节点部署使用相同的配置格式：

| 部署方式 | 配置文件 |
|---|---|
| [Docker Compose](../deployment/docker-compose.md#configuration) | `deploy/docker/config/default.json`，由两个服务共享。 |
| [静态多节点](../deployment/static-multi-node.md#3-configure-static-discovery) | `/etc/agentenv/control-plane.json`，在本指南中由两个服务共享。 |
| [Kubernetes](../deployment/kubernetes.md#configuration) | `deploy/k8s/base/config/scheduler.json` 和 `deploy/k8s/base/config/gateway.json`，通过 ConfigMap 提供。 |

配置按以下顺序加载，后加载的值会覆盖先加载的值：

1. 内置默认值。
2. 由 `-config` 选定的 JSON 文件。
3. 服务进程中受支持的环境变量。

「默认值」列列出的是内置值，可能与部署示例中显式设置的值不同。配置在启动时读取；应用更改后，请按相关部署指南重启受影响的服务。运行时节点的 TOML 设置单独记录在 [运行时节点配置](./reference.md) 中。

## 配置字段

| 字段 | 默认值 | 含义 |
|---|---|---|
| `log_level` | `"info"` | Zap 日志级别；常用值为 `debug`、`info`、`warn` 和 `error`。会被 `LOG_LEVEL` 覆盖。 |
| `log_format` | `"auto"` | `auto`、`console` 或 `json`。自动模式在交互式终端下选择控制台输出，否则选择 JSON。会被 `LOG_FORMAT` 覆盖。 |
| `scheduler` | 见下文 | Scheduler 配置对象。 |
| `gateway` | 见下文 | Gateway 配置对象。 |
| `service` | 由二进制设置 | 解析器接受该字段，但会被 `scheduler` 或 `gateway` 覆盖；请省略它。 |

### Scheduler 字段

下表中的所有字段都位于 `scheduler` 内。

| 字段 | 默认值 | 含义 |
|---|---|---|
| `grpc_listen_addr` | `":9090"` | 用于调度、绑定查询、节点上报及相关 RPC 的 gRPC 监听地址。会被 `SCHEDULER_GRPC_LISTEN_ADDR` 覆盖。 |
| `metrics_listen_addr` | `":9101"` | 用于 Prometheus 指标的独立 HTTP 监听地址。会被 `SCHEDULER_METRICS_LISTEN_ADDR` 覆盖。 |
| `strategy` | `"round_robin"` | 节点选择策略：`round_robin` 或 `random`。会被 `SCHEDULER_STRATEGY` 覆盖。 |
| `report_ttl` | `"30s"` | 在已观测节点视图中，心跳上报超过此时长即视为过期。使用时长字符串，如 `"30s"`，而非数字秒数。 |
| `binding_ttl` | `"30s"` | 沙盒到节点的路由绑定的生存期；分配记录和心跳协调会刷新绑定。使用时长字符串，如 `"30s"`，而非数字秒数。零或负的时长会回退为 `"30s"`。会被 `SCHEDULER_BINDING_TTL` 覆盖。 |
| `redis_addr` | `""` | 为空时使用进程内绑定存储；否则使用 Redis，格式为 `host:port` 或 Redis URL。会被 `SCHEDULER_REDIS_ADDR` 覆盖。 |
| `artifact_store_capacity` | `1000000` | 内存中 artifact 位置索引的最大不同 `(cluster_id, backend, key)` 条目数，采用 LRU 淘汰。对于正常运行的 Scheduler 必须为正数。会被 `SCHEDULER_ARTIFACT_STORE_CAPACITY` 覆盖。 |
| `artifact_lookup_node_limit` | `0` | artifact 查询返回的最大节点数；零或负值表示不限制。会被 `SCHEDULER_ARTIFACT_LOOKUP_NODE_LIMIT` 覆盖。 |
| `nodes` | 一个本地节点 | 静态运行时节点列表；见下文。 |
| `discovery` | 静态发现 | 发现配置对象；见下文。 |
| `node_resource_limit` | 未设置 | 在节点选择之前应用的可选每节点资源阈值；见下文。 |

#### `scheduler.discovery`

`scheduler.discovery.mode` 选择 Scheduler 发现运行时节点的方式：

- `static`（默认）使用 `scheduler.nodes` 中的节点列表。

每个 `scheduler.nodes` 条目接受：

| 字段 | 含义 |
|---|---|
| `id` | 运行时节点 ID，与该运行时的 `AENV_NODE_ID` 匹配。 |
| `endpoint` | 可从 Gateway 访问的运行时 API URL，如 `http://agentenv-a:8000`。 |

静态模式要求节点列表非空，且 ID 和 endpoint 均非空。如果完全省略 `nodes`，内置列表包含位于 `http://127.0.0.1:8000` 的 `local-node`。

例如，将以下字段合并到现有的 `scheduler` 对象中，以使用 [Docker Compose 部署](../deployment/docker-compose.md) 中的两个运行时服务：

```json
{
  "nodes": [
    {
      "id": "node-a",
      "endpoint": "http://agentenv-a:8000"
    },
    {
      "id": "node-b",
      "endpoint": "http://agentenv-b:8000"
    }
  ],
  "discovery": {
    "mode": "static"
  }
}
```

- `kubernetes` 使用 `scheduler.discovery.kubernetes` 监视 Service 的 EndpointSlice。

以下字段属于 `scheduler.discovery.kubernetes`，仅在 Kubernetes 模式下使用：

| 字段 | 默认值 | 含义 |
|---|---|---|
| `namespace` | `""` | Service 命名空间；Kubernetes 模式下必填。 |
| `service_name` | `""` | 其 EndpointSlice 提供运行时节点的 Service；必填。 |
| `port` | `0` | 要匹配并用于运行时 URL 的 EndpointSlice 端口；必须为正数。 |
| `scheme` | `"http"` | 构造运行时 URL 所用的 scheme。 |
| `ignore_pod_selector` | `""` | 可选的 Kubernetes 标签选择器；匹配的 Pod 会被完全排除在发现之外。 |
| `no_schedule_pod_selector` | `""` | 可选标签选择器；匹配的 Pod 会作为滞留节点保留用于现有路由，但被排除在新调度之外。 |

非空的选择器必须是合法的 Kubernetes 标签选择器。

例如，将以下字段合并到现有的 `scheduler` 对象中，用于命名空间 `agentenv-system` 中的 Service `agentenv-nodes`：

```json
{
  "discovery": {
    "mode": "kubernetes",
    "kubernetes": {
      "namespace": "agentenv-system",
      "service_name": "agentenv-nodes",
      "port": 8000,
      "scheme": "http"
    }
  }
}
```

#### `scheduler.node_resource_limit`

对于新的调度请求，Scheduler 会取已发现且非滞留的节点，根据 `scheduler.node_resource_limit` 进行过滤，然后使用轮询或随机选择选出其中一个。

每个资源限制字段都是可选的。省略某个字段即不施加限制。

| `scheduler.node_resource_limit` 下的字段 | 用于与阈值比较的节点指标 |
|---|---|
| `max_sandbox_count` | 活跃 Sandbox 数量，不含已暂停的 Sandbox。 |
| `max_sandbox_starting_count` | 当前正在启动的 Sandbox 数量。 |
| `max_cpu_used_percent` | 心跳中上报的 CPU 使用百分比。 |
| `max_cpu_allocated_percent` | 已分配 CPU / 节点总 CPU × 100。 |
| `max_memory_used_percent` | 已用内存 / 节点总内存 × 100。 |
| `max_memory_allocated_percent` | 已分配内存 / 节点总内存 × 100。 |
| `max_sandbox_count_including_paused` | 活跃加暂停的 Sandbox 数量。 |
| `max_allocated_cpu_including_paused` | 活跃加暂停的 CPU 分配，以 CPU 单位计，而非百分比。 |
| `max_allocated_memory_bytes_including_paused` | 活跃加暂停的内存分配，以字节计。 |

### Gateway 字段

下表中的所有字段都位于 `gateway` 内。

| 字段 | 默认值 | 含义 |
|---|---|---|
| `http_listen_addr` | `":8080"` | TLS 关闭时的主监听地址。会被 `GATEWAY_HTTP_LISTEN_ADDR` 覆盖。 |
| `https_listen_addr` | `":8080"` | TLS 启用时的主监听地址。会被 `GATEWAY_HTTPS_LISTEN_ADDR` 覆盖。 |
| `metrics_listen_addr` | `":9102"` | 用于 Prometheus 指标的独立 HTTP 监听地址。会被 `GATEWAY_METRICS_LISTEN_ADDR` 覆盖。 |
| `scheduler_addr` | `"127.0.0.1:9090"` | 主 Scheduler 的 gRPC 目标；通常为 `host:port`，不带 HTTP URL 前缀。会被 `GATEWAY_SCHEDULER_ADDR` 覆盖。 |
| `query_only_scheduler_addr` | `""` | 用于 `LookupNode` 的可选独立 Scheduler；为空时复用主 Scheduler。会被 `GATEWAY_QUERY_ONLY_SCHEDULER_ADDR` 覆盖。独立的仅查询 Scheduler 通过命令行标志 `--query-only` 启动，而非 JSON 字段。它要求 `scheduler.redis_addr` 指向共享的绑定存储，并仅支持绑定查询 RPC。 |
| `request_timeout` | `"30s"` | 路由以及有界控制平面/调度请求的超时时间。使用时长字符串，如 `"30s"`，而非数字秒数。会被 `GATEWAY_REQUEST_TIMEOUT` 覆盖。 |
| `forward_response_size` | `4194304` | 缓冲上游响应体以提取 ID 并记录绑定时的字节上限。超出时返回 502；非正值会禁用该限制。它不是对所有代理响应的通用限制。 |
| `sandbox_proxy_domains` | `[]` | 基于主机的 Sandbox 路由所允许的域名后缀，如 `["sandbox.example.com"]`。会被 `GATEWAY_SANDBOX_PROXY_DOMAINS` 覆盖。环境变量覆盖使用逗号分隔的列表。 |
| `debug_mode` | `false` | 启用调试行为，例如在代理响应中暴露后端节点 ID。会被 `GATEWAY_DEBUG_MODE` 覆盖。 |
| `tls` | TLS 关闭 | HTTPS 配置对象；见下文。 |

#### `gateway.tls`

| 字段 | 默认值 | 含义 |
|---|---|---|
| `mode` | `"off"` | `off`、`manual`（使用已有证书）或 `acme`（自动证书管理）。 |
| `cert_file` | `""` | 证书文件路径；manual 模式下必填。 |
| `key_file` | `""` | 私钥文件路径；manual 模式下必填。 |
| `domains` | 未设置 | 非空的证书 DNS 名称或符合条件的公网 IP 列表；ACME 模式下必填。 |
| `storage_dir` | `""` | 持久化的证书和 ACME 状态目录；ACME 模式下必填。 |
| `email` | `""` | ACME 账户联系邮箱。 |
| `ca` | `""` | 可选的 HTTPS ACME 目录 URL；为空时使用 CertMagic 默认 CA。 |
| `challenge` | `"tls-alpn-01"` | ACME 挑战方式：`tls-alpn-01` 或 `dns-01`。 |
| `dns_provider` | `""` | DNS-01 提供商：`cloudflare`、`alidns`、`tencentcloud` 或 `route53`。 |
| `dns_config` | 未设置 | 提供商特定的 JSON 对象；DNS-01 下必填。 |

启用 TLS 会将主监听器切换为 HTTPS。它不会同时启动 HTTP 监听器，也不会执行 HTTP 到 HTTPS 的重定向。

对于已有证书，在 `gateway` 内添加此对象：

```json
"tls": {
  "mode": "manual",
  "cert_file": "/etc/agentenv/tls/fullchain.pem",
  "key_file": "/etc/agentenv/tls/privkey.pem"
}
```

对于带通配符 Sandbox 域名的 ACME，Cloudflare 示例如下：

```json
"tls": {
  "mode": "acme",
  "domains": ["api.example.com", "*.sandbox.example.com"],
  "storage_dir": "/var/lib/agentenv/tls",
  "email": "operator@example.com",
  "challenge": "dns-01",
  "dns_provider": "cloudflare",
  "dns_config": {
    "api_token": "<cloudflare-api-token>"
  }
}
```

IP 证书要求 TLS-ALPN-01，每个监听器最多允许一个 IP，并拒绝私有或特殊用途的 IP。TLS-ALPN-01 不得包含 `dns_provider` 或 `dns_config`，且公网 TCP 端口 443 必须能到达 Gateway TLS 监听器以完成验证。

通配符证书要求 DNS-01。当 `challenge` 设置为 `dns-01` 时，请配置 `dns_provider` 及其对应的 `dns_config` 对象。支持的提供商及其配置字段如下：

| 提供商 | 字段 |
|---|---|
| `cloudflare` | 必填 `api_token`；可选 `zone_token`，用于单独的 Zone 读取访问。 |
| `alidns` | 必填 `access_key_id`、`access_key_secret`；可选 `region_id`、`security_token`。 |
| `tencentcloud` | 必填 `SecretId`、`SecretKey`；可选 `SessionToken`、`Region`。这些是当前依赖暴露的字段名。 |
| `route53` | 可选 `access_key_id` 和 `secret_access_key`，需同时提供或同时省略以使用 AWS 凭证链；还有 `session_token`、`region`、`profile` 和 `hosted_zone_id`。其他选项：`max_retries`（重试次数）、`route53_max_wait`（内部 AWS 同步的最大等待时间）、`wait_for_route53_sync`（是否等待该同步）、`skip_route53_sync_on_delete`（删除时跳过该同步）。`route53_max_wait` 使用整数**纳秒**数，而非 `"30s"` 这类时长字符串。 |

未知的提供商字段会被拒绝。

## 示例配置

此示例使用 [Docker Compose 部署](../deployment/docker-compose.md) 中的两个运行时节点、静态发现、轮询调度和 HTTP Gateway。

```json
{
  "log_level": "info",
  "log_format": "json",
  "scheduler": {
    "grpc_listen_addr": ":9090",
    "strategy": "round_robin",
    "discovery": {
      "mode": "static"
    },
    "nodes": [
      {
        "id": "node-a",
        "endpoint": "http://agentenv-a:8000"
      },
      {
        "id": "node-b",
        "endpoint": "http://agentenv-b:8000"
      }
    ],
    "node_resource_limit": {
      "max_cpu_used_percent": 85,
      "max_memory_used_percent": 90
    }
  },
  "gateway": {
    "http_listen_addr": ":8080",
    "scheduler_addr": "scheduler:9090",
    "request_timeout": "90s"
  }
}
```
