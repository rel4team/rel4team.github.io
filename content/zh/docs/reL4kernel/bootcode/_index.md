---
weight: 1
bookCollapseSection: false
title: "ReL4 编译和配置系统设计"
---

## ReL4 Kernel Boot Code

ReL4 Kernel 是 SeL4 Kernel 的 Rust 实现。通过一个个模块重写的方式，逐步将 SeL4 转换为 Rust 实现，目前基本已经完成所有模块的重写工作。

但是编译和配置系统，以及启动部分的代码还没有移植，编译仍然依赖 seL4 的 CMake 编译系统。导致引入了很多不必要的编译选项，增加了使用的复杂度。

因此希望将 reL4 升级成一个 Pure Rust 项目，不依赖 seL4 kernel 的任何代码和编译系统，只使用 rust 工具链就可以产生一个可用的 reL4 kernel。同时可以无缝接入原有的 seL4 project，例如 sel4test。

该工作主要分为以下几个部分

1. [riscv 启动代码移植](./riscv_boot.md)
2. [配置系统设计开发](./config_system.md)
3. [编译系统兼容](./cmake.md)

