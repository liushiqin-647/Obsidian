# 手动编译（单节点）

在单台 Linux 主机上直接从源码运行 AgentENV，适用于开发和测试。

如果你想跳过从源码构建，请参见 [快速开始](../getting-started/quickstart.md)。

## 前置条件

- Linux 内核 6.8+
- 用于执行 Firecracker microVM 的 `/dev/kvm` 访问权限
- Rust 工具链（stable）——通过 [rustup](https://rustup.rs) 安装
- `sudo` 访问权限

如果服务器不支持标准 KVM，请遵循 [PVM 部署](./pvm.md) 而非本指南。

## 克隆仓库

```bash
git clone https://github.com/kvcache-ai/AgentENV.git
cd AgentENV
```

## 构建

```bash
# Debug build
make

# Release build (recommended for production)
make release
```

## 启动服务器

启动服务器。首次正常启动时，它会在 `$AENV_HOME/secrets/api-key` 下生成一个 API key，并在后续启动中复用：

```bash
# Debug build
API_ADDR=0.0.0.0:8000 make start-server

# Release build
API_ADDR=0.0.0.0:8000 make start-server-release
```

服务器会在首次启动时自动下载运行时资产（Firecracker 二进制、内核、rootfs）。就绪后，它监听 `http://127.0.0.1:8000`。在发出认证请求之前，请读取生成的 key：

```bash
export AENV_API_KEY="$(cat "${AENV_HOME_PATH:-/var/lib/aenv}/secrets/api-key")"
```

## 验证

```bash
curl http://127.0.0.1:8000/health
curl -H "X-API-Key: ${AENV_API_KEY}" http://127.0.0.1:8000/sandboxes
```

HTTP 不会在传输过程中保护 key。对于远程客户端，请使用可信网络、VPN 或 TLS 终结反向代理。

## 配置

服务器默认读取 `config/default.toml`。可通过以下方式覆盖：

```bash
AENV_CONFIG_PATH=/path/to/config.toml make start-server
```

所有设置请参见 [运行时节点配置](../configuration/reference.md)。
