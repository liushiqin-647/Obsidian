# 代理（Proxy）

反向代理让你可以从外部访问 Sandbox 内部运行的服务。它支持 HTTP 请求、SSE 流和 WebSocket 连接。

## 端点

- `ANY /proxy` 转发到 Sandbox 内的 `/`
- `ANY /proxy/{path}` 转发到 Sandbox 内的 `/{path}`
- 对于其他未匹配路径上的基于头部的路由请求，会原样转发原始路径。这使客户端在发送路由头时可以对 API 和 Sandbox 数据流量使用相同的基础 URL。
- 配置了 Sandbox 代理域名后，形如 `{port}-{sandboxID}.{domain}` 的基于主机的 URL 无需路由头即可转发原始路径。

查询字符串会被原样转发。

例如，在运行中的 Sandbox 内启动一个监听 `8080` 端口的 HTTP 服务器：

```bash
aenv exec <sandbox-id> sh -c 'echo "Hello from AgentENV" > /tmp/index.html'
aenv exec <sandbox-id> python3 -m http.server 8080 --directory /tmp
```

通过 AgentENV 代理访问该服务：

```bash
curl http://127.0.0.1:8000/proxy/index.html \
  -H "x-agentenv-sandbox-id: <sandbox-id>" \
  -H "x-agentenv-target-port: 8080"
```

## 基于头部的路由

使用 `/proxy` 或通过头部路由其他未匹配路径时，用以下头部标识目标 Sandbox 和端口：

| 头部 | 描述 |
|--------|-------------|
| `x-agentenv-sandbox-id` | 要路由到的 Sandbox UUID |
| `x-agentenv-target-port` | Sandbox 内服务的端口 |

也接受 E2B 兼容别名：

| 头部 | 别名对应 |
|--------|-----------|
| `e2b-sandbox-id` | `x-agentenv-sandbox-id` |
| `e2b-sandbox-port` | `x-agentenv-target-port` |

这些路由头部会在请求被转发到 Sandbox 之前被剥离。

## 访问控制

代理认证与 AgentENV API 认证相互独立：

| 流量 | 适用时机 | 所需凭据 |
| --- | --- | --- |
| 公开应用入站 | Sandbox 使用 `allowPublicTraffic: true`（默认值）。 | 无 |
| 私有应用入站 | Sandbox 使用 `allowPublicTraffic: false`。 | `e2b-traffic-access-token: <trafficAccessToken>` |
| 安全 envd 流量 | Sandbox 创建时启用了安全通信。 | `X-Access-Token: <envdAccessToken>` |

`X-API-Key` 仅用于认证 AgentENV 控制平面 API。它不会授予对私有应用入站或安全 envd 的访问权限。匹配的平台密钥会在代理请求中被剥离；其他 `X-API-Key` 值仍可供 Sandbox 应用使用。AgentENV 也会剥离流量令牌，并且仅将 `X-Access-Token` 转发到匹配的安全 envd 端口。

基于主机的代理请求从 `Host` 推导这两个值，例如 `http://8080-<sandbox-uuid>.sandbox.example.com/health` 会指向 `8080` 端口。在单节点模式下，所配置的域名必须路由到 AgentENV 服务器；在多节点模式下必须路由到网关（Gateway）。基于主机的代理流量始终被视为数据平面流量；生命周期和其他控制平面 API 应使用基础 API 主机。
