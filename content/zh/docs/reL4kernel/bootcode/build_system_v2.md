---
title: "编译系统 v2 版本"
weight: 6
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# reL4 编译系统 v2 版本

## 1. 介绍

之前我们已经实现了一个可以使用的编译系统 [配置系统](./config_system.md)。但是这个编译系统是由 python 实现的，实现的也很粗糙，在 rust 项目中总感觉怪怪的。

因此重新设计了一个更加统一，有 rust 实现的编译系统，借助 build.rs 和 xtask，感觉项目更加纯洁了 😊

我们的目标是，每个子模块都可以通过 `cargo build` 编译，同时可以通过 `cargo xtask` 编译整个项目，支持他们的是新增的 rel4_config 模块，通过该模块获取 platform 相关的配置。

因此每个子模块的 build.rs 需要处理和自己相关的代码生成，linker 参数设置等工作。xtask 不参与子模块的代码生成和编译过程，只负责根据编译命令增加 feature，设置环境变量等工作。基本可以理解为之前 build.py 的代替。

## 2. 使用说明

基本上使用和之前 build.py 差不多，可以使用 -h 查看帮助

```
cargo xtask build -h

Usage: xtask build [OPTIONS]

Options:
  -p, --platform <PLATFORM>  [default: spike]
  -m, --mcs <MCS>            [default: false]
  -s, --smc <SMC>            [default: false]
      --nofastpath
      --arm-pcnt
      --arm-ptmr
      --rust-only
  -B, --bin
  -h, --help                 Print help

```
- 结合 seL4 project 编译

```
# 还可以增加其他相关编译选项，mcs, smc 等
# rel4 library 模式
cargo xtask build -p spike

# rel4 binary 模式
cargo xtask build -p spike --bin

# 和之前一样进入 build 运行 simulate
cd build
./simulate
```

- 只编译 rel4_kernel, 生成 kernel.elf

```
# 还可以增加其他相关编译选项
cargo xtask build -p spike --rust-only
```

## 3. 设计说明

### 3.1 rel4_config 模块

rel4_config 模块主要负责获取 platform 相关的配置，包括内存布局，linker scripts，汇编代码等。这些配置是在编译时就需要的，而不是运行时需要的。

同时提供代码生成的 api 供 build.rs 和 xtask 使用。

因此，我们将 platform 相关的 yml 配置文件放在 rel4_config/cfg 下，每个平台一个文件，文件名和平台名一致。例如 spike.yml

这样做的好处是，build.rs 和 xtask 不用重复实现配置获取和基础代码生成功能。如果后续增加或者修改配置相关设计，也更加统一。

目前 rel4_config 模块只包括 platform 相关的配置，后面根据需要可以增加更多配置。

> 可以把一些宏定义也加到 rel4_config 中

### 3.2 build.rs

#### 3.2.1 kernel/build.rs

kernel build.rs 中生成 kernel 编译所需的 asm 文件，linker scripts，这些生成的文件会放到 OUT_DIR 中，在 src 中引用。linker scripts 则会在 build.rs 中被设置为使用的链接脚本。

```
# rel4_kernel/kernel/src/arch/mod.rs

#[cfg(feature = "BUILD_BINARY")]
core::arch::global_asm!(include_str!(concat!(env!("OUT_DIR"), "/head.S")));
#[cfg(feature = "BUILD_BINARY")]
core::arch::global_asm!(include_str!(concat!(env!("OUT_DIR"),"/traps.S")));
```

### 3.2.2 sel4_common/build.rs

sel4_common 中目前已经有 build.rs 文件，用于解析 pbf 文件。在其中加上生成 platform_gen.rs。同样生成的 platform_gen.rs 放到 OUT_DIR 中，在 src 中引用。

```
# rel4_kernel/sel4_common/src/platform/mod.rs

#[cfg(feature = "BUILD_BINARY")]
include!(concat!(env!("OUT_DIR"), "/platform_gen.rs"));
```

### 3.3 xtask

完成上述两部分后，其实已经可以使用 cargo 编译 kernel 了。但是在 build.rs 中是无法加上 feature 等参数的，因此命令行需要加上太多的参数，比如

```
 MARCOS="-DPLATFOMR=spike -DCONFIG_FASTPATH=ON -DCONFIG_KERNEL_MCS=ON" PLATFORM="spike" cargo build --target=riscv64imac-unknown-none-elf --release --bin rel4_kernel --features BUILD_BINARY --features KERNEL_MCS
```

可以感受一下，这确实太长了，因此我们需要一个 xtask 来帮助我们完成这些工作。

xtask 本质上和 build.py 功能一样，目前支持的参数如下。

```
Usage: xtask build [OPTIONS]

Options:
  -p, --platform <PLATFORM>  [default: spike]
  -m, --mcs <MCS>            [default: false]
  -s, --smc <SMC>            [default: false]
      --nofastpath
      --arm-pcnt
      --arm-ptmr
      --rust-only
  -B, --bin
  -h, --help                 Print help
```

xtask 会根据参数设置环境变量，增加 feature，然后调用 cargo build 编译。

同时 xtask 会调用 cmake 命令，可以和之前一样，配合 sel4test 等 seL4 project 使用。 `cargo xtask build` 不仅生成 kernel.elf，还会生成 simulate 运行测例。同时还支持 rel4 lib binary 两种模式。基本上完全实现了 build.py 的功能。
