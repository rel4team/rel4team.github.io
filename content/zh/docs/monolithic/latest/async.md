---
weight: 10
bookCollapseSection: false
title: "Async kernel-thread"
commentsId: 1 
---

# 异步的 kernel-thread

对于 monolithic 程序来说，每一个用户线程都应该有一个内核线程，但是在 rel4-linux-kit 中kernel-thread和 LinuxApp 都在用户态，如果还是按照一个用户线程一个内核线程的对应方式，每启动一个 rel4-linux-kit 就需要启动一个 kernel-thread，这对于系统来说是一个很大的负担，所以就可以采取一些更加简单的方式来进行处理。

## 可用的选择

如何在一个线程中使用多条执行流来对 LinuxApp 的线程，可以采用比较合适的方式主要有以下两种方案：

1. 使用模拟线程机制来在一个线程中启动多个虚拟线程，以虚拟线程对应 LinuxApp 的线程，但是这种方案需要首先虚拟线程的相关函数和机制
2. 使用协程机制来启动多个协程，以协程来对应 LinuxApp 的线程，这种方案在 rust 中已经有很好的支持，并不需要手写太多的东西

## 事件驱动的协程

在 kernel-thread 启动完毕后，就会开始等待事件的到来，所等待的事件就是在 EndPoint 上等待事件的传递，由于 sel4 支持将 Notification 绑定在 TCB 上，当线程在 wait 的时候发生中断或其他消息，就会在将消息传递给用户程序，而用户程序在接受的时候会同时接受两个返回值，一个是 badge，一个是 MessageInfo。MessageInfo 用来包含 EndPoint 中传递的大小和 Cap 数量等消息，而 badge 就是分辨是否是 notification 还是 Endpoint, 以及是那一个 Endpoint 的消息。一般情况下，在 Notification 中会绑定不同的 badge 来区分。
