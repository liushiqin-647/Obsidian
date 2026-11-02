# 生命周期 Hook

为支持完整的沙盒生命周期，你的扩展应实现以下四个 API。当相应事件发生时，AgentENV 会向 `POST {url}/sandbox-hook/<hook>` 发送 JSON 请求。任何连接错误、超时或非 2xx 响应都会使相应的 Sandbox 操作失败（`stop` 除外，它是尽力而为的）。

返回 4xx 状态的 hook 响应会以客户端错误拒绝该 API 请求：create、cold-create、fork、connect 和 patch-params 为 400，resume 为 409。返回 5xx 状态的 hook 响应会产生 500 错误。错误消息会包含扩展的状态码和响应体。对于无效的调用方参数请使用 4xx，对于扩展内部故障请使用 5xx。

| Hook | 触发时机 | 请求 | 响应 |
|------|------|---------|----------|
| `start-fresh` | 全新 Sandbox 启动之前，在其网络槽位分配之后 | `sandboxId`、`sandboxInstanceId`、`networkNamespacePath`、`hostInteractionIp`、`customExtensionParams` | 可选的 `extraBootArgs`，追加到内核 cmdline |
| `start-resume` | Sandbox 从快照恢复之前（模板启动、暂停后恢复、分叉子 Sandbox） | 同上 | 无 |
| `patch-params` | 当用户 PATCH Sandbox 参数时 | `sandboxId`、`patch`（用户请求体的原样内容） | 更新后的**完整** `customExtensionParams` |
| `stop` | Sandbox runtime 被拆除时，在网络槽位释放之前 | `sandboxId`、`sandboxInstanceId` | 无 |

说明：

- **实例标识。** `sandboxId` 会在暂停/恢复周期中重复使用。每次 `start-fresh` / `start-resume` 都会携带一个新的 `sandboxInstanceId`，用于标识该 runtime 实例，后续的 `stop` 会携带相同的值。由于 `stop` 是尽力而为的且可能被重排序（例如某次暂停的 `stop` 在恢复的 `start-resume` 之后才到达），请将 `(sandboxId, sandboxInstanceId)` 视为运行实例的标识，并忽略那些 `sandboxInstanceId` 不是该 Sandbox 最新启动实例的 `stop` 通知。
- **`stop` 在暂停时也会触发。** 暂停会持久化 Sandbox 状态，然后停止 VM 进程并释放网络命名空间；随后的恢复会创建新的 runtime 并触发 `start-resume`。Snapshot 捕获期间的就地暂停+恢复不会触发任何 hook（并保持相同的 `sandboxInstanceId`）。
- `stop` 是尽力而为的：投递失败只会被记录，并且如果已启动的 Sandbox 在没有显式 stop 的情况下被丢弃，也会以 fire-and-forget 方式触发。
- `networkNamespacePath` 是 Sandbox netns 文件在宿主机上的路径（例如 `/var/run/netns/agentenv-ns-*`），因此扩展可以进入该命名空间（例如 `nsenter --net=...`）来设置防火墙规则或 VPN 接口。
- `hostInteractionIp` 是 AgentENV 路由到该 Sandbox 的每 runtime IPv4 地址。它可能在暂停/恢复后发生变化，因此扩展必须使用当前 start hook 中的值，而不是缓存旧值。
- 对同一 Sandbox 的并发 `patch-params` 调用不会被串行化；如果你的修补语义不满足可交换性，请在扩展中处理并发。

## 最小扩展示例

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

# Latest started runtime instance per sandbox: (sandboxId, sandboxInstanceId)
# is the identity of a running instance; a stop for a superseded instance
# (e.g. arriving after a newer start) is ignored.
latest_instance: dict[str, str] = {}

@app.post("/sandbox-hook/start-fresh")
async def start_fresh(req: Request):
    body = await req.json()
    latest_instance[body["sandboxId"]] = body["sandboxInstanceId"]
    # e.g. nsenter --net={body["networkNamespacePath"]} wg-quick up ...
    return {"extraBootArgs": None}

@app.post("/sandbox-hook/start-resume")
async def start_resume(req: Request):
    body = await req.json()
    latest_instance[body["sandboxId"]] = body["sandboxInstanceId"]
    return {}

@app.post("/sandbox-hook/patch-params")
async def patch_params(req: Request):
    body = await req.json()
    # apply body["patch"] however you like, then return the full new params
    return {"customExtensionParams": body["patch"]}

@app.post("/sandbox-hook/stop")
async def stop(req: Request):
    body = await req.json()
    if latest_instance.get(body["sandboxId"]) == body["sandboxInstanceId"]:
        latest_instance.pop(body["sandboxId"], None)
        # tear down resources for this instance
    return {}
```

任何非 2xx 响应或超时都会使相应的 Sandbox 操作失败，`stop` 除外，它始终是被容忍的。
