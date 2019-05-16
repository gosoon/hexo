---
title: 使用插件扩展 kubectl
date: 2019-05-16 11:20:30
tags: "kubectl plugin"
type: "kubectl plugin"

---
由于笔者所维护的集群规模较大，经常需要使用 kubectl 来排查一些问题，但是 kubectl 功能有限，有些操作还是需要写一个脚本对 kubectl 做一些封装才能达到目的。比如我经常做的一个操作就是排查一下线上哪些宿主的 cpu/memory request 使用率超过某个阈值，kubectl 并不能直接看到一个 master 下所有宿主的 request 使用率，但可以使用 `kubectl describe node xxx`查看某个宿主机的 request 使用率，所以只好写一个脚本来扫一遍了。

```
#!/bin/bash

echo -e "node\tcpu_requets  memory_requets"
for i in `kubectl get node | grep -v NAME | awk '{print $1}'`;do
    res=$(kubectl describe node $i  | grep -A 3 "Resource")
    cpu_requets=$(echo ${res} | awk '{print $9}' | awk -F '%' '{print $1}' | awk -F '(' '{print $2}')
    memory_requets=$(echo ${res} | awk '{print $14}' | awk -F '%' '{print $1}' | awk -F '(' '{print $2}')
    echo -e "$i\t${cpu_requets} \t${memory_requets}"
done
```

类似的需求比较多，此处不一一列举，这种操作经常需要做，虽然写一个脚本也能完全搞定，但确实比较 low，也不便提供给别人使用，基于此了解到目前官方对 kubectl 的插件机制做了一些改进，对 kubectl 的扩展也比较容易，所以下文会带你了解一下 kubectl 的扩展功能。



#### 一、编写 kubectl 插件

kubectl 命令从 `v1.8.0` 版本开始支持插件机制，之后的版本中我们都可以对 `kubectl` 命令进行扩展，kubernetes 在 `v1.12` 以后插件可以直接是以 kubectl- 开头命令的一个二进制文件，插件机制在 `v1.14` 进入 GA 状态，这种改进是希望用户以二进制文件形式可以扩展自己的 kubectl 子命令。当然，kubectl 插件机制是与语言无关的，也就是说你可以用任何语言编写插件。



如 [kubernetes 官方文档](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/)中描述，只要将二进制文件放在系统 PATH 下，kubectl 即可识别，二进制文件类似 `kubectl-foo-bar`，并且在使用时 kubectl 会匹配最长的二进制文件。

官方建议使用  [k8s.io/cli-runtime](https://github.com/kubernetes/cli-runtime) 库进行编写，若你的插件需要支持一些命令行参数，可以参考使用，官方也给了一个例子 [sample-cli-plugin](https://github.com/kubernetes/sample-cli-plugin)。



还是回到最初的问题，对于获取一个集群写所有 node 的资源使用率，笔者基于也编写了一个简单的插件。

```
// 安装插件

$ CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o bin/kubectl-view-node-resource cmd/view-node-resource/main.go

$ mv bin/kubectl-view-node-resource /usr/bin/   
```



使用 `kubectl plugin list` 查看 PATH 下有哪些可用的插件。

```
// 查看插件

$ kubectl plugin list
The following kubectl-compatible plugins are available:

/usr/bin/kubectl-view-node-resource
```

```
// 使用插件

$ kubectl view node taints --help
A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:
Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

Usage:
  view-node-taints [flags]

Flags:
      --config string   config file (default is $HOME/.view-node-taints.yaml)
  -h, --help            help for view-node-taints
  -t, --toggle          Help message for toggle


$ kubectl view node resource
 Name            PodCount  CPURequests  MemoryRequests  CPULimits     MemoryLimits
 192.168.1.110   4         0 (0.00%)    6.4 (41.26%)    8 (100.00%)   16.0 (103.14%)
```



此外，还开发一个查看集群下所有 node taints 的插件，kubectl 支持查看宿主的 label，但是没有直接查看所有宿主 taints 的命令，插件效果如下：

```
$ kubectl view node taints
 Name            Status                       Age   Version                         Taints
 192.168.1.110   Ready,SchedulingDisabled     49d   v1.8.1-35+9406f9d9909c61-dirty  enabledDiskSchedule=true:NoSchedule
```

> 插件代码地址：[kubectl-plugin](https://github.com/gosoon/kubectl-plugin)



#### 二、kubectl 插件管理工具 krew

上文讲了如何编写一个插件，但是官方也提供一个插件库并提供了一个插件管理工具 [krew](https://github.com/kubernetes-sigs/krew)  ，[krew](https://github.com/kubernetes-sigs/krew) 是 kubectl 插件的管理器，使用 krew 可以轻松的查找、安装和管理 kubectl 插件，它类似于 yum、apt、 dnf，krew 也可以帮助你将已写好的插件在多个平台上打包和分发，krew 自己也作为一个 kubectl 插件存在。

> krew 仅支持在 v1.12 及之后的版本中使用。



1、安装 krew

```
$ (
  set -x; cd "$(mktemp -d)" &&
  curl -fsSLO "https://storage.googleapis.com/krew/v0.2.1/krew.{tar.gz,yaml}" &&
  tar zxvf krew.tar.gz &&
  ./krew-"$(uname | tr '[:upper:]' '[:lower:]')_amd64" install \
    --manifest=krew.yaml --archive=krew.tar.gz
)

$ export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

2、krew 的使用

```
$ kubectl krew search               								# show all plugins
$ kubectl krew install view-secret  								# install a plugin named "view-secret"
$ kubectl view-secret default-token-4cwvh           # use the plugin
$ kubectl krew upgrade              								# upgrade installed plugins
$ kubectl krew remove view-secret   								# uninstall a plugin
```

若想让你自己的插件加入到 krew 的索引中，可以参考：[how to package and publish a plugin for krew](https://github.com/kubernetes-sigs/krew/blob/master/docs/DEVELOPER_GUIDE.md)。



参考：

[kubectl 插件命明规范](https://github.com/kubernetes-sigs/krew/blob/master/docs/NAMING_GUIDE.md)

https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/

https://github.com/gosoon/kubectl-plugin
