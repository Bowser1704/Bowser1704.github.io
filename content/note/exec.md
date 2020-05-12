---
title: "return与exit的不同"
date: 2019-11-17T11:23:47+08:00
draft: false
categories:
- cs
tags:
- Linux
description: 由exit与return区别引发的思考
---

## 首先问题就是有时候我们返回用return，有时候是exit是什么区别呢？

### 首先明确函数执行

1. 我们直接运行某些脚本，或者是``sh x.sh`，这背后是先fork一个shell再执行x.sh。
2. source，就是直接在当前进程下执行一些文件，一般都是参数文件，不fork创建子进程。
3. exec， 这是一堆程序集合，The  exec()  family  of  functions replaces the current process image with a new process image. 意思就是我们使用==一个新的进程==来代替==原来的进程==，也就是看上去很美，是一个进程。
4. system， The  system()  library  function  uses  fork(2)  to  create  a child process that executes the shell command specified in command using execl(3) as follows:，就是说system = fork + exec + waitpid，就是可以这样来执行一个你想execute的程序。

### 有两种fork fork 与 vfork

- fork 创建子进程， 复制数据到子进程
- vfork创建子进程， 父子共享内存，也就是为了exec，不用copy，因为其实他不需要很多父进程的数据。

所以如果是vfork，你直接return的话，父子进程公用堆栈，那你就干掉了父进程，但是exit是从子进程退出。

