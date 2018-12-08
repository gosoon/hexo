---
title: etcd 备份与恢复
date: 2017-03-02 18:04:00
type: "etcd"

---

**[etcd](https://github.com/coreos/etcd)** 是一款开源的分布式一致性键值存储,由 CoreOS 公司进行维护，详细的介绍请参考官方文档。

etcd 目前最新的版本的 v3.1.1，但它的 API 又有 v3 和 v2 之分，社区通常所说的 v3 与 v2 都是指 API 的版本号。从 etcd 2.3 版本开始推出了一个实验性的全新 v3 版本 API 的实现，v2 与 v3 API 使用了不同的存储引擎，所以客户端命令也完全不同。

    # etcdctl --version
    etcdctl version: 3.0.4
    API version: 2


官方指出 etcd v2 和 v3 的数据不能混合存放，[support backup of v2 and v3 stores](https://github.com/coreos/etcd/issues/7002) 。


**特别提醒：若使用 v3 备份数据时存在 v2 的数据则不影响恢复
若使用 v2 备份数据时存在 v3 的数据则恢复失败**

### 对于 API 2 备份与恢复方法   
[官方 v2 admin guide](https://github.com/coreos/etcd/blob/master/Documentation/v2/admin_guide.md#disaster-recovery)


etcd的数据默认会存放在我们的命令工作目录中，我们发现数据所在的目录，会被分为两个文件夹中：
* snap: 存放快照数据,etcd防止WAL文件过多而设置的快照，存储etcd数据状态。

* wal: 存放预写式日志,最大的作用是记录了整个数据变化的全部历程。在etcd中，所有数据的修改在提交前，都要先写入到WAL中。


    # etcdctl backup --data-dir /home/etcd/ --backup-dir /home/etcd_backup

    # etcd -data-dir=/home/etcd_backup/  -force-new-cluster


恢复时会覆盖 snapshot 的元数据(member ID 和 cluster ID)，所以需要启动一个新的集群。

### 对于 API 3 备份与恢复方法  
[官方 v3 admin guide](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/recovery.md)

在使用 API 3 时需要使用环境变量 ETCDCTL_API 明确指定。

在命令行设置：

	# export ETCDCTL_API=3
	
备份数据：

	# etcdctl --endpoints localhost:2379 snapshot save snapshot.db

恢复：

	# etcdctl snapshot restore snapshot.db --name m3 --data-dir=/home/etcd_data

> 恢复后的文件需要修改权限为 etcd:etcd
> --name:重新指定一个数据目录，可以不指定，默认为 default.etcd
> --data-dir：指定数据目录
> 建议使用时不指定 name 但指定 data-dir，并将 data-dir 对应于 etcd 服务中配置的 data-dir

etcd 集群都是至少 3 台机器，官方也说明了集群容错为 (N-1)/2，所以备份数据一般都是用不到，但是鉴上次 gitlab 出现的问题，对于备份数据也要非常重视。 

[官方文档翻译](https://skyao.gitbooks.io/leaning-etcd3/content/documentation/op-guide/recovery.html)
