---
title: "如何在 Book 中增加文档"
date: 2024-11-14T14:59:48+08:00
type: 'docs'
# bookComments: false
# bookSearchExclude: false
---

## 1. 介绍

本文介绍如何在 rel4 book 中增加文档。

## 2. 增加文档

从简单开始，先介绍如何在已有文件夹下增加一篇文档。

1. 获取 rel4team.github.io 项目到本地

```
git clone --recurse-submodules https://github.com/rel4team/rel4team.github.io.git

# or

git clone https://github.com/rel4team/rel4team.github.io.git
cd rel4team.github.io
git submodule update --init --recursive
```

2. 增加一个 markdown 文件到 content/zh/doc/contribution/examples 下

```
hugo new content/zh/doc/contribution/examples/{YOUR FILE}.md
```

3. 完成你的文档，Hugo 编译预览，[Hugo 安装](https://gohugo.io/installation/)

```
hugo server

# open http://localhost:1313/
```

## 3. 增加章节

**Rel4 Book 中，章节分为两种**

- 根目录章节，content/zh/doc 目录下的第一节子目录

![根目录章节](/contribution/root_dir.png)

- 子目录章节，除了根目录外的都是子目录，可以有多级子目录

![子目录章节](/contribution/subdir.png)

章节如 docs/install，对应的就是 **环境安装** 章节。之所以在侧边栏会产生一个章节，是因为 install 文件夹下有个 _index.md

```
# docs/install/_index.md

---
weight: 1
bookCollapseSection: true
bookFlatSection: true
title: "环境安装"
---

## 环境安装

本章节介绍 reL4, seL4, rust-sel4 等项目的编译、运行安装脚本。

```

虚线框内是文档的属性配置

- weight: 数字越小在侧边栏越靠前，用于目录排版
- bookCollapseSection: 目录里内容是否默认折叠
- bookFlatSection: 平铺显示，好看一点
- title: 目录标题

**所以，如果你想创建一个章节，需要在你的父章节路径下创建一个文件夹，并且在其中加上 _index.md 章节路径下的所有 markdown 文件会显示在该章节下**

![路径与章节映射关系](/contribution/file_mapping.png)

上述说明是根目录章节，子目录章节中唯一有区别的是属性配置中，**不要增加 bookFlatSection: true**。

```
# docs/contribution/examples/_index.md

---
weight: 100
bookCollapseSection: true
title: "文档例子"
---
```

## 4 增加图片

如果想要增加图片，需要将图片放置在 static 目录下的子目录中。然后在 markdown 中使用添加图片标准语法，路径填 static 下的相对路径即可。


![增加图片](/contribution/add_picture.png)

## 5 增加latex公式

我找到了一个新的主题，他看上去只需要按照正常的markdown格式的latex的书写规则就行了，但是据说更高级的数学公式的用法，需要按照下面的说明文档来写

[说明文档](https://mcshelby.github.io/hugo-theme-relearn/shortcodes/math/index.html)

## 6 文章发布

当你希望发布文章时，只需提 pull request 将改动合入 main 分支，即可自动发布。