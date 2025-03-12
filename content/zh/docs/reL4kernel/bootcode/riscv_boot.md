+++
weight = 5
date = '2025-01-08T14:30:15+08:00'
title = '启动代码和编译系统移植'
+++

# 启动代码和编译系统移植

## 1. 简介

和所有操作系统 Kernel 一样，在运行用户态程序之前，Kernel 有一系列初始化工作。

这部分代码由小部分汇编代码 + Rust 代码实现，同时需要设计 linker scripts，以确保代码段按照设计的内存布局放置。

同时 SeL4 编译系统基于 CMake，因此我们需要让 ReL4 适配原有编译系统，达到无缝替换 SeL4 Kernel 的目标。

## 2. Linker scripts

我们从 linker scripts 开始移植，seL4 中的 linker 脚本根据配置生成的，尤其是各个段的物理地址，虚拟地址等。

ReL4 中目前配置功能还不具备，简单根据各个 **平台** 定义的内存空间。下面是 spike 平台的 linker 脚本的主要内容

```
KERNEL_OFFSET = (0xFFFFFFFF80000000 + ((0x80000000 + 0x4000000) & ((1 << (30)) - 1))) - (0x80000000 + 0x4000000);
SECTIONS
{
    . = (0xFFFFFFFF80000000 + ((0x80000000 + 0x4000000) & ((1 << (30)) - 1)));
    .boot.text . : AT(ADDR(.boot.text) - KERNEL_OFFSET)
    {
        *(.boot.text)
    }
    .boot.rodata . : AT(ADDR(.boot.rodata) - KERNEL_OFFSET)
    {
        *(.boot.rodata)
    }
    .boot.data . : AT(ADDR(.boot.data) - KERNEL_OFFSET)
    {
        *(.boot.data)
    }
    .boot.bss . : AT(ADDR(.boot.bss) - KERNEL_OFFSET)
    {
        *(.boot.bss)
    }
    . = ALIGN(4K);
    ki_boot_end = .;
    .text . : AT(ADDR(.text) - KERNEL_OFFSET)
    {
        . = ALIGN(4K);
        /* Standard kernel */
        *(.text)
        *(.text.*)
    }
    /* Start of data section */
    _sdata = .;
    .small : {
        /* Small data that should be accessed relative to gp.  ld has trouble
           with the relaxation if they are not in a single section. */
        __global_pointer$ = . + 0x800;
        *(.srodata*)
        *(.sdata*)
        *(.sbss)
    }
    .rodata . : AT(ADDR(.rodata) - KERNEL_OFFSET)
    {
        *(.rodata)
        *(.rodata.*)
    }
    .data . : AT(ADDR(.data) - KERNEL_OFFSET)
    {
        *(.data)
        *(.data.*)
    }
    /* The kernel's idle thread section contains no code or data. */
    ._idle_thread . (NOLOAD): AT(ADDR(._idle_thread) - KERNEL_OFFSET)
    {
        *(._idle_thread)
    }
    . = ALIGN(4K);
    .bss . (NOLOAD): AT(ADDR(.bss) - KERNEL_OFFSET)
    {
        *(.bss)
        *(COMMON) /* fallback in case '-fno-common' is not used */
        /* 4k breakpoint stack */
        _breakpoint_stack_bottom = .;
        . = . + 4K;
        _breakpoint_stack_top = .;
        /* large data such as the globals frame and global PD */
        *(.bss.*)
    }
    ...
    ki_end = .;
}

```

其中包含的地址，如 0xFFFFFFFF80000000 0x80000000 0x4000000 等都是需要通过配置文件生成的，这部分在 reL4 中通过什么方式实现还需要讨论。

总的来说，没有什么特别的。值得说的就是 seL4 中将一部分代码和变量单独定义在了 boot.* section，reL4 会尽量和 seL4 保持相同，将这些代码也放在同样的段中，使其尽量靠近。

## 3. Startup 代码移植

该部分需要移植的代码主要包括一些汇编代码，和很少的 C 代码（大量的移植工作已经由其他同学完成了）。

汇编代码还是通过汇编实现，C 代码通过 Rust 重新实现。和 Linker 脚本系统相同，这些汇编代码同样是可配置的，暂时是通过 rust cfg + asm 的方式对这些汇编代码进行配置。

需要移植的代码如下（以 riscv spike 平台为例，aarch64 架构需要移植的汇编代码和 riscv 类似）

### 3.1 汇编代码

- **head.S**

head.S 是 kernel 的 entry 代码，主要是做一些寄存器初始化赋值，尤其是栈指针之类。完成后跳转到 init_kernel，最后通过 restore_user_context 回到用户态。

```
_start:
  fence.i
.option push
.option norelax
1:auipc gp, %pcrel_hi(__global_pointer$)
  addi gp, gp, %pcrel_lo(1b)
.option pop
  la sp, (kernel_stack_alloc + (1ul << (12)))
  csrw sscratch, x0
  jal init_kernel
  la ra, restore_user_context
  jr ra
```

- **trap.S**

在 riscv 架构上，trap.S 是 kernel 所有 trap 的入口，包括中断、异常等等。

trap_entry 的前一段保存用户空间寄存器上下文，之后根据 scause 寄存器内容选择 handle_syscall, c_handle_exception, c_handle_interrupt

由于 handle_syscall 中代码根据宏进行配置，所以使用 rust asm!() 实现，以方便使用 rust cfg() 特性。其他部分和 seL4 保持一致。

```
pub fn handle_syscall() {
    unsafe {
        asm!(
            "addi x1, x1, 4",
            "sd x1, (34*(64 / 8))(t0)",
            "li t3, -1",
            "bne a7, t3, 1f",
            "j c_handle_fastpath_call",
            "1:",
            "li t3, -2",
            "bne a7, t3, 2f"
        );
        
        // c_handle_fastpath_reply_recv need third parameter if open mcs option
        #[cfg(feature = "KERNEL_MCS")] {
            asm!("mv a2, a6");
        }

        asm!(
            "j c_handle_fastpath_reply_recv",
            "2:",
            "mv a2, a7",
            "j c_handle_syscall"
        );
    }
}
```

### 3.2 C 代码

尽管 ReL4 中已经将 SeL4 中绝大部分 C 代码使用 Rust 实现，但目前仍然有极少数的 **初始化阶段** 和 **CPU 架构相关的代码** 使用 C 实现。这些代码通常是架构之间不通用的，我们这里还是以 riscv 架构为例。

#### 3.2.1 Boot 阶段代码

Boot 阶段代码我们均放在 main.rs，主要是一个初始化函数和一些静态全局变量 (bss 段)

- **init_kernel**

init_kernel 是 head.S 中在完成基础的寄存器初始化后，调用的第一个函数。主要功能是

- Kernel 和 UserSpace 内存空间初始化
- 一些寄存器设置，比如中断入口寄存器，时钟中断使能等等
- 外设初始化，主要是 IRQ Controller
- 设备树加载
- rootserver 加载

总之就是使 kernel 做好准备，可以将控制权交给 rootserver

（本人对于初始化过程还不是很了解，可能有错漏）

```
#[no_mangle]
#[link_section = ".boot.text"]
pub fn init_kernel(
    ui_p_reg_start: usize,
    ui_p_reg_end: usize,
    pv_offset: isize,
    v_entry: usize,
    dtb_addr_p: usize,
    dtb_size: usize
) {
    sel4_common::println!("Now we use rel4 kernel binary");
    log::set_max_level(log::LevelFilter::Trace);
    boot::interface::pRegsToR(
        &avail_p_regs as *const p_region_t as *const usize, 
        core::mem::size_of_val(&avail_p_regs)/core::mem::size_of::<p_region_t>()
    );
    intStateIRQNodeToR(irqnode.0.as_ptr() as *mut usize);
    let result = rust_try_init_kernel(ui_p_reg_start, ui_p_reg_end, pv_offset, v_entry, dtb_addr_p, dtb_size);
    if !result {
        log::error!("ERROR: kernel init failed");
        panic!()
    }

    schedule();
    activateThread();
}
```

- **Boot 阶段变量**
  - **avail_p_regs**
  - **irqnode**

Boot 阶段使用的变量主要是一些内存范围的定义，这些内存范围在初始化时被赋值到一些静态变量，用于后续内存分配等等。这部分设计可以优化的地方很多，比如不应该从一个全局变量赋值到另一个全局变量，变量中的值是写死的等等。后面会进一步优化。


#### 3.2.2 Arch 相关代码

- **handle_syscall**

handle_syscall 在 seL4 中是汇编代码，但是由于其中有一些受配置选项影响的，所以使用 rust asm! + cfg 的方式实现配置功能，如下

```
#[cfg(feature = "KERNEL_MCS")]
{
    asm!("mv a2, a6");
}
```

- **c_handle_fastpath_call**

c_handle_fastpath_call 是一个 fastpath_call 的 wrapper，按照 seL4 C 代码实现即可。

- **c_handle_fastpath_reply_recv**

c_handle_fastpath_reply_recv 和 c_handle_fastpath_call 一样，是 fastpath_reply_recv 的 wrapper，稍有不同的是，分为 mcs 和 no mcs 两个版本。Rust 无法像 C 一样对任意一行使用 #ifdef，因此只能写两遍函数实现。

```
#[no_mangle]
#[cfg(feature = "BUILD_BINARY")]
#[cfg(not(feature = "KERNEL_MCS"))]
pub fn c_handle_fastpath_reply_recv(cptr: usize, msgInfo: usize) {
    use crate::kernel::fastpath::fastpath_reply_recv;
    fastpath_reply_recv(cptr, msgInfo);
}

#[no_mangle]
#[cfg(feature = "BUILD_BINARY")]
#[cfg(feature = "KERNEL_MCS")]
pub fn c_handle_fastpath_reply_recv(cptr: usize, msgInfo: usize, reply: usize) {
    use crate::kernel::fastpath::fastpath_reply_recv;
    fastpath_reply_recv(cptr, msgInfo, reply);
}
```

## 4. TODOs

- [x] 需要设计一套配置系统，实现根据配置生成所有架构和平台相关代码的功能，避免每个平台导入都通过手写代码的方式。

- [x] Pure Rust 模式运行 sel4test 比旧模式要慢，还需要进一步调查
- [x] aarch64 架构移植
- [x] 代码优化，避免冗余的全局变量

## 5. 移植遇到的坑

移植当中，遇到的问题记录如下

### **5.1 seL4 中预编译代码**

seL4 中使用大量的预编译代码，根据配置使用 cpp 命令生成实际的代码。即使是 linker 脚本和汇编代码也通过该方式生成。这部分无法直接移植，需要仔细分析其生成前后的文件，理解其含义。

### **5.2 rust 编译产生的符号所在的段和 C 有些区别**

比如 init_kernel 函数，默认会在 text.init_kernelxxxx 段，而不是 text 段。使用 seL4 原有的链接脚本时，会产生非常多的段（可以理解成每个函数会产生一个段），同时由于链接脚本中只定义了 text，data 等段放置的位置，这些 rust 编译产生的带后缀的段会被放在莫名其妙的位置，造成内存布局上 bss 段和 text 段的交叉，进一步造成修改 bss 段变量时会误修改 text 段数据。这是很严重的错误，会报非法指令（因为原有正常的指令都被改成 0 了）

```
# text.xxxxx 段的例子
  [214] .text._ZN9se[...] PROGBITS         ffffffff8401bd96  0001cd96
       0000000000000290  0000000000000000  AX       0     0     2
  [215] .text._ZN42_[...] PROGBITS         ffffffff8401c026  0001d026
       00000000000000b8  0000000000000000  AX       0     0     2
  [216] .text.map_it[...] PROGBITS         ffffffff8401c0de  0001d0de
       00000000000000c4  0000000000000000  AX       0     0     2
  [217] .text._ZN11s[...] PROGBITS         ffffffff8401c1a2  0001d1a2
       00000000000000e0  0000000000000000  AX       0     0     2
```

在链接脚本中，将 text.xxxxxx, bss.xxxxx 这些段全部归到 text, bss 段中即可解决该问题，例如

```
    .text . : AT(ADDR(.text) - KERNEL_OFFSET)
    {
        . = ALIGN(4K);
        /* Standard kernel */
        *(.text)
        # put text.xxxxx into .text section
        *(.text.*)
    }
```

### **5.3 pure rust 版本运行 rel4test 时间更长的问题**

定位到直接原因是执行 seL4_Untyped_Retype syscall 时间过长，整个调用链为

seL4_Untyped_Retype -> slowpath -> handleSyscall -> handleInvocation -> decode_invocation -> decode_untyed_invocation -> invoke_untyped_retype -> reset_untyped_cap

在 reset_untyped_cap 执行时间变长，根因是执行下面循环代码，每次都会执行时间更长一点

```
while offset != -(BIT!(chunk) as isize) {
    clear_memory(
        GET_OFFSET_FREE_PTR(region_base, offset as usize) as *mut u8,
        chunk,
    );
    prev_cap.set_capFreeIndex(OFFSET_TO_FREE_IDNEX(offset as usize) as u64);
    let status = unsafe { preemptionPoint() };
    if status != exception_t::EXCEPTION_NONE {
        return status;
    }
    offset -= BIT!(chunk) as isize;
}
```

其中主要影响是 preemptionPoint() ，这个函数执行不知道为什么时间变长。经过比较，发现其中的一个全局变量 **ksWorkUnitsCompleted** 放置的位置和之前不同。

- 旧版本变量位置

```
# 位于 boot.bss 段
ffffffff84001d10 D ksWorkUnitsCompleted 
```

因此我将 ksWorkUnitsCompleted 在 pure rust 版本中指定放在 boot.bss 段，解决了运行很慢的问题。

```
#[no_mangle]
#[link_section = ".boot.bss"]
pub static mut ksWorkUnitsCompleted: usize = 0;
```

虽然解决了该问题，但是不知道为什么这会造成这么大的影响。也许是缓存未命中？