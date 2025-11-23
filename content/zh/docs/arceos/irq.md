---
title: "ArceOS 中断移植开发"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## 1. 简介

将 arceos 移植到 seL4 系统上运行时，其中断功能需要重新设计。主要由于中断触发方式发生了重大变化，不再是之前硬件直接进入 trap ，而是通过 seL4 kernel 发送通知告知 arceos 当前有中断要处理。

因此主要改动在 **中断注册**, **中断 ack** 这两个部分，中断处理还是和之前一样。

## 2. 背景知识

seL4 是一个标准的微内核设计，当发生中断时，除了特定和 SMP 相关的两个 IPI 中断，其他中断均不处理。

用户态任务如果想要接入中断，必须先注册该中断，seL4 才会在发生中断时发送 notification 给对应任务。

## 3. 实现

### 3.1 中断管理对象

如前所说，实现了一套 oskit，将 os 移植需要的组件独立出来，不仅可以用于 arceos 移植，也许可以用于其他 os 的移植。

在 oskit 中，实现了一个 lock-free 的 IrqManager

```
/// 中断管理模块
/// 提供中断注册和注销功能
/// 这是一个 lock-free 的实现
pub struct IrqManager<'a, const N: usize> {
    // 用于注册的 notification 能力，中断就是通过 notification 通知 arceos
    global_notify: Notification,
    // 每个 core 上有一个独立的 IrqManager
    cpu_id: usize,
    // 中断注册时需要分配能力，所以提供一个 allocator 的引用
    obj_allocator: &'a ObjectAllocator,
    // lock-free 设计的核心，通过定长数组而不是 map 的形式储存 irq_handler cap，可以通过 irq 号找到对应的 irq 能力
    irq_caps: [AtomicUsize; N],
}
```

### 3.2 arceos 中适配

由于 aarch64 架构中存在 PPI 和 SPI，因此每个核心上都需要一个中断管理器。在 arceos 中可以直接使用 percpu 组件，即可实现每个核心都有一个中断管理器负责中断相关操作。

```
// 中断 handler table，根据 irq number 找到对应 handler，是 arceos 原有设计
#[percpu::def_percpu]
static IRQ_HANDLER_TABLE: HandlerTable<MAX_IRQ_COUNT> = HandlerTable::new();

// seL4 中断管理对象，用于 seL4 中断能力管理
#[percpu::def_percpu]
static IRQ_CAPS: LazyInit<IrqManager<MAX_IRQ_COUNT>> = LazyInit::new();

// 只能通过一个 bool 值模拟中断打开关闭
#[percpu::def_percpu]
static IRQ_ENABLED: bool = false;

```

在中断注册，处理等操作时，会先做 seL4 中断相关操作，然后会按照 arceos 原有中断框架进行注册，处理。

### 3.3 中断注册

中断注册 seL4 相关操作在 oskit 中实现如下

```
/// 注册一个中断号，并返回对应的通知对象
pub fn register_irq(&self, irq: usize) -> sel4::Result<bool> {
    // seL4 中对于 irq idx 是这么计算的 
    let idx = self.cpu_id * PPI_NUM + irq;
    // 从任务的通用 notification 中 mint 出一个引用，加上 irq index 的 badge，这样在接收到消息时可以通过 badge 知道中断号
    let notify_slot = alloc_slot();
    LeafSlot::from_cap(self.global_notify).mint_to(
        notify_slot,
        sel4::CapRights::all(),
        irq as _,
    )?;

    // 通过 root_task 获取 irq_handler 能力，这是因为只有 root_task 有权限调用中断注册 syscall
    let irq_slot = alloc_slot();
    register_irq(idx as _, irq_slot);

    // 将 notification 绑到 irq_handler 中，这样 kernel 会通过加了 badge 的 notification 通知该中断
    irq_slot
        .cap()
        .irq_handler_set_notification(notify_slot.cap())?;
    irq_slot.cap().irq_handler_ack()?;

    // 将 irq cap slot 的序号存到原子变量中，lock-free 实现的核心
    Ok(self.irq_caps[idx]
        .compare_exchange(0, irq_slot.raw(), Ordering::Acquire, Ordering::Relaxed)
        .is_ok())
}

```

在 arceos 中，完成 seL4 操作后，需要将中断处理函数注册到 IRQ_HANDLER_TABLE 即可

### 3.4 中断触发

注册中断后，当中断发生时，seL4 kernel 会通过 notification 告知用户态任务，因此用户态任务需要监听等待。在设计中，每个核心本来就有一个任务作为 event_handler，专门用于监听其他子任务向其发送的 IPC 调用，根据 seL4 的设计，在等待 IPC (本质是 recv endpoint) 时，kernel 发送 notification 也会触发 recv。利用这个机制，只要在原有的 event_handler 加上对 notification 的处理即可，几乎没有额外的消耗。

```
with_ipc_buffer_mut(|ib| {
    loop {
        let (msg, _badge) = DEFAULT_SERVE_EP.recv(());
        let msg_label = match ServiceEvent::try_from(msg.label()) {
            Ok(x) => x,
            // 当发现 label 不属于定义的事件 ID 时，认为是中断通知
            Err(_) => {
                // badge 就是中断号，交给中断处理函数
                #[cfg(feature = "irq")]
                handle_irq(_badge as _);

                continue;
            }
        };
```

### 3.5 中断处理

首先执行相应的中断处理函数，然后发送 ack 给 kernel，否则 kernel 不会响应下一个中断了。

在执行前会判断当前中断使能状态，如果是 disable，就不继续执行了，模仿中断 disable 的情况。


## 4. 讨论

### 4.1 能否做到跨 CPU 架构

因为都是通过 seL4 转发中断，应该是可以做到跨平台的。唯一有区别的就是，seL4 在多核情况时，对于中断号计算方式可能有点不同。比如 aarch64 有 PPI 和 SPI，中断号计算方式是 PPI_NUM * CORE_ID + IRQ. 当然这个应该是可以做兼容的，后续移植 riscv 时进一步探索。 

### 4.2 中断 disable 和 enable 的方式

目前是通过一个 flag 来决定是否执行中断处理函数来模拟中断使能状态，但这个设计似乎存在问题。比如 disable 后是否还 ack 告知 seL4 kernel, 如果 ack，还是会被一直打断，如果不 ack 呢，后续中断无法触发。

## 5. 待做项

1. aarch64 架构上，SPI 中断注册管理
