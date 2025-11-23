---
weight: 15
bookCollapseSection: false
bookFlatSection: true
title: "Arceos 移植"
---

## 1. 简介

最近一直在尝试将 [arceos](https://github.com/arceos-org/arceos) unikernel 移植到 seL4 系统上，本质是将 arceos 作为 seL4 的一个应用，在其上运行。

需要移植的内容主要包括以下内容

- [axplat](./platform.md)
- [中断和异常处理](./irq.md)
- [任务和调度](./task.md)
- [内存管理](./memory.md)

移植到 seL4 上的主要目的是利用 seL4 的安全性，通过 capability 机制分配所有资源。

在 sel4 上运行的时候，默认启用以下 arceos 特性

```
onsel4 = ["axhal/onsel4", "axruntime/onsel4", "axtask/onsel4", "multitask", "alloc", "page-alloc-4g", "tls"]
```

## 2. 讨论

1. 目前调度效率很低，涉及到多次 syscall 操作，希望能找到某种方法提高调度效率。
2. 打开抢占调度后，偶尔会出现运行卡住的情况，也许和调度以及中断是有关的，因为 seL4 kernel 中也是会在 timer 中执行抢占调度的，timer 本身也是中断，这中间是存在一些冲突的。
3. 支持 ArceOS 的宏内核 starry。
4. ArceOS 按照功能分成多个进程级操作系统服务。