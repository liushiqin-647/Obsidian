# 身份认证

AgentENV 使用三种凭据分别对应三类不同的访问：

| 凭据 | 保护对象 | 何时需要 | 请求头 |
| --- | --- | --- | --- |
| API key | AgentENV 生命周期和管理 API | 所有经过认证的控制平面请求 | `X-API-Key` |
| `trafficAccessToken` | 沙盒对外暴露的服务 | `allowPublicTraffic: false` | `e2b-traffic-access-token` |
| `envdAccessToken` | envd 操作，如命令执行和文件访问 | `secure: true` | `X-Access-Token` |

这些凭据不可互换。API key 属于 AgentENV 部署；另外两个令牌属于单个 Sandbox。

> **注意：**身份认证验证凭据，但不加密流量。请使用 HTTPS 终端、VPN、环回接口或可信私有网络来保护传输中的 API key 和 Sandbox 令牌。

---

接下来：

- [API 身份认证](./api-authentication.md) — 对生命周期和管理 API 请求进行身份认证。
- [应用入口身份认证](./application-ingress.md) — 保护 Sandbox 对外暴露的服务。
- [安全 Sandbox 身份认证](./secure-sandbox.md) — 对命令执行、文件传输和其他 envd 操作进行身份认证。
- [Sandbox 访问令牌种子](./access-token-seed.md) — 配置用于派生 Sandbox 访问令牌的种子。
