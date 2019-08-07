---
title: 使用 code-generator 为 CustomResources 生成代码
date: 2019-08-06 20:50:30
tags: ["code-generator","crd"]
type: "code-generator"

---

kubernetes 项目中有相当一部分代码是自动生成的，主要是 API 的定义和调用方法，kubernetes 项目下 `k8s.io/kubernetes/hack/` 目录中以 update 开头的大部分脚本都是用来生成代码的。[code-generator](https://github.com/kubernetes/code-generator/) 是官方提供的代码生成工具，在实现自定义 controller 的时候需要用到 CRD，也需要使用该工具生成对 CRD 操作的代码。


### 要生成哪些代码

在自定义 controller 时需要用到 typed clientsets，informers，listers 和 deep-copy 等函数，这些函数都可以使用 [code-generator](https://github.com/kubernetes/code-generator/) 来生成，具体的作用可以参考：[kubernetes 中 informer 的使用]([http://blog.tianfeiyu.com/2019/05/17/client-go_informer/](http://blog.tianfeiyu.com/2019/05/17/client-go_informer/)。

code-generator 里面包含多个生成代码的工具，下面是需要用到的几个：

- deepcopy-gen：为每种类型T生成方法： `func (t* T) DeepCopy() *T`，CustomResources 必须实现runtime.Object 接口且要有 DeepCopy 方法

- client-gen：为 CustomResource APIGroups 生成 typed clientsets

- informer-gen：为 CustomResources 创建 informers，用来 watch 对应 CRD 所触发的事件，以便对 CustomResources 的变化进行对应的处理

- lister-gen：为 CustomResources 创建 listers，用来对 GET/List 请求提供只读的缓存层



除了上面几个工具外，code-generator 中还提供了 conversion-gen、defaulter-gen、register-gen、set-gen，这些生成器可以应用在其他场景，比如构建聚合 API 服务时会用到一些内部的类型，conversion-gen 会为这些内部和外部类型之间创建转换函数，defaulter-gen 会处理某些字段的默认值。



### 代码生成步骤

使用 code-generator 生成代码还需要以下几步：

- 创建指定的目录格式
- 在代码中使用 tag 标注要生成哪些代码



首先要创建指定的目录格式，目录的格式可以参考官方提供的示例项目：[sample-controller](https://github.com/kubernetes/sample-controller)，下文也会讲到，目录中需要包含对应 CustomResources 的定义以及  group 和 version 信息。

其次要在在代码中使用 tag 标注要生成哪些代码，tag 有两钟类型，全局的和局部的，所有类型的 deepcopy tag 会默认启用，更多关于 tag 的使用方法可以参考：[Kubernetes Deep Dive: Code Generation for CustomResources](https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/)，也可以参考官方的示例 [code-generator/_example](https://github.com/kubernetes/code-generator/blob/master/_examples/crd/apis/example) 。



### 开始生成代码

本文以该 CRD 为例子进行演示，group 为`ecs.yun.com` ，version 为 `v1`：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: kubernetesclusters.ecs.yun.com
spec:
  group: ecs.yun.com
  names:
    kind: KubernetesCluster
    listKind: KubernetesClusterList
    plural: kubernetesclusters
    singular: kubernetescluster
    shortNames:
    - ecs
  scope: Namespaced
  subresources:
    status: {}
  version: v1
  versions:
  - name: v1
    served: true
    storage: true
```

创建指定目录结构 pkg/apis/${group}/${version}，group 可以定义一个 shortNames，也就是 CRD 中的 shortNames

```
$ mkdir -pv pkg/apis/ecs/v1
```

创建 doc.go：
```
$ cat << EOF > pkg/apis/ecs/v1/doc.go
// Package v1 contains API Schema definitions for the ecs v1 API group
// +k8s:deepcopy-gen=package,register
// +groupName=ecs.yun.com
package v1
EOF
```

创建 register.go：

```
$ cat << EOF > pkg/apis/ecs/v1/register.go
package v1

import (
	"github.com/gosoon/kubernetes-operator/pkg/apis/ecs"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
)

// SchemeGroupVersion is group version used to register these objects
var SchemeGroupVersion = schema.GroupVersion{Group: ecs.GroupName, Version: "v1"}

// Kind takes an unqualified kind and returns back a Group qualified GroupKind
func Kind(kind string) schema.GroupKind {
	return SchemeGroupVersion.WithKind(kind).GroupKind()
}

// Resource takes an unqualified resource and returns a Group qualified GroupResource
func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

var (
	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
	AddToScheme   = SchemeBuilder.AddToScheme
)

// Adds the list of known types to Scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&KubernetesCluster{},
		&KubernetesClusterList{},
	)
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}
EOF
```

创建 types.go，该文件中会定义多个 tag

```
$ cat << EOF > pkg/apis/ecs/v1/types.go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// KubernetesCluster is the Schema for the kubernetesclusters API
type KubernetesCluster struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   KubernetesClusterSpec   `json:"spec,omitempty"`
	Status KubernetesClusterStatus `json:"status,omitempty"`
}

// KubernetesClusterSpec defines the desired state of KubernetesCluster
type KubernetesClusterSpec struct {
	// Add custom validation using kubebuilder tags:
    // https://book.kubebuilder.io/beyond_basics/generating_crd.html
	TimeoutMins   string     `json:"timeout_mins,omitempty"`
	ClusterType   string     `json:"clusterType,omitempty"`
	ContainerCIDR string     `json:"containerCIDR,omitempty"`
	ServiceCIDR   string     `json:"serviceCIDR,omitempty"`
	MasterList    []Node     `json:"masterList" tag:"required"`
	MasterVIP     string     `json:"masterVIP,omitempty"`
	NodeList      []Node     `json:"nodeList" tag:"required"`
	EtcdList      []Node     `json:"etcdList,omitempty"`
	Region        string     `json:"region,omitempty"`
	AuthConfig    AuthConfig `json:"authConfig,omitempty"`
}

// AuthConfig defines the nodes peer authentication
type AuthConfig struct {
	Username      string `json:"username,omitempty"`
	Password      string `json:"password,omitempty"`
	PrivateSSHKey string `json:"privateSSHKey,omitempty"`
}

// KubernetesClusterStatus defines the observed state of KubernetesCluster
type KubernetesClusterStatus struct {
	// Add custom validation using kubebuilder tags: https://book.kubebuilder.io/beyond_basics/generating_crd.html
	Phase KubernetesOperatorPhase `json:"phase,omitempty"`

	// when job failed callback or job timeout used
	Reason string `json:"reason,omitempty"`

	// JobName is store each job name
	JobName string `json:"jobName,omitempty"`

	// Last time the condition transitioned from one status to another.
	LastTransitionTime metav1.Time `json:"lastTransitionTime,omitempty"`
}

// +genclient:nonNamespaced
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// KubernetesClusterList contains a list of KubernetesCluster
type KubernetesClusterList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []KubernetesCluster `json:"items"`
}

// users
// "None,Creating,Running,Failed,Scaling"
type KubernetesOperatorPhase string

type Node struct {
	IP string `json:"ip,omitempty"`
}
EOF
```



执行命令生成代码：

```
$ $GOPATH/src/k8s.io/code-generator/generate-groups.sh all github.com/gosoon/test/pkg/client github.com/gosoon/test/pkg/apis ecs:v1
```



generate-groups.sh 需要四个参数：

- 第一个 参数：all，也就是要生成所有的模块，clientset，informers，listers
- 第二个参数：github.com/gosoon/test/pkg/client  这个是你要生成代码的目录，目录的名称一般定义为 client
- 第三个参数：github.com/gosoon/test/pkg/apis  这个目录是已经创建好的源目录
- 第四个参数："ecs:v1" 是 group 和 version 信息，ecs 是 apis 下的目录，v1 是 ecs 下面的目录



生成的代码如下所示：

```
.
└── pkg
    ├── apis
    │   └── ecs
    │       └── v1
    │           ├── doc.go
    │           ├── register.go
    │           ├── types.go
    │           └── zz_generated.deepcopy.go
    └── client
        ├── clientset
        │   └── versioned
        │       ├── clientset.go
        │       ├── doc.go
        │       ├── fake
        │       │   ├── clientset_generated.go
        │       │   ├── doc.go
        │       │   └── register.go
        │       ├── scheme
        │       │   ├── doc.go
        │       │   └── register.go
        │       └── typed
        │           └── ecs
        │               └── v1
        │                   ├── doc.go
        │                   ├── ecs_client.go
        │                   ├── fake
        │                   │   ├── doc.go
        │                   │   ├── fake_ecs_client.go
        │                   │   └── fake_kubernetescluster.go
        │                   ├── generated_expansion.go
        │                   └── kubernetescluster.go
        ├── informers
        │   └── externalversions
        │       ├── ecs
        │       │   ├── interface.go
        │       │   └── v1
        │       │       ├── interface.go
        │       │       └── kubernetescluster.go
        │       ├── factory.go
        │       ├── generic.go
        │       └── internalinterfaces
        │           └── factory_interfaces.go
        └── listers
            └── ecs
                └── v1
                    ├── expansion_generated.go
                    └── kubernetescluster.go

21 directories, 26 files
```



CRD 以及生成的代码见：[kubernetes-operator](https://github.com/gosoon/kubernetes-operator)。



### 总结

本问讲述了如何使用 code-generator 生成代码，要使用自定义 controller 代码生成是最开始的一步，下文会继续讲述自定义 controller 的详细步骤，感兴趣的可以关注笔者 github 的项目 [kubernetes-operator](https://github.com/gosoon/kubernetes-operator)。



参考：

https://github.com/kubernetes/sample-controller

https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/

https://hexo.do1618.com/2018/04/04/Kubernetes-Deep-Dive-Code-Generation-for-CustomResources/
