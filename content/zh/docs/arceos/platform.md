---
title: "Platform"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## 1. 简介

arceos 中将硬件平台相关的实现都集中到 axplat 中，提供接口给 arceos 模块使用，这样增加硬件平台不用修改 arceos 主仓库。

我们可以将 seL4 系统抽象成一个硬件平台，实现 axplat-aarch64-sel4 crate，实现 bsp 相关功能。

需要实现的部分，axplat 已经定义了一个框架，主要是包括

- 内存管理
- 中断管理
- 时钟
- console
- power
- 多核实现

内存和中断实现参考上述段落，其他实现参考其他平台实现即可。

## 2. 背景

Arceos 实现了 [crate_interface](https://crates.io/crates/crate_interface) crate，方便各个模块间解耦。使用方只需要引入定义接口的 crate，无需引入实际实现接口的 crate。

## 3. 实现

内存和中断在 [memory](./memory.md) 和 [irq](./irq.md) 中分别介绍。时钟、console、power 的实现也很简单，主要是介绍对多核的支持。

### 3.1 多核系统实现

多核移植工作主要分为两部分

1. 从核心启动初始化
2. 任务在从核心上的调度，以及不同核心之间的迁移

接收到从核心启动命令后，会在目标核心上创建一个任务，当作核心启动，执行 Arceos 中 rust_main_secondary 函数。

```
// 主核心初始化时，会给每个从核心创建一个初始 seL4 任务
// 创建过程中，会把从核心的 tcb 放到 0x80 + cpu_id 能力槽，方便后续调用
#[cfg(feature = "smp")]
pub(crate) fn init_secondary_task() {
    for i in 1..CPU_NUM {
        let stack_top = unsafe { SECONDARY_BOOT_STACK[i - 1].as_ptr_range().end as usize };

        let entry = _start_secondary as usize;
        let _ = InitTask::new(entry, stack_top, i).unwrap();
    }
}

// 主核心在启动完成后，会调用从核心 cpu_boot 函数，启动每个核心上的初始任务
#[cfg(feature = "smp")]
fn cpu_boot(cpu_id: usize, _stack: usize) {
    // create a sel4 task and set affinity
    LeafSlot::new(0x80 + cpu_id).cap().tcb_resume().unwrap();
}

// 初始任务的实现，调用 arceos 标准核心启动函数，这样后续就完全和 arceos 设计一样了
#[cfg(feature = "smp")]
#[sel4_thread_entry]
extern "C" fn _start_secondary(cpu_id: usize) -> ! {
    axplat::call_secondary_main(cpu_id)
}

```

从核心上的初始化任务完成初始化工作后，会自动变成管理人物，监听子任务的 IPC 请求。这样每个核心上都有一个任务队列，切换时由各自的管理任务进行处理，不同核心之前没有影响。

## 4. 讨论

