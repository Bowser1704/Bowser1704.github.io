---
title: "Helm in k3s"
summary: “国内环境在 k3s 内使用 Helm"
tags: [cs]
categories: [cloud-native]
spotlight: true
date: 2020-06-25T10:11:19+08:00
---

## 在 k3s 内使用 Helm（国内）

### 1. 国内访问问题

[Helm](https://helm.sh/) 可以类似理解为 ”K8s OS“ 的包管理工具，再以前没有这个工具我们要手动写 yaml/json 文件来自动化部署应用，但是 Helm 就类似于 Homebrew 一样，有人帮我们写好了配置文件（Chart），我们只需要一个命令，加自定义的参数就行了。

```bash
helm install stable/mysql --generate-name
```

Helm 官方提供了一个 Chart 仓库，[Helm Hub](https://hub.helm.sh/)，但是众说周知的原因，无法访问。阿里云原生团队提供了镜像服务，同步所有 Charts 并且将其中的 `gcr.io` 等镜像地址全部替换为国内可以访问的镜像。

[AppHub](https://github.com/cloudnativeapp/charts)

> AppHub 的主要职责之一，是把所有 Helm 官方 Hub 托管的应用，都自动同步到国内，并自动将 Charts 文件中的 gcr.io 等有网络访问问题的 URL 替换成为稳定的国内镜像 URL。

### 2. k3s 提供的内建 CRD

k3s 内建了一个 CRD 并且也内置了 Helm V3 & V2， 从而使得 Helm 的使用可以使用另一种方式。

> 什么是 [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)，Custom Controllers，参考 k8s 官方介绍。
>
> 什么是 [Operator](https://coreos.com/operators/)，参考 CoreOS 的官方介绍。

下面是 k3s 中内建 Helm Operator 的资源类型。

```bash
[root@bowser1704 ~]# kubectl api-resources --api-group=helm.cattle.io
NAME         SHORTNAMES   APIGROUP         NAMESPACED   KIND
helmcharts                helm.cattle.io   true         HelmChart
```

那么在这个 CRD 里面怎么使用国内镜像呢？

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: example-helmchart
  namespace: kube-system
spec:
  chart: test-helmchart
  targetNamespace: test-namespace
  version: 0.0.1 
  repo: https://apphub.aliyuncs.com
```

-------

参考文献。

1. [Operator 与 Controller 的区别](https://octetz.com/docs/2019/2019-10-13-controllers-and-operators/)