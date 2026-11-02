# 沙盒访问令牌种子

AgentENV 从 Sandbox 标识和一个随机访问令牌种子（access-token seed）派生 `trafficAccessToken` 和 `envdAccessToken`。该种子独立于 AgentENV API key。

在集群部署中，请在每一个可能创建、承载、恢复或还原 Sandbox 的 runtime 节点上配置相同的种子。Gateway（网关）不派生 Sandbox 访问令牌，也不需要此种子。

## 使用默认生成的种子

默认情况下，无需手动配置。当未配置种子时，AgentENV 会在首次启动时生成一个种子，并将其持久化到 `$AENV_HOME/secrets/sandbox-access-token-hash-seed`。之后的启动会复用相同的值。

## 提供你自己的种子

要提供你自己的种子，请生成一个值：

```bash
ACCESS_TOKEN_SEED="$(openssl rand -hex 32)"
```

然后使用以下方法之一进行配置。

### 1. 环境变量

在启动 AgentENV 之前设置该环境变量：

```bash
export AENV_SANDBOX_ACCESS_TOKEN_HASH_SEED="$ACCESS_TOKEN_SEED"
```

### 2. AgentENV 配置文件

在 `config/default.toml` 中，或在 `AENV_CONFIG_PATH` 所选的配置文件中设置该值：

```toml
[sandbox]
access_token_hash_seed = "<generated-seed>"
```

### 3. Kubernetes Secret

Kubernetes 部署会将共享值存储在 `Secret/agentenv-runtime-secrets` 中 `sandbox-access-token-hash-seed` 键下。

在升级过程中请保留该种子。更改它会轮换两种 Sandbox 令牌，并且需要同时更新所有 runtime 节点。
