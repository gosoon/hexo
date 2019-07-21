---
title: 部署 kubernetes 可视化监控组件
date: 2019-04-22 22:43:30
tags: ["kube-dashboard","prometheus"]
type: "kubernetes"

---

随着 kubernetes 的大规模使用，对 kubernetes 组件及其上运行服务的监控也是非常重要的一个环节，目前开源的监控组件有很多种，例如 cAdvisor、Heapster、metrics-server、kube-state-metrics、Prometheus 等，对监控数据的可视化查看组件有 Dashboard、 Prometheus、Grafana 等，本文会介绍 kube-dashboard 和基于 prometheus 搭建数据可视化监控。

> kubernetes 版本：v1.12

### 一、kubernetes-dashboard 的部署

#### 1、创建 kubernetes-dashboard

```golang
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

$ kubectl get svc -n kube-system | grep kubernetes-dashboard
kubernetes-dashboard   ClusterIP    10.101.203.44   <none>        443/TCP    2h

$ kubectl get pod -n kube-system | grep kubernetes-dashboard
kubernetes-dashboard-65c76f6c97-8npsv    1/1     Running       0          2h
```

> 所需镜像下载地址：[k8s-system-images](https://github.com/gosoon/k8s-system-images)

#### 2、使用 nodePort 方式访问 kubernetes-dashboard

nodeport 的访问方式虽然有性能损失但是比较简单，kubernetes-dashboard 默认使用 clusterIP 的方式暴露服务，修改 kubernetes-dashboard svc 使用 nodePort 方式：

```
$ kubectl edit svc -n kube-system
	...
  spec:
    clusterIP: 10.101.203.44
    externalTrafficPolicy: Cluster
    ports:
    - nodePort: 8004  // 添加 nodeport 端口
      port: 443
      protocol: TCP
      targetPort: 8443
    selector:
      k8s-app: kubernetes-dashboard
    sessionAffinity: None
    type: NodePort   // 将 ClusterIP 修改为 NodePort
    ...
```

nodePort 端口默认为 30000-32767，若使用其他端口，需要修改 apiserver 的启动参数 `--service-node-port-range` 来指定 nodePort 范围，如：`--service-node-port-range 8000-9000`。

#### 3、创建 kubernetes-dashboard 管理员角色

 `kubernetes-dashboard-admin.yaml`：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard-admin
subjects:
  - kind: ServiceAccount
    name: dashboard-admin
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

创建角色并获取 token：

```
$ kubectl apply -f kubernetes-dashboard-admin.yaml

$ kubectl describe secrets `kubectl get secret -n kube-system | grep dashboard-admin | awk '{print $1}'` -n kube-system

Name:         dashboard-admin-token-hrhfd
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: 76805bdb-6047-11e9-ba0d-525400c322d9

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4taHJoZmQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNzY4MDViZGItNjA0Ny0xMWU5LWJhMGQtNTI1NDAwYzMyMmQ5Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.hJJRyp_O4sGvIULj3BhqidCkkPnD4A2AtnpkXJoEPCALaaQHC8zhCA5-nDNlo2fiEggZ02UZPwiyGxKKFPC57UlKhjTf5zYcMIhELVXlj5FdBmjzCZcCHVFF4tj_rCoOFlZi6fQ3vNCcX8CtLxX_OsH1YXaFVuUmR1gYm97hbyuO382_k3tFIPXFP3QG8zUtc_7QMkeMNEakJZLCvkW8xdlaCuC-GVAMhZl5Kq1MSthuF-8HY7KaXhvqQzfD4DQZrdQ7vf_7NG3rdvhsj8nQ__TTe1W0RjqwkQuxg5YdE4gbAsxwJjkek-N0K9HfnZhkS9WosaUaUe9pZaGZ9akqyQ
```

token 是访问 dashboard 需要用的。


若没有安装 kube-proxy，可以参考官方提供使用 `kubectl proxy` 的方式访问：

```
$ kubectl proxy --address=IP --disable-filter=true
```

访问 http://IP:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login

已部署 kube-proxy 的可直接访问 https://IP:nodePort 

![](http://cdn.tianfeiyu.com/dash-1.png)

选择令牌方式使用上面生成的 token 登录。

![](http://cdn.tianfeiyu.com/dash-2.png)

Dashboard 可以使用 Ingress、Let's Encrypt 等多种方式配置 ssl，关于 ssl 的详细配置此处不进行详解。


### 二、部署 prometheus 

prometheus 作为 CNCF 生态圈中的重要一员，其活跃度仅次于 Kubernetes, 现已广泛用于 Kubernetes 集群的监控系统中。prometheus 的部署相对比较简单，社区已经有了 [kube-prometheus](<https://github.com/coreos/kube-prometheus>)，kube-prometheus 会部署包含 prometheus-operator、grafana、kube-state-metrics 等多个组件。

```
$ git clone https://github.com/coreos/kube-prometheus

$ kubectl apply -f manifests/
```

为了使用简单，我也会将 prometheus 和 grafana 的端口修改为 nodePort 的方式进行暴露：

```
$ kubectl edit svc prometheus-k8s -n monitoring

$ kubectl edit svc grafana -n monitoring

$ kubectl get svc -n monitoring
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
alertmanager-main       NodePort    10.102.81.118   <none>        9093:8007/TCP       5d1h
alertmanager-operated   ClusterIP   None            <none>        9093/TCP,6783/TCP   5d1h
grafana                 NodePort    10.96.19.82     <none>        3000:8006/TCP       5d1h
kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP   5d1h
node-exporter           ClusterIP   None            <none>        9100/TCP            5d1h
prometheus-adapter      ClusterIP   10.107.103.58   <none>        443/TCP             5d1h
prometheus-k8s          NodePort    10.110.222.41   <none>        9090:8005/TCP       5d1h
prometheus-operated     ClusterIP   None            <none>        9090/TCP            5d1h
prometheus-operator     ClusterIP   None            <none>        8080/TCP            5d1h

$ kubectl get pod -n monitoring
NAME                                   READY   STATUS    RESTARTS   AGE
alertmanager-main-0                    2/2     Running   0          4d
alertmanager-main-1                    2/2     Running   0          4d
alertmanager-main-2                    2/2     Running   0          4d
grafana-9d97dfdc7-qfjts                1/1     Running   0          4d
kube-state-metrics-74d7dcd7dc-qfz5m    4/4     Running   0          3d11h
node-exporter-5cdl2                    2/2     Running   0          4d
prometheus-adapter-b7d894c9c-dvzzq     1/1     Running   0          4d
prometheus-k8s-0                       3/3     Running   1          2d2h
prometheus-k8s-1                       3/3     Running   1          4d
prometheus-operator-77b8b97459-7qfxj   1/1     Running   0          4d
```

上面几个组件成功运行后就可以在页面访问 prometheus 和 ganfana ：

![](http://cdn.tianfeiyu.com/dash-3.png)

进入 grafana 的 web 端，默认用户名和密码均为 admin：

![](http://cdn.tianfeiyu.com/dash-4.png)

grafana 支持导入其他的 Dashboard，在 grafana 官方网站可以搜到大量与 k8s 相关的 dashboard。 

### 三、总结

本文介绍了对 kubernetes 和容器监控比较成熟的两个方案，虽然目前开源的方案比较多，但是要形成采集、存储、展示、报警一个完成的体系还需要在使用过程中不断探索与完善。
