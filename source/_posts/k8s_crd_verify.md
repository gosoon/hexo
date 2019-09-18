---
title: kubernetes 自定义资源（CRD）的校验
date: 2019-07-02 11:00:00
tags: ["crd","admission control"]
type: "crd"

---

在以前的版本若要对 apiserver 的请求做一些访问控制，必须修改 apiserver 的源代码然后重新编译部署，非常麻烦也不灵活，apiserver 也支持一些动态的准入控制器，在 apiserver 配置中看到的`ServiceAccount,NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota` 等都是 apiserver 的准入控制器，但这些都是 kubernetes 中默认内置的。在 v1.9 中，kubernetes 的动态准入控制器功能中支持了  Admission Webhooks，即用户可以以插件的方式对 apiserver 的请求做一些访问控制，要使用该功能需要自己写一个 admission webhook，apiserver 会在请求通过认证和授权之后、对象被持久化之前拦截该请求，然后调用 webhook 已达到准入控制，比如 Istio 中 sidecar 的注入就是通过这种方式实现的，在创建 Pod 阶段 apiserver 会回调 webhook 然后将 Sidecar 代理注入至用户 Pod。 本文主要介绍如何使用 AdmissionWebhook 对 CR 的校验，一般在开发 operator 过程中，都是通过对 CR 的操作实现某个功能的，若 CR 不规范可能会导致某些问题，所以对提交 CR 的校验是不可避免的一个步骤。

kubernetes 目前提供了两种方式来对 CR 的校验，语法校验(`OpenAPI v3 schema`） 和语义校验
(`validatingadmissionwebhook`）。

CRD 的一个示例：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: kubernetesclusters.ecs.yun.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: ecs.yun.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: kubernetesclusters
    # singular name to be used as an alias on the CLI and for display
    singular: kubernetescluster
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: KubernetesCluster
	  # listKind
    listKind: KubernetesClusterList
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ecs
```



CRD 的一个对象：

```
apiVersion: ecs.yun.com/v1
kind: KubernetesCluster
metadata:
  name: test-cluster
spec:
  clusterType: kubernetes
  serviceCIDR: ''
  masterList:
  - ip: 192.168.1.10
  nodeList:
  - ip: 192.168.1.11
  privateSSHKey: ''
  scaleUp: 0
  scaleDown: 0
```



#### 一、OpenAPI v3 schema 

[OpenAPI](https://github.com/OAI/OpenAPI-Specification) 是针对 REST API 的 API 描述格式，也是一种规范。

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: kubernetesclusters.ecs.yun.com
spec:
  group: ecs.yun.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: kubernetesclusters
    singular: kubernetescluster
    kind: KubernetesCluster
    listKind: KubernetesClusterList
    shortNames:
    - ecs
  validation:
    openAPIV3Schema:
      properties:
        spec:
	  type: object
          required:
          - clusterType
          - masterList
          - nodeList
          properties:
            clusterType:
              type: string
            scaleUp:
              type: integer
            scaleDown:
              type: integer
              minimum: 0
```

上面是使用 OpenAPI v3 检验的一个例子，OpenAPI v3 仅支持一些简单的校验规则，可以校验参数的类型，参数值的类型(支持正则)，是否为必要参数等，但若要使用与、或、非等操作对多个字段同时校验还是做不到的，所以针对一些特定场景的校验需要使用 admission webhook。 



#### 二、Admission Webhooks

admission control 在 apiserver 中进行配置的，使用`--enable-admission-plugins` 或 `--admission-control`进行启用，admission control 配置的控制器列表是有顺序的，越靠前的越先执行，一旦某个控制器返回的结果是reject 的，那么整个准入控制阶段立刻结束，所以这里的配置顺序是有序的，建议使用官方的顺序配置。

在 v1.9 中，admission webhook 是通过在 `--admission-control` 中配置 `ValidatingAdmissionWebhook` 或 `MutatingAdmissionWebhook` 来支持使用的，两者区别如下：

- MutatingAdmissionWebhook：允许在 webhook 中对 object 进行 mutate 修改，但匹配到的 webhook **串行**执行，因为每个 webhook 都可能会 mutate object。
- ValidatingAdmissionWebhook: 不允许在 webhook 中对 Object 进行 mutate 修改，仅返回 true 或 false。

启用 admission webhook 后，每次对 CR 做 CRUD 操作时，请求就会被 apiserver 拦住，至于 CRUD 中哪些请求被拦住都是提前在 WebhookConfiguration 中配置的，然后会调用 AdmissionWebhook 进行检查是否 Admit 通过。



![kubernetes API request lifecycle](http://cdn.tianfeiyu.com/1562032999173.jpg)



#### 三、启用 Admission Webhooks 功能

> kubernetes 版本 >= v1.9

1、在 apiserver 中开启 admission webhooks

在 v1.9 版本中使用的是：

```shell
--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
```

在 v1.10 以后会弃用 `--admission-control`，取而代之的是  `--enable-admission-plugins`：

```
--enable-admission-plugins=NodeRestriction,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
```

启用之后在 api-resources 可以看到：

```
# kubectl api-resources | grep admissionregistration
mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration
```

2、启用 `admissionregistration.k8s.io/v1alpha1` API

```
//  检查 API 是否已启用
$ kubectl api-versions | grep admissionregistration.k8s.io
```
若不存在则需要在 apiserver 的配置中添加`--runtime-config=admissionregistration.k8s.io/v1alpha1`。

#### 四、编写 Admission Webhook Server

webhook 其实就是一个 RESTful API 里面加上自己的一些校验逻辑。

可以参考官方的示例： 
[https://github.com/kubernetes/kubernetes/blob/v1.13.0/test/images/webhook/main.go](https://github.com/kubernetes/kubernetes/blob/v1.13.0/test/images/webhook/main.go) 
或者 
[https://github.com/kubernetes/kubernetes/blob/v1.13.0/test/e2e/apimachinery/webhook.go](https://github.com/kubernetes/kubernetes/blob/v1.13.0/test/e2e/apimachinery/webhook.go)



> 完整代码参考：[https://github.com/gosoon/admission-webhook](https://github.com/gosoon/admission-webhook)

#### 五、部署 Admission Webhook Service

由于 apiserver 调用 webhook 时强制使用 TLS 认证，所以 WebhookConfiguration 中一定要配置 caBundle，也就是需要自己生成一套私有证书。

生成证书的方式比较多，以下使用 openssl 生成，脚本如下所示：

```
#!/bin/bash

# Generate the CA cert and private key
openssl req -nodes -new -x509 -days 365 -keyout ca.key -out ca.crt -subj "/CN=admission-webhook CA"

# Generate the private key for the webhook server
openssl genrsa -out admission-webhook-tls.key 2048

# Generate a Certificate Signing Request (CSR) for the private key, and sign it with the private key of the CA.
openssl req -new -key admission-webhook-tls.key -subj "/CN=admission-webhook.ecs-system.svc" \
    | openssl x509 -days 365 -req -CA ca.crt -CAkey ca.key -CAcreateserial -out admission-webhook-tls.crt

# Generate pem
openssl base64 -A < ca.crt > ca.pem
```

生成证书后将 ca.pem 中的内容复制到 caBundle 处。

ValidatingWebhook yaml 文件如下：

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  name: admission-webhook
webhooks:
  - name: admission-webhook.ecs-system.svc  # 必须为 <svc_name>.<svc_namespace>.svc.
    failurePolicy: Ignore
    clientConfig:
      service:
        name: admission-webhook
        namespace: ecs-system
        path: /ecs/operator/cluster  # webhook controller
      caBundle: xxx
    rules:
      - operations:   # 需要校验的方法
        - CREATE
        - UPDATE
        apiGroups:    # api group
        - ecs.yun.com
        apiVersions:  # version
        - v1
        resources:    # resource
        - kubernetesclusters
```

注意 `failurePolicy` 可以为 `Ignore`或者`Fail`，意味着如果和 webhook 通信出现问题导致调用失败，将根据 `failurePolicy`决定忽略失败（admit）还是准入失败(reject)。

最后将 webhook 部署在集群中。



参考：
https://github.com/gosoon/admission-webhook
https://banzaicloud.com/blog/k8s-admission-webhooks/
[http://blog.fatedier.com/2019/03/20/k8s-crd/](http://blog.fatedier.com/2019/03/20/k8s-crd/)
https://my.oschina.net/jxcdwangtao/blog/1591681
https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/
https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#is-there-a-recommended-set-of-admission-controllers-to-use
https://istio.io/zh/help/ops/setup/validation/
https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/
