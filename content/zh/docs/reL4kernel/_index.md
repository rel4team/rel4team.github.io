---
weight: 2
bookCollapseSection: true
bookFlatSection: true
title: "ReL4 Kernel 设计"
---

## ReL4 Kernel

ReL4 Kernel 是 SeL4 Kernel 的 Rust 实现。通过一个个模块重写的方式，逐步将 SeL4 转换为 Rust 实现，目前基本已经完成所有模块的重写工作。

该工作主要分为以下几个部分

1. [启动代码和编译系统移植](./startup.md)

