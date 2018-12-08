---
title: Docker 架构中的几个核心概念
date: 2018-12-05 20:57:00
type: "docker"

---


## 一、Docker 开源之路

2015 年 6 月 ，docker 公司将 libcontainer 捐出并改名为 runC 项目，交由一个完全中立的基金会管理，然后以 runC 为依据，大家共同制定一套容器和镜像的标准和规范 OCI。

2016 年 4 月，docker 1.11 版本之后开始引入了 containerd 和 runC，Docker 开始依赖于 containerd 和 runC 来管理容器，containerd 也可以操作满足 OCI 标准规范的其他容器工具，之后只要是按照 OCI 标准规范开发的容器工具，都可以被 containerd 使用起来。

从 2017 年开始，Docker 公司先是将 Docker项目的容器运行时部分 Containerd 捐赠给CNCF 社区，紧接着，Docker 公司宣布将 Docker 项目改名为 Moby。


## 二、Docker 架构

![docker 架构](https://upload-images.jianshu.io/upload_images/1262158-eee83eb356fccdb8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![docker 进程关系](https://upload-images.jianshu.io/upload_images/1262158-3f74443f956fa132.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 三、核心概念

docker 1.13 版本中包含以下几个二进制文件。
```
$ docker --version
Docker version 1.13.1, build 092cba3

$ docker
docker             docker-containerd-ctr   dockerd      docker-proxy
docker-containerd  docker-containerd-shim  docker-init  docker-runc
```

#### 1、docker 
docker 的命令行工具，是给用户和 docker daemon 建立通信的客户端。

#### 2、dockerd 
dockerd 是 docker 架构中一个常驻在后台的系统进程，称为 docker daemon，dockerd 实际调用的还是 containerd 的 api 接口（rpc 方式实现）,docker daemon 的作用主要有以下两方面：

- 接收并处理 docker client 发送的请求
- 管理所有的 docker 容器

有了 containerd 之后，dockerd 可以独立升级，以此避免之前 dockerd 升级会导致所有容器不可用的问题。

#### 3、containerd

containerd 是 dockerd 和 runc 之间的一个中间交流组件，docker 对容器的管理和操作基本都是通过 containerd 完成的。containerd 的主要功能有：
- 容器生命周期管理
- 日志管理
- 镜像管理
- 存储管理
- 容器网络接口及网络管理

#### 4、containerd-shim

containerd-shim 是一个真实运行容器的载体，每启动一个容器都会起一个新的containerd-shim的一个进程， 它直接通过指定的三个参数：容器id，boundle目录（containerd 对应某个容器生成的目录，一般位于：/var/run/docker/libcontainerd/containerID，其中包括了容器配置和标准输入、标准输出、标准错误三个管道文件），运行时二进制（默认为runC）来调用 runc 的 api 创建一个容器，上面的 docker 进程图中可以直观的显示。其主要作用是：

- 它允许容器运行时(即 runC)在启动容器之后退出，简单说就是不必为每个容器一直运行一个容器运行时(runC)
- 即使在 containerd 和 dockerd 都挂掉的情况下，容器的标准 IO 和其它的文件描述符也都是可用的
- 向 containerd 报告容器的退出状态

有了它就可以在不中断容器运行的情况下升级或重启 dockerd，对于生产环境来说意义重大。

#### 5、runC
runC 是 Docker 公司按照 OCI 标准规范编写的一个操作容器的命令行工具，其前身是 libcontainer 项目演化而来，runC 实际上就是 libcontainer 配上了一个轻型的客户端，是一个命令行工具端，根据 OCI（开放容器组织）的标准来创建和运行容器，实现了容器启停、资源隔离等功能。

一个例子，使用 runC 运行 busybox 容器:
```
# mkdir /container
# cd /container/
# mkdir rootfs

准备容器镜像的文件系统,从 busybox 镜像中提取
# docker export $(docker create busybox) | tar -C rootfs -xvf -    
# ls rootfs/
bin  dev  etc  home  proc  root  sys  tmp  usr  var

有了rootfs之后，我们还要按照 OCI 标准有一个配置文件 config.json 说明如何运行容器，
包括要运行的命令、权限、环境变量等等内容，runc 提供了一个命令可以自动帮我们生成
# docker-runc spec
# ls
config.json  rootfs
# docker-runc run simplebusybox    #启动容器
/ # ls
bin   dev   etc   home  proc  root  sys   tmp   usr   var
/ # hostname
runc
```
---
参考：
[Use of containerd-shim in docker-architecture](https://groups.google.com/forum/#!topic/docker-dev/zaZFlvIx1_k)
[从 docker 到 runC](https://www.cnblogs.com/sparkdev/p/9129334.html)
[OCI 和 runc：容器标准化和 docker](http://cizixs.com/2017/11/05/oci-and-runc/)
[Open Container Initiative](https://github.com/opencontainers)
