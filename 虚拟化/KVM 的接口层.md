# KVM 的接口层

> [!abstract] 一句话
> QEMU 和 KVM 之间没有函数调用，只有三个 fd 上的 ioctl、两块共享内存和一组 eventfd。

## 0. 为什么不是函数调用

QEMU 在用户态，KVM 在内核态，两边不能直接调函数。QEMU 能碰到的入口，只有打开 `/dev/kvm` 得到的一个 fd，以及由它派生出来的 vm fd 和 vcpu fd。

所以“QEMU 和 KVM 靠 ioctl 对接”这句话是有实体的：一次 `open`，一套 ioctl，几次 `mmap`。控制指令走 ioctl，数据不走。

## 1. 三层 fd

| fd | 从哪来 | 典型 ioctl | 管什么 |
|---|---|---|---|
| `/dev/kvm` | `open("/dev/kvm")` | `KVM_GET_API_VERSION`、`KVM_CREATE_VM` | 全局：版本、建虚拟机 |
| vm fd | `KVM_CREATE_VM` 的返回值 | `KVM_CREATE_VCPU`、`KVM_SET_USER_MEMORY_REGION` | 一台虚拟机：内存和 vCPU |
| vcpu fd | `KVM_CREATE_VCPU` 的返回值 | `KVM_RUN`、`KVM_GET_REGS`、`KVM_SET_LAPIC` | 一颗 vCPU：跑与停、寄存器、中断 |

每一层只认自己那一层的 ioctl。拿 vcpu fd 去发 `KVM_CREATE_VM` 会失败，反过来也一样。

## 2. 一次最短的启动

```c
int kvm = open("/dev/kvm", O_RDWR);
int vm  = ioctl(kvm, KVM_CREATE_VM, 0);

struct kvm_userspace_memory_region mem = {
    .slot            = 0,
    .guest_phys_addr = 0,
    .memory_size     = RAM_SIZE,
    .userspace_addr  = (uint64_t)qemu_mmap(RAM_SIZE),
};
ioctl(vm, KVM_SET_USER_MEMORY_REGION, &mem);

int vcpu = ioctl(vm, KVM_CREATE_VCPU, 0);
int run_size = ioctl(kvm, KVM_GET_VCPU_MMAP_SIZE, 0);
struct kvm_run *run = mmap(NULL, run_size, PROT_READ | PROT_WRITE,
                           MAP_SHARED, vcpu, 0);

for (;;) {
    ioctl(vcpu, KVM_RUN, 0);
    switch (run->exit_reason) {
    case KVM_EXIT_IO:   /* 端口 I/O */   break;
    case KVM_EXIT_MMIO: /* 设备寄存器 */ break;
    case KVM_EXIT_HLT:  /* guest 执行 hlt */ break;
    }
}
```

六步各管一件事：

1. 打开 `/dev/kvm`，顺便用 `KVM_GET_API_VERSION` 确认内核这边认不认这套接口。
2. 建一台虚拟机，拿到 vm fd。
3. 把 QEMU 自己 mmap 出来的那块内存登记进去，guest 从此有了“物理内存”。
4. 在虚拟机里建一颗 vCPU，拿到 vcpu fd。
5. 从 vcpu fd 上 mmap 出 `kvm_run`，退出原因以后就写在这里。
6. 循环调用 `KVM_RUN`，每次返回都读一次 `exit_reason`。

## 3. 控制面与数据面

ioctl 搬的是指令，不是数据。真正在动的数据有两条路。

- **共享内存**：guest 的物理内存本来就是 QEMU mmap 出来的，登记之后 KVM 直接读写同一批页，没有拷贝。`kvm_run` 也是 mmap 出来的，它是一块双方共用的状态区。
- **eventfd**：ioeventfd 把“guest 写了某个地址”直接接到 KVM，irqfd 把“该给 guest 发中断”接到 KVM。这两种通知都不经过 QEMU 用户态。

所以三样东西各管一段：ioctl 管配置和启动，共享内存管数据，eventfd 管异步通知。

## 4. 退出原因

`KVM_RUN` 返回时，原因写在 `run->exit_reason` 里。常见的有这么几种：

| 值 | 什么情况 |
|---|---|
| `KVM_EXIT_IO` | guest 做了端口 I/O，也就是 `in`/`out` |
| `KVM_EXIT_MMIO` | guest 访问了设备寄存器，需要用户态来模拟 |
| `KVM_EXIT_HLT` | guest 执行了 `hlt` |
| `KVM_EXIT_SHUTDOWN` | triple fault，虚拟机等于崩了 |
| `KVM_EXIT_INTR` | 有信号打到这个线程，先退出来处理 |

要看清楚一点：**绝大多数 Exit 在 KVM 里就处理完了**，比如 EPT 缺页，KVM 补完映射再 Entry 回去，QEMU 根本不知道发生过。只有需要用户态的那部分，才带着原因返回到 `KVM_RUN` 的下一条语句。

## 5. 别和别的 ioctl 混在一起

QEMU 同时还在对别的内核子系统发 ioctl。vhost 走 `/dev/vhost-net`，VFIO 直通走 `/dev/vfio/*`，串口和网络也可能直接开设备文件。

这些和 KVM 是并列的接口，不属于 KVM API。看到 QEMU 里一堆 `ioctl` 调用，先分清 fd 是从哪来的，才知道它在跟谁说话。

## 6. 和前面那篇怎么对上

[[CPU 工作模式]] 里讲的一次 I/O 往返，落到代码只有三行：

1. `KVM_RUN` 返回，`run->exit_reason == KVM_EXIT_MMIO`；
2. QEMU 读盘，把数据写进共享页；
3. 再调一次 `KVM_RUN`。

设备前后端怎么分工，见 [[virtio-blk 前后端]]。知道这三个 fd 各管什么之后，KVM 就不再是一个“黑盒内核模块”，而是一组有明确入口的系统调用。
