---
weight: 3
title: "运行 Linux 程序"
commentsId: 1 
---

# 运行 Linux 程序

该架构是 `rel4-linux-kit` 的设计方案，它基于 `seL4/reL4` 微内核，在其之上运行了多个用户态组件和 `Linux` 应用。整个系统被划分为内核态和用户态。

- 内核态
运行 `seL4/reL4` 微内核，提供进程调度、内存管理、IPC 以及安全隔离等基本服务。

- 用户态
由多个服务组件和 `Linux` 应用组成

## 架构图

![](/monolithic/latest/design/design-1.png)

## 详细说明

### 组件及说明

1. root-task
  - root-task 是系统的初始用户态进程，负责启动其他用户态组件，以及提供资源的管理和传递。
2. blk-thread
  - blk-thread 提供块设备驱动，管理磁盘 I/O。
3. ext4-thread
  - 负责文件系统的操作，依赖 blk-thread 来读取磁盘数据。
4. uart-thread
  - 串口驱动，处理输入输出相关的任务。
5. kernel-thread
  - kernel-thread 负责启动 Linux 应用，并为 Linux 应用运行时遇到的需要处理的系统调用提供支持。


### 设计说明

在 rel4-linux-kit 的启动阶段，root-task 负责启动所有服务进程。这些服务进程之间可能存在相互依赖，因此需要通过向 root-task 发送 find_service 请求来获取所需的 EndPoint，从而建立进程间通信。

Linux 应用由 kernel-thread 启动并提供服务。在 [二进制兼容]({{< ref "/docs/monolithic/latest/bin-compact.md" >}}) 文档中，我们介绍了当前提供的二进制兼容方案。kernel-thread 需要在 Linux-App 触发 FaultIPC 时提供支持。由于 kernel-thread 需要持续等待 IPC 请求，它是阻塞的，无法处理定时任务或唤醒其他服务。为了解决这一问题，kernel-thread 会创建一个新的线程来专门处理定时器。当用户态任务请求 sleep 时，kernel-thread 负责记录任务的等待时间并将其挂起，等待唤醒。kernel-thread-1 负责管理定时器，并在超时后唤醒相应的任务。


## 运行案例

### 环境安装

需要保证您的环境包含下列软件或功能：

- `rustup`
- `make`
- `python3`
- `git`

```shell
git clone https://github.com/reL4team2/rel4-linux-kit.git
cd rel4-linux-kit
git checkout 610451a

# 安装 aarch64 musl cross toolchain
wget https://musl.cc/aarch64-linux-musl-cross.tgz
tar zxf aarch64-linux-musl-cross.tgz
export PATH=$PATH:`pwd`/aarch64-linux-musl-cross/bin

# 下载 sel4 内核
mkdir -p .env
wget -qO- https://github.com/yfblock/rel4-kernel-autobuild/releases/download/release-2025-03-06/seL4.tar.gz | gunzip | tar -xvf - -C .env --strip-components 1

# 下载测例
wget -qO- https://github.com/yfblock/rel4-kernel-autobuild/releases/download/release-2025-03-06/aarch64.tgz | tar -xf - -C .env
mkdir -p testcases

# (Optional) 如果您的系统不支持直接使用 pip 安装包
python3 -m venv .env/python
source .env/python/bin/activate

# 安装 python 包
pip3 install capstone lief

# 处理测例
./tools/modify-multi.py .env/aarch64 testcases
```

### 运行

```shell
make run LOG=error
```

如果您想看到更加详细的测例，可以将 `LOG=error` 改为 `LOG=debug` 或其 `<trace> <debug> <info> <warn> <error>` 中的一个，这里的 `LOG` 只会影响 `kernel-thread`。