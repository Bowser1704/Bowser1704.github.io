---
title: "利用systemd管理nginx.service"
date: 2019-11-01T14:50:23+08:00
draft: false
spotlight: true
categories:
- cs
tags:
- tools
description: 如何将一个nginx.service用systemctl托管
---

## systemd是什么

以前的Linux系统启动，首先会启动一个init.d，然后其他所有的进程都是这个进程的子进程，但是现在的改进，启用systemd来启动管理daemon进程，不过据说有一堆的bug、

systemd具有一系列的命令，但是我只讲systemctl。

```shell
systemctl start nginx.service
systemctl stop nginx.service
systemctl restart nginx.service
```

字面意思。

## 如何让systemd托管我们的service

linux下一切皆文件，systemd也是靠读取一些指定的文件来管理的，我的系统service文件在`lib/systemd/system`下面，你可以自己去找一下，locate命令就可以。

而service长什么样子呢？以nginx.service为例子

```bash
[Unit]
Description=nginx - high performance web server.
Documentation=http://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target
# 这里是一些描述信息，不是特别重要
[Service]
Type=forking
PIDFile=/run/nginx.pid	#pid文件的位置
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf #启动前的命令一般都是test，其实就是nginx的命令
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf	#启动命令，使用绝对路劲。
ExecReload=/usr/local/nginx/sbin/nginx -s reopen
ExecStop=/usr/local/nginx/sbin/nginx -s stop
PrivateTmp=true
  
[Install]
WantedBy=multi-user.target
# 放入multi-user.target目录下面
```

## nginx配置

我们在安装nginx的时候，如果是默认包管理器安装的话，会自动加入systemd，但是我们自己安装就会有一堆问题。所以conf配置添加一句。

```shell
pid /run/nginx.pid
# pid /var/run/nginx.pid
# /var/run其实是一个指向/run的软连接。
```

### 可能会遇到的问题

1. nginx 找不到nginx.pid，那你就自己去创建一个nginx.pid。

