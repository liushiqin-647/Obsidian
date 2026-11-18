> [!abstract] 一句话
> AgentENV 快照 benchmark 的三张顺序图：暂停存快照、从快照恢复、并发恢复。

来源：`AgentENV/crates/benchmarks/benches/snapshot_benchmark.rs`

说明：从上到下是时间顺序。实线箭头是调用，虚线箭头是返回，黄色小框是同一角色内部的工作，左侧橙色竖括号内是打点范围。

## 一、暂停并保存快照（用例 1 / 2 / 3）

![暂停并保存快照](snapshot-benchmark-flows-1.svg)

打点范围：从 benchmark 发出 `pause()` 开始，到 AgentENV 返回快照配置结束。括号内包括刷持久卷缓存、暂停虚拟机、写运行状态、取脏页、打包内存层、封存盘层、写清单；建沙箱、停沙箱、删目录都在括号外。

## 二、从快照恢复（用例 4 / 5 / 6）

![从快照恢复](snapshot-benchmark-flows-2.svg)

打点范围：从 benchmark 发出 `resume_from_snapshot_config` 开始，到返回沙箱结束。括号内包括挂盘、起或复用进程、准备或复用内存设备、装载状态和内存、等 guest 初始化。

## 三、并发恢复（用例 6）

![并发恢复](snapshot-benchmark-flows-3.svg)

打点范围：橙色括号只包住每个线程的一次恢复调用，从发出到返回；停沙箱、等清理、进入下一轮都在括号外。成绩是各线程括号内耗时相加再除以 50。

相关：这张图里的停沙箱、起进程、等 guest 初始化，落在整套调用链的哪一步，见 [[指令派发完整链路（从零开始）]]。
