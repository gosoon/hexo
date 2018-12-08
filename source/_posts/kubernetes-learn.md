---
title: kubernetes 学习笔记
date: 2017-02-12 22:58:00
type: "kubernetes"

---

1 月初办理了入职手续，所在的团队是搞私有云的，目前只有小规模的应用，所采用 **kubernetes + docker** 技术栈，年前所做的事情也不算多，熟悉了 kubernetes 的架构，自己搭建单机版的 kubernetes，以及在程序中调用 kubernetes 的 `API` 进行某些操作。


## 1，kubernetes 搭建

[kubernetes](https://github.com/kubernetes/kubernetes) 是 google 的一个开源软件，其社区活跃量远超 **Mesos，Coreos** 的，若想深入学习建议参考**《kubernetes 权威指南》**，我们团队的人都是从这本书学起的，作为一个新技术，会踩到的坑非常多，以下提及的是我学习过程中整理的部分资料。


![kubernetes 架构图](http://upload-images.jianshu.io/upload_images/1262158-e050e035d6fa64ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


kubernetes 是一个分布式系统，所以它有多个组件，并且需要安装在多个节点，一般来说有三个节点，etcd，master 和 minion，但是每个节点却又有多台机器，etcd 作为高性能存储服务，一般独立为一个节点，当然容错是必不可少的，官方建议集群使用奇数个节点，我们的线下集群使用 3 个节点。etcd 的学习可以参考 **gitbook** 上面某大神的一本书 一 [etcd3学习笔记](https://skyao.gitbooks.io/leaning-etcd3/content/documentation/leaning/)。master 端需要安装 kube-apiserver、kube-controller-manager和kube-scheduler 组件，minion 节点需要部署 kubelet、kube-proxy、docker 组件。

> 注意：内核版本 > 3.10 的系统才支持 kubernetes，所以一般安装在centos 7 上。 

etcd 节点：
	
	# yum install -y etcd 
	# systemctl start etcd  

master 节点：

	# yum install -y kubernetes-master
	# systemctl start kube-apiserver 
	# systemctl start kube-controller-manager 
	# systemctl start kube-scheduler 

minion 节点：

	# yum install -y kubernetes  docker
	# systemctl start kubelet 
	# systemctl start kube-proxy 
	# systemctl start docker 
	

## 2，kubernetes 版本升级

以前一直以为公司会追求稳定性，在软件和系统的选取方便会优先考虑稳定的版本。但是来了公司才发现，某些软件出了新版本后，若有期待的功能并且在掌控范围内都会及时更新，所以也协助过导师更新了线下集群的 minion 节点。

下面是 minion 节点的升级操作，master 节点的操作类似。首先需要下载 [kubernetes-server-linux-amd64.tar.gz](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG.md#downloads-for-v160-alpha1)  这个包，下载你所要更新到的版本。

**升级步骤**：

- 1，先关掉 docker 服务。docker 关闭后，当前节点的 pod 随之会被调度到其他节点上
- 2，备份二进制程序（kubectl,kube-proxy）
- 3，将解压后的二进制程序覆盖以前的版本
- 4，最后重新启动服务

	
	# systemctl stop docker
	# which kubectl kube-proxy 
	/usr/bin/kubectl
	/usr/bin/kube-proxy

	# cp /usr/bin/{kubectl,kube-proxy} /tmp/
	# yes | cp bin/{kubectl,kube-proxy} /usr/bin/
	
	# systemctl status {kubectl,kube-proxy}

	# systemctl start docker 


## 3，kubeconfig 使用

若你使用的 kubelet 版本为 1.4，使用 `systemctl status kubelet`  会看到这样一句话：

	--api-servers option is deprecated for kubelet, so I am now trying to deploy with simply using --kubeconfig=/etc/kubernetes/node-kubeconfig.yaml

使用 kuconfig 是为了将所有的命令行启动选项放在一个文件中方便使用。由于我们已经升级到了 1.5，所以也得升级此功能，首先需要写一个 kubeconfig 的 yaml 文件，其 [官方文档](http://kubernetes.io/docs/user-guide/kubeconfig-file/) 有格式说明， 本人已将其翻译，翻译文档见下文。

**kubeconfig** 文件示例：

		apiVersion: v1
		clusters:
		- cluster:
		    server: http://localhost:8080
		  name: local-server
		contexts:
		- context:
		    cluster: local-server
		    namespace: the-right-prefix
		    user: myself
		  name: default-context
		current-context: default-context
		kind: Config
		preferences: {}
		users:
		- name: myself
		  user:
		    password: secret
		    username: admin

---

	   # kubelet --kubeconfig=/etc/kubernetes/config --require-kubeconfig=true


kubeconfig 参数：设置 kubelet 配置文件路径，这个配置文件用来告诉 kubelet 组件 api-server 组件的位置，默认路径是。

require-kubeconfig 参数：这是一个布尔类型参数，可以设置成true 或者 false，如果设置成 true，那么表示启用 kubeconfig 参数，从 kubeconfig 参数设置的配置文件中查找 api-server 组件，如果设置成 false，那么表示使用 kubelet 另外一个参数 “api-servers” 来查找 api-server 组件位置。

关于 kubeconfig 的一个 **issue**，[Kubelet won't read apiserver from kubeconfig](https://github.com/kubernetes/kubernetes/issues/36745)。

**升级步骤**，当然前提是你的 kubelet 版本已经到了 1.5：

* 1，关闭 kubelet、kube-proxy 服务；
* 2，注释掉 `/etc/kubernetes/kubelet` 文件中下面这一行:

    `KUBELET_API_SERVER="--api-servers=http://127.0.0.1:8080"`

然后在 **KUBELET_ARGS** 中添加： 

	--kubeconfig=/etc/kubernetes/kubeconfig --require-kubeconfig=true

这里的路径是你 yaml 文件放置的路径。 

- 3，重新启动刚关掉的两个服务

---
## 4，以下为 [kubeconfig 配置官方文档](https://kubernetes.io/docs/user-guide/kubeconfig-file/)的翻译

### kubernetes 中的验证对于不同的群体可以使用不同的方法.

* 运行 kubelet 可能有的一种认证方式（即证书）。
* 用户可能有不同的认证方式（即 token）。
* 管理员可以为每个用户提供一个证书列表。
* 可能会有多个集群，但我们想在一个地方定义它们 - 使用户能够用自己的证书并重用相同的全局配置。 

因此为了在多个集群之间轻松切换，对于多个用户，定义了一个 kubeconfig 文件。

此文件包含一系列认证机制和与 nicknames 有关的群集连接信息。它还引入了认证信息元组（用户）和集群连接信息的概念，被称为上下文也与 nickname 相关联。

如果明确指定，也可以允许使用多个 kubeconfig 文件。在运行时，它们被合并加载并覆盖从命令行指定的选项（参见下面的规则）。

### 相关讨论

	http://issue.k8s.io/1755

### kubeconfig 文件的组件 

kubeconfig 文件示例：

	current-context: federal-context
	apiVersion: v1
	clusters:
	- cluster:
	    api-version: v1
	    server: http://cow.org:8080
	  name: cow-cluster
	- cluster:
	    certificate-authority: path/to/my/cafile
	    server: https://horse.org:4443
	  name: horse-cluster
	- cluster:
	    insecure-skip-tls-verify: true
	    server: https://pig.org:443
	  name: pig-cluster
	contexts:
	- context:
	    cluster: horse-cluster
	    namespace: chisel-ns
	    user: green-user
	  name: federal-context
	- context:
	    cluster: pig-cluster
	    namespace: saw-ns
	    user: black-user
	  name: queen-anne-context
	kind: Config
	preferences:
	  colors: true
	users:
	- name: blue-user
	  user:
	    token: blue-token
	- name: green-user
	  user:
	    client-certificate: path/to/my/client/cert
	    client-key: path/to/my/client/key

### 组件的解释

#### cluster

	clusters:
	- cluster:
	    certificate-authority: path/to/my/cafile
	    server: https://horse.org:4443
	  name: horse-cluster
	- cluster:
	    insecure-skip-tls-verify: true
	    server: https://pig.org:443
	  name: pig-cluster


cluster 包含 kubernetes 集群的 endpoint 数据。它包括 kubernetes apiserver 完全限定的 URL，以及集群的证书颁发机构或 insecure-skip-tls-verify：true，如果集群的服务证书未由系统信任的证书颁发机构签名。集群有一个名称（nickname），该名称用作此 kubeconfig 文件中的字典键。你可以使用 kubectl config set-cluster 添加或修改集群条目。

#### user

	users:
	- name: blue-user
	  user:
	    token: blue-token
	- name: green-user
	  user:
	    client-certificate: path/to/my/client/cert
	    client-key: path/to/my/client/key

用户定义用于向 Kubernetes 集群进行身份验证的客户端凭证。在 kubeconfig 被加载/合并之后，用户具有在用户条目列表中充当其键的名称（nickname）。可用的凭证是客户端证书，客户端密钥，令牌和用户名/密码。用户名/密码和令牌是互斥的，但客户端证书和密钥可以与它们组合。你可以使用 kubectl config set-credentials 添加或修改用户条目。

### context

	contexts:
	- context:
	    cluster: horse-cluster
	    namespace: chisel-ns
	    user: green-user
	  name: federal-context

context 定义 cluster,user,namespace 元组的名称，用来向指定的集群使用提供的认证信息和命名空间向指定的集群发送请求。
三个都是可选的，仅指定 cluster，user，namespace 中的一个也是可用的，或者指定为 none。未指定的值或命名值，在加载的 kubeconfig 中没有对应的条目（例如，如果context 在上面的 kubeconfig 文件指定为 pink-user ）将被替换为默认值。有关覆盖/合并行为，请参阅下面的加载/合并规则。你可以使用 kubectl config set-context 添加或修改上下文条目。

#### current-context

	current-context: federal-context

current-context 是 cluster,user,namespace 中的 nickname 或者 ‘key’，kubectl 在从此文件加载配置时将使用默认值。通过给 kubelett 传递 --context=CONTEXT, --cluster=CLUSTER, --user=USER, and/or --namespace=NAMESPACE 可以从命令行覆盖任何值。你可以使用 kubectl config use-context 更改当前上下文。

#### 杂项

	apiVersion: v1
	kind: Config
	preferences:
	  colors: true

apiVersion 和 kind 标识客户端要解析的版本和模式，不应手动编辑。
preferences 指定选项(和当前未使用的) kubectl preferences.

### 查看 kubeconfig 文件

kubectl config view 会显示当前的 kubeconfig 配置。默认情况下，它会显示所有加载的 kubeconfig 配置， 你可以通过 --minify 选项来过滤与 current-context 相关的设置。请参见 kubectl config view 的其他选项。

### 创建你的 kubeconfig 文件

注意，如果你通过 kube-up.sh 部署 k8s，则不需要创建 kubeconfig 文件，脚本将为你创建。

在任何情况下，可以轻松地使用此文件作为模板来创建自己的 kubeconfig 文件。

因此，让我们快速浏览上述文件的基础知识，以便可以根据需要轻松修改...

以上文件可能对应于使用--token-auth-file = tokens.csv 选项启动的 api 服务器，其中 tokens.csv文件看起来像这样：
	
	blue-user,blue-user,1
	mister-red,mister-red,2

此外，由于不同用户使用不同的验证机制，api-server 可能已经启动其他的身份验证选项（有许多这样的选项，在制作 kubeconfig 文件之前确保你理解所关心的，因为没有人需要实现所有可能的认证方案）。

* 由于 current-context 的用户是 “green-user”，因此任何使用此 kubeconfig 文件的客户端自然都能够成功登录 api-server，因为我们提供了 “green-user” 的客户端凭据。
* 类似地，我们也可以选择改变 current-context 的值为 “blue-user”。
* 
在上述情况下，“green-user” 将必须通过提供证书登录，而 “blue-user” 只需提供 token。所有的这些信息将由我们处理通过


### 加载和合并规则

加载和合并 kubeconfig 文件的规则很简单，但有很多。最终配置按照以下顺序构建：

1，从磁盘获取 kubeconfig。通过以下层次结构和合并规则完成：
如果设置了 CommandLineLocation（kubeconfig 命令行选项的值），则仅使用此文件，不合并。只允许此标志的一个实例。

否则，如果 EnvVarLocation（$KUBECONFIG 的值）可用，将其用作应合并的文件列表。根据以下规则将文件合并在一起。将忽略空文件名。文件内容不能反序列化则产生错误。设置特定值或映射密钥的第一个文件将被使用，并且值或映射密钥永远不会更改。这意味着设置CurrentContext 的第一个文件将保留其 context。也意味着如果两个文件指定 “red-user”,，则仅使用来自第一个文件的 “red-user” 的值。来自第二个 “red-user” 文件的非冲突条目也将被丢弃。

对于其他的，使用 HomeDirectoryLocation（~/.kube/config）也不会被合并。

2，此链中第一个被匹配的 context 将被使用：

* 1，命令行参数 - 命令行选项中 context 的值
* 2，合并文件中的 current-context
* 3，此段允许为空

3，确定要使用的集群信息和用户。在此处，也可能没有 context。这个链中第一次使用的会被构建。（运行两次，一次为用户，一次为集群）：

* 1，命令行参数 - user 是用户名，cluster 是集群名
* 2，如果存在 context 则使用
* 3，允许为空

4，确定要使用的实际集群信息。在此处，也可能没有集群信息。基于链构建每个集群信息（首次使用的）：

* 1，命令行参数 - server，api-version，certificate-authority 和 insecure-skip-tls-verify
* 2，如果存在集群信息并且该属性的值存在，则使用它。
* 3，如果没有 server 位置则出错。

5，确定要使用的实际用户信息。用户构建使用与集群信息相同的规则，但每个用户只能具有一种认证方法：

* 1，加载优先级为 1）命令行参数，2） kubeconfig 的用户字段
* 2，命令行参数：客户端证书，客户端密钥，用户名，密码和 token。
* 3，如果两者有冲突则失败

6，对于仍然缺失的信息，使用默认值并尽可能提示输入身份验证信息。

7，kubeconfig 文件中的所有文件引用都是相对于 kubeconfig 文件本身的位置解析的。当文件引用显示在命令行上时，它们被视为相对于当前工作目录。当路径保存在 ~/.kube/config 中时，相对路径和绝对路径被分别存储。

kubeconfig 文件中的任何路径都是相对于 kubeconfig 文件本身的位置解析的。


### 通过 kubectl config <subcommand> 操作 kubeconfig

为了更容易地操作 kubeconfig 文件，可以使用 kubectl config 的子命令。请参见 kubectl/kubectl_config.md 获取帮助。

例如：

	$ kubectl config set-credentials myself --username=admin --password=secret
	$ kubectl config set-cluster local-server --server=http://localhost:8080
	$ kubectl config set-context default-context --cluster=local-server --user=myself
	$ kubectl config use-context default-context
	$ kubectl config set contexts.default-context.namespace the-right-prefix
	$ kubectl config view

输出：

	apiVersion: v1
	clusters:
	- cluster:
	    server: http://localhost:8080
	  name: local-server
	contexts:
	- context:
	    cluster: local-server
	    namespace: the-right-prefix
	    user: myself
	  name: default-context
	current-context: default-context
	kind: Config
	preferences: {}
	users:
	- name: myself
	  user:
	    password: secret
	    username: admin

一个 kubeconfig 文件类似这样：

	apiVersion: v1
	clusters:
	- cluster:
	    server: http://localhost:8080
	  name: local-server
	contexts:
	- context:
	    cluster: local-server
	    namespace: the-right-prefix
	    user: myself
	  name: default-context
	current-context: default-context
	kind: Config
	preferences: {}
	users:
	- name: myself
	  user:
	    password: secret
	    username: admin

示例文件的命令操作：

	$ kubectl config set preferences.colors true
	$ kubectl config set-cluster cow-cluster --server=http://cow.org:8080 --api-version=v1
	$ kubectl config set-cluster horse-cluster --server=https://horse.org:4443 --certificate-authority=path/to/my/cafile
	$ kubectl config set-cluster pig-cluster --server=https://pig.org:443 --insecure-skip-tls-verify=true
	$ kubectl config set-credentials blue-user --token=blue-token
	$ kubectl config set-credentials green-user --client-certificate=path/to/my/client/cert --client-key=path/to/my/client/key
	$ kubectl config set-context queen-anne-context --cluster=pig-cluster --user=black-user --namespace=saw-ns
	$ kubectl config set-context federal-context --cluster=horse-cluster --user=green-user --namespace=chisel-ns
	$ kubectl config use-context federal-context

最后的总结：

所以，看完这些，你就可以快速开始创建自己的 kubeconfig 文件了：

* 仔细查看并了解 api-server 如何启动：了解你的安全策略后，然后才能设计 kubeconfig 文件以便于身份验证
* 将上面的代码段替换为你集群的 api-server endpoint 的信息。
* 确保 api-server 已启动，以至少向其提供一个用户（例如：green-user）凭证。当然，你必须查看 api-server 文档，以确定以目前最好的技术提供详细的身份验证信息。
