---
title: k8s部署nginx+sshd实现文件上传下载
date: 2025-04-16 16:50:54
tags: k8s
categories: 云原生
---
要通过 `nginx` 和 `sshd` 实现文件的上传和下载，通常的做法是结合 SSH 协议和 HTTP 协议，使用 `nginx` 提供 Web 服务器功能，同时使用 `sshd`（即 SSH 服务）来处理通过 SSH 协议进行的文件传输。

- SSH 实现文件的上传和下载： 通过 `sshd` 实现文件上传和下载通常使用 SCP 或 SFTP 协议。你可以通过 SSH 客户端将文件上传到服务器，或从服务器下载文件。这个过程不依赖于 `nginx`，但你可以通过 `nginx` 提供 Web 界面来管理文件传输。

- nginx 提供 Web 界面进行文件上传和下载： `nginx` 本身并不直接处理文件上传功能，但你可以配合一些后端服务（如 PHP、Python、Node.js 等）来实现文件上传和下载的 Web 界面。

### 一、准备工作

**思路**

在同个pod部署nginx和sshd服务，然后共享一个存储卷即可

**准备nginx和ssd的镜像**

```bash
docker pull nginx:stable-alpine
docker pull circleci/sshd:0.1
```

 **共享目录**

```bash
/usr/share/nginx/html
```

**示意图**
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/4a2031994d8c4329b0a1d1bccacec958.png)

### 二、配置共享存储

**创建一个 PVC 来请求共享存储**

```bash
[root@node1.local ~]# nginx-ssh-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
    - ReadWriteMany  # 允许多个容器读写同一存储
  resources:
    requests:
      storage: 5Gi  # 存储大小可以根据需要调整

```

**部署 PVC**

```bash
kubectl apply -f nginx-ssh-pvc.yaml
```



### **三、sshd打docker镜像**

```bash
#查看目录
[root@node1.local sshd]# ll
total 20
drwxr-xr-x  2 root root 4096 Dec 24 13:50 ./
drwx------ 33 root root 4096 Dec 30 16:52 ../
-rw-r--r--  1 root root  174 Dec 24 12:00 Dockerfile
-rw-r--r--  1 root root  591 Dec 24 11:48 shadow
-rw-r--r--  1 root root  140 Dec 24 13:32 sshd_config

#生成加密密码
[root@node1.local sshd]# openssl passwd -6
Password: 
Verifying - Password: 
$6$YiALKQwJcDubTbBn$OEKLYvJfA8vkXAbgCGqTonP.hz5v4/gDcdvDJx0xHGiHlU.Obqpgji0m5tt1vHcTsUlqnFaMSzNiBlnn0USQZ0

#设置root密码
[root@node1.local sshd]# cat shadow 
root:$6$YiALKQwJcDubTbBn$OEKLYvJfA8vkXAbgCGqTonP.hz5v4/gDcdvDJx0xHGiHlU.Obqpgji0m5tt1vHcTsUlqnFaMSzNiBlnn0USQZ0:20081:0:::::
bin:!::0:::::
...

#将配置文件添加到容器
[root@node1.local sshd]# cat sshd_config 
UsePAM yes
PasswordAuthentication yes
PermitEmptyPasswords no
ChallengeResponseAuthentication no
PermitRootLogin yes
AllowTcpForwarding yes
```

**编写dockerfile**

```bash
[root@node1.local sshd]# cat Dockerfile 
FROM harbor.cherry.com/sshd/sshd:0.1

COPY shadow /etc/shadow
COPY sshd_config /etc/ssh/sshd_config

ENV TZ=Asia/Shanghai

RUN chmod 640 /etc/shadow
```

**打镜像**

```bash
docker build -t . sshd:v2
```

**推送harbor仓库**

```bash
docker tag sshd:v2 harbor.cherry.com/sshd/sshd:2
docker push harbor.cherry.com/sshd/sshd:2
```

### 四、部署 Nginx 和 SSH

在同个pod中来运行 Nginx 和 SSH 服务，并使用共享的 PVC 挂载文件存储

```bash
[root@node1.local ~]# nginx-ssh-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ssh-pod
spec:
  containers:
  - name: nginx
    image: nginx:stable-alpine  # 使用官方 Nginx 镜像
    ports:
      - containerPort: 80
    volumeMounts:
      - name: shared-storage
        mountPath: /usr/share/nginx/html  # 共享目录，用于提供文件下载
  - name: ssh
    image: harbor.cherry.com/sshd/sshd:2  # 使用自定义的 SSH 镜像
    ports:
      - containerPort: 22
    volumeMounts:
      - name: shared-storage
        mountPath: /usr/share/nginx/html  # 共享目录，用于文件上传
  volumes:
    - name: shared-storage
      persistentVolumeClaim:
        claimName: shared-pvc  # 使用上面创建的 PVC

```

此配置文件定义了一个包含两个容器的 Pod：

- Nginx 容器：它提供文件下载服务，将 `/usr/share/nginx/html` 目录挂载到共享存储。
- SSH 容器：它提供文件上传服务，将`/usr/share/nginx/html`目录挂载到共享存储

**部署pod**

```bash
kubectl apply -f nginx-ssh-pod.yaml
```

### 五、暴露 Nginx 和 SSH 服务

**创建 Nginx Service**

```bash
[root@node1.local ~]# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-ssh-pod
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

**创建 SSH Service**

```bash
[root@node1.local ~]# ssh-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ssh-service
spec:
  selector:
    app: nginx-ssh-pod
  ports:
    - protocol: TCP
      port: 22
      targetPort: 22
  type: LoadBalancer 
```

### 六、访问使用

- **文件下载**：可以通过直接访问web界面 http://<nginx-service-ip>/files/来下载文件。
- **文件上传**：可以通过winscp来实现上传文件