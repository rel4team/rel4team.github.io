---
title: "宏内核设计思路和实现"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## 1. 宏内核设计目标
我们的目标是在 seL4 的用户态开发宏内核程序，从某种意义上说是在开发一个库操作系统（LibOS）。他们有如下特点：

1. 该库操作系统可以兼容现有的 Linux APP，并且不会有太大的性能损失（IPC 相比于原有的函数调用，降低了性能，提高了安全性）。

2. 希望能够在 seL4 上运行多个“用户态内核”，这些“内核”之间是相互隔离的，通过 IPC 进行通信。真正的底层微内核通过 root-task 将内存等资源分配给用户态内核，用户态内核再通过自己的 object_allocator 进行内存管理，而不需要再次向 root-task 请求。

## 2. 宏内核设计思路

在微内核角度下，我们将任务分为了三类，并在当前的项目目录下实现了对应的 demo。

- 内核任务：负责接收来自用户程序的 syscall，并且将其转发到对应的设备任务上；对应的 demo 为 [kernel-thread](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/tree/main/crates/kernel-thread)
- 用户程序任务：负责运行待测的用户态二进制文件，并将其调用的 syscall 转化为 IPC 和内核任务进行通信；对应的 demo 为 [test-thread](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/tree/main/crates/test-thread)。
- 设备任务：如网络任务等，通过网络栈与各类网络设备交互，并且向上提供接口；对应的 demo 为 [net-thread](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/tree/main/crates/net-thread)。

一个关键的问题是如何设计不同类型任务之间对应的接口，从而在保持组件通用性的同时能够适配微内核的 IPC 的特点（不适合传递过大信息）。对于用户程序任务和内核任务，已经有较为完善的 syscall 接口的定义。因此我们需要考虑的有以下两部分：

- **内核任务和设备任务**之间的通信接口的设计：适用于宏内核

- **用户程序任务和设备任务**之间的通信接口设计：适用于 Unikernel、将来可能存在的 bypass kernel module 等

为了能够更好地应用 ArceOS 等其他已有的组件，我们提出了如下的兼容思路：复用原先的设备组件，如网络设备、文件系统设备等，其对外提供的接口不变，为函数调用的形式。当我们希望能将其作为用户组件运行的时候，就**为其添加一个 IPC 模块**，负责接收外部的请求，并且进行设备函数的调用，将响应结果返回回去。因此通过决定是否使用 IPC 接管，可以方便地决定该模块是否在用户态运行。


为了能够以 Rust 语言来书写 seL4 的用户态程序，我们在 [rust-seL4](https://github.com/rel4team/rust-sel4) 上进行开发.，相关代码仓库详见：[root-task-dev](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/tree/main)。


## 2. rust-seL4 说明
- 代码仓库：https://github.com/seL4/rust-sel4

- 项目介绍：rust-seL4 库是 seL4 开发团队额外开发的一套接口库，将 seL4 对外暴露的接口转化为了 rust 语言的形式，从而方便开发者以 rust 语言书写在 seL4 上运行的用户程序。

- 迁移说明：我们的 reL4 项目使用 rust 语言对 seL4 进行了重写和升级，在理想状态下期望 reL4 和 seL4 对外暴露的接口一致，实现的功能一致。因此使用 reL4 作为底层内核实现，应当也是可以支持 rust-seL4 功能的正确运行，这也是我们选用 rust-seL4 作为开发工具的一个重要的依据。事实上，我们确实也达到了这个目的，已经证明 **reL4 + rust-seL4的组合可以正常工作**，相关commit详见[这里](https://github.com/Huzhiwen1208/rust-root-task-demo/commit/b3b8217b3542def10e20d263df0fb80cf20f3198)。

## 3. 设备任务设计实现

为了能够更好的应用 ArceOS 的组件，我们以 net-thread 为例，复用了 ArceOS 的 [smoltcp 网络栈](https://github.com/arceos-org/arceos/tree/main/modules/axnet/src/smoltcp_impl)，并添加了对应的 IPC 接口，从而能让其与内核任务进行通信。并且我们还定义了一套专门用于网络通信的 IPC 接口，当网络设备任务和内核任务都实现了对应的 IPC 接口之后，就可以进行通信。

为了保证该组件的复用性，我们希望该组件仅提供必要的功能，不包含额外的 IPC 或者兼容 posix 接口的实现。因此我们将 IPC 和真正 syscall 功能的实现从复用组件当中划分出来，详细的设计实现如下：

- common：

  - lib/NetRequsetabel：定义了网络部分相关的 IPC 接口，接口的定义是按照 Unikernel Smoltcp 网络栈向外暴露的接口进行设计的，默认第一个参数为 socket id 用于识别进行当前网络操作的套接字，其他的参数的意义可以参见：[接口构建](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/blob/docs/crates/common/src/lib.rs#L293)

- net-thread：

  - smoltcp/：绝大部份移植了 ArceOS 的 Smoltcp 网络栈，但由于简便起步暂时只移植了 TCP 部分，这部分的组件可以给其他内核复用
  - ipc/：实现了网络设备接收 IPC 的接口。**另外为了维护微内核的设备功能，我们将 socket 的状态维护放在了 IPC 模块中，而不是内核任务中，从而让内核任务仅仅是作为一个中转设备的功能，通过 socket id 来请求、修改 socket 状态。**

- kernel-thread：

  - syscall/net/ipc：在内核端实现的和网络设备通信的 IPC 接口，将对设备的函数调用转化为 IPC
  - syscall/net/net_impl：syscall 语义中与网络相关的实现，目前还较为简单

- test-thread：用户程序任务，通过 vsyscall handler 来调用 syscall 实现（vsyscall_handler 本身是为了应对 syscall 转发使用的，但是目前的 test-thread 是我们自己编写的程序，会在 kernel-thread 中被启动起来，因此调用 vsyscall_handler 就只是一个普通的 function call）

这里我们使用了 IPC 传递 capability 的形式来进行跨地址空间的内存访问，因此要求 IPC 提供的接口不能有超过 1 个的内存访问需求，否则需要通过直接获取对方任务指针的形式，将调用方的内存映射到被调用方的地址空间中进行读写，一个例子是[write]系统调用(https://github.com/Azure-stars/rust-root-task-demo-mi-dev/blob/main/crates/kernel-thread/src/syscall/fs/io.rs#L19)（TODO：事实上这个地方也可以通过直接使用 IPC 传递的 capability 来实现跨地址空间访问，但是当时还没来得及考虑与实现）。  

## 4. 多线程支持

对于宏内核而言，最为核心的一般都是进程管理机制的确定。它决定了宏内核将以什么形式去统筹、管理这些资源。我们计划复用 seL4/reL4 的任务调度实现（如调度算法等），只在用户态负责进行任务创建、调度和删除等操作。因此我们需要实现的是一个简单的线程库，用于管理用户态任务的创建、调度和删除等操作。

可以参考[clone syscall 实现](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/blob/main/crates/kernel-thread/src/syscall/thread/task.rs#L136) 了解基本的多线程管理。目前的管理方式还非常简单，但是基本的实现和思路已经确定，可以靠这个内容来实现一个基本的 shell 操作了。
> 似乎还存在一些隐式的 bug，比如当任务占用空间过大时会导致 panic 等问题，这些问题需要进一步的调试和解决。

## 5. 其他文档
相关的指导文档见[这里](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/releases/tag/doc)。