---
title: "csapp Chapter10 UnixIO"
date: 2019-11-12T22:05:45+08:00
draft: false
categories:
- cs
tags:
- csapp
description: csapp第十章的笔记
---

## 基本概念

首先要有一个思维，Linux上面基本上所有外部设备都是文件，都是通过IO来读取，写入的，包括键盘，网络，终端.....

P.S.：为什么终端是呢，想象一下以前大型计算机，只有一台主机，不同人使用，就是有多个终端(物理设备)来操作系统，只不过现在我们用的微机就是一台主机了。

![上一张图](https://pic.superbed.cn/item/5dcabfbd8e0e2e3ee9b257f5.jpg)

看这张图片，可以看到：

- 外设都是走I/O总线，从而与系统交互。

### 那怎么使用文件呢

一句话：系统打开文件时会生成两个结构一个是在系统表上，一个是在v-node表上。前者存储的是某个进程对文件读取的偏移量，后者储存的是文件的固有属性，文件大小，文件权限。

- 文件读取的偏移量：你打开一个文件会进行读取，不可能一次读完，也不可能每次读都从头开始，而是记录你上次读到哪里了，这就是偏移量。

对于某个进程来说：一个进程会打开多个文件，怎么管理呢？有一张表，叫做描述符表，每个打开的文件都有一个ID，也叫作描述符，你打开文件后，描述符就代表你的文件了。看下面的图

![](https://pic.superbed.cn/item/5dcac3bc8e0e2e3ee9b6fc15.png)

- 子进程继承父进程的文件，有自己的描述符，但是指向相同的文件表。
- 一个进程打开两次同一个文件，会有两个描述符，但是同一个文件表结构。
- 文件表结构只有refcnt=0才会关闭，也就是引用他的文件描述符为0，才会关闭。

P.S.：每个进程会默认打开三个文件，有三个描述符 0=标准输入(Stdin)，1=标准输出(Stdout)，2=标准错误(Stderr)

| 整数值 |                          名称                           | unistd.h符号常量[[1\]](https://zh.wikipedia.org/wiki/文件描述符#cite_note-1) | stdio.h文件流[[2\]](https://zh.wikipedia.org/wiki/文件描述符#cite_note-2) |
| :----: | :-----------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|   0    |  [Standard input](https://zh.wikipedia.org/wiki/Stdin)  |                         STDIN_FILENO                         |                            stdin                             |
|   1    | [Standard output](https://zh.wikipedia.org/wiki/Stdout) |                        STDOUT_FILENO                         |                            stdout                            |
|   2    | [Standard error](https://zh.wikipedia.org/wiki/Stderr)  |                        STDERR_FILENO                         |                            stderr                            |

## Unix IO

打开文件用到了open函数返回的是文件描述符，注意两个参数

```c
int open(char *filename, int flags, mode_t mode);
//flags是进程如何访问这个文件，只读还是只写，或者不存在时新建
//mode是创建文件时指定权限，就是正常的文件权限，读写执行，这里有一个umask，
//每个进程都有一个umask，通过umask函数执行，真正的mode为 mode & ~umask
//出错返回-1 否则返回文件描述符
ssize_t read(int fd, void *buf, size_t n);
ssize_t write(int fd, const void *buf, size_t n);
```

读和写文件，Unix IO读写文件都是无缓冲的，也就是说每次读文件，都要陷入内核态，按照指定的字节大小读取放到给定的字符指针内，如果遇到EOF就返回0，否则返回读取字节大小，出错返回-1。

- ssize_t与size_t，前者是signed，因为可能返回-1，后者是unsigned，读取字节数>=0。
- EOF，指的是当当前文件位置指向文件的最后，也就是文件偏移量等于文件长度时候，引发的end-of-file。并不实际存在这个东西。

## RIO与标准库IO

这些函数有些事要是有缓冲区的，怎么实现的呢，就是新建一个缓冲结构，每次调用Unix IO时尽可能的读取足够多的字节(等于缓冲区大小)，陷入内核态费时间，这样的话，如果下次再读，直接去缓冲区拿(空间换时间)。

标准库io也是格式化输入输出，并且也有缓冲区，但是也有很多问题，不适合在网络编程上使用，

这里就是一些函数，去看书，看具体实现。

**要理解我们read文件的时候有才需要有缓冲，write文件的时候缓冲意义就不大了。**

1. rio read函数分为两批，参数不同，rio_t作为参数要先rio_init。R大写只不过是封装函数。

   ```c
   //用rio_t作为参数的，有缓冲的。
   rio_readinitb(rio_t *rp, int fd);
   rio_read();
   rio_readnb();//与下面的readline函数可以混合使用
   rio_readlineb();
   //用fd作为参数的，无缓冲的。
   rio_readn();
   ```

   write函数都只需要fd。

2. 写入一个文件描述符的时候

   其实writen和write都是差不多的，只不过writen会检查是否全部write进去了，因为在网络传输，等情况下的时候，可能不会安装设置n的大小写入n字节，所以writen就是加入了一个循环，如果每个字节都读进去，才会完成。

3. 我们文件都会有一些元数据(metadata)，例如创建时间，文件类型balabala的，有两个函数可一度去这些数据。

   ```c
   int stat(const char* filename, struct stat *buf);
   int fstat(int fd, struct stat *buf);
   ```

## 重定向

Linux shell中的 `>`

```sh
$bowser> ls > foo.txt
```

dup2函数，cgi编程会利用此函数，将std_out重定向到client_sockfd

```c
int dup2(int oldfd, int newfd);
//如果newfd已经打开，会先关闭它。
//一句话：把newtd指向oldfd的文件表项
dup(5, 0);
//标准输入重定向到描述符5指向的文件表
//会继承5打开文件的文件偏移量。因为指向的是同一个文件表。
```

## 疑问

明确几点：

1. 带缓冲函数，就是新建立一个结构，储存了fd(文件描述符)，还有缓冲的字节数组，等等东西，代替fd来使用。
2. 文件偏移量，也就是移动位置是放在文件表结构上的，这里有一个问题，似乎不能多个进程同时打开一个文件。
3. 文件元数据(metadata)，文件大小，文件类型放在v-node表上。
   - 文件类型，文件中有一个type结构指定，例如文本文件，二进制文件，sockets文件。
4. 标准库IO是设计为读取文本文件的，所以其他类型文件不合适。
