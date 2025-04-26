---
title: ceph基本概念，架构，部署（一）
date: 2025-04-18 11:14:44
tags: Ceph
categories: 存储篇
---
##  一、分布式存储概述

### 1.存储分类

存储分为**封闭系统的存储**和**开放系统的存储**，而对于开放系统的存储又被分为**内置存储**和**外挂存储**。

外挂存储又被细分为直连式存储(DAS)和网络存储(FAS)，而网络存储又被细分网络接入存储(NAS)和存储区域网络(SAN)等。

- DAS(Direct-attached Storage): 直连存储,即直接连接到主板的总线上去的，我们可以对这些设备进行格式化操作。 典型代表有：IDE，SATA，SCSI，SAS，USB等。

- SAN(Storage Area Network): 存储区域网络，是一个网络上的磁盘。它提供的一个块设备而非文件系统。 早期是通过SCSI协议传输数据，后来设计通过光纤通道交换机连接存储阵列和服务器主机，也称为FC SAN，当然也可以基于以太网传输，我们称之为ISCSI协议。

- NAS(Network Attached Storage): 网络附加存储，是一个网络上的文件系统，我们无法进行格式化操作。典型代表有：NFS，CIFS等。

专门的存储厂商可以通过RAID技术来实现数据的高效存储，国内外很多企业都有自己的存储设备，例如EMC,NetApp,IBM,惠普，Dell，爱数等。

但是这些专业的存储设备不仅价格是非常昂贵的，而且是非常重的，大多数都是基于FC SAN，ISCSI或者NAS访问接口，所以在某种意义上将他们的存储能力和扩展能力是非常有限的，这个时候我们就需要一个能够实现横向存储的分布式存储。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/7223093e4dcf42e2bccc4fb68f1b20bc.png)

### 2.存储系统的分类

- 块存储系统: 通常对应的是一个裸设备，比如一块磁盘，我们需要格式化后进行挂载方能使用。 代表产品: lvm，cinder。
- 文件系统存储系统: 文件系统只是数据组织存放的接口。文件系统通常是构建在一个块存储级别之上。 文件系统被分为元数据区域和数据区域，对于用户而言，它呈现为一个树形结构(实际上提供的是一个目录)。 代表产品: NFS，glusterfs。

- 对象存储系统: 对象存储并没有向文件系统那样划分为元数据区域和数据区域，而是按照不同的对象进行存储，而且每个对象内部维护着元数据和数据区域。因此每个对象都有自己独立的管理格式。 代表产品: Fastdfs，swift。

**补充知识：**

为什么网速很慢，上传10T这种大文件确很快？

因为每一个文件都有独有的hash值，就是我们上传文件时，系统去扫描文件的[哈希码](https://zhida.zhihu.com/search?content_id=451076183&content_type=Answer&match_order=1&q=哈希码&zhida_source=entity)并保存在数据库中，每个文件（包括其副本）的哈希码都是独一无二的，就是说，如果哈希码相同，我们可以定义为是同一个文件。接下来就简单了，上传时判断哈希码，如果数据库里有相同的哈希码就说明这个文件在云端有一摸一样的副本了，直接显示上传完成就可以了，文件并没有被真实上传，你的等待时间是计算哈希码的时间。如果在云端没有相同哈希码的文件，系统再老老实实上传。

而修改文件名并不会改变hash值，不要不更改文件内容，哈希码就不会改变。所以可以实现秒传的技术。

文件哈希码的特性使哈希码用途很广泛，因为用哈希码可以标定所有相同的文件。除了秒传、去重等功能，还可以用在判断文件是否更新、统一阻止违规文件传播、统一删除文件上上传文件等

如果多个用户拥有相同资源，实际上远程存储服务器也只是存储一份，如果我们本地删除改文件，其实也是删除软连接，远端存储服务器的数据并不会删除。所有也并不会影响其他用户访问。所有大家在网盘上传隐私性较强的东西一定要谨慎~



## 二、Ceph分布式存储系统概述

Ceph 是一个开源的分布式存储系统，旨在提供高性能、高可扩展性和高可靠性的存储解决方案。它能够通过统一的存储平台同时支持块存储、对象存储和文件存储。Ceph 是一个去中心化的系统，能够在大规模分布式环境中工作，广泛应用于云计算和大数据处理领域。

**特点**

- 高可扩展性：Ceph 能够动态扩展，随着更多的存储节点的加入，集群的容量和性能也能够得到相应的扩展。
- 高可靠性：Ceph 使用 CRUSH 算法来实现数据的分布式存储和副本管理，从而确保数据的高可靠性。当某个节点发生故障时，Ceph 可以自动恢复丢失的数据。
- 自我修复：Ceph 会自动检测节点故障并进行数据恢复，确保数据的高可用性。
- 灵活性：Ceph 可以同时提供块存储、对象存储和文件存储服务，适应多种存储需求。
- 性能优化：Ceph 使用对象存储和 CRUSH 算法来优化存储性能，支持高吞吐量和低延迟的存储需求。



### 1.Ceph 组件

**Ceph OSD (Object Storage Daemon)**

- **作用**：负责存储数据和处理数据的复制、恢复、重平衡等操作。每个 OSD 进程通常对应一个物理硬盘或磁盘阵列。
- **功能**：数据存储、数据复制、数据重建和故障转移等。OSD 是 Ceph 集群的工作核心。

**Ceph MON (Monitor)**

- **作用**：监控 Ceph 集群的健康状况，维护集群的元数据，并确保集群的一致性。MON 节点管理集群状态信息、存储池、OSD 设备等元数据。
- **功能**：提供集群的状态信息，处理客户端和 OSD 之间的交互，确保集群的一致性和健康状态。

**Ceph MDS (Metadata Server)**

- **作用**：仅在 Ceph 用作文件系统（CephFS）时使用，负责处理文件系统的元数据管理，如文件目录结构、文件权限、文件名等。
- **功能**：在 CephFS 中管理目录、文件元数据和锁，并提供文件访问接口。

**Ceph Client**

- **作用**：客户端是与 Ceph 集群交互的应用程序或系统，进行数据存储、访问、管理等操作。
- **功能**：客户端可以是使用 Ceph 提供的不同接口进行操作的应用，例如 Ceph RBD（块存储）、CephFS（文件存储）等。

**Ceph RGW (Rados Gateway)**

- **作用**：Rados Gateway 提供与 Ceph 集群的对象存储服务的接口，支持通过 HTTP 协议访问存储的数据，兼容 S3 和 OpenStack Swift API。
- **功能**：允许对象存储服务的访问和管理，提供 RESTful 接口，适用于云存储服务和大数据应用等。

**CRUSH (Controlled Replication Under Scalable Hashing)**

- **作用**：CRUSH 是 Ceph 的数据分布和复制算法，用于决定数据如何在 OSD 之间分配和存储，避免中央元数据服务的瓶颈。
- **功能**：实现数据的分布式存储和复制策略，确保数据冗余和高可用性。

**Ceph Pools**

- **作用**：池是 Ceph 中用于存储数据的逻辑分区，每个池有不同的设置和配置，如副本数量、对象大小等。
- **功能**：用于数据存储和隔离，支持不同的存储策略（例如，块存储、对象存储等）。

**Ceph Dashboard (可选)**

- **作用**：提供 Ceph 集群的 Web 界面，供管理员查看集群的状态、健康状况、性能指标、配置等。
- **功能**：集群监控和管理界面，支持查看集群健康、性能、日志等信息。

**Ceph RBD (Rados Block Device)**

- **作用**：Ceph 提供的块存储服务，允许客户端将 Ceph 集群作为块设备挂载到操作系统中，适用于虚拟化存储或数据库等应用场景。
- **功能**：提供与传统块设备（如 iSCSI）类似的功能，但具有 Ceph 集群的可扩展性和容错性。

**CephFS (Ceph File System)**

- **作用**：Ceph 的文件系统接口，允许将 Ceph 集群作为一个分布式文件系统进行访问。
- **功能**：提供 POSIX 兼容的文件系统，支持多客户端并发访问，适用于高性能计算、大数据处理等应用场景。

**总结：**

- **OSD**：处理数据存储和复制。
- **MON**：监控集群状态和元数据管理。
- **MDS**：管理 CephFS 文件系统的元数据。
- **RGW**：提供对象存储接口。
- **CRUSH**：数据分布和复制算法。
- **Pools**：逻辑存储池，用于数据组织。
- **Client**：与集群交互的客户端应用程序。
- **RBD**：块存储服务。
- **CephFS**：分布式文件系统。
- **Dashboard**：集群管理和监控界面。



### 2.Ceph逻辑单元

- pool（池）：pool是Ceph存储数据时的逻辑分区，它起到namespace的作用，在集群层面的逻辑切割。每个pool包含一定数量(可配置)的PG。
- PG（Placement Group）：PG是一个逻辑概念，每个对象都会固定映射进一个PG中，所以当我们要寻找一个对象时，只需要先找到对象所属的PG，然后遍历这个PG就可以了，无需遍历所有对象。而且在数据迁移时，也是以PG作为基本单位进行迁移。PG的副本数量也可以看作数据在整个集群的副本数量。一个PG 包含多个 OSD 。引入 PG 这一层其实是为了更好的分配数据和定位数据。
- OID：存储的数据都会被切分成对象（Objects）。每个对象都会有一个唯一的OID，由ino与ono生成，ino即是文件的File ID，用于在全局唯一标示每一个文件，而ono则是分片的编号，OID = ( ino + ono )= (File ID + File part number)，例如File Id = A，有两个分片，那么会产生两个OID，A01与A02。
- PgID：首先使用静态hash函数对OID做hash取出特征码，用特征码与PG的数量去模，得到的序号则是PGID。
- Object：Ceph最底层的存储单元是 Object对象，每个 Object 包含元数据和原始数据



### 3.pool、PG、OSD 关系

![img](https://gitee.com/ljh00928/csdn/raw/master/img/78357d84266d4f84beda5f35114fee04.png)

- 一个Pool里有很多PG，
- 一个PG里包含一堆对象；一个对象只能属于一个PG；
- PG有主从之分，一个PG分布在不同的OSD上（针对三副本类型）

## 三、Ceph 设计

### 1.整体设计

![img](https://gitee.com/ljh00928/csdn/raw/master/img/8ebc5de0fb864a0faaf77fc9ce1ed241.png)

**基础存储系统 RADOS**

Reliable, Autonomic,Distributed Object Store，即可靠的、自动化的、分布式的对象存储

这就是一个完整的对象存储系统，所有存储在 Ceph 系统中的用户数据事实上最终都是由这一层来存储的。而 Ceph 的高可靠、高可扩展、高性能、高自动化等等特性本质上也是由这一层所提供的

**基础库 librados**

这层的功能是对 RADOS 进行抽象和封装，并向上层提供 API，以便直接基于 RADOS（而不是整个 Ceph）进行应用开发。特别要注意的是，RADOS 是一个对象存储系统，因此，librados 实现的 API 也只是针对对象存储功能的。RADOS 采用 C++ 开发，所提供的原生 librados API 包括 C 和 C++ 两种。

**高层应用接口**

这层包括了三个部分：RADOS GW（RADOS Gateway）、 RBD（Reliable Block Device）和 Ceph FS（Ceph File System），其作用是在 librados 库的基础上提供抽象层次更高、更便于应用或客户端使用的上层接口。其中，RADOS GW 是一个提供与 Amazon S3 和 Swift 兼容的 RESTful API 的 gateway，以供相应的对象存储应用开发使用。RADOS GW 提供的 API 抽象层次更高，但功能则不如 librados 强大。

**应用层**

这层是不同场景下对于 Ceph 各个应用接口的各种应用方式，例如基于 librados 直接开发的对象存储应用，基于 RADOS GW 开发的对象存储应用，基于 RBD 实现的云硬盘等等。librados 和 RADOS GW 的区别在于，librados 提供的是本地 API，而 RADOS GW 提供的则是 RESTfulAPI。

由于 Swift 和 S3 支持的 API 功能近似，这里以 Swift 举例说明。Swift 提供的 API 功能主要包括：

- 用户管理操作：用户认证、获取账户信息、列出容器列表等；
- 容器管理操作：创建 / 删除容器、读取容器信息、列出容器内对象列表等；
- 对象管理操作：对象的写入、读取、复制、更新、删除、访问许可设置、元数据读取或更新等

### 2.逻辑架构

![img](https://gitee.com/ljh00928/csdn/raw/master/img/d8bc4b3fa3d64fe4bae0d2f10d8fd793.png)

### 3.Ceph物理组件架构

RADOS是Ceph的核心，我们谈及的物理组件架构也只是RADOS的物理架构。

RADOS集群是由若干服务器组成，每一个服务器上都相应会运行RADOS的核心守护进程（OSD、MON、MDS）。具体守护进程的数量需要根据集群的规模和既定的规则来配置。

 ![img](https://gitee.com/ljh00928/csdn/raw/master/img/a9a132418ceb480bb08be115eb61e6b3.png)

我们首先来看每一个集群节点上面的守护进程的主要作用：

**OSD Daemon**：两方面主要作用，一方面负责数据的处理操作，另外一方面负责监控本身以及其他OSD进程的健康状态并汇报给MON。OSD守护进程在每一个PG（Placement Group）当中，会有主次（Primary、Replication）之分，Primary主要负责数据的读写交互，Replication主要负责数据副本的复制。其故障处理机制主要靠集群的Crush算法来维持Primary和Replication之间的转化和工作接替。所有提供磁盘的节点上都要安装OSD 守护进程。

**MON Daemon**：三方面主要作用，首先是监控集群的全局状态（OSD Daemon Map、MON Map、PG Map、Crush Map），这里面包括了OSD和MON组成的集群配置信息，也包括了数据的映射关系。其次是管理集群内部状态，当OSD守护进程故障之后的系列恢复工作，包括数据的复制恢复。最后是与客户端的查询及授权工作，返回客户端查询的元数据信息以及授权信息。安装节点数目为2N+1，至少三个来保障集群算法的正常运行。

**MDS Daemon**：它是Ceph FS的元数据管理进程，主要是负责文件系统的元数据管理，它不需要运行在太多的服务器节点上。安装节点模式保持主备保护即可

### 4.Ceph数据对象组成

从客户端发出的一个文件请求，到Rados存储系统写入的过程当中会涉及到哪些逻辑对象，他们的关系又是如何的？首先，我们先来列出这些对象：

（1）文件（FILE）：用户需要存储或者访问的文件。对于一个基于Ceph开发的对象存储应用而言，这个文件也就对应于应用中的“对象”，也就是用户直接操作的“对象”。

（2）对象（Object）：RADOS所看到的“对象”。Object指的是最大size由RADOS限定（通常为2/4MB）之后RADOS直接进行管理的对象。因此，当上层应用向RADOS存入很大的file时，需要将file切分进行存储。

（3）PG（Placement Group）：PG是一个逻辑概念，阐述的是Object和OSD之间的地址映射关系，该集合里的所有对象都具有相同的映射策略；Object & PG，N：1的映射关系；PG & OSD，1：M的映射关系。一个Object只能映射到一个PG上，一个PG会被映射到多个OSD上。

（4）OSD（Object Storage Device）：存储对象的逻辑分区，它规定了数据冗余的类型和对应的副本分布策略；支持两种类型：副本和纠删码。

接下来，我们以更直观的方式来展现在Ceph当中数据是如何组织起来的：

 ![img](https://gitee.com/ljh00928/csdn/raw/master/img/038eb48ccda14b42850eecf189954af4.png)

（1） File > Object

本次映射为首次映射，即将用户要操作的File，映射为RADOS能够处理的Object。

具体映射操作本质上就是按照Object的最大Size对File进行切分，每一个切分后产生的Object将获得唯一的对象标识Oid。Oid的唯一性必须得到保证，否则后续映射就会出现问题。

（2） Object > PG

完成从File到Object的映射之后， 就需要将每个 Object 独立地映射到 唯一的 PG 当中 去。

Hash（Oid）& Mask > PGid

根据以上算法， 首先是使用Ceph系统指定的一个静态哈希函数计算 Oid 的哈希值，将 Oid 映射成为一个近似均匀分布的伪随机值。然后，将这个伪随机值和 Mask 按位相与，得到最终的PG序号（ PG id）。根据RADOS的设计，给定PG的总数为 X（X= 2的整数幂）， Mask=X-1 。因此，哈希值计算和按位与操作的整体结果事实上是从所有 X 个PG中近似均匀地随机选择一个。基于这一机制，当有大量object和大量PG时，RADOS能够保证object和PG之间的近似均匀映射。

（3） PG > OSD

最后的 映射就是将PG映射到数据存储单元OSD。RADOS采用一个名为CRUSH的算法，将 PGid 代入其中，然后得到一组共 N 个OSD。这 N 个OSD即共同负责存储和维护一个PG中的所有 Object 。和“object -> PG”映射中采用的哈希算法不同，这个CRUSH算法的结果不是绝对不变的，而是受到其他因素的影响。

① 集群状态（Cluster Map）：系统中的OSD状态 。数量发生变化时， CLuster Map 可能发生变化，而这种变化将会影响到PG与OSD之间的映射。

② 存储策略配置。系统管理员可以指定承载同一个PG的3个OSD分别位于数据中心的不同服务器乃至机架上，从而进一步改善存储的可靠性。

到这里，可能大家又会有一个问题“为什么这里要用CRUSH算法，而不是HASH算法？”

这一次映射，我们对映射算法有两种要求：

一方面，算法必须能够随着系统的节点数量位置的变化，而具备动态调整特性，保障在变化的环境当中仍然可以保持数据分布的均匀性；另外一方面还要有相对的稳定性，也就是说大部分的映射关系不会因为集群的动态变化发生变化，保持一定的稳定性。

而CRUSH算法正是符合了以上的两点要求，所以最终成为Ceph的核心算法。

## 四、ceph 数据存储过程

![img](https://gitee.com/ljh00928/csdn/raw/master/img/aae334b455d4499f9f9ae16bc55f9079.png)

如果我们想要将数据存储到ceph集群，那么大致步骤如下所示:

- Rados Cluster集群固定大小的object可能不符合我们要存储某个大文件，因此一个大文件想要存储到ceph集群，它可能会被拆分成多个data object对象进行存储;
- 通常情况下data object请求向某个pool存储数据时，通过CRUSH算法会先对data object进行一致性哈希计算，而后将存储任务映射到到该pool中的某个PG上;
- 紧接着，CRUSH算法(是用来完成object存储路由的一个算法)会根据pool的冗余副本数量和data object的存储类型找到足量的OSD进行存储，当然对应的PG是有active PG和standby PG角色之分的，通常副本数我们会设置为3;

## 五、ceph集群部署

### 1.集群环境准备

| 主机名  | ip         | 配置                |
| ------- | ---------- | ------------------- |
| ceph141 | 10.0.0.141 | 1c2G，300GB ，500GB |
| ceph142 | 10.0.0.142 | 1c2G，300GB ，500GB |
| ceph143 | 10.0.0.143 | 1c2G，300GB ，500GB |

 **ceph所有节点基础环境准备**

```
基于cephadm部署前提条件，官方提的要求Ubuntu 22.04 LTS出了容器运行时其他都满足
			- Python 3
			- Systemd
			- Podman or Docker for running containers
			- Time synchronization (such as Chrony or the legacy ntpd)
			- LVM2 for provisioning storage devices
			
参考链接:
	https://docs.ceph.com/en/latest/cephadm/install/#requirements
```

 **设置时区**

```
timedatectl set-timezone Asia/Shanghai
ll /etc/localtime
```

 **安装docker环境**

关于docker安装脚本可参考我这篇博文：[kubeadm 部署k8s-CSDN博客](https://blog.csdn.net/m0_69326428/article/details/144375315)

```
tar xf autoinstall-docker-docker-compose.tar.gz 
./install-docker.sh i
```

 **添加hosts文件解析**

```
cat >> /etc/hosts <<EOF
10.0.0.141 ceph141
10.0.0.142 ceph142
10.0.0.143 ceph143
EOF
```

### 2.cephadm部署初始化新集群

**下载需要安装ceph版本的cephadm**

```
[root@ceph141 ~]# CEPH_RELEASE=18.2.4
[root@ceph141 ~]# curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
```

 **将cephadm添加到PATH环境变量**

```
[root@ceph141 ~]# mv cephadm /usr/local/bin/
[root@ceph141 ~]# 
[root@ceph141 ~]# chmod +x /usr/local/bin/cephadm 
[root@ceph141 ~]# 
[root@ceph141 ~]# ls -l /usr/local/bin/cephadm
-rwxr-xr-x 1 root root 215316 Aug 20 22:19 /usr/local/bin/cephadm
```

 **创建新集群**

```
[root@ceph141 ~]# cephadm bootstrap --mon-ip 10.0.0.141 --cluster-network 10.0.0.0/24 --allow-fqdn-hostname
Creating directory /etc/ceph for ceph.conf
Verifying podman|docker is present...
Verifying lvm2 is present...
...
Generating a dashboard self-signed certificate...
Creating initial admin user...
Fetching dashboard port number...
Ceph Dashboard is now available at:

#访问的url，账号，密码
	     URL: https://ceph141:8443/
	    User: admin        
	Password: s3o1ou58iy

Enabling client.admin keyring and conf on hosts with "admin" label
```

 **查看docker镜像**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/20434c5525c6446bab47ddf615323b83.png)

**登录dashbod**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/4ae5a691f6e9475daad597f3f5f27df5.png)

**查看节点信息**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/3c11920a7a494e2fb3fee931d9ea4653.png)

参考地址：https://www.cuiliangblog.cn/

ceph的官方文档: https://docs.ceph.com/en/latest/ 
 官方架构图: https://docs.ceph.com/en/latest/architecture/
     