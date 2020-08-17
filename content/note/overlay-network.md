---
title: "初谈 SDN"
summary: ""
tags: [net]
categories: [cloud-native]
spotlight: true
date: 2020-08-11T23:37:44+08:00
draft: false
---

谈一谈容器网络和云上网络。

- Bridge
- VXLAN
- Docker default network
- Flannel

### 1. Bridge

Bridge 也就是网桥「也可以说是二层交换机/switch」，是一个二层设备，对受到的包，Forwarding 到目的端口，只涉及到 MAC 头的分析。

  >switch vs bridge 
  >
  >https://serverfault.com/questions/78184/whats-the-difference-between-a-bridge-and-a-switch
  >
  >>  The really cool thing about Linux is that it can be used to create both layer 2 and layer 3 switches and routers, allowing you to both bridge and route using Linux.

### 2. VXLAN

VXLAN (*Virtual eXtensible LAN*) 提供是一种网络虚拟化协议和方式，建立了一个 layer 2 network over layer 3。

### 3. Docker

容器网络，以 Docker 举例，在一台宿主机里面容器化出来不同容器之间的网络情况是什么样的呢？此时的容器有两种网络需求：

- 与外网通信「容器之外的所有网络，宿主机内网，公网....」。

- 与其他容器通信。

Docker 的方案是利用 Linux 的两个设备，Bridge 和 Veth Pair。

相当于用 Veth 虚拟了 n 个网卡，然后都插到一个叫做 Docker0 的 bridge 上，成功利用 bridge 连通为一个二层网络。

当一个 container-A / 10.42.1.34 像同一个 host 的 container-B / 10.42.1.35 通信的时候，经历下面几个步骤：

1. 查询路由表 route table

   ```bash
   ~ ❯ ip r
   default via 10.42.1.1 dev eth0
   10.42.0.0/16 via 10.42.1.1 dev eth0
   10.42.1.0/24 dev eth0 proto kernel scope link src 10.42.1.34
   ```

   根据 scope link 知道是二层互通的。

2. 直接查 arp table 封装成 frame，bridge 发挥作用转发到目的端口上。

当 container-A 向容器网络外通信时，下面几个步骤：

1. 查询路由表

   假设匹配到的是第一个路由，则下一跳是 10.42.1.1，使用 10.42.1.1 的 MAC address 封装成 frame。

   > 此时的 10.42.1.1 是 bridge 的地址，也可以说是 宿主机的地址。

2. frame 到达宿主机

   把 MAC header 拿掉之后，剩下的 ip 包继续走宿主机的网络栈。然后就是普通的网络通信了。

### 4. Flannel

flannel 提供一个 layer 3 network。

> Flannel is responsible for providing a layer 3 IPv4 network between multiple nodes in a cluster. Flannel does not control how containers are networked to the host, only how the traffic is transported between hosts. However, flannel does provide a CNI plugin for Kubernetes and a guidance on integrating with Docker.

假设 Flannel 使用的后端为 VXLAN (*Virtual eXtensible LAN*)  的话，VXLAN 提供二层网络，但是 Flannel 只提供三层网络。这因为 Flannel 对 VXLAN 的实现特殊，只是用了一些需要的 feature。

- Flannel 对于 ARP table 是在每个 node 加入的时候自动添加到每个其他节点的。并没有用到 ARP 查询 MAC 地址。
- 对于 Bridge 中的 FDB (Forwarding DataBase)，Flannel 并没有添加所有纪录，只是加入了不同 node 上的 flanneld。

当跨主机的 pod 通信时「不跨主机，就和 Docker 里面的容器一样」，下面几个步骤。

前面两步和 Docker 一样，直到到了宿主机网络栈之后，有一些不同的是下面的步骤：

1. 检查宿主机 route table

   ```bash
   [root@bowser ~]# ip r
   10.42.0.0/24 dev cni0 proto kernel scope link src 10.42.0.1
   10.42.1.0/24 via 10.42.1.0 dev flannel.1 onlink
   ```

   可以看到他的出口设备是 flannel.1，而 flannel.1 是一个 VTEP 设备，会把收到的数据包在用户态和内核态转换，转到用户态就是发送到 flanneld 进程，flanneld 对他进行封装处理，如果是 VXLAN 作为后端的话，怎么拿到 MAC 头的地址呢。

   ```bash
   [root@bowser ~]# ip n show dev flannel.1
   10.42.1.0 lladdr 42:61:6c:fd:6c:23 PERMANENT
   ```

   可以看到这个网段的 lladdr 都直接用同一个。

2. flanneld 封装完一个 VXLAN 数据需要知道目的 IP。

   这个目的 IP 的查询依靠于 flannel.1，flannel.1 还需要扮演一个 bridge 的角色，内部有一个 FDB ，根据已有的目的 VETP (Virtual Tunnel End Point) 设备的 MAC 地址和 FDB，我们可以查出来目的 IP。

   ```bash
   [root@bowser ~]# bridge fdb show dev flannel.1
   42:61:6c:fd:6c:23 dst 172.19.64. self permanent
   5a:56:5c:46:d4:55 dst 172.19.64.3 self permanent
   ```

3. 有了目的 IP，现在的数据包就像一个普通的 IP packet 发出去，经过之后所有的宿主网络栈。



