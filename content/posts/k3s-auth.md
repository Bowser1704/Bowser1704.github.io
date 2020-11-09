---
title: k3s authentication certificate
summary: ""
tags: [k3s]
categories: [cloud-native]
spotlight: false
date: 2020-11-08T22:18:18+08:00
---

## k3s authentication

- client certificate 也是一种认证方式
- token
- username and password

## certificate

在 k8s 的世界里面有两种证书，一种是 client certificate 用于认证，一种是 server certificate 用于 TLS。

用于认证的就是 `kubeconfig` 文件里面的 `client-certificate-data ` 和 `client-key-data`

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://kubernetes.docker.internal:6443
  name: docker-desktop
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-for-desktop
current-context: docker-desktop
kind: Config
preferences: {}
users:
- name: docker-desktop
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

- `client-certificate-data `

  证书文件，里面包含 public key 和 signature，在 k3s 中由 client-ca 签发。

- `client-key-data`

  上面 public key 的 private key，client certificate 这种认证方式所需要的。

- `certificate-authority-data`

  这是 ca.crt 也就是 ca 的根证书，在 k3s 中这个是 server-ca 签发。也就是 server-ca 的证书。

> k3s 中分了两个 ca，一个是 server-ca，一个是 client-ca，这不是必须的，可以只有一个 ca 签发所有证书。

### TLS 验证

在 k3s 中，有两个证书用于 HTTPS 验证，

- `serving-kube-apiserver.crt`

   `/var/lib/rancher/k3s/server/tls`  中的 `serving-kube-apiserver.crt` 这个用于集群内部组件认证。

  这个证书当我们重启机器的时候 k3s 会有一个 rotation。

- `k3s-serving`
  
  kube-system namespace 下面的 secret `k3s-serving`，用于外部访问，例如 kubectl，或者 curl 之类的。
  
  这个证书不会自动更新，所以可能集群内部没挂，但是你不能通过 ca.crt 访问他。
  
  经过检验，在 pod 内部访问依旧是返回的这个 cert。

参考 issue https://github.com/rancher/k3s/pull/1855

### serviceaccount

在 k3s 的 serviceaccout 中，默认的 secret 也就是 xxxx-token-xxx，里面的信息有三部分。

- ca.crt 也就是 server-ca.crt 用于 TLS 验证，也就是一般我们系统内置的根证书。
- token 就是认证信息
- namespace  也就是这个 sa 的 ns。

