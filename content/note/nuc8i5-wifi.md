---
title: "豆子峡谷黑苹果 5GHz WI-FI 无法使用 80 MHz"
description: 豆子峡谷黑苹果网络问题。
tags: [WI-FI, nuc]
categories: [hardware]
spotlight: true
date: 2020-06-27T15:37:05+08:00
---

## nuc8i5bench hackintosh WIFI 跑不满速度

路由器：斐讯 K2

网卡：BCM94360CS

K2 理论上 5GHz 速度应该可以达到 867 Mbps 但是这里只有 180 Mbps。网卡和路由器都是支持 802.11ac 的。

![image-20200627202039062](https://tva1.sinaimg.cn/large/007S8ZIlgy1gg74esygwwj307t06y76w.jpg)

经过一些排查，发现 信道/Channel 虽然是 5GHz 但是只有 40 MHz，并且路由器设置也开了 80 MHz，做了一些尝试都没用。

------------

修改信道，首先检查一下网卡接受的信道，例如我的网卡。

![image-20200627202525283](https://tva1.sinaimg.cn/large/007S8ZIlgy1gg74exhdoqj30fl016weg.jpg)

修改为 40 依旧跑不满，是 200 多，依旧为 40MHz，信道修改为 149 一下子就到了 580，也成了 80MHz。

![image-20200627202921619](https://tva1.sinaimg.cn/large/007S8ZIlgy1gg74eqrhqcj307q06y75l.jpg)

有时候也能达到 867Mbps。

-----------------

所以暂时解决方案为 修改信道到 149。