> [!info] 核心一句话
> `-fuse-ld=xxx` 是 GCC/Clang 的**链接器选择开关**：告诉编译驱动调用哪个链接器（默认是 GNU binutils 的 `ld`）。现代 Linux 上的选择是 **lld** 或 **mold**，曾经的 **gold 已死**。

## 主流链接器一览

| 维度     | `ld`（bfd）    | `gold`                                 | `lld`                   | `mold`                   |
| ------ | ------------ | -------------------------------------- | ----------------------- | ------------------------ |
| 出身     | GNU binutils | GNU binutils（2008）                     | LLVM 项目（2013+）          | 第三方开源（2020）              |
| 平台覆盖   | ELF + 其他     | 仅 ELF                                  | ELF、COFF/PE、Mach-O、WASM | ELF 为主（支持少量其他）           |
| 速度     | 慢（单线程）       | 快（多线程）                                 | 很快，内存低                  | **最快**，大项目碾压             |
| LTO 配合 | 插件支持 GCC LTO | 插件 `liblto_plugin` 支持 GCC LTO          | 原生支持 LLVM LTO/ThinLTO   | 原生支持 LLVM LTO、GCC LTO 插件 |
| 错误信息   | 一般           | 一般                                     | 详细友好                    | 友好                       |
| 维护状态   | 活跃（仍在更新）     | **已死**：binutils 2.44（2025-02）移除，官方声明废弃 | 活跃，随 LLVM 持续更新          | 活跃，快速迭代                  |

## gold vs lld 重点对比

- **出身**：gold 属于 GNU 工具链，lld 属于 LLVM 生态
- **平台覆盖**：gold 只做 ELF；lld 一套代码覆盖 Linux/macOS/Windows/WASM
- **速度与内存**：两者都快，但 lld 在多线程和内存占用上更优，超大二进制差距明显
- **LTO 配合**（实践中最容易踩的坑）：

```bash
# gold：需要手动指定插件（GCC LTO 场景）
gcc -flto -fuse-ld=gold -Wl,-plugin,liblto_plugin.so ...

# lld：LLVM 的 LTO 是原生的，Clang 直接配合
clang -flto=thin -fuse-ld=lld ...
```

> [!warning] gold 的现状
> gold 已从 binutils 2.44 起停止随默认发行，Fedora 等发行版已移除，Rust 等工具链早已放弃它。**新项目不要再用 gold。**

## 2026 年怎么选

- 默认 / 追求稳妥：`-fuse-ld=lld`（生态最完整，Clang LTO/ThinLTO 官方配套）
- 超大项目、构建速度优先：`-fuse-ld=mold`（当前最快的 ELF 链接器）
- 需要跨平台（macOS + Windows + Linux 同一套）：
  - macOS：Clang 默认 `ld64`，`-fuse-ld=lld` 可换 lld 的 Mach-O 版本
  - Windows：`lld-link` 配合 MSVC 或 clang-cl

## 换链接器，二进制会不一样吗

会，且**几乎必然不同**，分两个层面：

**字节级差异（几乎一定出现）**

- **布局与地址**：各链接器的节区排序、对齐、符号分配算法不同 → 符号地址、段偏移全不同
- **哈希表**：lld 默认 `--hash-style=both`（`.hash` + `.gnu.hash`）；GNU ld 默认只有 `.hash`
- **动态标签**：lld 默认 `DT_RUNPATH`，GNU ld 默认 `DT_RPATH`（影响运行时库搜索优先级）
- **`.comment` 节**：嵌入各自的版本串（`GNU ld 2.44` vs `LLD 23.x`）
- **孤儿节区放置**：无 linker script 时放置规则不同

**行为级差异（少数场景真的不同）**

- **静态库提取策略**：lld 扫描归档的方式与传统链接器不同（自动处理循环依赖，`--start-group` 都被忽略），个别归档组合下提取的成员集合可能不同
- **version script 容错**：lld 默认对引用未定义符号的版本脚本**直接报错**（`--no-undefined-version` 默认开启），GNU ld 可能只警告
- **LTO 集成路径**：gold 走 `liblto_plugin`（GCC LTO），lld 走原生 LLVM LTO → 优化决策不同 → 代码不同
- **`--gc-sections` / `--icf`**：默认都关，显式开启时各自的判定规则有差别

验证：

```bash
cmp a.out b.out
readelf -d b.out      # DT_NEEDED / DT_RUNPATH
objdump -d b.out | head
```

## 换链接器会影响运行效率吗

多数情况**影响可忽略**（<1%），真正拉开差距的是编译器优化。可能受影响的地方：

| 影响点 | 量级 | 说明 |
|---|---|---|
| LTO 集成路径不同 | 可感知 | 根源是优化器不同（GCC LTO vs LLVM LTO），不是链接器本身 |
| 二进制大小 → 加载/页错误 | 启动阶段 | `--gc-sections` + `--icf` 显式开启时差异明显 |
| 哈希表（gnu vs sysv） | 启动毫秒级 | 动态符号查找速度 |
| 函数/节区布局 | ~1% | lld 的 `--call-graph-profile-sort`（默认 cdsort） |

排查方法：

```bash
# 控制变量：同一编译器、同一 -O，只换链接器
gcc -O2 -fuse-ld=gold -o a.out a.c b.c
gcc -O2 -fuse-ld=lld  -o b.out a.c b.c
hyperfine --warmup 3 './a.out' './b.out'     # 多轮取中位数
perf stat -e cycles,instructions,cache-misses ./a.out
```

判断标准：稳态计算段差异 >1% 先怀疑 **LTO 路径不一致**；差异只在启动/加载阶段属正常，用 `--hash-style=gnu` + `--gc-sections` 缓解。

## 为什么 LTO 明明是编译选项，却要链接器配合

`-flto` 改变的是**编译产物**：.o 里装的不是机器码而是 IR（bitcode）。真正的优化在**链接时**执行，所以链接器必须"会做 LTO"：

| 场景 | 谁执行 LTO |
|---|---|
| `clang -flto` + lld | lld 原生（LLVM LTO/ThinLTO），官方路径 |
| `clang -flto` + GNU ld/gold | LLVMgold.so 插件 |
| `gcc -flto` + GNU ld/gold | liblto_plugin.so 插件（GCC 自己的 LTO） |
| `gcc -flto` + lld | ❌ lld 不认识 GCC 的 GIMPLE IR |

所以选链接器 ≈ 选"谁来做 LTO、用什么优化器做"。详见 [[LTO]]。

## 一句话总结

gold 是"曾经很快的 GNU 链接器"，lld 是"仍在进化的 LLVM 链接器"，mold 是"当前最快的后起之秀"；`-fuse-ld=` 就是在这几个候选之间切换的开关。
