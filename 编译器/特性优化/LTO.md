> [!info] 核心一句话
> **LTO（Link-Time Optimization，链接期优化）** = 把优化从"单个源文件"推迟到"整个程序链接时"，让编译器看到全部代码再做全局决策。

## 原理

传统编译每个源文件单独优化（看不到别的模块），链接时已经优化完，只能机械合并。LTO 让编译阶段只产出**中间表示**（GCC 的 GIMPLE / LLVM IR / bitcode），链接时统一加载、分析、优化后再生成机器码：

```
传统:  a.c ──编译+优化──> a.o ─┐
       b.c ──编译+优化──> b.o ─┤──链接(无优化)──> 可执行文件

LTO:   a.c ──编译(只出IR)──> a.o ─┐
       b.c ──编译(只出IR)──> b.o ─┤──链接+全局优化──> 可执行文件
```

## 常见优化收益

### 跨模块内联（最经典）

函数内联不再限于同一编译单元。`a.c` 里调用 `b.c` 的小函数，LTO 后可以跨文件内联，消除调用开销并开启下游优化。

### 全局死代码消除与常量传播

- 全程序可达性分析：**没被任何地方引用的函数/符号直接删掉**（对静态库减肥尤其有效）
- 常量传播能穿过函数边界：知道 `foo(42)` 总是传同一个值，就能跨模块折叠

### 调用图驱动的去虚拟化

有了整个程序的调用图，可以分析函数指针/虚函数的**可能目标集合**：

- 目标唯一 → 直接变成直调，可内联
- 目标很少 → 生成守卫做间接调用提升（效果类似 [[PGO]] 的值分析，但这是**静态推导**，不需要运行数据）

### 全局代码布局

基于全程序调用图做**函数排序、热冷分离**，优化 icache 局部性；GCC/Clang 还能让跨模块代码共用同一份类型/符号信息，减少重复代码。

### 过程间优化（IPO）

改进全局别名分析 → 更激进的寄存器分配、公共子表达式消除、内存访问优化。

## 实现变体

| 实现                          | 说明                                                      |
| --------------------------- | ------------------------------------------------------- |
| GCC `-flto`                 | 分模块并行（WPA + ltrans），`-fno-fat-lto-objects` 可产出纯 bitcode |
| Clang `-flto=full`          | 经典 LTO，整个程序一次载入优化                                       |
| Clang `-flto=thin`（ThinLTO） | 分模块并行 + 只交换摘要信息，编译更快、可增量、内存低，还能跨语言（C/C++/Rust）          |
| MSVC `/GL` + `/LTCG`        | 微软对应实现，常与 `/LTCG:PGI/PGO` 配合                            |

> [!tip] 配合使用
> **LTO + PGO 是经典组合**：LTO 提供全程序视野，PGO 提供真实运行数据，二者互补。ThinLTO 的摘要信息还能反哺 PGO 的跨模块内联决策。

## 实操：命令示例

```bash
# Clang + lld（LLVM 生态标准路径）
clang -flto=thin -fuse-ld=lld -O2 -o app a.c b.c

# GCC 走插件（gold / GNU ld）
gcc -flto -O2 -fuse-ld=gold -Wl,-plugin,liblto_plugin.so -o app a.c b.c

# MSVC
cl /GL a.c b.c
link /LTCG /OUT:app.exe a.obj b.obj

# 验证 LTO 真的执行了（-v 能看到传给 lld 的 --plugin-opt 参数）
clang -flto=thin -fuse-ld=lld -O2 -v -o app a.c b.c
```

## ThinLTO 是怎么工作的

完整 LTO 要把所有 bitcode 一次载入内存，大项目吃不消。ThinLTO 的思路是**只交换摘要，不搬代码**：

```
a.o (bitcode + index) ─┐
b.o (bitcode + index) ─┤── ThinLTO 后端 ──> 并行处理，只优化"被导入"的函数
c.o (bitcode + index) ─┘
  每个模块只 import 自己需要的其他模块函数；
  index 文件描述"谁导出了什么"，模块间只交换这份摘要
```

- 编译快（并行、可增量）、内存低
- 优化质量接近 fat LTO，是当前主流（Chrome、Android 等大项目都在用）

## rustc 怎么开 LTO

rustc 前端 + LLVM 后端，机制和 clang 同源（详见 [[前端 vs 后端]]）：

```toml
# Cargo.toml
[profile.release]
lto = true          # 或 "thin" / "fat" / false
codegen-units = 1   # fat LTO 建议配合
```

```bash
# 链接器用 rust-lld（rustc 自带的 lld，开箱即用）
RUSTFLAGS="-C lto=thin -C linker=rust-lld" cargo build --release

# 跨语言 LTO（Rust FFI + C/C++，要求两边 LLVM 版本匹配）
RUSTFLAGS="-C linker-plugin-lto -C linker=clang -C link-arg=-fuse-ld=lld" cargo build --release
```

要点：

- `lto` 设置作用于你的 crate **和所有依赖**（rlib 默认带 bitcode，能参与 LTO）
- LTO 与增量编译冲突：debug 默认不开 LTO、release 默认不开 incremental，恰好互不干扰
- 确认生效：`readelf -S target/release/deps/xxx.o` 里找 `.llvm.lto` 段

## 使用前提与坑

> [!warning] 注意
> - **编译和链接时间明显变长**，内存占用高（完整 LTO 尤甚，ThinLTO 缓解）
> - 需要**统一工具链**：所有编译单元必须用同一家编译器的 IR，混用不同编译器会失效
> - 调试体验变差：行号/变量信息需要额外元数据（`-g` 与 LTO 配合有开销）
> - 存在**跨语言限制**：不是所有语言组合都能 LTO（C/C++ 与 Rust 通过 LLVM 可部分互通）
> - 静态库场景收益最大，但也最容易因符号可达性分析误删导致链接失败（通常可加 `-Wl,--export-dynamic` 或标记保留）

## 一句话总结

LTO 把"局部优化"升级为"全局优化"，主要收益是**跨模块内联、全局死代码消除、调用图去虚拟化和代码布局**，代价是构建变慢；ThinLTO 和与 PGO 的组合是当前主流实践。
