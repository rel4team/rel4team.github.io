---
title: "Rel4 Linux Kit 学习笔记"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## 1. 介绍

本文记录了学习 seL4/reL4 用户态操作系统的过程和心得体会。由于 seL4 中有很多独特的设计，因此上手难度较高。我作为初学者，记录下我的学习过程，也许对其他人理解有一定参考价值。

## 2. reL4-Linux-Kit 简介

seL4 是一个高安全性的微内核，reL4 是 seL4 的 rust 实现。

但是和 Linux 这种宏内核不同，微内核功能过于简单，无法提供完整的操作系统服务，因此需要在用户态开发各种服务以实现完整的操作系统功能。

[reL4-linux-kit 架构](https://rel4team.github.io/zh/docs/monolithic/latest/design/)

reL4-Linux-Kit 是一个基于 reL4 的 Linux 运行时环境，它允许在 reL4 上运行 Linux 应用程序。reL4-Linux-Kit 提供了一个用户态的 Linux 环境，使得开发者可以在 seL4/reL4 上运行现有的 Linux 应用程序，同时享受 seL4/reL4 提供的高安全性和性能。

同时，reL4-Linux-Kit 提供了一些通用的操作系统服务，因此也可以嫁接别的操作系统到 seL4 上，比如 ArceOS

reL4-Linux-Kit 目前提供以下组件。

- root_task

- block device task

- ext4 file system task

- fat file system task

- uart task

- kernel task

其中 root task 和 kernel thread 需要重点介绍下，其他服务比较容易理解。

## 2.1 root_task

root_task 是系统的初始任务，内核态在初始化完成后，会创建 root_task 并将所有资源 (内存，中断等) 交给 root_task 管理

root_task 会根据配置创建其他服务进程，给他们分配初始资源。除此之外，还在运行时提供一些需要操作权限的服务，具体的说，root task 提供以下服务

- Channel 创建和加入，Channel 是 rust-linux-kit 实现的 IPC 机制，通过 share memory 和 notification 机制 (类似信号机制) 实现进程间通信

- Translate Addr，root_task 存储了分配出去的物理页帧和虚拟页的映射关系，因此可以通过 root_task 来查询虚拟地址对应的物理地址

- 服务发现，根据服务名找到对应服务的 IPC 端口 (端口是 seL4 重要概念，用于进程间通信)

- 中断号注册，seL4 中必须通过 irq control 这个能力来注册中断，而且该能力无法移动，只有 root_task 拥有，因此其他服务必须通过 root_task 来注册中断。注册完成后，发生中断时，内核会发送 notification 给用户态，告知其中断发生

- 分配 Notifacation， 这是一个很标准的 root_task retype cnode，然后将其赋予给其他任务的过程。简单理解，root_task 为其他任务创建一个 notifacation 实例 并将其分配给其他任务。由于 seL4 中，所有的内核实例都是 cnode，因此创建分配其他类型实例的过程都是一样的

- Alloc Page: 向 root_task 申请一个物理页，并且映射到指定的虚拟地址

## 2.2 kernel task

顾名思义，kernel task 是一个宏内核的用户态实现，补齐 seL4 微内核不具备的内核功能。其最大的功能是创造了一个可以运行 Linux 应用的运行时环境。

kernel task 一个很大的创新是通过工具自动修改 Linux 应用中的 syscall 指令，是的执行 syscall 的时候，触发 user fault，seL4 kernel 会将 fault 转发给 kernel task。因为 syscall 参数都在寄存器上，所以 kernel task 此时就可以直接处理 syscall 了。

[运行 Linux 原理说明](https://rel4team.github.io/zh/docs/monolithic/latest/bin-compact/)

## 3. 一些实现细节

### 3.1 系统启动过程

首先，微内核的启动过程和大家熟悉的 Linux 等宏内核就有很大区别。由于微内核没有驱动和文件系统功能，因此无法在内核初始化完成后，直接从文件系统中读取应用 elf 文件并加载。

因此，seL4 会将 root-task 和 kernel 打包成一个镜像，并且 root-task 在一个固定位置，这样 kernel 启动后就可以从内存中读取到 root-task 并且加载。

除此之外，由于在 rel4-linux-kit 的设计中，设备驱动，文件系统服务是单独的，root-task 没有实现。因此 root-task 至少需要把 uart 驱动，块设备驱动，文件系统服务，kernel-task 拉起来，才能提供一个完整的操作系统环境。

但是正如之前说的，root-task 如何找到这些进程呢？其实和 kernel 找 root-task 一样，在编译的时候，使用 include_bytes_aligned 的方法将这些服务的二进制文件放到了镜像的指定位置，所以 root-task 在启动时就可以直接从内存中读取到这些服务的二进制文件，并加载它们。

```
KernelServices {
    name: $name,
    file: include_bytes_aligned!(16, concat!("../../target/", $file)),
    mem: &[$(($mem_virt, $mem_phys, $mem_size)),*],
    dma: &[$(($dma_addr, $dma_size)),*],
}
```

root-task 启动后，首先会收集下内核给自己的 untyped 资源，包括实际的内存资源，和外设资源，其中外设资源就是外设的起始物理地址，这就要求用户态进程在申请使用这些资源时，需要知道外设的物理地址。

然后会重建 root-task 的 cspace，因为默认是 1 级 cspace，reL4-linux-kit 重建成 2 级 cspace. 这部分对我来说有点难以理解，会写一个我对 capability 机制的学习文档。

之后，root-task 会申请一个新的 endpoint，这个 endpoint 作为 root-task 和它的子任务之间 IPC 通道。在之后创建子任务的时候，会把这个 endpoint 传给子任务。

### 3.2 root-task 创建任务

通过 root-task 创建并拉起其他进程，是微内核很典型的操作，也是和宏内核很不同的设计。在 rel4-linux-kit 中，root-task 创建任务时需要做以下几件事情。

1. 和 root-task 一样，创建一个 2 级的页表。相比于 root-task 稍微简单一点的是，子任务的 cspace 是空的，所以无需像 root-task 处理已有的 cslot

2. 创建一个 vspace，将 ELF 文件中定义的虚拟地址加载到 vspace 中，再根据各个段的虚拟地址，申请物理地址，并且将 ELF 文件拷贝过去。这样就将子任务的 ELF 数据从 root-task 中分离出来了。

3. 创建该任务的 IPC Buffer，TCB 以及一个 endpoint, 这个 endpoint 是该任务提供服务的 IPC 接口

4. 将 root-task 的 endpoint mint 到子任务的 DEFAULT_PARENT_EP cslot 中，mint 主要的目的是加上 badge，这样 root-task 接收到消息后，可以根据 badge 判断发送者是谁。将刚刚创建的 endpoint copy 到子任务的 DEFAULT_SERVE_EP cslot

5. 将 cnode, vspace, tcb 等等 capability 拷贝到默认 slot，比如 TCB 是 1. 这一步操作的必要性是什么，暂时没弄明白。同时创建子任务的栈

6. 设置一些 task 属性，比如优先级等等

7. 映射 TASK 设置中定义的 device 和 DMA 物理地址到子任务 vspace 中，并且给 DMA 分配物理内存

8. 完成创建，调用 seL4_TCB_Resume 启动各个任务

### 3.3 IPC 请求的过程

微内核会将操作系统的各个组件拆分成各个服务，rel4-linux-kit 中每个基础组件都提供自己的服务，比如驱动，文件系统。root-task 管理所有系统资源，当然也会提供多种服务供其他进程调用。

在 rel4-linux-kit 的设计中，所有的组件提供服务的方式都是完全一样的。每个服务都有属于自己的 endpoint,其他服务只有获取到这个 endpoint 时，才可以向它发送请求。父进程在创建子进程的时候，会将自己的 endpoint 给到子进程，所以子进程创建后就可以访问其父进程的服务。如果想要访问其他服务，则需要先向 root-task 发送服务发现请求，root-task 会将对应的服务 endpoint 返回，然后才能访问其他服务。

以 root-task 提供的 RegisterIRQ 服务为例，在 waiting_and_handle 函数中，轮询等待接收 IPC 请求

```
pub fn waiting_and_handle(&mut self, ib: &mut IpcBuffer) -> ! {
    loop {
        // fault_ep 这个名称有点疑惑，它其实就是 root-task 唯一的接收端口，接收其他进程的 IPC 和 kernel 发来的 fault IPC
        // 获取 message_info 和 badge, message info 是 IPC buffer 中定义的 message tag, badge 代表发送者是谁
        let (message, badge) = self.fault_ep.recv(());
```

通过 message label 判断是什么服务的请求，如上所说，我们以 RegisterIRQ 为例，进入 RegisterIRQ 处理阶段。

在说明 service 处理请求之前，我们先看一眼 client 是如何发送请求的。client 使用 register_irq 函数发送请求

```
// irq: 需要注册的中断号，target_slot: 用来接收 service 返回的 irq capability
pub fn register_irq(irq: usize, target_slot: LeafSlot) {
    // 创建一个 msg info，指定是 RegisterIRQ 服务
    let msg = &mut MessageInfo::new(RootEvent::RegisterIRQ.into(), 0, 0, 1);

    // construct the IPC message
    let origin_slot = with_ipc_buffer_mut(|ipc_buffer| {
        // 把 target_slot 设置为 recv_slot, 用来接收返回的 irq cap
        ipc_buffer.set_recv_slot(&target_slot.abs_cptr());
        // 将 irq 放到消息寄存器里，传给 service
        ipc_buffer.msg_regs_mut()[0] = irq as _;

        // FIXME: using recv_slot()
        // 这部分主要是为了给 origin_slot 赋值，等于 root cnode，其实就是恢复初始状态
        init_thread::slot::CNODE
            .cap()
            .absolute_cptr(Null::from_bits(0))
    });

    // 调用 call, call 分为 send 和 recv 两部分
    // 在 recv 的时候，kernel 会将 service 返回的 irq cap 填到 target_slot 里面
    let recv_msg = call_ep!(msg.clone());
    assert!(recv_msg.extra_caps() == 1);

    // 恢复 ipc buffer，如上所说，设置为 root cnode
    with_ipc_buffer_mut(|ipc_buffer| ipc_buffer.set_recv_slot(&origin_slot));
}
```

说完客户端如何发送请求，再来看看服务端如何处理请求

```
RootEvent::RegisterIRQ => {
    // 从 msg reg 中读取中断号
    let irq = read_types!(ib, u64);
    // 创建一个临时的 slot，用来存储 syscall 获取的 irq cap
    let dst_slot = LeafSlot::new(0);
    // 调用 irq_control_get syscall,只有 root-task 具有 IRQ_CONTROL cap，并且可以进行 irq 注册操作
    // 注册完成后，kernel 会将产生的 IrqHandler cap 放到 dst_slot 中
    slot::IRQ_CONTROL
        .cap()
        .irq_control_get(irq, &dst_slot.abs_cptr())
        .unwrap();

    // 把 dst_slot 的 index 放到 ipc buffer 中，因为临时 slot index 就是 0，所以 = 0
    ib.caps_or_badges_mut()[0] = 0;
    // 调用 reply syscall, kernel 会将 ipc buffer 中存储的 cap 转移到客户端的 cspace 中
    sel4::reply(ib, rev_msg.extra_caps(1).build());

    // cap 已经转移，删除临时的 slot
    dst_slot.delete().unwrap();
}
```

上述就是一个很典型的 IPC 请求处理过程

### 3.4 Channel 的使用

上述的 IPC 请求只能处理数据量较小的请求，对于类似文件读取这种大数量的请求，用 endpoint 显然效率很低。因此基于 shared memory 设计的 channel，更适合用于该场景。

使用者，通常是 client 会根据需求创建 channel，并且会告诉服务端，创建的 channel id. 服务端会根据 channel id 加入 channel。在通信时，双方只需要告知对方数据的相对地址和长度，即可完成通信。

### 3.5 IPC 转 lib, lib 转 IPC 设计

关于函数转 IPC 的设计和实现请参考以下两篇文章

- [函数转 IPC 设计](https://rel4team.github.io/zh/docs/monolithic/funcs/func2ipc/)
- [函数转 IPC 实现](https://rel4team.github.io/zh/docs/monolithic/funcs/impl-v1/)

下面我从初学者的视角，解释下我的理解

用一句话来解释，**调用者通过 rust cfg 来决定是直接调用函数，还是发送 IPC**。调用后端的处理函数是同一个，只是调用它的方式不同。

我们以 kernel-task 为例，看一下这套工具是如何工作的。

从设计文档中，我们可以知道，当你使用 app-parser 工具时，如果只定义了一个进程。那么这个进程会自动将它依赖的进程的所有资源继承过来，调用依赖进程直接通过函数调用，而不是 IPC. 换句话说，它依赖的进程变成了 lib, 供调用。

```
$ python3 tools/app-parser.py kernel-thread
[Task {
	name = kernel-thread
	file = kernel-thread
    // 这些内存资源是从它的依赖那里继承的，具体的说，分别从 fs 和 uart 中继承，其中 fs 也是继承自 blk-device
	mem = [['VIRTIO_MMIO_VIRT_ADDR', 'VIRTIO_MMIO_ADDR', '0x1000'], ['VIRT_PL011_ADDR', 'PL011_ADDR', '0x1000']]
	dma = [['DMA_ADDR_START', '0x2000']]
	deps = ['fs-thread', 'uart-thread']
}]
```

如果我们解析成两个进程

```
$ python3 tools/app-parser.py kernel-thread fs-thread
[Task {
	name = kernel-thread
	file = kernel-thread
    // 不会继承 fs 的资源
	mem = [['VIRT_PL011_ADDR', 'PL011_ADDR', '0x1000']]
	dma = []
	deps = ['fs-thread', 'uart-thread']
}, Task {
	name = fs-thread
	file = lwext4-thread
	mem = [['VIRTIO_MMIO_VIRT_ADDR', 'VIRTIO_MMIO_ADDR', '0x1000']]
	dma = [['DMA_ADDR_START', '0x2000']]
	deps = ['block-thread']
}]

# tools/autoconfig.mk 中增加了一个 build cfg. 这样所有应用知道使用 fs_ipc 模式
RUSTFLAGS += --cfg=fs_ipc
```

通过上述比较，我们大概能明白设计想要达到的目标，就是可以灵活修改操作系统各个组件的组织方式。直接函数调用效率更高，但是如果多个进程有相同的依赖，那只能通过 IPC 的方式实现，同时 IPC 的方式，各个组件松耦合，系统更加安全可靠。

从全局了解该功能的效果后，我们继续看下各个服务是如何同时支持函数调用和 IPC 调用这两种方式的。

首先，既然直接函数调用和 IPC 调用实现的功能都是一样的，那么我们可以定义一个统一的接口 trait，直接函数调用还是 IPC 无非是如何实现接口的问题。比如 write_at 这个文件系统接口，直接调用就是调用 fs lib 中的 write_at 函数，IPC 方式就是调用一个 ipc call 函数，发送请求给 fs service.

在 crates/srv-gate lib.rs 中，可以看到定义了三个全局变量数组

```rust
#[distributed_slice]
pub static UART_IMPLS: [Lazy<Arc<Mutex<dyn UartIface>>>];
#[distributed_slice]
pub static BLK_IMPLS: [Lazy<Arc<Mutex<dyn BlockIface>>>];
#[distributed_slice]
pub static FS_IMPLS: [Lazy<Arc<Mutex<dyn FSIface>>>];
```

这里存储了刚才提到的接口的实例，无论是直接函数调用的实例还是 IPC 调用的实例，都可以放到这里，因为他们都实现了同样的接口。

比如 fs 的接口定义

```
#[ipc_trait(event = FS_EVENT)]
pub trait FSIface: Sync + Send {
    fn init(&mut self, channel_id: usize, addr: usize, size: usize);
    fn read_at(&mut self, inode: u64, offset: usize, buf: &mut [u8]) -> usize;
    fn write_at(&mut self, inode: u64, offset: usize, data: &[u8]) -> usize;
    fn open(&mut self, path: &str, flags: u32) -> Result<(usize, usize), Errno>;
    fn mkdir(&self, path: &str);
    fn unlink(&self, path: &str);
    fn close(&mut self, inode: usize);
    fn stat(&mut self, inode: usize) -> Stat;
    fn getdents64(&mut self, inode: u64, offset: usize, buf: &mut [u8]) -> (usize, usize);
}
```

IPC 调用的实现在 FSIfaceIPCImpl，详细内容直接阅读代码。函数调用实现可以有多个，比如 lwext4-thread 中的 EXT4FSImpl 也实现了 FSIfaceIPCImpl.

值得注意的是，通过 def_fs_impl 宏定义，生成一个 FSIfaceIPCImpl 实例并将其放到 `FS_IMPLS` 数组中。

```rust
// IPC 版本
def_fs_impl!(FS_IPC, FSIfaceIPCImpl {
    ep: find_service("fs-thread").unwrap().into(),
    share_addr: 0,
    share_size: 0
});

// 函数调用版本
// services/lwext4-thread/src/lib.rs
def_fs_impl!(EXT4FS, EXT4FSImpl::new());
```

调用者使用 FS_IPC 全局变量调用 FS 实例，可能是直接函数调用，也可能是发送 IPC.

```rust
// 比如 kernel thread 中的 fs 使用 FS_IMPLS 作为 fs 调用接口
impl FileSystem for IPCFileSystem {
    fn info(&self) -> super::vfs::FSInfo {
        todo!()
    }

    fn open(&self, path: &str, flags: u64) -> FileResult<Box<dyn FileInterface>> {
        // let (inode, fsize) = self.fs.open(path, flags)?;
        let (inode, fsize) = FS_IMPLS[self.fs].lock().open(path, flags as _)?;
        Ok(Box::new(IPCFile {
            path: String::from(path),
            inode: inode as _,
            fsize: fsize as _,
            fs: self.fs,
        }))
    }
}
```

