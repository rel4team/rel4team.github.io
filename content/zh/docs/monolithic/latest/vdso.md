---
weight: 13
title: "vdso 支持"
commentsId: 1 
---

# VDSO 支持

## VDSO 介绍

vDSO（Virtual Dynamic Shared Object，虚拟动态共享对象）是 Linux 内核提供的一种机制，用来高效地将某些内核服务暴露给用户空间，而无需进行用户态到内核态的系统调用（syscall）切换。

## Musl 中的 vdso for arm 支持

在 Musl 中，加载 vdso 需要内核在 exec 的时候向 libc 中传递对应的 AuxVector 参数 `SysInfoEhdr`，这个地址指向 vdso 在内存中映射的虚拟地址，在启动的时候 libc 库就会将自动解析 vdso 库，在调用系统函数的时候如果发现这个函数支持 vdso 调用且 vdso 被正确的加载了，那么就会从 vdso 中调用而不是发送系统调用。下面是 vdso 在 musl 中的的定义(arm)。


```c
#define VDSO_USEFUL
#define VDSO_CGT32_SYM "__vdso_clock_gettime"
#define VDSO_CGT32_VER "LINUX_2.6"
#define VDSO_CGT_SYM "__vdso_clock_gettime64"
#define VDSO_CGT_VER "LINUX_2.6"
#define VDSO_CGT_WORKAROUND 1
```

## 如何实现一个 vdso 

为了实现一个 vdso，首先需要了解 vdso 支持哪些函数，目前在 musl 中 vdso 仅支持 gettime 函数，因此在 rel4-linux-kit 中首先要编写一个同名函数来实现 `gettime` 函数。这个函数需要能够在用户态执行，arm 支持通过设置在用户态读取时钟 cntp, 这样就可以在用户态实现读取准确的时间而不用进入内核态。下面是 rel4-linux-kit 实现的 `gettime` 函数：

```c
#include <stdint.h>

// musl中用来保存时间的结构体
struct timespec
{
    uint64_t tv_sec;
    long tv_nsec;
};

struct TimeSpec
{
    uint64_t secs;
    uint32_t nanos;
    uint32_t _align;
};


// 获取内核时间
int __kernel_clock_gettime(uint64_t _id, struct timespec *tp)
{
    uint64_t cntpct, cntfrq;
    asm volatile("mrs %0, CNTPCT_EL0": "=r"(cntpct));
    asm volatile("mrs %0, CNTFRQ_EL0": "=r"(cntfrq));
    
    uint64_t sec = cntpct / cntfrq;
    uint32_t nsec = (cntpct % cntfrq) * 1000000000 / cntfrq;
    tp->tv_sec = sec;
    tp->tv_nsec = nsec;
    return 0;
}
```

上述代码实现了读取 cntp 中的时间，并根据时钟的频率转换为时间戳从而修改参数。这个时候，一个基本的 vdso 就已经成型了，然后就需要对 vdso 进行编译，这个过程看起来很复杂，其实整个操作非常简单。仅需要寻找对应架构的 gcc 编译成一个 so 文件。下面是 rel4-linux-kit 中的编译指令：

```shell
CC := aarch64-linux-musl-gcc

all: vdso

vdso: src/vdso.c
	$(CC) -fPIC -shared -nostdlib -o vdso.so $< -T linker.ld
```

这样就编译出一个 vdso 文件，后面在 exec 的时候完成加载即可。

## vdso 的加载

为了提高执行多个程序的时候 vdso 的加载速度，rel4-linux-kit 实现了 vdso 的预加载，即先加载 vdso 到内存中，在 exec 的时候复制相应的 cap 然后映射到对应的内存。

```rust
{
    vdso::init_vdso_addr();
    let vdso = File::open("/vdso.so", OpenFlags::RDONLY).unwrap();
    let vdso_size = vdso
        .read(unsafe { core::slice::from_raw_parts_mut(VDSO_KADDR as _, VDSO_AREA_SIZE) })
        .unwrap();
    assert!(vdso_size > 0);
}
```

当加载 ELF 文件到进程地址空间时，load_elf() 函数会将 VDSO 页面映射到用户空间。`VDSO_APP_ADDR` 作为 vdso 在用户空间放置的地址。