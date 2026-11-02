# AgentENV 补充文档（addtion）

> AgentENV 官方文档中**缺失或分散**的关键概念补充（已多轮合并去重，只留干货）。
> 官方原文：`../AgentENV/docs/src/`；中文译本：`../markdown/`。

## 📚 文档索引

| 文档 | 讲什么 |
|---|---|
| [`sandbox-vs-container.md`](./sandbox-vs-container.md) | 沙箱/容器概念澄清；**沙箱为什么需要隔离性**（信任边界、5 原因、威胁模型、维度）；VM 沙箱 vs 容器对比；AgentENV 选微VM 的理由 |
| [`orchestrator.md`](./orchestrator.md) | 编排器：职责、8 态状态机、能力（创建/恢复/暂停/快照/Fork/删除/TTL/重启恢复）、可靠性设计 |
| [`architecture-and-lifecycle.md`](./architecture-and-lifecycle.md) | 组件与分层、三条交互主线、**冷启动/快照/暂停/恢复**四流程、通信通道（Firecracker Unix socket / envd TCP / vsock 澄清）、依赖表 |
| [`storage-overlaybd-ublk.md`](./storage-overlaybd-ublk.md) | **overlaybd**：OCI 转换、zstd、LSMT、按块懒加载（索引+jump table+HTTP Range）、有界缓存；**ublk / daemon**：内核框架、daemon 职责、三类设备、与 Firecracker 协作、为什么用户态 |

## 🧭 建议阅读顺序

1. `sandbox-vs-container.md` — 为什么需要隔离/微VM
2. `orchestrator.md` — 每节点的沙盒总管
3. `architecture-and-lifecycle.md` — 组件如何协作 + 四条生命周期
4. `storage-overlaybd-ublk.md` — 块设备与镜像的按块懒加载

## 📌 相关

- 总体架构（官方）：`../markdown/internals/architecture.md`
- 学习路径：`../markdown/learning-path.md`
- 官方实测数据：`../docs-external/k3_tech_report.pdf`（Kimi K3 技术报告 §5.3.2）

## 🗂 整理记录

- 首轮：`sandbox-isolation.md` 并入 `sandbox-vs-container.md`
- 二轮：`component-interactions.md` + `sandbox-lifecycle-flows.md` + `firecracker-envd-communication.md` → `architecture-and-lifecycle.md`
- 二轮：`overlaybd-oci-conversion.md` + `ublk-kernel-daemon.md` → `storage-overlaybd-ublk.md`
- 三轮（去糟粕）：删除各篇重复的"交互路径/流程/总结/类比"，压缩表格与措辞，保留机制与代码位置
