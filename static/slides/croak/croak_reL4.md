---
marp: true
theme: gaia
_class: lead
paginate: true
style: |
  section {
    font-size: 1.2em;
  }
---

# reL4 项目工作总结


### 1. seL4 启动阶段代码的移植

### 2. reL4 编译配置系统开发

### 3. seL4 SMP 功能的移植

--- 

# seL4 启动代码移植

这部分代码由小部分汇编代码 + Rust 代码实现，以及 linker scripts。

- Linker scripts:
  - 每种 CPU 架构一个通用脚本，linker.ld.in 
  - 相同 CPU 架构的不同平台通过配置生成不同内存布局

```
# This file is auto generated
OUTPUT_ARCH(aarch64)

KERNEL_OFFSET = 0xffffff8000000000;
START_ADDR = 0xffffff8040000000;

INCLUDE kernel/src/arch/aarch64/linker.ld.in
```

- 汇编代码，seL4 中移植过来，没什么说的
  - head.S
  - trap.S

---

- Rust 代码，如下几类函数使用 rust 重新实现
  - init_kernel
  - syscall handler
  - interrupt handler
- CMake 修改，编译 sel4test 时使用 reL4，如果有 REL4_KERNEL 这个编译选项，就引入 reL4 module

```cmake
if(REL4_KERNEL)
    find_package(reL4 REQUIRED)
endif()

if(REL4_KERNEL)
    rel4_import_kernel()
endif()
```

```cmake
if(REL4_KERNEL)
    list(APPEND cpio_files "$<TARGET_FILE:reL4_kernel.elf>")
else()
    list(APPEND cpio_files "$<TARGET_FILE:kernel.elf>")
endif()
```

--- 

# reL4 配置编译系统开发

seL4 中使用 cmake 做编译，rust 中显示无法适配，因此把用到的编译逻辑重新实现

## 配置模块

![配置系统结构图](config_system.png)

---

## seL4 中的配置

seL4 中一大堆 #define 宏，这些按照属性我们简单的分为两类

- 常量：这些是写死不会变的，比如 seL4_CapNull 这种，是定义好的。这部分我们只需要在 reL4 中定义好就可以了。

  seL4 中常量分散在各个模块中，reL4 中出于简单考虑，我们只分成两种类型

  - arch 相关的常量，放到 [sel4_common::arch::config](https://github.com/reL4team2/rel4-integral/blob/master/sel4_common/src/arch/aarch64/config.rs) 中
  - arch 无关的常量，放到 [sel4_common::sel4_config](https://github.com/reL4team2/rel4-integral/blob/master/sel4_common/src/sel4_config.rs) 中


- 配置：这些是在编译时根据配置选项生成的，比如 CONFIG_KERNEL_STACK_BITS 这种，还有一些根据硬件平台属性生成的。这部分需要在编译时根据编译目标生成，无法写死

  配置是比较 seL4 中比较复杂的部分，据我知道的，seL4 中有至少三种不同的代码生成规则和方式，也很混乱

  - 配置生成 gen_config.yaml，再从 yaml 生成 gen_config.h 文件
  - 通过 jinja 模板生成 device_gen.h
  - 通过 cmake 配置 platform_gen.h.in 生成 platform_gen.h

  在 reL4 中，我们使用统一的工具 rel4_config 从一份配置文件 platform yaml 生成两种文件

  - config.rs
  - platform_gen.rs

---

## 编译系统

- kernel, sel4_common 依赖生成文件的模块，各自有 build.rs，确保编译依赖自己生成

  例如，引用 asm 文件, kernel build.rs 中生成 kernel 编译所需的 asm 文件，linker scripts，这些生成的文件会放到 OUT_DIR 中，在 src 中引用

  ```
  #[cfg(feature = "BUILD_BINARY")]
  core::arch::global_asm!(include_str!(concat!(env!("OUT_DIR"), "/head.S")));
  #[cfg(feature = "BUILD_BINARY")]
  core::arch::global_asm!(include_str!(concat!(env!("OUT_DIR"),"/traps.S")));
  ```

  同样，bpf 文件也通过 include 的方式引用

  ```
  // pbf auto generated code
  pub mod structures_gen {
      include!(concat!(env!("OUT_DIR"), "/pbf/structures.bf.rs"));
  }
  ```

- xtask, 支持编译各个平台，各模式
  ```
      Usage: xtask build [OPTIONS]

      Options:
      -p, --platform <PLATFORM>  [default: spike]
      -m, --mcs <MCS>            [default: false]
      -s, --smc <SMC>            [default: false]
  ```


---

# SMP 功能移植

[SMP 移植文档](https://rel4team.github.io/zh/docs/reL4kernel/smp/smp_code/)

seL4 中 smp 设计较为简单，主要是通过一个 big kernel lock 确保 kernel 和单核情况下表现几乎一致，不会出现竞争的情况。

seL4 smp 设计说明可参考 [seL4 smp](https://sel4.systems/Foundation/Summit/2022/slides/d1_07_Multiprocessing_on_seL4_with_verified_kernels_Kent_Mcleod.pdf)

关于 big kernel lock, seL4 认为对于微内核来说，内核大锁并不会过多影响效率. [big kernel lock](https://arxiv.org/pdf/1609.08372)

如上所说，seL4 的 SMP 避免了内核中竞态和中断嵌套的情况，主要实现的部分如下

1. 每个核心独立数据的管理，例如任务队列
2. 调度逻辑，除了调度当前核心的任务，还需要操作别的核心的任务
3. ipi 通信和相应处理函数的实现
4. 内核大锁的实现
5. smp 相关的初始化过程的实现
6. smp 对 mcs 的支持实现
