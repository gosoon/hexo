---
title: kubernetes 集群升级至 v1.12 需要注意的几个问题
date: 2019-03-05 18:30:30
tags: "kubernetes v1.12"
type: "v1.12"

---

最近我们生产环境的集群开始升级至 v1.12 版本了，之前的版本是 v1.8，由于跨了多个版本，风险还是比较大的，官方的建议也是一个一个版本升级，k8s 每三个月出一个版本，集群上了规模后升级太麻烦，鉴于我们真正使用 k8s 中的功能还是比较少的，耦合性没有那么大，所以风险还是相对可控，测试环境运行 v1.12 一段时间后发现问题不大，于是开始升级。此处记录几个升级过程要注意的问题：

### 1、注意 k8s 中 resource version 的变化

k8s 中许多 resouce 都是随着 k8s 的版本变化而变化的，例如，statefulset 在 v1.8 版本中 apiVersion 是 apps/v1beta1，在 v1.12 中变为了 apps/v1。k8s 有接口可以获取到当前版本所有的 OpenAPI ：

![image.png](https://upload-images.jianshu.io/upload_images/1262158-272c63b4eabe3cee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

参考文档：[The Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)

虽然 k8s 中 resource version 都是向下兼容的，但是在升级完成后尽量使用当前版本的 resource version 避免不必要的麻烦。

### 2、kubelet 配置文件格式

v1.8 中 kubelet 的配置是在 /etc/kubernetes/kubelet 文件中的 KUBELET_ARGS 后面指定，但是在 v1.12 中开始使用 config.yaml 文件，即所有的配置都可以放在 yaml 文件中，由于配置是兼容的，所以暂时也可以继续用以前的方式，其中有些参数仅支持在 config.yaml 文件中指定。

config.yaml 文件的官方说明：[Set Kubelet parameters via a config file](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)

一个例子：

```
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
maxOpenFiles: 1000000
maxPods: 110
nodeLeaseDurationSeconds: 40
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
port: 10250
staticPodPath: /etc/kubernetes/manifests
```

将 kubelet 配置文件中的 LOG_LEVEL 参数改为大于等于 5 可以看到 config.yaml 中配置的定义，以方便排查问题：

![image.png](https://upload-images.jianshu.io/upload_images/1262158-8af5facac815e980.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对应的日志输出：

```
I0228 16:14:14.064292  191819 server.go:260] KubeletConfiguration: 
config.KubeletConfiguration{TypeMeta:v1.TypeMeta{Kind:"", 
APIVersion:""}, StaticPodPath:"", Sync    
Frequency:v1.Duration{Duration:60000000000}, 
FileCheckFrequency:v1.Duration{Duration:20000000000},
HTTPCheckFrequency:v1.Duration{Duration:20000000000}, 
StaticPodURL:"", StaticPodURLHeader:map[string][]string(nil),
Address:"0.0.0.0", Port:10250, 
...
```

> 注意：kubelet 配置文件中 ARGS 中定义的参数会覆盖 config.yaml 中的定义。


### 3、feature-gates 中功能的使用

v1.12 中 feature-gates 中许多功能默认为开启状态，需要根据实际场景选择，不必要的功能在配置文件中将其关闭。

k8s 各版本中的 Feature 列表以及是否启用状态可以在 [Feature Gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) 中查看。


### 4、cadvisor 的使用 

k8s 在 1.12 中将 cadvisor 从 kubelet 中移除了，若要使用 cadvisor，官方建议使用 DaemonSet 进行部署。由于我们一直从 cadvisor 获取容器的监控数据然后推送到自有的监控系统中进行展示，所以 cadvisor 还得继续使用。

这是官方推荐的 cadvisor 部署方法，[cAdvisor Kubernetes Daemonset](
https://github.com/google/cadvisor/blob/master/deploy/kubernetes/README.md)，其中用了 `k8s.gcr.io/cadvisor:v0.30.2` 镜像，在我们的测试环境中，该镜像无法启动，报错 `/sys/fs/cgroup/cpuacct,cpu: no such file or directory`, 经查 cadvisor v0.30.2 版本的镜像使用 cgroup v2，v2 版本中已经没有了 cpuacct subsystem，而 linux kernel 4.5 以上的版本才支持 cgroup v2，与我们的实际场景不太相符，最后测试发现 v0.28.0 的镜像可以正常使用。

由于要兼容之前的使用方式，cadvisor 在宿主机上需要启动 4194 端口，但是创建容器又要结合自身的网络方案，最终我们使用 hostnetwork 的方式部署。

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cadvisor
  namespace: kube-system
  labels:
    app: cadvisor
spec:
  selector:
    matchLabels:
      name: cadvisor
  template:
    metadata:
      labels:
        name: cadvisor
    spec:
      hostNetwork: true
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
        key: enabledDiskSchedule
        value: "true"
        effect: NoSchedule
      containers:
      - name: cadvisor
        image: k8s.gcr.io/cadvisor:v0.28.0
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: rootfs
          mountPath: /rootfs
          readOnly: true
        - name: var-run
          mountPath: /var/run
          readOnly: false
        - name: sys
          mountPath: /sys
          readOnly: true
        - name: docker
          mountPath: /var/lib/docker
          readOnly: true
        ports:
          - name: http
            containerPort: 4194
            protocol: TCP
        readinessProbe:
          tcpSocket:
            port: 4194
          initialDelaySeconds: 5
          periodSeconds: 10
        args:
          - --housekeeping_interval=10s
          - --port=4194
      terminationGracePeriodSeconds: 30
      volumes:
      - name: rootfs
        hostPath:
          path: /
      - name: var-run
        hostPath:
          path: /var/run
      - name: sys
        hostPath:
          path: /sys
      - name: docker
        hostPath:
          path: /var/lib/docker
```

官方建议使用 kustomize 进行部署，kustomize 是 k8s 的一个配置管理工具，此处暂不详细解释。

> 注意：若集群中有打 taint 的宿主，需要在 yaml 文件中加上对应的 tolerations。


