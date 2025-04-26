---
title: k8s集成MinIo
date: 2025-04-18 11:25:01
tags: k8s
categories: 存储篇
---
 本篇文章分享一下在 k8s怎么集成 minio做存储，并实现 `PersistentVolume (PV)`、`PersistentVolumeClaim (PVC)`、动态存储卷`StorageClass`，以及演示让pod使用这些存储卷的完整流程。

# 一、理论

### 1、PV概念

PV是对K8S存储资源的抽象，PV一般由运维人员创建和配置，供容器申请使用。

没有PV之前，服务器的磁盘没有分区的概念，有了PV之后，相当于通过PV对服务器的磁盘进行分区。

### 2、PVC概念

PVC 是Pod对存储资源的一个申请，主要包括存储空间申请、访问模式等。创建PV后，Pod就可以通过PVC向PV申请磁盘空间了。类似于某个应用程序向`操作系统的D盘`申请1G的使用空间。

PVC 创建成功之后，Pod 就可以以存储卷（Volume）的方式使用 PVC 的存储资源了。Pod 在使用 PVC 时必须与PVC在同一个Namespace下。

### 3、PV / PVC的关系

PV相当于对磁盘的分区，PVC相当于APP（应用程序）向某个分区申请多少空间。比如说安装WPS程序时，一般会告知我们安装它需要多少存储空间，让你选择在某个磁盘下安装。如果将来某个分区磁盘满了，也不会影响别的分区磁盘的使用。

一旦 PV 与PVC绑定，Pod就可以使用这个 PVC 了。如果在系统中没有满足 PVC 要求的 PV，PVC则一直处于 Pending 状态，直到系统里产生了一个合适的 PV。

### 4、StorageClass概念

K8S有两种存储资源的供应模式：静态模式和动态模式，资源供应的最终目的就是将适合的PV与PVC绑定：

- 静态模式：管理员预先创建许多各种各样的PV，等待PVC申请使用。
- 动态模式：管理员无须预先创建PV，而是通过StorageClass自动完成PV的创建以及与PVC的绑定。

StorageClass就是动态模式，根据PVC的需求动态创建合适的PV资源，从而实现存储卷的按需创建。

一般某个商业性的应用程序，会用到大量的Pod，如果每个Pod都需要使用存储资源，那么就需要人工时不时的去创建PV，这也是个麻烦事儿。解决方法就是使用动态模式：当Pod通过PVC申请存储资源时，直接通过StorageClass去动态的创建对应大小的PV，然后与PVC绑定，所以基本上PV → PVC是一对一的关系。

# 二、pod数据存储流程

![img](https://gitee.com/ljh00928/csdn/raw/master/img/96f7b4369f9c4d3db6f70e391bd391fb.png)

# 三、部署minio

官方文档：[MinIO | Code and downloads to create high performance object storage](https://min.io/download?view=aistor-custom)

**下载minio**

```
# 下载服务端
[root@minio opt]# wget https://dl.min.io/server/minio/release/linux-amd64/minio
# 将下载所得minio文件拷贝到指定文件夹并赋权
[root@minio opt]# cp minio /usr/local/bin/
[root@minio opt]# chmod +x /usr/local/bin/minio
```

**为MinIO创建一个存储目录：**

```
[root@minio opt]# mkdir /data
```

**启动MinIO，并指定存储目录和访问地址：**

```
[root@minio opt]# minio server /data --console-address ":9099"
INFO: Formatting 1st pool, 1 set(s), 1 drives per set.
INFO: WARNING: Host local has more than 0 drives of set. A host failure will result in data becoming unavailable.
MinIO Object Storage Server
Copyright: 2015-2025 MinIO, Inc.
License: GNU AGPLv3 - https://www.gnu.org/licenses/agpl-3.0.html
Version: RELEASE.2025-01-18T00-31-37Z (go1.23.5 linux/amd64)

API: http://10.0.0.12:9000  http://127.0.0.1:9000 
   RootUser: minioadmin 
   RootPass: minioadmin 

WebUI: http://10.0.0.12:9099 http://127.0.0.1:9099   
   RootUser: minioadmin 
   RootPass: minioadmin 

CLI: https://min.io/docs/minio/linux/reference/minio-mc.html#quickstart
   $ mc alias set 'myminio' 'http://10.0.0.12:9000' 'minioadmin' 'minioadmin'

Docs: https://docs.min.io
WARN: Detected default credentials 'minioadmin:minioadmin', we recommend that you change these values with 'MINIO_ROOT_USER' and 'MINIO_ROOT_PASSWORD' environment variables
```

其中 `/data` 是用于存储 MinIO 数据的目录，`--console-address ":9099"` 是指定 MinIO Web 控制台的监听地址为端口 `9099`。

### **1.访问**

```
http://10.0.0.12:9099
账号：minioadmin 
密码：minioadmin
```

![img](https://gitee.com/ljh00928/csdn/raw/master/img/d99c83c9e3b54dbe9715249bf668cba2.png)

### 2.配置自启动服务

为简化MinIO配置，我们可将MinIO的配置统一写入一个配置文件，以供启动时调用。配置方式如下：

```
[root@minio ~]# vim /etc/default/minio
# 指定数据存储目录(注意：这个目录要存在且拥有相对应的权限)
MINIO_VOLUMES="/data"

# 监听端口
MINIO_OPTS="--address :9000 --console-address :9099"

# 老版本使用MINIO_ACCESS_KEY/MINIO_SECRET_KEY，新版本已不建议使用
# Access key (账号)
# MINIO_ACCESS_KEY="minioadmin"
# Secret key (密码)
# MINIO_SECRET_KEY="minioadmin"

# 新版本使用；指定默认的用户名和密码，其中用户名必须大于3个字母，否则不能启动
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="minioadmin666"

# 区域值，标准格式是“国家-区域-编号”，
MINIO_REGION="cn-north-1"

# 域名
# MINIO_DOMAIN=minio.your_domain.com
```

### 3.编写服务文件

创建minio.service服务文件，并写入配置信息：

```
[root@minio ~]# vim /usr/lib/systemd/system/minio.service
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio
[Service]
WorkingDirectory=/usr/local/

ProtectProc=invisible

# 指向3.1节中的配置文件
#EnvironmentFile=/etc/default/minio
EnvironmentFile=-/etc/default/minio

ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

# Let systemd restart this service always
Restart=always

# Specifies the maximum (1M) file descriptor number that can be opened by this process
LimitNOFILE=1048576

# Specifies the maximum number of threads this process can create
TasksMax=infinity

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=infinity
SendSIGKILL=no
SuccessExitStatus=0

[Install]
WantedBy=multi-user.target
Alias=minio.service
```

**使服务生效**

通过systemctl将服务生效并启动服务。

```
# 重新加载服务配置文件，使服务生效
systemctl daemon-reload

# 将服务设置为开机启动
systemctl enable minio

# 服务立即启动
systemctl start minio

# 查看minio服务当前状态
systemctl status minio
```

重新登录

```
http://10.0.0.12:9099
账号：minioadmin 
密码：minioadmin666
```

如果起不来可以查看日志排查

```
[root@minio ~]# journalctl -u minio -f 
```

# 四、minio集成k8s

MinIO控制台提供了一个图形用户界面（GUI），用于与MinIO租户进行交互。默认情况下，MinIO操作员为每个租户安装和配置控制台。

参考地址：https://github.com/yandex-cloud/k8s-csi-s3

下载资源包

```
[root@master231 kubernetes]# pwd 
/opt/k8s-csi-s3-0.35.5/deploy/kubernetes

[root@master231 kubernetes]# tree ./
./
├── attacher.yaml
├── csi-s3.yaml
├── examples
│   ├── pod.yaml
│   ├── pvc-manual.yaml
│   ├── pvc.yaml
│   ├── secret.yaml
│   └── storageclass.yaml
└── provisioner.yaml

1 directory, 8 files
```

**部署driver**

```
cd deploy/kubernetes
kubectl create -f provisioner.yaml
kubectl create -f driver.yaml
kubectl create -f csi-s3.yaml
```

**查看**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/56830496c9b84e6c8ed0b49c37b734bb.png)

**创建存储类**

```
[root@master231 examples]# vim storageclass.yaml 
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: csi-s3
provisioner: ru.yandex.s3.csi
parameters:
  mounter: geesefs
  # you can set mount options here, for example limit memory cache size (recommended)
  options: "--memory-limit 1000 --dir-mode 0777 --file-mode 0666"
  # to use an existing bucket, specify it here:
  #bucket: some-existing-bucket
  csi.storage.k8s.io/provisioner-secret-name: csi-s3-secret
  csi.storage.k8s.io/provisioner-secret-namespace: kube-system
  csi.storage.k8s.io/controller-publish-secret-name: csi-s3-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: kube-system
  csi.storage.k8s.io/node-stage-secret-name: csi-s3-secret
  csi.storage.k8s.io/node-stage-secret-namespace: kube-system
  csi.storage.k8s.io/node-publish-secret-name: csi-s3-secret
  csi.storage.k8s.io/node-publish-secret-namespace: kube-system
  
[root@master231 examples]# kubectl apply -f storageclass.yaml 
```

###  **1.模拟生成环境测试**

**创建pvc**

```
[root@master231 examples]# cat pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-s3-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: csi-s3
  
[root@master231 examples]# kubectl apply -f pvc.yaml 
```

**创建pod**

```
[root@master231 examples]# cat pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: csi-s3-test-nginx
  namespace: default
spec:
  containers:
   - name: csi-s3-test-nginx
     image: nginx
     volumeMounts:
       - mountPath: /usr/share/nginx/html/s3
         name: webroot
  volumes:
   - name: webroot
     persistentVolumeClaim:
       claimName: csi-s3-pvc
       readOnly: false
```

**创建secret**

```
[root@master231 examples]# cat secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  namespace: kube-system
  name: csi-s3-secret
stringData:
  accessKeyID: minioadmin
  secretAccessKey: minioadmin666
  #填写minio api接口地址
  endpoint: http://10.0.0.12:9000  
  # For AWS set it to AWS region
  #region: ""
```

 **检查pv,pvc,sc,pod状态**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/4e28afc200f2413687d45a788078a5ae.png)

**写入数据**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/637c135105f44604ac8ba1c2ff526070.png)

**minio web界面查看是否写入成功**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/5b7bb21d10a946bbbb18cfc0f6014bb7.png)