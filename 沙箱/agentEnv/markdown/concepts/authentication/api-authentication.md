# API 身份认证

API key 用于对创建和管理 AgentENV 资源（包括模板、沙盒和快照）的请求进行身份认证：

```bash
curl http://127.0.0.1:8000/sandboxes \
  -H "X-API-Key: <api-key>"
```

## 获取 API Key

runtime 节点会按顺序检查以下 API key 来源：

1. `AENV_API_KEY`
2. `/run/secrets/api-key`
3. `$AENV_HOME/secrets/api-key`

Gateway 仅检查 `AENV_API_KEY` 和 `/run/secrets/api-key`。同一个部署中的 Gateway 和每个 runtime 节点必须使用相同的 key。

### 提供 API Key

要提供你自己的 key，请生成一个包含 32 到 256 个 URL 安全字符的值，并通过上述位置之一使其可用。例如，生成一个 E2B 兼容的 key 并通过环境变量设置它：

```bash
export AENV_API_KEY="e2b_$(openssl rand -hex 32)"
```

在多节点部署中，请为 Gateway 和每个 runtime 节点使用同一个显式生成的 key。Docker Compose 可以通过其共享的 managed-secret 卷提供该 key，而 Kubernetes 会将其存储在 `Secret/agentenv-auth` 中。设置说明请参阅相应的部署指南。

### 使用自动生成的 API Key

如果 runtime 节点在这三个位置中都找不到 key，正常的服务启动会生成一个 E2B 兼容的 key，并以原子方式将其存储到 `$AENV_HOME/secrets/api-key`。请在服务启动后读取该文件，并在配置 CLI 或其他 API 客户端时使用其值。

自动生成对普通的单节点部署很方便。Gateway 从不生成 key，因此多节点部署必须让 Gateway 和所有 runtime 节点都能获得同一个共享 key。

## 配置 CLI

运行 `aenv auth`，输入 AgentENV 服务器 URL，并粘贴上面获取的 API key。按 Enter 接受默认的本地 URL。API key 输入内容会被隐藏：

```text
$ aenv auth
AENV server URL [http://localhost:8000]: http://localhost:8000
API key: <paste-api-key-here>
Credentials saved.
```

E2B 兼容的 SDK 会从 `E2B_API_KEY` 读取相同的 AgentENV API key：

```bash
export E2B_API_KEY="<paste-api-key-here>"
```

## 无需认证的端点

`GET /health` 和节点的 `GET /metrics` 不在 API key 身份认证范围内。Gateway 在单独的 metrics 监听器上暴露 Prometheus 指标。请使用你的监控部署所使用的网络和认证控制来保护这些端点。

## 轮换 API Key

更改 `AENV_API_KEY` 会使现有 API 客户端失效，但不会更改现有 Sandbox 的凭据。
