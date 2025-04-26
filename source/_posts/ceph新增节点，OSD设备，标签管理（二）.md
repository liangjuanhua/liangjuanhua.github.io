---
title: ceph新增节点，OSD设备，标签管理（二）
date: 2025-04-18 11:19:21
tags: Ceph
categories: 存储篇
---
##  一、访问客户端集群方式

方式一: 使用cephadm shell交互式配置

```
[root@ceph141 ~]# cephadm shell    # 注意，此命令会启动一个新的容器，运行玩后会退出！
Inferring fsid c153209c-d8a0-11ef-a0ed-bdb84668ed01
Inferring config /var/lib/ceph/c153209c-d8a0-11ef-a0ed-bdb84668ed01/mon.ceph141/config
Using ceph image with id '2bc0b0f4375d' and tag 'v18' created on 2024-07-24 06:19:35 +0800 CST
quay.io/ceph/ceph@sha256:6ac7f923aa1d23b43248ce0ddec7e1388855ee3d00813b52c3172b0b23b37906
root@ceph141:/# ceph -s
  cluster:
    id:     c153209c-d8a0-11ef-a0ed-bdb84668ed01
    health: HEALTH_WARN
            mon ceph141 is low on available space
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum ceph141 (age 17h)
    mgr: ceph141.iphxbv(active, since 17h)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
 
root@ceph141:/# exit
exit
```

方式二: 使用cephadm非交互式配置，依旧会启动容器，运行玩后会退出！

```
[root@ceph141 ~]# cephadm shell -- ceph -s
Inferring fsid c153209c-d8a0-11ef-a0ed-bdb84668ed01
Inferring config /var/lib/ceph/c153209c-d8a0-11ef-a0ed-bdb84668ed01/mon.ceph141/config
Using ceph image with id '2bc0b0f4375d' and tag 'v18' created on 2024-07-24 06:19:35 +0800 CST
quay.io/ceph/ceph@sha256:6ac7f923aa1d23b43248ce0ddec7e1388855ee3d00813b52c3172b0b23b37906
  cluster:
    id:     c153209c-d8a0-11ef-a0ed-bdb84668ed01
    health: HEALTH_WARN
            mon ceph141 is low on available space
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum ceph141 (age 17h)
    mgr: ceph141.iphxbv(active, since 17h)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```

方式三: 安装ceph通用包，其中包含所有ceph命令，包括ceph、rbd、mount.ceph（用于挂载CephFS文件系统）等【推荐使用】

```
[root@ceph141 ~]# cephadm add-repo --release reef # 配置本地的apt源
[root@ceph141 ~]# cephadm install ceph-common     # 此步骤，相当于apt -y install ceph-common
Installing packages ['ceph-common']...
[root@ceph141 ~]# ceph -s 
  cluster:
    id:     c153209c-d8a0-11ef-a0ed-bdb84668ed01
    health: HEALTH_WARN
            mon ceph141 is low on available space
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum ceph141 (age 17h)
    mgr: ceph141.iphxbv(active, since 17h)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```

**基于命令行的方式修改ceph的dashboard的密码**

修改admin的密码为1

```
[root@ceph141 ~]# echo 1 | ceph dashboard set-login-credentials admin -i -
******************************************************************
***          WARNING: this command is deprecated.              ***
*** Please use the ac-user-* related commands to manage users. ***
******************************************************************
Username and password updated
```

## 二、**ceph集群添加或移除主机**

1.查看现有的集群主机列表

```
[root@ceph141 ~]# ceph orch host ls
HOST     ADDR        LABELS  STATUS  
ceph141  10.0.0.141  _admin          
1 hosts in cluster
```

 2.拷贝密钥到其他服务器上

```
[root@ceph141 ~]# ssh-copy-id -f -i /etc/ceph/ceph.pub ceph142
[root@ceph141 ~]# ssh-copy-id -f -i /etc/ceph/ceph.pub ceph143
```

3.将密钥节点加入集群

```
[root@ceph141 ~]# ceph orch host add ceph142 10.0.0.142
Added host 'ceph142' with addr '10.0.0.142'

[root@ceph141 ~]# ceph orch host add ceph143 10.0.0.143
Added host 'ceph143' with addr '10.0.0.143'
```

4.再次查看主机列表

```
[root@ceph141 ~]# ceph orch host ls
HOST     ADDR        LABELS  STATUS  
ceph141  10.0.0.141  _admin          
ceph142  10.0.0.142                  
ceph143  10.0.0.143                  
3 hosts in cluster
```

当然，也可以通过查看WebUI观察ceph集群有多少个主机。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/0f88adaa536d4c5b9d39d71a2aca5a66.png)

5.移除主机【选做，如果你将来真有这个需求在操作】

```
[root@ceph141 ~]# ceph orch host rm ceph143
Removed  host 'ceph143'
[root@ceph141 ~]# 
[root@ceph141 ~]# ceph orch host ls
HOST     ADDR        LABELS  STATUS  
ceph141  10.0.0.141  _admin          
ceph142  10.0.0.142                  
2 hosts in cluster
```

##  三、**添加OSD设备到ceph集群**

1.添加OSD之前环境查看

**查看集群可用的设备**【每个设备想要加入到集群，则其大小不得小于5GB】

如果一个设备想要加入ceph集群，要求满足2个条件:

1.设备未被使用;

2.设备的存储大小必须大于5GB;

```
[root@ceph141 ~]# ceph orch device ls
HOST     PATH      TYPE  DEVICE ID                                             SIZE  AVAILABLE  REFRESHED  REJECT REASONS                               
ceph141  /dev/sdc  hdd                                                         200G  Yes        10m ago                                                 
ceph141  /dev/sr0  hdd   VMware_Virtual_SATA_CDRW_Drive_01000000000000000001  2006M  No         10m ago    Has a FileSystem, Insufficient space (<5GB)  
ceph142  /dev/sdb  hdd                                                         100G  Yes        4m ago                                                  
ceph142  /dev/sdc  hdd                                                         200G  Yes        4m ago                                                  
ceph142  /dev/sr0  hdd   VMware_Virtual_SATA_CDRW_Drive_01000000000000000001  2006M  No         4m ago     Has a FileSystem, Insufficient space (<5GB)  
ceph143  /dev/sdb  hdd                                                         100G  Yes        3m ago                                                  
ceph143  /dev/sdc  hdd                                                         200G  Yes        3m ago                                                  
ceph143  /dev/sr0  hdd   VMware_Virtual_SATA_CDRW_Drive_01000000000000000001  2006M  No         3m ago     Has a FileSystem, Insufficient space (<5GB)  
```

**查看各节点的空闲设备信息**

```
[root@ceph141 ~]# lsblk
....
sdb                         8:16   0  200G  0 disk 
sdc                         8:32   0  100G  0 disk 
....

[root@ceph142 ~]# lsblk 
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
...
sdb                         8:16   0  200G  0 disk 
sdc                         8:32   0  100G  0 disk 
...


[root@ceph143 ~]# lsblk 
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
...
sdb                         8:16   0  200G  0 disk 
sdc                         8:32   0  100G  0 disk
```

**查看OSD列表**

```
[root@ceph141 ~]# ceph osd tree
ID  CLASS  WEIGHT  TYPE NAME     STATUS  REWEIGHT  PRI-AFF
-1              0  root default 
```

**添加OSD设备到集群**

此步骤会在"/var/lib/ceph/<Ceph_Cluster_ID>/osd.<OSD_ID>/fsid"文件中记录对应ceph的OSD编号对应本地的磁盘设备标识。

```
[root@ceph141 ~]# ceph orch daemon add osd ceph141:/dev/sdb
Created osd(s) 0 on host 'ceph141'
[root@ceph141 ~]# ceph orch daemon add osd ceph141:/dev/sdc
Created osd(s) 1 on host 'ceph141'
[root@ceph141 ~]# ceph orch daemon add osd ceph142:/dev/sdb
Created osd(s) 2 on host 'ceph142'
[root@ceph141 ~]# ceph orch daemon add osd ceph142:/dev/sdc
Created osd(s) 3 on host 'ceph142'
[root@ceph141 ~]# ceph orch daemon add osd ceph143:/dev/sdb
Created osd(s) 4 on host 'ceph143'
```

 **查看ceph节点的硬盘和OSD的对应关系**

```
[root@ceph141 ~]# ll -d  /var/lib/ceph/*/osd.*
drwx------ 2 167 167 4096 Jan 23 11:19 /var/lib/ceph/c153209c-d8a0-11ef-a0ed-bdb84668ed01/osd.0/
drwx------ 2 167 167 4096 Jan 23 11:19 /var/lib/ceph/c153209c-d8a0-11ef-a0ed-bdb84668ed01/osd.1/

[root@ceph141 ~]# cat /var/lib/ceph/*/osd.*/fsid
1eb45f98-6058-42f2-a5bf-b034e008ac9b
96e33e3d-0acc-43f5-bcee-204feb1582c7


[root@ceph142 ~]# ll -d  /var/lib/ceph/*/osd.*
drwx------ 2 167 167 4096 Jan 23 11:21 /var/lib/ceph/c153209c-d8a0-11ef-a0ed-bdb84668ed01/osd.2/
drwx------ 2 167 167 4096 Jan 23 11:21 /var/lib/ceph/c153209c-d8a0-11ef-a0ed-bdb84668ed01/osd.3/

[root@ceph142 ~]# cat /var/lib/ceph/*/osd.*/fsid
3957c37c-c2fe-4783-984e-93c1184b74a3
b8d5b119-c3e3-4e4e-aab5-85bea49ae3ef

[root@ceph143 ~]# ll -d  /var/lib/ceph/*/osd.*
drwx------ 2 167 167 4096 Jan 23 11:22 /var/lib/ceph/c153209c-d8a0-11ef-a0ed-bdb84668ed01/osd.4/
drwx------ 2 167 167 4096 Jan 23 11:27 /var/lib/ceph/c153209c-d8a0-11ef-a0ed-bdb84668ed01/osd.5/

[root@ceph143 ~]# cat /var/lib/ceph/*/osd.*/fsid
e73be677-e55e-4b54-b007-a0ed5ba07316
2732dd29-342e-4290-ba2b-1b56924339ec
```

 **不难发现，ceph底层是基于lvm技术磁盘的。**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/187d60c358e74fd989e12828ca90dba8.png)

查看集群的osd总容量大小  

```
[root@ceph141 ~]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME         STATUS  REWEIGHT  PRI-AFF
-1         0.87895  root default                               
-3         0.29298      host ceph141                           
 0    hdd  0.19530          osd.0         up   1.00000  1.00000
 1    hdd  0.09769          osd.1         up   1.00000  1.00000
-5         0.29298      host ceph142                           
 2    hdd  0.09769          osd.2         up   1.00000  1.00000
 3    hdd  0.19530          osd.3         up   1.00000  1.00000
-7         0.29298      host ceph143                           
 4    hdd  0.09769          osd.4         up   1.00000  1.00000
 5    hdd  0.19530          osd.5         up   1.00000  1.00000
```

**查看集群大小**  

```
[root@ceph141 ~]# ceph -s
  cluster:
    id:     c153209c-d8a0-11ef-a0ed-bdb84668ed01
    health: HEALTH_WARN
            mon ceph141 is low on available space
 
  services:
    mon: 3 daemons, quorum ceph141,ceph142,ceph143 (age 26m)
    mgr: ceph141.iphxbv(active, since 26m), standbys: ceph142.mxilbz
    osd: 6 osds: 6 up (since 13m), 6 in (since 13m)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 449 KiB
    usage:   161 MiB used, 900 GiB / 900 GiB avail   #这里是集群总大小
    pgs:     1 active+clean
```

 **dashbord查看**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/0c976ae7fdbf4a8597b4f0bac742eeb5.png)

**ceph集群基于chrony进行同步时间**

**存在的问题**

```
[root@ceph141 ~]# ceph -s
  cluster:
    id:     c153209c-d8a0-11ef-a0ed-bdb84668ed01
    health: HEALTH_WARN
            clock skew detected on mon.ceph142
 
  services:
    mon: 3 daemons, quorum ceph141,ceph142,ceph143 (age 8m)
    mgr: ceph141.iphxbv(active, since 8m), standbys: ceph142.mxilbz
    osd: 6 osds: 6 up (since 8m), 6 in (since 112m)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 449 KiB
    usage:   575 MiB used, 899 GiB / 900 GiB avail
    pgs:     1 active+clean
```

**所有节点安装服务**

```
apt -y install chrony 
```

 **ceph141修改配置**

 **修改配置**

```
systemctl restart chronyd
```

**查看配置是否生效**

```
[root@ceph142 ~]# chronyc activity -v
200 OK
1 sources online
0 sources offline
0 sources doing burst (return to online)
0 sources doing burst (return to offline)
7 sources with unknown address
```

**再次查看集群**

```
[root@ceph141 ~]# ceph -s
  cluster:
    id:     c153209c-d8a0-11ef-a0ed-bdb84668ed01
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph141,ceph142,ceph143 (age 8m)
    mgr: ceph141.iphxbv(active, since 8m), standbys: ceph142.mxilbz
    osd: 6 osds: 6 up (since 8m), 6 in (since 112m)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 449 KiB
    usage:   575 MiB used, 899 GiB / 900 GiB avail
    pgs:     1 active+clean
```

## 四、ceph的管理节点配置

**拷贝apt源及认证文件**

```
[root@ceph141 ~]# scp  /etc/apt/sources.list.d/ceph.list ceph142:/etc/apt/sources.list.d/
[root@ceph141 ~]# scp /etc/apt/trusted.gpg.d/ceph.release.gpg  ceph142:/etc/apt/trusted.gpg.d/
```

**客户端更新源并安装客户端节点ceph客户端软件包**

```
[root@ceph142 ~]# ll /etc/apt/trusted.gpg.d/ceph.release.gpg 
-rw-r--r-- 1 root root 1143 Aug 21 16:35 /etc/apt/trusted.gpg.d/ceph.release.gpg
[root@ceph142 ~]# 
[root@ceph142 ~]# ll /etc/apt/sources.list.d/ceph.list 
-rw-r--r-- 1 root root 54 Aug 21 16:33 /etc/apt/sources.list.d/ceph.list
[root@ceph142 ~]# 
[root@ceph142 ~]# apt update
[root@ceph142 ~]# 
[root@ceph142 ~]# apt -y install ceph-common
[root@ceph142 ~]# 
[root@ceph142 ~]# ceph -v
ceph version 18.2.4 (e7ad5345525c7aa95470c26863873b581076945d) reef (stable)
[root@ceph142 ~]# 
[root@ceph142 ~]# ll /etc/ceph/
total 12
drwxr-xr-x   2 root root 4096 Aug 21 16:37 ./
drwxr-xr-x 101 root root 4096 Aug 21 16:37 ../
-rw-r--r--   1 root root   92 Jul 12 23:42 rbdmap
[root@ceph142 ~]# 
[root@ceph142 ~]# ceph -s  # 很明显，此节点的ceph管理ceph集群
Error initializing cluster client: ObjectNotFound('RADOS object not found (error calling conf_read_file)')
```

**ceph141节点拷贝认证文件到ceph142节点**

```
[root@ceph141 ~]# scp /etc/ceph/ceph.{conf,client.admin.keyring} ceph142:/etc/ceph/
```

**ceph142节点测试**

```
[root@ceph142 ~]# ll /etc/ceph/
total 20
drwxr-xr-x   2 root root 4096 Aug 21 16:40 ./
drwxr-xr-x 101 root root 4096 Aug 21 16:37 ../
-rw-------   1 root root  151 Aug 21 16:40 ceph.client.admin.keyring
-rw-r--r--   1 root root  259 Aug 21 16:40 ceph.conf
-rw-r--r--   1 root root   92 Jul 12 23:42 rbdmap

[root@ceph142 ~]# ceph -s
  cluster:
    id:     c153209c-d8a0-11ef-a0ed-bdb84668ed01
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph141,ceph142,ceph143 (age 21m)
    mgr: ceph141.iphxbv(active, since 21m), standbys: ceph142.mxilbz
    osd: 6 osds: 6 up (since 21m), 6 in (since 2h)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 449 KiB
    usage:   575 MiB used, 899 GiB / 900 GiB avail
    pgs:     1 active+clean
```

## 五、标签管理

一般情况下，管理节点，我们都会为节点打上对应的标签，以便于日后工作交接

**添加标签**

```
[root@ceph141 ~]# ceph orch host label add ceph142 _admin
Added label _admin to host ceph142
[root@ceph141 ~]# ceph orch host label add ceph143 _admin
Added label _admin to host ceph143
[root@ceph141 ~]# ceph orch host label add ceph143 cherry
Added label cherry to host ceph143
```

![img](https://gitee.com/ljh00928/csdn/raw/master/img/f310cf021830485da4cfcc35be26b243.png)

**移除标签**

```
[root@ceph141 ~]# ceph orch host label rm ceph143 cherry 
Removed label oldboyedu from host ceph143
[root@ceph141 ~]# ceph orch host label rm ceph143 _admin
Host ceph143 does not have label 'admin'. Please use 'ceph orch host ls' to list all the labels.
[root@ceph141 ~]# ceph orch host label rm ceph143 _admin
Removed label _admin from host ceph143
```

![img](https://gitee.com/ljh00928/csdn/raw/master/img/d8b564855fed4e15ac0e0b28691971d6.png)

