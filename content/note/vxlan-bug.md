---
title: "记录一次 vxlan Bug"
summary: "vxlan as the flannel backend"
tags: [k3s, flannel, vxlans]
categories: [cloud-native]
spotlight: false
date: 2020-06-27T13:21:45+08:00
---

## kubernetes network bug

使用 flanne 作为 cni 插件，并且使用 vxlan 作为后端会有 bug，表现为 node 不能访问 pod 被调度到其他 node 的 service，但是可以通过 pod real ip/ endpoint 访问。

>  Can't access pods by service cluster ip except you access from the node where pod in.

有几个 issue 与这个 bug 有关。

- k3s https://github.com/rancher/k3s/issues/1958
- k3s https://github.com/rancher/k3s/issues/1266
- flannel https://github.com/coreos/flannel/issues/1243
- kubernetes https://github.com/kubernetes/kubernetes/issues/87852

### 1. 更换 flanel 后端为 host-gw

> I changed the flannel backend from vxlan to host-gw, but it doesn't seem to work.

- method to change flannel backend.

```bash
vim /etc/systemd/system/k3s.service
```

- what I modified.

```bash
ExecStart=/usr/local/bin/k3s server --flannel-backend host-gw
```

- restart k3s

```bash
systemctl daemon-reload
systemctl restart k3s
```

- check my k3s net-conf

```bash
[root@bowser1704 ~]# cat /var/lib/rancher/k3s/agent/etc/flannel/net-conf.json
{
	"Network": "10.42.0.0/16",
	"Backend": {
	"Type": "host-gw"
}
}
```

```bash
[root@bowser1704 ~]# kubectl get svc -n food
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
redis          ClusterIP   10.43.105.118   <none>        7388/TCP   48d
food-backend   ClusterIP   10.43.105.114   <none>        8080/TCP   48d
[root@bowser1704 ~]# curl http://10.43.105.114:8080
^C
[root@bowser1704 ~]# kubectl get ep -n food
NAME           ENDPOINTS         AGE
food-backend   10.42.1.4:8080    48d
redis          10.42.0.10:6388   48d
[root@bowser1704 ~]# curl http://10.42.1.4:8080/sd/health
OK
```

所以还是没有找到最终解决方案。

### 2.

待续

