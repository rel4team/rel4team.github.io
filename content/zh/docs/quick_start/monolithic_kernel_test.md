---
weight: 2
bookCollapseSection: false
bookFlatSection: true
title: "reL4宏内核测例运行"
---

# reL4宏内核测例运行

我们设计了相关的rust语言编写的[宏内核测例](https://github.com/reL4team2/rel4-linux-kit)，用于说明reL4可以支持部分使用linux syscall的程序，该测例的默认运行环境是seL4下

为了自动化支持相关的测例，我们也构建了[相关脚本](https://github.com/reL4team2/rel4-kernel-autobuild/tree/rel4-integrate)，在这里，根据内核的编译方式，分别提供了lib方式（即依然和很少一部分C代码链接在一起的构建方式，主要是用于调试）和bin方式（reL4作为一个独立的内核和宏内核测例相链接）


## 在reL4上构建其他自己的内核

由于reL4内核和seL4内核一样，可以作为一个ELF镜像单独构建并通过kernel_loader和自定义的用户态代码连接在一起，因此，你也可以构建自己的root_task等用户态代码，并基于reL4内核，在此基础上构建相应的代码。相关代码可以参考上述脚本。