# BuildKit Template 构建内部机制

本页描述 `aenv build` 背后的内部 worker、缓存、发布和恢复生命周期。
面向用户的命令和配置，请参见 [创建 Template](../concepts/templates/creating.md#aenv-build)。

## Worker 准备与复用

服务器为每次 Dockerfile 构建供应并释放一个内部 worker。首次构建会在已配置
Snapshot repository 的私有命名空间中准备一个可复用的 builder Snapshot。
共享该 repository 的节点会恢复同一个 builder，并在启动 BuildKit 之前挂载一个新的
cache Volume（卷）。

同一节点上的并发首次请求会共享初始化过程。不同节点上的同时首次构建可能都会
准备一个 builder；后续构建会复用已发布的 Snapshot。builder 镜像、CPU、内存、
虚拟化模式和就绪设置共同标识该 Snapshot，因此更改这些输入会准备一个新的
Snapshot。worker 是内部的，不会出现在公开的 Sandbox 列表或端点中。

## 构建缓存生命周期

每次构建都会将最新的不可变 cache seed 克隆到自己的可写 Volume 中。
顺序构建会继承前一次构建的指令缓存和 `RUN --mount=type=cache` 数据。
并发构建可以分叉同一个 seed 而无需等待；它们的增量不会合并，
最后成功发布的缓存会成为下一个 seed。

镜像导入后，节点停止 BuildKit，通过常规 Volume 捕获路径检查点并发布其 cache
Volume，然后停止 VM，但不保存其内存、rootfs 或设备状态。关闭或缓存捕获失败时，
会保留之前的共享 seed。Volume 所有权会一直保持到 VM 停止。

缓存发布仅上传缺失的 layer。缓存发布失败不会导致原本成功的 Template 构建失败，
也不会替换之前的 seed。旧 cache Volume 在活跃子节点释放其 lease 后被移除。
缓存 head 更新和待处理退役会一起提交，而清理重试会独立处理退役。
BuildKit 的垃圾回收器管理缓存内容。

## 镜像发布

节点通过 SHA-256 digest 直接从 builder 读取完成的镜像，验证其字节，
并通过复用用于 registry 导入的、内容寻址的 OverlayBD 缓存，仅转换缺失的 layer。
镜像不会经过 CLI 传输。只有最终镜像配置会成为 Template 配置；
中间阶段和构建参数不会成为 Template 环境变量。

Registry 凭据来自本地 BuildKit 会话和常规 Docker 凭据配置。缓存共享保持在
Snapshot repository 现有的 API-key 信任边界内。

## 取消与恢复

取消和 deadline 会释放 worker 并丢弃其未完成的缓存子节点。如果客户端消失，
构建 deadline 最终会触发相同的清理。服务器重启会从持久化 journal 中恢复未完成的
构建和缓存预留。失败的清理会被保留并在服务器运行期间重试。活跃构建会阻止
Template 删除，直到取消或完成；不可读的 journal 条目不会阻止其他构建的恢复，
也不会阻止服务器启动。
