# AgentENV 中宿主与 Firecracker / envd 的通信通道

> 本文澄清 AgentENV 里宿主（AgentENV 节点）与 Firecracker、以及 guest 内 envd 的通信方式。
> 关键结论：这是**两条独立的通道**——控制 Firecracker 走 **Unix socket**，与 envd 通信走 **TCP（HTTP + gRPC）**；
> **envd 不使用 vsock**。

## 1. 两条独立通道总览

| 通道 | 对象 | 传输方式 | 用途 |
|---|---|---|---|
| **通道一** | Firecracker 进程 | **Unix domain socket**（`--api-sock`） | 控制 VM 本身：建机器、挂盘、配置网络、快照等 |
| **通道二** | guest 内 envd | **TCP（HTTP + gRPC）** 到 `guest_ip:49983` | 执行命令、读写文件、健康上报 |

## 2. 通道一：Firecracker API（Unix socket）

Firecracker 启动时带 `--api-sock <path>`（如 `firecracker.sock`）。宿主通过 **`UnixConnector`**（`src/sandbox/firecracker/connector.rs`）建立 `UnixStream` 连接，在该 socket 上发送 **HTTP API** 请求：

```rust
// connector.rs
impl Service<Uri> for UnixConnector {
    fn call(&mut self, _req: Uri) -> Self::Future {
        let stream = UnixStream::connect(self.path.clone()).await?;
        Ok(TokioIo::new(stream))
    }
}
```

典型 API 调用（`instance.rs`）：

```
PUT /machine-config
PUT /boot-source
PUT /drives/{id}
PUT /network-interfaces/{id}
PUT /mmds/config, /mmds
PUT /cpu-config, /logger, /balloon
PUT /actions (InstanceStart)
创建/加载快照、GET /vm/dirty-memory-ranges ...
```

**特点**：只用于"控制 Firecracker 这个进程"，与 guest 应用无关，也不经过网络。

## 3. 通道二：envd（TCP HTTP + gRPC）

envd 是运行在 guest 内部的守护进程（控制面端口默认 **49983**，见 `cfg.rs`）。宿主通过 **guest 的网络 IP** 访问它，经过 **tap 网络**（Firecracker 的 virtio-net）：

```rust
// sandbox.rs:1833
let envd_base_url = format!("http://{}:{}", interaction_ip, common.control_plane_port);
```

- `interaction_ip`：网络槽位的 `host_interaction_ip`（宿主视角可达的 guest IP）
- `control_plane_port`：默认 `49983`

协议分工（`src/sandbox/envd.rs`）：

| 用途 | 协议 | 说明 |
|---|---|---|
| **初始化 / bootstrap** | HTTP（reqwest） | `InitPostRequest`，带 `access_token` 鉴权；使用 `ENVD_BOOTSTRAP_HTTP_CLIENT` |
| **进程执行**（如 `aenv exec`） | gRPC | `ProcessClient::connect(grpc_address)` |
| **文件操作** | gRPC | `FilesystemClient::connect(grpc_address)` |

其中 `grpc_address` 就是 `envd_base_url`（同一控制面端口，HTTP 与 gRPC 并存）。

## 4. 数据流

### 通道一：控制 Firecracker

```
AgentENV 节点
  → UnixConnector（UnixStream）
  → /path/to/firecracker.sock（Firecracker API socket）
  → Firecracker 进程（建机器/挂盘/快照）
```

### 通道二：与 envd 通信

```
AgentENV 节点（envd 客户端：HTTP init + gRPC process/filesystem）
  → TCP connect interaction_ip:49983（经 tap 网络）
  → Firecracker virtio-net（tap 设备）
  → guest 内 envd（监听 49983）
```

## 5. 常见误解：envd 不走 vsock

`vsock` 在 AgentENV 中只用于：

- `storage/uffd-core/`：**userfaultfd 内存恢复**的 Unix domain socket 通道（`uffd_uds_path`）
- `thirdparty/firecracker-client` 的 vsock 模型定义（未被 envd 使用）

**envd 通信完全不使用 vsock**（`src/sandbox/envd.rs`、`src/sandbox/firecracker/sandbox.rs` 中无 vsock）。envd 走的是 **TCP 网络**。

## 6. 代码位置

| 主题 | 位置 |
|---|---|
| Firecracker API socket 连接器 | `src/sandbox/firecracker/connector.rs`（`UnixConnector`） |
| Firecracker API 调用 | `src/sandbox/firecracker/instance.rs`（`set_machine_config`、`add_drive`、`create_snapshot`、`load_snapshot` 等） |
| envd base URL / 端口 | `src/sandbox/firecracker/sandbox.rs`（`init_envd_instance`）、`src/cfg.rs`（`control_plane_port`，默认 49983） |
| envd 客户端（HTTP + gRPC） | `src/sandbox/envd.rs`（`ProcessClient`、`FilesystemClient`、`InitPostRequest`） |
| vsock / uffd 通道 | `storage/uffd-core/src/handler.rs`（`uds_path`） |

## 7. 一句话总结

**宿主控制 Firecracker 走 Unix socket（`--api-sock`），访问 guest 内 envd 走 TCP（HTTP init + gRPC process/filesystem）到 `guest_ip:49983` 经 tap 网络；vsock 只用于 userfaultfd 内存恢复等场景，与 envd 无关。**
