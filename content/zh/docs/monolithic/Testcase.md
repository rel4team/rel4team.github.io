---
title: "用户测试程序编写与适配"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

上一节提到我们编写、运行的“宏内核”本身是一个库操作系统，那么它应当如何像一个真正的宏内核（如 Linux）一样和用户测试程序进行交互呢？本节即将讨论这方面的内容。

## 1. 概念说明
在 seL4/reL4 看来，我们的“宏内核“就是用户态程序，而宏内核本身又需要读取、运行用户编写的测试程序或者实际程序。这里都提到了“用户程序”的概念，因此我们先在这里进行概念的界定。

- 用户测试程序（简称“测试程序”）：用户编写的测试程序，用于测试“宏内核”的功能是否正常。这里的测试程序可以是一个简单的 hello world 程序，也可以是一个复杂的实际程序，如 redis。
- 用户内核：我们的“宏内核”，在 seL4/reL4 上它是直接运行的用户态程序，负责接收来自程序的 syscall，并且将其转发到对应的设备任务上。

## 2. 测试程序的运行

### 2.1. syscall 的转发和接收
讨论测试程序的运行前，我们需要了解用户内核是如何接收 syscall 的。目前的机制如下：

- [kernel-thread](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/blob/main/crates/kernel-thread/src/child_test.rs): 内核任务，负责创建用户测试程序对应的任务，并监听用户测试程序的 IPC 请求，包括 syscall、fault 等。
- [test-thread](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/blob/main/crates/test-thread/src/main.rs): 用户测试程序，负责运行待测的用户态二进制文件，并通过 `vsyscall handler` 这一特殊的函数将其调用的 syscall 转化为 IPC 和内核任务进行通信。

### 2.2. rust 测试程序的编写

测试程序的编写可以参考 [test-thread](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/blob/main/crates/test-thread/src/main.rs)。下文的示例代码完成了一次 mmap 的调用：

```rust
/// vsyscall handler.
pub fn vsyscall_handler(
    id: usize,
    a: usize,
    b: usize,
    c: usize,
    d: usize,
    e: usize,
    f: usize,
) -> usize {
    // More code here
    // Write syscall registers to ipc buffer.
    with_ipc_buffer_mut(|buffer| {
        let msgs: &mut [u64] = buffer.msg_regs_mut();
        msgs[0] = id as _;
        msgs[1] = a as _;
        msgs[2] = b as _;
        msgs[3] = c as _;
        msgs[4] = d as _;
        msgs[5] = e as _;
        msgs[6] = f as _;
    });
    // Load endpoint and send SysCall message.
    let ep = Cap::from_bits(EP_CPTR.load(Ordering::SeqCst));
    let message = ep.call(MessageInfo::new(
        CustomMessageLabel::SysCall.to_label(),
        0,
        0,
        7 * WORD_SIZE,
    ));

    if prev_id != 0 {
        set_tp_reg(tp);
        if get_tid() != prev_id {
            return 0;
        }
    }
    // More code here
    // Get the result of the fake syscall
    let ret = with_ipc_buffer_mut(|buffer| buffer.msg_regs()[0]);

    ret as usize
}

let mmap_ptr = vsyscall_handler(
    Sysno::mmap.id() as usize,
    0x1000,
    0x1000,
    0b11,
    0b10000,
    0,
    0,
);
```

test-thread 会被编译为一个可执行文件，并且在用户内核中被加载运行。在用户内核中，我们通过 `vsyscall_handler` 来调用 test-thread 中的函数，从而完成 syscall 的转发。

### 2.3 c 测试程序的编写
目前我们还没有提供 c 语言的测试程序，但是可以参考 arceos 中的 [arceos_posix_api](https://github.com/arceos-org/arceos/tree/main/api/arceos_posix_api) 的实现，将 test-thread 中的 syscall handler 按照 musl-libc 或者 glibc 的接口进行实现，并对外作为一个库的形式提供。

用户编写了 c 语言的测试程序之后，我们可以让他们链接到我们提供的库上，并编译为一个可执行文件，然后像 test-thread 一样在用户内核中运行。

## 2.4 可执行二进制文件的适配
对于用户内核来说，我们更加希望能够直接运行已经编译完成的二进制文件，从而在可执行文件的级别完成兼容。

Linux APP 调用系统调用的时候直接使用 syscall 等同语义的指令，将执行流从用户态陷入到内核态中，并且完成相关的处理操作。但是由我们的用户内核事实上是一个用户态程序，而真实的 syscall 接口由微内核(seL4/reL4)使用，我们无法在内核态对 posix 接口进行分发和处理，因此需要对 syscall 进行拦截和处理。

一个参考实现是 [ChCore](https://gitlab.eduxiji.net/educg-group-22026-2376550/T202410248992613-554/-/blob/main/docs/binary-rewrite.md)，它实现了一个二进制重写的工具，在加载 Linux APP 的时候将 Linux APP 的二进制文件中的 syscall 指令替换为对应的 IPC 调用，从而实现了对 Linux APP 的兼容。

虽然这个工具只是面对 RISC-V 架构，但是我们可以参考其实现思路，实现一个类似的工具，从而能够在用户内核中直接运行已经编译好的二进制文件，实现思路包括如下要点：
1. 不同架构适配：RISC-V 作为定长指令集，可以用较为简单的方式读取到 syscall 指令并且进行重写。虽然 x86_64 等架构是变长指令集，但是假设起始运行地址确定，每条指令在读取的时候就确定了它的长度，因此每条指令的语义也是确定的（这也是反汇编能够成立的前提）。我们可以利用这个原理，完成对二进制文件的解析，重写 syscall 指令。事实上这个操作应该在 CTF 里面是比较常见的，因此这类工作应当有现成的工具可以使用。
2. lazy alloc：syscall 指令的拦截和重写可以和 lazy alloc 等机制结合，仅当物理页面被分配的时候才进行页面的解析和重写，从而减少加载文件时带来的性能损耗。
