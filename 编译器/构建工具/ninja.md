# Ninja 构建工具

Ninja 是一个**构建工具**（构建系统），类似 `make`，但更快、更简洁——它是 CMake 生成构建脚本后的"执行者"。

## 它是什么

- 定位：把编译命令按**依赖关系**调度执行，支持并行和增量构建
- 作者：Evan Martin（Google），设计哲学就是"快"
- 特点：
  - 默认**并行**（`-j`），增量构建只重编变化的部分
  - 没有 make 那套隐式规则，一切显式写在 `build.ninja` 里 → 简单、可预测
  - 用 `.ninja_log` / `.ninja_deps` 记录构建状态，实现高效增量

## 和 CMake 的关系（最常见的使用方式）

```
CMakeLists.txt ──cmake -G Ninja -B build──> build/build.ninja ──ninja -C build──> 编译并链接
```

- CMake 负责"配置"：探测环境、生成构建规则
- Ninja 负责"执行"：按依赖图并行跑编译、链接

```bash
cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=Release   # 生成 build.ninja
ninja -C build                                        # 构建整个项目
```

## 常用命令

| 命令 | 作用 |
|---|---|
| `ninja -C build` | 在 build 目录里执行构建（`-C` = change directory） |
| `ninja -C build clang` | 只构建指定目标（LLVM 里常用 `clang`、`lld`、`check-all`） |
| `ninja -j 16` | 指定并行度（默认已尽量并行） |
| `ninja -C build -t targets` | 列出所有可构建目标 |
| `ninja -C build clean` | 清理构建产物 |
| `ninja -C build install` | 安装产物 |
| `ninja -C build -t compdb` | 导出 compile_commands.json（IDE/编辑器用） |

## 和 make 的区别

| | make | ninja |
|---|---|---|
| 规则 | 支持隐式规则、模式规则 | 全部显式写在 `build.ninja` |
| 速度 | 大项目增量构建慢 | 专门优化过，LLVM/Chrome 等大项目首选 |
| 谁来写 | 手写 Makefile | **基本不手写**，由 CMake 等生成 |
| 适合 | 中小项目、手写构建 | 大项目、生成式构建 |

## 一句话总结

Ninja = "快版 make"。平时你不会直接写它的规则文件，而是用 `cmake -G Ninja` 生成，然后执行 `ninja -C build` 来构建——构建 LLVM 时那句 `ninja -C /build` 就是这个意思。
