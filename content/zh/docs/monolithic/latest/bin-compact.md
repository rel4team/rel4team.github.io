---
weight: 2
title: "二进制兼容"
commentsId: 1 
---

# 二进制兼容

## 背景

我们希望在用户态运行 `Linux` 应用程序，但首先需要解决一个关键问题：如何接收并处理 `Linux` 用户程序发出的系统调用。通常情况下，用户态程序会通过系统调用指令进入内核态，随后内核会解析系统调用号并执行相应的处理。然而，在 `reL4` 之上的 **用户态内核**这一特殊环境下，情况有所不同：**用户态内核**和 `Linux` 用户程序都运行在用户态，这意味着传统的系统调用处理机制无法直接适用。那么，在这种架构下，我们该如何捕获并处理系统调用呢？

在 [v1]({{< ref "/docs/monolithic/v1/architecture.md" >}}) 版本的设计中，我们选择对用户态程序进行重新编译，以实现对系统调用的拦截和重定向。具体而言，我们使用一个变量记录系统调用的目标地址，并在原本执行系统调用的地方，将其替换为跳转到该变量指向的地址。当**用户态内核**启动该程序时，它会修改这个变量的值，使其指向一个特定的 `vsyscall` 函数。该函数的作用是拦截系统调用并将其转换为 IPC（进程间通信），然后将请求发送给**用户态内核**进行处理。这样设计会面临一些问题：

1. **需要获取应用程序的源代码**：必须重新编译应用程序，无法直接运行现有的 `Linux` 二进制文件，这大大限制了适用范围。
2. **依赖用户态内核修改特定变量**：**用户态内核**需要在启动应用程序时修改存储系统调用地址的变量，增加了额外的管理和维护成本。
3. **需要为应用程序设计特殊的转发函数**：`vsyscall` 需要负责拦截并转发系统调用，而不同的应用可能对系统调用的使用方式不同，这可能会引入兼容性问题。

虽然我们也探索过一些更通用、性能更高的的设计方案[^1]，但考虑到实现的复杂性、开发工作量以及人力资源的限制，最终我们没有选择这些设计方案。

## rel4/sel4 错误处理机制

> 我们的 `reL4` 是 seL4 的 Rust 版本，在设计上保持了与 seL4 一致的 系统调用接口和机制，确保了兼容性和功能的对等性。

在 seL4 中，内核不会直接处理用户态程序的异常，而是将其交由用户空间处理。

seL4 允许用户为每个线程设置一个异常端点（endpoint），当线程发生异常时，内核会向该端点发送一条消息，通知异常情况[^2]。负责异常处理的通常是另一个用户态线程，它接收异常报告并决定如何处理，例如调整程序状态、执行修复操作或终止进程。

换句话说，我们可以将系统调用更改为一条异常指令，通过异常端点捕获并处理它们，而不必依赖传统的内核态系统调用处理机制。这样，我们可以在**用户态内核**中模拟类似 `Linux` 内核 系统调用处理流程，从而支持 `Linux` 用户程序的运行。

在这个场景下我们的**用户态内核**为用户程序设置异常端点，并为用户程序提供异常处理程序。

## 具体设计

因此，我们采用了将 **系统调用指令** 替换为 **异常指令** 的方案，使得内核能够向**用户态内核**发送 `Fault IPC`。在**用户态内核**接收到异常消息后，会检查触发异常的指令是否为我们指定的 特殊错误指令。

如果是，我们就将其视为系统调用请求并进行相应处理；如果不是，则认为这是一个真正的异常，按照正常的异常处理流程进行处理。这种方法有效地利用了 `seL4` 的异常机制，实现了系统调用的捕获与处理。

对于我们当前的设计，只需 **程序的二进制文件** 即可进行操作，而无需引入额外的组件。这种方式在编程和兼容性方面都具有显著的优势：

- **简化了部署过程**：用户只需提供现有的二进制文件，无需修改源代码或重新编译，降低了使用门槛;
- **提高了兼容性**：由于不依赖于特定的源代码或编译过程，更多的应用程序可以在我们的环境中无缝运行;
- **减少了开发工作量**：不用再为用户态 `vsyscall` 设计特殊的函数来将系统调用转换为 `IPC`。

### 设置用户态程序的异常端点

**用户态内核**在启动用户程序的时候为其设置异常端点[^3]，下面代码中的 `DEFAULT_PARENT_EP` 就是我们设置的异常端点，**这里传递的只是一个数字，表示发生异常时调用用户程序的哪个 EndPoint 发送异常 IPC**。

```rust
// 配置子任务
task.tcb.tcb_configure(
    CPtr::from_bits(DEFAULT_PARENT_EP),
    task.cnode,
    CNodeCapData::new(0, sel4::WORD_SIZE - CNODE_RADIX_BITS),
    task.vspace,
    ipc_buffer_addr,
    ipc_buf_page.cap(),
)?;
```

### 替换系统调用指令为特殊指令

为了使原本执行系统调用的地方能够触发异常并发送 Fault IPC，我们需要将 系统调用指令 替换为特定的指令。为此，我们选择使用 Python 编写该程序，借助 LIEF[^4] 库和 Capstone[^5] 库的强大功能。

LIEF 库可以帮助我们快速修改 ELF 文件，提供了对 ELF 格式的访问和操作功能，简化了二进制文件的修改过程。而 Capstone 库则能够高效地进行指令的反汇编和分析，使我们能够快速查找并修改特定的指令。

这两个库使我们能够以较少的代码实现指令的分析和修改，从而高效地完成替换工作。通过这种方式，我们可以自动化地处理二进制文件，确保系统调用指令被正确替换为我们的特殊指令，从而触发所需的异常处理机制。

```python
#!/usr/bin/python3
from capstone import *
import lief
import os
import sys

if len(sys.argv) <= 2:
    print("Usage: python3 modify.py <src> <dst>")
    exit(0)

src = sys.argv[1]
dst = sys.argv[2]
print("src " + src)
print("dst " + dst)

elf = lief.ELF.parse(src)

text_section = elf.get_section(".text")
text_data = list(text_section.content)  # 获取 `.text` 段的二进制数据

md = Cs(CS_ARCH_ARM64, CS_MODE_ARM)
md.disasm(bytes(text_data), text_section.virtual_address)

for ins in md.disasm(bytes(text_data), text_section.virtual_address):
    if ins.mnemonic == "svc":
        offset = ins.address - text_section.virtual_address
        text_data[offset : offset + 4] = (0xDEADBEEF).to_bytes(4, byteorder="little")

# 写回修改后的 .text 数据
text_section.content = text_data  # 关键点：重新赋值给 LIEF 结构

# 保存修改后的 ELF
elf.write(dst)
```

上述程序[^6]将 `svc` 指令替换为一个不存在的指令 `0xdeadbeef`。

### 处理异常

> **用户态内核**的 `SREVE_EP` 和 用户程序的 `PARENT_EP` 是一对可以互相通信的端点。

完成上述操作后，我们就可以将用户程序加载到内存中运行，当运行到我们替换的 `0xdeadbeef` 时，内核就会向**用户态内核**的 `SERVE_EP` 发送一个 `Fault IPC`，下面是我们监听异常处理的部分代码。

```rust
/// 循环等待并处理异常
pub fn waiting_and_handle() -> ! {
    loop {
        let (message, tid) = DEFAULT_SERVE_EP.recv(());
        assert!(message.label() < 8, "Unexpected IPC Message");

        let fault = with_ipc_buffer(|buffer| Fault::new(&buffer, &message));
        match fault {
            Fault::UserException(ue) => handle_user_exception(tid, ue),
            ...
        }
    }
}
```

在这个代码示例中，我们定义了一个 `waiting_and_handle` 函数来监听并处理接收到的 `Fault IPC` 消息。根据消息中包含的指令，这里对于处理系统调用来说比较重要的是 `userexception`，当用户程序运行非法指令的时候发送的就是 `UserException`。

在 `handle_user_exception` 函数中，我们完成对于当前异常是否由系统调用产生的判断，并对系统调用进行处理。

```rust
/// 处理用户异常
///
/// - `tid` 是用户进程绑定的任务 ID
/// - `vmfault` 是发生的错误，包含错误信息
///
/// 函数描述：
/// - 异常指令为 0xdeadbeef 时，说明是系统调用
/// - 异常指令为其他值时，说明是用户异常
pub fn handle_user_exception(tid: u64, exception: UserException) {
    let mut task_map = TASK_MAP.lock();
    let task = task_map.get_mut(&tid).unwrap();
    let ins = task.read_ins(exception.inner().get_FaultIP() as _);

    // 如果是某个特定的指令，则说明此次调用是系统调用
    if Some(0xdeadbeef) == ins {
        let mut user_ctx = task
            .tcb
            .tcb_read_all_registers(true)
            .expect("can't read task context");
        let result = handle_syscall(task, &mut user_ctx);
        debug!("\t SySCall Ret: {:x?}", result);
        ...
    } else {
        debug!("trigger fault: {:#x?}", exception);
    }
}
```

我们在异常处理程序中提供了完整的代码实现[^7]。

## 案例演示

下面的演示基于以下环境：

- 仓库：https://github.com/reL4team2/rel4-linux-kit.git
- commit: 167ba19

### 环境安装

请确保您已安装了 `rust` 工具链。

#### 下载代码

```shell
git clone https://github.com/reL4team2/rel4-linux-kit.git
cd rel4-linux-kit
git checkout 167ba19
# 下载 sel4 内核
wget -qO- https://github.com/yfblock/rel4-kernel-autobuild/releases/download/release-2025-01-08/seL4.tar.gz | gunzip | tar -xvf - -C .env --strip-components 1
```

#### 安装 python 依赖

```shell
pip install capstone lief
```

#### 安装 aarch64-linux-musl

> 如果您有自己的工具链也可以使用，我们目前的 syscall 仅有对于 musl 的支持。

```shell
wget https://musl.cc/aarch64-linux-musl-cross.tgz
tar zxf aarch64-linux-musl-cross.tgz
rm aarch64-linux-musl-cross.tgz
export PATH=$PATH:`pwd`/aarch64-linux-musl-cross/bin
```
### 编译修改 elf 文件

您可以直接使用我们在 `examples/linux-apps` 下的测例，这里我们使用 `sigtest` 测例。

```shell
# 编译测试程序，使用的是环境变量中提供的 aarch64-linux-musl
make -C examples/linux-apps/sigtest
# 修改可执行文件的系统调用指令
./tools/ins_modify.py examples/linux-apps/sigtest/main.elf .env/busybox-ins.elf
```

这里为什么是 `.env/busybox-ins.elf` 文件，目前这个版本我们还没有完善支持文件系统，所以可执行文件暂时内联，直接被打包进内核运行，而 `.env/busybox-ins.elf` 就是我们默认的内联文件名称。

在执行之后我们就能看到运行结果。

### 动态二进制兼容

上述的 python 脚本能够实现对于一个 elf 文件的修改，这种方式需要提前在宿主机上修改 elf 文件，流程增多，因此在 rel4-linux-kit 上基于上述原理实现了在 rust 代码中 load_elf 阶段对 syscall 执行的修改。

```rust
    /// 加载一个 elf 文件到当前任务的地址空间
    ///
    /// - `elf_data` 是 elf 文件的数据
    pub fn load_elf(&self, file: &File<'_>) {
        // 加载程序到内存
        file.sections()
            .filter(|x| x.name() == Ok(".text"))
            .for_each(|sec| {
                #[cfg(target_arch = "aarch64")]
                {
                    const SVC_INST: u32 = 0xd4000001;
                    const ERR_INST: u32 = 0xdeadbeef;
                    let data = sec.data().unwrap();
                    let ptr = data.as_ptr() as *mut u32;
                    for i in 0..sec.size() as usize / size_of::<u32>() {
                        unsafe {
                            if ptr.add(i).read() == SVC_INST {
                                ptr.add(i).write_volatile(ERR_INST);
                            }
                        }
                    }
                }
                #[cfg(not(target_arch = "aarch64"))]
                log::warn!("Modify Syscall Instruction Not Supported For This Arch.");
            });
    ...
}
```

上述代码实现了在加载 elf 文件的时候查询 svc 指令并将 svc 指令修改为一条错误指令。

# 引用链接

[^1]: 在用户态将 Syscall 转换为 IPC Call: https://github.com/reL4team2/rel4-linux-kit/discussions/3
[^2]: sel4 Fault Handling: https://docs.sel4.systems/Tutorials/fault-handlers.html
[^3]: 设置用户程序: https://github.com/reL4team2/rel4-linux-kit/blob/167ba1991a918b73178085864190b08bcb8d296e/services/kernel-thread/src/child_test.rs#L53C5-L61C8
[^4]: LIEF: https://lief.re
[^5]: Capstone: https://pypi.org/project/capstone/
[^6]: 指令替换程序: https://github.com/reL4team2/rel4-linux-kit/blob/main/tools/ins_modify.py
[^7]: 异常处理： https://github.com/reL4team2/rel4-linux-kit/blob/main/services/kernel-thread/src/exception.rs