# E2B

## E2B SDK

AgentENV 暴露兼容 E2B 的 API，因此官方 [E2B SDK](https://github.com/e2b-dev/e2b) 开箱即用。

### 通用设置

设置环境变量指向你的 AgentENV 服务器。各部署模式的值请参见[环境变量](../configuration/env-vars.md)。

```bash
# Single-node example
export E2B_API_URL=http://127.0.0.1:8000
export E2B_SANDBOX_URL=${E2B_API_URL}
export E2B_API_KEY=${AENV_API_KEY}
```

当 `network.allowPublicTraffic` 为 false 时，AgentENV 返回 `trafficAccessToken`；对于安全 Sandbox（沙盒），还会为 envd 控制流量返回 `envdAccessToken`。这些凭据具有不同的头部和信任边界：私有应用路由使用 `e2b-traffic-access-token`，而 `X-Access-Token` 仅用于 envd。公共应用路由不需要任何 token。

### TypeScript SDK

#### 设置

安装 SDK：

```bash
npm install e2b
```

#### 用法

```typescript
import { Sandbox } from "e2b";

// Create a sandbox from a template
const sandbox = await Sandbox.create("<template-id>", {
  apiKey: process.env.E2B_API_KEY,
});

// List running sandboxes
const running = Sandbox.list({
  apiKey: process.env.E2B_API_KEY,
  limit: 20,
  query: { state: ["running"] },
});
console.log(await running.nextItems());

// Run a command inside the sandbox
sandbox.commands.run("echo hello world");

// Pause the sandbox
await Sandbox.Pause(sandbox.sandboxId, {
  apiKey: process.env.E2B_API_KEY,
});

// Kill the sandbox
await sandbox.kill();
```

将 `<template-id>` 替换为本地 Template（模板）存储中存在的 Template。使用 `e2b template list` 或 `GET /v2/templates` 查看可用的 Template。

### 卷挂载

TypeScript SDK 可以在创建 Sandbox 时直接创建 Volume（卷）并传入：

```typescript
import { Sandbox, Volume } from "e2b";

const volume = await Volume.create("workspace-volume", {
  apiKey: process.env.E2B_API_KEY,
});
const sandbox = await Sandbox.create("<template-id>", {
  apiKey: process.env.E2B_API_KEY,
  volumeMounts: {
    "/workspace": volume,
  },
});
```

对于 Python SDK，使用 `aenv volume create` 或 `POST /volumes` 创建 Volume，然后在创建 Sandbox 时传入其名称：

```python
sandbox = Sandbox.create(
    "<template-id>",
    volume_mounts={"/workspace": "workspace-volume"},
)
```

AgentENV 支持 TypeScript SDK 的 Volume 创建、列出和删除操作，以及通过 Sandbox 访问挂载的文件系统。
不支持 E2B SDK 的直接 Volume 内容 API。

### Python SDK

#### 设置

安装 SDK：

```bash
pip install e2b
```

#### 用法

```python
from e2b import Sandbox, SandboxQuery, SandboxState

# Reuse the environment variables set in your shell:
# E2B_API_URL / E2B_SANDBOX_URL / E2B_API_KEY

# Create a sandbox from a template
sandbox = Sandbox.create("<template-id>")

# List running sandboxes
running = Sandbox.list(
    limit=20,
    query=SandboxQuery(state=[SandboxState.RUNNING]),
)
print(running.next_items())

# Run a command inside the sandbox
result = sandbox.commands.run("echo hello world")
print(result.stdout, end="")

# Pause the sandbox
sandbox.beta_pause()

# Kill the sandbox
sandbox.kill()
```

## E2B CLI

AgentENV 与 E2B CLI 兼容，但我们建议针对 AgentENV 工作流使用 [aenv CLI](../getting-started/aenv-cli/index.md)。
