---
title: "rel4升级MCS记录"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---
# rel4升级MCS记录

本文只做简单的大纲性质的代码升级描述，尽量涵盖升级mcs的方方面面，但是不对细节做展开，细节问题请自行查看rel4代码的逻辑

如果只是了解升级相关的代码，直接看本文即可

如果想要了解算法相关内容，请先看论文[Scheduling context capabilities](https://cloud.tsinghua.edu.cn/f/0a44a2e5e309431fbbc7/)

## 开启mcs特性及pbf的更新

首要问题是在开启了mcs特性之后能让他过编译。

1：如何让整个内核开启MCS支持

在整个内核中，有两个cmake文件

./project/sel4test/easy-settings.cmake

./easy-settings.cmake

在这两个cmake文件中，都需要开启mcs，但是实际查看mi-dev-repo的脚本你会发现是将sel4test的这个easy-setting脚本拷贝过去了。

但是开启之后，会存在一些问题

例如，在原有的代码中，我们注释掉了一些东西，而这些注释掉的东西和现在开启了mcs的代码并不兼容。

因此会编不过。

所以首先，我们需要做的是，修改kernel的代码让C版本的代码开启MCS的支持的那部分代码。



2、rel4添加mcs这个feature

这也需要对修改整个rel4的cargo的一些东西。

简单来说，mi-dev-repo里面的sel4-test对rel4的构建是从build.py开始的，因此我选择直接更改rel4的build.py。



3、rel4的pbf文件的更新

这里面变化的主要是增加sc的部分

我更新了之后的文件放在rel4 common下的pbf文件夹下的每个架构都新开了一个mcs文件夹。

但是，我觉得这样搞不是个事儿，还是得把那个解析bf文档的代码给弄出来塞进去，自动解析bf文件到pbf文件，否则的话，现在已经有每个架构下两个文件夹了，但是如果再加个smp，是不是每个架构下需要四个文件夹（smp+mcs，两个是否开启，总共四个嘛）

总感觉不是很方便快捷。

## 时钟模块

要做MCS，首先需要做的事情就在于时钟模块，因为原先只需要一个时钟中断就行了，现在需要精确的计时。

由于这些是公用的部分，所以我写在了sel4common里面。

这里面我又分为两个部分，一个是新开了一个文件夹platform，另一个写入了arch下的分别各自的架构

之所以这样设计，是因为，以arm为例，在arm下有很多不同的平台，原生的sel4不仅支持qemu，还支持了一大堆不同的平台，他们可能存在共有的特性，所以在arch aarch64下都是可以使用的，但是也有一些平台特有的时钟的特性，比如如何初始化时钟需要往特定的寄存器按照特定的序列读写，这些是跟platform相关的。

因此在这里的约定是，在arch下某个架构的timer.rs，用于描述在那个架构下特定的部分，而在platform下的代码则用于描述不同平台下的特定的板级的支持

当然，还有一些公用的宏定义，比如一个毫秒中有多少个微秒（肯定是1000），这种不管哪里都通用的宏定义，就放在platform下的time_def.rs中（虽然从逻辑上他是更高一级的内容就是了。）

而在platform的mod.rs中，设计了一个trait用于描述时钟的行为

```
pub trait Timer_func {
    fn initTimer(self);
    fn getCurrentTime(self) -> ticks_t;
    fn setDeadline(self, deadline: ticks_t);
    fn resetTimer(self);
    fn ackDeadlineIRQ(self);
}
```

（虽然rv下有比如sbi的时钟接口，不需要做什么ack deadline irq这种事情）



在每个arch下的timer.rs中则是该架构下的一些额外的辅助函数（实际上很多也可以当成宏定义写就是了）

## sched_context（以下简称sc，都代指这个）

对于这个部分，应该算是mcs的核心数据结构和操作了。

具体的算法逻辑，最推荐的是看[defects of the POSIX Sporadic Server and How to Correct Them](https://cloud.tsinghua.edu.cn/f/2a5f0fbf45a74426afc8/)

我在这里不介绍实际的算法

具体整个类的代码都放在sel4_task/src/sched_contex.rs

这里主要包括三个部分

一个是数据结构的定义

一个是refill系列（refill就是这个ss算法核心的部分）以及，和对这个数据结构的基础操作

还有就是schedContext开头的系列函数（这些基本上是handlesyscall解析的时候调用的函数）

数据结构本身其实没啥太多好说的（看过论文大概都能懂个大概，就是设定重填的时间，以及重填时间的一些整理）。但是对于refill的数据结构需要注意一点他在代码中的实际排布

```
pub fn refill_index(&self, index: usize) -> *mut refill_t {
    //&mut refill_t {
    convert_to_mut_type_ref::<refill_t>(
        (self.get_ptr() + size_of::<sched_context_t>()) + index * size_of::<refill_t>(),
    ) as *mut refill_t
}
```

也许这长得比较抽象，但是大概意思就是，在sched_context_t后面，紧跟着一堆refill_t的array，这是他实际的内存布局的情况

从而引出了后面refill系列的一系列的操作，这些都是跟这个内存分配相关的



schedContext开头的那一系列函数其实是在decode_sched_context_invocation的解析中，根据传入的label的不同，分别处理的，然后再经过一系列的处理之后，调用schedcontext的这些函数

总共操作类型包括以下几个

- SchedContextConsumed 更新时间consumed的数量
- SchedContextBind 根据实际情况，绑定thread或者notification
- SchedContextUnbindObject 同上的相反操作
- SchedContextYieldTo 切换sc



## sched_control

关于sched_context的control这个操作。其实也是基于sc的。之所以有这个东西，是因为跟原先的sel4的设计相关

sel4首先是个静态优先级的一个系统。

而且，内核不应该保留策略，也就是说，对于上述的sc的一些设定（比如设定的优先级是多少，设定的Period的大小是多少，budget是多少，他不应该在内核中给定策略，而应该是在root server来决定的。

因此就有了这个sched_control这个capability，表示用于操作sc的一个能力

他只有decode_sched_control_invocation这一个接口，里面也没怎么细分label，就一个SchedControlConfigureFlags

```
let budget_us: time_t = get_syscall_arg(0, buffer);
let budget_ticks = usToTicks(budget_us);
let period_us = get_syscall_arg(TIME_ARG_SIZE, buffer);
let period_ticks = usToTicks(period_us);
let extra_refills = get_syscall_arg(TIME_ARG_SIZE * 2, buffer);
let badge = get_syscall_arg(TIME_ARG_SIZE * 2 + 1, buffer);
```

然后它从syscall这里的参数获取了试图设定新的sc的这些参数

然后据此经过一些检查后，调用invokeSchedControl_ConfigureFlags

进行目标capability的一个配置。

## reply

reply这里主要是增加了一堆接口，具体实现相关存放在sel4_task/src/reply.rs

其实主要函数建议你看sel4那个include/object/reply.h

里面主要涉及到五个函数

- unlink
- push
- pop
- remove
- remove_tcb

关于这个reply，我个人看法是得看他的数据结构

他集成了一个call stack。不过话虽如此，我觉得这个call stack并不是个真的stack，而是一个使用指针实现的stack

所以这个push pop就很好理解了。

至于他到底push，pop的内容是什么

我认为是一个sc

## timeout以及相关寄存器规则的更改

关于timeout的相关处理可以看论文去，反正有啥问题往这儿发，当然也可以不做处理

但是首先是cspace里面，增加了一个slot，名字叫做tcbTimeoutHandler

不过这里涉及到关键性的数据结构的更改

主要是message的那些问题

首先是message的寄存器的扩充

```
#[cfg(not(feature = "KERNEL_MCS"))]
pub const fault_messages: [[usize; MAX_MSG_SIZE]; 2] = [
    [0, 1, 2, 3, 4, 5, 6, 7, 34, 31, 32, 33],
    [34, 31, 33, 0, 0, 0, 0, 0, 0, 0, 0, 0],
];
#[cfg(feature = "KERNEL_MCS")]
pub const fault_messages: [[usize; MAX_MSG_SIZE]; 3] = [
    [
        0, 1, 2, 3, 4, 5, 6, 7, 34, 31, 32, 33, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
        0, 0, 0, 0, 0,
    ],
    [
        34, 31, 33, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
        0, 0, 0, 0, 0,
    ],
    [
        34, 31, 33, 0, 1, 2, 3, 4, 5, 6, 7, 8, 16, 17, 18, 29, 30, 9, 10, 11, 12, 13, 14, 15, 19,
        20, 21, 22, 23, 24, 25, 26, 27, 28,
    ],
];
```

上面的代码这里，其实相当于矩阵中增加了一行，而实质性的意义在于给timeouthandler发消息的时候，需要往寄存器里面复制消息的内容变了

在do_fault_reply_transfer函数中，在复制的时候增加了这么一个case

```
#[cfg(feature = "KERNEL_MCS")]
seL4_Fault_tag::seL4_Fault_Timeout => {
    self.copy_fault_mrs_for_reply(
        receiver,
        MessageID_TimeoutReply,
        core::cmp::min(length, n_timeoutMessage),
    );
    return label as usize == 0;
}
```

这里MessageID_TimeoutReply等于2就是指代上面的那个矩阵里的第二行（从0开始数嘛）

当然在do_fault_tranfer里面也需要增加相应的处理，在此不再更多的列举代码

除此之外，内核还有相应的一系列的函数需要增加，比如handleTimeout，tcb中的validTimeoutHandler等相应的函数，不过handleTimeout这些基本依托于原有的消息通知相关的系统函数，并没有构成新的代码



## 消息通知

mcs特性的增加也对send和receive ipc等等相关代码，以及endpoint和notification的数据结构和算法操作产生了一些影响。

- endpoint

endpoint的部分，对send ipc和receive ipc、cancel ipc的代码进行了相应的修改（当然部分也会被timeout所使用）

函数接口上，最大的差别在于canDonate等参数的引入，这个参数的作用是说，是否可以发生sc捐赠，简单来说，如果是类似于迁移线程的被动服务器，那么需要捐赠sc，也就是把原先进程的sc代表时间片的剩余部分赠送给目标的被动服务器，并让目标服务器拥有一些sc代表的时间片

当然相应的，会对reply和sc的一些字段进行处理。这些处理我认为不再赘述





## 对于tcb、endpoint等结构体的函数的添加和修改

- 对于tcb模块的影响

tcb这里增加了两个字段

```
#[cfg(feature = "KERNEL_MCS")]
/// scheduling context that this TCB is running on
pub tcbSchedContext: usize,
#[cfg(feature = "KERNEL_MCS")]
/// scheduling context that this TCB yielded to
pub tcbYieldTo: usize,
```



- 对于is_schedulable的修复

原先在没有mcs的情况下，对这个schedulable的判定就是is_runnable。

但是这个必须专门提一嘴，因为在论文中特地的提及了这个问题。

就是对于调度器中选择下一个进程的条件，原先是只有优先级，选择优先级最高并且可以运行的进程，但是加上sc之后，有了新的条件，他不仅要可以运行，并且他的当前的时间片没有被用完。也就是满足了临界性的要求。

当然落实到代码中去，在原先并没有考虑mcs的特性，所以rel4在编写的时候，很多事实上是schedulable的，被等价的写成了runnable。这部分需要修复

- 优先级继承情况下的优先级调整

在原先的代码中也有set_priority的代码。不过看MCS的论文的介绍，他有一个优先级继承的算法。这里也需要加以描述

首先先不说别的，由于sel4本身是一个没有主动配置优先级的策略的，所以实际上，他也是一个属于tcb这个capability下的一个系统调用。

第二个地方在于，如果这是一个当前正在执行的代码的话，更改优先级，其实意味着允许抢占

原先的策略非常简单

```
pub fn set_priority(&mut self, priority: prio_t) {
    self.sched_dequeue();
    self.tcbPriority = priority;
    if self.is_runnable() {
        if self.is_current() {
            rescheduleRequired();
        } else {
            possible_switch_to(self)
        }
    }
}
```

首先是基本的优先级设定，第二个是判断是否可执行，如果可以，就要求重新调度

但是更改之后，就复杂多了

```
        match self.get_state() {
            ThreadState::ThreadStateRunning | ThreadState::ThreadStateRestart => {
            ...
            }
            ThreadState::ThreadStateBlockedOnReceive | ThreadState::ThreadStateBlockedOnSend => {
            	...
            	reorder_EP
            }
            ThreadState::ThreadStateBlockedOnNotification => {
            	...
            	reorder_NTFN
            }
```

他对不同情况下的进程状态做了处理。

至于给EP和NTFN的乱序，是因为，他在这里认为改了优先级之后，排在某个endpoint和notification上的tcb中，优先给他通信的那个tcb的顺序需要相应的发生变化。



- 调度算法相关

在调度算法的框架下，他主要修改了

1. sched_enqueue
2. sched_append

（其实就是assert加了几个，没啥大用，但是这个sched并不是sc的sched）

除此之外，增加的几个函数

1. Release_Remove
2. Release_Enqueue
3. tcb_Release_Dequeue

（应该是和原先的sched的enqueue，dequeue，append相对应的）

我个人的看法就是简单的把它从调度队列的链表上摘下来，并通过ksReleaseHead（后面会说ks系列的）记录下来。

或者是把ksReleaseHead的那个挂上去。

但是这和之前的调度算法有些差别，比如enqueue，是需要遍历所有的tcb结构，查看其sc的情况（基本就是个根据sc的rtime来进行排序的操作了）

确切的说，这可以认为，是在上述tcb的基础上，加了一套调度框架。



## 调度器相关和重填的时间

我们说过，MCS中的时间这件事情是重要的，所以第一个需要关注的是updateTimestamp这个更新时间记录的函数

这里会影响，当前的进程，何时记录时间，然后再决定

他出现的地方在于，handleInterruptEntry，preemptionPoint，还有MCS_DO_IF_BUDGET宏

这里同样需要关注在c版本代码中的MCS_DO_IF_BUDGET

```
#ifdef CONFIG_KERNEL_MCS
#define MCS_DO_IF_BUDGET(_block) \
    updateTimestamp(); \
    if (likely(checkBudgetRestart())) { \
        _block \
    }
#else
#define MCS_DO_IF_BUDGET(_block) \
    { \
        _block \
    }
#endif
```

这个宏

这个宏出现的位置确实没有几个地方，但是都是关键性的地方

他在handlesyscall，handleVMFaultEvent，handleUserLevelFault，handleUnknownSyscall

这几个地方出现

虽然handleInterruptEntry中，没有调用MCS_DO_IF_BUDGET这个宏，但是里面代码是这样的

```
#ifdef CONFIG_KERNEL_MCS
    if (SMP_TERNARY(clh_is_self_in_queue(), 1))
    {
        updateTimestamp();
        checkBudget();
    }
#endif
```

和MCS_DO_IF_BUDGET里面的定义不能说一样吧，只能说大同小异

也就是说，在进入到这些地方之后，都会做一个更新时间戳，并且checkbudget的操作

在checkbudget的操作中，他对当前的sc做了一堆的相关检查来更新当前的thread和sc的状态。（具体不细说了）



第二个是调度器的更改

调度器的更改其实涉及到两个函数，schedule和activateThread（好在一般调用的话，这俩都是连着调用的）

这部分代码按我的理解就是在原有的基础上加了一个sc的补丁，判定是否需要调度的规则。

除此之外，还有关键性的选择下一个调度对象的choosethread函数

这里增加了assert来保证选择的tcb是不会存在sc的时间片用完的一堆问题的。



第三个其实是fastpath部分的代码

众所周知哈，在内核进入syscall之前，会先经过fastpath的判定，如果是fastpath，就直接转发，否则经过slowpath，再根据syscall id来判定是unknown syscall，还是正常的syscall的处理流程。



在前面描述更新时间戳，以及budget检查的时候，并没有在fastpath这部分的代码中的实现，也就是说，他并不在fastpath这里做检查（注意这个检查时间戳的节点）

但是，在fastpath这里，仍然需要更新sc这个的reply，replytcb以及其他相关sc相关的结构体。

我认为他这里是fastpath，并且ss算法是能够接受一定的时间超额的，所以我认为在这里，不做检查，应该也许是合理的？



第四个是rescheduleRequired、suspend、restart相关的变更

这部分其实基本上直接写两套代码算了

其实主要做的事情就是，对于判定是否需要重新调度的判定，这里新增了sc的相关判定

## 内核初始化代码的更改

这里涉及到几个地方，首先是ks系列的全局变量有更新了，具体来说更新了这么几个

- ksCurSC，当前的sc
- ksConsumed，sc消耗掉的时间
- ksReprogram，用于调度中指示是否需要重新调度的
- ksReleaseHead
- ksCurTime，用于记录时间的

所以相应的，init_core_state的时候需要初始化上述的几个状态

除此之外主要更新还是在于几个函数

- try_init_kernel

- create_initial_thread

- create_idle_thread

这几个函数都主要在于添加初始化的sc以及sc_control的内容

## 剩余的问题：

1、有些跟mcs相关的测例看上去没有测试，好像是因为CONFIG_HAVE_TIMER这个没开导致直接没测试。之前请教过lxy关于如何开启这个config

2、一些明显的todo没做，虽然能过目前的测试，但是后续不保证全测试路径覆盖，需要检查所有的mcs feature的

3、关于unsafe的封装工作。

4、关于convert系列的清理工作。

5、invokeTCB_ThreadControlCaps系列函数的拆解工作（稍微讲一下，他这个如何操作是通过一个flags，然后不同的flags有不同操作的组合，彼此之间关联不大，完全可以拆分之后重组。现在的写法感觉不爽）

