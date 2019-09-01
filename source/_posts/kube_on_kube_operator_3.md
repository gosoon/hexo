---
title: kube-on-kube-operator 开发(三)
date: 2019-09-01 17:05:30
tags: ["operator","kube-on-kube"]
type: "kubernetes-operator"

---


- [kube-on-kube-operator 开发(一)](http://blog.tianfeiyu.com/2019/08/05/kube_on_kube_operator_1/)

- [kube-on-kube-operator 开发(二)](http://blog.tianfeiyu.com/2019/08/07/kube_on_kube_operator_2/)



本文是介绍 kubernetes-operator 开发的第三篇，前几篇已经提到过 kubernetes-operator 的主要目标是实现以下三种场景中的集群管理：

- kube-on-kube
- kube-to-kube
- kube-to-cloud-kube

目前笔者主要在开发 kube-to-kube，这一节会介绍 kube-to-kube 中如何使用二进制方式部署一个集群，问什么要先支持部署二进制集群呢，可以参考之前的文章。目前 kubernetes-operator 中部署集群是通过 ansible 调用笔者写的一些脚本部署的，由于 kubernetes 二进制文件比较大，暂时仅支持离线部署，部署前请下载好所需的二进制文件，笔者也提供了部署 v1.14 需要的所有二进制文件、镜像、yaml 等。



二进制安装 kubernetes 最困难的地方就在于其复杂的认证(Authentication)及鉴权(Authorization)机制，上篇文章已经介绍了 kubernetes 中的认证与鉴权机制以及其中的证书链，若安装过程中有疑问请参考  [浅析 kubernetes 的认证与鉴权机制](http://blog.tianfeiyu.com/2019/08/18/k8s_auth_rbac/)。



使用 kubernetes-operator 管理集群时首选需要有一个元集群，元集群可以使用 minkube 或者 kind 部署一个单机版集群，然后将 kubernetes-operator 部署到该集群中再通过创建 CR 来部署一个业务集群，最后使用该业务集群作为元集群即可，或者也可以使用 kubernetes-operator 中部署业务集群的方式来部署元集群。



> 部署集群前请先克隆 https://github.com/gosoon/kubernetes-operator 和 https://github.com/gosoon/kubernetes-utils 项目，部署集群所需要的一些工具、配置以及 bin 文件都存放在这两个项目中，你也可以使用自己的配置。

### 准备环境

禁用防火墙：

```
$ systemctl stop firewalld
$ systemctl disable firewalld
```

禁用 SELinux：

```
$ setenforce 0
$ sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

关闭 swap：

```
swapoff -a
```

修改内核参数：

```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```



### 配置 CA 及创建 TLS 证书

安装证书生成工具，本文使用 cfssl

```
$ cp kubernetes-utils/scripts/bin/certs/* /usr/bin/
```

#### etcd

```
$ cat << EOF > etcd-root-ca-csr.json
{
    "CN": "etcd-root-ca",
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "O": "etcd",
            "OU": "etcd Security",
            "L": "Beijing",
            "ST": "Beijing",
            "C": "CN"
        }
    ],
    "ca": {
        "expiry": "87600h"
    }
}
EOF

$ cat << EOF > etcd-gencert.json
{
  "signing": {
    "default": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "87600h"
    }
  }
}
EOF

$ cat << EOF > etcd-csr.json
{
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "O": "etcd",
            "OU": "etcd Security",
            "L": "Beijing",
            "ST": "Beijing",
            "C": "CN"
        }
    ],
    "CN": "etcd"
}
EOF

$ cat << EOF > config-etcd-peer.json
{
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "O": "etcd",
            "OU": "etcd Security",
            "L": "Beijing",
            "ST": "Beijing",
            "C": "CN"
        }
    ],
    "CN": "etcd"
}
EOF
```



```
$ cfssl gencert --initca=true etcd-root-ca-csr.json | cfssljson -bare output/ca

// 指定 etcd hosts，etcd server 和 etcd peer 中必须包含所有 etcd 的 hosts，eg：ETCD_HOSTS="10.0.4.15，10.0.2.15"
# etcd server
$ cfssl gencert \
  -ca=output/ca.pem \
  -ca-key=output/ca-key.pem \
  -config=ca-config.json \
  -hostname=127.0.0.1,${ETCD_HOSTS} \
  -profile=server \
  server.json | cfssljson -bare output/etcd-server
  
# etcd peer
$ cfssl gencert \
  -ca=output/ca.pem \
  -ca-key=output/ca-key.pem \
  -config=ca-config.json \
  -hostname=127.0.0.1,${ETCD_HOSTS} \
  -profile=peer \
  server.json | cfssljson -bare output/etcd-peer  
```

生成证书后反解 etcd server 和 peer 证书校验 ip 是否正确：

```
$ cfssl certinfo -cert etcd-peer.pem
```

#### master

由于 master 组件的 CSR 配置与 kubernetes 中的认证与鉴权相关联，需要严格按照 kubernetes 中默认的 RBAC 进行配置，每个组件都有默认的 user 或者 group。

```
$ cat << EOF > ca-csr.json
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "China",
      "L": "Shanghai",
      "O": "Kubernetes",
      "OU": "Shanghai",
      "ST": "Shanghai"
    }
  ]
}
EOF

$ cat << EOF > ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "87600h"
      }
    }
  }
}
EOF


// kube-apiserver csr
$ cat << EOF > kube-apiserver-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "China",
      "L": "Shanghai",
      "O": "Kubernetes",
      "OU": "Kubernetes",
      "ST": "Shanghai"
    }
  ]
}
EOF

// kube-controller-manager csr
$ cat << EOF > kube-controller-manager-csr.json
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "China",
      "L": "Shanghai",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes",
      "ST": "Shanghai"
    }
  ]
}
EOF

// kube-scheduler csr
$ cat << EOF > kube-scheduler-csr.json
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "China",
      "L": "Shanghai",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes",
      "ST": "Shanghai"
    }
  ]
}
EOF

// kubelet csr，请替换 nodeName
$ cat << EOF > kubelet-csr.json
{
  "CN": "system:node:<nodeName>",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "China",
      "L": "Shanghai",
      "O": "system:nodes",
      "OU": "Kubernetes",
      "ST": "Shanghai"
    }
  ]
}
EOF

// apiserver client csr
$ cat << EOF > apiserver-kubelet-client-csr.json
{
  "CN": "system:kubelet-api-admin",
  "key": {
      "algo": "rsa",
      "size": 2048
  },
  "names": [
    {
      "C": "China",
      "L": "Shanghai",
      "O": "system:masters",
      "OU": "Kubernetes",
      "ST": "Shanghai"
    }
  ]
}
EOF

// kube-proxy csr
$ cat << EOF >  kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "China",
      "L": "Shanghai",
      "O": "system:node-proxier",
      "OU": "Kubernetes",
      "ST": "Shanghai"
    }
  ]
}
EOF

// kubectl csr
$ cat << EOF >  admin-csr.json
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "China",
      "L": "Shanghai",
      "O": "system:masters",
      "OU": "Kubernetes",
      "ST": "Shanghai"
    }
  ]
}
EOF


$ cfssl gencert -initca ca-csr.json | cfssljson -bare output/ca

为了保证客户端与 Kubernetes API 的认证，Kubernetes API Server 凭证中必需包含 master 的静态 IP 地址,在 hostname 中指定
# apiserver
$ cfssl gencert \
  -ca=output/ca.pem \
  -ca-key=output/ca-key.pem \
  -config=ca-config.json \
  -hostname=10.250.0.1,${MASTER_HOSTS},${MASTER_VIP},127.0.0.1,kubernetes,kubernetes.default,kubernetes.default.svc \
  -profile=kubernetes \
  kube-apiserver-csr.json | cfssljson -bare output/kube-apiserver

# kubelet
for node in `echo ${NODE_HOSTS} | tr ',' ' '`;do
    cfssl gencert \
      -ca=output/ca.pem \
      -ca-key=output/ca-key.pem \
      -config=ca-config.json \
      -hostname=${NODE_HOSTS} \
      -profile=kubernetes \
      kubelet-csr.json | cfssljson -bare output/kubelet
done

# other component
for component in kube-controller-manager kube-scheduler kube-proxy apiserver-kubelet-client admin service-account;do
    cfssl gencert \
      -ca=output/ca.pem \
      -ca-key=output/ca-key.pem \
      -config=ca-config.json \
      -profile=kubernetes \
      ${component}-csr.json | cfssljson -bare output/${component}
done
```

### 生成 kubeconfig 

```
// 替换 apiserver 
KUBE_APISERVER="https://10.0.4.15:6443"
CERTS_DIR="/etc/kubernetes/ssl"

# 生成 kubectl 配置文件
echo "Create kubectl kubeconfig..."
kubectl config set-cluster kubernetes \
  --certificate-authority=${CERTS_DIR}/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=output/kubectl.kubeconfig
kubectl config set-credentials "system:masters" \
  --client-certificate=${CERTS_DIR}/admin.pem \
  --client-key=${CERTS_DIR}/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=output/kubectl.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=system:masters \
  --kubeconfig=output/kubectl.kubeconfig
kubectl config use-context default --kubeconfig=output/kubectl.kubeconfig

# 生成 kube-controller-manager 配置文件
echo "Create kube-controller-manager kubeconfig..."
kubectl config set-cluster kubernetes \
  --certificate-authority=${CERTS_DIR}/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=output/kube-controller-manager.kubeconfig
kubectl config set-credentials "system:kube-controller-manager" \
  --client-certificate=${CERTS_DIR}/kube-controller-manager.pem \
  --client-key=${CERTS_DIR}/kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=output/kube-controller-manager.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=system:kube-controller-manager \
  --kubeconfig=output/kube-controller-manager.kubeconfig
kubectl config use-context default --kubeconfig=output/kube-controller-manager.kubeconfig

# 生成 kube-scheduler 配置文件
echo "Create kube-scheduler kubeconfig..."
kubectl config set-cluster kubernetes \
  --certificate-authority=${CERTS_DIR}/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=output/kube-scheduler.kubeconfig
kubectl config set-credentials "system:kube-scheduler" \
  --client-certificate=${CERTS_DIR}/kube-scheduler.pem \
  --client-key=${CERTS_DIR}/kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=output/kube-scheduler.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=system:kube-scheduler \
  --kubeconfig=output/kube-scheduler.kubeconfig
kubectl config use-context default --kubeconfig=output/kube-scheduler.kubeconfig


# 生成 kubelet 配置文件,需要添加对应的 nodeName
echo "Create kubelet kubeconfig..."
kubectl config set-cluster kubernetes \
  --certificate-authority=${CERTS_DIR}/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=output/kubelet-${node}.kubeconfig

kubectl config set-credentials system:node:${node} \
  --client-certificate=${CERTS_DIR}/kubelet.pem \
  --client-key=${CERTS_DIR}/kubelet-key.pem \
  --embed-certs=true \
  --kubeconfig=output/kubelet-${node}.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=system:node:${node} \
  --kubeconfig=${CERTS_DIR}/kubelet-${node}.kubeconfig
kubectl config use-context default --kubeconfig=output/kubelet-${node}.kubeconfig


# 生成 kube-proxy 配置文件
echo "Create kube-proxy kubeconfig..."
kubectl config set-cluster kubernetes \
  --certificate-authority=${CERTS_DIR}/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=output/kube-proxy.kubeconfig
kubectl config set-credentials "system:kube-proxy" \
  --client-certificate=${CERTS_DIR}/kube-proxy.pem \
  --client-key=${CERTS_DIR}/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=output/kube-proxy.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=system:kube-proxy \
  --kubeconfig=output/kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=output/kube-proxy.kubeconfig
```



### 部署 

#### 部署 etcd

拷贝证书文件：

```
$ cp output/* /etc/etcd/ssl/
```

拷贝 bin 文件：

```
$ cp kubernetes-utils/scripts/bin/etcd_v3.3.13/* /usr/bin/
```

拷贝配置文件，配置文件中的 ip 需要手动替换掉：

```
$ cp kubernetes-operator/scripts/config/etcd/etcd.conf /etc/etcd/
```

#### 部署 k8s master 组件

拷贝证书文件：

```
$ cp output/* /etc/kubernetes/ssl/
```

拷贝 bin 文件：

```
$ cp kubernetes-utils/scripts/bin/kubernetes_v1.14.0/* /usr/bin/
```

拷贝配置文件，配置文件中的 ip 需要手动替换掉：

```
$ cp kubernetes-operator/scripts/config/master/* /etc/kubernetes/
```

#### 部署 k8s node 组件

部署 docker，拷贝 bin 文件：

```
$ cp kubernetes-utils/scripts/bin/docker-ce-18.06.1.ce/*  /usr/bin/
```

拷贝证书文件：

```
$ cp output/* /etc/kubernetes/ssl/
```

拷贝配置文件，配置文件中的 ip 需要手动替换掉：

```
$ cp kubernetes-operator/scripts/config/node/* /etc/kubernetes/
```

#### 创建 systemd 文件

拷贝所有服务的 systemd 文件：

拷贝配置文件，配置文件中的 ip 需要手动替换掉：

```
$ cp kubernetes-operator/scripts/systemd/* /usr/lib/systemd/system/
```



### 启动服务 

首先启动 etcd 服务，etcd 所部署的几个节点需要同时启动，否则服务会启动失败。

然后依次启动 master 上的组件和 node 上的组件。

### 总结

本文主要讲述了 kubernetes-operator 中 kube-to-kube 部署集群的方式，介绍了主要的部署步骤，文中部署集群所有的操作都提供了脚本的方式：https://github.com/gosoon/kubernetes-operator/tree/master/scripts。 kube-to-kube 的部署方式暂时是以 ansible + 自定义脚本的方式部署，部署方式也在持续更新与完善中。接下来会继续开发 kube-on-kube 的部署方式，kube-on-kube 会将业务集群的 master 组件部署在元集群中，kube-on-kube 方式暂时会采用对 kubeadm 封装的形式进行部署。



