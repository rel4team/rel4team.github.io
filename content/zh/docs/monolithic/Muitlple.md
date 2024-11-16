---
title: "多内核并存隔离实现"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 多内核并存隔离实现

我们期望能在用户态同时运行数个完整内核，其通过函数调用、IPC 等形式向基座 seL4 申请资源，或由 seL4 基座转发给驱动设备任务来进行处理。这些内核彼此应当是互相隔离的，通过 IPC 的方式进行交互。

为了实现这一点，我们应当理解 seL4/reL4 的内存分配模型。以 seL4 为例，root task 在完成初始化之后，会将 untyped_memory 交给用户程序使用，用户程序自身在完成任务的时候可以利用自身所分配到的 untyped_memory 来申请分配 capability。

因此我们设计了如下思路：

- 通过**硬编码**来确定需要运行什么任务，包括设备任务和内核任务，并手动硬编码他们所分配的 untyped memory
- 当内核获得了 root-task 所分配的 untyped_memory 之后，便可以在自身实现一个 object_allocator，负责管理自身以及派生子任务的 capability 申请分配，而不需要再次向 root-task 请求
- 对于宏内核所运行的用户程序以及派生出的其他任务，在加载时使用内核任务的 untyped memory 进行分配，如果有使用动态内存的需求，也是通过如 mmap 等系统调用的形式进行申请。上述申请均只会在对应内核任务完成处理，而不会经过其他内核任务或者 root-task，从而保证了不同内存管理的独立，并利用 capability 的特性保证了隔离
- 驱动等设备任务应当是**无状态**的，即他们只负责管理自身对应的设备并完成接收到的请求，而不应当存储与上层调用任务相关的状态，才能保证不同内核任务调用设备任务时的独立和透明性


相关实现和例子：TODO，待提交