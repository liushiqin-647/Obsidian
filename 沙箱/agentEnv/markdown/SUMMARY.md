# 目录

# 入门

- [概览](./getting-started/overview.md)
- [快速上手](./getting-started/quickstart.md)
- [按需加载](./getting-started/on-demand-loading.md)
- [aenv CLI 参考](./getting-started/aenv-cli/index.md)
  - [认证](./getting-started/aenv-cli/authentication.md)
  - [Sandbox](./getting-started/aenv-cli/sandboxes.md)
  - [模板](./getting-started/aenv-cli/templates.md)
  - [快照](./getting-started/aenv-cli/snapshots.md)
  - [卷](./getting-started/aenv-cli/volumes.md)
  - [Shell 补全](./getting-started/aenv-cli/shell-completion.md)

# 部署

- [Docker（单节点）](./deployment/docker.md)
- [Docker Compose（多节点模拟）](./deployment/docker-compose.md)
- [静态多节点（不使用 Kubernetes）](./deployment/static-multi-node.md)
- [Kubernetes（多节点）](./deployment/kubernetes.md)
- [手动编译（单节点）](./deployment/manual-compile.md)
- [PVM 部署（当 KVM 不可用时）](./deployment/pvm.md)

# 使用场景

- [运行 Python 代码](./use-cases/python.md)
- [在 AgentENV 中运行 Codex](./use-cases/codex.md)
- [使用 Miles 训练 Terminal-Bench-2](./use-cases/miles.md)

# 核心概念

- [AgentENV 的工作原理](./concepts/overview.md)
- [认证](./concepts/authentication/index.md)
  - [API 认证](./concepts/authentication/api-authentication.md)
  - [应用入口认证](./concepts/authentication/application-ingress.md)
  - [安全 Sandbox 认证](./concepts/authentication/secure-sandbox.md)
  - [Sandbox 访问令牌种子](./concepts/authentication/access-token-seed.md)
- [Sandbox](./concepts/sandboxes/index.md)
  - [启动 Sandbox](./concepts/sandboxes/starting.md)
  - [使用 Sandbox](./concepts/sandboxes/working.md)
  - [自动驱逐](./concepts/sandboxes/auto-eviction.md)
  - [网络](./concepts/sandboxes/networking.md)
  - [代理](./concepts/sandboxes/proxy.md)
- [Template](./concepts/templates/index.md)
  - [创建你的 Template](./concepts/templates/creating.md)
  - [管理 Template](./concepts/templates/managing.md)
- [Snapshot](./concepts/snapshots/index.md)
  - [创建 Snapshot](./concepts/snapshots/create.md)
  - [管理 Snapshot](./concepts/snapshots/manage.md)
  - [将 Rootfs 作为 OCI 镜像](./concepts/snapshots/use.md)
  - [可选的 P2P 可见性](./concepts/snapshots/p2p.md)
- [卷](./concepts/volumes/index.md)
  - [创建并挂载卷](./concepts/volumes/create-and-mount.md)
  - [管理卷](./concepts/volumes/manage.md)
  - [Sandbox Fork 与 Snapshot](./concepts/volumes/sandbox-forks-and-snapshots.md)
- [自定义扩展](./concepts/custom-extension/index.md)
  - [生命周期钩子](./concepts/custom-extension/lifecycle-hooks.md)
  - [使用扩展](./concepts/custom-extension/connect-and-use.md)

# API 参考

- [API 参考](./api/index.md)

# 集成

- [E2B](./integration/e2b.md)

# 故障排查

- [常见问题](./troubleshooting/common-issues.md)

# 配置

- [Gateway 与 Scheduler 配置](./configuration/control-plane-config.md)
- [运行时节点配置](./configuration/reference.md)
- [环境变量](./configuration/env-vars.md)

---

# 开发者内部原理

- [系统架构](./internals/architecture.md)
- [Sandbox 网络架构](./internals/networking.md)
- [Sandbox 内部原理与测试](./internals/sandbox-testing.md)
- [Template 构建器与测试](./internals/template-builder-testing.md)
- [BuildKit Template 构建](./internals/buildkit-template-builds.md)
- [持久化产物清单](./internals/persistence-artifact-inventory.md)
- [代理设计](./internals/proxy-design.md)
- [分布式控制平面](./internals/services.md)
- [P2P 产物传输](./internals/p2p-design.md)
