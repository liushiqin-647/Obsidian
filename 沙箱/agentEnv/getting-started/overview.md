# 入门

AgentENV（缩写为 AENV）是一个面向 AI 智能体的自托管 Sandbox（沙盒）运行时。它运行隔离的 Firecracker microVM，并暴露兼容 E2B 的 HTTP API——因此现有的 E2B SDK 代码无需修改即可在其上运行。
仓库位于 <https://github.com/kvcache-ai/AgentENV>。

## 为什么选择 AgentENV

- **跨多样环境扩展**：AENV 可在多台机器上运行海量 Firecracker 环境，并支持多种兼容 OCI 的镜像，通过 overlaybd 按需加载。本地磁盘充当有界缓存，保留热数据并淘汰冷数据，因此镜像总量可以超过磁盘容量，同时集群范围内的启动仍然很快，无需预热每台主机。
- **让空闲环境成本低廉**：由 Snapshot（快照）支持的环境可在 50 ms 内启动或恢复，并在 100 ms 内暂停。空闲环境可快速释放 CPU 和内存，然后在新任务到来时恢复。
- **原生 Snapshot 与 fork 支持**：AENV 增量地对内存和文件系统变更进行 Snapshot（Snapshot），即使磁盘修改繁重也能在 100 ms 内完成。运行中的环境可以 fork 成多个独立的 Sandbox（Sandbox），用于并行智能体工作流。Snapshot 可持久化到兼容 S3 的对象存储或共享分布式文件系统，以防止数据丢失。
- **长期保持性能与密度**：AENV 通过 ublk 提供高性能 I/O，同时在存储和内存 Snapshot 数据之间共享主机页缓存。内存 ballooning 将可回收的客户机内存归还给主机，随着环境运行时间变长且逐渐分化，仍能维持高 overcommit。

## 特性

- **Firecracker microVM**，每个 Sandbox 具备完整的 Linux 内核隔离
- **暂停与恢复**，配合内存 + 磁盘 Snapshot 实现瞬时冷启动
- **分层块设备**，通过 overlaybd + ublk 实现写时复制镜像共享
- **由 Snapshot 支持的 Template（模板）构建器**，用于发布可复用、预配置的 Sandbox 运行时
- **兼容 E2B 的 API**，现有 E2B SDK 和 CLI 开箱即用
- **反向代理**，可通过 HTTP 和 WebSocket 访问 Sandbox 内运行的服务
- **多节点扩展**，采用 Gateway（网关）+ Scheduler（调度器）控制平面

## 适用对象

AgentENV 面向运行 AI 智能体、需要隔离执行环境的团队：代码解释器、工具调用智能体、自主编码智能体，或任何希望为每个任务提供全新（或缓存）Linux 环境的工作负载。

## 与服务器交互

AgentENV 暴露 HTTP API。有四种使用方式：

| 方式 | 最适合 |
|--------|----------|
| **[aenv CLI](./aenv-cli/index.md)** | 交互式使用、脚本编写、本地开发 |
| **[E2B](../integration/e2b.md)** | 应用代码——现有的基于 E2B 的应用无需修改即可与 AgentENV 配合使用 |
| **[HTTP API](../api/index.md)** | 直接控制、其他语言、自动化 |

## 下一步

- **[快速开始](./quickstart.md)** — 安装服务器，运行你的第一个 Sandbox。在受支持的 Linux 主机上约需 5 分钟。
- **[部署](../deployment/manual-compile.md)** — 从源码构建、Docker Compose 多节点或 Kubernetes。
- **[PVM 部署](../deployment/pvm.md)** — 在无法使用标准 KVM 的服务器上使用 AgentENV。
