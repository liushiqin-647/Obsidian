# 自定义扩展

自定义扩展（custom extension）是一个可选的外部 HTTP 服务，AgentENV 会在沙盒生命周期中调用它。它让你能够实现部署特有的行为——例如将 Sandbox 连接到 VPN、应用自定义防火墙规则或添加挂载——而无需修改 AgentENV 本身。

扩展实现一小组称为 hook 的 HTTP 端点，AgentENV 充当客户端。该接口定义在 [`src/custom_extension_api/openapi.yml`](https://github.com/kvcache-ai/AgentENV/blob/main/src/custom_extension_api/openapi.yml)。

---

接下来：

- [生命周期 Hook](./lifecycle-hooks.md) — 实现四个生命周期端点。
- [使用扩展](./connect-and-use.md) — 配置 AgentENV 并管理扩展特有的 Sandbox 参数。
