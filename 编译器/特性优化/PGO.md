> [!info] 核心一句话
> **PGO（Profile-Guided Optimization）** = 先跑一遍程序记录真实运行数据，再根据数据重新编译做优化决策，让编译器从"猜"变成"看数据"。

## 三阶段流程

PGO 的核心思路：**用真实运行数据替代编译器的静态猜测**。整个过程需要编译两次，外加一次训练运行。

```mermaid
flowchart LR
    A["源码<br/>(source.c)"] -->|"① 插桩编译<br/>-fprofile-instr-generate"| B["插桩版程序<br/>(带计数器)"]
    B -->|"② 训练运行<br/>跑代表性负载"| C["profile 数据<br/>(.profraw / .gcda)"]
    C -->|"③ 反馈优化编译<br/>-fprofile-instr-use"| D["上线版程序<br/>(无插桩)"]
    classDef prod fill:#e8f0fe,stroke:#4a6fa5,stroke-width:1px;
    classDef data fill:#fff3e0,stroke:#e6a23c,stroke-width:1px;
    class A,B,D prod
    class C data
```

| 阶段 | 说明 | 产物 |
|---|---|---|
| **① 插桩编译** | 编译器在代码中插入计数器，用于统计各基本块、分支的实际执行次数。 | 插桩版可执行文件 |
| **② 训练运行** | 用能反映真实使用场景的负载跑这版插桩程序。运行时会把统计结果写入 profile 文件。 | `.profraw`（Clang）或 `.gcda`（GCC） |
| **③ 反馈优化编译** | 用同一份源码再次编译，读取 profile 数据。编译器会根据实际热点调整内联、代码布局、分支预测等决策，生成最终的优化版。 | 上线版可执行文件（无插桩） |

> [!tip] 注意
> - **训练负载很关键**：没跑到的路径只会被当成冷代码来处理，覆盖不全反而可能拖累性能。
> - **插桩版不能上线**：除了解析开销外，它不是最终优化版。
> - **profile 必须匹配**：如果源码有较大改动，旧的 profile 数据就不再适用，得重新跑一遍 ①→②→③。

主流编译器支持：GCC（`-fprofile-generate` / `-fprofile-use`）、Clang（`-fprofile-instr-generate` / `-fprofile-instr-use`）、MSVC（`/LTCG:PGINSTRUMENT` / `/LTCG:PGOPTIMIZE`）。具体命令见「实操」一节。

## PGO 优化的是哪个阶段（前端/中端/后端）

- **前端完全不参与**：词法/语法/语义分析不读任何运行数据，profile 对它毫无意义
- **中端（IR 优化）**：内联、常量传播、循环优化、去虚拟化——主要受益者
- **后端（机器层）**：代码布局、分支概率、寄存器分配——重要受益者
- 插桩发生在 IR 层（GCC 的 GIMPLE / LLVM IR），采集和消费都在中端及以后

| 下面的优化手段 | 作用阶段 |
|---|---|
| 内联决策 | 中端 |
| 间接调用提升（去虚拟化） | 中端 |
| 循环优化（展开因子等） | 中端 |
| 基本块重排、热冷分离 | 后端 |
| 函数排序 | 后端 |
| 分支概率 / if-conversion | 后端 |
| 结构体字段重排 | 后端（数据布局） |

PGO 不是某个组件的功能，而是**横跨中端+后端的决策数据**（编译器结构见 [[前端 vs 后端]]）。

## 常见优化手段

### 代码布局优化（最经典）

- **基本块重排**：按实际执行频率把"热路径"基本块放一起，错误处理等冷块挪到远处
- **热冷分离**：冷代码移出热循环区域（GCC 的 `-freorder-blocks-and-partition` 会把它放到 `.text.unlikely`）
- **函数排序**：热函数彼此相邻存放，提升指令缓存（icache）局部性和分支预测成功率

### 内联决策优化

静态内联是猜的，PGO 知道**每个调用点实际被调了多少次**：

- 热调用点果断内联（哪怕代码变大）
- 冷调用点放弃内联（避免 bloat）
- 配合 LTO 还能跨模块内联

### 间接调用提升（Devirtualization）

通过**值分析（value profiling）**记录函数指针/虚函数的实际目标：

```c
// PGO 前：每次都要间接跳转，无法预测
obj->vtable->run();

// PGO 后：最常见目标是 A::run，生成守卫
if (obj->type == TYPE_A) {          // 直调，可内联、可预测
    A::run(obj);
} else {
    obj->vtable->run();             // 少见路径保留兜底
}
```

对 C++ 虚函数、C 函数指针、Go 的 interface、以及 switch 的常见取值都有效。

### 分支概率与分支消除

插桩数据给出每个分支的真实概率：

- 低概率分支尽量**不对齐、不预取**，减少指令 cache 污染
- 极端情况下做 **if-conversion**：把分支变成条件执行指令，彻底消灭分支（消除分支预测失败）

### 循环优化决策

知道真实迭代次数后：

- **循环展开因子**按实际热循环调整
- **循环分块 / 循环剥离**策略更准
- 寄存器分配优先照顾热循环内的变量

### 数据布局优化

- **结构体字段重排**：按访问频率排列字段，热字段放同一缓存行
- **热/冷字段拆分**：把很少访问的大字段拆到别处，减少缓存行浪费
- 对访存密集型程序收益往往比代码优化还明显

### 采样式 PGO（不用插桩重编译）

**AutoFDO / sample-based PGO**：直接用硬件性能计数器（如 `perf`）在生产环境采样调用栈和热点，无需插桩版本，避免插桩开销和训练代码与线上代码不一致的问题。

### 更进阶的变体

- **CSPGO（上下文敏感 PGO）**：同一函数在不同调用点的 profile 分开记录，优化更精确（GCC 支持）
- **BOLT / Propeller**：编译完成后再对二进制做布局级 PGO 优化，不动源码

## 实操：完整流程（GCC / Clang）

```bash
# ===== GCC =====
gcc -O2 -fprofile-generate -o app.train app.c   # ① 插桩编译
./app.train < 训练数据                           # ② 跑训练负载 → 生成 app.gcda
gcc -O2 -fprofile-use -o app app.c              # ③ 反馈优化编译

# ===== Clang =====
clang -O2 -fprofile-instr-generate -o app.train app.c
./app.train < 训练数据                           # 生成 default.profraw
llvm-profdata merge -o app.profdata default.profraw   # 合并/转格式
clang -O2 -fprofile-instr-use=app.profdata -o app app.c

# ===== 与 LTO 组合（经典配置）=====
clang -O2 -flto=thin -fprofile-instr-generate -o app.train app.c
./app.train < 训练数据
llvm-profdata merge -o app.profdata default.profraw
clang -O2 -flto=thin -fprofile-instr-use=app.profdata -fuse-ld=lld -o app app.c
```

要点：

- **训练负载 = 性能基准的输入**：用真实的代表性场景，不要随便跑一遍
- 插桩版（第二版）只用于采集，**上线的永远是第三版**
- 源码改动后 profile 失效，编译器会报 mismatch 警告，需重新采集

## 使用前提与坑

> [!warning] 注意
> - **训练负载要有代表性**：profile 只代表你跑过的场景，覆盖不到的路径会被当成冷代码，可能反而变慢——这是 PGO 最大的过拟合风险
> - profile 必须与编译版本匹配（源码变了要重新采集）
> - 插桩版本本身有运行时开销（一般 10%~30%），只在测试环境跑

## 一句话总结

PGO 让编译器从"猜"变成"看数据"，收益集中在**代码布局、内联、去虚拟化**这几类决策上，典型能带来 5%~30% 的性能提升（视程序而定）。
