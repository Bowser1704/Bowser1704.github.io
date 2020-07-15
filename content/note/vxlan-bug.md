---
title: k8s vxlan 作为 cni 后端引发的 63 秒延迟
summary: "vxlan as the flannel backend"
tags: [k3s, flannel, vxlan]
categories: [cloud-native]
spotlight: true
date: 2020-06-27T13:21:45+08:00
---

使用 flanne 作为 cni 插件，并且使用 vxlan 作为后端会有 bug，表现为

1. 「 node 不能访问 pod 被调度到其他 node 的 service，但是可以通过 pod real IP/ endpoint 访问。」

2. 实际上是「node 通过 svc 访问 调度到其他  node 的 pod 会有延时， 63s」

> Flatcar / newer Linux kernels  延迟为 1 s

## 1. 遇到 bug：使用 cert-manager

首先是在使用 cert-manager 的时候

> cert-manager 的架构需要使用到一个 cert-manager-webhook 用来声明「创建一个 CR certificate」。但是如果 cert-manager-webhoook 被调度到所在的 node 之外的 node，就会出现 time-out。

```bash
Error from server (InternalError): error when creating "certificate.yaml": Internal error occurred: failed calling webhook "webhook.cert-manager.io": Post https://cert-manager-webhook.cert-manager.svc:443/mutate?timeout=30s: dial tcp 10.43.18.211:443: i/o timeout
```

如上，因为 cert-manager 设置了一个 timeout 所以结果肯定是失败的。

- 涉及 issue https://github.com/jetstack/cert-manager/issues/2811

## 2. 探寻原因

| 源             | 目标                    | 结果                | 组别  |
|:-------------:|:---------------------:|:-----------------:|:---:|
| nodeA         | service(pod in nodeA) | success           | 1   |
| nodeA         | service(pod in nodeB) | failure/delay 63s | 2   |
| pod(in nodeA) | service(pod in nodeB) | success           | 3   |

参考 [cert-manager#2811](https://github.com/jetstack/cert-manager/issues/2811)，也许修改 flannel backend to host-gw 可能有用。

> I changed the flannel backend from vxlan to host-gw, but it doesn't seem to work.

- 怎么修改 flannel backend

```bash
vim /etc/systemd/system/k3s.service
```

- 修改的部分
  
  ```bash
  ExecStart=/usr/local/bin/k3s server --flannel-backend host-gw
  ```

- restart k3s

```bash
systemctl daemon-reload
systemctl restart k3s
#如果没重启的话是不会的，所以要重启所有 node 「包括 agent」。
```

- check my k3s net-conf 这样只能看到 backed 的设置。

```bash
- 检查每个节点的 flannel backend type。

​```bash
[root@iZuf6dq9lezw045stckkhsZ ~]# kubectl get node bowser1704 -o yaml | grep backend-type
    flannel.alpha.coreos.com/backend-type: host-gw
```

~~所以对我来说 更换为 host-gw 似乎并没有作用。~~

因为使用的是阿里云的云企业网，并且使用不同账号 vpc，属于跨 vpc，所以 host-gw 不能使用。因为阿里云在中间做了一些拦截，把使用 node ip 作为路由下一跳的包全都拦了。

## 3. 着手 service：检查 iptables 规则

众所周知，service cluster IP 并不是真实 IP，而是通过 iptables 或者 ipvs 修改内核 netfilter 规则，做一些 [DNAT](https://en.wikipedia.org/wiki/Network_address_translation#DNAT)(Destination Network Address Translation)，将给定的协议 / 数据报转发到 endpoing 「实际上是 pod 的 real ip，建立在 overlay network 的」。

![FW-IDS-iptables-Flowchart-v2019-04-30-1](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggadttipz9j30u01270ya.jpg)

由上图可以知道数据包在 netfilter 内的走向。「需要有一定 iptables 知识」

并且我们知道，svc cluster ip 需要的是 DNAT，所以是 NAT table 中的 PREROUTING 和 OUTPUT chain，我们检查 PREROUING chain 和 OUTPUT chain。「在我们的场景下，packet 只会走 OUTPUT 和 POSTROUTING chain」

```bash
# check nat table PREROUTING chain
[root@bowser1704 be]# iptables -t nat -n -v -L PREROUTING/OUTPUT
Chain PREROUTING (policy ACCEPT 49 packets, 3398 bytes)
 pkts bytes target     prot opt in     out     source               destination
3193K  209M KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
1944K  142M CNI-HOSTPORT-DNAT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

# check KUBE-SERIVCES chain / food is my service name
[root@iZuf6dq9lezw045stckkhsZ be]# iptables -t nat -n -v -L KUBE-SERVICES | grep food
    2   120 KUBE-MARK-MASQ  tcp  --  *      *      !10.42.0.0/16         10.43.105.114        /* food/food-backend:http cluster IP */ tcp dpt:8080
    2   120 KUBE-SVC-J4YWF6HICEDZUWTC  tcp  --  *      *       0.0.0.0/0            10.43.105.114        /* food/food-backend:http cluster IP */ tcp dpt:8080
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.42.0.0/16         10.43.105.118        /* food/redis: cluster IP */ tcp dpt:7388
    0     0 KUBE-SVC-OXGTRCQ72XOGLTBD  tcp  --  *      *       0.0.0.0/0            10.43.105.118        /* food/redis: cluster IP */ tcp dpt:7388

# check food svc chain
[root@iZuf6dq9lezw045stckkhsZ be]# iptables -t nat -n -v -L  KUBE-SVC-J4YWF6HICEDZUWTC
Chain KUBE-SVC-J4YWF6HICEDZUWTC (1 references)
 pkts bytes target     prot opt in     out     source               destination
    2   120 KUBE-SEP-Z45YJMXQTGNBLAMQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0

# check kstack sep chain / sep is made for load balance
[root@iZuf6dq9lezw045stckkhsZ be]# iptables -t nat -n -v -L  KUBE-SEP-Z45YJMXQTGNBLAMQ
Chain KUBE-SEP-Z45YJMXQTGNBLAMQ (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.42.1.4            0.0.0.0/0
    2   120 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp to:10.42.1.4:8080

[root@bowser1704 be]# iptables -t nat -n -v -L KUBE-MARK-MASQ
Chain KUBE-MARK-MASQ (56 references)
 pkts bytes target     prot opt in     out     source               destination
   58  2400 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000
```

而 POSTROUTING 做了什么呢？

```bash
[root@iZuf6dq9lezw045stckkhsZ ~]# iptables -t nat -n -v -L POSTROUTING
Chain POSTROUTING (policy ACCEPT 5747 packets, 614K bytes)
 pkts bytes target     prot opt in     out     source               destination
4948K  362M CNI-HOSTPORT-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* CNI portfwd requiring masquerade */
4948K  362M KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
2346K  150M RETURN     all  --  *      *       10.42.0.0/16         10.42.0.0/16
 2608  198K MASQUERADE  all  --  *      *       10.42.0.0/16        !224.0.0.0/4
15770  690K RETURN     all  --  *      *      !10.42.0.0/16         10.42.0.0/24
    3   180 MASQUERADE  all  --  *      *      !10.42.0.0/16         10.42.0.0/16
[root@iZuf6dq9lezw045stckkhsZ ~]# iptables -t nat -n -v -L CNI-HOSTPORT-MASQ
Chain CNI-HOSTPORT-MASQ (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            mark match 0x2000/0x2000
[root@iZuf6dq9lezw045stckkhsZ ~]# iptables -t nat -n -v -L KUBE-POSTROUTING
Chain KUBE-POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination
  448 18044 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ mark match 0x4000/0x4000
```

也就是匹配加了 mark 0x4000/0x4000 的 packet，并做一次 SNAT。

kubu-proxy 用来实现 DNAT 和 SNAT 「k3s 二进制文件内置了，没有单独起一个进程。」

- 实现 SNAT 是利用 masquerading rules 「与 SNAT rules 有一些区别，SNAT rules 转换的 IP 是给定的，但是这种方式是 动态获取当前 IP 的。」
  
  实现 SNAT 只针对于，pod（in k3s it’s 10.42.0.0/16） 之外的网段访问 svc，具体过程是先用 KUBE-MARK-MASQ 加一个 mark，POSTROUING 阶段检查，如果有这个 mark 就做 SNAT。

- 实现 DNAT 是利用 DNAT rules。

--------------------------------

**根据第二步中的对照组，**

- 1 3 对照发现 node 和 pod in node 访问结果不一样，在这里区别是 node 做了 SNAT。所以 SNAT 可能有影响。

## 4. 查看 DNAT 之后的走向

```bash
[root@iZuf6dq9lezw045stckkhsZ ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.19.159.253  0.0.0.0         UG    0      0        0 eth0
10.42.0.0       0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.42.1.0       10.42.1.0       255.255.255.0   UG    0      0        0 flannel.1
10.42.2.0       10.42.2.0       255.255.255.0   UG    0      0        0 flannel.1
```

-------------------

**根据第二步中的对照组**

1 2 对照可以发现，1 和 2 结果不一样。在这里不一样的原因是 1 中直接走 cni 了，但是 2 中会走 flannel。所以走 flannel 可能有影响，并且根据别人使用 host-gw 会有帮助，也许 vxlan 有问题。

> cni0 是容器网桥，flannel.1 是 cni 插件 flannel 网桥，走 flannel 意味着会有 vxlan。

## 5. 追踪数据包 tcpdump 抓包

```bash
# 监听 pod ip / endpoing
tcpdump -i any -vv host 10.42.1.34

# 另一个 shell curl svc
curl 10.43.105.114:8080/sd/health

# P.S. 此时不能有其他人访问这个 pod。
```

发现有输出，说明监听到了，也就是 svc 转发到了对应的 pod。**并且等了一分钟左右他居然是能连通的**。

**初次之外，建立 TCP 连接时，一直重发 SYN，第七次才真正传送到了。**

明确两点：

- tcp 数据报是转发到了 pod IP 上的。
- 63s 的延迟之后又可以了。

![image-20200630235158877](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggar4e5zmqj30q304n0te.jpg)

在 tcp 三次握手时：

1. 客户端首先要发送一个 SYN 给服务端
2. 服务端再发送一个 SYN-ACK 回复客户端
3. 客户端最后发送一个 ACK 给服务端，握手就成功了。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggahwt6l8vj30go0960tb.jpg)

> 当连接未建立成功，重传需要 1s + 2s + 4s+ 8s+ 16s + 32s = 63s。并且第六次重传输会取消 checksum，也就是 'no cksum' 。

-------------------

根据 TCP 的原理以及前几步骤的确认，可以确认问题出现在 IP packet 进入 flannel 之后，并且做了 SNAT 以及 利用 VXLAN。

## 6. 追踪产生 bug 的原因

由第三步可以暂时认定是有 SNAT 的原因。并且做 SNAT 转出来还是自己的 IP。

关于内核 SNAT 的步骤，参考 [Linux NAT core](https://github.com/torvalds/linux/blob/24de3d377539e384621c5b8f8f8d8d01852dddc8/net/netfilter/nf_nat_core.c#L290-L301)。

1. 检查 source IP 是否为 IP pool 内的 IP，如果是的话直接返回。
2. 找到 IP pool 里面最少使用的 IP，将 IP packet 的 source IP 换成这个 IP。
3. 检查允许的端口，如果目前的端口，本来就空闲，就不变。之后再返回。
4. 找一个用于 SNAT 的端口 by calling [nf_nat_l4proto_unique_tuple()](https://github.com/torvalds/linux/blob/24de3d377539e384621c5b8f8f8d8d01852dddc8/net/netfilter/nf_nat_proto_common.c#L37-L85)。

iptables 中的 KUBE-MARK-MASQ 用作标记要不要 SNAT，POSTROUING 会做一次 SNAT，并且 VXLAN 封装之后还会做一次 SNAT。

- flannel issue
  
  [coreos/flannel#1282 (comment)](https://github.com/coreos/flannel/pull/1282#issuecomment-635639567)
  
  [coreos/flannel#1282 (comment)](https://github.com/coreos/flannel/pull/1282#issuecomment-635145841)
  
  [coreos/flannel#1282 (comment)](https://github.com/coreos/flannel/pull/1282#issuecomment-635145841)
  
  有人关于 --random-fully 参数做了详细的测试，这个参数是用于选择 SNAT 的两种算法。

大意是 iptables 和 kube-proxy 对 --random-fully 支持的问题。

- k8s issue https://github.com/kubernetes/kubernetes/issues/88986#issuecomment-640929804
  
  这个 issue 中的 comment 详细讲述的原因和如何去解决这个问题。

上面这些原因是引起使用 VXLAN/VETH 时 incorrect checksum 的原因，并且是一起触发的。

1. 由于 packet mark 的方式，当我们在第一次对 packet mark 0x4000/0x4000 并且进行 SNAT 之后，进入 VETH，经过 cni 插件出来之后，POSTROUTING 仍然会识别到这个包的 mark，并且进行二次 SNAT。

2. 并且因为 --random-fully SNAT 方式，source port 会改变，并且因为 kernel bug 不会进行第二次 check sum，所以最后的 checksum is incorrect，解决方案可以是禁止 double SNAT 所以不会有 bad checksum 的情况，或者是升级内核使得他会进行第二次 checksum。
   
   kernel checksum bug [参考链接](https://tech.vijayp.ca/linux-kernel-bug-delivers-corrupt-tcp-ip-data-to-mesos-kubernetes-docker-containers-4986f88f7a19)

----------------------------------

checksum-offload 指定了内核不做校验和，交给网卡 / 硬件去做，但是使用 VXLAN 时，由于上面某些原因校验值是错的。导致 checksum 错误，所以会丢弃，TCP 不得不重传，然后因为 5 次重传耗时 63s「第六次重传取消 checksum」，所以结果是 63s delay。

> https://github.com/projectcalico/calico/issues/3145
> 
> Linux’ TCP stack will sometimes/always attempt to send packets with an incorrect checksum, or that are far too large for the network link, with the result that the packet is rejected and TCP has to re-transmit. This slows down network throughput enormously.

如下图，本级 checksum 一直 incorrect。

![image-20200701010831331](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggatc17gefj31hc0i6n3x.jpg)

解决办法

1. 关闭 Checksum Offloading
   
   `ethtool -K flannel.1 tx-checksum-ip-generic off`

## 7. fix this bug

1. https://github.com/kubernetes/kubernetes/pull/92035#issuecomment-644502203
   
   pr 中详细讲述了触发这个 bug 的几要素。他的方法是使得走 flannel 后不会校验失败。

2. 关闭 Checksum Offloading
   
   因为是用的是 flannel 等虚拟网络硬件，做 checksum 等操作会 incorrect，所以关闭其实还是最省心的。

-------------

参考链接

- iptables https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture
- explaination for the same bug https://tech.xing.com/a-reason-for-unexplained-connection-timeouts-on-kubernetes-docker-abd041cf7e02
- linux code https://github.com/torvalds/linux/blob/24de3d377539e384621c5b8f8f8d8d01852dddc8/net/netfilter/nf_nat_core.c#L290-L291
- [weaveworks/weave#1255 (comment)](https://github.com/weaveworks/weave/issues/1255#issuecomment-221820171)
- VXLAN https://community.mellanox.com/s/article/vxlan-considerations-for-connectx-3-pro
- checksum-bug https://tech.vijayp.ca/linux-kernel-bug-delivers-corrupt-tcp-ip-data-to-mesos-kubernetes-docker-containers-4986f88f7a19
- https://www.dynatrace.com/news/blog/detecting-network-errors-impact-on-services/
