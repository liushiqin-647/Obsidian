# 创建与挂载 Volume

## 创建 Volume

`aenv volume create` 可创建一个空 Volume、一个基于现有 Volume 的写时复制（copy-on-write）分叉，或一个从 OCI 镜像初始化的 Volume。

用法：

```bash
aenv volume create <name> [options]
```

| 参数或选项 | 默认值 | 说明 |
| --- | --- | --- |
| `<name>` | 必填 | 新 Volume 的唯一名称。 |
| `--size-mb <MiB>` | `65536` | Volume 大小，单位为 MiB。该值必须大于零。 |
| `--mode <exclusive\|ro>` | `exclusive` | 新 Volume 的访问模式。 |
| `--from-volume <volume>` | 无 | 从现有 Volume 的 ID 或名称创建一个写时复制分叉。与 `--image` 冲突。 |
| `--image <image>` | 无 | 从 OCI 镜像初始化 Volume。与 `--from-volume` 冲突。 |

### 访问模式

Volume 的模式在创建时选定，之后无法更改。

| 模式 | 可写 | 挂载并发 | 适用场景 |
| --- | --- | --- | --- |
| `exclusive` | 是 | 一个沙盒 | 每个 Sandbox 独立的工作区、缓存、数据库和可变状态 |
| `ro` | 否 | 多个 Sandbox | 共享数据集、模型、工具以及其他不可变输入 |

`exclusive` Volume 会被挂载它的 Sandbox 保留。在该保留被释放之前，其他 Sandbox 无法挂载或删除它。只读 Volume 可以被多个 Sandbox 同时挂载，但 guest 侧的写入会失败。

### 创建空 Volume

同时省略 `--from-volume` 和 `--image` 即可创建一个空 Volume：

```bash
aenv volume create workspace --size-mb 65536 --mode exclusive
```

此示例中的两个选项都使用了默认值，因此 `aenv volume create workspace` 会创建相同的 Volume。

### 分叉现有 Volume

使用 `--from-volume` 从现有 Volume 的 ID 或名称创建一个写时复制分叉：

```bash
aenv volume create job-data --mode exclusive --from-volume dataset-base
```

该分叉会捕获源 Volume 在创建时的状态。之后对分叉的修改不会影响源 Volume 或其他分叉。分叉的大小必须与源 Volume 相同；如果源 Volume 使用了自定义大小，请通过 `--size-mb` 传入该大小。`exclusive` 源 Volume 必须在创建分叉前卸载，而只读源 Volume 可以保持挂载。

> **推荐工作流：先分叉再使用**
>
> 将一个共享 Volume 视为不可变的只读基础，并为每个需要修改它的 Sandbox 创建一个 `exclusive` 分叉。这样每个 Sandbox 都能获得独立的可写 Volume，而无需急切地复制所有源数据。
>
> ```bash
> aenv volume create job-a-data --mode exclusive --from-volume dataset-base
> aenv volume create job-b-data --mode exclusive --from-volume dataset-base
> ```
>
> 这是一个显式的 Volume 操作。挂载 Volume 不会改变其模式，也不会自动对其分叉。

### 从 OCI 镜像创建 Volume

使用 `--image` 以 OCI 镜像的内容初始化 Volume：

```bash
aenv volume create models-base \
  --mode ro \
  --image registry.example.com/team/models:latest
```

对于标准 OCI 镜像，AgentENV 会在创建 Volume 时下载并转换其各层。对于 OverlayBD 原生镜像，AgentENV 会保留远程层引用，并在创建 Volume 期间不下载层内容。当挂载的 Volume 被读取时，块会按需从 OCI registry 获取，然后保留在本地远程块缓存中。

## 启动 Sandbox 时挂载 Volume

使用 `aenv start --volume MOUNT_PATH=VOLUME_ID_OR_NAME` 在创建 Sandbox 时挂载 Volume。该选项对热启动和冷启动均有效，且可以重复使用：

```bash
aenv start ubuntu \
  --volume /workspace=workspace \
  --volume /models=models-base
```

挂载路径必须是除 `/` 以外的绝对 guest 路径，且不能与其他挂载重叠。
