参考：[openEuler kernel 技术分享 - 第1期 - kdump 基本原理、使用及案例介绍_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1M64y1Q7yp/?spm_id_from=333.337.search-card.all.click&vd_source=75270ed6f2004b3665c5eab2b8db3175)
# 介绍
kexec 快速重启机制，跳过bios（从正常内核启动崩溃内核）
kdump一种基于kexec的内核崩溃转储机制
# 原理
![[Pasted image 20260408234327.png]]
![[Pasted image 20260408234413.png]]
# 工具
## 用户态：kexec-tools
* **系统调用**：kexec_load、reboot
* **kexec**
	* 加载第二个内核 kexec -l
	* 启动到加载的内核 kexec -e
* **kdump**
	* 加载捕获内核 kexec -p
	* 系统crash，启动到捕获内核
## 内核态：kexec_load/reboot
* **kexec_load**

```c
SYSCALL_DEFINE4(kexec_load, unsigned long, entry, unsigned long, nr_segments,
		struct kexec_segment __user *, segments, unsigned long, flags)
```
* **reboot**
```c
reboot(LINUX_REBOOT_CMD_KEXEC)
```

# kdump执行流程
![[Pasted image 20260409000103.png]]
## 预留内存

* 分配的内存区域
	* /proc/iomem，dmesg
		* 物理内存布局
		* Crash kernel条目 `cat /proc/iomem | grep -i crash`
* 分配内存大小查看 `cat /sys/kernel/kexec_crash_size`

## 加载捕获内核

* 捕获内核是否加载成功 `cat /sys/kernel/kexec_crash_loaded`

## crash/oops
* \_\_show_regs, dump_backtrace 
* crash_kexec
	* crash_setup_regs：准备 panic kernel regs
	* crash_save_vmcoreinfo：更新vmcoreinfo
	* machine_crash_shutdown：保存panic cpu信息、清理中断
	* machine_kexec：

## /proc/vmcore
* ELF格式
* VMCOREINFO
	* 包含内核的各种信息
	* 用户态工具用来分析内存布局 makedumpfile
* ELFCOREHDR：描述panic内核的布局

## makedumpfile
* 减小转储的内存镜像的体积：页面过滤、页面压缩
* 使用 `makedumpfile -c -d 31 --message-level 31 /proc/vmcore vmcore`

# kdump服务：以openEuler为例
* Kdump 服务
	* 提供一些脚本
	* systemctl start/status/stop kdump
* 配置文件
	* /etc/kdump.conf
		```
		path /var/crash
		keep_old_dumps -1
		```
	*  /etc/sysconfig/kdump
		```
		KDUMP_COMMANDLINE_APPEND
		```

kdump使用案例
* 触发系统oops `echo c > /proc/sysrq-trigger`
* 解析 `crash vmlinux vmcore`
* 