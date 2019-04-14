---
title: kubernetes 指标采集组件 metrics-server 的部署
date: 2019-04-14 21:27:30
tags: ["metrics-server"]
type: "metrics-server"

---

[metrics-server](https://github.com/kubernetes-incubator/metrics-server) 是一个采集集群中指标的组件，类似于 cadvisor，在 v1.8 版本中引入，官方将其作为 heapster 的替代者，metric-server 属于 core metrics(核心指标)，提供 API metrics.k8s.io，仅可以查看 node、pod 当前 CPU/Memory/Storage 的资源使用情况，也支持通过 Metrics API 的形式获取，以此数据提供给 Dashboard、HPA、scheduler 等使用。

#### 一、开启 API Aggregation

由于 metrics-server 需要暴露 API，但 k8s 的 API 要统一管理，如何将 apiserver 的请求转发给 metrics-server ，解决方案就是使用 [kube-aggregator](https://github.com/kubernetes/kube-aggregator) ，所以在部署 metrics-server 之前，需要在 kube-apiserver 中开启 API Aggregation，即增加以下配置：

```
--proxy-client-cert-file=/etc/kubernetes/certs/proxy.crt
--proxy-client-key-file=/etc/kubernetes/certs/proxy.key
--requestheader-client-ca-file=/etc/kubernetes/certs/proxy-ca.crt
--requestheader-allowed-names=aggregator
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User
```

如果kube-proxy没有在Master上面运行，还需要配置

```
--enable-aggregator-routing=true
```

[kube-aggregator](https://github.com/kubernetes/kube-aggregator)  的详细设计文档请参考：[configure-aggregation-layer](https://kubernetes.io/docs/tasks/access-kubernetes-api/configure-aggregation-layer/)

#### 二、部署 metrics-server

##### 1、获取配置文件

```
$ git clone  https://github.com/kubernetes/kubernetes
$ cd  kubernetes/cluster/addons/metrics-server/
```

##### 2、修改 metrics-server 配置参数

修改 `resource-reader.yaml` 文件：

```
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - nodes/stats    #新增这一行
  - namespaces
  verbs:
  - get
  - list
  - watch
```

修改 `metrics-server-deployment.yaml`文件:

```

      ......
      # metrics-server containers 启动参数作如下修改：
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.1
        command:
        - /metrics-server
        - --metric-resolution=30s
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
        # These are needed for GKE, which doesn't support secure communication yet.
        # Remove these lines for non-GKE clusters, and when GKE supports token-based auth.
        #- --kubelet-port=10255
        #- --deprecated-kubelet-completely-insecure=true
				
	......           
	# 修改启动参数：
        command:
          - /pod_nanny
          - --config-dir=/etc/config
          - --cpu=80m
          - --extra-cpu=0.5m
          - --memory=80Mi
          - --extra-memory=8Mi
          - --threshold=5
          - --deployment=metrics-server-v0.3.1
          - --container=metrics-server
          - --poll-period=300000
          - --estimator=exponential
          # Specifies the smallest cluster (defined in number of nodes)
          # resources will be scaled to.
          #- --minClusterSize={{ metrics_server_min_cluster_size }}
```

##### 3、部署

```
kubectl apply -f .  
```

metrics-server 的资源占用量会随着集群中的 Pod 数量的不断增长而不断上升，因此需要 addon-resizer 垂直扩缩 metrics-server。addon-resizer 依据集群中节点的数量线性地扩展 metrics-server，以保证其能够有能力提供完整的metrics API 服务，具体参考：[addon-resizer](https://github.com/kubernetes/autoscaler/tree/master/addon-resizer)。

>  所需要的镜像可以在 [k8s-system-images](https://github.com/gosoon/k8s-system-images.git)  中下载。



检查是否部署成功：

```
$ kubectl get apiservices | grep metrics
v1beta1.metrics.k8s.io     kube-system/metrics-server   True        2m

$ kubectl get pod -n kube-system
metrics-server-v0.3.1-65b6db6945-rpqwf   2/2     Running   0          20h
```



#### 三、metrics-server 的使用

由于采集数据间隔为1分钟，等待数分钟后查看数据：

```
$ kubectl top node
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node1            108m         2%     1532Mi          40%

$ kubectl top pod -n kube-system
NAME                                     CPU(cores)   MEMORY(bytes)
coredns-576cbf47c7-8v6n8                 2m           14Mi
coredns-576cbf47c7-qk7rk                 2m           10Mi
etcd-node1                               11m          80Mi
kube-apiserver-node1                     17m          566Mi
kube-controller-manager-node1            17m          67Mi
kube-flannel-ds-amd64-8lvs2              2m           13Mi
kube-proxy-85lhl                         3m           19Mi
kube-scheduler-node1                     5m           16Mi
metrics-server-v0.3.1-65b6db6945-rpqwf   2m           19Mi
```

Metrics-server 可用 [API](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/resource-metrics-api.md) 列表如下：

- `http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/nodes`
- `http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/nodes/<node-name>`
- `http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/pods`
- `http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/namespace/<namespace-name>/pods/<pod-name>`

由于 k8s 在 v1.10 后废弃了 8080 端口，可以通过代理或者使用认证的方式访问这些 API：
```
$ kubectl proxy
$ curl http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/nodes
```

也可以直接通过 kubectl 命令来访问这些 API，比如：
```
$ kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
$ kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
$ kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes/<node-name>
$ kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespace/<namespace-name>/pods/<pod-name>
```

