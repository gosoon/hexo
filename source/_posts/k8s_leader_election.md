---
title: kubernets 中组件高可用的实现方式
date: 2019-03-13 07:49:30
tags: ["leader-election","component"]
type: "component-HA"

---
生产环境中为了保障业务的稳定性，集群都需要高可用部署，k8s 中 apiserver 是无状态的，可以横向扩容保证其高可用，kube-controller-manager 和 kube-scheduler 两个组件通过 leader 选举保障高可用，即正常情况下 kube-scheduler 或 kube-manager-controller 组件的多个副本只有一个是处于业务逻辑运行状态，其它副本则不断的尝试去获取锁，去竞争 leader，直到自己成为leader。如果正在运行的 leader 因某种原因导致当前进程退出，或者锁丢失，则由其它副本去竞争新的 leader，获取 leader 继而执行业务逻辑。

> kubernetes 版本： v1.12 

#### 组件高可用的使用

k8s 中已经为 kube-controller-manager、kube-scheduler 组件实现了高可用，只需在每个组件的配置文件中添加 `--leader-elect=true` 参数即可启用。在每个组件的日志中可以看到 HA 相关参数的默认值：

```
I0306 19:17:14.109511  161798 flags.go:33] FLAG: --leader-elect="true"
I0306 19:17:14.109513  161798 flags.go:33] FLAG: --leader-elect-lease-duration="15s"
I0306 19:17:14.109516  161798 flags.go:33] FLAG: --leader-elect-renew-deadline="10s"
I0306 19:17:14.109518  161798 flags.go:33] FLAG: --leader-elect-resource-lock="endpoints"
I0306 19:17:14.109520  161798 flags.go:33] FLAG: --leader-elect-retry-period="2s"
```

kubernetes 中查看组件 leader 的方法：


```
$ kubectl get endpoints kube-controller-manager --namespace=kube-system -o yaml && 
  kubectl get endpoints kube-scheduler --namespace=kube-system -o yaml
```

当前组件 leader 的 hostname 会写在 annotation 的 control-plane.alpha.kubernetes.io/leader 字段里。


#### Leader Election 的实现

Leader Election 的过程本质上是一个竞争分布式锁的过程。在 Kubernetes 中，这个分布式锁是以创建 Endpoint 资源的形式进行，谁先创建了该资源，谁就先获得锁，之后会对该资源不断更新以保持锁的拥有权。



下面开始讲述 kube-controller-manager 中 leader 的竞争过程，cm 在加载及配置完参数后就开始执行 run 方法了。代码在 `k8s.io/kubernetes/cmd/kube-controller-manager/app/controllermanager.go` 中：

```
// Run runs the KubeControllerManagerOptions.  This should never exit.
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
		...
		// kube-controller-manager 的核心
		run := func(ctx context.Context) {
		rootClientBuilder := controller.SimpleControllerClientBuilder{
			ClientConfig: c.Kubeconfig,
		}
		var clientBuilder controller.ControllerClientBuilder
		if c.ComponentConfig.KubeCloudShared.UseServiceAccountCredentials {
			if len(c.ComponentConfig.SAController.ServiceAccountKeyFile) == 0 {
				// It'c possible another controller process is creating the tokens for us.
				// If one isn't, we'll timeout and exit when our client builder is unable to create the tokens.
				glog.Warningf("--use-service-account-credentials was specified without providing a --service-account-private-key-file")
			}
			clientBuilder = controller.SAControllerClientBuilder{
				ClientConfig:         restclient.AnonymousClientConfig(c.Kubeconfig),
				CoreClient:           c.Client.CoreV1(),
				AuthenticationClient: c.Client.AuthenticationV1(),
				Namespace:            "kube-system",
			}
		} else {
			clientBuilder = rootClientBuilder
		}
		controllerContext, err := CreateControllerContext(c, rootClientBuilder, clientBuilder, ctx.Done())
		if err != nil {
			glog.Fatalf("error building controller context: %v", err)
		}
		saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController
		// 初始化及启动所有的 controller
		if err := StartControllers(controllerContext, saTokenControllerInitFunc, NewControllerInitializers(controllerContext.LoopMode), unsecuredMux); err != nil {
			glog.Fatalf("error starting controllers: %v", err)
		}

		controllerContext.InformerFactory.Start(controllerContext.Stop)
		close(controllerContext.InformersStarted)

		select {}
	}

    // 如果 LeaderElect 参数未配置,说明 controller-manager 是单点启动的，
    // 则直接调用 run 方法来启动需要被启动的控制器即可。
    if !c.ComponentConfig.Generic.LeaderElection.LeaderElect {
        run(context.TODO())
        panic("unreachable")
    }

	// 如果 LeaderElect 参数配置为 true,说明 controller-manager 是以 HA 方式启动的，
	// 则执行下面的代码进行 leader 选举，选举出的 leader 会回调 run 方法。
    id, err := os.Hostname()
    if err != nil {
        return err
    }

    // add a uniquifier so that two processes on the same host don't accidentally both become active
    id = id + "_" + string(uuid.NewUUID())
    
    // 初始化资源锁
    rl, err := resourcelock.New(c.ComponentConfig.Generic.LeaderElection.ResourceLock,
        "kube-system",
        "kube-controller-manager",
        c.LeaderElectionClient.CoreV1(),
        resourcelock.ResourceLockConfig{
            Identity:      id,
            EventRecorder: c.EventRecorder,
        })
    if err != nil {
        glog.Fatalf("error creating lock: %v", err)
    }
    // 进入到选举的流程
    leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{
        Lock:          rl,
        LeaseDuration: c.ComponentConfig.Generic.LeaderElection.LeaseDuration.Duration,
        RenewDeadline: c.ComponentConfig.Generic.LeaderElection.RenewDeadline.Duration,
        RetryPeriod:   c.ComponentConfig.Generic.LeaderElection.RetryPeriod.Duration,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: run,
            OnStoppedLeading: func() {
                glog.Fatalf("leaderelection lost")
            },
        },
        WatchDog: electionChecker,
        Name:     "kube-controller-manager",
    })
    panic("unreachable")
}
```

- 1、初始化资源锁，kubernetes 中默认的资源锁使用 `endpoints`，也就是 c.ComponentConfig.Generic.LeaderElection.ResourceLock 的值为 "endpoints"，在代码中我并没有找到对 ResourceLock 初始化的地方，只看到了对该参数的说明以及日志中配置的默认值：

![](https://upload-images.jianshu.io/upload_images/1262158-402b3215eb022307.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

​在初始化资源锁的时候还传入了 EventRecorder，其作用是当 leader 发生变化的时候会将对应的 events 发送到 apiserver。


- 2、rl 资源锁被用于 controller-manager 进行 leader 的选举，RunOrDie 方法中就是 leader 的选举过程了。

- 3、Callbacks 中定义了在切换状态后需要执行的操作，当成为 leader 后会执行 OnStartedLeading 中的 run 方法，run 方法是 controller-manager 的核心，run 方法中会初始化并启动所包含资源的 controller，以下是 kube-controller-manager 中所有的 controller：

```
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	controllers := map[string]InitFunc{}
	controllers["endpoint"] = startEndpointController
	controllers["replicationcontroller"] = startReplicationController
	controllers["podgc"] = startPodGCController
	controllers["resourcequota"] = startResourceQuotaController
	controllers["namespace"] = startNamespaceController
	controllers["serviceaccount"] = startServiceAccountController
	controllers["garbagecollector"] = startGarbageCollectorController
	controllers["daemonset"] = startDaemonSetController
	controllers["job"] = startJobController
	controllers["deployment"] = startDeploymentController
	controllers["replicaset"] = startReplicaSetController
	controllers["horizontalpodautoscaling"] = startHPAController
	controllers["disruption"] = startDisruptionController
	controllers["statefulset"] = startStatefulSetController
	controllers["cronjob"] = startCronJobController
	controllers["csrsigning"] = startCSRSigningController
	controllers["csrapproving"] = startCSRApprovingController
	controllers["csrcleaner"] = startCSRCleanerController
	controllers["ttl"] = startTTLController
	controllers["bootstrapsigner"] = startBootstrapSignerController
	controllers["tokencleaner"] = startTokenCleanerController
	controllers["nodeipam"] = startNodeIpamController
	if loopMode == IncludeCloudLoops {
		controllers["service"] = startServiceController
		controllers["route"] = startRouteController
	}
	controllers["nodelifecycle"] = startNodeLifecycleController
	controllers["persistentvolume-binder"] = startPersistentVolumeBinderController
	controllers["attachdetach"] = startAttachDetachController
	controllers["persistentvolume-expander"] = startVolumeExpandController
	controllers["clusterrole-aggregation"] = startClusterRoleAggregrationController
	controllers["pvc-protection"] = startPVCProtectionController
	controllers["pv-protection"] = startPVProtectionController
	controllers["ttl-after-finished"] = startTTLAfterFinishedController

	return controllers
}
```

OnStoppedLeading 是从 leader 状态切换为 slave 要执行的操作，此方法仅打印了一条日志。



```
func RunOrDie(ctx context.Context, lec LeaderElectionConfig) {
    le, err := NewLeaderElector(lec)
    if err != nil {
        panic(err)
    }
    if lec.WatchDog != nil {
        lec.WatchDog.SetLeaderElection(le)
    }
    le.Run(ctx)
}
```

在 RunOrDie 中首先调用 NewLeaderElector 初始化了一个 LeaderElector 对象，然后执行 LeaderElector 的 run 方法进行选举。


```
func (le *LeaderElector) Run(ctx context.Context) {
	defer func() {
		runtime.HandleCrash()
		le.config.Callbacks.OnStoppedLeading()
	}()
	if !le.acquire(ctx) {
		return // ctx signalled done
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	go le.config.Callbacks.OnStartedLeading(ctx)
	le.renew(ctx)
}
```

Run 中首先会执行 acquire 尝试获取锁，获取到锁之后会回调 OnStartedLeading 启动所需要的 controller，然后会执行 renew 方法定期更新锁，保持 leader 的状态。


```
func (le *LeaderElector) acquire(ctx context.Context) bool {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	succeeded := false
	desc := le.config.Lock.Describe()
	glog.Infof("attempting to acquire leader lease  %v...", desc)
	wait.JitterUntil(func() {
		// 尝试创建或者续约资源锁
		succeeded = le.tryAcquireOrRenew()
		// leader 可能发生了改变，在 maybeReportTransition 方法中会
		// 执行相应的 OnNewLeader() 回调函数,代码中对 OnNewLeader() 并没有初始化
		le.maybeReportTransition()
		if !succeeded {
			glog.V(4).Infof("failed to acquire lease %v", desc)
			return
		}
		le.config.Lock.RecordEvent("became leader")
		glog.Infof("successfully acquired lease %v", desc)
		cancel()
	}, le.config.RetryPeriod, JitterFactor, true, ctx.Done())
	return succeeded
}
```
在 acquire 中首先初始化了一个 ctx，通过 wait.JitterUntil 周期性的去调用 le.tryAcquireOrRenew 方法来获取资源锁，直到获取为止。如果获取不到锁，则会以 RetryPeriod 为间隔不断尝试。如果获取到锁，就会关闭 ctx 通知 wait.JitterUntil 停止尝试，tryAcquireOrRenew 是最核心的方法。



```
func (le *LeaderElector) tryAcquireOrRenew() bool {
	now := metav1.Now()
	leaderElectionRecord := rl.LeaderElectionRecord{
		HolderIdentity:       le.config.Lock.Identity(),
		LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
		RenewTime:            now,
		AcquireTime:          now,
	}

	// 1、获取当前的资源锁
	oldLeaderElectionRecord, err := le.config.Lock.Get()
	if err != nil {
		if !errors.IsNotFound(err) {
			glog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
			return false
		}
		// 没有获取到资源锁，开始创建资源锁，若创建成功则成为 leader 
		if err = le.config.Lock.Create(leaderElectionRecord); err != nil {
			glog.Errorf("error initially creating leader election record: %v", err)
			return false
		}
		le.observedRecord = leaderElectionRecord
		le.observedTime = le.clock.Now()
		return true
	}

	// 2、获取资源锁后检查当前 id 是不是 leader
	if !reflect.DeepEqual(le.observedRecord, *oldLeaderElectionRecord) {
		le.observedRecord = *oldLeaderElectionRecord
		le.observedTime = le.clock.Now()
	}
	// 如果资源锁没有过期且当前 id 不是 Leader，直接返回
	if le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
		!le.IsLeader() {
		glog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
		return false
	}

	// 3、如果当前 id 是 Leader，将对应字段的时间改成当前时间，准备续租
	// 如果是非 Leader 节点则抢夺资源锁
	if le.IsLeader() {
		leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
	} else {
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
	}

	// 更新资源
    // 对于 Leader 来说，这是一个续租的过程
    // 对于非 Leader 节点（仅在上一个资源锁已经过期），这是一个更新锁所有权的过程
	if err = le.config.Lock.Update(leaderElectionRecord); err != nil {
		glog.Errorf("Failed to update lock: %v", err)
		return false
	}
	le.observedRecord = leaderElectionRecord
	le.observedTime = le.clock.Now()
	return true
}
```

上面的这个函数的主要逻辑：
- 1、获取 ElectionRecord 记录，如果没有则创建一条新的 ElectionRecord 记录，创建成功则表示获取到锁并成为 leader 了。
- 2、当获取到资源锁后开始检查其中的信息，比较当前 id 是不是 leader 以及资源锁有没有过期，如果资源锁没有过期且当前 id 不是 Leader，则直接返回。
- 3、如果当前 id 是 Leader，将对应字段的时间改成当前时间，更新资源锁进行续租。
- 4、如果当前 id 不是 Leader 但是资源锁已经过期了，则抢夺资源锁，抢夺成功则成为 leader 否则返回。


最后是 renew 方法：

```
func (le *LeaderElector) renew(ctx context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	wait.Until(func() {
		timeoutCtx, timeoutCancel := context.WithTimeout(ctx, le.config.RenewDeadline)
		defer timeoutCancel()
		// 每间隔 RetryPeriod 就执行 tryAcquireOrRenew()
        // 如果 tryAcquireOrRenew() 返回 false 说明续租失败
		err := wait.PollImmediateUntil(le.config.RetryPeriod, func() (bool, error) {
			done := make(chan bool, 1)
			go func() {
				defer close(done)
				done <- le.tryAcquireOrRenew()
			}()

			select {
			case <-timeoutCtx.Done():
				return false, fmt.Errorf("failed to tryAcquireOrRenew %s", timeoutCtx.Err())
			case result := <-done:
				return result, nil
			}
		}, timeoutCtx.Done())

		le.maybeReportTransition()
		desc := le.config.Lock.Describe()
		if err == nil {
			glog.V(4).Infof("successfully renewed lease %v", desc)
			return
		}
		// 续租失败，说明已经不是 Leader，然后程序 panic
		le.config.Lock.RecordEvent("stopped leading")
		glog.Infof("failed to renew lease %v: %v", desc, err)
		cancel()
	}, le.config.RetryPeriod, ctx.Done())
}
```
获取到锁之后定期进行更新，renew 只有在获取锁之后才会调用，它会通过持续更新资源锁的数据，来确保继续持有已获得的锁，保持自己的 leader 状态。



#### Leader Election 功能的使用

以下是一个 demo，使用 k8s 中 `k8s.io/client-go/tools/leaderelection` 进行一个演示：


```
package main

import (
	"context"
	"flag"
	"fmt"
	"os"
	"time"

	"github.com/golang/glog"
	"k8s.io/api/core/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/kubernetes/scheme"
	v1core "k8s.io/client-go/kubernetes/typed/core/v1"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/tools/leaderelection"
	"k8s.io/client-go/tools/leaderelection/resourcelock"
	"k8s.io/client-go/tools/record"
)

var (
	masterURL  string
	kubeconfig string
)

func init() {
	flag.StringVar(&kubeconfig, "kubeconfig", "", "Path to a kubeconfig. Only required if out-of-cluster.")
	flag.StringVar(&masterURL, "master", "", "The address of the Kubernetes API server. Overrides any value in kubeconfig. Only required if out-of-cluster.")

	flag.Set("logtostderr", "true")
}

func main() {
	flag.Parse()
	defer glog.Flush()

	id, err := os.Hostname()
	if err != nil {
		panic(err)
	}
	
	// 加载 kubeconfig 配置
	cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
	if err != nil {
		glog.Fatalf("Error building kubeconfig: %s", err.Error())
	}

	// 创建 kubeclient
	kubeClient, err := kubernetes.NewForConfig(cfg)
	if err != nil {
		glog.Fatalf("Error building kubernetes clientset: %s", err.Error())
	}

	// 初始化 eventRecorder
	eventBroadcaster := record.NewBroadcaster()
	eventRecorder := eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "test-1"})
	eventBroadcaster.StartLogging(glog.Infof)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})

	run := func(ctx context.Context) {
		fmt.Println("run.........")
		select {}
	}

	id = id + "_" + "1"
	rl, err := resourcelock.New("endpoints",
		"kube-system",
		"test",
		kubeClient.CoreV1(),
		resourcelock.ResourceLockConfig{
			Identity:      id,
			EventRecorder: eventRecorder,
		})
	if err != nil {
		glog.Fatalf("error creating lock: %v", err)
	}

	leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{
		Lock:          rl,
		LeaseDuration: 15 * time.Second,
		RenewDeadline: 10 * time.Second,
		RetryPeriod:   2 * time.Second,
		Callbacks: leaderelection.LeaderCallbacks{
			OnStartedLeading: run,
			OnStoppedLeading: func() {
				glog.Info("leaderelection lost")
			},
		},
		Name: "test-1",
	})
}
```

分别使用多个 hostname 同时运行后并测试 leader 切换，可以在 events 中看到 leader 切换的记录：


```
# kubectl describe endpoints test  -n kube-system
Name:         test
Namespace:    kube-system
Labels:       <none>
Annotations:  control-plane.alpha.kubernetes.io/leader={"holderIdentity":"localhost_2","leaseDurationSeconds":15,"acquireTime":"2019-03-10T08:47:42Z","renewTime":"2019-03-10T08:47:44Z","leaderTransitions":2}
Subsets:
Events:
  Type    Reason          Age   From    Message
  ----    ------          ----  ----    -------
  Normal  LeaderElection  50s   test-1  localhost_1 became leader
  Normal  LeaderElection  5s    test-2  localhost_2 became leader
```


#### 总结

本文讲述了 kube-controller-manager 使用 HA 的方式启动后 leader 选举过程的实现说明，k8s 中通过创建 endpoints 资源以及对该资源的持续更新来实现资源锁轮转的过程。但是相对于其他分布式锁的实现，普遍是直接基于现有的中间件实现，比如 redis、zookeeper、etcd 等，其所有对锁的操作都是原子性的，那 k8s 选举过程中的原子操作是如何实现的？k8s 中的原子操作最终也是通过 etcd 实现的，其在做 update 更新锁的操作时采用的是乐观锁，通过对比 resourceVersion 实现的，详细的实现下节再讲。

![api resource](https://upload-images.jianshu.io/upload_images/1262158-7ec427b37b6b6a10.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



参考文档：
[API OVERVIEW](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/)
[Simple leader election with Kubernetes and Docker](https://kubernetes.io/blog/2016/01/simple-leader-election-with-kubernetes/)


