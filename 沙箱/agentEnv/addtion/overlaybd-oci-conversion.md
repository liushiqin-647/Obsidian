# overlaybd 如何把 OCI 镜像变成"可按块拉取"的格式

> 本文补充说明 AgentENV / overlaybd 的"OCI 层转换 + 按块懒加载"的完整机制，
> 并详解其中两个基础概念 **zstd** 与 **LSMT**。
> 官方文档中相关内容分散在 `configuration/reference.md`、`internals/architecture.md`、
> `internals/p2p-design.md`、`internals/persistence-artifact-inventory.md`，
> 但缺少一条从"原始 OCI tar.gz"到"按块拉取"的完整链路，因此单独整理。

## 1. 问题：原始 OCI 镜像层不是"块可寻址"的

标准 OCI / Docker 镜像层是 **tar.gz**：

- tar 是**顺序流**；
- gzip 是**顺序压缩**，要读中间某块必须从头解压到那里。

因此无法"只拉取第 N 个块"。若直接把原始 `tar.gz` 放在 registry 上，overlaybd 无法高效按块懒加载。

## 2. 解决：转换成 overlaybd 格式

AgentENV 会把标准 OCI 层转换成 **overlaybd 层**（可由 `convert_standard_oci` 控制，默认开启）：

```
原始 OCI 层 (tar.gz)
      │  转换 (obdconv / overlaybd tools)
      ▼
overlaybd 层：zstd 压缩 + jump table(跳转索引) + LSMT 段映射索引
      │
      ├──► 推送到 OCI registry（overlaybd-native / referrers artifact）
      └──► 或存入本地 image cache，并可通过 P2P 分发
```

转换后的层具备两大关键结构：

1. **索引（LSMT 段映射）**：记录每个逻辑块在层文件中的**字节偏移与长度**。
2. **可随机访问的压缩（zstd + jump table）**：压缩流中每隔固定大小（代码中 `JUMP_TABLE_READ_CHUNK_SIZE = 1 MiB`）设一个"跳转入口"，可以从目标块对应的压缩起点**独立解压**，无需从头解压。

## 3. zstd：一种能"按块独立解压"的压缩算法

**是什么**：Zstandard（zstd），Facebook 开源的压缩算法/格式。特点：**压缩/解压都很快**，压缩比可调（level 越高压得越小但越慢），一般比 gzip **更快**、压缩比也更高。

**为什么 overlaybd 用它**：关键在于它支持"**可随机访问**"。

- **gzip**：一整条连续压缩流，想解压中间某段，必须**从头解到尾**（顺序压缩）；
- **zstd + jump table（跳转表）**：把数据切成一段段，每段记一个"跳转入口"，**可以只解压目标那一小段**，不用从头来。

代码里的体现（`storage/overlaybd/src/compression/zfile.rs`）：

```rust
const ZSTD_C_LEVEL: i32 = 3;                       // 压缩级别 3（速度/压缩比平衡）
const JUMP_TABLE_READ_CHUNK_SIZE: usize = 1 << 20; // 每 1 MiB 设一个跳转入口
```

**类比**：gzip 像一根从头到尾的拉链（要拉中间得从开头拉）；zstd + jump table 像一件每段都有独立拉链的衣服（想开哪段开哪段）。

## 4. LSMT：overlaybd 的分层镜像格式

**全称**：Log-Structured Merge Tree（日志结构合并树）。在这里，它是 **overlaybd 用来组织"多层镜像"的格式**（不是数据库里那个 KV 存储，但思想相似）。

拆开看三个关键词：

**① Log-Structured（日志结构 = 只追加，不覆盖）**
写操作**追加**到可写层末尾，不回头改已有数据（像写日志）。好处：底层只读层永不改动，方便共享/快照。

**② Merge / Stacking（分层堆叠）**
镜像由多层组成：

```
lower 层0（只读，最旧）
lower 层1（只读）
...
lower 层N（只读）
upper 层（可写，最新）
```

读的时候**自上而下**查：先在 upper 找，没有就往下一层找（每层都有索引，能快速定位）。

**③ Tree / Index（索引树）**
每层保存一份**段映射索引**（`DiskSegmentMapping`），记录"逻辑块 X → 本层 data 文件的物理偏移"。这就是"块可寻址"的目录。

代码/文档证据：

- `lsmt/format.rs`：层文件的 `HeaderTrailer` 魔数是 `LSMT\0\1\2`；
- `lsmt/index.rs`：段映射索引；
- `lsmt/file/`：`readonly.rs`（只读层）、`readwrite.rs`（可写层）、`stack.rs`（堆叠/合并）。

**"merge（合并）"体现在哪**：Pause/Snapshot 时，把当前可写层**封存（seal）**成一个新的只读层，接到层链上（相当于把草稿合并进正式档案）。多个 Sandbox 还能共享同一批只读层。

**类比**：LSMT 像一本**账本**——只往后追加记录（不涂改），多本账本按时间叠起来查（多层），到某个点把草稿账本装订成正式账本（封存合并）。

## 5. zstd 与 LSMT 如何配合

| | 负责什么 | 关系 |
|---|---|---|
| **LSMT** | 分层 + 索引 + 追加写 + 封存 | 决定"镜像怎么组织、块在哪" |
| **zstd** | 每个块数据的压缩 + 可随机解压 | 决定"块数据怎么压小、怎么按块解" |

```
LSMT 层 = [ index（块定位，LSMT 提供） ] + [ data（块内容，zstd 压缩，带 jump table） ]
```

- **LSMT 的 index**：告诉你块在哪个偏移；
- **zstd 的 jump table**：告诉你该偏移的压缩数据从哪独立解压；
- 两者合起来，才实现"只拉一块、只解一块"。

## 6. 按块拉取的三个支柱

```
VM 读某块
  → ① 查索引            得到该块在层文件中的字节范围 [offset, offset+len]
  → ② HTTP Range        仅请求这段字节（不改动整个层）
  → ③ 局部解压          用 jump table 只解压该块对应的压缩段
  → 缓存到本地（有界缓存）
```

| 支柱 | 作用 | 代码位置 |
|---|---|---|
| **段映射索引** | 定位块在层文件中的字节偏移 | `storage/overlaybd/src/lsmt/format.rs`、`lsmt/index.rs` |
| **可寻址压缩（zstd+jump table）** | 只解压目标块所在的压缩段 | `storage/overlaybd/src/compression/zfile.rs` |
| **HTTP Range 请求** | 只从 registry 拉取需要的字节范围 | `storage/overlaybd/src/backend/registryfs_v2.rs` |

`registryfs_v2.rs` 中的关键实现：

```rust
fn build_range_header(offset: u64, count: usize) -> Result<String> {
    let range_end = offset.checked_add(count as u64)...;
    Ok(format!("bytes={offset}-{range_end}"))
}
async fn send_range_request(...) {
    ... .header(RANGE, range_header) ...   // 只取需要的字节
}
```

`zfile.rs` 中的关键常量：

```rust
const ZSTD_C_LEVEL: i32 = 3;
const JUMP_TABLE_READ_CHUNK_SIZE: usize = 1 << 20; // 1 MiB 跳转粒度
```

## 7. "块管理"是谁提供的？

**不是 registry / 对象存储原生提供，而是 overlaybd 格式带来的：**

- registry / 对象存储本质上只是"支持按字节范围（HTTP Range）读取的 blob 存储"；
- 真正的"块可寻址"来自 **overlaybd 层格式**（LSMT 段映射索引 + zstd jump table 压缩）；
- 原始 `tar.gz` 必须先转换，才有块级懒加载能力（这正是 `oci_conversion_pipeline` 存在的意义）。

## 8. "远端"的构成

后续读取（未命中本地缓存）时，数据可能来自三处：

| 远端来源 | 存什么 | 备注 |
|---|---|---|
| **OCI registry** | 原始或转换后的镜像层 | 主要来源；**Turbo-OCI referrers 不会被选用**，AgentENV 只实现 native (`application/vnd.containerd.overlaybd.native.v1+json`) 读路径 |
| **对象存储（S3/OSS）或 POSIX 共享存储** | Snapshot 持久化产物（overlaybd 格式） | 通过 `on-demand-loading.md` 的 `repository_backend` 配置 |
| **集群其它节点（P2P）** | 已缓存的层 / 转换产物 | 仅加速；转换产物以 P2P key `oci-layer/v1/{context-hash}` 分发 |

## 9. 本地磁盘是"有界缓存"

overlaybd 的远端块缓存（`registryfs_v2`）与本地 image cache 都是**有容量上限的缓存**，冷数据按 **LRU** 淘汰：

```toml
[image.cache]
capacity_gb = 100                 # 容量预算

[image.cache.gc]
enabled = true
min_age_secs = 600                # LRU 下限：至少 10 分钟未用才可淘汰
high_watermark_ratio = 0.95       # 超过 95% 开始淘汰
low_watermark_ratio = 0.70        # 淘汰到 70% 为止

[image.cache.remote_blocks]
max_size_gb = 100                 # overlaybd 远端块缓存上限
open_evict_idle_secs = 600        # 运行中 Sandbox 的 idle 条目缓存块可被回收（条目保留，按需重填）
```

效果：单机磁盘只缓存"热块"，冷块淘汰后按需回源；因此聚合镜像/快照总量可以远超本地磁盘容量（几个数量级），且无需为每台机器预下载全部镜像（`without pre-warming every host`）。

## 10. 与官方文档的对应关系（已有零散记录）

| 主题 | 官方文档位置 |
|---|---|
| 转换开关 `convert_standard_oci` | `configuration/reference.md`（Image cache / OCI 转换段） |
| overlaybd-native media type、拒绝 Turbo-OCI | `configuration/reference.md` |
| zstd + jump tables | `internals/architecture.md`（Storage → overlaybd → Compression） |
| LSMT 分层格式、段映射 | `internals/architecture.md`（Storage → overlaybd） |
| 转换产物 P2P 分发（`oci-layer/v1/{context-hash}`） | `internals/p2p-design.md` |
| OCI 转换索引文件 | `internals/persistence-artifact-inventory.md` |
| 本地有界缓存、冷数据淘汰 | `getting-started/on-demand-loading.md`、`configuration/reference.md` |
| 远端块缓存 / 水位线配置 | `configuration/reference.md`（`[image.cache]`、`[image.cache.gc]`、`[image.cache.remote_blocks]`） |

## 11. 一句话总结

**原始 OCI 层（tar.gz）不可按块寻址；overlaybd 先把它转换成"LSMT 索引 + zstd 可寻址压缩块"的格式，再借助 HTTP Range 只拉取需要的块、用 jump table 局部解压，从而实现按块懒加载；本地只是有界 LRU 缓存，完整数据仍在远端（registry / 对象存储），P2P 仅作加速。**
