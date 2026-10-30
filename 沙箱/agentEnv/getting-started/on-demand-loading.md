# 从共享存储按需加载

AgentENV 通过 overlaybd 按需加载镜像。本地磁盘充当有界缓存，保留热数据并淘汰冷数据，因此节点无需预热每个镜像，也无需保留每个快照的完整副本。

## 前置条件

- 存储连接速度应尽可能快。至少使用 1 Gbps 网络；强烈建议 10 Gbps 或更快。

## 1. 打开配置文件

如果 AgentENV 以 systemd 服务运行，打开已安装的配置：

```bash
sudo vim /var/lib/aenv/config/config.toml  # Or use the path to your config file.
```

如果 AgentENV 在 Docker 中运行，下载并编辑默认配置：

```bash
curl -fsSL https://raw.githubusercontent.com/kvcache-ai/AgentENV/main/config/default.toml -o config.toml
vim config.toml
```

## 2. 配置共享存储

AgentENV 支持两种共享存储后端：POSIXFS 和 OSS。请选择以下选项之一。

### 选项 A：POSIXFS

启用 POSIXFS 后端，设置共享 Snapshot 位置，并增大远程块缓存：

```toml
[snapshot]
repository_backend = "posix_fs"

[backend.posix_fs]
snapshot_store = "/mnt/aenv-snapshots"

[image.cache.remote_blocks]
max_size_gb = 100
```

### 选项 B：OSS

启用 OSS 后端，配置 OSS 连接，并增大远程块缓存：

```toml
[snapshot]
repository_backend = "oss"

[backend.oss]
endpoint = "YOUR_ENDPOINT"
bucket = "YOUR_BUCKET"
region = "YOUR_REGION"
prefix = "YOUR_PREFIX"
cache_max_size_gb = 100
access_key_id = "YOUR_ACCESS_KEY_ID"
access_key_secret = "YOUR_ACCESS_KEY_SECRET"

[image.cache.remote_blocks]
max_size_gb = 100
```

如果你的服务提供商要求或倾向于使用虚拟主机式存储桶寻址——例如 [Tigris](https://www.tigrisdata.com/docs/) 或 Cloudflare R2 部署——还需设置 `addressing_style = "virtual"`；详情参见[运行时节点配置](../configuration/reference.md#backendoss)。

## 3. 应用配置

如果 AgentENV 以 systemd 服务运行，请重启它：

```bash
sudo systemctl restart aenv
```

如果 AgentENV 在 Docker 中运行，停止当前容器，并使用更新后的配置和共享 Snapshot 目录重新创建它：

```bash
# The /mnt/aenv-snapshots mount is required only when using POSIXFS.
docker stop CONTAINER_ID_OR_NAME
docker run --rm -it --name aenv-server \
  --device /dev/kvm --privileged -v /dev:/dev \
  -v "$PWD/config.toml:/workspace/config/default.toml:ro" \
  -v /mnt/aenv-snapshots:/mnt/aenv-snapshots \
  -p 8000:8000 \
  ghcr.io/kvcache-ai/aenv-server:latest
```

配置完成后，AgentENV 默认从共享存储加载模板和 Snapshot。
