---
title: OOM排查思路
date: 2025-04-16 17:17:45
tags: 故障指南
categories: 故障指南
---
 K8S + 容器的云原生生态，改变了服务的交付方式，自愈能力和自动扩缩等功能简直不要太好用。

有好的地方咱要夸，不好的地方咱也要说，真正的业务是部署于容器内部，而容器之外，又有一逻辑层 Pod 。

对于容器和 K8S 不怎么熟悉的人，一旦程序发生了问题，排查问题就是个头疼的问题。

## 问题描述

事情的主角是 kubevirt 的一个开源项目叫 cdi，它的用途是在虚拟机启动之前将虚拟机的镜像导入到系统盘中。

在使用过程中，我们发现 cdi 在导入数据时会占用大量的内存空间。

而 cdi-controller 在创建 cdi-importer 的 pod 时，默认限定其最高只能使用 600M 的内存，到最后呢，pod 就发生了 OOMKilled

```
[root@master01 ~]# kubectl get po
NAME                               READY   STATUS      RESTARTS   AGE
importer-wbm-vda          0/1     OOMKilled   1          76s
```

## **一、查看 Pod 状态**

首先，检查相关 Pod 的状态，确定是否因为内存超限被杀死。

```
[root@master01 ~]# kubectl describe pod <pod-name> -n <namespace>
```

**定位关键词**

如果容器因为 OOM 被杀死，通常会显示如下信息

```
State:     Terminated
Reason:    OOMKilled
```

**查看 `Events` 部分：**

如果 Pod 因为内存不足而被 OOM Killer 杀死，你会在事件中看到类似的提示

```
Warning  OOMKilling  kubelet, <node-name>  Memory cgroup out of memory: Kill process
```

## 二、查看容器日志

如果 Pod 被杀死，可以通过查看容器日志，了解容器在发生 OOM 之前的行为。这有助于判断是否存在内存泄漏或内存使用过高的情况

```
[root@master01 ~]# kubectl logs <pod-name> -n <namespace> --previous
```

- 使用 `--previous` 参数查看被终止容器的日志。
- 如果容器在 OOM 之前没有记录错误信息，通常表示容器已经用尽了可用内存，导致进程直接被杀死。

## 三、查看 Kubelet 和 Node 日志

**查看 Kubelet 日志**

```
[root@master01 ~]# journalctl -u kubelet -f
```

**或者直接查找与 OOM 相关的日志**

```
[root@master01 ~]# journalctl -u kubelet | grep -i 'oom'
```

**查看 Node 上的 `dmesg` 日志，确认是否存在 OOM Killer 杀死进程的记录**

```
[root@master01 ~]# dmesg | grep -i 'oom'
```

## 四、查看资源限制（CPU 和内存）

确保 Pod 的资源限制配置正确。如果 Pod 的内存限制（`memory limit`）设置得太低，可能会导致 OOM 错误

```
[root@master01 ~]# kubectl get pod <pod-name> -n <namespace> -o yaml
```

检查 `resources` 配置

```
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

## 五、分析容器内存使用

查看容器的内存使用情况，确认容器是否超过了内存限制。可以使用 **`kubectl top`** 来查看资源使用情况

```
[root@master01 ~]# kubectl top pod <pod-name> -n <namespace>
```

- 该命令会显示容器当前的内存和 CPU 使用情况。
- 比较容器的 `memory usage` 和 `memory limit`，如果容器接近其内存限制，就可能发生 OOM 错误。

## 六、分析节点内存压力

OOM 错误可能不仅仅是由于容器本身的内存使用高导致的，也可能是因为节点的整体内存资源不足

```
[root@master01 ~]# kubectl describe node <node-name>
```

## 七、检查内存泄漏

如果某个容器频繁 OOM，并且它的内存使用量持续增长，可能是程序中存在内存泄漏

```
[root@master01 ~]# vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 1  0      0 3099200 590472 1748204    0    0     3    37  527    1  0  0 99  0  0  0
 0  0      0 3100144 590472 1748244    0    0     0     0  580  501  0  0 100  0  0  0
 0  0      0 3100708 590472 1748244    0    0     0     0  627  478  0  0 100  0  0  0
 0  0      0 3096128 590472 1748244    0    0     0     0 1628 1672  1  1 98  0  0  0
```

## 八、设置合理的资源限制

为了避免 OOM 问题的再次发生，确保设置合理的内存请求（`requests`）和限制（`limits`）

- **`requests`**：表示容器启动时需要的最小内存。Kubernetes 会根据 `requests` 为容器分配内存，确保容器有足够的资源启动。
- **`limits`**：表示容器可以使用的最大内存。容器使用超过该限制的内存会被 OOM Killer 杀死。

## 九、缓存问题

通过不断的 Google 搜索，我查到了 kubectl top 得到的内存使用数据原来是这么计算的

```
memory.usage_in_bytes-total_inactive_file
```

从这个公式可以看出， kubectl top 得到的内存使用数据原来是包含 cache 的。

正常的 cache 可以提高磁盘数据的读写数据，在读的时候，会拷贝一份文件数据放到内存中，这部分是可回收的，一旦程序内存不足了，会回收部分 cache 的空间，保证程序的正常运行。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/cd1012d805b44daca936117c8b9876aa.png)

可见读文件的缓存，不会影响内存的申请，更别说 OOM，但在写的时候，情况就不一样了

在写的时候，由于进程处理数据的速度，可能会远大于数据落盘的速度，所以为提高格式转化和数据导入的速度，一般会先将转化好的数据存入缓存中，存入缓存后，进程可以立马 return 回去继续下一堆数据的处理，不用傻傻地等待数据全写入磁盘。

而存在于缓存之中的数据，则由操作系统同步写入磁盘，这样一来，数据落盘就变成了一个异步的过程，大大提高了写入的速度。

如果 qemu-img 处理数据的速度远大于 cache 存入磁盘的速度，就会出现内存不足。