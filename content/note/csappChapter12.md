---
title: "csapp Chapter12 并发编程"
date: 2019-11-14T23:43:14+08:00
draft: false
categories:
- cs
tags:
- csapp
description: 12章笔记，并发编程
---

## 基于进程的并发编程

就像我们最初写的echo服务一样，listen来了一个请求，创建一个connfd，一个子进程去处理，写入coonfd。如下

```c
listenfd = Open_listenfd(argv[1]);
while (1) {
    clientlen = sizeof(struct sockaddr_storage); 
    connfd = Accept(listenfd, (SA *) &clientaddr, &clientlen);
    //accept请求，创建connfd，准备写入
    if (Fork() == 0) { 
        //子进程的操作
        Close(listenfd); //子进程会继承父进程的所有数据，包括listenfd，但是我们用不上，并且我们要关闭它。但是不关闭，从没有内存泄露的角度来说还是正确的，理由如关闭connfd所示。
        echo(connfd);    //写入数据。
        Close(connfd);   //如果不手动关闭的话，进程退出时，也是会自动关闭的。
        exit(0);         
    }
    //必须关闭两次，文件表引用才会显示为0，才会释放资源。
    Close(connfd);  
}

```

## 基于IO多路复用的并发编程

### select函数

```c
//select函数原型
int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);
//三个fd_set分别为，read，write，except，select拿到了准备可读的描述符，就往下进去读。但是其实io操作还是同步的。
//timeout是一个结构体有两个属性 seconds and microseconds
//四个宏对fd_set操作，放入拿出...
void FD_CLR(int fd, fd_set *set);
int  FD_ISSET(int fd, fd_set *set);
void FD_SET(int fd, fd_set *set);
void FD_ZERO(fd_set *set);
```

调用select函数之后，会对传入的参数修改，包括三个fd_set，所以如果循环检查我们要每次重新初始化fd_set，如果超时，select返回0，三个fd_set会被清空。

但是有点限制：

> select() can monitor only file descriptors numbers that are less than FD_SETSIZE
>
> poll(2) does not have this limitation
>
> 一般最大限制FD_SETSIZE为1024，修改的话只能修改glibc的头文件，这就没必要了。

## 基于线程的并发编程

进程是内核调度的类似进程，是进程中的对资源的再一次细分，内核层面。

与进程的区别

- 线程，有自己的线程上下文，栈，程序计数器，通用目的寄存器，但是和其他对等线程公用进程的堆
- 进程，独自的上下文，堆栈，寄存器...

所以线程的切换，消耗会小很多，还有一点就是，并发编程需要大家 线程通信，所以怎么通信呢？

1. 共享变量

   我们知道，线程公用一个堆，并且虽然不同线程有各自的栈，但是如果对等线程拿到了，其他线程栈的栈指针也是可以访问的，所以我们也可以通过在主线程中申明了全局变量，再把指针给其他对等线程，也是可以访问的。

2. 信号量

   共享变量最大的问题是在于，竞争问题，如果有多个线程同时要操作同一个对象，这就很麻烦。

   所以有了信号量：

   > P(s)：如果s非零，就将s-1，如果s为零就阻塞，挂起线程，这里s-1操作是原子操作。如果V(s)执行之后，就恢复。
   >
   > V(s)：将s+1，如果有其他线程在P(s)操作中被挂起，这将会恢复那个线程。

   如果我们将s初始值设置为1，那么这就是一个锁，P为加锁，当锁是锁上的时候，只能解锁，不能加锁，所以就可以阻塞竞争操作。V为解锁，当锁一解开，加锁操作竞争，最后只有一个加锁操作会执行成功。但是会引起很多其他的问题，例如，饥饿(哲学家就餐问题)。

### 线程安全问题



## 基于协程的并发

协程是语言层面实现的，可以看做是小线程，不用陷入内核调度，因为陷入内核是十分耗费资源的，不同语言有不同的实现，但都是对应于线程，也就是多少个线程对应多少个协程。

### 用户线程：内核线程　＝　１：１

JAVA就是这样实现的，语言创建一个线程就会新起一个内核线程，这样就很好的可以直接管理内核线程，但是问题是，会创建过多的内核线程，使得线程创建和销毁很浪费，很多内核线程也会导致效率低下。

### 用户线程：内核线程　＝　Ｍ：Ｎ

GO采用的是这种模型，由语言自己来调度，这样会更简洁，也会使得效率更高，如果内核线程数量与CPU核数一样多的话，很完美的并发。

有Ｍ个内核线程，Ｎ个goroutine，有一个调度器来管理goroutine与内核线程的映射，不需要程序员去做什么，goroutine直接当函数用就可以，

### 用户线程：内核线程　＝　１：Ｎ

这种其实严格上来说，程序不能并行，没有用到多核CPU，js是使用这种方式的，但是node的性能还可以....，并且都是异步的，不懂。