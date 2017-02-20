---
layout: post
title: Kindle Paperwhite3 越狱安装Koreader
categories: [Kindle]
description: 本文讲如何Kindle Paperwhite3 如何越狱安装Koreader
keywords: Kindle, Koreader
---

## 简介  
本教程基于Kindle Paperwhite3, 在尝试的过程中,软件版本已降为5.7.4,降级固件使用[Kindle 特制固件](https://kindlefere.com/post/409.html)  
## 越狱  

1. 准备文件  
[main-htmlviewer.tar.gz](http://www.mediafire.com/file/2n3czup7usb2j03/main-htmlviewer.tar.gz)  
[JailBreak-1.14.N-FW-5.x-hotfix.zip](http://www.mediafire.com/file/95o9e2tf8hucbe2/JailBreak-1.14.N-FW-5.x-hotfix.zip)  
2. 让设备进入飞行模式  
3. 设备连接电脑,在电脑可以看到kindle盘  
4. 把下载的main-htmlviewer.tar.gz放到kindle的根目录  
5. 把kindle从电脑中安全弹出  
6. 在kindle的主菜单中点“搜索”,然后输入“;installHtml”  
7. kindle自动重启,可进行下一步  
8. 设备重新连接电脑  
9. 把下载下来的JailBreak-1.14.N-FW-5.x-hotfix.zip解包,把Update_jailbreak_hotfix_1.14.N_install.bin放到kindle根目录  
10. 把kindle从电脑中安全弹出  
11. 点 “菜单” 再选择 “设置” –> “菜单” 再选择  –> “更新设备“  
12. kindle自动重启,如果成功可以看到屏幕显示更新成功,还可以看设备上看到一个文件”You are Jailbroken”  

注:该方法参考: [www.ereader-palace.com](https://www.ereader-palace.com/how-to-jailbreak-kindle-paperwhite-23-and-kindle-voyage-2016-method/)  

## 安装插件程序启动器KUAL  

1. 下载 [KUAL-v2.7.zip](http://www.mediafire.com/file/xhknag24bxcxped/KUAL-v2.7.zip)  
2. 设备连接电脑  
3. 解压KUAL-v2.7.zip  
4. 把KUAL-KDK-2.0.azw2放到kindle的documents目录(版本在5.0以上)  
5. 把kindle从电脑中安全弹出,可以看到Kindle Launcher  

## 安装插件安装器MRPI  

1. 下载 [kual-mrinstaller-1.6.N.zip](http://www.mediafire.com/file/avll9f7pgkxf3hx/kual-mrinstaller-1.6.N.zip)  
2. 设备连接电脑  
3. 解压kual-mrinstaller-1.6.N-r13408.tar.xz  
4. 把文件夹内的 extensions 和 mrpackages 拷贝到 kindle 根目录下(如果根目录已有 extensions 这个文件夹，可以只把解压得到的 extensions 文件夹中的内容拷贝到 Kindle 根目录原有的 extensions 文件夹内)  
5. 把kindle从电脑中安全弹出  
6. 进入设备的Kindle Launcher,可以看到”helper”,点进去可以看到”Install MR Packages”  

## 安装Koreader  
1. 下载 [kindle-kpvbooklet-0.6.4.zip](http://www.mediafire.com/file/4ku35zn9rsib4ya/kindle-kpvbooklet-0.6.4.zip), [koreader-kindle-arm-linux-gnueabi-v2015.11-stable.zip](http://www.mediafire.com/file/0k4db43rewyvlqy/koreader-kindle-arm-linux-gnueabi-v2015.11-stable.zip)  
2. 设备连接电脑  
3. 解压kindle-kpvbooklet-0.6.4.zip  
4. 把update_kpvbooklet_0.6.4_install.bin放到kindle的mrpackages目录下  
5. 把kindle从电脑中安全弹出  
6. 进入设备的Kindle Launcher,点 “Helper” –> “Install MR Packages”  
7. Kpvbooklet被安装,设备自动重启  
8. 设备再连接电脑  
9. 解压koreader-kindle-arm-linux-gnueabi-v2015.11-696-g2bc7b01.zip  
10. 把解压得到的extensions里面的内容放到kindle的extensions,把解压得到的koreader放到kindle的根目录.  
11. 安装完毕,通过Kindle Launcher可以使用koreader  

注:以上参考[kindlefere.com](https://kindlefere.com/post/311.html)
