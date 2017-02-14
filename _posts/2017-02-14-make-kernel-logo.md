---
layout: post
title: 用脚本生成内核logo
categories: [Shell]
description: 把png图片转换为ppm格式的内核logo
keywords: Kernel, Shell, logo
---

## 使用Shell脚本把png转换为ppm

```
#!/bin/bash

destName=`echo $1 | sed s/.png//`
logo=_logo
logoName=$destName$logo
pngtopnm $destName.png > $destName.pnm && pnmquant 224 $destName.pnm > temp.pnm && pnmtoplainpnm temp.pnm > $logoName.ppm
rm -f $destName.pnm temp.pnm
```
