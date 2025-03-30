---
title: "reL4-cli"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# reL4-cli

## 1. 简介

由于 reL4 项目分为[内核态](https://github.com/reL4team2/rel4-integral) 和 [用户态宏内核](https://github.com/reL4team2/rel4-linux-kit) 两部分，部署较为复杂，因此需要一个工具来帮助用户快速的部署 reL4 以及运行测试程序等。reL4-cli 就是这样一个工具。

rel4-cli 主要提供以下功能

- [x] 部署 reL4 内核，libseL4, sel4-loader，供用户态宏内核测试
- [ ] 安装运行环境，rust 库等等。比如后续 rust 版本更新可能不放在 docker 里，还是放在 rel4-cli 里
- [ ] 运行 sel4test，rel4-linux-kit testcase
- [ ] 编译内核态+用户态完整镜像

## 2. 安装

使用 cargo 安装 rel4-cli

```bash
cargo install --git https://github.com/reL4team2/reL4-cli.git
```

## 3. 使用

使用 rel4-cli -h 使用说明

### 3.1 部署 reL4 内核

```bash
rel4-cli install kernel -h

Install reL4 kernel, libseL4, kernel loader

Usage: rel4-cli install kernel [OPTIONS]

Options:
  -p, --platform <PLATFORM>        The target platform to install [default: qemu-arm-virt]
  -m, --mcs                        Enable kernel mcs mode
      --nofastpath                 Disable fastpath
  -B, --bin                        Rel4 has two modes: - Binary mode (pure Rust) - Lib mode (integrates with seL4 kernel)
  -P, --sel4-prefix <SEL4_PREFIX>  [default: /workspace/.seL4]
  -h, --help                       Print help (see more with '--help')
```

其中 -B 参数表示 reL4 内核的编译方式，reL4 内核的编译方式有两种：
- lib 模式：reL4 内核和 seL4 内核链接在一起，之前使用的方式
- bin 模式：reL4 内核单独编译成 一个 ELF 文件，和 seL4 内核没有任何关系，reL4 内核可以单独运行。该模式还在开发中，有更多的潜在 bug

-P 参数表示 seL4 内核的安装路径，默认是 /workspace/.seL4。
-p 参数表示目标硬件平台，默认是 qemu-arm-virt。reL4 内核目前支持的目标平台有：
- qemu-arm-virt
- spike

为了更直观的说明 reL4 内核的部署方式，我们举例说明，宏内核 rel4-linux-kit 开发者如何使用该工具部署更新 reL4 内核

```bash
cd rel4-linux-kit

# 更新 reL4 内核
rel4-cli install kernel --bin -P $(realpath .)/.env/seL4

# 运行 rel4-linux-kit 测例或者做相关的开发即可
make run LOG=error
```

### 3.2 运行测例

TODO

### 3.3 安装依赖环境

TODO

### 3.4 编译内核态+用户态完整镜像

TODO