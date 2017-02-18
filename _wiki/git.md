---
layout: wiki
title: git 使用技巧
categories: [Tools, git]
description: git是一款免费、开源的分布式版本控制系统，用于敏捷高效地处理任何或小或大的项目。个人比较喜欢使用git，这里记录个人使用git常用的技巧．
keywords: Linux, git
---

## 获取源码

```
git clone git@168.168.0.253:rk3188/tvbox-v442/jb.git tvbox-v442 //把服务器仓库克隆到本地tvbox-v445目录
git clone https://github.com/rockchip-linux/u-boot -b release-20160816 //克隆服务器仓库下的release-20160816分支
git fetch rk tchip_lc_beex:tchip_lc_beex  //获取远程分支
git pull ruanjg@168.168.0.11:~/jppro/rk3288/custom/tvbox-442/.git master:temp //从另一个服务器或仓库拉取代码到本地作为一个新分支
```

## 操作分支

```
git branch -m e5 temp  //重命名分支
git branch -d temp  //删除分支
git push origin :temp   //删除远程分支(可能要强制推送，强制推送需要加一个参数: -f)
git pull origin master  //将远程服务器上的master分支合并到当前分支中
git diff master HEAD kernel/drivers/input/remotectl/rkxx_remotectl.c //比较master分支和当前分支rkxx_remotectl.c文件的区别
```

## stash暂存

```
git stash save "temp" //把修改暂存，取名为temp
git stash list //打印暂存列表
git stash show stash@{0} //打印暂存列表stash@{0}的修改
git stash pop stash@{0} //把暂存stash@{0}取出(原暂存会被清除)
git stash apply stash@{0} //把暂存stash@{0}取出(原暂存不会被清除)
git stash drop stash@{0} //把暂存stash@{0}删除
```

## patch 补丁

```
git format-patch HEAD^ //最近的1次commit生成patch
git format-patch HEAD^^ //最近的2次commit生成patch
git format-patch -M master //当前分支所有超前master的提交生成patch
git format-patch –n 07fe //--n指patch数，07fe对应提交的名称
git diff > my.patch //当前修改生成patch，可以这样打patch: patch -p1 < my.patch
git diff --no-prefix > my.patch //当前修改生成patch，可以这样打patch: patch -p0 < my.patch
git am 0001-Kernel-remotectl-add-RK_IR_NO_DEEP_SLEEP-config.patch //把patch直接生成一个提交(用于git format-patch生成的patch)
```

## 打包SDK

```
git archive --format=tar --prefix=Android-rk3066-tchip/ HEAD | gzip > Android-rk3066-tchip.tar.gz //打包整个SDK
git archive --format=tar HEAD kernel | gzip > firefly-kernel.tar.gz //打包kernel目录
git archive --format=tar HEAD | gzip > ubuntu-factory-fenghuolun.tar.gz
```

## 仓库操作

```
git remote rm origin //删除仓库路径
git remote add origin git@168.168.0.253:rk3288/tvbox-442.git //添加仓库路径
git remote add mipi linjz@168.168.0.10:project/3288/tvbox-51/.git //添加一个新仓库
git remote set-url origin firefly@168.168.0.13:project/rockchip/3288/src/firefly-44/kitchen/.git //修改仓库路径
```

## 用socks5代理

```
sslocal -s [ip] -p 443 -k [password]
http:
git config --local http.proxy 'socks5://127.0.0.1:1080'
https:
git config --local https.proxy 'socks5://127.0.0.1:1080'
ssh:
git config --local ssh.proxy 'socks5://127.0.0.1:1080'
```

## 其他

```
git commit --amend --author="linjc <service@t-firefly.com>" //修改已提交信息
```
