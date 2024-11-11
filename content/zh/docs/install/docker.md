---
title: "Docker 环境安装"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## 1. 简介

本文主要叙述如何使用 rel4-dev docker 镜像进行 sel4 和 rel4 的尝试。该镜像可以用于运行 sel4 和 rel4 例程，但是目前还不支持运行 rust-sel4 中 demo 程序。

## 2. 使用说明

### 2.1 进入 Docker 环境

```
# start_docker, only need execute once
# LOCAL WORKSPACE is the dir which map into docker container. Usually, it's you workspace, will map to /workspace in docker container docker. Please replace the {LOCAL WORKSPACE} with real dir path 
curl --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/croakexciting/rel4_doc/refs/heads/main/scripts/start_docker.sh | bash -s -- {LOCAL WORKSPACE}

# enter docker with your current user
docker exec -u ${USER} -it rel4_dev bash

# You should be in the rel4_dev_env now
croak@rel4_dev_env:/workspace
```

### 2.2 编译并运行

下面是一个在 docker 环境中使用的例子。比如运行 rel4 测试用例。

```
# rel4test is already downloaded in workspace
cd rel4test

# follow the rel4 guide
cd rel4_kernel/

make env

./build.py

cd build

./simulate
```

seL4 也是一样，按照官方指南中执行即可
