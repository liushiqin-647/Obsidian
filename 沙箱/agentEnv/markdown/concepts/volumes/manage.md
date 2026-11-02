# 管理 Volume

本页介绍管理 Volume 的基本命令。

## 列出 Volume

`aenv volume list` 列出所有 Volume。

```bash
aenv volume list  # Alias: aenv volume ls
```

`--output` 接受 `table` 或 `json`。在交互式终端中默认为 `table`，当输出被管道传输或重定向时默认为 `json`。

## 检查 Volume

`aenv volume inspect <volume>` 以格式化 JSON 打印指定 Volume ID 或名称的完整记录。

Volume 状态决定其是否可被挂载：

| 状态 | 含义 |
| --- | --- |
| `ready` | Volume 可以挂载。 |
| `uploading` | 发布正在进行中；Volume 暂时不可用。 |
| `failed` | 发布失败；Volume 在恢复之前不可用。 |

## 删除 Volume

`aenv volume delete <volume>` 按 ID 或名称删除 Volume。

已挂载的 Volume 无法删除。请先删除其沙盒，再删除该 Volume。
