# 运行 Python 代码

本快速上手会将官方 Python 镜像导入为 AgentENV 模板，启动一个 sandbox，并在其中执行 Python 代码。Python 程序直接包含在命令中；无需示例项目或源文件。

开始前，请按 [Quick Start](../getting-started/quickstart.md) 所述部署 AgentENV 并配置 `aenv` CLI。

## 1. 创建 Python Template

导入官方 Python 3.12 镜像，并等待 Template 变为就绪状态：

```bash
aenv pull python:3.12-slim --name python
```

## 2. 启动 Sandbox

以分离模式启动 Template，并捕获生成的 sandbox ID：

```bash
SANDBOX_ID="$(aenv start python -d)"
```

## 3. 执行 Python

在 sandbox 内运行以下程序：

```bash
aenv exec "$SANDBOX_ID" python -c '
from statistics import mean

temperatures = [21.5, 23.0, 22.4, 24.1]
print(f"average temperature: {mean(temperatures):.2f} C")
'
```

预期输出：

```text
average temperature: 22.75 C
```

## 4. 删除 Sandbox 和 Template

```bash
aenv delete "$SANDBOX_ID"
aenv template delete python
```
