---
title: "Overlay Network"
summary: ""
tags: [“cloud-native"]
categories: [net]
spotlight: false
date: 2020-08-11T23:37:44+08:00
draft: true
---

VPC(VIrtual Pesonal Cloud) layer 2 over layer 3



layer 3 over layer 

flannel 提供 三层网络

> Flannel is responsible for providing a layer 3 IPv4 network between multiple nodes in a cluster. Flannel does not control how containers are networked to the host, only how the traffic is transported between hosts. However, flannel does provide a CNI plugin for Kubernetes and a guidance on integrating with Docker.

VXLAN 提供 二层网络 为什么不是直接就是 三层网络呢 FDB。

switch vs bridge 

https://serverfault.com/questions/78184/whats-the-differenc e-between-a-bridge-and-a-switch