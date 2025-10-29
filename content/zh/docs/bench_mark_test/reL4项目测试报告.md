# reL4项目测试报告

## 1. 概述

本报告旨在对reL4微内核和reL4-Linux-kit兼容层进行系统性的评估。评估工作主要围绕两部分展开：一是对系统功能完整性的验证，二是对系统综合性能的评测。

### 1.1 测试目标

- reL4内核与seL4内核的二进制兼容情况分析
- reL4-Linux-kit与部分关键Linux syscall的兼容性情况分析
- reL4-Linux-kit对调用相关Linux syscall的测例集合的运行情况分析
- 基于seL4或者reL4的reL4-Linux-kit运行情况的综合性能评估

## 2. 功能实现评估

### 2.1 reL4对seL4二进制接口的功能实现评估

reL4的基本目标为实现一个兼容seL4接口的Rust内核，通过seL4test测试套件验证功能正确性。

seL4test是seL4官方实现的原生测例，总共有241个测例，这些测例针对seL4内核中不同的syscall接口进行测试，用于测试内核是否被正确地实现了。我们以这些测例作为标准，测试reL4的接口的实现是否和seL4兼容。

在reL4现有实现的feature下能够支持的测例分类及其通过情况如**附录A**中所示。在附录A中列举的测例分类中，有部分测例本身虽然存在但无需测试，除此之外，reL4已经支持的feature所需的测例均可以正常通过。

在表1中列举了reL4中无需测试的测例。如EPT和IO Port特性相关的测例主要是在x86下才可以支持的，因此，在目前仅仅支持aarch64和riscv64的reL4中，无需通过，其他的测例也是类似，大多是由于目前reL4项目所涉及的硬件特性不足所无需支持。

**表1：reL4未测试的测例分类及其数量**

| 测例分类        | 测例个数 |
| :-------------- | :------- |
| Smp Irq         | 1        |
| SMMU            | 1        |
| Stack Alignment | 1        |
| EPT             | 13       |
| Benchmark       | 1        |
| Breakpoint      | 7        |
| Break Request   | 1        |
| Single Step     | 1        |
| Cache Flush     | 4        |
| Unknown Syscall | 1        |
| IO Ports        | 1        |
| IOPT            | 11       |
| vcpu            | 1        |
| **总计**        | **44**   |

### 2.2 reL4-Linux-kit对Linux syscall的实现评估

reL4-Linux-kit的设计目标为兼容现有的部分Linux用户态程序，但是，这些Linux程序所调用的Linux syscall，只是所有的Linux syscall的一部分，换而言之，只需要实现部分的Linux syscall，即可保证兼容性。

在**附录B**中列举了reL4-Linux-kit所支持的Linux syscall的数量，这些syscall基本构成了一个Linux系统所需的syscall集合。根据其功能的不同，可以分为进程管理、内存管理、权限管理、设备控制、时间管理、事件管理、文件系统、系统信息、信号处理、资源管理等10类共69个具体syscall。其中部分syscall使用了fake实现，即仅仅返回Ok(0)，而不做任何实际的处理。

基于上述实现的syscall集合，我们采用了一些测例进行功能正确性的验证，主要包括basic、run-static、busybox-testcase、lua、iozone、libc、lmbench等测试集合。在附录C中列举了上述测例集合所使用的syscall，这些测例集合基本能够涵盖我们实现的syscall集合，只有两个在reL4-Linux-kit中实现了的syscall并未被这些测例集合使用，这两个测例在表2中列举。

**表2：已实现但未被Linux测例使用的系统调用**

| 系统调用名称  | 类别     | 是否为fake实现 |
| :------------ | :------- | :------------- |
| getgid        | 权限管理 | 是             |
| clock_gettime | 时间管理 | 否             |

最后，必须指出，在我们的测例集合中，还有部分syscall被调用到了，但是并没有在reL4-Linux-kit中实现，这些syscall及其含义在表3列举。这些被调用但未实现的syscall主要集中在网络管理模块，由于reL4-Linux-kit并未实现相应的网络模块，因此这些syscall都无法实现，同时，在run-static测例集合中对应的socket测例也会相应的报错，我们忽略这部分的报错，其余少数未实现的系统调用因不影响测试核心结果的正确性，故在当前阶段予以保留。

**表3：Linux测例需要但未实现的系统调用**

| 系统调用名称    | 类别     | 简要描述               | 使用的测例组合                                           |
| :-------------- | :------- | :--------------------- | :------------------------------------------------------- |
| exit-group      | 进程管理 | 终止进程组中的所有线程 | run-static, busybox-testcase, lua, iozone, libc, lmbench |
| madvise         | 内存管理 | 提供内存使用建议       | iozone, libc                                             |
| sysinfo         | 系统信息 | 获取系统信息           | lua                                                      |
| set-robust-list | 进程管理 | 设置健壮互斥锁列表     | run-static                                               |
| umask           | 文件系统 | 设置文件模式创建掩码   | lmbench                                                  |
| accept          | 网络管理 | 接受套接字上的连接     | run-static                                               |
| bind            | 网络管理 | 将名称绑定到套接字     | run-static                                               |
| connect         | 网络管理 | 在套接字上发起连接     | run-static                                               |
| getsockname     | 网络管理 | 获取套接字名称         | run-static                                               |
| listen          | 网络管理 | 监听套接字上的连接     | run-static                                               |
| recvfrom        | 网络管理 | 从套接字接收消息       | run-static                                               |
| sendto          | 网络管理 | 向套接字发送消息       | run-static                                               |
| setsockopt      | 网络管理 | 设置套接字选项         | run-static                                               |
| socket          | 网络管理 | 创建通信端点           | run-static                                               |

基于seL4内核和基于reL4内核的reL4-Linux-kit目前均能够正确完成上述的Linux下的测例，这个结果更进一步地证明了reL4内核在实现上的正确性，即reL4内核能够二进制兼容seL4内核，同时也能够说明，reL4-Linux-kit的现有实现能够正确支持Linux下的程序的运行。

## 3. reL4性能评估

对于reL4相关的性能评测，目前seL4内核项目也已经实现了seL4bench的测例，用于测试seL4的各类性能。为了支持这些测试，必须在reL4项目中增加seL4bench相关syscall和一些硬件环境的支持。

### 3.1 One Way IPC测试

这里的IPC相关测试主要指的是单向的IPC的性能。seL4bench的相关测试并没有支持MCS特性，因此MCS相关syscall的测试并没有支持。

seL4bench中单向IPC性能测试涉及到的syscall包括：SYS_CALL、SYS_SEND、SYS_REPLY_RECV这三个syscall。在该测试中的相关测试还对比了IPC的两个线程是否在同一个地址空间下，以及不同IPC消息长度下的差别。

![](benchmark/one_way_ipc_figure_1_bar_chart.png)
![](benchmark/one_way_ipc_figure_2_bar_chart.png)
![](benchmark/one_way_ipc_figure_3_bar_chart.png)

测试结果揭示了以下现象：reL4在各个场景下都缺少IPC的优化，采用了统一的syscall的路径；在seL4下的同一地址空间的IPC具有良好的优化，其开销的Cycles为reL4下同样One Way IPC的五分之一左右；在不同地址空间下，同样为0B长度的IPC消息发送，不同的syscall接口也展现出了不同的性能；在长度为10B的IPC消息发送中，reL4和seL4虽然仍然存在一些性能上的差异，但是差异不大。

对此，后续的工作需要着眼于单向IPC测试中相关的系统调用路径，尤其是fastpath及其优化实现。

### 3.2 中断相关测试

对于reL4和seL4的中断相关性能的对比测试结果表明：seL4的irq，无论是否存在context switch，其波动范围都较小，具有很好的稳定性。reL4的irq稳定性则远不如seL4，其波动范围更大，极值发生概率更高，但是其均值在存在context switch和无context switch的两种情况下，基本相近，可以认为其出现了跟前一节同样的缺少优化的现象。

![](benchmark/irq_performance_comparison.png)

值得注意的是，在所有其他的测试中，均能表明reL4目前的实现，性能相比seL4存在一定的差距，但是，在本测试中，存在context switch的情形下，seL4进行IRQ处理需要的Cycles比reL4进行IRQ处理需要的Cycles更多。

可以认为reL4的测试依然没有对相关路径进行优化，只有这样才能解释reL4的irq处理需要的Cycles基本接近且远高于seL4的irq处理所需的Cycles。而存在context switch情形下，该数值低于seL4下的数值，应当可以认为使用Rust编写的代码在性能上经过路径优化之后，有一定潜力优于C编写的seL4。

### 3.3 信号相关测试

![](benchmark/signal_high_prio_comparison.png)
![](benchmark/signal_low_prio_comparison.png)

信号机制同样是seL4下重要的消息发送机制，对于信号相关测试的结果表明：在向高优先级进程发送信号的时候，平均延迟的Cycles呈现出了急剧的增长。在这两个测试结果中，seL4下的测试结果均展现出了很好的稳定性和比reL4更好的性能。

### 3.4 同步相关测试

同步相关测试主要目的是为了测试通过seL4/reL4的通信机制实现消息同步所带来的开销，相比于单纯的One Way IPC性能，该测试提供了更加全面的系统消息传递开销评估。

![](benchmark/wait_time_line_chart.png)

从测试结果来看，随着消息等待者的数量增长，在reL4和seL4下的平均等待时间均呈现下降趋势。随着消息等待者的数量上升，整体的消息发送和接收的时间一定是增长的，因为更多的消息需要发送和接收。而每个消息接收者存在从发送消息之后，到被调度出来的时延，因此在消息接收者数量少的时候，消息接收者必须等待其他进程被调度出来并执行，这增加了其时延。当消息接收者数量多的时候，一个消息接收者接收到消息之后，下一次调度的进程依然是一个消息接收者，因此其平均接收时间一定会逐步下降。

在测试中，seL4内核在所有测试结果中，均展现出来比reL4更好的性能和稳定性。考虑到reL4的One Way IPC章节所提出的可能，由于可能存的在syscall代码调用路径上的优化不足，在该测试中存在这样数量级别的性能差距，是可以理解的。

### 3.5 映射相关测试

映射相关测试主要针对于系统的内存映射性能的评估，测试项目包括：Prepare Page Tables、Allocate Pages Mapping、Mapping、Propect Pages as Read Only、Unprotect Pages。

![](benchmark/mapping_benchmark_grid_large.png)

从测试结果来看，reL4和seL4之间表现出来的性能行为基本一致，基本可以说明，reL4在功能上可以对标seL4内核，但是存在一个性能gap（请注意坐标轴为对数坐标轴），这在几乎所有的测试中都是存在的。考虑到之前的分析，这样的性能差距是可以理解的，并且是系统性的差距。

## 4. reL4-Linux-kit综合性能评估

对于reL4-Linux-kit的测试，由于其可以基于seL4或者reL4内核运行，因此，可以对比其在reL4和seL4上的执行效率来评测reL4-Linux-kit。同时，由于其支持Linux syscall，也可以将reL4-Linux-kit在reL4上的性能，对比原生Linux的执行性能。

对于reL4-Linux-kit的测试内容，主要包括libc，busybox，lua，iozone，lmbench等测试。除此之外，函数调用转IPC的设计，会导致不同模块合并成一个模块的行为会影响其性能，对此也需要做评估。我们只评测所有可拆分的模块均进行拆分，以及所有模块均进行合并的结果，而不是穷举所有模块的组合可能。

我们评测的平台配置如下：

- CPU: AMD Ryzen 9 9950X3D (32) @ 5.756GHz
- OS: Ubuntu 24.04.3 LTS x86-64
- Kernel: Linux 6.14.0-28-generic
- Memory: 96GB

测试结果如表4所示。

**表4：Linux测例在不同操作系统配置下的性能测试结果（单位：秒）**

| 测例类型         | Linux内核测例运行时间 | seL4内核reL4-Linux-kit(合并模块) | seL4内核reL4-Linux-kit(拆分模块) | reL4内核reL4-Linux-kit(合并模块) | reL4内核reL4-Linux-kit(拆分模块) |
| :--------------- | :-------------------- | :------------------------------- | :------------------------------- | :------------------------------- | :------------------------------- |
| basic            | 1.133                 | 1.555                            | 1.572                            | 1.723                            | 1.737                            |
| run-static       | 6.210                 | 4.252                            | 4.337                            | 4.945                            | 5.083                            |
| busybox-testcase | 1.270                 | 2.995                            | 3.038                            | 3.397                            | 3.484                            |
| lua              | 0.093                 | 0.760                            | 0.783                            | 0.832                            | 0.854                            |
| iozone           | 84.563                | 149.439                          | 157.940                          | 166.381                          | 183.172                          |
| libc             | 4.283                 | 13.719                           | 13.862                           | 18.296                           | 18.095                           |
| lmbench          | 99.260                | 60.494                           | 60.102                           | 63.552                           | 62.057                           |

对于表4的测评结果，可以清晰的看到基于seL4/reL4内核的reL4-Linux-kit和Linux内核之间具有较大的性能差距。微内核的几个测例中，以seL4作为内核，并将所有服务组件拼合成为单个服务组件的方式运行时，性能与Linux差距最小。由于取消了系统服务组件间的IPC，因此该测例下与Linux的性能差距主要来源于两个地方，一个是seL4截获用户态Linux app的syscall并转发给用户态系统服务组件，与直接进行syscall进入宏内核的性能差距，这是使用我们所设计的syscall转IPC方法无可避免的一个开销；另一个是用户态系统服务组件本身和Linux执行的性能差距，Linux作为一个经典的宏内核实现，其具有非常良好的优化，而reL4-Linux-kit仅仅作为验证兼容系统可行性的一个尝试。因此，seL4下的reL4-Linux-kit和原生Linux下存在如此巨大的性能差距，实属正常。

其次，对于我们设计的函数调用和IPC调用之间的透明转换机制，分别在reL4内核和seL4内核的基础上进行了评测。这两者之间的性能差距纯粹是来自于函数调用和IPC调用方式之间的差距，在同一个内核下，两者差距始终不大，甚至在libc测试中，使用reL4内核作为基础，拆开所有组件的测试结果比拼合所有组件性能略好的情况。当然在iozone测试中也有拉开一定差距的情况。总体而言，这两种通信方式之间差距相对没有那么巨大，但也跟不同测例集合需要的组件间通信数量有一定关系，在如iozone这样的组件间通信数量较多的测例集中，自然会出现函数调用通信转为IPC通信带来的开销更大的现象。总体而言，根据以上测试结果，可以认为我们设计的函数调用和IPC调用透明转换的机制具有较好的性能优势。

第三，对于reL4和seL4之间的实现性能对比。这是兼容同一个二进制接口的内核的两个不同实现，但是也很明显，reL4实现的性能略落后于seL4实现的性能，总体综合性能差距在10%左右。由于这些测例本身大量调用了seL4/reL4的各类系统调用而非个别系统调用路径，无法简单的认为是其中的某个系统调用路径性能过差而产生这样大的差距。该差距更有可能归因于Rust和C的差距，更确切的说，可能依然来自于重复检查或其他的为了在Rust下能够安全使用一些变量所引入的编译器额外检查开销。这并不意味着Rust的安全机制是没用的，只是在改写一个已经经过完整形式化验证的内核的时候，部分Rust检查可能是冗余的。除此之外，为了能够和seL4进行合并开发，我们实现了以Rust静态库和C代码链接的形式，引入了一些#[no_mangle]的Rust函数，这也会导致为了兼容性，而损失一定的编译优化性能。

最后，必须看到若干不正常的样例。很明显的，在run-static和lmbench两个测例中，Linux所花费的时间都更多，这并不是意味着Linux的性能更差，而是由于其中个别测例的问题。例如，在前面分析功能实现的时候，run-static测例集合中的socket测例集合并未实现，因此会存在fake实现和报错的问题，而Linux对网络socket的超时报错有着自己的机制，并非直接简单的fake实现，因此可能会导致测试完成的时间有所不同。这些差异实际来自于syscall实现的语义兼容性和reL4-Linux-kit只完成了部分Linux syscall导致的，而并非来自内核的实现性能。

## 5. 结论

本章对 reL4-Linux-kit 原型系统进行了全面的评估，主要包括功能实现评估和性能评估两个维度。

第一，在功能实现评估方面，通过系统性的测试验证了 reL4 内核与 seL4 内核的二进制接口兼容性，证实了基于组件化重构的 reL4 内核能够完整保持与 seL4 的二进制兼容特性。同时，详细评估了reL4-Linux-kit对Linux系统调用的兼容性支持范围，并测试了多组Linux应用程序在该环境下的运行情况和系统调用情况。

功能实现评估的实验结果表明，基于组件化架构重构的reL4微内核及其配套的 reL4-Linux-kit 用户态兼容层，作为一个完整的研究原型系统，成功实现了既定的功能性目标。这为构建全栈内存安全的操作系统架构提供了一个可行且具有实践价值的技术方案。

第二，在性能评估方面，我们对比分析了同一测试套件在原生 Linux环境、基于seL4内核的reL4-Linux-kit环境以及基于 reL4 内核的 reL4-Linux-kit 环境下的性能表现。测试还包含了系统组件合并与解耦两种模式下的性能差异比较。对于观察到的性能差异，我们从微内核架构特性、IPC 开销、Rust开发语言等角度进行了初步分析。

性能评估结果显示，尽管 reL4 和 reL4-Linux-kit 的实现与原生 Linux 性能存在预期内的差距，但作为一个研究原型系统，其性能表现仍在可接受范围内，证明了该架构的实用性和进一步发展潜力。这些性能数据也为后续优化工作提供了重要的参考依据。

## 附录

### 附录A：reL4不同feature总计需要的有效测例分类及数量

| 测例分类          | 测例内容简要描述                                             | 测例个数 | 测例通过个数 |
| :---------------- | :----------------------------------------------------------- | :------- | :----------- |
| Sys Call          | 测试seL4的基本syscall的执行                                  | 16       | 16           |
| Serserv Parent    | 测试服务器（Server）服务中的父进程功能                       | 10       | 10           |
| Timer             | 测试内核的定时器（Timer）中断功能                            | 2        | 2            |
| Bind              | 测试将通知（Notification）对象绑定到线程控制块（TCB）的操作  | 6        | 6            |
| Cnode OP          | 测试 CNode（Capability Node）的操作                          | 9        | 9            |
| Cspace            | 测试能力空间（Capability Space）的整体管理                   | 1        | 1            |
| Domains           | 测试 seL4 的域（Domain）调度机制                             | 5        | 5            |
| Cancel Badge Send | 测试在 IPC 发送过程中，带有标识（Badge）的通知在取消发送时的行为 | 2        | 2            |
| Page Fault        | 测试处理缺页异常（Page Fault）的机制                         | 10       | 9            |
| Time Out          | 测试MCS的TimeOut机制下的行为                                 | 3        | 3            |
| FPU               | 测试浮点运算单元（FPU）的状态在线程切换时的正确保存与恢复    | 3        | 3            |
| Frame Exports     | 测试帧（Frame）能力是否能够正确地跨地址空间映射              | 1        | 1            |
| Frame XN          | 测试将帧（Frame）映射为不可执行（eXecute Never）页的功能     | 2        | 2            |
| Framed IPC        | 测试使用专门的 IPC 帧（Frame）进行通信的机制                 | 3        | 3            |
| Retype            | 测试将未初始化的内存对象重定型（Retype）为特定内核对象的过程 | 3        | 3            |
| Interrupt         | 测试内核的中断处理架构                                       | 5        | 5            |
| IPCRIGHTS         | 测试在 IPC 消息中传递能力（Rights）的机制                    | 5        | 5            |
| IPC               | 测试基本的进程间通信（IPC）机制                              | 27       | 26           |
| Multi Core        | 测试内核在多核（SMP）环境下的正确性                          | 5        | 5            |
| NB Wait           | 测试非阻塞（Non-Blocking）的等待操作                         | 1        | 1            |
| PT                | 测试页表（Page Table）的操作                                 | 2        | 1            |
| Preempt Revoke    | 测试在撤销（Revoke）大量能力时，内核的抢占（Preemption）行为 | 1        | 1            |
| Regression        |                                                              | 3        | 3            |
| Sched Context     | 测试调度上下文（Sched Context）对象的分配、配置和控制        | 13       | 13           |
| Sched             | 测试内核的调度器（Scheduler）算法和行为                      | 21       | 21           |
| Serserv Client    | 测试客户端与服务器（Server）之间的交互逻辑                   | 5        | 5            |
| Serserv Cli Proc  | 测试作为客户端角色的进程与服务器之间的交互                   | 5        | 5            |
| SMC               | 测试安全监控调用（Secure Monitor Call）的支持                | 8        | 8            |
| SYNC              | 测试内核中通知机制实现的同步原语                             | 4        | 4            |
| Threads           | 测试线程（Thread）的创建、配置、启动、挂起和删除等生命周期管理 | 2        | 2            |
| tls               | 测试线程本地存储（Thread-Local Storage）功能                 | 3        | 3            |
| trivial           | 用于验证系统最基本的功能是否启动并运行                       | 3        | 3            |
| vspace            | 测试虚拟地址空间（VSpace）的管理                             | 8        | 7            |
| **总计**          |                                                              | **197**  | **193**      |

### 附录B：reL4-Linux-kit的Linux syscall实现情况

| syscall分类 | Syscall名称     | 简要含义                                   | 是否为fake实现 |
| :---------- | :-------------- | :----------------------------------------- | :------------- |
| 进程管理    | clone           | 创建新进程                                 | 否             |
| 进程管理    | execve          | 执行程序                                   | 否             |
| 进程管理    | exit            | 终止当前进程                               | 否             |
| 进程管理    | getpid          | 获取进程ID                                 | 否             |
| 进程管理    | getppid         | 获取父进程ID                               | 否             |
| 进程管理    | gettid          | 获取线程ID                                 | 否             |
| 进程管理    | kill            | 向进程发送信号                             | 否             |
| 进程管理    | tkill           | 向线程发送信号                             | 否             |
| 进程管理    | sched-yield     | 让出处理器                                 | 否             |
| 进程管理    | set-tid-address | 设置线程ID地址                             | 否             |
| 进程管理    | wait4           | 等待进程终止并获取资源使用情况             | 否             |
| 进程管理    | futex           | 快速用户空间互斥锁                         | 否             |
| 进程管理    | shmget          | 获取共享内存段                             | 否             |
| 进程管理    | shmat           | 附加共享内存段                             | 否             |
| 进程管理    | shmctl          | 控制共享内存段                             | 否             |
| 进程管理    | get-robust-list | 获取稳健互斥锁列表                         | 是             |
| 内存管理    | brk             | 改变数据段大小                             | 否             |
| 内存管理    | mmap            | 映射文件或设备到内存                       | 否             |
| 内存管理    | munmap          | 解除内存映射                               | 否             |
| 内存管理    | mprotect        | 设置内存保护权限                           | 是             |
| 权限管理    | getuid          | 获取用户ID                                 | 是             |
| 权限管理    | getgid          | 获取组ID                                   | 是             |
| 权限管理    | geteuid         | 获取有效用户ID                             | 是             |
| 权限管理    | getegid         | 获取有效组ID                               | 是             |
| 设备控制    | ioctl           | 设备控制操作                               | 否             |
| 时间管理    | clock_gettime   | 获取时钟时间                               | 否             |
| 时间管理    | gettimeofday    | 获取当前时间                               | 否             |
| 时间管理    | nanosleep       | 高精度睡眠                                 | 否             |
| 时间管理    | setitimer       | 设置间隔定时器                             | 否             |
| 事件管理    | ppoll           | 等待文件描述符上的事件（带超时）           | 否             |
| 事件管理    | pselect6        | 等待文件描述符上的事件（带超时和信号掩码） | 否             |
| 文件系统    | chdir           | 改变当前工作目录                           | 否             |
| 文件系统    | close           | 关闭文件描述符                             | 否             |
| 文件系统    | dup             | 复制文件描述符                             | 否             |
| 文件系统    | dup3            | 复制文件描述符并指定新描述符的flags        | 否             |
| 文件系统    | faccessat       | 检查文件访问权限（at版本）                 | 否             |
| 文件系统    | fcntl           | 操作文件描述符                             | 否             |
| 文件系统    | fstat           | 获取文件状态                               | 否             |
| 文件系统    | fstatat         | 获取文件状态（at版本）                     | 否             |
| 文件系统    | ftruncate       | 截断文件到指定长度                         | 否             |
| 文件系统    | statfs          | 获取文件系统统计信息                       | 否             |
| 文件系统    | getcwd          | 获取当前工作目录                           | 否             |
| 文件系统    | getdents64      | 获取目录条目                               | 否             |
| 文件系统    | lseek           | 移动文件读写指针                           | 否             |
| 文件系统    | mkdirat         | 创建目录（at版本）                         | 否             |
| 文件系统    | mount           | 挂载文件系统                               | 否             |
| 文件系统    | openat          | 打开文件（at版本）                         | 否             |
| 文件系统    | pipe2           | 创建管道（带有flags）                      | 否             |
| 文件系统    | read            | 从文件读取数据                             | 否             |
| 文件系统    | readv           | 从文件读取数据到多个缓冲区                 | 否             |
| 文件系统    | pread64         | 从文件指定偏移处读取数据                   | 否             |
| 文件系统    | write           | 向文件写入数据                             | 否             |
| 文件系统    | writev          | 从多个缓冲区向文件写入数据                 | 否             |
| 文件系统    | pwrite64        | 向文件指定偏移处写入数据                   | 否             |
| 文件系统    | renameat        | 重命名文件（at版本，这里调用renameat2）    | 否             |
| 文件系统    | sendfile        | 在文件描述符之间传输数据                   | 否             |
| 文件系统    | umount2         | 卸载文件系统                               | 否             |
| 文件系统    | unlinkat        | 删除文件（at版本）                         | 否             |
| 文件系统    | utimensat       | 设置文件时间戳（at版本）                   | 否             |
| 文件系统    | msync           | 同步内存映射文件到磁盘                     | 是             |
| 文件系统    | sync            | 同步所有缓存到磁盘                         | 是             |
| 文件系统    | fsync           | 同步文件缓存到磁盘                         | 是             |
| 系统信息    | getrusage       | 获取资源使用情况                           | 否             |
| 系统信息    | uname           | 获取系统信息                               | 否             |
| 信号处理    | rtsigaction     | 设置信号处理函数                           | 否             |
| 信号处理    | rtsigprocmask   | 设置信号掩码                               | 否             |
| 信号处理    | rtsigreturn     | 从信号处理返回                             | 否             |
| 信号处理    | rtsigtimedwait  | 等待信号（带超时）                         | 否             |
| 资源管理    | prlimit64       | 设置或获取资源限制                         | 否             |

### 附录C：Linux测例syscall调用统计表(aarch64)

| Syscall Type | basic                                                        | run-static                                                   | busybox-testcase                                             | lua                                                          | iozone                                                       | libc                                                         | lmbench                                                      |
| :----------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 文件管理     | getcwd, dup, dup3, mkdirat, unlinkat, umount2, mount, chdir, openat, close, getdents64, read, write, fstat | getcwd, dup, dup3, fcntl, ioctl, unlinkat, statfs, chdir, openat, close, lseek, read, write, readv, writev, pread64, fstatat, fstat, utimensat | getcwd, dup3, fcntl, ioctl, mkdirat, unlinkat, openat, close, getdents64, lseek, read, write, readv, writev, fstatat, utimensat | getcwd, fcntl, ioctl, unlinkat, renameat, faccessat, openat, close, read, write, readv, sendfile, ppoll | getcwd, fcntl, ioctl, unlinkat, openat, close, lseek, read, write, readv, writev, pread64, pwrite64, pselect6, fstatat, sync, fsync | unlinkat, openat, close, lseek, read, writev, ftruncate      | getcwd, dup, fcntl, ioctl, unlinkat, openat, close, pipe2, lseek, read, write, writev, pselect6, fstatat, fstat, fsync, utimensat, umask |
| 进程管理     | exit, getpid, getppid, gettid, clone, execve, wait4, sched-yield | exit, exit-group, set-tid-address, getpid, getppid, getuid, geteuid, getegid, gettid, clone, execve, wait4, prlimit64, futex, set-robust-list, get-robust-list | exit, exit-group, set-tid-address, getpid, getppid, getuid, gettid, clone, execve, wait4 | exit, exit-group, set-tid-address, getpid, getppid, getuid, gettid, clone, execve, wait4 | exit, exit-group, set-tid-address, getpid, getppid, getuid, gettid, clone, execve, wait4 | exit, exit-group, set-tid-address, gettid, clone, execve, wait4 | exit, exit-group, set-tid-address, getrusage, getpid, getppid, getuid, gettid, clone, execve, wait4, prlimit64 |
| 内存管理     | brk, munmap, mmap                                            | brk, munmap, mmap, mprotect                                  | brk, munmap, mmap                                            | brk, munmap, mmap                                            | brk, munmap, mmap, madvise                                   | brk, munmap, mmap, madvise                                   | brk, munmap, mmap, mprotect, msync                           |
| 信号处理     | rtsigaction, rtsigprocmask                                   | tkill, rtsigaction, rtsigprocmask, rtsigtimedwait, rtsigreturn | rtsigaction, rtsigprocmask, rtsigreturn                      | rtsigaction, rtsigprocmask                                   | rtsigaction, rtsigprocmask                                   | rtsigaction, rtsigprocmask                                   | kill, rtsigaction, rtsigprocmask, rtsigreturn                |
| 网络管理     |                                                              | socket, bind, listen, accept, connect, getsockname, sendto, recvfrom, setsockopt |                                                              |                                                              |                                                              |                                                              |                                                              |
| 时间管理     | nanosleep, gettimeofday                                      | nanosleep                                                    | nanosleep                                                    |                                                              | nanosleep                                                    |                                                              | setitimer                                                    |
| 系统信息     | uname                                                        | uname                                                        | uname                                                        | uname, sysinfo                                               | uname                                                        |                                                              | uname                                                        |
| 进程间通信   | pipe2                                                        | pipe2                                                        | pipe2                                                        |                                                              |                                                              | shmget, shmctl, shmat                                        | pipe2                                                        |

------