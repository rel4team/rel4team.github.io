+++
date = '2025-01-08T14:30:15+08:00'
draft = true
title = '启动代码和编译系统移植'
+++

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

需要移植的代码如下（以 riscv spike 平台为例）

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

尽管 ReL4 中已经将 SeL4 中绝大部分 C 代码使用 Rust 实现，但目前仍然有极少数的初始化阶段的代码使用 C 实现。这些代码通常是架构之间不通用的，我们这里还是以 riscv 架构为例。

- **init_kernel**

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

## 4. 编译系统及新旧版本兼容

## 5. TODOs

## 6. 移植遇到的坑

