# 管理 Template

本页提供管理 Template 的基本命令。

## 列出 Template

```bash
aenv template list        # alias: aenv template ls
aenv template list --output json
```

显示所有 Template 及其 ID、名称、构建状态、CPU、内存、磁盘大小和最后更新时间戳。`--output` 接受 `table` 或 `json`。在交互式终端中默认为 `table`，当输出被管道传输或重定向时默认为 `json`。

## 列出 Template 构建

列出某个 Template 的完整构建历史：

```bash
curl -H 'X-API-Key: test-key' \
  http://127.0.0.1:8000/templates/<template-id>
```

## 检查 Template 别名

检查别名是否存在，并将其解析为 Template ID：

```bash
curl -H 'X-API-Key: test-key' \
  http://127.0.0.1:8000/templates/aliases/<alias>
```

## 删除 Template

```bash
aenv template delete <template-id-or-name>   # alias: aenv template rm
```

删除开始后，Template 会停止解析，其名称可以立即重用。如果删除中途失败，请按 ID 再次运行以移除剩余内容。
