---
title: "单机 Prometheus 监控多集群方案"
summary: ""
tags: [prometheus, monitoring, k3s]
categories: [cloud-native]
spotlight: true
date: 2020-07-01T22:21:01+08:00
---

## 初始化：安装各种组件

Prometheus 和 Alertmanager 的安装都在[官网](https://prometheus.io/download/) 下载压缩包，手动下载下来解压。

```bash
# -C 指定目的目录
# * 代表版本，需要下载正确的系统版本。
tar xf prometheus-*.*.*.linux-amd64.tar.gz -C /opt/
tar xf alertmanager-*.*.*.linux-amd64.tar.gz -C /opt/
```

安装之后使用 systemd 来管理这几个服务。手写 service 放到 systemd 文件夹下面，我的目标是 `/usr/lib/systemd/system/` 。

prometheus 可以热加载配置文件，如果需要通过 web API 重启，必须要加参数 `--web.enable-lifecycle`，不安全，不推荐这么做。我们使用 systemd 管理 service，可以直接用 restart 命令重启。

注意我新建了两个配置文件，名字都为 config.yaml。

```service
[Unit]
Description=Prometheus Server
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStartPre=/opt/prometheus/promtool check config /opt/prometheus/config.yaml
ExecStart=/opt/prometheus/prometheus \
  --config.file /opt/prometheus/config.yaml

[Install]
WantedBy=multi-user.target
```

alertmanager

```service
[Unit]
Description=Alertmanager Server
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStartPre=/opt/alertmanager/amtool check-config /opt/alertmanager/config.yaml
ExecStart=/opt/alertmanager/altermanager \
  --config.file=/opt/alertmanager/config.yaml \
  --log.level=debug \
  --cluster.listen-address=""

[Install]
WantedBy=multi-user.target
```

安装 node-exporter 和 kube-state-metrics。

> 在 k3s 中我使用 helmchart 安装，但是这个 CRD 有一些问题，可以更换到 [helm-operator](https://github.com/fluxcd/helm-operator)。

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: prometheus-node-exporter
  namespace: kube-system
spec:
  helmVersion: v3
  chart: prometheus-node-exporter
  targetNamespace: monitoring
  repo: https://apphub.aliyuncs.com
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: kube-state-metrics
  namespace: kube-system
spec:
  helmVersion: v3
  chart: kube-state-metrics
  targetNamespace: monitoring
  repo: https://apphub.aliyuncs.com
```

安装两个 exporter 之后。

```bash
[root@bowser1704 ~]# kubectl get svc -n monitoring
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kube-state-metrics         ClusterIP   10.43.204.99    <none>        8080/TCP   19h
prometheus-node-exporter   ClusterIP   10.43.222.110   <none>        9100/TCP   39h
```

## 单机监控中心监控多集群

由标题，想要单机 prometheus 监控多个集群，不考虑单机压力问题，prometheus 配置和 exportor 的安装需要考虑两个问题：

1. prometheus 通过公网访问集群的 metrics。
2. 不使用 [prometheus federation](https://prometheus.io/docs/prometheus/latest/federation/#federation) 或者 [thanos](https://thanos.io/) 采集多集群的信息。

### 1. 公网访问集群的 metrics

集群内部已经自建了一些 metrics 指标，这种指标可以直接通过它们暴露的 API 访问，例如 kubelet metrics，cAdvisor，API server....。也可以通过 API server 提供的 proxy 访问。无论是怎么访问，从集群外部访问集群内部都需要手动配置 Authorization 可以直接通过 Bearer Token，也可以通过账号和密码。这里使用 token。

首先使用 [rbac](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/) 创建一个 serviceaccount，并且创建 clusterrole，和做 clusterbinding。

可以参考我的 [gist](https://gist.github.com/Bowser1704/63d982782a3a8645316669d3100d8fc1)。

> gist 中的 sa 并不能直接使用，需要后面进行二次修改，否则没有权限访问。
> 
> 使用 clusterrole 需要访问集群所有信息，不考虑某个 namespace。

```bash
# 假设创建的 serviceaccount 是 monitoring 下的 prometheus
kubectl get sa prometheus -n monitoring -o yaml

# secret 的名字是上面获取的
kubectl describe secret prometheus-token-cw4pd -n monitoring
```

#### 1.1 使用原生 API 访问内建指标

> 因为需要使用 https 但是证书是自签的，所以需要自己拿 ca.crt 下来，但是为了方便直接 disable validate certificates 了。

- kubelet
  
  Kubelet 监听于 10250 端口，暴露的 API 为 https://public-ip:10250/metircs，需要给 sa 的 clusterrole 设置一定的权限从而可以访问这个 API。我们使用上面创建的 sa，并且这个 sa 的 role 没有什么权限，假设只有一个 pod get 的权限，访问 API，结果是：
  
  ```http
  Forbidden (user=system:serviceaccount:monitoring:prometheus, verb=get, resource=nodes, subresource=metrics)
  ```
  
  说明这个 API 需要的权限是
  
  ```yaml
  - apiGroups:
    - ""
    resources:
    - nodes
    - nodes/metrics
    verbs:
    - get
  ```
  
  在我们的 ClusterRole 里面加上这个权限。然后就可以拿到数据了，因为 http 的[分块传输编码](https://zh.wikipedia.org/wiki/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81)，以及数据量很大，需要等待较长一段时间。
  
  而在 prometheus.yml 里面的设置为。
  
  > 最下面的 relabel_configs 是把自带的 label 全都拿过来。
  
  ```yaml
  # 我们的场景需要设置两个 tls_config 和 bearer_token。
  - job_name: 'kubelet'
      scheme: https
      tls_config:
      insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      static_configs:
      - target:
      - public-ip:10250
      relabel_configs:
      - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
  ```

- cAdvisor
  
  由于高版本的 cAdvisor 合并到了 kubelet 里面，并且经过实践权限不需要修改，所以可以可以直接添加 prometheus job 就可以了。暴露的 API 为 https://public-ip:10250/metircs/cadvisor。
  
  ```yaml
  - job_name: 'cAdvisor'
      scheme: https
      tls_config:
      insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      static_configs:
      - target:
      - public-ip:10250
      relabel_configs:
      - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
      - source_labels: [__meta_kubernetes_node_name]
      regex: (.+)
      target_label: __metrics_path__
      replacement: metrics/cadvisor
  ```
  
  > 最下面的 relabel_configs 是把内建的  `__metrics_path__` 修改为 metircs/cadvisor 默认为 metrics。

-

  ......

#### 1.2 使用 API server 代理访问到内建指标

根据 kubernetes 的架构，我们是通过 API server 操纵访问集群内的所有 API 对象的，并且我们也可以通过 API server 代理到我们的 service，pod 从而达到外网访问集群内部资源的效果。

这里还要介绍一个 prometheus config [kubernetes_sd_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)，请参考文档详细配置信息。

> Kubernetes SD configurations allow retrieving scrape targets from [Kubernetes'](https://kubernetes.io/) REST API and always staying synchronized with the cluster state.

也就是说设置 kubernetes_sd_config 可以帮助我们自动发现需要监控的指标，可以根据集群的状态变化，例如你加了一个 node 他可能多了几个指标，这个时候就不需要我们手动去设置了，也就是实现了 service discovery。

- kubelet
  
  账号依然是 1.1 中加了权限后的账号。token 还是那个 token， 证书可选。
  
  下面是 config。我在 relabel_configs 主要做了几点配置：
  
  1. 把 discover 发现的 pod ip，修改为 API server 暴露的地址 / master node public ip。
     
     > prometheus 找到并且使用的都是 pod ip / endpoints。
  
  2. 根据每个 node 的名字构建 metrics path，因为 kubelet metrics 每个 node 都有一个，所以需要采集多个 node 的 metircs，利用 kubernetes_sd_config 可以自动发现，不需要手动填写。
  
  ```yaml
  - job_name: stage-cadvisor
    honor_timestamps: true
    scrape_interval: 15s
    scrape_timeout: 15s
    metrics_path: /metrics
    scheme: https
    kubernetes_sd_configs:
    - api_server: https://:6443
      role: node
      bearer_token_file: /opt/prometheus/serviceaccount/stage/token
      tls_config:
        ca_file: /opt/prometheus/serviceaccount/stage/ca.crt
        insecure_skip_verify: true
    bearer_token_file: /opt/prometheus/serviceaccount/stage/token
    tls_config:
      ca_file: /opt/prometheus/serviceaccount/stage/ca.crt
      insecure_skip_verify: true
    relabel_configs:
    - separator: ;
      regex: __meta_kubernetes_node_label_(.+)
      replacement: $1
      action: labelmap
    - separator: ;
      regex: (.*)
      target_label: __address__
      replacement: public-ip:6443
      action: replace
    - source_labels: [__meta_kubernetes_node_name]
      separator: ;
      regex: (.+)
      target_label: __metrics_path__
      replacement: /api/v1/nodes/${1}/proxy/metrics
      action: replace
  ```

- cAdvisor
  
  cAdivisor 和 kubelet 类似，只需要在 metrics path 后面加一个 cadvisor 就可以了。
  
  ```yaml
  - job_name: stage-cadvisor
    honor_timestamps: true
    scrape_interval: 15s
    scrape_timeout: 15s
    metrics_path: /metrics
    scheme: https
    kubernetes_sd_configs:
    - api_server: https://public-ip:6443
      role: node
      bearer_token_file: /opt/prometheus/serviceaccount/stage/token
      tls_config:
        ca_file: /opt/prometheus/serviceaccount/stage/ca.crt
        insecure_skip_verify: true
    bearer_token_file: /opt/prometheus/serviceaccount/stage/token
    tls_config:
      ca_file: /opt/prometheus/serviceaccount/stage/ca.crt
      insecure_skip_verify: true
    relabel_configs:
    - separator: ;
      regex: __meta_kubernetes_node_label_(.+)
      replacement: $1
      action: labelmap
    - separator: ;
      regex: (.*)
      target_label: __address__
      replacement: public-ip:6443
      action: replace
    - source_labels: [__meta_kubernetes_node_name]
      separator: ;
      regex: (.+)
      target_label: __metrics_path__
      replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
      action: replace
    - separator: ;
      regex: (.*)
      target_label: cluster
      replacement: stage
      action: replace
  ```

- apiserver
  
  apiserver 可以直接通过 API server 暴露的地址加上一个 metrics path 就可以了，例如 https://public-ip:6443/metrics。
  
  我在 relabel_configs 主要做的配置是：
  
  1. 把 discover 发现的 pod ip，修改为 API server 暴露的地址 / master node public ip。
  
  ```yaml
  - job_name: stage-apiservers
    honor_timestamps: true
    scrape_interval: 15s
    scrape_timeout: 15s
    metrics_path: /metrics
    scheme: https
    kubernetes_sd_configs:
    - api_server: https://public-ip:6443
      role: endpoints
      bearer_token_file: /opt/prometheus/serviceaccount/stage/token
      tls_config:
        insecure_skip_verify: true
    bearer_token_file: /opt/prometheus/serviceaccount/stage/token
    tls_config:
      insecure_skip_verify: true
    relabel_configs:
    - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
      separator: ;
      regex: default;kubernetes;https
      replacement: $1
      action: keep
    - separator: ;
      regex: (.*)
      target_label: __address__
      replacement: public-ip:6443
      action: replace
  ```

#### 1.3 使用 API server 代理访问到自建指标

你可以将自建的 exporter 都部署到一个 namespace 然后根据这个 namespace 的名字抓取所有的 metrics，或者根据某个 annotation 选择，但是这样的坏处是，所有的指标都在一个 job 下面，不太美观，出了问题不能一样排查出来，所以我选择的是 node-exporter，kube-state-metrics，traefik 都分开来。

- Node-exporter
  
  node-exporter 会根据 node 的个数自己变化，所以我们肯定需要使用 kubernetes_sd_config 来自动发现。
  
  下面是 config，注意我们 kubernetes_sd_config 的 role 是 endpoints。在 relabel_configs 做的配置是
  
  1. 匹配 __meta_kubernetes_endpoints_name 为 prometheus-node-exporter 的，实际上这里是你 node-exporter service 的 name。
     
     > 你依然可以指定 namespace，指定 annotation，但是如果确定这个已经是唯一定位到你需要的 svc 就不需要写那么多了。
  
  2. 更换 metrics path
     
     目标为 `/api/v1/namespaces/$1/services/$2:$3/proxy$4`。
  
  ```yaml
  - job_name: stage-node-exporter
    honor_timestamps: true
    scrape_interval: 15s
    scrape_timeout: 15s
    metrics_path: /metrics
    scheme: https
    kubernetes_sd_configs:
    - api_server: https://public-ip:6443
      role: endpoints
      bearer_token_file: /opt/prometheus/serviceaccount/stage/token
      tls_config:
        insecure_skip_verify: true
    bearer_token_file: /opt/prometheus/serviceaccount/stage/token
    tls_config:
      insecure_skip_verify: true
    relabel_configs:
    - source_labels: [__meta_kubernetes_endpoints_name]
      separator: ;
      regex: prometheus-node-exporter
      replacement: $1
      action: keep
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
      separator: ;
      regex: (\d+)
      target_label: __meta_kubernetes_pod_container_port_number
      replacement: $1
      action: replace
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
      separator: ;
      regex: ()
      target_label: __meta_kubernetes_service_annotation_prometheus_io_path
      replacement: /metrics
      action: replace
    - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_pod_container_port_number, __meta_kubernetes_service_annotation_prometheus_io_path]
      separator: ;
      regex: (.+);(.+);(.+);(.+)
      target_label: __metrics_path__
      replacement: /api/v1/namespaces/$1/services/$2:$3/proxy$4
      action: replace
    - separator: ;
      regex: (.*)
      target_label: __address__
      replacement: public-ip:6443
      action: replace
    - separator: ;
      regex: __meta_kubernetes_service_label_(.+)
      replacement: $1
      action: labelmap
    - source_labels: [__meta_kubernetes_namespace]
      separator: ;
      regex: (.*)
      target_label: kubernetes_namespace
      replacement: $1
      action: replace
    - source_labels: [__meta_kubernetes_service_name]
      separator: ;
      regex: (.*)
      target_label: kubernetes_name
      replacement: $1
      action: replace
    - source_labels: [__meta_kubernetes_pod_node_name]
      separator: ;
      regex: (.*)
      target_label: instance
      replacement: $1
      action: replace
  ```

其他的自建指标都与这个类似，使用 label 可以唯一找到目标就行了。

同时有些指标可能不需要服务发现，可能只有一个 endpoint，那我们其实可以手动写 static_config。

- kube-state-metrics
- traefik
- CoreDNS

完整 config 可以参考 [gist](https://gist.github.com/Bowser1704/9ceb310cf9d0f1f605a5eeafb6ddbce4)

### 2. 监控多集群

由于 Prometheus 监控的单位是 job，并且一个 job 里面不能写多个 kubernetes_sd_config，但是 static_configs 里面可以有多个 target。所以说我们有两种思路：

1. 需要使用 kubernetes_sd_config 的，每个集群新建一个 job，而其他则可以使用 static_configs 并且使用 label 来区分。
   
   例如：
   
   ```yaml
   static_configs:
     - targets:
       - stage-ip:6443
       labels:
         cluster: stage
     - targets:
         - product-ip:6443
         labels:
             cluster: product
   ```

2. 每个集群都新建所有 job，这个比较简单，有了一个模版，改字段就可以了，我上面的 config 就是这样，如果是 product1 集群，这 Crtl+R 替换所有的 stage 为 product1。
   
   > stage、product 等名字都是自己定的。
   
   也不需要再添加新的 label，因为 job_name 就可以作为一个 label。

最后的结果也是汇总到同一个 time series 下面的。

## Prometheus Operator

目前来说集群的主流监控方案，使用的都是 [prometheus](https://prometheus.io/) 组件，并且配合 [Grafana](https://grafana.com/) 实现可视化。而随着 [Operators](https://coreos.com/blog/introducing-operators.html) 的提出，以及 CoreOS 首先实现的两个 Operator 其中之一是 [Prometheus Operator](https://coreos.com/blog/the-prometheus-operator.html)，主流的部署方案也不再是原生写 yaml 文件来部署 prometheus 等软件了，而是采用 CoreOS 提供的 [Prometheus Operator](https://github.com/coreos/prometheus-operator) 部署，另外由于 Helm 的流行，更多的人使用 Helm 来安装 Prometheus Operator 从而使得集群监控方案的部署就像普通 Linux 发行版安装一个软件时使用一条包管理器命令一样简单。

> To demonstrate the Operator concept in running code, we have two concrete examples to announce as open source projects today:
> 
> 1. The [*etcd Operator*](https://coreos.com/blog/introducing-the-etcd-operator.html) creates, configures, and manages etcd clusters. etcd is a reliable, distributed key-value store introduced by CoreOS for sustaining the most critical data in a distributed system, and is the primary configuration datastore of Kubernetes itself.
> 2. The [*Prometheus Operator*](https://coreos.com/blog/the-prometheus-operator.html) creates, configures, and manages Prometheus monitoring instances. Prometheus is a powerful monitoring, metrics, and alerting tool, and a Cloud Native Computing Foundation (CNCF) project supported by the CoreOS team.
> 
> --- CoreOS

## 为什么不使用原生 Prometheus

众所周知，目前来说，k8s 对于有状态（stateful）应用的管理还是没有很好，可以说是一个痛点，所以像 databases, caches, 和 monitoring systems 这些有状态应用，手动使用 StatefulSet 来部署，不过显然是没有 Operator 来的方便。

> operater 可以看成一个特定应用的 SRE 保姆。

并且使用 Operator 将很多复杂的配置都抽离出来了。使部署一些需要高运维的工具不再那么容易。不过我们的场景用不了 Operator。




