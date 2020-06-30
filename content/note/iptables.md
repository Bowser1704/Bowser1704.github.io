---
title: iptables 的一些学习总结。
summary: ""
tags: [Linux, iptables]
categories: [net]
spotlight: false
date: 2020-06-29T10:18:22+08:00
---

# iptables 的一些学习笔记。

netfilter 有 几个hook，分别有不同的功能。

- `PREROUTING`: Triggered by the `NF_IP_PRE_ROUTING` hook.
- `INPUT`: Triggered by the `NF_IP_LOCAL_IN` hook.
- `FORWARD`: Triggered by the `NF_IP_FORWARD` hook.
- `OUTPUT`: Triggered by the `NF_IP_LOCAL_OUT` hook.
- `POSTROUTING`: Triggered by the `NF_IP_POST_ROUTING` hook.

| Tables↓/Chains→               | PREROUTING | INPUT | FORWARD | OUTPUT | POSTROUTING |
| ----------------------------- | :--------: | :---: | :-----: | :----: | :---------: |
| (routing decision)            |            |       |         |   ✓    |             |
| **raw**                       |     ✓      |       |         |   ✓    |             |
| (connection tracking enabled) |     ✓      |       |         |   ✓    |             |
| **mangle**                    |     ✓      |   ✓   |    ✓    |   ✓    |      ✓      |
| **nat** (DNAT)                |     ✓      |       |         |   ✓    |             |
| (routing decision)            |     ✓      |       |         |   ✓    |             |
| **filter**                    |            |   ✓   |    ✓    |   ✓    |             |
| **security**                  |            |   ✓   |    ✓    |   ✓    |             |
| **nat** (SNAT)                |            |   ✓   |         |        |      ✓      |

![FW-IDS-iptables-Flowchart-v2019-04-30-1](https://tva1.sinaimg.cn/large/007S8ZIlgy1gg9iaj8p0nj30u0127hdt.jpg)

- **Incoming packets destined for the local system**: `PREROUTING` -> `INPUT`
- **Incoming packets destined to another host**: `PREROUTING` -> `FORWARD` -> `POSTROUTING`
- **Locally generated packets**: `OUTPUT` -> `POSTROUTING`

所以说，本机发送给 svc 的包，是通过 nat table OUTPUT chain 做了 DNAT 到 pod ip 再 通过 route 发送到 cni 插件/flannel 发送出去的。

------------

总之，学习 netfilter 首先要搞清楚每种数据包的踪迹，到底怎么走，经过了那些 hook。

------------------

参考文献

- https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture
- https://www.digitalocean.com/community/tutorials/how-to-choose-an-effective-firewall-policy-to-secure-your-servers
- https://www.karlrupp.net/en/computer/nat_tutorial