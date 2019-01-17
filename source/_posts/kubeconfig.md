---
title: kubernetes 中 kubeconfig 的用法
date: 2019-01-09 19:28:30
tags: "kubeconfig"
type: "kubeconfig"

---

用于配置集群访问信息的文件叫作 kubeconfig 文件，在开启了 TLS 的集群中，每次与集群交互时都需要身份认证，生产环境一般使用证书进行认证，其认证所需要的信息会放在 kubeconfig 文件中。此外，k8s 的组件都可以使用 kubeconfig 连接 apiserver，[client-go ](https://github.com/kubernetes/client-go/blob/master/examples/create-update-delete-deployment/main.go)、operator、helm 等其他组件也使用 kubeconfig 访问 apiserver。

## 一、kubeconfig 配置文件的生成

kubeconfig 的一个示例：
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: xxx
    server: https://xxx:6443
  name: cluster1
- cluster:
    certificate-authority-data: xxx
    server: https://xxx:6443
  name: cluster2
contexts:
- context:
    cluster: cluster1
    user: kubelet
  name: cluster1-context
- context:
    cluster: cluster2
    user: kubelet
  name: cluster2-context
current-context: cluster1-context
kind: Config
preferences: {}
users:
- name: kubelet
  user:
    client-certificate-data: xxx
    client-key-data: xxx
```

apiVersion 和 kind 标识客户端解析器的版本和模式，不应手动编辑。 preferences 指定可选（和当前未使用）的 kubectl 首选项。

#### 1、clusters模块
cluster中包含 kubernetes 集群的端点数据，包括 kubernetes apiserver 的完整 url 以及集群的证书颁发机构。

可以使用 `kubectl config set-cluster` 添加或修改 cluster 条目。

#### 2、users 模块
user 定义用于向 kubernetes 集群进行身份验证的客户端凭据。

可用凭证有 `client-certificate、client-key、token 和 username/password`。 
`username/password` 和 `token` 是二者只能选择一个，但 `client-certificate` 和 `client-key` 可以分别与它们组合。

可以使用 `kubectl config set-credentials` 添加或者修改 user 条目。

#### 3、contexts 模块

context 定义了一个命名的`cluster、user、namespace`元组，用于使用提供的认证信息和命名空间将请求发送到指定的集群。

三个都是可选的；
仅使用 cluster、user、namespace 之一指定上下文，或指定`none`。 

未指定的值或在加载的 kubeconfig 中没有相应条目的命名值将被替换为默认值。
加载和合并 kubeconfig 文件的规则很简单，但有很多，具体可以查看[加载和合并kubeconfig规则](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)。

可以使用`kubectl config set-context`添加或修改上下文条目。

#### 4、current-context 模块

current-context 是作为`cluster、user、namespace`元组的 key，
当 kubectl 从该文件中加载配置的时候会被默认使用。

可以在 kubectl 命令行里覆盖这些值，通过分别传入`--context=CONTEXT、--cluster=CLUSTER、--user=USER 和 --namespace=NAMESPACE`。
以上示例中若不指定 context 则默认使用 cluster1-context。
```
kubectl  get node --kubeconfig=./kubeconfig --context=cluster2-context
```
可以使用 `kubectl config use-context` 更改 current-context。

#### 5、kubectl 生成 kubeconfig 的示例

kubectl 可以快速生成 kubeconfig，以下是一个示例：
```
$ kubectl config set-credentials myself --username=admin --password=secret
$ kubectl config set-cluster local-server --server=http://localhost:8080
$ kubectl config set-context default-context --cluster=local-server --user=myself
$ kubectl config use-context cluster-context
$ kubectl config set contexts.default-context.namespace the-right-prefix
$ kubectl config view
```

若使用手写 kubeconfig 的方式，推荐一个工具 [kubeval](https://github.com/garethr/kubeval)，可以校验 kubernetes yaml 或 json 格式的配置文件是否正确。

## 二、使用 kubeconfig 文件配置 kuebctl 跨集群认证

kubectl 作为操作 k8s 的一个客户端工具，只要为 kubectl 提供连接 apiserver 的配置(kubeconfig)，kubectl 可以在任何地方操作该集群，当然，若 kubeconfig 文件中配置多个集群，kubectl 也可以轻松地在多个集群之间切换。

kubectl 加载配置文件的顺序：
1、kubectl 默认连接本机的 8080 端口
2、从 $HOME/.kube 目录下查找文件名为 config 的文件
3、通过设置环境变量 KUBECONFIG 或者通过设置去指定其它 kubeconfig 文件
```
# 设置 KUBECONFIG 的环境变量
export KUBECONFIG=/etc/kubernetes/kubeconfig/kubelet.kubeconfig
# 指定 kubeconfig 文件
kubectl get node --kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig
# 使用不同的 context 在多个集群之间切换
kubectl  get node --kubeconfig=./kubeconfig --context=cluster1-context
```
开篇的示例就是多集群认证方式配置的一种。

参考：
https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/
https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/
