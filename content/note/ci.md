---
title: CI 实践
summary: ""
tags: [ci]
categories: [cloud-native]
spotlight: false
date: 2020-08-03T16:55:07+08:00
---

## CI

1. Drone CI

   我们使用 Drone 和 GitHub Actions 一起作为 CI/CD 工具，

   - GitHub Actions 机器性能好，操作较为简单。
   - Drone 可定制程度高，拿到手之后想怎么做都可以。并且因为国内的网络环境，CD(Continous Delpoy) 放在国内是比较正确的。

   镜像构建放在 Github Actions 实际上比较好，减轻了我们的机器压力。但是镜像只能放在国内 deploy 才方便。所以要确认一下 GitHub 对阿里云镜像仓库的连通性。

   通过 Drone CI 来构建是正确的。比较轻便。

## CD

1. Argo CD

   直接访问 Git 可以不使用 hook ，可以直接通过一个 git 地址「ssh or https」，以及账号密码访问到仓库。

   真正的作用是 部署服务到 kubernetes 但必须在集群内，使用了 CRD。实际上感觉过于冗余了。

   限制就是一定要把 Argo CD server 部署在 kubernetes 上。

   ![Argo CD Architecture](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghltjcqshdj30kn0jo40k.jpg)

   ![Argo cloud-native CI/CD Workflow](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghmna3xsojj30sg0grjsp.jpg)

2. 手写 yaml 部署。

木犀云：

-

为什么不使用 Helm 实现 CD pipeline。

kruise



https://www.inovex.de/blog/spinnaker-vs-argo-cd-vs-tekton-vs-jenkins-x/





Argo CD 的使用

1. Argo CD 必须部署在 kubernetes 里面。「缺点之一」
2. 需要两个仓库 一个是代码仓库，一个是 deploy 仓库。
   1. 首先依旧使用 Drone CI 用于 build 以及修改 deploy 仓库，而 Argo CD 监控 deploy 仓库。
   2. 当 Argo CD 感知到 Argo CD 仓库发生变化，就会做一些 deploy 工作。





Muxi App Engine (MAE):

下面是需求

1. MKE (Muxi Kubernetes Engine) 能够在网站里面部署、回滚、向不同的机器部署、能够查看 pod 的 log。

2. 镜像构建服务，镜像构建基于 Drone CI，需要把它与整个 MAE 接入，需要接入阿里云镜像管理 API。

   有一些 1+2 的例子：

   - https://github.com/rancher/rio

3. 服务器管理系统 「阿里云机器管理「ECS、RDS」，利用子账号、阿里云开放 API」，可以看到过期时间、地区、可用区，更进一步的也许可以直接通过这个来续费，并且来上报学生机。

4. 基于 Prometheus 数据的监控仪板。通过写 PromQL 查询需要的数据，然后前端渲染。

下面是注意的地方 ⚠️

1. 需要利用到 RBAC 对每个人的权限进行控制。
2.
