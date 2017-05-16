---
layout: post
title: fixed-regulator的使用
categories: [Kernel]
description: 对于电压不可控的电源可使用fixed-regulator
keywords: Android, Kernel, regulator
---
## 简介
我们知道，在Linux系统中，DVFS(动态电压频率调整)对系统的功耗和稳定性有重大的意义。  
DVFS系统流程：  
1. 采集与系统负载有关的信号，计算当前的系统负载。  
2. 根据系统的当前负载，预测系统在下一时间段需要的性能。  
3. 将预测的性能转换成需要的频率，从而调整芯片的时钟设置。  
4. 根据新的频率计算相应的电压。通知电源管理模块调整给CPU的电压。  
由此，我们可以知道，DVFS是系统运行必不可少的，但有些用户可能因某些特殊原因，没有给系统提供调压的功能，即vdd_logic和vdd_arm不可控，或者是系统没有装pmu，正常情况系统会因为找不到mpu，无法获取到vdd_logic,vdd_arm，因此无法启动DVFS而无法开机，为解决这种情况，我们可以使用fixed-regulator。  
## 使用
Linux-kernel已经有fixed-regulator的驱动，其驱动文件在：`drivers/regulator/fixed.c`  
我们只要正确的在dts中定义好它就可以了。这里可以参考kernel文档`Documentation/devicetree/bindings/regulator/fixed-regulator.txt`：  
```
Example:

	abc: fixedregulator@0 {
		compatible = "regulator-fixed";
		regulator-name = "fixed-supply";
		regulator-min-microvolt = <1800000>;
		regulator-max-microvolt = <1800000>;
		gpio = <&gpio1 16 0>;
		startup-delay-us = <70000>;
		enable-active-high;
		regulator-boot-on;
		gpio-open-drain;
		vin-supply = <&parent_reg>;
	};
  ```
  compatible必须为“regulator-fixed”；  
  regulator-name对应vdd_logic或vdd_arm，这就相当于把某一路ldo定义为vdd_logic或vdd_arm；  
  由于这里是不可控的，所以regulator-min和regulator-max都是同样的值；  
  gpio可以定义为电源控制脚，如果没有可以不配上去。
