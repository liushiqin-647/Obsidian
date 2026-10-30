# 在 AgentENV Sandbox 中使用 Codex

运行 `aenv codex start ./my-project`，即可在项目的 sandbox 副本上打开原生的 Codex 终端。与 Codex 对话、跟踪其工具输出、用 `/diff` 审阅改动，并在之后恢复同一个会话。命令和文件编辑都在 AgentENV 中执行；你的原始项目保持不变。

## 一次性设置

你需要一个 AgentENV 部署，以及通过 `aenv auth` 配置好的 `aenv` CLI。部署和登录参见 [Quick Start](../getting-started/quickstart.md)。在 Linux 上运行 `aenv`，并安装 Git 和 Codex 0.155.1。它必须能够访问你的 AgentENV 部署和模型 API。如果通过 npm 安装 Codex，则需要 Node.js：

```bash
npm install -g @openai/codex@0.155.1
codex login
aenv codex setup
```

像往常一样在 Codex 中配置模型和认证。`aenv codex` 使用你的 `~/.codex` 目录（或 `CODEX_HOME`），包括 Codex 已配置的凭据存储。对于自定义的 Responses 兼容 API，请在 Codex 的 `config.toml` 中配置其 provider，并遵循 [Codex authentication](https://developers.openai.com/codex/auth/)。`aenv` 不会索取、复制或保存模型 API key。

设置只会在 `~/.config/aenv/codex.json` 中保存模板名称，并在内置的 `codex-executor` Template 不存在时构建它；后续会话会复用它。Template 文件已包含在 `aenv` 中，因此你无需检出仓库，也无需在本机安装 Python。首次构建使用 `aenv-buildctl`，它由 CLI 安装器随 `aenv` 一同安装。本地 Codex 和 Template 都使用 0.155.1 版本。

设置支持 `--template`，且不要求本地安装 Codex。`start` 和 `resume` 会检查本地 Codex 版本；如果安装在其他位置，请设置 `CODEX_BIN`。

Codex 0.155.1 中，`environments.toml` 的优先级高于 sandbox 连接。如果该文件存在于你的 Codex home 中，`aenv` 会在创建 sandbox 之前停止；请使用一个不含该文件的独立 `CODEX_HOME`，并在其中用 Codex 配置/登录。恢复会话时请使用相同的 Codex home。旧版 aenv 模型设置不再使用；重新运行 `setup` 会将其从 `codex.json` 中移除。对于由早期草案使用私有 `session/codex-home` 创建的会话，请用 `finish` 保存项目，再从该保存副本执行 `start`；这些会话的对话历史不会自动迁移到你的原生 Codex home。

## 在你的项目上工作

```bash
aenv codex start ~/projects/my-app
```

`aenv` 会创建一个 sandbox、上传你的项目，并打开 Codex 自带的终端界面。例如，让它修复一个解析器并运行测试，然后再发一条后续请求，要求增加另一个测试用例。

| 在 Codex 中 | 操作 |
| --- | --- |
| 输入任务或后续请求 | 在同一个 sandbox 中处理文件 |
| `/diff` | 查看 sandbox 的 Git diff |
| `/status` | 查看当前 Codex 会话 |
| `/new` | 在同一个 sandbox 中开始新对话 |
| `/quit` | 退出 Codex；`aenv` 会保存文件并暂停 sandbox |

输入编辑、工具输出、滚动和中断轮次均使用 Codex 的原生控制方式。`aenv` 会在 sandbox 内以自动命令批准模式启动 Codex。模型凭据始终保留在运行 `aenv` 的机器上。

即使在远程模式下，Codex 0.155.1 的 `@` 文件选择器仍会搜索 CLI 所在机器。要引用 sandbox 中的文件，请在消息中输入它们的相对路径。模型可以正常读取和编辑这些文件。

## 继续或结束

`aenv` 会在启动时以及退出时再次打印会话目录。用它来同时恢复 sandbox 和对话：

```bash
aenv codex resume /path/to/session
```

完成后，保存最终副本并删除 sandbox：

```bash
aenv codex finish /path/to/session
```

`start` 和 `resume` 需要交互式终端，并接受 `--model`。`start --template` 可选择另一个已准备好的 Template。

## 取回你的改动

默认情况下，会话位于 `~/.local/share/aenv/codex/sessions/` 下。`start --output` 会在你的项目之外选择另一个父目录。每次保存都会创建一个新的编号目录：

```text
session/
  latest.txt                 # Path to the latest successful save
  saves/0001/
    project/                 # Updated project files
    changes.patch            # Changes against your uploaded files
```

直接打开保存的项目，或者审阅并将 patch 应用到你的原始项目：

```bash
cd ~/projects/my-app
git apply --check /path/to/session/saves/0001/changes.patch
git apply /path/to/session/saves/0001/changes.patch
```

对于 Git 项目，上传内容包含工作区改动以及未被 Git 忽略的未跟踪文件。Git 历史、依赖/缓存目录、`.env` 文件以及常见的私钥文件会被排除。`.env.example` 和 `.env.sample` 会被包含。也支持普通目录。相对符号链接必须保持在项目内。保存内容包含受跟踪的文件以及未被 Git 忽略的新文件；请将交付物放在项目内且不位于被忽略的路径中。

在 `aenv` 运行期间，它每 30 秒续订一次 sandbox 的 5 分钟租约。退出会停止当前工作、保存项目副本并暂停 sandbox。如果保存失败，sandbox 会被保留，`aenv` 会打印用于恢复或找回它的命令。`finish` 会在删除前重试下载；`recover` 是它的别名。如果 `aenv` 被终止，租约到期会自动暂停 sandbox。可复用的 Template 仍可用于新会话。

一旦 `finish` 保存了最终副本，它会在删除 sandbox 前记录待删除状态。如果删除或其最终的本地状态更新失败，请重新运行 `finish`：它会保留已保存的副本并重试清理，而不会重新连接 sandbox。删除开始后，`resume` 将不可用。

如果无法确认服务已关闭，保存会停止，并且 sandbox 会被保留。这包括来自没有 supervisor 的早期草案的服务，或 supervisor 被强制杀死的服务。在重试 `finish` 之前，请先停止该 sandbox 中残留的服务进程。
