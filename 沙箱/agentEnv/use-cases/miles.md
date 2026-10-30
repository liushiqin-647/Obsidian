# 使用 Miles 训练 Terminal-Bench-2

AgentENV 与 [Miles](https://github.com/radixark/miles) 集成，作为面向智能体强化学习的自托管 sandbox 后端。Miles 是一个用于大规模模型后训练的高性能强化学习框架。

本示例遵循 Miles 的 [OpenEnv](https://github.com/huggingface/OpenEnv) 配方，在 [Terminal-Bench-2](https://github.com/laude-institute/terminal-bench-2) 上使用 GRPO 训练 [GLM-4.7-Flash](https://huggingface.co/zai-org/GLM-4.7-Flash)。OpenEnv 提供该配方所使用的任务交互与评估接口，而 AgentENV 为每个 episode 运行隔离的 sandbox。Terminal-Bench-2 是一套基于终端的软件任务基准套件，而非传统的仅提示词数据集：每个任务都包含一个环境、一条指令以及用于判定智能体是否完成任务的测试。

## 1. 部署 AgentENV

首先按照 [Quick Start](../getting-started/quickstart.md) 部署 AgentENV 服务器并完成认证。

使用 `docker run` 启动 AgentENV 时，发布 8000 和 80 端口，并将 `AENV_SANDBOX_PROXY_DOMAINS` 设置为 AgentENV 主机的 `sslip.io` 域名：

```bash
docker run -d --name aenv --privileged -v /dev:/dev \
  -p 8000:8000 -p 80:8000 \
  -e AENV_SANDBOX_PROXY_DOMAINS=<ip-with-dashes>.sslip.io \
  ghcr.io/kvcache-ai/aenv-server:latest
```

将 `<ip-with-dashes>` 替换为 AgentENV 主机的 IP，并将每个点替换为短横线。

## 2. 将 Miles 指向 AgentENV

```bash
pip install e2b

export E2B_API_URL=http://<server>:8000
export E2B_SANDBOX_URL=http://<server>:8000

# Export your api key fetched in the last step.
export E2B_API_KEY=<your-api-key>

# Per-sandbox URLs use HTTPS by default. This deployment uses plain HTTP.
export OPENENV_E2B_URL_SCHEME=http
```

API key 用于认证请求，但不会加密请求；请将此纯 HTTP 配置保持在可信网络中。

## 3. 准备 Miles、OpenEnv 和 Terminal-Bench-2

按照上游配方的要求，在现有的 Miles 训练环境中运行本教程。下面克隆 Miles 只会提供配方脚本；它不会安装 CUDA、模型服务或训练栈。

```bash
git clone https://github.com/radixark/miles.git
git clone https://github.com/huggingface/OpenEnv.git
git clone --depth 1 https://github.com/laude-institute/terminal-bench-2.git

pip install -e ./OpenEnv/envs/tbench2_env

python ./miles/examples/experimental/openenv/make_tbench2_data.py \
  --tasks_dir ./terminal-bench-2 \
  --output /root/tbench2_train.jsonl
```

为 `make_tbench2_data.py` 添加 `--n 8` 可获得一个小的冒烟测试子集。

## 4. 训练

```bash
export OPENENV_TB2_TASKS_DIR="$(realpath ./terminal-bench-2)"
OPENENV_SANDBOX_BACKEND=e2b \
  python ./miles/examples/experimental/openenv/run-openenv-tbench2.py
```

启动器会为每个 episode 创建一个 AgentENV microVM，并将任务的规范测试结果作为 GRPO reward 返回。`e2b` 使用指向 AgentENV 端点的 E2B 兼容后端。

有关完整的训练配置、provider 选项和运维说明，请参见上游的 [Miles OpenEnv Terminal-Bench-2 recipe](https://github.com/radixark/miles/blob/main/examples/experimental/openenv/README.md)。
