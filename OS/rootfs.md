# rootfs：根文件系统

> [!info] 一句话
> rootfs **不是格式，也不是一种新设备**，而是一个角色：**被挂载到 `/` 上的那个文件系统**。
> 磁盘回答“数据在哪”，文件系统回答“字节怎么解释”，rootfs 回答“从哪开始走路径”。

## 1. 三层区分

| 层 | 回答的问题 | 例子 |
|---|---|---|
| 块设备 | 数据存在哪 | `/dev/vda1` |
| 文件系统 | 字节怎么排布、怎么解释 | 分区上的 ext4 实例 |
| **rootfs** | **哪套文件系统当 `/`** | 一次挂载决策 |

“ext4”这个词日常指三样东西，别混：

| 说“ext4”时可能指 | 是什么 |
|---|---|
| **格式** | 排布规则：超级块放哪、inode 多长、目录项怎么编码 |
| **驱动** | 内核里解释这套规则的代码（`ext4.ko`） |
| **实例** | 某分区上按这套规则**实际写出来**的那堆数据 |

**挂载挂的是第三个（实例），不是格式。** 格式是图纸，驱动是施工队，实例是按图纸盖好的房子；挂载是把房子接到一条街上（`/`），不是把图纸挂上去。

## 2. 一块盘可以有多个文件系统

```
/dev/vda
├─ /dev/vda1  ext4    → 挂 /
├─ /dev/vda2  xfs     → 挂 /data
└─ /dev/vda3  swap    → 不挂，给内核当交换区
```

也可以整盘不分、只放一个 ext4；或用 LVM 切多个逻辑卷、用 loop 挂镜像文件里的文件系统。

所以“rootfs 是谁”要精确到**分区或逻辑卷上的文件系统**，不是整块盘。

## 3. 同一块盘，两种命运

```
mkfs.ext4 /dev/vda1，往里放 /bin /etc /lib
        │
        ├─ mount /dev/vda1 /mnt   → 只是 /mnt 下的一个文件系统
        └─ 内核 root=/dev/vda1    → 成为 rootfs，/etc 就是 /etc
```

**数据一个字节没变，变的只是它接在目录树的哪个位置。** 反过来，tmpfs、NFS 挂到 `/` 上也是 rootfs——所以 rootfs 与“是不是磁盘、是不是 ext4”无关。

## 4. 挂载做了什么

不是搬数据，是四件小事：

1. 打开设备，确认可读；
2. 读超级块、校验 magic（ext4 是 `0xEF53`）；
3. 载入根 inode，在内存里建一个 `vfsmount`（这个文件系统的活实例）；
4. 把根 dentry **接到挂载点的 dentry 上**。

三点补充：

- **挂载不复制数据**，只读几 KB 元数据，数据仍按需读取。
- **挂载状态在内核内存里**，重启即失，所以每次开机都要重挂。
- **不能省**：`/dev/vda1` 是设备节点，不是目录，没法 `cat /dev/vda1/etc/passwd`；`dd` 能读出字节，但读不出文件名。

## 5. ext4 是排布，不是外壳

```
offset 1024   超级块（magic 0xEF53）
之后          块组描述符、位图、inode 表、日志
再往后        数据块（由 extent 指向）
```

元数据和用户数据**混在同一堆块里**，不是贴在外面的一层包装。所以：

- 没有能解释这种排布的代码，字节仍在，但文件名、目录、权限都不存在；
- 能解释的不止内核驱动：`debugfs`、`libext2fs`、GRUB、FUSE 实现都行，**格式是契约，实现是签约方**；
- 换格式（如 xfs）是**整盘重排**，不是改后缀。

## 6. 启动时它怎么被用上

```
固件/引导 → 内核 → 先挂内部 rootfs(ramfs) → 解包 initramfs
                        ↓ 按 root= 找到真实设备
                   挂载真实根 → switch_root
                        ↓
                   /sbin/init → 用户态
```

- Linux 内核早期那个 ramfs 实例**名字就叫 `rootfs`**，这是术语的来源；最终挂在 `/` 上的才是一般说的 rootfs。
- **initramfs 不是 rootfs**，它是“找到并挂载 rootfs 的工具箱”；但 `root=/dev/ram` 时它自己就当根（microVM、救援系统常见）。
- 找不到就 panic：`VFS: Cannot open root device "vda1" or unknown-block(0,0)`。

## 7. rootfs 里有什么

```
/bin /sbin /lib /usr     可执行文件与库
/etc                     配置
/var /tmp                可变数据（常单独挂 tmpfs）
/proc /sys /dev          虚拟文件系统，运行时生成，不在镜像里
```

`/proc`、`/sys`、`/dev` 是**挂载出来的**，不属于 rootfs 的镜像内容。

## 8. 三种常见形态

| 形态 | 可写性 | 典型场景 |
|---|---|---|
| 只读镜像 + 可写上层 | 底层只读，改动落上层 | 容器 overlayfs；overlaybd / LSMT |
| 单块可写磁盘 | 直接读写 | 传统 VM、`qcow2` + backing file |
| 只读 + tmpfs | 根只读，`/tmp`、`/var` 用内存盘 | 不可变基础设施、沙箱 |

## 9. 与 bootfs 的区别

- **bootfs**：内核、initrd、固件（`vmlinuz`、`initramfs.img`、OVMF），启动完基本不用。
- **rootfs**：启动后一直在用的用户态文件系统。

microVM 里两者常拆开：kernel 由 `-kernel` 直接加载（不占盘），rootfs 才是那块 virtio-blk 盘。

## 10. 容器与 VM：同一个角色，不同载体

| | 载体 | 怎么变成 `/` |
|---|---|---|
| VM | ext4 磁盘镜像 | `root=` → mount → `switch_root` |
| 容器 | 目录（overlay 合并层） | `pivot_root` / mount namespace |

容器说 rootfs 指**镜像内容**，VM 说 rootfs 指**块设备上的文件系统**；两者可以互相转换（镜像 `mkfs` 进盘），但不在同一层面。

## 11. 相关

- [[virtio-blk 前后端]]：这块盘怎么被 guest 看见并读写
- [[storage-overlaybd-ublk]]：rootfs 盘背后的分层镜像与用户态块设备
