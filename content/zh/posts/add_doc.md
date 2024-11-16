---
title: "如何在 Book 中增加文档"
date: 2024-11-14T14:59:48+08:00
# bookComments: false
# bookSearchExclude: false
---

## 1. 介绍

本文介绍如何在 rel4 book 中增加文档。目前文档分为两类

- Rel4 手册 
  
  手册的内容需要按照合理的目录结构组织，文档主要介绍 Rel4 的设计、实现、安装、运行等等，内容正式。

- 博客：
  
  博客内容随意，记录零散的知识点，没有目录组织。


## 2. 增加博客

从简单开始，先介绍如何增加一篇博客。

1. 获取 rel4team.github.io 项目到本地

```
git clone --recurse-submodules https://github.com/rel4team/rel4team.github.io.git

# or

git clone https://github.com/rel4team/rel4team.github.io.git
cd rel4team.github.io
git submodule update --init --recursive
```

2. 增加一个 markdown 文件到 content/zh/posts 下

```
hugo new content/zh/posts/{YOUR FILE}.md
```

3. 完成你的文档，如果确认可发布，提交 pull request 到 main 分支，合入后会自动发布

## 3. 增加手册文档

相比增加博客，手册文档需要考虑目录结构，可能需要增加目录，其他和增加博客一样的操作不在赘述。

首先我们要知道把文档放在哪个目录下，比如 docs/install 下，对应的就是 **环境安装** 章节。之所以在侧边栏会产生一个章节，是因为 install 文件夹下有个 _index.md

```
# docs/install/_index.md

---
weight: 1
bookCollapseSection: true
title: "环境安装"
---

## 环境安装

本章节介绍 reL4, seL4, rust-sel4 等项目的编译、运行安装脚本。

```

--- 框内是文档的属性配置

- weight: 数字越小在侧边栏越靠前，用于目录排版
- bookCollapseSection: 目录里内容是否默认折叠
- title: 目录标题

**所以，如果你想创建一个章节，需要在你的父章节路径下创建一个文件夹，并且在其中加上 _index.md**

章节路径下的所有 markdown 文件会显示在该章节下

