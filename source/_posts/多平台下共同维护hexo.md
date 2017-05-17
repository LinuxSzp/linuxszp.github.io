---
title: 多平台下共同维护hexo
date: 2017-05-17 19:26:13
categories: [教程]
tags: [hexo]
---

因为经常需要在不同的操作系统上工作，所以需要在每个平台下都可以方便的写博客。

三平台：Mac + Fedora21 + Ubuntu16.04

博客最初是在Mac下，所以需要在Fedora21和Ubuntu16.04下也配置一下，方便写博客和维护。

<!-- more -->

## **前提** ##

首先，你应是在某一个平台下已经可以使用hexo创建博客，并可以使用hexo d同步到github，如果这一步还没实现，请自行网上搜索**使用Github和Hexo搭建个人博客**相关的关键字，或参考[Fedora21-hexo-github-搭建个人博客](https://linuxszp.github.io/2016/08/24/Fedora21-hexo-github-%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/)

## **方法** ##

### 已经搭建好的Hexo环境下 ###

为了防止误操作，最好先备份一下

1. 删除博客根目录和主题目录下的.git文件夹

2. 创建.gitignory文件(如果博客根目录下没有的话)，并修改内容为
``` bash
.deploy_git
public
```
3. 将博客源码保存在github博客项目的一个分支上
在博客根目录下，依次执行以下命令
``` bash
$ git init
$ git add .
$ git remote add origin git@xxxxxxxxxx.git	//自己的github中的博客项目
$ git branch src	//创建一个分支
$ git checkout src	//选择分支
$ git push -u origin src
```

### 新环境下 ###

已经在Fedora和Ubuntu下或其他环境中安装了npm，node.js，hexo的条件下
1. 直接克隆github中的博客项目
``` bash
$ git clone git@xxxxxxxxx.git
$ git checkout src	//切换到src分支下
```

## **日常使用** ##

1. 不论在那个平台下，都要切换到src分支上操作

// 在Mac下
1. 在src分支上正常使用hexo，当使用hexo new创建一个博客，并使用hexo clean，hexo g，hexo d同步到github(master分支)
2. 使用
``` bash 
$ git add .
$ git commit -m "src"
$ git push origin src
```
将更改提交到github上的src分支

// 在Fedora下
1. 在其他平台使用hexo更新了博客，并提交到了github的src分支，本平台下的本地src分支下的内容与github远端src分支的内容不一致，在本地src分支下使用
``` bash
$ git pull
```
与远端src分支进行同步

然后，重复在Mac下的1，2步骤
