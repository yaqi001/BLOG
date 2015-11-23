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

#### [Ceph Block Device](http://docs.ceph.com/docs/master/rbd/rbd/)：块就是一个字节序列（例如，一个 512 字节的数据块）。那么基于块的存储接口就是用传统的旋转存储接口（例如：硬盘，CD，软盘甚至是传统 9-track 磁带）存储数据的最常见的方式。块设备的普遍存在使得一个虚拟块设备成为能够与像 Ceph 这样的大量数据存储系统进行通信的最佳候选者。Ceph 块设备是精简配置且可调整大小的，还可以在一个 Ceph Cluster 上的多个 OSD 上存储带状数据。Ceph 块设备利用了 [RADOS](http://docs.ceph.com/docs/giant/man/8/rados/)功能，例如快照，复制，保证数据一致性。Ceph 的 [RADOS] 块设备（RBD）利用内核或者 [librbd] 库来和 OSD 进行通信。

#### [Ceph Filesystem](http://docs.ceph.com/docs/master/cephfs/)（Ceph FS）是一个兼容了 POSIX 的文件系统，它利用 Ceph Storage Cluster 来存储数据。Ceph 文件系统像 Ceph Block Devices 和 Ceph Object Storage 一样使用 Ceph Storage Cluster 和 S3， Swift API 或者本地邦定（librados）。要想使用 Ceph 文件系统，你的 Ceph Storage Cluster 中至少得有一个 [Ceph Metadata Server](http://docs.ceph.com/docs/v0.71/man/8/ceph-mds/)。

* [Ceph Metadata Server](http://docs.ceph.com/docs/v0.71/man/8/ceph-mds/)：MDS，Ceph 元数据软件。MDS 代表 Ceph 文件系统存储了元数据（Ceph Block Devices 和 Ceph Object Storage 则不用 MDS）。Ceph Metadata Server 能够让 POSIX 文件系统用户执行基本命令例如 `ls`，`find` 等，为 Ceph Storage Cluster 减轻了很多负担。

## 一些术语：

* [librados](http://docs.ceph.com/docs/giant/rados/api/librados-intro/)：提供了基础服务 —— 允许 Ceph 在一个存储系统中提供对象存储，块存储和文件存储。然而，你除了可以使用 RESTful，块或者 POSIX 接口外，还有另外一种选择，librados API 允许你基于 RADOS 为 Ceph Storage Cluster 自定义接口。librados API 允许你和 Ceph Storage Cluster 中的两类后台程序进行交互。
   * Ceph Monitor —— 维护集群映射的原版拷贝。
   * Ceph OSD Daemon —— 将数据以对象的形式存储到存储节点。
* replication： 计算机领域给出的定义是分享数据，确保数据在硬件或者软件组件上保持一致性，从而提高了数据的可依赖性，容错性和可读性。IBM 给出的[解释](https://www-01.ibm.com/software/data/replication/)
* [recovery](https://en.wikipedia.org/wiki/Data_recovery)：
* backfilling：(http://www.aussiestockforums.com/forums/showthread.php?t=19759)
* [rebalancing](http://drbd.linbit.com/users-guide-9.0/s-rebalance-workflow.html)
* active + clean：
* [heartbeat](http://linux-ha.org/wiki/Heartbeat)：
* [data consistency](http://recoveryspecialties.com/dc01.html)：指的是数据的可用性，在单一的环境中它被认为是理所当然存在的。一般在单一环境中进行数据备份（生产数据的副本用在原始数据领域中的时候）的时候会发生 data consistency。
* []

### 安装

#### Step 1: 准备

操作系统： CentOS 7

| **node 名称** | **IP 地址** |
| ------------- | ----------- |
| ceph-deploy-admin-node | 192.168.9.59 |
| ceph-node1 | 192.168.9.58 |
| ceph-node2 | 192.168.9.57 |
| ceph-node3 | 192.168.9.55 |

##### Ceph 部署安装

1. 在 `ceph-deploy` 中添加 Ceph 仓库：
   ~~~ bash 
   # cat >/etc/yum.repos.d/ceph.repo <<EOF
   > [ceph-noarch]
   > name=Ceph noarch packages
   > baseurl=http://download.ceph.com/rpm-firefly/el7/noarch
   > enabled=1
   > gpgcheck=1
   > type=rpm-md
   > gpgkey=https://download.ceph.com/keys/release.asc
   > EOF
   ~~~   
   
   > **提示：**<br/>
   > * `firefly` 是 Ceph 的版本名称，你可以修改为最近主要的版本名称，Ceph 的版本信息：[http://ceph.com/category/releases/](http://ceph.com/category/releases/)。
   > * `el7` 指的是我使用的 Linux 发行版。对应关系如下：
     <table>
        <tr>
           <td>el6</td>
           <td>CentOS 6</td>
        </tr>
        <tr>
           <td>el7</td>
           <td>CentOS 7</td>
        </tr>
        <tr>
           <td>rhel6</td>
           <td>Red Hat 6.5</td>
        </tr>
        <tr>
           <td>rhel7</td>
           <td>Red Hat 7</td>
        </tr>
        <tr>
           <td>fc19</td>
           <td>Fedora 19</td>
        </tr>
        <tr>
           <td>fc20</td>
           <td>Fedora 20</td>
        </tr>
     </table>

2. 更新系统，并安装 ceph-deploy 
   ~~~ bash
   $ sudo yum -y update
   $ sudo yum -y install ceph-deploy
   ~~~ 
   
##### Ceph Node 安装

1. 安装 NTP（Network Time Protocol）
   
   建议在 Ceph node（尤其是 Ceph 监控节点）上安装 NTP，这样可以避免出现由于时间差异产生的各种问题。
   ~~~ bash
   sudo yum -y install ntp ntpdate ntp-doc
   ~~~

2. 安装 SSH 服务
   ~~~ bash
   yum -y install openssh-server
   systemctl start sshd
   ~~~

3. 为 `ceph-deploy` 端创建用户
   `ceph-deploy` 工具必须要以一个无密码 **sudo** 权限的用户登录到其它的 `ceph-node` 端，这是因为 `ceph-deploy` 这个工具需要在无密码提示的情况下安装软件并配置相关文件。
    
   最新的 `ceph-deply` 的版本支持 `--username` 选项，所以你完全可以通过指定某个具有无密码 **sudo** 权限的用户（包括 root 用户，即便我们并不支持这么做）对其它 `ceph-node` 进行操作。建议在所有 `ceph-node` 上都创建该用户，这样一来就方便多了，但是最好不要用 `ceph` 作为你指定的用户名称。

   i. 在每个 ceph 节点上都创建一个新用户：
      我新创建的用户名是 `Ceph`
      ~~~ bash
      $ sudo useradd -d /home/Ceph -m Ceph
      $ sudo passwd Ceph
      ~~~
   
   ii. 为所有 ceph 节点上的新用户添加 `sudo` 权限： 
       ~~~ bash
       $ echo "Ceph ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/Ceph
       $ sudo chmod 0440 /etc/sudoers.d/Ceph
       ~~~
  
4. 开启无密码 SSH
   
   1. 使用你刚刚创建的新用户生成 SSH 密钥，口令设为空：
      ~~~ bash
      [Ceph@ceph-deploy-admin-node helen]$ ssh-keygen
      Generating public/private rsa key pair.
      Enter file in which to save the key (/home/Ceph/.ssh/id_rsa): 
      Created directory '/home/Ceph/.ssh'.
      Enter passphrase (empty for no passphrase): 
      Enter same passphrase again: 
      Your identification has been saved in /home/Ceph/.ssh/id_rsa.
      Your public key has been saved in /home/Ceph/.ssh/id_rsa.pub.
      The key fingerprint is:
      41:a6:62:50:7f:79:95:c0:88:55:4e:b8:0f:0d:3f:75 Ceph@ceph-deploy-admin-node
      The key's randomart image is:
      +--[ RSA 2048]----+
      |  ...  o+=+...   |
      |   . ..+++..o E  |
      |    o o +=o. .   |
      |   . . .oo+      |
      |        So .     |
      |          .      |
      |                 |
      |                 |
      |                 |
      +-----------------+
      ~~~
   
   2. 将密钥拷贝到其它三个 Ceph 节点中：
      ~~~ bash
      [Ceph@ceph-deploy-admin-node ~]$ ssh-copy-id Ceph@ceph-node1
      The authenticity of host 'ceph-node1 (192.168.9.58)' can't be established.
      ECDSA key fingerprint is 49:b1:e8:e6:5a:fb:a9:ac:a1:d0:ae:8a:2b:da:12:f7.
      Are you sure you want to continue connecting (yes/no)? yes
      /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
      /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
      Ceph@ceph-node1's password: 

      Number of key(s) added: 1

      Now try logging into the machine, with:   "ssh 'Ceph@ceph-node1'"
      and check to make sure that only the key(s) you wanted were added.
      ~~~
       
      ~~~ bash
      # ceph-node2 和 ceph-node3 同理
      ssh-copy-id Ceph@ceph-node2
      ssh-copy-id Ceph@ceph-node3
      ~~~

#### Step 2: 存储集群

##### 创建一个集群
##### 操作集群
##### 扩展集群
##### 存储/获取对象数据
#### Step 3: Ceph 客户端

