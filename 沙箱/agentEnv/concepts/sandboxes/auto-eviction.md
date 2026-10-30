# 自动驱逐（Auto-Eviction）

每个运行中的 Sandbox（沙盒）都有一个存活时间（time-to-live，TTL）。TTL 设定了一个过期截止时间，使 Sandbox 无法无限期地占用 CPU 和内存。当达到 TTL 时，AgentENV 会自动暂停或删除该 Sandbox，以便回收这些资源。

## 过期时的行为

当 Sandbox 达到其 TTL 时，AgentENV 会执行其配置的超时动作：

- **暂停**（`autoPause: true`，默认）：保留 Sandbox，以便之后恢复。
- **删除**（`autoPause: false`）：永久移除 Sandbox。

超时动作在创建 Sandbox 时选定。`aenv start` 命令使用默认动作，即 `autoPause: true`。若要在过期时改为删除，请通过 API 以 `autoPause: false` 创建 Sandbox。

从 Template（模板）或 Snapshot（快照）热启动：

```bash
curl -X POST \
  -H 'X-API-Key: test-key' \
  -H 'Content-Type: application/json' \
  -d '{
    "templateID": "my-template",
    "timeout": 600,
    "autoPause": false
  }' \
  http://127.0.0.1:8000/sandboxes
```

从 OCI 镜像冷启动：

```bash
curl -X POST \
  -H 'X-API-Key: test-key' \
  -H 'Content-Type: application/json' \
  -d '{
    "image": "ubuntu:24.04",
    "timeout": 600,
    "autoPause": false
  }' \
  http://127.0.0.1:8000/sandboxes-cold
```

## 设置或延长截止时间

`aenv start --timeout <seconds>` 设置初始 TTL。如果需要再次使用某个已自动暂停的 Sandbox，`aenv resume --timeout <seconds>` 会恢复它，并从恢复时刻起设置新的 TTL。两个命令的默认值均为 300 秒。

对于运行中的 Sandbox，可用从当前时刻起算的精确秒数替换其截止时间：

```bash
aenv timeout <sandbox-id> 600
```

这会将截止时间设为从发送命令时刻起 600 秒。再次调用会替换之前的截止时间，因此它既可以延长也可以缩短剩余时间。

若要在不缩短之后已有截止时间的前提下让运行中的 Sandbox 继续存活，请使用 refresh API：

```bash
curl -X POST \
  -H 'X-API-Key: test-key' \
  -H 'Content-Type: application/json' \
  -d '{"duration": 600}' \
  http://127.0.0.1:8000/sandboxes/<sandbox-id>/refreshes
```

如果当前截止时间更晚，refresh 不会缩短剩余 TTL。refresh 仅适用于运行中的 Sandbox；请先恢复已暂停的 Sandbox。如果省略 `duration`，则使用服务器的默认 Sandbox 超时。

`aenv connect` 在连接时会恢复已暂停的 Sandbox，并确保其 TTL 至少为默认的 300 秒。
