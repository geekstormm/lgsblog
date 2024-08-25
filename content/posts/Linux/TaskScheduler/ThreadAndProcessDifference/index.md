---
title: Index
subtitle:
date: 2024-08-17T13:32:20+08:00
slug: f1a5e95
draft: true
author:
  name:
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - draft
categories:
  - draft
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRss: false
hiddenFromRelated: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

<!--more-->

## 进程和线程

在网上大部分的介绍中, 总习惯于说明进程和线程的区别, 但是从内核的视角来看, 线程和进程的相同点远远大于不同点.
在内核的实现中, 将线程和进程统称为任务, 都是用一种结构体(task_struct)来表示, 在调度上没有任何区别, 上下文切换的开销也没太大区别.
线程的上下文切换的开销会比进程的上下文切换稍微小点, 能少点的原因是如果两个线程属于同一个进程, 那么切换的时候, 不需要切换进程的地址空间, 只需要切换线程的栈和寄存器.


### 进程和线程的相同点


### 进程和线程的区别