---
title: "reL4 配置模块"
weight: 7
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# reL4 配置系统 v2 版本

## 1. 介绍

reL4_config 在 [编译系统 v2 版本](./build_system_v2.md) 中已经简单介绍了，是一个管理配置和生成代码的模块。由于最近又增加了一些功能，将其单独拿出来写成一个文档。

同时介绍下目前 rel4 中的配置和常量，这部分由于本身 seL4 的配置和常量就非常多且分散，因此在 reL4 中也挺乱的。最近整理下，在本文中介绍下近况，方便参考。

## 2. reL4 中配置和常量

seL4 中一大堆 #define 宏，这些按照属性我们简单的分为两类

- 常量：这些是写死不会变的，比如 seL4_CapNull 这种，是定义好的。这部分我们只需要在 reL4 中定义好就可以了。
- 配置：这些是在编译时根据配置选项生成的，比如 CONFIG_KERNEL_STACK_BITS 这种，还有一些根据硬件平台属性生成的。这部分需要在编译时根据编译目标生成，无法写死

### 2.1 常量

seL4 中常量分散在各个模块中，reL4 中出于简单考虑，我们只分成两种类型

- arch 相关的常量，放到 [sel4_common::arch::config](https://github.com/reL4team2/rel4-integral/blob/master/sel4_common/src/arch/aarch64/config.rs) 中
- arch 无关的常量，放到 [sel4_common::sel4_config](https://github.com/reL4team2/rel4-integral/blob/master/sel4_common/src/sel4_config.rs) 中

如果你想添加一些新的常量，也请添加在这两个地方。集中在一起方便后续的管理和优化。

### 2.2 配置

配置是比较 seL4 中比较复杂的部分，据我知道的，seL4 中有至少三种不同的代码生成规则和方式，也很混乱

- 配置生成 gen_config.yaml，再从 yaml 生成 gen_config.h 文件
- 通过 jinja 模板生成 device_gen.h
- 通过 cmake 配置 platform_gen.h.in 生成 platform_gen.h

在 reL4 中，我们使用统一的工具 rel4_config 从一份配置文件 platform yaml 生成两种文件

- config.rs
- platform_gen.rs

#### 2.2.1 yaml 配置文件

其中 platform yaml 是针对每个硬件平台（比如 spike, qemu-arm-virt）的配置文件，包含这个硬件平台的 cpu ，内存，外设等硬件信息。同时包含一些平台相关的配置信息。yaml 配置文件内容如下

```yaml
# cpu arch
cpu:
  arch: aarch64
  freq: 62500000

# timer settings
timer:
  - {label: "CLK_MAGIC", value: 4611686019}
  - {label: "CLK_SHIFT", value: 58}
  - {label: "TIMER_PRECISION", value: 0}  
  - {label: "TIMER_OVERHEAD_TICKS", value: 0}
  - {label: "CONFIGURE_KERNEL_WCET", value: 10}

# device messages
device:
  device_region:
    - {paddr: 0x9000000, pptr_offset: 0x0, arm_execute_never: 1, user_available: 1, desc: "uart"}
    - {paddr: 0x8000000, pptr_offset: 0x1000, arm_execute_never: 1, user_available: 0, desc: "gicv2_distributor"}
    - {paddr: 0x8010000, pptr_offset: 0x2000, arm_execute_never: 1, user_available: 0, desc: "gicv2_controller"}
  irqs:
    - {label: "INTERRUPT_VTIMER_EVENT", number: 27}
    - {label: "KERNEL_TIMER_IRQ", number: 27}
    - {label: "maxIRQ", number: 159}
    - {label: "irqInvalid", number: 0}

# memory layout
memory:
  vmem_offset: 0xffffff8000000000
  pmem_start: 0x40000000
  kernel_start: 0x40000000
  avail_mem_zone:
    - {start: 0x40000000, end: 0x80000000}
  stack_bits: 12 # 2^12 4K

definitions:
  ARCH_AARCH32: false
  ARCH_AARCH64: true # KernelSel4ArchAarch64=ON
  ARCH_ARM_HYP: false
...

```

我们所有的配置信息都在这一个 yaml 文件中，所有的代码生成都基于一个文件。这样有些内容会冗余（比如 riscv 架构统一的配置需要写在每一个 riscv 硬件平台上），但是更加直观和简单。

#### 2.2.2 config.rs

rel4_config 根据 yaml 文件中的 definitions 生成 config.rs 文件，这个过程等于 seL4 中的 gen_config.h 生成过程。

生成 config.rs 代码参考 [generator.rs](https://github.com/reL4team2/rel4-integral/blob/master/rel4_config/src/generator.rs) 中的 config_gen 函数.

在 sel4_common build.rs 中调用 config_gen, 生成 config.rs 后通过 include! 的方式导入到 sel4_config.rs 中。

```rust
include!(concat!(env!("OUT_DIR"), "/config.rs"));
```

#### 2.2.3 platform_gen.rs

rel4_config 根据 yaml 文件中的 definitions 生成 platform_gen.rs 文件，这个过程是 seL4 中 platform_gen.h 和 device_gen.h 生成过程的综合

生成 config.rs 代码参考 [generator.rs](https://github.com/reL4team2/rel4-integral/blob/master/rel4_config/src/generator.rs) 中的 platform_gen 函数，是基于 [tera]() 实现的，tera 类似 jinja 的 rust 版本。

platform_gen.rs 使用的模板定义在 [platform_gen template](https://github.com/reL4team2/rel4-integral/blob/master/rel4_config/template/platform_gen.rs) 

后续如果还有其他的配置文件需要生成，我会优先考虑使用 tera 生成。比如之前 config 其实用 tera 实现更简单。
