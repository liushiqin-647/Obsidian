# Shell 补全

`aenv completion <shell>` 将 `aenv` CLI 的 shell 补全注册脚本打印到 stdout。支持的 shell：`bash`、`zsh` 和 `fish`。

该脚本注册一个补全函数，在补全时回调 `aenv` 二进制（`COMPLETE=<shell> aenv ...`），因此补全逻辑始终与已安装的 CLI 版本一致——升级 `aenv` 无需重新生成脚本。脚本按名称调用 `aenv`，因此该二进制必须位于你的 `PATH` 中。

## 生成并安装脚本

### Bash

```bash
mkdir -p ~/.local/share/bash-completion/completions
aenv completion bash > ~/.local/share/bash-completion/completions/aenv
```

Bash 补全文件由 [`bash-completion`](https://github.com/scop/bash-completion) 按需加载。这要求已安装 `bash-completion` 并在当前 shell 中初始化。

### Zsh

```zsh
mkdir -p ~/.local/share/zsh/site-functions
aenv completion zsh > ~/.local/share/zsh/site-functions/_aenv
```

Zsh 从 `fpath` 中列出的目录加载补全函数。`~/.local/share/zsh/site-functions` 并非在所有系统上都默认包含在 `fpath` 中。请在任何现有的 `compinit` 调用之前，将以下行添加到 `~/.zshrc`：

```zsh
fpath=(~/.local/share/zsh/site-functions $fpath)
autoload -Uz compinit
compinit
```

### Fish

```shell
mkdir -p ~/.config/fish/completions
aenv completion fish > ~/.config/fish/completions/aenv.fish
```

## 不安装直接激活

要在不保存生成文件的情况下测试当前 shell 会话的补全：

```bash
source <(aenv completion bash)         # Bash
eval "$(aenv completion zsh)"         # Zsh
aenv completion fish | source          # Fish
```

## 补全覆盖的范围

静态补全覆盖完整的 CLI 界面：

```bash
aenv <TAB>                       # top-level commands
aenv snapshot <TAB>              # nested subcommands (create, list, ...)
aenv list --output <TAB>         # enum values: table, json
aenv build ./<TAB>               # local path arguments
```

接受 Sandbox ID 的命令还会动态补全实时的 Sandbox ID，并按每个命令接受的状态进行过滤：

| 命令 | 补全的 Sandbox |
|---------|--------------------|
| `pause`、`exec`、`timeout`、`upload`、`download`、`snapshot create` | running |
| `resume` | paused |
| `connect`、`delete` | running 和 paused |

```bash
aenv resume <TAB>                # paused sandbox IDs
aenv exec <TAB>                  # running sandbox IDs
```

接受 Template 或 Snapshot 的命令会先补全它们的名称，对于没有名称的资源则以 ID 作为回退。Template 会按每个命令可操作的构建状态进行过滤：

```bash
aenv start <TAB>                 # ready templates and snapshots: names, then IDs
aenv template watch <TAB>        # templates whose build has not finished
aenv template delete <TAB>       # every template, whatever its build did
```

`aenv start --cold` 接受外部 OCI 镜像引用而非本地资源，因此它不提供 Template 或 Snapshot 候选项。

在 shell 支持的情况下，候选项会附带描述：Sandbox 的 Template 和状态，Template 的底层 ID 和构建状态，以及 Snapshot 的底层 ID。

动态查找是尽力而为的：它使用较短的超时（连接 500 ms、请求 1 s、每次补全请求总计 2 s），并在凭据、服务器或网络不可用时静默返回无候选项。在这种情况下，静态命令和标志补全仍可正常工作，且不会向你的命令行写入任何诊断输出。在规模大到触及这些限制的部署中，输入更长的前缀会缩小查找范围，并带回较短前缀不得不遗漏的候选项。
