---
title: "本地环境安装"
weight: 1
# bookToc: true
# bookHidden: false
# bookComments: false
---

## 1. 简介

rel4 目前本质上是 seL4 链接的一个静态库，因此二者可以在同一个环境中编译、运行。换句话说，rel4 只需要在 seL4 编译运行环境的基础上增加 rust 安装即可。

我已经写了一个环境安装脚本，本文会根据该脚本介绍安装过程。

> 脚本在 Ubuntu20.04 和 Ubuntu22.04 经过测试，其他环境无法执行。

> 由于网络原因，在 Docker 中进行开发比较痛苦，加上之前已经有做过的 rel4 开发镜像，因此本文只介绍如何在本地环境安装 rel4 && seL4 依赖

## 2. 快速安装

如果你不想了解细节，可以执行脚本安装。执行脚本安装前，最好配置好科学上网，否则无法保证执行成功。

```
curl --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/croakexciting/rel4_doc/main/scripts/install.sh | bash
```

如果因为网络原因，无法访问 raw.githubusercontent，可以 clone 后执行

```
git clone https://github.com/croakexciting/rel4_doc.git
cd rel4_doc
bash scripts/install.sh
```

如果因为网络原因，无法下载 qemu 和 riscv toolchain，请看下面环境安装介绍。

## 3. seL4 && rel4 runtime 环境安装

seL4 && rel4 环境安装基本按照 [seL4 环境配置文档](https://docs.sel4.systems/projects/buildsystem/host-dependencies.html) 和 [rel4 dockerfile](https://github.com/yfblock/rel4-docker/blob/main/Dockerfile) 执行。

```
function install_rel4_runtime() {
    install_apt
    install_pip
    install_qemu_and_toolchain
    install_rust

    echo "export PATH=\${PATH}:${HOME}/.local/bin:${HOME}/Downloads/riscv/bin" >> ${HOME}/.bashrc
    echo "source \$HOME/.cargo/env" >> ${HOME}/.bashrc
}
```

其中需要注意的是 qemu 和 toolchain 下载过程可能会失败。如果网络不好，可以下载后放到 ~/Downloads 下并解压，脚本会跳过下载步骤

```
mv qemu-8.2.5.tar.xz ~/Downloads
mv riscv.tar.gz ~/Downloads
cd ~/Downloads
tar xvJf qemu-8.2.5.tar.xz
tar xzvf riscv.tar.gz
```

## 4. rust-sel4 runtime 环境安装

使用 rust-sel4 时，可以使用其 [docker 环境](https://github.com/seL4/rust-root-task-demo) 运行 demo 程序。同样为了简化使用，install 脚本中可以将该环境安装到本地

> 该环境中的 seL4 kernel.elf 与 rel4 无关，版本也不同。所以目前只是按照 rust-sel4 官方文档，在 seL4 原生 kernel 上运行 demo。后续会将 kernel.elf 替换为 rel4 项目中的 kernel

> 由于安装该脚本需要 rust-2024-09-01 以上版本，因此不要在有 rust-toolchain.toml 的文件夹中运行该脚本，以免 rust 版本冲突


```
git clone https://github.com/croakexciting/rel4_doc.git
cd rel4_doc

# install all, include rel4 runtime and rust-seL4 runtime env
bash scripts/install.sh -a

# or only install rust-sel4 runtime env
bash scripts/install.sh -r

```
