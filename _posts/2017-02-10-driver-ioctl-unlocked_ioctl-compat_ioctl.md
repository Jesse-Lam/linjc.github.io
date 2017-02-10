---
layout: post
title: ioctl,unlocked_ioctl,compat_ioctl的区别
categories: [Kernel]
description: ioctl,unlocked_ioctl,compat_ioctl的区别
keywords: Kernel,Driver
---
## 简介
ioctl的命令主要用于应用程序通过该命令操作具体的硬件设备,实现具体的操作,在驱动中主要是对命令进行解析,通过switch-case语句实现不同命令的控制,进而实现不同的硬件操作.
在内核版本2.6.36以后ioctl函数已经不再存在了,而是用unlocked_ioctl和compat_ioctl两个函数实现以前版本的ioctl函数,同时在参数方面也发生了一定程度的改变,去除了原来ioctl中的struct inode参数,同时改变了返回值.但是驱动设计过程中不存在很大的变化,同样在应用程序设计中我们还是采用ioctl实现访问,而并不是unlocked_ioctl函数,因此我们还可以称之为ioctl函数的实现.

## ioctl的定义

```
long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
```
