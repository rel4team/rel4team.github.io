---
title: "Vspace 学习笔记"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## 1. 前言

在所有的 kernel 中，内存空间和管理都是非常重要的一部分。由于涉及到虚实地址转换，这部分相对来说难以理解一些。seL4 由于其独特的 capability 机制设计，vspace 设计更加难以理解一些，因此本文尝试详细解释下 vspace 设计。理解了 vspace 后，seL4 创建进程，内存分配等等过程就很容易理解了。

## 2. 总览

seL4 依赖以下四个能力 (内核对象) 进行内存管理

- asid_pool 全局只有一个，root-task 控制
- vspace, 每个任务一个，一级页表 (最高级的页表)
- page_table, 其他页表
- frame，页帧

基本上符合内存管理从上到下的组织结构。

## 3. vspace, page_table, frame

### 3.1 vspace

从名称上来看的话，其实 vspace 这个名称有点难以理解，其实它就是最高级页表。

```bf
block vspace_cap {
    field capVSMappedASID            16
    // 这个指针指向一级页表的内存地址
    field_high capVSBasePtr          48

    field capType                    5
    field capVSIsMapped              1
#ifdef CONFIG_ARM_SMMU
    field capVSMappedCB              8
    padding                          50
#else
    padding                          58
#endif
}
```

当你知道它就是一级页表后，就很好理解了。

vspace 本身占用 4K 空间，cap 数据结构中的 capVSBasePtr 指向这个空间的基地址

### 3.2 page_table

除了一级页表之外的页表，seL4 统一作为 page_table 管理。

因为都是页表，所以 page_table 和 vspace 区别不大，无需更多介绍。

### 3.3 frame

frame 对应具体的页帧，进程实际用到的内存就是这些页帧组成的。

值得注意的是，seL4 支持大页，以 aarch64 为例，frame 有以下三种大小

- Page          4KiB
- LargePage     2MiB
- HugePage      1GiB

frame cap 数据结构如下

```bf
block frame_cap {
    field capFMappedASID             16
    // 页帧基地址
    field_high capFBasePtr           48

    // 页帧类型，大小页
    field capType                    5
    // 页大小
    field capFSize                   2
    field_high capFMappedAddress     48
    field capFVMRights               2
    field capFIsDevice               1
    padding                          6
}
```

## 4 seL4 中的内存分配

### 4.1 page_table, frame 的创建

seL4 系统中所有的对象创建，本质上都是从 untyped 内存区域分配的。

> 功能上，你可以把 untyped_retype 理解为 new

下面以 page_table 创建为例，看下是如何创建一个对象，或者说能力

```c
// 用户态 API
LIBSEL4_INLINE seL4_Error
seL4_Untyped_Retype(
    // untyped cap，参考下 capability 的介绍
    seL4_Untyped _service, 
    // 创建对象的类型和用户态提供的对象大小
    seL4_Word type, seL4_Word size_bits, 
    // 指向创建对象的 cptr 所在 cslot
    seL4_CNode root, seL4_Word node_index, seL4_Word node_depth, 
    // 创建多个对象相关参数
    // 这里的多个对象必须是同一类型的，实际上在创建后，在内存区域上，也是连在一起的
    seL4_Word node_offset, seL4_Word num_objects)
{
    // syscall 过程就不看了，可参考 capability 这篇文章的介绍
}

```

```rust
// 内核态处理过程
pub fn invoke_untyped_retype(
    src_slot: &mut cte_t,
    reset: bool,
    retype_base: pptr_t,
    new_type: ObjectType,
    user_size: usize,
    dest_cnode: &mut cte_t,
    dest_offset: usize,
    dest_length: usize,
    device_mem: usize,
) -> exception_t {
    // 获取 untyped 区域的起始地址，untyped 区域就是一段目前还没有用到，没有对象占用的内存区域
    let region_base = cap::cap_untyped_cap(&src_slot.capability).get_capPtr() as usize;
    // 创建对象的大小，注意这里是指总大小，如果创建多个对象，那么是加起来的
    // get_object_size 会返回创建对象的大小，根据对象类型和用户态提供的大小产生，page_table 就是 4KiB
    let total_object_size = dest_length << new_type.get_object_size(user_size);
    let free_ref = retype_base + total_object_size;
    // 在 untyped 类型上做个标记，已经被占用了
    // 这个时候可以理解为，对象本身已经 new 完了，后面就是创建 cptr 的过程了
    cap::cap_untyped_cap(&src_slot.capability)
        .set_capFreeIndex(GET_FREE_INDEX(region_base, free_ref) as u64);
    // 本质是创建 cptr，这里是创建 cap_page_table, 并且加到 cspace 中，就不展开了
    create_new_objects(
        new_type,
        src_slot,
        dest_cnode,
        dest_offset,
        dest_length,
        retype_base,
        user_size,
        device_mem,
    );
    exception_t::EXCEPTION_NONE
}

pub fn arch_create_object() {
    ...
    // 这里创建 page_table cptr
    ObjectType::seL4_ARM_PageTableObject => {
        cap_page_table_cap::new(ASID_INVALID as u64, region_base as u64, 0, 0).unsplay()
    }
}
```

从上面过程可以看到，seL4 的内存分配过程很粗糙，基本就是按照顺序划一块区域作为对象区域。我不知道回收机制是啥样的，但是这么粗糙的方式真的可以有效利用内存空间吗。

还有一点是，untyped cap 中存储的内存地址是虚拟地址，但是由于 kernel 是 fixed offset map，虚实地址之间转换很简单，所以可以按照物理地址理解，untyped cap 就可以看作是 memory allocator


### 4.2 vspace 创建和进程内存初始化

vspace 本身就是个一级页表，创建过程和 page_table 是一样的，没啥特别说的

我们主要关注创建 vspace 之后的操作，重点是在 invoke_asid_pool 操作中，可以看到 copyGlobalMappings 操作，也就是将内核 PTE 都复制到新任务的 vspace 中

当然，上述流程只在 riscv 中，因为 riscv 只有一个 root pte

从上述操作中我们可以看到，seL4 是一个单页表设计，也就是内核空间和用户空间共用一个页表


### 4.3 进程数据加载的过程

我们先回忆下宏内核加载进程的常用设计。

创建一个新的地址空间 (可能是个结构体，包含使用的物理页帧，页表)，通过文件系统读取并解析 elf 文件。根据 elf 定义的地址范围，申请对应的物理页帧，并创建页表存储 pte，实现虚实地址 mapping

所有申请的物理页帧和创建的页表 (页表本身也是存在物理内存上的) 都一起放在地址空间中。最后将 root pte 写入寄存器，进程加载基本就完成了。

seL4 中其实也是这个流程，但是这所有的过程都需要用户态自己实现。kernel 只负责创建页帧，页表等等，至于怎么组织，如何创建，整个过程由用户态自己实现。

我们以 rel4-linux-kit 中 root-task 创建任务为例

```rust
pub(crate) fn make_child_vspace<'a>(
    cnode: sel4::cap::CNode,
    mapped_page: &mut BTreeMap<usize, PhysPage>,
    image: &'a impl Object<'a>,
    asid_pool: sel4::cap::AsidPool,
) -> (sel4::cap::VSpace, usize, SmallPage) {
    ...
    // 创建页表的过程，这个过程中，只 map 到页表这一级。
    // 也就是说，最后一级页表里面内容是空的，没有指向的物理页帧
    map_intermediate_translation_tables(
        allocator,
        child_vspace,
        image_footprint.start..(image_footprint.end + PAGE_SIZE),
    );

    // 创建页帧，map 到上面创建的页表中，将 elf data 拷贝到创建的页帧中
    map_image(
        allocator,
        mapped_page,
        child_vspace,
        image_footprint.clone(),
        image,
    );
}

// 创建所有页表
pub fn map_intermediate_translation_tables(
    allocator: &ObjectAllocator,
    vspace: sel4::cap::VSpace,
    // 这个是进程 elf 中描述用到的地址范围。
    // 这里没有分段，而是用了一个范围，稍微有点粗糙
    footprint: Range<usize>,
) {
    // 从上到下，生成每一级的页表，但是最后一级页表内容是空的
    for level in 1..sel4::vspace_levels::NUM_LEVELS {
        let span_bytes = 1 << sel4::vspace_levels::span_bits(level);
        let footprint_at_level = coarsen_footprint(&footprint, span_bytes);
        for i in 0..(footprint_at_level.len() / span_bytes) {
            let ty = sel4::TranslationTableObjectType::from_level(level).unwrap();
            let addr = footprint_at_level.start + i * span_bytes;
            allocator
                // untyped_retype cap, new page_table object
                .allocate_and_retype(ty.blueprint())
                .cast::<sel4::cap_type::UnspecifiedIntermediateTranslationTable>()
                // page_table cap, seL4_ARM_PageTable_Map, 实现到最有一级页表的映射关系
                .generic_intermediate_translation_table_map(
                    ty,
                    vspace,
                    addr,
                    sel4::VmAttributes::default(),
                )
                .unwrap()
        }
    }
}

// 创建页帧并 map
pub fn map_image<'a>(
    allocator: &ObjectAllocator,
    mapped_page: &mut BTreeMap<usize, PhysPage>,
    vspace: sel4::cap::VSpace,
    footprint: Range<usize>,
    image: &'a impl Object<'a>,
) {
    let num_pages = footprint.len() / PAGE_SIZE;
    // 创建任务加载所需的所有页帧，Granule 是 rust-sel4 命名，就是 Page
    let mut pages = (0..num_pages)
        .map(|_| (allocator.alloc_page(), sel4::CapRightsBuilder::all()))
        .collect::<Vec<(sel4::cap::Granule, sel4::CapRightsBuilder)>>();

    // 将数据拷贝到这些页帧，过程大概是，先将这个页帧 map 到 root-task 地址空间中，然后拷贝，再 unmap
    ...
    // 将页帧 map 到子任务地址空间中
}

```

上述就是一个 seL4 的任务创建的过程，可以看出，sel4 kernel 尽量将功能都踢到用户态,root-task 本质上是对于 kernel 功能的补充

### 4.4 内存动态分配

任务启动后，动态分配内存的过程，其实和上述 root-task 加载进程数据的过程一样。

有一点区别就是，无需直接创建所有页表，而是只在寻址报错时，才需要创建页表，如下实现

```rust
    /// 映射一个页表 [sel4::cap::Granule] 到指定的虚拟地址
    pub fn map_page(&mut self, vaddr: usize, page: PhysPage) {
        assert_eq!(vaddr % PAGE_SIZE, 0);
        for _ in 0..sel4::vspace_levels::NUM_LEVELS {
            let res: core::result::Result<(), sel4::Error> = page.cap().frame_map(
                self.vspace,
                vaddr as _,
                CapRights::all(),
                VMAttributes::DEFAULT,
            );
            match res {
                Ok(_) => {
                    self.mapped_page.insert(vaddr, page);
                    return;
                }
                // Map page tbale if the fault is Error::FailedLookup
                // (It's indicates that here was not a page table).
                Err(Error::FailedLookup) => {
                    let pt_cap = OBJ_ALLOCATOR.alloc_pt();
                    pt_cap
                        .pt_map(self.vspace, vaddr, VMAttributes::DEFAULT)
                        .unwrap();
                    self.mapped_pt.lock().push(pt_cap);
                }
                _ => res.unwrap(),
            }
        }
        unreachable!("Failed to map page!")
    }
```