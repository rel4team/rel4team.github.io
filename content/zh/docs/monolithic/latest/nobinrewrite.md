
---
weight: 18
title: 'Unknown Syscall 捕捉'
commentsId: 1 
---

# Unknown Syscall 捕捉

## 背景

在当前版本的 rel4-linux-kit 中所采用的支持 Linux App 的方式是利用二进制重写将 Syscall 指令 （arm 上的是 svc）重写为 `0xdeadbeef`(一条错误指令)，在程序运行的时候就会触发 InstructionFault，在 sel4 中为 `CpuFault` 或 `VCpuFault`(hypervisor=on)，然后在错误处理程序中查找当前执行的命令，如果是 `0xdeadbeef` 就会进入 SysCall 处理逻辑，否则就作为异常指令处理。

但是这种方式会对程序进行修改，会对程序的签名产生破坏（比如 md5 checksum），为了解决这个问题，可以采用一些其他的方式进行处理，比如，在 LinuxApp 中正常调用系统调用，由于 sel4 的系统调用号为 -1 开始的反向增长，所以几乎不会和 Linux 的系统调用产生冲突，这样就可以让 sel4 将错误转发给 LinuxApp 的 FaultHandler 进行处理。不过由于 sel4 在 aarch64 中使用 x7 作为 syscall id reg，LinuxApp 使用 x8 作为 syscall id reg, 所以 LinuxApp 在调用的时候可能会在 x7 寄存器中产生一些数值导致误调用了 sel4 的系统调用，在测试的过程中也确实发生了这样的情况，在 LinuxApp 发起系统调用的时候 x7 寄存器并不一直为 0。

所以对于 aarch64 来说的 sel4 来说，想要利用 Unknown Syscall 的方式来支持 LinuxApp 的运行，就需要对 libc 做一些修改，让程序能够在编译前完成寄存器的替换。因此 rel4-linux-kit 就对应修改了 musl 的库，并对应重新编译了 gcc 工具链。

## 编译测例

> 编译测例需要使用修改过后的 gcc 进行编译

在编译 lmbench 的时候需要注意，由于 lmbench_all 使用了 glibc 的 queue.h 库，但是 musl 默认是没有提供这个库的，**所以需要在本机安装 gnu gcc，然后将这个库复制到工具链的 includes/sys 目录下**。

## 相关资源

- 修改过后的 musl <https://github.com/rel4team2/musl>
- 修改过后的 gcc <https://github.com/reL4team2/musl-cross-make>
  - 预编译版本 <https://github.com/reL4team2/musl-cross-make/releases>
- 运行的测例 <https://github.com/reL4team2/testsuits-for-oskernel>
  - 预编译的测例 <https://github.com/reL4team2/rel4-linux-kit/releases/tag/without-binary-rewrite>
- 支持 Unknown Syscall 捕捉的 rel4-linux-kit <https://github.com/reL4team2/rel4-linux-kit/tree/without-binary-rewrite>

