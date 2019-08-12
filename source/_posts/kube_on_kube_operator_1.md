---
title: kube-on-kube-operator 开发(一)
date: 2019-08-05 16:37:30
tags: ["operator","kube-on-kube"]
type: "kubernetes-operator"

---

kubernetes 已经成为容器时代的分布式操作系统内核，目前也是所有公有云提供商的标配，在国内，阿里云、腾讯云、华为云这样的公有云大厂商都支持一键部署 kubernetes 集群，而 kubernetes 集群自动化管理则是迫切需要解决的问题。对于大部分不熟悉 kubernetes 而要上云的小白用户就强烈需要一个被托管及能自动化运维的集群，他们平时只是进行业务的部署与变更，只需要对 kubernetes 中部分概念了解即可。同样在私有云场景下，笔者所待过的几个大小公司一般都会维护多套集群，集群的运维工作就是一个很大的挑战，反观各大厂同样要有效可靠的管理大规模集群，kube-on-kube-operator 是一个很好的解决方案。

所谓 kube-on-kube-operator，就是将 kubernetes 运行在 kubernetes 上，用 kubernetes 托管 kubernetes 方式来自动化管理集群。和所有 operator 的功能类似，系统会定时检测集群当前状态，判断是否与目标状态一致，出现不一致时，operator 会发起一系列操作，驱动集群达到目标状态。今年 kubeCon 上，雅虎日本也分享了其管理大规模 kubernetes 集群的方法，4000 节点构建了 400 个 kubernetes 集群，同样采用的是 kube-on-kube-operator 架构，以 kubernetes as a service 的形式使用。

### kubernetes-operator 设计参考

记得 kube-on-kube-operator 的概念最初是在去年的 kubeCon China 上蚂蚁金服提出来的，先看看蚂蚁金服以及腾讯云 kube-on-kube-operator 的设计思路，其实腾讯云的架构和蚂蚁金服的是类似的。以下是蚂蚁金服的架构设计图：

![](http://cdn.tianfeiyu.com/image-20190805163304515.png)



首先部署一套 kubernetes 元集群，通过元集群部署业务集群，业务集群的 master 组件都分布在一台宿主上，该宿主以 node 节点的方式挂载在元集群中。在私有云场景下，这样的部署方式有一个很明显的问题就是元集群节点跨机房，虽然业务集群的 master 与 node 都是在同一个机房，但是元集群中的 node 节点大部分是分布在不同的机房，有些公司在不同机房之间会有网络上的限制，有可能网络不通或者只能使用专线连接。在公有云场景下，元集群自己有一套独立的 vpc 网络，它要怎么和用户的 vpc 结点进行通信呢？腾讯云的做法是利用 vpc 提供的弹性网卡能力，将这个弹性网卡直接绑定到运行 apiserver 的 pod 中，运行 master  的这个pod 既加入了元集群的 vpc，又加入了用户的 vpc，也就是一个 pod 同时在两个网络中，这样就可以很好的去实现和用户 node 相关的互通。这种方式都是通过 kubernetes  API 去管理 master 组件的，master 组件的升级以及故障自愈都可以通过 kubernetes 提供的方式实现。



### kubernetes-operator 设计

![kubernetes-operator 架构](http://cdn.tianfeiyu.com/image-20190804152312149.png)



> kubernetes-operator 项目地址：[https://github.com/gosoon/kubernetes-operator](https://github.com/gosoon/kubernetes-operator)

目前该项目的主要目标是实现以下三种场景中的集群管理：

- kube-on-kube
- kube-to-kube
- kube-to-cloud-kube

kubernetes-operator 不仅是要实现 kube-on-kube 架构，还有 kube-to-kube，kube-to-cloud-kube，kube-to-kube 即 kubernetes 集群管理业务独立的 kubernetes 集群，两个集群相互独立。kube-to-cloud-kube 即 kubernetes 集群管理多云环境上的 kubernetes 集群。



上面是项目的架构图，红色的线段表示对集群生命周期管理的一个操作，涉及集群的创建、删除、扩缩容、升级等，蓝色线段是对集群应用的操作，集群中应用的创建、删除、发布更新等，kubernetes-proxy 是一个 API 代理，所有涉及 API 的调用都要通过 kubernetes-proxy。左边部署有 kubernetes-operator 的是元集群，kubernetes-operator 使用 etcd 仅存储部分配置信息，其管理业务集群的生命周期，支持三种集群的创建方式，第一种方式就是可以创建出类似蚂蚁金服这种直接将业务集群 master 运行在元集群，node 节点在业务集群，第二种是以二进制方式创建业务集群，其中业务集群的 master 以及 node 都是在业务集群所在的机房，第三种方式就是在各种公有云厂商创建集群，以一种统一的方式管理公有云上的集群，也可以称作融合云。



#### 项目结构

总体来说，项目暂时分为三大块：

- kubernetes proxy：支持 API 透传、访问控制等功能；
- 控制器：也就是 kubernetes-operator，管理业务集群的生命周期；
- 集群部署模块：用来部署业务集群，目前主要在开发第二种方式使用二进制部署业务集群；
- kubernetes 应用安装模块：在新建完成的集群中部署监控、日志采集、镜像仓库、helm 等组件；



#### 控制器

控制器也就是 Operator + CR，目前开发 operator 的方式已知的有三种：

-  自定义 controller 的方式：kube-controller-manager 中所有的 controller 就是以自定义 controller 的方式，这种方式是最原生的方式，需要开发者了解 kubernetes 中的代码生成，informer 的使用等。
- operator-sdk 的方式：一个开发 Operator 的框架，对于一些不熟悉 kubernetes 的可以使用 operator-sdk 的方式，这种方式让开发者更注重业务的实现，但是不够灵活。
- kubebuilder 的方式：kubebuilder 是开发 controller manager 的框架，controller manager 会管理一个或者多个 operator。



kubebuilder, operator-sdk 都是对controller-runtime做了封装, controller runtime又是对client-go shardInfromer 做的封装，本质上其实都一样的。kubernetes-operator 使用的是自定义 controller  的方式，如果想要更深入的学习 kubernetes，非自定义 controller 方式莫属了，kube-controller-manager 组件中的各种 controller 都是使用这种方式开发的，完全可以按照官方这种套路来开发。在 kubernetes 中，目前有两种方式可以定义一个新对象，一是 CustomResourceDefinition（CRD）、二是 Aggregation ApiServer（AA），其中 CRD 是相对简单也是目前应用比较广的方法。kubernetes-operator 采用 CRD 的方式。



#### 集群部署

其实项目中最难的是集群部署这一部分，部署集群目前有两种方式，二进制部署和容器化部署，但是都有一些开源工具的支持。手动部署一个二进制集群需要熟悉 docker 的部署、etcd 的部署、角色证书的创建、RBAC 授权、网络配置、yaml 文件编写、kubernetes 集群运维等等，总之手动部署一个二进制集群是非常麻烦的，但是要真正会用 kubernetes 是逃不了部署这一步的。第二种方式就是以容器化的方式部署，这种部署方式相对来说比较简单，有现成的工具直接傻瓜式操作就能部署成功。但是我目前选择的是使用二进制的部署方式，由于自己运维过二进制的 kubernetes 集群，对于私有云场景一般都是直接将集群部署在物理机上，作为生产环境，自己认为容器化的方式部署还不是非常成熟的，目前工作过的大小公司中，生产环境暂时没有以容器化的方式运行集群。所以 kubernetes-operator 中目前主要支持的就是使用二进制部署集群。



目前比较成熟的用于生产环境的 kubernetes 集群部署工具有：kubeadm、kubespary、kops、rancher、kubeasz 等。kubeadm、kubespary、kops 都是官方开源的产品，kubeadm 使用容器化的方式部署，需要手动执行一些部署命令，暂时无法完全自动化部署。kubespary 是对 kubeadm 的一层封装，使用 ansible + kubeadm 的方式自动化进行部署，据说阿里云就是使用 kubespary 部署集群的。在公有云的环境(GCP、AWS)通常使用 kops  部署起来更方便些。kubeasz 是使用 ansible 自动化的方式部署二进制集群，目前也已经比较成熟了。



#### 应用安装

- 监控：当然是使用 promethus；
- 日志采集：使用 filebeat 或者基于 filebeat 封装的一些组件如 logpilot，其他的还有 logkit 等都可以尝试使用；
- 镜像仓库：当然是使用 harbor；
- HPA：组件以及应用的自动扩缩容；

应用安装使用 helm 的方式进行安装。



#### 集群升级

若以二进制部署最好是替换二进制文件的方式进行升级，若使用容器化部署，master 部署在元集群中可以使用 kubernetes 的滚动方式升级否则要以修改 manifest 文件的方式。

集群升级包括配置和版本的升级，集群部署完成后，master 的配置改动不会很频繁，由于要进行性能上的优化以及业务的支持，对于 node 组件上的配置升级还是比较多的。对于集群的版本升级，升级的难度系数随着版本的跨度增大而增大，若按照官方的升级流程，一般不会出现异常。升级操作一般都是先升 master 再升 node，在工作中经历的几次版本升级中，每次升级完 master 后理论上不会再回退了，除非升级过程中有问题，否则升级完成后已经很难回退了，master 升级完成后 APIServer 的一些 API 还有 pod 的字段都有可能改变，master 版本回退后一些已存在的应用可能会异常，或者还可以参考 openshift 的蓝绿升级方式。二进制部署的集群尽量以替换二进制文件的方式进行升级，对于容器化部署的集群，可以直接使用 kubernetes 的滚动方式升级或者是修改 manifest 文件的方式。

目前蚂蚁金服 kube-on-kube-operator 架构中在业务集群中会部署一个 node-operator，node-operator 会记录 master 组件的镜像、默认启动参数等信息，其作用就是节点配置管理、集群组件升级以及节点故障自愈，未来在项目中也会实现基于此的方式。



### 后期计划

- 支持部署 k3s、kubeedge：5G 时代，边缘计算将是非常火的，目前各大厂商也都在此布局，所以支持部署 k3s、kubeedge 这些专门支持边缘计算的产品还是非常有必要的。
- 支持使用 kops 部署
- 支持部署多版本 k8s
- node-operator 开发，支持集群的配置管理、自动化升级、故障自愈等功能
- 用户及权限管理：操作集群用户的权限和 kubernetes 中 RBAC 规则绑定
- Kubernetes-operator 一些功能的扩展和完善



参考：

[腾讯云容器服务TKE：一键部署实践](https://mp.weixin.qq.com/s/WScGf3DRDC8ryyrf_tY-Qw)

[一年时间打造全球最大规模之一的Kubernetes集群，蚂蚁金服怎么做到的？](https://mp.weixin.qq.com/s/bJrMNxKMn89TzmpEyIZrRg)

[https://github.com/gosoon/kubernetes-operator](https://github.com/gosoon/kubernetes-operator)
