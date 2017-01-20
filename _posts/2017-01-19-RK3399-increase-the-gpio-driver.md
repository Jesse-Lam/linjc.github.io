---
layout: post
title: RK3399 增强GPIO的驱动能力
categories: [Rockchip, Kernel]
description: 在调试Firefly-rk3399开发板的时候配合硬件所做的修改笔记．
keywords: Kernel, Pinctrl, GPIO
---

## 原理
在模块驱动调试的时候会出现电源驱动力不足而导致模块不工作或者速度提不上来的情况，这种情况可以通过修改硬件，用驱动力更强的电源通道或者调整电压，但是最方便的就是通过软件的方式修改相关引脚的驱动能力．

## 步骤

### 确定需要增强驱动能力的模块及引脚
比如：需要增强wifi模块clk引脚的驱动能力，从原理图可以看到：
![rockchip gpio driver](https://linjc.github.io/images/posts/rockchip/rk3399_gpio_driver.jpg)  
该引脚是属于SDIO模块里面的，接到主控的GPIO2_D1

### 找到对应的DTS文件
这是属于比较核心的配置，所以我们优先查找rk3399.dtsi，可以找到：

```
sdio0 {
    ......
    sdio0_clk: sdio0-clk {
        rockchip,pins =
            <2 24 RK_FUNC_1 &pcfg_pull_up>;
    };
    ......
};
```
这里“2 24”就是代表引脚GPIO2_D1，“pcfg_pull_up”代表该引脚为上拉脚，驱动能力为默认，我们要增强它的驱动能力，就在这里配置。

### 确定想要设定的驱动级别
在rk3399.dtsi文件中我们可以看到一些常用的定义：

```
pinctrl: pinctrl {
......
    pcfg_pull_up_20ma: pcfg-pull-up-20ma {
        bias-pull-up;
        drive-strength = <20>;
    };
    pcfg_pull_none_20ma: pcfg-pull-none-20ma {
        bias-disable;
        drive-strength = <20>;
    };
    pcfg_pull_none_18ma: pcfg-pull-none-18ma {
        bias-disable;
        drive-strength = <18>;
    };
    pcfg_pull_none_12ma: pcfg-pull-none-12ma {
        bias-disable;
        drive-strength = <12>;
    };
    pcfg_pull_up_8ma: pcfg-pull-up-8ma {
        bias-pull-up;
        drive-strength = <8>;
    };
    pcfg_pull_down_4ma: pcfg-pull-down-4ma {
        bias-pull-down;
        drive-strength = <4>;
    };
......
};
```
从这里我们可以看到SDK已经默认设置了几个级别的驱动能力，可以从中选择一个，至于是否可以自己添加，这个还没测试，修改之后如下：

```
sdio0 {
......
    sdio0_clk: sdio0-clk {
        rockchip,pins =
            <2 24 RK_FUNC_1 & pcfg_pull_up_20ma >;
    };
......
};
```

### 编译,烧写
修改完编译内核，然后烧写resource.img即可．
