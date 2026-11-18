# AgentBench 指令派发与结果回传顺序图

来源：`src/api/proxy.rs`、`src/orchestrator/proxy.rs`、`src/cfg.rs`（envd 端口 49983）

![顺序图](agentbench-command-flow.svg)

说明：实线箭头是调用，虚线箭头是返回，黄色小框是同一角色内部的工作。橙色括号是派发和执行阶段，蓝色括号是结果回传并写回轨迹阶段。图中只保留参与函数调用的进程。

## 派发

1. AgentBench 从轨迹里取出这一步的动作（一条命令），通过 E2B SDK 发 `POST /commands`，用沙箱 ID 告诉服务找哪个沙箱。
2. AgentENV 的反向代理按沙箱 ID 查路由表，拿到沙箱 IP 和 envd 端口 49983；如果沙箱处于暂停状态，先唤醒再转发。
3. 代理把 HTTP 请求原样转发到沙箱内的 envd，并补上鉴权头。
4. envd 解析命令，在沙箱里用 shell 执行。

## 结果回传

5. shell 把 stdout、stderr 和退出码交给 envd。
6. envd 按 E2B 协议回给代理，普通模式是一段 JSON，流式模式是边产生边回传。
7. 代理把响应原样回传给 AgentBench，状态码和 body 都不改。
8. SDK 解析出 stdout / stderr / exit_code，AgentBench 把这一段写进轨迹的 observation。
9. 下一步模型看到的是“历史轨迹 + 这次观察结果”，再生成下一个动作。

## 为什么没有 Firecracker

代理是直接 TCP 连沙箱 IP 的 49983 端口，代码里没有任何对 Firecracker 的调用。Firecracker 只是在内核层把包从 tap0 搬进虚拟机，属于传输管道，不参与函数调用，所以不画。

轨迹本身也不进入沙箱：沙箱只执行当前这一条命令，历史由 AgentBench 在外部维护。
