# 管理 Snapshot

本页提供管理 Snapshot 的基本命令。

## 获取 Snapshot

通过 HTTP API 按 ID 或别名检索某个 Snapshot 的信息：

```bash
curl -H 'X-API-Key: test-key' \
  http://127.0.0.1:8000/snapshots/<snapshot-id-or-alias>
```

## 列出 Snapshot

```bash
aenv snapshot list
aenv snapshot list --sandbox-id <sandbox-id>
```

`aenv snapshot ls` 是 `aenv snapshot list` 的别名。

| 选项 | 默认值 | 描述 |
| --- | --- | --- |
| `--sandbox-id <sandbox-id>` | 所有 Snapshot | 仅显示从指定 Sandbox 创建的 Snapshot，包括该 Sandbox 被删除之后的 Snapshot。 |
| `--output <table\|json>` | 交互式终端中为表格；重定向时为 JSON | 选择输出格式。 |

## 删除 Snapshot

通过 HTTP API 按 ID 或别名删除 Sandbox 的 Snapshot：

```bash
curl -X DELETE -H 'X-API-Key: test-key' \
  http://127.0.0.1:8000/snapshots/<snapshot-id-or-alias>
```

成功返回 `204`。Snapshot 不存在、已删除的 Snapshot 以及 Template 构建返回 `404`；请通过 Template API 删除 Template 构建。

Snapshot 和 Template 共享同一目录。通过将要删除 Snapshot 的必需 ID 或别名传给 Template 删除命令来删除它：

```bash
aenv template delete <snapshot-id-or-name>
```
