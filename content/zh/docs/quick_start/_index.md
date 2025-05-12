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
 - 以特定的配置参数运行我们编写的基于rel4内核的[用户态程序](./monolithic_kernel_test)。
 - 上述的用户态程序的运行基于最新的内核版本，如果需要进行内核和用户态测例的联合开发，请使用[rel4-cli](./rel4-cli.md)中关于内核态开发者的相关说明