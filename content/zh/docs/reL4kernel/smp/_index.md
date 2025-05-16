---
weight: 10
bookCollapseSection: true
title: "ReL4 SMP 移植说明"
---

## ReL4 Kernel SMP 移植说明

ReL4 Kernel 中实现了 seL4 中 SMP 设计，功能和 seL4 完全相同。

### SMP 测例运行

使用 qemu 运行 reL4 smp 测例相当简单，如下

首先参考 [sel4_test](../../quick_start/sel4_test.md) 搭建环境。

然后编译并运行 smp 测例

```
cd rel4_kernel

# riscv
cargo xtask build -p spike -N 4 --bin 

# aarch64
cargo xtask build -p qemu-arm-virt -N 4 --bin 

cd build && ./simulate --cpu-num 4

```

### 移植说明文档

1. [移植的 smp 相关代码](./smp_code.md)
2. [移植过程遇到的问题和解决思路](./develop_log.md)


### TODOs

1. [ ] MCS SMP 耦合部分移植
2. [ ] aarch64 smp 测例运行时间比 seL4 时间长
