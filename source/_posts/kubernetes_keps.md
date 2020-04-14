---
title: kubernetes 中的增强特性(Kubernetes Enhancement Proposal)
date: 2020-04-13 20:44:30
tags: ["kubernetes",]
type: "kubernetes"

---



[kubernetes 增强特性](https://github.com/kubernetes/enhancements)(kep)是为了解决社区中的疑难问题而创建的一个项目，每一个增强特性都对 kubernetes 的部分功能有较大的影响，需要 kubernetes 项目下的多个组(SIG)协作开发，对应的特性通常要经过 `alpha`、`beta `以及 `GA` 三个版本，所以每个方案的开发周期比较长，大多需要经过 9~10 个月才能完成，某些特性甚至已经讨论多年至今仍未开发完成，像 crd、dry-run、kubectl diff、pid limit 等已经开发完成的功能都是在 kep 中提出来的。本文会介绍几个比较重要的已经在 kep 中孵化的特性。



### 1、client-go 中对 resource 的操作支持传递 context 参数 

该特性的目标：

- （1）支持请求超时以及取消请求的调用；
- （2）支持分布式追踪；

  

以下是新旧版本中用 client-go list deployment 方式的一个对比：

```
// 老版本中的使用方式
deploymentList, _ := clientset.AppsV1().Deployments(apiv1.NamespaceDefault).List( metav1.ListOptions{})

// 新版本中的使用方式
deploymentList, err := clientset.AppsV1().Deployments(apiv1.NamespaceDefault).List(context.TODO(), metav1.ListOptions{})
```



可以看到在新版本中 client-go 对于 resource 的操作(verbs)首个参数需要传入 context，当然，社区考虑到用户升级 client-go 代码库时需要对应大量的代码进行改动，kubernetes 社区会对 client-go 的老版本进行一个快照，快照将存在以下几个包中：

```
k8s.io/apiextensions-apiserver/pkg/client/{clientset => deprecated}
k8s.io/client-go/{kubernetes => deprecated}
k8s.io/kube-aggregator/pkg/client/clientset_generated/{clientset => deprecated}
k8s.io/metrics/pkg/client/{clientset => deprecated}
```



此次升级无论对于用户还是 kubernetes 社区中的项目无疑都需要非常大的变动，使用 client-go 新版本的用户可以使用 sed 等工具修改代码中的相关用法。对于 kubernetes 社区内部项目代码，所有调用中会使用 `context.TODO()` 作为初始值添加到对 resource 操作的首个参数中。



参考：[20200123-client-go-ctx.md](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20200123-client-go-ctx.md)



### 2、从 apiserver 的 watch cache 中进行一致性读取

该特性的目标：

1、解决过期数据问题(https://github.com/kubernetes/kubernetes/issues/59848)；
2、当 watch cache 启用后，提高对 resource get 和 list 操作的可扩展性以及性能问题；


从以上 [issue](https://github.com/kubernetes/kubernetes/issues/59848) 中可以看到其问题出现的场景为：
- 1、集群中存在多个 master 实例，node-1 与 node-2 首先都连接至 apiserver-1；
- 2、由 controller 管理的 pod-0 最初在 node-1 节点上运行，`T2` 时刻 pod-0 被删除后调度至 node-2 节点，然后 node-2 节点启动了 pod-0；
- 3、pod-0 在 node-2 上启动的同时 node-1 节点因异常导致 kubelet 重新启动，此时 node-1 上的 kubelet 连接到了 apiserver-2 上，但 apiserver-2 此时的 watch cache 正好延迟于 `T2` 时刻(因 apiserver-2 网络或者性能问题导致数据延迟)，apiserver 会将自己的  delay cache 中的 pod list 发送给 node-1，此时 node-1 也会启动一个 pod-0，而 node-1 上面的 pod-0 已经处于运行状态；

kubelet 通过 apiserver list 数据时默认将 `resourceVersion` 设置为 0，此时返回的数据是 apiserver watch cache 中的，并非直接读取 etcd 而来，而因网络或其他原因此时 etcd 与 apiserver watch cache 中的数据可能不同。也就是说，在使用 list/get 时设置 `resourceVersion` 为 0 可能会获取到过期的数据，当然以上问题会出现在所有的 controller 中。众所周知，`resourceVersion` 有三种设置方法，第一种当不设置时会从 etcd 中基于  `quorum-read` 方式获取，此时数据是最新的，第二是设置为 0 从 apiserver cache 中获取，第三种则是设置为指定的 `resourceVersion`。



那难道在 kubelet list/get pod 时不设置 `resourceVersion` 解决不了吗？社区给了一个场景，试想在一个超大集群中，有 5K node 且每个 node 有 30 个 pods，此时集群中有 15 万 pods，在此集群中某个 node 使用 list 请求 apiserver 时，其仅仅需要本机的 30 个 pods，而 apiserver 需要从 etcd 中获取 15 万个 pods 对象并过滤出该 node 所需要的 30 个 pods，这种操作对集群的影响是不可预知的，集群性能骤降或者集群宕机都有可能出现。



#### 解决办法：

通过以上描述可知，根本问题是在 apiserver 与 etcd 之间的数据传输时有一定延迟导致的。而在 etcd 3.4+ 版本中支持了在客户端 watch 时启用 `WithProgressNotify` 参数，当 `WithProgressNotify` 参数启用后，etcd 会自动发送 progress events，此时客户端缓存中的数据与 etcd 中的数据是一致的，但 etcd 默认每 10 分钟发送一次，社区计划设置 progress events 的时延为 250ms 进行测试，根据社区的讨论，其会在数据准确性、性能以及可扩展性等方面进一步测试以及讨论该决策是否满足需求。

该功能会在 kubernetes 新版本中以 `WatchCacheConsistentReads` feature gate 的方式开放用户使用。



参考文档：[20191210-consistent-reads-from-cache.md](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20191210-consistent-reads-from-cache.md)



### 3、支持使用 cgroup v2

该特性的目标：
- 在 kubernetes 中支持使用 cgroup v2；

Linux 内核已经支持 cgroup v2 特性两年多，cgroup v2 一个大的特性就是可以用非 root 用户操作资源限制（例如：可以使用非 root 权限模式运行 kubernetes 组件），该特性在内核中也已经处于稳定版本，某些发现版(例如 Fedora)中已经默认使用 cgroup v2，所以社区计划在 kubernetes 中支持使用 cgroup v2。这是一个庞大的计划，需要分为多步进行，社区首先会在 kubelet 中支持使用 cgroup v2（该特性已经在进行中 [#85218](https://github.com/kubernetes/kubernetes/pull/85218)），并保证 cgroup v1 的配置在 cgroup v2 上依然可以使用，然后会对 runtime 进行改造以及进行适配，目前 docker，containerd，runc，cAdvisor 等都已经相继增加了对 cgroupv2 的支持。



而从 cgroup v1 转换到 cgroup v2 也有一些风险存在：
- 1、cgroups v1 中部分特性无法在 cgroup v2 中使用，如 `cpuacct.usage_percpu` 和 cgroup 中的 `network stats`；
- 2、cgroups v1 中的一些 controller 在 v2 中也不可用 ，如 `device` 和 `net_cls`, `net_prio` 等，对于这部分不可用的 controller 社区将会使用 eBPF 替换他们；

  

参考文档：[20191118-cgroups-v2.md](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/20191118-cgroups-v2.md)



### 4、volume 被挂载时支持禁止更改 volume 的所有者以及权限

该特性的目标：
- volume 在 mount 时允许跳过更改其所有者以及权限；

  

目前，在 pod 中使用 volume 时，将 volume 挂载到容器之前时该 volume 中文件的权限以及所有者将被递归地更改为所提供的 `fsGroup` 的值，这种更改权限的操作可能需要很长时间才能完成，尤其是在非常大的 volume 中(>=1TB)。更改权限是为了保证所提供的 `fsGroup` 可以对此 volume 进行读写，但此时 pod 可能会启动超时，部分文件权限更改也可能会导致 pod 中某些应用无法启动。为了解决这一问题，社区将会在 pod 中添加一个名为 `.Spec.SecurityContext.FSGroupChangePolicy` 的字段，允许用户指定希望 pod 使用的 volume 权限和所有者如何更改。



参考文档：[20200120-skip-permission-change.md](https://github.com/gosoon/enhancements/blob/master/keps/sig-storage/20200120-skip-permission-change.md)


### 5. 支持禁用 ConfigMap/Secret 的自动更新机制

该特性的目标：
- 1、引入一种保护机制来禁止 ConfigMap/Secret 的自动更新；
- 2、提高 kube-apiserver 的性能；


社区为 ConfigMap 和 Secret 增加了一个 `Immutable` 字段来禁止其自动更新：

```
  Immutable *bool
```

建议使用 `Immutable` 的 ConfigMap/Secret 主要有两个原因：

- 一是 pod 使用 ConfigMap/Secret 的模式一般是通过 Volume Mounts 的方式，而 kubelet 会通过 Watch/Poll 的方式去获取 ConfigMap/Secret 更新，同时将最近文件同步到 pod 中，这种方式下 pod 能够快速、无感地获取到 ConfigMap/Secret 更新。但这种更新是一把双刃剑，一次错误的更新可能会导致 pod 内进程异常甚至 pod 不可用，而大多数人都不希望使用这种功能，更多的是使用 Rolling Update 的方式，创建一个新的 ConfigMap/Secret 同时创建新的 pod 去引用新的 ConfigMap/Secret；
- 二个是在大规模集群内，kubelet 过多的 Watch/Poll 大量的 ConfigMap/Secret 会给 kube-apiserver 造成巨大的压力（尽管我们在[这个 PR ](https://github.com/kubernetes/kubernetes/issues/84001)中为每个 Watch 请求降低了一个 Goruntine 的消耗）。而使用了 `Immutable` 的 ConfigMap/Secret，kubelet 也就不会为其建立 Watch/Poll 请求；



官方文档：[20191117-immutable-secrets-configmaps.md](https://github.com/gosoon/enhancements/blob/master/keps/sig-storage/20191117-immutable-secrets-configmaps.md)




