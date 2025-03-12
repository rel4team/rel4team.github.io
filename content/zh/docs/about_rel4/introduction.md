---
weight: 1
bookCollapseSection: false
title: reL4项目概述
commentsId: 1
---

# 关于reL4项目

reL4项目最初的目标是使用rust复刻seL4内核。该内核最初只是逐步将sel4的C语言部分的代码部分去除，并将去除的代码改写成一个rust语言编写的静态链接库，和seL4的原生代码进行链接，逐步进行相应的过程，最终将整个内核都变成rust语言实现的。

关于这个目标，目前已经达成的主要进度有：

 - [x] 通过seL4基本的测例

 - [x] 支持riscv64和aarch64两个架构

 - [x] 添加MCS特性，以更好的支持实时性的应用

 - [x] 使reL4能够作为一个完整独立的，不依赖于C代码的纯rust编写的内核

 - [x] 支持arm下的SMC等特性。

除此之外，我们还推进了以下的一些基于reL4的项目

 - [x] 使用riscv下的用户态中断扩展，优化reL4的IPC性能

 - [x] 基于reL4内核，在其之上实现一个简单的宏内核的功能