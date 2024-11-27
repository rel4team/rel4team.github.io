---
title: "Rel4 Book 部署说明"
weight: 99
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
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

Hugo 定义了一些配置项，Theme 也会定义一些特有的配置，定义在 params 里，主要用到的配置项为

- 基础配置

```
# hugo.toml

baseURL = 'https://rel4team.github.io'
title = 'ReL4 Book'
theme = 'hugo-book'
```

- 语言配置，不配置默认为 English
  
```
[languages]
[languages.zh]
  languageName = '中文'
  contentDir = 'content/zh'
  weight = 1

[languages.en]
  languageName = 'English'
  contentDir = 'content/en'
  weight = 2

defaultContentLanguage = "zh"
defaultContentLanguageInSubdir = true
```

- 侧面菜单配置

```
[menu]
[[menu.after]]
  name = "Github"
  url = "https://github.com/rel4team"
  weight = 20
```

更多的 Hugo 配置可以参考 [Hugo Config](https://gohugo.io/getting-started/configuration/), 但通常我们不用了解这么多配置，如果只想搭一个 Blog 网站，只需要找到 Theme，根据 Theme 的说明进行配置即可。

## 4 增加评论

Hugo 可以集成一些评论组件，比较方便的是 [utterances](https://utteranc.es/), 这是一个基于 Github issue 的评论组件，所以我们无需注册额外的账号，非常适合技术文档使用。

Hugo Book Theme 自带支持 [disqus](https://disqus.com/) 评论组件，并且已经在模板中，将 partials/docs/comments.html 放在了页面 footer

```
# themes/hugo-book/layouts/_default/baseof.html:31

{{ template "comments" . }} <!-- Comments block -->
```

因此我们只需要将 comments.html 内容修改为 utterances 即可

我们无需在 theme 中修改，只需要在项目同样路径下增加一个同名文件，在其中加上修改后的内容即可。Hugo 在渲染时会优先选择项目路径下的文件。

比如，我们在 layouts/partials/docs 下增加了 comments.html 文件，该文件会覆盖 themes/hugo-book/layouts/partials/docs/comments.html

```
# layouts/partials/docs/comments.html

{{ if not .IsSection }}
    <script src="https://utteranc.es/client.js"
        repo="rel4team/rel4team.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
    </script>
{{ end }}
```

如上，按照 utterances 说明增加这段 JS 代码即可。在第一次评论时，会提示你在 Github 中安装 utterances，跟着步骤进行操作即可。

## 5 Github Page 部署

在 Github Pages 页面上选择 Hugo Action，会自动产生一个 workflow/hugo.yml, 简单易读，基本可以直接使用

```
# .github/workflows/hugo.yml

# Sample workflow for building and deploying a Hugo site to GitHub Pages
name: Deploy Hugo site to Pages

on:
  # Runs on pushes targeting the default branch
  push:
    branches: ["main"]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
concurrency:
  group: "pages"
  cancel-in-progress: false

# Default to bash
defaults:
  run:
    shell: bash

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.128.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb
      - name: Install Dart Sass
        run: sudo snap install dart-sass
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Install Node.js dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"
      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
        run: |
          hugo \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  # Deployment job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## 6 总结

本文说明了如何基于 Hugo 搭建一个简单的静态网站，并将其部署到 Github Page 上。总的来说，Hugo 是非常简单易用的，配合上 Github Page 使用，用户无需考虑搭建 http server，可以很快搭建一个静态网站。
