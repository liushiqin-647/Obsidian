# rustc 如何选择 codegen 后端

rustc 使用 LLVM 后端生成机器码，但这**不是前端决定的**——是 rustc 的 driver 在编译开始前就选定的。rustc 支持多个 codegen 后端，前端本身对此一无所知。

## 流程

```text
rustc 启动（driver）→ 选定 codegen 后端 → 前端解析 → 生成 MIR → 交给已选定的后端 → 机器码
```

选定发生在前端解析源码**之前**。等前端开始干活时，"用哪个车间"早就定好了。

## 前端止步于 MIR

rustc 有自己的一套 IR 层次：

```text
源码 → AST → HIR → MIR → (交给 codegen 后端) → 机器码
```

- **HIR（High-level IR）**：名字解析、类型检查等语言层面的工作，仍带大量 Rust 概念
- **MIR（Mid-level IR）**：rustc 定义的核心中间表示，形式接近控制流图（CFG）。借用检查、生命周期检查都在这一步完成。MIR **完全与 LLVM、GCC 无关**

前端和中端全程只跟 MIR 打交道。前端生成完 MIR 后，通过 `CodegenBackend` trait 把 MIR 传给已选定的后端实现——它不知道、也不需要知道后端是谁。

## 如何选择

| 方式 | 说明 |
|---|---|
| **默认** | 官方 rustc 默认使用 `rustc_codegen_llvm` |
| **命令行（nightly）** | `-Z codegen-backend=cranelift` 或 `gcc` |
| **Cargo** | `cargo +nightly rustc -- -Zcodegen-backend=cranelift` |
| **编译时** | rustc 二进制里内置了哪些后端实现，没编进去的就选不了 |

## 各 codegen 后端

| 后端 | 说明 |
|---|---|
| **`rustc_codegen_llvm`** | 最常用。把 MIR 翻译成 LLVM IR，再交给 LLVM 后端生成机器码 |
| **`rustc_codegen_cranelift`** | 直接从 MIR 翻译成机器码，**完全不经过 LLVM IR**——证明前端确实不知道 LLVM 存在 |
| **`rustc_codegen_gcc`** | 把 MIR 翻译成 GCC 能处理的形式，交给 GCC 后端 |

## 与 libLLVM.so 的关系

选择后端和加载 `libLLVM.so` 是两码事：

- **选择是逻辑决定**：driver 根据配置选定后端实现，这本身不依赖 `libLLVM.so` 是否存在
- **加载是实现细节**：只有选中 `rustc_codegen_llvm` 时才会用到 LLVM。可能动态加载 `libLLVM.so`（常见于 Linux 官方 rustc），也可能在构建 rustc 时静态链接进去——后者完全不需要外部 `libLLVM.so`
- **其他后端不用 LLVM**：Cranelift、GCC 后端完全不会加载 `libLLVM.so`

可用 `ldd $(which rustc)` 验证：动态链接的 rustc 能看到 `libLLVM.so`，静态链接的看不到。

## 核心要点

**driver 先选中 `rustc_codegen_llvm`，这个实现才去调用/加载 LLVM。** 不是检测到 `libLLVM.so` 才选 LLVM——"选哪个后端"由配置决定，"怎么调用 LLVM"由 `rustc_codegen_llvm` 自己决定。

相关：[[中间表示（IR）]]、[[前端 vs 后端]]
