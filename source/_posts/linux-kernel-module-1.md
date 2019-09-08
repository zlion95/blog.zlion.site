---
title: Linux Kernel Module 1
date: 2019-09-07 20:18:00
toc: true
categories: Technology
tags:
- linux kernel
description: 本文是linux内核模块开发系列的第一章，主要内容是说明如何编写一个linux内核模块，如何注册我们的内核模块，以及如何卸载一个内核模块。
---

# Linux内核开发系列 1

首先，这个博文系列主要是对**linux内核模块**开发过程中遇到的比较重要的东西进行总结，希望我能尽可能的总结清晰吧。因工作原因，文章基于的是**linux内核2.6.32版本[^1]**的，版本较老，如果你基于的是其他版本的内核，请仔细确认我所用的接口是否还能用，以免出错

在开始说明简单linux内核模块编写使用之前，我先申明一下，本人也是刚接触linux内核开发不久，因此在行文过程中也会有不少错误，或者是不合理的地方。如果你在使用文中代码过程中发现有严重的问题，希望能留言或发邮件给我指正，我会尽快更改

废话不多说了，让我们马上开始系列的第1章节，怎么编写和注册一个内核模块
[^1]: [Linux内核2.6.32版本](https://mirrors.edge.kernel.org/pub/linux/kernel/v2.6/linux-2.6.32.tar.gz)

## 为什么需要模块

按照国际惯例，要说一件事，我们需要先解释为什么需要解释这件事
首先Linux采用的是宏内核方式，代码量非常庞大，涉及组件非常多，如果我们添加一个新的功能时，都需要重新编译Linux内核，这将是非常耗时而且麻烦的事情。那如何才能使系统添加新功能时，不需要改动修改原来的内核，直接动态加载到内核中呢？

Linux自**1.2版本**开始就引入了LKM[^2]的设计思想，到2.6版本之后将原来编译的内核模块文件 .o 拓展为 .ko ("kernel object")，实现了模块与内核的分离，开发者编写的内核可以通过动态加载的方式装载入内核中执行

[^2]LKM: Loadable Kernel Modules 动态可加载内核模块

## 模块编写

内核模块的具体原理和实现这一章节先不做讨论，我们直接上一段代码来直观认识一下模块的大致组成

```c
/*
 * Copyright (C) 2010 Zhishi Zeng (zengzs1995@gmail.com)
 *
 * A simple kernel module to print hello.
 */
#include <linux/init.h>
#include <linux/module.h>

static char *user_name = "zlion";

static int __init sample_init(void)
{
    printk(KERN_INFO "Hello %s enter\n", user_name);
    return 0;
}

static void __exit sample_exit(void)
{
    printk(KERN_INFO "Hello %s exit\n", user_name);
}

module_init(sample_init);
module_exit(sample_exit);
MODULE_LICENSE("Dual MIT/GPL");
MODULE_AUTHOR("zlion <zengzs1995@gmail.com>");
MODULE_DESCRIPTION("A simple hello world module");
module_param(user_name, charp, S_IRUGO);
```

这是最简单的内核模块代码，我们先来总览一下整个的组成:
1. 模块加载函数
2. 模块卸载函数
3. 模块许可证
4. 模块参数，作者，信息等申明（可选）

### 模块加载函数

模块加载函数需要用`__init`来标识，并使用**module_init**宏来指定。模块加载函数的返回值为int类型，0代表模块加载成功，否则返回错误编码
在我们使用**insmod**或**modprobe**命令时，模块加载函数就会被调用

### 模块卸载函数

模块卸载函数需要用`__exit`来标识，并使用**module_exit**宏来指定。模块卸载函数没有返回值，在**rmmod**或**modprobe -r**时被调用

### 模块参数

