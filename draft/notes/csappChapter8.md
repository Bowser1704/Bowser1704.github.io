---
title: "Csapp Chapter08 异常控制流"
date: 2019-11-17T16:14:23+08:00
draft: true
categories:
- csapp
tags:
- exception
description: 异常控制流，主要是进程之间通信，管理。
---

## waitpid函数

当父进程fork一个子进程的时候，如果子进程结束了，但是父进程还没有回收他的话，子进程就成为了一个zombie，如果父进程结束了，子进程会被初始进程也许是init托管。但是有些父进程一直不结束，例如web程序，那么子进程就一直占据着资源，所以我们要选择的去回收子进程，利用waitpid。

csapp P517

```c
pid_t waitpid(pid_t pid, int *statloc, int options);
//wait也是waitpid
pid_t wait(int *statloc){return waitpid(-1, statloc, 0);}

/*参数
pid==-1 等待所有子进程
pid>0 等待指定pid的进程
pid==0 等待和调用进程，同一个GROUP的所有子进程。
pid<-1 等待GROUP等于pid绝对值的所有子进程。
options
1.WCONTINUED
2.WUNTRACED
3.WNOHANG  waitpid不阻塞
*/
```
