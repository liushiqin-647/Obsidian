# 应用入口身份认证

应用入口（application ingress）是经 AgentENV 代理发送到沙盒内运行服务的流量。它独立于 API 和 envd 身份认证。

当 `allowPublicTraffic: true`（默认值）时，应用入口不需要 AgentENV 凭据。

要创建私有 Sandbox，请设置 `allowPublicTraffic: false`。创建响应会包含该 Sandbox 的 `trafficAccessToken`；请捕获它以供之后的代理请求使用：

```bash
SANDBOX_RESPONSE=$(curl -sS -X POST http://127.0.0.1:8000/sandboxes \
  -H "X-API-Key: <api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "templateID": "my-template",
    "network": {
      "allowPublicTraffic": false
    }
  }')

SANDBOX_ID=$(printf '%s' "$SANDBOX_RESPONSE" | jq -r '.sandboxID')
TRAFFIC_ACCESS_TOKEN=$(printf '%s' "$SANDBOX_RESPONSE" | jq -r '.trafficAccessToken')
```

在访问 Sandbox 中的应用程序时，请在 `e2b-traffic-access-token` 中发送该令牌：

```bash
curl http://127.0.0.1:8000/proxy/ \
  -H "x-agentenv-sandbox-id: $SANDBOX_ID" \
  -H "x-agentenv-target-port: 8080" \
  -H "e2b-traffic-access-token: $TRAFFIC_ACCESS_TOKEN"
```

每个 Sandbox 都有自己的令牌，分叉出的 Sandbox 会获得独立的凭据。AgentENV 会在将请求转发到 Sandbox 应用程序之前移除 traffic 令牌。完整的代理路由和访问控制行为请参阅 [Proxy](../sandboxes/proxy.md#access-control)。
