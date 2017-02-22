---
layout: post
title: vim使用markdown插件
categories: [Vim]
description: vim编辑markdown文档时高亮及预览
keywords: Vim, Markdown
---

## 安装vim-markdown插件
* 如果使用Vundle管理插件:
在`~/.vimrc`中添加:

```
Plugin 'godlygeek/tabular'
Plugin 'plasticboy/vim-markdown'
```
在vim里面执行:

`:BundleInstall`
* 如果未安装插件管理器:

```
git clone https://github.com/plasticboy/vim-markdown.git
cd vim-markdown
sudo make install
vim-addon-manager install markdown
```

## vim自动识别markdown文件
在`~/.vimrc`中添加:

`autocmd BufNewFile,BufReadPost *.md set filetype=markdown`

## 安装vim-instant-markdown插件

* 安装新版的node.js:

```
sudo add-apt-repository ppa:chris-lea/node.js
sudo apt-get update
sudo apt-get install nodejs
```

* 安装instant-markdown-d

`sudo npm -g install instant-markdown-d`

* 安装vim-instant-markdown插件

在`~/.vimrc`里面添加:

`Plugin 'suan/vim-instant-markdown'`

在vim里面执行:

`:BundleInstall`

* 安装完成后，只要vim打开了markdown类型的文件就会自动打开一个浏览器窗口实时预览：
![vim-markdown](https://linjc.github.io/images/posts/vim-markdown-plugin.png)
