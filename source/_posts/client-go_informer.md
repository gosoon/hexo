---
title: kubernetes 中 informer 的使用
date: 2019-05-17 11:00:30
tags: ["client-go","informer"]
type: "client-go"
---

#### 一、kubernetes 集群的几种访问方式

在实际开发过程中，若想要获取 kubernetes 中某个资源（比如 pod）的所有对象，可以使用 kubectl、k8s REST API、client-go(ClientSet、Dynamic Client、RESTClient 三种方式) 等多种方式访问 k8s 集群获取资源。在笔者的开发过程中，最初都是直接调用 k8s 的 REST API 来获取的，使用 `kubectl get pod -v=9` 可以直接看到调用 k8s 的接口，然后在程序中直接访问还是比较方便的。但是随着集群规模的增长或者从国内获取海外 k8s 集群的数据，直接调用 k8s 接口获取所有 pod 还是比较耗时，这个问题有多种解决方法，最初是直接使用 k8s 原生的 watch 接口来获取的，下面是一个伪代码：

```
const (
	ADDED    string = "ADDED"
	MODIFIED string = "MODIFIED"
	DELETED  string = "DELETED"
	ERROR    string = "ERROR"
)

type Event struct {
	Type   string          `json:"type"`
	Object json.RawMessage `json:"object"`
}

func main() {
	resp, err := http.Get("http://apiserver:8080/api/v1/watch/pods?watch=yes")
	if err != nil {
		// ...
	}
	decoder := json.NewDecoder(resp.Body)
	for {
		var event Event
		err = decoder.Decode(&event)
		if err != nil {
			// ...
		}
		switch event.Type {
		case ADDED, MODIFIED:
			// ...
		case DELETED:
			// ...
		case ERROR:
			// ...
		}
	}
}
```


调用 watch 接口后会先将所有的对象 list 一次，然后 apiserver 会将变化的数据推送到 client 端，可以看到每次对于 watch 到的事件都需要判断后进行处理，然后将处理后的结果写入到本地的缓存中，原生的 watch 操作还是非常麻烦的。后来了解到官方推出一个客户端工具 client-go ，client-go 中的 Informer 对 watch 操作做了封装，使用起来非常方便，下面会主要介绍一下 client-go 的使用。

#### 二、Informer 的机制

cient-go 是从 k8s 代码中抽出来的一个客户端工具，Informer 是 client-go 中的核心工具包，已经被 kubernetes 中众多组件所使用。所谓 Informer，其实就是一个带有本地缓存和索引机制的、可以注册 EventHandler 的 client，本地缓存被称为 Store，索引被称为 Index。使用 informer 的目的是为了减轻 apiserver 数据交互的压力而抽象出来的一个 cache 层, 客户端对 apiserver 数据的 "读取" 和 "监听" 操作都通过本地 informer 进行。Informer 实例的`Lister()`方法可以直接查找缓存在本地内存中的数据。

Informer 的主要功能：

- 同步数据到本地缓存
- 根据对应的事件类型，触发事先注册好的 ResourceEventHandler

##### 1、Informer 中几个组件的作用
Informer 中主要有 Reflector、Delta FIFO Queue、Local Store、WorkQueue 几个组件。以下是 Informer 的工作流程图。

![Informer 组件](http://cdn.tianfeiyu.com/informer-1.png)



根据流程图来解释一下 Informer 中几个组件的作用：

- Reflector：称之为反射器，实现对 apiserver 指定类型对象的监控(ListAndWatch)，其中反射实现的就是把监控的结果实例化成具体的对象，最终也是调用 Kubernetes 的 List/Watch API；

- DeltaIFIFO Queue：一个增量队列，将 Reflector 监控变化的对象形成一个 FIFO 队列，此处的 Delta 就是变化；

- LocalStore：就是 informer 的 cache，这里面缓存的是 apiserver 中的对象(其中有一部分可能还在DeltaFIFO 中)，此时使用者再查询对象的时候就直接从 cache 中查找，减少了 apiserver 的压力，LocalStore 只会被 Lister 的 List/Get 方法访问。

- WorkQueue：DeltaIFIFO 收到时间后会先将时间存储在自己的数据结构中，然后直接操作 Store 中存储的数据，更新完 store 后 DeltaIFIFO 会将该事件 pop 到 WorkQueue 中，Controller 收到 WorkQueue  中的事件会根据对应的类型触发对应的回调函数。


##### 2、Informer 的工作流程

- Informer 首先会 list/watch apiserver，Informer 所使用的 Reflector 包负责与 apiserver 建立连接，Reflector 使用 ListAndWatch 的方法，会先从 apiserver 中 list 该资源的所有实例，list 会拿到该对象最新的 resourceVersion，然后使用 watch 方法监听该 resourceVersion 之后的所有变化，若中途出现异常，reflector 则会从断开的 resourceVersion 处重现尝试监听所有变化，一旦该对象的实例有创建、删除、更新动作，Reflector 都会收到"事件通知"，这时，该事件及它对应的 API 对象这个组合，被称为增量（Delta），它会被放进 DeltaFIFO 中。
- Informer 会不断地从这个 DeltaFIFO 中读取增量，每拿出一个对象，Informer 就会判断这个增量的时间类型，然后创建或更新本地的缓存，也就是 store。
- 如果事件类型是 Added（添加对象），那么 Informer 会通过 Indexer 的库把这个增量里的 API 对象保存到本地的缓存中，并为它创建索引，若为删除操作，则在本地缓存中删除该对象。
- DeltaFIFO 再 pop 这个事件到 controller 中，controller 会调用事先注册的 ResourceEventHandler 回调函数进行处理。
- 在 ResourceEventHandler 回调函数中，其实只是做了一些很简单的过滤，然后将关心变更的 Object 放到 workqueue 里面。
- Controller 从 workqueue 里面取出 Object，启动一个 worker 来执行自己的业务逻辑，业务逻辑通常是计算目前集群的状态和用户希望达到的状态有多大的区别，然后孜孜不倦地让 apiserver 将状态演化到用户希望达到的状态，比如为 deployment 创建新的 pods，或者是扩容/缩容 deployment。
- 在worker中就可以使用 lister 来获取 resource，而不用频繁的访问 apiserver，因为 apiserver 中 resource 的变更都会反映到本地的 cache 中。

  
Informer 在使用时需要先初始化一个 InformerFactory，目前主要推荐使用的是 SharedInformerFactory，Shared 指的是在多个 Informer 中共享一个本地 cache。

Informer 中的 ResourceEventHandler  函数有三种：

```
// ResourceEventHandlerFuncs is an adaptor to let you easily specify as many or
// as few of the notification functions as you want while still implementing
// ResourceEventHandler.
type ResourceEventHandlerFuncs struct {
    AddFunc    func(obj interface{})
    UpdateFunc func(oldObj, newObj interface{})
    DeleteFunc func(obj interface{})
}
```

这三种函数的处理逻辑是用户自定义的，在初始化 controller 时注册完 ResourceEventHandler 后，一旦该对象的实例有创建、删除、更新三中操作后就会触发对应的 ResourceEventHandler。


#### 三、Informer 使用示例

在实际的开发工作中，Informer 主要用在两处：
- 在访问 k8s apiserver 的客户端作为一个 client 缓存对象使用；
- 在一些自定义 controller 中使用，比如 operator 的开发；


#### 1、下面是一个作为 client 的使用示例：


```
package main

import (
	"flag"
	"fmt"
	"log"
	"path/filepath"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/util/runtime"

	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
)

func main() {
	var kubeconfig *string
	if home := homedir.HomeDir(); home != "" {
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	}
	flag.Parse()

	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}

	// 初始化 client
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Panic(err.Error())
	}

	stopper := make(chan struct{})
	defer close(stopper)
	
	// 初始化 informer
	factory := informers.NewSharedInformerFactory(clientset, 0)
	nodeInformer := factory.Core().V1().Nodes()
	informer := nodeInformer.Informer()
	defer runtime.HandleCrash()
	
	// 启动 informer，list & watch
	go factory.Start(stopper)
	
	// 从 apiserver 同步资源，即 list 
	if !cache.WaitForCacheSync(stopper, informer.HasSynced) {
		runtime.HandleError(fmt.Errorf("Timed out waiting for caches to sync"))
		return
	}

	// 使用自定义 handler
	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    onAdd,
		UpdateFunc: func(interface{}, interface{}) { fmt.Println("update not implemented") }, // 此处省略 workqueue 的使用
		DeleteFunc: func(interface{}) { fmt.Println("delete not implemented") },
	})
	
	// 创建 lister
	nodeLister := nodeInformer.Lister()
	// 从 lister 中获取所有 items
	nodeList, err := nodeLister.List(labels.Everything())
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println("nodelist:", nodeList)
	<-stopper
}

func onAdd(obj interface{}) {
	node := obj.(*corev1.Node)
	fmt.Println("add a node:", node.Name)
}
```

Shared指的是多个 lister 共享同一个cache，而且资源的变化会同时通知到cache和 listers。这个解释和上面图所展示的内容的是一致的，cache我们在Indexer的介绍中已经分析过了，lister 指的就是OnAdd、OnUpdate、OnDelete 这些回调函数背后的对象。



#### 2、以下是作为 controller 使用的一个整体工作流程

(1) 创建一个控制器
- 为控制器创建 workqueue
- 创建 informer, 为 informer 添加 callback 函数，创建 lister

(2) 启动控制器
- 启动 informer
- 等待本地 cache sync 完成后， 启动 workers

(3) 当收到变更事件后，执行 callback 
- 等待事件触发
- 从事件中获取变更的 Object
- 做一些必要的检查
- 生成 object key，一般是 namespace/name 的形式
- 将 key 放入 workqueue 中

(4) worker loop
- 等待从 workqueue 中获取到 item，一般为 object key
- 用 object key 通过 lister 从本地 cache 中获取到真正的 object 对象
- 做一些检查
- 执行真正的业务逻辑
- 处理下一个 item


下面是自定义 controller 使用的一个参考：

```
var (
    masterURL  string
    kubeconfig string
)

func init() {
    flag.StringVar(&kubeconfig, "kubeconfig", "", "Path to a kubeconfig. Only required if out-of-cluster.")
    flag.StringVar(&masterURL, "master", "", "The address of the Kubernetes API server. Overrides any value in kubeconfig. Only required if out-of-cluster.")
}

func main() {
    flag.Parse()

    stopCh := signals.SetupSignalHandler()

    cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
    if err != nil {
        glog.Fatalf("Error building kubeconfig: %s", err.Error())
    }

    kubeClient, err := kubernetes.NewForConfig(cfg)
    if err != nil {
        glog.Fatalf("Error building kubernetes clientset: %s", err.Error())
    }

    // 所谓 Informer，其实就是一个带有本地缓存和索引机制的、可以注册 EventHandler 的 client
    // informer watch apiserver,每隔 30 秒 resync 一次(list)
    kubeInformerFactory := informers.NewSharedInformerFactory(kubeClient, time.Second*30)

    controller := controller.NewController(kubeClient, kubeInformerFactory.Core().V1().Nodes())

    //  启动 informer
    go kubeInformerFactory.Start(stopCh)

	 // start controller 
    if err = controller.Run(2, stopCh); err != nil {
        glog.Fatalf("Error running controller: %s", err.Error())
    }
}


// NewController returns a new network controller
func NewController(
    kubeclientset kubernetes.Interface,
    networkclientset clientset.Interface,
    networkInformer informers.NetworkInformer) *Controller {

    // Create event broadcaster
    // Add sample-controller types to the default Kubernetes Scheme so Events can be
    // logged for sample-controller types.
    utilruntime.Must(networkscheme.AddToScheme(scheme.Scheme))
    glog.V(4).Info("Creating event broadcaster")
    eventBroadcaster := record.NewBroadcaster()
    eventBroadcaster.StartLogging(glog.Infof)
    eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{Interface: kubeclientset.CoreV1().Events("")})
    recorder := eventBroadcaster.NewRecorder(scheme.Scheme, corev1.EventSource{Component: controllerAgentName})

    controller := &Controller{
        kubeclientset:    kubeclientset,
        networkclientset: networkclientset,
        networksLister:   networkInformer.Lister(),
        networksSynced:   networkInformer.Informer().HasSynced,
        workqueue:        workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "Networks"),
        recorder:         recorder,
    }

    glog.Info("Setting up event handlers")
    // Set up an event handler for when Network resources change
    networkInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: controller.enqueueNetwork,
        UpdateFunc: func(old, new interface{}) {
            oldNetwork := old.(*samplecrdv1.Network)
            newNetwork := new.(*samplecrdv1.Network)
            if oldNetwork.ResourceVersion == newNetwork.ResourceVersion {
                // Periodic resync will send update events for all known Networks.
                // Two different versions of the same Network will always have different RVs.
                return
            }
            controller.enqueueNetwork(new)
        },
        DeleteFunc: controller.enqueueNetworkForDelete,
    })

    return controller
}

```
自定义 controller 的详细使用方法可以参考：[k8s-controller-custom-resource](https://github.com/resouer/k8s-controller-custom-resource)



#### 四、使用中的一些问题

##### 1、Informer 二级缓存中的同步问题

虽然 Informer 和 Kubernetes 之间没有 resync 机制，但 Informer 内部的这两级缓存 DeltaIFIFO 和 LocalStore 之间会存在 resync 机制，k8s 中 kube-controller-manager 的 StatefulSetController 中使用了两级缓存的 resync 机制（如下图所示），我们在生产环境中发现 sts 创建后过了很久 pod 才会创建，主要是由于 StatefulSetController 的两级缓存之间 30s 会同步一次，由于  StatefulSetController watch 到变化后就会把对应的 sts 放入 DeltaIFIFO 中，且每隔30s会把 LocalStore 中全部的 sts 重新入一遍 DeltaIFIFO，入队时会做一些处理，过滤掉一些不需要重复入队列的 sts，若间隔的 30s 内没有处理完队列中所有的 sts，则待处理队列中始终存在未处理完的 sts，并且在同步过程中产生的 sts 会加的队列的尾部，新加入队尾的 sts 只能等到前面的 sts 处理完成（也就是 resync 完成）才会被处理，所以导致的现象就是 sts 创建后过了很久 pod 才会创建。

优化的方法就是去掉二级缓存的同步策略（将 setInformer.Informer().AddEventHandlerWithResyncPeriod() 改为 informer.AddEventHandler()）或者调大同步周期，但是在研究 kube-controller-manager 其他 controller 时发现并不是所有的 controller 都有同步策略，社区也有相关的 issue 反馈了这一问题，[Remove resync period for sset controller](https://github.com/kubernetes/kubernetes/pull/75622)，社区也会在以后的版本中去掉两级缓存之间的 resync 策略。

`k8s.io/kubernetes/pkg/controller/statefulset/stateful_set.go`

![kube-controller-manager sts controller](http://cdn.tianfeiyu.com/informer-2.png)



##### 2、使用 Informer 如何监听所有资源对象？

一个 Informer 实例只能监听一种 resource，每个 resource 需要创建对应的 Informer 实例。



##### 3、为什么不是使用 workqueue？

建议使用 RateLimitingQueue，它相比普通的 workqueue 多了以下的功能: 

- 限流：可以限制一个 item 被 reenqueued 的次数。
- 防止 hot loop：它保证了一个 item 被 reenqueued 后，不会马上被处理。



#### 五、总结

本文介绍了 client-go 包中核心组件 Informer 的原理以及使用方法，Informer 主要功能是缓存对象到本地以及根据对应的事件类型触发已注册好的 ResourceEventHandler，其主要用在访问 k8s apiserver 的客户端和 operator 中。




参考：

[如何用 client-go 拓展 Kubernetes 的 API](https://mp.weixin.qq.com/s?__biz=MzU1OTAzNzc5MQ==&mid=2247484052&idx=1&sn=cec9f4a1ee0d21c5b2c51bd147b8af59&chksm=fc1c2ea4cb6ba7b283eef5ac4a45985437c648361831bc3e6dd5f38053be1968b3389386e415&scene=21#wechat_redirect)

https://www.kubernetes.org.cn/2693.html

[Kubernetes 大咖秀徐超《使用 client-go 控制原生及拓展的 Kubernetes API》](https://studygolang.com/articles/9270)

[Use prometheus conventions for workqueue metrics](https://github.com/kubernetes/kubernetes/issues/71165)

[深入浅出kubernetes之client-go的workqueue](https://blog.csdn.net/weixin_42663840/article/details/81482553#%E9%99%90%E9%80%9F%E9%98%9F%E5%88%97)

<https://gianarb.it/blog/kubernetes-shared-informer>

[理解 K8S 的设计精髓之 List-Watch机制和Informer模块](https://zhuanlan.zhihu.com/p/59660536)

<https://ranler.org/notes/file/528>

[Kubernetes Client-go Informer 源码分析](https://yq.aliyun.com/articles/688485)
