# aenv CLI 参考

`aenv` 是 AgentENV 的原生 CLI。它将 HTTP API 和 envd gRPC 端点封装成对开发者友好的界面，用于管理 Template（模板）、Sandbox（沙盒）、持久 Volume（卷）和 Snapshot（快照）。它还通过 [`aenv codex`](../../use-cases/codex.md) 在项目的 Sandbox 副本上运行原生 Codex 会话。

## 安装

```bash
curl -fsSL https://raw.githubusercontent.com/kvcache-ai/AgentENV/main/scripts/install-cli.sh | bash
```

或从源码构建（需要 Rust）：

```bash
git clone https://github.com/kvcache-ai/AgentENV.git
cd AgentENV
make install-aenv
```

