# virtio-blk：前端与后端

> [!info] 核心
> **前端** = guest 里的 `virtio_blk` 驱动，负责把 I/O 说清楚；
> **后端** = 宿主侧实现硬件寄存器语义、并真正把 I/O 做完的代码。
> 两端只靠三条通道耦合：virtqueue（数据）、feature/status（契约）、kick/中断（通知）。

## 1. 分层

```
guest 用户进程 read()
   ↓
guest 文件系统 / 块层
   ↓
virtio_blk 驱动（前端，guest 内核）      ← /dev/vda
   ↓  virtqueue（共享内存）+ kick/中断
═══════════ 虚拟机边界 ═══════════
   ↓
virtio-blk 设备模型（后端）              → 住在哪，见 §4
   ↓
镜像文件 / 块设备 / 远端存储
```

分工原则一句话：**前端只说清楚，后端只做到底**。前端不碰宿主文件，后端不解析文件系统。不是每个 VM 都走这条路——替代方案见 §5。

## 2. 一次读请求的全过程

guest 执行 `read(4KB @ 1MB)` 时，驱动在内存里摆出：

```
type   = IN       (u32)
sector = 2048     (u64；2048 × 512 = 1MB)
data   = 4KB 页
status = ?        (u8，后端填)
```

1. 文件系统算出块号，驱动分配 descriptor 链（header → 数据 → status）；
2. `avail->idx++`，写 doorbell → **VM Exit**（有 ioeventfd 则免退出）；
3. 后端取链、校验，`(sector, len)` 换成偏移 `2048×512`；
4. `preadv` 读盘（或查索引、解压、回源）；
5. 数据落进 guest 共享页，`status = OK`；
6. `used->idx++`，注入中断 → **VM Entry**；
7. guest 中断处理，唤醒等待者，`read()` 返回。

> 快递柜：前端放包裹、按铃；后端取件、送达、点“已送达”（status）、发短信（中断）。包裹一直在柜子里没动，动的只是“谁去执行”。

## 3. 假装硬件：发现期

§2 的前提是“设备已经被认领”。而 guest 驱动是**通用的**，它不认识 QEMU，只按 spec 逐个检查硬件该有的东西。后端要做的，就是把这些检查项实现成软件。

### 3.1 应答清单

| 层 | 检查项 | 后端应答 | 缺了会怎样 |
|---|---|---|---|
| 总线 | 0x00 Vendor ID | `0x1AF4` | 驱动不识别 |
| 总线 | 0x02 Device ID | `0x1042`（过渡 `0x1001`） | 不 match，`/dev/vda` 不出现 |
| 总线 | 0x0A Class Code | `0x0100` | 不被当作存储设备 |
| 总线 | 0x10 BAR0 | I/O 或 MMIO 窗口 | 映射失败 |
| 总线 | 0x34 Capability | common / notify / ISR / device | 找不到寄存器 |
| 设备 | `status = 0` | 复位、清队列 | 残留请求，状态错乱 |
| 设备 | `ACKNOWLEDGE / DRIVER` | 接受 | 初始化中止 |
| 设备 | 读 `device_feature` | FLUSH、DISCARD、MQ 位图 | 功能不可用 |
| 设备 | 写 `driver_feature` | 记录 guest 选择 | 协商不一致 |
| 设备 | `FEATURES_OK` | **回读时必须还在** | “no features”，probe 中止 |
| 设备 | 读 `device_cfg` | capacity、blk_size、seg_max | `capacity=0`，读不出分区表 |
| 设备 | 写 `queue_desc/avail/used` | 记下三个环的物理地址 | 队列无法建立 |
| 设备 | `queue_enable = 1` | 建立映射 | `virtqueue failed to enable` |
| 设备 | `DRIVER_OK` | 上线 | 不允许 I/O |

十步走完，`/dev/vda` 出现——此时一个字节的磁盘数据都还没读。内核还会往 BAR 写全 1 再读回探测大小，返回值不合法同样映射失败。

### 3.2 报的值从哪来

不是瞎编，是从真实后端翻译出来的：

| 字段 | 来源 |
|---|---|
| `capacity` | 镜像 `fstat` 字节数 ÷ 512 |
| `blk_size` | 文件/设备的逻辑块大小 |
| `seg_max` | 一次 `preadv` 的 iovec 上限 |
| `readonly` | `readonly=on` 或 `O_RDONLY` |
| `FLUSH` / `DISCARD` / `WRITE_ZEROES` | 底层是否真支持 `fsync`、`fallocate(PUNCH_HOLE)` |

广告了 `DISCARD` 却不处理，guest 一发 discard 就出错——**对 guest 说的每句话都得兑现**。

### 3.3 发现之后

| 阶段 | 后端要做 |
|---|---|
| 运行期 | 取环、执行、回写、注入中断；ISR **读后清零**；MSI-X 表项写入 |
| 变更期 | 扩容时改 capacity 并递增 `config_generation`，guest 据此重读 |
| 异常期 | `status=0` 复位、`NEEDS_RESET`/`FAILED` 上报、热拔、`unbind`/`bind` |

热插拔、`/sys/bus/pci/rescan`、kexec 都会**重走一遍发现**，状态机必须支持“复位 → 重新协商”。

## 4. 后端的实现选择

### 4.1 住哪

| 形态 | 谁干活 | 特点 |
|---|---|---|
| 进程内 | VMM 自己的设备线程 | 简单好调试；QEMU / StratoVirt 默认 |
| vhost（内核） | 宿主内核 | 少一次用户态往返；块设备的 vhost-blk 未进主线 |
| vhost-user | SPDK 等独立进程 | 隔离、可升级、可轮询；需共享内存 + Unix socket |

> [!warning] 两级前后端
> vhost-user 下，QEMU 对 guest 是**后端**，对 SPDK 又是**前端**（建环、传 fd、转 kick）。看到 “QEMU is the frontend of vhost-user-blk” 不矛盾，那是第二层。

### 4.2 快在哪

| 手段 | 作用 |
|---|---|
| ioeventfd | guest 写 doorbell 不退出用户态，KVM 直接叫醒后端线程 |
| irqfd | 写 eventfd 即注入中断，不必 `ioctl(KVM_INTERRUPT)` |
| iothread / dataplane | 队列处理与 I/O 跑专用线程，不占主循环 |
| 多队列 | 每 vCPU 一队列，队列绑线程，少跨核争锁 |
| O_DIRECT | 绕过宿主 page cache，避免双份缓存 |
| io_uring / polling | 批量提交；或忙轮询换最低延迟 |

### 4.3 两个现成实现

| 维度 | QEMU | StratoVirt |
|---|---|---|
| 语言 / 定位 | C，全功能 VMM | Rust，轻量微虚机 |
| 镜像格式 | raw、qcow2、rbd、nbd… | raw、qcow2（microvm 仅 raw） |
| O_DIRECT 默认 | 关（writeback） | **开**（`direct` 默认 true） |
| 并发 | iothread + vq 映射 | iothread + `num-queues`（≤32） |
| vhost-user-blk | 支持 | 支持，要求 `mem-share=on` + 大页 |

## 5. 边界：并非每个 VM 都有

三个条件同时成立才有：**VMM 暴露设备、guest 有驱动、传输层对得上。**

| 替代方案 | 驱动 | 场景 |
|---|---|---|
| IDE / SATA / AHCI | `ata_piix`、`ahci` | 兼容性优先、装系统 |
| virtio-scsi | `virtio_scsi` | 多盘、热插拔、SCSI 语义 |
| 模拟 NVMe | `nvme` | 要 NVMe 语义但不想直通 |
| VFIO 直通 | 真实盘驱动 | 性能极限 |
| Xen / Hyper-V / VMware | `xen-blkfront`、`storvsc`、`pvscsi` | 各自平台 |
| 无盘 | 无 | netboot、NFS root、只挂 virtiofs |

guest 侧三个坑：驱动没编译进去（`CONFIG_VIRTIO_BLK`）、模块没加载（没有设备时它本来就不加载）、Windows 缺 `viostor`（virtio-win）。

Firecracker 这类只实现 virtio 的 VMM 里，virtio-blk 才是“标配”——**局部经验，不是普适结论**。

## 6. 常见误解

- **后端 = 物理磁盘？** 不是。磁盘只是 `pread` 的对象；后端是“virtio 请求 → 宿主 syscall”的翻译层。
- **后端 = 宿主磁盘驱动？** 不是。后端在用户态模拟一块卡，之后才轮到 ext4/NVMe 碰真盘，中间还隔着 page cache（除非 O_DIRECT）。
- **后端 = 整个 VMM？** 不是。它是 VMM 里负责**这一块设备**的那部分代码。
- **后端一定独立进程？** 不是。§4.1 的三种住址只是位置，角色不变。
- **`virtio-blk` 和 `vhost-user-blk` 是两种设备？** 不是。对 guest 是同一个驱动、同一套 feature，换的只是宿主侧谁干活。
- **`virtio-scsi` 是更好的 blk？** 不是。多一层 SCSI 解析，换多盘、热插拔、`rescan`；单盘路径更长。

## 7. 实例（不属于概念本身）

```
Firecracker --virtio-blk--> 内核 ublk --> ublk daemon --> overlaybd --> OCI/OSS
```

后端被外包到用户态守护进程，交接协议用 ublk 而非 vhost-user。详见 [[storage-overlaybd-ublk]]、[[architecture-and-lifecycle]]。
