---
title: kubernetes 版本多久该升级一次
date: 2019-09-26 15:13:30
tags: ["release version",""]
type: "release version"

---

kubernetes  社区每三个月发布一个新版本，可以说发布新版本的速度非常快，当然，在生产环境中版本升级的速度可能跟不上新版本发布的速度，那么确保目前使用的版本还处于社区的维护阶段就非常重要了，kubernetes 官方对各个版本支持的时间是多长呢？

kubernetes 发行版通常支持9个月，在此期间，如果发现严重的bug或安全问题，会在对应的分支发布补丁版本。比如，当前版本为 v1.10.1，当社区修复一些 bug 后，就会发布 v1.10.2 版本。

官方支持时间说明如下：

| Kubernetes version | Release month  | End-of-life-month |
| :----------------- | :------------: | :---------------- |
| v1.6.x             |   March 2017   | December 2017     |
| v1.7.x             |   June 2017    | March 2018        |
| v1.8.x             | September 2017 | June 2018         |
| v1.9.x             | December 2017  | September 2018    |
| v1.10.x            |   March 2018   | December 2018     |
| v1.11.x            |   June 2018    | March 2019        |
| v1.12.x            | September 2018 | June 2019         |
| v1.13.x            | December 2018  | September 2019    |
| v1.14.x            |   March 2019   | December 2019     |
| v1.15.x            |   June 2019    | March 2020        |
| v1.16.x            | September 2019 | June 2020         |

到目前为止，v1.13.x 以前的版本已经停止支持了，请尽快升级至高版本。



### kubernetes 版本发布流程

>  翻译自官方文档：[Kubernetes Release Versioning](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/release/versioning.md)

 **说明**：**Kube X.Y.Z** 代表 kubernetes 已经发布的版本（git tag），这个版本包含所有的组件：apiserver, kubelet, kubectl, etc. (**X** 表示主版本号, **Y** 是此版本号, **Z** 是补丁版本。)



#### 版本发布时间

##### 次版本发布计划与时间表

- Kube X.Y.0-alpha.W, W > 0 ( 分支：master)
  - Alpha 版本大约每两周直接从 master 分支发布一次。
  - 没有 cherrypick 版本。如有有严重的 bug 被修复，可以基于 master 分支提前创建一个新版本。

- Kube X.Y.Z-beta.W (分支: release-X.Y)

  - 当 master 完成 Kube X.Y 的功能后，在距 X.Y.0 发布前两周会停掉 release-X.Y 分支，只将一些比较重要的 PR cherry-pick 到 X.Y。

  - 该分支会被标记为  X.Y.0-beta.0，master 分支会被移到  X.Y+1.0-alpha.0。

    ![](http://cdn.tianfeiyu.com/image-20190925195015864.png)

  - 如果 X.Y.0-beta.0 的功能有缺陷，还会发布其他的 beta 版本 (X.Y.0-beta.W | W > 0) 。

- Kube X.Y.0 (分支: release-X.Y)

  - 最终的 release 版本会提前两周从 release-X.Y 分支上产生。
  - 在同一分支的同一 commit  处也会被标记为 X.Y.1-beta.0。
  - 在 X.Y.0 发布 3-4 个月后会发布 X.(Y-1).0。

- Kube X.Y.Z, Z > 0 (分支: release-X.Y)

  - 当 cherrypick commits 到 release-X.Y 分支时，若有需要，也会发布相应的[补丁版本](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/release/versioning.md#patch-releases) （X.Y.Z-beta.W）。
  - X.Y.Z 是直接从 release-X.Y 分支上产生的，当使用 beta 版本在更新 pkg/version/base.go 后会被标记为 X.Y.Z+1-beta.0。

- Kube X.Y.Z, Z > 0 (分支: release-X.Y.Z)

  - 这是一个特殊的 tag，如果在上一个 release 分支后有重大的 bug 被修复，会有一个 X.Y.Z tag。

  - release-X.Y.Z 分支会被停掉以确保补丁版本是最新的。

  - 如果还有重要 bug 被修复会再有一个补丁版本  X.Y.(Z+1)。

  - 一般不会有[补丁版本](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/release/versioning.md#patch-releases)，补丁版本仅用于一些重大 bug 的修复。

  -  可以参考[#19849](https://issues.k8s.io/19849)看看补丁版本的作用。



#### 主版本时间线

主版本暂时没有预期发布的时间点，也没有公布 2.0.0 的标准。到目前为止，我们还没有对任何类型的不兼容更改(例如，组件参数更改)。之前讨论过在发布 2.0.0 后 删除 `v1` API group/version，但目前没有这样做的计划。



#### 支持的组件版本与兼容版本

我们希望用户在生产中使用 kubernetes 最稳定的版本，但升级版本需要一些时间，尤其是对于生产环境中的关键组件。我们也希望用户更新到最新的补丁版本，补丁版本中包含一些重要的 bugfix，希望用户尽快升级。

kubernetes 对各组件的版本也有一定的兼容性。具体的兼容策略是： slave组件可以与master组件最多延迟两个版本(minor version)，但是不能比 master 组件新。client 不能与 master 组件落后一个次版本，但是可以高一个版本，也就是说： v1.3 的 master 可以与 v1.1，v1.2，v1.3 的 slave 组件一起使用，与 v1.2，v1.3，v1.4 client 一起使用。

此外，我们希望一次“支持”三个次版本，“支持”意味着我们希望用户在生产环境中运行该版本，虽然我们可能对于不在支持的版本进行 bugfix。例如，当 v1.3 发布时，将不再支持 v1.0。此外新版本每三个月发行一次，也就是说一个版本仅支持 9 个月。



### 升级策略

用户可以使用滚动方式升级，一次升级一个小版本，不建议直接跨度两个及以上小版本，升级时先升级 master 再升级 node 节点。

以下是在实际升级过程中的一些经验：

金丝雀部署：即灰度升级，若使用二进制部署，则在原有集群直接替换二进制进行升级，运维代价小，不会导致服务中断；若以 pod 方式部署的 master 组件直接替换镜像进行升级，若以 deployment 方式部署 master 组件，对于 apiserver 可以参考阿里的经验，设置 maxSurge=3 的方式升级，以避免升级过程带来的性能抖动，但所有的 node 组件依然需要替换二进制升级。

蓝绿部署：搭建一套新的集群，这种方式升级方式比较麻烦，涉及到数据迁移，IP 更换操作，对于部分业务不适用，风险不可控。



可以看到，kubernetes 社区的更新速度非常快，坚决不建议自己维护一套 kubernetes 版本，每次升级巨麻烦，将所有修改过的 commit cherry-pick 到每个新版本上，也容易出错，有些新版本的改动也比较大，之前修改过的地方在新版中有可能已经被移除或放在别的位置了。

详细的升级策略可以参考：[kubernetes集群升级的正确姿势](https://www.cnblogs.com/gaorong/p/11266629.html)。



### 结论

kubernetes 每三个月发布一个版本，社区仅维护最新的三个版本，一个版本的维护时间为 9 个月，请尽量保持生产环境的版本在社区维护范围内，版本升级时尽量保持小版本滚动升级，不建议跨多个版本升级。


参考：
https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
https://github.com/kubernetes/community/blob/master/contributors/design-proposals/release/versioning.md

