# 安全沙盒身份认证

安全 Sandbox（secure sandbox）需要 `envdAccessToken` 才能执行 envd 控制操作，包括交互式连接、命令执行以及文件上传或下载。这不会保护 Sandbox 对外暴露的服务；应用入口按 [应用入口身份认证](./application-ingress.md)所述使用 `trafficAccessToken`。

## 启用安全模式

在 Sandbox 创建请求中设置 `secure: true`。例如，创建一个安全 Sandbox 并捕获响应中返回的令牌：

```bash
SANDBOX_RESPONSE=$(curl -sS -X POST http://127.0.0.1:8000/sandboxes \
  -H "X-API-Key: <api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "templateID": "my-template",
    "secure": true
  }')

SANDBOX_ID=$(printf '%s' "$SANDBOX_RESPONSE" | jq -r '.sandboxID')
ENVD_ACCESS_TOKEN=$(printf '%s' "$SANDBOX_RESPONSE" | jq -r '.envdAccessToken')
```

使用 envd 端口 `49983`，并在 `X-Access-Token` 中发送令牌：

```bash
curl http://127.0.0.1:8000/proxy/health \
  -H "x-agentenv-sandbox-id: $SANDBOX_ID" \
  -H "x-agentenv-target-port: 49983" \
  -H "X-Access-Token: $ENVD_ACCESS_TOKEN"
```

`aenv` CLI 始终请求安全 Sandbox 身份认证，并自动处理 envd 访问令牌：

```bash
aenv start <template-or-snapshot>
```

对于 `connect`、`exec`、`upload` 和 `download`，它会通过经过认证的 AgentENV API 获取令牌，并将 `X-Access-Token` 添加到后续的 envd 请求中。

## 安全模式生命周期

安全模式及其凭据在暂停、服务重启和恢复后仍然有效。分叉出的 Sandbox 会获得独立的凭据。
