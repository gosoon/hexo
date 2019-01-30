---
title: kubernetes 审计日志功能
date: 2019-01-30 16:26:30
tags: ["audit","log"]
type: "audit"

---
审计日志可以记录所有对 apiserver 接口的调用，让我们能够非常清晰的知道集群到底发生了什么事情，通过记录的日志可以查到所发生的事件、操作的用户和时间。kubernetes 在 v1.7 中支持了日志审计功能（Alpha），在 v1.8 中为 Beta 版本，v1.12 为 GA 版本。

> kubernetes feature-gates 中的功能 Alpha 版本默认为 false，到 Beta 版本时默认为 true，所以 v1.8 会默认启用审计日志的功能。


### 一、审计日志的策略

#### 1、日志记录阶段

kube-apiserver 是负责接收及相应用户请求的一个组件，每一个请求都会有几个阶段，每个阶段都有对应的日志，当前支持的阶段有：

- RequestReceived - apiserver 在接收到请求后且在将该请求下发之前会生成对应的审计日志。
- ResponseStarted - 在响应 header 发送后并在响应 body 发送前生成日志。这个阶段仅为长时间运行的请求生成（例如 watch）。
- ResponseComplete - 当响应 body 发送完并且不再发送数据。
- Panic - 当有 panic 发生时生成。

也就是说对 apiserver 的每一个请求理论上会有三个阶段的审计日志生成。

#### 2、日志记录级别

当前支持的日志记录级别有：

- None - 不记录日志。
- Metadata - 只记录 Request 的一些 metadata (例如 user, timestamp, resource, verb 等)，但不记录 Request 或 Response 的body。
- Request - 记录 Request 的 metadata 和 body。
- RequestResponse - 最全记录方式，会记录所有的 metadata、Request 和 Response 的 body。

#### 3、日志记录策略

在记录日志的时候尽量只记录所需要的信息，不需要的日志尽可能不记录，避免造成系统资源的浪费。

- 一个请求不要重复记录，每个请求有三个阶段，只记录其中需要的阶段
- 不要记录所有的资源，不要记录一个资源的所有子资源
- 系统的请求不需要记录，kubelet、kube-proxy、kube-scheduler、kube-controller-manager 等对 kube-apiserver 的请求不需要记录
- 对一些认证信息（secerts、configmaps、token 等）的 body 不记录 

k8s 审计日志的一个示例：

```
{
  "kind": "EventList",
  "apiVersion": "audit.k8s.io/v1beta1",
  "Items": [
    {
      "Level": "Request",
      "AuditID": "793e7ae2-5ca7-4ad3-a632-19708d2f8265",
      "Stage": "RequestReceived",
      "RequestURI": "/api/v1/namespaces/default/pods/test-pre-sf-de7cc-0",
      "Verb": "get",
      "User": {
        "Username": "system:unsecured",
        "UID": "",
        "Groups": [
          "system:masters",
          "system:authenticated"
        ],
        "Extra": null
      },
      "ImpersonatedUser": null,
      "SourceIPs": [
        "192.168.1.11"
      ],
      "UserAgent": "kube-scheduler/v1.12.2 (linux/amd64) kubernetes/73f3294/scheduler",
      "ObjectRef": {
        "Resource": "pods",
        "Namespace": "default",
        "Name": "test-pre-sf-de7cc-0",
        "UID": "",
        "APIGroup": "",
        "APIVersion": "v1",
        "ResourceVersion": "",
        "Subresource": ""
      },
      "ResponseStatus": null,
      "RequestObject": null,
      "ResponseObject": null,
      "RequestReceivedTimestamp": "2019-01-11T06:51:43.528703Z",
      "StageTimestamp": "2019-01-11T06:51:43.528703Z",
      "Annotations": null
    }
    ]
}
```

### 二、启用审计日志

当前的审计日志支持两种收集方式：保存为日志文件和调用自定义的 webhook，在 v1.13 中还支持动态的 webhook。

#### 1、将审计日志以 json 格式保存到本地文件

apiserver 配置文件的 KUBE_API_ARGS 中需要添加如下参数：
```
--audit-policy-file=/etc/kubernetes/audit-policy.yaml --audit-log-path=/var/log/kube-audit --audit-log-format=json
```

日志保存到本地后再通过 fluentd 等其他组件进行收集。
还有其他几个选项可以指定保留审计日志文件的最大天数、文件的最大数量、文件的大小等。

#### 2、将审计日志打到后端指定的 webhook

```
--audit-policy-file=/etc/kubernetes/audit-policy.yaml --audit-webhook-config-file=/etc/kubernetes/audit-webhook-kubeconfig
```

webhook 配置文件实际上是一个 kubeconfig，apiserver 会将审计日志发送 到指定的 webhook 后，webhook 接收到日志后可以再分发到 kafka 或其他组件进行收集。

`audit-webhook-kubeconfig` 示例：
```
apiVersion: v1
clusters:
- cluster:
    server: http://127.0.0.1:8081/audit/webhook
  name: metric
contexts:
- context:
    cluster: metric
    user: ""
  name: default-context
current-context: default-context
kind: Config
preferences: {}
users: []
```

前面提到过，apiserver 的每一个请求会记录三个阶段的审计日志，但是在实际中并不是需要所有的审计日志，官方也说明了启用审计日志会增加 apiserver 对内存的使用量。

> Note: The audit logging feature increases the memory consumption of the API server because some context required for auditing is stored for each request. Additionally, memory consumption depends on the audit logging configuration.


`audit-policy.yaml` 配置示例：

```
apiVersion: audit.k8s.io/v1
kind: Policy
# ResponseStarted 阶段不记录
omitStages:
  - "ResponseStarted"
rules:
  # 记录用户对 pod 和 statefulset 的操作
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods","pods/status"]
    - group: "apps"
      resources: ["statefulsets","statefulsets/scale"]
  # kube-controller-manager、kube-scheduler 等已经认证过身份的请求不需要记录
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*"
    - "/version"
  # 对 config、secret、token 等认证信息不记录请求体和返回体
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]
```

官方提供两个参考示例：

- [Use fluentd to collect and distribute audit events from log file](https://kubernetes.io/zh/docs/tasks/debug-application-cluster/audit/#%E6%97%A5%E5%BF%97%E9%80%89%E6%8B%A9%E5%99%A8%E7%A4%BA%E4%BE%8B)
- [Use logstash to collect and distribute audit events from webhook backend](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/#use-logstash-to-collect-and-distribute-audit-events-from-webhook-backend)

#### 3、subresource 说明


kubernetes 每个资源对象都有 subresource,通过调用 master 的 api 可以获取 kubernetes 中所有的 resource 以及对应的 subresource,比如 pod 有 logs、exec 等 subresource。


![](https://upload-images.jianshu.io/upload_images/1262158-4450a01f65b3f76d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
获取所有 resource（ 1.10 之后使用）：
$ curl  127.0.0.1:8080/openapi/v2
```

参考：[https://kubernetes.io/docs/concepts/overview/kubernetes-api/](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)

### 三、webhook 的一个简单示例

```
package main

import (
	"encoding/json"
	"io/ioutil"
	"log"
	"net/http"

	"github.com/emicklei/go-restful"
	"github.com/gosoon/glog"
	"k8s.io/apiserver/pkg/apis/audit"
)

func main() {
	// NewContainer creates a new Container using a new ServeMux and default router (CurlyRouter)
	container := restful.NewContainer()
	ws := new(restful.WebService)
	ws.Path("/audit").
		Consumes(restful.MIME_JSON).
		Produces(restful.MIME_JSON)
	ws.Route(ws.POST("/webhook").To(AuditWebhook))

	//WebService ws2被添加到container2中
	container.Add(ws)
	server := &http.Server{
		Addr:    ":8081",
		Handler: container,
	}
	//go consumer()
	log.Fatal(server.ListenAndServe())
}

func AuditWebhook(req *restful.Request, resp *restful.Response) {
	body, err := ioutil.ReadAll(req.Request.Body)
	if err != nil {
		glog.Errorf("read body err is: %v", err)
	}
	var eventList audit.EventList
	err = json.Unmarshal(body, &eventList)
	if err != nil {
		glog.Errorf("unmarshal failed with:%v,body is :\n", err, string(body))
		return
	}
	for _, event := range eventList.Items {
		jsonBytes, err := json.Marshal(event)
		if err != nil {
			glog.Infof("marshal failed with:%v,event is \n %+v", err, event)
		}
		// 消费日志
		asyncProducer(string(jsonBytes))
	}
	resp.AddHeader("Content-Type", "application/json")
	resp.WriteEntity("success")
}
```

> 完整代码请参考：[https://github.com/gosoon/k8s-audit-webhook](https://github.com/gosoon/k8s-audit-webhook)


### 四、总结

本文主要介绍了 kubernetes 的日志审计功能，kubernetes 最近也被爆出多个安全漏洞，安全问题是每个团队不可忽视的，kubernetes 虽然被多数公司用作私有云，但日志审计也是不可或缺的。


----

参考：
[https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
[ttps://kubernetes.io/docs/tasks/debug-application-cluster/audit/](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/)
[阿里云 Kubernetes 审计日志方案](https://yq.aliyun.com/articles/686982?utm_content=g_1000040449)

