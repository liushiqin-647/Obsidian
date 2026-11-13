# 虚拟 CPU（KVM 视角）

> [!info] 先记住三句话
> 1. 虚拟机里看到的每一颗 CPU，叫 **vCPU**。
> 2. 在 KVM 里，一个 vCPU 对应宿主上的**一个线程**，这个线程循环执行 `ioctl(KVM_RUN)`。
> 3. KVM 本身不调度、不模拟指令：它只负责**把物理 CPU 借给 guest 跑**，跑不下去时把原因交回用户态。

## 第 0 章：先分清四个角色

| 名字 | 是什么 | 住在哪 |
|---|---|---|
| **物理 CPU** | 真芯片，比如 8 核 16 线程 | 硬件 |
| **线程** | Linux 调度的最小单位，一个核轮流跑很多线程 | 宿主内核 |
| **vCPU** | 虚拟机内部的一颗"CPU"，KVM 里的一个对象 | KVM |
| **KVM** | Linux 内核模块，通过 `/dev/kvm` 提供虚拟化接口 | 宿主内核 |

关系一句话：

> QEMU 建一个线程 → 这个线程拿着 vCPU 的 fd 调 `KVM_RUN` → KVM 把物理核交给 guest → 出事再交还。

## 第 1 章：一个 vCPU 在 KVM 里是怎么被造出来的

整条链只有五步，全部是 ioctl：

```
open("/dev/kvm")                    → kvm_fd      拿到 KVM 大门
  └─ KVM_CREATE_VM                  → vm_fd       造一台"空虚拟机"
       └─ KVM_CREATE_VCPU(0)        → vcpu_fd     造 0 号 vCPU
            ├─ KVM_SET_CPUID2                     告诉它"你是什么型号的 CPU"
            ├─ KVM_SET_REGS / SREGS               设定开机时的寄存器（CS:IP 指向哪）
            └─ mmap(vcpu_fd)        → kvm_run     一块共享内存，用来收"退出原因"
                 └─ ioctl(vcpu_fd, KVM_RUN)       开跑
```

分工很清楚：

- **QEMU（用户态）** 负责前面所有准备工作：造 VM、造 vCPU、装 CPU 型号、装寄存器、装内存、装设备。
- **KVM（内核态）** 只负责最后那一步：真正让 CPU 进入 guest 执行。

`KVM_CREATE_VCPU` 的参数就是编号 `0, 1, 2...`，这个编号就是 guest 里的 CPU 序号。

## 第 2 章：`KVM_RUN` 到底做了什么

每个 vCPU 线程都在跑这个循环：

```c
while (1) {
    ioctl(vcpu_fd, KVM_RUN, 0);   // ① 进 guest
    handle_exit(&run->exit_reason); // ② 出来以后看原因，处理
}
```

① 在 KVM 内部做的事，可以拆成四步：

```
KVM_RUN
 ├─ vcpu_load()        把这个 vCPU 的状态装进当前物理核（VMCS 等）
 ├─ 检查有没有待处理的中断、事件
 ├─ VM Entry           执行 VMLAUNCH / VMRESUME，CPU 进入 non-root 模式
 │     └─ guest 代码就在这里原生执行，一条条跑在物理核上
 └─ VM Exit            CPU 回到 root 模式，KVM 把原因写进 kvm_run
```

然后 `KVM_RUN` 返回用户态，QEMU 读 `run->exit_reason`，决定怎么办。

**关键**：guest 的普通指令不会被拦截，是原生执行的。KVM 只在"guest 不能自己决定"的时刻才拿回控制权。所以虚拟机的计算性能和物理机接近，开销几乎全在进出这一下。

## 第 3 章：退出原因——KVM 把"球"踢给谁

`exit_reason` 是 KVM 和 QEMU 之间唯一的交接单据。常见几种：

| `KVM_EXIT_*` | 什么情况 | 谁来处理 |
|---|---|---|
| `IO` | guest 读写端口（老式设备） | QEMU，模拟设备寄存器 |
| `MMIO` | guest 读写设备内存 | QEMU |
| `EPT_VIOLATION` | guest 访问的"物理内存"还没映射 | 多数情况 KVM 自己补页表，不惊动 QEMU |
| `HLT` | guest 执行 `hlt`，没活干了 | KVM 让线程睡，等中断再醒 |
| `IRQ_WINDOW_OPEN` | guest 关中断，暂时不能投递 | KVM 等它开中断 |
| `SHUTDOWN` | guest  triple fault，崩了 | QEMU 决定重启还是关机 |
| `INTR` | 线程收到信号（比如被 kill） | QEMU 退出循环 |

这张表说明一件事：**KVM 管 CPU 和内存，QEMU 管设备。** 磁盘、网卡这些"外设"的模拟都在用户态，所以每次设备 I/O 都要出一次 guest——这正是 [[virtio-blk 前后端]] 里 virtio、ioeventfd 要消灭的开销。

## 第 4 章：guest 里的"2 颗 CPU"是怎么被编出来的

线程只是身体。要让 guest 相信自己有 2 颗 CPU，还得这几样东西配合：

| 编造物 | 谁提供 | 作用 |
|---|---|---|
| CPU 数量与编号 | QEMU 写 ACPI 表 | guest 启动时枚举出 CPU0、CPU1 |
| CPU 型号、特性 | `KVM_SET_CPUID2` | guest 读到的是 KVM 支持的指令集 |
| 中断控制器 | KVM 内核内建 LAPIC（`KVM_CREATE_IRQCHIP`） | vCPU 之间互发 IPI、收时钟中断 |
| 时钟 | KVM 的 TSC 偏移/缩放 | guest 读时间，读到的是调整过的 |
| 本地定时器 | 虚拟 LAPIC timer | guest 的调度节拍 |

所以 guest 内核启动 2 个调度队列，以为自己在管 2 颗真 CPU；实际上它管的只是 KVM 里的两个 `kvm_vcpu` 结构。

## 第 5 章：回到"它是线程"——KVM 里的准确说法

准确表述是这三句：

1. **KVM 不创建线程，QEMU 创建。** KVM 只提供一个 vCPU fd；哪个线程来调 `KVM_RUN`，KVM 不关心。
2. **同一个 vCPU 同一时刻只能被一个线程跑。** 两个线程同时 `KVM_RUN` 同一个 vCPU 是非法的。
3. **调度归 Linux，不归 KVM。** vCPU 线程和普通线程一样参与 CFS 调度，会被抢占、迁移、换出。

这就解释了所有性能现象：

| 现象 | KVM 视角的解释 |
|---|---|
| 8 核开 16 vCPU 变慢 | 16 个线程抢 8 个核，KVM 无能为力 |
| guest 里 `st` 高 | vCPU 线程被调度走了，物理核在跑别人 |
| 一个 vCPU 卡住，整机像死机 | 持锁的 vCPU 线程被换出，其他 vCPU 在 KVM 里空转等锁 |
| vCPU 绑核能提速 | 线程不再迁移，缓存和 TLB 不失效 |
| 单 vCPU 沙箱起得快 | 只有一个线程要调度，没有 IPI 和核间同步 |

`st` 就是 **steal time**：本该给你、被宿主拿去的时间。

## 第 6 章：一张图

```
用户态（QEMU/Firecracker）              内核态（KVM）              硬件
┌────────────────────────┐
│ 线程 CPU0 ── KVM_RUN ───────────────→ vcpu_load → VM Entry ──→ 物理核 3
│   ↑                                     │                        │
│   └──────── kvm_run.exit_reason ←───────┘←────── VM Exit ←───────┘
│ 线程 CPU1 ── KVM_RUN ───────────────→ 同理 ─────────────────→ 物理核 7
└────────────────────────┘
        ↑
   Linux 调度器决定这两个线程何时上核、上哪个核
```

> [!tip] 一句话记住
> **vCPU 是 KVM 发的"座位号"，线程是宿主上真正坐人的"椅子"。**
> KVM 只管发号，Linux 只管排椅子。

## 第 7 章：落到沙箱上

Firecracker 就是把这套东西压到最小：一个进程、每个 vCPU 一个线程、只留 virtio 设备，没有 BIOS、没有显示。所以它启动一台 microVM 只要百毫秒级。具体链路见 [[指令派发完整链路（从零开始）]]。

对照着记：

- **vCPU 多** → 线程多 → 调度和 IPI 开销大 → 沙箱启动慢、抖动大；
- **设备少** → `KVM_EXIT_IO/MMIO` 少 → 进出次数少 → 跑得快；
- **内存靠 EPT** → 缺页在 KVM 内部补，不惊动 QEMU → 比设备访问便宜得多。
