# AgentENV 内部运行原理学习路径

核心原则：**先跟一条完整请求走通，再逐层深入**，不要一上来啃存储代码。

## 阶段 0：建立整体图景（半天）

| 读什么 | 看什么 |
|---|---|
| `concepts/overview.md` | 三个概念 Template / Sandbox / Snapshot、请求流程 |
| `internals/architecture.md` | 组件表、单节点/分布式架构图 |
| `SUMMARY.md` | 全局目录，知道有哪些主题 |

目标：能画出「客户端 → API → Orchestrator → Firecracker VM → 存储」这条线。

## 阶段 1：跟一条"创建 Sandbox"的主线（1-2 天）

最有效的切入方式：沿代码走一遍 `POST /sandboxes`：

```
src/bin/server.rs                 节点入口（怎么启动）
  → src/api/                      HTTP 路由、鉴权
  → src/orchestrator/             生命周期状态机（Creating→Running）
  → src/sandbox/firecracker/      创建 microVM
  → src/sandbox/ublk/             挂块设备（rootfs/内存）
```

配套读：`concepts/sandboxes/starting.md`、`concepts/sandboxes/working.md`、`internals/services.md`。

目标：搞清"一个请求如何变成一台运行中的 VM"。

## 阶段 2：拿捏核心存储子系统（最硬核，2-3 天）

AgentENV 的精华都在这，建议**先读文档再读代码**：

| 主题 | 文档 | 源码 |
|---|---|---|
| 分层镜像（LSMT） | `internals/architecture.md` 存储章节 | `storage/overlaybd/src/lsmt/`、`image/image_file.rs` |
| 用户态块设备 | 同上 | `storage/ublk/src/`（ctrl / dev / queue） |
| io_uring 抽象 | 同上 | `storage/util/src/io_ring/` |

建议顺序：`lsmt/format.rs`（格式）→ `lsmt/index.rs`（段映射）→ `image_file.rs`（读写路径）→ `ublk/queue.rs`（I/O 分发）。

目标：能说清"一次读请求如何从 VM 穿透到镜像层"。

## 阶段 3：快照与恢复（1-2 天）

| 主题 | 文档 | 源码 |
|---|---|---|
| 快照创建/恢复 | `concepts/snapshots/*`、`internals/architecture.md` | `src/snapshot/`、`image/snapshot.rs` |
| 内存快照 + COW | `internals/architecture.md` | `src/sandbox/ublk/`、`storage/overlaybd` 内存层 |
| 共享（引用计数） | 同上 | 同上 |

目标：理解"暂停 <100ms、恢复 <50ms、多 Sandbox 共享同一内存镜像"的实现。

## 阶段 4：网络 / 代理 / 分布式（按需 1-2 天）

- `internals/networking.md` → `src/sandbox/` 网络命名空间
- `internals/proxy-design.md` → `src/api/proxy.rs`
- `internals/services.md` → `services/gateway`、`services/scheduler`（Go）

## 阶段 5：动手验证（贯穿始终）

1. **跑起来**：按 `deployment/docker.md` 或 `deployment/manual-compile.md` 本地启动，用 `aenv pull` / `aenv start` / `aenv exec` 走一遍
2. **读测试**：测试是理解行为的最好注释，看 `internals/sandbox-testing.md`，跑相关单测
3. **加日志/断点**：在 Orchestrator 生命周期、ublk I/O 路径打日志，观察真实调用

## 学习技巧

- **抓主线不贪全**：先搞懂"创建 / 暂停 / 恢复"三条链路，其它都是支线
- **对照源码读文档**：文档给全局，代码给细节
- **画图**：每阶段结束画一张时序图/数据流图（如"读 I/O 穿透图""Sandbox 生命周期状态机"）
- **从测试入手**：`e2e-tests`、`benchmarks` 目录能直接看到典型用法
- **优先存储**：AgentENV 的差异化价值在 overlaybd + ublk + 快照，值得多花时间

**推荐起步**：先读 `concepts/overview.md` + `internals/architecture.md`，然后从 `src/bin/server.rs` 和 `src/api/` 顺着 `POST /sandboxes` 走一遍主线。
