---
author: "Bowser"
date: 2019-07-28
title: "记录一次重装系统"
draft: yes
tags: [
   "Linux"
]
categories: [
   "Linux"
]
---

作为一个手贱的人，总是乱动系统，这次重装系统有几点收获

<!--more-->

### 1. UbuntuInstallScript

写了一个脚本，能够安装一些软件之类的，简单配置。

[UbuntuInstallScript-github](https://github.com/Bowser1704/Markdown/tree/master/linux/UbuntuInstallSricpt)

### 2. Hexo写github托管的blog

Hexo是一个blog框架，基本上就是给一个架子，什么都帮你写好了，连部署都直接配置好的，可以直接链接到github.io，真要感叹封装的强大。

不懂hexo，可以去看ZMC的[hexo’s blog](https://shadowmaple.github.io/2019/04/16/Hexo搭建github博客/)

讲一下链接github.io的概念，hexo是一个前端框架，前端仓库工程文件每次编译运行之后，生成css，js，html文件，然后我们选择推送到github上，利用github自带的page info服务，直接生成博客。

**所以这里就有两个repo，一个是<yourname.github.io>，另一个是hexo工程文件**

那么我们如何将工程文件保存下来呢？

1. 建立两个repo一个用来放blog，一个用来放hexo工程文件。
2. 利用git的branch，master存放blog文件，hexo放hexo工程文件。

第一种方法没什么好讲的，最简单的想法，讲一下第二种

#### 利用分支保存blog

- 第一次创建blog

  1. ```bash
     #确保安装了hexo
     mkdir blog	#建立一个空floder
     cd blog
     hexo init	#初始化hexo, 必须要空的floder才可以hexo init`
     npm install
     npm install hexo-deployer-git	#推送插件
     ```

  2. ```bash
     #建立new branch
     #建立github上仓库，例如Bowser1704.github.io
     #创建新分支hexo，并且将hexo设为默认分支	PS：在github->repo->setting里面
     git clone ...	#到本地
     git checkout hexo
     #现在github.io已经是hexo分支
     cp -r blog/* Bowser1704.github.io	
     #要删除几个文件，.deploy_git，theme下面的.git，因为github不允许.git循环嵌套
     ```

     文件差不多就这些，**注意配置hexo时候选择master分支**

     ```bash
     .DS_Store
     Thumbs.db
     db.json
     *.log
     node_modules/
     public/
     .deploy*/
     ```

     

  3. ```bash
     #现在就是两个branch
     #hexo用来写blog，每次写完的时候，可以push上去，保存下来
     #github.io在你每次执行hexo d 命令的时候，会自动更新
     hexo d
     ```

- 后面使用

  ```bash
  #直接git clone仓库下来。
  #不需要hexo init
  #已经是hexo仓库了
  npm install
  npm install hexo-deployer-git --save
  hexo g
  hexo s
  #OK
  ```

### 3. 关于系统备份问题

Jzc [文章](https://www.jianshu.com/p/7821027cc455)

