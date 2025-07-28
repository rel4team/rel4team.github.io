---
weight: 3
bookCollapseSection: false
bookFlatSection: true
title: "reL4快速上手"
---

# 快速上手概览

目前reL4的组织迁移到了[这里](https://github.com/reL4team2)，在组织下包括了我们使用到的全部代码、脚本等，内核的主仓库在[这里](https://github.com/reL4team2/rel4-integral)

注：由于reL4项目代码本身尚处在快速变动期，如果你发现本章节的内容发生了巨大变化以至于rel4无法工作，请通过[github仓库](https://github.com/reL4team2/rel4-integral)提交issue及时联系我们，非常感谢！

# 各种测试构建说明

目前rel4项目构建已经比较复杂，因此在此进行具体的说明。

rel4项目目前存在的构建和测试需求如下：
 - 以不同的配置参数运行[sel4原生内核测例](./sel4_test.md)
 - 以特定的配置参数运行我们编写的基于rel4内核和reL4-Linux-kit兼容层的[用户态程序](./monolithic_kernel_test)。
 - 使用特定的reL4内核和用户态程序进行联合构建运行

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
