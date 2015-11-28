# 在 oVirt 中整合 Cinder

**目录：**
   
- [安装](#installation)
   - [准备](#preflight)
      - [配置 Ceph Deploy](#CephDeploySetup)
      - [配置 Ceph 节点](#CephNodeSetup)
   - [存储集群之快速入门](#StorageClusterQuickStart)
      - [创建集群](#CreateCluster)
      - [操作集群](#OperatingCluster)
      - [扩展集群](#ExpandingCluster)
      - [存储/获取对象数据](#StoringRetrievingObjectData)
   - [块设备之快速入门](#BlockDeviceQuickStart)
      - [安装 Ceph](#CephInstallation)
      - [配置块设备](#ConfigureABlockDevice)

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

<h2 id="installation">安装</h2>

<h3 id="preflight">准备</h3>

操作系统： CentOS 7

| **node 名称** | **IP 地址** | **作用** |
| ------------- | ----------- | -------- |
| ceph-deploy-admin-node | 192.168.9.59 | 管理其它 Ceph 节点的 node |
| ceph-node1 | 192.168.9.58 | 管理 Ceph Storage cluster |
| ceph-node2 | 192.168.9.57 | 管理 Ceph Storage cluster |
| ceph-node3 | 192.168.9.55 | 管理 Ceph Storage cluster |

<h4 id="startInstall">配置 Ceph Deploy</h4>

1. 在 `ceph-deploy` 中添加 Ceph 仓库：
   ~~~ bash 
   # cat >/etc/yum.repos.d/ceph.repo <<EOF
   > [ceph-noarch]
   > name=Ceph noarch packages
   > baseurl=http://download.ceph.com/rpm-hammer/el7/noarch
   > enabled=1
   > gpgcheck=1
   > type=rpm-md
   > gpgkey=https://download.ceph.com/keys/release.asc
   > EOF
   ~~~   
   
   > **提示：**<br/>
   > * `hammer` 是 Ceph 的版本名称，你可以修改为最近主要的版本名称，需要注意的是版本名称应该写成小些形式。Ceph 的版本信息：[http://ceph.com/category/releases/](http://ceph.com/category/releases/)。
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
   
<h4 id="CephNodeSetup">配置 Ceph 节点</h4>

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

   1. 在每个 ceph 节点上都创建一个新用户：
      我新创建的用户名是 `Ceph`
      ~~~ bash
      $ sudo useradd -d /home/Ceph -m Ceph
      $ sudo passwd Ceph
      ~~~
   
   2. 为所有 ceph 节点上的新用户添加 `sudo` 权限： 

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

      > **注意：**：由于后面会用到 ceph 客户端，所以最好在 ceph-node1, ceph-node2, ceph-node3 之间也实现无密码登录，重复上面的步骤既可。

5. 开机时启动网络
  
   Ceph OSD 是相互对等的，且它们都通过网络向 Ceph Monitors 进行报告。如果默认情况下网络是关闭的状态，那么 Ceph Cluster 是不能够进行网络通信的。

   CentOS 7 默认的网络接口中配置的网络默认情况下是关闭的。您可以在 `/etc/sysconfig/network-scripts` 目录下的 `/etc/ifcfg-eth0` （eth0 是我的网卡名称）文件中将 `ONBOOT` 设置为 `yes`。

6. 确保连接性
 
   使用 ping 某台机器 IP 或主机名的方式来检测机器是否已经连网。如果用的是主机名，不要忘记配置本地 DNS。

7. 打开所需的端口
   
   <table>
        <tr>
           <td>服务</td>
           <td>默认端口</td>
        </tr>
        <tr>
           <td>Ceph Monitor</td>
           <td>6789</td>
        </tr>
        <tr>
           <td>Ceph OSD</td>
           <td>6800:7300</td>
        </tr>
   </table>

   Ceph OSD 可以使用多个网络连接来和客户端，Ceph Monitor，用于复制的 OSD，以及用于监控的心跳 OSD。
   
   1. 首先确保 `firewalld` 已经打开：
      ~~~ bash
      systemctl start firewalld
      systemctl enable firewalld
      ~~~       

   2. CentOS 7 默认的 `firewall` 配置相对严格。如若想要在 Ceph 客户端上与 Ceph node 进行通信，你必须调整 firewall 的设置。
      ~~~ bash
      $ sudo firewall-cmd --zone=public --add-port=6789/tcp --permanent
      success
      $ sudo firewall-cmd --zone=public --add-port=6800-7300/tcp --permanent 
      success
      ~~~
   
8. 关闭 `SELINUX`：
   
   为了简化安装，我们建议您设置 `SELINUX` 的权限或者直接关掉它。在关闭 `SELINUX` 之前，请确保 node 是可以正常使用的。
   ~~~ bash
   sudo setenforce 0
   ~~~

9. 设置调用源时的优先级： 

   1. 确保你的包管理器有安装好的优先级包。CentOS 中还需要安装 EPEL。
      ~~~ bash
      sudo yum -y install yum-plugin-priorities
      sudo yum -y install epel
      ~~~

<h3 id='StorageClusterQuickStart'>存储集群之快速入门</h3>

这里我们利用 ceph-deploy-admin-node 上的 `ceph-deploy` 来设置 Ceph Storage Cluster。创建一个具有三个节点的集群之后你就可以继续挖掘 Ceph 的功能了。创建一个 Ceph Monitor * 1 + Ceph OSD Daemon * 2 的集群。一旦 cluster 达到了 `active + clean` 的状态后，就可以通过添加第三个 OSD，添加 MDS 或者添加至少两个的 Ceph Monitor 来扩展它了^.^。

> **提示**：最好在管理节点上创建一个新的目录来保存 `ceph-deploy` 为集群生成的配置文件和 keys。
> ~~~ bash
  $ mkdir my-cluster
  $ cd my-cluster/
  ~~~
>  
> **注意**：`ceph-deploy` 工具会将输出目录存放至 `my-cluster` 目录中，所以你需要在该目录下执行 `ceph-deploy` 命令。

<h4 id='CreateCluster'>创建集群</h4>

1. 如果你的操作对集群造成了不可挽回的损毁，你想要重新来一遍怎么办呢？执行下面的命令来清理你的配置信息：
   ~~~ bash
   # {ceph-node} 用于替换非管理节点的主机名
   $ ceph-deploy purgedata {ceph-node} [{ceph-node}]
   $ ceph-deploy forgetkeys
   ~~~
   
   想要清理 Ceph 包，请执行如下操作：
   ~~~ bash
   $ ceph-deploy purge {ceph-node} [{ceph-node}]
   ~~~
 
   > **注意**：执行了 `purge` 之后，你必须要再安装一遍 `Ceph`。

   在管理节点的新创建的 `my-cluster` 目录下利用 `ceph-deploy` 工具执行如下操作：
  
2. 开始部署新的集群，并为其生成集群配置文件和 keyring
   ~~~ bash
   # 初始化作为监控节点的 ceph-node1
   [Ceph@ceph-deploy-admin-node my-cluster]$ ceph-deploy new ceph-node1
   ~~~
   
   这时你可以看到 `my-cluster` 中出现了三个新的文件
   ~~~ bash
   [Ceph@ceph-deploy-admin-node my-cluster]$ ll
   总用量 16
   -rw-rw-r--. 1 Ceph Ceph  232 11月 24 11:30 ceph.conf # Ceph 的配置文件
   -rw-rw-r--. 1 Ceph Ceph 6741 11月 24 11:30 ceph.log # 新集群的日志文件
   -rw-------. 1 Ceph Ceph   73 11月 24 11:30 ceph.mon.keyring # 监控节点的秘密密钥
   ~~~

3. Ceph 配置文件中默认的 OSD 的数量是 3，我们需要把它改成 2，才能达到在只有两个 Ceph OSD 的情况下保持 `active + clean` 的状态。将下面一行添加到 `[global]` 部分中。
   ~~~ bash
   osd pool default size = 2
   ~~~

4. 安装 Ceph
   ~~~ bash
   $ ceph-deploy install ceph-deploy-admin-node ceph-node1 ceph-node2 ceph-node3
   ~~~
   `ceph-deploy` 工具将会在每个 Ceph 节点上安装 Ceph。
   > **注意**：如果你执行了 `ceph-deploy purge` 这个命令，你必须重新执行该指令以重装 `Ceph`。

5. 添加初始化的监控节点，并生成密钥
   ~~~ bash
   [Ceph@ceph-deploy-admin-node my-cluster]$ ceph-deploy mon create-initial
   ~~~
   
   以上命令完成后，你会发现 `my-cluster` 目录下多了几个 key 文件出来：
   ~~~ bash
   总用量 220
   -rw-------. 1 Ceph Ceph     71 11月 24 15:29 ceph.bootstrap-mds.keyring
   -rw-------. 1 Ceph Ceph     71 11月 24 15:29 ceph.bootstrap-osd.keyring
   -rw-------. 1 Ceph Ceph     71 11月 24 15:29 ceph.bootstrap-rgw.keyring # 只有集群是 Hammer 或以上版本的时候才会有 bootstrap-rgw keyring
   -rw-------. 1 Ceph Ceph     63 11月 24 15:29 ceph.client.admin.keyring
   -rw-rw-r--. 1 Ceph Ceph    257 11月 24 11:57 ceph.conf
   -rw-rw-r--. 1 Ceph Ceph 154469 11月 24 15:34 ceph.log
   -rw-------. 1 Ceph Ceph     73 11月 24 11:30 ceph.mon.keyring
   ~~~

6. 添加两个 OSD
    
   1. 如果想要快速配置，就不必使用 Ceph OSD 的整个磁盘了，只用一个目录即可。登录到 Ceph 的 OSD 节点并为 OSD 后台程序创建一个目录。
      ~~~ bash
      [Ceph@ceph-node2 ~]$ sudo mkdir /var/local/osd0
      [Ceph@ceph-node3 ~]$ sudo mkdir /var/local/osd1
      ~~~

   2. 使用 Ceph 管理节点上的 `ceph-deploy` 添加两个 OSD
      ~~~ bash
      [Ceph@ceph-deploy-admin-node my-cluster]$ ceph-deploy osd prepare ceph-node2:/var/local/osd0 ceph-node3:/var/local/osd1
      ~~~
 
   3. 激活 OSD
      ~~~ bash
      [Ceph@ceph-deploy-admin-node my-cluster]$ ceph-deploy osd activate ceph-node2:/var/local/osd0 ceph-node3:/var/local/osd1
      ~~~
  
   4. 利用 `ceph-deploy` 工具将配置文件和 admin 密钥拷贝到你的 Ceph 管理节点（ceph-deploy-admin-node）& Ceph 非管理节点（ceph-node1, ceph-node2, ceph-node3）上，这样一来，当你使用 ceph 客户端执行某条指令的时候就无需指定监控节点地址和 `ceph.client.admin.keyring` 了。

      ~~~ bash
      [Ceph@ceph-deploy-admin-node my-cluster]$ ceph-deploy admin ceph-deploy-admin-node ceph-node1 ceph-node2 ceph-node3
      ~~~

      > **注意**：不要忘记在 Ceph 管理节点中的 `/etc/hosts/` 目录中配置本机域名。
 
   5. 为每个节点中的 `ceph.client.admin.keyring` 文件添加可读权限
      ~~~ bash
      $ sudo chmod +r /etc/ceph/ceph.client.admin.keyring
      ~~~
   
   6. 测试集群是否创建成功
      ~~~ bash
      $ ceph health
      ~~~

      **Your cluster should return an active + clean state when it has finished peering.**

<h4 id='OperatingCluster'>操作集群</h4>
   
   1. 操作集群的后台程序（OSD）：
      1. 开启所有的后台程序
         通过执行 ceph start 命令开启 Ceph 集群：
         ~~~ bash
         sudo /etc/init.d/ceph -a start
         [Ceph@ceph-deploy-admin-node my-cluster]$ sudo /etc/init.d/ceph -a start
         === osd.0 === 
         root@ceph-node2's password: 
         root@ceph-node2's password: 
         Starting Ceph osd.0 on ceph-node2...already running
         === osd.1 === 
         root@ceph-node3's password: 
         root@ceph-node3's password: 
         df: "/var/lib/ceph/osd/ceph-1/.": 没有那个文件或目录
         root@ceph-node3's password: 
         create-or-move updated item name 'osd.1' weight 1 at location {host=ceph-deploy-admin-node,root=default} to crush map
         Starting Ceph osd.1 on ceph-node3...
         root@ceph-node3's password: 
         Running as unit run-19894.service.
         ~~~
         
         `-a` 意味着对所有节点执行这条指令。
      
         > **提醒**：如果你不想每次都输入密码，请使用 SSH 无密码登录的方式：
                     ~~~ bash
                     sudo ssh-keygen
                     sudo ssh-copy-id -i /root/.ssh/id_rsa.pub root@ceph-node1
                     sudo ssh-copy-id -i /root/.ssh/id_rsa.pub root@ceph-node2
                     sudo ssh-copy-id -i /root/.ssh/id_rsa.pub root@ceph-node1
                     ~~~

      2. 开启不同类型的后台程序：
         * 想在本地 Ceph 节点开启同一类型的后台程序：
           例如 `ceph-node3` 上只运行了 osd.1：
           ~~~ bash
           === osd.1 === 
           create-or-move updated item name 'osd.1' weight 0.03 at location {host=ceph-node3,root=default} to crush map
           Starting Ceph osd.1 on ceph-node3...
           Running as unit run-22695.service.
           === osd.1 === 
           Starting Ceph osd.1 on ceph-node3...already running
           ~~~
 
         * 想在异地 Ceph 节点开启同一类型的后台程序：
           例如 `ceph-node2` 运行了 osd.0，`ceph-node3` 上运行了 osd.1：
           ~~~ bash
           === osd.1 === 
           create-or-move updated item name 'osd.1' weight 0.03 at location {host=ceph-node3,root=default} to crush map
           Starting Ceph osd.1 on ceph-node3...
           Running as unit run-23366.service.
           === osd.0 === 
           Starting Ceph osd.0 on ceph-node2...already running
           === osd.1 === 
           Starting Ceph osd.1 on ceph-node3...already running
           ~~~

      3. 关闭不同类型的后台程序：
         * 想在本地 Ceph 节点关闭同一类型的后台程序： 
           例如 `ceph-node1` 上运行的是 mon：
           ~~~ bash
           === mon.ceph-node1 === 
           Stopping Ceph mon.ceph-node1 on ceph-node1...kill 13239...done
           ~~~

         * 想在异地 Ceph 节点关闭同一类型的后台程序：
           例如在 `ceph-node3` 上关闭 `ceph-node1` 上的 mon 程序：
           ~~~ bash
           [Ceph@ceph-node3 ~]$ sudo /etc/init.d/ceph -a stop mon
           === mon.ceph-node1 === 
           Stopping Ceph mon.ceph-node1 on ceph-node1...kill 17739...done
           ~~~

      4. 开启具体的后台程序：
         * 开启本地节点上具体的某个 Ceph 后台程序
           例如在 `ceph-node1` 上开启 mon：
           ~~~ bash
           [Ceph@ceph-node1 ~]$ sudo /etc/init.d/ceph start mon.ceph-node1
           === mon.ceph-node1 === 
           Starting Ceph mon.ceph-node1 on ceph-node1...
           Running as unit run-18285.service.
           Starting ceph-create-keys on ceph-node1...
           ~~~

        * 在异地节点开启具体的某个 Ceph 后台程序
          例如在 `ceph-node1` 上开启 `ceph-node2` 上的 osd.0 程序：
          ~~~ bash
          [Ceph@ceph-node1 ~]$ sudo /etc/init.d/ceph -a start osd.0
          === osd.0 === 
          Starting Ceph osd.0 on ceph-node2...already running
          ~~~
          
      5. 关闭具体的后台程序：
         * 关闭本地节点上具体的某个 Ceph 后台程序
           例如在 `ceph-node1` 上关闭 mon：     
           ~~~ bash
           [Ceph@ceph-node1 ~]$ sudo /etc/init.d/ceph stop mon.ceph-node1
           === mon.ceph-node1 === 
           Stopping Ceph mon.ceph-node1 on ceph-node1...kill 18289...done
           ~~~
   
         * 在异地节点关闭具体的某个 Ceph 后台程序
           例如在 `ceph-node1` 上关闭 `ceph-node2` 上的 osd.0 程序：
           ~~~ bash
           [Ceph@ceph-node1 ~]$ sudo /etc/init.d/ceph -a stop osd.0
           === osd.0 === 
           Stopping Ceph osd.0 on ceph-node2...kill 16701...kill 16701...done
           ~~~
      6. ceph 作为一个服务来运行：
 
   2. 监控 Ceph 集群：
      1. 交互模式：
      2. 检查集群的健康状态：
      3. 观察集群：
      4. 观察集群的使用情况：
      5. 检查集群的状态：
      6. 检查集群的状态：
      7. 查看监控器的状态：
      8. 查看 MDS 的状态：
      9. 检查 placement group（放置组？）的状态：
      10. 是用 admin 套接字：

   3. 
<h4 id='ExpandingCluster'>扩展集群</h4>
<h4 id='StoringRetrievingObjectData'>存储/获取对象数据</h4>
<h3 id='BlockDeviceQuickStart'>块设备之快速入门</h3>
<h4 id='CephInstallation'>安装 Ceph</h4>
<h4 id='ConfigureABlockDevice'>配置块设备</h4>

