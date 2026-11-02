# P2P Artifact 传输

AgentENV 在 `src/p2p/` 中有一个跨项目的 P2P artifact 传输层。它让 runtime 模块能够从 peer 节点发布、发现和获取已验证的文件，而不依赖具体的传输实现。

第一个具体后端嵌入在 AgentENV 服务器进程中，使用 `iroh` 加 `iroh-blobs`。公共 API 保持在 trait 和可序列化类型之后，以便将来可以通过配置选择其他后端。

## 目标

- 为多个 AgentENV 模块提供一种可复用的节点到节点 artifact 传输。
- 使模块代码独立于具体后端。
- 使用 Scheduler 进行端点发现，并使用轻量级内存 artifact 到节点索引，而非 artifact 元数据存储或数据代理。
- 让传输后端使用其原生内容寻址或完整性检查，同时保持模块代码后端中立。
- 使 disabled 模式对单节点部署保持廉价且安全。

## 模块布局

| 路径 | 职责 |
|------|----------------|
| `src/p2p/mod.rs` | 公共导出和 `transport_from_config` 工厂 |
| `src/p2p/config.rs` | 解析后的 P2P 配置和传输类型选择 |
| `src/p2p/types.rs` | 与传输无关的 artifact、endpoint、peer 和 publish 类型 |
| `src/p2p/transport.rs` | `P2pTransport` trait 和 disabled 实现 |
| `src/p2p/discovery/` | 通过 Scheduler/static/no-op 实现进行 peer 发现和 artifact 索引提示 |
| `src/p2p/iroh/` | 嵌入式 `iroh` + `iroh-blobs` 后端和目录协议 |

## 公共 API

使用方使用 `Arc<dyn P2pTransport>`：

```rust
#[async_trait]
pub trait P2pTransport: Send + Sync {
    async fn lookup(&self, key: &P2pArtifactKey) -> Result<Option<P2pArtifactDescriptor>>;
    async fn lookup_with_hints(
        &self,
        key: &P2pArtifactKey,
        hints: &[P2pArtifactProviderHint],
    ) -> Result<Option<P2pArtifactDescriptor>>;
    async fn fetch(&self, descriptor: &P2pArtifactDescriptor, destination: &Path) -> Result<u64>;
    async fn fetch_bytes(&self, descriptor: &P2pArtifactDescriptor) -> Result<Bytes>;
    async fn fetch_byte_range(
        &self,
        descriptor: &P2pArtifactDescriptor,
        offset: u64,
        len: usize,
    ) -> Result<Bytes>;
    async fn publish(&self, request: &P2pPublishRequest) -> Result<()>;
    async fn unpublish(&self, key: &P2pArtifactKey) -> Result<bool>;
    fn local_endpoint(&self) -> Option<P2pEndpoint>;
    async fn shutdown(&self) -> Result<()>;
}
```

`P2pArtifactKey` 是由使用模块选择的稳定字符串。一个 key 代表一个逻辑 artifact。调用方应将完整的缓存上下文编码到该 key 中，并使用描述符元数据进行模块特定的验证。

`P2pArtifactDescriptor` 包含：

- `key`：稳定的查找 key。
- `providers`：可提供该 artifact 的节点，表示为 `P2pPeer` 值。
- `backend_locator`：可选的传输特定字符串，仅由匹配的后端使用。对于 iroh，这是 `iroh-blobs` 哈希。
- `metadata`：由调用方用于解释文件的模块定义 JSON。

`P2pPublishRequest` 包装稳定 key、本地源路径或内存字节、可选元数据以及 `P2pPublishMode`。`P2pPublishRequest::file(key, source)` 将元数据默认为 `null`，发布模式默认为 `Copy`；`P2pPublishRequest::bytes(key, bytes)` 发布已缓冲的字节。`Copy` 允许传输将字节复制到自己的存储中；`Reference` 允许后端在支持时保留或索引现有文件。

## Disabled 传输

当 `[p2p].enabled = false` 或 `transport = "disabled"` 时，选择 `DisabledP2pTransport`。

- `lookup` 返回 `Ok(None)`。
- `publish` 作为空操作成功。
- `unpublish` 作为空操作成功并返回 `Ok(false)`。
- `fetch` 返回 `TransportDisabled`。
- `local_endpoint` 返回 `None`。

这使使用方可以无条件调用 P2P 层，同时保持默认部署不变。

## Iroh 后端

当 `[p2p].enabled = true` 且 `[p2p].transport = "iroh"` 时，选择 `IrohBlobsP2pTransport`。

启动时它会：

1. 创建已配置的 blob 存储目录。
2. 打开一个带门控后台 GC 的 `iroh_blobs::store::fs::FsStore`。
3. 在 `<p2p.store_dir>/iroh/catalog.db` 打开一个由 RocksDB 支撑的本地目录。
4. 绑定一个 `iroh` 端点，可选使用 `[p2p].listen_addr`。
5. 启动一个同时服务 `iroh-blobs` 数据和 AgentENV 目录协议的路由器。
6. 通过 `local_endpoint()` 发布本地端点，以便可观测性心跳可以通告它。

后端使用一个本地目录：

- Published catalog：artifact key 到该节点可提供的描述符。内存 map 在启动时从 RocksDB 加载；每个 RocksDB 值是一个紧凑的 JSON 描述符。

查找首先检查本地已发布目录，然后向 Scheduler 询问在该 artifact key 下索引的节点，最后按提示/发现顺序通过 AgentENV 目录 ALPN 回退到已发现的 peer。Scheduler 索引的候选和回退候选仍会通过查询每个节点的目录进行验证；Scheduler 从不存储 locator 或元数据。描述符在其后端 locator 中携带 iroh blob 哈希，因此 `fetch` 不依赖先前的进程内查找状态。

Fetch 将描述符的后端 locator 解析为 iroh blob 哈希，将所有描述符 providers 传递给 `iroh-blobs`，将该 blob 下载到本地 `FsStore`，并导出到请求的目标位置。Iroh 的内容寻址 blob 传输会根据该哈希验证字节。如果任何描述符 provider 是本地节点，fetch 会直接从本地存储导出。

远程 fetch 将 blob 下载到本地存储后，iroh 后端会尽力通告本地副本。它应用与 publish 相同的确定性保留标签，构建一个以该节点为唯一本地目录 provider 的描述符，将该描述符 upsert 到本地目录，并在 Scheduler 中记录该 key。这使得获取节点成为后续 peer 的提供方，并让 artifact 在集群中扩散。通告或 Scheduler 索引失败会被记录，但不会导致 `fetch` 失败；只有下载和目标导出是必需 fetch 结果的一部分。

Publish 将本地文件导入 `FsStore`，应用从 artifact key 派生的确定性命名标签（`agentenv:p2p:v1:{sha256(key)}`），构建一个以该节点为唯一本地目录 provider、本地端点、iroh blob 哈希 locator 和请求元数据的描述符，将该描述符 upsert 到本地目录，并尽力在 Scheduler 中记录该 key。

Unpublish 从本地目录移除该 key，删除确定性命名标签，在 Scheduler 中忘记该 key，持久化目录，并返回是否移除了本地发布。删除标签会立即停止未来的目录查找，但不会同步删除 blob 字节。iroh 存储通过其 GC 回收未打标签的 blob。

GC 被门控，以避免在没有已知删除工作时进行周期性全存储扫描：

- 传输启动时有一个待处理 GC pass，因此重启可以清理那些在先前进程退出前标签已被移除的 blob。
- `unpublish` 删除保留标签后将 GC 标记为待处理。
- 每个 GC 间隔会唤醒 iroh-blobs GC 任务，但已配置的 `add_protected` 回调会在未设置待处理 GC 时中止 mark/sweep pass。
- 当设置了待处理 GC 时，会运行一次完整 mark/sweep pass。它保留已打标签和临时打标签的 blob，并删除不再受保护的 blob。

## Scheduler 发现与 Artifact 索引

Scheduler 仅是节点端点和 artifact 到节点提示的目录。

1. AgentENV 在 `src/bin/server.rs` 中启动已配置的 P2P 传输。
2. 如果传输暴露 `local_endpoint()`，`ObservabilityReporter` 会将其包含在心跳请求中。
3. Scheduler 将端点与已观测节点记录一起存储。
4. `SchedulerPeerDiscovery` 周期性调用 `ListP2pPeers(cluster_id, backend, exclude_node_id)`。
5. Scheduler 返回所请求后端中具有非空端点的就绪节点。
6. 传输直接拨号 peer 进行目录查找和字节传输。

为了加速 artifact 查找，iroh 后端还会调用：

- 在 publish 之后以及成功的远程 fetch 通告之后调用 `RecordP2pArtifact(cluster_id, backend, key, node_id)`。
- 在本地 unpublish 之后调用 `ForgetP2pArtifact(cluster_id, backend, key, node_id)`。
- 在广泛 peer 轮询之前调用 `LookupP2pArtifact(cluster_id, backend, key, exclude_node_id)`。

Scheduler 在内存中保留此 artifact 索引，仅存储 key 到节点的映射。查找响应会通过与 `ListP2pPeers` 相同的就绪节点和后端端点检查进行过滤。节点注销会移除该节点的所有 artifact 映射。Scheduler 不持久化 artifact 目录，不检查 artifact 元数据或 locator，也不代理字节。

## Snapshot 与 Overlaybd 使用方

`src/snapshot/p2p.rs` 是 Snapshot 存储集成层。它有意保持很小：Snapshot manager 在已配置 repository 提交成功后发布 artifact，而 runtime resolver 将 P2P 视为可选的加速路径，而非已提交真相。

Snapshot 发布会通告：

- `snapshot/v1/artifacts/{snapshot_id}/{relative_path}` 下的固定 Firecracker artifact：
  - `vm_state.bin` 从其本地文件路径发布。
  - `firecracker-manifest.json` 从已提交 manifest 序列化并作为字节发布。
- Snapshot 的 rootfs、memory 和 attached-drive 镜像配置引用的 Overlaybd layer。这些不会在 Snapshot 特定 key 下发布。它们复用由 `src/overlaybd/p2p/artifact.rs` 拥有的 overlaybd layer artifact 协议，key 形如 `overlaybd-layer/v1/sha256:<digest>`，并使用 overlaybd HTTP facade 理解的 `LayerMetadata`。

OSS repository 返回已提交记录以及准备好的本地 layer 上传。Manager 发布其现有 digest/size 描述符，而无需再次计算 SHA256，并保持压缩或 dense 导出的临时文件存活，直到提交后的 P2P 复制完成。这些本地路径永远不会进入持久化记录。来自 runtime 镜像配置的 UUID 发布仍然可用；重复的 digest key 只发布一次。

来自 runtime 镜像配置的、以 digest 为键的发布仍会针对已提交记录进行保护：只有当某个 digest 是记录为同一 subject（memory、rootfs 或匹配的 attached drive）引用的 digest 时，本地 layer 才会在 `overlaybd-layer/v1/sha256:<digest>` 下被通告。当 `[snapshot.publish_compression]` 将原始本地 layer 重新容器化为 ZFile 时，准备好的上传会提供压缩字节及其描述符；原始 raw digest 会被跳过。ZFile layer 不携带 LSMT uuid（已提交记录中 `uuid = None`）。发布这些 digest key 不会增加仅 digest 的 OSS 读取路由；该使用方路径是另一项更改。

这种拆分很重要。Snapshot 固定 artifact 限定于一个 Snapshot ID，并且仅由 Snapshot runtime resolver 使用。Overlaybd commit layer 是内容寻址的，可能被任何 overlaybd runtime 路径使用，包括通过 `/p2p-http/{origin}` 的前台范围读取。不要为 overlaybd layer 添加第二种 Snapshot 特定的 key 格式；这样做会发布 overlaybd 无法发现的字节。

OSS Snapshot 解析首先通过 P2P 使用固定 artifact：

- `vm_state.bin` 从 P2P 获取到节点本地 `LocalArtifactCache`，在未命中或获取失败时回退到 OSS。
- `firecracker-manifest.json` 通过 `fetch_bytes` 从 P2P 获取，在内存中解析，并在未命中、获取失败或解析失败时回退到 OSS。

POSIX Snapshot 解析不使用 P2P。POSIX repository 已经是一个可直接访问的已提交 artifact 存储；如果缺少必需文件，resolver 会返回 `ArtifactNotFound`，而不是从 peer 修复 repository。Runtime 块读取的 Overlaybd layer 加速仍由 overlaybd P2P facade 负责，而非 POSIX Snapshot resolver。

Snapshot P2P 发布是尽力而为的。P2P 发布失败会记录警告，且不会回滚已提交的 Snapshot 记录。这在使新发布的 artifact 在较慢的后端下载或上传成为瓶颈之前对 peer 可见的同时，保持了 repository 作为真相来源。

## 完整性与所有权

P2P 传输不定义 artifact 的含义。每个使用模块拥有：

- 构造稳定 key。
- 将元数据写入发布请求。
- 在信任获取的 artifact 之前验证元数据。
- 决定是否以及何时发布本地 artifact。

这一边界使传输保持可复用，并避免将模块特定的缓存语义固化到 `src/p2p` 中。

镜像 resolver 对标准 OCI 转换 artifact 使用相同的边界。转换后的 OverlayBD commit 在节点本地镜像缓存中持久化之后，镜像模块会在 `oci-layer/v1/{context-hash}` 下发布它。哈希从 `LayerConversionKey` 序列化；描述符元数据仅包含协议版本、输出 commit digest 和大小。使用方仍会从 registry 解析源 OCI manifest/config，然后在回退到本地 OCI 下载和转换之前，在 P2P 中查找每个转换上下文。
获取的字节在本地索引之前会被验证为已封存的 OverlayBD layer。P2P 发布和查找失败属于加速未命中，而成功的发布会记录在拥有该 artifact 的 image-cache hard commit 上，以便 GC 在删除文件之前取消发布它们。
