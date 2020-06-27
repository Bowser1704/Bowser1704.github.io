---
title: 在 k3s 内使用 cert-manager 管理证书
summary: “云原生使用 acme.sh 有一些无法很好解决的问题，所以使用 cert-manager 管理证书"
tags: [k3s, cert-manager]
categories: [cloud-native]
spotlight: truegy
date: 2020-06-25T19:30:20+08:00
---
## acme.sh vs cert-manager

两者的区别就是普通 acme 客户端与 cloudnative acme 客户端，acme.sh 可以很好的更新证书，但是在集群里面我们 cert 使用的是 secret 目前没有办法能够自动更新 secret 所以 cert 更新还是没有用，还是要手动去修改。

但是 cert-manager 作为一个 operator 可以定时更新 secret 并且 作为一个 acme.sh 客户端。实现了 acme 协议 以及 云原生更新证书的办法。

![High level overview diagram explaining cert-manager architecture](https://cert-manager.io/images/high-level-overview.svg)

简单讲一下 cert-manager 的原理，普通 acme.sh 客户端交付给我们的是证书，附带的功能也是更新证书，但是 cert-manager 交付给我们的是 一个 CRD Issuer/ClusterIssuer 这个对象可以用来签发  另一个 CRD [Certificate](https://cert-manager.io/docs/concepts/certificate/) 。Certificate 会自动生成 tls secret 并且管理更新。并且 Certificate 就是真正的颁发机构签发的证书，cert-manager 用他来生产 secret。

## 使用 cert-manager

普通使用 Helm 安装。
```bash
# step1 添加仓库
# 使用 HelmChart 不需要手动添加仓库
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

使用 k3s 提供的 HelmChart 安装 参考 [在 k3s 上使用 Helm](https://bowser1704.github.io/blog/20200625-helm-in-k3s/)
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

1. 使用 ClusterIssuer，作用于整个集群。

    ```yaml
    apiVersion: cert-manager.io/v1alpha2
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        # server: https://acme-v02.api.letsencrypt.org/directory
        # 建议先使用 测试 API，区别参考 https://letsencrypt.org/zh-cn/docs/staging-environment/
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        email: name@user.com
        privateKeySecretRef:
          name: letsencrypt-prod
        solvers:
        - http01:
           ingress:
             class: traefik
    ```
    可以使用两种检测方式，一种为 `HTTP-01`、一种为 `DNS-01`，区别参考[Let’s Encrypt 验证方式](https://letsencrypt.org/zh-cn/docs/challenge-types/)。

    如果使用 `DNS-01` 方式，因为原生不支持 AliDNS，所以可以使用 Alidns-[webhook](https://github.com/pragkent/alidns-webhook)。

2. ClusterIssuer 创建成功之后，创建 Certificate，针对于特定命名空间的。

   ```yaml
   apiVersion: cert-manager.io/v1alpha2
   kind: Certificate
   metadata:
     name: food
     namespace: food
   spec:
     secretName: food-secret
     dnsNames:
     - food.test.domain.com
     issuerRef:
       name: letsencrypt-prod
       kind: ClusterIssuer
   ```

   配置成功之后查看一下 certificate 的状态

   ```bash
   [root@bowser1704 food]# kubectl get certificate -n food
   NAME   READY   SECRET        AGE
   food   True    food-secret   3h27m
   ```

3. ingress 设置

   ```yaml
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: food
     namespace: food
     annotations:
       kubernetes.io/ingress.class: traefik
       # 添加一个 annotations
       cert-manager.io/issuer-name: letsencrypt-prod
   spec:
     rules:
     - host: food.test.domain.com
       http:
         paths:
         - path: /
           backend:
             serviceName: food-backend
             servicePort: http
     tls:
     - hosts:
       - food.test.domain.com
       # certificate 里面的 secret-name
       secretName: food-secret
   ```

   大功告成！以后就不会发生证书过期的事情了。

-----

出现错误

```bash
Error from server (InternalError): error when creating "cluster-issuer.yaml": Internal error occurred: failed calling webhook "webhook.cert-manager.io": Post https://cert-manager-webhook.cert-manager.svc:443/mutate?timeout=30s: dial tcp 10.43.247.15:443: i/o timeout
```

定位错误

- https://github.com/jetstack/cert-manager/issues/2811

  猜想是 flannel 的后端 VXLAN 的问题。issue 中暂时解决的办法是重启 k3s 更换 后端。

- https://github.com/coreos/flannel/issues/1243

  https://github.com/rancher/k3s/issues/1613

  上次其他服务碰到过一次这个问题，也就是说，也是 flannel 的问题。

  k3s 默认使用的 flannel + VXLAN 网络。也许我们可以使用 host-gw 作为后端。

- https://github.com/rancher/k3s/issues/538

- https://rancher.com/docs/k3s/latest/en/installation/network-options/ 

  可以通过修改 k3s.service 然后使用 `systemctl daemon-reload` restart k3s。

debug

1. 检查 svc 和 endpoint。

   ```bash
   # svc ip 
   [root@bowser1704 ~]# httpstat -k 10.43.247.15:443
   ^C
   
   # endpoint or pod ip
   [root@bowser1704 ~]# httpstat -k 10.42.1.28:10250
   
   Connected to 10.42.1.28:10250
   
   HTTP/1.1 404 Not Found
   Content-Length: 19
   Content-Type: text/plain; charset=utf-8
   Date: Fri, 26 Jun 2020 07:08:51 GMT
   X-Content-Type-Options: nosniff
   
   Body discarded
   
     DNS Lookup   TCP Connection   TLS Handshake   Server Processing   Content Transfer
   [      0ms  |           0ms  |         44ms  |              0ms  |             0ms  ]
               |                |               |                   |                  |
      namelookup:0ms            |               |                   |                  |
                          connect:0ms           |                   |                  |
                                      pretransfer:45ms              |                  |
                                                        starttransfer:45ms             |
                                                                                   total:45ms
   ```

   定位到是 svc 的问题。发现所有的 svc 都访问不到。

2. 检查 iptables

   ```bash
   [root@bowser1704 ~]# iptables-save | grep 10.43.247.15
   -A KUBE-SERVICES ! -s 10.42.0.0/16 -d 10.43.247.15/32 -p tcp -m comment --comment "cert-manager/cert-manager-webhook:https cluster IP" -m tcp --dport 443 -j KUBE-MARK-MASQ
   -A KUBE-SERVICES -d 10.43.247.15/32 -p tcp -m comment --comment "cert-manager/cert-manager-webhook:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-ZUD4L6KQKCHD52W4
   
   [root@bowser1704 ~]# iptables-save | grep KUBE-SVC-ZUD4L6KQKCHD52W4
   :KUBE-SVC-ZUD4L6KQKCHD52W4 - [0:0]
   -A KUBE-SERVICES -d 10.43.247.15/32 -p tcp -m comment --comment "cert-manager/cert-manager-webhook:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-ZUD4L6KQKCHD52W4
   -A KUBE-SVC-ZUD4L6KQKCHD52W4 -j KUBE-SEP-MM5Y2SAJQYJV572E
   
   [root@bowser1704 ~]# iptables-save | grep KUBE-SEP-MM5Y2SAJQYJV572E
   :KUBE-SEP-MM5Y2SAJQYJV572E - [0:0]
   -A KUBE-SEP-MM5Y2SAJQYJV572E -s 10.42.1.28/32 -j KUBE-MARK-MASQ
   -A KUBE-SEP-MM5Y2SAJQYJV572E -p tcp -m tcp -j DNAT --to-destination 10.42.1.28:10250
   -A KUBE-SVC-ZUD4L6KQKCHD52W4 -j KUBE-SEP-MM5Y2SAJQYJV572E
   
   # 下面是 KUBE-MARK-MASQ 的作用。
   -A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
   ```

   发现规则都在，没有问题。但是 svc 都访问不到。IPVS 可能有帮助？？？

3. 但是发现在 pod 的那个 node 上可以访问到 svc。使用 nodeSelector 转移到 master node. 也可以访问到了。

   ```bash
   [root@bowser1704 cert-manager]# kubectl get svc -n cert-manager
   NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
   cert-manager           ClusterIP   10.43.157.248   <none>        9402/TCP   20h
   cert-manager-webhook   ClusterIP   10.43.247.15    <none>        443/TCP    20h
   
   # can't access
   [root@bowser1704 cert-manager]# curl -k https://10.43.247.15:443
   ^C
   
   # check node
   [root@bowser1704 cert-manager]# kubectl describe pods cert-manager-webhook-7c6c464d7b-vp44p -n cert-manage
   
    ...
   Node:         foo1/172.19.145.208
   ...
   ```

   使用 nodeSelector 将 pod 迁移到 master node.

   ```bash
   # edit pod yaml
   nodeSelector:
    k3s.io/internal-ip: 172.16.219.51
      
   [root@bowser1704 cert-manager]# kubectl describe pods cert-manager-webhook-7c6c464d7b-vp44p -n cert-manage
   
   ...
   Node:         bowser1704/172.19.145.188
   ...
      
   # accessing in master node
   [root@bowser1704 cert-manager]# curl -k https://10.43.247.15:443
   404 page not found
   ```

   4. 提 [issue](https://github.com/rancher/k3s/issues/1958) 

定位到问题 为 flannel 后端 vxlan 的问题。解决方案参考 [记录一次 vxlan Bug](https://bowser1704.github.io/blog/20200627-%E8%AE%B0%E5%BD%95%E4%B8%80%E6%AC%A1-vxlan-bug/)

----------------------

参考链接

1. https://community.hetzner.com/tutorials/howto-k8s-traefik-certmanager