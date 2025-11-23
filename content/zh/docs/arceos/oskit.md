---
title: "seL4 oskit crate 实现"
weight: 5
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## 1. 介绍

在移植 Arceos 的过程中，实现了一些 seL4 系统上特有的功能，比如中断管理、内存管理等。这些模块和 arceos 无关，同时可以用于后续其他 os 的移植，因此抽象出来为 crate。

抽象的 crate 为两个

- [oskit, seL4 底层实现, 对能力的封装](https://github.com/reL4team2/rel4-linux-kit/tree/croak/arceos_support2/crates/sel4-oskit)
- [interface, seL4 底层 API 接口，如内存分配，中断注册，任务切换](https://github.com/reL4team2/rel4-linux-kit/tree/croak/arceos_support2/crates/sel4-if)

## 2. 背景

得益于 seL4 中的 capability 设计，其安全性很高，但同样也增加了使用复杂度。如果我们能将各种成熟 OS 移植到其上，作为操作系统服务，那无疑会大大降低其使用难度，增加适配性。比如将 Linux 移植上去，那可以继承所有 Linux 生态，解决了 seL4 用户态开发困难的问题。

进行这种移植，就需要将 seL4 相关的底层操作，如中断注册、内存分配和映射、任务切换等功能抽象出来，供操作系统服务调用，尽量减少移植工作量。


## 3. 实现

### 3.1 seL4 接口

目前 crate 提供向上的接口有

`目前设计很粗糙，需要仔细考虑优化`

```
/// 任务接口
#[def_plat_interface]
pub trait Sel4TaskIf {
    /// Switches to the given seL4 task.
    ///
    /// It returns the previous task's ID.
    fn switch_task(prev_task: usize, next_task: usize) -> usize;

    /// Creates a new seL4 task with the given parameters.
    ///
    /// It returns the created task's ID.
    fn create_task(
        tid: usize,
        entry_point: usize,
        stack_top: usize,
        priority: usize,
        cpu_id: usize,
    ) -> usize;

    /// Destroys the seL4 task with the given ID.
    fn destroy_task(task_id: usize);

    /// Migrates the seL4 task with the given ID to the target CPU.
    fn migrate_task(task_id: usize, target_cpu_id: usize);

    /// Starts the seL4 task with the given ID.
    fn start_task(task_id: usize);

    /// Stops the seL4 task with the given ID.
    fn stop_task(task_id: usize);

    /// Checks if the current task is the initial task.
    fn is_init_task() -> bool;

    /// Get Current Sel4 Task ID
    fn sel4_task_id() -> usize;

    /// Set Current Sel4 Task ID
    fn set_sel4_task_id(tid: usize);
}

/// 事件处理接口
#[def_plat_interface]
pub trait Sel4EventIf {
    /// 事件处理函数
    fn handler(cpu_id: usize) -> !;
}

/// 中断相关接口
#[cfg(feature = "irq")]
#[def_plat_interface]
pub trait Sel4IrqIf {
    /// Disables IRQs.
    fn disable_irqs();

    /// Enables IRQs.
    fn enable_irqs();

    /// Checks if IRQs are enabled.
    fn irqs_enabled() -> bool;
}
```

### 3.2 oskit 的实现

oskit 主要包括以下部分实现

- 中断管理，包括注册和处理中断
- 内存管理，内存分配并映射，类似页表相关的操作
- 能力管理，能力分配回收功能，上层无需考虑回收，直接使用分配即可
- IPC 事件定义

### 4 讨论

1. kit 的实现依赖 alloc，导致上层操作系统服务在初始化时就必须分配好堆空间。计划是至少将内存管理功能一部分剥离，不依赖 alloc，在其初始化中分配堆空间。
2. 目前接口设计相当随意，用到什么加什么，需要仔细设计