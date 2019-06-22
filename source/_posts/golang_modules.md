---
title: 使用 Go Modules 管理依赖
date: 2019-06-22 20:49:30
tags: ["go module"]
type: "go module"

---
Go Modules 是 Go 语言的一种依赖管理方式，该  feature 是在 Go 1.11 版本中出现的，由于最近在做的项目中，团队都开始使用 go module 来替代以前的 Godep，Kubernetes 也从 v1.15 开始采用 go module 来进行包管理，所以有必要了解一下 go module。go module 相比于原来的 Godep，go module 在打包、编译等多个环节上有着明显的速度优势，并且能够在任意操作系统上方便的复现依赖包，更重要的是 go module 本身的设计使得自身被其他项目引用变得更加容易，这也是 Kubernetes 项目向框架化演进的又一个重要体现。

使用 go module 管理依赖后会在项目根目录下生成两个文件  go.mod 和 go.sum。

`go.mod` 中会记录当前项目的所依赖，文件格式如下所示：

```
module github.com/gosoon/audit-webhook

go 1.12

require (
	github.com/elastic/go-elasticsearch v0.0.0
	github.com/gorilla/mux v1.7.2
	github.com/gosoon/glog v0.0.0-20180521124921-a5fbfb162a81
)
```

`go.sum`记录每个依赖库的版本和哈希值，文件格式如下所示：

```
github.com/elastic/go-elasticsearch v0.0.0 h1:Pd5fqOuBxKxv83b0+xOAJDAkziWYwFinWnBO0y+TZaA=
github.com/elastic/go-elasticsearch v0.0.0/go.mod h1:TkBSJBuTyFdBnrNqoPc54FN0vKf5c04IdM4zuStJ7xg=
github.com/gorilla/mux v1.7.2 h1:zoNxOV7WjqXptQOVngLmcSQgXmgk4NMz1HibBchjl/I=
github.com/gorilla/mux v1.7.2/go.mod h1:1lud6UwP+6orDFRuTfBEV8e9/aOM/c4fVVCaMa2zaAs=
github.com/gosoon/glog v0.0.0-20180521124921-a5fbfb162a81 h1:JP0LU0ajeawW2xySrbhDqtSUfVWohZ505Q4LXo+hCmg=
github.com/gosoon/glog v0.0.0-20180521124921-a5fbfb162a81/go.mod h1:1e0N9vBl2wPF6qYa+JCRNIZnhxSkXkOJfD2iFw3eOfg=
```

#### 一、如何启用 go module 功能

(1) go 版本 >= v1.11

(2) 设置`GO111MODULE`环境变量

要使用`go module` 首先要设置`GO111MODULE=on`，`GO111MODULE` 有三个值，off、on、auto，off 和 on 即关闭和开启，auto 则会根据当前目录下是否有 go.mod 文件来判断是否使用 modules 功能。无论使用哪种模式，module 功能默认不在 GOPATH 目录下查找依赖文件，所以使用 modules 功能时请设置好代理。

在使用 go module 时，将 `GO111MODULE` 全局环境变量设置为 off，在需要使用的时候再开启，避免在已有项目中意外引入  go module。

```
$ echo export GO111MODULE=off >> ~/.zshrc
or
$ echo export GO111MODULE=off >> ~/.bashrc
```

go mod 命令的使用：

```
download    download modules to local cache (下载依赖的module到本地cache))
edit        edit go.mod from tools or scripts (编辑go.mod文件)
graph       print module requirement graph (打印模块依赖图))
init        initialize new module in current directory (在当前文件夹下初始化一个新的module, 创建go.mod文件))
tidy        add missing and remove unused modules (增加丢失的module，去掉未使用的module)
vendor      make vendored copy of dependencies (将依赖复制到vendor下)
verify      verify dependencies have expected content (校验依赖)
why         explain why packages or modules are needed (解释为什么需要依赖)
```

#### 二、使用 go module 功能

对于新建项目使用 go module：

```
$ export GO111MODULE=on
	
$ go mod init github.com/you/hello
	
...
// go build 会将项目的依赖添加到 go.mod 中
$ go build 
```



对于已有项目要改为使用 go module：

```
$ export GO111MODULE=on

// 创建一个空的 go.mod 文件
$ go mod init .

// 查找依赖并记录在 go.mod 文件中
$ go get ./...

```

> go.mod 文件必须要提交到 git 仓库，但 go.sum 文件可以不用提交到 git 仓库(gi t忽略文件 .gitignore 中设置一下)。


#### 三、项目的打包

首先需要使用 `go mod vendor` 将项目所有的依赖下载到本地 vendor 目录中然后进行编译，下面是一个参考： 

```
#!/bin/bash

export GO111MODULE="on"
export GOPROXY="https://goproxy.io"
export CGO_ENABLED="0"
export GOOS="linux"
export GOARCH=amd64

go mod vendor
go build -ldflags "-s -w" -a -installsuffix cgo -o audit-webhook .
```



#### 四、注意事项

1、依赖下载

go module 默认不在 GOPATH 目录下查找依赖文件，其首先会在`$GOPATH/pkg/mod`中查找有没有所需要的依赖，没有的直接会进行下载。可以使用 `go mod download`下载好所需要的依赖，依赖默认会下载到`$GOPATH/pkg/mod`中，其他项目也会使用缓存的 module。

2、国内无法访问的依赖

使用 Go 的其他包管理工具 godep、govendor、glide、dep 等都避免不了翻墙的问题，Go Modules 也是一样，但在`go.mod`中可以使用`replace`将特定的库替换成其他库：

```
replace (
	golang.org/x/text v0.3.0 => github.com/golang/text v0.3.0
)
```

或者也可以在其他机器上使用 `go mod download`下载好所需要的依赖，然后再传输到本机。


参考：

https://github.com/kubernetes/enhancements/blob/master/keps/sig-architecture/2019-03-19-go-modules.md

https://blog.golang.org/using-go-modules
