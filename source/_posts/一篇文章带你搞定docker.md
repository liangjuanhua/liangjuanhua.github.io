---
title: 一篇文章带你搞定docker
date: 2025-04-16 16:52:59
tags: docker
categories: 云原生
---
>  Docker 是一个客户端-服务器（C/S）架构程序。Docker 客户端只需要向 Docker 服务器或者守护进程发出请求，服务器或者守护进程将完成所有工作并返回结果。Docker 提供了一个命令行工具 Docker 以及一整套 RESTful API。

通过下图可以得知，Docker 在运行时分为 Docker 引擎（服务端守护进程） 和 客户端工具，我们日常使用各种 docker 命令，其实就是在使用 客户端工具 与 Docker 引擎 进行交互。

# 一、架构图

![img](https://gitee.com/ljh00928/csdn/raw/master/img/1aa02be9df9c4e31b33d5df8f80c2b63.png)

 **Client 客户端**

Docker 是一个客户端-服务器（C/S）架构程序。Docker 客户端只需要向 Docker 服务器或者守护进程发出请求，服务器或者守护进程将完成所有工作并返回结果。Docker 提供了一个命令行工具 Docker 以及一整套 RESTful API。你可以在同一台宿主机上运行 Docker 守护进程和客户端，也可以从本地的 Docker 客户端连接到运行在另一台宿主机上的远程 Docker 守护进程。

**Image 镜像**

什么是 Docker 镜像？简单的理解，Docker 镜像就是一个 Linux 的文件系统（Root FileSystem），这个文件系统里面包含可以运行在 Linux 内核的程序以及相应的数据。
 通过镜像启动一个容器，一个镜像就是一个可执行的包，其中包括运行应用程序所需要的所有内容：包含代码，运行时间，库，环境变量和配置文件等。
 Docker 把 App 文件打包成为一个镜像，并且采用类似多次快照的存储技术，可以实现：
 
 多个 App 可以共用相同的底层镜像（初始的操作系统镜像）；
 App 运行时的 IO 操作和镜像文件隔离；
 通过挂载包含不同配置/数据文件的目录或者卷（Volume），单个 App 镜像可以用来运行无数个不同业务的容器。

**Container 容器**

镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

**repostory仓库**

仓库是集中存储镜像文件的沧桑，registry是仓库主从服务器，实际上参考注册服务器上存放着多个仓库，每个仓库中又包含了多个镜像，每个镜像有不同的标签（tag）。

仓库分为两种，公有参考，和私有仓库，最大的公开仓库是docker Hub，存放了数量庞大的镜像供用户下周，国内的docker pool，这里仓库的概念与Git类似，registry可以理解为github这样的托管服务

# 二、docker部署

**下载软件包**

```
wget https://download.docker.com/linux/static/stable/x86_64/docker-20.10.24.tgz
```

**解压软件包**

```
root@docker101:~# tar xf docker-20.10.24.tgz
```

**拷贝到PATH变量**

```
root@docker101:~# cp docker/* /usr/bin/
```

**启动docker**

```
root@docker101:~# dockerd 
root@docker101:~# dockerd &  #后台运行
```

**查看docker版本号**

```
root@docker101:~# docker version
```

# **三、卸载docker环境**

**停止docker环境**

```
root@docker101:~# pkill dockerd
```

**卸载**

```
root@docker101:~# for i in `ls docker`;do rm -f /usr/bin/$i ;done
```

**验证**

```
root@docker101:~# docker
Command 'docker' not found, but can be installed with:
```

# 四、docker常用命令

### Docker基础命令

启动/停止/重启docker

> \# 启动
>  systemctl start docker
>  \# 停止
>  systemctl stop docker
>  \# 重启
>  systemctl restart docker

设置开机自启动

> \# 设置
>  systemctl enable docker
>  \# 取消开机自启动
>  systemctl disable docker

查看docker状态

> systemctl status docker

查看版本信息

> docker version
>
> \#该命令显示当前安装的Docker客户端和服务器版本信息。

显示Docker系统信息

> docker info

查看帮助

> docker --help 

###  镜像管理命令

搜索镜像

> docker search [镜像名]

下载镜像

> docker pull [镜像名]:[标签] 

列出本地镜像

> docker images 

删除镜像

> docker rmi [镜像ID或镜像名] 

删除全部镜像

> docker rmi -f $(docker images -aq)
>
> -a 意思为显示全部,
>
> -q 意思为只显示ID 

构建镜像

> docker build -t [镜像名]:[标签] [Dockerfile所在路径]

导入镜像 

> docker load -i /data/nginx.tar

保存镜像

> docker save -o /data/nginx.tar nginx 

给镜像打标签

> docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
>  docker tag nginx 10.10.10.200/software/nginx:1.26 

###  容器管理命令

创建并运行容器

> docker run [选项] [镜像名]
>
> -d 后台运行容器
>
> -p 端口映射
>
> --name 指定容器名称
>
> -v 挂载卷 主机路径:容器路径

在后台运行一个名为mynginx的nginx容器，并映射端口：

> docker run -d -p 8080:80 --name mynginx nginx

 查看运行中的容器

> docker ps

查看所有容器（包括停止的）

>  docker ps -a

 启动和停止容器

> \# 启动容器
>  docker start [容器ID或容器名]
>  
>  \# 停止容器
>  docker stop [容器ID或容器名]

重启容器

> docker restart [容器ID或容器名] 

删除容器

> docker rm [容器ID或容器名] 

进入容器

> docker exec -it [容器ID或容器名] /bin/bash 

查看容器日志

> 1.查看实时日志
>  docker container logs -f  c1
>
> 
>
> 2.查看20分钟之内的日志
>  docker container logs -f  --since 20m c1
>
> 
>
> 3.查看20分钟之前的日志
>  docker container logs -f  --until 20m c1

查看容器内部细节

> docker inspect [容器ID或容器名] 

### 数据卷管理命令

数据卷（Volume）是Docker中持久化数据的关键，通过数据卷可以将容器内的数据持久化到宿主机中 。

创建数据卷

> docker volume create [卷名]
>  docker volume create data

 查看数据卷

> docker volume ls

删除数据卷

> docker volume rm [卷名]
>  docker volume rm data 

 查看数据卷详情

> docker volume inspect [卷名]

### 网络管理命令 

创建网络

> docker network create [网络名]

查看网络

> docker network ls 

 查看网络详情

> docker network inspect [网络名]

删除网络

> docker network rm [网络名]

将容器连接到网络

> docker network connect [网络名] [容器名或容器ID] 

将容器从网络断开

> docker network disconnect [网络名] [容器名或容器ID] 

### Docker Compose命令 

> 编译镜像
>  docker-compose build
>
> 后台创建并启动容器
>  docker-compose up -d
>
> 查看容器状态
>  docker-compose ps
>
> 停止所有服务
>  docker-compose stop -t 
>
> 启动所有服务
>  docker-compose start
>
> 重启指定服务
>  docker-compose restart doudizhu
>
> 删除并停止容器
>  docker-compose down -t 0

### 常用清理命令

删除所有已停止的容器

> docker container prune

 删除未使用的镜像

> docker image prune

删除所有未使用的数据卷

> docker volume prune 

 删除所有未使用的网络

> docker network prune

 清理所有未使用的资源（包括镜像、容器、卷和网络）

> docker system prune

#  五、docker网络原理

 docker默认使用的单机容器网络模型。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/03dd15f5ef154a0dbaf235e37f886edd.png)

1. 每个容器（Container）分别拥有自己的Network Namespace。
2. 容器通过对设备，连接到宿主机的Host Network Namespace。对设备在容器Network Namespace这一端的“网卡”是eth0，eth0配置的ip即容器的ip。对设备连接Host Namespace的那一端挂载到网桥设备docker0。
3. 网桥设备docker0，挂载着所有容器的对设备的Host Namespace这一端。并且，挂载在网桥上的设备，会被降级成网桥上的一个端口，端口的唯一作用就是转发网桥或另一端对设备的数据包。
4. 从Container1发送到Container2的数据包，首先经过Container1中的eth0，到达docker0网桥，docker0网桥经过二层转发，将数据包发送到Container2对应的端口（Container2对设备的docker0网桥这一端），这样数据包就被直接送到Container2中了。

# 六、docker网络类型

**单机网络类型**

> brdge    #默认网络类型，网桥模式，docker在宿主机创建docker 0网桥
>  none    #容器将没有网络连接，用于不需要网络功能的容器
>  host     #容器直接使用宿主机的端口，不需要知道端口
>  contianer #与另一个运行中的容器共享Network Namespace
>  自定义网络 #相当于内置dns，基于容器名称访问彼此

**跨主机网络类型**

> macvlan       #手动分配ip，绑定物理网卡
>  overlay+consul  #创建overlay网络，利用consul服务发现
>  flannel+etcd
>  calica+etcd

**macvlan和overlay的区别**

相同点: 都可以实现网络的互相通信

不同点:

- macvlan是内核支持模块，无需安装第三方插件，只需加载模块即可，overlay需要安装第三方插件consul;
- macvlan需要手动分配IP地址，而overlay网络无需手动分配IP地址;
- macvlan默认无法访问外网，需要手动配置桥接网络，而overlay默认可以访问外网;

**docker网络不足的总结**

- 1.docker在网络互联上存在缺陷，比如overlay网络各节点实现IP地址通信，当容器挂掉时，会自动为该容器分配IP地址。若容器重 启后，IP地址可能发生变化;
- 2.若配置文件写的都是IP地址，则容器重启后IP地址发生变化，可能导致服务不可用;

# 七、docker底层使用的linux技术

 Docker 是用Go 编程语言编写的，并利用 Linux 内核的几个特性来提供其功能。

 Docker 使用一种称为容器 `namespaces `的技术来提供隔离的工作空间。当您运行容器时，      Docker 会为该容器创建一组 命名空间*。*

 这些命名空间提供了一层隔离。容器的每个方面都在单独的命名空间中运行，并且它的访问权限仅限于该命名空间。

 Docker底层的核心技术包括

- Linux 上的名字空间（Namespaces）
- 控制组（Control groups）
- Union 文件系统（Union file systems）
- 容器格式（Container format）

### namespace

NameSpace 是 Linux 内核一个强大的特性。每个容器都有自己单独的名字空间，运行在其中的应用都像是在独立的操作系统中运行一样。名字空间保证了容器之间彼此互不影响。

- pid 名字空间
   不同用户的进程就是通过 pid 名字空间隔离开的，且不同名字空间中可以有相同 pid。所有的 LXC 进程在Docker 中的父进程为Docker进程，每个 LXC 进程具有不同的名字空间。同时由于允许嵌套，因此可以很方便的实现嵌套的 Docker 容器。
- net 名字空间
   有了 pid 名字空间, 每个名字空间中的 pid 能够相互隔离，但是网络端口还是共享 host 的端口。网络隔离是通过 net 名字空间实现的， 每个 net 名字空间有独立的 网络设备, IP 地址, 路由表, /proc/net 目录。这样每个容器的网络就能隔离开来。Docker 默认采用 veth 的方式，将容器中的虚拟网卡同 host 上的一 个Docker网桥 docker0 连接在一起。
- ipc 名字空间
   容器中进程交互还是采用了 Linux 常见的进程间交互方法(interprocess communication – IPC), 包括信号量、消息队列和共享内存等。然而同 VM 不同的是，容器的进程间交互实际上还是 host 上具有相同 pid 名字空间中的进程间交互，因此需要在 IPC 资源申请时加入名字空间信息，每个 IPC 资源有一个唯一的 32位 id。
- mnt 名字空间
   类似 chroot，将一个进程放到一个特定的目录执行。mnt 名字空间允许不同名字空间的进程看到的文件结构不同，这样每个名字空间 中的进程所看到的文件目录就被隔离开了。同 chroot 不同，每个名字空间中的容器在 /proc/mounts 的信息只包含所在名字空间的 mount point。
- uts 名字空间
   UTS(“UNIX Time-sharing System”) 名字空间允许每个容器拥有独立的 hostname 和 domain name, 使其在网络上可以被视作一个独立的节点而非 主机上的一个进程。
- user 名字空间
   每个容器可以有不同的用户和组 id, 也就是说可以在容器内用容器内部的用户执行程序而非主机上的用户。

### cgroups

cgroups 是 Linux 内核的一个特性，主要用来对共享资源进行隔离、限制、审计等。只有能控制分配到容器的资源，才能避免当多个容器同时运行时的对系统资源的竞争。

cgroups 技术最早是由 Google 的程序员 2006 年起提出，Linux 内核自 2.6.24 开始支持。

cgroups 可以提供对容器的内存、CPU、磁盘 IO 等资源的限制和审计管理。

### unionfs

联合文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下(unite several directories into asingle virtual filesystem)。

联合文件系统是 Docker 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。

另外，不同 Docker 容器就可以共享一些基础的文件系统层，同时再加上自己独有的改动层，大大提高了存储的效率。

Docker 中使用的 AUFS（AnotherUnionFS）就是一种联合文件系统。 AUFS 支持为每一个成员目录（类似 Git 的分支）设定只读（readonly）、读写（readwrite）和写出（whiteout-able）权限, 同时 AUFS 里有一个类似分层的概念, 对只读权限的分支可以逻辑上进行增量地修改(不影响只读部分的)。

Docker 目前支持的联合文件系统种类包括 AUFS, btrfs, vfs 和 DeviceMapper。

### 容器格式

最初，Docker 采用了 LXC 中的容器格式。自 1.20 版本开始，Docker 也开始支持新的 libcontainer 格式，并作为默认选项。

# 八、dockerfile

Dockerfile是用来快速创建自定义镜像的一种文本格式的配置文件，在持续集成和持续部署时，需要使用Dockerfile生成相关应用程序的镜像。

### **Dockerfile的常用命令**

> FROM：继承基础镜像
>  MAINTAINER：镜像制作作者的信息，已弃用，使用LABEL替代
>  LABEL：k=v形式，将一些元数据添加至镜像
>  RUN：用来执行shell命令
>  EXPOSE：暴露端口号
>  CMD：启动容器默认执行的命令，会被覆盖
>  ENTRYPOINT：启动容器真正执行的命令，不会被覆盖
>  VOLUME：创建挂载点ENV：配置环境变量
>  ADD：复制文件到容器，一般复制文件，压缩包自动解压
>  COPY：复制文件到容器，一般复制目录
>  WORKDIR：设置容器的工作目录
>  USER：容器使用的用户ARG：设置编译镜像时传入的参数

### **jar包打镜像案例**

```
FROM java:graalvm-ce-java8-21.2.0

ENV LANG C.UTF-8
ENV TZ=Asia/Shanghai

ADD contract-online-sign-server/target/*.jar /app.jar

ENTRYPOINT ["java", "-jar","/app.jar"]
```

### **env和arg指令有什么区别？**

> 都是向容器传递环境变量
>  \- arg是基于构建阶段传递环境变量
>  \- env不仅可以用于构建阶段传递环境变量还可以用于容器运行时传递环境变量

### **CMD和ENTRYPOINT有啥区别？**

> 都可以作为容器启动命令
>  \- entrypoint指定可以将指定启动命令作为参数传递
>  \- 他们两个一块使用的时候，cmd将作为传递参数，当然如果用户在运行时指定了启动命令，会覆盖cmd的add和copy区别？默认值

### **add和copy区别？** 

> 都可以拷贝文件
>  \- add在拷贝tar包文件会自动解压
>  \- copy指令在多阶段构建的时候可以从其他节点拷贝数据

### 镜像优化思路？ 

> 编译速度:
>    1.Dockefile在编译时，如果对应的指令记录被执行过，就可以直接使用缓存，因此将不频繁修改的指令往上放，将经常修改的指令往下放，以达到利用缓存的目的;
>    2.忽略Docker编译时不必要文件;
>    3.使用国内的软件源或本地仓库，以提示网络的下载速度; 
>    4.将比较大的软件包放在本地或内网的文件站点，避免下载;
>
> 镜像大小:
>    1.删除缓存，无用的软件包以减少镜像大小;
>    2.能够合并的指令，尽量合并，减少不必要的镜像分层，每一个Dockerfile指令都会产生一个中间层镜像(docker image ls -a);
>    3.使用较小的基础镜像
>
>   4.使用多阶段构建

