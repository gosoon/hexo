---
title: kubernetes 常用 API
date: 2018-09-02 13:13:00
type: "kubernetes"

---

kubectl  的所有操作都是调用 kube-apisever 的 API 实现的，所以其子命令都有相应的 API，每次在调用 kubectl 时使用参数  -v=9  可以看调用的相关 API，例：
 `$ kubectl get node -v=9` 

以下为 kubernetes 开发中常用的 API：
![deployment 常用 API](http://cdn.tianfeiyu.com/deploy-1.png)

![statefulset 常用 API](http://cdn.tianfeiyu.com/sts-1.png)

![pod 常用 API](http://cdn.tianfeiyu.com/pod-1.png)


![service 常用 API](http://cdn.tianfeiyu.com/service-1.png)

![endpoints 常用 API](http://cdn.tianfeiyu.com/endpoints-1.png)

![namespace 常用 API](http://cdn.tianfeiyu.com/namespace-1.png)

![node 常用 API](http://cdn.tianfeiyu.com/nodes-1.png)

![pv 常用 API](http://cdn.tianfeiyu.com/pv-1.png)

 Markdown 表格显示过大，此仅以图片格式展示。

