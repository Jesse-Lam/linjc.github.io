---
layout: post
title: android 设备结点属性问题
categories: [Android]
description: 可以用该文章描述方法改变设备结点属性
keywords: Android, Kernel
---
## 简介
在嵌入式驱动的调试过程中,如果内核添加了某个设备驱动,我们会在内核中添加一个设备结点,Android上层通过这个结点访问该设备.这时经常会出现设备结点的访问权限问题,可用以下方法解决

## 在内核中修改设备结点属性
### DEVICE_ATTR的使用
使用DEVICE_ATTR，可以在sys_fs中添加“文件”，通过修改该文件内容,可以实现在运行过程中动态控制device的目的.
查看DEVICE_ATTR的定义:kernel/include/linux/device.h

```
#define DEVICE_ATTR(_name, _mode, _show, _store) \
    struct device_attribute dev_attr_##_name = __ATTR(_name, _mode, _show, _store)
```
_name：名称，也就是将在sys_fs中生成的文件名称.
_mode：上述文件的访问权限，与普通文件相同，UGO的格式,如0664.
_show：显示函数，cat该文件时，此函数被调用.
_store：写函数，echo内容到该文件时，此函数被调用.

与DEVICE_ATTR类似的还有: DRIVER_ATTR，BUS_ATTR，CLASS_ATTR

## 在Android上层修改设备结点属性
可以在系统启动的时候修改设备结点的属性,即在device/rockchip/rk3399/init.rc文件中添加:

```
on boot
......

# firefly leds
chmod 0666 /sys/class/leds/firefly\:yellow\:user/brightness
chmod 0666 /sys/class/leds/firefly\:blue\:power/brightness
```
