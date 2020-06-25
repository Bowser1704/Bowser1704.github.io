---
title: 在 k3s 内使用 cert-manager 管理证书
summary: “云原生使用 acme.sh 有一些无法很好解决的问题，所以使用 cert-manager 管理证书"
tags: [cs]
categories: [cloud-native]
spotlight: true
date: 2020-06-25T19:30:20+08:00
---
## acme.sh vs cert-manager

两者的区别就是普通 acme 客户端与 cloudnative acme 客户端，acme.sh 可以很好的更新证书，但是在集群里面我们 cert 使用的是 secret 目前没有办法能够自动更新 secret 所以 cert 更新还是没有用，还是要手动去修改。

但是 cert-manager 作为一个 operator 可以定时更新 secret 并且 作为一个 acme.sh 客户端。实现了 acme 协议 以及 云原生更新证书的办法。

![High level overview diagram explaining cert-manager architecture](https://cert-manager.io/images/high-level-overview.svg)


## 使用 cert-manager

普通使用 Helm 安装。
```bash
# step1 添加仓库
$ helm repo add jetstack https://charts.jetstack.io && helm repo update

# step2 创建 ns
$ kubectl create namespace cert-manager

# step3 创建 CRD
# Kubernetes 1.15+ yaml 文件在 [GitHub官方主页](https://github.com/jetstack/cert-manager) 下载
#$ kubectl apply --validate=false -f cert-manager.crds.yaml
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager.yaml


# step4 使用 HelmChart 安装 cert-manager 或者直接 Helm 安装。
# Helm v3+
$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.15.1 \
  # --set installCRDs=true

# Helm v2
$ helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.15.1 \
  jetstack/cert-manager \
  # --set installCRDs=true
```

使用 k3s 提供的 HelmChart 安装
```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
	helmVersion: v3
  chart: cert-manager
  targetNamespace: cert-manager
  version: v0.15.1 
  repo: https://charts.jetstack.io
```

### 安装完成之后配置

使用 ClusterIssuer，作用于整个集群。

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: 896379346@qq.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
       ingress:
         class: traefik
```

可以使用两种检测方式，一种为 `HTTP-01`、一种为 `DNS-01`，区别参考[Let’s Encrypt 验证方式](https://letsencrypt.org/zh-cn/docs/challenge-types/)。

如果使用 `DNS-01` 方式，因为原生不支持 AliDNS，所以可以使用 Alidns-[webhook](https://github.com/pragkent/alidns-webhook)。

-----

出现错误

```bash
Error from server (InternalError): error when creating "cluster-issuer.yaml": Internal error occurred: failed calling webhook "webhook.cert-manager.io": Post https://cert-manager-webhook.cert-manager.svc:443/mutate?timeout=30s: dial tcp 10.43.247.15:443: i/o timeout
```

定位错误

- https://github.com/jetstack/cert-manager/issues/2811

  猜想是 flannel 的后端 VXLAN 的问题。issue 中暂时解决的办法是重启 k3s 更换 后端。

- https://github.com/coreos/flannel/issues/1243

  上次其他服务碰到过一次这个问题，也就是说，也是 flannel 的问题。

  k3s 默认使用的 flannel + VXLAN 网络。也许我们可以使用 host-gw 作为后端。

- https://github.com/rancher/k3s/issues/538

- https://rancher.com/docs/k3s/latest/en/installation/network-options/ 

  可以通过修改 k3s.service 然后使用 `systemctl daemon-reload` restart k3s。

