---
title: 浅析 kubernetes 的认证与鉴权机制
date: 2019-08-18 21:20:30
tags: ["Authentication","Authorization","RBAC"]
type: "Authentication,Authorization,RBAC"

---

笔者最初接触 kubernetes 时使用的是 v1.4 版本，集群间的通信仅使用 8080 端口，认证与鉴权机制还未得到完善，到后来开始使用 static token 作为认证机制，直到 v1.6 时才开始使用 TLS 认证。随着社区的发展，kubernetes 的认证与鉴权机制已经越来越完善，新版本已经全面趋于 TLS + RBAC 配置，但其认证与鉴权机制也极其复杂，本文将会带你一步步了解。



kubernetes 集群的所有操作基本上都是通过 apiserver 这个组件进行的，它提供 HTTP RESTful 形式的 API 供集群内外客户端调用。kubernetes 对于访问 API 来说提供了三个步骤的安全措施：认证、授权、准入控制，当用户使用 kubectl，client-go 或者 REST API 请求 apiserver 时，都要经过以上三个步骤的校验。认证解决的问题是识别用户的身份，鉴权是为了解决用户有哪些权限，准入控制是作用于 kubernetes 中的对象，通过合理的权限管理，能够保证系统的安全可靠。认证授权过程只存在 HTTPS 形式的 API 中，也就是说，如果客户端使用 HTTP 连接到 apiserver，是不会进行认证授权的，然而 apiserver 的非安全认证端口 8080 已经在 v1.12 中废弃了，未来将全面使用 HTTPS。



![](http://cdn.tianfeiyu.com/image-20190818210224777.png)





首先来看一下 kubernetes 中的认证、授权以及访问控制机制。



### kubernetes 的认证机制(Authentication)

kubernetes 目前所有的认证策略如下所示：

- X509 client certs
- Static Token File
- Bootstrap Tokens
- Static Password File
- Service Account Tokens
- OpenId Connect Tokens
- Webhook Token Authentication
- Authticating Proxy
- Anonymous requests
- User impersonation
- Client-go credential plugins 



可以看到，kubernetes 的认证机制非常多，要想一个个搞清楚也绝非易事，本文仅分析几个比较重要且使用广泛的认证机制。

#### X509 client certs

X509是一种数字证书的格式标准，现在 HTTPS 依赖的 SSL 证书使用的就是使用的 X509 格式。X509 客户端证书认证方式是 kubernetes 所有认证中使用最多的一种，相对来说也是最安全的一种，kubernetes 的一些部署工具 kubeadm、minkube 等都是基于证书的认证方式。客户端证书认证叫作 TLS 双向认证，也就是服务器客户端互相验证证书的正确性，在都正确的情况下协调通信加密方案。目前最常用的 X509 证书制作工具有 openssl、cfssl 等。

#### Service Account Tokens

有些情况下，我们希望在 pod 内部访问 apiserver，获取集群的信息，甚至对集群进行改动。针对这种情况，kubernetes 提供了一种特殊的认证方式：serviceaccounts。 serviceaccounts 是面向 namespace 的，每个 namespace 创建的时候，kubernetes 会自动在这个 namespace 下面创建一个默认的 serviceaccounts；并且这个 serviceaccounts 只能访问该 namespace 的资源。serviceaccounts 和 pod、service、deployment 一样是 kubernetes 集群中的一种资源，用户也可以创建自己的 serviceaccounts。

serviceaccounts 主要包含了三个内容：namespace、token 和 ca，每个 serviceaccounts 中都对应一个 secrets，namespace、token 和 ca 信息都是保存在 secrets 中且都通过 base64 编码的。namespace 指定了 pod 所在的 namespace，ca 用于验证 apiserver 的证书，token 用作身份验证，它们都通过 mount 的方式保存在 pod 的文件系统中，其三者都是保存在 `/var/run/secrets/kubernetes.io/serviceaccount/`目录下。



关于 serviceaccounts 的配置可以参考官方的 [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) 文档。



> 认证机制的官方文档，请参考：https://kubernetes.io/docs/reference/access-authn-authz/authentication/



##### 小结：

kubernetes 中有多种认证方式，上面讲了最常使用的两种认证方式，X509 client certs 认证方式是用在一些客户端访问 apiserver 以及集群组件之间访问时使用，比如 kubectl 请求 apiserver 时。serviceaccounts 是用在 pod 中访问 apiserver 时进行认证的，比如使用自定义 controller 时。 



认证解决的问题是识别用户的身份，那 kubernetes 中都有哪几种用户？目前 kubernetes 中的用户分为内部用户和外部用户，内部用户指在 kubernetes 集群中的 pod 要访问 apiserver 时所使用的，也就是 serviceaccounts，内部用户需要在 kubernetes 中创建。外部用户指 kubectl 以及一些客户端工具访问 apiserver 时所需要认证的用户，此类用户嵌入在客户端的证书中。



### kubernetes 的鉴权机制(Authorization)

kubernetes 目前支持如下四种鉴权机制：

- Node
- ABAC
- RBAC
- Webhook



下面仅介绍两种最常使用的鉴权机制：

#### Node

仅 v1.7 版本以上支持 Node 授权，配合 NodeRestriction 准入控制来限制 kubelet，使其仅可访问 node、endpoint、pod、service 以及 secret、configmap、pv、pvc 等相关的资源，在 apiserver 中使用以下配置来开启 node 的鉴权机制：

```
KUBE_ADMISSION_CONTROL="...,NodeRestriction,..."

KUBE_API_ARGS="...,--authorization-mode=Node,..."
```



#### RBAC

RBAC（Role-Based Access Control）是 kubernetes 中负责完成授权，是基于角色的访问控制，通过自定义角色并将角色和特定的 user，group，serviceaccounts 关联起来已达到权限控制的目的。



RBAC 中有三个比较重要的概念：

- Role：角色，它其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限；

- Subject：被作用者，包括 user，group，serviceaccounts，通俗来讲就是认证机制中所识别的用户；

- RoleBinding：定义了“被作用者”和“角色”的绑定关系，也就是将用户以及操作权限进行绑定；



RBAC 其实就是通过创建角色(Role），通过 RoleBinding 将被作用者（subject）和角色（Role）进行绑定。下图是 RBAC 中的几种绑定关系：

![rbac](http://cdn.tianfeiyu.com/rback.png)





> 鉴权机制的官方文档，请参考：https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules





### 准入控制(Admission Control)

准入控制是请求的最后一个步骤，准入控制有许多内置的模块，可以作用于对象的 "CREATE"、"UPDATE"、"DELETE"、"CONNECT" 四个阶段。在这一过程中，如果任一准入控制模块拒绝，那么请求立刻被拒绝。一旦请求通过所有的准入控制器后就会写入对象存储中。



准入控制是在 apiserver 中进行配置的：

```
KUBE_ADMISSION_CONTROL="--enable-admission-plugins=NamespaceLifecycle,LimitRanger,...MutatingAdmissionWebhook,ValidatingAdmissionWebhook,NodeRestriction..."
```

准入控制的配置是有序的，不同的顺序会影响 kubernetes 的性能，建议使用官方的配置。



若需要对 kubernetes 中的对象做一些扩展，可以使用准入控制，比如：创建 pod 时添加 initContainer 或者校验字段等。准入控制最常使用的扩展方式就是 [admission webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks)，以前写过一篇类似的文章，可以参考：[http://blog.tianfeiyu.com/2019/07/02/k8s_crd_verify/](http://blog.tianfeiyu.com/2019/07/02/k8s_crd_verify/)。



>  准入控制更详细的文档，请参考：https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/



#### 小结：

上文已经说了 kubernetes 中有两种用户，一种是内置用户被称为 serviceaccounts，一种外部用户，嵌入在客户端的证书中，那么 kubernetes 中有哪些证书链以及内嵌的用户如何与 RBAC 结合呢？



### kubernetes 中的证书链

笔者通过自己的研究及实践经验发现，在目前主流版本的 kubernetes 集群中，有四条重要的 CA 证书链，而在大多数生产环境中，则至少需要两条 CA 证书链。



- apiserver CA 证书链：主要用于 kubernetes 内部组件互相访问以及外部客户端访问 apiserver 使用
- etcd CA 证书链：主要用于 etcd 节点之间的访问以及 apiserver 访问 etcd 使用
- extension apiserver CA 证书链：用于访问 extension apiserver 使用，比如 metrics-server
- kubelet CA 信任链：用于 apiserver 访问 kubelet 时使用
- 其他证书链：admission webhook 证书链、audit webhook 证书链，用于 apiserver 访问 webhook 时使用



以上这几套 CA 证书链中，apiserver CA 证书链和 etcd CA 证书链是必要的。extension apiserver 的 CA 证书链只有在使用时才会用到，且不可与 apiserver CA 证书链相同。kubelet 的 CA 证书链不是必要的，根据部署的实际情况可以和 apiserver CA 证书链公用。



#### 证书中的内嵌用户如何与 RBAC 配置进行结合



##### 证书中的内嵌用户

以下是 kubelet 的证书请求文件（CSR）：

```
{
  "CN": "system:node:<nodeName>",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "China",
      "L": "Shanghai",
      "O": "system:nodes",
      "OU": "Kubernetes",
      "ST": "Shanghai"
    }
  ]
}
```



- “CN”：Common Name，从证书中提取该字段作为请求的用户名 (User Name)；
- “O”：Organization，从证书中提取该字段作为请求用户所属的组 (Group)；

kubernetes 使用 X509 证书中 CN(Common Name) 以及 O(Organization) 字段对应 kubernetes 中的 user 和 group，即 RBAC 中的 subject，而 kubernetes 也为多个组件内置了 Role 以及 RoleBinding，巧妙的将 Authentication 和 RBAC Authorization 结合到了一起。



查看 kubernetes 中内置的 RBAC：

```
$ kubectl get clusterrole

$ kubectl get clusterrolebinding
```



下面是 kubernetes 中核心组件内置的 user 和 group，在为每个组件生成证书时需要在其 CSR 中使用对应的 CN 和 O 字段。 

![csr](http://cdn.tianfeiyu.com/image-20190723194531004.png)



### 访问 apiserver 的几种方式

通过上文可以知道访问 apiserver 时需要通过认证、鉴权以及访问控制三个步骤，认证的方式可以使用  serviceaccounts 和 X509 证书，鉴权的方式使用 RBAC，访问控制若没有特殊需求可以不使用。



serviceaccounts 是 kubernetes 针对 pod 内访问 apiserver 提供的认证方式，那可以用在外部 client 端吗？答案是可以的，serviceaccounts 最终是通过 ca + token 的方式访问的，你只要创建一个 serviceaccounts 并从对应的 secrets 中获取 ca + token 即可访问 apiserver。那使用证书认证的方式可以在 pod 内访问 apiserver 吗？当然也可以，不过创建证书比 serviceaccounts 麻烦，证书默认是用于内置组件访问 apiserver 使用的。不论哪种方式，你都需要为其创建 RBAC 配置。



所以在 TLS +RBAC 模式下，访问 apiserver 目前有两种方式：

- 使用 serviceaccounts + RBAC ：需要创建 serviceaccounts 以及关联对应的 RBAC(ca + token + RBAC)
- 使用证书 + RBAC：需要用到 ca、client、client-key 以及关联对应的 RBAC(ca + client-key + client-cert + RBAC)



### 总结

本文主要讲述了 kubernetes 中的认证(Authentication)以及鉴权(Authorization)机制，其复杂性主要体现在部署 kubernetes 集群时组件之间的认证以及在集群中为附加组件配置正确的权限，希望通过本节你可以了解到 kubernetes 中的组件需要哪些权限认证以及如何为相关组件配置正确的权限。



参考： 

Controlling Access to the Kubernetes API：https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/

admission controllers：https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/

kubelet 配置权限认证：https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/  

master-node communication：https://kubernetes.io/docs/concepts/architecture/master-node-communication/

kubernetes 数字证书体系浅析：https://mp.weixin.qq.com/s/iXuDbPKjSc65t_9Y1--bog
