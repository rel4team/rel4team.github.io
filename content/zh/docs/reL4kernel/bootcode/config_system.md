---
weight = 10
date = '2025-01-25T09:47:15+08:00'
title = 'reL4 配置系统设计'
---

# reL4 配置系统设计

## 1. 目标

由于去除 seL4 的编译系统，因此 seL4 基于 CMake 的配置系统同样会被去除。因此我们需要设计一个简化的、足够使用的配置系统供 reL4 使用。

目的是将所有的配置项统一管理，以适应各种 CPU 架构，各种 SoC 平台，减少增加 SoC 平台所需的工作量。主要包括以下几类配置项。

1. Memory layout
2. CPU 信息，主频，时钟之类 (也许不需要，dtc 中有定义)
3. 一些定义在 config.rs 中的常量

编译过程中需要生成以下文件

1. linker scripts
2. device_gen.rs
3. config.rs
4. head.S, trap.S

**需要说明的是，和输入的 DTC 不同，这些配置生成文件是在编译时就需要的，而不是运行时需要的**

## 2. 文件生成

目前定义生成三种文件，使用一个 python 脚本，generator.py。用 rust build.rs 也可以做到，相对复杂一些，后面可能还是要使用 build.rs。

- dev_gen.rs
- linker scripts
- trap.S head.S

### 2.1 dev_gen.rs 生成

这部分非常需要讨论（当然整个配置系统的设计都需要），目前我只生成了 Memory layout，也就是 avail_p_regs。

但是我看到 seL4_common 里面有一些时钟相关的配置，我觉得这部分也应该用配置文件解决。platform 相关的代码和配置应该集中到一起，可以放在 kernel 中，也可以放在 seL4_common 中。

生成细节参考 generator.py 中 dev_gen 函数，总的来说，就是用写文件的方式将配置文件中的一些变量写到 rs 文件里。

### 2.2 linker scripts 生成

linker scripts 相对简单一些，因为相同 CPU 架构的 SoC platform 使用的 linker scripts 基本类似，有区别的就是内存布局。所以我们使用 linker 头文件 + linker scripts 的方式，绝大部分相同的定义都写在头文件里，linker scripts 中只需要对一些变量赋值即可。

头文件写法请参考 arch/riscv/linker.ld.in 和 arch/aarch64/linker.ld.in

linker scripts 中只需要定义 KERNEL_OFFSET START_ADDR 这些内存相关的变量即可，linker scripts 生成细节参考 generator.py 中的 linker_gen 函数

### 2.3 汇编文件生成

和前两个文件有点不同，汇编文件本质上和 platform 是没关系的。但是汇编文件中有很多 ifdef #define 的预编译的语句，几乎没有办法使用 rust cfg 代替，替换工作量巨大。因此还是考虑使用 gcc -E 做预编译的工作。

这样做的好处是，我们几乎可以将 seL4 中的汇编代码原封不动的迁移过来，大大减轻移植的工作量，缺点就是，我们还需要增加几个 .h 头文件，将一些相关的宏定义放进去。头文件在 kernel/include 下

汇编文件生成参考 asm_gen 函数

## 3. 配置项

目前完全是按需定义配置项，没有一个明确的设计，说实话我也不知道怎么通过配置文件完整的定义一个 SoC 平台。

现有的配置文件内容如下

```
# cpu arch
cpu:
  arch: aarch64
  freq: 10000000

# memory layout
memory:
  vmem_offset: 0xffffff8000000000
  pmem_start: 0x40000000
  kernel_start: 0x40000000
  avail_mem_zone:
    - {start: 0x40000000, end: 0x80000000}
  stack_bits: 12 # 2^12 4K
```

## 4. 讨论

1. 目前还有哪些 platform 相关的配置，需要合在一起
2. 目前这种配置方式是否满足 kernel 需求，有哪些需要改进的地方。现在只能做到尽量将所有配置相关代码集中到一起，方便后面优化。
3. 目前还没有生成 config.rs，但是长远看，如果完全去掉 cmake 后，config.rs 也是需要生成的。这部分还没想好怎么做。