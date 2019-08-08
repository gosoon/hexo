---
title: kube-on-kube-operator 开发(二)
date: 2019-08-07 17:47:30
tags: ["operator","kube-on-kube"]
type: "kubernetes-operator"

---

本文主要讲述 kubernetes-operator 的开发过程，kubernetes-operator 已经开发了一个多月，其核心功能已经实现，其中的架构以及功能设计主要来自于一些生产环境的经验以及自己从事 kubernetes 运维开发两年多的一些工作经验，如有问题望指正。


### kubernetes-operator 组件介绍

kubernetes-operator 中主要包含一个自定义的 controller 和一个 HTTP Server，如下图所示，controller 主要是监听 CRD 的变化以及使其达到终态，HTTP Server 提供了多个 RESTful API，用于操作 CRD(创建、删除、扩缩容、接收回调等)。

![](http://cdn.tianfeiyu.com/operator-2.png)



除此之外还有其他的组件，ansibleinit、precheck、admission-webhook，ansibleinit 是一个二进制文件用来作为容器内的 1 号进程，会调用 ansible 相关的命令以及处理信号、子进程收割等。precheck 主要用于在对集群操作前检查目标宿主机的环境，由于对集群的操作需要耗费数十秒，为了保证成功率需要在部署前检查宿主的环境。admission-webhook 暂时用于校验 CR 中字段，比如集群执行扩容操作时，master 等字段的值肯定是不能改变的。


### kubernetes-operator 的开发

下面主要讲 kubernetes-operator 中核心组件的开发，主要有以下几步：

- 定义 CRD
- 生成代码
- 开发 controller
- 开发 RESTful API

#### 定义 CRD

下面是 CRD 的定义，kubernetes-operator 中的自定义资源为 `KubernetesCluster`，项目中简称为 `ecs`。

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: kubernetesclusters.ecs.yun.com
spec:
  group: ecs.yun.com
  names:
    kind: KubernetesCluster
    listKind: KubernetesClusterList
    plural: kubernetesclusters
    singular: kubernetescluster
    shortNames:
    - ecs
  scope: Namespaced
  subresources:
    status: {}
  version: v1
  versions:
  - name: v1
    served: true
    storage: true
```

将 CRD 部署到 kubernetes 集群中，CRD 中的自定义资源`KubernetesCluster`(CR) 就成为了 kubernetes 中的一种资源，和 pod、deployment 等类似。


#### 生成代码

生成代码可以参考上一篇文章[使用 code-generator 为 CustomResources 生成代码](http://blog.tianfeiyu.com/2019/08/06/code_generator/)，此处不再详解。



#### 开发 controller

如下所示是 controller 最简单的一个声明：

```
for {
  desired := getDesiredState()
  current := getCurrentState()
  makeChanges(desired, current)
}
```

所有 controller 也都是以此进行演变的，controller 的代码模式或者套路可以参考[sample-controller](https://github.com/kubernetes/sample-controller) 或者 kube-controller-manager 中所有 [controller](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller) 的实现。



下面是 kubernetes-operator 中 controller 实现的一个流程图：

![](http://cdn.tianfeiyu.com/operator-3.png)


更新 CR 都是客户端的操作，所以在设计时客户端都是操作 annotation 中的字段，然后 operator 监听到相关的时间后会进行处理。例如，当用户要创建一个集群时，首先客户端将 `app.kubernetes.io/operation`设置为 **creating**，此时 operator watch 到 CR 变化后会处理新建集群的操作，operator 会创建一个用来部署集群的 job，以及创建 configmap 来保存本次的操作记录以及关联对应的 job，也能用来查询本次操作的日志，然后会更新 CR 中 status.phase 中的 **Creating** (新创建的 CR status.phase 为 "")，接下来为 CR 设置 finalizers，最后会启动一个 goroutine 检测 job 的状态。此时需要等待 job 的完成以及回调，若 job 失败或者超时都会被最后启动的 goroutine 检测到，job 成功与否都会触发更新 CR status.phase 的操作。若 job 执行完成成功回调，客户端会更新  `app.kubernetes.io/operation `为 **create-finished**，客户端更新完成后会触发一次事件，然后 operator 会将 status.phase 更新为 **Running** 状态，否则 job 异常 operator 会直接更新 status.phase 为 **Failed**。


关于 CR 中  `app.kubernetes.io/operation` 字段以及 status.phase 中所有的定义请参见 [kubernetes-operator/pkg/enum/task.go](https://github.com/gosoon/kubernetes-operator/blob/master/pkg/enum/task.go)。


#### 开发 RESTful API

在前后端分离的场景中，RESTful API 的开发仅需要一个 route 框架即可，kubernetes-operator 中用的是  mux，具体的代码在 [kubernetes-operator/pkg/server](https://github.com/gosoon/kubernetes-operator/tree/master/pkg/server) 下。


### 总结

本文主要讲述了 kubernetes-operator 中主要的模块以及 controller 的具体实现，其中许多细节暂未提及到，详细的实现请参考代码，目前项目只是笔者利用业余时间进行开发的，毕竟个人精力有限，在阅读文章或者代码的过程中如有问题可以随时留言，笔者会持续迭代版本。下一篇文章会讲述如何使用二进制文件部署 kubernetes 集群。


参考：

https://github.com/kubernetes/community/blob/8decfe4/contributors/devel/controllers.md  

https://github.com/kubernetes/sample-controller  

https://engineering.bitnami.com/articles/a-deep-dive-into-kubernetes-controllers.html  

https://www.cnblogs.com/gaorong/p/8854934.html

https://yucs.github.io/2017/12/21/2017-12-21-operator/
