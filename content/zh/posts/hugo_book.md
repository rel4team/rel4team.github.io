---
title: "Rel4 Book 部署说明"
date: 2024-11-14T15:44:03+08:00
# bookComments: false
# bookSearchExclude: false
---

## 1 介绍

Rel4 Book 在线文档基于 [Hugo](https://gohugo.io/) 和 [Hugo Book Theme](https://hugo-book-demo.netlify.app/docs/example/) 搭建。

简单来说，Hugo 是一个快速且灵活的静态网站生成器，它能够将 Markdown 文件和其他内容转换为静态 HTML 页面，适合用来构建博客、文档和其他静态网站。它支持主题、数据文件、各种输出格式，并具有快速构建速度和易于扩展的特点。

Hugo Book Theme 是一个用于 Hugo 的主题，Theme 类似模板，依赖 Theme，我们就无需自己设计页面布局。

使用 Hugo 搭建静态网站非常方便，尤其是简单的博客网站。

## 2 创建网站

基本按照文档说明一步步执行即可

1. 安装 Hugo，参考[官方文档](https://gohugo.io/installation/)
2. 创建网站项目

```
hugo new site mydocs; cd mydocs
git init
git submodule add https://github.com/alex-shpak/hugo-book themes/hugo-book

# copy some example to content
cp -R themes/hugo-book/exampleSite/content.en/* ./content

hugo server --minify --theme hugo-book
```

## 3 网站配置

Hugo 定义了一些配置项，Theme 也会定义一些特有的配置，定义在 params 里，主要用到的配置项为。

