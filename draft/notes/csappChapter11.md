---
title: "csapp Chapter11 网络编程"
date: 2019-11-13T21:30:14+08:00
draft: false
categories:
- csapp
tags:
- Network-Programming
description: csapp 第11章网络编程
---

主要记录一下socket相关内容，网络协议不多做描述。

下图是基本的步骤图

![](https://pic.superbed.cn/item/5dcd2bb58e0e2e3ee9164936.jpg)

## getaddrinfo

一句话：获取套接字地址，从url，也就是dns获取真正的ip地址，返回的结构中是一个链表，有多个sockaddr(也就是dns会给我们返回多个ip地址)，我们通过遍历来确定哪一个可用，如果都没有用，就error了。

注意点：

- sockaddr是socket地址结构，16个字节，前两个字节标明协议簇，后面的字节其实可以变动。

  因为IPV4与IPV6一个32位(4个字节)，一个128位(16个字节)，所以sockaddr装不下IPV6，就有sockaddr_in 16个字节，sockaddr_in6 28个字节。但是其实都是可以转化为sockaddr使用，也就是我们使用的就是sockaddr，为什么呢，因为我们每次传参传的是sockaddr指针，并且还有一个sockaddr_len，既然有了长度，结构体底层不一样也没事了，需要的是指针+长度，就可以得到完整数据。

  整理一下有几种sockaddr结构

   sockaddr            		16 bytes
   sockaddr_in        		16 bytes for IPv4
   sockaddr_in6        	28 bytes for IPv6
   sockaddr_storage    128 bytes for all protocol

- 返回的是addrinfo链表，里面除了sockaddr指针外，还存放了，数据协议，一些可选参数，balabala的。

- 利用它是直接返回addrinfo，我们在socket等函数中可以直接使用这些属性作为参数。

P.S.：csapp中有些函数讲的是全小写，使用时首字母却大写了，例如getaddrinfo，使用时却是Getaddrinfo，其实是做了一个封装。例如

```c
void Getaddrinfo(const char *node, const char *service, 
                 const struct addrinfo *hints, struct addrinfo **res)
{
    int rc;
	//加了一个错误处理，其他都差不多
    if ((rc = getaddrinfo(node, service, hints, res)) != 0) 
        gai_error(rc, "Getaddrinfo error");
}
```

## client创建

getaddrinfo -> socket -> connect ** open_clientfd

- getaddrinfo 不多做解释

- socket 得到一个clientfd，这个时候还没有网络什么事，仅仅是自己创建一个socket fd，内核会动态分配一个port。

- connect 这个时候开始有网络了，connect里面要填server的sockaddr，记住还有addrlen，这是sockaddr可以适用于不同协议的关键。

open_clientfd是用来替换上面几个步骤的。

## server创建

getaddrinfo -> socket  -> bind -> listen -> accept     

- socket+bind+listen 用来创建一个listenfd，** open_listenfd

  - socket 创建一个socket fd，暂时没什么用，参数有协议和一些可选参数。
  - bind 这个时候bind地址，也就是传入sockaddr和sockaddr_len，确定这个socket绑定什么socket地址，与client不同，前者是动态分配的。
  - listen 确认这个socket fd 是 listenfd 区分于 connfd(connecting fd)。

  open_listenfd就是把上面四个步骤整合了，但是还是要手写accept。

- accept 创建一个connfd，这是真正与client通信的socket，listenfd是用来监听的。

**注意一个问题，我们在getaddrinfo的时候hints.ai_flags设置了参数AI_PASSIVE，这是说我们host参数设置为0，也就是发送到本主机的所有ip地址我们都会接受。host值是通配符**

也就是说你舍友和你在同一个局域网内，是可以访问到你的。如果你设置了，也就是指定了ip，只有这个满足的ip才可以访问你，可以使用通配符。

这样的话你就可以使用局域网利用http协议和你室友传输文件了。

## HTTP

http请求注意格式，头部，实际内容。其实我们往socket里面写的就是字符串，但是是按照http的格式来写的，开头怎么样，头部怎么样，都是约定好的。所以我们解析http请求（注意uri是http请求内的一部分字段），做出不同的响应

### 静态内容响应

根据uri中的文件位置，我们返回文件。（如果是直接/，我们手动定位到home.html文件。），那么文件怎么返回/发送呢

```c
 /* Send response body to client */
    //打开文件，readonly
srcfd = Open(filename, O_RDONLY, 0);                       

//原做法
//虚拟内存内容相关，将文件/利用文件描述符 映射到一块虚拟空间，srcp。
srcp = Mmap(0, filesize, PROT_READ, MAP_PRIVATE, srcfd, 0); 
//关闭文件描述符号，已经有了虚拟内存空间上面的映射，不需要再read->write，防止内存泄露
Close(srcfd);                                               
//写入coonfd，send response body to client
Rio_writen(fd, srcp, filesize);                             
//释放虚拟内存中的空间，防止内存泄露
Munmap(srcp, filesize);

//作业11-9 做法
srcp = (char *) malloc(filesize * sizeof(char));
Rio_t rio;
Rio_readinitb(&rio, srcfd);
Rio_readnb(&rio, srcp, filesize);
//上面三步步骤可以用Rio_readn(srcfd, srcp, filesize)无缓冲函数; 直接代替.题目要求这种。
Rio_writen(fd, srcp, filesize);
free(srcp);
```

所以其实还是打开文件，写入另一个文件描述符，就是发送了。

**有一个问题：为什么不直接用srcfd，先读取srcfd，到buf，在写入coonfd中。**

作业11-9就让你修改这种方式，直接用malloc和rio_readn，rio_writen写入fd

**还记得我们上面说可以和室友传输文件吗？但是我们首先要实现支持MPG等视频文件的web**

作业11-7就是让我们手写支持MPG视频文件。

我们可以get_filetype加入一个mp4->viedo/mpge4，但是返回的直接是文件，还没有名字。

### 动态内容响应 cgi

parse_uri中会将uri中的参数，也就是调用的cgi程序的位置表明出来。

```c
 { /* Dynamic content */    
        //返回 '?'在uri中出现的index
        ptr = index(uri, '?'); 
        if (ptr)
        {
            strcpy(cgiargs, ptr + 1); //把？后面的参数copy给cgiargs
            *ptr = '\0';
        }

        //不存在?的情况
        else
            strcpy(cgiargs, ""); 
        //下面是把filename变成了.uri(./cgi-bin/adder)，也就是linux下的相对文件名字
        strcpy(filename, ".");   
        strcat(filename, uri);   
        printf("\ndynamic filename=%s uri=%s\n", filename, uri);
        return 0;
    }
```

所以我们在cgi中调用如下

```c
    if (Fork() == 0)
    { /* Child */ //line:netp:servedynamic:fork
        /* Real server would set all CGI vars here */
        setenv("QUERY_STRING", cgiargs, 1);                         
        Dup2(fd, STDOUT_FILENO); /* Redirect stdout to client */    
        Execve(filename, emptylist, environ); /* Run CGI program */ /
    }
    //等待子进程die，完成后函数才结束
    Wait(NULL); /* Parent waits for and reaps child */ 
```

**有一个问题，上面是直接wait，等待子进程死亡，在结束这个函数，联想一下，异常控制流那一章节，直接处理SIGCHLD信号，回收子进程的操作**

也就是作业11-8，怎么操作呢？未定，怎么解决呢？

```c
void ch_signal(int sig){
    pid_t pid;
    //子进程中都还没有终止，就立即返回0.否则返回终止子进程号，这样的话，在等待子进程的时候，我们还是可以做自己的事情的。
    while( (pid = waitpid(-1, NULL, WHOHANG)) > 0 )
        ;
}
//main函数中，加一个
signal(SIGCHILD, ch_signal);
```

## 注意几点

- 编译代码时，要加上-lpthread，gcc不再使用老的线程库pthread。

- CGI是标准

- http请求如何响应，我们都知道传输的是二进制序列（模拟信号->数字信号），拿过来之后也只能解析成字符串，那怎么分辩什么是什么呢？

  有两种方法，一种是就是每句话后面接上一个`\r\n`，标识这句话已经说完了，第二种是开头加上一个长度，标识后面这句话的长度是多少。http请求的头部是采用`\r\n`，而data是采用length的方法，所以你会看到http请求header里面会有一个Content-length字段。

  注意最后一个header会有两个“\r\n\r\n”，这是为了区分content和header。

- 我们将congtent-length设置为int，所以传输大文件会出问题。

可以去看我在tiny文件上做的笔记。[tiny.c](https://github.com/Bowser1704/csapp/blob/master/chapter11/tiny.c)
