---
title: "reL4项目总结"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# reL4项目总结

## 1 seL4内核概述
### 1.1 基于capability的内核对象抽象

在一个操作系统内核中,几乎所有的操作都是在不断地跟各类内核对象进行交互。但是直接使用内核对象的地址访问该内核对象存在安全问题,因此出现了使用句柄(handler)等方式访问内核对象的设计,seL4也同样采用了这样的设计思路。不同的是,seL4作为一个微内核,可以通过精巧的设计,将内核对象数量限定为有限的若干种,并通过抽象,将内核对象相关的重要信息封装在一个称为capability的,固定为128bits大小的句柄中。

seL4通过capability机制对内核对象进行统一的抽象,并通过一个集合了所有的capability的cspace对内核中的所有内核对象进行管理。

对于cspace的结构,可以类比于页表。在cspace的顶层,包含了2^redix个slot构成的array用于存放capability,每个slot大小128bits,和capability大小一样。在顶层cspace中的某些slot,也可以存放一些cnode,即用于表示二级或者更深级数的cspace的capability。

cspace通过一个cptr来指定某个slot,并找到其中存放的capability。这个cptr类似于页表中的虚拟地址,用于描述slot的地址。但是正如前面所述,这不是一个真实的内核对象的地址。这个cptr会通过guard bits和redix进行层级切分。其中guard bits是用于判断是否跟当前的cspace相符的一个判定,而redix正如前文所述,用于指明该层的slot array中包含多少个slots。

例如一个32位的cptr,如果第一层的guard占用了4个bits,redix占用了8个bits。那么最高的12个bits会被用于访问最顶层的slot array,该array具有2^8=256个slots。如果这个cptr最高的4位为0,剩余的低20位都是0,在最高4位后面跟着的8位所指出的那个slot处存放着查找到的目标slot,如果该slot不是一个cnode的capability,那么就认为找到了。如果并非如此,即可以向下查找,那么就查看第二层的guard bits和redix。如果同样是4个bits的guard和8个bits的redix,会继续匹配剩余20位中最高4位表示的guard是否都为0,再之后的8位用于查找slot,最后的20-12=8位如果为0,且该slot不为cnode,就认为找到了,如果不是就继续。以此类推。

除了通过cspace来进行capability的管理之外,seL4也维护了capability之间的关系,除了创建,销毁之外,还有Mint(从某个capability继承全部或者部分权限),Copy,Move,Revoke(不仅删除自己,还删除所有派生出去的capapbility)等更为复杂的capability操作。

capability系统让seL4对所有内核对象有了非常灵活的管理。

### 1.2 seL4的内存管理

#### 1.2.1 虚拟内存的管理

在支持虚拟内存的硬件架构上，操作系统可以开启分页或者分段或者段页式的虚拟内存机制，从而实现远超物理内存限制的地址空间，以及通过地址空间来隔离不同进程，提升安全性。seL4需要为用户态提供相应的机制。

seL4所基于的硬件架构，均支持多级的分页内存机制，因此，seL4提供了相应的统一多级页表的支持。

在名称上，seL4系统中的物理内存页被称为frame，而虚拟内存所需要的页表项和页表，包括page Directory，page table等。这些都被描述为不同类型的capability。当然，由于seL4支持多个架构，因此他们的页表项pte的相应属性都有一些架构相关的内容。

除了页表之外，seL4还提供了多地址空间asid的支持。包括了asid pool和asid control两个capability。

page fault的处理，也是传统内核中必要的一部分内容。在seL4中，则会传递给进程设定的fault handler的endpoint进行处理。

#### 1.2.2 内核所需要的内存分配

对于seL4而言，在系统启动之后，除了预先分配给内核的内存空间，剩余物理内存空间都交给了root-task。微内核在内核中只保留机制，而让策略放在用户态的设计哲学，决定了需要用户态的root-task来确定内存分配的策略。而内核在运行过程中需要不断地创建新的内核对象，因此需要提供相应的机制以供用户态的策略可以申请内核对象。

在seL4中，将未被分配出去的内存空间，也被当成一种特殊的capability，也就是untyped capability。untyped capability用于表示一段未被分配出去的内存空间，并可以通过该capability唯一的系统调用来将其type为其他的内核对象。

每个untyped的内存在申请的时候并不是一口气全部分配出去的，而是按照申请的内存大小向上对齐的2的整数次幂进行分配。此时，剩余的未分配的untyped的内存仍然会被记录，以便下一次进行再分配。

### 1.3 seL4的IPC

### 1.4 seL4的系统启动

### 1.5 seL4的中断和异常处理

## 2 使用Rust语言重写seL4——reL4内核的实现
### reL4项目概述
reL4项目源于李龙昊等（2023）的尝试，最初的目标是使用rust重写seL4内核。该内核最初只是逐步将sel4的C语言部分的代码部分去除，并将去除的代码改写成一个rust语言编写的静态链接库，和seL4的原生代码进行链接，逐步进行替换的过程，最终将整个内核都变成rust语言实现的。在本文工作开展之前，已经实现了riscv的分支，通过了seL4原生的不添加任何特性基本测例。

### 组件化的支持aarch64的reL4内核
对于reL4内核的aarch64支持，本文采用组件化的思想，对reL4进行重构。

#### reL4的组件化改造
组件化内核，顾名思义，就是使用组件化的方式来构造内核。它将操作系统的核心——内核，视为一项复杂的软件工程并使用组件化的思想进行编写。在这一过程中，我们打破了传统内核开发的界限，让各个组件独立开来编写，使得程序员能够如同编写普通程序般自如地编写内核模块，极大地提升了开发的灵活性与效率。

更为重要的是，一个经过精心定义与验证的组件，如同构建软件大厦的预制砖块，能够在多个项目或系统中灵活复用，极大地避免了重复劳动与资源浪费，真正实现了“一次编写，到处运行”的愿景。这种高度的可重用性，不仅提升了开发效率，还确保了代码质量的一致性与稳定性。

​ 在后续的维护阶段，组件化的优势同样显著。通过将系统划分为清晰界定的组件，开发者能够更加精准地定位问题所在，从而实施更加高效、精准的修复与升级策略。此外，由于组件间的松耦合设计，对某一组件的单独升级或改造几乎不会影响到系统的其他部分，这极大地降低了维护的复杂性与成本，确保了系统的持续稳定运行。

总之，组件化开发方式以其独特的优势，在大型软件工程的构建中发挥着不可替代的作用。

在reL4项目支持aarch64架构的过程中，我们首先使用组件化的思想，将原有的仅支持riscv的reL4内核先拆分成为多个组件化的模块，具体分为六个模块，分别为
 - common：最底层的支持，提供各类reL4的基础数据结构，宏定义等内容。
 - cspcae：提供了L4系列微内核所特有的capability的管理机制，维护了包含所有capability的表格，即cspace，包含了capability的插入删除和各类管理方法。
 - vspace：提供了微内核的地址空间相关的操作。
 - task：提供了微内核的任务管理相关基本操作。
 - ipc：提供了微内核的进程间通信的相关操作。
 - kernel：kernel作为整个项目最顶层的上层模块，在上述几个模块的基础上，对传入给kernel的syscall进行了解析。

另外，随着内核被拆分成为多个组件，而每个组件都放置在多个不同的github仓库中，按照组件化的思想每个组件单独维护，从而在多个仓库之间的协同工作引发了组件间的同步问题。为此，我们也创新地设计了大小仓库协同工作的协作范式。

一方面，我们承认使用组件化的思想，将组件拆分为不同仓库。另一方面，在多仓集成的开发环境和代码快速迭代的开发条件下，使用单个仓库，具有较高的开发效率。为此，我们同时维护一个大仓库和一个小仓库，并在进行git push等操作时进行同步，从而实现大小仓库协同开发的方案，同时具有组件化的可重用性和单一仓库的快速迭代优势。

我们使用git-subrepo工具，替代简单的git和submodule所存在的大小仓库同步不便的问题。该工具可以单独使用类似于git subrepo push sub-module-dir的命令，提交某个子仓库，也可以使用传统的git方式，push整个大仓库。

同时，为了自动化地便捷开发，我们在该工具的基础上改写了github的ci脚本，在触发了push的ci操作的时候，能够自动触发git subrepo的push操作，同步到多个子仓库中，并单独触发多个子仓库各自的ci，从而实现大小仓库的协同工作。

#### reL4的aarch64支持
在经过组件化改造之后的reL4的基础上，我们也实现了reL4的aarch64的支持。这涉及到多个不同的模块，每个模块下都有相应的arch文件夹和其他架构无关的代码，以提供不同的，具体来说如下：

common模块：第一，该模块需要提供各类基础数据结构的定义，其中包括ArchTCB中的定义，用于在上下文切换时候的寄存器的保存，由于aarch64和riscv64下的寄存器存在差异，因此需要为不同的架构提供不同的寄存器上下文保存区域。第二，包含了各类不同架构下的常量定义，第三，在进入内核的时候，会通过寄存器传递一些参数和信息，在不同架构下使用不同的寄存器进行参数的传递，同时，在不同架构下传递的信息内容也有不同，例如，向内核传递页表相关的操作请求，在不同架构下具有不同的含义。这些内容都需要分别维护

cspace模块：在cspace模块中，只有capability的管理，而在aarch64和riscv64两个架构下，具有不同的跟arch相关的capability，因此，在cspace模块下，需要为不同的arch实现公共的capability的操作处理。

vspace模块：在aarch64和riscv64架构下的页表项结构和虚拟内存管理具有非常明显的差异，因此，这个模块也有非常大的改动。第一，该模块提供了不同结构下的页表项的定义和相关页表操作的实现，也包括asid和刷新tlb等架构强相关内容。第二，在内核启动的时候，需要为内核映射相关区域的虚拟内存，这部分的处理也具有架构差异。

kernel模块：作为系统的顶层模块，必须处理架构相关内容。第一，在调用vspace的跟页表相关内容的基础上，同样需要增加其他的各类boot的代码。第二，在该模块主要负责对各类传入的syscall的解析，其中也包括跟架构相关的capability和message info的解析，这部分需要架构相关的处理。第三，aarch64使用Gic，具体来说是Gicv2，而riscv64使用plic作为外部中断控制器，在不启用其他外设的情况下，仍然需要时钟中断作为任务调度的基础，riscv64可以使用sbi直接提供中断，但是aarch64则必须启用Gic，这部分的处理具有差异，第四，对于从用户态进入内核的汇编代码，也有所不同，aarch64根据不同的el等级不同的sp和aarch64和aarch32的差别，以及中断原因是sync、irq、fiq或者SError等原因，构成四个大类共16个表项的中断向量表，而riscv则使用stvc寄存器存放中断向量并可选择向量模式还是单入口模式。而上下文保存的寄存器也各有不同，因此，这部分需要处理架构相关。

另外的ipc模块和task模块，由于组件化设计下良好的模块化抽象，没有单独的跟arch相关的代码，由此可见，组件化设计所具有的优势。

经过上述改造，reL4能够同时支持riscv64及aarch64两个不同的硬件架构。

### reL4添加各类其他特性支持
对reL4添加各类其他特性支持，首先需要改造整个reL4的构建脚本，以支持各类扩展特性的组合，同时需要解析包含各个特性的pbf文件，这部分内容放在后续章节进行描述。本章节只描述我们实现的具体的各个特性对内核的相应改动。
#### reL4的mcs特性支持
##### reL4中MCS(mixed-criticality system)策略概述
reL4中的MCS特性，主要基于Lyons（2017）的论文，


##### reL4中实现MCS特性概述
为了实现reL4对MCS特性的支持，需要做如下更改：

首先需要更改pbf文件，添加相应的sched_context（后文简称为sc）的capability的描述。

其次，MCS特性主要用于对进程实时性的控制，因此，必须添加相应的时钟模块以提供较为精准的定时等操作。在riscv64下，时钟模块主要使用sbi来进行设置，但是在aarch64下，情况较为复杂，在不同的开发板中，具有不同的时钟模块和相应的操作，并通过设备树文件传递给内核，在原有的seL4下也针对不同的时钟模块提供了相应的支持，但对于不同板级的描述则主要通过cmake文件和h文件中的定义，手动解析相关内容，并静态编译到内核中，而非自动解析设备树的方式来确定板级时钟支持情况。

由于目标平台仅为简单的qemu，无需过于强调板级兼容，reL4在这方面的处理为，抽象了一个platform子模块，并设计了一个trait来描述时钟的行为

```
pub trait Timer_func {
    fn initTimer(self);
    fn getCurrentTime(self) -> ticks_t;
    fn setDeadline(self, deadline: ticks_t);
    fn resetTimer(self);
    fn ackDeadlineIRQ(self);
}
```
将时钟的行为简单抽象为这样的对外接口。并在时钟中断的处理过程中，使用这些trait。

第三，需要增加基础数据结构sched_context的实现。sc是实现整个MCS特性的基础数据结构，它的主要实现方式为：

第四，增加reply的支持，修改相关的tcb，endpoint等结构体。

第五，增加timeout的消息传递支持

第六，修改调度器实现重填机制



#### reL4的smp特性支持


#### reL4的fpu特性支持

#### 其他额外特性支持
##### Arm的SMC call的支持

##### Arm的ptmr和pcnt的支持

### 支持reL4构建的辅助工具实现
##### reL4的xtask构建脚本支持

##### reL4的pbf文件格式解析支持

##### reL4的集中配置脚本支持

### 部分reL4实现中的bug及其修复


## 3 支持部分Linux syscall的reL4-Linux-kit的实现

### reL4-linux-kit概述

### root-task的设计和实现

### 其他内核模块的设计及其实现

### Linux用户态程序的二进制兼容方案

### 函数转IPC的方案

### 灵活可组合的函数调用和IPC调用组件设计

## 4 实验评估
### reL4内核及测例编译方法
reL4内核使用xtask进行构建配置的支持，同时，为了能够集成式地支持用户态的开发，我们也构建了reL4-cli工具，以进一步地简化用户态开发流程。在此将对目前各种情况下的测试编译方法进行说明。

#### reL4测试seL4原生测例
在reL4中进行seL4原生测例主要用于内核开发过程中通过各个测例，本项目维护了一个docker环境，来提供稳定的开发环境。
```
# 首先在本地环境中下载相关仓库
# 如果只运行不进行push，下面repo init的链接可以改成https://github.com/reL4team2/rel4-dev-repo.git，否则您需要具有该仓库的权限
mkdir rel4_dev && cd rel4_dev
repo init -u git@github.com:rel4team2/rel4-dev-repo.git --repo-url=https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/
repo sync

# 拉取镜像并进入docker中（这里最后的点是上面定义的文件夹名的路径）
curl --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/rel4team2/build-scripts/refs/heads/master/scripts/start_docker.sh | bash -s -- .
# 或者其实可以第二种方式构建（遇到curl的网络问题可以使用）
cat ./build-scripts/scripts/start_docker.sh | bash -s -- .
docker exec -u ${USER} -it rel4_dev bash

# 在docker中运行sel4-test测试
# -p参数指定platform，目前可选为spike（riscv64）和qemu-arm-virt（aarch64）
# -m参数指定MCS特性（混合关键系统）开启（on）与否（off）
# -s参数指定arm平台下的SMC特性开启（on）与否（off）
# --arm-ptmr参数开启arm平台下的KernelArmExportPCNTUser，默认关闭
# --arm-pcnt参数开启arm平台下的KernelArmExportPTMRUser，默认关闭
# --bin参数启用一个完整的rust内核，默认关闭并通过和原生seL4的C代码链接起来的方式构建以方便调试
# -N指定smp下使用的核心数量（需要在simulate的时候也增加--cpu-num参数），目前仅支持bin模式

cd rel4_kernel

# 参数根据上面的内容进行选择，以下仅为几个例子
cargo xtask build -p spike -m off -s on&& cd build && ./simulate && cd ..

# 使用smp的例子
cargo xtask build -p qemu-arm-virt -N 4 --bin && cd build && ./simulate --cpu-num 4 && cd ..

# 如果需要进行提交，exit命令离开docker，进入对应的文件夹进行提交
```
#### 使用reL4-Linux-kit进行兼容Linux syscall的测例运行

为了让用户态快速地部署内核而无需跟运行内核测例一样，我们也实现了一个reL4-cli用于支持用户态reL4-Linux-kit的运行。
对于reL4-Linux-kit的使用，见该仓库的readme

```
# 下载该仓库代码，同样无需提交也可使用https的链接
git clone git@github.com:reL4team2/reL4-linux-kit.git

cd reL4-linux-kit

# 安装aarch64的musl gcc的工具链，如果已经安装请忽略
wget https://musl.cc/aarch64-linux-musl-cross.tgz
tar zxf aarch64-linux-musl-cross.tgz
export PATH=$PATH:`pwd`/aarch64-linux-musl-cross/bin

# 安装sel4-kernel-loader-add-payload
cargo install --git https://github.com/seL4/rust-sel4 --rev 1cd063a0f69b2d2045bfa224a36c9341619f0e9b sel4-kernel-loader-add-payload

# 下载seL4内核，如果需要使用reL4内核，见后文的reL4-cli使用
mkdir -p .env
wget -qO- https://github.com/yfblock/rel4-kernel-autobuild/releases/download/release-2025-03-26/seL4.tar.gz | gunzip | tar -xvf - -C .env --strip-components 1

#运行
tools/app-parser.py kernel-thread block-thread uart-thread
make disk_img
make run LOG=error

```
按照上述步骤即可正常运行，实际命令以该仓库readme为准

但是上述的代码是使用seL4作为内核来运行的，以下是使用reL4-cli构建的方式

```
# 安装reL4-cli工具
cargo install --git https://github.com/reL4team2/reL4-cli.git
# 在上述reL4-linux-kit的下载seL4内核的步骤，使用下面的命令替换以使用reL4，如果已经存在.env文件夹，需要清理
cd rel4-linux-kit
mkdir -p .env
rel4-cli install kernel --bin -P $(realpath .)/.env/seL4
```
对于reL4-cli工具的一些选项描述如下

```
Options:
  -p, --platform <PLATFORM>        The target platform to install [default: qemu-arm-virt]
  -m, --mcs                        Enable kernel mcs mode
      --nofastpath                 Disable fastpath
  -B, --bin                        Rel4 has two modes: - Binary mode (pure Rust) - Lib mode (integrates with seL4 kernel)
  -P, --sel4-prefix <SEL4_PREFIX>  [default: /workspace/.seL4]
  -h, --help                       Print help (see more with '--help')
```
具体选项请参考reL4-cli的help

#### 使用特定的reL4内核和用户态程序进行构建运行

在使用reL4-cli工具时，其安装的reL4微内核是最新的主线分支的代码。但是如果需要进行代码调试，需要使用不在主线的代码，也可以在上述rel4-cli install kernel的步骤使用如下指令替换：

```
# 其中YOUR_REL4_INTEGRAL_PATH是本地的rel4_kernel仓库的路径
rel4-cli install kernel --bin --local ${YOUR_REL4_INTEGRAL_PATH}
```
即可在自定义的reL4内核上运行用户态测例。

### reL4内核评测
#### reL4的原生seL4测例通过情况

在reL4内核添加相关feature之后，相关feature之间具有不同的组合，存在有的feature不能兼容，而有的feature需要同时出现等。原生的seL4的test依据不同的feature，在测试中也会给出不同的测试样例。在本章节，我们给出了reL4内核在我们的测试环境下，相关测例的通过和未通过的情况。

我们使用的测试环境为：



该表格的含义如下：纵轴为每个测例，横向为各个feature。如果某个测例需要相关feature。如测例1需要feature A和C，就会在测例1所在行的feature A和C所在格添加记录。在开启了feature A和C的情况下，如果这些测试可以正常通过，那么这些记录都会打上√，否则都会打上×（或者在备注中注明无法成功测试的原因）。

目前已经实现的feature所支持的测例均已成功运行。

| 测例id | 测例名称                 | 无条件添加                                   | SMP  | CONFIG_ARCH_ARM                                      | CONFIG_KERNEL_MCS                       | CONFIG_HAVE_TIMER | 备注                                          |
| ------ | ------------------------ | -------------------------------------------- | ---- | ---------------------------------------------------- | --------------------------------------- | ----------------- | --------------------------------------------- |
| 1      | SERSERV_PARENT_001       | √                                            |      |                                                      |                                         |                   |                                               |
| 2      | SERSERV_PARENT_002       | √                                            |      |                                                      |                                         |                   |                                               |
| 3      | SERSERV_PARENT_003       | √                                            |      |                                                      |                                         |                   |                                               |
| 4      | SERSERV_PARENT_004       | √                                            |      |                                                      |                                         |                   |                                               |
| 5      | SERSERV_PARENT_005       | √                                            |      |                                                      |                                         |                   |                                               |
| 6      | SERSERV_PARENT_006       | √                                            |      |                                                      |                                         |                   |                                               |
| 7      | SERSERV_PARENT_007       | √                                            |      |                                                      |                                         |                   |                                               |
| 8      | SERSERV_PARENT_008       | √                                            |      |                                                      |                                         |                   |                                               |
| 9      | SERSERV_PARENT_009       | √                                            |      |                                                      |                                         |                   |                                               |
| 10     | SERSERV_PARENT_010       | √                                            |      |                                                      |                                         |                   |                                               |
| 11     | SMPIRQ0001               |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_TRACK_KERNEL_ENTRIES这个feature |
| 12     | SMMU0001                 |                                              |      |                                                      |                                         |                   |                                               |
| 13     | SYSCALL0000              | √                                            |      |                                                      |                                         |                   |                                               |
| 14     | SYSCALL0001              | √                                            |      |                                                      |                                         |                   |                                               |
| 15     | SYSCALL0002              | √                                            |      |                                                      |                                         |                   |                                               |
| 16     | SYSCALL0003              | √                                            |      |                                                      |                                         |                   |                                               |
| 17     | SYSCALL0004              | √                                            |      |                                                      |                                         |                   |                                               |
| 18     | SYSCALL0005              | √                                            |      |                                                      |                                         |                   |                                               |
| 19     | SYSCALL0014              | √                                            |      |                                                      |                                         |                   |                                               |
| 20     | SYSCALL0015              | √                                            |      |                                                      |                                         |                   |                                               |
| 21     | SYSCALL0016              |                                              |      |                                                      | √，只有不定义mcs的时候才能测试          |                   |                                               |
| 22     | SYSCALL0017              | √                                            |      |                                                      |                                         |                   |                                               |
| 23     | SYSCALL0006              | √                                            |      |                                                      |                                         |                   |                                               |
| 24     | SYSCALL0010              | √                                            |      |                                                      |                                         |                   |                                               |
| 25     | SYSCALL0011              | √                                            |      |                                                      |                                         |                   |                                               |
| 26     | SYSCALL0012              | √                                            |      |                                                      |                                         |                   |                                               |
| 27     | SYSCALL0013              | √                                            |      |                                                      |                                         |                   |                                               |
| 28     | SYSCALL0018              |                                              |      |                                                      | √                                       |                   |                                               |
| 29     | TIMER0001                |                                              |      |                                                      |                                         | √                 |                                               |
| 30     | TIMER0002                |                                              |      |                                                      |                                         | √                 |                                               |
| 31     | STACK_ALIGNMENT_001      |                                              |      |                                                      |                                         |                   | ×，需要x86的特性                              |
| 32     | EPT0001                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 33     | EPT0002                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 34     | EPT0003                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 35     | EPT1001                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 36     | EPT1002                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 37     | EPT0004                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 38     | EPT0005                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 39     | EPT0006                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX和CONFIG_ARCH_IA32           |
| 40     | EPT0007                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX和CONFIG_ARCH_IA32           |
| 41     | EPT0008                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 42     | EPT0009                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 43     | EPT0010                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 44     | EPT0011                  |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 45     | BENCHMARK_0001           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_BENCHMARK_TRACK_UTILISATION     |
| 46     | BIND0001                 | √                                            |      |                                                      |                                         |                   |                                               |
| 47     | BIND0002                 | √                                            |      |                                                      |                                         |                   |                                               |
| 48     | BIND0003                 | √                                            |      |                                                      |                                         |                   |                                               |
| 49     | BIND0004                 | √                                            |      |                                                      |                                         |                   |                                               |
| 50     | BIND005                  |                                              |      |                                                      | √                                       |                   |                                               |
| 51     | BIND006                  |                                              |      |                                                      | √                                       |                   |                                               |
| 52     | BREAKPOINT_001           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HARDWARE_DEBUG_API              |
| 53     | BREAKPOINT_002           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HARDWARE_DEBUG_API              |
| 54     | BREAKPOINT_003           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HARDWARE_DEBUG_API              |
| 55     | BREAKPOINT_004           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HARDWARE_DEBUG_API              |
| 56     | BREAKPOINT_005           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HARDWARE_DEBUG_API              |
| 57     | BREAKPOINT_006           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HARDWARE_DEBUG_API              |
| 58     | BREAKPOINT_007           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HARDWARE_DEBUG_API              |
| 59     | BREAK_REQUEST_001        |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HARDWARE_DEBUG_API              |
| 60     | SINGLESTEP_001           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HARDWARE_DEBUG_API              |
| 61     | CACHEFLUSH0001           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HAVE_CACHE                      |
| 62     | CACHEFLUSH0002           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HAVE_CACHE                      |
| 63     | CACHEFLUSH0003           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HAVE_CACHE                      |
| 64     | CACHEFLUSH0004           |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_HAVE_CACHE                      |
| 65     | CNODEOP0001              | √                                            |      |                                                      |                                         |                   |                                               |
| 66     | CNODEOP0002              | √                                            |      |                                                      |                                         |                   |                                               |
| 67     | CNODEOP0003              | √                                            |      |                                                      |                                         |                   |                                               |
| 68     | CNODEOP0004              | √                                            |      |                                                      |                                         |                   |                                               |
| 69     | CNODEOP0005              | √                                            |      |                                                      |                                         |                   |                                               |
| 70     | CNODEOP0006              | √                                            |      |                                                      |                                         |                   |                                               |
| 71     | CNODEOP0007              | √                                            |      |                                                      |                                         |                   |                                               |
| 72     | CNODEOP0008              | √                                            |      |                                                      |                                         |                   |                                               |
| 73     | CNODEOP0009              |                                              |      |                                                      | √，取反，就是没有这个特性的时候可以运行 |                   |                                               |
| 74     | CSPACE0001               | √                                            |      |                                                      |                                         |                   |                                               |
| 75     | DOMAINS0004              |                                              |      |                                                      |                                         | √                 |                                               |
| 76     | DOMAINS0005              |                                              | √    |                                                      |                                         | √                 |                                               |
| 77     | DOMAINS0001              | √                                            |      |                                                      |                                         |                   |                                               |
| 78     | DOMAINS0002              | √                                            |      |                                                      |                                         |                   |                                               |
| 79     | DOMAINS0003              | √                                            |      |                                                      |                                         |                   |                                               |
| 80     | CANCEL_BADGED_SENDS_0001 | √                                            |      |                                                      |                                         |                   |                                               |
| 81     | CANCEL_BADGED_SENDS_0002 | √                                            |      |                                                      |                                         |                   |                                               |
| 82     | PAGEFAULT0001            |                                              |      |                                                      |                                         |                   | √，没有CONFIG_FT的时候可以运行                |
| 83     | PAGEFAULT0002            |                                              |      |                                                      |                                         |                   | √，没有CONFIG_FT的时候可以运行                |
| 84     | PAGEFAULT0003            |                                              |      |                                                      |                                         |                   | √，没有CONFIG_FT的时候可以运行                |
| 85     | PAGEFAULT0004            | √                                            |      |                                                      |                                         |                   |                                               |
| 86     | PAGEFAULT0005            | √                                            |      |                                                      |                                         |                   |                                               |
| 87     | PAGEFAULT1001            | √                                            |      |                                                      |                                         |                   |                                               |
| 88     | PAGEFAULT1002            | √                                            |      |                                                      |                                         |                   |                                               |
| 89     | PAGEFAULT1003            | √                                            |      |                                                      |                                         |                   |                                               |
| 90     | PAGEFAULT1004            | √                                            |      |                                                      |                                         |                   |                                               |
| 91     | PAGEFAULT1005            | ×，无条件的false，无论如何都不会运行到该测例 |      |                                                      |                                         |                   |                                               |
| 92     | TIMEOUTFAULT0001         |                                              |      |                                                      | √                                       |                   |                                               |
| 93     | TIMEOUTFAULT0002         |                                              |      |                                                      | √                                       |                   |                                               |
| 94     | TIMEOUTFAULT0003         |                                              |      |                                                      | √                                       |                   |                                               |
| 95     | UNKNOWN_SYSCALL_001      |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_VTX                             |
| 96     | FPU0000                  | √                                            |      |                                                      |                                         |                   |                                               |
| 97     | FPU0001                  |                                              |      |                                                      |                                         |                   | √，没有CONFIG_FT的时候可以运行                |
| 98     | FPU0002                  |                                              | √    |                                                      |                                         |                   |                                               |
| 99     | FRAMEEXPORTS0001         | √                                            |      |                                                      |                                         |                   |                                               |
| 100    | FRAMEXN0001              |                                              |      | √                                                    |                                         |                   |                                               |
| 101    | FRAMEXN0002              |                                              |      | √                                                    |                                         |                   |                                               |
| 102    | FRAMEDIPC0001            |                                              |      |                                                      |                                         |                   | √，没有CONFIG_PLAT_SPIKE可以运行              |
| 103    | FRAMEDIPC0002            |                                              |      |                                                      |                                         |                   | √，没有CONFIG_PLAT_SPIKE可以运行              |
| 104    | FRAMEDIPC0003            | √                                            |      |                                                      |                                         |                   |                                               |
| 105    | RETYPE0000               | √                                            |      |                                                      |                                         |                   |                                               |
| 106    | RETYPE0001               | √                                            |      |                                                      |                                         |                   |                                               |
| 107    | RETYPE0002               | √                                            |      |                                                      |                                         |                   |                                               |
| 108    | INTERRUPT0002            |                                              |      |                                                      | √                                       | √                 |                                               |
| 109    | INTERRUPT0003            |                                              |      |                                                      | √                                       | √                 |                                               |
| 110    | INTERRUPT0004            |                                              |      |                                                      | √                                       | √                 |                                               |
| 111    | INTERRUPT0005            |                                              |      |                                                      | √                                       | √                 |                                               |
| 112    | INTERRUPT0006            |                                              |      |                                                      | √                                       | √                 |                                               |
| 113    | IOPORTS1000              |                                              |      |                                                      |                                         |                   | ×，需要x86                                    |
| 114    | IOPT0001                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_IOMMU                           |
| 115    | IOPT0002                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_IOMMU                           |
| 116    | IOPT0004                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_IOMMU                           |
| 117    | IOPT0008                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_IOMMU                           |
| 118    | IOPT0009                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_IOMMU                           |
| 119    | IOPT0011                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_IOMMU                           |
| 120    | IOPT0001                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_TK1_SMMU                        |
| 121    | IOPT0002                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_TK1_SMMU                        |
| 122    | IOPT0004                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_TK1_SMMU                        |
| 123    | IOPT0008                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_TK1_SMMU                        |
| 124    | IOPT0009                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_TK1_SMMU                        |
| 125    | IPCRIGHTS0001            | √                                            |      |                                                      |                                         |                   |                                               |
| 126    | IPCRIGHTS0002            | √                                            |      |                                                      |                                         |                   |                                               |
| 127    | IPCRIGHTS0003            | √                                            |      |                                                      |                                         |                   |                                               |
| 128    | IPCRIGHTS0004            |                                              |      |                                                      | √，取反，就是没有这个特性的时候可以运行 |                   |                                               |
| 129    | IPCRIGHTS0005            |                                              |      |                                                      | √，取反，就是没有这个特性的时候可以运行 |                   |                                               |
| 130    | IPC0001                  | √                                            |      |                                                      |                                         |                   |                                               |
| 131    | IPC0002                  | √                                            |      |                                                      |                                         |                   |                                               |
| 132    | IPC0003                  | √                                            |      |                                                      |                                         |                   |                                               |
| 133    | IPC0004                  | √                                            |      |                                                      |                                         |                   |                                               |
| 134    | IPC1001                  | √                                            |      |                                                      |                                         |                   |                                               |
| 135    | IPC1002                  | √                                            |      |                                                      |                                         |                   |                                               |
| 136    | IPC1003                  | √                                            |      |                                                      |                                         |                   |                                               |
| 137    | IPC1004                  | √                                            |      |                                                      |                                         |                   |                                               |
| 138    | IPC0010                  | √                                            |      |                                                      |                                         |                   |                                               |
| 139    | IPC0011                  |                                              |      |                                                      | √                                       |                   |                                               |
| 140    | IPC0012                  |                                              |      |                                                      | √                                       |                   |                                               |
| 141    | IPC0013                  |                                              |      |                                                      | √                                       |                   |                                               |
| 142    | IPC0014                  |                                              |      |                                                      | √                                       |                   |                                               |
| 143    | IPC0015                  |                                              |      |                                                      | √                                       |                   |                                               |
| 144    | IPC0016                  |                                              |      |                                                      | √                                       |                   |                                               |
| 145    | IPC0017                  |                                              |      |                                                      | √                                       |                   |                                               |
| 146    | IPC0018                  |                                              |      |                                                      | √                                       |                   |                                               |
| 147    | IPC0019                  |                                              |      |                                                      | √                                       |                   |                                               |
| 148    | IPC0020                  |                                              |      |                                                      | √                                       |                   |                                               |
| 149    | IPC0021                  |                                              |      |                                                      | √                                       |                   |                                               |
| 150    | IPC0022                  |                                              |      |                                                      | √                                       |                   |                                               |
| 151    | IPC0023                  |                                              |      |                                                      | √                                       |                   |                                               |
| 152    | IPC0024                  |                                              |      |                                                      | √                                       |                   |                                               |
| 153    | IPC0025                  |                                              |      |                                                      | √                                       |                   |                                               |
| 154    | IPC0026                  |                                              |      |                                                      | √                                       |                   |                                               |
| 155    | IPC0027                  |                                              |      |                                                      | √                                       |                   |                                               |
| 156    | IPC0028                  | ×，无条件的false，无论如何都不会运行到该测例 |      |                                                      |                                         |                   |                                               |
| 157    | MULTICORE0001            |                                              | √    |                                                      |                                         | √                 |                                               |
| 158    | MULTICORE0002            |                                              | √    |                                                      |                                         | √                 |                                               |
| 159    | MULTICORE0005            |                                              | √    |                                                      |                                         | √                 |                                               |
| 160    | MULTICORE0003            |                                              | √    |                                                      |                                         | √                 |                                               |
| 161    | MULTICORE0004            |                                              | √    |                                                      |                                         |                   |                                               |
| 162    | NBWAIT0001               | √                                            |      |                                                      |                                         |                   |                                               |
| 163    | PT0001                   |                                              |      | √                                                    |                                         |                   |                                               |
| 164    | PT0002                   |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_ARCH_AARCH32                    |
| 165    | PREEMPT_REVOKE           |                                              |      |                                                      |                                         | √                 |                                               |
| 166    | REGRESSIONS0001          | √                                            |      |                                                      |                                         |                   |                                               |
| 167    | REGRESSIONS0002          |                                              |      | √                                                    |                                         |                   |                                               |
| 168    | REGRESSIONS0003          |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_ARCH_IA32                       |
| 169    | SCHED_CONTEXT_0001       |                                              |      |                                                      | √                                       |                   |                                               |
| 170    | SCHED_CONTEXT_0002       |                                              |      |                                                      | √                                       | √                 |                                               |
| 171    | SCHED_CONTEXT_0003       |                                              |      |                                                      | √                                       |                   |                                               |
| 172    | SCHED_CONTEXT_0005       |                                              |      |                                                      | √                                       | √                 |                                               |
| 173    | SCHED_CONTEXT_0006       |                                              |      |                                                      | √                                       |                   |                                               |
| 174    | SCHED_CONTEXT_0007       |                                              |      |                                                      | √                                       |                   |                                               |
| 175    | SCHED_CONTEXT_0008       |                                              |      |                                                      | √                                       |                   |                                               |
| 176    | SCHED_CONTEXT_0009       |                                              |      |                                                      | √                                       | √                 |                                               |
| 177    | SCHED_CONTEXT_0010       |                                              |      |                                                      | √                                       | √                 |                                               |
| 178    | SCHED_CONTEXT_0011       |                                              |      |                                                      | √                                       | √                 |                                               |
| 179    | SCHED_CONTEXT_0012       |                                              |      |                                                      | √                                       | √                 |                                               |
| 180    | SCHED_CONTEXT_0013       |                                              |      |                                                      | √                                       | √                 |                                               |
| 181    | SCHED_CONTEXT_0014       |                                              | √    |                                                      | √                                       |                   |                                               |
| 182    | SCHED0000                |                                              |      |                                                      |                                         | √                 |                                               |
| 183    | SCHED0002                | √                                            |      |                                                      |                                         |                   |                                               |
| 184    | SCHED0003                |                                              |      |                                                      |                                         |                   | √，没有CONFIG_FT可以运行                      |
| 185    | SCHED0004                | √                                            |      |                                                      |                                         |                   |                                               |
| 186    | SCHED0005                |                                              |      |                                                      |                                         |                   | √，CONFIG_NUM_PRIORITIES>=7可以运行           |
| 187    | SCHED0006                |                                              |      |                                                      | √，取反，就是没有这个特性的时候可以运行 |                   |                                               |
| 188    | SCHED0007                |                                              |      |                                                      | √                                       |                   |                                               |
| 189    | SCHED0008                |                                              |      |                                                      | √                                       | √                 |                                               |
| 190    | SCHED0009                |                                              |      |                                                      | √                                       | √                 |                                               |
| 191    | SCHED0010                |                                              |      |                                                      | √                                       | √                 |                                               |
| 192    | SCHED0011                |                                              |      |                                                      | √                                       | √                 |                                               |
| 193    | SCHED0012                |                                              |      |                                                      | √                                       | √                 |                                               |
| 194    | SCHED0013                |                                              |      |                                                      | √                                       | √                 |                                               |
| 195    | SCHED0014                |                                              |      |                                                      | √                                       | √                 |                                               |
| 196    | SCHED0016                |                                              |      |                                                      | √                                       |                   |                                               |
| 197    | SCHED0017                |                                              |      |                                                      | √                                       |                   |                                               |
| 198    | SCHED0018                |                                              |      |                                                      | √                                       | √                 |                                               |
| 199    | SCHED0019                |                                              |      |                                                      | √                                       |                   |                                               |
| 200    | SCHED0020                | √                                            |      |                                                      |                                         |                   |                                               |
| 201    | SCHED0021                |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_SIMULATION                      |
| 202    | SCHED0022                |                                              | √    |                                                      | √                                       |                   |                                               |
| 203    | SERSERV_CLIENT_001       | √                                            |      |                                                      |                                         |                   |                                               |
| 204    | SERSERV_CLIENT_002       | √                                            |      |                                                      |                                         |                   |                                               |
| 205    | SERSERV_CLIENT_003       | √                                            |      |                                                      |                                         |                   |                                               |
| 206    | SERSERV_CLIENT_004       | √                                            |      |                                                      |                                         |                   |                                               |
| 207    | SERSERV_CLIENT_005       | √                                            |      |                                                      |                                         |                   |                                               |
| 208    | SERSERV_CLI_PROC_001     | √                                            |      |                                                      |                                         |                   |                                               |
| 209    | SERSERV_CLI_PROC_002     | √                                            |      |                                                      |                                         |                   |                                               |
| 210    | SERSERV_CLI_PROC_003     | √                                            |      |                                                      |                                         |                   |                                               |
| 211    | SERSERV_CLI_PROC_004     | √                                            |      |                                                      |                                         |                   |                                               |
| 212    | SERSERV_CLI_PROC_005     | √                                            |      |                                                      |                                         |                   |                                               |
| 213    | SMC0001                  |                                              |      |                                                      |                                         |                   | √，需要CONFIG_ALLOW_SMC_CALLS                 |
| 214    | SMC0002                  |                                              |      |                                                      |                                         |                   | √，需要CONFIG_ALLOW_SMC_CALLS                 |
| 215    | SMC0003                  |                                              |      |                                                      |                                         |                   | √，需要CONFIG_ALLOW_SMC_CALLS                 |
| 216    | SMC0004                  |                                              |      |                                                      |                                         |                   | √，需要CONFIG_ALLOW_SMC_CALLS                 |
| 217    | SMC0005                  |                                              |      |                                                      |                                         |                   | √，需要CONFIG_ALLOW_SMC_CALLS                 |
| 218    | SMC0006                  |                                              |      |                                                      |                                         |                   | √，需要CONFIG_ALLOW_SMC_CALLS                 |
| 219    | SMC0007                  |                                              |      |                                                      |                                         |                   | √，需要CONFIG_ALLOW_SMC_CALLS                 |
| 220    | SMC0008                  |                                              |      |                                                      |                                         |                   | √，需要CONFIG_ALLOW_SMC_CALLS                 |
| 221    | SYNC001                  | √                                            |      |                                                      |                                         |                   |                                               |
| 222    | SYNC002                  | √                                            |      |                                                      |                                         |                   |                                               |
| 223    | SYNC003                  | √                                            |      |                                                      |                                         |                   |                                               |
| 224    | SYNC004                  | √                                            |      |                                                      |                                         |                   |                                               |
| 225    | THREADS0004              | √                                            |      |                                                      |                                         |                   |                                               |
| 226    | THREADS0005              | √                                            |      |                                                      |                                         |                   |                                               |
| 227    | TLS0001                  | √                                            |      |                                                      |                                         |                   |                                               |
| 228    | TLS0002                  | √                                            |      |                                                      |                                         |                   |                                               |
| 229    | TLS0006                  | √                                            |      |                                                      |                                         |                   |                                               |
| 230    | TRIVIAL0000              | √                                            |      |                                                      |                                         |                   |                                               |
| 231    | TRIVIAL0001              | √                                            |      |                                                      |                                         |                   |                                               |
| 232    | TRIVIAL0002              | √                                            |      |                                                      |                                         |                   |                                               |
| 233    | VCPU0001                 |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_ARM_HYPERVISOR_SUPPORT          |
| 234    | VSPACE0000               | √                                            |      |                                                      |                                         |                   |                                               |
| 235    | VSPACE0001               |                                              |      | √，同样的aarch32和aarch64版本都存在，只取aarch64版本 |                                         |                   |                                               |
| 236    | VSPACE0002               | √                                            |      |                                                      |                                         |                   |                                               |
| 237    | VSPACE0003               | √                                            |      |                                                      |                                         |                   |                                               |
| 238    | VSPACE0004               | √                                            |      |                                                      |                                         |                   |                                               |
| 239    | VSPACE0005               | √                                            |      |                                                      |                                         |                   |                                               |
| 240    | VSPACE0006               | √                                            |      |                                                      |                                         |                   |                                               |
| 241    | VSPACE0010               |                                              |      |                                                      |                                         |                   | ×，需要CONFIG_ARCH_IA32                       |



### reL4-linux-kit评测
