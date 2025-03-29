---
author: "Bowser"
date: 2019-07-28
title: "git使用"
draft: true
tags: [
    "git"
]
categories: [
    "tools"
]
---

木犀后端星计划第三阶段 git解释

<!-- more -->

- 这里是木犀后端新计划的第三阶段，主要学习一个叫git的东西。

  git蛮重要，叫做分布式版本控制系统（Distributed Version Control System, 简称DVCS），简单说一下用途，我们在进行项目开发的时候，写的代码不是一个人写的，这个时候，怎么合并代码呢？有些人可能想到的就是复制粘贴，但是正规操作就要用到git，差不多这个意思，git是由大神linus(Linux之父)极其装逼的写出来的，如果想仔细了解看这里，git[简介](https://git-scm.com/book/zh/v2/%E8%B5%B7%E6%AD%A5-%E5%85%B3%E4%BA%8E%E7%89%88%E6%9C%AC%E6%8E%A7%E5%88%B6)

  github就是一个远程托管仓库，你写的代码存在他那里，给被人看很方便，自己也不怕掉，可能有一天你电脑坏了，代码可能就没了，但是有github不用怕，其实这不主要用github的原因，重点在于版本控制！

  版本控制，就是我们将每一次对文件的更改都作为一个版本，我们可以在任意版本切换，这是很重要的，因为在开发过程中，有bug会修改，但是改了很多地方才找到bug，这个时候你的源文件已经修改的了很多不必要的地方，这个时候利用版本退回，我可以丢弃那些没必要的东西，毕竟不必要了。

### 安装git以及创建github

- 安装git

  - 在windows上安装git

    在 Windows 上安装 Git 有几种安装方法。 官方版本可以在 Git 官方网站下载。 打开 http://git-scm.com/download/win，下载会自动开始。 要注意这是一个名为 Git for Windows的项目（也叫做 msysGit），和 Git 是分别独立的项目；更多信息请访问 http://msysgit.github.io/。

    另一个简单的方法是安装 GitHub for Windows。 该安装程序包含图形化和命令行版本的 Git。 它也能支持 Powershell，提供了稳定的凭证缓存和健全的 CRLF 设置。 稍后我们会对这方面有更多了解，现在只要一句话就够了。 你可以在 GitHub for Windows 网站下载，网址为 http://windows.github.com。

  - 在linux下安装

    ```bash
    sudo apt-get install git#Debian系下的系统,例如ubuntu
    #sudo是获得超级权限，类似于Windows的管理员。
    ```

- 创建github账号

  去github上创建一个账号，官网注册,https://github.com

###  起步，初次运行git前的配置

Git本身有一个git config工具来帮助我们设置不同目录,不同环境用不同的config,下面针对于linux系统

1. `/etc/gitconfig`文件,全局配置,如果使用`git config --system`,可以使用这个配置

2. `~/.gitignore`文件,或者`~/config/git/config`文件,只针对于当前用户,可以用`git config --global`使用这个配置

3. 当前`是仓库的目录下`有一个`.git/config`,就是当前仓库的配置

   

设置用户信息

   ```
   git config --global user.name "yourname"git config --global user.email xxx@example.com
   ```

 查看配置信息

   ```
   git config --list	#使用q退出进入的界面
   ```

学会使用git –help

输入git --help命令之后出现下面的

![](https://ccnupp.oss-cn-shanghai.aliyuncs.com/git.png)

### 一些简单基础的命令

```
git add [filename] #添加文件到索引
git commit -m  "对本次提交的说明"	#提交的本地的管理系统中
git push origin master #提交到例如github这种代码托管平台，master是分支名字，可以暂时不了解。
git status #显示工作区状态
```

### 第一个repository

- 在本地`git init`(初始化)一个repository

  ```
  git init 				
  #初始化一个仓库在当前目录
  git add *.c			
  #将文件添加到暂存区
  
  git commit -ｍ "xx"	
  #将暂存区的文件下提交#与远程仓库链接之后,有下面操作
  
  git push origin master			
  #将提交推送到远程仓库例如github,master为分支名
  ```

- `git clone` 一个仓库

  ```
  git clone https://github.com/Bowser1704/cpp-study.git	
  #本地名字也是cpp-study
  
  git clone https://github.com/Bowser1704/cpp-study.git my-cpp-study 
  #本地名字是my-cpp-study##此处用的是https协议,但是其实我们还会用ssh/git://协议,使用ssh协议可以让我们与远程仓库有一个关系的确定,关系的声明.
  
  git clone git@github.com:Bowser1704/cpp-study.git###最直观的表现就是不用每次push都输入账号和密码
  ```

### 实际操作

------

一般仅仅看可能学的效率不太好，布置几个小练习．

1. 新建一个文件夹muxi, 文件README.md, 推送到github上.
2. 在muxi文件夹下面新建一个文件muxi.md, 推送到github上.
3. 将muxi.md重命名为ccnu.txt, 推送到github上.
4. 将除READE.md之外的所有文件,文件夹全部删除,推送到github上.
5. 备注:理论上github上应该是有**四次commit**记录

### 参考文章

------

大家应该自己去看看这些文章

[git详细全面教程](https://git-scm.com/book/zh/v2)

[廖雪峰git教程](https://www.liaoxuefeng.com/wiki/896043488029600)