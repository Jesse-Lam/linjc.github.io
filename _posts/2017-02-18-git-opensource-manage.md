---
layout: post
title: 代码管理之内部代码与外部代码
categories: [Work_Skills]
description: 本文讲如何管理公司内部代码与开源代码
keywords: Git
---

## 简介  
现在很多公司因为某些原因会开源自己的产品代码,虽说是开源,但并不会完全开源,一些核心代码还是会保留的,本文是作者在工作中的一些经验积累,可能并不是最好的方式.

## 分支介绍  
project: 公司内部分支,工程师协作工作的分支  
project_public_tmp: 临时分支,从内部分支到外部分支转换过程的过渡分支  
project_public: 外部分支,开放给用户的分支  

## 操作说明  
* project_public_tmp分支的创建:  
在project分支的基础上删除某些不想开放的源码或产品以外的冗余代码,添加不开放源码编译生成的库文件,保留commit信息.  
* project_public分支的创建:  
在project_public_tmp分支的基础上把commit信息删除.  
* 代码更新:  
代码首先在project分支上添加更新,同步时先合并到project_public_tmp: `git checkout project_public_tmp && git merge project`  
然后把project_public_tmp的修改添加到project_public,把commit信息去掉,这里用一个脚本实现:

```
#!/bin/sh

if [ $# -ne 3 ]; then
    echo -e "\033[31m Usage:\033[0m ./git-update-to-public.sh src_branch dst_branch commit_message"
else
    src_branch="$1"
    dst_branch="$2"
    commit_message="$3"
    commit_tree=$(git commit-tree $src_branch^{tree} -p $(cat .git/refs/heads/$dst_branch) -m "$commit_message")
    echo $commit_tree
    git update-ref refs/heads/$dst_branch $commit_tree
fi
```
* 推送代码:  
最后把project_public推送到github或者gitlab即可.
