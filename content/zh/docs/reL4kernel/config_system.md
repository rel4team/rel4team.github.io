+++
date = '2025-01-25T09:47:15+08:00'
title = 'reL4 配置系统设计'
+++

# reL4 配置系统设计

## 1. 目标

由于去除 seL4 的编译系统，因此 seL4 基于 CMake 的配置系统同样会被去除。因此我们需要设计一个简化的、足够使用的配置系统供 reL4 使用。

目的是将所有的配置项统一管理，以适应各种 CPU 架构，各种 SoC 平台，减少增加 SoC 平台所需的工作量。主要包括以下几类配置项。

1. Memory layout
2. CPU 信息，主频之类 (也许不需要，dtc)
3. 一些定义在 config.rs 中的常量

编译过程中需要生成以下文件

1. linker scripts
2. device_gen.rs
3. config.rs

seL4 中之前使用 cpp 生成代码的工作暂时先不考虑做，而是使用 rust cfg 功能进行替代。

同样，之前一些通过 dtc 获取的信息，应该也不包含在其中。

## 2. 文件生成

### 2.1 linker scripts 生成

### 2.2 config.rs 生成

## 3. 配置项