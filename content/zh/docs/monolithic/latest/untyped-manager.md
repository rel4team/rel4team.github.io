---
weight: 10
bookCollapseSection: false
title: "内存管理"
commentsId: 1 
---

# Cap 方式内存管理

要说 Cap 管理就不得不说 [Slot管理]({{< ref "/docs/monolithic/latest/slot-manager.md" >}}) 

## sel4 的内存管理方式

在 sel4 中内存，分为 Untyped Capability 和 Typed Capability, Typed Capability 就是那些已经被分配了内存并且转换为特定类型的能力，包括 Frame, PageTable, VSpace, CNode, TCB, Endpoint, Notification 等。

所有的 Typed Capability 都是从 Untyped Capability 中派生出的，所以 Typed Capability 就需要从 Untyped Capability 中获取内存，但是在删除 Typed Capability 后内存并不会直接回到 Untyped Capability，需要在 Untyped
Capability 中调用 revoke 删除从 Untyped Capability 中派生的所有 Capability 并回收内存。这就产生了一个问题，没有办法对小块的内存进行快速的回收和重利用，所以 rel4-linux-kit 中使用了一些机制来进行简单的内存管理，能够回收部分内存并重新利用。

主要包含以下机制：

- 多级内存管理，为每一个进程单独提供一个或多个中等大小的内存块，进程申请内存从这个中等大小的内存块中申请
- 页表回收利用，对于 sel4 中的任务来说，主要需要的 Capability 是 Frame, 所以我们会在 Frame 将要被释放的时候并不直接删除，而是清空后放置在一个特殊的地方，等待再申请的时候不再从 Untyped Capability 中取得，而是直接利用之前回收的 Frame
- 不同级别拥有不同的内存管理方式，RootTask 拥有所有的 Capability，kernel-thread 拥有一块比较大的 Untyped Capability，在需要分配内存的时候 kernel-thread 首先自己处理，当其内存不足的时候会向 RootTask 继续申请内存。

## 多级内存管理

由于 sel4 回收从 Untyped Capability 中派生的对象的内存，必须在 Untyped Capability 上执行 revoke 删除所有的 Capability，所以就需要对内存进行切分，提高内存回收的频率，减少内存回收影响的 Capability。因此 rel4-linux-kit 使用了多级内存的机制，在 rel4-linux-kit 定一个一个名为 `CapMemSet` 的结构体，这个结构体会在各个场景中包含，比如进程的 PCB 结构中也会包含一个 `CapMemSet`，在进程需要申请页表或其他需要用的 Capability 的时候会在 `CapMemSet` 中申请，如果 `CapMemSet` 中的 Untyped Capability 内存不足，则会向 kernel-thread 中申请另一个中等大小的 Untyped Capability 去应对后面的申请。

在需要对释放页表的时候会对页表进行

正如上面提到了，`CapMemSet` 结构体会伴随在多种使用场景中，以 PCB 为例，在进程生命周期结束后，

## 页表回收利用

由于 sel4 删除 cap 的时候并不会将删除的内存回到 Untyped Cap，所以在频繁的申请和释放内存的时候内存不断被消耗，最后导致内存不够用，而大部分的内存都是被使用在 Frame Cap 中，所以就需要收集并重新利用消耗的 Frame Cap。在 rel4-linux-kit 中，在释放 Frame Cap 的时候将释放的 Frame 进行收集，再次申请的时候就可以将收集的 Frame 清空后重新利用。


## 需要优化的地方

1. 尝试在目前的这套机制下支持 COW(Copy On Write)

## 对比连续映射的优点

1. 连续映射所有的 cap 都在内核中存有一套备份，但是对于如果要实现 libOS 的场景下，如何保证单个线程能够获得足够多的内存，并且能够在 libos 中完成操作，而不用真正的进入到最强大的内核当中处理无法处理的环境？

