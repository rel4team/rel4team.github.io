---
weight: 14
title: "aarch64 用户态时钟读取机制"
commentsId: 1 
---

# aarch64 时钟读取机制

rel4-linux-kit 使用 ARM Generic Timer 作为硬件时钟，通过 aarch64 的系统控制器控制，arm cntp 支持通过设置在用户态访问，这样就解决了在用户态的内核访问时钟和设置定时器的问题。为了实现这个支持需要在 sel4 中开启 KernelArmExportPCNTUser 支持，开启后可以在用户态访问 `cntpct_el0` 寄存器和 `cntpfrq_el0` 寄存器读取 ticks 和定时器的工作频率。

## 时间转换

在 rel4-linux-kit 中，时间转换首先通过 `cntpct_el0` 寄存器读取到当前的 ticks，然后在 `cntpfrq_el0` 寄存器中读取 timer 的工作频率，将滴答数转换为真实的时间。最终通过 Duration::new() 将秒数和纳秒数组合成 Rust 的 Duration 类型。使用 Duration 可以保证在多个环节中能够有一个更加通用的类型以确保不会有过多的转换导致的精度和性能损耗。

```rust

/// 获取当前的时间(ns)
#[cfg(target_arch = "aarch64")]
#[inline]
pub fn current_time() -> core::time::Duration {
    let cnt: usize;
    let freq: usize = get_freq();
    unsafe {
        core::arch::asm!("mrs  {}, cntpct_el0", out(reg) cnt);
    }
    core::time::Duration::new((cnt / freq) as _, ((cnt % freq) * NS_PER_SEC / freq) as _)
}

/// 获取当前的时钟频率
#[inline]
fn get_freq() -> usize {
    let freq: usize;
    unsafe {
        core::arch::asm!("mrs  {}, cntfrq_el0", out(reg) freq);
    }
    freq
}
```
