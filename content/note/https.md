---
title: "Https"
summary: ""
tags: [https]
categories: [net]
spotlight: false
date: 2020-11-08T10:33:34+08:00
Draft: true
---

私钥签名，公钥加密。

1. 两对密钥

   互相发送信息时使用对方的公钥，这样只有 A 能够用私钥解密 B 发的信息，具有机密性，出现的问题是，任何人都可以用公钥加密信息发给 A，C 可能伪造 B 的信息发给 A。不具有可认证性。

   发送者的私钥，接受者的公钥，双重加密，具有机密性（接受者 公钥加密），具有可认证性（发送者 私钥数字签名）

现实世界中常常使用 bcrypt 用来生成密码，而不是自己存 salt 和 hash。







-------

通过 6443 port 从外部访问 k3s 这并不是一个好的做法。

K8s 默认通过 客户端证书认证，也就是 kubeconfig 中的信息。

k3s 默认通过 账号和密码。https://github.com/rancher/k3s/issues/1616

在 k3s 中有两个 ca, client-ca 和 server-ca, 如果验证 HTTPS / TLS 则使用 server 端，client certificate 认证则使用 client 端。

签发一个客户端证书用来 认证。

server 的证书是用来 HTTPS 验证的。并且在外部用的证书和内部使用的是不一样的。

https://github.com/rancher/k3s/issues/1868#issuecomment-640429039

https://github.com/rancher/k3s/pull/1855

