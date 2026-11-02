# AgentENV 沙盒生命周期流程：冷启动 / 快照 / 暂停 / 恢复

> 本文总结从用户请求到沙盒**冷启动、快照创建、暂停、恢复**的运行逻辑，以及内部各组件如何交互完成。
> 代码主要位于 `src/api/`、`src/orchestrator/`、`src/sandbox/firecracker/`、`storage/ublk*`、`storage/overlaybd/`、`src/snapshot/`。

## 组件清单

| 组件 | 代码 | 职责 |
|---|---|---|
| **API** | `src/api/`（Axum） | 接 HTTP 请求、鉴权、路由 |
| **Orchestrator** | `src/orchestrator/service.rs` | 生命周期状态机、协调各组件 |
| **SandboxBackendFactory** | `src/sandbox/backend.rs` | 造 Sandbox（fresh / from_snapshot / from_paused_state） |
| **FirecrackerSandbox** | `src/sandbox/firecracker/` | 操作 Firecracker VM + 网络 + 设备 + envd |
| **ublk daemon** | `storage/ublk-daemon/` | 提供块设备（overlaybd 前台） |
| **overlaybd** | `storage/overlaybd/` | 分层镜像、按需拉块、快照层 |
| **SnapshotManager** | `src/snapshot/` | 快照 commit/resolve/delete（repository） |
| **NetworkManager / Slot** | `src/sandbox/network/` | netns、tap、IP、出站策略 |
| **envd** | guest 内 | 命令/文件、健康上报 |
| **store / persister** | `src/orchestrator/store`、`persistence` | 沙盒元数据、paused 状态持久化 |

## 一、冷启动（create from fresh image）

```
用户 POST /sandboxes
  → API → Orchestrator.create_sandbox()
      → 解析镜像/template → LaunchPlan::for_create_fresh
      → launch_sandbox(plan)：
        ① factory.build(fresh) → FirecrackerSandbox
        ② sandbox.startup_artifacts()      保护镜像引用（防 GC）
        ③ sandbox.start_nowait() → start_fresh：
             - link_tools_drive()           tools 盘（只读 ublk）
             - attach_user_rootfs()         用户 rootfs（overlaybd via ublk，可写）
             - prepare_extra_drives()       额外盘
             - allocate_network_slot()      分配 netns/tap/IP + 出站策略
             - 组装 boot args（init=/init、ip=、agentenv_drives=）
             - spawn_firecracker(netns)     在命名空间内启动 FC 进程
             - 通过 FC HTTP API：machine-config / boot-source / drives /
               network-interfaces / mmds / balloon → InstanceStart
        ④ store.add(过渡状态 Creating)
        ⑤ sandbox.wait_for_ready()          等 guest 内 envd 就绪（TCP → ip:49983）
        ⑥ 状态 → Running，发布 proxy 路由
```

**交互要点**：Orchestrator 指挥 Factory 造沙盒；FirecrackerSandbox 调 **ublk daemon** 拿块设备、调 **NetworkManager** 拿网络槽位、调 **Firecracker API** 配置并启动 VM；最后确认 **envd** 就绪才算 Running。

## 二、快照创建（snapshot from running sandbox，原创继续运行）

```
用户/内部 触发 snapshot
  → FirecrackerSandbox.snapshot() → snapshot_to_dir()：
     ① 内存：
        fc_instance.get_dirty_memory_ranges()      GET /vm/dirty-memory-ranges
        convert_dirty_memory_to_overlaybd()        process_vm_readv 读取脏页
                                                   → 生成 overlaybd 内存层
        build_mem_snapshot_image_config()          父层堆叠成完整内存镜像
     ② 文件系统：
        restack_snapshot_overlaybd_rootfs()        调用 ublk daemon 封存(seal)
                                                   upper 层 → 新只读层
  → CapturedSandboxSnapshot
  → SnapshotManager.publish()                      落 repository（S3/POSIX）
```

**交互要点**：Firecracker 提供脏页范围，AgentENV 用 `process_vm_readv` 读内存并经 **overlaybd** 生成内存层；**ublk daemon** 负责文件系统层封存。**原创沙盒不受影响**。

## 三、暂停（pause）

```
用户 POST /sandboxes/{id}/pause
  → Orchestrator.pause_sandbox() → pause_sandbox_impl()：
     ① protect runtime artifacts              保护镜像/运行时引用
     ② persister.allocate_artifact_root()     分配 paused 持久化目录
     ③ sandbox.pause(artifact_root)：
          = snapshot_to_dir（同二：内存 + 文件系统封存）
          → 得到 FirecrackerSnapshotConfig（paused_state）
          → Firecracker 停止（只留 state/memory 快照，不占 CPU/内存）
     ④ persistence 保存 paused 状态
     ⑤ 状态 → Paused
```

**与"快照创建"的区别**：
- **Snapshot**：捕获一份可复用的快照（发布到 repository），**原创继续跑**；
- **Pause**：捕获 paused 状态并**停止 VM**，状态变 **Paused**（不占资源，可原地 Resume）。

## 四、恢复（resume）

```
用户 POST /sandboxes/{id}/resume
  → Orchestrator.resume_sandbox() → resume_sandbox_inner()
      → LaunchPlan::for_resume（过渡状态 Resuming）
      → launch_sandbox(plan)：
        ① factory.build_from_paused_state() → FirecrackerSandbox
        ② start_resume()：
             - 检查 ublk daemon（内存恢复依赖它）
             - adopt_warm_firecracker()       优先用 warm pool 的进程+网络槽位
             - link_tools_drive()
             - 恢复用户 rootfs（overlaybd via ublk）
             - prepare_resume_drives()
             - allocate/spawn firecracker（非 warm 时）
             - SnapshotLoadParams + BackendType::File（指向 ublk 内存设备）
               → Firecracker mmap，COW 恢复内存
             - init_envd_instance()
        ③ wait_for_ready() → 状态 Running
        ④ 删除持久化的 paused 记录
```

**交互要点**：恢复同样靠 **ublk daemon** 提供只读内存设备（多沙盒可引用计数共享）、**overlaybd** 恢复文件系统层、**Firecracker** 加载快照并 COW 恢复内存，最后重新拉起 **envd**。

## 组件交互总览

```
                    ┌─────────────┐
   HTTP 请求 ───────►│   API       │
                    └──────┬──────┘
                           ▼
                    ┌─────────────┐   状态机 Creating/Running/Paused/Resuming
                    │ Orchestrator│──────────────────────────────┐
                    └──────┬──────┘                              │
               factory.build / build_from_snapshot /             │ store/persister
               build_from_paused_state                          │ (元数据/paused状态)
                           ▼                                     ▼
            ┌───────────────────────────────┐
            │     FirecrackerSandbox        │
            └───┬────────┬──────────┬───────┘
                │        │          │
      ┌─────────▼──┐ ┌───▼───────┐ ┌▼──────────────┐
      │ ublk daemon│ │ Network   │ │ Firecracker    │
      │ (overlaybd)│ │ Manager   │ │ (VM + API)     │
      └─────┬──────┘ └───────────┘ └───────┬────────┘
            │                             │
      ┌─────▼──────┐                ┌─────▼──────┐
      │ overlaybd  │                │ guest envd │
      │ /repository│                │ (ip:49983) │
      └────────────┘                └────────────┘
```

## 一句话总结

**API 收请求 → Orchestrator 管状态并指挥 Factory 造沙盒 → FirecrackerSandbox 通过 ublk daemon/overlaybd 拿块设备、NetworkManager 拿网络、Firecracker API 配置/启动 VM、envd 确认就绪。**
冷启动是"建 VM"；快照创建是"读脏页 + 封存层并发布、原创继续跑"；暂停是"捕获并停 VM、状态转 Paused"；恢复是"重新拉起 VM、用 ublk 内存设备 + 快照 COW 恢复、重启 envd"。

## 相关代码位置

| 流程 | 入口 |
|---|---|
| 冷启动 | `Orchestrator::create_sandbox` / `launch_sandbox`；`FirecrackerSandbox::start_fresh` |
| 快照创建 | `FirecrackerSandbox::snapshot` → `snapshot_to_dir` / `snapshot_memory_to_overlaybd` |
| 暂停 | `Orchestrator::pause_sandbox` → `pause_sandbox_impl`；`FirecrackerSandbox::pause` |
| 恢复 | `Orchestrator::resume_sandbox` → `resume_sandbox_inner`；`FirecrackerSandbox::start_resume` |
