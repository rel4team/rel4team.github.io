---
weight: 4
bookCollapseSection: false
title: Github 管理项目
commentsId: 1
---

# 使用 Github Projects 和 Milestones

## TL;DR

Github Projects 更适合进行一个整体项目进度的管理和监控，Github Milestone 与一个仓库的绑定更加紧密，更适合用来管理版本和阶段性任务。

有一个简单的使用介绍：

- <https://github.com/FZUSESPR21/Class_Resources/discussions/11>

## Github Projects 介绍

GitHub Projects 是一个更通用的、看板风格的项目管理工具，用于可视化追踪 issues、PR 和任务的进度。Github Projects 可以包含多个 repo。

### 适用场景

- 一个长期开发项目的管理
- 多个仓库协同开发
- 对团队整体的开发进度进行追踪

### 核心功能

Github Project 包含看板，road map 等功能，通过 Github Project Workflow 可以自动关联 issue 和 PR.

### 相关文档
- <https://docs.github.com/zh/issues/planning-and-tracking-with-projects/creating-projects/creating-a-project>
- <https://docs.pingcode.com/ask/1188871.html>

## Github Milestones 介绍

Milestone 是某个仓库内用来分组 issue 和 PR 的机制，代表一个阶段性的目标（比如某个版本）。Github Milestones 仅仅只针对一个仓库。

Milestone 可以设置一个截止日期，将选中的和符合规则的 issue 包含其中，可以展示在这个阶段完成/未完成多少个 issue/PR。以百分比的形式展示整体的进度。

### 适用场景

- 在发布一个特定的版本之前进行归类
- 查看完成一个阶段的任务前需要完成哪些任务
- 管理 issues

### 相关文档

- <https://docs.github.com/zh/issues/using-labels-and-milestones-to-track-work/about-milestones>