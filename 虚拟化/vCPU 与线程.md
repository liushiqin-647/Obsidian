> [!info] 先记住三句话
> 1. 虚拟机里看到的每一颗 CPU，叫 **vCPU**。
> 2. 在 KVM 里，一个 vCPU 对应宿主上的**一个线程**，这个线程循环执行 `ioctl(KVM_RUN)`。
> 3. KVM 本身不调度、不模拟指令：它只负责**把物理 CPU 借给 guest 跑**，跑不下去时把原因交回用户态。

## 第 0 章：先分清四个角色

| 名字 | 是什么 | 住在哪 |
|---|---|---|
| **物理 CPU** | 真芯片，比如 8 核 16 线程 | 硬件 |
| **线程** | Linux 调度的最小单位，一个核轮流跑很多线程 | 宿主内核 |
| **vCPU** | 虚拟机内部的一颗“CPU”，KVM 里的一个对象 | KVM |
| **KVM** | Linux 内核模块，通过 `/dev/kvm` 提供虚拟化接口 | 宿主内核 |

关系一句话：

> QEMU 建好一个线程，这个线程拿着 vCPU 的 fd 反复调 `KVM_RUN`；KVM 每次把物理核交给 guest，直到它需要宿主替它做点什么才交还。

## 第 1 章：一个 vCPU 在 KVM 里是怎么被造出来的

造一个 vCPU 对象只要几步 ioctl：打开 `/dev/kvm`，用 `KVM_CREATE_VM` 建一台虚拟机，用 `KVM_CREATE_VCPU(0)` 建 0 号 vCPU，再用 `KVM_SET_CPUID2` 告诉它是什么型号、`KVM_SET_REGS` 摆好开机寄存器，最后从 vcpu fd 上 `mmap` 出一块 `kvm_run`，用来收退出原因。`KVM_CREATE_VCPU` 的参数就是编号，`0, 1, 2...` 直接对应 guest 里的 CPU 序号。

这一段只造对象，不碰线程。整条链上没有任何一环会创建线程、或把 vCPU 绑到某个核上，造出来的东西就静静躺在 KVM 里；真正让它跑起来的是 QEMU 另外建的线程，那是下一章的事。三个 fd 各自认哪些 ioctl，见 [[KVM 的接口层]]。

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
 ├─ vcpu_load()        把 vCPU 绑到当前物理核：让 CPU 指向它的 VMCS，并装入一部分宿主侧状态
 ├─ 检查有没有待处理的中断、事件
 ├─ VM Entry           执行 VMLAUNCH / VMRESUME，CPU 进入 non-root 模式
 │     └─ guest 代码就在这里原生执行，一条条跑在物理核上
 └─ VM Exit            CPU 回到 root 模式，KVM 把原因写进 kvm_run
```

然后 `KVM_RUN` 返回用户态，QEMU 读 `run->exit_reason`，决定怎么办。

**关键**：guest 的普通指令不会被拦截，是原生执行的。KVM 只在“guest 不能自己决定”的时刻才拿回控制权。所以纯计算的负载在 guest 里几乎和物理机一样快，开销集中在 Exit 本身和 Exit 之后那段处理上。

## 第 3 章：退出原因——vCPU 停下来时，线程在等什么

`exit_reason` 是每次退出时 KVM 交给 QEMU 的第一张单据，QEMU 靠它决定下一步。其中和线程有关的是下面三种：

| `KVM_EXIT_*` | 什么情况 | 谁来处理 |
|---|---|---|
| `HLT` | guest 执行 `hlt`，没活干了 | KVM 让这个 vCPU 的线程睡，等中断再醒 |
| `IRQ_WINDOW_OPEN` | guest 关着中断，中断暂时投不进去 | KVM 记下这笔，等中断窗口打开再投 |
| `INTR` | 线程收到信号（比如被 kill） | QEMU 退出循环 |

最容易被误解的是 `HLT`：睡下去的是线程，不是物理核，核立刻被调度器拿去跑别人；中断一到，线程被唤醒，再调一次 `KVM_RUN` 进去。`INTR` 同理，信号打的是线程，vCPU 自己没有任何感觉。

其余几种退出和线程无关。`IO`、`MMIO` 是设备 I/O，`EPT_VIOLATION` 是给没映射的 GPA 补页，`SHUTDOWN` 是 guest 崩了之后由 QEMU 决定重启还是关机。它们讲的是 KVM 和 QEMU 谁管什么，展开见 [[CPU 工作模式]] 的 §6 和 [[KVM 的接口层]]。

## 第 4 章：guest 里的“2 颗 CPU”是怎么被编出来的

线程让它真的跑起来，但线程自己并不知道在演一颗 CPU：guest 看到 2 颗 CPU，宿主上其实就是 2 个普通线程。要让这个假象立住，还得这几样东西配合：

| 编造物 | 谁提供 | 作用 |
|---|---|---|
| CPU 数量与编号 | QEMU 写 ACPI 表 | guest 启动时枚举出 CPU0、CPU1 |
| CPU 型号、特性 | `KVM_SET_CPUID2` | guest 读到的是 KVM 支持的指令集 |
| 中断控制器 | KVM 内核内建的 LAPIC（`KVM_CREATE_IRQCHIP` 时一并建） | vCPU 之间互发 IPI、收时钟中断 |
| 时钟 | KVM 的 TSC 偏移/缩放 | guest 读时间，读到的是调整过的 |
| 本地定时器 | 虚拟 LAPIC timer | guest 的调度节拍 |

所以 guest 内核会给 2 颗 CPU 各准备一个调度队列，以为自己在管 2 颗真 CPU；实际上它管的只是 KVM 里的两个 `kvm_vcpu` 结构。

## 第 5 章：回到“它是线程”——KVM 里的准确说法

准确表述是这三句：

1. **KVM 不创建线程，QEMU 创建。** KVM 只提供一个 vCPU fd；哪个线程来调 `KVM_RUN`，KVM 不关心。
2. **同一个 vCPU 同一时刻只能被一个线程跑。** 两个线程同时 `KVM_RUN` 同一个 vCPU 是非法的。
3. **调度归 Linux，不归 KVM。** vCPU 线程和普通线程一样参与 CFS 调度，会被抢占、迁移、换出。

这就解释了所有性能现象：

| 现象 | KVM 视角的解释 |
|---|---|
| 8 核开 16 vCPU 变慢 | 16 个线程抢 8 个核，KVM 无能为力 |
| guest 里 `st` 高 | vCPU 线程被调度走了，物理核在跑别人 |
| 一个 vCPU 卡住，整机像死机 | 持锁的 vCPU 线程被换出，其他 vCPU 在 guest 里空转等锁 |
| vCPU 绑核能提速 | 线程不再迁移，缓存和 TLB 不用反复失效 |
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
> **vCPU 是 KVM 发的“座位号”，线程是宿主上真正坐人的“椅子”。**
> KVM 只管发号，Linux 只管排椅子。

## 第 7 章：落到沙箱上

Firecracker 就是把这套东西压到最小：一个进程、每个 vCPU 一个线程、设备只留 virtio 块/网卡和一个串口，没有 BIOS、没有显示。所以它启动一台 microVM 只要百毫秒级。具体链路见 [[指令派发完整链路（从零开始）]]。

对照着记：

- **vCPU 数量**：每多一颗就多一个线程，调度和 IPI 的开销跟着涨，沙箱启动更慢、抖动更大。
- **设备数量**：每省掉一个设备，`KVM_EXIT_IO`/`MMIO` 就少一类，进出次数随之下降。
- **内存**：靠 EPT 隔离，没映射的 GPA 在 KVM 内部就补上了，不惊动 QEMU，所以比设备访问便宜得多。
