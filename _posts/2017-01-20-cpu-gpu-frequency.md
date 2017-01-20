---
layout: post
title: RK3399 定频操作
categories: [Rockchip]
description: 在做一些性能测试的时候可能需要做定频操作
keywords: Rockchip, RK3399, CPU, GPU
---

## CPU 定频

* 关闭温控  
`echo user_space >  /sys/class/thermal/thermal_zone0/policy`  

* 切换变频策略（A53是cpu0，A72是cpu4）  
`echo userspace > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`  
`echo userspace > /sys/devices/system/cpu/cpu4/cpufreq/scaling_governor`  

* 查看支持哪些频率（A53是cpu0，A72是cpu4）  
`cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies`  
`cat /sys/devices/system/cpu/cpu4/cpufreq/scaling_available_frequencies`  

* 定频（A53是cpu0，A72是cpu4）  
`echo 1512000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed`  
`echo 1992000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed`  

* 查看当前频率（A53是cpu0，A72是cpu4）  
`cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq`  
`cat /sys/devices/system/cpu/cpu4/cpufreq/cpuinfo_cur_freq`  

## GPU 定频

* 关闭温控  
`echo user_space >  /sys/class/thermal/thermal_zone1/policy`  

* 切换变频策略  
`echo userspace > /sys/class/devfreq/ff9a0000.gpu/governor`  

* 查看支持哪些频率  
`cat /sys/class/devfreq/ff9a0000.gpu/available_frequencies`  

* 定频  
`echo 800000000 > /sys/class/devfreq/ff9a0000.gpu/userspace/set_freq`  

* 查看当前频率  
`cat /sys/class/devfreq/ff9a0000.gpu/cur_freq`  
