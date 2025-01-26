+++
date = '2025-01-25T09:47:15+08:00'
title = 'reL4 配置系统设计'
+++

# reL4 配置系统设计

## 1. 目标

由于去除 seL4 的编译系统，因此 seL4 基于 CMake 的配置系统同样会被去除。因此我们需要设计一个简化的、足够使用的配置系统供 reL4 使用。

目的是将所有的配置项统一管理，以适应各种 CPU 架构，各种 SoC 平台，减少增加 SoC 平台所需的工作量。主要包括以下几类配置项。

1. Memory layout
2. CPU 信息，主频之类
3. 一些定义在 config.rs 中的常量

编译过程中需要生成以下文件

1. linker scripts
2. device_gen.rs
3. config.rs

seL4 中之前使用 cpp 生成代码的工作暂时先不考虑做，而是使用 rust cfg 功能进行替代。

同样，之前一些通过 dtc 获取的信息，应该也不包含在其中。

## 2. 文件生成

### 2.1 linker scripts 生成

在 linker scripts 中，主要是 **段的起始地址** 和 **虚实地址之间的 offset**，如下例子

```
# linker_gen.ld
# This file is auto generated
OUTPUT_ARCH(riscv)

KERNEL_OFFSET = 0xffffffff00000000;
START_ADDR = 0xffffffff84000000;

INCLUDE kernel/src/arch/linker.ld.in
```

相同的链接脚本定义都抽象在 linker.ld.in 中，linker_gen.ld 只需要定义相关参数，该文件根据配置文件中的参数自动生成。

文件生成实现参考 tools/generator.py 中的 linker_gen function

### 2.2 device_gen.rs 生成

device_gen.rs 文件定义与各个平台相关的硬件参数，比如 CPU 主频，可用的内存区域等等，如下生成的例子

```
// This file is auto generated
use crate::structures::p_region_t;

#[link_section = ".boot.bss"]
pub static avail_p_regs: [p_region_t; 1] = [
    p_region_t {
       start: 0x80200000,
       end: 0x17ff00000
    },
];
```

可以看到目前只定义了 avail_p_regs，该变量会根据配置文件中 avail_mem_zone 参数生成

### 2.3 config.rs 生成

config.rs 中存储与硬件无关的参数，是否需要通过配置文件生成，还需要进一步调查。

## 3. 配置项

配置项会根据开发过程不断更新，目前最简单的配置项设计如下，只有 CPU 相关和 内存 相关参数

```
# kernel/platform/spike.yml
# cpu arch
cpu:
  arch: riscv
  freq: 10000000

# memory layout
memory:
  vmem_offset: 0xFFFFFFFF00000000
  pmem_start: 0x80000000
  kernel_start: 0x84000000
  avail_mem_zone:
    - {start: 0x80200000, end: 0x17ff00000}

```

## 4. 疑难点

### 4.1 与 sel4_common 的冲突

在 sel4_common 中已经定义了一些 platform 相关变量，在 sel4_common/src/platform 下，包括 CPU 主频等定义。如果沿用之前的架构和设计，那么就需要在 sel4_common 中也生成一个 dev_gen.rs。如果把 sel4_common 中的代码移植到 kernel 中，可能工作量会很大，如何做还需要讨论。