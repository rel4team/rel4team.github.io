---
title: "Arceos 内存移植说明"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## 1. 简介

seL4 中所有内存都按照能力管理，当一个任务启动时，它其实是没有可以用于分配的内存空间，可以理解成完全静态的。如果想要使用内存，就需要找 kernel 从 untyped 区域分配 PAGE(4KB) LARGE_PAGE(2MB) 能力并进行页表映射，然后才能自由使用这些内存。

可以把该过程等同于操作系统中的内存映射然后使用的过程，所以内存的移植主要是将原先页表映射等操作改成通过 seL4 syscall 分配。

## 2. 背景

seL4 中内存资源也是能力，需要先申请，才能在应用中使用这段资源，具体介绍可看 [seL4 vspace](https://rel4team.github.io/zh/docs/beginners/vspace/)

## 3. 实现

目前只考虑了 unikernel 的实现，也就是在初始化时，直接将所有 untyped 区域申请为 LARGE_FRAME 能力，并进行映射，类似操作系统中 fixed offset map 映射方法，具体内容如下。


1. axruntime 中禁用 axmm::init_memory_management，其中设计很多页表等操作，实在无法剥离。因此我在 axhal 中单独写了一个内存初始化函数，放弃了 axmm 的功能。不过也可以修改 axmm 的后端，通过 axmm 操作 seL4 内存管理功能。

2. rel4-linux-kit 在创建进程时，给进程在固定位置 0x2000_0000 分配一块 2M 的大页。arceos 在启动时直接使用该段内存作为堆，进行初始化。这段内存初始化后仍然可以使用，作为堆空间的一部分。

3. rel4-linux-kit 启动时给进程分配一块最大的 untyped cap，arceos 启动后使用该段内存进行内存初始化等操作。目前基本上就是将 untyped cap 全部分配为 large_frame 并映射。为了保持地址的连续性，也就是大页和大页之前紧挨着，**这块 untyped cap 上只放大页**。其他所有的能力放置在另外一个较小的 untyped 区域上。也就是说，root-task 会给进程分配一块很大和一块较小的 untyped cap

4. 外设地址由 root-task 实现映射到指定虚拟地址，arceos 直接使用映射后的虚拟地址，不用使用物理地址了


### 3.1 MemoryManager 的设计

oskit 中提供了一个 MemoryManager，专门用于管理 seL4 任务中的内存空间。

```
/// 进程内存管理器
pub struct MemoryManager {
    // 任务的 VSpace 能力，按照根页表理解
    vspace: VSpace,
    // 专门用于申请页，页表等和内存相关能力的分配器
    obj_allocator: MemCapGlobalAllocator,
    // 虚拟页帧分配器，这个主要是给后续动态分配页能力准备的，比如 ipc buffer
    frame_allocator: Mutex<VirtFrameAllocator>,

    // 为了进行虚实地址转换，实现效率看起来是有点低的
    v2p_map: Mutex<BTreeMap<usize, AddrRange<usize>>>,
    p2v_map: Mutex<BTreeMap<usize, AddrRange<usize>>>,
}
```

该管理器提供以下 api


```
// 根据大小，按照大页分配，并映射到指定虚拟地址。比如 4M 大小就会自动分配 2 个大页，并映射
pub fn large_page_map_alloc(&self, start: VirtAddr, size: usize) -> sel4::Result<PhysAddr>

// 根据大小，按照页分配，并映射到指定虚拟地址。
pub fn map_alloc(&self, start: VirtAddr, size: usize) -> sel4::Result<PhysAddr> 

/// 虚拟地址转物理地址
pub fn virt_to_phys(&self, vaddr: VirtAddr) -> sel4::Result<PhysAddr>

/// 物理地址转虚拟地址
pub fn phys_to_virt(&self, paddr: PhysAddr) -> sel4::Result<VirtAddr>

/// 分配一个 IPC Buffer 页并进行映射
pub fn alloc_ipc_buffer(&self, allocator: Option<&mut CapSet>) -> sel4::Result<(VirtAddr, Granule)> 

/// 释放一个 IPC Buffer 页
pub fn dealloc_ipc_buffer(&self, vaddr: VirtAddr)

```

### 3.2 ArceOS 中的内存初始化

oskit 中将内存操作都封装好，在 ArceOS 中的操作简化很多

```
/// Initializes the memory space and sets up the global memory allocator.
pub(crate) fn init() {
    // root_task 会把专门给内存使用的 untyped 能力放到 24 号槽
    let vspace = init_thread::slot::VSPACE.cap();
    let untyped = sel4::Cap::from_bits(24);
    // 初始化 MemoryManager
    MEM_SPACE.init_once(MemoryManager::new(
        vspace,
        untyped,
        VIRT_FRAME_ADDR,
        VIRT_FRAME_SIZE,
    ));
    // 分配映射用于动态分配的内存区域，也是整个操作系统使用的区域
    MEM_SPACE
        .large_page_map_alloc(MEM_START_ADDR.into(), MEM_SIZE)
        .unwrap();
    // 由于 oskit 中基础组件的实现还依赖 alloc 特性，所以必须由 root_task 预先映射好一段内存，作为初始化时候的堆，当然初始化完成后继续作为堆的一部分使用
    let paddr = translate_addr(axconfig::plat::INIT_HEAP_BASE);
    MEM_SPACE.add_region(
        axconfig::plat::INIT_HEAP_BASE,
        paddr,
        axconfig::plat::INIT_HEAP_SIZE,
    );
}
```

可以看到，ArceOS 中之进行堆空间的内存初始化，这是因为其他内存空间都由其父进程初始化好了，我们可以将 arceos 中广义的内存空间分为以下四个部分

1. elf 文件中定义段，root-task 会将其映射并加载，并创建栈和 IPC Buffer ，是基本固定的部分，完全由 root-task 创建
2. 一个初始化时使用的堆空间，同样由 root-task 分配并映射到指定虚拟地址，供初始化时使用
3. 一个巨大的 untyped cap，直接给到 arceos，由自己维护。初始化将该段内存连续化映射后，交给 axalloc
4. 一个较小的 untyped cap，用来放内存相关之外的能力，比如 tcb, cnode 之类

## 4. 讨论

1. 目前 oskit 组件的实现依赖动态分配，导致 root_task 需要帮助 arceos 预先分配好一个堆空间，尝试去掉
2. MemoryManager 依赖锁，能否做一个无锁实现
3. 目前只考虑了 unikernel 的移植，如果扩展到宏内核，还需要更多设计
4. 目前完全 bypass 了 axmm 组件，但其实可以把 seL4 内存相关操作抽象为 页分配，创建页表操作，适配 axmm