---
title: "seL4 capability 学习记录"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---


## 1. 前言

seL4 中最大的特色设计是 capability 机制。seL4 文档以及一些其他文章已经详细介绍了 capability 的设计概念，但是似乎没有基于实际例子的入门介绍文档。

因此本文计划深入内核态和用户态的相关源码，基于一些实际使用例子来介绍 capability 机制，作为我的学习笔记，对于初学者来说也更容易理解。


## 2. Capability 机制简介

对于 seL4 capability 机制的介绍已经很多，用一句话来解释该机制

> 将内核中所有实力抽象为能力，并且将这些能力存储到 **能力空间 (cspace)** 中，kernel 根据 index 寻找对应的能力，根据能力找到对应实例的地址

可以从一下文档中加深理解，本文就不再赘述 capability 的设计和相关概念

- [官方文档](https://docs.sel4.systems/Tutorials/capabilities.html)
- [中文介绍博客 1](https://juejin.cn/post/7342330487077961767)
- [中文介绍博客 2](https://rel4team.github.io/zh/docs/async/Module-Interface/1.-Kernel-Object-Capabiltiy/)


## 3. Capability 如何使用

相比于构建 cspace，使用 capability 可能更广泛和容易理解，所以我们首先介绍下 capability 是如何生效的。

> 1. 所有内核对象都是 capability
> 2. 所有的 syscall 都是对一个 capability 进行某些操作
> 3. 每种 capability 可以看成一个内核服务，内核根据 syscall 中请求的 capability 使用对应的 handler 处理 syscall
> 4. cptr 和实际内核对象之间的关系，可以按照 C++ 中的智能指针来理解
> 5. 对于用户态来说，cslot 本质上是一个 index，cnode 理解成一个数组，cptr 时存储在其中元素。cptr 用户态是无法获取的，只能通过 index 告诉内核你想访问的 cptr

和 Linux, QNX 的 syscall 不同，seL4 的 syscall 应该理解为面对对象或者服务，而 Linux 的 syscall 则更像是一个个独立函数。

而 Capability 就是内核对象或服务，如 reL4 中实现的 syscall 处理函数

```rust
// 用户态 syscall
// static inline void riscv_sys_send_recv

    // a0 寄存器存放 dest cptr，就是上面提到的请求的内核服务
    register seL4_Word destptr asm("a0") = dest;
    // Message tag，具体可参考 seL4 IPCBuffer 概念
    register seL4_Word info asm("a1") = info_arg;

    /* Load beginning of the message into registers. */
    // 把尽量多的 msg 放到寄存器中，减少拷贝，msg 也就是需要传递的消息
    register seL4_Word msg0 asm("a2") = *in_out_mr0;
    register seL4_Word msg1 asm("a3") = *in_out_mr1;
    register seL4_Word msg2 asm("a4") = *in_out_mr2;
    register seL4_Word msg3 asm("a5") = *in_out_mr3;
    // syscall 类型，seL4 的 syscall 类型非常少，基本上都是根据动作来定义的，比如，call,send,recv 这种。而不是像 Linux 一样根据调用函数功能来，比如 write，poll 这种
    register seL4_Word scno asm("a7") = sys;

// 内核态处理函数
pub fn handle_invocation(isCall: bool, isBlocking: bool) -> exception_t {
    ......
    // capability cptr 放在寄存器 a0，根据 cptr 从 cspace 中回去对应的 cslot
    let cptr = thread.tcbArch.get_register(ArchReg::Cap);
    let lu_ret = thread.lookup_slot(cptr);

    // 从 cslot 中获取 cptr
    let capability = unsafe { (*(lu_ret.slot)).capability.clone() };
    // 调用 decode_invocation，在其中会根据 capability 分别调用不同的处理函数
    let status = decode_invocation(
        info.get_message_label(),
        length,
        unsafe { &mut *lu_ret.slot },
        &capability,
        cptr,
        isBlocking,
        isCall,
        buffer.unwrap(),
    );
    ......
}
```

我们以一个最简单的 syscall TCBSuspend 为例

```rust
// TCBSuspend 用户态函数
LIBSEL4_INLINE seL4_Error
seL4_TCB_Suspend(seL4_TCB _service)
{
	seL4_Error result;
	// 设置 msg tag 的 label 为 TCBSuspend
    // msg info 的解释  
    // The tag consists of four fields: the label, message length, number of
    // capabilities (the extraCaps field) and the capsUnwrapped field.
    seL4_MessageInfo_t tag = seL4_MessageInfo_new(TCBSuspend, 0, 0, 0);

    ...
    // 调用 syscall，获取 result
	output_tag = seL4_CallWithMRs(_service, tag,
		&mr0, &mr1, &mr2, &mr3);
	result = (seL4_Error) seL4_MessageInfo_get_label(output_tag);

    ...
}

// 内核态处理，上述我们已经提到了 decode_invocation，因此从这个函数开始解释
pub fn decode_invocation(...) {
    // 根据目标 cptr 的类型，找到对应的处理函数
    match capability.clone().splay() {
        ...
        // 发现能力是 tcb，调用 decode_tcb_invocation
        cap_Splayed::thread_cap(data) => {
            decode_tcb_invocation(label, length, &data, slot, call, buffer)
        }
    }
}

// decode_tcb_invocation
pub fn decode_tcb_invocation(
    // 从 tag 中解析出了 label，对应的就是 syscall 中的 TCBSuspend
    invLabel: MessageLabel,
    length: usize,
    capability: &cap_thread_cap,
    slot: &mut cte_t,
    call: bool,
    buffer: &seL4_IPCBuffer,
) -> exception_t {
    ...
    match invLabel {
        // 根据 label 进行 TCBSuspend 相关处理，因为是 suspend，所以将 task 状态改为 ThreadStateRestart
        MessageLabel::TCBSuspend => {
            set_thread_state(get_currenct_thread(), ThreadState::ThreadStateRestart);
            invoke_tcb_suspend(convert_to_mut_type_ref::<tcb_t>(
                capability.get_capTCBPtr() as usize
            ))
        }
    }
}

// invoke_tcb_suspend，真正进行操作的地方，通过 cptr 找到了 tcb，并对 tcb 进行一些操作
#[inline]
pub fn invoke_tcb_suspend(thread: &mut tcb_t) -> exception_t {
    // cancel_ipc(thread);
    thread.cancel_ipc();
    thread.suspend();
    exception_t::EXCEPTION_NONE
}

// 值得注意的是，在 handle_invocation 中，如果 decode_invocation 出现错误，会回复 error
pub fn handle_invocation(isCall: bool, isBlocking: bool) -> exception_t {
    ...
    if status == exception_t::EXCEPTION_SYSCALL_ERROR {
        if isCall {
            reply_error_from_kernel(thread);
        }
        return exception_t::EXCEPTION_NONE;
    }
}

// 在 reply_error_from_kernel 中主要执行的是，在 msginfo 寄存器 (a1) 中填入错误码
pub fn reply_error_from_kernel(thread: &mut tcb_t) {
    thread.tcbArch.set_register(ArchReg::Badge, 0);
    unsafe {
        let len = set_mrs_for_syscall_error(thread);
        thread.tcbArch.set_register(
            ArchReg::MsgInfo,
            seL4_MessageInfo::new(current_syscall_error._type as u64, 0, 0, len as u64).to_word(),
        );
    }
}

```

上述就是一个 syscall 完整过程，这个 syscall 虽然简单，但是也大致显示了 seL4 中 capability 是如何生效的。

但是这个过程中，没有涉及到 capability 的传递，下一章详细介绍该部分内容。

## 4. Capability 的传递

除了 root-task 掌握了所有资源外，其他任务的资源都依赖 root-task 或者父进程分配，因此 cptr 传递是一个很重要的设计。

我理解，seL4 中有显示和隐式两种 cptr 传递方式。

## 4.1 Capability 的显示传递

显示传递比较容易理解，就是通过如 seL4_CNode_Mint, seL4_CNode_Copy 等函数将一个 cslot 从一个 cnode 拷贝或者传递到另一个 cnode。在这个过程中，可以简单将 cslot 中存储的 cptr 当作是一个 C++ 智能指针，指向的实例始终是同一个，但是智能指针可以有多个。

我们以 seL4_CNode_Copy 为例，mint 相比于 copy，多出一些增加权限，badge 的功能。

seL4_CNode_Copy 一个很常用的场景是，root0-task 创造一个 capability，并将它传递给某一个子任务。

```c
// 用户态 syscall
LIBSEL4_INLINE seL4_Error
seL4_CNode_Copy(seL4_CNode _service, seL4_Word dest_index, seL4_Uint8 dest_depth, seL4_CNode src_root, seL4_Word src_index, seL4_Uint8 src_depth, seL4_CapRights_t rights)
{
	seL4_Error result;
    // 参考 seL4 message info 的定义，0 代表 unwrapped cptr num, 1 代表 有一个 extra cap， 5 代表 msg 长度
	seL4_MessageInfo_t tag = seL4_MessageInfo_new(CNodeCopy, 0, 1, 5);
	seL4_MessageInfo_t output_tag;
	seL4_Word mr0;
	seL4_Word mr1;
	seL4_Word mr2;
	seL4_Word mr3;

	/* Setup input capabilities. */
    // 将 src cptr root cnode 传递过来，从这可以看出，seL4_CNode_Copy 的调用者必须同时具有 src 和 dst cnode 的权限
	seL4_SetCap(0, src_root);

	/* Marshal and initialise parameters. */
    // 填入 src 和 dest cslot index，Copy 的时候不是直接将 cptr 实例拷贝，而是必须从一个 cslot 拷贝到另一个 cslot，cslot 本质上也就是一个 index
	mr0 = dest_index;
	mr1 = (dest_depth & 0xffull);
	mr2 = src_index;
	mr3 = (src_depth & 0xffull);
	seL4_SetMR(4, rights.words[0]);

	/* Perform the call, passing in-register arguments directly. */
	output_tag = seL4_CallWithMRs(_service, tag,
		&mr0, &mr1, &mr2, &mr3);
	result = (seL4_Error) seL4_MessageInfo_get_label(output_tag);
    ...
}
```

```rust
// 内核态处理
// 值得注意的是，如果需要使用多个 capbility 的时候，多余的 cptr 是放在 IPC Buffer 中的 extraCap 字段的，通过如下函数获取

let src_root = &get_extra_cap_by_index(0).unwrap().capability;

// 跳过之前介绍的 syscall 如何根据 cptr 类型进行处理的分析，直接来到 invoke_cnode_copy
// 主要分析 derive_cap 和 cte_insert 这两个函数

pub fn invoke_cnode_copy(
    src_slot: &mut cte_t,
    dest_slot: &mut cte_t,
    cap_right: seL4_CapRights,
) -> exception_t {
    ...
    // 按照前面的解释，cap 理解为一个智能指针，因此需要先 派生(derive) 出一个指向同一个实例的 cap，然后将派生出的这个 cptr 放到 dest cslot 中，完成复制过程
    let dc_ret = src_slot.derive_cap(&src_cap);
    ...
    cte_insert(&dc_ret.capability, src_slot, dest_slot);
}

pub fn derive_cap(&self, capability: &cptr) -> deriveCap_ret {
    if capability.is_arch_cap() {
        return self.arch_derive_cap(capability);
    }
    // 创建一个空的 deriveCap_ret, 用来存放派生出的 cptr
    let mut ret = deriveCap_ret {
        status: exception_t::EXCEPTION_NONE,
        capability: cap_null_cap::new().unsplay(),
    };

    match capability.get_tag() {
        // 这地方主要是进行一些检查，有一些 cptr 类型是不能派生的
        cap_tag::cap_zombie_cap => {
            ret.capability = cap_null_cap::new().unsplay();
        }
        cap_tag::cap_untyped_cap => {
            // untyped 类型派生时必须是一个全新的，至于为啥这么设计，没有研究
            // untyped 类型派生通常发生在，父进程从自己的 untyped cptr 中分出一部分内存，创建一个新的 untyped cap， 赋予给子进程
            ret.status = self.ensure_no_children();
            if ret.status != exception_t::EXCEPTION_NONE {
                ret.capability = cap_null_cap::new().unsplay();
            } else {
                ret.capability = capability.clone();
            }
        }
        #[cfg(not(feature = "kernel_mcs"))]
        cap_tag::cap_reply_cap => {
            ret.capability = cap_null_cap::new().unsplay();
        }
        cap_tag::cap_irq_control_cap => {
            // 很典型的例子，irq control 的能力只有 root-task 有，无法派生
            ret.capability = cap_null_cap::new().unsplay();
        }
        _ => {
            ret.capability = capability.clone();
        }
    }
    ret
}

// 完成派生后，就是将生成的 cptr 插入到指定 cslot 中
pub fn cte_insert(new_cap: &cap, src_slot: &mut cte_t, dest_slot: &mut cte_t) {
    ...
    // 插入的核心是这段，本质上就是将 src_slot 中的 cap 和其派生节点都移到 dest_slot
    dest_slot.capability = new_cap.clone();
    dest_slot.cteMDBNode = newMDB.clone();
    // 设置 src_slot 的派生节点为 dest slot，基本就是一个链表
    // 由于内核对象同处于一个地址空间，所以 cslot 中的派生 cptr 可以指向另一个 cnode 中的 cslot 
    src_slot
        .cteMDBNode
        .set_mdbNext(dest_slot as *const cte_t as u64);
    if newMDB.get_mdbNext() != 0 {
        let cte_ref = convert_to_mut_type_ref::<cte_t>(newMDB.get_mdbNext() as usize);
        cte_ref
            .cteMDBNode
            .set_mdbPrev(dest_slot as *const cte_t as u64);
    }
}
```

上述详细介绍了 seL4_CNode_Copy 在内核中的处理流程，其他 CNode 操作大致流程类似，感兴趣可以阅读 reL4 源码

## 4.2 Capability 的隐式传递

隐式传递其实就是通过 IPC 传递的机制，在通过 Endpoint 通信时，可以捎带上 cptr，传递给接收端。通常只能传递一个 cptr，多个也许也可以，没有仔细看文档了。传递的 cptr 放在 IPC Buffer 的 extra_cap 字段，接收方则需要在 IPC Buffer 中指定 receiveCNode receiveIndex receiveDepth 才可以接收，这三个字段都是定义在 IPC Buffer 中。

用户态我们以 rel4-linux-kit 中 RegisterIRQ 服务为例，client 请求发送功能代码如下

```rust
pub fn register_irq(irq: usize, target_slot: LeafSlot) {
    let msg = &mut MessageInfo::new(RootEvent::RegisterIRQ.into(), 0, 0, 1);

    // construct the IPC message
    let origin_slot = with_ipc_buffer_mut(|ipc_buffer| {
        // 在这设置了 receiveCNode receiveIndex receiveDepth
        ipc_buffer.set_recv_slot(&target_slot.abs_cptr());
        ipc_buffer.msg_regs_mut()[0] = irq as _;
        ...
    });

    // 调用了 seL4_call，call 可以理解为 send 和 recv 的组合，上面 recv_slot 在其中的 recv 阶段生效
    let recv_msg = call_ep!(msg.clone());
    ...
}
```

服务端做了以下处理

```rust
RootEvent::RegisterIRQ => {
    let irq = read_types!(ib, u64);
    let dst_slot = LeafSlot::new(0);
    // 调用自己的 IRQ_CONTROL 能力，创建一个 irq handler，内核收到对应 irq 中断时，会通知任务，有点像信号量
    slot::IRQ_CONTROL
        .cap()
        .irq_control_get(irq, &dst_slot.abs_cptr())
        .unwrap();

    // 将获取的 irq_handler cptr 所在的 cslot 放到 extra_cap 中
    ib.caps_or_badges_mut()[0] = 0;
    // 使用 reply syscall, reply 是一个专门用来恢复 call 的 syscall，可以简单理解为 send
    sel4::reply(ib, rev_msg.extra_caps(1).build());

    dst_slot.delete().unwrap();
}
```

上述时用户态的用法，看起来还是比较简洁的，接收方填入 recv_slot， 发送方将 cptr 所在的 cslot，kernel 起到的作用就是将发送方的 cslot move 到 recv_slot，我们看一下 kernel 代码

上面提到，reply 是一个专门回复 call 的 syscall，在这里我们不介绍其设计，而是直接进入 reply syscall

```rust
#[cfg(not(feature = "kernel_mcs"))]
fn handle_reply() {
    let current_thread = get_currenct_thread();
    // caller 就是调用 call 的进程，该进程在 call 的 send 阶段，会把自己填到接收方的 TCB_CALLER slot 里，存一个 reply cap 类型
    let caller_slot = current_thread.get_cspace_mut_ref(TCB_CALLER);
    if caller_slot.capability.clone().get_tag() == cap_tag::cap_reply_cap {
        if cap::cap_reply_cap(&caller_slot.capability).get_capReplyMaster() != 0 {
            return;
        }
        let caller = convert_to_mut_type_ref::<tcb_t>(
            cap::cap_reply_cap(&caller_slot.capability).get_capTCBPtr() as usize,
        );
        // 找到 caller 后
        current_thread.do_reply(
            caller,
            caller_slot,
            cap::cap_reply_cap(&caller_slot.capability).get_capReplyCanGrant() != 0,
        );
    }
}

#[cfg(not(feature = "kernel_mcs"))]
fn do_reply(&mut self, receiver: &mut tcb_t, slot: &mut cte_t, grant: bool) {
    // 检查下 caller 是否处于等待 reply 的状态
    assert_eq!(receiver.get_state(), ThreadState::ThreadStateBlockedOnReply);
    let fault_type = receiver.tcbFault.get_tag();
    if likely(fault_type == seL4_Fault_tag::seL4_Fault_NullFault) {
        // 进行 cptr 的传递，这里的 receiver 就是调用 call 的任务，作为接收者
        self.do_ipc_transfer(receiver, None, 0, grant);
        // 删除这个 reply cap，每个 call 会产生一个临时的 reply，用完就删掉
        slot.delete_one();
        // receiver 之前一直在阻塞等待 reply，reply 后，将其状态改成可运行，并切换到该任务
        set_thread_state(receiver, ThreadState::ThreadStateRunning);
        possible_switch_to(receiver);
    } else {
        ......
    }
}

// 所以传递 cptr 的核心函数是 do_ipc_transfer
fn do_ipc_transfer(
    &mut self,
    receiver: &mut tcb_t,
    ep: Option<&endpoint>,
    badge: usize,
    grant: bool,
) {
    if likely(self.tcbFault.get_tag() == seL4_Fault_tag::seL4_Fault_NullFault) {
        self.do_normal_transfer(receiver, ep, badge, grant)
    } else {
        self.do_fault_transfer(receiver, badge)
    }
}

// 我们暂时不考虑 Fault 的情况，调用 do_normal_transfer
fn do_normal_transfer(
    // self 是 tcb_t，这是一个 tcb_t 的成员函数
    &mut self,
    receiver: &mut tcb_t,
    ep: Option<&endpoint>,
    badge: usize,
    can_grant: bool,
) {
    let mut tag =
        seL4_MessageInfo::from_word_security(self.tcbArch.get_register(ArchReg::MsgInfo));
    let mut current_extra_caps = [0; SEL4_MSG_MAX_EXTRA_CAPS];
    if can_grant {
        // 具备 grant 权限时，才会获取当前 tcb IPC buffer 中的 extra_caps 信息
        // lookup_extra_caps 获取 extra_cap 字段并且填到 current_extra_caps 中
        let status = self.lookup_extra_caps(&mut current_extra_caps);
        if unlikely(status != exception_t::EXCEPTION_NONE) {
            current_extra_caps[0] = 0;
        }
    } 
    ...
    // 在这一步执行了传递
    receiver.set_transfer_caps(ep, &mut tag, &current_extra_caps);
    ...
}

// 在 set_transfer_caps 中进行传递
fn set_transfer_caps(
    &mut self,
    ep: Option<&endpoint>,
    info: &mut seL4_MessageInfo,
    current_extra_caps: &[pptr_t; SEL4_MSG_MAX_EXTRA_CAPS],
) {
    info.set_extraCaps(0);
    info.set_capsUnwrapped(0);
    let ipc_buffer = self.lookup_mut_ipc_buffer(true);
    if current_extra_caps[0] as usize == 0 || ipc_buffer.is_none() {
        return;
    }
    let buffer = ipc_buffer.unwrap();
    // 从 receiver 的 ipc buffer 中，获取 receive slot
    let mut dest_slot = self.get_receive_slot();
    let mut i = 0;
    while i < SEL4_MSG_MAX_EXTRA_CAPS && current_extra_caps[i] as usize != 0 {
        let slot = convert_to_mut_type_ref::<cte_t>(current_extra_caps[i]);
        let capability_cpy = &slot.capability.clone();
        // 这部分是一些 unwrapped 功能的实现，没细看
        ...

        else {
            if dest_slot.is_none() {
                // 只接受第一个传递的 cptr
                break;
            } else {
                // 和上述 copy 里面介绍的一样，生成一个派生，然后插入目标 slot
                let dest = dest_slot.take();
                let dc_ret = slot.derive_cap(&capability_cpy);
                if dc_ret.status != exception_t::EXCEPTION_NONE
                    || dc_ret.capability.get_tag() == cap_tag::cap_null_cap
                {
                    break;
                }
                cte_insert(&dc_ret.capability, slot, dest.unwrap());
                dest_slot = None;
            }
        }
        i += 1;
    }
    info.set_extraCaps(i as u64);
}
```

# 5 capability 创建

上述详细介绍了 capability 使用和流转的过程，现在最后介绍下 capability 创建的过程。前面说到，capability 可以理解成一个智能指针。那么创建的过程相当于

```cpp
// T 是具体的 capability 类型，如 tcb, endpoint 
auto cptr = std::make_shared<T>();
```

当然在 seL4 中，由于内存管理更加严格，创建过程远比上述复杂。同样，我们先看下用户态如何创建一个能力。

seL4 中，所有空闲内存都由 untyped 能力管理，通过 seL4_Untyped_Retype API 创建一个内核对象，也就是 **能力**

```c
LIBSEL4_INLINE seL4_Error
seL4_Untyped_Retype(seL4_Untyped _service, seL4_Word type, seL4_Word size_bits, seL4_CNode root, seL4_Word node_index, seL4_Word node_depth, seL4_Word node_offset, seL4_Word num_objects)
{
    ...

	/* Setup input capabilities. */
    // 设置一个接收 cnode，比如 root cnode，根据 node_index 和 depth 可以得到 slot
	seL4_SetCap(0, root);

	/* Marshal and initialise parameters. */
	// 填一些参数，比如对象类型，放置的 cslot 等等
    mr0 = type;
	mr1 = size_bits;
	mr2 = node_index;
	mr3 = node_depth;
    // 看起来是可以批量创建，我们暂且不分析
	seL4_SetMR(4, node_offset);
	seL4_SetMR(5, num_objects);

	/* Perform the call, passing in-register arguments directly. */
    // 调用 syscall 
	output_tag = seL4_CallWithMRs(_service, tag,
		&mr0, &mr1, &mr2, &mr3);
	result = (seL4_Error) seL4_MessageInfo_get_label(output_tag);

	/* Unmarshal registers into IPC buffer on error. */
    ...
}
```

我们再看下 rel4-linux-kit 中是如何创建内核对象的

```rust
// 使用 OBJ_ALLOCATOR 作为对象生成分配器，创建一个 endpoint
let fault_ep = OBJ_ALLOCATOR.lock().alloc_endpoint();

// OBJ_ALLOCATOR 中的创建新能力的方法
pub fn allocate_and_retype(
    &mut self,
    // 对象类型
    blueprint: sel4::ObjectBlueprint,
) -> sel4::cap::Unspecified {
    // 对象或者说能力需要一个 slot 承载
    let leaf_slot = self.allocate_slot();

    // 近似于调用 seL4_Untyped_Retype
    self.ut
        .untyped_retype(
            &blueprint,
            &leaf_slot.cnode_abs_cptr(),
            leaf_slot.offset_of_cnode(),
            // rel4-linux-kit 中只支持创建一个对象
            1,
        )
        .unwrap();
    // 返回创建完成的能力，其实能力在用户态也就是一个 index，实例存储在 kernel 中
    leaf_slot.cap()
}

```

内核态中有 invoke_untyped_retype 函数处理 seL4_Untyped_Retype. 该部分是 untyped 能力的重要实现。

> seL4 内核中的 cap 类型定义在 structures.bf 文件中，所有的 cap 类型都是一个 2usize 长度的位图。基本上所有的 cap 类型，比如 cnode，endpoint 都会存储它指向的实例地址，像一个指针。但是 untyped 能力不同，所有的信息都存在这 2usize 的位图中，没有所谓的实例。

下面我们详细分析下 decode_untyed_invocation 的实现，主要的实现如下

```rust
pub fn decode_untyed_invocation(
    inv_label: MessageLabel,
    length: usize,
    slot: &mut cte_t,
    capability: &cap_untyped_cap,
    buffer: &seL4_IPCBuffer,
) -> exception_t {
    ...
    // 获取对象创建类型
    let op_new_type = ObjectType::from_usize(get_syscall_arg(0, buffer));

    // 获取 syscall 参数
    let new_type = op_new_type.unwrap();
    // 有的类型，比如 untyped 类型申请的内存大小是不固定的
    let user_obj_size = get_syscall_arg(1, buffer);
    let node_index = get_syscall_arg(2, buffer);
    let node_depth = get_syscall_arg(3, buffer);
    let node_offset = get_syscall_arg(4, buffer);
    let node_window = get_syscall_arg(5, buffer);
    // 返回对象大小，有一些类型是固定的，有一些则从 user_obj_size 获取
    let obj_size = new_type.get_object_size(user_obj_size);

    // 有一些检查操作，确认参数有效
    // 找到 dest cnode，通常可能是 root_cnode
    let node_cap = &mut cap_cnode_cap::new(0, 0, 0, 0);
    let status = get_target_cnode(node_index, node_depth, node_cap);
    if status != exception_t::EXCEPTION_NONE {
        return status;
    }
    ...

    let status = slot.ensure_no_children();
    let (free_index, reset) = if status != exception_t::EXCEPTION_NONE {
        // 原始 untype 有子节点，获取当前 untyped 中空闲位置的 index，也就是相对位置
        (capability.get_capFreeIndex() as usize, false)
    } else {
        (0, true)
    };

    // 获取空闲内存地址，untyped cap base_addr + free_index
    let free_ref = GET_FREE_REF(capability.get_capPtr() as usize, free_index);
    // 获取还剩下的空闲内存大小
    let untyped_free_bytes = BIT!(capability.get_capBlockSize()) - FREE_INDEX_TO_OFFSET(free_index);
    // 做一个对齐处理，获取对齐后的地址
    let aligned_free_ref = alignUp(free_ref, obj_size);
    // 执行 invoke_untyped_retype，在其中创建对象，创建相应能力，并且将该能力插到指定 cslot 中
    invoke_untyped_retype(
        slot,
        reset,
        aligned_free_ref,
        new_type,
        user_obj_size,
        convert_to_mut_type_ref::<cte_t>(node_cap.get_capCNodePtr() as usize),
        node_offset,
        node_window,
        device_mem as usize,
    )
}
```
