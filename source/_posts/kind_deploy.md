---
title: 使用 kind 部署单机版 kubernetes 集群
date: 2019-09-06 10:30:30
tags: ["kind","deploy"]
type: "kind"

---

kubernetes 从一发布开始其学习门槛就比较高，首先就是部署难，用户要想学习 kubernetes 必须要过部署这一关，社区也推出了多个部署工具帮助简化集群的部署，社区中推出的部署工具主要目标有两大类，部署测试环境与生产环境，本节主要讲述测试环境的部署，目前社区已经有多套部署方案了：

- https://github.com/bsycorp/kind
- https://github.com/ubuntu/microk8s
- https://github.com/kinvolk/kube-spawn
- https://github.com/kubernetes/minikube
- https://github.com/danderson/virtuakube
- https://github.com/kubernetes-sigs/kubeadm-dind-cluster

而本文主要讲述使用 [kind](https://github.com/kubernetes-sigs/kind)（Kubernetes In Docker）部署 k8s 集群，因为 kind 使用起来实在太简单了，特别适用于在本机部署测试环境。



kind 的原理就是将 k8s 所需要的所有组件，全部部署在一个 docker 容器中，只需要一个镜像即可部署一套 k8s 环境，其底层是使用 kubeadm 进行部署，CRI 使用 Containerd，CNI 使用 weave。下面就来看看如何使用 kind 部署一套 kubernetes 环境，在使用 kind 前你需要确保目标机器已经安装了 docker 服务。

### 一、使用 kind 部署 k8s 集群

> 以下安装环境为 mac os。

安装 kind ：

```
$ wget https://github.com/kubernetes-sigs/kind/releases/download/v0.5.1/kind-darwin-amd64
$ chmod +x kind-darwin-amd64
$ mv kind-darwin-amd64 /usr/local/bin/kind
```

使用 kind 部署 kubernetes 集群：

```
// 默认的 cluster name 为 kind，可以使用 --name 指定
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.15.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Creating kubeadm config 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Cluster creation complete. You can now use the cluster with:
```

使用 **kind create cluster** 安装，是没有指定任何配置文件的安装方式。从安装打印出的输出来看，分为 6 步：
1. 安装基础镜像 kindest/node:v1.15.4，这个镜像里面包含了所需要的二进制文件、配置文件以及 k8s 左右组件镜像的 tar 包
2. 准备 node，检查环境、启动镜像等工作
3. 生成 kubeadm 的配置，然后使用 kubeadm 安装，和直接使用 kubeadm 的步骤类似
4. 启动服务
5. 部署 CNI 插件，kind 默认使用 weave。
6. 创建 StorageClass。



```
// 查看 kubeconfig path
$ kind get kubeconfig-path
/Users/feiyu/.kube/kind-config-kind

$ export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
```

kind 还有多个子命令，此处不再一一详解。



```
// 查看集群信息，
$ kubectl cluster-info
Kubernetes master is running at https://127.0.0.1:55387
KubeDNS is running at https://127.0.0.1:55387/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.


// 查看本地的 kind 容器
$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                                  NAMES
e26545538cc7        kindest/node:v1.15.3   "/usr/local/bin/entr…"   15 minutes ago      Up 15 minutes       55387/tcp, 127.0.0.1:55387->6443/tcp   kind-control-plane
```

可以看到，kind 容器暴露的 6443 端口映射在本机的一个随机端口(55387)上。



```
// 查看 node 的详细信息，可以看到 cni 为 containerd
$ kubectl describe node kind-control-plane
...
 Container Runtime Version:  containerd://1.2.6-0ubuntu1
 Kubelet Version:            v1.15.3
 Kube-Proxy Version:         v1.15.3
PodCIDR:                     10.244.0.0/24
ExternalID:                  kind-control-plane
...


# 进入 kind 容器查看 k8s 的配置，和单独使用 kubeadm 时一致
$ docker exec -it e26545538cc  bash
root@kind-control-plane:~# ls /etc/kubernetes/
admin.conf  controller-manager.conf  kubelet.conf  manifests  pki  scheduler.conf
root@kind-control-plane:~# ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml


# 查看 cni 配置
root@kind-control-plane:/etc/kubernetes# cat /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--container-runtime=remote --container-runtime-endpoint=/run/containerd/containerd.sock --fail-swap-on=false --node-ip=172.17.0.2"


# 查看容器的状态
root@kind-control-plane:~# crictl pods
POD ID              CREATED             STATE               NAME                                         NAMESPACE           ATTEMPT
fc8700af77ca2       About an hour ago   Ready               coredns-5c98db65d4-bjxl2                     kube-system         0
6378297d32811       About an hour ago   Ready               coredns-5c98db65d4-q2drh                     kube-system         0
124b42a35e0d1       About an hour ago   Ready               kube-proxy-99nc9                             kube-system         0
54b9511069534       About an hour ago   Ready               kindnet-xz8dp                                kube-system         0
61cb720ddece8       About an hour ago   Ready               etcd-kind-control-plane                      kube-system         0
4514b98de1a44       About an hour ago   Ready               kube-scheduler-kind-control-plane            kube-system         0
9a29dbebc8dd1       About an hour ago   Ready               kube-controller-manager-kind-control-plane   kube-system         0
ab028c5f5a3e5       About an hour ago   Ready               kube-apiserver-kind-control-plane            kube-system         0
```

删除集群：

```
$ kind delete cluster
```

kind 也支持创建多 master 以及多 work 节点的集群，需要自定义 yaml 配置：

```
# a cluster with 3 control-plane nodes and 3 workers
kind: Cluster
apiVersion: kind.sigs.k8s.io/v1alpha3
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker

// 创建集群指定 config
$ kind create cluster --config kind.yaml
```



kind 还支持自定义映射的端口号、支持使用自定义镜像仓库、支持启用 Feature Gates 等多个功能，详细的使用请参考官方文档 [quick-start](https://kind.sigs.k8s.io/docs/user/quick-start/)。



### 二、本地测试

既然 kind 不能用作生产环境，那怎么在本地测试时使用呢？由于 k8s 的新版已经全面启用了 TLS，不再支持非安全端口，访问 APIServer 的接口都需要认证，但是本地测试不需要那么麻烦，如下所示，为匿名用户设置访问权限即可。

```
// 为匿名用户关联 RBAC 规则
$ kubectl create clusterrolebinding system:anonymous --clusterrole=cluster-admin --user=system:anonymous

// 请求相关的 API
$ curl -k https://127.0.0.1:55387/api/v1/nodes
{
  "kind": "NodeList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/nodes",
    "resourceVersion": "11844"
  },
  "items": [
  ...
```

