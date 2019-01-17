---
title: kubeadm 安装 kubernetes
date: 2019-01-17 10:11:30
tags: "kubeadm"
type: "kubeadm"

---

kubeadm 是 Kubernetes 主推的部署工具之一，正在快速迭代开发中，当前版本为 GA，暂不建议用于部署生产环境，其先进的设计理念可以借鉴。

## 一、kubeadm 原理介绍

kubeadm 会在初始化的机器上首先部署 kubelet 服务，kubelet 创建 pod 的方式有三种，其中一种就是监控指定目下（/etc/kubernetes/manifests）容器状态的变化然后进行相应的操作。kubeadm 启动 kubelet 后会在 /etc/kubernetes/manifests 目录下创建出 etcd、kube-apiserver、kube-controller-manager、kube-scheduler 四个组件 static pod 的 yaml 文件，此时 kubelet 监测到该目录下有 yaml 文件便会将其创建为对应的 pod，最终 kube-apiserver、kube-controller-manager、kube-scheduler 以及 etcd 会以 static pod 的方式运行。


> 本次安装 kubernetes 版本：v1.12.0

当前宿主机系统与内核版本：
```
$ uname -r
3.10.0-514.16.1.el7.x86_64

$ cat /etc/redhat-release
CentOS Linux release 7.2.1511 (Core)
```
## 二、安装前的准备工作
```
# 关闭swap
$ sudo swapoff -a

# 关闭selinux
$ sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux 
$ setenforce 0

# 关闭防火墙
$ systemctl disable firewalld.service && systemctl stop firewalld.service

# 配置转发相关参数
$ cat << EOF >> /etc/sysctl.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
EOF 
$ sysctl -p
```

## 三、安装 Docker CE 
 
> 本次安装的 docker 版本：docker-ce-18.06.1.ce

```
# Install Docker CE
## Set up the repository
### Install required packages.
yum install yum-utils device-mapper-persistent-data lvm2

### Add docker repository.
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

## Install docker ce.
yum update && yum install docker-ce-18.06.1.ce

## Create /etc/docker directory.
mkdir /etc/docker

# Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart docker.
systemctl daemon-reload
systemctl restart docker
```
参考：https://kubernetes.io/docs/setup/cri/

## 四、安装 kubernetes master 组件

使用 kubeadm 初始化集群：
```
$ kubeadm init --kubernetes-version=v1.12.0 --pod-network-cidr=10.244.0.0/16
[init] using Kubernetes version: v1.12.0
[preflight] running pre-flight checks
[preflight/images] Pulling images required for setting up a Kubernetes cluster
[preflight/images] This might take a minute or two, depending on the speed of your internet connection
[preflight/images] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[preflight] Activating the kubelet service
[certificates] Using the existing front-proxy-client certificate and key.
[certificates] Using the existing etcd/server certificate and key.
[certificates] Using the existing etcd/peer certificate and key.
[certificates] Using the existing etcd/healthcheck-client certificate and key.
[certificates] Using the existing apiserver-etcd-client certificate and key.
[certificates] Using the existing apiserver certificate and key.
[certificates] Using the existing apiserver-kubelet-client certificate and key.
[certificates] valid certificates and keys now exist in "/etc/kubernetes/pki"
[certificates] Using the existing sa key.
[kubeconfig] Using existing up-to-date KubeConfig file: "/etc/kubernetes/admin.conf"
[kubeconfig] Using existing up-to-date KubeConfig file: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Using existing up-to-date KubeConfig file: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Using existing up-to-date KubeConfig file: "/etc/kubernetes/scheduler.conf"
[controlplane] wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests"
[init] this might take a minute or longer if the control plane images have to be pulled
[apiclient] All control plane components are healthy after 14.002350 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.12" in namespace kube-system with the configuration for the kubelets in the cluster
[markmaster] Marking the node 192.168.1.110 as master by adding the label "node-role.kubernetes.io/master=''"
[markmaster] Marking the node 192.168.1.110 as master by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "192.168.1.110" as an annotation
[bootstraptoken] using token: wu5hfy.lkuz9fih6hlqe1jt
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.1.110:6443 --token wu5hfy.lkuz9fih6hlqe1jt --discovery-token-ca-cert-hash sha256:e8d2649fceae9d7f6de94af0b7e294680b87f7d1e207c75c3cb496841b12ec23

```

这个命令会自动执行以下步骤：

- 系统状态检查
- 生成 token
- 生成自签名 CA 和 client 端证书
- 生成 kubeconfig 用于 kubelet 连接 API server
- 为 Master 组件生成 Static Pod manifests，并放到 /etc/kubernetes/manifests 目录中
- 配置 RBAC 并设置 Master node 只运行控制平面组件
- 创建附加服务，比如 kube-proxy 和 CoreDNS


配置 kubetl 认证信息： 
```
 $ mkdir -p $HOME/.kube
 $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

将本机作为 node 加入到 master 中：
```
$ kubeadm join 192.168.1.110:6443 --token wu5hfy.lkuz9fih6hlqe1jt --discovery-token-ca-cert-hash
```
kubeadm 默认 master 节点不作为 node 节点使用，初始化完成后会给 master 节点打上 taint 标签，若单机部署，使用以下命令去掉 taint 标签：
```
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

查看各组件是否正常运行：

```
$ kubectl get pod -n kube-system
NAME                                     READY   STATUS             RESTARTS   AGE
coredns-99b9bb8bd-pgh5t                  1/1     Running            0          48m
etcd                                     1/1     Running            2          48m
kube-apiserver                           1/1     Running            1          48m
kube-controller-manager                  1/1     Running            0          49m
kube-flannel-ds-amd64-b5rjg              1/1     Running            0          31m
kube-proxy-c8ktg                         1/1     Running            0          48m
kube-scheduler                           1/1     Running            2          48m
```

## 五、安装 kubernetes 网络

kubernetes 本身是不提供网络方案的，但是有很多开源组件可以帮助我们打通容器和容器之间的网络，实现 Kubernetes 要求的网络模型。从实现原理上来说大致分为以下两种：
- overlay 网络，通过封包解包的方式构造一个隧道，代表的方案有 flannel(udp/vxlan）、weave、calico(ipip)，openvswitch 等
- 通过路由来实现(更改 iptables 等手段)，flannel(host-gw)，calico(bgp)，macvlan 等

当然每种方案都有自己适合的场景，flannel 和 calico 是两种最常见的网络方案，我们要根据自己的实际需要进行选择。此次安装选择 flannel 网络：


此操作也会为 flannel 创建对应的 RBAC 规则，flannel 会以 daemonset 的方式创建出来：
```
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

创建一个 pod 验证集群是否正常：

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

## 六、kubeadm 其他相关的操作

1、删除安装:
```
$ kubeadm reset
```
2、版本升级
```
# 查看可升级的版本
$ kubeadm upgrade plan

# 升级至指定版本
$ kubeadm upgrade apply [version]
```
>  1. 要执行升级，需要先将 kubeadm 升级到对应的版本；
>  2. kubeadm 并不负责 kubelet 的升级，需要在升级完 master 组件后，手工对 kubelet 进行升级。


## 七、创建过程中的一些 case 记录

##### 1、flannel 容器启动报错：pod cidr not assgned

需要在  /etc/kubernetes/manifests/kube-controller-manager.yaml 文件中添加以下配置：

--allocate-node-cidrs=true
--cluster-cidr=10.244.0.0/16

参考：https://github.com/coreos/flannel/issues/728

##### 2、coredns 容器启动失败报错：/proc/sys/net/ipv6/conf/eth0/accept_dad: no such file or directory

```
$ vim /etc/default/grub and change the value of kernel parameter ipv6.disable from 1 to 0 in line
$ grub2-mkconfig -o /boot/grub2/grub.cfg
$ shutdown -r now
```
参考：https://github.com/containernetworking/cni/issues/569

##### 3、kubeadm 证书有效期问题

默认情况下，kubeadm 会生成集群运行所需的所有证书，我们也可以通过提供自己的证书来覆盖此行为。要做到这一点，必须把它们放在 --cert-dir 参数或者配置文件中的 CertificatesDir 指定的目录（默认目录为 /etc/kubernetes/pki），如果存在一个给定的证书和密钥对，kubeadm 将会跳过生成步骤并且使用已存在的文件。例如，可以拷贝一个已有的 CA 到 /etc/kubernetes/pki/ca.crt 和 /etc/kubernetes/pki/ca.key，kubeadm 将会使用这个 CA 来签署其余的证书。所以只要我们自己提供一个有效期很长的证书去覆盖掉默认的证书就可以来避免这个的问题。

##### 4、kubeadm join 时 token 无法生效

token 的失效为24小时，若忘记或者 token 过期可以使用 `kubeadm token create` 重新生成 token。

## 八、总结
本篇文章讲述了使用 kubeadm 来搭建一个 kubernetes 集群，kubeadm 暂时还不建议用于生产环境，若部署生产环境请使用二进制文件。kubeadm 搭建出的集群还是有很多不完善的地方，比如，集群 master 组件的参数配置问题，官方默认的并不会满足需求，有许多参数需要根据实际情况进行修改。


参考：
[Creating a single master cluster with kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
[kubeadm 工作原理](https://github.com/feiskyer/kubernetes-handbook/blob/master/components/kubeadm.md)
[DockOne微信分享（一六三）：Kubernetes官方集群部署工具kubeadm原理解析](http://dockone.io/article/4645)
[centos7.2 安装k8s v1.11.0](https://segmentfault.com/a/1190000015787725)

