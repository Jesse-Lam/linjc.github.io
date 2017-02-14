---
layout: wiki
title: vim使用
categories: [Tools, vim]
description: vim是一个功能强大、高度可定制的文本编辑器.
keywords: Linux, Vim
---

## 安装
ubuntu默认安装了vi,想要用vim需要自己安装，指令：  
`sudo apt-get install vim`

## 配置文件.vimrc
请看:
[vimrc](https://github.com/linjc/linux/blob/master/profile/vimprofile/vimrc)

## 插件管理器vundle

### 下载  
`git clone https://github.com/gmarik/vundle.git ~/.vim/bundle/vundle`

### 配置
在~/.vimrc中添加：

```
"vundle
set rtp+=~/.vim/bundle/vundle/
call vundle#rc()
Bundle 'gmarik/vundle'
```

### 常用命令

```
:BundleList             -列举列表(也就是.vimrc)中配置的所有插件  
:BundleInstall          -安装列表中的全部插件  
:BundleInstall!         -更新列表中的全部插件  
:BundleSearch foo       -查找foo插件  
:BundleSearch! foo      -刷新foo插件缓存  
:BundleClean            -清除列表中没有的插件  
:BundleClean!           -清除列表中没有的插件
```


## 常用插件

### vim-airline
* 在vimrc中添加：

```
" vim-airline
Bundle 'bling/vim-airline'
set laststatus=2
let g:airline_powerline_fonts=1
```
* 添加字体：  
`sudo apt-get install fonts-powerline`  
* 在terminal设置中选择powerline字体  
* 效果如下：  
![vim-airline](https://linjc.github.io/images/wiki/vim-airline.png)

### ctags & Tagbar
* 安装 ctags  
`sudo apt-get install ctags`  

* 在vimrc中添加：

```
Bundle 'majutsushi/tagbar'
map tb :Tagbar<CR>                        "快捷键设置
let g:tagbar_ctags_bin='ctags'            "ctags程序的路径
let g:tagbar_width=30                     "窗口宽度的设置
```
* 生成tags文件  
`ctags -R`  

* 常用快捷键  

```
Ctrl＋］ 跳到当前光标下单词的标签
Ctrl＋O 返回上一个标签
Ctrl＋T 返回上一个标签
```
