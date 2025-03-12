---
weight: 6
bookFlatSection: true
title: "reL4 宏内核用户态程序设计"
commentsId: 1 
---

# 在 sel4 上运行宏内核

seL4 作为完成了形式化验证的内核，其安全性和可靠性得到了一定程度上的保证。在近年来安全问题频发的场景下，用 seL4 构建适配更多场景的内核的需求也逐渐强烈起来。目前该方面已有一定的工作成果，本文仅介绍其中两个特定方向：

- 以 seL4 作为 hypervisor，在其上构建 guest OS：[Virtualization on seL4](https://docs.sel4.systems/projects/virtualization/) 中描述了 seL4 为了支持虚拟化而封装的库。在这些库的作用下，可以完成基本的虚拟化支持。但目前的虚拟化基本仍然是以硬编码来确定 guestOS 的种类与数量的，并不十分灵活。

- 以 seL4 为底层基座，在其上构建异构的内核：[seL4 as Hypervisor](https://www.iaik.tugraz.at/wp-content/uploads/teaching/master-thesis/system_sel4_linux.pdf) 描述了这类应用场景，他们计划将 Linux 等内核作为任务运行在用户态，并将驱动等模块分离出来作为其他辅助任务运行。目前该文档中仅描述了运行单个内核的情况，但是理论上可以同时运行多个内核，适配更多场景。



在前述工作的基础上，我们结合组件化的背景，希望完成如下任务：

1. 在 seL4 上以用户程序的形式构建出**多个**宏内核，他们彼此之间互相隔离，通过 IPC 进行通信。
2. 兼容现有的 Linux APP，并且不会有太大的性能损失（IPC 相比于原有的函数调用，降低了性能，提高了安全性）
3. 利用组件化设计的思想，尽可能复用组件化内核 [ArceOS](https://github.com/arceos-org/arceos)  和其宏内核形态扩展 [Starry](https://github.com/Starry-OS/Starry) 的组件内容，从而进一步展现组件复用的可行性，反过来推进组件化内核的接口合理设计和升级
