---
title: k8s 中定时任务的实现
date: 2019-02-16 21:59:30
tags: ["crontab","wait","k8s"]
type: "wait"

---
k8s 中有许多优秀的包都可以在平时的开发中借鉴与使用，比如，任务的定时轮询、高可用的实现、日志处理、缓存使用等都是独立的包，可以直接引用。本篇文章会介绍 k8s 中定时任务的实现，k8s 中定时任务都是通过 wait 包实现的，wait 包在 k8s 的多个组件中都有用到，以下是 wait 包在 kubelet 中的几处使用：


```		
func run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies, stopCh <-chan struct{}) (err error) {
		...
		// kubelet 每5分钟一次从 apiserver 获取证书
		closeAllConns, err := kubeletcertificate.UpdateTransport(wait.NeverStop, clientConfig, clientCertificateManager, 5*time.Minute)
		if err != nil {
			return err
		}
		
		closeAllConns, err := kubeletcertificate.UpdateTransport(wait.NeverStop, clientConfig, clientCertificateManager, 5*time.Minute)
		if err != nil {
			return err
		}
		...
}

...

func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration,   kubeDeps *kubelet.Dependencies, enableServer bool) {
    // 持续监听 pod 的变化
    go wait.Until(func() {
        k.Run(podCfg.Updates())
    }, 0, wait.NeverStop)
    ...
}
```

golang 中可以通过 time.Ticker 实现定时任务的执行，但在 k8s 中用了更原生的方式，使用 time.Timer 实现的。time.Ticker 和 time.Timer 的使用区别如下：

- ticker 只要定义完成，从此刻开始计时，不需要任何其他的操作，每隔固定时间都会自动触发。
- timer 定时器是到了固定时间后会执行一次，仅执行一次
- 如果 timer 定时器要每隔间隔的时间执行，实现 ticker 的效果，使用 `func (t *Timer) Reset(d Duration) bool`
 
一个示例：


```
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg sync.WaitGroup

	timer1 := time.NewTimer(2 * time.Second)
	ticker1 := time.NewTicker(2 * time.Second)

	wg.Add(1)
	go func(t *time.Ticker) {
		defer wg.Done()
		for {
			<-t.C
			fmt.Println("exec ticker", time.Now().Format("2006-01-02 15:04:05"))
		}
	}(ticker1)

	wg.Add(1)
	go func(t *time.Timer) {
		defer wg.Done()
		for {
			<-t.C
			fmt.Println("exec timer", time.Now().Format("2006-01-02 15:04:05"))
			t.Reset(2 * time.Second)
		}
	}(timer1)
	
	wg.Wait()
}

```

### 一、wait 包中的核心代码


核心代码（k8s.io/apimachinery/pkg/util/wait/wait.go）：


```
func JitterUntil(f func(), period time.Duration, jitterFactor float64, sliding bool, stopCh <-chan struct{}) {
	var t *time.Timer
	var sawTimeout bool

	for {
		select {
		case <-stopCh:
			return
		default:
		}

		jitteredPeriod := period
		if jitterFactor > 0.0 {
			jitteredPeriod = Jitter(period, jitterFactor)
		}

		if !sliding {
			t = resetOrReuseTimer(t, jitteredPeriod, sawTimeout)
		}

		func() {
			defer runtime.HandleCrash()
			f()
		}()

		if sliding {
			t = resetOrReuseTimer(t, jitteredPeriod, sawTimeout)
		}

		select {
		case <-stopCh:
			return
		case <-t.C:
			sawTimeout = true
		}
	}
}

...

func resetOrReuseTimer(t *time.Timer, d time.Duration, sawTimeout bool) *time.Timer {
    if t == nil {
        return time.NewTimer(d)
    }
    if !t.Stop() && !sawTimeout {
        <-t.C
    }
    t.Reset(d)
    return t
}
```

几个关键点的说明：

- 1、如果 sliding 为 true，则在 f() 运行之后计算周期。如果为 false，那么 period 包含 f() 的执行时间。
- 2、在 golang 中 select 没有优先级选择，为了避免额外执行 f(),在每次循环开始后会先判断 stopCh chan。

k8s 中 wait 包其实是对 time.Timer 做了一层封装实现。

### 二、wait 包常用的方法

##### 1、定期执行一个函数，永不停止，可以使用 Forever 方法：

func Forever(f func(), period time.Duration)

##### 2、在需要的时候停止循环，那么可以使用下面的方法，增加一个用于停止的 chan 即可，方法定义如下：

func Until(f func(), period time.Duration, stopCh <-chan struct{})

上面的第三个参数 stopCh 就是用于退出无限循环的标志，停止的时候我们 close 掉这个 chan 就可以了。

##### 3、有时候，我们还会需要在运行前去检查先决条件，在条件满足的时候才去运行某一任务，这时候可以使用 Poll 方法：

func Poll(interval, timeout time.Duration, condition ConditionFunc)

这个函数会以 interval 为间隔，不断去检查 condition 条件是否为真，如果为真则可以继续后续处理；如果指定了 timeout 参数，则该函数也可以只常识指定的时间。

##### 4、PollUntil 方法和上面的类似，但是没有 timeout 参数，多了一个 stopCh 参数，如下所示：

PollUntil(interval time.Duration, condition ConditionFunc, stopCh <-chan struct{}) error

此外还有 PollImmediate 、 PollInfinite 和 PollImmediateInfinite 方法。

### 三、总结

本篇文章主要讲了 k8s 中定时任务的实现与对应包（wait）中方法的使用。通过阅读 k8s 的源代码，可以发现 k8s 中许多功能的实现也都是我们需要在平时工作中用的，其大部分包的性能都是经过大规模考验的，通过使用其相关的工具包不仅能学到大量的编程技巧也能避免自己造轮子。

