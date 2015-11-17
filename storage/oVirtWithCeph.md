# 在 oVirt 中整合 Cinder

对于 OpenStack 了解甚少，刚开始看 Cinder 和 Ceph，真的是一头雾水。那么，就从官文看起吧！

## 先说说 Ceph

#### [Ceph Object Storage](http://docs.ceph.com/docs/master/glossary/#term-ceph-object-storage)：你可以称它为对象存储“产品”，服务或者一个功能。它主要包含了以下两个内容：

* [Ceph Storage Cluster](http://docs.ceph.com/docs/master/rados/)：是整个 Ceph 部署中的基础，基于 [RADOS](http://ceph.com/papers/weil-rados-pdsw07.pdf)，是核心存储软件之一，这里存储了用户的数据(MON + OSD)。部署 Ceph 存储集群的时候需要具备两种类型的后台程序：[OSD](http://docs.ceph.com/docs/master/man/8/ceph-osd/) —— 将数据以对象的形式保存在存储节点上；& [MON](http://docs.ceph.com/docs/v0.71/man/8/ceph-mon/) —— 用于维护集群映射的原版拷贝。一个 Ceph 存储集群可以包含上千的存储节点。为了满足数据的拷贝，即便是 Ceph 存储集群的最小配置也要包含至少一个 MON 和两个 OSD 后台程序。
   * [Ceph Monitor](http://docs.ceph.com/docs/v0.79/rados/operations/monitoring/)：Ceph Monitor 会持续更新 cluster 状态的 map，包括以下四个 map。Ceph 还维护了 Ceph Monitor，Ceph OSD 以及 Placement Group 中每次状态发生改变时的历史记录。
      * [monitor map]()
      * [OSD map]()
      * [Placement Group map](http://docs.ceph.com/docs/v0.69/rados/operations/placement-groups/)
      * [CRUSH map](http://docs.ceph.com/docs/master/rados/operations/crush-map/)：
   * [Ceph OSD Daemon](http://docs.ceph.com/docs/master/man/8/ceph-osd/)：用于存储数据；处理数据的同步，恢复数据，保持数据连续性(backfilling 是回填的意思，具体意思参考了[这里](http://www.aussiestockforums.com/forums/showthread.php?t=19759))以及使数据达到平衡的状态；在每次 Heartbeat 的时候对其它 Ceph OSD 后台程序进行检查从而为 Ceph Monitor 提供一些监控数据。如果集群为您的数据做了两份拷贝，那 Ceph Storage Cluster 需要至少两个 Ceph OSD 后台程序才能达到 active + clean 的状态（**什么是 active + clean 状态..o.O**）。


* [Ceph Object Gateway](http://docs.ceph.com/docs/master/radosgw/)：是构建在 `librados` 之上的对象存储接口，用来给应用程序提供 Ceph 存储集群的 RESTful gateway.这里支持以下两种接口；Ceph Object Storage 利用 Ceph Object Gateway 后台程序(**[radosgw](http://docs.ceph.com/docs/v0.69/man/8/radosgw/)**)，radosgw 是用于和 Ceph Storage Cluster 进行交互的 FastCGI 模块。
   * **S3-compatible**：兼容了 Amazon S3 RESTful API 的接口提供了对象存储的功能。
   * **Swift-compatible**：兼容了 OpenStack Swift API 的接口提供了对想存储的功能。

#### [Ceph Block Device](http://docs.ceph.com/docs/master/rbd/rbd/)：块就是一个字节序列（例如，一个 512 字节的数据块）。那么基于块的存储接口就是用传统的旋转存储接口（例如：硬盘，CD，软盘甚至是传统 9-track 磁带）存储数据最常见的方式。块设备的普遍存在使得一个虚拟块设备成为能够与像 Ceph 这样的大量数据存储系统进行通信的最佳候选者。Ceph 块设备是精简配置的，Ceph 块设备利用 [RADOS](http://docs.ceph.com/docs/giant/man/8/rados/)功能，例如快照，

#### [Ceph Filesystem](http://docs.ceph.com/docs/master/cephfs/)（Ceph FS）是一个兼容了 POSIX 的文件系统，它利用 Ceph Storage Cluster 来存储数据。Ceph 文件系统

一些术语：

* [librados](http://docs.ceph.com/docs/giant/rados/api/librados-intro/)：提供了基础服务 —— 允许 Ceph 在一个存储系统中提供对象存储，块存储和文件存储。然而，你除了可以使用 RESTful，块或者 POSIX 接口外，还有另外一种选择，librados API 允许你基于 RADOS 为 Ceph Storage Cluster 自定义接口。librados API 允许你和 Ceph Storage Cluster 中的两类后台程序进行交互。
   * Ceph Monitor —— 维护集群映射的原版拷贝。
   * Ceph OSD Daemon —— 将数据以对象的形式存储到存储节点。
* replication： 计算机领域给出的定义是分享数据，确保数据在硬件或者软件组件上保持一致性，从而提高了数据的可依赖性，容错性和可读性。IBM 给出的[解释](https://www-01.ibm.com/software/data/replication/)
* [recovery](https://en.wikipedia.org/wiki/Data_recovery)：
* backfilling：(http://www.aussiestockforums.com/forums/showthread.php?t=19759)
* [rebalancing](http://drbd.linbit.com/users-guide-9.0/s-rebalance-workflow.html)
* active + clean：
* [heartbeat](http://linux-ha.org/wiki/Heartbeat)：
