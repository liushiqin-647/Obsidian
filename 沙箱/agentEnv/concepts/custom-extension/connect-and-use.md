# 使用扩展

本页展示如何将 AgentENV 服务器连接到扩展并使用它。

## 将 AgentENV 连接到扩展

通过配置 URL 将 AgentENV 连接到你的扩展服务：

```toml
 config/default.toml (or your AENV_CONFIG_PATH)
[custom_extension]
url = "http://127.0.0.1:9090"
 timeout_ms = 5000   # optional, per-call timeout in milliseconds
```

`AENV_CUSTOM_EXTENSION_URL` 同样有效。当 `url` 未设置时，该集成完全禁用：不会调用任何 hook，且 `customExtensionParams` 必须为空。

## 使用扩展

使用 `customExtensionParams` 为沙盒传递扩展特有的设置。它是一个不透明的 JSON 对象，仅由你的扩展解释。缺省值和空对象是等价的。

### 在创建时设置

`POST /sandboxes` 和 `POST /sandboxes-cold` 都接受 `customExtensionParams`。例如，从模板创建一个 Sandbox，并为扩展提供 VPN 设置：

```bash
curl -X POST http://127.0.0.1:8000/sandboxes \
  -H 'X-API-Key: test-key' \
  -H 'Content-Type: application/json' \
  -d '{
    "templateID": "my-template",
    "customExtensionParams": {
      "vpn": { "network": "team-a" }
    }
  }'
```

对于冷启动 Sandbox，请在冷启动请求中包含相同的字段：

```bash
curl -X POST http://127.0.0.1:8000/sandboxes-cold \
  -H 'X-API-Key: test-key' \
  -H 'Content-Type: application/json' \
  -d '{
    "image": "docker.io/library/ubuntu:24.04",
    "customExtensionParams": {
      "vpn": { "network": "team-a" }
    }
  }'
```

### 读取

获取当前参数。当参数为空时，AgentENV 返回 `{}`：

```bash
curl http://127.0.0.1:8000/sandboxes/<sandbox-id>/custom-extension-params \
  -H 'X-API-Key: test-key'
```

### 修补

请求体会原样传递给扩展的 `patch-params` hook；其语义完全由扩展定义。该 hook 返回更新后的完整参数，AgentENV 会将其存储并返回：

```bash
curl -X PATCH http://127.0.0.1:8000/sandboxes/<sandbox-id>/custom-extension-params \
  -H 'X-API-Key: test-key' \
  -H 'Content-Type: application/json' \
  -d '{"vpn": {"network": "team-a", "peers": ["10.8.0.2", "10.8.0.3"]}}'
```

### 持久化

参数在暂停/恢复后仍然保留，并会被存储到从该 Sandbox 创建的快照中。从 Template 启动时，在创建时提供的 `customExtensionParams` 会覆盖 Snapshot 中存储的值；否则会继承 Snapshot 中的值。
