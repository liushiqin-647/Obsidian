# AgentENV 代理设计与用法

本文档描述将请求转发到各个 Sandbox（沙盒）的每节点反向代理。分布式路由层（Gateway（网关）到 Scheduler（调度器）到节点）请参见 [系统架构](./architecture.md)。

## 范围

每个 AgentENV 节点运行一个反向代理，在其 API 服务器上接受请求，
并通过 Sandbox 交互网络将它们转发到 Sandbox 服务。在多节点部署中，
Gateway 通过 Scheduler 解析 Sandbox 所有权，并将数据平面请求转发到
拥有该 Sandbox 的节点的代理表面。

当前入口点：

- `ANY /proxy`
- `ANY /proxy/{*proxy_path}`
- 任何携带 Sandbox 路由头的、其他未匹配路径
- 当配置了 `[sandbox_proxy].domains` 时，基于主机的 Sandbox 代理请求：
 `{port}-{sandboxID}.{domain}`

实现参考：

- `src/api/proxy.rs`
- `src/orchestrator/service.rs`
- `src/orchestrator/proxy.rs`

## 路由契约

每个被代理的请求必须标识：

- Sandbox ID
- Sandbox 服务平面内的目标端口

接受的头部：

- `x-agentenv-sandbox-id`
- `x-agentenv-target-port`
- E2B 兼容别名：
  - `e2b-sandbox-id`
  - `e2b-sandbox-port`

验证：

- Sandbox ID 必须是有效的 UUID 格式。
- 目标端口必须解析为 `u16` 且大于 `0`。

授权在路由解析之后由拥有该 Sandbox 的 runtime 评估：

- 控制平面路由需要部署的 `X-API-Key`，不接受
 Sandbox 凭据。
- 节点 Prometheus `/metrics` 和健康检查 `/health` 对应用层
 认证是公开的，应在部署边界单独保护。
- 非 envd 应用路由仅在 Sandbox 具有私有 ingress
 （`allowPublicTraffic: false`）时才需要 `e2b-traffic-access-token`。
- envd 端口仅对 secure Sandbox 需要 `X-Access-Token`。
- `X-API-Key` 永远不是数据平面凭据。

分布式 Gateway 有意不做出 Sandbox 授权
决策。它将数据平面请求，包括无凭据的公共 ingress 和不安全
envd 请求，路由到拥有该 Sandbox 的 runtime。runtime 拥有
应用策略所需的 Sandbox 元数据，并执行权威的
token 验证。

基于主机的路由从 `Host` 派生这两个字段。配置的域在转为小写并
移除可选尾点之后必须完全匹配。
主机路由中的 Sandbox ID 必须是有效 UUID，目标端口必须适合
`u16`。

## Runtime 路由模型

AgentENV 在 Orchestrator 中维护一个内存 runtime 路由表。

- 路由键：`SandboxId`
- 路由值：`ProxyTarget`（当前为 host interaction IP）加上路由元数据（`version`、`updated_at`）

设计规则：

- 只有 `Running` Sandbox 发布 runtime 路由。
- 非运行状态依赖元数据回退（而非路由表状态）。

查找行为（`proxy_lookup_for`）：

1. 如果存在 runtime 路由：`Ready(target)`
2. 否则读取元数据：
   - 无元数据：`NotFound`
   - 元数据状态为 `Running`：`RouteMissing`
   - 元数据状态为 `Paused`：`Paused { auto_resume }`
   - 其他状态：`Unavailable(state)`

这使热路径读取保持轻锁，并避免在 API 请求路径中读取 Sandbox 实例内部。

## 暂停 Sandbox 的自动恢复

当生命周期策略启用时，`/proxy` 可以自动恢复暂停的 Sandbox。

- 代理路由解析不读取 Sandbox 实例内部。
- Orchestrator 查找返回 `Paused { auto_resume }`，代理根据该信号决定行为。
- 自动恢复每个请求尝试一次。
- 恢复超时更新使用 `EnsureMinimum(5 minutes)`：
  - 有效 Sandbox 超时为 `max(existing_timeout, 5 minutes)`
- 代理最多等待：
  - 测试构建：短的单元测试超时
  - 非测试 runtime：`60s`

暂停 Sandbox 的请求结果：

- `auto_resume = false`：`410 Gone`
- `auto_resume = true` 且恢复成功，随后路由变为就绪：请求正常转发
- `auto_resume = true` 但恢复/查找失败：`502 Bad Gateway`
- `auto_resume = true` 但恢复等待超时：`504 Gateway Timeout`

## 生命周期钩子与竞态加固

路由发布/移除与生命周期状态转换绑定。

- 创建/恢复成功路径：
  - 将元数据持久化为 `Running`
  - 仅当启动的 Sandbox handle 仍是当前 handle 时才发布 runtime 路由
- 暂停/删除/回滚路径：
  - 在停止/最终化之前原子地分离 Sandbox handle 和 runtime 路由

竞态保护：

- 通过指针身份检查阻止来自陈旧 handle 的延迟路由发布。
- Handle 分离和路由移除在同一临界区中发生，以减少陈旧路由可见窗口。
- 启动回滚支持过渡状态回滚和运行状态回滚路径。

## HTTP 转发语义

### 路径与查询

- `/proxy` 转发到上游 `/`
- `/proxy/{*proxy_path}` 转发 `/proxy` 之后的原始 URI 路径后缀
- 头部路由的回退请求转发原始请求路径
- 基于主机的请求转发原始请求路径
- 查询字符串原样转发

重要细节：

- 百分号编码的路径段会被保留。
- 重复的前导斜杠会被保留。
  - 示例：`/proxy//api` 转发为 `//api`
- 基于主机的代理路由在 Axum 路由匹配之前运行。当使用已配置的
 Sandbox 代理主机时，该请求是数据平面流量，即使其路径
 类似于控制平面 API。

### 头部处理

控制平面路由头在转发到上游之前被剥离：

- `x-agentenv-sandbox-id`
- `x-agentenv-target-port`
- `e2b-sandbox-id`
- `e2b-sandbox-port`

Sandbox 凭据处理：

- `e2b-traffic-access-token` 在转发前被剥离。
- 成功验证的 secure-envd `X-Access-Token` 会转发给 envd；
 否则该头部被剥离。
- 与平台 `X-API-Key` 匹配的值会被剥离；其他值会
 作为应用头转发。

逐跳头部在请求和响应路径上都会被剥离，包括：

- 标准逐跳头部（`Connection`、`Upgrade`、`TE`、`Trailer`、`Transfer-Encoding`、`Proxy-Authenticate`、`Proxy-Authorization`、`Keep-Alive`）
- 由 `Connection` 指定的任何额外头部

注入的转发头部：

- `x-forwarded-host`
- `x-forwarded-proto`
- `x-forwarded-method`
- `x-forwarded-uri`

### 流式传输

HTTP body 作为流代理（请求和响应），包括 SSE 和大规模上传/下载。

## WebSocket 语义

通过相同的 `/proxy` 端点支持 WebSocket upgrade。

- 客户端 upgrade 请求被验证并转发到上游。
- 在上游握手成功后建立双向帧桥接。
- 从上游选择的子协议会传播给客户端。

握手失败行为：

- 如果上游以 HTTP 响应拒绝（例如 `401`、`403`、`404`），状态/body 原样转发。
- 传输或连接失败返回 `502 Bad Gateway`。
- 握手超时返回 `504 Gateway Timeout`。

## 错误映射

- `400 Bad Request`
  - 缺失/无效的 Sandbox 路由头
  - 缺失/无效的目标端口头
  - 无效的上游 URI 构造
- `404 Not Found`
  - 未找到 Sandbox
- `410 Gone`
  - Sandbox 已暂停且自动恢复被禁用
  - Sandbox 存在但在当前状态下不可代理
- `502 Bad Gateway`
  - 上游传输/连接失败
  - Sandbox 为 `Running` 但 runtime 路由缺失（`RouteMissing`）
  - 暂停 Sandbox 自动恢复失败
- `504 Gateway Timeout`
  - 暂停 Sandbox 自动恢复超时
  - 上游响应头超时
  - 上游 websocket 握手超时

## 用法示例

### HTTP

```bash
curl -i \
  -H 'e2b-traffic-access-token: <trafficAccessToken>' \
  -H 'x-agentenv-sandbox-id: <sandbox-uuid>' \
  -H 'x-agentenv-target-port: 8080' \
  'http://127.0.0.1:8000/proxy/health?full=true'
```

### E2B 兼容头部

```bash
curl -i \
  -H 'X-API-Key: test-key' \
  -H 'e2b-sandbox-id: <sandbox-uuid>' \
  -H 'e2b-sandbox-port: 8080' \
  'http://127.0.0.1:8000/proxy/status'
```

### WebSocket

示例（通用）：

- URL：`ws://127.0.0.1:8000/proxy/ws/echo`
- 头部：
  - `x-agentenv-sandbox-id: <sandbox-uuid>`
  - `x-agentenv-target-port: <port>`
  - 部署所需的 API 认证头

### 基于主机

```bash
curl -i \
  -H 'X-API-Key: test-key' \
  'http://8080-<sandbox-uuid>.sandbox.example.com/status'
```

## 测试说明

相关测试覆盖存在于：

- `src/api/proxy.rs` 单元测试（HTTP、SSE、大 body、websocket、头部、路径保留、错误映射）
- `src/orchestrator/service.rs` 单元测试（路由发布/移除行为和陈旧 handle 防护）
- `tests/integration/orchestrator.rs` 中的集成生命周期测试
- `scripts/tests/e2e/suites/06_proxy.sh` 中的 E2E 代理套件（头部兼容性和暂停自动恢复行为）

对于环境支撑的集成验证，请使用 repository 规定的集成目标。
